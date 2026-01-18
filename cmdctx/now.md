# Now: Quick Wins for Thread-Safe Command Execution

This document lists immediate, low-risk changes that pave the way for the full plan.md implementation.

---

## 1. Fix rtr_http.inc.c Block Handling (HIGH PRIORITY)

**File:** `libr/core/rtr_http.inc.c` (lines 200-231, 275-280)

**Current Issue:**
```c
// Line 204-211: Allocates newblk but doesn't handle blocksize changes
newblk = malloc (core->blocksize);
memcpy (newblk, core->block, core->blocksize);
core->block = newblk;
```

If a command changes `core->blocksize`, this leads to:
- Buffer overflow when writing to newblk
- Memory corruption when restoring origblk

**Quick Fix:**
```c
// Replace the block swap logic with on-demand reading
// Don't touch core->block at all - use r_io_read_at directly in cmdstr()

// In cmdstr(), before executing command:
static char *cmdstr_isolated(RCore *core, const char *cmd) {
    // Save state
    ut64 orig_addr = core->addr;
    ut32 orig_bsize = core->blocksize;

    // Execute command
    char *out = r_core_cmd_str_pipe (core, cmd);

    // Restore state (block is re-read by r_core_seek)
    if (core->addr != orig_addr || core->blocksize != orig_bsize) {
        core->blocksize = orig_bsize;
        r_core_seek (core, orig_addr, true);
    }
    return out;
}
```

**Effort:** 1-2 hours
**Risk:** Low

---

## 2. Add r_core_read_at_buf() Helper

**File:** `libr/core/cio.c`

**Purpose:** Allow reading at arbitrary address without modifying core state.

**Implementation:**
```c
R_API int r_core_read_at_buf(RCore *core, ut64 addr, ut8 *buf, int len) {
    R_RETURN_VAL_IF_FAIL (core && buf && len > 0, -1);
    return r_io_read_at (core->io, addr, buf, len);
}
```

**Header:** Add to `libr/include/r_core.h`
```c
R_API int r_core_read_at_buf(RCore *core, ut64 addr, ut8 *buf, int len);
```

**Benefit:** Commands can read data without seeking, reducing block invalidations.

**Effort:** 30 minutes
**Risk:** None (additive)

---

## 3. Add Block Validity Tracking

**File:** `libr/include/r_core.h` (struct r_core_t)

**Add field:**
```c
struct r_core_t {
    // ... existing fields ...
    ut64 block_addr;      // Address that block[] was read from
    bool block_valid;     // Whether block content matches block_addr
    // ... rest of struct ...
};
```

**Update r_core_block_read() in cio.c:**
```c
R_API int r_core_block_read(RCore *core) {
    R_RETURN_VAL_IF_FAIL (core, -1);
    int res = -1;
    R_CRITICAL_ENTER (core);
    if (core->block) {
        res = r_io_read_at (core->io, core->addr, core->block, core->blocksize);
        core->block_addr = core->addr;
        core->block_valid = (res >= 0);
    }
    R_CRITICAL_LEAVE (core);
    return res;
}
```

**Update r_core_seek() to invalidate:**
```c
R_API bool r_core_seek(RCore *core, ut64 addr, bool rb) {
    if (!rb && addr == core->addr) {
        return false;
    }
    if (addr != core->addr) {
        core->block_valid = false;  // Add this
    }
    core->addr = r_io_seek (core->io, addr, R_IO_SEEK_SET);
    if (rb) {
        r_core_block_read (core);
    }
    // ... rest of function
}
```

**Benefit:** Commands can check `core->block_valid` before using cached data.

**Effort:** 1 hour
**Risk:** Low (backward compatible)

---

## 4. Fix Task Console Context Leak

**File:** `libr/core/task.c` (lines 348-354)

**Current Issue:**
```c
if (create_cons) {
    task->cons_context = r_cons_context_clone (core->cons->context);
    if (!task->cons_context) {
        goto hell;
    }
    core->cur_cmd_depth = core->max_cmd_depth;  // Wrong: modifies core state
}
```

**Fix:**
```c
if (create_cons) {
    task->cons_context = r_cons_context_clone (core->cons->context);
    if (!task->cons_context) {
        goto hell;
    }
    // Don't modify core->cur_cmd_depth here - it's shared state
}
```

**Effort:** 10 minutes
**Risk:** Low

---

## 5. Enable CUSTOMCORE in task.c

**File:** `libr/core/task.c` (line 13)

**Current:**
```c
#define CUSTOMCORE 0
```

**Change to:**
```c
#define CUSTOMCORE 1
```

**And fix mycore_new():**
```c
static RCore *mycore_new(RCore *core) {
#if CUSTOMCORE
    // Create a lightweight core shell with isolated console
    RCore *c = R_NEW0 (RCore);
    if (!c) return core;

    // Copy essential pointers (shared, read-mostly)
    c->io = core->io;
    c->anal = core->anal;
    c->flags = core->flags;
    c->bin = core->bin;
    c->num = core->num;
    c->config = core->config;  // TODO: clone for full isolation
    c->rcmd = core->rcmd;

    // Create isolated console
    c->cons = r_cons_new ();
    if (!c->cons) {
        free (c);
        return core;
    }

    // Copy state
    c->addr = core->addr;
    c->blocksize = core->blocksize;
    c->block = malloc (core->blocksize);
    if (c->block) {
        memcpy (c->block, core->block, core->blocksize);
    }

    return c;
#else
    return core;
#endif
}

static void mycore_free(RCore *c) {
#if CUSTOMCORE
    if (c) {
        free (c->block);
        r_cons_free (c->cons);
        free (c);
    }
#endif
}
```

**Benefit:** Tasks get isolated block buffer and console.

**Effort:** 2 hours
**Risk:** Medium (test thoroughly)

---

## 6. Add R_CRITICAL to r_core_block_size()

**File:** `libr/core/core.c`

**Find r_core_block_size() and add locking:**
```c
R_API int r_core_block_size(RCore *core, int bsize) {
    R_RETURN_VAL_IF_FAIL (core, false);
    R_CRITICAL_ENTER (core);

    ut32 obs = core->blocksize;
    if (bsize < 1) {
        bsize = 1;
    } else if (bsize > core->blocksize_max) {
        bsize = core->blocksize_max;
    }

    ut8 *newblock = realloc (core->block, bsize);
    if (!newblock) {
        R_CRITICAL_LEAVE (core);
        return false;
    }

    core->block = newblock;
    core->blocksize = bsize;

    if (bsize > obs) {
        memset (core->block + obs, 0xff, bsize - obs);
    }

    r_core_block_read (core);

    R_CRITICAL_LEAVE (core);
    return true;
}
```

**Effort:** 30 minutes
**Risk:** Low

---

## 7. Document Thread-Unsafe Functions

**File:** Create `libr/core/THREAD_SAFETY.md`

```markdown
# Thread Safety in RCore

## Thread-Unsafe Functions (require main thread)

- `r_core_block_read()` - modifies core->block
- `r_core_block_size()` - resizes core->block
- `r_core_seek()` with rb=true - modifies core->block

## Thread-Safe Functions

- `r_io_read_at()` - can be called from any thread
- `r_core_read_at_buf()` - reads without modifying state (NEW)
- Flag operations - protected by R_CRITICAL

## Guidelines for New Code

1. Prefer `r_io_read_at()` over using `core->block`
2. Don't cache pointers to `core->block` across calls
3. Use `R_CRITICAL_ENTER/LEAVE` when modifying core state
```

**Effort:** 30 minutes
**Risk:** None (documentation)

---

## 8. Audit cmdstr() Config Restoration

**File:** `libr/core/rtr_http.inc.c` (lines 46-79)

**Current Issue:** Config is modified and restored, but if command crashes, restoration is skipped.

**Fix:** Use a cleanup pattern:
```c
static char *cmdstr(RCore *core, const char *cmd) {
    char *out = NULL;
    RConsContext *ctx = core->cons->context;
    ctx->noflush = false;

    // Save all config state upfront
    const bool orig_sandbox = r_config_get_b (core->config, "cfg.sandbox");
    const bool orig_scr_html = r_config_get_b (core->config, "scr.html");
    const int orig_scr_color = r_config_get_i (core->config, "scr.color");
    const bool orig_scr_interactive = r_config_get_b (core->config, "scr.interactive");
    const bool orig_asm_bytes = r_config_get_b (core->config, "asm.bytes");

    // Apply HTTP-specific settings
    if (r_config_get_b (core->config, "http.sandbox")) {
        r_config_set_b (core->config, "cfg.sandbox", true);
    }
    r_config_set_i (core->config, "scr.color", COLOR_MODE_DISABLED);
    r_config_set_b (core->config, "asm.bytes", false);
    r_config_set_b (core->config, "scr.interactive", false);

    // Execute
    out = r_core_cmd_str_pipe (core, cmd);

    // Always restore (even on error)
    r_config_set_b (core->config, "scr.html", orig_scr_html);
    r_config_set_i (core->config, "scr.color", orig_scr_color);
    r_config_set_b (core->config, "scr.interactive", orig_scr_interactive);
    r_config_set_b (core->config, "asm.bytes", orig_asm_bytes);
    r_config_set_b (core->config, "cfg.sandbox", orig_sandbox);

    return out;
}
```

**Effort:** 30 minutes
**Risk:** Low

---

## 9. Remove Unnecessary Block Operations in HTTP Loop

**File:** `libr/core/rtr_http.inc.c`

**Lines 225-231, 275-280 can be simplified:**

The current code saves/restores block on every request iteration. Since we're using `cmdstr()` which doesn't need the block to be persistent across requests, we can remove this overhead.

**Before:**
```c
newoff = core->addr;
newblk = core->block;
newblksz = core->blocksize;

core->addr = origoff;
core->block = origblk;
core->blocksize = origblksz;
```

**After:**
Simply don't swap blocks at all. Let each command read what it needs via `r_io_read_at()`.

**Effort:** 1 hour (requires testing)
**Risk:** Medium (behavioral change)

---

## 10. Add Task Isolation Level Config

**File:** `libr/core/cconfig.c`

**Add new eval variable:**
```c
SETPREF ("tasks.isolation", "snapshot", "Task isolation level (shared, snapshot, isolated)");
```

**In callback:**
```c
static bool cb_tasks_isolation(void *user, void *data) {
    RCore *core = (RCore *)user;
    RConfigNode *node = (RConfigNode *)data;
    if (!strcmp (node->value, "isolated")) {
        r_core_task_set_default_mode (&core->tasks, R_CORE_TASK_MODE_THREAD);
    } else if (!strcmp (node->value, "snapshot")) {
        r_core_task_set_default_mode (&core->tasks, R_CORE_TASK_MODE_THREAD);
    } else {
        r_core_task_set_default_mode (&core->tasks, R_CORE_TASK_MODE_COOP);
    }
    return true;
}
```

**Benefit:** Users can opt-in to more isolation as it becomes stable.

**Effort:** 1 hour
**Risk:** Low

---

## Priority Order

1. **#2 - Add r_core_read_at_buf()** - Foundation for everything else
2. **#3 - Block validity tracking** - Helps detect stale data
3. **#1 - Fix rtr_http.inc.c** - Most visible issue
4. **#6 - Add R_CRITICAL to block_size** - Prevents crashes
5. **#4 - Fix task console context leak** - Memory leak fix
6. **#8 - Audit cmdstr() config** - Correctness fix
7. **#7 - Document thread safety** - Helps contributors
8. **#5 - Enable CUSTOMCORE** - Enables real isolation
9. **#9 - Remove HTTP block swaps** - Performance improvement
10. **#10 - Task isolation config** - User-facing feature

---

## Quick Verification Tests

After implementing changes, run:

```bash
# Basic functionality
r2 -qc 'pd 10; px 32' /bin/ls

# HTTP server parallel requests
r2 -c '=h 9999' /bin/ls &
sleep 1
for i in $(seq 1 10); do
    curl "http://localhost:9999/cmd/pd%2010" &
done
wait

# Background tasks
r2 -qc '&pd 100; &px 100; &?t; ?t-' /bin/ls
```

---

## Files to Modify (Summary)

| File | Changes |
|------|---------|
| `libr/core/cio.c` | Add `r_core_read_at_buf()`, update `r_core_block_read()` |
| `libr/core/core.c` | Add R_CRITICAL to `r_core_block_size()`, init block_valid |
| `libr/core/rtr_http.inc.c` | Simplify block handling, fix cmdstr() |
| `libr/core/task.c` | Fix cons context leak, enable CUSTOMCORE |
| `libr/core/cconfig.c` | Add tasks.isolation config |
| `libr/include/r_core.h` | Add block_addr, block_valid fields, new API |
| `libr/core/THREAD_SAFETY.md` | New documentation file |

---

## Estimated Total Effort

- **Minimal viable changes** (#1, #2, #3, #6): ~4 hours
- **Full now.md implementation**: ~10 hours
- **Testing and stabilization**: ~4 hours

Total: **~18 hours** to significantly improve thread safety without the full architectural overhaul.

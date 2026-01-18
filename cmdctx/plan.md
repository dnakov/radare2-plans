# Plan: Task-Based Command Execution with Isolated State

## Executive Summary

This document outlines a long-term architectural plan to enable thread-safe parallel command execution in radare2. The core problem is that `core->block`, `core->addr`, `core->blocksize`, `core->config`, and `core->cons` are global mutable state shared across all command executions, making parallel execution unsafe.

The solution is to introduce **RCoreTaskContext** - a per-task execution context that captures and isolates all necessary state at command entry, allowing commands to execute independently without corrupting shared state.

---

## Current Architecture Analysis

### Global State in RCore (libr/include/r_core.h:327-442)

The following state is currently shared and mutable:

| Field | Type | Purpose | Thread-Safety Issue |
|-------|------|---------|---------------------|
| `addr` | `ut64` | Current seek position | Modified by seek commands, read by all |
| `blocksize` | `ut32` | Size of cached block | Modified by `b` command |
| `block` | `ut8*` | Cached data at current addr | Read/written by many commands |
| `config` | `RConfig*` | All eval variables | Modified by `e` command, read everywhere |
| `cons` | `RCons*` | Console output context | Contains mutable palette, grep state |
| `io` | `RIO*` | I/O layer | File descriptors shared |
| `anal` | `RAnal*` | Analysis database | Functions, xrefs, hints |
| `flags` | `RFlag*` | Named addresses | Already has R_CRITICAL protection |

### Current Workarounds

#### rtr_http.inc.c (lines 200-231, 275-280)
```c
// Saves and restores block/addr/blocksize per HTTP request
ut64 newoff, origoff = core->addr;
int newblksz, origblksz = core->blocksize;
ut8 *newblk, *origblk = core->block;

newblk = malloc (core->blocksize);
memcpy (newblk, core->block, core->blocksize);
core->block = newblk;

// Later restores:
core->addr = origoff;
core->block = origblk;
core->blocksize = origblksz;
```

This pattern is fragile because:
1. If a command changes blocksize, the saved origblk pointer becomes invalid
2. If addr changes, block contents are stale
3. Config changes leak between requests
4. No protection against concurrent access

#### task.c (lines 334-375, 755-758)
```c
// Creates console context clone but NOT block/config isolation
task->cons_context = r_cons_context_clone (core->cons->context);

// Clone for task is currently a no-op:
static RCore *r_core_clone_for_task(RCore *core) {
    return mycore_new (core);  // Returns core itself when CUSTOMCORE=0
}
```

---

## Proposed Architecture

### Phase 1: RCoreTaskContext Introduction

Create a new struct that captures all mutable state needed for command execution:

```c
typedef struct r_core_task_context_t {
    // Execution identity
    int task_id;
    ut64 start_time;

    // Address space state (captured at entry)
    ut64 addr;              // Captured seek position
    ut32 blocksize;         // Captured block size
    ut8 *block;             // Private block buffer (owned)
    bool block_valid;       // Whether block content is current

    // Config state (captured/cloned at entry)
    RConfig *config;        // Cloned config (owned) or NULL for shared
    bool config_dirty;      // Whether to propagate back to core

    // Console state (already exists in task.c)
    RConsContext *cons_context;

    // I/O state (reference only, not owned)
    RIO *io;               // Shared I/O (reads are safe, writes need care)

    // Propagation flags
    bool propagate_seek;    // Write addr back to core on completion
    bool propagate_config;  // Write config changes back on completion
} RCoreTaskContext;
```

### Phase 2: Context Lifecycle

```c
// Create context at command entry
R_API RCoreTaskContext *r_core_task_context_new(RCore *core, int flags);

// Read data into context's private block
R_API int r_core_task_context_block_read(RCoreTaskContext *ctx);

// Get block data (re-reads if addr changed)
R_API ut8 *r_core_task_context_get_block(RCoreTaskContext *ctx);

// Seek within context (invalidates block, optionally reads)
R_API bool r_core_task_context_seek(RCoreTaskContext *ctx, ut64 addr, bool read_block);

// Merge context changes back to core (called on completion)
R_API void r_core_task_context_commit(RCoreTaskContext *ctx, RCore *core);

// Free context
R_API void r_core_task_context_free(RCoreTaskContext *ctx);
```

### Phase 3: Command Execution Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Command Entry Point                       │
│  (r_core_cmd, r_core_cmd_str, HTTP /cmd/, etc.)             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Create RCoreTaskContext                         │
│  - Capture addr, blocksize from core                         │
│  - Allocate private block buffer                             │
│  - Clone config if isolation requested                       │
│  - Set up console context                                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Read Block Data (lazy or eager)                 │
│  r_io_read_at(ctx->io, ctx->addr, ctx->block, ctx->blocksize)│
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Execute Command with Context                    │
│  - Commands access ctx->block instead of core->block         │
│  - Seek operations update ctx->addr, invalidate ctx->block   │
│  - Config reads/writes go to ctx->config                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              Commit or Discard Context                       │
│  - If propagate flags set, write changes back to core        │
│  - Free private buffers                                      │
└─────────────────────────────────────────────────────────────┘
```

### Phase 4: Thread Safety Levels

Define three isolation levels for task execution:

| Level | Name | Block | Config | Use Case |
|-------|------|-------|--------|----------|
| 0 | `SHARED` | Shared | Shared | Legacy behavior, single-threaded |
| 1 | `SNAPSHOT` | Private copy | Shared | Read-only commands, HTTP server |
| 2 | `ISOLATED` | Private copy | Cloned | Full parallel execution |

```c
typedef enum {
    R_CORE_TASK_ISOLATION_SHARED = 0,
    R_CORE_TASK_ISOLATION_SNAPSHOT = 1,
    R_CORE_TASK_ISOLATION_ISOLATED = 2
} RCoreTaskIsolation;
```

---

## Implementation Phases

### Phase 1: Foundation (Week 1-2)

1. **Define RCoreTaskContext struct** in `r_core_task.h`
2. **Implement context lifecycle functions** in `task.c`
3. **Add context to RCoreTask struct**
4. **Create accessor macros** for transition:
   ```c
   #define CORE_ADDR(core) (core->task_ctx ? core->task_ctx->addr : core->addr)
   #define CORE_BLOCK(core) (core->task_ctx ? core->task_ctx->block : core->block)
   ```

### Phase 2: HTTP Server Migration (Week 3)

1. **Rewrite rtr_http.inc.c** to use RCoreTaskContext
2. **Remove manual block/addr save/restore**
3. **Use SNAPSHOT isolation level**
4. **Test parallel HTTP requests**

### Phase 3: Command Infrastructure (Week 4-6)

1. **Audit all core->block usages** (30 files identified)
2. **Create helper functions** for common patterns:
   ```c
   // Read at specific address into provided buffer
   R_API int r_core_read_at(RCore *core, ut64 addr, ut8 *buf, int len);

   // Read at context address into context block
   R_API int r_core_ctx_block_read(RCore *core);
   ```
3. **Migrate high-frequency commands** first:
   - `pd` (disassembly)
   - `px` (hex dump)
   - `af` (analysis)
   - `s` (seek)

### Phase 4: Config Isolation (Week 7-8)

1. **Implement config cloning for tasks**
2. **Add config merge/propagate logic**
3. **Handle config callbacks** (some trigger side effects)
4. **Test config changes in parallel tasks**

### Phase 5: Full Parallel Execution (Week 9-12)

1. **Enable ISOLATED mode for thread tasks**
2. **Add synchronization for shared resources**:
   - Flags (already has R_CRITICAL)
   - Analysis database
   - Symbols/relocations
3. **Implement resource locking strategy**:
   - Reader-writer locks for anal database
   - Per-resource locks for flags
4. **Performance testing and optimization**

---

## Critical Thread Safety Concerns

### 1. Block Buffer Race Conditions

**Current Problem:**
```c
// Thread A                          // Thread B
r_core_seek(core, 0x1000, true);     r_core_seek(core, 0x2000, true);
// core->block now has 0x2000 data   // Overwrote Thread A's data
memcpy(local, core->block, 32);      // Thread A reads wrong data!
```

**Solution:** Each task has private `ctx->block`, populated on-demand.

### 2. Config Variable Conflicts

**Current Problem:**
```c
// Thread A                          // Thread B
r_config_set_b(core->config,         r_config_set_b(core->config,
    "asm.bytes", false);                 "asm.bytes", true);
r_core_cmd_str(core, "pd 10");       // Uses Thread B's setting!
```

**Solution:** ISOLATED mode clones config; changes don't affect other tasks.

### 3. Seek Position Races

**Current Problem:**
```c
// Thread A                          // Thread B
r_core_seek(core, 0x1000, false);    r_core_seek(core, 0x2000, false);
// Both modify core->addr
r_core_cmd(core, "pd");              // Which address gets disassembled?
```

**Solution:** Each task has `ctx->addr`; seek updates context, not core.

### 4. Analysis Database Corruption

**Concern:** RAnal contains linked lists (functions, xrefs) that aren't thread-safe.

**Solution:**
- Read operations: Use reader-writer locks
- Write operations: Queue mutations, apply in main thread
- Long-term: Make RAnal internally thread-safe

### 5. Console Output Interleaving

**Current Mitigation:** `RConsContext` per task (already implemented in task.c)

**Additional Work:**
- Ensure grep state is per-context
- Ensure color palette is per-context (done via clone)

---

## API Changes Summary

### New Functions

```c
// Context management
R_API RCoreTaskContext *r_core_task_context_new(RCore *core, RCoreTaskIsolation level);
R_API void r_core_task_context_free(RCoreTaskContext *ctx);
R_API void r_core_task_context_commit(RCoreTaskContext *ctx, RCore *core);

// Block operations via context
R_API int r_core_task_context_block_read(RCoreTaskContext *ctx);
R_API ut8 *r_core_task_context_get_block(RCoreTaskContext *ctx);
R_API bool r_core_task_context_block_resize(RCoreTaskContext *ctx, ut32 size);

// Seek via context
R_API bool r_core_task_context_seek(RCoreTaskContext *ctx, ut64 addr, bool read_block);

// Config access via context
R_API const char *r_core_task_context_config_get(RCoreTaskContext *ctx, const char *key);
R_API bool r_core_task_context_config_set(RCoreTaskContext *ctx, const char *key, const char *val);

// Convenience: read at arbitrary address without affecting context state
R_API int r_core_read_at_buf(RCore *core, ut64 addr, ut8 *buf, int len);
```

### Deprecated (Long-term)

```c
// Direct access to core->block should be avoided
// Use r_core_task_context_get_block() or r_core_read_at_buf() instead
```

---

## Files Requiring Changes

### High Priority (core->block users)

1. `libr/core/cmd_print.inc.c` - Heavy block usage for px, pd, etc.
2. `libr/core/cmd_write.inc.c` - Writes to block
3. `libr/core/cmd_cmp.inc.c` - Compares block data
4. `libr/core/visual.c` - Visual mode reads block
5. `libr/core/disasm.c` - Disassembly reads block
6. `libr/core/rtr_http.inc.c` - HTTP server (immediate target)

### Medium Priority (block read calls)

7. `libr/core/cio.c` - `r_core_block_read` implementation
8. `libr/core/cmd_seek.inc.c` - Seek affects block
9. `libr/core/cmd_anal.inc.c` - Analysis reads block
10. `libr/core/panels.c` - Panels read block

### Lower Priority (indirect usage)

11. `libr/core/vmenus.c`
12. `libr/core/canal.c`
13. `libr/core/cbin.c`
14. `libr/core/casm.c`
15. `libr/core/yank.c`

---

## Testing Strategy

### Unit Tests

1. **Context isolation test**: Create two contexts, modify one, verify other unchanged
2. **Block invalidation test**: Seek in context, verify block marked invalid
3. **Commit test**: Modify context, commit, verify core updated
4. **Concurrent read test**: Multiple threads read same address

### Integration Tests

1. **HTTP parallel requests**: Send simultaneous `/cmd/pd 10` and `/cmd/px 10`
2. **Task parallel execution**: Run `&pd 100` and `&px 100` simultaneously
3. **Config isolation**: Set different `asm.bits` in parallel tasks

### Stress Tests

1. **High concurrency**: 100 parallel HTTP requests
2. **Memory pressure**: Large blocksizes in parallel
3. **Long-running**: Sustained parallel load for memory leaks

---

## Migration Path for Existing Code

### Step 1: Introduce Context-Aware Helpers

```c
// Old code:
memcpy(buf, core->block, len);

// Transition code (works with or without context):
ut8 *block = r_core_get_block(core);  // Returns ctx->block or core->block
memcpy(buf, block, len);
```

### Step 2: Update Command Implementations

```c
// Old cmd_pd:
static int cmd_pd(RCore *core, const char *input) {
    r_core_print_disasm(core, core->addr, core->block, core->blocksize, ...);
}

// New cmd_pd:
static int cmd_pd(RCore *core, const char *input) {
    ut64 addr = r_core_ctx_addr(core);
    ut8 *block = r_core_ctx_get_block(core);
    ut32 bsize = r_core_ctx_blocksize(core);
    r_core_print_disasm(core, addr, block, bsize, ...);
}
```

### Step 3: Enable Isolation

```c
// In task creation:
task->ctx = r_core_task_context_new(core, R_CORE_TASK_ISOLATION_SNAPSHOT);

// In HTTP handler:
r_core_task_context_new(core, R_CORE_TASK_ISOLATION_SNAPSHOT);
```

---

## Success Criteria

1. **HTTP server** can handle 10 concurrent requests without data corruption
2. **Background tasks** (`&` prefix) execute with correct isolated state
3. **No memory leaks** in context creation/destruction cycle
4. **Performance** within 10% of current single-threaded execution
5. **All existing tests pass** with new infrastructure

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Callback side effects from config | High | Medium | Audit callbacks, disable in isolated mode |
| RAnal corruption from parallel writes | Medium | High | Queue writes, apply in main thread |
| Memory bloat from block copies | Low | Medium | Lazy block reading, configurable sizes |
| ABI breakage | High | Low | Keep old fields, add context as optional |
| Performance regression | Medium | Medium | Profile, optimize hot paths |

---

## Appendix: Current core->block Usage Patterns

### Pattern 1: Read at current address
```c
// Most common - use block as cache for current seek
r_core_print_hexdump(core, core->addr, core->block, core->blocksize, ...);
```

### Pattern 2: Read at specific address
```c
// Read elsewhere, restore block after
ut64 orig = core->addr;
r_core_seek(core, target, true);
// use core->block
r_core_seek(core, orig, true);
```

### Pattern 3: Transform and write back
```c
// Modify block, write to file
memcpy(buf, core->block, len);
// transform buf
r_io_write_at(core->io, core->addr, buf, len);
r_core_block_read(core);  // refresh cache
```

### Pattern 4: Temporary block allocation
```c
// Need different size
ut8 *tmp = malloc(size);
r_io_read_at(core->io, addr, tmp, size);
// use tmp
free(tmp);
```

All these patterns will work with the new context system, with Pattern 4 being the closest to the new recommended approach.

# Thread Safety in RCore

## Overview

R_CRITICAL macros provide optional locking (enabled via R_CRITICAL_ENABLED).
When enabled, they use r_th_lock_enter/leave on the object's lock field.

## Thread-Safe Functions

- `r_io_read_at()` - stateless read, can be called from any thread
- Flag space operations - r_flag_space_*

## Functions with R_CRITICAL Protection

These use locking when R_CRITICAL_ENABLED is set:

- `r_core_block_read()` - writes to core->block
- `r_core_block_size()` - resizes core->block
- `r_core_seek_size()` - combines seek + resize

## Thread-Unsafe Functions (require main thread)

- `r_core_seek()` - modifies core->addr without lock
- `r_core_seek_delta()` - calls r_core_seek

## Guidelines for New Code

1. Prefer `r_io_read_at()` over using `core->block`
2. Don't cache pointers to `core->block` across calls
3. Use `R_CRITICAL_ENTER/LEAVE` when modifying shared state
4. Check `R_CRITICAL_ENABLED` is set for thread safety

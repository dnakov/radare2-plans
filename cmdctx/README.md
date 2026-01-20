# Command Context

In order to enable background commands, and multi-threading, it is necessary to
eliminate all global variables (spotted via `R_TH_LOCAL`), but more some
global context information like RCons, `core->block` or `core->arch->session` must be
also taken into account because those can change at any time.

The sanest way to refactor that is to make all commands to be executed in a separate
context, that context contains a reference to RCore, and the scheduler will be in
charge of syncing back the changes up to the parent core session using locks, but
at least information like RConfig settings, current address, etc. wont be affecting
other commands if they are executed in parallel (which is pretty common when you run
the background webserver).

It's important to note that we must reduce the amount of uses of RCons, because, right
now rcons is no longer a singleton global variable for all RCore instances... but it's
still tied to each RCore instance. But that's not enough for commands to be executed
in parallel, so one solution would be to first, use RStrBuf to build commands output
before flushing it back to rcons, this way we reduce the ups and downs in that api,
but also we need a way to sync back that data at the end of each command execution.

The proper stack based RConsContext approach is important for r2pipe, RCoreCmdStr,
and visual modes to work well, which works fine, but not for multithreading environments.

Check the rest of documents in this directory to find out the specific detail discussions
to get this refactoring done properly.

--pancake

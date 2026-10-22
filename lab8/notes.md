# Lab 8 – The Same Calls Inside Your Own OS (xv6)

Daniel Nicorici, FAF-243. Full report with outputs: `Lab8_DanielNicorici_FAF-243.pdf`.

## Part 1. run, inside xv6
1. xv6: `fork()`, `exec()`, `wait()`. Linux (Lab 7): `fork()`, `execlp()`, `waitpid()`. The three steps are the same: `fork` copies the process and returns 0 in the child, `exec` replaces the program and returns only on failure, `wait` blocks until the child ends and gives its status. Only the details differ: xv6 `exec` takes a path and an argv array (no PATH search) and `wait` takes just the status pointer.
2. `echo` prints "hello from the child". After a successful `exec` the child no longer runs the code of `run`; it runs `echo`.
3. The status is whatever the child passes to `exit`: the xv6 `run.c` calls `exit(1)` after a failed exec, the Linux one calls `exit(127)`.

## Part 2. pingpong
1. A pipe carries data in one direction only. With a single pipe, the parent could read back its own byte before the child reads it, and the two directions could not be told apart, so each direction needs its own pipe.
2. `read` makes the parent wait: `read(c2p[0], &b, 1)` blocks until the child writes the byte back. In the same way the child's `read` blocks until the ping arrives, which is why "ping" is always printed first.

Optional `primes.c` was not done.

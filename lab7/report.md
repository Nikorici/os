# Lab 7 – Three Small Programs (Linux)

Daniel Nicorici, FAF-243. Full report with outputs: `Lab7_DanielNicorici_FAF-243.pdf`.

## Part 1. fork + exec + wait
1. `perror("exec")` runs only if `execlp` returns, and `execlp` returns only when it fails. With `ls` or `date` the child's program is replaced, so the line never runs; with `nosuchprog` the exec fails and returns.
2. The child prints the PID of the parent (`getppid()`): 98814, the same number the parent printed as its own PID.
3. With `ls` the parent sees status 0; with `nosuchprog` it sees 127, from the child's `exit(127)` after the failed exec.

## Part 2. A zombie, on purpose
1. `ls` has `Z` in the STAT column (`ZN`; the `N` only means my session runs at a lower priority): a zombie, a process that has ended but whose exit status was not collected by its parent yet.
2. After 20 seconds the `[ls] <defunct>` line is gone. `waitpid(pid, &status, 0);` removed it: the parent collected the status and the kernel freed the entry.

## Part 3. Redirection
1. `open` gave descriptor 1, because `open` always returns the smallest free descriptor: 0 is stdin, 1 was just closed, 2 is stderr.
2. `ls` writes to descriptor 1 as always, but after `close(1)` and `open`, descriptor 1 is `out.txt`, and `exec` keeps open descriptors, so the listing goes into the file.
3. Close descriptor 2 before `open` to redirect the errors (`2> err.txt`).

## Part 4. A pipe
1. Yes: both printed 181.
2. `wc` never ends (`ls` finished, `wc -l` stayed in the process table).
3. `wc` waits for end-of-file on the pipe, which comes only when every write end is closed, and the parent still holds `p[1]`.

Optional `pipeline3.c` was not done.

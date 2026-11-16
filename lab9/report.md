# Lab 9 – Watching System Calls with strace (Linux)

Daniel Nicorici, FAF-243. Full report with outputs: `Lab9_DanielNicorici_FAF-243.pdf`.

## Part 1. See the calls of a real command
`ls` prints with the `write` system call, to descriptor 1 (standard output): `write(1, "total 4\n", 8) = 8`. One `ls -l` made 101 openat/read/write/close calls and 314 system calls in total.

## Part 2. A program that calls the kernel directly
Descriptors 0, 1 and 2 are already open when a program starts: standard input, standard output and standard error, inherited from the shell. The kernel gives the smallest free number, so the first file opened gets 3.

## Part 3. Count one kind of call
The handout PDF stops after Part 2, so I counted the `write` calls of `syscalls` with `strace -c -e trace=write`: 4 on a terminal (three `printf` lines + one `write` to the file), but only 2 when the output is redirected, because `printf` then buffers its three lines and sends them in a single `write` at exit.

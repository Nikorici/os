# Lab 2 – Meet the OS You Will Build

**Operating Systems · FCIM / FAF · UTM · 2026-2027**
**Student:** Daniel Nicorici, FAF-243
**Track B:** build your own small OS (xv6)

xv6 source: `mit-pdos/xv6-riscv`, commit `06aad25` (2026-09-26). It was built with
`riscv64-linux-gnu-gcc` 16.2.1 and run on QEMU 11.1.1 (`qemu-system-riscv64`).

## Part 1. Build and boot

```
$ make qemu
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic ...

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$
```

The kernel boots on 3 emulated RISC-V cores ("harts"), then `init` starts the shell.

## Part 2. Using xv6

```
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2441
cat            2 3 35912
echo           2 4 34808
grep           2 6 43336
ls             2 10 41992
sh             2 13 57120
wc             2 17 36880
...
sleep          2 23 34480
console        3 24 0
$ echo hello xv6
hello xv6
$ ls | grep c
cat            2 3 35912
echo           2 4 34808
wc             2 17 36880
sync           2 22 34152
console        3 24 0
$ wc README
48 336 2441 README
```

The `ls` columns are: name, type (1 = directory, 2 = file, 3 = device), inode number and size in bytes.
`console` has type 3: the terminal is a device file, just like `/dev/tty` on Linux.

**(1) Three programs xv6 ships with:** `ls`, `cat`, `grep` (also `echo`, `wc`, `sh`, `mkdir`, `rm`,
`kill`, `ln` and the test programs `usertests` and `forktest`).

**(2) What a pipe needs:**
- **Processes** (`fork` + `exec`): the shell creates two processes, one running `ls` and one
  running `grep`, and both run at the same time.
- **Inter-process communication** (the `pipe` system call plus file descriptors): the kernel creates
  a buffer with a write end and a read end. The shell uses `dup`/`close` to connect the
  standard output of `ls` to the write end and the standard input of `grep` to the read end.

**(3) xv6 shell vs. Linux shell:** for simple commands the xv6 shell works like bash (programs,
pipes, `>`/`<` redirection, `;` and `&`), but it is much more minimal: it has no history, no Tab
completion, no variables and no `*` globbing, and the prompt is just `$ `.

## Part 3. Reading the source

`user/cat.c` reads a file in a loop and writes each chunk to the screen:

```c
while ((n = read(fd, buf, sizeof(buf))) > 0) {
  if (write(1, buf, n) != n) {
```

**(1) System calls used in `user/cat.c`:**
- `open(path, O_RDONLY)`: asks the kernel to find the file by name and give back a file descriptor.
- `read(fd, buf, 512)`: asks the kernel to copy up to 512 bytes from that file into `buf`.
  It returns how many bytes were copied, 0 at end of file, and -1 on error.
- `write(1, buf, n)`: asks the kernel to copy `n` bytes from `buf` to file descriptor 1
  (standard output, the console).
- `close(fd)`: asks the kernel to release the file descriptor.
- `exit(status)`: asks the kernel to end the process and pass the status to the parent's `wait`.

(`fprintf` is not a system call. It is a library function in `user/printf.c` that formats the
text and then calls `write` on descriptor 2.)

**(2) Where `sys_read` is implemented:** in **`kernel/sysfile.c`, line 69** (`sys_read(void)`,
with its return type `uint64` on line 68). `sys_write` is right after it, on line 83.

```
$ grep -n "sys_read\|sys_write" kernel/sysfile.c
69:sys_read(void)
83:sys_write(void)
```

On the user side, `read` is not C code at all. `user/usys.pl` generates a 3-instruction stub:
`li a7, SYS_read` (number 5, from `kernel/syscall.h:6`), then `ecall`, which traps into the
kernel. The kernel looks up that number in the table in `kernel/syscall.c:115`
(`[SYS_read] = sys_read`).

**(3) `kernel/` vs `user/`:** the code in `kernel/` runs in privileged (supervisor) mode with
access to all memory and hardware and provides the services. The code in `user/` runs as ordinary,
isolated programs that can only get those services by asking the kernel through system calls.

## Part 4. My first program: `sleep`

In August 2025 the xv6 authors renamed the `sleep` system call to `pause`. On the current tree
`user/user.h` declares `int pause(int);` and has no `sleep`, so the handout code, which calls
`sleep()`, does not build. My `user/sleep.c` calls `pause()`, and I added `$U/_sleep\` to `UPROGS` in the
Makefile.

```
$ sleep 10
$ sleep
usage: sleep <ticks>
$ sleep abc 2
usage: sleep <ticks>
```

To check that it really waits, I timed `sleep 50`: the prompt came back after **4.96 s**. One
tick in xv6 is about 0.1 s (the timer is reprogrammed every 1,000,000 cycles in
`kernel/trap.c:179`), so 50 ticks ≈ 5 s.

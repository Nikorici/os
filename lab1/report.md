# Lab 1 – The OS as a Resource Manager

**Operating Systems · FCIM / FAF · UTM · 2026-2027**
**Student:** Daniel Nicorici, FAF-243
**Track A:** the real OS, from the outside (Linux)

I ran the lab on my own laptop (Arch Linux) instead of the course VM, so some numbers below
(RAM, process count, disks) are bigger than on the VM.

```
$ whoami
Nico
$ uname -a
Linux Nico 7.0.12-arch1-1 #1 SMP PREEMPT_DYNAMIC Wed, 10 Jun 2026 08:58:02 +0000 x86_64 GNU/Linux
$ uptime
 15:42:21 up  1:15,  1 user,  load average: 3.11, 2.50, 2.34
```

The kernel managing this machine is Linux 7.0.12 for x86_64, and it had been running for 1 hour 15 minutes.

---

## Part 1. Files and directories (file management)

```
$ cd ~/os-lab1 && mkdir demo && cd demo
$ echo "hello operating systems" > note.txt
$ cp note.txt copy.txt
$ mv copy.txt renamed.txt
$ ls -l
total 8
-rw-r--r-- 1 Nico Nico 24 Oct  5 15:42 note.txt
-rw-r--r-- 1 Nico Nico 24 Oct  5 15:42 renamed.txt
$ rm renamed.txt
$ ls -l note.txt
-rw-r--r-- 1 Nico Nico 24 Oct  5 15:42 note.txt
$ chmod 600 note.txt
$ ls -l note.txt
-rw------- 1 Nico Nico 24 Oct  5 15:42 note.txt
$ chmod 644 note.txt
```

**(1) Who owns the files, and which group?**
The owner is my user `Nico` and the group is also `Nico`. Arch creates a private group with
the same name for every user (`id` shows `uid=1000(Nico) gid=1000(Nico)`), and new files get
the user's primary group.

**(2) What do the ten characters mean after `chmod 600`?**
`-rw-------`:
- `-`: it is a regular file (`d` would be a directory, `l` a symbolic link, `c`/`b` a device).
- `rw-`: the owner (Nico) can read and write but not execute.
- `---`: members of the group have no rights.
- `---`: everyone else has no rights.

`600` is octal: 6 = 4 (read) + 2 (write) for the owner, 0 for group, 0 for others. After
`chmod 644` it went back to `-rw-r--r--`, so the group and others can read it again.

**(3) Two directories directly under `/`:**
- `/etc`: the system-wide configuration files (`passwd`, `fstab`, `hostname`, `pacman.conf`, ...).
- `/proc`: not a real directory on disk. The kernel generates it on the fly, with one folder per
  running process plus kernel information (its size shows as 0 in `ls -la /`).

**Interesting output.** In `ls -la /`, `bin`, `sbin`, `lib` and `lib64` are not real directories.
They are symbolic links into `/usr`:

```
lrwxrwxrwx   1 root root     7 Oct 12  2025 bin -> usr/bin
lrwxrwxrwx   1 root root     7 Oct 12  2025 lib -> usr/lib
lrwxrwxrwx   1 root root     7 Oct 12  2025 sbin -> usr/bin
dr-xr-xr-x 560 root root     0 Oct  5  2026 proc
```

---

## Part 2. Processes (process management)

```
$ ps aux | head -4
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0  26424 14864 ?        Ss   14:27   0:01 /usr/lib/systemd/systemd --switched-root --system --deserialize=55 splash
root           2  0.0  0.0      0     0 ?        S    14:27   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        S    14:27   0:00 [pool_workqueue_release]
$ ps aux | wc -l
501
$ top -b -n 1 | head -3
top - 15:42:29 up  1:15,  1 user,  load average: 3.03, 2.50, 2.35
Tasks: 500 total, 1 running, 498 sleep, 1 d-sleep, 0 stopped, 0 zombie
```

Starting a background process, finding it and stopping it:

```
$ sleep 300 &
[1] 239197
$ jobs
[1]+  Running                    sleep 300 &
$ ps -ef | grep sleep
Nico      239197  239194  0 15:42 ?        00:00:00 sleep 300
$ kill 239197
$ jobs
$
```

Looking at it through `/proc`:

```
$ sleep 300 & PID=$!
$ cat /proc/$PID/status | head -8
Name:   sleep
Umask:  0022
State:  S (sleeping)
Tgid:   239201
Ngid:   0
Pid:    239201
PPid:   239194
TracerPid:      0
$ kill $PID
```

**(1) PID 1:** it is `systemd` (`/usr/lib/systemd/systemd`), the init system. It is the first
user-space process the kernel starts at boot. It starts and supervises all the services, it is
the ancestor of every other user process, and it adopts orphaned processes whose parent died.
(PID 2, `kthreadd`, is the parent of the kernel's own threads.)

**(2) How many processes:** `ps aux | wc -l` printed 501. One line is the header, so about
**500 processes** existed. `top` agrees: `Tasks: 500 total, 1 running, 498 sleep`. Many of them are
kernel threads (the ones in `[brackets]`, like `[kworker/...]`), and the rest come from my desktop
session. Almost all of them are sleeping and waiting for something.

**(3) The `State:` line:** `State: S (sleeping)`. The `sleep` process is in an interruptible sleep:
it is waiting for its timer to expire and gets no CPU time until then. `PPid` is the PID of the
shell that started it, so the shell is its parent.

---

## Part 3. Memory (memory management)

```
$ free -h
               total        used        free      shared  buff/cache   available
Mem:            30Gi         9Gi        14Gi       479Mi       6.5Gi        20Gi
Swap:             0B          0B          0B
$ cat /proc/meminfo | head -6
MemTotal:       32124248 kB
MemFree:        15344088 kB
MemAvailable:   21642580 kB
Buffers:          867072 kB
Cached:          5035328 kB
SwapCached:            0 kB
$ sleep 300 & PID=$!
$ grep -E 'VmSize|VmRSS|RssAnon|RssFile' /proc/$PID/status
VmSize:     3216 kB
VmRSS:      2036 kB
RssAnon:     112 kB
RssFile:    1924 kB
$ kill $PID
```

**(1) RAM:** the machine has **30 GiB** of RAM in total (`MemTotal` 32,124,248 kB). **14 GiB** is
completely free, but **20 GiB** is *available*. Available is larger than free because the kernel
uses otherwise idle RAM as a disk cache (`buff/cache` 6.5 GiB), and it can drop that cache
whenever a program needs the memory.

**(2) Swap:** swap is a partition or file on disk where the kernel moves memory pages that
have not been used for a while when RAM gets full. The RAM is freed for other processes, but those
pages become slow to access again. On this laptop **no swap is configured** (`Swap: 0B`), and
`swapon --show` prints nothing.

**(3) VmRSS of `sleep`:** a bare `sleep` has **VmRSS = 2036 kB (~2 MB)** resident in RAM, out of
3216 kB of virtual memory. At first it surprised me that a program that does nothing takes 2 MB.
The breakdown explains it: only `RssAnon: 112 kB` is the process's own private memory, and
`RssFile: 1924 kB` is mapped files, i.e. the code of the `sleep` binary and of the C library
(libc). Those pages are shared with every other process that uses libc, so they are counted in
every process's RSS but exist in physical RAM only once. VmSize is bigger than VmRSS because the
OS loads pages lazily: only the pages that are actually touched get real RAM.

---

## Part 4. Devices and storage (I/O management)

```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p5  192G  152G   31G  84% /
devtmpfs         16G     0   16G   0% /dev
tmpfs            16G   11M   16G   1% /dev/shm
tmpfs            16G  131M   16G   1% /tmp
/dev/nvme0n1p1   96M   86M   11M  90% /boot/efi
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 953.9G  0 disk
├─nvme0n1p1 259:1    0   100M  0 part /boot/efi
├─nvme0n1p2 259:2    0    16M  0 part
├─nvme0n1p3 259:3    0 247.6G  0 part
├─nvme0n1p4 259:4    0 510.8G  0 part
└─nvme0n1p5 259:5    0 195.3G  0 part /
$ du -sh ~/os-lab1
12K     /home/Nico/os-lab1
$ ls -l /dev/null /dev/nvme0n1 /dev/tty /dev/zero /dev/random
crw-rw-rw- 1 root root   1, 3 Oct  5  2026 /dev/null
brw-rw---- 1 root disk 259, 0 Oct  5  2026 /dev/nvme0n1
crw-rw-rw- 1 root root   1, 8 Oct  5  2026 /dev/random
crw-rw-rw- 1 root tty    5, 0 Oct  5 15:38 /dev/tty
crw-rw-rw- 1 root root   1, 5 Oct  5  2026 /dev/zero
$ mount | head -3
/dev/nvme0n1p5 on / type ext4 (rw,relatime)
devtmpfs on /dev type devtmpfs (rw,nosuid,size=15872424k,nr_inodes=3968106,mode=755,inode64,huge=within_size)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,inode64,huge=within_size,usrquota)
```

**(1) Root filesystem:** `/` is mounted from **`/dev/nvme0n1p5`**, partition 5 of the NVMe SSD
`nvme0n1` (953.9 GB). It is an ext4 filesystem of 192 GB, 84% used.

**(2) One entry from `/dev`:** `/dev/nvme0n1` is the whole NVMe SSD. The `b` at the start means
it is a *block device* (read and written in blocks), and instead of a size `ls` shows the numbers
`259, 0` (major = which driver, minor = which device). Another one is `/dev/null`, a *character
device* (`c`, numbers `1, 3`) that is not real hardware at all: it throws away everything
written to it.

**(3) "Everything is a file":** the kernel shows disks, terminals, the random number generator,
and even processes (`/proc`) as entries in the same file tree, so the same tools (`ls`, `cat`) and
the same system calls (`open`, `read`, `write`) work on all of them.

**Interesting output.** `mount` shows that `/dev`, `/dev/shm` and `/tmp` are `devtmpfs`/`tmpfs`,
filesystems that live only in RAM and disappear at reboot.

---

## Conclusion

The operating system manages four resources: files, processes, memory and devices. I saw files
with `ls -l` (owners and permissions), processes with `ps aux`, memory with `free -h`, and devices
with `lsblk`. Each command is a different view of the same kernel, which shares these resources
between programs and protects them from each other.

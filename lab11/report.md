# Lab 11 – The Race and the Semaphore (Linux)

Daniel Nicorici, FAF-243. Full report with outputs: `Lab11_DanielNicorici_FAF-243.pdf`.

## Part 1. See the race
1. `counter++` is three steps: load the value from memory into a register, add 1 in the register, store the register back to memory.
2. Both threads load the same value (say 5), both add 1 and both store 6. Two additions were done, but the counter grew by one: one addition is lost.
3. The scheduler interleaves the two threads differently at every run, so a different number of additions collide. I got 1100695, 1022820 and 1076745.

## Part 2. Fix it: mutex, then semaphore
1. Yes: `mutex` and `sem` printed 2000000 in all three runs each.
2. With `sem_init(&s, 0, 2)` two threads may be inside at once, and there are only two threads, so nobody ever waits: the race is back (1978631, 1988948, 1982592).
3. Every addition now also pays for a lock and an unlock, and a thread that finds the lock taken has to wait: race 0.004 s, mutex 0.032 s, sem 0.051 s.

## Part 3. A semaphore as a signal
1. With start value 1, B does not wait: `sem_wait` finds 1 and passes at once, so B reads an empty message (`B: I read ""`) before A has written it.
2. A semaphore that starts at 0 means "the event has not happened yet": whoever waits is blocked until someone posts.

## Part 4. A semaphore as a counter
1. The largest value of `inside` is 3. `sem_wait(&places);` guarantees it (with `sem_init(&places, 0, PLACES)`): a fourth car blocks until a place is given back.
2. With PLACES = 1 the semaphore becomes a mutex (a binary semaphore): one car at a time, `inside` never above 1.

## Optional: atomic
`atomic` gives 2000000 with no lock, in 0.016 s (mutex: 0.032 s).

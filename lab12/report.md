# Lab 12 – Producer, Consumer and the Other Tools (Linux)

Daniel Nicorici, FAF-243. Full report with outputs: `Lab12_DanielNicorici_FAF-243.pdf`.

## Part 1. Producer and consumer
1. `sem_wait(&empty)` stops the producer: `empty` starts at 4, and at 0 the producer blocks until the consumer posts.
2. The consumer waits at the start, at `sem_wait(&full)`: `full` starts at 0 because the buffer is empty.
3. With the mutex locked before `sem_wait(&empty)`, the program prints put 1 to put 4 and hangs (exit code 124 under `timeout 5`). The producer blocks on `empty` while holding the mutex; the consumer needs that mutex to take an item and free a slot. Each waits for the other: a deadlock.

## Part 2. The condition variable
1. The ordinary variable `ready` remembers it; the condition variable has no memory.
2. Yes, with `if` it still works here: same three lines of output.
3. `while` re-checks the condition after waking. A thread can wake with `ready == 0` because of a spurious wakeup, or because another waiter woke first and used up the condition, or because the signal was sent for a different change.

## Part 3. Readers and writers
1. `if (++readers == 1) sem_wait(&room);` and `if (--readers == 0) sem_post(&room);`: only the first reader takes the room and only the last gives it back, so readers share it while a writer needs it alone.
2. `readers++` is a race like `counter++` in Lab 11. Without `rlock` two readers could both see 1 (the second blocks forever on `room`) or the count could be wrong, so the room is never released or is released while a reader is still inside.
3. `readers` never returns to 0, so `room` is never posted and writer 2 waits forever: starvation.

## Part 4. Build a deadlock, then remove it
1. `one` holds A and waits for B; `two` holds B and waits for A. Circle: one -> B -> two -> A -> one.
2. Without the sleeps, 9 of 10 runs hung and 1 finished. It is worse because the bug then appears only sometimes: a test can pass and the program still hangs later.
3. If every thread takes A before B, a thread that waits for B never holds something that the owner of B needs, so no circle can form.

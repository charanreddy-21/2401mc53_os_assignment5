#2401mc53
# OS Assignment 5 — xv6 Synchronization

## Process Synchronization Using xv6
---

# Task 1: Peterson's Algorithm

## Implementation Details

Peterson's algorithm provides **mutual exclusion** between two processes using only shared memory variables (`flag[]` and `turn`), without relying on hardware or kernel locks.

### Initialization

The parent process calls `shm_get()` to create a shared physical page and initializes:

```text
flag[0] = 0
flag[1] = 0
turn = 0
shared_counter = 0
```

After `fork()`, the child process attaches to the same shared memory page using `shm_get()`.

### Entry Protocol

Process `i`:

1. Sets `flag[i] = 1` to signal its intention to enter the critical section.
2. Sets `turn = other` to give priority to the other process.
3. Waits in a spin-loop while the other process also wants to enter and it is the other process's turn.

### Critical Section

The process increments the shared counter:

```c
shared_counter++;
```

Execution messages are also printed to observe the behavior of both processes.

### Exit Protocol

After completing the critical section, process `i` resets:

```c
flag[i] = 0;
```

This allows the other process to enter the critical section.

## Result Verification

Both the parent and child execute **10 loops**.

Therefore, the expected final value is:

```text
shared_counter = 20
```

The final value of `shared_counter` is exactly **20**, confirming that both processes enter the critical section one at a time without race conditions.

---

# Task 2: Producer-Consumer (Bounded Buffer)

## Implementation Details

A shared circular queue of size **5** (`struct pcbuf`) is allocated on the shared page.

Synchronization is controlled using three counting semaphores:

| Semaphore | Initial Value | Purpose |
|-----------|:-------------:|---------|
| `empty` | 5 | Tracks available free slots in the queue |
| `full` | 0 | Tracks filled slots ready for consumption |
| `mutex` | 1 | Guarantees exclusive access to buffer insertion and removal |

The parent creates all semaphores before calling `fork()`.

The child automatically receives the semaphore descriptors since `fork()` duplicates the process's memory and file descriptor state.

### Producer

The producer waits using:

```c
sem_wait(empty);
sem_wait(mutex);
```

It inserts an item into the buffer and then signals:

```c
sem_signal(mutex);
sem_signal(full);
```

If the buffer contains 5 items, the producer blocks on `sem_wait(empty)` until a free slot becomes available.

### Consumer

The consumer waits using:

```c
sem_wait(full);
sem_wait(mutex);
```

It removes an item from the buffer and then signals:

```c
sem_signal(mutex);
sem_signal(empty);
```

If the buffer is empty, the consumer blocks on `sem_wait(full)` until an item is produced.

## Result Verification

- The producer blocks whenever the buffer holds **5 items**.
- The consumer blocks whenever the buffer holds **0 items**.
- Only one process accesses the buffer at a time.
- Data items pass through the queue sequentially without missing elements, duplicates, or corrupted ordering.

---

# Task 3: Readers-Writers Problem

## Implementation Details

We implement a variant of the **First Readers-Writers algorithm** that adds a **turnstile** lock to eliminate writer starvation.

Three synchronization mechanisms are used:

### `read_mutex`

Serializes updates to the `read_count` counter.

This ensures that multiple readers cannot modify `read_count` simultaneously.

### `rw_mutex`

Controls access to the shared resource.

- The **first active reader** acquires `rw_mutex`.
- Multiple readers can read concurrently.
- The **last active reader** releases `rw_mutex`.
- Writers acquire `rw_mutex` exclusively.

### `turnstile`

Acts as an entry gate for both readers and writers.

Readers pass through the turnstile without holding it during their entire read operation. Waiting writers can hold the turnstile, preventing new readers from continuously entering.

This prevents incoming readers from starving waiting writers.

## Result Verification

Testing with **3 readers and 2 writers** shows that:

- Several readers can read the shared memory concurrently.
- Writers receive completely exclusive access.
- No writer writes while readers are active.
- No two writers write simultaneously.
- Waiting writers are not indefinitely starved by incoming readers.

---

# Task 4: Dining Philosophers Problem

## Implementation Details

The **5 chopsticks** are modeled using **5 binary semaphores**, each initialized to `1`.

Every philosopher runs as an independent process created using:

```c
fork();
```

This problem does not require `shm_get()` because each process only needs its respective left and right semaphore IDs.

## Deadlock Prevention

To prevent circular wait and deadlocks, we use an **asymmetric resource allocation strategy**.

### Even-Indexed Processes

Even-indexed philosophers acquire their chopsticks in the following order:

```text
Left → Right
```

### Odd-Indexed Processes

Odd-indexed philosophers acquire their chopsticks in the following order:

```text
Right → Left
```

This breaks the circular-wait condition where every philosopher could hold one chopstick while waiting indefinitely for the second.

## Philosopher States

Each philosopher repeatedly goes through the following sequence:

```text
THINKING → HUNGRY → EATING → THINKING
```

Each philosopher completes **5 full cycles**.

## Result Verification

All 5 philosopher processes complete their required cycles successfully.

The implementation demonstrates that:

- All philosophers finish execution.
- No deadlock occurs.
- No philosopher remains permanently blocked.
- Chopsticks are accessed exclusively.
- The system continues to make progress until all philosophers finish.

---

## Conclusion

This assignment demonstrates the implementation of classic **process synchronization algorithms** in the xv6 operating system.

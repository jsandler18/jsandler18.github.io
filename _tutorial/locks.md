---
layout: page
title:  Part 9 - Locks
---

Since our kernel now has concurrency, we are going to need to provide facilities to prevent data races.  There are many ways to do synchronization.  I will focus on two:
spin locks and mutexes.

If you want to play with the code for yourself, [see my git repo](https://github.com/jsandler18/raspi-kernel/tree/0b93bb87ba743d1d8c60205131ca83b992c4a0e9).

## The Atomic Swap
Both spin locks and mutexes rely on a "lock variable", a variable that is 1 when the lock is free, and 0 when taken.  The most important part of a lock implementation is
making sure you cannot be preempted while trying to take the lock.  For example, if you have the following code:
``` c
if (lock == 1)
    lock = 0
```
You can potentially be preempted between the if statement and the assignment.  We can avoid this if we essentially check if we got the lock after we take it, instead of
checking if we can take it and then taking it.
We can do this using an *atomic swap*.  We can swap the lock variable with 0 without being preempted.  If the value we get from the swap is still 0, that means that
someone else had the lock, and we did not get it.  We swapped 0 for 0, so nothing happens.  If, however, we swap out 0 and we get a 1 back, that means the lock was
available and we took it.  We left a 0 in its place, indicating that the lock is taken.

We can only access this atomic swap through assembly.  Here is the code:

**lock_asm.S**
```
.section ".text"

.global try_lock

// This function takes a pointer to a lock variable and uses atomic operations to aqcuire the lock.
// Returns 0 if the lock was not acquired and 1 if it was.
try_lock:
    mov     r1, #0
    swp     r2, r1, [r0]    // stores the value in r1 into the address in r0, and stores the value at the address in r0 into r2
    mov     r0, r2
    blx     lr 
```

Now we can use this in both spin locks and mutexes!

## Spin Locks
A *Spin Lock* is the most basic synchronization method possible.  It allows only one process at a time to hold the lock.  The implementation is very simple: try to acquire the lock in a loop until it is acquired.
This looping is what gives this lock its name.  The lock "spins" in this acquire loop, burning CPU cycles while waiting.

Because a spin lock burns CPU cycles, they are somewhat wasteful, and their primary use case is if you expect the resource will become available very soon.  This is fine
for accessing kernel data structures that are always in memory, but is terrible for synchronizing around network or disk access.

Here is the code:
``` c
typedef int spin_lock_t;
...
void spin_init(spin_lock_t * lock) {
    *lock = 1;
}

void spin_lock(spin_lock_t * lock) {
    while (!try_lock(lock));
}

void spin_unlock(spin_lock_t * lock) {
    *lock = 1;
}
```

## Mutexes
A *Mutex* is similar to a spin lock in that in only allows one process to hold it at a time.  It differs in that its implementation is more complicated, but allows the lock to be held for extended periods without burning CPU cycles.

It does this by keeping a queue of processes that try to lock it.  Instead of spinning, the mutex will add the process that wants the lock to its wait queue, and then schedule a new process without adding the current process back into the run queue.  This way, the process that wants the lock will never run while it is waiting, and thus will not burn CPU cycles.  When a process releases the lock, it takes a process of the wait queue and adds it back into the run queue so that it may claim the lock.

Here is the code:
``` c
typedef struct {
    int lock;
    process_control_block_t * locker;
    pcb_list_t wait_queue;
} mutex_t;

...
void mutex_init(mutex_t * lock) {
    lock->lock = 1;
    lock->locker = 0;
    INITIALIZE_LIST(lock->wait_queue);
}

void mutex_lock(mutex_t * lock) {
    process_control_block_t * new_thread, * old_thread;
    // If you don't get the lock, take self off run queue and put on to mutex wait queue
    while (!try_lock(&lock->lock)) {

        // Get the next thread to run.  For now we are using round-robin
        DISABLE_INTERRUPTS();
        new_thread = pop_pcb_list(&run_queue);
        old_thread = current_process;
        current_process = new_thread;

        // Put the current thread back of this mutex's wait queue, not on the run queue
        append_pcb_list(&lock->wait_queue, old_thread);

        // Context Switch
        switch_to_thread(old_thread, new_thread);
        ENABLE_INTERRUPTS();
    }
    lock->locker = current_process;
}
void mutex_unlock(mutex_t * lock) {
    process_control_block_t * thread;
    lock->lock = 1;
    lock->locker = 0;


    // If there is anyone waiting on this, put them back in the run queue
    if (size_pcb_list(&lock->wait_queue)) {
        thread = pop_pcb_list(&lock->wait_queue);  
        push_pcb_list(&run_queue, thread);
    }
}
```

## Using Locks
You can test the mutex by declaring a global mutex in `kernel.c`, initializing it in `kernel_main`, and adding the following code to the loops of `kernel_main` and `test`:
``` c
if (i % 10 == 0)
	mutex_lock(&test_mut);
else if (i % 10 == 9) 
	mutex_unlock(&test_mut);
```
Instead of alternating `main x` and `test x` every line, it should now alternate every 10 lines.

This is a neat test, but to get our money's worth, we can add synchronization to the list implementation, so all kernel lists will be safe from data races! We must use a spin lock for this, as the list implementation is used inside of mutexes, and we can't have that kind of circular definition.


**Previous:**
[Part 8 - Processes](/tutorial/process.html)

**Next:**
[Part 10 - Virtual Memory](/tutorial/vmem.html)

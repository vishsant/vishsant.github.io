+++
date = "2026-07-12"
draft = false
title = "Your Linux Mutex Is Not a Kernel Object"
+++

Ask most engineers what a mutex is, and you get some version of the same picture: a thing the kernel owns, a gate you ask the operating system to open and close, a privileged object that arbitrates who runs. Locking is "asking the kernel for the lock," unlocking is "telling the kernel you're done."

I believed that picture for years, and it is wrong in the case that matters most: the case that runs millions of times a second in every threaded program you've ever shipped. For an uncontended lock, the kernel is never involved at all. There is no syscall, no context switch, no transition into kernel mode. The mutex is not a kernel object. It is a 32-bit integer in your own address space, and locking it is a single atomic instruction.

## The Number

A modern Linux mutex is built on a primitive called a futex - a fast userspace mutex. The name is the whole design compressed into two words. "Userspace," because the lock state lives in your memory, not the kernel's. "Fast," because the common path never leaves userspace.

A pthread_mutex_t contains a small integer field, and the lock protocol is a convention about that integer's value. The canonical scheme uses three states:
- value 0: unlocked.
- value 1: locked, NO waiters.
- value 2: locked, AND ≥1 thread is asleep in the kernel.

But the happy path needs only two: 0 means unlocked, 1 means locked. Taking an uncontended lock is one atomic compare-and-swap - on x86 this compiles down to a lock cmpxchg.

If the word was 0, the swap succeeds, the word is now 1, and you hold the lock. No futex() call happened. The kernel scheduler never entered the picture. The kernel, in fact, has no idea this lock exists - there is no struct mutex registered anywhere in kernel memory, no handle of any kind. You never told it, and on this path you never will. Unlocking the uncontended case is the mirror image: one atomic exchange dropping the word back to 0. Still no syscall.

## The Demo

You can check this behavior by writing simple programs, then trace using strace.

```c
/* uncontended.c */
pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
for (long i = 0; i < 10000000; i++) {
    pthread_mutex_lock(&m);
    pthread_mutex_unlock(&m);
}
```

```console
$ strace -f -e trace=futex ./uncontended
+++ exited with 0 +++
```

Ten million lock/unlock pairs, and strace shows zero futex syscalls. The kernel never ran a single instruction on this lock's behalf. Now introduce a fight.

```c
/* contended.c */
#define THREADS 4
/* per thread */
#define ITERS   200000L

static pthread_mutex_t m = PTHREAD_MUTEX_INITIALIZER;
/* shared state, so the lock actually matters */
static long counter = 0;

static void *worker(void *arg)
{
    (void)arg;
    for (long i = 0; i < ITERS; i++) {
        pthread_mutex_lock(&m);
        counter++;
        pthread_mutex_unlock(&m);
    }
    return NULL;
}

int main(void)
{
    pthread_t t[THREADS];

    for (int i = 0; i < THREADS; i++)
        pthread_create(&t[i], NULL, worker, NULL);
    for (int i = 0; i < THREADS; i++)
        pthread_join(t[i], NULL);

    printf("done: %d threads x %ld = %ld acquisitions (counter=%ld)\\n",
           THREADS, ITERS, (long)THREADS * ITERS, counter);
    return 0;
}
```

Four threads hammer the same mutex in a tight loop, 200,000 lock/unlock pairs each:

```console
$ strace -f -e trace=futex ./contended 2>&1 | head
[pid 17925] futex(0x5fdf6af88060, FUTEX_WAIT_PRIVATE, 2, NULL <unfinished ...>
[pid 17924] futex(0x5fdf6af88060, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
[pid 17925] <... futex resumed>)  = -1 EAGAIN (Resource temporarily unavailable)
[pid 17924] <... futex resumed>)  = 0
...
```

Now the futex() calls appear and notice the value in the FUTEX_WAIT call is 2, not 1. Same code, same mutex, same API. The only thing that changed is whether two threads wanted the number at the same time. The kernel didn't get involved because you locked a mutex. It got involved because you lost a race for one.

## The Third State

So why does the waiting thread pass 2 to the kernel, when "locked" was supposed to be 1? Because a naive two-state lock has an expensive flaw, and the fix is the entire reason futexes are fast.

Think about unlocking. If the lock word is just 0 or1, the unlocker has no way to know whether anyone is asleep waiting for it. To be safe, it would have to make a FUTEX_WAKE syscall on every unlock, just in case a waiter exists - paying the kernel tax even when nobody is contending. That would defeat the whole point. Ulrich Drepper's classic "Futexes Are Tricky" lays out the fix: add a third state.

A thread that has to block first sets the word to 2 ("I'm going to sleep, somebody owes me a wake-up") and only then calls FUTEX_WAIT. And the unlocker's logic becomes: drop the word to 0, and only make the FUTEX_WAKE syscall if the value it just cleared was 2. If it was 1, there are provably no waiters, so it returns in pure userspace. You pay the syscall exactly when there is someone to wake, and never otherwise.

## When the Kernel Wakes Up

When a thread finally does call futex(uaddr, FUTEX_WAIT, 2, NULL), what does the kernel do with it? It does not learn about your lock as a lasting object. It does one transient thing: it parks the calling thread on a wait queue keyed by the address of the futex word.

Inside the kernel, futex() hashes that userspace address into a hash table of wait queues. The kernel is not holding "the mutex." It is holding a thread, indexed by a number's location in your memory.

When some other thread later calls FUTEX_WAKE on the same address, the kernel looks up that bucket, finds the parked waiters, and makes one runnable again. The lock itself stayed in userspace the entire time; only the blocked thread ever lived in the kernel, and only for as long as it was asleep.

There's a subtle race the design has to close, and it's worth seeing because it explains the odd signature of FUTEX_WAIT. Between a thread deciding "the lock is taken, I'll sleep" and the kernel putting it to sleep, the holder might release the lock and if the kernel simply slept the thread, it would lose that wake-up forever. So FUTEX_WAIT takes the expected value as an argument, and the kernel performs the check and the sleep atomically: it sleeps the thread only if *uaddr is still 2 at the moment it holds the bucket lock. If the value already changed, the kernel refuses the nap; it returns EAGAIN and the thread loops back to try the lock again.

```text
 pthread_mutex_lock(&m)
                         │
                  lock cmpxchg 0 → 1
                         │
              ┌──────────┴──────────┐
              │                     │
          success               already locked
              │                     │
              ▼                     ▼
       return immediately     mark word = 2
         (userspace)               │
                                   ▼
                        futex(FUTEX_WAIT, 2)
                                   │
                                   ▼
                    kernel queues sleeping thread
                      on wait queue keyed by &word
                                   │
                                   │
                 ... another thread unlocks mutex ...
                                   │
                                   ▼
                        futex(FUTEX_WAKE, 1)
                                   │
                                   ▼
                    kernel removes one waiter
                     from the wait queue and
                        makes it runnable
                                   │
                                   ▼
                     scheduler eventually runs it
                                   │
                                   ▼
                           retry lock cmpxchg
```

And that's the whole story of a mutex: you don't pay for the lock. You pay for the fight.

+++
date = "2026-07-26"
draft = false
title = "How epoll() Really Works on Linux"
+++

Every engineer who has written a network server learns the same three-step history. You start with select(), which is old and commonly limited to 1024 descriptors on Linux. You graduate to poll(), which removes select()'s fixed fd-set limit. Then, when you're serious, you move to epoll, which is the fast one. Three tools, one job, increasing quality. It is a clean story and almost everyone carries some version of it.

The story is a bit misleading at the third step. epoll is not a faster way to do what poll() does. It changes who is responsible for noticing that something happened.

## The Problem

poll() is like an interrogation. You build an array of descriptors, hand the whole array to the kernel, and ask which ones are ready. The kernel does not keep a persistent interest list from your previous call, so every invocation must rediscover readiness from the descriptors you provide. So it does the only thing available to it: it walks the array, calls each file's poll() method, collects the readiness bits, and hands the result back.

The cost is structural. If you are watching 20,000 sockets, the kernel touches 20,000 files to answer you, and it does that on every single call. It does it when 500 sockets are ready and it does it when none are, because the only way to learn that nothing happened is to check everything. Worse, the array crosses the user/kernel boundary each time.

This is a pull model. The kernel is passive and you are the one doing the asking, so the cost scales with the size of what you are watching, not with what actually happened.

## The Setup

epoll inverts the direction. The important work does not happen inside epoll_wait(). It happens earlier, when you add a file descriptor.

When you call:

```c
epoll_ctl(epfd, EPOLL_CTL_ADD, socket_fd, &event);
```

you are not asking, "Is this socket ready right now?".

You are telling the kernel, "Watch this socket. If something interesting happens later, tell me.".

To understand how that works, we need to look at what the underlying file object already has:  a wait queue. A wait queue is simply a list of things that want to be notified when some event happens. For a socket, that event might be new data arrived, the connection closed, the socket became writable etc.

Normally, a process waiting for data would attach itself to that wait queue:

```text
socket wait queue

    |
    +── sleeping process
```

When data arrives, the kernel wakes that process.

epoll uses the same notification mechanism but changes what gets attached. Instead of adding the process itself, epoll adds a callback:

```text
socket wait queue

    |
    +── ep_poll_callback()
```

When the socket changes state, the kernel invokes that callback.

Now let's see how the kernel installs it.

Inside `fs/eventpoll.c`:

```c
init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
```

After this setup finishes, epoll leaves. The descriptor now has a notification path back to the epoll instance.

## The Notification Path

From that point the relationship runs backwards compared to poll().

When data arrives on a socket, the socket implementation does what it always did: it wakes its own wait queue. It has no idea epoll exists. But attached to that queue is now ep_poll_callback.

The comment above it in `fs/eventpoll.c` describes the design:

```c
/*
 * This is the callback that is passed to the wait queue wakeup
 * mechanism. It is called by the stored file descriptors when they
 * have events to report.
 */
static int ep_poll_callback(wait_queue_entry_t *wait,
                            unsigned mode,
                            int sync,
                            void *key)
```

The important part is:

"called by the stored file descriptors when they have events to report."

The file object wakes its wait queue, and epoll's callback is one of the listeners attached to that queue. The same callback is used for every watched descriptor. The context attached to the callback tells epoll which descriptor caused the wakeup and which epoll instance is watching it.

Inside the callback:

```c
/* Is this descriptor already in the ready list? */
} else if (!ep_is_linked(epi)) {
    /* Add event to ready list. */
    list_add_tail(&epi->rdllink, &ep->rdllist);
    ep_pm_stay_awake_rcu(epi);
}
```

The first thing epoll does is check whether this descriptor is already waiting to be delivered. Because the same descriptor can trigger multiple wakeups before the application gets a chance to process it. Once the descriptor is added, epoll now has a ready separate list containing only descriptors that have reported activity. The ready list becomes the bridge between notification and delivery.

Only after this does epoll wake threads that are currently blocked inside the kernel implementation of epoll_wait().

But waking the application is only half the job.

## The Event Path

The wakeup does not return an event immediately.

It only says: "Something changed. Check the ready list."

It has not yet told the application which descriptor changed, whether it is readable or writable etc. That happens after epoll_wait() wakes up.

The application was sleeping here:

```c
epoll_wait(epfd, events, maxevents, timeout);
```

When ep_poll_callback() adds a descriptor to the ready list, it wakes the sleeping process. But the process is still inside the kernel. epoll_wait() resumes and processes the descriptors that are already known to be ready.

This is where ep_send_events() runs. It walks the ready list, not the complete set of registered descriptors. If two sockets are there in the ready list, other 19,998 descriptors are never touched. For each ready descriptor, epoll checks the current state and builds the event returned to userspace. This verification matters because the state can change between notification and delivery.

The socket may no longer be readable when the application receives control. So the callback records interest, and ep_send_events() confirms the current state before returning the result.

Only then does epoll_wait() copy the events back to userspace.

That separation is why epoll scales. The kernel does not repeatedly inspect everything you are watching. It waits for notifications, records them, and processes only the descriptors that already reported activity.

## The Full Chain

```text
 REGISTRATION

epoll_ctl(ADD)
      |
      v
vfs_poll(file)
      |
      v
    driver ->poll()
      |
      v
    poll_wait()
      |
      v
    ep_ptable_queue_proc()
      |
      v
attach ep_poll_callback()
to driver's wait queue

      |
      v

[ epoll walks away ]


                 NOTIFICATION

data arrives
      |
      v
driver wake_up()
      |
      v
ep_poll_callback()
      |
      v
add descriptor to
epoll ready list
      |
      v
wake epoll_wait()
      |
      v
ep_send_events()
      |
      v
return only ready descriptors
```

The reason this matters beyond one syscall is that the same inversion keeps reappearing wherever a system has to scale its attention. poll() and epoll are not fast and slow versions of one idea. They are pull and push, and the difference between them is who holds the responsibility for noticing. In a pull model, the watcher keeps asking, so cost grows with the size of what it watches. In a push model, the watched objects report, so cost grows with what actually happened.

Once you have the shape, you find it everywhere. Interrupts against polling a device register. inotify against re-stating a directory tree. Webhooks against cron-polling an API.

epoll inherited the name of the interface it replaced, not the mechanism that made it scale. Four letters accidentally preserved the old mental model: that the kernel is constantly checking. It is not. The scalable design was never faster polling. It was removing the need to poll.

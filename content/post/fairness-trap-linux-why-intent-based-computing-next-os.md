+++
date = "2026-02-19"
draft = false
title = "The Fairness Trap in Linux: Why Intent-Based Computing Is the Next OS Revolution"
+++

Linux wants you to believe it is a wise, neutral arbiter.

It manages the chaos of thousands of threads and brings order to them. It is fair. It is smart. It knows what your applications need better than you do.

This trust was well-placed for decades.

The algorithms inside the Linux kernel are marvels of engineering. They allow a single machine to serve as a web server, a database, and a desktop workstation simultaneously. They slice time so finely that you believe everything happens at once.

But this convenience comes at a cost we rarely measure.

The pursuit of general-purpose fairness has created a ceiling on performance. In our quest to treat every task with equal weight, we have forgotten that not all tasks matter equally.

A database holding a critical lock is not the same as a background backup script. A game rendering the next frame is not the same as a text editor waiting for a keystroke. Yet to the standard scheduler, they are just process identifiers. Just numbers in a red-black tree waiting for their turn.

This piece is about that disconnect.

It is about the illusion that the operating system understands your intent. It is about the physical reality when that illusion breaks. And it is about a quiet revolution happening right now in the kernel that is finally giving us the power to tell the machine exactly what we mean.

## The Traffic Controller

Imagine a city with a single lane of traffic. This lane must serve everyone: ambulances, delivery trucks, commuters, tourists.

Now imagine a traffic controller at the only intersection. He has a strict moral code. Every driver must be treated exactly the same. No special treatment. No cutting the line.

He allows the ambulance to drive for ten seconds. Then he stops it. He waves the tourist forward for ten seconds. Then the delivery truck. Then the commuter.

The ambulance driver screams that he has a dying patient. The controller checks his clipboard. The ambulance has already used its fair share of the road this minute. He points to the tourist. It is the tourist's turn.

This sounds absurd. In the real world, we understand priority. We know the value of using the road changes based on what you are carrying.

But inside your CPU, this is exactly what happens.

Your processor has a limited number of cores. These are the lanes. Your operating system has a scheduler. This is the controller. For the last fifteen years, the dominant philosophy has been fairness.

The scheduler looks at your threads. It does not see a video encoder or a database query. It sees a generic entity that consumes time. It tracks how long each entity has run. It keeps a ledger. If a task has run for too long relative to its peers, it is preempted. Pulled off the core to let someone else run.

This creates the illusion of multitasking. By switching rapidly between tasks, the computer tricks you. It makes you believe music is playing while you type while a file downloads. It feels simultaneous.

It is not simultaneous. It is sequential. And the logic deciding that sequence is blind to the urgency of your work.

We accepted this blindness because it was convenient. We could write code and let the OS handle the execution.

But as our systems grew faster and our workloads grew more specialized, the cracks began to show. The blindly fair traffic controller started causing traffic jams. We built Ferrari engines and forced them to wait for pedestrians.

The illusion works only as long as you do not look too closely at the latency. Once you do, you see the truth. The machine is not optimizing for your performance. It is optimizing for its own definition of order.

## The Definition of Fairness

For over 15 years, since Linux 2.6 in 2007 - the standard Linux scheduler was called the **Completely Fair Scheduler**, or CFS.

The name was intentional.

CFS treats the CPU as a resource to be divided according to weight among all running tasks. Two tasks with the same weight? Each gets fifty percent. Four tasks? Each gets twenty-five percent. But weights can be adjusted through nice values, control groups, and scheduling policies - cycles are not literally equal.

CFS tracks execution time, often called **vruntime**. Every time a task runs, its virtual runtime increases. The scheduler organizes all waiting tasks into a red-black tree. The task with the lowest vruntime sits at the left of the tree.

When a CPU core becomes free, the scheduler picks the task from the left. The one that has received the least amount of love relative to its weight.

This is a beautiful algorithm. It guarantees no task is ever starved. No process waits forever while another hogs the CPU. It is democratic. It prevents the tyranny of a single heavy process.

But high performance is rarely democratic.

Consider a web server handling a thousand requests. Nine hundred ninety-nine are static files, easy to serve. One is a complex database transaction. It needs to calculate a value, lock a row, write a record, and unlock.

While it holds that lock, no other transaction can proceed. The database is frozen for that fraction of a second.

The fair scheduler sees this heavy transaction running. It sees the execution time ticking up. It decides this thread may have had enough relative to others. It is taking more than its weighted share.

So the scheduler can preempt it. Pulls the thread off the CPU while it is still holding the lock. Puts a static file request on the core instead.

The database transaction sits dormant. The lock remains held. Other threads queue up behind the lock, waiting for it to release. The static file request finishes, but the damage is done. System throughput collapses because the critical path was paused in the name of fairness.

The scheduler did its job. It enforced weighted fairness. And in doing so, it destroyed performance.

This is the **fairness trap**. By **treating all CPU cycles as equal modulo weight**, we ignore that some cycles are structurally more important than others. The cycle that releases a lock is worth a hundred times more than the cycle that computes a background checksum.

The scheduler does not know this. It cannot know this. It sees weights and time slices, not lock holders or critical sections.

## The New Algorithm: EEVDF

In October 2023, Linux kernel 6.6 replaced CFS with a new scheduler: EEVDF (**Earliest Eligible Virtual Deadline First**) . First proposed in a 1995 academic paper, EEVDF took nearly 30 years to land in the kernel.

On paper, EEVDF is more sophisticated. It tracks "lag" - how much CPU time a task is owed relative to its weight and calculates a virtual deadline for eligible tasks, picking the one with the earliest deadline next. This allows latency-sensitive tasks with shorter time slices to be prioritized.

The name changed. The philosophy didn't.

EEVDF still aims to distribute CPU time fairly among tasks of the same priority, using lag and deadlines instead of pure vruntime. And the same problems emerged.

Amazon engineers discovered that EEVDF caused significant performance degradation in multiple database-oriented workloads. Running *MySQL+HammerDB* resulted in a 12-17% throughput reduction and 12-18% latency increase compared to kernel 6.5 running CFS.

The impact was severe enough that Amazon engineers described it as "comparable to the average performance difference of a CPU generation over its predecessor".

Testing combinations of scheduler features showed that the largest improvement came from disabling two specific EEVDF features: **PLACE_LAG** and **RUN_TO_PARITY**.

This is the fairness trap in 2025: even with a new algorithm, the pursuit of mathematical fairness hurts the workloads that matter most. Engineers are disabling features to make databases run fast.

The kernel's response? When Amazon proposed moving these knobs to *sysctl* for easier tuning, Peter Zijlstra responded: "*Nope - you have knobs in debugfs, and that's where they'll stay. Esp. PLACE_LAG is super dodgy and should not get elevated to anything remotely official*".

He elaborated: "*The problem with NO_PLACE_LAG is that by discarding lag, a task can game the system to 'gain' time. It fundamentally breaks fairness*".

The tension is clear: database performance vs. theoretical fairness.

The kernel chose fairness.

## The Physics of Stopping

There is another cost to this constant switching. The cost of the switch itself.

We tend to think of software as abstract logic. Lines of code executing in a clean, mathematical void. We forget the computer is a physical machine. It has metal pathways. Electricity. Inertia.

When a task runs on a CPU core, it builds up state. It loads data from main memory into the cache. The L1 and L2 caches are small, incredibly fast banks of memory right next to the processor.

Think of the cache like a chef's workspace. Knife, cutting board, vegetables arranged exactly where needed. He can chop fast because everything is within reach.

Now the manager walks in. Tells the chef to stop. Time for the baker to use the station. The chef must clear his board. Put vegetables away. The baker arrives and sets up flour and eggs. Works for ten minutes.

Then the manager returns. Time is up. The baker clears the station. The chef returns. He must get his vegetables out again. Find his knife. Rebuild his workspace before making a single cut.

This is a context switch.

Every time the scheduler decides to be fair and swap tasks, the CPU caches may grow cold. Data the previous task needed may be evicted. The new task must pull its data from main RAM, hundreds of times slower than the cache.

This is **cache thrashing**. The friction of multitasking. Not every switch evicts everything - some data may be reused, and hardware prefetching helps - but frequent switching amplifies the cost, especially for tasks with large working sets.

If the scheduler switches too often, the CPU spends more time setting up the workspace than doing actual work. You might see CPU usage at one hundred percent, but your application is crawling. The processor is busy shuffling data, not computing answers.

This is the physics of stopping.

A task that runs uninterrupted is efficient. It keeps its data close. It builds momentum. A task that is constantly interrupted loses that momentum.

The fair scheduler tries to balance this. It has heuristics ensuring a task runs for a "minimum granularity" before being swapped. But these heuristics are guesses. Generic settings for a general world.

For a high-performance application, a context switch is violence. A disruption of the physical state of the hardware. No amount of fairness compensates for time lost to physics.

## The Signal and the Noise

Engineers have known about these problems for years. We have tried to communicate with the scheduler. We have tried to give it hints.

Linux provides tools. System calls like ***nice***, which tells the scheduler to be nicer or meaner to a specific process. **Priority classes**. **CPU affinity**, where we pin a thread to a specific core and forbid it from moving. **Scheduling policies** like SCHED_FIFO and SCHED_RR for real-time tasks.

We sprinkle these hints through our code like prayers. We hope the scheduler listens.

But nice is a relative term. It changes the weight in the fairness algorithm. It does not change the fundamental logic. It does not tell the scheduler about locks, caches, or latency requirements in a way it can act on.

CPU affinity is a sledgehammer. It limits the scheduler's choices but does not make it smarter. You can pin a thread to a core, but if another thread is also pinned there, the scheduler will still toggle between them based on time accounting.

The communication channel is too narrow. We have complex intent. *I want this thread to run only when that queue is full.* *I want this thread to run immediately after an interrupt because it holds a critical lock.*

But the interface mostly accepts simple numbers. Priority -20 to +19. A handful of scheduling classes.

The signal we send is weak. The noise of the system is loud.

Valve Corporation faced this problem with the **Steam Deck**. They needed Windows games to run smoothly on Linux. Games are allergic to latency. If a frame rendering thread is delayed by a millisecond, the player sees a stutter. The experience is broken.

They found the standard scheduler offered no way to express this urgency. It would sometimes wake up a non-critical thread on the same core as the game, pre-empting the renderer. The game would stutter. The logic was fair, but the experience was terrible.

They realized tuning parameters was a dead end. You cannot tune a system solving the wrong problem. You cannot make a fairness engine into a latency engine just by changing the weights.

They needed a new engine.

## The Programmable Core

The solution that emerged is radical. It fundamentally changes the relationship between the kernel and the user.

For the history of general-purpose computing, the kernel was a black box. You ran it. It managed you. If you did not like how it managed you, your only option was to hack the source code, recompile the kernel, and reboot. Risky and difficult.

But Linux gained a superpower called **BPF**. It started as a way to filter network packets but evolved into something profound. It became a safe virtual machine inside the kernel.

It allows you to load small programs into the kernel at runtime. These programs run safely. They cannot crash the machine.

Developers began to ask: What if we could use BPF to write a scheduler? What if, instead of using hard-coded fairness logic, the kernel simply asked a BPF program how to schedule tasks?

This feature is called **sched_ext**. It provides a **BPF-based extensible scheduler class**, merged upstream around the Linux 6.12 timeframe. It allows implementing custom scheduling policies in BPF and loading them at runtime.

This is not an upgrade. It is an **inversion of control**.

Here is a simplified illustration of what a custom scheduler might look like (real *sched_ext* uses a specific set of BPF callbacks like *.select_cpu(), .enqueue(), .dispatch()*, not a single *pick_next_task()* function):

```c
SEC("sched_ext")
int pick_next_task(struct task_struct *prev)
{
    if (is_game_thread(prev))
        return prev->pid; /* run it now */
    return find_fair_task();
}
```

This captures the essence: ***you define the policy. The kernel provides the mechanism*.**

Valve used this to write a custom scheduler for the Steam Deck. Their scheduler is simple. It knows which threads are game threads. When a game thread creates work, the scheduler puts it on the CPU immediately. It does not check if it is fair. It does not update a red-black tree. It just runs the game.

The result was a dramatic reduction in stutter. The hardware did not change. The game code did not change. Only the decision logic changed.

Meta is using this in their data centers. They have machines with complex chip topologies. They wrote a scheduler that understands the physical layout of the processor silicon. It ensures threads that talk to each other stay on the same piece of silicon, avoiding the latency of crossing the chip boundary.

The kernel could never have supported these use cases by default. They are too specific. But with *sched_ext*, the kernel does not have to support them. It just has to get out of the way.

## The Kernel Finally Listens: Linux 6.19

The Linux 6.19 kernel (released this month) brings significant refinements to both the fair scheduler and sched_ext.

Most notably, it reintroduces NEXT_BUDDY - a feature that encourages the scheduler to run related tasks back-to-back, preserving cache warmth. Mel Gorman has reimplemented NEXT_BUDDY to align with EEVDF's goals, and it is included in the scheduler changes for 6.19.

This is significant.

The kernel is acknowledging what we've known all along: context switches have real costs. The best scheduler is the one that interrupts you least. By scheduling related tasks back-to-back, NEXT_BUDDY helps preserve cache warmth and reduce the physics cost of stopping.

Linux 6.19 also brings major improvements to sched_ext: better recovery mechanisms for misbehaving BPF schedulers, lockless peek operations for DSQs to reduce contention, and various robustness improvements.

The kernel is evolving toward intent-based computing. Not by guessing better, but by giving more control.

## Intent-Based Computing

We are entering a new era.

We are moving from General Purpose Computing toward Intent-Based Computing.

In the General Purpose era, we accepted the average. We used a filesystem optimized for everything, so it was great at nothing. We used a scheduler optimized for everyone, so it was perfect for no one.

This was necessary when we ran monolithic servers doing a hundred different things.

But today, we run specialized workloads. A database server often does only one thing: run the database. A game console does one thing: play games. A build node compiles code.

In this world, the General Purpose abstraction is a liability.

Intent-Based Computing means the operating system is no longer a manager imposing its will. It is a framework offering primitives. You, the application developer, provide the policy.

This does not mean every application will ship its own scheduler tomorrow. But the direction is different: specialized workloads will increasingly define their own scheduling policies, whether through sched_ext, better use of existing APIs, or new abstractions yet to emerge.

Your database might eventually ship with a scheduling policy tuned for lock-holder prioritization. Your game engine might ship with a policy that knows rendering threads from audio threads. These policies could be loaded alongside the application, telling the kernel: "*Here is how I want you to schedule my threads. Do not argue with me.*"

This creates a tighter loop between software and hardware. The friction of the "fairness" translation layer is reduced. The application's intent speaks more directly to the core.

It places more responsibility on the engineer. You can no longer blame the "ghost in the machine" for performance spikes. You have more tools to shape your destiny.

But it also grants greater control. It allows us to better align the physics of the machine with the logic of our code.

And when we tell the computer what matters, it may finally have the ears to hear us.

Now, a question for you: What latency issue or performance problem have you blamed on "the scheduler" without really understanding why? Would love to hear your story.

If you enjoyed this, I write about systems engineering, Linux internals, and the evolving relationship between software and hardware. Follow for more deep dives on operating system architecture.

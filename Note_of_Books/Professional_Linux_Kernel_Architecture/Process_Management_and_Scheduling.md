> If a system has only one processor, only one program can run on it at a given time. In multiprocessor systems, the numbers of processes that can truly run in parallel is determined by the number of physical CPUs.
> 
> The kernel and the processor create the illusion of *multitasking* (the ability to perform several operations in parallel) by switching repeatedly between the different applications running on system at very rapid intervals.

Several issues that the kernel must resolve:
- Applications must not interfere with each other unless this expressly desired. (memory protection)
- CPU time must be shared as fairly as possible between the various applications, whereby some programs are regarded as more important than others.

Which implies that:
- The kernel must decide how much time to devote to each process and when to switch to the next process. This begs the question as to *which* process is actually the next. Decisions of this kind are *not* platform dependent.
- When the kernel switches from process A to process B, it must ensure that the execution environment of B is exactly the same as when it last withdrew processor resources. (the contents of the processor registers and the structure of virtual address space) Which is extremely dependent on processor type. It can not be implemented with C only, but requires help by pure assembler portions.

Both tasks above are the responsibility of a kernel subsystem referred to as the *scheduler*. How CPU time is allocated is determined by the scheduler *policy*, which is totally separate from the *task switching* mechanism needed to switch between processes.

## Process Priorities

> Not all processes are of equal importance. In addition to process priority, there are different criticality classes. In a first coarse distinction, processes can be split into real-time processes and non-real-time processes.

- **Hard read-time processes** are subject to strict time limits during which certain tasks must be completed. (e.g. the flight control commands of an aircraft are processed by computer, they must be forwarded as quickly as possible, within a guaranteed period of time) The key character is that they must be processed within a guaranteed time frame. Note that this does not imply that the time frame is particularly short. Instead, the system must guarantee that a certain time frame is never exceeded, even when unlikely or adverse conditions prevail.
- **Soft read-time processes** are a softer form of hard real-time processes. Although quick results are still required, it is not the end of the world if they are a little late in arriving. (e.g. a write operation to a CD. Data must be received by the CD writer at a certain rate because data are written to the medium in a continuous stream. If system loading is too high, the data stream may be interrupted briefly, and this may result in an unusable CD, far less drastic than a plane crash. Nevertheless, the write process should always be granted CPU time when needed, before all other normal processes.)
- **Normal processes** have no specific time constraints but can still be classified as more important or less important by assigning *priorities* to them. (e.g. a long compiler run or numerical calculations need only very low priority because it is of little consequence if computation is interrupted occasionally for a second or two. In contrast, interactive applications should respond as quickly as possible to user commands)

**Linux does not support hard real-time processing** (embedded systems might support). The Linux kernel runs as a separate "process" in these approaches and handles less important software, while real-time work is done outside the kernel. The kernel may run only if no real-time critical actions are performed. Guaranteed response times are only very hard ti achieve. Nevertheless quite a bit of progress has been made to decrease the overall kernel latency, that is, the time elapses between making a request and its fulfillment. The efforts include the preemptible kernel mechanism, real-time mutexes, and the new completely fair scheduler.

*Preemptive multitasking*, each process is allocated a certain time period during which it may execute. Once this period has expired, the kernel withdraws control from the process and lets a different process run. Its runtime environment (essentially, the contents of all CPU registers and the page tables) is saved so that results are not lost and the process environment is fully reinstated when its turn comes around again. The length of the time slice varies depending on the priority assigned to it.

## Process Life Cycle

A process is not always ready to run. Occasionally, it has to wait for events from external sources beyond its control (for keyboard input in a text editor). Until the event occurs, the process can not run.

**The scheduler must know the status of every process in the system when switching between tasks**. Of equal importance are the transitions between individual process states.  
For example, if a process is waiting for data from a peripheral devices, it is the scheduler's responsibility to change the state of the process from *waiting* to *runnable* once the data have arrived (the event occurred).

A process may have following states:

- **Running**: the process is executing at the moment.
- **Waiting**: the process is able to run but is not allowed to because the CPU is allocated to another process. The scheduler can select the process, if it wants to, at the next task switch.
- **Sleeping**: the process is sleeping and cannot run because it is waiting for an external event. The scheduler *cannot* select the process at the next task switch. (there are two "sleeping" states that differ according to whether they can be interrupted by signals or not.)
- **Zombie**: such processes are defunct but are somehow still alive. In reality, they are dead because their resources (RAM, connects to peripherals, etc.) have already been released so they cannot and never will run again. However, they are still alive because there are still entries for them in the process table.
- **Stopped**: the process is stopped.

The system saves all processes in a process table. However, sleeping process are specially "marked" so that the scheduler knows they are not ready to run. There are also a number of queues that group sleeping processes so that they can be woken at a suitable time.

> How do zombies come about? The reason lies in the process creation and destruction structure under UNIX. A program terminates when two events occur:
> 1. The program must be killed by another process or by a user (this is usually done by sending a `SIGTERM` or `SIGKILL` signal, which is equivalent to terminating the process regularly).
> 2. The parent process from which the process originates must invoke or have already invoked the `wait4` system call when the child process terminates. This confirms to the kernel that the parent process has acknowledged the death of the child. The system call enables the kernel to free resources reserved by the child process.
> A zombie occurs when only the first condition (the program is terminated) applies but not the second (`wait4`). A process always switches briefly to the zombie state between termination and removal of its data from the process table. In some cases (if, e.g., the parent process is badly programmed and does not issue a `wait` call), a zombie can firmly lodge itself in the process table and remain there until the next reboot.

## Preemptive Multitasking

The structure of Linux process management requires two further process state options: user mode and kernel mode. This reflect the fact that all the modern CPUs have (at least) two different execution modes: one of which has unlimited rights while the other is subject to various restrictions. The distinction is an important prerequisite for creating locked "cages", which hold existing processes and prevent them from interfering with other parts of the system.

Normally the kernel is in user mode in which it may access only its own data and cannot therefore interfere with other applications (it usually doesn't even notice that there are other programs besides itself).

If a process wants to access system data or function, it must switch to kernel mode. One way is that "system call" to switch between modes. A second way is of switching from user mode to kernel mode is by means of interrupts. Unlike system calls, which are invoked intentionally user applications, interrupts occur more or less arbitrarily.

The preemptive scheduling model of the kernel establishes a hierarchy that determines which process states may be interrupted by which other states.

- Normal processes may always be interrupted, even by other processes (when an important process becomes runnable). The scheduler can decide whether to execute the process immediately, even if the current process is still running. (This kind of preemption makes an important contribution to good interactive behavior and low system latency.)
- If the system is in kernel mode and is processing a system call, no other process in the system is able to cause withdrawal of CPU time. The scheduler is forced to wait until execution of the system call has terminated before it can select another process. However, the system call can be suspended by an interrupt.
- Interrupts can suspend processes in user mode and in kernel mode. They have the highest priority because it is essential to handle them as soon as possible after they are issued.

## Process Representation

All algorithms of the Linux kernel concerned with processes and programs are build around a data structure named `task_struct` and defined in `include/sched.h`.

### `<sched.h>` -> `task_struct`

Fields are classified into **scheduling-critical** (`thread_info`, `saved_state` if configured, `__state`) and **randomized struct fields**.

The structure contents can be broken down into sections, each of which represents a specific aspect of the process:

- State and execution information such as pending signals, binary format used (and any emulation information for binary format of other systems), process identification number (`pid`), pointers to parents and other related process, priorities, and time information on program execution (e.g., CPU time).
- Information on allocated virtual memory.
- Process credentials such as user and group ID, capabilities, and so on. System calls can be used to query (or modify) these data.
- Files used: Not only the binary file with the program code but also filesystem information on all files handled by the process must be saved.
- Thread information, which records the CPU-specific runtime data of the process (the remaining fields in the structure are not dependent on the hardware used).
- Information on interprocess communication required when working with other applications.
- Signal handlers used by the process to respond to incoming signals.

`state` specifies the current state of a process and accepts the following values:

- `TASK_RUNNING` means that a task is in a runnable state. It does not mean that a CPU is actually allocated. The task can wait until it is selected by the scheduler.
- `TASK_INTERRUPTIBLE` is set for a sleeping process that is waiting for some event or other. When the kernel signals to the process that the event has occurred, it is placed in the `TASK_RUNNING` state and may resume execution as soon as it is selected by the scheduler.
- `TASK_UNINTERRUPTIBLE` is used for sleeping processes disabled on the instructions of the kernel. They may not be woken by external signals, only by the kernel itself.
- `TASK_STOPPED` indicates that the process was stopped on purpose (e.g., by a debugger).
- `TASK_TRACED` is not a process state. It is used to distinguish stopped tasks that are currently been traced (using the `ptrace` mechanism) from regular stopped tasks.

The following constants can be used both in the task state field of `struct task_struct`, but also in the field `exit_state`, which is specifically for exiting processes:

- `EXIT_ZOMBIE` is the zombie state described above.
- `EXIT_DEAD` is the state after an appropriate `wait` system call has been issued and before the task is completely removed from the system. This state is only of importance if multiple threads issue `wait` calls for the same task.



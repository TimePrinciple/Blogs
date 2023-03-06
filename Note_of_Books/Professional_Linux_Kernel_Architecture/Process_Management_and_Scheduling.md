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

### `<resource.h>` -> `struct rlimit`

Linux provides the *resource limit* (`rlimit`) mechanism to impose certain system resource usage limits on processes. The mechanism makes use of the `rlim` array in `task_struct`, whose elements are of the `strcut rlimit` type.

- `rlim_cur` is the current resource limit for the process. It is also referred to as *soft limit*.
- `rlim_max` is the maximum allowed value for the limit. It is therefore also referred to as the *hard limit*.

The system call `setrlimit` is used to change the current limit. However, the value specified in `rlim_max` may not be exceeded. `getrlimits` is used to check the current limit.

If a resource type may be used without limits (**the default setting for almost all resources**), `RLIM_INFINITY` is used as the value for `rlim_max`. Exceptions are:

- The number of open files (`RLIMIT_NOFILE`, limited to 1024 by default).
- The maximum number of processes per user (`RLIMIT_NPROC`), defined as `max_threads/2`. `max_threads` is a global variable whose value specifies how many threads may be generated so that an eighth of available RAM is used only for management of thread information, given a minimum possible memory usage of 20 threads.

> The boot-time limits for the `init` task are defined in `INIT_RLIMITS` in `include/asm-generic/resource.h`.

`$ cat /proc/self/limits` to inspect current `rlimit` values.

## Process Types

A classical UNIX process is an application that consists of binary code, a chronological thread (the computer follows a single path through the code, no other paths run at the same time), and a set of resources allocated to the application (e.g., memory, files, and so on).

New processes are generated using the `fork` and `exec` system calls:

- `fork` generates an identical copy of the current process: a *child process*. All resources of the original process are copied in a suitable way so that after the system call there are two independent instances of the original process. These instances are not linked in any way but have, for example, the same set of open files, the same working directory, the same data in memory (each with its own copy of data).
- `exec` replaces a running process with another application loaded from an executable binary file. In other words, a new program is loaded. Because `exec` does not create a new process, an old program must first be duplicated using `fork`, and then `exec` must be called to generate an additional application on the system.

Linux also provides the `clone` system call in addition to the two calls above that are available in all UNIX. In principle, `clone` works in the same way as `fork`, but the new process is not independent of its parent process and can share some resources with it. It is possible to specify *which* resources are to be shared and which are to be copied (e.g., data in memory, open files, the installed signal handlers of the parent process).

`clone` is used to implement *threads*. However, the system call alone is not enough to do this. Libraries are also needed in userspace to complete implementation. Examples of such libraries are *Linuxthreads* and *Next Generation Posix Threads*.

## Namespaces

Namespaces provide a lightweight form of virtualization by allowing us to view the global properties of a running system under different aspects.

#### Concept

Traditionally, many resources are managed globally in Linux. For instance, all processes in the system are conventionally identified by their `PID`, which implies that a global list of PIDs must be managed by the kernel. Likewise, the information about the system returned by the `uname` system call (which includes the system name and some information about the kernel) is the same for all callers. User IDs are managed in a similar fashion: Each user is identified by a `UID` number that is globally unique.

Instead of using virtualized systems such that one physical machine can run multiple kernels (which may well be from different operating systems) in parallel, a single kernel operates on a physical machine, and all previously global resources are abstracted in *namespace*. This allows for putting a group of processes into a *container*, and one container is separated from other containers. The separation can be such that members of one container have no connection whatsoever with other containers. It is possible to loosen the separation of containers by allowing them to share certain aspects of their life. For instance, containers could be set up to use their own set of PIDs, but still share portions of filesystems with each other.

Assume that containers are used in a hosting setup where each container must look like a single Linux machine. Each of them therefore has its own `init` task with PID 0, and the PIDs of other tasks are assigned in increasing order. 

While none of the child containers has any notion about other containers in the system, the parent is well informed about the children, and consequently sees all processes they execute. The "right" PID depends on the context in which the process is observed.

---

Namespace can also be non-hierarchical if they wrap simpler quantities, for instance, like UTS.

New namespaces can be established in two ways:

1. When a new process is created with the `fork` or `clone` system call, specific options control if namespaces will be shared with the parent process, or if new namespaces are created.
2. The `unshare` system call dissociates parts of a process from the parent, and this also includes namespaces.

Once a process has been disconnected from the parent namespace using any of the two mechanisms above, changing a global property (from its point of view) will not propagate into the parent namespace, and neither will a change on the parent side propagate into the child.

#### Implementation

The implementation of namespaces requires two components: per-subsystem namespace structures that wrap all formerly global components on a per-namespace basis, and a mechanism that associates a given process with the individual namespaces to which it belongs.

Formerly global properties of subsystems are wrapped up in namespace, and each process is associated with a particular selection of namespaces. Each kernel subsystem that is aware of namespaces must provide a data structure that collects all objects that must be available on a per-namespace basis. `struct nsproxy` is used to collect pointers to the subsystem-specific namespace wrappers.

##### `<nsproxy.h>`

Currently the following areas of the kernel are aware of namespaces:

- The UTS (UNIX Timesharing System) namespace contains the name of the running kernel, and its version, the underlying architecture type, and so on.
- All information related to inter-process communication (IPC) is stored in `struct ipc_namespace`.
- The view on the mounted filesystem is given in `strcut mnt_namespace`.
- `struct pid_namespace` provides information about process identifiers.
- `struct user_namespace` is required to hold per-user information that allows for limiting resource usage for individual users.
- `struct net_ns` contains all networking-related namespace parameters.

## Process Identification Numbers

UNIX processes are always assigned a number to uniquely identify them in their namespace. This number is called the *process identification number* or *PID* for short. Each process generated with `fork` or `clone` is automatically assigned a new unique PID value by the kernel.

### Process Identifiers

Each process is not only characterized by its PID but also by other identifiers. Several types are possible:

- All processes in a thread group (i.e., different execution contexts of a process create by calling `clone` with `CLONE_THREAD` as we will see below) have a uniform *thread group id* (TGID). If a process does not use threads, its PID and TGID are identical. The main process in a thread group is called the `group leader`. The `group_leader` element of the task structures of all `clone`d threads points to the `task_struct` instance of the group leader.
- Otherwise, independent processes can be combined into a *process group* (using the `setpgrp` system call). The `pgrp` elements of their task structures all have the same value, namely, the PID of the process group leader. Process groups facilitate the sending of signals to all members of the group, which is helpful for various system programming applications. **Notice that processes connected with pipes are contained in a process group**.
- Several process groups can be combined in a session. All processes in a session have the same session ID which is held in the *session* element of the task structure. The SID can be set using the `setsid` system call. It is used in terminal programming.

PID namespaces are organized in a hierarchy. When a new namespace is created, all PIDs that are used in this namespace are visible to the parent namespace, but the child namespace does not see PIDs of the parent anmespace. However this implies that some tasks are equipped with more than one PID, namely, one per namespace they are visible in. This must be reflected in the data structures. We have to distinguish between local and global IDs:

- *Global ID*s are identification numbers that are valid within the kernel itself and in the initial namespace to which the `init` tasks started during boot belongs. For each ID type, a given global identifier is guaranteed to be unique in the whole system.
- *Local ID*s belong to a specific namespace and are not globally valid. For each ID type, they are valid within the namespace to which they belong, but identifiers of identical type may appear with the same ID number in a *different* namespace.

The global PID and TGID are directly stored in the `task_struct`, namely in `pid` and `tgid`. The session and process group IDs are *not* directly contained in the task structure itself, but in the structure used for signal handling. `task_struct->signal->__session` denotes the global SID, while the global PGID is stored in `task_struct->signal->__pgrp`. The auxiliary functions `set_task_session` and `set_task_pgrp` are provided to modify the values.

### Managing PIDs

> In addition to these two fields, the kernel needs to find a way to manage all local per-namespace quantities, as well as the other identifiers like TID and SID. This requires several interconnected data structures and numerous auxiliary functions.

#### Data Structure

A small subsystem known as a *pid allocator* is available to speed up the allocation of new IDs. Besides, the kernel needs to provide auxiliary functions that allow for finding the task structure of a process by reference to an ID and its type, and functions that convert between the in-kernel representation of IDs and the numerical values visible to userspace.

##### `<pid_namespace.h>` -> `struct pid_namespace`

In reality, the structure also contains elements that are needed by the PID allocator to produce a stream of unique IDs, and following elements:

- Every PID namespace is equipped with a task that assumes the role taken by `init` in the global picture. One of the purposes if `init` is to call `wait4` for orphaned tasks, and this must likewise be done by the namespace-specific `init` variant. A pointer to the task structure of this task is stored in `child_reaper`.
- `parent` is a pointer to the parent namespace, and `level` denotes the depth in the namespace hierarchy. The initial namespace has level 0, any children of this namespace are in level 1, and so on. Counting the levels is important because IDs in higher levels must be visible in lower levels. From a given level setting, the kernel can infer how many IDs must be associated with a task.

##### `<pid.h>` -> `struct upid`

`struct upid` represents the information that is visible in a specific namespace.

- `nr` represents the numerical value of an ID.
- `ns` is a pointer to the namespace to which the value belongs.
- `pid_chain`: All `upid` instances are kept on a hash table, `pid_chain` allows for implementing hash overflow lists with standard methods of the kernel.

##### `<pid>` -> `struct pid`

`struct pid` is the kernel-internal representation of a PID.

- `count`: A reference counter.
- `tasks`: Lists of tasks that use this pid and an array with a hash list head for every ID type. This is necessary because ID can be used for several processes.
- `level` denotes how many namespaces the process is visible.
- `numbers` contains an instance of `upid` for each level.

###### `<pid>` -> `enum pid_type`

- `PIDTYPE_PID`
- `PIDTYPE_PGID`
- `PIDTYPE_SID`
- `PIDTYPE_MAX`

Note that thread group IDs are *not* contained in this collection. This is because the thread group ID is simply given by the PID of the thread group leader, so a separate entry is not necessary.



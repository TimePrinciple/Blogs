# Professional Linux Kernel Architecture

> Linux Kernel Version: 2.6.24

## Chapter 1

### Task of the Kernel (What is Kernel?)

On a purely technical level, the kernel is an intermediary layer between the hardware and the software. Its purpose is to pass application requests to the hardware and to act as a low-level driver to address the devices and components of the system.

Three point of views:

- *Enhanced machine*: In the view of the application, abstracts the computer on a high level.  
For example, when the kernel addresses a hard disk, it must decide which path to use to copy data from disk to memory, where the data reside, which commands must be sent to the disk via which path, and so on. Applications, on the other hand, need only issue the command that data are to be transferred. How this is done is irrelevant to the application -- the details are abstracted by the kernel, which, for them, represents the lowest level in the hierarchy they know -- and it therefore an enhanced machine. A machine can interpret and execute the command issued.

- *Resource manager*: It is justified when several programs are run concurrently on a system. In this case, the kernel is an instance that shares available resources -- CPU time, disk space, network connections, and so on -- between the various system processes while at the same time ensuring system integrity.

- *Library*: Which provides a range of system-oriented commands. As is generally known, *system calls* are used to send requests to the computer; with the help of the C standard library, these appear to the application programs as normal functions that are invoked in the same way as any other function.

### Kernel Implementation Strategies

Currently, there are two main paradigms on which the implementation of operation systems is based:

1. **Microkernels** -- In these, only the most elementary functions are implemented directly in a central kernel -- the *micro* kernel. All other functions are delegated to autonomous processes that communicate with the central kernel via clearly defined communication interfaces.  
For example, various filesystems, memory management, and so on. (The most elementary level of memory management that controls communication with the system itself is in the microkernel. However, handling on the system call level is implemented in external servers.) Theoretically, this is a very elegant approach because:
    - The individual parts are clearly segregated from each other, and this forces programmers to use "clean" programming techniques.
    - The dynamic extensibility and the ability to swap important components at run time.

However, owing **the additional CPU time needed to support complex communication between the components**, microkernels have not really established themselves in practice although they have been the subject of active and varied research for some time now.

2. **Monolithic Kernels** (Linux adapted strategy, the performance of monolithic kernels is still greater than that of microkernels) -- They are the alternative, traditional concept. Here, the entire code of the kernel -- including all its subsystems such as memory management, filesystems, or device drivers -- is packed into a single file. Each function has access to all other parts of the kernel; this can result in elaborately nested source code if programming is not done with great care.  
However, one major innovation has been introduced. *Modules* with kernel code that can be inserted or removed while the system is up-and-running support the dynamic addition of a whole range of functions to the kernel, thus compensating for some of the disadvantages of monolithic kernels. This is assisted by elaborate means of communication between the kernel and userland that allows for implementing hotplugging and dynamic loading of modules.

### Processes, Task Switching, and Scheduling

*Processes*: Applications, servers, and other programs running under UNIX are traditionally referred to as *processes*. Each process is assigned address space in the *virtual memory* of the CPU. The address spaces of the individual processes are totally independent so that the processes are unaware of each other -- as far as each process is concerned, it has impression of being the only process in the system.

Because Linux is a *multitasking system*, it supports what appears to be concurrent execution of several processes. Since only as many processes as there are CPUs in the system can really run at the same time, the kernel switches (unnoticed by users) between the processes at short intervals to give them the impression of simultaneous processing. Which leads to two problem areas:

1. *Task switching*: The kernel, with the help of the CPU, is responsible for the technical details of task switching. Each individual process must be given the illusion that the CPU is always available. This is achieved by saving all state-dependent elements of the process before CPU resources are withdrawn and the process is placed in an idle state. When the process reactivated, the exact saved state is restored.

2. *Scheduling*: The kernel must also decide how CPU time is shared between the existing processes. Important processes are given a larger share of CPU time, less important process a smaller share.

### UNIX Processes

Linux employs a hierarchical scheme in in which each process depends on a parent process. The kernel starts the `init` program (the System V init) as the first process that is responsible for further initialization actions and display of the login prompt or (in more widespread use today) display of a graphical login interface. `init` (with the `pid` of 1) is therefore the root (parent) process from which all processes originate. It can be examined by issuing `pstree` program.

> #### Why Replace `init` with `systemd`?
>
> `systemd` is System Daemon (with a primary component "system and service manager") named with UNIX convention to add `d` at the end of daemon. Similar to `init`, `systemd` (also with the `pid` of 1) is the parent process of all other processes directly or indirectly and is the first process that starts at boot. Itself is a background process which is designated to start processes in parallel, thus reducing the boot time and computational overhead.
>
> This `init` daemon is now used by modern systems and **starts system services in parallel** which removes unnecessary delays (in comparison to sequential start by System V init) and speed up the initialization process. `systemd` uses Unit Dependencies to define whether a service **wants/requires** other services to run successfully, and Unit Order to define whether a service needs other services to be started **before/after** it (System V init uses run level to start services sequentially). And these services is managed by `systemctl`.

For generating new process from the `init`, UNIX uses two mechanisms:

1. **fork** -- Generates an exact copy of the current process that differs from the parent process only in its PID (process identification). After the system call has been executed, there are two processes in the system, both performing the same actions. The memory contents of the initial process are duplicated -- at least in the view of the program. Linux uses a well-known technique known as *copy on write* that allows it to make the operation much more efficient by deferring the copy operations until either parent or child writes to a page -- read-only accessed can be satisfied from the same page for both.  
A possible scenario for using `fork` is, for example, when a user opens a second browser window. If the corresponding option is selected, the browser executes a `fork` to duplicate its code and then starts the appropriate actions to build a new window in the child process.

2. **exec** -- Loads a new program into an existing content and then executes it. The memory pages reserved by the old program are flushed, and their contents are replaced with new data. The new program then starts executing.

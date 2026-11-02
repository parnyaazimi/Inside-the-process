# Inside the Process: From User Mode to Context Switch

### A Code-Based Study of Process Lifecycle, Scheduling, Traps, and Context Switching in xv6-RISC-V

**Research Area:** Operating Systems / Computer Engineering  
**Platform:** xv6-RISC-V  
**Architecture:** RISC-V  
**Execution Environment:** QEMU  
**Research Method:** Source Code Analysis and Practical Experimentation  
**Author:** Parnya Azimi  
**Date:** November 2, 2026

---

## Abstract

A process is one of the fundamental abstractions provided by an operating system. Understanding how a process is created, scheduled, executed, interrupted, suspended, resumed, and eventually terminated provides valuable insight into the internal mechanisms of operating systems.

This research investigates the lifecycle of a process in xv6-RISC-V, a Unix-like educational operating system developed by MIT. Rather than studying individual kernel functions in isolation, the research follows a process through its lifecycle and examines how different kernel components cooperate to manage its execution.

The study explores process representation through `struct proc`, `struct context`, and `struct trapframe`; process creation through `allocproc()` and `kfork()`; executable loading through `kexec()`; scheduling and context switching through `scheduler()`, `sched()`, and `swtch()`; user-to-kernel transitions through the RISC-V trap mechanism; system call dispatch; process blocking and synchronization; and process termination through `kexit()` and `kwait()`.

The investigation combines source code analysis with practical execution in QEMU. By examining the initialization of the first user process, following the execution of `init` and the shell, and observing process states through `procdump()`, the study connects theoretical operating system concepts to their actual implementation.

The results demonstrate that process management is not a collection of independent operations. It is a coordinated mechanism involving CPU state preservation, memory management, privilege transitions, scheduling decisions, synchronization, and resource management.

**Keywords:** xv6-RISC-V, Operating Systems, Process Lifecycle, Context Switching, Process Scheduling, System Calls, Trap Handling, RISC-V.

---

## 1. Introduction

### 1.1 Background

Modern operating systems allow multiple processes to share the available CPU resources while maintaining the illusion that each process executes independently.

This behavior requires the operating system to manage process execution carefully. A process may execute normally, request a service from the kernel, wait for an event, lose the CPU to another process, resume execution, or terminate.

The kernel must preserve enough information to resume a process correctly after it stops executing. It must also coordinate process creation, memory allocation, CPU scheduling, and resource cleanup.

These responsibilities are central to operating system design.

### 1.2 Research Motivation

The concepts of process management and context switching are often introduced theoretically. However, understanding their actual implementation requires examining how different kernel components interact.

For example:

- How does the kernel represent a process?
- What happens when a new process is created?
- How does the CPU switch from one process to another?
- How does a user program enter kernel mode?
- How are system calls dispatched?
- What happens when a process must wait for an event?
- How does the kernel safely terminate a process and allow its parent to collect its resources?

These questions motivated this investigation.

### 1.3 Research Objectives

The main objective is to understand the complete process lifecycle in xv6-RISC-V by connecting kernel implementation details to observable execution behavior.

The specific objectives are:

1. Understand the kernel data structures used to represent a process.
2. Investigate process creation and executable loading.
3. Understand how the scheduler selects and executes processes.
4. Examine the mechanism of kernel context switching.
5. Follow transitions between User Mode and Kernel Mode.
6. Understand system call handling.
7. Investigate process blocking, wakeup, and synchronization.
8. Examine process termination and parent-child coordination.
9. Connect the examined mechanisms through a practical execution trace.

### 1.4 Scope

This research focuses on the process lifecycle and the kernel mechanisms directly involved in managing it.

The investigation includes:

- Process representation
- Process creation
- Executable loading
- Scheduling
- Context switching
- Trap handling
- System calls
- Blocking and wakeup
- Process termination
- Parent-child synchronization

Detailed investigation of unrelated subsystems, such as device drivers, filesystem internals, and hardware initialization, is outside the primary scope unless necessary to explain the process lifecycle.

---

## 2. Research Environment and Methodology

### 2.1 Experimental Environment

The research was conducted using xv6-RISC-V in a local development environment.

| Component | Description |
|---|---|
| Operating System | Ubuntu through WSL |
| Emulator | QEMU |
| Target Architecture | RISC-V 64-bit |
| Operating System Under Study | xv6-RISC-V |
| Programming Languages | C and RISC-V Assembly |
| Source Code | Local xv6-RISC-V repository |

QEMU provides a virtual hardware environment in which the xv6 kernel can be executed without requiring a physical RISC-V machine.

The kernel was started using:

```bash
make qemu
```

### 2.2 Research Method

The investigation followed a source-code-driven approach.

The following steps were used:

1. Locate relevant kernel structures and functions.
2. Examine their implementations directly in the source code.
3. Identify the relationship between functions and their callers.
4. Follow execution paths across kernel and user-space boundaries.
5. Execute xv6 in QEMU to observe its behavior.
6. Compare runtime observations with the source code analysis.
7. Organize the findings into a continuous process lifecycle.

The primary source files examined included:

| File | Main responsibility |
|---|---|
| `kernel/main.c` | Kernel initialization and scheduler startup |
| `kernel/proc.h` | Process-related data structures |
| `kernel/proc.c` | Process creation, scheduling, and lifecycle management |
| `kernel/swtch.S` | Kernel context switching |
| `kernel/trampoline.S` | User-to-kernel and kernel-to-user transition code |
| `kernel/trap.c` | Trap handling |
| `kernel/syscall.c` | System call dispatch |
| `kernel/exec.c` | Executable loading |
| `user/init.c` | Initial user-level process |
| `user/sh.c` | Shell implementation |
| `user/usys.S` | User-level system call entry stubs |

### 2.3 Source Version Considerations

The implementation details documented in this report were examined in the local xv6-RISC-V checkout used during the experiments.

Some function names and implementation details may differ across xv6 versions or modified repositories. In particular, the examined implementation uses `kexec()` for kernel-side executable loading and separates channel preparation through `sleep_prepare()`.

For reproducibility, the exact repository revision used in this research is recorded below.

- **Repository:** [xv6-RISC-V](https://github.com/mit-pdos/xv6-riscv)
- **Git Commit:** `e93dabeb98431235c6e1a101cb96c73c923fdbc4`

---

## 3. Process Representation

Understanding the process lifecycle begins with understanding how the kernel represents a process.

In xv6, process information is distributed across several important data structures.

### 3.1 `struct proc`

The `struct proc` structure is the central kernel representation of a process.

It contains information required by the kernel to manage process execution and resources.

Important fields include:

| Field | Purpose |
|---|---|
| `pid` | Process identifier |
| `state` | Current process state |
| `parent` | Pointer to the parent process |
| `kstack` | Kernel stack |
| `pagetable` | Process page table |
| `trapframe` | Pointer to saved user register state |
| `context` | Saved kernel execution context |
| `cwd` | Current working directory |
| `name` | Process name |

The process structure allows the kernel to identify a process, manage its execution, preserve its state, and coordinate its lifecycle.

### 3.2 `struct context`

The `struct context` structure stores the kernel execution context required by the context-switching mechanism.

In the examined implementation, it contains:

- `ra`: Return address
- `sp`: Stack pointer
- `s0` through `s11`: Callee-saved registers

The RISC-V calling convention requires these callee-saved registers to retain their values across function calls.

The `swtch()` assembly routine uses this structure to save the current kernel context and restore another kernel context.

An important distinction is that `struct context` does not represent the entire processor state. It contains the registers needed for xv6's kernel thread switching mechanism.

### 3.3 `struct trapframe`

The trap frame stores the user register state and additional kernel information required during transitions between User Mode and Kernel Mode.

It includes:

- User general-purpose registers
- The saved user program counter (`epc`)
- Kernel stack information
- Kernel trap handler information
- Kernel page table information
- CPU hart identification information

The trap frame allows the kernel to preserve user execution state when a trap occurs and restore it when returning to User Mode.

### 3.4 Relationship Between the Three Structures

These three structures serve different purposes:

| Structure | Responsibility |
|---|---|
| `struct proc` | Represents and manages the process |
| `struct context` | Preserves kernel execution state during context switching |
| `struct trapframe` | Preserves user execution state during traps |

This distinction is fundamental to understanding xv6.

A context switch between kernel threads and a transition between User Mode and Kernel Mode are related, but they are not the same operation.

---

## 4. Process Creation and Executable Loading

Process creation involves preparing the kernel structures and resources required for a process to execute.

### 4.1 `allocproc()`

The `allocproc()` function allocates and initializes a process structure.

Its main responsibilities include:

1. Searching the process table for an unused entry.
2. Acquiring the process lock.
3. Allocating a new process ID.
4. Changing the process state to `USED`.
5. Allocating a trap frame.
6. Creating the process page table.
7. Initializing the kernel context.
8. Setting the initial return address to `forkret`.
9. Setting the initial kernel stack pointer.

An important initialization is:

```c
p->context.ra = (uint64)forkret;
p->context.sp = p->kstack + PGSIZE;
```

This prepares a newly allocated process to begin execution in `forkret()` when the scheduler first switches to its context.

`allocproc()` prepares the process execution environment, but it does not itself load a user executable.

### 4.2 `userinit()` and the First Process

During kernel initialization, `main()` calls:

```c
procinit();
userinit();
```

The `procinit()` function initializes the process table.

The `userinit()` function creates the first user process by calling `allocproc()`.

The examined implementation performs the following operations:

```c
p = allocproc();
initproc = p;

p->cwd = namei("/");
p->state = RUNNABLE;

release(&p->lock);
```

This establishes the initial process and makes it eligible for scheduling.

An important observation is that `userinit()` does not load the `/init` executable.

Instead, it prepares the process so that the scheduler can execute it for the first time.

### 4.3 First Execution Through `forkret()`

The first process has an initial kernel context whose return address points to `forkret()`.

After the scheduler selects this process and performs a context switch, execution begins in `forkret()`.

The first execution of `forkret()` performs filesystem initialization:

```c
fsinit(ROOTDEV);
```

It then invokes:

```c
p->trapframe->a0 =
    kexec("/init", (char *[]){"/init", 0});
```

The `kexec()` function loads the initial executable into the process address space.

This explains how the first process obtains its user program even though `userinit()` did not load it.

The initial process is prepared first, and its executable is loaded later, from within `forkret()`.

### 4.4 `kfork()`

The `kfork()` function creates a child process from an existing process.

Its major responsibilities include:

1. Allocating a new process through `allocproc()`.
2. Copying the parent's user memory using `uvmcopy()`.
3. Copying the parent trap frame.
4. Setting the child's return value to zero.
5. Duplicating relevant process resources.
6. Establishing the parent-child relationship.
7. Changing the child's state to `RUNNABLE`.

An important instruction is:

```c
np->trapframe->a0 = 0;
```

This determines the return value observed by the child after `fork()`.

The parent receives the child's PID, while the child receives zero.

This allows both processes to continue from the same point in the program while following different execution paths.

### 4.5 `exec()` and `kexec()`

Process creation and program replacement are two different operations.

`fork()` creates a new process.

`exec()` replaces the current process's user program image with another executable.

In the examined implementation, the kernel-side loading operation is performed by `kexec()`.

The executable loader is responsible for replacing the user address space with the new program image and preparing the initial execution state.

Consequently, a process can retain its identity and PID while executing an entirely different user program.

This distinction is essential when following the shell's execution of commands.

---

## 5. Process Scheduling and Context Switching

The kernel must decide which process should execute on each CPU.

This responsibility belongs to the scheduler.

### 5.1 Process States

The examined xv6 implementation defines the following process states:

| State | Meaning |
|---|---|
| `UNUSED` | Process slot is available |
| `USED` | Process slot has been allocated |
| `SLEEPING` | Process is waiting for an event |
| `RUNNABLE` | Process is ready to execute |
| `RUNNING` | Process is currently executing |
| `ZOMBIE` | Process has terminated but has not yet been collected by its parent |

These states allow the kernel to track process execution and determine which processes can be scheduled.

A process does not necessarily move through every state in a single fixed sequence.

### 5.2 `scheduler()`

The `scheduler()` function runs continuously on each CPU.

Its main responsibilities are:

1. Enable and manage interrupts.
2. Examine entries in the process table.
3. Find processes in the `RUNNABLE` state.
4. Acquire the process lock.
5. Change the selected process state to `RUNNING`.
6. Associate the process with the current CPU.
7. Switch to the process context through `swtch()`.

The scheduler resumes when the process gives up the CPU through an operation such as `yield()`, `sleep()`, or process termination.

### 5.3 `yield()`

A running process may voluntarily give up the CPU.

The `yield()` function performs this operation by changing the process state to `RUNNABLE` and calling `sched()`.

Conceptually:

```c
acquire(&p->lock);
p->state = RUNNABLE;
sched();
release(&p->lock);
```

This allows the scheduler to select another eligible process.

### 5.4 `sched()`

The `sched()` function performs the controlled transition from a process's kernel execution context to the scheduler context.

Before switching, it checks important conditions:

- The process lock must be held.
- The process must not remain in the `RUNNING` state.
- Interrupts must be disabled.
- Locking conditions must be valid.

It then calls:

```c
swtch(&p->context, &mycpu()->context);
```

The current process context is saved and the CPU scheduler context is restored.

### 5.5 `swtch.S`

The actual register-saving and register-restoring operation is implemented in assembly.

The `swtch()` routine saves:

```asm
sd ra, 0(a0)
sd sp, 8(a0)
sd s0, 16(a0)
...
sd s11, 104(a0)
```

It then restores another context:

```asm
ld ra, 0(a1)
ld sp, 8(a1)
ld s0, 16(a1)
...
ld s11, 104(a1)
ret
```

The routine uses the addresses supplied through `a0` and `a1` to access the source and destination context structures.

The final `ret` transfers execution using the restored return address.

This is the low-level mechanism that makes kernel context switching possible.

### 5.6 Why Context Switching Is Important

A process cannot continuously occupy a CPU.

It may voluntarily give up execution, block while waiting for an event, or be interrupted by the timer.

The scheduler must therefore preserve its kernel execution state so that it can resume later.

The distinction between the scheduler context and process context is central to xv6's scheduling design.

---

## 6. User Mode and Kernel Mode Transitions

An operating system must protect its kernel from direct and unrestricted access by user programs.

xv6 uses RISC-V privilege levels to separate user execution from kernel execution.

### 6.1 User Mode and Kernel Mode

User programs execute in User Mode.

The kernel executes in Supervisor Mode.

User programs cannot directly execute privileged kernel operations or freely access kernel memory.

Instead, they request kernel services through controlled entry mechanisms such as system calls.

### 6.2 The `ecall` Instruction

On RISC-V, a user program invokes a system call using the `ecall` instruction.

This instruction causes a trap into Supervisor Mode.

The kernel then handles the request and eventually returns to the user program.

### 6.3 The Trampoline Mechanism

xv6 uses a special trampoline mapping to manage transitions between user and kernel address spaces.

The main assembly entry point is:

```asm
uservec
```

Its responsibilities include:

1. Saving user registers into the trap frame.
2. Recovering the original user `a0` register.
3. Loading the kernel stack information.
4. Loading kernel trap information.
5. Switching to the kernel page table.
6. Transferring execution to `usertrap()`.

This assembly code prepares the environment required for kernel-level trap handling.

### 6.4 `usertrap()`

The `usertrap()` function handles traps originating from User Mode.

It first verifies that the trap originated in User Mode.

It then saves the user program counter:

```c
p->trapframe->epc = r_sepc();
```

For a system call, it checks the RISC-V exception cause.

The relevant path is:

```c
if (r_scause() == 8) {
    if (killed(p))
        kexit(-1);

    p->trapframe->epc += 4;
    intr_on();
    syscall();
}
```

The saved program counter is advanced by four bytes because the RISC-V `ecall` instruction occupies four bytes.

This prevents the same system call instruction from being executed again when the process returns to User Mode.

After handling the trap, `usertrap()` prepares the return path.

### 6.5 Returning to User Mode

Before returning to User Mode, xv6 calls:

```c
prepare_return();
```

It then prepares the user page table and invokes the trampoline's `userret` entry point.

The `userret` assembly code restores the saved user registers, switches back to the user page table, and executes:

```asm
sret
```

This returns the CPU to User Mode.

The overall user-to-kernel path is:

```text
User Program
     |
     v
   ecall
     |
     v
  uservec
     |
     v
 usertrap
     |
     v
 syscall / Kernel Handler
     |
     v
 prepare_return
     |
     v
  userret
     |
     v
   sret
     |
     v
User Program Resumes
```

---

## 7. System Call Mechanism

System calls provide the interface through which user programs request kernel services.

### 7.1 System Call Entry

User-level system call functions are connected to assembly stubs in `user/usys.S`.

For example, a system call wrapper loads the system call number into register `a7` and executes `ecall`.

The arguments are passed through registers `a0` through `a5`.

### 7.2 `syscall()`

After the trap reaches the kernel, `usertrap()` invokes:

```c
syscall();
```

The `syscall()` function retrieves the system call number from the saved `a7` register in the process trap frame.

It uses this number to locate the corresponding kernel handler in the system call table.

The handler is then executed.

The return value is written into the saved `a0` register.

When the process returns to User Mode, the user program receives the system call result.

### 7.3 System Call Argument Handling

The kernel provides helper functions for retrieving system call arguments.

Examples include:

- `argraw()`
- `argint()`
- `argaddr()`
- `argstr()`

These functions retrieve arguments from the saved user register state and interpret them according to their expected types.

### 7.4 Example: `write()`

The `write()` system call provides a useful example of the complete user-to-kernel interaction.

A user program invokes:

```c
write(fd, buffer, n);
```

The system call eventually reaches the kernel's `sys_write()` handler.

The handler retrieves the file descriptor, buffer address, and requested length.

It then calls:

```c
filewrite(...)
```

The file-writing layer determines how the request should be handled according to the file type.

For the shell's console output, the device-specific path is relevant.

This demonstrates how a seemingly simple user-level function can involve several kernel layers.

---

## 8. Process Blocking and Wakeup

A process does not always remain eligible for execution.

It may need to wait for an event before continuing.

xv6 uses the `SLEEPING` state and the sleep/wakeup mechanism to coordinate such situations.

### 8.1 `sleep_prepare()`

In the examined implementation, `sleep_prepare()` associates the current process with a waiting channel.

It records the channel in:

```c
p->chan = chan;
```

This prepares the process to wait for a particular event.

### 8.2 `sleep()`

The `sleep()` function checks whether a waiting channel has been assigned.

If the process is still waiting, it changes its state:

```c
p->state = SLEEPING;
```

It then calls:

```c
sched();
```

This gives up the CPU and transfers execution to the scheduler.

The process remains suspended until an appropriate wakeup occurs.

### 8.3 `wakeup()`

The `wakeup()` function searches the process table for processes associated with the specified channel.

For matching processes, it clears the channel and changes a sleeping process back to:

```c
p->state = RUNNABLE;
```

An important distinction is that `wakeup()` does not directly execute the process.

It only makes the process eligible for scheduling.

The scheduler will select it later.

### 8.4 The Relationship Between Sleep and Scheduling

The complete conceptual sequence is:

```text
Process Running
      |
      v
Waiting for an Event
      |
      v
sleep_prepare()
      |
      v
sleep()
      |
      v
State = SLEEPING
      |
      v
sched()
      |
      v
Scheduler Runs Another Process
      |
      v
wakeup()
      |
      v
State = RUNNABLE
      |
      v
Scheduler Selects Process
      |
      v
Process Resumes
```

This mechanism is essential for operations such as waiting for a child process to terminate.

---

## 9. Process Termination and Parent-Child Synchronization

Process termination is not simply a matter of stopping execution.

The kernel must clean up resources while preserving enough information for the parent to identify and collect the terminated child.

### 9.1 `kexit()`

The `kexit()` function implements process termination.

Its main responsibilities include:

1. Closing open file descriptors.
2. Releasing the process's current working directory reference.
3. Reparenting child processes when necessary.
4. Waking the parent process.
5. Saving the exit status.
6. Changing the process state to `ZOMBIE`.
7. Switching back to the scheduler.

An important part of the implementation is:

```c
p->xstate = status;
p->state = ZOMBIE;
```

The process has terminated, but its process-table entry is retained until the parent collects it.

The exiting process does not return normally from `kexit()`.

### 9.2 Why Does the Kernel Use the ZOMBIE State?

The kernel cannot immediately discard all information about a terminated child.

The parent may still need:

- The child's PID
- Its termination status
- The ability to identify that the child has completed

The `ZOMBIE` state preserves this information until the parent performs the corresponding wait operation.

### 9.3 `kwait()`

The `kwait()` function allows a parent to wait for a child process to terminate.

It repeatedly examines the process table for children belonging to the current process.

If it finds a child in the `ZOMBIE` state, it:

1. Retrieves the child's PID.
2. Copies the exit status to user memory when a destination address is provided.
3. Detaches the child from its parent.
4. Calls `freeproc()` to release the remaining process resources.
5. Returns the child's PID.

If the child has not terminated, the parent prepares to sleep and waits for a wakeup.

### 9.4 Parent-Child Coordination

The interaction between `kexit()` and `kwait()` forms an important synchronization mechanism.

The exiting child wakes its parent.

The parent then checks the child state again.

If the child is `ZOMBIE`, the parent collects it.

The main sequence is:

```text
Child Process
      |
      v
   kexit()
      |
      v
wakeup(parent)
      |
      v
Save Exit Status
      |
      v
State = ZOMBIE
      |
      v
Parent Resumes
      |
      v
   kwait()
      |
      v
Collect Child Information
      |
      v
   freeproc()
      |
      v
Child Process Slot Released
```

### 9.5 Locking and Synchronization

The process lifecycle also requires synchronization to prevent concurrent CPUs from accessing or modifying process information inconsistently.

Two important locks examined in this implementation are:

- `p->lock`: Protects process-specific state and context.
- `wait_lock`: Protects parent-child relationships and coordination between exit and wait operations.

These locks are particularly important when examining `kexit()`, `kwait()`, and process scheduling.

The process lock is required during controlled transitions into the scheduler, while the wait lock protects relationships between processes.

This demonstrates that scheduling and process termination are closely connected to kernel synchronization.

---

## 10. End-to-End Execution Trace

The previous sections examined individual mechanisms.

This section connects them through a practical execution scenario.

The trace begins with kernel initialization and follows the first user process through the execution of a shell command.

### 10.1 Kernel Initialization

The kernel begins in `main()`.

Among its initialization operations, it initializes the process table and creates the first user process:

```c
procinit();
userinit();
```

After initialization, the kernel enters:

```c
scheduler();
```

The scheduler continuously searches for runnable processes.

### 10.2 First User Process

The first process is created by `userinit()` through `allocproc()`.

Its initial state becomes `RUNNABLE`.

The scheduler selects it and switches to its initial kernel context.

Because its initial return address points to `forkret()`, execution begins there.

The first execution of `forkret()` initializes the filesystem and invokes:

```c
kexec("/init", ...);
```

The initial user program is loaded and prepared for execution.

The trampoline return path then transfers execution to User Mode.

### 10.3 Execution of `init`

The first user program is implemented in:

```text
user/init.c
```

Its responsibilities include preparing standard input and output and starting the shell.

It opens the console and duplicates the file descriptor so that the standard input, output, and error streams refer to the console.

It then creates the shell using:

```c
pid = fork();
```

The parent process continues into `wait()`.

The child executes:

```c
exec("sh", argv);
```

The child process now executes the shell program.

### 10.4 Shell Execution

The shell implementation is located in:

```text
user/sh.c
```

The shell reads commands from the console.

Its main execution path includes:

```c
if (fork1() == 0)
    runcmd(parsecmd(cmd));

wait(0);
```

When a command is entered, the shell creates a child process.

The child parses the command and enters `runcmd()`.

For a simple executable command, the `EXEC` case calls:

```c
exec(ecmd->argv[0], ecmd->argv);
```

The kernel replaces the child process's program image with the requested executable.

### 10.5 Example: Executing `echo Hello`

Consider the following command entered into the xv6 shell:

```bash
echo Hello
```

The execution sequence is:

1. The shell reads the command.
2. The shell creates a child using `fork1()`.
3. The child enters `runcmd()`.
4. The command is identified as an executable command.
5. The child invokes `exec("echo", ...)`.
6. The kernel loads the `echo` program.
7. The `echo` program executes.
8. The program writes its output using `write()`.
9. The kernel handles the system call.
10. The program terminates through `exit()`.
11. The kernel changes the child state to `ZOMBIE`.
12. The shell's `wait()` returns.
13. The terminated child is collected through `freeproc()`.
14. The shell returns to reading commands.

The parent-child relationships can be represented as:

```text
init
 |
 | fork()
 |
 +------ Child: sh
          |
          | fork()
          |
          +------ Child: echo
```

The shell is a child of `init`, while the `echo` process is a child of the shell.

The two parent-child relationships are independent.

### 10.6 Complete Lifecycle Overview

```text
Kernel Initialization
        |
        v
     main()
        |
        v
    userinit()
        |
        v
    allocproc()
        |
        v
    scheduler()
        |
        v
    forkret()
        |
        v
 kexec("/init")
        |
        v
   User Mode: init
        |
        v
      fork()
        |
        +-------------------+
        |                   |
        v                   v
    Parent: init         Child: sh
        |                   |
        v                   v
      wait()             exec("sh")
                            |
                            v
                       Shell Running
                            |
                            v
                          fork()
                            |
                    +-------+-------+
                    |               |
                    v               v
                Parent: sh      Child: echo
                    |               |
                    v               v
                  wait()         exec("echo")
                                    |
                                    v
                               write("Hello")
                                    |
                                    v
                                  exit()
                                    |
                                    v
                              wakeup(parent)
                                    |
                                    v
                                  ZOMBIE
                                    |
                                    v
                              wait() returns
                                    |
                                    v
                                freeproc()
                                    |
                                    v
                            Shell resumes
```

This trace connects process creation, executable loading, system calls, scheduling, and termination into one continuous execution scenario.

---

## 11. Practical Experiments and Observations

The research was not limited to reading source code. xv6 was executed through QEMU to observe actual system behavior.

### 11.1 Booting xv6

The kernel was started using:

```bash
make qemu
```

The observed output included:

```text
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$
```

This confirmed that the kernel successfully initialized and started the initial user process and shell.

The output also showed that the system was running with multiple RISC-V harts.

### 11.2 Executing a User Command

The following command was entered into the xv6 shell:

```bash
echo Hello
```

The observed output was:

```text
$ echo Hello
Hello
$
```

This demonstrated that the shell successfully executed an external user program and returned to its command-reading loop.

### 11.3 Observing Process States

The `procdump()` function was examined to understand how xv6 displays process information.

It iterates over the process table and prints the PID, state, and process name for each allocated process.

The state names in the examined implementation include:

```text
unused
used
sleep
runble
run
zombie
```

The `Ctrl+P` keyboard shortcut was used to display the process table.

At the shell prompt, the observed output was:

```text
1 sleep  init
2 sleep  sh
```

This observation is particularly useful.

At that moment:

- `init` was sleeping while waiting for its child shell through `wait()`.
- `sh` was sleeping while waiting for console input.

The temporary child created to execute the previous `echo` command was no longer present in the process table.

This was consistent with the expected behavior of the shell: after command execution, the child terminates, and the parent collects it through `wait()`.

### 11.4 Practical Observation Summary

| Experiment | Observation | Related Mechanism |
|---|---|---|
| Boot xv6 | Kernel and initial shell start | `main()`, `userinit()`, `scheduler()` |
| Execute `echo Hello` | Command executes and prints output | `fork()`, `exec()`, `write()` |
| Return to shell | Shell displays another prompt | `wait()` and process cleanup |
| Press `Ctrl+P` | Process IDs and states are displayed | `procdump()` |
| Inspect idle process table | `init` and `sh` are sleeping | `kwait()` and console input waiting |

### 11.5 Interpretation

The practical observations supported the source-code analysis.

In particular, the process table output demonstrated that processes can remain allocated while sleeping and that process states provide a useful representation of their current execution conditions.

The experiment also showed why a single runtime observation cannot reveal every state a process experienced.

For example, the child executing `echo` existed during command execution, but it had already terminated and been collected by the time `procdump()` was invoked.

Therefore, source-code tracing and runtime observation must be used together to understand the complete lifecycle.

---

## 12. Discussion

The investigation reveals several important design principles in xv6-RISC-V.

### 12.1 Process Management Is a Coordinated Mechanism

Process management is not implemented by a single function or structure.

Instead, it depends on cooperation between process structures, memory management, scheduling, trap handling, system calls, and synchronization mechanisms.

### 12.2 Context Switching and Trap Handling Are Different

One of the most important distinctions identified during the research is the difference between a context switch and a user-to-kernel transition.

The trap mechanism preserves user execution state when the CPU enters the kernel.

The context-switching mechanism preserves kernel execution state when the CPU changes from one kernel thread to another.

Both are necessary for process execution, but they solve different problems.

### 12.3 `fork()` and `exec()` Have Different Responsibilities

The investigation also clarifies an important operating system concept.

`fork()` creates a new process with a separate process identity.

`exec()` replaces the current process's user program image.

The combination allows a shell to create a child process and execute a requested program within it.

### 12.4 Sleeping Does Not Mean Termination

A process in the `SLEEPING` state has temporarily stopped executing because it is waiting for an event.

It remains an active process and may become `RUNNABLE` again after `wakeup()`.

By contrast, a `ZOMBIE` process has terminated and is waiting for its parent to collect it.

These states represent fundamentally different lifecycle conditions.

### 12.5 Process Termination Requires Parent Coordination

A process cannot always release every part of its process-table representation immediately upon termination.

The parent may still need the child's PID and exit status.

The `ZOMBIE` state and the `wait()` mechanism provide the coordination needed to complete this process safely.

---

## 13. Conclusion

This research examined the lifecycle of a process in xv6-RISC-V, starting from kernel initialization and continuing through process creation, scheduling, execution, system calls, blocking, and termination.

By analyzing the actual source code and observing runtime behavior in QEMU, the investigation established how the kernel coordinates these mechanisms.

The study demonstrated the distinct roles of `struct proc`, `struct context`, and `struct trapframe`; the responsibilities of `allocproc()`, `kfork()`, and `kexec()`; and the cooperation between `scheduler()`, `sched()`, and `swtch()`.

It also examined the RISC-V trap mechanism, the system call dispatch path, and the synchronization mechanisms used to suspend and resume processes.

Finally, the practical execution trace connected the kernel implementation to a real user-level scenario involving `init`, the shell, and the execution of an `echo` command.

The most important outcome of this research is an integrated understanding of process management.

A process is not simply a program being executed by a CPU. It is a managed execution entity whose state, resources, memory, and interactions with the kernel must be carefully coordinated throughout its lifetime.

Studying xv6-RISC-V made these mechanisms observable and understandable at the source-code level, providing a practical foundation for further study of operating system architecture and kernel design.

---

## 14. Limitations and Future Work

This investigation focused on the core process lifecycle and the mechanisms directly involved in it.

Several related topics were intentionally kept outside the main scope, including detailed device-driver implementation, filesystem internals, and low-level hardware initialization.

Potential future investigations include:

- Exploring different CPU scheduling policies.
- Studying synchronization and concurrency in multicore systems.
- Investigating virtual memory and page-fault handling in greater depth.
- Examining inter-process communication mechanisms.
- Comparing xv6 process management with that of larger operating systems.

These topics could extend the current study without changing its central focus.

---

## 15. References

1. Cox, R., Kaashoek, M. F., & Morris, R. (2025). *xv6: A Simple, Unix-like Teaching Operating System*. MIT.  
   https://pdos.csail.mit.edu/6.1810/2025/xv6.html

2. MIT PDOS. *xv6-RISC-V Source Code Repository*.  
   https://github.com/mit-pdos/xv6-riscv

3. MIT PDOS. *xv6-RISC-V Book*.  
   https://github.com/mit-pdos/xv6-riscv-book

4. RISC-V International. *The RISC-V Instruction Set Architecture*.  
   https://riscv.org/technical/specifications/

5. Ritchie, D. M., & Thompson, K. (1974). *The UNIX Time-Sharing System*. Communications of the ACM, 17(7), 365–375.

---

## Appendix A. Important Functions and Their Responsibilities

| Function | Main responsibility |
|---|---|
| `main()` | Kernel initialization |
| `userinit()` | Creates the first user process |
| `allocproc()` | Allocates and initializes a process |
| `forkret()` | Initial kernel execution path for a newly scheduled process |
| `kfork()` | Creates a child process |
| `kexec()` | Loads an executable into a process |
| `scheduler()` | Selects runnable processes |
| `yield()` | Voluntarily gives up the CPU |
| `sched()` | Switches from process context to scheduler context |
| `swtch()` | Saves and restores kernel contexts |
| `uservec` | Entry point for traps from User Mode |
| `usertrap()` | Handles traps originating from User Mode |
| `prepare_return()` | Prepares the return to User Mode |
| `userret` | Restores user state and returns to User Mode |
| `syscall()` | Dispatches system calls |
| `sleep_prepare()` | Associates a process with a waiting channel |
| `sleep()` | Blocks the current process |
| `wakeup()` | Makes matching sleeping processes runnable |
| `kexit()` | Terminates the current process |
| `kwait()` | Waits for and collects a terminated child |
| `freeproc()` | Releases process resources |
| `procdump()` | Displays process information and states |

---

## Appendix B. Key Concepts

| Concept | Explanation |
|---|---|
| Process | An executing program together with its associated kernel-managed state |
| PID | Unique process identifier |
| Context Switch | Switching CPU execution between kernel threads |
| Trap | Transfer of control to the kernel due to a system call, exception, or interrupt |
| System Call | Controlled interface for requesting kernel services |
| User Mode | Restricted CPU privilege level for user applications |
| Kernel Mode | Privileged execution environment of the kernel |
| Scheduler | Kernel component responsible for selecting processes |
| Runnable | Process eligible to execute |
| Sleeping | Process waiting for an event |
| Zombie | Terminated process awaiting collection by its parent |
| `fork()` | Creates a child process |
| `exec()` | Replaces the current process's user program image |
| `wait()` | Waits for a child process to terminate |

---

**End of Research Report**
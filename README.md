# Inside the Process: From User Mode to Context Switch

A source-code-based study of process management in **xv6-RISC-V**, exploring how processes are created, scheduled, switched, interrupted, blocked, and terminated.

## Overview

This project explores the lifecycle of a process in the xv6-RISC-V operating system.

The study follows the execution path from the creation of the first user process to the execution of user programs, examining how the kernel manages process state, performs context switches, handles system calls, and coordinates process termination.

Rather than focusing only on theoretical operating-system concepts, this project uses the xv6-RISC-V source code as its primary subject of study.

## Objectives

The main objectives are to understand:

- How a process is represented inside the kernel.
- How processes are created using `allocproc()` and `kfork()`.
- How executable programs are loaded through `kexec()`.
- How the scheduler selects runnable processes.
- How context switching works through `swtch()`.
- How execution transitions between User Mode and Kernel Mode.
- How system calls are dispatched and executed.
- How processes sleep and become runnable again.
- How process termination and parent-child synchronization work.

## Topics Covered

The project examines the following kernel components and functions:

| Area | Main Components |
|---|---|
| Process Representation | `struct proc`, `struct context`, `struct trapframe` |
| Process Creation | `userinit()`, `allocproc()`, `kfork()` |
| Executable Loading | `forkret()`, `kexec()` |
| Scheduling | `scheduler()`, `yield()`, `sched()` |
| Context Switching | `swtch()` |
| Trap Handling | `usertrap()`, `uservec`, `userret` |
| System Calls | `syscall()`, `sys_write()` |
| Blocking and Wakeup | `sleep_prepare()`, `sleep()`, `wakeup()` |
| Process Termination | `kexit()`, `kwait()`, `freeproc()` |

## Practical Experiment

The operating system was executed using QEMU.

A simple command was run inside the xv6 shell:

```sh
echo ProcessTrace
```

The kernel process table was then inspected using `Ctrl+P`.

The observed output included:

```text
1 sleep  init
2 sleep  sh
```

This observation was used to connect the source-code analysis with actual process behavior during execution.

## Environment

- Operating System: xv6-RISC-V
- Host Environment: Ubuntu on WSL
- Emulator: QEMU
- Build and Execution: Make

## Project Structure

```text
inside-the-process/
├── README.md
└── report/
    └── inside-the-process.md
```

## References

- [xv6: A simple, Unix-like teaching operating system — MIT](https://pdos.csail.mit.edu/6.828/2025/xv6/book-riscv-rev5.pdf)
- [MIT xv6-RISC-V Source Repository](https://github.com/mit-pdos/xv6-riscv)
- [MIT xv6: Traps and System Calls](https://pdos.csail.mit.edu/6.828/2025/lec/l-traps.pdf)
- [MIT xv6: Scheduling](https://pdos.csail.mit.edu/6.828/2025/lec/l-scheduling.pdf)

## Purpose

This project is an educational exploration of process management and kernel execution in xv6-RISC-V, based on source-code study and practical observation.
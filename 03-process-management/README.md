# Process Management

## The Process

- executing program (text section) + resources (open files, pending signals. internal kernel data, processor state, memory address space, threads of execution, data section)
- Each thread includes *unique program counter*, *process stack*, & *set of processor registers*. **Kernel schedules individual threads, not processes**
- Linux does not differentiate between threads & processes. Thread is just special kind of process.

**Creation of process**

- In linux, process is created using `fork()` system call, which creates a new process by duplicating an existing one.
- Parent resumes execution & child starts execution where `fork()` returns. `fork()` syscall returns twice from kernel (parent & child)
- As child is running same program, it is desirable to to call `exec()` immediately after `fork()` to create a new address space & loads a new program into it.
- program exists via `exist()` syscall. parent can wait for child to exit using `wait4()` syscall.

## Process Descriptor & Task Structure

- kernel stores list of processes in a **circular doubly linked list** called `task list`.
- each element in the task list is **process descriptor** of the type `struct task_struct` which is defined in [<linux/sched.h>](https://github.com/torvalds/linux/blob/v5.17/include/linux/sched.h#L728)

### Allocating Process Descriptor

- `task_struct` is allocated via **slab allocator** to provide object reuse and cache coloring.
- Prior to 2.6 kernel series, `struct task_struct` was stored to end of the kernel stack of each process. This allowed architectures with few registers, such as x86, to calculate the location of process description via **stack pointer** without using extra register to store location.
- with process descriptor now dynamically created via slab allocator, new structure `struct thread_info` [x86 <asm/thread_info.h>](https://github.com/torvalds/linux/blob/v5.17/arch/x86/include/asm/thread_info.h) | [How 'task_struct' is accessed via 'thread_info' in linux latest kernel?](https://stackoverflow.com/questions/70043591/how-task-struct-is-accessed-via-thread-info-in-linux-latest-kernel) was created that again lives to end of kernel stack.


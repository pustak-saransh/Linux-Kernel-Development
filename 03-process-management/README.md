# Process Management

## The Process

- executing program (text section) + resources (open files, pending signals. internal kernel data, processor state, memory address space, threads of execution, data section)
- Each thread includes *unique program counter*, *process stack*, & *set of processor registers*. **Kernel schedules individual threads, not processes**
- Linux does not differentiate between threads & processes. Thread is just special kind of process.

**Creation of process**

- In linux, process is created using `fork()` system call, which creates a new process by duplicating an existing one.
- Parent resumes execution & child starts execution where `fork()` returns. `fork()` syscall returns twice from kernel (parent & child)
- As child is running same program, it is desirable to to call `exec()` immediately after `fork()` to create a new address space & loads a new program into it.
- program exists via `exit()` syscall. parent can wait for child to exit using `wait4()` syscall.

## Process Descriptor & Task Structure

- kernel stores list of processes in a **circular doubly linked list** called `task list`.
- each element in the task list is **process descriptor** of the type `struct task_struct` which is defined in [<linux/sched.h>](https://github.com/torvalds/linux/blob/v5.17/include/linux/sched.h#L728)

### Allocating Process Descriptor

- `task_struct` is allocated via **slab allocator** to provide object reuse and cache coloring.
- Prior to 2.6 kernel series, `struct task_struct` was stored to end of the kernel stack of each process. This allowed architectures with few registers, such as x86, to calculate the location of process description via **stack pointer** without using extra register to store location.
- with process descriptor now dynamically created via slab allocator, new structure `struct thread_info` [x86 <asm/thread_info.h>](https://github.com/torvalds/linux/blob/v5.17/arch/x86/include/asm/thread_info.h) | [How 'task_struct' is accessed via 'thread_info' in linux latest kernel?](https://stackoverflow.com/questions/70043591/how-task-struct-is-accessed-via-thread-info-in-linux-latest-kernel) was created that again lives to end of kernel stack.

![process descriptor & kernel stack](./images/process-descriptor-and-kernel-stack.PNG)

### Storing Process Descriptor

- System identifies process by **unique process identification (pid)**. PID is numerial value represented by opaque type`pid_t` which is typically an `int`.
- Maximun value can be **32768** which can be increased as high as fpur million (check [<linux/threads.h>](https://github.com/torvalds/linux/blob/v5.17/include/linux/threads.h))
- If system is willing to break compatibility with old applications, max value can be increased via `proc/sys/kernel/pid_max`
- Inside kernel, tasks are typically referenced directly by a pointer to their `task_struct`. Which is done via `current` macro (This macro must be independently implemented by each architecture). Generic `current` macro defined [<asm-generic/current.h>](https://github.com/torvalds/linux/blob/v5.17/include/asm-generic/current.h) which calls function `get_current()` which calls `current_thread_info()`. (Arch specific, for [x86](https://github.com/torvalds/linux/blob/v5.17/arch/x86/include/asm/current.h))
- Some architectures save pointer to `task_struct` of currently running process in a register, enabling efficient access.

## Process State

Each processon the system is in exactly one of the **five** different states. Code [<linux/sched.h>](https://github.com/torvalds/linux/blob/v5.17/include/linux/sched.h)

- **TASK_RUNNING**: Process is runnable. It is either currently running (user/kernel space) or on a run queue waiting to run.
- **TASK_INTERRUPTIBLE**: Process is sleeping (blocked), waiting for some condition to exist.
- **TASK_UNINTERRUPTIBLE**: This state is identical to TASK_INTERRUPTIBLE except that it *does not wake up and become runnable if it recieves a signal*.
  
  >This is why you have those dreaded unkillable processes with state D in ps(1). Because the task will not respond to signals, you cannot send it a SIGKILL signal. Further, even if you could terminate the task, it would not be wise because the task is supposedly in the middle of an important operation and may hold a semaphore
- **__TASK_TRACED**: The process is being *traced* by another process, such as a debuger via *ptrace*
- **__TASK_STOPPED**: Process execution has stopped. Task is not running nor is it eligible to run. This occurs if the task receives the **SIGSTOP, SIGTSTP, SIGTTIN or SIGTTOU** signal. or if it receives any signal while it is being debugged.

![process states](./images/process-states.PNG)

### Manipulating Current Process State

Kernel code often needs to change a process's state. 

```c
// looks like this is removed
// https://lore.kernel.org/lkml/1483121873-21528-1-git-send-email-dave@stgolabs.net/
// but same commit on github dont have change https://github.com/torvalds/linux/commit/be628be09563f8f6e81929efbd7cf3f45c344416
set_task_state(task, state);
```

Another method which I believe currently getting used is

```c
// https://github.com/torvalds/linux/blob/v5.17/include/linux/sched.h
set_current_state(state);
```

### Process Context

- Normal program executes in *user space*. When program executes **syscall or trigger exception**, it enters into *kernel space* (**Kernel is executing on behalf of process**) & is in process context.
- When in process context, the `current` macro is valid.
- Upon exiting kernel, the process resumes execution unless higher priority process has become runnable.

>Suppose while running normal process, interrupt happens. Then system is not running on behalf of a process but is executing an interrupt handler. No process is tied to interrupt handlers.

### Process Family Tree

- All processes are descendants of `init` process (PID=1). Kernel starts `init` in last step of boot process. `init` process reads system **initscripts** and execute more programs.
- Every process on system has exactly **one parent** and **zero or more children**.
- relationship between processes is stored in process descriptor. Each `task_struct` has a pointer to parent's `task_struct` named `parent` and list of children, named `children` ([Check](https://github.com/torvalds/linux/blob/v5.17/include/linux/sched.h#L954-L970))
- `init` task's descriptor is statically allocated as `init_task`

```c
struct task_struct *task;

for_each_process(task) {
    printk("%s[%d]\n", task->comm, task->pid);
}
```

## Process Creation

- Most OS implements *spawn* mechanism to create a new process in a new address space, read in an executable, and begin executing it.
- but in Linux, we have `fork()` and `exec()` 2 syscall to achieve same.
- `fork()` creates child process that is copy of current task. then `exec()` loads new executable into address space and begins executing it.

### Copy-on-Write

- Traditionally, upon `fork()`, all resources owned by parent are duplicated and copy is given to child. This is inefficient because if new process were to immediately execute new image, all copying would go waste.
- In Linux, `fork()` is implemented through the use of **copy-on-write**. It is technique to delay or altogether prevent copying of data. Rather than duplicate process address space, parent & child share single copy.
- When child write, duplicate is made available.

### Forking

- Linux implements fork() via the clone() system call.This call takes a series of flags that specify which resources, if any, the parent and child process should share. (Check [syscalls declared](https://github.com/torvalds/linux/blob/master/include/linux/syscalls.h) | [definition](https://github.com/torvalds/linux/blob/master/kernel/fork.c))
- syscall `fork()` calls routine `pid_t kernel_clone(struct kernel_clone_args *args)`. [Check routine](https://github.com/torvalds/linux/blob/master/kernel/fork.c#L2614)
- `kernel_clone` internally calls `copy_process` [Check routine](https://github.com/torvalds/linux/blob/master/kernel/fork.c#L1975)
  ```c
  static __latent_entropy struct task_struct *copy_process(
					struct pid *pid,
					int trace,
					int node,
					struct kernel_clone_args *args)
  ```
- Deliberately, the kernel runs the child process first.8 In the common case of the child simply calling exec() immediately, this eliminates any copy-on-write overhead that would occur if the parent ran first and began wr iting to the address space.

### vfork()

The `vfork()` system call has the same effect as `fork()`, except that the **page table entries of the parent process are not copied**. Instead, the child executes as the sole thread in the parent’s address space, and the **parent is blocked until the child either calls `exec()` or exits**. The child is **not allowed to write to the address space**.

```c
//https://github.com/torvalds/linux/blob/master/kernel/fork.c#L2727-L2753

// fork args
struct kernel_clone_args args = {
    .exit_signal = SIGCHLD,
};

// vfork args
struct kernel_clone_args args = {
    .flags		= CLONE_VFORK | CLONE_VM,
    .exit_signal	= SIGCHLD,
};
```

- Inside `kernel_clone`, special flag `CLONE_VFORK` is checked [here](https://github.com/torvalds/linux/blob/master/kernel/fork.c#L2673)

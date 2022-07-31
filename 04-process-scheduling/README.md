# Process Scheduling

Process scheduler decides which process runs, when & for how long. It divides finite resources of processor time between the runnable processes.

## Multitasking

Simultaneously inteleave execution of more than one process.
- **cooperative multitasking**: Process does not stop running until it voluntary decides to do so.
  👎 The scheduler can't make global decisions regarding how long process run. Processes can **monopolize the processor**.
- **Preemptive multitasking**: Scheduler decides when a process is to cease running & new process is to begin running.
  The time process runs before it is preempted is usually predetermined, and it is called the **timeslice**.
  👍 Scheduler can make global decisions for system and also it prevents any one process from monopolizing the processor.

## Linux's Process Schedular

## Policy

### IO bound Vs Processor bound processes

### Process Priority

### Timeslice

### Scheduling Policy in Action

## Linux Scheduling Algorithm

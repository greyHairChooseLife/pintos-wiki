
> [!qt] What is preemtive scheduling of OS?

Preemptive scheduling in operating systems is a process scheduling method
where the OS can forcibly interrupt a currently running process to allocate
CPU time to another process. This is typically done based on priorities,
time slices, or other criteria to ensure fairness, responsiveness, and prevent
monopolization of resources. It contrasts with non-preemptive (cooperative)
scheduling, where processes voluntarily yield control.




> [!qt] Should interrupt stop while scheduling?


Scheduling should not be interrupted to maintain system stability,
prevent race conditions, and ensure atomic operations on shared kernel
data structures. Interrupting could lead to inconsistent states, deadlocks,
or crashes.

Here are examples:

  - **Updating process states**: During scheduling, the kernel might move a
    process from "running" to "ready" in the queue. An interrupt (e.g.,
    timer) could preempt this, causing another process to access the
    partially updated queue, leading to a race condition where the wrong
    process runs.
    
  - **Context switching**: Switching CPU registers and memory mappings
    between processes. If interrupted mid-switch, the new process might
    inherit corrupted state, causing data corruption or invalid memory
    access.

  - **Priority queue management**: In a priority scheduler, recalculating
    priorities. An interrupt could alter the queue during recalculation,
    resulting in a process with incorrect priority executing, violating fairness
    or real-time deadlines.





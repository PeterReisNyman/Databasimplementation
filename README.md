# Implementing Deadline Scheduling in MINIX

## Background and the Original Scheduling

In the MINIX operating system, the scheduler plays a crucial role in managing how processes are given access to the CPU. The original scheduling algorithm in MINIX is a priority-based, time-sharing scheduler. Each process is assigned a static priority level and a time quantum. The scheduler selects the highest-priority process that is ready to run, allocating CPU time based on these priorities.

Processes consume their allotted time quantum and may be preempted if a higher-priority process becomes ready to run. This mechanism ensures a fair distribution of CPU time among processes but does not account for time-critical tasks that require meeting specific deadlines. Consequently, processes with urgent timing constraints may not receive CPU time promptly, leading to missed deadlines.

Understanding the limitations of the original scheduler is essential to appreciate the modifications introduced. The key aspect is that the original scheduler lacks the ability to handle deadlines or adjust scheduling decisions based on timing requirements.

## Aim

The goal of this project is to modify the MINIX scheduler to support deadline scheduling. By allowing processes to specify deadlines, the scheduler can prioritize them appropriately to increase the likelihood of meeting their timing constraints. The project aims to implement this functionality, test it under various system loads, and evaluate its effectiveness in ensuring that time-critical processes meet their deadlines.

## Approach: The Modified Scheduler and the Test Program

### Modifications to the Scheduler

To introduce deadline scheduling, we implemented a new system call `setdl(int ticks)`. This system call enables a process to set a deadline by specifying the number of ticks (system time units) until its deadline. The implementation involved several steps:

1. **System Call Definition**: Added `SETDL` to the system call numbers in `/usr/src/include/minix/callnr.h`.

2. **Process Manager Updates**:
   - Declared the system call handler `do_setdl` in `/usr/src/servers/pm/proto.h` and `/usr/src/servers/pm/callnr.h`.
   - Added `do_setdl` to the system call table in `/usr/src/servers/pm/table.c`.
   - Implemented the handler in `/usr/src/servers/pm/misc.c`, which passes the deadline value to the kernel.

3. **Kernel-Level Support**:
   - Added `SYS_SETDL` to `/usr/src/include/minix/com.h` and incremented `NR_SYS_CALLS`.
   - Implemented `sys_setdl` in `/usr/src/lib/syslib/sys_setdl.c`, which constructs a message with the process endpoint and deadline.
   - Declared `sys_setdl` in `/usr/src/include/minix/syslib.h`.
   - Mapped `SYS_SETDL` to `do_setdl` in `/usr/src/kernel/system.c` and updated the system call handler table.
   - Implemented `do_setdl` in `/usr/src/kernel/system/do_setdl.c`, which updates the `p_deadline` field in the process's kernel structure.

4. **Scheduler Adjustment**:
   - Modified the scheduler in `/usr/src/kernel/proc.c` by adding a condition to check if a process has a deadline:
     ```c
     if (rp->p_deadline > 0) {
         rp->p_priority = DEADLINE_PRIORITY;
     }
     ```
     Here, `DEADLINE_PRIORITY` is set to a high-priority level to ensure that processes with deadlines are scheduled before others.

### Rationale for the Modifications

By introducing the `setdl` system call and adjusting process priorities based on deadlines, we expect processes with time constraints to receive CPU time more promptly. Elevating their priority increases the scheduler's likelihood of selecting them over non-critical processes, thus enhancing their chances of meeting deadlines.

### Test Program Description

The test program, `deadline`, simulates a time-critical task. Its main function performs the following steps:

1. **Set Deadline**: Calls `setdl(ticks)` to specify its deadline.
2. **Execution Loop**: Enters a loop where it performs computations or sleeps to simulate work.
3. **Deadline Checking**: At each iteration, checks if the current time has exceeded the deadline.
   - If the deadline is missed, it reports a missed deadline and terminates.
   - If the deadline is met, it continues execution.

By running multiple instances of this program concurrently, we can create system load and observe how the scheduler handles scheduling processes with deadlines under different conditions.

## Results

### Testing Scenarios

We conducted tests under various scenarios to compare the performance of the original scheduler and the modified scheduler with deadline support.

1. **Baseline Test**:
   - **Setup**: Ran four instances of the `deadline` program with deadlines of 3 seconds.
   - **Original Scheduler**: All instances missed their deadlines consistently.
   - **Modified Scheduler**: All instances successfully met 100 successive deadlines.

2. **Increased Load Test**:
   - **Setup**: Ran six instances of the `deadline` program with deadlines of 15 ticks.
   - **Modified Scheduler**: Some instances missed their deadlines due to higher system load.

### Observations

- **Original Scheduler**: Lacked the ability to prioritize deadline processes, resulting in missed deadlines even under moderate load.
- **Modified Scheduler**: Improved deadline adherence significantly by elevating the priority of processes with deadlines.
- **System Limitations**: Under heavy load, the modified scheduler could not prevent all deadline misses, indicating resource limitations.

### Data Summary

| Number of Instances | Deadline (ticks) | Scheduler          | Deadlines Met | Deadlines Missed |
|---------------------|------------------|--------------------|---------------|------------------|
| 4                   | 3 seconds        | Original           | 0             | All              |
| 4                   | 3 seconds        | Modified           | All           | 0                |
| 6                   | 15 ticks         | Modified           | Majority      | Some             |

## Discussion

The project successfully implemented deadline scheduling in the MINIX scheduler. The introduction of the `setdl` system call and priority adjustments allowed processes to specify deadlines and be scheduled preferentially. The results demonstrate that the modified scheduler effectively improves the ability of time-critical processes to meet their deadlines under normal system loads.

The observations align with our expectations:

- **Improved Deadline Adherence**: Processes with deadlines received higher priority, leading to better scheduling and deadline fulfillment.
- **Resource Constraints**: Under heavy load, the system's finite resources limited the scheduler's ability to meet all deadlines, highlighting that priority adjustments alone cannot overcome physical limitations.

### Reflections and Improvements

While the modifications enhanced deadline handling, several areas for improvement remain:

- **Advanced Scheduling Algorithms**: Implementing real-time scheduling algorithms like Earliest Deadline First (EDF) could further optimize deadline adherence by dynamically scheduling processes based on the urgency of their deadlines.
- **Admission Control**: Introducing mechanisms to limit the number of deadline processes admitted to the system could prevent overload and ensure that accepted tasks have sufficient resources.
- **Fine-Grained Priority Levels**: Adjusting priorities based on how close a process is to its deadline could make scheduling decisions more responsive to timing constraints.

## Conclusion

The modified scheduler achieved the project's aim by providing basic deadline scheduling capabilities. Processes can now set deadlines and are given higher priority accordingly. While the scheduler performs well under normal conditions, heavy system loads still pose challenges. Future enhancements could build upon this foundation to create a more robust real-time scheduling system.

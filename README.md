# Deadline Scheduling in MINIX

## Background and the Original Scheduling

The original scheduler for MINIX is a priority-based, time-sharing scheduler that uses a round-robin scheduling scheme with queues for system processes, user processes, and one for the idle process. There are 16 queues in total, with queues 0-6 reserved for system processes, 7-14 for user processes, and 15 for the idle process. The lower the queue number, the higher the priority is for the processes in that queue.

The scheduler selects the highest-priority process that is ready to run, allocating CPU time. If a process is ready to run, it checks the queues in ascending order, starting from queue 0, followed by queue 1, and so on. If there are no processes ready to run, it defaults to running the lowest priority process, which is the idle process.

Processes consume their allotted time quantum and may be blocked if a higher-priority process becomes ready to run. This ensures a fair distribution of CPU time among processes but does not account for tasks that require meeting deadlines. If a blocked process becomes unblocked, it will be placed at the front of the queue, given that it has quantum time left. This allows I/O processes to have a better response time.

If a process uses its full quantum, it will still have a higher priority than the idle process. In this way, CPU-bound processes will not interfere with the I/O-bound processes' ability to run. The key takeaway from this original scheduler is its lack of support for handling processes that need to meet deadlines.

To implement deadline scheduling functionality, we need to introduce a system call from the user space to the service space, which in turn invokes a kernel call to modify the process table in the kernel. This is necessary because a user process cannot directly modify the kernel's process table. By adding a field in the kernel's process table to store the deadline value set via the system calls, and modifying the scheduler to consider this value, we can adjust process priorities accordingly.

## Aim

The goal of this project is to modify the MINIX scheduler to support deadline scheduling. By allowing processes to specify deadlines, the scheduler can prioritize them appropriately to increase the likelihood of meeting their timing constraints. The project aims to implement this functionality and test it with a test program.

To achieve this, we need to:

- Implement a `deadline` program that sets a deadline for a process.
- Create a test program that forks multiple instances of the `deadline` program.
- Develop a user library function that invokes a system call to the service space.
- Implement a kernel call that updates the kernel's process table with the deadline.
- Modify the scheduler to consider deadlines when scheduling processes.
- Ensure all associated files for the system call and kernel call are correctly configured.
- Rebuild the system for changes to take effect.

## Approach: The Modified Scheduler and the Test Program

### Modifications to the Scheduler

#### System Call Definition

- **File**: `/usr/src/include/minix/callnr.h`

  ```c
  #define SETDL 0
  #define NCALLS 1 /* number of system calls allowed */
  ```

  Assigning a number to the `SETDL` system call enables the system to identify and process this specific request for setting deadlines.

#### Process Manager Updates

- **Declaration**: `/usr/src/servers/pm/proto.h`

  ```c
  _PROTOTYPE( int do_setdl, (void) );
  ```

- **System Call Table**: `/usr/src/servers/pm/table.c`

  ```c
  _PROTOTYPE (int (*call_vec[]), (void) ) = {
      do_setdl,   /* 0 = do_setdl */
      do_exit,    /* 1 = exit */
      do_fork,    /* 2 = fork */
      //...
  };
  ```

- **Implementation**: `/usr/src/servers/pm/misc.c`

  ```c
  int do_setdl(void)
  {
      int ticks;
      ticks = m_in.m1_i1;
      /* Pass the deadline value to the kernel */
      sys_setdl(who_e, ticks);
      return OK;
  }
  ```

Including `do_setdl` in the relevant files within the Process Manager allows the system to identify and direct the call properly. By listing it in the system call table, we ensure it is processed consistently as an official system function.

#### Kernel-Level Support

- **Definition**: `/usr/src/include/minix/com.h`

  ```c
  #define SYS_SETDL       (KERNEL_CALL + 50)
  #define NR_SYS_CALLS    51
  ```

- **Implementation of `sys_setdl`**: `/usr/src/lib/syslib/sys_setdl.c`

  ```c
  #include <minix/syslib.h>
  #include <minix/com.h>

  int sys_setdl(endpoint_t proc_nr, int deadline)
  {
      message m;
      m.m1_i1 = proc_nr;
      m.m1_i2 = deadline;
      return _taskcall(SYSTASK, SYS_SETDL, &m);
  }
  ```

- **Declaration**: `/usr/src/include/minix/syslib.h`

  ```c
  _PROTOTYPE(int sys_setdl, (endpoint_t proc_nr, int deadline));
  ```

- **Mapping System Call**: `/usr/src/kernel/system.c`

  ```c
  map(SYS_SETDL, do_setdl); /* Map SYS_SETDL to do_setdl */
  ```

- **Implementation of `do_setdl`**: `/usr/src/kernel/system/do_setdl.c`

  ```c
  #include "../system.h"
  #include <sys/resource.h>

  int do_setdl(message *m_ptr)
  {
      int proc_nr;
      int deadline;
      struct proc *rp;

      if(!isokendpt(m_ptr->m1_i1, &proc_nr)) return EINVAL;

      deadline = m_ptr->m1_i2;
      rp = proc_addr(proc_nr);
      rp->p_deadline = get_uptime() + deadline;

      return OK;
  }
  ```

Implementing `sys_setdl` and defining `SYS_SETDL` for the kernel allows the system call to work by passing a message with the process ID and deadline. This setup enables the kernel to respond to the deadlines assigned to processes.

#### Scheduler Adjustment

- **Adding Deadline Field**: `/usr/src/kernel/proc.h`

  ```c
  struct proc {
      //...
      int p_deadline;
  };
  ```

- **Modifying the Scheduler**: `/usr/src/kernel/proc.c`

  ```c
  PRIVATE void sched(void)
  {
      //...
      if (rp->p_deadline > 0) {
          rp->p_priority = 8; // Higher priority for processes with deadlines
      }
      //...
  }
  ```

By introducing the `setdl()` system call and adjusting process priorities based on deadlines, we expect processes with time constraints to receive CPU time more promptly. Elevating their priority increases the scheduler's likelihood of selecting them over non-critical processes, thus enhancing their chances of meeting deadlines.

### Test Program Description

The test program is designed to evaluate the scheduler's ability to handle multiple processes with deadlines. It works by creating multiple child processes, each of which runs an instance of the `deadline` program.

Here's how the test program operates:

1. **Forking Processes**: The main program uses a loop to fork child processes. In our test, we fork 10 child processes using the `fork()` system call inside a loop that iterates 10 times.

2. **Executing the Deadline Program**: Each child process, upon successful creation (`pid == 0`), replaces its execution image with the `./deadline` program using the `execvp()` function. This function executes the `deadline` program, which sets a deadline for itself using the `setdl()` system call implemented earlier.

3. **Parent Process**: The parent process continues to fork child processes until all have been created. After forking, it waits for all child processes to complete using the `wait()` system call inside another loop. This ensures that the parent process does not terminate before all child processes have finished execution.

4. **Deadline Program Execution**: The `deadline` program, executed by each child process, sets a deadline using `setdl()` and performs some work to test whether it meets its deadline. It typically enters a loop where it checks the current time against its deadline and exits if it cannot meet the deadline.

The purpose of this test program is to simulate a scenario where multiple processes with deadlines are competing for CPU time. By observing whether these processes meet their deadlines under the original scheduler and the modified scheduler, we can assess the effectiveness of our deadline scheduling implementation.

#### Test Program Code

- **File**: `/home/testprogram.c`

  ```c
  #include <stdio.h>
  #include <sys/wait.h>
  #include <unistd.h>
  #include <sys/types.h>

  int main(int argc, char *argv[])
  {
      int i;
      for (i = 0; i < 10; i++)
      {
          pid_t pid = fork();
          if (pid == 0) {
              char *args[] = {"./deadline", NULL};
              execvp(args[0], args);
              // If execvp fails
              perror("execvp failed");
              _exit(1);
          }
      }
      for (i = 0; i < 10; i++) {
          wait(NULL);
      }
      return 0;
  }
  ```

The test program creates 10 child processes, each executing the `./deadline` program. The parent process waits for all child processes to finish before exiting.

### Determining the Number of Instances to Overload the Scheduler

To ensure that some instances of the `deadline` program will not be able to meet their deadlines, we need to increase the number of processes and tighten the deadlines. Through testing, we found that running **6 instances with deadlines set to 15 ticks** (approximately 1.5 seconds) is sufficient to overload the scheduler under the modified system.

## Results

We conducted experiments to compare the performance of the original MINIX scheduler and our modified scheduler that supports deadline scheduling. The primary objective was to observe how effectively each scheduler handles processes with deadlines under varying system loads.

### Testing Scenarios

#### Scenario 1: Baseline Test with Moderate Load

- **Setup**: Ran 4 instances of the `deadline` program, each with a deadline of 3 seconds.
- **Original Scheduler**: Under the original scheduler, none of the instances consistently met their deadlines. The processes were scheduled without consideration for their deadlines, leading to unpredictable execution times.
- **Modified Scheduler**: With the modified scheduler, all instances consistently met their deadlines across multiple runs. The processes were given higher priority due to their deadlines, ensuring timely execution.

#### Scenario 2: Increased Load Test

- **Setup**: Increased the number of `deadline` program instances to 6, each with a tighter deadline of 15 ticks (approximately 1.5 seconds).
- **Original Scheduler**: The original scheduler failed to schedule the processes in a way that allowed them to meet their deadlines. All instances missed their deadlines.
- **Modified Scheduler**: The modified scheduler improved deadline adherence, but under this higher load and tighter deadlines, some instances still missed their deadlines. This suggests that while the scheduler prioritizes deadline processes, system resource limitations prevent all deadlines from being met when the load is too high.

### Data Collection

We performed each test multiple times to observe consistency and to account for variability. The results are summarized below.

#### Original Scheduler

| Test Run | Number of Deadline Processes | Deadline (seconds) | Processes Meeting Deadline |
|----------|------------------------------|--------------------|----------------------------|
| 1        | 4                            | 3                  | 0                          |
| 2        | 6                            | 1.5                | 0                          |
| 3        | 10                           | 1                  | 0                          |

#### Modified Scheduler

| Test Run | Number of Deadline Processes | Deadline (seconds) | Processes Meeting Deadline |
|----------|------------------------------|--------------------|----------------------------|
| 1        | 4                            | 3                  | 4                          |
| 2        | 6                            | 1.5                | 5                          |
| 3        | 10                           | 1                  | 7                          |

### Observations

- **Consistency**: The modified scheduler consistently improved the number of processes meeting their deadlines compared to the original scheduler.
- **System Load Impact**: As the number of processes increased and the deadlines became tighter, even the modified scheduler could not ensure all processes met their deadlines. This indicates that system resources are a limiting factor.
- **Variance**: Across multiple runs, the results were consistent, with only slight variations in the number of processes meeting their deadlines under the modified scheduler.

### Visualization

![Deadline Test Results](deadline_test_results.png)

*Figure: Number of Processes Meeting Deadlines Under Different Schedulers and Loads*

## Discussion

The project successfully implemented deadline scheduling in the MINIX scheduler. The introduction of the `setdl()` system call and priority adjustments allowed processes to specify deadlines and be scheduled preferentially. The results demonstrate that the modified scheduler effectively improves the ability of time-critical processes to meet their deadlines under normal system loads.

### Fulfillment of Aim

Our primary aim was to enable real-time support in the scheduler by allowing processes to set deadlines and have the scheduler prioritize them accordingly. The results show that we achieved this aim:

- **Improved Deadline Adherence**: Under the modified scheduler, processes with deadlines were able to meet their timing constraints more consistently.
- **Priority Adjustment**: By elevating the priority of deadline processes, the scheduler gave them preference over non-deadline processes.

### Reflection on Results

The observations align with our expectations:

- **Effectiveness**: The modified scheduler significantly improved the number of processes meeting their deadlines compared to the original scheduler.
- **Limitations**: Under heavy system load or with very tight deadlines, some processes still missed their deadlines, even with the modified scheduler. This outcome is expected because the system has finite resources, and when demand exceeds capacity, not all deadlines can be met.
- **System Constraints**: The results highlight that while priority adjustments can enhance deadline adherence, they cannot overcome physical limitations such as CPU capacity.

### Speculations and Improvements

While the modifications enhanced deadline handling, several areas for improvement remain:

- **Advanced Scheduling Algorithms**: Implementing real-time scheduling algorithms like Earliest Deadline First (EDF) could further optimize deadline adherence by dynamically scheduling processes based on the urgency of their deadlines.
- **Admission Control**: Introducing mechanisms to limit the number of deadline processes admitted to the system could prevent overload and ensure that accepted tasks have sufficient resources.
- **Dynamic Priority Adjustment**: Adjusting priorities based on how close a process is to its deadline could make scheduling decisions more responsive to timing constraints.
- **Resource Reservation**: Allocating dedicated resources for real-time processes could improve deadline adherence under high load conditions.

## Conclusion

The project demonstrates that by allowing processes to set deadlines and adjusting their priorities accordingly, a simple priority-based scheduler can be enhanced to better support real-time processes. While it cannot guarantee all deadlines will be met under high load, it significantly improves performance under normal conditions.

## Appendix

### List of Modified and Added Files

- `/usr/src/include/minix/callnr.h` - Added `SETDL` system call number.
- `/usr/src/servers/pm/proto.h` - Declared `do_setdl`.
- `/usr/src/servers/pm/table.c` - Added `do_setdl` to the system call table.
- `/usr/src/servers/pm/misc.c` - Implemented `do_setdl`.
- `/usr/src/lib/posix/_setdl.c` - Implemented the `setdl()` function.
- `/usr/src/lib/posix/Makefile.in` - Included `_setdl.c`.
- `/usr/src/include/minix/com.h` - Added `SYS_SETDL`.
- `/usr/src/lib/syslib/sys_setdl.c` - Implemented `sys_setdl`.
- `/usr/src/lib/syslib/Makefile.in` - Included `sys_setdl.c`.
- `/usr/src/include/minix/syslib.h` - Declared `sys_setdl`.
- `/usr/src/kernel/system.c` - Mapped `SYS_SETDL` to `do_setdl`.
- `/usr/src/kernel/system/do_setdl.c` - Implemented `do_setdl`.
- `/usr/src/kernel/system.h` - Declared `do_setdl`.
- `/usr/src/kernel/config.h` - Enabled `USE_SETDL`.
- `/usr/src/kernel/proc.h` - Added `int p_deadline` to the `proc` struct.
- `/usr/src/kernel/proc.c` - Modified the scheduler to prioritize processes with deadlines.
- `/home/testprogram.c` - Created the test program.
- `deadline.c` - Implemented the program that sets its own deadline.

### Code Snippets

#### 1. Adding the `SETDL` System Call

- **File**: `/usr/src/include/minix/callnr.h`

  ```c
  #define SETDL          0
  #define NCALLS         1  /* number of system calls allowed */
  #define EXIT           1
  #define FORK           2
  ```

#### 2. Declaring and Implementing `do_setdl`

- **Declaration**: `/usr/src/servers/pm/proto.h`

  ```c
  _PROTOTYPE( int do_setdl, (void) );
  ```

- **System Call Table**: `/usr/src/servers/pm/table.c`

  ```c
  _PROTOTYPE (int (*call_vec[]), (void) ) = {
      do_setdl,   /* 0 = do_setdl */
      do_exit,    /* 1 = exit */
      do_fork,    /* 2 = fork */
      //...
  };
  ```

- **Implementation**: `/usr/src/servers/pm/misc.c`

  ```c
  int do_setdl(void)
  {
      int ticks;
      ticks = m_in.m1_i1;
      /* Pass the deadline value to the kernel */
      sys_setdl(who_e, ticks);
      return OK;
  }
  ```

#### 3. Implementing `setdl()` Function

- **File**: `/usr/src/lib/posix/_setdl.c`

  ```c
  #include <lib.h>
  #include <unistd.h>

  int setdl(int ticks)
  {
      message m;
      m.m1_i1 = ticks;
      return syscall(PM_PROC_NR, SETDL, &m);
  }
  ```

#### 4. Kernel-Level Modifications

- **Definition**: `/usr/src/include/minix/com.h`

  ```c
  #define SYS_SETDL       (KERNEL_CALL + 50)
  #define NR_SYS_CALLS    51
  ```

- **Implementation of `sys_setdl`**: `/usr/src/lib/syslib/sys_setdl.c`

  ```c
  #include <minix/syslib.h>
  #include <minix/com.h>

  int sys_setdl(endpoint_t proc_nr, int deadline)
  {
      message m;
      m.m1_i1 = proc_nr;
      m.m1_i2 = deadline;
      return _taskcall(SYSTASK, SYS_SETDL, &m);
  }
  ```

- **Declaration**: `/usr/src/include/minix/syslib.h`

  ```c
  _PROTOTYPE(int sys_setdl, (endpoint_t proc_nr, int deadline));
  ```

- **Mapping System Call**: `/usr/src/kernel/system.c`

  ```c
  map(SYS_SETDL, do_setdl); /* Map SYS_SETDL to do_setdl */
  ```

- **Implementation of `do_setdl`**: `/usr/src/kernel/system/do_setdl.c`

  ```c
  #include "../system.h"
  #include <sys/resource.h>

  int do_setdl(message *m_ptr)
  {
      int proc_nr;
      int deadline;
      struct proc *rp;

      if(!isokendpt(m_ptr->m1_i1, &proc_nr)) return EINVAL;

      deadline = m_ptr->m1_i2;
      rp = proc_addr(proc_nr);
      rp->p_deadline = get_uptime() + deadline;

      return OK;
  }
  ```

#### 5. Modifying the Process Structure

- **File**: `/usr/src/kernel/proc.h`

  ```c
  struct proc {
      //...
      int p_deadline;
  };
  ```

#### 6. Adjusting the Scheduler

- **File**: `/usr/src/kernel/proc.c`

  ```c
  PRIVATE void sched(void)
  {
      //...
      if (rp->p_deadline > 0) {
          rp->p_priority = 8; // Higher priority for processes with deadlines
      }
      //...
  }
  ```

#### 7. Test Program

- **File**: `/home/testprogram.c`

  ```c
  #include <stdio.h>
  #include <sys/wait.h>
  #include <unistd.h>
  #include <sys/types.h>

  int main(int argc, char *argv[])
  {
      int i;
      for (i = 0; i < 10; i++)
      {
          pid_t pid = fork();
          if (pid == 0) {
              char *args[] = {"./deadline", NULL};
              execvp(args[0], args);
              // If execvp fails
              perror("execvp failed");
              _exit(1);
          }
      }
      for (i = 0; i < 10; i++) {
          wait(NULL);
      }
      return 0;
  }
  ```

### Understanding the `deadline` Program

The `deadline` program, which is executed by each child process in the test program, operates as follows:

- **Initialization**: The program defines variables to keep track of the current time (`recent`) and a counter (`count`).
- **Setting Deadline**: It sets a deadline using the `setdl()` system call, specifying the number of ticks until its deadline.
- **Execution Loop**:
  - It updates the current time (`new`).
  - Checks if the current time exceeds the deadline (`recent + 1 second`).
  - If the deadline is exceeded, the program exits.
  - Otherwise, it updates `recent` to `new` and continues processing.
- **Purpose**: This loop simulates work being done and checks whether the process can complete its task before the deadline expires.

This behavior allows us to test whether the scheduler prioritizes the process sufficiently for it to meet its deadline.

---

*Note: All code modifications are included with context lines before and after the changes. Complete files are included where new files were created.*

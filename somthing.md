Here’s an organized breakdown of each code section based on the corresponding steps described:

---

### **2.2 HOW TO SET THE DEADLINE FOR PROCESSES**

1. **System Call Definition in Minix Headers**
   - **File**: `/usr/src/include/minix/callnr.h`
   ```c
   #define SETDL 0
   ```

2. **System Call Declaration in Process Manager**
   - **File**: `/usr/src/servers/pm/proto.h`
   ```c
   _PROTOTYPE(int do_setdl, (void));
   ```

3. **System Call Table Entry**
   - **File**: `/usr/src/servers/pm/table.c` (within the `misc.c` section)
   ```c
   do_setdl, /* 0 = do_setdl */
   ```

4. **System Call Implementation**
   - **File**: `/usr/src/servers/pm/misc.c`
   ```c
   int do_setdl(void) {
       printf("This is my system call!\n");
       return 0;
   }
   ```

5. **System Call Library Definition**
   - **File**: `/usr/include/mysyscalllib.h`
   ```c
   #include <lib.h>
   #include <unistd.h>

   int setdl(int ticks) {
       message m;
       m.m1_i1 = ticks;
       return (_syscall(PM_PROC_NR, SETDL, &m));
   }
   ```

6. **Calling the System Call in a Test Program**
   - **File**: `/root/test1.c`
   ```c
   #include <mysyscalllib.h>

   int main(void) {
       setdl(5);
       return 0;
   }
   ```

---

### **2.2.2 CREATE A KERNEL CALL FROM “SERVICE”-SPACE TO THE KERNEL**

1. **Kernel Call Definition in Minix Headers**
   - **File**: `/usr/src/include/minix/com.h`
   ```c
   #define SYS_SETDL (KERNEL_CALL + 50)
   #define NR_SYS_CALLS 51
   ```

2. **Kernel Call Implementation in Syslib**
   - **File**: `/usr/src/lib/syslib/sys_setdl.c`
   ```c
   #include <minix/syslib.h>
   #include <minix/com.h>

   int sys_setdl(endpoint proc_nr, int deadline) {
       message m;
       m.m1_i1 = proc_nr;
       m.m1_i2 = deadline;
       return (_taskcall(SYSTASK, SYS_SETDL, &m));
   }
   ```

3. **Include in Makefile**
   - **File**: `/usr/src/lib/syslib/Makefile.in`
   ```c
   _setdl.c \
   ```

4. **Declaration in Syslib Header**
   - **File**: `/usr/src/include/minix/syslib.h`
   ```c
   _PROTOTYPE(int sys_setdl, (int proc_nr, int deadline));
   ```

5. **Kernel Call Declarations**
   - **File**: `/usr/src/kernel/system.h`
   ```c
   _PROTOTYPE(int do_setdl, (message *m_ptr));
   #if !USE_SETDL
   #define do_setdl do_unused
   #endif
   ```

6. **Enable Kernel Call in System Configuration**
   - **File**: `/usr/src/kernel/config.h`
   ```c
   #define USE_SETDL 1
   ```

7. **Kernel Call Vector Mapping**
   - **File**: `/usr/src/kernel/system.c`
   ```c
   map(SYS_SETDL, do_setdl);
   ```

8. **Implementation of Kernel Call**
   - **File**: `/usr/src/kernel/system/do_setdl.c`
   ```c
   #include "../system.h"
   #include <sys/resource.h>

   int do_setdl(message *m_ptr) {
       int proc_nr;
       int deadline;
       register struct proc *rp;

       if (!isokendpt(m_ptr->m1_i1, &proc_nr)) return EINVAL;

       deadline = m_ptr->m1_i2;
       printf("This works for process %d with deadline %d ticks.\n", proc_nr, deadline);

       rp = proc_addr(proc_nr);
       rp->p_deadline = get_uptime() + deadline;
       printf("Deadline set at %ld ticks.\n", rp->p_deadline);

       return OK;
   }
   ```

9. **Include in Makefile for Kernel**
   - **File**: `/usr/src/kernel/system/Makefile`
   ```c
   $(SYSTEM)(do_setdl.o)
   ```

---

### **2.3 WHERE TO SAVE THE DEADLINE FOR THE PROCESSES?**

1. **Add Deadline Field in Process Structure**
   - **File**: `/usr/src/kernel/proc.h`
   ```c
   int p_deadline;
   ```

---

### **2.4 CHANGE THE CPU SCHEDULER**

1. **Adjust Scheduler to Handle Deadline**
   - **File**: `/usr/src/kernel/proc.c` (within `sched()`)
   ```c
   int time_left = rp->p_ticks_left > 0; /* quantum fully consumed */

   if (rp->p_deadline > 0) {
       rp->p_priority = 8;
   }
   ```

---

### **Test Program to Verify Scheduler**

1. **Forking and Executing Deadline Processes**
   - **Test Program Code**
   ```c
   #include <stdio.h>
   #include <sys/wait.h>
   #include <unistd.h>
   #include <sys/types.h>

   int main(int argc, char *argv[]) {
       int i;
       for (i = 0; i < 15; i++) {
           pid_t pid = fork();
           if (pid == 0) {
               char *argv[] = {"./deadline", NULL};
               execvp(argv[0], argv);
           }
       }
       for (i = 0; i < 15; i++) {
           wait(NULL);
       }
       return 0;
   }
   ```

Each code block above is matched with the corresponding part of the lab instructions for clarity and sequential implementation. Let me know if you need further assistance with any part!

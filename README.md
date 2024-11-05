# Operating Systems and Networks Assignment Report

## Introduction

In this project, I implemented a **lottery scheduler** in the **xv6 operating system**. The lottery scheduler assigns CPU time to processes based on the number of tickets they hold; a process with more tickets has a higher chance of being scheduled. To test and demonstrate that the scheduler works as intended, I wrote a user-level program `test_lottery.c` that creates three processes with tickets in a **3:2:1 ratio** (30, 20, and 10 tickets). I then collected data on the number of CPU time slices each process received over time and plotted the results to visualize the scheduler's behavior.

This assignment involved modifying the xv6 kernel's scheduling mechanism, introducing new system calls, and developing user-level programs to interact with the new scheduler. The primary objectives were to:

- Understand the existing xv6 scheduler.
- Implement a lottery scheduling algorithm.
- Introduce two new system calls: `settickets(int number)` and `getpinfo(struct pstat *)`.
- Develop user-level programs to test and demonstrate the scheduler.
- Generate a graph to illustrate the scheduler's behavior.

The original project instructions can be found at [OSTEP Projects: xv6 Lottery Scheduler](https://github.com/remzi-arpacidusseau/ostep-projects/blob/master/scheduling-xv6-lottery/README.md).

## Setup and Running Instructions

To set up the project and run the test, follow these steps:

1. **Clean the Build Environment**:

   ```bash
   make clean
   ```

2. **Compile the xv6 Operating System**:

   ```bash
   make
   ```

3. **Run xv6 with the Lottery Scheduler and Redirect Output**:

   ```bash
   make qemu-nox CPUS=1 > test_lottery_output.txt
   ```

   - This command runs xv6 in non-graphical mode with a single CPU and redirects the output to `test_lottery_output.txt`.
   - The `test_lottery` program will execute and print the 10 intervals of data into the output file.

4. **Terminate QEMU**:

   - After the `test_lottery` program has completed its execution (ensure the data is fully written), press `Ctrl + A` followed by `X` in your terminal to stop QEMU.

5. **Install Matplotlib (if not already installed)**:

   ```bash
   pip install matplotlib
   ```

6. **Run the Python Script to Generate the Graph**:

   ```bash
   python plot_ticks.py
   ```

   - This script reads `test_lottery_output.txt` and generates a graph visualizing the results.

## Original Code Overview

The original xv6 codebase provided several key components relevant to this assignment:

- **Kernel Files**:
  - `proc.c`: Contains the process table and the scheduler.
  - `proc.h`: Defines the `struct proc` and process states.
  - `sysproc.c`: Implements system calls related to process management.
  - `syscall.h`: Lists system call numbers.

- **User Programs**:
  - Basic utilities like `cat`, `echo`, `ls`, `sh`, etc.

- **Makefile**:
  - The build script to compile the kernel and user programs.

These files form the basis of the xv6 operating system and are essential for implementing and testing the lottery scheduler.

## Modifications Made

### 1. Adding `pstat.h`

**File Added**: `pstat.h`

**Purpose**: Defines the `struct pstat`, which is used by the `getpinfo()` system call to provide process information to user programs.

**Code**:

```c
#ifndef _PSTAT_H_
#define _PSTAT_H_

#include "param.h"

struct pstat {
  int inuse[NPROC];   // Whether this slot of the process table is in use (1 or 0)
  int tickets[NPROC]; // The number of tickets this process has
  int pid[NPROC];     // The PID of each process
  int ticks[NPROC];   // The number of ticks each process has accumulated
};

#endif // _PSTAT_H_
```

**Rationale**: The `struct pstat` is required to pass process information from the kernel to user space. It includes arrays for each field, with the size defined by `NPROC` (the maximum number of processes).

### 2. Modifying `proc.h` to Enhance the Process Structure

**File Modified**: `proc.h`

**Changes Made**:

- Added two new fields to the `struct proc`:

  ```c
  int tickets; // Number of tickets for the lottery scheduler
  int ticks;   // Number of times this process has been scheduled
  ```

**Rationale**:

- **`tickets`**: Stores the number of tickets each process holds, determining their probability of being scheduled.
- **`ticks`**: Tracks the number of times a process has been scheduled (accumulated CPU ticks), which is essential for testing and verification.

### 3. Implementing `settickets(int number)` System Call

**Files Modified**:

- `sysproc.c`: Implemented `sys_settickets()`.
- `syscall.h`: Added the syscall number for `SYS_settickets`.

**Implementation Details**:

- **`sys_settickets()` in `sysproc.c`**:

  ```c
  int sys_settickets(void) {
    int num;
    if (argint(0, &num) < 0 || num < 1)
      return -1;
    myproc()->tickets = num;
    return 0;
  }
  ```

  - Validates that the number of tickets is greater than zero.
  - Sets the `tickets` field of the calling process.

- **`syscall.h`**:

  ```c
  #define SYS_settickets 22
  ```

**Rationale**: The `settickets` system call allows a process to set its own ticket count, influencing its scheduling probability.

### 4. Implementing `getpinfo(struct pstat *)` System Call

**Files Modified**:

- `sysproc.c`: Implemented `sys_getpinfo()`.
- `syscall.h`: Added the syscall number for `SYS_getpinfo`.

**Implementation Details**:

- **`sys_getpinfo()` in `sysproc.c`**:

  ```c
  int sys_getpinfo(void) {
    struct pstat *ps;
    if (argptr(0, (void *)&ps, sizeof(*ps)) < 0 || ps == 0)
      return -1;

    acquire(&ptable.lock);
    for (int i = 0; i < NPROC; i++) {
      struct proc *p = &ptable.proc[i];
      ps->inuse[i] = (p->state != UNUSED);
      ps->tickets[i] = p->tickets;
      ps->pid[i] = p->pid;
      ps->ticks[i] = p->ticks;
    }
    release(&ptable.lock);

    return 0;
  }
  ```

  - Uses `argptr()` to safely obtain the user-space pointer.
  - Copies process information from the kernel to user space.
  - Acquires the process table lock to ensure consistency.

- **`syscall.h`**:

  ```c
  #define SYS_getpinfo 23
  ```

**Rationale**: The `getpinfo` system call provides user programs with access to process scheduling information, essential for testing and visualization.

### 5. Modifying `proc.c` to Implement Lottery Scheduling

**File Modified**: `proc.c`

**Implementation Details**:

#### a. Initializing Tickets and Ticks in `allocproc()`

```c
// Initialize tickets and ticks for the new process
p->tickets = 1; // Default ticket count for new processes
p->ticks = 0;   // Initialize tick count to zero
```

- **Rationale**: New processes start with a default of 1 ticket and zero accumulated ticks.

#### b. Ensuring Child Inherits Tickets in `fork()`

```c
// Copy parent's tickets to child
np->tickets = curproc->tickets;
```

- **Rationale**: Ensures consistency in scheduling behavior for child processes unless explicitly changed via `settickets()`.

#### c. Implementing Random Number Generator

```c
// Simple Linear Congruential Generator
unsigned long rand(void) {
  static unsigned long next = 1;
  next = next * 1664525 + 1013904223;
  return next;
}
```

- **Rationale**: A simple Linear Congruential Generator (LCG) provides the randomness needed for the lottery scheduler.

#### d. Modifying the `scheduler()` Function

```c
void scheduler(void) {
  struct proc *p;
  struct cpu *c = mycpu();
  c->proc = 0;

  for (;;) {
    sti();
    acquire(&ptable.lock);

    int total_tickets = 0;
    // Calculate total tickets of all RUNNABLE processes
    for (p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
      if (p->state == RUNNABLE)
        total_tickets += p->tickets;
    }

    if (total_tickets > 0) {
      int winning_ticket = rand() % total_tickets;
      int current_ticket = 0;

      for (p = ptable.proc; p < &ptable.proc[NPROC]; p++) {
        if (p->state != RUNNABLE)
          continue;

        current_ticket += p->tickets;

        if (current_ticket > winning_ticket) {
          // Found the winning process
          p->ticks++; // Increment tick count
          c->proc = p;
          switchuvm(p);
          p->state = RUNNING;
          swtch(&(c->scheduler), p->context);
          switchkvm();
          c->proc = 0;
          break;
        }
      }
    }

    release(&ptable.lock);
  }
}
```

- **Total Tickets Calculation**: Aggregates the number of tickets from all runnable processes.
- **Winning Ticket Selection**: A random number between `0` and `total_tickets - 1` is generated.
- **Process Selection Loop**:
  - Iterates over the process table.
  - Keeps a running total (`current_ticket`) of tickets.
  - When `current_ticket` exceeds `winning_ticket`, the process holds the winning ticket.
- **Context Switching**:
  - Increments the chosen process's `ticks` count.
  - Sets the process state to `RUNNING`.
  - Performs a context switch to start executing the process.

**Rationale**: The modifications implement the lottery scheduling algorithm by randomly selecting a process based on ticket counts, ensuring fairness proportional to the tickets held.

### 6. Updating the Makefile

**File Modified**: `Makefile`

**Changes Made**:

- **Added New User Programs**:

  ```makefile
  UPROGS=\
    ...
    _ps\
    _test_lottery\
    ...
  ```

- **Included New Source Files**:

  ```makefile
  EXTRA=\
    ...
    ps.c test_lottery.c\
    ...
  ```

- **Adjusted Compilation Flags**:

  ```makefile
  CFLAGS += -Wno-stringop-overflow
  CFLAGS += -Wno-error=infinite-recursion
  ```

**Rationale**: The Makefile was updated to compile the new user programs and to suppress specific warnings that could halt the build process.

### 7. Developing User-Level Programs

#### a. `test_lottery.c`

**Purpose**: Tests the lottery scheduler by creating three processes with different ticket counts in a 3:2:1 ratio (30, 20, and 10 tickets).

**Implementation**:

- **Process Creation and Ticket Assignment**:

  ```c
  int pid1, pid2, pid3;
  int tickets[3] = {30, 20, 10};

  pid1 = fork();
  if (pid1 == 0) {
    settickets(tickets[0]);
    infinite_loop();
    exit();
  } else if (pid1 > 0) {
    pid2 = fork();
    if (pid2 == 0) {
      settickets(tickets[1]);
      infinite_loop();
      exit();
    } else if (pid2 > 0) {
      pid3 = fork();
      if (pid3 == 0) {
        settickets(tickets[2]);
        infinite_loop();
        exit();
      }
      // Parent process continues...
    }
  }
  ```

- **Synchronization Using Pipes**:

  - Used pipes to ensure child processes set their tickets before starting the infinite loop.
  - Prevents the parent from collecting data before the children have properly initialized.

- **Data Collection**:

  ```c
  struct pstat ps;
  for (int interval = 0; interval < INTERVALS; interval++) {
    sleep(SLEEP_TIME);
    if (getpinfo(&ps) != 0) {
      printf(1, "getpinfo failed\n");
      exit();
    }
    // Print tick counts for child processes
  }
  ```

**Rationale**: This program simulates processes with different ticket counts competing for CPU time, allowing us to observe the scheduler's behavior.

#### b. Synchronization Mechanism with Pipes

**Implementation**:

- **Parent Process**:

  ```c
  // Create pipes
  for (int i = 0; i < 3; i++) {
    if (pipe(pipes[i]) < 0) {
      printf(1, "Pipe creation failed\n");
      exit();
    }
  }

  // After forking children
  // Close write ends in parent
  for (int i = 0; i < 3; i++) {
    close(pipes[i][1]);
  }

  // Wait for children to set tickets
  char buf;
  for (int i = 0; i < 3; i++) {
    read(pipes[i][0], &buf, 1);
    close(pipes[i][0]); // Close read end after use
  }
  ```

- **Child Processes**:

  ```c
  // Close read end of the pipe
  close(pipes[i][0]);

  // Set tickets
  settickets(tickets[i]);

  // Notify parent
  write(pipes[i][1], "1", 1);
  close(pipes[i][1]); // Close write end

  // Enter infinite loop
  infinite_loop();
  ```

**Rationale**: Ensures that the parent process begins data collection only after the child processes have set their ticket counts, preventing race conditions.

#### c. `plot_ticks.py`

**Purpose**: Reads the output from `test_lottery` and generates a graph showing the ticks accumulated over time by each process.

**Implementation**:

- **Parsing Output**:

  ```python
  with open('test_lottery_output.txt', 'r') as f:
    lines = f.readlines()

  # Extract intervals, PIDs, tickets, and ticks
  ```

- **Data Alignment**:

  - Ensures that all tick lists have the same length for plotting.

- **Plotting**:

  ```python
  plt.figure(figsize=(10, 6))
  for pid in sorted(tickets.keys()):
    plt.plot(intervals, ticks[pid], marker='o', label=f'PID {pid} ({tickets[pid]} tickets)')
  plt.title('Ticks Accumulated Over Time for Different Processes')
  plt.xlabel('Interval')
  plt.ylabel('Ticks')
  plt.legend()
  plt.grid(True)
  plt.savefig('ticks_over_time.png')
  plt.show()
  ```

**Rationale**: Visualizes the effectiveness of the lottery scheduler, demonstrating that processes receive CPU time proportional to their tickets.

## Results and Observations

**Data Collected**:

```
Interval 1:
PID: 3, Tickets: 30, Ticks: 138
PID: 4, Tickets: 20, Ticks: 68
PID: 5, Tickets: 10, Ticks: 29

Interval 2:
PID: 3, Tickets: 30, Ticks: 170
PID: 4, Tickets: 20, Ticks: 88
PID: 5, Tickets: 10, Ticks: 39

Interval 3:
PID: 3, Tickets: 30, Ticks: 255
PID: 4, Tickets: 20, Ticks: 137
PID: 5, Tickets: 10, Ticks: 65

Interval 4:
PID: 3, Tickets: 30, Ticks: 274
PID: 4, Tickets: 20, Ticks: 154
PID: 5, Tickets: 10, Ticks: 79

Interval 5:
PID: 3, Tickets: 30, Ticks: 347
PID: 4, Tickets: 20, Ticks: 195
PID: 5, Tickets: 10, Ticks: 107

Interval 6:
PID: 3, Tickets: 30, Ticks: 592
PID: 4, Tickets: 20, Ticks: 321
PID: 5, Tickets: 10, Ticks: 170

Interval 7:
PID: 3, Tickets: 30, Ticks: 644
PID: 4, Tickets: 20, Ticks: 356
PID: 5, Tickets: 10, Ticks: 181

Interval 8:
PID: 3, Tickets: 30, Ticks: 717
PID: 4, Tickets: 20, Ticks: 412
PID: 5, Tickets: 10, Ticks: 195

Interval 9:
PID: 3, Tickets: 30, Ticks: 909
PID: 4, Tickets: 20, Ticks: 548
PID: 5, Tickets: 10, Ticks: 241

Interval 10:
PID: 3, Tickets: 30, Ticks: 1028
PID: 4, Tickets: 20, Ticks: 627
PID: 5, Tickets: 10, Ticks: 273
```

**Graphical Representation**:

My graph can be found as ticks_over_time.png in the main folder.
![ticks_over_time](https://github.com/user-attachments/assets/2550ec47-8cbc-4658-bf87-ba7940d31745)


**Analysis**:

- **Proportional Scheduling**:
  - Process with 30 tickets accumulated the most ticks.
  - Process with 20 tickets accumulated fewer ticks.
  - Process with 10 tickets accumulated the least ticks.
- **Tick Ratios**:
  - By Interval 10:
    - Process 3 (30 tickets): 1028 ticks
    - Process 4 (20 tickets): 627 ticks
    - Process 5 (10 tickets): 273 ticks
  - Ratios approximate the ticket ratios (3:2:1), confirming that CPU time allocation is proportional to ticket counts.

**Conclusion**:

The data and graph confirm that the lottery scheduler is functioning as intended, allocating CPU time proportional to the number of tickets each process holds.

## Challenges Faced

### 1. Random Number Generation in the Kernel

- **Issue**: The standard `rand()` function is not available in the kernel.
- **Solution**: Implemented a simple Linear Congruential Generator (LCG) within `proc.c`.

  ```c
  unsigned long rand(void) {
    static unsigned long next = 1;
    next = next * 1664525 + 1013904223;
    return next;
  }
  ```

- **Considerations**: Ensured the generator is fast and does not introduce significant overhead.

### 2. Safe User-Kernel Data Transfer

- **Issue**: Passing pointers from user space to kernel space can lead to security vulnerabilities.
- **Solution**: Used `argptr()` in system calls to safely copy pointers and performed null checks.

  ```c
  if (argptr(0, (void *)&ps, sizeof(*ps)) < 0 || ps == 0)
    return -1;
  ```

### 3. Synchronization Between Parent and Child Processes

- **Issue**: Ensuring that child processes set their ticket counts before entering the infinite loop.
- **Solution**: Implemented pipes for synchronization, allowing the parent to wait until children have set their tickets.

### 4. Accurately Tracking Process Ticks

- **Issue**: The `ticks` field needed to be incremented each time a process is scheduled.
- **Solution**: Incremented `p->ticks` in the scheduler when the process is selected to run.

  ```c
  p->ticks++; // Increment tick count
  ```

## Conclusion

By implementing a lottery scheduler in xv6, we successfully demonstrated a scheduling algorithm that allocates CPU time proportional to the number of tickets held by each process. The use of new system calls `settickets()` and `getpinfo()` allowed for dynamic adjustment and monitoring of process tickets and ticks. The `test_lottery.c` program, along with the synchronization using pipes, ensured accurate data collection. The generated graph validated the effectiveness of the scheduler, showing that processes received CPU time in ratios closely matching their ticket allocations.

This project provided valuable insights into kernel development, process scheduling, and the implementation of system calls. It highlighted the importance of careful synchronization and data handling in kernel programming.

## Appendix

### A. Code Snippets

#### 1. `pstat.h`

```c
#ifndef _PSTAT_H_
#define _PSTAT_H_

#include "param.h"

struct pstat {
  int inuse[NPROC];   // Whether this slot of the process table is in use (1 or 0)
  int tickets[NPROC]; // The number of tickets this process has
  int pid[NPROC];     // The PID of each process
  int ticks[NPROC];   // The number of ticks each process has accumulated
};

#endif // _PSTAT_H_
```

#### 2. Modified `proc.h`

```c
// In struct proc:
int tickets; // Number of tickets for the lottery scheduler
int ticks;   // Number of times this process has been scheduled
```

#### 3. `test_lottery.c`

```c
#include "types.h"
#include "stat.h"
#include "user.h"
#include "pstat.h"

#define NPROC 64
#define INTERVALS 10  // Number of data collection intervals
#define SLEEP_TIME 50 // Time to sleep between intervals

void infinite_loop() {
  volatile int i = 0;
  while (1) {
    i++;
  }
}

int main(int argc, char *argv[]) {
  int pid1, pid2, pid3;
  int pids[3];
  int tickets[3] = {30, 20, 10};
  int pipes[3][2]; // Pipes for synchronization

  // Create pipes
  for (int i = 0; i < 3; i++) {
    if (pipe(pipes[i]) < 0) {
      printf(1, "Pipe creation failed\n");
      exit();
    }
  }

  pid1 = fork();
  if (pid1 == 0) {
    // First child process with 30 tickets
    close(pipes[0][0]); // Close read end
    settickets(tickets[0]);
    write(pipes[0][1], "1", 1); // Notify parent
    close(pipes[0][1]);
    infinite_loop();
    exit();
  }
  // Similar code for pid2 and pid3...

  // Parent process code...
}
```

#### 4. `plot_ticks.py`

```python
import matplotlib.pyplot as plt

# Data parsing and plotting code...

plt.title('Ticks Accumulated Over Time for Different Processes')
plt.xlabel('Interval')
plt.ylabel('Ticks')
plt.legend()
plt.grid(True)
plt.savefig('ticks_over_time.png')
plt.show()
```

### B. Test Output

```
Interval 1:
PID: 3, Tickets: 30, Ticks: 138
PID: 4, Tickets: 20, Ticks: 68
PID: 5, Tickets: 10, Ticks: 29

... (Intervals 2 to 10)
```

### C. Graph

*(Include the generated graph `ticks_over_time.png` here)*

### D. Makefile Changes

- **Added User Programs**:

  ```makefile
  UPROGS=\
    ...
    _ps\
    _test_lottery\
    ...
  ```

- **Included New Source Files**:


  ```makefile
  EXTRA=\
    ...
    ps.c test_lottery.c\
    ...
  ```

- **Compilation Flags**:

  ```makefile
  CFLAGS += -Wno-stringop-overflow
  CFLAGS += -Wno-error=infinite-recursion
  ```

## References

- **xv6 Source Code**: [MIT xv6](https://pdos.csail.mit.edu/6.828/2012/xv6.html)
- **Lottery Scheduling Algorithm**: [Lottery Scheduling](http://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched-lottery.pdf)
- **Original Project Instructions**: [OSTEP Projects: xv6 Lottery Scheduler](https://github.com/remzi-arpacidusseau/ostep-projects/blob/master/scheduling-xv6-lottery/README.md)
- **Installing xv6**: [OSTEP Projects: Installing xv6](https://github.com/remzi-arpacidusseau/ostep-projects/blob/master/INSTALL-xv6.md)


## AI statement
In completing this assignment, I consulted ChatGPT to gain insights into coding practices and get feedback on my code modifications. I used ChatGPT to help clarify concepts related to the lottery scheduler, including how to implement the settickets and getpinfo system calls, and to understand proper error handling techniques. By providing detailed explanations and suggesting best practices, ChatGPT guided me through each step of the code-writing process, enabling me to improve my understanding and complete the task efficiently.

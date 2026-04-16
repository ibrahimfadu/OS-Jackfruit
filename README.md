## Multi-Container Runtime (OS Jackfruit Project)

---

## 1. Team Information

* Team Member 1: Qamar Ahmed
* SRN: PES2UG24CS904

* Team Member 2: Joshwin Paul
* SRN: PES2UG24CS208

---

## 2. Build, Load, and Run Instructions

### Build the Project

```bash
make
```

---

### Load Kernel Module

```bash
sudo insmod monitor.ko
```

---

### Verify Device Creation

```bash
ls /dev/container_monitor
```

---

### Start Supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

---

### Create Root Filesystems

```bash
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

---

### Start Containers

```bash
sudo ./engine start alpha ./rootfs-alpha /bin/sh
sudo ./engine start beta ./rootfs-beta /bin/sh
```

---

### View Running Containers

```bash
sudo ./engine ps
```

---

### Run Workloads

```bash
cp cpu_hog ./rootfs-alpha/
sudo ./engine start alpha ./rootfs-alpha /cpu_hog
```

---

### View Logs

```bash
cat logs/alpha.log
```

---

### Stop Containers

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
```

---

### Unload Kernel Module

```bash
sudo rmmod monitor
```

---

## 3. Demo with Screenshots

### <img width="1855" height="670" alt="1" src="https://github.com/user-attachments/assets/b082886e-5feb-423f-858f-a036b7bc3108" />


Shows multiple containers running under a single supervisor process.

---

### <img width="1860" height="670" alt="2" src="https://github.com/user-attachments/assets/7fefe450-dd48-4a83-ab9f-94d76ad2e793" />


Displays container ID, PID, and state using the `ps` command.

---

### <img width="858" height="292" alt="4" src="https://github.com/user-attachments/assets/0d2a6048-dd26-4669-a7c3-e2d409e7ed80" />


Shows captured output of a container stored in log files using bounded-buffer logging.

---

### <img width="1864" height="677" alt="3" src="https://github.com/user-attachments/assets/5ac5f49d-3709-4d1f-a33a-c5d98b4ab43b" />


Demonstrates communication between CLI and supervisor using IPC (FIFO/socket).

---

### <img width="1646" height="506" alt="5 and 6" src="https://github.com/user-attachments/assets/8b5dbe26-66ff-4a4e-b20b-8bdfdbdf08b6" />


Shows both soft-limit warning and hard-limit enforcement using kernel logs:

```bash
sudo dmesg | grep container_monitor
```

---

### <img width="1256" height="563" alt="7" src="https://github.com/user-attachments/assets/e28b9974-3196-4dc4-a447-7de5f5ed2560" />


Comparison between containers with different nice values showing CPU allocation differences.

---

### <img width="2824" height="303" alt="8" src="https://github.com/user-attachments/assets/d78fd9cf-ff87-4eb4-8d0f-9e003244540c" />


Shows no zombie processes related to containers after execution.
Note: Any unrelated system-level defunct processes are ignored.

---

## 4. Engineering Analysis

### Isolation Mechanisms

The runtime uses Linux namespaces such as PID, UTS, and mount namespaces to isolate containers. Each container has its own process space and filesystem view using `chroot`. However, all containers share the same underlying host kernel.

---

### Supervisor and Process Lifecycle

A long-running supervisor process manages all containers. It creates containers using `clone()`, tracks metadata such as PID and state, and handles `SIGCHLD` signals to properly reap child processes and avoid zombies.

---

### IPC, Threads, and Synchronization

Two IPC mechanisms are used:

* FIFO/socket for CLI → supervisor communication
* Pipes for container → supervisor logging

A bounded buffer is used between producer and consumer threads. Mutex locks and condition variables ensure synchronization, preventing race conditions, data loss, and deadlocks.

---

### Memory Management and Enforcement

RSS (Resident Set Size) represents the actual physical memory used by a process. A soft limit generates warnings, while a hard limit enforces termination. This enforcement is implemented in kernel space to ensure reliability and prevent bypassing.

---

### Scheduling Behavior

Experiments show that processes with lower nice values receive more CPU time. CPU-bound processes with higher priority complete faster, while I/O-bound processes demonstrate better responsiveness due to scheduler design.

---

## 5. Design Decisions and Tradeoffs

* Used FIFO/socket IPC instead of shared memory for simplicity and reliability
* Used `chroot` instead of `pivot_root` for easier implementation
* Used mutex-based synchronization since operations may block
* Used threads for logging to avoid blocking the supervisor

Tradeoff: Simpler design reduces complexity but may not be as secure or optimized as production container runtimes.

---

## 6. Scheduler Experiment Results

Two CPU-bound containers were run with different nice values:

* `alpha` → nice = 10 (low priority)
* `beta` → nice = -5 (high priority)

Observation:

* The higher priority container completed faster
* It received more CPU time from the scheduler

Conclusion:
Linux Completely Fair Scheduler (CFS) allocates CPU time based on process priority, ensuring fairness while favoring higher-priority processes.

---

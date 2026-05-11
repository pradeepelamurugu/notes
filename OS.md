# How Operating Systems Work — Video Notes

## Overview

An operating system is the most underappreciated software ever written. It manages hundreds of processes, memory, hardware, and more — all simultaneously. The first OS, **GM-NAA I/O**, was shipped in **1956** at General Motors.

---

## Stage 1 — The Boot Loader

- You press the power button → electricity hits the motherboard → CPU wakes up in its most primitive state
- No memory management, no concept of files — just a single core executing instructions at a hard-coded address in **firmware**
- Modern machines use **UEFI**; older ones used **BIOS**
- Firmware's job: wake up just enough hardware to find a disk, then hand off to a **bootloader**
  - **Linux**: GRUB (Grand Unified Bootloader)
  - **Mac**: iBoot
  - **Windows**: Bootmgr
- Bootloader's job: find the kernel on disk and load it into RAM
- After handoff, the CPU runs kernel code with **full hardware privileges**

---

## Stage 2 — Privilege Rings

- CPUs are protected by multiple privilege levels
- On x86, there are **4 rings**, but only 2 matter:
  - **Ring 0** — Kernel: can do basically anything
  - **Ring 3** — User space: runs applications but must ask permission for everything else
- Without this separation, any program could read another program's memory and crash the whole system
- With privilege rings, a buggy program can usually only crash itself

---

## Stage 3 — Virtual Memory

- When a program requests a memory address, that address is **fake** (virtual)
- It gets translated to a real physical address by hardware called the **MMU** (Memory Management Unit)
- The MMU uses a data structure called a **page table** (built by the kernel)
- Memory is handed out in chunks called **pages** (typically 4 KB each)
- Each process gets its **own page table** → processes can't read each other's memory (e.g., browser can't read password manager memory)
- The MMU caches recent translations in the **TLB** (Translation Lookaside Buffer)
- If a program touches a page not in RAM → **page fault** → kernel loads the page from disk and resumes the program transparently

---

## Stage 4 — The File System

- A disk at the lowest level is just a long sequence of numbered blocks
- A **file system** abstracts this into files and folders
- The kernel **mounts** the file system
- Files are stored as **index nodes (inodes)**:
  - Contain: size, permissions, timestamps, pointer to actual data block on disk
  - Do **NOT** contain: the file name
- **File names** live in **directories**, which are special files mapping names → inode numbers
- This is why multiple file names can point to the same file (hard links)
- Common file systems: `ext4` (Linux), `NTFS` (Windows), `APFS` (Mac)
- Modern file systems use **journaling**: writes intentions before writing data → prevents corruption if power is lost mid-write

---

## Stage 5 — Device Drivers & Interrupts

- The kernel loads **device drivers** — specialized code that translates generic kernel requests into hardware-specific instructions
- Each piece of hardware (GPU, Wi-Fi, keyboard) gets its own driver loaded from disk
- Drivers run in **kernel mode** → one buggy driver can crash the entire OS (e.g., Windows BSOD often comes from graphics drivers)
- The kernel then enables **interrupts**:
  - Hardware fires an electrical signal → yanks the CPU out of whatever it's doing → jumps to an **interrupt handler** in the kernel
  - Examples: moving the mouse fires an interrupt, Wi-Fi receiving data fires an interrupt
  - The machine is essentially driven by tiny electrical signals from hardware saying "something happened, deal with it"

---

## Stage 6 — PID 1: The First Process

- The kernel creates the first user space program: **PID 1**
  - On Linux, this is usually **systemd**
- Creating a process involves:
  1. Allocating memory
  2. Loading an executable from disk
  3. Setting up the virtual address space and page table
  4. Adding an entry to the **process table**
- Every process gets a **PID** (Process ID)
- **PID 1** is the ancestor of every other process — if it dies, the kernel panics and the system goes down
- PID 1 runs in **Ring 3 (user space)** — from this point, everything must ask the kernel for permission

---

## Stage 7 — System Calls

- When a process wants to read a file, it can't access the disk directly — the CPU will physically refuse
- Instead, it makes a **system call**:
  1. Puts arguments into specific registers
  2. Triggers a special instruction
  3. CPU switches from Ring 3 → Ring 0
- This ring boundary is the reason your computer is secure
- Even `printf()` in C makes a `write` system call under the hood
- Linux has ~**400 system calls** — the actual API of the computer; everything else is a library on top
- Two of the most important: **`fork`** and **`exec`** — used to create new processes

---

## Stage 8 — The Scheduler

- A computer may have 8 CPU cores but hundreds of running processes
- The **scheduler** acts like an air traffic controller:
  - Processes = airplanes
  - CPU = runway
- Modern Linux uses **EEVDF** (Earliest Eligible Virtual Deadline First) to ensure every process gets its fair share of CPU time

---

## Stage 9 — Threads

- Some applications need to do multiple things at once without the overhead of multiple processes
- A **thread** shares the same memory and file descriptors as the parent process, but has its own:
  - Stack
  - Program counter
- Threads enable parallelism within a single program
- Risk: **race conditions** — two threads writing to the same variable simultaneously
- Modern languages try to prevent this:
  - **Go**: goroutines
  - **Rust**: borrow checker refuses to compile code that could race

---

## Stage 10 — IPC (Interprocess Communication)

- Different processes can't share memory like threads do
- IPC allows processes to communicate safely
- Common IPC mechanisms:
  - **Pipes** (invented 1973): output of one process becomes input of another (e.g., `cat file.txt | grep "term"`)
  - **Sockets**
  - **Message queues**

---

## Shutdown

1. PID 1 sends **SIGTERM** to every process — a polite request to stop
2. Well-behaved processes save state and quit
3. After a timeout, **SIGKILL** is sent — no argument, process is terminated
4. File system flushes its journals and unmounts
5. Drivers release hardware
6. Kernel syncs memory to disk
7. Interrupts are disabled
8. CPU halts
9. Firmware cuts power → screen goes black

---

## Summary Table

| Stage | Concept            | Key Idea                                              |
|-------|--------------------|-------------------------------------------------------|
| 1     | Boot Loader        | Loads kernel from disk into RAM                       |
| 2     | Privilege Rings    | Ring 0 (kernel) vs Ring 3 (user space)               |
| 3     | Virtual Memory     | Fake addresses → MMU → real addresses via page tables |
| 4     | File System        | Inodes, directories, journaling                       |
| 5     | Drivers/Interrupts | Hardware drivers + interrupt-driven I/O               |
| 6     | First Process      | PID 1 (systemd) — ancestor of all processes           |
| 7     | System Calls       | The only API to the kernel (~400 on Linux)            |
| 8     | Scheduler          | EEVDF — fair CPU time sharing across processes        |
| 9     | Threads            | Parallelism within a process; beware race conditions  |
| 10    | IPC                | Pipes, sockets, message queues for process comms      |

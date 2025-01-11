### **《八股》**
1. #### **Memory Related**
1. ##### **MMU(Memory Management Unit)**
- Hardware in computer that translates virtual memory address into physical memory addresses.
- It allows programs to run in isolated memory spaces and makes manage memory easier.
- Address Translation: Logical Address → Linear Address → Physical Address 
  - Segment Unit: convert Logical Address → Linear Address (through Global Descriptor Table)
  - Memory Management Unit: convert Linear Address → Physical Address (through TLB & Page Table) 
  - `pmap PID`: view memory mappings of a process.
2. ##### **OOM(Out of Memory)**
- when system runs out of memory and can't allocate more for programs. (for all available included disk swap space)
- When this happens, the OOM Killer is triggered to stop low-priority processes and free up memory.
- How to check?
  - `dmesg`: see if the OOM Killer was triggered.
  - `/proc/$PID/oom\_score`: see which processes are more likely to be killed. 
- **How to deal with high memory usage?**
  - **Short-term fixes:**
    - Increase the **nice value** to decrease the priority, so the memory request will less, if speed is not important.
    - `kill -SIGSTOP PID`: Pause unimportant processes.
    - `kill -l`: see the number of SIGSTOP
    - `kill -num PID`: kernel swap those memory to disk and free the memory for import task.
  - **What is nice value**? controls a process’s priority.
    - A higher nice value means lower priority and shorter CPU time, while a lower nice value means higher priority.
    - How to adjust it?	
      - Use the `nice` command when starting a process.
      - `renice`: change the priority of an existing process.
  - **Long-term fixes:**
    - `valgrind` or `ltrace`: Find memory leaks.
    - `ulimit`: Restrict memory usage.
    - Move the heavy process to another server.
`free -m`: view memory usage.
`top`: view the memory usage of each process in real time.
`vmstat`: monitor system performance.
3. ##### **Swap: extra space on disk used when RAM is full**
- The system moves less-used memory pages from RAM to Swap so that more memory is available for important tasks.
- Swap space: 
  - partition: Is an independent section of the hard disk used solely for swapping; no other files can reside there. 
  - file: Is a special file in the file system that resides among your system and data files. 
- Why use Swap? 1. additional memory 2. prevent OOM errors
- When does system Swap?
  - High Memory Utilization: When RAM usage exceeds a critical threshold (e.g., 70-80%).
  - Memory Overcommitment: When processes request more memory than physically available.
  - Application Behavior: Applications with large memory footprints or memory leaks.
- Disadvantage:
  - **Slower Performance**: Swap is stored on disk, and disk I/O is much slower than RAM.
  - **Increased Latency**: Processes needing swapped-out pages must wait for the disk I/O to load them back into RAM.
  - **I/O bottleneck**: Heavy swapping increase disk usage, potentially causing bottlenecks.
  - **Disk Wear**: Frequent swapping can cause wear on SSDs due to excessive writes.
`free -m`: Swap usage.
`swapon -s`: lists Swap status.
- increase Swap: Create a Swap file or **partition** and activate it with swapon.
4. ##### **Why Kernel memory is different from user memory?** 
- Kernel memory:
  - Cannot be swapped to disk(not pageable) for performance and stability reasons.
  - Managed differently from user memory(:lazy allocation, only when actual access). The kernel allocates all required memory when a process starts.
- Why this matters: Errors in kernel memory (like illegal access) cause major issues, such as system crashes (kernel oops).
- How to check?
  - `cat /proc/meminfo`: see kernel memory usage.
- Linux uses **Slab Allocator** to manage kernel memory efficiently.
- Zones of Kernel memory
  - ZONE\_DMA: For devices that need direct memory access.
  - ZONE\_NORMAL: The main memory used by the kernel.
  - ZONE\_HIGHMEM: Extra memory not directly used by the kernel.
5. ##### **Slab Layer: like a free list**
- Function: 
  - A slab layer divide different objects into groups called caches. There is one cache per type. 
  - If an object is needed, the Allocator fetches it from a slab instead of allocating new memory.
- Structure: Cache → Muti Slabs → Muti Objects 
- Status of slabs: 1. Full 2. Partial 3. Empty 
- Process: 
1) When the kernel requires a new data structure, the kernel will return a pointer to an already allocated, but unused data structure from a partial slab(or an empty slab if no partial slab exists). 
2) If neither partial slab nor empty slab exists, use allocate interface to allocate a new slab. 
3) The free interface is called only when available memory grows low and system is attempting to free memory. 
`slabtop`: see kernel slab usage.
6. ##### **Virtual Memory**
- A memory management technique that provides processes with the illusion of having a large, contiguous address space. It maps virtual addresses to physical memory using a page table.
- Advantages: 
  - Efficient Memory Use: Allows overcommitting memory.
  - Isolation: Protects processes from accessing each other's memory.
  - Convenience: Simplifies programming with a uniform memory model.
- Follow-Up Topics
  - Page Tables: How virtual addresses are translated to physical addresses.
  - TLB (Translation Lookaside Buffer): A cache for speeding up address translation.
  - Page Faults: Occur when a process accesses a page not in memory.
2. #### **Process Related**
1. ##### **CPU load is very high but CPU usage is quit low, why?**
- Probably large amount of processes are under uninterruptible sleep status (waiting for I/O) which makes the process queue very long.
- Since Linux system average load measures the number of the process in the status of both Running and Waiting for Run(included D status). But the number of processes in Running status is quite small. 
- Solution: 
  - `iotop`: see which process has a large I/O. 
  - Increase bottleneck disk device performance. 
  - `vmstat`: see the swap memory usage, if swap uses too much I/O resources, we should **increase memory capacity** or **check if memory leak**. 
2. ##### **Init Process**
- The init process is the first process started by the Linux kernel when the system boots up. Its PID is always 1. It initializes the system based on the runlevel.
- Where to find it? The behavior is defined in `/etc/inittab`.
- It’s the ancestor of all processes. If it fails, the system becomes unstable.
3. ##### **Create new Process**
1) A parent process calls the `fork() system call`. **fork()** creates a new process by duplicating the parent’s memory space(copy on write).
2) The parent continues running, and the child either executes the same code or `calls exec()` to load a new program.
3) The parent process can use `wait()` or `waitpid()` to check the child status.
4) When a child exits, it enters a zombie state until the parent retrieves its status.
- Key terms:
  - **Copy on Write**: Instead of copy the parent’s entire memory, the child only gets reference. Actual copy happens when the child modifies the memory.
  - **Zombie state**: A process that has finished executing but hasn’t been cleaned up by its parent.
4. ##### **Process Status**
- RUNNING (R): Actively running or ready to run.
- INTERRUPTIBLE (S): Sleeping, but can wake up when needed.
- UNINTERRUPTIBLE (D): Sleeping, but cannot be killed (e.g., waiting for I/O).
  - This is why some processes in the D state can only cleared by a reboot or wait for I/O to respond.
- STOPPED (T): Suspended, usually by a signal like SIGSTOP.
- ZOMBIE (Z): Finished execution, but the parent hasn’t cleaned it up.
5. ##### **PCB(Process control block)/process descriptor**
- The PCB is a data structure in the OS that stores all information about process:
  - PID: Process ID.
  - Status: Current state (e.g., running, sleeping).
  - Program Counter: Address of the next instruction to execute.
  - Memory Limits: Information about memory allocation, including page tables.
  - Open Files List: Files currently in use by the process.
  - Parent/Child Relationships: Links to the parent and child processes.
6. ##### **how does fork() work internally?? (use for create new process)**
- `fork()` is implemented using the `clone() system call`, which internally `calls do\_fork()`.
- A new `task\_struct` (process descriptor) is created for the child process.
- The child’s initially set to **UNINTERRUPTIBLE**, so it doesn’t run immediately.
- Resources like file descriptors are shared between parent and child.
- The child then assigned a new PID using `alloc\_pid()`.
- Once ready, the child process starts running.
- **Difference between fork() and vfork(): copy page table or not**
  - `vfork()` is optimized for when the child immediately calls exec(). It **doesn’t copy the parent’s page table**, and the parent is blocked until the child finishes.
7. ##### **Parent exit before child**
- The OS need to reparent the child process to another process; otherwise the child process will forever remain in a zombie state after it `call exit()`. 
- Solution: 
  - Reparent a task’s children when calling exit() to another process in the current thread group . 
  - If failed, reparent its children to init process. 
8. ##### **Process life cycle**
- Generate: `call fork()` → `call syscall clone()` → copy on write 
- If need to run a different program: `call syscall exec()` 
- Termination: `call syscall exit() `
- Suspend process until one child process exit: `call syscall wait4()` 
9. ##### **Linux Process Scheduling**
- CFS (Completely Fair Scheduler):
  - Linux uses the CFS to schedule processes. It **picks the process with the smallest vruntime (virtual runtime) to run next.**
  - use **red-black tree** to keep track of runnable processes.
- Real-time scheduling policies:
  - FIFO: First-In-First-Out, runs until the task blocks or yields.
  - Round-Robin: Each task gets a fixed time slice before switching.
10. ##### **Interprocess Communication (IPC)**
   - Shared Memory: Fastest method, but requires synchronization.
   - Signals: Notifications sent by the kernel or other processes (e.g., `SIGKILL`).
   - Pipes: Data channels between processes.
   - Message Queues: processes can send and receive messages.
1. ##### **pstree**
- The `pstree` command shows processes in a tree structure, displaying parent-child relationships.
`pstree`: display all processes as a tree.
`pstree -p`: include process IDs.
`pstree -u`: include usernames of the processes.
3. #### **Thread Related线程相关**
1. ##### **To the Linux kernel, there is no concept of a thread**
- A thread is just a type of process that shares certain resources (like memory) with other processes.
2. ##### **Create a thread**
- Threads are created similarly to processes using the `fork() system call`. However, the `clone() system call` is used with specific flags to indicate which resources are shared.
- **Thread:** clone(CLONE\_VM|CLONE\_FS|CLONE\_FILES|CLONE\_SIGHAND, 0)
  - Shares memory, file system, open files, and signal handlers with the parent.
- **fork():** clone(SIGCHLD, 0)
  - A new process is created without sharing resources.
- **vfork():** clone(CLONE\_VFORK | CLONE\_VM | SIGCHLD, 0)
  - The child shares memory with the parent temporarily, and the parent is blocked until the child calls exec() or exit().
3. ##### **Kernel Threads**
- Which run entirely in kernel space, they do not interact with user space.
- Unlike normal processes, **kernel threads do not have an address space** (their `mm pointer` is NULL).
- All kernel threads are forked from the special kernel process kthreadd, which has PID 2.
  - kthreadd is not created by init (PID 1); it is spawned by the kernel itself during boot. Its parent PID (PPID) is 0.
- `ps -ef`: view kernel threads.
4. #### **Disk I/O Related**
1. ##### **Troubleshoot on process’s disk I/O is very slow**
1) Use `pidstat -d -p PID` to see the I/O delay of the process.
2) Use `top` or `vmstat` to check the wa (I/O wait) value, which shows how much CPU time is spent waiting for I/O.
3) Use `iostat -x` to identify block devices with high wait times.
4) Use `iotop` to identify processes consuming large I/O resources.
5) Use `lsof -p PID` to find which files the process is accessing.
  - If Swap usage is frequent, use vmstat to check Swap activity and consider free memory.
2. ##### **Troubleshoot on NFS(Network File System)Log into NFS server**
1) Log into NFS server.
2) Use `iostat -x` to check which disk device has high wait times.
3) Use `lsof +D ${Dir\_Name}` to see which processes are using the directory.
4) Use `pidstat -d -p PID` to find how long the process waits for I/O.
5) Use `iotop` to identify processes consuming large I/O resources.
6) Use `lsof -p PID` to check which directories or files are accessed by the process.
3. ##### **Troubleshoot on Bad Disk Performance**
1. **Check if the disk is full:**
   1) Use `df -lh` to see disk usage.
   2) Use `du` to find directories or files consuming large amounts of space.
2. **Find which process is accessing the disk:**
   1) Use `iotop` to identify heavy I/O processes.
   1) Use `lsof +D ${DirPath}` to find processes accessing a directory.
3. **Check physical block device health:**
   1) Use `iostat -x` to check I/O stats.
      1. Bad device: Low r/s, w/s, rsec/s, or wsec/s.
      1. Use `fsck` to fix errors on the block device.
   2) Normal device: If high r\_await or w\_await is seen, check memory usage with `vmstat`.
      1. Frequent si/so indicates Swap usage occupying I/O resources.
   3) Optimize I/O: Reduce frequency by using batch processing in code.
4. ##### **Page Cache**
- **minimizes disk I/O by storing data in physical memory for faster access**.
- Mechanism:
  - When a process reads or writes data:
    - Cache hit: If the data is in the cache, it reads/writes from memory instead of the disk.
    - Cache miss: If the data is not in the cache, it reads/writes from the disk and updates the cache.
  - Write-back:
    - Data is written to the page cache first, marked as "dirty," and written back to the disk later.
    - Dirty pages are periodically flushed by flusher threads.
  - Cache eviction:
    - two-list strategy: active list (hot data) and inactive list (cold data), managed using the Least Recently Used (LRU) algorithm.
- Tools: Use `vmstat` to monitor page cache behavior.
5. ##### **Buffer缓冲区 and Cache缓存**
- Cache: The page cache stores file data in memory for faster access.
- Buffer: Buffers store metadata that points to actual file blocks in the page cache.
  - When data is requested, the kernel checks the buffer metadata first to locate the file in the page cache.
- Tools: Use `free -m` to see cache and buffer usage.
6. ##### **RAID(Redundant Array od Independent Disks)**
- Disk virtualization technology that combines multiple physical disks into one virtual storage (RAID array).
- RAID 1: Mirroring data across multiple disks to provide redundancy.
5. #### **File System Related**
1. ##### **Workflow**
- User-space: The `write() system call` is invoked by the user program →
- VFS: translates `write()` into `sys\_write()` and interacts with the filesystem →
- The specific filesystem (e.g., ext4) handles the write operation →
- The data is written to the underlying physical disk.
2. ##### **Key Term**
- Superblock: Contains the metadata of filesystem(size, type, status).
- Bitmap: Tracks with blocks are allocated or free.
- Inode: Contains metadata about a file(permissions, size, timestamps).
- Datablock: Store the actual data of a file.
3. ##### **Virtual Filesystem(VFS)**
- VFS provides an abstraction layer that allows multiple filesystem types to coexist. It defines the following objects:
  - Superblock: Represents a mounted filesystem.
  - Inode: Represents a file or directory.
  - Dentry: Used for directory operations, like path lookup.
  - File: Represents an open file by a process.
- Dentry Object 
  - States:
    - Used: Associated with a valid inode and has users.
    - Unused: Associated with a valid inode but currently not in use.
    - Negative: Not associated with a valid inode (e.g., invalid path).
  - Cache Structure:
    - A list of used dentries, each linked to its associated inode.
    - An LRU list of unused and negative dentries (for cache eviction).
    - Hash table to quickly locate corresponding dentries based on file paths.
- File Object 
  - Create: `call open()`
  - Destroy: `call close()`
- File Lookup Flow: File Object → Dentry Object → Inode Object → Data Block
4. ##### **kernel structures for describing filesystems**
- `file\_system\_type`: Describes the capabilities and behavior of a filesystem.
- `vfsmount`: Stores information about the current mount point and its relationship with other mount points.(当前挂载点和其他挂载点的info）
5. ##### **Block Devices**
- Sector: The smallest addressable unit on a block device.
- Block: The kernel operates on disk in terms of blocks, which are multiples of the sector size(and size need to power of 2, larger than the page size).
- Buffers: Blocks that are read or pending write operations are temporarily stored in memory buffers.
- I/O Scheduler: Manages request queues for block devices to reduce seek times and improve throughput.
  - Merging: Combines adjacent I/O requests into one.
  - Sorting: Reorders requests to minimize disk head movement.
6. ##### **Softlink vs Hardlink**
- Softlink (Symbolic Link):
  - A pointer to another file. If the original file is deleted, the softlink becomes invalid.
  - Implemented by creating a new inode pointing to the original file’s path.
- Hardlink:
  - A direct reference to the original file’s inode. Even if the original file is deleted, the hardlink retains the file’s data.
7. ##### **Disk Partition**
1) Use `lsblk` to list block devices.
2) Use `fdisk -l` to display current partitions.
3) Create a new partition:
   - Run `fdisk {device name}` → Press `n` → Configure (start and end sectors).
4) Format the partition: `mkfs.{filesystem name} {partition name}.`
5) Create a directory.
6) Mount the partition: `mount {partition name} {directory name}.` 挂载分区
6. #### **Kernel Related**
1. ##### **System Call?**
- What is system call?
  - A system call is the only legal way for user-space programs to **interact with the kernel**. It acts as an interface, ensuring **security and stability** by letting the kernel manage access to hardware and resources. 
  - Return value: long type, 0 indicate success, Negative indicate error.
    - can use `perror()` to translate into human readable msg.
- How system call work?
  - The program places the system call number in the **EAX register** and its arguments in other registers.
  - It executes the `int $0x80` instruction, triggering a software interrupt.
  - The processor switches to **kernel mode**, saving user registers and fetching the kernel stack.
  - The kernel executes the system call and verifies parameters.
  - Once finished, it switches back to **user mode** and restores user registers using the `iret` instruction.
- Why can’t we directly call kernel like user function?
  - Kernel code uses a **separate stack and memory segment**. Direct jumps won't work because **they don’t handle privilege level changes.**
2. ##### **Real Mode and Protected Mode**
- Real Mode:
  - Used during system boot, which can only execute 16-bit instructions and access up to 1MB of memory.
- Protected Mode:
  - Allows access to more memory and 32/64-bit instructions. The switch to protected mode happens by loading the **Global Descriptor Table (GDT)** into the **gdtr** register and setting a control bit in the **CR0** register.
3. ##### **Interrupts**
- Interrupts **allow hardware to send signals to the processor, prompting it to stop its current task and execute a specific routine called an interrupt handler.**
- Types:
  - Hardware Interrupt: Triggered by hardware devices (e.g., keyboard).
  - Software Interrupt: Triggered by a program using a special instruction (e.g., `int $0x80` for system calls).
- Interrupt Process
  - Top Half: 
    - Executes immediately after the interrupt occurs.
    - Handles time-sensitive tasks. 
    - Implemented by the interrupt handler in the device driver.
  - Bottom Half:
    - Defers less urgent work to be executed later.
    - Implemented by **softirqs, tasklets, or work queues.**
- Bottom Half Implementation
  - Softirqs软中断
    - Reserved for timing-critical tasks, mostly used by networking and block device subsystems.
    - **Runs in kernel context and cannot sleep.**
    - Softirqs have numerical priorities: lower numbers execute first.
  - Tasklets任务小分支
    - Built on top of softirqs but simpler to use.
    - Only one tasklet of a given type can run at a time.
    - Suitable for tasks that are **not highly threaded.**
  - Work queues
    - Runs in process context, so **it can sleep.**
    - Used for work that **requires sleeping or accessing user-space.**
  - **ksoftirqd** and Softirq Load Handling
    - **When many softirqs are triggered**, the kernel wakes up a kernel thread family called ksoftirqd to process them.
    - Each processor has its own **ksoftirqd/n thread** (e.g., ksoftirqd/0 for CPU 0).
    - These threads run at the lowest priority (nice value = 19).
- Interrupt Handler
  - a kernel function that executes in response to an interrupt.
  - Characteristics:
    - **Runs in interrupt context, so it cannot sleep.**
    - **Shares the kernel stack with the process it interrupts**, so blocking it would also block the interrupted process.
  - Interrupt Stack:
    - Historically, interrupt handlers shared the same stack as the interrupted process.
    - Modern kernels use a dedicated stack for interrupt handlers.
4. ##### **Exception Handling**
- Exceptions occur when the CPU detects an error during instruction execution (e.g., divide by zero or page fault).
- Types:
  - Hardware Interrupts: Asynchronous, triggered by external devices.
  - Exceptions: Synchronous, triggered during execution.
5. ##### **Interrupt Descriptor Table(IDT)**
- The IDT maps interrupts and exceptions to their respective handlers.
- It is accessed through the **idtr** register, which stores the IDT’s base address and length.
- Triggering the IDT: Hardware interrupts, Software interrupts, Exceptions.
6. ##### **Kernel synchronization methods**
1) Atomic Operations原子操作
   - Atomic operations ensure thread-safe manipulation of shared data without locks.
   - Operate on integers (atomic\_t ensures no compiler optimizations).
   - Operate on individual bits.
2) **Spin Lock自旋锁**
   - A lock that causes threads to spin until the lock becomes available.
   - **Non-recursive**: If a thread tries to acquire a lock it already holds, it causes a **deadlock.**
   - **Interrupts must be disabled locally** to prevent interrupt handlers from acquiring the same lock, leading to deadlock.
   - Best for **short critical sections** (less than two context-switch durations).
3) **Semaphores 允许多个线程同时拥有锁**
   - Allows multiple threads to hold the lock.
   - Blocks the thread and places it in a waiting queue if the lock is not available.
   - **Cannot be used in interrupt handlers**, as they can sleep.
4) **Mutex**
   - Similar to semaphores, but **only allows one thread to hold the lock.**
   - Best for **exclusive access** to resources.
- Some Situations
  - Short lock hold time: Spin lock.
  - Long lock hold time: Semaphore/Mutex.
  - Interrupt context: Spin lock.
  - Requires sleep while waiting: Semaphore/Mutex.
7. ##### **Differences Between Kernel and User Processes**
- Memory Space
  - User processes have isolated memory spaces.
  - Kernel threads **share the same memory space without private address spaces**.
- Function
  - Kernel threads perform background tasks.
- Ancestor
  - User processes are forked from `init` (PID 1).
  - Kernel threads are forked from `kthreadd` (PID 2).
8. ##### **Panic and Oops**
- Kernel Oops
  - A recoverable error where the kernel kills the offending process and prints an "oops" message.
  - May leave the kernel in an unstable state.
- Kernel Panic
  - A fatal error where the kernel cannot recover.
  - Dumps kernel memory to disk and requires a manual or automatic reboot.
9. ##### **CPU Cache**
- Levels
  - CPU caches are organized hierarchically: L1, L2, L3, and sometimes L4.
  - Higher levels (e.g., L3, L4) have larger capacity but slower speed.
- Cache Entry Structure
  - Tag: The address of the data in memory.
  - Data Block: The actual cached data.
  - Flag Bit: Indicates if the entry is valid.
- Workflow
  - Cache Hit: If data in the cache, the CPU r/w directly from the cache.
  - Cache Miss: If not, the data is fetched from memory and stored in the cache.
- Cache Strategies
  - Write-through: Writes data to both cache and memory simultaneously.
  - Write-back: Writes data to cache first and updates memory later.
  - LRU: Evicts the least recently used cache entry when space is needed.
7. #### **Networking**
1. ##### **ICMP Protocol互联网控制消息协议**
- ICMP (Internet Control Message Protocol) is typically used for **diagnostic or control purposes**, such as ping or traceroute. It can also to **do DDoS attacks.**
- How it works:
  - **ICMP is connectionless**; it does not rely on TCP or UDP, no need for device.
  - There is no need to establish a connection before sending ICMP messages, and they are not directed to specific ports.
2. ##### **Checking Connectivity**
- Ping
  - Sends ICMP Echo Requests to a host and waits for Echo Replies.
  - Two key metrics:
    - Reachability: Whether the host responds.
    - Latency: Time taken for responses.
  - Example:
    - ping -c 5 -s 1500 [www.google.com](http://www.google.com)
    - Sends 5 ICMP requests with a packet size of 1500 bytes.
- **Telnet: Tests the accessibility of a specific port on a remote server using TCP.**
  - Example: telnet 192.168.1.1 80
- **Traceroute**
  - **Displays the route packets take from the source to the destination.**
  - Measures the transit delays of packets.
  - Example: traceroute www.google.com
- `route -n`: list the routing table on the current server 
3. ##### **Network Card Receiving Packets**
- When a network card receives packets, it generates an interrupt.
- The kernel responds and runs the network card's interrupt handler:
  - Acknowledges the hardware.
  - Copies packets to main memory.
  - Removes packets from the card's buffer.
- If delays occur, the buffer may overflow, causing packet drops.
4. ##### **OSI Seven Layer**
1) Application Layer: High level api like HTTP & Telnet.
2) Presentation Layer: Translates between network service and application.
3) Session Layer: Manages connections between computers.
4) Transport Layer: Method to transfer data from a client to server(TCP & UDP).
5) Network Layer: Determine the routing of packets(e.g., IP).
6) Data Link Layer: Manages data frames between devices.
7) Physical Layer: Transmits raw bitstreams over physical media.
5. ##### **TCP/IP Four Layers**
1) Application Layer: Telnet, FTP, email, HTTP 
2) Transport Layer: TCP, UDP 
3) Network Layer: IP, ICMP 
4) Data Link Layer: device driver 
6. ##### **HandShakes**
- **Connection Establishment(三次握手)**:
1) The client sends a connection request to the server (SYN).
2) The server acknowledges and sends back a response (SYN-ACK).
3) The client sends an acknowledgment to complete the handshake (ACK).
- Data Transmission: Data packets are sent from the client to the server, and acknowledgments are sent back.
- **Connection Termination链接终止(四次挥手):**
1) The client sends a disconnect request (FIN).
2) The server acknowledges the request (ACK) and sends its own disconnect request (FIN).
3) The client acknowledges the server's request (ACK), closing the connection.
7. ##### **Type a URL in a Browser**
- **Step 1: Resolving the URL to an IP Address**
1) First searches for the URL in its *local DNS cache* to see if the IP address is already known.
2) If not in the local cache, the browser sends a *DNS query* to the configured DNS server (by ISP or manually configured).
3) The DNS server forwards the query to a root DNS server, which provides a pointer to the appropriate TLD (Top-Level Domain) server.
   Example: www.example.com, the root server points to the .com TLD server.
4) The TLD server responds with the address of the *authoritative DNS server* for the domain.
5) The authoritative DNS server *provides the IP address of the requested URL.*
6) The *resolved IP address is cached locally* for future use.
- **Step2: Connecting to the IP Address**
1) The browser initiates a **TCP connection** with the server using the resolved IP address. This is done using the three-way handshake:
   - => Client sends SYN req/Server responds with SYN-ACK/Client ack with ACK.
2) The browser generates an HTTP **GET** request to retrieve the content.
- **Step3: Packet Creation and Transmission**
  - HTTP Layer: The HTTP GET request is constructed by the browser.
  - TCP Layer: The TCP layer wraps the HTTP request in a TCP segment by adding a TCP header.
  - IP Layer: The TCP segment is encapsulated into an IP packet with an IP header, which contains the source and destination IP addresses.
  - Routing Table: The IP layer uses the system’s routing table to determine the best path to send the packet to the destination IP address.
  - Data Link Layer: The IP packet is further encapsulated into data frames by the data link layer, which adds a MAC header with source and destination MAC addresses.
  - Physical Transmission: The frames are transmitted as raw bits over the physical medium (e.g., Ethernet, Wi-Fi).
- **Step4: Receiving Response**
1) The server receives the GET request, processes it, and prepares a response (e.g., HTML content).
2) The response follows the reverse path: Encapsulated at the server’s HTTP → TCP → IP → Data Link layer → sent back to the client over the network.
3) The browser receives the response, processes the HTML, CSS, and JavaScript, and renders the webpage for the user.
- **Wrap up**
1) DNS Resolution (Local Cache → DNS Server → Root → TLD → Authoritative Server).
2) TCP Three-Way Handshake.
3) HTTP Request and Response.
4) Packet Encapsulation (HTTP → TCP → IP → Data Link → Physical).
8. ##### **Routing Table**
- `route -n`: Displays the routing table.
- Key Terms:
  - Destination: Target IP or network.
  - Gateway: Next hop or router IP.
  - Genmask: Subnet mask.
  - Flags: Route type (e.g., H: host, G: gateway).
  - Iface: Network interface name.
9. ##### **DNS**
- Resolves domain names to IP addresses.
- Uses UDP for queries.
- The process involves querying local caches, resolver servers, root servers, TLD servers, and authoritative name servers.
10. ##### **TCP vs UDP vs ICMP**
- TCP: Reliable, connection-based protocol for applications.
- UDP: Lightweight, connectionless protocol.
- ICMP: Control protocol, not for data transport.
8. #### **Container Related**
1. ##### **Namespaces**
- Use to isolate processes and enforce resource segregation.
- Key Example: **PID (Process ID) Namespace**
  - Assigns a unique set of PIDs within a namespace.
  - The first process in a namespace has PID 1, independent of the parent namespace.
  - A child process can also have its own PID namespace, where it is assigned PID 1 in that namespace while retaining its PID in the parent namespace.
2. ##### **Control Groups(Cgroups)**
- A Linux kernel feature to limit, account for, and isolate resource usage (CPU, memory, disk I/O, etc.) of a collection of processes.
- Widely used in containerization and Kubernetes environments.
- Key Features:
  - Resource Limits: Restricts memory or CPU usage per process or group of processes.
  - Prioritization: Allocates more or fewer resources to a cgroup compared to others.
  - Accounting: Tracks and reports resource usage.
  - Control: Allows pausing, stopping, or restarting all processes in a cgroup.
3. ##### **Virtualization**
- Structure: Guest Application → Guest OS → Hypervisor/VMM → Host OS → Hardware Resources.
- CPU Virtualization:
  - Trap: Instructions requiring kernel mode are intercepted by the VMM.
  - Emulate: Instructions are executed as functions in the VMM memory space.
- Memory Virtualization:
  - Shadow Page Table: Combines the guest and host page tables into one, mapping guest virtual addresses (GVA) to host physical addresses (HPA). **影子页表（Shadow Page Table）**：将客户和主机页表合并，将客户虚拟地址（GVA）映射到主机物理地址（HPA)
  - Modifications by the guest OS to its page table are intercepted by the VMM and reflected in the shadow page table. 客户端操作系统对页表的修改会被VMM截获，并反映到影子页表中。
  - Separate shadow page tables are used to distinguish user and kernel space.使用独立的影子页表来区分用户空间和内核空间。
4. ##### **Container Network Interface(CNI)**
- A minimal standard for connecting containers to a network.
- Cilium and Flannel are common CNI implementations in Kubernetes.
9. #### **Troubleshooting**
1. ##### **A Batched Task Failed on Some Machine (批量任务在某些机器上失败)**
- Disk of the Remote Machine is Full (远程机器磁盘已满):
  - Use `df -lh` to check disk usage on the remote machine.
- Disk Failure (磁盘故障):
  - Use `fsck` to check and repair the Linux file system.
- Insufficient Memory Resources (内存资源不足):
  - Use `dmesg | egrep -i 'killed process'` to check for OOM (Out of Memory) killer logs.
  - Use `free`, `vmstat`, or `top` to check current memory usage.
- Missing Dependencies or Files (缺失依赖或文件):
  - Run the script on the remote machine to see the output for missing dependencies.
- Resource Accessibility Issue (资源无法访问):
  - Use `ping` or `telnet` to test network connectivity to the required resource from the remote server.
2. ##### **Slow Database Operation (数据库操作缓慢)**
- CPU Saturation (CPU 使用率过高):
  - Use `top` to check CPU usage for this process or others.
- Insufficient Memory for Index (内存不足以加载索引):
  - Use `vmstat` or `free` to check total and available memory.
- Slow or Failing Block Device (块设备速度慢或故障):
  - Use `iostat` or `iotop` to gather I/O information.
  - Use `vmstat` to check swap space usage.
- Database Internal Operations (数据库内部操作占用资源):
  - Check database logs for activities like:
    - Migration: Data chunks being moved to other parts.
    - Synchronization: Data distribution in a distributed database.
3. ##### **Abnormally Low CPU Idle Time but Low Scheduler Saturation (CPU 空闲时间低但调度器饱和度低)**
- Too Many Interrupts (过多的中断):
  - Use `vmstat` to check system.in for interrupt counts.
  - Use `top` or `pidstat -p PID` to analyze processes causing high interrupt rates.
  - Use `strace` to trace the system calls made by a specific process.
4. ##### **Fork Bomb (子进程无限生成问题)**
- Set a Process Limit (设置最大进程数限制):
  - Modify `/etc/security/limits.conf` or use `ulimit -u ${num}` to limit the max processes a single user can own.
- Kill Processes by Session (按会话杀死进程):
  - Get the PID of the parent process: `pstree -sp ${PID}`
  - Print the session ID of the process: `ps --pid ${PID} -o sid`
  - Kill all processes in the session: `kill $(ps -ax --sid <session\_id> -o pid=)`
10. #### **Questions**
1. ##### **How Interrupt Works** 
1) A system call triggers an **interrupt** (e.g., $0x80 for x86 architecture).
2) The interrupt switches the CPU from user mode to **kernel mode**.
3) The **Interrupt Descriptor Table (IDT)** maps the interrupt to a specific function in the kernel.
4) The kernel verifies parameters, executes the system call, and restores the process state.
5) Finally, the **CPU** switches back to user mode using the `iret` instruction.
2. ##### **What Happens When Typing ls -l \*.c in Bash (执行命令的过程)**
1) The keyboard input is parsed by the shell into tokens (ls, -l, \*.c).
2) Shell expands wildcards (e.g., \*.c) into actual file names.
3) Shell checks if the command is a `built-in` or an executable.
4) For external commands, the shell forks a new process to execute it via `execve()`.
5) The ls command accesses file information using `inode` and writes results to STDOUT.
3. ##### **What Happens When Pressing Power On (启动 Linux 系统的流程)**
1) BIOS performs POST and loads the MBR.
2) MBR loads the boot loader (e.g., GRUB).
3) The kernel initializes hardware, mounts the root file system, and executes /sbin/init.
4) `Init` starts system services based on the current runlevel.
4. ##### **Machine Crash Due to Lack of Memory** 
1) Check OOM Killer logs with `dmesg`.
2) Disable OOM panic by modifying `/proc/sys/vm/panic\_on\_oom`.
3) Stop unnecessary processes using `kill -s SIGSTOP`.
4) Check for memory leaks with `valgrind` or monitor with `top`.
5) Use scripts to log memory usage for future analysis.
5. ##### **High I/O or I/O Bottleneck** 
1) Check CPU time spent waiting for I/O using `top` or `vmstat`.
2) Identify heavily used disks with `iostat`.
3) Use `iotop` to find processes consuming I/O resources.
4) Use `lsof -p PID` to check frequently accessed files.
6. ##### **Explain vmstat Output (解释 vmstat 输出内容)**
- `proc.r`: Processes waiting for CPU. 等待 CPU 的进程数
- `proc.b`: Processes in uninterruptible sleep. 不可中断的睡眠进程数
- memory.swpd: Swap space used. 已使用的交换空间
- `cpu.wa`: CPU time waiting for I/O. CPU 等待 I/O 的时间
7. ##### **How Does GDB Work? (GDB 的工作原理)**
- `Ptrace Syscall`: GDB attaches to a process using ptrace, enabling control over its execution.
- `Breakpoints`: GDB replaces instructions with INT3 to trigger a trap.
8. ##### **ELF Format (ELF 文件格式)**
- Sections: Used at link time (e.g., symbol tables).
- Segments: Used at runtime (e.g., code and data segments).
9. ##### **What Happens When Pressing Ctrl+C?** 
1) Triggers a hardware interrupt.
2) Kernel translates the signal to SIGINT.
3) SIGINT is stored in the process PCB and handled by the kernel.
10. ##### **Context Switch (上下文切换过程)**
1) Save the current process state in its PCB.
2) Load the new process state from its PCB.
11. ##### **Syscall and Context Switch Relationship (系统调用与上下文切换的关系)**
-  Not all syscalls cause a context switch unless waiting for unavailable resources.
12. ##### **Exception Handling Process (异常处理过程)**
1) Save register values.
2) Execute the exception handler in kernel mode.
3) Return to user mode.
13. ##### **ls Stuck (ls 卡住的解决办法)**
1) Use `strace ls` to debug stuck system calls.
2) Check mounted directories with `mount -l`.
14. ##### **Sending Signals to a Process (向进程发送信号)**
1) The signal is stored in the process PCB.
2) The kernel invokes the corresponding signal handler.

### **《面经》**
1. #### **vmstat能看出什么？**
- 1. **CPU performance**
1) 高 cpu.us：用户态进程消耗大量 CPU，可能是计算密集型任务。
2) 高 cpu.sy：内核态消耗过多，可能是 I/O 操作或系统调用过多。
3) 低 cpu.id：CPU 空闲时间几乎为零，说明 CPU 饱和。
4) **高 cpu.wa：CPU 大量等待 I/O，可能是磁盘或网络 I/O 瓶颈。**
5) 高 cpu.st：虚拟化环境下，CPU 被其他虚拟机抢占。
- 2. **Memory used**
1) 高 swpd：系统正在频繁使用交换空间，可能是内存不足。
2) 低 free：空闲内存不足，说明物理内存已耗尽。
3) 高 buff, cache：缓冲或缓存太多，正常现象，但需要确保应用进程有足够的可用内存。
4) **高 memory.inact：大量内存处于非活动状态，可能是内存泄漏。**
- 3. **I/O bottleneck**
1) 高 io.bi 或 io.bo：磁盘 I/O 密集操作（如文件读取或写入）。
2) **高 cpu.wa：CPU 等待磁盘 I/O 过多，可能是磁盘性能问题或 I/O 操作过多。**
- 4. **Interrupt or context swap**
1) 高 system.in：系统中断频繁，可能是硬件中断风暴（如网络或磁盘中断）。
2) 高 system.cs：上下文切换频繁，可能是进程数量过多或短生命周期任务（如频繁创建和销毁线程）。
- 5. **Process related** 
1) 高 `proc.r`：运行队列长度大，说明 CPU 忙碌。
2) 高 `proc.b`：大量进程处于不可中断睡眠状态（D 状态），通常是 I/O 资源瓶颈。
- 6. **Swap**
1) 高 swap.si 和 swap.so：系统频繁交换内存，可能是物理内存不足或应用程序内存使用不当。
2) swap 使用频繁时，会导致性能下降，因磁盘 I/O 比内存操作慢得多。
- 7. **System I/O**
1) 高 io.bi：从块设备读取数据频繁，可能是文件读取过多。
2) 高 io.bo：写入块设备数据频繁，可能是日志记录或磁盘缓存写回过多。
2. #### **数据库query变慢？**
**Scenario**: A customer reports that their query execution time has increased from seconds to over a minute. The task is to identify the root cause and provide solutions.
1) **Clarify the problem**
- Gather more details from the customer:
  - What query is affected?
  - Is it a specific query or all queries?
  - Did they make any changes to their database schema, application, or query recently?
  - When did the issue start?
2) **Check Query Execution**
- Verify query performance:
  - Use the `EXPLAIN` or `EXPLAIN ANALYZE` command to analyze the query execution plan.
  - Identify if there are inefficient operations (e.g., full table scans, missing indexes).
- Simulate the query:
  - Run the query in the database and measure its execution time.
  - Check if the issue is reproducible.
3) **Investigate Database Resource Usage**
- CPU Usage:
  - Use tools like **top or specific monitoring tools** to check if the database server is **CPU-bound**.
- Memory Usage:
  - Check if the query is consuming excessive memory.
  - Use `vmstat` or specific commands to monitor **memory pressure and swapping**.
- I/O Bottleneck:
  - Use `iostat` to monitor disk I/O.
  - Check if disk utilization is high or if there is a high I/O wait time.
- Network Issues:
  - If the database is distributed, check for network latency between nodes.
  - Use tools like **ping or traceroute to analyze network connectivity**.
4) **Database-Specific Check**
- Lock Contention锁争用:
  - Use SHOW PROCESSLIST (MySQL) or pg\_stat\_activity (PostgreSQL) to check for locked queries or contention.
- Index Usage:
  - Check if the query is using the appropriate index.
  - If indexes are missing, create them based on query patterns.
- Statistics and Optimizer:
  - Verify if table statistics are outdated and update them using ANALYZE or VACUUM.
- Query Caching:
  - Determine if query caching is enabled and functioning as expected.
5) **Solution based on Findings**
- Index Optimization: Create or adjust indexes to optimize the query execution plan.
- Query Optimization: Rewrite the query to reduce complexity (e.g., avoid SELECT \*, use joins effectively).
- Resource Scaling: Increase server resources like CPU, memory, or IOPS if the workload has grown.
- Partitioning: Partition large tables to improve query performance for specific ranges.
- Caching: Use caching mechanisms to reduce the load on the database for frequently accessed data.
3. #### **网站部分挂掉，排查后端问题？**
1) **Clarify the Problem Scope** 
- What is the behavior of the failure?
- Is the entire website down, or are some parts working (e.g., homepage loads, checkout fails)?
- Is this affecting all users or a subset (e.g., specific regions or devices)?
- When did this start, and were there any recent deployments or configuration changes?
2) **Start with High-Level Checks** 
- Frontend Metrics: Check CDN performance and response time for static assets.
- Traffic Metrics: Is traffic normal? Are there sudden drops or spikes?
- Infrastructure Metrics: Check CPU, memory, disk, and network usage on servers.
- Logs and Alerts: Are there recent error logs or alert notifications?
3) **Narrow Down to Backend Service (聚焦到后端服务)**
1. **Service Health Check**:
   1) Use tools like `curl` or `telnet` to check if backend services are reachable.
   2) Verify if the backend services are responding with expected status codes (e.g., 200, 500).
2. **Service Dependencies**:
   1) Check database connectivity and query performance.
   2) Verify external API dependencies (e.g., payment gateways).
3. **Process Health**:
   1) Use ps aux or systemctl status to see if backend processes are running.
4. **Load Balancer**:
   1) Check if the load balancer is routing traffic correctly.
4) **Investigate Backend Service Issues** 
1. **Configuration Errors**:
   1) Check for recent changes in backend configuration (e.g., environment variables).
   2) Use version control tools to compare configurations with the last working state.
2. **Resource Exhaustion**:
   1) Use `top` or `htop` to check for CPU or memory bottlenecks.
   2) Restart services if they are consuming excessive resources.
3. **Database Issues**:
   1) Use `EXPLAIN` to analyze slow queries.
   2) Check database logs for connection or transaction errors.
4. **Application Bugs**:
   1) Look for unhandled exceptions or memory leaks in application logs.
   2) Roll back to a previous version if the issue is caused by recent code changes.
5. **Dependency Failures**:
   1) Verify if external services (e.g., payment providers) are functioning.
   1) Add retry or fallback mechanisms for critical dependencies.
5) **Restore Service and Prevent Future Issues**
1. **Temporary Fix**:
   1) Restart affected services to temporarily restore functionality.
   2) Reroute traffic to healthy instances using a load balancer.
2. **Long-Term Fix**:
   1) Monitor service metrics more closely (e.g., response time, error rate).
   2) Implement better alerting to detect issues earlier.
   3) Conduct root cause analysis and fix underlying problems.
4. #### **Debug a Linux hosted website？**
- 一个linux电脑在家里run的server来host一个website。半夜突然网页下线了，怎么debug？
- Application Layer: Make sure the website is running without error
  - Check if the web server is running: `systemctl status nginx`  # or apache2
  - Inspect the web server logs for errors: `tail -n 50 /var/log/nginx/error.log`  # path is demo
  - Test locally if the web server is serving requests: `curl http://localhost`
  - Check for recent deployments or configuration changes: Roll back if necessary.
- Database Layer: Make sure it can be used as normal
  - Check if the database service is running: `systemctl status mysql`  # or postgresql
  - Test database connectivity: `mysql -u root -p -e "SHOW DATABASES;"`  # For MySQL
    -  `psql -U postgres -c "\l"`               # For PostgreSQL
  - Inspect database logs for errors: `tail -n 50 /var/log/mysql/error.log`  # Adjust path as needed
  - Check for slow queries or locks: Enable slow query logs for further analysis.
- System Layer
  - Check CPU and memory usage: `top`
  - Look for high I/O wait or disk issues: `iostat -x`
  - Verify disk space: `df -h`
  - Check for memory leaks or swap usage: `vmstat`  `free -h`
  - Inspect system logs: `journalctl -xe`  `dmesg | tail -n 50` 
- Network Layer
  - Test network connectivity: `ping -c 4 yourwebsite.com`  `traceroute yourwebsite.com`
  - Check for network errors or dropped packets: `netstat -s`
  - Inspect active connections: `ss -tuln`
  - Look for potential `DDoS attacks`:
    - Identify IPs with excessive requests: `sudo netstat -an | grep :80 | sort | uniq -c | sort -nr | head`
    - Block malicious IPs: `iptables -A INPUT -s <IP> -j DROP`

- **Improvement and Prevent**
- Monitoring and Alerts 
  - Use tools like Prometheus, Grafana, or Nagios to monitor resource usage, logs, and service 
  - Set up alerts for critical thresholds.
- **Prevent DDoS Attacks** 
  - Use a CDN (e.g., Cloudflare) to absorb traffic spikes.
  - Set up rate-limiting on the web server (e.g., limit\_req in NGINX).
- **Reduce File Storage Pressure** 
  - Use log rotation tools like logrotate.
  - Archive old files to cloud storage or a separate disk.
- **Optimize for High Availability** 
  - Implement load balancing to distribute traffic across multiple servers.
  - Use redundant systems to handle hardware failures.
5. #### **Help maintain website only with admin and pwd？**
1) **Gather Basic Information:** 
   1) Identify the Website Stack and where are the logs stored? 
   2) Check Running Services: Use systemctl or ps to see which services are running.
   3) Confirm Website Functionality:
2) **Set Up Monitoring and Alerts** 
   1) Monitoring Tools: configure monitoring tools like Prometheus + Grafana, Nagios, or Cloudwatch.
   2) Set Alerts: CPU usage, Memory usage, Disk space, Network traffic
   3) Log Monitoring: Use tools like logwatch or set up a simple script to watch /var/log.
3) **Establish Baselines (建立系统基线)**
   1) Collect Key Metrics:
      1. CPU, memory, and disk usage (`top`, `df -h`, `free`).
      1. Network traffic (`iftop` or `nload`).
   2) Test Application Performance: Load the website and check response times.
   3) Document Current State: Record the system's normal behavior as a reference.
4) Plan for Potential Issues (预防潜在问题)
   1) Disk Space: Check for large log files and set up log rotation (logrotate).
   2) Backups: Ensure database and application backups are running.
   3) High Traffic: Enable rate limiting in NGINX/Apache to handle spikes.
   4) DDoS Protection: Use a CDN like Cloudflare to mitigate attacks.
5) Respond to Alerts and Issues (应对警报与问题)
   1) Investigate alerts as they occur.
   2) Prioritize by impact and severity.
6. #### **帮uncle maintain website？**
- 前情提要：用了ping/traceroute，查了CPU memory都没啥问题。 iostat: high awaits. run some read query:正常。 然后跑了df -h, 发现disk full了, log file把空间占满了,怎么fix？ short term: delete old files. Long term: monitoring, adding disk, periodically deleting, balabala.
- **Initial Diagnosis: Disk Full Due to Log Files** 
  1) Confirm Disk Usage:
     1. Use `df -h` to check disk usage.	
     2. Identify which partition is full.
  2) **Locate Large Files**: 
     1. Use `du` to find large files: du -sh /var/log/\*
  3) **Confirm Log File Growth**:
     1. Inspect log files that are consuming space.
     2. ls -lh /var/log/tail -n 50 /var/log/large-log-file.log
- **Short-Term Fixes**
1) **Free Up Space Immediately** 
   1) Delete old or unnecessary log files: sudo find /var/log -type f -name "\*.log" -mtime +7 -exec rm -f {} \;  (This deletes log files older than 7 days.)
   2) Compress large log files: gzip /var/log/large-log-file.log
   3) Clear temporary files: sudo rm -rf /tmp/\*
   4) Truncate logs if immediate deletion isn’t an option: sudo truncate -s 0 /var/log/large-log-file.log
2) **Restart Services** 
- After freeing up space, restart affected services to ensure proper functionality:
```sudo systemctl restart nginx  # or apache2
   sudo systemctl restart mysql  # or other services
```
- **Long-Term Solutions** 
1. **Log Rotation** 
   1) Configure logrotate to manage log files automatically:
      1. Install logrotate if not already installed.
      2. Create a configuration file for the log directory (e.g., /etc/logrotate.d/myapp).
2. **Add Storage or Migrate Logs** 
   1) **Add additional disk space** or mount a separate partition for logs: sudo mount /dev/sdb1 /var/log
   2) Forward logs to a centralized logging system (e.g., ELK Stack, Splunk): This reduces pressure on the local disk.
3. **Proactive Monitoring**
   1) **Set up monitoring tools** like Prometheus, Grafana, or Nagios to:
      1. Track disk usage and alert on thresholds.
      2. Monitor log file sizes.
4. **Preventive Measures (预防措施)**
   1) Set retention policies to **delete or archive logs periodically**.
   2) Avoid excessive logging; adjust log levels to only capture necessary information.

















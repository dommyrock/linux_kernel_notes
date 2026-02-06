# Linux Kernel Internals: Comprehensive Developer Reference

## 1. Process Management & Scheduling

### What It Does and Why It Matters

The Linux process scheduler decides which runnable process gets CPU time, when, and for how long. It directly affects system responsiveness, throughput, and fairness.

### Key Data Structures

- **`struct task_struct`** -- The central process descriptor (~8 KB). Contains PID, state, memory mappings, file descriptors, signal handlers, scheduling info, credentials, and namespaces. Every thread and process has one.
- **`struct sched_entity`** -- Embedded inside `task_struct`, tracks per-task scheduling state: virtual runtime (`vruntime`), load weight, and the red-black tree node.
- **`struct rq`** -- Per-CPU run queue holding pointers to scheduler-class-specific sub-queues (CFS, RT, deadline).
- **`struct cfs_rq`** -- The CFS/EEVDF run queue with an augmented red-black tree of `sched_entity` nodes.

### Scheduling Classes (Priority Order)

1. **Stop scheduler** (`stop_sched_class`) -- Internal use only. Migration threads and kernel-critical work.
2. **Deadline scheduler** (`dl_sched_class`) / `SCHED_DEADLINE` -- Global Earliest Deadline First. Tasks specify runtime, deadline, and period. Highest-priority user-controllable policy.
3. **Real-time scheduler** (`rt_sched_class`) / `SCHED_FIFO`, `SCHED_RR` -- Fixed-priority preemptive, priorities 1-99. `SCHED_RR` adds time-slicing among equal-priority tasks.
4. **Normal/Fair scheduler** (`fair_sched_class`) / `SCHED_NORMAL`, `SCHED_BATCH`, `SCHED_IDLE` -- For regular workloads. Since kernel 6.6, this is EEVDF.
5. **Idle scheduler** (`idle_sched_class`) -- Runs the per-CPU idle thread when nothing else is runnable.

### CFS to EEVDF Transition (Kernel 6.6+)

- **CFS** used `vruntime` as the sole ordering criterion with ad-hoc heuristics.
- **EEVDF** computes a virtual deadline for each task based on requested time slice and lag. Tasks with lag >= 0 are "eligible," and among eligible tasks, the earliest virtual deadline runs next. Removes most CFS heuristics.

### Process Lifecycle: fork, clone, exec

- **`fork()`** -- Duplicates the parent process via `kernel_clone()`. Uses copy-on-write page table entries.
- **`clone()`** -- Generalized fork. Flags control sharing: `CLONE_VM` (share address space = thread), `CLONE_FILES`, `CLONE_NEWNS`, `CLONE_NEWPID`, etc.
- **`clone3()`** -- Modern extensible version using `struct clone_args`.
- **`exec()`** -- Replaces the current process image. Loads ELF binary, sets up new address space, maps segments.
- **Threads vs. Processes** -- In Linux, threads are processes sharing their address space (`CLONE_VM`). The kernel makes no fundamental distinction.

### Cgroups (Control Groups v2)

- **CPU controller** -- `cpu.weight` (relative weight, 1-10000). Bandwidth throttling: `cpu.max` (quota/period).
- **Memory controller** -- `memory.max`, `memory.high`, `memory.low`.
- **I/O controller** -- `io.weight`, `io.max`.
- **PID controller** -- `pids.max`.

### Namespaces

| Namespace | Clone Flag | Isolates |
|-----------|-----------|----------|
| Mount | `CLONE_NEWNS` | Filesystem mount points |
| PID | `CLONE_NEWPID` | Process ID number space |
| Network | `CLONE_NEWNET` | Network devices, stacks, ports |
| IPC | `CLONE_NEWIPC` | System V IPC, POSIX message queues |
| UTS | `CLONE_NEWUTS` | Hostname, domain name |
| User | `CLONE_NEWUSER` | UIDs/GIDs, capabilities |
| Cgroup | `CLONE_NEWCGROUP` | Cgroup root directory |
| Time | `CLONE_NEWTIME` | `CLOCK_MONOTONIC`, `CLOCK_BOOTTIME` offsets |

### Tuning Parameters

- `/proc/sys/kernel/sched_rr_timeslice_ms` -- Time slice for `SCHED_RR` tasks
- `/sys/fs/cgroup/cpu.weight` -- cgroup v2 CPU weight
- `/sys/fs/cgroup/cpu.max` -- cgroup v2 CPU bandwidth
- `taskset` / `sched_setaffinity()` -- CPU pinning
- `chrt` -- Set scheduling policy and priority

---

## 2. Memory Management

### What It Does and Why It Matters

The mm subsystem provides virtual address spaces, manages physical memory allocation, implements demand paging, handles memory-mapped I/O, and ensures memory isolation. The most complex subsystem in the kernel.

### Virtual Memory and Address Spaces

Each process has its own virtual address space (`struct mm_struct`). On x86-64 with 4-level paging: 256 TiB (48-bit). With 5-level paging (kernel 4.14+): 128 PiB (57-bit). Address space regions described by `struct vm_area_struct` (VMA), organized in a maple tree (since kernel 6.1).

### Page Table Hierarchy (x86-64)

```
4-level: [PGD:9][PUD:9][PMD:9][PTE:9][Offset:12] = 48-bit virtual
5-level: [PGD:9][P4D:9][PUD:9][PMD:9][PTE:9][Offset:12] = 57-bit virtual
```

- **PGD** -- Top level, loaded into CR3 on context switch
- **PMD** -- Can point to 2 MiB huge pages
- **PTE** -- Points to 4 KiB pages. Contains present, R/W, user/supervisor, accessed, dirty, NX bits

**TLB** caches translations. **PCID** tags entries per-process to avoid full flushes. **KPTI** (Meltdown mitigation) maintains separate PGDs for user/kernel mode.

### Physical Memory Allocators

**Buddy Allocator** -- Primary allocator for page frames. Manages blocks in powers of 2 (2^0 to 2^10 pages). Split/coalesce to minimize fragmentation. Key function: `alloc_pages(gfp_mask, order)`.

GFP flags: `GFP_KERNEL` (may sleep), `GFP_ATOMIC` (interrupt context), `GFP_DMA` (ZONE_DMA).

**SLUB Allocator** -- The only slab allocator since kernel 6.8. Efficiently allocates small kernel objects from pages. Per-CPU slabs minimize lock contention. `kmalloc()` uses size-class caches (8, 16, 32, ... 8192 bytes).

### Memory Zones

- **ZONE_DMA** (0-16 MiB) -- Legacy ISA DMA
- **ZONE_DMA32** (0-4 GiB) -- 32-bit DMA devices
- **ZONE_NORMAL** (above DMA32 on 64-bit) -- Standard kernel memory
- **ZONE_MOVABLE** -- Migratable pages (hotplug, compaction)

### Huge Pages

- **hugetlbfs** -- Explicit huge pages (2 MiB or 1 GiB). Pre-allocated via `/proc/sys/vm/nr_hugepages`. Used with `mmap(MAP_HUGETLB)`.
- **Transparent Huge Pages (THP)** -- Automatic 4 KiB -> 2 MiB promotion by `khugepaged`. Can cause latency spikes from compaction.

### Copy-on-Write (CoW)

After `fork()`, parent and child share physical pages with PTEs marked read-only. Writes trigger page faults and copy-on-demand.

### mmap

- **File-backed** -- Pages demand-paged from page cache. `MAP_SHARED` (visible to others) or `MAP_PRIVATE` (CoW).
- **Anonymous** (`MAP_ANONYMOUS`) -- Backed by swap. How `malloc()` allocates large blocks.

### OOM Killer

When memory is exhausted, `oom_badness()` selects a victim based on RSS and `/proc/[pid]/oom_score_adj` (-1000 to +1000). Cgroup-level OOM fires before global.

### Tuning Parameters

- `vm.swappiness` (0-200, default 60) -- Swap aggressiveness
- `vm.overcommit_memory` -- 0 (heuristic), 1 (always), 2 (strict)
- `vm.dirty_ratio` / `vm.dirty_background_ratio` -- Writeback thresholds
- `vm.min_free_kbytes` -- Minimum free memory reserved
- `/proc/[pid]/oom_score_adj` -- Per-process OOM tuning
- `/sys/kernel/mm/transparent_hugepage/enabled` -- THP policy

---

## 3. File Systems & VFS

### What It Does and Why It Matters

The Virtual File System (VFS) provides a uniform interface for all filesystem operations regardless of the underlying filesystem type.

### VFS Core Data Structures

- **`struct super_block`** -- Represents a mounted filesystem instance
- **`struct inode`** -- Represents a file on disk. Contains metadata: size, permissions, timestamps, link count. Cached in the inode cache
- **`struct dentry`** -- Associates a filename with an inode. Cached in the dentry cache (dcache). States: used, unused, negative (caches failed lookups)
- **`struct file`** -- Represents an open file. Contains file position, access mode, pointer to dentry/inode
- **`struct file_operations`** -- Function pointer table: `read`, `write`, `llseek`, `mmap`, `fsync`, etc.

### Path Lookup

Resolved component by component using the dcache. **RCU path walking** allows lockless lookup for the common (cached) case.

### Major Filesystems

- **ext4** -- Default Linux FS. Journaling, extent-based allocation, delayed allocation. Max volume: 1 EiB, max file: 16 TiB
- **XFS** -- High-performance 64-bit FS. Allocation groups for parallelism, B+ tree directories, reflinks. Excellent for large files
- **Btrfs** -- CoW B-tree FS. Snapshots, subvolumes, built-in RAID, compression (zstd), send/receive, deduplication
- **tmpfs** -- RAM-backed. Used for `/tmp`, `/run`, `/dev/shm`. Can be swapped
- **procfs** (`/proc`) -- Kernel/process info pseudo-filesystem
- **sysfs** (`/sys`) -- Device model hierarchy
- **overlayfs** -- Union mount. R/W upper over R/O lower layers. Foundation of container image layering

### Tuning Parameters

- Mount options: `noatime`, `data=writeback`, `discard`, `compress=zstd`
- `/proc/sys/vm/vfs_cache_pressure` -- Dentry/inode cache reclaim aggressiveness

---

## 4. Networking Stack

### Architecture Overview

```
RX: NIC -> DMA -> IRQ -> NAPI poll -> XDP hook -> tc ingress ->
    netfilter PREROUTING -> routing -> netfilter INPUT -> socket -> app

TX: app -> socket -> TCP/UDP -> IP -> netfilter OUTPUT -> routing ->
    netfilter POSTROUTING -> tc egress -> driver -> DMA -> NIC
```

### Key Data Structure: `struct sk_buff`

The fundamental network buffer. Contains pointers to protocol headers, linear data + paged fragments, routing cache entry.

### TCP/IP Implementation

- **Congestion control** -- Pluggable: CUBIC (default), BBR, DCTCP, Reno, Vegas
- **GRO/GSO** -- Coalesce/split packets to reduce per-packet overhead
- **Pacing** -- FQ qdisc + TCP internal pacing

### Netfilter / nftables

Hook points: `PREROUTING`, `INPUT`, `FORWARD`, `OUTPUT`, `POSTROUTING`. **nftables** is the modern replacement for iptables. **Connection tracking** (`nf_conntrack`) for stateful firewalling and NAT.

### eBPF/XDP for Networking

- **XDP** -- eBPF at the NIC driver level, before sk_buff allocation. ~10M pps drop rate per core
- **TC BPF** -- eBPF at traffic control. More features than XDP
- **AF_XDP** -- Fast path with shared UMEM ring buffers between kernel and userspace

### Kernel Bypass

- **DPDK** -- Userspace NIC drivers, poll-mode. Bypasses kernel entirely. Line rate on 100+ Gbps
- **AF_XDP** -- Selective bypass within the kernel framework

### Traffic Control (tc)

Qdiscs: `fq` (pacing), `fq_codel` (bufferbloat), `htb` (bandwidth shaping), `tbf`, `prio`.

### Tuning Parameters

- `net.core.somaxconn` -- Max listen backlog
- `net.core.rmem_max` / `wmem_max` -- Max socket buffer sizes
- `net.ipv4.tcp_rmem` / `tcp_wmem` -- TCP auto-tuning ranges
- `net.ipv4.tcp_congestion_control` -- Default CC algorithm
- `net.core.busy_poll` -- Busy polling timeout
- IRQ affinity: `/proc/irq/NNN/smp_affinity`

---

## 5. Device Drivers & Modules

### Device Types

- **Character devices** (`struct cdev`) -- Byte-stream, no buffering. Terminals, GPUs, serial ports
- **Block devices** (`struct gendisk`) -- Random-access, fixed-size blocks. Requests go through blk-mq
- **Network devices** (`struct net_device`) -- Packet-oriented. No `/dev` entry. NAPI for efficient reception

### Loadable Kernel Modules

ELF relocatable objects (`.ko`). Loaded with `modprobe`. Module signing for Secure Boot.

### DMA

- **Streaming DMA** -- One-shot: `dma_map_single()` / `dma_unmap_single()`
- **Coherent DMA** -- Long-lived shared buffers: `dma_alloc_coherent()`
- **Scatter/Gather** -- `dma_map_sg()` for multi-region transfers

### IOMMU

Translates device DMA addresses to physical. Provides isolation (prevents arbitrary DMA), remapping, and VM passthrough via VFIO.

---

## 6. Inter-Process Communication (IPC)

### Pipes and FIFOs

Unidirectional byte streams. Default 64 KiB circular buffer. Writes < `PIPE_BUF` (4096) are atomic. FIFOs add filesystem paths for unrelated processes.

### Shared Memory

- **POSIX** (`shm_open()` + `mmap()`) -- Fastest IPC, no data copying
- **System V** (`shmget()`, `shmat()`) -- Older interface
- **`memfd_create()`** -- Anonymous RAM-backed file, sealable for security

### Unix Domain Sockets

Full-duplex, reliable. Support `SCM_RIGHTS` (pass FDs), `SCM_CREDENTIALS` (pass PID/UID/GID). Significantly faster than TCP loopback.

### io_uring

Two shared-memory ring buffers (SQ for submissions, CQ for completions). With `IORING_SETUP_SQPOLL`, a kernel thread polls -- zero-syscall I/O. Supports 60+ opcodes beyond I/O: accept, connect, socket, futex, waitid. Linked operations, fixed files/buffers, multishot operations.

### eventfd

Lightweight notification via 64-bit counter. Works with `epoll()`. Used in KVM and event loops.

---

## 7. Security

### Capabilities

41+ discrete capabilities replacing root/non-root binary. Key examples: `CAP_NET_BIND_SERVICE`, `CAP_NET_RAW`, `CAP_SYS_ADMIN`, `CAP_BPF`, `CAP_PERFMON`. Three sets per thread: permitted, effective, inheritable.

### seccomp

- **Strict mode** -- Only `read()`, `write()`, `exit()`, `sigreturn()`
- **Filter mode** -- BPF program inspects each syscall. Returns allow/kill/errno/trace/user_notif
- **User notification** (since 5.0) -- Supervisor handles syscalls for sandboxed processes. Used by rootless containers

### Linux Security Modules (LSMs)

200+ hook points. One exclusive LSM active at a time:
- **SELinux** -- Type enforcement, label-based. Default-deny. Used by RHEL/Fedora/Android
- **AppArmor** -- Path-based. Easier to write. Used by Ubuntu/SUSE
- **BPF-LSM** (since 5.7) -- Dynamic eBPF programs at LSM hooks. Stackable with SELinux/AppArmor

### Other Security Features

- **Landlock** (since 5.13) -- Unprivileged sandboxing. Processes restrict their own filesystem/network access
- **Yama** -- Restricts `ptrace`
- **Lockdown** -- Restricts kernel modification under Secure Boot

---

## 8. System Calls & Kernel/User Space Interface

### Syscall Mechanism (x86-64)

1. Syscall number in `%rax`, args in `%rdi`, `%rsi`, `%rdx`, `%r10`, `%r8`, `%r9`
2. `syscall` instruction: save RIP/RFLAGS, load kernel RIP from `MSR_LSTAR`, switch to ring 0
3. Kernel entry: swap to kernel stack, save registers to `pt_regs`, switch page tables (KPTI), dispatch via `sys_call_table[]`
4. Return via `sysretq`

**Cost**: ~100-300 ns per syscall on modern hardware with mitigations.

### vDSO (Virtual Dynamic Shared Object)

Kernel maps a small shared library into every process. Provides userspace implementations of `clock_gettime()`, `gettimeofday()`, `getcpu()` without ring transitions. ~10-20 ns vs ~200+ ns for real syscall.

### Other Interfaces

- **ioctl** -- General-purpose device control for operations outside read/write
- **Netlink sockets** -- Kernel<->userspace message passing (networking config, audit, udev)
- **sysfs/configfs** -- File-based attribute interface
- **procfs** -- Process info and sysctl parameters

---

## 9. I/O Subsystem

### Block I/O Architecture

```
Filesystem / Application
    -> Page Cache (buffered I/O)
    -> struct bio
    -> blk-mq (Multi-Queue Block Layer)
    -> I/O Scheduler (optional)
    -> struct request (merged bios)
    -> Hardware Dispatch Queues
    -> Device Driver -> Hardware
```

### blk-mq

Per-CPU software queues -> hardware queues. Pre-allocated request tags for lockless allocation.

### I/O Schedulers

- **none** -- Direct dispatch. Best for NVMe with internal schedulers
- **mq-deadline** -- Deadline-based (read: 500ms, write: 5s). Good default for SATA/SAS
- **BFQ** -- Proportional-share, excellent interactive responsiveness. Most CPU-intensive
- **Kyber** -- Lightweight, latency-target-based (read: 2ms, write: 10ms)

### Buffered vs Direct I/O

- **Buffered** (default) -- Through page cache. Enables read-ahead and merging
- **Direct** (`O_DIRECT`) -- Bypasses page cache. Used by databases with own caching

### Async I/O

- **Linux AIO** -- Kernel-level, limited to direct I/O. Being superseded
- **io_uring** -- Supports buffered + direct. Zero-syscall with SQPOLL. >1M IOPS single-core on NVMe

### Tuning

- `/sys/block/sdX/queue/scheduler` -- I/O scheduler
- `/sys/block/sdX/queue/nr_requests` -- Max outstanding requests
- `/sys/block/sdX/queue/read_ahead_kb` -- Read-ahead window

---

## 10. Power Management & CPU Frequency

### C-States (Idle)

- **C0** -- Active
- **C1** -- Halt, ~1 us wake
- **C1E** -- Reduced voltage, ~10 us
- **C3** -- Caches flushed, ~100 us
- **C6** -- Core voltage near zero, ~200+ us

### CPUFreq Governors

- **`performance`** -- Always max frequency
- **`powersave`** -- Always min frequency
- **`schedutil`** (default) -- Driven by scheduler's PELT utilization tracking
- **`ondemand`** -- Periodic sampling, jumps to max above threshold

### intel_pstate / amd-pstate

Hardware-managed P-states (HWP). EPP hints (0=performance, 255=power). Active and passive modes.

### Tickless Kernel

- **`NO_HZ_IDLE`** (default) -- Stops ticks when CPU idle
- **`NO_HZ_FULL`** -- Stops ticks on CPUs running single task. For HFT/real-time

### Tuning

- `/sys/devices/system/cpu/cpu*/cpufreq/scaling_governor`
- `/sys/devices/system/cpu/intel_pstate/no_turbo`
- `/sys/devices/system/cpu/cpu*/cpuidle/state*/disable`
- `/dev/cpu_dma_latency` -- Write 0 to keep CPU fully awake
- Boot: `nohz_full=`, `isolcpus=`, `intel_pstate=disable`

---

## 11. eBPF

### What It Is

Sandboxed programs running inside the kernel without kernel modules. Verified, JIT-compiled to native code.

### Architecture

```
C source -> clang/LLVM -> BPF bytecode
    -> bpf() syscall -> Verifier -> JIT -> Attach to hook
```

### BPF Maps

| Type | Use |
|------|-----|
| `HASH` / `PERCPU_HASH` | Key-value store |
| `ARRAY` / `PERCPU_ARRAY` | Fixed-size O(1) lookup |
| `LRU_HASH` | Hash with eviction |
| `RINGBUF` | Efficient event streaming to userspace |
| `PROG_ARRAY` | Tail-call dispatch |
| `SOCKMAP` / `SOCKHASH` | Socket redirection |

### Program Types and Hooks

| Type | Hook | Use Case |
|------|------|----------|
| `XDP` | NIC driver RX | DDoS mitigation, load balancing |
| `SCHED_CLS` | tc classifier | Container networking |
| `KPROBE` | Any kernel function | Tracing, debugging |
| `TRACEPOINT` | Stable kernel events | Observability |
| `LSM` | LSM hooks | Dynamic security policy |
| `STRUCT_OPS` | Kernel struct_ops | Custom TCP CC in BPF |
| `SOCK_OPS` | TCP events | Connection tuning |

### Production Users

Cilium (K8s CNI), Falco/Tetragon (security), bpftrace (tracing), Katran (Facebook L4 LB).

---

## 12. Kernel Tracing & Debugging

### ftrace

Built-in function tracer via `/sys/kernel/tracing/`:
- Function tracer, function graph tracer
- Event tracing: `echo 1 > events/sched/sched_switch/enable`
- Filtering: `echo 'tcp_*' > set_ftrace_filter`
- Near-zero overhead when disabled (NOP code patching)

### perf

Primary performance tool on `perf_event_open()`:
- `perf stat` -- Hardware PMC statistics (cycles, IPC, cache misses)
- `perf record` / `perf report` -- Sampling profiler, flame graphs
- `perf top` -- Live function profiling
- `perf trace` -- Low-overhead syscall tracing
- `perf c2c` -- False sharing analysis

### kprobes / fprobe

Dynamic instrumentation on any kernel function. kprobes use `int3` breakpoints. fprobe (since 5.18) uses ftrace for lower overhead. **Not a stable ABI** -- use tracepoints for stable instrumentation.

### BPF Tracing

**bpftrace** examples:
```
# Count syscalls by process
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Histogram of read() latency
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }
             tracepoint:syscalls:sys_exit_read /@start[tid]/ {
               @us = hist((nsecs - @start[tid]) / 1000);
               delete(@start[tid]); }'
```

### Crash & Debug Tools

- **kdump** -- Secondary kernel captures memory dump on panic
- **KASAN** -- Runtime memory error detection (~3x memory overhead)
- **lockdep** -- Lock dependency validator, detects potential deadlocks
- **KCSAN** -- Data race detector

### Tuning

- `/proc/sys/kernel/perf_event_paranoid` -- Access control (-1 = no restrictions)
- Install kernel debug symbols for perf/bpftrace stack traces

---

## Cross-Subsystem Interactions

- **Scheduling + Memory** -- Page faults block threads. OOM killer uses scheduler state. Cgroup memory limits interact with CPU limits
- **VFS + Memory** -- Page cache is the intersection. `mmap()` maps process space to page cache. Dirty writeback triggered by memory pressure
- **Networking + eBPF** -- XDP before sk_buff allocation saves mm overhead. Socket BPF redirects without traversing the stack
- **I/O + Scheduling** -- I/O completions wake processes. io_uring SQPOLL affected by CPU affinity/cgroups
- **Security + Namespaces + Cgroups** -- Containers = namespaces + cgroups + seccomp + LSMs
- **Power + Scheduling** -- `schedutil` reads PELT utilization for frequency. `nohz_full` isolates from ticks
- **Tracing + Everything** -- eBPF can observe every subsystem from a unified framework

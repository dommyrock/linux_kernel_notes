# Core Linux Kernel Features & HFT Considerations

## Core Linux Kernel Features

- **Process Management** -- scheduling, forking, threads, cgroups
- **Memory Management** -- virtual memory, paging, slab allocator, huge pages
- **File Systems** -- VFS layer, ext4, XFS, tmpfs, procfs
- **Networking Stack** -- TCP/IP, socket API, netfilter, traffic control
- **Device Drivers** -- hardware abstraction, modules, DMA
- **IPC** -- pipes, shared memory, semaphores, message queues, Unix sockets
- **Security** -- namespaces, capabilities, SELinux, seccomp

---

## What Matters Most for HFT

### Latency-Critical (nanoseconds matter)

#### 1. Kernel Bypass Networking

- **DPDK** / **AF_XDP** / **Solarflare OpenOnload** -- bypass the kernel network stack entirely
- Eliminates context switches and syscall overhead on the hot path
- Raw packet access in userspace

#### 2. CPU Isolation & Pinning

- `isolcpus` -- remove cores from the scheduler so only your process runs there
- `taskset` / `sched_setaffinity` -- pin threads to specific cores
- `nohz_full` -- disable timer ticks on isolated cores (tickless mode)
- Prevents jitter from scheduler preemption

#### 3. NUMA Awareness

- `numactl` / `set_mempolicy` -- ensure memory is allocated on the same NUMA node as the CPU running your thread
- Cross-node memory access adds ~100ns latency

#### 4. Huge Pages

- `hugetlbfs` / Transparent Huge Pages (THP)
- Reduces TLB misses (2MB or 1GB pages vs 4KB)
- Pre-allocate with `hugetlb_shm` to avoid page faults on hot path

#### 5. Real-Time Scheduling

- `SCHED_FIFO` / `SCHED_DEADLINE` -- real-time scheduler policies
- `PREEMPT_RT` patch -- makes the kernel fully preemptible
- Reduces worst-case latency from milliseconds to microseconds

#### 6. IRQ Affinity

- `/proc/irq/<N>/smp_affinity` -- pin NIC interrupts to specific cores
- Keep IRQ processing off your trading cores
- Use `irqbalance` carefully or disable it entirely

### Memory & Allocation

#### 7. Lock-Free Memory

- `mlock` / `mlockall` -- prevent pages from being swapped out
- Pre-fault all memory at startup to avoid page faults
- Avoid `malloc` on the hot path -- use pre-allocated ring buffers

#### 8. Shared Memory

- `shmget` / `mmap` with `MAP_SHARED` -- fastest IPC between processes
- Used for market data distribution between components

### Networking Stack Tuning

#### 9. Socket Options (if not using kernel bypass)

- `SO_BUSY_POLL` -- spin-poll instead of sleeping on socket reads
- `TCP_NODELAY` -- disable Nagle's algorithm
- `SO_PRIORITY` / `SO_BINDTODEVICE` -- traffic steering
- Tune ring buffer sizes: `SO_RCVBUF`, `SO_SNDBUF`

#### 10. Timestamping

- `SO_TIMESTAMPING` -- hardware/software packet timestamps
- `clock_gettime(CLOCK_MONOTONIC_RAW)` -- avoid NTP adjustments
- PTP (Precision Time Protocol) via `ptp4l` for clock sync across machines

### Jitter Elimination

#### 11. Disable Noise Sources

- Disable `transparent_hugepage=madvise` (THP compaction causes stalls)
- Turn off `ksoftirqd` migration, watchdog timers on trading cores
- Disable power management: `intel_pstate=disable`, `idle=poll` (core never sleeps)
- `vm.swappiness=0` -- never swap

#### 12. cgroups v2

- Isolate trading processes from everything else (memory, CPU, I/O)
- Prevent OOM killer from touching trading processes

---

## Typical HFT Stack Summary

```
User space:    [Trading App] -- pinned to isolated core, mlocked, pre-allocated
                    |
Networking:    [DPDK / AF_XDP / OpenOnload] -- kernel bypass
                    |
Hardware:      [Solarflare / Mellanox NIC] -- hardware timestamping
                    |
Kernel:        Mostly avoided on hot path. Used for setup, monitoring, cold path only.
```

The key principle: **the fastest syscall is the one you don't make**. HFT systems use the kernel for bootstrapping and configuration, then bypass it entirely on the critical path.

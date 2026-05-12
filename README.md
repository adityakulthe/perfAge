# Performance Tuning and Best Practices in KVM

## Table of Contents

1. [Introduction](#introduction)
2. [VirtIO Framework](#virtio-framework)
3. [CPU Tuning](#cpu-tuning)
   - [vCPU Allocation](#vcpu-allocation)
   - [CPU Pinning](#cpu-pinning)
   - [CPU Topology](#cpu-topology)
4. [Memory Management](#memory-management)
   - [Memory Allocation](#memory-allocation)
   - [Memory Tuning](#memory-tuning)
   - [Memory Backing](#memory-backing)
   - [Hugepages](#hugepages)
   - [Kernel Same-page Merging (KSM)](#kernel-same-page-merging-ksm)
5. [NUMA Tuning](#numa-tuning)
   - [NUMA Topology](#numa-topology)
   - [NUMA Policies](#numa-policies)
   - [Automatic NUMA Balancing](#automatic-numa-balancing)
6. [Disk and Block I/O Tuning](#disk-and-block-io-tuning)
7. [Network Tuning](#network-tuning)
8. [Time-keeping Best Practices](#time-keeping-best-practices)
9. [Summary and Best Practices](#summary-and-best-practices)

---

## Introduction

KVM (Kernel-based Virtual Machine) is a full virtualization solution for Linux on x86 hardware containing virtualization extensions (Intel VT or AMD-V). Performance tuning in KVM environments is crucial for achieving optimal virtual machine performance and efficient resource utilization.

This guide covers comprehensive performance tuning techniques and best practices for KVM virtualization, including CPU optimization, memory management, NUMA awareness, I/O tuning, and network configuration.

---

## VirtIO Framework

VirtIO is a virtualization standard for network and disk device drivers where the guest's device driver "knows" it is running in a virtual environment and cooperates with the hypervisor. This cooperation enables better performance compared to traditional device emulation.

### Key Benefits

- **Reduced Overhead**: Minimizes the overhead of device emulation
- **Better Performance**: Provides near-native performance for I/O operations
- **Standardization**: Offers a common interface for various hypervisors

### VirtIO Devices

- **virtio-net**: Network device
- **virtio-blk**: Block device
- **virtio-scsi**: SCSI device
- **virtio-balloon**: Memory ballooning device
- **virtio-rng**: Random number generator

### Example Configuration

```xml
<devices>
  <disk type='file' device='disk'>
    <driver name='qemu' type='qcow2' cache='none' io='native'/>
    <source file='/var/lib/libvirt/images/vm.qcow2'/>
    <target dev='vda' bus='virtio'/>
  </disk>
  <interface type='network'>
    <source network='default'/>
    <model type='virtio'/>
  </interface>
</devices>
```

---

## CPU Tuning

### vCPU Allocation

Proper vCPU allocation is critical for VM performance. Over-allocation can lead to CPU contention and degraded performance.

#### Best Practices

1. **Match Physical Cores**: Allocate vCPUs based on physical CPU cores available
2. **Avoid Over-subscription**: Keep vCPU to physical CPU ratio reasonable (typically 1:1 or 2:1)
3. **Consider Workload**: Match vCPU count to application requirements

#### Configuration Example

```xml
<vcpu placement='static'>4</vcpu>
```

#### Using virsh Command

```bash
# Set vCPU count for a domain
virsh setvcpus <domain> <count> --config --maximum

# Set current vCPU count
virsh setvcpus <domain> <count> --config
```

### CPU Pinning

CPU pinning (also called CPU affinity) binds vCPUs to specific physical CPUs, reducing context switching and improving cache locality.

#### Benefits

- Reduced CPU migration overhead
- Better cache utilization
- Improved performance consistency
- Reduced latency for real-time workloads

#### Configuration Example

```xml
<cputune>
  <vcpupin vcpu='0' cpuset='0'/>
  <vcpupin vcpu='1' cpuset='1'/>
  <vcpupin vcpu='2' cpuset='2'/>
  <vcpupin vcpu='3' cpuset='3'/>
  <emulatorpin cpuset='0-1'/>
</cputune>
```

#### Using virsh Command

```bash
# Pin vCPU to physical CPU
virsh vcpupin <domain> <vcpu> <cpuset> --config

# Example: Pin vCPU 0 to physical CPU 2
virsh vcpupin myvm 0 2 --config

# View current pinning
virsh vcpupin <domain>
```

#### Advanced Pinning Strategy

For NUMA systems, pin vCPUs to CPUs within the same NUMA node:

```bash
# Check NUMA topology
numactl --hardware

# Pin vCPUs to NUMA node 0
virsh vcpupin myvm 0 0-7 --config
virsh vcpupin myvm 1 0-7 --config
```

### CPU Topology

Configuring CPU topology helps the guest OS make better scheduling decisions and improves performance for NUMA-aware applications.

#### Configuration Example

```xml
<cpu mode='host-passthrough'>
  <topology sockets='1' cores='4' threads='2'/>
  <numa>
    <cell id='0' cpus='0-3' memory='4194304' unit='KiB'/>
    <cell id='1' cpus='4-7' memory='4194304' unit='KiB'/>
  </numa>
</cpu>
```

#### CPU Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `host-passthrough` | Exposes host CPU features directly | Best performance, limited migration |
| `host-model` | Uses closest CPU model | Good performance, better migration |
| `custom` | Manually specify CPU model | Maximum compatibility |

#### Example with CPU Model

```xml
<cpu mode='custom' match='exact'>
  <model fallback='allow'>Skylake-Server</model>
  <topology sockets='2' cores='4' threads='2'/>
  <feature policy='require' name='vmx'/>
</cpu>
```

---

## Memory Management

### Memory Allocation

Proper memory allocation ensures VMs have sufficient memory while avoiding over-commitment that can lead to swapping.

#### Static Allocation

```xml
<memory unit='GiB'>8</memory>
<currentMemory unit='GiB'>8</currentMemory>
```

#### Dynamic Allocation with Memory Ballooning

```xml
<memory unit='GiB'>16</memory>
<currentMemory unit='GiB'>8</currentMemory>
<memballoon model='virtio'>
  <stats period='10'/>
</memballoon>
```

#### Best Practices

- Allocate memory based on workload requirements
- Leave sufficient memory for the host OS (typically 2-4 GB minimum)
- Monitor memory usage and adjust as needed
- Use memory ballooning for dynamic adjustment

### Memory Tuning

Fine-tune memory parameters for optimal performance.

#### Configuration Example

```xml
<memtune>
  <hard_limit unit='GiB'>10</hard_limit>
  <soft_limit unit='GiB'>8</soft_limit>
  <swap_hard_limit unit='GiB'>12</swap_hard_limit>
  <min_guarantee unit='GiB'>6</min_guarantee>
</memtune>
```

#### Parameters Explained

- **hard_limit**: Maximum memory the VM can use
- **soft_limit**: Memory limit enforced during memory contention
- **swap_hard_limit**: Maximum memory + swap
- **min_guarantee**: Minimum memory guaranteed to VM

### Memory Backing

Configure memory backing for performance optimization.

#### Hugepages Configuration

```xml
<memoryBacking>
  <hugepages>
    <page size='1' unit='GiB'/>
  </hugepages>
  <locked/>
  <nosharepages/>
</memoryBacking>
```

#### Options

- **hugepages**: Use hugepages for VM memory
- **locked**: Lock VM memory in RAM (prevent swapping)
- **nosharepages**: Disable KSM for this VM
- **source type='file'**: Use file-backed memory

### Hugepages

Hugepages reduce TLB (Translation Lookaside Buffer) misses and improve memory performance.

#### Understanding Hugepages

Hugepages were introduced in the Linux kernel to improve the performance of memory management. Memory is managed in blocks known as pages.

**Page Size Basics:**

- **Standard Pages**: x86 CPUs usually address memory in 4 KB pages
- **Hugepages**: CPUs are capable of using larger pages:
  - 2 MB pages (common)
  - 1 GB pages (for very large memory systems)

**Why Hugepages?**

When a system needs to handle huge amounts of memory, there are two options:

1. **Increase page table entries** in hardware MMU (expensive and limited)
   - Modern processors support only hundreds or thousands of page table entries
   - Struggles with high number of entries or manipulations

2. **Increase page size** (hugepages - more efficient)
   - 2 MB pages suitable for managing multiple gigabytes of memory
   - 1 GB pages best for scaling to terabytes of memory

**Translation Lookaside Buffer (TLB):**

A TLB is a cache used for virtual-to-physical address translations. It's a very scarce resource on processors. Operating systems try to make the best use of limited TLB resources. This optimization is more critical now as bigger physical memories (several GB) are more readily available.

**Benefits:**

- Reduced TLB misses
- Lower memory management overhead
- Better performance for memory-intensive workloads
- Improved memory access latency
- More efficient use of CPU cache

#### Enable Hugepages on Host

**1. Check Current Configuration:**

```bash
# Check current hugepage configuration
cat /proc/meminfo | grep Huge

# Example output:
# AnonHugePages:         0 kB
# HugePages_Total:       0
# HugePages_Free:        0
# HugePages_Rsvd:        0
# HugePages_Surp:        0
# Hugepagesize:       2048 kB

# Check hugepage size
cat /proc/meminfo | grep Hugepagesize
```

**2. Configure Hugepages:**

```bash
# View current hugepages value
cat /proc/sys/vm/nr_hugepages

# Or using sysctl
sysctl -a | grep huge

# Configure hugepages (2MB pages)
# Example: Set 1024 hugepages (1024 * 2MB = 2GB)
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# Configure 1GB hugepages
# Example: Set 8 hugepages (8 * 1GB = 8GB)
echo 8 > /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages

# Make persistent (add to /etc/sysctl.conf)
vm.nr_hugepages = 1024

# Apply sysctl changes
sysctl -p
```

**Important Note:** Total memory assigned for hugepages cannot be used by applications that are not hugepage-aware. If you over-allocate hugepages, normal functioning of the host system can be affected.

**3. Mount Hugepages:**

```bash
# Create mount point
mkdir -p /dev/hugepages

# Mount hugepages
mount -t hugetlbfs hugetlbfs /dev/hugepages

# Verify mount
mount | grep huge

# Make persistent (add to /etc/fstab)
hugetlbfs /dev/hugepages hugetlbfs defaults 0 0
```

**4. Restart libvirtd:**

```bash
# Restart libvirtd service
systemctl restart libvirtd
```

**5. Configure VM to Use Hugepages:**

```xml
<memoryBacking>
  <hugepages>
    <page size='1' unit='GiB' nodeset='0'/>
  </hugepages>
</memoryBacking>
```

**6. Verify VM is Using Hugepages:**

```bash
# Restart the hugepage-configured guest
virsh start <vm-name>

# Verify on host
cat /proc/meminfo | grep -i huge

# Check which process is using hugepages
grep -i hugepages /proc/*/smaps | grep -v "0 kB"
```

#### Transparent Hugepages (THP)

THP is an abstraction layer that automates hugepage size allocation based on application requests.

**What is THP?**

Transparent Hugepage support can be:
- Entirely disabled
- Enabled only inside MADV_HUGEPAGE regions (to avoid consuming more memory)
- Enabled system-wide

**THP Modes:**

| Mode | Description | Use Case |
|------|-------------|----------|
| `always` | Always use THP | Maximum performance, may use more memory |
| `madvise` | Use hugepages only in VMAs marked with MADV_HUGEPAGE | Balanced approach, application control |
| `never` | Disable THP | When THP causes issues or not needed |

**Check Current THP Setting:**

```bash
# Check current setting
cat /sys/kernel/mm/transparent_hugepage/enabled

# Example output:
# always [madvise] never
# The value in brackets [] is currently active
```

**Configure THP:**

```bash
# Enable always
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# Enable madvise (recommended)
echo madvise > /sys/kernel/mm/transparent_hugepage/enabled

# Disable THP
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Make persistent (add to /etc/rc.local or systemd service)
```

**THP Benefits:**

- System settings for performance automatically optimized
- Allows all free memory to be used as cache
- Performance increased by better memory utilization
- Can coexist with static hugepages

**THP and KVM:**

- When static hugepages are not used, KVM will use transparent hugepages instead of regular 4 KB page size
- Advantages: Less memory used for page tables, reduced TLB misses, increased performance
- **Important**: When using hugepages for guest memory, you can no longer swap or balloon guest memory

**THP vs Static Hugepages:**

| Feature | Static Hugepages | Transparent Hugepages |
|---------|------------------|----------------------|
| Configuration | Manual | Automatic |
| Memory Reservation | Pre-allocated | Dynamic |
| Swapping | Not supported | Supported (when not using hugepages) |
| Ballooning | Not supported | Supported (when not using hugepages) |
| Best For | Predictable workloads | Dynamic workloads |
| Performance | Slightly better | Very good |

#### Hugepages Best Practices

1. **Calculate Requirements**: Determine total memory needed for all VMs
2. **Reserve Appropriately**: Don't over-allocate; leave memory for host OS
3. **Use 1GB Pages**: For very large memory VMs (>64GB)
4. **Use 2MB Pages**: For most standard VMs
5. **Monitor Usage**: Regularly check hugepage utilization
6. **NUMA Awareness**: Allocate hugepages on correct NUMA nodes
7. **THP for Dynamic**: Use THP when workload is unpredictable
8. **Static for Production**: Use static hugepages for production VMs with known memory requirements

### Kernel Same-page Merging (KSM)

KSM allows the kernel to merge identical memory pages across VMs, reducing memory usage.

#### Enable KSM

```bash
# Enable KSM
echo 1 > /sys/kernel/mm/ksm/run

# Configure scan parameters
echo 100 > /sys/kernel/mm/ksm/sleep_millisecs
echo 1000 > /sys/kernel/mm/ksm/pages_to_scan
```

#### Monitor KSM

```bash
# Check KSM statistics
cat /sys/kernel/mm/ksm/pages_shared
cat /sys/kernel/mm/ksm/pages_sharing
cat /sys/kernel/mm/ksm/pages_unshared
```

#### Considerations

- **Pros**: Reduces memory usage, allows higher VM density
- **Cons**: CPU overhead for scanning, potential security concerns
- **Best for**: Environments with many similar VMs (e.g., VDI)

---

## NUMA Tuning

Non-Uniform Memory Access (NUMA) tuning is critical for performance on multi-socket systems.

### NUMA Topology

Understanding and configuring NUMA topology ensures optimal memory access patterns.

#### Check Host NUMA Topology

```bash
# View NUMA topology
numactl --hardware

# View NUMA statistics
numastat

# Check CPU to NUMA node mapping
lscpu | grep NUMA
```

#### Configure VM NUMA Topology

```xml
<cpu mode='host-passthrough'>
  <numa>
    <cell id='0' cpus='0-3' memory='4' unit='GiB'/>
    <cell id='1' cpus='4-7' memory='4' unit='GiB'/>
  </numa>
</cpu>
```

### NUMA Policies

Configure NUMA memory allocation policies for optimal performance.

#### NUMA Tuning Configuration

```xml
<numatune>
  <memory mode='strict' nodeset='0'/>
  <memnode cellid='0' mode='strict' nodeset='0'/>
  <memnode cellid='1' mode='strict' nodeset='1'/>
</numatune>
```

#### NUMA Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `strict` | Allocate only from specified nodes | Best performance, may fail if memory unavailable |
| `preferred` | Prefer specified nodes, fallback allowed | Good performance with flexibility |
| `interleave` | Distribute across nodes | Balanced memory bandwidth |
| `restrictive` | Restrict to specified nodes | Resource isolation |

#### Example: Strict NUMA Binding

```xml
<numatune>
  <memory mode='strict' nodeset='0-1'/>
</numatune>
<cputune>
  <vcpupin vcpu='0' cpuset='0-7'/>
  <vcpupin vcpu='1' cpuset='8-15'/>
</cputune>
```

### Automatic NUMA Balancing

Configure automatic NUMA balancing for dynamic workloads.

#### What is Automatic NUMA Balancing?

The main aim of automatic NUMA balancing is to improve the performance of different applications running in a NUMA-aware system. The strategy behind its design is simple: an application will generally perform best when the threads of its processes are accessing memory on the same NUMA node where the threads are scheduled by the kernel.

Automatic NUMA balancing moves tasks (threads or processes) closer to the memory they are accessing. It also moves application data to memory closer to the tasks that reference it. This is all done automatically by the kernel when automatic NUMA balancing is active.

#### Conditions for Automatic NUMA Balancing

Automatic NUMA balancing will be enabled when booted on hardware with NUMA properties. The main conditions or criteria are:

- `numactl --hardware`: Shows multiple nodes
- `cat /sys/kernel/debug/sched_features`: Shows NUMA in the flags

Example output:

```bash
[ humble-numaserver ]$ cat /sys/kernel/debug/sched_features
GENTLE_FAIR_SLEEPERS START_DEBIT NO_NEXT_BUDDY LAST_BUDDY CACHE_HOT_BUDDY
WAKEUP_PREEMPTION ARCH_POWER NO_HRTICK NO_DOUBLE_TICK LB_BIAS NONTASK_
POWER TTWU_QUEUE NO_FORCE_SD_OVERLAP RT_RUNTIME_SHARE NO_LB_MIN NUMA
NUMA_FAVOUR_HIGHER NO_NUMA_RESIST_LOWER
```

#### Enable on Host

```bash
# Check if enabled
cat /proc/sys/kernel/numa_balancing

# Enable automatic NUMA balancing
echo 1 > /proc/sys/kernel/numa_balancing

# Disable automatic NUMA balancing
echo 0 > /proc/sys/kernel/numa_balancing

# Make persistent (add to /etc/sysctl.conf)
kernel.numa_balancing = 1
```

#### Configuration Parameters

```bash
# Scan delay (milliseconds)
echo 1000 > /proc/sys/kernel/numa_balancing_scan_delay_ms

# Scan period (milliseconds)
echo 1000 > /proc/sys/kernel/numa_balancing_scan_period_min_ms
echo 60000 > /proc/sys/kernel/numa_balancing_scan_period_max_ms

# Scan size (MB)
echo 256 > /proc/sys/kernel/numa_balancing_scan_size_mb
```

#### How It Works

The Automatic NUMA balancing mechanism works based on several algorithms and data structures:

- **NUMA hinting page faults**: Triggers for memory migration decisions
- **NUMA page migration**: Moves pages closer to accessing CPUs
- **Task grouping**: Groups related tasks together
- **Fault statistics**: Tracks memory access patterns
- **Task placement**: Optimizes task scheduling
- **Pseudo-interleaving**: Balances memory across nodes

#### Important Notes

- Manual NUMA tuning of applications will override automatic NUMA balancing
- Disables the periodic unmapping of memory, NUMA faults, migration, and automatic NUMA placement for manually tuned applications
- In some cases, system-wide manual NUMA tuning is preferred
- **Best Practice**: Limit guest resources to the amount available on a single NUMA node to avoid unnecessarily splitting resources across NUMA nodes

#### Best Practices

1. **Pin VMs to NUMA nodes**: Avoid cross-NUMA memory access
2. **Match vCPU and memory**: Allocate both from same NUMA node
3. **Monitor NUMA statistics**: Check for remote memory access
4. **Size VMs appropriately**: Don't exceed single NUMA node capacity when possible
5. **Consider workload**: Automatic balancing works best for dynamic workloads

### Understanding numad and numastat

#### numad - Automatic NUMA Affinity Management Daemon

numad is a user-level daemon that provides placement advice and process management for efficient use of CPUs and memory on systems with NUMA topology. It monitors NUMA topology and resource usage within a system to dynamically improve NUMA resource allocation and management.

**Key Features:**

- Monitors NUMA topology and resource usage
- Attempts to locate processes for efficient NUMA locality and affinity
- Dynamically adjusts to changing system conditions
- Provides guidance for initial manual binding of CPU and memory resources

**When to Use numad:**

- **Best for**: Server consolidation environments with multiple applications or virtual guests
- **Most effective**: When processes can be localized in a subset of the system's NUMA nodes
- **Not recommended**: For systems dedicated to large in-memory databases with unpredictable memory access patterns

**Starting numad:**

```bash
# Start numad as executable
numad

# Check if running
ps aux | grep numad

# Stop numad
numad -i 0
```

**Important Notes:**

- When numad is enabled, its behavior overrides the default behavior of automatic NUMA balancing
- Stopping numad does not remove the changes it has made to improve NUMA affinity
- If system use changes significantly, running numad again will adjust the affinity

**numad Log File:**

Monitor numad activity in `/var/log/numad`:

```bash
# Example log entries
Tue Nov 17 06:49:43 2015: Changing THP scan time in /sys/kernel/mm/
transparent_hugepage/khugepaged/scan_sleep_millisecs from 10000 to 1000 ms.
Tue Nov 17 06:49:43 2015: Registering numad version 20140225 PID 9170
Tue Nov 17 06:49:45 2015: Advising pid 1479 (qemu-kvm) move from nodes
(0-1) to nodes (1)
Tue Nov 17 06:49:47 2015: Including PID: 1479 in CPUset: /sys/fs/cgroup/
CPUset/machine.slice/machine-qemu\x2dfedora21.scope/emulator
Tue Nov 17 06:49:48 2015: PID 1479 moved to node(s) 1 in 3.33 seconds
```

#### numastat - NUMA Memory Statistics

numastat shows per-NUMA-node memory statistics for processes and the operating system.

**Usage:**

```bash
# Show NUMA statistics for all processes
numastat

# Show statistics for specific process (e.g., qemu-kvm)
numastat -p $(pgrep qemu-kvm)

# Show statistics with process name
numastat qemu-kvm
```

**Package Information:**

- The `numactl` package provides the `numactl` binary/command
- The `numad` package provides the `numad` binary/command

**Monitoring NUMA Performance:**

Use numastat to monitor the difference before and after running the numad service to verify performance improvements.

#### KSM and NUMA

KSM is capable of detecting that a system is using NUMA memory and controlling merging pages across different NUMA nodes.

```bash
# Check merge_across_nodes setting
cat /sys/kernel/mm/ksm/merge_across_nodes

# By default, pages from all nodes can be merged (value = 1)
# Set to 0 to merge only pages from the same node
echo 0 > /sys/kernel/mm/ksm/merge_across_nodes
```

**Important Considerations:**

- When KSM merges across nodes on a NUMA host with multiple guest virtual machines, guests and CPUs from more distant nodes can suffer a significant increase in access latency to the merged KSM page
- Unless you are oversubscribing or overcommitting system memory, you will get better runtime performance by disabling KSM sharing
- Use the `nosharepages` option in guest XML to disable KSM for specific guests

#### Migration Considerations

**Important Warning:**

It is harder to live-migrate a pinned guest across hosts because a similar set of backing resources/configurations may not be available on the destination or target host where the VM is getting migrated. For example, the target host may have a different NUMA topology.

You should consider this fact when you tune a KVM environment. Automatic NUMA balancing may help, to a certain extent, reduce the need for manually pinning guest resources.

---

## Disk and Block I/O Tuning

Optimizing disk I/O is crucial for overall VM performance.

### Understanding Virtual Disk Backends

The virtual disk of a VM can be either a block device or an image file.

**Key Observations:**

- **Block Device Backend**: Preferred for better VM performance over image files on remote filesystems (NFS, GlusterFS, etc.)
- **File Backend**: Helps virt admins better manage guest disks and is useful in many scenarios
- **Mixed Usage**: No restriction on mixing block devices and files as storage disks for the same guest
- **Disk Limit**: Total number of virtual disks that can be attached to a VM has a limit

#### I/O Path Understanding

When an application inside a guest OS writes data to local storage, the I/O request traverses through multiple layers:

1. **Guest Filesystem**: Application writes to guest filesystem
2. **Guest I/O Subsystem**: Request passes through guest OS I/O subsystem
3. **qemu-kvm Process**: Hypervisor receives request from guest OS
4. **Host Processing**: Hypervisor processes I/O like any other host application

This multi-layer traversal explains why block device backends perform better than image file backends.

#### File Image Considerations

- **Additional Resource Demand**: File image is part of host filesystem, creating additional I/O overhead compared to block devices
- **Sparse Images**: Using sparse image files helps over-allocate host storage but reduces virtual disk performance
- **Partition Alignment**: Improper partitioning of guest storage with disk image files can cause unnecessary I/O operations due to misalignment of standard partition units

### Storage Backend Selection

Choose the appropriate storage backend for your workload.

#### Storage Options

| Backend | Description | Use Case | Performance |
|---------|-------------|----------|-------------|
| `qcow2` | QEMU Copy-On-Write format | Snapshots, thin provisioning | Good (with overhead) |
| `raw` | Raw disk image | Best performance | Excellent |
| `LVM` | Logical Volume Manager | Good performance, flexibility | Very Good |
| `Ceph/RBD` | Distributed storage | Scalability, high availability | Good (network dependent) |
| `Block Device` | Direct block device | Production workloads | Best |

**Recommendation**: Use raw format for best performance. The qcow format has performance overhead due to the format layer operations (e.g., allocating new clusters when growing images). However, qcow is useful when features like snapshots are required.

### VirtIO Disk Bus

**Always use the virtio disk bus** when configuring disks rather than the IDE bus. The virtio_blk driver uses the VirtIO API to provide high performance for storage I/O devices, significantly increasing storage performance, especially in large enterprise storage systems.

### VirtIO-SCSI vs VirtIO-BLK

Both are paravirtualized storage controllers, but they have different characteristics:

#### VirtIO-BLK Configuration

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' io='native'/>
  <source file='/var/lib/libvirt/images/vm.qcow2'/>
  <target dev='vda' bus='virtio'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
</disk>
```

#### VirtIO-SCSI Configuration

```xml
<controller type='scsi' index='0' model='virtio-scsi'>
  <driver queues='4'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
</controller>
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' io='native'/>
  <source file='/var/lib/libvirt/images/vm.qcow2'/>
  <target dev='sda' bus='scsi'/>
  <address type='drive' controller='0' bus='0' target='0' unit='0'/>
</disk>
```

### Cache Modes

Configure appropriate cache mode for your workload.

#### Understanding Cache Modes (Visual Representation)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Disk Cache Modes                              │
├─────────────┬─────────────┬─────────────┬─────────────────────────┤
│  writeback  │writethrough │    none     │     directsync          │
├─────────────┼─────────────┼─────────────┼─────────────────────────┤
│             │             │             │                         │
│   Guest     │   Guest     │   Guest     │      Guest              │
│     │       │     │       │     │       │        │                │
│     ▼       │     ▼       │     ▼       │        ▼                │
│   R   W     │   R   W     │   R   W     │      R   W              │
│   │   │     │   │   │     │   │   │     │      │   │              │
│   ▼   ▼     │   ▼   │     │   │   │     │      │   │              │
│  Host Page  │  Host Page  │   │   │     │      │   │              │
│   Cache     │   Cache     │   │   │     │      │   │              │
│   │   │     │   │   │     │   │   │     │      │   │              │
│   ▼   │     │   ▼   ▼     │   ▼   ▼     │      ▼   ▼              │
│  Disk Cache │  Disk Cache │  Disk Cache │   Physical Disk         │
│   │   │     │   │   │     │   │   │     │      │   │              │
│   ▼   ▼     │   ▼   ▼     │   ▼   ▼     │      ▼   ▼              │
│ Physical    │ Physical    │ Physical    │                         │
│   Disk      │   Disk      │   Disk      │                         │
└─────────────┴─────────────┴─────────────┴─────────────────────────┘

Legend:
R = Read operations
W = Write operations
│ = Data flow path
▼ = Direction of data flow
```

#### Cache Mode Options

| Mode | Host Page Cache | Disk Write Cache | O_DIRECT | O_SYNC | Use Case |
|------|----------------|------------------|----------|--------|----------|
| `none` | Bypassed | Used | Yes | No | **Best for production**, data integrity, supports migration |
| `writethrough` | Used for reads | Bypassed | No | No | Balanced performance and safety |
| `writeback` | Used | Used | No | No | Best performance, risk of data loss |
| `directsync` | Bypassed | Bypassed | Yes | Yes | Maximum data integrity |
| `unsafe` | Used | Ignored | No | No | **Testing only**, high data loss risk |
| `default` | System default | System default | - | - | Uses system's default settings |

#### Detailed Mode Descriptions

**cache=none** (Recommended for Production)
- Uses O_DIRECT flag to bypass host page cache
- I/O happens directly between qemu-kvm userspace buffers and storage device
- Guest I/O not cached on host, but may be kept in writeback disk cache
- Best choice for guests with large I/O requirements
- **Only option that supports migration**
- Semantics: Host page cache bypassed, disk write cache used

**cache=writethrough**
- Matches O_DSYNC semantics
- Guest I/O cached on host but written through to physical medium
- Writes reported as completed only when data committed to storage device
- Slower and prone to scaling problems
- Suitable for small number of guests with lower I/O requirements
- Recommended for guests that don't support writeback cache (when migration not needed)

**cache=writeback**
- Guest I/O cached on host
- Writes reported as completed when placed in host page cache
- Normal page cache management handles commitment to storage
- Host page cache used, writes reported to guest as completed when in cache
- Best performance but risk of data loss on host failure

**cache=directsync**
- Similar to writethrough but bypasses host page cache
- I/O from guest bypasses host page cache
- Writes reported as completed only when committed to storage device
- Use when writethrough behavior desired but also want to bypass host page cache

**cache=unsafe**
- Host may cache all disk I/O
- Sync requests from guests are ignored
- **Huge risk of data loss** in event of host failure
- May be useful for guest installation or similar non-critical tasks
- **Never use in production**

#### Example Configuration

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='raw' cache='none' io='native' discard='unmap'/>
  <source file='/var/lib/libvirt/images/vm.img'/>
  <target dev='vda' bus='virtio'/>
</disk>
```

#### Best Practice Recommendations

1. **Production Systems**: Use `cache='none'` for best balance of performance and data integrity
2. **Development/Testing**: `cache='writeback'` for maximum performance (accept the risk)
3. **Small Deployments**: `cache='writethrough'` for safety with moderate performance
4. **Maximum Safety**: `cache='directsync'` when data integrity is paramount
5. **Never in Production**: `cache='unsafe'` - only for temporary, non-critical operations

### I/O Tuning Parameters

Fine-tune I/O performance with iotune parameters.

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2'/>
  <source file='/var/lib/libvirt/images/vm.qcow2'/>
  <target dev='vda' bus='virtio'/>
  <iotune>
    <total_bytes_sec>104857600</total_bytes_sec>
    <read_bytes_sec>52428800</read_bytes_sec>
    <write_bytes_sec>52428800</write_bytes_sec>
    <total_iops_sec>1000</total_iops_sec>
    <read_iops_sec>500</read_iops_sec>
    <write_iops_sec>500</write_iops_sec>
  </iotune>
</disk>
```

### Multi-queue Support

Enable multi-queue for better I/O performance on multi-core systems.

```xml
<controller type='scsi' index='0' model='virtio-scsi'>
  <driver queues='4' iothread='1'/>
</controller>
```

### I/O Threads

Use I/O threads to offload I/O processing from vCPU threads.

```xml
<iothreads>4</iothreads>
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' iothread='1'/>
  <source file='/var/lib/libvirt/images/vm.qcow2'/>
  <target dev='vda' bus='virtio'/>
</disk>
```

### Best Practices

1. **Use VirtIO drivers**: Always use VirtIO for best performance
2. **Choose appropriate cache mode**: Use `cache='none'` for production
3. **Enable native AIO**: Use `io='native'` for better async I/O
4. **Use raw format when possible**: Better performance than qcow2
5. **Enable discard/TRIM**: Use `discard='unmap'` for SSD optimization
6. **Configure I/O threads**: Separate I/O processing from vCPU threads
7. **Use multi-queue**: Enable for multi-core VMs

---

## Network Tuning

Network performance is critical for many virtualized workloads.

### Network Traffic Segregation

**Best Practice**: Segregate network traffic to avoid congestion in KVM setups.

- Use **dedicated networks** for different traffic types:
  - Management traffic
  - Backup traffic
  - Live migration traffic
  - Production/application traffic

**Important**: Avoid multiple network interfaces for the same network segment. If you must use multiple interfaces on the same segment, apply network tuning such as `arp_filter` to prevent ARP Flux.

#### Preventing ARP Flux

ARP Flux is an undesirable condition that can occur in both hosts and guests, caused by the machine responding to ARP requests from more than one network interface.

```bash
# Enable arp_filter
echo 1 > /proc/sys/net/ipv4/conf/all/arp_filter

# Make persistent (add to /etc/sysctl.conf)
net.ipv4.conf.all.arp_filter = 1
```

### VirtIO Network Driver

Always use VirtIO network driver for best performance.

```xml
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <driver name='vhost' queues='4'/>
</interface>
```

### vhost-net Architecture

vhost-net is a kernel-level backend for virtio networking that significantly improves performance by reducing the number of context switches and system calls.

#### Architecture Comparison

**Without vhost-net (Traditional virtio):**
```
┌─────────────────────────────────────────────────────────────┐
│                      Virtual Machine                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Guest OS                                 │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │         virtio-net (frontend driver)           │  │   │
│  │  │              TX          RX                     │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                         │          ▲
                         ▼          │
┌─────────────────────────────────────────────────────────────┐
│                      Host Kernel                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    QEMU                               │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │         virtio backend                         │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
│                         │          ▲                         │
│                         ▼          │                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              TAP Device                               │   │
│  └──────────────────────────────────────────────────────┘   │
│                         │          ▲                         │
│                         ▼          │                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Bridge                                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                         │          ▲                         │
│                         ▼          │                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Physical NIC                             │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**With vhost-net (Optimized):**
```
┌─────────────────────────────────────────────────────────────┐
│                      Virtual Machine                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Guest OS                                 │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │         virtio-net (frontend driver)           │  │   │
│  │  │              TX          RX                     │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                         │          ▲
                         ▼          │
┌─────────────────────────────────────────────────────────────┐
│                      Host Kernel                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              vhost-net (kernel module)                │   │
│  │         (bypasses QEMU for data path)                 │   │
│  └──────────────────────────────────────────────────────┘   │
│                         │          ▲                         │
│                         ▼          │                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              TAP Device                               │   │
│  └──────────────────────────────────────────────────────┘   │
│                         │          ▲                         │
│                         ▼          │                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Bridge                                   │   │
│  └──────────────────────────────────────────────────────┘   │
│                         │          ▲                         │
│                         ▼          │                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Physical NIC                             │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Key Benefits of vhost-net:**
- Reduces context switches between kernel and userspace
- Lowers latency
- Reduces CPU usage
- Improves overall network throughput
- Data path bypasses QEMU userspace

#### Enabling vhost-net

**Load the kernel module:**

```bash
# Load vhost-net module
modprobe vhost-net

# Verify module is loaded
lsmod | grep vhost

# Check device file created
ls -l /dev/vhost-net

# Make persistent (add to /etc/modules-load.d/vhost.conf)
echo "vhost-net" > /etc/modules-load.d/vhost.conf
```

**Configure in VM XML:**

```xml
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <driver name='vhost'/>
</interface>
```

**QEMU Command Line:**

When QEMU is launched with `-netdev tap,vhost=on`, it opens `/dev/vhost-net` and initializes the vhost-net instance with several ioctl() calls. This initialization process binds QEMU with a vhost-net instance.

Example from qemu-kvm process:
```bash
-netdev tap,fd=25,id=hostnet0,vhost=on,vhostfd=27 -device virtio-net-pci,
netdev=hostnet0,id=net0,mac=52:54:00:49:3b:95,bus=pci.0,addr=0x3
```

#### vhost-net Zero Copy Transmit

The `experimental_zcopytx` parameter controls Bridge Zero Copy Transmit, which can improve performance for large packet workloads.

**What is Zero Copy Transmit?**

A system for providing zero copy transmission in virtualization environment where the hypervisor receives a guest OS request pertaining to a data packet. The data packet resides in a buffer of the guest OS or guest application and has at least a partial header created during networking stack processing. The hypervisor sends a request to the network device driver to transfer the data packet over the network, identifying the packet in the guest buffer, and refrains from copying the data to a hypervisor buffer.

**When to Use:**
- Environment uses large packet sizes
- Reduces host CPU overhead when guest communicates to external network
- **Does NOT affect**: Guest-to-guest, guest-to-host, or small packet workloads

**Enable Zero Copy:**

```bash
# Check current setting
cat /sys/module/vhost_net/parameters/experimental_zcopytx

# Enable zero copy transmit
echo 1 > /sys/module/vhost_net/parameters/experimental_zcopytx
```

### Multi-queue Networking

Multi-queue virtio-net provides significant performance improvements by allowing parallel packet processing across multiple vCPUs.

#### The Problem with Single Queue

Traditional virtio-net had a single RX (receive) and TX (transmit) queue, which created a bottleneck:

- Even with multiple vCPUs, networking throughput was limited
- Guests couldn't transmit or retrieve packets in parallel
- Virtual NICs couldn't utilize multi-queue support available in Linux kernel
- tap/virtio-net backend had to serialize concurrent transmission/receiving requests from different CPUs
- This serialization caused significant performance overhead

#### Multi-queue Solution

Multi-queue support was introduced in both frontend (guest) and backend (host) drivers to lift this bottleneck:

- Allows guests to scale network performance with more vCPUs
- Each queue can be processed by a different vCPU
- Parallel packet processing significantly improves throughput
- Better CPU cache utilization

#### Configuration

**Host Configuration (VM XML):**

```xml
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <driver name='vhost' queues='4'/>
</interface>
```

Where `queues` value can be 1 to 8 (kernel supports up to 8 queues for multi-queue tap device).

**QEMU Command Line:**

```bash
# Start guest with 2 queues
qemu-kvm -netdev tap,queues=2,... -device virtio-net-pci,queues=2,...
```

**Guest Configuration:**

Inside the guest, enable multi-queue support using ethtool:

```bash
# Check current queue configuration
ethtool -l eth0

# Example output:
# Channel parameters for eth0:
# Pre-set maximums:
# RX:             0
# TX:             0
# Other:          0
# Combined:       4    <-- Maximum queues available
# Current hardware settings:
# RX:             0
# TX:             0
# Other:          0
# Combined:       1    <-- Currently active queues

# Enable multi-queue (set combined queues, where K is from 1 to M)
ethtool -L eth0 combined 4

# Verify the change
ethtool -l eth0
```

**Enable RPS (Receive Packet Steering):**

```bash
# Enable RPS for better distribution
for i in /sys/class/net/eth0/queues/rx-*/rps_cpus; do
    echo f > $i
done

# Or set specific CPU mask (example: CPUs 0-3)
for i in /sys/class/net/eth0/queues/rx-*/rps_cpus; do
    echo 0f > $i
done
```

#### Performance Benefits

Multi-queue virtio-net provides the greatest performance benefit when:

1. **Large Packets**: Traffic packets are relatively large
2. **High Concurrency**: Guest is active on many connections simultaneously
3. **Multiple Traffic Patterns**: Traffic running between:
   - Guest to guest
   - Guest to host
   - Guest to external system
4. **Optimal Queue Count**: Number of queues equals number of vCPUs
   - Multi-queue support optimizes RX interrupt affinity
   - TX queue selection makes specific queue private to specific vCPU

#### Best Practices

1. **Match Queue Count to vCPUs**: Set `queues` parameter equal to number of vCPUs for optimal performance
2. **Monitor CPU Usage**: Multi-queue increases CPU consumption even with better throughput
3. **Test Your Workload**: Performance improvement varies by workload type
4. **Consider Trade-offs**: Higher throughput comes at cost of increased CPU usage
5. **Use with vhost-net**: Combine multi-queue with vhost-net for best results

#### Example: Complete Multi-queue Setup

```xml
<!-- VM with 4 vCPUs and 4 network queues -->
<domain type='kvm'>
  <name>multiqueue-vm</name>
  <vcpu placement='static'>4</vcpu>
  <devices>
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
      <driver name='vhost' queues='4'/>
    </interface>
  </devices>
</domain>
```

**Inside Guest:**

```bash
# Enable all 4 queues
ethtool -L eth0 combined 4

# Enable RPS
for i in /sys/class/net/eth0/queues/rx-*/rps_cpus; do
    echo f > $i
done

# Verify configuration
ethtool -l eth0
cat /sys/class/net/eth0/queues/rx-*/rps_cpus
```

#### Monitoring Multi-queue Performance

```bash
# Check interrupt distribution across CPUs
cat /proc/interrupts | grep virtio

# Monitor network statistics
ethtool -S eth0

# Check queue statistics
tc -s qdisc show dev eth0
```

### vhost-net

vhost-net offloads network processing to the kernel, improving performance.

```xml
<interface type='network'>
  <source network='default'/>
  <model type='virtio'/>
  <driver name='vhost'/>
</interface>
```

#### Enable vhost-net Module

```bash
# Load vhost-net module
modprobe vhost-net

# Make persistent (add to /etc/modules-load.d/vhost.conf)
vhost-net
```

### SR-IOV (Single Root I/O Virtualization)

SR-IOV provides near-native network performance by allowing a single physical PCIe device to present itself as multiple virtual devices.

#### SR-IOV Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Hypervisor                                   │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │              I/O MMU (Intel VT-d or AMD IOMMU)                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                       │
│  ┌─────────────────────┐              ┌─────────────────────┐       │
│  │     Guest OS 1      │              │     Guest OS 2      │       │
│  │  ┌──────────────┐   │              │  ┌──────────────┐   │       │
│  │  │ Virtual NIC  │   │              │  │ Virtual NIC  │   │       │
│  │  │   Driver     │   │              │  │   Driver     │   │       │
│  │  └──────────────┘   │              │  └──────────────┘   │       │
│  └─────────────────────┘              └─────────────────────┘       │
│           │                                      │                   │
│           │                                      │                   │
│           ▼                                      ▼                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              SR-IOV PCI Device (NIC)                         │   │
│  │                                                               │   │
│  │  ┌──────────────────────────────────────────────────────┐   │   │
│  │  │         Physical Function (PF)                        │   │   │
│  │  │      (Full PCIe function with SR-IOV capability)      │   │   │
│  │  │         - Physical NIC Driver                         │   │   │
│  │  └──────────────────────────────────────────────────────┘   │   │
│  │                                                               │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │   │
│  │  │   Virtual    │  │   Virtual    │  │   Virtual    │      │   │
│  │  │  Function 1  │  │  Function 2  │  │  Function N  │      │   │
│  │  │     (VF)     │  │     (VF)     │  │     (VF)     │      │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                    ┌───────────────────────┐
                    │   Physical Network    │
                    └───────────────────────┘
```

**Key Components:**

- **Physical Function (PF)**: Full PCIe function with SR-IOV capability, manages the SR-IOV functionality
- **Virtual Function (VF)**: Lightweight PCIe function that can be assigned to VMs
- **I/O MMU**: Provides memory address translation and isolation (Intel VT-d or AMD IOMMU)

#### Benefits of SR-IOV

1. **Near-Native Performance**: Direct hardware access without hypervisor intervention
2. **Low Latency**: Bypasses virtual switch and hypervisor network stack
3. **High Throughput**: Full bandwidth of physical NIC divided among VFs
4. **CPU Offload**: Network processing offloaded to hardware
5. **Scalability**: Single physical NIC supports multiple VMs

#### Prerequisites

- Hardware support for SR-IOV (check NIC specifications)
- IOMMU support enabled in BIOS (Intel VT-d or AMD-Vi)
- Kernel support for SR-IOV and IOMMU

#### Enable SR-IOV on Host

**1. Check SR-IOV Support:**

```bash
# Check if SR-IOV is supported
lspci -vvv | grep -i sriov

# Example output showing SR-IOV capability:
# Capabilities: [160] Single Root I/O Virtualization (SR-IOV)
#     IOVCap: Migration-, Interrupt Message Number: 000
#     IOVCtl: Enable+ Migration- Interrupt- MSE+ ARIHierarchy+
#     Initial VFs: 64, Total VFs: 64, Number of VFs: 4
```

**2. Enable IOMMU in Kernel:**

```bash
# For Intel processors (add to kernel boot parameters)
intel_iommu=on iommu=pt

# For AMD processors
amd_iommu=on iommu=pt

# Edit /etc/default/grub and add to GRUB_CMDLINE_LINUX
GRUB_CMDLINE_LINUX="... intel_iommu=on iommu=pt"

# Update grub and reboot
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot
```

**3. Enable Virtual Functions:**

```bash
# Check current VF count
cat /sys/class/net/eth0/device/sriov_numvfs

# Enable SR-IOV (example: create 4 VFs for Intel NIC)
echo 4 > /sys/class/net/eth0/device/sriov_numvfs

# Verify VFs created
lspci | grep -i virtual

# Check VF details
ip link show eth0
```

**4. Make Persistent:**

Create a systemd service or add to startup scripts:

```bash
# Create /etc/systemd/system/sriov-enable.service
[Unit]
Description=Enable SR-IOV
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo 4 > /sys/class/net/eth0/device/sriov_numvfs'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target

# Enable the service
systemctl enable sriov-enable.service
```

#### Configure VM with SR-IOV

**Method 1: Direct VF Assignment (hostdev)**

```xml
<interface type='hostdev' managed='yes'>
  <source>
    <address type='pci' domain='0x0000' bus='0x05' slot='0x10' function='0x0'/>
  </source>
  <mac address='52:54:00:6d:90:02'/>
  <vlan>
    <tag id='42'/>
  </vlan>
</interface>
```

**Method 2: Using Network Pool**

```xml
<!-- Define SR-IOV network pool -->
<network>
  <name>sriov-network</name>
  <forward mode='hostdev' managed='yes'>
    <pf dev='eth0'/>
  </forward>
</network>

<!-- Use in VM configuration -->
<interface type='network'>
  <source network='sriov-network'/>
  <mac address='52:54:00:6d:90:02'/>
</interface>
```

**Find VF PCI Address:**

```bash
# List all VFs
virsh nodedev-list --cap pci | grep -i virtual

# Get VF details
virsh nodedev-dumpxml pci_0000_05_10_0
```

#### SR-IOV Considerations

**Advantages:**
- Best network performance (near-native)
- Low CPU overhead
- Hardware-level isolation
- Support for advanced features (VLAN, QoS)

**Limitations:**
- **No Live Migration**: VMs with SR-IOV cannot be live migrated
- **Limited VFs**: Number of VFs limited by hardware (typically 32-64 per PF)
- **Hardware Dependency**: Tied to specific physical hardware
- **Driver Requirements**: Guest needs appropriate VF drivers
- **Management Complexity**: More complex than software-based networking

**Best Use Cases:**
- High-performance computing (HPC)
- Network Function Virtualization (NFV)
- Database servers requiring low latency
- Applications with high network throughput requirements
- When live migration is not required

#### Additional References

- [Fedora SR-IOV Documentation](https://fedoraproject.org/wiki/Features/SR-IOV)
- [KVM PCI Device Assignment](https://fedoraproject.org/wiki/Features/KVM_PCI_Device_Assignment)

### Network Tuning Parameters

#### Host-side Tuning

```bash
# Increase network buffer sizes
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"

# Enable TCP window scaling
sysctl -w net.ipv4.tcp_window_scaling=1

# Increase connection backlog
sysctl -w net.core.netdev_max_backlog=5000
```

#### Guest-side Tuning

```bash
# Enable offload features
ethtool -K eth0 tso on
ethtool -K eth0 gso on
ethtool -K eth0 gro on

# Increase ring buffer size
ethtool -G eth0 rx 4096 tx 4096
```

### Bridge Configuration

Optimize bridge configuration for better performance.

```bash
# Disable netfilter on bridges
sysctl -w net.bridge.bridge-nf-call-iptables=0
sysctl -w net.bridge.bridge-nf-call-ip6tables=0
sysctl -w net.bridge.bridge-nf-call-arptables=0

# Make persistent (add to /etc/sysctl.conf)
net.bridge.bridge-nf-call-iptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-arptables = 0
```

### Best Practices

1. **Use VirtIO with vhost-net**: Best performance for most workloads
2. **Enable multi-queue**: Match queue count to vCPU count
3. **Use SR-IOV for high-performance**: When maximum throughput is needed
4. **Tune buffer sizes**: Increase for high-bandwidth workloads
5. **Enable offload features**: TSO, GSO, GRO in guest
6. **Optimize bridge settings**: Disable unnecessary netfilter
7. **Monitor network statistics**: Use tools like iperf, netperf

---

## Time-keeping Best Practices

Accurate time-keeping is crucial for many applications and system operations. In virtualization environments, maintaining time synchronization between guest and host is critical as it affects many guest operations and can cause unpredictable results if not properly configured.

### Time-keeping Challenges in Virtualization

Different mechanisms exist for time-keeping, with Network Time Protocol (NTP) being one of the best-known techniques for clock synchronization between computer systems over networks.

**Key Considerations:**

- Guest time should be in sync with hypervisor/host
- Time drift can cause authentication failures, log inconsistencies, and application errors
- Multiple methods available for achieving time sync (NTP, hwclock, kvm-clock)
- Best approach depends on your specific setup

**Strategy:**

1. **First**: Make KVM host time stable and in sync (use NTP or similar)
2. **Then**: Keep guest time in sync with host
3. **Best Option**: Use kvm-clock for optimal results

### Clock Sources

Configure appropriate clock source for the VM.

```xml
<clock offset='utc'>
  <timer name='rtc' tickpolicy='catchup'/>
  <timer name='pit' tickpolicy='delay'/>
  <timer name='hpet' present='no'/>
  <timer name='hypervclock' present='yes'/>
</clock>
```

### kvm-clock (Recommended)

kvm-clock is a paravirtualized (virtualization-aware) clock device that provides the most accurate and stable time-keeping for KVM guests.

#### How kvm-clock Works

When kvm-clock is in use:

1. **Guest Requests Time**: Guest asks hypervisor for current time
2. **Shared Page**: Guest registers a page and shares address with hypervisor
3. **Continuous Updates**: Hypervisor keeps updating this shared page
4. **Guest Reads**: Guest simply reads this page whenever it needs time information
5. **Guaranteed Accuracy**: Provides both stable and accurate timekeeping

**Prerequisites:**

- Hypervisor must support kvm-clock
- Guest kernel must have kvm-clock support (built-in for modern Linux kernels)

#### Verify kvm-clock in Guest

**Check if kvm-clock is loaded:**

```bash
# Check kernel messages for kvm-clock
dmesg | grep kvm-clock

# Example output:
[root@kvmguest ]$ dmesg | grep kvm-clock
[ 0.000000] kvm-clock: Using msrs 4b564d01 and 4b564d00
[ 0.000000] kvm-clock: CPU 0, msr 4:27fcf001, primary CPU clock
[ 0.027170] kvm-clock: CPU 1, msr 4:27fcf041, secondary CPU clock
[ 0.376023] kvm-clock: CPU 30, msr 4:27fcf781, secondary CPU clock
[ 0.388027] kvm-clock: CPU 31, msr 4:27fcf7c1, secondary CPU clock
[ 0.597084] Switched to clocksource kvm-clock
```

**Verify kvm-clock is active clocksource:**

```bash
# Check current clocksource
cat /sys/devices/system/clocksource/clocksource0/current_clocksource

# Expected output:
kvm-clock

# List available clocksources
cat /sys/devices/system/clocksource/clocksource0/available_clocksource

# Example output:
# kvm-clock tsc acpi_pm
```

**Manually set kvm-clock (if needed):**

```bash
# Set kvm-clock as clocksource
echo kvm-clock > /sys/devices/system/clocksource/clocksource0/current_clocksource

# Verify
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
```

### Timer Configuration

#### For Linux Guests

```xml
<clock offset='utc'>
  <timer name='rtc' tickpolicy='catchup'/>
  <timer name='pit' tickpolicy='delay'/>
  <timer name='hpet' present='no'/>
  <timer name='kvmclock' present='yes'/>
</clock>
```

#### For Windows Guests

```xml
<clock offset='localtime'>
  <timer name='rtc' tickpolicy='catchup'/>
  <timer name='pit' tickpolicy='delay'/>
  <timer name='hpet' present='yes'/>
  <timer name='hypervclock' present='yes'/>
</clock>
```

### NTP Configuration in Guests

Even with kvm-clock, it's recommended to configure NTP/Chrony in guests for additional time synchronization.

#### Linux Guest with Chrony

```bash
# Install chrony
yum install chrony  # RHEL/CentOS
apt install chrony  # Debian/Ubuntu

# Edit /etc/chrony.conf
# Add NTP servers
server 0.pool.ntp.org iburst
server 1.pool.ntp.org iburst
server 2.pool.ntp.org iburst

# Use makestep for large time corrections
# makestep <threshold> <limit>
# threshold: step if offset is larger than this (seconds)
# limit: number of clock updates (-1 = unlimited)
makestep 1.0 -1

# Enable and start chronyd
systemctl enable chronyd
systemctl start chronyd

# Check synchronization status
chronyc tracking
chronyc sources
```

#### Linux Guest with NTP

```bash
# Install ntp
yum install ntp  # RHEL/CentOS
apt install ntp  # Debian/Ubuntu

# Edit /etc/ntp.conf
# Add NTP servers
server 0.pool.ntp.org iburst
server 1.pool.ntp.org iburst
server 2.pool.ntp.org iburst

# Disable NTP slewing (use stepping instead)
# This is important for VMs
tinker panic 0

# Enable and start ntpd
systemctl enable ntpd
systemctl start ntpd

# Check synchronization status
ntpq -p
ntpstat
```

### Guest Configuration

#### Linux Guest Complete Setup

```bash
# 1. Verify kvm-clock is active
cat /sys/devices/system/clocksource/clocksource0/current_clocksource

# 2. Set kvm-clock as clocksource (if not already set)
echo kvm-clock > /sys/devices/system/clocksource/clocksource0/current_clocksource

# 3. Make persistent (add to /etc/rc.local or systemd service)
echo 'echo kvm-clock > /sys/devices/system/clocksource/clocksource0/current_clocksource' >> /etc/rc.local

# 4. Configure NTP/Chrony (see above sections)

# 5. Verify time synchronization
timedatectl status
```

#### Windows Guest

1. **Install VirtIO drivers** including viostor and vioscsi
2. **Configure Windows Time service:**

```cmd
# Set time source to hypervisor
w32tm /config /syncfromflags:VM /update

# Restart Windows Time service
net stop w32time && net start w32time

# Check time service status
w32tm /query /status

# Force synchronization
w32tm /resync

# Display time source
w32tm /query /source
```

3. **Additional Windows Configuration:**

```cmd
# Set time service to automatic startup
sc config w32time start= auto

# Configure time correction settings
w32tm /config /update /manualpeerlist:"time.windows.com" /syncfromflags:manual

# Check configuration
w32tm /query /configuration
```

### TSC (Time Stamp Counter)

Configure TSC for better time-keeping performance.

```xml
<cpu mode='host-passthrough'>
  <feature policy='require' name='invtsc'/>
</cpu>
<clock offset='utc'>
  <timer name='tsc' frequency='3000000000' mode='native'/>
</clock>
```

### Best Practices

1. **Use kvmclock for Linux**: Best time source for KVM guests
2. **Use Hyper-V enlightenments for Windows**: Better time-keeping
3. **Disable HPET when not needed**: Reduces overhead
4. **Use catchup tickpolicy**: Handles time drift better
5. **Configure NTP/Chrony**: Keep guest time synchronized
6. **Use invtsc feature**: For better TSC stability
7. **Avoid CPU overcommitment**: Reduces time-keeping issues

---

## Summary and Best Practices

### Quick Reference Checklist

#### CPU Optimization
- [ ] Allocate appropriate number of vCPUs (avoid over-subscription)
- [ ] Pin vCPUs to physical CPUs for consistent performance
- [ ] Configure CPU topology matching physical layout
- [ ] Use host-passthrough mode for best performance
- [ ] Align vCPUs with NUMA nodes

#### Memory Optimization
- [ ] Allocate sufficient memory for workload
- [ ] Enable hugepages for memory-intensive workloads
- [ ] Configure NUMA memory policies
- [ ] Lock memory for real-time workloads
- [ ] Consider KSM for high-density environments

#### Storage Optimization
- [ ] Use VirtIO-SCSI or VirtIO-BLK drivers
- [ ] Set cache='none' for production workloads
- [ ] Enable native AIO (io='native')
- [ ] Use raw format when snapshots not needed
- [ ] Configure I/O threads for better performance
- [ ] Enable multi-queue for multi-core VMs
- [ ] Enable discard/TRIM for SSDs

#### Network Optimization
- [ ] Use VirtIO network driver with vhost-net
- [ ] Enable multi-queue networking
- [ ] Match queue count to vCPU count
- [ ] Consider SR-IOV for high-performance needs
- [ ] Tune network buffers and offload features
- [ ] Optimize bridge configuration

#### Time-keeping
- [ ] Use kvmclock for Linux guests
- [ ] Use Hyper-V enlightenments for Windows
- [ ] Configure appropriate timer settings
- [ ] Disable HPET when not needed
- [ ] Set up NTP/Chrony in guests

### Performance Monitoring

#### Host-level Monitoring

```bash
# CPU usage
top -b -n 1 | head -20
mpstat -P ALL 1

# Memory usage
free -h
vmstat 1

# Disk I/O
iostat -x 1
iotop

# Network
iftop
nethogs
```

#### VM-level Monitoring

```bash
# List all VMs
virsh list --all

# VM CPU stats
virsh cpu-stats <domain>

# VM memory stats
virsh dommemstat <domain>

# VM block stats
virsh domblkstat <domain> <device>

# VM network stats
virsh domifstat <domain> <interface>
```

### Common Performance Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| High CPU steal time | Poor performance, high %st in top | Reduce vCPU count or pin vCPUs |
| Memory swapping | Slow performance, high swap usage | Increase VM memory or enable hugepages |
| Disk I/O bottleneck | High iowait, slow disk operations | Use VirtIO, enable native AIO, use faster storage |
| Network latency | High ping times, packet loss | Enable multi-queue, use vhost-net or SR-IOV |
| Time drift | Clock skew, authentication issues | Configure kvmclock, use NTP |
| NUMA imbalance | Uneven memory access times | Pin vCPUs and memory to same NUMA node |

### Recommended VM Configuration Template

```xml
<domain type='kvm'>
  <name>optimized-vm</name>
  <memory unit='GiB'>8</memory>
  <currentMemory unit='GiB'>8</currentMemory>
  <memoryBacking>
    <hugepages>
      <page size='1' unit='GiB'/>
    </hugepages>
    <locked/>
  </memoryBacking>
  <vcpu placement='static'>4</vcpu>
  <iothreads>2</iothreads>
  <cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='3'/>
    <emulatorpin cpuset='0-1'/>
    <iothreadpin iothread='1' cpuset='4'/>
    <iothreadpin iothread='2' cpuset='5'/>
  </cputune>
  <numatune>
    <memory mode='strict' nodeset='0'/>
  </numatune>
  <cpu mode='host-passthrough'>
    <topology sockets='1' cores='4' threads='1'/>
    <numa>
      <cell id='0' cpus='0-3' memory='8' unit='GiB'/>
    </numa>
  </cpu>
  <clock offset='utc'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
    <timer name='kvmclock' present='yes'/>
  </clock>
  <devices>
    <disk type='file' device='disk'>
      <driver name='qemu' type='raw' cache='none' io='native' iothread='1'/>
      <source file='/var/lib/libvirt/images/vm.img'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <controller type='scsi' index='0' model='virtio-scsi'>
      <driver queues='4' iothread='2'/>
    </controller>
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
      <driver name='vhost' queues='4'/>
    </interface>
    <memballoon model='virtio'>
      <stats period='10'/>
    </memballoon>
  </devices>
</domain>
```

### Additional Resources

- **KVM Documentation**: https://www.linux-kvm.org/
- **libvirt Documentation**: https://libvirt.org/
- **Red Hat Virtualization Tuning Guide**: https://access.redhat.com/documentation/
- **QEMU Documentation**: https://www.qemu.org/documentation/

---

## Conclusion

Performance tuning in KVM requires a holistic approach considering CPU, memory, storage, and network optimization. By following the best practices outlined in this guide and continuously monitoring performance metrics, you can achieve optimal VM performance for your workloads.

Remember that performance tuning is an iterative process. Start with baseline measurements, apply optimizations incrementally, and measure the impact of each change. Different workloads may require different optimization strategies, so always test configurations in a non-production environment first.

---

**Document Version**: 1.0  
**Last Updated**: 2026-05-12
**Author**: perfAge Team :)

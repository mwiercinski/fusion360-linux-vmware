# Running virtualised Fusion 360 on Linux using VMWare Workstation Player

This is a recipe for running [Autodesk Fusion 360](https://www.autodesk.com/products/fusion-360/overview) on Linux by using [VMWare Workstation Player](https://www.vmware.com/products/workstation-player.html) running Windows 11.

It is a collection of settings I found useful to make it running smooth, along with notes on how I understand them. It is entirely possible that some settings are redundant/overlapping, and/or I've misinterpreted VMWare configuration knobs.  Please feel free to make suggestions or ask questions about this guide. I've purposely posted these instructions on Github to make it easy to suggest edits and point mistakes.

**Disclaimer**: I am not in any way, shape, nor form affiliated with neither Autodesk nor VMWare. I tried some things, and these worked for me.

## **TL;DR**

1) Stick the below in your `*.vmx` machine descriptor. Replace `XXX` with an amount of memory given to the machine:
```
# Reserves and preallocates all guest memory.
sched.mem.min = "XXX" # To match memsize
sched.mem.pin = "TRUE"
# Don't return memory to the host. Possibly redundant with full pre-allocation.
MemTrimRate = "0"
# Prevents VMware from adjusting the memory size of the virtual machine if it cannot allocate enough memory.
MemAllowAutoScaleDown = "FALSE"
# Fits all virtual machine memory into reserved host RAM.
prefvmx.useRecommendedLockedMemSize = "TRUE"
# Ratio of guest memory to be fit in host RAM. Possibly redundant to the above.
prefvmx.minVmMemPct = "100"
# Prevents VMware from creating a .vmem file. The option value is somewhat misleading. 
mainmem.backing = "swap"
```

2) Disable Windows swap. In "Advanced" tab  -> Performance -> Settings -> Advanced -> Virtual memory -> Change -> No paging file
3) Configure Fusion to use DirectX 9.
4) Give plenty GPU RAM to the guest.
5) Give plenty CPU to the guest

Still doesn't work? Disable memory compactions: `sudo sysctl -w vm.compaction_proactiveness=0`



## What, why?
### Step #1: Put guest memory in preallocated RAM 

I found anecdotal evidence that VMWare allocates memory in a way suboptimal for the application. During Fusion slowdowns, I've noticed `kcompactd` fully saturating a single CPU core it was running on. The process is responsible for defragmenting memory pages. According to an interpretation in this [bug thread](https://bugzilla.redhat.com/show_bug.cgi?id=1694305), this can be a result of unavailability of large contiguous memory blocks to satisfy VMware's request, what triggers memory defragmentation:

> vmware in particular is making a
> large number of high-order allocation requests and waking kcompactd at high
> frequency to try and allocate them and failing due to memory fragmentation
> issues

The easiest way to work around allocation performance is to do it once, at the startup. This way there is no host memory allocation to worry about during interactive usage. I.e. no allocation, no problems. Ultimately it also means less memory management for the host - rather than iteratively ramping up guest's allocation, potentially realigning memory layout multiple times in the process, we only do it once. There is an obvious downside - we need to dedicate enough RAM to support Fusion usage peaks, which will be otherwise wasted.

The config snippet below preallocates 100% of the guest RAM (24GB in my case), and precludes returning the memory to the host. 

```
memsize = "24576"  # Set by VMPlayer, quoted for reference only.
# Reserves and preallocates all guest memory.
sched.mem.min = "24576"
sched.mem.pin = "TRUE"
# Don't return memory to the host. Possibly redundant with full pre-allocation.
MemTrimRate = "0"
```

After starting VMWare, this can be verified with `pmap`:
```
$ sudo pmap -x $(pgrep vmware-vmx) | awk '$3 > 1000000'  # Third column is RSS, the resident size
Address           Kbytes     RSS   Dirty Mode  Mapping
00007f2e48000000 25165824 25165824 25165824 rw-s- vmem (deleted)
total kB         28453956 25526652 25466204
```


The bug quoted above discusses alternative workarounds that I found either to be either too intrusive for the host (using huge pages, tuning Linux memory compaction), or I haven't tested once I got the simple solution working (e.g. tuning VMWare's page trimming behaviour).

I also found a setting to prevent VMWare silently reducing memory allocation - this is not important for performance, but eliminates a potential confusion due to silent fallback to less memory:

```
# Prevents VMware from adjusting the memory size of the virtual machine if it cannot allocate enough memory.
MemAllowAutoScaleDown = "FALSE"
```

### Step #2: Prevent host swapping

We want to avoid any guest memory paged being stored on the host disk - this will slow things down a lot. Swapping can happen at the Linux (`kswapd`) or VMware level (`.vmem` files). Assuming we've forced 100% of memory into physical host RAM as per step above, no swapping should be needed, so let's just disable both. 

```
# Fits all virtual machine memory into reserved host RAM.
prefvmx.useRecommendedLockedMemSize = "TRUE"
# Ratio of guest memory to be fit in host RAM. Possibly redundant to the above.
prefvmx.minVmMemPct = "100"
# Prevents VMware from creating a .vmem file. The option value is somewhat misleading. 
mainmem.backing = "swap"
```

Note, this doesn't preclude use of swap for other host processes, what may be useful if you are stretching your RAM limits. There is little to gain by swapping guest's RAM on the host, though. Giving guest more memory than we have backed by physical RAM, would create an illusion of memory availability and potentially preclude memory recovery mechanisms on the guest side. While use of swap can arguably let us use less RAM, it opens a range of problems such as allocation performance discussed above, or exposing the swap management process to potential inefficiencies stemming from memory management being done by two operating systems. I.e. if you need to swap to make up for RAM shortage, let native Linux processes do so, and keep the VM fully in the physical RAM. It will be simpler to run, more efficient, and easier to troubleshoot.

### Step #3: Prevent guest swapping
Guest swapping is similarly undesired. Firstly, we want everything to run from the RAM for a better performance in principle.  If Fusion crashes due to memory pressure, it is a useful signal that allocation needs to be made higher, whereas swapped memory will result in sluggish performance that may not be obvious as to the reason of. Full RAM model is more performant, and easier to troubleshoot.

Secondly, we should consider how VMWare deals with image files. Windows 11 typically creates a special "pagefile" on a disk, that acts as a swap area. Assuming the simplest model, where system partition is backed by a single `.vmdk`, mixing swap and filesystem usage means VMWare has to make difficult choices - either rely on Linux buffers and caches, or attempt direct I/O (e.g. `O_DIRECT`). With former - Windows swap effectively interfaces with the disk via Linux virtual memory system, which means it will rival the host memory pages exactly at the time when extra RAM is necessary. It will either affect the host performance (meaning the memory pressure is strong), or if the host has an ample RAM supply - be completely unnecessary (meaning we might have given memory to the guest to avoid any I/O in the first place). If VMWare were to use `O_DIRECT` for swapping a guest within a single `.vmdk` it would sacrifice a lot of Linux memory-I/O optimisations. Non-surprisingly in my case VMWare uses regular I/O (this [can be checked](https://www.linuxquestions.org/questions/programming-9/how-to-find-out-which-flags-the-process-used-to-open-a-file-4175542258/) with `/proc/*/fdinfo`).

In order to disable swap on Windows 11:
 *  in search bar type "View advanced system settings"
 * in "Advanced" tab  -> Performance -> Settings -> Advanced -> Virtual memory -> Change -> No paging file

### Step #4: Use DirectX 9
I found that virtualized Fusion works better on DirectX 9 than on DirectX 11 (slower) and OpenGL  (unstable). 

To change click profile menu in Fusion -> Preferences -> General -> Graphics driver -> DirectX 9

### Step #5: Allocate GPU RAM

I gave the guest 8GB of the GPU RAM (out of total 12 on RTX 2080Ti).  `.vmx` settings (this can be also changed in the `vmplayer` UI):

```
vmotion.svga.graphicsMemoryKB = "8388608"
```
It seems to be plenty - the guest usually fits under 2GB and GPU  (can be checked with `nvidia-smi`). The GPU is used in 30-40% during bursts, e.g. when examining a relatively complex model (e.g 3d printer assembly). I run VM on a 4K monitor.

### Step #6: Set CPUs
I've used 8 out of 20 cores on `i9-9900x`. This can be done from VMPlayer UI.

### Step #7: Still problems? Disable memory compactions.

I've noticed even with all the host memory in physical RAM, I sometimes see `kcompactd` flagging up. This doesn't always happen, but when it does, disabling memory compactions for good makes Fusion butter smooth: 
```
# The default is 20.
sudo sysctl -w vm.compaction_proactiveness=0 
```
While you can also do it as a default `sysctl` setting, I wouldn't recommend doing so. 

## Misc

### Various kernel knobs

These are settings I came across but have not investigated fully:
 *  `vm.compact_unevictable_allowed`
 *  `vm.extfrag_threshold` - I suspect analyzing `/sys/kernel/debug/extfrag/extfrag_index` during the vmware slowness may shed light on the root cause. In which case it is likely possible to adjust the threshold instead of disabling compactions.


### Huge pages

It doesn't seem that VMware is making use of huge pages even if kernel allows for it via `madvise`. I've checked it by examining `/proc/meminfo`:

```
$ grep -i hugepages_ /proc/meminfo
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
```

###  Intel IOMMU kernel driver

This [post](https://unix.stackexchange.com/questions/679706/kcompacd0-using-100-cpu-with-vmware-workstation-16) discusses a situation where Intel IOMMU (Input-Output Memory Management Unit) kernel driver is not active, with similar symptoms. I have not tried experimenting with it, yet.

### Force pre-compacting memory
I tried force precompacting memory via `/proc/sys/vm/drop_caches` followed by `/proc/sys/vm/compact_memory` before launching Fusion, but that doesn't seem to stop `kcompactd` issues. 

### I/O setup performance
I've experimented with pre-allocating disk space for `.vmdk`, but I found no evidence of I/O being a significant performance factor, regardless of whether the image is preallocated or growing. There was also a little difference even after I've used `mmap` + `mlock` to force the image into RAM. That said, I highly recommend to use SSD as a backing storage.

### Misc VMware settings
Here are further settings I found as recommended, that happen to sit in my `.vmx`, but as I understand them they should not make a difference with fully pre-allocated memory.

```
# Memory sharing involves background scan to dedupe pages. 
# Apparently this is used to dedupe similar pages across multiple instance of the guest. I don't know if the process applies to single guest pages.
sched.mem.pshare.enable = "FALSE"

## These two are not very important for the runtime performance.
# Do not take snapshots in the background.
mainMem.partialLazySave = "FALSE"
# Do not restore snapshots in the background.
mainMem.partialLazyRestore = "FALSE"
```

### My setup:

My machine has `i9-9900X` CPU, 128G RAM, and `RTX 2080Ti` in a 12G version. While a powerful machine, it is a definitely overkill for the application, and the fact that without proper settings virtualised Fusion 360 is barely usable is a good testimony that throwing money at the problem doesn't always work.

```
$ uname -a
Linux hal 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:30:08 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux

$ grep -r . /sys/kernel/mm/transparent_hugepage/*
/sys/kernel/mm/transparent_hugepage/defrag:always defer defer+madvise [madvise] never
/sys/kernel/mm/transparent_hugepage/enabled:always [madvise] never
/sys/kernel/mm/transparent_hugepage/hpage_pmd_size:2097152
/sys/kernel/mm/transparent_hugepage/khugepaged/defrag:1
/sys/kernel/mm/transparent_hugepage/khugepaged/max_ptes_shared:256
/sys/kernel/mm/transparent_hugepage/khugepaged/scan_sleep_millisecs:10000
/sys/kernel/mm/transparent_hugepage/khugepaged/max_ptes_none:511
/sys/kernel/mm/transparent_hugepage/khugepaged/pages_to_scan:4096
/sys/kernel/mm/transparent_hugepage/khugepaged/max_ptes_swap:64
/sys/kernel/mm/transparent_hugepage/khugepaged/alloc_sleep_millisecs:60000
/sys/kernel/mm/transparent_hugepage/khugepaged/pages_collapsed:142
/sys/kernel/mm/transparent_hugepage/khugepaged/full_scans:252107
/sys/kernel/mm/transparent_hugepage/shmem_enabled:always within_size advise [never] deny force
/sys/kernel/mm/transparent_hugepage/use_zero_page:1

```

### Disable suspend on guest

I've initially found out that while sometimes Fusion in VMware would work okish for some time, it was always sluggish after recovering from suspension. So I've disabled power settings in the guest Windows, what was a performance bandaid at the time, but it is not necessary after working around `kcompactd` bottleneck with preallocated RAM. That said, I like my apps to not close on their own, and I simply keep Windows / Fusion app open for most of the time.

## References
* https://gist.github.com/wpivotto/3993502
* https://gist.github.com/2E0PGS/2560d054819843d1e6da76ae57378989#file-vmware-workstation-khugepaged-fix-md
* https://bugzilla.redhat.com/show_bug.cgi?id=1694305
* https://medium.com/@huynhquangthao/linux-large-memory-allocation-history-570730b09c95
* https://www.sanbarrow.com/vmx/vmx-config-ini.html
* https://communities.vmware.com/t5/VMware-Workstation-Pro/About-quot-mainMem-useNamedFile-FALSE-quot-Advantages/td-p/162612
* https://communities.vmware.com/t5/VMware-Workstation-Pro/VMWare-workstation-in-a-fistfight-with-Linux-Memory-Compactor/td-p/2876992
* https://communities.vmware.com/t5/VMware-Workstation-Pro/kcompacd0-using-100-CPU-with-VMware-Workstation-16/td-p/2896972
* https://wiki.archlinux.org/title/VMware
* https://www.kernel.org/doc/Documentation/sysctl/vm.txt
* https://doc.opensuse.org/documentation/leap/tuning/html/book-tuning/cha-tuning-memory.html
* https://docs.oracle.com/en/operating-systems/oracle-linux/9/osmanage/osmanage-ConfiguringHugePages.html#osmanage_ConfiguringHugePages
* https://patchwork.kernel.org/project/linux-mm/patch/20200615143614.15267-1-nigupta@nvidia.com/#23424875
* https://techtalk.intersec.com/2013/07/memory-part-2-understanding-process-memory/
* https://access.redhat.com/solutions/320303
* https://www.pingcap.com/blog/linux-kernel-vs-memory-fragmentation-1/ and https://www.pingcap.com/blog/linux-kernel-vs-memory-fragmentation-2/ - nice explanation of migration types and fragmentation index

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

## Misc
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

### Disable suspend on guest

I've initially found out that while sometimes Fusion in VMware would work okish for some time, it was always sluggish after recovering from suspension. So I've disabled power settings in the guest Windows, what was a performance bandaid at the time, but it is not necessary after working around `kcompactd` bottleneck with preallocated RAM. That said, I like my apps to not close on their own, and I simply keep Windows / Fusion app open for most of the time.

## References
* https://gist.github.com/wpivotto/3993502
* https://bugzilla.redhat.com/show_bug.cgi?id=1694305
* https://medium.com/@huynhquangthao/linux-large-memory-allocation-history-570730b09c95
* https://www.sanbarrow.com/vmx/vmx-config-ini.html
* https://communities.vmware.com/t5/VMware-Workstation-Pro/About-quot-mainMem-useNamedFile-FALSE-quot-Advantages/td-p/162612
* https://wiki.archlinux.org/title/VMware

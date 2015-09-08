###**Linux Boot Process**
------------------------------------
* **Overview (steps)**
	1. Power-up
	2. System startup (Bios/BootMonitor)
	3. Stage 1 bootloader (Master Boot Record)
	4. Stage 2 bootloader (LILO, GRUB)
	5. kernel (linux)
	6. init (userspace)


1. **Power-up**
----------------------------------
Booting Linux begins in the BIOS at address **0xFFFF0**, and it's composed of two steps.

* POST test
* Local device enumeration and initialization 

The BIOS runtime searches for the devices that are both *active* and *bootable* , and it selects the preferred one (this I believe is the one set in the BIOS boot sequence, and is kept in CMOS -- fancy). 
Considering that we usually boot from disk, then once the disks is active, the **MBR** is read. The **MBR** resides on the *first sector* (first 512-bytes, sector 1 of cylinder 0, head 0 ) of the bootable disk. After the **MBR** is *loaded* into RAM the BIOS *yields* control to it.

2. **Stage 1 boot loader**
-------------------------------------
![](http://www.ibm.com/developerworks/library/l-linuxboot/fig2.gif)
As seen in the image above the MBR is composed of 3 sections.

*	The first section (446 bytes) is the **primary boot loader code**
*	The second part (64 bytes) is the **partition table** (each partition has 16 bytes, and that is why we only get 4 primary partitions)
*	The last part (2 bytes) is the **magic numbers**(0xAA55), a section used to *validate* the MBR

The job of the stage 1 boot loader is to find and load the second stage boot loader. It does this by looking for an *active partition table* (I believe this one is set by parted/fdisk via the **83** HEX ID). Once it finds it it *scans* the rest of the partitions to ensure they are *inactive*. Once the verification is done it *reads* the *boot record* from the active partition and *loads* it into RAM, and stage 2 starts

3. **Stage 2 boot loader**
-------------------------------------------
Also known as *kernel loader*, because it's purpose it's to load the kernel and initrd into RAM. Once the kernel is in RAM it gets invoked. The most popular boot loader is GRUB, and as a special mention it also knows 
Linux files systems, meaning the kernel can reside on ext2/ext3 file systems.
**Should have more documentation, see GRUB**

4. **Kernel**
------------------------------------------
With the kernel loaded in memory the next stage begins. The first step is decompressing the kernel, because as remembered from the Gentoo days it is a bzimage.
The first section of this file contains a routine which does some minimal hardware setup, decompresses the kernel, and loads it into **high** memory. It also moves the intird disk, if present, for later use. After this the routine calls the kernel and the kernel boot begins
The routine looks like the following, and it is in essence the main function for the kernel:
![](http://www.ibm.com/developerworks/library/l-linuxboot/fig3.gif)
The second **statrup_32()** function is also called process 0 (swapper), because it initializes the page tables and it enables memory paging.
**start_kernel** does a lot of initialization to set up interrupts, perform further memory configuration, and load the initial RAM disk. In the end, a call is made to **kernel_thread** to start the **init** function, which is the first user-space process. Finally, the idle task is started and the scheduler can now take control (after the call to **cpu_idle**). With interrupts enabled, the pre-emptive scheduler periodically takes control to provide multitasking.
Stage 2 loaded and mounted the intird, so now that disk is used as a temporary root file system, until the root file system for the disk is ready. This is useful because the kernel can fully boot without mounting any disks so it doesn't require code to handle different disk drivers. After the boot the root file system is pivoted **pivot__root** and intird is unmounted.
Because the root file system is a file system on a disk, the initrd function provides a means of bootstrapping to gain access to the disk and mount the real root file system (with this feature we could keep the root file system on NFS).

5. **INIT**
------------------------------------------
After kernel boot the first user-space application is started. This is **init**, the first standard C compiled program

6. **Runlevels**
-----------------------------------------
Runlevels define what users can accomplish in the current state. Every linux system supports three basic run levels plus one or more for normal operation. 

*	**Basic**
| level|purpose|
|------|-------|
|0     |Shutdown, halt  |  
|1     |Single user mode|
|6     |Reboot          |

Beyond the basics, runlevel usage differs among distributions. The following is just and example.

* **Extended**
|level|purpose|
|-----|-------|
|2    |Multiuser mode without networking   |
|3    |Multiuser mode with networking      |
|5    |Multiuser mode with networking and X|

When the system starts the default runlevel is the one set in /etc/inittab, under the **id:** definition.
Methods for changing runlevels:

1.	Change the default runlevel in /etc/inittab and reboot
2.	Add the run level in the kernel stanza in grub.conf. This can be done at boot time, by pressing **e**(or necessary key to get the interactive GRUB prompt.)
3.	telinit can be used also to change run levels. Supposing you are in runlevel 3, and want to change to 5 you just write **telinit 5**. Telinit is a link to init, but behaves accordingly to the name it was called with.
	5.	Still unsure of the difference between telinit and init, because they seem to do the same thing.
		6.	The init executable knows whether it was called as init or telinit and behaves accordingly. 
		7.	Since init runs as PID 1 at boot time, it is also smart enough to know when you subsequently invoke it using init rather than telinit. If you do, it will assume you want it to behave as if you had called telinit instead. For example, you may use init 5 instead of telinit 5 to switch to runlevel 5

**Single user mode** is used to recover a system or test new hardware. To switch to it you can do **telinit 1** or even **telinit s**.  It is usually just a shell with no networking and only a handful of daemons running. Some systems don't implement authentication so it's a possibility to hack a linux system if you have physical access.
A little explanation of the **shutdown** command. The **shutdown** command normally puts the system in runlevel 1, and you have to specify -h for halt and -r for reboot. It also send some messages to users to announce the downtime, and you have to set a timer.
**inittab file**
the file has the following structure:

	id:runlevels:action:process
The available actions are:
|action| description|
|-----------|-------------------------------------------| 
|respawn|	Restart the process whenever it terminates. Usually used for getty processes, which monitor for logins.|
|wait|	Start the process once when the specified runlevel is entered and wait for its termination before init proceeds.|
|once|	Start the process once when the specified runlevel is entered.|
|initdefault|	Specifies the runlevel to enter after system boot.|
|ctrlaltdel|	Execute the associated process when init receives the SIGINT signal, for example, when someone on the system console presses CTRL-ALT-DEL. |
**Initialization scripts**
Initialization scripts are called in the following way:

    l3:3:wait:/etc/rc.d/rc 3
    l5:5:wait:/etc/rc.d/rc 5
    
The **rc** script takes as a parameter the runlevel number, and based on that it goes on disk in **/etc/rc.${RUNLEVEL}/** and runs the scripts from there.
The scripts from the **rc.${RUNLEVEL}**directory have the format **(K|S)[0-9]{2}script_name**. The S stands for start and K stands for kill, so when booting up the scripts starting with S are called, and when powering down the ones with K are called. The two digit number from the name is used to indicate to priority order.

**Upstart**
Upstart is one of the major systems which try to replace the old System V init process. The motivation for these kind of systems is that init is slow to start(it waits until all the scripts called are finished, one by one), and also modern systems make use of hot-pluggable devices or network file systems which my not be availabel at boot.
Upstart resolves these kind of issues by making use of events. Events may be triggered by hardware changes, starting or stopping or tasks, or by any other process on the system. Events are used to trigger tasks or services, collectively known as jobs. So, for example, connecting a USB drive might cause the udev service to send a block-device-added event, which would cause a defined task to check /etc/fstab and mount the drive if appropriate. As another example, an Apache web server may be started only when both a network and required filesystem resources are available.

**Systemd**
The other major player(and scurge of the linuxsphere) is systemd. Systemd creates sockets at startup, but only starts the associated task when a connection request for services on that socket is received. In this way, services can be started only when they are first required and not necessarily at system initialization. Services that need some other facility will block until it is available, so only those services that are waiting for some other process need block while that process starts. 
In the same vein the mount points are made available instantly via autofs, but are actually mounted only happens when a process accesses that mount point.


**TL;RD**
------------------------------------------
When a system is first booted, or is reset, the processor executes code at a well-known location. In a personal computer (PC), this location is in the basic input/output system (BIOS), which is stored in flash memory on the motherboard. The central processing unit (CPU) in an embedded system invokes the reset vector to start a program at a known address in flash/ROM. In either case, the result is the same. Because PCs offer so much flexibility, the BIOS must determine which devices are candidates for boot. We'll look at this in more detail later.

When a boot device is found, the first-stage boot loader is loaded into RAM and executed. This boot loader is less than 512 bytes in length (a single sector), and its job is to load the second-stage boot loader.

When the second-stage boot loader is in RAM and executing, a splash screen is commonly displayed, and Linux and an optional initial RAM disk (temporary root file system) are loaded into memory. When the images are loaded, the second-stage boot loader passes control to the kernel image and the kernel is decompressed and initialized. At this stage, the second-stage boot loader checks the system hardware, enumerates the attached hardware devices, mounts the root device, and then loads the necessary kernel modules. When complete, the first user-space program (init) starts, and high-level system initialization is performed.

That's Linux boot in a nutshell. Now let's dig in a little further and explore some of the details of the Linux boot process.

**Resources**
http://www.ibm.com/developerworks/library/l-linuxboot/
http://www.ibm.com/developerworks/library/l-lpic1-v3-101-3/







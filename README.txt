qemu-rpi (c) 2012-2013 Gregory Estrade, licensed under the GNU GPLv2 and later.

This README file is only there to keep a track of the project's progress.

To get the actual project files, see :
https://github.com/Torlus/qemu/tree/rpi

================================================================================
*** Update 01/29/2013
================================================================================
Added a very alpha version of the USB controller emulation.
On Linux, keyboard support works fine, mouse support... not so much (if at all).
And unfortunately, it prevents RiscOS to boot properly. :(

On latest official Debian images, you will have to disable sound, as it is not
currently emulated, and worse, not doing it will cause a kernel Oops whenever 
sound is requested (you will encounter it by starting X-Window, for instance).
To do so, remove the "-snapshot" from the QEMU command line (so you will not
lose your changes), log into the system and (as root) comment out the 
"snd-bcm2835" line in /etc/modules. Restart the system, and you're done.

Here is the new command line:
/path/to/qemu-system-arm -kernel kernel.img -cpu arm1176 -m 512 -M raspi -no-reboot -serial stdio -append "rw earlyprintk loglevel=8 panic=120 keep_bootcon rootwait dma.dmachans=0x7f35 bcm2708_fb.fbwidth=1024 bcm2708_fb.fbheight=768 bcm2708.boardrev=0xf bcm2708.serial=0xcad0eedf smsc95xx.macaddr=B8:27:EB:D0:EE:DF sdhci-bcm2708.emmc_clock_freq=100000000 vc_mem.mem_base=0x1c000000 vc_mem.mem_size=0x20000000  dwc_otg.lpm_enable=0 kgdboc=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait" -sd 2012-12-16-wheezy-raspbian.img -device usb-kbd -device usb-mouse

See below for more information about the general setup.

================================================================================
*** Update 01/05/2013
================================================================================
Happy new year!
For the first release of the year, I'm glad to announce that due to some work
performed on the framebuffer, VC->ARM property mailbox and the eMMC SDHC host,
RiscOS is now booting fine!

I though I could keep my patches separated from QEMU source code, but it turned
out that I was wrong. Some features are lacking in the current ARM1176 emulation
so I had to patch some QEMU files to add some of them.

However, I'll keep this README file up to date, with the latest news on this
project's progress.

================================================================================
*** Update 12/23/2012
================================================================================
Since the QEMU include structure has changed a few days ago, please use the
"rpi" branch I created instead.

https://github.com/Torlus/qemu/tree/rpi

================================================================================
About
================================================================================

This project aims at providing a basic chipset emulation of the Raspberry Pi,
to help low-level programmers or operating system developers in their tasks.

If you just want to test your favorite Linux distribution, there are easier 
ways, such as http://xecdesign.com/qemu-emulating-raspberry-pi-the-easy-way/

Emulated chipset parts are at the time of this writing:
- System Timer.
- UART.
- Mailbox system.
- Framebuffer interface.
- DMA.
- eMMC SD host controller.

The emulation is quite incomplete for many parts, however it is advanced enough
to boot a Pi-targetted Linux kernel, along with a SD image of a compatible
Linux distribution.
I've successfully tested it with the latest official Raspbian wheezy image.

As it is at its current stage some very "alpha" software, which probably doesn't
meet QEMU project's requirements, I haven't submitted it as a QEMU patch yet.

================================================================================
Installation instructions
================================================================================

Preparing QEMU:
- Make sure you can compile and run QEMU according to the instructions provided
  by http://xecdesign.com/compiling-qemu/
- Use https://github.com/Torlus/qemu/tree/rpi instead of the official QEMU
  repository. 
- Recompile and reinstall QEMU.

================================================================================
Running Linux
================================================================================

From a working SD image, extract the kernel image from the FAT32 partition.
On Raspbian wheezy SD image, it is the "kernel.img" file.

Now run QEMU using the following command (warning, long line) :

/path/to/qemu-system-arm -kernel kernel.img -cpu arm1176 -m 512 -M raspi -serial stdio -append "rw dma.dmachans=0x7f35 bcm2708_fb.fbwidth=1024 bcm2708_fb.fbheight=768 bcm2708.boardrev=0xf bcm2708.serial=0xcad0eedf smsc95xx.macaddr=B8:27:EB:D0:EE:DF sdhci-bcm2708.emmc_clock_freq=100000000 vc_mem.mem_base=0x1c000000 vc_mem.mem_size=0x20000000 dwc_otg.lpm_enable=0 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 elevator=deadline rootwait" -snapshot -sd 2012-10-28-wheezy-raspbian.img -d guest_errors

It should boot up normally, with a few additional error messages.
After booting it up, you can use the stdio console, but not the framebuffer's 
one, as there is no USB emulation yet (for keyboard and mouse support). 

Here are some explanations about the parameters provided to QEMU :
- "-m 512"
  defines the memory size, which corresponds in this case to a Model B.
- "-d guest_errors"
  logs the (considered) incorrect memory-mapped I/O access to /tmp/qemu.log  
- "-snapshot"
  commits write operations to temporary files instead of the SD image, which
  is probably wise, considering the current status of the emulation. :) 
  
Here are some explanations about the parameters provided to the Linux kernel :
- Most of them correspond to what is passed to the Linux kernel by the
  bootloader. You can retreive them with the "dmesg" command.
- "vc_mem.mem_base=0x1c000000 vc_mem.mem_size=0x20000000"
  defines the "memory split" between the ARM and the VideoCore.
  If you want to change it, you will have to change the VCRAM_SIZE value
  defined in bcm2835_common.h accordingly, otherwise you will encounter
  kernel memory corruption issues.
- "rw"
  forces the kernel to mount the root filesystem in read-write mode. 
  I have yet to find out why a "prepared" kernel mounts the root filesystem as
  read-write when booting on the hardware, whereas the same "unprepared" kernel 
  booting in QEMU mounts the root filesystem as read-only.
  I though it was due to different ARM boot tags settings, but it doesn't seem
  to be the reason why. If someone has an explanation, I'd be glad to hear it. :)

================================================================================
Running RiscOS
================================================================================

- Read the "Running Linux" section.
- Extract the RISCOS.IMG from the FAT32 partition of your SD card.

Now run QEMU using the following command (warning, long line) :
/path/to/qemu-system-arm -kernel RISCOS.IMG -initrd RISCOS.IMG -cpu arm1176 -m 512 -M raspi -snapshot -sd ro519-rc6-1876M.img -d guest_errors -serial stdio

You may have noticed the weird "-kernel" and "-initrd" stuff, both pointing to
the same file. This is a quick and dirty hack to activate the bootloader
emulation. If you do this with Linux, it won't boot as the "-append" parameters
won't be passed to the kernel, which requires them.

================================================================================
Gregory Estrade, 01/05/2013

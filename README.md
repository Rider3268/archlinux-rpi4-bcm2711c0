# archlinux-rpi4-bcm2711c0
Arch linux installation for Raspberry Pi 4 with bcm2711c0 SoC

problem >>
After installing image from archlinuxarm.org by following provided instructions, user may have an error:
```
mmc1: unrecognised SCR structure version 4
```
Booting from Raspbian works just fine.

solution #1 (not recommended) >> 
<details>
<summary>Details</summary>
replace arch linux kernel by raspbian kernel (this means no more kernel modularity, as you couldn't just replace it later from system, ex. `pacman -Syy linux-lts`, no wlan interface out of the box, and to make story short it'll we a HUGE pain in a butt. I've tried this method first as well).
</details>

solution #2 >> 
<details>
<summary>With more explanation</summary>
First check your processor revision (ex. cat /proc/cpuinfo), look through these lines:
```
Hardware	: BCM2835
Revision	: c03114
Serial		: 10000000e5e65676
Model		: Raspberry Pi 4 Model B Rev 1.4
```

Or if this doesn't work, try to run ```od -An -tx1 /proc/device-tree/emmc2bus/dma-ranges```, and it will give you different output depending on the stepping:
```
# B0
pi@raspberrypi:~$ od -An -tx1 /proc/device-tree/emmc2bus/dma-ranges
 00 00 00 00 c0 00 00 00 00 00 00 00 00 00 00 00
 40 00 00 00

# C0
pi@raspberrypi:~$ od -An -tx1 /proc/device-tree/emmc2bus/dma-ranges
 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
 fc 00 00 00
```
We need to make sure the board rev is 1.4 and stepping actually is "**c0**".
For me at some point cpuinfo shows SoC name as "BCM2835", but it doesn't matter, as according to official RPI [specs](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/specifications/) we know the SoC is BCM2711.

After some internet research i found this [link](https://bugzilla.redhat.com/show_bug.cgi?id=2011423).
It clearly shows it's neither os, nor kernel problem.

Let's look through rpi4 [boot sequence](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html).
Here's the key thesis:
- 	"The BCM2711 ROM does not support loading recovery.bin from USB mass storage or TFTP. 
	Instead, newer versions of the bootloader support a self-update mechanism where the bootloader is able to reflash the EEPROM itself."
-	"Raspberry Pi 4, 400 and Compute Module 4 computers use an EEPROM to boot the system. 
	All other models of Raspberry Pi computer use the bootcode.bin file located in the boot filesystem."

RPI uses _it's own proprietary bootloader_, while arch linux - _opensource "u-boot" bootloader_, so we need to **load u-boot from native RPI bootloader**.
Here's the [theoretical material](https://hechao.li/2021/12/20/Boot-Raspberry-Pi-4-Using-uboot-and-Initramfs/) for better understanding.


Now let's look through a boot partition of fresh installed arch linux on rpi.
We can see a lot of files here (**DO NOT TOUCH "dtbs" FOLDER!**). The one responsible for loading initram and kernel images is _boot.scr_ but it's a **precompiled script, which source is located in _boot.txt_ file**.
According to the article mentioned above, p. 7.1.3, we need to **replace _fdt_addr_r_ variables for _fdt_addr_ in _boot.txt_** and **compile it** (make sure u-boot-tools package is installed, _for debian-based distros installation command will be `apt install u-boot-tools`_):

`mkimage -A arm -O linux -T script -C none -n "U-boot boot script" -d <source_file_location> <output_file>`

ex. (assuming we're currently inside rpi _boot partition_.):

`mkimage -A arm -O linux -T script -C none -n "U-boot boot script" -d boot.txt boot.scr`

If you've installed _aarch64 version_ (assuming that you installed arch by archlinuxarm.org instruction), you need to **rollback all changes made in /etc/fstab file**:
(assuming that system is mounted on _root_ directory as mentioned in instruction:

`sed -i 's/mmcblk1/mmcblk0/g' root/etc/fstab`

P.s. Yeah, this actually is **_unnecessary step, as "mmcblk1" bug seems like fixed on the new rev rpi4 with bcm2711c0, so this step breaks things even more_**.
If you did all the things right, now you can successfully boot arch os with arch kernel.
</details>
	
<details>
<summary>Getting straight to the point</summary>
1. Check processor stepping: "od -An -tx1 /proc/device-tree/emmc2bus/dma-ranges"

	 "00 00 00 00 c0 00 00 00 00 00 00 00 00 00 00 00
	 40 00 00 00" -> B0 stepping.

	 "00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
	 fc 00 00 00" -> C0 stepping.
2. If it's C0 we need to modify bootloader script inside sdcard boot partition.

	Open _boot.txt_ file and **replace _fdt_addr_r_ variables for _fdt_addr_**

3. Now we need to compile the script using u-boot-tools:
	`mkimage -A arm -O linux -T script -C none -n "U-boot boot script" -d boot.txt boot.scr`

4. **If you've modified fstab file**
	("sed -i 's/mmcblk0/mmcblk1/g' root/etc/fstab" according to archlinuxarm.org instruction), undo all changes:
	ex. `sed -i 's/mmcblk1/mmcblk0/g' root/etc/fstab`
</details>

solution #3 (almost identical to archlinuxarm.org instructions) >> 
<details>
I'll add this solution later :)
</details>

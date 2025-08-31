What is kernel1.bin? A Practical Example

The term kernel1.bin is not a universal standard name but a convention used by some OpenWrt hardware targets and, most importantly, by the Breed bootloader.

In this context, kernel1.bin refers to the kernel part of the OpenWrt firmware. In many router architectures (especially those using MediaTek/MIPS chips like MT762x), the firmware is often split into two parts:

1. The Kernel (kernel1.bin): Contains the Linux kernel and essential drivers.
2. The Root Filesystem (rootfs0.bin): Contains the operating system itself, packages, and configuration files.

Breed expects these two parts to be flashed at specific offsets in the router's flash memory. This script builds a single file that respects those offsets.

---

Example: Finding kernel1.bin and rootfs0.bin

Let's take a very common router model, the Xiaomi Mi Router 3G (MIR3G), which uses a MediaTek MT7621 chipset.

1. Go to the OpenWrt download page for the target:
   · The directory structure is: https://downloads.openwrt.org/releases/<version>/targets/<target-subtree>/
   · For the MIR3G, the target is ramips/mt7621.
   · So for OpenWrt 23.05.3, the files are here: https://downloads.openwrt.org/releases/23.05.3/targets/ramips/mt7621/
2. Look for the files: In that directory, you will find several files. The ones you are interested in are:
   · openwrt-23.05.3-ramips-mt7621-xiaomi_mir3g-squashfs-kernel1.bin <- This is your kernel1.bin
   · openwrt-23.05.3-ramips-mt7621-xiaomi_mir3g-squashfs-rootfs0.bin <- This is your rootfs0.bin
   You will also see a sysupgrade.bin file. This is a combined image used for standard OpenWrt sysupgrade, but Breed often cannot handle this format directly. That's why this script exists—to create a Breed-compatible image from the split parts.

Summary for this example:

· kernel1 = openwrt-23.05.3-ramips-mt7621-xiaomi_mir3g-squashfs-kernel1.bin
· rootfs0 = openwrt-23.05.3-ramips-mt7621-xiaomi_mir3g-squashfs-rootfs0.bin

Command you would run:

```bash
./build.sh openwrt-23.05.3-ramips-mt7621-xiaomi_mir3g-squashfs-kernel1.bin openwrt-23.05.3-ramips-mt7621-xiaomi_mir3g-squashfs-rootfs0.bin
```

The script would then output a file called openwrt.bin that you can flash directly through the Breed bootloader's web interface.

---

How the Script Works (Step-by-Step Breakdown)

Let's walk through the script's logic using our example filenames.

1. Parameter Check:
   ```bash
   kernel1="openwrt-23.05.3-ramips-mt7621-xiaomi_mir3g-squashfs-kernel1.bin"
   rootfs0="openwrt-23.05.3-ramips-mt7621-xiaomi_mir3g-squashfs-rootfs0.bin"
   ```
   The script checks that these two files exist and that the kernel is less than 4MB.
2. Create a Blank 8MB Image:
   ```bash
   dd if=/dev/zero of="$image" bs=4M count=2
   ```
   This creates a 8MB file filled with zeros. This size is common for routers with 16MB of flash, reserving 8MB for the firmware and 8MB for Breed and other data.
3. Write the Kernel Twice (at two 4MB blocks):
   ```bash
   dd if="$kernel1" of="$image" bs=4M seek=0 conv=notrunc
   dd if="$kernel1" of="$image" bs=4M seek=1 conv=notrunc
   ```
   · seek=0 writes the kernel to the first 4MB block (offset 0).
   · seek=1 writes the same kernel to the second 4MB block (offset 4MB).
   · conv=notrunc ensures the main image file is not truncated; these writes just replace a section of it.
   · Why twice? This is a safety feature for Breed. It provides a backup kernel. If the first kernel gets corrupted during an update or due to bad blocks, the bootloader can automatically fall back to the second, known-good kernel copy, preventing a brick.
4. Write the Root Filesystem:
   ```bash
   dd if="$rootfs0" of="$image" bs=4M seek=2
   ```
   This writes the root filesystem to the third 4MB block (offset 8MB). The final structure of the openwrt.bin file looks like this:

Offset in Flash Contents Size
0x0000000 Kernel (primary copy) 4 MB
0x0400000 Kernel (backup copy) 4 MB
0x0800000 Root Filesystem 4 MB
0x0C00000 (End of file / Reserved) 4 MB

1. Final Output: The script renames the temporary file to openwrt.bin, which is now ready to be flashed via Breed.

In essence, this script is a firmware image assembler that takes the official OpenWrt kernel and rootfs components and packages them into a layout that the closed-source Breed bootloader expects and understands.

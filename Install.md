# Follow steps:

## 1. Create a Disk Image
Ensure `qemu` pkg is installed; install it if does not exist. Create a raw disk image of preferable size (a 100G in this case).
```bash
qemu-img create -f raw void-linux-disk.img 100G
```

## 2. Attach the Disk to a Loop Device
Attach the disk image to a loop device for partitioning:
```bash
sudo losetup -fP void-linux-disk.img
```

## 3. Verify Loop Devices
Check the available loop devices (optional):
```bash
sudo losetup -a
```
Example output:
```
/dev/loop0: [64771]:2501489 ($PATH/void-linux-disk.img)
```

## 4. Partition the Disk
Use `cfdisk or fdisk` to partition the disk, I'll use `fdisk`:
```bash
sudo fdisk /dev/loop0
```
- Create a GPT partition table: `g`
- Create a boot partition:
  - `n` (new partition), `1` (partition number)
  - First sector: `Enter`
  - Last sector: `+1G` (or less if desired for boot partition)
  - Change type to EFI system: `t`, `1` (you can press L to list all type and choose EFI system which is the norm now in most modern computers, note this step will be different if you’re using Bios)
- Create a root partition:
  - `n` (new partition), `2` (partition number)
  - Use default first and last sectors. (you can make a swap partition if you like but I’m not a fan!)
- Save changes: `w`

## 5. Verify Partitions (optional)
Check the created partitions:
```bash
sudo fdisk -l /dev/loop0
```
Expected output:
```
Device         Start       End   Sectors Size Type
/dev/loop0p1    2048   2099199   2097152   1G EFI System
/dev/loop0p2 2099200 209713151 207613952  99G Linux filesystem
```

## 6. Format Partitions
Ensure `dosfstools` is installed, then format the partitions:
```bash
sudo mkfs.vfat -F 32 /dev/loop0p1
sudo mkfs.ext4 /dev/loop0p2
```

## 7. Mount Partitions
Mount the root partition:
```bash
sudo mount /dev/loop0p2 /mnt
```
Create and mount the boot directory:
```bash
sudo mkdir -p /mnt/boot/efi/
sudo mount /dev/loop0p1 /mnt/boot/efi/
```

## 8. Download Void Linux Root Filesystem
Download the root filesystem tarball using the command below or using the official website:
```bash
curl -O https://repo-default.voidlinux.org/live/current/void-aarch64-ROOTFS-20240314.tar.xz
```

## 9. Extract the Root Filesystem
Extract the tarball to the mounted system:
```bash
sudo tar -xvf void-aarch64-ROOTFS-20240314.tar.xz -C /mnt
```
Check the structure (optional):
```bash
ls /mnt
```

Expected output:
```
bin   dev  home  lib32	lost+found  mnt  proc  run   sys  usr
boot  etc  lib	lib64	media	   opt  root  sbin  tmp  var
```

## 10. Bind System Directories
Before `chroot`, bind the necessary directories (very important to avoid errors later):
```bash
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo mount --bind /run /mnt/run
```

## 11. Enter the Chroot Environment
Enter the new system:
```bash
sudo chroot /mnt /bin/bash
```
Test the internet connection:
```bash
ping google.com
```
If "name resolution error" occurs, exit to your main system and copy the `resolv.conf` file:
```bash
exit
sudo cp /etc/resolv.conf /mnt/etc/resolv.conf
sudo chroot /mnt /bin/bash
```

## 12. Update and Configure the System
Update the package manager and system:
```bash
xbps-install -Su xbps
xbps-install -u
xbps-install base-system
xbps-remove base-container-full
xbps-install -S vim
```
Set the hostname:
```bash
echo voidlinux > /etc/hostname
```
Configure locales (languages if needed for glibc):
```bash
vim /etc/default/libc-locales
# Uncomment desired locales, e.g.,
en_US.UTF-8 UTF-8
en_US ISO-8859-1
xbps-reconfigure -f glibc-locales
```
Set root password:
```bash
passwd
```

## 13. Configure fstab

Use `blkid` to find UUIDs for the 2 partitions we created earlier:
```bash
blkid
```

Copy and edit the `fstab` file (very important step):
```bash
cp /proc/mounts /etc/fstab
vim /etc/fstab
```
Now replace /dev/loop0p2 and /dev/loop0p1 with their rerspective UUIDs to guarantee they’ll be found even if their name change also remove unnecessary entries like all lines with proc, sys, devtmpfs, tmpfs (since we have no swap partition) and only keep the first two lones with loop0p2 and loop0p1:
```
UUID=<UUID of /dev/loop0p2> / ext4 rw,relatime 0 1
UUID=<UUID of /dev/loop0p1> /boot/efi vfat rw,relatime 0 2
```

## 14. Install GRUB Bootloader
Install and configure GRUB:
```bash
xbps-install grub-arm64-efi
grub-install --target=arm64-efi --efi-directory=/boot/efi --bootloader-id="Void"
```
If an error like (failed to register the boot entry mount the efivars filesystem) occurs and if efivars is not found then add --no-nvram to the end of the above command:
```bash
mount -t efivarfs none /sys/firmware/efi/efivars
grub-install --target=arm64-efi --efi-directory=/boot/efi --bootloader-id="Void" --no-nvram
```
Generate GRUB configuration:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
Now if you check for ls /boot/efi/EFI/Void/ : youshould find a file with name 'grubaa64.efi'

Ensure compliance with UEFI by creating boot directory:
```bash
mkdir -p /boot/efi/EFI/boot
cp /boot/efi/EFI/Void/grubaa64.efi /boot/efi/EFI/boot/bootaa64.efi
```

## 15. Install Kernel and Firmware
Now if we ls /boot we’ll only see efi and grub directories but we need a kernal (initramfs and firmware) to boot:
```bash
xbps-install -S linux-firmware linux
```
Now if we check ls /boot/ we’ll see files like vmlinux-* and initramfs-*
Reconfigure packages to esure all packages are configured properly:
```bash
xbps-reconfigure -fa
```

## 16. Exit and Unmount
Exit `chroot` and unmount directories in the following arrangement (note if target is busy use the lazy detach e.g. sudo umount -l /mnt/sys):
```bash
exit
sudo umount /mnt/dev
sudo umount /mnt/proc
sudo umount /mnt/sys
sudo umount /mnt/run
sudo umount /mnt/boot/efi/
sudo umount -R /mnt
```

## 17. Transfer Disk Image
Copy the disk image to another system using `scp` if on same network or use a USB drive:
```bash
sudo scp void-linux-disk.img user@$IP:/path/to/destination
```

## 18. Create a Virtual Machine
Use UTM to create a new virtual machine and import the `.img` file as a VirtIO disk image.

## 19. Network Configuration
Log in using the root credentials and start the DHCP for eth0 or whatever interface you see service if necessary:
```bash
sudo dhcpcd eth0
```

Now you have a minimal Void Linux system ready to build upon!

# VOID_LINUX_UTM/QEMU
Wondering How to boot glibc rootfs tarball void linux on aarch64 M series Macs?

This repo provides a step-by-step guide on how to install a minimal Void Linux system inside a QEMU virtual machine (using a loopback disk image).  
Follow along to create, partition, format, and configure a bootable Void Linux environment!

## Table of Contents

1. [Introduction](./docs/00-intro.md)
2. [Create a Disk Image](./docs/01-create-disk-image.md)
3. [Partition the Disk](./docs/02-partition-disk.md)
4. [Format and Mount Partitions](./docs/03-format-and-mount.md)
5. [Extract Void Linux RootFS](./docs/04-extract-rootfs.md)
6. [Chroot & Setup](./docs/05-chroot-and-setup.md)
7. [System Update & Packages](./docs/06-system-update-and-packages.md)
8. [Configure fstab](./docs/07-configure-fstab.md)
9. [Install GRUB](./docs/08-install-grub.md)
10. [Finishing Touches](./docs/09-finishing-touches.md)

## What You'll Need

- A linux system with **QEMU** pkg installed (for creating and running the disk image).
- Basic command-line knowledge.
- Internet connection (to download the rootfs tarball and nescessary pkgs).

## Contributing

Feel free to submit pull requests or open issues if you have suggestions or improvements!


## Credits

Created with ♥ by (https://github.com/cxelshal).


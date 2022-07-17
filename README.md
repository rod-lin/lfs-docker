lfs-docker
---

Build LFS using Docker. Based on [LFS 11.1-systemd](https://www.linuxfromscratch.org/lfs/view/11.1-systemd).

# Usage

First, [install Docker](https://docs.docker.com/get-docker/) on your system.
Then `cd` into this repo.

## If you are using Docker >= 18.09
```
DOCKER_BUILDKIT=1 docker build -o . .
```
An ISO image `lfs.iso` will be produced in the current directory

## If you are using Docker < 18.09
```
docker build -t lfs .
```

The final Docker image `lfs` contains a single bootable ISO image `/lfs.iso`.
You can extract it by running the image in a container and using `docker cp`.

On a laptop with 8th-gen low-power Intel CPUs and 16 GB of memory,
the entire build took 2 hours and 9 minutes to finish and used 15 GB of disk space.

# More details on the ISO image boot process

For booting the ISO image, I decided to use GRUB 2 with the
goal to support booting with both legacy BIOS and UEFI.

This requires GRUB 2.06 to be compiled with support for both `i386-pc` and `x86_64-efi` targets.

The final directory structure of the ISO image looks like this:
```
boot/
  - grub/
      - i386-pc/
          - eltorito.img # for BIOS booting
      - x86_64-efi/
          - efi.img      # for UEFI booting
      - grub.cfg         # GRUB configuration
  - vmlinuz              # Linux kernel
  - initramfs.cpio.gz    # initramfs
system.squashfs          # The actual LFS system packaged using squashfs
```

`eltorito.img` is the concatenation of `/usr/lib/grub/i386-pc/cdboot.img` and an image generated by `grub-mkimage`.

`efi.img` is an image formatted with FAT, and it used as the EFI System Partition (ESP).
When mounted, `efi.img` should contain a single file `/efi/boot/bootx64.efi`,
which is also generated by `grub-mkimage`.
See the Dockerfile for more details on how `grub-mkimage` is called.

When booted with BIOS, the machine will load `eltorito.img`,
which will run GRUB with the configuration file `grub.cfg`.

When booted with UEFI, the image `efi.img` will be loaded instead, and `/efi/boot/bootx64.efi`
within the image will be executed with a preloaded GRUB configuration (see `resources/grub-stub.cfg`).
This preloaded configuration will then search for the actual boot drive and use `grub.cfg` from there.

There is also some black magic to allow booting from, say, a USB flash drive, when the ISO image is written to it (e.g. `dd if=lfs.iso of=<USB drive>`).

Some useful resources:
- UEFI support in kernel: https://www.linuxfromscratch.org/blfs/view/11.1-systemd/postlfs/grub-setup.html#uefi-kernel
- Making initramfs: https://lyngvaer.no/log/create-linux-initramfs
- Making a UEFI bootable ISO image using GRUB: https://github.com/syzdek/efibootiso
- Making a UEFI + BIOS bootable ISO image using GRUB: https://opendev.org/airship/images/src/commit/5e55597fbcebc9e16006e06b7514b21b9882dc8d/debian-isogen/files/functions.sh
- GRUB modules: https://www.linux.org/threads/understanding-the-various-grub-modules.11142/
- Syslinux/isolinux usage: https://wiki.syslinux.org/wiki/index.php?title=ISOLINUX
- Making a "hybrid" image for syslinux: https://wiki.syslinux.org/wiki/index.php?title=Isohybrid#UEFI

# Related work

- https://github.com/reinterpretcat/lfs
- https://github.com/0rland/lfs-docker
- https://github.com/pbret/lfs-docker
- https://github.com/EvilFreelancer/docker-lfs-build

How to build Linux image which can mount f2fs filesystem with compression. So I want this script to be working
```
mkdir -p /mnt/sda2
mount -o compress_algorithm=zstd:1,compress_chksum,atgc,gc_merge,lazytime /dev/sda2 /mnt/sda2
```
# Details
## Ubuntu and f2fs
In Ubuntu 24.04.1 works good. First step I can make f2fs partition
```
# mkfs.f2fs -f -l mylable123 -i -O extra_attr,inode_checksum,sb_checksum,compression -e raw -E bin /dev/sdb

    F2FS-tools: mkfs.f2fs Ver: 1.16.0 (2023-04-11)

Info: Disable heap-based policy
Info: Debug level = 0
Info: Add new cold file extension list
Info: Add new hot file extension list
Info: Label = mylable123
Info: Trim is enabled
Info: Enable Compression
Info: [/dev/sdb] Disk Model: Flash Disk      
    /dev/sdb appears to contain a partition table (dos).
Info: Segments per section = 1
Info: Sections per zone = 1
Info: sector size = 512
Info: total sectors = 15974400 (7800 MB)
Info: zone aligned segment0 blkaddr: 512
Info: format version with
  "Linux version 6.8.0-48-generic (buildd@lcy02-amd64-010) (x86_64-linux-gnu-gcc-13 (Ubuntu 13.2.0-23ubuntu4) 13.2.0, GNU ld (GNU Binutils for Ubuntu) 2.42) #48-Ubuntu SMP PREEMPT_DYNAMIC Fri Sep 27 14:04:52 UTC 2024"
Info: [/dev/sdb] Discarding device
Info: This device doesn't support BLKSECDISCARD
Info: This device doesn't support BLKDISCARD
Info: Overprovision ratio = 1.710%
Info: Overprovision segments = 67 (GC reserved = 65)
Info: format successful
```
Second (and main for me) step - I can mount with compression:
```
mkdir -p /mnt/sda2
mount -o compress_algorithm=zstd:1,compress_chksum,atgc,gc_merge,lazytime /dev/sdb /mnt/sda2
umount /mnt/sda2
```

## Own kernel and f2fs
### Mainline 6.6
Let's try to make our own Linux image. First time I have tried to use kernel 6.6 config but with f2fs options enabled
```
CONFIG_F2FS_FS=y
CONFIG_F2FS_CHECK_FS=y
CONFIG_F2FS_FAULT_INJECTION=y
CONFIG_F2FS_FS_COMPRESSION=y
```
Fail:
```
# mount -o compress_algorithm=zstd:1,compress_chksum,atgc,gc_merge,lazytime /dev/sda2 /mnt/sda2
mount: mounting /dev/sda2 on /mnt/sda2 failed: Invalid argument
```
### Ubuntu's kernel 6.8
Then I taken `/boot/config-6.8.0-48-generic` kernel config from Ubuntu 24 where f2fs compression can works and try to use it. I have cut some parts relative to hardware drivers... Of course, it's good practice not to change anything, but I can't compile Ubuntu kernel as is due to a lot of build failures... Finally the same problem, log
```
# uname -a
Linux buildroot 6.8.0 #2 SMP PREEMPT_DYNAMIC Fri Nov  8 21:36:02 KST 2024 i686 GNU/Linux
# mkdir /mnt/sda2
# mount -o compress_algorithm=zstd:1,compress_chksum,atgc,gc_merge,lazytime /dev
/sdc /mnt/sda2/
[  582.682848] ext3: Unknown parameter 'compress_algorithm'
[  582.683882] ext2: Unknown parameter 'compress_algorithm'
[  582.684615] ext4: Unknown parameter 'compress_algorithm'
[  582.685248] squashfs: Unknown parameter 'compress_algorithm'
[  582.686661] fuseblk: Unknown parameter 'compress_algorithm'
mount: mounting /dev/sdc on /mnt/sda2/ failed: Invalid argument
```
# How to repeat
## Clone
```
git clone --remote-submodules --recurse-submodules -j8 https://github.com/AndreiCherniaev/buildroot_QEMU_f2fs_x86.git
cd buildroot_QEMU_f2fs_x86
```
## Make image
```
make clean -C buildroot
make BR2_EXTERNAL=$PWD/my_external_tree -C $PWD/buildroot f2fs_qemu_x86_defconfig
make -C buildroot
```
## Save non-default buildroot .config
To save non-default buildroot's buildroot/.config to $PWD/my_external_tree/configs/f2fs_qemu_x86_defconfig
```
make -C $PWD/buildroot savedefconfig BR2_DEFCONFIG=$PWD/my_external_tree/configs/f2fs_qemu_x86_defconfig
```
## Tune and rebuild Kernel
Tune Linux kernel
```
make linux-menuconfig -C buildroot
```
Rebuild kernel
```
make linux-dirclean -C buildroot && make linux-rebuild -C buildroot && make -C buildroot
```
## Save non-default Linux .config
In case of Buildroot to save non-default Linux's .config to my_external_tree/board/my_company/my_board/kernel.config
```
make -C $PWD/buildroot/ linux-update-defconfig BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE=$PWD/my_external_tree/board/my_company/my_board/kernel.config
```
## Save non-default BusyBox .config
```
make -C $PWD/buildroot/ busybox-update-config BR2_PACKAGE_BUSYBOX_CONFIG=$PWD/my_external_tree/board/my_company/my_board/MyBusyBox.config
```
## Start in QEMU
This code is based on [emulation script](https://github.com/buildroot/buildroot/tree/master/board/qemu/x86_64), run the emulation with:
```
qemu-system-i386 -M pc -kernel buildroot/output/images/bzImage -drive file=buildroot/output/images/rootfs.ext2,if=virtio,format=raw -append "rootwait root=/dev/vda console=tty1 console=ttyS0" -serial stdio -net nic,model=virtio -net user
```
Login is `root`. Unfortunatly in my machine there is no blockdev which I can try to mount so mount will fails because can't lookup /dev/sda2. So I am ready for such error, but looks like f2fs code doesn't trig when I try mount... Let's check again using steps from "Ubuntu's kernel 6.8":
```
# uname -a
Linux buildroot 6.8.0 #2 SMP PREEMPT_DYNAMIC Fri Nov  8 21:36:02 KST 2024 i686 GNU/Linux
# modinfo f2fs
filename:       /lib/modules/6.8.0/kernel/fs/f2fs/f2fs.ko
author:         Samsung Electronics's Praesto Team
description:    Flash Friendly File System
license:        GPL
parm:           num_compress_pages:Number of intermediate compress pages to preallocate
alias:          fs-f2fs
depends:        zstd_compress,lz4_compress,lz4hc_compress
intree:         Y
vermagic:       6.8.0 SMP preempt modversions 686

# mkdir /mnt/sda2
# mount -o compress_algorithm=zstd:1,compress_chksum,atgc,gc_merge,lazytime /dev
/sdc /mnt/sda2/
[  582.682848] ext3: Unknown parameter 'compress_algorithm'
[  582.683882] ext2: Unknown parameter 'compress_algorithm'
[  582.684615] ext4: Unknown parameter 'compress_algorithm'
[  582.685248] squashfs: Unknown parameter 'compress_algorithm'
[  582.686661] fuseblk: Unknown parameter 'compress_algorithm'
mount: mounting /dev/sdc on /mnt/sda2/ failed: Invalid argument
```

## Steps
Based on "boot/grub2/readme.txt"
1. Create a disk image
```
cd /tmp
dd if=/dev/zero of=disk.img bs=1M count=128
```
2. Partition it with GPT partitions usinig `cgdisk disk.img` or
```
parted --script disk.img mklabel gpt mkpart primary 1MiB 127MiB
```
3. Setup loop device and loop partitions
```
loop_dev=$(sudo losetup -f --show disk.img)
sudo partx -a "$loop_dev"
```
5. Prepare the root partition
```
sudo mkfs.f2fs -f -l mylable123 -i -O extra_attr,inode_checksum,sb_checksum,compression -e raw -E bin "$loop_dev"
```
Another example `sudo mkfs.ext3 -L root "$loop_dev"p1`
6. Cleanup loop device
```
partx -d "$loop_dev"
losetup -d "$loop_dev"
```

## Build mkfs.f2fs from source code
To avoid extra dialog from gdb...
```
cat <<EOF > ~/.gdbinit
set debuginfod enabled on
EOF
```
Build
```
sudo apt install git uuid-dev libselinux1-dev libtool dh-autoreconf
git clone https://git.kernel.org/pub/scm/linux/kernel/git/jaegeuk/f2fs-tools.git && cd "f2fs-tools"
./autogen.sh
./configure
make -j$(( $(nproc) + 1))
```
Start
```
"$HOME/mycode/f2fs-tools/mkfs/mkfs.f2fs" -V
```
Start debug
```
libtool --mode=execute gdb -ex=r --args "$HOME/mycode/f2fs-tools/mkfs/mkfs.f2fs" -f -l mylable123 -i -O extra_attr,inode_checksum,sb_checksum,compression -e raw -E bin "$loop_dev"
```

How to build Linux image based on qemu_x86_defconfig but with f2fs filesystem.
## Compare with qemu_x86_defconfig
Config [f2fs_qemu_x86_defconfig](my_external_tree/configs/f2fs_qemu_x86_defconfig) is based on [qemu_x86_defconfig](https://github.com/buildroot/buildroot/blob/e82217622ea4778148de82a4b77972940b5e9a9e/configs/qemu_x86_defconfig). f2fs is enabled in Linux kernel. f2fs is enabled in GNU GRUB (BR2_TARGET_GRUB2_BUILTIN_MODULES_PC now contains f2fs). grub's timeout is 0 so you willn't see [him](https://upload.wikimedia.org/wikipedia/commons/thumb/8/81/Grub_logo_large.png/240px-Grub_logo_large.png).

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
## Start in QEMU
This code is based on [emulation script](https://github.com/buildroot/buildroot/tree/master/board/qemu/x86_64), run the emulation with:
```
qemu-system-i386 -M pc -kernel buildroot/output/images/bzImage -drive file=buildroot/output/images/rootfs.ext2,if=virtio,format=raw -append "rootwait root=/dev/vda console=tty1 console=ttyS0" -serial stdio -net nic,model=virtio -net user
```
Optionally add `-nographic` to see output not in extra screen but in console terminal. Or `-display curses` to pseudographic. 

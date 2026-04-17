# 我的 Linux 启动修复经历 - 2025-7-7

> 💡 背景  
> 默认情况下，安装操作系统时会在磁盘特定位置创建一个 EFI 分区。UEFI/BIOS 使用这个分区来正确启动系统。但如果你在同一块磁盘上安装另一个操作系统（比如 Windows），新的安装程序可能会覆盖原有的 EFI 分区，导致之前的系统无法启动。我就遇到了这种情况 —— Windows 把我的 Linux 引导加载程序覆盖了。

## 1. 准备修复

我没有现成的 Linux 安装介质，所以先从官网下载了我用的发行版 ISO 镜像，然后用刻录软件写入 U 盘。接着进入 UEFI/BIOS 设置，**禁用 Secure Boot 和 Fast Boot**，保存重启，从 U 盘启动到 Live 环境。

> 💡 如果使用 Ventoy 这类多系统安装介质启动时卡住，可以试试 GRUB 引导模式。

## 2. 挂载分区

在 Live 环境中，我先用 `lsblk` 和 `fdisk -l` 查看磁盘分区情况，确认哪个是 Linux 根分区、哪个是 EFI 分区（EFI 分区通常 100~500 MB，文件系统类型为 vfat）。下图是 DiskGenius 中看到的分区布局：

![DiskGenius 中的分区布局](../src/img/image.png)

然后依次挂载：

```bash
mount /dev/nvme0n1p5 /mnt           # 挂载根分区（以我的实际分区为例）
mount /dev/nvme0n1p1 /mnt/boot/efi  # 挂载 EFI 分区
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
mount --bind /run /mnt/run
```

（我并没有独立的 `/home` 或 `/boot` 分区，所以省略了它们. 但我事后再次经历这种引导问题，尤其是后面我手动一步一步的安装 *Arch Linux* 后，有了进一步理解,其实在Arch上是不需要挂载这么多, 默认是会自动帮你挂载 /dev，/proc，/sys，/run，这些必需目录的）

## 3. 进入 chroot 并修复 GRUB

```bash
arch-chroot /mnt   # 我用的 Arch Linux，所以用这个命令
# 确认引导模式
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"   # 输出 UEFI
# 重新安装 GRUB
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
# 生成配置
grub-mkconfig -o /boot/grub/grub.cfg
```

退出 chroot 并重启：

```bash
exit
reboot
```

## 4. 修复后遇到的问题

重启后，我并没有直接进入 Linux —— 先是遇到了两个常见问题。

### 问题一：只进入 GRUB shell

屏幕上出现 `grub>` 提示符，没有菜单。  
**解决方法**：重新生成 GRUB 配置。我再次启动 Live 环境，挂载分区并 chroot，然后执行 `grub-mkconfig -o /boot/grub/grub.cfg`。

### 问题二：修复问题一后，只出现 Windows Boot Manager

重启后直接进了 Windows，看不到 Linux 选项。  
**原因**：`/boot` 目录下缺少 Linux 内核文件（`vmlinuz-linux` 和 `initramfs` 镜像）。  
**解决方法**：在 Live 环境中挂载分区后，检查 `/boot` 和 `/boot/efi`，发现内核文件实际存在于 `/boot/efi` 中。我把它们复制到 `/boot` 目录，然后重新 chroot 并生成 GRUB 配置。

```bash
cp /mnt/boot/efi/vmlinuz-linux /mnt/boot/
cp /mnt/boot/efi/initramfs-linux.img /mnt/boot/
# 再次 chroot 并运行 grub-mkconfig
```

完成这两步后，GRUB 菜单终于正常显示了 Linux 和 Windows 两个选项。

## 5. 最后的内核报错（二维码倒计时自动修复）

我满怀期待地选择 Linux 启动，结果出现报错，![如图所示](../src/img/kenerl.png)屏幕中间有一个**二维码**，。

然后回车重启。这一次，Linux 顺利启动，进入了桌面！

事后我猜测，这个二维码可能是我的发行版（或者 systemd）内置的错误报告/自动修复机制，倒计时结束后系统尝试了应急恢复并成功修复了内核或 initramfs 的问题。总之，省去了我手动排查的麻烦。

## 6. 总结

从 Windows 覆盖引导，到 GRUB shell、缺少内核，再到内核报错二维码自动修复 —— 整个过程虽然曲折，但按照标准步骤一步步来，最终都解决了。如果你也遇到类似问题，希望我的经历能给你一些参考。
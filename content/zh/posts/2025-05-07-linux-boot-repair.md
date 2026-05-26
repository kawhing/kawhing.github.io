+++
date = '2025-05-07'
draft = false
title = '我的 Linux 启动修复经历'
+++

> 💡 背景知识
> 默认情况下，当你安装操作系统时，系统会在磁盘的特定位置创建一个 EFI 分区。UEFI/BIOS正是利用该分区来正确引导系统启动的。然而，如果你在同一块磁盘上安装了另一个操作系统（例如 Windows），新的安装程序可能会覆盖原有的 EFI 分区，从而导致之前的操作系统无法启动。我就遇到了这种情况——Windows 覆盖了我的 Linux 引导加载程序（Bootloader）。

## 1. 修复前的准备工作

当时我手头没有现成的 Linux 安装介质，于是我从官方网站下载了我所用发行版的 ISO 镜像文件，并将其写入到了一个 USB 闪存盘中。随后，我进入了 UEFI/BIOS 设置界面，**禁用了“安全启动”（Secure Boot）和“快速启动”（Fast Boot）**功能，保存设置并重启电脑，最终通过 USB 闪存盘成功引导进入了 Live 环境。

> 💡 如果你使用的是像 Ventoy 这样的多系统引导工具，且在正常引导模式下遇到了卡死的情况，不妨尝试切换到 GRUB 引导模式。

## 2. 挂载分区

进入 Live 环境后，我使用 `lsblk` 和 `fdisk -l` 命令来检查磁盘分区情况，以此确定哪一个是我 Linux 系统的根分区（Root Partition），哪一个又是 EFI 分区（通常大小在 100 至 500 MB 之间，且文件系统类型为 vfat）。下图展示了在 DiskGenius 中看到的磁盘分区布局：

![DiskGenius 中的分区布局](/images/2025-05-07-1.png)

接着，我挂载了这些分区：

```bash
mount /dev/nvme0n1p5 /mnt           # 挂载根分区（请根据你的实际配置进行调整）
mount /dev/nvme0n1p1 /mnt/boot/efi  # 挂载 EFI 分区
mount --bind /dev /mnt/dev
mount --bind /proc /mnt/proc
mount --bind /sys /mnt/sys
mount --bind /run /mnt/run
```

（我没有单独的 `/home` 或 `/boot` 分区，所以我跳过了它们。不过，后来再次遇到这种启动故障——特别是在一步步手动安装 Arch Linux 之后——我对这一过程有了更深入的理解。实际上，在 Arch Linux 中，你无需挂载如此多的目录；像 `/dev`、`/proc`、`/sys` 和 `/run` 这些必要的目录，系统会自动为你挂载。）

## 3. Chroot 与 GRUB 修复

```bash
arch-chroot /mnt   # 我使用的是 Arch Linux，因此使用此命令
# 确认启动模式
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"   # 输出：UEFI
# 重新安装 GRUB
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
# 生成配置文件
grub-mkconfig -o /boot/grub/grub.cfg
```

退出 chroot 环境并重启：

```bash
exit
reboot
```

## 4. 修复后遇到的问题

重启之后，我并没有直接进入 Linux 系统——而是遇到了两个常见的问题。

### 问题 1：仅显示 GRUB Shell

屏幕上只显示了 `grub>` 提示符，并没有出现启动菜单。
**解决方案**：重新生成 GRUB 配置文件。我再次进入 Live 环境，挂载分区，执行 chroot 操作，然后运行了 `grub-mkconfig -o /boot/grub/grub.cfg` 命令。 

### 问题 2：修复问题 1 后，仅显示 Windows 启动管理器

重启后，系统直接进入了 Windows；完全看不到 Linux 的启动选项。
**原因**：`/boot` 目录中缺失了 Linux 内核文件（即 `vmlinuz-linux` 和 `initramfs` 镜像）。
**解决方案**：在 Live 环境下，挂载分区后，我检查了 `/boot` 和 `/boot/efi` 目录。结果发现，内核文件实际上位于 `/boot/efi` 目录中。于是，我将这些文件复制到了 `/boot` 目录，随后再次通过 `chroot` 进入系统环境并运行了 `grub-mkconfig` 命令。

```bash
cp /mnt/boot/efi/vmlinuz-linux /mnt/boot/
cp /mnt/boot/efi/initramfs-linux.img /mnt/boot/
# 再次 chroot 进入系统环境，并运行 grub-mkconfig
```

完成上述两个步骤后，GRUB 启动菜单终于同时显示出了 Linux 和 Windows 的启动项。

## 5. 最后的内核崩溃（附带二维码自动修复功能）

我迫不及待地选择了 Linux 启动，但屏幕上随即弹出了一个错误提示，如下图所示。屏幕中央赫然显示着一个**二维码**。

![错误截图](/images/2025-05-07-2.png)

随后，我按下了回车键（Enter）进行重启。这一次，Linux 成功启动，我也顺利进入了桌面环境！

事后我推测，那个二维码应该是我所使用的 Linux 发行版（或者是 systemd）内置的错误报告/自动修复机制的一部分。在倒计时结束（或我按下回车键）之后，系统尝试执行了一次紧急恢复操作，并成功修复了内核或 initramfs 方面的问题。无论如何，这一机制让我免去了手动排查故障的繁琐过程。

## 6. 结语

从 Windows 覆盖启动引导程序（bootloader），到进入 GRUB Shell，再到内核文件缺失，直至最后遭遇附带二维码自动修复功能的内核崩溃——整个过程可谓一波三折；但只要按部就班地遵循标准的修复步骤，最终还是解决了所有问题。如果您也遇到了类似的问题，希望我的这段经历能为您提供一些参考。
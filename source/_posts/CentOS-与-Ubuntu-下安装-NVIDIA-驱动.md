---
title: CentOS 与 Ubuntu 下安装 NVIDIA 驱动
date: 2013-03-07
updated: 2017-02-20 12:59:34
tags: Linux
---

2017-02-20 更新：旧文搬运，略加修缮，觉得还实用，所以保留下来。
2013-08-29 更新：Ubuntu 下安装方法，详见最下。

NVIDIA 驱动安装的重点在于关闭系统本身默认运行的 nouveau 模块。

首先上参考文章吧。[文章一][post1]还有[文章二][post2]。两篇文章的方法略有不同，我综合了一下。

<!-- more -->

去 Nvidia 官网下载对应的显卡驱动，我下载的驱动文件名为：NVIDIA-Linux-x86_64-310.32.run。

这部分操作大量使用 root 权限，所以最好还是使用 root 身份来进行。

第一步需要关闭 X Windows，运行：`init 3`。

如果现在运行驱动安装程序 `sh NVIDIA-Linux-x86_64-310.32.run`，有可能会提示你：

>ERROR: The Nouveau kernel driver is currently in use by your system. This  driver is incompatible with the NVIDIA driver, and must be disabled  before proceeding.

意思就是 nouveau 这个模块正在运行中，该模块与NVIDIA驱动不兼容，必须要被禁用才可以进行。为了禁用这个模块，大费周折，找了不少资料文章，总算试验出有效的方法。

过程中也看见一些传说，说是某些 Linux 发行版本禁用该模块也不算麻烦，但是 CentOS 似乎并不在那个阵营里。

**更新：添加 blacklist 或许没有什么影响，可以选择跳过这部分，直接到修改 `/etc/grub.conf` 文件部分。**

编辑 `/etc/modprobe.d/blacklist.conf`，在某处（但是不要选在注释里）加上 `blacklist nouveau`。

然后下面一步我估计可以选做：

```bash
* 备份 the initramfs file
mv /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r).img.bak
* 重新建立 the initramfs file
dracut -v /boot/initramfs-$(uname -r).img $(uname -r)
```

如果你运行了上面的两句命令，那请记好你曾经备份过 initramfs 这个文件。如果以后出错，可以还原试试。

在 CentOS 里似乎必须还需要下面这一步。因为我单独试了上面两步后并没有效果。

编辑 `/etc/grub.conf` 文件，禁止 nouveau KMS 的使用。

在这个文件里找到现在所在的系统项目，应该形如：

```bash
title CentOS (2.6.32-279.el6.x86_64)
        root (hd0,4)
        kernel /vmlinuz-2.6.32-279.el6.x86_64 ro root=UUID=55151a0d-b460-462b-b5c6-97dfd4a3d328 rd_NO_LUKS  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_MD LANG=zh_CN.UTF-8 rd_NO_LVM rd_NO_DM rhgb quiet
        initrd /initramfs-2.6.32-279.el6.x86_64.img
```

在 kernel 行的最后加上 `nouveau.modeset=0`。加上后的文件应该如下：

```bash
title CentOS (2.6.32-279.el6.x86_64)
        root (hd0,4)
        kernel /vmlinuz-2.6.32-279.el6.x86_64 ro root=UUID=55151a0d-b460-462b-b5c6-97dfd4a3d328 rd_NO_LUKS  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_MD LANG=zh_CN.UTF-8 rd_NO_LVM rd_NO_DM rhgb quiet nouveau.modeset=0
        initrd /initramfs-2.6.32-279.el6.x86_64.img
```

保存退出后重新启动，再进入文本模式。使用下面的命令来查看一下 nouveau 是否有被加载：

```bash
lsmod | grep nouveau
```

如果结果为空，那应该是成功了，可以接下去进行驱动的安装。驱动的安装倒是很简单，我就不打算写一遍了。详细的可以参考[这一篇文章][psot3]。

---

更新 Ubuntu 下安装方法：

在 Ubuntu 下安装驱动的方法步骤还是与在 CentOS 中差不多。

在引导中加入 `nouveau.modeset=0` 禁用 nouveau（加不加blacklist关系不大），然后在 CLI 环境下安装驱动。

Ubuntu 的引导文件位置与 CentOS 并不一样，具体在哪里我也忘了，请自行搜索解决吧。

禁用 nouveau 后，需要进入 CLI 环境，但是使用 `init 3` 命令并不能关闭 X Window，会导致安装无法继续进行。

解决办法是使用 Ctrl+Alt+F1 切换到文字界面下，然后将 dm 服务停止就可以关闭 X Window。Ubuntu 的默认 dm 是 lightdm，其他还有 kdm、gdm 等，请根据自己的情况来选择。

运行命令 `sudo service lightdm stop` 来停止X Window。接下来就可以按流程安装驱动了。

[post1]: http://enetq.blog.51cto.com/479739/591622
[post2]: http://www.ideasr.com/thread-33171-1-1.html
[psot3]: http://www.ideasr.com/forum.php?mod=viewthread&tid=7738&extra=page%3D1&page=1
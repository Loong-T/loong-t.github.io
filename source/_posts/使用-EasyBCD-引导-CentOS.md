---
title: 使用 EasyBCD 引导 CentOS
date: 2013-03-09
updated: 2017-02-20 15:38:51
tags: Linux
---

2017-02-20 更新：旧文搬运、修缮。我记得 Win8 开始，使用了比较棘手的引导系统，不知现在这方法是否还有效。

昨晚重启后忽然进不了 Windows，各种搜索加各种尝试后依旧无果，不得已只能使用 PE 修复 MBR 的引导。

这个办法是我尽力避免的，因为修复后会把 GRUB 覆盖掉，那么我就无法进入 Linux 系统了。这之后修复 Linux 的引导又是一番额外功夫。以前也算是碰见过类似的问题，当时用了 EasyBCD 这个软件来引导系统，所以这一次也立马想到了这个软件。不过还是碰见了不少问题，一上午才真正解决了这个问题。经验难得，需要记录一下。

<!-- more -->

以前用 EasyBCD 的时候，纯粹是乱折腾，多加几个不同的引导，直白地碰运气，问题也解决了。但这一次没有这么好的人品。这次安装系统的时候把 /boot 单独挂载到一个分区上，猜测这就是以前的方法不奏效的原因。

经过这一次的折腾，对系统的引导算是多了一些理解。其中各种曲折，各种重启，写一下正确的解决办法。

参考文献有：[百度文库的一篇][baidu wenku link]（这个是重点），[EasyBCD 官方文档][easybcd doc]，[GRUB 的百度百科][grub baike]。

---

安装好 EasyBCD 后，添加引导，选择 NeoGrub，安装，配置。这时候出现使用记事本打开的 menu.lst，这里要添加的就是关键了。

首先来看看官方给出的Ubuntu引导实例：

```bash
title		Ubuntu Gutsy Gibbon    
root		(hd1,2)    #Load Ubuntu from the 2nd harddrive's 3rd partition.
#Next Line: Translate (hd1,2) to Linux notation and set that as the root partition
kernel		/boot/vmlinuz-2.6.22-14-generic root=/dev/sdbc
initrd		/boot/initrd.img-2.6.22-14-generic
```

title 是引导系统的名字，自己写一个能辨认的就好。

root 这一行是装载指定的分区，如果装载的分区不正确，那么下面指定的文件自然就不能被找到，引导自然失败。root 后有一个空格，括号内是第几个硬盘的第几个分区。hd0 是第一块硬盘，0 是这一块硬盘的第一个分区，依次类推。这里需要装载的是 /boot 所在的分区。

kernel 行指定 Linux 的内核，位置在 /boot 下，名字一般是以 vmlinuz 开头的一个文件。如果 /boot 是单独挂载，位置应该类似：`/vmlinuz-2.6.22-14-generic`。

如果不知道内核的名称，重启进入 NeoGrub，按 c 进入命令行模式，使用 root 命令装载分区后可以使用 TAB 键列出文件或命令。请注意这个功能，下面的 initrd 文件也需要使用相同的方法来获得。内核名字后的 `root=***` 必不可少，我在这里栽了很久。有一些 Linux 下硬盘相关知识的应该不难理解这一句。不是很清楚的请参考[鸟哥的相关内容][bird related content]。`/dev/sd??` 这个其实指的就是根目录/所在的分区。

initrd 也是一个文件，与内核 vmlinuz 同在 /boot 下。名称可能是 initrd 开头的一个文件，但也可能是 initramfs 开头的一个 img 文件，在我的系统上就是如此。

在这之后可能还需要添加一句 boot 命令。

这里说明一下我的误区。因为我的 /boot 是单独挂载的，所以不能同时用 root 命令装载 `/` 和 `/boot`。让我对怎么指定 `root=` 后的根目录很伤脑筋。在我查看 GRUB 的百度百科的时候，学习到在加载了内核文件后，/boot 等就已经挂载到根目录下了。所以只需要使用`root=/dev/sd??` 这样的写法来指定就好了，而不必考虑自己在 GRUB 中装载的是哪一个分区。

下面是我成功引导的menu.lst文件，供参考：

```bash
# NeoSmart NeoGrub Bootloader Configuration File
#
# This is the NeoGrub configuration file, and should be located at C:\NST\menu.lst
# Please see the EasyBCD Documentation for information on how to create/modify entries:
# http://neosmart.net/wiki/display/EBCD/

default 0
timeout 8

title CentOS 6.3
    root (hd0,4)
    kernel /vmlinuz-2.6.32-279.el6.x86_64 root=/dev/sda8
    initrd /initramfs-2.6.32-279.el6.x86_64.img
    boot
```

[baidu wenku link]: http://wenku.baidu.com/view/2367b2d926fff705cc170a48.html "百度文库资料"
[easybcd doc]: http://neosmart.net/wiki/display/EBCD/NeoGrub "EasyBCD 官方文档"
[grub baike]: http://baike.baidu.com/view/225343.htm "Grub 百科"
[bird related content]: http://vbird.dic.ksu.edu.tw/linux_basic/0130designlinux_2.php "鸟哥的 Linux 私房菜"
# R36S 内存卡提示"需要格式化"——先别点，这篇讲怎么救

**[English](README.md)** · **中文**

> **一句话结论。** 在一张健康的 R36S 卡上，这个弹窗几乎总是冲着**那个 Linux（ext4）分区**来的，
> 而 Windows 根本没有 ext4 驱动。卡没坏，游戏也没丢。**点"取消"。**
> 一旦点了格式化，你要么毁掉引导分区，要么毁掉操作系统——而 ArkOS 自己的说明是：
> 那样就只能整卡重新刷镜像。
> 真正算"损坏"的只有两种情况：**分区表丢了**，或者**卡本身要坏/是假容量卡**。
> 这两种都**不是**靠格式化解决的，而是靠**先把整卡镜像成文件**，然后只在副本上动手。

这篇是完整流程：怎么区分这两种情况、各自怎么办、怎么把卡做成镜像和克隆，
以及哪些流传很广的工具**不要**用在这类镜像上。

---

## 一、群里到底发生了什么

以下原文来自中文掌机 QQ 群，**只标群名**——原始数据无法可靠地把某条消息归属到具体发言人。

> `[r36s（开源掌机）交流1群]` 我TF卡突然读不出来东西了，一直显示要格式化，插进掌机里也打不开，谁能救一救

> `[r36s（开源掌机）交流1群]` 大约17点多想导入个游戏，首先显示要格式化，插进掌机里也打不开，现在很急。

> `[R36S掌机爱好者]` 我搞了那么久的系统主题，游戏测试归类，弄了320个游戏，结果今早连手机后，掌机就不读卡了，机都开不起
> 具体情况是，手机连tf卡整理文件，结果突然就掉连接了，再重新连接就始终显示不出来了，当时想估计是坏菜了
> 只读得起boot分区 ／ roms分区的盘都读不出来
> 对了，踩坑提醒，尽量还是不要用otg转接读卡器来读tf卡
> 也就是尽量还是不要用手机读tf卡，用电脑更稳妥

> `[r36s（开源掌机）交流1群]` 系统在卡上你格式化了不就烷基八氮了吗

*——这句话是对的，值得抄在读卡器上。*

> `[r36s（开源掌机）交流1群]` 请问大家有没有克隆内存卡的方法呢，我的月光宝盒掌机32G内存卡用傲梅轻松备份克隆，得用64G内存卡才行，我刚买的R36S在备份内存卡，不知道128G内存卡能不能克隆

> `[r36s（开源掌机）交流1群]` 请问大家，不同商家的R36S掌机都是V30主板的掌机，游戏内存卡可以通用吗

> `[r36s（开源掌机）交流1群]` 这个掌机内存卡实际容量只有90G，国产卡
> `[r36s（开源掌机）交流1群]` 花点钱买个好点的内存卡，一卡玩三代

其中有两个问题值得正面回答，这篇就是围绕它们展开的：
**Windows 读不了的那个分区是 Linux 分区，不是游戏分区**；
以及**游戏卡是可以通用的，系统卡不行**——因为系统卡里装着内核和针对某一块特定主板的设备树。

---

## 二、为什么会提示格式化

### 2.1 卡上有三个分区，Windows 只读得了其中两个

![为什么 Windows 会劝你格式化 R36S 的卡，以及每个分区是什么](docs/01-what-windows-sees.png)

一张正常的 R36S 卡长这样。下面这张表来自 R36S dArkOS 项目对自己镜像的文档——
MBR 分区表、三个分区；我们用 `fdisk` / `sfdisk` 在一个同尺寸的测试文件上复现，
好让偏移量和类型码都能看见：

![实测终端输出：三分区表及其偏移量、大小和类型码](docs/04-partition-table-verified.png)

| 分区 | 起始扇区 | 镜像内大小 | 类型码 | 是什么 |
|---|---|---|---|---|
| p1 | 32768 | 100 MiB | `0c` W95 FAT32 (LBA) | `BOOT`：`Image`（内核）、`uInitrd`、`boot.ini`、`.dtb` 文件、机型选择脚本 |
| p2 | 237568 | 6.7 GiB | `83` Linux (ext4) | 操作系统 |
| p3 | 14368768 | 100 MiB 占位 | `07` HPFS/NTFS/exFAT | 卷标 `EASYROMS`，首次开机时长成整卡 |

于是：

- **p1 `BOOT`** 是 FAT32。Windows 读得了，它就是最先冒出来的那个小盘。
- **p2（ext4）** Windows 完全读不了。微软列出的 Windows 支持文件系统是
  *"NTFS / ExFAT / UDF / FAT32"*——ext4 不在里面；微软自己的 WSL 文档也写得很直白：
  *"If you have an Ext4-formatted drive, you cannot mount it on your Windows file system."*
  微软还定义了触发这个弹窗的状态：*"If a partition doesn't have a file system (it's listed as RAW
  instead of NTFS, ReFS, FAT32, or exFAT) …"*，并且针对这种情况给了它自己的警告：
  *"Formatting a disk erases all data on it, so don't do this step if you're trying to recover files
  from the disk."*
- **p3 `EASYROMS`** 是 exFAT，Windows **读得了**。如果它没出现，通常只是缺盘符，
  该做的是分配盘符，不是格式化。

### 2.2 为什么"没有盘符"这么常见

ArkOS 的文档专门写给 Windows 10 1703 及更新版本的用户：
如果你看不到一个卷标为 `EASYROMS` 的盘符，就打开磁盘管理，右键那个带这个卷标的分区，给它分配一个盘符。
一台电脑通常只需要做一次。dArkOS 那边为 R36S 准备了同一页说明，
并且点明可能从资源管理器里消失的就是 `BOOT` 和 `EASYROMS` 这两个文件系统。

顺带说一件相关的、而且流传很广的"偏方"：**`wsl --mount` 在这件事上没用。**
微软文档写得很明确：*"the `wsl --mount` command does not currently support USB/flash drives/SD card
readers"*，而且它需要管理员权限和 Windows 11（或商店版 WSL）。
任何让你"把 R36S 的卡挂进 WSL"的教程都是走不通的——何况在 R36S 上你根本不需要它，
因为装游戏的那个分区是 exFAT，Windows 原生就读得了。

### 2.3 点一下格式化，到底损失什么

> ArkOS 文档原文：*"Once the image has been completed, Windows may ask you if you want to format the SD
> card. **MAKE SURE TO CLICK CANCEL ON THIS QUESTION!** If you let it format the SD card, you will need to
> install the image again."*

那说的是刷机那一步，但每次弹窗都适用：

- 格式化 **p1（`BOOT`）**：丢掉 `Image`（内核）、`uInitrd`、`boot.ini`、`.dtb` 文件，
  以及镜像回写到 `/boot` 里的那几个隐藏状态文件——其中包括**已经持久化的主板/屏幕选择**。
  机器起不来了；而在需要屏幕专用设备树的机器上，结果通常就是一块黑屏。
  游戏还在卡上，但你得重新刷一次镜像才能用它们。
- 格式化 **p2（ext4）**：丢掉操作系统。这通常就是 Windows 正在抱怨的那个分区，
  也恰恰是它什么都显示不给你看的那一个——所以人们才会顺手点"确定"。
- 格式化 **p3（`EASYROMS`）**：丢掉游戏、`roms` 目录结构，以及被 bind-mount 进系统的 `roms/tools`
  目录（PortMaster 等机内工具都在那里）。这一项是三个里唯一还能扛过去的——操作系统照样能启动。
  但**不要删掉本来就有的目录**：ArkOS 专门警告过，PSP 和 NDS 那两个目录在你放进第一个游戏之前，
  就已经有对应模拟器需要的文件在里面。

在一台"操作系统就住在同一张卡上"的设备上，格式化不是"把卡恢复到已知良好状态"的按钮。

### 2.4 只有两种情况是真坏了

![决策树：在让任何东西格式化这张卡之前，先确认什么](docs/02-decision-tree.png)

1. **分区表没了。这里可以直接借用微软自己的用词来区分两种情况**：
   分区显示为 **`RAW`**，意思是 Windows 看得见这个分区、但读不懂它的文件系统——
   这正是 Linux 分区该有的样子；而磁盘显示为 **"Not Initialized"**（微软的解释是
   *"the drive formatting has become corrupted somehow"*）或 **"Unreadable"**，
   意味着 Windows 连分区结构都理不清了。**后两种才是损坏或硬件故障，不是 Linux 文件系统。**
   这两种情况下 Windows 都会很乐意"帮你修"——建一个大空分区，而那正好会覆盖掉仍然指向你数据的分区表。
   **TestDisk** 就是干这个的：它重建丢失的分区表、从备份里恢复 FAT 引导扇区，
   还能定位 ext2/3/4 的备份超级块。
2. **卡正在坏、是假卡，或者被手机重新格式化过。** 群里那位用手机整理文件、丢掉 320 个游戏的人，
   需要一个具体的答案：这件事有两种**有据可查**的成因，而其中只有一种是硬件损坏。

   - **安卓的"采用为内部存储"（adoptable storage）。** 如果手机当时提示"设置这张 SD 卡"而用户点了同意，
     安卓会把它**重新格式化并加密**。Google 自己的文档：
     *"When external storage media is adopted, it's formatted and encrypted to only work with a single
     Android device at a time"*、*"they can choose to adopt the media, which formats and encrypts it"*。
     此后这张卡在 R36S 上读不出，在电脑上也读不出，任何恢复工具都拿不回游戏——
     但**没有任何硬件损坏**，处理办法就是把卡重新格式化、再把游戏拷一遍。
   - **卡自己置上了永久写保护位。** SD 卡有一个 `PERM_WRITE_PROTECT` 标志（CSD bit 13），
     一旦置上就是永久只读——Kingston 的规格书里写了这一位，而 Linux 在 MMC 层无条件执行它。
     从那以后这张卡看着一切正常，就是拒绝任何写入，表现为"一个修不好的文件系统"。

   第三种说法——"OTG 读卡器把卡搞坏了"——是流传最广的解释，而我们**找不到任何一手来源**支持它，
   请把它当成都市传说。

   > ⚠️ **未验证**：我们没有复现手机 / OTG 这个故障。上面两条机制是有据可查的那两条；
   > "读卡器把卡弄坏了"到处都是，但没有任何文档。

   不论哪一种，处理方式一样：把还能读出来的扇区尽快镜像成文件，然后换卡。
   开始出现这种症状的卡，救不回来。

第三种需要排除的情况是纯粹的**假容量卡**——标称 128 GB、实际只有 8 GB，
超出真实容量的数据会静默地自我覆盖。Linux/macOS 上用 `f3`（Fight Flash Fraud），
Windows 上用 `h2testw` 可以测。群里那位"内存卡实际容量只有90G，国产卡"的成员，
遇到的其实是同一件事的良性版本。

### 2.5 真机上的基准事实

![真机 R36S 的系统信息界面：SD Card 1 为 119.88 GB，SD Card 2 未插入](docs/06-real-r36s-system-info.png)

我们自己那台的"系统信息"界面。有两件事值得读出来：**SD Card 1 显示 119.88 GB**——
这正是 128 GB 这一档容量该有的读数，属于十进制与二进制之间的差值，不是假卡的证据
（SD 标准对 SDXC 的定义只有一句 *"over 32GB-2TB"*，并不规定精确字节数，
所以同标称容量的两张卡**本就不要求**一致）。
**SD Card 2 是单独一栏**，那就是第二个卡槽。下面"游戏卡能不能通用"这个问题，关键就在这个槽上。

---

## 三、按顺序救

**第 0 步：停止往卡里写任何东西。** 拔下来。不要跑任何"会写入"的修复工具。不要让任何东西格式化它。
往一张正在坏的卡上写一次，能救回来的东西就少一分。

**第 1 步：确认是哪个卷。** Windows 上 `Win + X` → 磁盘管理。看**分区列表**，不要看盘符：
一张健康的 R36S 卡是 约 100 MB FAT32 + 约 6.7 GiB Linux + 一个大 exFAT。
Linux 上插好卡后：

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT
sudo fdisk -l /dev/sdX
```

`lsblk` 告诉你文件系统和卷标；`fdisk -l` 打出带偏移量和类型码的分区表。
如果 `fdisk` 报出来的表和上面那张一致，那这张卡在结构上是好的，弹窗只是冲着 ext4 来的。
如果它报"一个分区都没有"，进第 3 步。

**第 2 步：如果弹窗指的是 p2，取消它，继续用。** 什么都没坏。
这就是一张健康的 R36S 卡在 Windows 上的正常状态。去读那个游戏分区，它是 exFAT。
`EASYROMS` 没盘符就分配一个；如果确实要动 ext4，就在 Linux 上做——
并且注意 ArkOS 明确点名 Paragon Linux File Systems for Windows 是导致卡分区损坏的原因，
所以那个 Windows 下的 ext4 驱动正是要避开的那个。

**第 3 步：在尝试任何修复之前，先把卡镜像出来。** 这一步大多数人都跳过，
而它是唯一让后面所有操作变安全的步骤：

```bash
sudo dd if=/dev/sdX of=~/r36s-card.img bs=4M status=progress conv=fsync
```

卡已经在坏的时候改用 `ddrescue`，这样一个坏扇区不会让整次拷贝中断：

```bash
sudo apt install gddrescue
sudo ddrescue -d -r3 /dev/sdX ~/r36s-card.img ~/r36s-card.map

# 改成"卡到卡"的话，手册里的写法要加 -f，不加会被直接拒绝：
sudo ddrescue -f -r3 /dev/sdX /dev/sdY mapfile
```

`-d`（`--idirect`）是让**输入端**绕过内核缓存（直接写是另一个标志 `-D`）；
`-r3`（`--retry-passes=3`）对坏区域重试三遍；第三个参数是 mapfile，
手册称它为 *"an essential part of ddrescue's effectiveness"*，它让这次拷贝可以续跑，
并且明确告诉你哪些区域最终没读回来。第二种写法里的 `-f` 不是可选项：
手册写的是 *"needed when outfile is not a regular file, but a device or partition"*——
不加它，ddrescue 会拒绝往设备上写。这道保险是有道理的，所以设备名请核对两遍。
Windows 上，**Win32DiskImager** 做同一件事——它的官方说明就是"把裸镜像写到可移动设备，
**或者把可移动设备备份成裸镜像文件**"，而且它建议在往任何设备写入之前，先做一次 Read。

之后就只在 `~/r36s-card.img` 或它的副本上动手。从这里开始，你做的任何事都不会让卡更糟。

**第 4 步：分区表没了就重建它。** 对卡（或对镜像）跑 TestDisk：
它能重写丢失的分区表、从备份恢复 FAT32 引导扇区、找到 ext2/3/4 的备份超级块。
只有在核对过分区大小和 2.1 节那张表一致（100 MB、约 6.7 GiB、剩下的是游戏分区）之后，才接受它给出的表。

**第 5 步：把文件捞出来。**
Windows 上，**DiskGenius** 官方文档写明可以读取和恢复 EXT4/3/2 分区，
也能浏览并拷出 Windows 自己拒绝挂载的分区里的文件。需要从 ext4 系统分区里取东西时就用它。
文件系统被删得厉害或者彻底乱了的时候，用 **PhotoRec**（和 TestDisk 同一个项目）：
它按文件特征码做内容雕刻，不依赖目录项还在不在。

**第 6 步：如果你只是想让这张卡能用，重新刷，而不是修。**
把一张已知良好卡的 dd 镜像、或者项目原始的 `.img`，用 `dd`、Rufus 或 USB Image Tool 写回去，
比手工修一个坏掉的 ext4 分区又快又可靠得多。然后再把游戏拷回来。

**第 7 步：检查文件系统，然后才可以信任这张卡。**

![实测终端输出：对测试镜像跑 fsck.vfat、e2fsck 和 resize2fs](docs/05-filesystem-check-verified.png)

```bash
sudo fsck.vfat -n /dev/sdX1     # -n = 只检查，不修改
sudo e2fsck -f -y /dev/sdX2     # -f = 强制完整检查，-y = 全部回答 yes
```

`fsck.vfat -n` 是只读检查；如果它报出问题，用 `-a` 再跑一次让它修。
对已挂载的文件系统跑 `e2fsck` 是坏主意——想先看一眼就用 `-n`。
卡可疑的时候，**先对镜像跑这些命令**。

**有一个标志必须用对，因为被抄错得太多了：** 对 `fsck.fat` 来说，`-f` **不是**"强制"的意思。
手册的定义是 *"salvage unused cluster chains to files"*（把游离簇链抢救成文件）——
这跟"安全地空跑一次检查"几乎正好相反，而且它会把已删除的数据暴露成新文件。
FAT 上"自动修复"的标志是 `-a`（或 `-p`）。而 `e2fsck` 里同一个字母**确实**是"强制"，
混淆就是这么来的。顺带一提：`fsck.vfat`、`fsck.fat`、`dosfsck` 是同一个程序的三个名字。

### 工具对照

| 要做什么 | Windows | Linux / macOS |
|---|---|---|
| 看分区表 | 磁盘管理、DiskGenius | `lsblk`、`fdisk -l`、`sfdisk --dump` |
| 把卡镜像成文件 | Win32DiskImager（Read）、DiskGenius 专业版 | `dd`、`ddrescue` |
| 把镜像写回卡 | Rufus、USB Image Tool、Win32DiskImager | `dd` |
| 重建丢失的分区表 | TestDisk、DiskGenius | TestDisk |
| 读取/恢复 ext4 内容 | DiskGenius（免费版只能拷小文件） | 挂载它、`e2fsck`、PhotoRec |
| 从坏掉的文件系统里雕刻文件 | PhotoRec | PhotoRec |
| 检查 FAT32 / ext4 完整性 | DiskGenius（检查与修复） | `fsck.vfat`、`e2fsck` |
| 测假容量卡 | `h2testw` | `f3probe`、`f3write` + `f3read` |

**依赖 DiskGenius 之前，有两件事必须知道。** 它确实有文档支持在 Windows 下读写 EXT4，
但各版本的差别在坏卡场景下很要命：在官方功能对比里，*"Copy Files from Ext2/3/4 Partition"* 这一行
在免费版**和**标准版都标着 *"Only small files can be copied"*；而
*"Back up Partition to Image by Sectors"*——也就是坏卡真正需要的那个**逐扇区镜像**——免费版没有。
所以：做镜像用 Linux 或 `ddrescue`，DiskGenius 留给"浏览和取文件"。

另外纠正一条被推荐得极广的工具：**balenaEtcher 不能把设备读成镜像文件。**
它自己的 issue 里 *"Implement a way to perform a backup from a drive"* 从 2016 年挂到现在。
它能做的是设备到设备的克隆。所以它当不了这个用途的备份工具——
而且 ArkOS 本来就叫你别用它写这些镜像。

> ⚠️ **未实测**：Windows 那一列全部引自各厂商自己的文档。我们不跑 Windows，
> 所以 DiskGenius、Win32DiskImager、DiskGenius 的 ext4 浏览、`h2testw` 这些我们都没有实际执行过。
> 本节里的 Linux 命令是写文时对测试镜像实跑的。

---

## 四、备份、克隆，以及"别人的卡我能用吗"

### 4.1 给卡做备份

![从卡到镜像文件再到新卡，以及会毁掉克隆的几条规则](docs/03-backup-and-clone.png)

整卡拷贝是**扇区级**拷贝：它带走分区表、引导扇区和两个文件系统。这正你想要的，
因为只有这种拷贝才能还原出一张**可启动**的卡。

```bash
# 卡 -> 镜像
sudo dd if=/dev/sdX of=~/r36s-card.img bs=4M status=progress conv=fsync
# 镜像 -> 另一张卡
sudo dd if=~/r36s-card.img of=/dev/sdY bs=4M status=progress conv=fsync
```

动手前先比容量。`blockdev --getsz /dev/sdX` 打印 512 字节扇区数，`lsblk` 打印字节数。
**同样是"128 GB"的卡，扇区数并不一样。** 这不是传说：SD 标准对 SDXC 的定义只有
*"over 32GB-2TB"*，不规定精确字节数，真实容量来自每张卡自己的 `C_SIZE` 寄存器，
所以印刷相同的两张卡**被允许**不一样。现实中也有人遇到两张同品牌同型号的"128 GB"卡，
其中一张小了几个 GB，装不下另一张的镜像。我们自己那台报 119.88 GB；
同容量档位的另一张卡差上一两个 GB 是常事。目标卡比源卡小的话，拷贝会中途写不下，
留下一张"看着挺像、但起不来"的截断卡。群里那位说 32 G 卡用傲梅轻松备份克隆"得用 64 G 卡才行"，
背后的机制就是这个——扇区级克隆要求目标不小于源。
所以稳妥的习惯是：往**同容量或更大**的卡上克隆，并且动手前先确认目标的容量。

> ⚠️ **未实测**：我们没有测试傲梅轻松备份。上面这条通用规则是扇区级拷贝的性质；
> 傲梅那一句是群里一位成员的经验，不是有据可查的规格说明。

### 4.2 克隆时不要做的事

- **不要在首次开机前手动扩容游戏分区。** ArkOS 用大写字母写明：镜像会自己在首次开机时扩容，
  在此之前手动扩容会导致开机卡死。如果自动扩容失败，用 GParted。
- **不要用 balenaEtcher 写这些镜像。** ArkOS 明确说了不要用，R36S 的 dArkOS 项目也重复了这一点：
  他们推荐的 Windows 工具是 Rufus、USB Image Tool 和 Win32DiskImager。
  dArkOSen 更进一步，把 Rufus 写成"支持要求"，并报告说其他写入工具会产生
  "文件丢失、主题丢失、模拟器毫无理由启动失败"的安装。
- **别让 Paragon Linux File Systems for Windows 碰这张卡。** ArkOS 点名它是导致 ext4 分区损坏的原因。
- **不要以为换更大的卡就自动多了空间。** 整卡克隆会照抄旧的分区尺寸，
  克隆到更大的卡上，多出来的空间是未分配的。在 Linux 上给 **exFAT** 分区扩容并不是一条命令的事，
  所以换大卡最可靠的路线是：重新刷镜像，然后把 `roms` 目录当普通文件拷过去。
  游戏分区是 exFAT，任何电脑都读得了。

### 4.3 不同商家的游戏内存卡能通用吗

这是群里最值得认真回答的一个问题，而答案要分成两半：

- **只装游戏的卡，能——但有条件。** 游戏就在 exFAT 分区上的 `roms/<平台>/` 里，那就是一堆文件。
  为这台 R36S 准备好的卡，插到另一台 R36S 上能用，前提是那台机器被告知去读它
  （第二卡槽要在设置里启用，而且它要求一个它能挂载的文件系统）。这部分跟主板无关。
- **系统卡，不能。** `BOOT` 分区里装着 `KERNEL`、`boot.ini` 和 `.dtb` 设备树文件，
  而设备树是针对**某一个具体主板版本、某一个具体屏幕**匹配的。
  R36S 的 dArkOS 项目维护着一整张矩阵，每一项在 `BOOT` 分区里都有自己的目录：
  `R36S-V12 2023-08-18 Panel 0`、`R36S-V12 2023-08-18 Panel 4`、`R36S-V21 2024-12-18`、
  `R36S-V22 2024-12-18`、`R36S-Y02 2024-12-18`、`HL-R36H-V20 2024-05-18`、
  G80C/G80CA 山寨板，以及 `Y3506`「酱油」系列。放对 `.dtb` 就能开机；
  放了别的板子的，你得到的就是大家嘴里那句"刷完黑屏"。

  真正该去翻的地方不是 wiki 那张支持表，而是项目自己的 `BOOT` 暂存目录——那里每个主板版本一个目录。
  我们核对时它包含：`R36S-V30 2025-11-18`、`R36S-V21 2024-12-18 2550/2551/2552`、
  `R36S-V22 2024-12-18`、`R36S-Y02 2024-11-27` 与 `2024-12-18`、`R36XX-V21 2024-12-18`、
  `R36S Plus-V20 2025-03-18`、`HL-R36H-V20` 与 `V21`、`GR36S-Y07 2025-09-11`，
  以及 `G80C`/`G80CA`/`G80D`/`GR36` 山寨板。
  也就是说 **V30 并不孤单**。但这里有个值得记下来的教训：QQ 日志里 V30 用户说
  "看群文件也没找到v30的相关信息"——这正是去查 wiki 支持表、而不是去查 `.dtb` 实际所在目录的结果。

  > ⚠️ **这是写作当时的核对结果，不是长期事实。** 我们是在 2026-10-11 列的这棵目录树。
  > 它每版都会变，依赖之前请回源核对。
- **实际结论：** 各管各的。系统放内置卡，游戏放第二张卡；
  要把卡借人，确认借出去的是游戏卡。系统卡如果一定要换机器用，
  就要预期重新选一次机型 / 换一次 `.dtb`。

> ⚠️ **未实测**：我们手上没有 V12 和 V21/V30 两台机器并排来实测互换卡。
> 上面的主板/屏幕矩阵引自 R36S dArkOS 项目自己的支持硬件列表。

---

## 五、怎么确认救好了

1. **`fdisk -l` / 磁盘管理里能看到三个分区**，偏移量与 2.1 节一致；
   `lsblk -o NAME,FSTYPE,LABEL` 显示 `vfat`/`BOOT`、`ext4`、`exfat`/`EASYROMS`。
2. **机器能启动进前端**，游戏数量和之前对得上。
3. **`sudo fsck.vfat -n /dev/sdX1` 和 `sudo e2fsck -f -y /dev/sdX2` 都干净通过。**
   反复检查都还在报错的卡，是应该换掉的卡，不是应该一直修的卡。
4. **随便挑一个已知文件读回来是对的。** 从卡上拷一个大文件出来，和原件比哈希。
   怀疑假卡时，这次读取就是整个测试的意义：`f3write` 之后跑 `f3read` 是系统化做法，
   Windows 上 `h2testw` 是同一件事。
5. **克隆过的话，克隆结果也要验。** 不要因为没报错就认为拷贝成功了。
   R36S 的 dArkOS 文档给了一个很干净的 Linux 校验法：

   ```bash
   stat -c '%s' ./r36s.img                        # 源镜像的字节数
   sha256sum ./r36s.img                           # 它的哈希
   sudo head -c IMAGE_SIZE /dev/sdX | sha256sum   # 从卡上读回同样多字节的哈希
   ```

   两个哈希必须完全一致。不一致就说明这张卡不是忠实副本，此时该做的是重写，
   而不是去排查掌机。

---

## 六、怎么别再踩

- **在卡还能用的那天就备份。** 一次 `dd` 或 Win32DiskImager 的 Read 花二十分钟，
  把"我丢了 320 个游戏和我做的主题"变成"我把镜像还原回去了"。
- **系统卡和游戏卡分开。** 这样一次格式化事故的代价是一堆文件，而不是一整套系统。
- **不要用手机或 OTG 读卡器在卡上整理文件。** 这条来自群里那位因为这么干丢掉 320 个游戏的人。
  有两种翻车方式，而且两种都救不回来：手机若提议把这张卡"用作内部存储"，点了同意就等于
  **重新格式化并加密**；而写到一半断连的可移动介质，没有日志可以回滚。
  正经做法是关机（`Start + Power`），取出卡，用真正的读卡器接电脑。
  手机一旦弹出"如何使用这张 SD 卡"，对 R36S 的卡来说，答案永远不是"用作内部存储"。
- **买正规渠道的正牌卡，并且到手就测一次。** ArkOS 原话：最小 8 GB，
  强烈建议 16 GB 以上，*"Do not use low quality or no name brand SD cards. Those will most likely fail
  quickly, cause inconsistent emulation performance, or fail in booting up."*
- **永远不要对一个还没确认过的格式化弹窗点"是"。** 先看分区列表。
  在磁盘管理里花两分钟，比重新刷一次镜像便宜得多。
- **永远不要修一张还没做过镜像的卡。** 镜像是你的撤销键；没有它，每一次修复尝试都不可逆。
- **规范弹出，取卡前先关机。** R36S 自己的指南也是这么说的：不要在开机状态下拔卡。

---

## 七、来源

| 主张 | 来源 |
|---|---|
| R36S dArkOS 镜像的真实分区表、偏移与尺寸；首次开机扩容并重启两次；`SELECT MODEL.bat`；主板/屏幕 → dtb 矩阵；"Rufus 是唯一受支持的刷写软件"及其原因 | [dArkOSen-R36S wiki → Installation](https://github.com/djparentx/dArkOSen-R36S/wiki/Installation) |
| "MAKE SURE TO CLICK CANCEL"；镜像会把 NTFS 分区扩容并转成 exFAT；不要手动扩容分区；不要用 balenaEtcher；Paragon Linux File Systems for Windows 会损坏卡；最小卡容量；Windows 10 1703+ 的盘符修法；PSP/NDS 目录依赖；镜像哈希校验法；用 Samba/FTP 而不是读卡器加游戏 | [ArkOS wiki → Home](https://github.com/christianhaitian/arkos/wiki/Home) |
| 第二卡槽支持的文件系统（exFAT/EXT4/FAT/FAT32/NTFS）、EXT4 的 `1002:1002` 权限警告、必须两张卡都在才能启动 | [ArkOS FAQ](https://github.com/christianhaitian/arkos/wiki/Frequently-Asked-Questions---RG353V) |
| `BOOT` 与 `EASYROMS` 从资源管理器里消失，以及磁盘管理分配盘符的修法 | [dArkOSRE-R36 wiki → Show Hidden Partitions in Windows](https://github.com/southoz/dArkOSRE-R36/wiki/Show-Hidden-Partitions-in-Windows) |
| 主板/屏幕版本矩阵与第二卡槽（`Switch to SD2 for Roms`）流程 | [dArkOSRE-R36 wiki → Firmware Installation](https://github.com/southoz/dArkOSRE-R36/wiki/Firmware-Installation) |
| "If you have an Ext4-formatted drive, you cannot mount it on your Windows file system"；"`wsl --mount` does not currently support USB/flash drives/SD card readers"；需要管理员权限与 Windows 11 | [Microsoft Learn → Mount a Linux disk in WSL 2](https://learn.microsoft.com/en-us/windows/wsl/wsl2-mount-disk) |
| "把裸镜像写到可移动设备，或者把可移动设备备份成裸镜像文件"；写入前建议先 Read | [Win32 Disk Imager](https://sourceforge.net/projects/win32diskimager/) |
| 分区表恢复、从备份恢复 FAT32 引导扇区、定位 ext2/3/4 备份超级块、从已删分区拷文件 | [CGSecurity → TestDisk](https://www.cgsecurity.org/wiki/TestDisk) |
| `BOOT` 分区的内容、真实的 `boot.ini`、以及各主板版本的 `.dtb` 目录 | [`southoz/dArkOSRE-R36` → `files/BOOT/`](https://github.com/southoz/dArkOSRE-R36/tree/main/files/BOOT) |
| 输出端是设备/分区时必须加 `-f`；`-d/--idirect` 与 `-D/--odirect` 的区别；mapfile 的作用 | [GNU ddrescue 手册](https://www.gnu.org/software/ddrescue/manual/ddrescue_manual.html) |
| `fsck.fat` 的 `-f` 意为"把游离簇链抢救成文件"；`fsck.vfat`/`dosfsck` 是同一个程序 | [dosfstools `fsck.fat(8)`](https://manpages.debian.org/bookworm/dosfstools/fsck.vfat.8.en.html) |
| Windows 支持哪些文件系统；`RAW` 的含义与微软自己"要恢复文件就别格式化"的警告；"Not Initialized" / "Unreadable" 磁盘状态 | [Microsoft Learn → 文件系统驱动](https://learn.microsoft.com/en-us/windows-hardware/drivers/ifs/about-file-system-drivers)、[磁盘管理排错](https://learn.microsoft.com/en-us/troubleshoot/windows-server/backup-and-storage/troubleshoot-disk-management) |
| 安卓把卡"采用为内部存储"时会格式化并加密，使其绑定到单台设备 | [Android Open Source Project → Adoptable storage](https://source.android.com/docs/core/storage/adoptable) |
| 卡自身的 `PERM_WRITE_PROTECT` 标志（CSD bit 13）会让卡永久只读 | Kingston SD 规格书；Linux `drivers/mmc/core/sd.c` |
| SDXC 只被定义为 "over 32GB-2TB"，精确容量因卡而异 | [SD Association → Capacity](https://www.sdcard.org/developers/sd-standard-overview/capacity-sd-sdhc-sdxc-sduc/) |
| DiskGenius 各版本限制（"Only small files can be copied"、免费版无逐扇区镜像）与 EXT4 读写支持 | [DiskGenius → Windows 读写 Ext4](https://www.diskgenius.com/how-to/ext4-windows.php)、[版本对比](https://www.diskgenius.com/editions.php) |
| balenaEtcher 无法把设备读成镜像 | [balena-io/etcher issue #266](https://github.com/balena-io/etcher/issues/266) |
| Windows 下读取/恢复 EXT4/3/2 文件与分区 | [DiskGenius → Linux Data Recovery](https://www.diskgenius.com/how-to/linux-data-recovery.php) |
| `-d/--idirect` 与 `-r/--retry-passes`、mapfile 的作用 | [GNU ddrescue 手册](https://www.gnu.org/software/ddrescue/manual/ddrescue_manual.html) |
| 假容量卡检测 | [AltraMayor/f3](https://github.com/AltraMayor/f3)；`h2testw`（[heise.de](https://www.heise.de/download/product/h2testw-50539)） |
| "要格式化磁盘"弹窗指的是 Linux 分区；`TF1-OS` / `TF2-GAME` 卡槽命名；开机时不要拔卡 | [r36s.org → How to Add Games to R36S](https://r36s.org/articles/tutorial-add-games) |
| 第一节引用的 QQ 原文 | 对 9 个中文掌机 QQ 群、22,869 条去重消息的内部分析 |

**QQ 数据的局限，直说：** 拿不到可靠的逐条时间戳；无法把某条消息可靠归属到某个具名发言人；
11 个掌机相关群里有 2 个只抓到群名没抓到正文；样本被一个大群主导。
本仓库里出现的任何计数都是**消息条数**，不是单位时间频次。

## 相关

- 姊妹篇：[R36S：游戏放进卡里却不显示/打不开](https://github.com/metahubaifeel/r36s-roms-not-loading)
- r36s.space —— [上手第一步与安全设置](https://r36s.space/guide)
- r36s.space —— [R36S 常见问题：充电、游戏文件、主板版本](https://r36s.space/faq)
- r36s.space —— [裸机 R36S：主板版本与配色](https://r36s.space/products/r36s)
- r36s.space —— [买裸机 R36S 之前：一份清单](https://r36s.space/blog/bare-r36s-checklist)

## 许可

MIT —— 见 [LICENSE](LICENSE)。`docs/` 里的截图包含我们自己硬件的实拍照片，可自由转载，注明出处即可。

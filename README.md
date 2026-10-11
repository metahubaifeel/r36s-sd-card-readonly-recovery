# R36S SD card: Windows says "you need to format the disk". Don't — here is how to recover it

**English** · **[中文](README.zh-CN.md)**

> **Short answer.** On a healthy R36S card that prompt is usually about the **Linux (ext4) partition**,
> which Windows has no driver for. The card is not broken and the games are not lost. Click **Cancel**.
> If you click Format, you destroy the boot partition or the operating system, and ArkOS's own
> instructions say the fix is then to re-image the whole card.
> Only two situations are real damage — the partition table is gone, or the card is dying/fake — and
> neither of them is fixed by formatting either. Both are fixed by **imaging the card to a file first**
> and then working on the copy.

This page is the whole procedure: how to tell the two cases apart, what to do in each, how to image and
clone a card so this never costs you a library again, and which popular tools you should not use on
these images.

---

## 1. What people actually report

Verbatim messages from Chinese handheld QQ groups. Group names only — the archive they came from does
not reliably identify individual senders.

> `[r36s（开源掌机）交流1群]` 我TF卡突然读不出来东西了，一直显示要格式化，插进掌机里也打不开，谁能救一救

*"My TF card suddenly won't read anything, it keeps saying it needs to be formatted, and it won't open
in the handheld either. Somebody save me."*

> `[r36s（开源掌机）交流1群]` 大约17点多想导入个游戏，首先显示要格式化，插进掌机里也打不开，现在很急。

*"Around 5pm I wanted to import a game. It first said it needed formatting, and it won't open in the
handheld either. I'm in a hurry."*

> `[R36S掌机爱好者]` 我搞了那么久的系统主题，游戏测试归类，弄了320个游戏，结果今早连手机后，掌机就不读卡了，机都开不起
> 具体情况是，手机连tf卡整理文件，结果突然就掉连接了，再重新连接就始终显示不出来了，当时想估计是坏菜了
> 只读得起boot分区 ／ roms分区的盘都读不出来
> 对了，踩坑提醒，尽量还是不要用otg转接读卡器来读tf卡
> 也就是尽量还是不要用手机读tf卡，用电脑更稳妥

*"I spent ages on my system theme, testing and sorting games, got 320 games together — then this morning
after connecting it to my phone, the handheld stopped reading the card and wouldn't even boot. What
happened: I was sorting files on the TF card through the phone, the connection suddenly dropped, and on
reconnecting it never showed up again. I figured it was dead. Only the boot partition reads — the roms
partition won't show up at all. By the way, a pitfall warning: try not to use an OTG card reader to read
a TF card. Try not to read TF cards with a phone — a computer is safer."*

> `[r36s（开源掌机）交流1群]` 系统在卡上你格式化了不就烷基八氮了吗

*"The system is on the card — if you format it, isn't that the end of everything?"*  ← correct, and worth
tattooing on the card reader.

> `[r36s（开源掌机）交流1群]` 请问大家有没有克隆内存卡的方法呢，我的月光宝盒掌机32G内存卡用傲梅轻松备份克隆，得用64G内存卡才行，我刚买的R36S在备份内存卡，不知道128G内存卡能不能克隆

*"Does anyone have a way to clone a memory card? For my 32 GB Moonlight Box card I used AOMEI Backupper
to clone and it needed a 64 GB card. I'm backing up the card of the R36S I just bought — I don't know
whether a 128 GB card can be cloned."*

> `[r36s（开源掌机）交流1群]` 请问大家，不同商家的R36S掌机都是V30主板的掌机，游戏内存卡可以通用吗

*"Question: these R36S handhelds from different sellers all have a V30 board — are the game memory cards
interchangeable?"*

> `[r36s（开源掌机）交流1群]` 这个掌机内存卡实际容量只有90G，国产卡
> `[r36s（开源掌机）交流1群]` 花点钱买个好点的内存卡，一卡玩三代

*"This handheld's card is only actually 90 GB — a domestic card." / "Spend a bit more on a decent card;
one card lasts three generations."*

Two of these deserve to be answered directly, and they are the two answers this page is built around:
**the volume Windows cannot read is the Linux one, not the games**, and **the game card is portable but
the OS card is not**, because the OS card carries the kernel and the device tree for one specific board.

---

## 2. Why Windows says it

### 2.1 The card has three partitions; Windows can read two of them

![Why Windows offers to format an R36S card, and what each partition is](docs/01-what-windows-sees.png)

A normal R36S card looks like this. The table is the one the R36S dArkOS project documents for its image
— an MBR table with three partitions — reproduced with `fdisk`/`sfdisk` on a same-sized test file so the
offsets and type codes are visible:

![Real terminal output showing the three-partition table with its offsets, sizes and type codes](docs/04-partition-table-verified.png)

| Partition | Start sector | Size in the image | Type code | What it is |
|---|---|---|---|---|
| p1 | 32768 | 100 MiB | `0c` W95 FAT32 (LBA) | `BOOT`: `Image` (the kernel), `uInitrd`, `boot.ini`, the `.dtb` files, the model selector |
| p2 | 237568 | 6.7 GiB | `83` Linux (ext4) | the operating system |
| p3 | 14368768 | 100 MiB placeholder | `07` HPFS/NTFS/exFAT | named `EASYROMS`, grown to fill the card on first boot |

So:

- **p1 `BOOT`** is FAT32. Windows reads it, and it is the small drive that appears first.
- **p2 (ext4)** Windows cannot read at all. Microsoft's list of the filesystems Windows supports is
  *"NTFS / ExFAT / UDF / FAT32"* — ext4 is not on it, and Microsoft's WSL documentation says so without
  hedging: *"If you have an Ext4-formatted drive, you cannot mount it on your Windows file system."*
  Microsoft then defines the state that triggers the dialog: *"If a partition doesn't have a file system
  (it's listed as RAW instead of NTFS, ReFS, FAT32, or exFAT) …"* — and adds its own warning, aimed at
  exactly this situation: *"Formatting a disk erases all data on it, so don't do this step if you're
  trying to recover files from the disk."*
- **p3 `EASYROMS`** is exFAT and Windows *can* read it. If it does not appear, that is usually a missing
  drive letter, and the documented fix is a drive letter, not a format.

### 2.2 Why the "no drive letter" case is so common

The ArkOS documentation addresses Windows 10 1703-and-newer users specifically: if you do not see a drive
letter named `EASYROMS`, open Disk Management, right-click the partition carrying that label and assign a
drive letter. You normally only need to do this once per PC. The dArkOS documentation has the same page
for the R36S, and it names both `BOOT` and `EASYROMS` as the two filesystems that can go missing from
Explorer for exactly this reason.

Related, and worth knowing because it is such a widely-suggested workaround: **you cannot use
`wsl --mount` for this.** Microsoft's documentation is explicit that *"the `wsl --mount` command does not
currently support USB/flash drives/SD card readers"*, and that it requires administrator rights and
Windows 11 (or the Store version of WSL). Any guide telling you to mount your R36S card in WSL does not
work — and on an R36S card you do not need it, because the partition that holds your games is exFAT,
which Windows reads natively.

### 2.3 What clicking Format actually costs

> ArkOS documentation, word for word: *"Once the image has been completed, Windows may ask you if you
> want to format the SD card. **MAKE SURE TO CLICK CANCEL ON THIS QUESTION!** If you let it format the SD
> card, you will need to install the image again."*

That is about the flashing step, but it applies every time the dialogue appears:

- Format **p1 (`BOOT`)** and you lose `Image` (the kernel), `uInitrd`, `boot.ini`, the `.dtb` files and
  the small hidden state files the image writes back into `/boot` — including the persisted board/panel
  selection. The device will not boot, and on a unit that needs a panel-specific device tree the result
  is usually a black screen. The card still holds your games, but you now need a re-flash to use them.
- Format **p2 (ext4)** and you lose the operating system. This is the partition Windows is usually
  complaining about, and it is the one with nothing on it that Windows can show you — which is exactly why
  people click OK.
- Format **p3 (`EASYROMS`)** and you lose the games, the `roms` folder structure and the `roms/tools`
  tree that is bind-mounted into the system (PortMaster and the other on-device tools live there). The
  operating system still boots, so this is the one of the three that is survivable — but you must not
  delete the folders that are already there: ArkOS warns that the PSP and NDS folders contain files those
  emulators need even before you add a single game.

Formatting is not a "reset the card to a known good state" button on a device whose operating system
lives on the same card.

### 2.4 The two cases where the card really is damaged

![Decision tree: what to check before you let anything format the card](docs/02-decision-tree.png)

1. **The partition table is gone.** Here you can use Microsoft's own vocabulary as the test, because it
   distinguishes the two cases for you: a partition that lists as **`RAW`** means Windows can see the
   partition but not its filesystem — which is simply what a Linux partition looks like — whereas a disk
   reporting **"Not Initialized"** (Microsoft: *"the drive formatting has become corrupted somehow"*) or
   **"Unreadable"** means Windows cannot even make sense of the partitioning. Those last two are
   corruption or hardware failure, not a Linux filesystem. Windows will happily offer to "fix" either one
   by creating a single empty partition, overwriting the table that still points at your data.
   **TestDisk** exists for exactly this: it reconstructs a lost partition table, recovers FAT boot sectors
   from their backups, and can locate ext2/3/4 backup superblocks.
2. **The card is dying, counterfeit, or has been reformatted by a phone.** The group member who lost 320
   games after sorting files on the card through a phone needs a specific answer, because there are two
   documented ways that happens and only one of them is hardware damage:

   - **Android adoptable storage.** If the phone offered to "set up" the card and that was accepted,
     Android reformats and encrypts it. Google's documentation: *"When external storage media is adopted,
     it's formatted and encrypted to only work with a single Android device at a time"* and *"they can
     choose to adopt the media, which formats and encrypts it."* The card is then unreadable in the R36S
     **and** on a PC, and no recovery tool will get the games back — but nothing is physically broken, and
     the fix is simply to reformat the card and copy the games across again.
   - **A card that has set its own permanent write-protect bit.** SD cards carry a
     `PERM_WRITE_PROTECT` flag (CSD bit 13) that makes them read-only for good — Kingston documents the
     bit, and Linux acts on it unconditionally in the MMC layer. From then on the card looks fine and
     refuses every write, which shows up as a filesystem that cannot be repaired.

   The third possibility, that the OTG reader physically damaged the card, is the explanation everyone
   repeats and we could not find a single primary source for it — treat it as folklore.

   > ⚠️ **Not verified:** we did not reproduce the phone/OTG failure. The two mechanisms above are the
   > documented ones; "the reader fried the card" is asserted widely and documented nowhere.

   Either way the response is the same: copy whatever sectors are still readable to an image file, then
   replace the card. A card that has started failing does not come back.

The third possibility worth ruling out is simply a **fake-capacity card** — one that reports 128 GB and
really has 8 GB, so everything past the real capacity silently overwrites itself. `f3`
("Fight Flash Fraud") on Linux/macOS and `h2testw` on Windows test for this. The quoted group member
noticing that their "128 GB" card is "actually only 90 GB" is the benign version of the same discovery.

### 2.5 Ground truth from a real unit

![A real R36S System Info screen showing SD Card 1 at 119.88 GB and SD Card 2 not inserted](docs/06-real-r36s-system-info.png)

Our own unit's System Info screen. Two things worth reading off it: **SD Card 1 reports 119.88 GB**,
which is the figure you expect from a card in the 128 GB class — the usual decimal-vs-binary gap, not
evidence of a fake card (the SD standard itself defines SDXC only as *"over 32GB-2TB"*, so the exact byte
count is per-card and two cards with the same printed capacity are not required to match) — and **SD Card 2 is listed separately**, which is the second slot. That second
slot is what makes the "my games card is interchangeable" question below interesting.

---

## 3. How to recover, in order

**Step 0 — stop writing to the card.** Unplug it. Do not run a "repair" tool that writes. Do not let
anything format it. Every write to a failing card reduces what can still be recovered.

**Step 1 — identify the volume.** On Windows, `Win + X` → Disk Management. Read the partition list, not
the drive letters: a healthy R36S card shows ~100 MB FAT32, ~6.7 GiB Linux, and a large exFAT partition.
On Linux, with the card inserted:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,MOUNTPOINT
sudo fdisk -l /dev/sdX
```

`lsblk` tells you the filesystems and labels; `fdisk -l` prints the partition table with offsets and
type codes. If `fdisk` reports a table that matches the one above, the card is structurally fine and the
prompt was about ext4. If it reports no partitions, go to step 3.

**Step 2 — if the prompt was about p2, cancel it and carry on.** Nothing is wrong. This is the normal
state of a healthy R36S card on Windows. Read the games partition, which is exFAT. If `EASYROMS` has no
drive letter, assign one; if you need to touch ext4, do it from Linux — and note that ArkOS specifically
names Paragon Linux File Systems for Windows as a cause of corrupted cards, so that particular Windows
ext4 driver is the one to avoid.

**Step 3 — image the card before you attempt any repair.** This is the step people skip, and it is the
only one that makes the rest safe:

```bash
sudo dd if=/dev/sdX of=~/r36s-card.img bs=4M status=progress conv=fsync
```

Use `ddrescue` instead when the card is failing, so a bad sector does not end the copy:

```bash
sudo apt install gddrescue
sudo ddrescue -d -r3 /dev/sdX ~/r36s-card.img ~/r36s-card.map

# card -> card instead? Then the manual's own form adds -f, and omits it would be refused:
sudo ddrescue -f -r3 /dev/sdX /dev/sdY mapfile
```

`-d` (`--idirect`) reads the *input* bypassing the kernel cache (writing direct is a separate flag, `-D`);
`-r3` (`--retry-passes=3`) retries bad areas three times; the third argument is the mapfile, which lets you
resume and tells you exactly which regions never came back — the manual calls it *"an essential part of
ddrescue's effectiveness"*. The `-f` in the second form is not optional: the manual says it is *"needed
when outfile is not a regular file, but a device or partition"*, and without it ddrescue refuses to write
to a device. That safeguard exists for a good reason, so read the device names twice. On Windows, **Win32DiskImager** does the same job — its own documentation describes it as being
designed to *"write a raw disk image to a removable device **or backup a removable device to a raw image
file**"*, and it recommends doing a Read before you ever write to a device.

Then work on `~/r36s-card.img`, or on a copy of it. Nothing you do from here can make the card worse.

**Step 4 — if the partition table is missing, rebuild it.** Run TestDisk against the card (or the image):
it can rewrite a lost partition table, recover a FAT32 boot sector from its backup, and find ext2/3/4
backup superblocks. Accept the proposed table only after checking the listed partition sizes against the
table in section 2.1 — 100 MB, roughly 6.7 GiB, then the rest.

**Step 5 — get your files out.**
On Windows, **DiskGenius** documents reading and recovering EXT4/3/2 partitions, and it can also browse
and copy files off a partition Windows itself refuses to mount. That is the tool to use when you need to
pull something out of the ext4 system partition.
For deleted or badly damaged filesystems, **PhotoRec** (same project as TestDisk) carves files out by
content and does not care that the directory entries are gone — it cares about the file signatures.

**Step 6 — if you only need the card to work again, re-flash rather than repair.** A dd image of a known
good card, or the original `.img` from the project, restored with `dd`, Rufus or USB Image Tool, is
faster and far more reliable than repairing a damaged ext4 partition by hand. Then copy your roms back.

**Step 7 — check the filesystems, and only then trust the card.**

![Real terminal output of fsck.vfat, e2fsck and resize2fs against test images](docs/05-filesystem-check-verified.png)

```bash
sudo fsck.vfat -n /dev/sdX1     # -n = check only, change nothing
sudo e2fsck -f -y /dev/sdX2     # -f = force a full check, -y = answer yes
```

`fsck.vfat -n` is a read-only check; if it reports problems, re-run it with `-a` to let it repair them.
`e2fsck` on a mounted filesystem is a bad idea — start with `-n` if you want a look before you let it
write. Run these against the **image** first if the card is suspect.

**One flag to get right, because a lot of copied advice gets it wrong:** for `fsck.fat`, `-f` does **not**
mean "force". The manual defines it as *"salvage unused cluster chains to files"* — essentially the
opposite of a no-op safety check, and it can expose deleted data as new files. The "repair automatically"
flag for FAT is `-a` (or `-p`). On `e2fsck` the same letter *does* mean force, which is where the
confusion comes from. Also note that `fsck.vfat`, `fsck.fat` and `dosfsck` are three names for one
program.

### Tool summary

| Need | Windows | Linux / macOS |
|---|---|---|
| Read the partition table | Disk Management, DiskGenius | `lsblk`, `fdisk -l`, `sfdisk --dump` |
| Image the card to a file | Win32DiskImager (Read), DiskGenius Professional | `dd`, `ddrescue` |
| Write an image back | Rufus, USB Image Tool, Win32DiskImager | `dd` |
| Rebuild a lost partition table | TestDisk, DiskGenius | TestDisk |
| Read / recover ext4 content | DiskGenius (free tier: small files only) | mount it, `e2fsck`, PhotoRec |
| Carve files out of a broken filesystem | PhotoRec | PhotoRec |
| Check FAT32 / ext4 integrity | DiskGenius (check & repair) | `fsck.vfat`, `e2fsck` |
| Test for a fake card | `h2testw` | `f3probe`, `f3write` + `f3read` |

**Two things to know before you rely on DiskGenius.** It does document EXT4 read *and* write on Windows,
but the tiers differ in ways that matter on a failing card: in the official feature comparison the
*"Copy Files from Ext2/3/4 Partition"* row is marked *"Only small files can be copied"* in the Free **and**
Standard editions, and *"Back up Partition to Image by Sectors"* — the sector image you actually want from
a dying card — is not in the Free edition. Use a Linux machine or `ddrescue` for the image, whatever you
use for browsing.

And a correction to a very common recommendation: **balenaEtcher cannot read a card to an image file.**
Its own issue tracker has had *"Implement a way to perform a backup from a drive"* open since 2016. It can
clone drive-to-drive. So it is not a backup tool for this purpose — and ArkOS tells you not to use it for
these images anyway.

> ⚠️ **Not tested here:** everything in the Windows column is quoted from the vendor's own documentation.
> We do not run Windows, so we did not execute DiskGenius, Win32DiskImager, DiskGenius' ext4 browsing or
> `h2testw` ourselves. The Linux commands in this section were run while writing, against test images.

---

## 4. Backing up, cloning, and "can I use someone else's card?"

### 4.1 Backing up the card

![Card to image file to new card, and the rules that break a clone](docs/03-backup-and-clone.png)

A whole-card copy is a sector-level copy: it carries the partition table, the boot sectors and both
filesystems. That is what you want, because it is the only kind of copy that restores a *bootable* card.

```bash
# card -> image
sudo dd if=/dev/sdX of=~/r36s-card.img bs=4M status=progress conv=fsync
# image -> another card
sudo dd if=~/r36s-card.img of=/dev/sdY bs=4M status=progress conv=fsync
```

Check the sizes before you start. `blockdev --getsz /dev/sdX` prints the capacity in 512-byte sectors,
`lsblk` prints it in bytes. **Cards sold as the same capacity do not have the same number of sectors.** This is not folklore: the SD
standard defines SDXC only as *"over 32GB-2TB"* and fixes no exact byte count, so the real size comes from
each card's own `C_SIZE` register and two cards with identical printing are allowed to differ. Real
reports exist of two same-brand, same-model "128 GB" cards where one was a few gigabytes too small to
receive the other's image. Our own unit reports 119.88 GB; another card in the same capacity class can
easily be a gigabyte or two different. If the target is smaller than the source, the copy runs out of room partway and leaves you with
a truncated card that looks plausible and does not boot. This is exactly the mechanism behind the group
member's report that cloning a 32 GB card with AOMEI Backupper *"needed a 64 GB card"* — sector-level
cloning wants a target at least as large as the source, so the safe habit is to clone onto a card that is
the same size or bigger, and to confirm the target's size first.

> ⚠️ **Not tested here:** we did not test AOMEI Backupper. The general rule above is a property of
> sector-level copying, and the AOMEI report is one group member's experience, not a documented
> specification.

### 4.2 What not to do when cloning

- **Do not expand the games partition by hand before the first boot.** ArkOS documents this in capital
  letters: the image expands that partition itself at first boot, and doing it manually beforehand makes
  the boot hang. If the automatic expansion fails, use GParted.
- **Do not use balenaEtcher for these images.** ArkOS says so explicitly, and the R36S dArkOS projects
  repeat it: their recommended Windows tools are Rufus, USB Image Tool and Win32DiskImager. dArkOSen goes
  further and makes Rufus a support requirement, reporting that other writers produce installations with
  missing files, missing themes and emulators that fail for no visible reason.
- **Do not let Paragon Linux File Systems for Windows near the card.** ArkOS names it as a documented
  cause of a corrupted ext4 partition.
- **Do not assume a bigger card gives you the extra space for free.** A raw clone copies the old
  partition sizes, so a clone onto a bigger card leaves the extra space unallocated. Growing an **exFAT**
  partition on Linux is not a routine one-command operation, which is why the reliable route for a bigger
  card is: re-flash the image, then copy the `roms` folder across as ordinary files. The games partition
  is exFAT, so any PC can read it.

### 4.3 Are game cards from different sellers interchangeable?

This is the group question worth answering properly, and the answer is split:

- **A games-only card, yes — within limits.** The games live in `roms/<system>/` on an exFAT partition, and
  that partition is just files. A card prepared for one R36S works in another R36S as long as the second
  unit is told to read it (the second-card slot has to be enabled in the settings, and it expects a
  filesystem it can mount). Nothing about it is board-specific.
- **A system card, no.** The `BOOT` partition carries `KERNEL`, `boot.ini` and the `.dtb` device-tree
  files, and the device tree is matched to one specific board revision and one specific panel. The R36S
  dArkOS project maintains a matrix of them, each with its own folder in the `BOOT` partition:
  `R36S-V12 2023-08-18 Panel 0`, `R36S-V12 2023-08-18 Panel 4`, `R36S-V21 2024-12-18`,
  `R36S-V22 2024-12-18`, `R36S-Y02 2024-12-18`, `HL-R36H-V20 2024-05-18`, the G80C/G80CA clone boards,
  and the `Y3506` "Soy Sauce" variants. Drop the right `.dtb` in and the device boots; drop in one from a
  different board and you get exactly the symptom people describe as "a black screen after flashing".

  The place to look is not the wiki's support table but the project's own `BOOT` staging tree, which
  carries one directory per board revision. That tree currently includes
  `R36S-V30 2025-11-18`, `R36S-V21 2024-12-18 2550/2551/2552`, `R36S-V22 2024-12-18`,
  `R36S-Y02 2024-11-27` and `2024-12-18`, `R36XX-V21 2024-12-18`, `R36S Plus-V20 2025-03-18`,
  `HL-R36H-V20` and `V21`, `GR36S-Y07 2025-09-11`, and the `G80C`/`G80CA`/`G80D`/`GR36` clone boards.
  So a V30 unit is not stranded — but note the lesson from the groups: V30 owners in the QQ log say they
  went looking for V30 information and could not find it, which is what happens when you search a wiki
  table instead of the directory the `.dtb` files actually live in.

  > ⚠️ **Verified when this was written, not a live fact.** We listed that tree on 2026-10-11. It changes
  > between releases; check it before relying on it.
- **The practical consequence:** keep them separate. Use the internal card for the system and a second
  card for games, and if you lend a card to someone, make sure it is the games card. If you must move a
  system card between units, expect to re-select the model/`.dtb` for the board it is going into.

> ⚠️ **Not tested here:** we do not have a V12 and a V21/V30 unit side by side to test card swapping.
> The panel/board matrix above is quoted from the R36S dArkOS project's own supported-hardware list.

---

## 5. How to check it worked

1. **`fdisk -l` / Disk Management shows three partitions** with the offsets from section 2.1, and
   `lsblk -o NAME,FSTYPE,LABEL` shows `vfat`/`BOOT`, `ext4` and `exfat`/`EASYROMS`.
2. **The device boots to the frontend** and the game count matches what it was before.
3. **`sudo fsck.vfat -n /dev/sdX1` and `sudo e2fsck -f -y /dev/sdX2` both come back clean.** A card that
   keeps producing errors on repeated checks is a card to replace, not to keep repairing.
4. **A known file reads back correctly.** Copy a large file off the card and compare its hash with the
   original. On a card you suspect of being fake, that read is the whole point of the test: `f3read`
   after `f3write` does it systematically, and `h2testw` does the same on Windows.
5. **If you cloned, verify the clone too.** Do not assume a copy succeeded because it did not print an
   error. The R36S dArkOS documentation gives a neat check for a flashed image on Linux:

   ```bash
   stat -c '%s' ./r36s.img                        # size of the source image
   sha256sum ./r36s.img                           # its hash
   sudo head -c IMAGE_SIZE /dev/sdX | sha256sum   # hash of the same number of bytes read back
   ```

   The two hashes must match exactly. If they do not, the card is not a faithful copy, and re-writing it
   is the fix — not troubleshooting the handheld.

---

## 6. How not to step on this again

- **Back up the card the day it works.** A `dd` image or a Win32DiskImager Read costs twenty minutes and
  turns "I lost 320 games and the theme I built" into "I restored the image".
- **Keep the system card and the games card separate.** Then a formatting accident costs you a folder of
  files, not an installation.
- **Do not sort files on the card through a phone or an OTG card reader.** This one comes from a group
  member who lost 320 games to exactly that. Two things can go wrong and neither is recoverable: if the
  phone offers to use the card as internal storage, accepting it reformats and encrypts the card; and a
  connection that drops mid-write on removable media has no journal to fall back on. Shut the device down
  (`Start + Power`), take the card out and use a real card reader on a computer. If a phone ever asks
  *"how will you use this SD card?"*, the answer for an R36S card is never "as internal storage".
- **Buy a name-brand card from a reputable seller, and test it once.** ArkOS: minimum 8 GB, 16 GB or more
  strongly recommended, *"Do not use low quality or no name brand SD cards. Those will most likely fail
  quickly, cause inconsistent emulation performance, or fail in booting up."*
- **Never answer "yes" to a format prompt you have not identified.** Read the partition list first. Two
  minutes in Disk Management is cheaper than a re-flash.
- **Never repair a card you have not imaged.** The image is your undo button; without it, every repair
  attempt is irreversible.
- **Eject properly, and shut the device down before removing the card.** The R36S's own guides say the
  same thing: do not pull the card while the device is on.

---

## Sources

| Claim | Source |
|---|---|
| Real partition table, offsets and sizes of the R36S dArkOS image; first boot expands partitions and reboots twice; `SELECT MODEL.bat`; the board/panel → dtb matrix; "Rufus is the only supported flashing software", and why | [dArkOSen-R36S wiki → Installation](https://github.com/djparentx/dArkOSen-R36S/wiki/Installation) |
| "MAKE SURE TO CLICK CANCEL"; the image expands an NTFS partition and converts it to exFAT; do not expand the partition manually; do not use balenaEtcher; Paragon Linux File Systems for Windows corrupts the card; minimum card sizes; Windows 10 1703+ drive-letter fix; the PSP/NDS folder dependencies; the image-hash verification recipe; adding roms via Samba/FTP instead of a card reader | [ArkOS wiki → Home](https://github.com/christianhaitian/arkos/wiki/Home) |
| Second-slot filesystem list (exFAT/EXT4/FAT/FAT32/NTFS), the EXT4 `1002:1002` permission warning, both cards required to boot | [ArkOS FAQ](https://github.com/christianhaitian/arkos/wiki/Frequently-Asked-Questions---RG353V) |
| `BOOT` and `EASYROMS` going missing from Windows Explorer, and the Disk Management drive-letter fix | [dArkOSRE-R36 wiki → Show Hidden Partitions in Windows](https://github.com/southoz/dArkOSRE-R36/wiki/Show-Hidden-Partitions-in-Windows) |
| Board/panel revision matrix and the second-card (`Switch to SD2 for Roms`) procedure | [dArkOSRE-R36 wiki → Firmware Installation](https://github.com/southoz/dArkOSRE-R36/wiki/Firmware-Installation) |
| "If you have an Ext4-formatted drive, you cannot mount it on your Windows file system"; "`wsl --mount` does not currently support USB/flash drives/SD card readers"; administrator rights and Windows 11 requirement | [Microsoft Learn → Mount a Linux disk in WSL 2](https://learn.microsoft.com/en-us/windows/wsl/wsl2-mount-disk) |
| "write a raw disk image to a removable device or backup a removable device to a raw image file"; Read-before-write recommendation | [Win32 Disk Imager](https://sourceforge.net/projects/win32diskimager/) |
| Partition-table recovery, FAT32 boot-sector recovery from backup, ext2/3/4 backup superblock location, file copy from deleted partitions | [CGSecurity → TestDisk](https://www.cgsecurity.org/wiki/TestDisk) |
| The `BOOT` partition contents, the real `boot.ini`, and the per-board `.dtb` directories | [`southoz/dArkOSRE-R36` → `files/BOOT/`](https://github.com/southoz/dArkOSRE-R36/tree/main/files/BOOT) |
| `-f --force` needed when the output is a device or partition; `-d/--idirect` vs `-D/--odirect`; what the mapfile is for | [GNU ddrescue manual](https://www.gnu.org/software/ddrescue/manual/ddrescue_manual.html) |
| `fsck.fat`'s `-f` means "salvage unused cluster chains to files"; `fsck.vfat`/`dosfsck` are the same program | [dosfstools `fsck.fat(8)`](https://manpages.debian.org/bookworm/dosfstools/fsck.vfat.8.en.html) |
| Which filesystems Windows supports; what `RAW` means and Microsoft's own "don't format if you are recovering files" warning; the "Not Initialized" / "Unreadable" disk states | [Microsoft Learn → About file system drivers](https://learn.microsoft.com/en-us/windows-hardware/drivers/ifs/about-file-system-drivers), [Troubleshoot Disk Management](https://learn.microsoft.com/en-us/troubleshoot/windows-server/backup-and-storage/troubleshoot-disk-management) |
| Android formats and encrypts a card when it is adopted, binding it to one device | [Android Open Source Project → Adoptable storage](https://source.android.com/docs/core/storage/adoptable) |
| The card's own `PERM_WRITE_PROTECT` flag (CSD bit 13) makes a card permanently read-only | Kingston SD datasheet; Linux `drivers/mmc/core/sd.c` |
| SDXC is defined only as "over 32GB-2TB", so the exact capacity is per card | [SD Association → Capacity](https://www.sdcard.org/developers/sd-standard-overview/capacity-sd-sdhc-sdxc-sduc/) |
| DiskGenius edition limits ("Only small files can be copied", sector image not in Free) and EXT4 read/write support | [DiskGenius → Read & Write Ext4 in Windows](https://www.diskgenius.com/how-to/ext4-windows.php), [edition comparison](https://www.diskgenius.com/editions.php) |
| balenaEtcher cannot create an image from a device | [balena-io/etcher issue #266](https://github.com/balena-io/etcher/issues/266) |
| EXT4/3/2 file and partition recovery on Windows | [DiskGenius → Linux Data Recovery](https://www.diskgenius.com/how-to/linux-data-recovery.php) |
| `-d/--idirect` and `-r/--retry-passes`, and what the mapfile does | [GNU ddrescue manual](https://www.gnu.org/software/ddrescue/manual/ddrescue_manual.html) |
| Fake-flash-capacity testing | [AltraMayor/f3](https://github.com/AltraMayor/f3); `h2testw` ([heise.de](https://www.heise.de/download/product/h2testw-50539)) |
| The "you need to format the disk" prompt refers to the Linux partition; `TF1-OS` / `TF2-GAME` slot naming; never remove the card while the device is on | [r36s.org → How to Add Games to R36S](https://r36s.org/articles/tutorial-add-games) |
| QQ messages quoted in section 1 | internal analysis of 22,869 unique messages from 9 Chinese handheld QQ groups |

**Limits of the QQ data, stated plainly:** no reliable per-message timestamps, no reliable attribution of
a message to a named sender, two of the eleven handheld groups yielded a name but no message bodies, and
the sample is dominated by one large group. Counts quoted anywhere in this repository are message counts,
not frequencies over time.

## Related

- Companion page: [R36S: the ROMs are on the card but the games don't launch](https://github.com/metahubaifeel/r36s-roms-not-loading)
- r36s.space — [first steps and safe setup](https://r36s.space/guide)
- r36s.space — [R36S FAQ: charging, game files, board revisions](https://r36s.space/faq)
- r36s.space — [the bare R36S: boards and finishes](https://r36s.space/products/r36s)
- r36s.space — [before you buy a bare R36S: a checklist](https://r36s.space/blog/bare-r36s-checklist)

## Licence

MIT — see [LICENSE](LICENSE). The screenshots in `docs/` include photographs of our own hardware; reuse
them freely with attribution.

# অধ্যায় ৩২: ডিস্ক, পার্টিশন, ফাইলসিস্টেম ও মাউন্ট

## এই অধ্যায়ে যা শিখবে

- লিনাক্স ডিস্ককে কীভাবে "দেখে" — block device কী, আর `sda`, `nvme0n1`, `vda`, `mmcblk0` নামগুলোর মানে কী
- ডিস্ক থেকে ফাইল পর্যন্ত পুরো স্তর (layer) — VFS, ফাইলসিস্টেম ড্রাইভার, page cache, block layer
- `lsblk`, `blkid`, `findmnt` দিয়ে কোন ডিস্কে কী আছে খুঁজে বের করা
- MBR আর GPT পার্টিশন টেবিলের পার্থক্য, ভেতরের গঠনসহ
- `fdisk`, `gdisk`, `parted` দিয়ে ধাপে ধাপে পার্টিশন তৈরি, মুছে ফেলা, টাইপ বদলানো
- `mkfs` দিয়ে ext4, XFS, vfat (FAT32), exFAT ফাইলসিস্টেম তৈরি
- mount point কী, `mount`/`umount` কীভাবে কাজ করে, "target is busy" সমস্যার সমাধান
- `/etc/fstab` ফাইলের প্রতিটি ফিল্ড, UUID কেন ব্যবহার করবে, `mount -a` ও `findmnt --verify`
- `df -h`, `du -sh`, `ncdu` দিয়ে ডিস্কের জায়গা মাপা — আর কেন `df` ও `du` আলাদা সংখ্যা দেখায়
- swap partition ও swapfile তৈরি, `swapon`, `swappiness`
- `fsck`, `e2fsck`, `xfs_repair`, `tune2fs` দিয়ে ফাইলসিস্টেম মেরামত ও টিউনিং
- USB পেনড্রাইভ নিরাপদে মাউন্ট ও আনমাউন্ট করা
- loop device ব্যবহার করে একদম ঝুঁকিহীনভাবে সব কিছু হাতে-কলমে চর্চা করা

---

## ৩২.১ শুরুর আগে একটা গল্প: আলমারি, তাক আর লাইব্রেরি

ধরো তুমি একটা নতুন বড় আলমারি কিনলে। এটা হলো তোমার **ডিস্ক** (হার্ডডিস্ক/SSD)। খালি আলমারিতে সরাসরি জিনিস রাখা যায়, কিন্তু সব এলোমেলো হয়ে যাবে। তাই তুমি:

1. আলমারিকে কয়েকটা **তাকে ভাগ করলে** — এটা হলো **পার্টিশন (partition)**।
2. প্রতিটা তাকে একটা **সাজানোর পদ্ধতি** ঠিক করলে — কোনটায় ফাইল-ফোল্ডার, কোনটায় বাক্স, প্রতিটার লেবেল আর একটা খাতায় তালিকা — এটা হলো **ফাইলসিস্টেম (filesystem)**, যেমন ext4 বা XFS।
3. তারপর বাড়ির কোনো ঘরের দরজার সাথে ওই তাকটাকে **জুড়ে দিলে**, যাতে "স্টোররুমে" ঢুকলেই ওই তাকের জিনিস পাওয়া যায় — এটা হলো **মাউন্ট (mount)**, আর ওই দরজাটা হলো **mount point**।

উইন্ডোজে প্রতিটা পার্টিশন আলাদা "ড্রাইভ" (C:, D:) হিসেবে আসে। লিনাক্সে কোনো ড্রাইভ লেটার নেই — পুরো সিস্টেম একটাই বিশাল গাছ (tree), যার গোড়া `/`। অন্য যেকোনো ডিস্ক বা পার্টিশন ওই গাছের কোনো একটা ডিরেক্টরিতে "জুড়ে" দেওয়া হয়। (ডিরেক্টরি গাছের বিস্তারিত দেখো [অধ্যায় ৭](07-filesystem-hierarchy.md)।)

```
             উইন্ডোজ                               লিনাক্স
   ┌──────┐  ┌──────┐  ┌──────┐                     /
   │  C:  │  │  D:  │  │  E:  │          ┌──────────┼──────────┬─────────┐
   └──────┘  └──────┘  └──────┘        /home       /boot       /mnt      /media
   (প্রতিটা আলাদা গাছ)                  (sda3)      (sda1)      /data     /usb
                                                              (sdb1)    (sdc1)
                                         একটাই গাছ, ডিস্কগুলো ডালে জোড়া
```

> 📝 **মনে রাখো:** ডিস্ক → পার্টিশন → ফাইলসিস্টেম → মাউন্ট। এই চারটা ধাপ মুখস্থ রাখলে এই অধ্যায়ের সবকিছু সহজ লাগবে।

---

## ৩২.২ ভেতরের গঠন: একটা `cat` কমান্ড ডিস্ক পর্যন্ত কীভাবে পৌঁছায়

তুমি যখন `cat /home/rahim/note.txt` চালাও, তখন ভেতরে অনেকগুলো স্তর পার হয়ে ডেটা আসে। [অধ্যায় ২](02-linux-architecture.md)-এ আমরা kernel-এর সাধারণ গঠন দেখেছি; এখানে শুধু স্টোরেজ অংশটা জুম করে দেখি:

```
 ┌────────────────────────────────────────────────────────────┐
 │  User space:  cat, cp, vim, তোমার প্রোগ্রাম                 │
 │                 │  open(), read(), write()  (system call)   │
 ├─────────────────▼──────────────────────────────────────────┤
 │  VFS (Virtual File System) — সব ফাইলসিস্টেমের "এক ভাষা"     │
 │        │              │              │             │        │
 │     ext4           XFS           vfat/exfat      tmpfs ...  │
 │  (ফাইলসিস্টেম ড্রাইভার — কোন ফাইল ডিস্কের কোন block-এ)       │
 │        └──────────────┴──────┬───────┘                      │
 │                      Page Cache (RAM-এ ক্যাশ)                │
 │                              │                              │
 │  Block Layer — I/O request সাজানো, I/O scheduler             │
 │        (mq-deadline, bfq, kyber, none)                      │
 │                              │                              │
 │  Device driver: sd (SATA/SCSI/USB), nvme, virtio_blk, mmc   │
 ├──────────────────────────────▼─────────────────────────────┤
 │  হার্ডওয়্যার: HDD / SSD / NVMe / USB stick / VM-এর ভার্চুয়াল ডিস্ক │
 └────────────────────────────────────────────────────────────┘
```

প্রতিটা স্তর কী করে:

- **VFS (Virtual File System):** kernel-এর ভেতরের একটা "অনুবাদক"। প্রোগ্রাম জানে না নিচে ext4 আছে নাকি পেনড্রাইভের FAT32 — সে শুধু `read()` ডাকে। VFS ঠিক ফাইলসিস্টেম ড্রাইভারের কাছে কাজটা পাঠিয়ে দেয়। এই কারণেই `cp` দিয়ে ext4 থেকে FAT32-এ কপি করা যায় কোনো বাড়তি ঝামেলা ছাড়া।
- **ফাইলসিস্টেম ড্রাইভার:** জানে ফাইলের নাম → inode → ডিস্কের কোন block-এ ডেটা। (inode-এর বিস্তারিত [অধ্যায় ১১](11-links-inode.md)-এ।)
- **Page cache:** একবার পড়া ডেটা RAM-এ রেখে দেয়, পরের বার ডিস্কে না গিয়েই দেয়। লেখার সময়ও ডেটা আগে RAM-এ যায়, পরে ডিস্কে লেখা হয় ("dirty pages")। এজন্যই পেনড্রাইভ টেনে খোলার আগে `sync`/`umount` জরুরি।
- **Block layer:** ডিস্ককে দেখে কতগুলো নির্দিষ্ট মাপের ব্লকের (সাধারণত 512 বা 4096 byte sector) সারি হিসেবে। I/O request গুলো জোড়া লাগায়, ক্রম সাজায়।
- **Device driver:** আসল হার্ডওয়্যারের সাথে কথা বলে।

> 💡 **টিপস:** কোন ডিস্কে কোন I/O scheduler চলছে দেখতে: `cat /sys/block/sda/queue/scheduler` — বন্ধনীর ভেতরেরটা সক্রিয়, যেমন `[mq-deadline] none`। NVMe-তে সাধারণত `[none]`।

### Block device আর character device

লিনাক্সে "সবকিছুই ফাইল", তাই ডিস্কও `/dev`-এর ভেতরে একটা ফাইল হিসেবে দেখা যায়।

- **Block device** (`b`): ডেটা নির্দিষ্ট মাপের ব্লকে পড়া-লেখা হয়, যেকোনো জায়গায় লাফিয়ে যাওয়া যায় (random access)। ডিস্ক, পার্টিশন, USB stick।
- **Character device** (`c`): ডেটা বাইটের স্রোতের মতো আসে। কীবোর্ড, টার্মিনাল (`/dev/tty`), `/dev/null`।

```bash
$ ls -l /dev/sda /dev/sda1 /dev/null
brw-rw---- 1 root disk 8, 0 Nov 12 09:10 /dev/sda
brw-rw---- 1 root disk 8, 1 Nov 12 09:10 /dev/sda1
crw-rw-rw- 1 root root 1, 3 Nov 12 09:10 /dev/null
```

প্রথম অক্ষর `b` মানে block, `c` মানে character। সাইজের জায়গায় যে `8, 0` দেখছ, তা হলো **major** ও **minor number**। major number বলে কোন ড্রাইভার (8 = `sd` ড্রাইভার), minor বলে কোন নির্দিষ্ট ডিভাইস (0 = পুরো ডিস্ক sda, 1 = প্রথম পার্টিশন)। `/dev`-এর এই ফাইলগুলো বানায় **udev** (systemd-udevd), হার্ডওয়্যার চিনতে পারার সাথে সাথে।

---

## ৩২.৩ Block device-এর নামকরণ: sda, nvme0n1, vda ...

নাম দেখে বুঝে নিতে পারবে কোন ধরনের ডিস্ক। এটা খুব কাজের দক্ষতা — ভুল ডিস্ক ফরম্যাট করা এড়ানোর প্রথম ধাপ।

| নাম | কী ধরনের ডিভাইস | পার্টিশনের নাম |
|------|-----------------|----------------|
| `sda`, `sdb`, `sdc` ... | SATA/SAS HDD-SSD, USB stick, SCSI — `sd` ড্রাইভার | `sda1`, `sda2`, `sdb1` |
| `nvme0n1`, `nvme1n1` | NVMe SSD (M.2) | `nvme0n1p1`, `nvme0n1p2` |
| `vda`, `vdb` | KVM/QEMU ভার্চুয়াল মেশিনের virtio ডিস্ক | `vda1`, `vda2` |
| `xvda` | Xen ভার্চুয়াল ডিস্ক (পুরোনো AWS EC2) | `xvda1` |
| `mmcblk0` | SD কার্ড / eMMC (Raspberry Pi, ল্যাপটপের কার্ড রিডার) | `mmcblk0p1` |
| `sr0` | CD/DVD ড্রাইভ | — |
| `loop0`, `loop1` | loop device — একটা সাধারণ ফাইলকে ডিস্ক বানানো (snap প্যাকেজ এভাবেই মাউন্ট হয়) | `loop0p1` |
| `dm-0`, `dm-1` | device-mapper (LVM, LUKS এনক্রিপশন) | — |
| `md0`, `md127` | সফটওয়্যার RAID | `md0p1` |
| `zram0` | RAM-এর ভেতরে কম্প্রেসড swap (Fedora-তে ডিফল্ট) | — |

**`sd` নামের নিয়ম:** প্রথম যে ডিস্ক পাওয়া যায় সেটা `sda`, পরেরটা `sdb`... `sdz`-এর পর `sdaa`। পার্টিশন নম্বর শুধু শেষে যোগ হয়।

**NVMe নামের নিয়ম:** `nvme0n1p2` ভাঙলে —

```
  nvme 0  n1  p2
   │   │   │   └── partition 2
   │   │   └────── namespace 1 (একটা NVMe ড্রাইভকে কয়েকটা namespace-এ ভাগ করা যায়)
   │   └────────── controller 0 (প্রথম NVMe কন্ট্রোলার)
   └────────────── NVMe ডিভাইস
```

নাম যখন সংখ্যায় শেষ হয় (`nvme0n1`, `mmcblk0`, `loop0`), তখন পার্টিশনের আগে একটা `p` বসে, যাতে `nvme0n11` নিয়ে বিভ্রান্তি না হয়।

> ⚠️ **সতর্কতা:** `sda`/`sdb` নামগুলো **স্থায়ী নয়**। আজ যেটা `sdb`, কাল একটা USB লাগিয়ে বুট করলে সেটা `sdc` হয়ে যেতে পারে — কোন ডিস্ক আগে সাড়া দিল তার ওপর নির্ভর করে। তাই `/etc/fstab`-এ কখনো `/dev/sdb1` লিখবে না, UUID লিখবে (৩২.১০ দেখো)।

### স্থায়ী নাম: `/dev/disk/by-*`

udev প্রতিটা ডিস্কের জন্য কিছু স্থায়ী symlink বানায় (symlink কী, দেখো [অধ্যায় ১১](11-links-inode.md)):

```bash
$ ls -l /dev/disk/by-uuid/
lrwxrwxrwx 1 root root 10 Nov 12 09:10 3f2a9c1e-7b4d-4e8a-9d61-2c5b8e0f1a77 -> ../../sda2
lrwxrwxrwx 1 root root 10 Nov 12 09:10 A1B2-C3D4 -> ../../sda1
```

| ডিরেক্টরি | কীসের ভিত্তিতে নাম |
|-----------|-------------------|
| `/dev/disk/by-uuid/` | ফাইলসিস্টেমের UUID (mkfs করার সময় তৈরি হয়) |
| `/dev/disk/by-partuuid/` | পার্টিশনের UUID (GPT টেবিলে থাকে, ফরম্যাট করলেও বদলায় না) |
| `/dev/disk/by-label/` | ফাইলসিস্টেমের লেবেল (যেমন `DATA`) |
| `/dev/disk/by-id/` | হার্ডওয়্যারের মডেল + সিরিয়াল নম্বর |
| `/dev/disk/by-path/` | কোন পোর্ট/কন্ট্রোলারে লাগানো |

---

## ৩২.৪ `lsblk` — কোন ডিস্কে কী আছে এক নজরে

`lsblk` (list block devices) হলো ডিস্কের কাজে তোমার প্রথম বন্ধু। এটা `/sys` থেকে তথ্য পড়ে, তাই সাধারণ ইউজার হিসেবেও চলে।

**Syntax:** `lsblk [options] [device...]`

```bash
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0  74.3M  1 loop /snap/core22/1663
sda           8:0    1  14.9G  0 disk
└─sda1        8:1    1  14.9G  0 part /media/rahim/PENDRIVE
nvme0n1     259:0    0 476.9G  0 disk
├─nvme0n1p1 259:1    0     1G  0 part /boot/efi
├─nvme0n1p2 259:2    0     2G  0 part /boot
└─nvme0n1p3 259:3    0 473.9G  0 part /
```

কলামগুলোর মানে:

| কলাম | মানে |
|-------|------|
| `NAME` | ডিভাইসের নাম; গাছের মতো দেখায় কোন পার্টিশন কোন ডিস্কের |
| `MAJ:MIN` | major:minor number |
| `RM` | removable কিনা (1 = হ্যাঁ, যেমন USB) |
| `SIZE` | আকার |
| `RO` | read-only কিনা (1 = হ্যাঁ) |
| `TYPE` | `disk`, `part`, `loop`, `lvm`, `crypt`, `rom`, `raid1` ইত্যাদি |
| `MOUNTPOINTS` | কোথায় মাউন্ট করা (পুরোনো ভার্সনে কলামের নাম `MOUNTPOINT`) |

গুরুত্বপূর্ণ অপশন:

| অপশন | কাজ |
|-------|-----|
| `-f` | ফাইলসিস্টেম তথ্য: FSTYPE, LABEL, UUID, খালি জায়গা |
| `-o NAME,SIZE,FSTYPE,UUID` | নিজের পছন্দের কলাম |
| `-p` | পুরো পাথ দেখাবে (`/dev/sda1`) |
| `-d` | শুধু ডিস্ক, পার্টিশন বাদ |
| `-b` | সাইজ byte-এ |
| `-e 7` | major 7 (loop) বাদ দাও — snap থাকলে তালিকা পরিষ্কার হয় |
| `-S` | SCSI/SATA ডিভাইসের তথ্য (মডেল, ভেন্ডর, ট্রান্সপোর্ট) |
| `-J` | JSON আউটপুট (স্ক্রিপ্টে কাজে লাগে) |

```bash
$ lsblk -f -e 7
NAME        FSTYPE FSVER LABEL    UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda
└─sda1      vfat   FAT32 PENDRIVE 5E3C-91AF                               9.1G    39% /media/rahim/PENDRIVE
nvme0n1
├─nvme0n1p1 vfat   FAT32          A1B2-C3D4                             1G     1% /boot/efi
├─nvme0n1p2 ext4   1.0            7c0e5d2a-1f3b-4a6e-8c9d-0b1a2e3f4d5c    1.6G    11% /boot
└─nvme0n1p3 ext4   1.0            3f2a9c1e-7b4d-4e8a-9d61-2c5b8e0f1a77  380.2G    14% /
```

> 💡 **টিপস:** নতুন ডিস্ক বা USB লাগানোর **আগে** একবার `lsblk` চালাও, **পরে** আরেকবার। যেটা নতুন এসেছে, সেটাই তোমার ডিভাইস। এই অভ্যাস তোমাকে ভুল ডিস্ক মুছে ফেলা থেকে বাঁচাবে।

---

## ৩২.৫ `blkid` ও `findmnt` — UUID, টাইপ আর মাউন্টের খোঁজ

### `blkid`

`blkid` ডিস্কের ভেতরে ফাইলসিস্টেমের "সিগনেচার" পড়ে টাইপ, UUID, লেবেল বলে দেয়।

```bash
$ sudo blkid
/dev/nvme0n1p1: UUID="A1B2-C3D4" BLOCK_SIZE="512" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="9b3c1d2e-..."
/dev/nvme0n1p3: UUID="3f2a9c1e-7b4d-4e8a-9d61-2c5b8e0f1a77" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="c4d5e6f7-..."
/dev/sda1: LABEL="PENDRIVE" UUID="5E3C-91AF" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="0a1b2c3d-01"
```

| অপশন/ব্যবহার | কাজ |
|--------------|-----|
| `blkid /dev/sdb1` | একটা নির্দিষ্ট ডিভাইস |
| `blkid -s UUID -o value /dev/sdb1` | শুধু UUID-এর মানটা (স্ক্রিপ্টে দারুণ) |
| `blkid -L DATA` | লেবেল দিয়ে ডিভাইস খোঁজা |
| `blkid -U <uuid>` | UUID দিয়ে ডিভাইস খোঁজা |
| `blkid -o list` | টেবিল আকারে |

> 📝 **মনে রাখো:** দুই রকম UUID আছে। `UUID=` হলো **ফাইলসিস্টেমের** (আবার mkfs করলে বদলে যায়)। `PARTUUID=` হলো **পার্টিশনের** (GPT টেবিলে লেখা; MBR-এ এটা ডিস্ক-আইডি + পার্টিশন নম্বর, যেমন `0a1b2c3d-01`)। FAT ফাইলসিস্টেমের UUID আসলে ছোট একটা "volume serial" (`5E3C-91AF`)।

### `findmnt`

কী কোথায় মাউন্ট আছে, গাছের মতো দেখায়:

```bash
$ findmnt /
TARGET SOURCE         FSTYPE OPTIONS
/      /dev/nvme0n1p3 ext4   rw,relatime,errors=remount-ro

$ findmnt -t ext4,vfat
TARGET     SOURCE         FSTYPE OPTIONS
/          /dev/nvme0n1p3 ext4   rw,relatime,errors=remount-ro
├─/boot    /dev/nvme0n1p2 ext4   rw,relatime
└─/boot/efi /dev/nvme0n1p1 vfat  rw,relatime,fmask=0077,dmask=0077,...
```

| অপশন | কাজ |
|-------|-----|
| `findmnt` | সব মাউন্ট, গাছ আকারে |
| `findmnt /mnt/data` | নির্দিষ্ট mount point |
| `findmnt -t xfs` | নির্দিষ্ট ফাইলসিস্টেম টাইপ |
| `findmnt --verify` | `/etc/fstab`-এ ভুল আছে কিনা যাচাই (খুব গুরুত্বপূর্ণ!) |
| `findmnt -s` | `/etc/fstab`-এ যা লেখা আছে তা দেখায় |
| `findmnt -D` | `df`-এর মতো আউটপুট |

---

## ৩২.৬ পার্টিশন টেবিল: MBR বনাম GPT

পার্টিশন বানানোর আগে ডিস্কে একটা "সূচিপত্র" লাগে — কোন পার্টিশন কোন sector থেকে কোথায় শেষ। এটাই **partition table** বা **disklabel**। দুই রকম আছে।

### MBR (Master Boot Record) — পুরোনো পদ্ধতি

১৯৮৩ সাল থেকে চলছে (IBM PC DOS 2.0)। ডিস্কের একেবারে প্রথম 512 byte-এ থাকে:

```
 ডিস্কের sector 0 (512 byte)
 ┌──────────────────────────────┬──────────────────┬────────┐
 │  Bootstrap code (446 byte)   │ Partition table  │ 0x55AA │
 │  (BIOS এটা চালায়)             │ 4 × 16 = 64 byte │ 2 byte │
 └──────────────────────────────┴──────────────────┴────────┘
                                  ↑ সর্বোচ্চ ৪টা এন্ট্রি
```

MBR-এর সীমাবদ্ধতা:

- সর্বোচ্চ **৪টা primary** পার্টিশন। বেশি লাগলে ১টাকে **extended** বানিয়ে তার ভেতরে অনেকগুলো **logical** পার্টিশন (লিনাক্সে এরা `sda5` থেকে শুরু হয়)।
- sector-এর ঠিকানা 32-bit, তাই 512-byte sector-এ সর্বোচ্চ **2 TiB** ডিস্ক।
- টেবিলের একটাই কপি — নষ্ট হলে সব পার্টিশন হারানোর ঝুঁকি।
- কোনো checksum নেই।

### GPT (GUID Partition Table) — আধুনিক পদ্ধতি

UEFI স্ট্যান্ডার্ডের অংশ। আজকের সব নতুন কম্পিউটারে এটাই ডিফল্ট।

```
 ┌─────────┬──────────┬─────────────────────┬──────────────────┬──────────────────┬──────────┐
 │ LBA 0   │ LBA 1    │ LBA 2–33            │  ... পার্টিশনগুলো  │ ব্যাকআপ entries   │ শেষ LBA  │
 │Protective│ GPT      │ Partition entries   │  (আসল ডেটা)        │                  │ ব্যাকআপ   │
 │  MBR    │ Header   │ (128 × 128 byte)    │                  │                  │ Header   │
 └─────────┴──────────┴─────────────────────┴──────────────────┴──────────────────┴──────────┘
```

- **Protective MBR:** পুরোনো টুল যেন ডিস্ককে "খালি" ভেবে মুছে না দেয়, তাই সামনে একটা নকল MBR থাকে যা বলে "পুরো ডিস্ক একটাই পার্টিশন, টাইপ 0xEE"।
- ডিফল্টে **১২৮টা** পার্টিশন, extended/logical-এর ঝামেলা নেই।
- 64-bit ঠিকানা — কার্যত সীমাহীন আকার (512-byte sector-এ ৮ ZiB)।
- **ডিস্কের শেষে ব্যাকআপ** হেডার ও টেবিল থাকে, আর **CRC32 checksum** দিয়ে নষ্ট হলে ধরা পড়ে।
- প্রতিটা পার্টিশনের একটা নিজস্ব GUID (PARTUUID), একটা টাইপ GUID ও একটা নাম (PARTLABEL) থাকে।

| বিষয় | MBR | GPT |
|-------|-----|-----|
| সর্বোচ্চ ডিস্ক আকার | 2 TiB (512-byte sector) | কার্যত সীমাহীন |
| পার্টিশন সংখ্যা | ৪ primary (extended দিয়ে বেশি) | ডিফল্ট ১২৮ |
| ব্যাকআপ টেবিল | নেই | আছে (ডিস্কের শেষে) |
| Checksum | নেই | CRC32 |
| বুট পদ্ধতি | Legacy BIOS | UEFI (BIOS+GRUB দিয়েও সম্ভব, BIOS boot partition লাগে) |
| `fdisk`-এ বানানোর কমান্ড | `o` | `g` |
| কবে ব্যবহার করবে | পুরোনো মেশিন, খুব পুরোনো ডিভাইসের সাথে সামঞ্জস্য | প্রায় সবসময় |

> 💡 **টিপস:** কোন ডিস্কে কোন টেবিল আছে দেখতে: `sudo fdisk -l /dev/sda` চালিয়ে `Disklabel type: gpt` বা `dos` দেখো (`dos` মানে MBR)। অথবা `lsblk -o NAME,PTTYPE`।

বুট প্রসেসে MBR/GPT ও UEFI কীভাবে কাজ করে তা বিস্তারিত আছে [অধ্যায় ২৭](27-boot-process.md)-এ।

### সাধারণ পার্টিশন টাইপ

| ব্যবহার | MBR টাইপ কোড | GPT (`gdisk` কোড) | `fdisk`-এ alias |
|---------|-------------|-------------------|-----------------|
| Linux filesystem | `83` | `8300` | `linux` |
| Linux swap | `82` | `8200` | `swap` |
| Linux LVM | `8e` | `8e00` | `lvm` |
| Linux RAID | `fd` | `fd00` | `raid` |
| EFI System Partition | `ef` | `ef00` | `uefi` |
| BIOS boot (GPT+BIOS-এ GRUB-এর জন্য) | — | `ef02` | — |
| Windows NTFS/exFAT/FAT32 ডেটা | `07` / `0c` | `0700` | — |

> 📝 **মনে রাখো:** পার্টিশনের "টাইপ" শুধু একটা লেবেল বা ইঙ্গিত। আসল ফাইলসিস্টেম কী হবে তা ঠিক হয় `mkfs`-এ। টাইপ `8300` রেখে তার ভেতরে XFS বানানো একদম স্বাভাবিক।

---

## ৩২.৭ পার্টিশন তৈরি — `fdisk`, `gdisk`, `parted`

> ⚠️ **সতর্কতা — পড়ো, তারপর এগোও:** পার্টিশন টুল ভুল ডিস্কে চালালে **সব ডেটা হারাতে পারো**, আর তা ফেরত আনা প্রায় অসম্ভব। নিয়ম তিনটা: (১) `lsblk` দিয়ে ডিস্কের নাম ও আকার মিলিয়ে নাও; (২) মাউন্ট করা ডিস্কে কাজ কোরো না; (৩) চর্চার জন্য loop device ব্যবহার করো (হাতে-কলমে অংশে দেখানো আছে)। `fdisk` আর `gdisk` তোমার পরিবর্তন `w` চাপার আগ পর্যন্ত ডিস্কে লেখে না — কিন্তু `parted` প্রতিটা কমান্ড **সাথে সাথেই** ডিস্কে লেখে!

### `fdisk` — সবচেয়ে প্রচলিত

`util-linux` প্যাকেজের অংশ, সব distro-তে থাকে। আধুনিক `fdisk` MBR ও GPT দুটোই পারে।

**শুধু দেখা:**

```bash
$ sudo fdisk -l /dev/sdb
Disk /dev/sdb: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

কোনো `Disklabel type` লাইন না থাকলে ডিস্কটা একদম খালি।

**ইন্টারঅ্যাকটিভ মোড:** `sudo fdisk /dev/sdb`। ভেতরে এক অক্ষরের কমান্ড চলে:

| কী | কাজ |
|----|-----|
| `m` | সাহায্য (সব কমান্ডের তালিকা) |
| `p` | বর্তমান পার্টিশন টেবিল দেখাও |
| `g` | নতুন খালি **GPT** টেবিল বানাও |
| `o` | নতুন খালি **MBR (DOS)** টেবিল বানাও |
| `n` | নতুন পার্টিশন |
| `d` | পার্টিশন মুছে ফেলো |
| `t` | পার্টিশনের টাইপ বদলাও |
| `l` | সব টাইপের তালিকা |
| `F` | খালি (unpartitioned) জায়গা দেখাও |
| `v` | টেবিল যাচাই করো |
| `x` | expert মোড |
| `w` | পরিবর্তন ডিস্কে **লিখে** বের হও |
| `q` | কিছু না লিখে বের হও (ভুল হলে এটাই বাঁচার উপায়!) |

**ধাপে ধাপে উদাহরণ:** একটা 20 GiB ডিস্ক `/dev/sdb`-তে GPT টেবিল, একটা 15 GiB ডেটা পার্টিশন আর একটা 2 GiB swap পার্টিশন।

```bash
$ sudo fdisk /dev/sdb

Welcome to fdisk (util-linux 2.39.3).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS (MBR) disklabel with disk identifier 0x5c1d2e3f.

Command (m for help): g
Created a new GPT disklabel (GUID: 6A0B...-...).

Command (m for help): n
Partition number (1-128, default 1):              ← Enter (ডিফল্ট 1)
First sector (2048-41943006, default 2048):       ← Enter
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-41943006, default 41940991): +15G

Created a new partition 1 of type 'Linux filesystem' and of size 15 GiB.

Command (m for help): n
Partition number (2-128, default 2):
First sector (31459328-41943006, default 31459328):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (31459328-41943006, default 41940991): +2G

Created a new partition 2 of type 'Linux filesystem' and of size 2 GiB.

Command (m for help): t
Partition number (1,2, default 2): 2
Partition type or alias (type L to list all): swap

Changed type of partition 'Linux filesystem' to 'Linux swap'.

Command (m for help): p
Disk /dev/sdb: 20 GiB, 21474836480 bytes, 41943040 sectors
...
Disklabel type: gpt
Disk identifier: 6A0B...

Device        Start      End  Sectors Size Type
/dev/sdb1      2048 31459327 31457280  15G Linux filesystem
/dev/sdb2  31459328 35653631  4194304   2G Linux swap

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

কয়েকটা জিনিস খেয়াল করো:

- **First sector 2048 কেন?** 2048 × 512 = 1 MiB। পার্টিশনকে 1 MiB সীমানায় শুরু করলে আধুনিক SSD ও 4K-sector ডিস্কে পারফরম্যান্স ভালো থাকে (**alignment**)। fdisk নিজেই এটা করে, তুমি শুধু Enter চাপো।
- **`+15G`** মানে "শুরু থেকে আরও 15 GiB"। `+500M`, `+1T` ইত্যাদিও চলে। শুধু Enter চাপলে বাকি সব জায়গা নেবে।
- বাকি ~3 GiB খালি থাকল — পরে কাজে লাগাতে পারবে।

**fdisk-এর স্ক্রিপ্টেবল ভাই `sfdisk`:** একটা লাইনে পুরো ডিস্ক পার্টিশন করা যায়, অটোমেশনে কাজে লাগে:

```bash
$ sudo sfdisk --dump /dev/sdb > sdb-table.txt     # টেবিলের ব্যাকআপ
$ sudo sfdisk /dev/sdb < sdb-table.txt            # ফেরত বসানো
```

> 💡 **টিপস:** কোনো ডিস্কে কাজ শুরুর আগে `sfdisk --dump` দিয়ে টেবিলের ব্যাকআপ রাখো। ভুলে পার্টিশন মুছে ফেললেও (ডেটা ওভাররাইট না হলে) টেবিল ফেরত বসিয়ে সব উদ্ধার করা যায়।

### পার্টিশন টেবিল নতুন করে পড়ানো

`w` দেওয়ার পর kernel সাধারণত নতুন টেবিল নিজেই পড়ে নেয়। কিন্তু ডিস্কের কোনো পার্টিশন ব্যবহারে থাকলে বার্তা আসতে পারে: *"Re-reading the partition table failed.: Device or resource busy"*। তখন:

```bash
$ sudo partprobe /dev/sdb        # parted প্যাকেজের অংশ
$ sudo partx -u /dev/sdb         # util-linux-এর বিকল্প
```

এতেও না হলে রিবুট।

### `gdisk` — GPT বিশেষজ্ঞ

`gdisk` (GPT fdisk) শুধু GPT-র জন্য, `fdisk`-এর মতোই ইন্টারঅ্যাকটিভ। Ubuntu/Debian-এ `sudo apt install gdisk`, Fedora/RHEL-এ `sudo dnf install gdisk`।

| কী | কাজ |
|----|-----|
| `?` | সাহায্য |
| `p` | টেবিল দেখাও |
| `n` | নতুন পার্টিশন |
| `d` | মুছে ফেলো |
| `t` | টাইপ বদলাও (৪-অক্ষরের হেক্স কোড) |
| `L` | টাইপ কোডের তালিকা |
| `c` | পার্টিশনের নাম (PARTLABEL) বদলাও |
| `o` | নতুন খালি GPT |
| `v` | যাচাই |
| `r` | recovery ও transformation মেনু (MBR→GPT রূপান্তরসহ) |
| `w` | লিখে বের হও |
| `q` | না লিখে বের হও |

```bash
$ sudo gdisk /dev/sdb
GPT fdisk (gdisk) version 1.0.10
...
Command (? for help): n
Partition number (1-128, default 1):
First sector (34-41943006, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-41943006, default = 41940991) or {+-}size{KMGTP}: +10G
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300): 8e00
Changed type of partition to 'Linux LVM'

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/sdb.
The operation has completed successfully.
```

`gdisk` এর একটা বড় সুবিধা: এটা MBR ডিস্ক খুললে ডেটা না মুছেই GPT-তে রূপান্তরের প্রস্তাব দেয় (তবুও আগে ব্যাকআপ রাখবে)। আর `sgdisk` হলো এর স্ক্রিপ্টেবল রূপ, যেমন `sudo sgdisk -n 1:0:+10G -t 1:8300 /dev/sdb`।

### `parted` — বড় ডিস্ক ও স্ক্রিপ্টিং

GNU `parted` MBR ও GPT দুটোই পারে, আর সরাসরি কমান্ড লাইন থেকে চালানো যায়। GUI টুল **GParted** এরই ওপর তৈরি।

> ⚠️ **সতর্কতা:** `parted`-এ কোনো "w" নেই — `mklabel`, `mkpart`, `rm` চাপামাত্রই ডিস্কে লেখা হয়ে যায়।

| কমান্ড (parted-এর ভেতরে) | কাজ |
|--------------------------|-----|
| `help` | সাহায্য |
| `print` | টেবিল দেখাও |
| `print free` | খালি জায়গাসহ দেখাও |
| `mklabel gpt` / `mklabel msdos` | নতুন টেবিল (msdos = MBR) |
| `mkpart NAME FSTYPE START END` | নতুন পার্টিশন (GPT-তে NAME, MBR-এ `primary`/`logical`) |
| `rm N` | N নম্বর পার্টিশন মোছো |
| `resizepart N END` | পার্টিশনের শেষ সীমা বদলাও (শুধু পার্টিশন, ফাইলসিস্টেম না!) |
| `set N esp on` | flag সেট (`esp`, `boot`, `lvm`, `raid`, `swap`) |
| `name N NAME` | GPT পার্টিশনের নাম |
| `unit MiB` | একক বদলাও (`s`, `MiB`, `GiB`, `%`) |
| `align-check optimal N` | alignment ঠিক আছে কিনা |
| `quit` | বের হও |

**এক লাইনে (non-interactive, `-s` = script mode):**

```bash
$ sudo parted -s /dev/sdc mklabel gpt
$ sudo parted -s -a optimal /dev/sdc mkpart data ext4 1MiB 100%
$ sudo parted /dev/sdc print
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdc: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  10.7GB  10.7GB               data
```

`mkpart`-এ যে `ext4` লিখলাম, সেটা শুধু টাইপের ইঙ্গিত — parted ফাইলসিস্টেম **বানায় না**। `File system` কলাম খালি, কারণ এখনো mkfs হয়নি। `1MiB 100%` মানে 1 MiB থেকে শুরু করে ডিস্কের শেষ পর্যন্ত — alignment-এর জন্য সবচেয়ে নিরাপদ লেখা।

> 💡 **টিপস:** parted-এ `GB` (1000³) আর `GiB` (1024³) আলাদা। দ্বিধা এড়াতে সবসময় `MiB`/`GiB` বা `%` ব্যবহার করো।

### কোন টুল কখন?

| পরিস্থিতি | টুল |
|-----------|-----|
| সাধারণ কাজ, MBR বা GPT | `fdisk` |
| শুধু GPT, টাইপ কোড/নাম নিয়ে সূক্ষ্ম কাজ, MBR→GPT রূপান্তর | `gdisk` |
| স্ক্রিপ্ট, পার্টিশন বড় করা, এক লাইনে কাজ | `parted`, `sfdisk`, `sgdisk` |
| মাউস দিয়ে, ডেস্কটপে | GParted, GNOME Disks |
| পুরোনো সিগনেচার মুছে ডিস্ক পরিষ্কার | `wipefs` |

```bash
$ sudo wipefs /dev/sdb               # কী কী সিগনেচার আছে দেখাও (কিছু মোছে না)
$ sudo wipefs -a /dev/sdb            # সব সিগনেচার মুছে দাও — সাবধান!
```

---

## ৩২.৮ ফাইলসিস্টেম তৈরি — `mkfs`

পার্টিশন হলো ফাঁকা তাক। তাতে জিনিস রাখার আগে "সাজানোর পদ্ধতি" বসাতে হবে — এটাই **ফরম্যাট করা** বা **ফাইলসিস্টেম তৈরি**।

### ফাইলসিস্টেম ভেতরে দেখতে কেমন (ext4 উদাহরণ)

```
 পার্টিশন /dev/sdb1
 ┌────────────┬───────────────────────────────┬───────────────────────────────┬─────
 │ Superblock │       Block Group 0           │       Block Group 1           │ ...
 │ (মাস্টার    ├────────┬───────┬──────┬───────┼────────┬───────┬──────┬───────┤
 │  তথ্য:      │ Group  │ Block │Inode │ Inode │ Data   │ ...   │      │       │
 │  আকার,     │ Desc.  │bitmap │bitmap│ Table │ Blocks │       │      │       │
 │  UUID,     │        │       │      │       │ (আসল   │       │      │       │
 │  লেবেল)     │        │       │      │       │ ফাইল)  │       │      │       │
 └────────────┴────────┴───────┴──────┴───────┴────────┴───────┴──────┴───────┘
                         + Journal (একটা বিশেষ inode-এ রাখা লগ)
```

- **Superblock:** পুরো ফাইলসিস্টেমের "আইডি কার্ড" — মোট আকার, block size, UUID, লেবেল, কতবার মাউন্ট হয়েছে। এত গুরুত্বপূর্ণ যে কয়েকটা block group-এ এর ব্যাকআপ কপি থাকে।
- **Bitmap:** কোন block/inode খালি, কোনটা ভরা — হ্যাঁ/না-এর লম্বা তালিকা।
- **Inode table:** প্রতিটা ফাইলের মেটাডেটা (মালিক, পারমিশন, সময়, কোন block-এ ডেটা)।
- **Journal:** বিদ্যুৎ চলে গেলে ফাইলসিস্টেম যাতে এলোমেলো না হয়। কোনো পরিবর্তন করার আগে ext4 সেটা জার্নালে লিখে রাখে ("এখন এই ফাইলের এই অংশ বদলাব")। ক্র্যাশের পর পুরো ডিস্ক না ঘেঁটে শুধু জার্নাল দেখে অসমাপ্ত কাজ শেষ বা বাতিল করে। উপমা: দোকানদার মাল বের করার আগে খাতায় লিখে রাখে — হঠাৎ ঝামেলা হলে খাতা দেখেই বোঝা যায় কোথায় থেমেছিল।

### জনপ্রিয় ফাইলসিস্টেমের তুলনা

| ফাইলসিস্টেম | কোথায় ডিফল্ট | শক্তি | দুর্বলতা/লক্ষ্য রাখো |
|-------------|--------------|-------|-------------------|
| **ext4** | Ubuntu, Debian | পরীক্ষিত, স্থিতিশীল, বড়-ছোট দুটোই করা যায় (shrink অফলাইনে) | inode সংখ্যা তৈরির সময় ঠিক হয়ে যায় |
| **XFS** | RHEL, Rocky, AlmaLinux, CentOS Stream | বড় ফাইল ও সমান্তরাল I/O-তে দ্রুত | **ছোট (shrink) করা যায় না**, শুধু বড় করা যায় |
| **Btrfs** | Fedora Workstation, openSUSE | snapshot, compression, subvolume, checksum | জটিল; RAID5/6 এখনো সুপারিশকৃত নয় |
| **vfat (FAT32)** | EFI পার্টিশন, পুরোনো পেনড্রাইভ | সব OS-এ চলে | একক ফাইল সর্বোচ্চ 4 GiB − 1 byte, পারমিশন নেই |
| **exFAT** | বড় পেনড্রাইভ, SD কার্ড | সব OS-এ চলে, 4 GiB-এর বেশি ফাইল | লিনাক্স পারমিশন নেই, জার্নাল নেই |
| **NTFS** | উইন্ডোজ | উইন্ডোজের সাথে শেয়ার | লিনাক্সে `ntfs3` (kernel 5.15+) বা `ntfs-3g` ড্রাইভার লাগে |
| **tmpfs** | `/tmp` (Fedora, Debian 13), `/run` | RAM-এ থাকে, খুব দ্রুত | রিবুটে সব মুছে যায় |

### `mkfs` কীভাবে কাজ করে

`mkfs` আসলে একটা "সামনের দরজা"। `mkfs -t ext4 /dev/sdb1` লিখলে সে ভেতরে `mkfs.ext4` চালায়। দেখো কোনগুলো ইনস্টল আছে:

```bash
$ ls /usr/sbin/mkfs.*
/usr/sbin/mkfs.bfs  /usr/sbin/mkfs.ext2  /usr/sbin/mkfs.ext4  /usr/sbin/mkfs.minix
/usr/sbin/mkfs.btrfs /usr/sbin/mkfs.ext3 /usr/sbin/mkfs.fat   /usr/sbin/mkfs.vfat
/usr/sbin/mkfs.cramfs /usr/sbin/mkfs.exfat /usr/sbin/mkfs.msdos /usr/sbin/mkfs.xfs
```

কোনো একটা না থাকলে প্যাকেজ ইনস্টল করো ([অধ্যায় ৩০](30-package-management.md)):

| ফাইলসিস্টেম | Debian/Ubuntu প্যাকেজ | Fedora/RHEL প্যাকেজ |
|-------------|----------------------|---------------------|
| ext4 | `e2fsprogs` | `e2fsprogs` |
| XFS | `xfsprogs` | `xfsprogs` |
| vfat | `dosfstools` | `dosfstools` |
| exFAT | `exfatprogs` | `exfatprogs` |
| Btrfs | `btrfs-progs` | `btrfs-progs` |
| NTFS | `ntfs-3g` | `ntfs-3g` (RHEL-এ EPEL থেকে) |

### `mkfs.ext4`

```bash
$ sudo mkfs.ext4 -L DATA /dev/sdb1
mke2fs 1.47.0 (5-Feb-2023)
Creating filesystem with 3932160 4k blocks and 983040 inodes
Filesystem UUID: 8d4f2b6a-3c1e-4f7a-9b2d-5e6f7a8b9c0d
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

আউটপুটে superblock-এর ব্যাকআপ কোথায় আছে লেখা — মূল superblock নষ্ট হলে এগুলো কাজে লাগে (৩২.১৪ দেখো)।

| অপশন | কাজ |
|-------|-----|
| `-L LABEL` | লেবেল দাও (সর্বোচ্চ 16 অক্ষর) |
| `-m 1` | root-এর জন্য সংরক্ষিত জায়গা % (ডিফল্ট 5%) |
| `-b 4096` | block size (byte) |
| `-i 16384` | প্রতি কত byte-এ একটা inode (কম মান = বেশি inode, অনেক ছোট ফাইলের জন্য) |
| `-N 2000000` | সরাসরি inode সংখ্যা |
| `-T largefile` | ব্যবহার-ধরন অনুযায়ী প্রিসেট (`small`, `largefile`, `largefile4`) |
| `-U UUID` | নির্দিষ্ট UUID বসাও |
| `-n` | কিছু না বানিয়ে শুধু দেখাও কী করত (dry run) |
| `-F` | জোর করে (কিছু সতর্কতা উপেক্ষা) |

> 💡 **টিপস:** 5% reserved জায়গা কেন? ডিস্ক পুরো ভরে গেলেও root যেন লগ লিখতে ও সিস্টেম ঠিক করতে পারে, আর ফ্র্যাগমেন্টেশন কম হয়। কিন্তু 4 TB-এর শুধু-ডেটা ডিস্কে 5% মানে 200 GB নষ্ট — সেখানে `-m 1` বা পরে `tune2fs -m 1` দাও। root (`/`) পার্টিশনে কমিয়ো না।

### `mkfs.xfs`

```bash
$ sudo mkfs.xfs -L backup /dev/sdc1
meta-data=/dev/sdc1              isize=512    agcount=4, agsize=655296 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=2621184, imaxpct=25
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

| অপশন | কাজ |
|-------|-----|
| `-L label` | লেবেল (সর্বোচ্চ **12** অক্ষর) |
| `-f` | আগে কোনো ফাইলসিস্টেম থাকলেও ওভাররাইট করো |
| `-b size=4096` | block size |

`agcount=4` — XFS ডিস্ককে কয়েকটা **allocation group**-এ ভাগ করে, প্রতিটা স্বাধীনভাবে কাজ করতে পারে। এজন্যই অনেক CPU একসাথে লিখলে XFS দ্রুত।

> ⚠️ **সতর্কতা:** XFS ছোট করা যায় না। পরে জায়গা কমাতে হলে ব্যাকআপ → নতুন ফাইলসিস্টেম → রিস্টোর ছাড়া উপায় নেই। বড় করতে `xfs_growfs /mountpoint` (মাউন্ট অবস্থায়)।

### `mkfs.vfat` ও `mkfs.exfat`

```bash
$ sudo mkfs.vfat -F 32 -n PENDRIVE /dev/sdd1
mkfs.fat 4.2 (2021-01-31)

$ sudo mkfs.exfat -n BIGUSB /dev/sdd1
exfatprogs version : 1.2.2
Creating exFAT filesystem(/dev/sdd1, cluster size=131072)
...
exFAT format complete!
```

| অপশন (`mkfs.vfat`) | কাজ |
|---------------------|-----|
| `-F 32` | FAT32 (অন্যথায় আকার দেখে FAT12/16/32 ঠিক করে) |
| `-n LABEL` | লেবেল (সর্বোচ্চ 11 অক্ষর, সাধারণত বড় হাতের) |

> ⚠️ **সতর্কতা:** `mkfs` চালালে পার্টিশনের আগের সব ডেটা কার্যত হারিয়ে যায়। চালানোর আগে আরেকবার `lsblk -f` দিয়ে নিশ্চিত হও — বিশেষ করে `/dev/sdb` (পুরো ডিস্ক) আর `/dev/sdb1` (পার্টিশন) গুলিয়ে ফেলো না।

---

## ৩২.৯ মাউন্ট করা — `mount` ও `umount`

### mount point কী

**Mount point** হলো একটা সাধারণ, (সাধারণত) খালি ডিরেক্টরি, যেখানে একটা ফাইলসিস্টেমকে জুড়ে দেওয়া হয়। মাউন্টের পর ওই ডিরেক্টরিতে ঢুকলে তুমি আসলে ওই পার্টিশনের ভেতরে আছ।

```
  মাউন্টের আগে:                      মাউন্টের পরে:  mount /dev/sdb1 /mnt/data
   /                                 /
   └── mnt/                          └── mnt/
       └── data/  (খালি, root         └── data/  ← এখন sdb1-এর "গোড়া"
                  ফাইলসিস্টেমের অংশ)        ├── photos/
                                            ├── movies/
                                            └── lost+found/
```

> 📝 **মনে রাখো:** মাউন্ট পয়েন্টে আগে থেকে কোনো ফাইল থাকলে সেগুলো **মুছে যায় না** — শুধু ঢাকা পড়ে (hidden)। আনমাউন্ট করলেই আবার দেখা যাবে। কখনো "ডিস্ক ভরা দেখাচ্ছে কিন্তু ফাইল পাচ্ছি না" সমস্যা হলে মনে রেখো, কোনো মাউন্টের নিচে লুকানো ফাইল থাকতে পারে।

প্রচলিত জায়গা:

| ডিরেক্টরি | ব্যবহার |
|-----------|---------|
| `/mnt` | অস্থায়ী, হাতে করা মাউন্ট (প্রশাসকের জন্য) |
| `/media/<user>/<label>` | Ubuntu/Debian-এ USB, CD-র অটো-মাউন্ট |
| `/run/media/<user>/<label>` | Fedora/RHEL-এ অটো-মাউন্ট |
| `/srv`, `/data`, `/var/lib/...` | স্থায়ী ডেটা ডিস্ক (নিজের পছন্দমতো) |

### ভেতরে কী হয়

`mount` কমান্ড kernel-কে `mount()` system call দিয়ে বলে: "এই block device-এ এই টাইপের ফাইলসিস্টেম আছে, একে এই ডিরেক্টরিতে বসাও।" kernel তখন ফাইলসিস্টেম ড্রাইভার দিয়ে superblock পড়ে, যাচাই করে, তারপর VFS-এর গাছে ওই ডিরেক্টরির জায়গায় নতুন গাছের গোড়াটা জুড়ে দেয়। সক্রিয় সব মাউন্টের তালিকা kernel রাখে, যা দেখা যায় `/proc/self/mounts` (= `/proc/mounts`) ফাইলে।

### `mount` এর ব্যবহার

**Syntax:** `mount [-t type] [-o options] DEVICE DIRECTORY`

```bash
$ sudo mkdir -p /mnt/data
$ sudo mount /dev/sdb1 /mnt/data
$ findmnt /mnt/data
TARGET    SOURCE    FSTYPE OPTIONS
/mnt/data /dev/sdb1 ext4   rw,relatime
```

`-t` না দিলে `mount` নিজেই `blkid`-এর মতো সিগনেচার পড়ে টাইপ বুঝে নেয়। শুধু `mount` লিখলে সব মাউন্ট দেখায় (অনেক লম্বা, তাই `findmnt` ভালো)।

গুরুত্বপূর্ণ অপশন:

| অপশন | কাজ |
|-------|-----|
| `-t ext4` | ফাইলসিস্টেম টাইপ নির্দিষ্ট করো |
| `-o ro` | শুধু-পড়া (read-only) |
| `-o rw` | পড়া-লেখা |
| `-o remount,rw /` | মাউন্ট না খুলেই অপশন বদলাও |
| `-o loop` | একটা ইমেজ ফাইল (যেমন `.iso`) মাউন্ট |
| `-o noexec` | এখান থেকে কোনো প্রোগ্রাম চালানো যাবে না |
| `-o nosuid` | setuid বিট কাজ করবে না ([অধ্যায় ১৭](17-special-permissions-acl.md)) |
| `-o nodev` | device ফাইল কাজ করবে না |
| `-o noatime` | পড়ার সময় access time আপডেট হবে না (কম লেখা) |
| `-o uid=1000,gid=1000` | FAT/exFAT/NTFS-এ সব ফাইলের মালিক কে হবে |
| `-o umask=022` | FAT/exFAT-এ পারমিশন মাস্ক |
| `--bind /src /dst` | একটা ডিরেক্টরিকে আরেক জায়গায় "আয়না" হিসেবে দেখাও |
| `-a` | `/etc/fstab`-এর সব (যেগুলোতে `noauto` নেই) মাউন্ট করো |
| `-v` | বিস্তারিত বলো |
| `-L DATA` / `-U <uuid>` | লেবেল/UUID দিয়ে ডিভাইস বোঝাও |

উদাহরণ:

```bash
# ISO ফাইলের ভেতর দেখা
$ sudo mkdir /mnt/iso
$ sudo mount -o loop,ro ubuntu-24.04.iso /mnt/iso
mount: /mnt/iso: WARNING: source write-protected, mounted read-only.

# সিস্টেম রিকভারির সময় root-কে লেখার উপযোগী করা
$ sudo mount -o remount,rw /

# UUID দিয়ে
$ sudo mount UUID=8d4f2b6a-3c1e-4f7a-9b2d-5e6f7a8b9c0d /mnt/data
```

> 📝 **মনে রাখো:** `relatime` (ডিফল্ট) হলো মাঝামাঝি সমাধান — access time তখনই আপডেট হয় যখন সেটা modify time-এর চেয়ে পুরোনো বা ২৪ ঘণ্টার বেশি পুরোনো। তাই বেশিরভাগ ক্ষেত্রে `noatime` না দিলেও চলে।

### `umount` — খুলে ফেলা

বানান খেয়াল করো: `umount`, **`unmount` না**।

```bash
$ sudo umount /mnt/data          # mount point দিয়ে
$ sudo umount /dev/sdb1          # অথবা device দিয়ে
```

আনমাউন্ট করার সময় kernel page cache-এর সব অলেখা (dirty) ডেটা ডিস্কে লিখে দেয় — তাই এটা নিরাপদে ডিস্ক খোলার আসল ধাপ।

**সবচেয়ে প্রচলিত ত্রুটি:**

```bash
$ sudo umount /mnt/data
umount: /mnt/data: target is busy.
```

মানে কোনো প্রসেস এখনো ভেতরের কোনো ফাইল খোলা রেখেছে, বা কোনো শেলের current directory ওটার ভেতরে। খুঁজে বের করো কে:

```bash
$ sudo fuser -vm /mnt/data
                     USER        PID ACCESS COMMAND
/mnt/data:           root     kernel mount /mnt/data
                     rahim      4821 ..c.. bash
                     rahim      5102 F.... vlc

$ sudo lsof +f -- /mnt/data
COMMAND  PID  USER   FD   TYPE DEVICE  SIZE/OFF NODE NAME
bash    4821 rahim  cwd    DIR   8,17      4096    2 /mnt/data
vlc     5102 rahim   23r   REG   8,17 734003200   15 /mnt/data/movies/a.mkv
```

`c` = current directory, `F` = খোলা ফাইল। সমাধান: ওই টার্মিনালে `cd ~` করো, vlc বন্ধ করো, তারপর আবার `umount`। (প্রসেস বন্ধ করার বিস্তারিত [অধ্যায় ২৪](24-process-management.md)।)

| অপশন | কাজ |
|-------|-----|
| `umount -l` | lazy: এখনই গাছ থেকে আলাদা করো, ব্যবহার শেষ হলে পুরোপুরি খোলো |
| `umount -f` | force (মূলত সাড়া না দেওয়া NFS-এর জন্য) |
| `umount -R /mnt/x` | ভেতরের সব নেস্টেড মাউন্টসহ (recursive) |
| `fuser -km /mnt/data` | ওই মাউন্ট ব্যবহারকারী সব প্রসেস **মেরে ফেলো** — সাবধান! |

> ⚠️ **সতর্কতা:** `umount -l` দিয়ে খোলার পর USB টেনে খুলো না — ডেটা তখনো পুরো লেখা নাও হতে পারে। lazy umount জরুরি অবস্থার জন্য, রোজকার অভ্যাসের জন্য না।

---

## ৩২.১০ `/etc/fstab` — বুটের সময় স্বয়ংক্রিয় মাউন্ট

`mount` দিয়ে যা মাউন্ট করো, রিবুটের পর তা থাকে না। স্থায়ী করতে হলে `/etc/fstab` (filesystem table) ফাইলে লিখতে হয়। বুটের সময় systemd এই ফাইল পড়ে প্রতিটা লাইনের জন্য একটা `.mount` unit বানায় (`systemd-fstab-generator` দিয়ে) এবং মাউন্ট করে। ([অধ্যায় ২৬](26-systemd-services.md) ও [অধ্যায় ২৭](27-boot-process.md) দেখো।)

```bash
$ cat /etc/fstab
# <file system>                            <mount point>  <type>  <options>          <dump> <pass>
UUID=3f2a9c1e-7b4d-4e8a-9d61-2c5b8e0f1a77  /              ext4    errors=remount-ro  0      1
UUID=A1B2-C3D4                             /boot/efi      vfat    umask=0077         0      1
/swap.img                                  none           swap    sw                 0      0
```

প্রতিটা লাইনে **৬টা ফিল্ড**, ফাঁকা জায়গা (space/tab) দিয়ে আলাদা:

```
 UUID=8d4f...9c0d   /mnt/data   ext4   defaults,noatime   0   2
 ───────┬────────   ────┬────   ──┬─   ────────┬───────   ┬   ┬
        │               │         │            │          │   └─ ৬. pass (fsck ক্রম)
        │               │         │            │          └───── ৫. dump
        │               │         │            └──────────────── ৪. options
        │               │         └───────────────────────────── ৩. type
        │               └─────────────────────────────────────── ২. mount point
        └─────────────────────────────────────────────────────── ১. device
```

**১. Device (fs_spec):** কোন ডিভাইস। লেখার উপায়:

| ধরন | উদাহরণ | মন্তব্য |
|------|--------|---------|
| UUID | `UUID=8d4f2b6a-...` | **সবচেয়ে প্রচলিত ও নিরাপদ** |
| লেবেল | `LABEL=DATA` | পড়তে সহজ, কিন্তু দুই ডিস্কে একই লেবেল হলে সমস্যা |
| PARTUUID | `PARTUUID=c4d5e6f7-...` | ফরম্যাট করলেও বদলায় না |
| ডিভাইস পাথ | `/dev/sdb1` | **এড়িয়ে চলো** — নাম বদলাতে পারে |
| LVM | `/dev/mapper/vg0-data` | LVM নাম স্থায়ী, তাই চলে ([অধ্যায় ৩৩](33-lvm-raid.md)) |
| নেটওয়ার্ক | `server:/export` (NFS), `//server/share` (CIFS) | |

**UUID কেন?** আগেই বলেছি, `sdb` কাল `sdc` হয়ে যেতে পারে। তখন fstab-এ `/dev/sdb1` থাকলে হয় মাউন্ট ব্যর্থ, নয়তো ভুল ডিস্ক ভুল জায়গায় মাউন্ট হবে। UUID ফাইলসিস্টেমের ভেতরেই লেখা থাকে, তাই ডিস্ক যেকোনো পোর্টে লাগাও, একই থাকবে। উপমা: `sdb` হলো ক্লাসের রোল নম্বর (প্রতি বছর বদলায়), UUID হলো জন্ম নিবন্ধন নম্বর।

**২. Mount point (fs_file):** কোথায় মাউন্ট হবে। ডিরেক্টরিটা আগে থেকে থাকতে হবে। swap-এর জন্য `none`।

**৩. Type (fs_vfstype):** `ext4`, `xfs`, `vfat`, `exfat`, `btrfs`, `ntfs3`, `swap`, `nfs`, `tmpfs` বা `auto`।

**৪. Options (fs_mntops):** কমা দিয়ে আলাদা, ফাঁকা জায়গা ছাড়া। `mount -o`-এর সব অপশন চলে, সাথে এগুলো:

| অপশন | মানে |
|-------|------|
| `defaults` | `rw,suid,dev,exec,auto,nouser,async` এর সমষ্টি |
| `auto` / `noauto` | `mount -a` ও বুটে মাউন্ট হবে / হবে না |
| `user` | সাধারণ ইউজারও মাউন্ট করতে পারবে (সাথে নিজে `noexec,nosuid,nodev` যোগ করে) |
| `nofail` | ডিভাইস না পেলে বুট আটকাবে না — **বাইরের/ডেটা ডিস্কে অবশ্যই দাও** |
| `x-systemd.automount` | প্রথমবার কেউ ডিরেক্টরিতে ঢুকলে তখন মাউন্ট (বুট দ্রুত হয়) |
| `x-systemd.device-timeout=10s` | ডিভাইসের জন্য কতক্ষণ অপেক্ষা (ডিফল্ট 90 সেকেন্ড) |
| `_netdev` | নেটওয়ার্ক চালু হওয়ার পর মাউন্ট (iSCSI ইত্যাদিতে) |
| `errors=remount-ro` | (ext4) ত্রুটি পেলে read-only করে দাও |
| `sw` | swap লাইনে প্রচলিত (কার্যত `defaults`-এর মতো) |

**৫. Dump (fs_freq):** পুরোনো `dump` ব্যাকআপ টুলের জন্য। আজকাল প্রায় সবসময় `0`।

**৬. Pass (fs_passno):** বুটের সময় `fsck` কোন ক্রমে চলবে।

| মান | মানে |
|------|------|
| `0` | fsck চালাবে না (swap, XFS, Btrfs, নেটওয়ার্ক) |
| `1` | প্রথমে — শুধু root (`/`) |
| `2` | root-এর পরে — বাকি ext4 ও vfat পার্টিশন |

> 💡 **টিপস:** XFS ও Btrfs-এ pass `0` দাও। এদের মেরামতের নিজস্ব পদ্ধতি আছে, বুট-টাইম fsck এদের জন্য কিছু করে না (`fsck.xfs` আসলে কিছুই করে না, শুধু `xfs_repair` ব্যবহারের পরামর্শ দেয়)।

### ধাপে ধাপে: নতুন ডিস্ক fstab-এ যোগ করা

```bash
# ১. UUID বের করো
$ sudo blkid -s UUID -o value /dev/sdb1
8d4f2b6a-3c1e-4f7a-9b2d-5e6f7a8b9c0d

# ২. mount point বানাও
$ sudo mkdir -p /mnt/data

# ৩. fstab-এর ব্যাকআপ রাখো
$ sudo cp /etc/fstab /etc/fstab.bak

# ৪. লাইন যোগ করো (অথবা nano/vim দিয়ে)
$ echo 'UUID=8d4f2b6a-3c1e-4f7a-9b2d-5e6f7a8b9c0d /mnt/data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab

# ৫. যাচাই করো — রিবুটের আগে এটা বাধ্যতামূলক!
$ sudo findmnt --verify
Success, no errors or warnings detected

# ৬. systemd-কে নতুন fstab জানাও
$ sudo systemctl daemon-reload

# ৭. fstab-এর সব মাউন্ট করো
$ sudo mount -a
$ df -h /mnt/data
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        15G   24K   14G   1% /mnt/data
```

`mount -a` কোনো আউটপুট না দিলে মানে সফল। ভুল থাকলে এখানেই ত্রুটি দেখাবে — রিবুটের পরে না। (`tee` কেন লাগল, `sudo echo >>` কেন কাজ করে না, তা [অধ্যায় ১৮](18-sudo-root.md) ও [অধ্যায় ১৯](19-redirection-pipe.md)-এ আছে।)

> ⚠️ **সতর্কতা:** fstab-এ ভুল (যেমন ভুল UUID) রেখে `nofail` ছাড়া রিবুট করলে সিস্টেম **emergency mode**-এ আটকে যেতে পারে: *"You are in emergency mode..."*। তখন root পাসওয়ার্ড দিয়ে ঢুকে `mount -o remount,rw /` করো, `/etc/fstab` ঠিক করো (বা `/etc/fstab.bak` ফেরত আনো), তারপর `systemctl daemon-reload` ও `reboot`। Ubuntu-তে root লক থাকলে GRUB মেনু থেকে recovery mode বা লাইভ USB লাগবে। তাই সবসময় `findmnt --verify` আর `mount -a` দিয়ে আগে পরীক্ষা করো।

---

## ৩২.১১ জায়গা মাপা — `df`, `du`, `ncdu`

### `df` — ফাইলসিস্টেম কতটা ভরা (disk free)

`df` প্রতিটা মাউন্ট করা **ফাইলসিস্টেমের** superblock থেকে মোট/ব্যবহৃত/খালি হিসাব পড়ে — খুব দ্রুত।

```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           1.6G  2.1M  1.6G   1% /run
/dev/nvme0n1p3  466G   63G  380G  15% /
tmpfs           7.8G     0  7.8G   0% /dev/shm
/dev/nvme0n1p2  2.0G  204M  1.6G  12% /boot
/dev/nvme0n1p1  1.1G  6.2M  1.1G   1% /boot/efi
/dev/sdb1        15G   24K   14G   1% /mnt/data
```

| অপশন | কাজ |
|-------|-----|
| `-h` | মানুষের পড়ার মতো (K, M, G — 1024 ভিত্তিক) |
| `-H` | 1000 ভিত্তিক (ডিস্ক বিক্রেতার হিসাব) |
| `-T` | ফাইলসিস্টেম টাইপ কলাম |
| `-i` | inode ব্যবহার (byte না) |
| `-x tmpfs -x devtmpfs` | এই টাইপগুলো বাদ দাও |
| `-t ext4` | শুধু এই টাইপ |
| `--total` | শেষে মোট যোগফল |
| `df -h /var` | `/var` যে ফাইলসিস্টেমে, শুধু সেটা |

খেয়াল করো: `/mnt/data`-তে Size 15G, Used 24K, Avail 14G — যোগ মিলছে না! কারণ ওই 5% reserved জায়গা (root-এর জন্য) Avail-এ ধরা হয় না।

**inode ফুরিয়ে যাওয়া:** কখনো `df -h` বলে 40% খালি, কিন্তু ফাইল বানাতে গেলে *"No space left on device"*। তখন `df -i` দেখো — লাখ লাখ ছোট ফাইলে (যেমন session/cache ফাইল) inode শেষ হয়ে গেছে:

```bash
$ df -hi /
Filesystem     Inodes IUsed IFree IUse% Mounted on
/dev/nvme0n1p3    30M   30M     0  100% /
```

### `du` — কোন ফোল্ডার কতটা জায়গা নিচ্ছে (disk usage)

`du` ডিরেক্টরির ভেতরে ঢুকে প্রতিটা ফাইলের আকার যোগ করে — বড় ডিরেক্টরিতে সময় লাগে।

```bash
$ du -sh ~/Downloads
4.7G	/home/rahim/Downloads

$ sudo du -h -d 1 /var | sort -h
16K	/var/mail
4.0K	/var/opt
...
1.2G	/var/cache
3.8G	/var/log
5.9G	/var/lib
11G	/var
```

| অপশন | কাজ |
|-------|-----|
| `-s` | শুধু মোট (summary) |
| `-h` | মানুষের পড়ার মতো |
| `-d 1` / `--max-depth=1` | এক স্তর গভীর পর্যন্ত |
| `-a` | ডিরেক্টরির সাথে ফাইলগুলোও দেখাও |
| `-x` | অন্য ফাইলসিস্টেমে ঢুকবে না (যেমন `/`-এ চালালে `/proc`, `/mnt` বাদ) |
| `-c` | শেষে মোট যোগফল |
| `--apparent-size` | ফাইলের "দেখানো" আকার, ডিস্কে আসলে কত নিয়েছে তা না |
| `--exclude='*.iso'` | প্যাটার্ন বাদ |

**সবচেয়ে কাজের কম্বো** (sort-এর বিস্তারিত [অধ্যায় ২৩](23-text-tools.md)):

```bash
$ du -sh * | sort -h | tail -5        # বর্তমান ফোল্ডারের সবচেয়ে বড় ৫টা
$ sudo du -xh / 2>/dev/null | sort -h | tail -20
```

### `df` আর `du` আলাদা সংখ্যা দেখায় কেন?

এটা ইন্টারভিউ আর বাস্তবে খুব প্রচলিত প্রশ্ন।

```
  df: "ফাইলসিস্টেমের হিসাবখাতা (superblock) কী বলছে?"
  du: "আমি গাছে হেঁটে যতগুলো ফাইল দেখতে পাচ্ছি, তাদের যোগফল কত?"
```

কারণগুলো:

1. **মুছে ফেলা কিন্তু খোলা ফাইল:** কোনো প্রসেস (যেমন nginx) একটা 10 GB লগ খোলা রেখেছে, আর তুমি `rm` দিয়ে মুছলে। নাম চলে গেল (তাই `du` দেখে না), কিন্তু inode আর ডেটা থাকে যতক্ষণ প্রসেসটা ফাইল খোলা রাখে (তাই `df` দেখায় ভরা)। খুঁজতে: `sudo lsof +L1`। সমাধান: সার্ভিস রিস্টার্ট ([অধ্যায় ২৬](26-systemd-services.md))।
2. **মাউন্টের নিচে লুকানো ফাইল** (৩২.৯-এ বলেছি)।
3. **Reserved block** আর ফাইলসিস্টেমের নিজের মেটাডেটা।
4. **Sparse file:** যে ফাইলের ভেতরে "ফাঁকা" অংশ ডিস্কে লেখা হয় না (VM ডিস্ক ইমেজ)। `ls -l` বড় দেখায়, `du` ছোট।

> 💡 **টিপস:** লগ ফাইল খালি করতে `rm` না করে `sudo truncate -s 0 /var/log/big.log` করো — তাহলে প্রসেস খোলা রাখলেও জায়গা সাথে সাথে খালি হবে।

### `ncdu` — ইন্টারঅ্যাকটিভ du

`du`-এর রঙিন, মেনু-ভিত্তিক রূপ। কোথায় জায়গা গেল খুঁজতে সবচেয়ে আরামদায়ক।

```bash
$ sudo apt install ncdu        # Debian/Ubuntu
$ sudo dnf install ncdu        # Fedora (RHEL-এ EPEL থেকে)
$ ncdu ~
$ sudo ncdu -x /               # শুধু root ফাইলসিস্টেম
```

```
ncdu 1.19 ~ Use the arrow keys to navigate, press ? for help
--- /home/rahim -----------------------------------------------------------
    4.7 GiB [##########] /Downloads
    2.1 GiB [####      ] /.cache
  812.4 MiB [#         ] /Videos
  120.0 MiB [          ] /.local
 Total disk usage:   7.8 GiB  Apparent size:   7.6 GiB  Items: 48213
```

| কী (`ncdu`-এর ভেতরে) | কাজ |
|---------------------|-----|
| `↑` `↓` বা `k` `j` | উপরে-নিচে |
| `Enter` বা `→` / `l` | ফোল্ডারে ঢোকো |
| `←` বা `h` | এক স্তর উপরে |
| `d` | নির্বাচিতটা মুছে ফেলো (নিশ্চিতকরণ চাইবে) |
| `s` | আকার অনুযায়ী সাজাও |
| `n` | নাম অনুযায়ী সাজাও |
| `C` | আইটেম সংখ্যা অনুযায়ী সাজাও |
| `a` | ডিস্কে আকার ↔ apparent size টগল |
| `g` | শতাংশ/গ্রাফ দেখানোর ধরন বদলাও |
| `e` | লুকানো/বাদ দেওয়া ফাইল দেখাও/লুকাও |
| `i` | নির্বাচিত আইটেমের তথ্য |
| `r` | আবার স্ক্যান করো |
| `?` | সাহায্য |
| `q` | বের হও |

| অপশন (কমান্ড লাইনে) | কাজ |
|----------------------|-----|
| `-x` | অন্য ফাইলসিস্টেমে যাবে না |
| `--exclude PATTERN` | বাদ দাও |
| `-o file` / `-f file` | স্ক্যান ফলাফল ফাইলে রাখো / পরে খোলো |
| `-r` | read-only, মোছা বন্ধ (নিরাপদ) |

---

## ৩২.১২ Swap — RAM-এর "বাড়তি তাক"

### Swap কী ও কেন

RAM ভরে গেলে kernel কম ব্যবহৃত মেমোরি পেজ (page, সাধারণত 4 KiB টুকরো) ডিস্কের একটা বিশেষ জায়গায় সরিয়ে রাখে — এটাই **swap**। পরে দরকার হলে আবার RAM-এ আনে।

উপমা: তোমার পড়ার টেবিল (RAM) ছোট। যে বইগুলো এখন পড়ছ না, সেগুলো পাশের তাকে (swap) রাখলে। দরকার হলে আবার আনতে সময় লাগে (ডিস্ক RAM-এর চেয়ে অনেক ধীর), কিন্তু অন্তত টেবিলে জায়গা ফুরিয়ে কাজ থেমে যায় না।

```
   ┌──────────── RAM ────────────┐          ┌──── Swap (ডিস্ক) ────┐
   │ Firefox │ Code │ এখন দরকারি  │ ─swap→   │ কাল খোলা সেই ট্যাব   │
   │         │      │             │ ←page in │ ঘুমন্ত প্রসেস         │
   └─────────────────────────────┘          └──────────────────────┘
       RAM পুরো ভরলে এবং swap-ও না থাকলে → OOM killer কোনো প্রসেস মেরে ফেলে
```

Swap কেন রাখবে:

- RAM ফুরিয়ে গেলে হঠাৎ প্রোগ্রাম মারা যাওয়া (OOM killer) ঠেকায়।
- অনেকদিন অব্যবহৃত মেমোরি সরিয়ে RAM-কে file cache-এর জন্য খালি করে।
- **Hibernate** (suspend to disk)-এর জন্য RAM-এর সমান বা বড় swap partition/file লাগে।

কত বড়? কঠিন নিয়ম নেই। সাধারণ পরামর্শ: RAM ≤ 2 GB হলে RAM-এর দ্বিগুণ; 2–8 GB হলে RAM-এর সমান; 8 GB-এর বেশি হলে 4–8 GB যথেষ্ট (hibernate চাইলে RAM-এর সমান বা বেশি)।

**Distro-ভেদে ডিফল্ট:**

| Distro | ডিফল্ট swap |
|--------|-------------|
| Ubuntu 24.04 | `/swap.img` নামে একটা swapfile |
| Debian 12 | ইনস্টলারে আলাদা swap partition |
| Fedora (33+) | **zram** — RAM-এর ভেতরেই কম্প্রেসড swap (`zram-generator`) |
| RHEL 9 | LVM-এর ভেতরে swap logical volume |

### বর্তমান swap দেখা

```bash
$ swapon --show
NAME      TYPE  SIZE USED PRIO
/swap.img file    4G 512M   -2

$ free -h
               total        used        free      shared  buff/cache   available
Mem:            15Gi       6.2Gi       2.1Gi       410Mi       7.4Gi       8.9Gi
Swap:          4.0Gi       512Mi       3.5Gi
```

(`free` আর মেমোরি বিশ্লেষণের বিস্তারিত [অধ্যায় ৪৬](46-performance-troubleshooting.md)-এ।)

### পদ্ধতি ১: Swap partition

ধরো ৩২.৭-এ বানানো `/dev/sdb2` (টাইপ Linux swap):

```bash
$ sudo mkswap -L myswap /dev/sdb2
Setting up swapspace version 1, size = 2 GiB (2147479552 bytes)
LABEL=myswap, UUID=1e2d3c4b-5a69-4788-97a6-b5c4d3e2f100

$ sudo swapon /dev/sdb2
$ swapon --show
NAME      TYPE      SIZE USED PRIO
/swap.img file        4G 512M   -2
/dev/sdb2 partition   2G   0B   -3
```

স্থায়ী করতে `/etc/fstab`-এ:

```
UUID=1e2d3c4b-5a69-4788-97a6-b5c4d3e2f100  none  swap  defaults  0  0
```

### পদ্ধতি ২: Swapfile (নতুন পার্টিশন ছাড়াই)

```bash
# ১. 2 GiB-এর ফাইল বানাও (দ্রুত পদ্ধতি)
$ sudo fallocate -l 2G /swapfile

#    fallocate সমর্থন না করলে বা swapon ত্রুটি দিলে, ধীর কিন্তু নিশ্চিত পদ্ধতি:
$ sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress

# ২. শুধু root পড়তে-লিখতে পারবে — নিরাপত্তার জন্য জরুরি
$ sudo chmod 600 /swapfile

# ৩. swap হিসেবে প্রস্তুত করো
$ sudo mkswap /swapfile
Setting up swapspace version 1, size = 2 GiB (2147479552 bytes)
no label, UUID=7a6b5c4d-3e2f-4a1b-9c8d-7e6f5a4b3c2d

# ৪. চালু করো
$ sudo swapon /swapfile

# ৫. স্থায়ী করো
$ echo '/swapfile none swap defaults 0 0' | sudo tee -a /etc/fstab
```

> ⚠️ **সতর্কতা:** swapfile-এ RAM-এর যেকোনো কিছু (পাসওয়ার্ড, কী) লেখা হতে পারে। তাই `chmod 600` বাধ্যতামূলক — না দিলে `swapon` সতর্কবার্তা দেয়: *"insecure permissions 0644, fix with: chmod 0600 /swapfile"*।

> ⚠️ **সতর্কতা (Btrfs):** Btrfs-এ সাধারণ `fallocate` দিয়ে swapfile চলে না (copy-on-write ও compression বন্ধ করতে হয়)। btrfs-progs 6.1+ থাকলে লেখো: `sudo btrfs filesystem mkswapfile --size 2G /swap/swapfile`।

**swap বন্ধ ও মুছে ফেলা:**

```bash
$ sudo swapoff /swapfile        # swap-এ থাকা পেজগুলো RAM-এ ফেরত আনে, তারপর বন্ধ
$ sudo rm /swapfile             # তারপর fstab থেকেও লাইনটা মুছে দাও
$ sudo swapoff -a               # সব swap বন্ধ
$ sudo swapon -a                # fstab-এর সব swap চালু
```

| কমান্ড/অপশন | কাজ |
|-------------|-----|
| `mkswap DEV` | swap সিগনেচার লেখো |
| `mkswap -L LABEL` | লেবেলসহ |
| `swapon DEV` | চালু |
| `swapon -p 10 DEV` | priority (বেশি সংখ্যা আগে ব্যবহার হবে) |
| `swapon --show` | সক্রিয় swap |
| `swapon -a` / `swapoff -a` | fstab-এর সব চালু / সব বন্ধ |
| `cat /proc/swaps` | kernel-এর নিজের তালিকা |

### Swappiness — kernel কতটা "swap-প্রিয়"

`vm.swappiness` (0–200, ডিফল্ট **60**) বলে দেয় RAM-এর চাপ এলে kernel কতটা আগ্রহে অ্যাপের মেমোরি (anonymous page) swap-এ পাঠাবে, ফাইল cache ফেলে দেওয়ার তুলনায়।

- **কম মান (যেমন 10):** kernel যতটা পারে অ্যাপকে RAM-এ রাখে, বরং ফাইল cache ছাড়ে। ডেস্কটপ ও ডাটাবেস সার্ভারে প্রচলিত।
- **বেশি মান (100+):** swap নির্দ্বিধায় ব্যবহার। zram-এর মতো দ্রুত swap-এ যুক্তিসংগত (Fedora-র zram সেটআপ নিজেই এটা বাড়িয়ে দিতে পারে)।
- **0:** swap প্রায় এড়ায়, কিন্তু একেবারে বন্ধ করে না।

```bash
$ cat /proc/sys/vm/swappiness
60
$ sysctl vm.swappiness
vm.swappiness = 60

# এখনই বদলাও (রিবুটে হারাবে)
$ sudo sysctl vm.swappiness=10
vm.swappiness = 10

# স্থায়ী করো
$ echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
$ sudo sysctl --system           # সব sysctl কনফিগ আবার লোড করো
```

> 📝 **মনে রাখো:** swappiness "RAM কত % ভরলে swap শুরু হবে" — এই প্রচলিত ধারণাটা **ভুল**। এটা একটা তুলনামূলক ওজন (anonymous memory বনাম file cache কোনটা আগে ছাড়া হবে)। `/proc/sys` ও sysctl নিয়ে আরও আছে [অধ্যায় ৪৭](47-kernel-proc-sys.md)-এ।

---

## ৩২.১৩ ফাইলসিস্টেম পরীক্ষা ও মেরামত — `fsck`, `e2fsck`, `xfs_repair`

হঠাৎ বিদ্যুৎ চলে যাওয়া, USB টেনে খোলা বা ডিস্কের সমস্যায় ফাইলসিস্টেমের মেটাডেটা (bitmap, inode, directory) অসঙ্গত হয়ে যেতে পারে। `fsck` (file system check) এগুলো খুঁজে ঠিক করে। উপমা: লাইব্রেরির ক্যাটালগ মিলিয়ে দেখা — যে বই তাকে আছে কিন্তু ক্যাটালগে নেই, বা ক্যাটালগে আছে কিন্তু তাকে নেই।

> ⚠️ **সবচেয়ে বড় সতর্কতা:** **মাউন্ট করা ফাইলসিস্টেমে কখনো মেরামতকারী fsck চালাবে না** — বাস্তবে ফাইলসিস্টেম আরও নষ্ট হয়ে যেতে পারে। আগে `umount`। root (`/`) পরীক্ষা করতে হলে বুটের সময়, রিকভারি মোডে বা লাইভ USB থেকে করো।

### `fsck` — সামনের দরজা

`mkfs`-এর মতোই, `fsck` টাইপ দেখে ভেতরে `fsck.ext4`, `fsck.vfat` ইত্যাদি চালায়।

```bash
$ sudo umount /dev/sdb1
$ sudo fsck /dev/sdb1
fsck from util-linux 2.39.3
e2fsck 1.47.0 (5-Feb-2023)
DATA: clean, 11/983040 files, 87309/3932160 blocks
```

"clean" মানে শেষবার ঠিকঠাক আনমাউন্ট হয়েছিল, তাই পুরো পরীক্ষা হয়নি। জোর করে পুরো পরীক্ষা:

```bash
$ sudo fsck -f /dev/sdb1
fsck from util-linux 2.39.3
e2fsck 1.47.0 (5-Feb-2023)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
DATA: 11/983040 files (0.0% non-contiguous), 87309/3932160 blocks
```

ext4-এর এই ৫টা ধাপ: (১) প্রতিটা inode ও তার block, (২) ডিরেক্টরির গঠন, (৩) সব ডিরেক্টরি root থেকে পৌঁছানো যায় কিনা, (৪) প্রতিটা inode-এর link count ঠিক আছে কিনা, (৫) bitmap ও summary হিসাব মেলে কিনা। কোনো মালিকহীন ফাইল পেলে সেগুলো `lost+found` ডিরেক্টরিতে রেখে দেয় (inode নম্বর দিয়ে নাম)।

| অপশন | কাজ |
|-------|-----|
| `-f` | "clean" হলেও জোর করে পরীক্ষা (e2fsck-এর অপশন, fsck পাস করে দেয়) |
| `-n` | কিছু বদলাবে না, শুধু রিপোর্ট — **সবচেয়ে নিরাপদ প্রথম পদক্ষেপ** |
| `-y` | সব প্রশ্নে স্বয়ংক্রিয়ভাবে "yes" |
| `-p` / `-a` | নিরাপদ সমস্যাগুলো প্রশ্ন ছাড়াই ঠিক করো (preen) |
| `-C 0` | progress bar (fsck-এর অপশন) |
| `-A` | fstab-এর সব (pass ক্রমে) |
| `-t ext4` | টাইপ নির্দিষ্ট |
| `-N` | কী চালাত তা দেখাও, চালাবে না |

**Exit code** (স্ক্রিপ্টে কাজে লাগে; একাধিক হলে যোগ হয়):

| কোড | মানে |
|------|------|
| `0` | কোনো ত্রুটি নেই |
| `1` | ত্রুটি পাওয়া গেছে ও ঠিক করা হয়েছে |
| `2` | ঠিক করা হয়েছে, রিবুট দরকার |
| `4` | ত্রুটি রয়ে গেছে, ঠিক হয়নি |
| `8` | অপারেশনাল ত্রুটি |
| `16` | ব্যবহারে ভুল (syntax) |
| `32` | ইউজার বাতিল করেছে |
| `128` | shared library ত্রুটি |

### `e2fsck` — ext2/3/4-এর আসল মেরামতকারী

```bash
$ sudo e2fsck -f -y /dev/sdb1      # জোর করে পরীক্ষা, সব ঠিক করো
$ sudo e2fsck -n /dev/sdb1         # শুধু দেখো
$ sudo e2fsck -c /dev/sdb1         # badblocks দিয়ে খারাপ sector খুঁজে চিহ্নিত করো (ধীর)
```

**মূল superblock নষ্ট হলে** (*"Bad magic number in super-block"*) — ব্যাকআপ superblock দিয়ে চেষ্টা:

```bash
$ sudo mke2fs -n /dev/sdb1          # -n: শুধু দেখায় ব্যাকআপ কোথায়, কিছুই বানায় না!
...
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, ...

$ sudo e2fsck -b 32768 /dev/sdb1
```

> ⚠️ **সতর্কতা:** `mke2fs -n`-এর `-n` ভুলে বাদ দিলে নতুন ফাইলসিস্টেম তৈরি হয়ে যাবে, সব ডেটা যাবে! আর আসল mkfs-এর সময় যে block size ব্যবহার হয়েছিল, সেটা না মিললে ব্যাকআপের অবস্থান ভুল দেখাবে। সন্দেহ থাকলে `sudo dumpe2fs /dev/sdb1 | grep -i superblock` চেষ্টা করো (যদি পড়তে পারে)।

### `xfs_repair` — XFS-এর জন্য

```bash
$ sudo umount /dev/sdc1
$ sudo xfs_repair -n /dev/sdc1      # শুধু পরীক্ষা
$ sudo xfs_repair /dev/sdc1         # মেরামত
```

যদি বলে log-এ অসমাপ্ত পরিবর্তন আছে, প্রথমে একবার মাউন্ট-আনমাউন্ট করে log replay করাও। সেটাও না পারলে শেষ উপায় `xfs_repair -L` (log মুছে ফেলে — কিছু সাম্প্রতিক পরিবর্তন হারাতে পারে)।

### বুটের সময় fsck

systemd-চালিত সিস্টেমে বুটে fsck চালায় `systemd-fsck@.service`, fstab-এর pass ফিল্ড মেনে। জোর করে পরের বুটে সব পরীক্ষা করাতে GRUB-এ kernel প্যারামিটার দাও `fsck.mode=force` (আর প্রশ্ন ছাড়া ঠিক করাতে `fsck.repair=yes`)। ([অধ্যায় ২৭](27-boot-process.md)-এ kernel parameter দেওয়ার পদ্ধতি দেখো।)

```bash
$ journalctl -u 'systemd-fsck*' -b       # এই বুটের fsck লগ
```

---

## ৩২.১৪ `tune2fs` — ext4-এর সেটিং বদলানো (ফরম্যাট ছাড়াই)

`tune2fs` দিয়ে একটা বিদ্যমান ext2/3/4 ফাইলসিস্টেমের প্যারামিটার দেখা ও বদলানো যায়। বেশিরভাগ পরিবর্তন মাউন্ট অবস্থাতেই চলে।

```bash
$ sudo tune2fs -l /dev/sdb1 | head -20
tune2fs 1.47.0 (5-Feb-2023)
Filesystem volume name:   DATA
Last mounted on:          /mnt/data
Filesystem UUID:          8d4f2b6a-3c1e-4f7a-9b2d-5e6f7a8b9c0d
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype extent 64bit flex_bg sparse_super large_file huge_file dir_nlink extra_isize metadata_csum
Default mount options:    user_xattr acl
Filesystem state:         clean
Errors behavior:          Continue
Inode count:              983040
Block count:              3932160
Reserved block count:     196608
...
Mount count:              3
Maximum mount count:      -1
```

`Maximum mount count: -1` মানে নির্দিষ্ট সংখ্যক মাউন্টের পর স্বয়ংক্রিয় fsck বন্ধ — আধুনিক e2fsprogs-এ এটাই ডিফল্ট (জার্নাল থাকায় দরকার কম)। `0xEF53` হলো ext ফাইলসিস্টেমের "magic number", যা দেখে `blkid` চিনতে পারে।

| অপশন | কাজ |
|-------|-----|
| `-l` | সব তথ্য দেখাও (superblock) |
| `-L LABEL` | লেবেল বদলাও (`e2label /dev/sdb1 NEWLABEL` একই কাজ করে) |
| `-m 1` | reserved block % বদলাও |
| `-r COUNT` | reserved block সংখ্যা সরাসরি |
| `-c 30` | প্রতি ৩০ বার মাউন্টের পর fsck জোর করো |
| `-i 1m` | প্রতি ১ মাসে fsck (`d`, `w`, `m` একক) |
| `-C 0` | বর্তমান mount count শূন্য করো |
| `-e remount-ro` | ত্রুটিতে আচরণ (`continue`, `remount-ro`, `panic`) |
| `-U random` | নতুন random UUID (আনমাউন্ট অবস্থায়; ক্লোন করা ডিস্কে দুটো একই UUID হলে কাজে লাগে) |
| `-o acl` | ডিফল্ট মাউন্ট অপশন বসাও |
| `-O ^has_journal` | feature বন্ধ (`^` = বন্ধ) — আনমাউন্ট অবস্থায় |
| `-j` | ext2-তে জার্নাল যোগ করো (ext3 বানাও) |

```bash
$ sudo tune2fs -m 1 /dev/sdb1
tune2fs 1.47.0 (5-Feb-2023)
Setting reserved blocks percentage to 1% (39321 blocks)

$ sudo tune2fs -L MYDATA /dev/sdb1
```

**অন্য ফাইলসিস্টেমে লেবেল বদলানো:**

| ফাইলসিস্টেম | কমান্ড |
|-------------|--------|
| ext4 | `tune2fs -L NAME DEV` বা `e2label DEV NAME` |
| XFS | `xfs_admin -L NAME DEV` (আনমাউন্ট অবস্থায়) |
| vfat | `fatlabel DEV NAME` |
| exFAT | `exfatlabel DEV NAME` |
| Btrfs | `btrfs filesystem label MOUNTPOINT NAME` |
| swap | `swaplabel -L NAME DEV` |

**ফাইলসিস্টেম বড় করা (এক নজরে):** পার্টিশন বড় করার পর (`parted resizepart` বা `growpart`), ext4-এ `sudo resize2fs /dev/sdb1` আর XFS-এ `sudo xfs_growfs /mnt/data`। LVM-এ এটা অনেক সহজ — [অধ্যায় ৩৩](33-lvm-raid.md) দেখো।

---

## ৩২.১৫ USB পেনড্রাইভ মাউন্ট করা

### ডেস্কটপে (স্বয়ংক্রিয়)

GNOME/KDE-তে USB লাগালেই **udisks2** সার্ভিস সেটা মাউন্ট করে দেয় — Ubuntu/Debian-এ `/media/<user>/<LABEL>`, Fedora/RHEL-এ `/run/media/<user>/<LABEL>`। ফাইল ম্যানেজারে ⏏ (eject) চাপলে নিরাপদে খোলে। ([অধ্যায় ৪](04-desktop-environment.md))

### টার্মিনালে, root ছাড়াই — `udisksctl`

```bash
$ udisksctl mount -b /dev/sdc1
Mounted /dev/sdc1 at /media/rahim/PENDRIVE

$ udisksctl unmount -b /dev/sdc1
Unmounted /dev/sdc1.

$ udisksctl power-off -b /dev/sdc       # পুরো ডিস্কের পাওয়ার বন্ধ — এখন টেনে খোলো
```

### টার্মিনালে, হাতে-কলমে (সার্ভারে, GUI ছাড়া)

```bash
# ১. লাগানোর আগে ও পরে তুলনা করো
$ lsblk -o NAME,SIZE,RM,FSTYPE,LABEL,MOUNTPOINTS
NAME   SIZE RM FSTYPE LABEL    MOUNTPOINTS
sdc   14.9G  1
└─sdc1 14.9G  1 vfat   PENDRIVE

# ২. kernel কী দেখল (কোনো সমস্যা হলে এখানে ধরা পড়ে)
$ sudo dmesg | tail -5
[ 1834.210] usb 2-1: new SuperSpeed USB device number 3 using xhci_hcd
[ 1834.512] sd 6:0:0:0: [sdc] 31277232 512-byte logical blocks: (16.0 GB/14.9 GiB)
[ 1834.530]  sdc: sdc1
[ 1834.531] sd 6:0:0:0: [sdc] Attached SCSI removable disk

# ৩. মাউন্ট করো — FAT/exFAT-এ নিজের uid দাও, নাহলে সব ফাইল root-এর হবে
$ sudo mkdir -p /mnt/usb
$ sudo mount -o uid=$(id -u),gid=$(id -g),umask=022 /dev/sdc1 /mnt/usb

# ৪. কাজ শেষে
$ cd ~
$ sync                              # page cache-এর সব ডেটা ডিস্কে লেখো
$ sudo umount /mnt/usb
$ sudo eject /dev/sdc               # (ঐচ্ছিক)
```

> 📝 **মনে রাখো:** FAT32/exFAT/NTFS-এ লিনাক্সের মালিক-পারমিশন সংরক্ষণ হয় না, তাই মাউন্টের সময় `uid`, `gid`, `umask` (বা `fmask`/`dmask`) দিয়ে ঠিক করে দিতে হয়। ext4 USB-তে এগুলো লাগে না — সেখানে স্বাভাবিক পারমিশন চলে ([অধ্যায় ১৬](16-permissions.md))।

**NTFS (উইন্ডোজের ডিস্ক):** kernel 5.15+-এ নিজস্ব `ntfs3` ড্রাইভার আছে: `sudo mount -t ntfs3 /dev/sdc1 /mnt/usb`। পুরোনো ও বহুল ব্যবহৃত বিকল্প `ntfs-3g` (FUSE-ভিত্তিক): `sudo mount -t ntfs-3g /dev/sdc1 /mnt/usb`। উইন্ডোজ "Fast Startup"/hibernate অবস্থায় বন্ধ হলে ডিস্ক dirty থাকে, শুধু read-only মাউন্ট হতে পারে — উইন্ডোজে পুরো Shut down করে আসো।

**পেনড্রাইভ নতুন করে ফরম্যাট (সব মুছে):**

```bash
$ sudo umount /dev/sdc1
$ sudo wipefs -a /dev/sdc
$ sudo parted -s /dev/sdc mklabel msdos mkpart primary 1MiB 100%
$ sudo mkfs.exfat -n MYUSB /dev/sdc1
```

(পুরোনো টিভি/গাড়ির স্টেরিওর জন্য MBR + FAT32 সবচেয়ে নিরাপদ। ISO থেকে বুটেবল USB বানানো `dd`-এর বিস্তারিত [অধ্যায় ৩৫](35-backup-rsync.md)-এ।)

> ⚠️ **সতর্কতা:** "safely remove" ছাড়া USB টেনে খুললে শেষের কিছু লেখা হারাতে পারে, এমনকি ফাইলসিস্টেম নষ্ট হতে পারে — কারণ লেখা ডেটা তখনো RAM-এর page cache-এ বসে থাকতে পারে। সবসময় `umount` বা eject করো।

---

## হাতে-কলমে অনুশীলন

এই ল্যাবে আমরা কোনো আসল ডিস্ক ছোঁব না। একটা সাধারণ ফাইলকে **loop device** বানিয়ে তার ওপর সব চর্চা করব — ভুল হলেও কোনো ক্ষতি নেই।

### ল্যাব ১: একটা ভার্চুয়াল ডিস্ক বানাও

```bash
$ mkdir -p ~/disklab && cd ~/disklab
$ truncate -s 1G disk.img                # 1 GiB-এর sparse ফাইল
$ ls -lh disk.img; du -h disk.img
-rw-rw-r-- 1 rahim rahim 1.0G Nov 12 10:00 disk.img
0	disk.img                                 ← sparse: ডিস্কে এখনো কিছুই নেয়নি!

$ sudo losetup -fP --show disk.img       # -f: খালি loop খোঁজো, -P: পার্টিশন স্ক্যান
/dev/loop20
```

তোমার মেশিনে নম্বর আলাদা হতে পারে (`/dev/loop0`, `/dev/loop20`...) — নিচের কমান্ডে সেটাই বসাবে। এখানে আমরা ধরে নিচ্ছি `/dev/loop20`।

### ল্যাব ২: GPT টেবিল ও দুটো পার্টিশন বানাও

```bash
$ sudo fdisk /dev/loop20
```

ভেতরে ক্রমে চাপো: `g` → `n` → Enter → Enter → `+600M` → `n` → Enter → Enter → Enter (বাকি সব) → `t` → `2` → `swap` → `p` → `w`।

```bash
$ lsblk /dev/loop20
NAME       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop20       7:20   0    1G  0 loop
├─loop20p1 259:4    0  600M  0 part
└─loop20p2 259:5    0  423M  0 part
```

(`loop20p2`-এর আকার মোটামুটি 423M দেখাবে; GPT-র ব্যাকআপ টেবিলের জন্য শেষে কিছু জায়গা যায়।)

### ল্যাব ৩: ফাইলসিস্টেম ও swap

```bash
$ sudo mkfs.ext4 -L LAB /dev/loop20p1
$ sudo mkswap -L LABSWAP /dev/loop20p2
$ lsblk -f /dev/loop20
```

FSTYPE কলামে `ext4` আর `swap` দেখা যাওয়া উচিত।

### ল্যাব ৪: মাউন্ট, লেখা, লুকানো ফাইলের রহস্য

```bash
$ sudo mkdir -p /mnt/lab
$ echo "আমি নিচে লুকিয়ে আছি" | sudo tee /mnt/lab/hidden.txt
$ sudo mount /dev/loop20p1 /mnt/lab
$ ls /mnt/lab
lost+found                                   ← hidden.txt কোথায় গেল?
$ sudo cp /etc/hostname /mnt/lab/
$ df -h /mnt/lab
$ sudo umount /mnt/lab
$ ls /mnt/lab
hidden.txt                                   ← ফিরে এসেছে!
$ sudo rm /mnt/lab/hidden.txt
```

### ল্যাব ৫: "target is busy" তৈরি করো ও সমাধান করো

```bash
$ sudo mount /dev/loop20p1 /mnt/lab
$ cd /mnt/lab
$ sudo umount /mnt/lab
umount: /mnt/lab: target is busy.
$ sudo fuser -vm /mnt/lab                    # দেখো তোমার bash-ই দোষী
$ cd ~
$ sudo umount /mnt/lab                       # এবার সফল
```

### ল্যাব ৬: fstab দিয়ে স্থায়ী মাউন্ট (নিরাপদভাবে)

```bash
$ UUID=$(sudo blkid -s UUID -o value /dev/loop20p1); echo $UUID
$ sudo cp /etc/fstab /etc/fstab.bak
$ echo "UUID=$UUID /mnt/lab ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
$ sudo findmnt --verify
$ sudo systemctl daemon-reload
$ sudo mount -a
$ findmnt /mnt/lab
```

> ⚠️ **গুরুত্বপূর্ণ:** loop device রিবুটের পর নিজে থেকে তৈরি হয় না, তাই ল্যাব শেষে এই লাইনটা অবশ্যই মুছে ফেলবে (ল্যাব ১০ দেখো)। `nofail` দিয়েছি বলে ভুলে গেলেও বুট আটকাবে না, তবে ৯০ সেকেন্ড পর্যন্ত অপেক্ষা করতে পারে।

### ল্যাব ৭: swap চালু করো ও swappiness দেখো

```bash
$ sudo swapon /dev/loop20p2
$ swapon --show
$ free -h
$ cat /proc/sys/vm/swappiness
$ sudo swapoff /dev/loop20p2
```

### ল্যাব ৮: tune2fs ও fsck

```bash
$ sudo umount /mnt/lab
$ sudo tune2fs -l /dev/loop20p1 | grep -E 'volume name|Reserved block count|Mount count'
$ sudo tune2fs -m 1 -L LAB2 /dev/loop20p1
$ sudo e2fsck -f /dev/loop20p1
$ echo $?                                     # 0 হলে সব ঠিক
```

### ল্যাব ৯: জায়গা খোঁজা

```bash
$ du -sh ~/* 2>/dev/null | sort -h | tail -5
$ df -hT -x tmpfs -x devtmpfs
$ df -i /
$ ncdu ~                                      # ইনস্টল থাকলে; q দিয়ে বের হও
```

### ল্যাব ১০: পরিষ্কার করো

```bash
$ sudo umount /mnt/lab 2>/dev/null
$ sudo sed -i '\#/mnt/lab#d' /etc/fstab       # ল্যাবের লাইন মুছে দাও
$ sudo findmnt --verify
$ sudo systemctl daemon-reload
$ sudo losetup -d /dev/loop20                 # loop device খুলে দাও
$ losetup -a | grep disklab                   # কিছু না দেখালে সফল
$ rm -rf ~/disklab
```

(`sed`-এর `\#...#d` সিনট্যাক্স বুঝতে [অধ্যায় ২১](21-sed.md) দেখো।)

**বোনাস চ্যালেঞ্জ:** একই ল্যাব `parted -s` দিয়ে এক লাইনে করো, আর প্রথম পার্টিশনে ext4-এর বদলে XFS বানিয়ে `xfs_repair -n` চালাও।

---

## সারসংক্ষেপ

| কাজ | কমান্ড |
|------|--------|
| সব ডিস্ক/পার্টিশন গাছ আকারে | `lsblk`, `lsblk -f` |
| UUID, টাইপ, লেবেল | `sudo blkid`, `blkid -s UUID -o value DEV` |
| কোথায় কী মাউন্ট | `findmnt`, `findmnt /path` |
| fstab যাচাই | `sudo findmnt --verify` |
| পার্টিশন টেবিল দেখা | `sudo fdisk -l /dev/sdX` |
| পার্টিশন (MBR/GPT) | `sudo fdisk /dev/sdX` → `g`/`o`, `n`, `t`, `p`, `w`/`q` |
| শুধু GPT | `sudo gdisk /dev/sdX` |
| স্ক্রিপ্টে পার্টিশন | `sudo parted -s DEV mklabel gpt mkpart NAME 1MiB 100%` |
| টেবিল ব্যাকআপ | `sudo sfdisk --dump DEV > file` |
| kernel-কে টেবিল পড়ানো | `sudo partprobe DEV` |
| সিগনেচার মোছা | `sudo wipefs -a DEV` |
| ফাইলসিস্টেম তৈরি | `mkfs.ext4 -L`, `mkfs.xfs -L`, `mkfs.vfat -F 32 -n`, `mkfs.exfat -n` |
| মাউন্ট | `sudo mount [-t T] [-o opts] DEV DIR` |
| অপশন বদলে আবার মাউন্ট | `sudo mount -o remount,rw /` |
| ISO মাউন্ট | `sudo mount -o loop,ro file.iso DIR` |
| fstab-এর সব মাউন্ট | `sudo systemctl daemon-reload && sudo mount -a` |
| আনমাউন্ট | `sudo umount DIR` |
| কে ব্যস্ত রেখেছে | `sudo fuser -vm DIR`, `sudo lsof +f -- DIR` |
| ফাইলসিস্টেমের জায়গা | `df -h`, `df -hT`, `df -i` |
| ফোল্ডারের আকার | `du -sh DIR`, `du -h -d 1 DIR \| sort -h` |
| ইন্টারঅ্যাকটিভ | `ncdu -x DIR` |
| মুছে ফেলা কিন্তু খোলা ফাইল | `sudo lsof +L1` |
| swap তৈরি ও চালু | `mkswap DEV`, `swapon DEV`, `swapon --show` |
| swapfile | `fallocate -l 2G /swapfile; chmod 600; mkswap; swapon` |
| swappiness | `sysctl vm.swappiness=10`, `/etc/sysctl.d/99-swappiness.conf` |
| fsck | `sudo fsck -n DEV`, `sudo e2fsck -f -y DEV`, `sudo xfs_repair DEV` |
| ext4 টিউনিং | `tune2fs -l`, `tune2fs -L`, `tune2fs -m 1` |
| USB (root ছাড়া) | `udisksctl mount -b DEV`, `unmount`, `power-off` |
| loop device | `sudo losetup -fP --show img`, `losetup -d /dev/loopN` |

**শর্টকাট/কী এক নজরে:**

| টুল | মূল কী |
|-----|--------|
| `fdisk` | `m` সাহায্য, `p` দেখাও, `g` GPT, `o` MBR, `n` নতুন, `d` মোছো, `t` টাইপ, `l` টাইপ তালিকা, `F` খালি জায়গা, `w` লেখো, `q` বাতিল |
| `gdisk` | `?` সাহায্য, `p`, `n`, `d`, `t`, `L` কোড তালিকা, `c` নাম, `r` recovery, `w`, `q` |
| `parted` | `print`, `print free`, `mklabel`, `mkpart`, `rm`, `resizepart`, `set`, `unit`, `quit` (সব সাথে সাথে লেখা হয়!) |
| `ncdu` | `↑↓`/`jk`, `Enter`, `←`, `d` মোছো, `s` আকার, `n` নাম, `C` সংখ্যা, `a`, `g`, `e`, `i`, `r`, `?`, `q` |

---

## নিজেকে যাচাই করো

1. `nvme1n1p3` নামটা ভাঙলে কী কী তথ্য পাওয়া যায়? আর `sdb3`-এর সাথে নামকরণে পার্থক্য কোথায়?
2. MBR-এ সর্বোচ্চ কতগুলো primary পার্টিশন থাকতে পারে, এবং 512-byte sector-এ ডিস্কের সীমা কত? GPT কীভাবে টেবিল নষ্ট হওয়ার ঝুঁকি কমায়?
3. `fdisk` আর `parted`-এর মধ্যে ডিস্কে লেখার সময় নিয়ে সবচেয়ে বিপজ্জনক পার্থক্যটা কী?
4. `/etc/fstab`-এ `/dev/sdb1` না লিখে UUID লেখা কেন উচিত? `UUID` আর `PARTUUID`-এর পার্থক্য কী?
5. fstab-এর ষষ্ঠ ফিল্ডে root, একটা ext4 ডেটা পার্টিশন, একটা XFS পার্টিশন আর swap-এর জন্য কী মান দেবে?
6. `umount` করতে গেলে "target is busy" এলো। কোন দুটো কমান্ড দিয়ে কারণ খুঁজবে?
7. `df -h` বলছে `/` 100% ভরা, কিন্তু `du -sh /` অনেক কম দেখাচ্ছে। সম্ভাব্য দুটো কারণ ও সমাধান বলো।
8. একটা 2 GiB swapfile তৈরি করে স্থায়ীভাবে চালু করার সব ধাপ লেখো। `chmod 600` কেন জরুরি?
9. `vm.swappiness=10` মানে কি "RAM 90% ভরলে swap শুরু হবে"? ঠিক কী বোঝায়?
10. একটা মাউন্ট করা ext4 পার্টিশনে কেন `e2fsck -y` চালানো উচিত না, আর XFS-এর জন্য কোন টুল ব্যবহার করবে?

<details><summary>উত্তর দেখো</summary>

1. `nvme1n1p3` = দ্বিতীয় NVMe কন্ট্রোলার (`1`, গণনা 0 থেকে), namespace 1 (`n1`), পার্টিশন 3 (`p3`)। `sd` নামে ডিস্ক অক্ষর দিয়ে (`a`, `b`...), আর পার্টিশন নম্বর সরাসরি যোগ হয় (`sdb3`); NVMe-র নাম সংখ্যায় শেষ হয় বলে পার্টিশনের আগে `p` বসে।
2. ৪টা primary (বেশি লাগলে একটা extended-এর ভেতরে logical), সীমা 2 TiB। GPT ডিস্কের শেষে হেডার ও পার্টিশন টেবিলের ব্যাকআপ রাখে, আর CRC32 checksum দিয়ে নষ্ট হলে ধরে ফেলে, তাই ব্যাকআপ থেকে ফেরত আনা যায়।
3. `fdisk` (ও `gdisk`) সব পরিবর্তন মেমোরিতে রাখে, `w` চাপলে তবেই লেখে — `q` দিয়ে বাতিল করা যায়। `parted` প্রতিটা কমান্ড সাথে সাথে ডিস্কে লেখে, ফেরানোর সুযোগ নেই।
4. `sdX` নাম ডিস্ক শনাক্তের ক্রম অনুযায়ী বুটভেদে বদলাতে পারে; UUID ফাইলসিস্টেমের ভেতরে লেখা থাকে, তাই স্থির। `UUID` ফাইলসিস্টেমের (আবার mkfs করলে বদলায়), `PARTUUID` পার্টিশন টেবিলের এন্ট্রির (ফরম্যাট করলেও বদলায় না)।
5. root = `1`, ext4 ডেটা = `2`, XFS = `0`, swap = `0`।
6. `sudo fuser -vm /mount/point` এবং `sudo lsof +f -- /mount/point`। তারপর সেই প্রসেস বন্ধ করো বা শেলে `cd` দিয়ে বেরিয়ে আসো।
7. (ক) কোনো প্রসেস একটা মুছে ফেলা ফাইল খোলা রেখেছে — `sudo lsof +L1` দিয়ে খুঁজে সেই প্রসেস/সার্ভিস রিস্টার্ট করো। (খ) কোনো মাউন্ট পয়েন্টের নিচে ফাইল লুকিয়ে আছে — আনমাউন্ট করে দেখো (বা `mount --bind / /mnt/x` দিয়ে নিচের অংশ দেখো)। এছাড়া reserved block-ও কিছু পার্থক্য করে।
8. `sudo fallocate -l 2G /swapfile` (বা `dd`) → `sudo chmod 600 /swapfile` → `sudo mkswap /swapfile` → `sudo swapon /swapfile` → `/etc/fstab`-এ `/swapfile none swap defaults 0 0` যোগ করো → `swapon --show` দিয়ে যাচাই। swapfile-এ RAM-এর গোপন তথ্য (পাসওয়ার্ড, কী) লেখা হতে পারে, তাই অন্য কেউ যেন পড়তে না পারে।
9. না। swappiness হলো একটা তুলনামূলক ওজন — মেমোরির চাপ এলে kernel অ্যাপের মেমোরি (anonymous page) swap করবে নাকি ফাইল cache ফেলে দেবে, তার পছন্দ। কম মান মানে kernel অ্যাপের মেমোরিকে RAM-এ রাখতে বেশি চেষ্টা করে।
10. মাউন্ট থাকা অবস্থায় kernel একই সময়ে ফাইলসিস্টেমে লিখছে; fsck তার নিচ থেকে মেটাডেটা বদলালে দুজনের হিসাব সংঘর্ষে ফাইলসিস্টেম আরও নষ্ট হতে পারে। আগে `umount` করতে হয়। XFS-এর জন্য `xfs_repair` (আগে `xfs_repair -n` দিয়ে শুধু পরীক্ষা)।

</details>

---

[← আগের অধ্যায়](31-compile-from-source.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](33-lvm-raid.md)

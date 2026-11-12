# অধ্যায় ৩৩: LVM ও RAID

> আগের অধ্যায়ে ([অধ্যায় ৩২](32-disk-partition-mount.md)) তুমি শিখেছ ডিস্ক, পার্টিশন, ফাইলসিস্টেম আর মাউন্ট। সেখানে একটা সমস্যা চোখে পড়ার কথা: পার্টিশন একবার বানালে তার সাইজ বদলানো ঝামেলার, আর একটা ডিস্ক নষ্ট হলে তার সব ডেটা চলে যায়। এই অধ্যায়ের দুটো প্রযুক্তি ঠিক এই দুটো সমস্যার সমাধান করে। **LVM** দেয় নমনীয়তা (flexibility), আর **RAID** দেয় নির্ভরযোগ্যতা (redundancy) ও গতি।

## এই অধ্যায়ে যা শিখবে

- সাধারণ পার্টিশনের সীমাবদ্ধতা কী, আর LVM কেন দরকার
- LVM-এর তিন স্তর: PV (Physical Volume), VG (Volume Group), LV (Logical Volume), আর PE/LE কী জিনিস
- ভেতরে কীভাবে কাজ করে: device-mapper, `/dev/mapper/`, LVM metadata
- `pvcreate`, `vgcreate`, `lvcreate` দিয়ে শূন্য থেকে LVM বানানো
- `pvs`, `vgs`, `lvs` এবং `*display` কমান্ড দিয়ে অবস্থা দেখা
- `lvextend -r`, `resize2fs`, `xfs_growfs` দিয়ে চলন্ত অবস্থায় জায়গা বাড়ানো
- `lvreduce` কেন বিপজ্জনক, আর কীভাবে নিরাপদে করতে হয়
- নতুন ডিস্ক যোগ (`vgextend`), ডেটা সরানো (`pvmove`), ডিস্ক বাদ দেওয়া (`vgreduce`)
- LVM snapshot: কীভাবে কাজ করে (Copy-on-Write), বানানো, ফিরিয়ে আনা (merge)
- RAID কী, RAID 0/1/5/6/10 এর তুলনা, ডায়াগ্রাম সহ
- Hardware RAID, Software RAID, Fake RAID এর পার্থক্য
- `mdadm` দিয়ে software RAID তৈরি, `/proc/mdstat` পড়া, কনফিগ সেভ করা
- নষ্ট ডিস্ক বদলানো (fail → remove → add → rebuild)
- RAID কেন ব্যাকআপ **নয়**
- LVM আর RAID একসাথে ব্যবহার

---

## ১. সমস্যাটা কী? সাধারণ পার্টিশনের সীমাবদ্ধতা

ধরো তুমি একটা সার্ভারে ৫০০ GB ডিস্ক নিয়ে এভাবে পার্টিশন করলে:

```
/dev/sda
┌──────────┬──────────────────┬──────────────────────────────┐
│ sda1     │ sda2             │ sda3                         │
│ /boot    │ /  (৫০ GB)       │ /var  (৪৪৯ GB)               │
│ 1 GB     │                  │                              │
└──────────┴──────────────────┴──────────────────────────────┘
```

ছয় মাস পরে `/` ভরে গেল, অথচ `/var`-এ ৩০০ GB খালি পড়ে আছে। এখন কী করবে?

- `/var` ছোট করে `/`-কে বড় করতে হলে পার্টিশনগুলোকে ডিস্কের ওপর **পাশাপাশি সরাতে** হবে। এটা ধীর, ঝুঁকিপূর্ণ, আর সাধারণত সার্ভার বন্ধ রেখে করতে হয়।
- নতুন একটা ডিস্ক লাগালে সেটা আলাদা `/dev/sdb` হয়। `/`-এর সাথে জোড়া লাগানো যায় না। বড়জোর অন্য কোথাও মাউন্ট করা যায়।

**উপমা:** সাধারণ পার্টিশন হলো ইটের দেয়াল দিয়ে ঘর ভাগ করা। একবার দেয়াল তুললে ঘর বড়-ছোট করতে দেয়াল ভাঙতে হয়। LVM হলো অফিসের নড়াচড়া-যোগ্য পার্টিশন বোর্ড (movable partition)। বোর্ড সরিয়ে যখন খুশি ঘরের মাপ বদলানো যায়, এমনকি পাশের বিল্ডিং ভাড়া নিয়ে জোড়াও দেওয়া যায়।

---

## ২. LVM কী এবং কেন

**LVM (Logical Volume Manager)** হলো kernel আর ফাইলসিস্টেমের মাঝখানে একটা স্তর (abstraction layer, অর্থাৎ বাস্তব জিনিসকে আড়াল করে একটা সুবিধাজনক রূপ দেখানো)। এটা একাধিক ফিজিক্যাল ডিস্ক বা পার্টিশনকে এক বড় "স্টোরেজ পুলে" জমা করে, তারপর সেই পুল থেকে ইচ্ছেমতো সাইজের "ভার্চুয়াল পার্টিশন" কেটে দেয়।

LVM-এর সুবিধা:

| সুবিধা | ব্যাখ্যা |
|---|---|
| অনলাইন রিসাইজ | ফাইলসিস্টেম মাউন্ট থাকা অবস্থায় (সার্ভার চালু রেখে) জায়গা বাড়ানো যায় |
| একাধিক ডিস্ক জোড়া | ৩টা ২ TB ডিস্ক মিলিয়ে একটা ৬ TB volume |
| Snapshot | কোনো মুহূর্তের "ছবি" রেখে দেওয়া, আপগ্রেডের আগে নিরাপত্তা |
| লাইভ মাইগ্রেশন | `pvmove` দিয়ে পুরনো ডিস্ক থেকে নতুন ডিস্কে ডেটা সরানো, সার্ভার চালু রেখেই |
| নাম দিয়ে চেনা | `/dev/sdb3` না বলে `/dev/vgdata/mysql`, পড়তে সহজ |
| Thin provisioning | বাস্তবে যত জায়গা আছে তার চেয়ে বেশি "প্রতিশ্রুতি" দেওয়া (অ্যাডভান্সড) |

> 📝 **মনে রাখো:** Ubuntu Server ইনস্টলার, RHEL/Rocky/AlmaLinux ইনস্টলার — ডিফল্ট সেটআপেই LVM ব্যবহার করে। তাই তুমি নিজে LVM না বানালেও প্রোডাকশন সার্ভারে প্রায় নিশ্চিত দেখা হবে। Fedora Workstation (৩৩ সংস্করণ থেকে) অবশ্য ডিফল্টে Btrfs ব্যবহার করে, LVM নয়।

---

## ৩. LVM-এর তিন স্তর: PV, VG, LV

LVM বুঝতে তিনটা শব্দ মাথায় গেঁথে নাও:

| স্তর | পুরো নাম | কী | উপমা |
|---|---|---|---|
| **PV** | Physical Volume | একটা ডিস্ক বা পার্টিশন, যাকে LVM-এর জন্য "চিহ্নিত" করা হয়েছে | আলাদা আলাদা জমির প্লট |
| **VG** | Volume Group | এক বা একাধিক PV মিলিয়ে বানানো স্টোরেজ পুল | সব প্লট মিলিয়ে একটা বড় খামার |
| **LV** | Logical Volume | VG থেকে কেটে নেওয়া অংশ, যার ওপর ফাইলসিস্টেম বসে | খামারের ভেতরে ভাগ করা ক্ষেত (ধান, পাট, সবজি) |

### স্তরের ডায়াগ্রাম

```
 ┌───────────────────────────────────────────────────────────────┐
 │  ফাইলসিস্টেম স্তর      ext4            xfs            swap      │
 │                       ( / )          (/var/lib/mysql)          │
 ├───────────────────────────────────────────────────────────────┤
 │  LV  (Logical Volume)  lv_root        lv_mysql       lv_swap   │
 │                        20 GB          60 GB          4 GB      │
 ├───────────────────────────────────────────────────────────────┤
 │  VG  (Volume Group)            vgdata  (মোট ≈ 100 GB)            │
 │        [PE][PE][PE][PE][PE][PE][PE][PE][PE][PE] ... খালি PE     │
 ├───────────────────────────────────────────────────────────────┤
 │  PV  (Physical Volume)  /dev/sdb1 (50 GB)   /dev/sdc (50 GB)    │
 ├───────────────────────────────────────────────────────────────┤
 │  হার্ডওয়্যার           ডিস্ক ১ (SSD)          ডিস্ক ২ (HDD)        │
 └───────────────────────────────────────────────────────────────┘
```

খেয়াল করো: `lv_mysql`-এর ডেটা আসলে দুটো ডিস্কেই ছড়িয়ে থাকতে পারে। ফাইলসিস্টেম সেটা জানেও না, জানার দরকারও নেই।

### PE আর LE: ইট দিয়ে বানানো দেয়াল

VG তৈরি হলে LVM পুরো জায়গাটাকে সমান সাইজের ছোট ছোট টুকরোয় ভাগ করে। প্রতিটা টুকরোকে বলে **PE (Physical Extent)**। ডিফল্ট সাইজ **4 MiB**।

LV বানানো মানে আসলে কিছু PE বরাদ্দ (allocate) করা। LV-এর দিক থেকে প্রতিটা টুকরোকে বলে **LE (Logical Extent)**। প্রতিটা LE একটা PE-এর সাথে ম্যাপ করা থাকে।

```
LV: lv_mysql
  LE0   LE1   LE2   LE3   LE4   LE5
   │     │     │     │     │     │
   ▼     ▼     ▼     ▼     ▼     ▼
  PV /dev/sdb1          PV /dev/sdc
  [PE10][PE11][PE12]    [PE0][PE1][PE2]
```

তাই LV বড় করা মানে আরো কিছু খালি PE যোগ করে ম্যাপ লম্বা করা। কোনো ডেটা সরাতে হয় না। এ জন্যই কাজটা এত দ্রুত আর নিরাপদ।

> 💡 **টিপস:** LV-এর সাইজ সবসময় PE সাইজের গুণিতক হয়। তুমি 10.001 GB চাইলে LVM সেটাকে কাছের PE সীমায় গোল (round up) করে নেবে এবং একটা বার্তা দেখাবে।

---

## ৪. ভেতরে কীভাবে কাজ করে (Under the hood)

### device-mapper

LVM নিজে ডেটা পড়ে-লেখে না। আসল কাজটা করে kernel-এর একটা অংশ, নাম **device-mapper** (dm)। device-mapper একটা "ম্যাপিং টেবিল" অনুযায়ী ভার্চুয়াল ব্লক ডিভাইসের অনুরোধ আসল ডিস্কে পাঠিয়ে দেয়।

```
 অ্যাপ (mysql)  ──write──►  ফাইলসিস্টেম (xfs)
                                  │  "ব্লক 5000 লেখো"
                                  ▼
                      /dev/dm-1  (device-mapper ডিভাইস)
                                  │  ম্যাপিং টেবিল দেখে:
                                  │  "ব্লক 5000 = /dev/sdc-র অফসেট X"
                                  ▼
                           /dev/sdc  (আসল ডিস্ক)
```

User-space-এর LVM টুলগুলো (`lvcreate` ইত্যাদি) শুধু metadata আপডেট করে এবং kernel-কে নতুন ম্যাপিং টেবিল দেয়। এজন্য তুমি একই LV-কে তিনটা নামে দেখবে:

```
$ ls -l /dev/vgdata/lv_mysql /dev/mapper/vgdata-lv_mysql
lrwxrwxrwx 1 root root 7 Nov 12 10:15 /dev/mapper/vgdata-lv_mysql -> ../dm-1
lrwxrwxrwx 1 root root 7 Nov 12 10:15 /dev/vgdata/lv_mysql -> ../dm-1
```

তিনটাই আসলে একই `/dev/dm-1`। প্রথম দুটো symlink (সফট লিংক, দেখো [অধ্যায় ১১](11-links-inode.md))। `/etc/fstab`-এ `/dev/mapper/...` বা `/dev/vgname/lvname` লেখা নিরাপদ, কিন্তু `/dev/dm-1` লিখবে না, কারণ বুটভেদে নম্বর বদলাতে পারে।

> 📝 **মনে রাখো:** `/dev/mapper/` নামে VG আর LV-এর মাঝে একটা হাইফেন (`-`) থাকে। নামের ভেতরেই হাইফেন থাকলে সেটা দুটো হাইফেন হয়ে যায়। যেমন VG `my-vg`, LV `data` → `/dev/mapper/my--vg-data`।

### dmsetup দিয়ে ম্যাপিং দেখা

```
$ sudo dmsetup ls
vgdata-lv_mysql	(252:1)
vgdata-lv_root	(252:0)

$ sudo dmsetup table vgdata-lv_mysql
0 62914560 linear 8:17 41945088
62914560 62914560 linear 8:32 2048
```

এখানে দেখা যাচ্ছে LV-এর প্রথম অংশ ডিভাইস `8:17` (`sdb1`) থেকে, দ্বিতীয় অংশ `8:32` (`sdc`) থেকে আসছে। `linear` মানে টুকরোগুলো পরপর জোড়া।

### LVM metadata কোথায় থাকে

প্রতিটা PV-এর শুরুর দিকে একটা ছোট জায়গায় LVM তার **metadata** (VG-এর নাম, কোন LV কোন PE ব্যবহার করে ইত্যাদি) লিখে রাখে। সব PV-তে metadata-র কপি থাকে। এছাড়া প্রতিবার পরিবর্তনের পর টেক্সট আকারে ব্যাকআপ রাখে:

- `/etc/lvm/backup/<VG-নাম>`: সর্বশেষ অবস্থা
- `/etc/lvm/archive/`: আগের অবস্থাগুলো

ভুল করে কোনো LV মুছে ফেললে `vgcfgrestore` দিয়ে metadata আগের অবস্থায় ফেরানো সম্ভব (যদি এর মধ্যে ওই জায়গায় নতুন কিছু লেখা না হয়ে থাকে)। এটা জরুরি অবস্থার টুল, রোজকার নয়।

```
$ sudo vgcfgrestore --list vgdata
```

---

## ৫. প্রস্তুতি: প্যাকেজ ও ডিস্ক চেনা

বেশিরভাগ সার্ভার distro-তে LVM টুল আগে থেকেই থাকে। না থাকলে:

```
# Ubuntu / Debian
$ sudo apt install lvm2

# Fedora / RHEL / Rocky / AlmaLinux
$ sudo dnf install lvm2
```

আগে দেখে নাও কোন ডিস্ক খালি আছে ([অধ্যায় ৩২](32-disk-partition-mount.md)-এ `lsblk` শিখেছ):

```
$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   40G  0 disk
├─sda1   8:1    0    1G  0 part /boot
└─sda2   8:2    0   39G  0 part /
sdb      8:16   0   10G  0 disk
sdc      8:32   0   10G  0 disk
sdd      8:48   0   10G  0 disk
```

`sdb`, `sdc`, `sdd` খালি। এগুলো নিয়েই আমরা কাজ করব।

> ⚠️ **সতর্কতা:** এই অধ্যায়ের প্রায় সব কমান্ড ডিস্কের ডেটা মুছে ফেলতে পারে। ডিভাইসের নাম তিনবার মিলিয়ে নাও। ভুল ডিস্কে `pvcreate` বা `mdadm --create` চালালে সেই ডিস্কের ডেটা হারাবে। শেখার সময় ভার্চুয়াল মেশিন বা loop device ব্যবহার করো (নিচের "হাতে-কলমে অনুশীলন" অংশে দেখানো আছে)।

### পুরো ডিস্ক নাকি পার্টিশন?

PV হিসেবে পুরো ডিস্ক (`/dev/sdb`) বা পার্টিশন (`/dev/sdb1`) দুটোই চলে। অনেকে একটা পার্টিশন বানিয়ে তার type "Linux LVM" (GPT-তে `fdisk`-এর type `lvm`, MBR-এ `8e`) দেয়, যাতে অন্য টুল বা মানুষ দেখেই বোঝে এটা খালি ডিস্ক নয়। দুটো পদ্ধতিই সঠিক।

---

## ৬. ধাপ ১: PV বানানো — `pvcreate`

**কী করে:** একটা ডিস্ক/পার্টিশনকে LVM-এর জন্য তৈরি করে (শুরুতে LVM label ও metadata এলাকা লেখে)।

**Syntax:**

```
pvcreate [অপশন] ডিভাইস...
```

| অপশন | কাজ |
|---|---|
| `-v` | বিস্তারিত বার্তা দেখায় |
| `-f` | জোর করে (আগের সিগনেচার থাকলেও) — খুব সাবধানে |
| `-y` | সব প্রশ্নে "হ্যাঁ" ধরে নেয় |
| `--dataalignment` | ডেটা শুরুর alignment ঠিক করে (সাধারণত ডিফল্টই ভালো) |

**উদাহরণ:**

```
$ sudo pvcreate /dev/sdb /dev/sdc
  Physical volume "/dev/sdb" successfully created.
  Physical volume "/dev/sdc" successfully created.
```

ডিস্কে আগে কোনো ফাইলসিস্টেম বা পার্টিশন টেবিল থাকলে LVM জিজ্ঞেস করবে:

```
WARNING: ext4 signature detected on /dev/sdb at offset 1080. Wipe it? [y/n]:
```

এখানে থামো, ভাবো, তারপর উত্তর দাও। আগের সিগনেচার পরিষ্কার করতে `wipefs` টুলও আছে:

```
$ sudo wipefs -a /dev/sdb      # সব সিগনেচার মুছবে, সাবধান!
```

---

## ৭. ধাপ ২: VG বানানো — `vgcreate`

**কী করে:** এক বা একাধিক PV মিলিয়ে একটা Volume Group তৈরি করে।

**Syntax:**

```
vgcreate [অপশন] VG-নাম PV...
```

| অপশন | কাজ |
|---|---|
| `-s`, `--physicalextentsize` | PE সাইজ ঠিক করে (ডিফল্ট 4M), যেমন `-s 16M` |
| `-v` | বিস্তারিত বার্তা |

**উদাহরণ:**

```
$ sudo vgcreate vgdata /dev/sdb /dev/sdc
  Volume group "vgdata" successfully created
```

> 💡 **টিপস:** VG-এর নাম অর্থবহ রাখো: `vgdata`, `vg_db`, `vg_backup`। RHEL ইনস্টলার ডিফল্টে `rhel` (বা `rl`, `almalinux`) নাম দেয়, Ubuntu দেয় `ubuntu-vg`।

> 💡 **টিপস:** `vgcreate` চালানোর সময় ডিভাইসে PV না থাকলে আধুনিক LVM নিজে থেকেই `pvcreate` করে নেয়। তবে শেখার সময় ধাপগুলো আলাদা করে চালানো ভালো, বোঝা সহজ হয়।

---

## ৮. ধাপ ৩: LV বানানো — `lvcreate`

**কী করে:** VG থেকে জায়গা কেটে একটা Logical Volume তৈরি করে।

**Syntax:**

```
lvcreate [অপশন] -n LV-নাম VG-নাম
```

| অপশন | কাজ |
|---|---|
| `-L সাইজ` | নির্দিষ্ট সাইজ: `-L 5G`, `-L 500M`, `-L 1T` |
| `-l সংখ্যা/শতাংশ` | PE সংখ্যা বা শতাংশ: `-l 100`, `-l 50%VG`, `-l 100%FREE` |
| `-n নাম` | LV-এর নাম |
| `-s` | snapshot বানায় (পরে দেখব) |
| `--type` | `linear` (ডিফল্ট), `striped`, `raid1`, `raid5`, `thin-pool` ইত্যাদি |
| `-i সংখ্যা` | stripe সংখ্যা (কয়টা PV-তে ছড়াবে) |
| `-m সংখ্যা` | mirror সংখ্যা (`--type raid1` এর সাথে) |

**উদাহরণ:**

```
$ sudo lvcreate -L 5G -n lv_web vgdata
  Logical volume "lv_web" created.

$ sudo lvcreate -l 100%FREE -n lv_db vgdata
  Logical volume "lv_db" created.
```

দ্বিতীয় কমান্ডে VG-এর বাকি সব খালি জায়গা `lv_db` নিয়ে নিল।

> 💡 **টিপস:** শুরুতে সব জায়গা দিয়ে দিও না। VG-তে কিছু খালি জায়গা রেখে দাও। পরে যে LV-র দরকার পড়বে তাকে বাড়িয়ে দেওয়া যাবে, আর snapshot-এর জন্যও খালি জায়গা লাগে। LV বড় করা সহজ, ছোট করা কঠিন।

### ফাইলসিস্টেম বানানো ও মাউন্ট

LV হলো একটা সাধারণ block device। এর ওপর যেকোনো ফাইলসিস্টেম বসানো যায়:

```
$ sudo mkfs.ext4 /dev/vgdata/lv_web
$ sudo mkfs.xfs  /dev/vgdata/lv_db

$ sudo mkdir -p /srv/web /srv/db
$ sudo mount /dev/vgdata/lv_web /srv/web
$ sudo mount /dev/vgdata/lv_db  /srv/db

$ df -h /srv/web /srv/db
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/vgdata-lv_web    4.9G   24K  4.6G   1% /srv/web
/dev/mapper/vgdata-lv_db      15G  140M   15G   1% /srv/db
```

রিবুটের পরও মাউন্ট রাখতে `/etc/fstab`-এ যোগ করো:

```
/dev/mapper/vgdata-lv_web  /srv/web  ext4  defaults  0 2
/dev/mapper/vgdata-lv_db   /srv/db   xfs   defaults  0 0
```

তারপর পরীক্ষা করো (ভুল থাকলে এখনই ধরা পড়বে, রিবুটে নয়):

```
$ sudo mount -a
$ sudo systemctl daemon-reload     # systemd-কে নতুন fstab জানাতে
```

> ⚠️ **সতর্কতা:** fstab-এ ভুল থাকলে সিস্টেম বুটে emergency mode-এ আটকে যেতে পারে। সবসময় `mount -a` বা `findmnt --verify` দিয়ে যাচাই করো।

---

## ৯. অবস্থা দেখা: `pvs`, `vgs`, `lvs`

এই তিনটা সংক্ষিপ্ত (এক লাইনে একটা) রিপোর্ট দেয়। দৈনন্দিন কাজে এগুলোই সবচেয়ে বেশি লাগে।

```
$ sudo pvs
  PV         VG     Fmt  Attr PSize   PFree
  /dev/sdb   vgdata lvm2 a--  <10.00g    0
  /dev/sdc   vgdata lvm2 a--  <10.00g    0
  /dev/sdd          lvm2 ---   10.00g 10.00g

$ sudo vgs
  VG     #PV #LV #SN Attr   VSize  VFree
  vgdata   2   2   0 wz--n- 19.99g    0

$ sudo lvs
  LV     VG     Attr       LSize  Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv_db  vgdata -wi-ao---- 14.99g
  lv_web vgdata -wi-ao----  5.00g
```

কলামগুলোর মানে:

| কলাম | মানে |
|---|---|
| `PSize` / `PFree` | PV-এর মোট সাইজ / খালি জায়গা |
| `#PV #LV #SN` | VG-তে কয়টা PV, কয়টা LV, কয়টা snapshot |
| `VSize` / `VFree` | VG-এর মোট / খালি জায়গা — **এটা দেখেই বোঝো আর কতটা বাড়ানো যাবে** |
| `LSize` | LV-এর সাইজ |
| `Origin` | snapshot হলে কোন LV-র snapshot |
| `Data%` | snapshot বা thin pool কতটা ভরেছে |
| `Cpy%Sync` | LVM RAID/mirror কতটা sync হয়েছে |

`<10.00g` মানে "১০ GB-এর সামান্য কম" — metadata-র জন্য কিছু জায়গা গেছে।

### Attr কলাম পড়া (lvs)

`-wi-ao----` প্রতিটা অক্ষরের একটা মানে আছে। প্রধানগুলো:

| অবস্থান | উদাহরণ | মানে |
|---|---|---|
| ১ | `-` / `s` / `o` / `r` / `t` / `V` | সাধারণ / snapshot / origin / raid / thin pool / thin volume |
| ২ | `w` / `r` | writeable / read-only |
| ৩ | `i` / `c` | allocation policy: inherited / contiguous |
| ৫ | `a` | active (চালু) |
| ৬ | `o` | open (মাউন্ট করা বা ব্যবহার হচ্ছে) |

### বিস্তারিত দেখা: `pvdisplay`, `vgdisplay`, `lvdisplay`

```
$ sudo vgdisplay vgdata
  --- Volume group ---
  VG Name               vgdata
  Format                lvm2
  VG Access             read/write
  VG Status             resizable
  Cur LV                2
  Open LV               2
  Cur PV                2
  VG Size               19.99 GiB
  PE Size               4.00 MiB
  Total PE              5118
  Alloc PE / Size       5118 / 19.99 GiB
  Free  PE / Size       0 / 0
  VG UUID               3Kp1fd-...
```

### নিজের পছন্দের কলাম

```
$ sudo lvs -o lv_name,vg_name,lv_size,devices
  LV     VG     LSize  Devices
  lv_db  vgdata 14.99g /dev/sdb(1280)
  lv_db  vgdata 14.99g /dev/sdc(0)
  lv_web vgdata  5.00g /dev/sdb(0)
```

`-o devices` দেখায় কোন LV কোন ডিস্কের কোন PE থেকে শুরু। `lv_db` দুটো ডিস্কে ছড়িয়ে আছে। অন্য কাজের অপশন:

| কমান্ড | কাজ |
|---|---|
| `lvs -a` | লুকানো (internal) LV-ও দেখায়, যেমন RAID-এর অংশ |
| `lvs --units m` | সাইজ MiB-তে দেখায় |
| `pvs -o +pv_used` | বাড়তি কলাম যোগ (`+` দিয়ে) |
| `pvscan`, `vgscan`, `lvscan` | সব ডিস্ক স্ক্যান করে LVM খোঁজে |

---

## ১০. জায়গা বাড়ানো: `lvextend`, `resize2fs`, `xfs_growfs`

এটাই LVM-এর সবচেয়ে বড় সুবিধা, আর বাস্তবে সবচেয়ে বেশি দরকার পড়ে। রাত ২টায় `/var` ভরে গেছে, সার্ভার বন্ধ না করেই বাড়াতে হবে।

### দুই স্তরের কাজ

মনে রাখো, জায়গা বাড়াতে **দুটো** জিনিস বড় করতে হয়:

```
 ধাপ ১: LV বড় করো           ধাপ ২: ফাইলসিস্টেম বড় করো
 ┌──────────────┐            ┌──────────────┐
 │ ext4 (5G)    │            │ ext4 (8G)    │◄── এখন ফাইল রাখার জায়গা বাড়ল
 ├──────────────┴───┐        ├──────────────┴───┐
 │ LV (8G)          │        │ LV (8G)          │
 └──────────────────┘        └──────────────────┘
  LV বড়, কিন্তু ফাইলসিস্টেম      দুটোই বড়
  এখনো জানে না
```

শুধু LV বড় করলে `df -h` আগের সাইজই দেখাবে। এটা নতুনদের সবচেয়ে সাধারণ বিভ্রান্তি।

### `lvextend` এর syntax

```
lvextend [অপশন] LV-পাথ [PV...]
```

| অপশন | কাজ |
|---|---|
| `-L +5G` | ৫ GB **যোগ** করো (`+` চিহ্নটা খেয়াল করো) |
| `-L 20G` | মোট সাইজ **২০ GB** করো |
| `-l +100%FREE` | VG-এর সব খালি জায়গা যোগ করো |
| `-l +50%FREE` | খালি জায়গার অর্ধেক যোগ করো |
| `-r`, `--resizefs` | LV বাড়ানোর পর ফাইলসিস্টেমও বাড়িয়ে দাও (একসাথে দুই ধাপ) |

> ⚠️ **সতর্কতা:** `-L +5G` আর `-L 5G` এর পার্থক্য বিশাল। প্রথমটা ৫ GB যোগ করে, দ্বিতীয়টা মোট সাইজ ৫ GB করে। `lvextend`-এ ছোট সাইজ দিলে LVM error দেবে (কারণ extend মানে শুধু বাড়ানো), কিন্তু `lvresize`-এ দিলে ছোট করে ফেলতে পারে।

### সহজ পথ: `-r` দিয়ে একবারে

প্রথমে VG-তে খালি জায়গা আছে কিনা দেখো (`vgs`-এর `VFree`)। না থাকলে আগে নতুন ডিস্ক যোগ করতে হবে (পরের অংশে)।

```
$ sudo lvextend -r -L +3G /dev/vgdata/lv_web
  Size of logical volume vgdata/lv_web changed from 5.00 GiB (1280 extents) to 8.00 GiB (2048 extents).
  Logical volume vgdata/lv_web successfully resized.
resize2fs 1.47.0 (5-Feb-2023)
Filesystem at /dev/mapper/vgdata-lv_web is mounted on /srv/web; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
The filesystem on /dev/mapper/vgdata-lv_web is now 2097152 (4k) blocks long.

$ df -h /srv/web
Filesystem                 Size  Used Avail Use% Mounted on
/dev/mapper/vgdata-lv_web  7.8G   24K  7.4G   1% /srv/web
```

`-r` ভেতরে `fsadm` নামের একটা স্ক্রিপ্ট ডাকে, যেটা ফাইলসিস্টেমের ধরন দেখে ঠিক টুল (ext4 হলে `resize2fs`, XFS হলে `xfs_growfs`) চালায়। ফাইলসিস্টেম মাউন্ট থাকা অবস্থাতেই কাজ হয়।

### হাতে-হাতে পথ: আলাদা দুই ধাপ

কখনো `-r` ভুলে গেলে, বা বুঝে শুনে আলাদা করে করতে চাইলে:

**ext2/ext3/ext4 এর জন্য `resize2fs`:**

```
$ sudo lvextend -L +2G /dev/vgdata/lv_web
$ sudo resize2fs /dev/vgdata/lv_web
```

`resize2fs` সাইজ না দিলে ডিভাইসের পুরো জায়গা পর্যন্ত বাড়ায়। এটা **ডিভাইসের পাথ** নেয়।

**XFS এর জন্য `xfs_growfs`:**

```
$ sudo lvextend -L +5G /dev/vgdata/lv_db
$ sudo xfs_growfs /srv/db
meta-data=/dev/mapper/vgdata-lv_db isize=512    agcount=4, agsize=982016 blks
...
data blocks changed from 3928064 to 5238784
```

> 📝 **মনে রাখো:** `xfs_growfs` **মাউন্ট পয়েন্ট** নেয় (যেমন `/srv/db`), আর XFS শুধু মাউন্ট থাকা অবস্থাতেই বড় করা যায়। ext4-এর `resize2fs` নেয় **ডিভাইস পাথ**, এবং মাউন্ট বা আনমাউন্ট দুই অবস্থাতেই বাড়াতে পারে।

| ফাইলসিস্টেম | বড় করা | ছোট করা | টুল |
|---|---|---|---|
| ext4 | মাউন্ট অবস্থায়ও পারে | শুধু আনমাউন্ট অবস্থায় | `resize2fs` |
| XFS | শুধু মাউন্ট অবস্থায় | **একেবারেই পারে না** | `xfs_growfs` |
| Btrfs | মাউন্ট অবস্থায় | মাউন্ট অবস্থায় | `btrfs filesystem resize` |
| swap | `swapoff` → `lvextend` → `mkswap` → `swapon` | একইভাবে | `mkswap` |

### বাস্তব ঘটনা: রুট (/) ভরে গেছে

Ubuntu Server ইনস্টলার অনেক সময় `ubuntu-lv`-কে ডিস্কের পুরো জায়গা দেয় না, বাকিটা VG-তে খালি রেখে দেয়। তখন অনেকে ভাবে "১০০ GB ডিস্কে মাত্র অল্প জায়গা কেন?" সমাধান:

```
$ sudo vgs
  VG        #PV #LV #SN Attr   VSize   VFree
  ubuntu-vg   1   1   0 wz--n- <98.00g  49.00g

$ sudo lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

---

## ১১. নতুন ডিস্ক যোগ করা ও সরানো

### VG বড় করা: `vgextend`

VG-তে জায়গা ফুরিয়েছে? নতুন ডিস্ক লাগাও, PV বানাও, VG-তে ঢোকাও:

```
$ sudo pvcreate /dev/sdd
  Physical volume "/dev/sdd" successfully created.

$ sudo vgextend vgdata /dev/sdd
  Volume group "vgdata" successfully extended

$ sudo vgs
  VG     #PV #LV #SN Attr   VSize  VFree
  vgdata   3   2   0 wz--n- 29.99g 10.00g
```

এখন `VFree` ১০ GB। যেকোনো LV-কে `lvextend` দিয়ে বাড়াও। ক্লাউডে (AWS EBS, Azure disk ইত্যাদি) এভাবেই নতুন ভলিউম জুড়ে জায়গা বাড়ানো হয়।

### পুরনো ডিস্ক থেকে ডেটা সরানো: `pvmove`

ধরো `/dev/sdb` পুরনো হয়ে গেছে, SMART সতর্কতা দিচ্ছে। সার্ভার চালু রেখেই এর সব ডেটা অন্য PV-তে সরাতে পারো:

```
$ sudo pvmove /dev/sdb
  /dev/sdb: Moved: 12.45%
  /dev/sdb: Moved: 57.80%
  /dev/sdb: Moved: 100.00%
```

নির্দিষ্ট গন্তব্য দিতে চাইলে: `pvmove /dev/sdb /dev/sdd`। শর্ত: বাকি PV-গুলোতে যথেষ্ট খালি জায়গা থাকতে হবে।

> 💡 **টিপস:** `pvmove` বড় ডিস্কে ঘণ্টার পর ঘণ্টা লাগতে পারে। মাঝপথে বাধা পড়লে (রিবুট ইত্যাদি) শুধু `pvmove` আবার চালালে যেখানে থেমেছিল সেখান থেকে চলবে। এমন লম্বা কাজ `tmux`-এর ভেতরে চালাও ([অধ্যায় ২৫](25-jobs-tmux.md))।

### ডিস্ক বাদ দেওয়া: `vgreduce` ও `pvremove`

```
$ sudo vgreduce vgdata /dev/sdb
  Removed "/dev/sdb" from volume group "vgdata"

$ sudo pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
```

এখন `/dev/sdb` খুলে ফেলা নিরাপদ। পুরো প্রক্রিয়া এক নজরে:

```
 পুরনো ডিস্ক বদলানো (LVM দিয়ে, ডাউনটাইম ছাড়া)
 ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
 │ pvcreate │──►│ vgextend │──►│  pvmove  │──►│ vgreduce │──►│ pvremove │
 │ নতুন ডিস্ক │   │ VG-তে যোগ │   │ ডেটা সরাও │   │ VG থেকে  │   │ লেবেল মুছো│
 └──────────┘   └──────────┘   └──────────┘   │ বাদ দাও  │   └──────────┘
                                              └──────────┘
```

### অন্যান্য দরকারি কমান্ড

| কমান্ড | কাজ |
|---|---|
| `lvrename vgdata lv_web lv_www` | LV-এর নাম বদলানো |
| `vgrename vgdata vgprod` | VG-এর নাম বদলানো (রুট VG হলে fstab/grub ঠিক করতে হবে) |
| `lvremove /dev/vgdata/lv_old` | LV মুছে ফেলা (আগে umount করো) |
| `vgremove vgdata` | VG মুছে ফেলা (আগে সব LV মুছতে হবে) |
| `lvchange -an vgdata/lv_web` | LV নিষ্ক্রিয় (deactivate) করা |
| `vgchange -ay` | সব VG-এর সব LV সক্রিয় করা |
| `pvresize /dev/sdb` | ক্লাউডে ডিস্ক বড় করার পর PV-কে নতুন সাইজ জানানো |

> 💡 **টিপস:** ক্লাউডে একটা ডিস্কের সাইজ বাড়ালে (যেমন ২০ GB → ৫০ GB) PV আপনা-আপনি বাড়ে না। ডিস্কে পার্টিশন থাকলে আগে পার্টিশন বড় করো (`growpart /dev/sdb 1`, `cloud-guest-utils` / `cloud-utils-growpart` প্যাকেজে), তারপর `pvresize`, তারপর `lvextend -r`।

---

## ১২. LV ছোট করা: `lvreduce` — সবচেয়ে বিপজ্জনক কাজ

বড় করা সহজ, কারণ নতুন জায়গা খালি। ছোট করা বিপজ্জনক, কারণ LV-এর শেষের অংশ কেটে ফেলা হয়। সেখানে যদি ফাইলের ডেটা থাকে, সেটা চিরতরে চলে যায়।

```
 ভুল ক্রম (ডেটা নষ্ট!):
 ┌─────────────────────────────────┐
 │ ফাইলসিস্টেম ১০G (ডেটা শেষ পর্যন্ত) │
 ├─────────────────┬───────────────┤
 │ LV (৬G রাখলাম)  │ ✂ কেটে ফেলা    │ ◄── এখানকার ডেটা হারিয়ে গেল
 └─────────────────┴───────────────┘

 সঠিক ক্রম:
 ধাপ ১: ফাইলসিস্টেম ছোট করো (ডেটা সামনে সরে আসে)
 ধাপ ২: তারপর LV ছোট করো
```

### নিয়ম

1. **আগে ব্যাকআপ নাও।** সত্যি বলছি, এটা ঐচ্ছিক নয়।
2. **XFS ছোট করা যায় না।** XFS হলে একমাত্র পথ: ব্যাকআপ → LV মুছে নতুন ছোট LV বানাও → রিস্টোর।
3. ext4 ছোট করতে হলে **আনমাউন্ট** করতে হবে। রুট (`/`) ফাইলসিস্টেম হলে live USB থেকে বুট করতে হবে।
4. ফাইলসিস্টেমে যত ডেটা আছে তার চেয়ে ছোট করা যাবে না।

### নিরাপদ উপায়: `-r` দিয়ে

`lvreduce -r` ঠিক ক্রমে (আগে ফাইলসিস্টেম, পরে LV) কাজ করে:

```
$ sudo umount /srv/web
$ sudo lvreduce -r -L 4G /dev/vgdata/lv_web
fsck from util-linux 2.39.3
/dev/mapper/vgdata-lv_web: 11/327680 files (0.0% non-contiguous), 42078/2097152 blocks
resize2fs 1.47.0 (5-Feb-2023)
Resizing the filesystem on /dev/mapper/vgdata-lv_web to 1048576 (4k) blocks.
  Size of logical volume vgdata/lv_web changed from 8.00 GiB (2048 extents) to 4.00 GiB (1024 extents).
  Logical volume vgdata/lv_web successfully resized.
$ sudo mount /srv/web
```

### হাতে-হাতে উপায় (ধারণা পরিষ্কার করার জন্য)

```
$ sudo umount /srv/web
$ sudo e2fsck -f /dev/vgdata/lv_web          # আগে বাধ্যতামূলক চেক
$ sudo resize2fs /dev/vgdata/lv_web 4G       # ১. ফাইলসিস্টেম ছোট
$ sudo lvreduce -L 4G /dev/vgdata/lv_web     # ২. LV ছোট (একই বা বড় সাইজ!)
$ sudo mount /srv/web
```

`lvreduce` -r ছাড়া চালালে সতর্কবার্তা দেখাবে:

```
  WARNING: Reducing active logical volume to 4.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce vgdata/lv_web? [y/n]:
```

> ⚠️ **সতর্কতা:** হাতে-হাতে করলে GB আর GiB, রাউন্ডিং — এসবের গোলমালে LV ফাইলসিস্টেমের চেয়ে সামান্য ছোট হয়ে যেতে পারে, আর তাতেই ফাইলসিস্টেম ভেঙে যায়। নিরাপদ কৌশল: ফাইলসিস্টেমকে লক্ষ্যের চেয়ে একটু বেশি ছোট করো (যেমন 3.5G), LV-কে লক্ষ্য সাইজে (4G) আনো, তারপর `resize2fs` সাইজ ছাড়া চালিয়ে ফাইলসিস্টেমকে LV-এর পুরো সাইজে বাড়িয়ে নাও। অথবা সোজা `-r` ব্যবহার করো।

> 💡 **টিপস:** নতুন LVM সংস্করণে (2.03.17+, যা Ubuntu 24.04, Fedora, RHEL 9-এর নতুন আপডেটে আছে) `lvreduce` ডিফল্টে ফাইলসিস্টেম ভাঙতে পারে এমন কাজে বাধা দেয়, যদি না `-r` বা `--fs` অপশন দাও। এটা একটা ভালো নিরাপত্তা জাল, তবে এর ওপর পুরো ভরসা কোরো না।

---

## ১৩. LVM Snapshot

### Snapshot কী

**Snapshot** হলো একটা LV-এর কোনো নির্দিষ্ট মুহূর্তের "জমে থাকা ছবি" (point-in-time copy)। বানাতে এক সেকেন্ডও লাগে না, কারণ শুরুতে কোনো ডেটা কপি হয় না।

**উপমা:** ধরো তুমি একটা খাতায় লিখছ। Snapshot নেওয়া মানে খাতার পাশে একটা খালি নোটপ্যাড রাখা। যখনই খাতার কোনো পাতা বদলাতে যাবে, বদলানোর **আগে** সেই পাতার পুরনো লেখাটা নোটপ্যাডে টুকে রাখবে। তাহলে খাতা + নোটপ্যাড মিলিয়ে সবসময় পুরনো অবস্থাটা ফিরে পাওয়া যায়। এই কৌশলের নাম **Copy-on-Write (CoW)**।

```
 Snapshot নেওয়ার মুহূর্তে:
   Origin LV:  [A][B][C][D]
   Snapshot:   [ ][ ][ ][ ]   (খালি, শুধু "Origin দেখো" নির্দেশ)

 পরে Origin-এ C বদলে X লেখা হলো:
   ১. আগে পুরনো C কপি হলো snapshot-এ
   ২. তারপর Origin-এ X লেখা হলো
   Origin LV:  [A][B][X][D]     ◄── বর্তমান অবস্থা
   Snapshot:   [ ][ ][C][ ]

 Snapshot পড়লে: A, B (origin থেকে), C (snapshot থেকে), D (origin থেকে)
                = [A][B][C][D]  ◄── পুরনো অবস্থা অক্ষত!
```

তাই snapshot-এর জন্য যে জায়গা দাও, সেটা শুধু **পরিবর্তনগুলো** জমা রাখার জন্য। পুরো LV-এর সমান জায়গা লাগে না।

### Snapshot বানানো

```
$ sudo lvcreate -s -L 1G -n lv_web_snap /dev/vgdata/lv_web
  Logical volume "lv_web_snap" created.

$ sudo lvs
  LV          VG     Attr       LSize Pool Origin Data%  Meta%
  lv_web      vgdata owi-aos--- 4.00g
  lv_web_snap vgdata swi-a-s--- 1.00g      lv_web 0.01
```

`lv_web`-এর Attr শুরু হলো `o` (origin) দিয়ে, `lv_web_snap` শুরু হলো `s` (snapshot) দিয়ে। `Data%` বলছে snapshot-এর জায়গা কতটা ভরেছে।

> ⚠️ **সতর্কতা:** Snapshot-এর জায়গা (এখানে 1G) ১০০% ভরে গেলে snapshot **অকেজো (invalid)** হয়ে যায় এবং আর ব্যবহার করা যায় না। পুরনো ধরনের snapshot তাই স্বল্পমেয়াদি কাজের জন্য। `lvs` দিয়ে নিয়মিত `Data%` দেখো, দরকারে `lvextend` দিয়ে snapshot বাড়াও। `/etc/lvm/lvm.conf`-এ `snapshot_autoextend_threshold` সেট করে স্বয়ংক্রিয় বাড়ানোও যায় (এর জন্য `lvm2-monitor`/dmeventd চালু থাকতে হয়)।

> ⚠️ **সতর্কতা:** Snapshot থাকা অবস্থায় origin-এ প্রতিটা নতুন লেখার আগে একটা বাড়তি কপি হয়, তাই লেখার গতি কমে যায়। কাজ শেষে snapshot মুছে ফেলো।

### Snapshot দিয়ে কী করা যায়

**১. ধারাবাহিক (consistent) ব্যাকআপ:** চালু ডাটাবেসের ফাইল কপি করার সময় ফাইল বদলাতে থাকে। Snapshot নিয়ে সেটা read-only মাউন্ট করে কপি করলে সব ফাইল একই মুহূর্তের হয়।

```
$ sudo mkdir -p /mnt/snap
$ sudo mount -o ro /dev/vgdata/lv_web_snap /mnt/snap
$ sudo tar -czf /backup/web-$(date +%F).tar.gz -C /mnt/snap .
$ sudo umount /mnt/snap
$ sudo lvremove -y /dev/vgdata/lv_web_snap
  Logical volume "lv_web_snap" successfully removed.
```

> 📝 **মনে রাখো:** XFS snapshot মাউন্ট করতে `-o ro,nouuid` লাগে, কারণ snapshot আর origin-এর UUID একই, আর XFS একই UUID দুবার মাউন্ট করতে দেয় না।

**২. আপগ্রেডের আগে নিরাপত্তা:** বড় আপগ্রেডের আগে snapshot নাও। কিছু ভাঙলে আগের অবস্থায় ফিরে যাও।

### Snapshot থেকে ফিরে যাওয়া: `lvconvert --merge`

```
$ sudo umount /srv/web
$ sudo lvconvert --merge /dev/vgdata/lv_web_snap
  Merging of volume vgdata/lv_web_snap started.
  vgdata/lv_web: Merged: 100.00%
$ sudo mount /srv/web
```

Merge হলে origin snapshot-এর মুহূর্তের অবস্থায় ফিরে যায়, আর snapshot নিজে মুছে যায়। Origin ব্যবহারে থাকলে (যেমন রুট ফাইলসিস্টেম) merge পরের বার LV সক্রিয় হওয়ার সময় (সাধারণত রিবুটে) হয়।

### Thin provisioning ও thin snapshot (এক নজরে)

আধুনিক পদ্ধতি হলো **thin pool**। একটা pool বানিয়ে তার ভেতরে thin volume বানাও, যেগুলো যতটুকু লেখা হয় ততটুকুই জায়গা নেয়।

```
$ sudo lvcreate --type thin-pool -L 10G -n tpool vgdata
$ sudo lvcreate -V 50G --thinpool vgdata/tpool -n lv_thin
$ sudo lvcreate -s -n lv_thin_snap vgdata/lv_thin   # thin snapshot, সাইজ লাগে না
```

এখানে ১০ GB pool-এ ৫০ GB-এর ভলিউম! একে **over-provisioning** বলে। Thin snapshot অনেকগুলো রাখা যায় এবং গতি কম কমে। কিন্তু pool সত্যিই ভরে গেলে সব thin volume-এ লেখা আটকে যায়, তাই `lvs`-এ pool-এর `Data%` আর `Meta%` দুটোই নজরে রাখতে হয়।

| বিষয় | পুরনো (thick) snapshot | Thin snapshot |
|---|---|---|
| সাইজ দিতে হয় | হ্যাঁ (`-L`) | না |
| ভরে গেলে | snapshot অকেজো | pool ভরে, সব thin volume আটকায় |
| অনেকগুলো রাখা | ধীর হয়ে যায় | তুলনামূলক কার্যকর |
| ব্যবহার | স্বল্পমেয়াদি ব্যাকআপ | দীর্ঘমেয়াদি, VM/কন্টেইনার |

> ⚠️ **সতর্কতা:** Snapshot একই ডিস্কে থাকে। ডিস্ক নষ্ট হলে origin আর snapshot দুটোই যায়। তাই snapshot **ব্যাকআপ নয়**, এটা ব্যাকআপ নেওয়ার একটা হাতিয়ার। আসল ব্যাকআপ নিয়ে [অধ্যায় ৩৫](35-backup-rsync.md) দেখো।

---

## ১৪. RAID কী এবং কেন

**RAID (Redundant Array of Independent Disks)** মানে একাধিক ডিস্ককে একসাথে জুড়ে এমনভাবে ব্যবহার করা, যাতে নিচের এক বা একাধিক সুবিধা পাওয়া যায়:

- **Redundancy (অতিরিক্ত নিরাপত্তা):** একটা ডিস্ক নষ্ট হলেও ডেটা থাকে, সার্ভার চলতে থাকে।
- **Performance (গতি):** একসাথে কয়েকটা ডিস্ক থেকে পড়া/লেখা, তাই দ্রুত।
- **Capacity (ধারণক্ষমতা):** কয়েকটা ডিস্ক মিলিয়ে একটা বড় ডিস্ক।

**উপমা:** একটা গুরুত্বপূর্ণ চিঠির কথা ভাবো।
- **RAID 0** = চিঠিটা টুকরো করে দুইজন পিয়নকে দিলে, অর্ধেক অর্ধেক। দ্বিগুণ দ্রুত পৌঁছায়, কিন্তু একজন হারিয়ে গেলে পুরো চিঠি অর্থহীন।
- **RAID 1** = চিঠির দুটো ফটোকপি দুইজনকে দিলে। একজন হারালেও অন্যজনেরটা আছে।
- **RAID 5** = তিনজনকে চিঠির টুকরো দিলে, সাথে একটা "হিসাবের চিরকুট" (parity)। যেকোনো একজন হারালে বাকিদের টুকরো আর চিরকুট মিলিয়ে হারানো অংশ আবার বানানো যায়।

### মূল তিনটা কৌশল

| কৌশল | মানে |
|---|---|
| **Striping** | ডেটা ছোট টুকরোয় (chunk, সাধারণত 512 KiB — mdadm-এর ডিফল্ট) ভাগ করে ডিস্কগুলোতে পালাক্রমে লেখা |
| **Mirroring** | একই ডেটা একাধিক ডিস্কে হুবহু লেখা |
| **Parity** | ডেটা থেকে একটা গাণিতিক হিসাব (XOR) রাখা, যা দিয়ে হারানো ডেটা আবার বানানো যায় |

### Parity কীভাবে কাজ করে (XOR জাদু)

XOR (exclusive OR) এর নিয়ম: দুটো বিট একই হলে 0, আলাদা হলে 1। এর বিশেষ গুণ হলো, যেকোনো একটা মান হারালে বাকিগুলো XOR করলে সেটা ফিরে আসে।

```
 ডিস্ক১ (D1) = 1011
 ডিস্ক২ (D2) = 0110
 Parity  (P) = D1 XOR D2 = 1101

 ডিস্ক২ নষ্ট! ফেরত আনো:
 D2 = D1 XOR P = 1011 XOR 1101 = 0110   ✔ ঠিক ফিরে এল
```

RAID 5 ঠিক এভাবেই একটা ডিস্কের ক্ষতি সামলায়। RAID 6 দুটো আলাদা parity (P ও Q, দ্বিতীয়টা ভিন্ন গণিতে — Reed-Solomon) রাখে, তাই দুটো ডিস্ক একসাথে নষ্ট হলেও টিকে থাকে।

---

## ১৫. RAID লেভেলগুলো — ডায়াগ্রাম সহ

নিচের ডায়াগ্রামে A1, A2, ... হলো ডেটার টুকরো (block/chunk), `P` ও `Q` হলো parity।

### RAID 0 — Striping (গতি, নিরাপত্তা শূন্য)

```
   ডিস্ক ১    ডিস্ক ২
  ┌──────┐  ┌──────┐
  │  A1  │  │  A2  │
  │  A3  │  │  A4  │
  │  A5  │  │  A6  │
  └──────┘  └──────┘
  যেকোনো একটা নষ্ট → সব ডেটা শেষ
```

- কমপক্ষে ২টা ডিস্ক। ধারণক্ষমতা = সব ডিস্কের যোগফল।
- পড়া ও লেখা দুটোই দ্রুত।
- ব্যবহার: ভিডিও এডিটিং-এর অস্থায়ী (scratch) জায়গা, ক্যাশ — যেখানে ডেটা হারালে ক্ষতি নেই।

> ⚠️ **সতর্কতা:** RAID 0-তে নামেই "Redundant" কিন্তু কোনো redundancy নেই। ২টা ডিস্কে নষ্ট হওয়ার ঝুঁকি একটা ডিস্কের প্রায় দ্বিগুণ।

### RAID 1 — Mirroring (নিরাপত্তা, সরল)

```
   ডিস্ক ১    ডিস্ক ২
  ┌──────┐  ┌──────┐
  │  A1  │  │  A1  │
  │  A2  │  │  A2  │
  │  A3  │  │  A3  │
  └──────┘  └──────┘
  একটা নষ্ট → অন্যটা থেকে সব চলে
```

- কমপক্ষে ২টা ডিস্ক। ধারণক্ষমতা = একটা ডিস্কের সমান (৫০% কাজে লাগে)।
- পড়া দ্রুত হতে পারে (দুটো থেকে পড়া যায়), লেখা একটা ডিস্কের মতো।
- ব্যবহার: OS ডিস্ক, ছোট সার্ভার, boot ড্রাইভ।

### RAID 5 — Striping + একক distributed parity

```
   ডিস্ক ১    ডিস্ক ২    ডিস্ক ৩
  ┌──────┐  ┌──────┐  ┌──────┐
  │  A1  │  │  A2  │  │  Ap  │
  │  B1  │  │  Bp  │  │  B2  │
  │  Cp  │  │  C1  │  │  C2  │
  └──────┘  └──────┘  └──────┘
  parity সব ডিস্কে ঘুরিয়ে রাখা হয় (distributed)
  যেকোনো ১টা নষ্ট → টিকে থাকে
```

- কমপক্ষে ৩টা ডিস্ক। ধারণক্ষমতা = (N − 1) × ডিস্কের সাইজ।
- পড়া দ্রুত। ছোট লেখা ধীর, কারণ প্রতিটা লেখায় parity আবার হিসাব করতে হয় (read-modify-write, যাকে "write penalty" বলে)।
- Parity এক ডিস্কে না রেখে ঘুরিয়ে রাখা হয়, যাতে কোনো এক ডিস্কের ওপর চাপ না পড়ে (RAID 4-এ এক ডিস্কে রাখত, তাই সেটা bottleneck হতো)।

> ⚠️ **সতর্কতা:** বড় (কয়েক TB) HDD-তে RAID 5 এখন ঝুঁকিপূর্ণ ধরা হয়। একটা ডিস্ক নষ্ট হলে rebuild-এ অনেক ঘণ্টা থেকে দিন লাগে, এই সময় বাকি সব ডিস্ক পুরোটা পড়তে হয়। তখন আরেকটা ডিস্ক নষ্ট হলে বা একটা অপাঠযোগ্য সেক্টর (URE, Unrecoverable Read Error) পেলে ডেটা হারানোর ঝুঁকি থাকে। বড় অ্যারেতে RAID 6 বা RAID 10 পছন্দ করো।

### RAID 6 — Striping + দ্বৈত parity

```
   ডিস্ক ১   ডিস্ক ২   ডিস্ক ৩   ডিস্ক ৪
  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
  │ A1  │  │ A2  │  │ Ap  │  │ Aq  │
  │ B1  │  │ Bp  │  │ Bq  │  │ B2  │
  │ Cp  │  │ Cq  │  │ C1  │  │ C2  │
  └─────┘  └─────┘  └─────┘  └─────┘
  যেকোনো ২টা একসাথে নষ্ট → টিকে থাকে
```

- কমপক্ষে ৪টা ডিস্ক। ধারণক্ষমতা = (N − 2) × ডিস্কের সাইজ।
- লেখা RAID 5-এর চেয়েও ধীর (দুটো parity হিসাব)।
- ব্যবহার: বড় ফাইল সার্ভার, আর্কাইভ, NAS।

### RAID 10 (1+0) — Mirror-এর ওপর Stripe

```
         ┌─────────── Stripe (RAID 0) ───────────┐
         │                                       │
   Mirror ক (RAID 1)                 Mirror খ (RAID 1)
   ডিস্ক ১    ডিস্ক ২                  ডিস্ক ৩    ডিস্ক ৪
  ┌──────┐  ┌──────┐                ┌──────┐  ┌──────┐
  │  A1  │  │  A1  │                │  A2  │  │  A2  │
  │  A3  │  │  A3  │                │  A4  │  │  A4  │
  └──────┘  └──────┘                └──────┘  └──────┘
  প্রতি mirror জোড়া থেকে ১টা করে নষ্ট হলেও চলে।
  একই জোড়ার দুটোই নষ্ট হলে → সব শেষ।
```

- কমপক্ষে ৪টা ডিস্ক (জোড় সংখ্যা, প্রথাগত বিন্যাসে)। ধারণক্ষমতা = N / 2।
- পড়া-লেখা দুটোই দ্রুত, parity হিসাব নেই, rebuild দ্রুত (শুধু জোড়ার সঙ্গীর থেকে কপি)।
- ব্যবহার: ডাটাবেস সার্ভার, ভার্চুয়ালাইজেশন — যেখানে গতি আর নিরাপত্তা দুটোই চাই।

> 📝 **মনে রাখো:** Linux `mdadm`-এর RAID 10 একটা নিজস্ব ড্রাইভার, যা চাইলে ২ বা ৩টা ডিস্কেও চলে (`near`, `far`, `offset` বিন্যাস)। তবে প্রথাগত ধারণায় RAID 10 = ৪+ ডিস্ক।

### তুলনা টেবিল

| লেভেল | কৌশল | ন্যূনতম ডিস্ক | ব্যবহারযোগ্য জায়গা | কয়টা ডিস্ক নষ্ট সহ্য করে | পড়ার গতি | লেখার গতি | কোথায় ব্যবহার |
|---|---|---|---|---|---|---|---|
| **RAID 0** | Striping | ২ | N (১০০%) | ০ | খুব দ্রুত | খুব দ্রুত | অস্থায়ী/ক্যাশ ডেটা |
| **RAID 1** | Mirroring | ২ | 1 ডিস্ক (৫০% যদি ২টা) | N − 1 | দ্রুত | সাধারণ | OS, ছোট সার্ভার |
| **RAID 5** | Stripe + ১ parity | ৩ | N − 1 | ১ | দ্রুত | ধীর (write penalty) | ছোট/মাঝারি ফাইল সার্ভার |
| **RAID 6** | Stripe + ২ parity | ৪ | N − 2 | ২ | দ্রুত | আরো ধীর | বড় স্টোরেজ, আর্কাইভ |
| **RAID 10** | Mirror + Stripe | ৪ | N / 2 | প্রতি জোড়ায় ১টা | খুব দ্রুত | দ্রুত | ডাটাবেস, VM |

**উদাহরণ হিসাব:** ৪টা ৪ TB ডিস্ক থাকলে:

| লেভেল | ব্যবহারযোগ্য জায়গা |
|---|---|
| RAID 0 | ১৬ TB |
| RAID 1 (৪টাই mirror) | ৪ TB |
| RAID 5 | ১২ TB |
| RAID 6 | ৮ TB |
| RAID 10 | ৮ TB |

> 💡 **টিপস:** RAID-এ সব ডিস্ক একই সাইজের রাখো। ভিন্ন সাইজ হলে সবচেয়ে ছোট ডিস্কের সাইজ ধরে হিসাব হয়, বড় ডিস্কের বাড়তি জায়গা নষ্ট হয়।

---

## ১৬. Hardware RAID, Software RAID ও Fake RAID

| ধরন | কে কাজটা করে | সুবিধা | অসুবিধা |
|---|---|---|---|
| **Hardware RAID** | আলাদা RAID কন্ট্রোলার কার্ড (নিজস্ব প্রসেসর, ক্যাশ, প্রায়ই ব্যাটারি) | CPU-তে চাপ নেই, OS শুধু একটা ডিস্ক দেখে, battery-backed cache-এ দ্রুত লেখা | দামি; কার্ড নষ্ট হলে প্রায়ই একই মডেলের কার্ড লাগে |
| **Software RAID** | Linux kernel (md driver), `mdadm` দিয়ে নিয়ন্ত্রণ | ফ্রি, যেকোনো Linux মেশিনে ডিস্ক সরিয়ে চালানো যায়, স্বচ্ছ | CPU কিছুটা ব্যবহার করে (আধুনিক CPU-তে নগণ্য) |
| **Fake RAID** (BIOS/firmware RAID) | মাদারবোর্ডের firmware + OS ড্রাইভার | Windows-এর সাথে dual boot-এ সুবিধা | আসলে software RAID-ই, কিন্তু বেশি জটিল; Linux-এ সাধারণত এড়িয়ে চলা ভালো |

```
 Hardware RAID:                 Software RAID (md):
 ┌─────────────┐                ┌─────────────┐
 │  Linux      │                │  Linux      │
 │ দেখে: sda   │                │ md driver   │──► /dev/md0
 └──────┬──────┘                └──┬───┬───┬──┘
        │                          │   │   │
 ┌──────┴──────┐                 sdb  sdc  sdd
 │ RAID কার্ড   │
 └─┬───┬───┬───┘
  ডি১  ডি২  ডি৩
```

এই অধ্যায়ে আমরা Linux-এর software RAID শিখব, কারণ এটা সব জায়গায় একইভাবে কাজ করে।

> 📝 **মনে রাখো:** Hardware RAID থাকলে ডিস্কের স্বাস্থ্য দেখতে ভেন্ডরের টুল লাগে (যেমন `storcli`, `ssacli`, `perccli`)। `smartctl` সরাসরি নাও দেখতে পারে, বিশেষ অপশন লাগে (`-d megaraid,N` ইত্যাদি)।

---

## ১৭. `mdadm` দিয়ে Software RAID তৈরি

### `md` ড্রাইভার ও `mdadm`

Linux kernel-এর **md (multiple devices)** ড্রাইভার RAID-এর আসল কাজ করে। `mdadm` (multiple devices admin) হলো user-space টুল যা দিয়ে আমরা RAID বানাই, দেখি, মেরামত করি। প্রতিটা সদস্য ডিস্কে একটা **superblock** (metadata, বর্তমান ডিফল্ট সংস্করণ 1.2) লেখা হয়, যেখানে লেখা থাকে এটা কোন অ্যারের অংশ, কোন অবস্থানে।

```
# Ubuntu / Debian
$ sudo apt install mdadm

# Fedora / RHEL
$ sudo dnf install mdadm
```

### `mdadm` এর মোড ও প্রধান অপশন

```
mdadm [মোড] <raid-ডিভাইস> [অপশন] <সদস্য-ডিভাইস...>
```

| অপশন | কাজ |
|---|---|
| `--create` / `-C` | নতুন অ্যারে বানানো |
| `--level=N` / `-l N` | RAID লেভেল: `0`, `1`, `5`, `6`, `10` |
| `--raid-devices=N` / `-n N` | সক্রিয় সদস্য ডিস্কের সংখ্যা |
| `--spare-devices=N` / `-x N` | কয়টা spare (অতিরিক্ত, অপেক্ষমাণ) ডিস্ক |
| `--chunk=SIZE` / `-c` | chunk সাইজ (KiB), যেমন `--chunk=256` |
| `--detail` / `-D` | অ্যারের বিস্তারিত তথ্য |
| `--examine` / `-E` | একটা সদস্য ডিস্কের superblock পড়া |
| `--assemble` / `-A` | আগের অ্যারে আবার জোড়া লাগানো |
| `--scan` / `-s` | কনফিগ/ডিস্ক স্ক্যান |
| `--fail` / `-f` | ডিস্ককে নষ্ট হিসেবে চিহ্নিত করা |
| `--remove` / `-r` | অ্যারে থেকে ডিস্ক বের করা |
| `--add` / `-a` | ডিস্ক যোগ করা |
| `--grow` / `-G` | অ্যারের আকার/লেভেল/সদস্য সংখ্যা বদলানো |
| `--stop` / `-S` | অ্যারে বন্ধ করা |
| `--zero-superblock` | ডিস্ক থেকে md superblock মুছে ফেলা |
| `--monitor` / `-F` | অ্যারে নজরে রেখে সমস্যা হলে মেইল/অ্যালার্ট দেওয়া |

### RAID 1 বানানো

```
$ sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
Continue creating array? y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

### RAID 5 বানানো (৩ ডিস্ক + ১ spare)

```
$ sudo mdadm --create /dev/md1 --level=5 --raid-devices=3 \
       --spare-devices=1 /dev/sdb /dev/sdc /dev/sdd /dev/sde
```

Spare ডিস্ক চুপচাপ বসে থাকে। কোনো সক্রিয় ডিস্ক নষ্ট হলে md নিজে থেকে spare দিয়ে rebuild শুরু করে, মানুষের হাত লাগে না।

### অন্য লেভেল

```
# RAID 0
$ sudo mdadm --create /dev/md0 --level=0 --raid-devices=2 /dev/sdb /dev/sdc

# RAID 6
$ sudo mdadm --create /dev/md0 --level=6 --raid-devices=4 /dev/sd[b-e]

# RAID 10
$ sudo mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sd[b-e]
```

`/dev/sd[b-e]` হলো shell globbing, মানে `sdb sdc sdd sde` ([অধ্যায় ৯](09-file-management.md))।

### ফাইলসিস্টেম, মাউন্ট, fstab

RAID ডিভাইস তৈরির পরপরই ব্যবহার শুরু করা যায়, পটভূমিতে sync চলতে থাকলেও:

```
$ sudo mkfs.ext4 /dev/md0
$ sudo mkdir -p /srv/raid
$ sudo mount /dev/md0 /srv/raid

$ sudo blkid /dev/md0
/dev/md0: UUID="7c1f0a2e-5b3d-4e8a-9f21-0c6d8e4b1a77" BLOCK_SIZE="4096" TYPE="ext4"
```

fstab-এ UUID ব্যবহার করো (কারণ বুটে `/dev/md0` কখনো `/dev/md127` হয়ে যেতে পারে):

```
UUID=7c1f0a2e-5b3d-4e8a-9f21-0c6d8e4b1a77  /srv/raid  ext4  defaults,nofail  0 2
```

`nofail` দিলে অ্যারে কোনো কারণে না উঠলেও সিস্টেম বুট আটকে থাকবে না।

### কনফিগারেশন সেভ করা (খুব জরুরি)

অ্যারের তথ্য `mdadm.conf`-এ লিখে রাখো এবং initramfs আপডেট করো, যাতে বুটের সময় অ্যারে ঠিক নামে জোড়া লাগে:

```
# Ubuntu / Debian — ফাইল: /etc/mdadm/mdadm.conf
$ sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
ARRAY /dev/md0 metadata=1.2 UUID=3a5c7e9f:1b2d4f6a:8c0e2a4b:6d8f0a2c
$ sudo update-initramfs -u

# Fedora / RHEL — ফাইল: /etc/mdadm.conf
$ sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf
$ sudo dracut -f
```

> ⚠️ **সতর্কতা:** এই ধাপ বাদ দিলে রিবুটের পর অ্যারে প্রায়ই `/dev/md127` নামে উঠবে, আর fstab-এ `/dev/md0` লেখা থাকলে মাউন্ট ব্যর্থ হবে। `tee -a` দুবার চালালে ডুপ্লিকেট ARRAY লাইন হবে — ফাইলটা খুলে একবার দেখে নাও।

---

## ১৮. RAID-এর অবস্থা দেখা: `/proc/mdstat` ও `mdadm --detail`

### `/proc/mdstat`

`/proc/mdstat` হলো kernel-এর তৈরি একটা ভার্চুয়াল ফাইল ([অধ্যায় ৪৭](47-kernel-proc-sys.md)), যেটা সব md অ্যারের বর্তমান অবস্থা দেখায়। RAID অ্যাডমিনের সবচেয়ে প্রিয় ফাইল।

```
$ cat /proc/mdstat
Personalities : [raid1] [raid6] [raid5] [raid4]
md0 : active raid1 sdc[1] sdb[0]
      10475520 blocks super 1.2 [2/2] [UU]
      [=======>.............]  resync = 38.4% (4023296/10475520) finish=0.5min speed=201164K/sec

unused devices: <none>
```

লাইন ধরে পড়া:

| অংশ | মানে |
|---|---|
| `Personalities` | kernel-এ কোন RAID লেভেলের ড্রাইভার লোড আছে |
| `md0 : active raid1` | অ্যারের নাম, চালু আছে, লেভেল raid1 |
| `sdc[1] sdb[0]` | সদস্য ডিস্ক ও তাদের অবস্থান নম্বর |
| `10475520 blocks` | সাইজ (1 KiB ব্লকে) |
| `super 1.2` | superblock সংস্করণ |
| `[2/2]` | দরকার ২টা / চালু আছে ২টা |
| `[UU]` | প্রতিটা ডিস্কের অবস্থা: `U` = up (ঠিক আছে), `_` = নেই/নষ্ট |
| `resync` / `recovery` | sync বা rebuild চলছে, কত শতাংশ, আনুমানিক কত সময় বাকি |

নষ্ট অবস্থার উদাহরণ:

```
md0 : active raid1 sdb[0] sdc[1](F)
      10475520 blocks super 1.2 [2/1] [U_]
```

`(F)` = failed, `[2/1]` = দুটোর জায়গায় একটা চলছে, `[U_]` = দ্বিতীয় ডিস্ক নেই। এই অবস্থাকে বলে **degraded**। অ্যারে চলছে, কিন্তু আর কোনো নিরাপত্তা নেই। এখনই ডিস্ক বদলাও!

অন্যান্য চিহ্ন: `(S)` = spare।

লাইভ নজর রাখতে:

```
$ watch -n 2 cat /proc/mdstat
```

(বের হতে `Ctrl+C`)

### `mdadm --detail`

```
$ sudo mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Nov 12 10:42:17 2026
        Raid Level : raid1
        Array Size : 10475520 (9.99 GiB 10.73 GB)
     Used Dev Size : 10475520 (9.99 GiB 10.73 GB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

             State : clean
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

              Name : server1:0  (local to host server1)
              UUID : 3a5c7e9f:1b2d4f6a:8c0e2a4b:6d8f0a2c
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
```

`State` লাইন দেখো: `clean` (সব ঠিক), `active`, `degraded` (সমস্যা), `resyncing`, `recovering`।

### একটা ডিস্কের superblock দেখা

```
$ sudo mdadm --examine /dev/sdb
```

কোনো অপরিচিত ডিস্ক আগে RAID-এর অংশ ছিল কিনা জানতে এটা কাজে লাগে।

### স্বয়ংক্রিয় নজরদারি

ডিস্ক নষ্ট হলে তুমি জানবে কী করে? `mdadm.conf`-এ একটা লাইন যোগ করো:

```
MAILADDR admin@example.com
```

Debian/Ubuntu ও RHEL-এ mdadm-এর monitor সার্ভিস (systemd unit-এর নাম distro-ভেদে `mdmonitor.service` বা `mdadm`-এর সাথে আসা unit) এটা পড়ে মেইল পাঠায় (মেইল পাঠানোর জন্য সিস্টেমে MTA/`sendmail` সেটআপ থাকতে হবে)। পরীক্ষামূলক অ্যালার্ট পাঠাতে:

```
$ sudo mdadm --monitor --scan --test --oneshot
```

এছাড়া অনেক distro নিয়মিত (মাসে একবার ইত্যাদি) "scrub"/check চালায়, যা পুরো অ্যারে পড়ে লুকানো খারাপ সেক্টর খোঁজে। হাতে চালাতে:

```
$ echo check | sudo tee /sys/block/md0/md/sync_action
$ cat /sys/block/md0/md/mismatch_cnt
0
```

---

## ১৯. নষ্ট ডিস্ক বদলানো

এটাই RAID অ্যাডমিনের আসল পরীক্ষা। প্রক্রিয়াটা চার ধাপের:

```
  ┌────────┐     ┌──────────┐     ┌───────────────┐     ┌────────┐
  │ --fail │ ──► │ --remove │ ──► │ ডিস্ক বদলাও    │ ──► │ --add  │ ──► rebuild
  │ চিহ্নিত │     │ বের করো   │     │ (শারীরিকভাবে)  │     │ যোগ করো │     (স্বয়ংক্রিয়)
  └────────┘     └──────────┘     └───────────────┘     └────────┘
```

### ধাপ ১: কোন ডিস্ক নষ্ট খুঁজে বের করো

```
$ cat /proc/mdstat
md0 : active raid1 sdb[0] sdc[1](F)
      10475520 blocks super 1.2 [2/1] [U_]
```

`sdc` নষ্ট। কিন্তু সার্ভারের সামনে গিয়ে কোন ফিজিক্যাল ডিস্কটা `sdc`? সিরিয়াল নম্বর দেখো:

```
$ lsblk -o NAME,SIZE,SERIAL,MODEL
$ sudo smartctl -i /dev/sdc | grep -i serial
Serial Number:    WD-WX12A3456789
```

ডিস্কের গায়ের লেবেলে এই সিরিয়াল মিলিয়ে বের করবে।

> ⚠️ **সতর্কতা:** ভুল (ভালো) ডিস্ক খুলে ফেললে degraded RAID 1/5 পুরোপুরি বসে যাবে। সিরিয়াল না মিলিয়ে কখনো ডিস্ক খুলবে না।

### ধাপ ২: fail ও remove

kernel নিজেই যদি `(F)` চিহ্ন দিয়ে থাকে তাহলে fail চিহ্নিত করা লাগবে না। নিজে বদলাতে চাইলে (যেমন SMART সতর্কতা দিচ্ছে কিন্তু এখনো নষ্ট হয়নি):

```
$ sudo mdadm /dev/md0 --fail /dev/sdc
mdadm: set /dev/sdc faulty in /dev/md0

$ sudo mdadm /dev/md0 --remove /dev/sdc
mdadm: hot removed /dev/sdc from /dev/md0
```

### ধাপ ৩: শারীরিকভাবে ডিস্ক বদলাও

Hot-swap সাপোর্ট থাকলে চালু অবস্থায়, না থাকলে সার্ভার বন্ধ করে। নতুন ডিস্ক পুরনোটার সমান বা বড় হতে হবে।

যদি সদস্যগুলো পার্টিশন হয় (`/dev/sdb1` ইত্যাদি), নতুন ডিস্কে একই পার্টিশন টেবিল কপি করো:

```
# GPT ডিস্কের জন্য (gdisk প্যাকেজের sgdisk):
$ sudo sgdisk -R /dev/sdc /dev/sdb    # sdb থেকে sdc-তে টেবিল কপি (দিক খেয়াল করো!)
$ sudo sgdisk -G /dev/sdc             # নতুন ডিস্কে নতুন GUID

# MBR ডিস্কের জন্য:
$ sudo sfdisk -d /dev/sdb | sudo sfdisk /dev/sdc
```

### ধাপ ৪: add — rebuild স্বয়ংক্রিয়

```
$ sudo mdadm /dev/md0 --add /dev/sdc
mdadm: added /dev/sdc

$ cat /proc/mdstat
md0 : active raid1 sdc[2] sdb[0]
      10475520 blocks super 1.2 [2/1] [U_]
      [===>.................]  recovery = 17.2% (1802240/10475520) finish=0.7min speed=200249K/sec
```

`recovery` শেষ হয়ে `[UU]` দেখালে কাজ শেষ। এই সময়ে অ্যারে চলতে থাকে, তবে একটু ধীর।

> 💡 **টিপস:** Rebuild-এর গতি `/proc/sys/dev/raid/speed_limit_min` আর `speed_limit_max` (KiB/s) দিয়ে নিয়ন্ত্রিত। ব্যস্ত সার্ভারে ধীর করো, রাতে দ্রুত। 

> 💡 **টিপস:** রুট বা `/boot` RAID 1-এ থাকলে নতুন ডিস্কেও bootloader বসাতে ভুলো না (BIOS-এ `grub-install /dev/sdc`, UEFI-তে EFI পার্টিশন আলাদাভাবে কপি/সেটআপ)। নইলে পরে প্রথম ডিস্ক নষ্ট হলে সার্ভার বুট হবে না।

### অ্যারে বড় করা: `--grow`

RAID 5-এ চতুর্থ ডিস্ক যোগ করে জায়গা বাড়ানো:

```
$ sudo mdadm /dev/md1 --add /dev/sdf
$ sudo mdadm --grow /dev/md1 --raid-devices=4
$ cat /proc/mdstat          # "reshape" চলবে, অনেক সময় লাগতে পারে
$ sudo resize2fs /dev/md1   # reshape শেষে ফাইলসিস্টেম বাড়াও (ext4)
```

> ⚠️ **সতর্কতা:** Reshape হলো সবচেয়ে ঝুঁকিপূর্ণ RAID অপারেশনগুলোর একটা। শুরুর আগে ব্যাকআপ নাও।

### অ্যারে পুরোপুরি মুছে ফেলা

```
$ sudo umount /srv/raid
$ sudo mdadm --stop /dev/md0
mdadm: stopped /dev/md0
$ sudo mdadm --zero-superblock /dev/sdb /dev/sdc
```

তারপর `mdadm.conf` থেকে ARRAY লাইন আর `/etc/fstab` থেকে এন্ট্রি মুছে initramfs আপডেট করো। `--zero-superblock` না করলে পরে ডিস্কগুলো অন্য কাজে লাগালে md পুরনো অ্যারে চিনে জোড়া লাগাতে চাইবে।

---

## ২০. RAID কেন ব্যাকআপ নয়

এই অংশটা এই অধ্যায়ের সবচেয়ে গুরুত্বপূর্ণ বাক্য:

> ⚠️ **সতর্কতা:** **RAID ব্যাকআপ নয়।** RAID রক্ষা করে শুধু **হার্ডওয়্যার (ডিস্ক) নষ্ট হওয়া থেকে**, আর সার্ভারকে চালু রাখে (availability)। ব্যাকআপ রক্ষা করে **ডেটা হারানো থেকে**, যেকোনো কারণেই হোক।

RAID 1-এ তুমি একটা ফাইল মুছলে, সেটা **দুটো ডিস্ক থেকেই সঙ্গে সঙ্গে** মুছে যায়। RAID ঠিক যা করার কথা তাই করছে: দুটো ডিস্ক হুবহু এক রাখছে, ভুলটাসহ।

| বিপদ | RAID বাঁচায়? | ব্যাকআপ বাঁচায়? |
|---|---|---|
| একটা ডিস্ক নষ্ট | ✅ হ্যাঁ | ✅ হ্যাঁ (তবে রিস্টোরে সময় লাগে) |
| ভুল করে `rm -rf` | ❌ না, সব ডিস্কে মুছে যায় | ✅ হ্যাঁ |
| Ransomware/ভাইরাস ফাইল এনক্রিপ্ট করল | ❌ না, এনক্রিপ্টেড ফাইল mirror হয় | ✅ হ্যাঁ (অফলাইন/অপরিবর্তনীয় কপি থাকলে) |
| ফাইলসিস্টেম corruption, সফটওয়্যার বাগ | ❌ না, corrupt ডেটাও কপি হয় | ✅ হ্যাঁ |
| RAID কন্ট্রোলার বা PSU পুড়ে একাধিক ডিস্ক নষ্ট | ❌ সম্ভবত না | ✅ হ্যাঁ |
| আগুন, বন্যা, চুরি | ❌ না, সব ডিস্ক এক জায়গায় | ✅ হ্যাঁ (অন্য জায়গায় কপি থাকলে) |
| Rebuild-এর সময় আরেকটা ডিস্ক নষ্ট | ❌ (RAID 5-এ) | ✅ হ্যাঁ |

**উপমা:** RAID হলো গাড়ির স্পেয়ার টায়ার। টায়ার পাংচার হলে চলা থামে না। কিন্তু গাড়িটাই দুর্ঘটনায় পড়লে বা চুরি হলে স্পেয়ার টায়ার কোনো কাজে আসে না। তার জন্য লাগে ইনশিওরেন্স — সেটাই ব্যাকআপ।

> 📝 **মনে রাখো — 3-2-1 নিয়ম:** ডেটার **৩টা** কপি, **২টা** ভিন্ন মাধ্যমে, **১টা** অন্য জায়গায় (offsite)। বিস্তারিত [অধ্যায় ৩৫](35-backup-rsync.md)-এ।

একইভাবে, এক ব্যাচে কেনা একই মডেলের ডিস্কগুলো প্রায়ই কাছাকাছি সময়ে নষ্ট হয়। তাই RAID থাকলেও SMART নজরদারি ([অধ্যায় ৪৬](46-performance-troubleshooting.md)) আর নিয়মিত ব্যাকআপ — দুটোই লাগবে।

---

## ২১. LVM আর RAID একসাথে

বাস্তবে সার্ভারে সাধারণ বিন্যাস হলো: নিচে RAID (নিরাপত্তা), ওপরে LVM (নমনীয়তা)।

```
 ┌────────────────────────────────────────────────────┐
 │  ext4 (/)     xfs (/var/lib/mysql)    ext4 (/home)  │
 ├────────────────────────────────────────────────────┤
 │  LV root       LV mysql               LV home       │  ◄── LVM: সাইজ বদলানো, snapshot
 ├────────────────────────────────────────────────────┤
 │              VG vg0                                 │
 ├────────────────────────────────────────────────────┤
 │              PV /dev/md0                            │
 ├────────────────────────────────────────────────────┤
 │        /dev/md0  (RAID 10, mdadm)                   │  ◄── RAID: ডিস্ক নষ্ট হলে টিকে থাকা
 ├──────────┬──────────┬──────────┬───────────────────┤
 │ /dev/sdb │ /dev/sdc │ /dev/sdd │ /dev/sde          │
 └──────────┴──────────┴──────────┴───────────────────┘
```

```
$ sudo mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sd[b-e]
$ sudo pvcreate /dev/md0
$ sudo vgcreate vg0 /dev/md0
$ sudo lvcreate -L 30G -n mysql vg0
```

### LVM-এর নিজস্ব RAID

LVM নিজেও md-এর একই kernel কোড ব্যবহার করে RAID বানাতে পারে, প্রতি-LV ভিত্তিতে:

```
$ sudo lvcreate --type raid1 -m 1 -L 5G -n lv_safe vgdata
$ sudo lvs -a -o name,copy_percent,devices vgdata
  LV                 Cpy%Sync Devices
  lv_safe            100.00   lv_safe_rimage_0(0),lv_safe_rimage_1(0)
  [lv_safe_rimage_0]          /dev/sdb(0)
  [lv_safe_rimage_1]          /dev/sdc(0)
  ...
```

`-m 1` মানে মূলের সাথে ১টা অতিরিক্ত কপি (মোট ২টা)। `--type raid5 -i 2` মানে ২টা ডেটা stripe + parity (মোট ৩টা PV লাগবে)। সুবিধা হলো এক VG-তে কোনো LV RAID, কোনোটা সাধারণ রাখা যায়। তবে বেশি প্রচলিত ও ডকুমেন্টেড পথ হলো mdadm + LVM।

> 📝 **মনে রাখো:** ZFS আর Btrfs হলো এমন ফাইলসিস্টেম যাদের ভেতরেই RAID ও volume management আছে, সাথে checksum দিয়ে নীরব ডেটা নষ্ট হওয়া (bit rot) ধরার ক্ষমতা। এগুলো আলাদা জগৎ; এখানে শুধু জেনে রাখো যে বিকল্প আছে।

---

## ২২. সমস্যা সমাধান (Troubleshooting) এক নজরে

| লক্ষণ | সম্ভাব্য কারণ | কী করবে |
|---|---|---|
| `lvextend` করার পর `df` আগের সাইজ দেখাচ্ছে | ফাইলসিস্টেম বাড়ানো হয়নি | `resize2fs` (ext4) বা `xfs_growfs <mountpoint>` (XFS) |
| `Insufficient free space` | VG-তে খালি জায়গা নেই | `vgs` দেখো, নতুন PV এনে `vgextend` |
| রিবুটে অ্যারে `/dev/md127` | `mdadm.conf` নেই/পুরনো | `mdadm --detail --scan` দিয়ে conf আপডেট, initramfs আপডেট |
| বুটে emergency mode | fstab-এ ভুল ডিভাইস | রেসকিউ মোডে fstab ঠিক করো, `nofail` যোগ করো |
| `/proc/mdstat`-এ `[U_]` | ডিস্ক নষ্ট/বের হয়ে গেছে | `dmesg`, `smartctl` দেখো, ডিস্ক বদলাও |
| অ্যারে `inactive` | যথেষ্ট সদস্য পাওয়া যায়নি | `mdadm --examine` দিয়ে সদস্য দেখো, `mdadm --assemble --scan` |
| Snapshot `Invalid` | snapshot-এর জায়গা ভরে গেছে | ওই snapshot মুছে বড় সাইজে নতুন বানাও |
| `Device ... excluded by a filter` | `/etc/lvm/lvm.conf`-এর filter বা ডিস্কে পুরনো সিগনেচার | `wipefs` দিয়ে সিগনেচার দেখো, filter দেখো |

---

## ২৩. সংশ্লিষ্ট কীবোর্ড শর্টকাট ও কৌশল

LVM/RAID নিজস্ব কোনো কীবোর্ড শর্টকাট নেই, কিন্তু এই কাজগুলোতে যেগুলো সবচেয়ে কাজে লাগে (বিস্তারিত [অধ্যায় ৬](06-terminal-shortcuts.md)):

| শর্টকাট / কৌশল | কাজ |
|---|---|
| `Tab` / `Tab Tab` | `lv<Tab><Tab>` দিলে সব `lv*` কমান্ড দেখায়; bash-completion থাকলে VG/LV নামও পূরণ করে |
| `Ctrl+C` | `watch cat /proc/mdstat` বা `mdadm --monitor` থেকে বের হওয়া |
| `Ctrl+R` | আগের লম্বা `mdadm --create ...` কমান্ড খুঁজে বের করা (সাবধানে আবার চালাও!) |
| `Alt+.` | আগের কমান্ডের শেষ argument বসানো (যেমন `/dev/vgdata/lv_web`) |
| `sudo lvm` | interactive LVM shell; ভেতরে `lvs`, `vgs` ইত্যাদি সরাসরি চালাও, `exit` দিয়ে বের হও |
| `Ctrl+B d` (tmux) | লম্বা `pvmove`/reshape চালু রেখে tmux থেকে detach ([অধ্যায় ২৫](25-jobs-tmux.md)) |

---

## হাতে-কলমে অনুশীলন

আসল ডিস্ক নষ্ট না করে শেখার জন্য আমরা **loop device** ব্যবহার করব। Loop device হলো একটা সাধারণ ফাইলকে ব্লক ডিভাইসের মতো দেখানো। ভার্চুয়াল মেশিনে করাই সবচেয়ে নিরাপদ।

> ⚠️ **সতর্কতা:** নিচে `losetup`-এর দেখানো আসল নাম (যেমন `/dev/loop10`) ব্যবহার করবে। Ubuntu-তে snap প্যাকেজগুলোও loop device ব্যবহার করে, তাই নম্বর আলাদা হবে। আমরা নামগুলো ভেরিয়েবলে রেখে নিচ্ছি যাতে ভুল না হয়।

### ল্যাব ০: ভুয়া ডিস্ক বানানো

```
$ mkdir -p ~/lab && cd ~/lab
$ for i in 1 2 3 4; do truncate -s 1G disk$i.img; done
$ D1=$(sudo losetup -f --show disk1.img)
$ D2=$(sudo losetup -f --show disk2.img)
$ D3=$(sudo losetup -f --show disk3.img)
$ D4=$(sudo losetup -f --show disk4.img)
$ echo $D1 $D2 $D3 $D4
/dev/loop10 /dev/loop11 /dev/loop12 /dev/loop13
$ lsblk $D1 $D2 $D3 $D4
```

### ল্যাব ১: LVM তৈরি

1. দুটো PV বানাও: `sudo pvcreate $D1 $D2`
2. VG বানাও: `sudo vgcreate vglab $D1 $D2`
3. `sudo vgs` দিয়ে দেখো `VSize` প্রায় 2G কিনা।
4. 500M-এর একটা LV: `sudo lvcreate -L 500M -n lvtest vglab`
5. ফাইলসিস্টেম ও মাউন্ট:
   ```
   $ sudo mkfs.ext4 /dev/vglab/lvtest
   $ sudo mkdir -p /mnt/lvtest
   $ sudo mount /dev/vglab/lvtest /mnt/lvtest
   $ echo "হ্যালো LVM" | sudo tee /mnt/lvtest/note.txt
   ```
6. `sudo lvs -o +devices` চালিয়ে দেখো LV কোন loop device-এ আছে।

### ল্যাব ২: অনলাইন বড় করা

1. `df -h /mnt/lvtest` নোট করো।
2. `sudo lvextend -r -L +700M /dev/vglab/lvtest`
3. আবার `df -h /mnt/lvtest` — মাউন্ট খোলা না রেখেই কি বড় হলো? `note.txt` আছে তো?
4. চ্যালেঞ্জ: `-r` ছাড়া আরো 100M বাড়াও, `df` দেখো (বাড়েনি!), তারপর `resize2fs` চালিয়ে ঠিক করো।

### ল্যাব ৩: নতুন ডিস্ক যোগ ও pvmove

1. `sudo pvcreate $D3 && sudo vgextend vglab $D3`
2. `sudo pvs` — নতুন PV-এর `PFree` দেখো।
3. `$D1` খালি করো: `sudo pvmove $D1`
4. `sudo pvs` দেখো — `$D1`-এর `PFree` এখন পুরো সাইজ?
5. বাদ দাও: `sudo vgreduce vglab $D1 && sudo pvremove $D1`
6. `cat /mnt/lvtest/note.txt` — ডেটা অক্ষত আছে, অথচ একটা "ডিস্ক" সরিয়ে ফেললে!

### ল্যাব ৪: Snapshot ও merge

1. `sudo lvcreate -s -L 200M -n snap1 /dev/vglab/lvtest`
2. "দুর্ঘটনা" ঘটাও: `sudo rm /mnt/lvtest/note.txt`
3. `sudo lvs` — snapshot-এর `Data%` একটু বেড়েছে কি?
4. snapshot মাউন্ট করে দেখো ফাইল আছে কিনা:
   ```
   $ sudo mkdir -p /mnt/snap
   $ sudo mount -o ro /dev/vglab/snap1 /mnt/snap
   $ cat /mnt/snap/note.txt
   হ্যালো LVM
   $ sudo umount /mnt/snap
   ```
5. পুরোটা ফিরিয়ে আনো:
   ```
   $ sudo umount /mnt/lvtest
   $ sudo lvconvert --merge /dev/vglab/snap1
   $ sudo mount /dev/vglab/lvtest /mnt/lvtest
   $ cat /mnt/lvtest/note.txt
   ```

### ল্যাব ৫: lvreduce (নিরাপদ পথে)

1. `sudo umount /mnt/lvtest`
2. `sudo lvreduce -r -L 600M /dev/vglab/lvtest`
3. আবার মাউন্ট করে ফাইল দেখো।
4. ভেবে দেখো: XFS হলে এই ল্যাব কেন ব্যর্থ হতো?

### ল্যাব ৬: LVM পরিষ্কার

```
$ sudo umount /mnt/lvtest
$ sudo lvremove -y vglab/lvtest
$ sudo vgremove vglab
$ sudo pvremove $D2 $D3
```

### ল্যাব ৭: RAID 1 বানানো, নষ্ট করা, সারানো

1. বানাও:
   ```
   $ sudo mdadm --create /dev/md100 --level=1 --raid-devices=2 $D1 $D2
   ```
   (boot ডিভাইস সংক্রান্ত প্রশ্নে `y`)
2. `cat /proc/mdstat` — resync দেখো, `[UU]` হওয়া পর্যন্ত অপেক্ষা করো (`watch -n1 cat /proc/mdstat`, বের হতে `Ctrl+C`)।
3. ফাইলসিস্টেম ও ডেটা:
   ```
   $ sudo mkfs.ext4 /dev/md100
   $ sudo mkdir -p /mnt/raid && sudo mount /dev/md100 /mnt/raid
   $ echo "RAID টেস্ট" | sudo tee /mnt/raid/file.txt
   ```
4. ডিস্ক "নষ্ট" করো:
   ```
   $ sudo mdadm /dev/md100 --fail $D2
   $ cat /proc/mdstat        # [U_] আর (F) দেখো
   $ cat /mnt/raid/file.txt  # ডেটা এখনো আছে!
   ```
5. বদলাও:
   ```
   $ sudo mdadm /dev/md100 --remove $D2
   $ sudo mdadm /dev/md100 --add $D3
   $ watch -n1 cat /proc/mdstat   # recovery দেখো
   ```
6. `sudo mdadm --detail /dev/md100` — `State : clean`, সদস্য `$D1` আর `$D3`?

### ল্যাব ৮: "RAID ব্যাকআপ নয়" নিজের চোখে

```
$ sudo rm /mnt/raid/file.txt
$ ls /mnt/raid
lost+found
```

দুটো ডিস্ক থাকা সত্ত্বেও ফাইল নেই। কোনো "অন্য কপি" থেকে ফেরত পাওয়ার উপায় নেই। মনে গেঁথে নাও।

### ল্যাব ৯: সব পরিষ্কার

```
$ sudo umount /mnt/raid
$ sudo mdadm --stop /dev/md100
$ sudo mdadm --zero-superblock $D1 $D2 $D3
$ sudo losetup -d $D1 $D2 $D3 $D4
$ rm ~/lab/disk*.img
```

### বোনাস চ্যালেঞ্জ

- চারটা loop device দিয়ে RAID 5 বানাও (৩টা সক্রিয় + ১টা spare)। একটা সক্রিয় ডিস্ক `--fail` করো এবং দেখো spare নিজে থেকে rebuild শুরু করে কিনা।
- RAID 1 অ্যারের ওপর PV → VG → LV বানিয়ে "LVM on RAID" সাজাও।

---

## সারসংক্ষেপ

### LVM দ্রুত রেফারেন্স

| কাজ | কমান্ড |
|---|---|
| PV বানানো | `pvcreate /dev/sdb` |
| VG বানানো | `vgcreate vgdata /dev/sdb /dev/sdc` |
| LV বানানো (সাইজ) | `lvcreate -L 10G -n lvname vgdata` |
| LV বানানো (সব খালি জায়গা) | `lvcreate -l 100%FREE -n lvname vgdata` |
| সংক্ষিপ্ত অবস্থা | `pvs`, `vgs`, `lvs` |
| বিস্তারিত অবস্থা | `pvdisplay`, `vgdisplay`, `lvdisplay` |
| LV + ফাইলসিস্টেম বড় করা | `lvextend -r -L +5G /dev/vgdata/lvname` |
| শুধু ext4 বড় করা | `resize2fs /dev/vgdata/lvname` |
| শুধু XFS বড় করা | `xfs_growfs /mountpoint` |
| LV ছোট করা (ext4, umount করে) | `lvreduce -r -L 8G /dev/vgdata/lvname` |
| VG-তে ডিস্ক যোগ | `vgextend vgdata /dev/sdd` |
| PV থেকে ডেটা সরানো | `pvmove /dev/sdb` |
| VG থেকে ডিস্ক বাদ | `vgreduce vgdata /dev/sdb` → `pvremove /dev/sdb` |
| ক্লাউড ডিস্ক বড় হওয়ার পর | `growpart` → `pvresize` → `lvextend -r` |
| Snapshot বানানো | `lvcreate -s -L 1G -n snap /dev/vgdata/lvname` |
| Snapshot ফেরত আনা | `lvconvert --merge /dev/vgdata/snap` |
| LV মুছে ফেলা | `lvremove /dev/vgdata/lvname` |
| device-mapper দেখা | `dmsetup ls`, `dmsetup table` |

### RAID দ্রুত রেফারেন্স

| কাজ | কমান্ড |
|---|---|
| RAID বানানো | `mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc` |
| অবস্থা দেখা | `cat /proc/mdstat` |
| বিস্তারিত | `mdadm --detail /dev/md0` |
| সদস্যের superblock | `mdadm --examine /dev/sdb` |
| কনফিগ সেভ (Debian/Ubuntu) | `mdadm --detail --scan >> /etc/mdadm/mdadm.conf` + `update-initramfs -u` |
| কনফিগ সেভ (RHEL/Fedora) | `mdadm --detail --scan >> /etc/mdadm.conf` + `dracut -f` |
| ডিস্ক নষ্ট চিহ্নিত | `mdadm /dev/md0 --fail /dev/sdc` |
| ডিস্ক বের করা | `mdadm /dev/md0 --remove /dev/sdc` |
| ডিস্ক যোগ | `mdadm /dev/md0 --add /dev/sdd` |
| অ্যারে বড় করা | `mdadm --grow /dev/md0 --raid-devices=4` |
| অ্যারে বন্ধ | `mdadm --stop /dev/md0` |
| সব অ্যারে জোড়া | `mdadm --assemble --scan` |
| superblock মোছা | `mdadm --zero-superblock /dev/sdb` |
| লাইভ নজর | `watch cat /proc/mdstat` |

### RAID লেভেল মুখস্থ সূত্র

| লেভেল | ন্যূনতম | জায়গা | সহ্য করে |
|---|---|---|---|
| 0 | ২ | N | ০ |
| 1 | ২ | 1 | N−1 |
| 5 | ৩ | N−1 | ১ |
| 6 | ৪ | N−2 | ২ |
| 10 | ৪ | N/2 | প্রতি জোড়ায় ১ |

---

## নিজেকে যাচাই করো

1. PV, VG আর LV এর সম্পর্ক একটা উপমা দিয়ে ব্যাখ্যা করো। PE কী?
2. `lvextend -L +5G /dev/vg/lv` চালানোর পর `df -h` আগের সাইজ দেখাচ্ছে। কেন? দুটো উপায়ে সমাধান লেখো।
3. `resize2fs` আর `xfs_growfs` কী argument নেয়, এবং কোনটা ফাইলসিস্টেম ছোটও করতে পারে?
4. `lvreduce` চালানোর সঠিক ক্রম কী, আর ভুল ক্রমে কী হয়?
5. LVM snapshot-এর জন্য origin LV-এর সমান জায়গা লাগে না কেন? Snapshot-এর জায়গা ভরে গেলে কী হয়?
6. ৬টা ২ TB ডিস্ক দিয়ে RAID 5, RAID 6 আর RAID 10-এ কত ব্যবহারযোগ্য জায়গা পাবে? প্রতিটা কয়টা ডিস্ক নষ্ট সহ্য করতে পারে?
7. `/proc/mdstat`-এ `[3/2] [UU_]` এর অর্থ কী? তুমি কী করবে?
8. RAID অ্যারে বানানোর পর `mdadm.conf` আপডেট আর initramfs পুনর্নির্মাণ কেন জরুরি?
9. "আমার সার্ভারে RAID 1 আছে, তাই ব্যাকআপ লাগবে না" — এই কথার তিনটা ভুল দেখাও।
10. সার্ভার চালু রেখে একটা পুরনো LVM ডিস্ক নতুন ডিস্ক দিয়ে বদলানোর ধাপগুলো লেখো।

<details><summary>উত্তর দেখো</summary>

1. PV হলো আলাদা জমির প্লট (ডিস্ক/পার্টিশন), VG হলো সব প্লট মিলিয়ে একটা খামার (স্টোরেজ পুল), LV হলো খামারের ভেতরের আলাদা ক্ষেত (যার ওপর ফাইলসিস্টেম বসে)। PE (Physical Extent) হলো VG-এর জায়গার সমান মাপের ছোট টুকরো (ডিফল্ট 4 MiB); LV বানানো বা বাড়ানো মানে কিছু PE বরাদ্দ করা।

2. শুধু LV (block device) বড় হয়েছে, ওপরের ফাইলসিস্টেম এখনো পুরনো সাইজ জানে। সমাধান: (ক) ফাইলসিস্টেম আলাদাভাবে বাড়াও — ext4 হলে `resize2fs /dev/vg/lv`, XFS হলে `xfs_growfs /mountpoint`; (খ) পরের বার `lvextend -r` ব্যবহার করো যাতে একসাথে দুটোই হয়।

3. `resize2fs` নেয় ডিভাইস পাথ (এবং ঐচ্ছিক সাইজ), ext4 বড় করা মাউন্ট অবস্থাতেও যায়, ছোট করা শুধু আনমাউন্ট অবস্থায়। `xfs_growfs` নেয় মাউন্ট পয়েন্ট, শুধু বড় করতে পারে। তাই ছোট করতে পারে শুধু `resize2fs` (ext4)। XFS কখনো ছোট করা যায় না।

4. ব্যাকআপ → আনমাউন্ট → `e2fsck -f` → আগে ফাইলসিস্টেম ছোট (`resize2fs`) → তারপর LV ছোট (`lvreduce`) → মাউন্ট। অথবা সোজা `lvreduce -r`। ভুল ক্রমে (আগে LV ছোট) ফাইলসিস্টেমের শেষের অংশ কেটে যায়, ডেটা ও ফাইলসিস্টেম নষ্ট হয়।

5. Snapshot Copy-on-Write পদ্ধতিতে কাজ করে: শুরুতে কিছু কপি হয় না, শুধু origin-এর যে ব্লক বদলায় তার পুরনো সংস্করণ snapshot-এ রাখা হয়। তাই জায়গা লাগে শুধু পরিবর্তনের পরিমাণের সমান। জায়গা ১০০% ভরে গেলে (পুরনো ধরনের) snapshot invalid হয়ে যায়, আর ব্যবহার করা যায় না।

6. RAID 5: (6−1)×2 = ১০ TB, ১টা ডিস্ক নষ্ট সহ্য করে। RAID 6: (6−2)×2 = ৮ TB, যেকোনো ২টা। RAID 10: 6/2×2 = ৬ TB, প্রতিটা mirror জোড়া থেকে ১টা করে (সর্বোচ্চ ৩টা, তবে একই জোড়ার দুটো নষ্ট হলে সব শেষ)।

7. অ্যারেতে ৩টা ডিস্ক থাকার কথা, চলছে ২টা; তৃতীয় অবস্থানের ডিস্ক নেই বা নষ্ট — অ্যারে degraded। করণীয়: `mdadm --detail` আর `dmesg`/`smartctl` দিয়ে কোন ডিস্ক বুঝে নাও, সিরিয়াল মিলিয়ে বের করো, (দরকারে `--fail`) `--remove`, নতুন ডিস্ক লাগিয়ে `--add`, তারপর `/proc/mdstat`-এ recovery শেষ হওয়া পর্যন্ত নজর রাখো।

8. বুটের সময় (প্রায়ই initramfs-এর ভেতর থেকে) অ্যারে জোড়া লাগানো হয়। `mdadm.conf`-এ ARRAY লাইন না থাকলে বা initramfs-এ পুরনো কনফিগ থাকলে অ্যারে `/dev/md127`-এর মতো অন্য নামে উঠতে পারে বা না-ও উঠতে পারে, ফলে fstab-এর মাউন্ট ব্যর্থ হয়ে বুট আটকে যেতে পারে।

9. (ক) ভুল করে মুছে ফেলা ফাইল দুটো ডিস্ক থেকেই মুছে যায়; (খ) ransomware, ভাইরাস বা ফাইলসিস্টেম corruption-ও mirror হয়ে যায়; (গ) আগুন/চুরি/কন্ট্রোলার বা পাওয়ার সাপ্লাই নষ্টে একসাথে সব ডিস্ক যেতে পারে। RAID শুধু ডিস্ক নষ্ট হওয়া থেকে রক্ষা করে ও uptime দেয়; ব্যাকআপ (3-2-1 নিয়ম) আলাদা প্রয়োজন।

10. নতুন ডিস্ক লাগাও → `pvcreate /dev/sdNEW` → `vgextend vgname /dev/sdNEW` → `pvmove /dev/sdOLD` (ডেটা সরে যাবে, সার্ভার চালু থাকবে) → `vgreduce vgname /dev/sdOLD` → `pvremove /dev/sdOLD` → পুরনো ডিস্ক খুলে ফেলো।

</details>

---

[← আগের অধ্যায়](32-disk-partition-mount.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](34-archive-compression.md)

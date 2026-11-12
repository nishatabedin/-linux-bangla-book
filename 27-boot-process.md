# অধ্যায় ২৭: লিনাক্স বুট প্রসেস — পাওয়ার বাটন থেকে লগইন পর্যন্ত

পাওয়ার বাটন চাপার পর মাত্র কয়েক সেকেন্ডে লগইন স্ক্রিন চলে আসে। কিন্তু এই কয়েক সেকেন্ডের ভেতরে অনেকগুলো প্রোগ্রাম পালা করে কাজ করে। প্রত্যেকে নিজের কাজটুকু সেরে পরের জনকে দায়িত্ব দেয়, রিলে দৌড়ের মতো। এই অধ্যায়ে সেই দৌড়টা শুরু থেকে শেষ পর্যন্ত দেখব।

বুট প্রসেস বোঝা কেন দরকার? যতদিন সব ঠিক চলে, ততদিন দরকার পড়ে না। কিন্তু যেদিন মেশিন বুট হয় না, কালো স্ক্রিনে `grub rescue>` বা `Give root password for maintenance` লেখা আসে, অথবা root পাসওয়ার্ড ভুলে যাও, সেদিন এই জ্ঞানই তোমাকে বাঁচাবে। বুট প্রসেস যে বোঝে, সে জানে কোন ধাপে সমস্যা হয়েছে, তাই কোথায় হাত দিতে হবে সেটাও জানে।

## এই অধ্যায়ে যা শিখবে

- পাওয়ার বাটন থেকে লগইন প্রম্পট পর্যন্ত বুটের প্রতিটি ধাপ, একটা পূর্ণাঙ্গ ডায়াগ্রামসহ
- Firmware কী: BIOS আর UEFI-র পার্থক্য, POST কী করে
- ডিস্কের পার্টিশন টেবিল: MBR বনাম GPT, আর ESP (EFI System Partition) কেন লাগে
- Bootloader GRUB2: `grub.cfg`, `/etc/default/grub`, `update-grub` / `grub2-mkconfig`, `grubby`
- Kernel parameter কী, কীভাবে সাময়িক বা স্থায়ীভাবে বদলায়
- Kernel কীভাবে লোড হয়, আর initramfs কেন লাগে (`update-initramfs`, `dracut`)
- PID 1 হিসেবে systemd, targets, আর runlevel-এর সাথে তাদের সম্পর্ক
- getty, login আর display manager কীভাবে লগইন স্ক্রিন দেখায়
- `dmesg`, `journalctl -b`, `systemd-analyze` দিয়ে বুট পরীক্ষা করা
- Rescue mode আর emergency mode
- ভুলে যাওয়া root বা ইউজার পাসওয়ার্ড রিসেট করা (Ubuntu/Debian ও RHEL/Fedora দুই পদ্ধতিতেই)
- Secure Boot, shim, MOK কী এবং কেন
- বুট-সংক্রান্ত সব কীবোর্ড শর্টকাট

---

## ২৭.১ বড় ছবিটা আগে দেখি

পুরো বুট প্রসেসটা এক নজরে:

```
 ┌──────────────────────────────────────────────────────────────────┐
 │  [১] পাওয়ার বাটন চাপা                                           │
 │       │  CPU চালু হয়, মাদারবোর্ডের ROM/flash থেকে firmware চলে     │
 │       ▼                                                          │
 │  [২] FIRMWARE (BIOS বা UEFI)                                     │
 │       │  POST: RAM, CPU, কীবোর্ড, ডিস্ক ঠিক আছে কি না যাচাই        │
 │       │  boot order দেখে কোন ডিস্ক থেকে বুট হবে ঠিক করে            │
 │       ▼                                                          │
 │  [৩] ডিস্কের শুরু / ESP                                          │
 │       │  BIOS  → MBR-এর প্রথম 446 byte-এর boot code               │
 │       │  UEFI  → ESP থেকে .efi ফাইল (shim → grubx64.efi)          │
 │       ▼                                                          │
 │  [৪] BOOTLOADER (GRUB2)                                          │
 │       │  মেনু দেখায়, grub.cfg পড়ে                                 │
 │       │  kernel (vmlinuz) + initramfs RAM-এ লোড করে                │
 │       │  kernel parameter (root=, quiet ...) দিয়ে kernel চালায়     │
 │       ▼                                                          │
 │  [৫] KERNEL                                                      │
 │       │  নিজেকে decompress করে, CPU/মেমরি/ড্রাইভার চালু করে       │
 │       │  initramfs-কে অস্থায়ী root হিসেবে মাউন্ট করে              │
 │       ▼                                                          │
 │  [৬] INITRAMFS                                                   │
 │       │  ডিস্ক ড্রাইভার, LVM, RAID, LUKS খোলে                      │
 │       │  আসল root filesystem মাউন্ট করে → switch_root              │
 │       ▼                                                          │
 │  [৭] /sbin/init = systemd (PID 1)                                │
 │       │  default.target পর্যন্ত সব unit চালু করে                  │
 │       │  (sysinit → basic → multi-user → graphical)               │
 │       ▼                                                          │
 │  [৮] LOGIN                                                       │
 │       │  টেক্সট: getty → login → shell                            │
 │       │  গ্রাফিক্যাল: display manager (GDM/SDDM/LightDM) → ডেস্কটপ │
 │       ▼                                                          │
 │      তুমি লগইন করলে! 🎉                                           │
 └──────────────────────────────────────────────────────────────────┘
```

> 💡 **উপমা:** বুট প্রসেস একটা স্কুল খোলার মতো। ভোরে দারোয়ান (firmware) গেট খুলে দেখে বিদ্যুৎ, পানি ঠিক আছে কি না (POST)। তারপর সে অফিস সহকারীকে (bootloader) ডাকে। সহকারী রুটিন খাতা (`grub.cfg`) দেখে প্রধান শিক্ষককে (kernel) ডেকে আনে, সাথে একটা ব্যাগভর্তি জরুরি জিনিসপত্র (initramfs) দেয়। প্রধান শিক্ষক ব্যাগ থেকে চাবি বের করে আসল ক্লাসরুমগুলো (root filesystem) খোলেন। তারপর ভাইস-প্রিন্সিপাল (systemd) সব শিক্ষককে (service) তাঁদের ক্লাসে পাঠান। সবশেষে গেটে রিসেপশনিস্ট (getty/display manager) বসে ছাত্রদের (ইউজার) নাম-পরিচয় যাচাই করে ভেতরে ঢুকতে দেয়।

প্রত্যেক ধাপের একটাই মূল কাজ: **পরের ধাপকে খুঁজে বের করে মেমরিতে লোড করা এবং তার হাতে নিয়ন্ত্রণ দেওয়া।** কোনো ধাপ ব্যর্থ হলে তার পরের ধাপগুলো আর শুরুই হয় না। তাই কোন স্ক্রিনে আটকে আছ সেটা দেখেই বোঝা যায় সমস্যা কোন ধাপে।

| কোথায় আটকে আছে / কী দেখছ | সম্ভাব্য সমস্যার ধাপ |
|---|---|
| কিছুই দেখায় না, বিপ শব্দ হয় | হার্ডওয়্যার / POST |
| "No bootable device" / "Operating System not found" | firmware বুট ডিভাইস পাচ্ছে না, বা MBR/ESP নষ্ট |
| `grub rescue>` প্রম্পট | GRUB নিজের ফাইল (`/boot/grub`) খুঁজে পাচ্ছে না |
| `grub>` প্রম্পট (মেনু নেই) | `grub.cfg` নেই বা নষ্ট |
| "Kernel panic - not syncing: VFS: Unable to mount root fs" | kernel root খুঁজে পাচ্ছে না, initramfs বা `root=` প্যারামিটারে সমস্যা |
| `(initramfs)` প্রম্পট (BusyBox) বা dracut emergency shell | initramfs আসল root মাউন্ট করতে পারেনি |
| "You are in emergency mode" | systemd কোনো জরুরি মাউন্ট (প্রায়ই `/etc/fstab`-এর ভুল) করতে পারেনি |
| লগইন স্ক্রিন আসে না, কালো স্ক্রিন বা কার্সর | display manager বা গ্রাফিক্স ড্রাইভারের সমস্যা |

এবার প্রতিটি ধাপ বিস্তারিত দেখি।

---

## ২৭.২ ধাপ ১: Firmware — BIOS ও UEFI

### Firmware কী?

**Firmware** হলো মাদারবোর্ডের একটা flash চিপে স্থায়ীভাবে রাখা সফটওয়্যার। হার্ডওয়্যার আর সফটওয়্যারের মাঝামাঝি বলে এর নাম "firm"। পাওয়ার অন করলে CPU সবার আগে এটাকেই চালায়, কারণ তখনো RAM ফাঁকা আর ডিস্ক থেকে কিছু পড়া হয়নি।

এর দুটি প্রজন্ম আছে:

- **BIOS** (Basic Input/Output System): ১৯৮০-র দশকের IBM PC থেকে চলে আসছে। 16-bit real mode-এ চলে, এক MB মেমরির সীমার মধ্যে কাজ করে।
- **UEFI** (Unified Extensible Firmware Interface): আধুনিক বিকল্প। ২০১০-এর পর থেকে প্রায় সব কম্পিউটারে এটাই থাকে। 32/64-bit মোডে চলে, নিজেই FAT ফাইলসিস্টেম পড়তে পারে, মাউস-চালিত সেটআপ স্ক্রিন থাকে, Secure Boot সাপোর্ট করে।

> 📝 **মনে রাখো:** অনেকে এখনও UEFI সেটআপ স্ক্রিনকে "BIOS" বলে। আধুনিক মেশিনে "BIOS-এ ঢোকো" মানে আসলে UEFI সেটআপে ঢোকা। অনেক UEFI-তে **CSM** (Compatibility Support Module) নামে একটা "Legacy BIOS" মোডও থাকে, যেটা পুরনো BIOS-এর মতো আচরণ করে। নতুন অনেক মেশিন থেকে CSM সরিয়ে দেওয়া হচ্ছে।

### POST — Power-On Self-Test

Firmware প্রথমে **POST** চালায়। মানে নিজে নিজে হার্ডওয়্যার পরীক্ষা:

1. CPU রেজিস্টার ও firmware-এর নিজের অখণ্ডতা যাচাই
2. RAM শনাক্ত ও পরীক্ষা
3. ভিডিও কার্ড চালু (এ জন্যই স্ক্রিনে প্রথম লোগো আসে)
4. কীবোর্ড, স্টোরেজ কন্ট্রোলার, USB ইত্যাদি শনাক্ত

কোনো গুরুতর সমস্যা পেলে স্ক্রিন চালু হওয়ার আগেই নির্দিষ্ট প্যাটার্নে **বিপ কোড** বাজায়, বা মাদারবোর্ডের LED জ্বালায়। প্যাটার্নের মানে নির্মাতা-ভেদে আলাদা, তাই মাদারবোর্ডের ম্যানুয়াল দেখতে হয়।

POST সফল হলে firmware তার **boot order** (বুট ক্রম) দেখে: আগে USB, তারপর NVMe ডিস্ক ইত্যাদি। তালিকার প্রথম বুটযোগ্য ডিভাইসে যায়।

### BIOS আর UEFI বুট করে কীভাবে?

এটাই আসল পার্থক্য:

```
   BIOS বুট                                 UEFI বুট
   ───────                                 ────────
 ┌──────────┐                           ┌──────────────┐
 │   BIOS   │                           │    UEFI      │
 └────┬─────┘                           └──────┬───────┘
      │ ডিস্কের প্রথম সেক্টর (512 byte)          │ NVRAM-এ রাখা boot entry পড়ে
      │ অন্ধভাবে RAM-এ কপি করে চালায়            │ ("ubuntu" → \EFI\ubuntu\shimx64.efi)
      ▼                                        ▼
 ┌──────────┐                           ┌──────────────┐
 │   MBR    │ 446 byte boot code         │ ESP (FAT32)  │ ফাইলসিস্টেম পড়ে
 │ boot.img │                            │ পার্টিশন     │ .efi ফাইল খোঁজে
 └────┬─────┘                           └──────┬───────┘
      │                                        │
      ▼                                        ▼
 ┌──────────┐                           ┌──────────────┐
 │ core.img │ MBR-এর পরের ফাঁকা জায়গায় │ shimx64.efi  │ (Secure Boot)
 │ (GRUB)   │ বা BIOS boot partition-এ   │ → grubx64.efi│
 └────┬─────┘                           └──────┬───────┘
      ▼                                        ▼
  /boot/grub/ থেকে মডিউল + grub.cfg          grub.cfg → kernel
```

**BIOS** ফাইলসিস্টেম চেনে না। সে শুধু ডিস্কের প্রথম 512 byte পড়ে চালিয়ে দেয়। ওই ছোট্ট জায়গায় পুরো bootloader আঁটে না, তাই GRUB কয়েক ধাপে নিজেকে লোড করে (boot.img → core.img → মডিউল)।

**UEFI** নিজেই FAT32 পড়তে পারে। ডিস্কের একটা বিশেষ পার্টিশন (ESP) থেকে সরাসরি একটা `.efi` প্রোগ্রাম ফাইল খুঁজে চালায়। কোন ফাইল চালাবে সেটা মাদারবোর্ডের **NVRAM**-এ (non-volatile RAM, বিদ্যুৎ গেলেও মনে রাখে) boot entry হিসেবে লেখা থাকে।

| বৈশিষ্ট্য | BIOS (Legacy) | UEFI |
|---|---|---|
| চালু হওয়ার সময় | 1980-এর দশক | 2005-এর পর থেকে ব্যাপক |
| CPU মোড | 16-bit real mode | 32/64-bit |
| বুট পদ্ধতি | ডিস্কের প্রথম সেক্টর চালায় | ESP থেকে `.efi` ফাইল চালায় |
| পার্টিশন টেবিল | সাধারণত MBR | সাধারণত GPT |
| ডিস্কের সীমা (MBR-এ) | 2 TiB | কার্যত সীমাহীন (GPT) |
| Secure Boot | নেই | আছে |
| boot entry | নেই (শুধু ডিস্কের ক্রম) | NVRAM-এ নামসহ entry |
| সেটআপ ইন্টারফেস | টেক্সট, কীবোর্ড | প্রায়ই গ্রাফিক্যাল, মাউস চলে |

### তোমার সিস্টেম কোন মোডে বুট হয়েছে?

সবচেয়ে সহজ পরীক্ষা: `/sys/firmware/efi` ডিরেক্টরি আছে কি না।

```
$ [ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS (Legacy)"
UEFI
```

UEFI হলে কত-bit তাও দেখা যায়:

```
$ cat /sys/firmware/efi/fw_platform_size
64
```

### `efibootmgr` — UEFI boot entry দেখা ও বদলানো

UEFI-র NVRAM-এ রাখা boot entry লিনাক্স থেকেই দেখা আর বদলানো যায়।

```
$ efibootmgr
BootCurrent: 0001
Timeout: 1 seconds
BootOrder: 0001,0000,0002
Boot0000* Windows Boot Manager
Boot0001* ubuntu
Boot0002* UEFI: USB Flash Drive
```

`-v` দিলে প্রতিটি entry কোন পার্টিশনের কোন ফাইল চালায় তা দেখায়:

```
$ efibootmgr -v
Boot0001* ubuntu	HD(1,GPT,3f2a...,0x800,0x100000)/File(\EFI\ubuntu\shimx64.efi)
```

| অপশন | কাজ |
|---|---|
| `-v` | বিস্তারিত (ফাইলের পাথসহ) |
| `-o 0001,0000` | boot order বদলাও |
| `-n 0002` | শুধু পরের একবার এই entry থেকে বুট হবে (BootNext) |
| `-b 0003 -B` | `0003` নম্বর entry মুছে ফেলো |
| `-c -d /dev/nvme0n1 -p 1 -L "নাম" -l '\EFI\x\y.efi'` | নতুন entry তৈরি |

```
$ sudo efibootmgr -o 0001,0000,0002
```

> ⚠️ **সতর্কতা:** `efibootmgr` দিয়ে ভুল entry মুছে ফেললে মেশিন বুট নাও হতে পারে। তবে ESP-র ফাইলগুলো অক্ষত থাকলে সাধারণত firmware-এর বুট মেনু বা লাইভ USB দিয়ে আবার entry বানানো যায়। অনেক firmware entry না পেলে fallback পাথ `\EFI\BOOT\BOOTX64.EFI` চেষ্টা করে।

লিনাক্স থেকে সরাসরি পরের বুটে UEFI সেটআপ স্ক্রিনে যেতে চাইলে (কোন key চাপতে হয় খুঁজতে না হয়):

```
$ sudo systemctl reboot --firmware-setup
```

---

## ২৭.৩ ধাপ ২: ডিস্কের পার্টিশন টেবিল — MBR বনাম GPT

Firmware ডিস্কে পৌঁছানোর পর প্রথমে দেখে ডিস্কটা কীভাবে ভাগ করা আছে। এই ভাগের তালিকাকে বলে **partition table** (পার্টিশন টেবিল)। এর দুটি ফরম্যাট: MBR আর GPT। পার্টিশন তৈরি আর ফরম্যাটিংয়ের পুরো আলোচনা আছে [অধ্যায় ৩২](32-disk-partition-mount.md)-এ। এখানে শুধু বুটের সাথে যেটুকু সম্পর্কিত, সেটা দেখব।

### MBR (Master Boot Record)

ডিস্কের একদম প্রথম 512 byte (sector 0):

```
  MBR — 512 byte
 ┌───────────────────────────────────────┬────────────────┬──────┐
 │      Bootstrap code (446 byte)        │ Partition table│ 0x55 │
 │  (GRUB-এর boot.img এখানে থাকে)          │  4 × 16 = 64 B │ 0xAA │
 └───────────────────────────────────────┴────────────────┴──────┘
   byte 0                             445  446          509  510-511
                                                          (signature)
```

- **446 byte**: bootstrap code। BIOS এটাই চালায়।
- **64 byte**: পার্টিশন টেবিল, সর্বোচ্চ ৪টি **primary partition** (প্রতিটি 16 byte)। বেশি লাগলে একটাকে **extended partition** বানিয়ে তার ভেতরে **logical partition** তৈরি করতে হয়।
- **2 byte**: `0x55AA` signature। এটা না থাকলে BIOS ডিস্কটাকে বুটযোগ্য মনে করে না।

সীমাবদ্ধতা: পার্টিশনের শুরু আর আকার 32-bit সংখ্যায় রাখা হয়, তাই 512-byte সেক্টরে সর্বোচ্চ 2 TiB পর্যন্ত ব্যবহার করা যায়। পার্টিশন টেবিলের কোনো ব্যাকআপ কপিও নেই।

### GPT (GUID Partition Table)

UEFI স্ট্যান্ডার্ডের অংশ, আধুনিক ডিস্কের ডিফল্ট:

```
  ডিস্কের শুরু                                           ডিস্কের শেষ
 ┌────────┬────────┬──────────┬───────────────────┬──────────┬────────┐
 │Protect.│ GPT    │Partition │   পার্টিশনগুলো     │Partition │ Backup │
 │  MBR   │ Header │ Entries  │ (ESP, root, home…)│ Entries  │ GPT    │
 │ LBA 0  │ LBA 1  │ (128 টা) │                   │ (ব্যাকআপ) │ Header │
 └────────┴────────┴──────────┴───────────────────┴──────────┴────────┘
```

- **Protective MBR**: পুরনো টুল যেন ডিস্কটাকে "খালি" ভেবে মুছে না ফেলে, তাই LBA 0-তে একটা নকল MBR রাখা থাকে, যেটা বলে "পুরো ডিস্ক একটাই পার্টিশন, টাইপ GPT"।
- ডিফল্টে **128টি পার্টিশন** রাখা যায়, extended/logical-এর ঝামেলা নেই।
- প্রতিটি পার্টিশনের একটা অনন্য **GUID** (PARTUUID) আর একটা টাইপ GUID থাকে।
- ডিস্কের শেষে **ব্যাকআপ header** থাকে, আর CRC32 checksum দিয়ে নষ্ট হয়েছে কি না যাচাই করা যায়।
- সীমা প্রায় 8 ZiB, বাস্তবে অসীম।

| বৈশিষ্ট্য | MBR | GPT |
|---|---|---|
| সর্বোচ্চ ডিস্ক (512-byte সেক্টর) | 2 TiB | ~8 ZiB |
| পার্টিশন সংখ্যা | ৪ primary (extended দিয়ে বেশি) | ডিফল্ট 128 |
| ব্যাকআপ | নেই | ডিস্কের শেষে আছে |
| অখণ্ডতা যাচাই | নেই | CRC32 |
| পার্টিশন পরিচয় | 1-byte টাইপ কোড (যেমন `83`) | GUID |
| সাধারণত কার সাথে | BIOS | UEFI |

তোমার ডিস্কে কোন টেবিল:

```
$ sudo fdisk -l /dev/nvme0n1 | grep -i "disklabel type"
Disklabel type: gpt
```

অথবা:

```
$ lsblk -o NAME,SIZE,PTTYPE,FSTYPE,MOUNTPOINTS
NAME          SIZE PTTYPE FSTYPE MOUNTPOINTS
nvme0n1     476.9G gpt
├─nvme0n1p1     1G gpt    vfat   /boot/efi
├─nvme0n1p2     2G gpt    ext4   /boot
└─nvme0n1p3 473.9G gpt    ext4   /
```

> 📝 **মনে রাখো:** BIOS + GPT একসাথেও চলে। সে ক্ষেত্রে GRUB-এর core.img রাখার জন্য ছোট্ট (~1 MiB) একটা **BIOS boot partition** লাগে, যার কোনো ফাইলসিস্টেম থাকে না (`parted`-এ flag `bios_grub`)। আবার UEFI + MBR-ও সম্ভব, তবে কম দেখা যায়।

### ESP — EFI System Partition

UEFI বুটের হৃদয় হলো **ESP**। এটা একটা সাধারণ **FAT32** পার্টিশন, যার GPT টাইপ GUID `C12A7328-F81F-11D2-BA4B-00A0C93EC93B` (`fdisk`-এ টাইপ "EFI System")। UEFI firmware এই পার্টিশন পড়ে `.efi` ফাইল চালায়।

- আকার: সাধারণত 100 MiB থেকে 1 GiB। Ubuntu-র নতুন ইনস্টলার 1 GiB-এর মতো দেয়, আর ছোট পুরনো ESP-তে জায়গা ফুরিয়ে যাওয়ার সমস্যা এড়াতে বড় রাখাই নিরাপদ।
- লিনাক্সে মাউন্ট পয়েন্ট: Ubuntu, Debian, Fedora, RHEL-এ **`/boot/efi`**। কিছু সেটআপে (systemd-boot, Arch) `/efi` বা `/boot`।
- একই ডিস্কে Windows আর লিনাক্স থাকলে **একটাই ESP** দুজনে ভাগ করে ব্যবহার করে, প্রত্যেকে নিজের ফোল্ডারে।

```
$ sudo tree /boot/efi -L 3
/boot/efi
└── EFI
    ├── BOOT
    │   ├── BOOTX64.EFI        ← fallback (entry না থাকলে firmware এটা চালায়)
    │   ├── fbx64.efi
    │   └── mmx64.efi
    ├── Microsoft
    │   └── Boot
    └── ubuntu
        ├── grub.cfg           ← ছোট্ট stub, আসল grub.cfg-কে দেখিয়ে দেয়
        ├── grubx64.efi        ← GRUB নিজে
        ├── mmx64.efi          ← MOK Manager
        └── shimx64.efi        ← Secure Boot-এর প্রথম ধাপ
```

Fedora/RHEL-এ ফোল্ডারের নাম `EFI/fedora` বা `EFI/redhat`।

> 💡 **টিপস:** ESP-তে কিছু মুছবে না, যদি না ঠিক জানো কী করছ। Windows dual-boot-এ `EFI/Microsoft` মুছলে Windows আর বুট হবে না।

---

## ২৭.৪ ধাপ ৩: Bootloader — GRUB2

### Bootloader কী আর কেন লাগে?

**Bootloader** হলো সেই প্রোগ্রাম যার কাজ OS-এর kernel খুঁজে মেমরিতে লোড করা আর চালানো। Firmware নিজে লিনাক্সের kernel চেনে না, ext4 বা Btrfs ফাইলসিস্টেমও পড়তে পারে না। Bootloader এই ফাঁকটা পূরণ করে:

- কোন OS বা কোন kernel ভার্সন বুট হবে তার মেনু দেখায়
- ext4, XFS, Btrfs, LVM, এমনকি এনক্রিপ্টেড `/boot` থেকেও kernel পড়তে পারে
- kernel-কে **parameter** (কমান্ড লাইন) দেয়
- initramfs ফাইলটাও মেমরিতে লোড করে দেয়

লিনাক্সের প্রধান bootloader হলো **GRUB2** (GRand Unified Bootloader, version 2)। Ubuntu, Debian, Fedora, RHEL সবাই এটা ব্যবহার করে। বিকল্প হিসেবে **systemd-boot** (সহজ, শুধু UEFI) আছে, Arch আর Pop!_OS-এ দেখা যায়। পুরনো **LILO** আর **GRUB Legacy** এখন অচল।

> 📝 **মনে রাখো:** Debian/Ubuntu-তে কমান্ডগুলোর নাম `grub-*` (যেমন `grub-install`, `grub-mkconfig`), আর Fedora/RHEL-এ `grub2-*` (যেমন `grub2-install`, `grub2-mkconfig`)। কাজ একই, শুধু নাম আলাদা।

### GRUB-এর ফাইলগুলো কোথায়?

| ফাইল/ডিরেক্টরি | কাজ | হাতে এডিট করবে? |
|---|---|---|
| `/etc/default/grub` | মূল সেটিংস (timeout, kernel parameter ইত্যাদি) | ✅ হ্যাঁ, এটাই এডিট করার জায়গা |
| `/etc/grub.d/` | কিছু শেল স্ক্রিপ্ট, যারা মিলে `grub.cfg` বানায় | মাঝে মাঝে (কাস্টম entry-র জন্য `40_custom`) |
| `/boot/grub/grub.cfg` (Debian/Ubuntu) | GRUB আসলে যেটা পড়ে | ❌ না, স্বয়ংক্রিয়ভাবে তৈরি |
| `/boot/grub2/grub.cfg` (Fedora/RHEL) | একই | ❌ না |
| `/boot/loader/entries/*.conf` (Fedora/RHEL) | প্রতিটি kernel-এর আলাদা entry (BLS) | `grubby` দিয়ে বদলাও |
| `/boot/grub/` বা `/boot/grub2/` | GRUB-এর মডিউল, ফন্ট, থিম | ❌ না |

এর পেছনের নকশাটা এরকম:

```
 /etc/default/grub  ──┐      (তোমার সেটিংস)
                      ├──►  grub-mkconfig / update-grub  ──►  /boot/grub/grub.cfg
 /etc/grub.d/*      ──┘      (স্ক্রিপ্টগুলো চালায়)                (GRUB বুটের সময় পড়ে)
   00_header
   10_linux        ← /boot-এ যত kernel পায়, প্রত্যেকটার entry লেখে
   20_linux_xen
   30_os-prober    ← Windows বা অন্য OS খুঁজে entry লেখে
   40_custom       ← তোমার নিজের entry
```

`grub.cfg`-এর মাথাতেই লেখা থাকে `DO NOT EDIT THIS FILE`। কারণ পরের বার kernel আপডেট হলে এটা নতুন করে তৈরি হবে, আর তোমার হাতের পরিবর্তন মুছে যাবে।

### `/etc/default/grub` — গুরুত্বপূর্ণ সেটিংস

একটা সাধারণ Ubuntu-র ফাইল:

```
$ cat /etc/default/grub
GRUB_DEFAULT=0
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=0
GRUB_DISTRIBUTOR=`( . /etc/os-release; echo ${NAME:-Ubuntu} ) 2>/dev/null || echo Ubuntu`
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX=""
```

| ভেরিয়েবল | মানে | উদাহরণ |
|---|---|---|
| `GRUB_DEFAULT` | কোন entry ডিফল্টে বুট হবে। সংখ্যা (০ থেকে শুরু), entry-র নাম, বা `saved` | `GRUB_DEFAULT=0` |
| `GRUB_SAVEDEFAULT` | `true` আর `GRUB_DEFAULT=saved` হলে শেষবার যেটা বেছেছিলে সেটাই মনে রাখে | `GRUB_SAVEDEFAULT=true` |
| `GRUB_TIMEOUT` | মেনু কত সেকেন্ড অপেক্ষা করবে। `-1` মানে অনির্দিষ্টকাল | `GRUB_TIMEOUT=5` |
| `GRUB_TIMEOUT_STYLE` | `menu` = মেনু দেখাও, `hidden` = লুকিয়ে রাখো, `countdown` = শুধু কাউন্টডাউন | `GRUB_TIMEOUT_STYLE=menu` |
| `GRUB_CMDLINE_LINUX` | **সব** entry-তে (recovery-সহ) যোগ হওয়া kernel parameter | `GRUB_CMDLINE_LINUX="ipv6.disable=1"` |
| `GRUB_CMDLINE_LINUX_DEFAULT` | শুধু সাধারণ entry-তে যোগ হয়, recovery-তে না (Debian/Ubuntu) | `"quiet splash"` |
| `GRUB_DISABLE_OS_PROBER` | `false` হলে Windows/অন্য OS খুঁজবে | `GRUB_DISABLE_OS_PROBER=false` |
| `GRUB_DISABLE_RECOVERY` | `true` হলে recovery entry তৈরি হবে না | |
| `GRUB_GFXMODE` | মেনুর রেজোলিউশন | `GRUB_GFXMODE=1920x1080` |
| `GRUB_ENABLE_BLSCFG` | Fedora/RHEL-এ BLS entry ব্যবহার হবে কি না | `true` |

> 📝 **মনে রাখো:** GRUB 2.06 থেকে `os-prober` ডিফল্টে বন্ধ (নিরাপত্তার কারণে)। তাই নতুন Ubuntu/Debian-এ Windows মেনুতে না এলে `GRUB_DISABLE_OS_PROBER=false` যোগ করে `update-grub` চালাও (`os-prober` প্যাকেজ ইনস্টল থাকতে হবে)।

### পরিবর্তন প্রয়োগ করা: `update-grub` ও `grub2-mkconfig`

`/etc/default/grub` এডিট করলে নিজে থেকে কিছু হয় না। `grub.cfg` নতুন করে তৈরি করতে হয়।

**Debian / Ubuntu:**

```
$ sudo nano /etc/default/grub
$ sudo update-grub
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-45-generic
Found initrd image: /boot/initrd.img-6.8.0-45-generic
Found linux image: /boot/vmlinuz-6.8.0-41-generic
Found initrd image: /boot/initrd.img-6.8.0-41-generic
Found Windows Boot Manager on /dev/nvme0n1p1@/EFI/Microsoft/Boot/bootmgfw.efi
Adding boot menu entry for UEFI Firmware Settings ...
done
```

`update-grub` আসলে একটা ছোট wrapper, যেটা ভেতরে চালায়:

```
$ sudo grub-mkconfig -o /boot/grub/grub.cfg
```

**Fedora / RHEL 9:**

```
$ sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

> ⚠️ **সতর্কতা (RHEL/Fedora + UEFI):** পুরনো গাইডে লেখা থাকে `grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg`। Fedora 34 আর RHEL 9 থেকে এটা **আর করবে না**। এখন ESP-তে থাকা `grub.cfg` একটা ছোট stub, যেটা `/boot/grub2/grub.cfg`-কে লোড করে। তুমি stub-টা ওভাররাইট করলে বুট ভেঙে যেতে পারে। সবসময় `-o /boot/grub2/grub.cfg` ব্যবহার করো।

### Fedora/RHEL-এর BLS আর `grubby`

Fedora আর RHEL 8/9 **BLS** (Boot Loader Specification) ব্যবহার করে। এখানে প্রতিটি kernel-এর entry `grub.cfg`-এর ভেতরে না থেকে `/boot/loader/entries/`-এ আলাদা ফাইলে থাকে:

```
$ sudo ls /boot/loader/entries/
a1b2c3d4...-5.14.0-427.13.1.el9_4.x86_64.conf
a1b2c3d4...-0-rescue.conf

$ sudo cat /boot/loader/entries/a1b2c3d4...-5.14.0-427.13.1.el9_4.x86_64.conf
title Red Hat Enterprise Linux (5.14.0-427.13.1.el9_4.x86_64) 9.4 (Plow)
version 5.14.0-427.13.1.el9_4.x86_64
linux /vmlinuz-5.14.0-427.13.1.el9_4.x86_64
initrd /initramfs-5.14.0-427.13.1.el9_4.x86_64.img $tuned_initrd
options root=/dev/mapper/rhel-root ro crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet
grub_users $grub_users
grub_arg --unrestricted
grub_class rhel
```

এগুলো সামলানোর জন্য টুল হলো **`grubby`**:

| কমান্ড | কাজ |
|---|---|
| `sudo grubby --default-kernel` | ডিফল্ট kernel কোনটা দেখাও |
| `sudo grubby --info=ALL` | সব entry-র বিস্তারিত |
| `sudo grubby --set-default=/boot/vmlinuz-<ver>` | ডিফল্ট kernel বদলাও |
| `sudo grubby --update-kernel=ALL --args="nomodeset"` | সব kernel-এ parameter যোগ করো |
| `sudo grubby --update-kernel=ALL --remove-args="quiet"` | parameter সরাও |
| `sudo grubby --update-kernel=DEFAULT --args="..."` | শুধু ডিফল্ট kernel-এ |

```
$ sudo grubby --default-kernel
/boot/vmlinuz-5.14.0-427.13.1.el9_4.x86_64
```

> 💡 **টিপস:** RHEL 9-এ নতুন kernel ইনস্টলের সময়ও parameter বজায় রাখতে চাইলে `grubby --update-kernel=ALL` এর পাশাপাশি `/etc/default/grub`-এর `GRUB_CMDLINE_LINUX`-এও যোগ করে রাখো।

### GRUB পুনরায় ইনস্টল: `grub-install`

`grub-mkconfig` শুধু কনফিগ ফাইল বানায়। আর `grub-install` GRUB-এর বাইনারিগুলো বুট লোকেশনে (MBR বা ESP) বসায়। সাধারণত এটা হাতে চালানোর দরকার হয় না। লাগে যখন Windows ইনস্টল করে GRUB মুছে গেছে, বা ডিস্ক বদলেছ।

```
# BIOS: ডিস্কে বসাও (পার্টিশনে না!)
$ sudo grub-install /dev/sda

# UEFI (Debian/Ubuntu):
$ sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu
```

> ⚠️ **সতর্কতা:** RHEL 9 / Fedora-তে UEFI + Secure Boot সিস্টেমে `grub2-install` চালানো উচিত না। সেখানে সাইন করা GRUB প্যাকেজ থেকে আসে, তাই মেরামতের জন্য `sudo dnf reinstall shim-x64 grub2-efi-x64` চালানো হয়।

### GRUB মেনু ব্যবহার করা

Ubuntu-তে একটাই OS থাকলে মেনু লুকানো থাকে। দেখতে চাইলে বুটের সময়:

- **BIOS মেশিনে:** `Shift` চেপে ধরে রাখো
- **UEFI মেশিনে:** `Esc` বারবার চাপো (বেশিবার চাপলে `grub>` কমান্ড লাইনে চলে যেতে পারে, তখন `normal` লিখে Enter দিলে মেনুতে ফিরবে)

মেনুতে:

```
                      GNU GRUB  version 2.12

 ┌────────────────────────────────────────────────────────────────┐
 │*Ubuntu                                                          │
 │ Advanced options for Ubuntu                                     │
 │ Windows Boot Manager (on /dev/nvme0n1p1)                        │
 │ UEFI Firmware Settings                                          │
 └────────────────────────────────────────────────────────────────┘
   Use the ↑ and ↓ keys to select which entry is highlighted.
   Press enter to boot the selected OS, `e' to edit the commands
   before booting or `c' for a command-line.
```

"Advanced options"-এর ভেতরে পুরনো kernel আর **recovery mode** entry থাকে। নতুন kernel-এ সমস্যা হলে এখান থেকে পুরনো kernel বেছে বুট করা যায়। এজন্যই প্যাকেজ ম্যানেজার আগের ১-২টি kernel রেখে দেয়।

### একটা GRUB entry-র ভেতরে কী থাকে

মেনুতে কোনো entry-তে দাঁড়িয়ে `e` চাপলে এরকম দেখাবে:

```
setparams 'Ubuntu'

        recordfail
        load_video
        gfxmode $linux_gfx_mode
        insmod gzio
        insmod part_gpt
        insmod ext2
        search --no-floppy --fs-uuid --set=root 7e1c...-...
        linux   /boot/vmlinuz-6.8.0-45-generic root=UUID=7e1c...-... ro  quiet splash $vt_handoff
        initrd  /boot/initrd.img-6.8.0-45-generic
```

| লাইন | মানে |
|---|---|
| `insmod part_gpt`, `insmod ext2` | GPT আর ext2/3/4 পড়ার GRUB মডিউল লোড করো |
| `search ... --set=root UUID` | এই UUID-এর পার্টিশন খুঁজে GRUB-এর `root` বানাও |
| `linux /boot/vmlinuz-...` | এই kernel লোড করো, বাকিটা kernel parameter |
| `initrd /boot/initrd.img-...` | এই initramfs লোড করো |

এখানে পরিবর্তন করে `Ctrl+x` বা `F10` চাপলে **শুধু এই একবারের জন্য** সেভাবে বুট হবে। পরের রিবুটে আগের মতো হয়ে যাবে। সমস্যা সমাধানে এটা খুব কাজের।

### GRUB কমান্ড লাইন: হাতে বুট করা

মেনুতে `c` চাপলে, বা `grub.cfg` না পেলে, `grub>` প্রম্পট আসে। এখান থেকে হাতে বুট করা যায়:

```
grub> ls
(hd0) (hd0,gpt1) (hd0,gpt2) (hd0,gpt3)

grub> ls (hd0,gpt2)/
lost+found/ boot/ etc/ home/ ... vmlinuz initrd.img

grub> set root=(hd0,gpt2)
grub> linux /boot/vmlinuz-6.8.0-45-generic root=/dev/nvme0n1p2 ro
grub> initrd /boot/initrd.img-6.8.0-45-generic
grub> boot
```

GRUB-এ ডিস্ক `hd0` থেকে আর পার্টিশন ১ থেকে গোনা হয়। `Tab` চাপলে নাম auto-complete হয়।

যদি **`grub rescue>`** আসে, তার মানে GRUB নিজের মডিউলই খুঁজে পাচ্ছে না, তাই সেখানে কমান্ড খুব সীমিত। তখন সঠিক পার্টিশন খুঁজে normal মডিউল লোড করতে হয়:

```
grub rescue> ls
grub rescue> set prefix=(hd0,gpt2)/boot/grub
grub rescue> set root=(hd0,gpt2)
grub rescue> insmod normal
grub rescue> normal
```

তারপর বুট হলে `sudo grub-install ...` আর `sudo update-grub` চালিয়ে স্থায়ীভাবে ঠিক করো।

### GRUB পাসওয়ার্ড — কেন?

একটু পরেই দেখবে, GRUB মেনুতে `e` চেপে যে কেউ root পাসওয়ার্ড রিসেট করতে পারে। সার্ভার রুম বা শেয়ার করা কম্পিউটারে এটা আটকাতে GRUB-এ পাসওয়ার্ড দেওয়া যায়:

- **RHEL/Fedora:** `sudo grub2-setpassword` (এটা `/boot/grub2/user.cfg` তৈরি করে)
- **Debian/Ubuntu:** `grub-mkpasswd-pbkdf2` দিয়ে হ্যাশ বানিয়ে `/etc/grub.d/40_custom`-এ `set superusers` ও `password_pbkdf2` লাইন যোগ করে `update-grub`

---

## ২৭.৫ Kernel Parameters — kernel-কে নির্দেশ দেওয়া

GRUB-এর `linux` লাইনে kernel ফাইলের পরে যা লেখা থাকে, সেগুলোই **kernel parameter** বা **kernel command line**। এগুলো kernel আর initramfs-কে বলে দেয় কোথায় root আছে, কতটা লগ দেখাবে, কোন ফিচার বন্ধ রাখবে ইত্যাদি। বর্তমান বুটে কী parameter ব্যবহার হয়েছে:

```
$ cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-6.8.0-45-generic root=UUID=7e1c...-... ro quiet splash vt.handoff=7
```

### সাধারণ ও কাজের parameter

| Parameter | কাজ |
|---|---|
| `root=UUID=...` / `root=/dev/sda2` | root filesystem কোথায় |
| `ro` / `rw` | root প্রথমে read-only না read-write মাউন্ট হবে (সাধারণত `ro`, পরে systemd `rw` করে) |
| `quiet` | kernel-এর বেশিরভাগ বার্তা স্ক্রিনে দেখাবে না |
| `splash` | Plymouth-এর গ্রাফিক্যাল বুট স্ক্রিন দেখাও (Ubuntu); Fedora/RHEL-এ এর সমতুল্য `rhgb` |
| `loglevel=7` | কত বিস্তারিত kernel বার্তা কনসোলে দেখাবে (০–৭) |
| `nomodeset` | kernel-এর গ্রাফিক্স ড্রাইভার mode-setting বন্ধ — নতুন NVIDIA কার্ডে কালো স্ক্রিন হলে প্রাথমিক সমাধান |
| `systemd.unit=rescue.target` | rescue mode-এ বুট |
| `systemd.unit=emergency.target` | emergency mode-এ বুট |
| `single`, `s`, `S`, `1` | পুরনো single-user mode; systemd এগুলোকে `rescue.target` হিসেবে নেয় |
| `3` / `5` | systemd-এ যথাক্রমে `multi-user.target` / `graphical.target` |
| `init=/bin/bash` | systemd-এর বদলে সরাসরি bash চালাও (PID 1 = bash) |
| `rd.break` | (dracut, Fedora/RHEL) আসল root-এ যাওয়ার ঠিক আগে initramfs-এর শেলে থামো |
| `rd.lvm.lv=vg/lv` | (dracut) initramfs-এ এই LV চালু করো |
| `resume=UUID=...` | hibernate থেকে ফেরার swap পার্টিশন |
| `console=ttyS0,115200` | সিরিয়াল কনসোলে আউটপুট পাঠাও (ক্লাউড VM-এ সাধারণ) |
| `mem=4G` | kernel-কে ৪ GB-এর বেশি RAM ব্যবহার করতে দিও না |
| `selinux=0` / `enforcing=0` | SELinux বন্ধ / permissive (শুধু ট্রাবলশুটিংয়ে) |
| `systemd.log_level=debug` | systemd-এর বিস্তারিত লগ |
| `fsck.mode=force` | বুটে ফাইলসিস্টেম চেক বাধ্যতামূলক করো |

### সাময়িকভাবে বদলানো (একবারের জন্য)

1. GRUB মেনু খোলো (`Shift`/`Esc`)
2. entry-তে দাঁড়িয়ে `e`
3. `linux` লাইনের শেষে parameter যোগ করো, যেমন `nomodeset`
4. `Ctrl+x` বা `F10` চাপো

### স্থায়ীভাবে বদলানো

**Debian/Ubuntu:**

```
$ sudo nano /etc/default/grub
# GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"
$ sudo update-grub
$ sudo reboot
$ cat /proc/cmdline       # যাচাই করো
```

**Fedora/RHEL:**

```
$ sudo grubby --update-kernel=ALL --args="nomodeset"
$ sudo reboot
```

> 💡 **টিপস:** বুট সমস্যা খুঁজতে `quiet` আর `splash` (বা `rhgb`) সরিয়ে বুট করো। তাহলে স্ক্রিনে সব বার্তা দেখা যাবে, কোথায় আটকাচ্ছে বোঝা যাবে।

> 📝 **মনে রাখো:** kernel যে parameter নিজে চেনে না, সেগুলো সে PID 1 (systemd)-কে দিয়ে দেয়। এভাবেই `systemd.unit=...`-এর মতো parameter systemd পর্যন্ত পৌঁছায়। `rd.*` parameter-গুলো initramfs (dracut) পড়ে।

---

## ২৭.৬ ধাপ ৪: Kernel লোড হওয়া

### `/boot`-এ কী কী থাকে

```
$ ls -lh /boot
-rw-r--r-- 1 root root 287K Sep 20 10:12 config-6.8.0-45-generic
drwx------ 3 root root 4.0K Jan  1  1970 efi
drwxr-xr-x 5 root root 4.0K Oct  2 09:30 grub
lrwxrwxrwx 1 root root   28 Oct  2 09:29 initrd.img -> initrd.img-6.8.0-45-generic
-rw-r--r-- 1 root root  68M Oct  2 09:30 initrd.img-6.8.0-45-generic
-rw------- 1 root root 8.9M Sep 20 10:12 System.map-6.8.0-45-generic
lrwxrwxrwx 1 root root   25 Oct  2 09:29 vmlinuz -> vmlinuz-6.8.0-45-generic
-rw------- 1 root root  15M Sep 20 10:12 vmlinuz-6.8.0-45-generic
```

| ফাইল | কী |
|---|---|
| `vmlinuz-<ver>` | কম্প্রেস করা kernel ইমেজ। নামের `z` মানে zipped (কম্প্রেসড) |
| `initrd.img-<ver>` (Debian/Ubuntu) / `initramfs-<ver>.img` (Fedora/RHEL) | initramfs আর্কাইভ |
| `config-<ver>` | kernel কোন অপশন নিয়ে কম্পাইল হয়েছে |
| `System.map-<ver>` | kernel-এর ফাংশন/ভেরিয়েবলের ঠিকানার তালিকা (ডিবাগিংয়ে লাগে) |
| `grub/` বা `grub2/` | GRUB-এর ফাইল |
| `efi/` | ESP-র মাউন্ট পয়েন্ট |

চলমান kernel-এর ভার্সন:

```
$ uname -r
6.8.0-45-generic
```

### Kernel চালু হলে ভেতরে কী হয়

GRUB kernel আর initramfs দুটোকেই RAM-এ বসিয়ে kernel-এর শুরুর ঠিকানায় লাফ দেয়। এরপর kernel:

```
 ┌───────────────────────────────────────────────────────────────┐
 │ 1. নিজেকে decompress করে                                       │
 │ 2. CPU-কে পূর্ণ 64-bit মোডে নেয়, page table / virtual memory   │
 │    চালু করে                                                    │
 │ 3. মেমরি ম্যানেজার, scheduler, interrupt চালু করে              │
 │ 4. kernel-এর ভেতরে বিল্ট-ইন (built-in) ড্রাইভারগুলো শুরু করে,   │
 │    হার্ডওয়্যার খোঁজে (PCI, USB, ACPI)                          │
 │ 5. initramfs আর্কাইভটা খুলে RAM-এ একটা ছোট ফাইলসিস্টেম বানায়  │
 │    (rootfs) এবং সেখানকার /init চালায়  ← PID 1 (অস্থায়ী)        │
 └───────────────────────────────────────────────────────────────┘
```

এই সময় kernel যা যা বার্তা লেখে, সব জমা হয় **kernel ring buffer**-এ, যা পরে `dmesg` দিয়ে পড়া যায় (২৭.১০ অংশে)।

kernel আর হার্ডওয়্যারের সম্পর্ক কীভাবে কাজ করে তার বিস্তারিত আছে [অধ্যায় ২](02-linux-architecture.md) আর [অধ্যায় ৪৭](47-kernel-proc-sys.md)-এ।

> ⚠️ **"Kernel panic"** মানে kernel এমন সমস্যায় পড়েছে যেখান থেকে সে আর এগোতে পারছে না। বুটের সময় সবচেয়ে প্রচলিত বার্তা: `Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)`। এর মানে প্রায়ই initramfs নেই বা নষ্ট, অথবা `root=` ভুল। সমাধান: GRUB-এর "Advanced options" থেকে পুরনো kernel দিয়ে বুট করে initramfs নতুন করে বানাও।

---

## ২৭.৭ ধাপ ৫: initramfs — ছোট্ট উদ্ধারকারী ফাইলসিস্টেম

### সমস্যাটা কী? মুরগি আগে না ডিম আগে

kernel-কে root filesystem মাউন্ট করতে হবে। কিন্তু:

- root যদি থাকে NVMe ডিস্কে, তাহলে NVMe ড্রাইভার লাগবে
- root যদি থাকে **LVM**-এ, তাহলে LVM টুল (`lvm`) চালাতে হবে
- root যদি **LUKS** দিয়ে এনক্রিপ্টেড হয়, তাহলে পাসওয়ার্ড চেয়ে ডিস্ক খুলতে হবে
- root যদি **RAID** বা নেটওয়ার্কে (NFS/iSCSI) থাকে, তাহলে আরও টুল লাগবে

ডিস্ট্রিবিউশনের সাধারণ kernel হাজার রকম হার্ডওয়্যারে চলে, তাই সব ড্রাইভার kernel-এর ভেতরে না রেখে আলাদা **module** (`.ko` ফাইল) হিসেবে রাখে, যেগুলো থাকে `/lib/modules/<ver>/`-এ। কিন্তু `/lib/modules` তো root filesystem-এর ভেতরে! root মাউন্ট করতে ড্রাইভার লাগে, আর ড্রাইভার আছে root-এর ভেতরে। এই চক্র ভাঙতেই **initramfs**।

> 💡 **উপমা:** তুমি বাসার চাবি বাসার ভেতরে রেখে বের হয়ে গেছ। initramfs হলো পকেটে রাখা একটা ছোট ব্যাগ, যাতে বাসার দরজা খোলার জন্য দরকারি চাবি আর যন্ত্রপাতি আছে। দরজা খোলা হয়ে গেলে ব্যাগটার আর দরকার নেই।

### initramfs আসলে কী?

**initramfs** (initial RAM filesystem) একটা কম্প্রেস করা **cpio আর্কাইভ**। এর ভেতরে থাকে একটা মিনি লিনাক্স:

- `/init` স্ক্রিপ্ট বা প্রোগ্রাম (Fedora/RHEL-এ dracut-এর ভেতরে systemd নিজেই)
- দরকারি kernel module (স্টোরেজ, ফাইলসিস্টেম, এনক্রিপশন)
- ছোট টুল: `udev`, `lvm`, `cryptsetup`, `mdadm`, BusyBox (Debian/Ubuntu)
- `/etc/fstab`-এর root লাইন, কখনো কীবোর্ড লেআউট

> 📝 **মনে রাখো:** পুরনো দিনে এর নাম ছিল **initrd** (initial RAM disk), যেটা আসলে একটা ডিস্ক ইমেজ ছিল। এখন ভেতরে cpio আর্কাইভ, অর্থাৎ initramfs, কিন্তু Debian/Ubuntu ফাইলের নাম এখনো `initrd.img-*` রেখেছে।

### initramfs-এর কাজের ধারা

```
 kernel
   │ initramfs খুলে RAM-এ বসায়, /init চালায়
   ▼
 ┌──────────────────── initramfs (RAM-এ) ─────────────────────┐
 │  udev চালু → হার্ডওয়্যার খোঁজে → দরকারি module লোড        │
 │  (দরকার হলে) RAID জোড়া লাগায় → LUKS পাসওয়ার্ড চায়        │
 │            → LVM volume চালু করে                           │
 │  root=UUID=... অনুযায়ী আসল root মাউন্ট করে                 │
 │     /sysroot (dracut) বা /root (initramfs-tools)-এ          │
 └────────────────────────────┬───────────────────────────────┘
                              │ switch_root
                              ▼
       আসল root filesystem হলো নতুন "/"
       initramfs-এর মেমরি মুক্ত হয়
       /sbin/init (= systemd) চালু হয়, PID 1 হিসেবে
```

`switch_root` নতুন root-কে `/` বানায়, পুরনো initramfs মুছে মেমরি খালি করে, আর `/sbin/init` চালায়। লক্ষ করো, নতুন প্রোগ্রামও PID 1 নম্বরটাই পায়, কারণ এটা `exec` করে চালানো হয়।

### কে initramfs বানায়: `initramfs-tools` বনাম `dracut`

| ডিস্ট্রো | টুল | কমান্ড | ফাইল |
|---|---|---|---|
| Debian 12, Ubuntu 24.04 | initramfs-tools | `update-initramfs` | `/boot/initrd.img-<ver>` |
| Fedora, RHEL 8/9, Rocky, Alma | dracut | `dracut` | `/boot/initramfs-<ver>.img` |

> 📝 **মনে রাখো:** Ubuntu 25.10 থেকে ডিফল্ট টুল dracut-এ বদলেছে। Ubuntu 24.04 LTS-এ এখনো initramfs-tools।

সাধারণত তোমাকে এগুলো হাতে চালাতে হয় না। kernel আপডেটের সময় প্যাকেজ ম্যানেজার নিজেই নতুন initramfs বানায়। হাতে লাগে যখন:

- `/etc/crypttab`, `/etc/fstab`-এর root, বা RAID কনফিগ বদলেছ
- নতুন স্টোরেজ ড্রাইভার লাগবে (যেমন VM অন্য হাইপারভাইজারে সরিয়েছ)
- কোনো module ব্ল্যাকলিস্ট করেছ (যেমন `nouveau`)
- initramfs নষ্ট হয়েছে বলে kernel panic হচ্ছে

### `update-initramfs` (Debian/Ubuntu)

```
$ sudo update-initramfs -u
update-initramfs: Generating /boot/initrd.img-6.8.0-45-generic
```

| অপশন | কাজ |
|---|---|
| `-u` | বিদ্যমান initramfs আপডেট (ডিফল্টে সর্বশেষ kernel-এর) |
| `-u -k all` | সব ইনস্টল করা kernel-এর জন্য আপডেট |
| `-c -k 6.8.0-45-generic` | নির্দিষ্ট kernel-এর জন্য নতুন তৈরি |
| `-d -k <ver>` | মুছে ফেলো |
| `-v` | বিস্তারিত আউটপুট |

কনফিগ থাকে `/etc/initramfs-tools/`-এ। যেমন `/etc/initramfs-tools/modules` ফাইলে কোনো module-এর নাম লিখলে সেটা initramfs-এ যোগ হবে।

ভেতরে কী আছে দেখতে:

```
$ lsinitramfs /boot/initrd.img-$(uname -r) | grep -E 'nvme|ext4' | head
usr/lib/modules/6.8.0-45-generic/kernel/drivers/nvme/host/nvme.ko.zst
usr/lib/modules/6.8.0-45-generic/kernel/drivers/nvme/host/nvme-core.ko.zst
usr/lib/modules/6.8.0-45-generic/kernel/fs/ext4/ext4.ko.zst
```

### `dracut` (Fedora/RHEL)

```
$ sudo dracut -f
```

| অপশন/কমান্ড | কাজ |
|---|---|
| `dracut -f` | চলমান kernel-এর initramfs ওভাররাইট করে নতুন বানাও |
| `dracut -f --kver 5.14.0-427.13.1.el9_4.x86_64` | নির্দিষ্ট kernel-এর জন্য |
| `dracut -f --regenerate-all` | সব kernel-এর জন্য |
| `dracut --add-drivers "virtio_blk" -f` | নির্দিষ্ট ড্রাইভার যোগ করো |
| `dracut -H -f` / `--hostonly` | শুধু এই মেশিনের দরকারি জিনিস রাখো (RHEL-এ ডিফল্ট) |
| `dracut -N -f` / `--no-hostonly` | সাধারণ (generic), যেকোনো হার্ডওয়্যারে চলবে, আকারে বড় |
| `lsinitrd` | initramfs-এর বিষয়বস্তু দেখো |
| `lsinitrd -m` | কোন dracut module আছে দেখো |

কনফিগ: `/etc/dracut.conf` আর `/etc/dracut.conf.d/*.conf`।

```
$ sudo lsinitrd | head -5
Image: /boot/initramfs-5.14.0-427.13.1.el9_4.x86_64.img: 38M
========================================================================
Early CPIO image
========================================================================
drwxr-xr-x   3 root     root            0 Oct  2 09:30 .
```

> ⚠️ **সতর্কতা:** initramfs বানানোর আগে চাইলে পুরনোটার কপি রেখে দাও (`sudo cp /boot/initrd.img-$(uname -r){,.bak}`)। নতুনটা ভুল হলে GRUB-এ `e` চেপে `initrd` লাইনে `.bak` যোগ করে বুট করা যাবে।

### initramfs-এর শেলে আটকে গেলে

initramfs root খুঁজে না পেলে একটা জরুরি শেলে থামে:

- **Debian/Ubuntu:** BusyBox শেল, প্রম্পট `(initramfs)`। সাধারণ বার্তা: `ALERT! UUID=... does not exist. Dropping to a shell!`
- **Fedora/RHEL:** dracut emergency shell, প্রম্পট `dracut:/#`, আর `/run/initramfs/rdsosreport.txt`-এ রিপোর্ট লেখা থাকে।

এখানে `ls /dev/disk/by-uuid/`, `blkid`, `cat /proc/cmdline` দিয়ে দেখো ডিস্ক দেখা যাচ্ছে কি না, UUID মিলছে কি না। ফাইলসিস্টেম নষ্ট হলে BusyBox-এ `fsck -y /dev/sda2` চালিয়ে `exit` দিলে বুট চালু হতে পারে।

---

## ২৭.৮ ধাপ ৬: systemd — PID 1, সব প্রসেসের জনক

আসল root-এ পৌঁছে চালু হয় `/sbin/init`। আধুনিক সব প্রধান ডিস্ট্রোতে এটা systemd-এর একটা symlink:

```
$ ls -l /sbin/init
lrwxrwxrwx 1 root root 20 Aug  9 12:00 /sbin/init -> /lib/systemd/systemd

$ ps -p 1 -o pid,comm
    PID COMMAND
      1 systemd
```

### PID 1 কেন বিশেষ?

- এটাই kernel-এর চালানো প্রথম user-space প্রসেস। বাকি সব প্রসেস এর বংশধর (`pstree -p` চালিয়ে দেখো)।
- কোনো প্রসেসের প্যারেন্ট মরে গেলে সেই "অনাথ" প্রসেস PID 1-এর কাছে যায়, আর PID 1 পরে তার zombie পরিষ্কার করে।
- PID 1 মরে গেলে kernel panic হয়। তাই systemd খুব যত্ন করে লেখা।
- kernel PID 1-কে এমন সিগন্যাল থেকে রক্ষা করে যার হ্যান্ডলার সে বসায়নি, তাই `kill -9 1` কাজ করে না।

systemd, unit ফাইল আর `systemctl`-এর পুরো আলোচনা আছে [অধ্যায় ২৬](26-systemd-services.md)-এ। এখানে শুধু বুটের সাথে সম্পর্কিত অংশ।

### systemd বুটে কী করে

systemd একটা **target** পর্যন্ত পৌঁছানোর চেষ্টা করে। target হলো অনেকগুলো unit-এর দল, একটা "অবস্থা" বা মাইলফলক। ডিফল্ট লক্ষ্য হলো `default.target`, যেটা সাধারণত `graphical.target` (ডেস্কটপ) বা `multi-user.target` (সার্ভার)-এর symlink।

systemd পুরো dependency গ্রাফ (`Wants=`, `Requires=`, `After=`) হিসাব করে, তারপর যেসব unit একে অন্যের ওপর নির্ভর করে না সেগুলো **একসাথে সমান্তরালে** চালায়। পুরনো SysV init একটার পর একটা স্ক্রিপ্ট চালাত, তাই systemd-এ বুট দ্রুত।

```
                   local-fs-pre.target
                          │
   (udev, journald,       ▼
    fsck, mount)     local-fs.target   swap.target   cryptsetup.target
         │                │                │              │
         └────────────────┴───────┬────────┴──────────────┘
                                  ▼
                           sysinit.target
                                  │
              ┌───────────────────┼─────────────────────┐
              ▼                   ▼                     ▼
        sockets.target      timers.target         paths.target
              └───────────────────┼─────────────────────┘
                                  ▼
                            basic.target
                                  │
                ┌─────────────────┴──────────────────┐
                ▼                                    ▼
     (sshd, cron, nginx, NetworkManager…)        getty.target
                └─────────────────┬──────────────────┘
                                  ▼
                          multi-user.target
                                  │
                                  ▼ display-manager.service
                           graphical.target
```

এই চিত্রটা `man bootup`-এর ডায়াগ্রামের সরল রূপ। পুরোটা দেখতে `man 7 bootup` পড়ো।

### Targets আর পুরনো Runlevel

SysV init-এর যুগে সিস্টেমের অবস্থা বোঝাতে **runlevel** (০–৬) ব্যবহার হতো। systemd সামঞ্জস্যের জন্য `runlevelN.target` নামে alias রেখেছে:

| Runlevel | systemd target | মানে |
|---|---|---|
| 0 | `poweroff.target` | বন্ধ |
| 1, s, single | `rescue.target` | single-user, রক্ষণাবেক্ষণ |
| 2, 3, 4 | `multi-user.target` | মাল্টি-ইউজার, নেটওয়ার্ক, কিন্তু GUI নেই |
| 5 | `graphical.target` | মাল্টি-ইউজার + GUI |
| 6 | `reboot.target` | রিবুট |
| — | `emergency.target` | একদম ন্যূনতম শেল |

```
$ systemctl get-default
graphical.target

$ sudo systemctl set-default multi-user.target     # পরের বুট থেকে GUI ছাড়া
Removed "/etc/systemd/system/default.target".
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/multi-user.target.

$ sudo systemctl isolate multi-user.target          # এখনই GUI বন্ধ করে টেক্সট মোডে
$ sudo systemctl isolate graphical.target           # আবার GUI

$ runlevel                                          # পুরনো কমান্ড, এখনো কাজ করে
N 5
```

`runlevel`-এর আউটপুটে প্রথমটা আগের runlevel (`N` মানে নেই), দ্বিতীয়টা বর্তমান।

> ⚠️ **সতর্কতা:** `isolate` বর্তমান target-এ না থাকা সব unit বন্ধ করে দেয়। ডেস্কটপ থেকে `isolate multi-user.target` চালালে তোমার GUI সেশন, খোলা ফাইলসহ, সাথে সাথে বন্ধ হয়ে যাবে।

### বুটে কত সময় লাগল: `systemd-analyze`

```
$ systemd-analyze
Startup finished in 6.512s (firmware) + 2.104s (loader) + 1.876s (kernel) + 3.951s (initrd) + 8.230s (userspace) = 22.674s
graphical.target reached after 8.201s in userspace.
```

এই এক লাইনেই পুরো বুট প্রসেসের ধাপগুলো দেখা যাচ্ছে: firmware → loader (GRUB) → kernel → initrd → userspace (systemd)। (firmware ও loader-এর সময় শুধু তখনই দেখায় যখন মেশিন UEFI আর bootloader সেই সময়ের তথ্য systemd-কে জানায়, যেমন systemd-boot। অনেক GRUB সেটআপে এই দুটো অংশ থাকে না, শুধু kernel + initrd + userspace দেখায়।)

| কমান্ড | কাজ |
|---|---|
| `systemd-analyze` | মোট বুট সময়, ধাপভিত্তিক |
| `systemd-analyze blame` | কোন unit কত সময় নিল, বেশি থেকে কম |
| `systemd-analyze critical-chain` | যে ধারাবাহিক শিকলটা বুটকে সবচেয়ে বেশি আটকে রেখেছে |
| `systemd-analyze critical-chain nginx.service` | নির্দিষ্ট unit পর্যন্ত শিকল |
| `systemd-analyze plot > boot.svg` | পুরো বুটের টাইমলাইন ছবি (ব্রাউজারে খোলো) |
| `systemd-analyze verify x.service` | unit ফাইলে ভুল আছে কি না |

```
$ systemd-analyze blame | head -5
5.102s NetworkManager-wait-online.service
1.401s snapd.seeded.service
  812ms dev-nvme0n1p3.device
  640ms systemd-journal-flush.service
  402ms udisks2.service
```

```
$ systemd-analyze critical-chain
graphical.target @8.201s
└─multi-user.target @8.199s
  └─snapd.seeded.service @6.790s +1.401s
    └─basic.target @2.110s
      └─sockets.target @2.109s
...
```

`@` এর পরের সময় হলো unit কখন সক্রিয় হলো, `+` এর পরেরটা নিজে চালু হতে কত সময় নিল।

> 💡 **টিপস:** `blame`-এ শীর্ষে থাকা unit সবসময় বুট ধীর করে না, কারণ অনেক unit সমান্তরালে চলে। আসল বাধা খুঁজতে `critical-chain` দেখো। ডেস্কটপে `NetworkManager-wait-online.service` প্রায়ই অপ্রয়োজনীয়। কোনো service-এর সত্যিই নেটওয়ার্ক-অপেক্ষার দরকার না থাকলে সেটা `disable` করা যায়।

---

## ২৭.৯ ধাপ ৭: লগইন — getty ও display manager

### টেক্সট লগইন: getty → login → shell

লিনাক্সে কয়েকটি **virtual console** (TTY) থাকে: `tty1`, `tty2`... `Ctrl+Alt+F1`...`F6` দিয়ে এগুলোর মধ্যে যাওয়া যায়। প্রতিটি কনসোলে লগইন প্রম্পট দেখায় **getty** (get teletype) প্রোগ্রাম, আধুনিক লিনাক্সে এর নাম **agetty**।

systemd এগুলো চালায় একটা টেমপ্লেট unit `getty@.service` দিয়ে: `getty@tty1.service`, `getty@tty2.service` ইত্যাদি। মজার ব্যাপার, systemd সব কনসোলে আগে থেকে getty চালায় না। তুমি `Ctrl+Alt+F3` চাপলে তখনই `getty@tty3.service` চালু হয় (`systemd-logind`-এর autovt ব্যবস্থা)। সিরিয়াল কনসোলের জন্য আছে `serial-getty@ttyS0.service`।

```
  agetty                      login                       bash
 ┌──────────────────┐   ┌────────────────────────┐   ┌──────────────┐
 │ টার্মিনাল সেটআপ   │   │ PAM দিয়ে পাসওয়ার্ড      │   │ ~/.bash_     │
 │ "hostname login:"│──►│ যাচাই (/etc/shadow)     │──►│ profile পড়ে │
 │ ইউজারনেম নেয়     │   │ UID/GID সেট, HOME-এ যায় │   │ প্রম্পট দেয়  │
 │ exec login       │   │ /etc/passwd-এর শেল exec │   │ $            │
 └──────────────────┘   └────────────────────────┘   └──────────────┘
```

প্রতিটি ধাপ `exec` করে পরের প্রোগ্রামে বদলে যায়, তাই PID একই থাকে। তুমি shell থেকে `exit` করলে প্রসেস শেষ হয়, systemd সেটা দেখে আবার নতুন getty চালায় (unit-এ `Restart=always`)। এজন্যই লগআউটের পর আবার লগইন প্রম্পট আসে।

```
$ systemctl status getty@tty1.service
● getty@tty1.service - Getty on tty1
     Loaded: loaded (/usr/lib/systemd/system/getty@.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-11-09 09:12:03 +06; 2h ago
   Main PID: 1032 (agetty)
     CGroup: /system.slice/system-getty.slice/getty@tty1.service
             └─1032 /sbin/agetty -o "-p -- \\u" --noclear - linux
```

ইউজার, `/etc/passwd`, `/etc/shadow`-এর বিস্তারিত পাবে [অধ্যায় ১৫](15-users-groups.md)-এ। লগইনের পর কোন কোন কনফিগ ফাইল পড়া হয় তা [অধ্যায় ৪৪](44-environment-bashrc.md)-এ।

### গ্রাফিক্যাল লগইন: display manager

`graphical.target`-এ পৌঁছালে চালু হয় **display manager** (DM), যেটা গ্রাফিক্যাল লগইন স্ক্রিন দেখায়:

| Display manager | সাধারণত কোথায় |
|---|---|
| GDM (GNOME Display Manager) | Ubuntu, Fedora Workstation, RHEL, Debian GNOME |
| SDDM | KDE Plasma (Kubuntu, Fedora KDE) |
| LightDM | Xfce, Linux Mint, Xubuntu |

যেটাই ব্যবহার হোক, systemd তাকে `display-manager.service` নামে চেনে (একটা alias symlink):

```
$ ls -l /etc/systemd/system/display-manager.service
lrwxrwxrwx 1 root root 32 Jan 10  2026 /etc/systemd/system/display-manager.service -> /usr/lib/systemd/system/gdm.service

$ systemctl status display-manager
```

DM পাসওয়ার্ড যাচাই করে (PAM দিয়ে), `systemd-logind`-এর কাছে একটা session নিবন্ধন করে, তারপর তোমার ডেস্কটপ (GNOME/KDE) চালায় Wayland বা X11-এ। বিস্তারিত [অধ্যায় ৪](04-desktop-environment.md)-এ।

> 📝 **মনে রাখো:** Ubuntu-তে GDM-এর লগইন স্ক্রিন সাধারণত `tty1`-এ থাকে আর লগইনের পর তোমার ডেস্কটপ সেশন `tty2`-তে চলে (Fedora-তেও একই রকম)। তাই GUI জমে গেলে `Ctrl+Alt+F3` চেপে টেক্সট কনসোলে লগইন করে সমস্যা দেখতে পারো, তারপর `Ctrl+Alt+F2` (বা F1) চেপে GUI-তে ফেরো।

---

## ২৭.১০ বুট লগ পড়া: `dmesg` ও `journalctl -b`

### `dmesg` — kernel-এর বার্তা

`dmesg` (display message) kernel ring buffer পড়ে। বুটের সময় হার্ডওয়্যার শনাক্ত, ড্রাইভার লোড, ডিস্ক এরর, USB লাগানো-খোলা সবই এখানে আসে।

```
$ sudo dmesg | head -5
[    0.000000] Linux version 6.8.0-45-generic (buildd@lcy02-amd64-075) (x86_64-linux-gnu-gcc-13 ...) #45-Ubuntu SMP PREEMPT_DYNAMIC ...
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-6.8.0-45-generic root=UUID=7e1c... ro quiet splash vt.handoff=7
[    0.000000] KERNEL supported cpus:
[    0.000000]   Intel GenuineIntel
[    0.000000]   AMD AuthenticAMD
```

বাঁদিকের `[ 0.000000]` হলো kernel চালু হওয়ার পর থেকে কত সেকেন্ড গেছে।

> 📝 **মনে রাখো:** Ubuntu ও Debian-এ `kernel.dmesg_restrict=1`, তাই সাধারণ ইউজার `dmesg` চালালে `dmesg: read kernel buffer failed: Operation not permitted` আসে। `sudo` লাগবে।

| অপশন | কাজ |
|---|---|
| `-T` / `--ctime` | সময় মানুষের পড়ার মতো তারিখ-সময়ে দেখাও (suspend/resume হলে একটু ভুল হতে পারে) |
| `-H` / `--human` | রঙিন, pager-সহ, পড়তে সহজ |
| `-w` / `--follow` | নতুন বার্তা আসলে সাথে সাথে দেখাও (USB লাগিয়ে দেখো) |
| `-l err,warn` / `--level` | শুধু নির্দিষ্ট লেভেল (emerg, alert, crit, err, warn, notice, info, debug) |
| `-x` / `--decode` | facility ও level লেখায় দেখাও |
| `-k` / `--kernel` | শুধু kernel-এর বার্তা |
| `-c` / `--read-clear` | পড়ে বাফার খালি করো |
| `-C` / `--clear` | বাফার খালি করো |

```
$ sudo dmesg -T -l err,warn
[Mon Nov  9 09:11:58 2026] ACPI BIOS Error (bug): Could not resolve symbol [\_SB.PC00.LPCB.HEC.ECAV], AE_NOT_FOUND
[Mon Nov  9 09:12:00 2026] nvme nvme0: missing or invalid SUBNQN field.

$ sudo dmesg -w
[ 9310.551204] usb 3-2: new high-speed USB device number 5 using xhci_hcd
[ 9310.703118] usb-storage 3-2:1.0: USB Mass Storage device detected
[ 9311.731902] sd 0:0:0:0: [sdb] 60620800 512-byte logical blocks: (31.0 GB/28.9 GiB)
[ 9311.745320]  sdb: sdb1
```

কাজের কিছু খোঁজ:

```
$ sudo dmesg | grep -iE 'error|fail|firmware'   # সমস্যা
$ sudo dmesg | grep -i 'secure boot'            # Secure Boot চালু কি না
$ sudo dmesg | grep -i oom                       # মেমরি শেষ হয়ে প্রসেস মারা হয়েছে কি না
```

> ⚠️ **সতর্কতা:** ring buffer-এর আকার সীমিত (সাধারণত কয়েকশো KB থেকে কয়েক MB)। অনেকদিন চলা সিস্টেমে বুটের প্রথম দিকের বার্তা মুছে যেতে পারে। আর রিবুট করলে `dmesg`-এর পুরনো বার্তা চলে যায়। আগের বুটের kernel বার্তার জন্য `journalctl` লাগবে।

### `journalctl -b` — পুরো বুটের লগ

systemd-এর journal kernel-এর বার্তা আর সব service-এর বার্তা একসাথে রাখে। বিস্তারিত [অধ্যায় ২৮](28-logs.md)-এ। বুটের জন্য দরকারি অংশ:

| কমান্ড | কাজ |
|---|---|
| `journalctl -b` | বর্তমান বুটের সব লগ |
| `journalctl -b -1` | আগের বুটের লগ (`-2` মানে তার আগের) |
| `journalctl --list-boots` | সব সংরক্ষিত বুটের তালিকা |
| `journalctl -k` / `--dmesg` | বর্তমান বুটের শুধু kernel বার্তা (dmesg-এর মতো) |
| `journalctl -k -b -1` | আগের বুটের kernel বার্তা |
| `journalctl -b -p err` | বর্তমান বুটের শুধু error ও তার চেয়ে গুরুতর |
| `journalctl -b -u ssh.service` | নির্দিষ্ট service-এর বুট লগ |

```
$ journalctl --list-boots
IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY
 -2 3c5e1a7f0b5d4c4f8a9d2e0c1b7a6f55 Sat 2026-11-07 08:01:12 +06 Sat 2026-11-07 23:40:02 +06
 -1 9a0d2e4c7b3f4a1e9d5c8b2a7f6e1d34 Sun 2026-11-08 08:30:44 +06 Sun 2026-11-08 22:15:10 +06
  0 f1e2d3c4b5a6478899aabbccddeeff00 Mon 2026-11-09 09:11:56 +06 Mon 2026-11-09 11:20:31 +06
```

> 💡 **টিপস:** "গতকাল মেশিন হঠাৎ বন্ধ হয়ে গেল কেন?" জানতে `journalctl -b -1 -e` চালাও (`-e` মানে শেষে চলে যাও)। আগের বুটের শেষ বার্তাগুলোতে কারণের সূত্র থাকে। এজন্য journal **persistent** হতে হবে, মানে `/var/log/journal/` ডিরেক্টরি থাকতে হবে। Ubuntu, Debian 12, Fedora, RHEL 9-এ ডিফল্টে এটা থাকে। না থাকলে `sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald` চালাও।

---

## ২৭.১১ Rescue mode ও Emergency mode

কখনো সিস্টেম স্বাভাবিকভাবে বুট হয় না, বা কোনো ভুল কনফিগ ঠিক করতে ন্যূনতম পরিবেশ লাগে। এর জন্য systemd-এর দুটি বিশেষ target আছে।

| | `rescue.target` | `emergency.target` |
|---|---|---|
| পুরনো নাম | single-user mode, runlevel 1 | — |
| কী কী মাউন্ট হয় | সব লোকাল ফাইলসিস্টেম (`/etc/fstab` থেকে) | শুধু root, তাও **read-only** |
| কোন service চলে | `sysinit.target` পর্যন্ত, মৌলিক service | প্রায় কিছুই না |
| নেটওয়ার্ক | না | না |
| কখন ব্যবহার | service বা কনফিগের সমস্যা, পাসওয়ার্ড বদল | `/etc/fstab` ভুল, ফাইলসিস্টেম নষ্ট, rescue-ও আটকে গেলে |
| kernel parameter | `systemd.unit=rescue.target` (বা `single`, `1`) | `systemd.unit=emergency.target` (বা `emergency`) |
| চলমান সিস্টেম থেকে | `sudo systemctl rescue` | `sudo systemctl emergency` |

> 💡 **উপমা:** rescue mode হলো দোকানের শাটার অর্ধেক তুলে ভেতরে মেরামত করা: আলো-পাখা চলছে, কিন্তু খদ্দের ঢুকতে পারছে না। emergency mode হলো বিদ্যুৎ ছাড়া টর্চ হাতে ঢোকা: শুধু সবচেয়ে জরুরি কাজটুকু।

দুই মোডেই systemd `sulogin` চালায়, যেটা **root-এর পাসওয়ার্ড** চায়:

```
You are in emergency mode. After logging in, type "journalctl -xb" to view
system logs, "systemctl reboot" to reboot, "systemctl default" or "exit"
to boot into default mode.
Give root password for maintenance
(or press Control-D to continue):
```

এখানে:

```
# journalctl -xb                 # কী ভুল হয়েছে দেখো
# mount -o remount,rw /          # (emergency-তে) root লেখার যোগ্য করো
# nano /etc/fstab                # ভুল লাইন ঠিক করো বা # দিয়ে কমেন্ট করো
# systemctl daemon-reload
# systemctl default              # স্বাভাবিক বুট চালিয়ে যাও (বা exit)
```

> ⚠️ **সতর্কতা (Ubuntu):** Ubuntu-তে ডিফল্টে root অ্যাকাউন্ট **লক** থাকে (পাসওয়ার্ড নেই, `sudo` ব্যবহার হয়)। তখন `sulogin` বলে `Cannot open access to console, the root account is locked.` আর Enter চাপলে আবার চেষ্টা করে। এ অবস্থায় rescue/emergency কাজে আসে না। Ubuntu-র GRUB-এর "Advanced options → (recovery mode)" entry এর বদলে একটা মেনু দেয় (`root — Drop to root shell prompt` সহ), যেখানে পাসওয়ার্ড লাগে না। সেটাও না চললে নিচের `init=/bin/bash` পদ্ধতি ব্যবহার করো।

> 📝 **মনে রাখো:** `/etc/fstab`-এ ভুল UUID থাকা emergency mode-এর সবচেয়ে প্রচলিত কারণ। বাইরের/USB ডিস্কের লাইনে `nofail` অপশন দিলে ডিস্ক না থাকলেও বুট আটকাবে না। fstab-এর বিস্তারিত [অধ্যায় ৩২](32-disk-partition-mount.md)-এ।

### Ubuntu-র recovery mode মেনু

GRUB → Advanced options for Ubuntu → `... (recovery mode)` বেছে নিলে:

```
 Recovery Menu (filesystem state: read-only)
 resume      Resume normal boot
 clean       Try to make free space
 dpkg        Repair broken packages
 fsck        Check all file systems
 grub        Update grub bootloader
 network     Enable networking
 root        Drop to root shell prompt
 system-summary   System summary
```

`root` বেছে নিলে root শেল পাবে। ফাইলসিস্টেম read-only থাকে, তাই কিছু বদলাতে প্রথমে `mount -o remount,rw /` চালাতে হয়।

---

## ২৭.১২ ভুলে যাওয়া root (বা ইউজার) পাসওয়ার্ড রিসেট

শুরুতেই একটা সত্য জেনে রাখো: **যার কাছে মেশিনের শারীরিক অ্যাক্সেস (physical access) আছে, সে-ই আসলে root**, যদি না ডিস্ক এনক্রিপ্টেড থাকে বা GRUB/firmware-এ পাসওয়ার্ড থাকে। এই পদ্ধতিগুলো সেই সত্যেরই প্রমাণ। শুধু নিজের বা অনুমতিপ্রাপ্ত মেশিনে ব্যবহার করবে।

> 📝 **মনে রাখো:** ডিস্ক LUKS দিয়ে এনক্রিপ্টেড হলে এই পদ্ধতিতেও আগে ডিস্কের passphrase দিতে হবে। passphrase ছাড়া ডেটায় পৌঁছানো যাবে না, আর এটাই এনক্রিপশনের উদ্দেশ্য।

### পদ্ধতি ক: `init=/bin/bash` (Ubuntu, Debian, প্রায় সব ডিস্ট্রো)

ধারণাটা হলো: kernel-কে বলব systemd-এর বদলে সরাসরি bash চালাও। তখন কোনো লগইন নেই, পাসওয়ার্ড যাচাই নেই, আর bash চলবে root হিসেবে, কারণ PID 1 সবসময় root।

1. রিবুট করো, GRUB মেনু আনো (`Shift` বা `Esc`)।
2. সাধারণ entry-তে দাঁড়িয়ে `e` চাপো।
3. `linux` দিয়ে শুরু হওয়া লাইনটা খুঁজে নাও। সেখানে `ro quiet splash $vt_handoff` অংশটা মুছে লেখো `rw init=/bin/bash`:

   ```
   linux /boot/vmlinuz-6.8.0-45-generic root=UUID=7e1c... rw init=/bin/bash
   ```

4. `Ctrl+x` বা `F10` চাপো। কয়েক সেকেন্ডে root প্রম্পট আসবে:

   ```
   root@(none):/#
   ```

5. root লেখার যোগ্য কি না নিশ্চিত হও (`rw` দিলে সাধারণত হয়ে যায়):

   ```
   # mount -o remount,rw /
   ```

6. পাসওয়ার্ড বদলাও:

   ```
   # passwd root            # root-এর জন্য
   # passwd nishat          # বা নির্দিষ্ট ইউজারের জন্য (Ubuntu-তে এটাই বেশি দরকারি)
   New password:
   Retype new password:
   passwd: password updated successfully
   ```

   ইউজারনেম মনে না থাকলে: `ls /home` বা `grep -E ':[0-9]{4}:' /etc/passwd`।

7. ডিস্কে লেখা নিশ্চিত করে রিবুট করো:

   ```
   # sync
   # exec /sbin/init
   ```

   `exec /sbin/init` bash-কে systemd দিয়ে প্রতিস্থাপন করে স্বাভাবিক বুট চালু করে। এখানে সাধারণ `reboot` কমান্ড কাজ নাও করতে পারে, কারণ systemd চলছে না। চাইলে `sync` এর পর `reboot -f` দিয়েও জোর করে রিবুট করা যায়।

> ⚠️ **সতর্কতা:** SELinux চালু থাকা সিস্টেমে (Fedora/RHEL) এই পদ্ধতিতে `/etc/shadow` ফাইলের SELinux লেবেল ভুল হয়ে যায়, ফলে পরে কেউ লগইন করতে পারবে না। তাই সেখানে পদ্ধতি খ ব্যবহার করো, বা পদ্ধতি ক-এর পর `touch /.autorelabel` করো।

### পদ্ধতি খ: `rd.break` (RHEL 9, Fedora, Rocky, Alma)

dracut-এর `rd.break` initramfs-এর ভেতরে, আসল root-এ যাওয়ার ঠিক আগে একটা শেল দেয়। তখন আসল root `/sysroot`-এ read-only মাউন্ট থাকে।

1. রিবুট করো, GRUB মেনুতে entry-তে দাঁড়িয়ে `e` চাপো।
2. `linux` লাইনের শেষে যোগ করো `rd.break`। (ক্লাউড/VM-এ সিরিয়াল কনসোল থাকলে `console=` parameter-গুলো সরাতে হতে পারে, যাতে শেল স্ক্রিনেই আসে।)
3. `Ctrl+x` চাপো। প্রম্পট আসবে:

   ```
   switch_root:/#
   ```

4. নিচের ধাপগুলো ক্রমানুসারে:

   ```
   switch_root:/# mount -o remount,rw /sysroot
   switch_root:/# chroot /sysroot
   sh-5.1# passwd root
   Changing password for user root.
   New password:
   Retype new password:
   passwd: all authentication tokens updated successfully.
   sh-5.1# touch /.autorelabel
   sh-5.1# exit
   switch_root:/# exit
   ```

| ধাপ | কেন |
|---|---|
| `mount -o remount,rw /sysroot` | আসল root এখন read-only, লিখতে হলে rw করতে হবে |
| `chroot /sysroot` | আসল সিস্টেমকে "/" বানিয়ে তার `passwd` আর `/etc/shadow` ব্যবহার করা |
| `passwd root` | পাসওয়ার্ড বদল |
| `touch /.autorelabel` | পরের বুটে SELinux পুরো ফাইলসিস্টেমের লেবেল ঠিক করবে, `/etc/shadow`-সহ |
| `exit` দুবার | প্রথমটা chroot থেকে, দ্বিতীয়টা dracut শেল থেকে, বুট চালিয়ে যায় |

> 💡 **টিপস:** `/.autorelabel`-এর কারণে পরের বুটে কয়েক মিনিট (বড় ডিস্কে আরও বেশি) সময় লাগবে আর মেশিন নিজে থেকে একবার রিবুট হবে। ভয় পেয়ো না, এটা স্বাভাবিক। দ্রুত বিকল্প: chroot-এর ভেতরে `touch` এর বদলে `load_policy -i` চালিয়ে তারপর `passwd` করো আর শেষে `restorecon -v /etc/shadow` চালাও।

> 📝 **মনে রাখো:** RHEL 9-এর অফিশিয়াল ডকুমেন্টেশনে এটাই "Resetting the root password" পদ্ধতি। RHCSA পরীক্ষার একটা পরিচিত কাজ এটা। RHEL-এর ইনস্টল মিডিয়ার "Troubleshooting → Rescue a Red Hat Enterprise Linux system" দিয়েও একই কাজ করা যায় (`chroot /mnt/sysroot`)।

### পদ্ধতি গ: লাইভ USB দিয়ে

GRUB-ও যদি পাসওয়ার্ড-সুরক্ষিত বা নষ্ট হয়:

```
# লাইভ USB থেকে বুট করে টার্মিনালে:
$ lsblk                                  # root পার্টিশন খোঁজো, ধরো /dev/nvme0n1p3
$ sudo mount /dev/nvme0n1p3 /mnt
$ sudo chroot /mnt passwd nishat
$ sudo umount /mnt
$ reboot
```

(LVM হলে আগে `sudo vgchange -ay`, তারপর `/dev/mapper/...` মাউন্ট করো।)

### এই "ফাঁক" কীভাবে বন্ধ করবে?

| সুরক্ষা | কী আটকায় |
|---|---|
| GRUB পাসওয়ার্ড (`grub2-setpassword` / `password_pbkdf2`) | মেনুতে `e` বা `c` চেপে এডিট |
| UEFI/BIOS সেটআপ পাসওয়ার্ড + USB বুট বন্ধ | লাইভ USB দিয়ে বুট |
| পূর্ণ ডিস্ক এনক্রিপশন (LUKS) | ডিস্ক খুলে অন্য মেশিনে পড়া, chroot |
| Secure Boot | স্বাক্ষরবিহীন bootloader/kernel চালানো |
| শারীরিক নিরাপত্তা (তালাবদ্ধ সার্ভার রুম) | সবকিছু |

আরও নিরাপত্তা-ব্যবস্থা পাবে [অধ্যায় ৪৫](45-security-hardening.md)-এ।

---

## ২৭.১৩ Secure Boot — শুধু বিশ্বস্ত কোড চলবে

### সমস্যাটা কী

বুটের একদম শুরুতে, OS চালু হওয়ার আগেই, যদি কোনো ম্যালওয়্যার bootloader বা kernel বদলে দেয় (একে বলে **bootkit** বা **rootkit**), তাহলে OS-এর অ্যান্টিভাইরাস কিছুই ধরতে পারবে না, কারণ ম্যালওয়্যার তার আগেই চালু হয়ে গেছে।

**Secure Boot** UEFI-র একটা ফিচার, যেটা প্রতিটি ধাপে পরের প্রোগ্রামের **ডিজিটাল স্বাক্ষর** (digital signature) যাচাই করে। স্বাক্ষর বিশ্বস্ত চাবির সাথে না মিললে চালাতে দেয় না। এভাবে তৈরি হয় একটা **chain of trust** (বিশ্বাসের শিকল)।

> 💡 **উপমা:** বিয়েবাড়ির গেটে দারোয়ান শুধু কার্ড দেখে ঢুকতে দেয়। কার্ডে বরের বাবার সই থাকতে হবে। ভেতরে ঢোকা অতিথি আবার নিজের সই করা পাস দিয়ে ড্রাইভারকে ঢোকাতে পারে। প্রত্যেকে পরের জনের জন্য দায়ী।

### লিনাক্সে Secure Boot-এর শিকল

UEFI firmware-এ ডিফল্টে প্রায় সবসময় **Microsoft**-এর চাবি থাকে। লিনাক্স ডিস্ট্রোগুলো প্রতিবার নিজের GRUB Microsoft-কে দিয়ে সই করানোর বদলে একটা ছোট মধ্যস্থ প্রোগ্রাম **shim** একবার সই করিয়ে নেয়। shim-এর ভেতরে ডিস্ট্রোর নিজের চাবি থাকে।

```
 ┌──────────────────────────────┐
 │ UEFI firmware                │  db-তে আছে Microsoft UEFI CA-র চাবি
 └──────────────┬───────────────┘
                │ যাচাই: Microsoft-এর সই আছে?
                ▼
 ┌──────────────────────────────┐
 │ shimx64.efi                  │  ভেতরে ডিস্ট্রোর চাবি (Canonical / Red Hat)
 │                              │  + MOK তালিকা (তোমার নিজের চাবি)
 └──────────────┬───────────────┘
                │ যাচাই: ডিস্ট্রো বা MOK-এর সই আছে?
                ▼
 ┌──────────────────────────────┐
 │ grubx64.efi                  │
 └──────────────┬───────────────┘
                │ যাচাই (shim-এর সাহায্যে): kernel সই করা?
                ▼
 ┌──────────────────────────────┐
 │ vmlinuz (signed)             │  lockdown মোডে চলে
 └──────────────┬───────────────┘
                │ যাচাই: প্রতিটি kernel module সই করা?
                ▼
        nvidia.ko, virtualbox ইত্যাদি → সই না থাকলে লোড হবে না
```

### Secure Boot চালু আছে কি না দেখা

```
$ mokutil --sb-state
SecureBoot enabled

$ sudo dmesg | grep -i "secure boot"
[    0.000000] secureboot: Secure boot enabled
[    0.006542] Kernel is locked down from EFI Secure Boot mode; see man kernel_lockdown.7
```

### MOK — Machine Owner Key

ধরো তুমি NVIDIA-র প্রোপ্রাইটারি ড্রাইভার বা VirtualBox ইনস্টল করলে। এগুলোর kernel module তোমার মেশিনেই কম্পাইল হয় (**DKMS** দিয়ে), তাই ডিস্ট্রোর সই থাকে না। Secure Boot সেগুলো লোড হতে দেবে না:

```
$ sudo modprobe vboxdrv
modprobe: ERROR: could not insert 'vboxdrv': Key was rejected by service
```

সমাধান হলো নিজের একটা চাবি বানিয়ে module সই করা, আর সেই চাবিকে বিশ্বস্ত বলে নিবন্ধন করা। এই নিজের চাবিকেই বলে **MOK** (Machine Owner Key)।

Ubuntu-তে DKMS স্বয়ংক্রিয়ভাবে চাবি বানায় (`/var/lib/shim-signed/mok/MOK.der`) আর ড্রাইভার ইনস্টলের সময় একটা পাসওয়ার্ড চায়। হাতে করতে চাইলে:

```
$ sudo mokutil --import /var/lib/shim-signed/mok/MOK.der
input password:
input password again:
$ sudo reboot
```

রিবুটে একটা নীল স্ক্রিন **MOK Manager** (`mmx64.efi`) আসবে: `Enroll MOK` → `Continue` → `Yes` → একটু আগের পাসওয়ার্ড দাও → `Reboot`। এরপর চাবিটা বিশ্বস্ত হবে।

| কমান্ড | কাজ |
|---|---|
| `mokutil --sb-state` | Secure Boot চালু কি না |
| `mokutil --list-enrolled` | নিবন্ধিত MOK চাবির তালিকা |
| `mokutil --import key.der` | নতুন চাবি নিবন্ধনের অনুরোধ (পরের বুটে নিশ্চিত করতে হয়) |
| `mokutil --delete key.der` | চাবি সরানোর অনুরোধ |
| `mokutil --disable-validation` | shim-এর যাচাই বন্ধ (প্রস্তাবিত না) |

> ⚠️ **সতর্কতা:** MOK Manager-এর নীল স্ক্রিনে সময়সীমা থাকে (প্রায় ১০ সেকেন্ড)। কোনো key না চাপলে সেটা চলে যাবে আর চাবি নিবন্ধন হবে না। তখন আবার `mokutil --import` চালাতে হবে।

> 💡 **টিপস:** সমস্যা হলে অনেকে Secure Boot বন্ধ করে দেয়। এটা কাজ করে, কিন্তু নিরাপত্তা কমে। MOK নিবন্ধনই সঠিক পথ। Secure Boot চালু থাকলে kernel **lockdown** মোডে থাকে, যেখানে কিছু নিম্নস্তরের কাজ (যেমন `/dev/mem`-এ লেখা, স্বাক্ষরবিহীন kernel-এ `kexec`) নিষিদ্ধ।

---

## ২৭.১৪ বিপরীত দিক: Shutdown ও Reboot

বুটের উল্টো প্রক্রিয়াও systemd সামলায়। `systemctl poweroff` বা `reboot` চালালে systemd `shutdown.target`-এর দিকে যায়: সব service-কে `SIGTERM` পাঠায় (না থামলে নির্দিষ্ট সময় পর, ডিফল্ট ৯০ সেকেন্ড, `SIGKILL`), ফাইলসিস্টেম আনমাউন্ট করে, শেষে kernel-কে বন্ধ বা রিবুট করতে বলে।

| কমান্ড | কাজ |
|---|---|
| `sudo systemctl reboot` বা `sudo reboot` | রিবুট |
| `sudo systemctl poweroff` বা `sudo poweroff` | বন্ধ |
| `sudo shutdown -h +10 "রক্ষণাবেক্ষণ"` | ১০ মিনিট পর বন্ধ, সবাইকে বার্তা |
| `sudo shutdown -r now` | এখনই রিবুট |
| `sudo shutdown -c` | নির্ধারিত shutdown বাতিল |
| `sudo systemctl reboot --firmware-setup` | রিবুট করে UEFI সেটআপে যাও |
| `sudo systemctl kexec` | firmware/GRUB এড়িয়ে সরাসরি নতুন kernel-এ রিবুট (kexec-tools লাগে) |

> 💡 **টিপস:** shutdown "A stop job is running for ..." বার্তায় অনেকক্ষণ আটকে থাকলে বুঝবে কোনো service থামতে দেরি করছে। পরের বুটে `journalctl -b -1 -e` দেখে কোনটা দেখে নাও।

---

## ২৭.১৫ বুট-সংক্রান্ত সব কীবোর্ড শর্টকাট

### Firmware পর্যায় (নির্মাতা-ভেদে আলাদা)

| কী | কাজ |
|---|---|
| `Del` / `F2` | UEFI/BIOS সেটআপে ঢোকা (বেশিরভাগ ডেস্কটপ মাদারবোর্ড ও অনেক ল্যাপটপ) |
| `F12` / `F11` / `F8` / `Esc` | এককালীন বুট মেনু (USB বেছে নিতে) |
| `F10` | কিছু HP-তে সেটআপ; অনেক সেটআপে "Save & Exit" |
| `F1` | কিছু Lenovo ThinkPad-এ সেটআপ (`Enter` চেপে interrupt মেনু) |
| `Option` (Alt) চেপে রাখা | Intel Mac-এ বুট ডিভাইস বেছে নেওয়া |

### GRUB পর্যায়

| কী | কাজ |
|---|---|
| `Shift` চেপে রাখা | (BIOS) লুকানো GRUB মেনু দেখাও |
| `Esc` | (UEFI) লুকানো GRUB মেনু দেখাও; এডিট/কমান্ড লাইন থেকে মেনুতে ফেরো |
| `↑` / `↓` | entry বাছাই |
| `Enter` | বেছে নেওয়া entry বুট করো |
| `e` | entry এডিট (সাময়িক) |
| `Ctrl+x` বা `F10` | এডিট করা entry দিয়ে বুট |
| `c` | GRUB কমান্ড লাইন (`grub>`) |
| `Ctrl+c` | এডিট স্ক্রিন থেকে কমান্ড লাইনে |
| `Tab` | GRUB কমান্ড লাইনে auto-complete |
| `Ctrl+a` / `Ctrl+e` | এডিটরে লাইনের শুরু / শেষে |
| `Ctrl+k` | এডিটরে কার্সর থেকে লাইনের শেষ পর্যন্ত মুছে ফেলো |

### Kernel/systemd ও সেশন পর্যায়

| কী | কাজ |
|---|---|
| `Esc` (Plymouth splash চলাকালীন) | গ্রাফিক্যাল বুট স্ক্রিন সরিয়ে টেক্সট বার্তা দেখাও |
| `Ctrl+Alt+F1`…`F6` | virtual console (TTY) বদলাও; GUI থেকে টেক্সট কনসোলে |
| `Alt+F1`…`F6` / `Alt+←` `Alt+→` | টেক্সট কনসোলে থাকা অবস্থায় অন্য TTY-তে |
| `Ctrl+Alt+Del` | (টেক্সট কনসোলে) `ctrl-alt-del.target` = রিবুট; ২ সেকেন্ডে ৭ বারের বেশি চাপলে জোর করে তাৎক্ষণিক রিবুট |
| `Ctrl+D` | sulogin প্রম্পটে স্বাভাবিক বুট চালিয়ে যাও / শেল থেকে বেরোও |
| `Alt+SysRq+R,E,I,S,U,B` | "Magic SysRq": জমে যাওয়া সিস্টেম নিরাপদে রিবুট (raw keyboard, SIGTERM, SIGKILL, Sync, read-only remount, reBoot) |

> 📝 **মনে রাখো (SysRq):** কোন SysRq কাজ চালু থাকবে তা ঠিক করে `kernel.sysrq` মান। Ubuntu-তে ডিফল্ট মান `176` (= 16 sync + 32 remount read-only + 128 reboot), মানে শুধু `S`, `U`, `B` কাজ করে; `R`, `E`, `I` বন্ধ। Fedora/RHEL-এ ডিফল্ট `16` (শুধু sync)। দেখতে: `cat /proc/sys/kernel/sysrq`। ল্যাপটপে SysRq সাধারণত `PrtSc` কী, কখনো `Fn` লাগে।

---

## হাতে-কলমে অনুশীলন

> ⚠️ **সতর্কতা:** নিচের ল্যাবগুলো করার জন্য সবচেয়ে ভালো জায়গা একটা ভার্চুয়াল মেশিন (VirtualBox, GNOME Boxes, virt-manager)। VM-এ snapshot নিয়ে রাখো, ভুল হলে ফিরিয়ে আনা যাবে। ল্যাব ৪–৬ কখনো প্রোডাকশন সার্ভারে করবে না।

### ল্যাব ১: তোমার বুট পরিচিতি তৈরি করো

```
$ [ -d /sys/firmware/efi ] && echo UEFI || echo BIOS
$ lsblk -o NAME,SIZE,PTTYPE,FSTYPE,MOUNTPOINTS
$ findmnt /boot/efi
$ ls -lh /boot
$ uname -r
$ cat /proc/cmdline
$ ls -l /sbin/init
$ systemctl get-default
```

প্রতিটির উত্তর একটা খাতায় লেখো: firmware মোড, পার্টিশন টেবিল, ESP কোথায় মাউন্ট, kernel ভার্সন, root কোন UUID, ডিফল্ট target।

### ল্যাব ২: বুট সময় বিশ্লেষণ

```
$ systemd-analyze
$ systemd-analyze blame | head -10
$ systemd-analyze critical-chain
$ systemd-analyze plot > ~/boot.svg
$ xdg-open ~/boot.svg         # ব্রাউজারে খুলবে
```

প্রশ্ন: কোন ধাপে (firmware/loader/kernel/initrd/userspace) সবচেয়ে বেশি সময় যাচ্ছে? critical chain-এ কোন unit সবচেয়ে বেশি `+` সময় নিচ্ছে?

### ল্যাব ৩: বুট লগ খোঁজা

```
$ sudo dmesg -T | head -20
$ sudo dmesg -T -l err,warn
$ journalctl --list-boots
$ journalctl -b -p err --no-pager
$ journalctl -k -b -1 | tail -20      # আগের বুটের শেষ kernel বার্তা
```

একটা USB ড্রাইভ লাগিয়ে আরেকটা টার্মিনালে দেখো:

```
$ sudo dmesg -wT
```

### ল্যাব ৪: GRUB মেনু দৃশ্যমান করা ও timeout বদলানো (Ubuntu/Debian)

```
$ sudo cp /etc/default/grub /etc/default/grub.bak
$ sudo nano /etc/default/grub
```

এই দুটো লাইন এভাবে করো:

```
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10
```

তারপর:

```
$ sudo update-grub
$ grep -E 'timeout|style' /boot/grub/grub.cfg | head
$ sudo reboot
```

রিবুটে ১০ সেকেন্ড মেনু দেখা যাবে। Fedora/RHEL-এ শেষ ধাপে `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`।

### ল্যাব ৫: সাময়িক kernel parameter আর বিভিন্ন target

1. রিবুট করে GRUB মেনুতে `e` চাপো।
2. `linux` লাইন থেকে `quiet splash` (বা `rhgb quiet`) মুছে শেষে লেখো `systemd.unit=multi-user.target`।
3. `Ctrl+x`। এবার সব বুট বার্তা দেখবে আর GUI ছাড়া টেক্সট লগইন পাবে।
4. লগইন করে যাচাই করো:

   ```
   $ cat /proc/cmdline
   $ systemctl is-active graphical.target
   inactive
   $ sudo systemctl isolate graphical.target     # GUI-তে ফেরো
   ```

5. আবার রিবুট করো। এবার পরিবর্তন ছাড়াই স্বাভাবিক বুট হবে, কারণ GRUB-এ এডিট ছিল সাময়িক।

### ল্যাব ৬: emergency mode-এ fstab মেরামত (শুধু VM-এ!)

1. snapshot নাও। তারপর `/etc/fstab`-এ ইচ্ছে করে একটা ভুল লাইন যোগ করো:

   ```
   $ sudo cp /etc/fstab /etc/fstab.bak
   $ echo 'UUID=00000000-dead-beef-0000-000000000000 /mnt/fake ext4 defaults 0 2' | sudo tee -a /etc/fstab
   $ sudo systemctl daemon-reload
   $ sudo reboot
   ```

2. প্রায় ৯০ সেকেন্ড অপেক্ষার পর emergency mode আসবে। root পাসওয়ার্ড দাও (Ubuntu-তে root লক থাকলে recovery mode → root, বা `init=/bin/bash` ব্যবহার করো)।
3. মেরামত:

   ```
   # journalctl -xb | grep -i fake
   # mount -o remount,rw /
   # sed -i '/dead-beef/d' /etc/fstab
   # systemctl daemon-reload
   # systemctl default
   ```

4. শিক্ষা: বাইরের ডিস্কের লাইনে `nofail` দিলে এমন হতো না।

### ল্যাব ৭: ইউজারের পাসওয়ার্ড রিসেট (শুধু VM-এ)

- Ubuntu/Debian VM-এ ২৭.১২-এর পদ্ধতি ক (`rw init=/bin/bash`) দিয়ে তোমার ইউজারের পাসওয়ার্ড বদলাও।
- RHEL/Rocky/Alma/Fedora VM থাকলে পদ্ধতি খ (`rd.break`) দিয়ে root পাসওয়ার্ড বদলাও, আর খেয়াল করো `/.autorelabel`-এর কারণে পরের বুট কতক্ষণ নেয়।

### ল্যাব ৮: initramfs অনুসন্ধান

```
# Ubuntu/Debian
$ lsinitramfs /boot/initrd.img-$(uname -r) | wc -l
$ lsinitramfs /boot/initrd.img-$(uname -r) | grep -E '/init$|busybox|cryptsetup|lvm'
$ sudo update-initramfs -u -v | tail -5

# Fedora/RHEL
$ sudo lsinitrd -m
$ sudo dracut -f -v 2>&1 | tail -5
```

প্রশ্ন: তোমার initramfs-এ কয়টি ফাইল আছে? `cryptsetup` আছে কি না, আর থাকলে কেন?

### ল্যাব ৯: Secure Boot ও UEFI entry

```
$ mokutil --sb-state
$ efibootmgr -v
$ sudo mokutil --list-enrolled | grep -i "subject:"
```

খেয়াল করো: তোমার লিনাক্স entry কোন `.efi` ফাইল চালায়? সেটা `shimx64.efi` হলে বুঝবে Secure Boot-এর শিকল shim দিয়ে শুরু।

---

## সারসংক্ষেপ

বুটের ধাপগুলো এক লাইনে:

**Power → Firmware (POST) → MBR/ESP → GRUB2 → kernel → initramfs → systemd (PID 1) → default.target → getty/display manager → login → shell/desktop**

### দ্রুত রেফারেন্স

| কাজ | কমান্ড / শর্টকাট |
|---|---|
| UEFI না BIOS | `[ -d /sys/firmware/efi ] && echo UEFI` |
| UEFI boot entry | `efibootmgr -v` |
| UEFI সেটআপে রিবুট | `sudo systemctl reboot --firmware-setup` |
| পার্টিশন টেবিলের ধরন | `lsblk -o NAME,PTTYPE` / `sudo fdisk -l` |
| GRUB সেটিংস | `/etc/default/grub` |
| grub.cfg পুনর্গঠন (Debian/Ubuntu) | `sudo update-grub` |
| grub.cfg পুনর্গঠন (Fedora/RHEL) | `sudo grub2-mkconfig -o /boot/grub2/grub.cfg` |
| kernel parameter (RHEL/Fedora) | `sudo grubby --update-kernel=ALL --args="..."` |
| ডিফল্ট kernel (RHEL/Fedora) | `sudo grubby --default-kernel` |
| GRUB পুনঃস্থাপন | `sudo grub-install ...` |
| বর্তমান kernel parameter | `cat /proc/cmdline` |
| kernel ভার্সন | `uname -r` |
| initramfs (Debian/Ubuntu) | `sudo update-initramfs -u [-k all]`, `lsinitramfs` |
| initramfs (Fedora/RHEL) | `sudo dracut -f [--regenerate-all]`, `lsinitrd` |
| ডিফল্ট target | `systemctl get-default` / `sudo systemctl set-default ...` |
| এখনই target বদল | `sudo systemctl isolate multi-user.target` |
| rescue / emergency | `systemd.unit=rescue.target` / `systemd.unit=emergency.target` |
| বুট সময় | `systemd-analyze`, `blame`, `critical-chain`, `plot` |
| kernel বার্তা | `sudo dmesg -T`, `-l err,warn`, `-w` |
| বুট লগ | `journalctl -b`, `-b -1`, `--list-boots`, `-k` |
| পাসওয়ার্ড রিসেট (Debian/Ubuntu) | GRUB `e` → `rw init=/bin/bash` → `passwd` → `exec /sbin/init` |
| পাসওয়ার্ড রিসেট (RHEL/Fedora) | GRUB `e` → `rd.break` → remount rw `/sysroot` → `chroot` → `passwd` → `touch /.autorelabel` |
| Secure Boot অবস্থা | `mokutil --sb-state` |
| MOK নিবন্ধন | `sudo mokutil --import key.der` → রিবুট → Enroll MOK |
| GRUB মেনু দেখাও | `Shift` (BIOS) / `Esc` (UEFI) |
| GRUB এডিট ও বুট | `e` → `Ctrl+x` বা `F10` |
| GRUB কমান্ড লাইন | `c` |
| TTY বদল | `Ctrl+Alt+F1`…`F6` |

---

## নিজেকে যাচাই করো

1. BIOS আর UEFI বুটের মূল পার্থক্য কী? MBR-এ bootloader-এর জন্য কত byte জায়গা থাকে?
2. MBR-এর তুলনায় GPT-র তিনটি সুবিধা বলো।
3. ESP কী, এর ফাইলসিস্টেম কী, আর Ubuntu/RHEL-এ এটা কোথায় মাউন্ট হয়?
4. `/boot/grub/grub.cfg` কেন সরাসরি এডিট করা উচিত না? স্থায়ীভাবে একটা kernel parameter যোগ করার সঠিক পদ্ধতি Ubuntu আর RHEL 9-এ কী?
5. initramfs কেন দরকার? এমন দুটি পরিস্থিতি বলো যেখানে initramfs ছাড়া বুট হবে না।
6. `rescue.target` আর `emergency.target`-এর পার্থক্য কী? `/etc/fstab`-এ ভুল UUID থাকলে সাধারণত কোনটায় গিয়ে পড়বে?
7. `systemd-analyze blame` আর `systemd-analyze critical-chain`-এর মধ্যে বুট ধীর হওয়ার কারণ খুঁজতে কোনটা বেশি নির্ভরযোগ্য, আর কেন?
8. RHEL 9-এ root পাসওয়ার্ড রিসেটে `touch /.autorelabel` কেন লাগে?
9. রিবুটের পর আগের বুটের kernel বার্তা `dmesg`-এ পাওয়া যায় না কেন? তাহলে কীভাবে দেখবে?
10. Secure Boot চালু থাকা অবস্থায় VirtualBox-এর module লোড করতে গিয়ে `Key was rejected by service` আসছে। সমস্যাটা কী আর সঠিক সমাধান কী?

<details><summary>উত্তর দেখো</summary>

1. BIOS ফাইলসিস্টেম চেনে না। সে ডিস্কের প্রথম 512-byte সেক্টর (MBR) অন্ধভাবে চালায়। UEFI নিজেই FAT32 পড়ে ESP থেকে `.efi` ফাইল চালায়, আর কোন ফাইল চালাবে তা NVRAM-এর boot entry বলে দেয়। এছাড়া UEFI-তে Secure Boot আছে। MBR-এ bootstrap code-এর জন্য 446 byte থাকে, তাই GRUB-কে বাকি অংশ (core.img) অন্য জায়গায় রাখতে হয়।

2. যেকোনো তিনটি: (ক) 2 TiB-এর চেয়ে বড় ডিস্ক সাপোর্ট করে; (খ) ডিফল্টে 128টি পার্টিশন, extended/logical-এর দরকার নেই; (গ) ডিস্কের শেষে ব্যাকআপ header ও পার্টিশন টেবিল থাকে; (ঘ) CRC32 দিয়ে নষ্ট হওয়া ধরা যায়; (ঙ) প্রতিটি পার্টিশনের অনন্য GUID থাকে।

3. ESP (EFI System Partition) হলো একটা FAT32 পার্টিশন, যেখানে UEFI firmware বুটযোগ্য `.efi` ফাইল (shim, GRUB, Windows Boot Manager) খোঁজে। Ubuntu, Debian, Fedora, RHEL-এ এটা `/boot/efi`-তে মাউন্ট হয়।

4. কারণ এটা স্ক্রিপ্ট দিয়ে স্বয়ংক্রিয়ভাবে তৈরি হয়। পরের kernel আপডেট বা `update-grub`-এ হাতের পরিবর্তন মুছে যাবে। Ubuntu-তে: `/etc/default/grub`-এ `GRUB_CMDLINE_LINUX_DEFAULT` (বা `GRUB_CMDLINE_LINUX`) বদলে `sudo update-grub`। RHEL 9-এ: `sudo grubby --update-kernel=ALL --args="..."` (চাইলে `/etc/default/grub`-এর `GRUB_CMDLINE_LINUX`-এ যোগ করে `grub2-mkconfig -o /boot/grub2/grub.cfg`)।

5. ডিস্ট্রোর kernel বেশিরভাগ ড্রাইভার module হিসেবে রাখে, যেগুলো root filesystem-এর ভেতরে থাকে। root মাউন্টের আগে সেগুলো দরকার। initramfs সেই ড্রাইভার ও টুল আগেভাগে RAM-এ এনে দেয়। পরিস্থিতি: root যখন LUKS-এনক্রিপ্টেড, LVM-এ, সফটওয়্যার RAID-এ, বা নেটওয়ার্কে (NFS/iSCSI) থাকে; অথবা স্টোরেজ ড্রাইভার kernel-এ বিল্ট-ইন না থাকে।

6. rescue mode সব লোকাল ফাইলসিস্টেম মাউন্ট করে আর মৌলিক service চালায় (single-user mode)। emergency mode শুধু root-কে read-only মাউন্ট করে আর প্রায় কিছুই চালায় না। `/etc/fstab`-এ ভুল UUID থাকলে মাউন্ট ব্যর্থ হয়ে `local-fs.target` পূরণ হয় না, তাই সাধারণত **emergency mode**-এ পড়বে।

7. `critical-chain`। `blame` শুধু বলে কোন unit কত সময় নিয়েছে, কিন্তু অনেক unit সমান্তরালে চলে, তাই বেশি সময় নেওয়া unit বুটকে আটকে নাও রাখতে পারে। `critical-chain` দেখায় সেই নির্ভরতার শিকল, যার জন্য target পৌঁছাতে আসলে দেরি হয়েছে।

8. `rd.break` + `chroot`-এ SELinux policy লোড হয় না। তাই `passwd` যে নতুন `/etc/shadow` লেখে, তার SELinux context ভুল থাকে। সেই অবস্থায় বুট করলে লগইন প্রসেস `/etc/shadow` পড়তে পারবে না, কেউ লগইন করতে পারবে না। `/.autorelabel` থাকলে পরের বুটে SELinux সব ফাইলের লেবেল ঠিক করে দেয়।

9. `dmesg` পড়ে kernel-এর ring buffer, যেটা RAM-এ থাকে আর রিবুটে মুছে যায়। আগের বুটের kernel বার্তা দেখতে `journalctl -k -b -1` চালাও। এর জন্য journal persistent হতে হবে (`/var/log/journal` থাকতে হবে)।

10. Secure Boot চালু থাকায় kernel শুধু বিশ্বস্ত চাবি দিয়ে সই করা module লোড করে। VirtualBox-এর module DKMS দিয়ে তোমার মেশিনে কম্পাইল হয়েছে, তাতে বিশ্বস্ত সই নেই। সঠিক সমাধান: নিজের MOK চাবি দিয়ে module সই করা আর সেই চাবি `sudo mokutil --import <key>.der` দিয়ে নিবন্ধন করা, তারপর রিবুটে MOK Manager-এ "Enroll MOK" নিশ্চিত করা। Secure Boot বন্ধ করলেও কাজ হবে, কিন্তু নিরাপত্তা কমে যায়।

</details>

---

[← আগের অধ্যায়](26-systemd-services.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](28-logs.md)

# অধ্যায় ৪৭: কার্নেল, মডিউল, /proc ও /sys

> "লিনাক্স" শব্দটা আসলে মূলত একটা জিনিসের নাম, সেটা হলো **kernel**। বাকি সব (shell, desktop, `ls`, `grep`) kernel-এর ওপরে বসানো সফটওয়্যার। এই অধ্যায়ে আমরা সেই kernel-এর একদম কাছে যাব: কোন kernel চলছে, কীভাবে নতুন driver লোড হয়, kernel নিজের ভেতরের খবর কোথায় রাখে, আর চলন্ত অবস্থাতেই kernel-এর আচরণ কীভাবে বদলানো যায়।

## এই অধ্যায়ে যা শিখবে

- kernel কী, এবং kernel version (`uname -r`) পড়ার নিয়ম ও নামকরণের মানে
- monolithic kernel আর loadable module-এর ধারণা
- module নিয়ে কাজ: `lsmod`, `modinfo`, `modprobe`, `insmod`, `rmmod`, `depmod`
- boot-এর সময় module লোড করা (`/etc/modules-load.d`), module option দেওয়া এবং blacklist করা
- `/proc` ফাইলসিস্টেম: `cpuinfo`, `meminfo`, প্রতিটি process-এর PID ডিরেক্টরি এবং আরও অনেক কিছু
- `/sys` (sysfs): ডিভাইস, driver ও kernel object-এর গাছ
- `sysctl` দিয়ে kernel-এর runtime প্যারামিটার দেখা ও বদলানো (`-a`, `-w`, `/etc/sysctl.d`)
- udev: ডিভাইস প্লাগ করলে কী ঘটে, আর নিজের udev rule কীভাবে লিখতে হয়
- হার্ডওয়্যারের তথ্য বের করা: `lscpu`, `lspci`, `lsusb`, `dmidecode`, `lshw` ইত্যাদি
- kernel আপডেট, একাধিক kernel রাখা, পুরোনো kernel-এ boot করা
- DKMS: kernel আপডেটের পরও third-party driver যেন কাজ করে
- kernel panic ও oops কী, কেন হয়, আর তখন কী করতে হয়

---

## ৪৭.১ kernel আসলে কী — এক নজরে

[অধ্যায় ২](02-linux-architecture.md)-এ আমরা লিনাক্সের স্তরগুলো দেখেছি। এখানে শুধু মনে করিয়ে দিই।

**Kernel** (বাংলায় বলা যায় "কেন্দ্রীয় শাঁস") হলো সেই প্রোগ্রাম যেটা সবার আগে মেমরিতে লোড হয়, আর কম্পিউটার বন্ধ না হওয়া পর্যন্ত চলতে থাকে। হার্ডওয়্যারের ওপর সরাসরি নিয়ন্ত্রণ শুধু এর হাতেই থাকে।

**উপমা:** একটা বড় হোটেলের কথা ভাবো। অতিথিরা (process/অ্যাপ্লিকেশন) রান্নাঘরে, স্টোররুমে বা ইলেকট্রিক রুমে নিজেরা ঢুকতে পারে না। তারা রিসেপশনে (system call) গিয়ে অনুরোধ করে, আর হোটেলের স্টাফ (kernel) কাজটা করে দেয়। এতে কোনো অতিথি অন্যের রুমে ঢুকতে পারে না, আর কেউ ভুল করে পুরো হোটেলের বিদ্যুৎ বন্ধও করে দিতে পারে না।

```
+-------------------------------------------------------------+
|                 USER SPACE (ইউজার স্পেস)                     |
|   bash   firefox   nginx   python   systemd   ls  ...       |
|        |          |          |          |                   |
|   glibc (C library) -- open(), read(), fork() wrapper       |
+--------------------------|----------------------------------+
                           |  system call (syscall)
+--------------------------v----------------------------------+
|                 KERNEL SPACE (কার্নেল স্পেস)                  |
|  +-----------+ +-----------+ +---------+ +----------------+ |
|  | Process   | | Memory    | | VFS +   | | Network stack  | |
|  | scheduler | | manager   | | FS      | | (TCP/IP)       | |
|  +-----------+ +-----------+ +---------+ +----------------+ |
|  +-------------------------------------------------------+  |
|  |  Device drivers  (built-in  +  loadable modules .ko)  |  |
|  +-------------------------------------------------------+  |
|        ^ /proc ^ /sys ^ sysctl  <-- kernel-এর "জানালা"      |
+--------|-----------------------------------------------------+
         v
+-------------------------------------------------------------+
|   HARDWARE: CPU, RAM, Disk, NIC, USB, GPU ...               |
+-------------------------------------------------------------+
```

এই অধ্যায়ের মূল কথা ডায়াগ্রামের নিচের দিকের ওই "জানালা"গুলো: `/proc`, `/sys` আর `sysctl`। এগুলো দিয়ে user space থেকে kernel-এর ভেতরে উঁকি দেওয়া যায়, কিছু ক্ষেত্রে বদলানোও যায়।

### Monolithic, তবু modular

লিনাক্স kernel হলো **monolithic kernel**। মানে scheduler, memory manager, filesystem, network stack আর driver সব একই address space-এ, একই বড় প্রোগ্রাম হিসেবে চলে। এর উল্টোটা **microkernel** (যেমন MINIX, QNX), যেখানে driver আলাদা process হিসেবে চলে।

monolithic হওয়ার সুবিধা হলো গতি, কারণ ভেতরের অংশগুলো একে অপরকে সরাসরি function call করতে পারে। অসুবিধা হলো, পৃথিবীর সব হার্ডওয়্যারের driver যদি একটা ফাইলে ঢুকিয়ে দেওয়া হয়, kernel বিশাল হয়ে যাবে। এর সমাধান হলো **Loadable Kernel Module (LKM)**। দরকারি অংশ kernel-এর ভেতরে **built-in** থাকে, আর বাকি driver আলাদা `.ko` (kernel object) ফাইলে থাকে। দরকার হলে চলন্ত kernel-এ এগুলো জুড়ে দেওয়া যায় (load), আবার খুলেও ফেলা যায় (unload)।

> 📝 **মনে রাখো:** module লোড হলে সেটা kernel space-এ চলে, পূর্ণ ক্ষমতা নিয়ে। একটা খারাপ module পুরো সিস্টেম ক্র্যাশ করাতে পারে। এজন্যই module লোড/আনলোড করতে root লাগে।

---

## ৪৭.২ kernel version: `uname` ও নামকরণ

### `uname` কমান্ড

`uname` (unix name) চলমান kernel আর মেশিনের মৌলিক তথ্য দেখায়।

**Syntax:** `uname [OPTION]`

| অপশন | মানে |
|---|---|
| `-s` | kernel-এর নাম (`Linux`), অপশন না দিলেও এটাই দেখায় |
| `-n` | hostname (নেটওয়ার্কে মেশিনের নাম) |
| `-r` | kernel release, সবচেয়ে বেশি ব্যবহৃত |
| `-v` | kernel version string (build নম্বর ও তারিখ) |
| `-m` | মেশিন হার্ডওয়্যার আর্কিটেকচার (`x86_64`, `aarch64`) |
| `-o` | operating system (`GNU/Linux`) |
| `-a` | সব একসাথে |

```
$ uname -r
6.8.0-45-generic

$ uname -m
x86_64

$ uname -a
Linux mypc 6.8.0-45-generic #45-Ubuntu SMP PREEMPT_DYNAMIC Fri Aug 30 12:02:04 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

একই তথ্য অন্য জায়গা থেকেও পাওয়া যায়:

```
$ cat /proc/version
Linux version 6.8.0-45-generic (buildd@lcy02-amd64-075) (x86_64-linux-gnu-gcc-13 ...) #45-Ubuntu SMP PREEMPT_DYNAMIC ...

$ cat /proc/sys/kernel/osrelease
6.8.0-45-generic

$ hostnamectl | grep -i kernel
          Kernel: Linux 6.8.0-45-generic
```

### version নম্বর কীভাবে পড়বে

upstream (Linus Torvalds-এর kernel.org) kernel-এর version তিন ভাগে: `MAJOR.MINOR.PATCH`, যেমন `6.8.12`।

- **6**: major। এটা বাড়ার পেছনে বড় কোনো প্রযুক্তিগত কারণ থাকে না; minor নম্বর অনেক বড় হয়ে গেলে Linus এটা বাড়ান (3.19 থেকে 4.0, 4.20 থেকে 5.0, 5.19 থেকে 6.0)।
- **8**: minor। প্রতি ৯–১০ সপ্তাহে একটা নতুন minor release আসে, তাতে নতুন feature থাকে।
- **12**: patch বা stable release। শুধু bug fix ও security fix।

Distribution-গুলো এর পেছনে নিজেদের অংশ জুড়ে দেয়:

```
  6 . 8 . 0 - 45 - generic           (Ubuntu 24.04)
  |   |   |   |     |
  |   |   |   |     +-- flavour: generic / lowlatency / aws / azure ...
  |   |   |   +-------- Ubuntu-র ABI/build নম্বর (প্যাকেজ আপডেটে বাড়ে)
  |   |   +------------ Ubuntu এখানে upstream patch নম্বর না বসিয়ে 0 রাখে
  +---+---------------- upstream 6.8 সিরিজের ওপর ভিত্তি করে বানানো

  6 . 1 . 0 - 25 - amd64             (Debian 12)
  5 . 14 . 0 - 427.13.1.el9_4 . x86_64   (RHEL 9 / Rocky / Alma)
  6 . 10 . 6 - 200.fc40 . x86_64      (Fedora 40)
```

> 💡 **টিপস:** RHEL 9-এর kernel `5.14` দেখে অনেকে ভাবে এটা "পুরোনো"। আসলে Red Hat পুরো RHEL 9-এর জীবনকাল ধরে 5.14 ভিত্তি ধরে রাখে, কিন্তু নতুন kernel থেকে হাজার হাজার fix আর feature **backport** করে (পেছনে টেনে আনে)। তাই শুধু নম্বর দেখে feature আছে কিনা বলা যায় না।

### kernel-এর ধরন: mainline, stable, LTS

| ধরন | মানে |
|---|---|
| **mainline** | Linus-এর সরাসরি ট্রি, `-rc1`, `-rc2`… (release candidate) |
| **stable** | প্রকাশিত version-এর bug fix সিরিজ, অল্প কিছুদিন চলে |
| **longterm (LTS)** | কয়েক বছর fix পায়, যেমন 5.15, 6.1, 6.6, 6.12 |
| **distro kernel** | distro নিজে patch করে, পরীক্ষা করে প্যাকেজ বানায়; সাধারণত এটাই ব্যবহার করা উচিত |

### kernel ফাইলগুলো কোথায় থাকে

```
$ ls /boot
config-6.8.0-45-generic      initrd.img-6.8.0-45-generic   vmlinuz-6.8.0-45-generic
config-6.8.0-41-generic      initrd.img-6.8.0-41-generic   vmlinuz-6.8.0-41-generic
System.map-6.8.0-45-generic  grub/  efi/ ...
```

| ফাইল | কী |
|---|---|
| `vmlinuz-<ver>` | সংকুচিত (compressed) kernel image। "z" মানে compressed |
| `initrd.img-<ver>` (Debian/Ubuntu) / `initramfs-<ver>.img` (Fedora/RHEL) | প্রাথমিক RAM ডিস্ক: আসল root ফাইলসিস্টেম মাউন্ট করার আগে যে driver লাগে সেগুলো এখানে থাকে |
| `config-<ver>` | এই kernel কোন অপশন নিয়ে compile হয়েছে |
| `System.map-<ver>` | kernel symbol-এর ঠিকানার তালিকা (debug-এর কাজে) |
| `/lib/modules/<ver>/` | এই kernel-এর সব module |

কোনো feature built-in নাকি module, `config` ফাইল থেকে দেখা যায়:

```
$ grep -E 'CONFIG_EXT4_FS=|CONFIG_BTRFS_FS=' /boot/config-$(uname -r)
CONFIG_EXT4_FS=y
CONFIG_BTRFS_FS=m
```

`=y` মানে built-in (kernel-এর ভেতরেই আছে), `=m` মানে module হিসেবে বানানো, আর `# ... is not set` মানে একেবারেই নেই। boot ও kernel image-এর পুরো যাত্রা নিয়ে [অধ্যায় ২৭](27-boot-process.md)-এ বিস্তারিত বলা আছে।

---

## ৪৭.৩ kernel module

### module ফাইল কোথায় থাকে

```
/lib/modules/6.8.0-45-generic/
├── kernel/
│   ├── drivers/        <- net, usb, gpu, block ... driver
│   ├── fs/             <- btrfs, xfs, nfs ... filesystem
│   ├── net/            <- netfilter, bridge ... network feature
│   ├── sound/
│   └── crypto/
├── modules.dep         <- কোন module কোনটার ওপর নির্ভরশীল
├── modules.alias       <- হার্ডওয়্যার ID থেকে module-এর নাম
├── modules.builtin     <- যেগুলো built-in (লোড করার দরকার নেই)
└── updates/ , extra/   <- DKMS বা third-party module
```

ফাইলের extension distro অনুযায়ী আলাদা হয়: `.ko`, `.ko.xz` (RHEL/Fedora), অথবা `.ko.zst` (Ubuntu 24.04)। শেষের দুটো compressed। `modprobe` নিজেই এগুলো খুলে নেয়।

> 📝 **মনে রাখো:** আধুনিক distro-তে `/lib` আসলে `/usr/lib`-এর symlink ("merged /usr"), তাই `/usr/lib/modules` আর `/lib/modules` একই জায়গা।

### `lsmod`: কোন module লোড আছে

`lsmod` আসলে `/proc/modules` ফাইলটাকে সুন্দর করে দেখায়।

```
$ lsmod | head -6
Module                  Size  Used by
snd_hda_intel          61440  4
snd_intel_dspcfg       36864  1 snd_hda_intel
snd_hda_codec         204800  2 snd_hda_codec_hdmi,snd_hda_intel
iwlwifi               598016  1 iwlmvm
cfg80211             1355776  3 iwlmvm,iwlwifi,mac80211
```

| কলাম | মানে |
|---|---|
| `Module` | module-এর নাম |
| `Size` | মেমরিতে কত বাইট নিয়েছে |
| `Used by` | কতজন ব্যবহার করছে (সংখ্যা), আর কোন module-গুলো এর ওপর নির্ভরশীল |

`Used by`-এর সংখ্যা ০ না হলে module-টা সাধারণত আনলোড করা যায় না।

```
$ lsmod | grep -i usb_storage
usb_storage            86016  1 uas
```

> 💡 **টিপস:** built-in অংশ `lsmod`-এ দেখা যায় না, কারণ সেগুলো আলাদা module হিসেবে লোডই হয় না। দেখতে চাইলে: `grep ext4 /lib/modules/$(uname -r)/modules.builtin`।

### `modinfo`: module-এর পরিচয়পত্র

```
$ modinfo e1000e
filename:       /lib/modules/6.8.0-45-generic/kernel/drivers/net/ethernet/intel/e1000e/e1000e.ko.zst
license:        GPL v2
description:    Intel(R) PRO/1000 Network Driver
author:         Intel Corporation, <linux.nics@intel.com>
alias:          pci:v00008086d0000550Esv*sd*bc*sc*i*
...
depends:
intree:         Y
name:           e1000e
vermagic:       6.8.0-45-generic SMP preempt mod_unload modversions
sig_id:         PKCS#7
signer:         Build time autogenerated kernel key
parm:           debug:Debug level (0=none,...,16=all) (int)
parm:           IntMode:Interrupt Mode (array of int)
```

| ফিল্ড | গুরুত্ব |
|---|---|
| `filename` | ডিস্কে ফাইলটা কোথায় |
| `description`, `author`, `license` | পরিচয়। license GPL না হলে kernel "tainted" চিহ্নিত হয় |
| `alias` | কোন হার্ডওয়্যার ID দেখলে এই module লোড হবে |
| `depends` | আগে কোন module লোড থাকতে হবে |
| `vermagic` | কোন kernel-এর জন্য বানানো; না মিললে লোড হবে না |
| `signer`, `sig_id` | digital signature (Secure Boot-এ গুরুত্বপূর্ণ) |
| `parm` | কোন কোন প্যারামিটার দেওয়া যায় |

কাজের অপশন:

| অপশন | কাজ |
|---|---|
| `-F field` | শুধু একটা ফিল্ড, যেমন `modinfo -F parm e1000e` |
| `-p` | শুধু প্যারামিটারগুলো |
| `-k <ver>` | অন্য kernel version-এর module দেখো |

### `modprobe`: বুদ্ধিমান লোডার

`modprobe` হলো module লোড/আনলোডের সঠিক উপায়। এটা `modules.dep` পড়ে আগে dependency লোড করে নেয়, তারপর আসল module। আর `/etc/modprobe.d/`-এর কনফিগও মেনে চলে।

```
$ sudo modprobe -v vfat
insmod /lib/modules/6.8.0-45-generic/kernel/fs/fat/fat.ko.zst
insmod /lib/modules/6.8.0-45-generic/kernel/fs/fat/vfat.ko.zst
```

দেখো, `vfat` চাওয়া হয়েছিল, কিন্তু `modprobe` নিজে থেকেই আগে `fat` লোড করেছে।

| অপশন | কাজ |
|---|---|
| `modprobe <name>` | লোড করো (dependency সহ) |
| `modprobe <name> param=value` | প্যারামিটার দিয়ে লোড করো |
| `-r <name>` | আনলোড করো (আর অব্যবহৃত dependency-গুলোও) |
| `-v` | কী করছে দেখাও (verbose) |
| `-n` | dry run: আসলে কিছু করবে না (`-n -v` একসাথে খুব কাজের) |
| `--show-depends <name>` | কোন কোন ফাইল লোড হবে সেটা দেখাও |
| `-c` | সব কার্যকর কনফিগারেশন দেখাও |
| `-a` | একাধিক module একসাথে লোড করো |

```
$ modprobe --show-depends vfat
insmod /lib/modules/6.8.0-45-generic/kernel/fs/fat/fat.ko.zst
insmod /lib/modules/6.8.0-45-generic/kernel/fs/fat/vfat.ko.zst

$ sudo modprobe -r vfat
$ lsmod | grep fat
$
```

### `insmod`, `rmmod`, `depmod`: নিচু স্তরের টুল

| কমান্ড | কাজ | সমস্যা |
|---|---|---|
| `insmod /path/file.ko` | নির্দিষ্ট ফাইল সরাসরি লোড করে | dependency দেখে না, পুরো path দিতে হয় |
| `rmmod <name>` | module সরিয়ে দেয় | dependency দেখে না; ব্যবহারে থাকলে ব্যর্থ হয় |
| `depmod -a` | `modules.dep` ও `modules.alias` আবার বানায় | নতুন module হাতে কপি করার পর চালাতে হয় |

```
$ sudo rmmod snd_hda_intel
rmmod: ERROR: Module snd_hda_intel is in use
```

> ⚠️ **সতর্কতা:** `rmmod -f` (force) দিয়ে ব্যবহৃত module জোর করে সরালে সিস্টেম ক্র্যাশ করতে পারে। বেশিরভাগ distro kernel-এ force unload বন্ধও থাকে। সাধারণ কাজে সবসময় `modprobe` / `modprobe -r` ব্যবহার করো।

### module স্বয়ংক্রিয়ভাবে কীভাবে লোড হয়

তুমি একটা USB Wi-Fi অ্যাডাপ্টার লাগালে, `modprobe` না চালালেও driver লোড হয়ে যায়। কীভাবে?

```
 [USB ডিভাইস লাগানো হলো]
          |
          v
 kernel ডিভাইসের ID পড়ে  --> "usb:v0BDAp8179..."  (modalias)
          |
          v
 kernel একটা uevent পাঠায় (netlink দিয়ে) ---> systemd-udevd
          |
          v
 udevd: "modprobe usb:v0BDAp8179..."  (alias দিয়ে)
          |
          v
 modprobe modules.alias খোঁজে --> r8188eu / rtl8xxxu
          |
          v
 module লোড হলো, driver ডিভাইসকে "bind" করল --> wlan0 তৈরি
```

যেকোনো ডিভাইসের modalias দেখা যায় `/sys`-এ:

```
$ cat /sys/class/net/enp3s0/device/modalias
pci:v00008086d000015BCsv000017AAsd00002292bc02sc00i00
```

### boot-এর সময় module লোড করা: `/etc/modules-load.d`

কিছু module আছে যেগুলো কোনো হার্ডওয়্যার দেখে লোড হয় না, যেমন `br_netfilter` বা `overlay` (container-এর জন্য), কিংবা `loop`। প্রতিবার boot-এ এগুলো লোড করতে `/etc/modules-load.d/` ডিরেক্টরিতে একটা `.conf` ফাইল বানাও, প্রতি লাইনে একটা module-এর নাম:

```
$ sudo tee /etc/modules-load.d/k8s.conf <<'EOF'
# Kubernetes-এর জন্য দরকারি module
overlay
br_netfilter
EOF
```

boot-এর সময় `systemd-modules-load.service` এই ফাইলগুলো পড়ে। এখনই পরীক্ষা করতে চাইলে:

```
$ sudo systemctl restart systemd-modules-load.service
$ lsmod | grep -E 'overlay|br_netfilter'
br_netfilter           32768  0
overlay               212992  0
```

| লোকেশন | কে ব্যবহার করে |
|---|---|
| `/etc/modules-load.d/*.conf` | administrator (তুমি) |
| `/usr/lib/modules-load.d/*.conf` | প্যাকেজ (হাত দিও না) |
| `/run/modules-load.d/*.conf` | runtime, অস্থায়ী |
| `/etc/modules` | Debian/Ubuntu-র পুরোনো পদ্ধতি, এখনও কাজ করে |

### module option: `/etc/modprobe.d`

module লোড হওয়ার সময় স্থায়ীভাবে কোনো প্যারামিটার দিতে চাইলে `/etc/modprobe.d/` ডিরেক্টরিতে লেখো:

```
# /etc/modprobe.d/sound.conf
options snd_hda_intel power_save=1
```

module এখন লোড থাকলে তার বর্তমান প্যারামিটার দেখা যায় `/sys/module` থেকে:

```
$ cat /sys/module/snd_hda_intel/parameters/power_save
1
```

`/etc/modprobe.d/`-এর ফাইলগুলোতে যেসব নির্দেশনা দেওয়া যায়:

| নির্দেশনা | উদাহরণ | মানে |
|---|---|---|
| `options` | `options kvm_intel nested=1` | প্যারামিটার দাও |
| `alias` | `alias mynet e1000e` | অন্য নাম |
| `blacklist` | `blacklist nouveau` | alias দিয়ে স্বয়ংক্রিয়ভাবে লোড হবে না |
| `install` | `install usb_storage /bin/false` | লোড চাইলে module-এর বদলে এই কমান্ড চলবে |
| `softdep` | `softdep nvme pre: crc32c` | আগে/পরে অন্য module লোড করো |

### blacklist: কোনো module যেন লোড না হয়

উদাহরণ: NVIDIA-র proprietary driver ইনস্টল করার সময় open-source `nouveau` driver আটকাতে হয়।

```
# /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
```

এখানে একটা সূক্ষ্ম বিষয় বুঝে রাখো। `blacklist` শুধু **alias দিয়ে স্বয়ংক্রিয় লোড** বন্ধ করে। কেউ হাতে `modprobe nouveau` চালালে, বা অন্য কোনো module এর ওপর নির্ভরশীল হলে, এটা তবুও লোড হবে। পুরোপুরি আটকাতে হলে:

```
# /etc/modprobe.d/disable-usb-storage.conf
install usb_storage /bin/false
```

এটা নিরাপত্তার জন্য প্রচলিত কৌশল: সার্ভারে USB pen drive চলবে না (দেখো [অধ্যায় ৪৫](45-security-hardening.md))।

**গুরুত্বপূর্ণ:** অনেক driver boot-এর একদম শুরুতে initramfs থেকে লোড হয়। তাই blacklist করার পর initramfs আবার বানাতে হয়:

```
# Debian / Ubuntu
$ sudo update-initramfs -u

# Fedora / RHEL
$ sudo dracut -f
```

আর সিস্টেম boot-ই না হলে GRUB-এ kernel command line-এ সাময়িকভাবে যোগ করা যায়: `modprobe.blacklist=nouveau`।

> 💡 **টিপস:** কোন blacklist কার্যকর আছে সেটা দেখতে: `modprobe -c | grep -i blacklist`।

---

## ৪৭.৪ `/proc`: kernel-এর জীবন্ত জানালা

### `/proc` কী

`/proc` একটা **virtual filesystem** (procfs)। এর একটা ফাইলও ডিস্কে নেই। তুমি যখন `cat /proc/meminfo` চালাও, kernel ঠিক তখনই মেমরির বর্তমান অবস্থা হিসাব করে লেখা বানিয়ে তোমাকে দেয়।

**উপমা:** গাড়ির ড্যাশবোর্ড। স্পিডোমিটারের কাঁটা কোথাও "সংরক্ষিত" থাকে না; তুমি তাকালেই তখনকার গতি দেখায়।

```
$ mount | grep '^proc'
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)

$ ls -l /proc/meminfo
-r--r--r-- 1 root root 0 Nov 12 10:00 /proc/meminfo
```

দেখো, ফাইলের সাইজ **0**। তবু `cat` করলে অনেক লেখা আসে। কারণ লেখাটা পড়ার মুহূর্তে তৈরি হয়।

> 📝 **মনে রাখো:** "Everything is a file" দর্শনের সবচেয়ে সুন্দর উদাহরণ `/proc` আর `/sys`। kernel-এর তথ্য পড়তে আলাদা কোনো API লাগে না; `cat`, `grep`, `awk` দিয়েই কাজ হয়ে যায়।

### `/proc/cpuinfo`

প্রতিটি logical CPU-র জন্য একটা করে ব্লক থাকে:

```
$ grep -m1 'model name' /proc/cpuinfo
model name	: Intel(R) Core(TM) i7-10510U CPU @ 1.80GHz

$ grep -c ^processor /proc/cpuinfo
8

$ grep -m1 -o -w -E 'vmx|svm' /proc/cpuinfo
vmx
```

| ফিল্ড | মানে |
|---|---|
| `processor` | logical CPU নম্বর (0 থেকে শুরু) |
| `model name` | CPU-র নাম |
| `cpu MHz` | এই মুহূর্তের clock speed (বদলাতে থাকে) |
| `cpu cores` | একটা physical chip-এ কয়টা core |
| `siblings` | একটা chip-এ কয়টা logical CPU (hyper-threading সহ) |
| `flags` | CPU কী কী feature সাপোর্ট করে (`vmx`/`svm` = virtualization, `aes`, `avx2` …) |

> 💡 **টিপস:** CPU সংখ্যা জানার সহজ উপায় `nproc`। আর `/proc/cpuinfo`-এর পুরো তথ্য সাজানো আকারে দেখায় `lscpu` (৪৭.৮ অংশে দেখবো)। ARM মেশিনে (Raspberry Pi, AWS Graviton) `cpuinfo`-র ফিল্ডগুলো অনেক আলাদা।

### `/proc/meminfo`

```
$ head -8 /proc/meminfo
MemTotal:       16165604 kB
MemFree:         1203416 kB
MemAvailable:    9876540 kB
Buffers:          412300 kB
Cached:          7890120 kB
SwapCached:          120 kB
Active:          6123456 kB
Inactive:        5987654 kB
```

| ফিল্ড | মানে |
|---|---|
| `MemTotal` | মোট ব্যবহারযোগ্য RAM |
| `MemFree` | একেবারে খালি RAM (এটা কম হলেও ভয়ের কিছু নেই) |
| `MemAvailable` | নতুন প্রোগ্রাম আসলে কতটা পেতে পারে (cache ছেড়ে দিয়ে); **আসল হিসাব এটাই** |
| `Buffers`, `Cached` | ডিস্কের ডেটার cache; দরকার হলে kernel ছেড়ে দেয় |
| `SwapTotal`, `SwapFree` | swap-এর অবস্থা |
| `Dirty` | ডিস্কে লেখার অপেক্ষায় থাকা ডেটা |
| `HugePages_Total` | hugepage (ডেটাবেসের জন্য বড় page) |

`free -h` কমান্ড এই ফাইলটাই পড়ে। মেমরি বিশ্লেষণ নিয়ে বিস্তারিত আছে [অধ্যায় ৪৬](46-performance-troubleshooting.md)-এ।

### `/proc`-এর আরও দরকারি ফাইল

| ফাইল | কী দেখায় | সংশ্লিষ্ট কমান্ড |
|---|---|---|
| `/proc/version` | kernel version ও build তথ্য | `uname -a` |
| `/proc/cmdline` | boot-এর সময় kernel-কে দেওয়া প্যারামিটার | |
| `/proc/uptime` | চালু থাকার সেকেন্ড, idle সেকেন্ড | `uptime` |
| `/proc/loadavg` | ১, ৫, ১৫ মিনিটের load average | `uptime`, `top` |
| `/proc/stat` | CPU সময়ের হিসাব, context switch | `vmstat`, `top` |
| `/proc/modules` | লোড হওয়া module | `lsmod` |
| `/proc/mounts` | মাউন্ট হওয়া ফাইলসিস্টেম (আসলে `self/mounts`-এর link) | `mount`, `findmnt` |
| `/proc/filesystems` | kernel কোন কোন filesystem চেনে | |
| `/proc/partitions` | ডিস্ক পার্টিশন | `lsblk` |
| `/proc/swaps` | সক্রিয় swap | `swapon --show` |
| `/proc/interrupts` | কোন CPU-তে কোন ডিভাইসের কতগুলো interrupt এসেছে | |
| `/proc/devices` | character ও block ডিভাইসের major নম্বর | |
| `/proc/net/` | network-এর অবস্থা (`tcp`, `dev`, `route`) | `ss`, `ip` |
| `/proc/sys/` | পরিবর্তনযোগ্য kernel প্যারামিটার | `sysctl` |
| `/proc/sysrq-trigger` | SysRq কমান্ড পাঠানোর জায়গা (৪৭.১১ দেখো) | |

```
$ cat /proc/cmdline
BOOT_IMAGE=/boot/vmlinuz-6.8.0-45-generic root=UUID=3f1c...e9 ro quiet splash vt.handoff=7

$ cat /proc/loadavg
0.52 0.61 0.58 2/1043 245871
```

`/proc/loadavg`-এর চতুর্থ ঘরে `2/1043` মানে এখন ২টা চলছে (runnable), মোট ১০৪৩টা thread আছে। শেষ সংখ্যাটা সবচেয়ে সাম্প্রতিক তৈরি হওয়া PID।

### PID ডিরেক্টরি: প্রতিটি process-এর খাতা

`/proc`-এ সংখ্যা নামের যেসব ডিরেক্টরি আছে, প্রতিটা একটা চলন্ত process (সংখ্যাটা তার PID)। process শেষ হলে ডিরেক্টরিটাও মিলিয়ে যায়।

```
$ ls -d /proc/[0-9]* | head -3
/proc/1
/proc/2
/proc/3

$ sleep 1000 &
[1] 24567
$ ls /proc/24567
attr  cgroup  cmdline  comm  cwd  environ  exe  fd  io  limits  maps  mountinfo  ns  oom_score  oom_score_adj  root  stat  statm  status  task ...
```

```
 /proc/<PID>/
 ├── cmdline     কোন কমান্ড দিয়ে চালু হয়েছে (argument গুলো NUL দিয়ে আলাদা)
 ├── comm        process-এর ছোট নাম
 ├── status      মানুষের পড়ার মতো সারাংশ: State, PPid, Uid, VmRSS, Threads
 ├── stat        ps/top-এর জন্য এক লাইনের কাঁচা তথ্য
 ├── environ     environment variable (NUL দিয়ে আলাদা)
 ├── cwd  ->     বর্তমান ডিরেক্টরির symlink
 ├── exe  ->     আসল executable ফাইলের symlink
 ├── root ->     process-এর root ডিরেক্টরি (chroot/container-এ আলাদা)
 ├── fd/         খোলা প্রতিটি ফাইলের symlink (0=stdin, 1=stdout, 2=stderr)
 ├── maps        মেমরিতে কী কী ম্যাপ করা (library, heap, stack)
 ├── limits      resource limit (ulimit)
 ├── io          কত বাইট পড়েছে/লিখেছে
 ├── ns/         কোন namespace-এ আছে (container বোঝার জন্য, অধ্যায় ৪৮)
 ├── oom_score_adj  মেমরি ফুরালে একে মারার অগ্রাধিকার (-1000 থেকে 1000)
 └── task/       প্রতিটি thread-এর আলাদা ডিরেক্টরি
```

বাস্তব উদাহরণ:

```
$ cat /proc/24567/cmdline | tr '\0' ' '; echo
sleep 1000

$ grep -E 'State|PPid|VmRSS|Threads' /proc/24567/status
State:	S (sleeping)
PPid:	23001
VmRSS:	    1792 kB
Threads:	1

$ ls -l /proc/24567/cwd /proc/24567/exe
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:05 /proc/24567/cwd -> /home/rahim
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:05 /proc/24567/exe -> /usr/bin/sleep

$ ls -l /proc/24567/fd
lrwx------ 1 rahim rahim 64 Nov 12 10:05 0 -> /dev/pts/1
lrwx------ 1 rahim rahim 64 Nov 12 10:05 1 -> /dev/pts/1
lrwx------ 1 rahim rahim 64 Nov 12 10:05 2 -> /dev/pts/1
```

**`/proc/self`**: যে process `/proc` পড়ছে, তার নিজের ডিরেক্টরির link।

```
$ readlink /proc/self
24601          <- এটা ls/readlink process-এর নিজের PID, তাই প্রতিবার আলাদা
```

> 💡 **টিপস (জীবন বাঁচানো কৌশল):** কেউ ভুল করে একটা log ফাইল `rm` করেছে, কিন্তু process-টা এখনও সেটা খোলা রেখে লিখে যাচ্ছে। ডিস্কে জায়গাও খালি হচ্ছে না। ফাইলটা `/proc/<PID>/fd/`-এ `(deleted)` লেখা নিয়ে এখনও দেখা যাবে, আর সেখান থেকে কপি করে উদ্ধার করা যায়: `cp /proc/<PID>/fd/3 ~/recovered.log`।

> ⚠️ **সতর্কতা:** `/proc/<PID>/environ`-এ password বা API key থাকতে পারে। এজন্য অন্য user-এর process-এর `environ`, `fd`, `maps` শুধু root বা ওই user-ই পড়তে পারে।

process নিয়ে বাকি সব কিছু (`ps`, `kill`, signal) আছে [অধ্যায় ২৪](24-process-management.md)-এ।

---

## ৪৭.৫ `/sys` ও sysfs: ডিভাইসের সাজানো গাছ

### `/proc` থাকতে `/sys` কেন

শুরুতে kernel সব তথ্য `/proc`-এ রাখত, ফলে সেখানে process, হার্ডওয়্যার, প্যারামিটার সব মিলে একটা জগাখিচুড়ি তৈরি হয়েছিল। kernel 2.6-এ **sysfs** এলো (`/sys`-এ মাউন্ট হয়)। এর কাজ শুধু একটা: kernel-এর ভেতরের **device model** (ডিভাইস, bus, driver, তাদের সম্পর্ক) একটা সুশৃঙ্খল ডিরেক্টরি-গাছ হিসেবে দেখানো।

**নিয়ম:** sysfs-এ সাধারণত **এক ফাইলে একটাই মান** থাকে। তাই script দিয়ে পড়া-লেখা খুব সহজ।

**উপমা:** `/proc` যদি হাসপাতালের জরুরি বিভাগের হোয়াইটবোর্ড হয় (কোন রোগী কোথায়, কী অবস্থা), তবে `/sys` হলো হাসপাতালের বিল্ডিং-এর নকশা: কোন ফ্লোরে কোন ওয়ার্ড, কোন ওয়ার্ডে কোন বেড, কোন ডাক্তার দায়িত্বে।

```
/sys
├── devices/    <- আসল গাছ: প্রতিটি ডিভাইস তার physical অবস্থান অনুযায়ী
│     └── pci0000:00/0000:00:1f.6/net/enp0s31f6 ...
├── bus/        <- bus অনুযায়ী (pci, usb, i2c ...), প্রতিটিতে devices/ ও drivers/
├── class/      <- কাজ অনুযায়ী (net, block, tty, power_supply, backlight ...)
│     └── net/enp0s31f6  --> ../../devices/... (symlink)
├── block/      <- ডিস্ক (sda, nvme0n1) — symlink
├── module/     <- লোড হওয়া module ও তাদের parameters/
├── firmware/   <- ACPI, EFI, DMI তথ্য
├── kernel/     <- kernel-এর কিছু সেটিং (mm/transparent_hugepage ...)
├── fs/         <- filesystem-ভিত্তিক তথ্য, cgroup (fs/cgroup)
└── power/      <- suspend/hibernate নিয়ন্ত্রণ
```

`/sys/class` আর `/sys/block`-এর বেশিরভাগ জিনিস আসলে `/sys/devices`-এর দিকে symlink। একই ডিভাইসকে "কাজ" দিয়ে খুঁজে পেতে এই ব্যবস্থা।

### বাস্তব উদাহরণ

```
# নেটওয়ার্ক কার্ডের MAC address, অবস্থা, গতি
$ cat /sys/class/net/enp0s31f6/address
8c:16:45:aa:bb:cc
$ cat /sys/class/net/enp0s31f6/operstate
up
$ cat /sys/class/net/enp0s31f6/speed
1000

# ল্যাপটপের ব্যাটারি
$ cat /sys/class/power_supply/BAT0/capacity
87
$ cat /sys/class/power_supply/BAT0/status
Discharging

# ডিস্ক SSD নাকি HDD (0 = SSD, 1 = ঘূর্ণায়মান HDD)
$ cat /sys/block/nvme0n1/queue/rotational
0

# ডিস্কের I/O scheduler (বন্ধনীর ভেতরেরটা সক্রিয়)
$ cat /sys/block/sda/queue/scheduler
none [mq-deadline] kyber bfq

# তাপমাত্রা (মিলি-ডিগ্রি সেলসিয়াসে)
$ cat /sys/class/thermal/thermal_zone0/temp
48000

# কোন নেটওয়ার্ক কার্ড কোন driver ব্যবহার করছে
$ readlink /sys/class/net/enp0s31f6/device/driver
../../../../bus/pci/drivers/e1000e
```

### `/sys`-এ লেখা

কিছু ফাইল লেখার যোগ্য, এবং লিখলে সঙ্গে সঙ্গে আচরণ বদলায়:

```
# স্ক্রিনের উজ্জ্বলতা (মান হার্ডওয়্যার অনুযায়ী আলাদা)
$ cat /sys/class/backlight/intel_backlight/max_brightness
19393
$ echo 9000 | sudo tee /sys/class/backlight/intel_backlight/brightness

# ডিস্কের scheduler বদলানো
$ echo bfq | sudo tee /sys/block/sda/queue/scheduler
```

> ⚠️ **সতর্কতা:** `sudo echo 5000 > /sys/...` কাজ করবে না! কারণ redirection (`>`) তোমার সাধারণ shell করে, `sudo` নয়, তাই "Permission denied" আসবে। সবসময় `echo ... | sudo tee /path` ব্যবহার করো ([অধ্যায় ১৮](18-sudo-root.md) ও [অধ্যায় ১৯](19-redirection-pipe.md) দেখো)।

> 📝 **মনে রাখো:** `/proc` ও `/sys`-এ লেখা পরিবর্তন **reboot হলেই হারিয়ে যায়**, কারণ এগুলো RAM-এ থাকা kernel-এর অবস্থা। স্থায়ী করতে `sysctl.d`, `modprobe.d`, বা udev rule ব্যবহার করতে হয়।

### `/proc` বনাম `/sys` বনাম `/dev`

| | `/proc` | `/sys` | `/dev` |
|---|---|---|---|
| filesystem | procfs | sysfs | devtmpfs |
| কী আছে | process আর kernel-এর পরিসংখ্যান, `/proc/sys` প্যারামিটার | ডিভাইস, driver, bus-এর গঠন ও attribute | ডিভাইস **ফাইল** (যেটা দিয়ে ডেটা পড়া/লেখা হয়) |
| উদাহরণ | `/proc/meminfo` | `/sys/block/sda/size` | `/dev/sda` |
| কে তৈরি করে | kernel | kernel | kernel (devtmpfs), নাম/পারমিশন ঠিক করে udev |

---

## ৪৭.৬ `sysctl`: চলন্ত kernel-এর সেটিং বদলানো

### ধারণা

`/proc/sys/` ডিরেক্টরির নিচে শত শত ফাইল আছে, যেগুলো kernel-এর **tunable** (সমন্বয়যোগ্য) প্যারামিটার। `sysctl` কমান্ড এগুলো পড়ে ও লেখে। নামের মধ্যে `/`-এর বদলে `.` ব্যবহার হয়:

```
/proc/sys/net/ipv4/ip_forward    <==>    net.ipv4.ip_forward
/proc/sys/vm/swappiness          <==>    vm.swappiness
```

| উপসর্গ | বিষয় |
|---|---|
| `kernel.*` | kernel-এর সাধারণ আচরণ (hostname, panic, pid_max, sysrq) |
| `vm.*` | virtual memory (swap, dirty page, overcommit) |
| `net.*` | network stack (IPv4/IPv6, TCP, buffer) |
| `fs.*` | filesystem (file handle, inotify) |
| `dev.*` | ডিভাইস-নির্দিষ্ট |

### কমান্ড ও অপশন

| কমান্ড | কাজ |
|---|---|
| `sysctl <name>` | একটা মান পড়ো |
| `sysctl -n <name>` | শুধু মানটা (নাম ছাড়া), script-এর জন্য |
| `sysctl -a` | সব প্যারামিটার দেখাও |
| `sudo sysctl -w <name>=<value>` | মান বদলাও (সাময়িক) |
| `sudo sysctl -p [file]` | ফাইল থেকে লোড করো (ফাইল না দিলে `/etc/sysctl.conf`) |
| `sudo sysctl --system` | সব কনফিগ ডিরেক্টরি থেকে নিয়ম মেনে লোড করো |
| `sysctl -a -r <regex>` | pattern দিয়ে খোঁজো |

```
$ sysctl vm.swappiness
vm.swappiness = 60

$ sysctl -n net.ipv4.ip_forward
0

$ sudo sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1

$ cat /proc/sys/net/ipv4/ip_forward      # একই জিনিস
1

$ sudo sysctl -a 2>/dev/null | wc -l
1100                  <- সংখ্যাটা kernel ও লোড হওয়া module অনুযায়ী আলাদা হবে

$ sysctl -a -r 'tcp_.*keepalive' 2>/dev/null
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_time = 7200
```

> 💡 **টিপস:** `sysctl -a` সাধারণ user হিসেবে চালালে কিছু প্যারামিটার পড়ার অনুমতি না থাকায় "permission denied" বার্তা আসে। `2>/dev/null` দিয়ে সেগুলো লুকাও, অথবা `sudo` দাও।

### স্থায়ী করা: `/etc/sysctl.d/`

`sysctl -w` দিয়ে করা পরিবর্তন reboot-এ হারায়। স্থায়ী করতে `/etc/sysctl.d/`-এ একটা ফাইল বানাও:

```
$ sudo tee /etc/sysctl.d/99-custom.conf <<'EOF'
# রাউটার/VPN/Docker-এর জন্য প্যাকেট ফরওয়ার্ডিং
net.ipv4.ip_forward = 1
# RAM বেশি থাকলে swap কম ব্যবহার করো
vm.swappiness = 10
EOF

$ sudo sysctl --system
* Applying /usr/lib/sysctl.d/10-...conf ...
* Applying /etc/sysctl.d/99-custom.conf ...
net.ipv4.ip_forward = 1
vm.swappiness = 10
```

boot-এর সময় `systemd-sysctl.service` এই ফাইলগুলো পড়ে। পড়ার নিয়ম:

```
 /usr/lib/sysctl.d/*.conf    (প্যাকেজের ডিফল্ট)
 /run/sysctl.d/*.conf        (runtime)
 /etc/sysctl.d/*.conf        (administrator — তোমার)
        |
        v  সব ফাইল নাম অনুযায়ী (lexical order) সাজানো হয়।
           একই নামের ফাইল থাকলে /etc জেতে /run ও /usr/lib-এর ওপর।
           একই প্যারামিটার একাধিক ফাইলে থাকলে, পরে পড়া ফাইলের মান টেকে।
        v
 /etc/sysctl.conf            (পুরোনো ফাইল, --system সবশেষে এটাও পড়ে, থাকলে)
```

তাই ফাইলের নাম `99-...conf` দেওয়া হয়, যাতে এটা সবার শেষে পড়া হয় আর এর মানই কার্যকর থাকে।

> 📝 **মনে রাখো:** কিছু প্যারামিটার (যেমন `net.bridge.*`, `net.netfilter.*`) তখনই থাকে যখন সংশ্লিষ্ট module লোড থাকে। module লোড হওয়ার আগে sysctl চললে ওই সেটিং প্রয়োগ হবে না। সমাধান: module-টা `/etc/modules-load.d`-এ রাখো।

### গুরুত্বপূর্ণ sysctl প্যারামিটার

| প্যারামিটার | সাধারণ ডিফল্ট | কী করে / কখন বদলাও |
|---|---|---|
| `net.ipv4.ip_forward` | 0 | 1 করলে মেশিন রাউটারের মতো প্যাকেট ফরওয়ার্ড করে (NAT, VPN, container) |
| `net.ipv6.conf.all.forwarding` | 0 | IPv6-এর জন্য একই |
| `vm.swappiness` | 60 | কত আগ্রহ নিয়ে swap ব্যবহার করবে (0–200); ডেটাবেস সার্ভারে প্রায়ই 1–10 |
| `vm.dirty_ratio` / `vm.dirty_background_ratio` | 20 / 10 | RAM-এর কত শতাংশ "লেখার অপেক্ষায়" ডেটা জমবে |
| `vm.overcommit_memory` | 0 | 0 = heuristic, 1 = সবসময় হ্যাঁ, 2 = কড়া হিসাব |
| `vm.max_map_count` | distro অনুযায়ী | Elasticsearch/OpenSearch কমপক্ষে 262144 চায় |
| `fs.file-max` | অনেক বড় (RAM অনুযায়ী) | সারা সিস্টেমে সর্বোচ্চ খোলা ফাইল |
| `fs.inotify.max_user_watches` | distro অনুযায়ী | VS Code, webpack ইত্যাদি "too many watchers" এরর দিলে বাড়াও |
| `net.core.somaxconn` | 4096 | listen queue-এর সর্বোচ্চ দৈর্ঘ্য (ব্যস্ত ওয়েব সার্ভার) |
| `net.ipv4.ip_local_port_range` | `32768 60999` | বাইরে যাওয়া connection-এর জন্য port পরিসর |
| `net.ipv4.tcp_syncookies` | 1 | SYN flood আক্রমণ থেকে রক্ষা |
| `net.ipv4.conf.all.rp_filter` | distro অনুযায়ী (০/১/২) | ভুয়া source IP-এর প্যাকেট বাদ দেওয়া |
| `net.ipv4.icmp_echo_ignore_all` | 0 | 1 করলে `ping`-এর উত্তর দেবে না |
| `kernel.hostname` | | hostname (স্থায়ী করতে `hostnamectl` ব্যবহার করো) |
| `kernel.pid_max` | 4194304 (64-bit-এ সাধারণত) | সর্বোচ্চ PID সংখ্যা |
| `kernel.panic` | 0 | panic হলে কত সেকেন্ড পর নিজে reboot (0 = আটকে থাকবে) |
| `kernel.sysrq` | distro অনুযায়ী (Ubuntu 176, Fedora 16) | Magic SysRq কোন কোন কাজে অনুমোদিত (bitmask) |
| `kernel.dmesg_restrict` | Ubuntu-তে 1 | 1 হলে সাধারণ user `dmesg` পড়তে পারবে না |
| `kernel.kptr_restrict` | 1 | kernel পয়েন্টার ঠিকানা লুকানো (নিরাপত্তা) |

> ⚠️ **সতর্কতা:** ইন্টারনেট থেকে "Linux performance tuning sysctl" লিখে পাওয়া লম্বা তালিকা অন্ধভাবে কপি কোরো না। আধুনিক kernel-এর ডিফল্টগুলো বেশিরভাগ ক্ষেত্রেই যথেষ্ট ভালো। প্রতিটা পরিবর্তন একটা একটা করে মাপো (দেখো [অধ্যায় ৪৬](46-performance-troubleshooting.md)), আর কেন বদলালে সেটা ফাইলে মন্তব্য করে লিখে রাখো।

নিরাপত্তা-সম্পর্কিত sysctl (যেমন `kernel.randomize_va_space`, `net.ipv4.conf.all.accept_redirects`) নিয়ে বিস্তারিত আছে [অধ্যায় ৪৫](45-security-hardening.md)-এ।

---

## ৪৭.৭ udev: ডিভাইস ম্যানেজার ও ডিভাইস নিয়ম

### সমস্যাটা কী

তুমি একটা pen drive লাগালে সেটা `/dev/sdb` হয়ে যায়। পরের বার আগে আরেকটা লাগানো থাকলে এটাই হয়ে যায় `/dev/sdc`। তাহলে script লিখবে কীভাবে? আর `/dev/sdb` ফাইলটা কে বানায়, তার owner আর পারমিশন কে ঠিক করে? এই কাজগুলো করে **udev**, যেটা আধুনিক সিস্টেমে `systemd-udevd` নামের একটা daemon (পটভূমিতে চলা সার্ভিস) হিসেবে চলে।

**উপমা:** অফিসের রিসেপশনিস্ট। নতুন কেউ এলে (ডিভাইস প্লাগ) নিরাপত্তারক্ষী (kernel) রিসেপশনে খবর পাঠায়। রিসেপশনিস্ট (udev) নিয়মের খাতা (rules) দেখে ঠিক করে: তাকে কোন রুমে বসাতে হবে (`/dev`-এ নাম), কে কে তার সাথে দেখা করতে পারবে (owner/group/mode), কাকে খবর দিতে হবে (`RUN`)।

```
  ডিভাইস প্লাগ/আনপ্লাগ
          |
          v
  KERNEL: driver ডিভাইস চিনল ---> /sys-এ নতুন এন্ট্রি
          |                     ---> devtmpfs-এ /dev/sdb তৈরি (মৌলিক)
          | uevent (netlink socket দিয়ে) — ACTION=add, DEVPATH=..., SUBSYSTEM=block
          v
  systemd-udevd
     | 1. /sys থেকে attribute পড়ে (vendor, model, serial ...)
     | 2. rules ফাইলগুলো ক্রমানুসারে মেলায়
     | 3. ফল: owner/group/mode বসায়, symlink বানায়
     |       (/dev/disk/by-id/..., /dev/disk/by-uuid/...)
     |       network interface-এর নাম দেয় (enp3s0), দরকার হলে RUN চালায়
     v
  অন্যরা খবর পায়: systemd (.device unit), udisks2 (ডেস্কটপে auto-mount) ...
```

এজন্যই `/dev/disk/by-uuid/` বা `/dev/disk/by-id/` দিয়ে ডিস্ক খুঁজলে নাম কখনো বদলায় না ([অধ্যায় ৩২](32-disk-partition-mount.md)-এ fstab-এ UUID ব্যবহারের কারণ এটাই)। আর তোমার নেটওয়ার্ক কার্ডের নাম `eth0` না হয়ে `enp3s0` হয় udev-এর **predictable network interface names** নিয়মের কারণে: `en` = ethernet, `p3` = PCI bus 3, `s0` = slot 0। Wi-Fi হলে শুরু হয় `wl` দিয়ে, যেমন `wlp2s0`।

### rules ফাইল কোথায় থাকে

| ডিরেক্টরি | কার জন্য |
|---|---|
| `/usr/lib/udev/rules.d/` | প্যাকেজের নিয়ম (এগুলো এডিট কোরো না) |
| `/etc/udev/rules.d/` | তোমার নিজের নিয়ম; একই নামের ফাইল থাকলে এটাই জেতে |
| `/run/udev/rules.d/` | runtime, অস্থায়ী |

সব ফাইল নাম অনুযায়ী সাজিয়ে পড়া হয়। ফাইলের নাম `NN-description.rules` আকারে হয়, যেমন `99-myusb.rules`।

### `udevadm`: udev-এর সব কাজের টুল

| কমান্ড | কাজ |
|---|---|
| `udevadm info -n /dev/sdb` | একটা ডিভাইসের udev তথ্য |
| `udevadm info -q property -n /dev/sdb` | শুধু property (`ENV`) গুলো |
| `udevadm info -a -n /dev/sdb` | ডিভাইস আর তার সব parent-এর attribute; rule লেখার জন্য সবচেয়ে কাজের |
| `udevadm monitor` | লাইভ event দেখো (প্লাগ/আনপ্লাগ করে দেখো) |
| `udevadm monitor --udev --property` | udev event, property সহ |
| `sudo udevadm control --reload` | rule ফাইল আবার পড়াও (পুরোনো নাম `--reload-rules`) |
| `sudo udevadm trigger` | বিদ্যমান ডিভাইসের জন্য আবার event পাঠাও, যাতে নতুন rule প্রয়োগ হয় |
| `udevadm test /sys/class/...` | rule আসলে কীভাবে মেলে, simulation করে দেখাও |
| `udevadm settle` | সব event প্রক্রিয়া হওয়া পর্যন্ত অপেক্ষা করো (script-এ) |

```
$ udevadm info -q property -n /dev/sdb | grep -E 'ID_VENDOR=|ID_MODEL=|ID_SERIAL_SHORT|ID_BUS'
ID_BUS=usb
ID_MODEL=Cruzer_Blade
ID_SERIAL_SHORT=4C530001230808113264
ID_VENDOR=SanDisk

$ udevadm monitor --udev
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing

UDEV  [12345.678901] add      /devices/pci0000:00/0000:00:14.0/usb1/1-2 (usb)
UDEV  [12345.701234] add      /devices/.../block/sdb (block)
UDEV  [12345.712345] add      /devices/.../block/sdb/sdb1 (block)
^C
```

### rule-এর ভাষা

প্রতিটি rule **এক লাইনে**, কমা দিয়ে আলাদা করা `key operator value` জোড়া। কিছু key **মেলানোর** জন্য (match), কিছু **বসানোর** জন্য (assign)।

| operator | ধরন | মানে |
|---|---|---|
| `==` | match | সমান হলে |
| `!=` | match | সমান না হলে |
| `=` | assign | মান বসাও (আগেরটা মুছে) |
| `+=` | assign | তালিকায় যোগ করো |
| `-=` | assign | তালিকা থেকে সরাও |
| `:=` | assign | চূড়ান্ত মান বসাও; পরের rule আর বদলাতে পারবে না |

| key | ধরন | মানে |
|---|---|---|
| `ACTION` | match | `add`, `remove`, `change`, `bind` … |
| `KERNEL` | match | kernel-এর দেওয়া নাম (`sd*`, `ttyUSB*`) |
| `SUBSYSTEM` | match | `block`, `net`, `tty`, `usb` … |
| `ATTR{file}` | match | এই ডিভাইসের sysfs attribute |
| `ATTRS{file}` | match | এই ডিভাইস **বা তার যেকোনো parent**-এর attribute |
| `ENV{KEY}` | match/assign | property |
| `SYMLINK` | assign | `/dev`-এ অতিরিক্ত symlink |
| `OWNER`, `GROUP`, `MODE` | assign | মালিক, গ্রুপ, পারমিশন |
| `NAME` | assign | শুধু network interface-এর নাম বদলানো যায় |
| `TAG+="uaccess"` | assign | ডেস্কটপে বসে থাকা (logged-in) user-কে অ্যাক্সেস দেওয়া |
| `RUN+=` | assign | event শেষে একটা প্রোগ্রাম চালাও (পুরো path) |

`%k` (kernel নাম), `%n` (kernel নম্বর) এর মতো substitution-ও ব্যবহার করা যায়।

### উদাহরণ ১: Arduino/USB-serial-কে স্থায়ী নাম দেওয়া

প্রথমে vendor ও product ID খোঁজো:

```
$ lsusb | grep -i ftdi
Bus 001 Device 007: ID 0403:6001 Future Technology Devices International, Ltd FT232 Serial (UART) IC
```

তারপর rule লেখো:

```
# /etc/udev/rules.d/99-arduino.rules
SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", ATTRS{idProduct}=="6001", SYMLINK+="arduino", MODE="0660", GROUP="dialout"
```

```
$ sudo udevadm control --reload
$ sudo udevadm trigger --subsystem-match=tty
$ ls -l /dev/arduino
lrwxrwxrwx 1 root root 7 Nov 12 11:00 /dev/arduino -> ttyUSB0
```

এখন ডিভাইস `ttyUSB0` হোক বা `ttyUSB3`, তোমার প্রোগ্রাম সবসময় `/dev/arduino` ব্যবহার করতে পারবে। আর তুমি `dialout` গ্রুপে থাকলে `sudo` ছাড়াই অ্যাক্সেস পাবে।

### উদাহরণ ২: নির্দিষ্ট pen drive লাগালে log লেখা

```
# /etc/udev/rules.d/99-backup-disk.rules
ACTION=="add", SUBSYSTEM=="block", ENV{ID_SERIAL_SHORT}=="4C530001230808113264", ENV{DEVTYPE}=="partition", RUN+="/usr/bin/logger 'backup disk %k plugged'"
```

> ⚠️ **সতর্কতা:** `RUN` দিয়ে চালানো প্রোগ্রাম খুব ছোট হতে হবে। udev অল্প সময় পর সেটাকে মেরে ফেলে, আর ভেতরে নেটওয়ার্ক বা mount ঠিকমতো কাজ নাও করতে পারে। লম্বা কাজের জন্য (যেমন ব্যাকআপ চালানো) `RUN` থেকে সরাসরি না চালিয়ে systemd service চালু করো: `ENV{SYSTEMD_WANTS}+="mybackup.service"` আর সাথে `TAG+="systemd"`।

> 💡 **টিপস:** rule কাজ না করলে, প্রথমে `udevadm info -a`-তে দেখো তুমি যে attribute মেলাচ্ছ সেটা আসলে আছে কিনা। মনে রাখবে, একটা rule-এ `ATTRS{}` শুধু **একটাই** parent ডিভাইস থেকে আসতে পারে, অনেকগুলো parent মিশিয়ে মেলানো যায় না। তারপর `udevadm test $(udevadm info -q path -n /dev/sdb)` চালাও।

---

## ৪৭.৮ হার্ডওয়্যারের তথ্য বের করা

নতুন সার্ভার হাতে পেলে, বা কোনো driver কাজ না করলে, সবার আগে জানতে হয় ভেতরে কী আছে। নিচের টুলগুলো এই তথ্য `/proc`, `/sys` ও firmware টেবিল থেকে পড়ে সাজিয়ে দেখায়।

| কমান্ড | কী দেখায় | প্যাকেজ | root লাগে? |
|---|---|---|---|
| `lscpu` | CPU-র গঠন | util-linux (সব জায়গায় থাকে) | না |
| `lspci` | PCI/PCIe ডিভাইস (GPU, NIC, NVMe) | pciutils | বেশিরভাগ না; বিস্তারিত (`-vv`) দেখতে হ্যাঁ |
| `lsusb` | USB ডিভাইস | usbutils | না |
| `lsblk` | ডিস্ক ও পার্টিশন | util-linux | না |
| `dmidecode` | BIOS/মাদারবোর্ড/RAM স্লট/সিরিয়াল নম্বর | dmidecode | **হ্যাঁ** |
| `lshw` | সব হার্ডওয়্যারের পূর্ণ তালিকা | lshw | পূর্ণ তথ্যের জন্য হ্যাঁ |
| `hwinfo` | বিস্তারিত (openSUSE-র টুল) | hwinfo | হ্যাঁ |
| `inxi` | সুন্দর সারাংশ, ফোরামে শেয়ার করার জন্য | inxi | না |
| `sensors` | তাপমাত্রা, ফ্যানের গতি | lm-sensors | না |
| `free`, `nproc` | RAM, CPU সংখ্যা | procps / coreutils | না |

### `lscpu`

```
$ lscpu
Architecture:             x86_64
  CPU op-mode(s):         32-bit, 64-bit
  Byte Order:             Little Endian
CPU(s):                   8
  On-line CPU(s) list:    0-7
Vendor ID:                GenuineIntel
  Model name:             Intel(R) Core(TM) i7-10510U CPU @ 1.80GHz
    Thread(s) per core:   2
    Core(s) per socket:   4
    Socket(s):            1
Virtualization features:
  Virtualization:         VT-x
Caches (sum of all):
  L1d:                    128 KiB (4 instances)
  L2:                     1 MiB (4 instances)
  L3:                     8 MiB (1 instance)
NUMA:
  NUMA node(s):           1
...
```

হিসাবটা বুঝে নাও: মোট logical CPU = Socket × Core per socket × Thread per core = 1 × 4 × 2 = **8**।

| অপশন | কাজ |
|---|---|
| `-e` | প্রতিটি CPU এক লাইনে (টেবিল আকারে) |
| `-J` | JSON আউটপুট |
| `--caches` | cache-এর বিস্তারিত |

### `lspci`

```
$ lspci
00:02.0 VGA compatible controller: Intel Corporation CometLake-U GT2 [UHD Graphics] (rev 02)
00:14.3 Network controller: Intel Corporation Comet Lake PCH-LP CNVi WiFi
00:1f.6 Ethernet controller: Intel Corporation Ethernet Connection (10) I219-V
02:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd NVMe SSD Controller SM981/PM981/PM983

$ lspci -k -s 00:1f.6
00:1f.6 Ethernet controller: Intel Corporation Ethernet Connection (10) I219-V
	Subsystem: Lenovo Device 2292
	Kernel driver in use: e1000e
	Kernel modules: e1000e
```

| অপশন | কাজ |
|---|---|
| `-k` | কোন kernel driver ব্যবহার হচ্ছে, কোন module সম্ভব; **troubleshooting-এর সেরা অপশন** |
| `-v`, `-vv` | বিস্তারিত |
| `-nn` | নামের সাথে সংখ্যাসূচক vendor:device ID (`[8086:15bc]`) |
| `-s <slot>` | নির্দিষ্ট একটা ডিভাইস |
| `-t` | গাছের আকারে |
| `-d <vendor>:<device>` | ID দিয়ে খোঁজো |

> 💡 **টিপস:** কোনো ডিভাইসের পাশে `Kernel driver in use` লাইনটা না থাকলে বুঝবে driver লোড হয়নি। তখন `lspci -nn` থেকে ID নিয়ে ইন্টারনেটে খোঁজো কোন module লাগবে। নামের ডেটাবেস পুরোনো হলে আপডেট করো: `sudo update-pciids`।

### `lsusb`

```
$ lsusb
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 003: ID 8087:0026 Intel Corp. AX201 Bluetooth
Bus 001 Device 004: ID 046d:c52b Logitech, Inc. Unifying Receiver
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub

$ lsusb -t
/:  Bus 001.Port 001: Dev 001, Class=root_hub, Driver=xhci_hcd/12p, 480M
    |__ Port 001: Dev 004, If 0, Class=Human Interface Device, Driver=usbhid, 12M
    ...
```

| অপশন | কাজ |
|---|---|
| `-t` | গাছ আকারে, driver ও গতি সহ |
| `-v` | বিস্তারিত descriptor |
| `-d vendor:product` | নির্দিষ্ট ডিভাইস |
| `-s bus:dev` | bus/device নম্বর দিয়ে |

### `dmidecode`

এটা BIOS/UEFI firmware-এর **SMBIOS/DMI** টেবিল পড়ে। এখানে এমন সব তথ্য থাকে যা kernel নিজে জানে না: মেশিনের সিরিয়াল নম্বর, RAM কোন স্লটে কত GB, সর্বোচ্চ কত RAM লাগানো যাবে।

```
$ sudo dmidecode -s system-manufacturer
LENOVO
$ sudo dmidecode -s system-product-name
20U9CTO1WW
$ sudo dmidecode -s bios-version
N2WET36W (1.26 )

$ sudo dmidecode -t memory | grep -E 'Size|Locator|Speed|Type:' | head
	Size: 8 GB
	Locator: ChannelA-DIMM0
	Type: DDR4
	Speed: 2667 MT/s
	Size: 8 GB
	Locator: ChannelB-DIMM0
	...
```

| অপশন | কাজ |
|---|---|
| `-t <type>` | নির্দিষ্ট ধরন: `bios`, `system`, `baseboard`, `chassis`, `processor`, `memory`, `cache`, `slot` |
| `-s <keyword>` | একটা মান: `system-serial-number`, `system-uuid`, `bios-version`, `baseboard-product-name` … |
| `-q` | কম কথা (quiet) |

> 📝 **মনে রাখো:** virtual machine-এ `dmidecode` hypervisor-এর বানানো তথ্য দেখায় (যেমন `QEMU`, `VMware`)। কোন hypervisor-এ চলছ সেটা জানার আরেকটা সহজ উপায় `systemd-detect-virt`।

### `lshw`

```
$ sudo lshw -short
H/W path          Device      Class          Description
=========================================================
                              system         20U9CTO1WW (LENOVO_MT_20U9)
/0                            bus            20U9CTO1WW
/0/0                          memory         128KiB BIOS
/0/4                          processor      Intel(R) Core(TM) i7-10510U
/0/2c                         memory         16GiB System Memory
/0/100/1f.6       enp0s31f6   network        Ethernet Connection (10) I219-V
/0/100/1d/0       /dev/nvme0  storage        Samsung SSD 970 EVO Plus 500GB
...
```

| অপশন | কাজ |
|---|---|
| `-short` | সংক্ষিপ্ত টেবিল |
| `-class <cls>` (`-c`) | একটা শ্রেণি: `network`, `disk`, `memory`, `display`, `processor` |
| `-businfo` | bus ঠিকানা সহ |
| `-html`, `-json`, `-xml` | রিপোর্ট ফাইল বানানোর জন্য |
| `-sanitize` | সিরিয়াল নম্বর, IP ইত্যাদি লুকাও (শেয়ার করার আগে) |

```
$ sudo lshw -class network -short
H/W path         Device      Class      Description
===================================================
/0/100/14.3      wlp0s20f3   network    Comet Lake PCH-LP CNVi WiFi
/0/100/1f.6      enp0s31f6   network    Ethernet Connection (10) I219-V
```

### অন্যান্য

```
$ inxi -Fxz          # সব কিছুর সারাংশ; -z ব্যক্তিগত তথ্য (MAC, IP) লুকায়
$ sensors            # প্রথমবার: sudo sensors-detect
$ lsblk -o NAME,SIZE,TYPE,MODEL,ROTA
$ cat /sys/class/dmi/id/product_name     # dmidecode-এর কিছু তথ্য root ছাড়াই
```

---

## ৪৭.৯ kernel আপডেট ও একাধিক kernel

### আপডেটের সময় কী হয়

বেশিরভাগ প্যাকেজ আপডেট হলে পুরোনো version মুছে নতুনটা বসে। কিন্তু kernel আলাদা: **নতুন kernel পুরোনোটার পাশে আলাদাভাবে ইনস্টল হয়।** কারণ নতুন kernel যদি তোমার মেশিনে boot না করে, তুমি যেন পুরোনোটায় ফিরে যেতে পারো। এটা একটা নিরাপত্তা-জাল।

```
 apt/dnf upgrade
     |
     +--> /boot/vmlinuz-<নতুন>              (নতুন kernel image)
     +--> /lib/modules/<নতুন>/              (নতুন module সেট)
     +--> initramfs তৈরি (update-initramfs / dracut)
     +--> DKMS module গুলো নতুন kernel-এর জন্য build (৪৭.১০)
     +--> bootloader এন্ট্রি আপডেট (update-grub / BLS entry)
     |
     v
 পুরোনো kernel এখনও চলছে!  -->  reboot  -->  নতুন kernel চালু
```

> 📝 **মনে রাখো:** kernel আপডেটের পর **reboot না করা পর্যন্ত নতুন kernel চলে না।** `uname -r` দেখলে পুরোনো version-ই দেখাবে। Ubuntu-তে reboot দরকার হলে `/var/run/reboot-required` ফাইলটা তৈরি হয়, আর `needrestart` টুলও সতর্ক করে।

### Debian / Ubuntu

```
# ইনস্টল করা kernel-এর তালিকা
$ dpkg --list 'linux-image*' | grep ^ii
ii  linux-image-6.8.0-41-generic  6.8.0-41.41  amd64  Signed kernel image generic
ii  linux-image-6.8.0-45-generic  6.8.0-45.45  amd64  Signed kernel image generic
ii  linux-image-generic           6.8.0-45.45  amd64  Generic Linux kernel image

# reboot দরকার কিনা
$ cat /var/run/reboot-required 2>/dev/null
*** System restart required ***

# পুরোনো অপ্রয়োজনীয় kernel সরানো (চলমান ও তার আগেরটা সাধারণত রেখে দেয়)
$ sudo apt autoremove --purge
```

- `linux-image-generic` (Ubuntu) বা `linux-image-amd64` (Debian) হলো **metapackage**। এটা নিজে কিছু না, শুধু "সর্বশেষ kernel" প্যাকেজের ওপর নির্ভর করে, তাই আপডেটে নতুন kernel টেনে আনে।
- Ubuntu LTS-এ নতুন হার্ডওয়্যারের জন্য **HWE (Hardware Enablement)** kernel পাওয়া যায়: `linux-generic-hwe-24.04`। Debian-এ নতুন kernel আসে `backports` repository থেকে।
- kernel আপডেট আপাতত আটকাতে: `sudo apt-mark hold linux-image-generic linux-headers-generic`।

### Fedora / RHEL

```
$ rpm -q kernel
kernel-5.14.0-427.13.1.el9_4.x86_64
kernel-5.14.0-427.18.1.el9_4.x86_64

$ sudo grubby --default-kernel
/boot/vmlinuz-5.14.0-427.18.1.el9_4.x86_64

$ sudo grubby --info=ALL | grep -E '^(index|kernel|title)'

# নির্দিষ্ট kernel-কে ডিফল্ট করো
$ sudo grubby --set-default /boot/vmlinuz-5.14.0-427.13.1.el9_4.x86_64
```

- কতগুলো kernel রাখা হবে সেটা ঠিক করে `/etc/dnf/dnf.conf`-এর `installonly_limit` (ডিফল্ট `3`)। সংখ্যা পার হলে সবচেয়ে পুরোনোটা নিজে থেকেই মুছে যায়।
- Fedora/RHEL-এ bootloader এন্ট্রিগুলো `/boot/loader/entries/*.conf`-এ আলাদা ফাইল হিসেবে থাকে (BootLoaderSpec বা BLS), আর `grubby` দিয়ে সেগুলো সামলানো হয়।
- সব kernel-এ একটা boot প্যারামিটার যোগ করতে: `sudo grubby --update-kernel=ALL --args="mitigations=auto"`। Debian/Ubuntu-তে একই কাজ হয় `/etc/default/grub`-এর `GRUB_CMDLINE_LINUX_DEFAULT` এডিট করে, তারপর `sudo update-grub` চালিয়ে।

### পুরোনো kernel দিয়ে boot করা

নতুন kernel-এ সমস্যা হলে (Wi-Fi কাজ করছে না, কালো স্ক্রিন):

1. reboot করো।
2. GRUB মেনু দেখাও। Ubuntu সাধারণত মেনু লুকিয়ে রাখে। UEFI মেশিনে boot শুরু হতেই **Esc** বারবার চাপো; পুরোনো BIOS মেশিনে **Shift** চেপে ধরে রাখো।
3. **Advanced options for Ubuntu** বেছে নাও, তারপর পুরোনো version-টা।
4. পুরোনোটায় সব ঠিক থাকলে সমস্যার রিপোর্ট করো, নতুন আপডেটের জন্য অপেক্ষা করো, অথবা পুরোনোটাকে ডিফল্ট বানাও।

### reboot ছাড়া patch: livepatch

সার্ভার reboot করা সবসময় সম্ভব হয় না। তাই গুরুতর security fix চলন্ত kernel-এ সরাসরি বসানোর ব্যবস্থা আছে: Ubuntu-তে **Canonical Livepatch** (Ubuntu Pro দিয়ে, `sudo pro enable livepatch`), RHEL-এ **kpatch**। তবে এগুলো সাময়িক সমাধান; শেষ পর্যন্ত reboot করে নতুন kernel-এ যেতেই হবে।

> ⚠️ **সতর্কতা:** যে kernel-এ এখন চলছ (`uname -r`), সেটা কখনো মুছবে না। আর `/boot` পার্টিশন ছোট হলে অনেকগুলো kernel জমে সেটা ভরে যায়, তখন আপডেট মাঝপথে ব্যর্থ হয়। নিয়মিত `df -h /boot` দেখো।

---

## ৪৭.১০ DKMS: kernel বদলালেও driver টিকে থাকে

### সমস্যা

kernel module একটা নির্দিষ্ট kernel version-এর জন্য compile করা হয় (মনে আছে `modinfo`-র `vermagic`?)। তুমি NVIDIA, VirtualBox বা কোনো Wi-Fi কার্ডের **out-of-tree** module (যেটা অফিশিয়াল kernel source-এর অংশ নয়) হাতে compile করে ইনস্টল করলে, পরের kernel আপডেটের পর সেটা আর লোড হবে না। নতুন kernel-এর `/lib/modules/<নতুন>/`-এ সেটা নেই-ই।

### সমাধান: DKMS

**DKMS (Dynamic Kernel Module Support)** module-এর **source code** নিজের কাছে রেখে দেয়। যখনই নতুন kernel ইনস্টল হয়, প্যাকেজ ম্যানেজারের একটা hook DKMS-কে ডাকে, আর DKMS নতুন kernel-এর জন্য module-টা আবার compile করে ইনস্টল করে দেয়।

**উপমা:** দর্জির কাছে তোমার শার্টের মাপ আর কাপড়ের নকশা রাখা আছে। তুমি একটু মোটা হলে (kernel আপডেট) দর্জি নিজেই নতুন মাপে শার্ট সেলাই করে দেয়; তোমাকে আবার নকশা নিয়ে যেতে হয় না।

```
 /usr/src/v4l2loopback-0.12.7/      <- source + dkms.conf
            |
            |  dkms add      (DKMS-এর খাতায় নাম তোলা)
            v
 /var/lib/dkms/v4l2loopback/0.12.7/
            |
            |  dkms build -k <kernel>   (kernel headers লাগে)
            v
   .ko তৈরি
            |
            |  dkms install -k <kernel>
            v
 /lib/modules/<kernel>/updates/dkms/v4l2loopback.ko   + depmod
            ^
            |  নতুন kernel ইনস্টল হলে: dkms autoinstall (স্বয়ংক্রিয়)
```

**পূর্বশর্ত:** kernel-এর **headers** ইনস্টল থাকতে হবে:

```
# Debian / Ubuntu
$ sudo apt install dkms linux-headers-$(uname -r)
# Fedora / RHEL (RHEL-এ dkms আসে EPEL থেকে)
$ sudo dnf install dkms kernel-devel kernel-headers
```

### DKMS কমান্ড

| কমান্ড | কাজ |
|---|---|
| `dkms status` | কোন module কোন kernel-এর জন্য কী অবস্থায় আছে |
| `sudo dkms add -m <name> -v <ver>` | `/usr/src/<name>-<ver>` থেকে নিবন্ধন |
| `sudo dkms build -m <name> -v <ver> [-k <kernel>]` | compile |
| `sudo dkms install -m <name> -v <ver> [-k <kernel>]` | ইনস্টল (দরকার হলে build-ও করে) |
| `sudo dkms remove -m <name> -v <ver> --all` | সব kernel থেকে সরাও |
| `sudo dkms autoinstall [-k <kernel>]` | সব নিবন্ধিত module একসাথে build ও install |

বাস্তবে বেশিরভাগ সময় তুমি নিজে এগুলো চালাবে না; `*-dkms` নামের প্যাকেজ ইনস্টল করলেই সব হয়ে যায়:

```
$ sudo apt install v4l2loopback-dkms
...
Building for 6.8.0-45-generic
Building initial module for 6.8.0-45-generic
Done.

$ dkms status
v4l2loopback/0.12.7, 6.8.0-45-generic, x86_64: installed
```

একটা সাধারণ `dkms.conf` দেখতে এমন:

```
PACKAGE_NAME="hello"
PACKAGE_VERSION="1.0"
BUILT_MODULE_NAME[0]="hello"
DEST_MODULE_LOCATION[0]="/updates/dkms"
AUTOINSTALL="yes"
```

### Secure Boot ও module signing

UEFI **Secure Boot** চালু থাকলে kernel শুধু বিশ্বস্ত key দিয়ে **signed** module লোড করে। না হলে এরর আসে:

```
$ sudo modprobe hello
modprobe: ERROR: could not insert 'hello': Key was rejected by service
```

```
$ mokutil --sb-state
SecureBoot enabled
```

Ubuntu-তে DKMS নিজের একটা key (MOK, Machine Owner Key) দিয়ে module sign করে। প্রথমবার সেই key-টা firmware-এ নিবন্ধন করতে হয়: `sudo mokutil --import /var/lib/shim-signed/mok/MOK.der` চালিয়ে একটা পাসওয়ার্ড দাও, তারপর reboot করলে নীল রঙের **MOK Manager** স্ক্রিনে "Enroll MOK" বেছে সেই পাসওয়ার্ড দাও। Fedora/RHEL-এ key-এর অবস্থান আলাদা (DKMS-এর কনফিগে উল্লেখ থাকে, সাধারণত `/var/lib/dkms/mok.pub`)।

> 💡 **টিপস:** kernel আপডেটের পর NVIDIA বা VirtualBox কাজ না করলে আগে `dkms status` দেখো। নতুন kernel-এর পাশে `installed` না থাকলে headers আছে কিনা দেখে নাও, তারপর `sudo dkms autoinstall`। build ব্যর্থ হলে log থাকে `/var/lib/dkms/<name>/<ver>/build/make.log`-এ।

সোর্স থেকে সফটওয়্যার বানানোর সাধারণ নিয়ম আছে [অধ্যায় ৩১](31-compile-from-source.md)-এ।

---

## ৪৭.১১ kernel panic, oops ও জরুরি অবস্থা

### oops বনাম panic

| | **Oops** | **Panic** |
|---|---|---|
| কী | kernel একটা ভুল ধরেছে (যেমন NULL pointer), কিন্তু চালিয়ে যেতে পারছে | kernel এমন অবস্থায় যে আর নিরাপদে চালানো সম্ভব নয় |
| ফল | যে process ভুলটা ঘটিয়েছে সেটা মারা যায়; সিস্টেম অস্থির থাকতে পারে | সব থেমে যায়। স্ক্রিনে বার্তা, কীবোর্ডের LED জ্বলা-নেভা |
| কোথায় দেখবে | `dmesg`, `journalctl -k` | স্ক্রিনে (বা kdump/serial console থাকলে সেখানে) |

**উপমা:** oops হলো রান্নাঘরে একটা হাঁড়ি পড়ে যাওয়া: পরিষ্কার করে কাজ চালানো যায়। panic হলো রান্নাঘরে আগুন: সবকিছু বন্ধ করে বেরিয়ে যেতে হয়, কারণ চালাতে থাকলে আরও ক্ষতি হবে (যেমন ডিস্কের ডেটা নষ্ট হয়ে যেতে পারে)।

### সাধারণ panic বার্তা ও কারণ

```
Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)
```

এর মানে kernel root ফাইলসিস্টেম খুঁজে পায়নি। কারণ সাধারণত initramfs নেই বা নষ্ট, অথবা `root=` প্যারামিটার ভুল। সমাধান: GRUB থেকে পুরোনো kernel-এ boot করে initramfs আবার বানাও (`sudo update-initramfs -u -k <ver>` বা `sudo dracut -f --kver <ver>`)।

```
Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000100
```

এর মানে PID 1 (systemd) মারা গেছে। কারণ হতে পারে নষ্ট root ফাইলসিস্টেম, ভাঙা glibc, বা ভুল `init=` প্যারামিটার।

অন্যান্য কারণের মধ্যে আছে নষ্ট RAM (`memtest86+` দিয়ে পরীক্ষা করো), অতিরিক্ত গরম হওয়া, buggy driver/module (বিশেষ করে proprietary ও out-of-tree), আর `vm.panic_on_oom=1` থাকা অবস্থায় মেমরি শেষ হয়ে যাওয়া।

### tainted kernel

proprietary বা unsigned module লোড হলে kernel নিজেকে "tainted" (দাগী) চিহ্নিত করে। এতে চলায় কোনো সমস্যা হয় না; শুধু bug রিপোর্টে kernel ডেভেলপাররা বুঝতে পারেন যে অজানা কোড যুক্ত ছিল।

```
$ cat /proc/sys/kernel/tainted
0              <- 0 মানে পরিষ্কার

$ cat /proc/sys/kernel/tainted
12289          <- bitmask: 1 (P, proprietary) + 4096 (O, out-of-tree) + 8192 (E, unsigned)
```

### panic-এর পর তথ্য সংগ্রহ

panic-এর মুহূর্তে ডিস্কে লেখা নিরাপদ নয়, তাই বার্তা প্রায়ই log-এ পৌঁছায় না। উপায়গুলো:

| উপায় | কীভাবে |
|---|---|
| আগের boot-এর kernel log | `journalctl -k -b -1` (persistent journal চালু থাকলে, দেখো [অধ্যায় ২৮](28-logs.md)) |
| pstore | কিছু firmware panic-এর বার্তা সংরক্ষণ করে রাখে, পরের boot-এ `/sys/fs/pstore/`-এ দেখা যায় |
| **kdump** | panic হলেই আগে থেকে লোড করা একটা দ্বিতীয় "crash kernel" চালু হয়, সে পুরো মেমরি ডাম্প করে ডিস্কে লিখে রাখে। Ubuntu: `sudo apt install kdump-tools` (বা `linux-crashdump`); RHEL: `kexec-tools`, `kdumpctl status`। ডাম্প জমা হয় `/var/crash/`-এ, বিশ্লেষণ করা হয় `crash` টুল দিয়ে |
| serial/netconsole | সার্ভারে kernel বার্তা অন্য মেশিনে পাঠানো |
| স্ক্রিনের ছবি | বার্তার প্রথম কয়েক লাইন সবচেয়ে গুরুত্বপূর্ণ, ফোনে একটা ছবি তুলে রাখো |

সার্ভারে panic হলে মেশিন আটকে না থেকে যেন নিজেই reboot করে:

```
$ echo 'kernel.panic = 10' | sudo tee /etc/sysctl.d/90-panic.conf
$ sudo sysctl --system
```

### `dmesg`: kernel-এর রিং বাফার

kernel-এর নিজের বার্তা (driver লোড, ডিভাইস প্লাগ, এরর, oops) একটা **ring buffer**-এ জমা হয়: মেমরিতে একটা নির্দিষ্ট মাপের চক্রাকার জায়গা, ভরে গেলে সবচেয়ে পুরোনো বার্তা মুছে যায়।

| কমান্ড | কাজ |
|---|---|
| `sudo dmesg` | সব বার্তা (Ubuntu-তে `dmesg_restrict=1`, তাই `sudo` লাগে) |
| `sudo dmesg -T` | মানুষের পড়ার মতো সময় |
| `sudo dmesg -H` | রঙিন, pager সহ |
| `sudo dmesg -w` | লাইভ দেখো (`tail -f`-এর মতো) |
| `sudo dmesg -l err,warn` | শুধু এরর ও warning |
| `journalctl -k` | journal থেকে kernel বার্তা (`-b -1` = আগের boot) |

```
$ sudo dmesg -T | tail -3
[Tue Nov 12 11:02:15 2026] usb 1-2: new high-speed USB device number 8 using xhci_hcd
[Tue Nov 12 11:02:15 2026] usb-storage 1-2:1.0: USB Mass Storage device detected
[Tue Nov 12 11:02:16 2026] sd 2:0:0:0: [sdb] 30031872 512-byte logical blocks: (15.4 GB/14.3 GiB)
```

### Magic SysRq: সিস্টেম জমে গেলে শেষ ভরসা

সিস্টেম পুরোপুরি জমে গেলে (মাউস নড়ে না, টার্মিনাল সাড়া দেয় না) সরাসরি পাওয়ার বাটন না চেপে, আগে kernel-কে সরাসরি কীবোর্ড থেকে নির্দেশ দেওয়া যায়। কম্বিনেশন হলো **Alt + SysRq (PrintScreen) + একটা অক্ষর**। কিছু ল্যাপটপে **Alt + Fn + PrtSc + অক্ষর** চাপতে হয়।

ক্লাসিক নিরাপদ reboot ক্রম হলো **R-E-I-S-U-B** (মনে রাখার বাক্য: "**R**eboot **E**ven **I**f **S**ystem **U**tterly **B**roken")। প্রতিটি অক্ষরের মাঝে কয়েক সেকেন্ড অপেক্ষা করো।

| শর্টকাট | অক্ষর | কাজ |
|---|---|---|
| Alt+SysRq+**R** | R | কীবোর্ড X/Wayland থেকে kernel-এর নিয়ন্ত্রণে ফেরানো (raw mode) |
| Alt+SysRq+**E** | E | init ছাড়া সব process-কে SIGTERM পাঠানো |
| Alt+SysRq+**I** | I | init ছাড়া সব process-কে SIGKILL পাঠানো |
| Alt+SysRq+**S** | S | সব ফাইলসিস্টেম sync (ডেটা ডিস্কে লেখা) |
| Alt+SysRq+**U** | U | সব ফাইলসিস্টেম read-only হিসেবে আবার মাউন্ট |
| Alt+SysRq+**B** | B | সাথে সাথে reboot |
| Alt+SysRq+**O** | O | পাওয়ার বন্ধ |
| Alt+SysRq+**F** | F | OOM killer ডেকে সবচেয়ে বেশি মেমরি খাওয়া process মারা |
| Alt+SysRq+**H** | H | সাহায্য (কনসোলে/dmesg-এ তালিকা) |

> ⚠️ **সতর্কতা:** সব অক্ষর ডিফল্টভাবে চালু থাকে না। কোনগুলো চালু সেটা ঠিক করে `kernel.sysrq` bitmask। Ubuntu-তে ডিফল্ট `176` (= 16 sync + 32 remount + 128 reboot/poweroff), তাই S, U, B, O কাজ করে, কিন্তু E, I, F করে না। সব চালু করতে: `sudo sysctl -w kernel.sysrq=1` (সাময়িক)। নিরাপত্তার কারণে পাবলিক বা শেয়ার্ড মেশিনে এটা সীমিত রাখাই ভালো।

একই কাজ টার্মিনাল থেকেও করা যায় (root হিসেবে):

```
$ echo s | sudo tee /proc/sysrq-trigger      # sync
$ echo c | sudo tee /proc/sysrq-trigger      # ইচ্ছে করে crash/panic! শুধু kdump পরীক্ষার জন্য, VM-এ
```

> ⚠️ **সতর্কতা:** `echo c > /proc/sysrq-trigger` সঙ্গে সঙ্গে kernel ক্র্যাশ করায়, আর সেভ না করা সব কাজ হারিয়ে যায়। এটা শুধু টেস্ট VM-এ kdump যাচাই করার জন্য।

### GRUB-এর কীবোর্ড শর্টকাট (kernel সমস্যায় দরকার হয়)

| কী | কাজ |
|---|---|
| **Esc** (UEFI) / **Shift** চেপে ধরা (BIOS) | লুকানো GRUB মেনু দেখানো (Ubuntu) |
| ↑ / ↓ | এন্ট্রি বাছাই |
| **Enter** | বাছাই করা এন্ট্রি boot |
| **e** | এন্ট্রি সাময়িকভাবে এডিট (যেমন `linux` লাইনে `modprobe.blacklist=nouveau` বা `systemd.unit=rescue.target` যোগ করা) |
| **Ctrl+X** বা **F10** | এডিট করা এন্ট্রি boot |
| **c** | GRUB কমান্ড লাইন |
| **Esc** (এডিট অবস্থায়) | পরিবর্তন বাতিল করে মেনুতে ফেরা |

---

## হাতে-কলমে অনুশীলন

> 💡 সব ল্যাব একটা virtual machine-এ করা সবচেয়ে নিরাপদ (দেখো [অধ্যায় ৩](03-distro-o-installation.md) ও [অধ্যায় ৪৮](48-containers-virtualization.md))।

### ল্যাব ১: তোমার kernel-কে চেনো

```
$ uname -r
$ uname -m
$ cat /proc/cmdline
$ ls /boot
$ grep -E 'CONFIG_EXT4_FS=|CONFIG_XFS_FS=|CONFIG_BTRFS_FS=' /boot/config-$(uname -r)
```

**প্রশ্ন:** তোমার kernel-এ ext4 built-in নাকি module? `/lib/modules/$(uname -r)/modules.builtin`-এ `ext4` আছে কিনা মিলিয়ে দেখো।

### ল্যাব ২: module লোড-আনলোড

```
# 1. dry run: কী কী লোড হবে?
$ sudo modprobe -n -v vfat

# 2. লোড করো এবং যাচাই করো
$ sudo modprobe -v vfat
$ lsmod | grep -E '^(vfat|fat) '

# 3. তথ্য দেখো
$ modinfo -F description vfat
$ modinfo -F depends vfat

# 4. আনলোড করো
$ sudo modprobe -r -v vfat
$ lsmod | grep fat || echo "fat গেল"
```

(যদি `vfat` ইতিমধ্যে ব্যবহারে থাকে, যেমন `/boot/efi` মাউন্ট করা থাকলে, তাহলে আনলোড হবে না। সেক্ষেত্রে `dummy` module দিয়ে চেষ্টা করো: `sudo modprobe dummy`, তারপর `ip link` দেখো, `dummy0` নামে নতুন ইন্টারফেস এসেছে কিনা।)

### ল্যাব ৩: boot-এ module লোড + blacklist

```
$ echo dummy | sudo tee /etc/modules-load.d/lab.conf
$ sudo systemctl restart systemd-modules-load.service
$ lsmod | grep dummy

# এবার blacklist পরীক্ষা
$ sudo modprobe -r dummy
$ echo 'install dummy /bin/false' | sudo tee /etc/modprobe.d/lab-block.conf
$ sudo modprobe dummy
modprobe: ERROR: ../libkmod/libkmod-module.c:... could not insert 'dummy': Operation not permitted
          <- বার্তার শব্দ kmod version অনুযায়ী একটু আলাদা হতে পারে

# পরিষ্কার করো
$ sudo rm /etc/modules-load.d/lab.conf /etc/modprobe.d/lab-block.conf
```

### ল্যাব ৪: `/proc` দিয়ে process তদন্ত

```
$ sleep 600 &
$ PID=$!
$ cat /proc/$PID/cmdline | tr '\0' ' '; echo
$ grep -E 'Name|State|PPid|VmRSS' /proc/$PID/status
$ ls -l /proc/$PID/exe /proc/$PID/cwd
$ ls -l /proc/$PID/fd
$ cat /proc/$PID/limits | head -5
$ kill $PID
$ ls /proc/$PID 2>&1 | head -1       # এখন আর নেই
```

### ল্যাব ৫: `/sys` থেকে হার্ডওয়্যার পড়া

```
$ ls /sys/class/net
$ for n in /sys/class/net/*; do echo "$(basename $n): $(cat $n/address) $(cat $n/operstate)"; done
$ for d in /sys/block/*; do echo "$(basename $d) rotational=$(cat $d/queue/rotational)"; done
$ cat /sys/class/power_supply/BAT*/capacity 2>/dev/null || echo "ব্যাটারি নেই (ডেস্কটপ/VM)"
```

### ল্যাব ৬: sysctl সাময়িক ও স্থায়ী

```
$ sysctl net.ipv4.ip_forward
$ sudo sysctl -w net.ipv4.ip_forward=1
$ cat /proc/sys/net/ipv4/ip_forward

# স্থায়ী করো
$ echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-lab.conf
$ sudo sysctl --system | grep ip_forward

# reboot-এর পর আবার দেখো, তারপর পরিষ্কার করো
$ sudo rm /etc/sysctl.d/99-lab.conf
$ sudo sysctl -w net.ipv4.ip_forward=0
```

### ল্যাব ৭: udev দেখা ও rule লেখা

```
# টার্মিনাল ১
$ udevadm monitor --udev
# এখন একটা pen drive লাগাও/খোলো, event দেখো; শেষে Ctrl+C

# তথ্য বের করো (তোমার ডিভাইসের নাম বসাও)
$ udevadm info -q property -n /dev/sdb | grep -E 'ID_VENDOR=|ID_MODEL=|ID_SERIAL_SHORT='

# rule: এই pen drive-এর জন্য /dev/mypendrive symlink
$ echo 'SUBSYSTEM=="block", ENV{ID_SERIAL_SHORT}=="<তোমার-সিরিয়াল>", ENV{DEVTYPE}=="disk", SYMLINK+="mypendrive"' | sudo tee /etc/udev/rules.d/99-lab.rules
$ sudo udevadm control --reload
# pen drive খুলে আবার লাগাও
$ ls -l /dev/mypendrive
```

### ল্যাব ৮: হার্ডওয়্যার রিপোর্ট

```
$ lscpu | grep -E 'Model name|Socket|Core|Thread|^CPU\(s\)'
$ lspci -k | grep -A3 -iE 'vga|network|ethernet'
$ lsusb -t
$ sudo dmidecode -s system-product-name
$ sudo dmidecode -t memory | grep -E '^\s+Size:'
$ sudo lshw -short -sanitize > ~/hardware.txt
```

**চ্যালেঞ্জ:** একটা script লেখো যেটা CPU মডেল, মোট RAM (`/proc/meminfo` থেকে), kernel version, আর প্রতিটি নেটওয়ার্ক কার্ডের driver (`/sys/class/net/*/device/driver`) এক পাতায় দেখায়। ([অধ্যায় ৪১](41-shell-scripting-1.md) দেখো।)

### ল্যাব ৯ (অ্যাডভান্সড, VM-এ): নিজের প্রথম kernel module

```
$ sudo apt install build-essential linux-headers-$(uname -r)    # Fedora: sudo dnf install gcc make kernel-devel
$ mkdir ~/hello-mod && cd ~/hello-mod
```

`hello.c`:

```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Rahim");
MODULE_DESCRIPTION("Amar prothom kernel module");

static int __init hello_init(void)
{
    pr_info("hello: kernel-e swagotom!\n");
    return 0;
}

static void __exit hello_exit(void)
{
    pr_info("hello: bidai!\n");
}

module_init(hello_init);
module_exit(hello_exit);
```

`Makefile` (লাইনের শুরুতে **Tab** দিতে হবে, space দিলে চলবে না):

```make
obj-m += hello.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```
$ make
$ modinfo ./hello.ko
$ sudo insmod ./hello.ko
$ sudo dmesg | tail -1
[ 9876.543210] hello: kernel-e swagotom!
$ lsmod | grep hello
$ sudo rmmod hello
$ sudo dmesg | tail -1
[ 9890.123456] hello: bidai!
$ cat /proc/sys/kernel/tainted      # out-of-tree module, তাই tainted হয়েছে
```

> ⚠️ Secure Boot চালু থাকলে `insmod` "Key was rejected by service" এরর দেবে। পরীক্ষার VM-এ Secure Boot বন্ধ রাখো, অথবা ৪৭.১০-এর মতো MOK দিয়ে sign করো।

---

## সারসংক্ষেপ

| কাজ | কমান্ড / ফাইল |
|---|---|
| kernel version | `uname -r`, `uname -a`, `cat /proc/version` |
| boot প্যারামিটার | `cat /proc/cmdline` |
| built-in নাকি module | `grep CONFIG_X /boot/config-$(uname -r)` |
| লোড হওয়া module | `lsmod` (= `/proc/modules`) |
| module-এর তথ্য | `modinfo <name>`, `modinfo -F parm <name>` |
| লোড / আনলোড | `sudo modprobe <name>` / `sudo modprobe -r <name>` |
| dry run | `modprobe -n -v <name>` |
| module dependency map আবার বানানো | `sudo depmod -a` |
| boot-এ লোড | `/etc/modules-load.d/*.conf` |
| module option ও blacklist | `/etc/modprobe.d/*.conf` (`options`, `blacklist`, `install ... /bin/false`) |
| initramfs আবার বানানো | `sudo update-initramfs -u` / `sudo dracut -f` |
| CPU / মেমরি | `/proc/cpuinfo`, `/proc/meminfo`, `lscpu`, `free -h` |
| process-এর তথ্য | `/proc/<PID>/{status,cmdline,fd,exe,cwd,environ}` |
| ডিভাইসের গাছ | `/sys/class`, `/sys/block`, `/sys/devices`, `/sys/module` |
| kernel প্যারামিটার পড়া | `sysctl <name>`, `sysctl -a` |
| সাময়িক বদল | `sudo sysctl -w <name>=<val>` |
| স্থায়ী বদল | `/etc/sysctl.d/99-*.conf` + `sudo sysctl --system` |
| udev তথ্য / লাইভ event | `udevadm info -a -n /dev/X`, `udevadm monitor` |
| udev rule আবার পড়া | `sudo udevadm control --reload && sudo udevadm trigger` |
| হার্ডওয়্যার | `lspci -k`, `lsusb -t`, `sudo dmidecode -t memory`, `sudo lshw -short` |
| ইনস্টল করা kernel | `dpkg --list 'linux-image*'` / `rpm -q kernel` |
| ডিফল্ট kernel (RHEL/Fedora) | `sudo grubby --default-kernel`, `--set-default` |
| DKMS | `dkms status`, `sudo dkms autoinstall` |
| Secure Boot অবস্থা | `mokutil --sb-state` |
| kernel বার্তা | `sudo dmesg -T`, `journalctl -k -b -1` |
| tainted? | `cat /proc/sys/kernel/tainted` |

| শর্টকাট | কাজ |
|---|---|
| Alt+SysRq+R,E,I,S,U,B | জমে যাওয়া সিস্টেম নিরাপদে reboot (`kernel.sysrq` অনুমতি দিলে) |
| Alt+SysRq+F | OOM killer ডাকা |
| Esc / Shift (boot-এর সময়) | GRUB মেনু দেখানো |
| `e` তারপর Ctrl+X / F10 | GRUB-এ kernel প্যারামিটার সাময়িক বদলে boot |
| Ctrl+C | `dmesg -w` বা `udevadm monitor` থামানো |

---

## নিজেকে যাচাই করো

1. `uname -r` দেখাচ্ছে `6.8.0-45-generic`। এর `6.8`, `45` আর `generic` অংশগুলো কী বোঝায়?
2. `modprobe` আর `insmod`-এর মধ্যে মূল পার্থক্য কী? সাধারণ কাজে কোনটা ব্যবহার করবে এবং কেন?
3. `/etc/modprobe.d/`-এ `blacklist usb_storage` লেখার পরও কেউ হাতে `modprobe usb_storage` চালালে কী হবে? পুরোপুরি আটকাতে কী লিখবে?
4. `/proc/meminfo` ফাইলের সাইজ ০ বাইট দেখায়, তবু `cat` করলে লেখা আসে কেন?
5. কোনো process ভুল করে মুছে ফেলা একটা log ফাইল এখনও খোলা রেখেছে। ফাইলটা কীভাবে উদ্ধার করবে?
6. `sudo echo 1 > /proc/sys/net/ipv4/ip_forward` কেন "Permission denied" দেয়? সঠিক উপায় দুটি বলো।
7. `sysctl -w` দিয়ে বদলানো মান reboot-এর পরও রাখতে কী করবে? ফাইলের নামের শুরুতে `99-` কেন দেওয়া হয়?
8. একটা USB-serial ডিভাইসকে সবসময় `/dev/gps` নামে পেতে চাও। কোন টুল দিয়ে attribute খুঁজবে, আর rule কোথায় রাখবে?
9. একটা নেটওয়ার্ক কার্ড কাজ করছে না। কোন কমান্ড দিয়ে দেখবে তার জন্য কোনো driver লোড হয়েছে কিনা?
10. kernel আপডেটের পর VirtualBox কাজ করছে না। DKMS-এর দিক থেকে কী কী পরীক্ষা করবে?

<details><summary>উত্তর দেখো</summary>

1. `6.8` হলো upstream kernel সিরিজ (major 6, minor 8)। `45` হলো Ubuntu-র ABI/build নম্বর, যেটা distro-র প্রতিটি kernel প্যাকেজ আপডেটে বাড়ে। `generic` হলো flavour, অর্থাৎ সাধারণ ডেস্কটপ/সার্ভারের জন্য বানানো (অন্যগুলো হলো `lowlatency`, `aws` ইত্যাদি)।
2. `insmod` একটা নির্দিষ্ট `.ko` ফাইল সরাসরি লোড করে: পুরো path লাগে, dependency দেখে না, `/etc/modprobe.d` মানে না। `modprobe` নাম দিয়ে কাজ করে, `modules.dep` থেকে dependency আগে লোড করে, কনফিগ (options, blacklist) মানে, আর `-r` দিয়ে পরিচ্ছন্নভাবে আনলোড করে। তাই সাধারণ কাজে `modprobe` ব্যবহার করবে।
3. লোড হয়ে যাবে, কারণ `blacklist` শুধু alias দিয়ে স্বয়ংক্রিয় লোড বন্ধ করে। পুরোপুরি আটকাতে লেখো `install usb_storage /bin/false`, তারপর initramfs আবার বানাও।
4. `/proc` একটা virtual filesystem (procfs)। ডিস্কে কোনো ডেটা নেই; পড়ার মুহূর্তে kernel লেখাটা তৈরি করে দেয়, তাই আগে থেকে সাইজ জানা যায় না।
5. `ls -l /proc/<PID>/fd/` দেখো, `(deleted)` লেখা লিংকটা খুঁজে বের করো, তারপর `cp /proc/<PID>/fd/<N> ~/recovered.log` চালাও।
6. redirection (`>`) চালায় তোমার সাধারণ shell, `sudo` নয়, তাই ফাইল খোলার অনুমতি পাওয়া যায় না। সঠিক উপায়: `echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward`, অথবা `sudo sysctl -w net.ipv4.ip_forward=1`।
7. `/etc/sysctl.d/99-custom.conf`-এ `key = value` লিখে `sudo sysctl --system` চালাও। ফাইলগুলো নাম অনুযায়ী ক্রমে পড়া হয়, আর পরে পড়া মান টেকে; `99-` দিলে ফাইলটা সবার শেষে পড়া হয়, ফলে অন্য কোনো ফাইল তোমার মান বদলে দিতে পারে না।
8. `lsusb` দিয়ে idVendor/idProduct, আর `udevadm info -a -n /dev/ttyUSB0` দিয়ে বিস্তারিত attribute খুঁজবে। rule রাখবে `/etc/udev/rules.d/99-gps.rules`-এ, যেমন `SUBSYSTEM=="tty", ATTRS{idVendor}=="xxxx", ATTRS{idProduct}=="yyyy", SYMLINK+="gps"`। তারপর `udevadm control --reload` ও `udevadm trigger`।
9. `lspci -k` (বা USB হলে `lsusb -t`) চালাও; `Kernel driver in use` লাইন না থাকলে driver লোড হয়নি। সাথে `sudo dmesg -T | grep -i <driver/firmware>` দেখো, আর `ip link` দিয়ে দেখো ইন্টারফেস তৈরি হয়েছে কিনা।
10. `dkms status` দেখো, নতুন kernel-এর জন্য `installed` আছে কিনা। নতুন kernel-এর headers ইনস্টল আছে কিনা যাচাই করো (`linux-headers-$(uname -r)` বা `kernel-devel`)। তারপর `sudo dkms autoinstall` চালাও। build ব্যর্থ হলে `make.log` দেখো। Secure Boot চালু থাকলে MOK enroll করা আছে কিনা দেখো (`mokutil --sb-state`, আর লোডের সময় "Key was rejected" এরর আসছে কিনা)।

</details>

---

[← আগের অধ্যায়](46-performance-troubleshooting.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](48-containers-virtualization.md)

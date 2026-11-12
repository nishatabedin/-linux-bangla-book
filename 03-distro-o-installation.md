# অধ্যায় ৩: ডিস্ট্রিবিউশন বাছাই ও ইনস্টলেশন

## এই অধ্যায়ে যা শিখবে

- **distro** (distribution) আসলে কী, আর এত distro কেন আছে
- বড় বড় distro পরিবার: Debian/Ubuntu, RHEL/Fedora/Rocky/Alma, Arch, SUSE, কোনটার কী বৈশিষ্ট্য
- **LTS** বনাম **rolling release** মডেল: কোনটা কখন ভালো
- একজন fresher-এর জন্য কোন distro দিয়ে শুরু করা উচিত, আর কেন
- ইনস্টল করার চারটি পদ্ধতি:
  - VirtualBox / VMware-এ **ভার্চুয়াল মেশিন (VM)**, ধাপে ধাপে
  - **WSL2**: Windows-এর ভেতরেই লিনাক্স
  - **Live USB** বানানো (Rufus, balenaEtcher, `dd`)
  - **Dual boot**: একই কম্পিউটারে Windows + Linux
- ISO ফাইল যাচাই করা (`sha256sum`)
- পার্টিশনিং-এর বেসিক: MBR/GPT, BIOS/UEFI, ESP, `/`, `/home`, swap
- প্রথম বুটের পরে কী কী করবে: আপডেট, ড্রাইভার, ব্যাকআপ
- সংশ্লিষ্ট কীবোর্ড শর্টকাট (BIOS/boot menu, VirtualBox, VMware, GRUB)

---

## ৩.১ Distro কী? আগে একটা উপমা

[অধ্যায় ২](02-linux-architecture.md)-এ আমরা দেখেছি, "Linux" শব্দটা আসলে শুধু **kernel** (সিস্টেমের কেন্দ্রীয় অংশ, যা হার্ডওয়্যার আর সফটওয়্যারের মাঝে দোভাষীর কাজ করে)-এর নাম। কিন্তু শুধু kernel দিয়ে তুমি কিছুই করতে পারবে না। তোমার লাগবে shell, কমান্ড (`ls`, `cp`...), লাইব্রেরি (glibc), init system (systemd), প্যাকেজ ম্যানেজার, ডেস্কটপ, ব্রাউজার, ইনস্টলার...

**Distribution (distro)** = Linux kernel + GNU টুলস + অন্যান্য সফটওয়্যার + প্যাকেজ ম্যানেজার + ইনস্টলার + কনফিগারেশন + সাপোর্ট নীতি, সব একসাথে প্যাকেট করা একটা সম্পূর্ণ অপারেটিং সিস্টেম।

> 💡 **উপমা: গাড়ির ইঞ্জিন আর গাড়ি**
> Linux kernel হলো একটা গাড়ির **ইঞ্জিন**। Toyota, Honda, Tata সবাই ইঞ্জিনের চারপাশে বডি, সিট, চাকা, ড্যাশবোর্ড লাগিয়ে আলাদা আলাদা গাড়ি বানায়। ঠিক তেমনি Ubuntu, Fedora, Arch একই (বা প্রায় একই) kernel নিয়ে নিজেদের মতো করে "গাড়ি" বানায়। চালানোর মূল নিয়ম (স্টিয়ারিং, ব্রেক) সব গাড়িতেই এক, তাই একটা শিখলে বাকিগুলো সহজ।

```
            ┌───────────────────────────────────────────────┐
            │              একটি Linux Distro               │
            │                                               │
            │  ┌─────────────┐  ┌────────────────────────┐ │
            │  │  ইনস্টলার    │  │ ডেস্কটপ (GNOME/KDE...) │ │
            │  └─────────────┘  └────────────────────────┘ │
            │  ┌──────────────────────────────────────────┐ │
            │  │ অ্যাপ্লিকেশন (Firefox, LibreOffice ...)  │ │
            │  └──────────────────────────────────────────┘ │
            │  ┌──────────────┐ ┌─────────────────────────┐ │
            │  │ প্যাকেজ       │ │ GNU টুলস + shell (bash) │ │
            │  │ ম্যানেজার     │ │ লাইব্রেরি (glibc)        │ │
            │  │ (apt/dnf/...) │ │ init (systemd)          │ │
            │  └──────────────┘ └─────────────────────────┘ │
            │  ┌──────────────────────────────────────────┐ │
            │  │            Linux Kernel                   │ │
            │  └──────────────────────────────────────────┘ │
            └───────────────────────────────────────────────┘
                              ▲
                              │
                         হার্ডওয়্যার
```

### distro-গুলো একে অপরের থেকে আসলে কোথায় আলাদা?

| বিষয় | উদাহরণ |
|---|---|
| **প্যাকেজ ফরম্যাট ও ম্যানেজার** | `.deb` + `apt` (Debian/Ubuntu), `.rpm` + `dnf` (Fedora/RHEL), `pacman` (Arch), `.rpm` + `zypper` (SUSE) |
| **রিলিজ মডেল** | নির্দিষ্ট সময় পরপর ভার্সন (fixed/LTS), নাকি সবসময় আপডেট (rolling) |
| **ডিফল্ট ডেস্কটপ** | GNOME, KDE Plasma, Cinnamon, XFCE ([অধ্যায় ৪](04-desktop-environment.md)) |
| **ডিফল্ট কনফিগারেশন** | ফায়ারওয়াল, SELinux/AppArmor, ফাইলসিস্টেম (ext4/Btrfs/XFS) |
| **সাপোর্ট ও লক্ষ্য** | ডেস্কটপ, সার্ভার, এন্টারপ্রাইজ, শেখা, পুরোনো হার্ডওয়্যার |
| **কে বানায়** | কমিউনিটি (Debian, Arch) নাকি কোম্পানি (Canonical→Ubuntu, Red Hat→RHEL, SUSE→SLES) |

> 📝 **মনে রাখো:** distro বদলালে বেশিরভাগ কমান্ড (`ls`, `cd`, `grep`, `systemctl`...) একই থাকে। মূলত বদলায় প্যাকেজ ইনস্টলের কমান্ড আর কিছু কনফিগ ফাইলের জায়গা। তাই এই বইয়ের ৯০% জিনিস যেকোনো distro-তে কাজ করবে।

---

## ৩.২ Distro পরিবার: বংশতালিকা

হাজারের বেশি distro আছে, কিন্তু বেশিরভাগই কয়েকটা "মূল" distro থেকে এসেছে। পরিবারগুলো চিনলে যেকোনো নতুন distro দেখেই বুঝবে সেটা কীভাবে চালাতে হবে।

```
Debian (1993) ─────────┬──► Ubuntu (2004) ──┬──► Linux Mint
   .deb / apt          │                    ├──► Pop!_OS
                       │                    ├──► Zorin OS
                       │                    └──► Kubuntu, Xubuntu ... (flavours)
                       ├──► Kali Linux (সিকিউরিটি টেস্টিং)
                       └──► Raspberry Pi OS

Red Hat ──► Fedora (2003, upstream, নতুন প্রযুক্তি)
   .rpm / dnf        │
                     ▼
               CentOS Stream (RHEL-এর ঠিক আগের ধাপ)
                     │
                     ▼
               RHEL (Red Hat Enterprise Linux, পেইড সাপোর্ট)
                     │  (সোর্স থেকে পুনর্নির্মাণ / সামঞ্জস্যপূর্ণ)
                     ├──► Rocky Linux
                     ├──► AlmaLinux
                     └──► Oracle Linux

Arch Linux (2002) ──┬──► Manjaro
   pacman / AUR      ├──► EndeavourOS
                     └──► CachyOS

SUSE ──┬──► openSUSE Leap   (fixed release)
.rpm / │──► openSUSE Tumbleweed (rolling)
zypper └──► SLES (SUSE Linux Enterprise Server, পেইড)

স্বাধীন / অন্যরকম: Slackware, Gentoo (সব সোর্স থেকে কম্পাইল),
                  Alpine (musl + BusyBox, কন্টেইনারে জনপ্রিয়), NixOS
```

### ৩.২.১ Debian / Ubuntu পরিবার

**Debian** সবচেয়ে পুরোনো ও সম্মানিত কমিউনিটি distro-গুলোর একটা। কোনো কোম্পানি এর মালিক না; হাজারো স্বেচ্ছাসেবক মিলে বানায়। এর মূলমন্ত্র **স্থিতিশীলতা (stability)**। Debian-এর তিনটি শাখা:

| শাখা | মানে |
|---|---|
| **stable** | প্রকাশিত, পরীক্ষিত ভার্সন। Debian 12 "Bookworm" (২০২৩), Debian 13 "Trixie" (আগস্ট ২০২৫) |
| **testing** | পরের stable হওয়ার প্রস্তুতি |
| **unstable ("sid")** | সবচেয়ে নতুন প্যাকেজ, ডেভেলপারদের জন্য |

Debian-এর ভার্সনের নাম Toy Story সিনেমার চরিত্র থেকে নেওয়া (Buster, Bullseye, Bookworm, Trixie...)।

**Ubuntu** বানায় Canonical কোম্পানি, Debian-এর ভিত্তির ওপর। Ubuntu-র লক্ষ্য ছিল "সাধারণ মানুষের জন্য লিনাক্স"। তাই ইনস্টল সহজ, ড্রাইভার সাপোর্ট ভালো, আর ইন্টারনেটে সবচেয়ে বেশি টিউটোরিয়াল Ubuntu নিয়েই। ভার্সন নম্বর = **বছর.মাস**: `24.04` মানে ২০২৪ সালের এপ্রিল।

- প্রতি ৬ মাসে একটা রিলিজ (এপ্রিল ও অক্টোবর)
- প্রতি ২ বছরে এপ্রিলের রিলিজটা **LTS** (24.04, 26.04...)
- Ubuntu-র নিজস্ব জিনিস: **Snap** প্যাকেজ, **Netplan** (নেটওয়ার্ক কনফিগ), **AppArmor**

**Linux Mint**: Ubuntu-ভিত্তিক, Cinnamon ডেস্কটপ, Windows ব্যবহারকারীদের কাছে খুব পরিচিত লাগে।

### ৩.২.২ Red Hat পরিবার: RHEL / Fedora / CentOS Stream / Rocky / Alma

এন্টারপ্রাইজ সার্ভারের দুনিয়ায় এই পরিবারের রাজত্ব। ব্যাংক, টেলিকম, সরকারি প্রতিষ্ঠানে প্রচুর RHEL চলে।

- **Fedora**: Red Hat-এর স্পনসরে কমিউনিটি distro। নতুন প্রযুক্তি (Wayland, PipeWire, Btrfs ডিফল্ট) সবার আগে Fedora-তে আসে। প্রায় ৬ মাস পরপর রিলিজ, প্রতিটার সাপোর্ট প্রায় ১৩ মাস। ডেভেলপারদের ডেস্কটপ হিসেবে খুব জনপ্রিয়।
- **CentOS Stream**: Fedora থেকে জিনিস পরিপক্ব হয়ে আসে এখানে, তারপর RHEL-এ। এটা RHEL-এর "পরের মাইনর ভার্সনের প্রিভিউ"।
- **RHEL**: পেইড সাবস্ক্রিপশন, ১০ বছরের লাইফসাইকেল, অফিশিয়াল সাপোর্ট। RHEL 9 (২০২২), RHEL 10 (২০২৫)। শেখার জন্য Red Hat-এর **ফ্রি Developer Subscription** দিয়ে বিনামূল্যে ব্যবহার করা যায়।
- **Rocky Linux** ও **AlmaLinux**: পুরোনো CentOS Linux বন্ধ হওয়ার পর (CentOS 8 শেষ হয় ডিসেম্বর ২০২১-এ, CentOS 7 শেষ হয় জুন ২০২৪-এ) এরা এসেছে RHEL-এর সঙ্গে সামঞ্জস্যপূর্ণ (compatible), ফ্রি বিকল্প হিসেবে। RHEL শিখতে চাইলে এগুলো দারুণ।

```
নতুন প্রযুক্তির প্রবাহ:

 Fedora  ──────►  CentOS Stream  ──────►  RHEL  ──────► Rocky / Alma
 (পরীক্ষাগার)      (প্রিভিউ)              (এন্টারপ্রাইজ)   (ফ্রি সামঞ্জস্যপূর্ণ)
```

> 💡 **টিপস:** তুমি যদি **RHCSA** (Red Hat Certified System Administrator) সার্টিফিকেশন দিতে চাও, তাহলে Rocky Linux / AlmaLinux / RHEL দিয়ে প্র্যাকটিস করো।

### ৩.২.৩ Arch পরিবার

**Arch Linux** এর দর্শন: **KISS** (Keep It Simple, Stupid) আর "ব্যবহারকারীই সব ঠিক করবে"। ডিফল্টে কোনো ডেস্কটপ নেই। তুমি নিজে কমান্ড দিয়ে সব ইনস্টল করবে (এখন `archinstall` নামে একটা গাইডেড স্ক্রিপ্টও আছে)।

- **Rolling release**: একবার ইনস্টল করো, সারাজীবন `pacman -Syu` দিয়ে আপডেট
- **AUR** (Arch User Repository): কমিউনিটির বানানো প্যাকেজ রেসিপির বিশাল ভান্ডার
- **ArchWiki**: লিনাক্সের অন্যতম সেরা ডকুমেন্টেশন। অন্য distro ব্যবহারকারীরাও এটা পড়ে

**Manjaro**, **EndeavourOS**: Arch-কে সহজ করে ইনস্টলার ও ডেস্কটপসহ দেয়।

> ⚠️ **সতর্কতা:** একদম প্রথম দিন Arch দিয়ে শুরু করলে হতাশ হওয়ার সম্ভাবনা বেশি। আগে Ubuntu/Mint/Fedora-তে বেসিক শিখে নাও, তারপর Arch চ্যালেঞ্জ হিসেবে নিও। তখন অনেক কিছু শিখবে।

### ৩.২.৪ SUSE পরিবার

জার্মান কোম্পানি SUSE। ইউরোপে এন্টারপ্রাইজে জনপ্রিয়।

- **openSUSE Leap**: fixed release, SLES-এর সঙ্গে কোড শেয়ার করে, স্থিতিশীল
- **openSUSE Tumbleweed**: rolling, কিন্তু **openQA** নামের স্বয়ংক্রিয় টেস্টিং পার হয়ে প্যাকেজ আসে, তাই rolling হয়েও তুলনামূলক নির্ভরযোগ্য
- **SLES**: পেইড এন্টারপ্রাইজ
- প্যাকেজ ম্যানেজার `zypper`। ঐতিহ্যগতভাবে **YaST** নামে একটা সর্বগ্রাসী সেটিংস টুল ছিল (নতুন Leap 16-এ ইনস্টলার হিসেবে Agama আর ম্যানেজমেন্টে Cockpit-এর দিকে সরে যাচ্ছে)।
- Btrfs + **Snapper** ডিফল্ট: আপডেটের আগে স্বয়ংক্রিয় স্ন্যাপশট, গোলমাল হলে আগের অবস্থায় ফিরে যাওয়া যায়।

### ৩.২.৫ পরিবারগুলোর তুলনা এক নজরে

| পরিবার | প্যাকেজ ফরম্যাট | ম্যানেজার | আপডেট কমান্ড | রিলিজ মডেল | মূল ব্যবহার |
|---|---|---|---|---|---|
| Debian | `.deb` | `apt` | `sudo apt update && sudo apt upgrade` | fixed (~২ বছর) | সার্ভার, স্থিতিশীল ডেস্কটপ |
| Ubuntu | `.deb` (+snap) | `apt` | একই | ৬ মাস / LTS ২ বছর | ডেস্কটপ, ক্লাউড, সার্ভার |
| Fedora | `.rpm` | `dnf` | `sudo dnf upgrade --refresh` | ~৬ মাস | ডেভেলপার ডেস্কটপ |
| RHEL/Rocky/Alma | `.rpm` | `dnf` | `sudo dnf update` | মেজর ~৩ বছর, ১০ বছর সাপোর্ট | এন্টারপ্রাইজ সার্ভার |
| Arch | `.pkg.tar.zst` | `pacman` | `sudo pacman -Syu` | rolling | উন্নত ব্যবহারকারী |
| openSUSE | `.rpm` | `zypper` | Leap: `sudo zypper up`, Tumbleweed: `sudo zypper dup` | Leap fixed / TW rolling | ডেস্কটপ, সার্ভার |

প্যাকেজ ম্যানেজমেন্ট বিস্তারিত দেখবে [অধ্যায় ৩০](30-package-management.md)-এ।

---

## ৩.৩ LTS বনাম Rolling Release

### Fixed / Point release এবং LTS

**Fixed release** মানে distro নির্দিষ্ট সময় পরপর একটা "ভার্সন" বের করে (Ubuntu 24.04, Debian 12)। ওই ভার্সনের ভেতরে প্যাকেজের **মূল ভার্সন জমিয়ে রাখা হয় (freeze)**; শুধু নিরাপত্তা ও বাগ ফিক্স আসে (backport)।

**LTS (Long Term Support)** = দীর্ঘমেয়াদি সাপোর্টের ভার্সন। যেমন:

| Distro | LTS/সাপোর্ট সময়কাল |
|---|---|
| Ubuntu LTS (যেমন 24.04) | ৫ বছর স্ট্যান্ডার্ড সাপোর্ট; **Ubuntu Pro** (ব্যক্তিগত ব্যবহারে ৫টি মেশিন পর্যন্ত ফ্রি) দিয়ে মোট ১০ বছর ESM, Legacy add-on সহ আরও বেশি |
| Ubuntu interim (যেমন 24.10, 25.04) | মাত্র ৯ মাস |
| Debian stable | ~৩ বছর নিয়মিত + LTS টিম মিলিয়ে মোট ~৫ বছর |
| RHEL / Rocky / Alma | ~১০ বছর |

### Rolling release

কোনো "ভার্সন" নেই। প্যাকেজের নতুন ভার্সন আসার পর (কিছু টেস্টের পরে) সরাসরি তোমার কাছে আসে। Arch, openSUSE Tumbleweed, Debian sid এই মডেলে চলে।

> 💡 **উপমা:** Fixed release হলো **বছরে একবার ছাপা বই**: প্রতিটা সংস্করণ যত্ন করে সম্পাদিত, ভুল কম, কিন্তু নতুন খবর পেতে পরের সংস্করণের অপেক্ষা। Rolling release হলো **প্রতিদিনের খবরের কাগজ**: সবসময় সর্বশেষ খবর, কিন্তু মাঝে মাঝে ছাপার ভুল থাকে।

```
Fixed / LTS:
  24.04 ●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ (৫+ বছর শুধু ফিক্স)
         26.04 ●━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  (বড় আপগ্রেড = নতুন ভার্সনে "লাফ")

Rolling:
  ●──●──●──●──●──●──●──●──●──●──●──●──●──●──●──●──►  (প্রতিদিন/সপ্তাহে ছোট ছোট আপডেট)
```

| দিক | LTS / Fixed | Rolling |
|---|---|---|
| স্থিতিশীলতা | অনেক বেশি | কম (মাঝে মাঝে কিছু ভাঙতে পারে) |
| সফটওয়্যারের নতুনত্ব | পুরোনো (কিন্তু পরীক্ষিত) | সবসময় সর্বশেষ |
| নতুন হার্ডওয়্যার সাপোর্ট | কিছুটা দেরিতে (Ubuntu HWE kernel দিয়ে আসে) | দ্রুত |
| রক্ষণাবেক্ষণ | কম মনোযোগ লাগে | নিয়মিত খবর রাখতে হয় (Arch news পড়া) |
| সার্ভারের জন্য | ✅ আদর্শ | সাধারণত ❌ |
| শেখার জন্য | ✅ fresher-এর জন্য ভালো | অভিজ্ঞ হলে |

> 📝 **মনে রাখো:** প্রোডাকশন সার্ভারে প্রায় সবসময় LTS/এন্টারপ্রাইজ distro চলে (Ubuntu LTS, Debian, RHEL/Rocky/Alma)। কারণ সার্ভারে "নতুন ফিচার" না, দরকার "কখনো না ভাঙা"।

---

## ৩.৪ Fresher-এর জন্য কোনটা?

### সিদ্ধান্তের ফ্লোচার্ট

```
                  তুমি কী চাও?
                       │
     ┌─────────────────┼──────────────────┬──────────────────┐
     ▼                 ▼                  ▼                  ▼
 প্রথমবার লিনাক্স,   Windows-এর মতো     সার্ভার/DevOps     RHCSA/এন্টারপ্রাইজ
 সবচেয়ে বেশি        দেখতে চাই          ক্যারিয়ার          চাকরি
 টিউটোরিয়াল           │                  │                  │
     │                 ▼                  ▼                  ▼
     ▼            Linux Mint        Ubuntu Server LTS    Rocky / AlmaLinux
 Ubuntu 24.04 LTS  (Cinnamon)        বা Debian            (বা ফ্রি RHEL dev)
 Desktop

 পুরোনো দুর্বল PC ──► Xubuntu / Lubuntu / Linux Mint XFCE
 নতুনতম সফটওয়্যার, ডেভেলপার ──► Fedora Workstation
 ভেতরটা গভীরভাবে জানতে চাই (কিছুদিন পরে) ──► Arch Linux
```

### আমাদের পরামর্শ

**এই বইয়ের জন্য Ubuntu 24.04 LTS** (Desktop) ব্যবহার করো, VirtualBox-এ বা WSL2-তে। কারণ:

1. সবচেয়ে বেশি গাইড, Stack Overflow উত্তর, YouTube ভিডিও
2. LTS, তাই বছরের পর বছর স্থিতিশীল
3. ক্লাউডে (AWS, Azure, GCP) সবচেয়ে বেশি ব্যবহৃত সার্ভার OS-গুলোর একটা, তাই শেখা সরাসরি চাকরিতে কাজে লাগবে
4. পরে চাইলে পাশাপাশি একটা Rocky Linux VM-ও বানাও, যাতে `dnf`-ভিত্তিক দুনিয়াও চেনো

### সিস্টেম রিকোয়ারমেন্ট (মোটামুটি)

| Distro | RAM | ডিস্ক | মন্তব্য |
|---|---|---|---|
| Ubuntu 24.04 Desktop | ৪ GB+ (৮ GB আরামদায়ক) | ২৫ GB+ | GNOME ভারী |
| Linux Mint (Cinnamon/XFCE) | ২–৪ GB | ২০ GB+ | XFCE হালকা |
| Fedora Workstation | ৪ GB+ | ২০ GB+ | |
| Ubuntu Server / Debian (GUI ছাড়া) | ১–২ GB | ১০ GB+ | শুধু টার্মিনাল |
| Lubuntu / Xubuntu | ১–২ GB | ১০–২০ GB | পুরোনো PC |

> 💡 **টিপস:** VM-এ চালালে তোমার আসল কম্পিউটারের RAM ভাগ হবে। হোস্টে ৮ GB RAM থাকলে VM-কে ৩–৪ GB দাও; ১৬ GB থাকলে ৪–৬ GB।

---

## ৩.৫ ইনস্টলেশনের চারটি পথ: তুলনা

| পদ্ধতি | ঝুঁকি | পারফরম্যান্স | আসল হার্ডওয়্যার অভিজ্ঞতা | কার জন্য |
|---|---|---|---|---|
| **VM** (VirtualBox/VMware) | প্রায় শূন্য | মাঝারি | না | ✅ সব fresher, প্রথম ধাপ |
| **WSL2** | প্রায় শূন্য | ভালো (CLI) | না, GUI সীমিত | Windows ব্যবহারকারী, ডেভেলপার |
| **Live USB** | শূন্য (ইনস্টল না করলে) | USB-নির্ভর | হ্যাঁ | চেষ্টা করে দেখা, রেসকিউ |
| **Dual boot** | মাঝারি (ভুলে ডেটা যেতে পারে) | সর্বোচ্চ | হ্যাঁ | যারা নিয়মিত লিনাক্স চালাবে |
| **পুরো ডিস্কে লিনাক্স** | Windows মুছে যাবে | সর্বোচ্চ | হ্যাঁ | পূর্ণ লিনাক্স ব্যবহারকারী |

সবচেয়ে নিরাপদ শুরু: **VM**। এবার প্রতিটা ধাপে ধাপে দেখি। তার আগে, সব পদ্ধতিতে যেটা লাগবে: **ISO ফাইল**।

---

## ৩.৬ ISO ফাইল ডাউনলোড ও যাচাই

**ISO** হলো পুরো ইনস্টলেশন ডিস্কের একটা হুবহু ইমেজ ফাইল (`.iso`)। সবসময় **অফিশিয়াল সাইট** থেকে নামাবে:

| Distro | সাইট |
|---|---|
| Ubuntu | `https://ubuntu.com/download` |
| Linux Mint | `https://linuxmint.com/download.php` |
| Fedora | `https://fedoraproject.org` |
| Debian | `https://www.debian.org/distrib/` |
| Rocky Linux | `https://rockylinux.org/download` |
| AlmaLinux | `https://almalinux.org` |
| Arch | `https://archlinux.org/download/` |

### কেন checksum যাচাই করবে?

ডাউনলোডের সময় ফাইল নষ্ট (corrupt) হতে পারে, বা কোনো ক্ষতিকর মিরর পরিবর্তিত ফাইল দিতে পারে। **SHA256 checksum** হলো ফাইলের একটা "আঙুলের ছাপ": ফাইলের একটা বিটও বদলালে ছাপ সম্পূর্ণ বদলে যায়।

```
  ubuntu-24.04.x-desktop-amd64.iso  ──►  [SHA-256 অ্যালগরিদম]  ──►  a1b2c3...৬৪ অক্ষর
                                                                          │
                       distro-র সাইটে দেওয়া SHA256SUMS ফাইলের মান ◄── মিলছে? ✅/❌
```

**লিনাক্স/macOS-এ:**

```bash
$ sha256sum ubuntu-24.04.3-desktop-amd64.iso
<64-অক্ষরের-হ্যাশ>  ubuntu-24.04.3-desktop-amd64.iso
```

SHA256SUMS ফাইলটাও একই ফোল্ডারে নামিয়ে থাকলে স্বয়ংক্রিয় মিলিয়ে দেখো:

```bash
$ sha256sum -c SHA256SUMS --ignore-missing
ubuntu-24.04.3-desktop-amd64.iso: OK
```

**Windows-এ** (PowerShell):

```powershell
PS> Get-FileHash .\ubuntu-24.04.3-desktop-amd64.iso -Algorithm SHA256
```

| `sha256sum` অপশন | কাজ |
|---|---|
| `-c FILE` | FILE-এ লেখা হ্যাশের সঙ্গে মিলিয়ে দেখে |
| `--ignore-missing` | `-c` এর সময় যে ফাইল ডিস্কে নেই, সেটা এড়িয়ে যায় |
| `--quiet` | শুধু ব্যর্থ ফলাফল দেখায় |

> 💡 **টিপস (অ্যাডভান্সড):** আরও নিরাপদ হতে চাইলে `SHA256SUMS.gpg` সিগনেচার `gpg --verify SHA256SUMS.gpg SHA256SUMS` দিয়ে যাচাই করো (আগে distro-র পাবলিক কী ইমপোর্ট করতে হয়)। এতে নিশ্চিত হও যে checksum ফাইলটাও আসল।

> 📝 ISO ফাইলের নামে ভার্সন নম্বর (যেমন `24.04.3`) সময়ের সঙ্গে বদলায় (point release)। তুমি সাইটে যেটা পাবে সেটাই নাও।

---

## ৩.৭ পদ্ধতি ১: VirtualBox-এ ভার্চুয়াল মেশিন (ধাপে ধাপে)

### VM ভেতরে কীভাবে কাজ করে?

**Virtual Machine (VM)** = সফটওয়্যার দিয়ে বানানো একটা নকল কম্পিউটার। তোমার আসল কম্পিউটার (**host**) তার CPU, RAM, ডিস্কের একটা অংশ ধার দেয় একটা "অতিথি" OS-কে (**guest**)। মাঝখানে থাকে **hypervisor** (VirtualBox, VMware), যে guest-কে বিশ্বাস করায় যে সে আসল হার্ডওয়্যারে চলছে।

> 💡 **উপমা:** তোমার বাড়ি (host কম্পিউটার)-এর ভেতরে একটা আলাদা ঘর (VM) ভাড়া দিলে। ভাড়াটে (guest OS) মনে করে এটা তার পুরো বাড়ি। ঘরে আগুন লাগলেও (guest ক্র্যাশ) মূল বাড়ির কিছু হয় না, আর ঘরটা যেকোনো সময় ভেঙে নতুন করে বানানো যায়।

```
┌────────────────────────────────────────────────────────┐
│   Guest: Ubuntu 24.04         Guest: Rocky Linux 9     │
│   ┌──────────────────┐        ┌──────────────────┐     │
│   │ apps / kernel    │        │ apps / kernel    │     │
│   │ ভার্চুয়াল CPU/RAM │        │ ভার্চুয়াল CPU/RAM │     │
│   │ ভার্চুয়াল ডিস্ক   │        │ (.vdi / .vmdk)    │     │
│   └──────────────────┘        └──────────────────┘     │
│ ───────────────────────────────────────────────────── │
│       Hypervisor (VirtualBox / VMware) — Type 2        │
│ ───────────────────────────────────────────────────── │
│       Host OS: Windows / macOS / Linux                 │
│ ───────────────────────────────────────────────────── │
│   আসল হার্ডওয়্যার: CPU (VT-x/AMD-V সহ), RAM, SSD       │
└────────────────────────────────────────────────────────┘
```

- **Type 2 hypervisor** (VirtualBox, VMware Workstation): একটা সাধারণ OS-এর ওপর অ্যাপ হিসেবে চলে।
- **Type 1 hypervisor** (KVM, VMware ESXi, Hyper-V, Xen): সরাসরি হার্ডওয়্যারের ওপর বা kernel-এর অংশ হিসেবে চলে, সার্ভারে ব্যবহৃত।
- আধুনিক CPU-তে **হার্ডওয়্যার ভার্চুয়ালাইজেশন** (Intel **VT-x**, AMD **AMD-V/SVM**) থাকে, যা VM-কে প্রায় নেটিভ গতিতে চালায়। এটা BIOS/UEFI-তে **চালু** থাকতে হবে।

বিস্তারিত ভার্চুয়ালাইজেশন ও কন্টেইনার: [অধ্যায় ৪৮](48-containers-virtualization.md)।

### ধাপ ০: প্রস্তুতি

1. **ভার্চুয়ালাইজেশন চালু আছে কিনা দেখো।** Windows-এ: Task Manager → Performance → CPU → নিচে **"Virtualization: Enabled"**। Disabled হলে BIOS/UEFI-তে ঢুকে (বুটের সময় `F2`/`Del`) **Intel Virtualization Technology / VT-x** বা **SVM Mode** চালু করো।
2. লিনাক্স host হলে যাচাই:
   ```bash
   $ grep -Ec '(vmx|svm)' /proc/cpuinfo
   8
   ```
   ০-এর বেশি হলে CPU সাপোর্ট করে।
3. **VirtualBox** নামাও: `https://www.virtualbox.org` (Windows/macOS-Intel/Linux সবার জন্য ফ্রি; Apple Silicon Mac-এ ARM ভার্সনের সাপোর্ট সীমিত, সেখানে UTM বা VMware Fusion সহজ)। সঙ্গে একই ভার্সনের **Extension Pack** চাইলে নিতে পারো (USB 3 পাসথ্রু ইত্যাদির জন্য; শেখার জন্য বাধ্যতামূলক না)।
4. Ubuntu ISO নামিয়ে checksum যাচাই করো (৩.৬)।

> ⚠️ **সতর্কতা:** Windows-এ **Hyper-V**, **WSL2**, বা **Memory Integrity (Core Isolation)** চালু থাকলে VirtualBox ধীর হতে পারে (VM-এর নিচের কোণায় সবুজ কচ্ছপ 🐢 আইকন দেখায়)। নতুন VirtualBox ভার্সনগুলো Hyper-V-এর ওপর দিয়ে চলতে পারে, কিন্তু গতি কম। বেশি সমস্যা হলে VMware Workstation চেষ্টা করো, যেটা Hyper-V-এর সঙ্গে ভালো মানিয়ে চলে।

### ধাপ ১: নতুন VM তৈরি

VirtualBox খুলে **New** (বা `Ctrl+N`)-এ ক্লিক করো:

| ঘর | কী দেবে |
|---|---|
| **Name** | `ubuntu-lab` |
| **Folder** | যেখানে VM ফাইল থাকবে (পর্যাপ্ত জায়গা আছে এমন ড্রাইভ) |
| **ISO Image** | নামানো `ubuntu-24.04.x-desktop-amd64.iso` |
| **Type / Version** | ISO দিলে নিজেই Linux / Ubuntu (64-bit) বসিয়ে নেয় |
| **Skip Unattended Installation** | ✅ টিক দাও (নিজের হাতে ইনস্টল করে শেখার জন্য) |

> 📝 **Unattended Installation** চালু থাকলে VirtualBox নিজেই ইউজারনেম/পাসওয়ার্ড দিয়ে চুপচাপ ইনস্টল করে দেয়। সুবিধাজনক, কিন্তু তুমি ইনস্টলারের ধাপগুলো দেখতে পাবে না। প্রথমবার নিজে করো।

### ধাপ ২: হার্ডওয়্যার বরাদ্দ

| সেটিং | প্রস্তাবিত মান | কেন |
|---|---|---|
| **Base Memory** | ৪০৯৬ MB (৪ GB) | GNOME ডেস্কটপ আরামে চলবে |
| **Processors** | ২ (হোস্টে যত কোর, তার অর্ধেকের বেশি না) | host যেন ধীর না হয় |
| **Enable EFI** | ✅ (ঐচ্ছিক) | আধুনিক UEFI বুট শেখার জন্য |

### ধাপ ৩: ভার্চুয়াল হার্ড ডিস্ক

| সেটিং | মান |
|---|---|
| **Create a Virtual Hard Disk Now** | নির্বাচন করো |
| **Size** | ২৫–৪০ GB |
| **Format** | VDI (VirtualBox Disk Image) |
| **Pre-allocate Full Size** | ❌ (dynamically allocated: যতটুকু ব্যবহার, ততটুকু জায়গা নেয়) |

> 💡 **dynamically allocated** ডিস্ক শুরুতে ছোট থাকে, ডেটা লিখলে বাড়ে। **fixed size** একটু দ্রুত, কিন্তু শুরুতেই পুরো জায়গা দখল করে।

**Finish** চাপো। এবার VM সিলেক্ট করে **Settings**-এ কয়েকটা জিনিস ঠিক করো:

- **Display → Video Memory**: সর্বোচ্চ (১২৮ MB), Graphics Controller: **VMSVGA**
- **Network → Adapter 1**: **NAT** (ডিফল্ট, ইন্টারনেট পাবে)। পরে নেটওয়ার্কিং/SSH শিখতে **Bridged Adapter** বা **Host-only** লাগবে ([অধ্যায় ৩৮](38-ssh-remote.md))।
- **General → Advanced → Shared Clipboard**: Bidirectional (Guest Additions ইনস্টলের পরে কাজ করবে)

| নেটওয়ার্ক মোড | কী হয় |
|---|---|
| **NAT** | VM ইন্টারনেট পায়, কিন্তু বাইরের কেউ VM-এ ঢুকতে পারে না (port forwarding ছাড়া) |
| **Bridged** | VM তোমার রাউটার থেকে নিজস্ব IP পায়, LAN-এর আরেকটা কম্পিউটারের মতো |
| **Host-only** | শুধু host আর VM-গুলোর মধ্যে নেটওয়ার্ক, ইন্টারনেট নেই |
| **NAT Network** | একাধিক VM এক NAT-এর পেছনে একে অপরকে দেখতে পায় |
| **Internal Network** | শুধু VM-গুলোর মধ্যে, host-ও দেখে না |

### ধাপ ৪: VM চালু ও Ubuntu ইনস্টলার

**Start** চাপো। GRUB মেনু আসবে: **"Try or Install Ubuntu"** বেছে Enter। কিছুক্ষণ পর Ubuntu-র ইনস্টলার (Flutter-এ বানানো নতুন ইনস্টলার) খুলবে:

1. **Language**: English (বাংলাও আছে, কিন্তু টার্মিনালের এরর মেসেজ ইংরেজিতে রাখলে গুগল করা সহজ)
2. **Accessibility**: দরকার হলে
3. **Keyboard layout**: English (US)
4. **Network**: VM-এ NAT থাকলে স্বয়ংক্রিয় সংযোগ
5. **Try or Install**: **Install Ubuntu**
6. **Type of installation**: **Interactive installation**
7. **Applications**: **Default selection** (হালকা) বা **Extended selection** (অফিস ইত্যাদি সহ)
8. **Optimise your computer**: ✅ *Install third-party software for graphics and Wi-Fi hardware* এবং ✅ *Download and install support for additional media formats* (আসল হার্ডওয়্যারে বিশেষ জরুরি)
9. **Disk setup**: **Erase disk and install Ubuntu**। চিন্তা নেই, এটা শুধু **ভার্চুয়াল ডিস্ক** মুছবে, তোমার আসল ডিস্ক না। (Advanced features-এ LVM/এনক্রিপশনের অপশন আছে; [অধ্যায় ৩৩](33-lvm-raid.md)-এ LVM শিখবে)
10. **Create your account**: নাম, কম্পিউটারের নাম (hostname, যেমন `ubuntu-lab`), ইউজারনেম (ছোট হাতের, স্পেস ছাড়া, যেমন `rahim`), শক্তিশালী পাসওয়ার্ড
11. **Timezone**: Asia/Dhaka
12. **Review** → **Install**

ইনস্টল শেষে **Restart now**। "Please remove the installation medium, then press ENTER" এলে Enter চাপো (VirtualBox সাধারণত ISO নিজেই খুলে নেয়)।

```
ইনস্টলার ভেতরে কী করে (সংক্ষেপে):

 [ডিস্ক পার্টিশন] → [ফাইলসিস্টেম তৈরি (ext4)] → [বেস সিস্টেম কপি]
        → [ইউজার তৈরি, /etc কনফিগ] → [bootloader (GRUB) ইনস্টল]
        → [initramfs তৈরি] → রিবুট
```

### ধাপ ৫: Guest Additions (VirtualBox)

**Guest Additions** = guest-এর ভেতরে চলা কিছু ড্রাইভার/সার্ভিস, যাতে পাওয়া যায়: স্বয়ংক্রিয় স্ক্রিন রিসাইজ, শেয়ার্ড ক্লিপবোর্ড, ড্র্যাগ-ড্রপ, শেয়ার্ড ফোল্ডার, ভালো মাউস।

Ubuntu-তে লগইন করে টার্মিনাল খোলো (`Ctrl+Alt+T`), তারপর:

```bash
$ sudo apt update
$ sudo apt install -y build-essential dkms linux-headers-$(uname -r) bzip2
```

এখন VirtualBox মেনু থেকে **Devices → Insert Guest Additions CD image...**। তারপর:

```bash
$ cd /media/$USER/VBox_GAs_*
$ sudo ./VBoxLinuxAdditions.run
Verifying archive integrity...  All good.
...
VirtualBox Guest Additions: Building the modules for kernel 6.8.0-xx-generic.
...
$ sudo reboot
```

> 💡 **সহজ বিকল্প:** অনেক সময় distro-র রিপোজিটরিতেই guest প্যাকেজ থাকে (Ubuntu-তে `virtualbox-guest-x11` ইত্যাদি)। তবে VirtualBox-এর সঙ্গে আসা ISO-টাই ভার্সন মেলানোর জন্য সবচেয়ে নির্ভরযোগ্য।

**শেয়ার্ড ফোল্ডার** ব্যবহার করতে নিজেকে `vboxsf` গ্রুপে যোগ করতে হয় (গ্রুপ নিয়ে [অধ্যায় ১৫](15-users-groups.md)):

```bash
$ sudo usermod -aG vboxsf $USER
# তারপর লগআউট-লগইন
```

### ধাপ ৬: Snapshot: তোমার "Undo" বোতাম

VM-এর সবচেয়ে বড় সুবিধা **snapshot**: VM-এর কোনো মুহূর্তের পুরো অবস্থা সংরক্ষণ। পরে কিছু ভেঙে ফেললে এক ক্লিকে ওই অবস্থায় ফিরে যাবে।

VirtualBox Manager → VM সিলেক্ট → Snapshots → **Take** (চলন্ত VM-এ `Host+T`)। নাম দাও: `fresh-install-updated`।

> 💡 **টিপস:** এই বইয়ের প্রতিটা "বিপজ্জনক" অধ্যায়ের (পার্টিশন, LVM, ফায়ারওয়াল, বুট প্রসেস) আগে একটা snapshot নিও। নির্ভয়ে পরীক্ষা করো: ভাঙলে restore!

### VMware Workstation দিয়ে করলে

২০২৪ থেকে **VMware Workstation Pro** (Windows/Linux) ও **Fusion Pro** (macOS) ব্যক্তিগত ও বাণিজ্যিক উভয় ব্যবহারে ফ্রি করা হয়েছে; Broadcom-এর সাপোর্ট পোর্টালে অ্যাকাউন্ট খুলে নামাতে হয়। ধাপ প্রায় একই:

1. **Create a New Virtual Machine** → **Typical** → ISO দেখাও
2. Linux / Ubuntu 64-bit, ডিস্ক ২৫–৪০ GB (*Store virtual disk as a single file* বা *split*, যেকোনো একটা)
3. **Customize Hardware**: Memory ৪ GB, Processors ২
4. ইনস্টল একই রকম। Guest tools হিসেবে **open-vm-tools** (Ubuntu ইনস্টলার সাধারণত নিজেই বসায়; না থাকলে):
   ```bash
   $ sudo apt install -y open-vm-tools open-vm-tools-desktop
   ```

| বিষয় | VirtualBox | VMware Workstation Pro |
|---|---|---|
| দাম | ফ্রি, ওপেন সোর্স (মূল অংশ) | ফ্রি (ক্লোজড সোর্স) |
| ডিস্ক ফরম্যাট | `.vdi` (`.vmdk`ও পারে) | `.vmdk` |
| Guest টুল | Guest Additions | open-vm-tools |
| 3D/গ্রাফিক্স | মোটামুটি | সাধারণত ভালো |
| Hyper-V-এর সঙ্গে | ধীর হতে পারে | ভালো চলে |

> 📝 **লিনাক্স host হলে:** সবচেয়ে ভালো পারফরম্যান্স **KVM/QEMU** (kernel-এর ভেতরের hypervisor) + **virt-manager** GUI দিয়ে। [অধ্যায় ৪৮](48-containers-virtualization.md)-এ দেখবে।

---

## ৩.৮ পদ্ধতি ২: WSL2: Windows-এর ভেতরে লিনাক্স

### WSL2 কী এবং ভেতরে কীভাবে কাজ করে?

**WSL** (Windows Subsystem for Linux) মাইক্রোসফটের একটা ফিচার, যা দিয়ে Windows-এর ভেতরেই আসল লিনাক্স চালানো যায়, আলাদা VM সফটওয়্যার ছাড়া।

- **WSL1** লিনাক্স system call-গুলোকে Windows kernel-এর call-এ "অনুবাদ" করত। কোনো আসল Linux kernel ছিল না।
- **WSL2** একটা হালকা, বিশেষভাবে বানানো **utility VM**-এর ভেতরে মাইক্রোসফটের রক্ষণাবেক্ষণ করা **আসল Linux kernel** চালায় (Hyper-V প্রযুক্তি দিয়ে)। তাই সামঞ্জস্য প্রায় ১০০%, Docker-ও চলে। চালু হতে মাত্র ১–২ সেকেন্ড লাগে।

```
┌────────────────────────── Windows 10/11 ───────────────────────────┐
│                                                                    │
│  Windows Terminal / PowerShell ──── wsl.exe                         │
│        │                                                           │
│        ▼                                                           │
│  ┌───────────────── হালকা Utility VM (Hyper-V প্ল্যাটফর্ম) ─────┐   │
│  │  আসল Linux kernel (মাইক্রোসফটের বিল্ড)                       │   │
│  │   ┌───────────────┐  ┌───────────────┐                      │   │
│  │   │ Ubuntu-24.04  │  │ Debian         │  ← প্রতিটা distro    │   │
│  │   │ (ext4 .vhdx)  │  │ (ext4 .vhdx)   │    আলাদা কন্টেইনারে   │   │
│  │   └───────────────┘  └───────────────┘                      │   │
│  │   WSLg: Wayland/X11 GUI অ্যাপ Windows ডেস্কটপে দেখায়         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│   C:\  ◄── /mnt/c হিসেবে লিনাক্স থেকে দেখা যায়                     │
│   \\wsl.localhost\Ubuntu-24.04 ◄── Windows Explorer থেকে লিনাক্স ফাইল │
└────────────────────────────────────────────────────────────────────┘
```

### ইনস্টল (Windows 11, বা Windows 10 ভার্সন 2004 / build 19041 বা তার পরে)

**Administrator** হিসেবে PowerShell বা Terminal খোলো (Start → "PowerShell" → Run as administrator):

```powershell
PS> wsl --install
```

এই একটা কমান্ডই প্রয়োজনীয় Windows ফিচার (Virtual Machine Platform, WSL) চালু করে, kernel নামায় আর ডিফল্ট distro (Ubuntu) বসায়। শেষে **রিস্টার্ট** করো। রিস্টার্টের পর Ubuntu উইন্ডো খুলে ইউজারনেম আর পাসওয়ার্ড চাইবে (এই পাসওয়ার্ড Windows-এর পাসওয়ার্ড না, লিনাক্সের `sudo`-র জন্য)।

নির্দিষ্ট distro চাইলে:

```powershell
PS> wsl --list --online
The following is a list of valid distributions that can be installed.
NAME                 FRIENDLY NAME
Ubuntu               Ubuntu
Ubuntu-24.04         Ubuntu 24.04 LTS
Debian               Debian GNU/Linux
...
PS> wsl --install -d Ubuntu-24.04
```

### গুরুত্বপূর্ণ `wsl` কমান্ড

| কমান্ড | কাজ |
|---|---|
| `wsl --install` | WSL + ডিফল্ট distro ইনস্টল |
| `wsl --install -d <Distro>` | নির্দিষ্ট distro ইনস্টল |
| `wsl -l -o` / `wsl --list --online` | ইনস্টলযোগ্য distro-র তালিকা |
| `wsl -l -v` / `wsl --list --verbose` | ইনস্টল করা distro, অবস্থা (Running/Stopped) ও WSL ভার্সন |
| `wsl` | ডিফল্ট distro-তে ঢোকা |
| `wsl -d <Distro>` | নির্দিষ্ট distro-তে ঢোকা |
| `wsl --set-default <Distro>` | ডিফল্ট distro বদলানো |
| `wsl --set-version <Distro> 2` | কোনো distro-কে WSL2-তে রূপান্তর |
| `wsl --set-default-version 2` | নতুন distro সবসময় WSL2 |
| `wsl --shutdown` | সব distro ও utility VM বন্ধ |
| `wsl --terminate <Distro>` | একটা distro বন্ধ |
| `wsl --update` | WSL ও kernel আপডেট |
| `wsl --status` / `wsl --version` | অবস্থা ও ভার্সন দেখা |
| `wsl --export <Distro> file.tar` | distro-র ব্যাকআপ |
| `wsl --import <Name> <Folder> file.tar` | ব্যাকআপ থেকে ফেরত/কপি |
| `wsl --unregister <Distro>` | ⚠️ distro ও এর **সব ডেটা মুছে ফেলা** |

```powershell
PS> wsl -l -v
  NAME            STATE           VERSION
* Ubuntu-24.04    Running         2
```

### WSL2 ব্যবহারের টিপস

- **ফাইল কোথায় রাখবে:** লিনাক্সের কাজ লিনাক্স ফাইলসিস্টেমে রাখো (`~/projects`), `/mnt/c/...`-তে না। Windows ড্রাইভ জুড়ে ফাইল অ্যাক্সেস অনেক ধীর।
- লিনাক্স থেকে বর্তমান ফোল্ডার Windows Explorer-এ খুলতে: `explorer.exe .`
- **systemd:** নতুন Ubuntu WSL ইনস্টলে systemd ডিফল্টে চালু থাকে। না থাকলে `/etc/wsl.conf`-এ লেখো:
  ```ini
  [boot]
  systemd=true
  ```
  তারপর PowerShell-এ `wsl --shutdown` দিয়ে আবার চালু করো। (systemd নিয়ে [অধ্যায় ২৬](26-systemd-services.md))
- **GUI অ্যাপ:** Windows 11 (ও আপডেটেড Windows 10)-এ **WSLg** দিয়ে লিনাক্স GUI অ্যাপ (যেমন `gedit`) সরাসরি চলে।
- **RAM/CPU সীমা:** Windows-এ `%UserProfile%\.wslconfig` ফাইলে:
  ```ini
  [wsl2]
  memory=4GB
  processors=2
  ```

> ⚠️ **সতর্কতা:** WSL2-তে বুট প্রসেস (GRUB, BIOS), ডিস্ক পার্টিশন, আসল নেটওয়ার্ক ইন্টারফেস ইত্যাদি পুরোপুরি শেখা যায় না, কারণ সেগুলো মাইক্রোসফট সামলায়। [অধ্যায় ২৭](27-boot-process.md), [৩২](32-disk-partition-mount.md), [৩৩](33-lvm-raid.md) চর্চার জন্য একটা পূর্ণ VM রাখো।

---

## ৩.৯ পদ্ধতি ৩: Live USB তৈরি

### Live USB কী?

**Live USB** = এমন একটা পেনড্রাইভ, যেটা থেকে কম্পিউটার বুট করলে লিনাক্স **হার্ড ডিস্কে কিছু না লিখেই** সরাসরি RAM থেকে চলে। কাজ:

1. ইনস্টলের আগে "চেখে দেখা": Wi-Fi, সাউন্ড, টাচপ্যাড চলে কিনা
2. আসল কম্পিউটারে ইনস্টল করা (dual boot বা পুরো ডিস্ক)
3. **রেসকিউ:** নষ্ট সিস্টেম থেকে ফাইল উদ্ধার, পাসওয়ার্ড রিসেট, GRUB মেরামত

```
Live USB বুট হলে ভেতরে কী হয়:

 USB ──► UEFI/BIOS ──► bootloader (GRUB) ──► kernel + initramfs RAM-এ লোড
                                                      │
                     কম্প্রেসড ফাইলসিস্টেম (squashfs) ◄─┘ শুধু-পড়া হিসেবে mount
                                                      │
                     ওপরে RAM-এ লেখার স্তর (overlay) ──► তুমি যা বদলাও তা RAM-এ,
                                                        রিবুট দিলে মুছে যায়
```

**লাগবে:** কমপক্ষে ৮ GB পেনড্রাইভ (এর **সব ডেটা মুছে যাবে**), যাচাই করা ISO।

### টুল তুলনা

| টুল | প্ল্যাটফর্ম | বৈশিষ্ট্য |
|---|---|---|
| **Rufus** | Windows | দ্রুত, ছোট; GPT/MBR নিজে বেছে দিতে দেয় |
| **balenaEtcher** | Windows, macOS, Linux | সবচেয়ে সহজ, ৩ ক্লিক, লেখার পর যাচাই করে |
| **Ventoy** | Windows, Linux | একবার USB-তে বসালে শুধু ISO ফাইল কপি করলেই হয়; একটা USB-তে অনেক ISO |
| **Fedora Media Writer** | সব | Fedora-র অফিশিয়াল |
| **GNOME Disks / Startup Disk Creator** | Linux | GUI, Ubuntu-তে বিল্ট-ইন |
| **`dd`** | Linux/macOS | কমান্ড লাইন, শক্তিশালী কিন্তু বিপজ্জনক |

### Rufus দিয়ে (Windows)

1. `https://rufus.ie` থেকে নামিয়ে চালাও (portable ভার্সনে ইনস্টল লাগে না)
2. **Device**: তোমার USB (নাম ও সাইজ দেখে নিশ্চিত হও!)
3. **Boot selection**: **SELECT** → ISO
4. **Partition scheme**: আধুনিক কম্পিউটার (২০১২-এর পরের) হলে **GPT**, target system **UEFI (non CSM)**; খুব পুরোনো BIOS কম্পিউটার হলে **MBR**
5. **File system**: ডিফল্ট (FAT32) রাখো
6. **START**। কোনো কোনো ISO-র ক্ষেত্রে মোড জিজ্ঞেস করবে: **ISO Image mode** (প্রস্তাবিত) বা **DD Image mode**। ISO mode-এ সমস্যা হলে DD mode চেষ্টা করো।
7. "সব ডেটা মুছে যাবে" সতর্কতায় OK

### balenaEtcher দিয়ে

1. `https://etcher.balena.io` থেকে নামাও
2. **Flash from file** → ISO
3. **Select target** → USB
4. **Flash!** → লেখা শেষে নিজেই **Validating** করে

### `dd` দিয়ে (লিনাক্স)

`dd` একটা লো-লেভেল কপি টুল: বাইট-বাই-বাইট হুবহু কপি করে। বিস্তারিত [অধ্যায় ৩৫](35-backup-rsync.md)-এ।

**ধাপ ১: USB-র ডিভাইস নাম খুঁজে বের করো** (USB লাগানোর আগে ও পরে চালিয়ে তুলনা করো):

```bash
$ lsblk -o NAME,SIZE,TYPE,TRAN,MODEL,MOUNTPOINTS
NAME          SIZE TYPE TRAN   MODEL              MOUNTPOINTS
nvme0n1     476.9G disk nvme   Samsung SSD 980
├─nvme0n1p1   512M part                           /boot/efi
└─nvme0n1p2 476.4G part                           /
sdb          14.6G disk usb    SanDisk Ultra
└─sdb1       14.6G part                           /media/rahim/USB
```

এখানে `TRAN` = `usb` আর সাইজ ১৪.৬G দেখে বোঝা যাচ্ছে USB হলো **`/dev/sdb`**।

**ধাপ ২: USB-র পার্টিশন unmount করো:**

```bash
$ sudo umount /dev/sdb1
```

**ধাপ ৩: ISO লেখো** (পার্টিশন `sdb1` না, পুরো ডিস্ক **`sdb`**):

```bash
$ sudo dd if=ubuntu-24.04.3-desktop-amd64.iso of=/dev/sdb bs=4M status=progress oflag=sync
6234832896 bytes (6.2 GB, 5.8 GiB) copied, 412 s, 15.1 MB/s
1486+1 records in
1486+1 records out
6237151232 bytes (6.2 GB, 5.8 GiB) copied, 413.5 s, 15.1 MB/s
$ sync
```

| `dd` অপশন | মানে |
|---|---|
| `if=FILE` | input file: কোথা থেকে পড়বে |
| `of=FILE` | output file: কোথায় লিখবে (এখানে ডিভাইস) |
| `bs=4M` | block size: একবারে ৪ MB করে পড়া-লেখা (দ্রুততর) |
| `status=progress` | চলাকালীন অগ্রগতি দেখায় |
| `oflag=sync` | প্রতিটা ব্লক সত্যিই ডিস্কে লেখা নিশ্চিত করে |
| `conv=fsync` | শেষে একবার সব ডেটা ডিস্কে flush করে (বিকল্প) |

> ⚠️ **ভয়ংকর সতর্কতা:** `dd`-কে লোকে মজা করে **"disk destroyer"** বলে। `of=` এ ভুল ডিভাইস (যেমন `/dev/nvme0n1` বা `/dev/sda`) দিলে কোনো প্রশ্ন না করেই **তোমার মূল ডিস্ক মুছে যাবে**, কোনো undo নেই। কমান্ড চালানোর আগে `lsblk` দিয়ে **দুইবার** দেখো।

> 📝 **কেন পার্টিশন (`sdb1`) না, পুরো ডিস্ক (`sdb`)?** আধুনিক লিনাক্স ISO-গুলো **hybrid ISO**: এর ভেতরে নিজস্ব পার্টিশন টেবিল ও bootloader আছে। তাই পুরো ডিস্কের শুরু থেকে হুবহু বসাতে হয়।

### USB থেকে বুট করা

1. USB লাগিয়ে কম্পিউটার রিস্টার্ট দাও
2. বুটের শুরুতে **Boot Menu key** বারবার চাপো (নিচের টেবিল)
3. তালিকা থেকে USB বেছে নাও (UEFI হলে নামের আগে "UEFI:" লেখা থাকে, সেটা বাছাই করো)

**সাধারণ BIOS/UEFI ও Boot Menu কী** (মডেল ভেদে বদলায়; স্ক্রিনে লেখা দেখে নিশ্চিত হও):

| ব্র্যান্ড | Boot Menu | BIOS/UEFI সেটআপ |
|---|---|---|
| Dell | `F12` | `F2` |
| HP | `F9` (বা `Esc` → মেনু) | `F10` (বা `Esc`) |
| Lenovo (ThinkPad/IdeaPad) | `F12` (কিছুতে Novo বাটন) | `F1` / `F2` |
| ASUS | `F8` বা `Esc` | `F2` / `Del` |
| Acer | `F12` (BIOS-এ চালু করতে হতে পারে) | `F2` |
| MSI | `F11` | `Del` |
| Gigabyte | `F12` | `Del` |
| Apple Intel Mac | `Option (⌥)` চেপে ধরে রাখো | — |

> 💡 **টিপস:** Windows থেকেও UEFI-তে যাওয়া যায়: Settings → System → Recovery → **Advanced startup → Restart now** → Troubleshoot → Advanced options → **UEFI Firmware Settings**। অথবা `Shift` চেপে ধরে Restart।

---

## ৩.১০ পার্টিশনিং-এর বেসিক (ইনস্টলের আগে জানা জরুরি)

পার্টিশন, ফাইলসিস্টেম, mount পুরো বিস্তারিত পাবে [অধ্যায় ৩২](32-disk-partition-mount.md)-এ। এখানে শুধু ইনস্টলের জন্য যতটুকু দরকার।

> 💡 **উপমা:** একটা হার্ড ডিস্ক একটা বড় জমির মতো। **পার্টিশন** হলো জমিকে দেয়াল দিয়ে কয়েকটা প্লটে ভাগ করা। **ফাইলসিস্টেম** (ext4, NTFS) হলো প্রতিটা প্লটে কীভাবে ঘর-তাক বানিয়ে জিনিস সাজানো হবে তার নকশা। **Partition table** (MBR/GPT) হলো জমির দলিল, যেখানে লেখা কোন প্লট কোথা থেকে কোথা পর্যন্ত।

### BIOS বনাম UEFI

| | **BIOS (Legacy)** | **UEFI** |
|---|---|---|
| যুগ | ১৯৮০-এর দশক থেকে | ~২০১০ থেকে আধুনিক সব PC |
| বুট কীভাবে | ডিস্কের প্রথম সেক্টর (MBR)-এর ছোট কোড চালায় | **ESP** পার্টিশনের `.efi` ফাইল চালায় |
| পার্টিশন টেবিল | সাধারণত MBR | GPT |
| Secure Boot | নেই | আছে |

### MBR বনাম GPT

| | **MBR** | **GPT** |
|---|---|---|
| সর্বোচ্চ ডিস্ক সাইজ | ২ TiB (৫১২-বাইট সেক্টরে) | কার্যত অসীম (৯.৪ ZB) |
| পার্টিশন সংখ্যা | ৪টি primary (বা ৩ primary + ১ extended, তার ভেতরে logical) | সাধারণত ১২৮টি |
| ব্যাকআপ টেবিল | নেই | ডিস্কের শেষে ব্যাকআপ কপি থাকে |
| ব্যবহার | পুরোনো BIOS সিস্টেম | ✅ আধুনিক UEFI সিস্টেম |

### লিনাক্সে ডিস্ক ও পার্টিশনের নাম

| নাম | মানে |
|---|---|
| `/dev/sda`, `/dev/sdb` | প্রথম, দ্বিতীয় SATA/SCSI/USB ডিস্ক |
| `/dev/sda1`, `/dev/sda2` | `sda`-র প্রথম, দ্বিতীয় পার্টিশন |
| `/dev/nvme0n1` | প্রথম NVMe SSD (controller 0, namespace 1) |
| `/dev/nvme0n1p1` | NVMe-র প্রথম পার্টিশন (`p` যোগ হয়) |
| `/dev/vda` | KVM VM-এর virtio ডিস্ক |
| `/dev/mmcblk0p1` | SD কার্ড/eMMC-র পার্টিশন |

### লিনাক্স ইনস্টলের জন্য সাধারণ পার্টিশন বিন্যাস (UEFI + GPT)

```
 ┌──────────┬─────────────┬────────────────────────┬───────────────────────────┐
 │  ESP     │  (ঐচ্ছিক)    │         /              │      /home (ঐচ্ছিক)         │
 │  FAT32   │  /boot      │       ext4/btrfs       │       ext4                │
 │ 512M–1G  │  1G ext4    │     ৩০–৫০ GB+          │     বাকি সব                │
 │/boot/efi │             │  OS + প্রোগ্রাম          │  তোমার ব্যক্তিগত ফাইল        │
 └──────────┴─────────────┴────────────────────────┴───────────────────────────┘
   + swap: আলাদা পার্টিশন, বা (আধুনিক distro-তে) একটা swapfile / zram
```

| মাউন্ট পয়েন্ট | ফাইলসিস্টেম | সাইজ (প্রস্তাবিত) | কাজ |
|---|---|---|---|
| `/boot/efi` (**ESP**) | FAT32 | ৫১২ MB – ১ GB | UEFI bootloader ফাইল। dual boot-এ Windows-এর ESP-টাই শেয়ার করা যায় |
| `/boot` | ext4 | ~১ GB (ঐচ্ছিক) | kernel ও initramfs; LVM/এনক্রিপশন থাকলে আলাদা লাগে |
| `/` (root) | ext4 / Btrfs / XFS | ৩০–৫০ GB+ | পুরো OS |
| `/home` | ext4 | বাকি জায়গা (ঐচ্ছিক) | ইউজারের ফাইল; আলাদা রাখলে OS পুনরায় ইনস্টলেও ফাইল থাকে |
| **swap** | swap | নিচে দেখো | RAM ভরে গেলে অতিরিক্ত জায়গা; **hibernate**-এর জন্য RAM-এর সমান বা বেশি |

**Swap কতটুকু?** আধুনিক নিয়ম মোটামুটি:

| RAM | swap (hibernate ছাড়া) | hibernate চাইলে |
|---|---|---|
| ≤ ২ GB | RAM-এর ২ গুণ | RAM-এর ৩ গুণ |
| ২–৮ GB | RAM-এর সমান | RAM-এর ২ গুণ |
| ৮–৬৪ GB | কমপক্ষে ৪ GB | RAM-এর ১.৫ গুণ |

> 📝 **distro-ভেদে ডিফল্ট:**
> - **Ubuntu**: ESP + একটা বড় `/` (ext4) পার্টিশন; swap হিসেবে `/swap.img` **swapfile**।
> - **Fedora Workstation**: **Btrfs** ফাইলসিস্টেম, `root` ও `home` আলাদা **subvolume** (একই পার্টিশনে); swap হিসেবে **zram** (RAM-এর ভেতরে কম্প্রেসড swap)।
> - **RHEL/Rocky/Alma**: **XFS** ফাইলসিস্টেম + **LVM** ডিফল্ট।
> - **openSUSE**: Btrfs + Snapper স্ন্যাপশট।

> 💡 **fresher-এর পরামর্শ:** VM-এ বা পুরো ডিস্কে ইনস্টল করলে ইনস্টলারের **স্বয়ংক্রিয় পার্টিশনিং** (Erase disk) নাও। ম্যানুয়াল পার্টিশনিং অভ্যাসের জন্য আলাদা একটা VM-এ চেষ্টা করো।

---

## ৩.১১ পদ্ধতি ৪: Dual Boot (Windows + Linux)

**Dual boot** মানে একই কম্পিউটারের ডিস্কে দুটো OS। চালু করার সময় **GRUB** মেনু জিজ্ঞেস করবে কোনটায় যাবে।

```
 পাওয়ার অন ──► UEFI firmware ──► ESP (/boot/efi)
                                    │
                    ┌───────────────┴────────────────┐
                    ▼                                ▼
          EFI/ubuntu/shimx64.efi              EFI/Microsoft/Boot/bootmgfw.efi
                    │                                ▲
                    ▼                                │
                GRUB মেনু ──── "Windows Boot Manager" ┘
                    │
                    └──► Ubuntu kernel ──► লিনাক্স
```

বুট প্রসেসের গভীরে যাবো [অধ্যায় ২৭](27-boot-process.md)-এ।

### ধাপ ০: নিরাপত্তার প্রস্তুতি (বাদ দিও না!)

1. **পুরো ব্যাকআপ নাও।** পার্টিশনে ভুল হলে ডেটা যেতে পারে।
2. **BitLocker রিকভারি কী** সংগ্রহ করো (Windows 11-এ অনেক ল্যাপটপে ডিভাইস এনক্রিপশন ডিফল্টে চালু)। কী পাবে `https://account.microsoft.com/devices/recoverykey`-এ, বা Command Prompt (Admin)-এ `manage-bde -status` দিয়ে অবস্থা দেখো। প্রয়োজনে ইনস্টলের সময় BitLocker সাময়িক **Suspend** করো। নাহলে বুট বদলানোর পর Windows রিকভারি কী চাইবে।
3. **Fast Startup বন্ধ করো:** Control Panel → Power Options → *Choose what the power buttons do* → *Change settings that are currently unavailable* → ❌ *Turn on fast startup*। কারণ Fast Startup আসলে Windows-কে আধা-hibernate অবস্থায় রাখে; NTFS পার্টিশন "লক" থাকে, লিনাক্স থেকে লিখলে ডেটা নষ্ট হতে পারে।
4. **ডিস্ক মোড:** কিছু Intel ল্যাপটপে স্টোরেজ মোড **RST/RAID/VMD**-এ থাকে, তখন লিনাক্স ইনস্টলার ডিস্ক দেখতে পায় না। সমাধান সাধারণত UEFI-তে **AHCI** মোডে নেওয়া, কিন্তু তার আগে Windows-কে Safe Mode দিয়ে প্রস্তুত না করলে Windows বুট হবে না। এটা ঝামেলার; প্রয়োজন হলে ভালো করে গাইড পড়ে করো।
5. **Secure Boot**: Ubuntu, Fedora, Debian, RHEL-পরিবার, openSUSE সাইন করা **shim** bootloader দিয়ে Secure Boot চালু রেখেই বুট করে। বন্ধ করার দরকার নেই। (NVIDIA-র মতো বাইরের kernel মডিউল বসালে **MOK** কী enroll করতে বলতে পারে, স্ক্রিনের নির্দেশ মানো।)

### ধাপ ১: Windows থেকে জায়গা খালি করা

1. `Win+X` → **Disk Management**
2. C: ড্রাইভে রাইট-ক্লিক → **Shrink Volume**
3. কত MB কমাবে লেখো (যেমন ৮০ GB = `81920` MB) → Shrink
4. এখন **Unallocated** কালো অংশ দেখা যাবে। এখানে কোনো পার্টিশন **বানাবে না**, লিনাক্স ইনস্টলার নিজেই ব্যবহার করবে।

### ধাপ ২: Live USB থেকে বুট ও ইনস্টল

ইনস্টলারে Disk setup ধাপে:

- **"Install Ubuntu alongside Windows Boot Manager"** অপশন দেখালে সেটাই সবচেয়ে সহজ: খালি জায়গায় নিজেই পার্টিশন বানায়, আর বিদ্যমান ESP-তে GRUB বসায়।
- না দেখালে **Manual installation**:
  - Unallocated জায়গা সিলেক্ট → `+` → সাইজ সব, ফরম্যাট **ext4**, mount `/`
  - বিদ্যমান ESP (FAT32, ~১০০–৫০০ MB) কে **ফরম্যাট না করে** `/boot/efi` হিসেবে নির্বাচন (অনেক সময় ইনস্টলার নিজেই করে)
  - **Device for boot loader installation**: পুরো ডিস্ক (যেমন `/dev/nvme0n1`)

> ⚠️ **সতর্কতা:** **"Erase disk and install Ubuntu"** বেছে নিলে **Windows সহ পুরো ডিস্ক মুছে যাবে**। dual boot-এ এটা কখনোই না। আর Windows-এর ESP **কখনো ফরম্যাট করবে না**, করলে Windows বুট হবে না।

### ধাপ ৩: রিবুট ও GRUB

রিবুটে GRUB মেনুতে Ubuntu আর Windows Boot Manager দুটোই থাকা উচিত। Windows না দেখালে, Ubuntu-তে ঢুকে:

```bash
$ sudo nano /etc/default/grub
# এই লাইনটা যোগ করো বা কমেন্ট (#) সরাও:
GRUB_DISABLE_OS_PROBER=false
$ sudo apt install os-prober
$ sudo update-grub
...
Found Windows Boot Manager on /dev/nvme0n1p1@/EFI/Microsoft/Boot/bootmgfw.efi
done
```

> 📝 Fedora/RHEL-পরিবারে `update-grub` নেই; সেখানে `sudo grub2-mkconfig -o /boot/grub2/grub.cfg`।

### Dual boot-এর সাধারণ সমস্যা

| সমস্যা | কারণ | সমাধান |
|---|---|---|
| লিনাক্সে গেলে Windows-এর ঘড়ি ভুল | Windows হার্ডওয়্যার ঘড়িকে local time ধরে, লিনাক্স UTC ধরে | লিনাক্সে `timedatectl set-local-rtc 1` (বা Windows-কে UTC-তে নাও) |
| GRUB না এসে সরাসরি Windows চালু হয় | UEFI boot order-এ Windows প্রথমে | UEFI সেটআপে "ubuntu" প্রথমে রাখো |
| Windows আপডেটের পর GRUB উধাও | Windows boot order বদলেছে | UEFI-তে আবার ঠিক করো, বা Live USB থেকে GRUB মেরামত (Boot-Repair টুল) |
| ইনস্টলার ডিস্ক দেখে না | RST/VMD মোড | ধাপ ০-এর ৪ নম্বর |
| BitLocker রিকভারি কী চাইছে | বুট চেইন বদলেছে | সংরক্ষিত কী দাও |

---

## ৩.১২ প্রথম বুটের পরে: প্রথম দিনের চেকলিস্ট

ইনস্টল শেষ, লগইন করেছো। এবার টার্মিনাল খোলো (`Ctrl+Alt+T` Ubuntu-তে; টার্মিনাল নিয়ে বিস্তারিত [অধ্যায় ৫](05-terminal-o-shell.md))।

### ১. কোন সিস্টেমে আছো, জানো

```bash
$ cat /etc/os-release
PRETTY_NAME="Ubuntu 24.04.3 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.3 LTS (Noble Numbat)"
VERSION_CODENAME=noble
ID=ubuntu
ID_LIKE=debian
...
```

`ID_LIKE=debian` দেখে বোঝা যায় এটা Debian পরিবারের, তাই `apt` চলবে। Rocky-তে দেখবে `ID_LIKE="rhel centos fedora"`।

```bash
$ hostnamectl
 Static hostname: ubuntu-lab
       Icon name: computer-vm
         Chassis: vm 🖴
Operating System: Ubuntu 24.04.3 LTS
          Kernel: Linux 6.8.0-xx-generic
    Architecture: x86-64
 Hardware Vendor: innotek GmbH
  Hardware Model: VirtualBox
...
$ uname -r
6.8.0-xx-generic
```

| কমান্ড | কী দেখায় |
|---|---|
| `cat /etc/os-release` | distro-র নাম, ভার্সন, পরিবার (সব distro-তে থাকে) |
| `lsb_release -a` | distro তথ্য (প্যাকেজ `lsb-release` থাকলে) |
| `hostnamectl` | hostname, OS, kernel, হার্ডওয়্যার/VM কিনা |
| `uname -r` | kernel ভার্সন |
| `uname -a` | kernel-এর সব তথ্য |

### ২. সিস্টেম আপডেট: সবচেয়ে জরুরি কাজ

ISO বানানোর পর থেকে অনেক নিরাপত্তা আপডেট এসেছে। তাই **প্রথম কাজ আপডেট।**

**কেন দুই ধাপে (Debian/Ubuntu)?**

```
 sudo apt update   ──► ইন্টারনেটের রিপোজিটরি থেকে "কোন প্যাকেজের কোন ভার্সন আছে"
                       তার তালিকা নামায় (দোকানের নতুন ক্যাটালগ আনা)
                       কিছু ইনস্টল করে না!
 sudo apt upgrade  ──► তালিকা মিলিয়ে ইনস্টল করা প্যাকেজগুলো নতুন ভার্সনে আপগ্রেড
                       (ক্যাটালগ দেখে আসলে জিনিস কেনা)
```

**Ubuntu / Debian / Mint:**

```bash
$ sudo apt update
[sudo] password for rahim:
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
Get:2 http://security.ubuntu.com/ubuntu noble-security InRelease [126 kB]
...
87 packages can be upgraded. Run 'apt list --upgradable' to see them.

$ sudo apt upgrade -y
...
$ sudo apt full-upgrade -y      # দরকার হলে প্যাকেজ সরিয়ে/নতুন যোগ করে আপগ্রেড (kernel ইত্যাদি)
$ sudo apt autoremove -y        # অপ্রয়োজনীয় পুরোনো প্যাকেজ সরানো
```

| কমান্ড | কাজ |
|---|---|
| `sudo apt update` | প্যাকেজ তালিকা রিফ্রেশ |
| `sudo apt upgrade` | প্যাকেজ আপগ্রেড (কিছু মুছবে না) |
| `sudo apt full-upgrade` | প্রয়োজনে প্যাকেজ মুছে/যোগ করে পূর্ণ আপগ্রেড |
| `sudo apt autoremove` | অনাথ নির্ভরতা প্যাকেজ মুছে ফেলা |
| `apt list --upgradable` | কোনগুলো আপগ্রেড হবে দেখা |
| `-y` | সব প্রশ্নে স্বয়ংক্রিয় "yes" |

**অন্য distro:**

| Distro | আপডেট কমান্ড |
|---|---|
| Fedora | `sudo dnf upgrade --refresh` |
| RHEL | প্রথমে `sudo subscription-manager register`, তারপর `sudo dnf update` |
| Rocky / Alma / CentOS Stream | `sudo dnf update` (`dnf upgrade`-এর সমার্থক) |
| Arch / Manjaro | `sudo pacman -Syu` |
| openSUSE Tumbleweed | `sudo zypper refresh && sudo zypper dup` |
| openSUSE Leap | `sudo zypper refresh && sudo zypper update` |

> 📝 Ubuntu-তে Snap অ্যাপগুলো আলাদা আপডেট হয় (স্বয়ংক্রিয়ভাবে দিনে কয়েকবার); হাতে করতে চাইলে `sudo snap refresh`। Flatpak থাকলে `flatpak update`।

**রিবুট দরকার কিনা?** kernel বা গুরুত্বপূর্ণ লাইব্রেরি আপডেট হলে রিবুট লাগে:

```bash
$ ls /var/run/reboot-required 2>/dev/null && cat /var/run/reboot-required
/var/run/reboot-required
*** System restart required ***
$ sudo reboot
```

(Fedora/RHEL-এ `sudo dnf needs-restarting -r`; এটা `dnf-plugins-core`/`dnf-utils` প্যাকেজের অংশ।)

### ৩. ড্রাইভার ও ফার্মওয়্যার (আসল হার্ডওয়্যারে)

```bash
$ ubuntu-drivers devices          # কোন হার্ডওয়্যারের জন্য কোন ড্রাইভার প্রস্তাবিত
$ sudo ubuntu-drivers install     # প্রস্তাবিত ড্রাইভার (যেমন NVIDIA) ইনস্টল
```

GUI-তে: **Software & Updates → Additional Drivers**। Fedora-তে NVIDIA ড্রাইভারের জন্য **RPM Fusion** রিপোজিটরি লাগে।

ফার্মওয়্যার (BIOS, SSD, টাচপ্যাড) আপডেট, সমর্থিত ডিভাইসে:

```bash
$ fwupdmgr refresh
$ fwupdmgr get-updates
$ fwupdmgr update
```

### ৪. প্রয়োজনীয় টুল বসানো

```bash
# Ubuntu/Debian
$ sudo apt install -y curl wget git vim htop tree build-essential
# Fedora/RHEL-পরিবার
$ sudo dnf install -y curl wget git vim htop tree
$ sudo dnf groupinstall -y "Development Tools"   # gcc, make ইত্যাদি
```

> 📝 RHEL/Rocky/Alma-তে `htop` ইত্যাদির জন্য আগে **EPEL** রিপোজিটরি লাগে: `sudo dnf install -y epel-release`।

### ৫. স্বয়ংক্রিয় নিরাপত্তা আপডেট (Ubuntu)

Ubuntu-তে **unattended-upgrades** সাধারণত ডিফল্টে চালু থাকে এবং নিরাপত্তা আপডেট নিজে বসায়। যাচাই/চালু করতে:

```bash
$ sudo apt install -y unattended-upgrades
$ sudo dpkg-reconfigure -plow unattended-upgrades
```

Fedora/RHEL-পরিবারে এর সমতুল্য `dnf-automatic`। [অধ্যায় ৪৫](45-security-hardening.md)-এ নিরাপত্তা নিয়ে বিস্তারিত।

### ৬. ফায়ারওয়াল ও ব্যাকআপ

- Ubuntu: `sudo ufw enable` (ডেস্কটপে সাধারণত ঝুঁকি কম, তবে চালু রাখা ভালো অভ্যাস); বিস্তারিত [অধ্যায় ৩৯](39-firewall.md)
- **Timeshift** (Mint-এ ডিফল্ট, Ubuntu-তে `sudo apt install timeshift`) দিয়ে সিস্টেম স্ন্যাপশট; VM-এ VirtualBox snapshot

### প্রথম দিনের চেকলিস্ট (সংক্ষেপে)

```
[ ] cat /etc/os-release দিয়ে distro চেনা
[ ] sudo apt update && sudo apt upgrade  (বা dnf/pacman/zypper)
[ ] প্রয়োজনে রিবুট
[ ] VM হলে Guest Additions / open-vm-tools
[ ] আসল হার্ডওয়্যারে ড্রাইভার + fwupd
[ ] বেসিক টুল (curl, git, vim, htop)
[ ] ফায়ারওয়াল চালু
[ ] snapshot / Timeshift ব্যাকআপ
```

---

## ৩.১৩ সংশ্লিষ্ট কীবোর্ড শর্টকাট

### VirtualBox (Host key = ডিফল্টে **ডান `Ctrl`**)

| শর্টকাট | কাজ |
|---|---|
| `Host` (ডান Ctrl) একবার | VM-এর আটকে থাকা মাউস/কীবোর্ড ছাড়ানো |
| `Host + F` | পূর্ণ স্ক্রিন (Full-screen) চালু/বন্ধ |
| `Host + L` | Seamless mode (guest-এর উইন্ডো host ডেস্কটপে মিশে যায়) |
| `Host + C` | Scaled mode |
| `Host + Home` | (Full-screen/Seamless-এ) VM মেনু দেখানো |
| `Host + Del` | guest-এ `Ctrl+Alt+Del` পাঠানো |
| `Host + Backspace` | guest-এ `Ctrl+Alt+Backspace` পাঠানো |
| `Host + T` | Snapshot নেওয়া |
| `Host + E` | Screenshot নেওয়া |
| `Host + P` | VM pause / resume |
| `Host + R` | VM রিসেট (হঠাৎ রিস্টার্ট, সাবধান) |
| `Host + H` | ACPI shutdown সিগন্যাল (ভদ্রভাবে বন্ধ) |
| `Host + Q` | VM বন্ধ করার ডায়ালগ |
| `Ctrl + N` (Manager-এ) | নতুন VM |

> 📝 guest-এর ভেতরে `Ctrl+Alt+F3` ইত্যাদি (virtual console) পাঠাতে VirtualBox-এ `Host + F3` চাপো, কারণ সরাসরি চাপলে host OS ধরে ফেলে। Virtual console নিয়ে [অধ্যায় ৫](05-terminal-o-shell.md)।

### VMware Workstation

| শর্টকাট | কাজ |
|---|---|
| `Ctrl + Alt` | মাউস/কীবোর্ড guest থেকে ছাড়ানো |
| `Ctrl + G` | guest-এ ইনপুট ধরানো (grab) |
| `Ctrl + Alt + Enter` | পূর্ণ স্ক্রিন চালু/বন্ধ |
| `Ctrl + Alt + Insert` | guest-এ `Ctrl+Alt+Del` পাঠানো |

### বুট সময়ের শর্টকাট

| শর্টকাট | কাজ |
|---|---|
| `F2` / `Del` / `F10` / `F1` | BIOS/UEFI সেটআপ (মডেল ভেদে; ৩.৯-এর টেবিল) |
| `F12` / `F11` / `F9` / `F8` / `Esc` | Boot menu (মডেল ভেদে) |
| `Shift` চেপে রাখা (BIOS বুট) / `Esc` চাপা (UEFI বুট) | লুকানো GRUB মেনু দেখানো (Ubuntu-তে একমাত্র OS হলে মেনু লুকানো থাকে) |
| `↑` / `↓` | GRUB-এ এন্ট্রি বাছাই |
| `e` | নির্বাচিত GRUB এন্ট্রি সাময়িক এডিট (kernel প্যারামিটার) |
| `Ctrl + X` বা `F10` | এডিট করা এন্ট্রি দিয়ে বুট |
| `c` | GRUB কমান্ড লাইন |
| `Esc` | এডিট বাতিল/আগের মেনু |
| Windows-এ `Shift` + Restart | Advanced startup (UEFI-তে যাওয়ার পথ) |

### Ubuntu ইনস্টলের পর কাজে লাগবে

| শর্টকাট | কাজ |
|---|---|
| `Ctrl + Alt + T` | টার্মিনাল খোলা (Ubuntu) |
| `Super` (Windows key) | Activities / অ্যাপ সার্চ |
| `Super + L` | স্ক্রিন লক |

### Windows (WSL প্রসঙ্গে)

| শর্টকাট | কাজ |
|---|---|
| `Win + X` | Power user মেনু (Disk Management, Terminal (Admin)) |
| `Win + R` → `wsl` | সরাসরি ডিফল্ট লিনাক্সে ঢোকা |
| `Ctrl + Shift + T` (Windows Terminal) | নতুন ট্যাব |

---

## হাতে-কলমে অনুশীলন

### ল্যাব ১: distro পরিচিতি (১০ মিনিট)

1. `https://distrowatch.com` খুলে প্রথম ১০টি distro দেখো। প্রত্যেকটার "Based on" ঘর থেকে লেখো কোন পরিবারের।
2. নিচের টেবিল পূরণ করো:

| Distro | পরিবার | প্যাকেজ ম্যানেজার | LTS/fixed না rolling |
|---|---|---|---|
| Linux Mint | | | |
| Fedora | | | |
| EndeavourOS | | | |
| AlmaLinux | | | |
| openSUSE Tumbleweed | | | |

### ল্যাব ২: ISO নামিয়ে যাচাই (১৫ মিনিট)

1. Ubuntu 24.04 LTS Desktop ISO আর একই পেজের `SHA256SUMS` ফাইল নামাও।
2. লিনাক্স/WSL/macOS-এ: `sha256sum -c SHA256SUMS --ignore-missing` চালাও। `OK` দেখা উচিত।
3. Windows-এ: `Get-FileHash` দিয়ে হ্যাশ বের করে `SHA256SUMS`-এর লাইনের সঙ্গে চোখে মেলাও।
4. **পরীক্ষা:** ISO-র একটা কপি বানিয়ে (`cp ubuntu.iso test.iso`) তার শেষে একটা বাইট যোগ করো: `echo x >> test.iso`। এবার `sha256sum test.iso` চালিয়ে দেখো হ্যাশ সম্পূর্ণ আলাদা। শেষে `rm test.iso`।

### ল্যাব ৩: প্রথম VM (৪৫–৬০ মিনিট)

1. ৩.৭-এর ধাপ মেনে VirtualBox-এ `ubuntu-lab` VM বানাও: ৪ GB RAM, ২ CPU, ৩০ GB ডিস্ক।
2. ইনস্টল শেষে টার্মিনালে:
   ```bash
   $ cat /etc/os-release
   $ hostnamectl
   $ uname -r
   $ lsblk
   ```
   `lsblk`-এর output দেখে লেখো: ডিস্কের নাম কী (`sda`), কয়টা পার্টিশন, কোনটা কোথায় mount।
3. `sudo apt update && sudo apt upgrade -y` চালাও। প্রয়োজনে রিবুট।
4. Guest Additions বসাও, উইন্ডো রিসাইজ করে দেখো স্ক্রিন বদলায় কিনা।
5. `fresh-install-updated` নামে snapshot নাও।

### ল্যাব ৪: snapshot-এর জাদু (১০ মিনিট)

1. VM-এ একটা ফাইল বানাও: `echo "আমি থাকব না" > ~/test.txt`
2. VM বন্ধ করে snapshot `fresh-install-updated` **Restore** করো।
3. চালু করে `ls ~/test.txt` দেখো। ফাইলটা নেই! এটাই নিরাপদ পরীক্ষার ভিত্তি।

### ল্যাব ৫ (Windows ব্যবহারকারীদের জন্য): WSL2 (২০ মিনিট)

1. Admin PowerShell-এ `wsl --install -d Ubuntu-24.04`, রিস্টার্ট, ইউজার বানাও।
2. `wsl -l -v` দিয়ে দেখো VERSION `2` কিনা।
3. লিনাক্সে ঢুকে `cat /etc/os-release`, `uname -r` (দেখবে kernel-এর নামে `microsoft-standard-WSL2` লেখা)।
4. `cd ~ && explorer.exe .` দিয়ে লিনাক্স হোম Windows Explorer-এ খোলো।
5. `ls /mnt/c` দিয়ে C: ড্রাইভ দেখো।
6. PowerShell-এ `wsl --shutdown`, তারপর আবার `wsl -l -v`। STATE `Stopped` দেখাবে।

### ল্যাব ৬ (ঐচ্ছিক, সাবধানে): Live USB (২০ মিনিট)

1. একটা খালি পেনড্রাইভ দিয়ে balenaEtcher বা Rufus দিয়ে Ubuntu Live USB বানাও।
2. লিনাক্স থাকলে `dd` দিয়ে করার আগে `lsblk` দিয়ে USB **নিশ্চিতভাবে** চিনে নাও।
3. Boot menu key দিয়ে USB থেকে বুট করে **"Try Ubuntu"** মোডে Wi-Fi, সাউন্ড, টাচপ্যাড পরীক্ষা করো। **Install চাপবে না।**

### ল্যাব ৭ (চ্যালেঞ্জ): দ্বিতীয় পরিবার

একটা **Rocky Linux** বা **AlmaLinux** VM বানাও (Minimal ISO-তে GUI নেই, শুধু টার্মিনাল)। ইনস্টলার (Anaconda) কীভাবে Ubuntu-র থেকে আলাদা লক্ষ করো। ইনস্টলের পর `sudo dnf update` চালাও আর `cat /etc/os-release`-এর `ID_LIKE` লক্ষ করো। ডিফল্ট ফাইলসিস্টেম দেখতে `lsblk -f` চালাও (XFS আর LVM দেখবে)।

---

## সারসংক্ষেপ

**মূল ধারণা:**

- **distro** = kernel + টুলস + প্যাকেজ ম্যানেজার + ইনস্টলার + নীতি
- চার বড় পরিবার: **Debian/Ubuntu** (`apt`, `.deb`), **Red Hat** (`dnf`, `.rpm`), **Arch** (`pacman`), **SUSE** (`zypper`, `.rpm`)
- **LTS/fixed** = স্থিতিশীল, সার্ভারের জন্য; **rolling** = সবসময় নতুন, অভিজ্ঞদের জন্য
- fresher-এর জন্য: **Ubuntu 24.04 LTS**, প্রথমে **VM**-এ
- পদ্ধতি: VM (সবচেয়ে নিরাপদ) → WSL2 (Windows-এ CLI) → Live USB (চেখে দেখা/রেসকিউ) → Dual boot (পূর্ণ গতি)
- আধুনিক PC = **UEFI + GPT**, লাগে **ESP** (`/boot/efi`, FAT32)
- ইনস্টলের পর প্রথম কাজ: **আপডেট**

**কমান্ড রেফারেন্স:**

| কমান্ড | কাজ |
|---|---|
| `sha256sum FILE` / `sha256sum -c SHA256SUMS --ignore-missing` | ISO-র হ্যাশ / যাচাই |
| `lsblk -o NAME,SIZE,TYPE,TRAN,MODEL` | ডিস্ক ও USB চেনা |
| `sudo dd if=X.iso of=/dev/sdX bs=4M status=progress oflag=sync` | Live USB লেখা (⚠️) |
| `wsl --install [-d Distro]` | WSL2 ইনস্টল |
| `wsl -l -v` / `wsl -l -o` | ইনস্টল করা / উপলব্ধ distro |
| `wsl --shutdown` | WSL বন্ধ |
| `cat /etc/os-release` | distro চেনা |
| `hostnamectl` / `uname -r` | সিস্টেম ও kernel তথ্য |
| `sudo apt update && sudo apt upgrade` | Debian/Ubuntu আপডেট |
| `sudo dnf upgrade --refresh` | Fedora/RHEL-পরিবার আপডেট |
| `sudo pacman -Syu` | Arch আপডেট |
| `sudo zypper dup` / `sudo zypper up` | Tumbleweed / Leap আপডেট |
| `sudo ubuntu-drivers install` | Ubuntu-তে প্রস্তাবিত ড্রাইভার |
| `fwupdmgr get-updates` / `update` | ফার্মওয়্যার আপডেট |
| `sudo update-grub` | GRUB মেনু পুনর্নির্মাণ (Debian/Ubuntu) |

**শর্টকাট রেফারেন্স:**

| শর্টকাট | কাজ |
|---|---|
| ডান `Ctrl` (VirtualBox Host) | মাউস ছাড়ানো |
| `Host+F` / `Host+T` / `Host+Del` | পূর্ণ স্ক্রিন / snapshot / Ctrl+Alt+Del |
| `Ctrl+Alt` (VMware) | মাউস ছাড়ানো |
| `F12`/`F11`/`F9`/`F8`/`Esc` | Boot menu (মডেল ভেদে) |
| `F2`/`Del` | BIOS/UEFI সেটআপ |
| `Shift` / `Esc` (বুটে) | GRUB মেনু দেখানো |
| `e` তারপর `Ctrl+X` (GRUB) | এন্ট্রি এডিট করে বুট |
| `Ctrl+Alt+T` | Ubuntu-তে টার্মিনাল |

---

## নিজেকে যাচাই করো

1. "Linux" আর "Linux distribution"-এর মধ্যে পার্থক্য কী?
2. Ubuntu, Linux Mint, Rocky Linux, Manjaro: প্রত্যেকটা কোন পরিবারের, এবং প্যাকেজ ইনস্টলের কমান্ড কী দিয়ে শুরু হয়?
3. LTS আর rolling release-এর একটা করে সুবিধা ও অসুবিধা বলো। প্রোডাকশন সার্ভারে কোনটা আর কেন?
4. Ubuntu `24.04`-এর `24` আর `04` কী বোঝায়? এটা LTS কিনা কীভাবে বুঝবে?
5. ISO নামানোর পর কেন `sha256sum` দিয়ে যাচাই করা উচিত?
6. `dd` দিয়ে Live USB বানানোর সময় `of=/dev/sdb1` না দিয়ে `of=/dev/sdb` কেন দিই? আর সবচেয়ে বড় বিপদ কী?
7. WSL1 আর WSL2-এর মূল প্রযুক্তিগত পার্থক্য কী?
8. UEFI সিস্টেমে ESP কী, কোন ফাইলসিস্টেমে থাকে, আর লিনাক্সে কোথায় mount হয়?
9. Dual boot করার আগে Windows-এ কোন তিনটি কাজ অবশ্যই করবে?
10. `sudo apt update` আর `sudo apt upgrade`-এর মধ্যে পার্থক্য কী?

<details>
<summary>উত্তর দেখো</summary>

1. **Linux** কঠোর অর্থে শুধু kernel। **Distribution** হলো kernel-এর সঙ্গে GNU টুলস, shell, লাইব্রেরি, init system, প্যাকেজ ম্যানেজার, ইনস্টলার, ডেস্কটপ ও অ্যাপ মিলিয়ে বানানো সম্পূর্ণ, ব্যবহারযোগ্য অপারেটিং সিস্টেম।
2. Ubuntu ও Linux Mint → Debian পরিবার → `apt`। Rocky Linux → Red Hat পরিবার → `dnf`। Manjaro → Arch পরিবার → `pacman`।
3. **LTS:** সুবিধা স্থিতিশীলতা ও দীর্ঘ সাপোর্ট; অসুবিধা সফটওয়্যার পুরোনো। **Rolling:** সুবিধা সবসময় সর্বশেষ সফটওয়্যার ও হার্ডওয়্যার সাপোর্ট; অসুবিধা আপডেটে মাঝে মাঝে কিছু ভাঙতে পারে, নিয়মিত মনোযোগ লাগে। প্রোডাকশন সার্ভারে **LTS/এন্টারপ্রাইজ**, কারণ সেখানে অপরিবর্তনীয়তা ও নির্ভরযোগ্যতা সবচেয়ে জরুরি।
4. `24` = ২০২৪ সাল, `04` = এপ্রিল মাস। Ubuntu-তে জোড় বছরের এপ্রিল রিলিজ (`.04`) হলো LTS; তাই 24.04 LTS। নিশ্চিত হতে `cat /etc/os-release`-এর VERSION-এ "LTS" দেখো।
5. ডাউনলোডের সময় ফাইল নষ্ট হতে পারে, বা কেউ ক্ষতিকর পরিবর্তিত ISO দিতে পারে। হ্যাশ অফিশিয়াল মানের সঙ্গে মিললে নিশ্চিত হওয়া যায় ফাইল হুবহু আসল। (GPG দিয়ে SHA256SUMS ফাইলের সিগনেচার যাচাই করলে নিরাপত্তা আরও বাড়ে।)
6. আধুনিক ISO **hybrid**: এর ভেতরে নিজস্ব পার্টিশন টেবিল ও bootloader আছে, তাই পুরো ডিস্কের শুরু থেকে লিখতে হয়। সবচেয়ে বড় বিপদ: `of=`-এ ভুল ডিস্ক দিলে কোনো সতর্কতা ছাড়াই সেই ডিস্কের সব ডেটা চিরতরে মুছে যায়। তাই আগে `lsblk` দিয়ে দুবার মিলিয়ে নিতে হয়।
7. **WSL1** লিনাক্স system call-কে Windows kernel-এর call-এ অনুবাদ করত, আসল Linux kernel ছিল না। **WSL2** একটা হালকা utility VM-এর ভেতরে আসল Linux kernel চালায়, তাই সামঞ্জস্য প্রায় পূর্ণ (Docker, systemd চলে), যদিও `/mnt/c`-এর মতো Windows ফাইল অ্যাক্সেস তুলনামূলক ধীর।
8. **ESP (EFI System Partition)** একটা ছোট পার্টিশন (~৫১২ MB–১ GB), যেখানে UEFI firmware যে `.efi` bootloader ফাইল চালায় সেগুলো থাকে। ফাইলসিস্টেম **FAT32** (vfat), লিনাক্সে mount হয় **`/boot/efi`**-তে।
9. (ক) পুরো ডেটার **ব্যাকআপ**; (খ) **BitLocker রিকভারি কী** সংরক্ষণ (বা সাময়িক suspend); (গ) **Fast Startup বন্ধ**। এরপর Disk Management দিয়ে **Shrink Volume** করে খালি জায়গা বানানো।
10. `apt update` শুধু রিপোজিটরি থেকে কোন প্যাকেজের কোন ভার্সন পাওয়া যাচ্ছে তার **তালিকা রিফ্রেশ** করে, কিছু ইনস্টল করে না। `apt upgrade` সেই তালিকা দেখে ইনস্টল করা প্যাকেজগুলোকে আসলে **নতুন ভার্সনে আপগ্রেড** করে। তাই আগে `update`, তারপর `upgrade`।

</details>

---

[← আগের অধ্যায়](02-linux-architecture.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](04-desktop-environment.md)

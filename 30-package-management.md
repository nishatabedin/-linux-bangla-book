# অধ্যায় ৩০: প্যাকেজ ম্যানেজমেন্ট — সফটওয়্যার ইনস্টল

## এই অধ্যায়ে যা শিখবে

- **package** (প্যাকেজ) আসলে কী, তার ভেতরে কী থাকে
- **dependency** (নির্ভরতা) কী, আর কেন এটা সামলানো কঠিন কাজ
- **repository** (রিপোজিটরি — সফটওয়্যারের অনলাইন গুদাম) কীভাবে কাজ করে
- low-level টুল (`dpkg`, `rpm`) আর high-level টুল (`apt`, `dnf`, `pacman`) এর পার্থক্য
- Debian/Ubuntu পরিবার: `apt` এর সব জরুরি সাবকমান্ড, `dpkg`, `sources.list`, deb822 ফরম্যাট, PPA, `apt-mark hold`
- RHEL/Fedora পরিবার: `dnf`/`yum`, `rpm`, `.repo` ফাইল
- Arch পরিবার: `pacman` ও AUR
- distro-নিরপেক্ষ প্যাকেজ: `snap`, `flatpak`, AppImage
- GPG key দিয়ে repository কীভাবে নিরাপদ রাখা হয়, আর পুরনো `apt-key` কেন বাদ দেওয়া হয়েছে
- প্যাকেজ সংক্রান্ত সাধারণ সমস্যা আর তার সমাধান

---

## ৩০.১ Windows বনাম Linux: সফটওয়্যার ইনস্টলের দুই দর্শন

Windows-এ সফটওয়্যার ইনস্টল করতে গেলে সাধারণত তুমি কী করো? Google-এ খোঁজো, কোনো ওয়েবসাইট থেকে `setup.exe` নামাও, "Next → Next → I Agree → Finish" চাপো। প্রতিটা সফটওয়্যার আলাদা জায়গা থেকে আসে, আলাদাভাবে আপডেট হয় (বা হয়ই না), আর কোনটা ভাইরাসযুক্ত সেটা বোঝার উপায় কম।

Linux-এর দর্শন একদম আলাদা। এখানে আছে **package manager** (প্যাকেজ ম্যানেজার — সফটওয়্যার ইনস্টল, আপডেট আর মুছে ফেলার কেন্দ্রীয় ব্যবস্থাপক)। তুমি শুধু বলো:

```bash
$ sudo apt install vlc
```

ব্যস। package manager নিজেই:

1. বিশ্বস্ত repository থেকে VLC খুঁজে বের করে
2. VLC চালাতে আর কী কী লাগবে (dependency) সেগুলো হিসাব করে
3. সবকিছু ডাউনলোড করে, ডিজিটাল সিগনেচার যাচাই করে
4. ঠিক জায়গায় ফাইলগুলো বসায়
5. একটা ডাটাবেসে লিখে রাখে কোন ফাইল কোন প্যাকেজের

পরে `sudo apt upgrade` দিলে সিস্টেমের **সব** সফটওয়্যার একসাথে আপডেট হয়ে যায় — কার্নেল থেকে ব্রাউজার পর্যন্ত।

> 💡 **উপমা:** Windows-এর পদ্ধতি হলো প্রতিটা জিনিস কিনতে আলাদা আলাদা দোকানে যাওয়া। Linux-এর package manager হলো একটা বিশ্বস্ত সুপারশপ, যেখানে একটা লিস্ট দিলে দোকানদার নিজেই সব জিনিস (আর সেগুলোর সাথে যা যা লাগে, যেমন টর্চের সাথে ব্যাটারি) প্যাক করে দেয়, আর মেয়াদ শেষ হলে নিজেই বদলে দেয়।

Android-এর Play Store বা iPhone-এর App Store আসলে এই Linux ধারণারই বংশধর — Debian-এর `apt` সেই ১৯৯৮ সাল থেকে এই কাজ করছে।

---

## ৩০.২ Package আসলে কী?

**Package** হলো একটা আর্কাইভ ফাইল (অনেকটা zip-এর মতো), যার ভেতরে থাকে:

| অংশ | কী থাকে | উদাহরণ |
|-----|---------|---------|
| প্রোগ্রামের ফাইল | compiled binary, library, কনফিগ, ডকুমেন্টেশন, man page | `/usr/bin/htop`, `/usr/share/man/man1/htop.1.gz` |
| metadata | নাম, version, লেখক, বর্ণনা, আকার, architecture | `Package: htop`, `Version: 3.3.0-4build1` |
| dependency তালিকা | এটা চালাতে আর কী কী প্যাকেজ লাগবে | `Depends: libc6, libncursesw6` |
| maintainer script | ইনস্টলের আগে/পরে চলার মতো ছোট স্ক্রিপ্ট | `preinst`, `postinst`, `prerm`, `postrm` |
| checksum | প্রতিটা ফাইলের হ্যাশ, যাতে নষ্ট হলে ধরা যায় | `md5sums` |

### Package ফাইলের ফরম্যাট

| পরিবার | ফরম্যাট | এক্সটেনশন | উদাহরণ ফাইলনাম |
|--------|---------|-----------|-----------------|
| Debian, Ubuntu, Linux Mint, Pop!_OS | deb | `.deb` | `htop_3.3.0-4build1_amd64.deb` |
| RHEL, Fedora, CentOS Stream, Rocky, AlmaLinux, openSUSE | RPM | `.rpm` | `htop-3.3.0-3.fc40.x86_64.rpm` |
| Arch, Manjaro, EndeavourOS | pacman package | `.pkg.tar.zst` | `htop-3.3.0-1-x86_64.pkg.tar.zst` |

ফাইলনামেই অনেক তথ্য লুকিয়ে আছে:

```
htop_3.3.0-4build1_amd64.deb
│     │     │       │
│     │     │       └── architecture (amd64 = 64-bit Intel/AMD)
│     │     └────────── Debian/Ubuntu revision (প্যাকেজিং-এর সংস্করণ)
│     └──────────────── upstream version (মূল সফটওয়্যারের version)
└────────────────────── প্যাকেজের নাম
```

> 📝 **মনে রাখো:** Debian `amd64` বলে, RPM-জগৎ একই জিনিসকে `x86_64` বলে। ARM 64-bit কে Debian বলে `arm64`, RPM বলে `aarch64`। `noarch` / `all` মানে প্যাকেজটা যেকোনো architecture-এ চলবে (যেমন Python স্ক্রিপ্ট বা ডকুমেন্টেশন)।

### ভেতরে উঁকি দেওয়া (under the hood)

একটা `.deb` ফাইল আসলে একটা `ar` আর্কাইভ, যার ভেতরে তিনটা জিনিস:

```
htop_3.3.0-4build1_amd64.deb   (ar archive)
├── debian-binary       → "2.0"  (ফরম্যাট version)
├── control.tar.zst     → metadata + maintainer scripts
└── data.tar.zst        → আসল ফাইলগুলো (/usr/bin/htop ...)
```

তুমি চাইলে নিজেই দেখতে পারো:

```bash
$ apt download htop          # শুধু .deb নামাবে, ইনস্টল করবে না
$ ar t htop_*.deb
debian-binary
control.tar.zst
data.tar.zst
$ dpkg-deb --info htop_*.deb      # metadata দেখাবে
$ dpkg-deb --contents htop_*.deb  # ভেতরের ফাইল তালিকা
```

`.rpm` ফাইলের ভেতরে থাকে একটা header (metadata) আর একটা cpio আর্কাইভ (payload)। `rpm2cpio file.rpm | cpio -idmv` দিয়ে ইনস্টল না করেই খোলা যায়।

---

## ৩০.৩ Dependency — নির্ভরতার জাল

**Dependency** মানে একটা প্রোগ্রাম চলার জন্য অন্য যে প্রোগ্রাম বা library লাগে। Linux-এ প্রায় সব প্রোগ্রামই **shared library** (ভাগাভাগি করা লাইব্রেরি — একাধিক প্রোগ্রাম একসাথে ব্যবহার করে এমন কোডের ফাইল, যেমন `libc.so.6`) ব্যবহার করে।

কেন? ধরো ১০০টা প্রোগ্রাম PNG ছবি পড়তে পারে। প্রত্যেকে যদি PNG পড়ার কোড নিজের ভেতরে রাখে, তাহলে:
- ১০০ গুণ ডিস্ক আর RAM খরচ
- PNG লাইব্রেরিতে নিরাপত্তা ত্রুটি পাওয়া গেলে ১০০টা প্রোগ্রাম আলাদাভাবে আপডেট করতে হবে

shared library থাকলে একবার `libpng` আপডেট করলেই ১০০টা প্রোগ্রাম নিরাপদ।

```
                    ┌──────────┐
                    │   vlc    │
                    └────┬─────┘
          ┌──────────────┼───────────────┐
          ▼              ▼               ▼
   ┌────────────┐ ┌────────────┐  ┌─────────────┐
   │ libvlc5    │ │ libqt5gui5 │  │ libavcodec  │
   └─────┬──────┘ └─────┬──────┘  └──────┬──────┘
         ▼              ▼                ▼
   ┌────────────┐ ┌────────────┐  ┌─────────────┐
   │ libvlccore │ │ libpng16   │  │ libx264     │
   └─────┬──────┘ └─────┬──────┘  └──────┬──────┘
         └──────────────┼────────────────┘
                        ▼
                  ┌──────────┐
                  │  libc6   │   ← প্রায় সবাই এর উপর নির্ভরশীল
                  └──────────┘
```

### Dependency-র ধরন (Debian পরিভাষা)

| ফিল্ড | মানে |
|-------|------|
| `Depends` | অবশ্যই লাগবে, না থাকলে ইনস্টল হবে না |
| `Pre-Depends` | ইনস্টল শুরুর **আগেই** লাগবে |
| `Recommends` | সাধারণত সাথে থাকা উচিত; `apt` ডিফল্টে এগুলোও ইনস্টল করে |
| `Suggests` | থাকলে ভালো, ডিফল্টে ইনস্টল হয় না |
| `Conflicts` / `Breaks` | এর সাথে একসাথে থাকতে পারবে না |
| `Provides` | "আমি অমুক ক্ষমতা দিই" — যেমন অনেকগুলো mail server সবাই `mail-transport-agent` provide করে |

RPM-এর দিকেও একই ধারণা আছে: `Requires`, `Recommends`, `Suggests`, `Conflicts`, `Provides`, `Obsoletes`।

### "Dependency Hell" — পুরনো দিনের দুঃস্বপ্ন

package manager আসার আগে, একটা প্রোগ্রাম ইনস্টল করতে গেলে বলত "libfoo লাগবে"। libfoo ইনস্টল করতে গেলে বলত "libbar 2.0 লাগবে"। কিন্তু অন্য একটা প্রোগ্রাম চায় libbar 1.0! এই প্যাঁচকে বলে **dependency hell**। আধুনিক high-level package manager একটা **dependency solver** (একধরনের গাণিতিক সমাধানকারী) চালিয়ে এই জট নিজেই খুলে দেয়। `dnf` ও openSUSE-র `zypper` এর জন্য `libsolv` নামের SAT-solver লাইব্রেরি ব্যবহার করে।

> ⚠️ **সতর্কতা:** ইন্টারনেট থেকে র‍্যান্ডম `.deb`/`.rpm` নামিয়ে ইনস্টল করলে dependency গড়মিল হতে পারে, আর সেটা অবিশ্বস্ত উৎস হলে ম্যালওয়্যারও হতে পারে। সম্ভব হলে সবসময় অফিসিয়াল repository ব্যবহার করো।

---

## ৩০.৪ Repository — সফটওয়্যারের গুদাম

**Repository** (সংক্ষেপে repo) হলো একটা সার্ভার (HTTP/HTTPS ওয়েবসাইটের মতো), যেখানে হাজার হাজার প্যাকেজ আর একটা **index** (সূচি) রাখা থাকে। index-এ লেখা থাকে কোন প্যাকেজের কোন version, কী dependency, আর checksum কত।

পৃথিবী জুড়ে এই repo-র অনেক কপি থাকে, যাদের বলে **mirror**। বাংলাদেশ থেকে কাছের mirror ব্যবহার করলে ডাউনলোড দ্রুত হয়।

```
   ┌─────────────────────────────────────────────────────────┐
   │          Ubuntu / Debian / Fedora Repository             │
   │  ┌──────────────┐   ┌───────────────────────────────┐    │
   │  │  Index ফাইল   │   │   pool/ — হাজার হাজার প্যাকেজ  │    │
   │  │  (InRelease,  │   │   htop_3.3.0..deb             │    │
   │  │   Packages)   │   │   vlc_3.0.20..deb  ...         │    │
   │  └──────┬───────┘   └──────────────┬────────────────┘    │
   └─────────┼──────────────────────────┼─────────────────────┘
             │ ① apt update             │ ③ apt install
             │   (শুধু index নামায়)     │   (আসল .deb নামায়)
             ▼                          ▼
   ┌─────────────────────────────────────────────────────────┐
   │                  তোমার কম্পিউটার                          │
   │  /var/lib/apt/lists/  ← index-এর লোকাল কপি               │
   │  ② apt dependency হিসাব করে                              │
   │  /var/cache/apt/archives/ ← নামানো .deb                  │
   │  ④ dpkg ফাইল বসায় → /var/lib/dpkg/status আপডেট           │
   └─────────────────────────────────────────────────────────┘
```

> 📝 **মনে রাখো:** `apt update` কোনো সফটওয়্যার আপডেট করে **না** — এটা শুধু "দোকানের নতুন মূল্যতালিকা" নামায়। আসল আপডেট হয় `apt upgrade` দিয়ে। নতুনরা প্রায়ই এই দুইটা গুলিয়ে ফেলে।

### Ubuntu-র repo component

Ubuntu-তে প্রতিটা repo চারটা ভাগে বিভক্ত:

| component | কী থাকে | কে সাপোর্ট দেয় |
|-----------|---------|-----------------|
| `main` | free/open source, Canonical অফিসিয়ালি সাপোর্ট করে | Canonical |
| `universe` | free/open source, community maintain করে | Community |
| `restricted` | proprietary ড্রাইভার (যেমন কিছু NVIDIA ড্রাইভার) | Canonical |
| `multiverse` | কপিরাইট/লাইসেন্স সমস্যাযুক্ত সফটওয়্যার | কেউ না |

Debian 12-এ ভাগগুলো: `main`, `contrib`, `non-free`, আর Debian 12 থেকে নতুন `non-free-firmware` (Wi-Fi/GPU ফার্মওয়্যারের জন্য)।

আর **suite** (বা pocket) গুলো Ubuntu 24.04 (codename `noble`) এ এমন:

| suite | মানে |
|-------|------|
| `noble` | রিলিজের দিনের মূল প্যাকেজ |
| `noble-updates` | রিলিজের পরের বাগ ফিক্স |
| `noble-security` | নিরাপত্তা আপডেট |
| `noble-backports` | নতুন version, যা পুরনো রিলিজে আনা হয়েছে |

---

## ৩০.৫ Low-level বনাম High-level টুল

এটা এই অধ্যায়ের সবচেয়ে গুরুত্বপূর্ণ architecture ধারণা। প্রতিটা distro-তে প্যাকেজ টুল দুই স্তরে কাজ করে:

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                     তুমি (ইউজার)                                  │
 └──────────────┬──────────────────────────────┬────────────────────┘
                │                              │
   ┌────────────▼────────────┐   ┌─────────────▼──────────────┐
   │  GUI: GNOME Software,   │   │  CLI High-level টুল         │
   │  KDE Discover           │──▶│  apt / apt-get   (Debian)   │
   └─────────────────────────┘   │  dnf / yum       (RHEL)     │
                                 │  pacman -S       (Arch)     │
   HIGH-LEVEL দায়িত্ব:          └─────────────┬──────────────┘
   • repo থেকে খোঁজা ও ডাউনলোড               │
   • dependency হিসাব (solver)               │
   • GPG সিগনেচার যাচাই                       │
                                 ┌─────────────▼──────────────┐
   LOW-LEVEL দায়িত্ব:           │  Low-level টুল              │
   • একটা লোকাল প্যাকেজ খোলা     │  dpkg            (Debian)   │
   • ফাইল বসানো/মোছা             │  rpm             (RHEL)     │
   • script চালানো               │  (pacman নিজেই দুটো কাজ করে) │
   • ডাটাবেস আপডেট               └─────────────┬──────────────┘
                                               │
                                 ┌─────────────▼──────────────┐
                                 │  প্যাকেজ ডাটাবেস             │
                                 │  /var/lib/dpkg/            │
                                 │  /var/lib/rpm/ (বা          │
                                 │   /usr/lib/sysimage/rpm)   │
                                 │  /var/lib/pacman/local/    │
                                 └─────────────┬──────────────┘
                                               ▼
                                 ┌────────────────────────────┐
                                 │ ফাইল সিস্টেম: /usr /etc ... │
                                 └────────────────────────────┘
```

| বৈশিষ্ট্য | Low-level (`dpkg`, `rpm`) | High-level (`apt`, `dnf`) |
|-----------|---------------------------|---------------------------|
| ইন্টারনেট থেকে নামাতে পারে? | না | হ্যাঁ |
| dependency নিজে সমাধান করে? | না, শুধু অভিযোগ করে | হ্যাঁ |
| কোন ফাইলে কাজ করে? | লোকাল `.deb`/`.rpm` ফাইল | repo-র প্যাকেজ নাম (লোকাল ফাইলও পারে) |
| কখন দরকার? | তদন্ত (কোন ফাইল কোন প্যাকেজের), ভাঙা অবস্থা মেরামত | দৈনন্দিন সব কাজ |

> 💡 **উপমা:** `dpkg` হলো রাজমিস্ত্রি — ইট দিলে সে বসিয়ে দেবে, কিন্তু ইট কোথা থেকে আসবে, কত সিমেন্ট লাগবে সেটা জানে না। `apt` হলো ঠিকাদার — সে হিসাব করে, মালামাল কিনে আনে, তারপর রাজমিস্ত্রিকে দিয়ে কাজ করায়।

`apt install vlc` দিলে ভেতরে আসলে `apt` সব `.deb` নামিয়ে শেষে `dpkg`-কে ডেকে কাজ করায়। Fedora-তে `dnf` একইভাবে `rpm` লাইব্রেরি (librpm) ব্যবহার করে।

---

## ৩০.৬ Debian পরিবার: `apt`

Debian, Ubuntu, Linux Mint, Pop!_OS, Kali — সবখানে `apt` চলে।

### `apt`, `apt-get`, `apt-cache` — পার্থক্য কী?

ইতিহাসে প্রথমে এসেছিল `apt-get` (ইনস্টল/আপডেট) আর `apt-cache` (খোঁজা/তথ্য)। ২০১৪ সালে (APT 1.0) এই দুটোর জনপ্রিয় কাজ এক করে আসে `apt` — রঙিন, প্রোগ্রেস বার সহ, মানুষের জন্য সুবিধাজনক।

| টুল | কার জন্য | মন্তব্য |
|-----|----------|---------|
| `apt` | মানুষ (interactive) | output ভবিষ্যতে বদলাতে পারে; স্ক্রিপ্টে ব্যবহার করলে সতর্কবার্তা দেয় |
| `apt-get` | স্ক্রিপ্ট, Dockerfile | output স্থির, পুরনো সব অপশন আছে |
| `apt-cache` | স্ক্রিপ্টে খোঁজা/তথ্য | `apt-cache policy`, `apt-cache depends`, `apt-cache rdepends` |

```bash
$ apt list --installed | head -3

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Listing...
accountsservice/noble,now 23.13.9-2ubuntu6 amd64 [installed,automatic]
```

> 💡 **টিপস:** শেল স্ক্রিপ্ট আর Dockerfile-এ `apt-get` লেখো, টার্মিনালে নিজে টাইপ করার সময় `apt`।

### `apt update` — index রিফ্রেশ

```bash
$ sudo apt update
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
Get:2 http://archive.ubuntu.com/ubuntu noble-updates InRelease [126 kB]
Get:3 http://security.ubuntu.com/ubuntu noble-security InRelease [126 kB]
Get:4 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 Packages [612 kB]
Fetched 864 kB in 2s (432 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
12 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

- `Hit` = আগের index এখনো চলতি, নতুন কিছু নেই
- `Get` = নতুন index নামানো হচ্ছে
- `Ign` = উপেক্ষা করা হয়েছে (সাধারণত অনুবাদ ফাইল না পেলে)
- `Err` = সমস্যা (নেটওয়ার্ক, ভুল URL, বা GPG key সমস্যা)

এই index জমা থাকে `/var/lib/apt/lists/` এ।

### `apt upgrade` বনাম `apt full-upgrade`

```bash
$ apt list --upgradable
Listing... Done
curl/noble-updates,noble-security 8.5.0-2ubuntu10.6 amd64 [upgradable from: 8.5.0-2ubuntu10.5]
libcurl4t64/noble-updates,noble-security 8.5.0-2ubuntu10.6 amd64 [upgradable from: 8.5.0-2ubuntu10.5]

$ sudo apt upgrade
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following packages will be upgraded:
  curl libcurl4t64
2 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Need to get 568 kB of archives.
After this operation, 0 B of additional disk space will be used.
Do you want to continue? [Y/n] y
```

| কমান্ড | নতুন প্যাকেজ ইনস্টল করতে পারে? | পুরনো প্যাকেজ মুছতে পারে? | কখন ব্যবহার |
|--------|-------------------------------|---------------------------|-------------|
| `apt-get upgrade` | না | না | সবচেয়ে রক্ষণশীল |
| `apt upgrade` | হ্যাঁ (নতুন dependency লাগলে) | না | দৈনন্দিন আপডেট |
| `apt full-upgrade` (= `apt-get dist-upgrade`) | হ্যাঁ | হ্যাঁ, দরকার হলে | বড় পরিবর্তন, নতুন kernel series, রিলিজ আপগ্রেডের অংশ |

> ⚠️ **সতর্কতা:** `full-upgrade` কোনো প্যাকেজ মুছে ফেলার প্রস্তাব দিলে `Y` চাপার আগে তালিকাটা পড়ো। "The following packages will be REMOVED" এর নিচে যদি গুরুত্বপূর্ণ কিছু (যেমন `ubuntu-desktop`) দেখো, থামো।

> 📝 **মনে রাখো:** "X packages have been kept back" লেখা আসলে মানে সেই প্যাকেজগুলোর আপগ্রেডে অন্য কিছু মুছতে বা বদলাতে হবে, অথবা Ubuntu-র **phased updates** (আপডেট ধাপে ধাপে কিছু ইউজারকে আগে দেওয়া) চলছে। phased update হলে অপেক্ষা করাই ভালো।

Ubuntu-র এক রিলিজ থেকে পরের রিলিজে (যেমন 22.04 → 24.04) যাওয়ার অফিসিয়াল টুল হলো `do-release-upgrade`, শুধু `full-upgrade` নয়।

একসাথে:

```bash
$ sudo apt update && sudo apt upgrade -y
```

`&&` মানে প্রথমটা সফল হলেই দ্বিতীয়টা চলবে ([অধ্যায় ১৯](19-redirection-pipe.md) ও [অধ্যায় ৪১](41-shell-scripting-1.md) দেখো)। `-y` মানে সব প্রশ্নে স্বয়ংক্রিয়ভাবে "yes"।

### `apt install` — ইনস্টল করা

```bash
$ sudo apt install htop
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  htop
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 171 kB of archives.
After this operation, 434 kB of additional disk space will be used.
Get:1 http://archive.ubuntu.com/ubuntu noble/main amd64 htop amd64 3.3.0-4build1 [171 kB]
Fetched 171 kB in 1s (210 kB/s)
Selecting previously unselected package htop.
(Reading database ... 198432 files and directories currently installed.)
Preparing to unpack .../htop_3.3.0-4build1_amd64.deb ...
Unpacking htop (3.3.0-4build1) ...
Setting up htop (3.3.0-4build1) ...
Processing triggers for man-db (2.12.0-4build2) ...
```

শেষের লাইনগুলো খেয়াল করো: `Unpacking` আর `Setting up` হলো `dpkg`-এর কাজ। `Processing triggers` মানে অন্য কোনো প্যাকেজ (এখানে `man-db`) বলেছিল "নতুন man page এলে আমাকে জানিও" — তাই man ডাটাবেস রিফ্রেশ হলো।

গুরুত্বপূর্ণ ব্যবহার:

```bash
$ sudo apt install nginx curl git           # একসাথে একাধিক
$ sudo apt install nginx=1.24.0-2ubuntu7    # নির্দিষ্ট version
$ sudo apt install -t noble-backports cockpit   # নির্দিষ্ট suite থেকে
$ sudo apt install ./google-chrome-stable_current_amd64.deb   # লোকাল .deb, dependency সহ!
$ sudo apt install --no-install-recommends ffmpeg  # Recommends বাদ দিয়ে, হালকা ইনস্টল
$ sudo apt reinstall htop                   # আবার ইনস্টল (নষ্ট ফাইল ঠিক করতে)
```

> 💡 **টিপস:** লোকাল `.deb` ইনস্টল করার সময় `./` দিতে ভুলো না। `apt install file.deb` লিখলে apt ভাববে এটা repo-তে থাকা প্যাকেজের নাম। `./file.deb` দিলে বুঝবে এটা ফাইল, আর `dpkg -i` এর মতো আটকে না গিয়ে dependency-ও নামিয়ে আনবে।

### `apt remove`, `apt purge`, `apt autoremove`

```
                    ┌───────────────────────────────────┐
                    │     nginx প্যাকেজের অংশ             │
                    ├───────────────┬───────────────────┤
                    │ প্রোগ্রাম ফাইল  │ কনফিগ ফাইল         │
                    │ /usr/sbin/... │ /etc/nginx/...    │
                    └───────────────┴───────────────────┘
  apt remove  →     ✗ মুছবে          ✓ রেখে দেবে
  apt purge   →     ✗ মুছবে          ✗ মুছবে
```

```bash
$ sudo apt remove nginx      # প্রোগ্রাম যায়, /etc কনফিগ থাকে (পরে আবার ইনস্টলে কাজে লাগে)
$ sudo apt purge nginx       # কনফিগ সহ সব মুছে দেয়
$ sudo apt autoremove        # আর কারো দরকার নেই এমন "অনাথ" dependency মুছে দেয়
$ sudo apt autoremove --purge   # অনাথদের কনফিগ সহ
```

> ⚠️ **সতর্কতা:** `purge` তোমার **home ডিরেক্টরির** ফাইল (যেমন `~/.config/...`) মোছে না — শুধু প্যাকেজের নিজস্ব সিস্টেম কনফিগ (`/etc/...`) মোছে। আর ডাটাবেস সার্ভারের ক্ষেত্রে (যেমন `mysql-server`) purge প্রশ্ন করতে পারে ডেটা ডিরেক্টরিও মুছবে কিনা — সাবধানে উত্তর দাও।

**autoremove কীভাবে জানে কোনটা "অনাথ"?** apt প্রতিটা প্যাকেজকে দুই ভাগে চিহ্নিত রাখে:
- **manual** — তুমি নিজে নাম দিয়ে ইনস্টল করেছ
- **automatic** — অন্য প্যাকেজের dependency হিসেবে এসেছে

কোনো automatic প্যাকেজের উপর যখন আর কোনো manual প্যাকেজ নির্ভর করে না, তখন সে autoremove-এর শিকার। এই তথ্য থাকে `/var/lib/apt/extended_states` ফাইলে।

```bash
$ apt-mark showmanual | head -3
$ sudo apt-mark manual libfoo1    # "এটা আমার দরকার, মুছবে না"
$ sudo apt-mark auto libfoo1      # "এটা শুধু dependency"
```

> ⚠️ **সতর্কতা:** `autoremove` চালানোর আগে তালিকাটা দেখো। কখনো কখনো একটা meta-package (যেমন `ubuntu-desktop`) মুছে ফেললে পুরো ডেস্কটপের প্যাকেজগুলো "অনাথ" হয়ে যায়, আর autoremove সেগুলো মুছতে চাইবে!

### `apt search`, `apt show`, `apt list`

```bash
$ apt search "image viewer"
Sorting... Done
Full Text Search... Done
eog/noble 45.3-1ubuntu1 amd64
  Eye of GNOME graphics viewer program

feh/noble 3.10.1-1build3 amd64
  imlib2 based image viewer
...

$ apt search --names-only '^python3-requests'   # শুধু নামে খোঁজো (regex)
```

```bash
$ apt show htop
Package: htop
Version: 3.3.0-4build1
Priority: optional
Section: utils
Origin: Ubuntu
Maintainer: Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com>
Installed-Size: 434 kB
Depends: libc6 (>= 2.38), libncursesw6 (>= 6), libnl-3-200 (>= 3.2.7), libnl-genl-3-200 (>= 3.2.7), libtinfo6 (>= 6)
Suggests: lm-sensors, lsof, strace
Homepage: https://htop.dev/
Download-Size: 171 kB
APT-Sources: http://archive.ubuntu.com/ubuntu noble/main amd64 Packages
Description: interactive processes viewer
 Htop is an ncursed-based process viewer similar to top, but it
 allows one to scroll the list vertically and horizontally ...
```

```bash
$ apt list --installed             # ইনস্টল করা সব
$ apt list --upgradable            # যাদের আপডেট আছে
$ apt list --all-versions nginx    # repo-তে থাকা সব version
$ apt list 'linux-image-*'         # pattern দিয়ে
```

### `apt policy` — কোন version কোথা থেকে আসবে

একই প্যাকেজ একাধিক repo-তে থাকতে পারে (যেমন Ubuntu-র নিজের repo আর একটা PPA)। apt কোনটা বেছে নেবে সেটা ঠিক করে **priority** (pin priority) দিয়ে।

```bash
$ apt policy nginx
nginx:
  Installed: (none)
  Candidate: 1.24.0-2ubuntu7.1
  Version table:
     1.24.0-2ubuntu7.1 500
        500 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu noble-security/main amd64 Packages
     1.24.0-2ubuntu7 500
        500 http://archive.ubuntu.com/ubuntu noble/main amd64 Packages
```

- `Installed` — এখন যা ইনস্টল আছে
- `Candidate` — `apt install` দিলে যা ইনস্টল হবে
- সংখ্যা `500` — সাধারণ repo-র ডিফল্ট priority; `100` = ইনস্টল করা version বা backports; `990` = target release

`apt policy` (নাম ছাড়া) দিলে সব repo আর তাদের priority দেখায়। কোনো repo-র priority বদলাতে চাইলে `/etc/apt/preferences.d/` এ ফাইল লেখো (একে বলে **APT pinning**):

```
# /etc/apt/preferences.d/firefox-mozilla
Package: firefox*
Pin: origin packages.mozilla.org
Pin-Priority: 1000
```

`man apt_preferences` এ পুরো নিয়ম আছে।

### অন্যান্য দরকারি apt কমান্ড

| কমান্ড | কাজ |
|--------|-----|
| `apt-cache depends pkg` | pkg কোন কোন প্যাকেজের উপর নির্ভর করে |
| `apt-cache rdepends pkg` | কে কে pkg-এর উপর নির্ভর করে (reverse dependency) |
| `apt download pkg` | শুধু `.deb` নামাও, ইনস্টল নয় |
| `apt source pkg` | সোর্স কোড নামাও ([অধ্যায় ৩১](31-compile-from-source.md)) |
| `sudo apt clean` | `/var/cache/apt/archives/` এর সব নামানো `.deb` মুছে জায়গা খালি করো |
| `sudo apt autoclean` | শুধু যেগুলো আর repo-তে নেই সেগুলো মোছো |
| `sudo apt --fix-broken install` (`-f`) | ভাঙা dependency মেরামত |
| `apt changelog pkg` | প্যাকেজের পরিবর্তনের ইতিহাস |
| `sudo apt edit-sources` | সোর্স ফাইল এডিটরে খোলো |

apt-এর কাজের লগ থাকে `/var/log/apt/history.log` (কী ইনস্টল/মুছেছে) আর `/var/log/apt/term.log` (পুরো টার্মিনাল output) এ — লগ সম্পর্কে বিস্তারিত [অধ্যায় ২৮](28-logs.md)-এ।

```bash
$ grep -A3 "Start-Date" /var/log/apt/history.log | tail -4
Start-Date: 2026-11-12  10:14:02
Commandline: apt install htop
Requested-By: rahim (1000)
Install: htop:amd64 (3.3.0-4build1)
```

### Ubuntu-র স্বয়ংক্রিয় নিরাপত্তা আপডেট

Ubuntu-তে ডিফল্টে `unattended-upgrades` প্যাকেজ চালু থাকে, যা প্রতিদিন নিরাপত্তা আপডেট নিজেই ইনস্টল করে। কনফিগ `/etc/apt/apt.conf.d/50unattended-upgrades` আর `20auto-upgrades`। Debian-এ `sudo apt install unattended-upgrades` দিয়ে, তারপর `sudo dpkg-reconfigure -plow unattended-upgrades` দিয়ে চালু করতে হয়। নিরাপত্তা নিয়ে আরও [অধ্যায় ৪৫](45-security-hardening.md)-এ।

> ⚠️ **সতর্কতা: lock error।** `E: Could not get lock /var/lib/dpkg/lock-frontend` দেখলে মানে অন্য কোনো apt/dpkg প্রক্রিয়া (প্রায়ই `unattended-upgrades` বা GUI Software Updater) চলছে। একসাথে দুটো প্যাকেজ টুল চললে ডাটাবেস নষ্ট হতে পারে, তাই এই lock। কয়েক মিনিট অপেক্ষা করো। lock ফাইল জোর করে `rm` করবে **না** — আগে `ps aux | grep -E 'apt|dpkg'` দিয়ে দেখো কে চলছে ([অধ্যায় ২৪](24-process-management.md))।

---

## ৩০.৭ `dpkg` — Debian-এর low-level টুল

`dpkg` সরাসরি `.deb` ফাইল নিয়ে কাজ করে, আর `/var/lib/dpkg/` এর ডাটাবেস রক্ষা করে। এখানে থাকে:
- `/var/lib/dpkg/status` — প্রতিটা প্যাকেজের অবস্থা (একটা বিশাল টেক্সট ফাইল)
- `/var/lib/dpkg/info/*.list` — প্রতিটা প্যাকেজের ফাইল তালিকা
- `/var/lib/dpkg/info/*.postinst` ইত্যাদি — maintainer script

### গুরুত্বপূর্ণ অপশন

| অপশন | লম্বা রূপ | কাজ |
|-------|-----------|-----|
| `-i file.deb` | `--install` | লোকাল .deb ইনস্টল (dependency নিজে আনে না) |
| `-r pkg` | `--remove` | প্যাকেজ মোছো, কনফিগ রেখে |
| `-P pkg` | `--purge` | কনফিগ সহ মোছো |
| `-l [pattern]` | `--list` | ইনস্টল করা প্যাকেজের তালিকা ও অবস্থা |
| `-L pkg` | `--listfiles` | প্যাকেজটা কোন কোন ফাইল বসিয়েছে |
| `-S path` | `--search` | এই ফাইলটা কোন প্যাকেজের? |
| `-s pkg` | `--status` | প্যাকেজের বিস্তারিত অবস্থা |
| `-c file.deb` | `--contents` | ইনস্টল না করে .deb-এর ভেতরের ফাইল দেখো |
| `-I file.deb` | `--info` | .deb-এর metadata |
| `--configure -a` | | আধা-ইনস্টল হওয়া সব প্যাকেজের কনফিগারেশন শেষ করো |
| `--get-selections` | | সব প্যাকেজ ও তাদের নির্বাচিত অবস্থা |

### `dpkg -i` ও dependency সমস্যা

```bash
$ sudo dpkg -i some-app_1.0_amd64.deb
Selecting previously unselected package some-app.
(Reading database ... 198500 files and directories currently installed.)
Preparing to unpack some-app_1.0_amd64.deb ...
Unpacking some-app (1.0) ...
dpkg: dependency problems prevent configuration of some-app:
 some-app depends on libfoo2; however:
  Package libfoo2 is not installed.

dpkg: error processing package some-app (--install):
 dependency problems - leaving unconfigured
Errors were encountered while processing:
 some-app
```

দেখো — `dpkg` শুধু অভিযোগ করল, নিজে `libfoo2` আনল না। সমাধান:

```bash
$ sudo apt --fix-broken install
```

apt তখন হারানো dependency নামিয়ে আধা-ইনস্টল প্যাকেজের কাজ শেষ করবে (বা সেটা সম্ভব না হলে প্যাকেজটা মুছে দেওয়ার প্রস্তাব দেবে)। এজন্যই আধুনিক পরামর্শ: সরাসরি `sudo apt install ./some-app_1.0_amd64.deb` ব্যবহার করো।

### `dpkg -l` — তালিকা আর অবস্থার কোড

```bash
$ dpkg -l 'nginx*'
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name           Version            Architecture Description
+++-==============-==================-============-=================================
ii  nginx          1.24.0-2ubuntu7.1  amd64        small, powerful, scalable web/proxy server
ii  nginx-common   1.24.0-2ubuntu7.1  all          small, powerful, scalable web/proxy server - common files
rc  nginx-core     1.18.0-6ubuntu14   amd64        nginx web/proxy server (standard version)
```

প্রথম দুই অক্ষর পড়তে শেখো:

| কোড | মানে |
|-----|------|
| `ii` | ঠিকঠাক ইনস্টল আছে (desired=install, status=installed) |
| `rc` | remove করা হয়েছে, কিন্তু কনফিগ ফাইল রয়ে গেছে (purge হয়নি) |
| `hi` | hold করা আছে, ইনস্টল অবস্থায় |
| `iU` | unpacked, কনফিগার হয়নি — অসম্পূর্ণ |
| `iF` | half-configured — কনফিগারেশনে ব্যর্থ |
| `un` | ইনস্টল নেই |

> 💡 **টিপস:** সব `rc` অবস্থার প্যাকেজের অবশিষ্ট কনফিগ একবারে পরিষ্কার করতে:
> ```bash
> $ dpkg -l | awk '/^rc/ {print $2}' | xargs -r sudo dpkg -P
> ```
> (`awk` আর `xargs` শিখবে [অধ্যায় ২২](22-awk.md) ও [অধ্যায় ২৩](23-text-tools.md)-এ।)

### `dpkg -L` ও `dpkg -S` — তদন্তকারী কমান্ড

```bash
$ dpkg -L htop
/.
/usr
/usr/bin
/usr/bin/htop
/usr/share
/usr/share/applications
/usr/share/applications/htop.desktop
/usr/share/doc/htop/changelog.Debian.gz
/usr/share/man/man1/htop.1.gz
...

$ dpkg -S /usr/bin/ls
coreutils: /usr/bin/ls

$ dpkg -S bin/ssh
openssh-client: /usr/bin/ssh
openssh-client: /usr/bin/ssh-add
...
```

> 📝 **মনে রাখো:** `dpkg -S` শুধু **ইনস্টল করা** প্যাকেজে খোঁজে। যে ফাইল এখনো তোমার সিস্টেমে নেই, সেটা কোন প্যাকেজে আছে জানতে `apt-file` ব্যবহার করো:
> ```bash
> $ sudo apt install apt-file && sudo apt-file update
> $ apt-file search bin/ifconfig
> net-tools: /usr/sbin/ifconfig
> ```

Ubuntu-তে কোনো কমান্ড না থাকলে নিজেই বলে দেয় (এটা `command-not-found` প্যাকেজের কাজ):

```bash
$ tree
Command 'tree' not found, but can be installed with:
sudo apt install tree
```

### `dpkg -r` ও মেরামত

```bash
$ sudo dpkg -r htop          # মোছো (কনফিগ থাকবে)
$ sudo dpkg -P htop          # পুরো মোছো
$ sudo dpkg --configure -a   # আপডেট মাঝপথে বন্ধ হলে (বিদ্যুৎ গেলে!) এটা আগে চালাও
$ sudo dpkg-reconfigure tzdata   # কোনো প্যাকেজের সেটআপ প্রশ্নগুলো আবার করো
```

> 💡 **টিপস:** আপডেটের মাঝে কম্পিউটার বন্ধ হয়ে গেলে, সবচেয়ে কাজের রেসিপি হলো:
> ```bash
> $ sudo dpkg --configure -a
> $ sudo apt --fix-broken install
> $ sudo apt update && sudo apt upgrade
> ```

### ইনস্টল তালিকা ব্যাকআপ ও নতুন মেশিনে পুনরুদ্ধার

```bash
$ apt-mark showmanual > my-packages.txt      # পুরনো মেশিনে
$ xargs -a my-packages.txt sudo apt install -y   # নতুন মেশিনে (একই distro version)
```

---

## ৩০.৮ Debian পরিবারের repository কনফিগারেশন

### পুরনো এক-লাইন ফরম্যাট: `sources.list`

Debian 12 এখনো ডিফল্টে `/etc/apt/sources.list` ব্যবহার করে:

```
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware
# deb-src http://deb.debian.org/debian bookworm main
```

প্রতিটা লাইনের গঠন:

```
deb  [options]  http://deb.debian.org/debian  bookworm  main contrib
│    │          │                             │         │
│    │          │                             │         └─ component
│    │          │                             └─────────── suite / codename
│    │          └───────────────────────────────────────── mirror URL
│    └──────────────────────────────────────────────────── যেমন [signed-by=/path/key.gpg]
└───────────────────────────────────────────────────────── deb = binary, deb-src = source
```

### নতুন deb822 ফরম্যাট (Ubuntu 24.04+)

Ubuntu 24.04 থেকে `/etc/apt/sources.list` প্রায় খালি (শুধু একটা কমেন্ট), আর আসল কনফিগ থাকে `/etc/apt/sources.list.d/ubuntu.sources` এ, **deb822** ফরম্যাটে (key: value ধরনের, বেশি পাঠযোগ্য):

```
$ cat /etc/apt/sources.list.d/ubuntu.sources
Types: deb
URIs: http://archive.ubuntu.com/ubuntu/
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

Types: deb
URIs: http://security.ubuntu.com/ubuntu/
Suites: noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

একটা ব্লক বন্ধ করতে চাইলে তাতে `Enabled: no` লাইন যোগ করো। Debian 13 (trixie)-ও deb822-এর দিকে যাচ্ছে।

### `/etc/apt/sources.list.d/`

থার্ড-পার্টি repo (Docker, Google Chrome, VS Code ইত্যাদি) মূল ফাইল না ঘেঁটে এই ডিরেক্টরিতে আলাদা ফাইল রাখে — `.list` (এক-লাইন ফরম্যাট) বা `.sources` (deb822)। এতে একটা repo মুছতে শুধু একটা ফাইল মুছলেই হয়।

> 💡 **টিপস:** কাছের mirror বেছে নিতে Ubuntu-তে GUI-র "Software & Updates" → "Download from" ব্যবহার করতে পারো, অথবা ফাইলে `archive.ubuntu.com` এর জায়গায় দেশের mirror (যেমন `bd.archive.ubuntu.com`) বসাতে পারো — বসানোর আগে mirror-টা সক্রিয় কিনা ব্রাউজারে দেখে নিও। ফাইল বদলের পরে সবসময় `sudo apt update`।

### PPA — Personal Package Archive (Ubuntu-only)

**PPA** হলো Launchpad (Canonical-এর প্ল্যাটফর্ম) এ কোনো ব্যক্তি বা দলের বানানো ছোট repo। অফিসিয়াল repo-তে যে নতুন version এখনো আসেনি, সেটা অনেক সময় PPA-তে পাওয়া যায়।

```bash
$ sudo apt install software-properties-common    # add-apt-repository এর জন্য (সাধারণত আগেই থাকে)
$ sudo add-apt-repository ppa:git-core/ppa
 The most current stable version of Git for Ubuntu.
 More info: https://launchpad.net/~git-core/+archive/ubuntu/ppa
Adding repository.
Press [ENTER] to continue or Ctrl-c to cancel.
...
$ sudo apt update
$ sudo apt install git
```

`add-apt-repository` নিজেই:
1. `/etc/apt/sources.list.d/` এ একটা ফাইল লেখে (24.04-এ `.sources` ফরম্যাটে)
2. PPA-র GPG key নামিয়ে বসায়
3. (সাম্প্রতিক version-এ) `apt update` চালায়

PPA সরাতে:

```bash
$ sudo add-apt-repository --remove ppa:git-core/ppa
$ sudo apt install ppa-purge && sudo ppa-purge ppa:git-core/ppa   # PPA-র প্যাকেজগুলোও অফিসিয়াল version-এ ফেরত নেয়
```

> ⚠️ **সতর্কতা:** PPA মানে কোনো ব্যক্তিকে তোমার কম্পিউটারে **root ক্ষমতা** দেওয়া — কারণ তার প্যাকেজের `postinst` স্ক্রিপ্ট root হিসেবে চলে। শুধু পরিচিত, বিশ্বস্ত PPA ব্যবহার করো। অনেক PPA যোগ করলে রিলিজ আপগ্রেডের সময় গোলমাল হয়। আর PPA শুধু Ubuntu-র জন্য — Debian-এ যোগ কোরো না।

### `apt-mark hold` — আপডেট আটকে রাখা

ধরো তোমার একটা অ্যাপ নির্দিষ্ট version-এর `docker-ce` এ ঠিকমতো চলে, আর নতুন version-এ সমস্যা আছে। তখন সেটাকে "hold" করো:

```bash
$ sudo apt-mark hold docker-ce
docker-ce set on hold.

$ apt-mark showhold
docker-ce

$ sudo apt upgrade
...
The following packages have been kept back:
  docker-ce
0 upgraded, 0 newly installed, 0 to remove and 1 not upgraded.

$ sudo apt-mark unhold docker-ce
Canceled hold on docker-ce.
```

`dpkg -l` এ hold করা প্যাকেজ `hi` দেখায়।

> ⚠️ **সতর্কতা:** hold করা প্যাকেজ নিরাপত্তা আপডেটও পাবে না। কাজ শেষ হলে unhold করতে ভুলো না। kernel hold করলে নিরাপত্তা ঝুঁকি বাড়ে।

---

## ৩০.৯ GPG key দিয়ে repository নিরাপত্তা

ইন্টারনেট থেকে প্যাকেজ নামাচ্ছ, আর সেটা root হিসেবে চলছে। কেউ যদি mirror হ্যাক করে, বা মাঝপথে (man-in-the-middle) প্যাকেজ বদলে দেয়? এর সমাধান **public-key cryptography** (পাবলিক-কী ক্রিপ্টোগ্রাফি) — GPG (GNU Privacy Guard) দিয়ে।

> 💡 **উপমা:** repo-র মালিকের কাছে আছে একটা গোপন সিলমোহর (**private key**)। সে প্রতিটা তালিকায় সিল মারে। তোমার কাছে আছে সেই সিলের একটা নমুনা ছবি (**public key**)। নমুনার সাথে সিল মিললে বোঝো আসল; না মিললে জাল। নমুনা দেখে কেউ নতুন সিল বানাতে পারে না।

### APT-এর বিশ্বাসের শিকল (chain of trust)

APT প্রতিটা প্যাকেজে আলাদা সিগনেচার দেখে না — এটা একটা শিকল বানায়:

```
  ┌───────────────────────────────────────────┐
  │ InRelease  (GPG দিয়ে সাইন করা)              │ ◀── public key দিয়ে যাচাই
  │   ├─ Packages ফাইলের SHA256 hash           │     (/usr/share/keyrings/
  │   └─ ...                                  │      বা /etc/apt/keyrings/)
  └──────────────────┬────────────────────────┘
                     │ hash মেলে?
  ┌──────────────────▼────────────────────────┐
  │ Packages  (প্রতিটা .deb-এর তালিকা)          │
  │   htop_3.3.0..deb  SHA256: 9f2c...        │
  └──────────────────┬────────────────────────┘
                     │ hash মেলে?
  ┌──────────────────▼────────────────────────┐
  │ htop_3.3.0-4build1_amd64.deb              │  ✓ বিশ্বস্ত
  └───────────────────────────────────────────┘
```

একটা লিংক ভাঙলেই apt থেমে যাবে। এজন্যই HTTP (HTTPS নয়) mirror-ও নিরাপদ — বিষয়বস্তু বদলালে hash মিলবে না। তবে HTTPS গোপনীয়তা (কে কী নামাচ্ছে) দেয়।

key না মিললে এমন error আসে:

```bash
$ sudo apt update
...
W: GPG error: https://example.com/repo stable InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 1234ABCD5678EF90
E: The repository 'https://example.com/repo stable InRelease' is not signed.
```

### `apt-key` কেন বাদ দেওয়া হলো

আগে সবাই লিখত `wget -qO- https://.../key | sudo apt-key add -`। সমস্যা: এতে key যেত একটা **global** keyring-এ (`/etc/apt/trusted.gpg`), মানে সেই key দিয়ে **যেকোনো** repo-র প্যাকেজ সাইন করা যেত — এমনকি কেউ চাইলে Ubuntu-র অফিসিয়াল `openssl` এর জাল version-ও! তাই `apt-key` deprecated; Debian 12 / Ubuntu 22.04 থেকে সতর্কতা দেয়, আর নতুন APT (Ubuntu 24.04-এর APT 2.7 সহ) এ এটা ধাপে ধাপে সরিয়ে ফেলা হচ্ছে।

### আধুনিক সঠিক পদ্ধতি: `signed-by`

এখন প্রতিটা থার্ড-পার্টি key আলাদা ফাইলে রাখা হয়, আর repo-র লাইনে বলে দেওয়া হয় "এই repo শুধু এই key দিয়ে যাচাই করো"।

উদাহরণ — Docker-এর অফিসিয়াল repo (Ubuntu):

```bash
# ১. key রাখার ডিরেক্টরি
$ sudo install -m 0755 -d /etc/apt/keyrings

# ২. key নামাও
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
$ sudo chmod a+r /etc/apt/keyrings/docker.asc

# ৩. repo যোগ করো, signed-by দিয়ে
$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# ৪. আপডেট ও ইনস্টল
$ sudo apt update
$ sudo apt install docker-ce
```

key যদি ASCII-armored (`-----BEGIN PGP PUBLIC KEY BLOCK-----`) হয়, `.asc` নামে রাখলে আধুনিক APT সরাসরি পড়তে পারে। পুরনো রীতিতে বাইনারি `.gpg` বানানো হয়:

```bash
$ curl -fsSL https://example.com/key.asc | sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg
```

deb822 ফরম্যাটে একই জিনিস:

```
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
```

> 📝 **মনে রাখো:** distro-র নিজস্ব key থাকে `/usr/share/keyrings/` এ (প্যাকেজ দিয়ে আসে)। তুমি নিজে যেগুলো যোগ করো সেগুলো `/etc/apt/keyrings/` এ রাখার রীতি।

> ⚠️ **সতর্কতা:** key-এর **fingerprint** সবসময় অফিসিয়াল ওয়েবসাইটের সাথে মিলিয়ে নাও:
> ```bash
> $ gpg --show-keys /etc/apt/keyrings/docker.asc
> pub   rsa4096 2017-02-22 [SCEA]
>       9DC858229FC7DD38854AE2D88D81803C0EBFCD88
> uid                      Docker Release (CE deb) <docker@docker.com>
> ```
> আর কখনো `[trusted=yes]` বা `--allow-unauthenticated` দিয়ে সিগনেচার যাচাই বন্ধ কোরো না।

---

## ৩০.১০ RHEL পরিবার: `dnf` ও `yum`

Fedora, RHEL, CentOS Stream, Rocky Linux, AlmaLinux, Oracle Linux — এখানে high-level টুল **`dnf`** (Dandified YUM)।

- **yum** (Yellowdog Updater, Modified) ছিল পুরনো টুল (RHEL 7 পর্যন্ত)।
- RHEL 8 থেকে `dnf` ডিফল্ট; `yum` কমান্ডটা এখন শুধু `dnf`-এর একটা symlink। তাই RHEL 9-এ `yum install` আর `dnf install` একই কাজ করে।
- Fedora 41 থেকে এসেছে **dnf5** — C++ এ নতুন করে লেখা, অনেক দ্রুত। বেশিরভাগ কমান্ড একই, তবে কিছু plugin কমান্ডের syntax বদলেছে (নিচে উল্লেখ করা আছে)।

### দৈনন্দিন dnf কমান্ড

| কাজ | কমান্ড | Debian-এর সমতুল্য |
|-----|--------|-------------------|
| index রিফ্রেশ | `dnf makecache` (সাধারণত দরকার নেই) | `apt update` |
| আপডেট আছে কিনা দেখো | `dnf check-update` | `apt list --upgradable` |
| সব আপডেট | `sudo dnf upgrade` (`update` একই) | `apt update && apt upgrade` |
| ইনস্টল | `sudo dnf install pkg` | `apt install pkg` |
| মোছো | `sudo dnf remove pkg` | `apt remove pkg` |
| অনাথ dependency মোছো | `sudo dnf autoremove` | `apt autoremove` |
| খোঁজো | `dnf search word` | `apt search word` |
| তথ্য | `dnf info pkg` | `apt show pkg` |
| কোন প্যাকেজে ফাইলটা আছে | `dnf provides /path/or/cmd` | `apt-file search` |
| তালিকা | `dnf list installed` | `apt list --installed` |
| repo তালিকা | `dnf repolist` | `apt policy` |
| ইতিহাস | `dnf history` | `/var/log/apt/history.log` |
| ক্যাশ মোছো | `sudo dnf clean all` | `apt clean` |
| প্যাকেজ গ্রুপ | `sudo dnf group install "Development Tools"` | (meta-package যেমন `build-essential`) |

> 📝 **মনে রাখো:** `dnf` মেটাডেটা নিজেই ক্যাশ থেকে মেয়াদ দেখে রিফ্রেশ করে, তাই `apt update` এর মতো আলাদা ধাপ সাধারণত লাগে না। আরেকটা পার্থক্য: dnf-এ `remove` করলে কনফিগ ফাইলের আলাদা purge ধারণা নেই — বদলানো কনফিগ ফাইল `.rpmsave` নামে রেখে দেয়।

### ইনস্টল উদাহরণ

```bash
$ sudo dnf install htop
Updating and loading repositories:
Repositories loaded.
Package                 Arch    Version          Repository      Size
Installing:
 htop                   x86_64  3.3.0-3.fc41     fedora      471.6 KiB

Transaction Summary:
 Installing:         1 package

Total size of inbound packages is 205 KiB. Need to download 205 KiB.
After this operation, 472 KiB extra will be used (install 472 KiB, remove 0 B).
Is this ok [y/N]: y
[1/3] htop-0:3.3.0-3.fc41.x86_64       100% | 1.2 MiB/s | 205.3 KiB | 00m00s
...
Complete!
```

(output-এর চেহারা dnf4 আর dnf5-এ কিছুটা আলাদা; উপরেরটা Fedora 41-এর dnf5 ধাঁচের।)

লক্ষ করো: dnf ডিফল্টে `[y/N]` — মানে Enter চাপলে **না**। apt-এ `[Y/n]` — Enter মানে হ্যাঁ।

### `dnf search`, `dnf info`, `dnf provides`

```bash
$ dnf search nginx
Matched fields: name, summary
 nginx.x86_64: A high performance web server and reverse proxy server
 ...

$ dnf info nginx
Name         : nginx
Epoch        : 2
Version      : 1.26.2
Release      : 1.fc41
Architecture : x86_64
Repository   : updates
Summary      : A high performance web server and reverse proxy server
License      : BSD-2-Clause
...

$ dnf provides /usr/bin/dig
bind-utils-32:9.18.28-1.fc41.x86_64 : Utilities for querying DNS name servers
Repo         : fedora
Matched From :
Filename     : /usr/bin/dig

$ dnf provides '*/ifconfig'      # wildcard দিয়ে
```

`dnf provides` এর বড় সুবিধা: ফাইলটা তোমার সিস্টেমে না থাকলেও repo-র ভেতরে খুঁজে দেয় (Debian-এ এর জন্য `apt-file` আলাদা লাগে)।

### `dnf history` — টাইম মেশিন

dnf প্রতিটা transaction (লেনদেন — একবারে করা ইনস্টল/আপডেট) নম্বর দিয়ে মনে রাখে, আর চাইলে **undo** করা যায়। এটা dnf-এর একটা দারুণ বৈশিষ্ট্য।

```bash
$ sudo dnf history list
ID | Command line            | Date and time    | Action(s) | Altered
---+-------------------------+------------------+-----------+--------
 15| install htop            | 2026-11-12 10:20 | Install   |       1
 14| upgrade                 | 2026-11-11 21:03 | Upgrade   |      37
...

$ sudo dnf history info 15     # ১৫ নম্বরে কী হয়েছিল
$ sudo dnf history undo 15     # ১৫ নম্বর উল্টে দাও (htop মুছে যাবে)
$ sudo dnf history rollback 13 # ১৩ নম্বরের পরের সব পরিবর্তন উল্টে দাও
```

> ⚠️ **সতর্কতা:** undo/rollback কাজ করে যদি পুরনো version এখনো repo-তে থাকে। অনেক সময় repo থেকে পুরনো version সরিয়ে ফেলা হয়, তখন undo ব্যর্থ হয়।

### অন্যান্য দরকারি dnf কমান্ড

```bash
$ sudo dnf reinstall pkg
$ sudo dnf downgrade pkg
$ sudo dnf install ./local-file.rpm        # লোকাল rpm, dependency সহ
$ dnf repoquery --requires pkg              # dependency তালিকা
$ dnf repoquery --whatrequires pkg          # reverse dependency
$ dnf list --upgrades                       # যাদের আপডেট আছে
$ sudo dnf upgrade --security               # শুধু নিরাপত্তা আপডেট (RHEL-এ বিশেষ উপযোগী)
$ dnf updateinfo list                       # কোন advisory আছে (dnf5-এ `dnf advisory list`)
```

### version আটকে রাখা (dnf-এ hold)

Debian-এর `apt-mark hold` এর সমতুল্য হলো **versionlock** plugin:

```bash
# dnf4 (RHEL 9)
$ sudo dnf install python3-dnf-plugin-versionlock
$ sudo dnf versionlock add docker-ce
$ sudo dnf versionlock list
$ sudo dnf versionlock delete docker-ce

# অথবা /etc/dnf/dnf.conf এ স্থায়ীভাবে বাদ দেওয়া:
# exclude=kernel*
```

dnf5 (Fedora 41+) এ versionlock মূল টুলেই আছে: `sudo dnf versionlock add pkg`।

### dnf কনফিগ ও repo ফাইল

মূল কনফিগ: `/etc/dnf/dnf.conf`

```ini
[main]
gpgcheck=True
installonly_limit=3
clean_requirements_on_remove=True
best=True
skip_if_unavailable=False
# max_parallel_downloads=10
# fastestmirror=True
```

`installonly_limit=3` মানে সর্বোচ্চ ৩টা kernel version রাখবে (kernel আপডেট হলে পুরনোটা মোছে না, কারণ নতুনটা বুট না হলে ফেরত যাওয়া যায় — [অধ্যায় ২৭](27-boot-process.md) দেখো)।

repo-গুলো থাকে `/etc/yum.repos.d/*.repo` এ (নাম পুরনো yum-এর হলেও dnf এটাই পড়ে):

```ini
# /etc/yum.repos.d/docker-ce.repo
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/fedora/$releasever/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/fedora/gpg
```

| ফিল্ড | মানে |
|-------|------|
| `[docker-ce-stable]` | repo ID (অনন্য নাম) |
| `baseurl` / `metalink` / `mirrorlist` | কোথা থেকে আনবে |
| `$releasever`, `$basearch` | ভেরিয়েবল — যেমন `41`, `x86_64` |
| `enabled=1` | চালু |
| `gpgcheck=1` | প্যাকেজের সিগনেচার যাচাই করবে |
| `gpgkey` | public key কোথায় |

repo যোগ করা (`dnf-plugins-core` লাগে, যা সাধারণত আগেই থাকে):

```bash
# dnf4 (RHEL 9, Rocky 9, Alma 9)
$ sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
$ sudo dnf config-manager --set-disabled docker-ce-stable

# dnf5 (Fedora 41+)
$ sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo
$ sudo dnf config-manager setopt docker-ce-stable.enabled=0
```

এককালীন কোনো repo চালু/বন্ধ করে কমান্ড চালাতে:

```bash
$ sudo dnf --enablerepo=updates-testing upgrade pkg
$ sudo dnf --disablerepo='*' --enablerepo=baseos install pkg
```

### RHEL-বিশেষ: subscription, EPEL, AppStream

- **RHEL** এ repo পেতে Red Hat subscription register করতে হয়: `sudo subscription-manager register`। (Rocky/Alma তে লাগে না।)
- RHEL 9-এ মূল দুটো repo: **BaseOS** (মূল OS) আর **AppStream** (অ্যাপ্লিকেশন, ভাষা, ডাটাবেস)।
- **EPEL** (Extra Packages for Enterprise Linux) — Fedora প্রকল্পের বানানো অতিরিক্ত প্যাকেজ। RHEL-এ প্রথমে CodeReady Builder (Rocky/Alma-তে `crb`) চালু করা লাগে:
  ```bash
  $ sudo dnf config-manager --set-enabled crb      # Rocky/Alma 9
  $ sudo dnf install epel-release
  ```
- Fedora-তে **RPM Fusion** হলো জনপ্রিয় থার্ড-পার্টি repo (মিডিয়া কোডেক, NVIDIA ড্রাইভার)।

### dnf-এ GPG

RPM-জগতে **প্রতিটা প্যাকেজ ফাইল আলাদাভাবে সাইন করা** থাকে (repo metadata ছাড়াও)। `gpgcheck=1` থাকলে dnf ইনস্টলের আগে প্রতিটা `.rpm` এর সিগনেচার দেখে। নতুন repo-র key প্রথমবার ব্যবহারের সময় dnf তোমাকে জিজ্ঞেস করবে:

```
Importing GPG key 0x621E9F35:
 Userid     : "Docker Release (CE rpm) <docker@docker.com>"
 Fingerprint: 060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35
 From       : https://download.docker.com/linux/fedora/gpg
Is this ok [y/N]:
```

fingerprint অফিসিয়াল সাইটের সাথে মিলিয়ে তারপর `y` দাও।

---

## ৩০.১১ `rpm` — RHEL-এর low-level টুল

`rpm` হলো Debian-এর `dpkg` এর সমতুল্য। সবচেয়ে বেশি ব্যবহার হয় **query** (`-q`) মোডে।

### query অপশন

| কমান্ড | কাজ | dpkg সমতুল্য |
|--------|-----|--------------|
| `rpm -qa` | সব ইনস্টল করা প্যাকেজ (query all) | `dpkg -l` |
| `rpm -q pkg` | প্যাকেজটা আছে কি না, কোন version | `dpkg -s pkg` |
| `rpm -qi pkg` | বিস্তারিত তথ্য (info) | `dpkg -s pkg` |
| `rpm -ql pkg` | কোন কোন ফাইল (list) | `dpkg -L pkg` |
| `rpm -qf /path` | ফাইলটা কোন প্যাকেজের (file) | `dpkg -S /path` |
| `rpm -qc pkg` | শুধু কনফিগ ফাইল | — |
| `rpm -qd pkg` | শুধু ডকুমেন্টেশন ফাইল | — |
| `rpm -qR pkg` | dependency (requires) | `apt-cache depends` |
| `rpm -q --changelog pkg` | changelog | `apt changelog` |
| `rpm -qpi file.rpm` | ইনস্টল না করা `.rpm` ফাইলের তথ্য (`p` = package file) | `dpkg -I file.deb` |
| `rpm -qpl file.rpm` | `.rpm` ফাইলের ভেতরের তালিকা | `dpkg -c file.deb` |
| `rpm -qa --last` | সাম্প্রতিক ইনস্টল আগে | — |

```bash
$ rpm -qa | grep -i openssh
openssh-9.6p1-1.fc41.x86_64
openssh-clients-9.6p1-1.fc41.x86_64
openssh-server-9.6p1-1.fc41.x86_64

$ rpm -qf /usr/bin/ls
coreutils-9.5-8.fc41.x86_64

$ rpm -ql htop
/usr/bin/htop
/usr/share/applications/htop.desktop
/usr/share/doc/htop
/usr/share/doc/htop/README
/usr/share/man/man1/htop.1.gz
...

$ rpm -qi bash
Name        : bash
Version     : 5.2.32
Release     : 1.fc41
Architecture: x86_64
Install Date: Mon 04 Nov 2026 09:12:41 AM +06
Group       : Unspecified
Size        : 8592513
License     : GPL-3.0-or-later
Signature   : RSA/SHA256, ... Key ID ...
Source RPM  : bash-5.2.32-1.fc41.src.rpm
...
```

### install/upgrade/erase

| কমান্ড | কাজ |
|--------|-----|
| `sudo rpm -ivh file.rpm` | install, verbose, hash (#) প্রোগ্রেস বার |
| `sudo rpm -Uvh file.rpm` | upgrade (না থাকলে install) |
| `sudo rpm -Fvh file.rpm` | freshen — শুধু আগে থেকে থাকলে আপগ্রেড |
| `sudo rpm -e pkg` | erase (মোছো) |
| `rpm -V pkg` | verify — ইনস্টলের পর ফাইল বদলেছে কিনা |
| `rpm -K file.rpm` (`--checksig`) | `.rpm` ফাইলের সিগনেচার যাচাই |
| `sudo rpm --import key.asc` | GPG public key যোগ করো |

```bash
$ rpm -V openssh-server
S.5....T.  c /etc/ssh/sshd_config
```

এখানে `S` = আকার বদলেছে, `5` = checksum বদলেছে, `T` = সময় বদলেছে, `c` = এটা কনফিগ ফাইল। মানে কেউ `sshd_config` এডিট করেছে — স্বাভাবিক। কিন্তু `/usr/bin/` এর কোনো binary এভাবে বদলালে সন্দেহজনক (হ্যাকিং-এর লক্ষণ হতে পারে)! Debian-এ সমতুল্য: `dpkg --verify pkg` বা `debsums` প্যাকেজ।

> ⚠️ **সতর্কতা:** `rpm -ivh` dependency নিজে আনে না, ঠিক `dpkg -i` এর মতো। আর `rpm -e --nodeps` দিয়ে জোর করে মুছলে সিস্টেম ভেঙে যেতে পারে। দৈনন্দিন কাজে `dnf install ./file.rpm` ব্যবহার করো।

RPM ডাটাবেস থাকে `/var/lib/rpm/` এ (Fedora 36+ এ আসলে `/usr/lib/sysimage/rpm/`, আর `/var/lib/rpm` একটা symlink)। RHEL 9 ও Fedora-তে এটা একটা SQLite ডাটাবেস।

---

## ৩০.১২ Arch পরিবার: `pacman` ও AUR

Arch Linux, Manjaro, EndeavourOS — এখানে `pacman` একাই low-level ও high-level দুই কাজ করে। Arch হলো **rolling release** (রোলিং রিলিজ — "Arch 24.04" জাতীয় নির্দিষ্ট version নেই; প্রতিনিয়ত সব সর্বশেষ version পায়)।

pacman-এর syntax প্রথমে অদ্ভুত লাগে: একটা বড় হাতের **operation** অক্ষর, তারপর ছোট হাতের **modifier**।

| operation | মানে |
|-----------|------|
| `-S` | Sync — repo থেকে কাজ (ইনস্টল, খোঁজা) |
| `-R` | Remove |
| `-Q` | Query — লোকাল ডাটাবেস |
| `-U` | Upgrade — লোকাল প্যাকেজ ফাইল থেকে ইনস্টল |
| `-F` | Files — repo-র ফাইল তালিকায় খোঁজা |

### সবচেয়ে ব্যবহৃত pacman কমান্ড

| কমান্ড | কাজ | apt সমতুল্য |
|--------|-----|-------------|
| `sudo pacman -Syu` | index রিফ্রেশ + পুরো সিস্টেম আপগ্রেড | `apt update && apt full-upgrade` |
| `sudo pacman -S pkg` | ইনস্টল | `apt install` |
| `sudo pacman -Syu pkg` | আপগ্রেড করে তারপর ইনস্টল (নিরাপদ পদ্ধতি) | — |
| `pacman -Ss word` | repo-তে খোঁজো | `apt search` |
| `pacman -Si pkg` | repo-র প্যাকেজের তথ্য | `apt show` |
| `sudo pacman -R pkg` | শুধু প্যাকেজটা মোছো | `apt remove` |
| `sudo pacman -Rs pkg` | সাথে অপ্রয়োজনীয় dependency-ও | `apt remove` + `autoremove` |
| `sudo pacman -Rns pkg` | সাথে কনফিগ ব্যাকআপও (`.pacsave`) না রেখে | `apt purge` + `autoremove` |
| `pacman -Q` | সব ইনস্টল করা প্যাকেজ | `apt list --installed` |
| `pacman -Qe` | যেগুলো নিজে (explicitly) ইনস্টল করেছ | `apt-mark showmanual` |
| `pacman -Qi pkg` | ইনস্টল করা প্যাকেজের তথ্য | `dpkg -s` |
| `pacman -Ql pkg` | ফাইল তালিকা | `dpkg -L` |
| `pacman -Qo /path` | ফাইলটা কোন প্যাকেজের (owns) | `dpkg -S` |
| `pacman -Qdt` | অনাথ dependency | — |
| `pacman -F cmd` | কোন প্যাকেজে আছে (আগে `sudo pacman -Fy`) | `apt-file search` |
| `sudo pacman -U file.pkg.tar.zst` | লোকাল প্যাকেজ ইনস্টল | `apt install ./file.deb` |
| `sudo pacman -Sc` | পুরনো ক্যাশ মোছো | `apt autoclean` |

```bash
$ sudo pacman -Syu
:: Synchronizing package databases...
 core                  117.5 KiB   580 KiB/s 00:00
 extra                   7.7 MiB  4.21 MiB/s 00:02
:: Starting full system upgrade...
resolving dependencies...
looking for conflicting packages...

Packages (3) curl-8.11.0-1  linux-6.11.8.arch1-1  openssl-3.4.0-1

Total Download Size:   140.12 MiB
Total Installed Size:  145.87 MiB
Net Upgrade Size:        0.42 MiB

:: Proceed with installation? [Y/n]
```

সব অনাথ dependency একবারে মোছা:

```bash
$ sudo pacman -Rns $(pacman -Qdtq)
```

(`-q` = quiet, শুধু নাম; `$( )` হলো command substitution — [অধ্যায় ৪১](41-shell-scripting-1.md) দেখো।)

> ⚠️ **সতর্কতা: partial upgrade নিষিদ্ধ।** Arch-এ কখনো `sudo pacman -Sy pkg` (index রিফ্রেশ কিন্তু বাকি সিস্টেম আপগ্রেড না করে নতুন প্যাকেজ) কোরো না। এতে নতুন প্যাকেজ নতুন library চায়, কিন্তু বাকি সিস্টেম পুরনো library-তে — ফলে সিস্টেম ভাঙতে পারে। নিয়ম: হয় `-S pkg` (রিফ্রেশ ছাড়া), নয়তো `-Syu pkg`।

> 💡 **টিপস:** Arch-এ আপগ্রেডের আগে archlinux.org এর হোমপেজের "News" দেখে নাও — কখনো কখনো হাতে কিছু করতে বলা হয়। কনফিগ বদলালে pacman নতুন ভার্সন `.pacnew` নামে রাখে; `pacdiff` (pacman-contrib প্যাকেজ) দিয়ে মেলাও।

কনফিগ: `/etc/pacman.conf` (repo, `ParallelDownloads`, `IgnorePkg` = hold-এর সমতুল্য)। mirror তালিকা: `/etc/pacman.d/mirrorlist` (`reflector` টুল দিয়ে দ্রুততম mirror বাছাই করা যায়)। লগ: `/var/log/pacman.log`।

### pacman-এ GPG

pacman প্রতিটা প্যাকেজের সিগনেচার যাচাই করে `pacman-key` দিয়ে পরিচালিত keyring (`/etc/pacman.d/gnupg/`) এর সাহায্যে। Arch-এর ডেভেলপারদের key আসে `archlinux-keyring` প্যাকেজে। অনেকদিন আপডেট না করলে "signature is unknown trust" error আসতে পারে, তখন:

```bash
$ sudo pacman -Sy archlinux-keyring && sudo pacman -Syu
```

### AUR — Arch User Repository

**AUR** হলো কমিউনিটির একটা বিশাল সংগ্রহ, যেখানে কোনো বানানো প্যাকেজ থাকে না — থাকে শুধু **PKGBUILD** নামের একটা বিল্ড-রেসিপি (শেল স্ক্রিপ্ট), যা বলে সোর্স কোথা থেকে নামাতে হবে আর কীভাবে বিল্ড করতে হবে।

```
  AUR (aur.archlinux.org)            তোমার কম্পিউটার
  ┌─────────────────────┐            ┌─────────────────────────────┐
  │ PKGBUILD (রেসিপি)    │──git clone─▶│ makepkg: সোর্স নামায়, বিল্ড  │
  └─────────────────────┘            │  করে → .pkg.tar.zst বানায়   │
                                     │ pacman -U: ইনস্টল করে        │
                                     └─────────────────────────────┘
```

ম্যানুয়াল পদ্ধতি:

```bash
$ sudo pacman -S --needed base-devel git
$ git clone https://aur.archlinux.org/yay.git
$ cd yay
$ less PKGBUILD          # ← অবশ্যই পড়ো!
$ makepkg -si            # -s: dependency ইনস্টল, -i: বিল্ডের পর ইনস্টল
```

`makepkg` কে কখনো root হিসেবে চালাবে না — এটা নিজেই নিষেধ করে।

**AUR helper** (যেমন `yay`, `paru`) এই পুরো কাজ স্বয়ংক্রিয় করে, আর pacman-এর মতো syntax দেয়:

```bash
$ yay -S google-chrome     # AUR থেকে
$ yay -Syu                # অফিসিয়াল repo + AUR দুটোই আপগ্রেড
$ paru -Ss visual-studio-code
```

> ⚠️ **সতর্কতা:** AUR **অফিসিয়ালি সাপোর্টেড নয়**, যে কেউ প্যাকেজ আপলোড করতে পারে। অতীতে ম্যালওয়্যারযুক্ত PKGBUILD পাওয়া গেছে। প্রতিবার PKGBUILD পড়ো — বিশেষ করে `source=` লাইন আর `curl`/`wget` দিয়ে অন্য কিছু নামানো হচ্ছে কিনা। ভোট সংখ্যা আর কমেন্ট দেখো। AUR helper অফিসিয়াল repo-তে নেই, এটা প্রথমবার হাতে বিল্ড করতে হয়।

সোর্স থেকে বিল্ড সম্পর্কে বিস্তারিত [অধ্যায় ৩১](31-compile-from-source.md)-এ।

### অন্য distro-র package manager এক নজরে

| distro | high-level | low-level |
|--------|-----------|-----------|
| openSUSE | `zypper` (`zypper in`, `zypper up`, `zypper dup`) | `rpm` |
| Alpine (Docker-এ জনপ্রিয়) | `apk` (`apk add`, `apk del`, `apk update`) | `apk` |
| Gentoo | `emerge` (সোর্স থেকে বিল্ড) | — |
| Void | `xbps-install` | — |
| NixOS | `nix` (declarative, একদম আলাদা দর্শন) | — |

---

## ৩০.১৩ Distro-নিরপেক্ষ প্যাকেজ: snap, flatpak, AppImage

উপরের সব প্যাকেজ নির্দিষ্ট distro-র জন্য বানানো। Ubuntu 24.04-এর `.deb` Fedora-তে চলবে না, এমনকি Ubuntu 20.04-এ ঠিকঠাক না-ও চলতে পারে (library version আলাদা)। ডেভেলপারের জন্য ১০টা distro-র জন্য আলাদা প্যাকেজ বানানো যন্ত্রণা। সমাধান: **universal package** — অ্যাপ নিজের সব dependency সাথে নিয়ে আসে, আর সিস্টেম থেকে কিছুটা আলাদা (sandbox — বালির বাক্সের মতো আলাদা ঘেরা জায়গা) পরিবেশে চলে।

```
  প্রথাগত (deb/rpm)                    Universal (snap/flatpak)
  ┌──────┐ ┌──────┐ ┌──────┐          ┌──────────────┐ ┌──────────────┐
  │ App1 │ │ App2 │ │ App3 │          │ App1         │ │ App2         │
  └──┬───┘ └──┬───┘ └──┬───┘          │ + নিজস্ব lib  │ │ + নিজস্ব lib  │
     └────────┼────────┘              │ [sandbox]    │ │ [sandbox]    │
     ┌────────▼────────┐              └──────┬───────┘ └──────┬───────┘
     │ সিস্টেমের shared │                     │  runtime/base   │
     │ library (এক কপি) │              ┌──────▼─────────────────▼──────┐
     └─────────────────┘              │ snapd / flatpak + kernel       │
  ছোট, দ্রুত, distro-নির্ভর             └───────────────────────────────┘
                                       বড়, যেকোনো distro-তে, বেশি নিরাপদ
```

### snap (Canonical)

Ubuntu-তে ডিফল্টে চালু। ব্যাকগ্রাউন্ডে `snapd` ডেমন চলে। প্রতিটা snap একটা compressed SquashFS ইমেজ, যা `/snap/<name>/<revision>/` এ **mount** হয় (তাই `lsblk` বা `df` এ অনেক `loop` ডিভাইস দেখো — [অধ্যায় ৩২](32-disk-partition-mount.md))। Store একটাই: Snap Store (Canonical নিয়ন্ত্রিত)।

```bash
$ snap find vlc
Name  Version  Publisher   Notes  Summary
vlc   3.0.20   videolan✓   -      The ultimate media player

$ sudo snap install vlc
vlc 3.0.20 from VideoLAN✓ installed

$ snap list
Name     Version   Rev    Tracking         Publisher   Notes
core22   20240823  1621   latest/stable    canonical✓  base
firefox  132.0.1   5273   latest/stable/…  mozilla✓    -
vlc      3.0.20    3777   latest/stable    videolan✓   -

$ sudo snap refresh            # সব snap আপডেট (এমনিতে দিনে ৪ বার নিজেই চেক করে)
$ sudo snap remove vlc
$ snap info vlc                # চ্যানেল (stable/candidate/beta/edge) দেখাও
$ sudo snap install code --classic   # --classic = sandbox ছাড়া পূর্ণ অ্যাক্সেস (IDE-র দরকার হয়)
$ sudo snap revert vlc         # আগের revision-এ ফেরত যাও
$ snap connections vlc         # কোন অনুমতি (interface) আছে
```

> 📝 **মনে রাখো:** Ubuntu 22.04 থেকে `sudo apt install firefox` দিলে আসলে একটা transitional `.deb` ইনস্টল হয়, যা snap থেকে Firefox ইনস্টল করে। Ubuntu-তে Chromium-ও একইভাবে snap।

### flatpak

Fedora, Linux Mint সহ বেশিরভাগ ডেস্কটপ distro-তে জনপ্রিয়, কমিউনিটি-চালিত। মূল store **Flathub**। অ্যাপগুলো শেয়ার করা **runtime** (যেমন `org.gnome.Platform`) ব্যবহার করে, তাই প্রথম অ্যাপ বড় লাগলেও পরেরগুলো ছোট। sandbox হয় bubblewrap আর kernel namespace দিয়ে ([অধ্যায় ৪৮](48-containers-virtualization.md))।

```bash
$ sudo apt install flatpak                   # Ubuntu/Debian (Fedora-তে আগেই থাকে)
$ flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo

$ flatpak search gimp
Name   Description                 Application ID     Version  Branch  Remotes
GIMP   Create images and edit...   org.gimp.GIMP      2.10.38  stable  flathub

$ flatpak install flathub org.gimp.GIMP
$ flatpak run org.gimp.GIMP
$ flatpak list
$ flatpak update
$ flatpak uninstall org.gimp.GIMP
$ flatpak uninstall --unused                 # অব্যবহৃত runtime মোছো
$ flatpak info --show-permissions org.gimp.GIMP
```

> 💡 **টিপস:** flatpak অ্যাপের অনুমতি (কোন ফোল্ডার দেখতে পারবে ইত্যাদি) GUI দিয়ে বদলাতে **Flatseal** অ্যাপটা ব্যবহার করো (এটাও Flathub-এ আছে)। CLI-তে `flatpak override --user --filesystem=~/Documents org.gimp.GIMP`।

Fedora-তে বাইরের flatpak repo ডিফল্টে সীমিত থাকতে পারে; পুরো Flathub চালু করতে উপরের `remote-add` কমান্ড দাও।

### AppImage

সবচেয়ে সরল: **একটা ফাইল = একটা অ্যাপ**। কোনো ইনস্টল নেই, কোনো ডেমন নেই, root লাগে না।

```bash
$ chmod +x Obsidian-1.7.5.AppImage    # execute পারমিশন দাও (অধ্যায় ১৬)
$ ./Obsidian-1.7.5.AppImage
```

ভেতরে এটা একটা ELF executable + SquashFS ইমেজ। চালালে নিজেকে FUSE (Filesystem in Userspace) দিয়ে অস্থায়ীভাবে mount করে চালায়।

> ⚠️ **সতর্কতা:** অনেক AppImage-এর পুরনো `libfuse2` লাগে। Ubuntu 24.04-এ: `sudo apt install libfuse2t64` (22.04-এ নাম ছিল `libfuse2`)। `fuse` প্যাকেজ ইনস্টল কোরো না — এতে Ubuntu-তে `fuse3` সরে যেতে পারে আর ডেস্কটপের অংশ ভাঙতে পারে।

AppImage-এর অসুবিধা: স্বয়ংক্রিয় আপডেট নেই (কিছু AppImage নিজে করে), sandbox নেই, মেনুতে আইকন আসে না (Gear Lever-এর মতো টুল এটা ঠিক করে)। আর যেকোনো সাইট থেকে নামানো ফাইল চালাচ্ছ — উৎস বিশ্বস্ত হতে হবে।

### তুলনা

| বৈশিষ্ট্য | deb/rpm | snap | flatpak | AppImage |
|-----------|---------|------|---------|----------|
| কে বানিয়েছে | distro | Canonical | কমিউনিটি (freedesktop) | কমিউনিটি |
| store | distro repo | Snap Store (একটাই) | Flathub + যেকোনো remote | নেই (ওয়েবসাইট) |
| sandbox | না | হ্যাঁ (AppArmor) | হ্যাঁ (bubblewrap) | না |
| স্বয়ংক্রিয় আপডেট | distro নিয়মে | হ্যাঁ, ডিফল্টে | `flatpak update` / GUI | সাধারণত না |
| CLI/সার্ভার টুলের জন্য | সেরা | হ্যাঁ | মূলত GUI অ্যাপের জন্য | কিছুটা |
| ডিস্ক খরচ | সবচেয়ে কম | বেশি | বেশি (runtime শেয়ার হয়) | মাঝারি |
| root লাগে | হ্যাঁ | হ্যাঁ | না (`--user` দিয়ে) | না |
| প্রথম চালু হওয়ার গতি | দ্রুত | কিছুটা ধীর | দ্রুত | মাঝারি |

> 💡 **কোনটা কখন?** সার্ভার, কমান্ড-লাইন টুল, সিস্টেম কম্পোনেন্ট → সবসময় distro-র deb/rpm। ডেস্কটপ GUI অ্যাপের নতুন version → flatpak (বা Ubuntu-তে snap)। একবার চালিয়ে দেখার জন্য বা পোর্টেবল অ্যাপ → AppImage।

ভাষা-নির্দিষ্ট package manager-ও আছে: Python-এর `pip`, Node-এর `npm`, Rust-এর `cargo`। এগুলো দিয়ে সিস্টেম-ব্যাপী (`sudo pip install`) ইনস্টল কোরো না — Ubuntu 24.04 / Debian 12-এ `pip install` সিস্টেম Python-এ এখন "externally-managed-environment" error দেয় (PEP 668)। এর বদলে `python3 -m venv` দিয়ে virtual environment, বা `pipx` ব্যবহার করো।

---

## ৩০.১৪ GUI সফটওয়্যার সেন্টার

টার্মিনাল ছাড়াও:
- **Ubuntu 24.04:** "App Center" (snap ও deb দুটোই দেখায়)
- **GNOME (Fedora, Debian):** "GNOME Software"
- **KDE:** "Discover"

এগুলো ভেতরে **PackageKit** নামের একটা মধ্যস্থ স্তর ব্যবহার করে, যা apt/dnf/zypper এর সাথে কথা বলে। `pkcon` হলো তার CLI। মূল কথা: GUI আর CLI একই ডাটাবেস নিয়ে কাজ করে, তাই GUI-তে ইনস্টল করা জিনিস `apt list --installed` এ দেখা যাবে। ডেস্কটপ পরিবেশ নিয়ে আরও [অধ্যায় ৪](04-desktop-environment.md)-এ।

---

## ৩০.১৫ সমস্যা সমাধান (troubleshooting)

| লক্ষণ | সম্ভাব্য কারণ | সমাধান |
|-------|--------------|--------|
| `E: Unable to locate package xyz` | `apt update` করোনি, বা নাম ভুল, বা `universe` বন্ধ | `sudo apt update`; `apt search xyz`; `sudo add-apt-repository universe` |
| `Could not get lock /var/lib/dpkg/lock-frontend` | অন্য apt/dpkg চলছে | অপেক্ষা করো; `ps aux \| grep apt` |
| `dpkg was interrupted, you must manually run 'sudo dpkg --configure -a'` | আগের ইনস্টল মাঝপথে থেমেছিল | বার্তাতে যা বলছে তাই চালাও |
| `unmet dependencies` / `held broken packages` | ভুল repo/PPA, mixed রিলিজ | `sudo apt --fix-broken install`; `apt policy pkg` দিয়ে উৎস দেখো; সমস্যার PPA সরাও |
| `NO_PUBKEY` | repo-র GPG key নেই | অফিসিয়াল নির্দেশনা মতো `/etc/apt/keyrings/` এ key রাখো, `signed-by` দাও |
| `Release file ... is not valid yet` | কম্পিউটারের ঘড়ি ভুল | `timedatectl` দিয়ে সময় ঠিক করো |
| `404 Not Found` (পুরনো Ubuntu) | রিলিজের সাপোর্ট শেষ, repo `old-releases` এ সরেছে | রিলিজ আপগ্রেড করো |
| `/boot` ভরে গেছে | পুরনো kernel জমেছে | `sudo apt autoremove --purge` |
| dnf: `Error: GPG check FAILED` | key import হয়নি বা প্যাকেজ বদলানো | `rpm --import`; উৎস যাচাই করো |
| pacman: `conflicting files` | ফাইল অন্য কোথাও থেকে বসানো | `pacman -Qo file` দিয়ে মালিক দেখো, Arch Wiki দেখো |

> ⚠️ **সতর্কতা:** কোনো ফোরামে দেখা `sudo dpkg --force-all ...` বা `rpm --nodeps --force` জাতীয় কমান্ড না বুঝে চালাবে না। এগুলো ডাটাবেসকে মিথ্যা বলতে বাধ্য করে, পরে আরও বড় সমস্যা হয়।

---

## হাতে-কলমে অনুশীলন

> নিচের ল্যাবগুলো Ubuntu 24.04 বা Debian 12-এর জন্য। Fedora থাকলে বন্ধনীর কমান্ড ব্যবহার করো। ভার্চুয়াল মেশিনে ([অধ্যায় ৩](03-distro-o-installation.md)) করাই সবচেয়ে নিরাপদ।

### ল্যাব ১: প্রথম ইনস্টল

```bash
$ sudo apt update                       # (Fedora: dnf check-update)
$ apt list --upgradable
$ sudo apt upgrade -y                   # (sudo dnf upgrade -y)
$ apt show cowsay                       # (dnf info cowsay)
$ sudo apt install cowsay               # (sudo dnf install cowsay)
$ cowsay "Ami Linux shikhchi"
 ____________________
< Ami Linux shikhchi >
 --------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

Debian/Ubuntu-তে `cowsay` থাকে `/usr/games/` এ — PATH-এ না থাকলে `/usr/games/cowsay` লিখে চালাও।

### ল্যাব ২: গোয়েন্দাগিরি

1. `ls` কমান্ড কোন প্যাকেজের? `dpkg -S $(which ls)` (Fedora: `rpm -qf $(which ls)`)
2. `cowsay` প্যাকেজ কোন কোন ফাইল বসিয়েছে? `dpkg -L cowsay` (`rpm -ql cowsay`)
3. মোট কতগুলো প্যাকেজ ইনস্টল আছে? `dpkg -l | grep -c '^ii'` (`rpm -qa | wc -l`)
4. `cowsay` এর dependency কী? `apt-cache depends cowsay` (`dnf repoquery --requires cowsay`)
5. সর্বশেষ কী ইনস্টল করেছ? `tail -20 /var/log/apt/history.log` (`dnf history list`)

### ল্যাব ৩: remove বনাম purge

```bash
$ sudo apt install nginx
$ ls /etc/nginx/                     # কনফিগ আছে
$ sudo apt remove nginx
$ dpkg -l nginx nginx-common         # 'rc' অবস্থা দেখো
$ ls /etc/nginx/                     # এখনো আছে!
$ sudo apt purge nginx nginx-common
$ ls /etc/nginx/                     # এবার নেই (বা খালি)
$ sudo apt autoremove                # অনাথ dependency
```

### ল্যাব ৪: hold

```bash
$ sudo apt-mark hold cowsay
$ apt-mark showhold
$ dpkg -l cowsay                     # 'hi'
$ sudo apt-mark unhold cowsay
```

### ল্যাব ৫: লোকাল .deb ও ভাঙা dependency

```bash
$ mkdir ~/pkglab && cd ~/pkglab
$ apt download htop
$ dpkg-deb --info htop_*.deb | grep Depends
$ dpkg-deb --contents htop_*.deb | head
$ sudo apt install ./htop_*.deb      # ./ লক্ষ করো
```

### ল্যাব ৬: থার্ড-পার্টি repo নিরাপদে যোগ করা

Docker-এর অফিসিয়াল নির্দেশনা (docs.docker.com) খোলো, আর ৩০.৯ অংশের মতো `signed-by` সহ repo যোগ করো। তারপর:

```bash
$ apt policy docker-ce               # কোন repo থেকে আসছে দেখো
$ gpg --show-keys /etc/apt/keyrings/docker.asc   # fingerprint মিলাও
```

শেষে পরিষ্কার: `/etc/apt/sources.list.d/docker.list` ও key ফাইল মুছে `sudo apt update`।

### ল্যাব ৭: universal package

```bash
$ snap list                          # Ubuntu-তে আগে থেকেই কী আছে
$ flatpak --version || sudo apt install flatpak
$ flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
$ flatpak search calculator
$ lsblk | grep loop                  # snap-এর mount দেখো
```

### চ্যালেঞ্জ

একটা স্ক্রিপ্ট লেখো (অধ্যায় ৪১-এর পরে আবার ফিরে এসো) যা `/etc/os-release` পড়ে distro বুঝবে, আর সঠিক কমান্ড দিয়ে `curl git htop` ইনস্টল করবে — `apt-get`, `dnf`, বা `pacman` যেটা প্রযোজ্য।

---

## সারসংক্ষেপ

### তিন পরিবারের তুলনামূলক চিটশিট

| কাজ | Debian/Ubuntu | Fedora/RHEL | Arch |
|-----|---------------|-------------|------|
| index রিফ্রেশ | `sudo apt update` | `dnf makecache` | `sudo pacman -Sy` (একা নয়!) |
| সব আপগ্রেড | `sudo apt upgrade` / `full-upgrade` | `sudo dnf upgrade` | `sudo pacman -Syu` |
| ইনস্টল | `sudo apt install pkg` | `sudo dnf install pkg` | `sudo pacman -S pkg` |
| লোকাল ফাইল ইনস্টল | `sudo apt install ./f.deb` | `sudo dnf install ./f.rpm` | `sudo pacman -U f.pkg.tar.zst` |
| মোছো | `sudo apt remove pkg` | `sudo dnf remove pkg` | `sudo pacman -R pkg` |
| কনফিগ সহ মোছো | `sudo apt purge pkg` | (প্রযোজ্য নয়) | `sudo pacman -Rns pkg` |
| অনাথ মোছো | `sudo apt autoremove` | `sudo dnf autoremove` | `sudo pacman -Rns $(pacman -Qdtq)` |
| খোঁজো | `apt search word` | `dnf search word` | `pacman -Ss word` |
| তথ্য | `apt show pkg` | `dnf info pkg` | `pacman -Si pkg` |
| ইনস্টল তালিকা | `apt list --installed` / `dpkg -l` | `dnf list installed` / `rpm -qa` | `pacman -Q` |
| ফাইলের তালিকা | `dpkg -L pkg` | `rpm -ql pkg` | `pacman -Ql pkg` |
| ফাইলটা কার | `dpkg -S /path` | `rpm -qf /path` | `pacman -Qo /path` |
| না-থাকা ফাইল খোঁজো | `apt-file search f` | `dnf provides f` | `pacman -F f` |
| version উৎস | `apt policy pkg` | `dnf list --showduplicates pkg` | `pacman -Si pkg` |
| আপডেট আটকাও | `apt-mark hold pkg` | `dnf versionlock add pkg` | `IgnorePkg` in `pacman.conf` |
| ইতিহাস | `/var/log/apt/history.log` | `dnf history` | `/var/log/pacman.log` |
| ক্যাশ মোছো | `sudo apt clean` | `sudo dnf clean all` | `sudo pacman -Sc` |
| repo কনফিগ | `/etc/apt/sources.list(.d/)` | `/etc/yum.repos.d/*.repo` | `/etc/pacman.conf` |
| ভাঙা মেরামত | `dpkg --configure -a`; `apt -f install` | `dnf distro-sync` | `pacman -Syu` |

### universal package

| কাজ | snap | flatpak | AppImage |
|-----|------|---------|----------|
| খোঁজো | `snap find x` | `flatpak search x` | ওয়েবসাইট |
| ইনস্টল | `sudo snap install x` | `flatpak install flathub id` | `chmod +x f.AppImage` |
| চালাও | `x` | `flatpak run id` | `./f.AppImage` |
| তালিকা | `snap list` | `flatpak list` | — |
| আপডেট | `sudo snap refresh` | `flatpak update` | নতুন ফাইল নামাও |
| মোছো | `sudo snap remove x` | `flatpak uninstall id` | `rm f.AppImage` |

### কীবোর্ড শর্টকাট

প্যাকেজ ম্যানেজমেন্টের নিজস্ব কোনো শর্টকাট নেই, তবে কাজের সময় এগুলো লাগে ([অধ্যায় ৬](06-terminal-shortcuts.md)):

| শর্টকাট | কাজ |
|---------|-----|
| `Tab` (দুইবার) | `apt install vl` + Tab Tab → প্যাকেজের নাম auto-complete (bash-completion থাকলে) |
| `Ctrl + C` | চলমান ডাউনলোড বাতিল (ইনস্টল-পর্বে চাপবে না!) |
| `Ctrl + R` | হিস্ট্রিতে আগের `apt install` কমান্ড খোঁজো |
| `!!` → `sudo !!` | `sudo` দিতে ভুলে গেলে আগের কমান্ড sudo দিয়ে আবার চালাও |
| `q` | `apt show` / `apt list` এর output pager-এ খুললে বের হও |
| `Y` / `Enter` | apt-এ নিশ্চিত করো (ডিফল্ট হ্যাঁ) |
| `y` + `Enter` | dnf-এ নিশ্চিত করো (ডিফল্ট না) |

---

## নিজেকে যাচাই করো

1. `apt update` আর `apt upgrade` এর মধ্যে পার্থক্য কী?
2. low-level আর high-level প্যাকেজ টুলের পার্থক্য একটা উপমা দিয়ে বোঝাও। Debian ও RHEL পরিবারে প্রত্যেক স্তরের একটা করে টুলের নাম বলো।
3. `apt remove` আর `apt purge` এর পার্থক্য কী? `dpkg -l` এ `rc` মানে কী?
4. `/usr/bin/curl` ফাইলটা কোন প্যাকেজের, Ubuntu, Fedora ও Arch-এ কীভাবে জানবে?
5. `apt-key add` কেন deprecated? এর বদলে কী করা উচিত?
6. `sudo dpkg -i app.deb` দিলে dependency error এলো। কী করবে? আর এই সমস্যা এড়ানোর ভালো উপায় কী ছিল?
7. Arch-এ `sudo pacman -Sy firefox` কেন বিপজ্জনক?
8. dnf-এ গতকালের একটা আপডেট উল্টাতে চাও। কোন কমান্ড?
9. snap আর flatpak-এর দুইটা মিল আর দুইটা অমিল বলো।
10. একটা প্যাকেজকে আপগ্রেড থেকে আটকে রাখতে Debian আর RHEL-এ কী করবে, আর এর ঝুঁকি কী?

<details><summary>উত্তর দেখো</summary>

1. `apt update` শুধু repo থেকে প্যাকেজের সর্বশেষ **তালিকা (index)** নামিয়ে `/var/lib/apt/lists/` এ রাখে; কোনো সফটওয়্যার বদলায় না। `apt upgrade` সেই তালিকা অনুযায়ী ইনস্টল করা প্যাকেজগুলোকে নতুন version-এ আপডেট করে। তাই সাধারণত আগে update, তারপর upgrade।

2. low-level টুল (রাজমিস্ত্রি) একটা লোকাল প্যাকেজ ফাইল খুলে ফাইল বসায় ও ডাটাবেস আপডেট করে, কিন্তু ডাউনলোড বা dependency সমাধান করে না। high-level টুল (ঠিকাদার) repo থেকে খোঁজে, dependency হিসাব করে, সিগনেচার যাচাই করে, ডাউনলোড করে, তারপর low-level টুল দিয়ে ইনস্টল করায়। Debian: `dpkg` (low) ও `apt` (high); RHEL: `rpm` (low) ও `dnf` (high)।

3. `remove` প্রোগ্রামের ফাইল মোছে কিন্তু `/etc` এর কনফিগ ফাইল রেখে দেয়; `purge` কনফিগ সহ মোছে (home ডিরেক্টরির ব্যক্তিগত ফাইল কোনোটাই মোছে না)। `rc` মানে: desired = **r**emove, status = **c**onfig-files — প্যাকেজ মোছা হয়েছে, কনফিগ রয়ে গেছে।

4. Ubuntu: `dpkg -S /usr/bin/curl`; Fedora: `rpm -qf /usr/bin/curl`; Arch: `pacman -Qo /usr/bin/curl`।

5. `apt-key` key-গুলো একটা global keyring-এ রাখত, ফলে যেকোনো থার্ড-পার্টি key দিয়ে **যেকোনো** repo-র (এমনকি অফিসিয়াল) প্যাকেজ বিশ্বস্ত বলে মেনে নেওয়া হতো। এখন প্রতিটা key আলাদা ফাইলে `/etc/apt/keyrings/` এ রাখা হয়, আর repo-র লাইনে `signed-by=/etc/apt/keyrings/x.gpg` (বা deb822-এ `Signed-By:`) দিয়ে বলে দেওয়া হয় এই repo শুধু এই key দিয়ে যাচাই হবে।

6. `sudo apt --fix-broken install` (বা `sudo apt -f install`) চালাবে, যা হারানো dependency এনে কনফিগারেশন শেষ করবে। এড়ানোর ভালো উপায় ছিল শুরুতেই `sudo apt install ./app.deb` ব্যবহার করা।

7. `-Sy` index রিফ্রেশ করে কিন্তু বাকি সিস্টেম আপগ্রেড করে না। ফলে firefox-এর নতুন version নতুন library চাইতে পারে, অথচ বাকি সিস্টেম পুরনো library-তে — একে partial upgrade বলে, আর এটা Arch-এ সাপোর্টেড নয়, সিস্টেম ভাঙতে পারে। সঠিক: `sudo pacman -Syu firefox`।

8. `sudo dnf history list` দিয়ে transaction ID খুঁজে বের করো, তারপর `sudo dnf history undo <ID>`। (পুরনো version repo-তে থাকতে হবে।)

9. মিল: দুটোই distro-নিরপেক্ষ, অ্যাপের dependency সাথে নিয়ে আসে, sandbox-এ চালায়। অমিল: snap Canonical-এর, store একটাই (Snap Store) ও `snapd` ডেমন লাগে, SquashFS ইমেজ loop ডিভাইসে mount হয়, সার্ভার/CLI অ্যাপও বিতরণ করে; flatpak কমিউনিটি-চালিত, যেকোনো remote (মূলত Flathub) যোগ করা যায়, শেয়ার করা runtime ব্যবহার করে, মূলত ডেস্কটপ GUI অ্যাপের জন্য, root ছাড়া `--user` ইনস্টল সম্ভব।

10. Debian: `sudo apt-mark hold pkg` (ফেরত: `unhold`, দেখো: `showhold`)। RHEL: `sudo dnf versionlock add pkg` (dnf4-এ `python3-dnf-plugin-versionlock` লাগে), বা `dnf.conf` এ `exclude=`। ঝুঁকি: প্যাকেজটা নিরাপত্তা আপডেটও পাবে না; অনেকদিন hold থাকলে অন্য প্যাকেজের dependency আটকে যেতে পারে।

</details>

---

[← আগের অধ্যায়](29-scheduling-cron.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](31-compile-from-source.md)

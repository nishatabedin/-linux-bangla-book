# অধ্যায় ২: লিনাক্স আর্কিটেকচার — ভেতরের গঠন

> "একটা গাড়ি চালাতে ইঞ্জিনের ভেতরটা জানতে হয় না। কিন্তু গাড়ি নষ্ট হলে ঠিক করতে চাইলে, বা রেসিং ড্রাইভার হতে চাইলে — ইঞ্জিন বুঝতেই হবে।"

আগের অধ্যায়ে ([অধ্যায় ১](01-linux-ki-o-keno.md)) আমরা জেনেছি লিনাক্স কী আর কেন শিখব। এই অধ্যায়ে আমরা লিনাক্সের "ইঞ্জিনের ঢাকনা" খুলে ভেতরে তাকাব। এখানে যা শিখবে, তা বাকি পুরো বইয়ের ভিত্তি। পরে যখন প্রসেস, পারমিশন, ফাইলসিস্টেম, নেটওয়ার্ক শিখবে, তখন বারবার মনে হবে — "ও আচ্ছা, এটা তো সেই kernel-এর কাজ!"

ভয় পেয়ো না। এই অধ্যায়ে কিছু নতুন শব্দ আসবে, কিন্তু প্রতিটি শব্দ আমরা উপমা দিয়ে, ধাপে ধাপে বুঝব।

---

## এই অধ্যায়ে যা শিখবে

- লিনাক্স সিস্টেম কয়টা স্তরে (layer) সাজানো — hardware থেকে application পর্যন্ত
- **kernel** আসলে কী, আর "লিনাক্স" বলতে আসলে কী বোঝায়
- **kernel space** আর **user space** কী, কেন এই ভাগাভাগি দরকার
- CPU-এর **privilege ring** (ring 0 / ring 3) কীভাবে নিরাপত্তা দেয়
- **monolithic kernel**, **microkernel** ও **hybrid kernel**-এর পার্থক্য
- **loadable kernel module** কী — `lsmod`, `modinfo`, `modprobe`
- kernel-এর পাঁচটি প্রধান কাজ: process scheduling, memory management, VFS, device drivers, networking
- **system call** কী, কীভাবে কাজ করে, আর `strace` দিয়ে কীভাবে তা চোখে দেখা যায়
- **glibc** ও অন্যান্য library-র ভূমিকা — `ldd` দিয়ে দেখা
- **shell** কোথায় বসে, কী করে
- লিনাক্সের বিখ্যাত দর্শন: **"Everything is a file"**
- টার্মিনালে `ls` লিখে Enter চাপলে ভেতরে ঠিক কী কী ঘটে — **fork → exec → wait** ধাপে ধাপে
- architecture-সম্পর্কিত কীবোর্ড শর্টকাট (signal, virtual console, Magic SysRq)

---

## ২.১ বড় ছবি: লিনাক্স সিস্টেমের স্তরগুলো

একটা লিনাক্স কম্পিউটারকে একটা বহুতল ভবন হিসেবে কল্পনা করো। সবার নিচে মাটি আর ভিত্তি (hardware), তার ওপর ইঞ্জিনিয়ারিং কন্ট্রোল রুম (kernel), তার ওপর রিসেপশন ডেস্ক (system call ইন্টারফেস), তার ওপর বিভিন্ন সার্ভিস অফিস (libraries), আর একদম ওপরের তলায় সাধারণ মানুষ তাদের কাজ করে (applications ও shell)।

```
 ┌──────────────────────────────────────────────────────────────────┐
 │                         তুমি (User)                               │
 └──────────────────────────────────────────────────────────────────┘
                 │ কীবোর্ড / মাউস / টাচ
                 ▼
 ┌──────────────────────────────────────────────────────────────────┐
 │  APPLICATIONS          firefox, vim, python, nginx, ls, cp ...    │
 │  SHELL                 bash, zsh, fish  (কমান্ড ইন্টারপ্রেটার)       │  USER
 ├──────────────────────────────────────────────────────────────────┤  SPACE
 │  LIBRARIES             glibc (libc.so.6), libm, libssl, libpthread│  (ring 3)
 │                        ld-linux.so (dynamic linker/loader)        │
 ╞══════════════════════════════════════════════════════════════════╡
 │  SYSTEM CALL INTERFACE   read() write() open() fork() execve() ...│ ◄── সীমান্ত
 ╞══════════════════════════════════════════════════════════════════╡
 │  KERNEL                                                           │
 │   ┌────────────┐ ┌────────────┐ ┌──────────┐ ┌─────────┐ ┌──────┐ │  KERNEL
 │   │ Process    │ │ Memory     │ │ VFS +    │ │ Network │ │Device│ │  SPACE
 │   │ Scheduler  │ │ Management │ │ File-    │ │ Stack   │ │Driver│ │  (ring 0)
 │   │            │ │            │ │ systems  │ │ (TCP/IP)│ │  s   │ │
 │   └────────────┘ └────────────┘ └──────────┘ └─────────┘ └──────┘ │
 │          Architecture-specific code (x86_64, arm64, riscv ...)     │
 ╞══════════════════════════════════════════════════════════════════╡
 │  HARDWARE     CPU, RAM, Disk (SSD/HDD/NVMe), Network card, GPU,   │
 │               USB, কীবোর্ড, স্ক্রিন ...                               │
 └──────────────────────────────────────────────────────────────────┘
```

প্রতিটি স্তর শুধু তার ঠিক নিচের স্তরের সাথে কথা বলে (মোটামুটি)। Firefox কখনো সরাসরি নেটওয়ার্ক কার্ডে বিদ্যুৎ সংকেত পাঠায় না। সে glibc-র একটা ফাংশন ডাকে, glibc একটা system call করে, kernel সেটা গ্রহণ করে network stack আর driver-এর মাধ্যমে hardware-কে নির্দেশ দেয়।

### প্রতিটি স্তর সংক্ষেপে

| স্তর | কী | উদাহরণ | উপমা |
|---|---|---|---|
| Hardware | ভৌত যন্ত্রাংশ | CPU, RAM, SSD, NIC | ভবনের মাটি, ইট, বিদ্যুৎ লাইন |
| Kernel | OS-এর কেন্দ্র; hardware নিয়ন্ত্রণ করে | Linux kernel 6.x | ভবনের ম্যানেজার ও কন্ট্রোল রুম |
| System call | user প্রোগ্রাম kernel-কে অনুরোধ করার একমাত্র দরজা | `read`, `write`, `openat` | রিসেপশন ডেস্কের ফর্ম |
| Libraries | প্রোগ্রামারদের জন্য সহজ ফাংশন; ভেতরে system call করে | glibc, OpenSSL | ফর্ম পূরণে সাহায্যকারী কেরানি |
| Shell | তোমার লেখা কমান্ড বুঝে প্রোগ্রাম চালায় | bash, zsh | দোভাষী |
| Applications | আসল কাজের প্রোগ্রাম | Firefox, LibreOffice, `ls` | ভবনের বাসিন্দা |

> 📝 **মনে রাখো:** "Linux" শব্দটা কঠোর অর্থে শুধু **kernel**-কে বোঝায়, যেটা ১৯৯১ সালে Linus Torvalds লেখা শুরু করেন। আমরা যে পুরো অপারেটিং সিস্টেম ব্যবহার করি (kernel + GNU টুল + library + shell + desktop), তাকে অনেকে **GNU/Linux** বলেন। প্রতিদিনের কথায় পুরোটাকেই "লিনাক্স" বলা হয় — এই বইতেও তাই বলব, তবে পার্থক্যটা জেনে রাখো।

---

## ২.২ Hardware — সবার নিচের স্তর

kernel-এর কাজ বুঝতে hardware সম্পর্কে অল্প কিছু জানা দরকার:

- **CPU (Central Processing Unit):** নির্দেশ (instruction) একটার পর একটা চালায়। আধুনিক CPU-তে একাধিক **core** থাকে; প্রতিটি core একসাথে একটা করে কাজ চালাতে পারে (hyper-threading থাকলে একটা core দুটো hardware thread চালায়)।
- **RAM (Memory):** দ্রুত কিন্তু অস্থায়ী (বিদ্যুৎ গেলে মুছে যায়) জায়গা, যেখানে চলমান প্রোগ্রাম ও তার ডেটা থাকে।
- **Storage (HDD/SSD/NVMe):** ধীর কিন্তু স্থায়ী জায়গা।
- **I/O devices:** কীবোর্ড, মাউস, স্ক্রিন, নেটওয়ার্ক কার্ড (NIC), USB ইত্যাদি।
- **MMU (Memory Management Unit):** CPU-র ভেতরের একটা অংশ, যেটা "virtual address"-কে "physical address"-এ অনুবাদ করে। (এটা পরে memory management-এ কাজে লাগবে।)
- **Interrupt:** hardware যখন CPU-র মনোযোগ চায় (যেমন কীবোর্ডে কী চাপা হলো, বা নেটওয়ার্কে প্যাকেট এলো), তখন সে একটা interrupt সংকেত পাঠায়। CPU যা করছিল থামিয়ে kernel-এর একটা নির্দিষ্ট কোড (interrupt handler) চালায়।

তোমার মেশিনের hardware দেখতে কয়েকটা কমান্ড:

```
$ lscpu | head -8
Architecture:             x86_64
  CPU op-mode(s):         32-bit, 64-bit
  Address sizes:          39 bits physical, 48 bits virtual
  Byte Order:             Little Endian
CPU(s):                   8
  On-line CPU(s) list:    0-7
Vendor ID:                GenuineIntel
  Model name:             Intel(R) Core(TM) i5-10210U CPU @ 1.60GHz

$ free -h
               total        used        free      shared  buff/cache   available
Mem:            15Gi       4.2Gi       6.1Gi       512Mi       5.3Gi        10Gi
Swap:          2.0Gi          0B       2.0Gi
```

(তোমার মেশিনে সংখ্যাগুলো আলাদা হবে।) এগুলো বিস্তারিত আমরা [অধ্যায় ৪৬](46-performance-troubleshooting.md)-এ দেখব।

---

## ২.৩ Kernel — অপারেটিং সিস্টেমের হৃদয়

**Kernel** (কার্নেল, শাব্দিক অর্থ "শস্যদানার শাঁস" বা "কেন্দ্র") হলো সেই প্রোগ্রাম, যেটা কম্পিউটার চালু হওয়ার পর সবার আগে মেমোরিতে লোড হয় এবং বন্ধ হওয়া পর্যন্ত সবসময় চলতে থাকে। এটা hardware আর বাকি সব প্রোগ্রামের মাঝখানে বসে থাকা **মধ্যস্থতাকারী**।

### উপমা: রেস্তোরাঁর রান্নাঘর

একটা ব্যস্ত রেস্তোরাঁর কথা ভাবো:

- **খদ্দের** (applications) রান্নাঘরে ঢুকতে পারে না। তারা শুধু মেনু থেকে অর্ডার দেয়।
- **মেনু** হলো system call-এর তালিকা — কী কী চাওয়া যাবে, তা নির্দিষ্ট।
- **ওয়েটার** (library/glibc) অর্ডারটা সঠিক ফরম্যাটে লিখে রান্নাঘরে পৌঁছে দেয়।
- **হেড শেফ** (kernel) ঠিক করে কোন রাঁধুনি কখন কোন চুলা ব্যবহার করবে (scheduling), কোন তাকে কোন উপকরণ থাকবে (memory), ফ্রিজ-গুদাম কীভাবে সাজানো (filesystem)।
- **চুলা, ছুরি, ফ্রিজ** হলো hardware। প্রতিটি যন্ত্র চালানোর আলাদা কায়দা আছে — সেই কায়দা জানে **device driver**।

খদ্দেররা যদি সরাসরি রান্নাঘরে ঢুকে যে যার মতো চুলা দখল করত, তাহলে বিশৃঙ্খলা, আগুন, চুরি — সব হতো। kernel ঠিক এটাই আটকায়।

### Kernel-এর মূল দায়িত্ব এক নজরে

1. **Process management** — কোন প্রোগ্রাম কখন CPU পাবে
2. **Memory management** — কে কতটুকু RAM পাবে, কেউ যেন অন্যের মেমোরিতে হাত না দেয়
3. **Filesystem (VFS)** — ডিস্কের ডেটা ফাইল-ফোল্ডার আকারে দেখানো
4. **Device drivers** — hardware-এর সাথে কথা বলা
5. **Networking** — TCP/IP দিয়ে ডেটা পাঠানো-আনা
6. **Security ও access control** — কে কী করতে পারবে (permissions, capabilities, SELinux/AppArmor)
7. **System call interface** — user প্রোগ্রামকে এসব সেবা দেওয়ার দরজা

### তোমার kernel-এর তথ্য দেখো

```
$ uname -r
6.8.0-45-generic

$ uname -a
Linux mypc 6.8.0-45-generic #45-Ubuntu SMP PREEMPT_DYNAMIC Fri Aug 30 12:02:04 UTC 2024 x86_64 x86_64 x86_64 GNU/Linux
```

| `uname` অপশন | কী দেখায় |
|---|---|
| `-s` | kernel-এর নাম (`Linux`) |
| `-n` | hostname (মেশিনের নাম) |
| `-r` | kernel release (ভার্সন) |
| `-v` | kernel build-এর তথ্য |
| `-m` | hardware architecture (`x86_64`, `aarch64`) |
| `-o` | অপারেটিং সিস্টেম (`GNU/Linux`) |
| `-a` | সব একসাথে |

kernel ভার্সন নম্বর পড়ার নিয়ম: `6.8.0-45-generic` মানে mainline kernel 6.8, তার ওপর distro-র (এখানে Ubuntu) নিজস্ব patch লেভেল `0-45`, আর `generic` হলো kernel flavour। Ubuntu 24.04 kernel 6.8 দিয়ে আসে, Debian 12 আসে 6.1 দিয়ে, RHEL 9 আসে 5.14 দিয়ে (Red Hat অনেক নতুন ফিচার backport করে), Fedora সাধারণত সবচেয়ে নতুন stable kernel রাখে।

kernel-এর ফাইলটা নিজে থাকে `/boot`-এ:

```
$ ls /boot
config-6.8.0-45-generic      initrd.img-6.8.0-45-generic  vmlinuz-6.8.0-45-generic
System.map-6.8.0-45-generic  grub                          ...
```

`vmlinuz` হলো compressed kernel image ("vm" = virtual memory সাপোর্ট, "z" = compressed)। `initrd.img`/`initramfs` হলো বুটের সময় ব্যবহৃত ছোট একটা অস্থায়ী root filesystem। এগুলো কীভাবে লোড হয়, তা [অধ্যায় ২৭](27-boot-process.md)-এ বিস্তারিত।

---

## ২.৪ Kernel Space বনাম User Space

এটা এই অধ্যায়ের সবচেয়ে গুরুত্বপূর্ণ ধারণাগুলোর একটা।

মেমোরি আর CPU-র ক্ষমতাকে লিনাক্স দুটো জগতে ভাগ করে:

- **Kernel space (কার্নেল জগৎ):** এখানে শুধু kernel-এর কোড চলে। এখানে কোড hardware-এ সরাসরি হাত দিতে পারে, যেকোনো মেমোরি পড়তে-লিখতে পারে, যেকোনো CPU instruction চালাতে পারে। পূর্ণ ক্ষমতা।
- **User space (ব্যবহারকারী জগৎ):** তোমার সব সাধারণ প্রোগ্রাম — bash, Firefox, Python, এমনকি root হিসেবে চলা প্রোগ্রামও — এখানে চলে। এখানে ক্ষমতা সীমিত: নিজের মেমোরি ছাড়া অন্য কিছুতে হাত দেওয়া যায় না, hardware সরাসরি ছোঁয়া যায় না।

```
          USER SPACE                        KERNEL SPACE
   ┌──────────────────────┐          ┌──────────────────────────┐
   │ bash  │ firefox │ vim│          │  kernel code + data      │
   │ (প্রতিটির নিজের আলাদা  │          │  (সব প্রসেসের জন্য একটাই)   │
   │  virtual memory)     │          │                          │
   │                      │ ──syscall──►  hardware-এ পূর্ণ অধিকার │
   │ সীমিত ক্ষমতা          │ ◄──result──                         │
   └──────────────────────┘          └──────────────────────────┘
          ring 3                              ring 0
```

### কেন এই ভাগাভাগি?

1. **স্থিতিশীলতা (Stability):** একটা প্রোগ্রাম crash করলে শুধু সেই প্রোগ্রামটাই মরে, পুরো সিস্টেম না। Firefox হ্যাং করলে তুমি তাকে মেরে ফেলো, কম্পিউটার চলতে থাকে।
2. **নিরাপত্তা (Security):** একটা ক্ষতিকর বা বাগযুক্ত প্রোগ্রাম অন্য প্রোগ্রামের মেমোরি (যেমন তোমার ব্রাউজারের পাসওয়ার্ড) পড়তে পারে না।
3. **ন্যায্য বণ্টন (Fairness):** কোনো প্রোগ্রাম CPU বা ডিস্ক একা দখল করে রাখতে পারে না, কারণ বণ্টন করে kernel।
4. **বিমূর্তকরণ (Abstraction):** প্রোগ্রামারকে জানতে হয় না ডিস্কটা SATA না NVMe — সে শুধু `write()` ডাকে।

### CPU Privilege Ring — hardware-স্তরের পাহারা

এই ভাগাভাগি শুধু "নিয়ম" না — CPU নিজেই এটা জোর করে মানায়। x86 CPU-তে চারটা **privilege level** বা **ring** আছে (0 থেকে 3)। লিনাক্স ব্যবহার করে মাত্র দুটো:

```
            ┌───────────────────────────┐
            │         Ring 3            │  ← user space (সব অ্যাপ)
            │    ┌─────────────────┐    │
            │    │  Ring 2 (অব্যবহৃত)│    │
            │    │  ┌───────────┐  │    │
            │    │  │ Ring 1    │  │    │
            │    │  │ ┌───────┐ │  │    │
            │    │  │ │Ring 0 │ │  │    │  ← kernel (সর্বোচ্চ ক্ষমতা)
            │    │  │ └───────┘ │  │    │
            │    │  └───────────┘  │    │
            │    └─────────────────┘    │
            └───────────────────────────┘
```

ARM64 CPU-তে একই ধারণাকে বলে **Exception Level** — EL0 (user), EL1 (kernel), EL2 (hypervisor)।

Ring 3-এ চলা কোড যদি কোনো "বিশেষাধিকারপ্রাপ্ত" instruction চালাতে চায় (যেমন interrupt বন্ধ করা, বা page table বদলানো), CPU নিজেই একটা exception তোলে, kernel সেটা ধরে, আর প্রোগ্রামটাকে সাধারণত একটা signal (যেমন `SIGSEGV` বা `SIGILL`) দিয়ে মেরে ফেলে।

> 💡 **টিপস:** "Segmentation fault (core dumped)" মেসেজটা দেখেছ? এর মানে একটা প্রোগ্রাম এমন মেমোরিতে হাত দিতে চেয়েছিল যেটা তার না। MMU সেটা ধরেছে, kernel প্রোগ্রামটাকে `SIGSEGV` signal পাঠিয়ে থামিয়েছে। সিস্টেমের বাকি সবকিছু নিরাপদ।

> ⚠️ **সতর্কতা:** **root user** আর **kernel space** এক জিনিস না! root হিসেবে চলা প্রোগ্রামও user space-এ (ring 3) চলে। root-এর বিশেষত্ব হলো kernel তার অনুরোধগুলো বেশি অনুমোদন করে (যেমন অন্যের ফাইল পড়া)। কিন্তু root-এর প্রোগ্রামও hardware-এ সরাসরি না, system call-এর মাধ্যমেই কাজ করে। root ও sudo নিয়ে বিস্তারিত [অধ্যায় ১৮](18-sudo-root.md)-এ।

### Mode switch — দুই জগতের মধ্যে যাতায়াত

একটা প্রোগ্রাম সারাজীবন user mode-এ থাকে না। তিনটা উপায়ে CPU user mode থেকে kernel mode-এ যায়:

| উপায় | কখন হয় | উদাহরণ |
|---|---|---|
| **System call** | প্রোগ্রাম নিজে ইচ্ছা করে kernel-এর সেবা চায় | ফাইল পড়া, প্রিন্ট করা |
| **Exception / fault** | প্রোগ্রাম এমন কিছু করেছে যা kernel-এর সামলানো দরকার | page fault, শূন্য দিয়ে ভাগ |
| **Hardware interrupt** | কোনো device মনোযোগ চাইছে | কীবোর্ডে কী চাপা, timer tick |

প্রতিবার এই সীমানা পার হতে অল্প হলেও সময় লাগে (CPU-র অবস্থা সংরক্ষণ করা, নিরাপত্তা যাচাই)। তাই দক্ষ প্রোগ্রাম অপ্রয়োজনীয় system call কমায় — যেমন এক বাইট এক বাইট করে না পড়ে একসাথে ৪ KB বা ৬৪ KB পড়ে।

---

## ২.৫ Kernel-এর ডিজাইন: Monolithic, Microkernel ও Hybrid

অপারেটিং সিস্টেমের kernel বানানোর কয়েকটা দর্শন আছে।

### Monolithic kernel (একশিলা kernel)

সব প্রধান সেবা — scheduler, memory manager, filesystem, driver, network stack — **একটাই বড় প্রোগ্রামে**, একই kernel space-এ, একই address space-এ চলে।

```
 MONOLITHIC (Linux)                   MICROKERNEL (MINIX 3, seL4, QNX)
 ┌───────────────────────┐            ┌─────────┐┌─────────┐┌─────────┐
 │  User applications    │            │  Apps   ││File srv ││ Driver  │ user
 └──────────┬────────────┘            └────┬────┘└────┬────┘└────┬────┘ space
 ═══════════╪════════════════          ═════╪══════════╪══════════╪═════
 ┌──────────▼────────────┐            ┌────▼──────────▼──────────▼────┐
 │ Scheduler │ Memory    │            │  ছোট kernel: শুধু IPC,          │ kernel
 │ VFS + FS  │ Network   │            │  scheduling, মৌলিক memory       │ space
 │ Drivers   │ IPC ...   │            └───────────────────────────────┘
 └───────────────────────┘
```

- **সুবিধা:** দ্রুত। এক অংশ আরেক অংশকে সরাসরি function call করে; message পাঠাতে হয় না।
- **অসুবিধা:** একটা driver-এ বাগ থাকলে পুরো kernel crash করতে পারে (একে বলে **kernel panic**)। কোড বড় হয়।

### Microkernel (ক্ষুদ্র kernel)

kernel-এ শুধু একেবারে মৌলিক জিনিস (IPC বা inter-process communication, মৌলিক scheduling, মৌলিক memory) থাকে। driver, filesystem ইত্যাদি user space-এ আলাদা "server" প্রসেস হিসেবে চলে, message পাঠিয়ে কথা বলে।

- **সুবিধা:** একটা driver crash করলে শুধু সেটা restart করলেই হয়; বেশি নিরাপদ, যাচাই করা সহজ।
- **অসুবিধা:** message আদান-প্রদানের কারণে ঐতিহাসিকভাবে ধীর।

### Hybrid kernel

দুটোর মিশ্রণ। Windows NT kernel ও macOS-এর XNU এই শ্রেণিতে পড়ে বলে সাধারণত ধরা হয়।

> 📝 **ইতিহাস:** ১৯৯২ সালে MINIX-এর স্রষ্টা অধ্যাপক Andrew Tanenbaum Usenet-এ লেখেন "LINUX is obsolete", কারণ এটা monolithic। Linus Torvalds জবাব দেন — শুরু হয় বিখ্যাত **Tanenbaum–Torvalds বিতর্ক**। বাস্তবে লিনাক্স monolithic থেকেও ব্যাপক সফল হয়েছে, কারণ নিচের "loadable module" কৌশল দিয়ে সে monolithic ডিজাইনের সবচেয়ে বড় দুর্বলতা — নমনীয়তার অভাব — অনেকটাই কাটিয়ে উঠেছে।

| বৈশিষ্ট্য | Monolithic (Linux) | Microkernel | Hybrid (Windows, macOS) |
|---|---|---|---|
| driver কোথায় চলে | kernel space | user space | মূলত kernel space |
| গতি | খুব দ্রুত | তুলনামূলক ধীর | দ্রুত |
| একটা driver বাগের প্রভাব | পুরো সিস্টেম পড়ে যেতে পারে | শুধু ঐ server | প্রায়ই পুরো সিস্টেম |
| উদাহরণ | Linux, FreeBSD | MINIX 3, seL4, QNX | Windows NT, XNU |

---

## ২.৬ Loadable Kernel Modules (LKM) — দরকারমতো যোগ-বিয়োগ

ভাবো, পৃথিবীর হাজার হাজার রকম hardware-এর সব driver যদি kernel-এর মূল ফাইলে ঢোকানো থাকত, তাহলে kernel হতো বিশাল, আর নতুন একটা Wi-Fi কার্ডের driver যোগ করতে পুরো kernel আবার build করে reboot করতে হতো।

লিনাক্সের সমাধান: **loadable kernel module** (`.ko` ফাইল, "kernel object")। এগুলো kernel-এর টুকরো, যা **চলন্ত অবস্থায়** লোড বা আনলোড করা যায়, reboot ছাড়াই। লোড হওয়ার পর module পুরোপুরি kernel space-এ, kernel-এর অংশ হিসেবেই চলে — তাই লিনাক্স এখনও monolithic, কিন্তু "মডুলার monolithic"।

**উপমা:** monolithic kernel একটা সুইস আর্মি ছুরির মতো। module হলো এমন ব্যবস্থা যেখানে দরকারি ব্লেড আলাদা করে লাগানো-খোলা যায়, কিন্তু লাগানোর পর সেটা ছুরির অংশই।

### Module কোথায় থাকে

```
$ ls /lib/modules/$(uname -r)/kernel/
arch  block  crypto  drivers  fs  kernel  lib  mm  net  samples  sound  ubuntu  virt
```

(সংস্করণ ও distro অনুযায়ী তালিকা একটু ভিন্ন হতে পারে। আধুনিক distro-তে `/lib` আসলে `/usr/lib`-এর symlink।) ফাইলগুলো প্রায়ই compressed থাকে — যেমন `.ko.zst` (Ubuntu 24.04, Fedora) বা `.ko.xz`।

### Module সংক্রান্ত কমান্ড

| কমান্ড | কাজ |
|---|---|
| `lsmod` | এখন লোড হওয়া module-এর তালিকা (আসলে `/proc/modules` সুন্দর করে দেখায়) |
| `modinfo <নাম>` | একটা module-এর বিবরণ: ফাইল, লাইসেন্স, লেখক, parameter, dependency |
| `sudo modprobe <নাম>` | module লোড করে, dependency সহ (প্রস্তাবিত উপায়) |
| `sudo modprobe -r <নাম>` | module আনলোড করে (অব্যবহৃত dependency সহ) |
| `sudo insmod /path/x.ko` | নির্দিষ্ট ফাইল সরাসরি লোড (dependency দেখে না) |
| `sudo rmmod <নাম>` | module সরাসরি আনলোড |
| `sudo depmod -a` | module dependency ডাটাবেস (`modules.dep`) আবার তৈরি করে |

উদাহরণ:

```
$ lsmod | head -5
Module                  Size  Used by
snd_hda_codec_hdmi     94208  1
iwlmvm                 806912  0
mac80211             1720320  1 iwlmvm
iwlwifi               598016  1 iwlmvm

$ modinfo loop | head -6
filename:       /lib/modules/6.8.0-45-generic/kernel/drivers/block/loop.ko.zst
license:        GPL
alias:          devname:loop-control
alias:          char-major-10-237
alias:          block-major-7-*
...
```

`lsmod`-এর কলাম: **Module** = নাম, **Size** = মেমোরিতে আকার (বাইট), **Used by** = কতজন এটা ব্যবহার করছে ও কারা। উপরে দেখো `mac80211` ব্যবহার করছে `iwlmvm` (Intel Wi-Fi driver) — এটাই dependency।

> 💡 **টিপস:** কিছু driver module না হয়ে kernel-এর ভেতরেই "built-in" থাকে। সেগুলো `lsmod`-এ দেখাবে না, কিন্তু `/lib/modules/$(uname -r)/modules.builtin` ফাইলে তালিকা থাকে। তাই "`lsmod`-এ নেই মানে driver নেই" — এই সিদ্ধান্ত সবসময় ঠিক না।

> ⚠️ **সতর্কতা:** ব্যবহৃত module জোর করে আনলোড করা বা অজানা উৎসের `.ko` লোড করা সিস্টেম crash করাতে পারে — কারণ module kernel space-এ পূর্ণ ক্ষমতা নিয়ে চলে। Secure Boot চালু থাকলে অনেক distro unsigned module লোড করতেই দেয় না। module, `/proc`, `/sys` নিয়ে গভীর আলোচনা [অধ্যায় ৪৭](47-kernel-proc-sys.md)-এ।

বুটের সময় স্বয়ংক্রিয়ভাবে module লোড করতে `/etc/modules-load.d/*.conf`, আর কোনো module ব্লক (blacklist) করতে বা option দিতে `/etc/modprobe.d/*.conf` ব্যবহার হয়। hardware সংযোগ হলে সঠিক module বেশিরভাগ ক্ষেত্রে **udev** স্বয়ংক্রিয়ভাবে লোড করে।

---

## ২.৭ Kernel-এর প্রধান কাজগুলো — একটু গভীরে

### ২.৭.১ Process Scheduling — CPU-র সময় ভাগ করা

**Process** (প্রসেস) হলো একটা চলমান প্রোগ্রাম। ডিস্কে থাকা `/usr/bin/ls` ফাইলটা একটা *program*; যখন সেটা চালানো হয় আর মেমোরিতে লোড হয়ে কাজ করতে থাকে, তখন সেটা একটা *process*। একই প্রোগ্রাম থেকে একসাথে অনেক প্রসেস চলতে পারে (যেমন তিনটা আলাদা টার্মিনালে তিনটা bash)।

তোমার কম্পিউটারে হয়তো ৮টা CPU core, কিন্তু চলছে ৩০০টা প্রসেস। সবাই একসাথে চলছে বলে মনে হয় কীভাবে?

**উপমা: দাবার গ্র্যান্ডমাস্টারের simultaneous exhibition।** একজন গ্র্যান্ডমাস্টার ৩০ জনের সাথে একসাথে খেলেন — এক বোর্ডে একটা চাল দিয়ে পরের বোর্ডে যান। প্রত্যেকে ভাবে তিনি তার সাথেই খেলছেন। kernel-এর **scheduler** ঠিক এটাই করে — প্রতিটি প্রসেসকে কয়েক মিলিসেকেন্ড করে CPU দেয়, তারপর পরেরটায় যায়। এত দ্রুত যে মনে হয় সবাই একসাথে চলছে। একে বলে **multitasking**, আর প্রসেস বদলানোকে বলে **context switch**।

```
CPU core 0 এর টাইমলাইন (মিলিসেকেন্ডে):
 |--bash--|--firefox--|--vim--|--firefox--|--sshd--|--bash--| ...
 0        3           7       9           13       14       17
          ▲ context switch: আগের প্রসেসের register অবস্থা সংরক্ষণ,
            পরেরটার অবস্থা ফিরিয়ে আনা
```

লিনাক্স **preemptive** multitasking করে: প্রসেস স্বেচ্ছায় CPU ছাড়ুক বা না ছাড়ুক, kernel timer interrupt দিয়ে তাকে থামিয়ে অন্যকে সুযোগ দিতে পারে। ফলে একটা অসীম loop-এ আটকে থাকা প্রোগ্রাম পুরো সিস্টেম জমিয়ে দিতে পারে না।

লিনাক্সের scheduler-এর ইতিহাস:

| সময়কাল | Scheduler | সংক্ষেপে |
|---|---|---|
| 2.6.0–2.6.22 | O(1) scheduler | অগ্রাধিকার-ভিত্তিক queue |
| 2.6.23 (2007)–6.5 | **CFS** (Completely Fair Scheduler) | প্রতিটি task-এর "virtual runtime" হিসাব করে, যে সবচেয়ে কম CPU পেয়েছে তাকে আগে দেয় (red-black tree ব্যবহার করে) |
| 6.6 (2023) থেকে | **EEVDF** (Earliest Eligible Virtual Deadline First) | CFS-এর জায়গা নিয়েছে; ন্যায্যতার সাথে latency আরও ভালোভাবে সামলায় |

তাই Debian 12 (kernel 6.1) ও RHEL 9 (5.14) CFS ব্যবহার করে, আর Ubuntu 24.04 (kernel 6.8) ও নতুন Fedora EEVDF ব্যবহার করে। সাধারণ ব্যবহারকারীর কাছে পার্থক্য প্রায় অদৃশ্য।

প্রতিটি প্রসেসের একটা **nice value** থাকে (−20 থেকে 19; কম মান = বেশি অগ্রাধিকার)। এছাড়া real-time কাজের জন্য আলাদা নীতি (`SCHED_FIFO`, `SCHED_RR`) আছে। প্রসেস, PID, state, signal, `nice`, `ps`, `top` — এসব বিস্তারিত [অধ্যায় ২৪](24-process-management.md)-এ।

> 📝 **মনে রাখো:** লিনাক্স kernel-এর চোখে process আর thread দুটোই আসলে **task** — একই ডেটা স্ট্রাকচার (`task_struct`)। thread হলো এমন task যারা একে অন্যের সাথে মেমোরি শেয়ার করে। দুটোই তৈরি হয় `clone()` system call দিয়ে, শুধু কোন জিনিস শেয়ার হবে তার flag আলাদা।

### ২.৭.২ Memory Management — RAM-এর হিসাবরক্ষক

**Virtual memory** লিনাক্সের অন্যতম চমৎকার ধারণা।

**উপমা: হোটেলের রুম নম্বর।** একটা চেইন হোটেলে প্রতিটি অতিথিকে বলা হয় "তোমার রুম ১০১ থেকে ১১০"। প্রত্যেক অতিথিই ভাবে সে ১০১–১১০ পেয়েছে। কিন্তু রিসেপশনের খাতায় লেখা আছে — অতিথি A-র "১০১" আসলে ৫ম তলার ৫১২ নম্বর রুম, অতিথি B-র "১০১" আসলে ২য় তলার ২০৩। অতিথিরা কখনো একে অপরের রুমে ঢুকতে পারে না, কারণ তাদের নম্বরগুলো আলাদা আসল রুমে যায়।

ঠিক এভাবে:

- প্রতিটি প্রসেস নিজের **virtual address space** পায়, যেন পুরো মেমোরি তারই।
- **page table** নামের "খাতা" (kernel পরিচালনা করে, MMU পড়ে) বলে দেয় কোন virtual address আসলে কোন physical RAM-এ।
- মেমোরি ভাগ করা হয় **page**-এ, সাধারণত ৪ KB করে (`getconf PAGESIZE` দিয়ে দেখো)।

```
  Process A (virtual)          Page table             Physical RAM
  ┌──────────┐                                    ┌──────────────┐
  │ page 0   │──────────────────┐                 │ frame 0      │
  │ page 1   │─────────┐        └────────────────►│ frame 1  (A0)│
  │ page 2   │──► (এখনো নেই →    │                 │ frame 2      │
  └──────────┘   page fault)    │       ┌────────►│ frame 3  (B0)│
                                └───────┼────────►│ frame 4  (A1)│
  Process B (virtual)                   │         │ frame 5      │
  ┌──────────┐                          │         │ ...          │
  │ page 0   │──────────────────────────┘         └──────────────┘
  └──────────┘                                   + ডিস্কে swap এলাকা
```

একটা প্রসেসের virtual memory সাধারণত এভাবে সাজানো থাকে (ওপরে বেশি address):

```
 উচ্চ address  ┌────────────────────────┐
               │ kernel space (প্রসেস    │ ← user mode থেকে অপ্রবেশযোগ্য
               │ দেখতে পায় না)           │
               ├────────────────────────┤
               │ Stack  ↓ (বাড়ে নিচে)    │ ← ফাংশন call, local variable
               │                        │
               │ Memory-mapped এলাকা     │ ← shared library (libc.so.6), mmap
               │                        │
               │ Heap   ↑ (বাড়ে ওপরে)   │ ← malloc() দিয়ে নেওয়া মেমোরি
               ├────────────────────────┤
               │ BSS / Data             │ ← global variable
               │ Text (কোড)             │ ← প্রোগ্রামের instruction (read-only)
 নিম্ন address  └────────────────────────┘
```

নিজে দেখো — `cat` প্রসেসের নিজের memory map:

```
$ cat /proc/self/maps | head -5
5d6f1a2b0000-5d6f1a2b2000 r--p 00000000 103:02 1835092   /usr/bin/cat
5d6f1a2b2000-5d6f1a2b7000 r-xp 00002000 103:02 1835092   /usr/bin/cat
...
7f3c8e400000-7f3c8e428000 r--p 00000000 103:02 1840190   /usr/lib/x86_64-linux-gnu/libc.so.6
```

(`/proc/self` সবসময় "যে প্রসেস এটা পড়ছে" তাকে নির্দেশ করে। address প্রতিবার আলাদা হবে — নিরাপত্তার জন্য ASLR, Address Space Layout Randomization।)

Memory management-এর গুরুত্বপূর্ণ ধারণাগুলো:

| ধারণা | ব্যাখ্যা |
|---|---|
| **Demand paging** | প্রোগ্রাম চালু হলেই সব মেমোরিতে আনা হয় না; যখন যে page প্রথম দরকার হয় তখন আনা হয় (**page fault** হয়ে) |
| **Copy-on-Write (CoW)** | `fork()`-এর পর parent ও child একই page শেয়ার করে; কেউ লিখতে চাইলে তখনই কপি হয় (নিচে fork অংশে আবার দেখব) |
| **Page cache** | ডিস্ক থেকে পড়া ডেটা RAM-এ রেখে দেওয়া, যাতে দ্বিতীয়বার দ্রুত পাওয়া যায়। `free`-এর `buff/cache` কলাম এটাই |
| **Swap** | RAM ভরে গেলে কম-ব্যবহৃত page ডিস্কে (swap partition/file) সরিয়ে রাখা |
| **OOM killer** | RAM আর swap দুটোই শেষ হলে kernel একটা প্রসেস (সাধারণত সবচেয়ে বেশি মেমোরি খাওয়া) মেরে সিস্টেম বাঁচায় |
| **Shared memory** | একাধিক প্রসেস একই physical page ব্যবহার করে — যেমন `libc.so.6` RAM-এ একবারই থাকে, সব প্রসেস শেয়ার করে |

> 💡 **টিপস:** নতুনরা `free` দেখে ভয় পায়: "আমার ১৬ GB-র মধ্যে free মাত্র ১ GB!" ভয় পেয়ো না — লিনাক্স অব্যবহৃত RAM-কে page cache হিসেবে কাজে লাগায় ("unused RAM is wasted RAM")। আসল সূচক হলো **`available`** কলাম — প্রয়োজনে কতটা মেমোরি নতুন প্রোগ্রামকে দেওয়া যাবে।

### ২.৭.৩ VFS — Virtual File System

লিনাক্স ডজনখানেক ফাইলসিস্টেম বোঝে: ext4, XFS, Btrfs, FAT32/vfat, exFAT, NTFS, NFS (নেটওয়ার্ক), tmpfs (RAM-এ), procfs, sysfs... প্রতিটির ভেতরের ফরম্যাট সম্পূর্ণ আলাদা।

তাহলে `cat` প্রোগ্রাম কীভাবে জানে USB ড্রাইভের FAT ফাইল আর SSD-র ext4 ফাইল কীভাবে পড়তে হয়? উত্তর: **সে জানে না, জানার দরকারও নেই।** `cat` শুধু `open()`, `read()` ডাকে। kernel-এর **VFS (Virtual File System)** স্তর সেই অনুরোধ সঠিক ফাইলসিস্টেম driver-এর কাছে পাঠায়।

**উপমা: সার্বজনীন বিদ্যুৎ অ্যাডাপ্টার।** তোমার ল্যাপটপের চার্জার (application) একটাই প্লাগ চেনে। অ্যাডাপ্টার (VFS) বিভিন্ন দেশের সকেট (ext4, NTFS, NFS) এর সাথে মানিয়ে নেয়।

```
     cat      cp      vim      firefox          ← user space
      │        │       │          │
 ═════╪════════╪═══════╪══════════╪═══════  open/read/write/close (syscall)
      ▼        ▼       ▼          ▼
 ┌────────────────────────────────────────┐
 │      VFS (একই সাধারণ ইন্টারফেস)          │ ← kernel
 │  objects: superblock, inode, dentry,    │
 │           file                          │
 └──┬───────┬────────┬────────┬────────┬──┘
    ▼       ▼        ▼        ▼        ▼
  ext4     XFS     vfat      NFS     procfs/sysfs/tmpfs
    │       │        │        │        │
    ▼       ▼        ▼        ▼        └─► kernel-এর ভেতরের ডেটা / RAM
  ┌─────────────────────┐   network
  │ Page cache          │   driver
  │ Block layer + I/O   │
  │ scheduler           │
  │ Disk driver (NVMe)  │
  └─────────────────────┘
```

VFS-এর চারটা প্রধান object:

| Object | কী প্রতিনিধিত্ব করে |
|---|---|
| **superblock** | একটা মাউন্ট করা পুরো ফাইলসিস্টেম (তার আকার, ধরন ইত্যাদি) |
| **inode** | একটা ফাইলের metadata — মালিক, পারমিশন, আকার, ডেটা কোথায় (নাম বাদে!) — দেখো [অধ্যায় ১১](11-links-inode.md) |
| **dentry** (directory entry) | একটা নাম ↔ inode সংযোগ; path খোঁজা দ্রুত করতে cache হয় |
| **file** | একটা প্রসেসের খোলা ফাইল — কোথায় পর্যন্ত পড়া হয়েছে (offset), কোন mode-এ খোলা |

এই VFS-ই "everything is a file" দর্শনকে বাস্তব করে (নিচে ২.১০ দেখো)। ফাইলসিস্টেম হায়ারার্কি [অধ্যায় ৭](07-filesystem-hierarchy.md)-এ, আর ডিস্ক-পার্টিশন-মাউন্ট [অধ্যায় ৩২](32-disk-partition-mount.md)-এ।

### ২.৭.৪ Device Drivers — hardware-এর দোভাষী

প্রতিটি hardware-এর নিজস্ব "ভাষা" আছে (নির্দিষ্ট register, command, protocol)। **device driver** হলো kernel-এর সেই কোড, যেটা ঐ ভাষা জানে আর kernel-এর বাকি অংশকে একটা সাধারণ ইন্টারফেস দেয়। লিনাক্স kernel-এর সোর্স কোডের সবচেয়ে বড় অংশই হলো driver (`drivers/` ডিরেক্টরি)।

User space থেকে device-গুলো দেখা যায় `/dev` ডিরেক্টরিতে **device file** হিসেবে। দুই প্রধান ধরন:

| ধরন | `ls -l`-এ প্রথম অক্ষর | বৈশিষ্ট্য | উদাহরণ |
|---|---|---|---|
| **Character device** | `c` | ডেটা বাইট-স্রোত হিসেবে, ক্রমানুসারে | `/dev/tty`, `/dev/null`, `/dev/random`, সিরিয়াল পোর্ট |
| **Block device** | `b` | নির্দিষ্ট আকারের block-এ, যেকোনো জায়গা থেকে পড়া-লেখা যায় | `/dev/sda`, `/dev/nvme0n1`, `/dev/loop0` |

```
$ ls -l /dev/sda /dev/null /dev/tty
brw-rw---- 1 root disk 8, 0 Nov 12 09:10 /dev/sda
crw-rw-rw- 1 root root 1, 3 Nov 12 09:10 /dev/null
crw-rw-rw- 1 root tty  5, 0 Nov 12 09:15 /dev/tty
```

সাধারণ ফাইলের আকারের জায়গায় এখানে দুটো সংখ্যা: **major number** (কোন driver — যেমন 8 = SCSI/SATA disk driver) আর **minor number** (সেই driver-এর কোন device — 0 = প্রথম ডিস্ক)। kernel এই সংখ্যা দেখেই বোঝে অনুরোধ কোন driver-এর কাছে যাবে।

আধুনিক লিনাক্সে `/dev` একটা `devtmpfs` ফাইলসিস্টেম, যেখানে kernel নিজেই device node তৈরি করে, আর **udev** (systemd-udevd) daemon নিয়ম অনুযায়ী পারমিশন ঠিক করে, বন্ধুসুলভ symlink বানায় (যেমন `/dev/disk/by-uuid/...`), প্রয়োজনীয় module লোড করে। তুমি pendrive লাগালে কী ঘটে:

```
 USB লাগালে → hardware interrupt → kernel-এর USB driver device চিনল
   → kernel একটা "uevent" পাঠাল → udev ধরল → module লোড, /dev/sdb তৈরি
   → desktop (udisks) জানতে পেল → ফাইল ম্যানেজারে আইকন হাজির
```

hardware ও driver দেখার কয়েকটা কমান্ড:

```
$ lspci -k | grep -A 2 -i network
02:00.0 Network controller: Intel Corporation Wi-Fi 6 AX201
        Subsystem: Intel Corporation Device 0070
        Kernel driver in use: iwlwifi

$ lsusb
Bus 001 Device 003: ID 046d:c52b Logitech, Inc. Unifying Receiver
...
```

`lspci -k`-এর "Kernel driver in use" লাইনটা বলে দেয় কোন driver ঐ device চালাচ্ছে — Wi-Fi কাজ না করলে troubleshooting-এর প্রথম ধাপ। (`lspci` ও `lsusb` যথাক্রমে `pciutils` ও `usbutils` প্যাকেজে থাকে, minimal সার্ভারে আলাদা ইনস্টল লাগতে পারে।)

### ২.৭.৫ Networking — Network Stack

kernel-এর ভেতরে একটা সম্পূর্ণ **TCP/IP network stack** আছে। যখন Firefox একটা ওয়েবসাইট খোলে:

```
 Firefox:  socket() → connect() → send() / recv()       (user space)
 ═══════════════════════════════════════════════════
 Socket layer                                           (kernel)
   ↓
 TCP (নির্ভরযোগ্যতা, ক্রম, পুনঃপ্রেরণ)  / UDP
   ↓
 IP (ঠিকানা, routing — কোন পথে যাবে)
   ↓
 Netfilter (firewall নিয়ম: nftables/iptables) — পথে পথে পরীক্ষা
   ↓
 Network device layer + driver (e1000e, iwlwifi ...)
 ═══════════════════════════════════════════════════
 NIC (network card) → তার/বাতাস → ইন্টারনেট             (hardware)
```

লক্ষ করো: TCP, IP, firewall — সবকিছু kernel-এর ভেতরে। Firefox শুধু "socket"-এ লেখে-পড়ে, ঠিক যেমন ফাইলে লেখে-পড়ে। socket-ও একটা file descriptor! নেটওয়ার্কের তত্ত্ব [অধ্যায় ৩৬](36-networking-basics.md)-এ, কমান্ড [অধ্যায় ৩৭](37-network-commands.md)-এ, আর firewall [অধ্যায় ৩৯](39-firewall.md)-এ।

### ২.৭.৬ অন্যান্য গুরুত্বপূর্ণ kernel উপব্যবস্থা

সংক্ষেপে আরও কিছু, যেগুলো পরের অধ্যায়ে কাজে লাগবে:

- **IPC (Inter-Process Communication):** প্রসেসদের মধ্যে কথা বলার উপায় — pipe, signal, Unix domain socket, shared memory, message queue।
- **Security:** user/group ভিত্তিক পারমিশন, **capabilities** (root-এর ক্ষমতাকে ছোট ছোট টুকরোয় ভাগ করা), **LSM** (Linux Security Modules) — SELinux (Fedora/RHEL-এ ডিফল্ট), AppArmor (Ubuntu/Debian-এ ডিফল্ট)। দেখো [অধ্যায় ৪৫](45-security-hardening.md)।
- **namespaces ও cgroups:** প্রসেসকে আলাদা "জগৎ" দেখানো আর তার resource সীমিত করা — এই দুটো দিয়েই Docker/Podman container তৈরি হয়। দেখো [অধ্যায় ৪৮](48-containers-virtualization.md)।
- **Timekeeping:** সময় রাখা, timer চালানো।

---

## ২.৮ System Call — user space থেকে kernel-এর দরজা

### System call কী?

**System call** (syscall) হলো kernel-এর কাছে একটা প্রসেসের আনুষ্ঠানিক অনুরোধ — "আমার হয়ে এই কাজটা করে দাও"। user space-এর প্রোগ্রাম hardware বা অন্যের ডেটায় হাত দিতে পারে না, তাই যা-ই করতে চাক — ফাইল খোলা, স্ক্রিনে লেখা, নেটওয়ার্কে পাঠানো, নতুন প্রসেস তৈরি, এমনকি নিজেকে বন্ধ করা — সব system call দিয়ে।

**উপমা: ব্যাংকের কাউন্টার।** তুমি ব্যাংকের ভল্টে ঢুকে নিজে টাকা তুলতে পারো না। তুমি কাউন্টারে একটা নির্দিষ্ট ফর্ম (system call নম্বর + তথ্য) জমা দাও। ক্যাশিয়ার (kernel) যাচাই করে তোমার অ্যাকাউন্টে টাকা আছে কিনা (পারমিশন), তারপর ভল্ট থেকে এনে দেয় — অথবা "অনুমতি নেই" বলে ফিরিয়ে দেয় (error code)।

### গুরুত্বপূর্ণ কিছু system call

| ক্যাটাগরি | System call | কাজ |
|---|---|---|
| ফাইল | `openat`, `read`, `write`, `close` | ফাইল খোলা, পড়া, লেখা, বন্ধ |
| ফাইল তথ্য | `stat`/`newfstatat`/`statx`, `lseek` | metadata জানা, অবস্থান সরানো |
| ডিরেক্টরি | `getdents64`, `mkdir`, `unlink`, `rename` | ডিরেক্টরির তালিকা, তৈরি, মোছা |
| প্রসেস | `clone` (`fork`), `execve`, `exit_group`, `wait4` | প্রসেস তৈরি, প্রোগ্রাম চালানো, শেষ, অপেক্ষা |
| মেমোরি | `mmap`, `munmap`, `brk`, `mprotect` | মেমোরি নেওয়া-ছাড়া-সুরক্ষা |
| নেটওয়ার্ক | `socket`, `connect`, `bind`, `listen`, `accept`, `sendto`, `recvfrom` | নেটওয়ার্ক যোগাযোগ |
| signal | `kill`, `rt_sigaction` | signal পাঠানো ও handler বসানো |
| অন্যান্য | `ioctl`, `getpid`, `uname` | device নিয়ন্ত্রণ, তথ্য |

x86_64 লিনাক্সে system call-এর সংখ্যা ৩০০-এর বেশি (প্রতিটির একটা নম্বর আছে; যেমন x86_64-এ `read` = 0, `write` = 1)। পূর্ণ তালিকা: `man 2 syscalls`।

> 💡 **টিপস:** `man` পেজের section নম্বর মনে রাখো: `man 1` = সাধারণ কমান্ড, `man 2` = system call, `man 3` = library ফাংশন। তাই `man 2 write` system call-টা দেখায়, আর `man 1 write` দেখায় অন্য user-কে মেসেজ পাঠানোর `write` কমান্ড! (`man`-এর সব section [অধ্যায় ৫](05-terminal-o-shell.md)-এ।)

### ভেতরে কীভাবে কাজ করে

ধরো একটা C প্রোগ্রাম `printf("Hello\n")` লিখল। ধাপগুলো (x86_64):

```
 ① printf("Hello\n")             ← তোমার কোড (user space)
        │
 ② glibc: লেখাটা format করে, buffer-এ রাখে, পরে write() wrapper ডাকে
        │
 ③ glibc-র write() wrapper:
        rax = 1        (syscall নম্বর: write)
        rdi = 1        (fd 1 = stdout)
        rsi = buffer-এর address
        rdx = 6        (কত বাইট)
        `syscall` instruction  ─────────┐
 ═════════════════════════════════════│════  ring 3 → ring 0
        ┌─────────────────────────────┘
 ④ kernel entry: register সংরক্ষণ, rax দেখে syscall table থেকে
    sys_write খুঁজে বের করে
        │
 ⑤ sys_write: fd 1 কোন "file"? → একটা terminal (tty) → tty driver
    → টার্মিনাল এমুলেটর → স্ক্রিনে "Hello"
        │
 ⑥ ফলাফল (লেখা বাইট সংখ্যা = 6, বা ঋণাত্মক error) rax-এ রেখে
    `sysret` দিয়ে ফেরত                 ring 0 → ring 3
        │
 ⑦ glibc: ফলাফল ঋণাত্মক হলে errno সেট করে −1 ফেরত দেয়
```

ARM64-এ একই কাজ `svc #0` instruction দিয়ে হয়, syscall নম্বর থাকে `x8` register-এ।

**errno:** system call ব্যর্থ হলে একটা error code দেয় — যেমন `ENOENT` (No such file or directory), `EACCES` (Permission denied), `EEXIST` (File exists)। টার্মিনালে যে বাংলা-ইংরেজি error মেসেজ দেখো, সেগুলোর বেশিরভাগই এই errno থেকে আসে।

### vDSO — syscall ছাড়া syscall

কিছু অনুরোধ এত ঘন ঘন হয় (যেমন "এখন কয়টা বাজে?" — `clock_gettime`, `gettimeofday`) যে প্রতিবার ring বদলানো অপচয়। তাই kernel প্রতিটি প্রসেসের মেমোরিতে একটা ছোট shared library বসিয়ে দেয় — **vDSO** (virtual dynamic shared object)। এই ফাংশনগুলো user space-এ বসেই উত্তর দেয়। `ldd`-এর output-এ `linux-vdso.so.1` দেখলে বুঝবে এটাই সেই জিনিস।

### strace — system call চোখে দেখা

`strace` একটা অসাধারণ টুল। এটা একটা প্রোগ্রাম চালায় আর সে যত system call করে, সব একটা একটা করে দেখায় — প্রোগ্রাম আর kernel-এর মধ্যকার কথোপকথনের রেকর্ডিং! (ইনস্টল: `sudo apt install strace` বা `sudo dnf install strace`।)

**Syntax:**

```
strace [অপশন] কমান্ড [আর্গুমেন্ট...]
strace [অপশন] -p PID
```

| অপশন | কাজ |
|---|---|
| `-c` | প্রতিটি call-এর বদলে শেষে সারসংক্ষেপ টেবিল (কোন syscall কতবার, কত সময়) |
| `-e trace=openat,read` | শুধু নির্দিষ্ট syscall দেখাও |
| `-e trace=%file` | ফাইল-নাম নেয় এমন সব syscall (এছাড়া `%process`, `%network`, `%memory` ইত্যাদি) |
| `-f` | child প্রসেস/thread-ও অনুসরণ করো |
| `-p PID` | আগে থেকে চলা প্রসেসে যুক্ত হও |
| `-o ফাইল` | output ফাইলে লেখো |
| `-t` / `-tt` | প্রতিটি লাইনে সময় / মাইক্রোসেকেন্ড সহ সময় |
| `-T` | প্রতিটি syscall-এ কত সময় লেগেছে |
| `-s N` | string-এর সর্বোচ্চ N অক্ষর দেখাও (ডিফল্ট 32) |
| `-y` | file descriptor-এর পাশে তার path দেখাও |

**উদাহরণ ১ — `echo` কীভাবে লেখে:**

```
$ strace -e trace=write /bin/echo হ্যালো
write(1, "\340\246\271\340\247\215\340\246\257\340\246\276\340\246\262\340\247\213\n", 19হ্যালো
) = 19
+++ exited with 0 +++
```

দেখো: `echo` আসলে `write(1, ..., 19)` system call করেছে — fd 1 (stdout)-এ ১৯ বাইট (বাংলা অক্ষর UTF-8-এ প্রতিটি ৩ বাইট, ৬টা code point = ১৮ বাইট + newline)। strace ঐ বাইটগুলো octal escape হিসেবে দেখায়, আর আসল লেখাটা একই লাইনে স্ক্রিনে চলে আসে, কারণ strace নিজের output stderr-এ দেয়।

**উদাহরণ ২ — একটা পুরো প্রোগ্রামের কথোপকথন (সংক্ষিপ্ত):**

```
$ strace cat /etc/hostname
execve("/usr/bin/cat", ["cat", "/etc/hostname"], 0x7ffd5c8e1a38 /* 31 vars */) = 0
brk(NULL)                               = 0x5612d4e3a000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f1b2c6f4000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
...
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0"..., 832) = 832
...
openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=5, ...}) = 0
read(3, "mypc\n", 131072)               = 5
write(1, "mypc\n", 5mypc
)                   = 5
read(3, "", 131072)                     = 0
close(3)                                = 0
close(1)                                = 0
close(2)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

এটা পড়তে শেখো, অনেক রহস্য পরিষ্কার হবে:

1. `execve(...)` — প্রোগ্রাম `/usr/bin/cat` লোড হলো।
2. `brk`, `mmap` — মেমোরি নেওয়া হচ্ছে।
3. `access("/etc/ld.so.preload")` → `ENOENT` — dynamic loader খুঁজল কোনো preload library আছে কিনা; নেই, স্বাভাবিক।
4. `openat("/etc/ld.so.cache")`, `openat(".../libc.so.6")` — dynamic loader glibc খুঁজে লোড করছে। `\177ELF` হলো লিনাক্স executable ফাইলের "magic number" (ELF = Executable and Linkable Format)।
5. `openat("/etc/hostname") = 3` — আসল কাজ শুরু। ফাইল খুলে **fd 3** পাওয়া গেল (0, 1, 2 আগে থেকেই stdin/stdout/stderr)।
6. `read(3, ...) = 5` — ৫ বাইট পড়ল; `write(1, ...) = 5` — stdout-এ ৫ বাইট লিখল।
7. `read(3, "", ...) = 0` — 0 মানে ফাইল শেষ (EOF)।
8. `close`, `exit_group(0)` — সব বন্ধ, exit status 0 (সফল)।

(তোমার সিস্টেমে লাইবেরির path, address, buffer আকার একটু আলাদা হতে পারে; Fedora/RHEL-এ libc থাকে `/lib64/libc.so.6`-এ।)

**উদাহরণ ৩ — সারসংক্ষেপ (`-c`):**

```
$ strace -c ls > /dev/null
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 23.41    0.000110           5        21           mmap
 14.04    0.000066           7         9           openat
 ...
  2.13    0.000010           5         2         2 statfs
------ ----------- ----------- --------- --------- ----------------
100.00    0.000470           3       127        10 total
```

একটা সাধারণ `ls`-ও ১০০-র বেশি system call করে! (সংখ্যা সিস্টেম অনুযায়ী আলাদা হবে।)

**বাস্তব troubleshooting কৌশল:** একটা প্রোগ্রাম বলছে "config file not found" কিন্তু কোন path-এ খুঁজছে বলছে না? —

```
$ strace -f -e trace=%file -o trace.txt ./myapp
$ grep ENOENT trace.txt
openat(AT_FDCWD, "/etc/myapp/config.yml", O_RDONLY) = -1 ENOENT (No such file or directory)
```

এখন তুমি জানো ঠিক কোন ফাইলটা দরকার। এটা লিনাক্স এক্সপার্টদের প্রিয় কৌশল।

> ⚠️ **সতর্কতা:** `strace` প্রোগ্রামকে অনেক ধীর করে দেয় — ব্যস্ত production সার্ভারে সাবধানে ব্যবহার করো। অন্য user-এর প্রসেসে `-p` দিয়ে যুক্ত হতে root লাগবে; Ubuntu-তে Yama নিরাপত্তা নিয়মের (`/proc/sys/kernel/yama/ptrace_scope` = 1) কারণে নিজের চলমান প্রসেসে attach করতেও সাধারণত `sudo` লাগে।

> 💡 **টিপস:** `ltrace` নামে একটা সহোদর টুল আছে, যেটা system call না, **library call** (যেমন `printf`, `malloc`) দেখায়। সব distro-তে ডিফল্ট ইনস্টল থাকে না।

> 📝 **মনে রাখো:** উদাহরণ ১-এ `echo` না লিখে `/bin/echo` লিখেছি। কারণ `echo` bash-এর একটা **builtin** — bash নিজেই সেটা চালায়, আলাদা প্রোগ্রাম চালায় না। `strace` শুধু আলাদা প্রোগ্রাম চালাতে পারে, তাই বাইরের `/bin/echo` ফাইলটা দিয়েছি। builtin বনাম বাইরের কমান্ড নিয়ে নিচে ২.১২-এ আরও বলব।

---

## ২.৯ Libraries — glibc ও বন্ধুরা

### Library কী?

**Library** হলো আগে থেকে লেখা, বারবার ব্যবহারযোগ্য ফাংশনের সংগ্রহ। প্রতিটি প্রোগ্রামারকে যদি স্ক্রিনে লেখা, স্ট্রিং তুলনা, মেমোরি নেওয়ার কোড নিজে লিখতে হতো — আর সেটাও প্রতিটি CPU-র syscall নিয়ম মেনে assembly-তে — জীবন দুর্বিষহ হতো।

লিনাক্সে সবচেয়ে গুরুত্বপূর্ণ library হলো **C standard library**। বেশিরভাগ distro-তে এটা **glibc** (GNU C Library), ফাইলের নাম `libc.so.6`। এটা দুটো কাজ করে:

1. **system call wrapper:** `open()`, `read()`, `fork()` ইত্যাদি — register-এ মান বসিয়ে `syscall` instruction চালানোর ঝামেলা লুকিয়ে রাখে।
2. **উচ্চস্তরের সুবিধা:** `printf()` (format করা + buffer করা), `malloc()` (ছোট ছোট মেমোরি টুকরো হিসাব রাখা; ভেতরে `brk`/`mmap` ব্যবহার করে), `strlen()`, `qsort()` — এগুলোর অনেকগুলো কোনো system call-ই করে না।

```
  তোমার প্রোগ্রাম:   printf()   malloc()   strlen()   fopen()
                       │          │          │          │
  glibc:          format+buffer  heap      সাধারণ     FILE* তৈরি
                       │       manager    লুপ (কোনো      │
                       ▼          ▼       syscall নেই)   ▼
  syscall:          write()   brk()/mmap()            openat()
```

অন্যান্য C library-ও আছে: **musl** (ছোট, Alpine Linux-এ ব্যবহার হয় — Docker image-এ জনপ্রিয়), Android-এ **Bionic**।

> 💡 **টিপস:** Python, Java, Node.js, Go — এরা কোথায়? Python ও Node-এর interpreter নিজে C/C++-এ লেখা, glibc ব্যবহার করে। Go একটু ব্যতিক্রম — Go-র runtime অনেক সময় glibc বাদ দিয়ে নিজেই সরাসরি syscall করে। কিন্তু যে ভাষাই হোক, শেষ পর্যন্ত kernel-এর দরজা একটাই: **system call**।

### Static বনাম Dynamic linking

| | Static linking | Dynamic linking |
|---|---|---|
| library কোথায় | প্রোগ্রামের ফাইলের ভেতরেই কপি করা | আলাদা `.so` ফাইলে, চালানোর সময় লোড |
| ফাইলের আকার | বড় | ছোট |
| RAM ব্যবহার | প্রতিটি প্রোগ্রামে আলাদা কপি | সব প্রোগ্রাম এক কপি শেয়ার করে |
| নিরাপত্তা আপডেট | প্রতিটি প্রোগ্রাম আবার build করতে হয় | শুধু library আপডেট করলেই সবাই পায় |
| এক্সটেনশন | `.a` (archive) | `.so` (shared object) |

লিনাক্সের বেশিরভাগ প্রোগ্রাম **dynamic**। চালু হওয়ার সময় **dynamic linker/loader** (`ld-linux-x86-64.so.2`, অর্থাৎ `ld.so`) প্রয়োজনীয় `.so` ফাইল খুঁজে মেমোরিতে map করে। খোঁজে `/etc/ld.so.cache` (যেটা `ldconfig` কমান্ড তৈরি করে), `LD_LIBRARY_PATH` environment variable ইত্যাদি থেকে।

### `ldd` — একটা প্রোগ্রাম কোন কোন library চায়

```
$ ldd /usr/bin/ls
        linux-vdso.so.1 (0x00007ffd3a7e6000)
        libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1 (0x00007f6b1c8a0000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f6b1c600000)
        libpcre2-8.so.0 => /lib/x86_64-linux-gnu/libpcre2-8.so.0 (0x00007f6b1c560000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f6b1c900000)
```

- `linux-vdso.so.1` — উপরে বলা vDSO; কোনো ফাইল নেই, kernel মেমোরিতে বসিয়ে দেয়।
- `libc.so.6` — glibc।
- `/lib64/ld-linux-x86-64.so.2` — dynamic loader নিজেই।

glibc-র ভার্সন দেখতে:

```
$ ldd --version | head -1
ldd (Ubuntu GLIBC 2.39-0ubuntu8.3) 2.39
```

(Ubuntu 24.04-এ glibc 2.39, Debian 12-এ 2.36, RHEL 9-এ 2.34।)

> ⚠️ **সতর্কতা:** অবিশ্বস্ত বা অজানা প্রোগ্রামে `ldd` চালাবে না। কিছু ক্ষেত্রে `ldd` প্রোগ্রামটাকে আসলে আংশিক চালিয়ে ফেলতে পারে (man পেজেও সতর্ক করা আছে)। অজানা ফাইলের জন্য `objdump -p ফাইল | grep NEEDED` নিরাপদ।

> ⚠️ **সতর্কতা:** কখনো glibc হাত দিয়ে মুছবে না বা অন্য ভার্সন দিয়ে জোর করে বদলাবে না। প্রায় সব প্রোগ্রাম — `ls`, `bash`, এমনকি `sudo` — এর ওপর নির্ভরশীল। glibc ভাঙলে সিস্টেম কার্যত অচল হয়ে যায়, রিকভারির জন্য live USB লাগে।

library কম্পাইল ও লিংক করার ব্যবহারিক দিক [অধ্যায় ৩১](31-compile-from-source.md)-এ।

---

## ২.১০ Shell — তোমার আর সিস্টেমের মাঝে দোভাষী

**Shell** (খোলস) হলো একটা সাধারণ user-space প্রোগ্রাম, যেটা তোমার লেখা কমান্ড পড়ে, ব্যাখ্যা করে, আর প্রয়োজনীয় প্রোগ্রাম চালায়। নাম "shell" কারণ এটা kernel-কে ঘিরে থাকা বাইরের খোলস — তুমি খোলসের সাথে কথা বলো, খোলস ভেতরের kernel-এর সাথে।

গুরুত্বপূর্ণ কথা: **shell-এর কোনো বিশেষ ক্ষমতা নেই।** সে Firefox বা vim-এর মতোই একটা সাধারণ প্রোগ্রাম, শুধু তার কাজ অন্য প্রোগ্রাম চালানো। তাই অনেক রকম shell আছে, আর তুমি চাইলে বদলাতে পারো:

| Shell | বৈশিষ্ট্য |
|---|---|
| `bash` | Bourne Again Shell; Ubuntu, Debian, Fedora, RHEL-এ user-দের ডিফল্ট |
| `sh` | POSIX shell; Debian/Ubuntu-তে `/bin/sh` আসলে `dash` (ছোট, দ্রুত), Fedora/RHEL-এ `bash` |
| `zsh` | অনেক ফিচার; macOS-এর ডিফল্ট; Oh My Zsh জনপ্রিয় |
| `fish` | নতুনদের জন্য বন্ধুসুলভ, স্বয়ংক্রিয় সাজেশন |

```
$ echo $SHELL
/bin/bash
$ cat /etc/shells
/bin/sh
/bin/bash
/usr/bin/bash
/bin/dash
...
```

**Terminal বনাম Shell:** terminal (যেমন GNOME Terminal, Konsole) হলো জানালা — কীবোর্ড ইনপুট নেয়, লেখা দেখায়। shell হলো তার ভেতরে চলা প্রোগ্রাম, যে কমান্ড বোঝে। দুটোর মাঝে kernel-এর একটা **pseudo-terminal (pty)** থাকে। বিস্তারিত [অধ্যায় ৫](05-terminal-o-shell.md)-এ।

```
 তুমি → [Terminal emulator] ⇄ /dev/pts/0 (kernel-এর pty + tty driver) ⇄ [bash] → প্রোগ্রাম
          (GUI জানালা)                                                (কমান্ড ব্যাখ্যা)
```

---

## ২.১১ "Everything is a File" — লিনাক্সের দর্শন

Unix (এবং তার উত্তরসূরি লিনাক্স)-এর সবচেয়ে বিখ্যাত নীতি: **"সবকিছুই ফাইল"**। আরও সঠিকভাবে বললে — **"সবকিছুই file descriptor"**: যতটা সম্ভব সবকিছুকে একই ইন্টারফেস (`open`, `read`, `write`, `close`) দিয়ে ব্যবহারযোগ্য করা।

**উপমা: সার্বজনীন রিমোট কন্ট্রোল।** একটা রিমোটের চারটা বোতাম (open/read/write/close) দিয়ে তুমি টিভি, এসি, ফ্যান, লাইট — সব চালাতে পারো। প্রতিটি যন্ত্রের জন্য আলাদা রিমোট শিখতে হয় না।

এর ফলে কী সুবিধা? `cat`, `grep`, `echo`, `>` রিডাইরেকশন — যে টুল সাধারণ ফাইলে কাজ করে, সেগুলো hardware, kernel-এর তথ্য, প্রসেস — সবকিছুতেই কাজ করে!

| "ফাইল" | আসলে কী | উদাহরণ ব্যবহার |
|---|---|---|
| `/home/rahim/note.txt` | সাধারণ ফাইল | `cat note.txt` |
| `/dev/sda` | পুরো হার্ডডিস্ক | `sudo dd if=/dev/sda ...` (সাবধান!) |
| `/dev/null` | "কালো গর্ত" — যা লেখো সব হারিয়ে যায়, পড়লে খালি | `command > /dev/null` |
| `/dev/zero` | অসীম শূন্য বাইটের উৎস | ফাঁকা ফাইল বানাতে |
| `/dev/urandom` | অসীম random বাইট | `head -c 16 /dev/urandom \| base64` |
| `/dev/tty` | তোমার বর্তমান টার্মিনাল | `echo hi > /dev/tty` |
| `/proc/cpuinfo` | CPU-র তথ্য (kernel তৎক্ষণাৎ তৈরি করে) | `cat /proc/cpuinfo` |
| `/proc/<PID>/` | একটা প্রসেসের সব তথ্য | `ls /proc/1/` |
| `/sys/class/...` | device ও driver-এর তথ্য ও সেটিং | ব্যাটারি, ব্রাইটনেস |
| pipe (`\|`) | দুই প্রসেসের মাঝে ডেটা চ্যানেল | `ls \| grep txt` |
| socket | নেটওয়ার্ক বা লোকাল সংযোগ | `/run/...sock` |

নিজে চেষ্টা করো:

```
$ cat /proc/version
Linux version 6.8.0-45-generic (buildd@lcy02-amd64-075) (x86_64-linux-gnu-gcc-13 ...) #45-Ubuntu SMP PREEMPT_DYNAMIC ...

$ cat /proc/uptime
12345.67 98765.43

$ cat /sys/class/power_supply/BAT0/capacity
87
```

(`BAT0` শুধু ল্যাপটপে থাকবে, নাম `BAT1` বা অন্যরকমও হতে পারে।) `/proc/uptime`-এর প্রথম সংখ্যা = সিস্টেম কত সেকেন্ড চালু, দ্বিতীয় = সব CPU মিলিয়ে কত সেকেন্ড অলস ছিল।

মজার ব্যাপার: `/proc` আর `/sys`-এর ফাইলগুলো **ডিস্কে নেই**! এগুলো **virtual filesystem** (procfs, sysfs)। তুমি `cat` করলে kernel সেই মুহূর্তে উত্তরটা তৈরি করে দেয়। এজন্য `ls -l` এগুলোর আকার প্রায়ই 0 দেখায়, অথচ `cat` করলে লেখা আসে।

### File Descriptor — খোলা ফাইলের টোকেন

প্রসেস যখন একটা ফাইল খোলে, kernel তাকে একটা ছোট পূর্ণসংখ্যা দেয় — **file descriptor (fd)**। উপমা: দর্জির দোকানে কাপড় জমা দিলে একটা টোকেন নম্বর দেয়; পরে শুধু টোকেন দেখালেই চলে। প্রতিটি প্রসেস তিনটা fd নিয়ে জন্মায়:

| fd | নাম | ডিফল্ট সংযোগ |
|---|---|---|
| 0 | stdin (standard input) | কীবোর্ড (টার্মিনাল) |
| 1 | stdout (standard output) | স্ক্রিন (টার্মিনাল) |
| 2 | stderr (standard error) | স্ক্রিন (টার্মিনাল) |

```
$ ls -l /proc/$$/fd
lrwx------ 1 rahim rahim 64 Nov 12 10:00 0 -> /dev/pts/0
lrwx------ 1 rahim rahim 64 Nov 12 10:00 1 -> /dev/pts/0
lrwx------ 1 rahim rahim 64 Nov 12 10:00 2 -> /dev/pts/0
lrwx------ 1 rahim rahim 64 Nov 12 10:00 255 -> /dev/pts/0
```

(`$$` = বর্তমান shell-এর PID। bash নিজের ব্যবহারের জন্য fd 255 খোলা রাখে।) তিনটাই একই টার্মিনাল `/dev/pts/0`-এর দিকে নির্দেশ করছে। রিডাইরেকশন (`>`, `<`, `2>`) আর পাইপ মানে আসলে এই fd-গুলোকে অন্য কোথাও জুড়ে দেওয়া — [অধ্যায় ১৯](19-redirection-pipe.md)-এ পুরোটা।

> 📝 **মনে রাখো:** "everything is a file" একটা দর্শন, পরম সত্য না। যেমন নেটওয়ার্ক ইন্টারফেস (`eth0`, `wlan0`) `/dev`-এ কোনো ফাইল হিসেবে থাকে না (যদিও `/sys/class/net/`-এ তাদের তথ্য আছে), আর socket তৈরি হয় `socket()` দিয়ে, `open()` দিয়ে না। তবে একবার তৈরি হলে socket-ও একটা fd, তাতে `read`/`write` চলে।

---

## ২.১২ একটা কমান্ড চালালে ভেতরে কী হয় — fork, exec, wait

এখন সব জ্ঞান একসাথে করি। তুমি টার্মিনালে লিখলে:

```
$ ls -l /home
```

আর Enter চাপলে। এক সেকেন্ডের ভগ্নাংশে যা ঘটে:

### ধাপ ১: কীবোর্ড থেকে shell পর্যন্ত

তুমি প্রতিটি কী চাপলে → কীবোর্ড interrupt (বা GUI-তে display server-এর মাধ্যমে terminal emulator ঘটনা পায়) → terminal emulator অক্ষরগুলো pty-তে লেখে → bash `read()` system call দিয়ে সেগুলো পড়ে। (bash লাইন এডিটিং নিজে করে, readline library দিয়ে — তাই Tab completion, Ctrl+A ইত্যাদি কাজ করে; দেখো [অধ্যায় ৬](06-terminal-shortcuts.md)।) Enter চাপলে bash পুরো লাইনটা হাতে পায়।

### ধাপ ২: Parsing — লাইনটা টুকরো করা

bash লাইনটা শব্দে ভাঙে: `ls`, `-l`, `/home`। তারপর expansion করে: variable (`$HOME`), wildcard (`*.txt`), `~`, ইত্যাদি। কোনো alias থাকলে বদলায় (Ubuntu-তে `ls` আসলে প্রায়ই `ls --color=auto`-এর alias)।

### ধাপ ৩: কমান্ড কোথায়? — খোঁজার ক্রম

bash এই ক্রমে খোঁজে:

1. **alias** আছে কি?
2. **keyword** (`if`, `for`...) বা **function**?
3. **builtin** — bash নিজেই জানে এমন কমান্ড (`cd`, `echo`, `pwd`, `export`, `exit`)? থাকলে bash নিজেই চালায়, **কোনো নতুন প্রসেস হয় না**।
4. নাহলে বাইরের প্রোগ্রাম — **`$PATH`** variable-এর ডিরেক্টরিগুলোতে বাঁ থেকে ডানে খোঁজে। (খুঁজে পেলে মনে রাখে — `hash` টেবিলে।)

```
$ type ls cd
ls is aliased to `ls --color=auto'
cd is a shell builtin

$ type -a ls
ls is aliased to `ls --color=auto'
ls is /usr/bin/ls
ls is /bin/ls

$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

(`/bin` আধুনিক distro-তে `/usr/bin`-এর symlink, তাই একই ফাইল দুইবার দেখাচ্ছে।)

> 💡 **টিপস:** `cd` কেন builtin হতেই হবে? কারণ "বর্তমান ডিরেক্টরি" প্রতিটি প্রসেসের নিজস্ব সম্পত্তি। যদি `cd` আলাদা প্রসেস হতো, সে নিজের ডিরেক্টরি বদলে মরে যেত — তোমার shell যেখানে ছিল সেখানেই থাকত! এজন্য shell-এর নিজের অবস্থা বদলানো কমান্ড (`cd`, `export`, `exit`, `source`) builtin।

### ধাপ ৪: fork() — নিজের একটা কপি বানানো

লিনাক্সে নতুন প্রসেস তৈরির মূল নিয়ম অদ্ভুত শোনাতে পারে: **নতুন প্রোগ্রাম সরাসরি "জন্ম" দেওয়া হয় না। আগে চলমান প্রসেস নিজের একটা কপি বানায় (fork), তারপর কপিটা নিজেকে নতুন প্রোগ্রাম দিয়ে বদলে ফেলে (exec)।**

bash `fork()` ডাকে (glibc-তে এটা আসলে `clone()` system call)। kernel:

- নতুন একটা `task_struct` তৈরি করে, নতুন **PID** দেয়;
- parent-এর প্রায় সবকিছু কপি করে: খোলা fd (তাই child-এরও stdin/stdout/stderr = একই টার্মিনাল), বর্তমান ডিরেক্টরি, environment variable, user ID;
- মেমোরি **কপি করে না** — **Copy-on-Write**: parent ও child একই physical page শেয়ার করে, page-গুলো read-only চিহ্নিত হয়। কেউ লিখতে চাইলে শুধু তখনই ঐ page-টার কপি হয়। এতে fork প্রায় তাৎক্ষণিক।

`fork()` একবার ডাকা হয়, কিন্তু **দুইবার ফেরে**: parent-এ ফেরত দেয় child-এর PID, আর child-এ ফেরত দেয় 0। এভাবে কোড জানে সে parent না child।

**উপমা: জেরক্স করা রেসিপি বই।** মা (bash) রান্নাঘরে কাজ করছেন। মেয়ে এলো (fork) — শুরুতে সে মায়ের হুবহু কপি, একই রান্নাঘর, একই চাবির গোছা (fd)। তারপর মেয়ে তার হাতের বইটা ফেলে সম্পূর্ণ নতুন বই (ls) খুলে সেই অনুযায়ী কাজ শুরু করল (exec)। মা অপেক্ষা করছেন মেয়ের কাজ শেষ হওয়ার (wait)।

### ধাপ ৫: execve() — নতুন প্রোগ্রামে রূপান্তর

child প্রসেস `execve("/usr/bin/ls", ["ls", "--color=auto", "-l", "/home"], envp)` ডাকে। kernel:

- পুরনো মেমোরি (bash-এর কোড ও ডেটার কপি) ফেলে দেয়;
- `/usr/bin/ls` ফাইলটা খোলে, ELF header পড়ে, পারমিশন (execute bit) যাচাই করে;
- নতুন মেমোরি তৈরি করে, dynamic loader (`ld-linux-x86-64.so.2`) map করে;
- **PID একই থাকে**, খোলা fd-গুলোও থাকে (যেগুলো close-on-exec চিহ্নিত না) — এজন্যই `ls`-এর output তোমার টার্মিনালে আসে।

execve সফল হলে কখনো ফেরে না — কারণ যে কোড ডেকেছিল, সে আর নেই! এরপর dynamic loader glibc লোড করে, আর `ls`-এর `main()` চলতে শুরু করে। এই পর্যায়টাই উপরে strace উদাহরণে `execve` → `openat(libc.so.6)` হিসেবে দেখেছ।

### ধাপ ৬: ls তার কাজ করে

`ls -l /home` করে: `openat("/home")` → `getdents64` (ডিরেক্টরির এন্ট্রি পড়া) → প্রতিটি এন্ট্রির জন্য `statx`/`newfstatat`/`lstat` (মালিক, পারমিশন, আকার জানতে; ঠিক কোনটা তা glibc ভার্সনের ওপর নির্ভর করে) → user ID-কে নাম বানাতে `/etc/passwd` পড়া → `write(1, ...)` দিয়ে ফলাফল লেখা। VFS, ফাইলসিস্টেম driver, page cache, ডিস্ক driver — সবাই এখানে কাজ করছে।

### ধাপ ৭: exit ও wait — শেষকৃত্য

`ls` শেষে `exit_group(0)` ডাকে। kernel তার মেমোরি, fd সব মুক্ত করে, কিন্তু প্রসেসের একটা ছোট্ট রেকর্ড (PID ও **exit status**) রেখে দেয়, যতক্ষণ না parent সেটা সংগ্রহ করে। এই মৃত-কিন্তু-সংগৃহীত-না অবস্থাকে বলে **zombie** (`ps`-এ state `Z`)।

এদিকে bash `wait4()` (glibc-র `waitpid()`) দিয়ে ঘুমিয়ে অপেক্ষা করছিল। child মরলে kernel bash-কে `SIGCHLD` signal দেয় ও জাগায়; bash exit status নেয় (zombie দূর হয়), `$?` variable-এ রাখে, তারপর আবার prompt দেখায়:

```
$ ls -l /home
total 4
drwxr-x--- 25 rahim rahim 4096 Nov 12 09:30 rahim
$ echo $?
0
```

`$?` = শেষ কমান্ডের exit status। 0 = সফল, অন্য কিছু = কোনো সমস্যা।

### পুরো ছবি একসাথে

```
  bash (PID 1000)
     │  ① লাইন পড়ল: "ls -l /home"   ② parse/expand   ③ PATH-এ /usr/bin/ls পেল
     │
     │  ④ fork() / clone()
     ├──────────────────────────────────┐
     │ parent (PID 1000) পেল 1234        │ child (PID 1234) পেল 0
     │                                  │  (bash-এরই কপি, CoW মেমোরি)
     │                                  │
     │ ⑦ wait4(1234) ... ঘুম 💤          │ ⑤ execve("/usr/bin/ls", ...)
     │                                  │    → এখন এটা ls (PID তবু 1234)
     │                                  │
     │                                  │ ⑥ openat, getdents64, statx...
     │                                  │    write(1, ...) → টার্মিনালে লেখা
     │                                  │
     │                                  │   exit_group(0) → zombie
     │ ◄──── SIGCHLD + exit status 0 ───┘
     │
     │ $? = 0, prompt আবার দেখাও
     ▼
  $ _
```

নিজে প্রমাণ দেখো — strace দিয়ে bash-এর fork/exec:

```
$ strace -f -e trace=clone,clone3,execve,wait4 bash -c 'ls /tmp > /dev/null; true'
execve("/usr/bin/bash", ["bash", "-c", "ls /tmp > /dev/null; true"], 0x7ffc... /* 31 vars */) = 0
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f...) = 5678
strace: Process 5678 attached
[pid  5678] execve("/usr/bin/ls", ["ls", "/tmp"], 0x55... /* 31 vars */) = 0
[pid  5678] +++ exited with 0 +++
wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 5678
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=5678, si_uid=1000, si_status=0, si_utime=0, si_stime=0} ---
+++ exited with 0 +++
```

(শেষে `; true` দিয়েছি, কারণ `bash -c 'ls /tmp'` এর মতো একটাই কমান্ড হলে bash অনেক সময় fork না করে নিজেই সরাসরি `exec` করে ফেলে — একটা অপটিমাইজেশন। PID, address, ক্রম তোমার সিস্টেমে আলাদা হতে পারে; কিছু সিস্টেমে `wait4` আর `SIGCHLD` লাইনের ক্রম উল্টো দেখাতে পারে।)

> 📝 **মনে রাখো:** লিনাক্সের প্রতিটি প্রসেসের একটা parent আছে। সবার আদি পিতা হলো **PID 1** — আধুনিক distro-তে `systemd` — যেটা kernel বুটের শেষে নিজে চালু করে। কোনো প্রসেসের parent আগে মরে গেলে সেই "এতিম" প্রসেসকে PID 1 (বা একটা subreaper) দত্তক নেয়। `pstree` কমান্ড দিয়ে এই পারিবারিক গাছ দেখতে পারো। systemd [অধ্যায় ২৬](26-systemd-services.md)-এ।

```
$ pstree -p | head -5
systemd(1)─┬─ModemManager(812)─┬─{ModemManager}(830)
           ├─NetworkManager(790)─┬─{NetworkManager}(815)
           ├─gnome-terminal-(2210)─┬─bash(2230)───pstree(2301)
           ...
```

---

## ২.১৩ বুট থেকে prompt — architecture-এর টাইমলাইন (সংক্ষেপে)

পুরো আর্কিটেকচার কোন ক্রমে "জেগে ওঠে", এক নজরে (বিস্তারিত [অধ্যায় ২৭](27-boot-process.md)-এ):

```
 পাওয়ার অন → UEFI/BIOS firmware (hardware পরীক্ষা)
          → Bootloader (GRUB / systemd-boot)
          → Kernel (vmlinuz) + initramfs লোড → kernel hardware চেনে, driver/module লোড
          → আসল root filesystem মাউন্ট
          → PID 1 = systemd (প্রথম user-space প্রসেস)
          → সার্ভিস চালু (নেটওয়ার্ক, লগ, ssh...)
          → লগইন স্ক্রিন (GDM ইত্যাদি) বা টেক্সট login
          → তোমার shell / desktop
```

kernel-এর বুট মেসেজ দেখতে পারো:

```
$ sudo dmesg | head -3
[    0.000000] Linux version 6.8.0-45-generic (buildd@lcy02-amd64-075) ...
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-6.8.0-45-generic root=UUID=... ro quiet splash
[    0.000000] KERNEL supported cpus:
```

(অনেক distro-তে সাধারণ user-এর জন্য `dmesg` বন্ধ থাকে — `kernel.dmesg_restrict` — তাই `sudo`; বিকল্প `journalctl -k`। লগ [অধ্যায় ২৮](28-logs.md)-এ।)

---

## ২.১৪ Architecture-সম্পর্কিত কীবোর্ড শর্টকাট

এই শর্টকাটগুলো আসলে kernel-এর কাজ — tty driver কী চাপা দেখে প্রসেসকে **signal** পাঠায়, বা kernel নিজে সরাসরি সাড়া দেয়।

### টার্মিনালের signal-শর্টকাট (tty driver পাঠায়)

| শর্টকাট | কী ঘটে (ভেতরে) | ফলাফল |
|---|---|---|
| `Ctrl + C` | foreground প্রসেসে `SIGINT` | প্রোগ্রাম থামে (সাধারণত) |
| `Ctrl + Z` | `SIGTSTP` | প্রোগ্রাম সাময়িক স্থগিত (`fg` দিয়ে ফেরত — [অধ্যায় ২৫](25-jobs-tmux.md)) |
| `Ctrl + \` | `SIGQUIT` | প্রোগ্রাম থামে, অনুমতি থাকলে core dump |
| `Ctrl + D` | কোনো signal না; ইনপুট লাইনের শুরুতে চাপলে `read()` 0 ফেরত দেয় = **EOF** | `cat`-এর ইনপুট শেষ; খালি prompt-এ চাপলে shell বন্ধ |
| `Ctrl + S` | tty-র output থামায় (XOFF flow control) | স্ক্রিন "জমে গেছে" মনে হয় |
| `Ctrl + Q` | output আবার চালু (XON) | `Ctrl+S`-এর পর জমে গেলে এটা চাপো |

```
$ stty -a | grep -o 'intr = [^;]*\|quit = [^;]*\|eof = [^;]*\|susp = [^;]*'
intr = ^C
quit = ^\
eof = ^D
susp = ^Z
```

`stty -a` দিয়ে দেখা যায় tty driver কোন কী-কে কোন কাজে লাগাচ্ছে — প্রমাণ যে এটা kernel-এর tty স্তরের সেটিং, bash-এর না।

### Virtual console ও সিস্টেম শর্টকাট

| শর্টকাট | কাজ |
|---|---|
| `Ctrl + Alt + F1` … `F6` | kernel-এর virtual console (টেক্সট টার্মিনাল, tty1–tty6) এ যাওয়া। কোনটায় GUI আছে তা distro অনুযায়ী আলাদা — Ubuntu-তে GDM সাধারণত tty1, তোমার desktop session প্রায়ই tty2-এ |
| `Alt + F1`…`F6` / `Alt + ←/→` | টেক্সট console থেকে আরেক console-এ (ইতিমধ্যে টেক্সট মোডে থাকলে) |
| `Ctrl + Alt + Delete` | টেক্সট console-এ systemd-কে reboot সংকেত দেয় (`ctrl-alt-del.target`); GNOME-এ সাধারণত log-out ডায়ালগ |
| `Alt + SysRq + <কী>` | **Magic SysRq** — kernel সরাসরি সাড়া দেয়, user space জমে গেলেও |

### Magic SysRq — শেষ ভরসা

kernel-এর ভেতরে বসানো একটা জরুরি কীবোর্ড। সিস্টেম পুরো জমে গেলে (মাউস নড়ছে না, কিছু কাজ করছে না) ডিস্ক নষ্ট না করে নিরাপদে reboot করার জন্য বিখ্যাত ক্রম **REISUB** (মনে রাখার বাক্য: "**R**eboot **E**ven **I**f **S**ystem **U**tterly **B**roken"), প্রতিটির মাঝে কয়েক সেকেন্ড বিরতি দিয়ে, `Alt + SysRq` চেপে ধরে:

| কী | কাজ |
|---|---|
| `R` | কীবোর্ড raw mode থেকে ফেরত নেয় |
| `E` | init (PID 1) বাদে সব প্রসেসে `SIGTERM` |
| `I` | init বাদে সব প্রসেসে `SIGKILL` |
| `S` | সব ফাইলসিস্টেম sync (RAM-এর ডেটা ডিস্কে লেখা) |
| `U` | সব ফাইলসিস্টেম read-only হিসেবে remount |
| `B` | তৎক্ষণাৎ reboot |

```
$ cat /proc/sys/kernel/sysrq
176
```

এই মান একটা bitmask — কোন SysRq ফাংশন অনুমোদিত। `1` = সব চালু, `0` = সব বন্ধ। Ubuntu-র ডিফল্ট 176 (= 128 + 32 + 16: reboot, remount read-only, sync অনুমোদিত, কিন্তু `E`/`I` এর মতো signal পাঠানো বন্ধ)। Fedora/RHEL-এ ডিফল্ট সাধারণত 16 (শুধু sync)। অনেক ল্যাপটপে SysRq কী হলো `PrtSc`, কখনো `Fn` সহ চাপতে হয়।

> ⚠️ **সতর্কতা:** Magic SysRq সত্যিকারের জরুরি অবস্থার জন্য। `B` সরাসরি reboot করে — `S` আর `U` না করলে না-সংরক্ষিত ডেটা হারাবে, ফাইলসিস্টেম ক্ষতিগ্রস্ত হতে পারে। মজা করে চাপবে না, বিশেষ করে সার্ভারে।

সব শর্টকাটের পূর্ণ তালিকা: [পরিশিষ্ট ক](appendix-a-shortcuts-cheatsheet.md)।

---

## ২.১৫ সাধারণ ভুল ধারণা

| ভুল ধারণা | সঠিক |
|---|---|
| "লিনাক্স = Ubuntu" | Ubuntu একটা distribution; লিনাক্স হলো kernel, যার ওপর শত শত distro তৈরি ([অধ্যায় ৩](03-distro-o-installation.md)) |
| "root হলে kernel mode-এ চলে" | না; root-ও user space-এ, শুধু বেশি অনুমতি পায় |
| "shell = terminal" | terminal জানালা, shell তার ভেতরে চলা প্রোগ্রাম |
| "free মেমোরি কম মানে RAM শেষ" | page cache দেখো; আসল সূচক `available` |
| "`/proc` এর ফাইল ডিস্কে থাকে" | এগুলো virtual, kernel তৎক্ষণাৎ তৈরি করে |
| "monolithic মানে পুরনো, অনমনীয়" | লিনাক্স module দিয়ে চলন্ত অবস্থায় বদলানো যায় |
| "প্রতিটি কমান্ড আলাদা প্রসেস" | builtin (`cd`, `echo`...) bash নিজেই চালায় |

---

## হাতে-কলমে অনুশীলন

যেকোনো লিনাক্স মেশিন, VM বা WSL-এ (WSL2-ও আসল লিনাক্স kernel চালায়) চেষ্টা করো। `strace` না থাকলে আগে ইনস্টল করো:

```
$ sudo apt install strace       # Ubuntu/Debian
$ sudo dnf install strace       # Fedora/RHEL
```

### ল্যাব ১: তোমার সিস্টেমের পরিচয়পত্র

```
$ uname -srm
$ cat /etc/os-release | head -3
$ lscpu | grep 'Model name'
$ free -h
$ ldd --version | head -1
```

একটা খাতায় লেখো: kernel ভার্সন, architecture, distro, CPU, RAM, glibc ভার্সন। প্রশ্ন: তোমার kernel কি CFS না EEVDF scheduler ব্যবহার করে? (ইঙ্গিত: ভার্সন 6.6 এর আগে না পরে?)

### ল্যাব ২: Kernel module অনুসন্ধান

```
$ lsmod | wc -l                 # কতগুলো module লোড আছে (হেডারসহ)
$ lsmod | sort -k2 -n -r | head -5   # সবচেয়ে বড় ৫টা module
$ modinfo -F description ext4   # ext4 module-এর বর্ণনা
$ lspci -k | grep -A3 -i vga     # তোমার গ্রাফিক্স কার্ড কোন driver ব্যবহার করে
```

(`ext4` তোমার সিস্টেমে built-in হলে `modinfo` তবুও তথ্য দেখাতে পারে, বা "not found" বলতে পারে — দুটোই স্বাভাবিক।)

### ল্যাব ৩: System call গোয়েন্দাগিরি

```
$ strace -c ls /etc > /dev/null
$ strace -e trace=openat cat /etc/hostname
$ strace -e trace=%file ls /nonexistent 2>&1 | grep ENOENT
```

প্রশ্ন: `strace -c` অনুযায়ী `ls` সবচেয়ে বেশি কোন system call করেছে? তৃতীয় কমান্ডে `ls` কোন system call দিয়ে বুঝল ফাইলটা নেই?

### ল্যাব ৪: Everything is a file

```
$ cat /proc/cpuinfo | grep 'model name' | head -1
$ cat /proc/meminfo | head -3
$ echo "আমি সরাসরি টার্মিনাল ফাইলে লিখছি" > /dev/tty
$ tty                           # তোমার টার্মিনাল ফাইলের নাম
$ ls -l /proc/$$/fd
$ head -c 12 /dev/urandom | base64
$ echo "হারিয়ে যাবে" > /dev/null ; echo $?
```

দ্বিতীয় একটা টার্মিনাল খোলো, সেখানে `tty` চালাও (ধরো `/dev/pts/1`)। এখন প্রথম টার্মিনাল থেকে `echo "হ্যালো পাশের জানালা" > /dev/pts/1` লেখো — দ্বিতীয় জানালায় লেখাটা হাজির হবে!

### ল্যাব ৫: fork/exec চোখে দেখা

```
$ echo "আমার shell-এর PID: $$"
$ bash                          # একটা child shell চালাও
$ echo "নতুন shell-এর PID: $$, parent: $PPID"
$ pstree -p $PPID
$ exit
$ type cd ls pwd echo
$ strace -f -e trace=clone,clone3,execve bash -c 'date; true' 2>&1 | grep -E 'clone|execve'
```

লক্ষ করো: child shell-এর `$PPID` তোমার প্রথম shell-এর `$$`-এর সমান। `strace` output-এ দুটো `execve` দেখবে — একটা `bash`-এর, একটা `date`-এর — আর মাঝে একটা `clone`।

### ল্যাব ৬: builtin বনাম বাইরের কমান্ড

```
$ type -a echo pwd
$ strace -c -f bash -c 'for i in 1 2 3; do echo hi; done' 2>&1 | grep -E 'clone|execve|total'
$ strace -c -f bash -c 'for i in 1 2 3; do /bin/echo hi; done' 2>&1 | grep -E 'clone|execve|total'
```

দ্বিতীয় ক্ষেত্রে `clone` আর `execve` অনেক বেশি হবে — কারণ প্রতিবার নতুন প্রসেস। এজন্যই বড় script-এ builtin ব্যবহার দ্রুততর।

---

## সারসংক্ষেপ

**মূল ধারণা:**

- লিনাক্সের স্তর: Hardware → Kernel → System call interface → Libraries (glibc) → Shell → Applications।
- Kernel ring 0-তে (kernel space) পূর্ণ ক্ষমতায়, সব user প্রোগ্রাম (root-এরটাও) ring 3-তে (user space) সীমিত ক্ষমতায় চলে।
- লিনাক্স monolithic kernel, কিন্তু loadable module দিয়ে নমনীয়।
- Kernel-এর কাজ: scheduling (CFS → EEVDF), memory (virtual memory, paging, CoW, page cache, swap, OOM), VFS, device drivers (`/dev`, char/block, udev), networking (TCP/IP stack, netfilter)।
- System call হলো user space থেকে kernel-এ যাওয়ার একমাত্র আনুষ্ঠানিক দরজা; `strace` দিয়ে দেখা যায়।
- "Everything is a file" — একই `open/read/write/close` দিয়ে ফাইল, device, kernel তথ্য, pipe, socket।
- কমান্ড চালানো = parse → খোঁজা (alias/builtin/PATH) → `fork` (clone, CoW) → `execve` → চলা → `exit` → parent-এর `wait` → `$?`।

**দ্রুত রেফারেন্স:**

| কমান্ড / শর্টকাট | কাজ |
|---|---|
| `uname -a` / `uname -r` | kernel-এর সব তথ্য / ভার্সন |
| `cat /proc/version` | kernel ভার্সন ও build তথ্য |
| `lscpu`, `free -h` | CPU ও মেমোরি তথ্য |
| `lsmod` | লোড করা kernel module |
| `modinfo <mod>` | module-এর বিবরণ |
| `sudo modprobe <mod>` / `-r` | module লোড / আনলোড (dependency সহ) |
| `lspci -k`, `lsusb` | PCI/USB device ও তাদের driver |
| `strace <cmd>` | system call দেখা |
| `strace -c`, `-f`, `-e trace=`, `-p` | সারসংক্ষেপ, child অনুসরণ, ফিল্টার, চলমান প্রসেস |
| `ldd <prog>` | প্রোগ্রামের shared library |
| `ldd --version` | glibc ভার্সন |
| `type -a <cmd>` | কমান্ডটা alias/builtin/ফাইল — কোনটা |
| `echo $PATH`, `echo $?`, `echo $$` | PATH, শেষ exit status, shell-এর PID |
| `ls -l /proc/$$/fd` | shell-এর খোলা file descriptor |
| `pstree -p` | প্রসেসের পারিবারিক গাছ |
| `sudo dmesg` / `journalctl -k` | kernel মেসেজ |
| `getconf PAGESIZE` | মেমোরি page-এর আকার |
| `Ctrl+C` / `Ctrl+Z` / `Ctrl+\` | SIGINT / SIGTSTP / SIGQUIT |
| `Ctrl+D` | EOF (ইনপুট শেষ) |
| `Ctrl+S` / `Ctrl+Q` | output থামানো / চালু |
| `Ctrl+Alt+F1…F6` | virtual console বদলানো |
| `Alt+SysRq+R,E,I,S,U,B` | জমে যাওয়া সিস্টেম নিরাপদে reboot |

---

## নিজেকে যাচাই করো

1. লিনাক্স সিস্টেমের ছয়টা স্তর নিচ থেকে ওপরে ক্রমানুসারে লেখো।
2. kernel space আর user space কেন আলাদা রাখা হয়? অন্তত তিনটা কারণ বলো।
3. root user হিসেবে চলা প্রোগ্রাম কি kernel space-এ চলে? ব্যাখ্যা করো।
4. monolithic kernel আর microkernel-এর মূল পার্থক্য কী? লিনাক্স monolithic হয়েও কীভাবে নমনীয়?
5. একটা `.ko` ফাইল কী? `modprobe` আর `insmod`-এর পার্থক্য কী?
6. `printf("hi\n")` থেকে স্ক্রিনে লেখা পর্যন্ত যাত্রায় glibc ও kernel কে কী করে?
7. `strace` output-এ `openat(...) = -1 ENOENT` মানে কী? এটা troubleshooting-এ কীভাবে কাজে লাগে?
8. `/proc/cpuinfo`-এর আকার `ls -l`-এ 0 দেখায়, অথচ `cat` করলে অনেক লেখা আসে। কেন?
9. `fork()` কেন দ্রুত, যদিও সে পুরো প্রসেসের "কপি" বানায়? `fork()` কয়বার ফেরে, কী ফেরত দেয়?
10. `cd` কেন একটা আলাদা প্রোগ্রাম হতে পারে না?

<details><summary>উত্তর দেখো</summary>

1. Hardware → Kernel → System call interface → Libraries (glibc) → Shell → Applications।

2. (ক) **স্থিতিশীলতা** — একটা প্রোগ্রাম crash করলে পুরো সিস্টেম পড়ে না; (খ) **নিরাপত্তা** — এক প্রোগ্রাম অন্যের মেমোরি বা hardware-এ হাত দিতে পারে না; (গ) **ন্যায্য বণ্টন** — CPU/RAM/ডিস্ক kernel ভাগ করে, কেউ একা দখল করতে পারে না; (ঘ) **বিমূর্তকরণ** — প্রোগ্রামারকে hardware-এর খুঁটিনাটি জানতে হয় না।

3. না। root-এর প্রোগ্রামও user space-এ (ring 3) চলে এবং system call দিয়েই kernel-এর সেবা নেয়। পার্থক্য হলো, kernel root-এর অনুরোধ বেশি অনুমোদন করে (যেমন যেকোনো ফাইল পড়া, module লোড করা)।

4. monolithic kernel-এ scheduler, memory, filesystem, driver, network সব একই kernel space-এ একটা প্রোগ্রাম হিসেবে চলে (দ্রুত, কিন্তু একটা driver বাগ পুরো সিস্টেম ফেলে দিতে পারে)। microkernel-এ kernel-এ শুধু মৌলিক জিনিস থাকে, বাকিগুলো user space-এ আলাদা server হিসেবে চলে, message দিয়ে কথা বলে। লিনাক্স **loadable kernel module** দিয়ে চলন্ত অবস্থায় driver ইত্যাদি যোগ-বিয়োগ করতে পারে, reboot ছাড়াই।

5. `.ko` (kernel object) হলো একটা loadable kernel module ফাইল — kernel-এর একটা টুকরো যা চলন্ত kernel-এ লোড করা যায়। `modprobe` নাম দিয়ে module খোঁজে (`/lib/modules/$(uname -r)` থেকে) এবং `modules.dep` দেখে dependency-ও লোড করে; `insmod` শুধু নির্দিষ্ট path-এর একটা ফাইল লোড করে, dependency দেখে না।

6. glibc-র `printf` লেখাটা format করে buffer-এ রাখে, তারপর `write()` wrapper ডাকে, যেটা register-এ syscall নম্বর (x86_64-এ write = 1), fd (1), buffer address, দৈর্ঘ্য বসিয়ে `syscall` instruction চালায়। CPU ring 0-তে যায়; kernel syscall table থেকে `sys_write` চালায়, fd 1 যে টার্মিনাল (pty/tty) তার driver-এর কাছে ডেটা পাঠায়, ফলাফল ফেরত দিয়ে ring 3-তে ফেরে। glibc ফলাফল দেখে (error হলে `errno` সেট করে)।

7. প্রোগ্রামটা একটা ফাইল খুলতে চেয়েছিল, কিন্তু kernel "No such file or directory" error দিয়েছে। `strace -f -e trace=%file` চালিয়ে `ENOENT` খুঁজলে বোঝা যায় প্রোগ্রাম ঠিক কোন path-এ কোন ফাইল (config, library) খুঁজছে — যা প্রোগ্রামের নিজের error মেসেজ প্রায়ই বলে না।

8. `/proc` একটা virtual filesystem (procfs) — ফাইলগুলো ডিস্কে নেই। পড়ার মুহূর্তে kernel লেখাটা তৈরি করে দেয়, তাই আগে থেকে "আকার" জানা থাকে না, 0 দেখায়।

9. কারণ **Copy-on-Write**: kernel মেমোরি আসলে কপি করে না; parent ও child একই physical page শেয়ার করে, শুধু কেউ লিখতে গেলে ঐ page-টা কপি হয়। আর বেশিরভাগ child তো সাথে সাথেই `execve` করে পুরো মেমোরি বদলে ফেলে। `fork()` একবার ডাকা হয় কিন্তু দুইবার ফেরে: parent-এ child-এর PID, child-এ 0 (ব্যর্থ হলে parent-এ −1)।

10. বর্তমান ডিরেক্টরি প্রতিটি প্রসেসের নিজস্ব সম্পত্তি। `cd` যদি আলাদা প্রোগ্রাম হতো, তাহলে fork হওয়া child প্রসেস নিজের ডিরেক্টরি বদলে exit করত — parent shell-এর ডিরেক্টরি অপরিবর্তিত থাকত। তাই shell-এর নিজের অবস্থা বদলাতে `cd` অবশ্যই shell-এর builtin হতে হবে।

</details>

---

[← আগের অধ্যায়](01-linux-ki-o-keno.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](03-distro-o-installation.md)

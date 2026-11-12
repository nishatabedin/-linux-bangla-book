# অধ্যায় ৪৬: পারফরম্যান্স মনিটরিং ও ট্রাবলশুটিং

> "সার্ভার স্লো হয়ে গেছে!" এই বাক্যটা একজন লিনাক্স অ্যাডমিন জীবনে হাজারবার শুনবে। নতুনরা এটা শুনে ঘাবড়ে যায়, আন্দাজে সার্ভার রিস্টার্ট করে। অভিজ্ঞরা একটা নির্দিষ্ট পদ্ধতি মেনে কয়েক মিনিটের মধ্যে সমস্যার উৎস খুঁজে বের করে। এই অধ্যায় তোমাকে সেই পদ্ধতিটা শেখাবে।

## এই অধ্যায়ে যা শিখবে

- পারফরম্যান্স সমস্যা খোঁজার বৈজ্ঞানিক উপায়: **USE পদ্ধতি** (Utilization, Saturation, Errors)
- `uptime` আর **load average** আসলে কী মাপে এবং সংখ্যাগুলো কীভাবে পড়তে হয়
- CPU দেখা: `top`, `mpstat`, `pidstat`, আর `us`, `sy`, `wa`, `st` এসবের মানে
- Memory বোঝা: `free -h`, **buff/cache** আর **available** এর রহস্য, `/proc/meminfo`, `vmstat`, swap, **OOM killer**
- Disk I/O: `iostat`, `iotop`, আর `%util` ও `await` এর মানে
- Network: `iftop`, `nethogs`, `sar -n DEV`
- **sysstat/sar** দিয়ে অতীতের পারফরম্যান্স ডেটা দেখা ("গত রাত ৩টায় কী হয়েছিল?")
- `lsof` দিয়ে খোলা ফাইল ও পোর্ট খোঁজা
- `strace` ও `ltrace` দিয়ে একটা প্রোগ্রামের ভেতরে উঁকি দেওয়া
- `perf` দিয়ে CPU প্রোফাইলিং-এর পরিচিতি
- আধুনিক ড্যাশবোর্ড টুল: `htop`, `btop`, `glances`
- **PSI** (Pressure Stall Information): আধুনিক কার্নেলের "চাপ মাপার যন্ত্র"
- "সার্ভার স্লো" সমস্যার ধাপে ধাপে ডায়াগনোসিস চেকলিস্ট
- সাধারণ সমস্যা ও সমাধান: disk full, high load, port already in use, permission denied

---

## ৪৬.১ পারফরম্যান্স ট্রাবলশুটিং কেন একটা "পদ্ধতি", আন্দাজ না

ধরো তুমি ডাক্তারের কাছে গেলে, বললে "শরীর খারাপ লাগছে"। ভালো ডাক্তার সাথে সাথে ওষুধ লিখে দেবে না। আগে জ্বর মাপবে, প্রেশার দেখবে, পালস গুনবে, দরকার হলে রক্ত পরীক্ষা করাবে। প্রতিটা অঙ্গ একটা একটা করে চেক করবে, তারপর রোগ ধরবে।

লিনাক্স সার্ভারের ক্ষেত্রেও একই কথা। কম্পিউটারের "অঙ্গ" মূলত চারটা:

```
            ┌──────────────────────────────────────────┐
            │              তোমার সার্ভার                │
            │                                          │
            │   ┌────────┐   ┌────────┐                │
            │   │  CPU   │   │ Memory │                │
            │   │ (মগজ)  │   │ (টেবিল) │                │
            │   └───┬────┘   └───┬────┘                │
            │       │            │                     │
            │   ┌───┴────────────┴───┐                 │
            │   │   Kernel (ম্যানেজার)  │                 │
            │   └───┬────────────┬───┘                 │
            │       │            │                     │
            │   ┌───┴────┐   ┌───┴─────┐               │
            │   │  Disk  │   │ Network │               │
            │   │(আলমারি)│   │(দরজা/রাস্তা)│               │
            │   └────────┘   └─────────┘               │
            └──────────────────────────────────────────┘
```

একটা উপমা দিই। একটা রেস্টুরেন্টের রান্নাঘর ভাবো:

| কম্পিউটারের অংশ | রান্নাঘরের উপমা | "স্লো" হলে কেমন দেখায় |
|---|---|---|
| CPU | রাঁধুনি (cook) | রাঁধুনি সবাই ব্যস্ত, অর্ডার জমে আছে |
| Memory (RAM) | রান্নার টেবিল (কাজের জায়গা) | টেবিল ভরা, বারবার জিনিস আলমারিতে তুলে রাখতে হচ্ছে |
| Disk | স্টোররুম/আলমারি | স্টোররুম থেকে মাল আনতে অনেক সময় লাগছে |
| Network | ওয়েটার আর দরজা | ওয়েটার কম, দরজায় ভিড় |

গ্রাহক যখন বলে "খাবার আসতে দেরি হচ্ছে", তখন তুমি জানো না সমস্যা রাঁধুনিতে, টেবিলে, স্টোররুমে নাকি ওয়েটারে। তাই প্রতিটা চেক করতে হবে। এই চেক করার সবচেয়ে বিখ্যাত পদ্ধতির নাম **USE Method**।

---

## ৪৬.২ USE পদ্ধতি

USE পদ্ধতি তৈরি করেছেন পারফরম্যান্স বিশেষজ্ঞ **Brendan Gregg**। মূল কথা খুব সহজ:

> **প্রতিটা resource (সম্পদ) এর জন্য তিনটা জিনিস চেক করো: Utilization, Saturation, Errors।**

| অক্ষর | মানে | সহজ বাংলায় | রান্নাঘরের উপমা |
|---|---|---|---|
| **U**tilization | resource কত শতাংশ সময় ব্যস্ত ছিল | কতটা কাজে লাগছে | রাঁধুনি দিনের কত % সময় রান্না করছে |
| **S**aturation | resource এর বাইরে কত কাজ অপেক্ষায় (queue) | কত কাজ লাইনে দাঁড়িয়ে | কতগুলো অর্ডার স্লিপ ঝুলছে, এখনো শুরু হয়নি |
| **E**rrors | error কয়টা হচ্ছে | কাজ কতবার ভুল/ব্যর্থ হলো | কতগুলো ডিশ পুড়ে গেল, ফেরত এলো |

📝 **মনে রাখো:** Utilization ১০০% হওয়া মানেই সমস্যা না। রাঁধুনি সারাক্ষণ ব্যস্ত, কিন্তু কোনো অর্ডার অপেক্ষায় নেই, তাহলে সব ঠিক আছে। আসল বিপদের সংকেত হলো **Saturation**, মানে যখন কাজ লাইনে দাঁড়িয়ে থাকে।

### USE চেকলিস্ট: কোন resource এর জন্য কোন টুল

| Resource | Utilization | Saturation | Errors |
|---|---|---|---|
| CPU | `mpstat -P ALL 1`, `top` এর `%Cpu` | load average > CPU সংখ্যা, `vmstat` এর `r` কলাম, `/proc/pressure/cpu` | `dmesg` (machine check error) |
| Memory | `free -h` (used vs available) | swap ব্যবহার, `vmstat` এর `si`/`so`, OOM, `/proc/pressure/memory` | `dmesg` এ "Out of memory", ECC error |
| Disk I/O | `iostat -x` এর `%util` | `iostat` এর `aqu-sz`, `await`, `vmstat` এর `b` | `dmesg` এ I/O error, `smartctl` |
| Disk space | `df -h`, `df -i` | (পূর্ণ হলে লেখাই যায় না) | "No space left on device" |
| Network | `sar -n DEV 1`, `iftop` (লিঙ্ক স্পিডের তুলনায়) | `ss -s`, drops, retransmits (`sar -n ETCP 1`) | `ip -s link` এর errors/dropped |

এই টেবিলটা মাথায় রাখলে তুমি কখনো "কোথা থেকে শুরু করব" ভেবে আটকে থাকবে না। এখন প্রতিটা টুল বিস্তারিত দেখি।

---

## ৪৬.৩ `uptime` এবং Load Average: সিস্টেমের "জ্বর মাপার থার্মোমিটার"

### `uptime` কী করে

সিস্টেম কতক্ষণ চালু আছে, কতজন ইউজার লগইন করে আছে আর load average কত, এক লাইনে দেখায়।

```bash
$ uptime
 14:32:10 up 12 days,  3:45,  2 users,  load average: 0.52, 0.78, 1.05
```

| অংশ | মানে |
|---|---|
| `14:32:10` | এখনকার সময় |
| `up 12 days, 3:45` | ১২ দিন ৩ ঘণ্টা ৪৫ মিনিট ধরে চালু (শেষ reboot এর পর থেকে) |
| `2 users` | লগইন করা session সংখ্যা |
| `load average: 0.52, 0.78, 1.05` | গত **১ মিনিট, ৫ মিনিট, ১৫ মিনিটের** গড় load |

দরকারি অপশন:

| অপশন | কাজ |
|---|---|
| `uptime -p` | সুন্দর করে দেখায়: `up 1 week, 5 days, 3 hours, 45 minutes` |
| `uptime -s` | কবে থেকে চালু: `2026-10-31 10:47:02` |

### Load average আসলে কী?

এটা লিনাক্সের সবচেয়ে ভুল বোঝা সংখ্যাগুলোর একটা। অনেকে ভাবে load মানে "CPU কত % ব্যবহার হচ্ছে"। **না।**

**Load average = গড়ে কতগুলো task (process/thread) একই সময়ে চলছে বা চলার জন্য অপেক্ষা করছে।**

লিনাক্সে এর মধ্যে দুই ধরনের task গোনা হয়:

1. **R (Running/Runnable)**: CPU তে চলছে, বা CPU পাওয়ার জন্য লাইনে দাঁড়িয়ে।
2. **D (Uninterruptible sleep)**: সাধারণত disk বা NFS এর মতো I/O শেষ হওয়ার অপেক্ষায় আটকে আছে।

⚠️ **সতর্কতা:** দ্বিতীয় পয়েন্টটা লিনাক্সের বিশেষত্ব। অন্য অনেক Unix (যেমন FreeBSD) শুধু CPU এর task গোনে। তাই লিনাক্সে load বেশি মানেই CPU ব্যস্ত না; হতে পারে disk স্লো, তাই অনেক process D state এ আটকে আছে। High load দেখলে সবসময় CPU **আর** I/O দুটোই চেক করবে।

#### উপমা: ব্যাংকের কাউন্টার

ধরো একটা ব্যাংকে ৪টা কাউন্টার (৪টা CPU core)।

```
  load = 2.0  (৪ কাউন্টারে)        load = 4.0              load = 8.0
  ┌──┐┌──┐┌──┐┌──┐               ┌──┐┌──┐┌──┐┌──┐         ┌──┐┌──┐┌──┐┌──┐
  │😀││😀││  ││  │               │😀││😀││😀││😀│         │😀││😀││😀││😀│
  └──┘└──┘└──┘└──┘               └──┘└──┘└──┘└──┘         └──┘└──┘└──┘└──┘
  ২টা কাউন্টার খালি              সব কাউন্টার ব্যস্ত,         😐😐😐😐 লাইনে ৪ জন
  (৫০% ব্যস্ত, আরামদায়ক)         কেউ লাইনে নেই (১০০%)       অপেক্ষা করছে (overload)
```

তাই load এর সংখ্যাটা **CPU core সংখ্যার সাথে তুলনা করে** পড়তে হয়:

```bash
$ nproc
4
```

| load ÷ CPU সংখ্যা | অবস্থা |
|---|---|
| < 0.7 | আরামদায়ক |
| ≈ 1.0 | পুরোপুরি ব্যস্ত, তবে লাইন নেই |
| > 1.0 | কাজ লাইনে দাঁড়িয়ে (saturation)। দীর্ঘক্ষণ থাকলে তদন্ত করো |
| > 2.0 বা বেশি | স্পষ্ট overload, ইউজার স্লো টের পাবে |

💡 **টিপস:** এগুলো শক্ত নিয়ম না, শুধু আন্দাজ। কিছু সার্ভার (যেমন batch job চালানো মেশিন) ইচ্ছা করেই সবসময় ১০০% চলে।

#### তিনটা সংখ্যার ট্রেন্ড পড়া

| উদাহরণ | ব্যাখ্যা |
|---|---|
| `load average: 8.20, 4.10, 1.50` | load **বাড়ছে**। সমস্যা এই মাত্র শুরু হয়েছে, এখনই দেখো |
| `load average: 1.20, 4.50, 8.30` | load **কমছে**। সমস্যা আগে ছিল, এখন সেরে যাচ্ছে |
| `load average: 4.00, 4.00, 4.00` | স্থির load। এটাই এই সার্ভারের স্বাভাবিক অবস্থা হতে পারে |

#### ভেতরে কীভাবে কাজ করে

Kernel প্রতি ৫ সেকেন্ডে R+D state এর task গোনে এবং একটা **exponentially-damped moving average** হিসাব করে। সহজ ভাষায়: নতুন মাপের গুরুত্ব বেশি, পুরনো মাপের প্রভাব ধীরে ধীরে মিলিয়ে যায়। এজন্য "১ মিনিটের গড়" আসলে ঠিক গত ৬০ সেকেন্ডের সরল গড় না, বরং ঝুঁকে থাকা একটা গড়। Kernel এই তিনটা সংখ্যা `/proc/loadavg` ফাইলে রাখে:

```bash
$ cat /proc/loadavg
0.52 0.78 1.05 2/1234 98765
```

শেষের `2/1234` মানে এই মুহূর্তে ২টা task runnable, মোট ১২৩৪টা task (thread) আছে। আর `98765` হলো সর্বশেষ তৈরি হওয়া PID। (`/proc` নিয়ে বিস্তারিত [অধ্যায় ৪৭](47-kernel-proc-sys.md) এ।)

---

## ৪৬.৪ CPU মনিটরিং

### CPU সময় কোথায় যায়: ভাগগুলো চেনো

যেকোনো CPU টুল (top, mpstat, vmstat, sar) CPU সময়কে কয়েকটা ভাগে দেখায়। এগুলো চিনলে অর্ধেক কাজ শেষ।

| ঘর | পুরো নাম | মানে | বেশি হলে কী বুঝবে |
|---|---|---|---|
| `us` | user | সাধারণ প্রোগ্রামের নিজের কোড চালাতে সময় | অ্যাপ্লিকেশন নিজেই ভারী হিসাব করছে (যেমন PHP, Java, compression) |
| `sy` | system | Kernel এর কাজে সময় (system call, network stack, file system) | অনেক বেশি system call, context switch, বা kernel এর ঝামেলা |
| `ni` | nice | কম priority (nice মান বাড়ানো) process এর user সময় | এমনিতে সমস্যা না |
| `id` | idle | CPU অলস বসে ছিল | বেশি হলে ভালো |
| `wa` | iowait | CPU অলস, **এবং** কেউ disk I/O এর অপেক্ষায় | Disk স্লো বা overload। Disk এর দিকে তাকাও |
| `hi` | hardware interrupt | হার্ডওয়্যার interrupt সামলাতে | NIC/disk এর অনেক interrupt |
| `si` | software interrupt | soft interrupt (প্রায়ই network packet প্রসেসিং) | অনেক network ট্রাফিক |
| `st` | steal | VM এর ক্ষেত্রে: hypervisor তোমার CPU সময় অন্য VM কে দিয়ে দিয়েছে | Cloud এর host overload, বা তোমার CPU credit শেষ (যেমন burstable instance)। তোমার সার্ভারের ভেতরে ঠিক করার কিছু নেই |

📝 **মনে রাখো:** `wa` (iowait) আসলে "idle এর একটা ধরন"। CPU কাজ করছে না, শুধু বসে আছে কারণ disk উত্তর দেয়নি। তাই `wa` বেশি হলে CPU কেনার দরকার নেই, disk দেখো।

### `top`: সবচেয়ে পরিচিত লাইভ মনিটর

`top` প্রায় সব লিনাক্সে (procps-ng প্যাকেজের অংশ হিসেবে) আগে থেকেই থাকে। Process ম্যানেজমেন্টের দিক থেকে `top` এর পরিচিতি [অধ্যায় ২৪](24-process-management.md) এ আছে; এখানে আমরা পারফরম্যান্সের চোখে দেখব।

```bash
$ top
top - 14:35:02 up 12 days,  3:48,  2 users,  load average: 3.91, 2.15, 1.20
Tasks: 245 total,   3 running, 242 sleeping,   0 stopped,   0 zombie
%Cpu(s): 72.5 us,  8.1 sy,  0.0 ni, 12.3 id,  6.6 wa,  0.0 hi,  0.5 si,  0.0 st
MiB Mem :   7841.2 total,    312.4 free,   5120.8 used,   2408.0 buff/cache
MiB Swap:   2048.0 total,   1980.5 free,     67.5 used.   2720.4 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   2314 www-data  20   0  812340 245120  18432 R  98.3   3.1  45:12.08 php-fpm8.3
   1877 mysql     20   0 2456780   1.2g  32100 S  65.1  15.7 120:33.41 mysqld
   3021 root      20   0   12345   4400   3500 R   2.0   0.1   0:00.15 top
```

#### উপরের "summary" অংশ লাইন ধরে

| লাইন | কী বলে |
|---|---|
| ১ম লাইন | `uptime` এর মতোই: সময়, uptime, ইউজার, load average |
| `Tasks:` | মোট process, কয়টা চলছে (R), ঘুমাচ্ছে (S), থেমে আছে (T), **zombie** (Z) |
| `%Cpu(s):` | উপরের টেবিলের ভাগগুলো (সব CPU এর গড়) |
| `MiB Mem:` | মোট, free, used, buff/cache (memory অংশে বিস্তারিত) |
| `MiB Swap:` | swap এর অবস্থা, আর শেষে `avail Mem` (নতুন কাজের জন্য আসলে কত memory পাওয়া যাবে) |

#### Process টেবিলের কলাম

| কলাম | মানে |
|---|---|
| `PID` | Process ID |
| `USER` | কার process |
| `PR` / `NI` | Priority / nice মান (-20 সবচেয়ে বেশি priority, 19 সবচেয়ে কম) |
| `VIRT` | Virtual memory: process যত memory "ঠিকানা" রিজার্ভ করেছে (সব আসলে ব্যবহার হয় না) |
| `RES` | Resident: আসলে RAM এ যতটুকু আছে। **এটাই আসল memory ব্যবহার বোঝার সবচেয়ে কাজের কলাম** |
| `SHR` | Shared: অন্য process এর সাথে ভাগ করা (যেমন shared library) |
| `S` | State: R (running), S (sleeping), D (uninterruptible, প্রায়ই I/O), T (stopped), Z (zombie), I (idle kernel thread) |
| `%CPU` | এক CPU core এর তুলনায় শতাংশ। **multi-core এ ১০০% এর বেশি হতে পারে** (যেমন ৪ core ব্যবহার করলে ৪০০%) |
| `%MEM` | মোট RAM এর কত % (RES ভিত্তিক) |
| `TIME+` | এ পর্যন্ত মোট কত CPU সময় খেয়েছে |
| `COMMAND` | প্রোগ্রামের নাম |

💡 **টিপস:** `%CPU` এর ১০০% এর বেশি দেখে ভয় পেয়ো না। `top` এ `Shift+I` চাপলে **Irix mode** বন্ধ হয়ে Solaris mode চালু হয়, তখন `%CPU` মোট CPU সংখ্যা দিয়ে ভাগ হয়ে দেখায় (সর্বোচ্চ ১০০%)।

#### `top` এর ভেতরের কীবোর্ড শর্টকাট (চলাকালীন চাপতে হয়)

| কী | কাজ |
|---|---|
| `P` | CPU অনুযায়ী সাজাও (ডিফল্ট) |
| `M` | Memory (`%MEM`) অনুযায়ী সাজাও |
| `T` | `TIME+` অনুযায়ী সাজাও |
| `N` | PID অনুযায়ী সাজাও |
| `<` / `>` | সাজানোর কলাম বাঁয়ে/ডানে সরাও |
| `R` | সাজানোর ক্রম উল্টাও |
| `1` | প্রতিটা CPU core আলাদা লাইনে দেখাও (খুব কাজের!) |
| `t` | CPU summary লাইনের দেখানোর ধরন বদলাও (bar graph) |
| `m` | Memory summary এর দেখানোর ধরন বদলাও |
| `c` | পুরো command line দেখাও (যেমন `php artisan queue:work` বোঝা যাবে) |
| `H` | thread আলাদা করে দেখাও |
| `V` | tree/forest view (কে কার child) |
| `u` | নির্দিষ্ট ইউজারের process ফিল্টার করো |
| `o` / `O` | কলামের মান দিয়ে ফিল্টার, যেমন `%CPU>5.0` |
| `k` | process কে signal পাঠাও (kill) |
| `r` | renice (priority বদলাও) |
| `d` বা `s` | রিফ্রেশ বিরতি বদলাও (ডিফল্ট ৩ সেকেন্ড) |
| `f` | কোন কলাম দেখাবে বেছে নাও |
| `E` / `e` | summary / টেবিলে memory একক বদলাও (KiB → MiB → GiB ...) |
| `x` / `y` | সাজানো কলাম / running task হাইলাইট |
| `z` | রঙিন মোড |
| `W` | বর্তমান সেটিং `~/.config/procps/toprc` এ সেভ করো |
| `h` বা `?` | হেল্প |
| `q` | বের হও |

#### `top` এর কমান্ড-লাইন অপশন

| অপশন | কাজ | উদাহরণ |
|---|---|---|
| `-d SEC` | রিফ্রেশ বিরতি | `top -d 1` |
| `-n N` | N বার রিফ্রেশ করে বের হও | `top -n 1` |
| `-b` | batch mode (স্ক্রিপ্ট বা ফাইলে আউটপুট দেওয়ার জন্য) | `top -b -n 1 > snapshot.txt` |
| `-p PID` | শুধু নির্দিষ্ট PID দেখাও | `top -p 1877,2314` |
| `-u USER` | নির্দিষ্ট ইউজার | `top -u www-data` |
| `-o FIELD` | কোন কলামে সাজাবে | `top -o %MEM` |
| `-H` | thread দেখাও | `top -H -p 1877` |

💡 **টিপস:** সমস্যার মুহূর্তের একটা "ছবি" সংরক্ষণ করতে: `top -b -n 1 -o %CPU | head -20 > /tmp/top-$(date +%F-%H%M).txt`। পরে টিকিটে বা টিমকে দেখাতে কাজে লাগবে।

### `htop`: top এর রঙিন বড় ভাই

`htop` আলাদা প্যাকেজ (`sudo apt install htop` / `sudo dnf install htop`; RHEL এ EPEL লাগতে পারে)। মাউস চলে, scroll করা যায়, রঙিন bar দেখায়।

| কী | কাজ |
|---|---|
| `F1` / `h` | হেল্প |
| `F2` | Setup (কলাম, রঙ, meter) |
| `F3` / `/` | নাম দিয়ে খোঁজা |
| `F4` / `\` | ফিল্টার |
| `F5` / `t` | tree view |
| `F6` / `>` | সাজানোর কলাম বেছে নাও |
| `F7` / `F8` | nice কমাও/বাড়াও (priority বাড়াও/কমাও) |
| `F9` / `k` | kill (signal বেছে নিতে পারবে) |
| `F10` / `q` | বের হও |
| `Space` | process ট্যাগ করো (একসাথে অনেকগুলো kill করতে) |
| `u` | ইউজার দিয়ে ফিল্টার |
| `H` | ইউজার thread লুকাও/দেখাও |
| `K` | kernel thread লুকাও/দেখাও |
| `p` | প্রোগ্রামের পুরো path দেখাও/লুকাও |
| `l` | ওই process এর খোলা ফাইল (`lsof` লাগবে) |
| `s` | ওই process এ `strace` লাগাও |
| `M` / `P` / `T` | memory / CPU / time অনুযায়ী সাজাও |

### `mpstat`: প্রতিটা CPU core আলাদা করে দেখা

`top` সব CPU এর গড় দেখায়। কিন্তু ধরো ৮ core এর মেশিনে একটা single-threaded প্রোগ্রাম একটা core কে ১০০% ব্যস্ত রেখেছে। গড় দেখাবে মাত্র ১২.৫%, তুমি ভাববে সব ঠিক। এই ফাঁদ ধরতে `mpstat`।

`mpstat` আসে **sysstat** প্যাকেজে:

```bash
# Debian/Ubuntu
$ sudo apt install sysstat
# Fedora/RHEL
$ sudo dnf install sysstat
```

Syntax: `mpstat [options] [interval [count]]`

| অপশন | কাজ |
|---|---|
| `-P ALL` | সব CPU আলাদা লাইনে |
| `-P 0,2` | শুধু CPU 0 আর 2 |
| `-u` | CPU ব্যবহার (ডিফল্ট) |
| `-I SUM` / `-I ALL` | interrupt পরিসংখ্যান |
| `interval count` | প্রতি interval সেকেন্ডে, count বার |

```bash
$ mpstat -P ALL 1 1
Linux 6.8.0-45-generic (web01)  11/12/2026  _x86_64_  (4 CPU)

02:40:11 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
02:40:12 PM  all   26.00    0.00    2.25    0.25    0.00    0.25    0.00    0.00    0.00   71.25
02:40:12 PM    0  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
02:40:12 PM    1    2.00    0.00    3.00    1.00    0.00    1.00    0.00    0.00    0.00   93.00
02:40:12 PM    2    1.00    0.00    4.00    0.00    0.00    0.00    0.00    0.00    0.00   95.00
02:40:12 PM    3    1.00    0.00    2.00    0.00    0.00    0.00    0.00    0.00    0.00   97.00
```

দেখো, `all` এ ৭১% idle, কিন্তু CPU 0 পুরো ১০০% `%usr` এ। মানে একটা single-threaded কাজ একটা core কে আটকে রেখেছে। এই ধরনের প্রোগ্রাম আরও CPU দিলেও দ্রুত হবে না।

### `pidstat`: কোন process কী খাচ্ছে, সময়ের সাথে

এটাও sysstat এর অংশ। `top` এর মতো স্ক্রিন মুছে ফেলে না, বরং লাইন ধরে লিখে যায়, তাই পরিবর্তন দেখতে সুবিধা।

| অপশন | কাজ |
|---|---|
| `-u` | CPU (ডিফল্ট) |
| `-r` | memory ও page fault |
| `-d` | disk I/O (প্রতি process কত পড়ছে/লিখছে) |
| `-w` | context switch |
| `-t` | thread সহ |
| `-p PID` | নির্দিষ্ট process |
| `-C NAME` | নাম মিলিয়ে process |

```bash
$ pidstat 1 3
02:41:01 PM   UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
02:41:02 PM    33      2314   97.00    1.00    0.00    0.00   98.00     0  php-fpm8.3
02:41:02 PM   999      1877   12.00    3.00    0.00    1.00   15.00     2  mysqld
```

`%wait` কলাম বলে process কতক্ষণ CPU পাওয়ার জন্য লাইনে অপেক্ষা করেছে। এটা বেশি মানে CPU saturation।

```bash
# কোন process disk এ সবচেয়ে বেশি লিখছে?
$ pidstat -d 1 5
```

---
## ৪৬.৫ Memory মনিটরিং

### `free -h`: memory এর এক নজরে হিসাব

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           7.7Gi       4.9Gi       305Mi       180Mi       2.6Gi       2.6Gi
Swap:          2.0Gi        67Mi       1.9Gi
```

নতুনরা এখানে `free` কলামে মাত্র 305Mi দেখে আতঙ্কিত হয়: "RAM শেষ!"। আসলে না। এটা বোঝার জন্য প্রথমে জানতে হবে লিনাক্স RAM কে কীভাবে দেখে।

#### "Free RAM is wasted RAM": লিনাক্সের দর্শন

লিনাক্স মনে করে খালি পড়ে থাকা RAM হলো অপচয়। তাই যেটুকু RAM কোনো প্রোগ্রাম চাইছে না, সেটুকু দিয়ে kernel disk থেকে পড়া ফাইলগুলো মনে রাখে। একে বলে **page cache**। পরের বার একই ফাইল লাগলে disk এ না গিয়ে RAM থেকেই দিয়ে দেয়, যা কয়েকশো থেকে কয়েক হাজার গুণ দ্রুত।

উপমা: তোমার পড়ার টেবিল। যে বইগুলো আজ পড়েছ, সেগুলো তাকে তুলে না রেখে টেবিলেই খোলা রেখেছ, কারণ আবার লাগতে পারে। কিন্তু নতুন একটা বড় খাতা রাখার জায়গা লাগলে সাথে সাথে পুরনো বই সরিয়ে জায়গা করে দেবে। টেবিল "ভরা" দেখালেও আসলে জায়গা আছে।

```
  মোট RAM (7.7 GiB)
  ┌─────────────────────────────┬──────────────────┬────────┐
  │        used (4.9 GiB)        │ buff/cache 2.6GiB │ free   │
  │  প্রোগ্রামের আসল ডেটা          │ ফাইলের cache       │ 305 MiB│
  │  (সরানো যায় না, swap ছাড়া)    │ (দরকার হলে ছেড়ে দেয়)│ একদম খালি│
  └─────────────────────────────┴──────────────────┴────────┘
                                 └──────── available (~2.6 GiB) ────────┘
                                   (free + ছেড়ে দেওয়া যায় এমন cache এর আনুমানিক যোগফল)
```

#### প্রতিটা কলামের মানে

| কলাম | মানে |
|---|---|
| `total` | মোট ব্যবহারযোগ্য RAM (kernel নিজের জন্য কিছু রেখে দেয়, তাই কেনা RAM এর চেয়ে একটু কম দেখায়) |
| `used` | প্রোগ্রামগুলো আসলে ব্যবহার করছে। হিসাব: `total - free - buff/cache` (procps-ng এর নতুন ভার্সনে `total - available`) |
| `free` | একদম অব্যবহৃত, কিছুই রাখা নেই |
| `shared` | মূলত `tmpfs` (যেমন `/dev/shm`, `/run`) ও shared memory |
| `buff/cache` | **buffers** (block device এর metadata এর cache) + **cache** (ফাইলের page cache ও reclaimable slab) |
| `available` | **নতুন প্রোগ্রাম swap ছাড়াই আনুমানিক কত memory পাবে**। এটাই আসল উত্তর "কত RAM বাকি আছে" |

📝 **মনে রাখো:** "RAM কত বাকি?" এই প্রশ্নের উত্তর সবসময় `available` কলামে, `free` কলামে না।

⚠️ **সতর্কতা:** ইন্টারনেটে অনেক জায়গায় দেখবে `echo 3 > /proc/sys/vm/drop_caches` চালিয়ে "RAM পরিষ্কার" করতে বলছে। Production এ এটা করবে না। এতে cache মুছে যায়, ফলে পরের কিছুক্ষণ সব ফাইল আবার disk থেকে পড়তে হয় এবং সিস্টেম **আরও স্লো** হয়। এটা শুধু benchmark টেস্টের সময় কাজে লাগে।

#### `free` এর অপশন

| অপশন | কাজ |
|---|---|
| `-h` | মানুষের পড়ার মতো একক (Mi, Gi) |
| `-m` / `-g` | MiB / GiB এ |
| `-w` | wide: `buffers` আর `cache` আলাদা কলামে |
| `-t` | নিচে RAM+swap এর মোট লাইন |
| `-s N` | প্রতি N সেকেন্ডে বারবার দেখাও |
| `-c N` | N বার দেখিয়ে থামো (`-s` এর সাথে) |

```bash
$ free -h -s 2 -c 3      # প্রতি ২ সেকেন্ডে, ৩ বার
```

### Swap: RAM এর "এক্সটেনশন কর্ড"

যখন RAM সত্যিই ফুরিয়ে আসে, kernel কম-ব্যবহৃত memory page গুলো disk এর একটা অংশে (swap partition বা swap file) সরিয়ে রাখে। একে **swap out** বলে। আবার দরকার হলে ফেরত আনে (**swap in**)।

- Swap **ব্যবহৃত** থাকা সমস্যা না। হয়তো অনেক আগে কিছু অলস page সরানো হয়েছিল, আর ফেরত আনার দরকার পড়েনি।
- সমস্যা হলো **সক্রিয় swapping**, মানে প্রতি সেকেন্ডে page আসা-যাওয়া। এটা `vmstat` এর `si`/`so` কলামে দেখা যায়। সক্রিয় swapping চলতে থাকলে সিস্টেম প্রায় থেমে যায়, একে বলে **thrashing**।

```bash
$ swapon --show
NAME      TYPE SIZE  USED PRIO
/swap.img file   2G 67.5M   -2

$ cat /proc/sys/vm/swappiness
60
```

`vm.swappiness` (0 থেকে 200, ডিফল্ট সাধারণত 60) বলে kernel কতটা আগ্রহ নিয়ে swap ব্যবহার করবে, cache ছাড়ার তুলনায়। Database সার্ভারে অনেকে এটা কমিয়ে 10 করে। (sysctl দিয়ে পরিবর্তন [অধ্যায় ৪৭](47-kernel-proc-sys.md) এ।)

💡 **টিপস:** Fedora (33+) ডিফল্টে **zram** ব্যবহার করে: RAM এর ভেতরেই একটা compressed swap। disk swap এর চেয়ে অনেক দ্রুত। `swapon --show` এ `/dev/zram0` দেখলে বুঝবে এটাই।

### `/proc/meminfo`: memory এর আসল উৎস

`free`, `top` সবাই আসলে এই ফাইল পড়ে হিসাব করে। এখানে kernel এর সব memory হিসাব আছে।

```bash
$ head -25 /proc/meminfo
MemTotal:        8029388 kB
MemFree:          312876 kB
MemAvailable:    2785432 kB
Buffers:          145220 kB
Cached:          2201344 kB
SwapCached:        12044 kB
Active:          4123456 kB
Inactive:        2654321 kB
...
SwapTotal:       2097148 kB
SwapFree:        2028012 kB
Dirty:              1840 kB
Writeback:             0 kB
AnonPages:       4402112 kB
Mapped:           512344 kB
Shmem:            184320 kB
...
Slab:             420112 kB
SReclaimable:     265432 kB
SUnreclaim:       154680 kB
```

গুরুত্বপূর্ণ লাইনগুলো:

| ফিল্ড | মানে |
|---|---|
| `MemTotal` | মোট ব্যবহারযোগ্য RAM |
| `MemFree` | একদম খালি |
| `MemAvailable` | kernel এর অনুমান: swap ছাড়া নতুন কাজ কত পাবে (`free` এর `available`) |
| `Buffers` / `Cached` | block device buffer / ফাইলের page cache |
| `SwapCached` | যে page swap থেকে ফেরত এসেছে কিন্তু swap এ কপিও রয়ে গেছে |
| `Active` / `Inactive` | সম্প্রতি ব্যবহৃত / অনেকক্ষণ ব্যবহৃত হয়নি (Inactive আগে ছাড়া হয়) |
| `Dirty` | RAM এ বদলানো হয়েছে কিন্তু এখনো disk এ লেখা হয়নি |
| `Writeback` | এই মুহূর্তে disk এ লেখা হচ্ছে |
| `AnonPages` | কোনো ফাইলের সাথে যুক্ত না এমন memory (heap, stack): প্রোগ্রামের "নিজের" ডেটা |
| `Shmem` | shared memory ও tmpfs |
| `Slab` | kernel এর নিজের data structure এর cache |
| `SReclaimable` / `SUnreclaim` | slab এর যে অংশ ছাড়া যায় / যায় না |
| `HugePages_*` | huge page (বড় আকারের memory page, database এ ব্যবহৃত) |

💡 **টিপস:** `SUnreclaim` যদি দিনে দিনে বাড়তে থাকে এবং কোনো process এর `RES` বাড়ছে না, তাহলে সন্দেহ করো kernel বা কোনো driver এ memory leak। `slabtop` কমান্ড দিয়ে দেখো কোন slab বাড়ছে।

### `vmstat`: এক লাইনে পুরো সিস্টেমের নাড়ি

`vmstat` (virtual memory statistics) procps-ng এর অংশ, তাই প্রায় সবখানে থাকে। নাম memory হলেও এটা CPU, process, I/O সব দেখায়।

Syntax: `vmstat [options] [delay [count]]`

```bash
$ vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0  69120 312876 145220 2201344    0    0    12    45  210  380  8  2 89  1  0
 5  0  69120 310112 145220 2201500    0    0     0   120 1890 3200 72  8 14  6  0
 6  2  69120 305400 145224 2201600    0    0   800  2400 2100 3500 70  9 10 11  0
 4  1  69120 301200 145224 2201700    0    0   640  1900 1950 3300 71  8 13  8  0
 5  0  69120 300900 145228 2201800    0    0     0   300 1800 3100 73  7 18  2  0
```

⚠️ **সতর্কতা:** **প্রথম লাইনটা বাদ দাও।** প্রথম লাইন হলো boot থেকে এখন পর্যন্ত গড়, এখনকার অবস্থা না। দ্বিতীয় লাইন থেকে আসল লাইভ ডেটা।

| কলাম | মানে | কখন সমস্যা |
|---|---|---|
| `r` | runnable process (চলছে বা CPU এর অপেক্ষায়) | CPU সংখ্যার চেয়ে স্থায়ীভাবে বেশি হলে CPU saturation |
| `b` | uninterruptible sleep (D state, প্রায়ই I/O অপেক্ষা) | নিয়মিত > 0 হলে I/O সমস্যা |
| `swpd` | ব্যবহৃত swap (KiB) | শুধু সংখ্যাটা চিন্তার না |
| `free` / `buff` / `cache` | memory (KiB) | |
| `si` / `so` | প্রতি সেকেন্ডে swap in / swap out | **নিয়মিত > 0 হলে memory সংকট** |
| `bi` / `bo` | block device থেকে পড়া / লেখা (block/s) | disk এর কাজের পরিমাণ |
| `in` | প্রতি সেকেন্ডে interrupt | |
| `cs` | প্রতি সেকেন্ডে context switch | হঠাৎ অনেক বেশি হলে অনেক thread একে অপরের সাথে কাড়াকাড়ি করছে |
| `us sy id wa st` | CPU ভাগ (৪৬.৪ এর টেবিল দেখো) | |

📝 **মনে রাখো:** procps-ng এর নতুন ভার্সনে শেষে `gu` (guest) কলামও দেখতে পারো। কলাম কম-বেশি হলে ঘাবড়াবে না, হেডার দেখে পড়ো।

| অপশন | কাজ |
|---|---|
| `-S M` | একক MB তে (`-S k`, `-S m`, `-S K`, `-S M`) |
| `-w` | wide (বড় সংখ্যা ঠিকমতো দেখাতে) |
| `-t` | প্রতি লাইনে timestamp |
| `-a` | buff/cache এর বদলে active/inactive |
| `-s` | memory ও event এর মোট পরিসংখ্যান, এক বার |
| `-d` | disk পরিসংখ্যান |

```bash
$ vmstat -w -t -S M 2       # প্রতি ২ সেকেন্ড, MB তে, timestamp সহ
```

### OOM Killer: যখন memory সত্যিই শেষ

#### কেন দরকার

লিনাক্স ডিফল্টে **overcommit** করে: প্রোগ্রাম যত memory চায়, তার চেয়ে বেশি "প্রতিশ্রুতি" দিয়ে দেয়, কারণ বেশিরভাগ প্রোগ্রাম যা চায় তার সবটা কখনো ব্যবহার করে না (উপমা: এয়ারলাইন ১০০ সিটের প্লেনে ১০৫টা টিকিট বিক্রি করে, কারণ কয়েকজন আসবেই না)। কিন্তু কোনোদিন সবাই একসাথে এসে গেলে? RAM আর swap দুটোই শেষ হলে kernel এর আর উপায় থাকে না। তখন সে **OOM (Out Of Memory) killer** চালায়: একটা process বেছে নিয়ে মেরে ফেলে, যাতে পুরো সিস্টেম না আটকায়।

#### কাকে মারবে, কীভাবে ঠিক করে

প্রতিটা process এর একটা **`oom_score`** থাকে (0 থেকে 1000 পর্যন্ত, মূলত কত memory ব্যবহার করছে তার ভিত্তিতে)। সবচেয়ে বেশি স্কোরের process মারা যায়।

```bash
$ cat /proc/1877/oom_score
682
$ cat /proc/1877/oom_score_adj
0
```

`oom_score_adj` (-1000 থেকে +1000) দিয়ে তুমি স্কোর বদলাতে পারো:
- `-1000` = কখনো মারবে না
- `+1000` = সবার আগে মারবে

```bash
# sshd কে OOM থেকে বাঁচাতে (root হিসেবে)
$ echo -500 | sudo tee /proc/$(pgrep -o sshd)/oom_score_adj
```

systemd সার্ভিসে স্থায়ীভাবে করতে unit ফাইলে `OOMScoreAdjust=-500` লেখো ([অধ্যায় ২৬](26-systemd-services.md))।

#### OOM হয়েছে কিনা কীভাবে বুঝবে

হঠাৎ কোনো সার্ভিস "কোনো কারণ ছাড়াই" মারা গেছে? প্রথমে kernel log দেখো:

```bash
$ sudo dmesg -T | grep -i -E 'out of memory|killed process'
[Wed Nov 12 03:14:22 2026] Out of memory: Killed process 1877 (mysqld) total-vm:2456780kB, anon-rss:1258000kB, file-rss:0kB, shmem-rss:0kB, UID:999 pgtables:3200kB oom_score_adj:0

$ sudo journalctl -k --since "today" | grep -i oom
```

💡 **টিপস:** Ubuntu 22.04+ এবং Fedora 34+ এ **systemd-oomd** নামে একটা userspace OOM daemon ডিফল্টে চালু থাকে। এটা kernel এর OOM killer এর **আগেই**, memory চাপ (PSI) দেখে পুরো cgroup (যেমন একটা সার্ভিস বা ইউজার session) মেরে দিতে পারে। এর লগ দেখতে: `journalctl -u systemd-oomd`, অবস্থা দেখতে: `oomctl`।

⚠️ **সতর্কতা:** OOM বারবার হলে সমাধান "OOM killer বন্ধ করা" না। আসল কারণ খোঁজো: কোন process এর memory বাড়ছে (memory leak?), কনফিগারেশনে অতিরিক্ত worker দেওয়া আছে কিনা (যেমন php-fpm এর `pm.max_children` বা MySQL এর `innodb_buffer_pool_size` RAM এর তুলনায় বেশি), নাকি সত্যিই RAM বাড়ানো দরকার।

### কোন process সবচেয়ে বেশি memory খাচ্ছে

```bash
$ ps aux --sort=-rss | head -6
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
mysql       1877 15.2 15.7 2456780 1258000 ?     Ssl  Oct31 120:33 /usr/sbin/mysqld
www-data    2314 25.1  3.1 812340 245120 ?       R    14:00  45:12 php-fpm: pool www
...
```

⚠️ **সতর্কতা:** অনেকগুলো process এর `RSS` যোগ করলে মোট RAM এর চেয়ে বেশি হতে পারে, কারণ shared library প্রতিটা process এর RSS এ গোনা হয়। প্রতিটা process এর "ন্যায্য ভাগ" জানতে **PSS** (Proportional Set Size) দেখো: `smem` টুল (`sudo apt install smem`) বা `/proc/PID/smaps_rollup` ফাইল।

```bash
$ sudo grep -E '^(Rss|Pss)' /proc/1877/smaps_rollup
Rss:             1258000 kB
Pss:             1240112 kB
```

---

## ৪৬.৬ Disk I/O মনিটরিং

### Disk কেন প্রায়ই "অপরাধী"

RAM এর তুলনায় disk অনেক ধীর। আন্দাজ (মোটা দাগে):

| মাধ্যম | একটা random access এ সময় (আনুমানিক) |
|---|---|
| RAM | ~100 ন্যানোসেকেন্ড |
| NVMe SSD | ~১০-১০০ মাইক্রোসেকেন্ড |
| SATA SSD | ~১০০ মাইক্রোসেকেন্ড |
| HDD (ঘূর্ণায়মান) | ~৫-১০ মিলিসেকেন্ড |

অর্থাৎ HDD, RAM এর চেয়ে প্রায় **লক্ষ গুণ** ধীর। তাই I/O তে সমস্যা হলে পুরো সিস্টেম ঝিমিয়ে পড়ে, load average বেড়ে যায় (D state এর কারণে), অথচ CPU অলস বসে থাকে (`wa` বেশি)।

```
  Application ─► VFS ─► Filesystem (ext4/xfs) ─► Page Cache
                                                     │ (cache miss হলে)
                                                     ▼
                                              Block Layer + I/O Scheduler
                                                     │  ◄── এখানে queue জমে (aqu-sz)
                                                     ▼
                                                Device Driver
                                                     │
                                                     ▼
                                                 Disk (sda/nvme0n1)
                         iostat এই স্তরটা মাপে ──────┘
```

### `iostat`: প্রতিটা disk এর অবস্থা

sysstat প্যাকেজের অংশ। সবচেয়ে কাজের রূপ:

```bash
$ iostat -xz 1
```

| অপশন | কাজ |
|---|---|
| `-x` | extended (বিস্তারিত) পরিসংখ্যান: **সবসময় এটা ব্যবহার করো** |
| `-z` | যে device এ কোনো কাজ হয়নি সেগুলো লুকাও |
| `-d` | শুধু device (CPU অংশ বাদ) |
| `-c` | শুধু CPU |
| `-h` | মানুষের পড়ার মতো |
| `-m` / `-k` | MB/s / kB/s এ |
| `-p sda` | sda এবং তার partition গুলো |
| `-t` | timestamp |
| `-N` | LVM/device-mapper এর নাম দেখাও ([অধ্যায় ৩৩](33-lvm-raid.md)) |

```bash
$ iostat -dxz 1 2
Linux 6.8.0-45-generic (web01)  11/12/2026  _x86_64_  (4 CPU)
...
Device      r/s     rkB/s   r_await rareq-sz   w/s     wkB/s   w_await wareq-sz  aqu-sz  %util
nvme0n1    12.00   480.00     0.35    40.00   85.00   9800.00    1.20   115.29    0.11   9.20
sda       210.00  8400.00    38.50    40.00  140.00  5600.00   95.10    40.00   21.40  99.60
```

(আসল আউটপুটে `rrqm/s`, `%rrqm`, `wrqm/s`, discard (`d/s`) ও flush (`f/s`) সম্পর্কিত আরও কলাম থাকে; এখানে বোঝার সুবিধার জন্য কিছু বাদ দেওয়া হয়েছে।)

| কলাম | মানে |
|---|---|
| `r/s`, `w/s` | প্রতি সেকেন্ডে read/write request (IOPS) |
| `rkB/s`, `wkB/s` | প্রতি সেকেন্ডে কত kB পড়া/লেখা (throughput) |
| `r_await`, `w_await` | একটা request এর গড় সময় (ms), **queue তে অপেক্ষা + আসল কাজ মিলিয়ে**। ইউজার এই latency ই টের পায় |
| `rareq-sz`, `wareq-sz` | গড় request এর আকার (kB) |
| `aqu-sz` | গড় queue এর দৈর্ঘ্য (আগের ভার্সনে `avgqu-sz`)। **Saturation এর মাপ** |
| `%util` | device কত % সময় ব্যস্ত ছিল |

উপরের উদাহরণে `sda` স্পষ্টভাবে সমস্যায়: `%util` প্রায় ১০০%, `aqu-sz` ২১, write এ ৯৫ ms অপেক্ষা। `nvme0n1` আরামে আছে।

⚠️ **সতর্কতা:** `%util` ১০০% মানেই disk "পুরো ভরা" না, বিশেষ করে SSD/NVMe আর RAID এর ক্ষেত্রে। এগুলো একসাথে অনেক request সমান্তরালে সামলাতে পারে। `%util` শুধু বলে "অন্তত একটা request চলছিল এতটা সময়"। তাই SSD তে `await` আর `aqu-sz` বেশি বিশ্বাসযোগ্য সংকেত। HDD (যেটা একবারে একটাই কাজ পারে) এর ক্ষেত্রে `%util` ভালো সংকেত।

💡 **টিপস:** সাধারণ নিয়ম (আন্দাজ): SSD তে `await` কয়েক ms এর বেশি, বা HDD তে ২০-৩০ ms এর বেশি স্থায়ীভাবে থাকলে তদন্ত করো।

### `iotop`: কোন process disk কাঁপাচ্ছে

`iostat` বলে কোন **disk** ব্যস্ত, কিন্তু কে ব্যস্ত রাখছে বলে না। সেজন্য `iotop`। এটা root হিসেবে চালাতে হয়।

```bash
$ sudo apt install iotop        # Ubuntu তে iotop-c নামে C ভার্সনও আছে
$ sudo dnf install iotop        # Fedora/RHEL
$ sudo iotop -o
Total DISK READ:        8.20 M/s | Total DISK WRITE:        5.47 M/s
Current DISK READ:      8.20 M/s | Current DISK WRITE:      9.10 M/s
    TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
   1877 be/4 mysql       7.90 M/s    4.80 M/s  0.00 %  88.12 %  mysqld
   4410 be/4 root      300.00 K/s  600.00 K/s  0.00 %  10.05 %  rsync -a /var/www /backup
```

| অপশন | কাজ |
|---|---|
| `-o` | শুধু যারা এখন I/O করছে |
| `-P` | thread না, process দেখাও |
| `-a` | মুহূর্তের হার না, শুরু থেকে জমা মোট |
| `-b` | batch mode (স্ক্রিপ্ট/লগের জন্য) |
| `-n N` | N বার দেখিয়ে থামো |
| `-d SEC` | রিফ্রেশ বিরতি |
| `-u USER` / `-p PID` | ফিল্টার |

| চলাকালীন কী | কাজ |
|---|---|
| `o` | শুধু active দেখানো চালু/বন্ধ |
| `p` | process/thread মোড বদল |
| `a` | accumulated মোড বদল |
| `←` / `→` | সাজানোর কলাম বদল |
| `r` | সাজানোর ক্রম উল্টাও |
| `q` | বের হও |

📝 **মনে রাখো:** `IO>` কলাম বলে ওই thread কত % সময় I/O এর অপেক্ষায় কাটিয়েছে। কিছু নতুন kernel এ এই হিসাবের জন্য `kernel.task_delayacct` sysctl চালু করতে হয় (`sudo sysctl kernel.task_delayacct=1`); iotop নিজেই প্রয়োজনে সতর্ক করে দেয়।

`iotop` না থাকলে বিকল্প: `pidstat -d 1` (sysstat)।

---

## ৪৬.৭ Network মনিটরিং

নেটওয়ার্কের মূলনীতি [অধ্যায় ৩৬](36-networking-basics.md) এ আর `ss`, `ping`, `traceroute`, `tcpdump` এর মতো ট্রাবলশুটিং কমান্ড [অধ্যায় ৩৭](37-network-commands.md) এ আছে। এখানে শুধু **কতটা bandwidth কে খাচ্ছে** সেটা দেখব।

### `iftop`: কোন কানেকশনে কত ট্রাফিক

`iftop` একটা interface এর উপর দিয়ে যাওয়া প্রতিটা কানেকশন (দুই host এর জোড়া) এর bandwidth লাইভ দেখায়। root লাগে।

```bash
$ sudo apt install iftop     # Fedora/RHEL: sudo dnf install iftop (RHEL এ EPEL)
$ sudo iftop -i eth0 -nP
                 12.5Mb          25.0Mb          37.5Mb          50.0Mb
└────────────────┴───────────────┴───────────────┴───────────────┴──────
10.0.0.5:443        => 203.0.113.45:51822      8.20Mb  6.10Mb  5.90Mb
                    <=                          210Kb   180Kb   170Kb
10.0.0.5:22         => 198.51.100.7:60211        12Kb    10Kb    9Kb
                    <=                          3.1Kb   2.8Kb   2.5Kb
──────────────────────────────────────────────────────────────────────
TX:             cum:   45.2MB   peak:  9.10Mb  rates:  8.21Mb  6.11Mb  5.91Mb
RX:                     1.6MB          250Kb            213Kb   183Kb   173Kb
TOTAL:                 46.8MB          9.35Mb          8.42Mb  6.29Mb  6.08Mb
```

ডানের তিনটা কলাম হলো গত **২ সেকেন্ড, ১০ সেকেন্ড, ৪০ সেকেন্ডের** গড় হার।

| অপশন | কাজ |
|---|---|
| `-i IFACE` | কোন interface (নাম জানতে `ip link`) |
| `-n` | IP কে hostname এ রূপান্তর (DNS lookup) করো না (দ্রুত, এবং নিজেই ট্রাফিক তৈরি করে না) |
| `-N` | port নম্বরকে service নামে বদলিও না |
| `-P` | port দেখাও |
| `-B` | bit এর বদলে byte এ দেখাও |
| `-f FILTER` | pcap ফিল্টার, যেমন `-f 'port 443'` |
| `-t` | text mode (ncurses ছাড়া), `-s N` এর সাথে স্ক্রিপ্টে |

| চলাকালীন কী | কাজ |
|---|---|
| `n` | DNS resolve চালু/বন্ধ |
| `p` | port দেখানো চালু/বন্ধ |
| `P` | pause |
| `t` | এক লাইন/দুই লাইন/শুধু পাঠানো/শুধু আসা ঘুরিয়ে দেখাও |
| `1` / `2` / `3` | ২s / ১০s / ৪০s কলাম দিয়ে সাজাও |
| `<` / `>` | source / destination নাম দিয়ে সাজাও |
| `l` | ফিল্টার করে দেখাও |
| `q` | বের হও |

### `nethogs`: কোন **process** কত bandwidth খাচ্ছে

`iftop` বলে কোন কানেকশন, `nethogs` বলে কোন প্রোগ্রাম। "কে ইন্টারনেট খেয়ে ফেলছে?" প্রশ্নের সরাসরি উত্তর।

```bash
$ sudo apt install nethogs     # Fedora: sudo dnf install nethogs
$ sudo nethogs eth0
NetHogs version 0.8.7

    PID USER     PROGRAM                          DEV        SENT      RECEIVED
   4410 root     rsync                            eth0     5210.4     12.1 KB/sec
   2314 www-data php-fpm: pool www                eth0      820.2     95.3 KB/sec
   1102 root     sshd: admin@pts/0                eth0        1.2      0.3 KB/sec
      ? root     unknown TCP                                  0.0      0.0 KB/sec
```

| অপশন / কী | কাজ |
|---|---|
| `-d SEC` | রিফ্রেশ বিরতি |
| `-t` | trace mode (লাইন ধরে আউটপুট) |
| `-v N` | একক: 0=KB/s, 1=মোট KB, 2=মোট B, 3=মোট MB |
| চলাকালীন `m` | একক ঘোরাও (KB/s → KB → B → MB) |
| `s` / `r` | sent / received দিয়ে সাজাও |
| `q` | বের হও |

### দ্রুত interface পরিসংখ্যান

```bash
$ ip -s link show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    RX:  bytes  packets errors dropped  missed   mcast
    9876543210  8123456      0      12       0       0
    TX:  bytes  packets errors dropped carrier collsns
    5432109876  6012345      0       0       0       0
```

`errors` আর `dropped` বাড়তে থাকলে এটা USE এর **E** (Errors)। কেবল, NIC driver, বা buffer এর সমস্যা হতে পারে।

---

## ৪৬.৮ sysstat ও `sar`: পারফরম্যান্সের "ব্ল্যাক বক্স রেকর্ডার"

`top`, `iostat` সবই এই মুহূর্তের ছবি দেখায়। কিন্তু সমস্যা প্রায়ই রাত ৩টায় হয়, আর সকালে তুমি এসে দেখো সব স্বাভাবিক। তখন কী করবে?

উপমা: বিমানের **black box** সারাক্ষণ সব রেকর্ড করে, দুর্ঘটনার পর সেটা দেখে কারণ বোঝা যায়। লিনাক্সে সেই black box হলো **sysstat** এর `sadc` (system activity data collector), যা নিয়মিত বিরতিতে (ডিফল্ট ১০ মিনিট) সব পরিসংখ্যান ফাইলে সংরক্ষণ করে। আর `sar` (system activity reporter) সেই ফাইল পড়ে দেখায়।

```
   systemd timer (sysstat-collect.timer, প্রতি ১০ মিনিট)
           │
           ▼
      sa1 ─► sadc ─► /var/log/sysstat/saDD   (Debian/Ubuntu)
                     /var/log/sa/saDD        (Fedora/RHEL)
                          │      (DD = মাসের তারিখ, যেমন sa12)
                          ▼
                         sar ─► তোমার স্ক্রিনে রিপোর্ট
```

### চালু করা

```bash
# Debian/Ubuntu
$ sudo apt install sysstat
$ sudo sed -i 's/^ENABLED="false"/ENABLED="true"/' /etc/default/sysstat
$ sudo systemctl enable --now sysstat

# Fedora/RHEL
$ sudo dnf install sysstat
$ sudo systemctl enable --now sysstat
```

📝 **মনে রাখো:** Debian/Ubuntu তে `/etc/default/sysstat` ফাইলের `ENABLED="true"` না করলে ডেটা জমা হয় না (Ubuntu র নতুন রিলিজে প্যাকেজ ইনস্টলের সময় এটা ঠিক করা থাকতে পারে, তবু একবার দেখে নাও)। কতদিনের ডেটা রাখবে তা ঠিক হয় `/etc/sysstat/sysstat` (Debian) বা `/etc/sysconfig/sysstat` (RHEL) ফাইলের `HISTORY` দিয়ে।

### `sar` এর গুরুত্বপূর্ণ অপশন

| অপশন | কী দেখায় |
|---|---|
| `-u` | CPU (ডিফল্ট); `-u ALL` সব ভাগ |
| `-P ALL` | প্রতিটা CPU আলাদা |
| `-q` | load average ও run queue (`runq-sz`, `ldavg-1`, ...) |
| `-r` | memory ব্যবহার |
| `-S` | swap ব্যবহার |
| `-W` | swapping এর হার (`pswpin/s`, `pswpout/s`) |
| `-B` | paging পরিসংখ্যান |
| `-b` | মোট I/O হার |
| `-d` | প্রতি block device (`-p` দিলে সুন্দর নাম) |
| `-n DEV` | প্রতি network interface এর ট্রাফিক |
| `-n EDEV` | network error |
| `-n TCP,ETCP` | TCP কানেকশন ও retransmit |
| `-n SOCK` | socket সংখ্যা |
| `-w` | context switch ও process তৈরির হার |
| `-A` | সব কিছু |
| `-f FILE` | নির্দিষ্ট ডেটা ফাইল পড়ো |
| `-s HH:MM:SS` / `-e HH:MM:SS` | শুরু / শেষ সময় |
| `interval count` | লাইভ মোড (ফাইল না পড়ে এখনই মাপো) |

### উদাহরণ

```bash
# লাইভ: প্রতি সেকেন্ডে ৩ বার network ট্রাফিক
$ sar -n DEV 1 3
02:50:01 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
02:50:02 PM        lo     10.00     10.00      1.20      1.20      0.00      0.00      0.00      0.00
02:50:02 PM      eth0   1520.00   2810.00     95.30   5210.40      0.00      0.00      0.00      4.27
```

```bash
# আজকের সংরক্ষিত CPU ডেটা, রাত ২টা থেকে ৪টা
$ sar -u -s 02:00:00 -e 04:00:00
02:00:01 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
02:10:01 AM     all      5.12      0.00      1.20      0.40      0.00     93.28
02:20:01 AM     all      4.98      0.00      1.18      0.35      0.00     93.49
...
03:10:01 AM     all     12.40      0.00      6.80     62.10      0.00     18.70
03:20:01 AM     all     11.95      0.00      6.50     65.40      0.00     16.15
...
```

রাত ৩টার পর `%iowait` হঠাৎ ৬০% এর উপরে। এখন সন্দেহ: ওই সময় কোনো backup বা cron job চলে কিনা ([অধ্যায় ২৯](29-scheduling-cron.md), [অধ্যায় ৩৫](35-backup-rsync.md))।

```bash
# গতকালের (মনে করো ১১ তারিখ) memory ডেটা
$ sar -r -f /var/log/sysstat/sa11          # Ubuntu/Debian
$ sar -r -f /var/log/sa/sa11               # RHEL/Fedora

# load ইতিহাস
$ sar -q
02:10:01 PM   runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
02:20:01 PM         2      1234      0.52      0.78      1.05         0
```

💡 **টিপস:** নতুন সার্ভার সেটআপ করলেই প্রথম দিন sysstat চালু করে দাও। সমস্যার দিন ডেটা না থাকলে আফসোস করা ছাড়া কিছু করার থাকে না।

---
## ৪৬.৯ PSI (Pressure Stall Information): আধুনিক "চাপ মাপার যন্ত্র"

Kernel 4.20 থেকে লিনাক্সে একটা দারুণ জিনিস আছে: **PSI**। Load average এর একটা দুর্বলতা হলো এটা CPU আর I/O মিশিয়ে ফেলে। PSI সরাসরি বলে: **গত ১০/৬০/৩০০ সেকেন্ডে কত % সময় task গুলো CPU, memory বা I/O এর অভাবে আটকে ছিল।** এটা খাঁটি **Saturation** এর মাপ।

```bash
$ cat /proc/pressure/cpu
some avg10=45.20 avg60=30.10 avg300=12.05 total=987654321
full avg10=0.00 avg60=0.00 avg300=0.00 total=0

$ cat /proc/pressure/memory
some avg10=0.00 avg60=0.00 avg300=0.00 total=12345
full avg10=0.00 avg60=0.00 avg300=0.00 total=6789

$ cat /proc/pressure/io
some avg10=12.50 avg60=8.30 avg300=3.10 total=55443322
full avg10=9.80 avg60=6.10 avg300=2.20 total=44332211
```

| শব্দ | মানে |
|---|---|
| `some` | অন্তত **একটা** task ওই resource এর অভাবে আটকে ছিল এমন সময়ের % |
| `full` | **সব** non-idle task একসাথে আটকে ছিল (কেউ কাজ করতে পারেনি) এমন সময়ের %। সবচেয়ে খারাপ অবস্থা |
| `avg10/60/300` | গত ১০, ৬০, ৩০০ সেকেন্ডের গড় |

উপরের উদাহরণে CPU তে ৪৫% সময় কেউ না কেউ অপেক্ষায় ছিল, আর I/O তে ৯.৮% সময় সবাই আটকে ছিল। Memory চাপ শূন্য। ব্যস, এক নজরে কোথায় সমস্যা বোঝা গেল। systemd-oomd এই memory PSI দেখেই সিদ্ধান্ত নেয়।

---

## ৪৬.১০ `lsof`: "কে কী খুলে রেখেছে?"

লিনাক্সে প্রায় সবকিছুই ফাইল: সাধারণ ফাইল, ডিরেক্টরি, device, pipe, এমনকি network socket। `lsof` (list open files) বলে কোন process কোন ফাইল/socket খুলে রেখেছে। ট্রাবলশুটিং এ এটা জাদুর কাঠি।

```bash
$ sudo apt install lsof    # সাধারণত আগে থেকেই থাকে; Fedora/RHEL: sudo dnf install lsof
```

⚠️ **সতর্কতা:** অন্যের process দেখতে `sudo` লাগবে, নইলে শুধু নিজেরগুলো দেখবে।

| অপশন | কাজ | উদাহরণ |
|---|---|---|
| `-i` | network socket | `lsof -i` |
| `-i :PORT` | নির্দিষ্ট port | `lsof -i :80` |
| `-i TCP` / `-i UDP` | প্রোটোকল | `lsof -i TCP:443` |
| `-i @HOST` | নির্দিষ্ট host এর সাথে কানেকশন | `lsof -i @10.0.0.9` |
| `-p PID` | নির্দিষ্ট process এর সব খোলা ফাইল | `lsof -p 1877` |
| `-u USER` | ইউজারের খোলা ফাইল | `lsof -u www-data` |
| `-c NAME` | যে command এর নাম NAME দিয়ে শুরু | `lsof -c nginx` |
| `+D DIR` | ডিরেক্টরির ভেতরে (recursive) খোলা ফাইল | `lsof +D /var/log` |
| `+L1` | **মুছে ফেলা হয়েছে কিন্তু এখনো খোলা** ফাইল (link count < 1) | `lsof +L1` |
| `-n` / `-P` | IP কে নাম / port কে service নামে বদলিও না (দ্রুত) | `lsof -nP -i` |
| `-t` | শুধু PID দেখাও (স্ক্রিপ্টে কাজে লাগে) | `kill $(lsof -t -i :8080)` |
| `-a` | একাধিক শর্ত AND করো (ডিফল্টে OR) | `lsof -a -u www-data -i` |
| `-r SEC` | প্রতি SEC সেকেন্ডে আবার দেখাও | `lsof -r 2 -i :443` |

```bash
$ sudo lsof -nP -i :80
COMMAND  PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   1020     root    6u  IPv4  23456      0t0  TCP *:80 (LISTEN)
nginx   1021 www-data    6u  IPv4  23456      0t0  TCP *:80 (LISTEN)
nginx   1021 www-data   12u  IPv4  98765      0t0  TCP 10.0.0.5:80->203.0.113.45:51822 (ESTABLISHED)
```

| কলাম | মানে |
|---|---|
| `FD` | File descriptor: `cwd` (current dir), `txt` (প্রোগ্রামের নিজের কোড), `mem` (memory-mapped ফাইল), সংখ্যা + `r`/`w`/`u` (read/write/দুটোই) |
| `TYPE` | `REG` (সাধারণ ফাইল), `DIR`, `CHR` (character device), `IPv4`/`IPv6`, `unix`, `FIFO` |
| `NAME` | ফাইলের path বা কানেকশন |

💡 **টিপস:** "ডিস্ক umount হচ্ছে না, `target is busy` বলছে"? `sudo lsof +D /mnt/usb` বা `sudo fuser -vm /mnt/usb` দিয়ে দেখো কে ওটা ব্যবহার করছে।

💡 **টিপস:** একটা process কতগুলো file descriptor খুলে রেখেছে: `ls /proc/1877/fd | wc -l`। সীমা দেখতে: `grep 'open files' /proc/1877/limits`। "Too many open files" error এ এটা কাজে লাগে।

---

## ৪৬.১১ `strace` ও `ltrace`: প্রোগ্রামের ভেতরে উঁকি

### System call কী, আবার মনে করি

একটা প্রোগ্রাম নিজে সরাসরি disk, network বা স্ক্রিনে হাত দিতে পারে না। সব কাজ তাকে kernel এর কাছে "আবেদন" করে করাতে হয়। এই আবেদনগুলোকে বলে **system call** (syscall), যেমন `openat()` (ফাইল খোলো), `read()`, `write()`, `connect()`। ([অধ্যায় ২](02-linux-architecture.md) এ user space আর kernel space এর পার্থক্য দেখেছ।)

```
    ┌────────────────────────────┐
    │     তোমার প্রোগ্রাম          │  User space
    │  (php, python, nginx...)   │
    │          │                 │
    │   libc (printf, fopen...)  │ ◄── ltrace এই স্তরের ফাংশন কল দেখায়
    └──────────┼─────────────────┘
               │  system call (openat, read, write, connect...)
               │                ◄── strace এই সীমানাটা দেখায়
    ┌──────────▼─────────────────┐
    │          Kernel            │  Kernel space
    └────────────────────────────┘
```

উপমা: প্রোগ্রাম হলো অফিসের একজন কর্মচারী, kernel হলো ম্যানেজার। কর্মচারী প্রতিটা কাজের জন্য ম্যানেজারকে চিঠি লেখে। `strace` হলো সেই পিয়ন যে প্রতিটা চিঠির কপি তোমাকে দেখায়: কী চাওয়া হলো আর ম্যানেজার কী উত্তর দিল।

### `strace`

```bash
$ sudo apt install strace     # Fedora/RHEL: sudo dnf install strace
```

| অপশন | কাজ |
|---|---|
| `strace CMD` | নতুন একটা কমান্ড চালিয়ে trace করো |
| `-p PID` | চলমান process এ যুক্ত হও (root/একই ইউজার লাগে); বের হতে `Ctrl+C` |
| `-f` | child process ও thread ও trace করো |
| `-e trace=openat,read` | শুধু নির্দিষ্ট syscall |
| `-e trace=%file` / `%network` / `%process` | syscall এর গ্রুপ (ফাইল/নেটওয়ার্ক/process সম্পর্কিত) |
| `-c` | প্রতিটা trace না দেখিয়ে শেষে একটা **সারাংশ টেবিল** দাও |
| `-T` | প্রতিটা syscall এ কত সময় লাগল |
| `-t` / `-tt` | প্রতিটা লাইনে সময় / মাইক্রোসেকেন্ড সহ সময় |
| `-o FILE` | আউটপুট ফাইলে |
| `-s N` | string কত দৈর্ঘ্য পর্যন্ত দেখাবে (ডিফল্ট 32) |
| `-y` | file descriptor নম্বরের পাশে path দেখাও |

#### উদাহরণ ১: প্রোগ্রাম কোন config ফাইল খুঁজছে?

```bash
$ strace -e trace=openat cat /etc/hostname
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
web01
+++ exited with 0 +++
```

প্রতিটা লাইনে syscall, তার argument, আর `=` এর পরে ফলাফল। ফলাফল `-1` হলে সাথে error নাম থাকে:

```bash
$ strace -e trace=openat myapp 2>&1 | grep ENOENT
openat(AT_FDCWD, "/etc/myapp/config.yml", O_RDONLY) = -1 ENOENT (No such file or directory)
```

এখন জানলে, প্রোগ্রাম `/etc/myapp/config.yml` খুঁজছে কিন্তু পাচ্ছে না। কোনো ডকুমেন্টেশন না পড়েই সমস্যা ধরা পড়ল। একইভাবে `EACCES` মানে permission denied।

#### উদাহরণ ২: আটকে থাকা process কী করছে?

```bash
$ sudo strace -p 2314 -tt -T
14:55:01.120345 poll([{fd=9, events=POLLIN}], 1, -1
```

process টা `poll()` এ অপেক্ষা করছে, কোনো সময়সীমা (`-1`) ছাড়াই। `-y` দিয়ে দেখো fd 9 কী: হয়তো একটা database বা external API এর socket যেটা উত্তর দিচ্ছে না।

#### উদাহরণ ৩: সময় কোথায় যাচ্ছে (সারাংশ)

```bash
$ sudo strace -c -f -p 2314
^C
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 62.10    0.412300         206      2001           read
 20.40    0.135400          45      3005       512 openat
 10.20    0.067700          22      3002           newfstatat
  ...
------ ----------- ----------- --------- --------- ----------------
100.00    0.663900                  9120       530 total
```

`openat` এ ৫১২টা error! হয়তো প্রোগ্রাম অনেক অস্তিত্বহীন ফাইল খুঁজছে (যেমন ভুল include path)।

⚠️ **সতর্কতা:** `strace` লক্ষ্য process কে **অনেক ধীর** করে দেয় (কখনো কয়েকগুণ), কারণ প্রতিটা syscall এ process থামিয়ে তথ্য পড়তে হয়। ব্যস্ত production সার্ভারে অল্প সময়ের জন্য ব্যবহার করো।

### `ltrace`

`ltrace` একই কাজ করে, কিন্তু **library function call** (যেমন `malloc`, `strlen`, `printf`) দেখায়, syscall না।

```bash
$ sudo apt install ltrace       # Fedora: sudo dnf install ltrace
$ ltrace -c ls > /dev/null
% time     seconds  usecs/call     calls      function
------ ----------- ----------- --------- --------------------
 25.10    0.001200          12       100 strlen
 ...
```

| অপশন | কাজ |
|---|---|
| `-p PID` | চলমান process |
| `-c` | সারাংশ |
| `-S` | library call এর সাথে syscall ও দেখাও |
| `-e malloc+free` | নির্দিষ্ট ফাংশন |
| `-f` | child process |

📝 **মনে রাখো:** আধুনিক distro তে অনেক binary বিশেষ নিরাপত্তা অপশন দিয়ে compile করা থাকে, ফলে `ltrace` সবসময় সব কল দেখাতে পারে না। দৈনন্দিন ট্রাবলশুটিং এ `strace` অনেক বেশি কাজের।

---

## ৪৬.১২ `perf` পরিচিতি: CPU কোন ফাংশনে সময় দিচ্ছে

`top` বলে "php-fpm ৯৮% CPU খাচ্ছে"। কিন্তু প্রোগ্রামের **কোন ফাংশনে**? এর উত্তর দেয় `perf`, লিনাক্স kernel এর সাথে আসা অফিসিয়াল profiling টুল। এটা CPU এর hardware counter আর kernel এর tracepoint ব্যবহার করে খুব কম overhead এ মাপে।

উপমা: `strace` হলো প্রতিটা চিঠি পড়া পিয়ন (ধীর)। `perf` হলো এমন একজন, যে প্রতি সেকেন্ডে কয়েকশো বার এক ঝলক উঁকি দিয়ে দেখে "এই মুহূর্তে কর্মচারী কী করছে"। হাজারটা ঝলক মিলিয়ে বোঝা যায় কোন কাজে সবচেয়ে বেশি সময় যায় (**sampling**)।

ইনস্টল:

```bash
# Ubuntu (চলমান kernel এর সাথে মিলিয়ে)
$ sudo apt install linux-tools-common linux-tools-$(uname -r)
# Debian 12
$ sudo apt install linux-perf
# Fedora/RHEL
$ sudo dnf install perf
```

| কমান্ড | কাজ |
|---|---|
| `sudo perf top` | লাইভ: পুরো সিস্টেমে কোন ফাংশন সবচেয়ে বেশি CPU খাচ্ছে |
| `sudo perf top -p PID` | নির্দিষ্ট process |
| `perf stat CMD` | একটা কমান্ডের মোট পরিসংখ্যান (cycles, instructions, cache miss...) |
| `sudo perf record -g -p PID -- sleep 30` | ৩০ সেকেন্ড ধরে call stack সহ রেকর্ড করো (`perf.data` ফাইলে) |
| `sudo perf report` | রেকর্ড করা ডেটা বিশ্লেষণ (interactive) |
| `perf list` | কোন কোন event মাপা যায় |

```bash
$ perf stat gzip -k bigfile.log

 Performance counter stats for 'gzip -k bigfile.log':

          2,104.55 msec task-clock                #    0.998 CPUs utilized
                12      context-switches          #    5.702 /sec
     8,120,334,221      cycles                    #    3.858 GHz
    11,402,118,003      instructions              #    1.40  insn per cycle
       ...
       2.108331021 seconds time elapsed
```

```bash
$ sudo perf top
Samples: 42K of event 'cycles:P', 4000 Hz, Event count (approx.): 9876543210
Overhead  Shared Object            Symbol
  18.42%  php-fpm8.3               [.] execute_ex
   9.10%  libc.so.6                [.] __memmove_avx_unaligned_erms
   6.75%  [kernel]                 [k] copy_user_enhanced_fast_string
```

`[.]` মানে user space ফাংশন, `[k]` মানে kernel ফাংশন।

📝 **মনে রাখো:** `kernel.perf_event_paranoid` sysctl ঠিক করে সাধারণ ইউজার কতটা perf চালাতে পারবে। অনেক distro তে এটা কড়া সেট করা থাকে, তাই সিস্টেম-জুড়ে profiling এর জন্য `sudo` দিয়ে চালাও। VM এ hardware counter (cycles, instructions) সবসময় পাওয়া যায় না, তখন `perf top` নিজে software event (`cpu-clock`) ব্যবহার করে।

💡 **টিপস:** `perf record -g` এর ডেটা থেকে **Flame Graph** (Brendan Gregg এর তৈরি) বানানো যায়: একটা ছবি যেখানে চওড়া বাক্স মানে বেশি CPU সময়। গভীরে যেতে চাইলে পরবর্তী ধাপ হলো **eBPF** টুল (`bpftrace`, bcc-tools এর `execsnoop`, `opensnoop`, `biolatency`), যেগুলো production এ খুব কম খরচে kernel এর ভেতরের ঘটনা মাপে।

---

## ৪৬.১৩ সব-এক-জায়গায় ড্যাশবোর্ড: `btop` ও `glances`

### `btop`

সুন্দর, রঙিন, মাউস-চালিত মনিটর: CPU (প্রতি core এর গ্রাফ সহ), memory, disk, network, process সব এক স্ক্রিনে।

```bash
$ sudo apt install btop        # Ubuntu 22.04+, Debian 12
$ sudo dnf install btop        # Fedora (RHEL এ EPEL)
$ btop
```

| কী | কাজ |
|---|---|
| `Esc` / `m` | মেনু |
| `o` | অপশন/সেটিংস |
| `h` | হেল্প |
| `1` / `2` / `3` / `4` | CPU / memory / network / process বক্স দেখাও-লুকাও |
| `←` / `→` | process সাজানোর কলাম বদল |
| `r` | সাজানোর ক্রম উল্টাও |
| `e` | tree view |
| `f` | process ফিল্টার |
| `↑` / `↓` + `Enter` | process বেছে বিস্তারিত দেখো |
| `k` / `t` / `s` | kill (SIGKILL) / terminate (SIGTERM) / signal বেছে পাঠাও |
| `+` / `-` | রিফ্রেশ বিরতি বাড়াও/কমাও |
| `q` | বের হও |

### `glances`

Python এ লেখা, এক স্ক্রিনে প্রায় সবকিছু, সাথে নিজে থেকে **সতর্কবার্তা** (সবুজ/নীল/বেগুনি/লাল রঙে) দেখায়। Web বা API মোডেও চালানো যায়।

```bash
$ sudo apt install glances     # Fedora: sudo dnf install glances
$ glances
$ glances -w                   # web UI, ডিফল্ট পোর্ট 61208 (ব্রাউজারে http://IP:61208)
```

| কী | কাজ |
|---|---|
| `a` | স্বয়ংক্রিয় সাজানো (যেটা সমস্যা করছে সেটা উপরে) |
| `c` / `m` / `i` / `p` / `t` | CPU / memory / I/O / নাম / সময় অনুযায়ী সাজাও |
| `1` | প্রতি core আলাদা |
| `d` / `n` / `s` | disk / network / sensor অংশ লুকাও-দেখাও |
| `f` | filesystem অংশ লুকাও-দেখাও |
| `h` | হেল্প |
| `q` / `Esc` | বের হও |

⚠️ **সতর্কতা:** `glances -w` এ ডিফল্টে কোনো পাসওয়ার্ড নেই। ইন্টারনেটের সামনে খোলা রাখবে না; দরকার হলে `--password` ব্যবহার করো এবং ফায়ারওয়াল দিয়ে আটকাও ([অধ্যায় ৩৯](39-firewall.md))।

### কোন টুল কখন: তুলনা

| টুল | সবচেয়ে ভালো কাজ | প্রায় সবখানে আছে? |
|---|---|---|
| `top` | দ্রুত এক নজর, যেকোনো সার্ভারে | হ্যাঁ |
| `htop` | ইন্টারঅ্যাকটিভ process ম্যানেজমেন্ট | প্রায়ই |
| `btop` | সুন্দর সার্বিক চিত্র | না, ইনস্টল লাগে |
| `glances` | সার্বিক চিত্র + সতর্কবার্তা + web | না |
| `vmstat`, `mpstat`, `iostat`, `pidstat` | নির্দিষ্ট resource এর নির্ভুল সংখ্যা | sysstat ইনস্টল লাগে (vmstat ছাড়া) |
| `sar` | অতীতের ডেটা | sysstat চালু থাকতে হবে |

---

## ৪৬.১৪ "সার্ভার স্লো!" ধাপে ধাপে ডায়াগনোসিস চেকলিস্ট

এটাই এই অধ্যায়ের সবচেয়ে গুরুত্বপূর্ণ অংশ। Brendan Gregg এর বিখ্যাত **"৬০ সেকেন্ডের বিশ্লেষণ"** এর উপর ভিত্তি করে সাজানো। লগইন করেই এই ক্রমে চালাও:

```
 ┌─────────────────────────────────────────────────────────────────────┐
 │ ধাপ ০: প্রশ্ন করো — "স্লো" মানে কী? কবে থেকে? সবার জন্য নাকি একজনের? │
 │        সম্প্রতি কী বদলেছে (deploy, update, config)?                   │
 └──────────────────────────────┬──────────────────────────────────────┘
                                ▼
         ১. uptime ──► load বাড়ছে না কমছে? CPU সংখ্যার তুলনায় কত?
                                ▼
         ২. dmesg -T | tail ──► OOM? disk error? hardware error?
                                ▼
         ৩. vmstat 1 ──► r (CPU লাইন), b (I/O), si/so (swap), wa, st
                                ▼
         ৪. mpstat -P ALL 1 ──► একটা core কি একা আটকে?
                                ▼
         ৫. pidstat 1 ──► কোন process CPU খাচ্ছে?
                                ▼
         ৬. iostat -xz 1 ──► কোন disk ধীর? await, aqu-sz, %util
                                ▼
         ৭. free -h ──► available কত? swap চলছে?
                                ▼
         ৮. sar -n DEV 1 ──► network কি লিঙ্কের সীমায়?
                                ▼
         ৯. sar -n TCP,ETCP 1 ──► retransmit (নেটওয়ার্ক সমস্যা)?
                                ▼
         ১০. top / htop ──► সব মিলিয়ে নিশ্চিত হও
                                ▼
         ১১. df -h; df -i ──► disk/inode ভরা?
                                ▼
         সন্দেহজনক process পেলে ─► strace -p / lsof -p / perf top -p / লগ দেখো
```

| ধাপ | কমান্ড | কী খুঁজবে |
|---|---|---|
| ১ | `uptime` | load এর ধারা (বাড়ছে/কমছে), `nproc` এর সাথে তুলনা |
| ২ | `sudo dmesg -T \| tail -30` | "Out of memory", "I/O error", "segfault", "link down" |
| ৩ | `vmstat 1 5` | `r` > CPU সংখ্যা? `b` > 0? `si`/`so` > 0? `wa` বা `st` বেশি? |
| ৪ | `mpstat -P ALL 1 3` | অসম CPU ব্যবহার (single-thread bottleneck) |
| ৫ | `pidstat 1 3` | কোন process CPU খাচ্ছে, `%wait` কত |
| ৬ | `iostat -xz 1 3` | `await`, `aqu-sz`, `%util` |
| ৭ | `free -h` | `available` কম? swap ব্যবহার বাড়ছে? |
| ৮ | `sar -n DEV 1 3` | `rxkB/s`/`txkB/s` লিঙ্ক স্পিডের কাছাকাছি? |
| ৯ | `sar -n TCP,ETCP 1 3` | `retrans/s` বেশি = নেটওয়ার্ক সমস্যা |
| ১০ | `top` | উপরের সব সন্দেহ এক জায়গায় মিলিয়ে দেখো |
| ১১ | `df -h; df -i` | ১০০% ভরা filesystem |
| ১২ | `cat /proc/pressure/{cpu,memory,io}` | কোন resource এ চাপ |
| ১৩ | `journalctl -p err --since "1 hour ago"` | সাম্প্রতিক error ([অধ্যায় ২৮](28-logs.md)) |

### ফলাফল দেখে সিদ্ধান্ত

| যা দেখলে | সম্ভাব্য কারণ | পরের পদক্ষেপ |
|---|---|---|
| load বেশি, `us` বেশি, `id` কম | অ্যাপ্লিকেশন CPU খাচ্ছে | `pidstat`/`top` এ process ধরো, `perf top -p` দিয়ে কোন ফাংশন |
| load বেশি, `wa` বেশি, CPU অলস | Disk bottleneck | `iostat -x` এ disk, `iotop` এ process |
| `sy` অস্বাভাবিক বেশি | অতিরিক্ত syscall/context switch | `strace -c -p`, `vmstat` এর `cs` |
| `st` বেশি | Cloud host এ CPU চুরি হচ্ছে | ক্লাউড প্রোভাইডারের সাথে কথা বলো বা instance বদলাও |
| `si`/`so` নিয়মিত > 0, `available` কম | Memory সংকট | `ps aux --sort=-rss`, memory leak খোঁজো |
| `dmesg` এ OOM | Memory শেষ হয়ে process মারা গেছে | ওপরের OOM অংশ দেখো |
| সব resource স্বাভাবিক, তবু স্লো | বাইরের নির্ভরতা (database, API, DNS) বা lock | `strace -p` এ কোথায় অপেক্ষা করছে দেখো, অ্যাপের লগ দেখো |

💡 **টিপস:** "সব স্বাভাবিক, তবু স্লো" খুব সাধারণ ঘটনা। তখন প্রায়ই সমস্যা অন্য সার্ভারে (ধীর database query, বাইরের API timeout) বা DNS এ। `strace -tt -T -p PID` দিয়ে দেখো কোন `connect()` বা `poll()` এ দীর্ঘ সময় যাচ্ছে।

⚠️ **সতর্কতা:** তদন্তের আগে reboot করবে না। Reboot করলে সমস্যা হয়তো সাময়িক চলে যাবে, কিন্তু সব প্রমাণ (চলমান process, memory এর অবস্থা) মুছে যাবে, আর সমস্যা আবার ফিরে আসবে। আগে উপরের কমান্ডগুলোর আউটপুট সংরক্ষণ করো, তারপর সিদ্ধান্ত নাও।

---

## ৪৬.১৫ সাধারণ সমস্যা ও সমাধান

### সমস্যা ১: Disk full ("No space left on device")

**লক্ষণ:** ফাইল লেখা যাচ্ছে না, database বন্ধ হয়ে গেছে, লগইন করতেও সমস্যা।

**ধাপ ১: কোন filesystem ভরা?**

```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        40G   40G     0 100% /
/dev/sda1       1.1G  6.1M  1.1G   1% /boot/efi
```

**ধাপ ২: কোথায় জায়গা গেল?** (`-x` দিয়ে অন্য filesystem এ যাওয়া বন্ধ রাখো)

```bash
$ sudo du -xh --max-depth=1 / 2>/dev/null | sort -h | tail -8
1.2G    /usr
2.1G    /home
33G     /var
40G     /

$ sudo du -xh --max-depth=1 /var | sort -h | tail -5
...
31G     /var/log
```

💡 **টিপস:** `ncdu` (`sudo apt install ncdu`) একটা ইন্টারঅ্যাকটিভ টুল, তীর চিহ্ন দিয়ে ডিরেক্টরিতে ঢুকে দেখা যায় কে কত জায়গা নিচ্ছে: `sudo ncdu -x /`। (`du`, `find -size` এর বিস্তারিত [অধ্যায় ১২](12-file-khoja.md) ও [অধ্যায় ৩২](32-disk-partition-mount.md) এ।)

**ধাপ ৩: সাধারণ অপরাধী ও সমাধান**

```bash
# বড় ফাইল খোঁজো
$ sudo find / -xdev -type f -size +500M -exec ls -lh {} \; 2>/dev/null

# systemd journal কত জায়গা নিচ্ছে, কমাও
$ journalctl --disk-usage
$ sudo journalctl --vacuum-size=500M

# পুরনো প্যাকেজ cache
$ sudo apt clean            # Debian/Ubuntu
$ sudo dnf clean all        # Fedora/RHEL

# Docker থাকলে
$ docker system df
$ docker system prune       # অব্যবহৃত image/container মোছে, সাবধানে!
```

**ধাপ ৪: ফাঁদ ১ — মুছে দিলাম, তবু জায়গা খালি হলো না!**

তুমি একটা ১০ GB লগ ফাইল `rm` করলে, কিন্তু `df` এখনো ১০০% দেখাচ্ছে। কেন? কারণ কোনো process (যেমন nginx) ফাইলটা এখনো খুলে রেখেছে। লিনাক্সে ফাইল আসলে মোছা হয় তখনই যখন তার **সব নাম (link) মুছে যায় এবং কেউ আর খুলে রাখেনি** ([অধ্যায় ১১](11-links-inode.md))।

```bash
$ sudo lsof +L1
COMMAND  PID     USER   FD   TYPE DEVICE    SIZE/OFF NLINK  NODE NAME
nginx   1021 www-data    5w   REG    8,2 10737418240     0 13021 /var/log/nginx/access.log (deleted)
```

সমাধান: সার্ভিস reload/restart করো (`sudo systemctl reload nginx`), ফাইল ছেড়ে দেবে। ভবিষ্যতে চলমান লগ ফাইল খালি করতে `rm` না করে truncate করো:

```bash
$ sudo truncate -s 0 /var/log/nginx/access.log
# বা
$ : | sudo tee /var/log/nginx/access.log
```

আর স্থায়ী সমাধান হলো `logrotate` ঠিকমতো কনফিগার করা ([অধ্যায় ২৮](28-logs.md))।

**ধাপ ৫: ফাঁদ ২ — জায়গা আছে, তবু "No space left"!**

```bash
$ df -i
Filesystem      Inodes   IUsed IFree IUse% Mounted on
/dev/sda2      2621440 2621440     0  100% /
```

**inode শেষ।** প্রতিটা ফাইলের জন্য একটা inode লাগে। লক্ষ লক্ষ ছোট ফাইল (যেমন PHP session ফাইল, mail queue, cache) inode শেষ করে দিতে পারে। কোন ডিরেক্টরিতে সবচেয়ে বেশি ফাইল:

```bash
$ sudo du --inodes -x / 2>/dev/null | sort -n | tail -5
```

**ফাঁদ ৩:** ext4 ডিফল্টে ৫% জায়গা root এর জন্য রিজার্ভ রাখে, তাই সাধারণ ইউজার `Avail` 0 দেখলেও root হয়তো লিখতে পারে। বড় ডেটা disk এ এটা কমানো যায়: `sudo tune2fs -m 1 /dev/sdb1` (root filesystem এ করবে না)।

### সমস্যা ২: High load average

```bash
$ uptime
 15:10:01 up 30 days, ... load average: 24.50, 18.20, 9.10
$ nproc
4
```

load ২৪, কিন্তু CPU মাত্র ৪টা। এখন বোঝো load টা **CPU** নাকি **I/O** থেকে:

```bash
# State অনুযায়ী process গোনো
$ ps -eo state= | sort | uniq -c
    230 S
     22 R
     19 D
      2 I

# D state (I/O তে আটকে) process গুলো কারা?
$ ps -eo pid,state,wchan:32,cmd | awk '$2=="D"'
  4410 D  io_schedule                  rsync -a /var/www /backup
  ...
```

- **R বেশি** → CPU সমস্যা: `top`/`pidstat` এ process খোঁজো, `perf top` দিয়ে কারণ। সমাধান: অপ্রয়োজনীয় process থামাও, `renice` দিয়ে priority কমাও ([অধ্যায় ২৪](24-process-management.md)), কোড optimize, বা CPU বাড়াও।
- **D বেশি** → I/O সমস্যা: `iostat -x` আর `iotop`। সমাধান: ভারী কাজ (backup, re-index) ব্যস্ত সময় থেকে সরাও, `ionice -c3 CMD` দিয়ে idle priority তে চালাও, দ্রুত disk (SSD) নাও। NFS mount সাড়া না দিলেও process D state এ আটকে load বাড়ায়।

```bash
# backup কে কম priority তে চালানো (CPU ও I/O দুটোতেই)
$ nice -n 19 ionice -c3 rsync -a /var/www /backup
```

### সমস্যা ৩: Port already in use ("Address already in use")

```bash
$ sudo systemctl start nginx
Job for nginx.service failed ...
$ sudo journalctl -u nginx -n 5
... bind() to 0.0.0.0:80 failed (98: Address already in use)
```

**কে port 80 দখল করে আছে?**

```bash
$ sudo ss -tulpn | grep ':80 '
tcp   LISTEN 0  511   0.0.0.0:80   0.0.0.0:*   users:(("apache2",pid=889,fd=4))

# বিকল্প উপায়
$ sudo lsof -nP -i :80
$ sudo fuser -v 80/tcp
```

| `ss` অপশন | মানে |
|---|---|
| `-t` / `-u` | TCP / UDP |
| `-l` | শুধু listening |
| `-p` | process দেখাও (root লাগে) |
| `-n` | নাম resolve করো না |

**সমাধান:**
- ভুল করে অন্য সার্ভিস চালু আছে (যেমন apache2 আর nginx দুটোই): `sudo systemctl disable --now apache2`।
- পুরনো process আটকে আছে: আগে স্বাভাবিকভাবে বন্ধ করো (`kill PID`), না হলে শেষ অস্ত্র `kill -9 PID`। `sudo fuser -k 80/tcp` port ব্যবহারকারী সব process মেরে দেয়, সাবধানে।
- নিজের অ্যাপ অন্য port এ চালাও।

📝 **মনে রাখো:** ১০২৪ এর নিচের port (যেমন ৮০, ৪৪৩) এ সাধারণ ইউজার bind করতে পারে না: সেক্ষেত্রে error হবে "Permission denied", "Address already in use" না। এছাড়া প্রোগ্রাম বন্ধ করার পরপরই আবার চালালে কখনো পুরনো কানেকশন `TIME-WAIT` অবস্থায় থাকে; ভালোভাবে লেখা সার্ভার `SO_REUSEADDR` ব্যবহার করে এই সমস্যা এড়ায়।

### সমস্যা ৪: Permission denied

এটা দেখতে সহজ কিন্তু কারণ অনেক রকম হতে পারে। ধাপে ধাপে চেক করো:

```bash
$ cat /var/www/app/storage/logs/app.log
cat: /var/www/app/storage/logs/app.log: Permission denied
```

**ধাপ ১: আমি কে?**

```bash
$ id
uid=1000(rahim) gid=1000(rahim) groups=1000(rahim),27(sudo)
```

**ধাপ ২: পুরো path এর প্রতিটা ধাপের permission** (একটা ডিরেক্টরিতে `x` না থাকলেই ভেতরে ঢোকা যায় না)

```bash
$ namei -l /var/www/app/storage/logs/app.log
f: /var/www/app/storage/logs/app.log
drwxr-xr-x root     root     /
drwxr-xr-x root     root     var
drwxr-xr-x root     root     www
drwxr-x--- www-data www-data app          ◄── এখানে আটকে গেছে! others এর কোনো অনুমতি নেই
drwxrwxr-x www-data www-data storage
drwxrwxr-x www-data www-data logs
-rw-r--r-- www-data www-data app.log
```

**ধাপ ৩: সাধারণ কারণ ও সমাধান**

| কারণ | কীভাবে বুঝবে | সমাধান |
|---|---|---|
| সাধারণ permission/ownership | `ls -l`, `namei -l` | `chmod`/`chown` বা ইউজারকে গ্রুপে যোগ ([অধ্যায় ১৬](16-permissions.md)) |
| গ্রুপে যোগ করলাম, তবু হচ্ছে না | `id` এ নতুন গ্রুপ নেই | logout/login করো, বা `newgrp GROUP` |
| ACL | `ls -l` এ `+` চিহ্ন, `getfacl FILE` | `setfacl` ([অধ্যায় ১৭](17-special-permissions-acl.md)) |
| immutable attribute | `lsattr FILE` এ `i` | `sudo chattr -i FILE` (কেন দেওয়া হয়েছিল আগে জানো) |
| script চলছে না | `ls -l` এ `x` নেই | `chmod +x script.sh` |
| `noexec` mount | `findmnt -no OPTIONS /tmp` এ `noexec` | অন্য জায়গা থেকে চালাও, বা `bash script.sh` |
| read-only filesystem | `findmnt` এ `ro`, error এ "Read-only file system" | `dmesg` দেখো (disk error এর কারণে kernel ro করে দিতে পারে) |
| **SELinux** (Fedora/RHEL) | permission ঠিক আছে তবু denied; `getenforce` = Enforcing; `sudo ausearch -m avc -ts recent` | `restorecon -Rv PATH`, সঠিক context/boolean ([অধ্যায় ৪৫](45-security-hardening.md)) |
| **AppArmor** (Ubuntu/Debian) | `sudo journalctl -k \| grep -i apparmor` এ `DENIED` | profile ঠিক করো ([অধ্যায় ৪৫](45-security-hardening.md)) |
| ১০২৪ এর নিচে port bind | সাধারণ ইউজার, port < 1024 | root/systemd দিয়ে চালাও বা `CAP_NET_BIND_SERVICE` |

⚠️ **সতর্কতা:** "Permission denied" এর সমাধান হিসেবে কখনো `chmod -R 777` করবে না। এটা সমস্যা "সমাধান" করে নিরাপত্তায় বিশাল ফুটো তৈরি করে। আর SELinux error এ `setenforce 0` স্থায়ী সমাধান না; কারণ খুঁজে ঠিক করো।

💡 **টিপস:** কোন ফাইলে আসলে permission আটকাচ্ছে বুঝতে না পারলে: `strace -f -e trace=%file CMD 2>&1 | grep EACCES`।

---

## হাতে-কলমে অনুশীলন

> এই ল্যাবগুলো নিজের টেস্ট মেশিন বা VM এ করো, production সার্ভারে না। শুরুতে প্রয়োজনীয় টুল ইনস্টল করো:
>
> ```bash
> $ sudo apt install sysstat iotop htop btop lsof strace stress-ng   # Ubuntu/Debian
> $ sudo dnf install sysstat iotop htop btop lsof strace stress-ng   # Fedora
> ```

### ল্যাব ১: CPU চাপ তৈরি করে ধরা

```bash
# টার্মিনাল ১: ২টা CPU core এ ৬০ সেকেন্ড চাপ দাও
$ stress-ng --cpu 2 --timeout 60s

# টার্মিনাল ২:
$ uptime                 # load কয়েক সেকেন্ড পরে বাড়তে থাকবে (১ মিনিটের মান)
$ mpstat -P ALL 1 3      # কোন ২টা core ১০০%?
$ pidstat 1 3            # stress-ng process দেখো
$ top                    # 1 চাপো (প্রতি core), P চাপো, তারপর q
```

প্রশ্ন: `top` এ stress-ng এর `%CPU` কত দেখাচ্ছে? `Shift+I` চাপলে কী বদলায়?

### ল্যাব ২: Memory বোঝা

```bash
$ free -h                          # available নোট করো
$ cat /usr/bin/* > /dev/null 2>&1   # অনেক ফাইল পড়ো (cache ভরবে)
$ free -h                          # buff/cache বেড়েছে, available প্রায় একই আছে কি?
$ grep -E 'MemAvailable|^Cached|Dirty' /proc/meminfo
$ vmstat 1 5                       # si/so কলাম দেখো
```

প্রশ্ন: `free` কমে গেলেও `available` কেন প্রায় একই রইল?

### ল্যাব ৩: Disk I/O চাপ

```bash
# টার্মিনাল ১: I/O চাপ (৬০ সেকেন্ড)
$ stress-ng --hdd 1 --timeout 60s

# টার্মিনাল ২:
$ iostat -dxz 1 5        # কোন disk এ w/s, w_await, %util বাড়ছে?
$ sudo iotop -o          # কোন process? (q দিয়ে বের হও)
$ vmstat 1 5             # wa আর b কলাম
$ cat /proc/pressure/io
```

### ল্যাব ৪: মুছে ফেলা কিন্তু খোলা ফাইল

```bash
$ dd if=/dev/zero of=/tmp/big.bin bs=1M count=200     # ২০০ MB ফাইল
$ df -h /tmp
$ tail -f /tmp/big.bin > /dev/null &                   # ফাইলটা খুলে রাখো
$ rm /tmp/big.bin
$ df -h /tmp                                           # জায়গা ফেরত আসেনি!
$ lsof +L1 | grep big.bin                              # ধরা পড়ল
$ kill %1                                              # tail বন্ধ করো
$ df -h /tmp                                           # এখন জায়গা খালি
```

(নোট: কিছু সিস্টেমে `/tmp` হলো `tmpfs`, মানে RAM; তখন `df -h /tmp` ছাড়াও `free -h` এর `shared` এ পরিবর্তন দেখবে।)

### ল্যাব ৫: Port in use

```bash
$ python3 -m http.server 8080 &         # port 8080 দখল করো
$ python3 -m http.server 8080           # দ্বিতীয়বার → "Address already in use"
$ ss -tlpn | grep 8080                  # কে দখল করেছে?
$ lsof -nP -i :8080
$ kill $(lsof -t -i :8080)              # মুক্ত করো
```

### ল্যাব ৬: strace দিয়ে গোয়েন্দাগিরি

```bash
$ strace -e trace=openat ls /nonexistent 2>&1 | tail -3
$ strace -c ls -R /usr/share/doc > /dev/null      # সারাংশে কোন syscall সবচেয়ে বেশি?
$ strace -f -e trace=%network curl -s https://example.com -o /dev/null 2>&1 | grep connect
```

প্রশ্ন: `curl` কোন IP তে `connect()` করল?

### ল্যাব ৭: sar চালু করে ইতিহাস দেখা

```bash
$ sudo systemctl enable --now sysstat
$ systemctl list-timers | grep sysstat
# অন্তত ২০-৩০ মিনিট অপেক্ষা করো, তারপর:
$ sar -u
$ sar -r
$ sar -q
```

### ল্যাব ৮: পুরো চেকলিস্ট একটা স্ক্রিপ্টে

নিচের স্ক্রিপ্টটা `~/perfcheck.sh` নামে সেভ করো, `chmod +x` দাও, চালাও। (শেল স্ক্রিপ্টিং [অধ্যায় ৪১](41-shell-scripting-1.md) থেকে।)

```bash
#!/usr/bin/env bash
# perfcheck.sh — সার্ভার স্লো হলে প্রথম ৬০ সেকেন্ডের তথ্য সংগ্রহ
out="/tmp/perfcheck-$(date +%F-%H%M%S).txt"
{
  echo "== uptime ==";           uptime; echo "CPUs: $(nproc)"
  echo "== dmesg (last 20) ==";  sudo dmesg -T | tail -20
  echo "== vmstat ==";           vmstat 1 5
  echo "== mpstat ==";           mpstat -P ALL 1 2
  echo "== pidstat ==";          pidstat 1 2
  echo "== iostat ==";           iostat -xz 1 2
  echo "== free ==";             free -h
  echo "== sar net ==";          sar -n DEV 1 2
  echo "== df ==";               df -h; df -i
  echo "== pressure ==";         grep . /proc/pressure/*
  echo "== top ==";              top -b -n 1 | head -25
} > "$out" 2>&1
echo "রিপোর্ট সেভ হয়েছে: $out"
```

---

## সারসংক্ষেপ

### মূল কমান্ড

| কমান্ড | কাজ | USE এর কোন অংশ |
|---|---|---|
| `uptime` | uptime ও load average | CPU+I/O saturation |
| `cat /proc/loadavg` | load এর কাঁচা ডেটা | saturation |
| `top` / `htop` / `btop` | লাইভ সার্বিক চিত্র | সব |
| `mpstat -P ALL 1` | প্রতি core এর ব্যবহার | CPU utilization |
| `pidstat 1` / `-d` / `-r` | প্রতি process CPU / disk / memory | কে দায়ী |
| `free -h` | memory সারাংশ (`available` দেখো!) | memory utilization |
| `cat /proc/meminfo` | memory এর বিস্তারিত | memory |
| `vmstat 1` | r, b, si/so, wa এক লাইনে | CPU/mem/I/O saturation |
| `swapon --show` | swap ডিভাইস | memory |
| `dmesg -T \| grep -i oom` | OOM ঘটনা | memory errors |
| `iostat -xz 1` | disk latency/queue/util | disk U+S |
| `iotop -o` | কোন process disk ব্যবহার করছে | disk |
| `iftop -i IFACE -nP` | কানেকশন ভিত্তিক bandwidth | network |
| `nethogs IFACE` | process ভিত্তিক bandwidth | network |
| `ip -s link` | interface errors/drops | network errors |
| `sar -u/-r/-q/-d/-n DEV` | ইতিহাস ও লাইভ পরিসংখ্যান | সব |
| `cat /proc/pressure/*` | PSI চাপ | saturation |
| `lsof -i :PORT` / `+L1` / `-p PID` | খোলা socket/ফাইল | ট্রাবলশুটিং |
| `ss -tulpn` | listening port ও process | port সমস্যা |
| `strace -p PID -c` / `-e trace=` | system call দেখা | গভীর তদন্ত |
| `ltrace` | library call দেখা | গভীর তদন্ত |
| `perf top` / `perf record -g` / `perf report` | CPU profiling | CPU কোথায় যাচ্ছে |
| `df -h` / `df -i` / `du -xh` / `ncdu` | disk জায়গা ও inode | disk space |
| `namei -l PATH` | path এর প্রতি ধাপের permission | permission denied |
| `nice` / `ionice -c3` | কম priority তে চালানো | load কমানো |

### `top` এর জরুরি শর্টকাট

| কী | কাজ | কী | কাজ |
|---|---|---|---|
| `P` | CPU দিয়ে সাজাও | `1` | প্রতি core |
| `M` | memory দিয়ে সাজাও | `c` | পুরো command |
| `T` | সময় দিয়ে সাজাও | `H` | thread |
| `k` | kill | `V` | tree |
| `r` | renice | `u` | ইউজার ফিল্টার |
| `Shift+I` | Irix mode টগল | `W` | সেটিং সেভ |
| `h` | হেল্প | `q` | বের হও |

### `htop` / `btop` / `iotop` / `iftop` দ্রুত মনে রাখো

| টুল | সার্চ/ফিল্টার | সাজানো | kill | tree | বের হওয়া |
|---|---|---|---|---|---|
| `htop` | `F3` / `F4` | `F6` | `F9` | `F5` | `F10`/`q` |
| `btop` | `f` | `←` `→` | `k` | `e` | `q` |
| `iotop` | — | `←` `→` | — | — | `q` |
| `iftop` | `l` | `1` `2` `3` | — | — | `q` |

---

## নিজেকে যাচাই করো

1. USE পদ্ধতির তিনটা অক্ষর কী বোঝায়? কোনটা সবচেয়ে বিপদের সংকেত এবং কেন?
2. একটা ৪ core সার্ভারে `load average: 6.10, 5.80, 2.30` দেখলে কী বুঝবে?
3. লিনাক্সে load average বেশি, অথচ `top` এ CPU ৮০% idle। এটা কীভাবে সম্ভব?
4. `free -h` এ `free` 200Mi কিন্তু `available` 5Gi। সিস্টেমে কি memory সংকট?
5. `vmstat` এর কোন কলাম দেখে বুঝবে সিস্টেম thrashing করছে?
6. একটা সার্ভিস হঠাৎ মারা গেছে, লগে কিছু নেই। OOM killer দায়ী কিনা কীভাবে নিশ্চিত হবে?
7. SSD তে `%util` ১০০% মানেই কি disk এর ক্ষমতা শেষ? কোন কলাম বেশি নির্ভরযোগ্য?
8. ১০ GB লগ ফাইল মুছে দেওয়ার পরও `df` জায়গা দেখাচ্ছে না। কারণ ও সমাধান কী?
9. `df -h` এ ৪০% খালি, তবু "No space left on device" আসছে। কী চেক করবে?
10. গত রাত ৩টায় সার্ভার স্লো ছিল, এখন সব ঠিক। কোন টুল দিয়ে তদন্ত করবে, আর সেটা কাজ করার পূর্বশর্ত কী?

<details><summary>উত্তর দেখো</summary>

1. **U**tilization (resource কত % সময় ব্যস্ত), **S**aturation (কত কাজ লাইনে অপেক্ষায়), **E**rrors (কতগুলো error)। Saturation সবচেয়ে সরাসরি বিপদের সংকেত, কারণ এর মানে কাজ অপেক্ষা করছে, অর্থাৎ ইউজার দেরি টের পাচ্ছে; শুধু উচ্চ utilization মানেই সমস্যা না। (Errors উপেক্ষা করা চলবে না, কারণ সেগুলো প্রায়ই নীরবে পারফরম্যান্স নষ্ট করে।)
2. গত ১ মিনিটে গড়ে ~৬টা task চলছিল বা অপেক্ষায় ছিল, ৪ core এর চেয়ে বেশি, তাই saturation আছে। ১৫ মিনিটের মান (2.30) কম, অর্থাৎ load সম্প্রতি **বাড়ছে**। সমস্যা এই মাত্র শুরু, এখনই তদন্ত করো।
3. লিনাক্সের load average এ D state (uninterruptible sleep, সাধারণত I/O এর অপেক্ষা) এর task ও গোনা হয়। ধীর disk বা আটকে যাওয়া NFS এর কারণে অনেক process D state এ থাকলে CPU অলস থাকলেও load বেশি হয়। `vmstat` এর `b`, `wa` আর `iostat -x` দেখো।
4. না। লিনাক্স খালি RAM দিয়ে ফাইল cache রাখে, যা দরকার হলে ছেড়ে দেয়। `available` (5Gi) বলে নতুন কাজের জন্য swap ছাড়াই আনুমানিক ৫ GiB পাওয়া যাবে। এটাই আসল মাপ।
5. `si` আর `so` (swap in/out প্রতি সেকেন্ড) নিয়মিত শূন্যের উপরে থাকলে; সাথে সাধারণত `wa` আর `b` বাড়ে।
6. `sudo dmesg -T | grep -i -E 'out of memory|killed process'` বা `journalctl -k | grep -i oom`। systemd-oomd থাকলে `journalctl -u systemd-oomd` ও দেখো। সার্ভিসের ক্ষেত্রে `systemctl status SERVICE` এ `oom-kill` result দেখাতে পারে।
7. না। SSD/NVMe এবং RAID একসাথে অনেক request সমান্তরালে সামলাতে পারে; `%util` শুধু বলে অন্তত একটা request চলছিল। `r_await`/`w_await` (latency) আর `aqu-sz` (queue) বেশি নির্ভরযোগ্য।
8. কোনো process ফাইলটা এখনো খুলে রেখেছে, তাই inode আর data block মুক্ত হয়নি। `sudo lsof +L1` দিয়ে process খুঁজে সার্ভিসটা reload/restart করো। ভবিষ্যতে চলমান লগ `rm` না করে `truncate -s 0` করো এবং logrotate ঠিক রাখো।
9. `df -i` দিয়ে inode। লক্ষ লক্ষ ছোট ফাইল inode শেষ করে দিতে পারে। `du --inodes -x /` দিয়ে কোন ডিরেক্টরিতে ফাইল বেশি খোঁজো। এছাড়া read-only mount বা ext4 এর root-reserved জায়গাও কারণ হতে পারে।
10. `sar` (sysstat), যেমন `sar -u -s 02:30:00 -e 03:30:00`, `sar -q`, `sar -r`, `sar -d`, দরকার হলে `-f /var/log/sysstat/saDD` (RHEL এ `/var/log/sa/saDD`)। পূর্বশর্ত: sysstat ইনস্টল ও ডেটা সংগ্রহ আগে থেকেই চালু থাকতে হবে (`systemctl enable --now sysstat`, Debian/Ubuntu তে `ENABLED="true"`)। সাথে ওই সময়ের `journalctl --since/--until` লগও দেখো।

</details>

---

[← আগের অধ্যায়](45-security-hardening.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](47-kernel-proc-sys.md)

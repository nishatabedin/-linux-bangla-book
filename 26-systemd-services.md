# অধ্যায় ২৬: systemd ও সার্ভিস ম্যানেজমেন্ট

## এই অধ্যায়ে যা শিখবে

- init system কী, আর কম্পিউটার চালু হওয়ার পর প্রথম process কেন এত গুরুত্বপূর্ণ
- পুরোনো SysV init থেকে লিনাক্স কেন systemd-তে সরে এল
- **unit** কী: service, socket, timer, target, mount সহ সব ধরনের unit
- `systemctl` দিয়ে সার্ভিস start, stop, restart, reload, status, enable, disable, mask করা
- target ও পুরোনো runlevel-এর মধ্যে সম্পর্ক
- unit ফাইল কোথায় থাকে (`/usr/lib/systemd/system` বনাম `/etc/systemd/system`) এবং ফাইলের গঠন
- নিজের একটি service unit লিখে চালানো
- `systemctl edit` দিয়ে override (drop-in) তৈরি করা
- dependency: `Wants`, `Requires`, `After`, `Before` ইত্যাদি
- `systemd-analyze` দিয়ে বুট সময় মাপা ও unit যাচাই করা
- `hostnamectl`, `timedatectl`, `localectl` দিয়ে সিস্টেম সেটিংস বদলানো
- user service (`systemctl --user`) ও নিরাপত্তা সেটিংস (sandboxing)

---

## ২৬.১ init system কী?

কম্পিউটার চালু করলে প্রথমে firmware (BIOS/UEFI), তারপর bootloader (GRUB), তারপর **kernel** লোড হয়। kernel হার্ডওয়্যার চিনে নেওয়ার পর একটা কাজই বাকি থাকে: user space-এ **প্রথম process** চালু করা। এই প্রথম process-এর PID (Process ID) সবসময় **১**। একেই বলে **init** (initialization, অর্থাৎ শুরু করার দায়িত্বে থাকা প্রোগ্রাম)।

বুটের পুরো ধাপগুলো বিস্তারিত দেখবে [অধ্যায় ২৭](27-boot-process.md)-এ। এখানে শুধু PID 1-এর কাজ নিয়ে কথা বলব।

```
  পাওয়ার বাটন
      │
      ▼
  UEFI/BIOS ──► GRUB ──► Linux kernel
                              │
                              │ kernel প্রথম user-space প্রোগ্রাম চালায়
                              ▼
                    ┌──────────────────┐
                    │  PID 1 = systemd │  ◄── init system
                    └────────┬─────────┘
          ┌──────────┬───────┼─────────┬──────────────┐
          ▼          ▼       ▼         ▼              ▼
     journald     udevd   sshd    NetworkManager   gdm (লগইন স্ক্রিন)
```

### init-এর দায়িত্ব

1. **সার্ভিস চালু করা:** নেটওয়ার্ক, SSH, ওয়েব সার্ভার, লগিং, লগইন স্ক্রিন।
2. **সঠিক ক্রম রক্ষা করা:** নেটওয়ার্ক ওঠার আগে ওয়েব সার্ভার চালু হলে ঝামেলা হতে পারে।
3. **সার্ভিসের দেখাশোনা করা:** কোনো সার্ভিস crash করলে দরকার হলে আবার চালু করা।
4. **orphan process দত্তক নেওয়া:** কোনো process-এর parent মারা গেলে সেই "এতিম" process-টা PID 1-এর অধীনে চলে আসে। init তখন সেটার exit status সংগ্রহ (reap) করে, যাতে zombie process জমে না যায় (process নিয়ে বিস্তারিত [অধ্যায় ২৪](24-process-management.md)-এ)।
5. **শাটডাউন:** বন্ধ করার সময় সব সার্ভিস ঠিক ক্রমে থামানো, ফাইলসিস্টেম unmount করা।

> 💡 **উপমা:** init হলো একটা অফিসের ম্যানেজার। সকালে অফিস খুললে সে ঠিক করে কে আগে আসবে (দারোয়ান দরজা খুলবে, তারপর রিসেপশনিস্ট, তারপর বাকিরা), কেউ অসুস্থ হয়ে চলে গেলে বদলি পাঠায়, আর সন্ধ্যায় সবাইকে সঠিক ক্রমে ছুটি দিয়ে দরজায় তালা দেয়।

নিজের মেশিনে দেখো PID 1 কে:

```
$ ps -p 1 -o pid,comm
    PID COMMAND
      1 systemd

$ ls -l /sbin/init
lrwxrwxrwx 1 root root 20 Apr  8  2024 /sbin/init -> /lib/systemd/systemd
```

`/sbin/init` আসলে systemd-এর দিকে একটা symlink (symlink নিয়ে [অধ্যায় ১১](11-links-inode.md))।

> 📝 **মনে রাখো:** PID 1 মারা গেলে kernel panic হয়। তাই init হতে হয় অত্যন্ত নির্ভরযোগ্য।

---

## ২৬.২ SysV init থেকে systemd: কেন এই পরিবর্তন?

### পুরোনো দিন: SysV init

১৯৮০-এর দশকের UNIX System V থেকে আসা **SysV init** বহু বছর লিনাক্সে চলেছে। এর পদ্ধতি ছিল সহজ:

- প্রতিটি সার্ভিসের জন্য `/etc/init.d/` ফোল্ডারে একটা **shell script** থাকত, যেমন `/etc/init.d/apache2`।
- **runlevel** (০ থেকে ৬ পর্যন্ত সংখ্যা) দিয়ে সিস্টেমের অবস্থা বোঝানো হতো।
- `/etc/rc3.d/` ফোল্ডারে `S20apache2`, `S10network` এরকম নামের symlink থাকত। `S` মানে start, `K` মানে kill, আর সংখ্যা দিয়ে ক্রম ঠিক হতো।
- স্ক্রিপ্টগুলো **একটার পর একটা (sequentially)** চলত।

```
SysV init (ক্রমিক, একটা শেষ হলে পরেরটা):

S10network ███████
S20ssh            ████
S30apache2            ██████████
S40cron                         ███
                                   ──► মোট সময় = সবগুলোর যোগফল

systemd (সমান্তরাল, dependency মেনে):

network    ███████
ssh               ████
apache2           ██████████
cron       ███
                            ──► মোট সময় অনেক কম
```

### SysV init-এর সমস্যাগুলো

| সমস্যা | ব্যাখ্যা |
|---|---|
| ধীর বুট | স্ক্রিপ্ট একটা একটা করে চলে, CPU-র একাধিক core কাজে লাগে না |
| সংখ্যা দিয়ে ক্রম | `S20` না `S25`? dependency আসলে কী তা স্পষ্ট লেখা থাকে না |
| দেখাশোনা নেই | সার্ভিস crash করলে init জানতেই পারে না, আবার চালু করে না |
| PID ফাইলের উপর নির্ভরতা | সার্ভিস কোন PID-এ চলছে তা জানতে `/var/run/*.pid` ফাইল লাগত, যা প্রায়ই ভুল/পুরোনো থাকত |
| প্রতিটি স্ক্রিপ্ট আলাদা | শত শত লাইনের shell script, প্রতিটি নিজের মতো লেখা, বাগে ভরা |
| child process ট্র্যাক করা কঠিন | সার্ভিস fork করে যে process বানায়, সেগুলো ধরে রাখার উপায় ছিল না |

### systemd-এর আগমন

২০১০ সালে Lennart Poettering ও Kay Sievers **systemd** তৈরি করেন। ২০১৫ সালের মধ্যে Fedora, RHEL 7, Debian 8, Ubuntu 15.04 সহ প্রায় সব বড় distro এটা গ্রহণ করে। (মাঝখানে Ubuntu কিছুদিন **Upstart** নামে নিজের init ব্যবহার করেছিল।)

systemd যা নতুন আনল:

1. **Parallel start:** dependency না থাকলে সার্ভিসগুলো একসাথে চালু হয়।
2. **Declarative unit ফাইল:** script-এর বদলে সহজ `key=value` কনফিগারেশন। "কীভাবে করবে" না লিখে "কী চাই" লেখো।
3. **cgroups (control groups) দিয়ে ট্র্যাকিং:** প্রতিটি সার্ভিস নিজের cgroup-এ চলে। সার্ভিস যত child process-ই বানাক, সবগুলো ওই cgroup-এর ভেতরে থাকে, তাই stop করলে কেউ ফাঁকি দিয়ে থেকে যায় না।
4. **Socket ও on-demand activation:** কেউ সংযোগ করলে তবেই সার্ভিস চালু।
5. **স্বয়ংক্রিয় restart:** `Restart=on-failure` লিখে দিলেই crash হলে আবার চালু।
6. **একীভূত টুলসেট:** journald (লগ), logind (লগইন সেশন), networkd, resolved, timesyncd, hostnamed, timedated, localed ইত্যাদি।

> ⚠️ **সতর্কতা:** systemd নিয়ে লিনাক্স কমিউনিটিতে বিতর্ক আছে। সমালোচকরা বলেন এটা "একটা কাজ ভালোভাবে করো" এই UNIX দর্শন ভাঙে। Devuan, Alpine (OpenRC), Void (runit), Gentoo (OpenRC ডিফল্ট) এর মতো কিছু distro systemd ব্যবহার করে না। তবে Ubuntu, Debian, Fedora, RHEL, Arch, openSUSE সবগুলোতে systemd-ই ডিফল্ট, তাই চাকরির বাজারে এটা জানা আবশ্যক।

### systemd-এর ভেতরের আর্কিটেকচার

```
┌──────────────────────────────────────────────────────────────┐
│                     ইউজার যা ব্যবহার করে                       │
│  systemctl   journalctl   hostnamectl   timedatectl          │
│  localectl   loginctl     systemd-analyze                    │
└───────────────────────────┬──────────────────────────────────┘
                            │  D-Bus (প্রোগ্রামের মধ্যে বার্তা পাঠানোর বাস)
┌───────────────────────────▼──────────────────────────────────┐
│                  systemd (PID 1) — ম্যানেজার                  │
│   unit লোড, dependency গ্রাফ, job queue, cgroup ব্যবস্থাপনা    │
├──────────────────────────────────────────────────────────────┤
│  সহায়ক daemon: systemd-journald  systemd-logind             │
│  systemd-udevd  systemd-resolved  systemd-timesyncd          │
│  systemd-hostnamed  systemd-timedated  systemd-localed       │
└───────────────────────────┬──────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────┐
│        Linux kernel: cgroups, namespaces, inotify, ...        │
└──────────────────────────────────────────────────────────────┘
```

তুমি যখন `systemctl restart nginx` লেখো, তখন `systemctl` নিজে কিছু চালু করে না। সে D-Bus দিয়ে PID 1-কে বার্তা পাঠায়: "nginx.service restart করার একটা **job** বানাও।" PID 1 সেটা job queue-তে রাখে, dependency দেখে, তারপর কাজটা করে।

---

## ২৬.৩ Unit: systemd-এর মূল ধারণা

systemd যা কিছু পরিচালনা করে, তার প্রতিটিকে বলে **unit**। unit-এর নামের শেষে যে extension থাকে, সেটা দেখে বোঝা যায় এটা কোন ধরনের unit।

| Unit টাইপ | Extension | কী বোঝায় | উদাহরণ |
|---|---|---|---|
| Service | `.service` | একটা daemon বা প্রোগ্রাম চালানো | `ssh.service`, `nginx.service` |
| Socket | `.socket` | একটা নেটওয়ার্ক/IPC socket; সংযোগ এলে service চালু করে | `ssh.socket`, `docker.socket` |
| Target | `.target` | কয়েকটা unit-এর দল; একটা "অবস্থা" বা মাইলফলক | `multi-user.target` |
| Timer | `.timer` | নির্দিষ্ট সময়ে service চালানো (cron-এর বিকল্প) | `apt-daily.timer` |
| Mount | `.mount` | একটা ফাইলসিস্টেম mount পয়েন্ট | `home.mount` |
| Automount | `.automount` | প্রথম অ্যাক্সেসে mount হবে | `proc-sys-fs-binfmt_misc.automount` |
| Device | `.device` | udev থেকে আসা হার্ডওয়্যার ডিভাইস | `dev-sda.device` |
| Path | `.path` | কোনো ফাইল/ফোল্ডার বদলালে service চালু | `cups.path` |
| Swap | `.swap` | swap পার্টিশন/ফাইল | `swapfile.swap` |
| Slice | `.slice` | cgroup-এর রিসোর্স ভাগ (CPU/মেমরি সীমা) | `user.slice`, `system.slice` |
| Scope | `.scope` | বাইরে থেকে চালু হওয়া process-এর দল (যেমন লগইন সেশন) | `session-3.scope` |

> 💡 **উপমা:** unit যেন একটা কোম্পানির বিভিন্ন কর্মী-পদ। service হলো যে আসল কাজ করে; socket হলো রিসেপশন ডেস্ক, যেখানে কেউ এলে সংশ্লিষ্ট কর্মীকে ডেকে আনা হয়; timer হলো এলার্ম ঘড়ি; target হলো একটা বিভাগ ("হিসাব বিভাগ পুরো চালু" মানে এর সব কর্মী হাজির); mount হলো গুদামঘরের দরজা খোলা।

### Service unit

সবচেয়ে বেশি ব্যবহার হয়। একটা প্রোগ্রাম কীভাবে চালাতে হবে, কোন ইউজার হিসেবে, crash করলে কী করতে হবে, সব লেখা থাকে।

### Socket unit ও socket activation

socket activation-এ systemd নিজে port-এ শুনতে (listen) থাকে। কোনো client সংযোগ করলে তবেই আসল service চালু করে socket-টা তার হাতে দিয়ে দেয়।

```
   client ──connect:22──►  ┌──────────────┐   প্রথম সংযোগে   ┌─────────────┐
                           │ systemd      │ ───────────────► │ ssh.service │
                           │ (ssh.socket  │   socket হস্তান্তর │ (sshd)      │
                           │  port 22-এ   │                  └─────────────┘
                           │  listen করে) │
                           └──────────────┘
```

সুবিধা: বুট দ্রুত হয় (সার্ভিস আগে থেকে চালু থাকতে হয় না), মেমরি বাঁচে, আর যে সার্ভিসগুলো একে অপরের socket-এর উপর নির্ভর করে, সেগুলোও একসাথে চালু হতে পারে।

> 📝 **মনে রাখো:** Ubuntu 22.10 থেকে (এবং 24.04-এ) OpenSSH ডিফল্টভাবে `ssh.socket` দিয়ে socket activation ব্যবহার করে। তাই Ubuntu 24.04-এ SSH-এর port বদলাতে চাইলে শুধু `sshd_config` বদলালে হবে না; `ssh.socket`-এর `ListenStream` বা `daemon-reload` + `ssh.socket` restart লাগে। Debian 12 ও Fedora/RHEL-এ `ssh.service`/`sshd.service` সরাসরি চলে। SSH নিয়ে বিস্তারিত [অধ্যায় ৩৮](38-ssh-remote.md)-এ।

### Target unit

target নিজে কোনো কাজ করে না; এটা অন্য unit-গুলোকে দলবদ্ধ করে একটা "সিঙ্ক্রোনাইজেশন পয়েন্ট" বানায়। যেমন `network-online.target` মানে "নেটওয়ার্ক পুরোপুরি চালু হয়েছে।" পরে ২৬.৫-এ বিস্তারিত দেখব।

### Timer unit

cron-এর আধুনিক বিকল্প। `backup.timer` ফাইলটি `backup.service`-কে সময়মতো চালু করে। বিস্তারিত [অধ্যায় ২৯](29-scheduling-cron.md)-এ।

### Mount unit

`/etc/fstab`-এর প্রতিটি লাইন থেকে systemd বুটের সময় স্বয়ংক্রিয়ভাবে একটা `.mount` unit বানায় (`systemd-fstab-generator` নামের একটা generator এই কাজ করে)। mount unit-এর নাম হয় mount পয়েন্টের path থেকে: `/` বাদ দিয়ে `-` বসিয়ে। যেমন `/var/lib/data` → `var-lib-data.mount`।

```
$ systemd-escape -p --suffix=mount /var/lib/data
var-lib-data.mount
```

ডিস্ক ও mount নিয়ে বিস্তারিত [অধ্যায় ৩২](32-disk-partition-mount.md)-এ।

### Template unit (`@` চিহ্নওয়ালা)

কিছু unit-এর নামে `@` থাকে, যেমন `getty@.service`। এটা একটা ছাঁচ (template)। `@`-এর পরে যা লিখবে, তা হবে **instance**-এর নাম:

```
getty@tty1.service   → tty1-এ লগইন প্রম্পট
getty@tty2.service   → tty2-এ লগইন প্রম্পট
```

unit ফাইলের ভেতরে `%i` লিখলে সেখানে instance নামটা বসে যায়। এভাবে এক ফাইল দিয়ে অনেক instance চালানো যায়।

---

## ২৬.৪ systemctl: সার্ভিস নিয়ন্ত্রণের প্রধান অস্ত্র

`systemctl` হলো systemd-এর সাথে কথা বলার মূল কমান্ড।

**সাধারণ syntax:**

```
systemctl [OPTIONS] COMMAND [UNIT...]
```

unit-এর নামে `.service` না লিখলে systemctl ধরে নেয় সেটা service। অর্থাৎ `systemctl status ssh` আর `systemctl status ssh.service` একই।

> ⚠️ **সতর্কতা:** অবস্থা দেখার কমান্ড (`status`, `is-active`, `list-units`) সাধারণ ইউজার চালাতে পারে, কিন্তু অবস্থা বদলানোর কমান্ড (`start`, `stop`, `enable` ইত্যাদি) চালাতে `sudo` লাগে ([অধ্যায় ১৮](18-sudo-root.md))। sudo ছাড়া চালালে desktop-এ polkit পাসওয়ার্ড চাইতে পারে।

> 📝 **মনে রাখো (distro-ভেদে নাম):** একই সফটওয়্যারের সার্ভিসের নাম distro-ভেদে আলাদা হয়।
>
> | সফটওয়্যার | Ubuntu/Debian | Fedora/RHEL |
> |---|---|---|
> | OpenSSH সার্ভার | `ssh` | `sshd` |
> | Apache | `apache2` | `httpd` |
> | cron | `cron` | `crond` |
> | MySQL/MariaDB | `mysql` / `mariadb` | `mysqld` / `mariadb` |

### ২৬.৪.১ status: অবস্থা দেখা

```
$ systemctl status ssh
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; disabled; preset: enabled)
     Active: active (running) since Tue 2026-11-10 09:14:02 +06; 2h 5min ago
TriggeredBy: ● ssh.socket
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 1043 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 1051 (sshd)
      Tasks: 1 (limit: 9318)
     Memory: 4.2M (peak: 5.0M)
        CPU: 61ms
     CGroup: /system.slice/ssh.service
             └─1051 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

Nov 10 09:14:02 server1 systemd[1]: Starting ssh.service - OpenBSD Secure Shell server...
Nov 10 09:14:02 server1 sshd[1051]: Server listening on 0.0.0.0 port 22.
Nov 10 09:14:02 server1 systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
```

প্রতিটি লাইন কী বলছে:

| লাইন | অর্থ |
|---|---|
| `●` (প্রথম লাইনের বিন্দু) | সবুজ = চলছে, সাদা = বন্ধ, লাল = failed |
| `Loaded:` | unit ফাইল কোথা থেকে লোড হলো; `enabled/disabled` = বুটে নিজে চালু হবে কি না; `preset` = distro-র ডিফল্ট পছন্দ |
| `Active:` | এখনকার অবস্থা ও কতক্ষণ ধরে |
| `TriggeredBy:` | কোন socket/timer এটাকে চালু করে |
| `Main PID:` | মূল process-এর PID |
| `Tasks`, `Memory`, `CPU` | cgroup থেকে পাওয়া রিসোর্স ব্যবহার |
| `CGroup:` | এই সার্ভিসের অধীনে থাকা সব process-এর গাছ |
| শেষের লাইনগুলো | journal থেকে সর্বশেষ ১০টা লগ লাইন |

উপরের উদাহরণে `disabled` লেখা থাকলেও সার্ভিস চলছে, কারণ এটা `ssh.socket` দিয়ে চালু হয়েছে (Ubuntu 24.04-এর বৈশিষ্ট্য)।

**Active অবস্থার সম্ভাব্য মান:**

| অবস্থা | অর্থ |
|---|---|
| `active (running)` | চলছে, process জীবিত |
| `active (exited)` | একবার চলে সফলভাবে শেষ (`oneshot` টাইপে স্বাভাবিক) |
| `active (waiting)` | চালু, কিন্তু কোনো ঘটনার অপেক্ষায় (যেমন timer) |
| `inactive (dead)` | বন্ধ |
| `activating` / `deactivating` | চালু/বন্ধ হওয়ার মাঝপথে |
| `failed` | ব্যর্থ হয়েছে (crash, exit code ≠ 0, timeout) |

দরকারি অপশন:

```
$ systemctl status nginx -n 50          # শেষ ৫০টা লগ লাইন
$ systemctl status nginx --no-pager     # pager (less) ছাড়া সরাসরি আউটপুট
$ systemctl status nginx -l             # লম্বা লাইন না কেটে পুরোটা দেখাও
$ systemctl status 1051                 # PID দিয়েও খোঁজা যায় (কোন unit-এর process)
```

> 💡 **টিপস:** status-এর শুধু ১০ লাইন লগ যথেষ্ট না হলে `journalctl -u nginx` চালাও। `journalctl -u nginx -f` দিলে লাইভ লগ দেখাবে, `-b` দিলে শুধু এই বুটের। journal নিয়ে বিস্তারিত [অধ্যায় ২৮](28-logs.md)-এ।

### ২৬.৪.২ start, stop, restart, reload

```
$ sudo systemctl start nginx       # এখনই চালু করো
$ sudo systemctl stop nginx        # এখনই বন্ধ করো
$ sudo systemctl restart nginx     # বন্ধ করে আবার চালু (process নতুন হবে)
$ sudo systemctl reload nginx      # process না থামিয়ে কনফিগ নতুন করে পড়ো
```

এই কমান্ডগুলো সফল হলে কিছুই প্রিন্ট করে না (UNIX-এর "নীরবতাই সাফল্য" নীতি)। ব্যর্থ হলে বার্তা দেখায়:

```
$ sudo systemctl start nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```

**restart বনাম reload:**

```
restart:  [পুরোনো process] ──SIGTERM──► মৃত   ...   [নতুন process] চালু
          সংযোগগুলো কেটে যায়, কিছুক্ষণ সার্ভিস অনুপলব্ধ

reload:   [একই process] ◄──SIGHUP বা ExecReload কমান্ড── নতুন কনফিগ পড়ে
          চলমান সংযোগ টিকে থাকে, ডাউনটাইম প্রায় শূন্য
```

| কমান্ড | কী করে |
|---|---|
| `restart` | সবসময় stop + start; বন্ধ থাকলে চালু করে |
| `reload` | unit ফাইলের `ExecReload=` চালায়; সব সার্ভিস এটা সমর্থন করে না |
| `try-restart` | চলমান থাকলে তবেই restart; বন্ধ থাকলে কিছু করে না |
| `reload-or-restart` | reload সমর্থন করলে reload, না হলে restart |
| `try-reload-or-restart` | চলমান থাকলেই, reload বা restart |

> ⚠️ **সতর্কতা:** `systemctl reload` সার্ভিসের **কনফিগারেশন** (যেমন `/etc/nginx/nginx.conf`) আবার পড়ায়। এটা systemd-এর **unit ফাইল** আবার পড়ায় না। unit ফাইল বদলালে `systemctl daemon-reload` লাগে (২৬.৪.৮)। এই দুটো গুলিয়ে ফেলা নতুনদের সবচেয়ে সাধারণ ভুল।

> 💡 **টিপস:** প্রোডাকশনে কনফিগ বদলানোর পর আগে সিনট্যাক্স যাচাই করো, তারপর reload: `sudo nginx -t && sudo systemctl reload nginx`। `&&` মানে প্রথমটা সফল হলেই দ্বিতীয়টা চলবে।

একসাথে একাধিক unit:

```
$ sudo systemctl restart nginx php8.3-fpm
```

### ২৬.৪.৩ enable ও disable: বুটে চালু হবে কি না

`start` মানে **এখন** চালু। `enable` মানে **পরের বুট থেকে** স্বয়ংক্রিয়ভাবে চালু। দুটো সম্পূর্ণ আলাদা।

```
$ sudo systemctl enable nginx
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /usr/lib/systemd/system/nginx.service.

$ sudo systemctl disable nginx
Removed "/etc/systemd/system/multi-user.target.wants/nginx.service".
```

**ভেতরে কী হলো?** `enable` আসলে শুধু একটা **symlink** বানায়। unit ফাইলের `[Install]` অংশে `WantedBy=multi-user.target` লেখা থাকে, তাই systemd `multi-user.target.wants/` ফোল্ডারে symlink রাখে। বুটের সময় systemd যখন `multi-user.target` চালু করে, তখন ওই `.wants` ফোল্ডারের সবাইকে চালু করে।

```
/etc/systemd/system/
└── multi-user.target.wants/
    ├── nginx.service  ──► /usr/lib/systemd/system/nginx.service
    ├── cron.service   ──► /usr/lib/systemd/system/cron.service
    └── ssh.service    ──► ...
```

**একসাথে enable ও start:**

```
$ sudo systemctl enable --now nginx      # enable + start
$ sudo systemctl disable --now nginx     # disable + stop
```

> 💡 **টিপস:** নতুন সার্ভিস ইনস্টল করে প্রায় সবসময় `enable --now` ব্যবহার করবে। শুধু `start` দিলে রিবুটের পর সার্ভিস আর চলবে না, আর শুধু `enable` দিলে রিবুট পর্যন্ত চলবে না।

> 📝 **মনে রাখো:** Debian/Ubuntu-তে কোনো সার্ভিস প্যাকেজ (যেমন `apt install nginx`) ইনস্টল করলে সাধারণত সেটা নিজে থেকেই enable ও start হয়ে যায়। Fedora/RHEL-এ ডিফল্টভাবে হয় না; নিজে `enable --now` করতে হয়। এই পার্থক্য আসে **preset** নীতি থেকে (`/usr/lib/systemd/system-preset/`)।

### ২৬.৪.৪ is-active, is-enabled, is-failed: স্ক্রিপ্টের জন্য প্রশ্ন

এগুলো এক শব্দে উত্তর দেয় এবং exit code (০ = হ্যাঁ) দিয়ে ফল জানায়, তাই শেল স্ক্রিপ্টে খুব কাজের ([অধ্যায় ৪১](41-shell-scripting-1.md))।

```
$ systemctl is-active nginx
active

$ systemctl is-enabled nginx
enabled

$ systemctl is-failed nginx
active
```

`is-failed` failed না হলে বর্তমান অবস্থা প্রিন্ট করে এবং exit code ১ দেয়।

`is-enabled`-এর সম্ভাব্য উত্তর: `enabled`, `disabled`, `static` (কোনো `[Install]` অংশ নেই, অন্য unit-এর dependency হিসেবেই চলে), `masked`, `alias`, `indirect`, `generated`, `linked`।

স্ক্রিপ্টে ব্যবহার:

```bash
if systemctl is-active --quiet nginx; then
    echo "nginx চলছে"
else
    echo "nginx বন্ধ! চালু করছি..."
    sudo systemctl start nginx
fi
```

`--quiet` (বা `-q`) কিছু প্রিন্ট না করে শুধু exit code দেয়।

### ২৬.৪.৫ mask ও unmask: সম্পূর্ণ নিষিদ্ধ করা

`disable` করা সার্ভিস হাতে `start` করা যায়, বা অন্য কোনো সার্ভিস dependency হিসেবে তাকে চালু করে দিতে পারে। **mask** করলে কেউই আর চালু করতে পারবে না।

```
$ sudo systemctl mask apache2
Created symlink /etc/systemd/system/apache2.service → /dev/null.

$ sudo systemctl start apache2
Failed to start apache2.service: Unit apache2.service is masked.

$ sudo systemctl unmask apache2
Removed "/etc/systemd/system/apache2.service".
```

**ভেতরে কী হয়:** `/etc/systemd/system/`-এ unit-এর নামে `/dev/null`-এর দিকে একটা symlink তৈরি হয়। `/etc` অগ্রাধিকার পায় (২৬.৬ দেখো), তাই systemd আসল ফাইলের বদলে একটা ফাঁকা "কিছুই না" পায়।

| অবস্থা | বুটে চালু হবে? | হাতে start করা যাবে? | অন্য unit চালু করাতে পারবে? |
|---|---|---|---|
| enabled | হ্যাঁ | হ্যাঁ | হ্যাঁ |
| disabled | না | হ্যাঁ | হ্যাঁ |
| masked | না | না | না |

> 💡 **উপমা:** disable মানে কর্মীকে বলা "কাল থেকে নিজে আসার দরকার নেই, ডাকলে এসো।" mask মানে তার অফিস কার্ড বাতিল করে দেওয়া; ডাকলেও ঢুকতে পারবে না।

> 💡 **টিপস:** একই port-এ দুটো সার্ভিস (যেমন apache2 ও nginx দুটোই port 80) চললে সংঘর্ষ হয়। যেটা চাও না সেটাকে `mask --now` করে দাও: `sudo systemctl mask --now apache2`। আবার, ল্যাপটপে sleep বন্ধ রাখতে কেউ কেউ `sleep.target suspend.target hibernate.target hybrid-sleep.target` mask করে।

### ২৬.৪.৬ list-units ও list-unit-files: তালিকা দেখা

```
$ systemctl list-units --type=service
  UNIT                      LOAD   ACTIVE SUB     DESCRIPTION
  cron.service              loaded active running Regular background program processing daemon
  nginx.service             loaded active running A high performance web server and a reverse proxy server
  ssh.service               loaded active running OpenBSD Secure Shell server
  systemd-journald.service  loaded active running Journal Service
  ...

LOAD   = Reflects whether the unit definition was properly loaded.
ACTIVE = The high-level unit activation state, i.e. generalization of SUB.
SUB    = The low-level unit activation state, values depend on unit type.
42 loaded units listed. Pass --all to see loaded but inactive units, too.
```

`list-units` শুধু **মেমরিতে লোড হওয়া** unit দেখায় (ডিফল্টভাবে শুধু active বা failed)। আর `list-unit-files` ডিস্কে থাকা **সব** unit ফাইল ও তাদের enable অবস্থা দেখায়।

```
$ systemctl list-unit-files --type=service --state=enabled
UNIT FILE                 STATE   PRESET
cron.service              enabled enabled
nginx.service             enabled enabled
systemd-timesyncd.service enabled enabled
...
```

**দরকারি অপশন:**

| অপশন | কাজ |
|---|---|
| `--type=service` (বা `-t service`) | নির্দিষ্ট ধরন: service, socket, timer, target, mount... |
| `--state=running` | নির্দিষ্ট অবস্থা: running, failed, exited, inactive, enabled... |
| `--all` (বা `-a`) | inactive unit সহ সব দেখাও |
| `--failed` | শুধু ব্যর্থ unit (`--state=failed`-এর সংক্ষেপ) |
| `--no-legend` | হেডার ও ফুটার বাদ (স্ক্রিপ্টের জন্য) |
| `--no-pager` | pager ছাড়া আউটপুট |
| প্যাটার্ন | `systemctl list-units 'ssh*'` |

সবচেয়ে দরকারি কয়েকটা:

```
$ systemctl --failed                                  # কী কী ব্যর্থ হয়েছে?
  UNIT            LOAD   ACTIVE SUB    DESCRIPTION
● myapp.service   loaded failed failed My Python App

$ systemctl list-units --type=service --state=running # কী কী চলছে?
$ systemctl list-timers                               # সব timer ও পরের চলার সময়
$ systemctl list-sockets                              # কোন socket কোন port-এ
```

শুধু `systemctl` লিখে Enter চাপলেও `list-units`-ই চলে।

> 💡 **টিপস:** সার্ভার লগইন করার পর প্রথম কমান্ড হিসেবে `systemctl --failed` চালানোর অভ্যাস করো। এক নজরে বুঝবে কিছু ভেঙেছে কি না। পুরো সিস্টেমের সারাংশ পেতে `systemctl status` (unit নাম ছাড়া) চালাও; উপরে `State: running` বা `State: degraded` দেখাবে।

### ২৬.৪.৭ আরও দরকারি systemctl কমান্ড

```
$ systemctl cat nginx                 # unit ফাইল ও সব override একসাথে দেখাও
# /usr/lib/systemd/system/nginx.service
[Unit]
Description=A high performance web server and a reverse proxy server
...

$ systemctl show nginx -p MainPID -p ActiveState -p Restart
MainPID=1210
ActiveState=active
Restart=no

$ systemctl list-dependencies nginx   # কার উপর নির্ভরশীল (গাছ আকারে)
$ systemctl list-dependencies --reverse nginx   # কে nginx-এর উপর নির্ভরশীল

$ sudo systemctl kill -s SIGUSR1 nginx      # সার্ভিসের process-এ signal পাঠাও
$ sudo systemctl reset-failed               # failed অবস্থা মুছে দাও
```

| কমান্ড | কাজ |
|---|---|
| `cat UNIT` | কার্যকর unit ফাইল (মূল + drop-in) দেখায় |
| `show UNIT` | সব property `Key=Value` আকারে (মেশিন-পাঠযোগ্য) |
| `show -p PROP UNIT` | নির্দিষ্ট property |
| `list-dependencies UNIT` | dependency গাছ |
| `kill UNIT` | cgroup-এর সব process-এ signal |
| `reset-failed [UNIT]` | failed অবস্থা ও restart কাউন্টার রিসেট |
| `edit UNIT` | override তৈরি (২৬.৮) |
| `revert UNIT` | সব override মুছে মূল অবস্থায় ফেরা |
| `daemon-reload` | unit ফাইল আবার পড়া |
| `get-default` / `set-default` | ডিফল্ট target |
| `isolate TARGET` | শুধু ওই target ও তার dependency রেখে বাকি সব থামাও |
| `poweroff`, `reboot`, `suspend`, `hibernate` | পাওয়ার ব্যবস্থাপনা |

পাওয়ার কমান্ড: `shutdown`, `reboot`, `poweroff`, `halt` কমান্ডগুলো আধুনিক সিস্টেমে আসলে systemctl-এরই symlink।

```
$ sudo systemctl reboot
$ sudo systemctl poweroff
$ sudo shutdown -h +10 "১০ মিনিটে বন্ধ হবে"   # শিডিউল করা শাটডাউন
$ sudo shutdown -c                             # বাতিল
```

### ২৬.৪.৮ daemon-reload: unit ফাইল বদলানোর পর

systemd বুটের সময় সব unit ফাইল পড়ে মেমরিতে রাখে। তুমি পরে কোনো unit ফাইল নিজে বদলালে systemd তা জানে না, পুরোনো সংস্করণ নিয়েই কাজ চালায়।

```
$ sudo nano /etc/systemd/system/myapp.service      # কিছু বদলালে

$ sudo systemctl restart myapp
Warning: The unit file, source configuration file or drop-ins of myapp.service changed on disk. Run 'systemctl daemon-reload' to reload units.

$ sudo systemctl daemon-reload                     # নতুন করে সব unit ফাইল পড়ো
$ sudo systemctl restart myapp                     # এবার নতুন সেটিং দিয়ে চলবে
```

```
  ডিস্ক                       PID 1-এর মেমরি
┌──────────────────┐        ┌──────────────────┐
│ myapp.service    │        │ myapp.service    │
│ (নতুন সংস্করণ)   │  ──X──► │ (পুরোনো সংস্করণ) │
└──────────────────┘        └──────────────────┘
          │  daemon-reload
          └───────────────────►  মেমরি আপডেট
```

> 📝 **মনে রাখো:** `daemon-reload` চলমান সার্ভিস restart করে না, শুধু definition আপডেট করে। নতুন সেটিং প্রয়োগ করতে সার্ভিসটা restart করতে হবে। আর `systemctl edit` ব্যবহার করলে daemon-reload নিজেই হয়ে যায়।

`daemon-reexec` আলাদা জিনিস: এটা systemd বাইনারিটাকেই নতুন করে চালায় (সাধারণত systemd প্যাকেজ আপগ্রেডের পর প্যাকেজ ম্যানেজার নিজেই করে)।

---

## ২৬.৫ Target ও Runlevel

### runlevel কী ছিল

SysV init-এ সিস্টেমের অবস্থা বোঝাতে ০–৬ সংখ্যা ব্যবহার হতো। systemd সংখ্যার বদলে নামওয়ালা **target** ব্যবহার করে, তবে পুরোনো অভ্যাসের সাথে মিল রাখতে `runlevelN.target` নামের alias রেখেছে।

| Runlevel | systemd target | অর্থ |
|---|---|---|
| 0 | `poweroff.target` | সিস্টেম বন্ধ |
| 1 | `rescue.target` | single-user মোড: শুধু root, নেটওয়ার্ক নেই, মেরামতের জন্য |
| 2 | `multi-user.target` | (Debian-এ পুরোনো দিনে পূর্ণ মোড ছিল) |
| 3 | `multi-user.target` | মাল্টি-ইউজার, নেটওয়ার্ক সহ, **GUI ছাড়া** (সার্ভারের সাধারণ মোড) |
| 4 | `multi-user.target` | ব্যবহারকারী-নির্ধারিত (কার্যত অব্যবহৃত) |
| 5 | `graphical.target` | multi-user + GUI লগইন স্ক্রিন (ডেস্কটপের সাধারণ মোড) |
| 6 | `reboot.target` | রিবুট |
| — | `emergency.target` | rescue-এর চেয়েও কম: শুধু root ফাইলসিস্টেম read-only, প্রায় কোনো সার্ভিস নেই |

target-গুলো একে অপরের উপর দাঁড়িয়ে থাকে:

```
                 graphical.target        (GUI: gdm/sddm)
                        │ Requires
                        ▼
                 multi-user.target       (ssh, cron, nginx ...)
                        │ Requires
                        ▼
                  basic.target           (sockets, timers, paths, slices)
                        │
                        ▼
                 sysinit.target          (mount, swap, udev, journald)
                        │
                        ▼
              local-fs.target + swap.target
```

এই ক্রম ও প্রতিটি ধাপের বিস্তারিত [অধ্যায় ২৭](27-boot-process.md)-এ পাবে।

### target নিয়ে কাজ

```
$ systemctl get-default
graphical.target

$ sudo systemctl set-default multi-user.target     # পরের বুট থেকে GUI ছাড়া
Removed "/etc/systemd/system/default.target".
Created symlink /etc/systemd/system/default.target → /usr/lib/systemd/system/multi-user.target.

$ sudo systemctl isolate multi-user.target         # এখনই GUI বন্ধ করে টেক্সট মোডে
$ sudo systemctl isolate graphical.target          # এখনই GUI-তে ফেরা
$ sudo systemctl rescue                            # rescue মোডে যাও (isolate rescue.target)
```

`default.target` আসলে একটা symlink; `set-default` শুধু সেটা বদলে দেয়।

> ⚠️ **সতর্কতা:** `isolate` ওই target-এর অংশ নয় এমন **সব** unit থামিয়ে দেয়। SSH দিয়ে দূরের সার্ভারে `isolate rescue.target` চালালে নেটওয়ার্ক বন্ধ হয়ে তোমার সংযোগ কেটে যাবে, আর ফিরে আসতে সার্ভারের কনসোলে সরাসরি যেতে হবে।

শুধু যে target-গুলোর unit ফাইলে `AllowIsolate=yes` আছে, সেগুলোকেই isolate করা যায়।

**বুটের সময় এককালীন target বেছে নেওয়া:** GRUB মেনুতে `e` চেপে kernel লাইনের (`linux ...`) শেষে লেখো `systemd.unit=rescue.target` (বা `emergency.target`), তারপর `Ctrl+X` বা `F10` চেপে বুট করো। পাসওয়ার্ড ভুলে গেলে বা fstab ভুলের কারণে বুট আটকে গেলে এটাই উদ্ধারের রাস্তা।

পুরোনো কমান্ডগুলোও এখনো কাজ করে:

```
$ runlevel
N 5
$ who -r
         run-level 5  2026-11-10 09:13
```

`N 5` মানে আগের runlevel নেই (N = none), এখন ৫।

---

## ২৬.৬ Unit ফাইল কোথায় থাকে

systemd তিনটি প্রধান জায়গা থেকে unit ফাইল পড়ে, এবং একই নামের ফাইল একাধিক জায়গায় থাকলে **অগ্রাধিকার (priority)** মেনে চলে।

| অবস্থান | কে রাখে | অগ্রাধিকার |
|---|---|---|
| `/etc/systemd/system/` | **তুমি (অ্যাডমিন)** | সর্বোচ্চ |
| `/run/systemd/system/` | runtime-এ তৈরি (রিবুটে মুছে যায়) | মাঝারি |
| `/usr/lib/systemd/system/` (বা `/lib/systemd/system/`) | **প্যাকেজ ম্যানেজার** (apt/dnf) | সর্বনিম্ন |

```
একই নামের unit খোঁজার ক্রম:

  /etc/systemd/system/nginx.service      ← থাকলে এটাই জিতবে
          │ না থাকলে
          ▼
  /run/systemd/system/nginx.service
          │ না থাকলে
          ▼
  /usr/lib/systemd/system/nginx.service  ← প্যাকেজের দেওয়া মূল ফাইল
```

> 📝 **মনে রাখো (/lib বনাম /usr/lib):** Ubuntu 24.04, Debian 12, Fedora ও RHEL 9 সবগুলোই "merged /usr" ব্যবহার করে, অর্থাৎ `/lib` আসলে `/usr/lib`-এর একটা symlink। তাই `/lib/systemd/system` আর `/usr/lib/systemd/system` একই ফোল্ডার। পুরোনো Debian/Ubuntu ডকুমেন্টেশনে `/lib/systemd/system` লেখা দেখবে; Fedora/RHEL-এ `/usr/lib/systemd/system`। FHS নিয়ে আরও জানতে [অধ্যায় ৭](07-filesystem-hierarchy.md)।

> ⚠️ **সতর্কতা:** `/usr/lib/systemd/system/`-এর ফাইল **কখনো সরাসরি এডিট করবে না।** পরের প্যাকেজ আপডেটে তোমার পরিবর্তন মুছে যাবে। পরিবর্তন সবসময় `/etc/systemd/system/`-এ করো, সবচেয়ে ভালো উপায় `systemctl edit` (২৬.৮)।

**user unit-এর অবস্থান** (`systemctl --user`, ২৬.১১):

| অবস্থান | ব্যাখ্যা |
|---|---|
| `~/.config/systemd/user/` | নিজের ইউজারের unit |
| `/etc/systemd/user/` | অ্যাডমিন দিয়েছে, সব ইউজারের জন্য |
| `/usr/lib/systemd/user/` | প্যাকেজের দেওয়া |

পুরো তালিকা দেখতে চাইলে:

```
$ systemd-analyze unit-paths
/etc/systemd/system.control
/run/systemd/system.control
/run/systemd/transient
/run/systemd/generator.early
/etc/systemd/system
/etc/systemd/system.attached
/run/systemd/system
...
/usr/lib/systemd/system
/run/systemd/generator.late
```

---

## ২৬.৭ Unit ফাইলের গঠন

unit ফাইল হলো INI-ধরনের একটা টেক্সট ফাইল: `[Section]` হেডার, তার নিচে `Key=Value` লাইন। `#` বা `;` দিয়ে শুরু লাইন হলো মন্তব্য।

একটা বাস্তব উদাহরণ (Ubuntu-র `cron.service`, সামান্য সংক্ষেপিত):

```ini
[Unit]
Description=Regular background program processing daemon
Documentation=man:cron(8)
After=remote-fs.target nss-user-lookup.target

[Service]
EnvironmentFile=-/etc/default/cron
ExecStart=/usr/sbin/cron -f -P $EXTRA_OPTS
IgnoreSIGPIPE=false
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

(`EnvironmentFile=`-এর আগে `-` মানে "ফাইলটা না থাকলেও ভুল ধরো না।")

```
┌────────────────────────────────────────────────────────┐
│ [Unit]     → সব unit টাইপে থাকে: পরিচিতি ও dependency  │
├────────────────────────────────────────────────────────┤
│ [Service]  → টাইপ-নির্দিষ্ট অংশ: কীভাবে চালাতে হবে     │
│ ([Socket], [Timer], [Mount], [Path] ... টাইপ অনুযায়ী)  │
├────────────────────────────────────────────────────────┤
│ [Install]  → enable করলে কোথায় যুক্ত হবে              │
└────────────────────────────────────────────────────────┘
```

### [Unit] সেকশনের গুরুত্বপূর্ণ key

| Key | কাজ |
|---|---|
| `Description=` | এক লাইনের বর্ণনা (status-এ দেখায়) |
| `Documentation=` | man পেজ বা URL |
| `Wants=` | দুর্বল dependency (২৬.৯) |
| `Requires=` | শক্ত dependency |
| `After=` / `Before=` | চালু হওয়ার ক্রম |
| `Conflicts=` | একসাথে চলতে পারবে না |
| `ConditionPathExists=` | শর্ত পূরণ না হলে নীরবে skip |
| `AssertPathExists=` | শর্ত পূরণ না হলে failed |
| `StartLimitIntervalSec=`, `StartLimitBurst=` | নির্দিষ্ট সময়ে সর্বোচ্চ কতবার start করা যাবে |
| `OnFailure=` | ব্যর্থ হলে অন্য কোন unit চালু হবে (যেমন নোটিফিকেশন) |

### [Service] সেকশনের গুরুত্বপূর্ণ key

| Key | কাজ |
|---|---|
| `Type=` | সার্ভিস কীভাবে "চালু হয়েছে" জানায় (নিচের টেবিল) |
| `ExecStart=` | মূল কমান্ড (**absolute path** দিতে হবে বা PATH-এ থাকতে হবে) |
| `ExecStartPre=` / `ExecStartPost=` | আগে/পরে চালানোর কমান্ড |
| `ExecReload=` | `reload`-এ কী চলবে, যেমন `/bin/kill -HUP $MAINPID` |
| `ExecStop=` | থামানোর কমান্ড (না দিলে SIGTERM পাঠানো হয়) |
| `Restart=` | কখন আবার চালু হবে |
| `RestartSec=` | restart-এর আগে অপেক্ষা (ডিফল্ট 100ms) |
| `User=` / `Group=` | কোন ইউজার হিসেবে চলবে (ডিফল্ট root) |
| `WorkingDirectory=` | কোন ফোল্ডার থেকে চলবে |
| `Environment=` | এনভায়রনমেন্ট ভেরিয়েবল, যেমন `Environment="PORT=8080"` |
| `EnvironmentFile=` | ফাইল থেকে ভেরিয়েবল পড়া |
| `TimeoutStartSec=` / `TimeoutStopSec=` | কতক্ষণে চালু/বন্ধ না হলে ব্যর্থ ধরা হবে (ডিফল্ট সাধারণত 90s) |
| `KillMode=` | stop-এ কাদের মারা হবে: `control-group` (ডিফল্ট, সবাই), `mixed`, `process`, `none` |
| `StandardOutput=` / `StandardError=` | আউটপুট কোথায় যাবে (ডিফল্ট journal) |

**`Type=` এর মান:**

| Type | কখন ব্যবহার করবে | systemd কখন "চালু হয়েছে" ধরে |
|---|---|---|
| `simple` | ডিফল্ট; প্রোগ্রাম foreground-এ চলতে থাকে | fork করার সাথে সাথেই |
| `exec` | simple-এর মতো, তবে বেশি নির্ভুল | প্রোগ্রাম আসলে exec হলে (বাইনারি না থাকলে start-ই ব্যর্থ হবে) |
| `forking` | পুরোনো ধাঁচের daemon যারা নিজে background-এ চলে যায় | parent process exit করলে (প্রায়ই `PIDFile=` লাগে) |
| `oneshot` | একবার চলে শেষ হয় (স্ক্রিপ্ট, সেটআপ কাজ) | process শেষ হলে; সাথে `RemainAfterExit=yes` দিলে পরে active থাকে |
| `notify` | প্রোগ্রাম নিজে `sd_notify("READY=1")` পাঠায় | READY বার্তা পেলে |
| `dbus` | D-Bus-এ নাম নেয় | `BusName=` পাওয়া গেলে |
| `idle` | simple-এর মতো, কিন্তু বাকি job শেষ হওয়া পর্যন্ত অপেক্ষা করে | কনসোল আউটপুট এলোমেলো না করার জন্য |

> 💡 **টিপস:** নিজের লেখা প্রোগ্রাম (Python, Node.js, Go) সাধারণত foreground-এ চলে, তাই `Type=simple` বা `Type=exec` দাও। প্রোগ্রামের ভেতরে "daemonize" বা `&` দিয়ে background-এ পাঠাবে না; systemd নিজেই সেটা সামলায়।

**`Restart=` এর মান:**

| মান | কখন restart |
|---|---|
| `no` | কখনো না (ডিফল্ট) |
| `on-failure` | exit code ≠ 0, signal-এ মৃত্যু, timeout, watchdog |
| `on-abnormal` | signal, timeout, watchdog (exit code ≠ 0 হলে নয়) |
| `on-abort` | শুধু অপ্রত্যাশিত signal-এ মৃত্যু |
| `on-success` | শুধু পরিষ্কারভাবে exit 0 হলে |
| `always` | যেভাবেই থামুক (হাতে `systemctl stop` বাদে) |

> ⚠️ **সতর্কতা:** `Restart=always` দিলে আর প্রোগ্রামে কনফিগ ভুল থাকলে সেটা বারবার crash করবে। ডিফল্টভাবে ১০ সেকেন্ডে ৫ বারের বেশি start হলে systemd হাল ছেড়ে দিয়ে `failed` করে (বার্তা: `start request repeated too quickly`)। তখন ভুল ঠিক করে `systemctl reset-failed myapp` তারপর start করো।

### [Install] সেকশন

| Key | কাজ |
|---|---|
| `WantedBy=` | enable করলে কোন target-এর `.wants/`-এ symlink হবে (সবচেয়ে সাধারণ: `multi-user.target`) |
| `RequiredBy=` | `.requires/`-এ symlink |
| `Alias=` | বিকল্প নাম |
| `Also=` | এটা enable করলে আর কী কী enable হবে |

`[Install]` অংশ শুধু `enable`/`disable` এর সময় পড়া হয়; সার্ভিস চলার সময় এর কোনো ভূমিকা নেই। যে unit-এ `[Install]` নেই, সেটা `static`; তাকে enable করা যায় না, অন্য unit dependency হিসেবে চালায়।

> 💡 **টিপস:** যেকোনো key-এর বিস্তারিত পেতে: `man systemd.unit` (সাধারণ ও [Unit]/[Install]), `man systemd.service` ([Service]), `man systemd.exec` (User, Environment, sandboxing), `man systemd.socket`, `man systemd.timer`, `man systemd.kill`, `man systemd.resource-control`। সব directive-এর সূচি: `man systemd.directives`।

### Socket unit-এর একটা ছোট উদাহরণ

```ini
# /etc/systemd/system/echo.socket
[Unit]
Description=Echo socket

[Socket]
ListenStream=127.0.0.1:7777
Accept=yes

[Install]
WantedBy=sockets.target
```

`Accept=yes` মানে প্রতিটি সংযোগের জন্য আলাদা instance চালু হবে, তাই সাথে একটা template service লাগে:

```ini
# /etc/systemd/system/echo@.service
[Unit]
Description=Echo service

[Service]
ExecStart=/usr/bin/cat
StandardInput=socket
```

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now echo.socket
$ echo "hello systemd" | nc -q1 127.0.0.1 7777
hello systemd
```

(`nc` না থাকলে `sudo apt install netcat-openbsd`। Fedora-তে `nmap-ncat` প্যাকেজের `nc` ব্যবহার করো, যেখানে `-q1` অপশন নেই; সেখানে `--send-only` বা Ctrl+C দিয়ে বেরোও।)

---

## ২৬.৮ নিজের service unit লেখা: সম্পূর্ণ উদাহরণ

ধরো তুমি একটা ছোট Python ওয়েব অ্যাপ বানিয়েছ, চাও সেটা বুটে নিজে চালু হোক, crash করলে আবার উঠুক, আর root হিসেবে না চলুক।

### ধাপ ১: প্রোগ্রাম তৈরি

```
$ sudo mkdir -p /opt/hello
$ sudo nano /opt/hello/app.py
```

```python
#!/usr/bin/env python3
import http.server, os
port = int(os.environ.get("PORT", "8080"))
handler = http.server.SimpleHTTPRequestHandler
print(f"Starting on port {port}", flush=True)
http.server.ThreadingHTTPServer(("0.0.0.0", port), handler).serve_forever()
```

### ধাপ ২: আলাদা system user তৈরি

সার্ভিস কখনো root হিসেবে চালানো উচিত না। একটা লগইন-অক্ষম সিস্টেম ইউজার বানাও ([অধ্যায় ১৫](15-users-groups.md)):

```
$ sudo useradd --system --no-create-home --shell /usr/sbin/nologin hello
$ sudo chown -R hello:hello /opt/hello
```

### ধাপ ৩: unit ফাইল লেখা

```
$ sudo nano /etc/systemd/system/hello.service
```

```ini
[Unit]
Description=Hello Python web app
Documentation=https://example.com/hello
After=network-online.target
Wants=network-online.target

[Service]
Type=exec
User=hello
Group=hello
WorkingDirectory=/opt/hello
Environment="PORT=8080"
ExecStart=/usr/bin/python3 /opt/hello/app.py
Restart=on-failure
RestartSec=5

# নিরাপত্তা (sandboxing)
NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=yes

[Install]
WantedBy=multi-user.target
```

### ধাপ ৪: যাচাই, লোড, চালু

```
$ sudo systemd-analyze verify /etc/systemd/system/hello.service
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now hello
Created symlink /etc/systemd/system/multi-user.target.wants/hello.service → /etc/systemd/system/hello.service.

$ systemctl status hello --no-pager
● hello.service - Hello Python web app
     Loaded: loaded (/etc/systemd/system/hello.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-12 10:02:41 +06; 3s ago
   Main PID: 4821 (python3)
      Tasks: 1 (limit: 9318)
     Memory: 9.8M (peak: 10.1M)
        CPU: 52ms
     CGroup: /system.slice/hello.service
             └─4821 /usr/bin/python3 /opt/hello/app.py

Nov 12 10:02:41 server1 systemd[1]: Started hello.service - Hello Python web app.
Nov 12 10:02:41 server1 python3[4821]: Starting on port 8080

$ curl -sI http://localhost:8080 | head -1
HTTP/1.0 200 OK
```

`verify` কোনো সমস্যা না পেলে কিছুই প্রিন্ট করে না।

### ধাপ ৫: auto-restart পরীক্ষা

```
$ sudo kill -9 4821
$ sleep 6; systemctl status hello --no-pager | head -3
● hello.service - Hello Python web app
     Loaded: loaded (/etc/systemd/system/hello.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-12 10:04:12 +06; 1s ago

$ journalctl -u hello -n 5 --no-pager
Nov 12 10:04:07 server1 systemd[1]: hello.service: Main process exited, code=killed, status=9/KILL
Nov 12 10:04:07 server1 systemd[1]: hello.service: Failed with result 'signal'.
Nov 12 10:04:12 server1 systemd[1]: hello.service: Scheduled restart job, restart counter is at 1.
Nov 12 10:04:12 server1 systemd[1]: Started hello.service - Hello Python web app.
```

process মারা যাওয়ার ৫ সেকেন্ড পর (`RestartSec=5`) systemd নিজে থেকেই নতুন PID-এ আবার চালু করেছে।

> 💡 **টিপস:** `print()`-এ `flush=True` দিয়েছি কারণ Python stdout pipe-এ গেলে buffer করে রাখে, ফলে লগ দেরিতে আসে। বিকল্প: `Environment="PYTHONUNBUFFERED=1"`।

> 📝 **মনে রাখো:** `After=network-online.target` একা যথেষ্ট না; ক্রম ঠিক করে শুধু, target-কে টেনে আনে না। তাই সাথে `Wants=network-online.target` দিতে হয়। কেন, সেটা পরের অংশে।

### একটা oneshot উদাহরণ

বুটে একবার কোনো স্ক্রিপ্ট চালাতে:

```ini
# /etc/systemd/system/boot-note.service
[Unit]
Description=Write boot time to a file

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'date >> /var/log/boot-times.log'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

> ⚠️ **সতর্কতা:** `ExecStart=` কোনো শেল না। `>>`, `|`, `&&`, `*` এর মতো শেল বৈশিষ্ট্য সরাসরি কাজ করে না। দরকার হলে উপরের মতো `/bin/sh -c '...'` দিয়ে মোড়াও, অথবা একটা আলাদা স্ক্রিপ্ট ফাইল বানাও। তবে `$VAR` ও `${VAR}` ভেরিয়েবল systemd নিজেই বদলে দেয়।

---

## ২৬.৯ Override: `systemctl edit` দিয়ে drop-in

ধরো nginx-এর unit-এ `Restart=` নেই, কিন্তু তুমি চাও crash করলে আবার চালু হোক। মূল ফাইল এডিট করা নিষেধ (আপডেটে মুছে যাবে)। সমাধান: **drop-in** ফাইল। এটা মূল ফাইলের উপরে শুধু তোমার পরিবর্তনগুলো "চাপিয়ে" দেয়।

```
$ sudo systemctl edit nginx
```

একটা এডিটর খুলবে (Ubuntu-তে সাধারণত nano; `SYSTEMD_EDITOR` বা `EDITOR` ভেরিয়েবল দিয়ে বদলানো যায়)। মাঝখানে লেখার জায়গা থাকবে, নিচে মন্তব্য হিসেবে মূল ফাইল:

```ini
### Editing /etc/systemd/system/nginx.service.d/override.conf
### Anything between here and the comment below will become the contents of the drop-in file

[Service]
Restart=on-failure
RestartSec=3

### Edits below this comment will be discarded
...
```

সংরক্ষণ করে বেরোলে systemd নিজেই `/etc/systemd/system/nginx.service.d/override.conf` তৈরি করে এবং `daemon-reload` করে। এরপর শুধু restart:

```
$ sudo systemctl restart nginx
$ systemctl cat nginx
# /usr/lib/systemd/system/nginx.service
[Unit]
Description=A high performance web server and a reverse proxy server
...

# /etc/systemd/system/nginx.service.d/override.conf
[Service]
Restart=on-failure
RestartSec=3
```

```
   চূড়ান্ত কনফিগারেশন = মূল ফাইল + drop-in

   /usr/lib/systemd/system/nginx.service   ──┐
                                             ├──► মেমরিতে একত্রিত unit
   /etc/systemd/system/nginx.service.d/      │
        override.conf                     ───┘   (drop-in-এর মান জিতবে)
```

### list-ধরনের key রিসেট করা

কিছু key (যেমন `ExecStart=`, `Environment=`, `After=`) একাধিকবার লেখা যায়, আর প্রতিবার **যোগ** হয়। তাই `ExecStart=` বদলাতে হলে প্রথমে খালি `ExecStart=` দিয়ে পুরোনোটা মুছতে হয়:

```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on; worker_processes 2;'
```

খালি লাইনটা না দিলে, `Type=simple`/`exec` সার্ভিসে দুটো `ExecStart` থাকার কারণে ভুল দেখাবে: `Service has more than one ExecStart= setting, which is only allowed for Type=oneshot services.`

### অন্যান্য edit অপশন

| কমান্ড | কাজ |
|---|---|
| `systemctl edit UNIT` | `override.conf` drop-in তৈরি/এডিট |
| `systemctl edit --full UNIT` | মূল ফাইলের একটা পূর্ণ কপি `/etc/systemd/system/`-এ বানিয়ে এডিট (প্যাকেজের ভবিষ্যৎ পরিবর্তন আর পাবে না) |
| `systemctl edit --drop-in=NAME UNIT` | ভিন্ন নামের drop-in ফাইল |
| `systemctl edit --force --full NEW.service` | একদম নতুন unit ফাইল তৈরি |
| `systemctl edit --runtime UNIT` | `/run`-এ অস্থায়ী override (রিবুটে মুছে যায়) |
| `systemctl revert UNIT` | `/etc` ও `/run`-এর সব override মুছে প্যাকেজের মূলে ফেরা |
| `systemd-delta` | সিস্টেমের সব override/masked/extended unit-এর তালিকা |

```
$ systemd-delta --type=extended
[EXTENDED]   /usr/lib/systemd/system/nginx.service → /etc/systemd/system/nginx.service.d/override.conf

1 overridden configuration files found.
```

> 💡 **টিপস:** drop-in ফোল্ডারে একাধিক `.conf` ফাইল থাকলে সেগুলো নামের বর্ণানুক্রমে পড়া হয়। তাই `10-restart.conf`, `20-limits.conf` এভাবে নাম দিলে ক্রম নিয়ন্ত্রণ সহজ।

---

## ২৬.১০ Dependency: Wants, Requires, After ও অন্যরা

systemd-এ দুটো সম্পূর্ণ আলাদা প্রশ্ন আছে, আর এই পার্থক্য বোঝাই এই অধ্যায়ের সবচেয়ে গুরুত্বপূর্ণ ধারণাগুলোর একটা:

1. **requirement (প্রয়োজন):** A চালু হলে B-কেও চালু করতে হবে কি? → `Wants=`, `Requires=`
2. **ordering (ক্রম):** দুটোই চালু হলে কে আগে? → `After=`, `Before=`

এ দুটো একে অপরের থেকে **স্বাধীন**।

```
শুধু Requires=B (After নেই):    A আর B একসাথে (সমান্তরালে) চালু হয়
                                 A ████
                                 B ████

Requires=B + After=B:            B পুরো চালু হওয়ার পর A
                                 B ████
                                 A     ████

শুধু After=B (Requires নেই):     B-কে কেউ চালু না করলে A একাই চলে;
                                 B-ও চালু হলে, তবে B আগে
```

> 💡 **উপমা:** বিয়ের অনুষ্ঠান। "বাবুর্চি লাগবেই" = Requires। "ফটোগ্রাফার থাকলে ভালো, না এলেও অনুষ্ঠান চলবে" = Wants। "রান্না শেষ হওয়ার পর খাবার পরিবেশন" = After। বাবুর্চি ডাকা আর কখন পরিবেশন হবে, দুটো আলাদা সিদ্ধান্ত।

### requirement directive

| Directive | অর্থ | B ব্যর্থ হলে A? | B থামানো হলে A? |
|---|---|---|---|
| `Wants=B` | B-কে চালু করার চেষ্টা করো | A চলবে | A চলবে |
| `Requires=B` | B অবশ্যই লাগবে | A চালু হবে না (যদি `After=B`-ও থাকে) | A-ও থামবে |
| `Requisite=B` | B আগে থেকেই চালু থাকতে হবে, নিজে চালু করবে না | A চালু হবে না | — |
| `BindsTo=B` | Requires-এর চেয়েও শক্ত; B হঠাৎ inactive হলেও A থামবে | A চালু হবে না | A-ও থামবে |
| `PartOf=B` | B stop/restart হলে A-ও stop/restart হবে (একমুখী, B-কে চালু করে না) | — | A-ও থামবে |
| `Conflicts=B` | A চালু হলে B থামবে, আর উল্টোটা | — | — |

### ordering directive

| Directive | অর্থ |
|---|---|
| `After=B` | B চালু হওয়া শেষ হলে তবেই A চালু শুরু; বন্ধের সময় উল্টো ক্রম (A আগে বন্ধ) |
| `Before=B` | A চালু শেষ হলে তবেই B |

> 📝 **মনে রাখো:** systemd-এর ডকুমেন্টেশন নিজেই বলে, বেশিরভাগ ক্ষেত্রে `Wants=` + `After=` ব্যবহার করাই সবচেয়ে ভালো। `Requires=` সিস্টেমকে ভঙ্গুর করে: একটা ছোট সার্ভিস ব্যর্থ হলে পুরো চেইন পড়ে যায়।

> 📝 **মনে রাখো:** বেশিরভাগ service unit-এ systemd স্বয়ংক্রিয়ভাবে কিছু dependency যোগ করে (`DefaultDependencies=yes`): যেমন `Requires=sysinit.target`, `After=sysinit.target basic.target`, `Conflicts=shutdown.target`, `Before=shutdown.target`। তাই সাধারণ সার্ভিসে এগুলো নিজে লিখতে হয় না।

### network.target বনাম network-online.target

| Target | অর্থ |
|---|---|
| `network.target` | নেটওয়ার্ক ম্যানেজমেন্ট সফটওয়্যার চালু হয়েছে; IP পাওয়া গেছে এমন নিশ্চয়তা **নেই**। মূলত শাটডাউনের সময় নেটওয়ার্ক শেষে বন্ধ হওয়ার জন্য |
| `network-online.target` | নেটওয়ার্ক কনফিগার হয়ে সত্যিই ব্যবহারযোগ্য (`NetworkManager-wait-online` বা `systemd-networkd-wait-online` দিয়ে নির্ধারিত) |

তোমার অ্যাপ চালু হওয়ার সময়ই কোনো দূরের ডাটাবেসে সংযোগ করতে চাইলে `Wants=network-online.target` + `After=network-online.target` দাও।

### dependency দেখা

```
$ systemctl list-dependencies hello
hello.service
● ├─system.slice
● ├─network-online.target
● │ └─NetworkManager-wait-online.service
● └─sysinit.target
●   ├─apparmor.service
●   ├─dev-hugepages.mount
...

$ systemctl show hello -p Wants -p After
Wants=network-online.target
After=network-online.target sysinit.target basic.target system.slice systemd-journald.socket
```

---

## ২৬.১১ User service: `systemctl --user`

systemd প্রতিটি লগইন করা ইউজারের জন্য একটা নিজস্ব systemd instance (`user@UID.service`) চালায়। এতে sudo ছাড়াই নিজের সার্ভিস চালানো যায়।

```
$ mkdir -p ~/.config/systemd/user
$ nano ~/.config/systemd/user/notes-sync.service
```

```ini
[Unit]
Description=Sync my notes folder

[Service]
Type=oneshot
ExecStart=/usr/bin/rsync -a %h/notes/ %h/backup/notes/
```

(`%h` = ইউজারের home ফোল্ডার। আরও specifier: `%u` = ইউজারনেম, `%i` = instance নাম, `%n` = পুরো unit নাম।)

```
$ systemctl --user daemon-reload
$ systemctl --user start notes-sync
$ systemctl --user status notes-sync
$ journalctl --user -u notes-sync
```

> ⚠️ **সতর্কতা:** ডিফল্টভাবে ইউজার লগআউট করলে তার user systemd instance (এবং সব user service) বন্ধ হয়ে যায়। সার্ভারে লগআউটের পরও চালু রাখতে **lingering** চালু করো:
>
> ```
> $ sudo loginctl enable-linger $USER
> $ loginctl show-user $USER -p Linger
> Linger=yes
> ```

> 💡 **টিপস:** user unit-এ `WantedBy=default.target` দাও (user instance-এ `multi-user.target` নেই)।

---

## ২৬.১২ রিসোর্স সীমা ও নিরাপত্তা (sandboxing)

প্রতিটি সার্ভিস নিজের cgroup-এ চলে বলে systemd সহজেই সীমা বসাতে পারে।

```ini
[Service]
MemoryMax=512M        # এর বেশি মেমরি চাইলে OOM kill
CPUQuota=50%          # একটা core-এর সর্বোচ্চ ৫০%
TasksMax=100          # সর্বোচ্চ ১০০ process/thread
LimitNOFILE=65536     # খোলা ফাইলের সীমা (ulimit -n)
```

লাইভ সীমা বসানো (unit ফাইল না বদলে):

```
$ sudo systemctl set-property hello MemoryMax=256M
```

এটা `/etc/systemd/system.control/hello.service.d/` এ একটা drop-in লিখে রাখে, তাই রিবুটের পরেও টিকে থাকে (`--runtime` দিলে টিকে থাকে না)।

cgroup অনুযায়ী লাইভ রিসোর্স ব্যবহার দেখতে:

```
$ systemd-cgtop          # top-এর মতো, কিন্তু সার্ভিস অনুযায়ী
$ systemd-cgls           # cgroup গাছ: কোন সার্ভিসের অধীনে কোন process
```

**নিরাপত্তা directive** (বিস্তারিত `man systemd.exec`):

| Directive | কাজ |
|---|---|
| `NoNewPrivileges=yes` | process ও তার child-রা setuid ইত্যাদি দিয়ে নতুন ক্ষমতা পাবে না |
| `PrivateTmp=yes` | নিজস্ব আলাদা `/tmp` ও `/var/tmp` |
| `ProtectSystem=strict` | পুরো ফাইলসিস্টেম read-only (শুধু `/dev`, `/proc`, `/sys` বাদে); লিখতে হলে `ReadWritePaths=` |
| `ProtectSystem=full` | `/usr`, `/boot`, `/etc` read-only |
| `ProtectHome=yes` | `/home`, `/root`, `/run/user` অদৃশ্য |
| `PrivateDevices=yes` | আসল হার্ডওয়্যার ডিভাইস অদৃশ্য |
| `DynamicUser=yes` | চলার সময় একটা অস্থায়ী ইউজার তৈরি করে চালায় |
| `StateDirectory=NAME` | `/var/lib/NAME` তৈরি করে সার্ভিসের ইউজারকে মালিক বানায় |
| `CapabilityBoundingSet=` | কোন Linux capability থাকবে |

সার্ভিস কতটা সুরক্ষিত, তার একটা নম্বর পাবে:

```
$ systemd-analyze security hello
  NAME                         DESCRIPTION                                   EXPOSURE
✓ NoNewPrivileges=             Service processes cannot acquire new privileges
✓ PrivateTmp=                  Service has no access to other software's temporary files   
✗ PrivateDevices=              Service potentially has access to hardware devices    0.2
...
→ Overall exposure level for hello.service: 6.3 MEDIUM 😐
```

(নম্বর ০ = সবচেয়ে নিরাপদ, ১০ = সবচেয়ে অরক্ষিত। আসল সংখ্যা তোমার unit ও systemd সংস্করণভেদে আলাদা হবে।) নিরাপত্তা নিয়ে আরও [অধ্যায় ৪৫](45-security-hardening.md)-এ।

---

## ২৬.১৩ systemd-analyze: বুট বিশ্লেষণ ও যাচাই

```
$ systemd-analyze
Startup finished in 4.120s (firmware) + 2.310s (loader) + 1.902s (kernel) + 6.744s (userspace) = 15.077s
graphical.target reached after 6.701s in userspace.
```

(`systemd-analyze` আর `systemd-analyze time` একই। firmware/loader অংশ শুধু UEFI সিস্টেমে দেখায়।)

**কোন সার্ভিস সবচেয়ে বেশি সময় নিল:**

```
$ systemd-analyze blame
3.211s NetworkManager-wait-online.service
1.104s snapd.service
 812ms dev-nvme0n1p2.device
 640ms systemd-journal-flush.service
 402ms apparmor.service
...
```

> ⚠️ **সতর্কতা:** `blame`-এ বেশি সময় মানেই বুট ধীর হওয়ার জন্য দায়ী, এমন নয়। অনেক সার্ভিস সমান্তরালে চলে, আর কেউ কেউ শুধু অপেক্ষা করে। আসল বাধা খুঁজতে `critical-chain` দেখো।

**critical chain:** যে ক্রমিক শিকলটা শেষ target পৌঁছাতে দেরি করিয়েছে:

```
$ systemd-analyze critical-chain
The time when unit became active or started is printed after the "@" character.
The time the unit took to start is printed after the "+" character.

graphical.target @6.701s
└─multi-user.target @6.700s
  └─snapd.seeded.service @5.420s +1.270s
    └─snapd.service @4.300s +1.104s
      └─basic.target @1.250s
        └─sockets.target @1.249s
...
```

**অন্যান্য দরকারি subcommand:**

| কমান্ড | কাজ |
|---|---|
| `systemd-analyze time` | মোট বুট সময় |
| `systemd-analyze blame` | প্রতি unit-এর start সময় (বেশি থেকে কম) |
| `systemd-analyze critical-chain [UNIT]` | সময়-নির্ণায়ক শিকল |
| `systemd-analyze plot > boot.svg` | পুরো বুটের একটা SVG টাইমলাইন ছবি (ব্রাউজারে খোলো) |
| `systemd-analyze dot UNIT \| dot -Tsvg > g.svg` | dependency গ্রাফ (graphviz লাগে) |
| `systemd-analyze verify FILE` | unit ফাইলের ভুল খোঁজো |
| `systemd-analyze security [UNIT]` | নিরাপত্তা স্কোর |
| `systemd-analyze calendar 'Mon *-*-* 09:00'` | timer-এর সময় এক্সপ্রেশন যাচাই ([অধ্যায় ২৯](29-scheduling-cron.md)) |
| `systemd-analyze timespan 1h30m` | সময়ের মান কীভাবে পড়া হবে |
| `systemd-analyze unit-paths` | unit ফাইল খোঁজার সব ফোল্ডার |
| `systemd-analyze cat-config systemd/journald.conf` | কনফিগ ফাইল ও তার drop-in একসাথে |

```
$ systemd-analyze calendar 'Mon..Fri 09:00'
  Original form: Mon..Fri 09:00
Normalized form: Mon..Fri *-*-* 09:00:00
    Next elapse: Fri 2026-11-13 09:00:00 +06
       (in UTC): Fri 2026-11-13 03:00:00 UTC
       From now: 22h left
```

> 💡 **টিপস:** বুট ধীর হলে প্রায়ই অপরাধী হয় `NetworkManager-wait-online.service` বা `systemd-networkd-wait-online.service`। ডেস্কটপে কোনো সার্ভিস নেটওয়ার্কের জন্য অপেক্ষা না করলে এটা disable করা যায়, তবে সার্ভারে সাবধান: যে সার্ভিসগুলো `network-online.target`-এর উপর নির্ভর করে, তারা আগেভাগে চালু হয়ে ব্যর্থ হতে পারে।

---

## ২৬.১৪ hostnamectl, timedatectl, localectl

systemd কিছু সিস্টেম সেটিং বদলানোর জন্য ছোট ছোট daemon (hostnamed, timedated, localed) ও তাদের কমান্ড দেয়। এগুলো সঠিক কনফিগ ফাইলে লেখে এবং চলমান সিস্টেমকেও সাথে সাথে জানায়, তাই ফাইল নিজে এডিট করার চেয়ে নিরাপদ।

### hostnamectl: কম্পিউটারের নাম

```
$ hostnamectl
 Static hostname: server1
       Icon name: computer-vm
         Chassis: vm 🖴
      Machine ID: 3f2a9c0d1e5b4c7a8f6e2d1c0b9a8f7e
         Boot ID: 9b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e
  Virtualization: kvm
Operating System: Ubuntu 24.04.1 LTS
          Kernel: Linux 6.8.0-48-generic
    Architecture: x86-64
```

hostname তিন ধরনের:

| ধরন | অর্থ | কোথায় থাকে |
|---|---|---|
| static | স্থায়ী নাম, বুটে সেট হয় | `/etc/hostname` |
| transient | চলার সময় DHCP ইত্যাদি থেকে আসা অস্থায়ী নাম | kernel |
| pretty | মানুষের জন্য সুন্দর নাম, স্পেস/ইউনিকোড চলে | `/etc/machine-info` |

```
$ sudo hostnamectl set-hostname web01
$ sudo hostnamectl set-hostname "নিশাতের ল্যাপটপ" --pretty
$ hostnamectl hostname
web01
```

> ⚠️ **সতর্কতা:** hostname বদলানোর পর `/etc/hosts` ফাইলে পুরোনো নামটা থাকলে (Ubuntu-তে সাধারণত `127.0.1.1 oldname` লাইন) সেটাও বদলে দাও, নইলে `sudo` চালাতে দেরি হবে ও `unable to resolve host` সতর্কবার্তা দেখাবে। নতুন নাম শেলের prompt-এ দেখতে নতুন টার্মিনাল খোলো।

### timedatectl: সময়, তারিখ ও timezone

```
$ timedatectl
               Local time: Thu 2026-11-12 10:30:15 +06
           Universal time: Thu 2026-11-12 04:30:15 UTC
                 RTC time: Thu 2026-11-12 04:30:15
                Time zone: Asia/Dhaka (+06, +0600)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

| কমান্ড | কাজ |
|---|---|
| `timedatectl list-timezones` | সব timezone (`grep Asia` দিয়ে ছাঁকো) |
| `sudo timedatectl set-timezone Asia/Dhaka` | timezone সেট |
| `sudo timedatectl set-ntp true` | NTP দিয়ে স্বয়ংক্রিয় সময় মেলানো চালু |
| `sudo timedatectl set-ntp false` | বন্ধ |
| `sudo timedatectl set-time '2026-11-12 10:30:00'` | হাতে সময় সেট (NTP বন্ধ থাকলেই কেবল কাজ করে) |
| `timedatectl timesync-status` | systemd-timesyncd-এর বিস্তারিত (শুধু timesyncd ব্যবহার হলে) |
| `sudo timedatectl set-local-rtc 0` | হার্ডওয়্যার ঘড়ি UTC-তে রাখো (প্রস্তাবিত) |

```
$ timedatectl list-timezones | grep -i dhaka
Asia/Dhaka
```

timezone আসলে `/etc/localtime` নামের একটা symlink (`/usr/share/zoneinfo/Asia/Dhaka`-এর দিকে)।

> 📝 **মনে রাখো (distro পার্থক্য):** Ubuntu ও Debian ডিফল্টভাবে NTP-র জন্য `systemd-timesyncd` ব্যবহার করে; Fedora ও RHEL ব্যবহার করে `chronyd` (`chronyc tracking` দিয়ে দেখো)। `timedatectl set-ntp` দুটোর সাথেই কাজ করে।

> 💡 **টিপস:** Windows ও Linux dual-boot করলে ঘড়ি ৬ ঘণ্টা আগে-পিছে হয়ে যায়, কারণ Windows হার্ডওয়্যার ঘড়িকে local time ধরে আর Linux ধরে UTC। সবচেয়ে ভালো সমাধান Windows-কে UTC ব্যবহার করানো; সহজ সমাধান Linux-এ `sudo timedatectl set-local-rtc 1` (তবে systemd এতে সতর্কবার্তা দেয়)।

### localectl: ভাষা ও কীবোর্ড

```
$ localectl
System Locale: LANG=en_US.UTF-8
    VC Keymap: us
   X11 Layout: us
```

| কমান্ড | কাজ |
|---|---|
| `localectl list-locales` | উপলব্ধ (জেনারেট করা) locale |
| `sudo localectl set-locale LANG=bn_BD.UTF-8` | সিস্টেমের ভাষা |
| `localectl list-keymaps` | কনসোলের কীবোর্ড ম্যাপ |
| `sudo localectl set-keymap us` | কনসোল (VC) কীবোর্ড |
| `sudo localectl set-x11-keymap us,bd pc105 "" grp:alt_shift_toggle` | GUI কীবোর্ড: দুটি লেআউট, Alt+Shift দিয়ে বদল |

> 📝 **মনে রাখো:** Debian/Ubuntu-তে কোনো locale `localectl list-locales`-এ না থাকলে আগে সেটা জেনারেট করতে হয়: `sudo locale-gen bn_BD.UTF-8` (বা `sudo dpkg-reconfigure locales`)। Fedora/RHEL-এ ভাষা প্যাক ইনস্টল করো: `sudo dnf install glibc-langpack-bn`। সেটিং থাকে Debian/Ubuntu-তে `/etc/default/locale`, Fedora/RHEL-এ `/etc/locale.conf`।

> 💡 **টিপস:** সার্ভারে সিস্টেম locale `en_US.UTF-8` বা `C.UTF-8` রাখাই ভালো; বাংলা locale দিলে error বার্তা বাংলায় আসে আর ইন্টারনেটে সমাধান খোঁজা কঠিন হয়। ডেস্কটপে নিজের পছন্দ মতো রাখো।

অন্যান্য একই ধরনের `*ctl` টুল: `loginctl` (লগইন সেশন), `networkctl` (systemd-networkd), `resolvectl` (DNS; [অধ্যায় ৩৭](37-network-commands.md)), `journalctl` ([অধ্যায় ২৮](28-logs.md)), `coredumpctl` (crash dump), `bootctl` (systemd-boot)।

---

## ২৬.১৫ সমস্যা সমাধানের কর্মপদ্ধতি

সার্ভিস চালু হচ্ছে না? এই ক্রমে দেখো:

```
১. systemctl status myapp            ← অবস্থা ও শেষ লগ
        │
২. journalctl -xeu myapp             ← পুরো লগ, ব্যাখ্যা সহ, শেষ থেকে
        │
৩. systemctl cat myapp               ← কার্যকর কনফিগ কী (override সহ)?
        │
৪. systemd-analyze verify ...        ← সিনট্যাক্স ভুল?
        │
৫. হাতে চালিয়ে দেখো:                  ← একই ইউজার হিসেবে
   sudo -u hello /usr/bin/python3 /opt/hello/app.py
        │
৬. ঠিক করে: daemon-reload → reset-failed → restart
```

**সাধারণ ভুল ও তার মানে:**

| বার্তা / লক্ষণ | সম্ভাব্য কারণ |
|---|---|
| `Unit myapp.service not found.` | নামে ভুল, ফাইল ভুল ফোল্ডারে, বা `daemon-reload` হয়নি |
| `status=203/EXEC` | `ExecStart`-এর প্রোগ্রাম পাওয়া যায়নি বা executable না (`chmod +x`, absolute path দাও, স্ক্রিপ্টে shebang আছে কি না দেখো) |
| `status=217/USER` | `User=`-এ দেওয়া ইউজার নেই |
| `status=200/CHDIR` | `WorkingDirectory` নেই বা অ্যাক্সেস নেই |
| `status=226/NAMESPACE` | sandbox directive-এ দেওয়া path নেই |
| `Permission denied` | ফাইলের মালিকানা/পারমিশন ([অধ্যায় ১৬](16-permissions.md)), বা SELinux/AppArmor |
| `start request repeated too quickly` | বারবার crash; `reset-failed` করে আসল কারণ খোঁজো |
| active (exited) কিন্তু আসলে চলছে না | প্রোগ্রাম নিজে background-এ চলে গেছে অথচ `Type=simple` দেওয়া; `Type=forking` লাগবে বা foreground মোড দাও |
| timeout হয়ে failed | `Type=notify` কিন্তু প্রোগ্রাম READY পাঠায় না; বা `forking` কিন্তু parent exit করে না |

---

## ২৬.১৬ কীবোর্ড শর্টকাট

`systemctl status`, `list-units`, `journalctl` এর আউটপুট লম্বা হলে **less** pager-এ খোলে। সেখানে এই কী-গুলো কাজে লাগে (less নিয়ে বিস্তারিত [অধ্যায় ১০](10-file-dekha.md)):

| শর্টকাট | কাজ |
|---|---|
| `q` | pager থেকে বের হও |
| `Space` / `PageDown` | এক পাতা নিচে |
| `b` / `PageUp` | এক পাতা উপরে |
| `↓` / `j`, `↑` / `k` | এক লাইন নিচে/উপরে |
| `→` / `←` | লম্বা লাইনে ডানে/বামে স্ক্রল (systemctl লাইন কাটে না, ডানে সরায়) |
| `g` / `G` | একদম শুরুতে / শেষে |
| `/শব্দ` | নিচের দিকে খোঁজো; `n` পরেরটা, `N` আগেরটা |
| `?শব্দ` | উপরের দিকে খোঁজো |
| `Ctrl+C` | `journalctl -f` বা `status` এর লাইভ অনুসরণ থামাও |
| `Tab` | unit নাম, subcommand auto-complete (bash-completion প্যাকেজ লাগে) |

সিস্টেম-স্তরের শর্টকাট:

| শর্টকাট | কাজ |
|---|---|
| `Ctrl+Alt+Del` (কনসোলে) | `ctrl-alt-del.target` চালু করে, ডিফল্টভাবে রিবুট; ২ সেকেন্ডের মধ্যে ৭ বার চাপলে তাৎক্ষণিক জোরপূর্বক রিবুট |
| `Ctrl+Alt+F1`…`F6` | ভিন্ন virtual terminal (প্রতিটির পিছনে `getty@ttyN.service`) |
| GRUB-এ `e` | বুট এন্ট্রি এডিট (যেমন `systemd.unit=rescue.target` যোগ) |
| GRUB এডিটে `Ctrl+X` / `F10` | এডিট করা এন্ট্রিতে বুট |
| `systemctl edit`-এ nano: `Ctrl+O`, `Enter`, `Ctrl+X` | সংরক্ষণ ও বের হওয়া ([অধ্যায় ১৩](13-nano-editor.md)) |

> ⚠️ **সতর্কতা:** সার্ভারে কেউ যেন ভুল করে Ctrl+Alt+Del চেপে রিবুট না করে, সেজন্য অনেকে এটা mask করে: `sudo systemctl mask ctrl-alt-del.target`।

> 💡 **টিপস:** pager একেবারেই না চাইলে `--no-pager` দাও, অথবা `export SYSTEMD_PAGER=` সেট করো। স্থায়ীভাবে রাখতে `.bashrc`-এ লেখো ([অধ্যায় ৪৪](44-environment-bashrc.md))। আর `systemctl`-কে বারবার লিখতে বিরক্ত লাগলে alias: `alias sc='sudo systemctl'`।

---

## হাতে-কলমে অনুশীলন

> 💡 একটা VM বা পরীক্ষামূলক মেশিনে করো ([অধ্যায় ৩](03-distro-o-installation.md)), প্রোডাকশন সার্ভারে নয়।

### ল্যাব ১: সিস্টেম ঘুরে দেখা

```
$ ps -p 1 -o comm=
$ systemctl --version | head -1
$ systemctl get-default
$ systemctl --failed
$ systemctl list-units --type=service --state=running --no-pager | wc -l
$ systemctl list-unit-files --state=enabled --no-pager
$ systemctl list-timers --no-pager
```

প্রশ্ন: কয়টা সার্ভিস চলছে? কোন timer সবার আগে চলবে?

### ল্যাব ২: একটা সার্ভিসের পুরো জীবনচক্র

```
$ sudo apt install -y nginx          # Fedora/RHEL: sudo dnf install -y nginx
$ systemctl is-enabled nginx         # Ubuntu-তে enabled, Fedora-তে disabled
$ sudo systemctl enable --now nginx
$ curl -sI localhost | head -1
$ sudo systemctl stop nginx; systemctl is-active nginx
$ sudo systemctl mask nginx; sudo systemctl start nginx    # ব্যর্থ হবে
$ sudo systemctl unmask nginx; sudo systemctl start nginx
$ ls -l /etc/systemd/system/multi-user.target.wants/ | grep nginx
```

### ল্যাব ৩: নিজের সার্ভিস

২৬.৮-এর `hello.service` পুরোটা তৈরি করো। তারপর:

1. `kill -9` দিয়ে মেরে দেখো আবার উঠে কি না।
2. `app.py`-তে ইচ্ছে করে সিনট্যাক্স ভুল ঢুকিয়ে restart করো; `journalctl -u hello` দেখে ভুল খোঁজো। `start request repeated too quickly` দেখা পর্যন্ত অপেক্ষা করো। ঠিক করে `reset-failed` ও `start`।
3. `systemd-analyze security hello` চালাও, তারপর `PrivateDevices=yes` যোগ করে আবার চালাও; স্কোর কমলো কি না দেখো।

### ল্যাব ৪: override

```
$ sudo systemctl edit hello
```

ভেতরে লেখো:

```ini
[Service]
Environment="PORT=9090"
```

```
$ sudo systemctl restart hello
$ systemctl cat hello
$ curl -sI localhost:9090 | head -1
$ systemd-delta --type=extended
$ sudo systemctl revert hello && sudo systemctl restart hello
$ curl -sI localhost:8080 | head -1        # আবার পুরোনো port-এ
```

`Environment=` একাধিকবার দিলে পরেরটা একই ভেরিয়েবলের আগের মান বদলে দেয়, তাই এখানে খালি লাইন লাগেনি।

### ল্যাব ৫: dependency পরীক্ষা

দুটো oneshot সার্ভিস বানাও: `a.service` ও `b.service`, প্রতিটি `ExecStart=/bin/sh -c 'echo X; sleep 3'`। `a`-তে প্রথমে শুধু `Wants=b.service` দিয়ে `sudo systemctl start a`, `journalctl -u a -u b` দেখে লক্ষ করো দুটো একসাথে শুরু হলো। তারপর `After=b.service` যোগ করে (daemon-reload) আবার চালাও; এবার `b` শেষ হওয়ার পর `a` শুরু হবে।

### ল্যাব ৬: বুট বিশ্লেষণ ও সিস্টেম সেটিং

```
$ systemd-analyze
$ systemd-analyze blame | head -5
$ systemd-analyze critical-chain
$ systemd-analyze plot > ~/boot.svg      # ফাইলটা ব্রাউজারে খোলো
$ hostnamectl
$ sudo hostnamectl set-hostname lab01 && hostnamectl hostname
$ timedatectl
$ sudo timedatectl set-timezone Asia/Dhaka
$ localectl
```

### ল্যাব ৭: user service ও target

1. ২৬.১১-এর `notes-sync.service` তৈরি করে `systemctl --user` দিয়ে চালাও।
2. VM-এ `sudo systemctl isolate multi-user.target` চালিয়ে দেখো GUI বন্ধ হয় কি না, তারপর `sudo systemctl isolate graphical.target` (SSH-এ করবে না)।

---

## সারসংক্ষেপ

| কমান্ড | কাজ |
|---|---|
| `systemctl status UNIT` | অবস্থা, PID, মেমরি, শেষ লগ |
| `systemctl start / stop UNIT` | এখনই চালু / বন্ধ |
| `systemctl restart UNIT` | বন্ধ করে আবার চালু |
| `systemctl reload UNIT` | process না থামিয়ে কনফিগ পড়াও |
| `systemctl enable / disable UNIT` | বুটে চালু হবে / হবে না (symlink) |
| `systemctl enable --now UNIT` | enable + start |
| `systemctl is-active / is-enabled / is-failed UNIT` | এক-শব্দের উত্তর, স্ক্রিপ্টের জন্য |
| `systemctl mask / unmask UNIT` | সম্পূর্ণ নিষিদ্ধ / নিষেধ তোলা |
| `systemctl list-units [--type=service] [--state=…]` | লোড হওয়া unit |
| `systemctl list-unit-files` | ডিস্কের সব unit ও enable অবস্থা |
| `systemctl --failed` | ব্যর্থ unit |
| `systemctl cat / show UNIT` | unit ফাইল / property |
| `systemctl list-dependencies UNIT` | dependency গাছ |
| `systemctl daemon-reload` | unit ফাইল বদলানোর পর আবার পড়াও |
| `systemctl edit UNIT` | drop-in override |
| `systemctl edit --full UNIT` | পূর্ণ কপি এডিট |
| `systemctl revert UNIT` | override মুছে ফেলো |
| `systemctl reset-failed` | failed অবস্থা মুছে দাও |
| `systemctl get-default / set-default TARGET` | ডিফল্ট target |
| `systemctl isolate TARGET` | এখনই target বদলাও |
| `systemctl --user …` | নিজের user service |
| `systemd-analyze [blame / critical-chain / plot]` | বুট সময় বিশ্লেষণ |
| `systemd-analyze verify / security` | unit যাচাই / নিরাপত্তা স্কোর |
| `systemd-delta` | override-এর তালিকা |
| `systemd-cgtop`, `systemd-cgls` | cgroup অনুযায়ী রিসোর্স / process গাছ |
| `journalctl -u UNIT [-f]` | সার্ভিসের লগ |
| `hostnamectl [set-hostname NAME]` | hostname |
| `timedatectl [set-timezone TZ / set-ntp true]` | সময় ও timezone |
| `localectl [set-locale / set-keymap]` | ভাষা ও কীবোর্ড |
| `loginctl enable-linger USER` | লগআউটের পরও user service চালু |

| ফাইল/ফোল্ডার | ভূমিকা |
|---|---|
| `/usr/lib/systemd/system/` (= `/lib/...`) | প্যাকেজের unit (এডিট কোরো না) |
| `/etc/systemd/system/` | অ্যাডমিনের unit ও override (সর্বোচ্চ অগ্রাধিকার) |
| `/etc/systemd/system/UNIT.d/*.conf` | drop-in |
| `/etc/systemd/system/*.target.wants/` | enable করা unit-এর symlink |
| `~/.config/systemd/user/` | user unit |

---

## নিজেকে যাচাই করো

1. PID 1 কী, এবং কেন এটা মারা গেলে পুরো সিস্টেম পড়ে যায়?
2. SysV init-এর তুলনায় systemd বুট দ্রুত করে কোন দুটো প্রধান কৌশলে?
3. `systemctl start` আর `systemctl enable`-এর পার্থক্য কী? `enable` ভেতরে আসলে কী করে?
4. `reload`, `restart` আর `daemon-reload` তিনটা কখন ব্যবহার করবে?
5. `disable` আর `mask`-এর পার্থক্য কী? mask করলে ফাইলসিস্টেমে কী তৈরি হয়?
6. runlevel 3 ও 5-এর systemd সমতুল্য target কী? পরের বুট থেকে GUI বন্ধ রাখতে কোন কমান্ড দেবে?
7. `/usr/lib/systemd/system/nginx.service` ও `/etc/systemd/system/nginx.service` দুটোই থাকলে কোনটা কার্যকর হবে? প্যাকেজের ফাইল সরাসরি এডিট করা কেন উচিত নয়?
8. drop-in-এ `ExecStart` বদলাতে কেন আগে একটা খালি `ExecStart=` লিখতে হয়?
9. `Wants=B` দেওয়া আছে কিন্তু `After=B` নেই। A ও B কোন ক্রমে চালু হবে?
10. একটা সার্ভিস `status=203/EXEC` দিয়ে ব্যর্থ হচ্ছে। কী কী পরীক্ষা করবে?

<details><summary>উত্তর দেখো</summary>

1. kernel যে প্রথম user-space process চালায়, সেটাই PID 1 (init, এখন systemd)। এটা বাকি সব সার্ভিস চালায়, orphan process দত্তক নেয় ও reap করে। PID 1 exit করলে kernel চালিয়ে যাওয়ার উপায় পায় না, তাই kernel panic হয়।
2. (ক) dependency না থাকলে সার্ভিসগুলো সমান্তরালে (parallel) চালানো, (খ) socket ও on-demand activation: socket আগে তৈরি করে রাখা, ফলে সার্ভিস প্রস্তুত হওয়ার আগেই অন্যরা সংযোগ করতে পারে এবং দরকার না হলে সার্ভিস চালুই হয় না।
3. `start` এখনই চালু করে কিন্তু রিবুটে টেকে না; `enable` বুটে চালু হওয়ার ব্যবস্থা করে কিন্তু এখন চালু করে না। `enable` `[Install]`-এর `WantedBy=` পড়ে সংশ্লিষ্ট target-এর `.wants/` ফোল্ডারে (যেমন `/etc/systemd/system/multi-user.target.wants/`) একটা symlink বানায়। দুটো একসাথে: `enable --now`।
4. `reload`: সার্ভিসের নিজের কনফিগ (যেমন nginx.conf) বদলালে, process না থামিয়ে। `restart`: process পুরো নতুন করে চালাতে (বাইনারি আপডেট, reload সমর্থন না করলে)। `daemon-reload`: systemd-এর unit ফাইল বা drop-in বদলালে, যাতে PID 1 নতুন definition পড়ে; এরপর সাধারণত restart-ও লাগে।
5. disable করা সার্ভিস হাতে বা অন্য unit-এর dependency হিসেবে চালু হতে পারে; masked সার্ভিস কোনোভাবেই চালু হয় না। mask করলে `/etc/systemd/system/UNIT` নামে `/dev/null`-এর দিকে একটা symlink তৈরি হয়।
6. runlevel 3 = `multi-user.target`, runlevel 5 = `graphical.target`। কমান্ড: `sudo systemctl set-default multi-user.target`।
7. `/etc/systemd/system/`-এরটা, কারণ এর অগ্রাধিকার সর্বোচ্চ। প্যাকেজের ফাইল এডিট করলে পরের প্যাকেজ আপডেটে পরিবর্তন মুছে যাবে; তাই `systemctl edit` দিয়ে drop-in বানাও।
8. `ExecStart=` একটা list-ধরনের key; প্রতিটি নতুন লাইন আগের মানের সাথে যোগ হয়। খালি `ExecStart=` আগের সব মান মুছে দেয়, তারপর নতুন মান বসে। না করলে simple/exec সার্ভিসে দুটো ExecStart হয়ে যায়, যা অনুমোদিত নয়।
9. কোনো নির্দিষ্ট ক্রম নেই, দুটো সমান্তরালে চালু হবে। `Wants`/`Requires` শুধু "চালু করো" বলে, ক্রম ঠিক করে `After`/`Before`।
10. `ExecStart`-এ দেওয়া path ঠিক ও absolute কি না, ফাইলটা আছে কি না, executable পারমিশন (`chmod +x`) আছে কি না, স্ক্রিপ্ট হলে প্রথম লাইনে সঠিক shebang (`#!/bin/bash`) আছে কি না, `User=`-এর ইউজারের ফাইলটা চালানোর অনুমতি আছে কি না, এবং ফাইলে Windows-এর CRLF লাইন-এন্ডিং আছে কি না। `journalctl -xeu UNIT`-এ বিস্তারিত দেখো।

</details>

---

[← আগের অধ্যায়](25-jobs-tmux.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](27-boot-process.md)

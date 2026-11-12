# অধ্যায় ২৮: লগ ও সিস্টেম মনিটরিং লগ

## এই অধ্যায়ে যা শিখবে

- লগ (log) কী, কেন দরকার, আর লিনাক্সে লগ কে লেখে
- `/var/log` ডিরেক্টরির গুরুত্বপূর্ণ ফাইলগুলো: `syslog`/`messages`, `auth.log`/`secure`, `kern.log`, `dpkg.log`, `nginx` লগ ইত্যাদি, আর distro-ভেদে পার্থক্য
- লিনাক্স লগিং-এর পুরো আর্কিটেকচার: kernel ring buffer → systemd-journald → rsyslog → ফাইল
- syslog-এর **facility** আর **severity** কী, এবং rsyslog কনফিগারেশন কীভাবে লেখে
- `journalctl` — `-u`, `-f`, `-b`, `-p`, `--since`, `--until`, `-k`, `-o json` সহ সব দরকারি অপশন
- journal-কে persistent (রিবুটের পরও টিকে থাকা) করা এবং তার সাইজ নিয়ন্ত্রণ করা
- `logrotate` দিয়ে লগ ফাইল ঘুরিয়ে-কেটে-কম্প্রেস করা
- `logger` ও `systemd-cat` দিয়ে নিজের স্ক্রিপ্ট থেকে লগ লেখা
- `dmesg` দিয়ে kernel-এর বার্তা দেখা
- `last`, `lastb`, `lastlog` দিয়ে লগইন ইতিহাস দেখা
- লগ দেখে সমস্যা খোঁজার একটি ধাপে ধাপে পদ্ধতি (troubleshooting methodology)

---

## ২৮.১ লগ কী এবং কেন?

ধরো তুমি একটি বড় অফিস বিল্ডিং-এর ম্যানেজার। গেটে একজন দারোয়ান খাতা রাখে: কে কখন ঢুকল, কে বের হলো, কে ভুল কার্ড দেখাল। লিফটের টেকনিশিয়ান আলাদা খাতায় লেখে লিফট কখন আটকে গিয়েছিল। ক্যান্টিনের খাতায় থাকে কখন গ্যাস শেষ হলো। কোনো একদিন যদি শোনো "কাল রাতে কেউ ঢুকেছিল", তাহলে তুমি প্রথমে কী করবে? খাতাগুলো খুলবে।

লিনাক্সের **লগ (log)** ঠিক এই খাতাগুলো। Log মানে হলো সময়সহ ঘটনার লিখিত রেকর্ড। সিস্টেমে যা যা ঘটে — কেউ লগইন করল, কোনো সার্ভিস চালু বা ক্র্যাশ হলো, ডিস্কে error এল, নেটওয়ার্ক কার্ড ডাউন হলো, কেউ ভুল পাসওয়ার্ড দিল — প্রায় সবকিছুই কোথাও না কোথাও লেখা হয়।

লগ কেন এত জরুরি:

| কারণ | উদাহরণ |
|---|---|
| **ট্রাবলশুটিং** (সমস্যা খোঁজা) | nginx চালু হচ্ছে না — লগে লেখা থাকবে কোন লাইনে কনফিগ ভুল |
| **নিরাপত্তা** (security) | কেউ রাতভর SSH-এ পাসওয়ার্ড অনুমান করছে কি না |
| **অডিট** (audit) | কে কখন `sudo` দিয়ে কী চালিয়েছে |
| **পারফরম্যান্স** | কোন সময়ে সার্ভার ধীর হয়েছিল, OOM killer কোন প্রসেস মেরেছে |
| **ইতিহাস** | কোন প্যাকেজ কবে আপডেট হয়েছিল |

> 📝 **মনে রাখো:** লিনাক্সে সমস্যা হলে প্রথম প্রশ্ন সবসময় একটাই — **"লগ কী বলছে?"** অভিজ্ঞ অ্যাডমিনরা অনুমান করে না, লগ পড়ে।

---

## ২৮.২ লিনাক্স লগিং-এর আর্কিটেকচার — ভেতরে কী ঘটে

আধুনিক লিনাক্সে (Ubuntu 24.04, Debian 12, Fedora, RHEL 9) লগ লেখার দুটো প্রধান ব্যবস্থা পাশাপাশি চলে:

1. **systemd-journald** — systemd-এর অংশ, সব লগ একটা বাইনারি ডাটাবেসে (journal) রাখে। এর সাথে কথা বলার টুল `journalctl`।
2. **rsyslog** — পুরোনো ঐতিহ্যবাহী syslog ব্যবস্থার আধুনিক রূপ। লগকে সাধারণ টেক্সট ফাইলে (`/var/log/syslog` ইত্যাদি) লেখে।

আর এদের সবার নিচে আছে **kernel ring buffer** — kernel নিজের বার্তা এখানে রাখে।

```
                ┌──────────────────────────────────────────────┐
                │                 KERNEL                       │
                │   printk() ──► ring buffer (/dev/kmsg)       │──► dmesg পড়ে
                └───────────────────────┬──────────────────────┘
                                        │ kernel বার্তা
   ┌──────────────┐  stdout/stderr      ▼
   │ systemd       │──────────────►┌──────────────────────┐
   │ services      │               │  systemd-journald    │
   └──────────────┘               │  (সব লগ জমা হয়)     │
   ┌──────────────┐  syslog()/     │                      │
   │ অ্যাপ্লিকেশন  │──/dev/log──►  │  /run/log/journal    │ (volatile, RAM)
   │ ও logger     │   socket      │  /var/log/journal    │ (persistent, ডিস্ক)
   └──────────────┘               └──────────┬───────────┘
                                             │ ফরোয়ার্ড
                                             ▼
                                  ┌──────────────────────┐
                                  │      rsyslogd        │
                                  │ (/etc/rsyslog.conf)  │
                                  └───┬──────┬──────┬────┘
                                      ▼      ▼      ▼
                          /var/log/syslog  auth.log  রিমোট সার্ভার
                          (বা messages)   (secure)  (@host:514)

   ┌──────────────┐  নিজের ফাইলে সরাসরি লেখে
   │ nginx, MySQL │──────────────────────────► /var/log/nginx/access.log
   └──────────────┘
```

কয়েকটি গুরুত্বপূর্ণ কথা এই ছবি থেকে:

- **`/dev/log`** একটি Unix socket (একই মেশিনের প্রসেসগুলোর মধ্যে কথা বলার "পাইপ")। কোনো প্রোগ্রাম C লাইব্রেরির `syslog()` ফাংশন ডাকলে বার্তা এই socket-এ যায়। systemd-ওয়ালা সিস্টেমে এই socket-টি আসলে journald ধরে রাখে।
- systemd যে সার্ভিস চালায়, তার `stdout` আর `stderr`-এ যা প্রিন্ট হয়, সেটাও স্বয়ংক্রিয়ভাবে journal-এ চলে যায়। তাই systemd সার্ভিসের জন্য আলাদা লগ ফাইল না থাকলেও `journalctl -u` দিয়ে লগ পাওয়া যায়।
- rsyslog journal থেকে বার্তা নেয় (Debian/Ubuntu-তে সাধারণত journald-এর syslog socket দিয়ে, RHEL/Fedora-তে `imjournal` মডিউল দিয়ে সরাসরি journal পড়ে) এবং নিয়ম অনুযায়ী টেক্সট ফাইলে ভাগ করে লেখে।
- কিছু প্রোগ্রাম (nginx, Apache, MySQL, PostgreSQL) syslog-কে পাশ কাটিয়ে **নিজেরাই** নিজেদের ফাইলে লেখে।

### distro-ভেদে পার্থক্য

| বিষয় | Ubuntu 24.04 | Debian 12 | Fedora (সাম্প্রতিক) | RHEL 9 / Rocky / Alma |
|---|---|---|---|---|
| rsyslog ডিফল্ট ইনস্টল | হ্যাঁ | **না** (শুধু journal) | **না** | হ্যাঁ |
| প্রধান টেক্সট লগ | `/var/log/syslog` | (rsyslog ইনস্টল করলে) `/var/log/syslog` | নেই, `journalctl` | `/var/log/messages` |
| অথেন্টিকেশন লগ | `/var/log/auth.log` | (rsyslog থাকলে) `auth.log` | journal | `/var/log/secure` |
| persistent journal | হ্যাঁ (`/var/log/journal` থাকে) | হ্যাঁ | হ্যাঁ | distro সংস্করণ/ইনস্টল ভেদে — নিজে চেক করো |

> 💡 **টিপস:** Debian 12 বা Fedora-তে `/var/log/syslog` খুঁজে না পেলে ভয় পেয়ো না। সব লগ journal-এ আছে, `journalctl` দিয়ে দেখো। টেক্সট ফাইল চাইলে `sudo apt install rsyslog` (Debian) বা `sudo dnf install rsyslog` (Fedora)।

বুট প্রসেসের সময় journald কখন চালু হয়, সেটা [অধ্যায় ২৭](27-boot-process.md)-এ আর systemd-এর সার্বিক ধারণা [অধ্যায় ২৬](26-systemd-services.md)-এ পাবে।

---

## ২৮.৩ `/var/log` — লগের বাড়ি

[অধ্যায় ৭](07-filesystem-hierarchy.md)-এ দেখেছ যে `/var` মানে "variable" — যে ডেটা সময়ের সাথে বদলায়। লগ ঠিক তাই, তাই লগের বাড়ি `/var/log`।

```bash
$ ls /var/log
alternatives.log  apt        btmp            dpkg.log   kern.log   nginx     ubuntu-advantage.log
auth.log          auth.log.1 cloud-init.log  faillog    journal    syslog    unattended-upgrades
boot.log          dist-upgrade  installer    lastlog    private    syslog.1  wtmp
```

### গুরুত্বপূর্ণ লগ ফাইলের তালিকা

| ফাইল (Debian/Ubuntu) | ফাইল (RHEL/Fedora) | কী থাকে |
|---|---|---|
| `/var/log/syslog` | `/var/log/messages` | সাধারণ সিস্টেম বার্তা — প্রায় সবকিছু |
| `/var/log/auth.log` | `/var/log/secure` | লগইন, SSH, `sudo`, `su`, PAM অথেন্টিকেশন |
| `/var/log/kern.log` | (`messages`-এ থাকে) | kernel বার্তা |
| `/var/log/dpkg.log` | `/var/log/dnf.log`, `dnf.rpm.log` | প্যাকেজ ইনস্টল/রিমুভ |
| `/var/log/apt/history.log` | `dnf history` কমান্ড | কোন apt কমান্ড কবে চালানো হয়েছিল |
| `/var/log/boot.log` | `/var/log/boot.log` | বুটের সময় সার্ভিস স্টার্টআপ বার্তা (সব সিস্টেমে থাকে না) |
| `/var/log/mail.log` | `/var/log/maillog` | মেইল সার্ভার (Postfix ইত্যাদি) |
| (`syslog`-এ থাকে) | `/var/log/cron` | cron জবের রেকর্ড |
| `/var/log/ufw.log` | — | ufw ফায়ারওয়ালের ব্লক করা প্যাকেট |
| — | `/var/log/audit/audit.log` | auditd ও SELinux denial (RHEL-এ খুব গুরুত্বপূর্ণ) |
| `/var/log/nginx/access.log`, `error.log` | একই | nginx ওয়েব সার্ভার |
| `/var/log/apache2/` | `/var/log/httpd/` | Apache ওয়েব সার্ভার |
| `/var/log/mysql/` | `/var/log/mysqld.log` বা `mariadb/` | ডাটাবেস |
| `/var/log/wtmp` | একই | সফল লগইন/লগআউট ও রিবুটের ইতিহাস (বাইনারি, `last` দিয়ে পড়ো) |
| `/var/log/btmp` | একই | ব্যর্থ লগইন চেষ্টা (বাইনারি, `lastb` দিয়ে পড়ো) |
| `/var/log/lastlog` | একই | প্রতি ইউজারের শেষ লগইন (বাইনারি, `lastlog` দিয়ে পড়ো) |
| `/var/log/journal/` | একই | systemd journal (বাইনারি, `journalctl` দিয়ে পড়ো) |

> ⚠️ **সতর্কতা:** `wtmp`, `btmp`, `lastlog` আর `journal/`-এর ফাইলগুলো **বাইনারি**। এগুলো `cat` বা `less` দিয়ে খুললে অর্থহীন অক্ষর দেখবে, টার্মিনাল এলোমেলোও হয়ে যেতে পারে (তখন `reset` লিখে Enter চাপো)। এগুলো সবসময় নির্দিষ্ট কমান্ড দিয়ে পড়বে।

> 📝 **মনে রাখো:** কিছু নতুন distro সংস্করণে Y2038 সমস্যার কারণে `wtmp`/`lastlog`-এর বদলে `wtmpdb` ও `lastlog2` আসছে। কমান্ড না পেলে তোমার distro-র ডকুমেন্টেশন দেখো।

### লগ পড়ার অনুমতি

বেশিরভাগ লগ সাধারণ ইউজার পড়তে পারে না, কারণ এতে গোপন তথ্য থাকে।

```bash
$ ls -l /var/log/auth.log
-rw-r----- 1 syslog adm 48213 Nov 12 10:15 /var/log/auth.log
$ cat /var/log/auth.log
cat: /var/log/auth.log: Permission denied
```

Ubuntu/Debian-এ **`adm`** গ্রুপের সদস্যরা বেশিরভাগ লগ পড়তে পারে। journal পড়ার জন্য **`systemd-journal`** গ্রুপ (Fedora/RHEL-এ `wheel` গ্রুপও সাধারণত পড়তে পারে)। উপায় দুটো: `sudo` ব্যবহার করো, অথবা নিজেকে গ্রুপে যোগ করো ([অধ্যায় ১৫](15-users-groups.md)):

```bash
$ sudo usermod -aG adm,systemd-journal $USER
# তারপর লগআউট করে আবার লগইন করো
```

### একটি syslog লাইনের গঠন

```
Nov 12 10:15:32 webserver sshd[2841]: Accepted publickey for rahim from 192.168.1.50 port 52144 ssh2
└──────┬──────┘ └───┬───┘ └─┬─┘└─┬─┘  └──────────────────────────┬──────────────────────────────┘
    সময়         হোস্টনেম  প্রোগ্রাম PID                         বার্তা (message)
```

> 💡 **টিপস:** Ubuntu 24.04-এর rsyslog ডিফল্টভাবে উচ্চ-নির্ভুল ISO ফরম্যাটে সময় লেখে, যেমন `2026-11-12T10:15:32.123456+06:00`। পুরোনো সিস্টেমে উপরের মতো `Nov 12 10:15:32` দেখবে। দুটোরই অর্থ এক।

### টেক্সট লগ পড়ার মৌলিক কমান্ড

এসব কমান্ড বিস্তারিত [অধ্যায় ১০](10-file-dekha.md) আর [অধ্যায় ২০](20-grep-regex.md)-এ আছে; এখানে লগের প্রেক্ষিতে দেখো।

```bash
# শেষ ২০ লাইন
$ sudo tail -n 20 /var/log/syslog

# লাইভ দেখা (নতুন লাইন আসলেই দেখাবে) — থামাতে Ctrl+C
$ sudo tail -f /var/log/auth.log

# -F: ফাইল rotate হলেও (নাম বদলে নতুন ফাইল এলেও) ধরে রাখবে — লগের জন্য -f-এর চেয়ে ভালো
$ sudo tail -F /var/log/nginx/access.log

# less দিয়ে পড়া, শেষ থেকে শুরু
$ sudo less +G /var/log/syslog

# less-এ "follow" মোড (tail -f-এর মতো), Ctrl+C চাপলে সাধারণ less-এ ফেরে
$ sudo less +F /var/log/syslog

# খোঁজা
$ sudo grep -i "error" /var/log/syslog

# পুরোনো, কম্প্রেস করা লগে খোঁজা
$ sudo zgrep "Failed password" /var/log/auth.log.2.gz
$ sudo zcat /var/log/syslog.3.gz | less
```

একাধিক লগ একসাথে লাইভ দেখতে:

```bash
$ sudo tail -F /var/log/syslog /var/log/auth.log
==> /var/log/syslog <==
2026-11-12T10:20:01.512+06:00 web CRON[3011]: (root) CMD (command -v debian-sa1 > /dev/null && debian-sa1 1 1)

==> /var/log/auth.log <==
2026-11-12T10:20:05.871+06:00 web sudo:    rahim : TTY=pts/0 ; PWD=/home/rahim ; USER=root ; COMMAND=/usr/bin/apt update
```

---

## ২৮.৪ syslog প্রোটোকল: facility ও severity

syslog একটি পুরোনো (১৯৮০-র দশকের) কিন্তু এখনো সর্বব্যাপী মানদণ্ড, যা পরে RFC 5424-এ লিখিত হয়েছে। প্রতিটি syslog বার্তার সাথে দুটো ট্যাগ থাকে:

- **facility** — বার্তাটা **কোন বিভাগ** থেকে এসেছে (kernel? মেইল? অথেন্টিকেশন?)
- **severity** (বা priority/level) — বার্তাটা **কতটা গুরুতর**

উপমা: হাসপাতালের ইমার্জেন্সি। facility হলো রোগী কোন ওয়ার্ড থেকে এসেছে (হৃদরোগ, হাড়, শিশু), আর severity হলো রোগীর অবস্থা কতটা জরুরি (লাল, হলুদ, সবুজ)। rsyslog হলো রিসেপশনিস্ট, যে এই দুই ট্যাগ দেখে ঠিক করে রোগীকে কোথায় পাঠাবে।

### Facility তালিকা

| কোড | facility | কী জন্য |
|---|---|---|
| 0 | `kern` | kernel বার্তা |
| 1 | `user` | সাধারণ ইউজার-লেভেল প্রোগ্রাম (ডিফল্ট) |
| 2 | `mail` | মেইল সিস্টেম |
| 3 | `daemon` | সিস্টেম ডেমন (ব্যাকগ্রাউন্ড সার্ভিস) |
| 4 | `auth` | অথেন্টিকেশন/নিরাপত্তা |
| 5 | `syslog` | syslogd নিজের বার্তা |
| 6 | `lpr` | প্রিন্টার |
| 7 | `news` | Usenet news |
| 8 | `uucp` | UUCP (পুরোনো) |
| 9 | `cron` | cron ও at |
| 10 | `authpriv` | ব্যক্তিগত/গোপন অথেন্টিকেশন বার্তা |
| 11 | `ftp` | FTP সার্ভার |
| 16–23 | `local0` – `local7` | তোমার নিজের ব্যবহারের জন্য খালি রাখা |

> 💡 **টিপস:** নিজের অ্যাপ বা স্ক্রিপ্টের লগ আলাদা ফাইলে পাঠাতে চাইলে `local0`–`local7` ব্যবহার করো। এগুলো ঠিক এই কাজের জন্যই আছে।

### Severity তালিকা (সংখ্যা যত কম, তত গুরুতর)

| কোড | নাম | অর্থ | উদাহরণ |
|---|---|---|---|
| 0 | `emerg` | সিস্টেম অচল | kernel panic |
| 1 | `alert` | এখনই ব্যবস্থা নিতে হবে | ডাটাবেস করাপ্ট |
| 2 | `crit` | গুরুতর অবস্থা | হার্ডওয়্যার ফেইলিওর |
| 3 | `err` | error | সার্ভিস চালু হতে পারেনি |
| 4 | `warning` | সতর্কতা | ডিস্ক ৯০% ভরা |
| 5 | `notice` | স্বাভাবিক কিন্তু উল্লেখযোগ্য | কনফিগ রিলোড হয়েছে |
| 6 | `info` | তথ্যমূলক | ইউজার লগইন করেছে |
| 7 | `debug` | ডিবাগ | ডেভেলপারদের খুঁটিনাটি |

মনে রাখার ছড়া (ইংরেজিতে প্রথম অক্ষর): **"Every Alley Cat Eats Whiskas Not In Daytime"** → emerg, alert, crit, err, warning, notice, info, debug।

> 📝 **মনে রাখো:** "priority" শব্দটা দুইভাবে ব্যবহার হয়। rsyslog কনফিগে `facility.severity` জোড়াকে "selector" বা "priority" বলে; আর journalctl-এর `-p` অপশনে "priority" মানে শুধু severity।

---

## ২৮.৫ rsyslog — কনফিগারেশন ও রাউটিং

### কনফিগ ফাইল কোথায়

```
/etc/rsyslog.conf          ← মূল ফাইল (মডিউল লোড, গ্লোবাল সেটিং)
/etc/rsyslog.d/*.conf      ← অতিরিক্ত নিয়ম; এখানে নিজের ফাইল রাখো
```

Ubuntu-তে আসল নিয়মগুলো থাকে `/etc/rsyslog.d/50-default.conf`-এ:

```bash
$ grep -v '^#' /etc/rsyslog.d/50-default.conf | grep -v '^$'
auth,authpriv.*                 /var/log/auth.log
*.*;auth,authpriv.none          -/var/log/syslog
kern.*                          -/var/log/kern.log
mail.*                          -/var/log/mail.log
mail.err                        /var/log/mail.err
*.emerg                         :omusrmsg:*
```

RHEL 9-এ `/etc/rsyslog.conf`-এর ভেতরেই:

```
*.info;mail.none;authpriv.none;cron.none    /var/log/messages
authpriv.*                                  /var/log/secure
mail.*                                      -/var/log/maillog
cron.*                                      /var/log/cron
*.emerg                                     :omusrmsg:*
local7.*                                    /var/log/boot.log
```

### Selector-এর ব্যাকরণ

প্রতিটি নিয়মের দুটো অংশ: **বাঁয়ে selector (কোন বার্তা), ডানে action (কোথায় পাঠাবে)।**

| লেখা | অর্থ |
|---|---|
| `kern.*` | kern facility-র সব severity |
| `mail.err` | mail facility-র `err` **এবং তার চেয়ে গুরুতর** (err, crit, alert, emerg) |
| `mail.=err` | **শুধু** `err` লেভেল |
| `mail.!err` | `err`-এর চেয়ে কম গুরুতর (warning থেকে debug) |
| `auth,authpriv.*` | একাধিক facility কমা দিয়ে |
| `*.*;auth,authpriv.none` | সবকিছু, **কিন্তু** auth/authpriv বাদে (`;` দিয়ে একাধিক selector, `none` মানে বাদ) |
| `*.emerg` | যেকোনো facility-র emergency |

| Action | অর্থ |
|---|---|
| `/var/log/file` | ফাইলে লেখো |
| `-/var/log/file` | ফাইলে লেখো, প্রতিবার sync করার দরকার নেই (পুরোনো সিনট্যাক্স; আধুনিক rsyslog-এ ডিফল্টই এমন) |
| `:omusrmsg:*` | লগইন করা সব ইউজারের টার্মিনালে দেখাও |
| `@192.168.1.10:514` | রিমোট সার্ভারে **UDP** দিয়ে পাঠাও |
| `@@192.168.1.10:514` | রিমোট সার্ভারে **TCP** দিয়ে পাঠাও |
| `stop` | এই বার্তা নিয়ে আর কিছু করো না (পরের নিয়মে যাবে না) |

### নিজের অ্যাপের জন্য আলাদা লগ ফাইল

```bash
$ sudo nano /etc/rsyslog.d/30-myapp.conf
```

```
# local3 facility-র সব বার্তা আলাদা ফাইলে, তারপর থামো
local3.*    /var/log/myapp.log
& stop
```

`& stop` মানে "আগের লাইনের মতো একই বার্তার জন্য — থামো", যাতে এগুলো `syslog`-এ দ্বিতীয়বার না যায়। (মনে রেখো, ফাইলের নাম `30-` দিয়ে শুরু করেছি যাতে `50-default.conf`-এর আগে পড়া হয়।)

কনফিগ ঠিক আছে কি না যাচাই করে রিস্টার্ট:

```bash
$ sudo rsyslogd -N1
rsyslogd: version 8.2312.0, config validation run (level 1), master config /etc/rsyslog.conf
rsyslogd: End of config validation run. Bye.
$ sudo systemctl restart rsyslog
```

> ⚠️ **সতর্কতা:** Ubuntu-তে rsyslog `syslog` ইউজার হিসেবে চলে। নতুন লগ ফাইল `/var/log`-এ তৈরি করতে না পারলে কিছু লেখা হবে না। সমস্যা হলে আগে থেকে ফাইল বানিয়ে মালিকানা দাও: `sudo touch /var/log/myapp.log && sudo chown syslog:adm /var/log/myapp.log`।

### আধুনিক RainerScript সিনট্যাক্স (সংক্ষেপে)

পুরোনো selector সিনট্যাক্সের পাশাপাশি rsyslog-এর নিজস্ব স্ক্রিপ্ট ভাষা আছে, যাতে প্রোগ্রামের নাম বা লেখার ভেতরের শব্দ দেখে ফিল্টার করা যায়:

```
if $programname == 'nginx' and $msg contains 'error' then {
    action(type="omfile" file="/var/log/nginx-errors-syslog.log")
    stop
}
```

### সেন্ট্রালাইজড লগিং (একাধিক সার্ভারের লগ এক জায়গায়)

দশটা সার্ভার থাকলে দশ জায়গায় লগ খোঁজা কষ্টের, আর হ্যাকার সার্ভারে ঢুকে প্রথমে লগ মুছে দেয়। তাই লগ আরেকটা সার্ভারে পাঠানো ভালো অভ্যাস।

**লগ গ্রহণকারী সার্ভারে** (`/etc/rsyslog.d/10-receive.conf`):

```
module(load="imtcp")
input(type="imtcp" port="514")
```

**লগ পাঠানো ক্লায়েন্টে** (`/etc/rsyslog.d/90-forward.conf`):

```
*.*  @@logserver.example.com:514
```

তারপর দুই জায়গায় `sudo systemctl restart rsyslog`, আর সার্ভারে ফায়ারওয়ালে 514/tcp খুলতে হবে ([অধ্যায় ৩৯](39-firewall.md))।

> 💡 **টিপস:** বড় প্রতিষ্ঠানে Graylog, Elasticsearch/OpenSearch + Kibana (ELK), বা Grafana Loki-এর মতো সিস্টেমে লগ জমা হয়, যেখানে ওয়েব ইন্টারফেসে খোঁজা ও গ্রাফ করা যায়। এগুলোর ভিত্তিও এই একই syslog/journal ধারণা।

---

## ২৮.৬ systemd journal ও `journalctl`

### journal কেন আলাদা

পুরোনো টেক্সট লগের কিছু সীমাবদ্ধতা ছিল: প্রতিটা প্রোগ্রাম আলাদা ফরম্যাটে লেখে, কোন লাইন কোন সার্ভিসের সেটা নিশ্চিতভাবে বোঝা কঠিন, আর লেখাটা পরে চুপচাপ বদলে দেওয়া যায়। journald এগুলোর সমাধান দেয়:

- **স্ট্রাকচার্ড ডেটা**: প্রতিটা এন্ট্রির সাথে অনেক ফিল্ড থাকে — `_PID`, `_UID`, `_COMM` (কমান্ডের নাম), `_SYSTEMD_UNIT`, `_BOOT_ID`, `PRIORITY` ইত্যাদি। এই ফিল্ডগুলো (যেগুলো `_` দিয়ে শুরু) journald নিজে যোগ করে, তাই প্রোগ্রাম এগুলো জাল করতে পারে না।
- **ইনডেক্স করা**: লক্ষ লক্ষ লাইনের মধ্যেও নির্দিষ্ট সার্ভিস বা বুটের লগ দ্রুত বের করা যায়।
- **বুট অনুযায়ী ভাগ**: "আগের বুটে কী হয়েছিল" সহজে দেখা যায়।
- **শুরু থেকে সব**: initramfs আর kernel-এর প্রথম বার্তা থেকেই ধরে।
- **ঐচ্ছিক seal**: `--setup-keys` দিয়ে Forward Secure Sealing চালু করলে লগ বদলানো হয়েছে কি না যাচাই করা যায়।

উপমা: টেক্সট লগ যেন হাতে লেখা ডায়েরি — পড়তে সহজ কিন্তু খুঁজতে কষ্ট। journal যেন লাইব্রেরির কম্পিউটার ক্যাটালগ — বই (এন্ট্রি) লেখক, বিষয়, তারিখ দিয়ে মুহূর্তে খোঁজা যায়।

### প্রাথমিক ব্যবহার

```bash
$ journalctl
```

কোনো অপশন ছাড়া পুরোনো থেকে নতুন — **সব** লগ দেখায়, `less` pager-এর ভেতরে। sudo ছাড়া চালালে (এবং তুমি `adm`/`systemd-journal` গ্রুপে না থাকলে) শুধু নিজের ইউজারের লগ দেখবে।

```
Nov 12 09:01:12 laptop kernel: Linux version 6.8.0-48-generic (buildd@lcy02-amd64-040) ...
Nov 12 09:01:12 laptop kernel: Command line: BOOT_IMAGE=/boot/vmlinuz-6.8.0-48-generic root=UUID=...
...
```

রঙ: `err` ও তার চেয়ে গুরুতর লাল ও মোটা, `warning` হলুদ/মোটা, `notice` মোটা — তাই চোখ বুলিয়েই সমস্যা ধরা যায়।

### journalctl-এর গুরুত্বপূর্ণ অপশন

| অপশন | কাজ |
|---|---|
| `-u UNIT` | নির্দিষ্ট systemd unit (সার্ভিস)-এর লগ; একাধিকবার দেওয়া যায় |
| `-f` | follow — লাইভ নতুন লগ (`tail -f`-এর মতো); থামাতে Ctrl+C |
| `-n N` | শেষ N লাইন (শুধু `-n` দিলে ১০) |
| `-e` | pager খুলে সরাসরি শেষে যাও |
| `-r` | উল্টো ক্রম — নতুনটা আগে |
| `-b` | বর্তমান বুট থেকে; `-b -1` আগের বুট, `-b -2` তার আগের |
| `--list-boots` | সব রেকর্ড করা বুটের তালিকা |
| `-k` | শুধু kernel বার্তা (`-b`-সহ বোঝায়), অনেকটা `dmesg`-এর মতো |
| `-p LEVEL` | এই severity ও তার চেয়ে গুরুতর; `-p warning..err` দিয়ে রেঞ্জ |
| `--since`, `-S` | এই সময় থেকে |
| `--until`, `-U` | এই সময় পর্যন্ত |
| `-t ID` | syslog identifier (ট্যাগ) দিয়ে ফিল্টার |
| `-g PATTERN` | বার্তার ভেতরে regex দিয়ে খোঁজা (`--grep`) |
| `-o FORMAT` | আউটপুট ফরম্যাট (নিচে টেবিল) |
| `-x` | যেখানে পাওয়া যায়, ব্যাখ্যামূলক অতিরিক্ত লেখা যোগ করো |
| `--no-pager` | pager ছাড়া সরাসরি প্রিন্ট (স্ক্রিপ্ট ও পাইপের জন্য) |
| `--user` | নিজের ইউজার সার্ভিসের লগ |
| `--user-unit UNIT` | নির্দিষ্ট ইউজার-unit-এর লগ |
| `-F FIELD` | একটা ফিল্ডে কী কী মান আছে তার তালিকা |
| `-N` | journal-এ থাকা সব ফিল্ডের নাম |
| `--disk-usage` | journal ডিস্কে কত জায়গা নিচ্ছে |
| `--vacuum-size=`, `--vacuum-time=`, `--vacuum-files=` | পুরোনো journal ফাইল মুছে জায়গা খালি করো |
| `--verify` | journal ফাইলের সঠিকতা যাচাই |
| `--flush` | RAM-এর (`/run`) journal ডিস্কে (`/var`) নিয়ে যাও |
| `--rotate` | এখনই নতুন journal ফাইল শুরু করো |

### `-u` — একটা সার্ভিসের লগ

```bash
$ journalctl -u ssh
Nov 12 09:01:20 server systemd[1]: Starting ssh.service - OpenBSD Secure Shell server...
Nov 12 09:01:20 server sshd[812]: Server listening on 0.0.0.0 port 22.
Nov 12 09:01:20 server systemd[1]: Started ssh.service - OpenBSD Secure Shell server.
Nov 12 10:15:32 server sshd[2841]: Accepted publickey for rahim from 192.168.1.50 port 52144 ssh2
```

> 📝 **মনে রাখো:** Ubuntu/Debian-এ SSH সার্ভিসের নাম `ssh`, RHEL/Fedora-তে `sshd`। Apache-এর নাম Debian পরিবারে `apache2`, RHEL পরিবারে `httpd`।

একাধিক সার্ভিস একসাথে:

```bash
$ journalctl -u nginx -u php8.3-fpm --since today
```

সবচেয়ে বেশি ব্যবহৃত কম্বো — কোনো সার্ভিস ফেইল করলে:

```bash
$ systemctl status nginx          # সংক্ষিপ্ত অবস্থা + শেষ কয়েক লাইন লগ
$ journalctl -u nginx -n 50 --no-pager
$ journalctl -xeu nginx           # শেষে যাও, ব্যাখ্যাসহ
```

### `-f` — লাইভ লগ

```bash
$ journalctl -f                   # সব লাইভ
$ journalctl -fu nginx            # শুধু nginx লাইভ
$ sudo journalctl -f -p err       # শুধু error লাইভ
```

একটা টার্মিনালে `journalctl -f` চালিয়ে আরেকটা টার্মিনালে সার্ভিস রিস্টার্ট করে দেখো — কী হচ্ছে চোখের সামনে দেখবে। ট্রাবলশুটিং-এ এটা সোনার মতো দামি কৌশল।

### `-b` — বুট অনুযায়ী

```bash
$ journalctl --list-boots
IDX BOOT ID                          FIRST ENTRY                 LAST ENTRY
 -2 3f1c2a9e8b5d4c7a9e1f0b2c3d4e5f6a Sun 2026-11-10 08:12:01 +06 Sun 2026-11-10 23:40:12 +06
 -1 7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d Mon 2026-11-11 09:00:44 +06 Mon 2026-11-11 22:15:03 +06
  0 c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9 Wed 2026-11-12 09:01:12 +06 Wed 2026-11-12 10:45:30 +06

$ journalctl -b          # এই বুট
$ journalctl -b -1       # আগের বুট
$ journalctl -b -1 -e    # আগের বুটের একদম শেষ অংশ — "কম্পিউটার হঠাৎ বন্ধ হলো কেন?"
```

> ⚠️ **সতর্কতা:** journal persistent না হলে (`/run/log/journal`-এ থাকলে) প্রতিটা রিবুটে আগের লগ হারিয়ে যায়, আর `--list-boots`-এ শুধু একটা বুট দেখাবে। নিচে ২৮.৭-এ persistent করার পদ্ধতি দেখো।

### `-p` — severity দিয়ে ফিল্টার

```bash
$ journalctl -p err -b                 # এই বুটের সব error ও তার চেয়ে গুরুতর
$ journalctl -p 3 -b                   # একই কথা, সংখ্যা দিয়ে
$ journalctl -p warning..err           # শুধু warning থেকে err পর্যন্ত
```

নতুন সিস্টেম হাতে পেলে প্রথম স্বাস্থ্য-পরীক্ষা:

```bash
$ sudo journalctl -b -p err --no-pager
Nov 12 09:01:14 laptop kernel: ACPI BIOS Error (bug): Could not resolve symbol [\_SB.PCI0.GPP0], AE_NOT_FOUND
Nov 12 09:01:25 laptop bluetoothd[901]: Failed to set mode: Failed (0x03)
```

> 💡 **টিপস:** প্রতিটা error মানেই সমস্যা নয়। ACPI BIOS error-এর মতো অনেক বার্তা বেশিরভাগ ল্যাপটপে দেখা যায় এবং ক্ষতিহীন। দেখতে হবে — যে সমস্যা নিয়ে তুমি চিন্তিত, তার **সময়ের সাথে** কোনো error মেলে কি না।

### `--since` ও `--until` — সময়ের জানালা

```bash
$ journalctl --since "2026-11-12 09:00:00" --until "2026-11-12 09:30:00"
$ journalctl --since "2026-11-12"            # ওই দিনের মধ্যরাত থেকে
$ journalctl --since today
$ journalctl --since yesterday --until today
$ journalctl --since "1 hour ago"
$ journalctl --since "-15min"
$ journalctl -u nginx --since "10:00" --until "10:30"   # আজকের ১০টা থেকে সাড়ে ১০টা
```

গ্রহণযোগ্য সময়ের রূপ: `"YYYY-MM-DD HH:MM:SS"`, `"YYYY-MM-DD"`, `"HH:MM"`, `today`, `yesterday`, `tomorrow`, `now`, `"N hour ago"`/`"N min ago"`, অথবা `-1h`, `+30min`-এর মতো আপেক্ষিক রূপ। (বিস্তারিত: `man systemd.time`)

### `-k` — kernel বার্তা

```bash
$ journalctl -k              # এই বুটের kernel বার্তা
$ journalctl -k -b -1        # আগের বুটের kernel বার্তা — dmesg যেটা পারে না!
$ journalctl -k -p warning
```

`dmesg` শুধু বর্তমান ring buffer দেখায়, কিন্তু persistent journal থাকলে `journalctl -k -b -1` দিয়ে আগের বুটের kernel বার্তাও পাবে।

### ফিল্ড দিয়ে ম্যাচ করা

journalctl-এ `FIELD=value` লিখলেই ফিল্টার হয়:

```bash
$ journalctl _PID=2841
$ journalctl _UID=1000 --since today          # একজন ইউজারের সব প্রসেসের লগ
$ journalctl _COMM=sshd                        # কমান্ডের নাম দিয়ে
$ journalctl /usr/sbin/sshd                    # executable-এর পথ দিয়ে
$ journalctl _SYSTEMD_UNIT=cron.service
$ journalctl -t sudo                           # syslog ট্যাগ দিয়ে
```

নিয়ম: **ভিন্ন ফিল্ড** দিলে AND (দুটোই মিলতে হবে); **একই ফিল্ড** দুবার দিলে OR; মাঝে `+` দিলে দুই দিকের শর্তের OR।

```bash
$ journalctl _SYSTEMD_UNIT=nginx.service _PID=1234     # AND
$ journalctl _COMM=sshd _COMM=sudo                     # OR
$ journalctl _COMM=sshd + _SYSTEMD_UNIT=cron.service   # OR (দুই গ্রুপ)
```

কোন ফিল্ডে কী মান আছে জানতে:

```bash
$ journalctl -F _SYSTEMD_UNIT | head
cron.service
systemd-journald.service
ssh.service
nginx.service
...
```

### `-g` — বার্তার ভেতরে খোঁজা

```bash
$ journalctl -u ssh -g "Failed password" --since today
$ journalctl -g "(?i)out of memory" -b
```

ডিফল্টভাবে প্যাটার্নে সব ছোট হাতের অক্ষর থাকলে খোঁজা case-insensitive হয়। `-g` কাজ না করলে (কিছু বিল্ডে PCRE2 ছাড়া কম্পাইল করা থাকে) পাইপ দিয়ে grep করো: `journalctl -u ssh --no-pager | grep "Failed password"`।

### `-o` — আউটপুট ফরম্যাট

| ফরম্যাট | কেমন দেখায় / কী কাজে |
|---|---|
| `short` | ডিফল্ট, syslog-এর মতো |
| `short-iso` | ISO 8601 সময় (`2026-11-12T10:15:32+06:00`) |
| `short-precise` | মাইক্রোসেকেন্ড পর্যন্ত সময় |
| `short-monotonic` | বুটের পর থেকে কত সেকেন্ড |
| `verbose` | এন্ট্রির **সব** ফিল্ড |
| `cat` | শুধু বার্তা, সময়/হোস্ট ছাড়া |
| `json` | প্রতি এন্ট্রি এক লাইনে JSON (মেশিনে প্রসেসের জন্য) |
| `json-pretty` | সাজানো, পড়ার উপযোগী JSON |
| `export` | বাইনারি-নিরাপদ এক্সপোর্ট ফরম্যাট (ব্যাকআপ/স্থানান্তর) |

```bash
$ journalctl -u ssh -n 1 -o json-pretty
{
        "__REALTIME_TIMESTAMP" : "1762920932512345",
        "_HOSTNAME" : "server",
        "PRIORITY" : "6",
        "SYSLOG_IDENTIFIER" : "sshd",
        "_PID" : "2841",
        "_UID" : "0",
        "_COMM" : "sshd",
        "_EXE" : "/usr/sbin/sshd",
        "_SYSTEMD_UNIT" : "ssh.service",
        "_BOOT_ID" : "c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9",
        "MESSAGE" : "Accepted publickey for rahim from 192.168.1.50 port 52144 ssh2",
        ...
}
```

(আসল আউটপুটে আরও অনেক ফিল্ড থাকে।) JSON আউটপুট `jq` দিয়ে প্রসেস করা যায়:

```bash
$ journalctl -u ssh --since today -o json --no-pager | jq -r '.MESSAGE' | grep Accepted
```

নির্দিষ্ট ফিল্ডগুলোই দেখতে: `journalctl -o json-pretty --output-fields=MESSAGE,_PID -n 3`।

### journalctl pager-এর কীবোর্ড শর্টকাট

journalctl ডিফল্টভাবে `less` ব্যবহার করে, তাই less-এর সব শর্টকাট কাজ করে:

| কী | কাজ |
|---|---|
| `Space` / `f` | এক পাতা নিচে |
| `b` | এক পাতা উপরে |
| `↓` / `j`, `↑` / `k` | এক লাইন নিচে/উপরে |
| `G` | একদম শেষে (সবচেয়ে নতুন) |
| `g` | একদম শুরুতে |
| `/শব্দ` | নিচের দিকে খোঁজো |
| `?শব্দ` | উপরের দিকে খোঁজো |
| `n` / `N` | পরের / আগের মিল |
| `→` / `←` | লম্বা লাইন ডানে/বাঁয়ে স্ক্রল (journalctl লম্বা লাইন কাটে না, পাশে সরায়) |
| `F` | follow মোড (লাইভ); Ctrl+C দিয়ে ফেরো |
| `-S` তারপর Enter | লম্বা লাইন কাটা/মোড়ানো অদলবদল |
| `h` | less-এর সাহায্য |
| `q` | বের হও |
| `Ctrl+C` | `journalctl -f` বা `tail -f` থামাও |

---

## ২৮.৭ journal-এর স্টোরেজ: volatile বনাম persistent

### দুই জায়গা

```
/run/log/journal/<machine-id>/   ← volatile: RAM-এ (tmpfs), রিবুটে মুছে যায়
/var/log/journal/<machine-id>/   ← persistent: ডিস্কে, রিবুটের পরও থাকে
```

কোনটা ব্যবহার হবে সেটা ঠিক করে `/etc/systemd/journald.conf`-এর `Storage=` সেটিং:

| মান | আচরণ |
|---|---|
| `volatile` | শুধু RAM-এ |
| `persistent` | ডিস্কে; দরকার হলে `/var/log/journal` নিজেই তৈরি করে |
| `auto` (ডিফল্ট) | `/var/log/journal` ডিরেক্টরি **থাকলে** ডিস্কে, না থাকলে RAM-এ |
| `none` | কিছুই রাখে না (শুধু ফরোয়ার্ড করে) |

বুটের শুরুতে `/var` হয়তো এখনো মাউন্ট হয়নি, তাই journald প্রথমে `/run`-এ লেখে, তারপর `systemd-journal-flush.service` সব ডিস্কে সরিয়ে নেয়।

### তোমার সিস্টেম persistent কি না দেখো

```bash
$ ls -d /var/log/journal
/var/log/journal
$ journalctl --list-boots | wc -l
12
```

ডিরেক্টরি থাকলে আর একাধিক বুট দেখালে persistent।

### persistent করার পদ্ধতি

পদ্ধতি ১ — ডিরেক্টরি বানানো (`Storage=auto` হলে যথেষ্ট):

```bash
$ sudo mkdir -p /var/log/journal
$ sudo systemd-tmpfiles --create --prefix /var/log/journal
$ sudo systemctl restart systemd-journald
```

`systemd-tmpfiles` ঠিক মালিকানা ও পারমিশন (গ্রুপ `systemd-journal`, ACL) বসিয়ে দেয়।

পদ্ধতি ২ — কনফিগে স্পষ্ট বলে দেওয়া। মূল ফাইল না বদলে drop-in ফাইল বানানো ভালো:

```bash
$ sudo mkdir -p /etc/systemd/journald.conf.d
$ sudo nano /etc/systemd/journald.conf.d/10-persistent.conf
```

```ini
[Journal]
Storage=persistent
SystemMaxUse=1G
MaxRetentionSec=1month
```

```bash
$ sudo systemctl restart systemd-journald
```

### journald.conf-এর গুরুত্বপূর্ণ সেটিং

| সেটিং | অর্থ |
|---|---|
| `Storage=` | উপরে বলা হয়েছে |
| `SystemMaxUse=` | `/var/log/journal` সর্বোচ্চ কত জায়গা নেবে (ডিফল্ট: ফাইলসিস্টেমের ১০%, সর্বোচ্চ ৪G) |
| `SystemKeepFree=` | ডিস্কে অন্তত কত জায়গা খালি রাখবে (ডিফল্ট ১৫%, সর্বোচ্চ ৪G) |
| `SystemMaxFileSize=` | একটা journal ফাইলের সর্বোচ্চ সাইজ, তারপর নতুন ফাইল |
| `RuntimeMaxUse=` | `/run`-এর (RAM) জন্য একই সীমা |
| `MaxRetentionSec=` | কত পুরোনো এন্ট্রি রাখবে (যেমন `2week`, `1month`) |
| `Compress=` | বড় এন্ট্রি কম্প্রেস করবে কি না (ডিফল্ট yes) |
| `ForwardToSyslog=` | syslog socket-এ ফরোয়ার্ড করবে কি না |
| `RateLimitIntervalSec=`, `RateLimitBurst=` | একটা সার্ভিস নির্দিষ্ট সময়ে কতগুলো বার্তা পাঠাতে পারবে; বেশি হলে বাকিগুলো ফেলে দেয় |

> 💡 **টিপস:** লগে `Suppressed N messages from nginx.service` দেখলে বুঝবে rate limit-এ বার্তা বাদ গেছে। খুব বেশি লগ লেখা সার্ভিসের জন্য rate limit বাড়ানো যায়, কিন্তু আগে ভাবো — এত লগ আসছে কেন?

### জায়গা পরিষ্কার করা

```bash
$ journalctl --disk-usage
Archived and active journals take up 1.2G in the file system.

$ sudo journalctl --vacuum-size=500M       # মোট ৫০০M-এ নামাও
$ sudo journalctl --vacuum-time=2weeks     # ২ সপ্তাহের পুরোনো মোছো
$ sudo journalctl --vacuum-files=5         # সর্বোচ্চ ৫টা archived ফাইল রাখো
Vacuuming done, freed 712.0M of archived journals from /var/log/journal/....
```

> ⚠️ **সতর্কতা:** vacuum শুধু **archived** (বন্ধ হয়ে যাওয়া) journal ফাইল মোছে, চালু (active) ফাইল নয়। সবকিছু মুছতে চাইলে আগে `sudo journalctl --rotate` চালাও, তারপর vacuum। আর `/var/log/journal`-এর ফাইল কখনো হাতে `rm` করবে না যখন journald চলছে।

### journal-এ যারা ফাইল পড়ে — অন্য মেশিনের journal

ডেড সিস্টেমের ডিস্ক অন্য মেশিনে লাগিয়ে তার লগ পড়া যায়:

```bash
$ journalctl -D /mnt/broken/var/log/journal -b -1 -e
$ journalctl --file /path/to/system.journal
```

উদ্ধারের সময় এটা খুব কাজের ([অধ্যায় ৪৬](46-performance-troubleshooting.md)-এ ট্রাবলশুটিং দেখো)।

---

## ২৮.৮ `logger` ও `systemd-cat` — নিজে লগ লেখা

### logger

`logger` কমান্ড দিয়ে টার্মিনাল বা শেল স্ক্রিপ্ট থেকে syslog-এ বার্তা পাঠানো যায়। বার্তা journal-এ যায়, আর rsyslog থাকলে টেক্সট ফাইলেও।

**Syntax:** `logger [অপশন] "বার্তা"`

| অপশন | কাজ |
|---|---|
| `-p facility.level` | facility ও severity (ডিফল্ট `user.notice`) |
| `-t TAG` | ট্যাগ/identifier (ডিফল্ট: তোমার ইউজারনেম) |
| `-i` | প্রসেসের PID যোগ করো |
| `--id=PID` | নির্দিষ্ট PID যোগ করো |
| `-s` | বার্তা stderr-এও প্রিন্ট করো |
| `-f FILE` | ফাইলের প্রতিটা লাইন লগ করো |
| `-n HOST` | রিমোট syslog সার্ভারে পাঠাও |
| `-P PORT` | রিমোট পোর্ট (ডিফল্ট ৫১৪) |
| `-T` / `-d` | রিমোটে TCP / UDP ব্যবহার করো |
| `--journald[=FILE]` | journald-এ সরাসরি স্ট্রাকচার্ড এন্ট্রি পাঠাও (ফিল্ডসহ) |

উদাহরণ:

```bash
$ logger "হ্যালো লগ, এটা আমার প্রথম বার্তা"
$ journalctl -n 1
Nov 12 10:40:02 laptop rahim[5120]: হ্যালো লগ, এটা আমার প্রথম বার্তা

$ logger -t backup -p local0.err "Backup failed: disk full"
$ journalctl -t backup -n 1
Nov 12 10:41:15 laptop backup[5133]: Backup failed: disk full

$ journalctl -t backup -p err      # severity দিয়েও খুঁজে পাবে
```

স্ক্রিপ্টে বাস্তব ব্যবহার ([অধ্যায় ৪১](41-shell-scripting-1.md)-এ স্ক্রিপ্টিং শিখবে):

```bash
#!/bin/bash
TAG="nightly-backup"
logger -t "$TAG" -p local0.info "Backup started"
if rsync -a /home/ /mnt/backup/home/; then
    logger -t "$TAG" -p local0.info "Backup finished successfully"
else
    logger -t "$TAG" -p local0.err "Backup FAILED with exit code $?"
fi
```

এখন cron দিয়ে চালানো এই স্ক্রিপ্টের ফলাফল `journalctl -t nightly-backup` দিয়ে যেকোনো সময় দেখতে পাবে ([অধ্যায় ২৯](29-scheduling-cron.md))।

একটা কমান্ডের পুরো আউটপুট লগে পাঠাতে:

```bash
$ df -h | logger -t diskreport
```

### systemd-cat

`systemd-cat` কোনো কমান্ডের stdout/stderr সরাসরি journal-এ পাঠায়:

```bash
$ systemd-cat -t mytask -p info ls -l /etc/ssh
$ echo "manual note: server patched" | systemd-cat -t admin-note -p notice
$ journalctl -t admin-note
Nov 12 10:45:01 laptop admin-note[5201]: manual note: server patched
```

---

## ২৮.৯ `dmesg` — kernel-এর ডায়েরি

kernel বুট হওয়ার মুহূর্ত থেকেই বার্তা লেখে — কোন CPU পেল, কত RAM, কোন ডিস্ক, কোন ড্রাইভার লোড হলো, USB লাগানো হলো কি না। এগুলো যায় একটা নির্দিষ্ট সাইজের **ring buffer**-এ (মেমরিতে গোল চাকার মতো একটা জায়গা — ভরে গেলে সবচেয়ে পুরোনোটা মুছে নতুন লেখা হয়)। এই buffer পড়ার কমান্ড `dmesg` (display message)। ভেতরে এটা `/dev/kmsg` পড়ে।

```
   ring buffer (নির্দিষ্ট সাইজ)
   ┌────┬────┬────┬────┬────┬────┐
   │ m7 │ m8 │ m3 │ m4 │ m5 │ m6 │   ← নতুন m9 এলে সবচেয়ে পুরোনো m3 মুছে যাবে
   └────┴────┴──▲─┴────┴────┴────┘
                └ পরবর্তী লেখার জায়গা
```

এজন্য দীর্ঘদিন চলা সার্ভারে `dmesg`-এ বুটের শুরুর বার্তা নাও থাকতে পারে — তখন `journalctl -k -b` ব্যবহার করো।

```bash
$ dmesg
dmesg: read kernel buffer failed: Operation not permitted
$ sudo dmesg | head -5
[    0.000000] Linux version 6.8.0-48-generic (buildd@lcy02-amd64-040) ...
[    0.000000] Command line: BOOT_IMAGE=/boot/vmlinuz-6.8.0-48-generic root=UUID=... ro quiet splash
[    0.000000] KERNEL supported cpus:
[    0.000000]   Intel GenuineIntel
[    0.000000]   AMD AuthenticAMD
```

> 📝 **মনে রাখো:** Ubuntu ও Debian-এ `kernel.dmesg_restrict = 1` থাকে, তাই সাধারণ ইউজার `dmesg` চালাতে পারে না, `sudo` লাগে। বাঁ পাশের `[    0.000000]` মানে বুটের পর কত সেকেন্ড পরে বার্তাটা এসেছে।

### dmesg-এর অপশন

| অপশন | কাজ |
|---|---|
| `-T` | মানুষের পড়ার মতো তারিখ-সময় (সাসপেন্ড/রিজিউমের পর সময় একটু ভুল হতে পারে) |
| `-H` | human-readable: রঙ, pager, আপেক্ষিক সময় |
| `-w` | follow — নতুন বার্তা আসলে দেখাবে (`--follow`) |
| `-l LEVEL` | নির্দিষ্ট লেভেল, যেমন `-l err,warn` |
| `-f FACILITY` | নির্দিষ্ট facility, যেমন `-f kern` |
| `-k` | শুধু kernel facility |
| `-x` | facility ও level লেখায় দেখাও |
| `-L` | রঙিন আউটপুট |
| `-c` | পড়ে buffer খালি করো |
| `-C` | না দেখিয়ে buffer খালি করো |

```bash
$ sudo dmesg -T -l err,warn
[Wed Nov 12 09:01:14 2026] ACPI BIOS Error (bug): Could not resolve symbol [\_SB.PCI0.GPP0], AE_NOT_FOUND
[Wed Nov 12 09:01:16 2026] usb 1-3: device descriptor read/64, error -71
```

USB ড্রাইভ লাগানোর সময় কী হয় দেখতে, একটা টার্মিনালে চালাও তারপর পেনড্রাইভ লাগাও:

```bash
$ sudo dmesg -Tw
[Wed Nov 12 10:50:21 2026] usb 2-1: new SuperSpeed USB device number 3 using xhci_hcd
[Wed Nov 12 10:50:21 2026] usb-storage 2-1:1.0: USB Mass Storage device detected
[Wed Nov 12 10:50:22 2026] sd 1:0:0:0: [sdb] 60555264 512-byte logical blocks: (31.0 GB/28.9 GiB)
[Wed Nov 12 10:50:22 2026]  sdb: sdb1
```

এখান থেকে জানলে নতুন ডিস্কের নাম `sdb` আর পার্টিশন `sdb1` — মাউন্ট করার জন্য ঠিক এই তথ্যই লাগে ([অধ্যায় ৩২](32-disk-partition-mount.md))।

dmesg-এ যে গুরুত্বপূর্ণ বার্তাগুলো চিনে রাখবে:

| বার্তায় যা লেখা থাকে | মানে |
|---|---|
| `Out of memory: Killed process 1234 (java)` | RAM শেষ, OOM killer প্রসেস মেরেছে |
| `I/O error, dev sda, sector ...` | ডিস্কে পড়া/লেখার সমস্যা — ডিস্ক নষ্ট হচ্ছে হতে পারে |
| `EXT4-fs error` / `XFS ... Corruption` | ফাইলসিস্টেম করাপশন |
| `segfault at ... in libxyz.so` | কোনো প্রোগ্রাম ক্র্যাশ করেছে |
| `link is not ready` / `Link is Up` | নেটওয়ার্ক কেবল খোলা/লাগানো |
| `temperature above threshold` | CPU অতিরিক্ত গরম |
| `audit: ... apparmor="DENIED"` | AppArmor কিছু ব্লক করেছে ([অধ্যায় ৪৫](45-security-hardening.md)) |

kernel-এর ভেতরের গঠন ও লগ লেভেল (`/proc/sys/kernel/printk`) নিয়ে আরও জানতে [অধ্যায় ৪৭](47-kernel-proc-sys.md)।

---

## ২৮.১০ লগইন ইতিহাস: `last`, `lastb`, `lastlog`, `who`

```bash
$ last -n 5
rahim    pts/0        192.168.1.50     Wed Nov 12 10:15   still logged in
rahim    tty2         tty2             Wed Nov 12 09:02   still logged in
reboot   system boot  6.8.0-48-generic Wed Nov 12 09:01   still running
rahim    pts/1        192.168.1.50     Tue Nov 11 18:20 - 19:05  (00:45)
reboot   system boot  6.8.0-48-generic Tue Nov 11 09:00 - 22:15  (13:14)

wtmp begins Sat Nov  1 08:10:02 2026
```

| কমান্ড | কী দেখায় | ফাইল |
|---|---|---|
| `last` | সফল লগইন, লগআউট, রিবুট | `/var/log/wtmp` |
| `last reboot` | শুধু রিবুটের ইতিহাস | `/var/log/wtmp` |
| `last -x` | shutdown ও runlevel পরিবর্তনও | `/var/log/wtmp` |
| `last rahim` | নির্দিষ্ট ইউজারের | `/var/log/wtmp` |
| `sudo lastb` | **ব্যর্থ** লগইন চেষ্টা | `/var/log/btmp` |
| `lastlog` | প্রতিটা ইউজারের সবশেষ লগইন | `/var/log/lastlog` |
| `who` / `w` | এই মুহূর্তে কে লগইন আছে | `/run/utmp` |

```bash
$ sudo lastb -n 3
admin    ssh:notty    203.0.113.45     Wed Nov 12 03:14 - 03:14  (00:00)
root     ssh:notty    203.0.113.45     Wed Nov 12 03:14 - 03:14  (00:00)
test     ssh:notty    198.51.100.7     Wed Nov 12 02:58 - 02:58  (00:00)
```

ইন্টারনেটে খোলা যেকোনো SSH সার্ভারে এমন লাইন হাজারে হাজারে থাকবে — বট সারাক্ষণ চেষ্টা করছে। এজন্যই পাসওয়ার্ড লগইন বন্ধ করে key ব্যবহার করা উচিত ([অধ্যায় ৩৮](38-ssh-remote.md))।

---

## ২৮.১১ `logrotate` — লগ ফাইলের রক্ষণাবেক্ষণ

### সমস্যা

লগ ফাইল শুধু বাড়ে, কখনো ছোট হয় না। একটা ব্যস্ত ওয়েব সার্ভারের `access.log` দিনে কয়েক GB হতে পারে। কিছু না করলে একদিন ডিস্ক ভরে যাবে, আর ডিস্ক ভরা মানে ডাটাবেস লিখতে পারবে না, সার্ভিস বন্ধ হবে।

### সমাধান: rotation (ঘোরানো)

উপমা: অফিসের রেজিস্টার খাতা। মাস শেষে দারোয়ান খাতাটা বন্ধ করে আলমারিতে রাখে, নতুন খাতা শুরু করে। এক বছরের পুরোনো খাতা ফেলে দেওয়া হয়। `logrotate` ঠিক এটাই করে:

```
rotation-এর আগে:          rotation-এর পরে (rotate 4, compress, delaycompress):
syslog       (বড়)    ──►  syslog          (নতুন, খালি)
syslog.1                   syslog.1        (একটু আগের syslog)
syslog.2.gz                syslog.2.gz     (আগের syslog.1, এখন কম্প্রেসড)
syslog.3.gz                syslog.3.gz
syslog.4.gz                syslog.4.gz
                           (আগের syslog.4.gz মুছে গেছে)
```

### logrotate কে চালায়

logrotate নিজে ডেমন নয়; এটা একবার চলে, কাজ করে, বের হয়ে যায়। আধুনিক distro-তে (Ubuntu 24.04, Debian 12, Fedora, RHEL 9) একটা systemd timer প্রতিদিন চালায়:

```bash
$ systemctl list-timers logrotate.timer
NEXT                        LEFT     LAST                        PASSED  UNIT            ACTIVATES
Thu 2026-11-13 00:00:00 +06 13h left Wed 2026-11-12 00:00:03 +06 10h ago logrotate.timer logrotate.service
```

পুরোনো সিস্টেমে `/etc/cron.daily/logrotate` স্ক্রিপ্ট দিয়ে চলত। কোন ফাইল শেষবার কবে rotate হয়েছে তা মনে রাখে state ফাইলে: Debian/Ubuntu-তে `/var/lib/logrotate/status`, RHEL/Fedora-তে `/var/lib/logrotate/logrotate.status`।

### কনফিগ ফাইল

```
/etc/logrotate.conf      ← গ্লোবাল ডিফল্ট + "include /etc/logrotate.d"
/etc/logrotate.d/        ← প্রতিটা প্যাকেজের নিজস্ব ফাইল (nginx, rsyslog, apt ...)
```

Ubuntu-র `/etc/logrotate.d/rsyslog` (সংক্ষেপে):

```
/var/log/syslog
/var/log/mail.log
/var/log/kern.log
/var/log/auth.log
{
        rotate 4
        weekly
        missingok
        notifempty
        compress
        delaycompress
        sharedscripts
        postrotate
                /usr/lib/rsyslog/rsyslog-rotate
        endscript
}
```

nginx-এর `/etc/logrotate.d/nginx` (Ubuntu, সংক্ষেপে):

```
/var/log/nginx/*.log {
        daily
        missingok
        rotate 14
        compress
        delaycompress
        notifempty
        create 0640 www-data adm
        sharedscripts
        postrotate
                invoke-rc.d nginx rotate >/dev/null 2>&1
        endscript
}
```

### গুরুত্বপূর্ণ directive

| directive | অর্থ |
|---|---|
| `daily` / `weekly` / `monthly` / `yearly` | কত ঘনঘন rotate |
| `rotate N` | কতগুলো পুরোনো কপি রাখবে (তারপর মুছবে); `rotate 0` মানে কোনো কপি নয় |
| `size 100M` | সাইজ ১০০M ছাড়ালে rotate (সময় যাই হোক) |
| `maxsize 500M` | সময় না হলেও এই সাইজ ছাড়ালে rotate |
| `minsize 1M` | সময় হলেও অন্তত এই সাইজ না হলে rotate নয় |
| `compress` | পুরোনো কপি gzip করো |
| `delaycompress` | সর্বশেষ পুরোনো কপি (`.1`) কম্প্রেস না করে পরের বারে করো |
| `missingok` | ফাইল না থাকলে error দেখিও না |
| `notifempty` | ফাইল খালি হলে rotate কোরো না |
| `create MODE OWNER GROUP` | rotate-এর পর নতুন খালি ফাইল এই পারমিশনে বানাও |
| `copytruncate` | ফাইলটা কপি করে আসল ফাইলটা শূন্য করে দাও (রিনেম না করে) |
| `postrotate ... endscript` | rotate-এর পর এই কমান্ড চালাও |
| `prerotate ... endscript` | rotate-এর আগে চালাও |
| `sharedscripts` | একাধিক ফাইল থাকলেও postrotate একবারই চালাও |
| `dateext` | `.1`-এর বদলে তারিখ দিয়ে নাম, যেমন `app.log-20261112` |
| `maxage N` | N দিনের পুরোনো rotated ফাইল মুছে দাও |
| `olddir DIR` | পুরোনো কপি আলাদা ডিরেক্টরিতে রাখো |
| `su USER GROUP` | এই ইউজার/গ্রুপ হিসেবে rotate করো (ডিরেক্টরির পারমিশন সমস্যা এড়াতে) |

### ভেতরে কী ঘটে: postrotate ও copytruncate কেন লাগে

এটা লিনাক্সের একটা সূক্ষ্ম কিন্তু খুব গুরুত্বপূর্ণ বিষয়। nginx যখন `access.log` খোলে, kernel তাকে একটা **file descriptor** দেয়, যা আসলে ফাইলের নাম নয়, ফাইলের **inode**-কে নির্দেশ করে ([অধ্যায় ১১](11-links-inode.md))।

```
rotate-এর আগে:      nginx ──fd──► inode 5001  (নাম: access.log)

logrotate রিনেম করল: nginx ──fd──► inode 5001  (নাম এখন: access.log.1)
                    নতুন access.log = inode 5002 (খালি)

ফলাফল: nginx এখনো পুরোনো access.log.1-এ লিখছে! নতুন ফাইল খালিই থাকে।
```

সমাধান দুইটা:

1. **postrotate-এ সিগন্যাল পাঠানো** (ভালো পদ্ধতি): nginx-কে `USR1` সিগন্যাল (বা `rotate`/`reload` কমান্ড) পাঠালে সে লগ ফাইল নতুন করে খোলে, এবং নতুন inode 5002-তে লিখতে শুরু করে। এজন্য উপরের কনফিগে postrotate আছে। (সিগন্যাল নিয়ে [অধ্যায় ২৪](24-process-management.md))
2. **copytruncate**: যে প্রোগ্রাম ফাইল নতুন করে খুলতে জানে না, তার জন্য। logrotate ফাইলের কপি বানায়, তারপর আসল ফাইলটা শূন্য করে দেয় — inode একই থাকে। অসুবিধা: কপি আর truncate-এর মাঝের মুহূর্তে লেখা কয়েকটা লাইন হারিয়ে যেতে পারে।

> ⚠️ **সতর্কতা:** লগ ফাইল বড় হয়ে গেলে চলমান সার্ভিসের লগ `rm` দিয়ে মুছবে না! ফাইলটা ডিরেক্টরি থেকে চলে যাবে, কিন্তু প্রোগ্রাম তখনো খোলা রাখায় ডিস্কের জায়গা খালি হবে না (`df` আগের মতোই ভরা দেখাবে)। নিরাপদ উপায়: `sudo truncate -s 0 /var/log/big.log` অথবা `: | sudo tee /var/log/big.log`। মুছে ফেলা কিন্তু খোলা ফাইল খুঁজতে: `sudo lsof +L1`।

### নিজের অ্যাপের জন্য logrotate কনফিগ

```bash
$ sudo nano /etc/logrotate.d/myapp
```

```
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 myapp adm
    dateext
    maxsize 200M
    sharedscripts
    postrotate
        systemctl reload myapp.service >/dev/null 2>&1 || true
    endscript
}
```

### পরীক্ষা করা

```bash
# -d (debug): কী করবে দেখাবে, কিন্তু কিছু করবে না — সবসময় আগে এটা চালাও
$ sudo logrotate -d /etc/logrotate.d/myapp
reading config file /etc/logrotate.d/myapp
...
considering log /var/log/myapp/app.log
  log does not need rotating (log has already been rotated)

# -f (force): সময় না হলেও এখনই rotate করো
$ sudo logrotate -f /etc/logrotate.d/myapp

# -v (verbose): করার সময় বিস্তারিত দেখাও
$ sudo logrotate -vf /etc/logrotate.conf
```

> 📝 **মনে রাখো:** logrotate শুধু **টেক্সট লগ ফাইল** দেখে। systemd journal নিজের সাইজ নিজে সামলায় (`SystemMaxUse=` ইত্যাদি)। এই দুটো আলাদা ব্যবস্থা।

---

## ২৮.১২ ওয়েব সার্ভারের লগ: nginx উদাহরণ

nginx দুই ধরনের লগ লেখে:

- **access.log** — প্রতিটা HTTP রিকোয়েস্টের এক লাইন
- **error.log** — সমস্যা, কনফিগ error, upstream ব্যর্থতা

access.log-এর একটা লাইন (combined ফরম্যাট):

```
192.168.1.50 - - [12/Nov/2026:10:55:12 +0600] "GET /index.html HTTP/1.1" 200 615 "-" "Mozilla/5.0 (X11; Linux x86_64)"
└─────┬────┘     └───────────┬────────────┘ └──────────┬──────────┘ └┬┘ └┬┘ └┬┘ └──────────┬──────────┘
  ক্লায়েন্ট IP           সময়                     রিকোয়েস্ট      স্ট্যাটাস বাইট রেফারার    ব্রাউজার (user agent)
```

[অধ্যায় ২২](22-awk.md) ও [অধ্যায় ২৩](23-text-tools.md)-এর টুল দিয়ে বিশ্লেষণ:

```bash
# সবচেয়ে বেশি রিকোয়েস্ট পাঠানো ১০টা IP
$ sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
   4521 203.0.113.10
    880 192.168.1.50
    ...

# স্ট্যাটাস কোড অনুযায়ী গণনা
$ sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
  15230 200
    412 404
     37 500

# সব 5xx error
$ sudo awk '$9 ~ /^5/' /var/log/nginx/access.log | tail

# সবচেয়ে বেশি 404 পাওয়া URL
$ sudo awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# error.log লাইভ
$ sudo tail -F /var/log/nginx/error.log
2026/11/12 10:58:03 [error] 1290#1290: *77 connect() failed (111: Connection refused) while connecting to upstream, client: 192.168.1.50, server: example.com, request: "GET /api HTTP/1.1", upstream: "http://127.0.0.1:8000/api"
```

শেষের লাইনটা পড়লেই বোঝা যায় — nginx ঠিক আছে, কিন্তু পেছনের অ্যাপ (পোর্ট ৮০০০) চলছে না। লগ ঠিক এভাবেই সমস্যার দিক দেখায়। ওয়েব সার্ভার সেটআপ বিস্তারিত [অধ্যায় ৪০](40-web-server-services.md)-এ।

> 💡 **টিপস:** টার্মিনালে লগ ঘাঁটার জন্য `lnav` (Log File Navigator) চমৎকার একটা টুল — রঙ, সময় অনুযায়ী একাধিক ফাইল মেলানো, SQL দিয়ে কুয়েরি। `sudo apt install lnav` / `sudo dnf install lnav` দিয়ে ইনস্টল করে `sudo lnav /var/log/nginx/` চালিয়ে দেখো।

---

## ২৮.১৩ লগ দিয়ে সমস্যা খোঁজার পদ্ধতি

অভিজ্ঞ অ্যাডমিনরা একটা নির্দিষ্ট ক্রমে চিন্তা করে। এটাকে মুখস্থ করে ফেলো:

```
  ১. লক্ষণ স্পষ্ট করো ──► "কী কাজ করছে না? কখন থেকে? কার জন্য?"
          │
  ২. সময় চিহ্নিত করো ──► সমস্যা প্রথম কখন হয়েছে → --since/--until
          │
  ৩. সঠিক লগ বাছো    ──► কোন সার্ভিস? → journalctl -u / নিজস্ব লগ ফাইল
          │
  ৪. গুরুতর আগে       ──► -p err, grep -i "error|fail|denied|refused|timeout"
          │
  ৫. প্রথম error খোঁজো ──► শেষেরটা নয়! প্রথমটাই সাধারণত মূল কারণ
          │
  ৬. প্রসঙ্গ পড়ো      ──► error-এর আগের-পরের লাইন (grep -B5 -A5)
          │
  ৭. নিচের স্তরে যাও  ──► kernel (dmesg), ডিস্ক (df -h), মেমরি (OOM), নেটওয়ার্ক
          │
  ৮. পুনরুৎপাদন করো   ──► journalctl -f চালিয়ে সমস্যাটা আবার ঘটাও
          │
  ৯. ঠিক করো ও যাচাই করো ─► লগে আর error আসছে না নিশ্চিত হও
```

> 📝 **মনে রাখো:** একটা মূল সমস্যা প্রায়ই অনেকগুলো পরবর্তী error তৈরি করে (cascade)। যেমন ডিস্ক ভরে গেলে ডাটাবেস error দেয়, তারপর অ্যাপ error দেয়, তারপর nginx 502 দেয়। শেষের 502 দেখে nginx ঘাঁটলে সময় নষ্ট হবে — **সবচেয়ে আগের error**-টা খোঁজো।

### সাধারণ পরিস্থিতি ও কোথায় দেখবে

| সমস্যা | প্রথমে কী চালাবে |
|---|---|
| সার্ভিস চালু হচ্ছে না | `systemctl status X`, `journalctl -xeu X` |
| কম্পিউটার হঠাৎ রিবুট/বন্ধ হয়েছে | `journalctl -b -1 -e`, `last -x reboot` |
| প্রসেস রহস্যজনকভাবে মরে গেছে | `journalctl -k -g "Out of memory"`, `dmesg -T \| grep -i oom` |
| কেউ লগইন করেছে কি না | `/var/log/auth.log` বা `secure`, `last`, `lastb`, `journalctl -u ssh` |
| `sudo` দিয়ে কে কী চালিয়েছে | `journalctl _COMM=sudo` বা `grep sudo /var/log/auth.log` |
| ডিস্ক বা USB সমস্যা | `dmesg -T -l err,warn`, `journalctl -k -p err` |
| প্যাকেজ আপডেটের পর কিছু ভেঙেছে | `/var/log/apt/history.log`, `/var/log/dpkg.log`, `dnf history` |
| ওয়েবসাইট 502/500 দেখাচ্ছে | nginx `error.log`, তারপর অ্যাপের লগ |
| RHEL-এ পারমিশন ঠিক থাকলেও "Permission denied" | `/var/log/audit/audit.log`, `ausearch -m AVC -ts recent` (SELinux) |
| cron জব চলেনি | `journalctl -u cron` (Debian/Ubuntu) বা `-u crond` (RHEL), `/var/log/cron` |
| ডিস্ক ভরে যাচ্ছে | `sudo du -sh /var/log/* \| sort -h`, `journalctl --disk-usage` |

### উদাহরণ ১: SSH brute-force শনাক্ত

```bash
$ sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -5
   1532 203.0.113.45
    410 198.51.100.7
     12 192.168.1.77
```

(`$(NF-3)` মানে শেষ থেকে চার নম্বর ফিল্ড, যেখানে "from IP port N ssh2" ধাঁচের লাইনে IP থাকে।) RHEL-এ একই কাজ `/var/log/secure`-এ, আর journal দিয়ে:

```bash
$ sudo journalctl -u ssh --since today -g "Failed password" --no-pager | wc -l
```

### উদাহরণ ২: সার্ভিস স্টার্ট হচ্ছে না

```bash
$ sudo systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.

$ sudo journalctl -xeu nginx --no-pager | tail -5
Nov 12 11:05:10 web nginx[3301]: nginx: [emerg] unknown directive "servr_name" in /etc/nginx/sites-enabled/default:23
Nov 12 11:05:10 web nginx[3301]: nginx: configuration file /etc/nginx/nginx.conf test failed
Nov 12 11:05:10 web systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
Nov 12 11:05:10 web systemd[1]: nginx.service: Failed with result 'exit-code'.
```

লগ সরাসরি বলে দিচ্ছে: ফাইল `default`, লাইন ২৩, বানান ভুল `servr_name`।

### উদাহরণ ৩: আগের রাতে সার্ভার কেন রিবুট হলো

```bash
$ last -x reboot shutdown | head -3
$ journalctl -b -1 -n 30 --no-pager
```

শেষ লাইনগুলোয় যদি স্বাভাবিক shutdown বার্তা থাকে (`Reached target ... Power-Off`/`System Reboot`), তাহলে কেউ বা কিছু ইচ্ছাকৃত রিবুট করেছে (`unattended-upgrades` স্বয়ংক্রিয় রিবুটও করতে পারে)। কোনো shutdown বার্তা ছাড়াই লগ হঠাৎ থেমে গেলে — বিদ্যুৎ বিভ্রাট, kernel panic, বা হার্ডওয়্যার সমস্যা।

### উদাহরণ ৪: একটা সময়ের জানালায় সব কিছু

"১০:৪০ থেকে ১০:৪৫-এর মধ্যে সাইট ডাউন ছিল" — তখন সব উৎস একসাথে দেখো:

```bash
$ sudo journalctl --since "10:40" --until "10:45" -p warning --no-pager
```

---

## হাতে-কলমে অনুশীলন

> ⚠️ **সতর্কতা:** এই ল্যাবগুলো নিজের ল্যাপটপ বা ভার্চুয়াল মেশিনে করো, প্রোডাকশন সার্ভারে নয়।

### ল্যাব ১: লগ ফাইলের সাথে পরিচিতি

1. `ls -lh /var/log` চালাও। কোন ফাইলগুলো সবচেয়ে বড়?
2. তোমার distro-তে প্রধান টেক্সট লগ কোনটা — `syslog` না `messages`? নাকি কোনোটাই নেই?
3. `sudo tail -n 20 /var/log/auth.log` (বা `/var/log/secure`) চালাও।
4. আরেকটা টার্মিনালে `sudo ls` চালাও, তারপর আবার auth লগের শেষে দেখো — তোমার sudo কমান্ড রেকর্ড হয়েছে?

### ল্যাব ২: journalctl অনুশীলন

```bash
$ journalctl --list-boots
$ journalctl -b -p err --no-pager
$ journalctl -u ssh --since today        # RHEL/Fedora: -u sshd
$ journalctl -k -n 20
$ journalctl --disk-usage
$ journalctl -n 1 -o json-pretty
$ journalctl -F _SYSTEMD_UNIT | wc -l
```

প্রতিটার আউটপুট দেখে নিজেকে বলো এটা কী দেখাচ্ছে।

### ল্যাব ৩: লাইভ লগ দেখা

1. টার্মিনাল-১: `journalctl -f`
2. টার্মিনাল-২: `logger -t lab "Live test message"`
3. টার্মিনাল-১-এ বার্তাটা সাথে সাথে দেখা গেল? Ctrl+C দিয়ে থামাও।
4. এবার টার্মিনাল-১-এ `journalctl -f -p err` চালিয়ে টার্মিনাল-২-এ `logger -p user.err -t lab "an error"` আর `logger -p user.info -t lab "just info"` চালাও। কোনটা দেখা গেল, কেন?

### ল্যাব ৪: নিজের অ্যাপের জন্য লগিং (Ubuntu/RHEL, যেখানে rsyslog আছে)

1. `/etc/rsyslog.d/30-lab.conf` বানাও:
   ```
   local5.*    /var/log/lab.log
   & stop
   ```
2. `sudo rsyslogd -N1` দিয়ে যাচাই, তারপর `sudo systemctl restart rsyslog`।
3. `logger -p local5.warning -t labapp "first lab entry"`
4. `sudo cat /var/log/lab.log` — বার্তাটা আছে? `/var/log/syslog`-এ নেই তো (`& stop`-এর কারণে)?
5. ফাইল না বানানো গেলে উপরের সতর্কতা অনুযায়ী `touch` ও `chown syslog:adm` করো।

### ল্যাব ৫: logrotate পরীক্ষা

1. `/etc/logrotate.d/lab` বানাও:
   ```
   /var/log/lab.log {
       rotate 3
       daily
       compress
       missingok
       notifempty
       create 0640 syslog adm
       postrotate
           /usr/lib/rsyslog/rsyslog-rotate
       endscript
   }
   ```
   (RHEL-এ postrotate-এর ভেতরে `systemctl kill -s HUP rsyslog.service` দাও এবং ইউজার `root` রাখো।)
2. `sudo logrotate -d /etc/logrotate.d/lab` — কী করবে পড়ো।
3. কয়েকবার `logger -p local5.info "line"` চালিয়ে `sudo logrotate -f /etc/logrotate.d/lab` চালাও।
4. `ls -l /var/log/lab.log*` — `lab.log.1.gz` তৈরি হয়েছে? নতুন বার্তা `lab.log`-এ যাচ্ছে?

### ল্যাব ৬: dmesg ও হার্ডওয়্যার

1. `sudo dmesg -T | tail` চালাও।
2. `sudo dmesg -Tw` চালু রেখে একটা পেনড্রাইভ লাগাও ও খোলো। ডিভাইসের নাম (`sdb` ইত্যাদি) বের করো।
3. `sudo dmesg -T -l err,warn` — কোনো সতর্কতা আছে? সেগুলোর মানে ইন্টারনেটে খুঁজে দেখো।

### ল্যাব ৭: journal persistent করা

1. `ls -d /var/log/journal` — আছে কি?
2. না থাকলে ২৮.৭-এর পদ্ধতিতে persistent করো, তারপর রিবুট করে `journalctl --list-boots` দেখো।
3. `/etc/systemd/journald.conf.d/size.conf`-এ `SystemMaxUse=300M` দিয়ে `systemd-journald` রিস্টার্ট করো।

### ল্যাব ৮: ছোট গোয়েন্দা কাজ

নিচের প্রশ্নগুলোর উত্তর শুধু লগ দেখে বের করো:
- এই কম্পিউটার শেষবার কখন রিবুট হয়েছিল?
- আজ কতবার `sudo` চালানো হয়েছে? (`journalctl _COMM=sudo --since today | grep -c COMMAND`)
- সবশেষ কোন প্যাকেজ ইনস্টল/আপডেট হয়েছে? (`tail /var/log/apt/history.log` বা `dnf history | head`)
- বর্তমান বুটে কোন সার্ভিস ফেইল করেছে? (`systemctl --failed` এবং `journalctl -b -p err`)

---

## সারসংক্ষেপ

| কাজ | কমান্ড |
|---|---|
| সাধারণ সিস্টেম লগ | `/var/log/syslog` (Debian/Ubuntu), `/var/log/messages` (RHEL) |
| অথেন্টিকেশন লগ | `/var/log/auth.log` (Debian/Ubuntu), `/var/log/secure` (RHEL) |
| লগের শেষ অংশ / লাইভ | `tail -n 50 FILE`, `tail -F FILE`, `less +F FILE` |
| কম্প্রেসড লগে খোঁজা | `zgrep PATTERN FILE.gz`, `zcat FILE.gz` |
| সব journal | `journalctl` |
| একটা সার্ভিস | `journalctl -u NAME` |
| লাইভ | `journalctl -f`, `journalctl -fu NAME` |
| এই বুট / আগের বুট | `journalctl -b`, `journalctl -b -1`, `journalctl --list-boots` |
| শুধু error | `journalctl -p err -b` |
| সময়ের জানালা | `journalctl --since "..." --until "..."` |
| kernel বার্তা | `journalctl -k`, `sudo dmesg -T` |
| শেষে + ব্যাখ্যা | `journalctl -xeu NAME` |
| JSON আউটপুট | `journalctl -o json` / `-o json-pretty` |
| ফিল্ড দিয়ে | `journalctl _PID=N`, `_UID=N`, `_COMM=X`, `-t TAG` |
| বার্তায় খোঁজা | `journalctl -g PATTERN` |
| journal সাইজ / পরিষ্কার | `journalctl --disk-usage`, `--vacuum-size=500M`, `--vacuum-time=2weeks` |
| persistent journal | `mkdir -p /var/log/journal` বা `Storage=persistent` |
| নিজে লগ লেখা | `logger -t TAG -p local0.err "msg"`, `systemd-cat -t TAG CMD` |
| kernel buffer লাইভ | `sudo dmesg -Tw` |
| লগইন ইতিহাস | `last`, `sudo lastb`, `lastlog`, `last reboot` |
| rsyslog কনফিগ যাচাই | `sudo rsyslogd -N1` |
| logrotate পরীক্ষা / জোর করে | `sudo logrotate -d FILE`, `sudo logrotate -f FILE` |
| মুছে ফেলা কিন্তু খোলা ফাইল | `sudo lsof +L1` |

| শর্টকাট (less / journalctl pager) | কাজ |
|---|---|
| `G` / `g` | শেষে / শুরুতে |
| `/` , `?` , `n` , `N` | খোঁজা ও পরের/আগের মিল |
| `F` | লাইভ follow মোড |
| `←` / `→` | লম্বা লাইন পাশে স্ক্রল |
| `q` | বের হওয়া |
| `Ctrl+C` | `tail -f` / `journalctl -f` / `dmesg -w` থামানো |

---

## নিজেকে যাচাই করো

1. Ubuntu-তে SSH লগইনের রেকর্ড কোন ফাইলে থাকে, আর RHEL 9-এ কোন ফাইলে?
2. syslog-এর severity লেভেলগুলো গুরুতর থেকে কম গুরুতর ক্রমে বলো। `err`-এর সংখ্যাসূচক মান কত?
3. rsyslog কনফিগে `mail.err` আর `mail.=err`-এর পার্থক্য কী?
4. আগের বুটের শুধু kernel বার্তা দেখার কমান্ড কী? এটা `dmesg` দিয়ে কেন সম্ভব নয়?
5. `Storage=auto` থাকলে journal কখন ডিস্কে আর কখন RAM-এ থাকে?
6. গতকাল সন্ধ্যা ৬টা থেকে ৭টার মধ্যে `nginx` সার্ভিসের লগ দেখার কমান্ড লেখো।
7. logrotate-এ `postrotate` স্ক্রিপ্টে সার্ভিসকে সিগন্যাল বা reload কেন পাঠানো হয়? `copytruncate` কখন ব্যবহার করবে?
8. একটা বড় লগ ফাইল `rm` দিয়ে মুছে দেওয়ার পরেও `df -h` জায়গা খালি দেখাচ্ছে না কেন? সমাধান কী?
9. একটা শেল স্ক্রিপ্ট থেকে `local2` facility আর `warning` severity দিয়ে "Disk almost full" বার্তা পাঠানোর কমান্ড কী?
10. কোনো প্রসেস হঠাৎ মরে গেছে, সন্দেহ RAM শেষ হয়ে গিয়েছিল। কোন কমান্ড দিয়ে নিশ্চিত হবে?

<details>
<summary>উত্তর দেখো</summary>

1. Ubuntu/Debian: `/var/log/auth.log`; RHEL/Fedora-পরিবার (rsyslog থাকলে): `/var/log/secure`। দুই জায়গাতেই `journalctl -u ssh` (বা `-u sshd`) দিয়েও দেখা যায়।
2. emerg (0), alert (1), crit (2), err (3), warning (4), notice (5), info (6), debug (7)। `err`-এর মান ৩।
3. `mail.err` মানে mail facility-র `err` এবং তার চেয়ে গুরুতর সব (crit, alert, emerg); `mail.=err` মানে শুধু ঠিক `err` লেভেল।
4. `journalctl -k -b -1`। `dmesg` শুধু বর্তমানে চলা kernel-এর মেমরির ring buffer পড়ে, রিবুটে সেটা মুছে যায়; অন্যদিকে persistent journal আগের বুটের kernel বার্তা ডিস্কে রেখে দেয়।
5. `/var/log/journal` ডিরেক্টরি থাকলে ডিস্কে (persistent), না থাকলে `/run/log/journal`-এ RAM-এ (volatile), যা রিবুটে মুছে যায়।
6. `journalctl -u nginx --since "yesterday 18:00" --until "yesterday 19:00"` অথবা নির্দিষ্ট তারিখ দিয়ে `--since "2026-11-11 18:00" --until "2026-11-11 19:00"`।
7. সার্ভিস ফাইলটা খোলা রাখে file descriptor (অর্থাৎ inode) দিয়ে, নাম দিয়ে নয়; রিনেমের পরও সে পুরোনো ফাইলে লিখতে থাকে। সিগন্যাল/reload পেলে সে নতুন ফাইল খোলে। যে প্রোগ্রাম লগ ফাইল নতুন করে খুলতে পারে না, তার জন্য `copytruncate` — তবে এতে অল্প কিছু লাইন হারানোর ঝুঁকি থাকে।
8. প্রোগ্রামটা ফাইল তখনো খোলা রেখেছে, তাই inode ও ডেটা ব্লক মুক্ত হয়নি। সমাধান: সার্ভিস রিস্টার্ট/reload করা; ভবিষ্যতে `rm`-এর বদলে `truncate -s 0 FILE` ব্যবহার করা। এমন ফাইল খুঁজতে `sudo lsof +L1`।
9. `logger -p local2.warning -t myscript "Disk almost full"`
10. `journalctl -k -g "Out of memory"` (বা `-b -1` যোগ করে আগের বুটে), অথবা `sudo dmesg -T | grep -i -E "out of memory|oom"`। সেখানে `Killed process PID (নাম)` লেখা থাকবে।

</details>

---

[← আগের অধ্যায়](27-boot-process.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](29-scheduling-cron.md)

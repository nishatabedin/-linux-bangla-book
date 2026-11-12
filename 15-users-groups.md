# অধ্যায় ১৫: ইউজার ও গ্রুপ ম্যানেজমেন্ট

> "লিনাক্স একটা বড় অ্যাপার্টমেন্ট বিল্ডিং। প্রত্যেক ভাড়াটিয়ার নিজের ফ্ল্যাট আর নিজের চাবি আছে। দারোয়ান (kernel) নাম দেখে না, দেখে ফ্ল্যাট নম্বর। আর বিল্ডিংয়ের মালিক (root) সব দরজা খুলতে পারে।"

## এই অধ্যায়ে যা শিখবে

- লিনাক্স কেন **multi-user** সিস্টেম, আর এর ফলে নিরাপত্তা কীভাবে তৈরি হয়
- **UID** ও **GID** কী, kernel কেন নাম না দেখে শুধু নম্বর দেখে
- **system user** আর **regular user**-এর পার্থক্য, কোন নম্বর-রেঞ্জ কার জন্য
- `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow` ফাইলের প্রতিটি ফিল্ড
- ইউজার তৈরি, পরিবর্তন ও মুছে ফেলা: `useradd`, `adduser`, `usermod`, `userdel`
- পাসওয়ার্ড ও তার মেয়াদ: `passwd`, `chage`, `chpasswd`
- গ্রুপ ম্যানেজমেন্ট: `groupadd`, `groupmod`, `groupdel`, `gpasswd`, `newgrp`
- কে আমি, কে লগইন করে আছে: `id`, `groups`, `whoami`, `who`, `w`, `last`, `lastlog`
- `su` দিয়ে ইউজার বদলানো, **login shell** বনাম non-login shell
- `/etc/skel` দিয়ে নতুন ইউজারের ডিফল্ট ফাইল সাজানো
- `nologin` দিয়ে এমন অ্যাকাউন্ট বানানো যেটা দিয়ে লগইন করা যায় না
- ভেতরে কী ঘটে: login → PAM → NSS → kernel credential

---

## ১৫.১ multi-user ধারণা — লিনাক্স কেন একসাথে অনেকের

Windows-এর পুরোনো দিনের কম্পিউটার মূলত একজন মানুষের জন্য বানানো ছিল। লিনাক্সের পূর্বপুরুষ Unix জন্মেছিল ১৯৭০-এর দশকে, যখন একটা বড় কম্পিউটারে একসাথে অনেক মানুষ terminal দিয়ে কাজ করত। তাই শুরু থেকেই লিনাক্স **multi-user** (একসাথে বহু ব্যবহারকারী) এবং **multi-tasking** (একসাথে বহু কাজ) সিস্টেম।

আজ তোমার ল্যাপটপে হয়তো তুমি একাই মানুষ, কিন্তু তবুও সিস্টেমে ডজনখানেক "ইউজার" আছে: `root`, `www-data`, `systemd-network`, `messagebus`, `nobody` ইত্যাদি। এরা মানুষ না, এরা বিভিন্ন **service** (পেছনে চলা প্রোগ্রাম)-এর পরিচয়।

কেন এমন? কারণ **least privilege** (ন্যূনতম ক্ষমতা) নীতি: প্রত্যেক প্রোগ্রাম শুধু ততটুকু ক্ষমতা পাবে যতটুকু তার দরকার। ধরো ওয়েব সার্ভার `nginx` হ্যাক হলো। সেটা যদি `root` হিসেবে চলত, হ্যাকার পুরো সিস্টেম পেয়ে যেত। কিন্তু সেটা চলে `www-data` (Debian/Ubuntu) বা `nginx` (Fedora/RHEL) ইউজার হিসেবে, তাই হ্যাকার শুধু ওই ইউজারের সীমিত ফাইলেই হাত দিতে পারবে।

```
        ┌──────────────── লিনাক্স সিস্টেম ────────────────┐
        │                                                  │
        │   root (UID 0)  ── সব কিছুর মালিক, সীমাহীন ক্ষমতা │
        │                                                  │
        │   system users (UID 1–999)                       │
        │     www-data, sshd, systemd-*, mysql ...         │
        │     → service চালায়, সাধারণত লগইন করতে পারে না  │
        │                                                  │
        │   regular users (UID 1000+)                      │
        │     rahim, karim, sumi ...                       │
        │     → মানুষ, নিজের /home আছে, লগইন করে             │
        └──────────────────────────────────────────────────┘
```

**উপমা:** অ্যাপার্টমেন্ট বিল্ডিং। `root` = বাড়িওয়ালা, মাস্টার-কী আছে। system user = লিফটম্যান, গার্ড, ক্লিনার — প্রত্যেকে নিজের কাজের ঘরের চাবি পায়, কারো ফ্ল্যাটে ঢুকতে পারে না। regular user = ভাড়াটিয়া, নিজের ফ্ল্যাট (`/home/rahim`) আছে। **group** = একই ফ্লোরের ভাড়াটিয়ারা, যারা একটা কমন ছাদ বা স্টোররুম শেয়ার করে।

> 📝 **মনে রাখো:** লিনাক্সে প্রতিটি file ও প্রতিটি process-এর একজন মালিক (user) এবং একটি গ্রুপ থাকে। ইউজার-গ্রুপ না বুঝলে পরের অধ্যায়ের [ফাইল পারমিশন](16-permissions.md) বোঝা অসম্ভব।

---

## ১৫.২ UID ও GID — kernel নাম চেনে না, নম্বর চেনে

**UID (User ID)** হলো প্রতিটি ইউজারের একটি পূর্ণসংখ্যা পরিচয়-নম্বর। **GID (Group ID)** হলো প্রতিটি গ্রুপের নম্বর।

এখানে সবচেয়ে গুরুত্বপূর্ণ কথা: **kernel "rahim" নামটা জানে না।** kernel-এর কাছে rahim মানে শুধু `1000`। ডিস্কে প্রতিটি ফাইলের inode-এ (দেখো [অধ্যায় ১১](11-links-inode.md)) মালিকের নাম নয়, UID আর GID নম্বর লেখা থাকে। প্রতিটি process-এর ভেতরেও kernel UID/GID নম্বর রাখে।

তাহলে `ls -l` চালালে নাম দেখি কেন? কারণ `ls` নম্বরটা নিয়ে `/etc/passwd` (বা অন্য ডাটাবেস) খুঁজে নামে রূপান্তর করে দেখায়। নাম মানুষের সুবিধার জন্য, নম্বর মেশিনের জন্য।

```
     তুমি দেখো:   -rw-r--r-- 1 rahim rahim 120 report.txt
                               │      │
                     ls খোঁজে /etc/passwd ও /etc/group
                               │      │
     ডিস্কে আছে:   inode → owner UID = 1000, group GID = 1000
```

প্রমাণ দেখতে `ls -n` চালাও, এটা নাম না দেখিয়ে নম্বর দেখায়:

```bash
$ ls -l report.txt
-rw-r--r-- 1 rahim rahim 120 Nov 12 10:00 report.txt
$ ls -n report.txt
-rw-r--r-- 1 1000 1000 120 Nov 12 10:00 report.txt
```

> 💡 **টিপস:** কোনো ইউজার মুছে ফেললে তার ফাইলগুলো `ls -l`-এ নামের জায়গায় শুধু নম্বর দেখায় (যেমন `1003`)। কারণ নম্বরটা আছে, কিন্তু সেই নম্বরের নাম আর ডাটাবেসে নেই। এমন ফাইলকে বলে **orphan file**। খুঁজতে: `sudo find / -nouser -o -nogroup` ([অধ্যায় ১২](12-file-khoja.md))।

### UID-এর রেঞ্জ

| রেঞ্জ | কার জন্য | মন্তব্য |
|---|---|---|
| `0` | `root` | সুপারইউজার। নাম নয়, **UID 0** হওয়াটাই ক্ষমতার উৎস |
| `1–99` | distro-র স্থির (static) system অ্যাকাউন্ট | Debian-এ প্যাকেজ base-passwd এগুলো ঠিক করে |
| `100–999` (Debian/Ubuntu) / `201–999` (Fedora/RHEL) | ডাইনামিক system user | `useradd -r` এখান থেকে নম্বর দেয় |
| `1000–60000` | regular (মানুষ) ইউজার | প্রথম ইউজার সাধারণত `1000` পায় |
| `61184–65519` | systemd DynamicUser | service চালু থাকাকালীন অস্থায়ী ইউজার |
| `65534` | `nobody` / `nogroup` (Fedora/RHEL-এ গ্রুপের নাম `nobody`) | "কেউ না" — সবচেয়ে কম ক্ষমতার পরিচয় |

এই সীমাগুলো আসে `/etc/login.defs` ফাইল থেকে:

```bash
$ grep -E '^(UID|GID|SYS_UID|SYS_GID)_(MIN|MAX)' /etc/login.defs
UID_MIN                  1000
UID_MAX                 60000
GID_MIN                  1000
GID_MAX                 60000
```

(Debian/Ubuntu-তে `SYS_UID_MIN`/`SYS_UID_MAX` লাইনগুলো সাধারণত comment করা থাকে, ডিফল্ট মান ব্যবহার হয়।)

> ⚠️ **সতর্কতা:** UID 0 মানেই root, নাম যাই হোক। কেউ যদি `/etc/passwd`-এ `hacker:x:0:0:...` লাইন ঢুকিয়ে দেয়, সেই `hacker` পূর্ণ root ক্ষমতা পাবে। তাই `awk -F: '$3==0' /etc/passwd` দিয়ে মাঝেমধ্যে চেক করো — শুধু `root` আসা উচিত।

### Primary group আর Supplementary group

প্রতিটি ইউজারের ঠিক **একটি primary group** (প্রধান গ্রুপ) থাকে, যার GID `/etc/passwd`-এ লেখা থাকে। নতুন ফাইল বানালে ফাইলের গ্রুপ হয় এই primary group।

এছাড়া একজন ইউজার অনেকগুলো **supplementary group** (অতিরিক্ত গ্রুপ)-এর সদস্য হতে পারে, যেগুলো `/etc/group`-এ লেখা থাকে। যেমন `sudo` গ্রুপে থাকলে sudo চালানো যায়, `docker` গ্রুপে থাকলে docker চালানো যায়।

আধুনিক distro-গুলো **User Private Group (UPG)** পদ্ধতি মানে: প্রতিটি নতুন ইউজারের জন্য তার নিজের নামে একটা গ্রুপ তৈরি হয় (rahim ইউজারের primary group `rahim`)। এতে প্রত্যেকের ফাইল ডিফল্টভাবে নিজের গ্রুপে থাকে, অন্যদের সাথে অনিচ্ছায় শেয়ার হয় না।

---

## ১৫.৩ system user বনাম regular user

| বৈশিষ্ট্য | system user | regular user |
|---|---|---|
| উদাহরণ | `www-data`, `sshd`, `mysql`, `systemd-resolve` | `rahim`, `sumi` |
| UID | ১–৯৯৯ | ১০০০+ |
| কে বানায় | প্যাকেজ ইনস্টলের সময় (`useradd -r`), বা `systemd-sysusers` | অ্যাডমিন (`useradd`/`adduser`) বা ইনস্টলার |
| home directory | প্রায়ই নেই, বা `/var/lib/...`, `/nonexistent` | `/home/<নাম>` |
| shell | `/usr/sbin/nologin` বা `/bin/false` | `/bin/bash`, `/bin/zsh` ইত্যাদি |
| পাসওয়ার্ড | নেই (shadow-এ `*` বা `!`) | থাকে |
| password aging | প্রযোজ্য নয় | প্রযোজ্য |
| উদ্দেশ্য | service-কে আলাদা ও সীমিত পরিচয় দেওয়া | মানুষের লগইন ও কাজ |

> 📝 **মনে রাখো:** system user ও regular user-এর মধ্যে kernel-এর কাছে কোনো পার্থক্য নেই — দুটোই শুধু একটা নম্বর। পার্থক্যটা নিয়ম (convention) আর সেটিংসের: UID রেঞ্জ, shell, home। লগইন স্ক্রিন সাধারণত UID < 1000 ইউজারদের তালিকায় দেখায় না।

---

## ১৫.৪ ইউজার ডাটাবেস — চারটি গুরুত্বপূর্ণ ফাইল

ইউজার ও গ্রুপের তথ্য সাধারণ টেক্সট ফাইলে থাকে। এটাই Unix দর্শন: "সব কিছু টেক্সট, যাতে মানুষ পড়তে পারে আর টুল দিয়ে প্রসেস করা যায়।"

| ফাইল | কী থাকে | পারমিশন (সাধারণত) | কে পড়তে পারে |
|---|---|---|---|
| `/etc/passwd` | ইউজারের মৌলিক তথ্য | `-rw-r--r--` root:root | সবাই |
| `/etc/shadow` | পাসওয়ার্ড hash ও মেয়াদ | `-rw-r-----` root:shadow (Debian) / `----------` root:root (RHEL) | শুধু root (ও shadow গ্রুপ) |
| `/etc/group` | গ্রুপ ও সদস্য | `-rw-r--r--` | সবাই |
| `/etc/gshadow` | গ্রুপ পাসওয়ার্ড ও অ্যাডমিন | `/etc/shadow`-এর মতো | শুধু root |

কেন দুই ভাগ? `/etc/passwd` সবাইকে পড়তে দিতেই হয়, কারণ `ls` বা `ps`-এর মতো সাধারণ প্রোগ্রাম UID থেকে নাম বের করে। আগেকার Unix-এ পাসওয়ার্ডের hash-ও এখানে থাকত — ফলে যে কেউ hash কপি করে অফলাইনে crack করার চেষ্টা করতে পারত। তাই hash সরিয়ে নেওয়া হলো শুধু-root-পড়তে-পারে এমন `/etc/shadow`-এ। একে বলে **shadow password** পদ্ধতি।

```
   /etc/passwd  (সবাই পড়ে)          /etc/shadow (শুধু root)
   ┌────────────────────────┐       ┌───────────────────────────┐
   │ rahim:x:1000:1000:...  │──────▶│ rahim:$y$j9T$...:20769:...│
   └────────────────────────┘  "x"  └───────────────────────────┘
                              মানে "পাসওয়ার্ড shadow-এ আছে"
```

### `/etc/passwd` — ফিল্ড-বাই-ফিল্ড

```bash
$ grep rahim /etc/passwd
rahim:x:1000:1000:Rahim Uddin,,,:/home/rahim:/bin/bash
```

৭টি ফিল্ড, `:` (colon) দিয়ে আলাদা:

```
rahim : x : 1000 : 1000 : Rahim Uddin,,, : /home/rahim : /bin/bash
  │     │    │      │          │               │            │
  1     2    3      4          5               6            7
```

| # | ফিল্ড | উদাহরণ | ব্যাখ্যা |
|---|---|---|---|
| ১ | username | `rahim` | লগইন নাম। ছোট হাতের অক্ষর, সংখ্যা, `-`, `_`; সংখ্যা দিয়ে শুরু না করাই ভালো; সর্বোচ্চ ৩২ অক্ষর |
| ২ | password | `x` | `x` মানে আসল hash `/etc/shadow`-এ। খালি থাকলে পাসওয়ার্ড ছাড়াই লগইন — বিপজ্জনক! |
| ৩ | UID | `1000` | ইউজার নম্বর |
| ৪ | GID | `1000` | **primary group**-এর নম্বর |
| ৫ | GECOS | `Rahim Uddin,,,` | মন্তব্য: পুরো নাম, রুম নম্বর, অফিস ফোন, বাসার ফোন — কমা দিয়ে আলাদা। নামটা ঐতিহাসিক (GE-র একটা OS থেকে) |
| ৬ | home | `/home/rahim` | লগইনের পর যেখানে শুরু হবে, `~` এটাকেই বোঝায় |
| ৭ | shell | `/bin/bash` | লগইনের পর যে প্রোগ্রাম চলবে। `nologin` হলে লগইন বন্ধ |

কয়েকটা system user দেখো:

```bash
$ head -5 /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
```

লক্ষ্য করো `root`-এর home `/home/root` নয়, `/root`। কারণ `/home` আলাদা পার্টিশনে থাকতে পারে, আর সেটা mount না হলেও root যেন লগইন করে ঠিক করতে পারে।

### `/etc/shadow` — ফিল্ড-বাই-ফিল্ড

```bash
$ sudo grep rahim /etc/shadow
rahim:$y$j9T$Vh3k...দীর্ঘ_hash...$Qm1:20769:0:99999:7:::
```

৯টি ফিল্ড:

```
rahim:$y$j9T$...:20769:0:99999:7:::
  1      2         3   4   5   6 7 8 9
```

| # | ফিল্ড | উদাহরণ | ব্যাখ্যা |
|---|---|---|---|
| ১ | username | `rahim` | `/etc/passwd`-এর সাথে মেলে |
| ২ | password hash | `$y$j9T$...` | এনক্রিপ্ট নয়, **hash** (একমুখী)। বিশেষ মানগুলো নিচে |
| ৩ | last change | `20769` | শেষ পাসওয়ার্ড বদলের তারিখ, **১ জানুয়ারি ১৯৭০ থেকে কত দিন**। `0` মানে পরের লগইনে বদলাতেই হবে |
| ৪ | minimum | `0` | বদলানোর পর কমপক্ষে কত দিন আবার বদলানো যাবে না |
| ৫ | maximum | `99999` | কত দিন পর পাসওয়ার্ড মেয়াদোত্তীর্ণ হবে (`99999` ≈ কখনো না) |
| ৬ | warning | `7` | মেয়াদ শেষের কত দিন আগে সতর্ক করবে |
| ৭ | inactive | (খালি) | মেয়াদ শেষের পর কত দিন grace period, তারপর অ্যাকাউন্ট বন্ধ |
| ৮ | expire | (খালি) | পুরো **অ্যাকাউন্টের** মেয়াদ শেষের তারিখ (১৯৭০ থেকে দিন) |
| ৯ | reserved | (খালি) | ভবিষ্যতের জন্য সংরক্ষিত |

**Hash-এর শুরুর অংশ বলে দেয় কোন অ্যালগরিদম:**

| prefix | অ্যালগরিদম | কোথায় ডিফল্ট |
|---|---|---|
| `$y$` | yescrypt | Ubuntu 22.04+, Debian 11+, Fedora 35+ |
| `$6$` | SHA-512 crypt | RHEL 8/9, পুরোনো distro |
| `$5$` | SHA-256 crypt | কম ব্যবহৃত |
| `$1$` | MD5 crypt | পুরোনো, অনিরাপদ |

**বিশেষ মান:**

| hash ফিল্ডে | মানে |
|---|---|
| `*` | কখনো পাসওয়ার্ড সেট হয়নি, পাসওয়ার্ডে লগইন অসম্ভব (system user) |
| `!` বা `!!` | লক / পাসওয়ার্ড সেট নেই (`!!` সাধারণত Fedora/RHEL-এ নতুন ইউজারের) |
| `!$y$...` | hash-এর আগে `!` = অ্যাকাউন্ট **লক** করা (`passwd -l`), hash অক্ষত আছে |
| খালি | পাসওয়ার্ড ছাড়াই লগইন — কখনো রেখো না |

তারিখ সংখ্যাটা মানুষের ভাষায় দেখতে:

```bash
$ date -d "1970-01-01 +20769 days" +%F
2026-11-12
```

**hash কীভাবে কাজ করে?** পাসওয়ার্ড কখনো সংরক্ষিত হয় না। তুমি লগইনের সময় যা টাইপ করো, সিস্টেম সেটাকে একই **salt** (প্রতি ইউজারের র‍্যান্ডম অংশ, `$y$j9T$` এর পরের অংশ) সহ hash করে, তারপর সংরক্ষিত hash-এর সাথে মেলায়। মিললে ঢুকতে দেয়। salt থাকায় দুজনের একই পাসওয়ার্ড হলেও hash আলাদা হয়।

### `/etc/group` — ফিল্ড-বাই-ফিল্ড

```bash
$ grep -E '^(sudo|developers):' /etc/group
sudo:x:27:rahim
developers:x:1050:rahim,sumi,karim
```

৪টি ফিল্ড:

| # | ফিল্ড | উদাহরণ | ব্যাখ্যা |
|---|---|---|---|
| ১ | group name | `developers` | গ্রুপের নাম |
| ২ | password | `x` | `x` = gshadow-এ আছে (প্রায় কখনো ব্যবহৃত হয় না) |
| ৩ | GID | `1050` | গ্রুপ নম্বর |
| ৪ | members | `rahim,sumi,karim` | **supplementary** সদস্যদের তালিকা, কমা দিয়ে, স্পেস ছাড়া |

> ⚠️ **সতর্কতা:** যে গ্রুপ কারো **primary** group, সেই ইউজারের নাম সাধারণত চতুর্থ ফিল্ডে লেখা থাকে না। যেমন `rahim:x:1000:` — শেষ ফিল্ড খালি, তবুও rahim এই গ্রুপের সদস্য (কারণ `/etc/passwd`-এ তার GID 1000)। তাই কারা সদস্য জানতে শুধু `/etc/group` দেখলে ভুল হবে।

### `/etc/gshadow` — ফিল্ড-বাই-ফিল্ড

```bash
$ sudo grep developers /etc/gshadow
developers:!:rahim:rahim,sumi,karim
```

| # | ফিল্ড | ব্যাখ্যা |
|---|---|---|
| ১ | group name | গ্রুপের নাম |
| ২ | encrypted password | গ্রুপ পাসওয়ার্ড (`newgrp`-এর জন্য); `!` বা `*` = নেই |
| ৩ | administrators | গ্রুপ অ্যাডমিন — এরা `gpasswd` দিয়ে সদস্য যোগ/বাদ দিতে পারে, root না হয়েও |
| ৪ | members | সদস্য তালিকা (`/etc/group`-এর সাথে মিল রাখতে হয়) |

### নিরাপদে এই ফাইলগুলো এডিট করা

⚠️ কখনো `nano /etc/passwd` দিয়ে সরাসরি এডিট করো না। একটা colon ভুল হলে কেউ লগইন করতে পারবে না, এমনকি root-ও না। পরিবর্তে:

| কমান্ড | কাজ |
|---|---|
| `sudo vipw` | `/etc/passwd` লক করে এডিট করে, সেভের পর syntax চেক করে |
| `sudo vipw -s` | `/etc/shadow` এডিট |
| `sudo vigr` | `/etc/group` এডিট |
| `sudo vigr -s` | `/etc/gshadow` এডিট |
| `sudo pwck` | passwd/shadow-এর অসঙ্গতি খোঁজে (`-r` = শুধু পড়ে, ঠিক করে না) |
| `sudo grpck` | group/gshadow-এর অসঙ্গতি খোঁজে |

তবে ৯৯% ক্ষেত্রে নিচের `useradd`/`usermod`-জাতীয় কমান্ডই ব্যবহার করা উচিত।

### `getent` — সঠিক উপায়ে খোঁজা

বড় প্রতিষ্ঠানে ইউজার শুধু `/etc/passwd`-এ থাকে না; LDAP, Active Directory (SSSD দিয়ে) বা systemd-homed থেকেও আসতে পারে। `grep /etc/passwd` তখন তাদের খুঁজে পাবে না। `getent` সব উৎস একসাথে খোঁজে:

```bash
$ getent passwd rahim
rahim:x:1000:1000:Rahim Uddin,,,:/home/rahim:/bin/bash
$ getent group sudo
sudo:x:27:rahim
$ getent passwd 0
root:x:0:0:root:/root:/bin/bash
```

কোন উৎস কোন ক্রমে খোঁজা হবে তা ঠিক করে `/etc/nsswitch.conf`:

```bash
$ grep -E '^(passwd|group|shadow)' /etc/nsswitch.conf
passwd:         files systemd
group:          files systemd
shadow:         files systemd
```

(`files` = `/etc/passwd` ইত্যাদি; Fedora/RHEL-এ `sss` ও দেখা যায়।)

---

## ১৫.৫ ভেতরে কী ঘটে — login থেকে process পর্যন্ত

তুমি যখন লগইন করো, পেছনে এই ঘটনাগুলো ঘটে:

```
 ┌──────────┐   username/password    ┌────────────────────────┐
 │ তুমি     │ ─────────────────────▶ │ login / gdm / sshd     │
 └──────────┘                        └───────────┬────────────┘
                                                 │ "এই ইউজারকে যাচাই করো"
                                                 ▼
                                     ┌────────────────────────┐
                                     │ PAM (/etc/pam.d/*)      │
                                     │ pam_unix: shadow-এর hash│
                                     │ মেলায়, মেয়াদ চেক করে     │
                                     └───────────┬────────────┘
                                                 │ NSS (nsswitch.conf) দিয়ে
                                                 │ passwd/group পড়ে
                                                 ▼
                                     ┌────────────────────────┐
                                     │ setgroups() → setgid() │
                                     │ → setuid()  (kernel)   │
                                     │ process-এর credential   │
                                     │ এখন UID 1000            │
                                     └───────────┬────────────┘
                                                 │ chdir(home), exec(shell)
                                                 ▼
                                     ┌────────────────────────┐
                                     │ /bin/bash (UID 1000)   │
                                     │ তোমার সব child process │
                                     │ এই UID/GID উত্তরাধিকার  │
                                     │ সূত্রে পায়               │
                                     └────────────────────────┘
```

- **PAM (Pluggable Authentication Modules)** — প্রমাণীকরণের প্লাগইন-ব্যবস্থা। পাসওয়ার্ড, ফিঙ্গারপ্রিন্ট, 2FA — সব PAM মডিউল দিয়ে যুক্ত হয়।
- **NSS (Name Service Switch)** — নাম ↔ নম্বর রূপান্তরের ব্যবস্থা।
- **credential** — kernel প্রতিটি process-এর সাথে UID, GID, supplementary group-এর তালিকা রাখে। ফাইল খোলার সময় kernel এই নম্বরগুলো ফাইলের মালিকের নম্বরের সাথে মিলিয়ে পারমিশন দেয় ([অধ্যায় ১৬](16-permissions.md))।

এর একটা গুরুত্বপূর্ণ ফল: **গ্রুপের সদস্যপদ লগইনের সময় একবারই process-এ বসে।** তাই তোমাকে নতুন গ্রুপে যোগ করলে, পুরোনো শেলে সেটা কাজ করবে না — আবার লগইন (বা `newgrp`) করতে হবে। এটা নতুনদের সবচেয়ে সাধারণ বিভ্রান্তি।

process-এর credential সরাসরি দেখতে:

```bash
$ grep -E '^(Uid|Gid|Groups)' /proc/self/status
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
Groups:	4 24 27 1000
```

চারটি কলাম হলো real, effective, saved, filesystem UID — এদের পার্থক্য [অধ্যায় ১৭](17-special-permissions-acl.md)-এ setuid প্রসঙ্গে আসবে।

---

## ১৫.৬ আমি কে? — `whoami`, `id`, `groups`

### `whoami`

বর্তমান **effective** ইউজারের নাম দেখায়।

```bash
$ whoami
rahim
$ sudo whoami
root
```

### `id`

UID, GID ও সব গ্রুপ একসাথে দেখায়। সবচেয়ে কাজের কমান্ড।

**Syntax:** `id [OPTION]... [USER]`

```bash
$ id
uid=1000(rahim) gid=1000(rahim) groups=1000(rahim),4(adm),24(cdrom),27(sudo),46(plugdev)
$ id sumi
uid=1001(sumi) gid=1001(sumi) groups=1001(sumi),1050(developers)
```

| অপশন | কাজ |
|---|---|
| `-u` | শুধু UID |
| `-g` | শুধু primary GID |
| `-G` | সব গ্রুপের GID |
| `-n` | নম্বরের বদলে নাম (`-u`, `-g`, `-G`-এর সাথে) |
| `-r` | effective-এর বদলে real ID |

```bash
$ id -u
1000
$ id -un
rahim
$ id -Gn
rahim adm cdrom sudo plugdev
```

> 💡 **টিপস:** স্ক্রিপ্টে root কিনা চেক করতে: `if [ "$(id -u)" -ne 0 ]; then echo "root হয়ে চালাও"; exit 1; fi`

> 📝 **মনে রাখো:** `id` (ইউজার নাম ছাড়া) **বর্তমান process**-এর গ্রুপ দেখায়, আর `id rahim` **ডাটাবেস** থেকে দেখায়। নতুন গ্রুপে যোগ হওয়ার পর `id rahim`-এ গ্রুপটা দেখাবে কিন্তু শুধু `id`-তে দেখাবে না — যতক্ষণ না আবার লগইন করো।

### `groups`

```bash
$ groups
rahim adm cdrom sudo plugdev
$ groups sumi
sumi : sumi developers
```

---

## ১৫.৭ ইউজার তৈরি — `useradd` ও `adduser`

দুটো কমান্ড আছে, আর distro-ভেদে তাদের আচরণ আলাদা — এটা নতুনদের অনেক বিভ্রান্ত করে।

| | `useradd` | `adduser` |
|---|---|---|
| ধরন | low-level, shadow-utils-এর অংশ, সব distro-তে আছে | Debian/Ubuntu-তে একটা বন্ধুত্বপূর্ণ interactive Perl স্ক্রিপ্ট |
| প্রশ্ন করে? | না, সব অপশন দিয়ে বলতে হয় | হ্যাঁ, পাসওয়ার্ড, পুরো নাম জিজ্ঞেস করে |
| Fedora/RHEL-এ | একই | `adduser` শুধু `useradd`-এর symlink |
| স্ক্রিপ্টে | ✅ ব্যবহার করো (portable) | ❌ এড়িয়ে চলো |

### `adduser` (Debian/Ubuntu)

```bash
$ sudo adduser sumi
info: Adding user `sumi' ...
info: Selecting UID/GID from range 1000 to 59999 ...
info: Adding new group `sumi' (1001) ...
info: Adding new user `sumi' (1001) with group `sumi (1001)' ...
info: Creating home directory `/home/sumi' ...
info: Copying files from `/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
Changing the user information for sumi
Enter the new value, or press ENTER for the default
	Full Name []: Sumaiya Akter
	Room Number []:
	Work Phone []:
	Home Phone []:
	Other []:
Is the information correct? [Y/n] y
```

(পুরোনো ভার্সনে `info:` লেখা থাকে না, বাকিটা একই।) এক কমান্ডেই home, গ্রুপ, পাসওয়ার্ড — সব হয়ে গেল। ডেস্কটপ ইউজারের জন্য সবচেয়ে সহজ।

`adduser` দিয়ে কাউকে বিদ্যমান গ্রুপে যোগ করা যায়: `sudo adduser sumi developers`।

### `useradd`

**Syntax:** `useradd [OPTIONS] LOGIN`

| অপশন | লম্বা রূপ | কাজ |
|---|---|---|
| `-m` | `--create-home` | home directory বানাও (+ `/etc/skel` কপি) |
| `-M` | `--no-create-home` | home বানিও না |
| `-d DIR` | `--home-dir` | home-এর পাথ নির্দিষ্ট করো |
| `-s SHELL` | `--shell` | login shell |
| `-u UID` | `--uid` | নির্দিষ্ট UID |
| `-g GROUP` | `--gid` | primary group (নাম বা GID; গ্রুপটা আগে থেকে থাকতে হবে) |
| `-G G1,G2` | `--groups` | supplementary group-এর তালিকা |
| `-c "TEXT"` | `--comment` | GECOS / পুরো নাম |
| `-e YYYY-MM-DD` | `--expiredate` | অ্যাকাউন্টের মেয়াদ শেষের তারিখ |
| `-f DAYS` | `--inactive` | পাসওয়ার্ড মেয়াদ শেষের পর কত দিনে অ্যাকাউন্ট নিষ্ক্রিয় |
| `-r` | `--system` | system user (system UID রেঞ্জ, aging নেই, ডিফল্টে home নেই) |
| `-k DIR` | `--skel` | `/etc/skel`-এর বদলে অন্য skeleton (`-m`-এর সাথে) |
| `-N` | `--no-user-group` | নিজের নামে গ্রুপ বানিও না |
| `-U` | `--user-group` | নিজের নামে গ্রুপ বানাও |
| `-o` | `--non-unique` | ডুপ্লিকেট UID অনুমতি দাও (`-u`-এর সাথে; সাধারণত এড়াও) |
| `-D` | `--defaults` | ডিফল্ট মান দেখাও/বদলাও |

> ⚠️ **সতর্কতা (distro পার্থক্য):** Debian/Ubuntu-তে `useradd` শুধু নাম দিলে **home বানায় না** এবং shell ডিফল্ট হতে পারে `/bin/sh`। Fedora/RHEL-এ `/etc/login.defs`-এ `CREATE_HOME yes` থাকায় home বানায় আর shell `/bin/bash`। তাই portable হতে সবসময় `-m -s /bin/bash` স্পষ্ট করে লেখো।

**বাস্তব উদাহরণ — একজন ডেভেলপার যোগ করা:**

```bash
$ sudo useradd -m -s /bin/bash -c "Karim Hasan" -G developers,docker karim
$ sudo passwd karim
New password:
Retype new password:
passwd: password updated successfully
$ id karim
uid=1002(karim) gid=1002(karim) groups=1002(karim),1050(developers),998(docker)
$ ls -la /home/karim
total 20
drwxr-x--- 2 karim karim 4096 Nov 12 10:30 .
drwxr-xr-x 5 root  root  4096 Nov 12 10:30 ..
-rw-r--r-- 1 karim karim  220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 karim karim 3771 Mar 31  2024 .bashrc
-rw-r--r-- 1 karim karim  807 Mar 31  2024 .profile
```

(home-এর পারমিশন `HOME_MODE`/`UMASK` সেটিংসের উপর নির্ভর করে: Ubuntu 21.04+-এ `750`, অনেক distro-তে `700`।)

> 📝 **মনে রাখো:** `useradd` **পাসওয়ার্ড সেট করে না**। নতুন অ্যাকাউন্ট তখন লক অবস্থায় থাকে (`!` বা `!!`)। লগইন করাতে চাইলে পরে `passwd` চালাতে হবে।

**একটা service-এর জন্য system user:**

```bash
$ sudo useradd -r -s /usr/sbin/nologin -d /var/lib/myapp -M myapp
$ getent passwd myapp
myapp:x:998:998::/var/lib/myapp:/usr/sbin/nologin
```

**অস্থায়ী (কন্ট্রাক্টর) অ্যাকাউন্ট — ২০২৬-১২-৩১ এ মেয়াদ শেষ:**

```bash
$ sudo useradd -m -s /bin/bash -e 2026-12-31 tempdev
```

### useradd-এর ডিফল্ট কোথা থেকে আসে

`useradd` তিন জায়গা থেকে ডিফল্ট নেয়:

```
  /etc/login.defs       → UID/GID রেঞ্জ, CREATE_HOME, UMASK/HOME_MODE,
                          PASS_MAX_DAYS, ENCRYPT_METHOD, USERGROUPS_ENAB
  /etc/default/useradd  → SHELL, HOME (base dir), SKEL, INACTIVE, EXPIRE
  /etc/skel/            → নতুন home-এ কপি হওয়া ফাইল
```

```bash
$ useradd -D
GROUP=100
HOME=/home
INACTIVE=-1
EXPIRE=
SHELL=/bin/sh
SKEL=/etc/skel
CREATE_MAIL_SPOOL=no
LOG_INIT=yes
```

ডিফল্ট shell bash করতে: `sudo useradd -D -s /bin/bash` (এটা `/etc/default/useradd` বদলায়)।

---

## ১৫.৮ `/etc/skel` — নতুন ইউজারের "welcome kit"

`skel` = skeleton (কঙ্কাল)। `useradd -m` বা `adduser` নতুন home বানানোর সময় `/etc/skel`-এর সব কিছু (লুকানো ফাইলসহ) কপি করে দেয়, আর মালিকানা নতুন ইউজারকে দেয়।

**উপমা:** হোটেলে নতুন অতিথি এলে রুমে আগে থেকেই তোয়ালে, সাবান, পানির বোতল রাখা থাকে। `/etc/skel` হলো সেই স্ট্যান্ডার্ড সেটআপ।

```bash
$ ls -la /etc/skel
total 20
drwxr-xr-x   2 root root 4096 Apr 23  2024 .
drwxr-xr-x 130 root root 4096 Nov 12 10:30 ..
-rw-r--r--   1 root root  220 Mar 31  2024 .bash_logout
-rw-r--r--   1 root root 3771 Mar 31  2024 .bashrc
-rw-r--r--   1 root root  807 Mar 31  2024 .profile
```

কোম্পানির সব নতুন কর্মীর জন্য একটা নিয়মাবলি ফাইল আর alias রাখতে চাও:

```bash
$ sudo tee /etc/skel/README-company.txt > /dev/null <<'EOF'
স্বাগতম! প্রজেক্ট ফাইল রাখো ~/projects-এ।
EOF
$ sudo mkdir /etc/skel/projects
$ echo "alias ll='ls -alF'" | sudo tee -a /etc/skel/.bashrc
```

এখন থেকে নতুন প্রতিটি ইউজার এগুলো পাবে।

> ⚠️ **সতর্কতা:** `/etc/skel` বদলালে **পুরোনো** ইউজারদের home-এ কোনো প্রভাব পড়ে না — কপি হয় শুধু তৈরির মুহূর্তে। আর এখানে কখনো গোপন তথ্য (key, পাসওয়ার্ড) রেখো না, কারণ সবাই কপি পাবে।

`.bashrc`, `.profile` আসলে কী করে, তা বিস্তারিত দেখবে [অধ্যায় ৪৪](44-environment-bashrc.md)-এ।

---

## ১৫.৯ পাসওয়ার্ড — `passwd` ও `chpasswd`

### `passwd`

নিজের বা (root হলে) অন্যের পাসওয়ার্ড বদলায়, আর অ্যাকাউন্ট লক/আনলক করে।

**Syntax:** `passwd [OPTIONS] [USER]`

```bash
$ passwd
Changing password for rahim.
Current password:
New password:
Retype new password:
passwd: password updated successfully
```

টাইপ করার সময় কিছুই দেখা যায় না (কোনো `*`-ও না) — এটা স্বাভাবিক, পাশ থেকে কেউ যেন দৈর্ঘ্যও বুঝতে না পারে।

| অপশন | কাজ |
|---|---|
| (নাম ছাড়া) | নিজের পাসওয়ার্ড বদলাও (পুরোনোটা লাগবে) |
| `USER` (root হিসেবে) | অন্যের পাসওয়ার্ড সেট করো (পুরোনোটা লাগে না) |
| `-l` | অ্যাকাউন্ট লক (hash-এর আগে `!` বসায়) |
| `-u` | আনলক |
| `-d` | পাসওয়ার্ড মুছে দাও (খালি পাসওয়ার্ড — বিপজ্জনক) |
| `-e` | পাসওয়ার্ড এখনই মেয়াদোত্তীর্ণ — পরের লগইনে বদলাতে বাধ্য |
| `-S` | পাসওয়ার্ডের অবস্থা দেখাও |
| `-n/-x/-w/-i DAYS` | min/max/warn/inactive দিন (chage-এর মতো) |
| `--stdin` | stdin থেকে পাসওয়ার্ড পড়ো — **শুধু Fedora/RHEL**, Debian/Ubuntu-তে নেই |

```bash
$ sudo passwd -S sumi
sumi P 2026-11-12 0 99999 7 -1
```

স্ট্যাটাস কোড: `P` = usable password, `L` = locked, `NP` = no password। পরের সংখ্যাগুলো: শেষ বদল, min, max, warn, inactive।

**নতুন কর্মীকে প্রথম লগইনে পাসওয়ার্ড বদলাতে বাধ্য করো:**

```bash
$ sudo passwd -e sumi
passwd: password changed.
```

পরের লগইনে sumi দেখবে: `You are required to change your password immediately (administrator enforced).`

> ⚠️ **সতর্কতা:** `passwd -l` শুধু **পাসওয়ার্ড** লগইন বন্ধ করে। SSH key দিয়ে ইউজার তখনও ঢুকতে পারে! পুরো অ্যাকাউন্ট বন্ধ করতে লক করার সাথে সাথে `sudo usermod -e 1 USER` (মেয়াদ ১৯৭০-এ শেষ) অথবা shell `nologin` করো।

### `chpasswd` — অনেক পাসওয়ার্ড একসাথে

`user:password` ফর্ম্যাটে stdin থেকে পড়ে। সব distro-তে কাজ করে, স্ক্রিপ্টের জন্য আদর্শ।

```bash
$ echo 'sumi:N3w-Str0ng#Pass' | sudo chpasswd
$ sudo chpasswd < users.txt      # প্রতি লাইনে user:password
```

> 💡 **টিপস:** কমান্ড লাইনে সরাসরি পাসওয়ার্ড লিখলে সেটা shell history-তে থেকে যায় ([অধ্যায় ৬](06-terminal-shortcuts.md))। bash-এ কমান্ডের আগে একটা স্পেস দিলে (যদি `HISTCONTROL`-এ `ignorespace` থাকে, Ubuntu-তে ডিফল্ট `ignoreboth`) সেটা history-তে যায় না।

---

## ১৫.১০ পাসওয়ার্ডের মেয়াদ — `chage`

`chage` = **ch**ange **age**। `/etc/shadow`-এর ফিল্ড ৩–৮ সহজে বদলায়।

**Syntax:** `chage [OPTIONS] USER`

| অপশন | shadow ফিল্ড | কাজ |
|---|---|---|
| `-l` | — | বর্তমান সেটিংস দেখাও (সাধারণ ইউজারও নিজেরটা দেখতে পারে) |
| `-d DATE` | ৩ | শেষ বদলের তারিখ সেট করো; `-d 0` = পরের লগইনে বদলাতে বাধ্য |
| `-m DAYS` | ৪ | বদলের মাঝে ন্যূনতম দিন |
| `-M DAYS` | ৫ | সর্বোচ্চ দিন (এরপর মেয়াদ শেষ); `-1` = সীমা তুলে দাও |
| `-W DAYS` | ৬ | কত দিন আগে সতর্ক করবে |
| `-I DAYS` | ৭ | মেয়াদ শেষের পর কত দিন grace, তারপর লক; `-1` = বন্ধ |
| `-E DATE` | ৮ | অ্যাকাউন্টের মেয়াদ (YYYY-MM-DD); `-1` = কখনো না |

```bash
$ sudo chage -l sumi
Last password change					: Nov 12, 2026
Password expires					: never
Password inactive					: never
Account expires						: never
Minimum number of days between password change		: 0
Maximum number of days between password change		: 99999
Number of days of warning before password expires	: 7
```

**কোম্পানির নীতি:** প্রতি ৯০ দিনে পাসওয়ার্ড বদলাতে হবে, ১ দিনের আগে আবার বদলানো যাবে না, ১৪ দিন আগে সতর্কতা, মেয়াদ পেরোনোর ৭ দিন পর লক:

```bash
$ sudo chage -M 90 -m 1 -W 14 -I 7 sumi
$ sudo chage -l sumi
Last password change					: Nov 12, 2026
Password expires					: Feb 10, 2027
Password inactive					: Feb 17, 2027
Account expires						: never
Minimum number of days between password change		: 1
Maximum number of days between password change		: 90
Number of days of warning before password expires	: 14
```

সময়রেখায় বোঝো:

```
 শেষ বদল         min(1)               max(90)      +inactive(7)
   │──────────────│─────────────────────│──────────────│──────▶ সময়
   Nov 12        Nov 13      ⚠ warn    Feb 10         Feb 17
               এর আগে বদল    (Jan 27   পাসওয়ার্ড       অ্যাকাউন্ট
               করা যাবে না    থেকে)     মেয়াদোত্তীর্ণ;   নিষ্ক্রিয়
                                      লগইনে বদলাতে
                                      বাধ্য
```

নতুন সব ইউজারের জন্য ডিফল্ট বদলাতে `/etc/login.defs`-এ `PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_WARN_AGE` বদলাও (শুধু ভবিষ্যতের ইউজারের উপর প্রভাব ফেলে)।

> 💡 **টিপস:** আধুনিক নিরাপত্তা নির্দেশিকা (যেমন NIST SP 800-63B) বলে, ঘন ঘন জোর করে পাসওয়ার্ড বদলানোর চেয়ে লম্বা, অনন্য পাসওয়ার্ড + 2FA বেশি কার্যকর। তবে অনেক প্রতিষ্ঠানের compliance নিয়মে aging লাগে, তাই `chage` জানা জরুরি।

---

## ১৫.১১ ইউজার পরিবর্তন — `usermod`

**Syntax:** `usermod [OPTIONS] LOGIN`

| অপশন | কাজ |
|---|---|
| `-aG G1,G2` | supplementary group-এ **যোগ** করো (append) — সবচেয়ে বেশি ব্যবহৃত |
| `-G G1,G2` | supplementary group-এর তালিকা **প্রতিস্থাপন** করো (বাকিগুলো থেকে বাদ!) |
| `-g GROUP` | primary group বদলাও |
| `-l NEWNAME` | লগইন নাম বদলাও (home বা গ্রুপের নাম বদলায় না) |
| `-d DIR -m` | home পাথ বদলাও এবং পুরোনো ফাইল নতুন জায়গায় সরাও |
| `-s SHELL` | login shell বদলাও |
| `-c "TEXT"` | GECOS / পুরো নাম |
| `-u UID` | UID বদলাও (home-এর ফাইলের মালিকানাও আপডেট হয়) |
| `-L` | লক করো (`passwd -l`-এর মতো) |
| `-U` | আনলক |
| `-e DATE` | অ্যাকাউন্টের মেয়াদ; `-e ""` = মেয়াদ তুলে দাও |
| `-rG G` | নির্দিষ্ট গ্রুপ থেকে বাদ দাও (নতুন shadow-utils, ৪.১৩+; Ubuntu 24.04-এ আছে, পুরোনোতে `gpasswd -d` ব্যবহার করো) |

### সবচেয়ে বিখ্যাত ভুল: `-a` ভুলে যাওয়া

```bash
$ id rahim
uid=1000(rahim) gid=1000(rahim) groups=1000(rahim),4(adm),27(sudo)
$ sudo usermod -G docker rahim          # ❌ -a নেই!
$ id rahim
uid=1000(rahim) gid=1000(rahim) groups=1000(rahim),998(docker)
```

rahim এখন `sudo` গ্রুপ থেকে বাদ পড়ে গেছে! যদি এটাই একমাত্র অ্যাডমিন হয়, আর root লক থাকে (Ubuntu-তে ডিফল্ট), তাহলে তুমি নিজের সিস্টেম থেকে অ্যাডমিন ক্ষমতা হারালে। ঠিক করতে recovery mode লাগবে।

সঠিক উপায়:

```bash
$ sudo usermod -aG docker rahim         # ✅ append
$ id rahim
uid=1000(rahim) gid=1000(rahim) groups=1000(rahim),4(adm),27(sudo),998(docker)
```

> ⚠️ **সতর্কতা:** **`-G` কখনো `-a` ছাড়া ব্যবহার কোরো না**, যদি না ইচ্ছা করেই পুরো তালিকা বদলাতে চাও। মনে রাখার কৌশল: "**a**G = **a**dd to **G**roup"।

যোগ করার পর rahim-কে **লগআউট করে আবার লগইন** করতে হবে (ভেতরে কেন — ১৫.৫ দেখো)। তাৎক্ষণিক পরীক্ষা করতে `newgrp docker` (১৫.১৩)।

### অন্যান্য উদাহরণ

```bash
# লগইন নাম বদল + home ও গ্রুপের নামও মিলিয়ে বদল
$ sudo usermod -l rahim2 rahim
$ sudo usermod -d /home/rahim2 -m rahim2
$ sudo groupmod -n rahim2 rahim

# shell zsh করা (zsh ইনস্টল থাকতে হবে)
$ sudo usermod -s /usr/bin/zsh sumi

# অ্যাকাউন্ট আজ থেকে বন্ধ (লক + মেয়াদ শেষ)
$ sudo usermod -L -e 1 karim
```

> ⚠️ **সতর্কতা:** যে ইউজার লগইন অবস্থায় আছে বা যার process চলছে, তার নাম/UID/home বদলানো যায় না: `usermod: user rahim is currently used by process 1234`। অন্য অ্যাডমিন অ্যাকাউন্ট থেকে করো।

---

## ১৫.১২ ইউজার মুছে ফেলা — `userdel`

**Syntax:** `userdel [OPTIONS] LOGIN`

| অপশন | কাজ |
|---|---|
| (কিছু না) | শুধু অ্যাকাউন্ট মোছে; home ও ফাইল থেকে যায় |
| `-r` | home directory ও mail spool-ও মোছে |
| `-f` | লগইন থাকলেও জোর করে মোছে (বিপজ্জনক) |

```bash
$ sudo userdel -r tempdev
userdel: tempdev mail spool (/var/mail/tempdev) not found
```

(mail spool না থাকলে এই বার্তা আসে — সমস্যা নয়।)

Debian/Ubuntu-তে বন্ধুত্বপূর্ণ বিকল্প `deluser`:

```bash
$ sudo deluser --remove-home tempdev
$ sudo deluser --backup --remove-home tempdev   # মোছার আগে home-এর ব্যাকআপ
$ sudo deluser sumi developers                   # শুধু গ্রুপ থেকে বাদ
```

> ⚠️ **সতর্কতা:** `-r` শুধু home আর mail spool মোছে। `/tmp`, `/var/www` বা অন্য কোথাও থাকা ঐ ইউজারের ফাইল থেকে যায় orphan হয়ে। পরে নতুন ইউজার একই UID পেলে সে ঐ ফাইলগুলোর মালিক হয়ে যাবে! মোছার আগে: `sudo find / -xdev -user tempdev 2>/dev/null`। আর প্রতিষ্ঠানে প্রায়ই মোছার বদলে প্রথমে **লক** করা ভালো — অডিটের জন্য।

---

## ১৫.১৩ গ্রুপ ম্যানেজমেন্ট

**উপমা:** গ্রুপ হলো অফিসের বিভাগ। "Accounts" বিভাগের সবাই একাউন্টস রুমে ঢুকতে পারে। নতুন কর্মী আসলে তাকে বিভাগে যোগ করো — প্রতিটি ফাইলের পারমিশন আলাদা করে দিতে হয় না।

### `groupadd`

| অপশন | কাজ |
|---|---|
| `-g GID` | নির্দিষ্ট GID |
| `-r` | system group (system GID রেঞ্জ) |
| `-f` | গ্রুপ আগে থেকে থাকলে error না দিয়ে সফলভাবে বেরিয়ে যাও |

```bash
$ sudo groupadd developers
$ sudo groupadd -g 1050 designers
$ getent group developers
developers:x:1051:
```

### `groupmod`

| অপশন | কাজ |
|---|---|
| `-n NEWNAME` | নাম বদলাও |
| `-g GID` | GID বদলাও (⚠️ বিদ্যমান ফাইলের GID বদলায় না — সেগুলো orphan হবে) |

```bash
$ sudo groupmod -n devteam developers
```

### `groupdel`

```bash
$ sudo groupdel designers
$ sudo groupdel sumi
groupdel: cannot remove the primary group of user 'sumi'
```

কোনো ইউজারের **primary group** মোছা যায় না — আগে সেই ইউজারকে মুছতে হবে বা তার primary group বদলাতে হবে।

### `gpasswd` — গ্রুপের সদস্য সামলানো

| অপশন | কাজ |
|---|---|
| `-a USER GROUP` | সদস্য যোগ |
| `-d USER GROUP` | সদস্য বাদ |
| `-M u1,u2 GROUP` | পুরো সদস্য তালিকা সেট (প্রতিস্থাপন) |
| `-A u1,u2 GROUP` | গ্রুপ অ্যাডমিন সেট (gshadow-এর ৩ নম্বর ফিল্ড) |
| `-r GROUP` | গ্রুপ পাসওয়ার্ড মোছো |
| `GROUP` (শুধু) | গ্রুপ পাসওয়ার্ড সেট |

```bash
$ sudo gpasswd -a sumi devteam
Adding user sumi to group devteam
$ sudo gpasswd -d sumi devteam
Removing user sumi from group devteam
$ sudo gpasswd -A rahim devteam      # rahim এখন sudo ছাড়াই সদস্য যোগ করতে পারবে
$ gpasswd -a karim devteam           # rahim হিসেবে, sudo ছাড়া
Adding user karim to group devteam
```

> 💡 **টিপস:** কাউকে গ্রুপ থেকে **বাদ** দেওয়ার সবচেয়ে নিরাপদ ও portable উপায় `gpasswd -d USER GROUP` — বাকি গ্রুপে হাত দেয় না।

### `newgrp` ও `sg` — লগআউট ছাড়াই নতুন গ্রুপ

```bash
$ sudo usermod -aG docker rahim
$ id -Gn
rahim adm sudo                      # docker এখনো নেই
$ newgrp docker                     # নতুন subshell, primary group = docker
$ id -Gn
docker adm sudo rahim
$ exit                              # আগের শেলে ফিরে যাও
```

`newgrp` একটা নতুন shell চালু করে, যার primary group হয় নির্দিষ্ট গ্রুপটি। শুধু একটা কমান্ড চালাতে চাইলে `sg`:

```bash
$ sg docker -c "docker ps"
```

### গুরুত্বপূর্ণ প্রচলিত গ্রুপ

| গ্রুপ | Debian/Ubuntu | Fedora/RHEL | সদস্য হলে যা পারে |
|---|---|---|---|
| অ্যাডমিন | `sudo` | `wheel` | `sudo` দিয়ে root ক্ষমতা ([অধ্যায় ১৮](18-sudo-root.md)) |
| `adm` | ✅ | ✅ | `/var/log`-এর অনেক লগ পড়া |
| `systemd-journal` | ✅ | ✅ | সব journal লগ পড়া ([অধ্যায় ২৮](28-logs.md)) |
| `docker` | প্যাকেজ বানায় | প্যাকেজ বানায় | docker চালানো (কার্যত root-এর সমান!) |
| `dialout` | ✅ | ✅ | serial port (`/dev/ttyUSB0`, Arduino) |
| `video`, `audio`, `render` | ✅ | ✅ | ক্যামেরা/GPU/সাউন্ড ডিভাইস |
| `plugdev` | ✅ | — | কিছু removable ডিভাইস |
| `lpadmin` | ✅ | — | প্রিন্টার পরিচালনা |
| `libvirt` | ✅ | ✅ | ভার্চুয়াল মেশিন পরিচালনা ([অধ্যায় ৪৮](48-containers-virtualization.md)) |

> ⚠️ **সতর্কতা:** `docker` বা `libvirt` গ্রুপে কাউকে যোগ করা মানে কার্যত তাকে root দেওয়া, কারণ সে container-এ host-এর `/` mount করে যা খুশি করতে পারে। বিশ্বস্ত মানুষ ছাড়া দিও না।

---

## ১৫.১৪ কে লগইন আছে, কে এসেছিল — `who`, `w`, `users`, `last`, `lastlog`

### `who`

এই মুহূর্তে লগইন করা ইউজারদের তালিকা (তথ্য আসে `/var/run/utmp` থেকে; নতুন কিছু distro-তে systemd-logind থেকে)।

```bash
$ who
rahim    seat0        2026-11-12 09:01 (login screen)
rahim    tty2         2026-11-12 09:01 (tty2)
karim    pts/1        2026-11-12 10:12 (192.168.1.25)
```

| অপশন | কাজ |
|---|---|
| `-b` | শেষ boot-এর সময় |
| `-r` | বর্তমান runlevel |
| `-q` | শুধু নাম ও মোট সংখ্যা |
| `-H` | কলামের শিরোনাম দেখাও |
| `-a` | সব তথ্য |
| `am i` / `-m` | শুধু বর্তমান terminal-এর ইউজার |

```bash
$ who -b
         system boot  2026-11-10 07:11
```

`tty` = সরাসরি কনসোল (Ctrl+Alt+F3 ইত্যাদি), `pts/N` = pseudo-terminal (GUI terminal window বা SSH)।

### `w`

`who`-এর চেয়ে বেশি: প্রত্যেকে **কী করছে** তাও দেখায়।

```bash
$ w
 10:15:32 up 2 days,  3:04,  2 users,  load average: 0.15, 0.10, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU  WHAT
rahim    tty2     -                09:01    1:14m  2:10   0.05s /usr/libexec/gnome-session-binary
karim    pts/1    192.168.1.25     10:12    0.00s  0.04s  0.00s vim deploy.sh
```

| কলাম | মানে |
|---|---|
| প্রথম লাইন | বর্তমান সময়, কতক্ষণ চালু (`uptime`-এর মতো), ইউজার সংখ্যা, load average ([অধ্যায় ৪৬](46-performance-troubleshooting.md)) |
| `FROM` | কোথা থেকে (SSH হলে IP) |
| `LOGIN@` | কখন লগইন করেছে |
| `IDLE` | কতক্ষণ কিছু টাইপ করেনি |
| `JCPU` | ঐ terminal-এর সব process মিলে CPU সময় |
| `PCPU` | বর্তমান process-এর CPU সময় |
| `WHAT` | এখন কী চালাচ্ছে |

`w -h` = শিরোনাম ছাড়া, `w USER` = শুধু ঐ ইউজার। `users` কমান্ড শুধু নামগুলো এক লাইনে দেখায়।

### `last`

অতীতের লগইন/লগআউট ও reboot-এর ইতিহাস (`/var/log/wtmp` থেকে)।

```bash
$ last -n 5
karim    pts/1        192.168.1.25     Thu Nov 12 10:12   still logged in
rahim    tty2         tty2             Thu Nov 12 09:01   still logged in
reboot   system boot  6.8.0-48-generic Tue Nov 10 07:11   still running
sumi     pts/0        10.0.0.7         Mon Nov  9 18:20 - 19:45  (01:25)
rahim    tty2         tty2             Mon Nov  9 08:55 - down   (10:12)

wtmp begins Sun Nov  1 00:00:02 2026
```

| অপশন | কাজ |
|---|---|
| `-n N` / `-N` | শেষ N টি এন্ট্রি |
| `USER` | শুধু ঐ ইউজার |
| `reboot` | শুধু reboot-এর ইতিহাস |
| `-x` | shutdown ও runlevel পরিবর্তনও |
| `-F` | পূর্ণ তারিখ-সময় |
| `-i` | hostname-এর বদলে IP |
| `-s` / `-t` | নির্দিষ্ট সময়ের পর/আগে (যেমন `-s yesterday`) |

**ব্যর্থ লগইনের চেষ্টা** দেখতে `sudo lastb` (`/var/log/btmp` থেকে) — SSH brute-force আক্রমণ ধরতে খুব কাজের:

```bash
$ sudo lastb -n 3
admin    ssh:notty    45.83.12.9       Thu Nov 12 03:14 - 03:14  (00:00)
root     ssh:notty    45.83.12.9       Thu Nov 12 03:14 - 03:14  (00:00)
test     ssh:notty    103.21.4.77      Thu Nov 12 02:58 - 02:58  (00:00)
```

### `lastlog`

প্রতিটি ইউজারের **সর্বশেষ** লগইন একনজরে:

```bash
$ lastlog
Username         Port     From             Latest
root                                       **Never logged in**
daemon                                     **Never logged in**
rahim            tty2                      Thu Nov 12 09:01:44 +0600 2026
karim            pts/1    192.168.1.25     Thu Nov 12 10:12:03 +0600 2026
```

`lastlog -u karim` = শুধু karim, `lastlog -b 90` = যারা ৯০ দিনের বেশি লগইন করেনি (অব্যবহৃত অ্যাকাউন্ট খুঁজতে দারুণ)।

> 📝 **মনে রাখো (distro পার্থক্য):** পুরোনো utmp/wtmp ফরম্যাট ২০৩৮ সালের সময়-সীমা (Y2038) সমস্যায় ভোগে। তাই কিছু নতুন distro (যেমন Debian 13, Ubuntu 25.04+, openSUSE) `wtmpdb` ও `lastlog2` ব্যবহার শুরু করেছে; সেখানে `last` কমান্ড `wtmpdb last` দিয়ে চলে আর `lastlog`-এর জায়গায় `lastlog2` থাকতে পারে। RHEL 9-এ `lastlog` আছে, ভবিষ্যতে বাদ পড়ার কথা। systemd-ভিত্তিক বিকল্প: `loginctl list-sessions` আর `journalctl -u systemd-logind`।

---

## ১৫.১৫ ইউজার বদলানো — `su`

`su` = **s**ubstitute **u**ser (বা switch user)। অন্য ইউজার হিসেবে নতুন shell খোলে — **লক্ষ্য ইউজারের পাসওয়ার্ড** লাগে (root থেকে চালালে লাগে না)।

**Syntax:** `su [OPTIONS] [-] [USER [ARG...]]`

| অপশন | কাজ |
|---|---|
| `su` | root হও, কিন্তু বর্তমান environment রাখো (non-login shell) |
| `su -` / `su -l` / `su --login` | root হও, **login shell** হিসেবে — পরিষ্কার environment |
| `su - USER` | ঐ ইউজারের login shell |
| `-c "CMD"` | শুধু একটা কমান্ড চালিয়ে ফিরে এসো |
| `-s SHELL` | নির্দিষ্ট shell ব্যবহার করো |
| `-p` / `-m` | environment সংরক্ষণ করো |

```bash
$ su - sumi
Password:
sumi@laptop:~$ pwd
/home/sumi
sumi@laptop:~$ exit
logout
$ su -c "id" sumi
Password:
uid=1001(sumi) gid=1001(sumi) groups=1001(sumi),1051(devteam)
```

### `su` বনাম `su -` — পার্থক্যটা গুরুত্বপূর্ণ

```
  su root  (dash ছাড়া)                su - root  (dash সহ)
  ┌─────────────────────────┐          ┌─────────────────────────┐
  │ UID = 0                 │          │ UID = 0                 │
  │ PWD = যেখানে ছিলে        │          │ PWD = /root             │
  │ PATH = rahim-এর PATH     │          │ PATH = root-এর PATH     │
  │ HOME = (distro-ভেদে)     │          │ HOME = /root            │
  │ .profile পড়ে না           │          │ /etc/profile, ~/.profile│
  │                         │          │ পড়ে (login shell)        │
  └─────────────────────────┘          └─────────────────────────┘
       ↑ মিশ্র environment                ↑ সত্যিকারের root সেশন
```

dash ছাড়া `su` করলে পুরোনো ইউজারের `PATH` থেকে যেতে পারে — ফলে `/usr/sbin`-এর কমান্ড না-ও পাওয়া যেতে পারে, বা আরও খারাপ, কেউ নিজের `PATH`-এ ফাঁদ পেতে রাখলে root সেই ফাঁদের প্রোগ্রাম চালাতে পারে। **সবসময় `su -` ব্যবহার করো।**

> ⚠️ **সতর্কতা (Ubuntu):** Ubuntu-তে ডিফল্টভাবে root-এর কোনো পাসওয়ার্ড সেট নেই (লক), তাই `su -` কাজ করে না: `su: Authentication failure`। Ubuntu-তে `sudo -i` ব্যবহার করো। Fedora ইনস্টলারে root পাসওয়ার্ড সেট করার অপশন থাকে, RHEL-এও। `sudo` বনাম `su` এর পূর্ণ তুলনা [অধ্যায় ১৮](18-sudo-root.md)-এ।

> 💡 **টিপস:** `nologin` shell-ওয়ালা service user হিসেবে কমান্ড চালাতে: `sudo -u www-data whoami` বা `sudo su -s /bin/bash www-data`। শুধু `su www-data` চালালে nologin বার্তা দিয়ে বেরিয়ে যাবে।

---

## ১৫.১৬ login shell, non-login shell ও `/etc/shells`

### login shell কী?

যে shell তুমি **লগইন করার সাথে সাথে** পাও — tty-তে লগইন, SSH, `su -`, `bash -l` — সেটাই **login shell**। আর GUI-তে terminal window খুললে বা লগইন অবস্থায় `bash` টাইপ করলে যেটা পাও, সেটা সাধারণত **non-login interactive shell**।

পার্থক্য হলো কোন কনফিগ ফাইল পড়া হয়:

```
 login shell (bash)                   non-login interactive (bash)
 ─────────────────                    ─────────────────────────────
 /etc/profile                          /etc/bash.bashrc (Debian) বা
   └─ /etc/profile.d/*.sh                /etc/bashrc (Fedora, ~/.bashrc থেকে)
 তারপর প্রথম যেটা পায়:                   ~/.bashrc
   ~/.bash_profile
   ~/.bash_login
   ~/.profile   (Debian/Ubuntu-তে এটা
                 নিজেই ~/.bashrc ডাকে)
```

চেনার উপায়:

```bash
$ echo $0
-bash            # শুরুতে '-' মানে login shell
$ shopt -q login_shell && echo login || echo non-login
login
```

এই ফাইলগুলোর বিস্তারিত ও কাস্টমাইজেশন [অধ্যায় ৪৪](44-environment-bashrc.md)-এ।

### `/etc/shells` ও `chsh`

`/etc/shells` হলো বৈধ login shell-এর তালিকা। `chsh` দিয়ে সাধারণ ইউজার শুধু এই তালিকার shell বেছে নিতে পারে।

```bash
$ cat /etc/shells
# /etc/shells: valid login shells
/bin/sh
/bin/bash
/usr/bin/bash
/bin/rbash
/usr/bin/rbash
/bin/dash
/usr/bin/dash
$ chsh -s /usr/bin/zsh          # নিজের shell zsh করো
Password:
$ getent passwd rahim | cut -d: -f7
/usr/bin/zsh
```

(`zsh` ইনস্টল করলে সেটা `/etc/shells`-এ যোগ হয়।) পরিবর্তন কার্যকর হবে পরের লগইনে। Fedora/RHEL-এ `chsh` আলাদা প্যাকেজ (`util-linux-user`) থেকে আসে; না থাকলে `sudo usermod -s` ব্যবহার করো।

GECOS (পুরো নাম ইত্যাদি) বদলাতে `chfn`: `chfn -f "Rahim Uddin"`।

---

## ১৫.১৭ `nologin` — লগইন বন্ধ কিন্তু অ্যাকাউন্ট সক্রিয়

service user-দের অ্যাকাউন্ট দরকার (ফাইলের মালিক হতে, process চালাতে), কিন্তু কেউ যেন ঐ অ্যাকাউন্ট দিয়ে লগইন করে shell না পায়। সমাধান: shell হিসেবে `/usr/sbin/nologin` দাও।

```bash
$ grep www-data /etc/passwd
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
$ sudo su - www-data
This account is currently not available.
```

`nologin` একটা ছোট প্রোগ্রাম যেটা ঐ বার্তা দেখিয়ে non-zero exit code দিয়ে বেরিয়ে যায়। বার্তা কাস্টমাইজ করতে `/etc/nologin.txt` ফাইল বানাও।

| | `/usr/sbin/nologin` | `/bin/false` | `/etc/nologin` **ফাইল** |
|---|---|---|---|
| কী | shell হিসেবে সেট করা প্রোগ্রাম | কিছু না করে exit 1 | একটা ফাইল, থাকলেই প্রভাব |
| প্রভাব | শুধু ঐ ইউজার | শুধু ঐ ইউজার | **root ছাড়া সবার** লগইন বন্ধ |
| বার্তা | ভদ্র বার্তা দেখায় ও লগ করে | চুপচাপ বেরিয়ে যায় | ফাইলের লেখা দেখায় |
| ব্যবহার | service account (পছন্দনীয়) | পুরোনো রীতি | মেইনটেন্যান্সের সময় |

```bash
$ echo "সার্ভার মেইনটেন্যান্সে আছে, রাত ১২টায় ফিরব।" | sudo tee /etc/nologin
$ sudo rm /etc/nologin       # কাজ শেষে অবশ্যই মুছো!
```

(systemd সিস্টেমে বুটের সময় `/run/nologin` ফাইলও তৈরি হয় যাতে বুট পুরো না হওয়া পর্যন্ত লগইন না হয়।)

> ⚠️ **সতর্কতা:** `nologin` shell থাকলেও SSH-এর কিছু ফিচার (যেমন port forwarding `ssh -N`) কাজ করতে পারে, কারণ সেগুলো shell চালায় না। শক্তভাবে আটকাতে `sshd_config`-এ `AllowUsers`/`DenyUsers` ব্যবহার করো ([অধ্যায় ৩৮](38-ssh-remote.md))।

---

## ১৫.১৮ সংশ্লিষ্ট কীবোর্ড শর্টকাট

| শর্টকাট / কমান্ড | কাজ |
|---|---|
| `Ctrl + D` | বর্তমান shell থেকে বেরোও (logout / `su` থেকে ফেরা) — খালি লাইনে |
| `exit` / `logout` | shell থেকে বেরোও (`logout` শুধু login shell-এ কাজ করে) |
| `Ctrl + Alt + F3` … `F6` | টেক্সট কনসোল (tty)-এ যাও, আলাদা ইউজারে লগইন করো |
| `Ctrl + Alt + F1` বা `F2` | GUI-তে ফিরে যাও (distro-ভেদে; Ubuntu/Fedora-তে সাধারণত F1 বা F2) |
| `Super + L` | স্ক্রিন লক (GNOME) |
| GNOME: Power মেনু → Switch User | লগআউট না করে অন্য ইউজার হিসেবে ঢোকো |
| পাসওয়ার্ড প্রম্পটে `Ctrl + U` | ভুল টাইপ করা পুরো পাসওয়ার্ড মুছে আবার লেখো |
| পাসওয়ার্ড প্রম্পটে `Ctrl + C` | বাতিল করো |
| `!!` (যেমন `sudo !!`) | আগের কমান্ড sudo দিয়ে আবার চালাও (permission denied পেলে) |

---

## হাতে-কলমে অনুশীলন

> ⚠️ এগুলো একটা VM বা টেস্ট মেশিনে করো ([অধ্যায় ৩](03-distro-o-installation.md))। কাজ শেষে ল্যাব ৭-এ সব পরিষ্কার করা আছে।

**ল্যাব ১: নিজেকে চেনো**

```bash
$ whoami
$ id
$ groups
$ getent passwd "$USER"
$ grep -E '^(Uid|Gid|Groups)' /proc/self/status
```

প্রশ্ন: তোমার UID কত? primary group কোনটি? তুমি কি `sudo` বা `wheel` গ্রুপে আছো?

**ল্যাব ২: সিস্টেমের ইউজার গোনো**

```bash
$ awk -F: '$3 >= 1000 && $3 < 60000 {print $1, $3, $7}' /etc/passwd   # মানুষ ইউজার
$ awk -F: '$3 < 1000 {print $1}' /etc/passwd | wc -l                   # system user সংখ্যা
$ awk -F: '$3 == 0 {print $1}' /etc/passwd                             # UID 0 কারা (শুধু root থাকা উচিত)
$ grep -c nologin /etc/passwd
```

**ল্যাব ৩: একটা টিম সেটআপ**

```bash
$ sudo groupadd labteam
$ sudo useradd -m -s /bin/bash -c "Lab User One" -G labteam labuser1
$ sudo useradd -m -s /bin/bash -c "Lab User Two" labuser2
$ echo 'labuser1:Lab#Pass2026' | sudo chpasswd
$ echo 'labuser2:Lab#Pass2026' | sudo chpasswd
$ sudo usermod -aG labteam labuser2
$ getent group labteam
labteam:x:1052:labuser1,labuser2
$ id labuser2
```

**ল্যাব ৪: `-a` ভুলের প্রভাব নিজের চোখে দেখো**

```bash
$ sudo groupadd labextra
$ sudo usermod -G labextra labuser2      # -a নেই
$ id labuser2                            # labteam উধাও!
$ sudo usermod -aG labteam labuser2      # ঠিক করো
$ id labuser2
```

**ল্যাব ৫: পাসওয়ার্ড নীতি ও লক**

```bash
$ sudo chage -M 60 -W 10 -I 5 labuser1
$ sudo chage -l labuser1
$ sudo passwd -e labuser1                # প্রথম লগইনে বদলাতে বাধ্য
$ su - labuser1                          # পাসওয়ার্ড বদলাতে বলবে, বদলে exit করো
$ sudo passwd -l labuser2
$ sudo passwd -S labuser2                # L দেখাবে
$ sudo grep labuser2 /etc/shadow         # hash-এর আগে '!' দেখো
$ sudo passwd -u labuser2
```

**ল্যাব ৬: skel ও nologin**

```bash
$ echo "Welcome to the lab!" | sudo tee /etc/skel/WELCOME.txt
$ sudo useradd -m -s /bin/bash labuser3
$ sudo ls -la /home/labuser3              # WELCOME.txt আছে?
$ sudo usermod -s /usr/sbin/nologin labuser3
$ sudo su - labuser3                      # "This account is currently not available."
$ sudo -u labuser3 whoami                 # তবুও এভাবে কমান্ড চালানো যায়
$ who; w; last -n 5
```

**ল্যাব ৭: পরিষ্কার করো**

```bash
$ sudo userdel -r labuser1
$ sudo userdel -r labuser2
$ sudo userdel -r labuser3
$ sudo groupdel labteam
$ sudo groupdel labextra
$ sudo rm /etc/skel/WELCOME.txt
$ sudo pwck -r && sudo grpck -r           # সব ঠিক আছে কিনা
```

---

## সারসংক্ষেপ

| কমান্ড / ফাইল | কাজ |
|---|---|
| `/etc/passwd` | `name:x:UID:GID:GECOS:home:shell` — সবাই পড়ে |
| `/etc/shadow` | `name:hash:last:min:max:warn:inactive:expire:` — শুধু root |
| `/etc/group` | `name:x:GID:members` |
| `/etc/gshadow` | `name:password:admins:members` |
| `/etc/login.defs`, `/etc/default/useradd` | নতুন ইউজারের ডিফল্ট |
| `/etc/skel/` | নতুন home-এ কপি হওয়া ফাইল |
| `getent passwd USER` | সব উৎস থেকে ইউজার খোঁজো |
| `id`, `id -un`, `groups` | পরিচয় ও গ্রুপ দেখো |
| `useradd -m -s /bin/bash -G g1 USER` | ইউজার বানাও (portable) |
| `adduser USER` | interactive ইউজার তৈরি (Debian/Ubuntu) |
| `useradd -r -s /usr/sbin/nologin USER` | system user |
| `passwd [USER]` | পাসওয়ার্ড বদলাও |
| `passwd -l / -u / -e / -S` | লক / আনলক / মেয়াদোত্তীর্ণ / অবস্থা |
| `chpasswd` | stdin থেকে `user:pass` |
| `chage -l / -M / -m / -W / -I / -E / -d 0` | পাসওয়ার্ড ও অ্যাকাউন্টের মেয়াদ |
| `usermod -aG GROUP USER` | গ্রুপে **যোগ** (কখনো `-a` বাদ দিও না) |
| `usermod -l / -d -m / -s / -L / -U / -e` | নাম / home / shell / লক / আনলক / মেয়াদ |
| `userdel -r USER` | ইউজার ও home মোছো |
| `groupadd / groupmod -n / groupdel` | গ্রুপ তৈরি / নাম বদল / মোছা |
| `gpasswd -a / -d / -A` | সদস্য যোগ / বাদ / অ্যাডমিন |
| `newgrp G`, `sg G -c CMD` | লগআউট ছাড়া নতুন গ্রুপ |
| `who`, `w`, `users` | এখন কে লগইন আছে |
| `last`, `lastb`, `lastlog` | লগইন ইতিহাস, ব্যর্থ চেষ্টা, শেষ লগইন |
| `su - USER`, `su -c CMD USER` | ইউজার বদলাও (login shell) |
| `chsh -s SHELL`, `/etc/shells` | login shell বদলাও |
| `vipw`, `vigr`, `pwck`, `grpck` | নিরাপদে এডিট ও যাচাই |
| `/usr/sbin/nologin`, `/etc/nologin` | ইউজারের / সবার লগইন বন্ধ |

| শর্টকাট | কাজ |
|---|---|
| `Ctrl + D` | shell / su থেকে বেরোও |
| `Ctrl + Alt + F3–F6` | টেক্সট কনসোল |
| `Super + L` | স্ক্রিন লক |
| পাসওয়ার্ড প্রম্পটে `Ctrl + U` | টাইপ করা মুছে ফেলো |

---

## নিজেকে যাচাই করো

1. kernel ফাইলের মালিক হিসেবে কী সংরক্ষণ করে — নাম না নম্বর? `ls -l` তাহলে নাম দেখায় কীভাবে?
2. `/etc/passwd` সবাই পড়তে পারে, তাহলে পাসওয়ার্ড নিরাপদ থাকে কীভাবে?
3. `/etc/shadow`-এ hash-এর জায়গায় `!$y$...` দেখলে কী বুঝবে? আর `*` দেখলে?
4. `sudo usermod -G docker rahim` আর `sudo usermod -aG docker rahim`-এর পার্থক্য কী?
5. তোমাকে `docker` গ্রুপে যোগ করা হয়েছে, কিন্তু `id`-তে দেখাচ্ছে না। কেন, আর সমাধান কী?
6. Debian-এ `sudo useradd sumi` চালানোর পর sumi লগইন করতে পারছে না এবং তার home নেই। কেন?
7. নতুন কর্মী যেন প্রথম লগইনেই পাসওয়ার্ড বদলায় — দুটো উপায় বলো।
8. `su` আর `su -`-এর মূল পার্থক্য কী? কোনটা ব্যবহার করা উচিত?
9. `/usr/sbin/nologin` shell আর `/etc/nologin` ফাইলের পার্থক্য কী?
10. কোনো ইউজারের primary group কেন `groupdel` দিয়ে মোছা যায় না?

<details><summary>উত্তর দেখো</summary>

1. **নম্বর** (UID ও GID), inode-এ। `ls -l` NSS (`/etc/nsswitch.conf` অনুযায়ী `/etc/passwd`, `/etc/group` ইত্যাদি) থেকে নম্বরকে নামে রূপান্তর করে দেখায়। `ls -n` আসল নম্বর দেখায়।
2. `/etc/passwd`-এ পাসওয়ার্ডের জায়গায় শুধু `x` থাকে। আসল hash থাকে `/etc/shadow`-এ, যা শুধু root (Debian-এ shadow গ্রুপও) পড়তে পারে। আর সেখানেও পাসওয়ার্ড নয়, salt-সহ একমুখী hash থাকে।
3. `!` প্রিফিক্স মানে অ্যাকাউন্ট **লক** (`passwd -l` বা `usermod -L`), hash অক্ষত, `passwd -u` দিয়ে খোলা যাবে। `*` মানে কখনো পাসওয়ার্ড সেট হয়নি — পাসওয়ার্ড দিয়ে লগইন অসম্ভব (সাধারণত system user)।
4. `-G` একা supplementary group-এর **পুরো তালিকা প্রতিস্থাপন** করে — rahim অন্য সব গ্রুপ (যেমন sudo) থেকে বাদ পড়বে। `-aG` শুধু **যোগ** করে, বাকিগুলো অক্ষত থাকে।
5. গ্রুপের তালিকা লগইনের সময় process-এর credential-এ বসে; পুরোনো shell পুরোনো তালিকা নিয়েই চলছে। লগআউট করে আবার লগইন করো, অথবা সাময়িকভাবে `newgrp docker` / `sg docker -c "..."`। `id rahim` (নাম দিয়ে) ডাটাবেস দেখে, তাই সেখানে দেখাবে।
6. Debian/Ubuntu-তে `useradd` ডিফল্টভাবে home বানায় না (`-m` লাগে) এবং পাসওয়ার্ড সেট করে না (অ্যাকাউন্ট লক থাকে)। সমাধান: `useradd -m -s /bin/bash sumi` তারপর `passwd sumi`, অথবা `adduser sumi`।
7. `sudo passwd -e USER` অথবা `sudo chage -d 0 USER`।
8. `su` (dash ছাড়া) বর্তমান environment (PATH, PWD ইত্যাদি) রেখে দেয়, non-login shell। `su -` পূর্ণ login shell দেয় — লক্ষ্য ইউজারের home, PATH ও profile ফাইল। নিরাপত্তা ও সঠিকতার জন্য `su -` ব্যবহার করা উচিত (Ubuntu-তে root-এর জন্য `sudo -i`)।
9. `/usr/sbin/nologin` হলো নির্দিষ্ট একজন ইউজারের login shell — শুধু তার লগইন বন্ধ হয়। `/etc/nologin` ফাইল থাকলে **root ছাড়া সবার** লগইন বন্ধ হয়ে যায়, ফাইলের লেখা বার্তা হিসেবে দেখায় — মেইনটেন্যান্সের সময় কাজে লাগে।
10. কারণ `/etc/passwd`-এ ঐ ইউজারের GID ফিল্ড সেই গ্রুপকে নির্দেশ করে — মুছলে ইউজারের primary group অস্তিত্বহীন হয়ে যাবে। আগে ইউজার মুছতে হবে (`userdel`) বা `usermod -g` দিয়ে অন্য primary group দিতে হবে।

</details>

---

[← আগের অধ্যায়](14-vim.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](16-permissions.md)

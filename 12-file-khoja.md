# অধ্যায় ১২: ফাইল ও কমান্ড খোঁজা

> "কম্পিউটারে ফাইল হারায় না, শুধু আমরা ভুলে যাই কোথায় রেখেছি।" এই অধ্যায় শেষ করলে কোনো ফাইল বা কমান্ড খুঁজে পেতে আর বেশি সময় লাগবে না।

## এই অধ্যায়ে যা শিখবে

- লিনাক্সে "খোঁজা" কেন দুই রকম: **লাইভ সার্চ** (`find`, `fd`) আর **ডেটাবেস সার্চ** (`locate`/`plocate`)
- `find` কমান্ড শুরু থেকে শেষ পর্যন্ত: `-name`, `-iname`, `-type`, `-size`, `-mtime`, `-mmin`, `-user`, `-perm`, `-maxdepth`, `-empty` ইত্যাদি
- `find`-এর ভেতরের কাজ: directory tree কীভাবে ঘোরে আর expression কীভাবে বাম থেকে ডানে মূল্যায়ন হয়
- লজিক্যাল অপারেটর (`-a`, `-o`, `!`, `\( \)`) আর `-prune`
- খুঁজে পাওয়া ফাইলের ওপর কাজ চালানো: `-exec {} \;`, `-exec {} +`, `-execdir`, `-ok`, `-delete`
- নিরাপদ পাইপলাইন: `-print0` আর `xargs -0`
- `locate`/`plocate` আর `updatedb`: কেন এত দ্রুত, আর কখন ভুল ফল দেয়
- কমান্ড খোঁজা: `which`, `whereis`, `type`, `command -v`, `hash`, `apropos`
- আধুনিক বিকল্প `fd`
- রোজকার কাজে লাগে এমন অনেকগুলো রেসিপি

---

## ১২.১ খোঁজার দুই পদ্ধতি: লাইভ বনাম ডেটাবেস

একটা লাইব্রেরিতে বই খোঁজার কথা ভাবো। দুইভাবে খোঁজা যায়:

1. **প্রতিটা তাক ঘুরে দেখা**: সময় লাগে, কিন্তু যা পাবে তা এই মুহূর্তে তাকে আছে। এটা `find`-এর পদ্ধতি।
2. **ক্যাটালগ খাতা দেখা**: মুহূর্তে উত্তর পাবে, কিন্তু খাতাটা গত রাতে লেখা। আজ সকালে যে বই এসেছে সেটা খাতায় নেই, আর যে বই আজ কেউ নিয়ে গেছে সেটা খাতায় এখনও আছে। এটা `locate`-এর পদ্ধতি।

```
            ফাইল খোঁজার দুই রাস্তা
 ┌──────────────────────────┐      ┌────────────────────────────┐
 │  লাইভ সার্চ               │      │  ডেটাবেস সার্চ               │
 │  find / fd               │      │  locate / plocate           │
 ├──────────────────────────┤      ├────────────────────────────┤
 │ ডিস্কের প্রতিটা ডিরেক্টরি  │      │ আগে বানানো একটা index ফাইল │
 │ এখনই পড়ে দেখে            │      │ (plocate.db) পড়ে           │
 │ ধীর (বড় ডিস্কে সেকেন্ড-   │      │ খুব দ্রুত (মিলিসেকেন্ড)     │
 │ মিনিট)                    │      │                             │
 │ সবসময় হালনাগাদ             │      │ শেষ updatedb পর্যন্ত হালনাগাদ│
 │ নাম, সাইজ, সময়, মালিক,    │      │ শুধু নাম/পাথ দিয়ে খোঁজে    │
 │ পারমিশন সব দিয়ে খোঁজা যায় │      │                             │
 └──────────────────────────┘      └────────────────────────────┘
```

কমান্ড খোঁজা (যেমন `ls` প্রোগ্রামটা ডিস্কে কোথায় আছে) আলাদা জিনিস, সেটা শেষের দিকে `which`, `type` ইত্যাদি দিয়ে দেখব।

> 📝 **মনে রাখো:** ফাইলের *নাম* দিয়ে খুঁজতে চাইলে `find` বা `locate`। ফাইলের *ভেতরের লেখা* দিয়ে খুঁজতে চাইলে `grep -r`, যেটা [অধ্যায় ২০](20-grep-regex.md)-এ আছে।

---

## ১২.২ `find` পরিচিতি

`find` লিনাক্সের সবচেয়ে শক্তিশালী খোঁজার টুল। এটা GNU findutils প্যাকেজের অংশ, তাই সব বড় distro-তে (Ubuntu, Debian, Fedora, RHEL) আগে থেকেই ইনস্টল থাকে।

### সিনট্যাক্স

```
find [অপশন] [কোথা থেকে শুরু...] [expression]
```

- **কোথা থেকে শুরু (starting point)**: এক বা একাধিক ডিরেক্টরি। না দিলে বর্তমান ডিরেক্টরি (`.`) ধরে নেয়।
- **expression**: কী খুঁজবে আর পেলে কী করবে। এর ভেতরে তিন রকম জিনিস থাকে:
  - **test** (পরীক্ষা): `-name`, `-type`, `-size` ইত্যাদি। প্রতিটা ফাইলের জন্য true বা false দেয়।
  - **action** (কাজ): `-print`, `-delete`, `-exec` ইত্যাদি।
  - **option**: `-maxdepth`, `-mindepth`, `-xdev` ইত্যাদি, যা পুরো সার্চের আচরণ বদলায়।
  - **operator**: `-a`, `-o`, `!`, `\( \)`, যা test আর action জোড়া লাগায়।

সবচেয়ে সহজ ব্যবহার:

```
$ find
.
./notes.txt
./project
./project/main.c
./project/README.md
```

কোনো test বা action না দিলে `find` সব ফাইল দেখায়, কারণ default action হলো `-print`।

### ভেতরে কীভাবে কাজ করে (under the hood)

`find` আসলে একটা **depth-first tree walker**। প্রতিটা ডিরেক্টরিতে ঢুকে kernel-কে system call দিয়ে জিজ্ঞেস করে "এখানে কী কী আছে?" (`getdents64`), তারপর দরকার হলে প্রতিটা ফাইলের metadata চায় (`lstat`/`statx`): সাইজ, সময়, মালিক, পারমিশন। এই metadata আসে ফাইলের inode থেকে ([অধ্যায় ১১](11-links-inode.md))।

```
  find /home/rahim -name "*.txt"

  /home/rahim                         ← শুরু
  ├── docs/        ──► ভেতরে ঢোকো (depth-first)
  │   ├── a.txt    ──► test: -name "*.txt" → TRUE  → -print
  │   └── b.pdf    ──► test → FALSE → বাদ
  ├── music/       ──► ভেতরে ঢোকো
  │   └── song.mp3 ──► FALSE
  └── todo.txt     ──► TRUE → -print

  প্রতিটা ফাইলের জন্য expression বাম → ডান মূল্যায়ন হয়।
  কোনো test FALSE হলে বাকি অংশ আর চেক হয় না (short-circuit)।
```

**short-circuit** মানে: `find . -name "*.log" -size +10M` লিখলে আগে নাম মেলানো হয়। নাম না মিললে সাইজ আর দেখা হয় না। নাম মেলানোর জন্য শুধু ডিরেক্টরি এন্ট্রি পড়লেই চলে, কিন্তু সাইজ জানতে `stat` লাগে। তাই সস্তা test আগে রাখলে `find` দ্রুত চলে।

> 💡 **টিপস:** বড় ডিরেক্টরিতে `find` চালানোর সময় `-name`/`-type` আগে আর `-size`/`-mtime`/`-exec` পরে রাখো।

### "Permission denied" নিয়ন্ত্রণ

সাধারণ ইউজার হিসেবে `/` থেকে খুঁজলে অনেক ডিরেক্টরিতে ঢোকার অনুমতি থাকে না:

```
$ find / -name "sshd_config"
find: '/root': Permission denied
find: '/proc/1/task/1/fd': Permission denied
...
/etc/ssh/sshd_config
```

এরর মেসেজগুলো stderr-এ যায়, তাই ফেলে দেওয়া যায় ([অধ্যায় ১৯](19-redirection-pipe.md)):

```
$ find / -name "sshd_config" 2>/dev/null
/etc/ssh/sshd_config
```

অথবা `sudo find ...` দিয়ে সব জায়গায় খুঁজতে পারো।

---

## ১২.৩ নাম দিয়ে খোঁজা: `-name`, `-iname`, `-path`, `-regex`

### `-name` আর `-iname`

```
$ find ~/Documents -name "report.pdf"
/home/rahim/Documents/2024/report.pdf

$ find ~/Documents -name "*.pdf"
/home/rahim/Documents/2024/report.pdf
/home/rahim/Documents/cv.pdf

$ find ~/Documents -iname "*.PDF"      # বড়-ছোট হাতের অক্ষর উপেক্ষা
/home/rahim/Documents/2024/report.pdf
/home/rahim/Documents/cv.pdf
/home/rahim/Documents/SCAN.PDF
```

`-name` shell-এর মতো wildcard (glob) বোঝে:

| প্যাটার্ন | মানে | উদাহরণ |
|---|---|---|
| `*` | যেকোনো সংখ্যক অক্ষর (শূন্যও) | `"*.txt"` |
| `?` | ঠিক একটা অক্ষর | `"file?.log"` → `file1.log` |
| `[abc]` | এর যেকোনো একটা | `"[Rr]eadme*"` |
| `[0-9]` | রেঞ্জ | `"img[0-9][0-9].jpg"` |
| `[!a]` | a ছাড়া যেকোনো | `"[!.]*"` |

> ⚠️ **সতর্কতা: প্যাটার্ন সবসময় কোটেশনে রাখো।** `find . -name *.txt` লিখলে shell আগেই `*.txt`-কে বর্তমান ডিরেক্টরির ফাইলের নাম দিয়ে বদলে দেয় (globbing)। বর্তমান ডিরেক্টরিতে `a.txt` আর `b.txt` থাকলে কমান্ডটা হয়ে যায় `find . -name a.txt b.txt`, ফলে `find: paths must precede expression` এরর আসে, বা ভুল ফল আসে। সঠিক: `find . -name "*.txt"`।

📝 `-name` শুধু ফাইলের **শেষ অংশ** (basename) মেলায়, পুরো পাথ না। `-name "docs/*.txt"` কখনো মিলবে না।

### `-path` আর `-ipath`: পুরো পাথ মেলানো

```
$ find . -path "*/docs/*.txt"
./project/docs/intro.txt
./old/docs/notes.txt
```

এখানে `*` ঘোরাফেরা করে `/`-সহ পুরো পাথে। (`-wholename` হলো `-path`-এর পুরোনো নাম।)

### `-regex` আর `-iregex`

রেগুলার এক্সপ্রেশন দিয়ে **পুরো পাথ** মেলায় ([অধ্যায় ২০](20-grep-regex.md))। GNU find-এর default regex টাইপ emacs-style; extended regex চাইলে `-regextype posix-extended` দাও:

```
$ find . -regextype posix-extended -regex '.*/img_[0-9]{4}\.(jpg|png)'
./photos/img_2024.jpg
./photos/img_0001.png
```

> 💡 regex পুরো পাথের সাথে মেলে, তাই শুরুতে `.*/` দিতে ভুলো না।

---

## ১২.৪ ফাইলের ধরন দিয়ে: `-type`

লিনাক্সে "সব কিছুই ফাইল", কিন্তু ফাইলের ধরন আলাদা আলাদা।

| অক্ষর | ধরন | উদাহরণ |
|---|---|---|
| `f` | সাধারণ ফাইল (regular file) | `/etc/hostname` |
| `d` | ডিরেক্টরি | `/home` |
| `l` | সিম্বলিক লিংক | `/bin` (অনেক distro-তে `/usr/bin`-এর লিংক) |
| `b` | block device | `/dev/sda` |
| `c` | character device | `/dev/null` |
| `p` | named pipe (FIFO) | |
| `s` | socket | `/run/systemd/notify` |

```
$ find ~/project -type d
/home/rahim/project
/home/rahim/project/src
/home/rahim/project/docs

$ find /etc -maxdepth 1 -type l
/etc/localtime
/etc/os-release
/etc/mtab
```

GNU find-এ একসাথে একাধিক টাইপ কমা দিয়ে দেওয়া যায়: `-type f,l`।

**ভাঙা লিংক** খোঁজা: `-xtype l` লিংক অনুসরণ করে দেখে, গন্তব্যও যদি লিংকের মতো দেখায় (মানে গন্তব্য নেই) তবে true।

```
$ find ~ -xtype l
/home/rahim/.config/old-link
```

---

## ১২.৫ সাইজ দিয়ে: `-size` আর `-empty`

```
find . -size [+|-]n[একক]
```

| একক | মানে |
|---|---|
| `c` | বাইট |
| `k` | কিবিবাইট (1024 বাইট) |
| `M` | মেবিবাইট (1024 k) |
| `G` | গিবিবাইট (1024 M) |
| `b` | 512-বাইটের ব্লক (কোনো একক না দিলে এটাই default) |
| `w` | 2-বাইটের word |

| লেখা | মানে |
|---|---|
| `-size +100M` | 100 MiB-এর **বেশি** |
| `-size -10k` | 10 KiB-এর **কম** |
| `-size 4k` | ঠিক 4 KiB (রাউন্ড করার পরে) |

```
$ find ~ -type f -size +500M 2>/dev/null
/home/rahim/Downloads/ubuntu-24.04-desktop-amd64.iso
/home/rahim/Videos/wedding.mp4

$ find /var/log -type f -size +10M -size -100M     # 10M থেকে 100M-এর মাঝে
```

> ⚠️ **সতর্কতা: রাউন্ডিং-এর ফাঁদ।** `find` সাইজকে আগে একক অনুযায়ী **উপরের দিকে রাউন্ড** করে তারপর তুলনা করে। তাই `-size -1M` মানে "রাউন্ড করার পর 1 MiB-এর কম", অর্থাৎ শুধু 0 বাইটের ফাইল! 1 বাইটের ফাইলও রাউন্ড হয়ে 1M হয়ে যায়। 1 MiB-এর ছোট ফাইল চাইলে বাইটে লেখো: `-size -1048576c`, অথবা `-size -1024k`।

### `-empty`

খালি ফাইল (0 বাইট) বা খালি ডিরেক্টরি খোঁজে:

```
$ find ~/project -empty
/home/rahim/project/build
/home/rahim/project/src/placeholder.txt

$ find ~/project -type d -empty      # শুধু খালি ডিরেক্টরি
/home/rahim/project/build
```

---

## ১২.৬ সময় দিয়ে: `-mtime`, `-mmin` ও আত্মীয়রা

প্রতিটা inode-এ তিনটা প্রধান সময় থাকে:

| সময় | মানে | কখন বদলায় | find test |
|---|---|---|---|
| **mtime** (modify) | কনটেন্ট শেষ কবে বদলেছে | ফাইলে লিখলে | `-mtime`, `-mmin` |
| **atime** (access) | শেষ কবে পড়া হয়েছে | পড়লে (তবে নিচের নোট দেখো) | `-atime`, `-amin` |
| **ctime** (change) | inode/metadata শেষ কবে বদলেছে | chmod, chown, rename, লেখা | `-ctime`, `-cmin` |

> 📝 **মনে রাখো:** ctime মানে "creation time" **না**। লিনাক্সের অনেক ফাইলসিস্টেম (ext4, btrfs, xfs) birth time রাখে, যা `stat` কমান্ডে `Birth:` হিসেবে দেখা যায়, আর GNU find-এ (ফাইলসিস্টেম সমর্থন করলে) `-newerBt` দিয়ে তুলনা করা যায়। কিন্তু `-ctime` সবসময় change time।
>
> আধুনিক distro-তে ফাইলসিস্টেম সাধারণত `relatime` অপশনে মাউন্ট হয়, তাই প্রতিবার পড়লেই atime বদলায় না। তাই `-atime`-এর ওপর পুরোপুরি ভরসা কোরো না।

### সংখ্যার মানে: `+n`, `-n`, `n`

`-mtime` দিন হিসেবে কাজ করে (১ দিন = ২৪ ঘণ্টা), `-mmin` মিনিট হিসেবে।

```
 এখন ◄────────────────────────────────────────── অতীত
     │  0 দিন  │  1 দিন  │  2 দিন  │  3 দিন  │ ...
     │ (0-24ঘ) │(24-48ঘ) │(48-72ঘ) │
     
 -mtime 0   → শেষ 24 ঘণ্টার মধ্যে বদলেছে
 -mtime 1   → 24 থেকে 48 ঘণ্টা আগে
 -mtime -3  → 3 দিনের কম আগে (0, 1, 2 নম্বর ঘর)
 -mtime +3  → 3 নম্বর ঘরের পরে, মানে কমপক্ষে 4×24 = 96 ঘণ্টা আগে
```

> ⚠️ **সতর্কতা:** `find` বয়সকে ২৪ ঘণ্টা দিয়ে ভাগ করে **ভগ্নাংশ ফেলে দেয়**। তাই `-mtime +1` মানে "২ দিন বা তার বেশি পুরোনো", "১ দিনের বেশি" না। ৩০ ঘণ্টা পুরোনো ফাইল `-mtime +1`-এ মিলবে না। নিখুঁত হিসাব চাইলে মিনিট ব্যবহার করো: `-mmin +1440`।

```
$ find ~ -type f -mmin -30           # শেষ ৩০ মিনিটে বদলানো ফাইল
/home/rahim/.bash_history
/home/rahim/project/main.c

$ find /tmp -type f -mtime +7        # ৭ দিনের বেশি (কমপক্ষে ৮ দিন) পুরোনো
/tmp/old-report.csv

$ find /var/log -name "*.log" -mtime -1    # আজকের (শেষ ২৪ ঘণ্টা) লগ
```

### অন্য ফাইল বা তারিখের সাথে তুলনা: `-newer`, `-newermt`

```
$ touch -d "2026-11-01 00:00" /tmp/mark
$ find ~/project -newer /tmp/mark          # mark-এর পরে বদলানো
```

GNU find-এ সরাসরি তারিখ দেওয়া যায়:

```
$ find ~/project -type f -newermt "2026-11-01"
$ find ~/project -type f -newermt "2026-11-01" ! -newermt "2026-11-08"   # ১ থেকে ৭ নভেম্বর
$ find . -newermt "2 hours ago"
```

`-newerXY`-এ X আর Y হতে পারে `a` (atime), `m` (mtime), `c` (ctime), `B` (birth), আর Y-তে `t` মানে "লেখা তারিখ"।

`-daystart` অপশন দিলে সময় হিসাব হয় আজকের দিনের শুরু (রাত ১২টা) থেকে, ২৪ ঘণ্টা আগে থেকে না:

```
$ find . -daystart -mtime 0      # আজ (ক্যালেন্ডার দিন) বদলানো
```

---

## ১২.৭ মালিক ও গ্রুপ দিয়ে: `-user`, `-group`, `-nouser`

```
$ find /home -user rahim -name "*.sh"
$ find /var/www -group www-data -type f
$ find / -uid 1001 2>/dev/null       # UID দিয়ে
$ sudo find / -nouser -o -nogroup 2>/dev/null   # যে ফাইলের মালিক আর নেই
```

`-nouser` খুব কাজের: কোনো ইউজার ডিলিট করার পর তার ফাইলগুলো "অনাথ" হয়ে যায় (মালিকের জায়গায় শুধু সংখ্যা দেখায়)। ইউজার ও গ্রুপ নিয়ে বিস্তারিত [অধ্যায় ১৫](15-users-groups.md)-এ।

---

## ১২.৮ পারমিশন দিয়ে: `-perm`

পারমিশন কী, তা [অধ্যায় ১৬](16-permissions.md) আর [অধ্যায় ১৭](17-special-permissions-acl.md)-এ আছে। এখানে শুধু খোঁজার নিয়ম। `-perm`-এর তিনটা রূপ আছে, আর এটাই অনেকে গুলিয়ে ফেলে:

| রূপ | মানে | উপমা |
|---|---|---|
| `-perm 644` | পারমিশন **হুবহু** 644 | চাবির খাঁজ হুবহু মিলতে হবে |
| `-perm -644` | 644-এর **সবগুলো** বিট সেট আছে (বেশিও থাকতে পারে) | "অন্তত এগুলো তো থাকতেই হবে" |
| `-perm /022` | উল্লেখ করা বিটের **যেকোনো একটা** সেট আছে | "এর কোনো একটা থাকলেই হলো" |

```
$ find ~ -type f -perm 777            # হুবহু rwxrwxrwx
$ find ~ -type f -perm -o=w           # others লিখতে পারে (symbolic রূপ)
$ find . -type f -perm /u=x,g=x,o=x   # কারো না কারো execute পারমিশন আছে
$ sudo find / -xdev -perm -4000 -type f 2>/dev/null   # SUID প্রোগ্রাম
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/su
...
```

> 📝 **মনে রাখো:** পুরোনো `-perm +mode` রূপ এখন আর চলে না; তার বদলে `/mode`।

শুধু তোমার জন্য অনুমতি পরীক্ষা করতে `-readable`, `-writable`, `-executable` আছে (এগুলো আসল access চেক করে, ACL-সহ):

```
$ find /etc -maxdepth 1 -writable    # তুমি লিখতে পারো এমন
```

---

## ১২.৯ গভীরতা নিয়ন্ত্রণ: `-maxdepth`, `-mindepth`, `-xdev`

```
 depth 0:  /etc
 depth 1:  /etc/ssh
 depth 2:  /etc/ssh/sshd_config
```

```
$ find /etc -maxdepth 1 -name "*.conf"      # শুধু /etc-এর সরাসরি ফাইল
/etc/host.conf
/etc/resolv.conf
/etc/sysctl.conf
...

$ find /etc -mindepth 2 -maxdepth 2 -name "*.conf"   # ঠিক এক লেভেল নিচে
```

`-xdev` (বা `-mount`) অন্য ফাইলসিস্টেমে ঢোকে না। `/` থেকে খুঁজলে এটা দিলে `/proc`, `/sys`, নেটওয়ার্ক মাউন্ট, USB ড্রাইভ বাদ পড়ে, ফলে সার্চ অনেক দ্রুত আর পরিষ্কার হয় ([অধ্যায় ৩২](32-disk-partition-mount.md))।

> ⚠️ **সতর্কতা:** `-maxdepth`, `-mindepth`, `-xdev` হলো global option, তাই এগুলো starting point-এর ঠিক পরে, অন্য test-এর **আগে** লেখো। পরে লিখলে GNU find warning দেয়: `find: warning: you have specified the global option -maxdepth after the argument -name...`।

### সিম্বলিক লিংক অনুসরণ

default-এ `find` লিংকের ভেতরে ঢোকে না (`-P` আচরণ)। `-L` দিলে লিংক অনুসরণ করে; `-H` শুধু কমান্ড লাইনে দেওয়া পাথের লিংক অনুসরণ করে। এগুলো starting point-এর **আগে** বসে:

```
$ find -L /opt/app -name "*.jar"
```

---

## ১২.১০ লজিক্যাল অপারেটর

| অপারেটর | বিকল্প লেখা | মানে |
|---|---|---|
| `expr1 expr2` | `expr1 -a expr2`, `-and` | AND (দুটোই সত্য) — কিছু না লিখলে এটাই |
| `expr1 -o expr2` | `-or` | OR (যেকোনো একটা) |
| `! expr` | `-not expr` | NOT (উল্টো) |
| `\( expr \)` | | গ্রুপ করা (বন্ধনী) |
| `expr1 , expr2` | | দুটোই চালাও, শেষেরটার ফল নাও |

```
$ find . -name "*.jpg" -o -name "*.png"            # jpg অথবা png
$ find . -type f ! -name "*.txt"                   # txt বাদে সব ফাইল
$ find . -type f \( -name "*.jpg" -o -name "*.png" \) -size +1M
```

### অগ্রাধিকার (precedence) এর ফাঁদ

গণিতে যেমন গুণ যোগের আগে হয়, তেমনি `find`-এ **AND, OR-এর আগে** হয়। ক্রম: `\( \)` > `!` > `-a` > `-o`।

```
$ find . -name "*.jpg" -o -name "*.png" -print
```

এটা আসলে মানে:

```
  ( -name "*.jpg" )  -o  ( -name "*.png" -a -print )
```

ফলে শুধু png ফাইলগুলো print হয়! jpg মিললেও print হয় না, কারণ `-print` শুধু ডান দিকের অংশের সাথে AND হয়েছে। সমাধান বন্ধনী:

```
$ find . \( -name "*.jpg" -o -name "*.png" \) -print
```

> 💡 **টিপস:** `-o` ব্যবহার করলে সবসময় বন্ধনী দাও। বন্ধনীর আগে `\` দিতে হয় (বা `'('` কোটেশনে), নাহলে shell সেটাকে নিজের subshell চিহ্ন ভাববে। বন্ধনীর দুপাশে স্পেস লাগবে।

> 📝 যখন expression-এ `-prune` ছাড়া আর কোনো action (`-print`, `-exec`, `-delete`, `-ls` ইত্যাদি) থাকে না, `find` পুরো expression-কে `\( ... \) -print` হিসেবে ধরে। তাই `find . -name "*.jpg" -o -name "*.png"` (কোনো `-print` ছাড়া) ঠিকঠাক দুটোই দেখায়।

### `-prune`: কোনো ডিরেক্টরি পুরোপুরি বাদ দেওয়া

`! -path "./node_modules/*"` লিখলে ফলাফল থেকে বাদ যায়, কিন্তু `find` তবুও ভেতরে ঢুকে সব ফাইল দেখে (সময় নষ্ট)। `-prune` বলে "এই ডিরেক্টরির ভেতরে ঢুকবেই না":

```
$ find . -path ./node_modules -prune -o -name "*.js" -print
```

পড়ার নিয়ম: "পাথটা `./node_modules` হলে prune করো (ঢুকো না), **নাহলে** নাম `*.js` হলে print করো।"

একাধিক ডিরেক্টরি বাদ:

```
$ find . \( -name .git -o -name node_modules -o -name vendor \) -prune -o -type f -name "*.php" -print
```

> ⚠️ **সতর্কতা:** `-prune`-এর সাথে শেষে স্পষ্ট `-print` দিতেই হবে, নাহলে বাদ দেওয়া ডিরেক্টরির নামটাও আউটপুটে চলে আসবে।

---

## ১২.১১ আউটপুট সাজানো: `-print`, `-ls`, `-printf`

| action | কাজ |
|---|---|
| `-print` | পাথ, তারপর newline (default) |
| `-print0` | পাথ, তারপর NUL অক্ষর (`\0`) — নিরাপদ পাইপের জন্য |
| `-ls` | `ls -dils`-এর মতো বিস্তারিত |
| `-printf FORMAT` | নিজের মতো ফরম্যাট (GNU) |
| `-fprint FILE` | ফলাফল সরাসরি ফাইলে |
| `-quit` | প্রথমটা পেলেই থামো |

```
$ find ~/project -name "*.c" -ls
  1835123   4 -rw-r--r--   1 rahim  rahim    1240 Nov 10 14:02 /home/rahim/project/main.c
```

`-printf`-এর দরকারি ফরম্যাট:

| কোড | মানে |
|---|---|
| `%p` | পুরো পাথ |
| `%f` | শুধু ফাইলের নাম |
| `%h` | পাথের ডিরেক্টরি অংশ |
| `%s` | সাইজ (বাইট) |
| `%u` / `%g` | মালিক / গ্রুপের নাম |
| `%m` | পারমিশন (অক্টাল) |
| `%M` | পারমিশন (`-rw-r--r--` রূপে) |
| `%TY-%Tm-%Td %TH:%TM` | mtime তারিখ-সময় |
| `%T@` | mtime, epoch থেকে সেকেন্ড |
| `%d` | গভীরতা |
| `%i` | inode নম্বর |
| `\n` | নতুন লাইন |

```
$ find ~/project -type f -printf "%s\t%p\n" | sort -n | tail -3
2048    /home/rahim/project/README.md
10240   /home/rahim/project/data.csv
524288  /home/rahim/project/logo.png

$ find . -maxdepth 1 -type f -printf "%M %u %TY-%Tm-%Td %f\n"
-rw-r--r-- rahim 2026-11-10 notes.txt
```

প্রথম মিলেই থামা (বড় ডিস্কে সময় বাঁচায়):

```
$ find / -name "nginx.conf" -print -quit 2>/dev/null
/etc/nginx/nginx.conf
```

---

## ১২.১২ খুঁজে পাওয়া ফাইলে কাজ চালানো: `-exec`

`find` শুধু খুঁজেই থামে না; পাওয়া প্রতিটা ফাইলের ওপর অন্য কমান্ড চালাতে পারে।

### `-exec কমান্ড {} \;`

```
$ find . -name "*.txt" -exec wc -l {} \;
12 ./notes.txt
40 ./docs/intro.txt
```

- `{}` জায়গায় বসে পাওয়া ফাইলের পাথ।
- `\;` বলে কমান্ড এখানে শেষ। `;` shell-এর বিশেষ অক্ষর, তাই `\` দিয়ে বা `';'` লিখে escape করতে হয়।
- **প্রতিটা ফাইলের জন্য একবার** কমান্ড চলে।

### `-exec কমান্ড {} +`

```
$ find . -name "*.txt" -exec wc -l {} +
12 ./notes.txt
40 ./docs/intro.txt
52 total
```

এখানে `find` অনেকগুলো ফাইলের নাম জড়ো করে **একবারে** কমান্ড চালায় (`wc -l ./notes.txt ./docs/intro.txt`)। তাই আউটপুটে `total` এসেছে।

```
  -exec ... {} \;                     -exec ... {} +
  ┌──────────────────────┐            ┌─────────────────────────────────┐
  │ wc -l a.txt          │  fork+exec │ wc -l a.txt b.txt c.txt ... │
  │ wc -l b.txt          │  ×1000     │ (কমান্ড-লাইন সীমা ভরলে        │
  │ wc -l c.txt          │            │  আরেকটা ব্যাচ)                 │
  │ ... (১০০০ বার)        │            │ fork+exec ×1 বা কয়েকবার        │
  └──────────────────────┘            └─────────────────────────────────┘
       ধীর                                   অনেক দ্রুত
```

ভেতরের কারণ: প্রতিটা কমান্ড চালাতে kernel-কে নতুন process বানাতে হয় (`fork` + `exec`, [অধ্যায় ২৪](24-process-management.md))। ১০,০০০ ফাইলে ১০,০০০টা process বনাম হাতে-গোনা কয়েকটা, পার্থক্য বিশাল।

> 📝 **মনে রাখো:** `+` রূপে `{}` অবশ্যই `+`-এর ঠিক আগে, শেষে থাকতে হবে। `-exec cp {} /backup/ +` চলবে না। এমন ক্ষেত্রে কমান্ডের "গন্তব্য আগে" অপশন ব্যবহার করো: `-exec cp -t /backup/ {} +` (GNU cp-এর `-t`)।

### `-execdir`: নিরাপদ বিকল্প

`-execdir` কমান্ডটা চালায় ফাইলটা যে ডিরেক্টরিতে আছে **সেখানে গিয়ে**, আর `{}`-এ দেয় `./ফাইলের_নাম`। এতে race condition কমে, আর কিছু কমান্ডে (যেমন একই ফোল্ডারে rename) সুবিধা হয়:

```
$ find . -name "*.jpeg" -execdir mv {} {}.bak \;
```

### `-ok`: জিজ্ঞেস করে চালাও

`-exec`-এর মতো, কিন্তু প্রতিবার নিশ্চিত হতে চায়:

```
$ find . -name "*.tmp" -ok rm {} \;
< rm ... ./a.tmp > ? y
< rm ... ./b.tmp > ? n
```

শিখতে থাকার সময় ঝুঁকিপূর্ণ কাজে `-ok` (বা `-okdir`) ব্যবহার করো।

### `-exec` দিয়ে shell-এর জটিল কাজ

একটা কমান্ডে পাইপ বা একাধিক ধাপ লাগলে `sh -c` ব্যবহার করো:

```
$ find . -name "*.log" -exec sh -c 'gzip -9 "$1" && echo "compressed: $1"' _ {} \;
```

`_` হলো `$0`-এর জায়গা, আর `{}` যায় `$1`-এ। ফাইলের নাম কখনো সরাসরি `sh -c '... {} ...'`-এর স্ট্রিং-এর ভেতরে বসিও না: অদ্ভুত নামের ফাইল (`; rm -rf ~`) তখন কোড হিসেবে চলে যেতে পারে।

`-exec`-ও একটা test: কমান্ড exit status 0 দিলে true। তাই ফিল্টার হিসেবেও ব্যবহার করা যায়:

```
$ find . -name "*.txt" -exec grep -q "TODO" {} \; -print   # যে txt-তে TODO আছে
```

---

## ১২.১৩ `-delete`: খুঁজে মুছে ফেলা

```
$ find /tmp/myapp -type f -name "*.tmp" -delete
$ find ~/project -type d -empty -delete     # খালি ডিরেক্টরি মুছো
```

`-delete` নিজে থেকেই `-depth` চালু করে, মানে ডিরেক্টরির ভেতরের জিনিস আগে প্রসেস হয়, ডিরেক্টরি নিজে পরে। তাই খালি ডিরেক্টরির শিকল (a/b/c সব খালি) একবারেই মুছে যায়। তবে `-delete` খালি না এমন ডিরেক্টরি মুছতে পারে না।

> ⚠️ **সতর্কতা (খুব গুরুত্বপূর্ণ):**
> 1. `-delete` সবসময় **শেষে** রাখো। `find . -delete -name "*.tmp"` লিখলে বাম থেকে ডানে প্রথমেই `-delete` চলে, তাই **সব ফাইল** মুছে যায়, তারপর নাম চেক হয়!
> 2. আগে `-delete` ছাড়া (`-print` দিয়ে) চালিয়ে তালিকা দেখো, তারপর একই কমান্ডের শেষে `-delete` যোগ করো।
> 3. `-delete` দিয়ে মুছলে Trash-এ যায় না, ফেরত আনা যায় না।
> 4. `-delete` আর `-prune` একসাথে কাজ করে না (`-delete` `-depth` চালু করায় `-prune` অকার্যকর হয়)।

---

## ১২.১৪ `-print0` আর `xargs`

### `xargs` কী করে

`xargs` stdin থেকে লাইন/শব্দ পড়ে সেগুলোকে অন্য কমান্ডের **আর্গুমেন্ট** বানিয়ে দেয়। অনেক কমান্ড (যেমন `rm`, `cp`) stdin পড়ে না, আর্গুমেন্ট চায়; `xargs` সেই সেতু।

```
  find ... ──(পাথের তালিকা)──► xargs ──► rm a b c d ...
            stdout → stdin              আর্গুমেন্ট বানায়
```

```
$ find . -name "*.bak" | xargs rm
```

### সমস্যা: ফাঁকা জায়গাওয়ালা নাম

ধরো ফাইলের নাম `My Song.mp3`। সাধারণ `xargs` স্পেস দেখলে আলাদা শব্দ ভাবে, তাই `rm My` আর `rm Song.mp3` চালাতে চায়: ভুল ফাইল মুছতে পারে বা এরর দেয়। লিনাক্সে ফাইলের নামে newline-ও থাকতে পারে! একমাত্র যে অক্ষর পাথে থাকতে পারে না তা হলো **NUL (`\0`)**।

সমাধান: `find` NUL দিয়ে আলাদা করবে, `xargs` NUL দিয়ে পড়বে।

```
$ find . -name "*.mp3" -print0 | xargs -0 ls -l
-rw-r--r-- 1 rahim rahim 4123456 Nov  2 10:00 './My Song.mp3'
```

> 💡 **টিপস:** `find` থেকে `xargs`-এ পাঠালে **সবসময়** `-print0 | xargs -0` ব্যবহার করো। এটাকে অভ্যাস বানাও।

### `xargs`-এর দরকারি অপশন

| অপশন | কাজ |
|---|---|
| `-0` | ইনপুট NUL দিয়ে আলাদা |
| `-n N` | একবারে সর্বোচ্চ N টা আর্গুমেন্ট |
| `-I {}` | প্রতিটা ইনপুট আইটেমের জন্য একবার চালাও, `{}` জায়গায় বসাও |
| `-P N` | একসাথে N টা process সমান্তরালে (parallel) চালাও |
| `-r` | ইনপুট খালি হলে কমান্ড চালিও না (GNU-তে `--no-run-if-empty`) |
| `-t` | চালানোর আগে কমান্ডটা stderr-এ দেখাও |
| `-p` | প্রতিবার জিজ্ঞেস করো |
| `-d DELIM` | নিজের পছন্দের বিভাজক (GNU) |

```
$ find . -name "*.png" -print0 | xargs -0 -r -P 4 -n 10 optipng -o2
# ৪টা CPU core-এ সমান্তরালে, প্রতিবার ১০টা করে ছবি

$ find . -name "*.txt" -print0 | xargs -0 -I {} cp {} /backup/
```

### `-exec +` না `xargs`?

| বিষয় | `-exec {} +` | `-print0 \| xargs -0` |
|---|---|---|
| অদ্ভুত নামে নিরাপদ | হ্যাঁ | হ্যাঁ (`-0` দিলে) |
| ব্যাচিং | হ্যাঁ | হ্যাঁ |
| সমান্তরাল চালানো | না | হ্যাঁ (`-P`) |
| ব্যাচ সাইজ নিয়ন্ত্রণ | না | হ্যাঁ (`-n`) |
| বাড়তি process | না | হ্যাঁ (`xargs` নিজে) |

সাধারণ কাজে `-exec ... {} +` সবচেয়ে সহজ। সমান্তরাল বা বিশেষ নিয়ন্ত্রণ দরকার হলে `xargs`। `xargs` নিয়ে আরও বিস্তারিত [অধ্যায় ২৩](23-text-tools.md)-এ।

### shell লুপ দিয়ে (নিরাপদ পদ্ধতি)

```bash
find . -name "*.jpg" -print0 | while IFS= read -r -d '' f; do
    echo "প্রসেস করছি: $f"
done
```

লুপ ও `read` [অধ্যায় ৪২](42-shell-scripting-2.md)-এ বিস্তারিত।

---

## ১২.১৫ `locate` / `plocate` ও `updatedb`

### কীভাবে কাজ করে

```
  দিনে একবার (systemd timer)                     যখন তুমি খোঁজো
  ┌────────────┐   পুরো ডিস্ক ঘুরে   ┌──────────────────────┐   ┌─────────┐
  │  updatedb  │ ──সব পাথ লেখে──────►│ /var/lib/plocate/    │◄──│ locate  │
  └────────────┘                     │      plocate.db       │   │  foo    │
                                     │ (সংকুচিত index)       │   └─────────┘
                                     └──────────────────────┘    মিলিসেকেন্ডে
                                                                 উত্তর
```

`updatedb` পুরো ফাইলসিস্টেম একবার ঘুরে সব পাথ একটা ডেটাবেসে লিখে রাখে। `locate` শুধু সেই ডেটাবেসে খোঁজে, তাই ডিস্ক ঘোরা লাগে না। `plocate` হলো আধুনিক সংস্করণ: সংকুচিত trigram index ব্যবহার করে, তাই লাখ লাখ ফাইলেও প্রায় সঙ্গে সঙ্গে উত্তর দেয়।

### distro-ভেদে

| Distro | প্যাকেজ | ইনস্টল |
|---|---|---|
| Ubuntu 24.04 / Debian 12 | `plocate` | `sudo apt install plocate` |
| Fedora (৩৬ থেকে) | `plocate` (আগে `mlocate`) | `sudo dnf install plocate` |
| RHEL 9 / Rocky / Alma 9 | `mlocate` | `sudo dnf install mlocate` |

সব ক্ষেত্রে কমান্ডের নাম `locate` আর `updatedb` (plocate প্যাকেজ `plocate` নামেও কমান্ড দেয়)। ইনস্টলের পর ডেটাবেস আপডেট হয় systemd timer দিয়ে (Ubuntu/Debian/Fedora-তে `plocate-updatedb.timer`, RHEL 9-এর mlocate-এ `mlocate-updatedb.timer`), দিনে একবার। timer নিয়ে [অধ্যায় ২৯](29-scheduling-cron.md)।

ইনস্টলের ঠিক পরে ডেটাবেস থাকে না, তাই প্রথমবার হাতে চালাও:

```
$ sudo updatedb
$ locate sshd_config
/etc/ssh/sshd_config
/usr/share/doc/openssh-server/examples/sshd_config
```

### দরকারি অপশন

| অপশন | কাজ |
|---|---|
| `-i` | বড়-ছোট অক্ষর উপেক্ষা |
| `-c` | শুধু কয়টা মিলেছে সেই সংখ্যা |
| `-l N` / `-n N` | প্রথম N টা ফল |
| `-b` | শুধু basename-এ মেলাও (পুরো পাথে না) |
| `-r REGEX` | regex দিয়ে খোঁজো |
| `-0` | NUL দিয়ে আলাদা আউটপুট |
| `-S` | ডেটাবেসের পরিসংখ্যান (mlocate) |

```
$ locate -i readme.md | head -3
$ locate -c "*.conf"
842
$ locate -b '\passwd'          # ঠিক "passwd" নামের ফাইল (\ দিলে * যোগ হয় না)
/etc/passwd
/usr/bin/passwd
$ locate -r '\.ya\?ml$' | head
```

📝 প্যাটার্নে wildcard না থাকলে `locate` ধরে নেয় `*প্যাটার্ন*`, মানে পাথের যেকোনো জায়গায় মিললেই হলো। তাই `locate passwd` অনেক ফল দেয়।

### সীমাবদ্ধতা

- **হালনাগাদ না:** আজ বানানো ফাইল `sudo updatedb` না চালানো পর্যন্ত পাবে না। আর ডিলিট হওয়া ফাইল তালিকায় থেকে যেতে পারে।
- শুধু নাম/পাথ দিয়ে খোঁজে; সাইজ, সময়, মালিক দিয়ে খোঁজা যায় না।
- কিছু পাথ ইচ্ছা করে বাদ দেওয়া হয়: `/etc/updatedb.conf`-এ `PRUNEPATHS` (যেমন `/tmp`, `/var/spool`), `PRUNEFS` (যেমন `nfs`, `proc`), `PRUNENAMES` (যেমন `.git`)।
- **নিরাপত্তা:** ডেটাবেসে সব ফাইলের নাম থাকে, কিন্তু plocate/mlocate তোমাকে শুধু সেই ফাইলগুলোই দেখায় যেগুলোর ডিরেক্টরিতে তোমার অনুমতি আছে।

```
$ grep PRUNE /etc/updatedb.conf
PRUNE_BIND_MOUNTS="yes"
PRUNENAMES=".git .bzr .hg .svn"
PRUNEPATHS="/tmp /var/spool /media /var/lib/os-prober /var/lib/ceph /home/.ecryptfs /var/lib/schroot"
PRUNEFS="NFS afs autofs binfmt_misc ceph cgroup ... proc ... tmpfs ..."
```

(তোমার সিস্টেমে মান কিছুটা আলাদা হতে পারে।)

> 💡 **কখন কোনটা:** "এই নামের কোনো ফাইল সিস্টেমে কোথায় আছে?" → `locate` (দ্রুত)। "গত এক ঘণ্টায় বদলানো 10MB-এর বড় ফাইল" → `find`।

---

## ১২.১৬ কমান্ড খোঁজা: প্রোগ্রামটা আসলে কোথায়?

তুমি টার্মিনালে `ls` লিখলে shell কীভাবে জানে কোন প্রোগ্রাম চালাবে? shell একটা নির্দিষ্ট ক্রমে খোঁজে:

```
  তুমি লিখলে:  ls
        │
        ▼
  1. alias?          (alias ls='ls --color=auto')   ──হ্যাঁ──► সেটা বিস্তার করো
        │না
        ▼
  2. shell keyword?  (if, for, while ...)
        │না
        ▼
  3. function?       (তুমি বানানো shell ফাংশন)
        │না
        ▼
  4. builtin?        (cd, echo, type, export ...)
        │না
        ▼
  5. hash table?     (আগে খুঁজে রাখা পাথ মনে আছে?)
        │না
        ▼
  6. $PATH-এর প্রতিটা ডিরেক্টরি বাম থেকে ডানে:
     /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:...
     প্রথম যেখানে executable 'ls' পাওয়া গেল → চালাও
        │কোথাও নেই
        ▼
     "ls: command not found"
```

```
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

`PATH` নিয়ে বিস্তারিত [অধ্যায় ৪৪](44-environment-bashrc.md)-এ।

### `which`

`$PATH`-এ খুঁজে executable ফাইলের পাথ দেখায়:

```
$ which python3
/usr/bin/python3

$ which -a python3        # PATH-এ যতগুলো আছে সব
/usr/bin/python3
/bin/python3
```

(Ubuntu/Debian-এ `/bin` হলো `/usr/bin`-এর লিংক, তাই একই প্রোগ্রাম দুবার দেখা যায়। এটাকে "usrmerge" বলে; [অধ্যায় ৭](07-filesystem-hierarchy.md) দেখো।)

> ⚠️ **সতর্কতা:** `which` একটা বাইরের প্রোগ্রাম; সে shell-এর alias, function বা builtin চেনে না। `which cd` হয় কিছুই দেখায় না, নয়তো ভুল কিছু দেখায়, অথচ `cd` আসলে builtin। আসল উত্তর চাইলে `type` ব্যবহার করো। (Fedora/RHEL-এ `which` একটা alias হিসেবে সেট করা থাকতে পারে যা alias-ও দেখায়।)

### `type` (bash builtin)

shell আসলে কোনটা চালাবে, তার সবচেয়ে সঠিক উত্তর:

```
$ type ls
ls is aliased to `ls --color=auto'

$ type cd
cd is a shell builtin

$ type if
if is a shell keyword

$ type python3
python3 is /usr/bin/python3

$ type -a echo
echo is a shell builtin
echo is /usr/bin/echo
echo is /bin/echo

$ type -t ls
alias
```

| অপশন | কাজ |
|---|---|
| `-a` | সব সম্ভাব্য জায়গা (alias, builtin, ফাইল) |
| `-t` | শুধু ধরন: `alias`, `keyword`, `function`, `builtin`, `file` |
| `-p` | ডিস্কের ফাইল হলে শুধু পাথ |
| `-P` | alias/function উপেক্ষা করে PATH-এ খোঁজো |

### `command -v`

POSIX মানসম্মত, সব shell-এ (bash, dash, zsh) চলে। স্ক্রিপ্টে কোনো প্রোগ্রাম ইনস্টল আছে কিনা দেখার **সবচেয়ে ভালো উপায়**:

```
$ command -v git
/usr/bin/git
$ command -v ll
alias ll='ls -alF'

if command -v docker >/dev/null 2>&1; then
    echo "docker আছে"
else
    echo "docker ইনস্টল করো"
fi
```

`command`-এর আরেকটা কাজ: alias/function এড়িয়ে আসল কমান্ড চালানো। `command ls` লিখলে alias ছাড়া `ls` চলে (`\ls` লিখলেও alias এড়ানো যায়)।

### `whereis`

শুধু executable না, সাথে **man page আর source**-ও খোঁজে, নির্দিষ্ট কিছু স্ট্যান্ডার্ড ডিরেক্টরিতে (PATH-এর ওপর পুরোপুরি নির্ভর করে না):

```
$ whereis ls
ls: /usr/bin/ls /usr/share/man/man1/ls.1.gz

$ whereis -b bash     # শুধু binary
bash: /usr/bin/bash

$ whereis -m passwd   # শুধু man page
passwd: /usr/share/man/man1/passwd.1ssl.gz /usr/share/man/man5/passwd.5.gz /usr/share/man/man1/passwd.1.gz
```

| অপশন | কাজ |
|---|---|
| `-b` | শুধু binary |
| `-m` | শুধু manual |
| `-s` | শুধু source |
| `-u` | যেগুলোর কোনো একটা অংশ পাওয়া যায়নি |

### `hash`: shell-এর মনে রাখা খাতা

bash প্রতিবার পুরো PATH ঘোরে না; একবার পেলে পাথটা একটা hash table-এ মনে রাখে।

```
$ hash
hits    command
   3    /usr/bin/git
   1    /usr/bin/ls

$ hash -r     # খাতা মুছে ফেলো
```

📝 কোনো প্রোগ্রাম এক জায়গা থেকে অন্য জায়গায় সরালে (যেমন `/usr/local/bin`-এ নতুন ভার্সন বসালে) bash তবুও পুরোনো পাথ চালাতে চাইতে পারে এবং `No such file or directory` দেখাতে পারে। তখন `hash -r` চালাও।

### নাম জানা নেই, কাজ জানা আছে: `apropos` / `man -k`

```
$ apropos "copy files"
cp (1)               - copy files and directories
...
$ man -k partition
fdisk (8)            - manipulate disk partition table
parted (8)           - a partition manipulation program
...
```

এগুলো man page-এর ছোট বিবরণে খোঁজে (ডেটাবেস `mandb` দিয়ে বানানো)। কোনো কমান্ড ইনস্টল নেই কিন্তু কোন প্যাকেজে আছে জানতে চাইলে Ubuntu-তে `command-not-found` সাহায্য করে, আর `apt-file search` / `dnf provides` ([অধ্যায় ৩০](30-package-management.md))।

### তুলনা টেবিল

| টুল | ধরন | alias/function/builtin চেনে | man page দেখায় | স্ক্রিপ্টে উপযুক্ত |
|---|---|---|---|---|
| `which` | বাইরের প্রোগ্রাম | না | না | মোটামুটি |
| `type` | bash builtin | হ্যাঁ | না | bash-এ হ্যাঁ |
| `command -v` | POSIX builtin | হ্যাঁ | না | **সবচেয়ে ভালো** |
| `whereis` | বাইরের প্রোগ্রাম | না | হ্যাঁ | না |

---

## ১২.১৭ `fd`: আধুনিক, দ্রুত, সহজ

`fd` হলো Rust-এ লেখা `find`-এর একটা আধুনিক বিকল্প। ব্যবহারে সহজ, রঙিন আউটপুট, আর একাধিক থ্রেডে সমান্তরালে খোঁজে।

### ইনস্টল

| Distro | কমান্ড | কমান্ডের নাম |
|---|---|---|
| Ubuntu / Debian | `sudo apt install fd-find` | `fdfind` (নাম-সংঘাত এড়াতে) |
| Fedora | `sudo dnf install fd-find` | `fd` |
| RHEL 9 | EPEL রিপোজিটরি থেকে `fd-find` | `fd` |

Ubuntu/Debian-এ `fd` নামে চালাতে:

```
$ mkdir -p ~/.local/bin
$ ln -s $(which fdfind) ~/.local/bin/fd
```

(নিশ্চিত হও `~/.local/bin` তোমার `PATH`-এ আছে।)

### `find` থেকে কী আলাদা

- প্যাটার্ন default-এ **regex**, আর পুরো পাথ না, নামের যেকোনো অংশে মেলে।
- **smart case**: প্যাটার্ন পুরোটা ছোট হাতের হলে case-insensitive, কোনো বড় হাতের অক্ষর থাকলে case-sensitive।
- default-এ **লুকানো ফাইল** (`.` দিয়ে শুরু) আর **`.gitignore`-এ থাকা ফাইল** বাদ দেয়। প্রজেক্ট ফোল্ডারে এটা খুব আরামদায়ক (`node_modules`, `vendor` আপনিই বাদ)।

```
$ fd config
src/config.rs
docs/configuration.md

$ fd -e md                 # এক্সটেনশন
README.md
docs/configuration.md

$ fd -t d src              # শুধু ডিরেক্টরি
$ fd -H bashrc ~           # লুকানো ফাইলসহ
/home/rahim/.bashrc
```

### দরকারি অপশন

| অপশন | কাজ | find-এর সমতুল্য |
|---|---|---|
| `-e EXT` | এক্সটেনশন | `-name "*.EXT"` |
| `-t f/d/l/x/e` | ফাইল/ডিরেক্টরি/লিংক/executable/empty | `-type`, `-empty` |
| `-H` | লুকানো ফাইলও | (default) |
| `-I` | `.gitignore` মানবে না | (default) |
| `-s` / `-i` | case-sensitive / insensitive | `-name` / `-iname` |
| `-g` | regex-এর বদলে glob প্যাটার্ন | `-name` |
| `-p` | পুরো পাথে মেলাও | `-path`/`-regex` |
| `-d N` | সর্বোচ্চ গভীরতা | `-maxdepth N` |
| `-E PATTERN` | বাদ দাও | `-prune` |
| `-S +1m` | সাইজ | `-size` |
| `--changed-within 2d` | সাম্প্রতিক বদল | `-mtime -2` |
| `--changed-before 1w` | পুরোনো | `-mtime +7` (মোটামুটি) |
| `-x CMD` | প্রতিটার জন্য কমান্ড | `-exec {} \;` |
| `-X CMD` | সবগুলো একবারে | `-exec {} +` |
| `-0` | NUL বিভাজক | `-print0` |
| `-l` | `ls -l` ধরনের তালিকা | `-ls` |

```
$ fd -e jpg -x convert {} {.}.png     # প্রতিটা jpg → png
$ fd -e log -X rm                      # সব log একবারে মুছো
$ fd -t f -S +100m . ~                 # 100 MB-এর বড় ফাইল
$ fd -E node_modules -e js
```

`-x`-এ বিশেষ placeholder: `{}` পুরো পাথ, `{/}` basename, `{//}` parent ডিরেক্টরি, `{.}` এক্সটেনশন ছাড়া পাথ, `{/.}` এক্সটেনশন ছাড়া basename।

> 💡 **টিপস:** রোজকার প্রজেক্টের ভেতরে `fd` দারুণ। কিন্তু সার্ভারে `fd` ইনস্টল না-ও থাকতে পারে, আর স্ক্রিপ্টে portability দরকার। তাই `find` ভালোভাবে জানা বাধ্যতামূলক, `fd` বোনাস।

---

## ১২.১৮ ব্যবহারিক রেসিপি

```bash
# ১. ডিস্ক ভরে গেছে: সবচেয়ে বড় ২০টা ফাইল
sudo find / -xdev -type f -size +100M -printf "%s\t%p\n" 2>/dev/null | sort -rn | head -20 | numfmt --field=1 --to=iec

# ২. শেষ ১০ মিনিটে কোন কনফিগ ফাইল বদলেছে
sudo find /etc -type f -mmin -10

# ৩. ৩০ দিনের পুরোনো লগ কম্প্রেস
sudo find /var/log/myapp -name "*.log" -mtime +30 -exec gzip {} +

# ৪. ৯০ দিনের পুরোনো .gz লগ মুছো (আগে -print দিয়ে দেখে নাও!)
sudo find /var/log/myapp -name "*.gz" -mtime +90 -print
sudo find /var/log/myapp -name "*.gz" -mtime +90 -delete

# ৫. ওয়েব রুটে ডিরেক্টরি 755, ফাইল 644
sudo find /var/www/html -type d -exec chmod 755 {} +
sudo find /var/www/html -type f -exec chmod 644 {} +

# ৬. সবাই লিখতে পারে এমন ফাইল (নিরাপত্তা ঝুঁকি)
sudo find / -xdev -type f -perm -o=w 2>/dev/null

# ৭. SUID/SGID প্রোগ্রামের তালিকা (নিরাপত্তা অডিট, অধ্যায় ৪৫)
sudo find / -xdev \( -perm -4000 -o -perm -2000 \) -type f -printf "%M %u %p\n" 2>/dev/null

# ৮. ভাঙা সিম্বলিক লিংক খুঁজে মুছো
find ~ -xtype l -print
find ~ -xtype l -delete

# ৯. ডুপ্লিকেট ফাইল: md5 হ্যাশ মিলিয়ে একই কনটেন্টের ফাইল
find ~/Pictures -type f -print0 | xargs -0 md5sum | sort | uniq -w32 -D

# ১০. প্রজেক্টে কয় লাইন PHP কোড (vendor বাদে)
find . -path ./vendor -prune -o -name "*.php" -print0 | xargs -0 cat | wc -l

# ১১. কোন ফাইলে "password" লেখা আছে (নাম + কনটেন্ট মিলিয়ে)
find . -type f -name "*.env*" -exec grep -l "password" {} +

# ১২. inode নম্বর দিয়ে ফাইল (অদ্ভুত নামের ফাইল মুছতে, অধ্যায় ১১)
ls -i
find . -maxdepth 1 -inum 1835123 -delete

# ১৩. কোনো ফাইলের সব হার্ড লিংক
find / -xdev -samefile /path/to/file 2>/dev/null

# ১৪. অনাথ ফাইল (মালিক ইউজার আর নেই)
sudo find / -xdev \( -nouser -o -nogroup \) 2>/dev/null

# ১৫. আজ বদলানো সব ফাইলের ব্যাকআপ আর্কাইভ (অধ্যায় ৩৪)
find ~/Documents -type f -daystart -mtime 0 -print0 | tar --null -czf today.tar.gz -T -

# ১৬. ফোল্ডারে কতগুলো ফাইল (টাইপ অনুযায়ী)
find . -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn | head

# ১৭. সবচেয়ে সম্প্রতি বদলানো ১০টা ফাইল
find . -type f -printf "%T@ %p\n" | sort -rn | head -10 | cut -d' ' -f2-

# ১৮. খালি ফোল্ডার পরিষ্কার
find ~/Downloads -mindepth 1 -type d -empty -delete
```

> 💡 **টিপস:** রেসিপি ১-এর `numfmt --to=iec` বাইটকে `1.2G` ধরনের পড়ার মতো রূপে দেখায়। ডিরেক্টরি অনুযায়ী জায়গা দেখতে `du -sh *` বা `ncdu` ([অধ্যায় ৩২](32-disk-partition-mount.md))।

> ⚠️ **সতর্কতা:** রেসিপি ১১-এর মতো কনটেন্ট খোঁজায় ছোট প্রজেক্টে `grep -rl "password" --include="*.env*" .` আরও সহজ ([অধ্যায় ২০](20-grep-regex.md))।

---

## ১২.১৯ খোঁজার সাথে সম্পর্কিত কীবোর্ড শর্টকাট

| শর্টকাট | কোথায় | কাজ |
|---|---|---|
| `Ctrl + C` | টার্মিনাল | চলমান `find` থামাও |
| `Ctrl + Z` | টার্মিনাল | `find` সাময়িক থামাও (`bg`/`fg` দিয়ে ফেরাও, [অধ্যায় ২৫](25-jobs-tmux.md)) |
| `Ctrl + S` / `Ctrl + Q` | টার্মিনাল | আউটপুট স্ক্রল থামাও / আবার চালাও |
| `Ctrl + R` | bash | আগে চালানো `find` কমান্ড হিস্ট্রিতে খোঁজো ([অধ্যায় ৬](06-terminal-shortcuts.md)) |
| `Tab` / `Tab Tab` | bash | পাথ বা কমান্ডের নাম সম্পূর্ণ করো / সব বিকল্প দেখাও |
| `Alt + .` | bash | আগের কমান্ডের শেষ আর্গুমেন্ট বসাও (একই পাথে আবার খুঁজতে) |
| `Ctrl + Shift + F` | GNOME Terminal | টার্মিনালের লেখার ভেতরে খোঁজো |
| `/লেখা`, `n`, `N` | `less` | অনেক লম্বা ফলাফল `find ... \| less` দিয়ে দেখার সময় খোঁজো, পরের/আগের মিল |
| `q` | `less` | বের হও |
| `Ctrl + F` বা সরাসরি টাইপ করা | Files (Nautilus) | GUI-তে ফাইল খোঁজো ([অধ্যায় ৪](04-desktop-environment.md)) |
| `Super` তারপর টাইপ | GNOME | অ্যাপ ও ফাইল খোঁজো |

---

## হাতে-কলমে অনুশীলন

একটা নিরাপদ খেলার মাঠ বানিয়ে নাও, যাতে ভুল হলেও আসল ফাইলের ক্ষতি না হয়।

**ধাপ ১: খেলার মাঠ তৈরি**

```
$ mkdir -p ~/findlab/{docs,photos,logs,code/node_modules,empty}
$ cd ~/findlab
$ touch docs/{a,b,c}.txt docs/Report.PDF "docs/my notes.txt"
$ touch photos/img_{0001..0005}.jpg photos/logo.png
$ for i in 1 2 3; do echo "line $i" > logs/app$i.log; done
$ touch code/main.js code/node_modules/lib.js
$ dd if=/dev/zero of=photos/big.raw bs=1M count=5 status=none
$ touch -d "40 days ago" logs/app1.log
$ chmod 777 code/main.js
$ ln -s /nowhere broken-link
$ tree
```

**ধাপ ২: নাম দিয়ে**

```
$ find . -name "*.txt"
$ find . -iname "*.pdf"
$ find . -name "img_000[1-3].jpg"
```
লক্ষ করো, `my notes.txt` ঠিকঠাক এসেছে কিনা।

**ধাপ ৩: টাইপ, সাইজ, empty**

```
$ find . -type d
$ find . -type f -size +1M          # big.raw আসবে
$ find . -empty                      # খালি ফাইল আর empty/ ডিরেক্টরি
$ find . -type f -size -1M | wc -l   # কতগুলো? কেন? (রাউন্ডিং মনে করো)
```

**ধাপ ৪: সময় আর পারমিশন**

```
$ find . -mtime +30                  # app1.log
$ find . -mmin -5                    # সদ্য তৈরি সব
$ find . -perm 777                   # main.js
$ find . -xtype l                    # broken-link
```

**ধাপ ৫: লজিক আর prune**

```
$ find . -name "*.jpg" -o -name "*.png" -print      # কী এলো? কেন শুধু png?
$ find . \( -name "*.jpg" -o -name "*.png" \) -print
$ find . -name "*.js"
$ find . -path ./code/node_modules -prune -o -name "*.js" -print
```

**ধাপ ৬: exec ও xargs**

```
$ find logs -name "*.log" -exec wc -l {} \;
$ find logs -name "*.log" -exec wc -l {} +
$ find . -name "*.txt" -print0 | xargs -0 ls -l
$ find . -name "*.txt" | xargs ls -l      # "my notes.txt"-এ কী এরর আসে?
```

**ধাপ ৭: নিরাপদে মোছা**

```
$ find logs -mtime +30 -print       # আগে দেখো
$ find logs -mtime +30 -delete      # তারপর মুছো
$ find . -type d -empty -print
```

**ধাপ ৮: locate পরীক্ষা**

```
$ locate findlab/docs/a.txt          # সম্ভবত কিছু আসবে না, কেন?
$ sudo updatedb
$ locate findlab/docs/a.txt
```

**ধাপ ৯: কমান্ড খোঁজা**

```
$ type ls cd if find
$ which -a ls
$ command -v find
$ whereis find
$ apropos "search for files"
```

**ধাপ ১০ (ঐচ্ছিক): fd**

```
$ fdfind -e jpg            # Fedora-তে: fd -e jpg
$ fdfind js                # node_modules কি আসে? (gitignore না থাকলে আসবে)
```

শেষে পরিষ্কার: `rm -rf ~/findlab`

---

## সারসংক্ষেপ

| কাজ | কমান্ড |
|---|---|
| নাম দিয়ে | `find DIR -name "*.txt"` |
| case উপেক্ষা | `find DIR -iname "*.jpg"` |
| পুরো পাথ | `find DIR -path "*/docs/*"` |
| ধরন | `-type f` / `d` / `l` |
| সাইজ | `-size +100M`, `-size -10k` |
| খালি | `-empty` |
| বদলের সময় | `-mtime -7`, `-mmin -30`, `-newermt "2026-11-01"` |
| মালিক | `-user rahim`, `-group www-data`, `-nouser` |
| পারমিশন | `-perm 644`, `-perm -4000`, `-perm /o=w` |
| গভীরতা | `-maxdepth 1`, `-mindepth 2`, `-xdev` |
| AND / OR / NOT | (খালি) বা `-a`, `-o`, `!`, `\( \)` |
| ডিরেক্টরি বাদ | `-path ./x -prune -o ... -print` |
| ফরম্যাট | `-ls`, `-printf "%s %p\n"` |
| প্রতিটায় কমান্ড | `-exec cmd {} \;` |
| একবারে কমান্ড | `-exec cmd {} +` |
| জিজ্ঞেস করে | `-ok cmd {} \;` |
| মোছা | `... -delete` (শেষে!) |
| নিরাপদ পাইপ | `-print0 \| xargs -0 cmd` |
| প্রথমটাতেই থামো | `-print -quit` |
| ডেটাবেস খোঁজা | `locate -i NAME`, `sudo updatedb` |
| executable কোথায় | `which -a`, `command -v` |
| shell কী চালাবে | `type -a NAME` |
| binary + man | `whereis NAME` |
| কাজ দিয়ে কমান্ড | `apropos KEYWORD`, `man -k` |
| PATH ক্যাশ মোছা | `hash -r` |
| আধুনিক বিকল্প | `fd -e EXT`, `fd -t d`, `fd -H`, `fd -x` |

---

## নিজেকে যাচাই করো

1. `find` আর `locate`-এর মূল পার্থক্য কী? কোন পরিস্থিতিতে `locate` ভুল উত্তর দেয়?
2. `find . -name *.txt` (কোটেশন ছাড়া) কেন বিপজ্জনক?
3. `-size -1M` দিয়ে খুঁজলে শুধু খালি ফাইল আসে কেন? 1 MiB-এর ছোট সব ফাইল কীভাবে খুঁজবে?
4. `-mtime +1` ঠিক কোন ফাইলগুলো মেলায়?
5. `-perm 755`, `-perm -755` আর `-perm /755`-এর পার্থক্য কী?
6. `find . -name "*.jpg" -o -name "*.png" -print` কেন jpg দেখায় না? ঠিক করো।
7. `-exec rm {} \;` আর `-exec rm {} +`-এর পার্থক্য কী, আর কোনটা দ্রুত কেন?
8. `find . -delete -name "*.tmp"` চালালে কী হবে?
9. `xargs`-এর সাথে `-print0` আর `-0` কেন দরকার?
10. `which cd` আর `type cd`-এর উত্তর আলাদা কেন? স্ক্রিপ্টে কোনো প্রোগ্রাম আছে কিনা দেখতে কোনটা ব্যবহার করবে?

<details><summary>উত্তর দেখো</summary>

1. `find` এই মুহূর্তে ডিস্ক ঘুরে খোঁজে (ধীর, কিন্তু সবসময় সঠিক, আর সাইজ/সময়/মালিক দিয়েও খোঁজা যায়)। `locate` আগে বানানো ডেটাবেসে খোঁজে (খুব দ্রুত, কিন্তু শুধু নাম দিয়ে)। শেষ `updatedb`-এর পরে তৈরি হওয়া ফাইল `locate` দেখায় না, আর ডিলিট হওয়া ফাইল দেখাতে পারে। `PRUNEPATHS`-এ থাকা জায়গাও (যেমন `/tmp`) দেখায় না।
2. shell আগেই `*.txt`-কে বর্তমান ডিরেক্টরির মিলে যাওয়া ফাইলের নাম দিয়ে বদলে দেয়। একটা মিললে শুধু সেই নামেই খোঁজা হয়, একাধিক মিললে `paths must precede expression` এরর আসে।
3. `find` সাইজকে একক অনুযায়ী উপরের দিকে রাউন্ড করে। 1 বাইটের ফাইলও 1M হয়ে যায়, তাই "1M-এর কম" মানে শুধু 0। সমাধান: `-size -1048576c` বা `-size -1024k`।
4. বয়স ÷ ২৪ ঘণ্টা (ভগ্নাংশ বাদ) 1-এর বেশি, মানে কমপক্ষে 48 ঘণ্টা পুরোনো ফাইল।
5. `755` = হুবহু rwxr-xr-x। `-755` = এই সব বিট সেট আছে (আরও বেশি থাকতে পারে, যেমন 777-ও মিলবে)। `/755` = এর মধ্যে যেকোনো একটা বিট সেট থাকলেই হলো।
6. AND-এর অগ্রাধিকার OR-এর চেয়ে বেশি, তাই `-print` শুধু `-name "*.png"`-এর সাথে জুড়েছে। ঠিক রূপ: `find . \( -name "*.jpg" -o -name "*.png" \) -print`।
7. `\;` প্রতিটা ফাইলের জন্য আলাদা `rm` process চালায়; `+` অনেক নাম জড়ো করে অল্প কয়েকবার চালায়। process তৈরি (fork+exec) ব্যয়বহুল, তাই `+` অনেক দ্রুত।
8. expression বাম থেকে ডানে চলে, তাই `-delete` আগে চলে, বর্তমান ডিরেক্টরির নিচের (মুছতে পারা) সব ফাইল ও খালি ডিরেক্টরি মুছে যায়। `-delete` সবসময় শেষে।
9. ফাইলের নামে স্পেস, ট্যাব, এমনকি newline থাকতে পারে, যা সাধারণ `xargs` ভেঙে আলাদা আর্গুমেন্ট বানায়। NUL অক্ষর পাথে থাকতে পারে না, তাই সেটাই একমাত্র নিরাপদ বিভাজক।
10. `which` বাইরের প্রোগ্রাম, শুধু PATH-এ ফাইল খোঁজে, shell builtin চেনে না। `type` হলো bash-এর নিজের কমান্ড, তাই জানে `cd` একটা builtin। স্ক্রিপ্টে `command -v NAME >/dev/null 2>&1` সবচেয়ে ভালো, কারণ এটা POSIX এবং সব shell-এ চলে।

</details>

---

[← আগের অধ্যায়](11-links-inode.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](13-nano-editor.md)

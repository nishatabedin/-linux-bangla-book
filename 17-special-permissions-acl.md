# অধ্যায় ১৭: বিশেষ পারমিশন, ACL ও ফাইল অ্যাট্রিবিউট

[অধ্যায় ১৬](16-permissions.md)-এ আমরা `rwx`, owner/group/others আর `chmod`, `chown`, `umask` শিখেছি। ওই ৯টি বিট দিয়ে বেশিরভাগ কাজ চলে যায়। তবু কিছু প্রশ্নের উত্তর ওখানে মেলে না:

- সাধারণ ইউজার `passwd` চালিয়ে নিজের পাসওয়ার্ড বদলায়। অথচ পাসওয়ার্ড থাকে `/etc/shadow`-এ, যেটায় শুধু root লিখতে পারে। তাহলে কাজটা হয় কীভাবে?
- `/tmp`-এ সবাই লিখতে পারে। তাহলে আমি কি অন্যের ফাইল মুছে দিতে পারি?
- একটা ফাইল শুধু একজন বাড়তি ইউজারকে পড়তে দিতে চাই, অথচ নতুন group বানাতে চাই না। উপায় কী?
- এমন ফাইল কি বানানো যায় যেটা root-ও ভুল করে মুছতে পারবে না?

এই অধ্যায়ে এসবের উত্তর পাবে।

## এই অধ্যায়ে যা শিখবে

- তিনটি বিশেষ পারমিশন বিট: **SUID**, **SGID** আর **sticky bit**। এগুলো কী, কেন আছে, ভেতরে কীভাবে কাজ করে
- Real UID আর Effective UID-এর পার্থক্য (SUID বোঝার মূল চাবি)
- `passwd` কীভাবে `/etc/shadow` বদলায়
- ফাইল আর ডিরেক্টরির ওপর SGID-এর দুই রকম আচরণ
- `/tmp`-এ sticky bit কেন থাকে
- 4/2/1 octal দিয়ে চার অঙ্কের মোড (`4755`, `2775`, `1777`)
- `ls -l`-এ `s`, `S`, `t`, `T` অক্ষরের মানে
- সিস্টেমের সব SUID/SGID ফাইল খোঁজা, আর এগুলোর নিরাপত্তা ঝুঁকি
- **ACL** (Access Control List): `getfacl`, `setfacl -m/-x/-b/-k/-d/-R`, mask, default ACL
- **ফাইল অ্যাট্রিবিউট**: `chattr` আর `lsattr` (`+i` immutable, `+a` append-only ইত্যাদি)
- **Linux capabilities**: `getcap`, `setcap`। root-এর ক্ষমতাকে ছোট ছোট টুকরো করে দেওয়ার পদ্ধতি

---

## ১৭.১ আগে একটু ঝালিয়ে নিই: সাধারণ পারমিশন কোথায় থাকে

প্রতিটি ফাইলের metadata থাকে তার **inode**-এ ([অধ্যায় ১১](11-links-inode.md))। সেখানে `mode` নামে একটা ১৬-বিটের ঘর আছে। এর ৪টি বিট ফাইলের ধরন বলে (regular file, directory, symlink ইত্যাদি), বাকি ১২টি পারমিশনের জন্য:

```
 mode ফিল্ডের পারমিশন অংশ (১২ বিট)
 ┌───────────────┬───────────┬───────────┬───────────┐
 │ বিশেষ বিট      │  owner(u) │  group(g) │ others(o) │
 │ SUID SGID Sticky│  r  w  x  │  r  w  x  │  r  w  x  │
 │  4    2    1   │  4  2  1  │  4  2  1  │  4  2  1  │
 └───────────────┴───────────┴───────────┴───────────┘
   ↑ এই অধ্যায়ের বিষয়    ↑──────── অধ্যায় ১৬-এর বিষয় ────────↑
```

অধ্যায় ১৬-এ আমরা শুধু ডানের ৯টি বিট দেখেছি (যেমন `755`)। এখন সবচেয়ে বাঁয়ের ৩টি বিটের পালা। এগুলো নিয়েই পুরো মোড চার অঙ্কের হয়, যেমন `4755`।

> 📝 **মনে রাখো:** `chmod 755` লেখা আর `chmod 0755` লেখা একই কথা। প্রথম অঙ্ক না দিলে ধরে নেওয়া হয় সেটা `0`, মানে কোনো বিশেষ বিট নেই। (ডিরেক্টরির ক্ষেত্রে একটা ব্যতিক্রম আছে, ১৭.৬-এ দেখবে।)

---

## ১৭.২ Real UID আর Effective UID: SUID বোঝার আগে

লিনাক্সে প্রতিটি process ([অধ্যায় ২৪](24-process-management.md)) কয়েকটা পরিচয় সাথে নিয়ে চলে:

| পরিচয় | মানে |
|---|---|
| **Real UID (RUID)** | যে ইউজার আসলে process-টা চালু করেছে |
| **Effective UID (EUID)** | পারমিশন চেক করার সময় kernel যে UID দেখে |
| **Saved set-user-ID** | EUID-এর পুরোনো মান জমা রাখার জায়গা, যাতে প্রোগ্রাম চাইলে পরে আবার ফিরে যেতে পারে |

একইভাবে group-এর জন্যও Real GID, Effective GID আর saved GID আছে।

সাধারণত RUID আর EUID একই থাকে। তুমি (`nishat`, UID 1000) `cat` চালালে process-এর RUID=1000, EUID=1000। kernel কোনো ফাইল খোলার অনুমতি দেবে কি না, সেটা ঠিক করে **EUID** দেখে, RUID নয়।

**উপমা:** তুমি একটা অফিসে গেছ। গেটে তোমার নিজের নাম লেখা হলো (Real UID)। কিন্তু রিসেপশন তোমার গলায় একটা "স্টাফ" পাস ঝুলিয়ে দিল (Effective UID)। ভেতরের দরজাগুলো খোলে পাস দেখে, খাতার নাম দেখে না।

SUID বিটের পুরো খেলাটা এই EUID বদলানো নিয়েই।

---

## ১৭.৩ SUID (Set User ID)

### SUID কী

কোনো executable ফাইলে SUID বিট দেওয়া থাকলে, ফাইলটা যে-ই চালাক, process-এর **Effective UID হয়ে যায় ফাইলের owner-এর UID**।

- octal মান: **4** (যেমন `4755`)
- symbolic: `u+s`
- `ls -l`-এ owner-এর execute ঘরে `x`-এর জায়গায় **`s`** দেখায়

### সবচেয়ে বিখ্যাত উদাহরণ: `passwd`

```
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 64152 May 30  2024 /usr/bin/passwd
```

(সাইজ আর তারিখ তোমার মেশিনে আলাদা হতে পারে।) owner-এর ঘরে `rws` দেখো। ওই `s` মানে SUID চালু আছে, আর execute বিটও আছে।

এবার `/etc/shadow` দেখো:

```
$ ls -l /etc/shadow
-rw-r----- 1 root shadow 1450 Oct  2 10:12 /etc/shadow
```

Ubuntu/Debian-এ এটা `root:shadow`, মোড `640`। Fedora/RHEL-এ সাধারণত `----------` (`000`) আর `root:root`। দুই ক্ষেত্রেই সাধারণ ইউজার এখানে লিখতে পারে না।

### ভেতরে কী ঘটে

```
  nishat (UID 1000) টার্মিনালে লিখল:  $ passwd
            │
            ▼
  ┌────────────────────────────────────────────┐
  │ shell fork() করে, তারপর execve("/usr/bin/passwd") │
  └────────────────────────────────────────────┘
            │  kernel ফাইলের inode দেখে: SUID বিট আছে, owner = root (UID 0)
            ▼
  ┌────────────────────────────────────────────┐
  │ নতুন process:  RUID = 1000 (nishat)          │
  │                EUID = 0    (root)  ← বদলে গেল│
  └────────────────────────────────────────────┘
            │
            ▼
  passwd নিজের কোডে getuid() দিয়ে দেখে "কে আসলে চালাল?" → 1000
  তাই শুধু nishat-এর পাসওয়ার্ড বদলাতে দেয় (অন্যেরটা নয়)
            │
            ▼
  /etc/shadow খোলার চেষ্টা → kernel EUID=0 দেখে → অনুমতি দেয় ✔
```

এখানে দুটো জিনিস একসাথে কাজ করছে:

1. **kernel** EUID বদলে দেয়, তাই ফাইলটা খোলা যায়।
2. **প্রোগ্রাম নিজে** RUID দেখে ঠিক করে কাকে কী করতে দেবে। তাই `nishat` চাইলেও `passwd alice` চালিয়ে alice-এর পাসওয়ার্ড বদলাতে পারে না। ওই নিয়ম `passwd`-এর কোডে লেখা আছে, kernel-এ নয়।

এ থেকেই বোঝা যায় SUID প্রোগ্রাম কেন বিপজ্জনক। পুরো নিরাপত্তা নির্ভর করছে প্রোগ্রামটা কতটা সাবধানে লেখা তার ওপর। প্রোগ্রামে একটা বাগ মানেই কেউ root হয়ে যেতে পারে।

### নিজের চোখে দেখো

একটা টার্মিনালে `passwd` চালাও, পাসওয়ার্ড চাওয়ার জায়গায় থেমে থাকো। আরেকটা টার্মিনালে:

```
$ ps -o pid,ruser,euser,comm -C passwd
    PID RUSER    EUSER    COMM
   4821 nishat   root     passwd
```

RUSER `nishat`, কিন্তু EUSER `root`। প্রথম টার্মিনালে `Ctrl+C` চেপে `passwd` বন্ধ করে দাও।

### আরও কিছু চেনা SUID প্রোগ্রাম

| প্রোগ্রাম | কেন SUID লাগে |
|---|---|
| `/usr/bin/passwd` | `/etc/shadow` বদলাতে |
| `/usr/bin/sudo` | ইউজারকে root বা অন্য ইউজার হিসেবে কমান্ড চালাতে দিতে ([অধ্যায় ১৮](18-sudo-root.md)) |
| `/usr/bin/su` | অন্য ইউজারে switch করতে |
| `/usr/bin/chsh`, `/usr/bin/chfn` | `/etc/passwd`-এ নিজের shell বা নাম বদলাতে |
| `/usr/bin/newgrp`, `/usr/bin/gpasswd` | group সংক্রান্ত কাজে |
| `/usr/bin/mount`, `/usr/bin/umount` | `/etc/fstab`-এ `user` অপশন দেওয়া ডিস্ক সাধারণ ইউজারকে মাউন্ট করতে দিতে |
| `/usr/bin/fusermount3` | ইউজার-লেভেলে FUSE ফাইলসিস্টেম মাউন্ট করতে |
| `/usr/lib/dbus-1.0/dbus-daemon-launch-helper` | D-Bus সার্ভিস চালু করতে |

ঠিক কোন কোন ফাইল SUID, সেটা distro আর ইনস্টল করা প্যাকেজভেদে আলাদা হয়।

### SUID সেট করা ও তোলা

```
$ chmod u+s myprog        # symbolic
$ chmod 4755 myprog       # octal: 4 = SUID, 755 = rwxr-xr-x
$ chmod u-s myprog        # SUID তুলে দাও
```

### SUID-এর কিছু জরুরি নিয়ম

1. **শেল স্ক্রিপ্টে SUID কাজ করে না।** লিনাক্স kernel `#!` দিয়ে শুরু হওয়া স্ক্রিপ্টের (bash, python ইত্যাদি) SUID বিট উপেক্ষা করে। কারণ স্ক্রিপ্ট চালু হওয়া আর interpreter-এর ফাইলটা পড়ার মাঝে একটা race condition থাকে, তার ওপর environment variable দিয়েও interpreter-কে প্রভাবিত করা যায়। SUID শুধু compiled binary-তে (ELF) কাজ করে।
2. **ফাইলে কেউ লিখলে SUID চলে যায়।** root ছাড়া অন্য কেউ SUID ফাইলে write করলে kernel নিজে থেকেই SUID (আর group-execute থাকলে SGID) বিট মুছে দেয়। কেউ যাতে একটা SUID-root ফাইলের ভেতরে নিজের কোড বসাতে না পারে, সেজন্য এই ব্যবস্থা।
3. **`chown` করলেও SUID/SGID মুছে যায়।** owner বদলালে আগের SUID আর অর্থবহ থাকে না, তাই `chown` বিটটা পরিষ্কার করে দেয়। দরকার হলে `chown`-এর পরে আবার `chmod u+s` দাও।
4. **`nosuid` মাউন্ট অপশন।** কোনো ফাইলসিস্টেম `nosuid` দিয়ে মাউন্ট করা থাকলে ([অধ্যায় ৩২](32-disk-partition-mount.md)) সেখানে SUID/SGID কাজ করে না। USB ড্রাইভ, `/tmp`, `/home`-এর মতো জায়গার জন্য এটা ভালো অভ্যাস।
5. **ডিরেক্টরিতে SUID-এর কোনো মানে নেই।** লিনাক্সে ডিরেক্টরিতে SUID বিট দিলে কোনো কাজ হয় না।

> ⚠️ **সতর্কতা:** নিজের বানানো কোনো প্রোগ্রামে বা সাধারণ টুলে (`vim`, `find`, `python3`, `bash`, `cp`) কখনো SUID-root দেবে না। এতে সিস্টেমের প্রত্যেক ইউজার সরাসরি root হয়ে যেতে পারবে। ১৭.৮-এ উদাহরণসহ দেখবে।

---

## ১৭.৪ SGID (Set Group ID)

SGID-এর আচরণ দুই রকম: ফাইলে এক রকম, ডিরেক্টরিতে আরেক রকম।

- octal মান: **2** (যেমন `2755`)
- symbolic: `g+s`
- `ls -l`-এ group-এর execute ঘরে **`s`** দেখায়

### (ক) ফাইলে SGID

executable ফাইলে SGID থাকলে process-এর **Effective GID হয়ে যায় ফাইলের group**। SUID যা UID-এর জন্য করে, SGID সেটাই GID-এর জন্য করে।

উদাহরণ, `wall` (সব টার্মিনালে মেসেজ পাঠানোর কমান্ড):

```
$ ls -l /usr/bin/wall
-rwxr-sr-x 1 root tty 22912 Apr  9  2024 /usr/bin/wall
```

group-এর ঘরে `r-s`। টার্মিনাল ডিভাইসগুলো (`/dev/pts/*`) `tty` group-এর, আর ওই group-এর সেখানে write পারমিশন আছে:

```
$ ls -l /dev/pts/0
crw--w---- 1 nishat tty 136, 0 Nov 12 09:30 /dev/pts/0
```

তাই `wall` চালালে সে `tty` group-এর সদস্যের মতো আচরণ করে আর সবার টার্মিনালে লিখতে পারে, কিন্তু root হয় না। পুরো root ক্ষমতার বদলে শুধু একটা group-এর ক্ষমতা পায়। এটা **least privilege** (যতটুকু দরকার ঠিক ততটুকু ক্ষমতা) নীতির একটা ভালো উদাহরণ।

Debian/Ubuntu-তে `crontab` কমান্ডও এভাবে চলে:

```
$ ls -l /usr/bin/crontab
-rwxr-sr-x 1 root crontab 39664 Mar 31  2024 /usr/bin/crontab
```

এটা `crontab` group হয়ে `/var/spool/cron/crontabs/`-এ ইউজারের crontab ফাইল লেখে ([অধ্যায় ২৯](29-scheduling-cron.md))। Fedora/RHEL-এ `crontab` এর বদলে SUID-root।

### (খ) ডিরেক্টরিতে SGID: টিমের শেয়ার্ড ফোল্ডারের রহস্য

এটাই SGID-এর সবচেয়ে কাজের ব্যবহার।

**সমস্যা:** `/srv/project` ফোল্ডারে `dev` group-এর সবাই কাজ করে। alice একটা ফাইল বানালে লিনাক্সের সাধারণ নিয়মে ফাইলের group হয় alice-এর **primary group** (Ubuntu-তে সাধারণত `alice`)। ফলে bob (dev group-এর সদস্য) ফাইলটা group পারমিশন দিয়ে এডিট করতে পারে না।

**সমাধান:** ডিরেক্টরিতে SGID দাও। তখন:

1. ওই ডিরেক্টরিতে নতুন বানানো সব ফাইল আর সাব-ডিরেক্টরি **ডিরেক্টরির group** পায়, যে বানিয়েছে তার primary group নয়।
2. নতুন সাব-ডিরেক্টরিগুলো নিজেরাও SGID বিট পায়, তাই নিয়মটা গভীরে নামতে থাকে।

```
  SGID ছাড়া:                         SGID সহ (chmod g+s /srv/project):
  /srv/project  (group: dev)         /srv/project  (group: dev, drwxrwsr-x)
     └── alice.txt  group: alice ✗      └── alice.txt  group: dev ✔
     └── bob.txt    group: bob   ✗      └── bob.txt    group: dev ✔
                                        └── docs/      group: dev, SGID ✔
```

ধাপে ধাপে (root হিসেবে বা `sudo` দিয়ে):

```
$ sudo groupadd dev
$ sudo usermod -aG dev alice
$ sudo usermod -aG dev bob
$ sudo mkdir -p /srv/project
$ sudo chown root:dev /srv/project
$ sudo chmod 2775 /srv/project
$ ls -ld /srv/project
drwxrwsr-x 2 root dev 4096 Nov 12 10:00 /srv/project
```

এখন alice ফাইল বানালে:

```
$ touch /srv/project/plan.txt
$ ls -l /srv/project/plan.txt
-rw-rw-r-- 1 alice dev 0 Nov 12 10:05 /srv/project/plan.txt
```

group `dev`। bob-ও ফাইলটা এডিট করতে পারবে, তবে শর্ত হলো alice-এর `umask` `002` হতে হবে, যাতে ফাইলে group-write থাকে। Ubuntu-তে সাধারণ ইউজারের ডিফল্ট umask `002`, আর Fedora/RHEL-এও আধুনিক সংস্করণে ইউজারের নিজস্ব private group থাকলে `002`।

> 💡 **টিপস:** SGID ঠিক করে ফাইলের **group কোনটা হবে**। কিন্তু group-এর **পারমিশন কী হবে** সেটা ঠিক করে umask। ভরসাযোগ্যভাবে টিম-শেয়ারিং চাইলে SGID-এর সাথে default ACL (১৭.৯) ব্যবহার করো। ACL umask-এর ওপর নির্ভর করে না।

> 📝 **মনে রাখো:** group-এর সদস্যপদ বদলালে (`usermod -aG`) সেটা চালু হতে ইউজারকে log out করে আবার login করতে হয়, অথবা `newgrp dev` চালাতে হয়। `id` কমান্ড দিয়ে চেক করো ([অধ্যায় ১৫](15-users-groups.md))।

---

## ১৭.৫ Sticky Bit

### সমস্যাটা

`/tmp` এমন জায়গা যেখানে সব ইউজার আর সব প্রোগ্রাম অস্থায়ী ফাইল রাখে। তাই এখানে সবার লেখার অনুমতি থাকতে হয় (`rwx` for others)। কিন্তু অধ্যায় ১৬-এর নিয়ম মনে করো: **কোনো ফাইল মোছা বা rename করার অনুমতি নির্ভর করে ডিরেক্টরির write পারমিশনের ওপর, ফাইলের পারমিশনের ওপর নয়।** তাহলে তো যে কেউ যে কারও ফাইল মুছে দিতে পারত!

### সমাধান: sticky bit

ডিরেক্টরিতে sticky bit থাকলে সেখানকার কোনো ফাইল **মুছতে বা rename করতে** পারে শুধু:

- ফাইলের owner, অথবা
- ডিরেক্টরির owner, অথবা
- root (আরও নিখুঁতভাবে বললে, `CAP_FOWNER` ক্ষমতা আছে এমন process)

```
$ ls -ld /tmp
drwxrwxrwt 18 root root 4096 Nov 12 10:10 /tmp
```

others-এর ঘরে শেষ অক্ষর `t`। মোড `1777`।

- octal মান: **1**
- symbolic: `o+t` বা শুধু `+t`
- `ls -l`-এ others-এর execute ঘরে **`t`**

**উপমা:** অফিসের কমন ফ্রিজ। সবাই খাবার রাখতে পারে (write)। কিন্তু নিয়ম আছে, নিজের টিফিন বাক্স ছাড়া অন্যেরটা ফেলে দেওয়া যাবে না। ফ্রিজের মালিক (ডিরেক্টরির owner) আর অফিস ম্যানেজার (root) অবশ্য সব পরিষ্কার করতে পারে।

### পরীক্ষা

```
alice$ echo "secret" > /tmp/alice.txt
alice$ chmod 666 /tmp/alice.txt          # সবাই লিখতে পারে এমন ফাইল

bob$ echo "hacked" >> /tmp/alice.txt      # ফাইলে লেখা গেল (ফাইলের পারমিশন 666)
bob$ rm /tmp/alice.txt
rm: cannot remove '/tmp/alice.txt': Operation not permitted
```

এখানে খেয়াল করার মতো একটা ব্যাপার আছে। sticky bit **ফাইলের ভেতরের লেখা রক্ষা করে না**, শুধু মোছা আর rename আটকায়। কন্টেন্ট রক্ষা হয় ফাইলের নিজের পারমিশন দিয়ে। আবার আধুনিক কার্নেলে `fs.protected_regular` চালু থাকলে (systemd-ভিত্তিক distro-তে সাধারণত থাকে) উপরের `>>` লাইনটাও sticky world-writable ডিরেক্টরিতে আটকে যেতে পারে। নিচে দেখো।

### কার্নেলের বাড়তি সুরক্ষা

sticky world-writable ডিরেক্টরিতে (যেমন `/tmp`) কিছু পুরোনো আক্রমণ ঠেকাতে কার্নেলের কয়েকটা sysctl আছে:

```
$ sysctl fs.protected_symlinks fs.protected_hardlinks fs.protected_regular
fs.protected_symlinks = 1
fs.protected_hardlinks = 1
fs.protected_regular = 2
```

| sysctl | কী করে |
|---|---|
| `fs.protected_symlinks` | sticky world-writable ডিরেক্টরিতে অন্য কারও বানানো symlink অনুসরণ করতে দেয় না (কিছু শর্ত সাপেক্ষে) |
| `fs.protected_hardlinks` | যে ফাইল তোমার নয় আর যেটায় তোমার read/write অনুমতি নেই, তার hard link বানাতে দেয় না |
| `fs.protected_regular` | sticky ডিরেক্টরিতে অন্যের আগে থেকে থাকা ফাইল `O_CREAT` দিয়ে খোলা আটকায় |

মানগুলো distro-ভেদে আলাদা হতে পারে। বিস্তারিত [অধ্যায় ৪৫](45-security-hardening.md) আর [অধ্যায় ৪৭](47-kernel-proc-sys.md)-এ।

### ফাইলে sticky bit?

একেবারে পুরোনো UNIX-এ executable ফাইলে sticky bit দিলে প্রোগ্রামটা swap-এ "লেগে" (sticky) থাকত, তাই এই নাম। **আধুনিক লিনাক্স ফাইলের sticky bit পুরোপুরি উপেক্ষা করে।** এর ব্যবহার এখন শুধু ডিরেক্টরিতে।

```
$ chmod +t /srv/shared        # sticky দাও
$ chmod 1777 /srv/shared      # /tmp-এর মতো: সবাই লিখবে, কিন্তু নিজেরটা ছাড়া মুছবে না
$ chmod -t /srv/shared        # তুলে দাও
```

`/var/tmp`, আর কখনো কখনো `/dev/shm` আর `/dev/mqueue`-ও sticky বিট নিয়ে থাকে।

---

## ১৭.৬ চার অঙ্কের octal মোড: 4/2/1

তিনটা বিশেষ বিটের মানও rwx-এর মতো 4/2/1:

| বিট | মান | symbolic | কোথায় দেখায় |
|---|---|---|---|
| SUID | 4 | `u+s` | owner-এর x ঘরে `s`/`S` |
| SGID | 2 | `g+s` | group-এর x ঘরে `s`/`S` |
| Sticky | 1 | `o+t` / `+t` | others-এর x ঘরে `t`/`T` |

প্রথম অঙ্কটা এই মানগুলোর যোগফল:

| মোড | মানে | বাস্তব উদাহরণ |
|---|---|---|
| `0755` | কোনো বিশেষ বিট নেই | সাধারণ প্রোগ্রাম |
| `4755` | SUID + rwxr-xr-x | `/usr/bin/passwd` |
| `2755` | SGID + rwxr-xr-x | `/usr/bin/wall` |
| `2775` | SGID + rwxrwxr-x | টিমের শেয়ার্ড ডিরেক্টরি |
| `1777` | sticky + rwxrwxrwx | `/tmp` |
| `3775` | SGID + sticky (2+1) | টিম ফোল্ডার যেখানে একে অন্যের ফাইল মুছতে পারবে না |
| `6755` | SUID + SGID (4+2) | দুটোই দরকার এমন বিরল প্রোগ্রাম |

```
         chmod 2775 /srv/project
                │ │││
                │ ││└─ others = 5 = r-x
                │ │└── group  = 7 = rwx
                │ └─── owner  = 7 = rwx
                └───── বিশেষ  = 2 = SGID
```

`stat` দিয়ে চার অঙ্কেই মোড দেখা যায়:

```
$ stat -c '%a %A %U:%G %n' /usr/bin/passwd /tmp /srv/project
4755 -rwsr-xr-x root:root /usr/bin/passwd
1777 drwxrwxrwt root:root /tmp
2775 drwxrwsr-x root:dev /srv/project
```

### ডিরেক্টরিতে numeric chmod-এর একটা ফাঁদ

GNU `chmod` (লিনাক্সে যেটা থাকে) ডিরেক্টরির SUID/SGID বিট রক্ষা করে, যদি না তুমি স্পষ্ট করে সেগুলো বদলাতে বলো। তাই:

```
$ ls -ld /srv/project
drwxrwsr-x 2 root dev 4096 Nov 12 10:00 /srv/project
$ sudo chmod 755 /srv/project
$ ls -ld /srv/project
drwxr-sr-x 2 root dev 4096 Nov 12 10:00 /srv/project      # SGID রয়ে গেছে!
```

SGID তুলতে চাইলে symbolic মোড সবচেয়ে নিরাপদ:

```
$ sudo chmod g-s /srv/project
```

numeric দিয়ে করতে চাইলে GNU coreutils-এর ডকুমেন্টেশন অনুযায়ী বাড়তি একটা শূন্য লাগে, যেমন `chmod 00755`। sticky বিট এই নিয়মের বাইরে, `chmod 755` দিলে সেটা উঠে যায়।

---

## ১৭.৭ `s` বনাম `S`, `t` বনাম `T`: বড় হাতের অক্ষর মানে সমস্যা

`ls -l` একটা ঘরে দুটো তথ্য দেখায়, execute বিট আছে কি না আর বিশেষ বিট আছে কি না:

| ঘর | execute আছে | execute নেই |
|---|---|---|
| owner, SUID সহ | `s` | `S` |
| group, SGID সহ | `s` | `S` |
| others, sticky সহ | `t` | `T` |
| বিশেষ বিট নেই | `x` | `-` |

```
$ touch demo
$ chmod 4644 demo
$ ls -l demo
-rwSr--r-- 1 nishat nishat 0 Nov 12 10:20 demo
```

বড় হাতের `S` মানে SUID দেওয়া আছে, কিন্তু owner-এর execute বিট নেই। ফাইলটা চালানোই যায় না, তাই SUID এখানে অর্থহীন। এটা সাধারণত ভুলের লক্ষণ।

```
$ chmod u+x demo
$ ls -l demo
-rwsr--r-- 1 nishat nishat 0 Nov 12 10:20 demo
```

একইভাবে `T` মানে sticky দেওয়া আছে, কিন্তু others-এর execute নেই। ডিরেক্টরিতে execute না থাকলে others ভেতরে ঢুকতেই পারে না, তাই এটাও প্রায় সবসময় ভুল।

একটা ব্যতিক্রম: **ডিরেক্টরিতে `S` (SGID, group execute ছাড়া)** দেখলে সেটা পুরো অর্থহীন নয়। নতুন ফাইলের group inherit করার নিয়ম তখনও খাটে। তবে group-এর সদস্যরা ভেতরে ঢুকতে পারবে না, তাই বাস্তবে এর ব্যবহার নেই। আর ফাইলে group-execute ছাড়া SGID দিলে কিছু পুরোনো সিস্টেমে সেটা mandatory locking বোঝাত। আধুনিক লিনাক্স কার্নেল (5.15 থেকে) mandatory locking সমর্থন বন্ধ করে দিয়েছে।

> 💡 **টিপস:** ছোট হাতের `s`/`t` = স্বাভাবিক। বড় হাতের `S`/`T` = "কোথাও গড়বড় আছে, চেক করো।"

---

## ১৭.৮ SUID/SGID ফাইল খোঁজা ও নিরাপত্তা ঝুঁকি

### কেন খুঁজবে

প্রতিটি SUID-root প্রোগ্রাম সম্ভাব্য একটা দরজা। হ্যাকার সাধারণ ইউজার হিসেবে সিস্টেমে ঢুকলে প্রথম যে কাজগুলো করে, তার একটা হলো SUID ফাইলের তালিকা বের করা (**privilege escalation**, মানে ক্ষমতা বাড়ানোর চেষ্টা)। তাই সিস্টেম অ্যাডমিন হিসেবে তোমারও জানা থাকা দরকার কোন কোন ফাইল SUID, আর তালিকায় হঠাৎ নতুন কিছু যোগ হলো কি না।

### `find` দিয়ে খোঁজা

`find`-এর `-perm` অপশন ([অধ্যায় ১২](12-file-khoja.md)) দিয়ে কাজটা হয়:

| এক্সপ্রেশন | মানে |
|---|---|
| `-perm -4000` | SUID বিট **আছে** (অন্য বিট যা-ই থাকুক) |
| `-perm -2000` | SGID বিট আছে |
| `-perm -1000` | sticky বিট আছে |
| `-perm /6000` | SUID **অথবা** SGID, যেকোনো একটা আছে |
| `-perm -6000` | SUID **এবং** SGID দুটোই আছে |
| `-perm 4755` | মোড **হুবহু** 4755 (`-` বা `/` ছাড়া মানে exact match) |

```
$ sudo find / -xdev -type f -perm -4000 2>/dev/null
/usr/bin/passwd
/usr/bin/su
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/umount
/usr/bin/fusermount3
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
...
```

- `-xdev`: অন্য ফাইলসিস্টেমে (যেমন `/proc`, নেটওয়ার্ক ড্রাইভ) ঢুকবে না। প্রতিটি পার্টিশন আলাদা করে চেক করতে চাইলে এটা বাদ দাও।
- `2>/dev/null`: "Permission denied" জাতীয় error লুকায় ([অধ্যায় ১৯](19-redirection-pipe.md))।

বিস্তারিত তালিকা দেখতে:

```
$ sudo find / -xdev -type f -perm /6000 -exec ls -l {} + 2>/dev/null
-rwxr-sr-x 1 root shadow   31312 Apr  5  2024 /usr/bin/chage
-rwsr-xr-x 1 root root     72792 May 30  2024 /usr/bin/chfn
-rwxr-sr-x 1 root crontab  39664 Mar 31  2024 /usr/bin/crontab
...
```

sticky ডিরেক্টরিগুলো দেখতে:

```
$ find / -xdev -type d -perm -1000 2>/dev/null
/tmp
/var/tmp
...
```

### একটা baseline রাখো

ভালো অভ্যাস হলো একবার তালিকা বানিয়ে রাখা, তারপর মাঝে মাঝে মিলিয়ে দেখা:

```
$ sudo find / -xdev -type f -perm /6000 2>/dev/null | sort > ~/suid-baseline.txt
# কয়েক দিন পর:
$ sudo find / -xdev -type f -perm /6000 2>/dev/null | sort | diff ~/suid-baseline.txt -
> /home/bob/.cache/bash
```

`>` দিয়ে শুরু হওয়া লাইনগুলো নতুন। কারও home ডিরেক্টরিতে SUID-root `bash` পাওয়া মানে প্রায় নিশ্চিতভাবে সিস্টেম আক্রান্ত হয়েছে। প্যাকেজ আপডেটের পরেও তালিকা বদলাতে পারে, সেটা স্বাভাবিক।

> 💡 **টিপস:** `rkhunter`, `aide`, `lynis`-এর মতো টুল এই কাজ স্বয়ংক্রিয়ভাবে করে ([অধ্যায় ৪৫](45-security-hardening.md))। কোনো ফাইল কোন প্যাকেজের, সেটা দেখতে Debian/Ubuntu-তে `dpkg -S /usr/bin/passwd`, Fedora/RHEL-এ `rpm -qf /usr/bin/passwd` ([অধ্যায় ৩০](30-package-management.md))। কোনো প্যাকেজেরই নয় এমন SUID ফাইল সন্দেহজনক।

### ঝুঁকি কোথায়: একটা বাস্তব উদাহরণ

ধরো কোনো অ্যাডমিন "সুবিধার জন্য" `find`-এ SUID দিয়ে দিয়েছে:

```
$ ls -l /usr/bin/find
-rwsr-xr-x 1 root root 224848 Apr  8  2024 /usr/bin/find
```

`find`-এর `-exec` অপশন যেকোনো কমান্ড চালাতে পারে। আর সেই কমান্ড চলবে EUID=0 নিয়ে:

```
nishat$ find . -exec /bin/sh -p \; -quit
# whoami
root
```

`-p` অপশন না দিলে `bash` আর `dash` বুঝতে পারে যে RUID আর EUID আলাদা, আর নিজে থেকে ক্ষমতা ছেড়ে দেয়। `-p` (privileged mode) সেই সুরক্ষা বন্ধ করে। এই একটা ভুলেই যেকোনো ইউজার root।

এ রকম শত শত প্রোগ্রাম আছে (`vim`, `less`, `python3`, `perl`, `awk`, `tar`, `cp`, `env` …) যেগুলোতে SUID থাকলে root পাওয়া যায়। নিরাপত্তা গবেষকরা এগুলোর তালিকা রাখেন। সবচেয়ে পরিচিত তালিকা হলো **GTFOBins** প্রজেক্ট।

### ঝুঁকি কমানোর নিয়ম

1. যে প্রোগ্রাম ব্যবহার করো না, তার SUID তুলে দাও। (তবে প্যাকেজ আপডেট হলে বিটটা আবার ফিরে আসতে পারে। Debian/Ubuntu-তে স্থায়ীভাবে বদলাতে `dpkg-statoverride` ব্যবহার করা হয়।)
2. সাধারণ টুলে কখনো SUID দেবে না। কাউকে নির্দিষ্ট একটা কমান্ড root হিসেবে চালাতে দিতে চাইলে `sudo`-র নিয়ম লেখো ([অধ্যায় ১৮](18-sudo-root.md))।
3. পুরো root-এর বদলে শুধু একটা ক্ষমতা দরকার হলে **capabilities** ব্যবহার করো (১৭.১১)।
4. ইউজাররা লিখতে পারে এমন পার্টিশন (`/home`, `/tmp`, USB) `nosuid` দিয়ে মাউন্ট করো।
5. SUID ফাইলের তালিকা নিয়মিত মিলিয়ে দেখো।

---

## ১৭.৯ ACL (Access Control List): owner/group/others-এর বাইরে

### সমস্যা

`report.txt` ফাইলটা তোমার (`nishat`), group `nishat`, মোড `640`। এখন:

- alice-কে পড়তে আর লিখতে দিতে চাও,
- bob-কে শুধু পড়তে দিতে চাও,
- আর কাউকে কিছুই না।

সাধারণ পারমিশনে মাত্র তিনটা ঘর (owner, একটা group, others)। এই কাজের জন্য নতুন group বানাতে হতো, আর প্রতিটি ফাইলের জন্য আলাদা মিশ্রণ চাইলে group-এর সংখ্যা বাড়তেই থাকত। এর সমাধান **ACL**: প্রতিটি ফাইলে যত খুশি ইউজার আর group-এর জন্য আলাদা আলাদা নিয়ম।

**উপমা:** সাধারণ পারমিশন যেন বাড়ির তিনটা চাবি: একটা মালিকের, একটা পরিবারের, একটা বাকি সবার। ACL হলো দারোয়ানের হাতে একটা লিস্ট: "রহিম ভাই ঢুকতে পারবে, করিম শুধু বসার ঘর পর্যন্ত।"

### ভেতরে কোথায় থাকে

ACL inode-এর ১২টি বিটে আঁটে না। তাই এটা থাকে ফাইলের **extended attribute (xattr)**-এ, `system.posix_acl_access` আর (ডিরেক্টরির জন্য) `system.posix_acl_default` নামে। ext4, XFS, Btrfs, আর আধুনিক distro-র বেশিরভাগ ফাইলসিস্টেমে ACL ডিফল্টভাবে চালু থাকে। FAT/exFAT-এ ACL নেই।

`getfacl`/`setfacl` আসে `acl` প্যাকেজে। সাধারণত আগে থেকেই ইনস্টল থাকে, না থাকলে:

```
$ sudo apt install acl        # Debian/Ubuntu
$ sudo dnf install acl        # Fedora/RHEL
```

### ACL entry-র ধরন

| entry | লেখার ধরন | মানে |
|---|---|---|
| owner | `user::rw-` | ফাইলের owner (সাধারণ owner বিটের সমান) |
| named user | `user:alice:rw-` | নির্দিষ্ট একজন ইউজার |
| owning group | `group::r--` | ফাইলের group |
| named group | `group:dev:r-x` | নির্দিষ্ট একটা group |
| mask | `mask::rw-` | named user, named group আর owning group সর্বোচ্চ কতটা পাবে তার সীমা |
| others | `other::---` | বাকি সবাই |

### `getfacl`: ACL দেখা

ACL ছাড়া একটা ফাইল:

```
$ getfacl report.txt
# file: report.txt
# owner: nishat
# group: nishat
user::rw-
group::r--
other::---
```

এটা আসলে সাধারণ `640` পারমিশনকেই ACL-এর ভাষায় দেখানো।

### `setfacl -m`: entry যোগ বা বদল (modify)

```
$ setfacl -m u:alice:rw report.txt
$ setfacl -m u:bob:r report.txt
$ getfacl report.txt
# file: report.txt
# owner: nishat
# group: nishat
user::rw-
user:alice:rw-
user:bob:r--
group::r--
mask::rw-
other::---
```

`u` = user, `g` = group, `m` = mask, `o` = other (পুরো শব্দ `user:`, `group:` লিখলেও চলে)। একসাথে একাধিক entry দিতে কমা দাও:

```
$ setfacl -m u:alice:rw,u:bob:r,g:dev:r report.txt
```

এখন `ls -l` দেখো:

```
$ ls -l report.txt
-rw-rw----+ 1 nishat nishat 120 Nov 12 11:00 report.txt
```

দুটো জিনিস খেয়াল করো:

1. শেষে **`+`** চিহ্ন। মানে এই ফাইলে ACL আছে, শুধু `ls -l` দেখে পুরো পারমিশন বোঝা যাবে না।
2. group-এর ঘরে এখন `rw-`, যদিও `group::` entry `r--`। **ACL থাকলে `ls -l`-এর group ঘর mask দেখায়**, owning group-এর পারমিশন নয়। এটা নতুনদের সবচেয়ে বড় বিভ্রান্তি।

### mask: ACL-এর ছাঁকনি

mask হলো একটা উচ্চসীমা। named user, named group আর owning group, এদের প্রত্যেকের **কার্যকর (effective) পারমিশন = তার entry ∩ mask** (দুটোতেই যে বিট আছে, শুধু সেটুকু)। owner আর others-এর ওপর mask-এর প্রভাব নেই।

```
           entry       mask       effective
  alice:   rw-    ∩    r--    =   r--
  dev:     rwx    ∩    r--    =   r--
  owner:   rw-    (mask প্রযোজ্য নয়)  = rw-
  other:   ---    (mask প্রযোজ্য নয়)  = ---
```

```
$ setfacl -m m::r report.txt
$ getfacl report.txt
# file: report.txt
# owner: nishat
# group: nishat
user::rw-
user:alice:rw-                  #effective:r--
user:bob:r--
group::r--
mask::r--
other::---
```

`#effective:` মন্তব্য বলছে alice-এর entry-তে `rw-` লেখা থাকলেও সে এখন শুধু পড়তে পারবে।

mask নিয়ে তিনটা নিয়ম মনে রাখো:

- `setfacl -m` দিয়ে কিছু বদলালে mask **নিজে থেকে আবার হিসাব হয়**। সব named user, named group আর owning group-এর পারমিশন মিলিয়ে (union) নতুন mask বসে। তাই উপরে mask ম্যানুয়ালি `r` করার পর আবার কোনো `-m` চালালে mask আবার বড় হয়ে যেতে পারে। এটা চাইলে `-n` (`--no-mask`) অপশন দাও।
- ACL থাকা ফাইলে **`chmod g=...` দিলে mask বদলায়**, owning group-এর entry নয়। যেমন `chmod g-w report.txt` সব named entry থেকে write কার্যকরভাবে কেড়ে নেয়।
- তাই ACL থাকা ফাইলে `chmod 600` চালালে mask `---` হয়ে যায়, আর সব named user-এর প্রবেশ কার্যকরভাবে বন্ধ হয়ে যায়।

> ⚠️ **সতর্কতা:** ACL থাকা ফাইলে অভ্যাসবশে `chmod 644` বা `chmod 600` চালিয়ে দিলে ACL-এর পুরো হিসাব চুপচাপ বদলে যেতে পারে। ফাইলের শেষে `+` দেখলে আগে `getfacl` দেখো।

### kernel কীভাবে ACL চেক করে

```
  process ফাইল খুলতে চাইল
        │
        ▼
  EUID = owner? ──হ্যাঁ──► user:: entry ব্যবহার (mask নয়)
        │না
        ▼
  কোনো user:NAME: entry মেলে? ──হ্যাঁ──► সেই entry ∩ mask
        │না
        ▼
  process-এর কোনো group owning group বা কোনো group:NAME:-এর সাথে মেলে?
        │ হ্যাঁ ──► মেলা entry-গুলোর যেকোনো একটা (∩ mask) অনুমতি দিলে ✔, নইলে ✘
        │না
        ▼
  other:: entry ব্যবহার
```

প্রথম যে ধাপ মেলে, সিদ্ধান্ত সেখানেই হয়, পরের ধাপে যায় না। তাই কারও `user:` entry `---` হলে, তার group-এর অনুমতি থাকলেও সে ঢুকতে পারবে না।

### `setfacl -x`: নির্দিষ্ট entry মুছে ফেলা

```
$ setfacl -x u:bob report.txt
$ setfacl -x g:dev report.txt
```

`-x`-এ পারমিশন লিখতে হয় না, শুধু কার entry সেটা।

### `setfacl -b`: সব ACL মুছে ফেলা

```
$ setfacl -b report.txt
$ ls -l report.txt
-rw-r-----  1 nishat nishat 120 Nov 12 11:00 report.txt
```

`+` চলে গেছে। ফাইল আবার সাধারণ পারমিশনে ফিরেছে। (`-b` = remove all extended ACL entries; owner/group/other থেকে যায়।)

### Default ACL (`-d`): ডিরেক্টরিতে উত্তরাধিকার

ডিরেক্টরিতে **default ACL** দিলে ভেতরে নতুন বানানো প্রতিটি ফাইল আর সাব-ডিরেক্টরি সেই ACL নিজের ACL হিসেবে পায়। SGID যেমন group inherit করায়, default ACL তেমনি পারমিশন inherit করায়।

```
$ sudo setfacl -d -m g:dev:rwX /srv/project
$ sudo setfacl -m g:dev:rwX /srv/project       # ডিরেক্টরির নিজের জন্যও
$ getfacl /srv/project
# file: srv/project
# owner: root
# group: dev
# flags: -s-
user::rwx
group::rwx
group:dev:rwx
mask::rwx
other::r-x
default:user::rwx
default:group::rwx
default:group:dev:rwx
default:mask::rwx
default:other::r-x
```

- `default:` দিয়ে শুরু হওয়া লাইনগুলো default ACL।
- `# flags: -s-` মানে ডিরেক্টরিতে SGID আছে (তিনটা অক্ষর SUID, SGID, sticky ক্রমে)।
- getfacl লেখার সময় path-এর শুরুর `/` বাদ দেয়, তাই `srv/project` দেখাচ্ছে।
- **বড় হাতের `X`**: execute দেবে শুধু ডিরেক্টরিতে, অথবা কোনো ফাইলে আগে থেকেই কারও execute থাকলে। তাই সাধারণ টেক্সট ফাইল ভুল করে executable হয়ে যায় না।

এখন যে কেউ এখানে ফাইল বানালে:

```
alice$ touch /srv/project/notes.txt
alice$ getfacl /srv/project/notes.txt
# file: srv/project/notes.txt
# owner: alice
# group: dev
user::rw-
group::rwx                      #effective:rw-
group:dev:rwx                   #effective:rw-
mask::rw-
other::r--
```

`touch` ফাইলটা `rw-` দিয়ে বানানোর অনুরোধ করে (mode 666), তাই default ACL-এর `x`, mask দিয়ে ছেঁকে বাদ পড়েছে। মূল কথা: **default ACL থাকলে umask ব্যবহার হয় না**। পারমিশন আসে default ACL আর প্রোগ্রাম যে মোড চেয়েছে, এই দুটো থেকে।

default ACL মুছতে:

```
$ sudo setfacl -k /srv/project       # শুধু default ACL মুছবে
```

### `-R`: রিকার্সিভ

আগে থেকে থাকা সব ফাইলে একসাথে ACL দিতে:

```
$ sudo setfacl -R -m g:dev:rwX /srv/project
```

`-R` শুধু **এখন যা আছে** তাতে ACL দেয়। ভবিষ্যতের ফাইলের জন্য `-d` লাগবে। তাই দুটো সাধারণত একসাথে চলে।

### ACL-এর ব্যাকআপ ও রিস্টোর

```
$ getfacl -R /srv/project > project-acl.txt       # পুরো ট্রির ACL টেক্সট ফাইলে
$ sudo setfacl --restore=project-acl.txt          # হুবহু ফিরিয়ে আনো
```

একটা ফাইলের ACL আরেকটায় কপি করতে:

```
$ getfacl source.txt | setfacl --set-file=- target.txt
```

### ফাইল কপি করলে ACL কি থাকে?

| কমান্ড | ACL রাখে? |
|---|---|
| `cp file new` | না, সাধারণত হারায় |
| `cp -a` বা `cp --preserve=all` | হ্যাঁ |
| `mv` (একই ফাইলসিস্টেমে) | হ্যাঁ (inode একই থাকে) |
| `tar` | `--acls` দিলে ([অধ্যায় ৩৪](34-archive-compression.md)) |
| `rsync` | `-A` দিলে ([অধ্যায় ৩৫](35-backup-rsync.md)) |

### `setfacl`-এর অপশন সারণি

| অপশন | কাজ |
|---|---|
| `-m spec` | entry যোগ বা বদল |
| `-x spec` | নির্দিষ্ট entry মুছে ফেলা |
| `-b` | সব extended ACL মুছে ফেলা |
| `-k` | শুধু default ACL মুছে ফেলা |
| `-d` | পরের `-m`/`-x` default ACL-এ প্রযোজ্য হবে |
| `-R` | রিকার্সিভ |
| `-n` | mask নতুন করে হিসাব করবে না |
| `--set=spec` | পুরো ACL বদলে দেবে (আগেরটা মুছে) |
| `--restore=file` | `getfacl -R`-এর আউটপুট থেকে ফিরিয়ে আনবে |
| `--test` | আসলে বদলাবে না, ফলাফল কী হবে শুধু দেখাবে |

> 💡 **টিপস:** কোন ফাইলগুলোতে ACL আছে খুঁজতে `getfacl -R -s -p /srv 2>/dev/null | grep '^# file'`। এখানে `-s` (skip base) শুধু ACL থাকা ফাইল দেখায়।

---

## ১৭.১০ ফাইল অ্যাট্রিবিউট: `chattr` ও `lsattr`

পারমিশন আর ACL বলে "**কে** কী করতে পারবে"। ফাইল অ্যাট্রিবিউট বলে "**কেউই** কী করতে পারবে না"। root-ও না, যতক্ষণ না অ্যাট্রিবিউটটা তুলে নেওয়া হয়।

এগুলো ফাইলসিস্টেম-লেভেলের ফ্ল্যাগ। মূলত ext2/3/4-এর জন্য তৈরি। XFS আর Btrfs-ও কিছু অ্যাট্রিবিউট (যেমন `i`, `a`) সমর্থন করে, তবে সবগুলো নয়। টুল দুটো আসে `e2fsprogs` প্যাকেজে, যেটা প্রায় সব distro-তে ইনস্টল থাকে।

### `lsattr`: অ্যাট্রিবিউট দেখা

```
$ lsattr notes.txt
--------------e------- notes.txt
```

ext4-এ `e` (extents) প্রায় সব ফাইলেই দেখবে। এটা ফাইলসিস্টেম নিজে বসায়, এ নিয়ে মাথা ঘামানোর দরকার নেই। ড্যাশের সংখ্যা `e2fsprogs`-এর version-ভেদে আলাদা হতে পারে।

| অপশন | কাজ |
|---|---|
| `lsattr file` | ফাইলের অ্যাট্রিবিউট |
| `lsattr -d dir` | ডিরেক্টরির নিজের অ্যাট্রিবিউট (ভেতরের ফাইলের নয়) |
| `lsattr -R dir` | রিকার্সিভ |
| `lsattr -a` | লুকানো (`.` দিয়ে শুরু) ফাইলও |
| `lsattr -l` | অক্ষরের বদলে পুরো নাম দেখাও |

### `chattr`: অ্যাট্রিবিউট বসানো ও তোলা

```
chattr [+|-|=]অ্যাট্রিবিউট ফাইল...
```

`+` যোগ করে, `-` তুলে নেয়, `=` হুবহু ওইটুকু বসায়। `-R` দিলে রিকার্সিভ। `i` আর `a` বসানো বা তোলার জন্য root লাগে (আরও নিখুঁতভাবে, `CAP_LINUX_IMMUTABLE` ক্ষমতা)।

### `+i`: Immutable (অপরিবর্তনীয়)

```
$ sudo chattr +i important.conf
$ lsattr important.conf
----i---------e------- important.conf

$ sudo rm important.conf
rm: cannot remove 'important.conf': Operation not permitted
$ sudo sh -c 'echo test >> important.conf'
sh: 1: cannot create important.conf: Operation not permitted
$ sudo mv important.conf x.conf
mv: cannot move 'important.conf' to 'x.conf': Operation not permitted
$ sudo chmod 777 important.conf
chmod: changing permissions of 'important.conf': Operation not permitted
```

immutable ফাইলে:
- লেখা যায় না, মোছা যায় না, rename করা যায় না
- hard link বানানো যায় না
- পারমিশন, owner, ACL বদলানো যায় না
- root-ও পারে না, যতক্ষণ না `chattr -i` দেওয়া হয়

ডিরেক্টরিতে `+i` দিলে ভেতরে নতুন ফাইল বানানো, মোছা বা rename করা যায় না। তবে ভেতরের আগে থেকে থাকা ফাইলের কন্টেন্ট বদলানো যায় (ফাইলগুলোর নিজের `+i` না থাকলে)।

তুলে নিতে:

```
$ sudo chattr -i important.conf
```

**কোথায় কাজে লাগে:** গুরুত্বপূর্ণ কনফিগ ফাইল (যেমন হাতে লেখা `/etc/resolv.conf`, যেটা অন্য কোনো সার্ভিস যেন বদলে না দেয়), ভুল করে `rm -rf` থেকে কিছু বাঁচানো।

> ⚠️ **সতর্কতা:** হ্যাকাররাও `chattr +i` ব্যবহার করে, যাতে তাদের বসানো ফাইল (যেমন `~/.ssh/authorized_keys`-এ ঢোকানো key) অ্যাডমিন মুছতে না পারে। কোনো ফাইল root হয়েও মুছতে না পারলে প্রথমে `lsattr` দেখো। আর মনে রেখো, immutable ফাইল প্যাকেজ আপডেটকেও আটকায়। `/etc`-এর কোনো ফাইলে `+i` দিয়ে ভুলে গেলে পরে `apt`/`dnf` আপডেট ব্যর্থ হতে পারে।

### `+a`: Append-only (শুধু শেষে যোগ করা যাবে)

```
$ sudo chattr +a audit.log
$ echo "new entry" | sudo tee -a audit.log      # >> (append) কাজ করে
new entry
$ echo "overwrite" | sudo tee audit.log          # > (overwrite) কাজ করে না
tee: audit.log: Operation not permitted
$ sudo rm audit.log
rm: cannot remove 'audit.log': Operation not permitted
```

আগের লেখা মুছে ফেলা বা বদলানো যায় না, শুধু শেষে নতুন লাইন যোগ করা যায়। **লগ ফাইলের জন্য আদর্শ।** কোনো আক্রমণকারী root পেলেও নিজের চিহ্ন মুছতে পারবে না, যদি না আগে `chattr -a` চালায়। তবে লগ rotate করার টুলের (logrotate, [অধ্যায় ২৮](28-logs.md)) সাথে এটা ঝামেলা করতে পারে, তাই বুঝে ব্যবহার করো।

### অন্য অ্যাট্রিবিউটগুলো

| অ্যাট্রিবিউট | নাম | কাজ | মন্তব্য |
|---|---|---|---|
| `i` | immutable | কোনো পরিবর্তন নয় | root লাগে |
| `a` | append only | শুধু শেষে যোগ | root লাগে |
| `A` | no atime | access time আপডেট হবে না | disk I/O কমায় |
| `d` | no dump | পুরোনো `dump` ব্যাকআপ টুল এই ফাইল বাদ দেবে | |
| `S` | synchronous | প্রতিটি লেখা সাথে সাথে ডিস্কে যাবে | ধীর, কিন্তু নিরাপদ |
| `C` | no copy-on-write | Btrfs-এ CoW বন্ধ করে | VM image ও ডাটাবেস ফাইলে কাজে লাগে; খালি ফাইল বা ডিরেক্টরিতে দিতে হয় |
| `c` | compressed | ফাইলটা কম্প্রেস করে রাখবে | Btrfs-এ কাজ করে, ext4-এ নয় |
| `e` | extents | ফাইল extents ব্যবহার করে | ext4 নিজে বসায়; হাতে বদলানোর দরকার নেই |
| `s`, `u` | secure delete, undelete | | **ext4 এগুলো মানে না**। ম্যান পেজেও এটা লেখা আছে, এদের ওপর ভরসা কোরো না |

---

## ১৭.১১ Linux Capabilities: root-এর ক্ষমতা টুকরো টুকরো করা

### সমস্যা

পুরোনো UNIX মডেলে দুই রকম process: root (UID 0, সব পারে) আর বাকিরা (প্রায় কিছুই পারে না বিশেষ)। কিন্তু অনেক প্রোগ্রামের দরকার মাত্র **একটা** বিশেষ ক্ষমতা:

- `ping`-এর দরকার raw network socket খোলা
- একটা ওয়েব সার্ভারের দরকার ১০২৪-এর নিচের পোর্ট (৮০, ৪৪৩) bind করা

এর জন্য পুরো root (SUID-root) দেওয়া মানে একটা ছোট কাজের জন্য গোটা বাড়ির চাবি দিয়ে দেওয়া।

### সমাধান

লিনাক্স কার্নেল root-এর ক্ষমতাকে ৪০টার মতো আলাদা **capability**-তে ভাগ করেছে। এখন একটা process-কে শুধু দরকারি ক্ষমতাটুকু দেওয়া যায়।

```
         পুরোনো মডেল                      capabilities মডেল
  ┌──────────────────────┐        ┌────────┬────────┬────────┬────────┐
  │        root          │        │NET_RAW │NET_BIND│ CHOWN  │SYS_TIME│ ...
  │  (সব ক্ষমতা একসাথে)  │  ───►  ├────────┼────────┼────────┼────────┤
  └──────────────────────┘        │ KILL   │SETUID  │SYS_ADMIN│ ...   │
                                  └────────┴────────┴────────┴────────┘
                                   যেটা দরকার শুধু সেটাই দাও
```

### কিছু গুরুত্বপূর্ণ capability

| capability | কী অনুমতি দেয় |
|---|---|
| `CAP_NET_BIND_SERVICE` | ১০২৪-এর নিচের পোর্টে bind করা |
| `CAP_NET_RAW` | raw socket (যেমন `ping`, packet capture) |
| `CAP_NET_ADMIN` | নেটওয়ার্ক কনফিগার (IP, routing, firewall) |
| `CAP_CHOWN` | যেকোনো ফাইলের owner বদলানো |
| `CAP_DAC_OVERRIDE` | ফাইলের read/write/execute পারমিশন চেক এড়িয়ে যাওয়া |
| `CAP_DAC_READ_SEARCH` | যেকোনো ফাইল পড়া আর ডিরেক্টরি ঘাঁটা |
| `CAP_FOWNER` | owner না হয়েও owner-এর মতো কিছু কাজ (যেমন sticky ডিরেক্টরিতে মোছা) |
| `CAP_KILL` | অন্যের process-এ signal পাঠানো |
| `CAP_SETUID` / `CAP_SETGID` | যেকোনো UID/GID ধারণ করা |
| `CAP_LINUX_IMMUTABLE` | `chattr +i`/`+a` বসানো ও তোলা |
| `CAP_SYS_TIME` | সিস্টেম ঘড়ি বদলানো |
| `CAP_SYS_ADMIN` | বিশাল একটা ঝুড়ি: mount, অনেক admin কাজ। কার্যত প্রায় root |

পুরো তালিকা: `man 7 capabilities`।

### file capability-র তিনটা ফ্ল্যাগ

ফাইলে capability বসানোর সময় তিনটা ফ্ল্যাগের কথা আসে:

| ফ্ল্যাগ | নাম | মানে |
|---|---|---|
| `p` | Permitted | process এই ক্ষমতা পেতে পারে |
| `e` | Effective | চালু হওয়ার সাথে সাথে ক্ষমতাটা সক্রিয় থাকবে |
| `i` | Inheritable | exec-এর পরেও কিছু শর্তে উত্তরাধিকার হবে |

সাধারণ প্রোগ্রামের জন্য প্রায় সবসময় `=ep` (বা `+ep`) লেখা হয়।

ACL-এর মতো file capability-ও extended attribute-এ (`security.capability`) থাকে। ফাইলে লিখলে সেটা মুছে যায়, আর `nosuid` মাউন্টে file capability-ও কাজ করে না।

### `getcap`: capability দেখা

```
$ getcap /usr/bin/ping
/usr/bin/ping cap_net_raw=ep
```

কিছু distro-তে (যেমন Debian/Ubuntu) `ping`-এ এভাবে `cap_net_raw` দেওয়া থাকে। Fedora-র মতো কিছু distro-তে এর দরকার হয় না, কারণ সেখানে `net.ipv4.ping_group_range` sysctl দিয়ে সাধারণ ইউজারকে ICMP "ping socket" খোলার অনুমতি দেওয়া থাকে। তাই তোমার সিস্টেমে আউটপুট খালি হলেও অবাক হয়ো না।

পুরো সিস্টেমে capability-যুক্ত ফাইল খুঁজতে:

```
$ getcap -r / 2>/dev/null
/usr/bin/ping cap_net_raw=ep
/usr/bin/mtr-packet cap_net_raw=ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin,cap_sys_nice=ep
```

(তালিকা ইনস্টল করা প্যাকেজের ওপর নির্ভর করবে।) SUID তালিকার মতো এটাও নিরাপত্তা অডিটে দেখা উচিত।

### `setcap`: capability বসানো

`setcap` আসে Debian/Ubuntu-তে `libcap2-bin`, Fedora/RHEL-এ `libcap` প্যাকেজে।

উদাহরণ: একটা প্রোগ্রামকে root ছাড়াই পোর্ট ৮০-তে চালাতে চাই। `setcap` symlink-এ কাজ করে না, আসল ফাইলে করতে হয়। আর সিস্টেমের `python3`-এ এটা দিলে সব python স্ক্রিপ্টই ক্ষমতাটা পেয়ে যেত, তাই একটা কপিতে পরীক্ষা করি:

```
$ cp "$(readlink -f /usr/bin/python3)" ~/mypython
$ ~/mypython -m http.server 80
PermissionError: [Errno 13] Permission denied

$ sudo setcap cap_net_bind_service=+ep ~/mypython
$ getcap ~/mypython
/home/nishat/mypython cap_net_bind_service=ep
$ ~/mypython -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

`Ctrl+C` দিয়ে সার্ভার বন্ধ করো। তারপর capability তুলে দাও আর কপিটা মুছে ফেলো:

```
$ sudo setcap -r ~/mypython
$ rm ~/mypython
```

| কমান্ড | কাজ |
|---|---|
| `getcap file` | ফাইলের capability দেখাও |
| `getcap -r dir` | রিকার্সিভ খোঁজো |
| `setcap cap_x=+ep file` | capability বসাও |
| `setcap cap_a,cap_b=+ep file` | একাধিক একসাথে |
| `setcap -r file` | সব capability মুছে দাও |
| `capsh --print` | বর্তমান shell-এর capability দেখাও |
| `grep Cap /proc/$$/status` | process-এর capability (hex আকারে) |
| `capsh --decode=HEX` | hex মানটা নামে রূপান্তর |

```
$ grep CapEff /proc/$$/status
CapEff:	0000000000000000
```

সাধারণ ইউজারের shell-এ effective capability শূন্য।

> ⚠️ **সতর্কতা:** capability-ও বিপজ্জনক হতে পারে। `cap_setuid` দেওয়া `python3`, `cap_dac_override` দেওয়া এডিটর, বা `cap_sys_admin` দেওয়া যেকোনো কিছু মানে কার্যত root। SUID-এর মতো এখানেও একই নিয়ম: সাধারণ interpreter বা টুলে capability দেবে না।

> 💡 **টিপস:** systemd সার্ভিসে `AmbientCapabilities=CAP_NET_BIND_SERVICE` লিখে ফাইলে হাত না দিয়েই সার্ভিসকে capability দেওয়া যায় ([অধ্যায় ২৬](26-systemd-services.md))। আর শুধু নিচের পোর্ট খোলার জন্য আরেক উপায় হলো `net.ipv4.ip_unprivileged_port_start` sysctl। container-এর জগতে (Docker, Podman) capability-ই মূল নিরাপত্তা হাতিয়ার ([অধ্যায় ৪৮](48-containers-virtualization.md))।

---

## ১৭.১২ কোনটা কখন: তুলনা

| প্রয়োজন | ব্যবহার করো |
|---|---|
| সাধারণ ইউজার নির্দিষ্ট একটা প্রোগ্রাম দিয়ে root-এর ফাইল বদলাবে | SUID (শুধু ভালোভাবে লেখা, বিশ্বস্ত binary-তে) |
| প্রোগ্রাম শুধু একটা group-এর ক্ষমতা পাবে | SGID (ফাইলে) |
| টিম ফোল্ডারের সব ফাইল একই group পাবে | SGID (ডিরেক্টরিতে) |
| সবাই লিখবে, কিন্তু অন্যের ফাইল মুছবে না | sticky bit |
| নির্দিষ্ট কয়েকজনকে আলাদা আলাদা অনুমতি | ACL |
| নতুন ফাইলেও স্বয়ংক্রিয়ভাবে একই অনুমতি | default ACL |
| কেউই (root-ও) যেন বদলাতে না পারে | `chattr +i` |
| লগে শুধু যোগ করা যাবে | `chattr +a` |
| প্রোগ্রামের শুধু একটা root-ক্ষমতা দরকার | capabilities |
| ইউজার নির্দিষ্ট কমান্ড root হিসেবে চালাবে | `sudo` ([অধ্যায় ১৮](18-sudo-root.md)) |

### প্রাসঙ্গিক কীবোর্ড ও নোটেশন শর্টকাট

এই অধ্যায়ের নিজস্ব কোনো কীবোর্ড শর্টকাট নেই, তবে কাজের গতি বাড়ায় এমন কয়েকটা জিনিস:

| শর্টকাট | কাজ |
|---|---|
| `Tab` | `setfacl`, `chattr`-এর পরে ফাইলের নাম auto-complete |
| `Alt+.` | আগের কমান্ডের শেষ argument (যেমন একই ফাইলে `chattr` তারপর `lsattr`) ([অধ্যায় ৬](06-terminal-shortcuts.md)) |
| `sudo !!` | শেষ কমান্ডটা sudo দিয়ে আবার চালাও (যখন "Operation not permitted" আসে) |
| `u+s` / `g+s` / `+t` | SUID / SGID / sticky-র symbolic রূপ |
| `u+s,g+s` | কমা দিয়ে একসাথে একাধিক পরিবর্তন |
| `X` (বড় হাতের) | শুধু ডিরেক্টরি বা আগে থেকেই executable ফাইলে execute |
| `u:`, `g:`, `m:`, `o:` | setfacl-এ `user:`, `group:`, `mask:`, `other:`-এর সংক্ষিপ্ত রূপ |
| `d:` | setfacl spec-এর শুরুতে দিলে `-d`-এর সমান (যেমন `setfacl -m d:u:alice:rw dir`) |

---

## হাতে-কলমে অনুশীলন

একটা VM বা টেস্ট মেশিনে করো। `sudo` লাগবে। (sudo নিয়ে বিস্তারিত [অধ্যায় ১৮](18-sudo-root.md)-এ।)

### ল্যাব ১: SUID দেখা

```
$ ls -l /usr/bin/passwd /usr/bin/sudo
$ stat -c '%a %n' /usr/bin/passwd
```

1. owner ঘরে `s` আছে কি না দেখো, octal মান `4755` কি না দেখো।
2. একটা টার্মিনালে `passwd` চালিয়ে থেমে থাকো, অন্যটায় `ps -o pid,ruser,euser,comm -C passwd` দিয়ে RUSER আর EUSER মেলাও। তারপর `Ctrl+C`।

### ল্যাব ২: `s` আর `S`

```
$ cd ~ && touch lab2
$ chmod 4644 lab2 && ls -l lab2        # rwS দেখবে
$ chmod u+x lab2 && ls -l lab2         # rws দেখবে
$ chmod 2644 lab2 && ls -l lab2        # group ঘরে S
$ rm lab2
```

### ল্যাব ৩: টিম ফোল্ডার (SGID + sticky)

```
$ sudo groupadd labteam
$ sudo useradd -m -G labteam ali
$ sudo useradd -m -G labteam bela
$ sudo mkdir /srv/labteam
$ sudo chown root:labteam /srv/labteam
$ sudo chmod 3775 /srv/labteam
$ ls -ld /srv/labteam
drwxrwsr-t 2 root labteam 4096 Nov 12 12:00 /srv/labteam
```

```
$ sudo -u ali touch /srv/labteam/ali.txt
$ ls -l /srv/labteam
-rw-r--r-- 1 ali labteam 0 Nov 12 12:01 ali.txt
```

group `labteam` হয়েছে (SGID-এর কাজ)। এখন bela মোছার চেষ্টা করুক:

```
$ sudo -u bela rm /srv/labteam/ali.txt
rm: cannot remove '/srv/labteam/ali.txt': Operation not permitted
```

এটা sticky-র কাজ। (`sudo -u` দিয়ে চালানো কমান্ডে umask সাধারণত root-এর `022` থেকে আসে, তাই ফাইলে group-write নেই। পরের ল্যাবে ACL দিয়ে এটা ঠিক করব।)

### ল্যাব ৪: ACL

আগে sticky বিটটা তুলে দাও। কারণ sticky আর group-writable ডিরেক্টরিতে `fs.protected_regular = 2` থাকলে kernel অন্যের ফাইল `>>` দিয়ে খোলাও আটকে দেয় (১৭.৫ দেখো), তখন ACL-এর প্রভাব আলাদা করে বোঝা যাবে না।

```
$ sudo chmod -t /srv/labteam
$ sudo setfacl -m g:labteam:rwX /srv/labteam
$ sudo setfacl -d -m g:labteam:rwX /srv/labteam
$ getfacl /srv/labteam
$ sudo -u ali touch /srv/labteam/shared.txt
$ getfacl /srv/labteam/shared.txt        # group:labteam:rw- দেখবে
$ sudo -u bela sh -c 'echo hi >> /srv/labteam/shared.txt' && cat /srv/labteam/shared.txt
hi
```

এখন mask নিয়ে খেলো:

```
$ sudo setfacl -m m::r /srv/labteam/shared.txt
$ getfacl /srv/labteam/shared.txt        # #effective:r-- দেখবে
$ sudo -u bela sh -c 'echo again >> /srv/labteam/shared.txt'
sh: 1: cannot create /srv/labteam/shared.txt: Permission denied
```

ব্যাকআপ নিয়ে সব মুছে আবার ফিরিয়ে আনো:

```
$ getfacl -R /srv/labteam > /tmp/labacl.txt
$ sudo setfacl -R -b /srv/labteam
$ ls -l /srv/labteam                     # + নেই
$ cd / && sudo setfacl --restore=/tmp/labacl.txt
$ ls -l /srv/labteam                     # + ফিরে এসেছে
```

(`--restore` ফাইলের relative path ব্যবহার করে, তাই `/` থেকে চালাতে হয়।)

### ল্যাব ৫: chattr

```
$ echo "do not touch" > ~/lab5.txt
$ sudo chattr +i ~/lab5.txt
$ lsattr ~/lab5.txt
$ rm -f ~/lab5.txt                       # Operation not permitted
$ sudo chattr -i ~/lab5.txt
$ sudo chattr +a ~/lab5.txt
$ echo "line 2" >> ~/lab5.txt            # কাজ করবে
$ echo "replace" > ~/lab5.txt            # Operation not permitted
$ sudo chattr -a ~/lab5.txt && rm ~/lab5.txt
```

### ল্যাব ৬: SUID ও capability অডিট

```
$ sudo find / -xdev -type f -perm /6000 2>/dev/null | sort > ~/suid-baseline.txt
$ wc -l ~/suid-baseline.txt
$ getcap -r / 2>/dev/null
```

তালিকার তিনটা ফাইল বেছে নাও, আর `dpkg -S` বা `rpm -qf` দিয়ে দেখো কোন প্যাকেজের। প্রতিটি কেন SUID/SGID, সেটা ব্যাখ্যা করার চেষ্টা করো।

### পরিষ্কার করা

```
$ sudo rm -rf /srv/labteam
$ sudo userdel -r ali; sudo userdel -r bela
$ sudo groupdel labteam
```

---

## সারসংক্ষেপ

| কমান্ড / নোটেশন | কাজ |
|---|---|
| `chmod u+s f` / `chmod 4755 f` | SUID বসাও |
| `chmod g+s d` / `chmod 2775 d` | SGID বসাও |
| `chmod +t d` / `chmod 1777 d` | sticky বসাও |
| `chmod u-s,g-s f` | SUID/SGID তুলে দাও (ডিরেক্টরিতে এটাই নিরাপদ পথ) |
| `s` / `S` | SUID বা SGID, execute সহ / execute ছাড়া |
| `t` / `T` | sticky, others-execute সহ / ছাড়া |
| `stat -c '%a %A %n' f` | চার অঙ্কের মোড দেখাও |
| `find / -xdev -type f -perm -4000` | SUID ফাইল খোঁজো |
| `find / -xdev -type f -perm /6000` | SUID বা SGID খোঁজো |
| `getfacl f` | ACL দেখাও |
| `setfacl -m u:alice:rw f` | ইউজারকে অনুমতি দাও |
| `setfacl -m g:dev:rX f` | group-কে অনুমতি দাও |
| `setfacl -m m::r f` | mask বদলাও |
| `setfacl -x u:alice f` | entry মুছে দাও |
| `setfacl -b f` | সব ACL মুছে দাও |
| `setfacl -d -m g:dev:rwX d` | default ACL (inherit) |
| `setfacl -k d` | default ACL মুছে দাও |
| `setfacl -R ...` | রিকার্সিভ |
| `getfacl -R d > bk` / `setfacl --restore=bk` | ACL ব্যাকআপ ও রিস্টোর |
| `lsattr f` / `lsattr -d d` | অ্যাট্রিবিউট দেখাও |
| `chattr +i f` / `chattr -i f` | immutable চালু / বন্ধ |
| `chattr +a f` | append-only |
| `getcap f` / `getcap -r /` | capability দেখাও / খোঁজো |
| `setcap cap_net_bind_service=+ep f` | capability বসাও |
| `setcap -r f` | capability মুছে দাও |
| `capsh --print` | বর্তমান capability |

---

## নিজেকে যাচাই করো

1. সাধারণ ইউজার `/etc/shadow`-এ লিখতে পারে না, তবু `passwd` দিয়ে পাসওয়ার্ড বদলাতে পারে কেন? এখানে Real UID আর Effective UID-এর ভূমিকা কী?
2. `-rwSr--r--`-এর বড় হাতের `S` কী বোঝায়? এটা কি কাজের?
3. একটা ডিরেক্টরিতে SGID দিলে নতুন ফাইল আর নতুন সাব-ডিরেক্টরির ওপর কী প্রভাব পড়ে?
4. `/tmp`-এর মোড কত, আর ওখানে sticky bit না থাকলে কী সমস্যা হতো?
5. `chmod 6755` আর `chmod 3775`-এ কোন কোন বিশেষ বিট আছে?
6. একটা bash স্ক্রিপ্টে `chmod u+s` দিলে কি সেটা root হিসেবে চলবে? কেন?
7. একটা ফাইলের `ls -l` আউটপুট `-rw-rw----+`। শেষের `+` কী বোঝায়, আর group ঘরের `rw-` আসলে কী দেখাচ্ছে?
8. ACL থাকা একটা ফাইলে `user:alice:rw-` আর `mask::r--`। alice কী করতে পারবে?
9. `setfacl -x`, `-b` আর `-k`-এর পার্থক্য কী?
10. root হয়েও একটা ফাইল মোছা যাচ্ছে না, "Operation not permitted" আসছে। প্রথমে কী চেক করবে? আর ওয়েব সার্ভারকে পোর্ট ৮০ দিতে SUID-root-এর চেয়ে ভালো উপায় কী?

<details><summary>উত্তর দেখো</summary>

1. `/usr/bin/passwd`-এ SUID বিট আছে আর owner root। তাই চালালে process-এর Effective UID হয় 0 (root), যদিও Real UID থাকে ইউজারের নিজের। kernel ফাইল খোলার অনুমতি দেয় EUID দেখে, তাই `/etc/shadow` খোলা যায়। `passwd` প্রোগ্রাম নিজে Real UID দেখে নিশ্চিত করে যে ইউজার শুধু নিজের পাসওয়ার্ডই বদলাচ্ছে।
2. SUID বিট দেওয়া আছে কিন্তু owner-এর execute বিট নেই। ফাইলটা চালানো যায় না, তাই SUID অর্থহীন। সাধারণত এটা ভুলের লক্ষণ।
3. নতুন ফাইল আর সাব-ডিরেক্টরি ডিরেক্টরির group পায় (যে বানিয়েছে তার primary group নয়)। নতুন সাব-ডিরেক্টরিগুলো নিজেরাও SGID বিট পায়।
4. `1777` (`drwxrwxrwt`)। sticky না থাকলে ডিরেক্টরিতে write পারমিশন থাকার কারণে যেকোনো ইউজার অন্য যেকোনো ইউজারের ফাইল মুছে বা rename করে ফেলতে পারত।
5. `6755` = SUID + SGID (4+2), সাথে `rwxr-xr-x`। `3775` = SGID + sticky (2+1), সাথে `rwxrwxr-x`।
6. না। লিনাক্স kernel `#!` দিয়ে শুরু হওয়া interpreted স্ক্রিপ্টের SUID উপেক্ষা করে। race condition আর environment variable দিয়ে interpreter-কে প্রভাবিত করার ঝুঁকির কারণে। এ রকম দরকারে `sudo`-র নিয়ম লেখো।
7. `+` মানে ফাইলে extended ACL আছে। ACL থাকলে `ls -l`-এর group ঘর ACL **mask** দেখায়, owning group-এর পারমিশন নয়। আসল ছবি দেখতে `getfacl` চালাও।
8. শুধু পড়তে পারবে। effective পারমিশন = entry ∩ mask = `rw-` ∩ `r--` = `r--`।
9. `-x` একটা নির্দিষ্ট entry মুছে দেয় (যেমন `-x u:alice`)। `-b` সব extended ACL entry মুছে দেয় (default সহ), শুধু সাধারণ owner/group/other থাকে। `-k` শুধু default ACL মুছে দেয়।
10. প্রথমে `lsattr` দিয়ে দেখো ফাইলে `i` (immutable) বা `a` (append-only) অ্যাট্রিবিউট আছে কি না। থাকলে `chattr -i`/`-a` দিয়ে তুলে নাও। (এ ছাড়া ফাইলসিস্টেম read-only মাউন্ট করা কি না, সেটাও দেখতে পারো।) পোর্ট ৮০-র জন্য SUID-root না দিয়ে শুধু `cap_net_bind_service` capability দাও (`setcap cap_net_bind_service=+ep binary`), অথবা systemd সার্ভিসে `AmbientCapabilities=CAP_NET_BIND_SERVICE`।

</details>

---

[← আগের অধ্যায়](16-permissions.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](18-sudo-root.md)

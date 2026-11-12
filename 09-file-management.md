# অধ্যায় ৯: ফাইল ও ডিরেক্টরি ম্যানেজমেন্ট

আগের অধ্যায়ে তুমি শিখেছ কীভাবে ফাইল সিস্টেমে ঘুরে বেড়াতে হয় (`pwd`, `cd`, `ls`)। এবার সময় হাতে কাজ করার: নতুন ফাইল আর ফোল্ডার বানানো, কপি করা, সরানো, নাম বদলানো, মুছে ফেলা। লিনাক্স ব্যবহারকারী দিনে সবচেয়ে বেশি যে কমান্ডগুলো চালায়, এগুলো তার মধ্যে পড়ে। তাই এই অধ্যায়টা মন দিয়ে পড়ো। এখানে একটা ভুল কমান্ড, বিশেষ করে `rm`, তোমার কয়েক মাসের কাজ মুহূর্তে মুছে দিতে পারে।

## এই অধ্যায়ে যা শিখবে

- `touch` দিয়ে খালি ফাইল বানানো এবং ফাইলের তিন রকম timestamp (atime, mtime, ctime) কী, কেন আছে
- `mkdir` ও `mkdir -p` দিয়ে এক কমান্ডে পুরো ফোল্ডার-গাছ বানানো
- `cp` দিয়ে কপি: `-r`, `-i`, `-v`, `-p`, `-a`, `-u` সহ গুরুত্বপূর্ণ সব অপশন
- `mv` দিয়ে নাম বদলানো ও সরানো, এবং `-i`, `-n` দিয়ে ভুল করে ওভাররাইট ঠেকানো
- `rm` ও `rmdir` দিয়ে মোছা, `rm -rf /` কেন ভয়ংকর, আর `trash-cli` দিয়ে নিরাপদে মোছা
- `stat` দিয়ে ফাইলের পুরো "জন্মকুণ্ডলী" দেখা
- নামের মধ্যে space, `-`, `*`, `$` ইত্যাদি বিশেষ অক্ষর থাকলে কীভাবে quoting করতে হয়
- ফাইল কপি, মুভ বা ডিলিট করলে ভেতরে inode আর data block-এর কী হয়
- `basename`, `dirname`, `realpath` দিয়ে পাথ নিয়ে কাজ
- ওয়াইল্ডকার্ড (glob) আর brace expansion দিয়ে একসাথে অনেক ফাইলে কাজ
- এসব কাজ দ্রুত করার কীবোর্ড শর্টকাট

---

## ৯.১ শুরুর আগে: ফাইল আসলে কী? (inode-এর এক ঝলক)

কমান্ড শেখার আগে একটা জিনিস মাথায় ঢুকিয়ে নাও। এটা বুঝলে বাকি সব কমান্ডের আচরণ নিজে থেকেই পরিষ্কার হয়ে যাবে।

লিনাক্সে একটা "ফাইল" তিনটা আলাদা অংশ দিয়ে তৈরি:

1. **Directory entry (নাম)**: ডিরেক্টরি আসলে একটা বিশেষ ফাইল, যার ভেতরে একটা তালিকা থাকে: "এই নাম → এই inode নম্বর"।
2. **inode (index node)**: ফাইলের metadata বা তথ্যপত্র। এতে থাকে ফাইলের সাইজ, মালিক (owner), পারমিশন, timestamp, link count, আর ডেটা ডিস্কের কোন কোন block-এ আছে তার ঠিকানা। **ফাইলের নাম inode-এ থাকে না।**
3. **Data blocks**: ডিস্কের যেখানে ফাইলের আসল কনটেন্ট থাকে।

একটা লাইব্রেরির কথা ভাবো:

- ক্যাটালগ কার্ডে লেখা থাকে বইয়ের নাম আর একটা নম্বর। এটা **directory entry**।
- নম্বর মিলিয়ে রেজিস্টার খাতায় পাবে বইয়ের সব তথ্য: লেখক, পৃষ্ঠাসংখ্যা, কোন তাকে আছে। এটা **inode**।
- তাকের উপর রাখা আসল বইটা হলো **data blocks**।

```
  ডিরেক্টরি /home/rahim/docs (নিজেও একটা ফাইল)
  ┌─────────────────────────────┐
  │  নাম             inode নং   │
  │  report.txt  ──► 1835093    │──────┐
  │  photo.jpg   ──► 1835101    │      │
  └─────────────────────────────┘      ▼
                         ┌──────────────────────────────┐
                         │ inode 1835093                │
                         │  type: regular file          │
                         │  size: 2048 bytes            │
                         │  owner: rahim, perm: 0644    │
                         │  atime / mtime / ctime       │
                         │  link count: 1               │
                         │  data blocks: 88120, 88121 ──┼──► [ডিস্কের ডেটা]
                         └──────────────────────────────┘
```

এই ছবি থেকে কয়েকটা জিনিস বোঝা যায়, যা পরে কাজে লাগবে:

- **নাম বদলানো (`mv`, একই ফাইলসিস্টেমে)** মানে শুধু ক্যাটালগ কার্ডে নাম পাল্টানো। বই সরে না, তাই কাজটা প্রায় সঙ্গে সঙ্গে শেষ হয়।
- **কপি (`cp`)** মানে নতুন inode, নতুন data block, পুরো ডেটার আরেকটা প্রতিলিপি। বড় ফাইলে তাই সময় লাগে।
- **মোছা (`rm`)** মানে ক্যাটালগ কার্ড ছিঁড়ে ফেলা। কোনো কার্ড আর ওই বইকে না দেখালে (link count 0) এবং কেউ বইটা পড়ছে না থাকলে, তবেই তাকের জায়গা খালি হিসেবে চিহ্নিত হয়।

inode আর লিংকের বিস্তারিত আলোচনা আছে [অধ্যায় ১১](11-links-inode.md)-এ। এখানে শুধু ততটুকু নিচ্ছি যতটুকু এই অধ্যায়ের জন্য দরকার।

> 📝 **মনে রাখো:** লিনাক্সে "ফাইলের নাম" আর "ফাইল" এক জিনিস না। নাম হলো ডিরেক্টরির ভেতরের একটা এন্ট্রি, যেটা একটা inode-কে দেখায়।

---

## ৯.২ `touch`: খালি ফাইল বানানো ও timestamp বদলানো

### কী করে

`touch` এর মূল কাজ আসলে ফাইলের **timestamp আপডেট করা**। ফাইল না থাকলে সেটা একটা খালি (০ বাইট) ফাইল বানায়। এই দ্বিতীয় আচরণটার জন্যই বেশিরভাগ মানুষ `touch` চেনে।

### Syntax

```bash
touch [OPTION]... FILE...
```

### সাধারণ ব্যবহার

```bash
$ touch notes.txt
$ ls -l notes.txt
-rw-r--r-- 1 rahim rahim 0 Mar 12 10:15 notes.txt
```

একসাথে অনেকগুলো:

```bash
$ touch a.txt b.txt c.txt
$ touch file{1..5}.log        # brace expansion: file1.log ... file5.log
$ ls
a.txt  b.txt  c.txt  file1.log  file2.log  file3.log  file4.log  file5.log
```

ফাইল আগে থেকেই থাকলে `touch` তার কনটেন্টে হাত দেয় না। শুধু সময়টা "এখন" করে দেয়।

### তিন রকম timestamp: atime, mtime, ctime

প্রতিটা inode-এ অন্তত তিনটা সময় লেখা থাকে:

| Timestamp | পুরো নাম | কখন বদলায় | উপমা |
|---|---|---|---|
| **atime** | access time | ফাইলের **কনটেন্ট পড়লে** (যেমন `cat`) | বইটা শেষ কবে কেউ খুলে পড়েছে |
| **mtime** | modification time | ফাইলের **কনটেন্ট বদলালে** | বইয়ের লেখা শেষ কবে বদলানো হয়েছে |
| **ctime** | change time | **inode বদলালে**: কনটেন্ট, পারমিশন, মালিক, নাম, link count | রেজিস্টার খাতার এন্ট্রি শেষ কবে বদলানো হয়েছে |
| **btime** | birth time | ফাইল তৈরির সময় (সব ফাইলসিস্টেমে থাকে না) | বইটা লাইব্রেরিতে কবে এসেছে |

> ⚠️ **সতর্কতা:** অনেকে ভাবে ctime মানে "creation time"। ভুল। ctime হলো **change** time, মানে metadata শেষ কবে বদলেছে। তৈরির সময়কে বলে birth time (btime)। ext4, XFS, Btrfs-এর মতো আধুনিক ফাইলসিস্টেম এটা রাখে, আর নতুন coreutils-এর `stat` (Ubuntu 24.04, Debian 12, Fedora, RHEL 9) এটা `Birth:` লাইনে দেখায়।

কিছু নিয়ম মনে রাখো:

- mtime বদলালে ctime-ও বদলায়, কারণ কনটেন্ট বদলালে inode-এর সাইজ ও mtime ফিল্ডও বদলায়।
- `chmod`, `chown`, `mv` (নাম বদলানো), `ln` (হার্ড লিংক) শুধু ctime বদলায়, mtime না।
- **ctime তুমি ইচ্ছামতো সেট করতে পারো না।** `touch` দিয়েও না। inode-এ যেকোনো পরিবর্তন হলে কার্নেল নিজেই ctime বর্তমান সময়ে সেট করে দেয়। ফরেনসিক তদন্তে ctime এজন্যই এত দরকারি।

#### atime আর `relatime`: পারফরম্যান্সের গল্প

প্রতিবার ফাইল পড়লেই atime আপডেট করতে হলে প্রতিটা **read** অপারেশন একটা ডিস্ক **write** তৈরি করত। এটা ধীর, আর SSD-র আয়ু কমায়। তাই আধুনিক লিনাক্সে ডিফল্ট mount অপশন হলো **`relatime`**। এতে atime আপডেট হয় কেবল তখন, যখন:

- বর্তমান atime, mtime বা ctime-এর চেয়ে পুরনো, অথবা
- শেষ আপডেটের পর ২৪ ঘণ্টার বেশি পার হয়েছে।

এজন্য তুমি `cat` করার পরেও atime না বদলাতে দেখলে অবাক হয়ো না। `findmnt /` চালালে OPTIONS কলামে সাধারণত `relatime` দেখবে। আরও আছে `noatime` (কখনো atime আপডেট হবে না) এবং `strictatime` (প্রতিবার হবে)। mount অপশন নিয়ে বিস্তারিত আছে [অধ্যায় ৩২](32-disk-partition-mount.md)-এ।

### `touch`-এর গুরুত্বপূর্ণ অপশন

| অপশন | কাজ |
|---|---|
| `-a` | শুধু atime বদলাও |
| `-m` | শুধু mtime বদলাও |
| `-c`, `--no-create` | ফাইল না থাকলে নতুন বানিও না |
| `-d STRING`, `--date=STRING` | নির্দিষ্ট সময় দাও (মানুষের ভাষায়, যেমন `"2 days ago"`, `"2024-01-15 14:30"`) |
| `-t STAMP` | `[[CC]YY]MMDDhhmm[.ss]` ফরম্যাটে সময় দাও |
| `-r FILE`, `--reference=FILE` | অন্য একটা ফাইলের timestamp কপি করো |
| `-h`, `--no-dereference` | symlink হলে লিংকের নিজের timestamp বদলাও, যে ফাইলকে দেখায় তার না |

### উদাহরণ

```bash
# নির্দিষ্ট তারিখ বসানো
$ touch -d "2024-01-15 14:30" old-report.txt
$ ls -l old-report.txt
-rw-r--r-- 1 rahim rahim 0 Jan 15  2024 old-report.txt

# -t ফরম্যাট: 2023 সালের 25 ডিসেম্বর রাত 11:59
$ touch -t 202312252359 xmas.txt

# শুধু mtime "২ দিন আগে" করে দাও
$ touch -m -d "2 days ago" build.log

# a.txt-এর সময় b.txt-এ বসাও
$ touch -r a.txt b.txt

# ফাইল থাকলে আপডেট, না থাকলে কিছু কোরো না
$ touch -c maybe.txt
```

> 💡 **টিপস:** `make` এর মতো build tool mtime দেখে ঠিক করে কোন ফাইল আবার compile করতে হবে। `touch source.c` চালালে `make` ভাববে ফাইলটা বদলেছে এবং আবার build করবে। [অধ্যায় ৩১](31-compile-from-source.md)-এ এটা কাজে লাগবে।

> 📝 **মনে রাখো:** ফাইলে কনটেন্ট সহ নতুন ফাইল বানাতে চাইলে `touch` লাগে না। `echo "hello" > file.txt` বা এডিটর ([অধ্যায় ১৩](13-nano-editor.md)) ব্যবহার করো। `>` রিডাইরেকশন নিয়ে বিস্তারিত আছে [অধ্যায় ১৯](19-redirection-pipe.md)-এ।

---

## ৯.৩ `stat`: ফাইলের পুরো তথ্যপত্র

`ls -l` তোমাকে ফাইলের কিছু তথ্য দেখায়। `stat` সরাসরি inode খুলে **সব** তথ্য দেখায়।

### Syntax

```bash
stat [OPTION]... FILE...
```

### উদাহরণ ও আউটপুট

```bash
$ echo "hello linux" > notes.txt
$ stat notes.txt
  File: notes.txt
  Size: 12              Blocks: 8          IO Block: 4096   regular file
Device: 8,2     Inode: 1835093     Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/   rahim)   Gid: ( 1000/   rahim)
Access: 2025-03-12 10:20:41.118034512 +0600
Modify: 2025-03-12 10:20:41.118034512 +0600
Change: 2025-03-12 10:20:41.118034512 +0600
 Birth: 2025-03-12 10:20:41.118034512 +0600
```

লাইন ধরে মানে:

| ফিল্ড | মানে |
|---|---|
| `Size: 12` | ফাইলে ১২ বাইট ("hello linux" = ১১ অক্ষর + ১টা newline) |
| `Blocks: 8` | ডিস্কে কতগুলো ৫১২-বাইট ইউনিট দখল করেছে (৮ × ৫১২ = ৪০৯৬, মানে একটা ৪KB ব্লক) |
| `IO Block: 4096` | ফাইলসিস্টেমের পছন্দের I/O সাইজ |
| `regular file` | ফাইলের ধরন (directory, symbolic link ইত্যাদিও হতে পারে) |
| `Device: 8,2` | কোন ডিভাইসে আছে (major,minor নম্বর) |
| `Inode: 1835093` | inode নম্বর |
| `Links: 1` | কয়টা নাম (হার্ড লিংক) এই inode-কে দেখাচ্ছে |
| `Access: (0644/-rw-r--r--)` | পারমিশন, সংখ্যায় ও অক্ষরে ([অধ্যায় ১৬](16-permissions.md)) |
| `Uid` / `Gid` | মালিক ইউজার ও গ্রুপ |
| `Access/Modify/Change/Birth` | atime / mtime / ctime / btime |

> 📝 **মনে রাখো:** ১২ বাইটের ফাইলও ডিস্কে ৪০৯৬ বাইট দখল করে, কারণ ফাইলসিস্টেম জায়গা দেয় পুরো block হিসেবে। `ls -l` দেখায় আসল সাইজ, `du` দেখায় দখল করা জায়গা।

### timestamp বদলানো হাতে-কলমে দেখো

```bash
$ cat notes.txt > /dev/null        # পড়লাম → atime (relatime নিয়ম মেনে)
$ echo "more" >> notes.txt         # লিখলাম → mtime + ctime
$ chmod 600 notes.txt              # পারমিশন বদলালাম → শুধু ctime
$ stat -c 'atime=%x%nmtime=%y%nctime=%z' notes.txt
atime=2025-03-12 10:20:41.118034512 +0600
mtime=2025-03-12 10:22:05.402118833 +0600
ctime=2025-03-12 10:23:10.771003410 +0600
```

### `stat`-এর গুরুত্বপূর্ণ অপশন

| অপশন | কাজ |
|---|---|
| `-c FORMAT`, `--format=FORMAT` | নিজের মতো ফরম্যাটে আউটপুট (স্ক্রিপ্টে খুব কাজের) |
| `--printf=FORMAT` | `-c`-এর মতো, তবে `\n` এর মতো escape বোঝে আর শেষে নিজে থেকে newline দেয় না |
| `-L`, `--dereference` | symlink হলে লিংক যে ফাইলকে দেখায় তার তথ্য দেখাও |
| `-f`, `--file-system` | ফাইলের বদলে ফাইলসিস্টেমের তথ্য দেখাও |
| `-t`, `--terse` | সংক্ষিপ্ত, এক লাইনের আউটপুট |

### দরকারি format specifier

| Specifier | মানে |
|---|---|
| `%n` | ফাইলের নাম |
| `%s` | সাইজ (বাইট) |
| `%i` | inode নম্বর |
| `%h` | হার্ড লিংক সংখ্যা |
| `%U` / `%G` | মালিক ইউজার / গ্রুপের নাম |
| `%a` / `%A` | পারমিশন সংখ্যায় (644) / অক্ষরে (-rw-r--r--) |
| `%F` | ফাইলের ধরন |
| `%x` / `%y` / `%z` / `%w` | atime / mtime / ctime / btime (পড়ার মতো ফরম্যাটে) |
| `%X` / `%Y` / `%Z` / `%W` | একই সময়, Unix epoch সেকেন্ডে |

```bash
$ stat -c '%n: %s bytes, inode %i, perm %a, owner %U' notes.txt
notes.txt: 17 bytes, inode 1835093, perm 600, owner rahim

$ stat -c '%F' /etc /etc/passwd /dev/null
directory
regular file
character special file
```

---

## ৯.৪ `mkdir`: ডিরেক্টরি বানানো

### Syntax

```bash
mkdir [OPTION]... DIRECTORY...
```

### সাধারণ ব্যবহার

```bash
$ mkdir projects
$ mkdir music videos photos         # একসাথে তিনটা
```

### সমস্যা: মাঝের ফোল্ডার না থাকলে

```bash
$ mkdir projects/website/css
mkdir: cannot create directory 'projects/website/css': No such file or directory
```

`projects/website` নেই, তাই তার ভেতরে `css` বানানো যাচ্ছে না। এর সমাধান `-p`।

### `mkdir -p`: পুরো পথ একবারে

```bash
$ mkdir -p projects/website/css
$ mkdir -pv projects/website/js/lib
mkdir: created directory 'projects/website/js'
mkdir: created directory 'projects/website/js/lib'
```

`-p` (parents) দুটো কাজ করে:

1. পথের মাঝে যা নেই সব বানিয়ে দেয়।
2. ডিরেক্টরি আগে থেকে থাকলে **error দেয় না**। এজন্য স্ক্রিপ্টে `mkdir -p` লেখাই নিরাপদ অভ্যাস, কারণ বারবার চালালেও ভাঙে না। এই গুণকে বলে *idempotent*।

### brace expansion দিয়ে পুরো প্রজেক্টের কাঠামো

```bash
$ mkdir -p myapp/{src/{models,views,controllers},tests,docs,config}
$ tree myapp
myapp
├── config
├── docs
├── src
│   ├── controllers
│   ├── models
│   └── views
└── tests

7 directories, 0 files
```

(`tree` আলাদা প্যাকেজ: `sudo apt install tree` বা `sudo dnf install tree`।)

> 💡 **টিপস:** `{a,b,c}` brace expansion কাজটা করে **শেল**, `mkdir` না। কমান্ড চালানোর আগেই bash এটাকে `myapp/src/models myapp/src/views ...` এ ভেঙে ফেলে। কী হবে আগে দেখতে চাইলে সামনে `echo` বসাও: `echo mkdir -p myapp/{a,b}`। `{` আর `,`-এর আশেপাশে space দিলে কিন্তু expansion হবে না।

### `mkdir`-এর অপশন

| অপশন | কাজ |
|---|---|
| `-p`, `--parents` | মাঝের ডিরেক্টরিও বানাও; থাকলে error দিও না |
| `-v`, `--verbose` | প্রতিটা তৈরি ডিরেক্টরির নাম দেখাও |
| `-m MODE`, `--mode=MODE` | তৈরির সময়েই পারমিশন সেট করো (যেমন `-m 700`) |

```bash
$ mkdir -m 700 private      # শুধু আমি ঢুকতে পারব
$ ls -ld private
drwx------ 2 rahim rahim 4096 Mar 12 10:40 private
```

> 📝 **মনে রাখো:** নতুন ডিরেক্টরির link count কমপক্ষে ২ হয়। একটা হলো প্যারেন্টে তার নাম, আরেকটা হলো তার নিজের ভেতরের `.`। প্রতিটা সাব-ডিরেক্টরি এর সাথে আরও ১ যোগ করে, কারণ সাব-ডিরেক্টরির `..` এটাকে দেখায়।

---

## ৯.৫ নামে space ও বিশেষ অক্ষর: quoting

এই অংশটা `cp`, `mv`, `rm` শেখার আগেই জানা দরকার, কারণ এখানে ভুল করলে ভুল ফাইল মুছে যেতে পারে।

### সমস্যাটা কী

শেল space দেখে কমান্ডকে আলাদা আলাদা "শব্দে" (argument) ভাগ করে। ধরো তোমার একটা ফাইল আছে `my notes.txt`:

```bash
$ rm my notes.txt
rm: cannot remove 'my': No such file or directory
rm: cannot remove 'notes.txt': No such file or directory
```

শেল ভেবেছে তুমি দুটো ফাইল মুছতে বলেছ: `my` আর `notes.txt`। কপাল খারাপ হলে `notes.txt` নামে সত্যিই একটা ফাইল থাকতে পারত। তাহলে সেটা মুছে যেত।

### সমাধান: তিন রকম quoting

| পদ্ধতি | উদাহরণ | আচরণ |
|---|---|---|
| Single quote `'...'` | `'my notes.txt'` | ভেতরের **সবকিছু** হুবহু থাকে। `$`, `*`, `\` এর কোনো বিশেষ মানে থাকে না |
| Double quote `"..."` | `"my notes.txt"` | space রক্ষা করে, কিন্তু `$VAR`, `$(cmd)`, `` ` ``, `\` কাজ করে |
| Backslash `\` | `my\ notes.txt` | ঠিক পরের একটা অক্ষরের বিশেষ মানে বাতিল করে |

```bash
$ touch 'my notes.txt'
$ ls -l "my notes.txt"
-rw-r--r-- 1 rahim rahim 0 Mar 12 10:45 'my notes.txt'
$ mv my\ notes.txt my_notes.txt
```

লক্ষ করো, `ls` নিজেই space-ওয়ালা নাম single quote দিয়ে দেখাচ্ছে। coreutils 8.25-এর পর থেকে টার্মিনালে এটাই ডিফল্ট আচরণ।

### variable সবসময় double quote-এ রাখো

```bash
$ f="my notes.txt"
$ rm $f          # ভুল! ভেঙে দুটো argument হয়ে যায়
$ rm "$f"        # ঠিক
```

শেল স্ক্রিপ্টে এটাই সবচেয়ে বেশি বাগ তৈরি করে ([অধ্যায় ৪১](41-shell-scripting-1.md))।

### বিশেষ অক্ষরের তালিকা

এই অক্ষরগুলো নামে থাকলে quote ছাড়া চলবে না:

| অক্ষর | শেলের কাছে মানে |
|---|---|
| space, tab | argument আলাদা করে |
| `*` `?` `[ ]` | wildcard (glob) |
| `$` | variable |
| `'` `"` `` ` `` | quoting / command substitution |
| `\` | escape |
| `&` `;` `\|` | কমান্ড আলাদা করা / ব্যাকগ্রাউন্ড / pipe |
| `<` `>` | রিডাইরেকশন |
| `(` `)` `{` `}` | subshell / brace expansion |
| `!` | history expansion (ইন্টার‍্যাক্টিভ bash-এ) |
| `#` | শব্দের শুরুতে থাকলে comment |
| `~` | শব্দের শুরুতে থাকলে home ডিরেক্টরি |

```bash
$ touch 'price$100.txt' 'what?.txt' 'rock&roll.mp3'
$ ls
'price$100.txt'  'rock&roll.mp3'  'what?.txt'
```

### নাম যদি `-` দিয়ে শুরু হয়

`-file.txt` নামের ফাইল থাকলে quote দিয়েও কাজ হয় না, কারণ quote শুধু শেলের জন্য। কমান্ড নিজে `-f...` দেখে সেটাকে অপশন ভাবে:

```bash
$ rm -file.txt
rm: invalid option -- 'l'
Try 'rm ./-file.txt' to get rid of the file '-file.txt'.
Try 'rm --help' for more information.
```

দুটো সমাধান আছে:

```bash
$ rm -- -file.txt      # "--" মানে: এখানে অপশন শেষ, এর পরে সব ফাইলের নাম
$ rm ./-file.txt       # path দিলে আর '-' দিয়ে শুরু হয় না
```

`--` এর নিয়মটা প্রায় সব GNU কমান্ডে কাজ করে।

> 💡 **টিপস:** সবচেয়ে সহজ উপায় **Tab completion**। `rm my` টাইপ করে `Tab` চাপো। শেল নিজেই `rm my\ notes.txt` বানিয়ে দেবে, escape সহ। এতে টাইপো হয় না, আর quoting নিয়ে মাথাও ঘামাতে হয় না।

> ⚠️ **সতর্কতা:** লিনাক্সে ফাইলের নামে `/` আর NUL (`\0`) ছাড়া যেকোনো অক্ষর বৈধ, এমনকি newline-ও। তবে নিজের ফাইলের নামে space ও বিশেষ অক্ষর এড়িয়ে চলো। `my_notes.txt` বা `my-notes.txt` লেখো। পরে নিজেরই সুবিধা হবে।

---

## ৯.৬ ওয়াইল্ডকার্ড (glob): একসাথে অনেক ফাইল

`cp`, `mv`, `rm` প্রায়ই একসাথে অনেক ফাইলে চালাতে হয়। এজন্য শেল দেয় **globbing**:

| প্যাটার্ন | মেলে | উদাহরণ |
|---|---|---|
| `*` | যেকোনো সংখ্যক যেকোনো অক্ষর (শূন্যও) | `*.txt` → সব .txt |
| `?` | ঠিক একটা যেকোনো অক্ষর | `file?.log` → file1.log, fileA.log |
| `[abc]` | বন্ধনীর যেকোনো একটা অক্ষর | `[abc]*.txt` |
| `[0-9]` | পরিসরের একটা অক্ষর | `img[0-9].png` |
| `[!0-9]` বা `[^0-9]` | বন্ধনীর বাইরের একটা অক্ষর | `[!.]*` |

```bash
$ ls
a.txt  b.txt  c.log  img1.png  img2.png  img10.png
$ ls *.txt
a.txt  b.txt
$ ls img?.png
img1.png  img2.png
$ ls img[0-9]*.png
img10.png  img1.png  img2.png
```

**ভেতরে কী হয়:** glob শেল নিজে expand করে, কমান্ড চলার **আগেই**। `rm *.txt` চালালে `rm` আসলে পায় `rm a.txt b.txt`। `rm` কখনো `*` দেখতেই পায় না।

```
  তুমি টাইপ করো:   rm *.txt
         │
         ▼
  bash: ডিরেক্টরি পড়ে প্যাটার্ন মেলায়
         │
         ▼
  আসলে চলে:        rm a.txt b.txt     ← rm শুধু এটাই দেখে
```

> 📝 **মনে রাখো:** `*` ডিফল্টে hidden ফাইল (`.` দিয়ে শুরু) ধরে না। কোনো ফাইল না মিললে bash প্যাটার্নটা যেমন আছে তেমনই কমান্ডকে দিয়ে দেয়, যেমন `ls *.xyz` তখন error দেয় `cannot access '*.xyz'`।

> 💡 **টিপস:** বিপজ্জনক কাজের আগে সবসময় একই glob দিয়ে `ls` বা `echo` চালাও: `echo rm *.log`। কী মুছবে আগে দেখে নাও, তারপর `echo` বাদ দিয়ে চালাও।

---

## ৯.৭ `cp`: কপি করা

### Syntax

```bash
cp [OPTION]... SOURCE DEST              # একটা ফাইল কপি / নতুন নামে কপি
cp [OPTION]... SOURCE... DIRECTORY      # অনেক ফাইল একটা ডিরেক্টরিতে
cp [OPTION]... -t DIRECTORY SOURCE...   # target আগে দিয়ে
```

### মৌলিক ব্যবহার

```bash
$ cp report.txt report-backup.txt        # একই জায়গায় নতুন নামে
$ cp report.txt ~/Documents/             # অন্য ডিরেক্টরিতে, একই নামে
$ cp report.txt ~/Documents/final.txt    # অন্য ডিরেক্টরিতে, নতুন নামে
$ cp a.txt b.txt c.txt backup/           # অনেক ফাইল একসাথে
$ cp *.jpg ~/Pictures/                   # glob দিয়ে
```

শেষ argument যদি একটা existing ডিরেক্টরি হয়, ফাইল তার **ভেতরে** যায়। না হলে শেষ argument-টা নতুন ফাইলের নাম হয়।

> ⚠️ **সতর্কতা:** `cp` ডিফল্টে **কোনো প্রশ্ন না করেই** গন্তব্যের ফাইল ওভাররাইট করে দেয়। `cp new.txt important.txt` চালালে `important.txt`-এর পুরনো কনটেন্ট চিরতরে হারিয়ে যায়।

### ডিরেক্টরি কপি: `-r`

```bash
$ cp projects backup
cp: -r not specified; omitting directory 'projects'

$ cp -r projects backup
```

`-r` (বা `-R`) মানে recursive: ভেতরের সব ফাইল আর সাব-ডিরেক্টরি সহ কপি।

**একটা সূক্ষ্ম নিয়ম আছে যেটা নতুনদের বিভ্রান্ত করে:**

- `backup` **না থাকলে**: `cp -r projects backup` চালালে `backup` তৈরি হয়, আর তার ভেতরে থাকে `projects`-এর কনটেন্ট।
- `backup` **আগে থেকে থাকলে**: একই কমান্ড `backup/projects` তৈরি করে।

তাই একই কমান্ড দ্বিতীয়বার চালালে ভিন্ন ফল আসে! শুধু কনটেন্ট কপি করতে চাইলে লেখো:

```bash
$ cp -r projects/. backup/      # projects-এর ভেতরের সব (hidden সহ) backup-এ
```

(`rsync`-এ trailing slash-এর মানে আবার আলাদা। সেটা আছে [অধ্যায় ৩৫](35-backup-rsync.md)-এ।)

### `-i`: ওভাররাইটের আগে জিজ্ঞেস করো

```bash
$ cp -i new.txt important.txt
cp: overwrite 'important.txt'? n
```

`y` দিলে ওভাররাইট হবে, `n` বা অন্য কিছু দিলে বাদ যাবে।

### `-v`: কী হচ্ছে দেখাও

```bash
$ cp -v *.txt backup/
'a.txt' -> 'backup/a.txt'
'b.txt' -> 'backup/b.txt'
```

### `-p`: মূল তথ্য ধরে রাখো

সাধারণ `cp`-তে নতুন ফাইলের মালিক হও তুমি, mtime হয় "এখন", আর পারমিশন হয় umask অনুযায়ী (বিস্তারিত [অধ্যায় ১৬](16-permissions.md)-এ)। `-p` দিলে মূল ফাইলের **mode (পারমিশন), ownership আর timestamps** ধরে রাখা হয়।

```bash
$ ls -l old.txt
-rw-r----- 1 rahim rahim 120 Jan 15  2024 old.txt
$ cp old.txt c1.txt
$ cp -p old.txt c2.txt
$ ls -l c1.txt c2.txt
-rw-r----- 1 rahim rahim 120 Mar 12 11:02 c1.txt
-rw-r----- 1 rahim rahim 120 Jan 15  2024 c2.txt
```

> 📝 **মনে রাখো:** ownership ধরে রাখতে পারে শুধু root। সাধারণ ইউজার অন্যের মালিকানার ফাইল `-p` দিয়ে কপি করলে মালিক নিজেই হয়ে যায়।

### `-a`: আর্কাইভ মোড (ব্যাকআপের জন্য সেরা)

`-a` = `-dR --preserve=all`। সহজ ভাষায়:

- recursive (`-R`)
- symlink-কে symlink হিসেবেই কপি করে, যে ফাইলকে দেখায় সেটা না (`-d` = `--no-dereference --preserve=links`)
- mode, ownership, timestamps, হার্ড লিংক, এবং সম্ভব হলে context, xattr ও ACL সব ধরে রাখে

```bash
$ sudo cp -a /etc /backup/etc-2025-03-12
```

ফোল্ডারের "হুবহু" কপি চাইলে `cp -a` ব্যবহার করো।

### `-u`: শুধু নতুন হলে কপি করো

```bash
$ cp -u *.txt backup/
```

গন্তব্যে ফাইল না থাকলে, অথবা উৎসের mtime গন্তব্যের চেয়ে নতুন হলে, তবেই কপি হয়। এটা সহজ "সিঙ্ক" এর মতো কাজ করে। আসল সিঙ্কের জন্য `rsync` ভালো।

### `cp`-এর সম্পূর্ণ অপশন টেবিল

| অপশন | কাজ |
|---|---|
| `-r`, `-R`, `--recursive` | ডিরেক্টরি সহ ভেতরের সব কপি |
| `-i`, `--interactive` | ওভাররাইটের আগে জিজ্ঞেস |
| `-n`, `--no-clobber` | existing ফাইল কখনো ওভাররাইট করবে না |
| `-f`, `--force` | গন্তব্য খোলা না গেলে মুছে আবার চেষ্টা |
| `-v`, `--verbose` | প্রতিটা কপির বিবরণ দেখাও |
| `-p` | mode, ownership, timestamps ধরে রাখো |
| `--preserve=LIST` | নির্দিষ্ট গুণ ধরে রাখো (`mode,ownership,timestamps,links,context,xattr,all`) |
| `-a`, `--archive` | `-dR --preserve=all`: হুবহু কপি |
| `-u`, `--update` | উৎস নতুন হলে বা গন্তব্য না থাকলেই কপি |
| `-l`, `--link` | কপির বদলে হার্ড লিংক বানাও |
| `-s`, `--symbolic-link` | কপির বদলে symlink বানাও |
| `-L`, `--dereference` | symlink-এর লক্ষ্য ফাইল কপি করো |
| `-P`, `--no-dereference` | symlink-কে symlink হিসেবেই কপি করো |
| `-b`, `--backup` | ওভাররাইটের আগে পুরনোটার ব্যাকআপ রাখো (`file~`) |
| `-t DIR`, `--target-directory=DIR` | গন্তব্য ডিরেক্টরি আগে দাও |
| `-T`, `--no-target-directory` | DEST-কে সবসময় ফাইল হিসেবে ধরো |
| `-x`, `--one-file-system` | অন্য ফাইলসিস্টেমের mount point-এ ঢুকবে না |
| `--reflink[=WHEN]` | Btrfs/XFS-এ copy-on-write ক্লোন (তাৎক্ষণিক, জায়গা বাঁচায়) |
| `--sparse=WHEN` | sparse ফাইলের ফাঁকা অংশ সামলানো (`auto`, `always`, `never`) |

> 📝 **মনে রাখো:** coreutils 9.x-এ `-n` এর আচরণ কয়েকবার বদলেছে (ফাইল বাদ পড়লে exit status কী হবে, সেটা নিয়ে)। স্ক্রিপ্টে নিশ্চিত আচরণ চাইলে নতুন coreutils-এ `--update=none` ব্যবহার করতে পারো। মানে "গন্তব্যে থাকলে বাদ দাও, error ছাড়া"।

```bash
$ cp -b notes.txt backup/notes.txt
$ ls backup/
notes.txt  notes.txt~

$ cp -t backup/ a.txt b.txt c.txt         # xargs/find এর সাথে খুব কাজের
```

### কপি করলে ভেতরে কী হয় (inode দৃষ্টিকোণ)

```bash
$ ls -i report.txt
1835093 report.txt
$ cp report.txt copy.txt
$ ls -i report.txt copy.txt
1835093 report.txt
1835240 copy.txt
```

**inode নম্বর আলাদা।** মানে `cp` সম্পূর্ণ নতুন একটা ফাইল তৈরি করেছে। ধাপে ধাপে:

```
  cp report.txt copy.txt
  ─────────────────────────────────────────────────────────────
  1. open("report.txt", O_RDONLY)        ← উৎস পড়ার জন্য খোলো
  2. open("copy.txt", O_WRONLY|O_CREAT)  ← কার্নেল নতুন inode দেয় (1835240)
                                           ডিরেক্টরিতে "copy.txt → 1835240" এন্ট্রি বসে
  3. loop: read() → write()              ← ডেটা নতুন data block-এ লেখা হয়
     (আধুনিক coreutils দ্রুততার জন্য copy_file_range() বা
      Btrfs/XFS-এ reflink ব্যবহার করতে পারে)
  4. -p/-a থাকলে: chmod/chown/utimensat দিয়ে metadata বসাও
  5. close() দুটোই
  ─────────────────────────────────────────────────────────────

   report.txt ──► inode 1835093 ──► [blocks A B C]
   copy.txt   ──► inode 1835240 ──► [blocks X Y Z]  ← সম্পূর্ণ আলাদা কপি
```

ফলাফল:

- একটা বদলালে অন্যটায় কোনো প্রভাব পড়ে না।
- ডিস্কে দ্বিগুণ জায়গা লাগে (reflink ছাড়া)।
- নতুন ফাইলের সময় "এখন" হয়, যদি না `-p`/`-a` দাও।

`cp -l` দিলে নতুন inode তৈরি হয় না। একই inode-এর জন্য আরেকটা নাম (হার্ড লিংক) তৈরি হয়। এ নিয়ে বিস্তারিত [অধ্যায় ১১](11-links-inode.md)-এ।

> 💡 **টিপস:** Btrfs বা XFS ফাইলসিস্টেমে (Fedora-র ডিফল্ট Btrfs) coreutils 9.0 থেকে `cp` ডিফল্টে `--reflink=auto` ব্যবহার করে। এতে ১০ GB ফাইলের কপিও মুহূর্তে হয়ে যায়: শুরুতে দুটো inode একই block শেয়ার করে, আর কেউ কিছু বদলালে তখনই নতুন block লেখা হয় (copy-on-write)। ext4 এটা সাপোর্ট করে না।

---

## ৯.৮ `mv`: সরানো ও নাম বদলানো

লিনাক্সে আলাদা কোনো "rename" কমান্ড লাগে না। নাম বদলানো আর সরানো আসলে একই কাজ: ফাইলের ঠিকানা (path) বদলানো। দুটোই করে `mv` (move)।

### Syntax

```bash
mv [OPTION]... SOURCE DEST
mv [OPTION]... SOURCE... DIRECTORY
mv [OPTION]... -t DIRECTORY SOURCE...
```

### উদাহরণ

```bash
$ mv draft.txt final.txt                 # নাম বদলানো
$ mv final.txt ~/Documents/              # সরানো (নাম একই)
$ mv final.txt ~/Documents/report.txt    # সরানো + নাম বদলানো
$ mv *.mp3 ~/Music/                      # অনেক ফাইল সরানো
$ mv old_project new_project             # ডিরেক্টরির নাম বদল (-r লাগে না!)
```

> 📝 **মনে রাখো:** `cp` আর `rm`-এর মতো `mv`-তে ডিরেক্টরির জন্য `-r` লাগে না। ডিরেক্টরি সরানো মানে শুধু তার একটা এন্ট্রি সরানো, ভেতরের ফাইলগুলোয় হাত দিতে হয় না (একই ফাইলসিস্টেমে হলে)।

### `-i`, `-n`, `-f`: ওভাররাইট নিয়ন্ত্রণ

`cp`-এর মতোই `mv` ডিফল্টে প্রশ্ন না করে গন্তব্যের ফাইল ওভাররাইট করে। ফলে গন্তব্যের পুরনো ফাইল চিরতরে হারিয়ে যায়।

```bash
$ mv -i a.txt b.txt
mv: overwrite 'b.txt'? n

$ mv -n a.txt b.txt        # b.txt থাকলে চুপচাপ বাদ দেবে, কিছুই জিজ্ঞেস করবে না
$ mv -f a.txt b.txt        # কোনো প্রশ্ন ছাড়াই ওভাররাইট (-i alias থাকলেও)
```

`-i`, `-n`, `-f` একসাথে দিলে **শেষে যেটা দেওয়া হয়েছে সেটাই কার্যকর** হয়।

### `mv`-এর অপশন টেবিল

| অপশন | কাজ |
|---|---|
| `-i`, `--interactive` | ওভাররাইটের আগে জিজ্ঞেস |
| `-n`, `--no-clobber` | existing ফাইল কখনো ওভাররাইট করবে না |
| `-f`, `--force` | প্রশ্ন ছাড়া ওভাররাইট |
| `-v`, `--verbose` | কী হচ্ছে দেখাও |
| `-u`, `--update` | উৎস নতুন হলে বা গন্তব্য না থাকলেই সরাও |
| `-b`, `--backup` | ওভাররাইট হওয়া ফাইলের ব্যাকআপ রাখো (`file~`) |
| `-t DIR`, `--target-directory=DIR` | গন্তব্য ডিরেক্টরি আগে দাও |
| `-T`, `--no-target-directory` | DEST-কে ডিরেক্টরি হিসেবে না ধরে নাম হিসেবে ধরো |

```bash
$ mv -v report.txt archive/
renamed 'report.txt' -> 'archive/report.txt'
```

### ভেতরে কী হয়: একই vs ভিন্ন ফাইলসিস্টেম

এই অংশটা খুব গুরুত্বপূর্ণ। এটা বুঝলেই পরিষ্কার হবে কেন ৫০ GB ফাইল এক জায়গায় মুহূর্তে সরে যায়, আর অন্য জায়গায় ১০ মিনিট লাগে।

**কেস ১: একই ফাইলসিস্টেমে (যেমন `/home/rahim/a` থেকে `/home/rahim/b`)**

`mv` কার্নেলের `rename()` system call ডাকে। কার্নেল শুধু:

1. পুরনো ডিরেক্টরি থেকে নামের এন্ট্রিটা সরায়,
2. নতুন ডিরেক্টরিতে একই inode নম্বর দিয়ে নতুন এন্ট্রি বসায়।

ডেটা একটুও নড়ে না। **inode নম্বর একই থাকে**, আর কাজটা মুহূর্তে শেষ হয়, ফাইল যত বড়ই হোক।

```bash
$ ls -i movie.mkv
2097321 movie.mkv
$ mv movie.mkv ~/Videos/
$ ls -i ~/Videos/movie.mkv
2097321 /home/rahim/Videos/movie.mkv      ← একই inode!
```

```
   আগে:  ~/            ──► "movie.mkv" → inode 2097321 ──► [data]
         ~/Videos/     ──► (খালি)

   rename()

   পরে:  ~/            ──► (এন্ট্রি নেই)
         ~/Videos/     ──► "movie.mkv" → inode 2097321 ──► [data]  (একই ডেটা)
```

`rename()` **atomic**। মানে অন্য কোনো প্রোগ্রাম কখনো এমন অবস্থা দেখবে না যেখানে ফাইলটা দুই জায়গাতেই আছে বা কোথাও নেই। এজন্য অনেক প্রোগ্রাম ফাইল নিরাপদে আপডেট করে এভাবে: আগে `file.tmp`-এ লেখে, তারপর `mv file.tmp file`।

**কেস ২: ভিন্ন ফাইলসিস্টেমে (যেমন `/home` থেকে USB ড্রাইভ `/media/usb`)**

inode নম্বর শুধু নিজের ফাইলসিস্টেমের ভেতরে অর্থ বহন করে। তাই `rename()` এখানে `EXDEV` error দেয় ("cross-device link")। তখন `mv` নিজেই:

1. ফাইল **কপি** করে (নতুন ফাইলসিস্টেমে নতুন inode, `cp -a`-এর মতো metadata সহ),
2. কপি সফল হলে উৎসটা **মুছে** দেয়।

তাই এটা ধীর, আর inode নম্বর বদলে যায়।

```
   /home (ext4)                    /media/usb (exFAT)
   inode 2097321 ──copy data──►   নতুন inode 15
        │
        └── কপি সফল হলে unlink (মুছে ফেলা)
```

> ⚠️ **সতর্কতা:** অন্য ফাইলসিস্টেমে বড় ডিরেক্টরি `mv` করার সময় মাঝপথে থামিয়ে দিলে (Ctrl+C, পাওয়ার চলে যাওয়া) কিছু ফাইল এক জায়গায় আর কিছু আরেক জায়গায় থেকে যেতে পারে। গুরুত্বপূর্ণ ডেটার ক্ষেত্রে আগে `cp -a` বা `rsync` দিয়ে কপি করো, যাচাই করো, তারপর উৎস মোছো।

### একসাথে অনেক ফাইলের নাম বদলানো

`mv` একবারে একটা ফাইলের নাম বদলায়। অনেকগুলোর জন্য শেল লুপ ব্যবহার করো:

```bash
# সব .txt কে .md বানাও
$ for f in *.txt; do mv -- "$f" "${f%.txt}.md"; done
```

`${f%.txt}` মানে variable থেকে শেষের `.txt` কেটে ফেলা ([অধ্যায় ৪২](42-shell-scripting-2.md))। এছাড়া `rename` নামে একটা টুল আছে, তবে distro-ভেদে তার দুটো আলাদা সংস্করণ:

| Distro | কমান্ড | উদাহরণ |
|---|---|---|
| Debian/Ubuntu (`rename` প্যাকেজ, Perl-ভিত্তিক) | `rename` | `rename 's/\.txt$/.md/' *.txt` |
| Fedora/RHEL (util-linux) | `rename` | `rename .txt .md *.txt` |

দুটোরই `-n` অপশন আছে, যেটা আসলে কিছু না বদলে শুধু দেখায় কী হবে।

---

## ৯.৯ `rm`: মোছা

### সবার আগে একটা কঠিন সত্য

> ⚠️ **সতর্কতা:** টার্মিনালের `rm` দিয়ে মোছা ফাইল **Trash/Recycle Bin-এ যায় না।** কোনো "Undo" নেই। একবার মুছলে সাধারণত চিরতরে চলে যায়।

### Syntax

```bash
rm [OPTION]... FILE...
```

### মৌলিক ব্যবহার

```bash
$ rm old.txt
$ rm a.txt b.txt c.txt
$ rm *.tmp
```

write-protected ফাইল হলে `rm` প্রশ্ন করে:

```bash
$ chmod 444 locked.txt
$ rm locked.txt
rm: remove write-protected regular file 'locked.txt'? y
```

### ডিরেক্টরি মোছা: `-r`

```bash
$ rm myfolder
rm: cannot remove 'myfolder': Is a directory
$ rm -r myfolder
```

`-r` ভেতরের সবকিছু সহ ডিরেক্টরি মুছে দেয়।

### `-f`, `-i`, `-I`

```bash
$ rm -i *.txt
rm: remove regular file 'a.txt'? y
rm: remove regular empty file 'b.txt'? n

$ rm -I *.log           # ৩টার বেশি ফাইল বা -r হলে একবারই জিজ্ঞেস করে
rm: remove 5 arguments? y

$ rm -f nofile.txt      # ফাইল না থাকলেও error দেবে না, কিছু জিজ্ঞেসও করবে না
```

### `rm`-এর অপশন টেবিল

| অপশন | কাজ |
|---|---|
| `-r`, `-R`, `--recursive` | ডিরেক্টরি ও তার ভেতরের সব মোছো |
| `-f`, `--force` | কোনো প্রশ্ন নয়; না থাকা ফাইলের জন্য error নয় |
| `-i` | প্রতিটা ফাইলের আগে জিজ্ঞেস |
| `-I` | ৩টার বেশি ফাইল বা recursive হলে একবার জিজ্ঞেস (`-i`-এর চেয়ে কম বিরক্তিকর) |
| `-d`, `--dir` | খালি ডিরেক্টরি মোছো |
| `-v`, `--verbose` | কী মুছছে দেখাও |
| `--preserve-root` | `/` মুছতে দেবে না (**ডিফল্টে চালু**) |
| `--no-preserve-root` | `/`-এর সুরক্ষা বন্ধ (**কখনো ব্যবহার কোরো না**) |
| `--one-file-system` | recursive মোছার সময় অন্য ফাইলসিস্টেমের mount point-এ ঢুকবে না |

```bash
$ rm -rv project/
removed 'project/main.c'
removed 'project/README'
removed directory 'project/'
```

### ভেতরে কী হয়: `unlink()`

`rm` আসলে ডেটা "মোছে" না। এটা কার্নেলের `unlink()` (ডিরেক্টরির জন্য `rmdir()`) ডাকে:

```
  rm report.txt
  ──────────────────────────────────────────────────────────
  1. ডিরেক্টরি থেকে "report.txt → 1835093" এন্ট্রি সরাও
  2. inode 1835093-এর link count এক কমাও (1 → 0)
  3. link count == 0 এবং কোনো প্রসেস ফাইলটা খোলা রাখেনি?
        হ্যাঁ → inode ও data block "খালি" হিসেবে চিহ্নিত
        না  → প্রসেস ফাইল বন্ধ না করা পর্যন্ত ডেটা থেকে যায়
  ──────────────────────────────────────────────────────────
```

এর তিনটা বাস্তব ফল আছে:

1. **ফাইল মোছার জন্য দরকার ডিরেক্টরির write পারমিশন, ফাইলের না।** কারণ তুমি আসলে ডিরেক্টরির তালিকা বদলাচ্ছ ([অধ্যায় ১৬](16-permissions.md))।
2. **ডিস্ক ফুল, বড় লগ ফাইল মুছলাম, তবু জায়গা খালি হলো না!** কারণ কোনো প্রসেস (যেমন চালু থাকা সার্ভিস) ফাইলটা এখনো খোলা রেখেছে। `lsof +L1` দিয়ে এমন "deleted কিন্তু খোলা" ফাইল খুঁজে পাবে, আর সার্ভিস রিস্টার্ট করলে জায়গা ফেরত আসবে। এর বিস্তারিত [অধ্যায় ৪৬](46-performance-troubleshooting.md)-এ।
3. **ডেটা তাৎক্ষণিক ওভাররাইট হয় না।** তাই বিশেষ টুল দিয়ে কখনো কখনো উদ্ধার করা যায় (নিশ্চয়তা নেই, SSD-তে TRIM থাকলে প্রায় অসম্ভব)। উল্টো দিকে, গোপন ফাইল নিশ্চিতভাবে মুছতে `shred -u file` আছে, তবে journaling ফাইলসিস্টেম আর SSD-তে এটাও পুরোপুরি নির্ভরযোগ্য না।

### `rm -rf /` এর বিপদ

`rm -rf /` মানে: "রুট থেকে শুরু করে **সবকিছু**, কোনো প্রশ্ন ছাড়া মুছে ফেলো।" এতে পুরো অপারেটিং সিস্টেম, তোমার সব ডেটা, আর mount করা সব ডিস্ক (USB, নেটওয়ার্ক ড্রাইভ) মুছে যেত।

আধুনিক GNU `rm`-এ `--preserve-root` ডিফল্টে চালু থাকে, তাই সরাসরি চালালে আটকে দেয়:

```bash
$ sudo rm -rf /
rm: it is dangerous to operate recursively on '/'
rm: use --no-preserve-root to override this failsafe
```

**কিন্তু এই সুরক্ষা খুব সীমিত।** নিচের কমান্ডগুলো আটকায় না:

```bash
sudo rm -rf /*          # glob expand হয়ে /bin /boot /etc ... হয়ে যায়, "/" আর থাকে না!
sudo rm -rf / home      # space-এর ভুল: আসলে "/" আর "home" দুটো
rm -rf ~ /Downloads     # space-এর ভুল: পুরো home ডিরেক্টরি চলে যাবে
```

স্ক্রিপ্টে সবচেয়ে কুখ্যাত বাগ হলো খালি variable:

```bash
# ভয়ংকর:
rm -rf "$BUILD_DIR"/*
# BUILD_DIR খালি বা unset হলে এটা হয়ে যায়:  rm -rf /*
```

নিরাপদ বিকল্প:

```bash
rm -rf "${BUILD_DIR:?BUILD_DIR is not set}"/*   # খালি হলে স্ক্রিপ্ট error দিয়ে থেমে যায়
```

অথবা স্ক্রিপ্টের শুরুতে `set -u` দাও ([অধ্যায় ৪৩](43-shell-scripting-3.md))।

> ⚠️ **সতর্কতা — `rm`-এর সোনার নিয়ম:**
> 1. `rm -rf` টাইপ করার আগে **একবার থামো**। কোন ডিরেক্টরিতে আছ (`pwd`) আর কী মুছছ, পড়ে দেখো।
> 2. glob থাকলে আগে `ls` দিয়ে দেখো: `ls *.log` → ঠিক থাকলে `rm *.log`।
> 3. `sudo rm -rf` হলো সবচেয়ে বিপজ্জনক সংমিশ্রণ। root-এর কোনো পারমিশন বাধা নেই।
> 4. ইন্টারনেট থেকে কপি করা কমান্ড বুঝে না চালালে কখনো চালাবে না। "`sudo rm -rf / --no-preserve-root` চালাও, তোমার কম্পিউটার দ্রুত হবে" ধরনের মজা অনলাইনে ঘোরে।
> 5. নিয়মিত ব্যাকআপ রাখো ([অধ্যায় ৩৫](35-backup-rsync.md))। এটাই শেষ সুরক্ষা।

### `rm -i` alias: distro-ভেদে পার্থক্য

Fedora/RHEL-এ **root** ইউজারের `~/.bashrc`-এ ডিফল্টে এগুলো থাকে:

```bash
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
```

Ubuntu/Debian-এ এমন alias ডিফল্টে থাকে না। নিজের জন্য যোগ করতে চাইলে `~/.bashrc`-এ `alias rm='rm -I'` লিখতে পারো ([অধ্যায় ৪৪](44-environment-bashrc.md))। তবে alias-এর উপর পুরো ভরসা কোরো না: অন্য মেশিনে গেলে, `sudo` দিয়ে চালালে, বা স্ক্রিপ্টে চালালে alias নাও থাকতে পারে। alias এড়িয়ে আসল কমান্ড চালাতে লেখো `\rm` বা `command rm`।

### নিরাপদ বিকল্প: `trash-cli`

GUI ফাইল ম্যানেজারে Delete চাপলে ফাইল Trash-এ যায়। টার্মিনালে সেই সুবিধা দেয় `trash-cli`। এটা FreeDesktop Trash স্পেসিফিকেশন মেনে চলে, তাই GNOME/KDE-র Trash-এর সাথেই শেয়ার হয়।

```bash
# ইনস্টল
$ sudo apt install trash-cli          # Ubuntu/Debian
$ sudo dnf install trash-cli          # Fedora (RHEL-এ EPEL লাগতে পারে বা pipx install trash-cli)
```

| কমান্ড | কাজ |
|---|---|
| `trash-put FILE...` (বা `trash FILE`) | ফাইল/ডিরেক্টরি Trash-এ পাঠাও |
| `trash-list` | Trash-এ কী আছে দেখাও |
| `trash-restore` | ইন্টার‍্যাক্টিভভাবে ফাইল ফিরিয়ে আনো |
| `trash-rm PATTERN` | Trash থেকে নির্দিষ্ট ফাইল চিরতরে মোছো |
| `trash-empty` | পুরো Trash খালি করো |
| `trash-empty 30` | ৩০ দিনের পুরনো জিনিস মোছো |

```bash
$ trash-put report.txt old_project/
$ trash-list
2025-03-12 11:30:02 /home/rahim/report.txt
2025-03-12 11:30:02 /home/rahim/old_project
$ trash-restore
   0 2025-03-12 11:30:02 /home/rahim/report.txt
   1 2025-03-12 11:30:02 /home/rahim/old_project
What file to restore [0..1]: 0
```

Trash আসলে থাকে `~/.local/share/Trash/`-এ (`files/` আর `info/` সাব-ডিরেক্টরিতে)। GNOME সিস্টেমে বাড়তি কিছু ইনস্টল না করেই `gio trash FILE` দিয়েও একই কাজ করা যায়।

> 💡 **টিপস:** অনেকে `alias rm='echo "Use trash-put or \rm"'` লিখে রাখে, যাতে অভ্যাসবশত `rm` টাইপ করলেও থামতে হয়। নিজের প্রতিদিনের কাজে `trash-put` ব্যবহারের অভ্যাস করো, আর `rm` রাখো স্ক্রিপ্ট ও নিশ্চিত কাজের জন্য।

---

## ৯.১০ `rmdir`: খালি ডিরেক্টরি মোছা

`rmdir` শুধু **খালি** ডিরেক্টরি মোছে। শুনতে সীমাবদ্ধ মনে হলেও এটাই এর সুবিধা: ভুল করে ভরা ডিরেক্টরি মোছার কোনো সম্ভাবনা নেই।

```bash
$ rmdir empty_folder
$ rmdir photos
rmdir: failed to remove 'photos': Directory not empty
```

| অপশন | কাজ |
|---|---|
| `-p`, `--parents` | ডিরেক্টরি আর তার খালি প্যারেন্টগুলোও মোছো |
| `-v`, `--verbose` | কী মুছছে দেখাও |
| `--ignore-fail-on-non-empty` | খালি না হওয়ার কারণে ব্যর্থ হলে error দেখাবে না |

```bash
$ mkdir -p a/b/c
$ rmdir -pv a/b/c
rmdir: removing directory, 'a/b/c'
rmdir: removing directory, 'a/b'
rmdir: removing directory, 'a'
```

একটা ট্রি-র সব খালি ডিরেক্টরি খুঁজে মুছতে: `find . -type d -empty -delete` ([অধ্যায় ১২](12-file-khoja.md))।

> 📝 **মনে রাখো:** hidden ফাইল (যেমন `.gitkeep`) থাকলেও ডিরেক্টরি "খালি" না। `ls -A` দিয়ে দেখো।

---

## ৯.১১ পাথ নিয়ে কাজ: `basename`, `dirname`, `realpath`

এই তিনটা কমান্ড ফাইল তৈরি বা মোছে না। এরা শুধু **path string** কেটে-ছেঁটে দেয়। স্ক্রিপ্টে প্রায়ই লাগে।

```
         /home/rahim/projects/report.final.pdf
         └──────── dirname ────────┘└── basename ──┘
```

### `basename`: শেষ অংশ (নাম)

```bash
$ basename /home/rahim/projects/report.pdf
report.pdf
$ basename /home/rahim/projects/report.pdf .pdf      # suffix কেটে দাও
report
$ basename -s .pdf a.pdf b.pdf                        # অনেকগুলো একসাথে (-s নিজেই -a বোঝায়)
a
b
$ basename /home/rahim/projects/
projects
```

| অপশন | কাজ |
|---|---|
| `-a`, `--multiple` | একাধিক path নাও |
| `-s SUFFIX`, `--suffix=SUFFIX` | শেষের suffix কাটো (`-a` বোঝায়) |
| `-z`, `--zero` | newline-এর বদলে NUL দিয়ে আলাদা করো |

### `dirname`: ডিরেক্টরি অংশ

```bash
$ dirname /home/rahim/projects/report.pdf
/home/rahim/projects
$ dirname report.pdf
.
$ dirname /etc/
/
```

### একটা গুরুত্বপূর্ণ কথা

`basename` আর `dirname` **শুধু লেখা নিয়ে কাজ করে**। ফাইলটা আসলে আছে কি না, সেটা দেখে না:

```bash
$ basename /this/does/not/exist.txt
exist.txt
```

শেলেই এর দ্রুত বিকল্প আছে (নতুন প্রসেস চালাতে হয় না): `${path##*/}` দেয় basename, আর `${path%/*}` দেয় dirname।

### `realpath`: আসল, পূর্ণ (absolute) path

`realpath` relative path, `..`, `.` আর symlink সব সমাধান করে চূড়ান্ত absolute path দেয়।

```bash
$ pwd
/home/rahim/projects
$ realpath ../Documents/./notes.txt
/home/rahim/Documents/notes.txt

$ ln -s /etc/os-release myos
$ realpath myos
/usr/lib/os-release          # (Ubuntu/Fedora-তে /etc/os-release নিজেই একটা symlink)
```

| অপশন | কাজ |
|---|---|
| `-e`, `--canonicalize-existing` | path-এর সব অংশ অবশ্যই থাকতে হবে, না থাকলে error |
| `-m`, `--canonicalize-missing` | কিছুই না থাকলেও চলবে |
| `-s`, `--strip`, `--no-symlinks` | symlink resolve কোরো না, শুধু `.`/`..` ঠিক করো |
| `--relative-to=DIR` | DIR থেকে relative path দেখাও |
| `-q`, `--quiet` | বেশিরভাগ error message চেপে যাও |

```bash
$ realpath --relative-to=/home/rahim /home/rahim/projects/app/main.c
projects/app/main.c
```

`readlink -f FILE` প্রায় একই কাজ করে। স্ক্রিপ্ট নিজের ফোল্ডার খুঁজে পেতে এই ধরনটা বহুল ব্যবহৃত:

```bash
SCRIPT_DIR="$(dirname "$(realpath "$0")")"
```

---

## ৯.১২ আরও কয়েকটা দরকারি সহায়ক কমান্ড

| কমান্ড | কাজ | উদাহরণ |
|---|---|---|
| `install` | কপি + পারমিশন/মালিক একসাথে সেট (ইনস্টল স্ক্রিপ্টে) | `sudo install -m 755 app /usr/local/bin/` |
| `install -d` | ডিরেক্টরি বানাও (mode সহ) | `install -d -m 700 ~/.secret` |
| `mktemp` | নিরাপদ, ইউনিক temp ফাইল বানাও | `tmp=$(mktemp)` |
| `mktemp -d` | ইউনিক temp ডিরেক্টরি | `dir=$(mktemp -d)` |
| `truncate -s SIZE` | ফাইলের সাইজ বদলাও / খালি করো | `truncate -s 0 big.log` |
| `file` | ফাইলের ধরন চেনো | `file photo.jpg` ([অধ্যায় ১০](10-file-dekha.md)) |
| `ln` | লিংক বানাও | [অধ্যায় ১১](11-links-inode.md) |

> 💡 **টিপস:** চালু থাকা সার্ভিসের বড় লগ ফাইল খালি করতে `rm` না করে `truncate -s 0 app.log` (বা `: > app.log`) করো। এতে inode একই থাকে, প্রসেস লিখতে থাকে, আর জায়গা সাথে সাথে খালি হয়।

---

## ৯.১৩ কীবোর্ড শর্টকাট

### টার্মিনালে (bash)

| শর্টকাট | কাজ |
|---|---|
| `Tab` | ফাইল/ডিরেক্টরির নাম সম্পূর্ণ করো (space escape সহ) |
| `Tab Tab` | একাধিক মিল থাকলে সব দেখাও |
| `Alt+.` বা `Esc` তারপর `.` | আগের কমান্ডের শেষ argument বসাও (যেমন `mkdir newdir` এরপর `cd ` + `Alt+.`) |
| `Ctrl+C` | চলমান কমান্ড থামাও (ভুল `cp -r` বা `rm -r` আটকাতে) |
| `Ctrl+U` | কার্সরের আগের পুরো লাইন মুছে দাও (বিপজ্জনক কমান্ড বাতিল করতে) |
| `Ctrl+A` / `Ctrl+E` | লাইনের শুরু / শেষে যাও |
| `Ctrl+W` | কার্সরের আগের একটা শব্দ মোছো |
| `Ctrl+R` | পুরনো কমান্ড খোঁজো |
| `!$` | আগের কমান্ডের শেষ argument (`mkdir proj && cd !$`) |
| `Ctrl+X` তারপর `*` | কার্সরের নিচের glob শব্দটাকে লাইনেই আসল ফাইলের নামে expand করো (চালানোর আগে যাচাই) |
| `Ctrl+X` তারপর `g` | glob কোন কোন ফাইল ধরবে তার তালিকা দেখাও, লাইন না বদলে |

এসব শর্টকাটের পূর্ণ তালিকা আছে [অধ্যায় ৬](06-terminal-shortcuts.md)-এ।

> 💡 **টিপস:** `rm *.log` লিখে Enter চাপার আগে কার্সর `*.log`-এর উপর রেখে `Ctrl+X` তারপর `g` চাপো। দেখবে আসলে কোন কোন ফাইল মুছতে যাচ্ছ। `Ctrl+X` তারপর `*` চাপলে লাইনেই নামগুলো বসে যাবে।

### GUI ফাইল ম্যানেজারে (GNOME Files/Nautilus, KDE Dolphin প্রায় একই)

| শর্টকাট | কাজ |
|---|---|
| `Ctrl+C` / `Ctrl+X` / `Ctrl+V` | কপি / কাট / পেস্ট |
| `F2` | নাম বদলাও |
| `Delete` | Trash-এ পাঠাও |
| `Shift+Delete` | চিরতরে মুছে ফেলো (Trash ছাড়া) |
| `Ctrl+Shift+N` | নতুন ফোল্ডার |
| `Ctrl+Z` | শেষ কাজ undo করো |
| `Ctrl+H` | hidden ফাইল দেখাও/লুকাও |
| `Ctrl+L` | path টাইপ করার বার |
| `Ctrl+A` | সব সিলেক্ট করো |
| `Alt+Enter` (Dolphin) / `Ctrl+I` (Nautilus) | Properties |

ডেস্কটপ পরিবেশ নিয়ে বিস্তারিত আছে [অধ্যায় ৪](04-desktop-environment.md)-এ।

---

## হাতে-কলমে অনুশীলন

নিজের home ডিরেক্টরিতে একটা নিরাপদ "খেলার মাঠ" বানাও। এই ল্যাবে `sudo` লাগবে না।

**ধাপ ১: খেলার মাঠ তৈরি**

```bash
$ mkdir -p ~/lab09 && cd ~/lab09
$ mkdir -pv shop/{products/{phones,laptops},orders,archive}
mkdir: created directory 'shop'
mkdir: created directory 'shop/products'
mkdir: created directory 'shop/products/phones'
mkdir: created directory 'shop/products/laptops'
mkdir: created directory 'shop/orders'
mkdir: created directory 'shop/archive'
```

**ধাপ ২: ফাইল তৈরি ও timestamp**

```bash
$ touch shop/orders/order{1..5}.txt
$ echo "Samsung A55" > shop/products/phones/list.txt
$ touch -d "2024-06-01 09:00" shop/orders/order1.txt
$ ls -l shop/orders/
$ stat -c '%n  mtime=%y' shop/orders/order1.txt
```

খেয়াল করো `order1.txt`-এর তারিখ ২০২৪ দেখাচ্ছে।

**ধাপ ৩: ctime বদলাতে দেখো**

```bash
$ stat -c 'm=%y | c=%z' shop/products/phones/list.txt
$ chmod 600 shop/products/phones/list.txt
$ stat -c 'm=%y | c=%z' shop/products/phones/list.txt
```

mtime একই থাকবে, ctime বদলে যাবে। কেন? (উত্তর: পারমিশন inode-এর metadata, কনটেন্ট না।)

**ধাপ ৪: কপি আর inode**

```bash
$ cd shop/products/phones
$ cp list.txt list-copy.txt
$ cp -p list.txt list-preserve.txt
$ ls -li
```

তিনটা ফাইলের inode নম্বর আলাদা, আর `list-preserve.txt`-এর পারমিশন ও সময় মূলটার মতো। লক্ষ করো।

**ধাপ ৫: mv আর inode**

```bash
$ ls -i list-copy.txt
$ mv -v list-copy.txt ../laptops/moved.txt
$ ls -i ../laptops/moved.txt
```

inode একই! কারণ একই ফাইলসিস্টেমে `rename()` হয়েছে।

**ধাপ ৬: ওভাররাইট থেকে বাঁচা**

```bash
$ echo "old" > a.txt; echo "new" > b.txt
$ cp -i b.txt a.txt          # 'n' দাও
$ cat a.txt                  # old
$ cp -b b.txt a.txt
$ ls                         # a.txt~ ব্যাকআপ দেখবে
$ cat a.txt~                 # old
```

**ধাপ ৭: অদ্ভুত নামের ফাইল**

```bash
$ cd ~/lab09
$ touch 'my file.txt' -- '-weird.txt' 'cost$5.txt'
$ ls
$ rm 'my file.txt'
$ rm -- -weird.txt
$ rm cost\$5.txt
$ ls
```

(`touch -- '-weird.txt'`-এর `--` ছাড়া `touch` ভাবত `-w` একটা অপশন।)

**ধাপ ৮: ডিরেক্টরি কপির ফাঁদ**

```bash
$ cp -r shop shop-backup        # shop-backup ছিল না → কনটেন্ট সরাসরি ভেতরে
$ cp -r shop shop-backup        # এবার আছে → shop-backup/shop তৈরি হবে
$ ls shop-backup
```

দুবার চালিয়ে ফলের পার্থক্য দেখো।

**ধাপ ৯: path টুল**

```bash
$ f=~/lab09/shop/products/phones/list.txt
$ basename "$f"; basename "$f" .txt; dirname "$f"
$ cd shop/products
$ realpath ../../shop/./orders
$ realpath --relative-to="$HOME" "$f"
```

**ধাপ ১০: নিরাপদ মোছা**

```bash
$ sudo apt install trash-cli      # বা: sudo dnf install trash-cli (এই ধাপে sudo লাগবে)
$ trash-put ~/lab09/shop-backup
$ trash-list
$ trash-restore                   # ফিরিয়ে আনো
```

**ধাপ ১১: পরিষ্কার করা**

```bash
$ cd ~
$ rmdir lab09                     # ব্যর্থ হবে, কারণ খালি না
$ ls lab09                        # কী মুছবে আগে দেখে নাও
$ rm -rI lab09
rm: remove 1 argument recursively? y
```

---

## সারসংক্ষেপ

| কাজ | কমান্ড |
|---|---|
| খালি ফাইল / timestamp আপডেট | `touch file` |
| নির্দিষ্ট সময় বসানো | `touch -d "2024-01-15 14:30" file` |
| অন্য ফাইলের সময় কপি | `touch -r ref file` |
| inode-এর সব তথ্য | `stat file` |
| নিজের ফরম্যাটে তথ্য | `stat -c '%n %s %i %a' file` |
| ডিরেক্টরি (মাঝেরগুলো সহ) | `mkdir -p a/b/c` |
| পারমিশন সহ ডিরেক্টরি | `mkdir -m 700 dir` |
| ফাইল কপি | `cp src dst` |
| ডিরেক্টরি কপি | `cp -r src/ dst/` |
| হুবহু কপি (ব্যাকআপ) | `cp -a src dst` |
| জিজ্ঞেস করে / দেখিয়ে কপি | `cp -iv src dst` |
| শুধু নতুন হলে কপি | `cp -u src dst` |
| নাম বদল / সরানো | `mv old new` |
| ওভাররাইট না করে সরানো | `mv -n src dst` |
| ফাইল মোছা | `rm file` |
| ডিরেক্টরি সহ মোছা | `rm -r dir` |
| একবার নিশ্চিত হয়ে মোছা | `rm -rI dir` |
| `-` দিয়ে শুরু হওয়া নাম | `rm -- -file` বা `rm ./-file` |
| Trash-এ পাঠানো | `trash-put file` |
| Trash থেকে ফেরত | `trash-restore` |
| খালি ডিরেক্টরি মোছা | `rmdir dir` / `rmdir -p a/b/c` |
| নাম অংশ | `basename path [suffix]` |
| ডিরেক্টরি অংশ | `dirname path` |
| পূর্ণ absolute path | `realpath path` |
| শেষ argument পুনরায় ব্যবহার | `Alt+.` / `!$` |
| glob কোন ফাইল ধরবে দেখা | `Ctrl+X g` / `Ctrl+X *` |

---

## নিজেকে যাচাই করো

1. atime, mtime আর ctime-এর পার্থক্য কী? `chmod` চালালে কোনটা বদলায়?
2. `touch` দিয়ে কি ctime ইচ্ছামতো সেট করা যায়? কেন?
3. `mkdir -p` সাধারণ `mkdir`-এর চেয়ে দুটো কোন দিক থেকে আলাদা?
4. `backup` ডিরেক্টরি আগে থেকে থাকলে `cp -r projects backup` কী তৈরি করে? না থাকলে কী করে?
5. `cp -p` আর `cp -a`-এর পার্থক্য কী?
6. একই ফাইলসিস্টেমে ১০ GB ফাইল `mv` করতে মুহূর্ত লাগে, কিন্তু USB ড্রাইভে সরাতে অনেক সময় লাগে। কেন? দুই ক্ষেত্রে inode নম্বরের কী হয়?
7. বড় লগ ফাইল `rm` করার পরেও `df` তে জায়গা খালি হলো না। সম্ভাব্য কারণ কী, আর কীভাবে খুঁজবে?
8. `sudo rm -rf /` আধুনিক সিস্টেমে আটকে যায়, কিন্তু `sudo rm -rf /*` আটকায় না। কেন?
9. `-notes.txt` নামের ফাইল মোছার দুটো উপায় লেখো।
10. স্ক্রিপ্টে `rm -rf "$DIR"/*` কেন বিপজ্জনক, আর এটাকে নিরাপদ করবে কীভাবে?

<details>
<summary>উত্তর দেখো</summary>

1. **atime**: কনটেন্ট শেষ কবে পড়া হয়েছে। **mtime**: কনটেন্ট শেষ কবে বদলেছে। **ctime**: inode (metadata বা কনটেন্ট) শেষ কবে বদলেছে। `chmod` শুধু **ctime** বদলায়।
2. না। inode-এ যেকোনো পরিবর্তন হলে (`touch` নিজে atime/mtime বদলালেও) কার্নেল ctime বর্তমান সময়ে সেট করে দেয়। সাধারণ টুল দিয়ে ctime পেছনে নেওয়া যায় না।
3. (ক) পথের মাঝে না থাকা প্যারেন্ট ডিরেক্টরিগুলো বানিয়ে দেয়; (খ) ডিরেক্টরি আগে থেকে থাকলে error দেয় না।
4. থাকলে তৈরি হয় `backup/projects/...`। না থাকলে `backup` নিজেই তৈরি হয় এবং `projects`-এর কনটেন্ট তার ভেতরে যায়। শুধু কনটেন্ট চাইলে লেখো `cp -r projects/. backup/`।
5. `-p` ধরে রাখে mode, ownership আর timestamps। `-a` = `-dR --preserve=all`: recursive, symlink-কে symlink রাখে, হার্ড লিংক, xattr, ACL, SELinux context সহ সব ধরে রাখে।
6. একই ফাইলসিস্টেমে `mv` শুধু `rename()` ডাকে, ডিরেক্টরি এন্ট্রি বদলায়, ডেটা নড়ে না। তাই inode **একই থাকে**। ভিন্ন ফাইলসিস্টেমে `rename()` ব্যর্থ হয় (EXDEV), তাই `mv` পুরো ডেটা কপি করে তারপর উৎস মোছে। তাই **নতুন inode** তৈরি হয় আর সময় লাগে।
7. কোনো চালু প্রসেস ফাইলটা এখনো খোলা রেখেছে, তাই link count 0 হলেও ডেটা মুক্ত হয়নি। `lsof +L1` দিয়ে খুঁজে প্রসেস/সার্ভিস রিস্টার্ট করো। ভবিষ্যতে এমন ক্ষেত্রে `truncate -s 0 file` ব্যবহার করো।
8. `--preserve-root` শুধু আক্ষরিক `/` argument আটকায়। `/*` শেল expand করে `/bin /boot /etc ...` বানিয়ে দেয়, তাই `rm` আর `/` দেখতে পায় না।
9. `rm -- -notes.txt` অথবা `rm ./-notes.txt`।
10. `DIR` খালি বা unset হলে কমান্ডটা হয়ে যায় `rm -rf /*`। নিরাপদ রূপ: `rm -rf "${DIR:?}"/*` (খালি হলে স্ক্রিপ্ট থেমে যায়), অথবা স্ক্রিপ্টে `set -u` দাও এবং আগে `[ -d "$DIR" ]` যাচাই করো।

</details>

---

[← আগের অধ্যায়](08-navigation.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](10-file-dekha.md)

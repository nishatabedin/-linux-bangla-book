# অধ্যায় ১১: inode, হার্ড লিংক ও সফট লিংক

এতদিন আমরা ফাইলকে দেখেছি তার **নাম** দিয়ে: `notes.txt`, `/etc/passwd`, `photo.jpg`। কিন্তু লিনাক্সের ভেতরে ফাইলের আসল পরিচয় তার নাম নয়, একটা **নম্বর**। সেই নম্বরের নাম **inode number**। এই অধ্যায়ের বিষয়টা শুরুতে একটু তাত্ত্বিক মনে হতে পারে। তবু এটা একবার বুঝলে অনেক রহস্য খুলে যাবে। যেমন: `mv` এত দ্রুত কেন, ফাইল মুছলেও ডিস্ক খালি হয় না কেন, `df -h`-এ জায়গা থাকা সত্ত্বেও "No space left on device" কেন আসে, `/bin` আসলে কোথায়, আর `python3` টাইপ করলে কোন ভার্সন চলে।

## এই অধ্যায়ে যা শিখবে

- inode কী, এর ভেতরে কী কী তথ্য থাকে আর কী থাকে **না**
- ডিরেক্টরি আসলে কী: নাম থেকে inode নম্বরে যাওয়ার একটা টেবিল
- `ls -i` আর `stat` দিয়ে inode দেখা
- হার্ড লিংক (`ln`) এবং link count
- সিম্বলিক বা সফট লিংক (`ln -s`)
- হার্ড লিংক ও সফট লিংকের পার্থক্য
- ভাঙা লিংক (broken/dangling link) খুঁজে বের করা ও ঠিক করা
- `readlink` আর `realpath` দিয়ে লিংকের আসল গন্তব্য বের করা
- `rm` চালালে ভেতরে আসলে কী ঘটে, আর মুছে ফেলা কিন্তু এখনো খোলা ফাইল
- `df -i` এবং inode শেষ হয়ে যাওয়ার সমস্যা
- বাস্তব ব্যবহার: `/etc/alternatives`, ভার্সনযুক্ত লাইব্রেরি, `/bin → /usr/bin`, ডিপ্লয়মেন্টের `current` লিংক

---

## ১১.১ একটা উপমা দিয়ে শুরু করি: লাইব্রেরি

ধরো একটা বড় লাইব্রেরিতে গেলে। বই খুঁজতে তুমি প্রথমে **ক্যাটালগ কার্ডে** যাও:

- কার্ডে লেখা থাকে: *"পথের পাঁচালী" → শেলফ নম্বর 4521*
- শেলফ 4521-এ বইয়ের একটা **রেজিস্টার এন্ট্রি** আছে: লেখক কে, কত পৃষ্ঠা, কবে কেনা, কে ধার নিতে পারবে, বইটা কোন তাকে রাখা
- আর আছে **বইটা নিজে**, মানে আসল পাতাগুলো

লিনাক্সের সাথে মিলিয়ে দেখো:

| লাইব্রেরি | লিনাক্স |
|---|---|
| ক্যাটালগ কার্ড (নাম → নম্বর) | **ডিরেক্টরি এন্ট্রি** (filename → inode number) |
| রেজিস্টার এন্ট্রি (মেটাডেটা) | **inode** |
| বইয়ের পাতা | **data blocks** (ডিস্কে আসল কনটেন্ট) |

একই বইয়ের জন্য দুটো ক্যাটালগ কার্ড থাকতে পারে, একটা "পথের পাঁচালী" নামে আর একটা "Pather Panchali" নামে। দুটোই একই শেলফ নম্বর দেখায়। এটাই **হার্ড লিংক**।

আবার এমন একটা কার্ড থাকতে পারে যেটায় শুধু লেখা: *"পথের পাঁচালী কার্ডটা দেখো"*। এটা বইটাকে সরাসরি দেখায় না, আরেকটা **নাম** দেখায়। এটাই **সফট লিংক** (symbolic link)।

> 📝 **মনে রাখো:** লিনাক্সে ফাইলের **নাম** আর ফাইল **নিজে** দুটো আলাদা জিনিস। নাম থাকে ডিরেক্টরিতে, ফাইলের পরিচয় থাকে inode-এ, আর কনটেন্ট থাকে data block-এ।

---

## ১১.২ inode কী?

**inode** (index node, অর্থাৎ সূচক নোড) হলো ফাইলসিস্টেমের ভেতরের একটা ছোট ডেটা স্ট্রাকচার। প্রতিটা ফাইল, ডিরেক্টরি, সিম্বলিক লিংক বা ডিভাইস ফাইলের একটা করে inode থাকে। প্রতিটা inode-এর একটা নম্বর থাকে, যেটা **ওই ফাইলসিস্টেমের ভেতরে** ইউনিক।

ext4-এর মতো প্রচলিত লিনাক্স ফাইলসিস্টেমে একটা inode সাধারণত 256 বাইটের হয়।

### inode-এ কী কী থাকে

```
┌───────────────────────────────────────────────┐
│                 inode #1835123                │
├───────────────────────────────────────────────┤
│ File type    : regular file                   │  ← ফাইল, ডিরেক্টরি, লিংক...
│ Permissions  : rw-r--r-- (0644)               │  ← অধ্যায় ১৬
│ Owner (UID)  : 1000                           │
│ Group (GID)  : 1000                           │
│ Size         : 12 bytes                       │
│ Link count   : 1                              │  ← কয়টা নাম এই inode দেখায়
│ Timestamps   : atime, mtime, ctime, (btime)   │
│ Block count  : 8 (512-byte units)             │
│ Data pointers: extents → block 884736 ...     │──┐
└───────────────────────────────────────────────┘  │
                                                   ▼
                                  ┌──────────────────────────┐
                                  │ Data block(s) on disk    │
                                  │ "Hello Linux\n"          │
                                  └──────────────────────────┘
```

বিস্তারিত:

| তথ্য | ব্যাখ্যা |
|---|---|
| **File type** | regular file, directory, symbolic link, block/character device, FIFO (named pipe), socket |
| **Mode / permissions** | rwx পারমিশন + setuid/setgid/sticky bit ([অধ্যায় ১৬](16-permissions.md), [অধ্যায় ১৭](17-special-permissions-acl.md)) |
| **UID / GID** | মালিক ইউজার ও গ্রুপের **নম্বর** (নাম নয়) |
| **Size** | বাইটে ফাইলের আকার |
| **Link count** | কয়টা ডিরেক্টরি এন্ট্রি (নাম) এই inode-কে দেখায় |
| **atime** | access time: শেষ কবে **পড়া** হয়েছে |
| **mtime** | modify time: শেষ কবে **কনটেন্ট** বদলেছে |
| **ctime** | change time: শেষ কবে **inode** বদলেছে (পারমিশন, মালিক, link count, কনটেন্ট) |
| **btime / crtime** | birth time: কবে তৈরি হয়েছে (ext4, XFS, btrfs সাপোর্ট করে) |
| **Block pointers / extents** | ডিস্কে ডেটা কোথায় আছে তার ঠিকানা |

### inode-এ কী থাকে **না**

> ⚠️ **সতর্কতা:** inode-এ ফাইলের **নাম** থাকে না! নাম থাকে ডিরেক্টরিতে। এই একটা তথ্য বুঝলেই হার্ড লিংক, `mv` আর `rm` সব পরিষ্কার হয়ে যায়।

> 💡 **টিপস:** `ctime`-কে অনেকে "creation time" ভাবে। এটা ভুল। `ctime` মানে **change** time। তৈরির সময় হলো `btime` (Birth)।

---

## ১১.৩ ডিরেক্টরি আসলে কী?

তুমি হয়তো ভাবো ডিরেক্টরি হলো একটা "বাক্স" যার ভেতরে ফাইল রাখা থাকে। আসলে ডিরেক্টরিও একটা **বিশেষ ফাইল**। এর নিজের inode আছে, আর এর কনটেন্ট হলো একটা **টেবিল**: কোন নাম কোন inode নম্বরে যায়।

```
ডিরেক্টরি /home/rahim/project   (নিজের inode: 1835100)
┌──────────────────┬──────────────┐
│ Name             │ Inode number │
├──────────────────┼──────────────┤
│ .                │ 1835100      │ ← নিজে
│ ..               │ 1835001      │ ← প্যারেন্ট (/home/rahim)
│ notes.txt        │ 1835123      │
│ report.pdf       │ 1835124      │
│ src              │ 1835130      │ ← সাব-ডিরেক্টরি
└──────────────────┴──────────────┘
```

তাহলে কার্নেল যখন `/home/rahim/project/notes.txt` খোলে, তখন ভেতরে এই কাজগুলো হয় (একে বলে **path resolution** বা path lookup):

```
"/"           → root ডিরেক্টরি, inode 2 (ext4-এ root সবসময় 2)
  └─ "home"   → / এর টেবিলে খোঁজো       → inode 1572865
      └─ "rahim"   → home এর টেবিলে খোঁজো → inode 1835001
          └─ "project" → ...              → inode 1835100
              └─ "notes.txt" → ...        → inode 1835123
                                              │
                                              ▼
                             inode পড়ো → পারমিশন চেক → data block পড়ো
```

> 💡 **টিপস:** বারবার ডিস্ক থেকে এই টেবিল না পড়তে কার্নেল একটা ক্যাশ রাখে। এর নাম **dentry cache** (directory entry cache)। এজন্য একই path দ্বিতীয়বার অনেক দ্রুত খোলে।

এই মডেল থেকে কয়েকটা জিনিস বোঝা যায়:

1. **ফাইল rename (`mv`) একই ফাইলসিস্টেমে প্রায় তাৎক্ষণিক।** 10 GB ফাইল সরালেও ডেটা নড়ে না, শুধু ডিরেক্টরির টেবিলে একটা লাইন বদলায়। inode নম্বর একই থাকে।
2. **ভিন্ন ফাইলসিস্টেমে `mv` আসলে copy + delete।** নতুন ফাইলসিস্টেমে নতুন inode তৈরি হয়, তাই সময় লাগে।
3. **ফাইল তৈরি বা মোছার জন্য ডিরেক্টরিতে write পারমিশন লাগে**, ফাইলে না। কারণ বদলাচ্ছে ডিরেক্টরির টেবিল। এটা [অধ্যায় ১৬](16-permissions.md)-এ আবার দেখবে।
4. **ডিরেক্টরির `x` পারমিশন** মানে টেবিলে নাম ধরে inode খুঁজে ভেতরে ঢোকার অনুমতি। **`r` পারমিশন** মানে নামগুলোর তালিকা দেখার অনুমতি।

---

## ১১.৪ `ls -i`: inode নম্বর দেখা

`ls`-এর `-i` (`--inode`) অপশন প্রতিটা ফাইলের আগে তার inode নম্বর দেখায়।

**Syntax:**

```bash
ls -i [OPTION]... [FILE]...
```

**উদাহরণ:**

```console
$ mkdir ~/inode-lab && cd ~/inode-lab
$ echo "Hello Linux" > notes.txt
$ echo "second" > other.txt
$ ls -i
1835123 notes.txt  1835124 other.txt
```

`-l`-এর সাথে মিলিয়ে দেখলে আরও বেশি তথ্য পাওয়া যায়:

```console
$ ls -li
total 8
1835123 -rw-r--r-- 1 rahim rahim 12 Nov 12 10:15 notes.txt
1835124 -rw-r--r-- 1 rahim rahim  7 Nov 12 10:15 other.txt
   │         │      │
   │         │      └── link count
   │         └───────── type + permissions
   └─────────────────── inode number
```

| অপশন | কাজ |
|---|---|
| `-i` | inode নম্বর দেখাও |
| `-l` | লম্বা ফরম্যাট (link count সহ) |
| `-d` | ডিরেক্টরির ভেতরের জিনিস না দেখিয়ে ডিরেক্টরিটাকেই দেখাও |
| `-a` | `.` আর `..` সহ লুকানো ফাইলও দেখাও |
| `-L` | সিম্বলিক লিংক হলে লিংকের বদলে টার্গেটের তথ্য দেখাও |

```console
$ ls -lia
total 16
1835100 drwxr-xr-x  2 rahim rahim 4096 Nov 12 10:15 .
1835001 drwxr-x--- 25 rahim rahim 4096 Nov 12 10:14 ..
1835123 -rw-r--r--  1 rahim rahim   12 Nov 12 10:15 notes.txt
1835124 -rw-r--r--  1 rahim rahim    7 Nov 12 10:15 other.txt
```

> 📝 **মনে রাখো:** তোমার সিস্টেমে inode নম্বরগুলো ভিন্ন হবে। এখানের নম্বরগুলো শুধু উদাহরণ।

---

## ১১.৫ `stat`: inode-এর পুরো তথ্য

`ls -l` কিছু তথ্য দেখায়, কিন্তু inode-এর **সব** তথ্য দেখতে চাইলে `stat` লাগবে।

**Syntax:**

```bash
stat [OPTION]... FILE...
```

```console
$ stat notes.txt
  File: notes.txt
  Size: 12        	Blocks: 8          IO Block: 4096   regular file
Device: 8,2	Inode: 1835123     Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/   rahim)   Gid: ( 1000/   rahim)
Access: 2026-11-12 10:15:42.118204511 +0600
Modify: 2026-11-12 10:15:42.118204511 +0600
Change: 2026-11-12 10:15:42.118204511 +0600
 Birth: 2026-11-12 10:15:42.118204511 +0600
```

প্রতিটা লাইনের মানে:

| ফিল্ড | মানে |
|---|---|
| `Size` | বাইটে আকার |
| `Blocks` | কতগুলো 512-বাইট ব্লক বরাদ্দ হয়েছে (12 বাইটের জন্যও পুরো 4 KB ব্লক নেয়, মানে 8 × 512) |
| `IO Block` | ফাইলসিস্টেমের পছন্দের I/O ব্লকের আকার |
| `Device` | কোন ডিভাইসে আছে (major,minor নম্বর) |
| `Inode` | inode নম্বর |
| `Links` | link count |
| `Access: (0644/...)` | পারমিশন, অক্টাল ও rwx দুই রূপেই |
| `Uid` / `Gid` | মালিকের নম্বর ও নাম |
| `Access` / `Modify` / `Change` / `Birth` | চারটা টাইমস্ট্যাম্প |

> 💡 **টিপস:** পুরনো coreutils-এ `Device` লাইনটা `Device: 802h/2050d` রূপে দেখায় (hex আর decimal)। মানে একই, শুধু ফরম্যাট আলাদা। Ubuntu 24.04-এর coreutils 9.x `8,2` রূপে দেখায়।

### `stat`-এর গুরুত্বপূর্ণ অপশন

| অপশন | কাজ |
|---|---|
| `-c FORMAT` / `--format=FORMAT` | নিজের মতো ফরম্যাটে নির্দিষ্ট তথ্য দেখাও |
| `-L` / `--dereference` | সিম্বলিক লিংক হলে টার্গেটের তথ্য দেখাও |
| `-f` / `--file-system` | ফাইলের বদলে ফাইলসিস্টেমের তথ্য দেখাও (মোট/ফ্রি inode সহ) |
| `-t` / `--terse` | সংক্ষিপ্ত এক-লাইনের আউটপুট (স্ক্রিপ্টের জন্য) |

### `-c` ফরম্যাটের দরকারি চিহ্ন

| চিহ্ন | মানে |
|---|---|
| `%n` | ফাইলের নাম |
| `%i` | inode নম্বর |
| `%h` | হার্ড লিংক সংখ্যা |
| `%s` | আকার (বাইট) |
| `%F` | ফাইলের ধরন (লেখায়) |
| `%a` / `%A` | পারমিশন অক্টাল / rwx রূপে |
| `%U` / `%G` | মালিক / গ্রুপের নাম |
| `%d` | ডিভাইস নম্বর (decimal) |
| `%x` `%y` `%z` `%w` | atime, mtime, ctime, birth time |
| `%N` | উদ্ধৃত নাম; লিংক হলে `-> টার্গেট` সহ |

```console
$ stat -c '%n  inode=%i  links=%h  type=%F' notes.txt other.txt
notes.txt  inode=1835123  links=1  type=regular file
other.txt  inode=1835124  links=1  type=regular file
```

ফাইলসিস্টেমের inode পরিসংখ্যান দেখতে:

```console
$ stat -f /
  File: "/"
    ID: 1a2b3c4d5e6f7a8b Namelen: 255     Type: ext2/ext3
Block size: 4096       Fundamental block size: 4096
Blocks: Total: 61202592   Free: 40123456   Available: 37000000
Inodes: Total: 15630336   Free: 14812001
```

> 📝 **মনে রাখো:** ext4-ও `stat -f`-এ `ext2/ext3` হিসেবে দেখায়, কারণ তিনটার magic number একই। ফাইলসিস্টেমের আসল ধরন জানতে `df -T` বা `findmnt` ব্যবহার করো ([অধ্যায় ৩২](32-disk-partition-mount.md))।

---

## ১১.৬ হার্ড লিংক (Hard Link)

### ধারণা

**হার্ড লিংক** মানে একই inode-এর জন্য আরেকটা **নাম**, অর্থাৎ ডিরেক্টরিতে আরেকটা এন্ট্রি। "আসল" আর "কপি" বলে কিছু নেই। সব নাম সমান মর্যাদার। প্রথম যে নামে ফাইল তৈরি হয়েছিল, সেটাও আসলে একটা হার্ড লিংক।

```
 ডিরেক্টরি এন্ট্রি                   inode                  data
┌──────────────────────┐
│ notes.txt → 1835123  │──┐
└──────────────────────┘  │     ┌─────────────────┐     ┌──────────────┐
                          ├────▶│ inode 1835123   │────▶│ Hello Linux  │
┌──────────────────────┐  │     │ Links: 2        │     └──────────────┘
│ backup.txt → 1835123 │──┘     └─────────────────┘
└──────────────────────┘
```

### `ln` কমান্ড

**Syntax:**

```bash
ln [OPTION]... TARGET LINK_NAME        # একটা লিংক বানাও
ln [OPTION]... TARGET... DIRECTORY     # ডিরেক্টরির ভেতরে একই নামে লিংক বানাও
```

> 📝 **মনে রাখো:** ক্রম `cp`-এর মতো: আগে **যেটা আছে**, পরে **নতুন নাম**। মনে রাখার কৌশল: "ln **থেকে** **কী**"।

```console
$ ln notes.txt backup.txt
$ ls -li
total 12
1835123 -rw-r--r-- 2 rahim rahim 12 Nov 12 10:15 backup.txt
1835123 -rw-r--r-- 2 rahim rahim 12 Nov 12 10:15 notes.txt
1835124 -rw-r--r-- 1 rahim rahim  7 Nov 12 10:15 other.txt
```

খেয়াল করো:
- দুটো নামের inode নম্বর **একই** (1835123)
- link count এখন **2**
- আকার, পারমিশন, সময় সবকিছু হুবহু এক, কারণ এগুলো inode-এ থাকে, আর inode একটাই

একটা নাম দিয়ে বদলালে অন্য নামেও বদল দেখা যাবে:

```console
$ echo "added line" >> backup.txt
$ cat notes.txt
Hello Linux
added line
```

একটা নাম মুছলে অন্যটা টিকে থাকে:

```console
$ rm notes.txt
$ ls -li backup.txt
1835123 -rw-r--r-- 1 rahim rahim 23 Nov 12 10:20 backup.txt
$ cat backup.txt
Hello Linux
added line
```

link count আবার 1 হয়ে গেছে, ডেটা অক্ষত।

### `ln`-এর গুরুত্বপূর্ণ অপশন

| অপশন | কাজ |
|---|---|
| `-s`, `--symbolic` | হার্ড লিংকের বদলে সিম্বলিক লিংক বানাও |
| `-f`, `--force` | একই নামে আগে কিছু থাকলে সেটা মুছে নতুন লিংক বানাও |
| `-i`, `--interactive` | আগের ফাইল মোছার আগে জিজ্ঞেস করো |
| `-n`, `--no-dereference` | LINK_NAME যদি ডিরেক্টরির দিকে সিমলিংক হয়, তাকে সাধারণ ফাইলের মতো ধরো (`-f`-এর সাথে ব্যবহার হয়) |
| `-T`, `--no-target-directory` | LINK_NAME-কে সবসময় ফাইলের নাম ধরো, ডিরেক্টরি নয় |
| `-r`, `--relative` | সিম্বলিক লিংকে relative path বসাও (`-s`-এর সাথে) |
| `-v`, `--verbose` | কী বানানো হলো দেখাও |
| `-b`, `--backup` | আগের ফাইল মুছে না দিয়ে `~` যোগ করে ব্যাকআপ রাখো |
| `-t DIR`, `--target-directory=DIR` | কোন ডিরেক্টরিতে লিংক বানাবে আগে বলে দাও |
| `-L` / `-P` | হার্ড লিংক করার সময় TARGET সিমলিংক হলে তার টার্গেটে (`-L`) নাকি সিমলিংকে নিজেই (`-P`) লিংক করবে |

```console
$ ln -v backup.txt copy2.txt
'copy2.txt' => 'backup.txt'
```

### link count-এর রহস্য: ডিরেক্টরি

```console
$ mkdir demo
$ ls -ld demo
drwxr-xr-x 2 rahim rahim 4096 Nov 12 10:25 demo
```

খালি ডিরেক্টরির link count **2** কেন? কারণ দুটো নাম এই inode-কে দেখায়:
1. প্যারেন্ট ডিরেক্টরিতে `demo` এন্ট্রি
2. `demo`-র ভেতরে `.` এন্ট্রি

এবার একটা সাব-ডিরেক্টরি বানাও:

```console
$ mkdir demo/sub
$ ls -ld demo
drwxr-xr-x 3 rahim rahim 4096 Nov 12 10:26 demo
```

3 হলো, কারণ `demo/sub/..`-ও `demo`-কে দেখায়। সূত্রটা তাহলে:

```
ডিরেক্টরির link count = 2 + (সরাসরি সাব-ডিরেক্টরির সংখ্যা)
```

> 💡 **টিপস:** এটা ext4 ও XFS-এর আচরণ। btrfs-এ ডিরেক্টরির link count সবসময় 1 দেখায়, তাই সব ফাইলসিস্টেমে এই হিসাবের উপর নির্ভর কোরো না।

### হার্ড লিংকের সীমাবদ্ধতা

**১. ডিরেক্টরির হার্ড লিংক করা যায় না:**

```console
$ ln demo demo-link
ln: demo: hard link not allowed for directory
```

কেন? ডিরেক্টরির হার্ড লিংক অনুমতি পেলে ফাইলসিস্টেম ট্রিতে **লুপ** তৈরি হতে পারত (A-র ভেতরে B, B-র ভেতরে A)। তখন `find` বা `du` অনন্তকাল ঘুরত, আর `..` কোন প্যারেন্টকে বোঝাবে সেটাও অস্পষ্ট হতো। `.` আর `..` শুধু কার্নেল নিজে বানায়।

**২. ফাইলসিস্টেমের সীমা পার হওয়া যায় না:**

```console
$ ln ~/inode-lab/backup.txt /mnt/usb/backup.txt
ln: failed to create hard link '/mnt/usb/backup.txt' => '/home/rahim/inode-lab/backup.txt': Invalid cross-device link
```

কেন? inode নম্বর শুধু **একটা ফাইলসিস্টেমের ভেতরে** ইউনিক। USB ড্রাইভের ডিরেক্টরি টেবিলে "1835123" লিখলে সেটা USB-এর নিজের 1835123 নম্বর inode-কে বোঝাবে, যেটা সম্পূর্ণ অন্য ফাইল।

> 📝 **মনে রাখো:** দুটো ফাইল একই কিনা নিশ্চিত হতে inode নম্বরের সাথে device নম্বরও মিলাতে হয়: `stat -c '%d:%i' FILE`।

**৩. নিরাপত্তা সীমা (protected_hardlinks):** আধুনিক distro-তে `fs.protected_hardlinks = 1` চালু থাকে। এর মানে সাধারণ ইউজার এমন ফাইলের হার্ড লিংক বানাতে পারে না যেটার মালিক সে নয় এবং যেটা পড়া/লেখার অনুমতিও তার নেই। এতে `/tmp`-এ লিংক বানিয়ে প্রিভিলেজড প্রোগ্রামকে ধোঁকা দেওয়ার আক্রমণ বন্ধ হয়।

```console
$ sysctl fs.protected_hardlinks fs.protected_symlinks
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
```

### হার্ড লিংক খোঁজা

একই inode-এর সব নাম খুঁজতে:

```console
$ find ~/inode-lab -samefile backup.txt
/home/rahim/inode-lab/backup.txt
/home/rahim/inode-lab/copy2.txt

$ find ~/inode-lab -inum 1835123
/home/rahim/inode-lab/backup.txt
/home/rahim/inode-lab/copy2.txt

$ find ~/inode-lab -type f -links +1     # একাধিক নাম আছে এমন ফাইল
```

`find` নিয়ে বিস্তারিত আছে [অধ্যায় ১২](12-file-khoja.md)-এ।

> ⚠️ **সতর্কতা:** কিছু টুল ফাইল সেভ করার সময় **নতুন inode** তৈরি করে পুরনোটার জায়গায় rename করে বসায়। যেমন `sed -i` এবং অনেক এডিটরের "safe save"। এতে হার্ড লিংক ভেঙে যায়: তুমি যে নাম দিয়ে এডিট করেছ সেটা নতুন inode পায়, আর অন্য নামটা পুরনো কনটেন্ট নিয়ে পড়ে থাকে। `echo >>` বা `>`-এর মতো in-place লেখা হার্ড লিংক রক্ষা করে।

```console
$ ls -i backup.txt copy2.txt
1835123 backup.txt  1835123 copy2.txt
$ sed -i 's/Hello/Hi/' backup.txt
$ ls -i backup.txt copy2.txt
1835140 backup.txt  1835123 copy2.txt      # লিংক ভেঙে গেছে!
```

---

## ১১.৭ সিম্বলিক লিংক / সফট লিংক (Symbolic Link)

### ধারণা

**সিম্বলিক লিংক** (symlink বা soft link) একটা **আলাদা ফাইল**। এর নিজের inode আছে, আর এর কনটেন্ট শুধু একটা **path লেখা**। কার্নেল এই ফাইলের মধ্য দিয়ে যাওয়ার সময় সেই লেখা path-টা অনুসরণ করে। Windows-এর "shortcut"-এর মতো, তবে অনেক বেশি স্বচ্ছভাবে কাজ করে: প্রায় সব প্রোগ্রাম এটাকে আসল ফাইলের মতোই ব্যবহার করে।

```
 ডিরেক্টরি এন্ট্রি              inode 1835150 (symlink)        inode 1835123 (regular)
┌─────────────────────┐       ┌──────────────────────┐
│ link.txt → 1835150  │──────▶│ type: symbolic link  │
└─────────────────────┘       │ content:"backup.txt" │
                              └──────────┬───────────┘
                                         │ (নাম ধরে আবার খোঁজা)
┌─────────────────────┐                  ▼
│ backup.txt → 1835123│───────────────────────────────▶ ┌──────────────┐
└─────────────────────┘                                 │ Links: 1     │──▶ data
                                                        └──────────────┘
```

খেয়াল করো: symlink টার্গেটের inode-কে চেনে না, শুধু **নামটা** জানে। এজন্য টার্গেটের link count বাড়ে না।

### তৈরি করা: `ln -s`

```bash
ln -s TARGET LINK_NAME
```

```console
$ ln -s backup.txt link.txt
$ ls -li
total 8
1835123 -rw-r--r-- 2 rahim rahim 23 Nov 12 10:20 backup.txt
1835123 -rw-r--r-- 2 rahim rahim 23 Nov 12 10:20 copy2.txt
1835150 lrwxrwxrwx 1 rahim rahim 10 Nov 12 10:30 link.txt -> backup.txt
```

আউটপুটটা পড়ো:
- প্রথম অক্ষর `l` মানে সিম্বলিক লিংক
- পারমিশন সবসময় `rwxrwxrwx`। লিনাক্স symlink-এর নিজের পারমিশন **উপেক্ষা করে**, আসল পারমিশন চেক হয় টার্গেটের।
- আকার **10**, কারণ `backup.txt` লেখাটায় 10টা অক্ষর!
- `-> backup.txt` দেখায় কোথায় নির্দেশ করছে
- টার্গেটের link count এখনো 2 (`backup.txt` + `copy2.txt`), symlink-এর জন্য বাড়েনি

> 💡 **টিপস:** ext4-এ টার্গেট path 60 বাইটের কম হলে সেটা আলাদা data block-এ না রেখে inode-এর ভেতরেই রাখা হয়। একে **fast symlink** বলে। তাই ছোট symlink-এর `stat`-এ `Blocks: 0` দেখতে পারো।

```console
$ cat link.txt
Hello Linux
added line
```

### ডিরেক্টরির সিম্বলিক লিংক

হার্ড লিংকে যা নিষেধ, symlink-এ তা চলে:

```console
$ ln -s /var/log logs
$ ls -l logs
lrwxrwxrwx 1 rahim rahim 8 Nov 12 10:32 logs -> /var/log
$ cd logs && pwd
/home/rahim/inode-lab/logs
$ pwd -P
/var/log
```

`pwd` তোমাকে "লজিক্যাল" path দেখায় (যে পথে এসেছ), আর `pwd -P` দেখায় "ফিজিক্যাল" আসল path ([অধ্যায় ৮](08-navigation.md))।

### Absolute বনাম Relative symlink

এটা অনেক নতুনদের বিভ্রান্ত করে, তাই মন দিয়ে পড়ো:

> 📝 **মনে রাখো:** relative symlink-এর টার্গেট path হিসাব হয় **লিংকটা যে ডিরেক্টরিতে আছে সেখান থেকে**, তুমি এখন যেখানে দাঁড়িয়ে আছ সেখান থেকে নয়।

```console
$ cd ~
$ ln -s inode-lab/backup.txt inode-lab/wrong-link
$ cat inode-lab/wrong-link
cat: inode-lab/wrong-link: No such file or directory
```

কেন ভাঙল? লিংকটা আছে `~/inode-lab/`-এ, আর তার ভেতরে লেখা `inode-lab/backup.txt`। কার্নেল খুঁজছে `~/inode-lab/inode-lab/backup.txt`, যেটা নেই!

সমাধান তিনটা:

```console
$ ln -sf backup.txt inode-lab/wrong-link               # টার্গেট লিংকের ডিরেক্টরি থেকে হিসাব করে লেখো
$ ln -sf ~/inode-lab/backup.txt inode-lab/wrong-link   # absolute path দাও
$ ln -srf inode-lab/backup.txt inode-lab/wrong-link    # -r দিলে ln নিজেই relative বানিয়ে নেয়
```

| ধরন | উদাহরণ | সুবিধা | অসুবিধা |
|---|---|---|---|
| Absolute | `-> /home/rahim/data/file` | লিংক যেখানেই সরাও, কাজ করবে | টার্গেটের ফোল্ডার অন্য জায়গায় সরালে বা অন্য মেশিনে কপি করলে ভাঙবে |
| Relative | `-> ../data/file` | পুরো প্রজেক্ট ফোল্ডার একসাথে সরালে/কপি করলে কাজ করে | শুধু লিংকটা আলাদা করে সরালে ভাঙবে |

> 💡 **টিপস:** প্রজেক্ট, git রিপো বা ব্যাকআপের ভেতরের লিংকে relative path ব্যবহার করো। সিস্টেম-ব্যাপী জায়গা দেখাতে (যেমন `/opt/app/current`) absolute ভালো।

### symlink-এর উপর কমান্ডের আচরণ

কোন কমান্ড লিংক "অনুসরণ" (dereference) করে আর কোনটা লিংককে নিজেকে নিয়েই কাজ করে, এটা জানা জরুরি:

| কমান্ড | ডিফল্ট আচরণ | উল্টোটা করতে |
|---|---|---|
| `cat`, `less`, `vim`, `cd` | টার্গেট অনুসরণ করে | (প্রযোজ্য নয়) |
| `ls -l` | লিংক নিজেকে দেখায় | `ls -lL` (টার্গেট দেখাও) |
| `stat` | লিংক নিজেকে দেখায় | `stat -L` |
| `rm` | **লিংক** মোছে, টার্গেট নয় | (প্রযোজ্য নয়) |
| `chmod` | **টার্গেটের** পারমিশন বদলায় | লিংকের পারমিশন বদলানোর অর্থ নেই |
| `chown` | টার্গেটের মালিক বদলায় | `chown -h` দিলে লিংকের মালিক বদলায় |
| `cp` (একটা ফাইল) | টার্গেটের কনটেন্ট কপি করে | `cp -P` দিলে লিংকটাই কপি করে |
| `cp -r` / `cp -a` | ভেতরের লিংককে লিংক হিসেবেই কপি করে | `cp -rL` দিলে অনুসরণ করে |
| `find` | লিংক অনুসরণ করে না | `find -L` |
| `tar` | লিংককে লিংক হিসেবে রাখে | `tar -h` (`--dereference`) |

### symlink-এর লুপ

symlink ডিরেক্টরি দেখাতে পারে, তাই লুপ তৈরি সম্ভব:

```console
$ ln -s loopB loopA
$ ln -s loopA loopB
$ cat loopA
cat: loopA: Too many levels of symbolic links
```

কার্নেল একটা path resolve করতে সর্বোচ্চ 40টা symlink অনুসরণ করে, তারপর `ELOOP` এরর দেয়। এভাবে অনন্ত লুপ থেকে সিস্টেম রক্ষা পায়।

---

## ১১.৮ হার্ড লিংক বনাম সফট লিংক: পার্থক্যের টেবিল

| বৈশিষ্ট্য | হার্ড লিংক | সিম্বলিক (সফট) লিংক |
|---|---|---|
| কমান্ড | `ln target name` | `ln -s target name` |
| আসলে কী | একই inode-এর আরেকটা নাম | আলাদা ফাইল, যার কনটেন্ট একটা path |
| নিজের inode | না, টার্গেটের সাথে **একই** | হ্যাঁ, **আলাদা** |
| `ls -l`-এ চেনার উপায় | সাধারণ ফাইলের মতো, link count > 1 | `l` টাইপ, `-> target` |
| টার্গেটের link count বাড়ে? | হ্যাঁ | না |
| ডিরেক্টরির লিংক | না | হ্যাঁ |
| ভিন্ন ফাইলসিস্টেম/পার্টিশন | না | হ্যাঁ |
| টার্গেট মুছলে | ডেটা টিকে থাকে | লিংক **ভেঙে** যায় (dangling) |
| টার্গেট rename/move করলে | কোনো সমস্যা নেই | লিংক ভেঙে যায় |
| অস্তিত্বহীন জিনিসের দিকে বানানো | সম্ভব নয় | সম্ভব |
| আকার | টার্গেটের সমান (একই inode) | টার্গেট path-এর অক্ষরসংখ্যা |
| পারমিশন | টার্গেটের সাথে এক (একই inode) | সবসময় `rwxrwxrwx`, উপেক্ষিত |
| গতি | একটু দ্রুত (সরাসরি inode) | একটা অতিরিক্ত lookup লাগে |
| সাধারণ ব্যবহার | ডিডুপ্লিকেশন, স্ন্যাপশট ব্যাকআপ | শর্টকাট, ভার্সন সুইচ, কনফিগ, লাইব্রেরি |

> 💡 **টিপস:** দৈনন্দিন কাজে ৯৫% সময় তোমার দরকার হবে **সফট লিংক**। হার্ড লিংক বিশেষ কাজের জন্য, যেমন `rsync --link-dest` দিয়ে জায়গা-বাঁচানো ব্যাকআপ ([অধ্যায় ৩৫](35-backup-rsync.md))।

---

## ১১.৯ ভাঙা লিংক (Broken / Dangling Link)

symlink-এর টার্গেট মুছে বা সরিয়ে ফেললে লিংকটা কোথাও নির্দেশ করে না। একে **broken** বা **dangling** লিংক বলে।

```console
$ echo data > target.txt
$ ln -s target.txt pointer
$ rm target.txt
$ ls -l pointer
lrwxrwxrwx 1 rahim rahim 10 Nov 12 10:40 pointer -> target.txt
$ cat pointer
cat: pointer: No such file or directory
```

মজার ব্যাপার হলো `ls` বলছে `pointer` আছে, কিন্তু `cat` বলছে "No such file"। কারণ লিংকটা আছে, তার টার্গেট নেই। রঙিন টার্মিনালে `ls` সাধারণত ভাঙা লিংক **লাল** (কখনো জ্বলা-নেভা) রঙে দেখায়, আর ভালো লিংক সায়ান/আকাশি রঙে।

### চেনার উপায়

```console
$ file pointer
pointer: broken symbolic link to target.txt

$ test -e pointer && echo "ok" || echo "broken or missing"
broken or missing
$ test -L pointer && echo "it is a symlink"
it is a symlink
```

`test -e` (বা `[ -e ]`) লিংক অনুসরণ করে, আর `test -L` (বা `-h`) শুধু দেখে লিংকটা আছে কিনা। দুটো মিলিয়ে বোঝা যায় লিংক ভাঙা কিনা। এই টেস্টগুলো শেল স্ক্রিপ্টিংয়ে কাজে লাগবে ([অধ্যায় ৪১](41-shell-scripting-1.md))।

### সব ভাঙা লিংক খুঁজে বের করা

```console
$ find ~/inode-lab -xtype l
/home/rahim/inode-lab/pointer

$ find -L ~/inode-lab -type l          # একই কাজের আরেক উপায়
```

`-xtype l` মানে "লিংক অনুসরণ করার পরেও যেটা এখনো লিংক", অর্থাৎ যার টার্গেট খুঁজে পাওয়া যায়নি।

### ঠিক করা বা মোছা

```console
$ ln -sfn /new/place/target.txt pointer     # নতুন টার্গেটে নির্দেশ করাও
$ find ~/inode-lab -xtype l -delete          # সব ভাঙা লিংক মুছে দাও (আগে -delete ছাড়া চালিয়ে দেখে নাও!)
```

> 💡 **টিপস:** ভাঙা লিংক সবসময় ভুল নয়। কিছু প্রোগ্রাম ইচ্ছাকৃতভাবে বানায়, যেমন Firefox প্রোফাইলের `lock` ফাইল একটা symlink যার "টার্গেট" আসলে IP আর PID লেখা একটা স্ট্রিং। মোছার আগে দেখে নাও।

---

## ১১.১০ `readlink` ও `realpath`: লিংকের গন্তব্য বের করা

### `readlink`

**Syntax:**

```bash
readlink [OPTION]... FILE...
```

অপশন ছাড়া চালালে symlink-এর ভেতরে **যা লেখা আছে ঠিক তাই** দেখায়, এক ধাপ, কোনো resolve ছাড়া:

```console
$ ln -s backup.txt link.txt
$ ln -s link.txt link2.txt
$ readlink link2.txt
link.txt
$ readlink -f link2.txt
/home/rahim/inode-lab/backup.txt
```

| অপশন | কাজ |
|---|---|
| (কিছু না) | এক ধাপ: লিংকের ভেতরের লেখা দেখাও। ফাইলটা লিংক না হলে কিছু না দেখিয়ে exit code 1 দেয় |
| `-f`, `--canonicalize` | সব লিংক অনুসরণ করে পুরো absolute path দাও; **শেষ অংশটা** ছাড়া বাকি সব path-এর অস্তিত্ব থাকতে হবে |
| `-e`, `--canonicalize-existing` | `-f`-এর মতো, তবে **সব** অংশের অস্তিত্ব থাকতে হবে |
| `-m`, `--canonicalize-missing` | `-f`-এর মতো, তবে কোনো অংশের অস্তিত্ব না থাকলেও চলবে |
| `-n`, `--no-newline` | শেষে newline দিও না |
| `-v`, `--verbose` | এরর মেসেজ দেখাও |

### `realpath`

`realpath` হলো `readlink -f`-এর আধুনিক ও বেশি সুবিধার সংস্করণ:

```console
$ realpath link2.txt
/home/rahim/inode-lab/backup.txt
$ realpath -s link2.txt                        # symlink resolve কোরো না, শুধু path পরিষ্কার করো
/home/rahim/inode-lab/link2.txt
$ realpath --relative-to=/home/rahim /var/log
../../var/log
```

বাস্তব উদাহরণ: সিস্টেমে কোন `python3` আসলে চলে?

```console
$ which python3
/usr/bin/python3
$ ls -l /usr/bin/python3
lrwxrwxrwx 1 root root 10 Apr  2  2024 /usr/bin/python3 -> python3.12
$ readlink -f "$(which python3)"
/usr/bin/python3.12
```

> 💡 **টিপস:** স্ক্রিপ্টের ভেতরে স্ক্রিপ্ট নিজে কোন ফোল্ডারে আছে জানতে এই কৌশলটা খুব প্রচলিত: `SCRIPT_DIR="$(dirname "$(realpath "$0")")"`। স্ক্রিপ্টটা symlink দিয়ে চালালেও এটা সঠিক উত্তর দেয়।

`namei` কমান্ড path-এর প্রতিটা ধাপ ভেঙে দেখায়, তাই কোথায় লিংক আছে বোঝা সহজ হয় (`util-linux` প্যাকেজে আছে):

```console
$ namei /usr/bin/python3
f: /usr/bin/python3
 d /
 d usr
 d bin
 l python3 -> python3.12
   - python3.12
```

`namei -l` দিলে প্রতিটা ধাপের পারমিশনও দেখায়, যেটা "Permission denied" কোথা থেকে আসছে খুঁজতে দারুণ কাজে লাগে।

---

## ১১.১১ ফাইল মুছলে আসলে কী হয়?

### `rm` = unlink

লিনাক্সে "ফাইল মোছা" বলে আলাদা কোনো system call নেই! `rm` আসলে `unlink()` (বা `unlinkat()`) system call চালায়। এটা শুধু:

1. ডিরেক্টরির টেবিল থেকে **নামটা** সরিয়ে দেয়
2. inode-এর **link count এক কমায়**

```
rm notes.txt
     │
     ▼
ডিরেক্টরি থেকে "notes.txt" এন্ট্রি মোছো
     │
     ▼
inode link count -= 1
     │
     ▼
link count == 0 ? ──No──▶ শেষ (অন্য নাম দিয়ে ফাইল এখনো আছে)
     │Yes
     ▼
কোনো প্রসেস কি ফাইলটা খোলা রেখেছে? ──Yes──▶ অপেক্ষা: শেষ প্রসেস close করলে মুক্ত হবে
     │No
     ▼
inode ও data block "ফ্রি" হিসেবে চিহ্নিত করো
```

> 📝 **মনে রাখো:** একটা ফাইলের ডেটা মুক্ত হয় তখনই, যখন **দুটো** শর্ত পূরণ হয়: (১) link count = 0, এবং (২) কোনো প্রসেস ফাইলটা খোলা রাখেনি।

> ⚠️ **সতর্কতা:** `rm` ডেটা "শূন্য" দিয়ে মুছে দেয় না, শুধু ব্লকগুলো ফ্রি চিহ্নিত করে। নতুন ডেটা ওপরে লেখা না হওয়া পর্যন্ত বিশেষ টুল দিয়ে উদ্ধার সম্ভব হতে পারে। আবার উল্টোদিকে, লিনাক্সে কোনো **Recycle Bin নেই**। `rm` করা ফাইল সাধারণ উপায়ে ফিরে আসে না। (GUI ফাইল ম্যানেজারের "Trash" আলাদা ব্যাপার: সেটা আসলে `~/.local/share/Trash`-এ `mv` করে।)

`unlink` নামে একটা কমান্ডও আছে, যেটা ঠিক একটা ফাইলের জন্য এই system call চালায়:

```console
$ unlink pointer
```

### মুছে ফেলা কিন্তু খোলা ফাইল: "ডিস্ক ভরা, কিন্তু ফাইল তো নেই!"

সার্ভারে এটা খুব পরিচিত সমস্যা। একটা 20 GB লগ ফাইল `rm` করলে, কিন্তু `df -h` এখনো বলছে ডিস্ক ভরা। কারণ যে প্রসেস লগটা লিখছিল (যেমন nginx বা কোনো অ্যাপ), সে এখনো ফাইলটা খোলা রেখেছে। link count 0, কিন্তু open file descriptor আছে, তাই ডেটা মুক্ত হয়নি।

হাতে-কলমে দেখো:

```console
$ dd if=/dev/zero of=big.bin bs=1M count=200 status=none
$ df -h . | tail -1
/dev/sda2       234G   80G  142G  37% /
$ tail -f big.bin > /dev/null &
[1] 45210
$ rm big.bin
$ df -h . | tail -1
/dev/sda2       234G   80G  142G  37% /       ← জায়গা ফেরত আসেনি!
```

খুঁজে বের করো কোন প্রসেস মুছে ফেলা ফাইল ধরে আছে:

```console
$ lsof +L1
COMMAND   PID  USER   FD   TYPE DEVICE  SIZE/OFF NLINK    NODE NAME
tail    45210 rahim    3r   REG    8,2 209715200     0 1835160 /home/rahim/inode-lab/big.bin (deleted)
```

`+L1` মানে "link count 1-এর কম", অর্থাৎ 0। `NLINK 0` আর `(deleted)` দেখো। (`lsof` ইনস্টল না থাকলে `sudo apt install lsof` বা `sudo dnf install lsof`।)

`/proc` দিয়েও দেখা যায় ([অধ্যায় ৪৭](47-kernel-proc-sys.md)):

```console
$ ls -l /proc/45210/fd
lr-x------ 1 rahim rahim 64 Nov 12 10:50 3 -> '/home/rahim/inode-lab/big.bin (deleted)'
```

সমাধানের তিনটা উপায়:

```console
$ kill 45210                      # ১. প্রসেস বন্ধ করো (বা সার্ভিস restart: অধ্যায় ২৬)
$ : > /proc/45210/fd/3            # ২. প্রসেস না থামিয়ে ফাইলটা truncate করো (খালি করো)
$ cp /proc/45210/fd/3 ~/rescued.bin   # ৩. ভুল করে মুছেছ? এখনো উদ্ধার করা যায়!
```

> 💡 **টিপস:** ভবিষ্যতে বড় লগ খালি করতে `rm` না করে `truncate -s 0 app.log` বা `: > app.log` ব্যবহার করো। এতে inode একই থাকে, প্রসেস লিখে যেতে পারে, আর জায়গা সাথে সাথে ফেরত আসে। লগ রোটেশনের জন্য `logrotate` আছে ([অধ্যায় ২৮](28-logs.md))।

### এই আচরণ কেন ভালো?

এটা আসলে লিনাক্সের একটা শক্তিশালী ডিজাইন। প্যাকেজ আপডেট করার সময় চলমান প্রোগ্রামের binary ফাইল বদলানো হয়। নতুন ফাইল নতুন inode পায় আর পুরনো নাম তার জায়গা নেয়। পুরনো চলমান প্রসেস তার পুরনো inode নিয়ে নির্বিঘ্নে চলতে থাকে, ক্র্যাশ করে না। এজন্যই লিনাক্সে বেশিরভাগ আপডেটের পর রিবুট লাগে না (শুধু সার্ভিস restart করলেই নতুন ভার্সন চালু হয়)। Windows-এ চলমান ফাইল সাধারণত লক থাকে, তাই সেখানে প্রায়ই রিবুট লাগে।

### অদ্ভুত নামের ফাইল inode দিয়ে মোছা

কখনো কখনো এমন ফাইলের নাম পাবে যা টাইপ করা কঠিন (যেমন `-rf` নামে একটা ফাইল, বা অদৃশ্য অক্ষরসহ নাম)। inode নম্বর দিয়ে মুছে ফেলো:

```console
$ ls -li
1835170 -rw-r--r-- 1 rahim rahim 0 Nov 12 11:00 ?????
$ find . -maxdepth 1 -inum 1835170 -delete
```

(বিকল্প: `rm -- '-rf'` বা `rm ./-rf`।)

---

## ১১.১২ inode শেষ হয়ে যাওয়া: `df -i`

### সমস্যাটা কী

ext4-এ inode-এর মোট সংখ্যা **ফাইলসিস্টেম তৈরির (`mkfs`) সময়েই নির্দিষ্ট হয়ে যায়**। ডিফল্টে প্রতি 16 KB জায়গার জন্য একটা inode বানানো হয়। লক্ষ লক্ষ ছোট ফাইল (সেশন ফাইল, ক্যাশ, মেইল কিউ, ছোট থাম্বনেইল) থাকলে ডিস্কে প্রচুর জায়গা খালি থাকা সত্ত্বেও **inode শেষ হয়ে যেতে পারে**। তখন নতুন ফাইল তৈরি করা যায় না:

```console
$ touch newfile
touch: cannot touch 'newfile': No space left on device
$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       234G   61G  162G  28% /          ← জায়গা তো আছে!?
```

উপমা: একটা পার্কিং লটে প্রচুর খালি জমি আছে, কিন্তু **পার্কিং টিকিট** শেষ। টিকিট ছাড়া কেউ গাড়ি ঢোকাতে পারবে না।

### `df -i` দিয়ে দেখা

```console
$ df -i
Filesystem       Inodes   IUsed    IFree IUse% Mounted on
/dev/sda2      15630336 15630336        0  100% /
tmpfs           2041216    1205  2040011    1% /run
/dev/sda1             0       0        0     - /boot/efi
```

| কলাম | মানে |
|---|---|
| `Inodes` | মোট inode |
| `IUsed` | ব্যবহৃত |
| `IFree` | বাকি |
| `IUse%` | ব্যবহারের শতাংশ |

> 📝 **মনে রাখো:** "No space left on device" এলে সবসময় **দুটোই** চেক করো: `df -h` (জায়গা) আর `df -i` (inode)।

`/boot/efi`-তে `0` আর `-` দেখাচ্ছে কারণ FAT32-এ inode ধারণাটাই নেই। আর XFS বা btrfs-এ inode প্রয়োজন অনুযায়ী তৈরি হয় (dynamic), তাই সেখানে এই সমস্যা প্রায় হয় না। RHEL/Fedora সার্ভারে ডিফল্ট ফাইলসিস্টেম XFS, আর Fedora Workstation-এ btrfs। Ubuntu/Debian ডিফল্টে ext4 ব্যবহার করে।

### কোথায় এত ফাইল, খুঁজে বের করো

```console
$ sudo du --inodes -x / 2>/dev/null | sort -n | tail -5
182311  /usr
240155  /var/lib
1210887 /var/spool/postfix/maildrop
1215643 /var/spool
15630011 /
```

`du --inodes` জায়গার বদলে inode (ফাইল সংখ্যা) গোনে, আর `-x` অন্য ফাইলসিস্টেমে ঢোকে না। আরেকটা পুরনো কিন্তু কাজের উপায়:

```console
$ sudo find /var -xdev -type f | cut -d/ -f2,3 | sort | uniq -c | sort -n | tail
```

### সমাধান

1. অপ্রয়োজনীয় ছোট ফাইল মুছে ফেলো (পুরনো সেশন, ক্যাশ, আটকে থাকা মেইল কিউ)। অনেক ফাইল একসাথে মুছতে `find DIR -type f -delete` ব্যবহার করো, কারণ `rm *` দিলে "Argument list too long" আসতে পারে।
2. সমস্যার মূল ঠিক করো: কোন প্রোগ্রাম এত ফাইল বানাচ্ছে? (cron job, PHP session cleanup ইত্যাদি)
3. নতুন ফাইলসিস্টেম বানানোর সময় বেশি inode রাখো:

```console
$ sudo mkfs.ext4 -i 4096 /dev/sdb1       # প্রতি 4 KB-তে একটা inode (বেশি inode)
$ sudo mkfs.ext4 -N 50000000 /dev/sdb1   # সরাসরি মোট inode সংখ্যা বলে দাও
$ sudo mkfs.ext4 -T small /dev/sdb1      # ছোট ফাইলের জন্য প্রিসেট
```

> ⚠️ **সতর্কতা:** `mkfs` চালালে পার্টিশনের **সব ডেটা মুছে যায়**। ext4-এ ইতিমধ্যে তৈরি ফাইলসিস্টেমের inode সংখ্যা পরে বাড়ানো যায় না। ডিস্ক নিয়ে বিস্তারিত [অধ্যায় ৩২](32-disk-partition-mount.md)-এ।

ext4-এ inode-এর তথ্য দেখতে `tune2fs` ব্যবহার করা যায়:

```console
$ sudo tune2fs -l /dev/sda2 | grep -i inode
Inode count:              15630336
Free inodes:              14812001
Inodes per group:         8192
Inode size:               256
...
```

> 💡 **টিপস (অ্যাডভান্সড):** `debugfs` দিয়ে ext4-এর ভেতরে সরাসরি উঁকি দেওয়া যায়, যেমন `sudo debugfs -R 'stat <1835123>' /dev/sda2`। এটা শুধু শেখার জন্য read-only ভাবে ব্যবহার করো। ভুল কমান্ডে ফাইলসিস্টেম নষ্ট হতে পারে।

---

## ১১.১৩ বাস্তব জীবনে লিংকের ব্যবহার

### ১. `/bin` → `/usr/bin` (usrmerge)

আধুনিক Ubuntu, Debian 12, Fedora ও RHEL 9-এ পুরনো ডিরেক্টরিগুলো আসলে symlink:

```console
$ ls -ld /bin /sbin /lib /lib64
lrwxrwxrwx 1 root root 7 Apr 22  2024 /bin -> usr/bin
lrwxrwxrwx 1 root root 8 Apr 22  2024 /sbin -> usr/sbin
lrwxrwxrwx 1 root root 7 Apr 22  2024 /lib -> usr/lib
lrwxrwxrwx 1 root root 9 Apr 22  2024 /lib64 -> usr/lib64
```

এতে পুরনো স্ক্রিপ্টে লেখা `#!/bin/bash` এখনো কাজ করে, অথচ সব binary এক জায়গায় (`/usr/bin`) থাকে। FHS নিয়ে বিস্তারিত আছে [অধ্যায় ৭](07-filesystem-hierarchy.md)-এ।

### ২. `/etc/alternatives`: একই কাজের একাধিক প্রোগ্রাম

সিস্টেমে হয়তো `nano`, `vim` দুটোই ইনস্টল আছে। `editor` কমান্ড চালালে কোনটা খুলবে? Debian/Ubuntu এটা দুই স্তরের symlink দিয়ে সমাধান করে:

```console
$ ls -l /usr/bin/editor
lrwxrwxrwx 1 root root 24 Apr 22  2024 /usr/bin/editor -> /etc/alternatives/editor
$ ls -l /etc/alternatives/editor
lrwxrwxrwx 1 root root 9 Apr 22  2024 /etc/alternatives/editor -> /bin/nano
```

```
/usr/bin/editor ──▶ /etc/alternatives/editor ──▶ /bin/nano
   (স্থির নাম)          (সুইচবোর্ড)                (আসল প্রোগ্রাম)
                                       └── বদলে ──▶ /usr/bin/vim.basic
```

কেন দুই স্তর? `/usr/bin`-এর ফাইল প্যাকেজ ম্যানেজারের এলাকা, আর `/etc` হলো কনফিগারেশনের এলাকা। অ্যাডমিন শুধু `/etc/alternatives`-এর লিংক বদলায়।

এটা হাতে না বদলে `update-alternatives` টুল ব্যবহার করো:

```console
$ update-alternatives --display editor
editor - auto mode
  link best version is /bin/nano
  link currently points to /bin/nano
  link editor is /usr/bin/editor
/bin/nano - priority 40
/usr/bin/vim.basic - priority 30

$ sudo update-alternatives --config editor
There are 2 choices for the alternative editor (providing /usr/bin/editor).

  Selection    Path                Priority   Status
------------------------------------------------------------
* 0            /bin/nano            40        auto mode
  1            /bin/nano            40        manual mode
  2            /usr/bin/vim.basic   30        manual mode

Press <enter> to keep the current choice[*], or type selection number: 2
```

| কমান্ড | কাজ |
|---|---|
| `update-alternatives --display NAME` | বর্তমান অবস্থা ও সব বিকল্প দেখাও |
| `update-alternatives --list NAME` | শুধু বিকল্পগুলোর path দেখাও |
| `sudo update-alternatives --config NAME` | মেনু থেকে বেছে নাও |
| `sudo update-alternatives --set NAME PATH` | মেনু ছাড়াই সরাসরি সেট করো (স্ক্রিপ্টের জন্য) |
| `sudo update-alternatives --auto NAME` | আবার স্বয়ংক্রিয় (সর্বোচ্চ priority) মোডে ফেরো |
| `sudo update-alternatives --install LINK NAME PATH PRIORITY` | নতুন বিকল্প যোগ করো |

> 💡 **টিপস:** Fedora/RHEL-এ একই সিস্টেম আছে, তবে কমান্ডের নাম `alternatives` (যেমন `sudo alternatives --config java`)। Java, Python বা `editor`-এর মতো জায়গায় এটা খুব কাজে লাগে।

### ৩. ভার্সনযুক্ত শেয়ার্ড লাইব্রেরি (`.so` ফাইল)

```console
$ ls -l /usr/lib/x86_64-linux-gnu/libz.so*
lrwxrwxrwx 1 root root     13 ... libz.so -> libz.so.1.3
lrwxrwxrwx 1 root root     13 ... libz.so.1 -> libz.so.1.3
-rw-r--r-- 1 root root 113000 ... libz.so.1.3
```

(সঠিক ভার্সন নম্বর তোমার distro-তে আলাদা হতে পারে। আর `libz.so` লিংকটা শুধু `-dev`/`-devel` প্যাকেজ ইনস্টল থাকলে থাকে।)

তিন স্তরের নামকরণ:

```
libz.so          ──▶ libz.so.1     ──▶ libz.so.1.3
(linker name)        (soname)          (real name: আসল ফাইল)
compile-এর সময়      চলার সময়         আসল কোড
gcc -lz খোঁজে       প্রোগ্রাম খোঁজে
```

- **real name** (`libz.so.1.3`): আসল লাইব্রেরি ফাইল, পুরো ভার্সন সহ
- **soname** (`libz.so.1`): major ভার্সন। একই major ভার্সনের ভেতরে ABI সামঞ্জস্যপূর্ণ থাকে। কম্পাইল করা প্রোগ্রামের ভেতরে এই নামটাই লেখা থাকে।
- **linker name** (`libz.so`): শুধু কম্পাইল করার সময় `gcc -lz` এটা খোঁজে

**কেন এভাবে?** zlib-এ সিকিউরিটি প্যাচ এলে `libz.so.1.3.1` ইনস্টল হয় আর `libz.so.1` লিংকটা নতুন ফাইলে নির্দেশ করে। তোমার কোনো প্রোগ্রাম আবার কম্পাইল করতে হয় না, সব প্রোগ্রাম সাথে সাথে প্যাচড লাইব্রেরি পায়। আর একসাথে `libfoo.so.1` ও `libfoo.so.2` থাকতে পারে, তাই পুরনো ও নতুন দুই ধরনের প্রোগ্রামই চলে।

soname লিংকগুলো `ldconfig` স্বয়ংক্রিয়ভাবে বানায়/আপডেট করে, আর `ldd` দেখায় কোনো প্রোগ্রাম কোন লাইব্রেরি ব্যবহার করে:

```console
$ ldd /usr/bin/gzip
	linux-vdso.so.1 (0x00007ffd...)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f...)
	/lib64/ld-linux-x86-64.so.2 (0x00007f...)
```

সোর্স থেকে বিল্ড করার সময় এগুলো আবার আসবে ([অধ্যায় ৩১](31-compile-from-source.md))।

### ৪. ডিপ্লয়মেন্টে `current` লিংক (zero-downtime switch)

ওয়েব অ্যাপ ডিপ্লয়ের একটা জনপ্রিয় কৌশল:

```
/var/www/app/
├── releases/
│   ├── 2026-11-10/
│   ├── 2026-11-11/
│   └── 2026-11-12/      ← নতুন রিলিজ
└── current -> releases/2026-11-11     (ওয়েব সার্ভার এখানে দেখে)
```

নতুন রিলিজ প্রস্তুত হলে শুধু লিংকটা ঘুরিয়ে দাও। সমস্যা হলে আগেরটায় ফিরিয়ে দাও (rollback), কয়েক মিলিসেকেন্ডে।

```console
$ ln -sfn releases/2026-11-12 /var/www/app/current
```

`-n` এখানে জরুরি। এটা না দিলে `current` যেহেতু একটা ডিরেক্টরির দিকে লিংক, `ln` নতুন লিংকটা **`current`-এর ভেতরে** (মানে পুরনো রিলিজ ফোল্ডারের ভেতরে) বানিয়ে ফেলবে!

> 💡 **টিপস (অ্যাডভান্সড):** `ln -sfn` ভেতরে আগে পুরনো লিংক মোছে, তারপর নতুন বানায়। এই দুই ধাপের মাঝে খুব অল্প সময়ের জন্য `current` থাকে না। পুরোপুরি atomic সুইচ চাইলে এভাবে করো (rename system call atomic):
> ```bash
> ln -s releases/2026-11-12 /var/www/app/current.tmp
> mv -T /var/www/app/current.tmp /var/www/app/current
> ```

### ৫. আরও কিছু জায়গা যেখানে লিংক দেখবে

| জায়গা | কী |
|---|---|
| `/etc/systemd/system/*.wants/` | `systemctl enable` আসলে এখানে symlink বানায় ([অধ্যায় ২৬](26-systemd-services.md)) |
| `/etc/nginx/sites-enabled/` | `sites-available`-এর ফাইলের দিকে symlink (Debian/Ubuntu) ([অধ্যায় ৪০](40-web-server-services.md)) |
| `/etc/localtime` | `/usr/share/zoneinfo/Asia/Dhaka`-এর দিকে symlink |
| `/dev/disk/by-uuid/` | আসল ডিভাইস (`../../sda2`)-এর দিকে symlink |
| `/proc/self` | যে প্রসেস পড়ছে তার নিজের `/proc/PID`-এর দিকে symlink |
| `/etc/resolv.conf` | Ubuntu-তে `../run/systemd/resolve/stub-resolv.conf`-এর দিকে symlink |
| `~/.bashrc` ইত্যাদি dotfiles | অনেকে git রিপো থেকে symlink করে সব মেশিনে একই সেটআপ রাখে ([অধ্যায় ৪৪](44-environment-bashrc.md)) |
| হার্ড লিংক ব্যাকআপ | `rsync --link-dest` আর `cp -al`: অপরিবর্তিত ফাইল আবার কপি না করে হার্ড লিংক করে, তাই প্রতিটা স্ন্যাপশট "পূর্ণ" দেখায় কিন্তু জায়গা প্রায় নেয় না |

---

## ১১.১৪ কীবোর্ড শর্টকাট

এই টপিকের কাজ মূলত কমান্ড দিয়ে, তবে কিছু শর্টকাট সময় বাঁচায়:

| শর্টকাট | কোথায় | কাজ |
|---|---|---|
| `Tab` | টার্মিনাল | ফাইল/লিংকের নাম auto-complete (লিংক ডিরেক্টরি হলে শেষে `/` যোগ হয়) |
| `Alt + .` | Bash | আগের কমান্ডের শেষ argument বসাও (যেমন `ln -s target link` চালিয়ে তারপর `ls -l` + `Alt+.`) |
| `Ctrl + M` | Nautilus (GNOME Files) | নির্বাচিত ফাইলের লিংক বানাও (Make Link) |
| `Ctrl + Shift` + ড্র্যাগ | Nautilus | ছেড়ে দেওয়ার জায়গায় লিংক তৈরি হবে |
| `Ctrl + X`, তারপর `L` | Midnight Commander (`mc`) | হার্ড লিংক বানাও |
| `Ctrl + X`, তারপর `S` | Midnight Commander | সিম্বলিক লিংক বানাও |
| `Ctrl + X`, তারপর `Ctrl + S` | Midnight Commander | বিদ্যমান symlink এডিট করো |

টার্মিনালের অন্য সব শর্টকাট পাবে [অধ্যায় ৬](06-terminal-shortcuts.md) আর [পরিশিষ্ট ক](appendix-a-shortcuts-cheatsheet.md)-এ।

---

## হাতে-কলমে অনুশীলন

একটা নিরাপদ খেলার জায়গা বানিয়ে নাও। সব কাজ এর ভেতরে করো:

```console
$ mkdir -p ~/link-lab && cd ~/link-lab
```

### ল্যাব ১: inode আর নাম আলাদা

```console
$ echo "original" > a.txt
$ ls -i a.txt                       # inode নম্বর লিখে রাখো
$ mv a.txt b.txt
$ ls -i b.txt                       # নম্বর কি বদলাল? (না)
$ cp b.txt c.txt
$ ls -i b.txt c.txt                 # cp নতুন inode বানায়
```

**প্রশ্ন:** `mv` আর `cp`-এর পর inode নম্বরে কী পার্থক্য দেখলে, কেন?

### ল্যাব ২: হার্ড লিংক ও link count

```console
$ ln b.txt hard1
$ ln b.txt hard2
$ stat -c '%n %i %h' b.txt hard1 hard2
$ echo "changed" >> hard2
$ cat b.txt                          # পরিবর্তন দেখা যাচ্ছে?
$ rm b.txt hard1
$ stat -c '%n %i %h' hard2           # link count এখন কত?
$ cat hard2                          # ডেটা আছে তো?
```

### ল্যাব ৩: সিম্বলিক লিংক, relative বনাম absolute

```console
$ mkdir -p data && echo "secret" > data/info.txt
$ ln -s data/info.txt rel-link
$ ln -s "$PWD/data/info.txt" abs-link
$ ls -l rel-link abs-link
$ mkdir moved && mv rel-link abs-link moved/
$ cat moved/rel-link                 # ভাঙল কেন?
$ cat moved/abs-link                 # কাজ করল কেন?
$ ln -srf data/info.txt moved/rel-link
$ cat moved/rel-link                 # এখন ঠিক
$ readlink moved/rel-link            # ../data/info.txt
```

### ল্যাব ৪: ভাঙা লিংক শিকার

```console
$ for i in 1 2 3; do echo $i > f$i; ln -s f$i l$i; done
$ rm f2
$ find . -xtype l                    # ./l2 দেখাবে
$ file l2
$ find . -xtype l -delete
$ ls -l
```

### ল্যাব ৫: মুছে ফেলা কিন্তু খোলা ফাইল

```console
$ echo "important data" > keep.txt
$ sleep 600 < keep.txt &             # একটা প্রসেস ফাইলটা খোলা রাখল
$ PID=$!
$ rm keep.txt
$ ls -l /proc/$PID/fd/0              # (deleted) দেখো
$ cat /proc/$PID/fd/0                # ডেটা এখনো পড়া যাচ্ছে!
$ cp /proc/$PID/fd/0 recovered.txt
$ kill $PID
$ cat recovered.txt
```

### ল্যাব ৬: inode শেষ করার পরীক্ষা (নিরাপদভাবে, RAM-এ)

`tmpfs`-এ একটা ছোট ফাইলসিস্টেম বানিয়ে ইচ্ছে করে inode শেষ করো। এতে তোমার আসল ডিস্কের কোনো ক্ষতি হবে না:

```console
$ mkdir -p ~/tinyfs
$ sudo mount -t tmpfs -o size=10M,nr_inodes=100 tmpfs ~/tinyfs
$ sudo chown "$USER" ~/tinyfs
$ df -i ~/tinyfs
$ for i in $(seq 1 200); do touch ~/tinyfs/f$i || break; done
touch: cannot touch '/home/rahim/tinyfs/f100': No space left on device
$ df -h ~/tinyfs                     # জায়গা প্রায় পুরোটা খালি!
$ df -i ~/tinyfs                     # কিন্তু IUse% 100%
$ sudo umount ~/tinyfs
```

(যে ফাইলে গিয়ে থামবে তার নম্বর একটু আলাদা হতে পারে, কারণ root ডিরেক্টরি নিজেও একটা inode নেয়।)

### ল্যাব ৭: alternatives অনুসন্ধান (Debian/Ubuntu)

```console
$ ls -l /etc/alternatives | head
$ readlink -f /usr/bin/editor
$ update-alternatives --display editor
$ ls -l /usr/lib/x86_64-linux-gnu/libc.so*
```

### ল্যাব ৮: atomic ডিপ্লয় সিমুলেশন

```console
$ mkdir -p app/releases/v1 app/releases/v2
$ echo "v1" > app/releases/v1/index.html
$ echo "v2" > app/releases/v2/index.html
$ ln -s releases/v1 app/current
$ cat app/current/index.html                       # v1
$ ln -s releases/v2 app/current.tmp && mv -T app/current.tmp app/current
$ cat app/current/index.html                       # v2
$ ln -sfn releases/v1 app/current                  # rollback
$ cat app/current/index.html                       # v1
```

শেষে পরিষ্কার করো:

```console
$ cd ~ && rm -rf ~/link-lab ~/tinyfs
```

---

## সারসংক্ষেপ

| কমান্ড / শর্টকাট | কাজ |
|---|---|
| `ls -i` / `ls -li` | inode নম্বর দেখাও |
| `stat FILE` | inode-এর পুরো তথ্য |
| `stat -c '%i %h %n' FILE` | নির্দিষ্ট তথ্য (inode, link count, নাম) |
| `stat -L LINK` | লিংকের টার্গেটের তথ্য |
| `stat -f /` | ফাইলসিস্টেমের তথ্য (inode সহ) |
| `ln TARGET NAME` | হার্ড লিংক বানাও |
| `ln -s TARGET NAME` | সিম্বলিক লিংক বানাও |
| `ln -sr TARGET NAME` | relative সিম্বলিক লিংক |
| `ln -sfn TARGET NAME` | বিদ্যমান লিংক (ডিরেক্টরির দিকেও) বদলাও |
| `mv -T tmp link` | atomic ভাবে লিংক প্রতিস্থাপন |
| `readlink LINK` | লিংকের ভেতরের লেখা (এক ধাপ) |
| `readlink -f` / `realpath` | চূড়ান্ত absolute path |
| `namei -l PATH` | path-এর প্রতিটা ধাপ ও পারমিশন |
| `find . -samefile F` / `-inum N` | একই inode-এর সব নাম |
| `find . -xtype l` | ভাঙা লিংক খোঁজো |
| `file LINK` | লিংক ভাঙা কিনা বলে |
| `rm LINK` / `unlink LINK` | লিংক মোছো (টার্গেট নয়, শেষে `/` দিও না) |
| `lsof +L1` | মুছে ফেলা কিন্তু খোলা ফাইল |
| `: > FILE` / `truncate -s 0 FILE` | inode না বদলে ফাইল খালি করো |
| `df -i` | inode ব্যবহার দেখো |
| `du --inodes -x DIR` | কোথায় বেশি ফাইল |
| `update-alternatives --config NAME` | বিকল্প প্রোগ্রাম বাছাই (Fedora/RHEL: `alternatives`) |
| `ldd PROGRAM` | প্রোগ্রাম কোন লাইব্রেরি ব্যবহার করে |
| `Ctrl+M` (Nautilus) | GUI-তে লিংক বানাও |
| `Ctrl+X L` / `Ctrl+X S` (mc) | হার্ড / সিম্বলিক লিংক |

> ⚠️ **সতর্কতা:** ডিরেক্টরির দিকে নির্দেশ করা symlink মোছার সময় কখনো শেষে `/` দিও না। `rm mylink` ঠিক আছে। কিন্তু `rm -rf mylink/` লিংকের বদলে **টার্গেট ডিরেক্টরির ভেতরের সবকিছু** মুছে দিতে পারে!

---

## নিজেকে যাচাই করো

1. inode-এ কোন কোন তথ্য থাকে? কোন গুরুত্বপূর্ণ তথ্যটা থাকে **না**, আর সেটা কোথায় থাকে?
2. একই ফাইলসিস্টেমে 50 GB ফাইল `mv` করতে এক মুহূর্ত লাগে, কিন্তু অন্য ডিস্কে সরাতে অনেক সময় লাগে। কেন?
3. একটা নতুন খালি ডিরেক্টরির link count 2 কেন? তার ভেতরে 3টা সাব-ডিরেক্টরি বানালে কত হবে (ext4-এ)?
4. হার্ড লিংক দিয়ে কোন দুটো কাজ করা যায় না, আর কেন?
5. `ln -s ../config/app.conf settings` চালানো হলো `/home/rahim/project/bin/` ডিরেক্টরিতে দাঁড়িয়ে, আর লিংকটাও সেখানেই তৈরি হলো। লিংকটা কোন absolute path-এ নির্দেশ করবে?
6. `readlink link` আর `readlink -f link`-এর পার্থক্য কী?
7. একটা সার্ভারে 30 GB লগ ফাইল `rm` করার পরও `df -h` জায়গা ফেরত দেখাচ্ছে না। কারণ কী, আর কীভাবে খুঁজে বের করবে ও ঠিক করবে?
8. `touch` বলছে "No space left on device", অথচ `df -h` দেখাচ্ছে 60% খালি। প্রথমে কোন কমান্ড চালাবে, আর সমস্যাটা কী হতে পারে?
9. `libssl.so.3` কী ধরনের নাম, আর লাইব্রেরি এভাবে symlink দিয়ে সাজানোর সুবিধা কী?
10. symlink-এর পারমিশন সবসময় `lrwxrwxrwx` দেখায়। এর মানে কি যে কেউ লিংক দিয়ে টার্গেট ফাইল এডিট করতে পারবে?

<details>
<summary>উত্তর দেখো</summary>

1. inode-এ থাকে ফাইলের ধরন, পারমিশন, মালিক (UID), গ্রুপ (GID), আকার, link count, টাইমস্ট্যাম্প (atime, mtime, ctime, birth time) এবং ডেটা ব্লকের ঠিকানা। **ফাইলের নাম থাকে না।** নাম থাকে প্যারেন্ট ডিরেক্টরির টেবিলে (name → inode number)।

2. একই ফাইলসিস্টেমে `mv` শুধু ডিরেক্টরি এন্ট্রি বদলায় (rename)। inode আর ডেটা নড়ে না। অন্য ফাইলসিস্টেমে inode নম্বরের কোনো অর্থ নেই, তাই সেখানে পুরো ডেটা কপি করে নতুন inode বানাতে হয়, তারপর পুরনোটা মুছতে হয়।

3. কারণ দুটো নাম এটাকে দেখায়: প্যারেন্টে থাকা এর নাম, আর এর নিজের ভেতরের `.`। প্রতিটা সাব-ডিরেক্টরির `..` আরেকটা করে যোগ করে, তাই 3টা সাব-ডিরেক্টরি থাকলে 2 + 3 = **5**।

4. (ক) ডিরেক্টরির হার্ড লিংক বানানো যায় না, কারণ এতে ট্রিতে লুপ তৈরি হতে পারে আর `..`-এর মানে অস্পষ্ট হয়ে যায়। (খ) ভিন্ন ফাইলসিস্টেমে হার্ড লিংক বানানো যায় না, কারণ inode নম্বর শুধু এক ফাইলসিস্টেমের ভেতরে ইউনিক ("Invalid cross-device link")।

5. relative টার্গেট হিসাব হয় লিংকের ডিরেক্টরি (`/home/rahim/project/bin/`) থেকে, তাই নির্দেশ করবে `/home/rahim/project/config/app.conf`-এ।

6. `readlink link` লিংকের ভেতরে যা লেখা আছে শুধু সেটাই দেখায় (এক ধাপ, relative হলে relative-ই)। `readlink -f link` সব লিংকের চেইন অনুসরণ করে চূড়ান্ত **absolute canonical path** দেখায় (`realpath`-এর মতো)।

7. কোনো প্রসেস (লগ লেখা সার্ভিস) এখনো ফাইলটা খোলা রেখেছে। link count 0 হলেও open file descriptor থাকায় ডেটা মুক্ত হয়নি। খুঁজতে `sudo lsof +L1` চালাও বা `/proc/*/fd`-এ `(deleted)` খোঁজো। ঠিক করতে সার্ভিস restart করো, অথবা `: > /proc/PID/fd/N` দিয়ে truncate করো। ভবিষ্যতে `rm`-এর বদলে `truncate -s 0` ব্যবহার করো।

8. `df -i`। সম্ভবত inode শেষ হয়ে গেছে (IUse% 100%), কারণ লক্ষ লক্ষ ছোট ফাইল আছে। `du --inodes -x /` দিয়ে কোথায় সেগুলো খুঁজে বের করো আর অপ্রয়োজনীয়গুলো মুছে ফেলো।

9. এটা **soname**: লাইব্রেরির নাম আর তার major ভার্সন (3)। কম্পাইল করা প্রোগ্রামের ভেতরে এই নামটাই লেখা থাকে। বেশিরভাগ লাইব্রেরিতে soname একটা symlink, যেটা পূর্ণ ভার্সনযুক্ত আসল ফাইলের দিকে নির্দেশ করে, আর `ldconfig` এই লিংক সাজায়। (কিছু লাইব্রেরিতে, যেমন অনেক distro-র OpenSSL 3-এ, আসল ফাইলের নামই soname, তখন আলাদা লিংক থাকে না।) সুবিধা: প্যাচ বা minor আপডেটে শুধু লিংক বদলালেই সব প্রোগ্রাম নতুন লাইব্রেরি পায়, আবার কম্পাইল করতে হয় না। আর ভিন্ন major ভার্সন পাশাপাশি থাকতে পারে।

10. না। লিনাক্স symlink-এর নিজের পারমিশন উপেক্ষা করে। লিংক দিয়ে ফাইল খুললে **টার্গেটের** পারমিশন চেক হয়। টার্গেট `rw-r--r--` আর মালিক root হলে সাধারণ ইউজার লিংক দিয়েও শুধু পড়তে পারবে, লিখতে পারবে না।

</details>

---

[← আগের অধ্যায়](10-file-dekha.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](12-file-khoja.md)

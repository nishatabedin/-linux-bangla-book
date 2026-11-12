# অধ্যায় ৩৪: আর্কাইভ ও কম্প্রেশন

## এই অধ্যায়ে যা শিখবে

- **Archive** (অনেক ফাইলকে এক ফাইলে বাঁধা) আর **Compression** (ফাইলের আকার ছোট করা) আসলে দুটো আলাদা কাজ, আর লিনাক্সে কেন এরা আলাদা টুল দিয়ে হয়
- কম্প্রেশন ভেতরে কীভাবে কাজ করে: repetition খোঁজা, dictionary, entropy, lossless বনাম lossy
- `tar` কমান্ড পুরোপুরি: `-c -x -t -v -f -z -j -J --zstd -C --exclude -a`, আর অপশনগুলো মনে রাখার কৌশল
- `gzip`, `gunzip`, `zcat`, `zless`, `zgrep`: সবচেয়ে প্রচলিত কম্প্রেসর
- `bzip2`, `xz`, `zstd`: কোনটা কখন, কোনটা দ্রুত, কোনটা বেশি ছোট করে
- গতি বনাম কম্প্রেশন অনুপাতের তুলনা টেবিল
- `zip`/`unzip` (Windows-এর সাথে আদান-প্রদানের জন্য), `7z`, `rar`/`unrar`
- `.tar.gz`, `.tgz`, `.tar.xz`, `.tar.zst`, `.deb`, `.jar` সহ এক্সটেনশন দেখে চেনা, আর `file` কমান্ড দিয়ে আসল ফরম্যাট বের করা
- পাইপ দিয়ে `tar`-কে `ssh`-এর ভেতর দিয়ে পাঠানো: ডিস্কে কোনো মধ্যবর্তী ফাইল না বানিয়ে এক সার্ভার থেকে আরেক সার্ভারে ফোল্ডার কপি
- multi-core কম্প্রেশন (`pigz`, `xz -T0`, `zstd -T0`), বড় আর্কাইভ ভাগ করা (`split`), আর tarbomb ও path traversal-এর মতো ঝুঁকি

---

## ৩৪.১ শুরুর কথা: আর্কাইভ আর কম্প্রেশন কেন দরকার?

ধরো তুমি বাসা বদলাচ্ছ। তোমার কাছে একশোটা বই, পঞ্চাশটা জামা, কিছু থালা-বাসন। এগুলো একটা একটা করে ট্রাকে তুললে বিশৃঙ্খল হবে, কিছু হারাবেও। তাই তুমি দুটো কাজ করবে:

১. **সব জিনিস কয়েকটা বাক্সে ভরবে।** জিনিসের সংখ্যা কমল না, শুধু সব এক জায়গায় গোছানো হলো। এটাই **archive** (আর্কাইভ)।

২. **জামাগুলো vacuum bag-এ ভরে বাতাস বের করে দেবে।** জামা একই থাকল, কিন্তু জায়গা কম লাগল। এটাই **compression** (কম্প্রেশন)।

কম্পিউটারেও ঠিক তাই:

| কাজ | উদ্দেশ্য | লিনাক্সের টুল |
|---|---|---|
| Archive | অনেক ফাইল + ফোল্ডার + তাদের permission, owner, সময়, symlink সব একটি ফাইলে রাখা | `tar`, `cpio`, `ar` |
| Compression | একটি ডেটা-স্রোতের (stream) আকার ছোট করা | `gzip`, `bzip2`, `xz`, `zstd`, `lz4` |
| দুটোই একসাথে | Windows-ঘরানার ফরম্যাট | `zip`, `7z`, `rar` |

> 📝 **মনে রাখো:** `gzip`, `bzip2`, `xz`, `zstd` এরা **একটা ফাইলকে** কম্প্রেস করে। একটা ফোল্ডার এদের দিয়ে সরাসরি কম্প্রেস করা যায় না। ফোল্ডার কম্প্রেস করতে হলে আগে `tar` দিয়ে এক ফাইলে বাঁধো, তারপর কম্প্রেস করো। এ কারণেই লিনাক্সে `.tar.gz` এর মতো "দুই-স্তরের" এক্সটেনশন দেখা যায়।

### কেন লিনাক্সে দুটো কাজ আলাদা?

এটা Unix দর্শনের একটা চমৎকার উদাহরণ: **"একটা প্রোগ্রাম একটা কাজই করবে, এবং ভালোভাবে করবে।"** ([অধ্যায় ১](01-linux-ki-o-keno.md)-এ এই দর্শনের কথা বলা আছে।)

- `tar` জানে ফাইল সিস্টেমের খুঁটিনাটি: permission, owner, symlink, hard link, timestamp। কিন্তু কম্প্রেশনের কিছুই জানে না।
- `gzip` শুধু bytes-এর একটা স্রোত নেয় আর ছোট করে দেয়। সেটা ফাইল নাকি ফোল্ডার, তা জানে না, জানার দরকারও নেই।

এই আলাদা থাকার সুবিধা হলো যখন নতুন ও ভালো কম্প্রেসর (যেমন `zstd`) এল, তখন `tar` বদলাতে হলো না। শুধু `tar ... | zstd` লিখলেই হলো। পাইপ ([অধ্যায় ১৯](19-redirection-pipe.md)) দিয়ে দুটো টুল জোড়া লাগানো যায়।

```
  ফোল্ডার project/                 এক ফাইল                ছোট এক ফাইল
 ┌──────────────────┐           ┌────────────┐          ┌──────────┐
 │ a.txt            │           │            │          │          │
 │ b.jpg            │  ──tar──▶ │ project.tar│ ──gzip─▶ │project   │
 │ src/main.c       │ (archive) │  (১০০ MB)  │ (compress│.tar.gz   │
 │ src/util.h       │           │            │  )       │ (৩০ MB)  │
 └──────────────────┘           └────────────┘          └──────────┘
   অনেক ফাইল, metadata সহ        আকার প্রায় একই         আকার ছোট
```

> 💡 **টিপস:** Windows-এর `.zip` ফাইল দুটো কাজই একসাথে করে, আর প্রতিটা ফাইল **আলাদা আলাদা** কম্প্রেস করে। `.tar.gz` পুরো আর্কাইভকে **একসাথে** কম্প্রেস করে। এর ফলাফল নিয়ে একটু পরে কথা বলব ("solid" কম্প্রেশন, ৩৪.১০)।

---

## ৩৪.২ কম্প্রেশন ভেতরে কীভাবে কাজ করে (Under the Hood)

কম্প্রেশন কোনো জাদু নয়। এর মূল ধারণা হলো **ডেটার ভেতরের পুনরাবৃত্তি (redundancy) খুঁজে বের করে সংক্ষেপে লেখা।**

### Lossless বনাম Lossy

| ধরন | মানে | উদাহরণ |
|---|---|---|
| **Lossless** (ক্ষতিহীন) | ডিকম্প্রেস করলে হুবহু আগের ডেটা ফেরত আসে, একটা bit-ও বদলায় না | gzip, bzip2, xz, zstd, zip, 7z, PNG, FLAC |
| **Lossy** (ক্ষতিকর) | কিছু তথ্য ইচ্ছা করে ফেলে দেওয়া হয় যা মানুষ সহজে টের পায় না | JPEG, MP3, MP4/H.264 |

এই অধ্যায়ের সব টুল **lossless**। প্রোগ্রাম, কনফিগ ফাইল বা ডাটাবেস ব্যাকআপে একটা bit বদলে গেলেও সর্বনাশ, তাই এখানে lossless ছাড়া উপায় নেই।

### ধারণা ১: Run-Length (একই জিনিস বারবার)

```
আসল:      AAAAAAAAAABBBBBCC        (১৭ অক্ষর)
সংক্ষেপে:  10A5B2C                  (৭ অক্ষর)
```

### ধারণা ২: LZ77 (আগে যা দেখেছি তার রেফারেন্স)

এটাই `gzip`, `xz`, `zstd` সবার হৃদয়। কম্প্রেসর একটা "জানালা" (sliding window) ধরে রাখে, অর্থাৎ সাম্প্রতিক কিছু ডেটা মনে রাখে। নতুন কোনো অংশ আগে দেখা অংশের সাথে মিলে গেলে পুরোটা আবার না লিখে লেখে: **"পেছনে X ঘর গিয়ে Y অক্ষর কপি করো"।**

```
লেখা:  "ami bhat khai, tumi bhat khao"

            ┌─── এখানে "bhat kha" আগেও এসেছে (১৬ অক্ষর পেছনে)
            ▼
"ami bhat khai, tumi " + <পেছনে ১৬, ৮ অক্ষর কপি> + "o"
```

জানালা যত বড়, তত দূরের পুনরাবৃত্তি ধরা যায়, আর তত ভালো কম্প্রেশন হয়, কিন্তু মেমোরি বেশি লাগে।

| টুল | মূল অ্যালগরিদম | জানালা/ব্লক |
|---|---|---|
| gzip | DEFLATE = LZ77 + Huffman | ৩২ KB জানালা |
| bzip2 | Burrows-Wheeler Transform + Huffman | ১০০–৯০০ KB ব্লক |
| xz | LZMA2 | ডিফল্ট লেভেলে ৮ MB, সর্বোচ্চ লেভেলে ৬৪ MB dictionary |
| zstd | LZ77 + Finite State Entropy + Huffman | লেভেল অনুযায়ী বদলায়, `--long` দিলে অনেক বড় |

### ধারণা ৩: Entropy coding (ঘন ঘন আসা জিনিসের ছোট কোড)

Morse code-এর কথা ভাবো: ইংরেজিতে সবচেয়ে বেশি আসে `E`, তাই তার কোড সবচেয়ে ছোট (`.`)। কম আসা `Q`-এর কোড লম্বা (`--.-`)। **Huffman coding** ঠিক এভাবেই প্রতিটা byte-এর জন্য তার ব্যবহারের হার অনুযায়ী কোড বানায়।

### কেন কিছু ফাইল কম্প্রেস হয় না?

`.jpg`, `.mp4`, `.mp3`, `.zip`, `.gz` এগুলো **আগে থেকেই কম্প্রেস করা**। এদের ভেতরে পুনরাবৃত্তি প্রায় নেই। এদের আবার কম্প্রেস করলে আকার প্রায় একই থাকে, কখনো একটু **বেড়েও** যায় (header-এর কারণে)। এলোমেলো (random) ডেটাও কম্প্রেস হয় না।

```
$ head -c 10M /dev/urandom > random.bin
$ gzip -k random.bin
$ ls -l random.bin random.bin.gz
-rw-r--r-- 1 rahim rahim 10485760 Nov 12 10:00 random.bin
-rw-r--r-- 1 rahim rahim 10488981 Nov 12 10:00 random.bin.gz
```

দেখো, `.gz` ফাইলটা আসলটার চেয়ে **বড়**!

> 📝 **মনে রাখো:** টেক্সট, লগ, সোর্স কোড, SQL dump, CSV দারুণ কম্প্রেস হয় (প্রায়ই ৫–১০ গুণ ছোট)। ছবি, ভিডিও, গান প্রায় হয়ই না। তাই ১০ GB ভিডিও ফোল্ডার `xz -9` দিয়ে ঘণ্টাখানেক কম্প্রেস করে লাভ নেই, শুধু `tar` করো।

### Compression level

প্রায় সব টুলে `-1` থেকে `-9` লেভেল আছে:

- `-1`: দ্রুততম, কম ছোট করে (কম খোঁজে)
- `-9`: ধীরতম, সবচেয়ে ছোট করে (বেশি খোঁজে, বেশি মেমোরি নেয়)

একটা গুরুত্বপূর্ণ ব্যাপার: **ডিকম্প্রেশনের গতি প্রায় লেভেলের উপর নির্ভর করে না।** `-9` দিয়ে বানানো ফাইল খুলতেও প্রায় `-1`-এর মতোই সময় লাগে। এ কারণে যে ফাইল একবার বানিয়ে হাজার জনে ডাউনলোড করবে (যেমন সফটওয়্যার রিলিজ), সেটা উঁচু লেভেলে বানানোই লাভজনক।

---

## ৩৪.৩ `tar`: Tape Archiver

### নামের ইতিহাস ও tar ফাইলের গঠন

`tar` মানে **T**ape **AR**chive। ১৯৭৯ সালের Unix-এ এটা বানানো হয়েছিল ম্যাগনেটিক টেপে ব্যাকআপ রাখার জন্য। টেপে random access নেই, শুধু শুরু থেকে শেষ পর্যন্ত পড়া যায়। তাই tar ফরম্যাটও একদম **সরলরৈখিক (sequential)**: ফাইলের পর ফাইল সাজানো।

```
 project.tar এর ভেতরের গঠন (প্রতিটা ঘর ৫১২-byte ব্লকের গুণিতক)

 ┌──────────┬──────────────────┬──────────┬───────────┬──────────┬─────────┐
 │ Header   │ a.txt-এর ডেটা     │ Header   │ src/main.c│ Header   │ দুটো    │
 │ (a.txt)  │ (৫১২-এর গুণিতকে   │(src/     │-এর ডেটা   │ ...      │ খালি    │
 │ ৫১২ byte │  padding করা)     │ main.c)  │           │          │ ব্লক    │
 └──────────┴──────────────────┴──────────┴───────────┴──────────┴─────────┘
     │                                                              (শেষ চিহ্ন)
     ▼
  Header-এ থাকে: ফাইলের নাম, mode (permission), uid/gid, owner/group নাম,
  আকার, mtime (শেষ পরিবর্তনের সময়), ধরন (সাধারণ ফাইল/ডিরেক্টরি/symlink/
  hard link/device), symlink-এর লক্ষ্য, checksum
```

এই গঠন থেকে কয়েকটা জরুরি সিদ্ধান্ত বের হয়:

1. **কোনো কেন্দ্রীয় সূচিপত্র (index) নেই।** তাই `.tar.gz`-এর ভেতরে কী আছে দেখতে (`tar -t`) পুরো ফাইলটা শুরু থেকে শেষ পর্যন্ত পড়তে ও ডিকম্প্রেস করতে হয়। ১০ GB আর্কাইভ থেকে একটা ছোট ফাইল বের করতেও পুরোটা পড়তে হতে পারে। `.zip`-এর শেষে একটা সূচিপত্র (central directory) থাকে, তাই `unzip -l` তাৎক্ষণিক।
2. **শেষে নতুন ফাইল যোগ করা সহজ** (`tar -r`), কিন্তু শুধু কম্প্রেস-না-করা `.tar`-এ।
3. **স্রোত হিসেবে কাজ করে।** আর্কাইভ বানাতে বানাতেই পাইপ দিয়ে অন্য দিকে পাঠানো যায়। এ কারণেই `tar | ssh` এত শক্তিশালী (৩৪.১২)।

আধুনিক লিনাক্সে (Ubuntu, Debian, Fedora, RHEL) `tar` মানে **GNU tar**। এটা POSIX `ustar` ও `pax` ফরম্যাট বোঝে, আর লম্বা ফাইলনাম, বড় ফাইল (৮ GB-এর বেশি) সবই সামলায়।

```
$ tar --version | head -1
tar (GNU tar) 1.35
```

(সংস্করণ নম্বর distro অনুযায়ী আলাদা হবে।)

### tar-এর মূল syntax

```
tar [মোড] [অপশন...] -f আর্কাইভ_ফাইল [ফাইল/ফোল্ডার...]
```

**মোড (প্রতিবার ঠিক একটা লাগবে):**

| অপশন | লম্বা রূপ | কাজ |
|---|---|---|
| `-c` | `--create` | নতুন আর্কাইভ বানাও |
| `-x` | `--extract` (`--get`) | আর্কাইভ থেকে ফাইল বের করো |
| `-t` | `--list` | ভেতরে কী আছে তার তালিকা দেখাও |
| `-r` | `--append` | শেষে ফাইল যোগ করো (শুধু কম্প্রেস-না-করা `.tar`) |
| `-u` | `--update` | শুধু নতুন/বদলানো ফাইল শেষে যোগ করো |
| `-d` | `--diff` / `--compare` | আর্কাইভ আর ডিস্কের ফাইলের পার্থক্য দেখাও |
| | `--delete` | আর্কাইভ থেকে ফাইল মুছে দাও (শুধু কম্প্রেস-না-করা `.tar`) |
| `-A` | `--concatenate` | একটা tar-এর শেষে আরেকটা tar জোড়া লাগাও |

**সাধারণ অপশন:**

| অপশন | লম্বা রূপ | কাজ |
|---|---|---|
| `-f FILE` | `--file=FILE` | কোন আর্কাইভ ফাইল নিয়ে কাজ। `-` দিলে stdin/stdout |
| `-v` | `--verbose` | প্রতিটা ফাইলের নাম দেখাও (`-vv` দিলে `ls -l` ধাঁচের বিস্তারিত) |
| `-z` | `--gzip` | gzip দিয়ে কম্প্রেস/ডিকম্প্রেস (`.tar.gz`) |
| `-j` | `--bzip2` | bzip2 দিয়ে (`.tar.bz2`) |
| `-J` | `--xz` | xz দিয়ে (`.tar.xz`) |
| | `--zstd` | zstd দিয়ে (`.tar.zst`), GNU tar 1.31+ |
| `-a` | `--auto-compress` | আর্কাইভের এক্সটেনশন দেখে কম্প্রেসর ঠিক করো |
| `-I CMD` | `--use-compress-program=CMD` | নিজের পছন্দের কম্প্রেসর (যেমন `pigz`, `'zstd -19'`) |
| `-C DIR` | `--directory=DIR` | কাজ শুরুর আগে DIR-এ চলে যাও |
| | `--exclude=PATTERN` | মিলে যাওয়া ফাইল বাদ দাও |
| `-X FILE` | `--exclude-from=FILE` | FILE-এ লেখা প্যাটার্নগুলো বাদ দাও |
| | `--exclude-vcs` | `.git`, `.svn` ইত্যাদি বাদ দাও |
| `-T FILE` | `--files-from=FILE` | কোন ফাইলগুলো নেবে তা FILE থেকে পড়ো |
| `-p` | `--preserve-permissions` | extract-এর সময় permission হুবহু রাখো (root হলে ডিফল্ট) |
| | `--same-owner` | extract-এর সময় owner হুবহু রাখো (root হলে ডিফল্ট) |
| | `--numeric-owner` | নামের বদলে uid/gid সংখ্যা ব্যবহার করো |
| | `--strip-components=N` | extract-এর সময় path-এর প্রথম N অংশ বাদ দাও |
| `-k` | `--keep-old-files` | আগে থেকে থাকা ফাইল ওভাররাইট কোরো না (থাকলে error) |
| | `--skip-old-files` | আগে থেকে থাকা ফাইল চুপচাপ এড়িয়ে যাও |
| `-h` | `--dereference` | symlink-এর বদলে আসল ফাইলটা রাখো |
| `-P` | `--absolute-names` | শুরুর `/` মুছবে না (বিপজ্জনক, ৩৪.১৪ দেখো) |
| `-W` | `--verify` | লেখার পর যাচাই (কম্প্রেস-না-করা আর্কাইভে) |
| | `--wildcards` | extract/list-এ `*` ধরনের প্যাটার্ন বোঝো |
| `-g FILE` | `--listed-incremental=FILE` | incremental ব্যাকআপ ([অধ্যায় ৩৫](35-backup-rsync.md)) |
| | `--xattrs`, `--acls`, `--selinux` | extended attribute, ACL, SELinux লেবেল রাখো |
| | `--checkpoint=N` | প্রতি N ব্লক পর পর অগ্রগতি জানাও |

### মনে রাখার কৌশল 🧠

`tar`-এর অপশন নতুনদের কাছে মন্ত্রের মতো লাগে। কিন্তু একটু ভাঙলেই সহজ:

```
   tar  -c   z   v   f   backup.tar.gz   ফোল্ডার/
         │   │   │   │
         │   │   │   └── f = file   "এই নামের ফাইলে" (সবসময় শেষে, তার ঠিক পরেই নাম)
         │   │   └────── v = verbose "বলতে বলতে করো"
         │   └────────── z = gzip    "চেপে ছোট করো"
         └────────────── c = create  "বানাও"
```

**তিনটা মোড, তিনটা প্রশ্ন:**

| অক্ষর | ইংরেজি | বাংলায় মনে রাখো |
|---|---|---|
| `c` | **C**reate | **"চাই বানাতে"** |
| `x` | e**X**tract | **"খুলে বের করো"** (x দেখতে খোলা কাঁচির মতো ✂) |
| `t` | lis**T** / **T**able | **"তালিকা দেখাও"** |

**কম্প্রেসরের অক্ষর:**

| অক্ষর | কম্প্রেসর | মনে রাখার উপায় |
|---|---|---|
| `z` | g**z**ip | gzip-এর মধ্যে z আছে |
| `j` | bzip2 | "b" অক্ষর আগে থেকে ব্যস্ত (`-b` = blocking factor), তাই পাশের j |
| `J` | xz | বড় হাতের J: "বড়" কম্প্রেশন, xz সবচেয়ে বেশি ছোট করে |
| `--zstd` | zstd | এর জন্য কোনো এক-অক্ষরের অপশন নেই, পুরো নাম লেখো |

**জনপ্রিয় ইংরেজি কৌশল:**

- `tar -czvf` → "**C**reate **Z**e **V**ile **F**ile"
- `tar -xzvf` → "e**X**tract **Z**e **V**ile **F**ile"
- `tar -tvf` → "**T**ell **V**erbose **F**ile" (কী আছে বলো)

> ⚠️ **সতর্কতা:** `-f` অপশনের **ঠিক পরেই** আর্কাইভের নাম দিতে হয়। `tar -cfv backup.tar dir/` লিখলে tar ভাববে আর্কাইভের নাম `v`! তখন `v` নামে একটা ফাইল তৈরি হবে আর `backup.tar` আর্কাইভের **ভেতরে** ঢুকে যাবে (যদি সেটা থাকে)। তাই নিয়ম: **`f` সবসময় অক্ষরগুচ্ছের শেষে।**

> 💡 **টিপস:** `tar`-এ ড্যাশ ছাড়াও লেখা যায়: `tar czvf x.tar.gz dir/`। এটা পুরোনো (BSD) স্টাইল, এখনো সব জায়গায় চলে। এই বইয়ে আমরা ড্যাশসহ লিখব, কারণ সেটা অন্য কমান্ডের সাথে সামঞ্জস্যপূর্ণ।

### ৩৪.৩.১ আর্কাইভ বানানো (`-c`)

প্রথমে অনুশীলনের জন্য একটা ফোল্ডার বানাই:

```
$ mkdir -p ~/lab34/project/{src,docs,logs}
$ cd ~/lab34
$ echo 'int main(){return 0;}' > project/src/main.c
$ echo '# My project' > project/docs/README.md
$ seq 1 100000 > project/logs/app.log
$ tree project
project
├── docs
│   └── README.md
├── logs
│   └── app.log
└── src
    └── main.c

4 directories, 3 files
```

কম্প্রেশন ছাড়া শুধু আর্কাইভ:

```
$ tar -cvf project.tar project/
project/
project/src/
project/src/main.c
project/docs/
project/docs/README.md
project/logs/
project/logs/app.log
```

gzip দিয়ে কম্প্রেস করে:

```
$ tar -czvf project.tar.gz project/
```

bzip2, xz, zstd দিয়ে:

```
$ tar -cjf project.tar.bz2 project/
$ tar -cJf project.tar.xz  project/
$ tar --zstd -cf project.tar.zst project/
```

আকার তুলনা করি:

```
$ ls -lh project.tar*
-rw-r--r-- 1 rahim rahim 600K Nov 12 10:05 project.tar
-rw-r--r-- 1 rahim rahim  48K Nov 12 10:06 project.tar.bz2
-rw-r--r-- 1 rahim rahim 122K Nov 12 10:05 project.tar.gz
-rw-r--r-- 1 rahim rahim  40K Nov 12 10:06 project.tar.xz
-rw-r--r-- 1 rahim rahim  75K Nov 12 10:06 project.tar.zst
```

(সংখ্যাগুলো আনুমানিক, তোমার মেশিনে একটু আলাদা হবে। `seq`-এর আউটপুট খুবই পুনরাবৃত্তিময়, তাই অনুপাত অনেক ভালো দেখাচ্ছে।)

**`-a` দিয়ে এক্সটেনশন দেখে স্বয়ংক্রিয় বাছাই:**

```
$ tar -caf project.tar.xz project/      # .xz দেখে xz ব্যবহার করবে
$ tar -caf project.tar.zst project/     # .zst দেখে zstd
```

**নিজের পছন্দের কম্প্রেসর ও লেভেল (`-I`):**

```
$ tar -I 'zstd -19 -T0' -cf project.tar.zst project/
$ tar -I pigz -cf project.tar.gz project/          # multi-core gzip
$ XZ_OPT='-9 -T0' tar -cJf project.tar.xz project/ # পরিবেশ ভেরিয়েবল দিয়ে xz-কে অপশন
```

> 📝 **মনে রাখো:** তুমি যে path দিয়ে আর্কাইভ বানাবে, ভেতরে সেই path-ই থাকবে। `tar -czf a.tar.gz /home/rahim/lab34/project` লিখলে ভেতরে থাকবে `home/rahim/lab34/project/...` (শুরুর `/` বাদ দিয়ে)। extract করলে এই পুরো ফোল্ডার-শিকল তৈরি হবে। পরিষ্কার আর্কাইভ চাইলে `-C` ব্যবহার করো (নিচে দেখো)।

### ৩৪.৩.২ শুরুর `/` সরিয়ে দেওয়ার বার্তা

```
$ tar -czf etc-backup.tar.gz /etc/hostname /etc/hosts
tar: Removing leading `/' from member names
```

এটা কোনো error না, **নিরাপত্তা ব্যবস্থা**। আর্কাইভে `/etc/hosts` হুবহু থাকলে অন্য কেউ সেটা extract করার সময় তার আসল `/etc/hosts` ওভাররাইট হয়ে যেত। GNU tar তাই `etc/hosts` হিসেবে রাখে, যাতে extract সবসময় **বর্তমান ফোল্ডারের ভেতরে** হয়।

### ৩৪.৩.৩ `-C`: কোন ফোল্ডার থেকে/কোন ফোল্ডারে

`-C DIR` মানে "কাজ শুরুর আগে `cd DIR` করো"।

**বানানোর সময়:** পুরো path-এর ঝামেলা ছাড়া আর্কাইভ:

```
$ tar -czf /tmp/project.tar.gz -C /home/rahim/lab34 project
$ tar -tzf /tmp/project.tar.gz | head -3
project/
project/src/
project/src/main.c
```

একাধিক `-C` দিয়ে আলাদা আলাদা জায়গা থেকে একসাথে নেওয়া যায়:

```
$ tar -czf mix.tar.gz -C /etc hostname -C /home/rahim/lab34 project
```

**খোলার সময়:** নির্দিষ্ট ফোল্ডারে extract:

```
$ mkdir -p /tmp/restore
$ tar -xzf project.tar.gz -C /tmp/restore
$ ls /tmp/restore
project
```

> ⚠️ **সতর্কতা:** `-C`-এর লক্ষ্য ফোল্ডার **আগে থেকে থাকতে হবে**। না থাকলে `tar: /tmp/restore: Cannot open: No such file or directory`। তাই আগে `mkdir -p` চালাও।

### ৩৪.৩.৪ ভেতরে কী আছে দেখা (`-t`)

**আর্কাইভ খোলার আগে সবসময় একবার দেখে নাও।** এটা ভালো অভ্যাস।

```
$ tar -tzf project.tar.gz
project/
project/src/
project/src/main.c
project/docs/
project/docs/README.md
project/logs/
project/logs/app.log
```

বিস্তারিত দেখতে `-v`:

```
$ tar -tvzf project.tar.gz
drwxr-xr-x rahim/rahim       0 2026-11-12 10:04 project/
drwxr-xr-x rahim/rahim       0 2026-11-12 10:04 project/src/
-rw-r--r-- rahim/rahim      22 2026-11-12 10:04 project/src/main.c
drwxr-xr-x rahim/rahim       0 2026-11-12 10:04 project/docs/
-rw-r--r-- rahim/rahim      13 2026-11-12 10:04 project/docs/README.md
drwxr-xr-x rahim/rahim       0 2026-11-12 10:04 project/logs/
-rw-r--r-- rahim/rahim  588895 2026-11-12 10:04 project/logs/app.log
```

কলামগুলো: permission ([অধ্যায় ১৬](16-permissions.md)), owner/group, আকার (byte), তারিখ-সময়, path।

নির্দিষ্ট প্যাটার্ন খুঁজতে:

```
$ tar -tzf project.tar.gz --wildcards '*.c'
project/src/main.c
$ tar -tzf big.tar.gz | grep -i readme
```

> 💡 **টিপস:** আধুনিক GNU tar **extract আর list করার সময়** ফাইলের শুরুর "magic bytes" দেখে নিজেই কম্প্রেশন চিনে ফেলে। তাই `tar -tf project.tar.xz` বা `tar -xf project.tar.zst` কোনো `-z/-J` ছাড়াই কাজ করে। কিন্তু **বানানোর সময়** (`-c`) কম্প্রেসর বলে দিতে হয় (`-z`, `-J`, `--zstd` বা `-a`)। শুধু stdin থেকে পড়ার সময় (`-f -`) স্বয়ংক্রিয় চেনা কাজ করে না, তখন অপশন লিখতে হয়।

### ৩৪.৩.৫ আর্কাইভ খোলা (`-x`)

```
$ cd /tmp/restore
$ tar -xzvf ~/lab34/project.tar.gz
project/
project/src/
project/src/main.c
...
```

**শুধু একটা বা কয়েকটা ফাইল বের করা:** ঠিক যে path `-t`-এ দেখায়, সেটা লেখো:

```
$ tar -xzf project.tar.gz project/docs/README.md
$ tar -xzf project.tar.gz --wildcards 'project/src/*.c'
```

**`--strip-components`:** path-এর শুরুর অংশ ফেলে দেওয়া। সফটওয়্যার ডাউনলোড করলে প্রায়ই `nodejs-v22.1.0-linux-x64/bin/node` ধরনের গঠন থাকে। শুধু ভেতরের জিনিস চাইলে:

```
$ tar -xzf project.tar.gz --strip-components=1 -C /tmp/flat
$ ls /tmp/flat
docs  logs  src
```

**stdout-এ বের করা (`-O`):** ডিস্কে না লিখে সরাসরি দেখা:

```
$ tar -xzOf project.tar.gz project/docs/README.md
# My project
```

**ওভাররাইট নিয়ন্ত্রণ:**

| পরিস্থিতি | অপশন |
|---|---|
| ডিফল্ট | একই নামের ফাইল থাকলে ওভাররাইট করে |
| থাকলে error দাও, ছুঁয়ো না | `-k` / `--keep-old-files` |
| থাকলে চুপচাপ এড়িয়ে যাও | `--skip-old-files` |
| শুধু আর্কাইভের কপিটা নতুন হলে ওভাররাইট | `--keep-newer-files` |

**মালিকানা ও permission:**

- সাধারণ ইউজার হিসেবে extract করলে সব ফাইলের মালিক **তুমি** হবে, আর permission থেকে তোমার `umask` ([অধ্যায় ১৬](16-permissions.md)) অনুযায়ী কিছু bit বাদ যাবে।
- `sudo tar -x...` করলে আর্কাইভের আসল owner ও permission হুবহু ফিরে আসে (`--same-owner` ও `-p` root-এর জন্য ডিফল্ট)। সিস্টেম ব্যাকআপ ফেরত আনার সময় এটা জরুরি।
- উল্টোটা চাইলে, অর্থাৎ root হয়েও owner হিসেবে নিজেকে রাখতে, `--no-same-owner` দাও।

### ৩৪.৩.৬ বাদ দেওয়া (`--exclude`)

```
$ tar -czf project.tar.gz --exclude='*.log' project/
$ tar -czf site.tar.gz \
      --exclude='node_modules' \
      --exclude='.git' \
      --exclude='*.tmp' \
      --exclude='project/cache/*' \
      project/
```

একটা ফাইলে প্যাটার্নের তালিকা রেখে:

```
$ cat exclude.txt
*.log
node_modules
.git
$ tar -czf project.tar.gz -X exclude.txt project/
```

অন্যান্য সুবিধাজনক বাদ দেওয়ার অপশন:

| অপশন | কাজ |
|---|---|
| `--exclude-vcs` | `.git`, `.svn`, `.hg`, `CVS` ইত্যাদি বাদ |
| `--exclude-vcs-ignores` | `.gitignore` ফাইলে লেখা জিনিস বাদ |
| `--exclude-caches` | `CACHEDIR.TAG` ফাইলওয়ালা ফোল্ডারের ভেতরের জিনিস বাদ |
| `--exclude-tag=FILE` | যে ফোল্ডারে FILE আছে তার ভেতরের জিনিস বাদ |
| `--one-file-system` | অন্য মাউন্ট করা ফাইলসিস্টেমে ঢুকবে না (যেমন `/proc`) |

> ⚠️ **সতর্কতা:** পুরোনো GNU tar-এ `--exclude` অপশনটা ফাইল/ফোল্ডারের নামের **আগে** লিখতে হয়। নতুন সংস্করণে পরে দিলেও সাধারণত কাজ করে, তবে সতর্কবার্তা দেয়। সবচেয়ে নিরাপদ অভ্যাস: **সব অপশন আগে, ফাইলের নাম শেষে।**

> 💡 **টিপস:** প্যাটার্ন সবসময় একক উদ্ধৃতিতে (`'*.log'`) রাখো। না রাখলে শেল নিজেই `*.log` বর্তমান ফোল্ডারের ফাইলনামে বদলে দেবে (globbing, [অধ্যায় ৯](09-file-management.md)), তখন tar ভুল প্যাটার্ন পাবে।

### ৩৪.৩.৭ ফাইলের তালিকা অন্য কমান্ড থেকে (`-T`)

শেষ ৭ দিনে বদলানো ফাইলগুলোর ব্যাকআপ ([অধ্যায় ১২](12-file-khoja.md)-এর `find`):

```
$ find project -type f -mtime -7 -print0 | tar --null -czf recent.tar.gz -T -
```

এখানে `-print0` আর `--null` ফাইলনামে স্পেস বা নতুন লাইন থাকলেও সমস্যা হতে দেয় না। `-T -` মানে "তালিকা stdin থেকে পড়ো"।

### ৩৪.৩.৮ যোগ, আপডেট, মোছা, তুলনা

```
$ tar -rf project.tar newfile.txt          # শেষে যোগ
$ tar -uf project.tar project/             # শুধু বদলানো ফাইল যোগ
$ tar --delete -f project.tar project/logs/app.log
$ tar -df project.tar                      # ডিস্কের সাথে তুলনা
project/src/main.c: Mod time differs
project/src/main.c: Size differs
```

> ⚠️ **সতর্কতা:** `-r`, `-u`, `--delete` **কম্প্রেস করা আর্কাইভে কাজ করে না।** `tar -rzf x.tar.gz` চালালে `tar: Cannot update compressed archives` error আসবে। কারণ কম্প্রেস করা স্রোতের মাঝখানে কিছু বসাতে বা মুছতে হলে পুরোটা আবার কম্প্রেস করতে হয়। তখন: ডিকম্প্রেস → বদলাও → আবার কম্প্রেস।

> 📝 **মনে রাখো:** `-u` পুরোনো কপি মোছে না, শুধু নতুনটা শেষে যোগ করে। extract করলে শেষেরটা (নতুনটা) আগেরটার উপর লেখা হয়, তাই ফলাফল ঠিক থাকে, কিন্তু আর্কাইভের আকার বাড়ে।

### ৩৪.৩.৯ অগ্রগতি দেখা

বড় আর্কাইভে কতটুকু হলো জানতে `pv` (Pipe Viewer, `sudo apt install pv` / `sudo dnf install pv`):

```
$ tar -cf - project/ | pv | gzip > project.tar.gz
 590KiB 0:00:00 [ 115MiB/s] [ <=>                       ]

$ pv project.tar.gz | tar -xzf -
 122KiB 0:00:00 [ 210MiB/s] [================================>] 100%
```

অথবা tar-এর নিজস্ব checkpoint:

```
$ tar --checkpoint=1000 --checkpoint-action=dot -czf big.tar.gz bigdir/
..........
```

---

## ৩৪.৪ `gzip`: সবচেয়ে প্রচলিত কম্প্রেসর

`gzip` (GNU zip) ১৯৯২ সাল থেকে আছে। এটা দ্রুত, সব জায়গায় থাকে, ওয়েব সার্ভার এটা দিয়েই HTTP response কম্প্রেস করে (`Content-Encoding: gzip`), আর লিনাক্সের লগ রোটেশন (`/var/log/syslog.2.gz`) এটাই ব্যবহার করে।

### syntax ও আচরণ

```
gzip [অপশন] ফাইল...
```

**গুরুত্বপূর্ণ:** ডিফল্টে `gzip` **মূল ফাইলটা মুছে** তার জায়গায় `.gz` ফাইল রাখে। অর্থাৎ ফাইল বদলে যায়, কপি থাকে না।

```
$ ls
notes.txt
$ gzip notes.txt
$ ls
notes.txt.gz
$ gunzip notes.txt.gz
$ ls
notes.txt
```

### অপশনের টেবিল

| অপশন | কাজ |
|---|---|
| `-k` / `--keep` | মূল ফাইল রেখে দাও (gzip 1.6+) |
| `-d` / `--decompress` | ডিকম্প্রেস (`gunzip`-এর সমান) |
| `-c` / `--stdout` | ফলাফল stdout-এ পাঠাও, মূল ফাইল অক্ষত থাকে |
| `-1` … `-9` | লেভেল; `-1` = `--fast`, `-9` = `--best`, ডিফল্ট `-6` |
| `-l` / `--list` | কম্প্রেসড/আসল আকার ও অনুপাত দেখাও |
| `-t` / `--test` | ফাইল ঠিক আছে কি না যাচাই |
| `-v` / `--verbose` | প্রতিটা ফাইলের অনুপাত দেখাও |
| `-r` / `--recursive` | ফোল্ডারের ভেতরের **প্রতিটা ফাইল আলাদা করে** কম্প্রেস |
| `-f` / `--force` | জোর করে (আগে থেকে `.gz` থাকলে বা symlink হলেও) |
| `-n` / `--no-name` | মূল নাম ও সময় header-এ রেখো না (reproducible build-এ কাজে লাগে) |
| `-N` / `--name` | ডিকম্প্রেসের সময় header-এর মূল নাম ও সময় ফেরত আনো |

### উদাহরণ

```
$ gzip -kv project/logs/app.log
project/logs/app.log:	 78.9% -- created project/logs/app.log.gz

$ gzip -l project/logs/app.log.gz
         compressed        uncompressed  ratio uncompressed_name
             124185              588895  78.9% project/logs/app.log

$ gzip -t project/logs/app.log.gz && echo "ফাইল ঠিক আছে"
ফাইল ঠিক আছে
```

মূল ফাইল না ছুঁয়ে অন্য নামে বানানো:

```
$ gzip -c report.csv > /backup/report-2026-11-12.csv.gz
```

> ⚠️ **সতর্কতা:** `gzip -r folder/` ফোল্ডারটাকে একটা `.gz` বানায় **না**। এটা ভেতরের প্রতিটা ফাইলকে `a.txt.gz`, `b.txt.gz` এভাবে আলাদা করে কম্প্রেস করে আর মূলগুলো মুছে দেয়। পুরো ফোল্ডার এক ফাইলে চাইলে `tar -czf` ব্যবহার করো।

> 📝 **মনে রাখো:** `gzip -l`-এ দেখানো আসল আকার ৪ GB-এর বেশি ফাইলের জন্য ভুল হতে পারে, কারণ gzip-এর trailer-এ আকার মাত্র ৩২ bit-এ রাখা হয়। নিশ্চিত হতে `zcat file.gz | wc -c` চালাও।

### `gunzip`, `zcat` ও z-পরিবার

| কমান্ড | কাজ | সমতুল্য |
|---|---|---|
| `gunzip f.gz` | ডিকম্প্রেস করে `f` বানায়, `.gz` মুছে | `gzip -d` |
| `zcat f.gz` | ডিকম্প্রেস করে স্ক্রিনে দেখায়, ফাইল অক্ষত | `gzip -dc` |
| `zless f.gz` | `less` দিয়ে পাতায় পাতায় পড়া | |
| `zmore f.gz` | `more` দিয়ে পড়া | |
| `zgrep pattern f.gz` | খোলা ছাড়াই `grep` | |
| `zegrep`, `zfgrep` | `grep -E`, `grep -F`-এর z-সংস্করণ | |
| `zdiff a.gz b.gz` | দুটো কম্প্রেসড ফাইলের `diff` | |
| `zcmp` | `cmp`-এর z-সংস্করণ | |
| `zforce` | `.gz` ছাড়া নামের gzip ফাইলে `.gz` লাগায় | |
| `znew` | পুরোনো `.Z` ফাইলকে `.gz`-এ রূপান্তর | |

লগ নিয়ে কাজের বাস্তব উদাহরণ ([অধ্যায় ২৮](28-logs.md)):

```
$ ls /var/log/syslog*          # Ubuntu/Debian; RHEL/Fedora-তে /var/log/messages*
/var/log/syslog  /var/log/syslog.1  /var/log/syslog.2.gz  /var/log/syslog.3.gz

$ zgrep -i "error" /var/log/syslog.2.gz | head -3
$ zcat /var/log/syslog.*.gz | grep -c "CRON"
42
$ zless /var/log/syslog.2.gz
```

> 💡 **টিপস:** আধুনিক `less` (অনেক distro-তে `lesspipe`-এর মাধ্যমে) সরাসরি `less file.gz` দিয়েও খুলতে পারে। Ubuntu/Debian-এ `~/.bashrc`-এ সাধারণত `eval "$(lesspipe)"` লাইনটা থাকে।

### Multi-core gzip: `pigz`

সাধারণ `gzip` একটা CPU core ব্যবহার করে। `pigz` (parallel implementation of gzip) সব core ব্যবহার করে, আর একই `.gz` ফরম্যাট বানায়:

```
$ sudo apt install pigz       # Debian/Ubuntu
$ sudo dnf install pigz       # Fedora/RHEL
$ pigz -k -p 8 bigfile.sql    # ৮টা thread
$ tar -I pigz -cf backup.tar.gz /data
```

---

## ৩৪.৫ `bzip2`

`bzip2` gzip-এর চেয়ে ভালো ছোট করে কিন্তু বেশ ধীর, বিশেষ করে ডিকম্প্রেশনে। এটা Burrows-Wheeler Transform (BWT) ব্যবহার করে। BWT ডেটা ব্লক ধরে এমনভাবে সাজায় যে একই রকম অক্ষর পাশাপাশি চলে আসে, তারপর সেটা সহজে কম্প্রেস হয়।

আজকাল নতুন কাজে `bzip2`-এর জায়গা প্রায় পুরোটাই `xz` আর `zstd` নিয়ে নিয়েছে। তবে পুরোনো আর্কাইভে অনেক `.bz2` পাবে, তাই চেনা দরকার।

| অপশন | কাজ |
|---|---|
| `-k` | মূল ফাইল রাখো |
| `-d` | ডিকম্প্রেস (`bunzip2`) |
| `-c` | stdout-এ |
| `-1` … `-9` | **ব্লকের আকার** (১০০ KB … ৯০০ KB), ডিফল্ট `-9` |
| `-t` | যাচাই |
| `-v` | বিস্তারিত |
| `-f` | জোর করে |

```
$ bzip2 -kv app.log
  app.log:  12.517:1,  0.639 bits/byte, 92.01% saved, 588895 in, 47048 out.
$ bunzip2 -k app.log.bz2
$ bzcat app.log.bz2 | tail -1
100000
```

সঙ্গী কমান্ড: `bunzip2`, `bzcat`, `bzless`, `bzmore`, `bzgrep`, `bzdiff`, `bzip2recover` (নষ্ট `.bz2` ফাইল থেকে অক্ষত ব্লকগুলো উদ্ধার করে)।

Multi-core সংস্করণ: `pbzip2` বা `lbzip2`।

---

## ৩৪.৬ `xz`: সর্বোচ্চ ছোট করার জন্য

`xz` LZMA2 অ্যালগরিদম ব্যবহার করে (7-Zip-এর মূল অ্যালগরিদমের উত্তরসূরি)। সাধারণ টুলগুলোর মধ্যে এটা সাধারণত **সবচেয়ে ছোট** ফাইল বানায়, কিন্তু কম্প্রেস করতে ধীর আর বেশি মেমোরি খায়। ডিকম্প্রেশন মোটামুটি দ্রুত (bzip2-এর চেয়ে অনেক দ্রুত)।

কোথায় দেখবে: লিনাক্স কার্নেল সোর্স (`linux-6.x.tar.xz`), অনেক সফটওয়্যার রিলিজ, Debian প্যাকেজের ভেতরের ডেটা, Fedora/RHEL-এর কার্নেল মডিউল (`.ko.xz`)।

| অপশন | কাজ |
|---|---|
| `-k` | মূল ফাইল রাখো |
| `-d` | ডিকম্প্রেস (`unxz`) |
| `-c` | stdout-এ |
| `-0` … `-9` | লেভেল, ডিফল্ট `-6` |
| `-e` / `--extreme` | একই লেভেলে আরও একটু ছোট, আরও ধীর (যেমন `-9e`) |
| `-T N` / `--threads=N` | N thread; `-T0` = যতগুলো core আছে |
| `-l` / `--list` | আর্কাইভের তথ্য দেখাও |
| `-t` | যাচাই |
| `-v` | অগ্রগতি দেখাও |
| `-M LIMIT` / `--memlimit=` | মেমোরির সীমা (যেমন `-M 1GiB`) |
| `--format=lzma` | পুরোনো `.lzma` ফরম্যাট |

```
$ xz -kv -T0 app.log
app.log (1/1)
  100 %      38.2 KiB / 575.1 KiB = 0.066

$ xz -l app.log.xz
Strms  Blocks   Compressed Uncompressed  Ratio  Check   Filename
    1       1     38.2 KiB    575.1 KiB  0.066  CRC64   app.log.xz

$ xzcat app.log.xz | head -2
1
2
```

সঙ্গী কমান্ড: `unxz`, `xzcat`, `xzless`, `xzmore`, `xzgrep`, `xzdiff`, আর পুরোনো `.lzma`-র জন্য `lzma`, `unlzma`, `lzcat`।

**মেমোরির ব্যাপার:** `xz -9` কম্প্রেস করতে প্রায় ৬৭৪ MiB, আর ডিকম্প্রেস করতে প্রায় ৬৫ MiB মেমোরি লাগে। `-6` (ডিফল্ট) কম্প্রেস করতে প্রায় ৯৪ MiB। থ্রেড যত, কম্প্রেশন-মেমোরিও প্রায় তত গুণ। ছোট VPS বা Raspberry Pi-তে `-9 -T0` চালালে মেমোরি শেষ হয়ে যেতে পারে।

> 📝 **মনে রাখো:** ২০২৪ সালের মার্চে **xz-utils backdoor (CVE-2024-3094)** ধরা পড়েছিল। কেউ কয়েক বছর ধরে বিশ্বাস অর্জন করে xz-এর 5.6.0 ও 5.6.1 সংস্করণে লুকানো কোড ঢুকিয়েছিল, যার লক্ষ্য ছিল নির্দিষ্ট পরিস্থিতিতে SSH সার্ভারে অনুপ্রবেশ। মূলত কিছু rolling ও পরীক্ষামূলক distro (Fedora Rawhide/40 beta, Debian unstable, Kali, Arch ইত্যাদি) আক্রান্ত হয়েছিল, স্থিতিশীল রিলিজ নয়। এটা একটা বিখ্যাত **supply-chain attack**। এর শিক্ষা নিয়ে [অধ্যায় ৪৫](45-security-hardening.md)-এ আরও আলোচনা আছে। বর্তমান সংস্করণগুলো নিরাপদ, তাই xz ব্যবহারে ভয়ের কিছু নেই, সিস্টেম আপডেট রাখলেই হলো।

---

## ৩৪.৭ `zstd`: আধুনিক সব-কাজের কম্প্রেসর

`zstd` (Zstandard) Facebook (Meta)-এর Yann Collet ২০১৬ সালে বানিয়েছেন। এর বিশেষত্ব: **খুব বিস্তৃত লেভেলের পরিসর।** নিচু লেভেলে এটা gzip-এর চেয়ে অনেক দ্রুত এবং প্রায়ই বেশি ছোট করে, আর উঁচু লেভেলে xz-এর কাছাকাছি অনুপাত দেয়। সবচেয়ে বড় কথা, **ডিকম্প্রেশন সব লেভেলেই খুব দ্রুত।**

কোথায় ব্যবহার হয়:
- Arch Linux প্যাকেজ (`.pkg.tar.zst`)
- Fedora-র RPM প্যাকেজের payload
- Ubuntu-র `.deb` প্যাকেজ (21.10 থেকে) ও initramfs
- Btrfs ফাইলসিস্টেমের স্বচ্ছ (transparent) কম্প্রেশন (`compress=zstd`)
- লিনাক্স কার্নেল ইমেজ ও মডিউল কম্প্রেশনের একটা বিকল্প

### ইনস্টল

```
$ sudo apt install zstd       # Debian/Ubuntu (প্রায়ই আগে থেকেই থাকে)
$ sudo dnf install zstd       # Fedora/RHEL
```

### অপশনের টেবিল

| অপশন | কাজ |
|---|---|
| (কিছু না) | কম্প্রেস; **মূল ফাইল রেখে দেয়** (gzip-এর উল্টো) |
| `--rm` | কাজ শেষে মূল ফাইল মুছে দাও |
| `-d` | ডিকম্প্রেস (`unzstd`) |
| `-c` | stdout-এ |
| `-o FILE` | আউটপুট ফাইলের নাম |
| `-1` … `-19` | লেভেল, ডিফল্ট `-3` |
| `--ultra -20` … `-22` | আরও উঁচু লেভেল (প্রচুর মেমোরি লাগে) |
| `--fast=N` | ঋণাত্মক লেভেল: আরও দ্রুত, কম ছোট |
| `-T N` | N thread; `-T0` = সব core |
| `--long` | বড় জানালা (ডিফল্ট ১২৮ MB), বড় ফাইলে দূরের পুনরাবৃত্তি ধরে |
| `-l` | তথ্য দেখাও |
| `-t` | যাচাই |
| `-v` / `-q` | বেশি/কম বার্তা |
| `-b#` | নিজের মেশিনে # লেভেলে benchmark |
| `--train` | ছোট ছোট অনেক ফাইলের জন্য dictionary বানাও |

### উদাহরণ

```
$ zstd app.log
app.log              : 12.66%   (   575 KiB =>   72.8 KiB, app.log.zst)

$ ls app.log*
app.log  app.log.zst

$ zstd -19 -T0 app.log -o app19.zst
$ zstd -d app.log.zst -o restored.log
$ zstdcat app.log.zst | wc -l
100000
$ zstd -l app.log.zst
Frames  Skips  Compressed  Uncompressed  Ratio  Check  Filename
     1      0    72.8 KiB       575 KiB  7.902  XXH64  app.log.zst
```

নিজের মেশিনে লেভেল ১ থেকে ১৯ পর্যন্ত গতি বনাম অনুপাত মেপে দেখো:

```
$ zstd -b1 -e19 app.log
```

সঙ্গী কমান্ড: `unzstd`, `zstdcat`, `zstdless`, `zstdgrep`, `zstdmt` (= `zstd -T0`)।

> 💡 **টিপস:** নতুন কাজে কোন কম্প্রেসর নেবে বুঝতে না পারলে **`zstd` নাও।** দৈনন্দিন ব্যাকআপে `-3` (ডিফল্ট) বা `-T0`, আর একবার বানিয়ে বহুবার বিতরণ করবে এমন জিনিসে `-19`। শুধু যেখানে প্রাপকের মেশিনে zstd না-ও থাকতে পারে (খুব পুরোনো সার্ভার), সেখানে নিরাপদ বিকল্প `gzip`।

> ⚠️ **সতর্কতা:** `--long` দিয়ে বানানো বা `--ultra -20`-এর উপরের লেভেলের ফাইল ডিকম্প্রেস করতে বড় জানালা লাগে। প্রাপক যদি `zstd -d` চালিয়ে "Frame requires too much memory" error পায়, তাকে `zstd -d --long=31` বা `--memory=2GB` দিতে হতে পারে।

### আরেকটা দ্রুতগামী: `lz4`

`lz4` (একই লেখকের) কম্প্রেশন অনুপাতে সবচেয়ে দুর্বল, কিন্তু গতিতে অবিশ্বাস্য, প্রায় মেমোরি কপির গতিতে। ব্যবহার হয় যেখানে CPU বাঁচানো বেশি জরুরি: zram (RAM-এ কম্প্রেস করা swap), কার্নেল/initramfs-এর দ্রুত বুট, ডাটাবেস।

```
$ lz4 app.log            # app.log.lz4 বানায়, মূল ফাইল রেখে দেয়
$ lz4 -d app.log.lz4 out.log
```

---

## ৩৪.৮ তুলনা টেবিল: গতি বনাম অনুপাত

নিচের টেবিলটা **আনুমানিক ও আপেক্ষিক**। আসল সংখ্যা ডেটার ধরন, CPU আর সংস্করণের উপর অনেক নির্ভর করে। ধারণা হিসেবে ধরো: প্রায় ১ GB সাধারণ টেক্সট/লগ/সোর্স কোড, একটা আধুনিক CPU core।

| টুল ও লেভেল | কম্প্রেস গতি | ডিকম্প্রেস গতি | অনুপাত (বড় = ভালো) | মেমোরি | মন্তব্য |
|---|---|---|---|---|---|
| `lz4` | ⚡⚡⚡⚡⚡ (কয়েকশো MB/s) | ⚡⚡⚡⚡⚡ (GB/s) | ★☆☆☆☆ (~২×) | খুব কম | গতিই সব |
| `zstd -1` | ⚡⚡⚡⚡ | ⚡⚡⚡⚡⚡ | ★★☆☆☆ (~২.৮×) | কম | gzip-এর চেয়ে দ্রুত ও ভালো |
| `zstd -3` (ডিফল্ট) | ⚡⚡⚡⚡ | ⚡⚡⚡⚡⚡ | ★★★☆☆ (~৩×) | কম | **সেরা সর্বজনীন বিকল্প** |
| `gzip -1` | ⚡⚡⚡ | ⚡⚡⚡ | ★★☆☆☆ (~২.৭×) | খুব কম | |
| `gzip -6` (ডিফল্ট) | ⚡⚡ | ⚡⚡⚡ | ★★☆☆☆ (~৩×) | খুব কম | সব জায়গায় চলে |
| `gzip -9` | ⚡ | ⚡⚡⚡ | ★★☆☆☆ (-৬-এর চেয়ে সামান্য ভালো) | খুব কম | অতিরিক্ত সময়ের তুলনায় লাভ কম |
| `bzip2 -9` (ডিফল্ট) | ⚡ | 🐢 | ★★★☆☆ (~৩.৫×) | কম | ডিকম্প্রেশন ধীর |
| `zstd -19` | 🐢 | ⚡⚡⚡⚡⚡ | ★★★★☆ (~৩.৮×) | মাঝারি | বিতরণের জন্য চমৎকার |
| `xz -6` (ডিফল্ট) | 🐢 | ⚡⚡ | ★★★★☆ (~৪×) | মাঝারি | |
| `xz -9e` | 🐢🐢 | ⚡⚡ | ★★★★★ | বেশি | সর্বোচ্চ ছোট |

**মোটামুটি নিয়ম (একই ধরনের ডেটায়):**

```
অনুপাত (ছোট করার ক্ষমতা), কম থেকে বেশি:
  lz4  <  gzip  ≈  zstd(নিচু)  <  bzip2  <  zstd(উঁচু)  ≤  xz

কম্প্রেস গতি, দ্রুত থেকে ধীর:
  lz4  >  zstd(নিচু)  >  gzip  >  bzip2  >  zstd(উঁচু)  ≈  xz

ডিকম্প্রেস গতি, দ্রুত থেকে ধীর:
  lz4  >  zstd(যেকোনো লেভেল)  >  gzip  >  xz  >  bzip2
```

### কখন কোনটা? সিদ্ধান্তের ছক

```
                     কী কম্প্রেস করবে?
                            │
          ┌─────────────────┼──────────────────┐
          ▼                 ▼                  ▼
    ছবি/ভিডিও/zip      Windows ব্যবহারকারীকে    লিনাক্সেই থাকবে
    (আগে থেকে          পাঠাবে
     কম্প্রেস করা)            │                  │
          │                 ▼                  ▼
          ▼              zip বা 7z      গতি জরুরি নাকি আকার?
   শুধু tar, কম্প্রেস                         │
   কোরো না                  ┌────────────────┼─────────────────┐
                            ▼                ▼                 ▼
                       খুব দ্রুত চাই     দৈনন্দিন ব্যাকআপ   একবার বানাবে,
                       (লাইভ ডেটা)       / সাধারণ কাজ      বহুবার ডাউনলোড
                            │                │                 │
                            ▼                ▼                 ▼
                        lz4 /           zstd -3 -T0       xz -9 -T0 বা
                        zstd --fast     (পুরোনো সিস্টেম     zstd -19 -T0
                                         হলে gzip)
```

---

## ৩৪.৯ এক্সটেনশন চেনা ও আসল ফরম্যাট বের করা

### এক্সটেনশনের তালিকা

| এক্সটেনশন | কী | বানানো | খোলা |
|---|---|---|---|
| `.tar` | শুধু আর্কাইভ | `tar -cf` | `tar -xf` |
| `.tar.gz`, `.tgz` | tar + gzip | `tar -czf` | `tar -xzf` (বা `tar -xf`) |
| `.tar.bz2`, `.tbz2`, `.tbz`, `.tb2` | tar + bzip2 | `tar -cjf` | `tar -xjf` |
| `.tar.xz`, `.txz` | tar + xz | `tar -cJf` | `tar -xJf` |
| `.tar.zst`, `.tzst` | tar + zstd | `tar --zstd -cf` | `tar --zstd -xf` |
| `.tar.lz4` | tar + lz4 | `tar -I lz4 -cf` | `tar -I lz4 -xf` |
| `.tar.Z`, `.taz` | tar + পুরোনো `compress` | | `tar -xZf` |
| `.tar.lzma`, `.tlz` | tar + lzma | `tar --lzma -cf` | `tar --lzma -xf` |
| `.gz` | একটা gzip-করা ফাইল | `gzip` | `gunzip` |
| `.bz2` | একটা bzip2 ফাইল | `bzip2` | `bunzip2` |
| `.xz` | একটা xz ফাইল | `xz` | `unxz` |
| `.zst` | একটা zstd ফাইল | `zstd` | `unzstd` |
| `.lz4` | একটা lz4 ফাইল | `lz4` | `lz4 -d` |
| `.Z` | পুরোনো Unix `compress` | `compress` (ncompress প্যাকেজ) | `uncompress` বা `gunzip` |
| `.zip` | zip আর্কাইভ | `zip -r` | `unzip` |
| `.7z` | 7-Zip আর্কাইভ | `7z a` | `7z x` |
| `.rar` | RAR আর্কাইভ | `rar a` | `unrar x` |
| `.jar`, `.war`, `.apk`, `.docx`, `.xlsx`, `.odt`, `.epub` | ভেতরে আসলে zip | | `unzip` |
| `.deb` | Debian প্যাকেজ (`ar` আর্কাইভ, ভেতরে tar) | `dpkg-deb` | `dpkg-deb -x`, `ar x` |
| `.rpm` | RPM প্যাকেজ (header + cpio payload) | `rpmbuild` | `rpm2cpio f.rpm \| cpio -idmv` |
| `.cpio` | cpio আর্কাইভ (initramfs-এ ব্যবহৃত) | `cpio -o` | `cpio -idv` |
| `.iso` | CD/DVD ডিস্ক ইমেজ | `xorriso`/`genisoimage` | `mount -o loop` ([অধ্যায় ৩২](32-disk-partition-mount.md)), `7z x` |
| `.img`, `.img.xz` | ডিস্ক ইমেজ | `dd` ([অধ্যায় ৩৫](35-backup-rsync.md)) | `xz -dc f.img.xz \| dd ...` |

> 💡 **টিপস:** `.docx` আসলে একটা zip! চেষ্টা করে দেখো: `unzip -l report.docx`। ভেতরে `word/document.xml` ইত্যাদি XML ফাইল পাবে।

### এক্সটেনশন মিথ্যা বলতে পারে: `file` কমান্ড

লিনাক্সে ফাইলের নাম শুধু একটা লেবেল ([অধ্যায় ১](01-linux-ki-o-keno.md))। আসল ফরম্যাট বোঝা যায় ফাইলের শুরুর কয়েকটা byte, অর্থাৎ **magic number** দেখে। `file` কমান্ড ঠিক এটাই করে।

```
$ file project.tar.gz mystery.bin app.log.zst doc.docx
project.tar.gz: gzip compressed data, from Unix, original size modulo 2^32 614400
mystery.bin:    XZ compressed data, checksum CRC64
app.log.zst:    Zstandard compressed data (v0.8+), Dictionary ID: None
doc.docx:       Microsoft Word 2007+
```

| ফরম্যাট | প্রথম byte-গুলো (hex) | ASCII-তে |
|---|---|---|
| gzip | `1f 8b` | |
| bzip2 | `42 5a 68` | `BZh` |
| xz | `fd 37 7a 58 5a 00` | `.7zXZ.` |
| zstd | `28 b5 2f fd` | |
| zip | `50 4b 03 04` | `PK..` (আবিষ্কারক Phil Katz-এর নামের আদ্যক্ষর) |
| 7z | `37 7a bc af 27 1c` | `7z....` |
| rar | `52 61 72 21 1a 07` | `Rar!..` |
| tar (ustar) | ২৫৭ নম্বর byte থেকে | `ustar` |

নিজে দেখো:

```
$ head -c 4 project.tar.gz | xxd
00000000: 1f8b 0800                                ....
$ head -c 4 app.log.zst | od -An -tx1
 28 b5 2f fd
```

কম্প্রেস করা tar-এর ভেতরে আসলে কী আছে দেখতে `-z` (look inside) অপশন:

```
$ file -z project.tar.gz
project.tar.gz: POSIX tar archive (GNU) (gzip compressed data, from Unix)
```

---

## ৩৪.১০ `zip` ও `unzip`: Windows-এর সাথে বন্ধুত্ব

### `.zip` আর `.tar.gz`-এর গঠনগত পার্থক্য

```
 .tar.gz  ("solid": পুরো স্রোত একসাথে কম্প্রেস)
 ┌──────────────────────────────────────────────────────┐
 │ gzip( [hdr a][data a][hdr b][data b][hdr c][data c] ) │
 └──────────────────────────────────────────────────────┘
   + একই ধরনের ফাইলগুলোর মধ্যের মিল কাজে লাগে → বেশি ছোট
   − একটা ফাইল বের করতে শুরু থেকে পড়তে হয়
   − মাঝখানে নষ্ট হলে তার পরের সবকিছু ঝুঁকিতে

 .zip  (প্রতিটা ফাইল আলাদা কম্প্রেস, শেষে সূচিপত্র)
 ┌──────────┬──────────┬──────────┬────────────────────┐
 │[hdr a]   │[hdr b]   │[hdr c]   │ Central Directory  │
 │deflate(a)│deflate(b)│deflate(c)│ (সব ফাইলের সূচিপত্র │
 │          │          │          │  ও তাদের অবস্থান)   │
 └──────────┴──────────┴──────────┴────────────────────┘
   + সূচিপত্র শেষে → তালিকা তাৎক্ষণিক, যেকোনো ফাইল সরাসরি বের করা যায়
   + একটা ফাইল নষ্ট হলে বাকিগুলো সাধারণত টিকে যায়
   − ফাইলগুলোর মধ্যের মিল কাজে লাগে না → সাধারণত বড়
   − লিনাক্সের owner/group, ACL, কিছু বিশেষ ফাইল ঠিকমতো রাখে না
```

> 📝 **মনে রাখো:** লিনাক্স সার্ভারের ব্যাকআপ, যেখানে permission, owner, symlink সব অক্ষত রাখা জরুরি, সেখানে **tar** ব্যবহার করো। Windows/Mac ব্যবহারকারীকে ফাইল পাঠাতে **zip** ব্যবহার করো।

### ইনস্টল

```
$ sudo apt install zip unzip     # Debian/Ubuntu
$ sudo dnf install zip unzip     # Fedora/RHEL
```

### `zip`

```
zip [অপশন] আর্কাইভ.zip ফাইল...
```

| অপশন | কাজ |
|---|---|
| `-r` | ফোল্ডারের ভেতরে recursive ঢোকো (**এটা না দিলে ফোল্ডারের ভেতরের ফাইল যাবে না!**) |
| `-0` … `-9` | লেভেল; `-0` = শুধু রাখো, কম্প্রেস নয়; ডিফল্ট `-6` |
| `-e` | পাসওয়ার্ড দিয়ে এনক্রিপ্ট (প্রম্পটে জিজ্ঞেস করবে) |
| `-P pass` | কমান্ড লাইনেই পাসওয়ার্ড (**অনিরাপদ**, history-তে থাকে) |
| `-x PATTERN` | বাদ দাও |
| `-u` | শুধু নতুন/বদলানো ফাইল আপডেট |
| `-d` | আর্কাইভ থেকে মুছে দাও |
| `-m` | zip-এ ঢোকানোর পর মূল ফাইল মুছে দাও (move) |
| `-q` | চুপচাপ |
| `-y` | symlink-কে symlink হিসেবেই রাখো |
| `-j` | ফোল্ডারের path বাদ দিয়ে শুধু ফাইলের নাম রাখো |
| `-s SIZE` | আর্কাইভকে ভাগ করো (যেমন `-s 100m`) |
| `-T` | বানানোর পর যাচাই |

```
$ zip -r project.zip project/
  adding: project/ (stored 0%)
  adding: project/src/ (stored 0%)
  adding: project/src/main.c (stored 0%)
  adding: project/docs/ (stored 0%)
  adding: project/docs/README.md (stored 0%)
  adding: project/logs/ (stored 0%)
  adding: project/logs/app.log (deflated 79%)

$ zip -r site.zip website/ -x '*.git*' -x '*/node_modules/*'
$ zip -e secret.zip passwords.txt
Enter password:
Verify password:
  adding: passwords.txt (deflated 12%)
```

> ⚠️ **সতর্কতা:** `zip -e`-এর ডিফল্ট এনক্রিপশন (ZipCrypto) **খুব দুর্বল**, সহজেই ভাঙা যায়। সত্যিকারের গোপন জিনিসের জন্য `7z -p -mhe=on` (AES-256) বা `gpg` ব্যবহার করো ([অধ্যায় ৪৫](45-security-hardening.md))।

### `unzip`

| অপশন | কাজ |
|---|---|
| `-l` | তালিকা দেখাও |
| `-v` | বিস্তারিত তালিকা (অনুপাত, CRC সহ) |
| `-d DIR` | DIR-এ extract (না থাকলে বানিয়ে নেয়) |
| `-o` | জিজ্ঞেস না করে ওভাররাইট |
| `-n` | কখনো ওভাররাইট কোরো না |
| `-t` | যাচাই |
| `-q` | চুপচাপ |
| `-j` | ফোল্ডার কাঠামো বাদ দিয়ে সব ফাইল এক জায়গায় |
| `-p` | stdout-এ (পাইপের জন্য) |
| `-P pass` | পাসওয়ার্ড |
| `-x PATTERN` | বাদ দিয়ে extract |
| `-O CHARSET` | ফাইলনামের এনকোডিং (Windows থেকে আসা অ-ইংরেজি নামের জন্য, Debian/Ubuntu-র patched unzip-এ) |

```
$ unzip -l project.zip
Archive:  project.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2026-11-12 10:04   project/
        0  2026-11-12 10:04   project/src/
       22  2026-11-12 10:04   project/src/main.c
        0  2026-11-12 10:04   project/docs/
       13  2026-11-12 10:04   project/docs/README.md
        0  2026-11-12 10:04   project/logs/
   588895  2026-11-12 10:04   project/logs/app.log
---------                     -------
   588930                     7 files

$ unzip project.zip -d /tmp/unz
$ unzip -o project.zip 'project/src/*'    # শুধু src অংশ, ওভাররাইট
$ unzip -p project.zip project/docs/README.md
# My project
```

ফাইল আগে থেকে থাকলে unzip জিজ্ঞেস করে:

```
replace project/src/main.c? [y]es, [n]o, [A]ll, [N]one, [r]ename:
```

সঙ্গী কমান্ড: `zipinfo` (বিস্তারিত তথ্য), `zipgrep` (zip-এর ভেতরে grep), `zipsplit`, `zipnote`।

---

## ৩৪.১১ `7z` ও `rar`

### `7z`: 7-Zip

7-Zip-এর ফরম্যাট `.7z` সাধারণত LZMA/LZMA2 ব্যবহার করে, "solid" কম্প্রেশন করে, আর **AES-256 এনক্রিপশন** দেয়, যেটা ফাইলের নামও লুকাতে পারে। এর বড় সুবিধা: একটা টুল দিয়েই প্রায় সব ফরম্যাট খোলা যায় (zip, tar, gz, xz, iso, rar-এর অনেক সংস্করণ, cab, msi...)।

**ইনস্টল (distro অনুযায়ী আলাদা):**

| Distro | প্যাকেজ | কমান্ড |
|---|---|---|
| Ubuntu 24.04 / Debian 12 | `p7zip-full` (পুরোনো ঐতিহ্যবাহী port) | `7z`, `7za` |
| Ubuntu 24.04 / Debian 12 | `7zip` (মূল লেখকের অফিসিয়াল লিনাক্স সংস্করণ) | `7zz` (নতুন সংস্করণে `7z` নামেও) |
| Fedora | `7zip` (নতুন রিলিজে) বা `p7zip p7zip-plugins` | `7z` বা `7zz` |
| RHEL 9 / Rocky / Alma | EPEL repository থেকে `p7zip p7zip-plugins` | `7z` |

> 📝 **মনে রাখো:** প্যাকেজের নাম ও কমান্ডের নাম সময়ের সাথে বদলাচ্ছে (`p7zip` থেকে অফিসিয়াল `7zip`-এর দিকে)। ইনস্টলের পর `which 7z 7za 7zz` চালিয়ে দেখো কোনটা আছে। নিচের উদাহরণে `7z` লিখেছি; তোমার সিস্টেমে শুধু `7zz` থাকলে সেটাই লেখো, অপশন একই।

**7z-এর গঠন অন্যদের থেকে আলাদা:** প্রথমে একটা **কমান্ড অক্ষর** (ড্যাশ ছাড়া), তারপর সুইচ (`-` দিয়ে)।

| কমান্ড | কাজ |
|---|---|
| `a` | add: আর্কাইভ বানাও/যোগ করো |
| `x` | extract **পূর্ণ path সহ** (সাধারণত এটাই চাও) |
| `e` | extract **সব ফাইল এক ফোল্ডারে**, কাঠামো বাদ দিয়ে |
| `l` | তালিকা |
| `t` | যাচাই |
| `d` | মুছে দাও |
| `u` | আপডেট |

| সুইচ | কাজ |
|---|---|
| `-p` | পাসওয়ার্ড (শুধু `-p` দিলে প্রম্পটে জিজ্ঞেস করবে, যেটা নিরাপদ) |
| `-mhe=on` | ফাইলের নামও এনক্রিপ্ট করো (header encryption, শুধু `.7z`-এ) |
| `-mx=N` | লেভেল ০–৯ (`-mx=9` = ultra) |
| `-o DIR` | আউটপুট ফোল্ডার (**`-o` আর DIR-এর মাঝে কোনো স্পেস নেই**: `-o/tmp/out`) |
| `-t TYPE` | ফরম্যাট (`-tzip`, `-t7z`, `-ttar`) |
| `-v SIZE` | ভলিউমে ভাগ করো (`-v100m`) |
| `-x!PATTERN` | বাদ দাও |
| `-y` | সব প্রশ্নে হ্যাঁ |

```
$ 7z a project.7z project/
$ 7z a -p -mhe=on -mx=9 secret.7z private/
Enter password:
Verify password:

$ 7z l project.7z
$ 7z x project.7z -o/tmp/out
$ 7z x anything.iso -o./iso-contents     # ISO-ও খোলা যায়
$ 7z t project.7z
Everything is Ok
```

> ⚠️ **সতর্কতা:** `.7z` ফরম্যাট লিনাক্সের owner/group ও পুরো permission তথ্য রাখে না। সার্ভার ব্যাকআপের জন্য তাই আগে `tar` করে, তারপর দরকার হলে সেই `.tar`-কে 7z করো: `tar -cf - dir | 7z a -si dir.tar.7z`।

### `rar` ও `unrar`

RAR একটা **proprietary (মালিকানাধীন)** ফরম্যাট। Windows-এ WinRAR জনপ্রিয় বলে প্রায়ই `.rar` ফাইল পাবে।

| কাজ | টুল | ইনস্টল |
|---|---|---|
| খোলা (অফিসিয়াল) | `unrar` | Ubuntu: `multiverse` repo থেকে `sudo apt install unrar`; Debian: `non-free` থেকে; Fedora: RPM Fusion (nonfree) থেকে |
| খোলা (মুক্ত বিকল্প) | `unar`, `lsar` | `sudo apt install unar` / `sudo dnf install unar` |
| খোলা (7-Zip দিয়ে) | `7z x f.rar` | প্লাগইন/সংস্করণ অনুযায়ী RAR5 সমর্থন ভিন্ন হতে পারে |
| বানানো | `rar` | শেয়ারওয়্যার, RARLAB থেকে; মুক্ত সফটওয়্যারে রার বানানোর কোনো উপায় নেই |

```
$ unrar l movie.rar           # তালিকা
$ unrar x movie.rar           # পূর্ণ path সহ extract
$ unrar e movie.rar           # সব এক ফোল্ডারে
$ unrar t movie.rar           # যাচাই
$ unrar x movie.part1.rar     # বহু-খণ্ড আর্কাইভ: প্রথম খণ্ড দিলেই বাকিগুলো নিজে নেয়

$ unar movie.rar              # মুক্ত বিকল্প
$ lsar movie.rar
```

> 💡 **টিপস:** অন্যকে দেওয়ার জন্য নিজে কখনো `.rar` বানিয়ো না। মুক্ত ও সব জায়গায় খোলা যায় এমন ফরম্যাট বেছে নাও: `.zip`, `.7z`, বা লিনাক্স ব্যবহারকারীর জন্য `.tar.gz`/`.tar.zst`।

### `zip` বনাম `7z` বনাম `rar` বনাম `tar.*`

| বৈশিষ্ট্য | `.tar.gz` / `.tar.zst` / `.tar.xz` | `.zip` | `.7z` | `.rar` |
|---|---|---|---|---|
| লাইসেন্স | মুক্ত | মুক্ত | মুক্ত (LGPL) | মালিকানাধীন |
| লিনাক্স permission/owner/symlink | ✅ পুরোপুরি | ⚠️ আংশিক (permission ও symlink, owner নয়) | ❌ | ❌ |
| solid কম্প্রেশন | ✅ সবসময় | ❌ | ✅ (ডিফল্ট) | ঐচ্ছিক |
| দ্রুত তালিকা / random access | ❌ | ✅ | আংশিক | ✅ |
| শক্তিশালী এনক্রিপশন | ❌ (gpg লাগাও) | ⚠️ ঐচ্ছিক AES, ডিফল্ট দুর্বল | ✅ AES-256 | ✅ AES |
| Windows-এ বিল্ট-ইন খোলা | Windows 11-এ সম্প্রতি এসেছে | ✅ | Windows 11-এ সম্প্রতি এসেছে | Windows 11-এ সম্প্রতি এসেছে |
| পাইপ/স্ট্রিম | ✅ চমৎকার | ⚠️ সীমিত | ⚠️ সীমিত | ❌ |

---

## ৩৪.১২ পাইপ দিয়ে `ssh`-এর ভেতর দিয়ে `tar`

এই অধ্যায়ের সবচেয়ে শক্তিশালী কৌশল। SSH নিয়ে বিস্তারিত [অধ্যায় ৩৮](38-ssh-remote.md)-এ আছে, এখানে শুধু tar-এর সাথে এর জুটিটা দেখব।

### মূল ধারণা

`tar -f -` মানে "আর্কাইভ ফাইলে না লিখে **stdout-এ** লেখো" (`-c`-এর সাথে) বা "**stdin থেকে** পড়ো" (`-x`-এর সাথে)। আর `ssh host 'কমান্ড'` রিমোট মেশিনে কমান্ড চালায়, আর আমাদের stdin সেখানে পাঠিয়ে দেয়। দুটো জুড়লে:

```
  লোকাল মেশিন                                         রিমোট সার্ভার
 ┌──────────────────────┐                         ┌──────────────────────────┐
 │                      │   এনক্রিপ্টেড SSH টানেল   │                          │
 │ tar -czf - project/ ─┼──▶ ═══════════════════ ──┼▶ tar -xzf - -C /srv/app  │
 │  (stdout-এ স্রোত)     │   (bytes বয়ে যায়)       │  (stdin থেকে পড়ে খোলে)   │
 │                      │                         │                          │
 └──────────────────────┘                         └──────────────────────────┘
      ডিস্কে কোনো .tar.gz ফাইল তৈরি হয় না, কোনো পাশেই!
```

কেন `scp -r` বা `rsync`-এর বদলে এটা?
- **কোনো মধ্যবর্তী ফাইল লাগে না।** ১০০ GB ফোল্ডার পাঠাতে লোকালে ১০০ GB খালি জায়গা দরকার নেই।
- **হাজার হাজার ছোট ফাইলে দ্রুত।** `scp` প্রতিটা ফাইলের জন্য আলাদা আদান-প্রদান করে; tar সব এক স্রোতে পাঠায়।
- **permission, owner, symlink, timestamp অক্ষত থাকে।**
- পথে **কম্প্রেস** হয়, তাই ধীর নেটওয়ার্কে লাভ।

(বারবার সিঙ্ক করার জন্য অবশ্য `rsync` সেরা, কারণ শুধু পরিবর্তনটুকু পাঠায়। দেখো [অধ্যায় ৩৫](35-backup-rsync.md)।)

### ১. লোকাল থেকে রিমোটে পাঠানো (push)

```
$ tar -czf - project/ | ssh rahim@server.example.com 'tar -xzf - -C /home/rahim/'
```

রিমোটে আগে ফোল্ডার বানিয়ে নিয়ে:

```
$ tar -czf - project/ | ssh rahim@server 'mkdir -p /srv/app && tar -xzf - -C /srv/app'
```

### ২. রিমোট থেকে লোকালে আনা (pull)

```
$ ssh rahim@server 'tar -czf - -C /var/www html' | tar -xzf - -C ~/site-copy/
```

### ৩. রিমোট ফোল্ডারের ব্যাকআপ সরাসরি লোকাল ফাইলে

```
$ ssh rahim@server 'tar -czf - /etc' > server-etc-$(date +%F).tar.gz
tar: Removing leading `/' from member names
```

(বার্তাটা রিমোট tar-এর stderr থেকে আসছে, ব্যাকআপ ঠিকই আছে।)

উল্টোটা, লোকাল ব্যাকআপ সরাসরি রিমোট ফাইলে:

```
$ tar -czf - ~/documents | ssh rahim@backup-server 'cat > /backups/docs.tar.gz'
```

### ৪. দ্রুত নেটওয়ার্কে কম্প্রেশন ছাড়া, বা zstd দিয়ে

LAN-এ (১ Gbps বা বেশি) gzip-এর একক-core গতিই প্রায়ই বাধা হয়ে দাঁড়ায়। তখন:

```
# কম্প্রেশন ছাড়া (দ্রুত LAN-এ)
$ tar -cf - bigdir/ | ssh server 'tar -xf - -C /data'

# zstd দিয়ে, সব core ব্যবহার করে (দুই প্রান্তেই zstd থাকতে হবে)
$ tar -cf - bigdir/ | zstd -T0 -3 | ssh server 'zstd -d | tar -xf - -C /data'

# অগ্রগতি দেখে
$ tar -cf - bigdir/ | pv | zstd -T0 | ssh server 'zstd -d | tar -xf - -C /data'
```

> 💡 **টিপস:** `ssh -C` দিয়েও SSH নিজেই কম্প্রেস করতে পারে (zlib/gzip-এর মতো)। কিন্তু `tar -z` আর `ssh -C` **দুটো একসাথে দিয়ো না**। ডেটা দুবার কম্প্রেস করলে লাভ নেই, শুধু CPU নষ্ট।

### ৫. root-এর ফাইল, owner অক্ষত রেখে

```
$ sudo tar -cpf - -C / etc/nginx | ssh admin@server 'sudo tar -xpf - -C /'
```

> ⚠️ **সতর্কতা:** রিমোটে `sudo` পাসওয়ার্ড চাইলে এটা ঝামেলা করবে, কারণ stdin ইতিমধ্যে tar-এর ডেটা বইছে। সমাধান: রিমোটে ওই কমান্ডের জন্য NOPASSWD sudo নিয়ম ([অধ্যায় ১৮](18-sudo-root.md)), অথবা আগে লোকাল ফাইলে বানিয়ে `scp` করে তারপর রিমোটে খোলা। আর **`-C /`-তে extract করার আগে দুবার ভাবো**, কারণ সেটা রিমোটের আসল সিস্টেম ফাইল ওভাররাইট করবে।

### ৬. একই মেশিনে এক ডিস্ক থেকে আরেক ডিস্কে

ssh ছাড়াও এই পাইপ কাজের:

```
$ sudo tar -cf - -C /old/disk . | sudo tar -xpf - -C /new/disk
```

এতে hard link, symlink, permission, owner সব অক্ষত থেকে কপি হয় (আজকাল `cp -a` বা `rsync -aHAX` দিয়েও করা যায়)।

---

## ৩৪.১৩ বড় আর্কাইভ ভাগ করা ও জোড়া লাগানো

FAT32 পেনড্রাইভে ৪ GB-এর বড় ফাইল রাখা যায় না, ইমেইলে বড় ফাইল পাঠানো যায় না। তখন `split` ([অধ্যায় ২৩](23-text-tools.md)-এর কাছাকাছি টুল):

```
$ tar -czf - bigdir/ | split -b 1G -d - backup.tar.gz.part
$ ls
backup.tar.gz.part00  backup.tar.gz.part01  backup.tar.gz.part02

# জোড়া লাগিয়ে খোলা
$ cat backup.tar.gz.part* | tar -xzf -
```

`-b 1G` = প্রতি খণ্ড ১ GiB, `-d` = সংখ্যাযুক্ত প্রত্যয়, `-` = stdin থেকে পড়ো।

---

## ৩৪.১৪ নিরাপত্তা ও সাধারণ ভুল

### Tarbomb: বোমার মতো ছড়িয়ে পড়া আর্কাইভ

কেউ যদি ফোল্ডারে না রেখে সরাসরি একশোটা ফাইল আর্কাইভ করে, তুমি সেটা তোমার home ফোল্ডারে extract করলে একশোটা ফাইল ছড়িয়ে পড়বে, তোমার নিজের ফাইলের সাথে মিশে যাবে, এমনকি একই নামের ফাইল ওভাররাইট করবে।

**প্রতিরোধ:**

```
$ tar -tf unknown.tar.gz | head        # আগে দেখো
$ mkdir unknown && tar -xf unknown.tar.gz -C unknown   # আলাদা ফোল্ডারে খোলো
```

নিজে আর্কাইভ বানানোর সময়ও ভদ্রতা বজায় রাখো: সবকিছু এক মূল ফোল্ডারের ভেতরে রাখো (`project-1.0/...`)।

### Path traversal ও অবিশ্বস্ত আর্কাইভ

বিদ্বেষপূর্ণ আর্কাইভে `../../.bashrc` ধরনের path থাকতে পারে, যাতে extract করলে লক্ষ্য ফোল্ডারের **বাইরে** ফাইল লেখা হয়। একে বলে **"Zip Slip"**।

- GNU tar ডিফল্টে শুরুর `/` আর `..` অংশ সরিয়ে দেয় (`tar: Removing leading '../' from member names`)। **`-P` বা `--absolute-names` দিলে এই সুরক্ষা বন্ধ হয়ে যায়**, তাই অবিশ্বস্ত আর্কাইভে কখনো দিয়ো না।
- symlink দিয়েও কৌশল করা যায় (আগে একটা symlink, পরে সেই পথে ফাইল)। অবিশ্বস্ত আর্কাইভ **কখনো `sudo` দিয়ে খুলো না**, আর আলাদা খালি ফোল্ডারে খোলো।
- `unzip`-এর আধুনিক সংস্করণ `../` ধরনের path আটকায় ও সতর্কবার্তা দেয়।

### Zip bomb / decompression bomb

কয়েক KB-এর একটা ফাইল ডিকম্প্রেস করলে কয়েকশো GB হয়ে ডিস্ক ভরিয়ে দিতে পারে। অজানা ফাইল খোলার আগে আকার দেখে নাও (`gzip -l`, `xz -l`, `unzip -l`, `zstd -l`)।

### সাধারণ ভুল ও সমাধান

| ভুল | কী হয় | সঠিক |
|---|---|---|
| `tar -cvf dir/` | `dir/`-কে আর্কাইভের নাম ধরে, "Cowardly refusing to create an empty archive" | `tar -cvf dir.tar dir/` |
| `tar -cfv x.tar dir` | আর্কাইভের নাম হয় `v` | `tar -cvf x.tar dir` |
| `tar -xzf x.tar.xz` | `-z` বলে দেওয়ায় tar gzip চালায়, আর "not in gzip format" error আসে | `tar -xJf` বা শুধু `tar -xf` (তখন tar নিজেই চিনে নেয়) |
| `gzip folder/` | "is a directory -- ignored" | `tar -czf folder.tar.gz folder/` |
| `gzip file` চালিয়ে মূল ফাইল খোঁজা | মূল ফাইল মুছে গেছে | `gzip -k file` |
| `zip x.zip folder/` | শুধু খালি ফোল্ডার যায় | `zip -r x.zip folder/` |
| `tar -czf backup.tar.gz .` চালানো ঠিক সেই ফোল্ডারে যেখানে backup.tar.gz তৈরি হচ্ছে | "file changed as we read it" সতর্কবার্তা, আর্কাইভ নিজের ভেতরে নিজেকে নিতে চায় | আর্কাইভ অন্য ফোল্ডারে বানাও, বা `--exclude=backup.tar.gz` |
| `-C` এর ফোল্ডার নেই | "Cannot open: No such file or directory" | আগে `mkdir -p` |

> 📝 **মনে রাখো:** tar-এর exit code: `0` = সফল, `1` = কিছু ফাইল পড়ার সময় বদলে গেছে (যেমন চলমান লগ ফাইল; আর্কাইভ সাধারণত ব্যবহারযোগ্য), `2` = গুরুতর error। ব্যাকআপ স্ক্রিপ্টে ([অধ্যায় ৪১](41-shell-scripting-1.md)) `$?` চেক করার সময় এটা কাজে লাগে।

### ব্যাকআপ যাচাই করার অভ্যাস

যে ব্যাকআপ খুলে দেখা হয়নি, তা ব্যাকআপ নয়, শুধু আশা।

```
$ gzip -t backup.tar.gz && tar -tzf backup.tar.gz > /dev/null && echo "OK"
OK
$ sha256sum backup.tar.gz > backup.tar.gz.sha256
$ sha256sum -c backup.tar.gz.sha256
backup.tar.gz: OK
```

---

## ৩৪.১৫ GUI ও অন্যান্য টুকিটাকি

- **GUI:** GNOME-এ Files (Nautilus) অ্যাপে ফাইলে right-click করে "Compress…" ও "Extract Here" পাওয়া যায় (ভেতরে File Roller/Archive Manager বা GNOME-এর নিজস্ব autoar লাইব্রেরি)। KDE-তে **Ark**। ([অধ্যায় ৪](04-desktop-environment.md))
- **`cpio`:** আরেকটা পুরোনো আর্কাইভার, যেটা stdin থেকে ফাইলের নামের তালিকা নেয়। initramfs ([অধ্যায় ২৭](27-boot-process.md)) ও RPM-এর ভেতরে এটা ব্যবহার হয়।

  ```
  $ find project | cpio -ov > project.cpio
  $ cpio -idv < project.cpio
  ```

- **`ar`:** `.deb` প্যাকেজ ও স্ট্যাটিক লাইব্রেরি (`.a`) এর ফরম্যাট।

  ```
  $ ar t some-package.deb
  debian-binary
  control.tar.zst
  data.tar.zst
  ```

  (পুরোনো বা Debian-এর প্যাকেজে `control.tar.xz`, `data.tar.xz` দেখতে পারো।) প্যাকেজ নিয়ে বিস্তারিত [অধ্যায় ৩০](30-package-management.md)-এ।

- **ফাইলসিস্টেম-স্তরের কম্প্রেশন:** Btrfs-এ `compress=zstd` মাউন্ট অপশন দিলে সব ফাইল স্বচ্ছভাবে কম্প্রেস হয়, তোমাকে কিছুই করতে হয় না ([অধ্যায় ৩২](32-disk-partition-mount.md))।

---

## কীবোর্ড শর্টকাট ও দ্রুত কৌশল

আর্কাইভ টুলগুলোর নিজস্ব কীবোর্ড শর্টকাট কম, তবে কাজের সময় এগুলো লাগবে:

| শর্টকাট / কৌশল | কাজ |
|---|---|
| `Ctrl + C` | চলমান tar/gzip/xz বন্ধ করো (অর্ধেক তৈরি আর্কাইভ মুছে দিয়ো, ওটা অসম্পূর্ণ) |
| `Ctrl + Z`, তারপর `bg` | লম্বা কম্প্রেশন ব্যাকগ্রাউন্ডে পাঠাও ([অধ্যায় ২৫](25-jobs-tmux.md)) |
| `Tab` | ফাইলনাম সম্পূর্ণ করো; bash-completion থাকলে `tar -xf <Tab>` শুধু আর্কাইভ ফাইলই দেখায় |
| `Alt + .` | আগের কমান্ডের শেষ argument বসাও: `tar -tzf x.tar.gz` চালিয়ে তারপর `tar -xzf ` লিখে `Alt+.` ([অধ্যায় ৬](06-terminal-shortcuts.md)) |
| `!$` | আগের কমান্ডের শেষ argument (একই কাজ, history expansion) |
| `^-t^-x` | আগের কমান্ডের প্রথম `-t`-কে `-x` দিয়ে বদলে চালাও: `tar -tzf a.tar.gz` → `tar -xzf a.tar.gz`। (শুধু `^t^x` লিখলে প্রথম মিল `tar`-এর `t` বদলে `xar` হয়ে যাবে!) |
| `zless`/`xzless`-এর ভেতরে `/শব্দ`, `n`, `q` | খোঁজা, পরের মিল, বের হওয়া (`less`-এর মতোই, [অধ্যায় ১০](10-file-dekha.md)) |
| unzip প্রম্পটে `y` / `n` / `A` / `N` / `r` | হ্যাঁ / না / সবগুলোতে হ্যাঁ / সবগুলোতে না / নতুন নাম |
| `7z`-এর প্রম্পটে `y` / `n` / `a` / `s` / `u` / `q` | হ্যাঁ / না / সব হ্যাঁ / সব এড়াও / অটো-রিনেম / বের হও |

---

## হাতে-কলমে অনুশীলন

### ল্যাব ১: চার কম্প্রেসরের প্রতিযোগিতা

```
$ mkdir -p ~/lab34/race && cd ~/lab34/race
$ cp -r /usr/share/doc ./docs 2>/dev/null     # অনেক টেক্সট ফাইল
$ du -sh docs
$ tar -cf docs.tar docs
$ ls -lh docs.tar

$ time gzip  -k docs.tar
$ time bzip2 -k docs.tar
$ time xz    -k -T0 docs.tar
$ time zstd  -T0 docs.tar
$ time zstd  -19 -T0 docs.tar -o docs.tar.19.zst

$ ls -lhS docs.tar*
```

প্রশ্নের উত্তর খাতায় লেখো: কোনটা সবচেয়ে ছোট? কোনটা সবচেয়ে দ্রুত? এবার ডিকম্প্রেশনের সময় মাপো:

```
$ time zcat  docs.tar.gz  > /dev/null
$ time bzcat docs.tar.bz2 > /dev/null
$ time xzcat docs.tar.xz  > /dev/null
$ time zstdcat docs.tar.zst > /dev/null
```

`real` সময়ের তুলনা করে ৩৪.৮-এর টেবিলের সাথে মেলাও।

### ল্যাব ২: tar-এর মোড ও `-C`

```
$ cd ~/lab34
$ tar -czf /tmp/p.tar.gz -C ~/lab34 project      # শুধু "project/" path নিয়ে
$ tar -tzf /tmp/p.tar.gz                          # যাচাই: path "project/..." দিয়ে শুরু?
$ mkdir -p /tmp/r1 /tmp/r2
$ tar -xzf /tmp/p.tar.gz -C /tmp/r1               # পুরোটা
$ tar -xzf /tmp/p.tar.gz -C /tmp/r2 --strip-components=1   # এক স্তর বাদ
$ ls /tmp/r1 /tmp/r2
$ tar -xzOf /tmp/p.tar.gz project/src/main.c      # স্ক্রিনে দেখাও
```

### ল্যাব ৩: বাদ দেওয়া ও তালিকা থেকে নেওয়া

```
$ cd ~/lab34
$ mkdir -p project/node_modules/lib && touch project/node_modules/lib/x.js project/tmp.tmp
$ tar -czf clean.tar.gz --exclude='node_modules' --exclude='*.tmp' project/
$ tar -tzf clean.tar.gz | grep -E 'node_modules|\.tmp' || echo "বাদ পড়েছে ✓"
$ find project -name '*.c' -print0 | tar --null -czf csrc.tar.gz -T -
$ tar -tzf csrc.tar.gz
```

### ল্যাব ৪: gzip-পরিবার দিয়ে লগ বিশ্লেষণ

```
$ cd ~/lab34
$ for i in 1 2 3; do seq 1 5000 | sed "s/^/day$i line /" > log$i.txt; done
$ echo "day2 ERROR disk full" >> log2.txt
$ gzip log1.txt log2.txt log3.txt
$ ls
$ zgrep -l ERROR log*.gz          # কোন ফাইলে ERROR?
$ zcat log*.gz | wc -l            # মোট কত লাইন?
$ gzip -l log2.txt.gz
$ gunzip log2.txt.gz && ls
```

### ল্যাব ৫: zip ও Windows-বন্ধু আর্কাইভ

```
$ zip -r share.zip project -x '*/node_modules/*'
$ unzip -l share.zip
$ mkdir /tmp/z && unzip -q share.zip -d /tmp/z && ls /tmp/z/project
$ zip -d share.zip 'project/tmp.tmp'    # একটা ফাইল মুছে দাও
$ unzip -t share.zip
```

### ল্যাব ৬: পাইপ দিয়ে "রিমোট" কপি (নিজের মেশিনেই)

রিমোট সার্ভার না থাকলে `localhost`-এ SSH দিয়ে অনুশীলন করো (`openssh-server` চালু থাকতে হবে, [অধ্যায় ৩৮](38-ssh-remote.md)), অথবা ssh ছাড়াই পাইপটা বোঝো:

```
# ssh ছাড়া: এক ফোল্ডার থেকে আরেক ফোল্ডারে স্রোত
$ mkdir -p /tmp/dest
$ tar -czf - -C ~/lab34 project | tar -xzf - -C /tmp/dest
$ diff -r ~/lab34/project /tmp/dest/project && echo "হুবহু এক ✓"

# ssh দিয়ে (যদি localhost-এ ssh চালু থাকে)
$ tar -czf - -C ~/lab34 project | ssh localhost 'mkdir -p /tmp/sshdest && tar -xzf - -C /tmp/sshdest'
```

### ল্যাব ৭: `file` দিয়ে ছদ্মবেশী ফাইল ধরা

```
$ cp ~/lab34/race/docs.tar.xz ~/lab34/photo.jpg
$ file ~/lab34/photo.jpg
/home/rahim/lab34/photo.jpg: XZ compressed data, checksum CRC64
$ tar -tf ~/lab34/photo.jpg | head -3     # নাম .jpg হলেও tar ঠিকই খোলে
```

### ল্যাব ৮: ভাগ করা ব্যাকআপ ও যাচাই

```
$ cd ~/lab34/race
$ tar -cf - docs | zstd -T0 | split -b 5M -d - docs.tzst.part
$ ls -lh docs.tzst.part*
$ cat docs.tzst.part* | zstd -d | tar -tf - | wc -l
$ cat docs.tzst.part* | sha256sum
```

পরিষ্কার করতে: `rm -rf ~/lab34 /tmp/r1 /tmp/r2 /tmp/dest /tmp/z /tmp/p.tar.gz`

---

## সারসংক্ষেপ

| কাজ | কমান্ড |
|---|---|
| tar আর্কাইভ বানাও | `tar -cvf a.tar dir/` |
| tar.gz বানাও | `tar -czvf a.tar.gz dir/` |
| tar.bz2 / tar.xz / tar.zst | `tar -cjf` / `tar -cJf` / `tar --zstd -cf` |
| এক্সটেনশন দেখে স্বয়ংক্রিয় | `tar -caf a.tar.xz dir/` |
| নিজের কম্প্রেসর | `tar -I 'zstd -19 -T0' -cf a.tar.zst dir/` |
| তালিকা দেখাও | `tar -tvf a.tar.gz` |
| খোলো | `tar -xvf a.tar.gz` |
| নির্দিষ্ট ফোল্ডারে খোলো | `tar -xf a.tar.gz -C /path` |
| এক স্তর path বাদ | `tar -xf a.tar.gz --strip-components=1` |
| বাদ দিয়ে বানাও | `tar -czf a.tar.gz --exclude='*.log' dir/` |
| path ছোট রাখো | `tar -czf a.tar.gz -C /parent dir` |
| gzip / খোলা | `gzip -k f` / `gunzip f.gz` (বা `gzip -d`) |
| খোলা ছাড়া পড়া / খোঁজা | `zcat`, `zless`, `zgrep` (bz/xz/zstd-এরও আছে) |
| bzip2 | `bzip2 -k f`, `bunzip2`, `bzcat` |
| xz | `xz -k -T0 f`, `unxz`, `xzcat`, `xz -l` |
| zstd | `zstd f` (মূল থাকে), `zstd -19 -T0`, `unzstd`, `zstdcat` |
| তথ্য দেখো | `gzip -l`, `xz -l`, `zstd -l` |
| যাচাই | `gzip -t`, `bzip2 -t`, `xz -t`, `zstd -t`, `unzip -t`, `7z t` |
| zip | `zip -r a.zip dir/`, `unzip -l a.zip`, `unzip a.zip -d dir` |
| 7z | `7z a a.7z dir/`, `7z x a.7z -o/out`, `7z a -p -mhe=on s.7z dir/` |
| rar খোলা | `unrar x a.rar`, বা `unar a.rar` |
| আসল ফরম্যাট | `file f`, `file -z f` |
| রিমোটে পাঠাও | `tar -czf - dir \| ssh host 'tar -xzf - -C /dst'` |
| রিমোট থেকে আনো | `ssh host 'tar -czf - -C /src dir' \| tar -xzf -` |
| ভাগ করো / জোড়া দাও | `... \| split -b 1G -d - a.part` / `cat a.part* \| tar -xzf -` |
| অগ্রগতি | `tar -cf - dir \| pv \| gzip > a.tar.gz` |
| multi-core | `pigz`, `xz -T0`, `zstd -T0`, `pbzip2` |

**tar মনে রাখার সূত্র:** `c` = বানাও, `x` = খোলো, `t` = তালিকা; `z` = gzip, `j` = bzip2, `J` = xz; `v` = বলতে বলতে; `f` = ফাইল (সবসময় শেষে, তার পরেই নাম)।

---

## নিজেকে যাচাই করো

1. Archive আর compression-এর মধ্যে মূল পার্থক্য কী? লিনাক্সে কেন `gzip` দিয়ে সরাসরি একটা ফোল্ডার কম্প্রেস করা যায় না?
2. `tar -cfv backup.tar /home/rahim` চালালে কী ভুল হবে? সঠিক কমান্ড লেখো।
3. `/var/www/html` ফোল্ডারটাকে এমনভাবে `site.tar.gz` বানাও যাতে আর্কাইভের ভেতরের path `html/...` দিয়ে শুরু হয়, `var/www/html/...` দিয়ে নয়।
4. একটা `.tar.gz` আর্কাইভে `-r` দিয়ে নতুন ফাইল যোগ করা যায় না কেন?
5. `gzip report.txt` চালানোর পর `report.txt` আর পাওয়া যাচ্ছে না। কেন? মূল ফাইল রেখে কম্প্রেস করার দুটো উপায় বলো।
6. প্রতিদিন রাতে ৫০ GB ডাটাবেস ডাম্প কম্প্রেস করতে হবে, দ্রুত শেষ হওয়া দরকার, আর মাঝে মাঝে দ্রুত খুলতেও হবে। কোন টুল ও অপশন নেবে, এবং কেন?
7. একটা ফাইলের নাম `data.zip`, কিন্তু `unzip` বলছে এটা zip নয়। আসল ফরম্যাট কীভাবে বের করবে?
8. Windows ব্যবহারকারী বন্ধুকে একটা ফোল্ডার পাঠাবে, আর লিনাক্স সার্ভারের `/etc` ফোল্ডারের ব্যাকআপ নেবে। দুটোর জন্য কোন ফরম্যাট বেছে নেবে, এবং কেন?
9. লোকাল `~/photos` ফোল্ডারটা কোনো মধ্যবর্তী ফাইল না বানিয়ে `user@nas` সার্ভারের `/backup` ফোল্ডারে কপি করার এক লাইনের কমান্ড লেখো। ছবি বলে কম্প্রেস করা উচিত কি?
10. ইন্টারনেট থেকে ডাউনলোড করা অজানা `tool.tar.gz` নিরাপদে খোলার আগে কোন কোন ধাপ নেবে?

<details>
<summary>উত্তর দেখো</summary>

1. Archive অনেক ফাইল ও তাদের metadata (permission, owner, সময়, symlink) এক ফাইলে বাঁধে, আকার প্রায় একই থাকে। Compression একটা byte-স্রোতের আকার ছোট করে। `gzip` শুধু একটা ফাইলের স্রোত বোঝে, ফোল্ডারের কাঠামো বোঝে না। Unix দর্শন অনুযায়ী এ কাজ `tar`-এর। তাই আগে `tar` দিয়ে বাঁধো, তারপর কম্প্রেস করো (`tar -czf`)।

2. `-f`-এর ঠিক পরে `v` আছে, তাই tar আর্কাইভের নাম ধরবে `v`। `backup.tar` আর `/home/rahim` দুটোকেই আর্কাইভ করার জিনিস ভাববে (`backup.tar` না থাকলে error দেবে)। সঠিক: `tar -cvf backup.tar /home/rahim`।

3. `tar -czf site.tar.gz -C /var/www html`

4. `.tar.gz` পুরো আর্কাইভকে একটা gzip স্রোত হিসেবে কম্প্রেস করে। শেষে কিছু যোগ করতে হলে কম্প্রেস করা স্রোতটা খুলে নতুন করে কম্প্রেস করতে হবে, যা `-r` করে না। তাই GNU tar "Cannot update compressed archives" দেখায়। সমাধান: `gunzip`, তারপর `tar -rf`, তারপর আবার `gzip`।

5. `gzip`-এর ডিফল্ট আচরণ হলো মূল ফাইল মুছে তার জায়গায় `report.txt.gz` রাখা। মূল রাখার উপায়: `gzip -k report.txt`, অথবা `gzip -c report.txt > report.txt.gz`। (`gunzip report.txt.gz` দিলে মূল ফাইল ফিরেও আসবে।)

6. `zstd -T0` (ডিফল্ট লেভেল ৩, বা ১–৩)। এটা multi-core, gzip-এর চেয়ে অনেক দ্রুত এবং প্রায়ই বেশি ছোট করে, আর ডিকম্প্রেশন খুবই দ্রুত। উদাহরণ: `mysqldump ... | zstd -T0 > db.sql.zst`। পুরোনো সিস্টেমে zstd না থাকলে `pigz` বিকল্প। `xz -9`/`bzip2` এখানে খুব ধীর হবে।

7. `file data.zip` চালাও। এটা নাম না দেখে ফাইলের শুরুর magic bytes দেখে আসল ফরম্যাট বলে দেবে (যেমন "gzip compressed data" বা "XZ compressed data")। দরকার হলে `head -c 8 data.zip | xxd` দিয়ে নিজেও দেখা যায়। তারপর সেই ফরম্যাটের সঠিক টুল দিয়ে খোলো।

8. Windows বন্ধুর জন্য `.zip` (Windows-এ বিল্ট-ইন খোলে), গোপনীয় হলে `.7z` AES দিয়ে। `/etc`-এর ব্যাকআপের জন্য `sudo tar -czpf` (বা `--zstd`)। কারণ tar owner, group, permission, symlink হুবহু রাখে, যা zip/7z পুরোপুরি রাখে না, আর সিস্টেম ফাইলে এগুলো অপরিহার্য।

9. `tar -cf - -C ~ photos | ssh user@nas 'tar -xf - -C /backup'`। JPEG ছবি আগে থেকেই কম্প্রেস করা, তাই `-z` দিলে লাভ প্রায় নেই, শুধু CPU খরচ হবে। দ্রুত নেটওয়ার্কে কম্প্রেশন ছাড়াই ভালো (`/backup` ফোল্ডার আগে থেকে থাকতে হবে, না থাকলে `mkdir -p /backup &&` আগে লাগাও)।

10. (ক) `file tool.tar.gz` দিয়ে ফরম্যাট নিশ্চিত করো; (খ) প্রকাশক checksum দিলে `sha256sum` দিয়ে মেলাও; (গ) `tar -tvzf tool.tar.gz | less` দিয়ে ভেতরে দেখো: tarbomb কিনা (সব এক ফোল্ডারে আছে কিনা), সন্দেহজনক `../` বা `/` path আছে কিনা, অস্বাভাবিক বড় আকার আছে কিনা; (ঘ) একটা নতুন খালি ফোল্ডারে সাধারণ ইউজার হিসেবে খোলো: `mkdir t && tar -xzf tool.tar.gz -C t`; (ঙ) **কখনো `sudo` দিয়ে বা `-P` দিয়ে খুলবে না।**

</details>

---

[← আগের অধ্যায়](33-lvm-raid.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](35-backup-rsync.md)

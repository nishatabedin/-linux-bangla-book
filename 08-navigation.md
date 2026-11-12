# অধ্যায় ৮: ফাইল সিস্টেমে চলাফেরা

আগের অধ্যায়ে ([অধ্যায় ৭](07-filesystem-hierarchy.md)) তুমি দেখেছ লিনাক্সের ফাইল সিস্টেম একটা উল্টো গাছের মতো সাজানো। গোড়ায় `/` (root), তার নিচে `/home`, `/etc`, `/var` ইত্যাদি। এখন প্রশ্ন হলো, এই গাছের ভেতরে ঘুরব কীভাবে? এক ফোল্ডার থেকে আরেক ফোল্ডারে যাব কীভাবে, আর কোন ফোল্ডারে কী আছে দেখব কীভাবে?

GUI-তে তুমি ফোল্ডারে ডাবল-ক্লিক করো। টার্মিনালে সেই কাজ হয় কয়েকটা ছোট কমান্ড দিয়ে: `pwd`, `cd`, `ls`। এগুলো তুমি দিনে শত শত বার চালাবে। তাই এই অধ্যায়টা ভালোভাবে শিখলে বাকি বই অনেক সহজ লাগবে।

## এই অধ্যায়ে যা শিখবে

- "Current working directory" কী, আর shell কীভাবে মনে রাখে তুমি কোথায় আছ
- `pwd` দিয়ে নিজের অবস্থান জানা (`-L` বনাম `-P`)
- `cd` এর সব রূপ: `cd`, `cd ~`, `cd -`, `cd ..`, `cd /`, `cd ~user`
- Absolute path আর relative path এর পার্থক্য
- `.` আর `..` আসলে কী (এগুলো ফাইল সিস্টেমের সত্যিকারের এন্ট্রি)
- `~` (tilde) expansion কীভাবে কাজ করে
- `ls` এর গুরুত্বপূর্ণ অপশন: `-l -a -A -h -t -r -S -R -d -i -F -1 --color` সহ আরও
- `ls -l` আউটপুটের প্রতিটি কলামের মানে
- `tree` দিয়ে গাছের মতো করে ডিরেক্টরি দেখা
- `pushd`, `popd`, `dirs` দিয়ে ডিরেক্টরি stack ব্যবহার
- Wildcard/globbing (`*`, `?`, `[]`) আর brace expansion (`{}`)
- Tab completion এর কৌশল আর নেভিগেশন সংক্রান্ত সব শর্টকাট

---

## ৮.১ তুমি এখন কোথায়? — Current Working Directory

### ধারণাটা কী

ধরো তুমি একটা বিশাল শপিং মলে আছ। কোনো দোকান খুঁজতে হলে আগে জানতে হয় "আমি এখন কোথায়"। মলের ম্যাপে লাল একটা বিন্দু থাকে, "You are here"। লিনাক্সে এই বিন্দুর নাম **current working directory** (বর্তমান কাজের ডিরেক্টরি), সংক্ষেপে **cwd**।

টার্মিনালে তুমি যে কমান্ডই চালাও, সেটা ধরে নেয় তুমি এই ডিরেক্টরিতে আছ। যেমন `ls` লিখলে সে cwd-র ভেতরের জিনিস দেখায়। `cat notes.txt` লিখলে সে cwd-তেই `notes.txt` খোঁজে।

### ভেতরে কীভাবে কাজ করে (under the hood)

লিনাক্সে প্রতিটা **process** (চলমান প্রোগ্রাম, বিস্তারিত [অধ্যায় ২৪](24-process-management.md)-এ) এর নিজস্ব একটা cwd থাকে। kernel প্রতিটা process এর তথ্যের মধ্যে এটা রেখে দেয়। তোমার shell (যেমন `bash`) নিজেও একটা process, তাই তারও একটা cwd আছে।

```
   ┌──────────────────────── Kernel ────────────────────────┐
   │                                                         │
   │   Process: bash (PID 4321)                              │
   │   ┌───────────────────────────────┐                     │
   │   │ cwd  ──────────► /home/rahim  │                     │
   │   │ root ──────────► /            │                     │
   │   │ open files: 0,1,2 (terminal)  │                     │
   │   └───────────────────────────────┘                     │
   │                                                         │
   │   Process: ls (PID 4400)  ← bash থেকে জন্ম নেয়           │
   │   ┌───────────────────────────────┐                     │
   │   │ cwd  ──────────► /home/rahim  │  (বাবার cwd পায়)      │
   │   └───────────────────────────────┘                     │
   └─────────────────────────────────────────────────────────┘
```

গুরুত্বপূর্ণ কয়েকটা কথা:

1. যখন shell কোনো নতুন প্রোগ্রাম চালায় (যেমন `ls`), নতুন process টা বাবা (shell) এর cwd **উত্তরাধিকার** হিসেবে পায়। এজন্যই `ls` জানে কোন ফোল্ডারের জিনিস দেখাতে হবে।
2. cwd বদলানোর জন্য kernel এর একটা **system call** আছে: `chdir()`। `cd` কমান্ড ভেতরে ভেতরে এই `chdir()` ডাকে।
3. তুমি চাইলে যেকোনো process এর cwd দেখতে পারো `/proc/<PID>/cwd` থেকে (এটা একটা symbolic link, `/proc` নিয়ে বিস্তারিত [অধ্যায় ৪৭](47-kernel-proc-sys.md)-এ):

```
$ ls -l /proc/$$/cwd
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:15 /proc/4321/cwd -> /home/rahim
```

এখানে `$$` হলো বর্তমান shell এর PID।

---

## ৮.২ `pwd` — Print Working Directory

### কী করে

`pwd` তোমার বর্তমান ডিরেক্টরির পুরো ঠিকানা (absolute path) প্রিন্ট করে।

### Syntax

```
pwd [OPTION]
```

### উদাহরণ

```
$ pwd
/home/rahim
```

```
$ cd /var/log
$ pwd
/var/log
```

### অপশন

| অপশন | মানে |
|------|------|
| `-L` | **Logical** path দেখায়। symlink দিয়ে ঢুকলে symlink এর নামসহ path দেখায়। bash-এ এটাই ডিফল্ট। |
| `-P` | **Physical** path দেখায়। সব symlink resolve করে আসল ডিরেক্টরির path দেখায়। |

### `-L` বনাম `-P` উদাহরণ

ধরো `/home/rahim/proj` আসলে `/srv/projects/website` এর একটা symlink (symlink নিয়ে বিস্তারিত [অধ্যায় ১১](11-links-inode.md)-এ)।

```
$ cd ~/proj
$ pwd
/home/rahim/proj
$ pwd -L
/home/rahim/proj
$ pwd -P
/srv/projects/website
```

### ভেতরের কথা: shell builtin বনাম `/usr/bin/pwd`

`pwd` দুই রকম আছে:

```
$ type -a pwd
pwd is a shell builtin
pwd is /usr/bin/pwd
```

- **builtin `pwd`**: bash নিজেই চালায়, নতুন process তৈরি হয় না। bash একটা variable `PWD` তে logical path মনে রাখে, সেখান থেকেই দেখায়।
- **`/usr/bin/pwd`**: GNU coreutils এর আলাদা প্রোগ্রাম। এটা ডিফল্টে `-P` এর মতো আচরণ করে (আসল path দেখায়)।

```
$ echo $PWD
/home/rahim/proj
$ echo $OLDPWD
/home/rahim
```

> 📝 **মনে রাখো:** `PWD` variable এ থাকে বর্তমান ডিরেক্টরি, আর `OLDPWD` এ থাকে ঠিক আগের ডিরেক্টরি। `cd` প্রতিবার এই দুটো আপডেট করে।

> 💡 **টিপস:** বেশিরভাগ distro-তে prompt নিজেই বর্তমান ডিরেক্টরি দেখায়, যেমন `rahim@laptop:~/Documents$`। তাই প্রায়ই `pwd` লাগে না। কিন্তু স্ক্রিপ্টের ভেতরে বা লম্বা path হলে `pwd` খুব কাজের।

---

## ৮.৩ Path — ঠিকানা লেখার দুই নিয়ম

ফাইল বা ডিরেক্টরির ঠিকানাকে বলে **path**। Path লেখার দুটো উপায় আছে।

### Absolute path (পূর্ণ ঠিকানা)

- সবসময় `/` দিয়ে শুরু হয়।
- root থেকে শুরু করে পুরো রাস্তা বলে দেয়।
- তুমি যেখানেই থাকো, একই absolute path সবসময় একই জায়গা বোঝায়।

উপমা: চিঠির পূর্ণ ঠিকানা — "বাংলাদেশ, ঢাকা, মিরপুর, রোড ১০, বাড়ি ৫"। পৃথিবীর যেকোনো জায়গা থেকে পাঠালে চিঠি একই বাড়িতে পৌঁছাবে।

```
/home/rahim/Documents/report.txt
/etc/ssh/sshd_config
/var/log/syslog
```

### Relative path (আপেক্ষিক ঠিকানা)

- `/` দিয়ে শুরু **হয় না**।
- বর্তমান ডিরেক্টরি (cwd) থেকে হিসাব করা হয়।
- তুমি কোথায় আছ তার ওপর নির্ভর করে একই relative path ভিন্ন জায়গা বোঝাতে পারে।

উপমা: কাউকে বলা "এখান থেকে ডানে দুই বাড়ি পরে"। কোথা থেকে বলছ তার ওপর পুরো অর্থ নির্ভর করে।

```
Documents/report.txt
../shared/data.csv
./script.sh
```

### একটা গাছ দিয়ে বুঝি

```
/
├── etc/
│   └── hosts
└── home/
    ├── rahim/          ◄── তুমি এখানে (cwd)
    │   ├── Documents/
    │   │   └── report.txt
    │   └── Music/
    └── karim/
        └── notes.txt
```

তুমি `/home/rahim` এ আছ। একই ফাইলগুলো দুইভাবে লেখা:

| লক্ষ্য | Absolute path | Relative path (cwd = `/home/rahim`) |
|--------|---------------|------------------------------------|
| report.txt | `/home/rahim/Documents/report.txt` | `Documents/report.txt` |
| Music ফোল্ডার | `/home/rahim/Music` | `Music` বা `./Music` |
| karim এর notes | `/home/karim/notes.txt` | `../karim/notes.txt` |
| hosts ফাইল | `/etc/hosts` | `../../etc/hosts` |

### ভেতরে kernel কীভাবে path খোঁজে (path resolution)

kernel যখন একটা path পায়, সে সেটাকে `/` চিহ্ন দিয়ে টুকরো টুকরো করে এক এক ধাপ এগোয়:

```
Path: /home/rahim/Documents/report.txt

শুরু: '/' দিয়ে শুরু? হ্যাঁ ──► শুরু করো root ডিরেক্টরি থেকে
  ধাপ ১: root এর ভেতরে "home" খোঁজো      ──► পাওয়া গেল (ডিরেক্টরি)
  ধাপ ২: home এর ভেতরে "rahim" খোঁজো      ──► পাওয়া গেল
  ধাপ ৩: rahim এর ভেতরে "Documents" খোঁজো ──► পাওয়া গেল
  ধাপ ৪: Documents এর ভেতরে "report.txt"  ──► পাওয়া গেল (ফাইল) ✔

Path: Documents/report.txt
শুরু: '/' দিয়ে শুরু? না ──► শুরু করো process এর cwd থেকে
  বাকিটা একই রকম…
```

প্রতিটা ধাপে kernel পরীক্ষা করে ওই ডিরেক্টরিতে ঢোকার (execute, `x`) permission তোমার আছে কিনা। না থাকলে `Permission denied` আসে। Permission নিয়ে বিস্তারিত [অধ্যায় ১৬](16-permissions.md)-এ।

> 💡 **টিপস:** স্ক্রিপ্ট আর cron job-এ সবসময় absolute path ব্যবহার করো। কারণ ওগুলো কোন cwd থেকে চলবে তার নিশ্চয়তা নেই। ইন্টারঅ্যাক্টিভ কাজে relative path টাইপ করা দ্রুত।

---

## ৮.৪ `.` আর `..` — দুটি বিশেষ ডিরেক্টরি

প্রতিটা ডিরেক্টরির ভেতরে দুটো লুকানো এন্ট্রি সবসময় থাকে:

| নাম | মানে | উদাহরণ |
|-----|------|--------|
| `.` | এই ডিরেক্টরি নিজেই | `./script.sh` = এই ফোল্ডারের script.sh |
| `..` | এর ঠিক উপরের (parent) ডিরেক্টরি | `cd ..` = এক ধাপ উপরে যাও |

```
$ ls -a
.  ..  Documents  Music  .bashrc
```

### এগুলো কি shell এর কারসাজি? না!

অনেকে ভাবে `.` আর `..` shell বানায়। আসলে না। এগুলো ফাইল সিস্টেমে ডিরেক্টরির ভেতরের সত্যিকারের এন্ট্রি (ext4 এর মতো ফাইলসিস্টেমে)। `.` নিজের inode কে নির্দেশ করে, আর `..` parent এর inode কে। inode নম্বর মিলিয়ে দেখো:

```
$ cd /home/rahim
$ ls -id . .. /home /home/rahim
1048578 .   1048577 ..   1048577 /home   1048578 /home/rahim
```

দেখো, `.` আর `/home/rahim` এর inode একই, আর `..` আর `/home` এর inode একই। (inode নিয়ে বিস্তারিত [অধ্যায় ১১](11-links-inode.md)-এ।)

> 📝 **মনে রাখো:** root `/` এর `..` হলো `/` নিজেই। তাই `/` এ থেকে `cd ..` দিলে তুমি `/` তেই থাকবে। গাছের গোড়ার উপরে আর কিছু নেই।

### কেন `./script.sh` লিখতে হয়?

তুমি যখন শুধু `script.sh` লেখো, shell সেটাকে কমান্ড ভেবে `PATH` variable এর ডিরেক্টরিগুলোতে (যেমন `/usr/bin`) খোঁজে, cwd-তে না। নিরাপত্তার কারণে cwd ডিফল্টে `PATH` এ থাকে না। তাই নিজের ফোল্ডারের স্ক্রিপ্ট চালাতে বলতে হয় "এই ফোল্ডারের" — `./script.sh`। (`PATH` নিয়ে বিস্তারিত [অধ্যায় ৪৪](44-environment-bashrc.md)-এ।)

```
$ script.sh
script.sh: command not found
$ ./script.sh
Hello!
```

---

## ৮.৫ `~` (Tilde) Expansion — বাড়ির শর্টকাট

`~` হলো তোমার **home directory** এর সংক্ষিপ্ত রূপ। home directory হলো প্রতিটা ইউজারের নিজস্ব ঘর, যেখানে তার ফাইল, সেটিংস থাকে (সাধারণত `/home/<username>`, আর root ইউজারের জন্য `/root`)।

### কে `~` কে বদলায়?

**Shell** বদলায়, kernel না। তুমি Enter চাপার পর, কমান্ড চালানোর আগে, bash `~` কে আসল path দিয়ে বদলে দেয়। এই প্রক্রিয়ার নাম **tilde expansion**। প্রোগ্রাম কখনো `~` দেখে না, সে পায় পুরো path।

```
$ echo ~
/home/rahim
$ echo ~/Documents
/home/rahim/Documents
```

### Tilde এর সব রূপ

| রূপ | কীসে বদলায় | উদাহরণ output |
|-----|------------|---------------|
| `~` | নিজের home (`$HOME`) | `/home/rahim` |
| `~/dir` | home এর ভেতরের dir | `/home/rahim/dir` |
| `~karim` | ইউজার karim এর home | `/home/karim` |
| `~root` | root ইউজারের home | `/root` |
| `~+` | বর্তমান ডিরেক্টরি (`$PWD`) | `/var/log` |
| `~-` | আগের ডিরেক্টরি (`$OLDPWD`) | `/home/rahim` |

```
$ echo ~root
/root
$ cd /var/log
$ echo ~+ ~-
/var/log /home/rahim
```

### কখন tilde expand হয় না

```
$ echo "~"
~
$ echo '~/Documents'
~/Documents
$ echo a~b
a~b
```

- উদ্ধৃতি চিহ্নের (`"..."` বা `'...'`) ভেতরে `~` expand হয় না।
- শব্দের শুরুতে না থাকলে expand হয় না (তবে `VAR=~/x` এর মতো assignment এ `=` এর পরে হয়)।

> ⚠️ **সতর্কতা:** স্ক্রিপ্টে `"~/backup"` লিখলে সেটা আক্ষরিক `~` নামের ফোল্ডার বোঝাবে! উদ্ধৃতির ভেতরে home বোঝাতে `"$HOME/backup"` লেখো।

---

## ৮.৬ `cd` — Change Directory

### কী করে

`cd` তোমার shell এর বর্তমান ডিরেক্টরি বদলায়।

### Syntax

```
cd [-L|-P] [DIRECTORY]
```

### কেন `cd` একটা shell builtin? (খুব গুরুত্বপূর্ণ ধারণা)

```
$ type cd
cd is a shell builtin
```

`cd` কে **অবশ্যই** shell এর ভেতরেই থাকতে হয়। কেন?

মনে করো `cd` একটা আলাদা প্রোগ্রাম হতো (`/usr/bin/cd`)। তাহলে shell একটা নতুন child process বানাত, সেই child নিজের cwd বদলাত, তারপর শেষ হয়ে যেত। আর বাবা shell এর cwd যেমন ছিল তেমনই থাকত! কারণ একটা process অন্য process এর cwd বদলাতে পারে না।

```
যদি cd আলাদা প্রোগ্রাম হতো (কাজ করত না):

  bash (cwd=/home/rahim)
     │ fork + exec
     ▼
  /usr/bin/cd /tmp  ──► chdir("/tmp") ──► নিজের cwd=/tmp ──► exit
     │
     ▼
  bash (cwd=/home/rahim)   ✘ কিছুই বদলায়নি!

আসলে যা হয় (builtin):

  bash (cwd=/home/rahim) ──► নিজেই chdir("/tmp") ──► bash (cwd=/tmp) ✔
```

উপমা: তুমি তোমার ছোট ভাইকে পাঠালে অন্য ঘরে যেতে। সে গেল, কিন্তু তাতে তুমি নিজে ঘর বদলাওনি। নিজে যেতে হলে নিজেকেই হাঁটতে হবে। `cd` হলো shell এর নিজে হাঁটা।

### `cd` এর সব রূপ

| কমান্ড | কোথায় যায় |
|--------|-----------|
| `cd` | নিজের home এ (`$HOME`) |
| `cd ~` | নিজের home এ |
| `cd ~karim` | karim এর home এ (permission থাকলে) |
| `cd /` | root ডিরেক্টরিতে |
| `cd ..` | এক ধাপ উপরে (parent) |
| `cd ../..` | দুই ধাপ উপরে |
| `cd -` | ঠিক আগের ডিরেক্টরিতে (`$OLDPWD`), আর নতুন path প্রিন্ট করে |
| `cd .` | একই জায়গায় থাকে (কিছু বদলায় না) |
| `cd /etc/ssh` | absolute path এ |
| `cd Documents` | cwd এর ভেতরের Documents এ (relative) |

### উদাহরণ — একটা ছোট ভ্রমণ

```
$ pwd
/home/rahim
$ cd /var/log
$ pwd
/var/log
$ cd ..
$ pwd
/var
$ cd -
/var/log
$ cd -
/var
$ cd
$ pwd
/home/rahim
$ cd /
$ pwd
/
$ cd ~/Documents
$ pwd
/home/rahim/Documents
```

> 💡 **টিপস:** `cd -` হলো টিভির রিমোটের "Previous Channel" বাটনের মতো। দুটো ডিরেক্টরির মধ্যে বারবার আসা-যাওয়ার জন্য অসাধারণ। যেমন `/etc/nginx` আর `/var/log/nginx` এর মধ্যে।

### স্পেস থাকা নাম

ফোল্ডারের নামে স্পেস থাকলে shell সেটাকে দুটো আলাদা শব্দ ভাবে। তাই উদ্ধৃতি বা backslash দিতে হয়:

```
$ cd My Documents
bash: cd: too many arguments
$ cd "My Documents"
$ cd My\ Documents
$ cd 'My Documents'
```

Tab completion ব্যবহার করলে shell নিজেই `\ ` বসিয়ে দেয়।

### `cd -L` বনাম `cd -P` (symlink এর সাথে)

```
$ cd -P ~/proj      # symlink resolve করে আসল ডিরেক্টরিতে যায়
$ pwd
/srv/projects/website
```

ডিফল্ট `-L`। এর একটা মজার দিক আছে: logical মোডে `cd ..` symlink এর **নামের** parent এ যায়, আসল ডিরেক্টরির parent এ না।

```
$ cd ~/proj          # (symlink → /srv/projects/website)
$ cd ..
$ pwd
/home/rahim          # logical parent

$ cd -P ~/proj
$ cd ..
$ pwd
/srv/projects        # physical parent
```

### `CDPATH` — cd এর জন্য খোঁজার তালিকা

`CDPATH` variable এ কিছু ডিরেক্টরি দিলে, relative নাম দিয়ে `cd` করলে bash ওই জায়গাগুলোতেও খুঁজবে।

```
$ export CDPATH=.:~/projects
$ cd website
/home/rahim/projects/website
```

`CDPATH` থেকে কোনো ডিরেক্টরি পাওয়া গেলে bash নতুন path প্রিন্ট করে দেয়। শুরুতে `.` রাখা জরুরি, যাতে cwd আগে খোঁজা হয়।

> ⚠️ **সতর্কতা:** `CDPATH` কে `export` করলে কিছু স্ক্রিপ্ট অপ্রত্যাশিত ডিরেক্টরিতে চলে যেতে পারে। নিরাপদ উপায় হলো `~/.bashrc` তে `export` ছাড়া শুধু `CDPATH=...` লেখা।

### `shopt -s autocd` — শুধু নাম লিখেই ঢোকা

bash এর এই অপশন চালু থাকলে শুধু ডিরেক্টরির নাম লিখলেই `cd` হয়ে যায়:

```
$ shopt -s autocd
$ /etc
cd -- /etc
$ pwd
/etc
```

আর `shopt -s cdspell` চালু করলে `cd` এ ছোটখাটো বানান ভুল (একটা অক্ষর বাদ/বেশি/অদলবদল) bash নিজেই ঠিক করে নেয়।

### সাধারণ error

```
$ cd /root
bash: cd: /root: Permission denied
$ cd notes.txt
bash: cd: notes.txt: Not a directory
$ cd /nonexistent
bash: cd: /nonexistent: No such file or directory
```

---

## ৮.৭ `ls` — List Directory Contents

### কী করে

`ls` একটা ডিরেক্টরির ভেতরে কী কী ফাইল/ফোল্ডার আছে তার তালিকা দেখায়। কোনো argument না দিলে cwd দেখায়।

### Syntax

```
ls [OPTION]... [FILE]...
```

### ভেতরে কীভাবে কাজ করে

ডিরেক্টরি আসলে একটা বিশেষ ফাইল, যার ভেতরে লেখা থাকে "নাম → inode নম্বর" এর একটা তালিকা। `ls`:

1. `opendir()`/`getdents64()` system call দিয়ে নামগুলো পড়ে।
2. `-l` এর মতো বিস্তারিত চাইলে প্রতিটা ফাইলের জন্য `stat()` (আধুনিক glibc তে `statx()` বা `lstat()`) ডেকে inode থেকে size, owner, permission, সময় ইত্যাদি আনে।
3. ডিফল্টে নামের বর্ণানুক্রমে sort করে (locale অনুযায়ী) তারপর প্রিন্ট করে।

```
   ডিরেক্টরি /home/rahim (একটা বিশেষ ফাইল)
   ┌──────────────┬──────────┐
   │ নাম          │ inode    │        inode টেবিল
   ├──────────────┼──────────┤       ┌─────────────────────────┐
   │ .            │ 1048578  │       │ 1311001: type=file      │
   │ ..           │ 1048577  │       │   perm=rw-r--r--        │
   │ notes.txt    │ 1311001 ─┼──────►│   owner=rahim size=220  │
   │ Documents    │ 1311050  │       │   mtime=Nov 12 09:30 …  │
   └──────────────┴──────────┘       └─────────────────────────┘
        ▲                                   ▲
   ls শুধু নাম চাইলে এখানেই শেষ         ls -l চাইলে stat() দিয়ে এখানে আসে
```

এজন্যই হাজার হাজার ফাইলের ফোল্ডারে `ls` দ্রুত, কিন্তু `ls -l` কিছুটা ধীর (প্রতিটা ফাইলে আলাদা stat লাগে)।

### সাধারণ ব্যবহার

```
$ ls
Desktop  Documents  Downloads  Music  Pictures  notes.txt
$ ls /etc/ssh
moduli  ssh_config  ssh_config.d  sshd_config  sshd_config.d  ssh_host_ed25519_key ...
$ ls Documents Music
Documents:
report.txt  resume.pdf

Music:
song.mp3
```

### গুরুত্বপূর্ণ অপশনের টেবিল

| অপশন | পুরো নাম | কী করে |
|------|---------|--------|
| `-l` | long format | বিস্তারিত তথ্য: permission, link, owner, group, size, সময়, নাম |
| `-a` | `--all` | লুকানো ফাইলসহ (`.` দিয়ে শুরু) সব দেখায়, `.` আর `..` সহ |
| `-A` | `--almost-all` | লুকানো ফাইল দেখায়, কিন্তু `.` আর `..` বাদে |
| `-h` | `--human-readable` | `-l` এর সাথে size কে K, M, G আকারে দেখায় |
| `-t` | | শেষ পরিবর্তনের সময় (mtime) অনুযায়ী sort, নতুনটা আগে |
| `-r` | `--reverse` | sort এর ক্রম উল্টে দেয় |
| `-S` | | size অনুযায়ী sort, বড়টা আগে |
| `-R` | `--recursive` | ভেতরের সব সাব-ডিরেক্টরিও দেখায় |
| `-d` | `--directory` | ডিরেক্টরির ভেতরের জিনিস না, ডিরেক্টরিটাকেই দেখায় |
| `-i` | `--inode` | প্রতিটা ফাইলের inode নম্বর দেখায় |
| `-F` | `--classify` | নামের শেষে টাইপ চিহ্ন: `/` ডিরেক্টরি, `*` executable, `@` symlink, `|` FIFO, `=` socket |
| `-1` | | প্রতি লাইনে একটা করে নাম |
| `-n` | `--numeric-uid-gid` | `-l` এর মতো, কিন্তু owner/group নামের বদলে UID/GID সংখ্যা |
| `-X` | | extension অনুযায়ী sort |
| `-v` | | স্বাভাবিক সংখ্যা ক্রমে sort (file2 আগে, file10 পরে) |
| `-u` | | `-lt` এর সাথে: access time (atime) অনুযায়ী |
| `-c` | | `-lt` এর সাথে: inode change time (ctime) অনুযায়ী |
| `-s` | `--size` | প্রতিটা ফাইলের ডিস্কে বরাদ্দকৃত block size |
| `-m` | | কমা দিয়ে আলাদা করা তালিকা |
| `--color=auto` | | টার্মিনাল হলে রঙ দিয়ে টাইপ বোঝায় (`always`/`never` ও আছে) |
| `--group-directories-first` | | ডিরেক্টরিগুলো আগে, তারপর ফাইল |
| `--time-style=STYLE` | | সময়ের ফরম্যাট: `full-iso`, `long-iso`, `iso`, `locale`, বা `+FORMAT` |
| `--full-time` | | `-l --time-style=full-iso` এর সমান |
| `-Z` | `--context` | SELinux security context দেখায় (Fedora/RHEL এ কাজের) |

একাধিক ছোট অপশন একসাথে লেখা যায়: `ls -l -a -h` = `ls -lah`।

### প্রতিটা অপশনের উদাহরণ

**`-a` আর `-A` — লুকানো ফাইল**

লিনাক্সে যে ফাইলের নাম `.` দিয়ে শুরু, সেটা **hidden** (লুকানো)। এটা কোনো বিশেষ attribute না, শুধু একটা নিয়ম: `ls` আর ফাইল ম্যানেজার এগুলো ডিফল্টে দেখায় না। সাধারণত কনফিগারেশন ফাইলগুলো এভাবে রাখা হয়, যাতে home ফোল্ডার পরিষ্কার দেখায়।

```
$ ls
Documents  Music  notes.txt
$ ls -a
.  ..  .bash_history  .bashrc  .cache  .config  .profile  Documents  Music  notes.txt
$ ls -A
.bash_history  .bashrc  .cache  .config  .profile  Documents  Music  notes.txt
```

**`-l` আর `-h` — বিস্তারিত ও পড়ার মতো size**

```
$ ls -l
total 2056
drwxr-xr-x 2 rahim rahim    4096 Nov 10 14:22 Documents
-rw-r--r-- 1 rahim rahim 2097152 Nov 12 09:30 backup.tar
-rw-r--r-- 1 rahim rahim     220 Nov  1 08:00 notes.txt
$ ls -lh
total 2.1M
drwxr-xr-x 2 rahim rahim 4.0K Nov 10 14:22 Documents
-rw-r--r-- 1 rahim rahim 2.0M Nov 12 09:30 backup.tar
-rw-r--r-- 1 rahim rahim  220 Nov  1 08:00 notes.txt
```

**`-t`, `-r`, `-S` — sort করা**

```
$ ls -lt          # নতুন পরিবর্তিত আগে
$ ls -ltr         # পুরনো আগে, নতুনটা একদম নিচে (লগ দেখতে দারুণ)
$ ls -lS          # সবচেয়ে বড় আগে
$ ls -lSr         # সবচেয়ে ছোট আগে
$ ls -lhS /var/log | head -5
total 48M
-rw-r----- 1 syslog adm 21M Nov 12 10:01 syslog
-rw-r----- 1 syslog adm 15M Nov  9 00:00 syslog.1
-rw-r--r-- 1 root   root 4.1M Nov 12 09:55 dpkg.log
-rw-r----- 1 syslog adm 2.3M Nov 12 10:01 kern.log
```

> 💡 **টিপস:** `ls -ltr` সিস্টেম অ্যাডমিনদের প্রিয় কমান্ড। কোনো ফোল্ডারে "এইমাত্র কোন ফাইল বদলেছে" জানতে চাইলে এটা চালাও, সবচেয়ে নতুনটা স্ক্রিনের একদম নিচে, চোখের সামনে থাকবে।

**`-R` — recursive**

```
$ ls -R project
project:
README.md  src

project/src:
main.c  utils

project/src/utils:
helper.c
```

> ⚠️ **সতর্কতা:** `ls -R /` চালালে পুরো সিস্টেমের লাখ লাখ ফাইল স্ক্রিনে বয়ে যাবে। আটকাতে `Ctrl+C` চাপো।

**`-d` — ডিরেক্টরিকে নিজেকেই দেখা**

```
$ ls -l /etc        # /etc এর ভেতরের শত শত ফাইল দেখাবে
$ ls -ld /etc       # শুধু /etc ডিরেক্টরিটার তথ্য
drwxr-xr-x 139 root root 12288 Nov 12 08:10 /etc
$ ls -d */          # cwd তে শুধু ডিরেক্টরিগুলো
Documents/  Music/  Pictures/
```

**`-i` — inode নম্বর**

```
$ ls -i
1311050 Documents  1311001 notes.txt
```

**`-F` — টাইপ চিহ্ন**

```
$ ls -F
Documents/  backup.sh*  link-to-notes@  notes.txt
```

**`--color` — রঙ**

Ubuntu/Debian-এ `~/.bashrc` তে `alias ls='ls --color=auto'` আগে থেকেই থাকে, Fedora/RHEL-এ `/etc/profile.d/colorls.sh` এটা করে। সাধারণ রঙের মানে (ডিফল্ট থিমে):

| রঙ | সাধারণত মানে |
|----|-------------|
| নীল (bold) | ডিরেক্টরি |
| সবুজ (bold) | executable ফাইল |
| সায়ান (হালকা নীল) | symbolic link |
| লাল (bold) | archive (`.tar`, `.zip`, `.gz`) |
| ম্যাজেন্টা | ছবি/ভিডিও |
| হলুদ, কালো পটভূমি | device ফাইল (`/dev` এ) |
| লাল পটভূমিতে লেখা | ভাঙা symlink (orphan) |

রঙগুলো আসে `LS_COLORS` variable থেকে, যেটা `dircolors` কমান্ড সেট করে। `dircolors -p` দিয়ে পুরো ডিফল্ট সেটিং দেখতে পারো।

> 📝 **মনে রাখো:** `--color=auto` এর মানে হলো output টার্মিনালে গেলে রঙ, আর pipe বা ফাইলে গেলে রঙ ছাড়া। `--color=always` দিয়ে pipe এও রঙ পাঠালে `less -R` দিয়ে দেখতে হয়।

### Ubuntu এর তৈরি alias গুলো

Ubuntu/Debian এর ডিফল্ট `~/.bashrc` তে এগুলো থাকে:

```
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
```

তাই Ubuntu তে `ll` লিখলেই `ls -alF` চলে। Fedora/RHEL এ `ll` হলো `ls -l` (সাথে color)। যাচাই করতে:

```
$ type ll
ll is aliased to `ls -alF'
```

---

## ৮.৮ `ls -l` আউটপুটের প্রতিটা কলাম — বিস্তারিত

এই আউটপুটটা পড়তে পারা লিনাক্সের অন্যতম মৌলিক দক্ষতা। একটা লাইন ভেঙে দেখি:

```
-rw-r--r-- 1 rahim developers 2097152 Nov 12 09:30 backup.tar
│└──┬────┘ │   │       │         │         │          │
│   │      │   │       │         │         │          └─ ⑦ নাম
│   │      │   │       │         │         └─ ⑥ শেষ পরিবর্তনের সময় (mtime)
│   │      │   │       │         └─ ⑤ size (bytes এ)
│   │      │   │       └─ ④ group
│   │      │   └─ ③ owner (মালিক ইউজার)
│   │      └─ ② hard link সংখ্যা
│   └─ ① (খ) permission (৯ অক্ষর)
└─ ① (ক) ফাইলের টাইপ (১ অক্ষর)
```

### ① ফাইলের টাইপ আর permission

প্রথম অক্ষর টাইপ বোঝায়:

| অক্ষর | টাইপ | কোথায় দেখা যায় |
|-------|------|----------------|
| `-` | সাধারণ ফাইল (regular file) | টেক্সট, ছবি, প্রোগ্রাম |
| `d` | ডিরেক্টরি | `/home`, `Documents` |
| `l` | symbolic link | `/bin -> usr/bin` |
| `c` | character device | `/dev/tty`, `/dev/null` |
| `b` | block device | `/dev/sda`, `/dev/nvme0n1` |
| `p` | named pipe (FIFO) | `mkfifo` দিয়ে বানানো |
| `s` | socket | `/run/systemd/...` |

পরের ৯টা অক্ষর তিনটা দলে permission:

```
 r w -   r - -   r - -
 └─┬─┘   └─┬─┘   └─┬─┘
 owner   group   others
```

`r` = পড়া, `w` = লেখা, `x` = চালানো (ডিরেক্টরির জন্য ঢোকা), `-` = অনুমতি নেই। কখনো কখনো `s`, `t` দেখবে (বিশেষ permission, [অধ্যায় ১৭](17-special-permissions-acl.md)), আর শেষে `+` থাকলে ACL আছে, `.` থাকলে SELinux context আছে (Fedora/RHEL এ প্রায় সব ফাইলে)। পুরো permission সিস্টেম [অধ্যায় ১৬](16-permissions.md)-এ।

### ② Hard link সংখ্যা

ফাইলের জন্য: এই inode কে কতগুলো নাম নির্দেশ করছে। সাধারণত `1`।

ডিরেক্টরির জন্য: সাধারণত `2 + সাব-ডিরেক্টরির সংখ্যা`। কেন? ডিরেক্টরির নিজের নাম (parent এ) একটা link, ভেতরের `.` আরেকটা, আর প্রতিটা সাব-ডিরেক্টরির `..` একটা করে। (btrfs এর মতো কিছু ফাইলসিস্টেম এই নিয়ম মানে না, সেখানে ডিরেক্টরিতে সবসময় `1` দেখায়।) বিস্তারিত [অধ্যায় ১১](11-links-inode.md)।

```
$ ls -ld /home
drwxr-xr-x 4 root root 4096 Nov  1 08:00 /home   # ২ + ২টা ইউজার ফোল্ডার
```

### ③ Owner, ④ Group

কোন ইউজার ফাইলটার মালিক আর কোন group এর। kernel আসলে নাম রাখে না, রাখে সংখ্যা (UID, GID)। `ls` সেগুলো `/etc/passwd` আর `/etc/group` থেকে নামে রূপান্তর করে। `ls -n` দিলে কাঁচা সংখ্যা দেখবে:

```
$ ls -ln notes.txt
-rw-r--r-- 1 1000 1000 220 Nov  1 08:00 notes.txt
```

ইউজার মুছে ফেলা হলে নামের জায়গায় শুধু সংখ্যা দেখা যায়। ইউজার/গ্রুপ নিয়ে [অধ্যায় ১৫](15-users-groups.md)।

### ⑤ Size

ফাইলের আকার byte এ। `-h` দিলে K/M/G (1024 ভিত্তিক)। `--si` দিলে 1000 ভিত্তিক।

> 📝 **মনে রাখো:** ডিরেক্টরির size (যেমন `4096`) মানে ভেতরের ফাইলগুলোর মোট আকার **না**। এটা হলো ডিরেক্টরির নিজের "নামের তালিকা" রাখতে কতটা জায়গা লাগছে। ভেতরের মোট আকার জানতে `du -sh dirname` লাগে ([অধ্যায় ৩২](32-disk-partition-mount.md))।

Device ফাইলে size এর জায়গায় দুটো সংখ্যা থাকে: **major, minor** নম্বর (কোন driver আর কোন device):

```
$ ls -l /dev/sda
brw-rw---- 1 root disk 8, 0 Nov 12 08:00 /dev/sda
```

### ⑥ সময়

ডিফল্টে **mtime** (modification time): ফাইলের ভেতরের লেখা শেষ কখন বদলেছে। ছয় মাসের বেশি পুরনো (বা ভবিষ্যতের) হলে ঘণ্টা-মিনিটের বদলে বছর দেখায়:

```
-rw-r--r-- 1 rahim rahim 512 Mar  3  2024 old.txt
-rw-r--r-- 1 rahim rahim 220 Nov  1 08:00 notes.txt
```

তিন ধরনের সময় আছে:

| সময় | মানে | `ls` এ দেখার উপায় |
|------|------|-------------------|
| mtime | কন্টেন্ট বদলানোর সময় | `ls -l` (ডিফল্ট) |
| atime | শেষ পড়ার সময় | `ls -lu` |
| ctime | inode (permission, owner, নাম ইত্যাদি) বদলানোর সময় | `ls -lc` |

সঠিক সেকেন্ড পর্যন্ত দেখতে:

```
$ ls -l --full-time notes.txt
-rw-r--r-- 1 rahim rahim 220 2026-11-01 08:00:13.482193201 +0600 notes.txt
```

### ⑦ নাম

symlink হলে নামের পরে `->` দিয়ে লক্ষ্য দেখায়:

```
$ ls -l /bin
lrwxrwxrwx 1 root root 7 Apr 22  2024 /bin -> usr/bin
```

### প্রথম লাইনের `total` কী?

```
$ ls -l
total 2056
```

এটা তালিকার সব ফাইলের ডিস্কে **বরাদ্দকৃত** block এর যোগফল, GNU `ls` এ ডিফল্টে 1 KiB একক হিসেবে। এটা byte size এর যোগফল না। `-h` দিলে `total 2.1M` এর মতো দেখায়।

---

## ৮.৯ `tree` — গাছের মতো করে দেখা

### কী করে

`tree` একটা ডিরেক্টরির ভেতরের সব কিছু গাছের ডালপালার আকারে দেখায়। কোন ফাইল কোন ফোল্ডারের ভেতরে, এক নজরে বোঝা যায়।

### ইনস্টল

`tree` বেশিরভাগ distro-তে ডিফল্টে থাকে না:

```
$ sudo apt install tree        # Ubuntu / Debian
$ sudo dnf install tree        # Fedora / RHEL 9
```

(প্যাকেজ ইনস্টল নিয়ে বিস্তারিত [অধ্যায় ৩০](30-package-management.md)-এ।)

### উদাহরণ

```
$ tree project
project
├── README.md
├── docs
│   └── guide.md
└── src
    ├── main.c
    └── utils
        └── helper.c

4 directories, 4 files
```

### গুরুত্বপূর্ণ অপশন

| অপশন | কী করে |
|------|--------|
| `-L N` | সর্বোচ্চ N স্তর গভীর পর্যন্ত দেখায় |
| `-d` | শুধু ডিরেক্টরি |
| `-a` | লুকানো ফাইলসহ |
| `-f` | প্রতিটার পুরো path দেখায় |
| `-h` | size মানুষের পড়ার মতো করে |
| `-p` | permission দেখায় |
| `-u`, `-g` | owner, group দেখায় |
| `-D` | শেষ পরিবর্তনের সময় |
| `-I PATTERN` | pattern মিলে যাওয়া জিনিস বাদ দেয় (যেমন `-I 'node_modules|.git'`) |
| `-P PATTERN` | শুধু pattern মিলে যাওয়া ফাইল দেখায় |
| `--dirsfirst` | ডিরেক্টরি আগে |
| `-C` | রঙ জোর করে চালু |
| `-J` / `-X` | JSON / XML আকারে output |

```
$ tree -L 1 /
/
├── bin -> usr/bin
├── boot
├── dev
├── etc
├── home
...
$ tree -d -L 2 ~
$ tree -h --dirsfirst -I 'node_modules|.git' myapp
```

> 💡 **টিপস:** `tree` না থাকলে কাছাকাছি কাজ `find . -maxdepth 2` দিয়ে করা যায় (`find` নিয়ে [অধ্যায় ১২](12-file-khoja.md))।

---

## ৮.১০ `pushd`, `popd`, `dirs` — ডিরেক্টরি স্ট্যাক

### সমস্যাটা কী

`cd -` শুধু একটা আগের জায়গা মনে রাখে। কিন্তু তুমি যদি তিন-চারটা ডিরেক্টরিতে ঘুরে আবার ক্রম অনুযায়ী ফিরে আসতে চাও?

### সমাধান: stack

**Stack** মানে প্লেটের স্তূপের মতো। নতুন প্লেট রাখো একদম উপরে (push), তুলতেও হয় উপর থেকে (pop)। একে বলে LIFO, Last In First Out।

```
pushd /etc         pushd /var/log      popd              popd
┌──────────┐      ┌──────────┐       ┌──────────┐      ┌──────────┐
│ /etc     │ ◄top │ /var/log │ ◄top  │ /etc     │ ◄top │ ~        │ ◄top
│ ~        │      │ /etc     │       │ ~        │      └──────────┘
└──────────┘      │ ~        │       └──────────┘
                  └──────────┘
cwd = /etc        cwd = /var/log     cwd = /etc        cwd = ~
```

Stack এর উপরের জিনিসটা সবসময় বর্তমান ডিরেক্টরি।

### কমান্ডগুলো

| কমান্ড | কী করে |
|--------|--------|
| `pushd DIR` | বর্তমান ডিরেক্টরি stack এ রেখে DIR এ যায় |
| `pushd` | উপরের দুটো এন্ট্রি অদলবদল করে (দুই জায়গায় টগল) |
| `pushd +N` | stack ঘুরিয়ে বাম দিক থেকে N নম্বর এন্ট্রিকে উপরে আনে (০ থেকে গোনা) |
| `pushd -n DIR` | ডিরেক্টরি না বদলে শুধু stack এ যোগ করে |
| `popd` | উপরের এন্ট্রি সরিয়ে নতুন উপরের ডিরেক্টরিতে যায় |
| `popd +N` | N নম্বর এন্ট্রি সরিয়ে দেয় |
| `dirs` | stack দেখায় (এক লাইনে) |
| `dirs -v` | নম্বরসহ প্রতি লাইনে একটা করে দেখায় |
| `dirs -p` | নম্বর ছাড়া প্রতি লাইনে একটা |
| `dirs -l` | `~` এর বদলে পুরো path |
| `dirs -c` | পুরো stack মুছে দেয় |

### উদাহরণ

```
$ pwd
/home/rahim
$ pushd /etc
/etc ~
$ pushd /var/log
/var/log /etc ~
$ pushd /usr/share
/usr/share /var/log /etc ~
$ dirs -v
 0  /usr/share
 1  /var/log
 2  /etc
 3  ~
$ pushd +2
/etc ~ /usr/share /var/log
$ popd
~ /usr/share /var/log
$ pwd
/home/rahim
```

`pushd` আর `popd` প্রতিবার পুরো stack প্রিন্ট করে। চুপ রাখতে চাইলে `pushd /etc > /dev/null`।

`~N` দিয়ে stack এর এন্ট্রি সরাসরি ব্যবহার করা যায় (tilde expansion এর অংশ):

```
$ dirs -v
 0  /usr/share
 1  /var/log
$ cp ~1/syslog .        # ~1 = /var/log
```

> 💡 **টিপস:** স্ক্রিপ্টে কোনো ফোল্ডারে গিয়ে কাজ করে আবার আগের জায়গায় ফিরতে `pushd dir > /dev/null; ...; popd > /dev/null` খুব পরিষ্কার পদ্ধতি। আরেকটা উপায় হলো subshell: `( cd dir && make )`, এতে বাইরের shell এর cwd বদলায়ই না।

---

## ৮.১১ Wildcards / Globbing — অনেক ফাইল একসাথে ধরা

### কী এবং কেন

ধরো একটা ফোল্ডারে ৫০০টা ফাইল, তুমি শুধু `.jpg` গুলো দেখতে চাও। প্রতিটার নাম টাইপ করা অসম্ভব। এখানে কাজে আসে **wildcard**, বিশেষ চিহ্ন যা নামের একটা "ছাঁচ" (pattern) বোঝায়। এই pattern কে ফাইলের নামে মিলিয়ে তালিকা বানানোর প্রক্রিয়ার নাম **globbing** (বা pathname expansion)।

উপমা: তাসের খেলায় জোকার যেকোনো তাসের জায়গা নিতে পারে। `*` হলো সেই জোকার।

### ভেতরে কে কাজ করে? — Shell!

এটা বোঝা খুব জরুরি: globbing করে **shell**, কমান্ড না। `ls *.txt` লিখলে `ls` কখনো `*` দেখে না।

```
তুমি টাইপ করলে:      ls *.txt
                          │
             bash cwd পড়ে নাম মেলায়
                          │
                          ▼
আসলে যা চলে:         ls a.txt b.txt notes.txt
```

প্রমাণ দেখতে `echo` ব্যবহার করো, কারণ `echo` শুধু যা পায় তাই প্রিন্ট করে:

```
$ echo *.txt
a.txt b.txt notes.txt
```

> 💡 **টিপস:** কোনো ঝুঁকিপূর্ণ কমান্ডের আগে (যেমন `rm *.log`) আগে `echo *.log` চালিয়ে দেখে নাও কোন ফাইলগুলো ধরা পড়বে।

### Wildcard এর চিহ্নগুলো

| চিহ্ন | মেলে | উদাহরণ | মিলবে |
|-------|------|--------|------|
| `*` | যেকোনো সংখ্যক অক্ষর (শূন্যসহ) | `*.txt` | `a.txt`, `notes.txt`, `.txt` না (hidden) |
| `?` | ঠিক একটা অক্ষর | `file?.txt` | `file1.txt`, `fileA.txt`; `file10.txt` না |
| `[abc]` | বন্ধনীর যেকোনো একটা অক্ষর | `[abc]*.txt` | `a.txt`, `b1.txt`, `cat.txt` |
| `[a-z]` | পরিসরের একটা অক্ষর | `file[0-9].txt` | `file0.txt` … `file9.txt` |
| `[!abc]` বা `[^abc]` | বন্ধনীর বাইরের একটা অক্ষর | `[!0-9]*` | যা সংখ্যা দিয়ে শুরু হয়নি |
| `[[:digit:]]` | character class | `*[[:digit:]]` | যা সংখ্যা দিয়ে শেষ |

Character class গুলো: `[:alpha:]` (অক্ষর), `[:digit:]` (সংখ্যা), `[:alnum:]` (অক্ষর+সংখ্যা), `[:upper:]`, `[:lower:]`, `[:space:]`, `[:punct:]`। এগুলো অবশ্যই আরেক জোড়া বন্ধনীর ভেতরে: `[[:upper:]]`।

> ⚠️ **সতর্কতা:** `[a-z]` এর মানে locale এর ওপর নির্ভর করতে পারে (কিছু locale এ বড় হাতের অক্ষরও মিলে যেত)। নিশ্চিতভাবে ছোট হাতের অক্ষর চাইলে `[[:lower:]]` ব্যবহার করো।

### উদাহরণ

ধরো ফোল্ডারে আছে:

```
$ ls
data1.csv  data2.csv  data10.csv  photo.jpg  Photo.PNG  report.txt  .hidden
```

```
$ ls *.csv
data10.csv  data1.csv  data2.csv
$ ls data?.csv
data1.csv  data2.csv
$ ls data??.csv
data10.csv
$ ls [pP]hoto.*
photo.jpg  Photo.PNG
$ ls [!d]*
photo.jpg  Photo.PNG  report.txt
$ ls *
data10.csv  data1.csv  data2.csv  photo.jpg  Photo.PNG  report.txt
```

লক্ষ্য করো, `*` লুকানো ফাইল `.hidden` ধরেনি।

### নিয়মগুলো যা মনে রাখতে হবে

1. **লুকানো ফাইল:** `*` আর `?` নামের শুরুর `.` এর সাথে মেলে না। লুকানো ফাইল ধরতে `.*` লিখতে হয়, অথবা `shopt -s dotglob` চালু করতে হয়।
2. **`/` এর সাথে মেলে না:** `*` কখনো `/` পার হয় না। তাই `*.txt` সাব-ফোল্ডারের ভেতরে খোঁজে না। `*/*.txt` লিখলে এক স্তর নিচে খোঁজে।
3. **কিছু না মিললে:** bash ডিফল্টে pattern টাকে অক্ষত রেখে দেয়:

```
$ ls *.pdf
ls: cannot access '*.pdf': No such file or directory
```

এখানে কোনো `.pdf` নেই, তাই bash আক্ষরিক `*.pdf` পাঠিয়েছে, আর `ls` সেই নামের ফাইল খুঁজে পায়নি।

4. **উদ্ধৃতির ভেতরে globbing হয় না:** `ls "*.txt"` আক্ষরিক `*.txt` নামের ফাইল খুঁজবে। এটাই `find . -name "*.txt"` এর মতো কমান্ডে উদ্ধৃতি দেওয়ার কারণ, যাতে shell না, `find` নিজে pattern পায়।
5. **sort:** ফলাফল locale অনুযায়ী বর্ণানুক্রমে সাজানো থাকে।

### `shopt` দিয়ে globbing এর আচরণ বদলানো (bash)

| অপশন | কী করে |
|------|--------|
| `shopt -s dotglob` | `*` লুকানো ফাইলও ধরে (`.` আর `..` ছাড়া) |
| `shopt -s nullglob` | কিছু না মিললে pattern টা খালি হয়ে যায় (স্ক্রিপ্টে লুপের জন্য উপকারী) |
| `shopt -s failglob` | কিছু না মিললে error দেয়, কমান্ড চালায় না |
| `shopt -s nocaseglob` | বড়/ছোট হাতের পার্থক্য উপেক্ষা করে |
| `shopt -s globstar` | `**` দিয়ে সব স্তরের সাব-ডিরেক্টরি ধরা যায় |
| `shopt -s extglob` | বাড়তি pattern চালু করে (নিচে দেখো) |
| `shopt` | সব অপশনের বর্তমান অবস্থা দেখায় |

`-s` = set (চালু), `-u` = unset (বন্ধ)।

**globstar (`**`)**:

```
$ shopt -s globstar
$ ls **/*.c
src/main.c  src/utils/helper.c
```

**extglob** (Ubuntu তে ইন্টারঅ্যাক্টিভ shell এ সাধারণত আগে থেকেই চালু থাকে, bash-completion এর কারণে):

| Pattern | মানে |
|---------|------|
| `?(pat)` | শূন্য বা একবার |
| `*(pat)` | শূন্য বা যতবার খুশি |
| `+(pat)` | এক বা তার বেশি বার |
| `@(pat1\|pat2)` | ঠিক একবার, যেকোনো একটা |
| `!(pat)` | pattern ছাড়া সব কিছু |

```
$ shopt -s extglob
$ ls !(*.csv)
photo.jpg  Photo.PNG  report.txt
$ ls *.@(jpg|PNG)
photo.jpg  Photo.PNG
```

> 💡 **টিপস:** `rm !(important.txt)` দিয়ে একটা ফাইল রেখে বাকি সব মোছা যায়। কিন্তু আগে অবশ্যই `echo !(important.txt)` চালিয়ে দেখে নাও।

> 📝 **মনে রাখো:** Globbing এর pattern আর regular expression ([অধ্যায় ২০](20-grep-regex.md)) এক জিনিস না। glob এ `*` মানে "যেকোনো অক্ষর", regex এ `*` মানে "আগের জিনিসটা শূন্য বা বেশি বার"। glob এর `?` = regex এর `.`।

---

## ৮.১২ Brace Expansion `{}` — নাম তৈরির কারখানা

### Glob থেকে পার্থক্য

Brace expansion **দেখতে** wildcard এর মতো, কিন্তু কাজ সম্পূর্ণ আলাদা:

| | Globbing (`*`, `?`, `[]`) | Brace expansion (`{}`) |
|--|---------------------------|------------------------|
| কাজ | **আছে এমন** ফাইলের নাম খোঁজে | নতুন শব্দ **তৈরি** করে |
| ফাইল থাকতে হবে? | হ্যাঁ | না, ফাইল দেখেই না |
| কখন হয় | পরে | সব expansion এর আগে, সবার প্রথমে |

উপমা: globbing হলো রেজিস্টার খাতা থেকে "যাদের নাম 'রা' দিয়ে শুরু" খুঁজে বের করা। Brace expansion হলো ছাপাখানা, যা তালিকা অনুযায়ী নতুন নামের কার্ড ছেপে দেয়।

### তালিকা রূপ `{a,b,c}`

```
$ echo file{1,2,3}.txt
file1.txt file2.txt file3.txt
$ echo {red,green,blue}-shirt
red-shirt green-shirt blue-shirt
$ mkdir -p project/{src,docs,tests}
```

শেষেরটা একটা কমান্ডেই তিনটা ফোল্ডার বানায়। (`mkdir` নিয়ে [অধ্যায় ৯](09-file-management.md)।)

### পরিসর রূপ `{start..end}` ও `{start..end..step}`

```
$ echo {1..5}
1 2 3 4 5
$ echo {a..e}
a b c d e
$ echo {5..1}
5 4 3 2 1
$ echo {01..10}
01 02 03 04 05 06 07 08 09 10
$ echo {0..20..5}
0 5 10 15 20
$ echo {a..z..2}
a c e g i k m o q s u w y
```

শুরুতে `0` দিলে (`{01..10}`) সব সংখ্যা একই প্রস্থে শূন্য বসিয়ে দেখায়।

### নেস্টেড আর গুণ (combination)

```
$ echo {A,B}{1,2}
A1 A2 B1 B2
$ echo img_{2025,2026}_{01..03}.jpg
img_2025_01.jpg img_2025_02.jpg img_2025_03.jpg img_2026_01.jpg img_2026_02.jpg img_2026_03.jpg
$ echo {a,b{1,2},c}
a b1 b2 c
```

### সবচেয়ে কাজের কৌশল: ব্যাকআপ কপি

```
$ cp config.yml{,.bak}
```

এটা হয়ে যায় `cp config.yml config.yml.bak`। কারণ `{,.bak}` এর প্রথম অংশ খালি, দ্বিতীয়টা `.bak`।

```
$ echo config.yml{,.bak}
config.yml config.yml.bak
$ mv report.{txt,md}          # report.txt কে report.md নামে
```

### নিয়মগুলো

- কমা বা `..` না থাকলে expansion হয় না: `echo {a}` → `{a}`।
- কমার পাশে স্পেস দেওয়া যাবে না: `{a, b}` কাজ করবে না (স্পেস শব্দ ভেঙে দেয়)।
- উদ্ধৃতির ভেতরে হয় না: `echo "{1..3}"` → `{1..3}`।
- Range এ variable কাজ করে না, কারণ brace expansion variable expansion এর **আগে** হয়: `n=5; echo {1..$n}` → `{1..5}`। এর বদলে `seq 1 $n` ব্যবহার করো।
- এটা bash/zsh এর ফিচার; POSIX `sh` (যেমন Debian/Ubuntu এর `dash`) এ নেই।

### Shell expansion এর ক্রম

bash একটা কমান্ড লাইন কোন ক্রমে বদলায়, সেটা জানলে অনেক রহস্য পরিষ্কার হয়:

```
 ১. Brace expansion          {a,b}  {1..5}
 ২. Tilde expansion          ~  ~user
 ৩. Parameter/variable       $HOME  ${var}
    Arithmetic               $((1+2))
    Command substitution     $(date)
 ৪. Word splitting           স্পেস দিয়ে শব্দ ভাগ
 ৫. Pathname expansion       *  ?  [ ]   (globbing)
 ৬. Quote removal            উদ্ধৃতি চিহ্ন সরানো
                │
                ▼
          কমান্ড চলে
```

(২ ও ৩ নম্বরের ধাপগুলো আসলে বাম থেকে ডানে একসাথে হয়।) এই ক্রমের কারণেই `{1..$n}` কাজ করে না, আর `~/{a,b}` কাজ করে। Shell expansion এর বিস্তারিত [অধ্যায় ৪১](41-shell-scripting-1.md)-এ।

---

## ৮.১৩ Tab Completion — টাইপ কম, কাজ বেশি

### কী

নাম বা কমান্ডের প্রথম কয়েকটা অক্ষর লিখে `Tab` চাপলে shell বাকিটা নিজেই পূরণ করে দেয়। এটা লিনাক্স ব্যবহারকারীর সবচেয়ে বড় গতি-বর্ধক।

### কীভাবে কাজ করে

```
টাইপ: cd Doc[Tab]
          │
   bash দেখে: কোন শব্দ পূরণ করতে হবে? → "Doc"
   এটা প্রথম শব্দ? → না, তাই ফাইল/ডিরেক্টরি খোঁজো
   (cd এর জন্য নিয়ম থাকলে শুধু ডিরেক্টরি)
          │
   cwd তে "Doc" দিয়ে শুরু:
     ┌─ একটাই মিলেছে (Documents) → পুরো নাম বসাও + "/"
     ├─ কয়েকটা মিলেছে (Documents, Doctor) → সাধারণ অংশ পর্যন্ত বসাও,
     │                                       বিপ বা কিছু না; আবার Tab চাপলে তালিকা
     └─ কিছুই মেলেনি → কিছু হয় না (বা বিপ)
```

- লাইনের **প্রথম শব্দে** Tab দিলে bash কমান্ডের নাম পূরণ করে (`PATH` এর প্রোগ্রাম, alias, builtin, function)।
- পরের শব্দগুলোতে সাধারণত ফাইলের নাম।
- `$` দিয়ে শুরু হলে variable এর নাম, `~` দিয়ে শুরু হলে ইউজারের নাম পূরণ করে।

### Programmable completion (`bash-completion`)

`bash-completion` প্যাকেজ (Ubuntu, Debian, Fedora তে সাধারণত আগে থেকেই ইনস্টল থাকে) প্রতিটা কমান্ডের জন্য আলাদা বুদ্ধিমান নিয়ম যোগ করে:

```
$ git che[Tab]          → git checkout
$ systemctl sta[Tab][Tab]
start   status
$ ssh us[Tab]           → ~/.ssh/config বা known_hosts থেকে host এর নাম
$ sudo apt install ngi[Tab]   → nginx (প্যাকেজের নাম)
$ ls --col[Tab]         → ls --color=
```

নিয়মগুলো থাকে `/usr/share/bash-completion/completions/` এ। না থাকলে `sudo apt install bash-completion` বা `sudo dnf install bash-completion`। `complete -p cd` দিয়ে দেখতে পারো কোনো কমান্ডের জন্য কোন নিয়ম আছে।

### Tab completion এর কৌশল

1. **সবসময় Tab চাপো।** শুধু দ্রুততার জন্য না, এটা বানান যাচাইও। Tab কাজ না করলে বুঝবে নাম ভুল বা ফাইল নেই।
2. **দুইবার Tab** (`Tab Tab`) চাপলে সব সম্ভাব্য বিকল্প দেখায়।
3. **অনেক বিকল্প থাকলে** bash জিজ্ঞেস করে: `Display all 2154 possibilities? (y or n)`। `n` চাপো, তারপর আরও কিছু অক্ষর লেখো।
4. **স্পেস থাকা নাম:** Tab নিজেই `\ ` বসিয়ে দেয় (`My\ Documents/`)।
5. **লম্বা path এক এক ধাপে:** `cd /u[Tab]sh[Tab]do[Tab]` → `cd /usr/share/doc/`।
6. **খালি লাইনে `Tab Tab`** সব কমান্ডের তালিকা দেখায় (হাজারখানেক!)।

### Readline কনফিগারেশন (`~/.inputrc`) দিয়ে Tab কে আরও ভালো করা

bash এর লাইন-এডিটিং আসে **Readline** লাইব্রেরি থেকে। এর সেটিং থাকে `~/.inputrc` এ (system-wide `/etc/inputrc`):

```
# ~/.inputrc
$include /etc/inputrc
set completion-ignore-case on      # বড়/ছোট হাতের পার্থক্য উপেক্ষা
set show-all-if-ambiguous on       # একবার Tab এই তালিকা দেখাও
set mark-symlinked-directories on  # symlink করা ডিরেক্টরির শেষে / বসাও
set colored-stats on               # তালিকায় ls এর মতো রঙ
"\e[A": history-search-backward    # ↑ = যা লিখেছ তা দিয়ে শুরু হওয়া পুরনো কমান্ড
"\e[B": history-search-forward
```

নতুন টার্মিনাল খুললে বা `bind -f ~/.inputrc` চালালে কার্যকর হয়। (প্রথম লাইনটা system এর ডিফল্ট সেটিং হারানো ঠেকায়।) শেল কাস্টমাইজেশন নিয়ে বিস্তারিত [অধ্যায় ৪৪](44-environment-bashrc.md)-এ।

> 💡 **টিপস:** zsh বা fish shell এ Tab completion আরও উন্নত (মেনু, বর্ণনাসহ)। shell এর পার্থক্য [অধ্যায় ৫](05-terminal-o-shell.md)-এ আছে।

---

## ৮.১৪ নেভিগেশন সংক্রান্ত কীবোর্ড শর্টকাট

এগুলো bash এর ডিফল্ট (emacs মোড) শর্টকাট। পুরো শর্টকাট-তালিকা [অধ্যায় ৬](06-terminal-shortcuts.md) আর [পরিশিষ্ট ক](appendix-a-shortcuts-cheatsheet.md)-এ।

| শর্টকাট | কাজ |
|---------|-----|
| `Tab` | নাম/কমান্ড পূরণ (completion) |
| `Tab Tab` | সব সম্ভাব্য বিকল্প দেখাও |
| `Alt+?` | সম্ভাব্য completion এর তালিকা দেখাও (`Tab Tab` এর মতো) |
| `Alt+*` | সব সম্ভাব্য completion লাইনে বসিয়ে দাও |
| `Alt+/` | ফাইলের নাম হিসেবে completion জোর করো |
| `Alt+.` | আগের কমান্ডের শেষ argument বসাও (যেমন `mkdir newdir` তারপর `cd Alt+.`) |
| `Ctrl+L` | স্ক্রিন পরিষ্কার (`clear` এর মতো) |
| `Ctrl+C` | চলমান কমান্ড থামাও / লাইন বাতিল |
| `Ctrl+A` / `Ctrl+E` | লাইনের শুরু / শেষে যাও |
| `Ctrl+W` | কার্সরের আগের একটা শব্দ মোছো (path এর ভুল অংশ মুছতে কাজের) |
| `Ctrl+U` | কার্সরের আগের পুরো লাইন মোছো |
| `Ctrl+R` | পুরনো কমান্ড খোঁজো (যেমন আগের লম্বা `cd` path) |
| `Shift+PageUp` / `PageDown` | টার্মিনালের আউটপুট স্ক্রল (বেশিরভাগ টার্মিনাল এমুলেটরে) |

কমান্ড-ভিত্তিক "শর্টকাট":

| লেখো | মানে |
|------|------|
| `cd` | home এ যাও |
| `cd -` | আগের ডিরেক্টরিতে |
| `cd ..` | উপরে |
| `~` | home |
| `~-` / `~+` | আগের / বর্তমান ডিরেক্টরি |
| `!$` | আগের কমান্ডের শেষ argument (history expansion) |

```
$ mkdir -p ~/work/2026/reports
$ cd !$
cd ~/work/2026/reports
```

---

## ৮.১৫ সব মিলিয়ে একটা বাস্তব দৃশ্য

একজন অ্যাডমিনের সকাল:

```
$ cd /var/log                    # লগ ফোল্ডারে
$ ls -ltr | tail -5              # সবচেয়ে নতুন ৫টা ফাইল
$ ls -lhS | head                 # কোনটা সবচেয়ে বড়?
$ pushd /etc/nginx > /dev/null   # কনফিগ দেখতে যাই, লগের জায়গা stack এ থাকুক
$ ls sites-enabled/
default  mysite.conf
$ cp sites-enabled/mysite.conf{,.bak}   # ব্যাকআপ
$ popd > /dev/null               # লগে ফেরত
$ pwd
/var/log
$ ls nginx/*.log
nginx/access.log  nginx/error.log
$ cd                             # বাড়ি
```

---

## হাতে-কলমে অনুশীলন

নিচের ল্যাবগুলো নিজের home ডিরেক্টরিতে নিরাপদে করা যাবে। (`mkdir`, `touch` এর বিস্তারিত পরের অধ্যায়ে, এখানে শুধু অনুশীলনের জায়গা বানাতে ব্যবহার করছি।)

### ল্যাব ১: পরিবেশ তৈরি

```
$ cd
$ mkdir -p lab8/{docs,src/{utils,tests},music,.secret}
$ touch lab8/docs/{report,notes,todo}.txt
$ touch lab8/src/main.c lab8/src/utils/helper.{c,h}
$ touch lab8/music/song{1..12}.mp3
$ touch lab8/.config lab8/README.md
$ tree -a lab8
```

প্রত্যাশিত (আংশিক):

```
lab8
├── .config
├── .secret
├── README.md
├── docs
│   ├── notes.txt
│   ├── report.txt
│   └── todo.txt
├── music
│   ├── song1.mp3
...
```

(`tree` না থাকলে `ls -R lab8` ব্যবহার করো।)

### ল্যাব ২: pwd ও cd

1. `cd ~/lab8/src/utils` চালাও, তারপর `pwd`।
2. `cd ../..` দিয়ে কোথায় পৌঁছলে? `pwd` দিয়ে যাচাই করো (উত্তর: `~/lab8`)।
3. `cd /etc` তারপর `cd -` — কোথায় ফিরলে?
4. `cd` (কিছু না দিয়ে) — এখন কোথায়?
5. `/` এ গিয়ে `cd ..` চালাও, তারপর `pwd`। কী দেখলে এবং কেন?

### ল্যাব ৩: absolute vs relative

`~/lab8/docs` এ থেকে নিচের প্রতিটা ফাইলে **দুইভাবে** `ls` চালাও (absolute ও relative):

- `~/lab8/src/main.c`
- `~/lab8/music/song5.mp3`
- `/etc/hostname`

উদাহরণ: `ls ../src/main.c` আর `ls /home/<তোমার-নাম>/lab8/src/main.c`।

### ল্যাব ৪: ls এর অপশন

```
$ cd ~/lab8
$ ls
$ ls -a
$ ls -A
$ ls -lh
$ ls -F
$ ls -ld docs
$ ls -i docs
$ ls -lt music | head -3
$ ls -v music
$ ls -R
```

প্রশ্ন: `ls music` আর `ls -v music` এর ক্রমে পার্থক্য কী? (ইঙ্গিত: `song10` আর `song2` কোথায় আছে দেখো।)

### ল্যাব ৫: `ls -l` পড়া

```
$ ls -l ~/lab8 /etc/passwd /dev/null
```

প্রতিটা লাইনের জন্য খাতায় লেখো: টাইপ, owner এর permission, link count, owner, group, size। `/dev/null` এর size এর জায়গায় কী আছে আর কেন?

### ল্যাব ৬: globbing

```
$ cd ~/lab8/music
$ echo song?.mp3
$ echo song??.mp3
$ echo song[1-3].mp3
$ echo song1*.mp3
$ echo song[!1]*.mp3
$ echo *.wav
$ shopt -s nullglob; echo *.wav; shopt -u nullglob
$ cd ..
$ echo *
$ echo .*
$ shopt -s dotglob; echo *; shopt -u dotglob
$ shopt -s globstar; echo **/*.c; shopt -u globstar
```

প্রতিটার output লিখে নিজের কাছে ব্যাখ্যা করো কেন এটা এসেছে।

### ল্যাব ৭: brace expansion

```
$ echo backup-{mon,tue,wed}.tar.gz
$ echo {10..1..3}
$ cp ~/lab8/README.md{,.bak}
$ ls ~/lab8/README*
```

### ল্যাব ৮: directory stack

```
$ cd ~/lab8
$ pushd docs
$ pushd ../src/utils
$ pushd /etc
$ dirs -v
$ pushd +2
$ popd
$ popd
$ dirs -v
```

প্রতিটা ধাপের পর `pwd` দিয়ে দেখো কোথায় আছ।

### ল্যাব ৯: Tab completion

1. `cd ~/la` লিখে `Tab` চাপো।
2. `ls ~/lab8/music/song1` লিখে `Tab Tab` চাপো, কয়টা বিকল্প এলো?
3. `sys` লিখে `Tab Tab` চাপো, কোন কোন কমান্ড দেখাল?
4. `mkdir ~/lab8/temp` চালাও, তারপর `cd ` লিখে `Alt+.` চাপো।

### পরিষ্কার

```
$ cd
$ rm -r ~/lab8
```

> ⚠️ **সতর্কতা:** `rm -r` এর আগে path টা আবার দেখে নাও। এটা ফাইল চিরতরে মুছে দেয়, Trash এ যায় না ([অধ্যায় ৯](09-file-management.md))।

---

## সারসংক্ষেপ

| কমান্ড / চিহ্ন | কাজ |
|---------------|-----|
| `pwd` | বর্তমান ডিরেক্টরি দেখাও |
| `pwd -P` | symlink resolve করে আসল path |
| `cd DIR` | DIR এ যাও |
| `cd` / `cd ~` | home এ যাও |
| `cd -` | আগের ডিরেক্টরিতে |
| `cd ..` | parent এ |
| `cd /` | root এ |
| `/a/b` | absolute path (root থেকে) |
| `a/b` | relative path (cwd থেকে) |
| `.` / `..` | বর্তমান / parent ডিরেক্টরি |
| `~`, `~user`, `~+`, `~-` | home, অন্যের home, `$PWD`, `$OLDPWD` |
| `ls -l` | বিস্তারিত তালিকা |
| `ls -a` / `ls -A` | লুকানো ফাইলসহ (`.` `..` সহ / ছাড়া) |
| `ls -lh` | পড়ার মতো size |
| `ls -ltr` | পুরনো থেকে নতুন (নতুনটা নিচে) |
| `ls -lS` | বড় থেকে ছোট |
| `ls -R` | recursive |
| `ls -ld DIR` | ডিরেক্টরির নিজের তথ্য |
| `ls -i` | inode নম্বর |
| `ls -F` | টাইপ চিহ্ন (`/ * @`) |
| `tree -L 2` | ২ স্তর পর্যন্ত গাছ |
| `pushd DIR` / `popd` / `dirs -v` | ডিরেক্টরি stack |
| `*` `?` `[abc]` `[!abc]` | globbing |
| `**` | সব স্তর (`shopt -s globstar`) |
| `{a,b}` `{1..10}` `{1..10..2}` | brace expansion |
| `Tab` / `Tab Tab` | completion / বিকল্প তালিকা |
| `Alt+.` | আগের কমান্ডের শেষ argument |
| `Ctrl+L` | স্ক্রিন পরিষ্কার |

---

## নিজেকে যাচাই করো

1. `cd` কেন একটা shell builtin, আলাদা প্রোগ্রাম হতে পারে না?
2. তুমি `/home/rahim/Documents` এ আছ। `/home/karim/notes.txt` এর relative path কী?
3. `ls -a` আর `ls -A` এর পার্থক্য কী?
4. `ls -l` এর একটা লাইন: `drwxr-x--- 5 rahim dev 4096 Nov 12 09:30 proj`। এখানে টাইপ কী, group এর permission কী, আর `5` সংখ্যাটা কী বোঝায়?
5. `ls -l /etc` আর `ls -ld /etc` এর পার্থক্য কী?
6. `echo *.txt` চালালে `*` কে expand করে কে — `echo` নাকি shell? কিছু না মিললে bash ডিফল্টে কী করে?
7. `echo {1..3}` আর `echo [1-3]` এর মধ্যে মূল পার্থক্য কী?
8. `cp app.conf{,.orig}` কমান্ডটা আসলে কী চালায়?
9. `pushd /etc`, `pushd /var`, `popd` চালানোর পর তুমি কোথায় থাকবে (শুরু `~` থেকে)?
10. স্ক্রিপ্টে `"~/backup"` লিখলে কী সমস্যা হবে, আর ঠিক উপায় কী?

<details><summary>উত্তর দেখো</summary>

1. cwd প্রতিটা process এর নিজস্ব। আলাদা প্রোগ্রাম হলে child process নিজের cwd বদলে বেরিয়ে যেত, আর shell এর cwd অপরিবর্তিত থাকত। তাই shell কে নিজেই `chdir()` ডাকতে হয়।
2. `../../karim/notes.txt` (Documents → rahim → home, তারপর karim)।
3. `-a` সব লুকানো এন্ট্রিসহ `.` আর `..` দেখায়; `-A` লুকানো ফাইল দেখায় কিন্তু `.` আর `..` বাদ দেয়।
4. `d` = ডিরেক্টরি; group এর permission `r-x` (পড়া ও ঢোকা যাবে, লেখা যাবে না); `5` হলো hard link সংখ্যা, ডিরেক্টরির ক্ষেত্রে সাধারণত ২ + ৩টা সাব-ডিরেক্টরি।
5. `ls -l /etc` /etc এর ভেতরের সব ফাইলের তালিকা দেখায়; `ls -ld /etc` শুধু /etc ডিরেক্টরিটার নিজের তথ্য দেখায়।
6. Shell expand করে; `echo` আগেই তৈরি তালিকা পায়। কিছু না মিললে bash pattern টাকে অক্ষত (`*.txt`) রেখে দেয় (যদি না `nullglob`/`failglob` চালু থাকে)।
7. `{1..3}` ফাইল থাকুক না থাকুক সবসময় `1 2 3` তৈরি করে (brace expansion)। `[1-3]` হলো glob, শুধু cwd-তে `1`, `2` বা `3` নামের ফাইল থাকলে সেগুলোর সাথে মেলে, না থাকলে আক্ষরিক `[1-3]` থাকে।
8. `cp app.conf app.conf.orig` — অর্থাৎ ফাইলের একটা ব্যাকআপ কপি।
9. `/etc` এ। stack ছিল `/var /etc ~`; `popd` উপরের `/var` সরিয়ে `/etc` এ নিয়ে যায়।
10. উদ্ধৃতির ভেতরে tilde expand হয় না, তাই আক্ষরিক `~` নামের ফোল্ডার বোঝাবে। ঠিক উপায়: `"$HOME/backup"`।

</details>

---

[← আগের অধ্যায়](07-filesystem-hierarchy.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](09-file-management.md)

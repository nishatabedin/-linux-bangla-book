# অধ্যায় ২৪: প্রসেস ম্যানেজমেন্ট

তুমি যখন টার্মিনালে `ls` লিখে Enter চাপো, ব্রাউজারে একটা ট্যাব খোলো, বা ব্যাকগ্রাউন্ডে কোনো ওয়েব সার্ভার চলে, প্রতিটা ক্ষেত্রেই লিনাক্সের ভেতরে একটা করে **process** তৈরি হয়। process বুঝলে বোঝা যায় কম্পিউটার আসলে "কী করছে"। সার্ভার হঠাৎ ধীর হয়ে গেলে, কোনো প্রোগ্রাম আটকে গেলে বা মেমরি ফুরিয়ে গেলে প্রথম যে দক্ষতা কাজে লাগে, তা হলো প্রসেস ম্যানেজমেন্ট।

## এই অধ্যায়ে যা শিখবে

- program, process আর thread-এর পার্থক্য
- PID, PPID কী, প্রসেস ট্রি কীভাবে গঠিত হয়, PID 1 কেন বিশেষ
- `fork()`, `exec()`, `wait()`, `exit()` দিয়ে প্রসেসের জন্ম থেকে মৃত্যু পর্যন্ত জীবনচক্র
- প্রসেসের state: `R`, `S`, `D`, `Z`, `T` ও আরও কয়েকটি
- zombie আর orphan প্রসেস কী, কেন হয়, কী করতে হয়
- `ps` কমান্ড: `aux`, `-ef`, `-o`, `--forest`, sorting
- `top` আর `htop`, সাথে প্রায় সব interactive শর্টকাট
- `pgrep`, `pkill`, `pidof` দিয়ে প্রসেস খোঁজা
- signals: SIGHUP, SIGINT, SIGTERM, SIGKILL, SIGSTOP, SIGCONT ইত্যাদি
- `kill`, `killall` দিয়ে প্রসেস থামানো বা সিগন্যাল পাঠানো
- `nice`, `renice` দিয়ে priority বদলানো
- `/proc/PID` ফোল্ডারের ভেতরে কী আছে
- load average পড়া ও বোঝা

---

## ২৪.১ program, process আর thread

### program: রেসিপির বই

**program** হলো ডিস্কে রাখা একটা ফাইল, যেমন `/usr/bin/ls` বা `/usr/bin/python3`। এর ভেতরে আছে মেশিন কোড (CPU যে নির্দেশ বোঝে)। ফাইলটা চুপচাপ পড়ে থাকে, কোনো কাজ করে না।

**উপমা:** রান্নার রেসিপির বই। বইটা তাকে থাকলে কোনো রান্না হয় না।

### process: রান্না চলছে

**process** হলো চলমান program, মানে program-এর একটা জীবন্ত instance। kernel যখন program চালায়, তখন সে:

- মেমরিতে জায়গা দেয় (code, data, heap, stack)
- একটা ইউনিক নম্বর দেয়: **PID** (Process ID)
- খোলা ফাইলের তালিকা (file descriptor) রাখে
- কোন user চালাচ্ছে, কোন ডিরেক্টরিতে আছে, environment variable কী, এসব হিসাব রাখে
- CPU-তে কখন কতক্ষণ চলবে তা scheduler ঠিক করে

**উপমা:** রেসিপি দেখে রান্নাঘরে রান্না চলছে। একই রেসিপি দিয়ে তিনটা চুলায় তিনবার রান্না হতে পারে। ঠিক তেমনি একই `bash` program থেকে একসাথে দশটা আলাদা bash process চলতে পারে, প্রতিটার নিজস্ব PID আর নিজস্ব মেমরি।

```
ডিস্কে (program)                 মেমরিতে (process)
┌────────────────┐   exec    ┌─────────────────────────┐
│ /usr/bin/bash  │ ────────► │ PID 2041  bash (tty1)   │
│  (একটাই ফাইল)  │ ────────► │ PID 3310  bash (pts/0)  │
│                │ ────────► │ PID 3388  bash (pts/1)  │
└────────────────┘           └─────────────────────────┘
```

### thread: এক রান্নাঘরে কয়েকজন রাঁধুনি

একটা process-এর ভেতরে এক বা একাধিক **thread** (কাজের সুতো) থাকতে পারে। একই process-এর সব thread একই মেমরি, একই খোলা ফাইল শেয়ার করে, কিন্তু প্রত্যেকের নিজস্ব stack আর CPU register থাকে। তাই তারা একসাথে আলাদা আলাদা CPU core-এ চলতে পারে।

**উপমা:** একটা রান্নাঘরে (process) তিনজন রাঁধুনি (thread)। একজন সবজি কাটছে, একজন ভাত রাঁধছে, একজন মসলা বাটছে। ফ্রিজ, চুলা, হাঁড়ি সবাই শেয়ার করছে।

```
┌──────────────────── Process (PID 5000) ─────────────────────┐
│  শেয়ারড: code, heap, global data, খোলা ফাইল, PID           │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                 │
│  │ Thread 1 │   │ Thread 2 │   │ Thread 3 │                 │
│  │ TID 5000 │   │ TID 5001 │   │ TID 5002 │                 │
│  │ নিজস্ব   │   │ নিজস্ব   │   │ নিজস্ব   │                 │
│  │ stack,   │   │ stack,   │   │ stack,   │                 │
│  │ register │   │ register │   │ register │                 │
│  └──────────┘   └──────────┘   └──────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

> 📝 **মনে রাখো:** লিনাক্স kernel-এর চোখে thread আর process প্রায় একই জিনিস। দুটোই একটা **task**, তৈরি হয় `clone()` system call দিয়ে। পার্থক্য শুধু কতটা জিনিস শেয়ার করা হলো। প্রথম thread-এর TID (Thread ID) আর process-এর PID একই হয়। `ps`-এ thread-কে **LWP** (Light Weight Process) বলে দেখানো হয়।

| বিষয় | program | process | thread |
|------|---------|---------|--------|
| কোথায় থাকে | ডিস্কে | মেমরিতে, চলমান | process-এর ভেতরে |
| পরিচয় | ফাইলের path | PID | TID (LWP) |
| মেমরি | নেই | নিজস্ব, আলাদা | process-এর সাথে শেয়ারড |
| একটা crash করলে | প্রযোজ্য নয় | অন্য process নিরাপদ | পুরো process মারা যেতে পারে |
| তৈরি হয় | compiler/installer দিয়ে | `fork()` + `exec()` | `clone()` (pthread লাইব্রেরি) |

একটা process-এ কয়টা thread আছে দেখতে:

```
$ ps -o pid,nlwp,cmd -p 1
    PID NLWP CMD
      1    1 /sbin/init splash

$ ps -T -p $(pidof -s firefox)
    PID    SPID TTY          TIME CMD
   7123    7123 ?        00:01:12 firefox
   7123    7130 ?        00:00:00 gmain
   7123    7131 ?        00:00:03 gdbus
   ...
```

`NLWP` = Number of LWP, মানে thread সংখ্যা। `-T` দিলে প্রতিটা thread আলাদা লাইনে দেখায়, `SPID` হলো thread ID। সব process-এর সব thread একসাথে দেখতে `ps -eLf`।

---

## ২৪.২ PID ও PPID

প্রতিটা process-এর দুটো গুরুত্বপূর্ণ নম্বর থাকে:

- **PID** (Process ID): তার নিজের পরিচয় নম্বর।
- **PPID** (Parent Process ID): যে process তাকে জন্ম দিয়েছে, তার PID।

লিনাক্সে প্রতিটা process (PID 1 বাদে) অন্য কোনো process থেকে জন্মায়। তাই সবার একটা parent আছে।

```
$ echo $$          # বর্তমান শেলের PID
3310
$ echo $PPID       # বর্তমান শেলের parent-এর PID
3295
$ ps -o pid,ppid,cmd -p $$
    PID    PPID CMD
   3310    3295 bash
$ ps -o pid,ppid,cmd -p 3295
    PID    PPID CMD
   3295    1890 /usr/libexec/gnome-terminal-server
```

তাহলে বোঝা গেল: gnome-terminal (3295) একটা bash (3310) চালিয়েছে।

শেলের কয়েকটা বিশেষ variable:

| variable | মানে |
|----------|------|
| `$$` | বর্তমান শেলের PID |
| `$PPID` | বর্তমান শেলের parent-এর PID |
| `$!` | সবশেষে ব্যাকগ্রাউন্ডে (`&` দিয়ে) চালানো process-এর PID |
| `$BASHPID` | বর্তমান bash process-এর আসল PID (subshell-এর ভেতরে `$$` থেকে আলাদা হতে পারে) |

```
$ sleep 100 &
[1] 4521
$ echo $!
4521
```

### PID কীভাবে দেওয়া হয়

- kernel ক্রমানুসারে বাড়তে থাকা নম্বর দেয়। সর্বোচ্চ সীমায় পৌঁছালে আবার ছোট নম্বর থেকে শুরু করে (যেগুলো খালি)।
- সর্বোচ্চ সীমা থাকে `/proc/sys/kernel/pid_max`-এ। kernel-এর নিজস্ব ডিফল্ট `32768`, তবে Ubuntu 24.04, Fedora, RHEL 9-এর মতো আধুনিক 64-bit সিস্টেমে systemd এটা সাধারণত `4194304` করে দেয়।

```
$ cat /proc/sys/kernel/pid_max
4194304
```

| বিশেষ PID | কে |
|-----------|----|
| 0 | idle task (ঐতিহাসিক নাম "swapper")। এটা kernel-এর ভেতরে থাকে, `ps`-এ দেখা যায় না। |
| 1 | init process, আধুনিক distro-তে **systemd** |
| 2 | `kthreadd`, সব kernel thread-এর parent |

> ⚠️ **সতর্কতা:** PID স্থায়ী নয়। একটা process মরে গেলে কিছুক্ষণ পর তার PID অন্য নতুন process পেতে পারে। তাই স্ক্রিপ্টে পুরোনো PID ফাইলে লিখে রেখে অনেকক্ষণ পর `kill` করলে ভুল process মারা যেতে পারে।

---

## ২৪.৩ প্রসেস ট্রি ও PID 1

সব process মিলে একটা গাছ (tree) তৈরি হয়। গাছের গোড়ায় থাকে **PID 1**।

```
                       systemd (1)
          ┌───────────────┼──────────────────┬──────────────┐
   systemd-journald   sshd (812)        gdm (950)     kthreadd (2) ← kernel threads
                          │                 │              ├── kworker/0:1
                   sshd: nishat (2101)  gnome-session      ├── ksoftirqd/0
                          │                 │              └── ...
                     bash (2120)     gnome-terminal (3295)
                          │                 │
                     vim (2210)        bash (3310)
                                            │
                                       pstree (3401)
```

### PID 1 কেন বিশেষ

PID 1 হলো প্রথম user-space process। kernel বুট শেষে এটা চালায় (বিস্তারিত [অধ্যায় ২৭](27-boot-process.md))। আধুনিক Ubuntu, Debian, Fedora, RHEL সবখানে এটা **systemd**। এর দায়িত্ব:

1. বাকি সব সার্ভিস চালু করা ([অধ্যায় ২৬](26-systemd-services.md))।
2. **অনাথ (orphan) process দত্তক নেওয়া** এবং তাদের মৃত্যুর পর পরিষ্কার করা (reap)।
3. এটা মরতে পারে না। PID 1 মারা গেলে kernel "Attempted to kill init!" বলে **kernel panic** করে।
4. kernel PID 1-কে বিশেষ সুরক্ষা দেয়: যে signal-এর জন্য PID 1 নিজে handler বসায়নি, সেগুলো তার কাছে পৌঁছায় না। তাই root হয়েও `kill -9 1` দিলে systemd মরে না।

```
$ ps -p 1 -o pid,comm
    PID COMMAND
      1 systemd
$ ls -l /sbin/init
lrwxrwxrwx 1 root root 22 ... /sbin/init -> ../lib/systemd/systemd
```

### pstree: গাছটা চোখে দেখো

```
$ pstree
systemd─┬─ModemManager───3*[{ModemManager}]
        ├─NetworkManager───3*[{NetworkManager}]
        ├─cron
        ├─sshd───sshd───sshd───bash───pstree
        ├─systemd-journal
        ...
```

`3*[{ModemManager}]` মানে ModemManager-এর ভেতরে ৩টা thread (কোঁকড়া বন্ধনী `{}` = thread)। একই রকম অনেকগুলো child থাকলে `N*[...]` আকারে গুটিয়ে দেখায়।

| অপশন | কাজ |
|------|-----|
| `-p` | প্রতিটা নামের পাশে PID দেখায় (এতে গুটানো বন্ধ হয়) |
| `-a` | পুরো command line (arguments সহ) দেখায় |
| `-u` | user বদলালে বন্ধনীতে user নাম দেখায় |
| `-h` | বর্তমান process ও তার পূর্বপুরুষদের highlight করে |
| `-s PID` | ওই PID-এর parent-দের (ancestor) দেখায় |
| `-T` | thread লুকায় (নতুন psmisc ভার্সনে) |
| `-n` | নাম না, PID অনুযায়ী সাজায় |
| `-c` | একই রকম subtree গুটায় না |
| `USER` বা `PID` | শুধু ওই user বা ওই PID থেকে শুরু হওয়া গাছ |

```
$ pstree -p $$
bash(3310)───pstree(3420)

$ pstree -s -p $$
systemd(1)───systemd(1720)───gnome-terminal-(3295)───bash(3310)───pstree(3421)
```

> 💡 **টিপস:** `pstree` আছে `psmisc` প্যাকেজে। মিনিমাল সার্ভারে না থাকলে `sudo apt install psmisc` (Debian/Ubuntu) বা `sudo dnf install psmisc` (Fedora/RHEL)। বিকল্প হিসেবে `ps -ef --forest` সব জায়গায় কাজ করে।

---

## ২৪.৪ process-এর জীবনচক্র: fork, exec, wait, exit

লিনাক্সে নতুন process তৈরির মূল কৌশল দুই ধাপের, যেটা Unix-এর একটা সুন্দর ডিজাইন:

1. **`fork()`**: parent নিজের একটা প্রায়-হুবহু কপি বানায়। নতুন child-এর PID আলাদা, PPID = parent-এর PID। বাকি সব (মেমরি, খোলা ফাইল, environment) কপি হয়।
2. **`exec()`** (আসলে `execve()`): child তার নিজের মেমরি মুছে নতুন program লোড করে। PID একই থাকে, কিন্তু ভেতরের code বদলে যায়।
3. **`exit()`**: child কাজ শেষে exit code (0 = সফল, অন্য সংখ্যা = সমস্যা) দিয়ে বের হয়ে যায়।
4. **`wait()`** (বা `waitpid()`): parent child-এর exit code সংগ্রহ করে। তখনই kernel child-এর শেষ চিহ্ন মুছে ফেলে।

### উদাহরণ: শেলে `ls` লিখলে ভেতরে কী হয়

```
   bash (PID 3310)
        │
        │ 1. fork()
        ├──────────────────────────────┐
        │                              ▼
        │                     bash-এর কপি (PID 3500, PPID 3310)
        │                              │
        │ 2. wait() করে বসে থাকে       │ 3. execve("/usr/bin/ls", ...)
        │    (state = S)               ▼
        │                     ls (PID 3500) ← একই PID, নতুন program
        │                              │
        │                              │ 4. ফাইলের তালিকা প্রিন্ট করে
        │                              │ 5. exit(0)
        │                              ▼
        │                     [zombie: শুধু exit code বাকি]
        │ 6. wait() exit code পায় ◄───┘
        │    kernel child মুছে দেয়
        ▼
   আবার prompt দেখায়:  $
   echo $? → 0
```

**উপমা:** fork মানে নিজের ফটোকপি তৈরি করা। exec মানে সেই ফটোকপির ওপর পুরো নতুন লেখা ছাপিয়ে দেওয়া। wait মানে মা অপেক্ষা করছে সন্তান কাজ শেষ করে রিপোর্ট দেবে।

### কেন দুই ধাপে?

fork আর exec-এর মাঝখানে child-এর হাতে একটা সুযোগ থাকে, নিজের পরিবেশ ঠিক করে নেওয়ার। শেল ঠিক এই ফাঁকেই redirection আর pipe সেট করে ([অধ্যায় ১৯](19-redirection-pipe.md))। যেমন `ls > out.txt` চালালে child fork-এর পর stdout-কে `out.txt`-এ ঘুরিয়ে দেয়, তারপর exec করে। `ls` নিজে জানেই না যে তার output ফাইলে যাচ্ছে।

### copy-on-write: কপি আসলে সস্তা

fork-এ পুরো মেমরি কপি করা খুব ব্যয়বহুল মনে হতে পারে। কিন্তু kernel **copy-on-write (COW)** ব্যবহার করে: শুরুতে parent আর child একই মেমরি page শেয়ার করে (read-only হিসেবে চিহ্নিত)। কেউ লিখতে গেলে তখনই শুধু ওই page কপি হয়। যেহেতু child সাধারণত সাথে সাথেই exec করে, বেশিরভাগ page কখনো কপিই হয় না।

### builtin কমান্ডে fork হয় না

`cd`, `echo`, `export` এগুলো shell builtin। শেল নিজের ভেতরেই চালায়, নতুন process বানায় না। `cd` যদি আলাদা process হতো, তাহলে child-এর ডিরেক্টরি বদলাত, parent শেলের না, তাই কোনো লাভই হতো না।

```
$ type cd ls
cd is a shell builtin
ls is aliased to `ls --color=auto'
```

### `exec` কমান্ড: fork ছাড়া exec

শেলে `exec` builtin দিলে fork হয় না, বর্তমান শেলটাই নতুন program দিয়ে বদলে যায়:

```
$ exec top     # top থেকে q চাপলে টার্মিনাল ট্যাবটাই বন্ধ হয়ে যাবে, কারণ শেল আর নেই
```

---

## ২৪.৫ প্রসেসের state (অবস্থা)

একটা process সবসময় CPU-তে চলে না। বেশিরভাগ সময় সে কোনো কিছুর জন্য অপেক্ষা করে। kernel প্রতিটা process-এর একটা state রাখে। `ps`-এর `STAT` কলাম আর `top`-এর `S` কলামে এটা দেখা যায়।

| কোড | নাম | মানে | উপমা |
|-----|-----|------|------|
| `R` | Running / Runnable | CPU-তে চলছে, বা চলার জন্য লাইনে (run queue) দাঁড়িয়ে | কাউন্টারে সার্ভ হচ্ছে বা লাইনে আছে |
| `S` | Interruptible Sleep | কোনো ঘটনার জন্য অপেক্ষা (কীবোর্ড ইনপুট, নেটওয়ার্ক, টাইমার)। signal এলে জেগে ওঠে | ফোনের অপেক্ষায় ঘুমাচ্ছে, ফোন বাজলেই উঠবে |
| `D` | Uninterruptible Sleep | সাধারণত disk/NFS I/O শেষ হওয়ার অপেক্ষা। signal-এও জাগে না, এমনকি SIGKILL-এও না | অপারেশন টেবিলে, মাঝপথে থামানো যাবে না |
| `T` | Stopped | SIGSTOP বা Ctrl+Z (SIGTSTP) দিয়ে থামানো | pause বাটন চাপা ভিডিও |
| `t` | Tracing stop | debugger (যেমন `gdb`, `strace`) থামিয়ে রেখেছে | |
| `Z` | Zombie | মারা গেছে, কিন্তু parent এখনো `wait()` করেনি | মৃত্যু সনদ জমা হয়নি |
| `I` | Idle | অলস kernel thread (load average-এ গোনা হয় না) | |
| `X` | Dead | একেবারে শেষ মুহূর্ত, প্রায় কখনো দেখা যায় না | |

### state transition ডায়াগ্রাম

```
            fork()
              │
              ▼
        ┌───────────┐  scheduler সময় দিল   ┌──────────────┐
        │ R (ready, │ ────────────────────► │ R (CPU-তে    │
        │ run queue)│ ◄──────────────────── │   চলছে)      │
        └───────────┘  সময় শেষ/preempt     └──────┬───────┘
          ▲     ▲                                  │
          │     │ ঘটনা ঘটল / I/O শেষ              │ I/O বা ঘটনার অপেক্ষা
          │     │        ┌──────────────┐          │
          │     └────────│  S  বা  D    │◄─────────┤
          │              └──────────────┘          │
          │ SIGCONT      ┌──────────────┐ SIGSTOP  │
          └──────────────│      T       │◄─────────┤
                         └──────────────┘          │ exit()
                                                   ▼
                                            ┌──────────────┐ parent wait()  ┌──────────┐
                                            │   Z (zombie) │ ─────────────► │ মুছে গেল │
                                            └──────────────┘                └──────────┘
```

### STAT কলামের অতিরিক্ত অক্ষর

`ps aux`-এ STAT প্রায়ই `Ss`, `R+`, `Sl`, `SN` এমন দেখায়। প্রথম অক্ষর state, বাকিগুলো বাড়তি তথ্য:

| অক্ষর | মানে |
|-------|------|
| `<` | উচ্চ priority (nice মান ঋণাত্মক) |
| `N` | নিম্ন priority (nice মান ধনাত্মক) |
| `L` | কিছু page মেমরিতে lock করা |
| `s` | session leader (যেমন লগইন শেল) |
| `l` | multi-threaded |
| `+` | foreground process group-এ আছে (টার্মিনালে সামনে চলছে) |

উদাহরণ: `Ssl` = ঘুমাচ্ছে, session leader, multi-threaded। `R+` = চলছে, টার্মিনালের সামনের সারিতে।

> ⚠️ **সতর্কতা:** অনেকগুলো process দীর্ঘ সময় `D` state-এ থাকলে সাধারণত disk বা নেটওয়ার্ক স্টোরেজ (NFS) সমস্যা বোঝায়। এদের `kill -9` দিয়েও মারা যায় না, কারণ kernel-এর ভেতরের I/O কাজ শেষ না হলে signal ডেলিভারিই হয় না। আসল সমস্যা (ডিস্ক, NFS সার্ভার) ঠিক করতে হয়।

---

## ২৪.৬ zombie ও orphan প্রসেস

### zombie (Z): মরেও মরেনি

child যখন `exit()` করে, kernel তার মেমরি, খোলা ফাইল সব ছেড়ে দেয়। কিন্তু process table-এ একটা ছোট্ট এন্ট্রি (PID আর exit status) রেখে দেয়, যাতে parent পরে `wait()` করে জানতে পারে child কীভাবে মারা গেছে। এই মাঝখানের অবস্থাই **zombie**। ps-এ নামের পাশে `<defunct>` লেখা থাকে।

```
$ ps aux | awk '$8 ~ /^Z/'
nishat    5123  0.0  0.0      0     0 pts/0    Z+   10:22   0:00 [myapp] <defunct>
```

- সাধারণত zombie কয়েক মিলিসেকেন্ড থাকে, তারপর parent wait() করলেই চলে যায়।
- parent প্রোগ্রামে বাগ থাকলে (wait না করলে) zombie জমতে থাকে।
- zombie কোনো CPU বা মেমরি খায় না, কিন্তু একটা PID আটকে রাখে। হাজার হাজার জমলে PID ফুরিয়ে নতুন process তৈরি বন্ধ হয়ে যেতে পারে।

**zombie মারার উপায়?** zombie তো আগেই মৃত, তাই `kill -9` কাজ করে না। উপায়:

1. parent-কে SIGCHLD পাঠাও, যদি সে সেটা পেয়ে wait() করে: `kill -s SIGCHLD <PPID>`
2. না হলে parent-কে মারো (বা restart করো)। তখন zombie গুলো orphan হয়ে PID 1-এর কাছে যায়, আর systemd সাথে সাথে wait() করে সাফ করে দেয়।

```
$ ps -o pid,ppid,stat,cmd -p 5123
    PID    PPID STAT CMD
   5123    5100 Z+   [myapp] <defunct>
$ kill 5100      # দায়ী parent-কে বন্ধ করো
```

### orphan: অনাথ process

parent আগে মারা গেলে child হয় **orphan**। kernel তখন তাকে দত্তক দেয় PID 1-কে (বা কাছের কোনো "subreaper"-কে, যেমন user-এর `systemd --user` process)। orphan সম্পূর্ণ স্বাভাবিকভাবে চলতে থাকে, এতে কোনো ক্ষতি নেই। daemon (ব্যাকগ্রাউন্ড সার্ভিস) ইচ্ছা করেই এভাবে নিজেকে orphan বানাত।

```
$ bash -c 'sleep 300 & echo "child PID: $!"'
child PID: 6001
$ ps -o pid,ppid,cmd -p 6001
    PID    PPID CMD
   6001    1720 sleep 300        ← parent bash শেষ, এখন systemd --user (1720) দত্তক নিয়েছে
```

(ডেস্কটপে সাধারণত `systemd --user` দত্তক নেয়, সার্ভারে SSH সেশনে প্রায়ই PID 1 বা user manager।)

| | zombie | orphan |
|--|--------|--------|
| জীবিত? | না, শুধু এন্ট্রি বাকি | হ্যাঁ, স্বাভাবিকভাবে চলছে |
| কারণ | parent wait() করেনি | parent আগেই মারা গেছে |
| state | `Z` | যেকোনো (R, S...) |
| ক্ষতিকর? | বেশি জমলে PID ফুরায় | না |
| সমাধান | parent ঠিক করো/মারো | দরকার নেই |

---

## ২৪.৭ ps: প্রসেসের স্ন্যাপশট

`ps` (process status) কমান্ড চালানোর মুহূর্তের একটা ছবি দেয়। আসলে এটা `/proc` ফোল্ডার পড়ে সাজিয়ে দেখায়।

একটা মজার ইতিহাস আছে: `ps` তিন ধরনের অপশন স্টাইল বোঝে:

| স্টাইল | উদাহরণ | উৎস |
|--------|--------|-----|
| BSD | `ps aux` (ড্যাশ নেই) | BSD Unix |
| UNIX/POSIX | `ps -ef` (এক ড্যাশ) | System V Unix |
| GNU long | `ps --forest`, `--sort` (দুই ড্যাশ) | GNU |

> ⚠️ **সতর্কতা:** `ps aux` আর `ps -aux` এক জিনিস নয়। `-aux` লিখলে ps সেটাকে "user x-এর process" হিসেবে বুঝতে পারে। সবসময় `ps aux` লেখো, ড্যাশ ছাড়া।

### শুধু `ps`

```
$ ps
    PID TTY          TIME CMD
   3310 pts/0    00:00:00 bash
   3502 pts/0    00:00:00 ps
```

শুধু বর্তমান টার্মিনালের process দেখায়।

### `ps aux`: সব process, BSD স্টাইল

- `a` = সব user-এর (টার্মিনাল আছে এমন) process
- `u` = user-কেন্দ্রিক বিস্তারিত কলাম
- `x` = টার্মিনাল ছাড়া process-ও (daemon)

```
$ ps aux | head -5
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.1 167780 13120 ?        Ss   09:01   0:03 /sbin/init splash
root           2  0.0  0.0      0     0 ?        S    09:01   0:00 [kthreadd]
root           3  0.0  0.0      0     0 ?        S    09:01   0:00 [pool_workqueue_release]
nishat      3310  0.0  0.0  11520  5300 pts/0    Ss   10:05   0:00 bash
```

| কলাম | মানে |
|------|------|
| `USER` | কোন user-এর নামে চলছে |
| `PID` | process ID |
| `%CPU` | process-এর পুরো জীবনকাল ধরে গড় CPU ব্যবহার (তাৎক্ষণিক নয়!) |
| `%MEM` | মোট RAM-এর কত শতাংশ RSS |
| `VSZ` | virtual memory size (KiB), process যত মেমরি "চেয়ে রেখেছে" |
| `RSS` | resident set size (KiB), আসলে RAM-এ যতটুকু আছে |
| `TTY` | কোন টার্মিনাল। `?` মানে টার্মিনাল নেই (daemon) |
| `STAT` | state (উপরের টেবিল দেখো) |
| `START` | কখন শুরু হয়েছে |
| `TIME` | মোট কত CPU সময় খেয়েছে |
| `COMMAND` | কমান্ড। `[ ]` বন্ধনীতে থাকলে সেটা kernel thread |

> 💡 **টিপস:** VSZ বড় দেখে ভয় পেও না। VSZ-এ শেয়ারড লাইব্রেরি আর এখনো ব্যবহার না করা মেমরিও গোনা হয়। আসল RAM ব্যবহারের ভালো ধারণা দেয় RSS।

### `ps -ef`: সব process, UNIX স্টাইল

- `-e` = সব process
- `-f` = full format (PPID সহ)

```
$ ps -ef | head -4
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 09:01 ?        00:00:03 /sbin/init splash
root           2       0  0 09:01 ?        00:00:00 [kthreadd]
root           3       2  0 09:01 ?        00:00:00 [pool_workqueue_release]
```

| কলাম | মানে |
|------|------|
| `UID` | user |
| `PPID` | parent PID |
| `C` | CPU ব্যবহারের হার (পূর্ণসংখ্যা) |
| `STIME` | শুরুর সময় |

`ps -ef`-এ PPID থাকে, `ps aux`-এ থাকে %MEM আর RSS। তাই parent-child সম্পর্ক খুঁজতে `-ef`, মেমরি দেখতে `aux` সুবিধাজনক।

`-F` দিলে আরও বেশি তথ্য (RSS, কোন CPU-তে চলছে = `PSR`), আর `-l` দিলে long format (PRI, NI, WCHAN সহ)।

### `--forest`: গাছ আকারে

```
$ ps -ef --forest | grep -A3 sshd
root         812       1  0 09:01 ?        00:00:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
root        2101     812  0 10:00 ?        00:00:00  \_ sshd: nishat [priv]
nishat      2119    2101  0 10:00 ?        00:00:00      \_ sshd: nishat@pts/1
nishat      2120    2119  0 10:00 pts/1    00:00:00          \_ -bash
```

BSD স্টাইলে: `ps auxf` (শেষে `f` = forest)।

### `-o`: নিজের পছন্দমতো কলাম

```
$ ps -eo pid,ppid,user,stat,ni,%cpu,%mem,rss,etime,cmd --sort=-%mem | head -5
    PID    PPID USER     STAT  NI %CPU %MEM   RSS     ELAPSED CMD
   7123    1720 nishat   Sl     0 12.1  9.8 1598320    02:13:44 /usr/lib/firefox/firefox
   2410    1720 nishat   Ssl    0  3.0  4.1 668012    03:01:10 /usr/bin/gnome-shell
   ...
```

কাজের কিছু কলাম নাম (`ps L` দিলে, ড্যাশ ছাড়া, সব কলাম নামের তালিকা পাবে; `-L` কিন্তু thread দেখানোর অপশন):

| কলাম | মানে |
|------|------|
| `pid`, `ppid`, `pgid`, `sid` | process, parent, process group, session ID |
| `user`, `uid`, `group` | মালিক |
| `stat`, `state` | state (বিস্তারিত / শুধু এক অক্ষর) |
| `ni`, `pri` | nice মান, priority |
| `%cpu`, `%mem`, `rss`, `vsz` | রিসোর্স |
| `etime`, `etimes` | চালু হওয়ার পর কত সময় (ফরম্যাটেড / সেকেন্ডে) |
| `lstart` | শুরুর পুরো তারিখ-সময় |
| `time` | মোট CPU সময় |
| `tty` | টার্মিনাল |
| `comm` | শুধু program নাম |
| `args`, `cmd` | পুরো command line |
| `nlwp` | thread সংখ্যা |
| `wchan` | `S`/`D` state-এ থাকলে kernel-এর কোন ফাংশনে অপেক্ষা করছে |
| `psr` | এখন কোন CPU core-এ |

হেডার বাদ দিতে কলাম নামের পরে `=` দাও। স্ক্রিপ্টে খুব কাজের:

```
$ ps -o pid=,etimes= -p 1
      1      12345
```

### ফিল্টার করা

| কমান্ড | কাজ |
|--------|-----|
| `ps -u nishat` | ওই user-এর process (effective user) |
| `ps -U root -u root` | real ও effective উভয় user root |
| `ps -p 1,2,3310` | নির্দিষ্ট PID গুলো |
| `ps -C nginx` | নির্দিষ্ট command নাম (exact) |
| `ps --ppid 812` | নির্দিষ্ট parent-এর child গুলো |
| `ps -t pts/0` | নির্দিষ্ট টার্মিনালের |
| `ps -eLf` | সব thread সহ |

### সাজানো (sort)

```
$ ps aux --sort=-%cpu | head -6     # সবচেয়ে বেশি CPU খাওয়া ৫টা
$ ps aux --sort=-rss | head -6      # সবচেয়ে বেশি RAM খাওয়া ৫টা
$ ps -eo pid,etime,cmd --sort=start_time | tail -5   # সবচেয়ে নতুন ৫টা
```

`-` মানে বড় থেকে ছোট, `+` বা কিছু না দিলে ছোট থেকে বড়।

### `ps | grep`-এর পুরোনো ফাঁদ

```
$ ps aux | grep nginx
root     1010  0.0  0.1  55200  1500 ?   Ss  09:01  0:00 nginx: master process /usr/sbin/nginx
www-data 1011  0.0  0.2  55900  5200 ?   S   09:01  0:00 nginx: worker process
nishat   3610  0.0  0.0   9200  2300 pts/0 S+ 10:30  0:00 grep --color=auto nginx
```

শেষ লাইনে `grep` নিজেই চলে এসেছে। সমাধান: `ps aux | grep '[n]ginx'` (regex `[n]ginx` "nginx"-এর সাথে মেলে, কিন্তু grep-এর নিজের command line-এ থাকা `[n]ginx` লেখার সাথে মেলে না)। আরও ভালো সমাধান হলো `pgrep` (নিচে দেখো)।

---

## ২৪.৮ top: লাইভ মনিটর

`ps` একটা ছবি, আর `top` হলো লাইভ ভিডিও। ডিফল্টে প্রতি ৩ সেকেন্ডে আপডেট হয়। এটা `procps` (Fedora/RHEL-এ `procps-ng`) প্যাকেজের অংশ, সব distro-তে থাকে।

```
$ top
top - 10:42:15 up  1:41,  2 users,  load average: 0.52, 0.61, 0.58
Tasks: 312 total,   1 running, 310 sleeping,   0 stopped,   1 zombie
%Cpu(s):  4.2 us,  1.3 sy,  0.0 ni, 93.9 id,  0.4 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem :  15876.4 total,   6120.3 free,   5210.8 used,   4895.2 buff/cache
MiB Swap:   4096.0 total,   4096.0 free,      0.0 used.  10665.6 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   7123 nishat    20   0   11.2g   1.5g 210340 S  12.3   9.8  15:01.22 firefox
   2410 nishat    20   0 5512340 668012 120400 S   3.0   4.1   4:12.10 gnome-shell
      1 root      20   0  167780  13120   8400 S   0.0   0.1   0:03.11 systemd
```

### হেডার অংশ লাইন ধরে ধরে

**লাইন ১:** বর্তমান সময়, কতক্ষণ চালু (`up 1:41`), কতজন user লগইন, আর **load average** (নিচে ২৪.১৪-এ বিস্তারিত)।

**লাইন ২ (Tasks):** মোট, running, sleeping, stopped, zombie সংখ্যা।

**লাইন ৩ (%Cpu(s)):** CPU সময় কোথায় যাচ্ছে:

| ঘর | মানে |
|----|------|
| `us` | user space-এ সাধারণ প্রোগ্রাম |
| `sy` | kernel (system) কাজে |
| `ni` | nice বাড়ানো (কম priority) user প্রোগ্রামে |
| `id` | অলস (idle) |
| `wa` | I/O wait, CPU অলস কিন্তু ডিস্কের অপেক্ষায় আছে এমন কাজ আছে |
| `hi` | hardware interrupt |
| `si` | software interrupt |
| `st` | steal: VM হলে hypervisor অন্য VM-কে যে সময় দিয়েছে |

> 💡 **টিপস:** `wa` বেশি হলে সমস্যা ডিস্কে, CPU-তে না। `st` বেশি হলে তোমার cloud VM-এর host overloaded।

**লাইন ৪–৫ (Mem/Swap):** লিনাক্স ফাঁকা RAM নষ্ট না করে ফাইল cache হিসেবে ব্যবহার করে (`buff/cache`)। তাই `free` কম দেখলে ঘাবড়াবে না। আসলে কত মেমরি নতুন প্রোগ্রামের জন্য পাওয়া যাবে, সেটা বলে **`avail Mem`**।

### process টেবিলের কলাম

| কলাম | মানে |
|------|------|
| `PR` | kernel-এর চোখে priority। সাধারণ process-এ `20 + NI`, real-time হলে `rt` বা ঋণাত্মক সংখ্যা |
| `NI` | nice মান (-20 থেকে 19) |
| `VIRT` | virtual memory (ps-এর VSZ) |
| `RES` | RAM-এ বাস্তবে আছে (ps-এর RSS) |
| `SHR` | RES-এর মধ্যে শেয়ারড অংশ |
| `S` | state |
| `%CPU` | **শেষ রিফ্রেশের পর** CPU ব্যবহার। মাল্টি-কোরে ১০০%-এর বেশি হতে পারে (৪ core পুরো খেলে ৪০০%) |
| `TIME+` | মোট CPU সময়, শতাংশ-সেকেন্ড পর্যন্ত |

> 📝 **মনে রাখো:** `ps`-এর `%CPU` হলো জীবনভর গড়, `top`-এর `%CPU` হলো এই মুহূর্তের। তাই এখন কে CPU খাচ্ছে জানতে `top` বা `htop` দেখো।

### top-এর interactive শর্টকাট (চলমান অবস্থায়)

top চলার সময় এই কী-গুলো চাপো (case-sensitive: `P` আর `p` আলাদা):

| কী | কাজ |
|----|-----|
| `h` বা `?` | help স্ক্রিন |
| `q` | বের হও |
| `Space` বা `Enter` | এখনই রিফ্রেশ |
| `d` বা `s` | রিফ্রেশ বিরতি বদলাও (যেমন `1` সেকেন্ড) |
| `P` | %CPU অনুযায়ী সাজাও (ডিফল্ট) |
| `M` | %MEM অনুযায়ী সাজাও |
| `T` | TIME+ অনুযায়ী সাজাও |
| `N` | PID অনুযায়ী সাজাও |
| `R` | সাজানোর ক্রম উল্টে দাও |
| `<` / `>` | sort কলাম বাঁয়ে / ডানে সরাও |
| `x` | sort কলাম highlight |
| `y` | running task highlight |
| `b` | bold/reverse highlight টগল (x/y-এর সাথে) |
| `z` | রঙিন মোড টগল |
| `Z` | রঙ কাস্টমাইজ |
| `1` | প্রতিটা CPU core আলাদা লাইনে দেখাও |
| `t` | CPU সারাংশের ধরন বদলাও (bar/লেখা/লুকানো) |
| `m` | মেমরি সারাংশের ধরন বদলাও |
| `l` | load average লাইন টগল |
| `E` | হেডারের মেমরি একক বদলাও (KiB→MiB→GiB...) |
| `e` | টেবিলের মেমরি একক বদলাও |
| `c` | শুধু নাম ↔ পুরো command line টগল |
| `V` | forest (গাছ) ভিউ |
| `H` | process ↔ thread ভিউ |
| `i` | idle (ঘুমন্ত) process লুকাও/দেখাও |
| `u` বা `U` | নির্দিষ্ট user-এর process দেখাও |
| `o` / `O` | কলামে ফিল্টার, যেমন `COMMAND=nginx` বা `%CPU>5.0` |
| `Ctrl+O` | বর্তমান ফিল্টার দেখাও |
| `=` | ফিল্টার মুছে দাও |
| `L` | লেখা খুঁজে বের করো (locate) |
| `&` | পরের মিল খোঁজো |
| `n` বা `#` | কয়টা process দেখাবে তা সীমিত করো |
| `f` বা `F` | fields manager: কলাম যোগ/বাদ/সাজানো |
| `k` | process-কে signal পাঠাও (PID চাইবে, তারপর signal, ডিফল্ট 15) |
| `r` | renice (PID, তারপর নতুন nice মান) |
| `W` | বর্তমান সেটিং `~/.config/procps/toprc`-এ সেভ করো |
| `A` | alternate display (৪টা উইন্ডো) |

### top-এর কমান্ড-লাইন অপশন

| অপশন | কাজ |
|------|-----|
| `-d 1` | ১ সেকেন্ড পর পর রিফ্রেশ |
| `-n 5` | ৫ বার রিফ্রেশ করে বেরিয়ে যাও |
| `-b` | batch mode: স্ক্রিনে না সাজিয়ে লেখা হিসেবে আউটপুট, ফাইলে সেভ বা স্ক্রিপ্টের জন্য |
| `-p PID[,PID]` | শুধু নির্দিষ্ট process |
| `-u user` | শুধু নির্দিষ্ট user |
| `-o FIELD` | ওই কলাম দিয়ে সাজাও, যেমন `-o %MEM` |
| `-H` | thread মোডে শুরু |
| `-c` | পুরো command line দেখিয়ে শুরু |

```
$ top -b -n 1 -o %MEM | head -15 > snapshot.txt    # মেমরির একটা স্ন্যাপশট ফাইলে
$ top -p $(pgrep -d, nginx)                       # শুধু nginx-এর process গুলো
```

---

## ২৪.৯ htop: রঙিন, সহজ top

`htop` হলো top-এর আধুনিক, বন্ধুসুলভ সংস্করণ: রঙিন CPU/মেমরি বার, মাউস চলে, স্ক্রল করা যায়, গাছ দেখায়। ডিফল্টে সবসময় ইনস্টল থাকে না:

```
$ sudo apt install htop      # Debian/Ubuntu
$ sudo dnf install htop      # Fedora; RHEL 9-এ EPEL রিপো লাগে
```

```
  0[|||||               12.4%]   Tasks: 148, 612 thr, 210 kthr; 1 running
  1[||||                 9.1%]   Load average: 0.52 0.61 0.58
  2[|||||||             18.0%]   Uptime: 01:41:22
  3[||                   4.0%]
  Mem[|||||||||||||||5.09G/15.5G]
  Swp[                    0K/4.00G]

    PID USER       PRI  NI  VIRT   RES   SHR S CPU% MEM%   TIME+  Command
   7123 nishat      20   0 11.2G 1.52G  205M S 12.3  9.8 15:01.22 /usr/lib/firefox/firefox
 F1Help F2Setup F3Search F4Filter F5Tree F6SortBy F7Nice- F8Nice+ F9Kill F10Quit
```

বারের রঙের মানে (ডিফল্ট থিমে): CPU বারে সবুজ = সাধারণ user process, লাল = kernel, নীল = কম priority (nice)। মেমরি বারে সবুজ = ব্যবহৃত, নীল = buffer, হলুদ/কমলা = cache। সঠিক রঙ-তালিকা `F1` চাপলে দেখা যায়।

### htop শর্টকাট

| কী | কাজ |
|----|-----|
| `F1` বা `h` বা `?` | help |
| `F2` বা `S` | setup: meter, কলাম, রঙ বদলাও |
| `F3` বা `/` | নাম খোঁজো (হাইলাইট করে) |
| `F4` বা `\` | ফিল্টার: শুধু মিলে যাওয়া process দেখাও |
| `F5` বা `t` | tree view টগল |
| `F6` বা `<` `>` | কোন কলাম দিয়ে সাজাবে বাছাই |
| `F7` বা `]` | nice কমাও = priority বাড়াও (root লাগে) |
| `F8` বা `[` | nice বাড়াও = priority কমাও |
| `F9` বা `k` | নির্বাচিত process-কে signal পাঠাও (তালিকা থেকে বাছো) |
| `F10` বা `q` | বের হও |
| `↑` `↓` `PgUp` `PgDn` `Home` `End` | তালিকায় চলাফেরা |
| `Space` | process ট্যাগ করো (একসাথে অনেকগুলোকে kill করতে) |
| `c` | process আর তার সব child ট্যাগ করো |
| `U` | সব ট্যাগ মুছে দাও |
| `u` | নির্দিষ্ট user-এর process দেখাও |
| `P` | CPU% অনুযায়ী সাজাও |
| `M` | MEM% অনুযায়ী সাজাও |
| `T` | TIME অনুযায়ী সাজাও |
| `I` | সাজানোর ক্রম উল্টাও |
| `+` / `-` (বা `*`) | tree view-তে শাখা খোলো/বন্ধ করো |
| `H` | user thread দেখাও/লুকাও |
| `K` | kernel thread দেখাও/লুকাও |
| `p` | program-এর পুরো path দেখাও/লুকাও |
| `F` | নির্বাচিত process-কে "follow" করো (সাজানো বদলালেও cursor তার সাথে থাকে) |
| `Z` (Shift+z) | আপডেট pause/resume |
| `Ctrl+L` | স্ক্রিন রিফ্রেশ |
| `e` | নির্বাচিত process-এর environment variable দেখাও |
| `l` | `lsof` দিয়ে খোলা ফাইল দেখাও (lsof ইনস্টল থাকতে হবে) |
| `s` | `strace` দিয়ে system call দেখাও (strace লাগবে) |
| `a` | CPU affinity সেট করো (কোন core-এ চলবে) |
| `w` | পুরো command line একাধিক লাইনে দেখাও |
| `Tab` | (htop 3.x) স্ক্রিন ট্যাব বদলাও, যেমন I/O ট্যাব |

কমান্ড-লাইন অপশন: `htop -u nishat`, `htop -p 1234,5678`, `htop -t` (tree দিয়ে শুরু), `htop -d 10` (রিফ্রেশ বিরতি, **এক-দশমাংশ সেকেন্ড** এককে, তাই `-d 10` = ১ সেকেন্ড), `htop -s PERCENT_MEM` (ওই কলামে সাজিয়ে শুরু)।

> 💡 **টিপস:** আরও আধুনিক বিকল্প: `btop` (খুব সুন্দর গ্রাফ), `glances` (CPU, ডিস্ক, নেটওয়ার্ক সব এক স্ক্রিনে), `atop` (ইতিহাস রেকর্ড করে)। পারফরম্যান্স বিশ্লেষণ বিস্তারিত [অধ্যায় ৪৬](46-performance-troubleshooting.md)-এ।

---

## ২৪.১০ pgrep, pkill, pidof: নাম দিয়ে process খোঁজা

`ps | grep | awk` দিয়ে PID বের করার বদলে এই তিনটা কমান্ড সরাসরি কাজ করে। `pgrep`/`pkill` আছে `procps` প্যাকেজে, `pidof` আছে Debian 12-এ `sysvinit-utils`-এ, আর Fedora/RHEL ও নতুন Debian/Ubuntu-তে `procps`/`procps-ng`-এ। সব distro-তে ডিফল্টে থাকে।

### pgrep

```
$ pgrep sshd
812
2101
2119

$ pgrep -l sshd          # PID + নাম
812 sshd
2101 sshd
2119 sshd

$ pgrep -a nginx         # PID + পুরো command line
1010 nginx: master process /usr/sbin/nginx
1011 nginx: worker process
```

| অপশন | কাজ |
|------|-----|
| `-l` | নামসহ দেখাও |
| `-a` | পুরো command line দেখাও |
| `-f` | শুধু নাম না, পুরো command line-এ খোঁজো (যেমন `pgrep -f "python3 app.py"`) |
| `-x` | নাম হুবহু মিলতে হবে (exact) |
| `-u user` | নির্দিষ্ট (effective) user-এর |
| `-U user` | নির্দিষ্ট real user-এর |
| `-P PPID` | নির্দিষ্ট parent-এর child |
| `-t pts/0` | নির্দিষ্ট টার্মিনালের |
| `-n` | সবচেয়ে নতুনটা |
| `-o` | সবচেয়ে পুরোনোটা |
| `-c` | শুধু সংখ্যা গোনো |
| `-d ,` | আউটপুটের মাঝে delimiter (ডিফল্ট newline) |
| `-v` | উল্টো, যারা মেলে না |
| `-i` | case-insensitive |

> 📝 **মনে রাখো:** pattern একটা regex, আর ডিফল্টে শুধু process-এর নাম (`comm`, সর্বোচ্চ ১৫ অক্ষর) মেলায়। নাম ১৫ অক্ষরের বেশি হলে কেটে যায়, তখন `-f` লাগে। আবার `pgrep ssh` লিখলে `sshd`, `ssh-agent` সবই মিলবে। নির্দিষ্টটা চাইলে `-x` দাও।

### pkill

pgrep-এর মতো খোঁজে, কিন্তু পেলে signal পাঠায় (ডিফল্ট SIGTERM)। অপশন প্রায় একই।

```
$ pkill -x firefox                 # firefox-কে SIGTERM
$ pkill -HUP -x nginx              # nginx-কে SIGHUP (config reload)
$ pkill -u guest                   # guest user-এর সব process
$ pkill -9 -f "python3 worker.py"  # command line মিলিয়ে জোর করে মারো
$ pkill -e sleep                   # -e: কাকে মারল তা প্রিন্ট করো
sleep killed (pid 6001)
```

> ⚠️ **সতর্কতা:** `pkill` আগে `pgrep` দিয়ে পরীক্ষা করে নাও। `pkill -f python` দিলে সিস্টেমের সব python process মরবে, এমনকি দরকারি সার্ভিসও। প্রথমে `pgrep -af python` চালিয়ে দেখো কারা মিলছে।

### pidof

শুধু program-এর হুবহু নাম দিয়ে PID দেয়। স্ক্রিপ্টে খুব সহজ:

```
$ pidof systemd
1720 1
$ pidof -s nginx       # -s: শুধু একটা PID
1011
$ pidof -x backup.sh   # -x: শেল স্ক্রিপ্টও খোঁজো
4410
```

---

## ২৪.১১ signals: process-কে বার্তা পাঠানো

**signal** হলো kernel বা অন্য process থেকে একটা process-কে পাঠানো ছোট্ট সংকেত, একটা নম্বর মাত্র। process তিনভাবে সাড়া দিতে পারে:

1. **ডিফল্ট কাজ** (বেশিরভাগ ক্ষেত্রে মরে যাওয়া, কোনোটায় core dump, কোনোটায় থেমে যাওয়া)
2. **ignore** (উপেক্ষা)
3. **catch**: নিজের handler ফাংশন চালানো (যেমন ফাইল সেভ করে পরিষ্কারভাবে বের হওয়া)

**উপমা:** signal হলো দরজায় টোকা। SIGTERM মানে ভদ্রভাবে বলা "দয়া করে বেরিয়ে যান", মানুষটা জিনিসপত্র গুছিয়ে বের হয়। SIGKILL মানে পুলিশ এসে সরাসরি তুলে নিয়ে গেল, গোছানোর সুযোগ নেই।

```
  sender (kill / kernel / কীবোর্ড)
        │  kill(PID, SIGTERM)
        ▼
  ┌──────────┐    kernel signal-টা target-এর
  │  kernel  │    "pending" তালিকায় রাখে
  └────┬─────┘
       │ process পরের বার CPU পেলে / ঘুম থেকে জাগলে
       ▼
  ┌──────────────── target process ───────────────┐
  │ handler আছে?  ── হ্যাঁ ─► handler চালাও         │
  │      │                                         │
  │      না ─► ignore করা? ── হ্যাঁ ─► কিছুই না     │
  │                 │                              │
  │                 না ─► ডিফল্ট কাজ (মরা/থামা...)  │
  └────────────────────────────────────────────────┘
  ব্যতিক্রম: SIGKILL ও SIGSTOP catch বা ignore করা যায় না,
             kernel নিজেই সরাসরি কার্যকর করে।
```

### গুরুত্বপূর্ণ signal টেবিল

(নম্বরগুলো x86 ও ARM লিনাক্সের। অন্য architecture-এ কিছু নম্বর আলাদা হতে পারে, তাই স্ক্রিপ্টে নম্বরের বদলে নাম লেখা নিরাপদ।)

| নম্বর | নাম | ডিফল্ট কাজ | কখন/কেন |
|-------|-----|-----------|---------|
| 1 | `SIGHUP` | মরে যাওয়া | "hang up": টার্মিনাল বন্ধ হলে। daemon-রা প্রথা অনুযায়ী এটা পেলে config reload করে (nginx, sshd) |
| 2 | `SIGINT` | মরে যাওয়া | কীবোর্ডে `Ctrl+C` |
| 3 | `SIGQUIT` | মরে যাওয়া + core dump | কীবোর্ডে `Ctrl+\` |
| 6 | `SIGABRT` | মরে যাওয়া + core dump | প্রোগ্রাম নিজে `abort()` করলে |
| 9 | `SIGKILL` | মরে যাওয়া | জোর করে হত্যা। **catch/ignore করা যায় না** |
| 10 | `SIGUSR1` | মরে যাওয়া | প্রোগ্রাম নিজের মতো অর্থ দেয় (যেমন `dd` এটা পেলে progress দেখায়) |
| 11 | `SIGSEGV` | মরে যাওয়া + core dump | ভুল মেমরি অ্যাক্সেস ("Segmentation fault") |
| 12 | `SIGUSR2` | মরে যাওয়া | প্রোগ্রাম-নির্ধারিত |
| 13 | `SIGPIPE` | মরে যাওয়া | পাইপের অপর প্রান্ত বন্ধ, তবু লেখার চেষ্টা |
| 14 | `SIGALRM` | মরে যাওয়া | টাইমার শেষ |
| 15 | `SIGTERM` | মরে যাওয়া | **ভদ্র অনুরোধ**: `kill`-এর ডিফল্ট। process গুছিয়ে বের হতে পারে |
| 17 | `SIGCHLD` | ignore | child মারা গেলে বা থামলে parent পায় |
| 18 | `SIGCONT` | থেমে থাকলে আবার চালু | `fg`/`bg` এটাই পাঠায় |
| 19 | `SIGSTOP` | থেমে যাওয়া | জোর করে pause। **catch/ignore করা যায় না** |
| 20 | `SIGTSTP` | থেমে যাওয়া | কীবোর্ডে `Ctrl+Z` (ভদ্র stop, catch করা যায়) |
| 28 | `SIGWINCH` | ignore | টার্মিনাল উইন্ডোর মাপ বদলেছে |

সব signal দেখতে:

```
$ kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
...
$ kill -l 9
KILL
$ kill -l TERM
15
```

### signal-এর সাথে জড়িত কীবোর্ড শর্টকাট

| শর্টকাট | কী পাঠায় | ফল |
|---------|----------|-----|
| `Ctrl+C` | SIGINT | foreground process বন্ধ |
| `Ctrl+Z` | SIGTSTP | foreground process থামিয়ে ব্যাকগ্রাউন্ডে রাখে (`fg` দিয়ে ফেরাও) |
| `Ctrl+\` | SIGQUIT | বন্ধ + core dump (Ctrl+C কাজ না করলে চেষ্টা করো) |
| `Ctrl+D` | কোনো signal নয় | ইনপুট শেষ (EOF) জানায়; শেলে দিলে লগআউট |
| `Ctrl+S` / `Ctrl+Q` | কোনো signal নয় | টার্মিনাল আউটপুট থামাও / চালু করো (flow control) |

এই শর্টকাট গুলো আসলে টার্মিনাল ড্রাইভার (tty) signal-এ রূপান্তর করে। `stty -a` দিলে দেখা যায় কোন কী কোন কাজে বাঁধা (`intr = ^C; quit = ^\; susp = ^Z` ইত্যাদি)। job control (`fg`, `bg`, `jobs`) বিস্তারিত [অধ্যায় ২৫](25-jobs-tmux.md)-এ।

> 📝 **মনে রাখো:** একটা process signal-এ মারা গেলে শেলে তার exit code হয় **128 + signal নম্বর**। Ctrl+C-তে মরলে `$?` = 130, `kill -9`-এ মরলে 137। ডকারে "exit code 137" দেখলে বুঝবে SIGKILL (প্রায়ই OOM killer)।

---

## ২৪.১২ kill ও killall

### kill: PID ধরে signal পাঠানো

নাম "kill" হলেও এর আসল কাজ **signal পাঠানো**। ডিফল্ট signal SIGTERM (15)।

```
kill [-signal] PID...
```

signal লেখার সব সমতুল্য রূপ:

```
$ kill 4521              # SIGTERM
$ kill -15 4521          # একই
$ kill -TERM 4521        # একই
$ kill -SIGTERM 4521     # একই
$ kill -s TERM 4521      # একই
```

| ব্যবহার | কাজ |
|---------|-----|
| `kill PID` | ভদ্রভাবে বন্ধ হতে বলো |
| `kill -9 PID` | জোর করে মারো (শেষ উপায়) |
| `kill -HUP PID` | reload (daemon হলে) |
| `kill -STOP PID` | pause করো |
| `kill -CONT PID` | আবার চালু করো |
| `kill -0 PID` | কোনো signal যায় না, শুধু যাচাই: process আছে কি না এবং signal পাঠানোর অনুমতি আছে কি না (exit code 0 = হ্যাঁ) |
| `kill -l` | signal তালিকা |
| `kill -- -PGID` | পুরো process group-কে (ঋণাত্মক সংখ্যা = group) |

```
$ sleep 500 &
[1] 7001
$ kill -0 7001 && echo "চলছে"
চলছে
$ kill 7001
[1]+  Terminated              sleep 500
$ kill -0 7001
bash: kill: (7001) - No such process
```

### সঠিক পদ্ধতি: আগে TERM, পরে KILL

```
1. kill PID            → process কে গুছিয়ে বের হওয়ার সুযোগ দাও
2. কয়েক সেকেন্ড অপেক্ষা (sleep 5)
3. kill -0 PID          → এখনো আছে?
4. থাকলে: kill -9 PID  → শেষ অস্ত্র
```

systemd নিজেও সার্ভিস বন্ধ করার সময় ঠিক এটাই করে: আগে SIGTERM, তারপর ডিফল্টে ৯০ সেকেন্ড অপেক্ষা করে SIGKILL।

> ⚠️ **সতর্কতা:** অভ্যাসবশত `kill -9` দিও না। SIGKILL পেলে process temp ফাইল মুছতে পারে না, ডাটাবেস লেনদেন শেষ করতে পারে না, lock ফাইল রেখে যায়। ফলে ডেটা নষ্ট হতে পারে বা পরের বার প্রোগ্রাম চালু হবে না।

> ⚠️ **সতর্কতা:** `kill -9 -1` মানে "আমার অনুমতি আছে এমন সব process মারো"। সাধারণ user দিলে তার নিজের সব process (ডেস্কটপ সেশন সহ) মরে যাবে, root দিলে প্রায় পুরো সিস্টেম। ভুলেও চালাবে না।

**অনুমতি:** সাধারণ user শুধু নিজের process-কে signal পাঠাতে পারে। অন্যের বা root-এর process-এ `Operation not permitted` আসবে, তখন `sudo kill ...` লাগে ([অধ্যায় ১৮](18-sudo-root.md))।

> 📝 **মনে রাখো:** `kill` দুটো আছে: bash-এর builtin `kill` আর `/usr/bin/kill` (procps-ng/util-linux)। সাধারণত builtin চলে, যেটা `%1` এর মতো job নম্বরও বোঝে (`kill %1`)।

### killall: নাম ধরে মারা

`killall` (`psmisc` প্যাকেজ) process-এর হুবহু নাম দিয়ে signal পাঠায়।

```
$ killall firefox
$ killall -HUP nginx
$ killall -u guest bash           # guest-এর সব bash
$ killall -i sleep                # প্রতিটার আগে জিজ্ঞেস করবে
Kill sleep(7101) ? (y/N) y
$ killall -w myapp                # সব মরা পর্যন্ত অপেক্ষা করো
$ killall -o 2h backup            # ২ ঘণ্টার বেশি পুরোনো backup process
```

| অপশন | কাজ |
|------|-----|
| `-s SIG` বা `-SIG` | কোন signal |
| `-i` | interactive, প্রতিবার নিশ্চিত হও |
| `-u user` | নির্দিষ্ট user-এর |
| `-w` | মরা পর্যন্ত অপেক্ষা |
| `-e` | exact: ১৫ অক্ষরের বেশি লম্বা নাম পুরো মিলাও |
| `-I` | case-insensitive |
| `-r` | নামকে regex হিসেবে ধরো |
| `-o TIME` / `-y TIME` | এর চেয়ে পুরোনো / নতুন (যেমন `30m`, `2h`, `1d`) |
| `-v` | কী হলো তা জানাও |
| `-l` | signal তালিকা |

> ⚠️ **সতর্কতা:** Solaris-এর মতো কিছু পুরোনো Unix-এ `killall` মানে "সব process মারো"! লিনাক্সে নিরাপদ, কিন্তু অন্য Unix সার্ভারে অভ্যাসবশত চালিও না।

### কোনটা কখন

| প্রয়োজন | কমান্ড |
|----------|--------|
| নির্দিষ্ট একটা PID | `kill PID` |
| নামের সব instance | `killall name` বা `pkill -x name` |
| command line-এর অংশ মিলিয়ে | `pkill -f "pattern"` |
| সিস্টেম সার্ভিস | `sudo systemctl stop/restart/reload name` ([অধ্যায় ২৬](26-systemd-services.md)) |
| GUI-তে জমে যাওয়া উইন্ডো | `xkill` (X11-এ), বা System Monitor |

---

## ২৪.১৩ nice, renice ও priority

CPU core সীমিত, process অনেক। কাকে কতটা সময় দেওয়া হবে, তা ঠিক করে kernel-এর **scheduler**। সাধারণ process-এর জন্য লিনাক্স ব্যবহার করে CFS (Completely Fair Scheduler), আর kernel 6.6 থেকে এর উত্তরসূরি **EEVDF** scheduler (Ubuntu 24.04-এর kernel 6.8, তাই সেখানে EEVDF)। দুটোতেই "nice" মান একই ভাবে কাজ করে।

### nice মান কী

- পরিসর: **-20** (সবচেয়ে বেশি priority) থেকে **19** (সবচেয়ে কম)। ডিফল্ট **0**।
- নাম "nice" কারণ মান যত বেশি, process অন্যদের প্রতি তত "ভদ্র", তত বেশি জায়গা ছেড়ে দেয়।
- এটা কঠোর ক্রম না, ওজন (weight)। nice মান ১ বাড়লে মোটামুটি ~১০% কম CPU ভাগ পায়। ফাঁকা সিস্টেমে nice 19 process-ও পুরো CPU পাবে; প্রতিযোগিতা থাকলেই পার্থক্য বোঝা যায়।

**উপমা:** ব্যাংকের লাইন না, বরং একটা পিৎজা ভাগ করা। যার ওজন বেশি, সে বড় টুকরো পায়, কিন্তু কেউ একেবারে না খেয়ে থাকে না।

```
nice:   -20 ◄──────────────── 0 ────────────────► 19
        বেশি CPU ভাগ         ডিফল্ট           কম CPU ভাগ
        (শুধু root)                             (যে কেউ দিতে পারে)

top-এর PR:  0  ...........  20  ...........  39     (PR = 20 + NI)
ps -l এর PRI: 60 .........  80  ...........  99     (ps-এর হিসাব ভিন্ন স্কেলে)
```

| কে কী পারে | |
|-------------|--|
| সাধারণ user | নিজের process-এর nice **বাড়াতে** পারে (priority কমাতে) |
| সাধারণ user | nice **কমাতে** (priority বাড়াতে) পারে না, একবার বাড়ালে ফেরাতেও না (যদি না `/etc/security/limits.conf`-এ `nice` limit দেওয়া থাকে) |
| root | যেকোনো দিকে, -20 পর্যন্ত |

### nice: নতুন process নির্দিষ্ট priority-তে চালানো

```
$ nice                           # বর্তমান nice মান
0
$ nice tar czf backup.tar.gz /home/nishat/docs    # ডিফল্ট +10 বাড়িয়ে চালাবে
$ nice -n 15 ./heavy-report.sh                    # nice 15-এ
$ sudo nice -n -5 ./important-job                 # ঋণাত্মক: root লাগে
$ nice -n -5 ./job
nice: cannot set niceness: Permission denied
```

> 💡 **টিপস:** পুরোনো লেখায় `nice -10 cmd` দেখবে, যার মানে +10 (ড্যাশটা অপশন চিহ্ন, ঋণাত্মক নয়)! আর `nice --10` মানে -10। এই বিভ্রান্তি এড়াতে সবসময় `-n` লেখো।

### renice: চলমান process-এর priority বদলানো

```
$ renice -n 10 -p 7123
7123 (process ID) old priority 0, new priority 10

$ sudo renice -n -5 -p 1010
1010 (process ID) old priority 0, new priority -5

$ sudo renice -n 5 -u guest          # guest-এর সব process
$ renice -n 19 -g 4500               # পুরো process group
```

| অপশন | কাজ |
|------|-----|
| `-n N` | নতুন nice মান (নতুন util-linux ভার্সনে `--relative` যোগ করলে মানটা বর্তমান nice-এর সাথে যোগ হয়; নিশ্চিত হতে `man renice` দেখো) |
| `-p PID` | process ধরে (ডিফল্ট) |
| `-u user` | user-এর সব process |
| `-g PGID` | process group ধরে |

`top`-এ `r` আর `htop`-এ `F7`/`F8` দিয়েও একই কাজ করা যায়।

### nice-এর বাইরে: real-time ও I/O priority

- **`chrt`**: real-time scheduling policy (SCHED_FIFO, SCHED_RR) দেয়। এরা সব সাধারণ (nice) process-এর আগে চলে। `chrt -p PID` দিয়ে দেখো, `sudo chrt -f 50 cmd` দিয়ে চালাও। ভুল ব্যবহারে সিস্টেম জমে যেতে পারে, সাবধান।
- **`ionice`**: ডিস্ক I/O-এর priority। `ionice -c 3 cmd` (idle class: ডিস্ক ফাঁকা থাকলেই শুধু I/O করবে)। ব্যাকআপের জন্য চমৎকার: `nice -n 19 ionice -c 3 rsync ...`।
- systemd সার্ভিসে `Nice=`, `CPUWeight=`, `IOWeight=` সেটিং দিয়ে আরও আধুনিক উপায়ে (cgroups) নিয়ন্ত্রণ করা যায় ([অধ্যায় ২৬](26-systemd-services.md), [অধ্যায় ৪৮](48-containers-virtualization.md))।

---

## ২৪.১৪ /proc/PID: process-এর ভেতরে উঁকি

`/proc` হলো একটা **virtual filesystem**: ডিস্কে নেই, kernel মেমরি থেকে তাৎক্ষণিক বানিয়ে দেয়। প্রতিটা process-এর জন্য একটা ফোল্ডার `/proc/<PID>`। `ps`, `top`, `htop` সবাই আসলে এখান থেকেই পড়ে। (`/proc` আর `/sys` পুরো বিস্তারিত [অধ্যায় ৪৭](47-kernel-proc-sys.md)-এ।)

```
$ ls /proc/$$
attr       cmdline  environ  limits     mounts      oom_score_adj  stat     task
cgroup     comm     exe      loginuid   ns          root           statm    wchan
...        cwd      fd       maps       oom_score   sched          status   ...
```

`/proc/self` সবসময় যে process পড়ছে, তার নিজের ফোল্ডারে নির্দেশ করে।

| ফাইল/ফোল্ডার | কী আছে |
|--------------|--------|
| `cmdline` | পুরো command line, arguments গুলো NUL (`\0`) দিয়ে আলাদা |
| `comm` | program-এর ছোট নাম (১৫ অক্ষর পর্যন্ত) |
| `status` | মানুষের পড়ার মতো সারাংশ: নাম, State, PPid, Uid, Threads, VmRSS... |
| `stat` | একই তথ্য এক লাইনে, স্ক্রিপ্ট/ps-এর জন্য |
| `statm` | মেমরির সংক্ষিপ্ত হিসাব (page এককে) |
| `exe` | আসল executable-এর symlink |
| `cwd` | বর্তমান ডিরেক্টরির symlink |
| `root` | process-এর root ডিরেক্টরি (chroot/কন্টেইনারে আলাদা) |
| `environ` | environment variable (NUL দিয়ে আলাদা) |
| `fd/` | খোলা প্রতিটা file descriptor-এর symlink (0=stdin, 1=stdout, 2=stderr) |
| `maps` | মেমরি ম্যাপ: কোন লাইব্রেরি কোথায় লোড |
| `smaps`, `smaps_rollup` | বিস্তারিত মেমরি (PSS, swap ইত্যাদি) |
| `limits` | resource limit (`ulimit` এর মান) |
| `io` | কত বাইট পড়েছে/লিখেছে |
| `task/` | প্রতিটা thread-এর জন্য একটা উপ-ফোল্ডার |
| `cgroup` | কোন cgroup-এ (systemd সার্ভিস/কন্টেইনার) আছে |
| `ns/` | namespace গুলো (কন্টেইনারের ভিত্তি) |
| `oom_score` | মেমরি ফুরালে OOM killer একে মারার সম্ভাবনা (বেশি = আগে মরবে) |
| `oom_score_adj` | সেটা সমন্বয় করা যায় (-1000 থেকে 1000) |
| `wchan` | ঘুমিয়ে থাকলে kernel-এর কোন ফাংশনে অপেক্ষা করছে |

### কিছু কাজের উদাহরণ

```
$ grep -E '^(Name|State|PPid|Uid|Threads|VmRSS)' /proc/$$/status
Name:	bash
State:	S (sleeping)
PPid:	3295
Uid:	1000	1000	1000	1000
VmRSS:	    5300 kB
Threads:	1

$ tr '\0' ' ' < /proc/1010/cmdline; echo
nginx: master process /usr/sbin/nginx

$ ls -l /proc/$$/cwd /proc/$$/exe
lrwxrwxrwx 1 nishat nishat 0 ... /proc/3310/cwd -> /home/nishat
lrwxrwxrwx 1 nishat nishat 0 ... /proc/3310/exe -> /usr/bin/bash

$ ls -l /proc/$$/fd
lrwx------ 1 nishat nishat 64 ... 0 -> /dev/pts/0
lrwx------ 1 nishat nishat 64 ... 1 -> /dev/pts/0
lrwx------ 1 nishat nishat 64 ... 2 -> /dev/pts/0
lrwx------ 1 nishat nishat 64 ... 255 -> /dev/pts/0

$ ls /proc/$(pidof -s firefox)/task | wc -l     # firefox-এর thread সংখ্যা
87
```

> 💡 **টিপস (জীবন বাঁচানো কৌশল):** ভুল করে একটা ফাইল মুছে ফেলেছ, কিন্তু কোনো process সেটা এখনো খুলে রেখেছে? ফাইলটা আসলে এখনো ডিস্কে আছে! `ls -l /proc/PID/fd` দিয়ে `(deleted)` লেখা লাইনটা খোঁজো, তারপর `cp /proc/PID/fd/4 ~/recovered.log` দিয়ে ফেরত আনো। (কারণ inode-এর link count শূন্য হলেও খোলা reference থাকলে ডেটা মোছা হয় না, দেখো [অধ্যায় ১১](11-links-inode.md))

> ⚠️ **সতর্কতা:** অন্য user-এর process-এর `environ`, `fd`, `maps` ইত্যাদি সাধারণ user পড়তে পারে না, এগুলোতে পাসওয়ার্ড বা টোকেন থাকতে পারে। root লাগে।

---

## ২৪.১৫ load average: সিস্টেম কতটা ব্যস্ত

`uptime`, `top`, `w` সবখানে তিনটা সংখ্যা দেখায়:

```
$ uptime
 10:42:15 up  1:41,  2 users,  load average: 0.52, 0.61, 0.58

$ cat /proc/loadavg
0.52 0.61 0.58 2/1310 7342
```

### সংখ্যাগুলো কী

তিনটা সংখ্যা হলো গত **১ মিনিট, ৫ মিনিট, ১৫ মিনিটের** গড়। কিসের গড়? এমন task-এর সংখ্যার যারা:

- **R** state-এ (চলছে বা CPU-র লাইনে দাঁড়িয়ে), **অথবা**
- **D** state-এ (uninterruptible, সাধারণত disk I/O-র অপেক্ষায়)

লিনাক্সে D state-ও গোনা হয়, তাই load average শুধু CPU চাপ না, I/O চাপও দেখায়। এটা লিনাক্সের নিজস্ব বৈশিষ্ট্য (অন্য অনেক Unix শুধু CPU গোনে)। গড়গুলো সাধারণ গড় না, exponentially damped (সাম্প্রতিক মানের ওজন বেশি)।

`/proc/loadavg`-এর চতুর্থ ঘর `2/1310` = এই মুহূর্তে runnable task / মোট task (thread সহ), আর পঞ্চম ঘর `7342` = সবশেষে তৈরি হওয়া PID।

### কীভাবে পড়বে: CPU সংখ্যার সাথে তুলনা

**উপমা:** একটা ব্রিজে লেন আছে (CPU core)। load হলো ব্রিজে বা ব্রিজের অপেক্ষায় থাকা গাড়ির সংখ্যা।

```
৪ core সিস্টেম (৪ লেনের ব্রিজ):

load 2.0  ║🚗 🚗 ·  · ║   ৫০% ব্যস্ত, আরামদায়ক
load 4.0  ║🚗 🚗 🚗 🚗║   পুরো ব্যবহার, কেউ অপেক্ষায় নেই
load 8.0  ║🚗 🚗 🚗 🚗║ 🚗🚗🚗🚗 ← ৪টা লাইনে দাঁড়িয়ে, দ্বিগুণ চাপ
```

```
$ nproc
4
```

| load / core সংখ্যা | অর্থ |
|---------------------|------|
| < 0.7 | স্বস্তিতে আছে |
| প্রায় 1.0 | পুরো ব্যবহার হচ্ছে, নজর রাখো |
| > 1.0 নিয়মিত | task অপেক্ষা করছে, সিস্টেম ধীর লাগবে |
| অনেক বেশি কিন্তু CPU `id` বেশি | D state-এর জন্য: ডিস্ক/NFS সমস্যা খোঁজো (`top`-এ `wa`, `ps`-এ `D`) |

### তিনটা সংখ্যা থেকে প্রবণতা (trend)

| উদাহরণ | ব্যাখ্যা |
|--------|---------|
| `8.10, 3.20, 1.05` | চাপ সবে বাড়ছে (১ মিনিট > ১৫ মিনিট) |
| `0.40, 2.50, 6.80` | চাপ কমে আসছে |
| `4.00, 4.02, 3.98` | স্থির চাপ |

উচ্চ load পেলে D state-এর process খুঁজতে:

```
$ ps -eo pid,stat,wchan:30,cmd | awk '$2 ~ /^D/'
```

> 📝 **মনে রাখো:** load average একটা "সংখ্যা", শতাংশ না। ১৬ core সার্ভারে load 10 মোটেও সমস্যা না, কিন্তু ১ core VM-এ load 10 মানে ভয়াবহ অবস্থা। সবসময় `nproc`-এর সাথে মিলিয়ে দেখো। আধুনিক kernel-এ আরও নিখুঁত মাপের জন্য PSI (Pressure Stall Information) আছে: `cat /proc/pressure/cpu`, বিস্তারিত [অধ্যায় ৪৬](46-performance-troubleshooting.md)-এ।

---

## ২৪.১৬ আরও কিছু কাজের টুল (সংক্ষেপে)

| কমান্ড | কাজ |
|--------|-----|
| `pmap PID` | process-এর মেমরি ম্যাপ সুন্দর করে |
| `lsof -p PID` | process কোন কোন ফাইল/সকেট খুলে রেখেছে |
| `fuser -v /path` | কোন process এই ফাইল/ডিরেক্টরি ব্যবহার করছে (umount না হলে কাজের) |
| `strace -p PID` | process কী system call করছে লাইভ দেখো |
| `timeout 10s cmd` | ১০ সেকেন্ড পর cmd-কে SIGTERM দাও |
| `watch -n 2 'ps ...'` | প্রতি ২ সেকেন্ডে কমান্ড আবার চালাও |
| `free -h` | মেমরির সারাংশ |
| `vmstat 1` | প্রতি সেকেন্ডে CPU, মেমরি, run queue (`r`) ও blocked (`b`) task |
| `systemd-cgls` | systemd-এর cgroup অনুযায়ী process গাছ |
| `loginctl` / `who` | কে লগইন করে আছে |

---

## হাতে-কলমে অনুশীলন

একটা টার্মিনাল খোলো (দুটো ট্যাব হলে আরও ভালো)। কোনো ধাপেই সিস্টেমের ক্ষতি হবে না।

### ল্যাব ১: নিজের শেলের পরিচয়

```
$ echo "আমার PID: $$, parent: $PPID"
$ ps -o pid,ppid,stat,cmd -p $$
$ pstree -s -p $$
```

পর্যবেক্ষণ: তোমার শেল থেকে PID 1 পর্যন্ত পুরো বংশধারা দেখা গেল।

### ল্যাব ২: fork দেখো

```
$ bash               # নতুন একটা শেল (child) চালাও
$ echo $$ $PPID      # PPID = আগের শেলের PID
$ ps --forest -o pid,ppid,cmd
$ exit               # আগের শেলে ফেরো
```

### ল্যাব ৩: state পরিবর্তন নিজের চোখে

```
$ sleep 1000 &
[1] 8001
$ ps -o pid,stat,cmd -p 8001          # S (ঘুমাচ্ছে)
$ kill -STOP 8001
$ ps -o pid,stat,cmd -p 8001          # T (থেমে আছে)
$ kill -CONT 8001
$ ps -o pid,stat,cmd -p 8001          # আবার S
$ kill 8001                            # SIGTERM
```

### ল্যাব ৪: একটা zombie বানাও ও সরাও

```
$ bash -c 'sleep 1 & exec sleep 120' &
[1] 8100
```

এখানে bash একটা `sleep 1` child বানিয়ে নিজেকে `exec` দিয়ে `sleep 120`-এ বদলে ফেলে। নতুন `sleep 120` জানে না যে তার একটা child আছে, তাই wait() করে না। ১ সেকেন্ড পর:

```
$ ps -o pid,ppid,stat,cmd --ppid 8100
    PID    PPID STAT CMD
   8101    8100 Z    [sleep] <defunct>
$ kill -9 8101                  # কিছুই হবে না, zombie আগেই মৃত
$ ps -o pid,stat -p 8101        # এখনো Z
$ kill 8100                     # parent মারো
$ ps -p 8101                    # zombie নেই, PID 1/subreaper সাফ করে দিয়েছে
```

### ল্যাব ৫: SIGTERM বনাম SIGKILL

একটা স্ক্রিপ্ট বানাও যেটা SIGTERM catch করে (`trap` সম্পর্কে বিস্তারিত [অধ্যায় ৪৩](43-shell-scripting-3.md)):

```
$ cat > ~/graceful.sh << 'EOF'
#!/bin/bash
trap 'echo "SIGTERM পেয়েছি, গুছিয়ে বের হচ্ছি..."; exit 0' TERM
echo "আমার PID: $$"
while true; do sleep 1; done
EOF
$ chmod +x ~/graceful.sh
$ ~/graceful.sh &
আমার PID: 8200
$ kill 8200
SIGTERM পেয়েছি, গুছিয়ে বের হচ্ছি...
$ ~/graceful.sh &
আমার PID: 8210
$ kill -9 8210                  # কোনো বার্তা নেই, সরাসরি মৃত্যু
[1]+  Killed                  ~/graceful.sh
```

(trap থাকা অবস্থায় bash ঘুমন্ত `sleep 1` শেষ হওয়ার পরেই handler চালায়, তাই বার্তা আসতে ১ সেকেন্ড পর্যন্ত লাগতে পারে।)

### ল্যাব ৬: priority পরীক্ষা

দুটো CPU-খাদক চালাও, একই core-এ বেঁধে (`taskset -c 0`), একটার nice 19:

```
$ taskset -c 0 sh -c 'while :; do :; done' &
$ taskset -c 0 nice -n 19 sh -c 'while :; do :; done' &
$ top -o %CPU          # প্রথমটা ~৯৫%+, দ্বিতীয়টা কয়েক % মাত্র
$ pkill -f 'while :; do :; done'      # পরিষ্কার করো
```

### ল্যাব ৭: top ও htop চর্চা

1. `top` চালাও। `1` চাপো (প্রতিটা core), `M` (মেমরি দিয়ে সাজাও), `c` (পুরো command), `V` (গাছ), `u` লিখে নিজের user, `q`।
2. `htop` চালাও। `F5` (গাছ), `F4` দিয়ে "bash" ফিল্টার, `F6` দিয়ে MEM% সাজাও, একটা `sleep` process-এ `F9` দিয়ে SIGTERM পাঠাও।

### ল্যাব ৮: /proc অন্বেষণ

```
$ sleep 600 &
$ P=$!
$ cat /proc/$P/status | head -10
$ tr '\0' ' ' < /proc/$P/cmdline; echo
$ ls -l /proc/$P/cwd /proc/$P/exe
$ cat /proc/loadavg
$ kill $P
```

---

## সারসংক্ষেপ

| কমান্ড/শর্টকাট | কাজ |
|-----------------|-----|
| `echo $$` / `$PPID` / `$!` | নিজের PID / parent / শেষ background PID |
| `ps` | বর্তমান টার্মিনালের process |
| `ps aux` | সব process, CPU/MEM সহ |
| `ps -ef` | সব process, PPID সহ |
| `ps -ef --forest` / `ps auxf` | গাছ আকারে |
| `ps -eo pid,ppid,stat,%cpu,cmd --sort=-%cpu` | কাস্টম কলাম + সাজানো |
| `ps -eLf` / `ps -T -p PID` | thread দেখা |
| `pstree -p` | process গাছ, PID সহ |
| `top` / `htop` | লাইভ মনিটর |
| `top`: `P` `M` `T` `k` `r` `1` `c` `V` `q` | CPU / মেম / সময় সাজাও, kill, renice, core, cmd, গাছ, বের হও |
| `htop`: `F3` `F4` `F5` `F6` `F7` `F8` `F9` `F10` | খোঁজা, ফিল্টার, গাছ, সাজানো, nice-, nice+, kill, বের হও |
| `pgrep -a name` | নাম দিয়ে PID খোঁজা |
| `pkill [-SIG] name` | নাম দিয়ে signal |
| `pidof name` | হুবহু নামের PID |
| `kill PID` | SIGTERM (ভদ্র) |
| `kill -9 PID` | SIGKILL (জোর) |
| `kill -HUP PID` | reload |
| `kill -STOP` / `-CONT` | pause / resume |
| `kill -0 PID` | বেঁচে আছে কি না যাচাই |
| `kill -l` | signal তালিকা |
| `killall name` | নামের সব process |
| `nice -n 10 cmd` | কম priority-তে চালাও |
| `renice -n 5 -p PID` | চলমান process-এর priority বদলাও |
| `ionice -c 3 cmd` | idle I/O priority |
| `cat /proc/PID/status` | process-এর বিস্তারিত |
| `uptime` / `cat /proc/loadavg` | load average |
| `nproc` | CPU core সংখ্যা |
| `Ctrl+C` | SIGINT |
| `Ctrl+Z` | SIGTSTP (থামাও) |
| `Ctrl+\` | SIGQUIT |
| `Ctrl+D` | EOF (signal নয়) |

---

## নিজেকে যাচাই করো

1. program আর process-এর মধ্যে মূল পার্থক্য কী? একই program থেকে কয়টা process চলতে পারে?
2. একই process-এর দুটো thread কী শেয়ার করে আর কী আলাদা রাখে?
3. শেলে `ls` চালালে কোন কোন system call কোন ক্রমে ঘটে? `cd` কেন আলাদা process হিসেবে চলে না?
4. `S` আর `D` state-এর পার্থক্য কী? কোনটাকে `kill -9` দিয়েও সাথে সাথে মারা যায় না, এবং কেন?
5. একটা zombie process `kill -9` দিয়ে মরছে না। কেন, আর কী করবে?
6. `ps aux`-এর `%CPU` আর `top`-এর `%CPU`-এর পার্থক্য কী?
7. SIGTERM আর SIGKILL-এর পার্থক্য কী? কেন সবসময় আগে SIGTERM পাঠানো উচিত?
8. একটা সাধারণ user `nice -n -5 ./app` চালালে কী হবে? `renice -n 10` দেওয়ার পর কি সে আবার 0-তে ফিরিয়ে আনতে পারবে?
9. ৮ core সার্ভারে load average `12.5, 9.0, 4.2`। কী বুঝবে?
10. একটা প্রোগ্রাম "exit code 137" দিয়ে মারা গেছে। এর মানে কী?

<details><summary>উত্তর দেখো</summary>

1. program হলো ডিস্কে রাখা নিষ্ক্রিয় executable ফাইল; process হলো সেই program-এর চলমান instance, যার নিজস্ব PID, মেমরি, খোলা ফাইল আছে। একই program থেকে যত খুশি process চলতে পারে (মেমরি ও PID সীমা পর্যন্ত)।
2. শেয়ার করে: code, heap, global data, খোলা file descriptor, PID। আলাদা: stack, CPU register, thread ID (TID)।
3. bash `fork()` করে নিজের কপি বানায় → child `execve("/usr/bin/ls")` করে → ls কাজ করে `exit()` করে → parent bash `wait()` দিয়ে exit code নেয়। `cd` আলাদা process হলে child-এর ডিরেক্টরি বদলাত, parent শেলের না; তাই এটা shell builtin।
4. `S` (interruptible sleep) signal পেলে জেগে ওঠে; `D` (uninterruptible) সাধারণত kernel-এর ভেতরে I/O সম্পূর্ণ হওয়ার অপেক্ষায় থাকে এবং signal-এ জাগে না। তাই `D`-তে থাকা process-এ SIGKILL পৌঁছায় না যতক্ষণ না I/O শেষ হয়।
5. zombie আগেই মৃত, শুধু process table-এ exit status-এর এন্ট্রি বাকি। মারার কিছু নেই। parent-কে `SIGCHLD` পাঠিয়ে wait() করাতে পারো, না হলে parent-কে বন্ধ/restart করো; তখন zombie PID 1 (বা subreaper)-এর কাছে গিয়ে সাফ হয়ে যায়।
6. `ps`-এর `%CPU` হলো process শুরু থেকে এখন পর্যন্ত গড়; `top`-এর `%CPU` হলো শেষ রিফ্রেশের পর থেকে, মানে এই মুহূর্তের ব্যবহার।
7. SIGTERM (15) একটা অনুরোধ, process catch করে পরিষ্কারভাবে (ফাইল সেভ, lock মুছে) বের হতে পারে। SIGKILL (9) kernel সরাসরি কার্যকর করে, catch/ignore করা যায় না, কোনো পরিষ্কারের সুযোগ নেই, ফলে ডেটা নষ্ট বা stale lock থাকতে পারে।
8. `nice: cannot set niceness: Permission denied` বার্তা আসবে (GNU nice তবু কমান্ডটা ডিফল্ট priority-তে চালিয়ে দেয়)। ঋণাত্মক nice শুধু root দিতে পারে। আর nice বাড়ানোর পর সাধারণ user সেটা কমিয়ে আগের অবস্থায় ফেরাতে পারে না (limits.conf-এ অনুমতি না থাকলে)।
9. ১ মিনিটের গড় 12.5 > 8 core, তাই এই মুহূর্তে task-রা অপেক্ষা করছে (overloaded)। ১৫ মিনিটে 4.2 থেকে বাড়তে বাড়তে 12.5, মানে চাপ সাম্প্রতিক এবং বাড়ছে। `top`-এ দেখো CPU (`us`/`sy`) নাকি I/O (`wa`, D state) এর কারণ।
10. 137 = 128 + 9, মানে process SIGKILL পেয়ে মারা গেছে। প্রায়ই এটা মেমরি ফুরিয়ে OOM killer-এর কাজ, বা কেউ `kill -9` দিয়েছে (`journalctl -k | grep -i oom` দিয়ে যাচাই করো)।

</details>

---

[← আগের অধ্যায়](23-text-tools.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](25-jobs-tmux.md)

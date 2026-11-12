# অধ্যায় ২৫: জব কন্ট্রোল, nohup, screen ও tmux

## এই অধ্যায়ে যা শিখবে

- **foreground** আর **background** process কী, আর কেন এই দুটো আলাদা
- `&` দিয়ে কমান্ড background-এ চালানো, `$!` আর `wait` দিয়ে তাকে ধরা
- `Ctrl+Z` চাপলে ভেতরে কী হয় (SIGTSTP), `jobs`, `fg`, `bg` দিয়ে job নিয়ন্ত্রণ
- job specification: `%1`, `%+`, `%-`, `%%`, `%name`, `%?text`
- টার্মিনাল বন্ধ করলে process কেন মরে যায়: **SIGHUP**, session, process group, controlling terminal
- `disown`, `nohup`, `setsid` দিয়ে process-কে টার্মিনাল থেকে মুক্ত করা
- systemd-logind-এর `KillUserProcesses` আর `loginctl enable-linger`
- **GNU screen**: session, window, detach/attach, split, সব `Ctrl+A` শর্টকাট
- **tmux** পুরোটা: client-server architecture, session/window/pane, `Ctrl+B` prefix-এর সব শর্টকাট, split, resize, copy mode, `.tmux.conf`, scripting
- বাস্তব কাজের ধরন: SSH-এ লম্বা কাজ চালানো, সার্ভারে নিরাপদে deploy

---

## ২৫.১ গল্প দিয়ে শুরু: রান্নাঘরের চুলা

ধরো তুমি রান্নাঘরে একা। একটা চুলায় ভাত বসিয়েছ। তুমি দুইভাবে কাজ করতে পারো:

1. **চুলার সামনে দাঁড়িয়ে থাকো** যতক্ষণ না ভাত হয়। এই সময়ে আর কিছু করতে পারছ না। এটা **foreground**।
2. **ভাত বসিয়ে দিয়ে সবজি কাটতে চলে যাও।** ভাত নিজের মতো সেদ্ধ হচ্ছে, তুমি অন্য কাজ করছ। এটা **background**।

মাঝে মাঝে তুমি ভাতের হাঁড়িটা চুলা থেকে নামিয়ে পাশে রেখে দাও (**pause**, মানে থামিয়ে রাখা), পরে আবার চুলায় তোলো (**resume**)।

এখন ধরো তুমি রান্নাঘর থেকে বেরিয়ে বাসা থেকেই চলে গেলে, আর যাওয়ার সময় গ্যাসের মেইন সুইচ বন্ধ করে দিলে। সব চুলা নিভে গেল। টার্মিনাল বন্ধ করলে ঠিক এটাই হয়: টার্মিনাল হলো "মেইন সুইচ", আর তার সাথে যুক্ত সব process একটা **SIGHUP** সিগন্যাল পেয়ে মরে যায়।

এই অধ্যায়ে আমরা শিখব:
- কীভাবে একই টার্মিনালে অনেক "চুলা" (job) সামলাবে → **job control**
- কীভাবে একটা চুলা মেইন সুইচ থেকে আলাদা করবে → **nohup, disown, setsid**
- কীভাবে পুরো রান্নাঘরটাই এমনভাবে বানাবে যে তুমি বাইরে গেলেও সব চলতে থাকে, আর ফিরে এসে ঠিক যেখানে ছিলে সেখান থেকে শুরু করতে পারো → **screen ও tmux**

> 📝 **মনে রাখো:** process কী, PID কী, signal কী, `kill` কীভাবে কাজ করে — এগুলো [অধ্যায় ২৪](24-process-management.md)-এ বিস্তারিত আছে। এখানে আমরা সেই জ্ঞানকে টার্মিনাল আর shell-এর দৃষ্টিকোণ থেকে ব্যবহার করব।

---

## ২৫.২ Job আর Process: পার্থক্য কী?

**Process** হলো kernel-এর ধারণা: চলমান একটা প্রোগ্রাম, যার একটা PID আছে।

**Job** হলো **shell-এর** ধারণা: তুমি একবার Enter চেপে যে কমান্ড লাইন চালাও, shell সেটাকে একটা job হিসেবে ধরে। একটা job-এ একাধিক process থাকতে পারে। যেমন:

```bash
$ cat access.log | grep 404 | sort | uniq -c
```

এখানে চারটা process (`cat`, `grep`, `sort`, `uniq`), কিন্তু **job একটাই**। shell এই চারটাকে একটা **process group**-এ রাখে, যাতে একসাথে থামানো, চালানো বা মারা যায়।

```
                 ┌──────────────── Job [1] ────────────────┐
                 │   Process Group (PGID = 5001)           │
                 │                                         │
  bash  ───────► │  cat(5001) → grep(5002) → sort(5003)    │
 (shell)         │                       → uniq(5004)      │
                 └─────────────────────────────────────────┘
```

> 💡 **টিপস:** job number (`[1]`, `[2]`...) শুধু ওই shell-এর ভেতরে অর্থবহ। অন্য টার্মিনাল থেকে `%1` লিখলে কাজ করবে না; সেখানে PID ব্যবহার করতে হবে।

---

## ২৫.৩ ভেতরে কীভাবে কাজ করে: Session, Process Group, Controlling Terminal

job control বুঝতে হলে Unix-এর তিনটা স্তর বুঝতে হবে। একটু তাত্ত্বিক, কিন্তু এটা বুঝলে বাকি সব (SIGHUP, nohup, setsid, tmux) একদম পরিষ্কার হয়ে যাবে।

### Session (সেশন)

তুমি যখন টার্মিনাল খোলো বা SSH দিয়ে লগইন করো, তখন একটা **session** তৈরি হয়। সাধারণত তোমার shell (`bash`) হয় **session leader**। session-এর একটা ID থাকে (SID), যা session leader-এর PID-এর সমান।

### Process Group (প্রসেস গ্রুপ)

session-এর ভেতরে প্রতিটা job একটা আলাদা **process group**। একটা সময়ে ঠিক **একটা** process group থাকে **foreground**-এ; বাকিগুলো **background**।

### Controlling Terminal (নিয়ন্ত্রণকারী টার্মিনাল)

session-টা একটা টার্মিনাল ডিভাইসের সাথে বাঁধা থাকে, যেমন `/dev/pts/0`। এটাকে বলে **controlling terminal**। টার্মিনাল ঠিক করে কিবোর্ডের ইনপুট আর `Ctrl+C`, `Ctrl+Z`-এর মতো সিগন্যাল কোন process group পাবে: শুধু **foreground group**।

```
┌─────────────────────── Session (SID = 4000) ───────────────────────┐
│                                                                    │
│   Controlling terminal: /dev/pts/0                                 │
│            │                                                       │
│            │ কিবোর্ড ইনপুট, Ctrl+C (SIGINT), Ctrl+Z (SIGTSTP)       │
│            ▼ শুধু foreground group-এ যায়                           │
│   ┌──────────────────────┐                                         │
│   │ FOREGROUND group     │   ◄── এখন তুমি এর সাথে কথা বলছ          │
│   │ PGID 5100: vim       │                                         │
│   └──────────────────────┘                                         │
│                                                                    │
│   ┌──────────────────────┐   ┌──────────────────────┐              │
│   │ BACKGROUND group     │   │ BACKGROUND group     │              │
│   │ PGID 5001: tar ...   │   │ PGID 5050: sleep 999 │              │
│   └──────────────────────┘   └──────────────────────┘              │
│                                                                    │
│   Session leader: bash (PID 4000, PGID 4000)                       │
└────────────────────────────────────────────────────────────────────┘
```

নিজের চোখে দেখো:

```bash
$ sleep 300 &
[1] 5050
$ ps -o pid,ppid,pgid,sid,tty,stat,cmd
    PID    PPID    PGID     SID TT       STAT CMD
   4000    3990    4000    4000 pts/0    Ss   bash
   5050    4000    5050    4000 pts/0    S    sleep 300
   5061    4000    5061    4000 pts/0    R+   ps -o pid,ppid,pgid,sid,tty,stat,cmd
```

লক্ষ করো:
- সবার `SID` একই (4000) → সবাই একই session-এ।
- প্রত্যেক job-এর নিজস্ব `PGID`।
- `STAT` কলামে `s` মানে session leader, `+` মানে foreground process group-এর সদস্য। `ps`-এর পাশে `+` আছে, `sleep`-এর পাশে নেই।

### Background process টার্মিনাল থেকে পড়তে চাইলে কী হয়?

একটা background process যদি কিবোর্ড থেকে **ইনপুট পড়তে** চায়, kernel তাকে **SIGTTIN** সিগন্যাল পাঠিয়ে থামিয়ে দেয়। কারণ কিবোর্ড শুধু foreground-এর জন্য।

```bash
$ read name &
[1] 5200
[1]+  Stopped                 read name
```

**লেখার** ক্ষেত্রে ডিফল্টভাবে background process টার্মিনালে লিখতে পারে (তাই কখনো কখনো আউটপুট এলোমেলোভাবে prompt-এর মাঝে চলে আসে)। তবে `stty tostop` চালু করলে background-এর লেখার চেষ্টাতেও **SIGTTOU** পাঠিয়ে থামানো হয়।

```bash
$ stty tostop        # চালু
$ stty -tostop       # বন্ধ (ডিফল্ট)
```

---

## ২৫.৪ Foreground আর Background-এ কমান্ড চালানো

### Foreground (ডিফল্ট)

```bash
$ sleep 10
```

10 সেকেন্ড prompt ফেরত আসবে না। shell অপেক্ষা করছে (`waitpid()` system call দিয়ে)।

### `&` দিয়ে Background-এ চালানো

কমান্ডের শেষে `&` দিলে shell কমান্ড চালু করেই সাথে সাথে prompt ফেরত দেয়।

```bash
$ sleep 100 &
[1] 6123
$
```

- `[1]` হলো **job number**
- `6123` হলো ওই job-এর শেষ process-এর **PID** (pipeline হলে শেষ কমান্ডের PID)

কয়েকটা চালাও:

```bash
$ sleep 200 &
[2] 6130
$ sleep 300 &
[3] 6131
```

job শেষ হলে পরের বার Enter চাপলে shell জানায়:

```bash
$
[1]   Done                    sleep 100
```

> 💡 **টিপস:** shell সাধারণত পরের prompt দেখানোর ঠিক আগে job-এর অবস্থা জানায়। সাথে সাথে জানতে চাইলে `set -b` (বা `set -o notify`) চালু করো।

### `$!`: শেষ background process-এর PID

```bash
$ gzip -k big.log &
[1] 6200
$ echo $!
6200
```

স্ক্রিপ্টে এটা খুব কাজের:

```bash
long_task &
pid=$!
echo "কাজ চলছে, PID = $pid"
wait "$pid"
echo "শেষ হয়েছে, exit status = $?"
```

### `wait`: background job শেষ হওয়া পর্যন্ত অপেক্ষা

| কমান্ড | কাজ |
|---|---|
| `wait` | সব background child শেষ হওয়া পর্যন্ত অপেক্ষা |
| `wait PID` | নির্দিষ্ট PID শেষ হওয়া পর্যন্ত অপেক্ষা, তার exit status ফেরত দেয় |
| `wait %2` | job 2-এর জন্য অপেক্ষা |
| `wait -n` | যেকোনো **একটা** job শেষ হলেই ফেরত আসে (bash 4.3+) |

উদাহরণ — তিনটা ডাউনলোড সমান্তরালে:

```bash
curl -sO https://example.com/a.iso &
curl -sO https://example.com/b.iso &
curl -sO https://example.com/c.iso &
wait
echo "তিনটাই নামানো শেষ"
```

### `&` ছাড়া `;` আর `&&`-এর পার্থক্য

| লেখা | অর্থ |
|---|---|
| `a ; b` | a শেষ হলে b, a-এর ফলাফল যাই হোক |
| `a && b` | a সফল হলেই b |
| `a & b` | a background-এ, b সাথে সাথে foreground-এ |
| `(a; b) &` | পুরো গ্রুপটা একটা subshell-এ background-এ |

> ⚠️ **সতর্কতা:** `&` দিয়ে চালালেও process তখনো এই টার্মিনালের session-এর সদস্য। টার্মিনাল বন্ধ করলে সাধারণত এটা মরে যাবে। নিচে ২৫.৭-এ দেখব কেন।

---

## ২৫.৫ `Ctrl+Z`, `jobs`, `fg`, `bg`

### `Ctrl+Z`: চলমান কাজ থামানো (suspend)

ধরো তুমি একটা বড় ফাইল কপি শুরু করলে, কিন্তু `&` দিতে ভুলে গেছ:

```bash
$ cp ubuntu.iso /mnt/usb/
```

এখন prompt আটকে আছে। `Ctrl+Z` চাপো:

```bash
^Z
[1]+  Stopped                 cp ubuntu.iso /mnt/usb/
$
```

**ভেতরে কী হলো?**
1. তুমি `Ctrl+Z` চাপলে টার্মিনাল ড্রাইভার (kernel-এর tty layer) এটাকে সাধারণ অক্ষর হিসেবে না দেখে চিনে নেয়, কারণ এটা `susp` ক্যারেক্টার হিসেবে সেট করা (`stty -a` দিয়ে দেখো: `susp = ^Z`)।
2. kernel foreground process group-এর সব process-কে **SIGTSTP** পাঠায়।
3. process-গুলো **Stopped** অবস্থায় যায় (`ps`-এ `STAT` = `T`)। CPU ব্যবহার করে না, কিন্তু মেমরিতে থাকে।
4. shell দেখে তার child থেমে গেছে, তাই টার্মিনালের foreground নিজের কাছে ফিরিয়ে নিয়ে prompt দেখায়।

```
  Ctrl+Z ─► tty driver ─► SIGTSTP ─► foreground group (cp)  ─► STOPPED (T)
                                          │
                       bash টার্মিনাল ফেরত পায় ◄──┘
```

> 📝 **মনে রাখো:** `Ctrl+Z` কাজটা **মারে না**, শুধু থামায়। `Ctrl+C` (SIGINT) সাধারণত মেরে ফেলে। দুটো গুলিয়ে ফেলো না।

### `jobs`: এই shell-এর সব job-এর তালিকা

```bash
$ sleep 500 &
[2] 6301
$ vim notes.txt
^Z
[3]+  Stopped                 vim notes.txt
$ jobs
[1]   Stopped                 cp ubuntu.iso /mnt/usb/
[2]   Running                 sleep 500 &
[3]+  Stopped                 vim notes.txt
```

- `+` = **current job** (সবচেয়ে সাম্প্রতিক; `fg`/`bg` অপশন ছাড়া দিলে এটাই ধরে)
- `-` = **previous job** (এর আগেরটা)

| অপশন | কাজ |
|---|---|
| `jobs -l` | job number-এর সাথে PID-ও দেখায় |
| `jobs -p` | শুধু process group leader-এর PID দেখায় (স্ক্রিপ্টে কাজের) |
| `jobs -r` | শুধু running job |
| `jobs -s` | শুধু stopped job |
| `jobs -n` | শেষবার জানানোর পর যাদের অবস্থা বদলেছে |

```bash
$ jobs -l
[1]  6250 Stopped                 cp ubuntu.iso /mnt/usb/
[2]  6301 Running                 sleep 500 &
[3]+ 6310 Stopped                 vim notes.txt
```

### `bg`: থামানো job-কে background-এ চালু করা

```bash
$ bg %1
[1]  cp ubuntu.iso /mnt/usb/ &
$ jobs
[1]   Running                 cp ubuntu.iso /mnt/usb/ &
[2]-  Running                 sleep 500 &
[3]+  Stopped                 vim notes.txt
```

ভেতরে: shell ওই process group-কে **SIGCONT** পাঠায়, কিন্তু টার্মিনালের foreground নিজের কাছেই রাখে।

> 💡 **টিপস:** "ভুলে `&` দিইনি" সমস্যার ক্লাসিক সমাধান: `Ctrl+Z` তারপর `bg`। কাজটা আবার শুরু থেকে চালাতে হয় না।

### `fg`: job-কে foreground-এ আনা

```bash
$ fg %3
vim notes.txt
```

vim আবার পর্দায় ফিরে আসবে, ঠিক যেখানে ছিলে। ভেতরে shell দুটো কাজ করে: `tcsetpgrp()` দিয়ে টার্মিনালের foreground group বদলে ওই job-কে দেয়, তারপর SIGCONT পাঠায় (job থেমে থাকলে)।

### সবচেয়ে বাস্তব কাজের ধরন: vim ↔ shell

প্রোগ্রামাররা প্রতিদিন এটা করে:

```bash
$ vim app.py      # কোড লিখছ
^Z                # shell-এ এলে
$ python3 app.py  # চালিয়ে দেখলে
$ fg              # vim-এ ফেরত
```

> ⚠️ **সতর্কতা:** stopped job রেখে `exit` করলে bash সতর্ক করে:
> ```
> $ exit
> exit
> There are stopped jobs.
> ```
> আবার `exit` দিলে সে বেরিয়ে যাবে এবং stopped job-গুলো SIGHUP পেয়ে মারা যাবে। vim-এ unsaved কাজ থাকলে তা হারাবে (vim অবশ্য swap ফাইল রেখে যায়, [অধ্যায় ১৪](14-vim.md) দেখো)।

---

## ২৫.৬ Job Specification: `%`-এর পুরো ভাষা

`fg`, `bg`, `kill`, `wait`, `disown` সবাই এই নোটেশন বোঝে:

| লেখা | অর্থ | উদাহরণ |
|---|---|---|
| `%n` | job number n | `fg %2` |
| `%+` বা `%%` | current job (যার পাশে `+`) | `bg %%` |
| `%-` | previous job (যার পাশে `-`) | `fg %-` |
| `%string` | যে job-এর কমান্ড `string` দিয়ে **শুরু** | `fg %vim` |
| `%?string` | যে job-এর কমান্ডে কোথাও `string` আছে | `kill %?iso` |

শর্টকাট:
- শুধু `fg` বা `bg` = `%+`
- শুধু `%1` লিখে Enter = `fg %1`
- `%1 &` লিখে Enter = `bg %1`

```bash
$ %2          # job 2 foreground-এ
$ %vim &      # vim নামের job background-এ (যদিও vim background-এ বসে থামবে)
```

> ⚠️ **সতর্কতা:** `%string` যদি একাধিক job-এর সাথে মেলে, bash ভুল দেখায়: `bash: fg: vim: ambiguous job spec`। তখন নম্বর ব্যবহার করো।

### `kill` দিয়ে job মারা

```bash
$ kill %1             # SIGTERM পাঠায় job 1-এর পুরো process group-এ
$ kill -9 %2          # SIGKILL
$ kill -CONT %3       # থামানো job চালু (কিন্তু background-এ থাকবে)
$ kill -STOP %2       # Ctrl+Z-এর মতো, তবে SIGSTOP ধরা/উপেক্ষা করা যায় না
```

> 📝 **মনে রাখো:** `kill %1` হলো bash-এর **builtin** `kill`-এর সুবিধা। `/bin/kill %1` কাজ করবে না, কারণ বাইরের প্রোগ্রাম shell-এর job table জানে না।

stopped job-কে সিগন্যাল পাঠালে কী হয়? থেমে থাকা process বেশিরভাগ সিগন্যাল হাতে পায় চালু হওয়ার পরে। bash-এর builtin `kill` এটা জানে: stopped job-কে SIGTERM বা SIGHUP পাঠালে সে নিজে থেকেই সাথে একটা SIGCONT-ও পাঠায়, তাই job জেগে উঠে সিগন্যাল পেয়ে শেষ হয়। অন্য সিগন্যালে (যেমন `kill -INT %1`) job "Stopped"-ই থেকে যেতে পারে; তখন `kill -CONT %1` দাও, বা শেষ উপায় `kill -9 %1` (SIGKILL থামানো process-কেও সাথে সাথে মারে)।

---

## ২৫.৭ SIGHUP: টার্মিনাল বন্ধ করলে process কেন মরে?

### "Hang up" শব্দটা কোথা থেকে এল

পুরনো যুগে মানুষ টেলিফোন লাইন আর modem দিয়ে কম্পিউটারে যুক্ত হতো। ফোন রেখে দিলে (hang up) লাইন কেটে যেত। তখন kernel ওই লাইনের session-কে জানাত: "ইউজার চলে গেছে।" সেই সিগন্যালের নাম **SIGHUP** (signal hang up, নম্বর 1)। আজ modem নেই, কিন্তু নিয়মটা আছে: টার্মিনাল উইন্ডো বন্ধ করা বা SSH সংযোগ কেটে যাওয়া = "hang up"।

### ধাপে ধাপে কী ঘটে

```
 তুমি টার্মিনাল উইন্ডো বন্ধ করলে / SSH কেটে গেল
              │
              ▼
 ① pseudo-terminal-এর master side বন্ধ হলো (GNOME Terminal বা sshd সেটা ধরে রাখত)
              │
              ▼
 ② kernel controlling terminal-এর session leader-কে (bash) SIGHUP পাঠায়
              │
              ▼
 ③ bash SIGHUP পেয়ে বেরোনোর আগে তার job table-এর সব job-কে SIGHUP পাঠায়
    (stopped job-কে SIGCONT-ও পাঠায় যাতে তারা সিগন্যালটা হাতে পায়)
              │
              ▼
 ④ SIGHUP-এর ডিফল্ট কাজ = process শেষ করা  →  তোমার background কাজ মারা গেল
```

এছাড়া session leader মারা গেলে kernel নিজেও foreground process group-কে SIGHUP পাঠায়, আর কোনো process group "orphaned" হয়ে গেলে এবং তাতে stopped process থাকলে তাকে SIGHUP + SIGCONT পাঠায়।

### bash-এর `huponexit` অপশন

ওপরের ধাপ ③ ঘটে যখন bash **SIGHUP পেয়ে** বেরোয়। কিন্তু তুমি যদি নিজে `exit` টাইপ করে বা `Ctrl+D` দিয়ে বেরোও, তখন কী? তখন bash ডিফল্টভাবে running background job-দের SIGHUP **পাঠায় না** (stopped job থাকলে আগে সতর্ক করে)। এটা নিয়ন্ত্রণ করে `huponexit`:

```bash
$ shopt huponexit
huponexit       off
```

`shopt -s huponexit` দিলে login shell থেকে `exit` করার সময়ও সব job-কে SIGHUP পাঠাবে।

> 💡 **টিপস:** এজন্যই দেখা যায় "`&` দিয়ে চালিয়ে `exit` দিলাম, কাজ চলছে, কিন্তু উইন্ডো ক্রস দিয়ে বন্ধ করলাম তো কাজ মরে গেল।" উইন্ডো বন্ধ করা = SIGHUP, `exit` = SIGHUP নয়। তবে এর ওপর ভরসা কোরো না; নিচের পদ্ধতিগুলো ব্যবহার করো।

### systemd-logind-এর ভূমিকা: `KillUserProcesses`

আধুনিক distro-তে আরেকটা স্তর আছে। systemd-logind প্রতিটা লগইনকে একটা **scope** (cgroup)-এ রাখে। `/etc/systemd/logind.conf`-এ:

```ini
[Login]
KillUserProcesses=no
```

- `no` (Ubuntu, Debian, Fedora, RHEL-এ ডিফল্ট): লগআউটের পরও nohup/disown/tmux দিয়ে চালানো process বেঁচে থাকে।
- `yes`: লগআউট করলেই ওই session-এর cgroup-এর **সব** process মেরে ফেলা হয়, nohup দিলেও। এমনকি tmux-ও মরে যাবে!

`yes` থাকা সিস্টেমে কাজ বাঁচানোর উপায়:

```bash
$ loginctl enable-linger $USER                     # ইউজারের systemd instance লগআউটেও চালু থাকবে
$ systemd-run --user --scope tmux new -s work      # tmux-কে আলাদা scope-এ চালাও
```

> 📝 **মনে রাখো:** সত্যিকারের দীর্ঘমেয়াদি সার্ভিস (ওয়েব সার্ভার, ব্যাকগ্রাউন্ড worker) tmux বা nohup দিয়ে চালানো ঠিক নয়। ওগুলো systemd service বানাও, [অধ্যায় ২৬](26-systemd-services.md) দেখো। tmux/nohup হলো এক-দুবারের লম্বা কাজের জন্য (বড় কপি, compile, migration, ডেটা প্রসেসিং)।

---

## ২৫.৮ `disown`: job table থেকে বাদ দেওয়া

`disown` bash-এর builtin। সে job-কে shell-এর job table থেকে সরিয়ে দেয়, ফলে টার্মিনাল বন্ধ হলে bash যখন সব job-কে SIGHUP পাঠায়, এই job সেই তালিকায় থাকে না।

```bash
$ ./long_backup.sh &
[1] 7001
$ disown %1
$ jobs
$                   # তালিকা খালি, কিন্তু process চলছে
$ ps -p 7001
    PID TTY          TIME CMD
   7001 pts/0    00:00:03 long_backup.sh
```

| অপশন | কাজ |
|---|---|
| `disown` | current job (`%+`) সরায় |
| `disown %n` | নির্দিষ্ট job সরায় |
| `disown -a` | সব job সরায় |
| `disown -r` | শুধু running job সরায় |
| `disown -h %n` | job table-এ রাখে (`jobs`-এ দেখা যাবে), কিন্তু চিহ্ন দিয়ে রাখে যাতে shell তাকে SIGHUP না পাঠায় |

**সবচেয়ে কাজের রেসিপি**: ভুলে nohup ছাড়া চালিয়ে ফেলেছ, এখন বাড়ি যেতে হবে:

```bash
$ ./import_data.sh          # foreground-এ চলছে
^Z
[1]+  Stopped                 ./import_data.sh
$ bg
[1]+ ./import_data.sh &
$ disown -h %1              # বা: disown %1
$ exit
```

> ⚠️ **সতর্কতা:** `disown`-এর পরেও process-এর stdout/stderr ওই টার্মিনালেই যুক্ত। টার্মিনাল চলে গেলে সে লিখতে চেষ্টা করলে write error (EIO) বা SIGPIPE পেতে পারে, আর কিছু প্রোগ্রাম তাতে মারা যায়। নিরাপদ পথ হলো শুরু থেকেই আউটপুট ফাইলে পাঠানো: `./import_data.sh > import.log 2>&1 &`

> 📝 **মনে রাখো:** `disown` kernel-এর SIGHUP বন্ধ করে না, শুধু **bash**-এর পাঠানো SIGHUP থেকে বাঁচায়। process ভিন্ন process group-এ background-এ থাকলে kernel সাধারণত সরাসরি তাকে hang-up পাঠায় না, তাই বাস্তবে এটাই যথেষ্ট হয়।

---

## ২৫.৯ `nohup`: SIGHUP উপেক্ষা করে চালানো

`nohup` (no hang up) একটা ছোট প্রোগ্রাম (coreutils-এর অংশ)। সে যা করে:

1. নিজের জন্য SIGHUP-কে **ignore** (SIG_IGN) সেট করে।
2. তারপর `exec` দিয়ে তোমার কমান্ডে রূপান্তরিত হয়। ignore করা সিগন্যাল `exec`-এর পরেও ignore থাকে, তাই তোমার প্রোগ্রাম SIGHUP পেলে কিছুই হয় না।
3. stdout যদি টার্মিনাল হয়, আউটপুট `nohup.out` ফাইলে পাঠায় (বর্তমান ডিরেক্টরিতে, লেখা না গেলে `$HOME/nohup.out`)।
4. stderr টার্মিনাল হলে সেটাকেও stdout-এ পাঠায়।
5. stdin টার্মিনাল হলে সেটাকে একটা অপাঠযোগ্য ফাইল থেকে redirect করে (ইনপুট ignore)।

### Syntax

```bash
nohup COMMAND [ARGS...]
```

### উদাহরণ

```bash
$ nohup ./train_model.py &
[1] 7200
$ nohup: ignoring input and appending output to 'nohup.out'
```

নিজে log ফাইল ঠিক করে দেওয়া (প্রস্তাবিত পদ্ধতি):

```bash
$ nohup ./train_model.py > train.log 2>&1 &
[1] 7210
$ tail -f train.log
Epoch 1/50 ... loss 0.84
Epoch 2/50 ... loss 0.71
```

এবার নিশ্চিন্তে SSH থেকে বেরিয়ে যাও। পরে ফিরে এসে:

```bash
$ pgrep -af train_model
7210 python3 ./train_model.py
$ tail train.log
```

> ⚠️ **সতর্কতা:**
> - `nohup` নিজে process-কে background-এ পাঠায় না। শেষে `&` দিতেই হবে।
> - `nohup` শুধু SIGHUP থেকে বাঁচায়। `Ctrl+C` (foreground-এ থাকলে), SIGTERM, SIGKILL, OOM killer বা `KillUserProcesses=yes` থেকে বাঁচায় না।
> - `nohup cd /tmp` বা `nohup alias` কাজ করে না। nohup একটা আলাদা প্রোগ্রাম, তাই shell builtin, alias বা function চালাতে পারে না। দরকার হলে: `nohup bash -c 'cmd1 && cmd2' > out.log 2>&1 &`

### nohup বনাম disown

| বিষয় | `nohup` | `disown` |
|---|---|---|
| কে | আলাদা প্রোগ্রাম (`/usr/bin/nohup`) | bash builtin |
| কখন ব্যবহার | কমান্ড **শুরুর আগে** | কমান্ড **চালু হয়ে যাওয়ার পরে** |
| কীভাবে বাঁচায় | process নিজেই SIGHUP ignore করে | bash তাকে SIGHUP পাঠানোর তালিকা থেকে বাদ দেয় |
| আউটপুট | স্বয়ংক্রিয়ভাবে `nohup.out`-এ | কোনো পরিবর্তন নেই |
| অন্য কেউ `kill -HUP` দিলে | বেঁচে যায় | মারা যায় |

---

## ২৫.১০ `setsid`: সম্পূর্ণ নতুন session

`setsid` (util-linux প্যাকেজ) process-কে একদম **নতুন session**-এ চালায়। নতুন session মানে:
- নতুন SID, process নিজেই session leader
- **কোনো controlling terminal নেই**

টার্মিনালের সাথে কোনো সম্পর্কই নেই, তাই টার্মিনাল বন্ধ হলে তার কাছে SIGHUP যাওয়ার প্রশ্নই ওঠে না। এটা অনেকটা daemon-এর মতো।

```
 আগে:  Session(bash, pts/0) ──► job: ./server
 পরে:  Session(bash, pts/0)      Session(./server, tty = ?)   ← সম্পূর্ণ আলাদা
```

### Syntax ও অপশন

```bash
setsid [options] program [arguments]
```

| অপশন | কাজ |
|---|---|
| `-f`, `--fork` | সবসময় fork করে নতুন process-এ চালায় (shell সাথে সাথে prompt ফেরত পায়) |
| `-w`, `--wait` | প্রোগ্রাম শেষ হওয়া পর্যন্ত অপেক্ষা করে তার exit status ফেরত দেয় |
| `-c`, `--ctty` | বর্তমান টার্মিনালকে নতুন session-এর controlling terminal বানায় |

### উদাহরণ

```bash
$ setsid -f ./worker.sh > worker.log 2>&1 < /dev/null
$ ps -o pid,pgid,sid,tty,cmd -p $(pgrep -f worker.sh)
    PID    PGID     SID TT       CMD
   7400    7400    7400 ?        /bin/bash ./worker.sh
```

`TT` কলামে `?` মানে কোনো টার্মিনাল নেই, `SID` = নিজের PID।

> 💡 **টিপস:** GUI অ্যাপ টার্মিনাল থেকে খুলে টার্মিনাল বন্ধ করতে চাইলে: `setsid -f firefox >/dev/null 2>&1`

### তিনটার তুলনা এক নজরে

| পদ্ধতি | কখন | টার্মিনাল থেকে বিচ্ছিন্নতার মাত্রা |
|---|---|---|
| `cmd &` | দ্রুত, টার্মিনাল খোলা থাকবে | নেই |
| `cmd & disown` | চালানোর পরে বাঁচাতে | মাঝারি |
| `nohup cmd &` | শুরুতেই জানো লম্বা চলবে | ভালো |
| `setsid -f cmd` | পুরোপুরি আলাদা করতে চাও | সম্পূর্ণ |
| `tmux`/`screen` | পরে ফিরে এসে আউটপুট দেখতে/ইন্টারঅ্যাক্ট করতে চাও | সম্পূর্ণ + ফিরে আসা যায় |
| `systemd-run`/service | স্থায়ী সার্ভিস | সম্পূর্ণ + পরিচালিত |

একটা বড় সীমাবদ্ধতা: nohup/disown/setsid দিয়ে চালানো process-এর কাছে তুমি আর **ফিরে যেতে পারবে না**। তার পর্দা দেখা বা কিবোর্ড দিয়ে উত্তর দেওয়া সম্ভব নয়। এই সমস্যার সমাধানই হলো **terminal multiplexer**: screen আর tmux।

---

## ২৫.১১ Terminal Multiplexer: ধারণা

**Multiplexer** মানে একটা জিনিস দিয়ে অনেককে চালানো। terminal multiplexer একটা টার্মিনাল উইন্ডোর ভেতরে অনেকগুলো "ভার্চুয়াল টার্মিনাল" দেয়, আর সবচেয়ে বড় কথা: এগুলো তোমার টার্মিনাল থেকে **বিচ্ছিন্ন (detach)** হয়েও বেঁচে থাকে।

**উপমা:** টেলিভিশন আর কেবল অপারেটর। টিভি (তোমার টার্মিনাল) বন্ধ করলেও কেবল অপারেটরের কাছে চ্যানেল (tmux session) চলতেই থাকে। আবার টিভি খুললে (attach) দেখবে অনুষ্ঠান সামনে এগিয়ে গেছে। চাইলে অন্য ঘরের টিভিতেও (অন্য কম্পিউটার থেকে SSH) একই চ্যানেল দেখা যায়।

```
  তোমার ল্যাপটপ                              সার্ভার
 ┌────────────┐     SSH      ┌──────────────────────────────────────┐
 │ Terminal   │─────────────►│ sshd ─► bash ─► tmux client          │
 └────────────┘  (কেটে গেলে  │                    │  (socket)        │
                  শুধু এই    │                    ▼                  │
                  অংশ মরে)   │           ┌── tmux server ──┐        │
                             │           │ (setsid করা,     │        │
                             │           │  টার্মিনালহীন)   │        │
                             │           │ session "work"  │        │
                             │           │  ├─ bash        │        │
                             │           │  └─ ./migrate   │ ← বেঁচে │
                             │           └─────────────────┘   থাকে  │
                             └──────────────────────────────────────┘
```

**কেন কাজ করে:** tmux/screen-এর server process নিজেকে আলাদা session-এ (setsid-এর মতো) নিয়ে যায় এবং তোমার প্রোগ্রামগুলোকে **নিজের তৈরি pseudo-terminal**-এ চালায়। তোমার আসল টার্মিনাল শুধু একটা "জানালা" যা দিয়ে তুমি ওই pseudo-terminal দেখছ। জানালা ভাঙলেও ঘর ঠিক থাকে।

### ইনস্টলেশন

```bash
# Ubuntu / Debian
$ sudo apt install tmux screen

# Fedora
$ sudo dnf install tmux screen

# RHEL 9 / Rocky / Alma: tmux base repo-তে আছে; screen RHEL 8 থেকে বাদ পড়েছে, EPEL-এ পাওয়া যায়
$ sudo dnf install tmux
$ sudo dnf install epel-release && sudo dnf install screen
```

সংস্করণ দেখো:

```bash
$ tmux -V
tmux 3.4
$ screen --version
Screen version 4.09.01 (GNU) 20-Aug-23
```

> 💡 **টিপস:** নতুন কেউ হলে **tmux** শেখো; এটা সক্রিয়ভাবে উন্নত হচ্ছে, কনফিগার করা সহজ, scripting শক্তিশালী। তবে পুরনো সার্ভারে প্রায়ই শুধু screen থাকে, তাই screen-এর বেসিকও জানা দরকার।

---

## ২৫.১২ GNU screen

screen ১৯৮৭ সাল থেকে আছে। প্রায় সব পুরনো Unix/Linux সার্ভারে পাওয়া যায়।

### screen-এর গঠন

```
 screen session "deploy"
 ├── window 0: bash       (লগ দেখছ)
 ├── window 1: vim        (কনফিগ এডিট)
 └── window 2: htop       (মনিটর)
```

screen-এ **session** আর **window** আছে। একটা window = একটা পূর্ণ-পর্দার ভার্চুয়াল টার্মিনাল। পর্দাকে ভাগ করা অংশকে screen বলে **region**।

### কমান্ড লাইন থেকে screen

| কমান্ড | কাজ |
|---|---|
| `screen` | নতুন নামহীন session শুরু |
| `screen -S deploy` | `deploy` নামে নতুন session |
| `screen -ls` (বা `-list`) | সব session-এর তালিকা |
| `screen -r` | একমাত্র detached session-এ ফেরত যাও |
| `screen -r deploy` | নির্দিষ্ট session-এ attach |
| `screen -d -r deploy` | অন্য কোথাও attached থাকলে সেখান থেকে detach করে এখানে attach |
| `screen -D -R deploy` | জোর করে detach + attach, session না থাকলে নতুন তৈরি করে |
| `screen -x deploy` | ইতিমধ্যে attached session-এ **একসাথে** attach (দুজন একই পর্দা দেখবে) |
| `screen -dmS job1 ./backup.sh` | detached অবস্থায় session চালু করে কমান্ড চালায় |
| `screen -S deploy -X quit` | বাইরে থেকে session বন্ধ |
| `screen -wipe` | মৃত (dead) session-এর আবর্জনা পরিষ্কার |

```bash
$ screen -S deploy
# ... কাজ করো, তারপর Ctrl+A d
[detached from 8120.deploy]
$ screen -ls
There is a screen on:
        8120.deploy     (Detached)
1 Socket in /run/screen/S-rahim.
$ screen -r deploy
```

### screen-এর শর্টকাট (prefix = `Ctrl+A`)

screen-এর সব কিবোর্ড কমান্ড শুরু হয় **`Ctrl+A`** দিয়ে (একে বলে command key বা prefix)। প্রথমে `Ctrl+A` চেপে ছেড়ে দাও, তারপর পরের কী চাপো। নিচের টেবিলে `C-a d` মানে "`Ctrl+A`, তারপর `d`"।

| শর্টকাট | কাজ |
|---|---|
| `C-a ?` | সব key binding-এর সাহায্য তালিকা |
| `C-a d` | **detach** (session চলতে থাকবে) |
| `C-a c` | নতুন window তৈরি |
| `C-a n` / `C-a Space` | পরের window |
| `C-a p` / `C-a Backspace` | আগের window |
| `C-a 0` … `C-a 9` | নির্দিষ্ট নম্বরের window-এ যাও |
| `C-a C-a` | শেষ ব্যবহৃত window-এ ফেরত (দুই window-এর মধ্যে toggle) |
| `C-a "` | window-এর তালিকা থেকে বেছে নাও |
| `C-a w` | window তালিকা status line-এ দেখাও |
| `C-a '` | নাম/নম্বর লিখে window-এ যাও |
| `C-a A` | বর্তমান window-এর নাম (title) বদলাও |
| `C-a k` | বর্তমান window kill (নিশ্চিত করতে বলবে) |
| `C-a \` | সব window বন্ধ করে screen থেকে বেরোও (নিশ্চিত করতে বলবে) |
| `C-a S` | পর্দা ওপর-নিচে ভাগ (horizontal split) |
| `C-a \|` | পর্দা বাম-ডানে ভাগ (vertical split, screen 4.1+) |
| `C-a Tab` | পরের region-এ যাও |
| `C-a X` | বর্তমান region বন্ধ |
| `C-a Q` | বর্তমান ছাড়া সব region বন্ধ |
| `C-a [` বা `C-a Esc` | copy/scrollback mode (vi-এর মতো চলাচল, `Space` দিয়ে শুরু-শেষ চিহ্নিত করে কপি) |
| `C-a ]` | কপি করা লেখা paste |
| `C-a H` | বর্তমান window-এর log ফাইল (`screenlog.N`) চালু/বন্ধ |
| `C-a M` | এই window-এ activity হলে জানাও (monitor) |
| `C-a _` | এই window চুপ থাকলে (silence) জানাও |
| `C-a x` | session lock (পাসওয়ার্ড দিয়ে খুলতে হবে) |
| `C-a :` | screen-এর কমান্ড prompt (যেমন `:resize 10`) |
| `C-a a` | প্রোগ্রামের কাছে আসল `Ctrl+A` পাঠাও (bash-এ লাইনের শুরুতে যেতে) |
| `C-a t` | সময় ও লোড দেখাও |
| `C-a i` | বর্তমান window-এর তথ্য |

> 💡 **টিপস:** split করার পর নতুন region-টা খালি থাকে। `C-a Tab` দিয়ে সেখানে গিয়ে `C-a c` দিয়ে নতুন window তৈরি করতে হয়, অথবা `C-a 1` দিয়ে কোনো বিদ্যমান window দেখাতে হয়।

> ⚠️ **সতর্কতা:** screen-এ split layout detach করলে হারিয়ে যায় (window-গুলো থাকে, ভাগ থাকে না)। tmux-এ pane-গুলো detach-এর পরেও অক্ষত থাকে।

### `~/.screenrc` উদাহরণ

```bash
# স্বাগত বার্তা বন্ধ
startup_message off
# scrollback বড় করো
defscrollback 10000
# নিচে সবসময় window তালিকা দেখাও
hardstatus alwayslastline
hardstatus string '%{= kG}[%H] %{= kw}%-w%{= kW}[%n %t]%{-}%+w %=%{= kY}%c'
# visual bell
vbell on
```

---

## ২৫.১৩ tmux: পরিচিতি ও Architecture

**tmux** (terminal multiplexer) ২০০৭ সালে এসেছে, OpenBSD-তে ডিফল্ট, আর আধুনিক লিনাক্স ব্যবহারকারী ও DevOps ইঞ্জিনিয়ারদের প্রিয় টুল।

### Client-Server মডেল

tmux চালু করলে আসলে দুটো জিনিস তৈরি হয়:

1. **tmux server**: একটা background process যে সব session, window, pane আর তাদের ভেতরের প্রোগ্রাম ধরে রাখে। প্রথম `tmux` কমান্ডেই এটা স্বয়ংক্রিয়ভাবে চালু হয়, আর শেষ session বন্ধ হলে নিজে থেকেই বন্ধ হয়।
2. **tmux client**: তোমার টার্মিনালে যা দেখছ। এটা শুধু server-এর সাথে কথা বলে আর পর্দা আঁকে।

দুজনে কথা বলে একটা **Unix domain socket** দিয়ে, যা থাকে `/tmp/tmux-UID/default`-এ (যেমন `/tmp/tmux-1000/default`)।

```
   Terminal A          Terminal B (অন্য SSH)
  ┌───────────┐        ┌───────────┐
  │tmux client│        │tmux client│
  └─────┬─────┘        └─────┬─────┘
        │   /tmp/tmux-1000/default (Unix socket)
        └──────────┬─────────┘
                   ▼
        ┌─────────────────────┐
        │     tmux server      │  ← টার্মিনালহীন, নিজের session-এ
        │  ┌───────────────┐   │
        │  │ session: work │   │
        │  │  window 0 ... │   │
        │  └───────────────┘   │
        │  ┌───────────────┐   │
        │  │ session: db   │   │
        │  └───────────────┘   │
        └─────────────────────┘
```

এজন্যই detach করলে কিছু হারায় না: শুধু client বেরিয়ে যায়, server আর তার ভেতরের সব কিছু চলতে থাকে।

```bash
$ ps -o pid,sid,tty,cmd -C tmux
    PID     SID TT       CMD
   9001    9001 ?        tmux new -s work       ← server (TT = ?)
   8990    4000 pts/0    tmux new -s work       ← client
```

### Session → Window → Pane

tmux-এর তিন স্তরের ধারণা:

| স্তর | কী | উপমা |
|---|---|---|
| **Session** | একটা প্রজেক্ট/কাজের পুরো সংগ্রহ; attach/detach হয় এই স্তরে | একটা ব্রাউজার উইন্ডো |
| **Window** | session-এর ভেতরে পূর্ণ-পর্দার একটা পাতা; নিচের status bar-এ তালিকা | ব্রাউজারের একটা ট্যাব |
| **Pane** | window-কে ভাগ করা আয়তাকার অংশ; প্রতিটায় আলাদা shell | ট্যাবের ভেতরে split view |

```
┌────────────────────── tmux server ───────────────────────┐
│                                                          │
│  Session "webapp"                    Session "db"        │
│  ├── Window 0 "code"                 └── Window 0 "psql" │
│  │    ┌──────────────┬───────────┐                       │
│  │    │              │  Pane 1   │                       │
│  │    │   Pane 0     │  (tests)  │                       │
│  │    │   (vim)      ├───────────┤                       │
│  │    │              │  Pane 2   │                       │
│  │    │              │  (git)    │                       │
│  │    └──────────────┴───────────┘                       │
│  ├── Window 1 "server"  (npm run dev)                    │
│  └── Window 2 "logs"    (tail -f)                        │
│                                                          │
└──────────────────────────────────────────────────────────┘

 status bar:  [webapp] 0:code* 1:server- 2:logs        "host" 14:32 12-Nov
```

status bar-এ `*` = বর্তমান window, `-` = আগের window, `Z` = zoom করা pane, `#` = activity হয়েছে।

---

## ২৫.১৪ tmux: কমান্ড লাইন থেকে ব্যবহার

| কমান্ড | সংক্ষিপ্ত রূপ | কাজ |
|---|---|---|
| `tmux` | | নতুন নামহীন session (নাম হবে 0, 1, ...) |
| `tmux new-session -s work` | `tmux new -s work` | `work` নামে session |
| `tmux new -s work -n editor` | | প্রথম window-এর নাম `editor` |
| `tmux new -d -s job 'cmd'` | | detached অবস্থায় session চালু করে কমান্ড চালাও |
| `tmux new -A -s work` | | `work` থাকলে attach, না থাকলে তৈরি |
| `tmux list-sessions` | `tmux ls` | সব session-এর তালিকা |
| `tmux attach-session -t work` | `tmux a -t work` | attach |
| `tmux attach` | `tmux a` | সবচেয়ে সাম্প্রতিক session-এ attach |
| `tmux attach -d -t work` | | অন্য client-দের detach করে attach |
| `tmux detach` | | (tmux-এর ভেতর থেকে) detach |
| `tmux rename-session -t 0 work` | | session-এর নাম বদল |
| `tmux kill-session -t work` | | একটা session বন্ধ |
| `tmux kill-session -a` | | বর্তমানটা ছাড়া সব session বন্ধ |
| `tmux kill-server` | | সব session সহ server বন্ধ |
| `tmux new-window -n logs` | `tmux neww` | নতুন window |
| `tmux split-window -h` | `tmux splitw -h` | বাম-ডানে ভাগ |
| `tmux split-window -v` | | ওপর-নিচে ভাগ |
| `tmux send-keys -t work:0 'ls' Enter` | `tmux send` | কোনো pane-এ কী-স্ট্রোক পাঠাও |
| `tmux list-keys` | `tmux lsk` | সব key binding |
| `tmux show-options -g` | `tmux show -g` | global option দেখাও |
| `tmux source-file ~/.tmux.conf` | `tmux source` | কনফিগ পুনরায় লোড |
| `tmux capture-pane -p -t work` | | pane-এর লেখা stdout-এ বের করো |
| `tmux has-session -t work` | | session আছে কিনা (exit status 0/1) |

### Target নোটেশন (`-t`)

```
session:window.pane
```

যেমন `work:1.2` = session `work`, window 1, pane 2। `work:logs` = নাম দিয়ে window। শুধু `work` = session।

উদাহরণ:

```bash
$ tmux new -s work
# ... কাজ, তারপর Ctrl+B d
[detached (from session work)]
$ tmux ls
work: 3 windows (created Thu Nov 12 10:15:02 2026)
$ tmux a -t work
```

> 💡 **টিপস:** `tmux` কমান্ডের সব subcommand অস্পষ্ট না হলে সংক্ষেপে লেখা যায়: `tmux a`, `tmux ls`, `tmux new`, `tmux kill-ses -t x`।

---

## ২৫.১৫ tmux শর্টকাট: `Ctrl+B` prefix

tmux-এ সব শর্টকাটের আগে **prefix** চাপতে হয়: ডিফল্টভাবে **`Ctrl+B`**। পদ্ধতি:

1. `Ctrl` ধরে `B` চাপো
2. **দুটোই ছেড়ে দাও**
3. তারপর কমান্ড কী চাপো (যেমন `c`)

নিচে `C-b c` মানে ঠিক এটাই। `M-` মানে Alt (Meta) কী। `C-` মানে Ctrl।

> ⚠️ **সতর্কতা:** নতুনদের সবচেয়ে কমন ভুল হলো `Ctrl+B+c` একসাথে চেপে ধরা। `Ctrl+B` ছেড়ে দিয়ে **তারপর** `c` চাপো। আর `%` বা `"` টাইপ করতে Shift লাগে, সেটা স্বাভাবিকভাবে চাপবে।

### Session-সংক্রান্ত

| শর্টকাট | কাজ |
|---|---|
| `C-b d` | বর্তমান client **detach** |
| `C-b D` | কোন client detach করবে তা বেছে নাও |
| `C-b s` | session-এর গাছ (tree) থেকে বেছে নাও |
| `C-b $` | বর্তমান session-এর নাম বদলাও |
| `C-b (` | আগের session-এ যাও |
| `C-b )` | পরের session-এ যাও |
| `C-b L` | শেষ ব্যবহৃত session-এ যাও |
| `C-b C-z` | tmux client-কে suspend করো |

### Window-সংক্রান্ত

| শর্টকাট | কাজ |
|---|---|
| `C-b c` | নতুন window |
| `C-b ,` | বর্তমান window-এর নাম বদলাও |
| `C-b &` | বর্তমান window kill (নিশ্চিত করতে বলবে) |
| `C-b n` | পরের window |
| `C-b p` | আগের window |
| `C-b l` | শেষ ব্যবহৃত window (toggle) |
| `C-b 0` … `C-b 9` | ওই নম্বরের window |
| `C-b '` | নম্বর লিখে window-এ যাও (১০-এর বেশি window হলে) |
| `C-b w` | সব session ও window-এর গাছ, preview সহ, বেছে নাও |
| `C-b f` | লেখা দিয়ে window খোঁজো |
| `C-b .` | window-কে অন্য নম্বরে সরাও |
| `C-b i` | বর্তমান window-এর তথ্য দেখাও |

### Pane-সংক্রান্ত

| শর্টকাট | কাজ |
|---|---|
| `C-b %` | বাম-ডানে ভাগ (নতুন pane ডানে) |
| `C-b "` | ওপর-নিচে ভাগ (নতুন pane নিচে) |
| `C-b ←` `↑` `→` `↓` | তীর চিহ্নের দিকের pane-এ যাও |
| `C-b o` | পরের pane-এ যাও |
| `C-b ;` | শেষ ব্যবহৃত pane-এ যাও |
| `C-b q` | pane নম্বর দেখাও (দেখানোর সময় নম্বর চাপলে সেখানে যাবে) |
| `C-b x` | বর্তমান pane kill (নিশ্চিত করতে বলবে) |
| `C-b z` | বর্তমান pane **zoom** করো (পূর্ণ পর্দা) / আবার চাপলে ফেরত |
| `C-b !` | বর্তমান pane-কে আলাদা নতুন window বানাও |
| `C-b {` | বর্তমান pane-কে আগের pane-এর সাথে অদলবদল |
| `C-b }` | বর্তমান pane-কে পরের pane-এর সাথে অদলবদল |
| `C-b C-o` | সব pane সামনের দিকে ঘোরাও (rotate) |
| `C-b M-o` | সব pane উল্টো দিকে ঘোরাও |
| `C-b Space` | পরের preset layout-এ যাও |
| `C-b M-1` | layout: even-horizontal (সব pane পাশাপাশি সমান) |
| `C-b M-2` | layout: even-vertical (সব pane ওপর-নিচে সমান) |
| `C-b M-3` | layout: main-horizontal (ওপরে একটা বড়, নিচে বাকিরা) |
| `C-b M-4` | layout: main-vertical (বামে একটা বড়, ডানে বাকিরা) |
| `C-b M-5` | layout: tiled (গ্রিড) |
| `C-b E` | pane-গুলো সমানভাবে ছড়িয়ে দাও (tmux 3.1+) |
| `C-b m` | বর্তমান pane চিহ্নিত (mark) করো |
| `C-b M` | চিহ্ন মুছে দাও |

### Pane Resize

| শর্টকাট | কাজ |
|---|---|
| `C-b C-←/↑/→/↓` | ওই দিকে ১ ঘর করে আকার বদলাও (prefix-এর পর Ctrl ধরে তীর; বারবার চাপা যায়) |
| `C-b M-←/↑/→/↓` | ওই দিকে ৫ ঘর করে আকার বদলাও |
| `C-b :` তারপর `resize-pane -L 10` | বামে ১০ ঘর (`-R`, `-U`, `-D` অন্য দিক) |
| `C-b :` তারপর `resize-pane -Z` | zoom toggle (`C-b z`-এর সমান) |

> 💡 **টিপস:** resize-এর কীগুলো **repeatable**: একবার prefix চাপার পর `repeat-time`-এর মধ্যে (ডিফল্ট 500 মিলিসেকেন্ড) বারবার তীর চাপতে পারো, প্রতিবার prefix লাগবে না।

> ⚠️ **সতর্কতা:** অনেক টার্মিনাল বা ডেস্কটপ (যেমন GNOME-এ `Ctrl+Alt+←`) কিছু Ctrl/Alt-তীর কম্বিনেশন নিজেরাই খেয়ে ফেলে। কাজ না করলে `resize-pane` কমান্ড বা mouse ব্যবহার করো।

### Copy mode, buffer ও অন্যান্য

| শর্টকাট | কাজ |
|---|---|
| `C-b [` | **copy mode**-এ ঢোকো (scroll করা, খোঁজা, কপি) |
| `C-b PageUp` | copy mode-এ ঢুকে এক পাতা ওপরে scroll |
| `C-b ]` | সবচেয়ে সাম্প্রতিক buffer paste |
| `C-b #` | সব paste buffer-এর তালিকা |
| `C-b =` | তালিকা থেকে buffer বেছে paste |
| `C-b -` | সবচেয়ে সাম্প্রতিক buffer মুছে দাও |
| `C-b :` | tmux কমান্ড prompt |
| `C-b ?` | **সব key binding-এর তালিকা** (q দিয়ে বেরোও) |
| `C-b t` | বড় ঘড়ি দেখাও (যেকোনো কী চাপলে বন্ধ) |
| `C-b r` | client-এর পর্দা নতুন করে আঁকো (redraw) |
| `C-b ~` | tmux-এর আগের বার্তাগুলো দেখাও |
| `C-b C-b` | প্রোগ্রামের কাছে আসল `Ctrl+B` পাঠাও |

> 📝 **মনে রাখো:** কোনো শর্টকাট ভুলে গেলে শুধু `C-b ?` চাপো। এটাই tmux-এর বিল্ট-ইন চিটশিট।

### Mouse

ডিফল্টভাবে mouse বন্ধ। `C-b :` চেপে লেখো `set -g mouse on`। তাহলে:
- pane-এ ক্লিক করে বেছে নেওয়া
- pane-এর সীমানা টেনে resize
- status bar-এ window-এ ক্লিক
- চাকা ঘুরিয়ে scroll (স্বয়ংক্রিয়ভাবে copy mode)

> 💡 **টিপস:** mouse চালু থাকলে টার্মিনালের সাধারণ মাউস-সিলেকশন tmux নিয়ে নেয়। টার্মিনালের নিজস্ব সিলেকশন চাইলে **Shift** চেপে ধরে টানো (বেশিরভাগ টার্মিনালে কাজ করে)।

---

## ২৫.১৬ tmux Copy Mode বিস্তারিত

pane-এর বাইরে চলে যাওয়া পুরনো আউটপুট দেখতে, খুঁজতে আর কপি করতে copy mode লাগে। ডিফল্টভাবে tmux প্রতিটা pane-এর ২০০০ লাইন মনে রাখে (`history-limit`)।

copy mode-এর কী-গুলো দুই ধরনের: **emacs** (ডিফল্ট) আর **vi**। `EDITOR` বা `VISUAL` environment variable-এ `vi` থাকলে tmux নিজে থেকেই vi ধরনে চলে। নিশ্চিত করতে `.tmux.conf`-এ লেখো `setw -g mode-keys vi`।

| কাজ | vi mode | emacs mode |
|---|---|---|
| copy mode-এ ঢোকা | `C-b [` | `C-b [` |
| বেরোনো | `q` | `q` বা `Esc` |
| চলাচল | `h` `j` `k` `l` / তীর | তীর |
| এক পাতা ওপরে/নিচে | `C-b` / `C-f` | `PageUp` / `PageDown` |
| অর্ধ পাতা | `C-u` / `C-d` | `M-Up` / `M-Down` |
| ইতিহাসের একদম ওপরে/নিচে | `g` / `G` | `M-<` / `M->` |
| লাইনের শুরু/শেষ | `0` / `$` | `C-a` / `C-e` |
| পরের/আগের শব্দ | `w` / `b` | `M-f` / `M-b` |
| সামনে খোঁজা | `/` | `C-s` |
| পেছনে খোঁজা | `?` | `C-r` |
| পরের/আগের মিল | `n` / `N` | `n` / `N` |
| সিলেকশন শুরু | `Space` | `C-Space` |
| আয়তাকার (block) সিলেকশন | `v` (toggle) | `R` |
| কপি করে বেরোনো | `Enter` | `M-w` |
| সিলেকশন বাতিল | `Esc` | `C-g` |

উদাহরণ (vi mode): একটা error লাইন কপি করে অন্য pane-এ paste

```
C-b [          → copy mode
?ERROR Enter   → পেছন দিকে "ERROR" খোঁজো
0              → লাইনের শুরু
Space          → সিলেকশন শুরু
$              → লাইনের শেষ পর্যন্ত
Enter          → কপি, copy mode থেকে বেরোও
C-b ;          → আগের pane-এ যাও
C-b ]          → paste
```

অনেকে vim-এর মতো `v` দিয়ে সিলেকশন আর `y` দিয়ে কপি পছন্দ করে। `.tmux.conf`-এ:

```bash
setw -g mode-keys vi
bind -T copy-mode-vi v send-keys -X begin-selection
bind -T copy-mode-vi y send-keys -X copy-selection-and-cancel
bind -T copy-mode-vi C-v send-keys -X rectangle-toggle
```

> 💡 **টিপস:** tmux-এর buffer আর সিস্টেমের clipboard আলাদা। tmux 3.2+ এ `set -g set-clipboard on` থাকলে এবং টার্মিনাল OSC 52 সমর্থন করলে কপি সিস্টেম clipboard-এ যায়। নইলে X11-এ `copy-pipe-and-cancel 'xclip -selection clipboard -in'` বা Wayland-এ `wl-copy` ব্যবহার করো।

pane-এর পুরো ইতিহাস ফাইলে সেভ করতে:

```bash
$ tmux capture-pane -p -S - -t work:0.0 > pane-history.txt
```

(`-S -` মানে ইতিহাসের একদম শুরু থেকে।)

---

## ২৫.১৭ `.tmux.conf`: কাস্টমাইজেশন

tmux চালু হওয়ার সময় কনফিগ পড়ে `~/.tmux.conf` থেকে (tmux 3.1+ এ `~/.config/tmux/tmux.conf`-ও কাজ করে)। সব ইউজারের জন্য `/etc/tmux.conf`।

> 📝 **মনে রাখো:** কনফিগ শুধু **server** চালু হওয়ার সময় পড়া হয়। বদলানোর পর হয় `tmux source-file ~/.tmux.conf` চালাও, নয়তো সব session বন্ধ করে (`tmux kill-server`) আবার শুরু করো। নতুন `tmux` উইন্ডো খুললেই কনফিগ পড়া হয় না যদি server আগে থেকে চলছে।

### ধাপে ধাপে একটা ভালো কনফিগ

```bash
# ~/.tmux.conf

# ---------- prefix: Ctrl+B থেকে Ctrl+A (screen-এর মতো, হাতের কাছে) ----------
unbind C-b
set -g prefix C-a
bind C-a send-prefix          # C-a C-a চাপলে প্রোগ্রাম আসল Ctrl+A পাবে

# ---------- সাধারণ ----------
set -g mouse on                       # mouse চালু
set -g history-limit 50000            # বড় scrollback
set -g base-index 1                   # window নম্বর 1 থেকে (কিবোর্ডে 0 দূরে)
setw -g pane-base-index 1             # pane নম্বরও 1 থেকে
set -g renumber-windows on            # window বন্ধ হলে নম্বর আবার সাজাও
set -sg escape-time 10                # Esc-এর দেরি কমাও (vim ব্যবহারকারীদের জন্য জরুরি)
set -g default-terminal "tmux-256color"
set -ga terminal-overrides ",*256col*:Tc"   # true color
set -g focus-events on
setw -g mode-keys vi                  # copy mode-এ vi কী

# ---------- সহজ split: | আর -, বর্তমান ডিরেক্টরিতেই নতুন pane ----------
unbind '"'
unbind %
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
bind c new-window -c "#{pane_current_path}"

# ---------- vim-এর মতো pane চলাচল ----------
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# ---------- resize (-r = repeatable) ----------
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5

# ---------- কনফিগ reload ----------
bind r source-file ~/.tmux.conf \; display-message "কনফিগ reload হয়েছে"

# ---------- prefix ছাড়াই Alt+তীর দিয়ে pane বদল ----------
bind -n M-Left  select-pane -L
bind -n M-Right select-pane -R
bind -n M-Up    select-pane -U
bind -n M-Down  select-pane -D

# ---------- copy mode vim-এর মতো ----------
bind -T copy-mode-vi v send-keys -X begin-selection
bind -T copy-mode-vi y send-keys -X copy-selection-and-cancel

# ---------- status bar ----------
set -g status-position bottom
set -g status-interval 5
set -g status-style "bg=colour235,fg=colour250"
set -g status-left "#[fg=green,bold][#S] "
set -g status-left-length 30
set -g status-right "#[fg=yellow]#H #[fg=cyan]%Y-%m-%d %H:%M"
setw -g window-status-current-style "fg=black,bg=green,bold"
setw -g monitor-activity on
```

বুঝে নাও কয়েকটা জিনিস:
- `set -g` = global session option; `setw -g` (`set-window-option`) = window option; `set -s` = server option। tmux-এর নতুন সংস্করণে `set` নিজেই option-এর ধরন বুঝে নেয়, তবু এভাবে লেখা প্রচলিত।
- `bind KEY CMD` = prefix-এর পরে KEY চাপলে CMD; `bind -n` = prefix **ছাড়াই**; `bind -r` = repeatable; `bind -T copy-mode-vi` = নির্দিষ্ট key table-এ।
- `#S` = session নাম, `#H` = hostname, `#I` = window index, `#W` = window নাম, `#{pane_current_path}` = pane-এর বর্তমান ডিরেক্টরি। এগুলোকে বলে **format**।
- `bind ... \; ...` = একসাথে দুটো কমান্ড।

> ⚠️ **সতর্কতা:** prefix `C-a` করলে bash-এ লাইনের শুরুতে যাওয়ার `Ctrl+A` ([অধ্যায় ৬](06-terminal-shortcuts.md)) আর সরাসরি কাজ করবে না; তখন `C-a C-a` চাপতে হবে (ওপরের `send-prefix` binding-এর জন্য)। অনেকে এ কারণে `C-b` রেখে দেয় বা `C-Space` ব্যবহার করে।

### Plugin (ঐচ্ছিক): TPM

**TPM** (Tmux Plugin Manager) দিয়ে plugin বসানো যায়, যেমন `tmux-resurrect` (রিবুটের পরেও session-এর layout ফিরিয়ে আনা)।

```bash
$ git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

`.tmux.conf`-এর **একদম শেষে**:

```bash
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'
run '~/.tmux/plugins/tpm/tpm'
```

তারপর tmux-এর ভেতরে `C-b I` (বড় হাতের I) চাপলে plugin ইনস্টল হবে। tmux-resurrect-এ `C-b C-s` = সেভ, `C-b C-r` = রিস্টোর।

> 📝 **মনে রাখো:** রিবুট হলে tmux server মারা যায়, তাই ভেতরের সব প্রোগ্রামও। tmux রিবুট থেকে বাঁচায় না; resurrect শুধু layout আর কিছু প্রোগ্রাম আবার চালু করে।

---

## ২৫.১৮ tmux Scripting: এক কমান্ডে পুরো কাজের পরিবেশ

প্রতিদিন একই layout হাতে বানানোর বদলে একটা স্ক্রিপ্ট লেখো:

```bash
#!/usr/bin/env bash
# dev.sh — প্রজেক্টের tmux পরিবেশ তৈরি
SESSION=webapp
DIR=~/projects/webapp

# আগে থেকে থাকলে শুধু attach
if tmux has-session -t "$SESSION" 2>/dev/null; then
    exec tmux attach -t "$SESSION"
fi

tmux new-session  -d -s "$SESSION" -n code -c "$DIR"
tmux send-keys    -t "$SESSION:code" 'vim .' Enter
tmux split-window -h -t "$SESSION:code" -c "$DIR" -l 40%
tmux send-keys    -t "$SESSION:code.1" 'git status' Enter

tmux new-window   -t "$SESSION" -n server -c "$DIR"
tmux send-keys    -t "$SESSION:server" 'npm run dev' Enter

tmux new-window   -t "$SESSION" -n logs
tmux send-keys    -t "$SESSION:logs" 'journalctl -f' Enter

tmux select-window -t "$SESSION:code"
tmux select-pane   -t "$SESSION:code.0"
exec tmux attach -t "$SESSION"
```

> 📝 **মনে রাখো:** pane নম্বর `pane-base-index`-এর ওপর নির্ভর করে। ওপরের স্ক্রিপ্ট ডিফল্ট (০ থেকে শুরু) ধরে লেখা; `pane-base-index 1` থাকলে `.0`→`.1`, `.1`→`.2` করো। `-l 40%` শতাংশ হিসেবে আকার দেওয়া tmux 3.1+ এ কাজ করে।

### অনেক সার্ভারে একসাথে টাইপ: `synchronize-panes`

চারটা pane-এ চারটা সার্ভারে SSH করে একই কমান্ড একসাথে চালাতে:

```
C-b :  setw synchronize-panes on
```

এখন যা টাইপ করবে সব pane-এ যাবে। বন্ধ করতে `setw synchronize-panes off`।

> ⚠️ **সতর্কতা:** synchronize চালু রেখে ভুলে `sudo reboot` বা `rm` টাইপ করলে **সব** সার্ভারে চলবে। কাজ শেষ হলেই বন্ধ করো।

### Pair programming: একই session দুজন

একই ইউজার দুই টার্মিনাল থেকে `tmux a -t work` দিলে দুজনেই একই পর্দা দেখবে ও টাইপ করতে পারবে। আলাদা window দেখতে চাইলে **grouped session**:

```bash
$ tmux new -t work -s work-friend     # work-এর window শেয়ার করে, কিন্তু নিজের বর্তমান window
```

---

## ২৫.১৯ screen বনাম tmux

| বিষয় | screen | tmux |
|---|---|---|
| ডিফল্ট prefix | `Ctrl+A` | `Ctrl+B` |
| detach | `C-a d` | `C-b d` |
| নতুন window | `C-a c` | `C-b c` |
| পরের/আগের window | `C-a n` / `C-a p` | `C-b n` / `C-b p` |
| window তালিকা | `C-a "` | `C-b w` |
| window নাম বদল | `C-a A` | `C-b ,` |
| ওপর-নিচে split | `C-a S` | `C-b "` |
| বাম-ডানে split | `C-a \|` | `C-b %` |
| region/pane বদল | `C-a Tab` | `C-b o` / তীর |
| copy mode | `C-a [` | `C-b [` |
| paste | `C-a ]` | `C-b ]` |
| সাহায্য | `C-a ?` | `C-b ?` |
| session তালিকা | `screen -ls` | `tmux ls` |
| attach | `screen -r name` | `tmux a -t name` |
| detach-এর পর split টেকে? | না | হ্যাঁ |
| scripting | সীমিত (`-X`) | খুব শক্তিশালী |
| serial console (`screen /dev/ttyUSB0 115200`) | হ্যাঁ | না |
| RHEL 9 base repo-তে | না (EPEL) | হ্যাঁ |

> 💡 **টিপস:** screen-এর একটা অনন্য কাজ আছে: serial port-এ সংযোগ। রাউটার, সুইচ বা Raspberry Pi-এর serial console-এ ঢুকতে `screen /dev/ttyUSB0 115200` এখনো জনপ্রিয়। বেরোতে `C-a k`।

---

## ২৫.২০ বাস্তব পরিস্থিতি: কোন টুল কখন

| পরিস্থিতি | সমাধান |
|---|---|
| vim থেকে এক মুহূর্তের জন্য shell-এ | `Ctrl+Z` … `fg` |
| `&` দিতে ভুলে গেছ | `Ctrl+Z`, `bg` |
| চালু কাজ রেখে লগআউট করতে হবে | `Ctrl+Z`, `bg`, `disown -h` |
| SSH-এ ২ ঘণ্টার database migration | `tmux new -s migrate` → চালাও → `C-b d` |
| সংযোগ কেটে যেতে পারে এমন দুর্বল Wi-Fi | সবসময় tmux-এর ভেতরে কাজ করো |
| স্ক্রিপ্ট থেকে একটা কাজ ছেড়ে দিতে, আউটপুট লগে | `nohup cmd > log 2>&1 &` |
| টার্মিনাল থেকে GUI অ্যাপ খুলে টার্মিনাল বন্ধ | `setsid -f app` |
| সবসময় চলবে, রিবুটেও ফিরে আসবে | systemd service ([অধ্যায় ২৬](26-systemd-services.md)) |
| নির্দিষ্ট সময়ে চালাতে হবে | cron/systemd timer ([অধ্যায় ২৯](29-scheduling-cron.md)) |

> 💡 **টিপস:** SSH লগইনের সাথে সাথে tmux-এ ঢুকতে অনেকে `~/.bashrc`-এর শেষে লেখে:
> ```bash
> if [[ -n "$SSH_CONNECTION" && -z "$TMUX" && $- == *i* ]]; then
>     tmux new -A -s main
> fi
> ```
> `$TMUX` variable tmux-এর ভেতরে থাকলে সেট থাকে, তাই tmux-এর ভেতরে tmux (nesting) হবে না। `$- == *i*` নিশ্চিত করে শুধু interactive shell-এ চলবে (scp/rsync ভাঙবে না)।

> ⚠️ **সতর্কতা:** tmux-এর ভেতরে আবার tmux চালালে tmux সতর্ক করে: `sessions should be nested with care, unset $TMUX to force`। nested অবস্থায় ভেতরের tmux-এ prefix পাঠাতে `C-b C-b` (দুবার) চাপতে হয়।

---

## হাতে-কলমে অনুশীলন

### ল্যাব ১: job control

```bash
$ sleep 1000 &
$ sleep 2000 &
$ top                      # চালু হলে Ctrl+Z
$ jobs -l                  # তিনটা job, PID সহ
$ fg %top                  # top ফেরত আনো, তারপর আবার Ctrl+Z
$ bg %3                    # top-কে background-এ (দেখবে আবার Stopped হয়ে যাবে — কেন?)
$ kill %1 %2               # দুটো sleep মারো
$ jobs
$ fg                       # top-এ গিয়ে q দিয়ে বেরোও
```

প্রশ্ন: `bg` দেওয়ার পরেও top কেন stopped হলো? (উত্তর: top টার্মিনাল নিয়ন্ত্রণ করতে চায়/কিবোর্ড পড়তে চায়, background থেকে তা করলে kernel SIGTTIN/SIGTTOU দিয়ে থামিয়ে দেয়।)

### ল্যাব ২: SIGHUP নিজের চোখে দেখা

1. একটা নতুন টার্মিনাল উইন্ডো খোলো, চালাও:
   ```bash
   $ sleep 600 &
   $ nohup sleep 700 > /dev/null 2>&1 &
   $ sleep 800 & disown
   $ setsid -f sleep 900
   ```
2. উইন্ডোটা **ক্রস চিহ্নে ক্লিক করে** বন্ধ করো।
3. অন্য টার্মিনালে দেখো:
   ```bash
   $ pgrep -a sleep
   2345 sleep 700
   2350 sleep 800
   2360 sleep 900
   ```
   `sleep 600` মারা গেছে (SIGHUP), বাকি তিনটা বেঁচে আছে।
4. পরিষ্কার করো: `pkill -x sleep`

### ল্যাব ৩: screen বেসিক

```bash
$ screen -S lab
$ top                          # window 0-এ top
C-a c                          # নতুন window
$ ping -c 100 1.1.1.1          # window 1-এ ping
C-a "                          # তালিকা থেকে window 0 বেছে নাও
C-a A                          # নাম দাও "monitor"
C-a d                          # detach
$ screen -ls
$ screen -r lab                # ফেরত, সব চলছে
C-a \                          # সব বন্ধ (y দিয়ে নিশ্চিত)
```

### ল্যাব ৪: tmux সম্পূর্ণ চক্র

```bash
$ tmux new -s lab
C-b %                          # বাম-ডান ভাগ
C-b "                          # ডান অংশ ওপর-নিচে ভাগ
C-b ←                          # বামের pane-এ যাও
$ htop                         # (না থাকলে top)
C-b →                          # ডানের pane
$ watch -n1 date
C-b z                          # zoom, আবার C-b z দিয়ে ফেরত
C-b M-5                        # tiled layout
C-b c                          # নতুন window
C-b ,                          # নাম দাও "logs"
$ sudo journalctl -f
C-b 0                          # প্রথম window-এ ফেরত
C-b d                          # detach
$ tmux ls
lab: 2 windows (created ...)
$ tmux a -t lab                # সব ঠিক আছে, layout সহ
```

### ল্যাব ৫: SSH কেটে গেলেও কাজ বাঁচানো

1. কোনো সার্ভার বা VM-এ SSH করো (না থাকলে `ssh localhost`, [অধ্যায় ৩৮](38-ssh-remote.md))।
2. `tmux new -s longjob`
3. চালাও: `for i in $(seq 1 300); do echo "step $i"; sleep 1; done`
4. সংযোগ জোর করে কাটো: SSH-এর escape sequence `Enter` তারপর `~.` চাপো।
5. আবার SSH করো, `tmux a -t longjob`। দেখবে লুপ চলছেই।

### ল্যাব ৬: copy mode ও `.tmux.conf`

1. ওপরের ২৫.১৭-এর কনফিগ `~/.tmux.conf`-এ লেখো ([অধ্যায় ১৩](13-nano-editor.md) বা [১৪](14-vim.md) দিয়ে)।
2. `tmux source-file ~/.tmux.conf` (বা নতুন prefix-এর পর `r`)।
3. `seq 1 5000` চালাও, তারপর copy mode-এ ঢুকে `/2500` দিয়ে খোঁজো, `v` দিয়ে সিলেক্ট, `y` দিয়ে কপি, prompt-এ prefix + `]` দিয়ে paste করো।

### ল্যাব ৭: `dev.sh` স্ক্রিপ্ট

২৫.১৮-এর স্ক্রিপ্টটা নিজের একটা ফোল্ডারের জন্য বদলে লেখো, `chmod +x dev.sh` দিয়ে চালাও। দুবার চালিয়ে দেখো দ্বিতীয়বার শুধু attach হয়।

---

## সারসংক্ষেপ

### Job control

| কমান্ড/কী | কাজ |
|---|---|
| `cmd &` | background-এ চালাও |
| `$!` | শেষ background PID |
| `wait [id]` | শেষ হওয়া পর্যন্ত অপেক্ষা |
| `Ctrl+Z` | foreground job থামাও (SIGTSTP) |
| `Ctrl+C` | foreground job মারো (SIGINT) |
| `jobs -l` | job তালিকা PID সহ |
| `fg %n` | foreground-এ আনো |
| `bg %n` | background-এ চালু করো |
| `kill %n` | job-কে সিগন্যাল |
| `%+` `%-` `%n` `%str` `%?str` | job specification |
| `disown [-h] %n` | shell-এর SIGHUP থেকে মুক্ত |
| `nohup cmd > log 2>&1 &` | SIGHUP ignore করে চালাও |
| `setsid -f cmd` | নতুন session, টার্মিনাল ছাড়া |
| `shopt huponexit` | exit-এ SIGHUP পাঠাবে কিনা |
| `loginctl enable-linger` | লগআউটের পরেও ইউজার সার্ভিস |

### screen (`C-a`)

| কী | কাজ | কী | কাজ |
|---|---|---|---|
| `d` | detach | `c` | নতুন window |
| `n`/`p` | পরের/আগের | `"` | তালিকা |
| `A` | নাম বদল | `k` | window kill |
| `S`/`\|` | split | `Tab` | region বদল |
| `[`/`]` | copy/paste | `?` | সাহায্য |

### tmux (`C-b`)

| কী | কাজ | কী | কাজ |
|---|---|---|---|
| `d` | detach | `s` | session বাছাই |
| `$` | session নাম | `c` | নতুন window |
| `,` | window নাম | `&` | window kill |
| `n`/`p`/`l` | পরের/আগের/শেষ window | `w` | window tree |
| `%` | বাম-ডান split | `"` | ওপর-নিচে split |
| তীর / `o` / `;` | pane বদল | `x` | pane kill |
| `z` | zoom | `!` | pane → window |
| `{`/`}` | pane swap | `Space` / `M-1..5` | layout |
| `C-তীর` / `M-তীর` | resize ১/৫ | `[` / `]` | copy / paste |
| `:` | কমান্ড prompt | `?` | সব শর্টকাট |

| tmux কমান্ড | কাজ |
|---|---|
| `tmux new -s NAME` | নতুন session |
| `tmux ls` | তালিকা |
| `tmux a -t NAME` | attach |
| `tmux new -A -s NAME` | থাকলে attach, নইলে তৈরি |
| `tmux kill-session -t NAME` | session বন্ধ |
| `tmux kill-server` | সব বন্ধ |
| `tmux source-file ~/.tmux.conf` | কনফিগ reload |

---

## নিজেকে যাচাই করো

1. job আর process-এর মধ্যে পার্থক্য কী? `ls | sort | head`-এ কয়টা job আর কয়টা process?
2. `Ctrl+Z` আর `Ctrl+C`-এর মধ্যে কোন সিগন্যাল যায়, আর ফলাফলে কী পার্থক্য?
3. একটা কমান্ড foreground-এ চালু করে ফেলেছ, এখন তাকে background-এ পাঠিয়ে লগআউট করতে চাও। ধাপগুলো লেখো।
4. টার্মিনাল উইন্ডো বন্ধ করলে background job মারা যায়, কিন্তু `exit` লিখে বেরোলে অনেক সময় বাঁচে। কেন?
5. `nohup` আর `disown`-এর মূল পার্থক্য কী? কোনটা কখন ব্যবহার করবে?
6. `setsid` দিয়ে চালানো process-এর `ps`-এ `TT` কলামে `?` কেন দেখায়?
7. `KillUserProcesses=yes` সেট থাকলে লগআউটের পর tmux session-এর কী হবে? কীভাবে বাঁচাবে?
8. tmux-এ detach করলে প্রোগ্রাম কেন চলতে থাকে? architecture দিয়ে ব্যাখ্যা করো।
9. tmux-এ session, window আর pane-এর সম্পর্ক কী? একটা করে তৈরির শর্টকাট বা কমান্ড লেখো।
10. `.tmux.conf` বদলানোর পর নতুন টার্মিনালে `tmux` খুললেও পরিবর্তন দেখা যাচ্ছে না কেন? সমাধান কী?

<details>
<summary>উত্তর দেখো</summary>

1. process হলো kernel-এর চলমান প্রোগ্রাম (PID সহ); job হলো shell-এর দৃষ্টিতে একবারে চালানো একটা কমান্ড লাইন, যা একটা process group। `ls | sort | head`-এ **১টা job**, **৩টা process**।
2. `Ctrl+Z` → **SIGTSTP**, process থেমে থাকে (stopped), পরে `fg`/`bg` দিয়ে চালানো যায়। `Ctrl+C` → **SIGINT**, সাধারণত process শেষ হয়ে যায়।
3. `Ctrl+Z` → `bg` → `disown -h %1` (বা `disown %1`) → `exit`। (আউটপুট টার্মিনালে গেলে সমস্যা হতে পারে, তাই ভবিষ্যতে শুরুতেই redirect করো বা tmux ব্যবহার করো।)
4. উইন্ডো বন্ধ হলে kernel session leader (bash)-কে SIGHUP পাঠায়, আর bash SIGHUP পেয়ে বেরোনোর আগে সব job-কে SIGHUP পাঠায়। নিজে `exit` করলে bash running job-দের SIGHUP পাঠায় না, যদি না `huponexit` চালু থাকে।
5. `nohup` আলাদা প্রোগ্রাম; কমান্ড **শুরুর আগে** দিতে হয়, process নিজেই SIGHUP ignore করে এবং আউটপুট `nohup.out`-এ যায়। `disown` bash builtin; কমান্ড **চালুর পরে** দেওয়া যায়, শুধু bash-এর SIGHUP পাঠানোর তালিকা থেকে বাদ দেয়।
6. `setsid` নতুন session তৈরি করে যার কোনো controlling terminal নেই, তাই `TT` = `?`।
7. লগআউটের সাথে সাথে ওই session-এর scope-এর সব process মারা যাবে, tmux server-ও। বাঁচাতে `loginctl enable-linger $USER` চালু করো এবং `systemd-run --user --scope tmux new -s work` দিয়ে tmux-কে আলাদা scope-এ চালাও, অথবা অ্যাডমিন `KillUserProcesses=no` সেট করবে।
8. tmux client-server মডেলে চলে। server টার্মিনাল থেকে বিচ্ছিন্ন (নিজের session, `TT` = `?`) এবং প্রোগ্রামগুলোকে নিজের তৈরি pseudo-terminal-এ চালায়। detach-এ শুধু client বেরিয়ে যায়; server আর ভেতরের প্রোগ্রাম চলতে থাকে। আবার attach করলে নতুন client socket (`/tmp/tmux-UID/default`) দিয়ে সংযুক্ত হয়।
9. একটা session-এ অনেক window, একটা window-এ অনেক pane। session: `tmux new -s name`; window: `C-b c`; pane: `C-b %` বা `C-b "`।
10. tmux কনফিগ শুধু server চালু হওয়ার সময় পড়ে; server আগে থেকেই চলছে বলে নতুন client নতুন কনফিগ পড়ে না। সমাধান: `tmux source-file ~/.tmux.conf` বা `C-b :` এ `source-file ~/.tmux.conf`, অথবা `tmux kill-server` করে আবার শুরু।

</details>

---

[← আগের অধ্যায়](24-process-management.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](26-systemd-services.md)

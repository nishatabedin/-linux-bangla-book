# অধ্যায় ৪৪: এনভায়রনমেন্ট, .bashrc, alias ও শেল কাস্টমাইজেশন

## এই অধ্যায়ে যা শিখবে

- shell variable আর environment variable-এর পার্থক্য, আর `export` আসলে ভেতরে কী করে
- `env`, `printenv`, `set`, `unset`, `declare`, `readonly` দিয়ে variable দেখা ও নিয়ন্ত্রণ করা
- `PATH` দিয়ে শেল কীভাবে একটা কমান্ড খুঁজে বের করে, আর নিজের ফোল্ডার `PATH`-এ ঠিকভাবে যোগ করা
- গুরুত্বপূর্ণ variable: `HOME`, `USER`, `SHELL`, `PS1`, `LANG`, `EDITOR` এবং আরও কিছু
- login বনাম non-login shell, interactive বনাম non-interactive shell চেনা
- শেল চালু হলে কোন স্টার্টআপ ফাইল কোন ক্রমে পড়া হয় (`/etc/profile`, `~/.bash_profile`, `~/.profile`, `~/.bashrc`, `/etc/bash.bashrc`)
- `alias` ও `unalias` দিয়ে লম্বা কমান্ড ছোট করা
- `PS1` বদলে নিজের মতো রঙিন prompt বানানো
- zsh ও oh-my-zsh-এর পরিচিতি
- dotfiles কী, আর Git দিয়ে কীভাবে সেগুলো এক মেশিন থেকে আরেক মেশিনে নেওয়া যায়

---

## ৪৪.১ এনভায়রনমেন্ট কী, আর কেন এটা জানা দরকার

ধরো তুমি একটা নতুন অফিসে কাজ শুরু করলে। তোমার ডেস্কে একটা নোটবোর্ড আছে, তাতে লেখা: "তোমার নাম: রহিম", "প্রিন্টার: ৩ তলায়", "ভাষা: বাংলা", "জরুরি ফোন: ১০১"। অফিসে যে-ই তোমার হয়ে কোনো কাজ করে, সে আগে এই বোর্ডটা দেখে নেয়।

লিনাক্সে প্রতিটি process (চলমান প্রোগ্রাম, বিস্তারিত [অধ্যায় ২৪](24-process-management.md)-এ) এমনই একটা "নোটবোর্ড" নিয়ে চলে। একে বলে **environment** (এনভায়রনমেন্ট, পরিবেশ)। এটা আসলে কিছু `NAME=value` জোড়ার একটা তালিকা:

```
HOME=/home/rahim
USER=rahim
LANG=en_US.UTF-8
PATH=/usr/local/bin:/usr/bin:/bin
```

প্রোগ্রামগুলো এই তালিকা পড়ে ঠিক করে কীভাবে চলবে। যেমন:

- `ls` দেখে `LANG`/`LC_*`, তারপর ঠিক করে তারিখ কোন ফরম্যাটে দেখাবে
- `git commit` দেখে `EDITOR` বা `VISUAL`, তারপর ঠিক করে কোন এডিটর খুলবে
- শেল দেখে `PATH`, তারপর ঠিক করে `python3` লিখলে আসলে কোন ফাইলটা চালাবে

এই অধ্যায় শেষ করলে তুমি এই নোটবোর্ড পড়তে পারবে, লিখতে পারবে, আর প্রতিবার টার্মিনাল খুললে সেটা যেন নিজে থেকেই ঠিকমতো সাজানো থাকে, সেই ব্যবস্থাও করতে পারবে।

---

## ৪৪.২ Shell variable বনাম Environment variable

শেল স্ক্রিপ্টিং অধ্যায়ে ([অধ্যায় ৪১](41-shell-scripting-1.md)) তুমি variable বানানো শিখেছ:

```bash
$ name="Rahim"
$ echo "$name"
Rahim
```

এটা একটা **shell variable** (শেল ভ্যারিয়েবল)। এটা শুধু এই শেলের নিজের মেমোরিতে থাকে। এই শেল থেকে নতুন কোনো প্রোগ্রাম (child process) চালালে সে এটা দেখতে পায় না।

**Environment variable** (এনভায়রনমেন্ট ভ্যারিয়েবল) হলো এমন variable যেটা child process-এর কাছেও পৌঁছায়।

### হাতে-কলমে পার্থক্যটা দেখো

```bash
$ fruit="mango"          # শুধু shell variable
$ bash                   # নতুন child shell চালু করলাম
$ echo "fruit = $fruit"
fruit =
$ exit                   # child shell থেকে বের হলাম

$ export fruit           # এবার environment-এ তুলে দিলাম
$ bash
$ echo "fruit = $fruit"
fruit = mango
$ exit
```

প্রথমবার child shell কিছুই পায়নি। `export` করার পর পেয়েছে।

### ভেতরে কী হচ্ছে (under the hood)

লিনাক্সে নতুন প্রোগ্রাম চালু হয় দুই ধাপে: `fork()` আর `execve()` system call দিয়ে ([অধ্যায় ২](02-linux-architecture.md) ও [অধ্যায় ২৪](24-process-management.md) দেখো)। `execve()`-কে তিনটা জিনিস দিতে হয়: প্রোগ্রামের পাথ, argument তালিকা, আর **environment তালিকা** (`envp`)। Kernel এই তালিকাটা নতুন process-এর মেমোরিতে কপি করে দেয়।

শেল তার সব variable দুই ভাগে রাখে। যেগুলোর গায়ে "export" চিহ্ন আছে, শুধু সেগুলোই `envp` তালিকায় যায়।

```
        ┌────────────────────── bash (parent) ──────────────────────┐
        │                                                            │
        │   Shell variables (সব)          Exported? (environment)   │
        │   ───────────────────           ────────────────────────   │
        │   fruit=mango                   ✔  ──────────┐             │
        │   name=Rahim                    ✘            │             │
        │   PATH=/usr/bin:/bin            ✔  ──────────┤             │
        │   HOME=/home/rahim              ✔  ──────────┤             │
        │                                              │             │
        └──────────────────────────────────────────────┼─────────────┘
                                                       │ fork() + execve(path, argv, envp)
                                                       ▼  (শুধু ✔ চিহ্নিতগুলোর কপি)
                                   ┌──────── child process ────────┐
                                   │ fruit=mango                   │
                                   │ PATH=/usr/bin:/bin            │
                                   │ HOME=/home/rahim              │
                                   └───────────────────────────────┘
```

> 📝 **মনে রাখো:** environment হলো **কপি**, শেয়ার না। child যদি নিজের `fruit` বদলে ফেলে, parent-এর `fruit` বদলায় না। তথ্য সবসময় parent থেকে child-এর দিকে যায়, উল্টো দিকে কখনো না।

এই কারণেই একটা স্ক্রিপ্টের ভেতরে `export PATH=...` লিখে স্ক্রিপ্ট চালালে তোমার বর্তমান টার্মিনালের `PATH` বদলায় না। স্ক্রিপ্টটা আলাদা child process-এ চলে, আর শেষ হলে তার environment-ও হারিয়ে যায়। বর্তমান শেলে প্রভাব ফেলতে চাইলে স্ক্রিপ্টটা `source` করতে হয় (৪৪.৮ অংশে দেখবে)।

উপমা দিলে: বাবা ছেলেকে বাজারের একটা লিস্টের ফটোকপি দিলেন। ছেলে নিজের কপিতে কাটাকাটি করলে বাবার আসল লিস্ট বদলায় না।

### যেকোনো চলমান process-এর environment দেখা

প্রতিটি process-এর environment `/proc/<PID>/environ` ফাইলে দেখা যায় ([অধ্যায় ৪৭](47-kernel-proc-sys.md))। এখানে entry-গুলো NUL (`\0`) ক্যারেক্টার দিয়ে আলাদা থাকে, তাই `tr` দিয়ে নতুন লাইনে বদলে নাও:

```bash
$ tr '\0' '\n' < /proc/$$/environ | head -5
SHELL=/bin/bash
LANG=en_US.UTF-8
HOME=/home/rahim
USER=rahim
LOGNAME=rahim
```

`$$` মানে বর্তমান শেলের PID।

> ⚠️ **সতর্কতা:** `/proc/PID/environ` দেখায় process **শুরু হওয়ার সময়** তার environment কেমন ছিল। পরে শেলের ভেতরে `export` করলে এই ফাইলে সেটা দেখা যাবে না। বর্তমান অবস্থা দেখতে `env` বা `printenv` ব্যবহার করো।

---

## ৪৪.৩ Variable দেখা ও পরিচালনার কমান্ড

### `export` — variable-কে environment-এ পাঠানো

**Syntax:**

```bash
export NAME=value      # বানাও এবং export করো
export NAME            # আগে থেকে থাকা variable export করো
export -n NAME         # export চিহ্ন সরাও (variable থাকবে, শুধু shell variable হয়ে যাবে)
export -p              # সব exported variable দেখাও
export -f funcname     # একটা shell function export করো (bash-specific)
```

| অপশন | কাজ |
|------|-----|
| `-p` | সব exported variable `declare -x` ফরম্যাটে দেখায় |
| `-n` | export চিহ্ন তুলে দেয় |
| `-f` | variable না, function export করে (child bash পাবে) |

```bash
$ export EDITOR=nano
$ export -p | grep EDITOR
declare -x EDITOR="nano"
$ export -n EDITOR
$ export -p | grep EDITOR
$                         # কিছু নেই — কিন্তু $EDITOR এখনো শেলে আছে
$ echo $EDITOR
nano
```

> 💡 **টিপস:** `=`-এর দুই পাশে কোনো স্পেস দেবে না। `export A = 5` লিখলে শেল ভাববে তুমি `A`, `=` আর `5` নামের তিনটা জিনিস export করতে চাইছ, আর error দেবে।

### `env` — environment দেখা অথবা পরিবর্তিত environment-এ কমান্ড চালানো

`env` আসলে একটা আলাদা প্রোগ্রাম (`/usr/bin/env`, GNU coreutils-এর অংশ)। এর দুটো কাজ:

1. কোনো argument না দিলে বর্তমান environment প্রিন্ট করে
2. `env NAME=value command` দিলে **শুধু ওই কমান্ডের জন্য** environment বদলে কমান্ডটা চালায়

| অপশন | কাজ |
|------|-----|
| (কিছু না) | সব environment variable দেখায় |
| `-i` বা `-` | একদম খালি environment দিয়ে শুরু করে |
| `-u NAME` | ওই variable সরিয়ে দিয়ে কমান্ড চালায় |
| `-0` | প্রতিটা লাইন newline-এর বদলে NUL দিয়ে শেষ করে |
| `-C DIR` | কমান্ড চালানোর আগে DIR-এ যায় (coreutils 8.28+) |
| `-S "string"` | স্ট্রিংটাকে ভেঙে একাধিক argument বানায় (shebang-এ কাজে লাগে) |

```bash
$ env | head -3
SHELL=/bin/bash
PWD=/home/rahim
LOGNAME=rahim

$ env LANG=C date               # শুধু এই date কমান্ড ইংরেজি "C" locale-এ চলবে
Thu Nov 12 10:15:42 AM +06 2026

$ env -i bash -c 'echo "PATH=[$PATH] HOME=[$HOME]"'
PATH=[/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games] HOME=[]
```

শেষ উদাহরণে খেয়াল করো: `env -i` দিয়ে environment খালি করলেও bash নিজে থেকে একটা ডিফল্ট `PATH` বসিয়ে নিয়েছে, কিন্তু `HOME` খালি। কোনো স্ক্রিপ্ট "পরিষ্কার" environment-এ ঠিকমতো চলে কিনা, সেটা পরীক্ষা করতে `env -i` খুব কাজের। (bash-এর এই ডিফল্ট `PATH`-এর মান কম্পাইলের সময় ঠিক হয়, তাই distro ভেদে একটু আলাদা হতে পারে।)

শেলেও একই কাজ `env` ছাড়া করা যায়:

```bash
$ LANG=C date          # কমান্ডের আগে NAME=value লিখলে শুধু ওই কমান্ডের জন্য প্রযোজ্য
```

স্ক্রিপ্টের প্রথম লাইনে `#!/usr/bin/env python3` লেখার রহস্যও এটাই: `env` তোমার `PATH`-এ `python3` খুঁজে সেটা চালায়। ফলে python যে মেশিনে যেখানেই ইনস্টল থাকুক, স্ক্রিপ্ট চলবে।

### `printenv` — নির্দিষ্ট variable-এর মান

```bash
$ printenv HOME
/home/rahim
$ printenv HOME USER
/home/rahim
rahim
$ printenv NOT_EXIST; echo "exit code: $?"
exit code: 1
```

`printenv`-এর সুবিধা হলো variable না থাকলে exit code 1 দেয়, তাই স্ক্রিপ্টে পরীক্ষা করা সহজ। আর এটা **শুধু environment** দেখায়, export না করা shell variable দেখায় না:

```bash
$ secret=123
$ printenv secret; echo $?
1
$ echo $secret
123
```

### `set` — সবকিছু দেখা, এবং শেলের অপশন বদলানো

argument ছাড়া `set` চালালে **সব** shell variable (exported আর non-exported দুটোই) **এবং সব function** দেখায়। তাই আউটপুট অনেক লম্বা হয়:

```bash
$ set | wc -l
2873
$ set | less
```

(সংখ্যাটা তোমার মেশিনে আলাদা হবে, কারণ bash-completion অনেক function লোড করে।)

`set`-এর আসল আরেকটা কাজ হলো শেলের আচরণ বদলানো (স্ক্রিপ্টিং অধ্যায়ে `set -euo pipefail` দেখেছ, [অধ্যায় ৪৩](43-shell-scripting-3.md)):

| কমান্ড | কাজ |
|--------|-----|
| `set -o` | সব অপশন কোনটা on, কোনটা off দেখায় |
| `set -o vi` / `set -o emacs` | কমান্ড লাইন এডিটিং vi বা emacs মোডে ([অধ্যায় ৬](06-terminal-shortcuts.md)) |
| `set -o noclobber` বা `set -C` | `>` দিয়ে আগে থাকা ফাইল overwrite আটকায় |
| `set -x` | প্রতিটা কমান্ড চালানোর আগে প্রিন্ট করে (ডিবাগ) |
| `set -u` | অসংজ্ঞায়িত variable ব্যবহার করলে error |
| `set +o NAME` | অপশন বন্ধ করে (`+` মানে বন্ধ, `-` মানে চালু) |

### `env` / `printenv` / `set` / `declare` — কোনটা কী দেখায়

| কমান্ড | Exported variable | Shell-only variable | Function | ধরন |
|--------|:---:|:---:|:---:|-----|
| `env` | ✔ | ✘ | ✘ | বাইরের প্রোগ্রাম |
| `printenv` | ✔ | ✘ | ✘ | বাইরের প্রোগ্রাম |
| `export -p` | ✔ | ✘ | ✘ | শেল builtin |
| `set` | ✔ | ✔ | ✔ | শেল builtin |
| `declare -p` | ✔ | ✔ | ✘ | শেল builtin |
| `declare -x` | ✔ | ✘ | ✘ | শেল builtin |

> 📝 **মনে রাখো:** `env` আর `printenv` বাইরের প্রোগ্রাম। শেল এগুলো চালানোর সময় যা পাঠায় শুধু তা-ই এরা দেখে, অর্থাৎ শুধু exported variable। `set` শেলের নিজের অংশ (builtin), তাই শেলের ভেতরের সবকিছু দেখতে পায়।

### `unset` — variable বা function মুছে ফেলা

```bash
$ fruit=mango
$ unset fruit
$ echo "[$fruit]"
[]

$ greet() { echo hi; }
$ unset -f greet         # function মুছতে -f
$ unset -v fruit         # শুধু variable (ডিফল্ট আচরণও এটাই)
```

| অপশন | কাজ |
|------|-----|
| `-v` | variable মোছে |
| `-f` | function মোছে |

> ⚠️ **সতর্কতা:** `unset PATH` করলে বর্তমান শেলে `ls`, `cat` কিছুই খুঁজে পাবে না ("command not found")। ভয় পাওয়ার কিছু নেই, টার্মিনাল বন্ধ করে আবার খোলো, বা `export PATH=/usr/local/bin:/usr/bin:/bin` টাইপ করো। Builtin (`cd`, `echo`, `export`) তখনও কাজ করবে, কারণ ওগুলোর জন্য PATH লাগে না।

### `declare` ও `readonly`

```bash
$ declare -x COLOR=blue      # export করা variable বানানো (export COLOR=blue-এর সমান)
$ declare -r PI=3.14         # read-only
$ PI=3
bash: PI: readonly variable
$ readonly                   # সব readonly variable-এর তালিকা
$ declare -p HOME            # একটা variable-এর ধরন ও মান দেখা
declare -x HOME="/home/rahim"
```

`declare -p`-এর আউটপুটে `-x` মানে exported, `-r` মানে readonly, `-i` মানে integer, `-a` মানে indexed array, `-A` মানে associative array। readonly variable বর্তমান শেলে আর `unset` করা যায় না।

---

## ৪৪.৪ PATH — শেল কীভাবে কমান্ড খুঁজে পায়

### সমস্যাটা কী

তুমি `ls` লিখলে। কিন্তু `ls` আসলে একটা ফাইল, `/usr/bin/ls`। তুমি তো পুরো পাথ লেখোনি। তাহলে শেল জানল কীভাবে?

উত্তর `PATH` variable:

```bash
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

এটা কোলন (`:`) দিয়ে আলাদা করা কিছু ডিরেক্টরির তালিকা। শেল **বাম থেকে ডানে** একটা একটা করে ডিরেক্টরিতে খোঁজে, আর **প্রথম যেখানে পায়** সেটাই চালায়।

উপমা: তুমি চাবি খুঁজছ। প্রথমে পকেটে, তারপর ব্যাগে, তারপর টেবিলের ড্রয়ারে। পকেটে পেয়ে গেলে ব্যাগ আর খোলো না। ব্যাগে আরেকটা চাবি থাকলেও সেটা কাজে লাগে না।

### কমান্ড খোঁজার সম্পূর্ণ ক্রম

তুমি Enter চাপার পর bash নিচের ক্রমে খোঁজে:

```
 তুমি টাইপ করলে:  ll /tmp
            │
            ▼
 ┌──────────────────────────┐   শব্দে '/' আছে? (যেমন ./run.sh, /usr/bin/ls)
 │ ০. পাথ সরাসরি দেওয়া?     │── হ্যাঁ ──► সরাসরি ওই ফাইল চালাও, PATH দেখবে না
 └────────────┬─────────────┘
              │ না
              ▼
 ┌──────────────────────────┐
 │ ১. alias?                │── হ্যাঁ ──► alias-কে আসল লেখায় বদলে আবার শুরু
 └────────────┬─────────────┘
              ▼
 ┌──────────────────────────┐
 │ ২. keyword? (if, for...)  │── হ্যাঁ ──► শেলের ব্যাকরণ হিসেবে ধরো
 └────────────┬─────────────┘
              ▼
 ┌──────────────────────────┐
 │ ৩. shell function?       │── হ্যাঁ ──► function চালাও
 └────────────┬─────────────┘
              ▼
 ┌──────────────────────────┐
 │ ৪. builtin? (cd, echo)    │── হ্যাঁ ──► শেলের ভেতরেই চালাও (নতুন process নেই)
 └────────────┬─────────────┘
              ▼
 ┌──────────────────────────┐
 │ ৫. hash table-এ আছে?     │── হ্যাঁ ──► মনে রাখা পাথ ব্যবহার করো
 └────────────┬─────────────┘
              ▼
 ┌──────────────────────────────────────────────┐
 │ ৬. PATH-এর প্রতিটি ডিরেক্টরিতে বাম→ডান খোঁজো  │── পেলে ──► hash-এ রাখো, fork+execve
 └────────────┬─────────────────────────────────┘
              ▼ কোথাও নেই
   "command not found"  (exit code 127)
```

> 💡 **টিপস:** Ubuntu/Debian-এ কমান্ড না পেলে "Command 'x' not found, but can be installed with: sudo apt install ..." বার্তা আসে। এটা `command-not-found` প্যাকেজের কাজ, যেটা bash-এর `command_not_found_handle` function ব্যবহার করে। Fedora-তে একই কাজ করে `PackageKit-command-not-found`।

### `type`, `command -v`, `which`, `hash` — কমান্ড আসলে কোথা থেকে আসছে

```bash
$ type ls
ls is aliased to `ls --color=auto'
$ type cd
cd is a shell builtin
$ type -a echo
echo is a shell builtin
echo is /usr/bin/echo
echo is /bin/echo
$ type -t if
keyword
$ command -v python3
/usr/bin/python3
$ which python3
/usr/bin/python3
```

| কমান্ড | কী বলে | মন্তব্য |
|--------|--------|---------|
| `type NAME` | alias / keyword / function / builtin / file | সবচেয়ে নির্ভরযোগ্য (bash builtin) |
| `type -a NAME` | সব জায়গা যেখানে NAME পাওয়া যায় | PATH-এ একাধিক কপি ধরতে কাজে লাগে |
| `type -t NAME` | শুধু এক শব্দে ধরন | স্ক্রিপ্টে কাজে লাগে |
| `command -v NAME` | কোন পাথ চালানো হবে | POSIX, স্ক্রিপ্টে "আছে কি না" পরীক্ষায় সেরা |
| `which NAME` | PATH-এ ফাইলের পাথ | বাইরের প্রোগ্রাম, alias/builtin চেনে না |
| `hash` | এই সেশনে মনে রাখা পাথ ও কতবার চলেছে | |
| `hash -r` | মনে রাখা সব পাথ ভুলে যাও | |

`type -a echo`-তে `/usr/bin/echo` আর `/bin/echo` দুটোই আসার কারণ: আধুনিক Ubuntu, Debian, Fedora, RHEL-এ `/bin` আসলে `/usr/bin`-এর symlink ("usr merge", [অধ্যায় ৭](07-filesystem-hierarchy.md) দেখো)।

### Hash table: কেন মাঝে মাঝে "ভুল" প্রোগ্রাম চলে

প্রতিবার PATH-এর সব ডিরেক্টরি ঘাঁটলে সময় নষ্ট হয়। তাই bash একবার কোনো কমান্ড খুঁজে পেলে তার পাথ একটা hash table-এ মনে রাখে:

```bash
$ hash
hits	command
   3	/usr/bin/ls
   1	/usr/bin/git
```

ধরো তুমি `/usr/local/bin/node`-এ নতুন node ইনস্টল করলে, কিন্তু এই সেশনে আগে `/usr/bin/node` চালিয়েছিলে। bash পুরনো মনে রাখা পাথটা ব্যবহার করতে পারে। আর পুরনো ফাইলটা মুছে ফেললে bash নিজেই আবার খোঁজে। এ রকম গোলমাল হলে:

```bash
$ hash -r        # ভুলে যাও, নতুন করে খোঁজো
```

### PATH-এ নিজের ডিরেক্টরি যোগ করা

ধরো তোমার নিজের স্ক্রিপ্টগুলো `~/bin`-এ রাখো, আর চাও যেকোনো জায়গা থেকে শুধু নাম লিখে চালাতে।

**ধাপ ১: শুধু বর্তমান টার্মিনালে (সাময়িক)**

```bash
$ mkdir -p ~/bin
$ export PATH="$HOME/bin:$PATH"     # শুরুতে যোগ করলাম
$ echo $PATH
/home/rahim/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:...
```

টার্মিনাল বন্ধ করলেই এটা হারিয়ে যাবে।

**ধাপ ২: স্থায়ী করা**

লাইনটা `~/.bashrc`-এর শেষে যোগ করো (কেন `.bashrc`, সেটা ৪৪.৭ অংশে বুঝবে):

```bash
$ echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
$ source ~/.bashrc
```

**শুরুতে না শেষে?**

| লেখার ধরন | ফল |
|-----------|-----|
| `PATH="$HOME/bin:$PATH"` | তোমার ডিরেক্টরি আগে খোঁজা হবে। একই নামের সিস্টেম কমান্ডকে "ঢেকে" দেবে |
| `PATH="$PATH:$HOME/bin"` | সিস্টেম কমান্ড আগে। তোমারটা শুধু তখনই চলবে যখন সিস্টেমে ওই নামের কিছু নেই |

> 💡 **টিপস:** Ubuntu/Debian-এর ডিফল্ট `~/.profile`-এ আগে থেকেই এই কোড আছে: `~/bin` বা `~/.local/bin` থাকলে PATH-এ যোগ করে। তাই ফোল্ডারটা বানিয়ে একবার logout/login করলেই কাজ হয়ে যায়। Fedora/RHEL-এর ডিফল্ট `~/.bashrc`-ও `~/.local/bin` আর `~/bin` যোগ করে। `pip install --user` আর `pipx` প্রোগ্রাম রাখে `~/.local/bin`-এ।

**একই ডিরেক্টরি বারবার যোগ হওয়া ঠেকানো:**

`.bashrc` প্রতিটা নতুন শেলে চলে। তাই child shell খুললে বা বারবার `source` করলে PATH-এ একই ডিরেক্টরি জমতে থাকে। নিরাপদ উপায়:

```bash
# ~/.bashrc-এ
case ":$PATH:" in
    *":$HOME/bin:"*) ;;                       # আগে থেকেই আছে, কিছু করো না
    *) export PATH="$HOME/bin:$PATH" ;;
esac
```

শুরু আর শেষে `:` বসানোর কৌশলের কারণে PATH-এর প্রথম বা শেষ এন্ট্রিও সঠিকভাবে মিলবে।

> ⚠️ **সতর্কতা: PATH-এ কখনো `.` (বর্তমান ডিরেক্টরি) রাখবে না।** কেউ `/tmp`-এ `ls` নামের একটা ক্ষতিকর স্ক্রিপ্ট রেখে দিলে, আর তুমি `/tmp`-এ গিয়ে `ls` লিখলে, সেটা চলবে। এ কারণেই লিনাক্সে নিজের ফোল্ডারের স্ক্রিপ্ট চালাতে `./script.sh` লিখতে হয়। খালি এন্ট্রিও (`::`, বা শুরু/শেষে `:`) বর্তমান ডিরেক্টরি বোঝায়, তাই `PATH=":$PATH"` জাতীয় ভুলও বিপজ্জনক।

> ⚠️ **সতর্কতা:** `export PATH=$HOME/bin` (পুরনো `$PATH` ছাড়া) লিখলে বাকি সব ডিরেক্টরি হারিয়ে যাবে। সবসময় `$PATH` রাখো।

### সিস্টেম-জুড়ে PATH

- `/etc/environment`: Ubuntu/Debian-এ সিস্টেমের ডিফল্ট PATH এখানে থাকে। এটা **শেল স্ক্রিপ্ট নয়**। এটা পড়ে PAM-এর `pam_env` মডিউল লগইনের সময়। তাই এখানে শুধু `NAME="value"` লেখা যায়, `$PATH` বা `export` চলে না।
- `/etc/profile` আর `/etc/profile.d/*.sh`: সব ইউজারের login shell এগুলো চালায়। সব ইউজারের জন্য PATH যোগ করতে `/etc/profile.d/myapp.sh` নামে একটা ফাইল বানানোই প্রচলিত ও পরিষ্কার পদ্ধতি।
- `sudo`-এর PATH আলাদা: `/etc/sudoers`-এর `secure_path` সেটা ঠিক করে ([অধ্যায় ১৮](18-sudo-root.md))। তাই `~/bin`-এর কমান্ড `sudo mycmd` দিয়ে চালালে "command not found" আসতে পারে।

---

## ৪৪.৫ গুরুত্বপূর্ণ Environment Variable-গুলো

| Variable | কী রাখে | উদাহরণ মান | কে সেট করে |
|----------|---------|-----------|-----------|
| `HOME` | তোমার হোম ডিরেক্টরি | `/home/rahim` | `login`/PAM, `/etc/passwd` থেকে |
| `USER` | বর্তমান ইউজারনেম | `rahim` | login প্রক্রিয়া |
| `LOGNAME` | লগইন নাম (POSIX) | `rahim` | login প্রক্রিয়া |
| `SHELL` | তোমার **ডিফল্ট লগইন শেল** | `/bin/bash` | `/etc/passwd`-এর শেষ ঘর |
| `PATH` | কমান্ড খোঁজার ডিরেক্টরি তালিকা | `/usr/local/bin:/usr/bin:/bin` | PAM, `/etc/profile`, `.profile` |
| `PWD` | বর্তমান ডিরেক্টরি | `/home/rahim/docs` | শেল (প্রতিটা `cd`-তে) |
| `OLDPWD` | আগের ডিরেক্টরি (`cd -` এটা ব্যবহার করে) | `/tmp` | শেল |
| `PS1` | প্রধান prompt-এর চেহারা | `\u@\h:\w\$ ` | `.bashrc` |
| `LANG` | ভাষা, অঞ্চল ও ক্যারেক্টার এনকোডিং | `en_US.UTF-8`, `bn_BD.UTF-8` | locale সেটিং |
| `LC_ALL` | সব `LC_*` ও `LANG`-কে ছাপিয়ে যায় | `C` | সাধারণত ফাঁকা |
| `EDITOR` | ডিফল্ট টেক্সট এডিটর | `nano`, `vim` | তুমি |
| `VISUAL` | পূর্ণ-স্ক্রিন এডিটর (থাকলে `EDITOR`-এর আগে দেখা হয়) | `vim` | তুমি |
| `PAGER` | লম্বা আউটপুটের pager | `less` | তুমি |
| `TERM` | টার্মিনালের ধরন (রঙ, কী-কোড) | `xterm-256color` | টার্মিনাল এমুলেটর |
| `HOSTNAME` | মেশিনের নাম | `myserver` | bash নিজে |
| `UID` | তোমার ইউজার আইডি (readonly) | `1000` | bash নিজে |
| `HISTSIZE` / `HISTFILE` | হিস্ট্রির আকার ও ফাইল ([অধ্যায় ৬](06-terminal-shortcuts.md)) | `1000` / `~/.bash_history` | `.bashrc` |
| `TZ` | টাইমজোন | `Asia/Dhaka` | ঐচ্ছিক |
| `XDG_CONFIG_HOME` | কনফিগ ফাইলের জায়গা (না থাকলে `~/.config` ধরা হয়) | `~/.config` | ঐচ্ছিক |
| `DISPLAY` / `WAYLAND_DISPLAY` | GUI কোন ডিসপ্লেতে দেখাবে | `:0` / `wayland-0` | ডেস্কটপ সেশন ([অধ্যায় ৪](04-desktop-environment.md)) |

### `SHELL` নিয়ে একটা সাধারণ ভুল ধারণা

`$SHELL` **বর্তমানে চলমান শেল বলে না**। এটা বলে `/etc/passwd`-এ তোমার জন্য লেখা লগইন শেল কোনটা।

```bash
$ echo $SHELL
/bin/bash
$ zsh                  # এখন zsh-এর ভেতরে
% echo $SHELL
/bin/bash              # তবুও bash দেখাচ্ছে!
% echo $0
zsh                    # এটাই আসল চলমান শেল
% ps -p $$ -o comm=
zsh
```

ডিফল্ট লগইন শেল বদলাতে `chsh -s /usr/bin/zsh` (৪৪.১১ অংশ)।

### `LANG` ও locale

`LANG` ঠিক করে প্রোগ্রাম কোন ভাষায় মেসেজ দেখাবে, তারিখ-সংখ্যা কীভাবে লিখবে, sort কোন ক্রমে হবে, আর ক্যারেক্টার এনকোডিং কী হবে। আরও সূক্ষ্ম নিয়ন্ত্রণের জন্য আলাদা `LC_*` variable আছে।

```bash
$ locale
LANG=en_US.UTF-8
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
...
LC_ALL=
$ locale -a                # ইনস্টল করা সব locale
C
C.utf8
en_US.utf8
POSIX
```

অগ্রাধিকার: **`LC_ALL` > নির্দিষ্ট `LC_xxx` > `LANG`**।

| Variable | কী নিয়ন্ত্রণ করে |
|----------|------------------|
| `LC_CTYPE` | ক্যারেক্টারের ধরন ও এনকোডিং |
| `LC_NUMERIC` | দশমিক চিহ্ন (`.` না `,`) |
| `LC_TIME` | তারিখ ও সময়ের ফরম্যাট |
| `LC_COLLATE` | sort-এর ক্রম |
| `LC_MONETARY` | মুদ্রার ফরম্যাট |
| `LC_MESSAGES` | প্রোগ্রামের মেসেজের ভাষা |

> 💡 **টিপস:** স্ক্রিপ্টে আউটপুট parse করার সময় `LC_ALL=C` ব্যবহার করো। এতে মেসেজ ইংরেজিতে আর ফরম্যাট সবসময় একই রকম থাকে, আর `sort`/`grep` দ্রুত চলে। যেমন `LC_ALL=C sort file.txt`।

সিস্টেমের locale সেট করা: Ubuntu/Debian-এ `sudo locale-gen bn_BD.UTF-8` দিয়ে locale তৈরি করো, তারপর `sudo update-locale LANG=en_US.UTF-8`। systemd-ভিত্তিক সব distro-তে (Fedora/RHEL সহ) `localectl set-locale LANG=en_US.UTF-8`। Fedora/RHEL-এ নতুন ভাষা লাগলে `glibc-langpack-bn` জাতীয় প্যাকেজ ইনস্টল করতে হয়।

### `EDITOR` ও `VISUAL`

`git commit`, `crontab -e`, `visudo`, `sudoedit`, `less`-এ `v` চাপা — এরা সবাই এই variable দেখে কোন এডিটর খুলবে ঠিক করে।

```bash
$ export EDITOR=nano
$ export VISUAL=vim
```

ঐতিহাসিকভাবে `EDITOR` ছিল লাইন-এডিটরের (যেমন `ed`) জন্য, `VISUAL` পূর্ণ-স্ক্রিন এডিটরের জন্য। আজকাল বেশিরভাগ প্রোগ্রাম আগে `VISUAL` দেখে, না থাকলে `EDITOR`। দুটোতে একই মান দেওয়াই সহজ। Debian/Ubuntu-তে এগুলো না থাকলে `sensible-editor` ব্যবহার হয়, আর সিস্টেম-ডিফল্ট বদলানো যায় `sudo update-alternatives --config editor` দিয়ে, অথবা প্রথমবার `select-editor` জিজ্ঞেস করে।

---

## ৪৪.৬ শেলের চার রকম চরিত্র: Login / Non-login, Interactive / Non-interactive

স্টার্টআপ ফাইল বোঝার আগে জানতে হবে শেল কোন "মোডে" চালু হয়েছে। দুটো আলাদা প্রশ্ন আছে:

**প্রশ্ন ১: Interactive (ইন্টারঅ্যাকটিভ) কি না?** মানে, শেল কি তোমার সঙ্গে কথা বলছে (prompt দেখিয়ে কমান্ডের অপেক্ষা করছে), নাকি একটা স্ক্রিপ্ট চালিয়ে চুপচাপ শেষ হয়ে যাবে?

**প্রশ্ন ২: Login (লগইন) কি না?** মানে, এটা কি এই সেশনের প্রথম শেল, যেটা তোমার পরিচয় যাচাইয়ের (লগইন) ঠিক পরে চালু হয়েছে?

|  | **Interactive** | **Non-interactive** |
|---|---|---|
| **Login** | `ssh user@server`, tty-তে (Ctrl+Alt+F3) লগইন, `su - rahim`, `sudo -i`, `bash --login` | `bash -l script.sh`, `bash --login -c 'cmd'` (বিরল) |
| **Non-login** | ডেস্কটপে টার্মিনাল খোলা (GNOME Terminal-এর ডিফল্ট), টার্মিনালে `bash` টাইপ করা, `su rahim` (মাইনাস ছাড়া) | `bash script.sh`, `./script.sh`, cron job, `ssh server 'ls'` |

> 📝 **মনে রাখো:** `ssh server 'ls'` (কমান্ড সহ ssh) একটা **non-login, non-interactive** শেল চালায়। তবে bash-এর একটা বিশেষ নিয়ম আছে: সে বুঝতে পারলে যে ssh থেকে চালু হয়েছে, তখন non-interactive হলেও `~/.bashrc` পড়ে। এ কারণেই Ubuntu/Debian-এর `~/.bashrc`-এর শুরুতে একটা পাহারা থাকে, যেটা non-interactive হলে বাকি ফাইল পড়া বন্ধ করে দেয় (৪৪.৭)।

### নিজের শেল কোন মোডে, কীভাবে বুঝবে

```bash
# login কি না
$ shopt -q login_shell && echo "login" || echo "non-login"
non-login

# আরেক উপায়: $0-এর আগে '-' থাকলে login
$ echo $0
bash            # non-login
$ echo $0       # (ssh বা tty লগইনে)
-bash           # login

# interactive কি না: $- variable-এ 'i' অক্ষর আছে কি না
$ echo $-
himBHs
$ [[ $- == *i* ]] && echo "interactive"
interactive
```

`$0`-এর সামনের `-` চিহ্নটা আসে `login` বা `sshd` প্রোগ্রাম থেকে। এরা শেল চালানোর সময় ইচ্ছা করে argv[0]-এ `-bash` লিখে দেয়। শেল এটা দেখেই বোঝে সে login shell।

### কেন এই ভাগ?

ইতিহাসটা বুঝলে কারণটা পরিষ্কার হয়। আগেকার দিনে তুমি একবার লগইন করতে, আর সেই একটা শেলের ভেতর থেকেই সারা দিন কাজ করতে। তাই:

- **একবারই যা করা দরকার** (environment variable সেট করা, যেমন PATH; লগইনের বার্তা দেখানো) তা login shell-এ করা হতো। child-রা তো environment উত্তরাধিকারসূত্রে পাবেই।
- **প্রতিটা শেলে যা দরকার, কিন্তু উত্তরাধিকারসূত্রে আসে না** (alias, function, prompt, শেল অপশন) তা প্রতিটা interactive শেলে আলাদা করে করতে হতো।

মনে রাখো: alias আর shell function `execve()` দিয়ে child-এর কাছে যায় না, কারণ ওগুলো environment-এর অংশ না। তাই প্রতিটা নতুন শেলে এগুলো আবার লোড করতে হয়।

---

## ৪৪.৭ স্টার্টআপ ফাইল: কোনটা কখন পড়া হয়

### bash-এর নিয়ম (`man bash`-এর INVOCATION অংশ অনুযায়ী)

**Interactive login shell** (বা `--login` দিয়ে চালানো যেকোনো শেল):

1. `/etc/profile` পড়ে (থাকলে)
2. তারপর `~/.bash_profile`, `~/.bash_login`, `~/.profile` — এই তিনটার মধ্যে **প্রথম যেটা পাওয়া যায় শুধু সেটাই** পড়ে, বাকিগুলো উপেক্ষা করে
3. logout করার সময় `~/.bash_logout` পড়ে (আর থাকলে `/etc/bash.bash_logout`)

**Interactive non-login shell:**

1. সিস্টেম-ব্যাপী bashrc: Debian/Ubuntu-তে bash এমনভাবে কম্পাইল করা যে নিজেই `/etc/bash.bashrc` পড়ে। Fedora/RHEL-এ এটা নেই; সেখানে `~/.bashrc` নিজে `/etc/bashrc` source করে।
2. `~/.bashrc`

**Non-interactive shell** (স্ক্রিপ্ট):

- কোনো স্টার্টআপ ফাইল পড়ে না। শুধু `BASH_ENV` variable সেট থাকলে ওই ফাইলটা পড়ে।
- (ব্যতিক্রম: আগের অংশে বলা ssh-এর ক্ষেত্রে `~/.bashrc`।)

### ডায়াগ্রাম

```
                       ┌──────────────────────┐
                       │   bash চালু হলো       │
                       └──────────┬───────────┘
                                  │
                  ┌───────────────┴────────────────┐
                  │ login shell? (-bash, --login,   │
                  │ ssh, tty, su -, sudo -i)        │
                  └───────┬─────────────────┬──────┘
                      হ্যাঁ│                 │না
                          ▼                 ▼
        ┌──────────────────────────┐   ┌───────────────────────────┐
        │ /etc/profile              │   │ interactive?               │
        │   └► /etc/profile.d/*.sh  │   └──────┬──────────────┬──────┘
        │   └► /etc/bash.bashrc     │      হ্যাঁ│              │না (স্ক্রিপ্ট)
        │      (Ubuntu/Debian)      │          ▼              ▼
        └────────────┬─────────────┘  ┌────────────────────┐ ┌──────────────┐
                     ▼                │ /etc/bash.bashrc    │ │ $BASH_ENV    │
     ┌───────────────────────────────┐│ (Debian/Ubuntu)     │ │ (সেট থাকলে)   │
     │ প্রথম যেটা পাবে শুধু সেটাই:    ││        │            │ └──────────────┘
     │  ① ~/.bash_profile            ││        ▼            │
     │  ② ~/.bash_login              ││ ~/.bashrc           │
     │  ③ ~/.profile                 ││   └► (Fedora/RHEL:  │
     └───────────────┬───────────────┘│      /etc/bashrc)   │
                     │                └────────────────────┘
                     ▼
     ┌───────────────────────────────┐
     │ ~/.bashrc  ← নিজে থেকে পড়ে না!│
     │ .profile / .bash_profile-এর   │
     │ ভেতরে "source ~/.bashrc" লেখা │
     │ থাকে বলেই পড়া হয়              │
     └───────────────┬───────────────┘
                     ▼
              ... কাজ শেষে logout ...
                     ▼
              ~/.bash_logout
```

### সবচেয়ে গুরুত্বপূর্ণ কথা: login shell নিজে থেকে `~/.bashrc` পড়ে না

এটাই সবচেয়ে বেশি বিভ্রান্তি তৈরি করে। নিয়ম অনুযায়ী login shell `.bashrc` পড়ে **না**। তবুও ssh-এ ঢুকলে তোমার alias কাজ করে, কারণ distro-গুলো ডিফল্ট ফাইলে নিচের কোড আগে থেকেই বসিয়ে রাখে।

**Ubuntu/Debian `~/.profile`** (সংক্ষেপে):

```bash
# if running bash
if [ -n "$BASH_VERSION" ]; then
    # include .bashrc if it exists
    if [ -f "$HOME/.bashrc" ]; then
        . "$HOME/.bashrc"
    fi
fi

# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/bin" ] ; then
    PATH="$HOME/bin:$PATH"
fi
if [ -d "$HOME/.local/bin" ] ; then
    PATH="$HOME/.local/bin:$PATH"
fi
```

**Fedora/RHEL `~/.bash_profile`:**

```bash
# Get the aliases and functions
if [ -f ~/.bashrc ]; then
    . ~/.bashrc
fi
```

> ⚠️ **সতর্কতা: সবচেয়ে প্রচলিত ফাঁদ।** Ubuntu-তে কোনো টিউটোরিয়াল পড়ে তুমি একটা নতুন `~/.bash_profile` বানালে। এখন login shell `~/.bash_profile` পেয়ে গেছে, তাই `~/.profile` আর পড়ছে না। ফলে `.profile`-এর ভেতরের `.bashrc` লোড বন্ধ হয়ে গেল, আর ssh-এ তোমার সব alias আর রঙিন prompt উধাও! সমাধান: হয় `~/.bash_profile` মুছে দাও, নয়তো তার ভেতরে এই লাইন রাখো:
>
> ```bash
> [ -f ~/.profile ] && . ~/.profile
> ```

### Ubuntu-র `~/.bashrc`-এর শুরুর পাহারা

```bash
# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac
```

এর মানে: শেল interactive না হলে (`$-`-এ `i` নেই) এখানেই ফাইল পড়া থামাও। এটা না থাকলে `scp` বা `rsync` ভেঙে যেতে পারে। কারণ ওগুলো ssh দিয়ে non-interactive শেল চালায়, আর `.bashrc`-এর কোনো `echo "Welcome!"` তাদের ডেটা স্ট্রিমে ঢুকে প্রোটোকল নষ্ট করে দেয়।

> 💡 **টিপস:** তাই `.bashrc`-এ কিছু প্রিন্ট করতে চাইলে সেটা এই পাহারার **নিচে** লেখো। আর `export PATH=...` জাতীয় জিনিস যদি non-interactive ssh কমান্ডেও লাগে, তাহলে সেটা পাহারার **উপরে** রাখো।

### কোন জিনিস কোন ফাইলে রাখবে — ব্যবহারিক নিয়ম

| কী রাখতে চাও | কোথায় রাখবে | কারণ |
|--------------|-------------|------|
| `export PATH=...`, `export EDITOR=...` | `~/.profile` (বা `~/.bash_profile`), অথবা সরলতার জন্য `~/.bashrc` | একবার লাগে, child-রা পায় |
| `alias`, shell function | `~/.bashrc` | child-রা পায় না, প্রতিটা শেলে লাগে |
| `PS1`, `shopt`, `set -o`, completion | `~/.bashrc` | interactive শেলেরই বৈশিষ্ট্য |
| logout-এ স্ক্রিন পরিষ্কার, টেম্প মুছে ফেলা | `~/.bash_logout` | |
| সব ইউজারের জন্য environment | `/etc/profile.d/নাম.sh` | `/etc/profile` সরাসরি এডিট করলে আপডেটে ঝামেলা হয় |
| সব ইউজারের জন্য alias | `/etc/bash.bashrc` (Debian/Ubuntu) বা `/etc/bashrc` (Fedora/RHEL) | |
| GUI অ্যাপসহ সব জায়গার জন্য একটা variable | `~/.config/environment.d/*.conf` (systemd user session) বা `/etc/environment` | GUI-থেকে-চালানো অ্যাপ তোমার `.bashrc` পড়ে না |

> 📝 **মনে রাখো:** ডেস্কটপ থেকে আইকনে ক্লিক করে চালানো অ্যাপ (যেমন VS Code) কোনো শেলের ভেতর দিয়ে চালু হয় না, তাই তোমার `~/.bashrc`-এর `export` সেটা পায় না। GNOME-এ লগইনের সময় সাধারণত `~/.profile` পড়া হয় (distro ও display manager ভেদে আলাদা), আর systemd ভিত্তিক সেশনে `~/.config/environment.d/` নির্ভরযোগ্যভাবে কাজ করে। ওই ফাইলে `NAME=value` লেখো, শেল সিনট্যাক্স নয়।

### নিজে পরীক্ষা করে দেখো কোন ফাইল পড়া হলো

প্রতিটা ফাইলের শেষে একটা ট্রেস লাইন যোগ করো (পরীক্ষা শেষে মুছে দিও):

```bash
$ echo 'echo "[.profile পড়া হলো]"' >> ~/.profile
$ echo '[[ $- == *i* ]] && echo "[.bashrc পড়া হলো]"' >> ~/.bashrc

$ bash                   # non-login interactive
[.bashrc পড়া হলো]
$ exit
$ bash -l                # login interactive
[.bashrc পড়া হলো]       # .profile-এর ভেতর থেকে source হয়েছে, তাই আগে
[.profile পড়া হলো]
$ exit
$ bash -c 'true'         # non-interactive: কিছুই না
$
```

`bash -l`-এ `.bashrc`-এর লাইন আগে এল কেন? কারণ `.profile`-এর ভেতরে `.bashrc` source হয় ফাইলের মাঝামাঝি, আর আমাদের ট্রেস লাইনটা আছে `.profile`-এর একদম শেষে।

আরেকটা কাজের উপায়: `bash -lixc exit 2>&1 | less` চালালে login shell চালু হওয়ার সময় যা যা execute হয় সবকিছু (`set -x` ট্রেস) দেখা যাবে।

### `--norc`, `--noprofile`, `--rcfile`

| অপশন | কাজ |
|------|-----|
| `bash --norc` | `~/.bashrc` (ও সিস্টেম bashrc) পড়বে না। কনফিগে গোলমাল খুঁজতে কাজে লাগে |
| `bash --noprofile` | login shell হলেও `/etc/profile` আর `~/.bash_profile`… পড়বে না |
| `bash --rcfile FILE` | `~/.bashrc`-এর বদলে FILE পড়বে |
| `bash -l` / `bash --login` | login shell হিসেবে চালাও |
| `bash -i` | জোর করে interactive |

> 💡 **টিপস:** `.bashrc`-এ এমন ভুল করেছ যে টার্মিনাল খুললেই বন্ধ হয়ে যাচ্ছে? (যেমন `.bashrc`-এ `exit` লিখে ফেলেছ।) তাহলে tty-তে (Ctrl+Alt+F3) লগইন করো বা অন্য টার্মিনাল থেকে `bash --norc` চালাও, তারপর ফাইলটা ঠিক করো। Ubuntu-র মূল কপি `/etc/skel/.bashrc`-এ থাকে: `cp /etc/skel/.bashrc ~/` দিয়ে ডিফল্টে ফেরত যাওয়া যায়।

---

## ৪৪.৮ `source` (`.`) — পরিবর্তন সাথে সাথে চালু করা

`.bashrc` এডিট করার পর পরিবর্তন দেখতে নতুন টার্মিনাল খুলতে হয়। অথবা বর্তমান শেলেই ফাইলটা আবার পড়াও:

```bash
$ source ~/.bashrc
$ . ~/.bashrc          # একই জিনিস; '.' হলো POSIX নাম, 'source' bash-এর নাম
```

**`source file` বনাম `bash file` বনাম `./file`:**

```
  bash script.sh / ./script.sh            source script.sh
  ─────────────────────────────           ─────────────────
  ┌─────────┐                             ┌─────────────────────────┐
  │ তোমার    │── fork ─► ┌──────────┐     │ তোমার শেল নিজেই ফাইলের   │
  │ শেল      │           │ নতুন bash│     │ প্রতিটা লাইন পড়ে চালায় │
  └─────────┘           │ লাইন চালায়│     │                         │
       ▲                └─────┬────┘     │ export, cd, alias সব    │
       │  শেষ হলে সব পরিবর্তন  │          │ তোমার শেলেই থেকে যায়   │
       └──── হারিয়ে যায় ◄─────┘          └─────────────────────────┘
```

একটা স্ক্রিপ্ট যদি `cd` করে বা variable সেট করে, আর তুমি চাও সেটা তোমার শেলে থেকে যাক (যেমন Python-এর `source venv/bin/activate`), তাহলে `source` করতে হবে।

---

## ৪৪.৯ alias — লম্বা কমান্ডের ডাকনাম

ধরো তোমার বন্ধুর নাম "মোহাম্মদ আব্দুর রহমান চৌধুরী", কিন্তু তুমি তাকে ডাকো "রাজু"। alias হলো কমান্ডের এমন একটা ডাকনাম।

### alias তৈরি ও দেখা

```bash
$ alias ll='ls -alF'
$ ll
total 32
drwxr-x--- 5 rahim rahim 4096 Nov 12 10:20 ./
drwxr-xr-x 3 root  root  4096 Oct  1 09:00 ../
-rw-r--r-- 1 rahim rahim 3771 Oct  1 09:00 .bashrc
...

$ alias                 # সব alias দেখাও
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
alias grep='grep --color=auto'
alias l='ls -CF'
alias la='ls -A'
alias ll='ls -alF'
alias ls='ls --color=auto'

$ alias ll              # একটার মান দেখাও
alias ll='ls -alF'
```

উপরের তালিকাটা Ubuntu-র ডিফল্ট `~/.bashrc` থেকে আসে। Fedora/RHEL-এ `/etc/profile.d/colorls.sh` থেকে `ll='ls -l --color=auto'` আর root ইউজারের `/root/.bashrc`-এ `rm='rm -i'`, `cp='cp -i'`, `mv='mv -i'` থাকে।

**Syntax-এর নিয়ম:**

- `alias নাম='কমান্ড'` — `=`-এর দুই পাশে স্পেস দেবে না
- single quote (`'`) দাও, যাতে variable alias **ব্যবহারের সময়** expand হয়, alias **তৈরির সময়** না

```bash
$ alias here="echo $PWD"     # ভুল: এখনকার ডিরেক্টরি স্থির হয়ে গেল
$ alias here='echo $PWD'     # সঠিক: প্রতিবার চালানোর সময় দেখবে
```

### alias স্থায়ী করা

শেলে লেখা alias টার্মিনাল বন্ধ করলে হারিয়ে যায়। স্থায়ী করতে `~/.bashrc`-এ লেখো। Ubuntu-র `.bashrc` আলাদা একটা ফাইল খোঁজে:

```bash
# Ubuntu ~/.bashrc-এ আগে থেকেই আছে:
if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi
```

তাই Ubuntu/Debian-এ নিজের alias `~/.bash_aliases`-এ রাখাই পরিষ্কার পদ্ধতি:

```bash
$ nano ~/.bash_aliases
$ source ~/.bashrc
```

### কাজের কিছু alias

```bash
# ~/.bash_aliases

# নিরাপত্তা: overwrite/মোছার আগে জিজ্ঞেস করো
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# চলাফেরা
alias ..='cd ..'
alias ...='cd ../..'

# তথ্য
alias df='df -h'
alias du1='du -h --max-depth=1'
alias free='free -h'
alias ports='ss -tulpn'
alias myip='curl -s ifconfig.me; echo'

# প্যাকেজ (Ubuntu/Debian)
alias update='sudo apt update && sudo apt upgrade'
# Fedora/RHEL-এর জন্য:
# alias update='sudo dnf upgrade --refresh'

# git
alias gs='git status'
alias gl='git log --oneline --graph --decorate -20'

# sudo-র পরেও alias যেন কাজ করে (শেষের স্পেসটা খেয়াল করো)
alias sudo='sudo '
```

শেষ লাইনটা একটা সূক্ষ্ম কৌশল। bash-এর নিয়ম: alias-এর মান যদি **স্পেস দিয়ে শেষ হয়**, তাহলে পরের শব্দটাও alias কি না দেখা হয়। নইলে `sudo ll` লিখলে "ll: command not found" আসে, কারণ `sudo` তোমার শেলের alias চেনে না।

### alias এড়িয়ে আসল কমান্ড চালানো

```bash
$ \rm file.txt          # সামনে backslash: alias উপেক্ষা
$ command rm file.txt   # alias আর function দুটোই উপেক্ষা
$ /usr/bin/rm file.txt  # পুরো পাথ
$ 'rm' file.txt         # quote দিলেও alias expand হয় না
```

### `unalias`

```bash
$ unalias ll            # একটা মুছে ফেলো
$ unalias -a            # বর্তমান শেলের সব alias মুছে ফেলো
```

এটা শুধু বর্তমান শেলে কাজ করে। স্থায়ীভাবে সরাতে `.bashrc`/`.bash_aliases` থেকে লাইনটা মুছে দাও।

### alias-এর সীমাবদ্ধতা — কখন function লাগবে

alias হলো সাধারণ লেখা বদলে দেওয়া। এতে argument মাঝখানে বসানো যায় না:

```bash
$ alias mkcd='mkdir -p $1 && cd $1'    # কাজ করবে না! alias-এ $1 নেই
```

এ ক্ষেত্রে function বানাও ([অধ্যায় ৪২](42-shell-scripting-2.md)):

```bash
# ~/.bashrc
mkcd() {
    mkdir -p -- "$1" && cd -- "$1"
}

# যেকোনো আর্কাইভ খোলা
extract() {
    case "$1" in
        *.tar.gz|*.tgz)   tar xzf "$1" ;;
        *.tar.xz)         tar xJf "$1" ;;
        *.tar.bz2)        tar xjf "$1" ;;
        *.zip)            unzip "$1" ;;
        *.gz)             gunzip "$1" ;;
        *) echo "চিনি না: $1" >&2; return 1 ;;
    esac
}
```

| বৈশিষ্ট্য | alias | function |
|-----------|-------|----------|
| argument মাঝে বসানো | ✘ | ✔ (`$1`, `$@`) |
| একাধিক লাইন, if/loop | কষ্টকর | ✔ |
| স্ক্রিপ্টে (non-interactive) কাজ করে | ডিফল্টে ✘ | ✔ |
| child bash-এ পৌঁছায় | ✘ | `export -f` দিয়ে ✔ |
| সহজ ডাকনাম | ✔ সবচেয়ে সহজ | একটু বেশি লিখতে হয় |

> 📝 **মনে রাখো:** স্ক্রিপ্টে alias ডিফল্টে expand হয় না (চালু করতে `shopt -s expand_aliases` লাগে)। স্ক্রিপ্টে alias-এর ওপর ভরসা না করে পূর্ণ কমান্ড বা function লেখো।

---

## ৪৪.১০ PS1 — নিজের মতো Prompt

তোমার টার্মিনালের `rahim@laptop:~/docs$ ` লেখাটাই prompt। এটা আসে `PS1` variable থেকে:

```bash
$ echo "$PS1"
\[\e]0;\u@\h: \w\a\]${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$
```

দেখতে ভয়ের, কিন্তু টুকরো টুকরো করলে সহজ।

### Prompt-এর বিশেষ escape কোড

| কোড | মানে | উদাহরণ |
|-----|------|--------|
| `\u` | ইউজারনেম | `rahim` |
| `\h` | hostname-এর প্রথম অংশ (প্রথম `.` পর্যন্ত) | `laptop` |
| `\H` | পূর্ণ hostname | `laptop.home.lan` |
| `\w` | বর্তমান ডিরেক্টরি (হোমকে `~` দিয়ে) | `~/docs/notes` |
| `\W` | বর্তমান ডিরেক্টরির শুধু শেষ অংশ | `notes` |
| `\$` | সাধারণ ইউজারে `$`, root-এ `#` | `$` |
| `\d` | তারিখ | `Thu Nov 12` |
| `\t` | সময় ২৪-ঘণ্টা HH:MM:SS | `14:05:09` |
| `\T` | সময় ১২-ঘণ্টা HH:MM:SS | `02:05:09` |
| `\@` | ১২-ঘণ্টা am/pm | `02:05 PM` |
| `\A` | ২৪-ঘণ্টা HH:MM | `14:05` |
| `\D{format}` | `strftime` ফরম্যাটে সময় | `\D{%Y-%m-%d}` → `2026-11-12` |
| `\n` | নতুন লাইন | |
| `\s` | শেলের নাম | `bash` |
| `\v` / `\V` | bash version / পূর্ণ version | `5.2` / `5.2.21` |
| `\j` | ব্যাকগ্রাউন্ড job সংখ্যা ([অধ্যায় ২৫](25-jobs-tmux.md)) | `2` |
| `\!` | এই কমান্ডের history নম্বর | `501` |
| `\#` | এই সেশনের কমান্ড নম্বর | `12` |
| `\l` | টার্মিনাল ডিভাইসের নাম | `0` (pts/0 থেকে) |
| `\e` বা `\033` | ESC ক্যারেক্টার (রঙের কোড শুরু) | |
| `\[` ... `\]` | এর ভেতরের অংশ "অদৃশ্য", জায়গা নেয় না | রঙের কোডের চারপাশে **অবশ্যই** |

### ধাপে ধাপে prompt বানানো

```bash
$ PS1='\u@\h:\w\$ '
rahim@laptop:~/docs$

$ PS1='[\A] \W\$ '
[14:05] docs$

$ PS1='\u@\h \w\n\$ '      # দুই লাইনের prompt
rahim@laptop ~/docs/project/very/long/path
$
```

### রঙ যোগ করা (ANSI escape code)

রঙের কোডের গঠন: `\e[` + সংখ্যা + `m`। একাধিক সংখ্যা `;` দিয়ে দেওয়া যায়।

| কোড | মানে | কোড | মানে |
|-----|------|-----|------|
| `0` | সব রিসেট | `1` | মোটা (bold) |
| `4` | আন্ডারলাইন | `7` | উল্টো রঙ |
| `30` | কালো | `90` | উজ্জ্বল কালো (ধূসর) |
| `31` | লাল | `91` | উজ্জ্বল লাল |
| `32` | সবুজ | `92` | উজ্জ্বল সবুজ |
| `33` | হলুদ | `93` | উজ্জ্বল হলুদ |
| `34` | নীল | `94` | উজ্জ্বল নীল |
| `35` | বেগুনি (magenta) | `95` | উজ্জ্বল বেগুনি |
| `36` | সায়ান | `96` | উজ্জ্বল সায়ান |
| `37` | সাদা | `97` | উজ্জ্বল সাদা |
| `40`–`47` | পটভূমির রঙ | `38;5;N` | ২৫৬ রঙের N নম্বর |

```bash
PS1='\[\e[1;32m\]\u@\h\[\e[0m\]:\[\e[1;34m\]\w\[\e[0m\]\$ '
#    └─ সবুজ bold ─┘ user@host └রিসেট┘  └─ নীল bold ─┘ dir └রিসেট┘
```

> ⚠️ **সতর্কতা: `\[` আর `\]` ভুলে যেও না।** bash prompt-এর দৈর্ঘ্য গুনে ঠিক করে লাইন কখন ভাঙবে। রঙের কোড স্ক্রিনে দেখা যায় না, কিন্তু `\[ \]` না দিলে bash এগুলোকেও অক্ষর হিসেবে গোনে। ফলে লম্বা কমান্ড লিখলে লেখা আগের লাইনের ওপর চেপে বসে, আর Ctrl+A/Ctrl+E-তে কার্সর ভুল জায়গায় যায়।

### `tput` দিয়ে আরও পরিচ্ছন্ন রঙ

হাতে escape code না লিখে `tput` ব্যবহার করলে টার্মিনালের ধরন (`TERM`) অনুযায়ী ঠিক কোড পাওয়া যায়:

```bash
# ~/.bashrc
green=$(tput setaf 2); blue=$(tput setaf 4); bold=$(tput bold); reset=$(tput sgr0)
PS1='\[$bold$green\]\u@\h\[$reset\]:\[$bold$blue\]\w\[$reset\]\$ '
```

### root হলে লাল prompt

```bash
if [ "$EUID" -eq 0 ]; then
    PS1='\[\e[1;31m\]\u@\h:\w#\[\e[0m\] '      # root: লাল, সতর্ক থাকো
else
    PS1='\[\e[1;32m\]\u@\h\[\e[0m\]:\[\e[1;34m\]\w\[\e[0m\]\$ '
fi
```

### আগের কমান্ড সফল কি না, prompt-এ দেখানো

```bash
# ~/.bashrc
__prompt() {
    local ec=$?                              # সবার আগে exit code ধরে রাখো
    local mark='\[\e[32m\]✔\[\e[0m\]'
    [ $ec -ne 0 ] && mark="\[\e[31m\]✘ $ec\[\e[0m\]"
    PS1="$mark \[\e[1;34m\]\w\[\e[0m\]\$ "
}
PROMPT_COMMAND=__prompt
```

```bash
✔ ~$ ls /nonexistent
ls: cannot access '/nonexistent': No such file or directory
✘ 2 ~$
```

`PROMPT_COMMAND`-এ যা থাকে, bash প্রতিবার prompt দেখানোর **ঠিক আগে** সেটা চালায়। (bash 5.1 থেকে `PROMPT_COMMAND` একটা array-ও হতে পারে।)

### Git branch দেখানো

git-এর সাথে একটা স্ক্রিপ্ট আসে, যার `__git_ps1` function বর্তমান branch-এর নাম দেয়। Ubuntu/Debian-এ bash-completion চালু থাকলে এটা সাধারণত আপনা-আপনি লোড হয় (`/usr/lib/git-core/git-sh-prompt`)। Fedora/RHEL-এ থাকে `/usr/share/git-core/contrib/completion/git-prompt.sh`, সেটা source করতে হয়।

```bash
# ~/.bashrc (Fedora/RHEL-এ প্রথম লাইনটা লাগবে)
[ -f /usr/share/git-core/contrib/completion/git-prompt.sh ] && \
    . /usr/share/git-core/contrib/completion/git-prompt.sh
GIT_PS1_SHOWDIRTYSTATE=1          # অসংরক্ষিত পরিবর্তন থাকলে * দেখাবে
PS1='\u@\h:\w\[\e[33m\]$(__git_ps1 " (%s)")\[\e[0m\]\$ '
```

```
rahim@laptop:~/myproject (main *)$
```

খেয়াল করো, `$(__git_ps1 ...)` **single quote-এর ভেতরে** আছে, তাই এটা প্রতিবার prompt দেখানোর সময় চলে। double quote দিলে `.bashrc` পড়ার সময় একবারই চলত।

### অন্য prompt variable-গুলো

| Variable | কখন দেখায় | ডিফল্ট |
|----------|-----------|--------|
| `PS1` | প্রধান prompt | distro-ভেদে |
| `PS2` | কমান্ড অসম্পূর্ণ হলে পরের লাইনে (যেমন quote খোলা রেখে Enter) | `> ` |
| `PS3` | `select` লুপের মেনু প্রশ্ন | `#? ` |
| `PS4` | `set -x` ডিবাগ আউটপুটের প্রতিটা লাইনের শুরু | `+ ` |

```bash
$ PS4='+ ${BASH_SOURCE}:${LINENO}: '   # set -x-এ ফাইল ও লাইন নম্বরও দেখাবে
```

> 💡 **টিপস:** একটা নিরীহ স্ক্রিপ্ট বা অনলাইন জেনারেটর দিয়ে (যেমন "bash prompt generator" লিখে খুঁজলে পাবে) PS1 বানিয়ে পরীক্ষা করতে পারো। আগে বর্তমান শেলে `PS1='...'` লিখে দেখো, পছন্দ হলে তবেই `.bashrc`-এ রাখো।

---

## ৪৪.১১ zsh ও oh-my-zsh পরিচিতি

### zsh কী

**zsh** (Z shell) bash-এর মতোই আরেকটা শেল। বেশিরভাগ bash কমান্ড আর স্ক্রিপ্টিং সিনট্যাক্স এতে চলে, সাথে আছে বাড়তি সুবিধা। macOS Catalina (10.15) থেকে macOS-এর ডিফল্ট শেল zsh, আর Kali Linux-ও ২০২০.৪ থেকে zsh ব্যবহার করে। Ubuntu/Debian/Fedora/RHEL-এর ডিফল্ট এখনো bash।

zsh-এর বাড়তি সুবিধা:

- আরও স্মার্ট Tab completion: মেনু থেকে তীর দিয়ে বাছাই, অপশনের ব্যাখ্যা দেখা
- `**` দিয়ে recursive glob ডিফল্টে চালু (`ls **/*.txt`); bash-এ এর জন্য `shopt -s globstar` লাগে
- glob qualifier: `ls *(.)` শুধু সাধারণ ফাইল, `ls *(/)` শুধু ডিরেক্টরি, `ls *(m-1)` শেষ ১ দিনে বদলানো ফাইল
- ডিরেক্টরির নাম লিখে Enter দিলেই `cd` (`setopt autocd`)
- বানান ভুল ঠিক করার প্রস্তাব (`setopt correct`)
- একাধিক টার্মিনালে history ভাগ করা (`setopt share_history`)
- ডানপাশের prompt (`RPROMPT`)

> ⚠️ **সতর্কতা: কিছু পার্থক্য আছে।** zsh-এ array-এর index ১ থেকে শুরু (bash-এ ০ থেকে)। আর quote ছাড়া `$var` স্পেসে ভেঙে আলাদা শব্দ হয় না (word splitting হয় না)। তাই সার্ভারে চালানোর স্ক্রিপ্ট সবসময় `#!/bin/bash` দিয়ে লেখো। zsh শুধু নিজের interactive ব্যবহারের জন্য রাখো।

### zsh ইনস্টল ও ডিফল্ট করা

```bash
# Ubuntu/Debian
$ sudo apt install zsh
# Fedora/RHEL
$ sudo dnf install zsh

$ zsh --version
zsh 5.9 (x86_64-ubuntu-linux-gnu)

$ chsh -s "$(command -v zsh)"      # ডিফল্ট লগইন শেল বদলাও (পাসওয়ার্ড চাইবে)
$ grep "^$USER:" /etc/passwd
rahim:x:1000:1000:Rahim,,,:/home/rahim:/usr/bin/zsh
```

পরিবর্তন কার্যকর হবে পরের লগইনে (logout/login করো)। `chsh` শুধু `/etc/shells`-এ তালিকাভুক্ত শেল গ্রহণ করে (`cat /etc/shells`)। Fedora/RHEL-এ `chsh` না থাকলে `sudo dnf install util-linux-user` দিয়ে ইনস্টল করো।

প্রথমবার zsh চালালে কোনো `~/.zshrc` না থাকলে `zsh-newuser-install` একটা মেনু দেখায়। `q` চাপলে কিছু না করে বের হবে, `0` চাপলে একটা খালি `~/.zshrc` বানাবে।

### zsh-এর স্টার্টআপ ফাইল

zsh-এর নিয়ম bash-এর চেয়ে পরিষ্কার:

```
 সব zsh (স্ক্রিপ্ট সহ)       login হলে      interactive হলে    login হলে     logout-এ
 ────────────────────      ──────────     ───────────────    ─────────     ─────────
 /etc/zsh/zshenv      ──►  /etc/zsh/   ──► /etc/zsh/zshrc ──► /etc/zsh/  ──► ...
 ~/.zshenv                 zprofile        ~/.zshrc            zlogin        ~/.zlogout
                           ~/.zprofile                         ~/.zlogin
```

(সিস্টেম ফাইলগুলো Debian/Ubuntu-তে `/etc/zsh/` ডিরেক্টরিতে, Fedora/RHEL-এ সরাসরি `/etc/`-এ, যেমন `/etc/zshrc`।)

| ফাইল | কখন | কী রাখবে |
|------|-----|---------|
| `~/.zshenv` | **সব** zsh-এ, এমনকি স্ক্রিপ্টেও | শুধু অপরিহার্য environment (কোনো আউটপুট নয়) |
| `~/.zprofile` | login shell | PATH ইত্যাদি (bash-এর `.profile`-এর মতো) |
| `~/.zshrc` | interactive shell | alias, prompt, plugin, `setopt` |
| `~/.zlogin` | login shell, `.zshrc`-এর পরে | বিরল |
| `~/.zlogout` | login shell থেকে বের হলে | পরিষ্কার করার কাজ |

> 📝 **মনে রাখো:** zsh তোমার `~/.bashrc` বা `~/.bash_aliases` পড়ে না। bash থেকে zsh-এ গেলে alias আর PATH `~/.zshrc`-এ কপি করো, অথবা `~/.zshrc`-এ `source ~/.bash_aliases` লেখো (সাধারণ alias হলে দুই শেলেই চলবে)।

### oh-my-zsh

**oh-my-zsh** হলো zsh-এর একটা জনপ্রিয় কমিউনিটি ফ্রেমওয়ার্ক (zsh-এর অংশ নয়, আলাদা প্রজেক্ট)। এতে শত শত plugin (alias আর completion-এর সংগ্রহ) আর অনেক theme (prompt-এর ডিজাইন) আছে।

**ইনস্টল** (আগে zsh, git আর curl লাগবে):

```bash
$ sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

> ⚠️ **সতর্কতা:** `curl ... | sh` বা `sh -c "$(curl ...)"` মানে ইন্টারনেটের একটা স্ক্রিপ্ট না দেখেই চালানো। বিশ্বস্ত উৎস হলেও ভালো অভ্যাস হলো আগে ডাউনলোড করে পড়ে নেওয়া:
>
> ```bash
> $ curl -fsSL -o install.sh https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh
> $ less install.sh
> $ sh install.sh
> ```

ইনস্টলার যা করে: `~/.oh-my-zsh`-এ রিপো clone করে, তোমার পুরনো `~/.zshrc` থাকলে সেটাকে `~/.zshrc.pre-oh-my-zsh` নামে ব্যাকআপ রাখে, নতুন একটা `~/.zshrc` বসায়, আর ডিফল্ট শেল zsh করার প্রস্তাব দেয়।

**`~/.zshrc`-এর মূল অংশ:**

```bash
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="robbyrussell"        # ডিফল্ট থিম; "agnoster", "random" ইত্যাদি
plugins=(git sudo docker)       # স্পেস দিয়ে আলাদা, কমা নয়!
source $ZSH/oh-my-zsh.sh

# নিজের alias এই source লাইনের নিচে লেখো
alias ll='ls -alF'
```

| কাজ | কমান্ড |
|-----|--------|
| থিম বদলানো | `~/.zshrc`-এ `ZSH_THEME="..."` বদলে `source ~/.zshrc` |
| থিমের তালিকা | `ls ~/.oh-my-zsh/themes/` |
| plugin-এর তালিকা | `ls ~/.oh-my-zsh/plugins/` |
| আপডেট | `omz update` |
| আনইনস্টল | `uninstall_oh_my_zsh` |

জনপ্রিয় কিছু plugin:

| Plugin | কী করে |
|--------|--------|
| `git` | `gst` (git status), `gco` (git checkout), `gp` (git push) ইত্যাদি অনেক alias |
| `sudo` | দুবার Esc চাপলে আগের/বর্তমান কমান্ডের সামনে `sudo` বসে যায় |
| `z` | যেসব ডিরেক্টরিতে বেশি যাও সেগুলোতে লাফ: `z proj` |
| `extract` | `x file.tar.gz` — যেকোনো আর্কাইভ খোলে |
| `zsh-autosuggestions` | history থেকে ধূসর রঙে পরামর্শ দেখায়, → চাপলে গ্রহণ (আলাদা ডাউনলোড লাগে) |
| `zsh-syntax-highlighting` | টাইপ করার সময় ঠিক কমান্ড সবুজ, ভুল লাল (আলাদা ডাউনলোড লাগে) |

শেষের দুটো oh-my-zsh-এর সাথে আসে না, `custom/plugins`-এ clone করতে হয়:

```bash
$ git clone https://github.com/zsh-users/zsh-autosuggestions \
    ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
$ git clone https://github.com/zsh-users/zsh-syntax-highlighting \
    ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
# তারপর ~/.zshrc-এ: plugins=(git sudo zsh-autosuggestions zsh-syntax-highlighting)
```

(Ubuntu/Debian-এ `apt install zsh-autosuggestions zsh-syntax-highlighting` দিয়েও পাওয়া যায়। তখন oh-my-zsh plugin হিসেবে না দিয়ে `/usr/share/...` থেকে source করতে হয়।)

> 💡 **টিপস:** বেশি plugin দিলে টার্মিনাল খুলতে দেরি হয়। কতক্ষণ লাগছে মাপতে: `time zsh -i -c exit`। একই রকম জনপ্রিয় বিকল্প হলো **Starship** নামের prompt, যা bash, zsh, fish সব শেলে কাজ করে। আর **fish** শেল নিজেই ডিফল্টে autosuggestion আর রঙ দেয়, তবে এর স্ক্রিপ্টিং সিনট্যাক্স POSIX-এর মতো নয়।

---

## ৪৪.১২ dotfiles — তোমার কনফিগারেশনের সম্পদ

### dotfiles কী

লিনাক্সে যে ফাইলের নাম `.` দিয়ে শুরু হয়, সেটা লুকানো থাকে (`ls`-এ দেখা যায় না, `ls -a`-তে দেখা যায়)। প্রোগ্রামগুলো তাদের কনফিগারেশন হোম ডিরেক্টরির এমন লুকানো ফাইলে রাখে। তাই এদের বলে **dotfiles**।

```bash
$ ls -a ~
.  ..  .bash_history  .bash_logout  .bashrc  .cache  .config  .gitconfig
.local  .profile  .ssh  .vimrc  Desktop  Documents  Downloads
```

| ফাইল/ডিরেক্টরি | কিসের কনফিগ |
|----------------|-------------|
| `~/.bashrc`, `~/.profile`, `~/.bash_aliases` | bash |
| `~/.zshrc` | zsh |
| `~/.vimrc` | Vim ([অধ্যায় ১৪](14-vim.md)) |
| `~/.nanorc` | nano ([অধ্যায় ১৩](13-nano-editor.md)) |
| `~/.gitconfig` | git |
| `~/.tmux.conf` | tmux ([অধ্যায় ২৫](25-jobs-tmux.md)) |
| `~/.inputrc` | readline: bash-এর কী-বাইন্ডিং ও completion আচরণ |
| `~/.ssh/config` | SSH হোস্টের শর্টকাট ([অধ্যায় ৩৮](38-ssh-remote.md)) |
| `~/.config/` | XDG নিয়ম মেনে চলা আধুনিক অ্যাপের কনফিগ |

**নতুন ইউজারের dotfiles আসে কোথা থেকে?** `/etc/skel/` থেকে। `useradd -m` বা `adduser` নতুন ইউজার বানানোর সময় এই ফোল্ডারের সবকিছু তার হোমে কপি করে দেয় ([অধ্যায় ১৫](15-users-groups.md))। তাই সব নতুন ইউজারকে একটা ডিফল্ট `.bashrc` দিতে চাইলে `/etc/skel/.bashrc` এডিট করো।

### কেন dotfiles নিয়ে মাথা ঘামাবে

বছরের পর বছর তুমি নিজের মতো alias, prompt, vim সেটিং বানাবে। নতুন ল্যাপটপ বা নতুন সার্ভার পেলে সব আবার হাতে করা কষ্টকর। তাই অভিজ্ঞ লিনাক্স ইউজাররা dotfiles **Git রিপোজিটরিতে** রাখে, আর নতুন মেশিনে কয়েক মিনিটেই নিজের পরিচিত পরিবেশ ফিরিয়ে আনে। GitHub-এ "dotfiles" লিখে খুঁজলে হাজার হাজার মানুষের রিপো পাবে, যেখান থেকে শেখা যায়।

### পদ্ধতি ১: ফোল্ডার + symlink (সবচেয়ে সহজ)

```bash
$ mkdir -p ~/dotfiles
$ mv ~/.bashrc ~/.bash_aliases ~/.vimrc ~/dotfiles/
$ ln -s ~/dotfiles/.bashrc       ~/.bashrc
$ ln -s ~/dotfiles/.bash_aliases ~/.bash_aliases
$ ln -s ~/dotfiles/.vimrc        ~/.vimrc
$ ls -l ~/.bashrc
lrwxrwxrwx 1 rahim rahim 27 Nov 12 11:00 /home/rahim/.bashrc -> /home/rahim/dotfiles/.bashrc

$ cd ~/dotfiles && git init && git add . && git commit -m "my dotfiles"
```

symlink নিয়ে বিস্তারিত [অধ্যায় ১১](11-links-inode.md)-এ। প্রোগ্রাম `~/.bashrc` খোলে, আর symlink তাকে `~/dotfiles/.bashrc`-এ নিয়ে যায়।

### পদ্ধতি ২: GNU Stow (symlink স্বয়ংক্রিয়)

```bash
$ sudo apt install stow        # Fedora/RHEL: sudo dnf install stow (RHEL-এ EPEL লাগতে পারে)

# গঠন: প্রতিটা "প্যাকেজ" একটা ফোল্ডার, ভেতরে হোমের মতো একই কাঠামো
~/dotfiles/
├── bash/
│   ├── .bashrc
│   └── .bash_aliases
└── vim/
    └── .vimrc

$ cd ~/dotfiles
$ stow bash vim               # ~/.bashrc → dotfiles/bash/.bashrc ইত্যাদি symlink বানাবে
$ stow -D vim                 # vim-এর symlink সরাবে
```

`stow` ডিফল্টে বর্তমান ফোল্ডারের **parent** (এখানে `~`)-কে টার্গেট ধরে। হোমে একই নামের আসল ফাইল আগে থেকে থাকলে stow ঝুঁকি না নিয়ে conflict দেখিয়ে থেমে যায়। তখন আসল ফাইলটা সরিয়ে আবার চালাও।

### পদ্ধতি ৩: bare git রিপো (কোনো symlink নেই)

```bash
$ git init --bare $HOME/.dotfiles
$ alias dot='git --git-dir=$HOME/.dotfiles --work-tree=$HOME'
$ dot config --local status.showUntrackedFiles no   # হোমের সব ফাইল untracked দেখাবে না
$ dot add ~/.bashrc ~/.vimrc
$ dot commit -m "add bashrc and vimrc"
$ dot remote add origin git@github.com:rahim/dotfiles.git
$ dot push -u origin main
```

এখানে তোমার হোম ডিরেক্টরিই git-এর work tree, তাই ফাইলগুলো নিজের জায়গাতেই থাকে। `dot` alias-টা `.bashrc`-এ রেখে দিও।

> ⚠️ **সতর্কতা: পাবলিক রিপোতে গোপন জিনিস রাখবে না।** `~/.ssh/id_*` (private key), `~/.aws/credentials`, `.netrc`, API token বা পাসওয়ার্ড যুক্ত `export` লাইন — এগুলো কখনো commit করবে না। গোপন জিনিস আলাদা একটা ফাইলে রাখো (যেমন `~/.bashrc.local`), সেটা git-এর বাইরে রাখো, আর `.bashrc` থেকে লোড করো:
>
> ```bash
> [ -f ~/.bashrc.local ] && . ~/.bashrc.local
> ```

---

## ৪৪.১৩ একটা পূর্ণাঙ্গ, পরিচ্ছন্ন `~/.bashrc` (নমুনা)

Ubuntu-র ডিফল্ট ফাইলের শেষে তুমি এ রকম একটা অংশ যোগ করতে পারো:

```bash
# ===== আমার কাস্টমাইজেশন =====

# ---- PATH (একবারই যোগ হবে) ----
for d in "$HOME/bin" "$HOME/.local/bin"; do
    [ -d "$d" ] || continue
    case ":$PATH:" in *":$d:"*) ;; *) PATH="$d:$PATH" ;; esac
done
export PATH

# ---- ডিফল্ট প্রোগ্রাম ----
export EDITOR=nano
export VISUAL=nano
export PAGER=less

# ---- History (বিস্তারিত অধ্যায় ৬) ----
HISTSIZE=10000
HISTFILESIZE=20000
HISTCONTROL=ignoreboth           # স্পেস দিয়ে শুরু ও পরপর একই কমান্ড বাদ
HISTTIMEFORMAT='%F %T '          # history-তে তারিখ-সময়
shopt -s histappend              # সব টার্মিনালের history যোগ হবে, overwrite নয়

# ---- শেলের আচরণ ----
shopt -s checkwinsize            # উইন্ডোর আকার বদলালে LINES/COLUMNS আপডেট
shopt -s globstar                # ** দিয়ে recursive glob
shopt -s cdspell                 # cd-তে ছোটখাটো বানান ভুল ঠিক করে
shopt -s autocd                  # ডিরেক্টরির নাম লিখলেই cd

# ---- Prompt ----
__prompt() {
    local ec=$?
    local c_ok='\[\e[32m\]' c_err='\[\e[31m\]' c_dir='\[\e[1;34m\]' r='\[\e[0m\]'
    local mark="${c_ok}✔${r}"
    [ $ec -ne 0 ] && mark="${c_err}✘${ec}${r}"
    PS1="${mark} \u@\h:${c_dir}\w${r}\n\$ "
}
PROMPT_COMMAND=__prompt

# ---- alias ও function ----
[ -f ~/.bash_aliases ] && . ~/.bash_aliases
mkcd() { mkdir -p -- "$1" && cd -- "$1"; }

# ---- গোপন বা মেশিন-নির্দিষ্ট সেটিং ----
[ -f ~/.bashrc.local ] && . ~/.bashrc.local
```

> 💡 **টিপস:** `.bashrc` এডিট করার আগে একটা কপি রাখো: `cp ~/.bashrc ~/.bashrc.bak`। আর এডিটের পর **পুরনো টার্মিনাল বন্ধ না করেই** নতুন একটা টার্মিনাল খুলে পরীক্ষা করো। কিছু ভাঙলে পুরনো টার্মিনাল থেকে ঠিক করতে পারবে।

`~/.inputrc` দিয়েও কিছু কাজের সেটিং করা যায় (এটা bash-এর লাইন-এডিটিং লাইব্রেরি readline পড়ে):

```
# ~/.inputrc
$include /etc/inputrc
set completion-ignore-case on        # Tab completion-এ ছোট-বড় হাতের অক্ষর এক ধরা
set show-all-if-ambiguous on         # একবার Tab-এই সব বিকল্প দেখাও
"\e[A": history-search-backward      # ↑: যা টাইপ করেছ তা দিয়ে শুরু হওয়া history
"\e[B": history-search-forward       # ↓
```

পরিবর্তন নতুন শেলে কার্যকর হবে, বা বর্তমান শেলে `bind -f ~/.inputrc` দিয়ে।

---

## ৪৪.১৪ এই অধ্যায়ের সাথে সম্পর্কিত কীবোর্ড শর্টকাট

| শর্টকাট | কাজ | শেল |
|---------|-----|-----|
| `Ctrl + Alt + E` | বর্তমান লাইনের alias, history ও variable expand করে দেখাও (চালায় না) | bash (readline `shell-expand-line`) |
| `Esc` তারপর `.` / `Alt + .` | আগের কমান্ডের শেষ argument বসাও (যেমন `mkdir dir` তারপর `cd Alt+.`) | bash, zsh |
| `Ctrl + X`, তারপর `Ctrl + E` | বর্তমান কমান্ড `$VISUAL`/`$EDITOR`-এ খুলে এডিট করো, সেভ করে বের হলে চলবে | bash |
| `Tab` (দুবার) | সম্ভাব্য সব completion দেখাও, যেমন `$HO` লিখে Tab দিলে `$HOME` | bash |
| `Esc` `Esc` | কমান্ডের সামনে `sudo` বসাও | zsh + oh-my-zsh `sudo` plugin |
| `→` বা `End` | ধূসর autosuggestion গ্রহণ | zsh-autosuggestions |
| `Ctrl + L` | স্ক্রিন পরিষ্কার (prompt নতুন করে আঁকে, PS1 পরীক্ষায় কাজে লাগে) | bash, zsh |
| `Ctrl + D` | খালি লাইনে চাপলে শেল থেকে বের হও (login shell হলে `~/.bash_logout` চলবে) | bash, zsh |
| `Ctrl + Alt + T` | নতুন টার্মিনাল খোলো (non-login interactive shell) | Ubuntu GNOME ([অধ্যায় ৪](04-desktop-environment.md)) |
| `Ctrl + Alt + F3`…`F6` | টেক্সট কনসোল, যেখানে **login shell** চলে; `.bashrc` ভাঙলে উদ্ধার পেতে কাজে লাগে | বেশিরভাগ distro |

`bind -P | grep -v 'not bound'` চালালে বর্তমানে bound সব readline শর্টকাট দেখা যাবে। শর্টকাটের পূর্ণ তালিকা আছে [অধ্যায় ৬](06-terminal-shortcuts.md) আর [পরিশিষ্ট ক](appendix-a-shortcuts-cheatsheet.md)-এ।

---

## হাতে-কলমে অনুশীলন

### ল্যাব ১: shell variable বনাম environment variable

```bash
$ city="Dhaka"
$ bash -c 'echo "child দেখছে: [$city]"'
child দেখছে: []
$ export city
$ bash -c 'echo "child দেখছে: [$city]"'
child দেখছে: [Dhaka]

# child বদলালে parent-এ প্রভাব পড়ে কি?
$ bash -c 'city="Chittagong"; echo "child-এ: $city"'
child-এ: Chittagong
$ echo "parent-এ: $city"
parent-এ: Dhaka

# শুধু একটা কমান্ডের জন্য variable
$ city=Sylhet bash -c 'echo $city'
Sylhet
$ echo $city
Dhaka
```

**প্রশ্ন:** শেষ দুটো আউটপুট আলাদা কেন? (উত্তর: `city=Sylhet command` লিখলে মানটা শুধু ওই কমান্ডের environment-এ যায়।)

### ল্যাব ২: PATH-এ নিজের কমান্ড

```bash
$ mkdir -p ~/bin
$ cat > ~/bin/hello <<'EOF'
#!/bin/bash
echo "হ্যালো, $USER! আজ $(date +%A)"
EOF
$ chmod +x ~/bin/hello
$ hello
bash: hello: command not found       # PATH-এ ~/bin না থাকলে

$ export PATH="$HOME/bin:$PATH"
$ hello
হ্যালো, rahim! আজ Thursday
$ type hello
hello is /home/rahim/bin/hello
```

এবার PATH-এর ক্রমের প্রভাব দেখো:

```bash
$ cat > ~/bin/ls <<'EOF'
#!/bin/bash
echo "এটা আমার নকল ls!"
EOF
$ chmod +x ~/bin/ls
$ hash -r
$ \ls                       # alias এড়িয়ে
এটা আমার নকল ls!
$ type -a ls
ls is aliased to `ls --color=auto'
ls is /home/rahim/bin/ls
ls is /usr/bin/ls
ls is /bin/ls
$ rm ~/bin/ls && hash -r    # পরিষ্কার করো!
```

এই কারণেই PATH-এর শুরুতে কোন ডিরেক্টরি আছে, আর সেখানে কে লিখতে পারে, সেটা নিরাপত্তার জন্য গুরুত্বপূর্ণ ([অধ্যায় ৪৫](45-security-hardening.md))।

### ল্যাব ৩: login বনাম non-login

```bash
$ echo $0; shopt login_shell
bash
login_shell    	off
$ bash -l
$ echo $0; shopt login_shell
bash
login_shell    	on
$ exit
$ su - $USER                  # পাসওয়ার্ড দাও
$ echo $0
-bash
$ exit
```

(`bash -l`-এ `$0`-এর সামনে `-` থাকে না, কারণ argv[0] বদলানো হয়নি, শুধু `--login` অপশন দেওয়া হয়েছে। তাই `shopt login_shell` সবচেয়ে নির্ভরযোগ্য পরীক্ষা।)

### ল্যাব ৪: স্থায়ী alias ও function

```bash
$ cp ~/.bashrc ~/.bashrc.bak
$ cat >> ~/.bash_aliases <<'EOF'
alias ..='cd ..'
alias ll='ls -alFh'
alias ports='ss -tulpn'
EOF
$ grep -q bash_aliases ~/.bashrc || echo '[ -f ~/.bash_aliases ] && . ~/.bash_aliases' >> ~/.bashrc
$ echo 'mkcd() { mkdir -p -- "$1" && cd -- "$1"; }' >> ~/.bashrc
$ source ~/.bashrc
$ mkcd /tmp/test/a/b && pwd
/tmp/test/a/b
$ ..
$ pwd
/tmp/test/a
```

(Fedora/RHEL-এর `.bashrc` `~/.bash_aliases` খোঁজে না, তাই দ্বিতীয় লাইনের `grep` সেই লাইনটা যোগ করে দেবে।)

### ল্যাব ৫: নিজের prompt

1. বর্তমান prompt সংরক্ষণ করো: `OLD_PS1="$PS1"`
2. চেষ্টা করো: `PS1='\[\e[1;35m\][\A]\[\e[0m\] \[\e[1;32m\]\u\[\e[0m\]:\W\$ '`
3. একটা লম্বা কমান্ড লিখে দেখো, লাইন ঠিকমতো ভাঙছে কি না
4. এবার ইচ্ছা করে `\[ \]` বাদ দিয়ে দেখো কী গোলমাল হয়: `PS1='\e[1;32m\u\e[0m\$ '`, তারপর অনেক লম্বা কিছু টাইপ করে Ctrl+A চাপো
5. ফেরত যাও: `PS1="$OLD_PS1"`
6. পছন্দেরটা `~/.bashrc`-এর শেষে লেখো

### ল্যাব ৬: dotfiles রিপো

```bash
$ mkdir -p ~/dotfiles/bash
$ cp ~/.bashrc ~/.bash_aliases ~/dotfiles/bash/
$ cd ~/dotfiles && git init -q && git add . && git commit -qm "initial dotfiles"
$ git log --oneline
a1b2c3d initial dotfiles
```

চ্যালেঞ্জ: `stow` ইনস্টল করে আসল ফাইল দুটো সরাও (`mv ~/.bashrc ~/.bashrc.orig` ইত্যাদি), তারপর `cd ~/dotfiles && stow bash` চালাও। `ls -l ~/.bashrc` দিয়ে দেখো symlink হয়েছে কি না।

### ল্যাব ৭ (ঐচ্ছিক): zsh ঘুরে দেখা

```bash
$ sudo apt install zsh        # বা: sudo dnf install zsh
$ zsh                         # প্রথমবার মেনু এলে 'q'
% echo $0
zsh
% ls /etc/**/*.conf | head    # recursive glob
% exit
```

শুধু ঘুরে দেখার জন্য ডিফল্ট শেল বদলানোর দরকার নেই।

---

## সারসংক্ষেপ

| কমান্ড / জিনিস | কাজ |
|----------------|-----|
| `NAME=value` | shell variable (শুধু এই শেলে) |
| `export NAME=value` | environment variable (child process-ও পায়) |
| `export -n NAME` | export চিহ্ন তুলে দাও |
| `export -p` / `declare -x` | সব exported variable |
| `env` | environment দেখাও |
| `env -i cmd` / `env -u NAME cmd` | খালি / একটা variable বাদ দিয়ে environment-এ কমান্ড চালাও |
| `NAME=value cmd` | শুধু ওই কমান্ডের জন্য variable |
| `printenv NAME` | একটা environment variable-এর মান |
| `set` | সব variable ও function; `set -o` অপশন |
| `unset NAME` / `unset -f fn` | variable / function মুছে ফেলো |
| `declare -p NAME`, `readonly NAME` | variable-এর ধরন দেখাও / স্থির করো |
| `echo $PATH` | কমান্ড খোঁজার ডিরেক্টরি তালিকা |
| `export PATH="$HOME/bin:$PATH"` | PATH-এর শুরুতে যোগ |
| `type -a`, `command -v`, `which` | কমান্ড কোথা থেকে আসছে |
| `hash`, `hash -r` | মনে রাখা পাথ দেখাও / ভুলে যাও |
| `locale`, `localectl` | ভাষা/অঞ্চলের সেটিং |
| `shopt login_shell`, `echo $-` | login / interactive কি না |
| `/etc/profile` → `~/.bash_profile` বা `~/.bash_login` বা `~/.profile` | login shell (প্রথম পাওয়া একটা) |
| `/etc/bash.bashrc` → `~/.bashrc` | interactive non-login shell (Debian/Ubuntu) |
| `~/.bashrc` → `/etc/bashrc` | interactive non-login shell (Fedora/RHEL) |
| `~/.bash_logout` | login shell থেকে বের হলে |
| `source file` / `. file` | বর্তমান শেলেই ফাইল চালাও |
| `bash --norc`, `bash -l` | rc ছাড়া / login হিসেবে শেল |
| `alias name='cmd'`, `unalias name` | alias বানাও / মুছে ফেলো |
| `\cmd`, `command cmd` | alias এড়িয়ে আসল কমান্ড |
| `PS1`, `PS2`, `PS4`, `PROMPT_COMMAND` | prompt কাস্টমাইজেশন |
| `\u \h \w \W \$ \t \A \n \[ \]` | PS1-এর প্রধান escape কোড |
| `chsh -s /usr/bin/zsh` | ডিফল্ট লগইন শেল বদলাও |
| `~/.zshenv ~/.zprofile ~/.zshrc ~/.zlogin` | zsh-এর স্টার্টআপ ফাইল |
| `ZSH_THEME`, `plugins=(...)`, `omz update` | oh-my-zsh |
| `/etc/skel/` | নতুন ইউজারের ডিফল্ট dotfiles |
| `stow pkg`, bare git repo | dotfiles ব্যবস্থাপনা |
| `Ctrl+Alt+E`, `Ctrl+X Ctrl+E`, `Alt+.` | expand / এডিটরে কমান্ড / আগের argument |

---

## নিজেকে যাচাই করো

1. `x=5` লিখে তারপর `bash -c 'echo $x'` চালালে কী আসবে, আর কেন? এটাকে কাজ করাতে কী করতে হবে?
2. একটা স্ক্রিপ্টের ভেতরে `export PATH="$HOME/tools:$PATH"` লিখে `./setup.sh` চালালে তোমার টার্মিনালের PATH কেন বদলায় না? কীভাবে চালালে বদলাবে?
3. `env`, `printenv` আর `set`-এর আউটপুটের মূল পার্থক্য কী?
4. `PATH=/home/rahim/bin:/usr/bin` আর দুই ডিরেক্টরিতেই `backup` নামের প্রোগ্রাম থাকলে কোনটা চলবে? কীভাবে নিশ্চিত হবে?
5. Ubuntu-তে তুমি একটা নতুন `~/.bash_profile` বানালে, আর এখন ssh-এ লগইন করলে তোমার alias কাজ করছে না। কেন? সমাধান কী?
6. ডেস্কটপে GNOME Terminal খোলা, `ssh user@host`, আর `bash script.sh` — এই তিনটা কোন কোন ধরনের শেল (login/non-login, interactive/non-interactive)?
7. PS1-এ রঙের কোডের চারপাশে `\[` আর `\]` কেন দিতে হয়?
8. alias-এ argument মাঝখানে বসানো যায় না কেন, আর তার বিকল্প কী?
9. zsh-এ যেয়ে তোমার `~/.bashrc`-এর alias কাজ করছে না কেন?
10. dotfiles রিপো GitHub-এ পাবলিক রাখলে কোন সাবধানতা মানতে হবে?

<details>
<summary>উত্তর দেখো</summary>

1. খালি লাইন আসবে। `x` একটা shell variable, export করা হয়নি, তাই child bash-এর environment-এ এটা যায় না। `export x` (বা `export x=5`) করলে আসবে `5`। অথবা শুধু ওই কমান্ডের জন্য `x=5 bash -c 'echo $x'`।

2. `./setup.sh` একটা আলাদা child process-এ চলে। সে নিজের environment কপিতে PATH বদলায়, আর শেষ হলে সেই কপি হারিয়ে যায়। environment কখনো child থেকে parent-এ যায় না। `source ./setup.sh` বা `. ./setup.sh` দিয়ে চালালে বর্তমান শেলেই চলবে, আর PATH বদলাবে।

3. `env` আর `printenv` বাইরের প্রোগ্রাম, তাই শুধু exported (environment) variable দেখায়। `printenv NAME` একটা নির্দিষ্ট variable-এর মান দেয়, আর না থাকলে exit code 1 দেয়। `set` শেলের builtin, তাই exported আর non-exported সব variable, সাথে সব shell function-ও দেখায়।

4. `/home/rahim/bin/backup` চলবে, কারণ PATH বাম থেকে ডানে খোঁজা হয় আর প্রথম যেটা পাওয়া যায় সেটাই চলে। `type -a backup` বা `command -v backup` দিয়ে নিশ্চিত হও। সম্প্রতি PATH বা ফাইল বদলালে `hash -r` করে নাও।

5. login shell `~/.bash_profile`, `~/.bash_login`, `~/.profile`-এর মধ্যে প্রথম যেটা পায় শুধু সেটাই পড়ে। নতুন `~/.bash_profile` থাকায় `~/.profile` আর পড়া হচ্ছে না, ফলে তার ভেতরের `. ~/.bashrc`-ও চলছে না। সমাধান: `~/.bash_profile` মুছে ফেলো, অথবা তার ভেতরে `[ -f ~/.profile ] && . ~/.profile` (বা সরাসরি `. ~/.bashrc`) যোগ করো।

6. GNOME Terminal: non-login, interactive (ডিফল্ট সেটিংয়ে)। `ssh user@host` (কমান্ড ছাড়া): login, interactive। `bash script.sh`: non-login, non-interactive (কোনো স্টার্টআপ ফাইল পড়ে না, শুধু `$BASH_ENV`)।

7. রঙের escape কোড স্ক্রিনে দেখা যায় না, কিন্তু bash prompt-এর দৈর্ঘ্য গুনে ঠিক করে কার্সর কোথায় আর লাইন কখন ভাঙবে। `\[ \]` দিয়ে জানিয়ে দিতে হয় যে এই অংশ জায়গা নেয় না। না দিলে লম্বা কমান্ডে লাইন ভাঙা আর কার্সর চলাচল এলোমেলো হয়ে যায়।

8. alias শুধু প্রথম শব্দটাকে একটা লেখা দিয়ে বদলে দেয়। এতে `$1` জাতীয় argument-এর ধারণা নেই, পরের শব্দগুলো শুধু শেষে জুড়ে যায়। মাঝখানে argument বসাতে বা যুক্তি (if/loop) লাগলে shell function ব্যবহার করো, যেমন `mkcd() { mkdir -p -- "$1" && cd -- "$1"; }`।

9. zsh নিজের ফাইল পড়ে (`~/.zshenv`, `~/.zprofile`, `~/.zshrc`, `~/.zlogin`), bash-এর `~/.bashrc` বা `~/.bash_aliases` পড়ে না। alias `~/.zshrc`-এ কপি করো, অথবা `~/.zshrc`-এ `source ~/.bash_aliases` লেখো।

10. গোপন তথ্য কখনো commit করবে না: SSH private key, ক্লাউড credential, API token, পাসওয়ার্ড যুক্ত `export` লাইন। এগুলো আলাদা ফাইলে রাখো (যেমন `~/.bashrc.local`), সেটা `.gitignore`-এ দাও বা রিপোর বাইরে রাখো, আর `[ -f ~/.bashrc.local ] && . ~/.bashrc.local` দিয়ে লোড করো। push করার আগে `git diff --cached` দেখে নাও।

</details>

---

[← আগের অধ্যায়](43-shell-scripting-3.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](45-security-hardening.md)

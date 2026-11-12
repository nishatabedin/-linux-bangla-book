# অধ্যায় ২০: grep ও রেগুলার এক্সপ্রেশন

> "লাখ লাখ লাইনের লগ ফাইল থেকে একটা লাইন খুঁজে বের করতে হবে? মানুষের লাগবে কয়েক ঘণ্টা, `grep`-এর লাগবে এক সেকেন্ডেরও কম।"

## এই অধ্যায়ে যা শিখবে

- `grep` কী, নামটা কোথা থেকে এল, আর ভেতরে এটা কীভাবে কাজ করে
- `grep`-এর সব দরকারি অপশন: `-i -v -n -c -l -L -r -R -w -x -o -A -B -C -E -F -P --color --include --exclude` এবং আরও কিছু
- রেগুলার এক্সপ্রেশন (regex) কী, কেন দরকার, আর তার সব মেটাক্যারেক্টার: `. * + ? ^ $ [] [^] {n,m} () | \b \d`
- BRE, ERE আর PCRE — তিন ধরনের regex-এর পার্থক্য
- POSIX character class (`[[:digit:]]`, `[[:alpha:]]` ইত্যাদি)
- greedy matching কী, আর কীভাবে তার ফাঁদ এড়াবে
- `egrep` ও `fgrep` কেন এখন "পুরোনো" (obsolescent)
- আধুনিক, অতি-দ্রুত বিকল্প **ripgrep (`rg`)**
- বাস্তব কাজ: লগ থেকে IP ঠিকানা বের করা, ফাইল থেকে email খোঁজা, কোডবেসে খোঁজা, স্ক্রিপ্টে `grep` ব্যবহার

---

## ২০.১ grep কী, আর কেন এটা এত গুরুত্বপূর্ণ

`grep` হলো লিনাক্সের **টেক্সট খোঁজার যন্ত্র**। তুমি একটা প্যাটার্ন (pattern — খোঁজার নকশা) দাও, আর `grep` এক বা একাধিক ফাইলের (অথবা পাইপ দিয়ে আসা ইনপুটের) প্রতিটা লাইন পড়ে দেখে — যে লাইনে প্যাটার্নটা মেলে, শুধু সেই লাইনগুলো স্ক্রিনে ছাপে।

**উপমা:** ধরো তোমার সামনে একটা মোটা টেলিফোন ডিরেক্টরি। তোমাকে বের করতে হবে কোন কোন লোকের নামে "রহমান" আছে। তুমি প্রতিটা পাতার প্রতিটা লাইনে আঙুল রেখে দেখবে "রহমান" আছে কি না, থাকলে সেই লাইনে হাইলাইটার দিয়ে দাগ দেবে। `grep` ঠিক এই কাজটাই করে — কিন্তু সেকেন্ডে কয়েকশো মেগাবাইট গতিতে।

### নামটা কোথা থেকে এল?

১৯৭৩ সালে Ken Thompson ইউনিক্সের জন্য `grep` লেখেন। সেই সময়ের লাইন-এডিটর `ed`-এ একটা কমান্ড ছিল:

```
g/re/p
```

মানে **g**lobal — পুরো ফাইলে, **re** — একটা regular expression খোঁজো, **p**rint — মেলা লাইন ছাপো। এই কমান্ডটাকেই আলাদা প্রোগ্রাম বানানো হলো, নাম হলো `grep`। আজও প্রোগ্রামাররা "grep করা" কথাটা ক্রিয়াপদ হিসেবে ব্যবহার করে — যেমন "লগটা একটু grep করে দেখো তো।"

### কোথায় কোথায় লাগে?

| কাজ | উদাহরণ |
|---|---|
| লগ ফাইলে error খোঁজা | `grep -i error /var/log/syslog` |
| কনফিগ ফাইলে একটা সেটিং খোঁজা | `grep Port /etc/ssh/sshd_config` |
| কোডবেসে কোনো ফাংশন কোথায় ব্যবহার হয়েছে | `grep -rn "login(" src/` |
| অন্য কমান্ডের আউটপুট ছাঁকা | `ps aux \| grep nginx` |
| স্ক্রিপ্টে শর্ত যাচাই | `if grep -q "^bob:" /etc/passwd; then ...` |
| ডেটা বের করা (IP, email, ফোন নম্বর) | `grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' access.log` |

> 📝 **মনে রাখো:** `grep` ফাইল **খোঁজে না**, ফাইলের **ভেতরের লেখা** খোঁজে। নাম দিয়ে ফাইল খুঁজতে `find` বা `locate` লাগে — সেটা [অধ্যায় ১২](12-file-khoja.md)-এ আছে।

---

## ২০.২ ভেতরে কীভাবে কাজ করে (Under the hood)

`grep` যখন চলে, তখন মোটামুটি এই ঘটনাগুলো ঘটে:

```
   তোমার কমান্ড:  grep -i 'error' app.log
                       |      |        |
                       |      |        +--> ইনপুট সোর্স (ফাইল, না থাকলে stdin)
                       |      +-----------> প্যাটার্ন (regex)
                       +------------------> অপশন (কীভাবে মেলাবে/দেখাবে)

  +--------------------------------------------------------------+
  |                        grep প্রসেস                           |
  |                                                              |
  |  ১. প্যাটার্ন পার্স  --->  ২. প্যাটার্ন কম্পাইল (DFA / NFA)   |
  |                                    |                         |
  |  ৩. ফাইল থেকে বড় বড় টুকরো (buffer) read() করে পড়া          |
  |                                    |                         |
  |  ৪. buffer-এ দ্রুত স্ক্যান --> মিল পেলে সেই লাইনের শুরু-শেষ বের করা |
  |                                    |                         |
  |  ৫. অপশন অনুযায়ী ফরম্যাট (-n, -o, --color ...)  ---> stdout   |
  +--------------------------------------------------------------+
                                       |
                              exit status: 0 / 1 / 2
```

কিছু গুরুত্বপূর্ণ ব্যাপার:

1. **প্যাটার্ন কম্পাইল হয়।** তুমি যে regex দাও, `grep` সেটাকে একটা ছোট "যন্ত্রে" (state machine — অবস্থা-যন্ত্র) রূপান্তর করে। GNU grep যতটা সম্ভব **DFA** (Deterministic Finite Automaton) ব্যবহার করে — এটা প্রতিটা অক্ষর একবারই দেখে, তাই ভীষণ দ্রুত। কিন্তু back-reference (`\1`)-এর মতো ফিচার DFA দিয়ে হয় না, তখন সে ধীর **backtracking** ইঞ্জিনে চলে যায়।
2. **সাধারণ শব্দ হলে আরও দ্রুত।** প্যাটার্নে যদি কোনো স্থির শব্দ থাকে (যেমন `error`), GNU grep Boyer–Moore ধাঁচের অ্যালগরিদম দিয়ে আগে শুধু সেই শব্দটা খোঁজে — এই অ্যালগরিদম অনেক অক্ষর "লাফিয়ে" যেতে পারে, সব অক্ষর পড়তেও হয় না।
3. **লাইন ভাগ করে না আগে।** grep প্রথমে পুরো buffer-এ মিল খোঁজে, মিল পেলে তবেই আশেপাশের newline খুঁজে লাইনটা আলাদা করে। এ কারণেই বড় ফাইলেও grep এত দ্রুত।
4. **exit status (প্রস্থান কোড)** দিয়ে ফলাফল জানায় — স্ক্রিপ্টে এটা খুব কাজের:

| exit status | মানে |
|---|---|
| `0` | অন্তত একটা লাইন মিলেছে |
| `1` | কোনো লাইন মেলেনি |
| `2` | কোনো error হয়েছে (ফাইল নেই, ভুল regex ইত্যাদি) |

```
$ grep root /etc/passwd
root:x:0:0:root:/root:/bin/bash
$ echo $?
0
$ grep zzzz /etc/passwd
$ echo $?
1
$ grep root /nai/emon/file
grep: /nai/emon/file: No such file or directory
$ echo $?
2
```

> 💡 **টিপস:** `-q` দিলে কোনো ফাইলে error হলেও (যেমন একটা ফাইল নেই) অন্য কোথাও মিল পেলে GNU grep `0`-ই দেয়। শুধু "মিলেছে কি না" জানতে চাইলে `-q` (quiet) ব্যবহার করো — কিছুই ছাপবে না, প্রথম মিল পেলেই থেমে যাবে।

---

## ২০.৩ অনুশীলনের জন্য নমুনা ফাইল

এই অধ্যায়ের বেশিরভাগ উদাহরণ নিচের ফাইলটার উপর। চলো আগে বানিয়ে নিই (heredoc কীভাবে কাজ করে, সেটা [অধ্যায় ১৯](19-redirection-pipe.md)-এ আছে):

```
$ mkdir -p ~/grep-lab && cd ~/grep-lab
$ cat > notes.txt << 'EOF'
Linux is powerful
linux is free
I love LINUX
Windows is also an OS
macOS is based on Unix

The kernel is the core
error: disk full
Error: permission denied
warning: low memory
EOF
```

লাইন নম্বরসহ দেখি (ষষ্ঠ লাইনটা ফাঁকা, খেয়াল রাখো):

```
$ cat -n notes.txt
     1	Linux is powerful
     2	linux is free
     3	I love LINUX
     4	Windows is also an OS
     5	macOS is based on Unix
     6	
     7	The kernel is the core
     8	error: disk full
     9	Error: permission denied
    10	warning: low memory
```

---

## ২০.৪ grep-এর মৌলিক ব্যবহার

### Syntax

```
grep [OPTIONS] PATTERN [FILE...]
grep [OPTIONS] -e PATTERN ... [FILE...]
grep [OPTIONS] -f PATTERN_FILE ... [FILE...]
```

- `PATTERN` — যা খুঁজবে (সাধারণ শব্দ বা regex)
- `FILE` — কোথায় খুঁজবে। কোনো ফাইল না দিলে grep **stdin** (কীবোর্ড বা পাইপ) থেকে পড়ে।

### সবচেয়ে সহজ উদাহরণ

```
$ grep linux notes.txt
linux is free
```

খেয়াল করো — শুধু দ্বিতীয় লাইন এসেছে। কারণ grep ডিফল্টভাবে **case-sensitive** (বড় হাত-ছোট হাতের অক্ষর আলাদা ধরে)। `Linux` আর `LINUX` তার কাছে আলাদা শব্দ।

### পাইপ দিয়ে grep

grep সবচেয়ে বেশি ব্যবহার হয় অন্য কমান্ডের আউটপুট ছাঁকতে:

```
$ ls /etc | grep conf
adduser.conf
ca-certificates.conf
debconf.conf
...
```

```
$ history | grep ssh
  112  ssh user@192.168.1.20
  145  sudo systemctl restart ssh
```

```
     cmd1 (ls /etc)            grep conf               টার্মিনাল
   +---------------+  pipe  +-------------+  stdout  +----------+
   | সব নাম ছাপে    | -----> | ছাঁকনি: শুধু  | -------> | ফলাফল    |
   | (stdout)      |        | 'conf' লাইন  |          |          |
   +---------------+        +-------------+          +----------+
```

### প্যাটার্ন সবসময় single quote-এ রাখো

```
$ grep 'error: disk' notes.txt
error: disk full
```

> ⚠️ **সতর্কতা:** প্যাটার্নে space, `*`, `?`, `[`, `$`, `|` ইত্যাদি থাকলে সেটা অবশ্যই `'single quote'`-এ রাখো। নইলে **shell** (শেল) grep-এর আগেই এগুলোকে নিজের মতো ব্যাখ্যা করে ফেলবে। যেমন `grep a* file` লিখলে শেল `a*`-কে বর্তমান ফোল্ডারের `a` দিয়ে শুরু হওয়া ফাইলের নামে বদলে দিতে পারে (globbing) — grep তখন ভুল জিনিস খুঁজবে। Double quote-এর ভেতরে `$` আর `` ` `` তখনও শেল ব্যাখ্যা করে, তাই regex-এর জন্য single quote সবচেয়ে নিরাপদ।

---

## ২০.৫ grep-এর অপশন — একটা একটা করে

### সব গুরুত্বপূর্ণ অপশনের টেবিল

| অপশন | লম্বা রূপ | কাজ |
|---|---|---|
| `-i` | `--ignore-case` | বড়/ছোট হাতের অক্ষরের পার্থক্য উপেক্ষা |
| `-v` | `--invert-match` | যে লাইন **মেলে না** সেগুলো দেখাও |
| `-n` | `--line-number` | লাইন নম্বর দেখাও |
| `-c` | `--count` | কতগুলো লাইন মিলেছে শুধু সেই সংখ্যা |
| `-l` | `--files-with-matches` | শুধু যে ফাইলে মিল আছে তার নাম |
| `-L` | `--files-without-match` | শুধু যে ফাইলে মিল **নেই** তার নাম |
| `-r` | `--recursive` | ফোল্ডারের ভেতরে সব ফাইলে (সাবফোল্ডারসহ) খোঁজো |
| `-R` | `--dereference-recursive` | `-r`-এর মতো, কিন্তু সব symlink অনুসরণ করে |
| `-w` | `--word-regexp` | শুধু পুরো শব্দ মেলাও |
| `-x` | `--line-regexp` | পুরো লাইন হুবহু মিললে তবেই |
| `-o` | `--only-matching` | পুরো লাইন না, শুধু মেলা অংশটুকু |
| `-A N` | `--after-context=N` | মেলা লাইনের **পরের** N লাইনও দেখাও |
| `-B N` | `--before-context=N` | মেলা লাইনের **আগের** N লাইনও দেখাও |
| `-C N` | `--context=N` | আগে-পরে দুই দিকেই N লাইন |
| `-E` | `--extended-regexp` | Extended regex (ERE) |
| `-F` | `--fixed-strings` | প্যাটার্নকে regex না, সাধারণ লেখা হিসেবে ধরো |
| `-P` | `--perl-regexp` | Perl-compatible regex (PCRE) |
| `-G` | `--basic-regexp` | Basic regex (BRE) — এটাই ডিফল্ট |
| `-e PAT` | `--regexp=PAT` | প্যাটার্ন দাও (একাধিকবার দেওয়া যায়) |
| `-f FILE` | `--file=FILE` | প্যাটার্নগুলো একটা ফাইল থেকে পড়ো (প্রতি লাইনে একটা) |
| `-h` | `--no-filename` | আউটপুটে ফাইলের নাম দেখিও না |
| `-H` | `--with-filename` | সবসময় ফাইলের নাম দেখাও |
| `-m N` | `--max-count=N` | N-টা মেলা লাইন পেলেই থামো |
| `-q` | `--quiet`, `--silent` | কিছু ছাপবে না, শুধু exit status |
| `-s` | `--no-messages` | "ফাইল নেই"-জাতীয় error message লুকাও |
| `-b` | `--byte-offset` | ফাইলের শুরু থেকে কত বাইট পরে লাইনটা, দেখাও |
| `-a` | `--text` | binary ফাইলকেও টেক্সট হিসেবে পড়ো |
| `-I` | — | binary ফাইল বাদ দাও |
| `-Z` | `--null` | ফাইলের নামের পরে newline-এর বদলে NUL বাইট |
| `-z` | `--null-data` | ইনপুটের "লাইন" NUL দিয়ে আলাদা ধরো |
| — | `--color[=WHEN]` | মিল রঙিন করো (`never`, `always`, `auto`) |
| — | `--include=GLOB` | শুধু এই নামের ফাইলে খোঁজো (`-r`-এর সাথে) |
| — | `--exclude=GLOB` | এই নামের ফাইল বাদ দাও |
| — | `--exclude-dir=GLOB` | এই নামের ফোল্ডার বাদ দাও |
| — | `--line-buffered` | প্রতি লাইন সাথে সাথে আউটপুট দাও (পাইপে লাইভ লগের জন্য) |

এবার প্রতিটা বিস্তারিত দেখি।

### `-i` — case উপেক্ষা

```
$ grep -i linux notes.txt
Linux is powerful
linux is free
I love LINUX
```

### `-n` — লাইন নম্বর

```
$ grep -n -i linux notes.txt
1:Linux is powerful
2:linux is free
3:I love LINUX
```

অপশনগুলো একসাথে জুড়ে লেখা যায়: `grep -ni linux notes.txt` একই কাজ করে।

> 💡 **টিপস:** লাইন নম্বর জানলে সরাসরি এডিটরে সেই লাইনে যেতে পারো: `vim +3 notes.txt` বা `nano +3 notes.txt`। ([অধ্যায় ১৩](13-nano-editor.md), [অধ্যায় ১৪](14-vim.md))

### `-v` — উল্টো খোঁজা (invert)

যে লাইনে `linux` (যেকোনো case) **নেই**, সেগুলো:

```
$ grep -vi linux notes.txt
Windows is also an OS
macOS is based on Unix

The kernel is the core
error: disk full
Error: permission denied
warning: low memory
```

ফাঁকা লাইনটাও এসেছে, কারণ ওতে `linux` নেই।

বাস্তব ব্যবহার — কনফিগ ফাইল থেকে comment (`#`) আর ফাঁকা লাইন বাদ দিয়ে শুধু আসল সেটিং দেখা:

```
$ grep -v '^#' /etc/ssh/sshd_config | grep -v '^$'
Include /etc/ssh/sshd_config.d/*.conf
KbdInteractiveAuthentication no
UsePAM yes
...
```

(একটা কমান্ডেই: `grep -Ev '^(#|$)' /etc/ssh/sshd_config` — `-E` আর `|` নিচে শিখবে।)

### `-c` — গুনে দেখা

```
$ grep -ci linux notes.txt
3
$ grep -vci linux notes.txt
7
```

> ⚠️ **সতর্কতা:** `-c` **লাইন** গোনে, মিলের সংখ্যা না। একটা লাইনে `linux` তিনবার থাকলেও সেটা `1` গোনা হবে। মোট কতবার মিলেছে জানতে: `grep -o linux file | wc -l`।

একাধিক ফাইলে `-c` দিলে প্রতিটা ফাইলের আলাদা হিসাব আসে (যার মধ্যে মিল নেই, সেটা `0`)।

### `-w` — পুরো শব্দ মেলানো

এটা খুবই গুরুত্বপূর্ণ, কারণ সাধারণ grep শব্দের **ভেতরের** অংশও মিলিয়ে ফেলে:

```
$ grep -n 'is' notes.txt
1:Linux is powerful
2:linux is free
4:Windows is also an OS
5:macOS is based on Unix
7:The kernel is the core
8:error: disk full
9:Error: permission denied
```

৮ নম্বর লাইনে `is` কোথায়? → d**is**k! ৯ নম্বরে → perm**is**sion! এবার `-w` দিয়ে:

```
$ grep -nw 'is' notes.txt
1:Linux is powerful
2:linux is free
4:Windows is also an OS
5:macOS is based on Unix
7:The kernel is the core
```

`-w`-এর নিয়ম: মেলা অংশের আগে ও পরে হয় লাইনের শুরু/শেষ থাকবে, নয়তো এমন অক্ষর যেটা "word character" না। word character মানে অক্ষর, সংখ্যা আর underscore (`_`)।

### `-x` — পুরো লাইন হুবহু

```
$ grep -x 'linux is free' notes.txt
linux is free
$ grep -x 'linux' notes.txt
$
```

দ্বিতীয়টায় কিছুই আসেনি, কারণ কোনো লাইন **শুধু** `linux` না। `-x` অনেকটা প্যাটার্নের দুই পাশে `^` আর `$` বসানোর মতো (নিচে দেখবে)।

### `-o` — শুধু মেলা অংশ

```
$ grep -oi 'linux' notes.txt
Linux
linux
LINUX
```

ডেটা বের করার (extract) কাজে `-o` সবচেয়ে দরকারি — IP, email, URL বের করা এটা দিয়েই হয়। একই লাইনে একাধিক মিল থাকলে প্রতিটা আলাদা লাইনে আসে।

### `-A`, `-B`, `-C` — আশেপাশের লাইন (context)

লগ পড়ার সময় শুধু error লাইনটা যথেষ্ট না — তার আগে-পরে কী হয়েছিল সেটাও জানা দরকার।

```
$ grep -A 1 kernel notes.txt
The kernel is the core
error: disk full

$ grep -B 1 kernel notes.txt

The kernel is the core

$ grep -n -C 1 kernel notes.txt
6-
7:The kernel is the core
8-error: disk full
```

খেয়াল করো: `-n`-এর সাথে **মেলা লাইনের** নম্বরের পরে `:` আর **context লাইনের** পরে `-` বসে। এভাবে চোখে পড়ে কোনটা আসল মিল।

একাধিক মিলের মাঝে ফাঁক থাকলে grep `--` দিয়ে দলগুলো আলাদা করে:

```
$ grep -A 1 -i '^error' notes.txt
error: disk full
Error: permission denied
warning: low memory
```

প্রথমটায় কোনো `--` নেই, কারণ ৮, ৯, ১০ নম্বর লাইন পরপর — দলগুলো জোড়া লেগে গেছে। এবার দূরের দুটো মিল:

```
$ grep -A1 -e Windows -e kernel notes.txt
Windows is also an OS
macOS is based on Unix
--
The kernel is the core
error: disk full
```

### `-l` ও `-L` — শুধু ফাইলের নাম

একটা ছোট প্রজেক্ট ফোল্ডার বানাই:

```
$ mkdir -p project/notes
$ printf 'import os\n\n# TODO: fix login\nprint("hello")\n' > project/app.py
$ echo 'TODO: write docs' > project/README.md
$ echo 'debug = true' > project/config.ini
$ echo 'buy milk' > project/notes/todo.txt
```

```
$ grep -rl TODO project
project/app.py
project/README.md

$ grep -rL TODO project
project/config.ini
project/notes/todo.txt
```

(ফাইলের ক্রম তোমার মেশিনে আলাদা হতে পারে — grep ডিরেক্টরি যে ক্রমে পড়ে, সেই ক্রমে দেখায়।)

### `-r` ও `-R` — ফোল্ডারের ভেতরে খোঁজা

```
$ grep -rn TODO project
project/app.py:3:# TODO: fix login
project/README.md:1:TODO: write docs
```

একাধিক ফাইলে খুঁজলে grep নিজে থেকেই প্রতিটা লাইনের আগে ফাইলের নাম বসায় (`file:line:text`)।

**`-r` বনাম `-R` পার্থক্য:**

| | `-r` | `-R` |
|---|---|---|
| কমান্ড লাইনে সরাসরি দেওয়া symlink | অনুসরণ করে | অনুসরণ করে |
| ফোল্ডারের ভেতরে পাওয়া symlink | **অনুসরণ করে না** | অনুসরণ করে |
| ঝুঁকি | কম | symlink লুপ বা বিশাল জায়গায় ঢুকে যেতে পারে |

symlink কী, সেটা [অধ্যায় ১১](11-links-inode.md)-এ আছে।

> 💡 **টিপস:** `grep -r pattern` লিখে কোনো ফোল্ডার না দিলে GNU grep বর্তমান ফোল্ডার (`.`) ধরে নেয়।

### `--include`, `--exclude`, `--exclude-dir` — কোন ফাইলে খুঁজবে

```
$ grep -rn --include='*.py' TODO project
project/app.py:3:# TODO: fix login

$ grep -rn --exclude='*.md' TODO project
project/app.py:3:# TODO: fix login

$ grep -rn --exclude-dir=notes buy project
$
```

বাস্তব ডেভেলপার জীবনে সবচেয়ে কাজের লাইন:

```
$ grep -rn --exclude-dir={.git,node_modules,vendor} 'function login' .
```

এখানে `{.git,node_modules,vendor}` হলো **bash-এর brace expansion** — শেল এটাকে তিনটা আলাদা `--exclude-dir=...` অপশনে ভেঙে দেয়। একাধিক এক্সটেনশন: `--include='*.php' --include='*.js'`।

> ⚠️ **সতর্কতা:** `--include`-এর glob-টাও quote-এ রাখো (`'*.py'`), নইলে বর্তমান ফোল্ডারে `.py` ফাইল থাকলে শেল সেটা আগেই expand করে দেবে।

### `-E`, `-F`, `-P`, `-G` — regex-এর ধরন বাছাই

এগুলো নিচে regex অংশে বিস্তারিত আছে। সংক্ষেপে:

```
$ grep -E 'error|warning' notes.txt          # ERE: | মানে "অথবা"
error: disk full
warning: low memory

$ grep -F 'a.b*c' file.txt                   # হুবহু "a.b*c" লেখাটাই খুঁজবে
$ grep -P '\d+' file.txt                     # Perl-ধাঁচের \d (সংখ্যা)
```

`-F` শুধু নিরাপদই না, দ্রুতও — কারণ regex ইঞ্জিন লাগে না। ডট, তারা, ব্র্যাকেটওয়ালা লেখা (যেমন IP `192.168.1.1` বা `array[0]`) হুবহু খুঁজতে `-F` দাও।

### `-e` ও `-f` — একাধিক প্যাটার্ন

```
$ grep -e error -e warning notes.txt
error: disk full
warning: low memory
```

প্যাটার্ন যদি `-` দিয়ে শুরু হয়, তাহলে grep সেটাকে অপশন ভেবে ভুল করবে। সমাধান `-e` বা `--`:

```
$ grep -e '-v' file.txt
$ grep -- '-v' file.txt
```

অনেকগুলো প্যাটার্ন একটা ফাইলে রেখে:

```
$ printf 'error\nwarning\n' > patterns.txt
$ grep -f patterns.txt notes.txt
error: disk full
warning: low memory
```

### `-h`, `-H`, `-m`, `-b`, `-q`, `-s`

```
$ grep -h TODO project/app.py project/README.md     # ফাইলের নাম ছাড়া
# TODO: fix login
TODO: write docs

$ grep -H kernel notes.txt                           # এক ফাইলেও নাম দেখাও
notes.txt:The kernel is the core

$ grep -m 1 -i linux notes.txt                       # প্রথম মিলেই থামো
Linux is powerful

$ grep -b kernel notes.txt                           # বাইট অফসেট
91:The kernel is the core

$ grep -q kernel notes.txt && echo "পাওয়া গেছে"
পাওয়া গেছে

$ grep -s TODO nai.txt notes.txt                      # nai.txt না থাকলেও চুপ
$
```

### `--color` — রঙিন হাইলাইট

```
$ grep --color=auto kernel notes.txt
```

- `auto` — আউটপুট টার্মিনালে গেলে রঙ, পাইপ/ফাইলে গেলে রঙ না
- `always` — সবসময় রঙ (escape code সহ)
- `never` — কখনো না

Ubuntu ও Debian-এর ডিফল্ট `~/.bashrc`-এ `alias grep='grep --color=auto'` আগে থেকেই থাকে, তাই রঙ এমনিতেই দেখো। Fedora/RHEL-এ `/etc/profile.d/colorgrep.sh` একই কাজ করে।

রঙসহ আউটপুট `less`-এ দেখতে:

```
$ grep --color=always -rn TODO . | less -R
```

রঙ বদলাতে `GREP_COLORS` এনভায়রনমেন্ট ভেরিয়েবল (যেমন `export GREP_COLORS='mt=01;32'` — মিল সবুজ, বোল্ড)। পুরোনো `GREP_COLOR` ভেরিয়েবল এখন obsolete; নতুন grep এটা দেখলে warning দেয়। এনভায়রনমেন্ট ভেরিয়েবল নিয়ে বিস্তারিত [অধ্যায় ৪৪](44-environment-bashrc.md)-এ।

### Binary ফাইল নিয়ে

```
$ grep -r ELF /usr/bin/ls
grep: /usr/bin/ls: binary file matches
```

grep দেখল ফাইলটা binary (ভেতরে NUL বাইট আছে), তাই আস্ত লাইন না ছেপে শুধু জানিয়ে দিল। বিকল্প:

- `-a` — জোর করে টেক্সট ধরে ছাপো (টার্মিনাল আবর্জনায় ভরে যেতে পারে)
- `-I` — binary ফাইল একেবারে বাদ দাও (কোড খোঁজার সময় কাজের)
- binary থেকে পড়ার মতো লেখা বের করতে `strings /usr/bin/ls | grep GLIBC`

### `-Z`/`-z` — অদ্ভুত ফাইলনাম সামলানো

ফাইলের নামে space থাকলে `grep -l ... | xargs rm` ভেঙে যায়। NUL দিয়ে আলাদা করলে নিরাপদ:

```
$ grep -rlZ TODO project | xargs -0 ls -l
```

`xargs` নিয়ে বিস্তারিত [অধ্যায় ২৩](23-text-tools.md)-এ।

### `--line-buffered` — লাইভ লগ

```
$ tail -f /var/log/nginx/access.log | grep --line-buffered ' 500 ' | tee errors.txt
```

পাইপে লেখার সময় grep আউটপুট জমিয়ে (buffer) একবারে পাঠায়; `--line-buffered` দিলে প্রতিটা লাইন সাথে সাথে পাঠায়, ফলে লাইভ দেখা যায়।

---

## ২০.৬ রেগুলার এক্সপ্রেশন (Regex) — খোঁজার ভাষা

### Regex কী?

**Regular expression** (রেগুলার এক্সপ্রেশন, সংক্ষেপে regex বা regexp) হলো একটা ছোট্ট "ভাষা", যেটা দিয়ে **লেখার নকশা** বর্ণনা করা হয়। "ঠিক এই শব্দটা খোঁজো" না বলে তুমি বলতে পারো:

- "যে লাইন `error` দিয়ে **শুরু**"
- "যেকোনো **তিন অঙ্কের সংখ্যা**"
- "`@` চিহ্নের দুই পাশে কিছু অক্ষর, তারপর ডট আর ডোমেইন" — মানে email

**উপমা:** ধরো তুমি পুলিশকে চোরের বর্ণনা দিচ্ছ — "লম্বা, কালো শার্ট, বাম হাতে ঘড়ি।" তুমি নাম জানো না, কিন্তু **বৈশিষ্ট্য** জানো। regex হলো লেখার জন্য এমন একটা বর্ণনা — নির্দিষ্ট লেখা না, লেখার **ধরন**।

Regex শুধু grep-এ না — `sed` ([অধ্যায় ২১](21-sed.md)), `awk` ([অধ্যায় ২২](22-awk.md)), `vim`, `less`, Python, JavaScript, PHP — প্রায় সব জায়গায় ব্যবহার হয়। একবার শিখলে সারাজীবন কাজে লাগবে।

### দুই ধরনের অক্ষর

1. **Literal (আক্ষরিক)** — নিজেকেই বোঝায়। `a` মানে `a`, `cat` মানে `c-a-t`।
2. **Metacharacter (মেটাক্যারেক্টার)** — বিশেষ ক্ষমতাওয়ালা অক্ষর, যেমন `.` মানে "যেকোনো একটা অক্ষর"।

### Regex ইঞ্জিন কীভাবে মেলায়

```
 প্যাটার্ন:  c.t            লেখা:  "the cat sat on a cot"

  অবস্থান 0:  t  -> 'c' চাই, 't' পেলাম ............ ব্যর্থ, এক ঘর এগোও
  অবস্থান 1:  h  -> ব্যর্থ
  অবস্থান 2:  e  -> ব্যর্থ
  অবস্থান 3:  ' '-> ব্যর্থ
  অবস্থান 4:  c  -> 'c' মিলল
              a  -> '.' যেকোনো অক্ষর, মিলল
              t  -> 't' মিলল ......................... মিল! "cat"
  (grep -o হলে এখান থেকে আবার খোঁজা শুরু, পরে "cot"-ও পাবে)
```

ইঞ্জিন বাম থেকে ডানে প্রতিটা অবস্থান থেকে মেলানোর চেষ্টা করে; প্রথম যেখানে মেলে, সেটাই নেয় — একে বলে **leftmost match** (সবচেয়ে বামের মিল)।

---

## ২০.৭ মেটাক্যারেক্টার — এক এক করে

নিচের উদাহরণগুলোতে ERE (`grep -E`) ব্যবহার করব, কারণ এতে backslash কম লাগে। BRE-তে কী আলাদা, সেটা পরের সেকশনে।

### সংক্ষিপ্ত তালিকা

| মেটাক্যারেক্টার | মানে | উদাহরণ | কী মেলে |
|---|---|---|---|
| `.` | যেকোনো একটা অক্ষর (newline বাদে) | `c.t` | cat, cot, c9t, c t |
| `*` | আগেরটা **০ বা তার বেশি** বার | `ab*c` | ac, abc, abbbc |
| `+` | আগেরটা **১ বা তার বেশি** বার | `ab+c` | abc, abbc (ac না) |
| `?` | আগেরটা **০ বা ১** বার (ঐচ্ছিক) | `colou?r` | color, colour |
| `^` | লাইনের **শুরু** | `^error` | "error" দিয়ে শুরু লাইন |
| `$` | লাইনের **শেষ** | `full$` | "full" দিয়ে শেষ লাইন |
| `[abc]` | ব্র্যাকেটের যেকোনো **একটা** অক্ষর | `gr[ae]y` | gray, grey |
| `[a-z]` | রেঞ্জ | `[0-9]` | যেকোনো একটা অঙ্ক |
| `[^abc]` | এগুলো **ছাড়া** যেকোনো একটা | `[^0-9]` | অঙ্ক নয় এমন অক্ষর |
| `{n}` | ঠিক n বার | `[0-9]{4}` | 2026 |
| `{n,}` | অন্তত n বার | `a{2,}` | aa, aaa, ... |
| `{n,m}` | n থেকে m বার | `[0-9]{1,3}` | 7, 42, 255 |
| `{,m}` | সর্বোচ্চ m বার (GNU) | `x{,2}` | (ফাঁকা), x, xx |
| `( )` | গ্রুপ (দল) | `(ab)+` | ab, abab |
| `\` | পরের মেটাক্যারেক্টারকে সাধারণ বানাও (escape) | `\.` | আসল ডট |
| `\b` | শব্দের সীমানা (word boundary) | `\bis\b` | আলাদা শব্দ "is" |
| `\B` | শব্দের সীমানা **নয়** | `\Bis\B` | d**is**k-এর is |
| `\<` `\>` | শব্দের শুরু / শেষ (GNU) | `\<un` | un দিয়ে শুরু শব্দ |
| `\w` | word character (অক্ষর, অঙ্ক, `_`) (GNU) | `\w+` | hello_123 |
| `\W` | word character নয় | | |
| `\s` / `\S` | whitespace / whitespace নয় (GNU) | `\s+` | স্পেস, ট্যাব |
| `\d` | অঙ্ক — **শুধু `-P` মোডে** | `\d{3}` | 123 |
| `\1`, `\2` | back-reference: আগের গ্রুপ যা মিলেছে, হুবহু তাই আবার | `(a)\1` | aa |

আরেকটা জরুরি মেটাক্যারেক্টার হলো **পাইপ চিহ্ন** — মানে "অথবা" (alternation)। ERE-তে `cat|dog` মানে cat অথবা dog (টেবিলের ভেতরে পাইপ লেখা ঝামেলা, তাই এখানে আলাদা করে বললাম)।

### `.` — যেকোনো একটা অক্ষর

```
$ echo -e "cat\ncut\ncart\nct" | grep 'c.t'
cat
cut
```

`cart`-এ `c` আর `t`-এর মাঝে দুটো অক্ষর, আর `ct`-তে কোনো অক্ষরই নেই — তাই মেলেনি।

> ⚠️ **সতর্কতা:** আসল ডট খুঁজতে `\.` লিখতে হবে। `grep '192.168.1.1'` লিখলে `192x168y1z1`-ও মিলে যাবে! নিরাপদ পথ: `grep -F '192.168.1.1'` অথবা `grep '192\.168\.1\.1'`।

### `^` ও `$` — anchor (নোঙর)

anchor কোনো অক্ষর মেলায় না, **অবস্থান** মেলায়।

```
$ grep -i '^error' notes.txt
error: disk full
Error: permission denied

$ grep 'ory$' notes.txt
warning: low memory

$ grep -c '^$' notes.txt        # ফাঁকা লাইন গোনা
1

$ grep -n '^.*$' notes.txt | head -2
1:Linux is powerful
2:linux is free
```

> 📝 **মনে রাখো:** `^` ব্র্যাকেটের **বাইরে** মানে "লাইনের শুরু", কিন্তু `[^...]`-এর **ভেতরে প্রথমে** থাকলে মানে "এগুলো ছাড়া"। একই চিহ্ন, জায়গাভেদে দুই মানে।

> ⚠️ **সতর্কতা (Windows ফাইল):** Windows-এ বানানো ফাইলের লাইনের শেষে অদৃশ্য `\r` (carriage return) থাকে। তখন `grep 'full$'` মেলে না, কারণ শেষ অক্ষর `l` না, `\r`। সমাধান: `dos2unix file` অথবা `grep $'full\r$'`। `cat -A file` দিলে লাইনের শেষে `^M$` দেখলেই বুঝবে।

### `[ ]` — character class (অক্ষরের দল)

```
$ echo -e "gray\ngrey\ngroy" | grep 'gr[ae]y'
gray
grey

$ echo -e "file1\nfileA\nfile_" | grep 'file[0-9]'
file1

$ echo -e "file1\nfileA\nfile_" | grep 'file[^0-9]'
fileA
file_
```

ব্র্যাকেটের ভেতরের নিয়ম:

| লেখা | মানে |
|---|---|
| `[abc]` | a, b, বা c |
| `[a-z]`, `[A-Z]`, `[0-9]` | রেঞ্জ |
| `[a-zA-Z0-9_]` | একাধিক রেঞ্জ একসাথে |
| `[^abc]` | a, b, c বাদে যেকোনো |
| `[]abc]` | `]`-কে সাধারণ অক্ষর রাখতে **প্রথমে** বসাও |
| `[abc-]` বা `[-abc]` | `-`-কে সাধারণ রাখতে শুরুতে বা শেষে বসাও |
| `[.*]` | ব্র্যাকেটের ভেতরে `.` আর `*` সাধারণ অক্ষর! escape লাগে না |

> ⚠️ **সতর্কতা (locale):** কিছু locale-এ `[a-z]` রেঞ্জ শুধু ছোট হাতের অক্ষরে সীমাবদ্ধ না-ও থাকতে পারে (পুরোনো সিস্টেমে এতে বড় হাতের অক্ষরও ঢুকে যেত)। আধুনিক GNU grep সাধারণত ঠিকঠাক আচরণ করে, তবু নিশ্চিত থাকতে POSIX class (`[[:lower:]]`) অথবা `LC_ALL=C grep ...` ব্যবহার করো।

### `*`, `+`, `?` — quantifier (কতবার)

quantifier সবসময় তার **ঠিক আগের** জিনিসটার উপর কাজ করে — একটা অক্ষর, একটা `[...]`, বা একটা `(...)` গ্রুপ।

```
$ echo -e "ac\nabc\nabbbc" | grep -E 'ab*c'
ac
abc
abbbc

$ echo -e "ac\nabc\nabbbc" | grep -E 'ab+c'
abc
abbbc

$ echo -e "color\ncolour\ncolouur" | grep -E '^colou?r$'
color
colour
```

> ⚠️ **সতর্কতা — শুরুর সবচেয়ে কমন ভুল:** shell-এর glob-এ `*` মানে "যেকোনো কিছু", কিন্তু regex-এ `*` মানে "**আগেরটা** ০ বা বেশিবার"। regex-এ "যেকোনো কিছু" হলো `.*`। আর `grep 'a*'` প্রতিটা লাইনেই মিলবে — কারণ ০টা `a`-ও তো মিল!

### `{n,m}` — নির্দিষ্ট সংখ্যক বার

```
$ echo -e "7\n42\n255\n1000" | grep -E '^[0-9]{1,3}$'
7
42
255

$ echo -e "01712345678\n0171234\n02812345678" | grep -E '^01[3-9][0-9]{8}$'
01712345678
```

শেষেরটা বাংলাদেশি মোবাইল নম্বর যাচাই: `01`, তারপর ৩–৯ এর একটা অঙ্ক, তারপর ঠিক ৮টা অঙ্ক = মোট ১১ অঙ্ক।

### `( )` ও `|` — গ্রুপ ও বিকল্প

```
$ grep -E 'error|warning' notes.txt
error: disk full
warning: low memory

$ echo -e "ha\nhaha\nhahaha\nhah" | grep -Ex '(ha)+'
ha
haha
hahaha
```

`|`-এর অগ্রাধিকার সবচেয়ে কম। `^error|warning$` মানে "(`^error`) অথবা (`warning$`)", **না** যে "^(error|warning)$"। সীমা ঠিক করতে বন্ধনী দাও:

```
$ grep -Ei '^(error|warning):' notes.txt
error: disk full
Error: permission denied
warning: low memory
```

### Back-reference `\1` — "আগে যা পেয়েছি, আবার সেটাই"

```
$ echo -e "the the cat\nthe cat\nis is" | grep -E '\b(\w+) \1\b'
the the cat
is is
```

এটা ডুপ্লিকেট শব্দ ধরে — লেখা প্রুফরিড করার জন্য দারুণ। `(\w+)` যা মেলাবে, `\1` ঠিক সেটাকেই আবার চায়।

> 📝 **মনে রাখো:** POSIX-এ back-reference শুধু BRE-তে আছে; GNU grep ERE-তেও দেয়। Back-reference থাকলে grep দ্রুত DFA বাদ দিয়ে ধীর ইঞ্জিনে চলে, বড় ফাইলে স্লো লাগতে পারে।

### `\b`, `\<`, `\>` — শব্দের সীমানা

```
$ grep -n '\bis\b' notes.txt      # grep -w 'is' এর মতোই ফল
1:Linux is powerful
2:linux is free
4:Windows is also an OS
5:macOS is based on Unix
7:The kernel is the core

$ grep -o '\<[Ww][a-z]*' notes.txt
Windows
warning
```

`\b` শব্দের শুরু বা শেষ দুটোই বোঝায়; `\<` শুধু শুরু, `\>` শুধু শেষ।

### `\d` — অঙ্ক (শুধু PCRE)

```
$ echo "Order 4521 shipped" | grep -oP '\d+'
4521

$ echo "Order 4521 shipped" | grep -oE '[0-9]+'
4521
```

> ⚠️ **সতর্কতা:** `\d` POSIX regex-এর অংশ **না**। `-P` ছাড়া `grep '\d'` লিখলে GNU grep সেটাকে সাধারণ `d` অক্ষর হিসেবে মেলায় — Ubuntu 24.04-এর grep 3.11-এ কোনো error বা warning ছাড়াই! মানে `echo xd | grep '\d'` মিলে যাবে, অথচ কোনো সংখ্যাই নেই। এটা নিঃশব্দ ভুল, তাই সবচেয়ে বিপজ্জনক। BRE/ERE-তে অঙ্কের জন্য `[0-9]` বা `[[:digit:]]` লেখো।

---

## ২০.৮ POSIX character class

`[[:name:]]` রূপের এই class-গুলো locale-নিরাপদ এবং সব POSIX টুলে (grep, sed, awk, tr) কাজ করে।

| Class | মানে | সমতুল্য (ASCII-তে) |
|---|---|---|
| `[[:digit:]]` | অঙ্ক | `[0-9]` |
| `[[:alpha:]]` | বর্ণ | `[A-Za-z]` |
| `[[:alnum:]]` | বর্ণ বা অঙ্ক | `[A-Za-z0-9]` |
| `[[:lower:]]` | ছোট হাতের | `[a-z]` |
| `[[:upper:]]` | বড় হাতের | `[A-Z]` |
| `[[:space:]]` | সব whitespace (space, tab, newline, CR, FF, VT) | `[ \t\n\r\f\v]` |
| `[[:blank:]]` | শুধু space আর tab | `[ \t]` |
| `[[:punct:]]` | যতিচিহ্ন | `` [!-/:-@[-`{-~] `` |
| `[[:xdigit:]]` | hex অঙ্ক | `[0-9A-Fa-f]` |
| `[[:print:]]` | ছাপার যোগ্য (space সহ) | |
| `[[:graph:]]` | ছাপার যোগ্য (space ছাড়া) | |
| `[[:cntrl:]]` | control character | |

**ডাবল ব্র্যাকেট কেন?** বাইরের `[ ]` হলো সাধারণ ব্র্যাকেট এক্সপ্রেশন, ভেতরের `[:digit:]` হলো তার একটা সদস্য। তাই অন্যগুলোর সাথে মেশানো যায়: `[[:digit:]_-]` মানে "অঙ্ক, underscore বা হাইফেন"।

```
$ echo -e "abc\nabc123\n   \n" | grep -c '^[[:space:]]*$'
2

$ echo "MAC: 3c:22:fb:9a:01:ee" | grep -oE '([[:xdigit:]]{2}:){5}[[:xdigit:]]{2}'
3c:22:fb:9a:01:ee
```

(প্রথম উদাহরণে `echo -e` শেষে নিজের একটা newline যোগ করে, তাই `"   "` লাইন আর একটা পুরো ফাঁকা লাইন — মোট ২টা।)

> ⚠️ **সতর্কতা:** `grep '[:digit:]'` (একক ব্র্যাকেট) ভুল। GNU grep তখন error দেয়:
> ```
> $ grep '[:digit:]' notes.txt
> grep: character class syntax is [[:space:]], not [:space:]
> ```

---

## ২০.৯ BRE বনাম ERE বনাম PCRE

ইতিহাসের কারণে regex-এর তিনটা প্রধান "উপভাষা" (dialect) আছে:

- **BRE (Basic Regular Expression)** — সবচেয়ে পুরোনো। grep-এর ডিফল্ট (`grep` বা `grep -G`)। এখানে `+ ? | ( ) { }` সাধারণ অক্ষর; বিশেষ ক্ষমতা পেতে সামনে `\` লাগে।
- **ERE (Extended Regular Expression)** — `grep -E`। এখানে ওই চিহ্নগুলো সরাসরি বিশেষ; সাধারণ অক্ষর হিসেবে চাইলে `\` লাগে। পড়তে অনেক সহজ।
- **PCRE (Perl Compatible Regular Expression)** — `grep -P`। Perl ভাষা থেকে আসা সবচেয়ে শক্তিশালী উপভাষা; Python, JavaScript, PHP-র regex-ও প্রায় এমন। `\d`, lazy quantifier, lookahead/lookbehind এখানে আছে।

### তুলনামূলক টেবিল

| ফিচার | BRE (`grep`) | ERE (`grep -E`) | PCRE (`grep -P`) |
|---|---|---|---|
| যেকোনো অক্ষর | `.` | `.` | `.` |
| ০ বা বেশি | `*` | `*` | `*` |
| ১ বা বেশি | `\+` (GNU) | `+` | `+` |
| ০ বা ১ | `\?` (GNU) | `?` | `?` |
| অথবা (alternation) | টেবিলের নিচে দেখো | টেবিলের নিচে দেখো | টেবিলের নিচে দেখো |
| গ্রুপ | `\( \)` | `( )` | `( )` |
| পুনরাবৃত্তি | `\{n,m\}` | `{n,m}` | `{n,m}` |
| back-reference | `\1` | `\1` (GNU) | `\1` |
| `\b \w \s` | আছে (GNU) | আছে (GNU) | আছে |
| `\d` (অঙ্ক) | নেই | নেই | আছে |
| lazy `*?` `+?` | নেই | নেই | আছে |
| lookahead `(?=...)` / lookbehind `(?<=...)` | নেই | নেই | আছে |
| `\K` (এর আগের অংশ ফলাফল থেকে বাদ) | নেই | নেই | আছে |
| case-insensitive অংশ `(?i)` | নেই | নেই | আছে |

**অথবা-চিহ্ন:** BRE-তে `a\|b` (GNU এক্সটেনশন), ERE ও PCRE-তে সরাসরি `a|b`।

একই প্যাটার্ন তিনভাবে — "`ab` এক বা একাধিকবার, তারপর `c` বা `d`":

```
$ echo "ababd" | grep    '\(ab\)\+\(c\|d\)'      # BRE
ababd
$ echo "ababd" | grep -E '(ab)+(c|d)'            # ERE
ababd
$ echo "ababd" | grep -P '(ab)+(c|d)'            # PCRE
ababd
```

BRE-তে backslash-এর জঙ্গল দেখেছ? এ কারণেই বেশিরভাগ মানুষ `grep -E` ব্যবহার করে।

```
         regex-এর শক্তি ও জটিলতা
   কম  <------------------------------------>  বেশি

   grep -F      grep (BRE)     grep -E (ERE)     grep -P (PCRE)
   হুবহু লেখা    পুরোনো, \ বেশি   পরিষ্কার, জনপ্রিয়    সবচেয়ে শক্তিশালী
   সবচেয়ে দ্রুত    দ্রুত (DFA)     দ্রুত (DFA)        backtracking, কখনো ধীর
```

### PCRE-র বিশেষ ক্ষমতা — উদাহরণ

**Lookbehind + `\K` দিয়ে শুধু মান বের করা:**

```
$ echo "user=rahim id=42" | grep -oP 'user=\K\w+'
rahim

$ echo "user=rahim id=42" | grep -oP '(?<=id=)\d+'
42
```

**Lookahead — পরে কিছু আছে এমন অংশ:**

```
$ echo "price: 500taka 20usd" | grep -oP '\d+(?=taka)'
500
```

**Case-insensitive একটা অংশ:**

```
$ echo "LINUX rocks" | grep -P '(?i)linux'
LINUX rocks
```

> 💡 **টিপস:** `-P` GNU grep-এর একটা ঐচ্ছিক ফিচার — PCRE লাইব্রেরি নিয়ে কম্পাইল করা থাকতে হয়। Ubuntu, Debian, Fedora, RHEL-এর grep-এ এটা আছে, কিন্তু macOS বা Alpine-এর (BusyBox) grep-এ নেই। পোর্টেবল স্ক্রিপ্টে তাই ERE-কেই প্রাধান্য দাও।

> ⚠️ **সতর্কতা:** PCRE backtracking ইঞ্জিন। `(a+)+$`-এর মতো প্যাটার্ন লম্বা লেখায় অত্যধিক সময় নিতে পারে (catastrophic backtracking)। GNU grep তখন সীমা পেরোলে `exceeded PCRE's backtracking limit` জাতীয় error দেয়। নেস্টেড quantifier এড়িয়ে চলো।

---

## ২০.১০ Greedy matching — "লোভী" মিল

regex-এর `*`, `+`, `?`, `{n,m}` ডিফল্টভাবে **greedy** (লোভী) — যতটা সম্ভব **বেশি** অক্ষর গিলে ফেলে, তবে পুরো প্যাটার্ন যেন মেলে সেটা নিশ্চিত রেখে।

**উপমা:** বুফেতে একজন লোভী মানুষ প্লেট যতটা ধরে ততটা খাবার তোলে; শুধু শেষ আইটেম রাখার জায়গা না থাকলে একটু ফেরত দেয়।

```
$ echo '<b>bold</b> and <i>italic</i>' | grep -o '<.*>'
<b>bold</b> and <i>italic</i>
```

আমরা চেয়েছিলাম `<b>`, `</b>` ইত্যাদি আলাদা ট্যাগ, পেলাম পুরো লাইন! কারণ:

```
  <b>bold</b> and <i>italic</i>
  ^                           ^
  '<' এখানে মিলল              '.*' লাইনের শেষ পর্যন্ত গিলে নিল,
                              তারপর '>' পেতে একটু একটু পিছিয়ে (backtrack)
                              সবচেয়ে শেষের '>'-এ থামল
```

**সমাধান ১ — negated class (POSIX, সব জায়গায় চলে, দ্রুত):**

```
$ echo '<b>bold</b> and <i>italic</i>' | grep -o '<[^>]*>'
<b>
</b>
<i>
</i>
```

`[^>]*` মানে "`>` ছাড়া যেকোনো কিছু" — তাই প্রথম `>`-এর আগেই থামতে বাধ্য।

**সমাধান ২ — lazy quantifier (শুধু PCRE):**

```
$ echo '<b>bold</b> and <i>italic</i>' | grep -oP '<.*?>'
<b>
</b>
<i>
</i>
```

`*?` মানে "যত **কম** সম্ভব"। PCRE-তে `+?`, `??`, `{n,m}?`-ও আছে।

> 📝 **মনে রাখো:** POSIX (BRE/ERE) ইঞ্জিন "leftmost-longest" নিয়মে চলে — সবচেয়ে বামের মিল, এবং সেখান থেকে **সবচেয়ে লম্বা** মিল। তাই BRE/ERE-তে lazy বলে কিছু নেই; negated class-ই ভরসা।

---

## ২০.১১ egrep ও fgrep — পুরোনো নাম

আগে তিনটা আলাদা প্রোগ্রাম ছিল: `grep` (BRE), `egrep` (ERE), `fgrep` (fixed string)। এখন সব কাজ এক `grep` দিয়েই হয়:

| পুরোনো | আধুনিক সমতুল্য |
|---|---|
| `egrep '(ab)+'` | `grep -E '(ab)+'` |
| `fgrep 'a.b'` | `grep -F 'a.b'` |

GNU grep ৩.৮ (২০২২) থেকে এগুলোকে আনুষ্ঠানিকভাবে obsolescent (বিলুপ্তির পথে) ঘোষণা করা হয়েছে। আসল GNU সংস্করণের `egrep` চালালে এমন warning আসে:

```
$ egrep 'error|warning' notes.txt
egrep: warning: egrep is obsolescent; using grep -E
error: disk full
warning: low memory
```

Debian ও Ubuntu (Debian 12, Ubuntu 24.04) নিজেদের প্যাকেজে `egrep`/`fgrep`-কে ছোট্ট wrapper স্ক্রিপ্ট বানিয়ে রেখেছে, যেটা warning ছাড়াই `grep -E`/`grep -F` চালায় — তাই সেখানে warning না-ও দেখতে পারো। তবু অন্য distro-তে বা ভবিষ্যতে এগুলো সরে যেতে পারে, তাই পুরোনো স্ক্রিপ্টে `egrep` থাকলে `grep -E`-এ বদলে নাও।

### grep পরিবারের অন্য সদস্য

| কমান্ড | কাজ |
|---|---|
| `zgrep` | `.gz` কম্প্রেস করা ফাইলে খোঁজে (যেমন `/var/log/syslog.2.gz`) — খোলা ছাড়াই |
| `xzgrep`, `bzgrep`, `zstdgrep` | `.xz`, `.bz2`, `.zst` ফাইলের জন্য (প্যাকেজ ইনস্টল থাকলে) |
| `pgrep` | নাম দিয়ে প্রসেস খোঁজে ([অধ্যায় ২৪](24-process-management.md)) |
| `journalctl -g PATTERN` | systemd journal-এ regex দিয়ে খোঁজে ([অধ্যায় ২৮](28-logs.md)) |
| `git grep` | git রিপোজিটরিতে ট্র্যাক করা ফাইলে খোঁজে |

```
$ zgrep -i 'failed' /var/log/auth.log.2.gz
```

---

## ২০.১২ ripgrep (`rg`) — আধুনিক, তুফান-গতির grep

**ripgrep** হলো Rust ভাষায় লেখা একটা নতুন টুল (কমান্ড `rg`)। কোড খোঁজার জন্য এটা এখন ডেভেলপারদের প্রিয়। VS Code-এর "Search in files" ভেতরে ripgrep-ই ব্যবহার করে।

### কেন দ্রুত ও সুবিধাজনক?

- **ডিফল্টভাবে recursive** — `-r` লাগে না।
- **`.gitignore` মানে** — `node_modules`, build ফোল্ডার ইত্যাদি নিজেই বাদ দেয়।
- **লুকানো ফাইল (`.` দিয়ে শুরু) আর binary ফাইল ডিফল্টভাবে বাদ।**
- **একাধিক CPU core** ব্যবহার করে অনেক ফাইল একসাথে (parallel) পড়ে।
- টার্মিনালে লাইন নম্বর, রঙ, ফাইল অনুযায়ী গ্রুপ — সব ডিফল্ট।
- regex ইঞ্জিন Rust-এর `regex` crate — backtracking ছাড়া, তাই catastrophic backtracking হয় না।

### ইনস্টলেশন

```
$ sudo apt install ripgrep          # Ubuntu / Debian
$ sudo dnf install ripgrep          # Fedora (RHEL 9-এ EPEL রিপো চালু করে)
```

প্যাকেজ ম্যানেজমেন্ট নিয়ে বিস্তারিত [অধ্যায় ৩০](30-package-management.md)-এ।

### ব্যবহার

```
$ rg TODO project
project/app.py
3:# TODO: fix login

project/README.md
1:TODO: write docs
```

(ফাইলের ক্রম বদলাতে পারে, কারণ rg একসাথে অনেক ফাইল পড়ে।)

### গুরুত্বপূর্ণ অপশন

| অপশন | কাজ | grep-এর সমতুল্য |
|---|---|---|
| `-i` | case উপেক্ষা | `-i` |
| `-S` / `--smart-case` | প্যাটার্ন সব ছোট হাতের হলে case উপেক্ষা, বড় হাতের থাকলে না | — |
| `-w` | পুরো শব্দ | `-w` |
| `-F` | হুবহু লেখা | `-F` |
| `-v` | উল্টো | `-v` |
| `-l` | শুধু ফাইলের নাম | `-l` |
| `-c` | ফাইলপ্রতি মেলা লাইনের সংখ্যা | `-c` |
| `-o` | শুধু মেলা অংশ | `-o` |
| `-A/-B/-C N` | context | একই |
| `-t TYPE` | শুধু এই ধরনের ফাইল (`-t py`, `-t js`) | `--include` |
| `-T TYPE` | এই ধরন বাদ | `--exclude` |
| `-g GLOB` | glob দিয়ে বাছাই; `!` দিয়ে বাদ (`-g '!*.min.js'`) | `--include/--exclude` |
| `--hidden` | লুকানো ফাইলেও খোঁজো | — |
| `--no-ignore` | `.gitignore` উপেক্ষা করো | — |
| `-u`, `-uu`, `-uuu` | ধাপে ধাপে ফিল্টার বন্ধ (ignore → hidden → binary) | — |
| `-P` | PCRE2 ইঞ্জিন (lookaround ইত্যাদির জন্য) | `-P` |
| `-r TEXT` | আউটপুটে মিলের জায়গায় TEXT বসিয়ে **দেখাও** (ফাইল বদলায় না) | — |
| `--files` | কোন কোন ফাইলে খুঁজবে তার তালিকা | — |
| `--type-list` | সব ফাইল-টাইপের তালিকা | — |

```
$ rg -t py TODO                    # শুধু Python ফাইলে
$ rg -S 'Login'                    # বড় হাতের আছে, তাই case-sensitive
$ rg -g '!*.test.js' 'fetch\('     # test ফাইল বাদ
$ rg -uuu 'password'               # কিছুই বাদ না, grep -r এর মতো
```

> ⚠️ **সতর্কতা:** rg-এর `-r` মানে **replace**, grep-এর মতো recursive **না**! আর rg-এর regex উপভাষা ERE/PCRE-এর কাছাকাছি (`\d` চলে, `+ ? |` সরাসরি), কিন্তু back-reference আর lookaround শুধু `-P` দিলে চলে।

> 💡 **টিপস:** সার্ভারে কিছু ইনস্টল করার অনুমতি না থাকলে সবখানে আছে এমন `grep`-ই ভরসা। তাই grep ভালো করে শেখো, rg বোনাস।

---

## ২০.১৩ বাস্তব উদাহরণ

### ১. লগ থেকে IP ঠিকানা বের করা

নমুনা ওয়েব লগ:

```
$ cat > server.log << 'EOF'
2026-11-01 10:15:01 192.168.1.10 GET /index.html 200
2026-11-01 10:15:07 10.0.0.5 POST /login 401
2026-11-01 10:16:22 192.168.1.10 GET /about.html 200
2026-11-01 10:17:45 203.0.113.7 POST /login 401
2026-11-01 10:18:03 203.0.113.7 POST /login 401
2026-11-01 10:18:09 999.300.1.2 GET /test 404
EOF
```

**সহজ IP প্যাটার্ন:** চারটা ১–৩ অঙ্কের সংখ্যা, মাঝে ডট।

```
$ grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' server.log
192.168.1.10
10.0.0.5
192.168.1.10
203.0.113.7
203.0.113.7
999.300.1.2
```

প্যাটার্নটা ভেঙে দেখি:

```
  \b  ( [0-9]{1,3}  \. ) {3}   [0-9]{1,3}  \b
  |   |  |          |    |     |           |
  |   |  ১-৩ অঙ্ক   আসল  এই দলটা  শেষ অংশ     শব্দের
  |   |             ডট   ৩ বার    ১-৩ অঙ্ক    সীমানা
  |   গ্রুপ শুরু
  শব্দের সীমানা (যাতে 1192.168... এর মাঝখান থেকে না ধরে)
```

কিন্তু `999.300.1.2` বৈধ IP না (প্রতিটা অংশ সর্বোচ্চ 255)! **কড়া (strict) প্যাটার্ন:**

```
$ grep -oE '\b((25[0-5]|2[0-4][0-9]|1[0-9]{2}|[1-9]?[0-9])\.){3}(25[0-5]|2[0-4][0-9]|1[0-9]{2}|[1-9]?[0-9])\b' server.log
192.168.1.10
10.0.0.5
192.168.1.10
203.0.113.7
203.0.113.7
```

একটা অংশ (octet) এভাবে ভাঙা হয়েছে:

| বিকল্প | কী মেলে |
|---|---|
| `25[0-5]` | 250–255 |
| `2[0-4][0-9]` | 200–249 |
| `1[0-9]{2}` | 100–199 |
| `[1-9]?[0-9]` | 0–99 |

**কোন IP কতবার এসেছে** (পাইপলাইন, `sort`/`uniq` নিয়ে বিস্তারিত [অধ্যায় ২৩](23-text-tools.md)):

```
$ grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' server.log | sort | uniq -c | sort -rn
      2 203.0.113.7
      2 192.168.1.10
      1 999.300.1.2
      1 10.0.0.5
```

**শুধু ব্যর্থ লগইন (401) করা IP:**

```
$ grep ' 401$' server.log | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' | sort -u
10.0.0.5
203.0.113.7
```

**SSH-এ brute-force আক্রমণ খোঁজা** (Ubuntu/Debian-এ `/var/log/auth.log`, Fedora/RHEL-এ `/var/log/secure` বা `journalctl`):

```
$ sudo grep 'Failed password' /var/log/auth.log | grep -oE 'from ([0-9]{1,3}\.){3}[0-9]{1,3}' | awk '{print $2}' | sort | uniq -c | sort -rn | head
     57 203.0.113.50
     12 198.51.100.23
```

```
$ sudo journalctl -u ssh --since today | grep -i 'failed password'     # systemd সিস্টেমে
```

(Ubuntu-তে সার্ভিসের নাম `ssh`, Fedora/RHEL-এ `sshd`। লগ নিয়ে পুরো আলোচনা [অধ্যায় ২৮](28-logs.md)-এ, নিরাপত্তা [অধ্যায় ৪৫](45-security-hardening.md)-এ।)

### ২. ফাইল থেকে email ঠিকানা বের করা

```
$ cat > contacts.txt << 'EOF'
Contact rahim@example.com for sales.
Support: support.team@mail.example.org
Invalid: karim@ and @nodomain.com
Fatema: fatema_99@gmail.com, backup fatema.b@yahoo.co.uk
EOF

$ grep -oE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' contacts.txt
rahim@example.com
support.team@mail.example.org
fatema_99@gmail.com
fatema.b@yahoo.co.uk
```

```
  [A-Za-z0-9._%+-]+   @   [A-Za-z0-9.-]+   \.   [A-Za-z]{2,}
  |                   |   |                |    |
  username অংশ        @   ডোমেইন            ডট   TLD (com, org, uk ...)
  (১ বা বেশি অক্ষর)        (১ বা বেশি)            কমপক্ষে ২ অক্ষর
```

`karim@` (ডোমেইন নেই) আর `@nodomain.com` (username নেই) বাদ পড়েছে। খেয়াল করো — `[A-Za-z0-9._%+-]`-এ `-` শেষে রাখা হয়েছে, যাতে রেঞ্জ না বোঝায়।

> 📝 **মনে রাখো:** পুরোপুরি নিখুঁত email regex (RFC 5322) বিশাল আর অব্যবহারিক। বাস্তবে উপরের মতো "যথেষ্ট ভালো" প্যাটার্নই চলে।

কোন ডোমেইন কতবার:

```
$ grep -oE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' contacts.txt | cut -d@ -f2 | sort | uniq -c
      1 example.com
      1 gmail.com
      1 mail.example.org
      1 yahoo.co.uk
```

### ৩. আরও কাজের রেসিপি

```
# সিস্টেমে কোন কোন ইউজারের লগইন শেল bash
$ grep 'bash$' /etc/passwd

# UID 1000 বা তার বেশি (সাধারণ ইউজার) — awk দিয়ে আরও সহজ, অধ্যায় ২২
$ grep -E '^[^:]+:[^:]*:[0-9]{4,}:' /etc/passwd

# ফাঁকা ও comment লাইন বাদে আসল কনফিগ
$ grep -Ev '^[[:space:]]*(#|$)' /etc/nginx/nginx.conf

# কোডবেসে কোনো ফাংশন কোথায় কল হয়েছে, লাইন নম্বরসহ
$ grep -rnw --include='*.php' 'sendMail' app/

# URL বের করা
$ grep -oE 'https?://[^[:space:]"<>]+' page.html

# তারিখ (YYYY-MM-DD) বের করা
$ grep -oE '\b[0-9]{4}-(0[1-9]|1[0-2])-(0[1-9]|[12][0-9]|3[01])\b' server.log | sort -u
2026-11-01

# কোন প্রসেস চলছে — grep নিজেকে যেন না দেখায়
$ ps aux | grep '[n]ginx'
```

শেষের কৌশলটা মজার: `ps aux | grep nginx` চালালে `grep nginx` প্রসেসটাও তালিকায় আসে। কিন্তু `[n]ginx` প্যাটার্নটা "nginx" লেখা মেলায়, অথচ grep-এর নিজের কমান্ড লাইনে লেখা আছে `[n]ginx` — যেটা "nginx" না! তাই grep নিজেকে বাদ দেয়। (অবশ্য সবচেয়ে ভালো উপায় `pgrep -a nginx`।)

### ৪. স্ক্রিপ্টে grep

```bash
#!/usr/bin/env bash
if grep -q "^${USER}:" /etc/passwd; then
    echo "ইউজার $USER লোকাল /etc/passwd-এ আছে"
else
    echo "নেই (হয়তো LDAP/SSSD ইউজার)"
fi

# কনফিগে লাইন না থাকলেই যোগ করো (idempotent)
grep -qxF 'vm.swappiness=10' /etc/sysctl.conf || echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
```

`-qxF` = চুপচাপ, পুরো লাইন, হুবহু — স্ক্রিপ্টের সবচেয়ে কাজের কম্বো। স্ক্রিপ্টিং বিস্তারিত [অধ্যায় ৪১](41-shell-scripting-1.md) থেকে।

---

## ২০.১৪ পারফরম্যান্স টিপস

| টিপ | কেন |
|---|---|
| হুবহু লেখা খুঁজলে `-F` | regex ইঞ্জিন লাগে না |
| `LC_ALL=C grep ...` | UTF-8 মাল্টিবাইট হিসাব বাদ, কখনো অনেক দ্রুত (তবে বাংলা/ইউনিকোড প্যাটার্নে ব্যবহার কোরো না) |
| শুধু অস্তিত্ব দেখতে `-q` বা `-m 1` | প্রথম মিলেই থামে |
| `--exclude-dir`, `--include` দিয়ে জায়গা ছোট করো | কম ফাইল পড়া |
| বড় কোডবেসে `rg` | parallel, `.gitignore` মানে |
| back-reference ও নেস্টেড quantifier এড়াও | ধীর ইঞ্জিনে নিয়ে যায় |
| `cat file \| grep x` না লিখে `grep x file` | অপ্রয়োজনীয় প্রসেস ("useless use of cat") |

---

## ২০.১৫ সম্পর্কিত কীবোর্ড শর্টকাট

grep-এর নিজের কোনো শর্টকাট নেই, কিন্তু grep নিয়ে কাজ করার সময় এগুলো লাগবেই:

| শর্টকাট | কোথায় | কাজ |
|---|---|---|
| `Ctrl + C` | টার্মিনাল | চলমান grep থামাও (যেমন বিশাল ফোল্ডারে `-r`) |
| `Ctrl + D` | টার্মিনাল | ফাইল না দিলে grep stdin-এ অপেক্ষা করে — ইনপুট শেষ জানাতে |
| `Ctrl + R` | bash | আগের grep কমান্ড history থেকে খোঁজো ([অধ্যায় ৬](06-terminal-shortcuts.md)) |
| `Ctrl + Z` | টার্মিনাল | লম্বা grep সাময়িক থামাও (`fg` দিয়ে ফেরত) |
| `Alt + .` | bash | আগের কমান্ডের শেষ argument (যেমন ফাইলের নাম) বসাও |
| `/pattern` | `less`, `man`, `vim` | সামনের দিকে regex খোঁজো |
| `?pattern` | `less`, `man`, `vim` | পেছনের দিকে খোঁজো |
| `n` / `N` | `less`, `vim` | পরের / আগের মিল |
| `&pattern` | `less` | শুধু মেলা লাইনগুলো দেখাও (less-এর ভেতরের grep!) |
| `-i` তারপর Enter | `less` | খোঁজায় case উপেক্ষা চালু/বন্ধ |
| `Esc u` | `less` | হাইলাইট বন্ধ/চালু |
| `Ctrl + Shift + F` | টার্মিনাল এমুলেটর (GNOME Terminal ইত্যাদি) | স্ক্রিনের লেখায় খোঁজো |

> 💡 **টিপস:** grep আটকে আছে মনে হচ্ছে, কিছুই ছাপছে না? সম্ভবত ফাইলের নাম দিতে ভুলে গেছ — grep তখন কীবোর্ড থেকে ইনপুটের অপেক্ষায়। `Ctrl + C` চাপো।

---

## হাতে-কলমে অনুশীলন

### ল্যাব ১: মৌলিক খোঁজা

```
$ cd ~/grep-lab
```

1. `notes.txt`-এ case উপেক্ষা করে `error` আছে এমন লাইন লাইন নম্বরসহ দেখাও।
   ```
   $ grep -ni error notes.txt
   8:error: disk full
   9:Error: permission denied
   ```
2. কতগুলো লাইনে `is` আলাদা শব্দ হিসেবে আছে গোনো → `grep -cw is notes.txt` → `5`
3. ফাঁকা লাইন ছাড়া সব লাইন দেখাও → `grep -v '^$' notes.txt`
4. `kernel` লাইনের আগে-পরে ১ লাইনসহ দেখাও → `grep -C1 kernel notes.txt`

### ল্যাব ২: regex অনুশীলন

```
$ cat > words.txt << 'EOF'
apple
banana
cherry
date
fig
grape
kiwi
level
radar
EOF
```

1. স্বরবর্ণ দিয়ে শুরু শব্দ: `grep -E '^[aeiou]' words.txt` → `apple`
2. ঠিক ৪ অক্ষরের শব্দ: `grep -Ex '.{4}' words.txt` → `date`, `kiwi`
3. একই অক্ষর পরপর দুবার: `grep -E '(.)\1' words.txt` → `apple`, `cherry`
4. `a` দিয়ে শেষ: `grep 'a$' words.txt` → `banana`
5. ৫ অক্ষরের palindrome (উল্টো পড়লেও এক): 
   ```
   $ grep -Ex '(.)(.).\2\1' words.txt
   level
   radar
   ```

### ল্যাব ৩: প্রজেক্টে খোঁজা

1. `project` ফোল্ডারে `TODO` আছে এমন ফাইলের নাম → `grep -rl TODO project`
2. শুধু `.py` ফাইলে, লাইন নম্বরসহ → `grep -rn --include='*.py' TODO project`
3. `notes` ফোল্ডার বাদ দিয়ে সব ফাইলে `milk` → `grep -r --exclude-dir=notes milk project` (কিছুই আসবে না)
4. rg ইনস্টল থাকলে একই কাজ: `rg -t py TODO project`

### ল্যাব ৪: ডেটা বের করা

1. `server.log` থেকে সব IP, ইউনিক → `grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' server.log | sort -u`
2. `404` স্ট্যাটাসের লাইন → `grep ' 404$' server.log`
3. `contacts.txt` থেকে শুধু gmail ঠিকানা → `grep -oE '[A-Za-z0-9._%+-]+@gmail\.com' contacts.txt` → `fatema_99@gmail.com`
4. PCRE দিয়ে `user=`-এর পরের মান: `echo "id=7 user=karim" | grep -oP 'user=\K\w+'` → `karim`

### ল্যাব ৫ (চ্যালেঞ্জ): নিজের সিস্টেমে

1. `/etc/passwd`-এ কতজন ইউজারের শেল `nologin` → `grep -c 'nologin$' /etc/passwd`
2. `/etc` ফোল্ডারে (সাবফোল্ডারসহ) কোন কোন ফাইলে `PermitRootLogin` আছে → `sudo grep -rl PermitRootLogin /etc 2>/dev/null`
3. আজকের লগে `error` শব্দ কতবার (Ubuntu) → `sudo grep -oi error /var/log/syslog | wc -l` (সিস্টেমে `/var/log/syslog` না থাকলে `journalctl -b | grep -ci error`)

---

## সারসংক্ষেপ

### grep দ্রুত রেফারেন্স

| কমান্ড | কাজ |
|---|---|
| `grep pat file` | pat আছে এমন লাইন |
| `grep -i pat file` | case উপেক্ষা |
| `grep -v pat file` | pat নেই এমন লাইন |
| `grep -n pat file` | লাইন নম্বরসহ |
| `grep -c pat file` | মেলা লাইনের সংখ্যা |
| `grep -w pat file` | পুরো শব্দ |
| `grep -x pat file` | পুরো লাইন |
| `grep -o pat file` | শুধু মেলা অংশ |
| `grep -l / -L pat files` | মিল আছে / নেই এমন ফাইলের নাম |
| `grep -rn pat dir` | ফোল্ডারে recursive, লাইন নম্বরসহ |
| `grep -R pat dir` | recursive, সব symlink অনুসরণ |
| `grep -A/-B/-C N pat file` | পরের/আগের/দুই দিকের N লাইন |
| `grep -E 'colou?r'` | ERE — `+ ? ( ) { }` ও অথবা-চিহ্ন সরাসরি |
| `grep -F 'a.b'` | হুবহু লেখা |
| `grep -P '\d+'` | PCRE |
| `grep -e p1 -e p2` / `-f file` | একাধিক প্যাটার্ন |
| `grep -q pat file` | চুপচাপ, শুধু exit status |
| `grep -m N` | N মিলের পর থামো |
| `--include / --exclude / --exclude-dir` | ফাইল/ফোল্ডার বাছাই |
| `--color=auto` | মিল রঙিন |
| `rg pat` | ripgrep — recursive, .gitignore মানে |

### Regex দ্রুত রেফারেন্স

| চিহ্ন | মানে |
|---|---|
| `.` | যেকোনো একটা অক্ষর |
| `*` / `+` / `?` | ০+ / ১+ / ০ বা ১ বার |
| `{n}` `{n,}` `{n,m}` | নির্দিষ্ট সংখ্যক বার |
| `^` / `$` | লাইনের শুরু / শেষ |
| `[abc]` / `[^abc]` / `[a-z]` | দল / বাদে / রেঞ্জ |
| `( )` | গ্রুপ |
| `\b` `\<` `\>` | শব্দের সীমানা |
| `\w` `\s` | word char / whitespace (GNU) |
| `\d` | অঙ্ক (শুধু `-P`) |
| `\1` | back-reference |
| `*?` `+?` | lazy (শুধু `-P`) |
| `(?=..)` `(?<=..)` `\K` | lookahead / lookbehind / keep-out (শুধু `-P`) |
| `[[:digit:]]` `[[:alpha:]]` `[[:space:]]` ... | POSIX class |

অথবা (alternation): ERE ও PCRE-তে `a|b`, BRE-তে `a\|b`।

---

## নিজেকে যাচাই করো

1. `grep` নামটা কোথা থেকে এসেছে?
2. `grep -c error app.log` আর `grep -o error app.log | wc -l` — এ দুটোর ফল কখন আলাদা হবে?
3. `grep 'is' notes.txt` লিখলে `disk` শব্দওয়ালা লাইন কেন আসে? কীভাবে ঠেকাবে?
4. `grep -r` আর `grep -R`-এর পার্থক্য কী?
5. BRE-তে "এক বা একাধিক" কীভাবে লেখো, আর ERE-তে কীভাবে?
6. `grep '\d+' file` কেন আশানুরূপ কাজ করে না? দুইটা সঠিক বিকল্প লেখো।
7. `echo '"a" and "b"' | grep -o '".*"'` কী ছাপবে, এবং শুধু `"a"` আর `"b"` আলাদা পেতে কী করবে?
8. grep-এর exit status `0`, `1`, `2` মানে কী? স্ক্রিপ্টে কোন অপশন দিয়ে শুধু status ব্যবহার করবে?
9. `192.168.1.1` হুবহু খুঁজতে `grep 192.168.1.1` লেখা কেন ঝুঁকিপূর্ণ?
10. ripgrep ডিফল্টভাবে কোন কোন ফাইল বাদ দেয়, আর rg-এর `-r` অপশনের মানে কী?

<details>
<summary>উত্তর দেখো</summary>

1. পুরোনো লাইন-এডিটর `ed`-এর কমান্ড `g/re/p` থেকে — **g**lobal ভাবে একটা **r**egular **e**xpression খুঁজে মেলা লাইন **p**rint করো।
2. যখন একই লাইনে `error` একাধিকবার থাকে। `-c` মেলা **লাইন** গোনে, আর `-o ... | wc -l` প্রতিটা **মিল** আলাদা গোনে।
3. কারণ grep শব্দের ভেতরের অংশও মেলায় — d**is**k-এ `is` আছে। ঠেকাতে `grep -w 'is'` বা `grep '\bis\b'`।
4. দুটোই recursive। `-r` ফোল্ডারের ভেতরে পাওয়া symlink অনুসরণ করে না (শুধু কমান্ড লাইনে দেওয়াগুলো করে); `-R` সব symlink অনুসরণ করে।
5. BRE-তে `\+` (GNU এক্সটেনশন) বা পোর্টেবলভাবে `aa*`; ERE-তে সরাসরি `+`।
6. `\d` POSIX BRE/ERE-র অংশ না, আর BRE-তে `+` সাধারণ অক্ষর। বিকল্প: `grep -E '[0-9]+' file` অথবা `grep -P '\d+' file` (বা `grep -E '[[:digit:]]+'`)।
7. greedy মিলের কারণে পুরো `"a" and "b"` ছাপবে। আলাদা পেতে `grep -o '"[^"]*"'` অথবা `grep -oP '".*?"'`।
8. `0` = মিল পাওয়া গেছে, `1` = মিল নেই, `2` = error। `-q` দিয়ে।
9. কারণ `.` মানে যেকোনো অক্ষর — `192x168y1z1` বা `1921681171`-এর মতো লেখাও মিলে যেতে পারে। নিরাপদ: `grep -F '192.168.1.1'` বা ডট escape করে `'192\.168\.1\.1'` (আর পুরো IP হিসেবে চাইলে `-w`)।
10. `.gitignore`/`.ignore`-এ থাকা ফাইল, লুকানো (`.` দিয়ে শুরু) ফাইল ও ফোল্ডার, আর binary ফাইল। rg-এর `-r` মানে **replace** — আউটপুটে মিলের জায়গায় অন্য লেখা বসিয়ে দেখানো (ফাইল বদলায় না); recursive না, কারণ rg এমনিতেই recursive।

</details>

---

[← আগের অধ্যায়](19-redirection-pipe.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](21-sed.md)

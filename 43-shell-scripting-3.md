# অধ্যায় ৪৩: শেল স্ক্রিপ্টিং — পর্ব ৩ (অ্যাডভান্সড ও বাস্তব প্রজেক্ট)

আগের দুই পর্বে ([অধ্যায় ৪১](41-shell-scripting-1.md) ও [অধ্যায় ৪২](42-shell-scripting-2.md)) তুমি variable, `if`, loop, function, array শিখেছ। এখন তুমি স্ক্রিপ্ট লিখতে পারো। কিন্তু "লিখতে পারা" আর "এমন স্ক্রিপ্ট লেখা যেটা রাত ৩টায় cron থেকে চলবে, ভুল হলে নিজেই থামবে, লগ রাখবে, দুইবার একসাথে চলবে না, আর পরের মানুষ পড়ে বুঝবে" — এ দুটো আলাদা জিনিস। এই অধ্যায় দ্বিতীয়টা শেখায়। এটাই একজন শখের স্ক্রিপ্ট-লেখক আর একজন প্রফেশনাল sysadmin/DevOps engineer-এর পার্থক্য।

## এই অধ্যায়ে যা শিখবে

- `set -euo pipefail` — "strict mode" কী, প্রতিটি অংশ ভেতরে কীভাবে কাজ করে, আর এর ফাঁদগুলো কোথায়
- `trap` দিয়ে signal ধরা, cleanup করা, `EXIT` ও `ERR` trap
- ডিবাগিং: `bash -n`, `bash -x`, `set -x`, `PS4`, `BASH_XTRACEFD`
- `shellcheck` — স্ক্রিপ্টের "বানান-ব্যাকরণ পরীক্ষক"
- `getopts` দিয়ে প্রফেশনাল কমান্ড-লাইন অপশন পার্স করা
- lock ফাইল (`flock`, `mkdir` কৌশল) — একই স্ক্রিপ্ট দুইবার একসাথে না চলা
- লগিং ফাংশন, `logger` দিয়ে syslog/journald-এ লেখা
- রঙিন আউটপুট (ANSI escape code ও `tput`)
- exit code-এর কনভেনশন
- নিরাপদ temp ফাইল (`mktemp`), `readonly`, `local`, `main "$@"` প্যাটার্ন
- ভালো অভ্যাসের চেকলিস্ট ও একটি পুনর্ব্যবহারযোগ্য স্ক্রিপ্ট টেমপ্লেট
- ৫টি সম্পূর্ণ বাস্তব প্রজেক্ট: ব্যাকআপ, ডিস্ক অ্যালার্ট, লগ বিশ্লেষক, bulk ইউজার তৈরি, সার্ভিস হেলথ চেক — লাইন-বাই-লাইন ব্যাখ্যাসহ

---

## ৪৩.১ কেন "অ্যাডভান্সড" স্ক্রিপ্টিং দরকার?

ধরো তুমি একটা ছোট স্ক্রিপ্ট লিখলে:

```bash
#!/bin/bash
cd /var/www/old_site
rm -rf *
```

উদ্দেশ্য: পুরোনো সাইটের ফাইল মুছে ফেলা। কিন্তু যদি `/var/www/old_site` ফোল্ডারটা না থাকে? `cd` ব্যর্থ হবে, একটা error দেখাবে — কিন্তু bash **থামবে না**। পরের লাইন `rm -rf *` চলবে **বর্তমান ডিরেক্টরিতে** — হয়তো সেটা `/` বা তোমার home! এরকম ঘটনায় বাস্তবে কোম্পানির পুরো সার্ভার মুছে গেছে।

> 📝 **মনে রাখো:** bash-এর ডিফল্ট আচরণ হলো "কোনো কমান্ড ফেল করলেও চুপচাপ পরের লাইনে চলে যাও।" ইন্টারঅ্যাকটিভ টার্মিনালে এটা সুবিধাজনক, কিন্তু স্ক্রিপ্টে এটা বিপজ্জনক।

উপমা: একজন রাঁধুনির কথা ভাবো যে রেসিপি অনুসরণ করছে। "ডিম ফাটাও" ধাপে দেখল ডিম নেই — কিন্তু সে থামল না, পরের ধাপে "ফাটানো ডিম কড়াইয়ে দাও" করতে গিয়ে খালি বাটি কড়াইয়ে ঢেলে দিল। অ্যাডভান্সড স্ক্রিপ্টিং মানে রাঁধুনিকে শেখানো: "কোনো ধাপ ব্যর্থ হলে থামো, রান্নাঘর গুছিয়ে রাখো (cleanup), আর খাতায় লিখে রাখো কী হয়েছিল (logging)।"

---

## ৪৩.২ Strict Mode: `set -euo pipefail`

প্রায় সব প্রফেশনাল bash স্ক্রিপ্টের শুরুতে তুমি এই লাইনটা দেখবে:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

এটা আসলে তিনটা আলাদা সেটিং একসাথে: `-e`, `-u`, আর `-o pipefail`। একটা একটা করে দেখি।

### ৪৩.২.১ `set -e` (errexit) — ফেল করলেই থামো

`set -e` চালু থাকলে, কোনো "simple command" যদি non-zero exit status ফেরত দেয়, bash তৎক্ষণাৎ স্ক্রিপ্ট বন্ধ করে দেয়।

```bash
#!/bin/bash
set -e
echo "শুরু"
ls /nai_emon_folder
echo "এটা কখনো প্রিন্ট হবে না"
```

```
$ ./test.sh
শুরু
ls: cannot access '/nai_emon_folder': No such file or directory
$ echo $?
2
```

স্ক্রিপ্টটা `ls`-এর exit code (2) নিয়েই বের হয়ে গেল।

**ভেতরে কীভাবে কাজ করে:** প্রতিটি কমান্ড শেষ হলে bash তার exit status `$?`-তে রাখে। `errexit` অপশন চালু থাকলে bash প্রতিবার চেক করে — "এই কমান্ডটা কি এমন প্রসঙ্গে চলেছে যেখানে ফেল করা প্রত্যাশিত?" যদি না হয়, আর status ≠ 0, তাহলে `exit` করে।

**কোন ক্ষেত্রে `set -e` কাজ করে না (খুব গুরুত্বপূর্ণ!):**

| প্রসঙ্গ | উদাহরণ | কেন থামে না |
|---|---|---|
| `if`/`while`/`until`-এর শর্তে | `if grep -q x file; then` | এখানে ফেল করাটা "প্রশ্নের উত্তর", error না |
| `&&` বা `\|\|` তালিকায় (শেষ কমান্ড বাদে) | `false && echo hi` | bash ধরে নেয় তুমি নিজে ফেল সামলাচ্ছ |
| `!` দিয়ে উল্টানো কমান্ড | `! grep x file` | |
| pipeline-এর শেষটা বাদে বাকিগুলো | `false \| true` | pipeline-এর status = শেষ কমান্ডের (যদি না `pipefail` থাকে) |
| ওপরের কোনো প্রসঙ্গে ডাকা function-এর ভেতরে | `if myfunc; then` | পুরো function-এর ভেতরে `-e` নিষ্ক্রিয় হয়ে যায় |

> ⚠️ **সতর্কতা — `((count++))` ফাঁদ:** `count=0` থাকলে `((count++))` এর মান হয় 0 (post-increment পুরোনো মান দেয়), আর `(( ))` এর ভেতরের মান 0 হলে exit status হয় 1। ফলে `set -e` স্ক্রিপ্টটাকে চুপচাপ মেরে ফেলবে! নিরাপদ বিকল্প: `count=$((count + 1))` বা `((count += 1))` বা `((++count))`। (শেষ দুটোও মান 0 হলে ফেল করবে, তাই `count=$((count+1))` সবচেয়ে নিরাপদ।)

> ⚠️ **সতর্কতা — `local var=$(cmd)` ফাঁদ:** `local x=$(false)` এর exit status `local`-এর (যেটা সফল = 0), `false`-এর না। তাই `set -e` ধরবে না। সমাধান: আলাদা লাইনে লেখো:
> ```bash
> local x
> x=$(some_command)
> ```
> (shellcheck এটাকে SC2155 হিসেবে ধরিয়ে দেয়।)

যে কমান্ড ফেল করতে পারে এবং তুমি সেটা মেনে নিতে চাও, সেখানে স্পষ্টভাবে বলো:

```bash
rm -f /tmp/maybe_exists          # -f নিজেই ফেল না করার ব্যবস্থা
grep -q "pattern" file || true   # ফেল হলেও চলবে
if ! ping -c1 -W2 host >/dev/null; then
    echo "host down"
fi
```

### ৪৩.২.২ `set -u` (nounset) — অচেনা variable মানেই ভুল

```bash
#!/bin/bash
set -u
TARGET_DIR="/tmp/data"
rm -rf "$TARGTE_DIR/"*      # বানান ভুল!
```

`set -u` ছাড়া `$TARGTE_DIR` হয়ে যেত খালি স্ট্রিং, আর কমান্ডটা হতো `rm -rf /*` — সর্বনাশ! `set -u` থাকলে:

```
$ ./test.sh
./test.sh: line 4: TARGTE_DIR: unbound variable
```

ঐচ্ছিক variable-এর জন্য default দাও:

```bash
name="${1:-guest}"           # $1 না থাকলে "guest"
: "${CONFIG_FILE:=/etc/app.conf}"   # না থাকলে সেট করে দাও
if [[ -n "${DEBUG:-}" ]]; then ...   # সেট আছে কি না নিরাপদে চেক
```

> 💡 **টিপস:** bash 4.4-এর আগে `set -u` থাকলে খালি array-এ `"${arr[@]}"` লিখলে error দিত। Ubuntu 24.04 (bash 5.2), Debian 12 (bash 5.2), RHEL 9 (bash 5.1)-এ এই সমস্যা নেই।

### ৪৩.২.৩ `set -o pipefail` — pipeline-এর লুকানো ব্যর্থতা ধরো

ডিফল্টে একটা pipeline-এর exit status হলো **শুধু শেষ কমান্ডের** status:

```
$ cat /nai_file | sort
cat: /nai_file: No such file or directory
$ echo $?
0
```

`sort` সফল হয়েছে (খালি ইনপুট সাজাতে পেরেছে), তাই পুরো pipeline "সফল"! `pipefail` চালু করলে pipeline-এর status হয় **ডান দিক থেকে প্রথম non-zero status** (সব সফল হলে 0):

```
$ set -o pipefail
$ cat /nai_file | sort
cat: /nai_file: No such file or directory
$ echo $?
1
```

কোন অংশ ফেল করেছে জানতে `PIPESTATUS` array দেখো:

```
$ false | true | grep -q x </dev/null; echo "${PIPESTATUS[@]}"
1 0 1
```

> ⚠️ **সতর্কতা — SIGPIPE ফাঁদ:** `some_cmd | head -n1` — `head` প্রথম লাইন পড়েই বন্ধ হয়ে যায়, তখন `some_cmd` লিখতে গিয়ে SIGPIPE পায় এবং 141 (128+13) status দিয়ে মরে। `pipefail` থাকলে পুরো pipeline "ফেল"! যেমন `yes | head -n1` pipefail-এ 141 দেয়। এমন জায়গায় `|| true` দাও বা ডিজাইন বদলাও।

### ৪৩.২.৪ ভেতরের চিত্র

```
         set -euo pipefail
               │
   ┌───────────┼─────────────────┐
   ▼           ▼                 ▼
 -e (errexit)  -u (nounset)   -o pipefail
 কমান্ড ফেল   অচেনা $var     pipeline-এর
 = স্ক্রিপ্ট   = error +      যেকোনো অংশ ফেল
 থামাও        থামাও          = পুরো pipeline ফেল
   │                             │
   └────────── একসাথে ───────────┘
                │
   "চুপচাপ ভুল" → "জোরে ব্যর্থতা" (fail fast, fail loud)
```

বর্তমান অপশন দেখতে:

```
$ echo $-
himBHs
$ set -o | grep -E 'errexit|nounset|pipefail'
errexit         off
nounset         off
pipefail        off
```

`$-` এ `e` বা `u` অক্ষর থাকলে সেগুলো চালু। কোনো অপশন বন্ধ করতে `+` দাও: `set +e` (সাময়িকভাবে errexit বন্ধ), পরে আবার `set -e`।

> 📝 **মনে রাখো:** strict mode কোনো জাদু না — এটা একটা "সেফটি নেট"। এর ফাঁকগুলো (if-এর ভেতর function, `local x=$(..)`, `((i++))`) জানা থাকলেই এটা সত্যিকার কাজে আসে।

আরেকটা কাজের অপশন হলো `IFS`। অনেক টেমপ্লেটে দেখবে:

```bash
IFS=$'\n\t'
```

এতে word splitting শুধু newline আর tab-এ হয়, space-এ না — ফাইলনামে space থাকলে কম বিপদ। তবে সবসময় variable quote করা (`"$var"`) এর চেয়েও বড় সুরক্ষা।

---

## ৪৩.৩ `trap` — signal ধরা ও cleanup

### ৪৩.৩.১ Signal কী?

Signal হলো kernel বা অন্য process থেকে একটা process-কে পাঠানো ছোট্ট "নোটিশ"। ([অধ্যায় ২৪](24-process-management.md)-এ বিস্তারিত আছে।) স্ক্রিপ্টের জন্য গুরুত্বপূর্ণগুলো:

| Signal | নম্বর | কখন আসে | trap করা যায়? |
|---|---|---|---|
| `SIGHUP` | 1 | টার্মিনাল বন্ধ হলে | হ্যাঁ |
| `SIGINT` | 2 | `Ctrl+C` চাপলে | হ্যাঁ |
| `SIGQUIT` | 3 | `Ctrl+\` চাপলে | হ্যাঁ |
| `SIGKILL` | 9 | `kill -9` | **না** |
| `SIGTERM` | 15 | `kill` (ডিফল্ট), `systemctl stop` | হ্যাঁ |
| `SIGSTOP` | 19 | `kill -STOP` | **না** |
| `SIGTSTP` | 20 | `Ctrl+Z` | হ্যাঁ |

এগুলো ছাড়াও bash-এর নিজস্ব কিছু "pseudo-signal" আছে:

| Pseudo-signal | কখন চলে |
|---|---|
| `EXIT` | স্ক্রিপ্ট যেভাবেই শেষ হোক (স্বাভাবিক, `exit`, `set -e`, বা trap করা signal-এ) |
| `ERR` | কোনো কমান্ড non-zero দিলে (`set -e` যে নিয়মে থামে, প্রায় সেই নিয়মেই) |
| `DEBUG` | প্রতিটি কমান্ডের **আগে** |
| `RETURN` | function বা `source` করা ফাইল শেষ হলে |

### ৪৩.৩.২ trap-এর syntax

```bash
trap 'কমান্ড' SIGNAL [SIGNAL ...]
trap - SIGNAL        # ডিফল্ট আচরণে ফেরাও
trap '' SIGNAL       # signal উপেক্ষা করো (ignore)
trap -p              # বর্তমান trap গুলো দেখাও
trap -l              # সব signal-এর নাম ও নম্বর
```

উপমা: trap হলো বাড়ির "ইমার্জেন্সি প্ল্যান"। আগুন লাগলে (signal) কী করবে — গ্যাস বন্ধ, দরজা খোলা — তা আগে থেকে লিখে রাখা।

### ৪৩.৩.৩ সবচেয়ে কাজের প্যাটার্ন: EXIT trap দিয়ে cleanup

```bash
#!/usr/bin/env bash
set -euo pipefail

tmpdir=$(mktemp -d)
cleanup() {
    rm -rf -- "$tmpdir"
    echo "temp ফোল্ডার মুছে ফেলা হয়েছে"
}
trap cleanup EXIT

echo "কাজ চলছে $tmpdir এ"
cp /etc/hosts "$tmpdir/"
false            # ইচ্ছাকৃত ব্যর্থতা — set -e স্ক্রিপ্ট থামাবে
echo "এটা আসবে না"
```

```
$ ./demo.sh
কাজ চলছে /tmp/tmp.k3Jd8sQw1x এ
temp ফোল্ডার মুছে ফেলা হয়েছে
```

স্ক্রিপ্ট ফেল করলেও cleanup চলেছে। এটাই EXIT trap-এর শক্তি।

### ৪৩.৩.৪ exit code রক্ষা করা ও signal-এ ঠিকমতো বের হওয়া

```bash
cleanup() {
    local rc=$?          # প্রথম লাইনেই exit code ধরে রাখো!
    rm -rf -- "$tmpdir"
    exit "$rc"
}
trap cleanup EXIT
trap 'exit 130' INT     # Ctrl+C → 130 (128+2)
trap 'exit 143' TERM    # kill → 143 (128+15)
```

INT/TERM trap-এ `exit` ডাকলে EXIT trap-ও চলবে, তাই cleanup একবারই লিখতে হয়।

> 💡 **টিপস:** bash-এ শুধু `trap cleanup EXIT` দিলেই সাধারণত `set -e` ব্যর্থতা ও স্বাভাবিক শেষ — দুটোতেই cleanup হয়। কিন্তু SIGINT/SIGTERM-এ bash কিছু ক্ষেত্রে EXIT trap চালায় না (অন্য shell-এ তো নয়ই), তাই INT/TERM আলাদাভাবে ধরে `exit` করা সবচেয়ে নির্ভরযোগ্য।

### ৪৩.৩.৫ ERR trap — কোথায় ফেল করল জানো

```bash
#!/usr/bin/env bash
set -eEuo pipefail     # -E (errtrace) — function-এর ভেতরেও ERR trap উত্তরাধিকার পাবে

on_error() {
    local rc=$? line=$1
    echo "ERROR: লাইন $line এ কমান্ড ফেল করেছে: '$BASH_COMMAND' (exit $rc)" >&2
}
trap 'on_error $LINENO' ERR

load_data() {
    cat /nai/file
}
load_data
```

```
$ ./err.sh
cat: /nai/file: No such file or directory
ERROR: লাইন 9 এ কমান্ড ফেল করেছে: 'cat /nai/file' (exit 1)
```

- `$LINENO` — বর্তমান লাইন নম্বর
- `$BASH_COMMAND` — এই মুহূর্তে চলা কমান্ডের লেখা
- `-E` (`set -o errtrace`) না দিলে function, command substitution ও subshell-এর ভেতরে ERR trap কাজ করে না।

> ⚠️ **সতর্কতা:** `SIGKILL` (`kill -9`) আর `SIGSTOP` কখনো trap করা যায় না। kernel সরাসরি process-কে মেরে ফেলে — কোনো cleanup হয় না। তাই গুরুত্বপূর্ণ জিনিস (যেমন lock) এমনভাবে ডিজাইন করো যেন process মরলে নিজে থেকেই মুক্ত হয় (দেখো ৪৩.৬, `flock`)।

### ৪৩.৩.৬ trap দিয়ে Ctrl+C উপেক্ষা

```bash
trap '' INT          # এখন Ctrl+C কাজ করবে না
critical_update
trap - INT           # আবার স্বাভাবিক
```

যেমন ডাটাবেস migration-এর মাঝখানে কেউ ভুলে Ctrl+C চাপলে যেন অর্ধেক কাজ না হয়।

---

## ৪৩.৪ ডিবাগিং — স্ক্রিপ্টের ভেতরে উঁকি দেওয়া

### ৪৩.৪.১ ডিবাগিং টুলের তালিকা

| কমান্ড/অপশন | কী করে |
|---|---|
| `bash -n script.sh` | শুধু syntax পরীক্ষা, চালায় না (noexec) |
| `bash -v script.sh` | প্রতিটি লাইন পড়ার সাথে সাথে হুবহু প্রিন্ট করে (verbose) |
| `bash -x script.sh` | প্রতিটি কমান্ড expand হওয়ার পর `+` দিয়ে প্রিন্ট করে (xtrace) |
| `set -x` / `set +x` | স্ক্রিপ্টের নির্দিষ্ট অংশে trace চালু/বন্ধ |
| `PS4` | trace লাইনের শুরুর prefix (ডিফল্ট `+ `) |
| `BASH_XTRACEFD` | trace আউটপুট কোন file descriptor-এ যাবে |
| `trap '...' DEBUG` | প্রতিটি কমান্ডের আগে কিছু চালাও |
| `caller` | function কে কোথা থেকে ডেকেছে |

### ৪৩.৪.২ `bash -n` — চালানোর আগেই syntax ভুল ধরো

```
$ cat bad.sh
#!/bin/bash
if [ -f /etc/hosts ]; then
    echo "আছে"
$ bash -n bad.sh
bad.sh: line 4: syntax error: unexpected end of file
```

`fi` ভুলে গেছ — চালানোর আগেই ধরা পড়ল।

### ৪৩.৪.৩ `bash -x` — প্রতিটি ধাপ দেখো

```bash
#!/bin/bash
name="world"
for i in 1 2; do
    echo "hello $name $i"
done
```

```
$ bash -x hello.sh
+ name=world
+ for i in 1 2
+ echo 'hello world 1'
hello world 1
+ for i in 1 2
+ echo 'hello world 2'
hello world 2
```

`+` দিয়ে শুরু লাইনগুলো trace — variable expand হওয়ার **পরের** আসল কমান্ড। এখানেই বোঝা যায় `$name` আসলে কী মান পেয়েছিল। nested subshell-এ `++`, `+++` দেখবে (গভীরতা অনুযায়ী `PS4`-এর প্রথম অক্ষর পুনরাবৃত্তি হয়)।

### ৪৩.৪.৪ নির্দিষ্ট অংশে trace

```bash
echo "এই অংশ স্বাভাবিক"
set -x
result=$(compute_something "$input")
set +x
echo "আবার স্বাভাবিক"
```

### ৪৩.৪.৫ `PS4` দিয়ে তথ্যবহুল trace

ডিফল্ট `+ ` থেকে কিছু বোঝা যায় না কোন ফাইলের কোন লাইন। `PS4` বদলাও:

```bash
export PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]:-main}: '
```

```
$ bash -x deploy.sh
+ deploy.sh:5:main: target=/srv/app
+ deploy.sh:12:copy_files: cp -r build/. /srv/app
```

> ⚠️ **সতর্কতা:** `PS4` সিঙ্গেল কোটে লেখো, যাতে `$LINENO` ইত্যাদি প্রতিবার trace-এর সময় expand হয়, সেট করার সময় একবার না।

আরও সময়সহ:

```bash
export PS4='+ $(date +%T.%N | cut -c1-12) ${LINENO}: '
```

(প্রতিটি লাইনে `date` চলবে, তাই ধীর হবে — শুধু পারফরম্যান্স খুঁজতে ব্যবহার করো। bash 5-এ `$EPOCHREALTIME` আরও দ্রুত বিকল্প।)

### ৪৩.৪.৬ trace আলাদা ফাইলে পাঠানো

```bash
exec 5> /tmp/debug.log
BASH_XTRACEFD=5
set -x
```

এখন trace যাবে `/tmp/debug.log`-এ, আর স্ক্রিপ্টের সাধারণ stderr পরিষ্কার থাকবে।

### ৪৩.৪.৭ DEBUG mode সুইচ

প্রফেশনাল স্ক্রিপ্টে প্রায়ই এভাবে থাকে:

```bash
[[ "${DEBUG:-0}" == "1" ]] && set -x
```

```
$ DEBUG=1 ./myscript.sh
```

> ⚠️ **সতর্কতা:** `set -x` পাসওয়ার্ড, token সহ সব কিছু প্রিন্ট করে! গোপন তথ্য নিয়ে কাজের সময় `set +x` করে নাও, আর trace লগ কখনো শেয়ার করার আগে পরিষ্কার করো।

---

## ৪৩.৫ ShellCheck — স্ক্রিপ্টের "ব্যাকরণ শিক্ষক"

`shellcheck` হলো একটা static analysis টুল — স্ক্রিপ্ট না চালিয়েই পড়ে বলে দেয় কোথায় bug আছে বা হতে পারে। উপমা: Word-এর বানান-চেকারের মতো, কিন্তু শুধু বানান না — ব্যাকরণ আর অর্থের ভুলও ধরে।

### ইনস্টল

```bash
sudo apt install shellcheck          # Ubuntu / Debian
sudo dnf install ShellCheck          # Fedora (RHEL 9-এ EPEL repo লাগবে)
```

### ব্যবহার

```
$ cat example.sh
#!/bin/bash
files=$(ls *.txt)
for f in $files; do
    cp $f /backup/
done
cd /some/dir
echo "done"
```

```
$ shellcheck example.sh

In example.sh line 2:
files=$(ls *.txt)
        ^-------^ SC2012 (info): Use find instead of ls to better handle non-alphanumeric filenames.

In example.sh line 4:
    cp $f /backup/
       ^-- SC2086 (info): Double quote to prevent globbing and word splitting.

Did you mean:
    cp "$f" /backup/

In example.sh line 6:
cd /some/dir
^----------^ SC2164 (warning): Use 'cd ... || exit' or 'cd ... || return' in case cd fails.
```

(সঠিক আউটপুটের ফরম্যাট সংস্করণভেদে সামান্য আলাদা হতে পারে।)

### সবচেয়ে পরিচিত সতর্কতা

| কোড | মানে | সমাধান |
|---|---|---|
| SC2086 | variable quote করা হয়নি | `"$var"` |
| SC2164 | `cd` ফেল করলে কী হবে? | `cd dir \|\| exit 1` |
| SC2155 | `local x=$(cmd)` exit code লুকায় | আলাদা লাইনে declare ও assign |
| SC2034 | variable তৈরি হয়েছে কিন্তু ব্যবহার হয়নি | মুছে ফেলো বা বানান ঠিক করো |
| SC2046 | `$(...)` quote করা হয়নি | `"$(...)"` |
| SC2006 | পুরোনো backtick `` `cmd` `` | `$(cmd)` |
| SC2002 | অপ্রয়োজনীয় `cat` (useless use of cat) | `grep x file` |
| SC2068 | `$@` quote করা হয়নি | `"$@"` |
| SC1090/SC1091 | `source` করা ফাইল খুঁজে পাচ্ছে না | `# shellcheck source=path` |

### গুরুত্বপূর্ণ অপশন

| অপশন | কাজ |
|---|---|
| `-s bash` / `-s sh` | কোন shell হিসেবে চেক করবে |
| `-e SC2086` | নির্দিষ্ট সতর্কতা বাদ দাও |
| `-S warning` | ন্যূনতম severity (error, warning, info, style) |
| `-x` | `source` করা ফাইলও অনুসরণ করো |
| `-f gcc` / `-f diff` | আউটপুট ফরম্যাট (diff ফরম্যাট দিয়ে auto-fix patch পাওয়া যায়) |

নির্দিষ্ট লাইনে ইচ্ছাকৃতভাবে সতর্কতা বন্ধ করতে, ঠিক ওপরের লাইনে:

```bash
# shellcheck disable=SC2086
echo $intentionally_unquoted
```

> 💡 **টিপস:** VS Code-এ "ShellCheck" extension, আর Vim-এ ALE plugin লাগালে লেখার সময়ই লাল দাগ দেখাবে। CI pipeline-এ `shellcheck *.sh` যোগ করে দাও — খারাপ স্ক্রিপ্ট merge-ই হবে না।

---

## ৪৩.৬ `getopts` দিয়ে কমান্ড-লাইন অপশন পার্স

তুমি `ls -la`, `tar -czf file.tgz` দেখেছ। তোমার স্ক্রিপ্টও এমন অপশন নিতে পারে। bash-এর built-in `getopts` এই কাজ করে।

### ৪৩.৬.১ মূল ধারণা

```bash
while getopts "OPTSTRING" opt; do
    case "$opt" in
        ...
    esac
done
shift $((OPTIND - 1))
```

- **OPTSTRING:** কোন অক্ষরগুলো বৈধ অপশন। কোনো অক্ষরের পরে `:` থাকলে সেই অপশনের একটা মান (argument) লাগবে।
  - `"hvo:"` → `-h`, `-v` (মান ছাড়া), `-o VALUE` (মান সহ)
  - শুরুতে `:` (যেমন `":hvo:"`) → "silent mode": getopts নিজে error প্রিন্ট করবে না, তুমি সামলাবে।
- **`$opt`:** প্রতিবার লুপে পাওয়া অপশনের অক্ষর।
- **`$OPTARG`:** অপশনের মান (যেমন `-o file.txt` হলে `file.txt`)।
- **`$OPTIND`:** পরের যে argument প্রক্রিয়া করা হবে তার index। লুপ শেষে `shift $((OPTIND-1))` করলে অপশনগুলো সরে যায়, বাকি থাকে positional argument।

Silent mode-এ error হলে:

| পরিস্থিতি | `$opt` | `$OPTARG` |
|---|---|---|
| অচেনা অপশন (`-z`) | `?` | `z` |
| মান লাগবে কিন্তু দেওয়া হয়নি (`-o` একা) | `:` | `o` |

### ৪৩.৬.২ ভেতরে কীভাবে কাজ করে

```
 $ ./script.sh -v -o out.txt -n 5 file1 file2
                 │   │   │      │  │    │     │
 index:          1   2   3      4  5    6     7

 getopts call 1 → opt=v            OPTIND=2
 getopts call 2 → opt=o OPTARG=out.txt   OPTIND=4
 getopts call 3 → opt=n OPTARG=5         OPTIND=6
 getopts call 4 → "file1" অপশন না → return 1 (লুপ শেষ)
 shift $((6-1)) → $1=file1  $2=file2
```

getopts `-vn 5` এর মতো একত্রিত অপশন, আর `-n5` (ফাঁক ছাড়া মান) — দুটোই বোঝে। `--` পেলে অপশন পার্স থামিয়ে দেয়।

### ৪৩.৬.৩ সম্পূর্ণ উদাহরণ

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    cat <<EOF
ব্যবহার: ${0##*/} [-h] [-v] [-n COUNT] [-o OUTFILE] FILE...

  -h          এই সাহায্য দেখাও
  -v          বিস্তারিত (verbose) আউটপুট
  -n COUNT    কতগুলো লাইন (ডিফল্ট: 10)
  -o OUTFILE  ফলাফল কোন ফাইলে লেখা হবে
EOF
}

verbose=0
count=10
outfile=""

while getopts ":hvn:o:" opt; do
    case "$opt" in
        h) usage; exit 0 ;;
        v) verbose=1 ;;
        n) count="$OPTARG" ;;
        o) outfile="$OPTARG" ;;
        :) echo "ত্রুটি: -$OPTARG অপশনের জন্য মান দরকার" >&2; usage >&2; exit 2 ;;
        \?) echo "ত্রুটি: অচেনা অপশন -$OPTARG" >&2; usage >&2; exit 2 ;;
    esac
done
shift $((OPTIND - 1))

if [[ $# -eq 0 ]]; then
    echo "ত্রুটি: অন্তত একটা FILE দাও" >&2
    usage >&2
    exit 2
fi

if ! [[ "$count" =~ ^[0-9]+$ ]]; then
    echo "ত্রুটি: COUNT একটা সংখ্যা হতে হবে" >&2
    exit 2
fi

(( verbose )) && echo "count=$count outfile=${outfile:-stdout} files=$*"

for f in "$@"; do
    if [[ -n "$outfile" ]]; then
        head -n "$count" -- "$f" >> "$outfile"
    else
        head -n "$count" -- "$f"
    fi
done
```

```
$ ./myhead.sh -v -n 2 /etc/hostname /etc/os-release
count=2 outfile=stdout files=/etc/hostname /etc/os-release
myserver
PRETTY_NAME="Ubuntu 24.04.1 LTS"
NAME="Ubuntu"
$ ./myhead.sh -x
ত্রুটি: অচেনা অপশন -x
ব্যবহার: myhead.sh [-h] [-v] [-n COUNT] [-o OUTFILE] FILE...
...
$ ./myhead.sh -n
ত্রুটি: -n অপশনের জন্য মান দরকার
```

> 📝 **মনে রাখো:** `getopts` শুধু ছোট (single-letter) অপশন বোঝে, `--verbose` এর মতো লম্বা অপশন বোঝে না। লম্বা অপশন লাগলে হয় `while [[ $# -gt 0 ]]; do case "$1" in ... esac; shift; done` দিয়ে নিজে পার্স করো, নয়তো util-linux-এর বাহ্যিক `getopt` (শেষে `s` ছাড়া) কমান্ড ব্যবহার করো। দুটো নাম কাছাকাছি হলেও আলাদা জিনিস: `getopts` bash built-in, `getopt` আলাদা প্রোগ্রাম।

লম্বা অপশনের হাতে-লেখা পার্সার:

```bash
while [[ $# -gt 0 ]]; do
    case "$1" in
        -h|--help)    usage; exit 0 ;;
        -v|--verbose) verbose=1 ;;
        -n|--count)   count="${2:?--count এর মান দরকার}"; shift ;;
        --count=*)    count="${1#*=}" ;;
        --)           shift; break ;;
        -*)           echo "অচেনা অপশন: $1" >&2; exit 2 ;;
        *)            break ;;
    esac
    shift
done
```

`${2:?message}` — `$2` না থাকলে বা খালি হলে message দিয়ে error করে থামে।

---

## ৪৩.৭ Lock ফাইল — একই স্ক্রিপ্ট দুইবার একসাথে নয়

### কেন দরকার?

ধরো ব্যাকআপ স্ক্রিপ্ট cron থেকে প্রতি ঘণ্টায় চলে। একদিন ডেটা বেশি হওয়ায় ব্যাকআপে ৯০ মিনিট লাগল। এর মধ্যে পরের ঘণ্টার কপিটাও শুরু হয়ে গেল — দুটো একই ফাইলে লিখছে, ডিস্ক IO দ্বিগুণ, ব্যাকআপ নষ্ট। সমাধান: **lock** — বাথরুমের দরজার ছিটকিনির মতো; ভেতরে কেউ থাকলে পরের জন অপেক্ষা করবে বা চলে যাবে।

### ৪৩.৭.১ ভুল পদ্ধতি (race condition)

```bash
# ❌ এটা করো না
if [[ -f /tmp/my.lock ]]; then exit 1; fi
touch /tmp/my.lock
```

"চেক করা" আর "তৈরি করা" — দুটো আলাদা ধাপ। দুটো process ঠিক একই মুহূর্তে চেক করলে দুজনেই দেখবে lock নেই, দুজনেই ঢুকে যাবে। এর নাম **race condition**। তার ওপর, স্ক্রিপ্ট `kill -9` হলে lock ফাইল চিরকাল থেকে যাবে (stale lock)।

### ৪৩.৭.২ সঠিক পদ্ধতি ১: `flock` (সবচেয়ে ভালো)

`flock` (util-linux প্যাকেজের অংশ, সব বড় distro-তে থাকে) kernel-এর advisory file lock ব্যবহার করে। lock ধরে থাকে একটা **খোলা file descriptor** — process মরে গেলে (এমনকি `kill -9`) kernel নিজে থেকে fd বন্ধ করে, ফলে lock মুক্ত হয়ে যায়। stale lock-এর ঝামেলা নেই।

```bash
#!/usr/bin/env bash
set -euo pipefail

LOCKFILE="/run/lock/mybackup.lock"    # বা /var/lock/... (সাধারণত /run/lock-এর symlink)

exec 9>"$LOCKFILE"                     # fd 9 খুলে lock ফাইলের সাথে যুক্ত করো
if ! flock -n 9; then                  # -n = অপেক্ষা না করে সাথে সাথে চেষ্টা
    echo "আরেকটা কপি ইতিমধ্যে চলছে, বের হচ্ছি।" >&2
    exit 1
fi

echo "lock পেয়েছি, কাজ শুরু (PID $$)"
sleep 30
```

```
$ ./lockdemo.sh &
[1] 41234
lock পেয়েছি, কাজ শুরু (PID 41234)
$ ./lockdemo.sh
আরেকটা কপি ইতিমধ্যে চলছে, বের হচ্ছি।
```

```
 Process A                    Kernel                 Process B
 exec 9>lockfile  ───────►  inode-এ fd খোলা
 flock -n 9       ───────►  LOCK দিলাম (A)
                                         ◄─────── flock -n 9
                            ব্যস্ত! fail  ───────►  exit 1
 (A মারা গেল / শেষ)
 kernel fd বন্ধ করে ─────►  LOCK মুক্ত
```

`flock`-এর গুরুত্বপূর্ণ অপশন:

| অপশন | কাজ |
|---|---|
| `-n`, `--nonblock` | lock না পেলে অপেক্ষা না করে সাথে সাথে ফেল (exit 1) |
| `-w SEC`, `--timeout SEC` | সর্বোচ্চ এত সেকেন্ড অপেক্ষা |
| `-x`, `--exclusive` | exclusive lock (ডিফল্ট) |
| `-s`, `--shared` | shared lock (অনেক reader একসাথে) |
| `-u`, `--unlock` | lock ছেড়ে দাও |
| `-E N`, `--conflict-exit-code N` | `-n`/`-w` ফেল হলে কোন exit code |

cron-এ এক লাইনেই ব্যবহার:

```
0 * * * * /usr/bin/flock -n /run/lock/backup.lock /usr/local/bin/backup.sh
```

এখানে `flock` নিজে lock নেয়, তারপর কমান্ডটা চালায়।

### ৪৩.৭.৩ সঠিক পদ্ধতি ২: `mkdir` (atomic)

`mkdir` একটাই atomic অপারেশন — ডিরেক্টরি হয় তৈরি হয় (সফল), নয়তো আগে থেকে থাকলে ফেল। চেক আর তৈরি একসাথে হয়, তাই race নেই।

```bash
LOCKDIR="/tmp/myscript.lock"
if mkdir "$LOCKDIR" 2>/dev/null; then
    trap 'rmdir "$LOCKDIR"' EXIT
    echo "$$" > "$LOCKDIR/pid"        # (rmdir-এর আগে এটাও মুছতে হবে: rm -rf)
else
    echo "চলছে (PID $(cat "$LOCKDIR/pid" 2>/dev/null))" >&2
    exit 1
fi
```

অসুবিধা: `kill -9` হলে lock থেকে যায়। তাই PID সংরক্ষণ করে `kill -0 PID` দিয়ে সেটা বেঁচে আছে কি না পরীক্ষা করার কোড লাগে। এজন্যই `flock` পছন্দনীয়।

> 💡 **টিপস:** systemd service হিসেবে চালালে systemd নিজেই একটা unit-এর একাধিক instance একসাথে চালায় না — আলাদা lock নাও লাগতে পারে। দেখো [অধ্যায় ২৬](26-systemd-services.md) ও [অধ্যায় ২৯](29-scheduling-cron.md)।

---

## ৪৩.৮ লগিং ফাংশন

`echo` দিয়ে এলোমেলো মেসেজের চেয়ে একটা নিয়মিত লগ ফরম্যাট অনেক কাজের: সময়, লেভেল, মেসেজ।

```bash
LOG_FILE="${LOG_FILE:-/var/log/myscript.log}"

log() {
    local level="$1"; shift
    local ts
    ts=$(date '+%Y-%m-%d %H:%M:%S')
    printf '%s [%s] %s\n' "$ts" "$level" "$*" | tee -a "$LOG_FILE" >&2
}
info()  { log INFO  "$@"; }
warn()  { log WARN  "$@"; }
error() { log ERROR "$@"; }
die()   { error "$@"; exit 1; }
```

```
$ info "ব্যাকআপ শুরু"
2026-11-12 10:15:02 [INFO] ব্যাকআপ শুরু
$ die "ডিস্কে জায়গা নেই"
2026-11-12 10:15:03 [ERROR] ডিস্কে জায়গা নেই
```

ব্যাখ্যা:
- লগ **stderr-এ** (`>&2`) পাঠানো হচ্ছে — যাতে স্ক্রিপ্টের আসল আউটপুট (stdout) পাইপ করলে লগ মেশে না। এটা Unix-এর নীতি: "ডেটা stdout-এ, ডায়াগনস্টিক stderr-এ।"
- `tee -a` একসাথে ফাইলে append ও স্ক্রিনে দেখায়।
- `printf` ব্যবহার করা হয়েছে কারণ `echo`-এর আচরণ (যেমন `-e`, `-n`) shell ভেদে আলাদা।

### লেভেল অনুযায়ী ফিল্টার

```bash
declare -A LEVELS=([DEBUG]=0 [INFO]=1 [WARN]=2 [ERROR]=3)
LOG_LEVEL="${LOG_LEVEL:-INFO}"

log() {
    local level="$1"; shift
    (( ${LEVELS[$level]} >= ${LEVELS[$LOG_LEVEL]} )) || return 0
    printf '%s [%-5s] %s\n' "$(date '+%F %T')" "$level" "$*" >&2
}
```

`LOG_LEVEL=DEBUG ./script.sh` চালালে সব দেখাবে, ডিফল্টে DEBUG লুকানো।

### `logger` — syslog/journald-এ লেখা

```bash
logger -t mybackup -p user.err "ব্যাকআপ ফেল করেছে"
```

| অপশন | কাজ |
|---|---|
| `-t TAG` | কোন প্রোগ্রামের নামে লগ হবে |
| `-p FACILITY.LEVEL` | priority, যেমন `user.info`, `local0.warning`, `user.err` |
| `-s` | একসাথে stderr-এও দেখাও |
| `-i` | PID যোগ করো |

```
$ journalctl -t mybackup --since "10 min ago"
Nov 12 10:20:11 myserver mybackup[5123]: ব্যাকআপ ফেল করেছে
```

লগ নিয়ে বিস্তারিত [অধ্যায় ২৮](28-logs.md)-এ।

### পুরো স্ক্রিপ্টের আউটপুট একসাথে লগে পাঠানো

```bash
exec > >(tee -a "$LOG_FILE") 2>&1
```

এরপর থেকে সব stdout ও stderr স্ক্রিনেও আসবে, লগ ফাইলেও যাবে। (`>(...)` হলো process substitution — [অধ্যায় ১৯](19-redirection-pipe.md) দেখো।)

---

## ৪৩.৯ রঙিন আউটপুট

### ANSI escape code

টার্মিনাল কিছু বিশেষ অক্ষর-ক্রম (escape sequence) পেলে সেগুলো লেখা হিসেবে না দেখিয়ে রঙ বদলায়। গঠন: `ESC[<code>m`, যেখানে ESC = `\e` বা `\033`।

| কোড | মানে | কোড | মানে |
|---|---|---|---|
| `0` | রিসেট (সব স্বাভাবিক) | `1` | বোল্ড |
| `2` | ফিকে (dim) | `4` | আন্ডারলাইন |
| `30` | কালো | `31` | লাল |
| `32` | সবুজ | `33` | হলুদ |
| `34` | নীল | `35` | ম্যাজেন্টা |
| `36` | সায়ান | `37` | সাদা |
| `40–47` | ব্যাকগ্রাউন্ড রঙ | `90–97` | উজ্জ্বল রঙ |

```bash
printf '\e[31mলাল লেখা\e[0m স্বাভাবিক\n'
printf '\e[1;32mবোল্ড সবুজ\e[0m\n'
echo -e "\033[33mহলুদ\033[0m"
```

> ⚠️ **সতর্কতা:** শেষে `\e[0m` (রিসেট) না দিলে পরের সব লেখা, এমনকি তোমার prompt-ও, রঙিন থেকে যাবে।

### `tput` — বেশি portable উপায়

`tput` terminfo ডাটাবেস দেখে বর্তমান টার্মিনালের জন্য সঠিক কোড দেয়:

```bash
RED=$(tput setaf 1); GREEN=$(tput setaf 2); YELLOW=$(tput setaf 3)
BOLD=$(tput bold); RESET=$(tput sgr0)
echo "${GREEN}সফল${RESET}"
```

`tput cols` (টার্মিনালের প্রস্থ), `tput lines`, `tput clear` ইত্যাদিও কাজে লাগে।

### কখন রঙ দেবে না

আউটপুট যদি ফাইলে বা পাইপে যায়, রঙের কোড আবর্জনা হয়ে যায় (`^[[31m`)। তাই চেক করো stdout আসলেই টার্মিনাল কি না:

```bash
if [[ -t 1 && -z "${NO_COLOR:-}" ]]; then
    C_RED=$'\e[31m'; C_GRN=$'\e[32m'; C_YLW=$'\e[33m'; C_RST=$'\e[0m'
else
    C_RED=''; C_GRN=''; C_YLW=''; C_RST=''
fi

ok()   { printf '%s[ OK ]%s %s\n'   "$C_GRN" "$C_RST" "$*"; }
fail() { printf '%s[FAIL]%s %s\n'   "$C_RED" "$C_RST" "$*" >&2; }
```

- `[[ -t 1 ]]` — fd 1 (stdout) একটা টার্মিনাল হলে সত্য।
- `NO_COLOR` — একটা প্রচলিত কনভেনশন (no-color.org): এই environment variable সেট থাকলে প্রোগ্রাম রঙ দেয় না।
- `$'\e[31m'` — ANSI-C quoting; variable-এ আসল ESC অক্ষর ঢুকে যায়, তাই `printf`/`echo` যেকোনোটায় কাজ করে।

---

## ৪৩.১০ Exit code কনভেনশন

প্রতিটি process শেষে 0–255 এর মধ্যে একটা সংখ্যা ফেরত দেয়। **0 = সফল, বাকি সব = কোনো না কোনো ব্যর্থতা।** (উল্টো মনে হতে পারে, কিন্তু যুক্তি হলো: সফল হওয়ার একটাই উপায়, ব্যর্থ হওয়ার অনেক কারণ।)

| কোড | প্রচলিত অর্থ |
|---|---|
| `0` | সফল |
| `1` | সাধারণ ত্রুটি |
| `2` | ভুল ব্যবহার (shell builtin-এর ভুল argument; অনেক টুল usage error-এ 2 দেয়) |
| `126` | কমান্ড পাওয়া গেছে কিন্তু চালানো যায়নি (execute permission নেই) |
| `127` | কমান্ড পাওয়া যায়নি (command not found) |
| `128+N` | signal N-এ মারা গেছে |
| `130` | `Ctrl+C` (128+2, SIGINT) |
| `137` | `kill -9` (128+9, SIGKILL) — Docker/OOM killer-এ প্রায়ই দেখা যায় |
| `143` | `kill` / SIGTERM (128+15) |
| `255` | সীমার বাইরে (যেমন `exit -1` → 255) |

```
$ nai_emon_command; echo $?
nai_emon_command: command not found
127
$ exit 256   # আসলে 256 % 256 = 0 হয়ে যায়!
```

> ⚠️ **সতর্কতা:** exit code ৮ বিটের, তাই `exit 256` = `exit 0`, `exit 300` = `exit 44`। কখনো 255-এর বেশি দিও না।

BSD-এর `sysexits.h` আরও নির্দিষ্ট কিছু কোড প্রস্তাব করে (অনেক প্রোগ্রাম, যেমন mail server, এগুলো মানে):

| কোড | নাম | অর্থ |
|---|---|---|
| 64 | `EX_USAGE` | কমান্ড ভুলভাবে ব্যবহার |
| 65 | `EX_DATAERR` | ইনপুট ডেটা ভুল |
| 66 | `EX_NOINPUT` | ইনপুট ফাইল নেই/পড়া যায় না |
| 69 | `EX_UNAVAILABLE` | সার্ভিস পাওয়া যাচ্ছে না |
| 70 | `EX_SOFTWARE` | ভেতরের সফটওয়্যার ত্রুটি |
| 73 | `EX_CANTCREAT` | আউটপুট ফাইল তৈরি করা যায়নি |
| 75 | `EX_TEMPFAIL` | সাময়িক ব্যর্থতা, পরে আবার চেষ্টা করো |
| 77 | `EX_NOPERM` | অনুমতি নেই |
| 78 | `EX_CONFIG` | কনফিগারেশন ভুল |

নিজের স্ক্রিপ্টে নামযুক্ত constant ব্যবহার করো এবং `--help`-এ ডকুমেন্ট করো:

```bash
readonly E_OK=0 E_USAGE=2 E_LOCKED=3 E_DISKFULL=4
```

Monitoring টুল (Nagios/Icinga plugin) এর নিজস্ব কনভেনশন: `0=OK, 1=WARNING, 2=CRITICAL, 3=UNKNOWN`। হেলথ-চেক স্ক্রিপ্টে এটা প্রায়ই ব্যবহার হয় (প্রজেক্ট ৫ দেখো)।

---

## ৪৩.১১ আরও কিছু প্রফেশনাল কৌশল

### ৪৩.১১.১ নিরাপদ temp ফাইল: `mktemp`

```bash
# ❌ বিপজ্জনক: নাম অনুমান করা যায়, অন্য ইউজার symlink বসিয়ে আক্রমণ করতে পারে
tmp=/tmp/myscript.tmp

# ✅ নিরাপদ
tmp=$(mktemp)                          # /tmp/tmp.XXXXXXXXXX, permission 600
tmpdir=$(mktemp -d)                    # ডিরেক্টরি, permission 700
tmp=$(mktemp /tmp/backup.XXXXXX.log)   # নিজের টেমপ্লেট (অন্তত ৩টা X)
tmp=$(mktemp -t backup.XXXXXX)         # $TMPDIR বা /tmp-এর ভেতরে
```

### ৪৩.১১.২ `readonly` ও `local`

```bash
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="${0##*/}"

myfunc() {
    local count=0    # শুধু এই function-এর ভেতরে
}
```

`readonly` variable পরে ভুল করে বদলাতে গেলে error দেবে। `local` না দিলে function-এর variable পুরো স্ক্রিপ্টে global হয়ে যায় — অন্য জায়গার একই নামের variable নষ্ট করতে পারে।

### ৪৩.১১.৩ `main "$@"` প্যাটার্ন

```bash
main() {
    parse_args "$@"
    check_requirements
    do_work
}
main "$@"
```

সুবিধা:
1. সব function আগে define হয়, তারপর চলে — ক্রম নিয়ে ঝামেলা নেই।
2. স্ক্রিপ্টের "গল্প" `main` পড়েই বোঝা যায়।
3. ফাইল অর্ধেক ডাউনলোড হলে (`curl | bash` এর সময়) শেষ লাইন `main "$@"` না পৌঁছালে কিছুই চলবে না — অর্ধেক কাজের বিপদ নেই।

### ৪৩.১১.৪ প্রয়োজনীয় কমান্ড ও root চেক

```bash
require_cmd() {
    local c
    for c in "$@"; do
        command -v "$c" >/dev/null 2>&1 || die "প্রয়োজনীয় কমান্ড নেই: $c"
    done
}
require_cmd rsync tar gzip

require_root() {
    [[ $EUID -eq 0 ]] || die "এই স্ক্রিপ্ট root হিসেবে চালাতে হবে (sudo দাও)"
}
```

`command -v` হলো POSIX উপায়ে কমান্ড আছে কি না জানা (`which`-এর চেয়ে নির্ভরযোগ্য)। `$EUID` হলো effective user ID — root হলে 0। ([অধ্যায় ১৮](18-sudo-root.md))

### ৪৩.১১.৫ Dry-run মোড

```bash
DRY_RUN=0
run() {
    if (( DRY_RUN )); then
        printf '[dry-run] %q ' "$@"; echo
    else
        "$@"
    fi
}
run rm -rf -- "$old_dir"
```

`-n` অপশনে `DRY_RUN=1` সেট করলে স্ক্রিপ্ট শুধু বলবে কী করত, আসলে কিছু করবে না। বিপজ্জনক অপারেশনের আগে এটা অমূল্য। `printf '%q'` argument-কে shell-quoted আকারে দেখায়।

### ৪৩.১১.৬ নিশ্চিতকরণ চাওয়া

```bash
confirm() {
    local reply
    read -r -p "${1:-নিশ্চিত?} [y/N] " reply
    [[ "$reply" =~ ^[Yy]([Ee][Ss])?$ ]]
}
confirm "সব লগ মুছে ফেলব?" || exit 0
```

### ৪৩.১১.৭ Retry লজিক

```bash
retry() {
    local max=$1 delay=$2 n=1; shift 2
    until "$@"; do
        (( n >= max )) && return 1
        warn "চেষ্টা $n ব্যর্থ, ${delay}s পরে আবার..."
        sleep "$delay"
        n=$((n + 1))
        delay=$((delay * 2))       # exponential backoff
    done
}
retry 5 2 curl -fsS https://example.com/health
```

### ৪৩.১১.৮ `timeout`

```bash
timeout 30 some_slow_command || warn "৩০ সেকেন্ডে শেষ হয়নি"
```

`timeout` (coreutils) কমান্ড নির্দিষ্ট সময়ের বেশি চললে SIGTERM পাঠায় এবং exit code 124 দেয়। `-k 5` দিলে তারপরও না মরলে ৫ সেকেন্ড পরে SIGKILL।

---

## ৪৩.১২ ভালো অভ্যাস চেকলিস্ট

প্রতিটি প্রোডাকশন স্ক্রিপ্ট লেখার পর এই তালিকা মিলিয়ে নাও:

| # | চেক | কেন |
|---|---|---|
| ১ | `#!/usr/bin/env bash` বা `#!/bin/bash` shebang আছে | কোন interpreter চালাবে নিশ্চিত |
| ২ | `set -euo pipefail` আছে (আর এর ফাঁদগুলো জানো) | fail fast |
| ৩ | সব variable `"$quoted"` | space/glob থেকে সুরক্ষা |
| ৪ | `shellcheck` পরিষ্কার | লুকানো bug |
| ৫ | function-এ `local` | variable সংঘর্ষ নেই |
| ৬ | constant-এ `readonly`, UPPERCASE নাম; সাধারণ variable lowercase | পড়তে সহজ, environment variable-এর সাথে সংঘর্ষ কম |
| ৭ | `mktemp` + `trap cleanup EXIT` | আবর্জনা ফেলে যায় না |
| ৮ | `-h`/`--help` এবং usage মেসেজ | অন্যরা ব্যবহার করতে পারে |
| ৯ | ইনপুট যাচাই (সংখ্যা? ফাইল আছে?) | ভুল ইনপুটে বিপর্যয় নেই |
| ১০ | error মেসেজ stderr-এ, অর্থবহ exit code | অন্য টুল/cron বুঝতে পারে |
| ১১ | লগ: সময় + লেভেল | পরে তদন্ত সহজ |
| ১২ | cron/timer-এ চললে lock (`flock`) | দ্বৈত চালনা নেই |
| ১৩ | `rm -rf "$var/"` এর আগে `$var` খালি না হওয়া নিশ্চিত (`${var:?}`) | `rm -rf /` দুর্ঘটনা নেই |
| ১৪ | পরম পাথ (absolute path) বা `cd` এর ফল চেক | cron-এর ভিন্ন working directory |
| ১৫ | cron-এর জন্য `PATH` সেট করা | cron-এ PATH খুব ছোট থাকে |
| ১৬ | গোপন তথ্য (password) কোডে না, আলাদা ফাইলে `chmod 600` | নিরাপত্তা |
| ১৭ | idempotent — দুইবার চালালেও ক্ষতি নেই | পুনরায় চালানো নিরাপদ |
| ১৮ | dry-run মোড (বিপজ্জনক কাজে) | আগে দেখে নাও |
| ১৯ | `ls` এর আউটপুট পার্স না করে glob/`find` ব্যবহার | অদ্ভুত ফাইলনামে ভাঙে না |
| ২০ | মন্তব্য: "কী" না, "কেন" লেখো | কোড নিজেই "কী" বলে |

`${var:?}` কৌশল:

```bash
rm -rf -- "${BACKUP_DIR:?BACKUP_DIR খালি!}/old"
```

`BACKUP_DIR` খালি বা unset হলে কমান্ড চলার আগেই স্ক্রিপ্ট থামবে।

### পুনর্ব্যবহারযোগ্য স্ক্রিপ্ট টেমপ্লেট

```bash
#!/usr/bin/env bash
#
# script-name.sh — এক লাইনে কী করে
# ব্যবহার: script-name.sh [-h] [-v] [-n] ARG
#
set -Eeuo pipefail

readonly SCRIPT_NAME="${0##*/}"
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

VERBOSE=0
DRY_RUN=0

# ---------- রঙ ও লগিং ----------
if [[ -t 2 && -z "${NO_COLOR:-}" ]]; then
    C_R=$'\e[31m' C_G=$'\e[32m' C_Y=$'\e[33m' C_0=$'\e[0m'
else
    C_R='' C_G='' C_Y='' C_0=''
fi
log()  { printf '%s %s\n' "$(date '+%F %T')" "$*" >&2; }
info() { log "${C_G}INFO${C_0}  $*"; }
warn() { log "${C_Y}WARN${C_0}  $*"; }
die()  { log "${C_R}ERROR${C_0} $*"; exit 1; }

# ---------- cleanup ও error ----------
TMPDIR_WORK=""
cleanup() {
    local rc=$?
    [[ -n "$TMPDIR_WORK" ]] && rm -rf -- "$TMPDIR_WORK"
    exit "$rc"
}
trap cleanup EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
trap 'warn "লাইন $LINENO এ ব্যর্থ: $BASH_COMMAND"' ERR

usage() {
    cat <<EOF
ব্যবহার: $SCRIPT_NAME [-h] [-v] [-n] ARG
  -h  সাহায্য
  -v  verbose
  -n  dry-run (শুধু দেখাও, কিছু করো না)
EOF
}

parse_args() {
    local opt
    while getopts ":hvn" opt; do
        case "$opt" in
            h) usage; exit 0 ;;
            v) VERBOSE=1 ;;
            n) DRY_RUN=1 ;;
            \?) usage >&2; die "অচেনা অপশন: -$OPTARG" ;;
        esac
    done
    shift $((OPTIND - 1))
    [[ $# -ge 1 ]] || { usage >&2; exit 2; }
    ARG="$1"
}

run() {
    if (( DRY_RUN )); then printf '[dry-run] %q ' "$@"; echo; else "$@"; fi
}

main() {
    parse_args "$@"
    (( VERBOSE )) && set -x
    TMPDIR_WORK=$(mktemp -d)
    info "শুরু: ARG=$ARG (dir: $SCRIPT_DIR)"
    # ... আসল কাজ ...
    info "শেষ"
}

main "$@"
```

নতুন স্ক্রিপ্ট লেখার সময় এই টেমপ্লেট কপি করে শুরু করলে প্রতিবার ভালো অভ্যাসগুলো আপনা-আপনি চলে আসবে।

---

## ৪৩.১৩ বাস্তব প্রজেক্ট ১: স্বয়ংক্রিয় ব্যাকআপ স্ক্রিপ্ট

**লক্ষ্য:** এক বা একাধিক ডিরেক্টরির সময়-চিহ্নিত (timestamped) `.tar.gz` ব্যাকআপ তৈরি করা, ব্যাকআপ ঠিক আছে কি না যাচাই করা, checksum রাখা, আর নির্দিষ্ট দিনের পুরোনো ব্যাকআপ মুছে ফেলা (rotation)। cron থেকে চালানোর উপযোগী — lock সহ।

```
  /etc  /home/ali  /var/www      (সোর্স)
        │
        ▼  tar -czf  (প্রথমে .part নামে)
  backup-web01-20261112-020001.tar.gz.part
        │  tar -tzf দিয়ে যাচাই
        ▼  mv (atomic rename)
  backup-web01-20261112-020001.tar.gz  +  .sha256
        │
        ▼  find -mtime +7 -delete
  ৭ দিনের পুরোনো ব্যাকআপ মুছে গেল
```

```bash
 1  #!/usr/bin/env bash
 2  # backup.sh — ডিরেক্টরির timestamped tar.gz ব্যাকআপ ও পুরোনো ব্যাকআপ মোছা
 3  set -Eeuo pipefail
 4
 5  readonly SCRIPT_NAME="${0##*/}"
 6  DEST="/var/backups/mybackup"
 7  KEEP_DAYS=7
 8  readonly LOCK="/run/lock/${SCRIPT_NAME%.sh}.lock"
 9
10  log() { printf '%s [%s] %s\n' "$(date '+%F %T')" "$1" "${*:2}" >&2; }
11  die() { log ERROR "$*"; logger -t "$SCRIPT_NAME" -p user.err -- "$*" || true; exit 1; }
12
13  usage() {
14      cat <<EOF
15  ব্যবহার: $SCRIPT_NAME [-d DEST] [-k DAYS] SOURCE_DIR...
16    -d DEST   ব্যাকআপ কোথায় রাখা হবে (ডিফল্ট: $DEST)
17    -k DAYS   কত দিনের পুরোনো ব্যাকআপ রাখা হবে (ডিফল্ট: $KEEP_DAYS)
18  EOF
19  }
20
21  while getopts ":d:k:h" opt; do
22      case "$opt" in
23          d) DEST="$OPTARG" ;;
24          k) KEEP_DAYS="$OPTARG" ;;
25          h) usage; exit 0 ;;
26          :) die "-$OPTARG অপশনের মান দরকার" ;;
27          \?) die "অচেনা অপশন: -$OPTARG" ;;
28      esac
29  done
30  shift $((OPTIND - 1))
31  (( $# > 0 )) || { usage >&2; exit 2; }
32  SRC_DIRS=("$@")
33  [[ "$KEEP_DAYS" =~ ^[0-9]+$ ]] || die "DAYS একটা ধনাত্মক পূর্ণসংখ্যা হতে হবে"
34
35  exec 9>"$LOCK"
36  flock -n 9 || die "আরেকটি ব্যাকআপ ইতিমধ্যে চলছে"
37
38  for d in "${SRC_DIRS[@]}"; do
39      [[ -d "$d" ]] || die "সোর্স ডিরেক্টরি নেই: $d"
40  done
41  mkdir -p -- "$DEST"
42
43  ts=$(date +%Y%m%d-%H%M%S)
44  archive="$DEST/backup-$(hostname -s)-$ts.tar.gz"
45  partial="$archive.part"
46  trap 'rm -f -- "$partial"' EXIT
47
48  log INFO "ব্যাকআপ শুরু: ${SRC_DIRS[*]} -> $archive"
49  rc=0
50  tar -czf "$partial" "${SRC_DIRS[@]}" 2> >(grep -v 'Removing leading' >&2) || rc=$?
51  if (( rc > 1 )); then
52      die "tar ব্যর্থ (exit $rc)"
53  elif (( rc == 1 )); then
54      log WARN "কিছু ফাইল পড়ার সময় বদলেছে (tar exit 1) — ব্যাকআপ তবুও রাখা হলো"
55  fi
56
57  tar -tzf "$partial" >/dev/null || die "আর্কাইভ নষ্ট, যাচাই ব্যর্থ"
58  mv -- "$partial" "$archive"
59  ( cd "$DEST" && sha256sum -- "${archive##*/}" > "${archive##*/}.sha256" )
60
61  size=$(du -h -- "$archive" | cut -f1)
62  log INFO "সফল: $archive ($size)"
63
64  deleted=$(find "$DEST" -maxdepth 1 -type f \
65      \( -name 'backup-*.tar.gz' -o -name 'backup-*.tar.gz.sha256' \) \
66      -mtime +"$KEEP_DAYS" -print -delete | wc -l)
67  log INFO "রোটেশন: $deleted টি পুরোনো ফাইল মোছা হয়েছে (> $KEEP_DAYS দিন)"
68  logger -t "$SCRIPT_NAME" -p user.info -- "backup ok: $archive ($size)" || true
```

(লাইন নম্বরগুলো শুধু ব্যাখ্যার সুবিধার জন্য; আসল ফাইলে লিখবে না।)

**লাইন-বাই-লাইন ব্যাখ্যা:**

| লাইন | ব্যাখ্যা |
|---|---|
| 1 | `env` দিয়ে PATH থেকে bash খোঁজা — bash যেখানেই ইনস্টল থাকুক কাজ করবে। |
| 3 | strict mode, সাথে `-E` যাতে function-এর ভেতরেও ERR trap উত্তরাধিকার পায়। |
| 5 | `${0##*/}` — `$0` থেকে শেষ `/` পর্যন্ত সব কেটে শুধু ফাইলের নাম (`basename`-এর দ্রুত বিকল্প)। |
| 6–7 | ডিফল্ট মান; অপশন দিয়ে বদলানো যাবে, তাই `readonly` না। |
| 8 | `${SCRIPT_NAME%.sh}` শেষের `.sh` কেটে দেয় → `/run/lock/backup.lock`। `/run` tmpfs, তাই reboot-এ এমনিতেই পরিষ্কার। |
| 10 | `"${*:2}"` — দ্বিতীয় argument থেকে বাকি সব একসাথে। লগ stderr-এ। |
| 11 | `die` স্ক্রিনে + syslog/journald-এ error লিখে বের হয়। `logger` না থাকলেও যেন স্ক্রিপ্ট আটকে না যায় তাই `\|\| true`। `--` মানে "এরপর আর অপশন না" — মেসেজ `-` দিয়ে শুরু হলেও নিরাপদ। |
| 13–19 | here-document দিয়ে help টেক্সট; `$DEST` এখানে expand হয়ে আসল ডিফল্ট দেখায়। |
| 21–30 | `getopts` পার্সিং (৪৩.৬-এ বিস্তারিত)। |
| 31 | অন্তত একটা সোর্স না দিলে usage দেখিয়ে exit 2 (usage error)। |
| 32 | বাকি সব argument একটা array-তে — space-যুক্ত পাথও নিরাপদ থাকে। |
| 33 | regex দিয়ে ইনপুট যাচাই; `-k abc` দিলে `find` পরে অদ্ভুত error দিত। |
| 35–36 | fd 9-এ lock ফাইল খুলে non-blocking `flock`। স্ক্রিপ্ট মরলে lock আপনা থেকে মুক্ত। |
| 38–40 | সব সোর্স আগে থেকেই আছে কি না যাচাই — অর্ধেক কাজ করে ফেল করার চেয়ে শুরুতেই ফেল করা ভালো। |
| 41 | গন্তব্য না থাকলে তৈরি; `-p` থাকলে আগে থেকে থাকলেও error দেয় না। |
| 43–44 | `hostname -s` (short hostname) + সময় দিয়ে অনন্য নাম; `%Y%m%d-%H%M%S` ফরম্যাট বর্ণানুক্রমে সাজালে সময়ানুক্রমেও সাজে। |
| 45–46 | প্রথমে `.part` নামে লেখা হয়; মাঝপথে ফেল করলে EXIT trap অসম্পূর্ণ ফাইলটা মুছে দেয়, তাই "আধা-ব্যাকআপ" কখনো আসল ব্যাকআপের মতো দেখায় না। |
| 49–50 | GNU tar-এর exit code: 0 = সফল, 1 = কিছু ফাইল পড়ার সময় বদলেছে, 2 = মারাত্মক ত্রুটি। `set -e` exit 1-এও থামাত, তাই `\|\| rc=$?` দিয়ে কোডটা ধরে নিজে সিদ্ধান্ত নিচ্ছি। `2> >(grep -v ...)` — absolute path দিলে tar যে "Removing leading `/' from member names" সতর্কতা দেয়, সেটা ফিল্টার করে বাকি error stderr-এ পাঠায়। |
| 51–55 | 2 বা তার বেশি = মারাত্মক → die; 1 = সতর্কতা, ব্যাকআপ রাখা হয় (চালু সিস্টেমে লগ ফাইল বদলানো স্বাভাবিক)। |
| 57 | `-t` = list; পুরো আর্কাইভ পড়ে দেখে gzip/tar গঠন ঠিক আছে কি না। ব্যাকআপ যাচাই না করা মানে ব্যাকআপ না থাকা! |
| 58 | একই ফাইলসিস্টেমে `mv` হলো atomic rename — হয় পুরোনো নাম, নয় নতুন নাম; মাঝামাঝি অবস্থা নেই। |
| 59 | subshell `( ... )`-এ `cd` করা, যাতে মূল স্ক্রিপ্টের ডিরেক্টরি না বদলায়। checksum ফাইলে শুধু ফাইলের নাম থাকায় পরে `sha256sum -c` দিয়ে যেকোনো জায়গা থেকে যাচাই করা যায়। |
| 61 | `du -h` আউটপুট `1.2G<TAB>path`; `cut -f1` প্রথম ঘর (tab-separated)। |
| 64–66 | রোটেশন: শুধু এই ডিরেক্টরির (`-maxdepth 1`) নির্দিষ্ট প্যাটার্নের ফাইল, যেগুলো `KEEP_DAYS` দিনের বেশি পুরোনো (`-mtime +N`), সেগুলো প্রিন্ট ও মুছে; `wc -l` দিয়ে গুনছি। প্যাটার্ন নির্দিষ্ট রাখায় ভুল করে অন্য ফাইল মুছবে না। |
| 68 | সফলতাও syslog-এ — monitoring সিস্টেম "শেষ সফল ব্যাকআপ কখন" জানতে পারে। |

ব্যবহার ও আউটপুট:

```
$ sudo ./backup.sh -k 14 /etc /var/www
2026-11-12 02:00:01 [INFO] ব্যাকআপ শুরু: /etc /var/www -> /var/backups/mybackup/backup-web01-20261112-020001.tar.gz
2026-11-12 02:00:19 [INFO] সফল: /var/backups/mybackup/backup-web01-20261112-020001.tar.gz (148M)
2026-11-12 02:00:19 [INFO] রোটেশন: 2 টি পুরোনো ফাইল মোছা হয়েছে (> 14 দিন)
$ cd /var/backups/mybackup && sha256sum -c backup-web01-20261112-020001.tar.gz.sha256
backup-web01-20261112-020001.tar.gz: OK
```

cron-এ প্রতিদিন রাত ২টায় (`sudo crontab -e`):

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
0 2 * * * /usr/local/bin/backup.sh -k 14 /etc /var/www >> /var/log/backup.log 2>&1
```

> 💡 **টিপস:** একই মেশিনে রাখা ব্যাকআপ আসলে পুরো ব্যাকআপ না — ডিস্ক নষ্ট হলে দুটোই যাবে। ৩-২-১ নিয়ম (৩ কপি, ২ ধরনের মিডিয়া, ১টা অন্য জায়গায়) আর `rsync` দিয়ে দূরে কপি করার কৌশল [অধ্যায় ৩৫](35-backup-rsync.md)-এ আছে। আর্কাইভ নিয়ে বিস্তারিত [অধ্যায় ৩৪](34-archive-compression.md)-এ।

---

## ৪৩.১৪ বাস্তব প্রজেক্ট ২: ডিস্ক স্পেস অ্যালার্ট

**লক্ষ্য:** সব আসল ফাইলসিস্টেমের ব্যবহার দেখা; WARNING ও CRITICAL সীমা পার হলে রঙিন সতর্কতা, syslog-এ লেখা, ঐচ্ছিকভাবে ইমেইল পাঠানো; আর monitoring-বান্ধব exit code (0/1/2) দেওয়া।

```bash
 1  #!/usr/bin/env bash
 2  # disk-alert.sh — ফাইলসিস্টেম ব্যবহারের সতর্কতা
 3  set -euo pipefail
 4
 5  WARN=80
 6  CRIT=90
 7  MAIL_TO=""
 8
 9  if [[ -t 1 ]]; then
10      R=$'\e[31m' Y=$'\e[33m' G=$'\e[32m' N=$'\e[0m'
11  else
12      R='' Y='' G='' N=''
13  fi
14
15  while getopts ":w:c:m:h" opt; do
16      case "$opt" in
17          w) WARN="$OPTARG" ;;
18          c) CRIT="$OPTARG" ;;
19          m) MAIL_TO="$OPTARG" ;;
20          h) echo "ব্যবহার: ${0##*/} [-w WARN%] [-c CRIT%] [-m EMAIL]"; exit 0 ;;
21          *) echo "ভুল অপশন" >&2; exit 3 ;;
22      esac
23  done
24  [[ "$WARN" =~ ^[0-9]+$ && "$CRIT" =~ ^[0-9]+$ ]] || { echo "সীমা সংখ্যা হতে হবে" >&2; exit 3; }
25  (( WARN < CRIT )) || { echo "WARN অবশ্যই CRIT এর চেয়ে ছোট" >&2; exit 3; }
26
27  status=0
28  report=""
29
30  while read -r pcent avail mount; do
31      use="${pcent%\%}"
32      if (( use >= CRIT )); then
33          line="CRITICAL ${use}% ব্যবহৃত, খালি ${avail} — ${mount}"
34          printf '%s%s%s\n' "$R" "$line" "$N"
35          status=2
36      elif (( use >= WARN )); then
37          line="WARNING  ${use}% ব্যবহৃত, খালি ${avail} — ${mount}"
38          printf '%s%s%s\n' "$Y" "$line" "$N"
39          (( status < 1 )) && status=1
40      else
41          printf '%sOK       %s%% %s%s\n' "$G" "$use" "$mount" "$N"
42          continue
43      fi
44      report+="$line"$'\n'
45  done < <(df -h --output=pcent,avail,target \
46              -x tmpfs -x devtmpfs -x squashfs -x overlay -x efivarfs | tail -n +2)
47
48  if (( status > 0 )); then
49      logger -t disk-alert -p user.warning -- "$(hostname -s): ${report//$'\n'/; }" || true
50      if [[ -n "$MAIL_TO" ]] && command -v mail >/dev/null; then
51          printf '%s\n' "$report" | mail -s "[disk-alert] $(hostname -s)" "$MAIL_TO"
52      fi
53  fi
54  exit "$status"
```

**লাইন-বাই-লাইন ব্যাখ্যা:**

| লাইন | ব্যাখ্যা |
|---|---|
| 5–7 | ডিফল্ট সীমা: ৮০% সতর্কতা, ৯০% সংকট। |
| 9–13 | stdout টার্মিনাল হলে তবেই রঙ; cron-এর লগে escape code যাবে না। একই লাইনে একাধিক assignment চলে। |
| 15–23 | অপশন পার্স। এখানে ভুলে exit 3 — Nagios কনভেনশনে 3 = UNKNOWN ("চেকটাই ঠিকমতো চলেনি")। |
| 24–25 | ইনপুট যাচাই — `-w 95 -c 90` দেওয়ার মতো যুক্তিহীন কনফিগারেশন আটকানো। |
| 27–28 | `status` সামগ্রিক ফল ধরে রাখে; `report` সমস্যাযুক্ত লাইনগুলো জমায়। |
| 30 | `read -r pcent avail mount` — প্রতিটি লাইন তিন ঘরে ভাগ। শেষ variable `mount` লাইনের বাকি সবটা পায়, তাই mount point-এ space থাকলেও ঠিক থাকে। `df` ডানদিকে সাজানোর জন্য যে শুরুর space দেয়, ডিফল্ট `IFS` সেটা বাদ দেয়। |
| 31 | `${pcent%\%}` — শেষের `%` চিহ্ন কেটে শুধু সংখ্যা (`87%` → `87`)। |
| 32–35 | সংকট হলে লাল রঙে লেখা ও status 2। |
| 36–39 | সতর্কতা হলে হলুদ; কিন্তু আগেই 2 হয়ে থাকলে সেটা কমিয়ে 1 করব না — তাই শর্ত। |
| 40–42 | ঠিক থাকলে সবুজ OK, আর `continue` দিয়ে report-এ না যোগ করে পরের লাইনে। `%%` হলো printf-এ আক্ষরিক `%`। |
| 44 | `+=` দিয়ে string-এ যোগ; `$'\n'` আসল newline অক্ষর। |
| 45–46 | **গুরুত্বপূর্ণ:** `df ... \| while read` না লিখে `while ... done < <(df ...)` লেখা হয়েছে। pipe দিলে `while` লুপ একটা **subshell**-এ চলত, আর লুপের ভেতরে সেট করা `status` ও `report` লুপ শেষে হারিয়ে যেত! process substitution `<(...)` এ লুপ মূল shell-এই চলে। `--output=pcent,avail,target` শুধু দরকারি কলাম দেয় (GNU coreutils)। `-x` দিয়ে RAM-ভিত্তিক বা read-only ভার্চুয়াল ফাইলসিস্টেম বাদ — snap-এর `squashfs` সবসময় 100% দেখায়, ওগুলো মিথ্যা অ্যালার্ম দিত। `tail -n +2` হেডার লাইন বাদ দেয়। |
| 49 | `${report//$'\n'/; }` — সব newline কে `; ` দিয়ে প্রতিস্থাপন, যাতে syslog-এ এক লাইনে আসে। |
| 50–51 | ইমেইল ঠিকানা দেওয়া থাকলে ও `mail` কমান্ড (Debian/Ubuntu-তে `mailutils` বা `bsd-mailx`, Fedora/RHEL-এ `s-nail`) ইনস্টল থাকলে তবেই মেইল। |
| 54 | 0/1/2 exit code — cron wrapper বা monitoring টুল এ দেখে সিদ্ধান্ত নিতে পারে। |

```
$ ./disk-alert.sh -w 70 -c 85
OK       34% /
WARNING  78% ব্যবহৃত, খালি 22G — /var
OK       12% /boot
OK       5% /boot/efi
CRITICAL 91% ব্যবহৃত, খালি 9.1G — /home
$ echo $?
2
```

> 💡 **টিপস:** ডিস্ক "full" হওয়ার আরেকটা কারণ inode শেষ হয়ে যাওয়া (অনেক ছোট ফাইল)। একই স্ক্রিপ্টে `--output=ipcent,iavail,target` দিয়ে inode-ও চেক করতে পারো। inode সম্পর্কে [অধ্যায় ১১](11-links-inode.md) আর ডিস্ক সম্পর্কে [অধ্যায় ৩২](32-disk-partition-mount.md) দেখো।

---

## ৪৩.১৫ বাস্তব প্রজেক্ট ৩: ওয়েব সার্ভার লগ বিশ্লেষক

**লক্ষ্য:** Nginx/Apache-এর "combined" ফরম্যাটের access log পড়ে একটা রিপোর্ট বানানো: মোট request, সবচেয়ে বেশি request করা IP, জনপ্রিয় URL, HTTP status-এর বণ্টন, error rate, মোট ডেটা, আর সবচেয়ে বেশি 404। `.gz` করা পুরোনো লগও পড়বে।

একটা combined লগ লাইন ও তার ঘর (awk-এর ডিফল্ট whitespace বিভাজনে):

```
203.0.113.7 - - [12/Nov/2026:10:15:32 +0600] "GET /index.html HTTP/1.1" 200 5123 "-" "Mozilla/5.0 ..."
    $1      $2 $3        $4            $5     $6      $7        $8     $9   $10   $11    $12...
   (IP)                (সময়)                (method) (URL)            (status)(bytes)
```

```bash
 1  #!/usr/bin/env bash
 2  # log-report.sh — combined ফরম্যাট access log বিশ্লেষণ
 3  set -euo pipefail
 4
 5  TOP=10
 6  usage() { echo "ব্যবহার: ${0##*/} [-n TOP] LOGFILE..."; }
 7
 8  while getopts ":n:h" opt; do
 9      case "$opt" in
10          n) TOP="$OPTARG" ;;
11          h) usage; exit 0 ;;
12          *) usage >&2; exit 2 ;;
13      esac
14  done
15  shift $((OPTIND - 1))
16  (( $# > 0 )) || { usage >&2; exit 2; }
17  [[ "$TOP" =~ ^[1-9][0-9]*$ ]] || { echo "TOP ধনাত্মক সংখ্যা হতে হবে" >&2; exit 2; }
18
19  for f in "$@"; do
20      [[ -r "$f" ]] || { echo "পড়া যাচ্ছে না: $f" >&2; exit 66; }
21  done
22
23  data=$(mktemp)
24  trap 'rm -f -- "$data"' EXIT
25  zcat -f -- "$@" > "$data"
26
27  section() { printf '\n=== %s ===\n' "$1"; }
28  rank() { sort | uniq -c | sort -rn | head -n "$TOP" || true; }
29
30  total=$(wc -l < "$data")
31  if (( total == 0 )); then echo "লগ খালি"; exit 0; fi
32
33  section "সারাংশ"
34  awk -v total="$total" '
35      { bytes += ($10 ~ /^[0-9]+$/) ? $10 : 0
36        if ($9 ~ /^4/) c4++
37        if ($9 ~ /^5/) c5++ }
38      END {
39        printf "মোট request : %d\n", total
40        printf "মোট ডেটা    : %.2f MB\n", bytes/1048576
41        printf "4xx হার     : %.2f%%\n", c4*100/total
42        printf "5xx হার     : %.2f%%\n", c5*100/total
43      }' "$data"
44  printf 'প্রথম entry : %s\n' "$(head -n1 "$data" | awk '{print substr($4,2)}')"
45  printf 'শেষ entry   : %s\n' "$(tail -n1 "$data" | awk '{print substr($4,2)}')"
46
47  section "শীর্ষ $TOP IP"
48  awk '{print $1}' "$data" | rank
49
50  section "শীর্ষ $TOP URL"
51  awk '{print $7}' "$data" | rank
52
53  section "HTTP status বণ্টন"
54  awk '{print $9}' "$data" | sort | uniq -c | sort -rn
55
56  section "শীর্ষ $TOP 404 URL"
57  awk '$9 == 404 {print $7}' "$data" | rank
58
59  section "প্রতি ঘণ্টায় request"
60  awk '{print substr($4, 14, 2)}' "$data" | sort | uniq -c
```

**লাইন-বাই-লাইন ব্যাখ্যা:**

| লাইন | ব্যাখ্যা |
|---|---|
| 5 | ডিফল্টে শীর্ষ ১০টা দেখাবে। |
| 8–17 | অপশন পার্স ও যাচাই; regex `^[1-9][0-9]*$` শূন্য বা ঋণাত্মক আটকায়। |
| 19–21 | প্রতিটি ফাইল পড়া যায় কি না (`-r`)। না গেলে exit 66 (`EX_NOINPUT`)। |
| 23–24 | একটা নিরাপদ temp ফাইল, যেটা শেষে মুছে যাবে। |
| 25 | `zcat -f` — gzip ফাইল হলে খুলে দেয়, সাধারণ ফাইল হলে হুবহু দেয় (`-f` = force)। তাই `access.log` আর `access.log.2.gz` একসাথে দেওয়া যায়। সব ডেটা একবার temp ফাইলে রাখায় বারবার decompress করতে হয় না। |
| 27 | রিপোর্টের প্রতিটি অংশের শিরোনাম ছাপার ছোট্ট helper। |
| 28 | (নাম `rank`, কারণ `top` নামে আগেই একটা লিনাক্স কমান্ড আছে।) "গুনে সাজাও" প্যাটার্ন: `sort` (একই জিনিস পাশাপাশি আনে) → `uniq -c` (গোনে) → `sort -rn` (সংখ্যা অনুযায়ী বড় থেকে ছোট) → `head`। এটা লিনাক্সের সবচেয়ে বিখ্যাত one-liner প্যাটার্নগুলোর একটা ([অধ্যায় ২৩](23-text-tools.md))। এই function stdin থেকে পড়ে, তাই pipe-এর শেষে বসানো যায়। শেষের `\|\| true` জরুরি: অনেক আলাদা IP থাকলে `head` ১০ লাইন পড়েই বন্ধ হয়, আর তখনো লিখতে থাকা `sort` SIGPIPE পেয়ে 141 দেয় — `pipefail` + `set -e` পুরো রিপোর্ট মাঝপথে থামিয়ে দিত (৪৩.২.৩-এর ফাঁদ)। |
| 30–31 | `wc -l < file` — redirection দিলে আউটপুটে ফাইলের নাম আসে না, শুধু সংখ্যা। খালি লগে শূন্য দিয়ে ভাগ এড়াতে আগেই বের হই। |
| 34 | `-v total=...` দিয়ে shell variable awk-এ পাঠানো ([অধ্যায় ২২](22-awk.md))। |
| 35 | bytes ঘরে `-` থাকতে পারে (যেমন 304 response-এ), তাই সংখ্যা হলে তবেই যোগ। |
| 36–37 | status 4 বা 5 দিয়ে শুরু হলে গুনি। |
| 38–43 | `END` ব্লক সব লাইন পড়ার পর একবার চলে; `%%` আক্ষরিক `%`। `1048576` = 1024×1024। |
| 44–45 | `$4` হলো `[12/Nov/2026:10:15:32`; `substr($4,2)` প্রথম `[` বাদ দেয়। |
| 48 | প্রথম ঘর = ক্লায়েন্ট IP। কোনো একটা IP অস্বাভাবিক বেশি হলে সেটা bot বা আক্রমণ হতে পারে। |
| 51 | সপ্তম ঘর = অনুরোধ করা URL। |
| 54 | status বণ্টন — এখানে `head` নেই, সব status দেখাই। |
| 57 | awk-এর pattern `$9 == 404` — শুধু ওই লাইনগুলোর URL। ভাঙা লিংক বা স্ক্যানার (`/wp-login.php` খোঁজা bot) ধরা পড়ে। |
| 60 | `$4`-এর ১৪তম অক্ষর থেকে ২টা অক্ষর = ঘণ্টা (`[12/Nov/2026:` ১৩ অক্ষর)। ঘণ্টা অনুযায়ী ট্রাফিক বোঝা যায়। |

```
$ ./log-report.sh -n 3 /var/log/nginx/access.log /var/log/nginx/access.log.2.gz

=== সারাংশ ===
মোট request : 48213
মোট ডেটা    : 912.47 MB
4xx হার     : 6.31%
5xx হার     : 0.42%
প্রথম entry : 10/Nov/2026:00:00:04
শেষ entry   : 12/Nov/2026:10:15:32

=== শীর্ষ 3 IP ===
   5120 198.51.100.23
   2301 203.0.113.7
    988 192.0.2.44

=== শীর্ষ 3 URL ===
  10544 /
   6120 /api/v1/status
   3300 /static/app.js

=== HTTP status বণ্টন ===
  41022 200
   3610 304
   2301 404
    981 301
    203 500
     96 403

=== শীর্ষ 3 404 URL ===
    812 /wp-login.php
    430 /.env
    101 /favicon.ico
...
```

> 📝 **মনে রাখো:** একাধিক ফাইল দিলে ক্রম গুরুত্বপূর্ণ — "প্রথম/শেষ entry" সঠিক পেতে পুরোনো ফাইল আগে দাও। আর `$4`-ভিত্তিক বিশ্লেষণ শুধু combined/common লগ ফরম্যাটে কাজ করে; কাস্টম `log_format` হলে ঘরের নম্বর বদলাবে।

> 💡 **টিপস:** `/.env`, `/wp-login.php`-এর মতো অনুরোধ মানে কেউ তোমার সার্ভার স্ক্যান করছে। এমন IP স্বয়ংক্রিয়ভাবে ব্লক করার জন্য fail2ban বা ফায়ারওয়াল ([অধ্যায় ৩৯](39-firewall.md), [অধ্যায় ৪৫](45-security-hardening.md)) দেখো।

---

## ৪৩.১৬ বাস্তব প্রজেক্ট ৪: CSV থেকে একসাথে অনেক ইউজার তৈরি

**লক্ষ্য:** নতুন ৫০ জন কর্মী যোগ দিয়েছে। একটা CSV ফাইল থেকে সবার অ্যাকাউন্ট তৈরি করা, প্রয়োজনীয় গ্রুপ তৈরি করা, প্রত্যেককে একটা র‍্যান্ডম অস্থায়ী পাসওয়ার্ড দেওয়া যেটা প্রথম লগইনে বদলাতে বাধ্য করবে, আর পাসওয়ার্ডগুলো একটা সুরক্ষিত ফাইলে রাখা। ইতিমধ্যে থাকা ইউজার এড়িয়ে যাবে (idempotent)। dry-run মোডও থাকবে।

ইনপুট ফাইল `users.csv`:

```
# username,full name,groups (; দিয়ে আলাদা)
rahim,Rahim Uddin,developers;docker
karim,Karim Hasan,developers
nasrin,Nasrin Akter,hr
```

```bash
 1  #!/usr/bin/env bash
 2  # bulk-useradd.sh — CSV থেকে ইউজার তৈরি
 3  set -euo pipefail
 4
 5  DRY_RUN=0
 6  CREDS="/root/new-users-$(date +%F).csv"
 7
 8  log()  { printf '[%s] %s\n' "$1" "${*:2}" >&2; }
 9  run()  { if (( DRY_RUN )); then printf '[dry-run] %q ' "$@"; echo; else "$@"; fi; }
10
11  while getopts ":no:h" opt; do
12      case "$opt" in
13          n) DRY_RUN=1 ;;
14          o) CREDS="$OPTARG" ;;
15          h) echo "ব্যবহার: ${0##*/} [-n] [-o CREDS_FILE] USERS.csv"; exit 0 ;;
16          *) echo "ভুল অপশন" >&2; exit 2 ;;
17      esac
18  done
19  shift $((OPTIND - 1))
20  [[ $# -eq 1 ]] || { echo "ঠিক একটা CSV ফাইল দাও" >&2; exit 2; }
21  CSV="$1"
22  [[ -r "$CSV" ]] || { log ERROR "পড়া যাচ্ছে না: $CSV"; exit 66; }
23  [[ $EUID -eq 0 || $DRY_RUN -eq 1 ]] || { log ERROR "root হিসেবে চালাও"; exit 77; }
24  command -v openssl >/dev/null || { log ERROR "openssl লাগবে"; exit 69; }
25
26  if (( ! DRY_RUN )); then
27      ( umask 077; : >> "$CREDS" )
28  fi
29
30  created=0 skipped=0 failed=0
31  lineno=0
32  while IFS=, read -r -u 3 user fullname groups || [[ -n "$user" ]]; do
33      lineno=$((lineno + 1))
34      groups="${groups%$'\r'}"
35      [[ -z "$user" || "$user" == \#* ]] && continue
36
37      if ! [[ "$user" =~ ^[a-z_][a-z0-9_-]{0,31}$ ]]; then
38          log ERROR "লাইন $lineno: অবৈধ username '$user'"
39          failed=$((failed + 1)); continue
40      fi
41      if id -u "$user" >/dev/null 2>&1; then
42          log SKIP "$user ইতিমধ্যে আছে"
43          skipped=$((skipped + 1)); continue
44      fi
45
46      opts=(-m -s /bin/bash -c "$fullname")
47      if [[ -n "$groups" ]]; then
48          IFS=';' read -r -a glist <<< "$groups"
49          for g in "${glist[@]}"; do
50              getent group "$g" >/dev/null || run groupadd "$g"
51          done
52          opts+=(-G "$(IFS=,; echo "${glist[*]}")")
53      fi
54
55      if ! run useradd "${opts[@]}" "$user"; then
56          log ERROR "$user তৈরি ব্যর্থ"
57          failed=$((failed + 1)); continue
58      fi
59
60      pass=$(openssl rand -base64 12)
61      if (( DRY_RUN )); then
62          echo "[dry-run] $user এর পাসওয়ার্ড সেট ও chage -d 0"
63      else
64          printf '%s:%s\n' "$user" "$pass" | chpasswd
65          chage -d 0 "$user"
66          printf '%s,%s\n' "$user" "$pass" >> "$CREDS"
67      fi
68      log OK "$user ($fullname) তৈরি হয়েছে"
69      created=$((created + 1))
70  done 3< "$CSV"
71
72  log DONE "তৈরি: $created, এড়ানো: $skipped, ব্যর্থ: $failed"
73  (( DRY_RUN )) || log INFO "পাসওয়ার্ড ফাইল: $CREDS (শুধু root পড়তে পারে)"
74  (( failed == 0 ))
```

**লাইন-বাই-লাইন ব্যাখ্যা:**

| লাইন | ব্যাখ্যা |
|---|---|
| 6 | পাসওয়ার্ড ফাইল root-এর home-এ, তারিখসহ নাম। |
| 9 | dry-run wrapper (৪৩.১১.৫)। |
| 20–22 | ঠিক একটা ইনপুট ফাইল, পড়ার যোগ্য। |
| 23 | `$EUID` 0 না হলে (root না) exit 77 (`EX_NOPERM`); তবে dry-run কেউ যেকোনো ইউজার হিসেবে চালাতে পারে। |
| 24 | পাসওয়ার্ড বানাতে `openssl` লাগবে; না থাকলে 69 (`EX_UNAVAILABLE`)। |
| 26–28 | subshell-এ `umask 077` দিয়ে ফাইল তৈরি → permission `600` (শুধু মালিক পড়তে/লিখতে পারে)। subshell হওয়ায় umask বাকি স্ক্রিপ্টে প্রভাব ফেলে না। `: >>` মানে "কিছু না লিখে ফাইলটা খোলো" (না থাকলে তৈরি হবে)। পাসওয়ার্ড লেখার **আগেই** permission ঠিক করা জরুরি। ([অধ্যায় ১৬](16-permissions.md)) |
| 32 | মূল লুপ। `IFS=,` শুধু এই `read`-এর জন্য বিভাজক কমা। `-u 3` = fd 3 থেকে পড়ো (লাইন 70-এ `3< "$CSV"`)। কেন stdin না? লুপের ভেতরের কোনো কমান্ড (যেমন `ssh`, বা ইন্টারঅ্যাকটিভ প্রোগ্রাম) stdin পড়ে ফেললে CSV-এর বাকি লাইন খেয়ে ফেলত — আলাদা fd ব্যবহার এই সূক্ষ্ম bug প্রতিরোধ করে। `\|\| [[ -n "$user" ]]` — ফাইলের শেষ লাইনে newline না থাকলে `read` ফেল করে কিন্তু ডেটা পড়ে ফেলে; এই শর্ত সেই শেষ লাইনটাও প্রক্রিয়া করে। |
| 34 | Windows-এ (Excel) বানানো CSV-তে লাইনের শেষে `\r` (CRLF) থাকে; শেষ ঘর থেকে সেটা কাটা। নাহলে গ্রুপের নাম হতো `hr\r` — অদৃশ্য কিন্তু মারাত্মক bug। |
| 35 | খালি লাইন আর `#` দিয়ে শুরু মন্তব্য বাদ। |
| 37–40 | username যাচাই: ছোট হাতের অক্ষর বা `_` দিয়ে শুরু, সর্বোচ্চ ৩২ অক্ষর — নিরাপদ, portable নাম। ভুল হলে গুনে পরের লাইনে। |
| 41–44 | `id -u` সফল মানে ইউজার আছে → এড়িয়ে যাও। তাই স্ক্রিপ্ট দুইবার চালালেও ক্ষতি নেই (idempotent)। |
| 46 | useradd-এর অপশন **array**-তে রাখা — `-c "Rahim Uddin"`-এর মতো space-যুক্ত মান নিরাপদে একটা argument থাকে। `-m` home ডিরেক্টরি তৈরি (Debian/Ubuntu-তে `useradd` ডিফল্টে home তৈরি করে না, RHEL/Fedora-তে করে — তাই স্পষ্ট করে দেওয়াই ভালো), `-s` login shell, `-c` comment/পূর্ণ নাম। |
| 48 | `IFS=';' read -a` — `developers;docker` কে array-তে ভাঙা। `<<<` হলো here-string। |
| 49–51 | `getent group` গ্রুপ খোঁজে (`/etc/group` + LDAP ইত্যাদি সব উৎসে)। না থাকলে তৈরি। |
| 52 | subshell-এ `IFS=,` সেট করে `"${glist[*]}"` → array-কে কমা দিয়ে জোড়া: `developers,docker` — `-G` এই ফরম্যাটই চায়। |
| 55–58 | `useradd` ব্যর্থ হলে পুরো স্ক্রিপ্ট না থামিয়ে (এটা `if`-এর শর্ত, তাই `set -e` থামায় না) এই ইউজার ব্যর্থ হিসেবে গুনে পরেরটায় যাই। |
| 60 | `openssl rand -base64 12` — ১২টা র‍্যান্ডম বাইট → ১৬ অক্ষরের পাসওয়ার্ড। (`tr -dc ... </dev/urandom \| head -c 16` ও প্রচলিত, কিন্তু `pipefail`-এ `tr` SIGPIPE পেয়ে 141 দেয় আর স্ক্রিপ্ট মরে যায় — ৪৩.২.৩-এর ফাঁদ!) |
| 64 | `chpasswd` stdin থেকে `user:password` পড়ে পাসওয়ার্ড সেট করে। কমান্ড-লাইন argument-এ পাসওয়ার্ড দিলে সেটা `ps`-এ অন্যরা দেখতে পারত — pipe দিয়ে দেওয়ায় দেখা যায় না (`printf` bash builtin, আলাদা process হয় না)। |
| 65 | `chage -d 0` — "শেষ পাসওয়ার্ড বদলের তারিখ" ০ করে দেয়, ফলে প্রথম লগইনেই নতুন পাসওয়ার্ড দিতে বাধ্য। |
| 66 | পাসওয়ার্ড শুধু ৬০০-permission ফাইলে লেখা হয়, স্ক্রিনে বা লগে না। |
| 70 | `3< "$CSV"` — পুরো লুপের জন্য fd 3-এ CSV খোলা। |
| 74 | শেষ কমান্ডের status-ই স্ক্রিপ্টের exit code: কোনো ব্যর্থতা না থাকলে 0, থাকলে 1। (`set -e` শেষ কমান্ডের ক্ষেত্রে আলাদা কিছু করে না — exit code একই থাকে।) |

```
$ sudo ./bulk-useradd.sh -n users.csv
[dry-run] groupadd developers
[dry-run] useradd -m -s /bin/bash -c Rahim\ Uddin -G developers,docker rahim
[dry-run] rahim এর পাসওয়ার্ড সেট ও chage -d 0
[OK] rahim (Rahim Uddin) তৈরি হয়েছে
...
$ sudo ./bulk-useradd.sh users.csv
[OK] rahim (Rahim Uddin) তৈরি হয়েছে
[OK] karim (Karim Hasan) তৈরি হয়েছে
[OK] nasrin (Nasrin Akter) তৈরি হয়েছে
[DONE] তৈরি: 3, এড়ানো: 0, ব্যর্থ: 0
[INFO] পাসওয়ার্ড ফাইল: /root/new-users-2026-11-12.csv (শুধু root পড়তে পারে)
$ sudo ./bulk-useradd.sh users.csv
[SKIP] rahim ইতিমধ্যে আছে
[SKIP] karim ইতিমধ্যে আছে
[SKIP] nasrin ইতিমধ্যে আছে
[DONE] তৈরি: 0, এড়ানো: 3, ব্যর্থ: 0
$ sudo chage -l rahim | head -n1
Last password change					: password must be changed
```

> ⚠️ **সতর্কতা:** dry-run-এ গ্রুপ আসলে তৈরি হয় না, তাই একই গ্রুপ একাধিক ইউজারের জন্য বারবার `[dry-run] groupadd` দেখাতে পারে — এটা স্বাভাবিক। পাসওয়ার্ড ফাইল ইউজারদের হাতে দেওয়ার পর `shred -u` দিয়ে মুছে ফেলো। প্রশাসনিক ক্ষমতা দিতে হলে Ubuntu/Debian-এ `sudo` গ্রুপ, Fedora/RHEL-এ `wheel` গ্রুপ। ইউজার-গ্রুপের বিস্তারিত [অধ্যায় ১৫](15-users-groups.md)-এ।

---

## ৪৩.১৭ বাস্তব প্রজেক্ট ৫: সার্ভিস হেলথ চেক ও স্বয়ংক্রিয় রিস্টার্ট

**লক্ষ্য:** কয়েকটা systemd সার্ভিস চলছে কি না, কয়েকটা TCP পোর্ট খোলা কি না, আর কয়েকটা HTTP endpoint ঠিক response দিচ্ছে কি না পরীক্ষা করা। `-r` দিলে বন্ধ সার্ভিস রিস্টার্ট করার চেষ্টা। ফলাফল রঙিন টেবিলে, আর Nagios-ধাঁচের exit code।

```
            health-check.sh
     ┌──────────┼───────────────┐
     ▼          ▼               ▼
 systemctl   /dev/tcp/host/port   curl -w '%{http_code}'
 is-active   (TCP connect)        (HTTP status)
     │          │               │
     └────► OK / FAIL ◄──────────┘
                │
       -r? → systemctl restart → আবার চেক
                │
        exit 0 (সব ঠিক) / 2 (কিছু ভাঙা)
```

```bash
 1  #!/usr/bin/env bash
 2  # health-check.sh — সার্ভিস, পোর্ট ও HTTP endpoint পরীক্ষা
 3  set -uo pipefail
 4
 5  SERVICES=(nginx ssh cron)
 6  PORTS=(localhost:22 localhost:80)
 7  URLS=(http://localhost/ http://localhost/healthz)
 8  AUTO_RESTART=0
 9  CONF="/etc/health-check.conf"
10  [[ -r "$CONF" ]] && source "$CONF"
11
12  if [[ -t 1 ]]; then G=$'\e[32m' R=$'\e[31m' Y=$'\e[33m' N=$'\e[0m'; else G='' R='' Y='' N=''; fi
13  ok()   { printf '  %s[ OK ]%s %s\n' "$G" "$N" "$*"; }
14  bad()  { printf '  %s[FAIL]%s %s\n' "$R" "$N" "$*"; problems=$((problems + 1)); }
15  note() { printf '  %s[INFO]%s %s\n' "$Y" "$N" "$*"; }
16
17  while getopts ":rh" opt; do
18      case "$opt" in
19          r) AUTO_RESTART=1 ;;
20          h) echo "ব্যবহার: ${0##*/} [-r]   (-r = বন্ধ সার্ভিস রিস্টার্ট)"; exit 0 ;;
21          *) echo "ভুল অপশন" >&2; exit 3 ;;
22      esac
23  done
24
25  problems=0
26
27  check_service() {
28      local svc=$1
29      if systemctl is-active --quiet "$svc"; then
30          ok "service $svc চলছে"
31          return
32      fi
33      if (( AUTO_RESTART )); then
34          note "service $svc বন্ধ — রিস্টার্ট করছি"
35          systemctl restart "$svc" 2>/dev/null
36          sleep 3
37          if systemctl is-active --quiet "$svc"; then
38              ok "service $svc রিস্টার্টের পর চলছে"
39              logger -t health-check -- "restarted $svc" || true
40              return
41          fi
42      fi
43      bad "service $svc বন্ধ ($(systemctl is-active "$svc" 2>/dev/null))"
44  }
45
46  check_port() {
47      local host=${1%:*} port=${1##*:}
48      if timeout 3 bash -c ": > /dev/tcp/$host/$port" 2>/dev/null; then
49          ok "port $host:$port খোলা"
50      else
51          bad "port $host:$port এ সংযোগ হচ্ছে না"
52      fi
53  }
54
55  check_url() {
56      local url=$1 code time
57      read -r code time < <(curl -s -o /dev/null --max-time 5 \
58                              -w '%{http_code} %{time_total}' "$url")
59      if [[ "$code" =~ ^[23] ]]; then
60          ok "url $url → $code (${time}s)"
61      else
62          bad "url $url → ${code:-000}"
63      fi
64  }
65
66  echo "== Health check: $(hostname -s) @ $(date '+%F %T') =="
67  echo "Services:"; for s in "${SERVICES[@]}"; do check_service "$s"; done
68  echo "Ports:";    for p in "${PORTS[@]}";    do check_port "$p";    done
69  echo "HTTP:";     for u in "${URLS[@]}";     do check_url "$u";     done
70
71  if (( problems > 0 )); then
72      echo "${R}মোট সমস্যা: $problems${N}"
73      logger -t health-check -p user.err -- "$problems problem(s) on $(hostname -s)" || true
74      exit 2
75  fi
76  echo "${G}সব ঠিক আছে${N}"
77  exit 0
```

**লাইন-বাই-লাইন ব্যাখ্যা:**

| লাইন | ব্যাখ্যা |
|---|---|
| 3 | লক্ষ্য করো: এখানে `-e` **নেই**, ইচ্ছাকৃতভাবে। হেলথ-চেকের কাজই হলো ব্যর্থতা খোঁজা — একটা চেক ফেল করলে পুরো স্ক্রিপ্ট থেমে গেলে বাকি চেক হবে না। strict mode অন্ধভাবে নয়, বুঝে ব্যবহার করো। |
| 5–7 | ডিফল্ট লক্ষ্য। Ubuntu/Debian-এ SSH সার্ভিসের নাম `ssh` (সাথে `sshd` alias), Fedora/RHEL-এ `sshd`; cron Fedora/RHEL-এ `crond`। নিজের সার্ভার অনুযায়ী বদলাও। |
| 9–10 | কনফিগ ফাইল থাকলে `source` করে variable গুলো ওভাররাইড করা — কোড না বদলে প্রতি সার্ভারে আলাদা তালিকা। ⚠️ `source` করা ফাইল কোড হিসেবে চলে, তাই এটা root-এর মালিকানায় ও অন্যদের অলেখ্য হতে হবে। |
| 12–15 | রঙ ও তিনটা আউটপুট helper। `bad` প্রতিবার `problems` বাড়ায় (global variable)। |
| 17–23 | `-r` থাকলে auto-restart চালু। |
| 27–28 | `local` দিয়ে function-এর নিজস্ব variable। |
| 29 | `systemctl is-active --quiet` কিছু প্রিন্ট করে না, শুধু exit code দেয়: সক্রিয় হলে 0। স্ক্রিপ্টে চেক করার আদর্শ উপায়। |
| 33–42 | auto-restart চালু থাকলে রিস্টার্ট, ৩ সেকেন্ড অপেক্ষা (সার্ভিস ওঠার সময়), আবার চেক; সফল হলে syslog-এ লিখে রাখা — পরে কেউ জানবে কেন সার্ভিস রিস্টার্ট হয়েছিল। (রিস্টার্টের জন্য root লাগে।) |
| 43 | ব্যর্থ — `systemctl is-active` এর লেখা অবস্থাও (`inactive`, `failed`) দেখানো হচ্ছে, সমস্যা বুঝতে সাহায্য করে। |
| 47 | `${1%:*}` শেষ `:` থেকে ডান দিক কেটে host, `${1##*:}` শেষ `:` পর্যন্ত বাঁ দিক কেটে port। |
| 48 | bash-এর বিশেষ ফিচার: `/dev/tcp/HOST/PORT` আসলে কোনো ফাইল না — এতে redirect করলে bash নিজেই TCP সংযোগ খোলে। আলাদা `bash -c` এ চালানো হয়েছে যাতে `timeout 3` পুরোটা মেরে ফেলতে পারে (ফায়ারওয়াল চুপচাপ প্যাকেট ফেলে দিলে সংযোগ অনেকক্ষণ ঝুলে থাকতে পারত)। `nc -z host port` ও বিকল্প, কিন্তু সব জায়গায় ইনস্টল থাকে না। |
| 56–58 | `curl -s` (চুপচাপ), `-o /dev/null` (body ফেলে দাও), `--max-time 5` (সর্বোচ্চ ৫ সেকেন্ড), `-w` দিয়ে শুধু status code ও মোট সময় প্রিন্ট। `read ... < <(...)` দিয়ে দুটো মান দুই variable-এ। সংযোগই না হলে curl `000` প্রিন্ট করে। |
| 59 | 2xx বা 3xx = ঠিক আছে। |
| 66–69 | তিন ধরনের চেক ক্রমানুসারে। |
| 71–77 | সমস্যা থাকলে syslog-এ লিখে exit 2 (CRITICAL), না থাকলে 0 (OK)। |

```
$ sudo ./health-check.sh -r
== Health check: web01 @ 2026-11-12 10:40:05 ==
Services:
  [ OK ] service nginx চলছে
  [ OK ] service ssh চলছে
  [INFO] service cron বন্ধ — রিস্টার্ট করছি
  [ OK ] service cron রিস্টার্টের পর চলছে
Ports:
  [ OK ] port localhost:22 খোলা
  [ OK ] port localhost:80 খোলা
HTTP:
  [ OK ] url http://localhost/ → 200 (0.003121s)
  [FAIL] url http://localhost/healthz → 404
মোট সমস্যা: 1
$ echo $?
2
```

প্রতি ৫ মিনিটে চালাতে (`sudo crontab -e`):

```
*/5 * * * * /usr/bin/flock -n /run/lock/health.lock /usr/local/bin/health-check.sh -r >/dev/null 2>&1
```

> 💡 **টিপস:** systemd নিজেও সার্ভিস মরলে রিস্টার্ট করতে পারে — unit ফাইলে `Restart=on-failure`। তাই এই স্ক্রিপ্টের আসল মূল্য হলো "সার্ভিস চলছে কিন্তু কাজ করছে না" (HTTP 500, পোর্ট বন্ধ) ধরতে পারা। [অধ্যায় ২৬](26-systemd-services.md) ও cron-এর বদলে systemd timer [অধ্যায় ২৯](29-scheduling-cron.md)-এ দেখো।

---

## ৪৩.১৮ সম্পর্কিত কীবোর্ড শর্টকাট

স্ক্রিপ্ট চালানো, থামানো ও ডিবাগ করার সময় যেগুলো লাগে (টার্মিনালের পূর্ণ তালিকা [অধ্যায় ৬](06-terminal-shortcuts.md)-এ, Vim-এর [অধ্যায় ১৪](14-vim.md)-এ):

| শর্টকাট | কাজ | স্ক্রিপ্টিং-এ প্রাসঙ্গিকতা |
|---|---|---|
| `Ctrl+C` | foreground process-কে SIGINT পাঠায় | `trap ... INT` ধরে; ডিফল্টে exit 130 |
| `Ctrl+\` | SIGQUIT পাঠায় | `Ctrl+C` উপেক্ষা করা process থামাতে; core dump হতে পারে |
| `Ctrl+Z` | SIGTSTP — process সাময়িক থামায় | `fg`/`bg` দিয়ে ফেরানো ([অধ্যায় ২৫](25-jobs-tmux.md)) |
| `Ctrl+D` | ইনপুটের শেষ (EOF) | `read` বা here-doc ইনপুট শেষ করতে |
| `Ctrl+S` / `Ctrl+Q` | টার্মিনাল আউটপুট থামাও / আবার চালাও | `set -x` এর দ্রুত চলমান trace থামিয়ে পড়তে; ভুলে `Ctrl+S` চাপলে "টার্মিনাল জমে গেছে" মনে হয় — `Ctrl+Q` চাপো |
| `Ctrl+X Ctrl+E` | বর্তমান কমান্ড `$EDITOR`-এ খোলো | লম্বা one-liner এডিটরে লিখে চালাও |
| `Ctrl+R` | হিস্ট্রিতে পেছনে খোঁজা | আগে চালানো `bash -x ...` দ্রুত ফেরত আনা |
| `Alt+.` | আগের কমান্ডের শেষ argument বসাও | `vim script.sh` এর পরে `shellcheck` + `Alt+.` |
| Vim-এ `:!shellcheck %` | এডিটর না ছেড়ে বর্তমান ফাইল চেক | `%` = বর্তমান ফাইলের নাম |
| Vim-এ `:w !bash` | সেভ না করেই buffer bash-এ চালাও | দ্রুত পরীক্ষা |

---

## হাতে-কলমে অনুশীলন

একটা অনুশীলন ফোল্ডার বানাও: `mkdir -p ~/lab43 && cd ~/lab43`

**ল্যাব ১ — strict mode-এর ফাঁদ নিজে দেখো**

1. `trap1.sh` লেখো:
   ```bash
   #!/usr/bin/env bash
   set -euo pipefail
   count=0
   ((count++))
   echo "count = $count"
   ```
2. চালাও: `bash trap1.sh; echo "exit=$?"`। কিছু প্রিন্ট হলো না, exit=1 — কেন? (উত্তর ৪৩.২.১-এ)
3. `((count++))` বদলে `count=$((count + 1))` করো, আবার চালাও।
4. `echo $UNDEFINED_VAR` যোগ করে `set -u`-এর error দেখো।
5. `cat /nai | sort; echo ok` লিখে `pipefail` সহ ও ছাড়া (`set +o pipefail`) পার্থক্য দেখো।

**ল্যাব ২ — trap ও cleanup**

1. একটা স্ক্রিপ্ট লেখো যেটা `mktemp -d` দিয়ে ফোল্ডার বানায়, `trap cleanup EXIT` সেট করে, ফোল্ডারের নাম প্রিন্ট করে, তারপর `sleep 60`।
2. চালিয়ে `Ctrl+C` চাপো। `ls` দিয়ে দেখো temp ফোল্ডার আছে কি না।
3. আরেক টার্মিনাল থেকে `kill <PID>` (SIGTERM) দিয়ে চেষ্টা করো, তারপর `kill -9 <PID>` — পার্থক্যটা লক্ষ করো। কেন `-9`-এ cleanup হলো না?
4. `trap 'exit 130' INT` এবং `trap 'exit 143' TERM` যোগ করে আবার পরীক্ষা করো।

**ল্যাব ৩ — ডিবাগিং**

1. প্রজেক্ট ২-এর স্ক্রিপ্ট `disk-alert.sh` নামে সেভ করো।
2. `bash -n disk-alert.sh` — syntax ঠিক?
3. `PS4='+ ${LINENO}: ' bash -x disk-alert.sh 2>&1 | head -30` — প্রতিটি লাইন কীভাবে চলে দেখো।
4. ইচ্ছাকৃতভাবে `done < <(df ...)` কে `df ... | while read ...` এ বদলাও; `-w 1` দিয়ে চালিয়ে দেখো `exit` code কী আসে, এবং কেন সবসময় 0।

**ল্যাব ৪ — shellcheck**

1. `sudo apt install shellcheck` (বা `sudo dnf install ShellCheck`)।
2. ৪৩.৫-এর `example.sh` লিখে `shellcheck example.sh` চালাও।
3. প্রতিটি সতর্কতা ঠিক করো যতক্ষণ না কোনো আউটপুট আসে।
4. এই অধ্যায়ের পাঁচটা প্রজেক্ট স্ক্রিপ্টের ওপর `shellcheck` চালাও — কোনো সতর্কতা এলে বোঝার চেষ্টা করো সেটা আসল সমস্যা কি না।

**ল্যাব ৫ — getopts ও lock**

1. ৪৩.৬.৩-এর `myhead.sh` লেখো। `-h`, `-n 3`, `-x`, `-n` (মান ছাড়া) দিয়ে চালিয়ে আচরণ দেখো।
2. `-q` (quiet) নামে একটা নতুন অপশন যোগ করো।
3. ৪৩.৭.২-এর lock demo-তে lock পাথ `/tmp/lockdemo.lock` করে (sudo ছাড়া চালানোর জন্য) দুই টার্মিনাল থেকে একসাথে চালাও।
4. প্রথমটা `kill -9` করো, তারপর দ্বিতীয়টা আবার চালাও — lock নিজে থেকে মুক্ত হয়েছে কি না দেখো।

**ল্যাব ৬ — নিজের প্রজেক্ট**

1. প্রজেক্ট ৪ (`bulk-useradd.sh`) একটা **ভার্চুয়াল মেশিনে বা কন্টেইনারে** (নিজের আসল মেশিনে না!) `-n` দিয়ে প্রথমে চালাও, তারপর আসলে চালাও। শেষে তৈরি করা ইউজার মুছতে `userdel -r` দিয়ে একটা "rollback" স্ক্রিপ্ট নিজে লেখো।
2. ৪৩.১২-এর টেমপ্লেট থেকে শুরু করে একটা `cleanup-old-files.sh` লেখো: `-d DIR -a DAYS` নেবে, `-n` dry-run সমর্থন করবে, আর নির্দিষ্ট দিনের পুরোনো `*.log` ফাইল মুছবে। `shellcheck` পরিষ্কার হওয়া চাই।

---

## সারসংক্ষেপ

| বিষয় | কমান্ড / কৌশল | মনে রাখার কথা |
|---|---|---|
| Strict mode | `set -euo pipefail` | if-শর্ত, `&&`/`\|\|`, `local x=$(..)`, `((i++))` ফাঁদ |
| ERR trap উত্তরাধিকার | `set -E` / `set -o errtrace` | function-এর ভেতরেও ERR trap |
| ঐচ্ছিক variable | `${var:-default}`, `${var:?msg}` | `set -u`-এর সাথে নিরাপদ |
| pipeline-এর অবস্থা | `${PIPESTATUS[@]}` | প্রতিটি অংশের exit code |
| Cleanup | `trap cleanup EXIT` | প্রথম লাইনে `local rc=$?` |
| Signal | `trap 'exit 130' INT`, `trap 'exit 143' TERM` | SIGKILL/SIGSTOP ধরা যায় না |
| Error এর অবস্থান | `trap '... $LINENO $BASH_COMMAND' ERR` | |
| Syntax চেক | `bash -n script.sh` | চালায় না |
| Trace | `bash -x`, `set -x` / `set +x` | গোপন তথ্য ফাঁস হতে পারে |
| Trace prefix | `PS4='+ ${BASH_SOURCE}:${LINENO}: '` | সিঙ্গেল কোটে |
| Trace আলাদা ফাইলে | `exec 5>f; BASH_XTRACEFD=5` | |
| Static analysis | `shellcheck script.sh` | `# shellcheck disable=SCxxxx` |
| অপশন পার্স | `while getopts ":ab:" opt` + `shift $((OPTIND-1))` | শুধু ছোট অপশন; `:`→মান নেই, `?`→অচেনা |
| Lock | `exec 9>lock; flock -n 9` | process মরলে আপনা থেকে মুক্ত |
| cron-এ lock | `flock -n /run/lock/x.lock cmd` | |
| Atomic lock | `mkdir lockdir` | stale lock সমস্যা |
| syslog | `logger -t TAG -p user.err "msg"` | `journalctl -t TAG` |
| সব আউটপুট লগে | `exec > >(tee -a log) 2>&1` | |
| রঙ | `$'\e[31m'...$'\e[0m'`, `tput setaf 1` / `tput sgr0` | `[[ -t 1 ]]`, `NO_COLOR` |
| Exit code | 0 সফল, 1 সাধারণ, 2 usage, 126, 127, 128+N | সর্বোচ্চ 255 |
| Temp ফাইল | `mktemp`, `mktemp -d` | permission 600/700 |
| কমান্ড আছে? | `command -v cmd` | |
| root? | `[[ $EUID -eq 0 ]]` | |
| সময়সীমা | `timeout 30 cmd` | timeout হলে exit 124 |
| subshell ফাঁদ | `while ...; done < <(cmd)` | `cmd \| while` এ variable হারায় |
| লুপে নিরাপদ ইনপুট | `read -u 3 ...; done 3< file` | ভেতরের কমান্ড stdin খায় না |

---

## নিজেকে যাচাই করো

1. `set -e` চালু থাকা অবস্থায় `if grep -q foo file; then ...` লাইনে `grep` কিছু না পেলে স্ক্রিপ্ট থামে না কেন?
2. `set -o pipefail` ছাড়া `false | true` এর exit status কত? চালু থাকলে কত?
3. `count=0; ((count++))` `set -e`-যুক্ত স্ক্রিপ্টে কী সমস্যা তৈরি করে, আর এর নিরাপদ বিকল্প কী?
4. `trap cleanup EXIT` থাকা সত্ত্বেও `kill -9` দিলে cleanup হয় না কেন?
5. `PS4` সবসময় সিঙ্গেল কোটে লেখা উচিত কেন?
6. `getopts ":vo:"` এ প্রথম `:` আর `o`-এর পরের `:` — দুটোর মানে কী?
7. `[ -f lockfile ] || touch lockfile` দিয়ে lock করা ভুল কেন, আর `flock` কীভাবে সমস্যা দুটো সমাধান করে?
8. একটা স্ক্রিপ্টের exit code 137 এলে কী বোঝা যায়? 127 এলে?
9. প্রজেক্ট ২-এ `df ... | while read ...` না লিখে `while ...; done < <(df ...)` লেখা হয়েছে কেন?
10. রঙিন আউটপুট দেওয়ার আগে `[[ -t 1 ]]` চেক করা হয় কেন?

<details><summary>উত্তর দেখো</summary>

1. `if`, `while`, `until`-এর শর্তে থাকা কমান্ডের ব্যর্থতাকে bash "প্রশ্নের উত্তর" হিসেবে ধরে, error হিসেবে না। তাই `errexit` সেখানে প্রযোজ্য নয়।
2. `pipefail` ছাড়া 0 (শুধু শেষ কমান্ড `true`-এর status)। `pipefail` সহ 1 — ডান দিক থেকে প্রথম non-zero status।
3. post-increment `count++` পুরোনো মান 0 ফেরত দেয়; `(( ))` এর মান 0 হলে exit status 1, ফলে `set -e` স্ক্রিপ্ট চুপচাপ থামিয়ে দেয়। নিরাপদ বিকল্প: `count=$((count + 1))`।
4. SIGKILL kernel সরাসরি কার্যকর করে; process-কে কোনো কোড চালানোর সুযোগ দেয় না, তাই কোনো trap (EXIT সহ) চলে না। SIGKILL ও SIGSTOP ধরা বা উপেক্ষা করা যায় না।
5. যাতে `$LINENO`, `$BASH_SOURCE`, `${FUNCNAME[0]}` ইত্যাদি সেট করার সময় একবার নয়, প্রতিটি trace লাইন ছাপার সময় নতুন করে expand হয়। ডাবল কোটে দিলে সবসময় একই (সেট করার সময়ের) লাইন নম্বর দেখাত।
6. প্রথম `:` = silent error mode: getopts নিজে error প্রিন্ট করে না; অচেনা অপশনে `opt='?'` আর মান না পেলে `opt=':'`, দুই ক্ষেত্রেই `OPTARG`-এ অপশনের অক্ষর। `o:` = `-o` অপশনের একটা মান লাগবে, যেটা `$OPTARG`-এ আসবে।
7. (ক) race condition: চেক আর তৈরি দুটো আলাদা ধাপ, তাই দুটো process একই সময়ে দুজনেই "lock নেই" দেখে ঢুকে যেতে পারে। (খ) stale lock: স্ক্রিপ্ট মারা গেলে ফাইল থেকে যায়। `flock` kernel-এর atomic file lock ব্যবহার করে (race নেই), আর lock একটা খোলা file descriptor-এর সাথে বাঁধা — process মরলে kernel fd বন্ধ করে lock মুক্ত করে দেয়।
8. 137 = 128 + 9 → SIGKILL দিয়ে মারা গেছে (যেমন `kill -9` বা OOM killer)। 127 = কমান্ড খুঁজে পাওয়া যায়নি (command not found)।
9. pipe-এর প্রতিটি অংশ (bash-এ ডিফল্টে শেষ অংশটাও) একটা subshell-এ চলে। তাই `while` লুপের ভেতরে বদলানো `status`/`report` লুপ শেষে মূল shell-এ হারিয়ে যেত। process substitution দিয়ে লুপ মূল shell-এ চলে, variable টিকে থাকে।
10. `-t 1` সত্য হয় যখন stdout একটা টার্মিনাল। আউটপুট ফাইলে, পাইপে বা cron-এর মেইলে গেলে ANSI escape code আবর্জনা (`^[[31m`) হিসেবে দেখা যেত এবং `grep`/`awk` দিয়ে প্রসেস করা কঠিন হতো।

</details>

---

[← আগের অধ্যায়](42-shell-scripting-2.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](44-environment-bashrc.md)

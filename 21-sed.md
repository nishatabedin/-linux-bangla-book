# অধ্যায় ২১: sed — স্ট্রিম এডিটর

## এই অধ্যায়ে যা শিখবে

- `sed` কী, কেন একে "stream editor" বলে, আর কোথায় এটা `nano`/`vim`-এর চেয়ে বেশি কাজের
- `sed` ভেতরে কীভাবে কাজ করে: **pattern space**, **hold space** আর প্রতিটি লাইনের **cycle** (ডায়াগ্রামসহ)
- সবচেয়ে বেশি ব্যবহৃত কমান্ড `s///` (substitute) আর তার flag: `g`, `i`/`I`, `p`, নম্বর (`2`, `3g`), `w`
- delimiter বদলানো (`s|...|...|`, `s#...#...#`), যাতে path নিয়ে কাজ সহজ হয়
- গুরুত্বপূর্ণ অপশন: `-n`, `-e`, `-f`, `-E`/`-r`, `-i`, `-i.bak`, `-s`, `-z`, `--debug`
- address: লাইন নম্বর, range, regex, `$`, `!` (উল্টো), `first~step`, `addr,+N`
- কমান্ড: `d`, `p`, `a`, `i`, `c`, `y`, `q`, `Q`, `=`, `n`, `N`, `r`, `w`
- `&` আর back-reference `\1`...`\9`, সাথে GNU-র `\U`, `\L` কেস-বদল
- hold space পরিচিতি: `h`, `H`, `g`, `G`, `x`
- একাধিক কমান্ড একসাথে চালানো, script ফাইল লেখা
- বাস্তব রেসিপি: কনফিগ ফাইল এডিট, ফাঁকা লাইন মোছা, কমেন্ট বাদ দেওয়া, লাইন জোড়া লাগানো, আর অনেক কিছু
- GNU sed বনাম BSD/macOS sed-এর পার্থক্য আর সাধারণ ভুলগুলো

---

## ২১.১ sed কী এবং কেন?

`sed` শব্দটা এসেছে **s**tream **ed**itor থেকে। ১৯৭৩-৭৪ সালে Bell Labs-এ Lee McMahon এটা তৈরি করেন, পুরনো লাইন-এডিটর `ed`-এর কমান্ডগুলোকে ভিত্তি করে। এজন্যই `ed`, `vi`/`vim` আর `sed`-এর substitute কমান্ড দেখতে প্রায় একই রকম: `s/old/new/`।

**Stream** মানে "স্রোত" — ডেটা যা এক দিক থেকে ঢুকে অন্য দিক দিয়ে বেরিয়ে যায়। `sed` একটা ফাইল খুলে তোমাকে এডিট করতে দেয় না (যেমন `nano` বা `vim` দেয়)। বরং এটা ইনপুট থেকে **একটা একটা লাইন** পড়ে, তোমার দেওয়া নির্দেশ অনুযায়ী লাইনটা বদলায়, তারপর output-এ ছেড়ে দেয়। কোনো মানুষ বসে বসে কী-বোর্ড টিপছে না — সব স্বয়ংক্রিয়।

> 💡 **উপমা:** একটা কারখানার conveyor belt কল্পনা করো। বেল্টের ওপর দিয়ে একটা একটা বাক্স (লাইন) আসছে। বেল্টের মাঝখানে একজন কর্মী (sed) দাঁড়িয়ে আছে, হাতে একটা নির্দেশের কাগজ (sed script)। সে প্রতিটা বাক্স তুলে নেয়, কাগজ দেখে লেবেল বদলায়, কোনোটা ফেলে দেয়, কোনোটার পাশে নতুন বাক্স রাখে, তারপর বাক্সটা আবার বেল্টে ছেড়ে দেয়। সে কখনো পেছনে ফিরে আগের বাক্স ধরতে পারে না (যদি না আলাদা করে একটা তাকে — hold space-এ — রেখে দেয়)।

### কেন sed শিখবে?

| পরিস্থিতি | sed ছাড়া | sed দিয়ে |
|---|---|---|
| ৫০টা কনফিগ ফাইলে `old.server.com` বদলে `new.server.com` করা | ৫০ বার ফাইল খুলে এডিট | এক লাইনের কমান্ড |
| 2 GB লগ ফাইল থেকে ১০০০–১০২০ নম্বর লাইন দেখা | এডিটরে খুললে মেমরি শেষ | `sed -n '1000,1020p' big.log` — মুহূর্তে |
| স্ক্রিপ্টের ভেতর থেকে কনফিগ বদলানো (automation) | মানুষ লাগবে | পুরোপুরি স্বয়ংক্রিয় |
| pipe-এর মাঝখানে টেক্সট রূপান্তর | অসম্ভব/কঠিন | `cmd | sed '...' | cmd2` |

`sed` প্রায় সব লিনাক্স সিস্টেমে আগে থেকেই থাকে (এমনকি ছোট container image বা rescue shell-এও, সাধারণত BusyBox-এর অংশ হিসেবে)। তাই সার্ভারে কাজ করতে গেলে এটা জানা প্রায় বাধ্যতামূলক।

> 📝 **মনে রাখো:** `grep` লাইন **খোঁজে**, `sed` লাইন **বদলায়**, আর `awk` লাইনকে কলামে ভেঙে **হিসাব করে**। `grep` আর regex আমরা [অধ্যায় ২০](20-grep-regex.md)-এ শিখেছি, `awk` আসছে [অধ্যায় ২২](22-awk.md)-এ।

### কোন sed তোমার কাছে আছে?

লিনাক্সে প্রায় সবসময় **GNU sed** থাকে। macOS আর FreeBSD-তে থাকে **BSD sed**, যেটার কিছু আচরণ আলাদা (বিশেষ করে `-i`)। দেখে নাও:

```bash
$ sed --version
sed (GNU sed) 4.9
Packaged by Debian
Copyright (C) 2022 Free Software Foundation, Inc.
...
```

Ubuntu 24.04 ও Debian 12-এ GNU sed 4.9, RHEL 9-এ 4.8। এই অধ্যায়ের সব উদাহরণ GNU sed ধরে লেখা; যেখানে কিছু শুধু GNU-তে চলে, সেখানে **(GNU)** লিখে দেওয়া আছে।

---

## ২১.২ ভেতরে কীভাবে কাজ করে — pattern space ও cycle

`sed` বুঝতে হলে দুটো "বাক্স" বা বাফার চিনতে হবে:

1. **Pattern space** — কাজের টেবিল। বর্তমান লাইনটা এখানে রাখা হয়, আর সব কমান্ড (যেমন `s`, `d`, `p`) এর ওপরই কাজ করে।
2. **Hold space** — পাশের তাক। শুরুতে খালি। এখানে তুমি ইচ্ছা করে কিছু জমিয়ে রাখতে পারো, এবং এটা এক লাইন থেকে আরেক লাইনে **টিকে থাকে**। (বিস্তারিত ২১.১১-এ)

### এক cycle-এ কী ঘটে

প্রতিটি ইনপুট লাইনের জন্য sed নিচের চক্র (cycle) চালায়:

```
            ইনপুট (ফাইল বা stdin)
                    │
                    ▼
   ┌────────────────────────────────────┐
   │ ১. পরের লাইন পড়ো, শেষের newline    │
   │    (\n) কেটে pattern space-এ রাখো   │
   └────────────────┬───────────────────┘
                    ▼
   ┌────────────────────────────────────┐
   │ ২. script-এর প্রথম কমান্ড থেকে      │
   │    শেষ কমান্ড পর্যন্ত চালাও।        │
   │    প্রতিটি কমান্ডের address মিললে  │      ┌──────────────┐
   │    তবেই কমান্ডটি চলে।               │◄────►│  HOLD SPACE  │
   │    (d চললে → সরাসরি ধাপ ৪-এ, কিছু   │      │ (cycle পেরিয়েও│
   │     প্রিন্ট না করে)                  │      │  টিকে থাকে)  │
   └────────────────┬───────────────────┘      └──────────────┘
                    ▼
   ┌────────────────────────────────────┐
   │ ৩. -n দেওয়া না থাকলে pattern space  │
   │    প্রিন্ট করো + newline যোগ করো    │ ───► stdout (স্ক্রিন)
   └────────────────┬───────────────────┘
                    ▼
   ┌────────────────────────────────────┐
   │ ৪. pattern space খালি করো           │
   │    (hold space অক্ষত থাকে)          │
   └────────────────┬───────────────────┘
                    │
        আরও লাইন আছে? ── হ্যাঁ ──► ধাপ ১-এ ফেরত
                    │
                   না
                    ▼
                  শেষ
```

এই ডায়াগ্রাম থেকে কয়েকটা গুরুত্বপূর্ণ সত্য বেরিয়ে আসে:

- **sed ডিফল্টভাবে সব লাইন প্রিন্ট করে** (ধাপ ৩)। তাই `sed 's/a/b/' file` চালালে যে লাইনে কিছু বদলায়নি সেগুলোও দেখা যায়।
- **`-n` দিলে স্বয়ংক্রিয় প্রিন্ট বন্ধ হয়।** তখন শুধু তুমি `p` দিয়ে যা প্রিন্ট করতে বলবে তাই আসবে।
- **`d` মানে "এই লাইন ফেলে দাও এবং পরের cycle শুরু করো"** — `d`-এর পরের কমান্ডগুলো ওই লাইনের জন্য চলেই না।
- **মূল ফাইল বদলায় না** (যদি না `-i` দাও)। output যায় stdout-এ।
- **sed একবারে পুরো ফাইল মেমরিতে তোলে না**, একটা লাইন রাখে। তাই ১০০ GB ফাইলেও অল্প মেমরিতে চলে।

একটা উদাহরণ দিয়ে দেখি। ফাইল `fruits.txt`:

```bash
$ cat fruits.txt
apple
banana
cherry
```

```bash
$ sed 's/a/A/' fruits.txt
Apple
bAnana
cherry
```

ভেতরে যা হলো:

```
cycle 1: pattern space = "apple"   → s/a/A/ → "Apple"   → প্রিন্ট
cycle 2: pattern space = "banana"  → s/a/A/ → "bAnana"  → প্রিন্ট (শুধু প্রথম a!)
cycle 3: pattern space = "cherry"  → s/a/A/ → মিলল না   → "cherry" প্রিন্ট
```

### GNU `--debug` দিয়ে ভেতরটা দেখো (GNU 4.6+)

```bash
$ printf 'apple\nbanana\n' | sed --debug 's/a/A/'
SED PROGRAM:
  s/a/A/
INPUT:   'STDIN' line 1
PATTERN: apple
COMMAND: s/a/A/
PATTERN: Apple
END-OF-CYCLE:
Apple
INPUT:   'STDIN' line 2
PATTERN: banana
COMMAND: s/a/A/
PATTERN: bAnana
END-OF-CYCLE:
bAnana
```

> 💡 **টিপস:** জটিল sed কমান্ড কেন কাজ করছে না বুঝতে না পারলে `--debug` সবচেয়ে ভালো বন্ধু। এটা প্রতিটি ধাপে pattern space-এ কী আছে দেখিয়ে দেয়।

---

## ২১.৩ sed-এর মৌলিক syntax

```
sed [অপশন] 'script' [ফাইল...]
sed [অপশন] -e 'script1' -e 'script2' [ফাইল...]
sed [অপশন] -f script.sed [ফাইল...]
```

**script**-এর প্রতিটি নির্দেশের গঠন:

```
[address[,address]][!]command[arguments]
   │                  │    │
   │                  │    └── কী করবে (s, d, p, a ...)
   │                  └─────── address উল্টো করো (মিলেনি এমন লাইনে চালাও)
   └────────────────────────── কোন লাইনে করবে (না দিলে = সব লাইনে)
```

উদাহরণ:

| কমান্ড | মানে |
|---|---|
| `sed 'd' f` | সব লাইন মুছো (address নেই → সব লাইন) |
| `sed '3d' f` | ৩ নম্বর লাইন মুছো |
| `sed '/error/d' f` | যে লাইনে `error` আছে তা মুছো |
| `sed '/error/!d' f` | যে লাইনে `error` **নেই** তা মুছো (অর্থাৎ `grep error`-এর মতো) |
| `sed '2,5s/x/y/' f` | ২ থেকে ৫ নম্বর লাইনে x→y |

ইনপুট তিনভাবে দিতে পারো:

```bash
$ sed 's/hi/bye/' file.txt              # ফাইল থেকে
$ echo "hi there" | sed 's/hi/bye/'     # pipe থেকে
bye there
$ sed 's/hi/bye/' < file.txt            # redirection থেকে
```

> ⚠️ **সতর্কতা:** script সবসময় **single quote** (`'...'`) দিয়ে ঘিরে দাও। double quote দিলে shell আগে `$`, `` ` ``, `\`, `!` ইত্যাদি নিজে ব্যাখ্যা করে ফেলবে, ফলে `$` (শেষ লাইন) বা `\1` গোলমাল হবে। শুধু যখন shell variable ঢোকাতেই হবে তখন double quote ব্যবহার করো (২১.১৪ দেখো)।

---

## ২১.৪ গুরুত্বপূর্ণ অপশন

| অপশন | লম্বা রূপ | কাজ |
|---|---|---|
| `-n` | `--quiet`, `--silent` | স্বয়ংক্রিয় প্রিন্ট বন্ধ; শুধু `p` দিয়ে যা চাইবে তা আসবে |
| `-e script` | `--expression=script` | একটা script যোগ করো; একাধিক `-e` দেওয়া যায় |
| `-f file` | `--file=file` | script একটা ফাইল থেকে পড়ো |
| `-E` / `-r` | `--regexp-extended` | Extended Regex (ERE) ব্যবহার করো; `-E` POSIX মানসম্মত, `-r` পুরনো GNU রূপ |
| `-i[SUFFIX]` | `--in-place[=SUFFIX]` | ফাইলকে সরাসরি বদলাও; SUFFIX দিলে ব্যাকআপ রাখো |
| `-s` | `--separate` | একাধিক ফাইলকে আলাদা আলাদা ধরো (`$`, লাইন নম্বর প্রতি ফাইলে নতুন করে) |
| `-z` | `--null-data` | লাইনের বিভাজক newline না ধরে NUL (`\0`) ধরো (GNU) |
| `-u` | `--unbuffered` | output বাফার না করে সাথে সাথে পাঠাও (`tail -f`-এর সাথে কাজে লাগে) |
| `-l N` | `--line-length=N` | `l` কমান্ডের লাইন-মোড়ানোর দৈর্ঘ্য (GNU) |
| | `--follow-symlinks` | `-i`-এর সময় symlink অনুসরণ করে আসল ফাইল বদলাও (GNU) |
| | `--posix` | GNU extension বন্ধ করে খাঁটি POSIX আচরণ |
| | `--debug` | প্রতিটি ধাপ দেখাও (GNU 4.6+) |

### `-n` আর `p` — জুটি

`p` মানে pattern space প্রিন্ট করো। `-n` ছাড়া `p` দিলে মিলে যাওয়া লাইন **দুবার** আসে:

```bash
$ sed '2p' fruits.txt
apple
banana
banana
cherry

$ sed -n '2p' fruits.txt
banana
```

প্রথম ক্ষেত্রে `banana` একবার `p` থেকে, একবার স্বয়ংক্রিয় প্রিন্ট থেকে। তাই নিয়ম: **নির্দিষ্ট লাইন দেখতে চাইলে সবসময় `-n` + `p`।**

### `-e` — একাধিক script

```bash
$ sed -e 's/apple/APPLE/' -e 's/cherry/CHERRY/' fruits.txt
APPLE
banana
CHERRY
```

### `-E` — Extended regex

Basic regex (BRE)-এ group-এর জন্য `\(` `\)`, repetition-এর জন্য `\{2,3\}` লিখতে হয়। `-E` দিলে backslash ছাড়াই লেখা যায়:

```bash
$ echo "2024-06-15" | sed 's/\([0-9]\{4\}\)-\([0-9]\{2\}\)-\([0-9]\{2\}\)/\3\/\2\/\1/'
15/06/2024

$ echo "2024-06-15" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
15/06/2024
```

দুটোর ফল একই, কিন্তু দ্বিতীয়টা পড়া অনেক সহজ। BRE বনাম ERE-র পুরো পার্থক্য [অধ্যায় ২০](20-grep-regex.md)-এ আছে; সংক্ষেপে:

| জিনিস | BRE (ডিফল্ট) | ERE (`-E`) |
|---|---|---|
| group | `\( \)` | `( )` |
| একাধিকবার | `\{n,m\}` | `{n,m}` |
| এক বা বেশি | `\+` (GNU) | `+` |
| শূন্য বা এক | `\?` (GNU) | `?` |
| বিকল্প | `\|` (GNU) | `|` |

### `-i` — ফাইল সরাসরি বদলানো (in-place)

ডিফল্টে sed শুধু স্ক্রিনে দেখায়। ফাইলেই পরিবর্তন রাখতে চাইলে `-i`:

```bash
$ sed -i 's/banana/mango/' fruits.txt
$ cat fruits.txt
apple
mango
cherry
```

ব্যাকআপ রেখে বদলাতে চাইলে `-i`-এর **ঠিক পরেই, ফাঁকা না রেখে** suffix দাও:

```bash
$ sed -i.bak 's/mango/banana/' fruits.txt
$ ls fruits.txt*
fruits.txt  fruits.txt.bak
```

এখন `fruits.txt.bak`-এ আগের (mango-ওয়ালা) সংস্করণ আছে।

#### `-i` ভেতরে কীভাবে কাজ করে?

অনেকে ভাবে `-i` ফাইলের ভেতরে লিখে দেয়। আসলে তা নয়:

```
 ১. sed একই ডিরেক্টরিতে একটা temporary ফাইল বানায় (যেমন ./sedAb12Cd)
 ২. মূল ফাইল পড়ে, প্রক্রিয়া করে, output temp ফাইলে লেখে
 ৩. suffix দেওয়া থাকলে মূল ফাইলকে rename করে  fruits.txt → fruits.txt.bak
 ৪. temp ফাইলকে rename করে মূল নামে আনে  sedAb12Cd → fruits.txt
```

এর কয়েকটা গুরুত্বপূর্ণ ফল আছে (inode-এর ধারণা [অধ্যায় ১১](11-links-inode.md)-এ):

- **ফাইলের inode নম্বর বদলে যায়।** নতুন ফাইল মানে নতুন inode।
- **হার্ড লিংক ভেঙে যায়।** অন্য নামের হার্ড লিংক পুরনো কনটেন্টই দেখাবে।
- **symlink-এর ওপর `-i` চালালে symlink-টা সাধারণ ফাইলে পরিণত হয়**, আসল টার্গেট অপরিবর্তিত থাকে। আসল ফাইল বদলাতে চাইলে `--follow-symlinks` দাও।
- ডিরেক্টরিতে **লেখার পারমিশন** লাগে (temp ফাইল বানানোর জন্য), শুধু ফাইলে লেখার পারমিশন যথেষ্ট নয়।
- GNU sed মালিকানা ও পারমিশন যতটা পারে আগের মতো রাখার চেষ্টা করে।

```bash
$ ls -i fruits.txt
1835012 fruits.txt
$ sed -i 's/a/a/' fruits.txt
$ ls -i fruits.txt
1835019 fruits.txt          # inode বদলে গেছে!
```

> ⚠️ **সতর্কতা — `-i` আর অন্য অপশন একসাথে:** `sed -in 's/x/y/' file` লিখলে GNU sed `n`-কে **suffix** হিসেবে নেয়! ফলে `filen` নামে ব্যাকআপ তৈরি হয় আর `-n` কাজই করে না। তাই `-i`-কে সবসময় আলাদা রাখো: `sed -n -i ...` অথবা `sed -i -n ...`।

> ⚠️ **সতর্কতা — macOS/BSD:** BSD sed-এ `-i`-এর পর suffix **বাধ্যতামূলক আলাদা argument**। ব্যাকআপ না চাইলে খালি string দিতে হয়: `sed -i '' 's/x/y/' file`। GNU-তে `sed -i '' ...` লিখলে `''`-কে script ধরে ভুল হবে। দুই সিস্টেমেই চলবে এমন নিরাপদ রূপ: `sed -i.bak 's/x/y/' file && rm file.bak`।

> 💡 **টিপস:** `-i` চালানোর আগে **সবসময় প্রথমে `-i` ছাড়া চালিয়ে output দেখে নাও।** সন্তুষ্ট হলে তারপর `-i` যোগ করো। আরও নিরাপদ: `sed 's/x/y/' file | diff file -` দিয়ে কী কী বদলাবে দেখে নাও।

### `-s` — একাধিক ফাইল আলাদাভাবে

একাধিক ফাইল দিলে sed ডিফল্টে সবগুলোকে **একটা লম্বা stream** হিসেবে দেখে:

```bash
$ printf 'a1\na2\n' > a.txt; printf 'b1\nb2\n' > b.txt
$ sed -n '1p' a.txt b.txt
a1
$ sed -n '$p' a.txt b.txt
b2
$ sed -s -n '1p' a.txt b.txt
a1
b1
```

`-s` দিলে প্রতিটি ফাইলের প্রথম/শেষ লাইন আলাদাভাবে ধরা হয়। (`-i` দিলে স্বয়ংক্রিয়ভাবে `-s` আচরণ চালু হয়।)

### `-z` — পুরো ফাইলকে এক লাইন হিসেবে (GNU)

`-z` দিলে sed লাইন ভাঙে NUL (`\0`) অক্ষরে। সাধারণ টেক্সট ফাইলে NUL থাকে না, তাই পুরো ফাইল একবারে pattern space-এ আসে — এবং `\n` দিয়ে newline ধরা যায়:

```bash
$ printf 'one\ntwo\nthree\n' | sed -z 's/\n/, /g'
one, two, three, 
```

`find -print0` আর `xargs -0`-এর সাথেও এটা কাজে লাগে।

---

## ২১.৫ s/// — substitute কমান্ড (sed-এর রাজা)

sed-এর ৯০% কাজ এই একটা কমান্ড দিয়েই হয়।

```
s/REGEX/REPLACEMENT/FLAGS
│  │       │          │
│  │       │          └─ g, p, i, নম্বর, w file ...
│  │       └──────────── কী দিয়ে বদলাবে (& , \1 ব্যবহার করা যায়)
│  └──────────────────── কী খুঁজবে (regular expression)
└─────────────────────── substitute
```

```bash
$ echo "I love cats" | sed 's/cats/dogs/'
I love dogs
```

### Flags

| Flag | কাজ | উদাহরণ |
|---|---|---|
| *(কিছু না)* | প্রতি লাইনে **প্রথম** মিলটা বদলাও | `s/a/A/` |
| `g` | লাইনের **সব** মিল বদলাও (global) | `s/a/A/g` |
| `N` (১–৫১২) | শুধু **N-তম** মিল বদলাও | `s/a/A/2` |
| `Ng` | N-তম এবং তার পরের সব মিল (GNU) | `s/a/A/2g` |
| `p` | বদল হলে pattern space প্রিন্ট করো | `sed -n 's/a/A/p'` |
| `i` বা `I` | বড়/ছোট হাতের অক্ষর উপেক্ষা (GNU) | `s/hello/hi/I` |
| `w file` | বদল হওয়া লাইন ফাইলে লেখো | `s/a/A/w out.txt` |
| `m` বা `M` | multiline mode — `^` `$` প্রতিটি লাইনের শুরু/শেষ ধরে (GNU) | `s/^/> /Mg` |
| `e` | ফলাফলকে shell কমান্ড হিসেবে চালাও, output দিয়ে বদলাও (GNU, সাবধান!) | |

উদাহরণগুলো দেখি:

```bash
$ echo "banana" | sed 's/a/A/'
bAnana
$ echo "banana" | sed 's/a/A/g'
bAnAnA
$ echo "banana" | sed 's/a/A/2'
banAna
$ echo "banana" | sed 's/a/A/2g'
banAnA
$ echo "Hello HELLO hello" | sed 's/hello/hi/Ig'
hi hi hi
```

#### `p` flag আর `-n` — "grep + বদল"

শুধু যে লাইনগুলোতে বদল হয়েছে সেগুলো দেখাতে:

```bash
$ cat config.txt
port=80
host=localhost
port_ssl=443
$ sed -n 's/port/PORT/p' config.txt
PORT=80
PORT_ssl=443
```

#### `w` flag

```bash
$ sed 's/port/PORT/w changed.txt' config.txt
PORT=80
host=localhost
PORT_ssl=443
$ cat changed.txt
PORT=80
PORT_ssl=443
```

> 📝 **মনে রাখো:** flag-এর ক্রম গুরুত্বপূর্ণ নয় (`gI` আর `Ig` একই), তবে `w file` সবসময় **শেষে** থাকতে হবে, কারণ এর পরের সবকিছুকে ফাইলের নাম ধরা হয়।

### Delimiter বদলানো

`/` আসলে বাধ্যতামূলক নয়। `s`-এর ঠিক পরের অক্ষরটাই delimiter হয়ে যায়। এটা দারুণ কাজে আসে যখন খোঁজার জিনিসেই `/` থাকে, যেমন ফাইলের path।

`/` দিয়ে লিখলে প্রতিটি `/` escape করতে হয় — এটাকে মজা করে "leaning toothpick syndrome" বলে:

```bash
$ echo "/usr/local/bin" | sed 's/\/usr\/local\/bin/\/opt\/bin/'
/opt/bin
```

অন্য delimiter দিলে অনেক পরিষ্কার:

```bash
$ echo "/usr/local/bin" | sed 's|/usr/local/bin|/opt/bin|'
/opt/bin
$ echo "/usr/local/bin" | sed 's#/usr/local/bin#/opt/bin#'
/opt/bin
$ echo "/usr/local/bin" | sed 's:/usr/local:/opt:'
/opt/bin
```

নিয়ম:
- delimiter যেকোনো একক অক্ষর হতে পারে (newline আর backslash বাদে)। `|`, `#`, `:`, `,`, `@` সবচেয়ে জনপ্রিয়।
- যে অক্ষর delimiter বানাবে, সেটা pattern/replacement-এর ভেতরে লাগলে `\` দিয়ে escape করতে হবে।
- regex **address**-এও delimiter বদলানো যায়, তবে প্রথমটার আগে `\` দিতে হয়: `\|/usr/bin|d` (২১.৬ দেখো)।

### Replacement-এর বিশেষ অক্ষর

replacement অংশ regex নয়, তবে এখানে কয়েকটা অক্ষরের বিশেষ মানে আছে:

| অক্ষর | মানে |
|---|---|
| `&` | পুরো মিলে যাওয়া অংশ |
| `\1` … `\9` | ১ম…৯ম group (`\( \)` বা `-E`-তে `( )`) |
| `\n` | newline (GNU) |
| `\t` | tab (GNU) |
| `\&` | আক্ষরিক `&` |
| `\\` | আক্ষরিক `\` |
| `\U` | এর পর থেকে সব বড় হাতের (GNU) |
| `\L` | এর পর থেকে সব ছোট হাতের (GNU) |
| `\u` | পরের একটা অক্ষর বড় হাতের (GNU) |
| `\l` | পরের একটা অক্ষর ছোট হাতের (GNU) |
| `\E` | `\U`/`\L` বন্ধ করো (GNU) |

এগুলোর বিস্তারিত পরের সেকশনে।

---

## ২১.৬ `&` ও back-reference `\1`

### `&` — পুরো মিল

`&` মানে "যা খুঁজে পেয়েছি ঠিক সেটা"। মিলটাকে মুছে না ফেলে তার চারপাশে কিছু যোগ করতে চাইলে এটা কাজে লাগে:

```bash
$ echo "price 250 taka" | sed 's/[0-9]\+/[&]/'
price [250] taka
$ echo "hello world" | sed 's/[a-z]*/"&"/g'
"hello" "world"
```

আক্ষরিক `&` চাইলে escape করো:

```bash
$ echo "Tom and Jerry" | sed 's/ and / \& /'
Tom & Jerry
```

> ⚠️ **সতর্কতা:** URL-এ প্রায়ই `&` থাকে (`?a=1&b=2`)। replacement-এ URL বসাতে গেলে `&` escape না করলে sed সেখানে মিলে যাওয়া অংশ বসিয়ে দেবে — খুব প্রচলিত ভুল।

### `\1` … `\9` — group ধরা

regex-এর একটা অংশকে group-এ রাখলে (`\(...\)` BRE-তে, `(...)` `-E`-তে), replacement-এ সেটাকে নম্বর দিয়ে ফেরত আনা যায়। বাম দিক থেকে প্রথম খোলা বন্ধনী = `\1`, দ্বিতীয় = `\2`, ইত্যাদি।

```
  echo "Rahim Uddin" | sed -E 's/(\w+) (\w+)/\2, \1/'
                                  ───── ─────
                                    \1    \2
  ফল:  Uddin, Rahim
```

```bash
$ echo "Rahim Uddin" | sed -E 's/(\w+) (\w+)/\2, \1/'
Uddin, Rahim
```

আরও উদাহরণ:

```bash
# key=value কে value=key করো
$ echo "name=karim" | sed -E 's/([^=]+)=(.*)/\2=\1/'
karim=name

# একই শব্দ পরপর দুবার (the the) খুঁজে একবার করো — regex-এর ভেতরেও \1 চলে
$ echo "this is the the book" | sed -E 's/\b(\w+) \1\b/\1/g'
this is the book

# ফোন নম্বর ফরম্যাট
$ echo "01712345678" | sed -E 's/^(0[0-9]{4})([0-9]{6})$/\1-\2/'
01712-345678
```

> 📝 **মনে রাখো:** `\w`, `\b`, `\s` GNU extension; POSIX-নিরাপদ রূপ হলো `[[:alnum:]_]`, `[[:space:]]` ইত্যাদি। BSD sed-এ `\w` নাও চলতে পারে।

### কেস বদলানো: `\U`, `\L`, `\u`, `\l`, `\E` (GNU)

```bash
$ echo "hello world" | sed 's/.*/\U&/'
HELLO WORLD
$ echo "hello world" | sed -E 's/\b(.)/\u\1/g'
Hello World
$ echo "MY_VARIABLE" | sed 's/.*/\L&/'
my_variable
$ echo "name: karim" | sed -E 's/(name: )(.*)/\1\U\2\E!/'
name: KARIM!
```

---

## ২১.৭ Address — কোন লাইনে কাজ হবে

address না দিলে কমান্ড **সব লাইনে** চলে। address দিয়ে নির্দিষ্ট লাইন বেছে নেওয়া যায়। অনুশীলনের জন্য একটা ফাইল বানাই:

```bash
$ seq 1 10 > nums.txt
$ cat nums.txt
1
2
3
...
10
```

### Address-এর ধরন

| Address | মানে | উদাহরণ |
|---|---|---|
| `N` | N নম্বর লাইন | `sed -n '3p'` |
| `$` | শেষ লাইন | `sed -n '$p'` |
| `/regex/` | regex মেলে এমন লাইন | `sed -n '/error/p'` |
| `\%regex%` | অন্য delimiter দিয়ে regex (`%` যেকোনো অক্ষর) | `sed -n '\|/usr|p'` |
| `/regex/I` | কেস উপেক্ষা করে regex (GNU) | `sed -n '/error/Ip'` |
| `A,B` | A থেকে B পর্যন্ত range | `sed -n '3,6p'` |
| `A,$` | A থেকে শেষ পর্যন্ত | `sed -n '8,$p'` |
| `/re1/,/re2/` | re1 মেলা লাইন থেকে re2 মেলা লাইন পর্যন্ত | `sed -n '/BEGIN/,/END/p'` |
| `A,+N` | A আর তার পরের N লাইন (GNU) | `sed -n '/error/,+2p'` |
| `A,~N` | A থেকে পরের N-এর গুণিতক লাইন পর্যন্ত (GNU) | `sed -n '5,~4p'` → ৫–৮ |
| `first~step` | first থেকে শুরু করে প্রতি step-তম লাইন (GNU) | `sed -n '1~2p'` → বিজোড় লাইন |
| `0,/regex/` | একদম প্রথম লাইন থেকেই regex খোঁজা শুরু (GNU) | নিচে দেখো |
| `addr!` | address-এর **উল্টো** | `sed '/^#/!d'` |

উদাহরণ:

```bash
$ sed -n '3p' nums.txt
3
$ sed -n '$p' nums.txt
10
$ sed -n '4,6p' nums.txt
4
5
6
$ sed -n '1~3p' nums.txt
1
4
7
10
$ sed -n '0~5p' nums.txt
5
10
$ sed '2,9d' nums.txt
1
10
$ sed -n '2,9!p' nums.txt
1
10
```

### Range কীভাবে কাজ করে (ভেতরের কথা)

range (`A,B`) একটা "সুইচ"-এর মতো:

```
  লাইন আসছে ──►  [সুইচ বন্ধ]  ── A মিলল? ── হ্যাঁ ──► [সুইচ চালু] (এই লাইনও ধরা)
                                                         │
                         পরের লাইনগুলো ধরা হচ্ছে ◄───────┘
                                                         │
                           B মিলল? ── হ্যাঁ ──► এই লাইন ধরা, তারপর [সুইচ বন্ধ]
                                                         │
                         আবার A খোঁজা শুরু (range বারবার চালু হতে পারে)
```

মজার কিন্তু গুরুত্বপূর্ণ নিয়মগুলো:

1. **end address (B) শুরুর লাইনে পরীক্ষা করা হয় না।** তাই `/x/,/x/` লিখলে একই লাইনে শুরু আর শেষ হয় না; পরের `x`-ওয়ালা লাইন পর্যন্ত যায়।
2. **B যদি লাইন নম্বর হয় এবং সেটা A-র চেয়ে ছোট বা সমান হয়**, তাহলে শুধু A-র একটা লাইনই ধরা হয়। যেমন `sed -n '5,2p'` শুধু লাইন ৫ দেখাবে।
3. **B কখনো না মিললে** range ফাইলের শেষ পর্যন্ত চলে।
4. regex range **একাধিকবার** চালু হতে পারে — ফাইলে যতবার BEGIN…END জোড়া আছে, সবগুলো ধরা পড়বে।

```bash
$ cat block.txt
intro
BEGIN
  data 1
END
middle
BEGIN
  data 2
END
$ sed -n '/BEGIN/,/END/p' block.txt
BEGIN
  data 1
END
BEGIN
  data 2
END
```

### `0,/regex/` — প্রথম মিলটাই শুধু (GNU)

ধরো, ফাইলের **শুধু প্রথম** `foo` বদলাতে চাও (প্রতিটি লাইনের প্রথম নয়, পুরো ফাইলের প্রথম)।

```bash
$ printf 'foo\nfoo\nfoo\n' | sed '0,/foo/s//bar/'
bar
foo
foo
```

`1,/foo/` লিখলে কাজ হতো না যদি প্রথম লাইনেই foo থাকে, কারণ নিয়ম ১ অনুযায়ী end address লাইন ১-এ পরীক্ষাই হয় না — range পরের foo পর্যন্ত চলে যেত। `0,` এই সমস্যার সমাধান। আর লক্ষ করো `s//bar/` — **খালি regex মানে "শেষবার ব্যবহৃত regex"**, অর্থাৎ এখানে `foo`।

### Negation `!`

```bash
# কমেন্ট না এমন সব লাইন মুছো (অর্থাৎ শুধু কমেন্ট দেখাও)
$ sed '/^#/!d' file

# শেষ লাইন ছাড়া সব লাইনের শেষে কমা বসাও
$ printf 'a\nb\nc\n' | sed '$!s/$/,/'
a,
b,
c
```

> 💡 **টিপস:** shell-এ `!` history expansion ঘটায় (interactive bash-এ, double quote-এর ভেতরে)। single quote-এর ভেতরে লিখলে এই সমস্যা হয় না — আরও একটা কারণ single quote ব্যবহারের।

---

## ২১.৮ মৌলিক কমান্ডগুলো

### `d` — delete

pattern space মুছে পরের cycle শুরু করে (স্বয়ংক্রিয় প্রিন্ট হয় না)।

```bash
$ sed '1d' nums.txt            # প্রথম লাইন বাদ (header বাদ দেওয়ার কৌশল)
$ sed '$d' nums.txt            # শেষ লাইন বাদ
$ sed '/^$/d' file             # ফাঁকা লাইন বাদ
$ sed '/^#/d' file             # # দিয়ে শুরু লাইন বাদ
```

### `p` — print

```bash
$ sed -n '/root/p' /etc/passwd
root:x:0:0:root:/root:/bin/bash
```

এটা `grep root /etc/passwd`-এর মতোই।

### `=` — লাইন নম্বর প্রিন্ট

```bash
$ sed -n '/bash$/=' /etc/passwd
1
34
$ sed -n '$=' /etc/passwd          # মোট লাইন সংখ্যা (wc -l এর মতো)
45
```

### `a`, `i`, `c` — append, insert, change

| কমান্ড | কাজ |
|---|---|
| `a text` | লাইনের **পরে** নতুন লাইন যোগ |
| `i text` | লাইনের **আগে** নতুন লাইন যোগ |
| `c text` | লাইনটাকে **পুরো বদলে** text বসাও |

GNU sed এক লাইনে লেখার সুবিধা দেয়:

```bash
$ printf 'one\ntwo\nthree\n' | sed '2a NEW'
one
two
NEW
three
$ printf 'one\ntwo\nthree\n' | sed '2i NEW'
one
NEW
two
three
$ printf 'one\ntwo\nthree\n' | sed '2c NEW'
one
NEW
three
$ printf 'one\ntwo\nthree\n' | sed '$a THE END'
one
two
three
THE END
```

POSIX/BSD-তে `a`, `i`, `c`-এর পর `\` আর newline দিতে হয়:

```bash
sed '2a\
NEW' file
```

একাধিক লাইন যোগ করতে (GNU):

```bash
$ sed '1a line A\nline B' file      # GNU: text-এ \n চলে
```

> 📝 **মনে রাখো:**
> - `a` দিয়ে যোগ করা টেক্সট pattern space-এ যায় না; এটা একটা queue-তে থাকে এবং cycle-এর শেষে (বা পরের লাইন পড়ার সময়) output-এ যায়। তাই পরের কোনো `s` কমান্ড এই টেক্সট বদলাতে পারে না।
> - `i` এর টেক্সট সাথে সাথে output-এ চলে যায়।
> - `a`-এর পর text-এর শুরুর ফাঁকা জায়গা GNU sed কেটে ফেলে। শুরুতে স্পেস রাখতে চাইলে `a\` দিয়ে লেখো: `sed '2a\    indented'`।
> - **range-এর সাথে `c`:** GNU sed-এ `2,4c TEXT` পুরো range মুছে **একবার** TEXT বসায় (range-এর শেষে)। কিন্তু `!` থাকলে প্রতিটি লাইনে বসায়।

```bash
$ seq 5 | sed '2,4c ---removed---'
1
---removed---
5
```

### `y` — অক্ষর অনুবাদ (transliterate)

`y/abc/xyz/` মানে a→x, b→y, c→z। দুই পাশে **সমান সংখ্যক** অক্ষর থাকতে হবে। এটা regex নয়, একটা একটা অক্ষর মেলায় — অনেকটা `tr`-এর মতো ([অধ্যায় ২৩](23-text-tools.md))।

```bash
$ echo "hello" | sed 'y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/'
HELLO
$ echo "a-b-c" | sed 'y/-/_/'
a_b_c
```

> ⚠️ **সতর্কতা:** `y`-তে range (`a-z`) চলে না — `y/a-z/A-Z/` শুধু a→A, -→-, z→Z করবে। সব অক্ষর লিখে দিতে হয়।

### `q` ও `Q` — quit

`q` বর্তমান লাইন প্রিন্ট করে বেরিয়ে যায়; `Q` (GNU) প্রিন্ট না করেই বেরোয়। বিশাল ফাইলে দ্রুত কাজ শেষ করার জন্য এটা দারুণ — বাকি ফাইল আর পড়া হয় না।

```bash
$ sed '3q' nums.txt          # head -n 3 এর মতো
1
2
3
$ sed '3Q' nums.txt
1
2
$ sed -n '1000000{p;q}' huge.log   # ১০ লাখতম লাইন দেখে সাথে সাথে বেরোও
```

GNU-তে exit code-ও দেওয়া যায়: `sed '/ERROR/q 5' file; echo $?` — ERROR পেলে exit code 5। স্ক্রিপ্টে কাজে লাগে।

### `l` — অদৃশ্য অক্ষর দেখাও

```bash
$ printf 'a\tb  \r\n' | sed -n 'l'
a\tb  \r$
```

tab (`\t`), Windows-এর carriage return (`\r`) আর লাইনের শেষ (`$`) দেখা যাচ্ছে — trailing space বা CRLF সমস্যা ধরতে চমৎকার।

### `r` ও `w` — ফাইল পড়া ও লেখা

```bash
$ sed '/<!-- footer -->/r footer.html' page.html   # মিলে যাওয়া লাইনের পরে footer.html ঢোকাও
$ sed -n '/ERROR/w errors.log' app.log             # ERROR লাইনগুলো errors.log-এ লেখো
```

### `n` ও `N` — পরের লাইন

| কমান্ড | কাজ |
|---|---|
| `n` | (`-n` না থাকলে) বর্তমান pattern space প্রিন্ট করে, তারপর পরের লাইন দিয়ে **প্রতিস্থাপন** করে |
| `N` | পরের লাইনকে `\n` দিয়ে জুড়ে pattern space-এ **যোগ** করে (এখন pattern space-এ দুই লাইন) |
| `P` | pattern space-এর প্রথম `\n` পর্যন্ত প্রিন্ট |
| `D` | pattern space-এর প্রথম `\n` পর্যন্ত মুছে, নতুন লাইন না পড়ে cycle আবার শুরু |

```bash
# প্রতি জোড় লাইন মুছো: n দিয়ে বিজোড়টা প্রিন্ট, পরেরটা নিয়ে d
$ seq 6 | sed 'n;d'
1
3
5

# দুই লাইন করে জুড়ে দাও
$ seq 6 | sed 'N;s/\n/ /'
1 2
3 4
5 6
```

> 📝 **মনে রাখো:** GNU sed-এ শেষ লাইনে `N` চললে (পরের লাইন নেই) pattern space প্রিন্ট করে বেরিয়ে যায়। POSIX অনুযায়ী প্রিন্ট না করেই বেরোনোর কথা — তাই বহনযোগ্য স্ক্রিপ্টে `$!N` লেখা নিরাপদ।

### `b`, `t`, `T` ও label — শাখা (একটু অ্যাডভান্সড)

sed-এ ছোট্ট "goto" আছে:

| কমান্ড | কাজ |
|---|---|
| `:label` | একটা জায়গার নাম দাও |
| `b label` | label-এ লাফ দাও (label না দিলে script-এর শেষে) |
| `t label` | শেষ ইনপুট পড়ার পর কোনো `s` সফল হলে লাফ দাও |
| `T label` | কোনো `s` সফল **না** হলে লাফ দাও (GNU) |

বিখ্যাত উদাহরণ — পুরো ফাইলের সব লাইন এক লাইনে জোড়া:

```bash
$ seq 5 | sed ':a;N;$!ba;s/\n/,/g'
1,2,3,4,5
```

ভেঙে দেখি:

```
:a        → এখানে label "a"
N         → পরের লাইন জুড়ে দাও
$!ba      → শেষ লাইন না হলে a-তে ফেরত যাও (লুপ)
s/\n/,/g  → শেষে সব newline কে কমা বানাও
```

(GNU-তে আজকাল সহজ রূপ: `sed -z 's/\n/,/g'`, তবে শেষ newline-ও কমা হয়ে যাবে।)

সংখ্যায় হাজারের কমা বসানো (`t` দিয়ে লুপ):

```bash
$ echo "1234567" | sed -E ':a;s/([0-9])([0-9]{3})($|,)/\1,\2\3/;ta'
1,234,567
```

---

## ২১.৯ একাধিক কমান্ড একসাথে

চারটা উপায় আছে:

```bash
# ১. সেমিকোলন
$ sed 's/a/A/;s/b/B/' file

# ২. একাধিক -e
$ sed -e 's/a/A/' -e 's/b/B/' file

# ৩. quote-এর ভেতরে newline
$ sed '
s/a/A/
s/b/B/
' file

# ৪. script ফাইল
$ sed -f fix.sed file
```

### `{ }` — একটা address-এ কয়েকটা কমান্ড

```bash
# যে লাইনে "server" আছে, শুধু সেখানে দুটো বদল করো
$ sed '/server/{s/http/https/;s/80/443/}' config

# ৫ নম্বর লাইন প্রিন্ট করে বেরিয়ে যাও
$ sed -n '5{p;q}' file
```

address-এর ভেতরে address (nested) — শুধু `[database]` সেকশনের ভেতরে `port` বদলানো:

```bash
$ cat app.ini
[web]
port=80
[database]
port=3306
[cache]
port=6379
$ sed '/^\[database\]/,/^\[/{s/^port=.*/port=5432/}' app.ini
[web]
port=80
[database]
port=5432
[cache]
port=6379
```

### sed script ফাইল

জটিল কাজ হলে ফাইলে লেখো — কমেন্টও দেওয়া যায় (`#`):

```bash
$ cat cleanup.sed
# কমেন্ট লাইন মুছো
/^[[:space:]]*#/d
# ফাঁকা লাইন মুছো
/^[[:space:]]*$/d
# লাইনের শেষের অপ্রয়োজনীয় স্পেস কাটো
s/[[:space:]]*$//

$ sed -f cleanup.sed /etc/ssh/sshd_config
```

ফাইলটাকে সরাসরি চালানোর যোগ্যও বানানো যায়, প্রথম লাইনে shebang দিয়ে (shebang ও স্ক্রিপ্ট [অধ্যায় ৪১](41-shell-scripting-1.md)-এ):

```bash
#!/usr/bin/sed -f
s/foo/bar/g
```

```bash
$ chmod +x my.sed
$ ./my.sed input.txt
```

> ⚠️ **সতর্কতা:** `a`, `i`, `c`, `r`, `w`, `:label` — এদের পর সেমিকোলন দিয়ে আরেকটা কমান্ড লিখলে সেটা text বা ফাইলের নামের অংশ হয়ে যায়। যেমন `sed '2a hello; 3d'` লাইন ৩ মুছবে না, বরং `hello; 3d` টেক্সট যোগ করবে। এদের পর নতুন `-e` বা newline ব্যবহার করো: `sed -e '2a hello' -e '3d'`।

---

## ২১.১০ রেগুলার এক্সপ্রেশন: sed-এ যা বিশেষভাবে জানতে হবে

regex-এর পুরো আলোচনা [অধ্যায় ২০](20-grep-regex.md)-এ। sed-এর জন্য বাড়তি কথা:

- **sed সবসময় সবচেয়ে বাম দিকের, সবচেয়ে লম্বা (greedy) মিল নেয়।** `.*` যতটা সম্ভব খেয়ে ফেলে।

```bash
$ echo '<b>bold</b> and <i>italic</i>' | sed 's/<.*>//g'

$ echo '<b>bold</b> and <i>italic</i>' | sed 's/<[^>]*>//g'
bold and italic
```

প্রথমটা পুরো লাইন মুছে দিল কারণ `<.*>` প্রথম `<` থেকে শেষ `>` পর্যন্ত ধরেছে। sed-এ non-greedy `.*?` **নেই**; তার বদলে "নেগেটিভ ক্লাস" `[^>]*` ব্যবহার করো।

- **pattern space-এ শেষের newline থাকে না**, তাই `$` লাইনের শেষ বোঝায় এবং `\n` দিয়ে সাধারণ লাইনে কিছু মেলে না (যদি না `N` দিয়ে লাইন জোড়া হয়)।
- **খালি regex `//`** মানে শেষ ব্যবহৃত regex: `sed '/error/s//ERROR/g'`।
- character class: `[[:digit:]]`, `[[:alpha:]]`, `[[:space:]]`, `[[:upper:]]` — locale-নিরাপদ।

> 💡 **টিপস:** বিশেষ অক্ষর (`.`, `*`, `[`, `]`, `^`, `$`, `\`, `/`) আক্ষরিকভাবে খুঁজতে চাইলে `\` দাও। যেমন IP-তে `.` খুঁজতে: `s/192\.168\.1\.1/10.0.0.1/` — `\.` না দিলে `.` যেকোনো অক্ষর মেলাবে, ফলে `192x168` ও মিলে যেত।

---

## ২১.১১ Hold space পরিচিতি

এতক্ষণ আমরা শুধু pattern space ব্যবহার করেছি, যা প্রতিটি cycle শেষে খালি হয়ে যায়। কিন্তু কখনো আগের লাইনের তথ্য মনে রাখা দরকার হয় — তখন **hold space**।

> 💡 **উপমা:** pattern space হলো তোমার হাত, hold space হলো তোমার পকেট। প্রতিটা বাক্স হাতে নিয়ে কাজ করো, কাজ শেষে হাত খালি হয়ে যায়। কিন্তু পকেটে যা রাখো তা পরের বাক্সের সময়েও থাকে।

| কমান্ড | মনে রাখার কৌশল | কাজ |
|---|---|---|
| `h` | **h**old | pattern → hold (hold-এর পুরনো জিনিস মুছে **কপি**) |
| `H` | **H**old append | pattern → hold-এ **যোগ** (মাঝে `\n` দিয়ে) |
| `g` | **g**et | hold → pattern (pattern-এর জিনিস মুছে কপি) |
| `G` | **G**et append | hold → pattern-এ **যোগ** (মাঝে `\n`) |
| `x` | e**x**change | pattern আর hold **অদলবদল** |

```
          h (কপি) / H (যোগ)
   ┌──────────────┐ ───────────────────► ┌──────────────┐
   │ PATTERN SPACE│                      │  HOLD SPACE  │
   │   (হাত)      │ ◄─────────────────── │   (পকেট)     │
   └──────────────┘   g (কপি) / G (যোগ)   └──────────────┘
          ▲                                      ▲
          └─────────────── x (অদলবদল) ───────────┘
```

### উদাহরণ ১: প্রতি লাইনের পর ফাঁকা লাইন (double spacing)

hold space শুরুতে খালি। `G` প্রতিটি লাইনের শেষে `\n` + (খালি hold) যোগ করে:

```bash
$ printf 'a\nb\nc\n' | sed G
a

b

c

```

### উদাহরণ ২: ফাইল উল্টো করা (`tac`-এর মতো)

```bash
$ printf 'one\ntwo\nthree\n' | sed -n '1!G;h;$p'
three
two
one
```

ধাপে ধাপে:

```
লাইন  কমান্ড        PATTERN                  HOLD
 1    (1!G বাদ)      one                      ""
      h              one                      one
 2    G              two\none                 one
      h              two\none                 two\none
 3    G              three\ntwo\none          two\none
      h              three\ntwo\none          three\ntwo\none
      $p             → প্রিন্ট: three / two / one
```

### উদাহরণ ৩: কোনো লাইনের আগের লাইন দেখাও

```bash
$ printf 'user=ali\nstatus=FAIL\nuser=sumi\nstatus=OK\n' | sed -n '/FAIL/{x;p;x;p};h'
user=ali
status=FAIL
```

এখানে প্রতিটি লাইন `h` দিয়ে পকেটে রাখা হচ্ছে; FAIL পেলে `x` দিয়ে আগের লাইন বের করে প্রিন্ট, আবার `x` দিয়ে বর্তমানটা ফিরিয়ে প্রিন্ট।

> 📝 **মনে রাখো:** hold space শক্তিশালী, কিন্তু জটিল যুক্তির কাজ হলে `awk` ([অধ্যায় ২২](22-awk.md)) সাধারণত বেশি পরিষ্কার। sed-এ ৩–৪ লাইনের বেশি hold-space কৌশল লাগলে awk ভেবে দেখো।

---

## ২১.১২ বাস্তব রেসিপি — রোজকার কাজের sed

### ক. ফাঁকা লাইন ও স্পেস পরিষ্কার

```bash
sed '/^$/d' file                    # সম্পূর্ণ ফাঁকা লাইন মুছো
sed '/^[[:space:]]*$/d' file        # শুধু স্পেস/ট্যাব আছে এমন লাইনও মুছো
sed 's/^[[:space:]]*//' file        # লাইনের শুরুর স্পেস কাটো (ltrim)
sed 's/[[:space:]]*$//' file        # লাইনের শেষের স্পেস কাটো (rtrim)
sed 's/^[[:space:]]*//;s/[[:space:]]*$//' file   # দুই দিকেই
sed 's/  */ /g' file                # একাধিক স্পেসকে এক স্পেস
sed '/^$/N;/^\n$/D' file            # পরপর অনেক ফাঁকা লাইনকে একটায় নামাও (cat -s এর মতো)
```

### খ. কমেন্ট বাদ দিয়ে আসল কনফিগ দেখা

সার্ভারের কনফিগ ফাইলে শত শত লাইন কমেন্ট থাকে। আসল সেটিং দেখতে:

```bash
$ sed -e '/^[[:space:]]*#/d' -e '/^[[:space:]]*$/d' /etc/ssh/sshd_config
Include /etc/ssh/sshd_config.d/*.conf
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem	sftp	/usr/lib/openssh/sftp-server
```

(output Ubuntu 24.04-এর ডিফল্ট ফাইলের মতো; তোমার সিস্টেমে ভিন্ন হতে পারে।)

### গ. কনফিগ ফাইল এডিট করা

**একটা সেটিং বদলানো** (কমেন্ট করা থাকুক বা না থাকুক):

```bash
# "#PermitRootLogin prohibit-password" বা "PermitRootLogin yes" — যা-ই থাকুক → "PermitRootLogin no"
$ sudo sed -i.bak -E 's/^#?[[:space:]]*PermitRootLogin[[:space:]].*/PermitRootLogin no/' /etc/ssh/sshd_config
$ grep -n '^PermitRootLogin' /etc/ssh/sshd_config
33:PermitRootLogin no
```

তারপর অবশ্যই কনফিগ যাচাই করে সার্ভিস reload করো (SSH নিয়ে বিস্তারিত [অধ্যায় ৩৮](38-ssh-remote.md), systemd [অধ্যায় ২৬](26-systemd-services.md)-এ):

```bash
$ sudo sshd -t && sudo systemctl reload ssh      # RHEL/Fedora-তে সার্ভিসের নাম sshd
```

**একটা লাইন কমেন্ট করা / আন-কমেন্ট করা:**

```bash
sed -i '/^Listen 8080/s/^/# /' httpd.conf      # কমেন্ট করো
sed -i 's/^#[[:space:]]*\(Listen 8080\)/\1/' httpd.conf   # কমেন্ট সরাও
```

**key=value মান বদলানো:**

```bash
$ cat .env
APP_ENV=local
APP_DEBUG=true
DB_HOST=127.0.0.1
$ sed -i 's/^APP_DEBUG=.*/APP_DEBUG=false/' .env
$ sed -i 's|^DB_HOST=.*|DB_HOST=db.internal|' .env
$ cat .env
APP_ENV=local
APP_DEBUG=false
DB_HOST=db.internal
```

**লাইন না থাকলে যোগ করো, থাকলে বদলাও** (idempotent — বারবার চালালেও ফল একই):

```bash
grep -q '^MAX_CONN=' app.conf \
  && sed -i 's/^MAX_CONN=.*/MAX_CONN=200/' app.conf \
  || echo 'MAX_CONN=200' >> app.conf
```

**নির্দিষ্ট লাইনের পরে নতুন সেটিং যোগ:**

```bash
sed -i '/^\[mysqld\]/a max_connections = 500' /etc/mysql/my.cnf
```

### ঘ. লাইন বের করা (বড় ফাইল থেকে)

```bash
sed -n '100,200p' big.log           # ১০০–২০০ লাইন
sed -n '100,200p;201q' big.log      # একই, কিন্তু ২০১-এ থেমে যায় → অনেক দ্রুত
sed -n '/10:00:00/,/10:05:00/p' app.log   # দুটো সময়ের মাঝের লগ
sed -n '$p' file                    # শেষ লাইন (tail -n 1)
sed '1d' data.csv                   # CSV-এর header বাদ
```

### ঙ. টেক্সট রূপান্তর

```bash
sed 's/\r$//' winfile.txt                    # Windows CRLF → Linux LF (dos2unix-এর মতো)
sed 's/$/\r/' unixfile.txt                   # উল্টো: LF → CRLF (GNU)
sed 's/^/    /' code.txt                      # প্রতি লাইনের শুরুতে ৪ স্পেস (indent)
sed 's/^/> /' email.txt                       # ইমেইলে quote করা
sed = file | sed 'N;s/\n/\t/'                 # লাইন নম্বর বসাও (cat -n এর মতো)
sed 's/,/\t/g' data.csv                       # CSV → TSV (সাধারণ CSV-র জন্য)
sed -E 's/([0-9]{1,3}\.){3}[0-9]{1,3}/x.x.x.x/g' access.log   # IP লুকাও
```

### চ. একসাথে অনেক ফাইলে বদল

```bash
# প্রজেক্টের সব .php ফাইলে পুরনো ডোমেইন বদলাও (আগে দেখো, তারপর -i)
$ grep -rl 'old-domain.com' --include='*.php' . | xargs sed -i 's/old-domain\.com/new-domain.com/g'

# ফাইলের নামে স্পেস থাকলে নিরাপদ রূপ
$ grep -rlZ 'old-domain.com' --include='*.php' . | xargs -0 sed -i 's/old-domain\.com/new-domain.com/g'

# find দিয়ে
$ find . -name '*.conf' -exec sed -i.bak 's/foo/bar/g' {} +
```

(`xargs` আর `find -exec` বিস্তারিত [অধ্যায় ১২](12-file-khoja.md) ও [অধ্যায় ২৩](23-text-tools.md)-এ।)

> ⚠️ **সতর্কতা:** অনেক ফাইলে একসাথে `-i` চালানো খুব শক্তিশালী — এবং খুব বিপজ্জনক। ভুল regex দিলে শত ফাইল একসাথে নষ্ট হবে। git রিপোজিটরিতে থাকলে আগে `git status` পরিষ্কার রাখো যাতে `git diff` দিয়ে দেখতে পারো, আর `git checkout -- .` দিয়ে ফেরত যেতে পারো। না হলে `-i.bak` ব্যবহার করো।

### ছ. pipe-এর মাঝে sed

```bash
$ ip -4 addr show | sed -n 's/.*inet \([0-9.]*\).*/\1/p'
127.0.0.1
192.168.1.105

$ echo "$PATH" | sed 's/:/\n/g'
/usr/local/sbin
/usr/local/bin
/usr/sbin
/usr/bin
...

$ tail -f /var/log/syslog | sed -u 's/error/\x1b[31mERROR\x1b[0m/Ig'   # লাইভ লগে error লাল রঙে
```

(শেষটায় `-u` দিয়েছি যাতে output সাথে সাথে দেখা যায়; `\x1b` GNU-র escape।)

---

## ২১.১৩ Shell variable ব্যবহার করা

স্ক্রিপ্টে প্রায়ই মান variable থেকে আসে। তখন double quote লাগে:

```bash
$ name="Karim"
$ echo "Hello USER" | sed "s/USER/$name/"
Hello Karim
```

সমস্যা হয় যখন variable-এ delimiter বা বিশেষ অক্ষর থাকে:

```bash
$ dir="/var/www/html"
$ echo "root=X" | sed "s/X/$dir/"
sed: -e expression #1, char 9: unknown option to `s'
$ echo "root=X" | sed "s|X|$dir|"
root=/var/www/html
```

> ⚠️ **সতর্কতা:** variable-এ `&`, `\` বা delimiter থাকলে ফল ভুল হবে বা error আসবে। ইউজারের দেওয়া অজানা টেক্সট replacement-এ বসাতে হলে আগে escape করো:
> ```bash
> safe=$(printf '%s' "$value" | sed 's/[&/\]/\\&/g')
> sed "s/PLACEHOLDER/$safe/" template.txt
> ```

single আর double quote মিশিয়েও লেখা যায় — শুধু variable অংশ double quote-এ:

```bash
$ sed 's/^port=.*/port='"$PORT"'/' app.conf
```

---

## ২১.১৪ GNU sed বনাম BSD (macOS) sed

| বিষয় | GNU sed (Linux) | BSD sed (macOS/FreeBSD) |
|---|---|---|
| ব্যাকআপ ছাড়া in-place | `sed -i 's/a/b/' f` | `sed -i '' 's/a/b/' f` |
| ERE | `-E` বা `-r` | `-E` |
| `\n`, `\t` replacement-এ | চলে | সাধারণত চলে না |
| `\U \L \u \l` | চলে | চলে না |
| `I` flag / `/re/I` | চলে | চলে না (পুরনো সংস্করণে) |
| `a text` এক লাইনে | চলে | `a\` + newline লাগে |
| `0,/re/`, `first~step`, `addr,+N` | চলে | চলে না |
| `Q`, `T`, `e`, `--debug`, `-z` | চলে | চলে না |

> 💡 **টিপস:** macOS-এ GNU sed চাইলে `brew install gnu-sed` করো; এটা `gsed` নামে আসে। পোর্টেবল স্ক্রিপ্ট লিখতে চাইলে Linux-এ `sed --posix` দিয়ে পরীক্ষা করে নিতে পারো।

---

## ২১.১৫ সাধারণ ভুল ও সমাধান

| ভুল | কী হয় | ঠিক রূপ |
|---|---|---|
| `sed 's/a/b/' f > f` | shell আগে `f` খালি করে দেয় → **ফাইল শূন্য!** | `sed -i 's/a/b/' f` |
| `sed -in ...` | `n` suffix হয়ে যায় | `sed -n -i ...` |
| `-n` ছাড়া `p` | লাইন দুবার আসে | `sed -n '...p'` |
| `g` ভুলে যাওয়া | প্রতি লাইনে শুধু প্রথম মিল বদলায় | `s/x/y/g` |
| path-এ `/` | `unknown option to 's'` | `s|...|...|` |
| `.` escape না করা | `1.2` এর জায়গায় `1x2`-ও মেলে | `1\.2` |
| `.*` greedy | অপ্রত্যাশিতভাবে বেশি মুছে যায় | `[^X]*` |
| BRE-তে `+`, `(` | আক্ষরিক অক্ষর হিসেবে ধরা হয় | `-E` দাও বা `\+`, `\(` |
| double quote-এ `$` | shell variable ভেবে বদলে দেয় | single quote |
| CRLF ফাইল | `$` দিয়ে লাইনের শেষ মেলে না | আগে `s/\r$//` |

প্রথম ভুলটা সবচেয়ে বিপজ্জনক, তাই আবার বলি:

```
  sed 's/a/b/' file > file
                    ───────
                       │
  shell কমান্ড চালানোর আগেই redirection সেট করে → file কে 0 বাইট করে ফেলে
  → তারপর sed খালি ফাইল পড়ে → কিছুই লেখে না → সব ডেটা হারালে!
```

(redirection কীভাবে কাজ করে তা [অধ্যায় ১৯](19-redirection-pipe.md)-এ আছে।)

---

## ২১.১৬ সম্পর্কিত কীবোর্ড শর্টকাট

sed নিজে কোনো interactive প্রোগ্রাম নয়, তাই এর নিজস্ব শর্টকাট নেই। তবে sed নিয়ে কাজ করার সময় এই টার্মিনাল শর্টকাটগুলো বারবার লাগে (পুরো তালিকা [অধ্যায় ৬](06-terminal-shortcuts.md)-এ):

| শর্টকাট | কাজ | sed-এর সাথে কোথায় কাজে লাগে |
|---|---|---|
| `Ctrl + D` | stdin শেষ (EOF) পাঠাও | ফাইল ছাড়া `sed 's/a/b/'` চালিয়ে হাতে টাইপ করে পরীক্ষা শেষে |
| `Ctrl + C` | চলমান কমান্ড থামাও | ভুলে ফাইল না দিলে sed stdin-এর অপেক্ষায় আটকে থাকে — তখন বেরোতে |
| `↑` / `Ctrl + P` | আগের কমান্ড | `-i` ছাড়া পরীক্ষা করে, ↑ চেপে `-i` যোগ করো |
| `Ctrl + R` | history-তে খোঁজা | আগে লেখা জটিল sed খুঁজে বের করা |
| `Alt + .` | আগের কমান্ডের শেষ argument | `sed ... file` এর পর `cat Alt+.` |
| `Ctrl + A` / `Ctrl + E` | লাইনের শুরু / শেষে যাও | লম্বা sed কমান্ড এডিট |
| `Ctrl + X Ctrl + E` | কমান্ডটা এডিটরে খোলো (bash) | বহু-লাইনের sed script লিখতে |
| `Ctrl + W` | আগের শব্দ মুছো | |
| `!!` | আগের পুরো কমান্ড | `sudo !!` — পারমিশন না থাকায় ব্যর্থ হলে |
| `^old^new` | আগের কমান্ডে old→new বদলে চালাও | নিজেই এক ছোট্ট sed! যেমন `^s/a/s/b` |

> 💡 **টিপস:** Vim-এর `:%s/old/new/g` আর sed-এর `s/old/new/g` একই বংশের। sed শিখলে vim-এর search-replace ([অধ্যায় ১৪](14-vim.md)) এমনিতেই শেখা হয়ে যায়, উল্টোটাও সত্যি।

---

## হাতে-কলমে অনুশীলন

একটা আলাদা ফোল্ডারে কাজ করো যাতে কিছু নষ্ট না হয়।

### ল্যাব ১: প্রস্তুতি

```bash
$ mkdir -p ~/sed-lab && cd ~/sed-lab
$ cat > students.txt << 'EOF'
# Student list 2024
id,name,dept,cgpa

1,rahim uddin,cse,3.75
2,karim ahmed,eee,3.20
3,sumi akter,cse,3.90

4,nila das,bba,2.95
5,tanvir hasan,cse,3.10
EOF
```

### ল্যাব ২: দেখা ও বের করা

1. শুধু ৩ নম্বর লাইন দেখাও: `sed -n '3p' students.txt`
2. মোট লাইন গোনো: `sed -n '$=' students.txt` → `9`
3. `cse` ছাত্রদের দেখাও: `sed -n '/,cse,/p' students.txt`
4. কমেন্ট ও ফাঁকা লাইন বাদ দাও: `sed '/^#/d;/^$/d' students.txt`
5. header-সহ প্রথম তিনজন: `sed -n '2,6p' students.txt` — দেখো ফাঁকা লাইনও এসেছে; এবার `sed -n '2,6{/^$/!p}' students.txt` চালাও।

### ল্যাব ৩: substitute

1. `cse` কে `CSE` করো: `sed 's/,cse,/,CSE,/' students.txt`
2. নামের প্রতিটি শব্দের প্রথম অক্ষর বড় হাতের করো:
   ```bash
   $ sed -E '/^[0-9]/s/,([a-z]+) ([a-z]+),/,\u\1 \u\2,/' students.txt
   ```
   প্রত্যাশিত: `1,Rahim Uddin,cse,3.75` ইত্যাদি।
3. কমা বদলে ` | ` দাও: `sed 's/,/ | /g' students.txt`
4. শুধু দ্বিতীয় কমা বদলাও: `sed 's/,/;/2' students.txt` — কী হলো ব্যাখ্যা করো।
5. cgpa-কে বন্ধনীতে রাখো `&` দিয়ে: `sed -E 's/[0-9]\.[0-9]{2}$/(&)/' students.txt`
6. নাম আর dept-এর জায়গা বদলাও:
   ```bash
   $ sed -E 's/^([0-9]+),([^,]+),([^,]+),/\1,\3,\2,/' students.txt
   ```

### ল্যাব ৪: a, i, c, d, y, q

```bash
$ sed '/^id,/i # ---- data start ----' students.txt
$ sed '$a 6,new student,mat,3.00' students.txt
$ sed '/nila/c 4,REMOVED,-,-' students.txt
$ sed '/^[0-9]/y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/' students.txt
$ sed '/sumi/q' students.txt        # sumi-তে পৌঁছে থামো
```

### ল্যাব ৫: in-place ও ব্যাকআপ

```bash
$ cp students.txt work.txt
$ ls -i work.txt                                      # inode নোট করো
$ sed -i.orig '/^$/d;/^#/d' work.txt
$ ls -i work.txt work.txt.orig                         # inode বদলেছে? .orig-এ কী আছে?
$ diff work.txt.orig work.txt
```

### ল্যাব ৬: কনফিগ ফাইল

```bash
$ cat > app.ini << 'EOF'
[server]
#port = 8080
host = 0.0.0.0
[database]
port = 3306
user = root
EOF
```

1. `[server]` সেকশনের কমেন্ট করা port আন-কমেন্ট করে 9090 বানাও:
   ```bash
   $ sed -E '/^\[server\]/,/^\[/{s/^#?port = .*/port = 9090/}' app.ini
   ```
2. শুধু `[database]` সেকশনের `user` বদলে `app` করো (নিজে লেখো)।
3. `[database]`-এর পরে `password = secret` যোগ করো।
4. সব ঠিক থাকলে `-i.bak` দিয়ে ফাইলে লেখো।

### ল্যাব ৭: hold space

```bash
$ seq 5 | sed -n '1!G;h;$p'           # উল্টো
$ seq 5 | sed G                        # double space
$ seq 5 | sed ':a;N;$!ba;s/\n/+/g'     # 1+2+3+4+5
$ seq 5 | sed ':a;N;$!ba;s/\n/+/g' | bc   # যোগফল 15
```

### চ্যালেঞ্জ

- `/etc/passwd` থেকে শুধু ইউজারনেম আর shell দেখাও, `:` দিয়ে: (ইঙ্গিত: `sed -E 's/^([^:]*):.*:([^:]*)$/\1:\2/'`)
- একটা HTML ফাইল থেকে সব ট্যাগ মুছে শুধু লেখা রাখো।
- একটা লগ ফাইলে ঘণ্টা `HH:MM:SS` সময়কে `[HH:MM]` রূপে দেখাও।

---

## সারসংক্ষেপ

| কমান্ড/অপশন | কাজ |
|---|---|
| `sed 's/a/b/' f` | প্রতি লাইনে প্রথম a→b |
| `sed 's/a/b/g' f` | সব a→b |
| `sed 's/a/b/2' f` | দ্বিতীয় a→b |
| `sed 's/a/b/I' f` | কেস উপেক্ষা (GNU) |
| `sed -n 's/a/b/p' f` | শুধু বদলানো লাইন দেখাও |
| `sed 's|/x|/y|' f` | delimiter বদল |
| `sed -n '5p' f` | ৫ নম্বর লাইন |
| `sed -n '5,10p' f` | ৫–১০ লাইন |
| `sed -n '$p' f` | শেষ লাইন |
| `sed -n '/re/p' f` | regex মেলা লাইন |
| `sed '/re/d' f` | regex মেলা লাইন মুছো |
| `sed '/re/!d' f` | regex না মেলা লাইন মুছো |
| `sed '/^$/d' f` | ফাঁকা লাইন মুছো |
| `sed '3a text' f` | ৩-এর পরে লাইন যোগ |
| `sed '3i text' f` | ৩-এর আগে লাইন যোগ |
| `sed '3c text' f` | ৩ নম্বর লাইন বদলাও |
| `sed 'y/abc/xyz/' f` | অক্ষর অনুবাদ |
| `sed '10q' f` | ১০ লাইন দেখিয়ে থামো |
| `sed -n '$=' f` | লাইন সংখ্যা |
| `sed -E '...'` | extended regex |
| `sed -e '..' -e '..'` | একাধিক কমান্ড |
| `sed -f x.sed f` | script ফাইল থেকে |
| `sed -i '...' f` | ফাইলেই বদল |
| `sed -i.bak '...' f` | ব্যাকআপ রেখে বদল |
| `&` | পুরো মিল |
| `\1`..`\9` | group |
| `\U \L \u \l \E` | কেস বদল (GNU) |
| `h H g G x` | hold space কমান্ড |
| `n N P D` | পরের লাইন নিয়ে কাজ |
| `:a ... ba`, `t` | লুপ/শাখা |
| `--debug` | ধাপে ধাপে দেখো (GNU) |

---

## নিজেকে যাচাই করো

1. sed-এর cycle-এ pattern space-এর কী হয়? `-n` দিলে কোন ধাপটা বন্ধ হয়?
2. `sed '2p' file` চালালে দ্বিতীয় লাইন দুবার আসে কেন?
3. `s/a/A/`, `s/a/A/g`, `s/a/A/3` আর `s/a/A/3g`-এর পার্থক্য কী?
4. `/usr/bin` কে `/opt/bin` বানানোর সবচেয়ে পরিষ্কার sed কমান্ড লেখো।
5. `sed 's/x/y/' file > file` কেন বিপজ্জনক? সঠিক উপায় কী?
6. `sed -i` চালালে ফাইলের inode বদলায় কেন?
7. `John Smith` কে `Smith, John` বানাতে কোন কমান্ড দেবে?
8. শুধু ১০ থেকে ২০ নম্বর লাইন মুছতে, আর শুধু ১০ থেকে ২০ নম্বর লাইন **রাখতে** কী লিখবে?
9. `a`, `i` আর `c`-এর পার্থক্য কী?
10. hold space কী, আর `sed -n '1!G;h;$p'` কী করে?

<details><summary>উত্তর দেখো</summary>

1. প্রতিটি cycle-এ একটা লাইন (শেষের newline ছাড়া) pattern space-এ পড়া হয়, script-এর কমান্ডগুলো এর ওপর চলে, তারপর pattern space প্রিন্ট হয় এবং খালি করা হয়। `-n` দিলে **স্বয়ংক্রিয় প্রিন্টের ধাপ** বন্ধ হয়।
2. একবার `p` কমান্ড প্রিন্ট করে, আরেকবার cycle-শেষের স্বয়ংক্রিয় প্রিন্ট। সমাধান: `sed -n '2p' file`।
3. `s/a/A/` প্রতি লাইনের প্রথম a; `g` সব a; `3` শুধু তৃতীয় a; `3g` তৃতীয় ও তার পরের সব a (GNU)।
4. `sed 's|/usr/bin|/opt/bin|'` (বা অন্য কোনো delimiter যেমন `#`)।
5. shell redirection `> file` কমান্ড চালানোর আগেই ফাইলকে শূন্য করে দেয়, ফলে sed খালি ফাইল পড়ে এবং ডেটা হারায়। সঠিক: `sed -i 's/x/y/' file` বা অন্য ফাইলে লিখে `mv`।
6. কারণ `-i` আসলে একটা temporary ফাইলে নতুন কনটেন্ট লিখে তারপর rename করে মূল নামে আনে — নতুন ফাইল মানে নতুন inode। এজন্য হার্ড লিংক ভেঙে যায়।
7. `echo "John Smith" | sed -E 's/(\w+) (\w+)/\2, \1/'`
8. মুছতে: `sed '10,20d' file`; রাখতে: `sed -n '10,20p' file` (অথবা `sed '10,20!d' file`)।
9. `a` লাইনের পরে, `i` লাইনের আগে নতুন লাইন যোগ করে; `c` লাইনটাকেই নতুন টেক্সট দিয়ে প্রতিস্থাপন করে।
10. hold space একটা সহায়ক বাফার যা cycle পেরিয়েও টিকে থাকে — আগের লাইন মনে রাখতে ব্যবহার হয়। `1!G;h;$p` প্রতিটি নতুন লাইনের নিচে জমানো লাইনগুলো জুড়ে (`G`), পুরোটা আবার hold-এ রাখে (`h`), শেষ লাইনে প্রিন্ট করে — ফলে ফাইল উল্টো ক্রমে দেখায় (`tac`-এর মতো)।

</details>

---

[← আগের অধ্যায়](20-grep-regex.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](22-awk.md)


# অধ্যায় ১৯: ইনপুট/আউটপুট রিডাইরেকশন ও পাইপ

## এই অধ্যায়ে যা শিখবে

- প্রতিটি প্রোগ্রামের তিনটি "জন্মগত" চ্যানেল: **stdin (0)**, **stdout (1)**, **stderr (2)** — এবং **file descriptor** আসলে কী
- `>`, `>>`, `<`, `2>`, `2>&1`, `&>`, `&>>`, `>|`, `<>` — প্রতিটি রিডাইরেকশন অপারেটর কী করে এবং কেন ক্রম (order) গুরুত্বপূর্ণ
- `/dev/null` — লিনাক্সের "ব্ল্যাক হোল", আর তার ভাইবোন `/dev/zero`, `/dev/stdin`, `/dev/fd/N`
- **here document** (`<<EOF`, `<<-EOF`, `<<'EOF'`) ও **here string** (`<<<`)
- **pipe** (`|`) — ভেতরে kernel কীভাবে pipe buffer দিয়ে দুটি প্রসেসকে জোড়া দেয়, `SIGPIPE`, `PIPESTATUS`, `pipefail`
- `tee` — আউটপুট একসাথে স্ক্রিনে ও ফাইলে, এবং `sudo tee` কৌশল
- কমান্ড চেইনিং: `;`, `&&`, `||`
- **subshell** `( )` বনাম **group** `{ }`
- **command substitution** `$( )` ও backtick
- **process substitution** `<( )` ও `>( )`
- **named pipe** (`mkfifo`)
- **exit status** `$?` — কোন সংখ্যার কী মানে
- `exec` দিয়ে স্থায়ী রিডাইরেকশন ও নিজের file descriptor খোলা

---

## ১৯.১ বড় ছবি: লিনাক্সের "সবকিছুই ফাইল" দর্শন

লিনাক্সের (আসলে Unix-এর) সবচেয়ে সুন্দর ধারণাগুলোর একটি হলো: **ছোট ছোট প্রোগ্রাম, প্রত্যেকে একটা কাজ ভালোভাবে করে, আর তাদের জোড়া লাগিয়ে বড় কাজ করা যায়।** এই "জোড়া লাগানোর আঠা" হলো রিডাইরেকশন ও পাইপ।

একটা উপমা দিই। ধরো তোমার বাসার পানির লাইন। প্রতিটি কল (প্রোগ্রাম) এর একটা **ইনলেট** (পানি ঢোকে), একটা **আউটলেট** (পরিষ্কার পানি বের হয়) আর একটা **ড্রেন** (ময়লা পানি বের হয়)। প্লাম্বার (shell) চাইলে:

- আউটলেটের পাইপ বালতিতে (ফাইলে) ঘুরিয়ে দিতে পারে — **রিডাইরেকশন**
- এক কলের আউটলেট অন্য কলের ইনলেটে জুড়ে দিতে পারে — **পাইপ**
- ড্রেনের পানি নর্দমায় ফেলে দিতে পারে — **`/dev/null`**

প্রোগ্রাম নিজে জানেই না তার পানি কোথায় যাচ্ছে! সে শুধু "আউটলেটে লেখো" জানে। এটাই রিডাইরেকশনের জাদু — **প্রোগ্রাম না বদলিয়েই তার ইনপুট/আউটপুট বদলানো।**

> 📝 মনে রাখো: রিডাইরেকশন ও পাইপ সাজায় **shell** (bash), প্রোগ্রাম নয়। প্রোগ্রাম চালু হওয়ার *আগেই* shell সব "পাইপলাইন" ফিট করে রাখে।

---

## ১৯.২ File Descriptor: প্রোগ্রামের হাতে থাকা টিকিট

### File descriptor কী?

**File descriptor (FD)** হলো একটা ছোট পূর্ণসংখ্যা (0, 1, 2, 3 …) যা দিয়ে একটা প্রসেস kernel-কে বলে "এই খোলা ফাইলটায় পড়ো/লেখো।" যখন কোনো প্রোগ্রাম ফাইল খোলে (`open()` system call), kernel একটা নম্বর ফেরত দেয় — ঠিক যেমন ব্যাংকে টোকেন নম্বর পাও। পরে শুধু টোকেন দেখালেই চলে।

এখানে "ফাইল" মানে শুধু ডিস্কের ফাইল না — টার্মিনাল, pipe, socket, device সবকিছুই FD দিয়ে পড়া-লেখা হয়। এজন্যই বলা হয় লিনাক্সে "everything is a file"।

### তিনটি স্ট্যান্ডার্ড স্ট্রিম

প্রতিটি প্রসেস জন্ম থেকেই তিনটি FD খোলা অবস্থায় পায়:

| FD | নাম | পূর্ণ রূপ | ডিফল্টভাবে কোথায় যুক্ত | কাজ |
|----|-----|----------|------------------------|-----|
| 0 | stdin | standard input | কীবোর্ড (টার্মিনাল) | প্রোগ্রাম এখান থেকে ইনপুট পড়ে |
| 1 | stdout | standard output | স্ক্রিন (টার্মিনাল) | স্বাভাবিক ফলাফল এখানে লেখে |
| 2 | stderr | standard error | স্ক্রিন (টার্মিনাল) | এরর ও ডায়াগনস্টিক মেসেজ এখানে লেখে |

```
                     ┌──────────────────────┐
   কীবোর্ড ─────────▶│ 0  stdin             │
                     │                      │
                     │      প্রসেস          │──── 1 stdout ───▶  স্ক্রিন
                     │   (যেমন: ls)         │
                     │                      │──── 2 stderr ───▶  স্ক্রিন
                     └──────────────────────┘
```

**কেন stdout আর stderr আলাদা?** কল্পনা করো তুমি `ls` এর আউটপুট ফাইলে রাখছ। যদি এরর মেসেজও একই জায়গায় যেত, তাহলে তোমার ডেটা ফাইলে "No such file or directory" জাতীয় আবর্জনা মিশে যেত। আলাদা চ্যানেল থাকায় ডেটা এক জায়গায়, সমস্যার খবর আরেক জায়গায় — দুটো আলাদাভাবে সামলানো যায়।

### Under the hood: FD table

kernel প্রতিটি প্রসেসের জন্য একটা **file descriptor table** রাখে। তুমি নিজেই দেখতে পারো:

```
$ ls -l /proc/$$/fd
total 0
lrwx------ 1 rahim rahim 64 Nov 12 10:00 0 -> /dev/pts/0
lrwx------ 1 rahim rahim 64 Nov 12 10:00 1 -> /dev/pts/0
lrwx------ 1 rahim rahim 64 Nov 12 10:00 2 -> /dev/pts/0
lrwx------ 1 rahim rahim 64 Nov 12 10:00 255 -> /dev/pts/0
```

এখানে `$$` মানে বর্তমান shell-এর PID। দেখো, 0, 1, 2 তিনটাই `/dev/pts/0` — অর্থাৎ তোমার টার্মিনাল। (255 হলো bash-এর নিজের ব্যবহারের জন্য রাখা একটা FD।) `/proc` সম্পর্কে বিস্তারিত [অধ্যায় ৪৭](47-kernel-proc-sys.md)-এ।

```
   প্রসেস (PID 1234)                kernel-এর open file table        আসল জিনিস
   ┌──────────────┐
   │ FD table     │
   │  0 ──────────┼──────────────▶  [open file: read/write, pos] ──▶ /dev/pts/0
   │  1 ──────────┼──────────────▶  (একই entry শেয়ার)            ──▶ /dev/pts/0
   │  2 ──────────┼──────────────▶                                ──▶ /dev/pts/0
   │  3 ──────────┼──────────────▶  [open file: write, pos=0]   ──▶ /home/rahim/out.txt
   └──────────────┘
```

রিডাইরেকশন মানে আসলে এই টেবিলের কোনো সারির "তীর" অন্য দিকে ঘুরিয়ে দেওয়া। shell এটা করে `fork()` এর পর আর `exec()` এর আগে, `open()` ও `dup2()` system call দিয়ে। যেমন `ls > out.txt` চালালে shell মোটামুটি এই কাজ করে:

```
1. fork()                     → shell-এর একটা কপি (child) তৈরি
2. child-এ: fd = open("out.txt", O_WRONLY|O_CREAT|O_TRUNC)   → ধরো fd=3
3. child-এ: dup2(3, 1)        → FD 1 এখন out.txt-কে দেখায়
4. child-এ: close(3)
5. child-এ: exec("ls")        → ls চালু হয়, সে ভাবে স্বাভাবিকভাবেই FD 1-এ লিখছে
```

`ls` জানেই না যে তার stdout এখন একটা ফাইল! (fork/exec সম্পর্কে আরও জানতে [অধ্যায় ২৪](24-process-management.md) দেখো।)

---

## ১৯.৩ আউটপুট রিডাইরেকশন: `>` ও `>>`

### `>` — ফাইলে লেখো (ওভাররাইট)

**Syntax:** `command > file` (আসলে এটা `command 1> file`-এর সংক্ষিপ্ত রূপ)

```
$ echo "হ্যালো লিনাক্স" > greeting.txt
$ cat greeting.txt
হ্যালো লিনাক্স
```

- ফাইল না থাকলে **তৈরি হবে**।
- ফাইল থাকলে **আগের সব লেখা মুছে যাবে** (truncate), তারপর নতুন লেখা বসবে।

```
$ echo "প্রথম লাইন" > note.txt
$ echo "দ্বিতীয় লাইন" > note.txt
$ cat note.txt
দ্বিতীয় লাইন
```

প্রথম লাইন হারিয়ে গেল!

> ⚠️ সতর্কতা: `>` কোনো প্রশ্ন না করেই ফাইল খালি করে দেয়। ভুলে `> important.conf` লিখলে পুরো কনফিগ শেষ। আর ফাইল খালি হয় **কমান্ড চালু হওয়ার আগেই** — তাই কমান্ড ভুল হলেও ক্ষতি হয়ে যায়।

### সবচেয়ে কুখ্যাত ভুল: একই ফাইল পড়া ও লেখা

```
$ sort names.txt > names.txt      # ❌ ভুল!
$ cat names.txt
$                                  # ফাইল খালি!
```

কেন? shell প্রথমে `names.txt` কে `>` দিয়ে খোলে (তখনই খালি হয়ে যায়), তারপর `sort` চালু হয়ে খালি ফাইল পড়ে। সমাধান:

```
$ sort names.txt > names.sorted && mv names.sorted names.txt
$ sort -o names.txt names.txt      # sort-এর নিজস্ব -o অপশন নিরাপদ
```

### `noclobber` — দুর্ঘটনা থেকে বাঁচার সুইচ

```
$ set -o noclobber          # চালু করো
$ echo "test" > note.txt
bash: note.txt: cannot overwrite existing file
$ echo "test" >| note.txt   # জেনেশুনে ওভাররাইট করতে চাইলে >|
$ set +o noclobber          # বন্ধ করো
```

| অপারেটর | কাজ |
|---------|-----|
| `set -o noclobber` (বা `set -C`) | `>` দিয়ে বিদ্যমান ফাইল ওভাররাইট নিষেধ |
| `>|` | noclobber থাকা সত্ত্বেও জোর করে ওভাররাইট |
| `set +o noclobber` | আবার স্বাভাবিক অবস্থায় ফেরা |

> 💡 টিপস: `~/.bashrc`-এ `set -o noclobber` যোগ করে রাখলে অনেক দুর্ঘটনা এড়ানো যায় ([অধ্যায় ৪৪](44-environment-bashrc.md))।

### `>>` — ফাইলের শেষে যোগ করো (append)

```
$ echo "সকাল ৯টা: সার্ভার চালু" >> diary.log
$ echo "সকাল ১০টা: ব্যাকআপ শেষ" >> diary.log
$ cat diary.log
সকাল ৯টা: সার্ভার চালু
সকাল ১০টা: ব্যাকআপ শেষ
```

`>>` ফাইল খোলে `O_APPEND` ফ্ল্যাগ দিয়ে — kernel প্রতিটি write-এর আগে লেখার জায়গা ফাইলের শেষে নিয়ে যায়। তাই একাধিক প্রসেস একই লগ ফাইলে `>>` দিয়ে লিখলেও একে অপরের লেখা ওভাররাইট করে না।

### খালি ফাইল তৈরি বা ফাইল খালি করার কৌশল

```
$ > empty.txt          # ফাইল তৈরি বা খালি (bash-এ কাজ করে)
$ : > app.log          # ':' কিছু করে না, শুধু রিডাইরেকশন হয় — বেশি portable
$ truncate -s 0 app.log   # আরেক উপায়
```

> 💡 টিপস: চলমান সার্ভারের বড় লগ ফাইল `rm` করলে ডিস্ক স্পেস ফেরত আসে না (প্রসেস তখনো ফাইলটা খোলা রেখেছে — [অধ্যায় ১১](11-links-inode.md) দেখো)। বরং `: > app.log` দিয়ে খালি করো।

---

## ১৯.৪ ইনপুট রিডাইরেকশন: `<`

**Syntax:** `command < file` (= `command 0< file`)

stdin কীবোর্ডের বদলে ফাইল থেকে আসে।

```
$ wc -l < /etc/passwd
42
$ wc -l /etc/passwd
42 /etc/passwd
```

পার্থক্য লক্ষ করো: দ্বিতীয় ক্ষেত্রে `wc` নিজে ফাইল খুলেছে তাই নাম জানে; প্রথম ক্ষেত্রে shell ফাইল খুলে দিয়েছে, `wc` শুধু একটা "ধারা" পেয়েছে — নাম জানে না। স্ক্রিপ্টে শুধু সংখ্যা দরকার হলে `<` কাজের।

```
$ tr 'a-z' 'A-Z' < notes.txt      # tr নিজে ফাইল নাম নেয় না, তাই < লাগবেই
```

### `<>` — পড়া ও লেখা দুটোর জন্য খোলা

```
$ exec 3<> /tmp/data.txt     # FD 3 পড়া-লেখার জন্য খোলা (না থাকলে তৈরি হয়)
```

এটা কম ব্যবহৃত, তবে device বা socket নিয়ে কাজে লাগে (১৯.১৫-তে আরও)।

---

## ১৯.৫ এরর রিডাইরেকশন: `2>`, `2>&1`, `&>`

### `2>` — শুধু এরর আলাদা করো

```
$ ls /etc/hostname /nai
ls: cannot access '/nai': No such file or directory
/etc/hostname

$ ls /etc/hostname /nai 2> errors.txt
/etc/hostname
$ cat errors.txt
ls: cannot access '/nai': No such file or directory
```

stdout স্ক্রিনে রইল, stderr ফাইলে গেল। দুটো আলাদা ফাইলেও পাঠানো যায়:

```
$ ls /etc/hostname /nai > ok.txt 2> err.txt
```

`2>>` দিয়ে এরর append করা যায়।

### `2>&1` — stderr-কে stdout যেখানে যাচ্ছে সেখানে পাঠাও

`2>&1` পড়ো এভাবে: "FD 2-কে FD 1-এর **বর্তমান** গন্তব্যের কপি বানাও।" `&` না দিলে (`2>1`) shell ভাবত "1" নামের একটা ফাইল!

```
$ ls /etc/hostname /nai > all.txt 2>&1
$ cat all.txt
ls: cannot access '/nai': No such file or directory
/etc/hostname
```

### ⚠️ ক্রম (order) কেন গুরুত্বপূর্ণ

shell রিডাইরেকশনগুলো **বাম থেকে ডানে** একটার পর একটা প্রয়োগ করে। এটাই সবচেয়ে বেশি বিভ্রান্তি সৃষ্টি করে।

**ঠিক:** `cmd > file 2>&1`

```
ধাপ ০ (শুরু):     1 → টার্মিনাল     2 → টার্মিনাল
ধাপ ১ (> file):   1 → file          2 → টার্মিনাল
ধাপ ২ (2>&1):     1 → file          2 → file        ✅ দুটোই ফাইলে
```

**ভুল (প্রত্যাশামতো কাজ করে না):** `cmd 2>&1 > file`

```
ধাপ ০ (শুরু):     1 → টার্মিনাল     2 → টার্মিনাল
ধাপ ১ (2>&1):     1 → টার্মিনাল     2 → টার্মিনাল   (2 কপি করল 1-এর তখনকার গন্তব্য)
ধাপ ২ (> file):   1 → file          2 → টার্মিনাল   ❌ এরর এখনো স্ক্রিনে
```

`2>&1` কোনো স্থায়ী "বাঁধন" না — এটা ঠিক ওই মুহূর্তের একটা কপি (`dup2(1, 2)`)।

> 📝 মনে রাখো: "আগে ঠিকানা, তারপর জোড়া" — `> file 2>&1`।

### `&>` ও `&>>` — bash-এর শর্টকাট

```
$ cmd &> all.txt       # = cmd > all.txt 2>&1
$ cmd &>> all.txt      # = cmd >> all.txt 2>&1   (bash 4+)
```

> ⚠️ সতর্কতা: `&>` হলো bash/zsh-এর সুবিধা, POSIX `sh`-এ নেই। Debian/Ubuntu-তে `/bin/sh` আসলে `dash` — সেখানে `cmd &> file` মানে হয় "`cmd` কে background-এ চালাও (`&`), আর `> file` দিয়ে একটা খালি ফাইল বানাও"! `#!/bin/sh` স্ক্রিপ্টে সবসময় `> file 2>&1` লেখো।

### `1>&2` বা `>&2` — নিজের মেসেজ stderr-এ পাঠানো

স্ক্রিপ্টে এরর মেসেজ সঠিক চ্যানেলে দাও:

```bash
echo "ত্রুটি: কনফিগ ফাইল পাওয়া যায়নি" >&2
```

এতে কেউ তোমার স্ক্রিপ্টের আউটপুট ফাইলে রাখলেও এরর মেসেজ স্ক্রিনে দেখবে।

### stdout ও stderr অদলবদল (swap)

একটা অতিরিক্ত FD 3 ধার করে:

```
$ cmd 3>&1 1>&2 2>&3 3>&-
```

ধাপে ধাপে: 3 = stdout-এর কপি; 1 = stderr-এর জায়গায়; 2 = 3 (আগের stdout); তারপর 3 বন্ধ (`3>&-`)। এখন শুধু এররগুলো পাইপে পাঠানো যায়:

```
$ find / -name '*.conf' 3>&1 1>&2 2>&3 3>&- | grep -c 'Permission denied'
```

### সব রিডাইরেকশন অপারেটর একনজরে

| অপারেটর | অর্থ | উদাহরণ |
|---------|------|---------|
| `> f` / `1> f` | stdout → f (ওভাররাইট) | `ls > list.txt` |
| `>> f` | stdout → f (append) | `date >> log.txt` |
| `< f` | stdin ← f | `wc -l < f` |
| `2> f` | stderr → f | `cmd 2> err.txt` |
| `2>> f` | stderr → f (append) | `cmd 2>> err.log` |
| `2>&1` | stderr → stdout যেখানে আছে | `cmd > f 2>&1` |
| `>&2` / `1>&2` | stdout → stderr যেখানে আছে | `echo err >&2` |
| `&> f` | stdout+stderr → f (bash) | `cmd &> all.txt` |
| `&>> f` | stdout+stderr → f append (bash) | `cmd &>> all.log` |
| `>| f` | noclobber উপেক্ষা করে ওভাররাইট | `echo x >| f` |
| `<> f` | পড়া-লেখার জন্য খোলা | `exec 3<> f` |
| `n>&-` | FD n বন্ধ করো | `exec 3>&-` |
| `n>&m` | FD n = FD m-এর কপি | `3>&1` |
| `|&` | `2>&1 |` এর সংক্ষেপ (bash) | `make |& less` |

> 💡 টিপস: রিডাইরেকশন কমান্ড লাইনের যেকোনো জায়গায় লেখা যায় — `> out.txt echo hi` ও কাজ করে। তবে পড়ার সুবিধার জন্য শেষে রাখো।

---

## ১৯.৬ `/dev/null` ও তার ভাইবোন

### `/dev/null` — লিনাক্সের ব্ল্যাক হোল

`/dev/null` একটা **character device** — এতে যা লেখো সব হারিয়ে যায় (kernel ফেলে দেয়), আর পড়তে গেলে সাথে সাথে EOF (ফাইলের শেষ) পাও।

```
$ ls -l /dev/null
crw-rw-rw- 1 root root 1, 3 Nov 12 09:00 /dev/null
```

প্রথম অক্ষর `c` = character device; `1, 3` হলো major/minor device নম্বর। সবাই পড়তে-লিখতে পারে (`rw-rw-rw-`)।

ব্যবহার:

```
$ find / -name passwd 2>/dev/null          # শুধু Permission denied এরর লুকাও
/etc/passwd
/usr/bin/passwd
...

$ cp bigfile /backup/ > /dev/null 2>&1      # কোনো আউটপুটই চাই না
$ cmd &> /dev/null                          # bash-এ একই

$ if grep -q root /etc/passwd; then echo আছে; fi     # -q থাকলে /dev/null লাগে না
আছে

$ cat /dev/null > app.log                   # ফাইল খালি করার আরেক উপায়
$ some_cmd < /dev/null                      # কোনো ইনপুট দিও না (প্রোগ্রাম অপেক্ষায় আটকাবে না)
```

> ⚠️ সতর্কতা: `2>/dev/null` দিয়ে সব এরর ঢেকে দেওয়া অভ্যাস করো না। সমস্যা হলে তুমি কিছুই জানবে না। শুধু যেগুলো জানা ও নিরীহ (যেমন `find`-এর Permission denied) সেগুলোই লুকাও।

### অন্যান্য বিশেষ ফাইল

| ফাইল | কাজ |
|------|-----|
| `/dev/null` | লিখলে হারিয়ে যায়; পড়লে খালি (EOF) |
| `/dev/zero` | পড়লে অসীম শূন্য বাইট (`\0`) দেয় |
| `/dev/random`, `/dev/urandom` | এলোমেলো বাইট দেয় |
| `/dev/full` | লিখতে গেলে "No space left on device" এরর দেয় (টেস্টের জন্য) |
| `/dev/stdin`, `/dev/stdout`, `/dev/stderr` | বর্তমান প্রসেসের FD 0/1/2 (আসলে `/proc/self/fd/0..2` এর symlink) |
| `/dev/fd/N` | বর্তমান প্রসেসের FD N |
| `/dev/tty` | বর্তমান কন্ট্রোলিং টার্মিনাল (রিডাইরেকশন থাকলেও সরাসরি টার্মিনালে লেখে/পড়ে) |

```
$ head -c 16 /dev/urandom | od -An -tx1
 3f a1 07 9c 5e 22 d0 18 b4 66 0e 7a c9 31 f2 4d
$ dd if=/dev/zero of=test.img bs=1M count=10
10+0 records in
10+0 records out
10485760 bytes (10 MB, 10 MiB) copied, 0.0123 s, 852 MB/s
$ echo "এটা সরাসরি টার্মিনালে" > /dev/tty
```

`/dev/stdin` দিয়ে এমন প্রোগ্রামকে stdin দেওয়া যায় যারা শুধু ফাইলনাম চায়:

```
$ echo "hello" | diff /dev/stdin <(echo "hello")
$
```

---

## ১৯.৭ Here Document (`<<`)

### কী ও কেন

ধরো স্ক্রিপ্টের ভেতর থেকে একটা কমান্ডকে কয়েক লাইনের ইনপুট দিতে চাও — আলাদা ফাইল না বানিয়ে। এটাই **here document** (heredoc): "ইনপুট এখানেই, এই ডকুমেন্টে।"

**Syntax:**

```
command <<DELIMITER
লাইন ১
লাইন ২
DELIMITER
```

`DELIMITER` যেকোনো শব্দ হতে পারে (প্রথাগতভাবে `EOF`, `END`)। শেষের delimiter **একা একটা লাইনে, শুরুতে বা শেষে কোনো স্পেস ছাড়া** থাকতে হবে।

```
$ cat <<EOF
নাম: $USER
হোম: $HOME
আজ: $(date +%A)
EOF
নাম: rahim
হোম: /home/rahim
আজ: Thursday
```

দেখো, ভেতরের ভেরিয়েবল ও `$( )` expand হয়েছে।

### Heredoc দিয়ে ফাইল তৈরি

```
$ cat > config.ini <<EOF
[server]
host = localhost
port = 8080
EOF
$ cat config.ini
[server]
host = localhost
port = 8080
```

### `<<'EOF'` — expansion বন্ধ (আক্ষরিক লেখা)

delimiter-কে quote করলে (`'EOF'`, `"EOF"` বা `\EOF`) ভেতরে কিছুই expand হয় না। স্ক্রিপ্টের ভেতর থেকে আরেকটা স্ক্রিপ্ট লিখতে এটা অপরিহার্য:

```
$ cat > hello.sh <<'EOF'
#!/bin/bash
echo "আমার নাম $USER"
EOF
$ cat hello.sh
#!/bin/bash
echo "আমার নাম $USER"
```

quote না করলে `$USER` তখনই তোমার নাম দিয়ে বদলে যেত।

### `<<-EOF` — শুরুর ট্যাব মুছে দাও

`<<-` ব্যবহার করলে প্রতিটি লাইনের (এবং শেষ delimiter-এর) **শুরুর TAB অক্ষর** মুছে যায় — ফলে স্ক্রিপ্টে সুন্দর ইনডেন্ট রাখা যায়:

```bash
if true; then
	cat <<-EOF
		এই লাইনের শুরুর ট্যাব মুছে যাবে
	EOF
fi
```

> ⚠️ সতর্কতা: `<<-` শুধু **TAB** মোছে, **space** নয়। অনেক এডিটর ট্যাবকে স্পেসে বদলে দেয় — তখন delimiter চিনতে না পেরে স্ক্রিপ্ট ভেঙে যায় ("here-document delimited by end-of-file" warning)।

### Heredoc-এর বাস্তব ব্যবহার

```
# root হিসেবে ফাইল লেখা (sudo tee — ১৯.১০ দেখো)
$ sudo tee /etc/motd > /dev/null <<'EOF'
স্বাগতম! এই সার্ভারে সব কার্যকলাপ লগ হয়।
EOF

# রিমোট সার্ভারে একাধিক কমান্ড
$ ssh user@server <<'EOF'
uptime
df -h /
EOF

# ডেটাবেস কোয়েরি
$ mysql -u root -p mydb <<EOF
SELECT COUNT(*) FROM users;
EOF
```

### Under the hood

bash heredoc-এর লেখাটা একটা অস্থায়ী জায়গায় রাখে এবং সেটার রিড-এন্ড কমান্ডের stdin (FD 0) হিসেবে দেয়। bash 5.1 থেকে লেখা ছোট হলে (pipe buffer-এর মধ্যে ধরে গেলে) pipe ব্যবহার করে, বড় হলে temp file — তোমার দিক থেকে কোনো পার্থক্য নেই, কমান্ড শুধু stdin থেকে পড়ে।

---

## ১৯.৮ Here String (`<<<`)

একটা মাত্র স্ট্রিং stdin হিসেবে দিতে চাইলে heredoc বাড়াবাড়ি। তখন **here string**:

**Syntax:** `command <<< "string"`

```
$ wc -w <<< "আমি লিনাক্স শিখছি"
3
$ tr 'a-z' 'A-Z' <<< "hello world"
HELLO WORLD
$ bc <<< "2^10"
1024
$ read first rest <<< "রহিম করিম যদু"
$ echo "$first"
রহিম
```

here string বনাম `echo ... |`:

```
$ echo "hello" | read x; echo "x=$x"
x=
$ read x <<< "hello"; echo "x=$x"
x=hello
```

পাইপে `read` একটা subshell-এ চলে (১৯.৯ দেখো), তাই ভেরিয়েবল হারিয়ে যায়। here string-এ `read` বর্তমান shell-এই চলে।

> 📝 মনে রাখো: here string শেষে একটা newline (`\n`) যোগ করে দেয়। `wc -c <<< "abc"` দেবে `4`, `3` নয়।

> ⚠️ সতর্কতা: `<<<` bash/zsh/ksh-এ আছে, POSIX `sh`/`dash`-এ নেই।

---

## ১৯.৯ পাইপ (`|`) — প্রোগ্রাম জোড়া লাগানোর আঠা

### কী করে

**Syntax:** `cmd1 | cmd2 | cmd3 ...`

`cmd1`-এর **stdout** সরাসরি `cmd2`-এর **stdin**-এ চলে যায় — মাঝে কোনো ফাইল নেই।

```
$ ls /usr/bin | wc -l
1873
$ cat /etc/passwd | cut -d: -f1 | sort | head -3
_apt
backup
bin
```

উপমা: কারখানার কনভেয়র বেল্ট। প্রথম শ্রমিক জিনিস কাটে, বেল্টে রাখে; দ্বিতীয়জন সাথে সাথে তুলে রং করে; তৃতীয়জন প্যাক করে। সবাই **একসাথে** কাজ করছে — কেউ পুরো ব্যাচ শেষ হওয়ার অপেক্ষায় বসে নেই।

### Under the hood: kernel pipe buffer

যখন তুমি `ls | wc -l` লেখো, bash এই কাজগুলো করে:

```
1. pipe()   → kernel একটা pipe বানায়, দুটো FD দেয়: fd[0]=read end, fd[1]=write end
2. fork()   → child A (ls-এর জন্য):  dup2(fd[1], 1)  — stdout = pipe-এর write end
                                    বাকি pipe FD বন্ধ, তারপর exec("ls")
3. fork()   → child B (wc-এর জন্য):  dup2(fd[0], 0)  — stdin = pipe-এর read end
                                    বাকি pipe FD বন্ধ, তারপর exec("wc")
4. parent shell নিজের কপির দুই প্রান্তই বন্ধ করে, তারপর দুই child-এর জন্য অপেক্ষা (wait)
```

```
   ┌─────────┐   FD1   ┌───────────────────────────────┐   FD0   ┌─────────┐
   │   ls    │────────▶│ kernel pipe buffer (64 KiB)   │────────▶│   wc    │
   │ (PID A) │  write  │ [ b i n \n c a t \n ... ]     │  read   │ (PID B) │
   └─────────┘         └───────────────────────────────┘         └─────────┘
        │ FD2                (kernel memory-তে রিং বাফার)              │ FD1
        ▼                                                             ▼
     টার্মিনাল                                                    টার্মিনাল
```

গুরুত্বপূর্ণ বিষয়গুলো:

1. **pipe buffer kernel memory-তে থাকে**, ডিস্কে না। Linux-এ ডিফল্ট সাইজ **65536 বাইট (64 KiB)**। (non-root একটা pipe-কে সর্বোচ্চ `/proc/sys/fs/pipe-max-size` পর্যন্ত বড় করতে পারে, সাধারণত 1 MiB।)

   ```
   $ cat /proc/sys/fs/pipe-max-size
   1048576
   ```

2. **Flow control (প্রবাহ নিয়ন্ত্রণ):**
   - buffer **ভর্তি** হলে লেখক (`ls`) `write()`-এ **block** হয়ে ঘুমায় — যতক্ষণ না পাঠক কিছু পড়ে জায়গা খালি করে।
   - buffer **খালি** হলে পাঠক (`wc`) `read()`-এ block হয়ে অপেক্ষা করে।
   - লেখক সব write end বন্ধ করলে (প্রসেস শেষ), পাঠক `read()`-এ **0 (EOF)** পায় — বুঝে যায় ডেটা শেষ।

   এভাবেই দ্রুত প্রোডিউসার ধীর কনজিউমারকে মেমরি দিয়ে ডুবিয়ে দিতে পারে না।

3. **সব কমান্ড একসাথে (concurrently) চলে।** `cmd1 | cmd2` মানে cmd1 শেষ হলে cmd2 চালু — এমন না। দুটো একই সময়ে আলাদা প্রসেস হিসেবে চলে। প্রমাণ:

   ```
   $ sleep 3 | sleep 3       # মোট ~3 সেকেন্ড লাগে, 6 না
   ```

4. **Atomic write:** `PIPE_BUF` (Linux-এ 4096 বাইট) বা তার কম আকারের একটা write কখনো অন্য লেখকের write-এর সাথে মিশে যায় না।

5. **SIGPIPE:** পাঠক আগে চলে গেলে (read end সব বন্ধ) লেখক পরের `write()`-এ **SIGPIPE** সিগন্যাল পায় এবং সাধারণত চুপচাপ মারা যায়। এজন্যই `yes | head -3` চিরকাল চলে না:

   ```
   $ yes | head -3
   y
   y
   y
   ```

   `head` তিন লাইন পড়ে বেরিয়ে যায় → `yes` লিখতে গিয়ে SIGPIPE পায় → শেষ। এজন্যই `cat huge.log | head` দ্রুত শেষ হয়, পুরো ফাইল পড়ে না।

### শুধু stdout পাইপে যায়, stderr না

```
$ ls /etc/hostname /nai | wc -l
ls: cannot access '/nai': No such file or directory
1
```

এরর সরাসরি স্ক্রিনে এল, পাইপে গেল না। stderr-ও পাঠাতে চাইলে:

```
$ ls /etc/hostname /nai 2>&1 | wc -l
2
$ ls /etc/hostname /nai |& wc -l       # bash শর্টকাট
2
```

### পাইপলাইনের exit status: `PIPESTATUS` ও `pipefail`

ডিফল্টভাবে একটা পাইপলাইনের exit status হলো **শেষ কমান্ডের** exit status।

```
$ false | true
$ echo $?
0
```

`false` ব্যর্থ হলো, কিন্তু আমরা জানতে পারলাম না! সমাধান দুটো:

```
$ false | true | false
$ echo "${PIPESTATUS[@]}"        # প্রতিটি কমান্ডের status (bash array)
1 0 1

$ set -o pipefail                 # কোনো একটা ব্যর্থ হলেই পুরো পাইপলাইন ব্যর্থ
$ false | true
$ echo $?
1
```

> ⚠️ সতর্কতা: `PIPESTATUS` পরের যেকোনো কমান্ড চালালেই (এমনকি `echo $?`) বদলে যায়। দরকার হলে সাথে সাথে কপি করো: `st=("${PIPESTATUS[@]}")`।

> 💡 টিপস: সিরিয়াস স্ক্রিপ্ট শুরু করো `set -euo pipefail` দিয়ে ([অধ্যায় ৪৩](43-shell-scripting-3.md))।

### পাইপের প্রতিটি অংশ আলাদা প্রসেস (subshell সমস্যা)

bash-এ পাইপলাইনের প্রতিটি কমান্ড আলাদা subshell-এ চলে — শেষটাও। তাই এটা কাজ করে না:

```
$ count=0
$ cat names.txt | while read -r line; do count=$((count+1)); done
$ echo $count
0
```

`while` লুপটা subshell-এ চলেছে; তার `count` আলাদা কপি, শেষে হারিয়ে গেছে। সমাধান:

```
$ count=0
$ while read -r line; do count=$((count+1)); done < names.txt
$ echo $count
5
```

অথবা process substitution (১৯.১৪): `done < <(some_command)`। অথবা স্ক্রিপ্টে `shopt -s lastpipe` (শুধু যখন job control বন্ধ — যেমন non-interactive স্ক্রিপ্টে — তখন শেষ কমান্ড বর্তমান shell-এ চলে)।

### UUOC — "Useless Use of Cat"

```
$ cat file | grep error     # কাজ করে, কিন্তু অপ্রয়োজনীয় একটা প্রসেস
$ grep error file           # ভালো
$ grep error < file         # এটাও ভালো
```

ছোট কাজে পার্থক্য নগণ্য, তবে অভ্যাসটা ভালো রাখা উচিত। পড়ার সুবিধার জন্য `cat` দিয়ে শুরু করলে সেটা অপরাধ না।

---

## ১৯.১০ `tee` — T-আকৃতির পাইপ

### কী করে

প্লাম্বিংয়ের **T-জয়েন্ট** দেখেছ? একটা পাইপের পানি দুই দিকে ভাগ হয়। `tee` ঠিক তাই: stdin থেকে পড়ে, **একসাথে** stdout-এ ও এক বা একাধিক ফাইলে লেখে।

```
               ┌──────────▶ stdout ──▶ (স্ক্রিন বা পরের পাইপ)
  stdin ──▶  tee
               └──────────▶ file.txt
```

**Syntax:** `command | tee [OPTION]... [FILE]...`

| অপশন | কাজ |
|------|-----|
| `-a`, `--append` | ফাইল ওভাররাইট না করে শেষে যোগ করো |
| `-i`, `--ignore-interrupts` | Ctrl+C (SIGINT) উপেক্ষা করো |

```
$ ls /etc | tee etc-list.txt | wc -l
etc-list.txt তৈরি হলো, আর স্ক্রিনে:
231
```

(এখানে `tee`-র stdout পাইপে গেছে, তাই স্ক্রিনে শুধু সংখ্যা। লিস্টটা ফাইলে আছে।)

```
$ ping -c 3 google.com | tee ping.log          # দেখোও, রাখোও
$ make 2>&1 | tee build.log                     # বিল্ডের সব আউটপুট দেখো ও লগ রাখো
$ echo "নতুন লাইন" | tee -a log1.txt log2.txt   # একসাথে দুই ফাইলে append
```

### `sudo tee` — সবচেয়ে কাজের কৌশল

```
$ sudo echo "nameserver 1.1.1.1" > /etc/resolv.conf
bash: /etc/resolv.conf: Permission denied
```

কেন? `sudo` শুধু `echo`-কে root বানিয়েছে। কিন্তু `>` রিডাইরেকশন করছে **তোমার নিজের shell** — যে সাধারণ ইউজার! ফাইল খোলার কাজটা sudo-র বাইরে।

সমাধান: ফাইলে লেখার কাজটা root প্রসেসকে দাও:

```
$ echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf > /dev/null
$ echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf > /dev/null
```

`> /dev/null` দিলাম যাতে `tee` লেখাটা আবার স্ক্রিনে না দেখায়। বিকল্প: `sudo sh -c 'echo ... > /etc/file'`। (sudo বিস্তারিত: [অধ্যায় ১৮](18-sudo-root.md)।)

---

## ১৯.১১ কমান্ড চেইনিং: `;`, `&&`, `||`

### Exit status আগে বুঝে নাও

প্রতিটি কমান্ড শেষে একটা সংখ্যা ফেরত দেয় — **exit status**। `0` মানে সফল, অন্য যেকোনো সংখ্যা (1–255) মানে কোনো না কোনো ব্যর্থতা। (বিস্তারিত ১৯.১৭-তে।)

### `;` — শর্তহীন ক্রম

```
$ cd /tmp ; ls ; echo "শেষ"
```

আগেরটা সফল হোক বা না হোক, পরেরটা চলবে। নতুন লাইনে লেখা আর `;` দেওয়া একই।

### `&&` — "সফল হলে তবেই"

```
$ mkdir project && cd project && git init
```

`mkdir` ব্যর্থ হলে `cd` চলবে না; `cd` ব্যর্থ হলে `git init` চলবে না। এটা নিরাপত্তার জন্য খুব গুরুত্বপূর্ণ:

```
$ cd /var/tmp/cache ; rm -rf *      # ❌ cd ব্যর্থ হলে বর্তমান ফোল্ডারেই সব মুছবে!
$ cd /var/tmp/cache && rm -rf *     # ✅ cd ব্যর্থ হলে rm চলবেই না
```

```
$ sudo apt update && sudo apt upgrade -y
```

### `||` — "ব্যর্থ হলে তবেই"

```
$ ping -c1 -W2 8.8.8.8 > /dev/null || echo "ইন্টারনেট নেই!"
$ mkdir /data 2>/dev/null || echo "ফোল্ডার তৈরি করা যায়নি"
$ grep -q "^rahim:" /etc/passwd || sudo useradd rahim
```

### `&& ... ||` — if-else-এর সংক্ষেপ (সাবধানে)

```
$ [ -f /etc/hostname ] && echo "আছে" || echo "নেই"
আছে
```

> ⚠️ সতর্কতা: `A && B || C` আসল if-else **না**। যদি `A` সফল হয় কিন্তু `B` ব্যর্থ হয়, তাহলে `C`-ও চলবে! যেমন `true && false || echo "এটাও চলল"`। জটিল ক্ষেত্রে আসল `if` ব্যবহার করো ([অধ্যায় ৪১](41-shell-scripting-1.md))।

### অগ্রাধিকার (precedence)

`&&` ও `||` সমান অগ্রাধিকার, বাম থেকে ডানে মূল্যায়ন হয়। `;` ও `&` এদের চেয়ে দুর্বল, আর `|` সবচেয়ে শক্তিশালী।

```
$ false && echo A ; echo B
B
```

`echo B` সবসময় চলে কারণ `;` সবকিছুকে আলাদা করে।

| অপারেটর | কখন পরের কমান্ড চলে | উদাহরণ |
|---------|-------------------|---------|
| `;` | সবসময় | `date ; uptime` |
| `&&` | আগেরটা সফল (status 0) হলে | `make && make install` |
| `||` | আগেরটা ব্যর্থ (status ≠ 0) হলে | `cmd || echo failed` |
| `&` | আগেরটা background-এ পাঠিয়ে সাথে সাথে | `sleep 60 & echo চলছে` |
| `|` | একসাথে, stdout→stdin জোড়া | `ps aux | grep nginx` |

(`&` ও job control: [অধ্যায় ২৫](25-jobs-tmux.md)।)

---

## ১৯.১২ Subshell `( )` বনাম Group `{ }`

দুটোই কয়েকটা কমান্ডকে এক ইউনিট বানায় — যাতে একসাথে রিডাইরেক্ট বা চেইন করা যায়। পার্থক্য হলো **কোথায় চলে।**

### `( )` — subshell: আলাদা প্রসেসে

`( )`-এর ভেতরের কমান্ডগুলো একটা **child shell** (fork করা কপি)-এ চলে। ভেতরের পরিবর্তন (`cd`, ভেরিয়েবল, `export`, `umask`) বাইরে আসে না।

```
$ pwd
/home/rahim
$ (cd /tmp && ls | head -2)
systemd-private-abc
tmp.XyZ
$ pwd
/home/rahim           # বাইরে কিছু বদলায়নি
```

```
$ x=1; (x=99; echo "ভেতরে: $x"); echo "বাইরে: $x"
ভেতরে: 99
বাইরে: 1
```

উপমা: subshell হলো ফটোকপি করা খাতায় লেখা — আসল খাতা অক্ষত থাকে।

ব্যবহার:

```
$ (cd /var/www && tar czf - html) > /backup/site.tar.gz
$ (date; uptime; df -h /) > report.txt       # তিনটির আউটপুট এক ফাইলে
```

### `{ }` — group: বর্তমান shell-এই

```
$ { date; uptime; } > report.txt
```

এখানে নতুন প্রসেস তৈরি হয় না (দ্রুততর), আর ভেতরের পরিবর্তন বাইরে থেকে যায়:

```
$ x=1; { x=99; }; echo $x
99
$ { cd /tmp; }; pwd
/tmp
```

> ⚠️ সতর্কতা — syntax খুব কড়া:
> - `{` এর পরে **একটা স্পেস** লাগবেই (`{` হলো একটা reserved word, অপারেটর না)।
> - `}` এর আগে **`;` বা newline** লাগবেই।
> - `{date; uptime}` ❌ ভুল; `{ date; uptime; }` ✅ সঠিক।
> - `( )` এ এসব লাগে না: `(date;uptime)` ঠিক আছে।

### একটা কাজের প্যাটার্ন: ব্যর্থ হলে মেসেজ দিয়ে বেরিয়ে যাও

```bash
cd /important/dir || { echo "ফোল্ডারে যাওয়া যায়নি" >&2; exit 1; }
```

এখানে `( )` দিলে `exit 1` শুধু subshell থেকে বের হতো, স্ক্রিপ্ট চলতেই থাকত! তাই `{ }` দরকার।

| বিষয় | `( ... )` subshell | `{ ...; }` group |
|------|-------------------|-----------------|
| নতুন প্রসেস | হ্যাঁ (fork) | না |
| `cd`/ভেরিয়েবল বাইরে টেকে | না | হ্যাঁ |
| `exit` | শুধু subshell শেষ | পুরো shell/স্ক্রিপ্ট শেষ |
| গতি | একটু ধীর | দ্রুত |
| syntax | স্পেস/`;` ঐচ্ছিক | `{ ` স্পেস ও শেষে `;` বাধ্যতামূলক |

> 📝 মনে রাখো: `$BASHPID` দিয়ে দেখো কোন প্রসেসে আছ — `echo $BASHPID; (echo $BASHPID)` দুটো আলাদা সংখ্যা দেবে। (`$$` কিন্তু subshell-এও মূল shell-এর PID দেখায়।)

---

## ১৯.১৩ Command Substitution: `$( )` ও backtick

### কী করে

একটা কমান্ডের **আউটপুট** নিয়ে আরেকটা কমান্ডের **আর্গুমেন্ট** বা ভেরিয়েবলের মান হিসেবে বসিয়ে দেয়।

```
$ echo "আজ $(date +%d-%m-%Y)"
আজ 12-11-2026
$ today=$(date +%F)
$ echo $today
2026-11-12
$ mkdir backup-$(date +%F)
$ ls
backup-2026-11-12
$ echo "এই ফোল্ডারে $(ls | wc -l) টি ফাইল আছে"
এই ফোল্ডারে 7 টি ফাইল আছে
$ kill $(pgrep -f myscript.sh)
```

### Under the hood

shell একটা subshell fork করে, তার stdout একটা pipe-এ জুড়ে দেয়, পুরো আউটপুট পড়ে নেয়, **শেষের সব newline মুছে ফেলে**, তারপর সেই লেখা কমান্ড লাইনে বসায়।

```
$ x=$(printf 'hello\n\n\n'); echo "[$x]"
[hello]
```

### Backtick `` `...` `` — পুরনো রূপ

```
$ echo "আজ `date +%A`"
আজ Thursday
```

কাজ একই, কিন্তু `$( )` ভালো কারণ:

1. **Nesting সহজ:** `$(dirname $(which python3))` বনাম `` `dirname \`which python3\`` `` (ব্যাকস্ল্যাশ লাগে)
2. **পড়তে সহজ:** backtick ও single quote (`'`) গুলিয়ে যায়
3. ভেতরে ব্যাকস্ল্যাশের নিয়ম সরল

```
$ echo $(basename $(dirname /usr/local/bin/tool))
local
```

### Quote করার গুরুত্ব

```
$ files=$(ls)
$ echo $files          # quote ছাড়া: newline হারিয়ে সব এক লাইনে
a.txt b.txt c.txt
$ echo "$files"        # quote সহ: আসল রূপ রক্ষা পায়
a.txt
b.txt
c.txt
```

> ⚠️ সতর্কতা: quote ছাড়া `$( )` এর ফলাফল স্পেস দিয়ে ভেঙে যায় (word splitting) এবং `*` এর মতো wildcard expand হয় (globbing)। প্রায় সবসময় `"$(...)"` লেখো।

> 💡 টিপস: `$(< file)` হলো `$(cat file)`-এর দ্রুত bash রূপ — নতুন `cat` প্রসেস চালায় না: `content=$(< /etc/hostname)`।

> 📝 মনে রাখো: `$( )` (কমান্ড চালাও) আর `$(( ))` (গাণিতিক হিসাব) আলাদা: `echo $((5 + 3))` → `8`।

---

## ১৯.১৪ Process Substitution: `<( )` ও `>( )`

### সমস্যা

কিছু কমান্ড শুধু **ফাইলনাম** চায়, stdin না — যেমন `diff`, `comm`, `paste` দুটো ফাইল তুলনা করে। দুটো কমান্ডের আউটপুট তুলনা করতে চাইলে? temp file বানানো ঝামেলা।

```
$ ls dir1 > /tmp/a; ls dir2 > /tmp/b; diff /tmp/a /tmp/b; rm /tmp/a /tmp/b   # ঝামেলা
```

### সমাধান: `<(command)`

```
$ diff <(ls dir1) <(ls dir2)
3c3
< report.pdf
---
> report-final.pdf
```

`<(cmd)` জায়গায় shell একটা **ফাইলনাম** বসিয়ে দেয়, যেটা পড়লে `cmd`-এর আউটপুট পাওয়া যায়:

```
$ echo <(true)
/dev/fd/63
$ ls -l <(true)
lr-x------ 1 rahim rahim 64 Nov 12 10:30 /dev/fd/63 -> pipe:[123456]
```

### Under the hood

```
      ┌────────────┐   pipe   ┌──────────────────────────────────┐
      │  ls dir1   │─────────▶│ FD 63 → diff-এ "/dev/fd/63" নামে │
      └────────────┘          │                                  │
      ┌────────────┐   pipe   │          diff                    │
      │  ls dir2   │─────────▶│ FD 62 → "/dev/fd/62" নামে        │
      └────────────┘          └──────────────────────────────────┘
```

shell একটা pipe বানায়, `cmd`-কে background-এ চালিয়ে তার stdout pipe-এ জোড়ে, আর pipe-এর read end-কে child-এ একটা FD (যেমন 63) হিসেবে খোলা রাখে। `/dev/fd/63` পাথটা খুললে সেই pipe-ই পাওয়া যায়। `diff` ভাবে সে সাধারণ ফাইল খুলছে।

> 📝 মনে রাখো: এটা pipe, আসল ফাইল না — তাই শুধু **একবার শুরু থেকে শেষ পর্যন্ত** পড়া যায়। যে প্রোগ্রাম ফাইলে আগে-পিছে যায় (seek করে), সে ঠিকমতো কাজ নাও করতে পারে।

### আরও উদাহরণ

```
# দুটো সার্ভারের প্যাকেজ তালিকা তুলনা
$ diff <(ssh web1 'dpkg -l | awk "{print \$2}"') <(ssh web2 'dpkg -l | awk "{print \$2}"')

# শুধু প্রথম ফাইলে আছে এমন লাইন (comm-এর জন্য sorted ইনপুট লাগে)
$ comm -23 <(sort list1.txt) <(sort list2.txt)

# subshell সমস্যা ছাড়া while loop (১৯.৯ মনে করো)
$ count=0
$ while read -r line; do count=$((count+1)); done < <(ls /etc)
$ echo $count
231
```

শেষেরটায় খেয়াল করো: `< <(...)` — প্রথম `<` হলো ইনপুট রিডাইরেকশন, তারপর একটা **স্পেস**, তারপর `<(...)`। স্পেস ছাড়া `<<(` লিখলে heredoc ভেবে ভুল হবে।

### `>(command)` — আউটপুট দিকে

```
$ ls -l | tee >(grep '^d' > dirs.txt) >(grep '^-' > files.txt) > /dev/null
```

একটা আউটপুট একসাথে দুটো ফিল্টারে পাঠালাম — একটা ডিরেক্টরি আলাদা করছে, আরেকটা সাধারণ ফাইল।

```
$ tar czf >(ssh backup-server 'cat > site.tar.gz') /var/www
```

> ⚠️ সতর্কতা: process substitution bash/zsh/ksh-এর বৈশিষ্ট্য, POSIX `sh`/`dash`-এ নেই। `#!/bin/sh` স্ক্রিপ্টে ব্যবহার করলে `Syntax error: "(" unexpected` দেখবে।

---

## ১৯.১৫ Named Pipe (FIFO) — `mkfifo`

### সাধারণ pipe বনাম named pipe

`|` দিয়ে তৈরি pipe-এর কোনো নাম নেই (**anonymous pipe**) — শুধু একই shell থেকে শুরু হওয়া সম্পর্কিত প্রসেসগুলো ব্যবহার করতে পারে। **Named pipe** বা **FIFO** (First In, First Out) হলো ফাইলসিস্টেমে একটা **নাম** সহ pipe — যেকোনো দুটো অসম্পর্কিত প্রসেস, এমনকি আলাদা টার্মিনাল থেকে, এটা দিয়ে কথা বলতে পারে।

উপমা: anonymous pipe হলো দুই ভাইয়ের মধ্যে হাতে হাতে চিঠি দেওয়া; named pipe হলো বাড়ির সামনে একটা ডাকবাক্স — যে-কেউ চিঠি রাখতে পারে, যে-কেউ নিতে পারে।

**Syntax:** `mkfifo [-m MODE] NAME...`

| অপশন | কাজ |
|------|-----|
| `-m MODE`, `--mode=MODE` | পারমিশন সেট করো (যেমন `-m 600`) |

```
$ mkfifo /tmp/mypipe
$ ls -l /tmp/mypipe
prw-r--r-- 1 rahim rahim 0 Nov 12 10:40 /tmp/mypipe
```

প্রথম অক্ষর `p` = pipe। সাইজ সবসময় 0 — কারণ ডেটা ডিস্কে থাকে না, kernel-এর pipe buffer-এ থাকে। ফাইলসিস্টেমে শুধু "নাম" আর inode আছে।

### হাতে-কলমে: দুই টার্মিনালে

**টার্মিনাল ১:**

```
$ cat /tmp/mypipe
```

(এখানে আটকে থাকবে — লেখক আসার অপেক্ষায়)

**টার্মিনাল ২:**

```
$ echo "হ্যালো অন্য টার্মিনাল!" > /tmp/mypipe
```

**টার্মিনাল ১-এ এখন দেখাবে:**

```
হ্যালো অন্য টার্মিনাল!
$
```

### Blocking আচরণ

FIFO খোলার সময়:
- **পড়ার জন্য** খুললে — কেউ লেখার জন্য না খোলা পর্যন্ত `open()` block হয়।
- **লেখার জন্য** খুললে — কেউ পড়ার জন্য না খোলা পর্যন্ত block হয়।

দুই পক্ষ "দেখা না হওয়া পর্যন্ত" কেউ এগোয় না — একে বলে rendezvous।

### বাস্তব ব্যবহার

```
# একটা প্রসেসের লগ লাইভ আরেকটা প্রসেসে প্রসেস করা
$ mkfifo /tmp/logpipe
$ grep --line-buffered ERROR < /tmp/logpipe > errors.txt &
$ ./my_app > /tmp/logpipe

# একটা কমান্ডের আউটপুট একসাথে কম্প্রেস ও চেকসাম
$ mkfifo /tmp/p
$ sha256sum < /tmp/p > checksum.txt &
$ tar cf - mydir | tee /tmp/p | gzip > mydir.tar.gz
$ wait
$ rm /tmp/p
```

FIFO সাধারণ ফাইলের মতোই `rm` দিয়ে মোছা যায়। (`mknod name p` ও একই কাজ করে, তবে `mkfifo` পছন্দনীয়।)

| | Anonymous pipe (`|`) | Named pipe (FIFO) |
|--|---------------------|------------------|
| তৈরি | shell স্বয়ংক্রিয়ভাবে | `mkfifo` |
| ফাইলসিস্টেমে নাম | নেই | আছে (`p` টাইপ) |
| কারা ব্যবহার করে | সম্পর্কিত প্রসেস (একই পাইপলাইন) | যেকোনো প্রসেস, পারমিশন থাকলে |
| আয়ু | প্রসেস শেষ হলে শেষ | `rm` না করা পর্যন্ত নাম থাকে |
| ডেটা কোথায় | kernel buffer | kernel buffer (ডিস্কে না) |

---

## ১৯.১৬ `exec` দিয়ে স্থায়ী রিডাইরেকশন ও নিজস্ব FD

কোনো কমান্ড ছাড়া `exec` এর সাথে রিডাইরেকশন দিলে সেটা **বর্তমান shell-এর** জন্য স্থায়ীভাবে প্রযোজ্য হয়।

```bash
#!/bin/bash
# স্ক্রিপ্টের বাকি সব আউটপুট ও এরর লগ ফাইলে
exec > /var/log/myscript.log 2>&1
echo "এটা লগ ফাইলে যাবে"
ls /nai          # এররও লগে যাবে
```

স্ক্রিনেও দেখতে চাইলে process substitution আর tee মিলিয়ে:

```bash
exec > >(tee -a /var/log/myscript.log) 2>&1
```

### নিজের FD খোলা ও বন্ধ করা

FD 3–9 তুমি নিজের কাজে ব্যবহার করতে পারো (bash 10 ও তার বেশি নম্বরও সমর্থন করে, তবে shell নিজেও সেগুলো ব্যবহার করে, তাই সাবধানে)।

```
$ exec 3> /tmp/out.txt        # FD 3 লেখার জন্য খোলা
$ echo "প্রথম" >&3
$ echo "দ্বিতীয়" >&3
$ exec 3>&-                   # FD 3 বন্ধ
$ cat /tmp/out.txt
প্রথম
দ্বিতীয়

$ exec 4< /etc/hostname       # FD 4 পড়ার জন্য খোলা
$ read -r line <&4
$ echo "$line"
myserver
$ exec 4<&-
```

bash 4.1+ তে স্বয়ংক্রিয় FD নম্বর নেওয়া যায়:

```
$ exec {myfd}> /tmp/auto.txt
$ echo $myfd
10
$ echo "hi" >&$myfd
$ exec {myfd}>&-
```

### মজার কৌশল: bash-এর `/dev/tcp`

bash নিজে (kernel না) `/dev/tcp/HOST/PORT` পাথকে বিশেষভাবে চেনে এবং একটা TCP সংযোগ খোলে:

```
$ exec 3<> /dev/tcp/example.com/80
$ printf 'HEAD / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n' >&3
$ head -1 <&3
HTTP/1.1 200 OK
$ exec 3<&-
```

> 📝 মনে রাখো: `/dev/tcp` আসলে ডিস্কে নেই (`ls /dev/tcp` এরর দেবে); এটা শুধু bash-এর রিডাইরেকশনের ভেতরে কাজ করে। পোর্ট খোলা কিনা দ্রুত দেখতে কাজের: `timeout 2 bash -c '</dev/tcp/10.0.0.5/22' && echo open`।

---

## ১৯.১৭ Exit Status `$?`

### কী

প্রতিটি কমান্ড শেষ হওয়ার সময় kernel-এর মাধ্যমে parent-কে একটা 0–255 সংখ্যা জানায়। bash সর্বশেষ কমান্ডের এই সংখ্যা `$?` ভেরিয়েবলে রাখে।

```
$ ls /etc > /dev/null
$ echo $?
0
$ ls /nai
ls: cannot access '/nai': No such file or directory
$ echo $?
2
```

> 📝 মনে রাখো: সাধারণ যুক্তির উল্টো — **0 = সফল (সত্য)**, অশূন্য = ব্যর্থ (মিথ্যা)। কারণ সফল হওয়ার একটাই উপায়, কিন্তু ব্যর্থ হওয়ার অনেক কারণ থাকতে পারে, আর অশূন্য সংখ্যাগুলো কারণ বলে দেয়।

### প্রচলিত exit code

| কোড | অর্থ |
|-----|------|
| `0` | সফল |
| `1` | সাধারণ এরর (যেমন `grep` কিছু না পেলে) |
| `2` | ভুল ব্যবহার / গুরুতর সমস্যা (bash builtin-এর ভুল syntax; `ls`-এ গুরুতর সমস্যা; `grep`-এ এরর) |
| `126` | কমান্ড পাওয়া গেছে কিন্তু চালানো যায় না (execute পারমিশন নেই) |
| `127` | কমান্ড পাওয়া যায়নি (command not found) |
| `128+N` | সিগন্যাল N দিয়ে মারা গেছে |
| `130` | = 128+2, Ctrl+C (SIGINT) |
| `137` | = 128+9, SIGKILL (যেমন OOM killer বা `kill -9`) |
| `143` | = 128+15, SIGTERM |

```
$ nosuchcommand
nosuchcommand: command not found
$ echo $?
127
$ sleep 100
^C
$ echo $?
130
```

> ⚠️ সতর্কতা: প্রতিটি প্রোগ্রাম নিজের অর্থ ঠিক করে। যেমন `grep`: `0` = মিলেছে, `1` = মেলেনি, `2` = এরর। `diff`: `0` = একই, `1` = পার্থক্য আছে, `2` = সমস্যা। সন্দেহ হলে `man` পেজের "EXIT STATUS" অংশ পড়ো।

### `$?` একবারই বাঁচে

```
$ false
$ echo $?
1
$ echo $?
0          # আগের echo সফল হয়েছিল!
```

দরকার হলে সাথে সাথে সংরক্ষণ করো: `cmd; rc=$?`।

### নিজের exit status দেওয়া

```bash
#!/bin/bash
if [ ! -f "$1" ]; then
    echo "ফাইল নেই: $1" >&2
    exit 1
fi
echo "ঠিক আছে"
exit 0
```

`exit` এ 255-এর বেশি দিলে 256 দিয়ে ভাগশেষ নেওয়া হয় (`exit 256` → `0`!)। ফাংশন থেকে `return N` একইভাবে কাজ করে ([অধ্যায় ৪২](42-shell-scripting-2.md))।

### `$?` না দেখে সরাসরি কমান্ড পরীক্ষা

```bash
# কম ভালো
grep -q error app.log
if [ $? -eq 0 ]; then echo "এরর আছে"; fi

# ভালো
if grep -q error app.log; then echo "এরর আছে"; fi
```

`if` নিজেই কমান্ডের exit status দেখে।

### `!` — status উল্টানো

```
$ ! grep -q rahim /etc/passwd && echo "ইউজার নেই"
```

---

## ১৯.১৮ সব একসাথে: বাস্তব পাইপলাইন

```
# সবচেয়ে বেশি মেমরি ব্যবহারকারী ৫টি প্রসেস
$ ps aux --sort=-%mem | head -6

# লগে কোন IP সবচেয়ে বেশি রিকোয়েস্ট পাঠিয়েছে
$ awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -5
   1523 203.0.113.7
    842 198.51.100.23
    ...

# কোন ফোল্ডার সবচেয়ে বেশি জায়গা নিচ্ছে
$ du -sh /var/* 2>/dev/null | sort -rh | head -5

# ব্যাকআপ: সফল হলে লগে লেখো, ব্যর্থ হলে এরর লগে
$ tar czf /backup/home-$(date +%F).tar.gz /home 2>/tmp/tar.err \
    && echo "$(date): ব্যাকআপ সফল" >> /var/log/backup.log \
    || { echo "$(date): ব্যাকআপ ব্যর্থ" >&2; cat /tmp/tar.err >&2; }
```

(`sort`, `uniq`, `awk` এর বিস্তারিত পরের অধ্যায়গুলোতে — [অধ্যায় ২০](20-grep-regex.md), [২২](22-awk.md), [২৩](23-text-tools.md)।)

> 💡 টিপস: লম্বা পাইপলাইন ধাপে ধাপে বানাও। প্রথম কমান্ড চালাও, আউটপুট দেখো; তারপর `↑` চেপে `| পরের_কমান্ড` যোগ করো। প্রতিটি ধাপে দেখো ডেটা কেমন দেখাচ্ছে।

> 💡 টিপস: লাইন শেষে `|`, `&&` বা `||` থাকলে bash নিজেই বোঝে কমান্ড শেষ হয়নি — `\` ছাড়াই পরের লাইনে লেখা যায়।

---

## ১৯.১৯ সম্পর্কিত কীবোর্ড শর্টকাট

| শর্টকাট | কাজ | এই অধ্যায়ে প্রাসঙ্গিকতা |
|---------|-----|------------------------|
| `Ctrl+D` | stdin-এ EOF পাঠাও (লাইনের শুরুতে) | `cat > file` বা `wc` চালিয়ে হাতে টাইপ শেষ করতে |
| `Ctrl+C` | foreground প্রসেস/পাইপলাইনে SIGINT | আটকে থাকা পাইপ বা FIFO রিডার থামাতে (exit 130) |
| `Ctrl+Z` | foreground পাইপলাইন সাময়িক বন্ধ (SIGTSTP) | পরে `bg`/`fg` |
| `Ctrl+\` | SIGQUIT পাঠাও | Ctrl+C কাজ না করলে |
| `Ctrl+S` / `Ctrl+Q` | টার্মিনাল আউটপুট থামাও / আবার চালাও | দ্রুত চলা আউটপুট থামিয়ে পড়তে (ভুলে Ctrl+S চাপলে টার্মিনাল "জমে" যায় — Ctrl+Q চাপো) |
| `Alt+.` | আগের কমান্ডের শেষ আর্গুমেন্ট বসাও | `ls > out.txt` এর পর `cat ` + `Alt+.` → `out.txt` |
| `Ctrl+X Ctrl+E` | বর্তমান কমান্ড এডিটরে খোলো | লম্বা পাইপলাইন আরামে লিখতে |
| `↑` / `Ctrl+P` | আগের কমান্ড | পাইপলাইন ধাপে ধাপে বাড়াতে |
| `Ctrl+L` | স্ক্রিন পরিষ্কার | |
| `Shift+\` | `|` টাইপ (US কীবোর্ড) | |

পূর্ণ তালিকা: [অধ্যায় ৬](06-terminal-shortcuts.md) ও [পরিশিষ্ট ক](appendix-a-shortcuts-cheatsheet.md)।

```
$ cat > shopping.txt
চাল
ডাল
তেল
^D
$ cat shopping.txt
চাল
ডাল
তেল
```

(`^D` মানে Ctrl+D চেপেছ — স্ক্রিনে সাধারণত দেখা যায় না।)

---

## হাতে-কলমে অনুশীলন

একটা অনুশীলন ফোল্ডার বানাও:

```
$ mkdir -p ~/lab19 && cd ~/lab19
```

### ল্যাব ১: stdout ও stderr আলাদা করা

```
$ ls /etc/hostname /nai > out.txt 2> err.txt
$ cat out.txt
/etc/hostname
$ cat err.txt
ls: cannot access '/nai': No such file or directory
```

এখন ক্রমের পার্থক্য নিজে দেখো:

```
$ ls /etc/hostname /nai > both1.txt 2>&1
$ ls /etc/hostname /nai 2>&1 > both2.txt
ls: cannot access '/nai': No such file or directory
$ wc -l both1.txt both2.txt
 2 both1.txt
 1 both2.txt
 3 total
```

### ল্যাব ২: ওভাররাইট বনাম append এবং noclobber

```
$ echo one > f.txt; echo two > f.txt; cat f.txt
two
$ echo three >> f.txt; cat f.txt
two
three
$ set -o noclobber
$ echo four > f.txt
bash: f.txt: cannot overwrite existing file
$ echo four >| f.txt; cat f.txt
four
$ set +o noclobber
```

### ল্যাব ৩: FD টেবিল নিজের চোখে

```
$ ls -l /proc/$$/fd
$ exec 3> lab.txt
$ ls -l /proc/$$/fd | grep ' 3 '
l-wx------ 1 rahim rahim 64 Nov 12 11:00 3 -> /home/rahim/lab19/lab.txt
$ echo "FD 3 দিয়ে লেখা" >&3
$ exec 3>&-
$ cat lab.txt
FD 3 দিয়ে লেখা
```

### ল্যাব ৪: pipe concurrency ও SIGPIPE

```
$ time (sleep 2 | sleep 2)
real    0m2.00s
...
$ yes "লিনাক্স" | head -n 3
লিনাক্স
লিনাক্স
লিনাক্স
$ yes | head -1; echo "${PIPESTATUS[@]}"
y
141 0
```

`141` = 128 + 13 (SIGPIPE) — `yes` SIGPIPE-এ মারা গেছে।

### ল্যাব ৫: pipefail

```
$ ls /nai | wc -l; echo "status=$?"
ls: cannot access '/nai': No such file or directory
0
status=0
$ set -o pipefail
$ ls /nai | wc -l; echo "status=$?"
ls: cannot access '/nai': No such file or directory
0
status=2
$ set +o pipefail
```

### ল্যাব ৬: heredoc দিয়ে স্ক্রিপ্ট তৈরি

```
$ cat > greet.sh <<'EOF'
#!/bin/bash
name=${1:-বন্ধু}
echo "স্বাগতম, $name!"
EOF
$ chmod +x greet.sh
$ ./greet.sh রহিম
স্বাগতম, রহিম!
```

এখন `'EOF'` এর বদলে `EOF` (quote ছাড়া) দিয়ে আবার বানাও এবং `cat greet.sh` করে পার্থক্য দেখো।

### ল্যাব ৭: tee ও sudo tee

```
$ ls /etc | tee etc.txt | wc -l
$ head -3 etc.txt
$ echo "127.0.0.1 mylab.local" | sudo tee -a /etc/hosts > /dev/null
$ grep mylab /etc/hosts
127.0.0.1 mylab.local
```

(পরে মুছে দিতে: `sudo sed -i '/mylab.local/d' /etc/hosts` — sed: [অধ্যায় ২১](21-sed.md)।)

### ল্যাব ৮: subshell বনাম group

```
$ cd ~/lab19
$ (cd /tmp; pwd); pwd
/tmp
/home/rahim/lab19
$ { cd /tmp; pwd; }; pwd
/tmp
/tmp
$ cd ~/lab19
```

### ল্যাব ৯: command ও process substitution

```
$ echo "এখানে $(ls | wc -l) টি আইটেম, আজ $(date +%A)"
$ mkdir a b; touch a/{1,2,3}.txt b/{2,3,4}.txt
$ diff <(ls a) <(ls b)
1d0
< 1.txt
3a3
> 4.txt
$ comm -12 <(ls a) <(ls b)
2.txt
3.txt
```

### ল্যাব ১০: named pipe দুই টার্মিনালে

টার্মিনাল ১:
```
$ mkfifo ~/lab19/chat
$ while true; do cat ~/lab19/chat; done
```
টার্মিনাল ২:
```
$ echo "কেমন আছ?" > ~/lab19/chat
$ date > ~/lab19/chat
```
টার্মিনাল ১-এ মেসেজ আসবে। শেষে টার্মিনাল ১-এ `Ctrl+C`, তারপর `rm ~/lab19/chat`।

### ল্যাব ১১: exit status শিকার

```
$ true; echo $?
0
$ false; echo $?
1
$ grep -q root /etc/passwd; echo $?
0
$ grep -q nobodyxyz /etc/passwd; echo $?
1
$ /etc/passwd; echo $?
bash: /etc/passwd: Permission denied
126
$ nocmd; echo $?
nocmd: command not found
127
```

### চ্যালেঞ্জ

একটা এক-লাইনের কমান্ড লেখো যা: `/etc` এর নিচে `.conf` ফাইল খুঁজবে, Permission denied এরর লুকাবে, তালিকাটা `conf-list.txt`-এ রাখবে **এবং** স্ক্রিনে মোট সংখ্যা দেখাবে, আর `find` ব্যর্থ হলেও তুমি তা জানতে পারবে।

<details><summary>একটা সম্ভাব্য সমাধান</summary>

```
$ set -o pipefail
$ find /etc -name '*.conf' 2>/dev/null | tee conf-list.txt | wc -l || echo "কিছু সমস্যা হয়েছে" >&2
```

লক্ষ করো: Permission denied থাকলে `find` নিজেই অশূন্য status দেয়, তাই `pipefail` চালু থাকলে এখানে "কিছু সমস্যা হয়েছে" দেখা যেতে পারে — এটাই প্রমাণ যে `2>/dev/null` এরর লুকালেও exit status লুকায় না।
</details>

---

## সারসংক্ষেপ

| বিষয় | Syntax | এক কথায় |
|------|--------|---------|
| stdout ফাইলে | `cmd > f` | ওভাররাইট |
| stdout append | `cmd >> f` | শেষে যোগ |
| stdin ফাইল থেকে | `cmd < f` | ইনপুট বদল |
| stderr ফাইলে | `cmd 2> f` | শুধু এরর |
| দুটোই এক ফাইলে | `cmd > f 2>&1` / `cmd &> f` | ক্রম জরুরি |
| দুটোই append | `cmd >> f 2>&1` / `cmd &>> f` | |
| নিঃশব্দ | `cmd > /dev/null 2>&1` | সব ফেলে দাও |
| মেসেজ stderr-এ | `echo msg >&2` | স্ক্রিপ্টে এরর |
| noclobber | `set -o noclobber`, `>|` | দুর্ঘটনা রোধ |
| here document | `cmd <<EOF ... EOF` | বহু-লাইন ইনপুট |
| আক্ষরিক heredoc | `cmd <<'EOF'` | expansion বন্ধ |
| ট্যাব-মোছা heredoc | `cmd <<-EOF` | শুধু TAB |
| here string | `cmd <<< "text"` | এক-লাইন ইনপুট |
| পাইপ | `a | b` | stdout→stdin |
| stderr সহ পাইপ | `a 2>&1 | b` / `a |& b` | |
| পাইপ status | `${PIPESTATUS[@]}`, `set -o pipefail` | |
| T-জয়েন্ট | `cmd | tee [-a] f` | দেখো ও রাখো |
| root ফাইলে লেখা | `echo x | sudo tee f > /dev/null` | |
| ক্রম | `a ; b` | সবসময় |
| সফল হলে | `a && b` | |
| ব্যর্থ হলে | `a || b` | |
| subshell | `( a; b )` | আলাদা প্রসেস |
| group | `{ a; b; }` | একই shell |
| command substitution | `"$(cmd)"`, `` `cmd` `` | আউটপুট বসাও |
| process substitution | `<(cmd)`, `>(cmd)` | আউটপুটকে ফাইলনাম বানাও |
| named pipe | `mkfifo name` | নামওয়ালা pipe |
| স্থায়ী রিডাইরেকশন | `exec > log 2>&1` | পুরো স্ক্রিপ্টে |
| নিজের FD | `exec 3> f`, `>&3`, `exec 3>&-` | |
| exit status | `$?` | 0 = সফল |
| EOF পাঠানো | `Ctrl+D` | |

---

## নিজেকে যাচাই করো

1. stdin, stdout ও stderr এর FD নম্বর কত, এবং stdout ও stderr আলাদা রাখার যুক্তি কী?
2. `cmd > file 2>&1` আর `cmd 2>&1 > file` এর মধ্যে পার্থক্য কী? কেন?
3. `sort data.txt > data.txt` চালানোর পর `data.txt` খালি কেন হয়?
4. `sudo echo "x" > /etc/somefile` কেন "Permission denied" দেয়, আর সমাধান কী?
5. `cmd1 | cmd2` তে buffer ভর্তি হলে কী ঘটে? আর `cmd2` আগে বেরিয়ে গেলে `cmd1`-এর কী হয়?
6. `false | true; echo $?` কী দেখাবে? কীভাবে `false`-এর ব্যর্থতা ধরা যায়?
7. `cat <<EOF` আর `cat <<'EOF'` এর পার্থক্য কী?
8. `( cd /tmp )` আর `{ cd /tmp; }` চালানোর পর তুমি কোন ফোল্ডারে থাকবে?
9. `diff <(ls a) <(ls b)` তে `diff` আসলে কী ফাইলনাম পায়?
10. exit status `127`, `126` ও `130` এর মানে কী?

<details><summary>উত্তর দেখো</summary>

1. stdin = 0, stdout = 1, stderr = 2। আলাদা রাখলে ডেটা (stdout) ফাইলে বা পাইপে পাঠানোর সময় এরর মেসেজ তাতে মিশে যায় না, আর এরর আলাদাভাবে দেখা/লগ করা যায়।
2. প্রথমটায় আগে stdout ফাইলে যায়, তারপর stderr-কে stdout-এর (এখন ফাইলের) কপি বানানো হয় — দুটোই ফাইলে। দ্বিতীয়টায় stderr আগে stdout-এর তখনকার গন্তব্য (টার্মিনাল) কপি করে, পরে শুধু stdout ফাইলে যায় — এরর টার্মিনালেই থাকে। কারণ রিডাইরেকশন বাম থেকে ডানে প্রয়োগ হয় এবং `2>&1` একটা মুহূর্তের কপি (`dup2`)।
3. shell `sort` চালু করার আগেই `>` দিয়ে `data.txt` খোলে ও truncate করে; তাই `sort` খালি ফাইল পড়ে। সমাধান: `sort -o data.txt data.txt` বা অন্য ফাইলে লিখে `mv`।
4. `sudo` শুধু `echo`-কে root হিসেবে চালায়, কিন্তু `>` রিডাইরেকশনটা করে সাধারণ ইউজারের shell। সমাধান: `echo "x" | sudo tee /etc/somefile > /dev/null` (বা `-a` দিয়ে append), অথবা `sudo sh -c 'echo x > /etc/somefile'`।
5. buffer (Linux-এ ডিফল্ট 64 KiB) ভর্তি হলে `cmd1` `write()`-এ block হয়ে অপেক্ষা করে যতক্ষণ না `cmd2` পড়ে জায়গা খালি করে। `cmd2` বেরিয়ে গেলে (read end বন্ধ) `cmd1` পরের write-এ SIGPIPE পায় এবং সাধারণত শেষ হয়ে যায় (status 141)।
6. `0` দেখাবে, কারণ পাইপলাইনের status হলো শেষ কমান্ডের। ধরতে: `echo "${PIPESTATUS[@]}"` (দেখাবে `1 0`) অথবা আগে `set -o pipefail`।
7. quote ছাড়া `EOF`-এ ভেতরের `$VAR`, `$(cmd)`, `` `cmd` `` ইত্যাদি expand হয়; `'EOF'` (quoted) হলে সব আক্ষরিকভাবে থাকে।
8. `( cd /tmp )` এর পর আগের ফোল্ডারেই থাকবে (subshell-এর পরিবর্তন বাইরে আসে না); `{ cd /tmp; }` এর পর `/tmp`-এ থাকবে (একই shell)।
9. `/dev/fd/63` ও `/dev/fd/62` জাতীয় পাথ — যা আসলে pipe-এর read end; পড়লে `ls a` ও `ls b` এর আউটপুট পাওয়া যায়।
10. `127` = কমান্ড পাওয়া যায়নি; `126` = পাওয়া গেছে কিন্তু চালানো যায় না (execute পারমিশন নেই বা ডিরেক্টরি); `130` = 128+2, অর্থাৎ SIGINT (Ctrl+C) দিয়ে থামানো হয়েছে।

</details>

---

[← আগের অধ্যায়](18-sudo-root.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](20-grep-regex.md)

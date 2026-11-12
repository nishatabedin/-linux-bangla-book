# অধ্যায় ৬: টার্মিনাল শর্টকাট ও কমান্ড হিস্ট্রি — দ্রুত কাজের চাবিকাঠি

> একজন নতুন ইউজার আর একজন লিনাক্স এক্সপার্টের মধ্যে সবচেয়ে চোখে পড়ার মতো পার্থক্য কী জানো? এক্সপার্ট কম টাইপ করে। সে জানে কীভাবে আগের কমান্ড ফিরিয়ে আনতে হয়, কীভাবে এক চাপে পুরো লাইন মুছতে হয়, কীভাবে লম্বা ফাইলের নাম Tab চেপে পূরণ করতে হয়। এই অধ্যায়ে আমরা ঠিক সেই "জাদু"গুলো শিখব, আর দেখব ভেতরে আসলে কী ঘটে।

## এই অধ্যায়ে যা শিখবে

- কীবোর্ডে একটা কী চাপলে সেটা টার্মিনাল থেকে শেল পর্যন্ত কীভাবে পৌঁছায় (terminal emulator → pty → line discipline → readline → bash)
- **GNU Readline** কী, আর কেন bash-এর প্রায় সব এডিটিং শর্টকাট আসলে readline-এর
- কার্সর চলাফেরার শর্টকাট: `Ctrl+A`, `Ctrl+E`, `Alt+B`, `Alt+F` ইত্যাদি
- লেখা মোছা, কাটা আর পেস্ট করা (kill ring): `Ctrl+U`, `Ctrl+K`, `Ctrl+W`, `Alt+D`, `Ctrl+Y`, `Alt+Y`
- বড়/ছোট হাতের অক্ষর বদলানো ও অদলবদল: `Alt+U`, `Alt+L`, `Alt+C`, `Ctrl+T`, `Alt+T`
- কন্ট্রোল শর্টকাট: `Ctrl+C`, `Ctrl+D`, `Ctrl+Z`, `Ctrl+L`, `Ctrl+S`/`Ctrl+Q` — এবং এগুলো আসলে signal নাকি readline-এর কাজ
- Tab completion, double Tab, আর `bash-completion` প্যাকেজ
- `history` কমান্ড, `~/.bash_history` ফাইল, আর history কীভাবে সেভ হয়
- History expansion: `!!`, `!n`, `!-n`, `!string`, `!?string?`, `!$`, `!^`, `!*`, `^old^new`, modifiers
- `Ctrl+R` reverse search — একদম খুঁটিনাটি
- `HISTSIZE`, `HISTFILESIZE`, `HISTCONTROL`, `HISTIGNORE`, `HISTTIMEFORMAT`, `histappend`
- `fc` কমান্ড দিয়ে আগের কমান্ড এডিটরে খুলে ঠিক করা
- Terminal emulator-এর নিজস্ব শর্টকাট (`Ctrl+Shift+C/V/T/N` ইত্যাদি)
- emacs mode বনাম vi mode (`set -o vi`)
- `~/.inputrc` দিয়ে readline কাস্টমাইজ করা, আর `bind` কমান্ড

---

## ৬.১ কেন শর্টকাট শিখব?

ধরো তুমি এই লম্বা কমান্ডটা টাইপ করলে:

```bash
$ sudo systemctl restart nginx.servcie
```

Enter চাপার পর দেখলে বানান ভুল — `servcie`। নতুন ইউজার কী করে? পুরো লাইন আবার টাইপ করে। এক্সপার্ট কী করে?

```bash
$ ^servcie^service
sudo systemctl restart nginx.service
```

মাত্র ১৭টা অক্ষর! আবার ধরো ২০ মিনিট আগে একটা লম্বা `rsync` কমান্ড চালিয়েছিলে। এক্সপার্ট `Ctrl+R` চেপে `rsync` লিখবে — কমান্ডটা সামনে চলে আসবে।

দিনে হাজারবার কমান্ড লিখলে প্রতিবার ৫ সেকেন্ড বাঁচানো মানে দিনে প্রায় দেড় ঘণ্টা! তাছাড়া কম টাইপ মানে কম ভুল। সার্ভারে কাজ করার সময় একটা ভুল অক্ষর অনেক বড় বিপদ ডেকে আনতে পারে।

> 💡 **টিপস:** এই অধ্যায়ের সব শর্টকাট একসাথে মুখস্থ করার চেষ্টা কোরো না। প্রথম সপ্তাহে শুধু `Ctrl+A`, `Ctrl+E`, `Ctrl+U`, `Ctrl+R`, `Tab` আর `!!` ব্যবহার করো। আঙুলে বসে গেলে পরের সপ্তাহে আরও ৫টা যোগ করো।

---

## ৬.২ ভেতরে কী ঘটে — কীবোর্ড থেকে শেল পর্যন্ত

শর্টকাটগুলো কেন এভাবে আচরণ করে, সেটা বুঝতে হলে আগে জানতে হবে একটা কী চাপলে সেটা কোথায় কোথায় যায়। [অধ্যায় ৫](05-terminal-o-shell.md)-এ আমরা terminal, shell আর console-এর পার্থক্য দেখেছি। এখন একটু গভীরে যাই।

```
 ┌──────────────┐
 │   কীবোর্ড     │  তুমি Ctrl+Shift+C বা Ctrl+A চাপলে
 └──────┬───────┘
        │ (X11 / Wayland ইভেন্ট)
        ▼
 ┌──────────────────────────────┐
 │ Terminal Emulator            │  ← ধাপ ১: GNOME Terminal / Konsole / Ptyxis
 │ (GUI প্রোগ্রাম)                │    Ctrl+Shift+C/V/T/N এখানেই ধরা পড়ে,
 │                              │    শেল পর্যন্ত যায়ই না
 └──────┬───────────────────────┘
        │ বাকি কী → বাইট হিসেবে লেখা হয় (Ctrl+A = 0x01)
        ▼
 ┌──────────────────────────────┐
 │ PTY master  ⇄  PTY slave      │  ← pseudo-terminal জোড়া (/dev/pts/N)
 │      Line Discipline (tty)   │  ← ধাপ ২: Ctrl+C → SIGINT, Ctrl+Z → SIGTSTP,
 │                              │    Ctrl+\ → SIGQUIT, Ctrl+S/Q → flow control
 └──────┬───────────────────────┘
        │ বাকি বাইট
        ▼
 ┌──────────────────────────────┐
 │ bash                         │
 │  └─ GNU Readline লাইব্রেরি     │  ← ধাপ ৩: Ctrl+A/E/U/K/W/R, Alt+B/F, Tab...
 │       (লাইন এডিটর)             │    সব এডিটিং এখানে
 │  └─ History expansion        │  ← ধাপ ৪: Enter চাপার পর !!, !$, ^a^b
 └──────────────────────────────┘
```

তাহলে শর্টকাট মোটামুটি তিন জায়গায় ধরা পড়ে:

| স্তর | কারা ধরে | উদাহরণ |
|------|----------|---------|
| Terminal emulator | GUI অ্যাপ নিজে | `Ctrl+Shift+C`, `Ctrl+Shift+T`, `F11` |
| Kernel-এর tty line discipline | কার্নেল | `Ctrl+C`, `Ctrl+Z`, `Ctrl+\`, `Ctrl+S`, `Ctrl+Q` |
| Readline (bash-এর ভেতরে) | bash প্রসেস | `Ctrl+A`, `Ctrl+R`, `Alt+.`, `Tab` |

### উপমা: রেস্টুরেন্টের অর্ডার

ধরো তুমি রেস্টুরেন্টে অর্ডার দিচ্ছ।
- **ওয়েটার** (terminal emulator) কিছু কথা নিজেই সামলায় — "পানি দিন", "টেবিল মুছে দিন"। এগুলো রান্নাঘরে যায়ই না (যেমন `Ctrl+Shift+T` নতুন ট্যাব)।
- **ম্যানেজার** (tty line discipline) কিছু জরুরি নির্দেশ সামলায় — "অর্ডার বাতিল!" — সে সরাসরি রান্নাঘরে গিয়ে রাঁধুনিকে থামিয়ে দেয় (যেমন `Ctrl+C` = SIGINT)।
- **অর্ডার-প্যাড** (readline) — তুমি অর্ডার লিখছ, কাটছ, ঠিক করছ। চূড়ান্ত না হওয়া পর্যন্ত (Enter) রাঁধুনি (bash) কিছুই রাঁধে না।

### নিজে দেখো: `stty -a`

কোন Ctrl-কী কোন tty কাজে বাঁধা, তা দেখতে:

```bash
$ stty -a
speed 38400 baud; rows 40; columns 120; line = 0;
intr = ^C; quit = ^\; erase = ^?; kill = ^U; eof = ^D; eol = <undef>;
eol2 = <undef>; swtch = <undef>; start = ^Q; stop = ^S; susp = ^Z; rprnt = ^R;
werase = ^W; lnext = ^V; discard = ^O; min = 1; time = 0;
...
```

এখানে `^C` মানে `Ctrl+C`। `intr = ^C` মানে Ctrl+C চাপলে interrupt signal (SIGINT) যাবে। `susp = ^Z` মানে suspend।

> 📝 **মনে রাখো:** Readline লাইন এডিট করার সময় tty-কে প্রায় "raw" মোডে রাখে, তাই `^U`, `^W`-এর মতো কী readline নিজেই সামলায় (`stty`-র সেটিং দেখে)। কিন্তু `Ctrl+C`, `Ctrl+Z` সবসময় কার্নেলই signal-এ রূপান্তর করে — তাই এগুলো যেকোনো প্রোগ্রামে কাজ করে, শুধু bash-এ নয়।

### GNU Readline কী?

**Readline** হলো একটা লাইব্রেরি (অনেক প্রোগ্রাম মিলে ব্যবহার করা কোডের সংগ্রহ), যেটা "একটা লাইন ইনপুট নাও, আর ইউজারকে সেটা আরামে এডিট করতে দাও" — এই কাজটা করে। bash ছাড়াও `python3` REPL, `mysql`, `psql`, `gdb`, `sqlite3`-এর মতো অনেক প্রোগ্রাম readline ব্যবহার করে। তাই এই অধ্যায়ে যা শিখবে, সেগুলো ওই প্রোগ্রামগুলোতেও প্রায় হুবহু কাজ করবে!

Readline-এর ডিফল্ট কী-বাইন্ডিং এসেছে **Emacs** এডিটর থেকে। তাই একে বলে **emacs mode**। চাইলে **vi mode**-ও চালু করা যায় (৬.১১ অংশে দেখব)।

### "Meta" কী আর Alt

Readline-এর ডকুমেন্টেশনে `M-f`, `M-b` লেখা থাকে। **M** মানে **Meta**। আজকের কীবোর্ডে Meta = `Alt`। আর `C-a` মানে `Ctrl+A`।

| ডকুমেন্টেশনে লেখা | কীবোর্ডে চাপো |
|---|---|
| `C-a` | `Ctrl+A` |
| `M-f` | `Alt+F` (অথবা `Esc` ছেড়ে তারপর `F`) |
| `C-x C-e` | `Ctrl+X` চেপে ছাড়ো, তারপর `Ctrl+E` |
| `M-C-e` | `Ctrl+Alt+E` |

> ⚠️ **সতর্কতা:** কিছু terminal emulator-এ `Alt+F` চাপলে "File" মেনু খুলে যায়, শেলে পৌঁছায় না। সমাধান: (১) টার্মিনালের Preferences থেকে "Enable menu access keys" বন্ধ করো, অথবা (২) `Alt`-এর বদলে `Esc` চেপে ছেড়ে দিয়ে তারপর `F` চাপো — readline এটাকে `Alt+F` হিসেবেই ধরে। macOS Terminal-এ "Use Option as Meta key" চালু করতে হয়।

---

## ৬.৩ কার্সর চলাফেরা (Movement)

ধরো এই লাইন টাইপ করেছ, কার্সর একদম শেষে:

```bash
$ cp /var/log/nginx/access.log /home/rahim/backup/█
```

এখন শুরুতে `sudo` বসাতে চাও। ←-কী চেপে ৫০ বার যাবে? না! `Ctrl+A` চাপো, কার্সর একলাফে শুরুতে।

| শর্টকাট | Readline ফাংশন | কাজ |
|---|---|---|
| `Ctrl+A` | `beginning-of-line` | লাইনের **শুরুতে** যাও (A = alphabet-এর শুরু) |
| `Ctrl+E` | `end-of-line` | লাইনের **শেষে** যাও (E = End) |
| `Ctrl+B` | `backward-char` | এক অক্ষর পেছনে (← এর মতো) (B = Back) |
| `Ctrl+F` | `forward-char` | এক অক্ষর সামনে (→ এর মতো) (F = Forward) |
| `Alt+B` | `backward-word` | এক **শব্দ** পেছনে |
| `Alt+F` | `forward-word` | এক **শব্দ** সামনে |
| `Ctrl+←` / `Ctrl+→` | (বেশিরভাগ distro-তে `/etc/inputrc`-এ বাঁধা) | শব্দ ধরে পেছনে/সামনে |
| `Home` / `End` | | লাইনের শুরু/শেষ |
| `Ctrl+X Ctrl+X` | `exchange-point-and-mark` | কার্সর আর "mark"-এর মধ্যে লাফ দাও (দুই প্রান্তে আসা-যাওয়ার জন্য দারুণ) |
| `Ctrl+]` তারপর একটা অক্ষর | `character-search` | সামনে ওই অক্ষরের পরের উপস্থিতিতে লাফ দাও |
| `Ctrl+Alt+]` তারপর অক্ষর | `character-search-backward` | পেছনে খোঁজো |

উদাহরণ — `Ctrl+]` চেপে `/` চাপলে কার্সর পরের `/`-তে চলে যাবে।

> 📝 **মনে রাখো — "শব্দ" কী?** `Alt+B`/`Alt+F`-এর কাছে শব্দ মানে শুধু অক্ষর আর সংখ্যা (alphanumeric)। তাই `/var/log/nginx`-এ `var`, `log`, `nginx` আলাদা তিনটা শব্দ। কিন্তু `Ctrl+W`-এর কাছে শব্দ মানে স্পেস দিয়ে আলাদা করা অংশ — এই পার্থক্যটা পরের অংশে কাজে লাগবে।

---

## ৬.৪ মোছা, কাটা আর পেস্ট করা — Kill Ring

Readline-এ "মুছে ফেলা" দুই রকম:
1. **Delete** — অক্ষর একদম হারিয়ে যায় (`Backspace`, `Ctrl+D`, `Delete`)।
2. **Kill** — লেখাটা কাটা হয় এবং একটা বিশেষ ক্লিপবোর্ডে জমা থাকে, যার নাম **kill ring**। পরে `Ctrl+Y` (yank) দিয়ে ফেরত আনা যায়।

### Kill ring — উপমা

Kill ring হলো একটা গোল ঘূর্ণায়মান তাক। প্রতিবার কিছু kill করলে সেটা তাকের সামনে রাখা হয়। `Ctrl+Y` সামনের জিনিসটা বের করে। `Ctrl+Y`-এর ঠিক পরে `Alt+Y` চাপলে তাক ঘুরে আগেরটা আসে।

```
     Kill ring (ঘূর্ণায়মান)
   ┌─────────────────────────┐
   │ [৩] "nginx"     ← সর্বশেষ │  Ctrl+Y → "nginx"
   │ [২] "--force "          │  তারপর Alt+Y → "--force "
   │ [১] "sudo "             │  আবার Alt+Y → "sudo "
   └─────────────────────────┘
```

> 📝 **মনে রাখো:** এই kill ring শুধু ওই শেলের ভেতরে। এটা তোমার ডেস্কটপের ক্লিপবোর্ড (`Ctrl+Shift+C/V`) থেকে সম্পূর্ণ আলাদা!

### Kill ও Delete শর্টকাট

| শর্টকাট | Readline ফাংশন | কাজ |
|---|---|---|
| `Ctrl+U` | `unix-line-discard` | কার্সর থেকে **লাইনের শুরু পর্যন্ত** কেটে ফেলো |
| `Ctrl+K` | `kill-line` | কার্সর থেকে **লাইনের শেষ পর্যন্ত** কেটে ফেলো |
| `Ctrl+W` | `unix-word-rubout` | কার্সরের **আগের শব্দ** কাটো (স্পেস পর্যন্ত) |
| `Alt+Backspace` | `backward-kill-word` | আগের শব্দ কাটো (alphanumeric শব্দ — `/` এ থামে) |
| `Alt+D` | `kill-word` | কার্সর থেকে **সামনের শব্দের শেষ** পর্যন্ত কাটো |
| `Ctrl+Y` | `yank` | সর্বশেষ কাটা লেখা পেস্ট করো |
| `Alt+Y` | `yank-pop` | ঠিক আগে `Ctrl+Y` করে থাকলে — kill ring ঘুরিয়ে আগের কাটা লেখা আনো |
| `Ctrl+D` | `delete-char` | কার্সরের নিচের অক্ষর মোছো (লাইন খালি হলে = EOF, নিচে দেখো) |
| `Backspace` | `backward-delete-char` | কার্সরের আগের অক্ষর মোছো |
| `Alt+\` | `delete-horizontal-space` | কার্সরের চারপাশের সব স্পেস/ট্যাব মুছে ফেলো |
| `Ctrl+_` বা `Ctrl+X Ctrl+U` | `undo` | শেষ এডিট **undo** করো (বারবার চাপা যায়) |
| `Alt+R` | `revert-line` | লাইনে করা সব পরিবর্তন বাতিল করো |

### `Ctrl+W` বনাম `Alt+Backspace` — হাতে-কলমে

```bash
$ ls /var/log/nginx█
```

- `Ctrl+W` চাপলে: পুরো `/var/log/nginx` মুছে যাবে (কারণ স্পেস পর্যন্ত এক শব্দ)।
- `Alt+Backspace` চাপলে: শুধু `nginx` মুছবে, থাকবে `ls /var/log/`।

পাথের শেষ অংশ বদলাতে `Alt+Backspace` আদর্শ।

### বাস্তব উদাহরণ ১: ভুলে যাওয়া `sudo`

```bash
$ apt install htop
E: Could not open lock file /var/lib/dpkg/lock-frontend - open (13: Permission denied)
```

সবচেয়ে সহজ:
```bash
$ sudo !!
sudo apt install htop
```

`!!` সম্পর্কে ৬.৮ অংশে বিস্তারিত।

### বাস্তব উদাহরণ ২: অর্ধেক লেখা কমান্ড "পার্ক" করা

তুমি একটা লম্বা কমান্ড লিখছ, হঠাৎ মনে হলো আগে দেখা দরকার ফোল্ডারটা আছে কিনা।

```bash
$ tar -czvf backup-2026.tar.gz /home/rahim/projects --exclude=node_modules█
```

1. `Ctrl+E` (শেষে যাও, যদি না থাকো), তারপর `Ctrl+U` — পুরো লাইন kill ring-এ চলে গেল।
2. `ls /home/rahim/projects` লিখে Enter।
3. `Ctrl+Y` — আগের পুরো কমান্ড ফেরত!

> 💡 **টিপস:** এর চেয়েও সুন্দর একটা উপায় আছে: লাইনের শুরুতে `#` বসিয়ে (`Ctrl+A`, তারপর `#`) Enter চাপো, অথবা সরাসরি `Alt+#` চাপো (`insert-comment`)। কমান্ডটা কমেন্ট হিসেবে history-তে সেভ হবে, চলবে না। পরে `↑` চেপে এনে `#` মুছে চালাও।

---

## ৬.৫ বড়-ছোট হাতের অক্ষর ও অদলবদল

| শর্টকাট | Readline ফাংশন | কাজ |
|---|---|---|
| `Alt+U` | `upcase-word` | কার্সর থেকে শব্দের শেষ পর্যন্ত **UPPERCASE** করো |
| `Alt+L` | `downcase-word` | কার্সর থেকে শব্দের শেষ পর্যন্ত **lowercase** করো |
| `Alt+C` | `capitalize-word` | শব্দের প্রথম অক্ষর বড়, বাকি ছোট (**Capitalize**) |
| `Ctrl+T` | `transpose-chars` | কার্সরের আগের অক্ষর আর কার্সরের নিচের অক্ষর অদলবদল |
| `Alt+T` | `transpose-words` | আগের শব্দ আর বর্তমান শব্দ অদলবদল |

### উদাহরণ: টাইপো ঠিক করা

```bash
$ sl -la█        # ভুল: sl লিখতে চেয়েছিলে ls
```

`Ctrl+A` চেপে শুরুতে যাও, `Ctrl+F` একবার (কার্সর `l`-এর ওপর), তারপর `Ctrl+T`:

```bash
$ ls -la
```

> 💡 **টিপস:** লাইনের একদম শেষে `Ctrl+T` চাপলে শেষ দুটো অক্ষর অদলবদল হয়। যেমন `grpe` লিখে ফেলেছ (কার্সর শেষে) — সাথে সাথে `Ctrl+T` চাপো, হয়ে যাবে `grep`। মূল কথা: `Ctrl+T` কার্সরের **ঠিক আগের** অক্ষরকে এক ঘর সামনে টেনে আনে। টাইপ করার সাথে সাথে শেষ দুটো অক্ষর উল্টো হলে (যেমন `sl` → `ls`) এটা সবচেয়ে কাজের।

### উদাহরণ: environment variable বড় হাতে

```bash
$ echo $home█
```

`Alt+B` চেপে `home`-এর শুরুতে যাও, তারপর `Alt+U`:

```bash
$ echo $HOME
/home/rahim
```

---

## ৬.৬ কন্ট্রোল শর্টকাট: Ctrl+C, D, Z, L, S, Q

এগুলো লাইন এডিট করে না — প্রসেস বা টার্মিনালকে নিয়ন্ত্রণ করে। ভেতরে কে সামলায় সেটা খেয়াল করো।

| শর্টকাট | কে সামলায় | কাজ |
|---|---|---|
| `Ctrl+C` | tty → **SIGINT** | চলমান প্রোগ্রামকে থামতে বলো (interrupt)। প্রম্পটে চাপলে বর্তমান লাইন বাতিল হয়ে নতুন প্রম্পট আসে |
| `Ctrl+Z` | tty → **SIGTSTP** | চলমান প্রোগ্রাম **সাময়িক থামাও** (suspend), শেলে ফিরে আসো |
| `Ctrl+\` | tty → **SIGQUIT** | আরও কড়া থামানো; প্রোগ্রাম core dump তৈরি করতে পারে |
| `Ctrl+D` | tty/readline → **EOF** | খালি লাইনে: "ইনপুট শেষ" — শেল/প্রোগ্রাম বন্ধ হয়। লেখা থাকলে: কার্সরের নিচের অক্ষর মোছে |
| `Ctrl+L` | readline (`clear-screen`) | স্ক্রিন পরিষ্কার, বর্তমান লাইন অক্ষত থাকে |
| `Ctrl+Alt+L` | readline (`clear-display`, bash 5.1+) | স্ক্রিন **এবং** scrollback দুটোই পরিষ্কার (টার্মিনাল সাপোর্ট করলে) |
| `Ctrl+S` | tty (**XOFF**) | টার্মিনাল আউটপুট **জমিয়ে দেয়** (freeze) |
| `Ctrl+Q` | tty (**XON**) | জমানো আউটপুট আবার চালু করে |
| `Ctrl+V` তারপর কী | readline (`quoted-insert`) | পরের কী-টা আক্ষরিক অর্থে বসাও (যেমন আসল Tab অক্ষর) |
| `Ctrl+J` বা `Ctrl+M` | readline | Enter-এর সমান (newline / carriage return) |
| `Ctrl+O` | readline (`operate-and-get-next`) | বর্তমান লাইন চালাও, আর history-র পরের লাইনটা প্রস্তুত করে রাখো |

### `Ctrl+C` — কেন এটা "Copy" নয়?

উইন্ডোজে অভ্যস্তরা সবচেয়ে বেশি এখানে ধাক্কা খায়। টার্মিনালে `Ctrl+C` চাপলে কার্নেলের tty driver বাইট `0x03` দেখে চলমান **foreground process group**-কে `SIGINT` signal পাঠায়। বেশিরভাগ প্রোগ্রাম এতে বন্ধ হয়ে যায়।

```bash
$ ping google.com
PING google.com (142.250.193.14) 56(84) bytes of data.
64 bytes from ...: icmp_seq=1 ttl=117 time=24.1 ms
64 bytes from ...: icmp_seq=2 ttl=117 time=23.8 ms
^C
--- google.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
```

টার্মিনালে কপি করতে `Ctrl+Shift+C` লাগে (৬.১০ অংশ)। Signal সম্পর্কে বিস্তারিত পড়বে [অধ্যায় ২৪](24-process-management.md)-এ।

> 📝 **মনে রাখো:** `Ctrl+C` একটা **অনুরোধ**। প্রোগ্রাম চাইলে SIGINT উপেক্ষা করতে পারে। আটকে গেলে `Ctrl+\` চেষ্টা করো, তাতেও না হলে অন্য টার্মিনাল থেকে `kill -9 <PID>`।

### `Ctrl+Z` — থামাও, কিন্তু মেরো না

```bash
$ vim notes.txt
# vim-এ কাজ করছিলে, Ctrl+Z চাপলে:
[1]+  Stopped                 vim notes.txt
$ ls          # অন্য কাজ করো
$ fg          # vim ফিরে আসবে, যেখানে ছিলে ঠিক সেখানে
```

> ⚠️ **সতর্কতা:** অনেকে ভাবে `Ctrl+Z` প্রোগ্রাম বন্ধ করে। না! প্রোগ্রাম মেমোরিতে থেকে যায়। শেল থেকে `exit` করতে গেলে bash সাবধান করবে: `There are stopped jobs.` `jobs`, `fg`, `bg` নিয়ে বিস্তারিত [অধ্যায় ২৫](25-jobs-tmux.md)-এ।

### `Ctrl+D` — EOF, অর্থাৎ "ইনপুট শেষ"

EOF মানে End Of File। খালি লাইনে `Ctrl+D` চাপা মানে শেলকে বলা "আমার আর কিছু বলার নেই" — শেল বেরিয়ে যায় (টার্মিনাল ট্যাব বন্ধ হয়, বা SSH সেশন শেষ হয়)।

```bash
$ cat > list.txt
আলু
পেঁয়াজ
^D          ← খালি লাইনে Ctrl+D, cat থামল, ফাইল সেভ হলো
$ cat list.txt
আলু
পেঁয়াজ
```

> ⚠️ **সতর্কতা:** ভুল করে `Ctrl+D` চেপে গুরুত্বপূর্ণ root শেল বা SSH সেশন বন্ধ হয়ে যাওয়া খুব কমন। প্রতিরোধ:
> ```bash
> $ set -o ignoreeof        # এখন খালি লাইনে Ctrl+D-তে বেরোবে না
> $ ^D
> Use "exit" to leave the shell.
> ```
> অথবা `IGNOREEOF=3` সেট করলে পরপর ৩ বার `Ctrl+D` চাপলে তবেই বেরোবে।

### `Ctrl+L` বনাম `clear` বনাম `reset`

| উপায় | কী করে |
|---|---|
| `Ctrl+L` | স্ক্রিন পরিষ্কার, অর্ধেক লেখা কমান্ড হারায় না |
| `clear` | স্ক্রিন পরিষ্কার (সাধারণত scrollback-ও মুছে দেয়) |
| `reset` | টার্মিনাল পুরো রিসেট করে — যখন কোনো binary ফাইল `cat` করে টার্মিনাল আজব অক্ষরে ভরে যায় |

### `Ctrl+S` — "টার্মিনাল জমে গেছে!" রহস্য

নতুনরা প্রায়ই ভুলে `Ctrl+S` চাপে (অন্য অ্যাপে "Save" এর অভ্যাস থেকে) এবং মনে করে টার্মিনাল হ্যাং হয়ে গেছে — কিছু টাইপ করলে দেখা যায় না। আসলে এটা ১৯৭০-এর দশকের **XON/XOFF flow control** — ধীর প্রিন্টার-টার্মিনালকে "একটু থামো" বলার ব্যবস্থা।

**সমাধান: `Ctrl+Q` চাপো।** সবকিছু আবার চলবে (এর মধ্যে যা টাইপ করেছিলে তা-ও চলে আসবে — সাবধান!)

সমস্যা হলো, readline-এ `Ctrl+S` আসলে **forward history search**-এর জন্য বাঁধা, কিন্তু tty আগেই ধরে ফেলে বলে কখনো readline পর্যন্ত পৌঁছায় না। flow control বন্ধ করে দিলে `Ctrl+S` readline-এর কাজে লাগবে:

```bash
$ stty -ixon       # এই সেশনের জন্য; স্থায়ী করতে ~/.bashrc-এ লেখো
```

---

## ৬.৭ Tab Completion — অর্ধেক লেখো, বাকিটা শেল লিখবে

Tab completion হলো লিনাক্স টার্মিনালের সবচেয়ে বড় সময়-বাঁচানো ফিচার। কোনো কিছুর শুরুটা লিখে `Tab` চাপো:

- **একটাই মিল থাকলে** → শেল পুরোটা লিখে দেয়।
- **একাধিক মিল থাকলে** → যতটুকু সবার মধ্যে কমন, ততটুকু লিখে থামে, একটা বিপ (bell) দিতে পারে।
- **তখন আবার `Tab` চাপলে (double Tab)** → সব সম্ভাব্য অপশনের তালিকা দেখায়।

```bash
$ cd /usr/sh<Tab>
$ cd /usr/share/

$ sys<Tab><Tab>
sysctl                    systemd-analyze           systemd-run
systemctl                 systemd-ask-password      systemd-tmpfiles
systemd                   systemd-cat               ...
$ sys
```

অনেক বেশি মিল থাকলে (ডিফল্ট ১০০-র বেশি) জিজ্ঞেস করে:

```bash
$ <Tab><Tab>
Display all 2381 possibilities? (y or n)
```

`n` চাপো, তারপর আরও কয়েকটা অক্ষর লিখে আবার Tab।

### কী কী complete হয়?

| কী লিখছ | কী complete হয় |
|---|---|
| লাইনের প্রথম শব্দ | কমান্ড (PATH-এর প্রোগ্রাম, alias, function, builtin) |
| পরের শব্দ | ফাইল ও ডিরেক্টরির নাম |
| `$` দিয়ে শুরু | ভেরিয়েবলের নাম (`$HO<Tab>` → `$HOME`) |
| `~` দিয়ে শুরু | ইউজারনেম (`~ro<Tab>` → `~root/`) |
| `@` দিয়ে শুরু | হোস্টনেম (`/etc/hosts` থেকে) |

### Programmable completion ও `bash-completion` প্যাকেজ

Ubuntu/Debian/Fedora-তে সাধারণত `bash-completion` প্যাকেজ ইনস্টল থাকে। এটা থাকলে শেল কমান্ড বুঝে স্মার্ট completion দেয়:

```bash
$ git che<Tab>
$ git checkout
$ systemctl rest<Tab>
$ systemctl restart
$ sudo apt ins<Tab>
$ sudo apt install
$ ssh <Tab><Tab>          # ~/.ssh/config আর known_hosts থেকে সার্ভারের নাম
```

এটা কাজ করে `complete` নামে bash-এর একটা builtin দিয়ে, যেখানে প্রতিটা কমান্ডের জন্য "কী কী সাজেস্ট করতে হবে" তার নিয়ম বাঁধা থাকে।

```bash
$ complete -p git
complete -o bashdefault -o default -o nospace -F __git_wrap__git_main git
```

না থাকলে ইনস্টল করো (প্যাকেজ ইনস্টল নিয়ে বিস্তারিত [অধ্যায় ৩০](30-package-management.md)-এ):

```bash
$ sudo apt install bash-completion      # Ubuntu/Debian
$ sudo dnf install bash-completion      # Fedora/RHEL
```

নিজের কমান্ডের জন্য সহজ completion বানাতে পারো:

```bash
$ complete -W "start stop restart status" myservice
$ myservice st<Tab><Tab>
start   status  stop
```

### Completion সংক্রান্ত অন্যান্য শর্টকাট

| শর্টকাট | Readline ফাংশন | কাজ |
|---|---|---|
| `Tab` | `complete` | Complete করো |
| `Tab Tab` | — | সম্ভাব্য সব দেখাও |
| `Alt+?` | `possible-completions` | সম্ভাব্য সব দেখাও (Tab Tab-এর মতো) |
| `Alt+*` | `insert-completions` | সব মিল একসাথে লাইনে বসিয়ে দাও |
| `Alt+/` | `complete-filename` | জোর করে ফাইলনেম complete করো |
| `Alt+$` | `complete-variable` | ভেরিয়েবল complete করো |
| `Alt+~` | `complete-username` | ইউজারনেম complete করো |
| `Alt+@` | `complete-hostname` | হোস্টনেম complete করো |
| `Alt+!` | `complete-command` | কমান্ডের নাম complete করো |

উদাহরণ — `Alt+*`:

```bash
$ rm *.log<Alt+*>
$ rm access.log error.log debug.log
```

মোছার আগে ঠিক কোন ফাইলগুলো যাবে, তা দেখে নেওয়ার চমৎকার উপায়!

> 💡 **টিপস:** Tab-কে "বানান যাচাইকারী" হিসেবেও ব্যবহার করো। ফাইলের নাম লিখে Tab চাপলে যদি complete না হয়, তার মানে নাম ভুল বা ফাইল নেই — Enter চাপার আগেই বুঝে যাবে।

---

## ৬.৮ কমান্ড হিস্ট্রি — শেলের স্মৃতি

তুমি যত কমান্ড চালাও, bash সব মনে রাখে। প্রথমে **মেমোরিতে** (একটা তালিকায়), আর শেল বন্ধ হওয়ার সময় সেগুলো লেখে **`~/.bash_history`** ফাইলে।

```
  শেল চালু হলো
       │
       ▼
 ┌──────────────────┐   পড়ে নেয় (HISTFILESIZE পর্যন্ত লাইন রাখে)
 │ ~/.bash_history  │ ─────────────────────────┐
 └──────────────────┘                          ▼
        ▲                         ┌──────────────────────────┐
        │                         │ মেমোরির history তালিকা      │
        │  শেল বন্ধের সময় লেখে       │ (সর্বোচ্চ HISTSIZE টি)       │
        │  (histappend থাকলে       │  ↑/↓, Ctrl+R, !! সব       │
        │   append, নাহলে overwrite)│  এখান থেকেই কাজ করে         │
        └─────────────────────────└──────────────────────────┘
                                         ▲
                              প্রতিটা নতুন কমান্ড যোগ হয়
                              (HISTCONTROL/HISTIGNORE দিয়ে ফিল্টার হয়ে)
```

> 📝 **মনে রাখো:** একটা টার্মিনালে চালানো কমান্ড অন্য খোলা টার্মিনালে সাথে সাথে `↑` চেপে পাবে না — কারণ প্রতিটা শেলের নিজস্ব মেমোরি-তালিকা আছে, আর ফাইলে লেখা হয় শেল বন্ধের সময়। এর সমাধান ৬.৯ অংশে।

### History-তে ঘোরাফেরা

| শর্টকাট | Readline ফাংশন | কাজ |
|---|---|---|
| `↑` বা `Ctrl+P` | `previous-history` | আগের কমান্ড (P = Previous) |
| `↓` বা `Ctrl+N` | `next-history` | পরের কমান্ড (N = Next) |
| `Alt+<` | `beginning-of-history` | history-র একদম প্রথম কমান্ড |
| `Alt+>` | `end-of-history` | বর্তমান (খালি) লাইনে ফেরো |
| `Ctrl+R` | `reverse-search-history` | পেছনের দিকে খোঁজো (incremental) |
| `Ctrl+S` | `forward-search-history` | সামনের দিকে খোঁজো (`stty -ixon` দরকার) |
| `Alt+P` | `non-incremental-reverse-search-history` | পুরো শব্দ লিখে Enter, তারপর খোঁজে |
| `Alt+N` | `non-incremental-forward-search-history` | ওপরেরটার সামনের দিকের সংস্করণ |
| `Alt+.` বা `Alt+_` | `yank-last-arg` | আগের কমান্ডের **শেষ argument** বসাও; বারবার চাপলে আরও আগের কমান্ডে যায় |
| `Ctrl+Alt+Y` | `yank-nth-arg` | আগের কমান্ডের **প্রথম argument** বসাও |
| `Alt+^` | `history-expand-line` | লাইনের `!!`, `!$` ইত্যাদি এখনই খুলে দেখাও (না চালিয়ে) |
| `Ctrl+Alt+E` | `shell-expand-line` | alias, ভেরিয়েবল, history সব expand করে দেখাও |
| `Ctrl+X Ctrl+E` | `edit-and-execute-command` | বর্তমান লাইন `$EDITOR`-এ খোলো; সেভ করে বেরোলে চলবে |

### `Alt+.` — সবচেয়ে ভালোবাসার শর্টকাট

```bash
$ mkdir -p /home/rahim/projects/website/assets/images
$ cd <Alt+.>
$ cd /home/rahim/projects/website/assets/images
```

বারবার `Alt+.` চাপলে আরও পেছনের কমান্ডগুলোর শেষ argument ঘুরে ঘুরে আসে।

**সংখ্যা সহ (numeric argument):** `Alt` চেপে ধরে একটা সংখ্যা চাপো, তারপর `Alt+.` — তখন আগের কমান্ডের **ওই নম্বরের** শব্দ আসে (০ = কমান্ডের নাম)।

```bash
$ cp report.pdf /mnt/usb/
$ ls -l <Alt+1><Alt+.>
$ ls -l report.pdf
```

> 💡 **টিপস:** Readline-এর প্রায় সব ফাংশনের আগে `Alt+সংখ্যা` দিয়ে "কতবার" বলে দেওয়া যায় (`digit-argument`)। যেমন `Alt+3 Ctrl+W` = আগের ৩টা শব্দ মুছে ফেলো। `Alt+-` দিয়ে উল্টো দিক।

### `Ctrl+X Ctrl+E` — লম্বা কমান্ডের জন্য এডিটর

একটা জটিল one-liner লিখছ, পাঁচটা পাইপ দিয়ে — টার্মিনালের এক লাইনে এডিট করা কষ্টের। `Ctrl+X Ctrl+E` চাপলে লাইনটা তোমার এডিটরে (`$VISUAL`, না থাকলে `$EDITOR`, না থাকলে ডিফল্ট — সাধারণত Debian/Ubuntu-তে `editor` → nano) খুলবে। সেভ করে বেরোলে কমান্ডটা চলবে।

```bash
$ export EDITOR=nano     # বা vim — nano শিখবে অধ্যায় ১৩-এ, vim অধ্যায় ১৪-এ
```

> ⚠️ **সতর্কতা:** এডিটর থেকে সেভ করে বেরোলে কমান্ড **সাথে সাথে চলে**। চালাতে না চাইলে এডিটরে সব লেখা মুছে সেভ করো।

---

## ৬.৯ `history` কমান্ড

### Syntax

```bash
history [n]
history -c
history -d offset
history -d start-end
history -a | -n | -r | -w [filename]
history -s arg [arg...]
```

### গুরুত্বপূর্ণ অপশন

| অপশন | কাজ |
|---|---|
| (কিছু না) | পুরো history তালিকা, নম্বর সহ |
| `n` (একটা সংখ্যা) | শেষ `n`টা কমান্ড দেখাও |
| `-c` | মেমোরির পুরো history মুছে দাও (clear) |
| `-d offset` | নির্দিষ্ট নম্বরের এন্ট্রি মোছো |
| `-d start-end` | একটা রেঞ্জ মোছো (bash 5.0+) |
| `-a` | এই সেশনের নতুন লাইনগুলো ফাইলে **append** করো |
| `-n` | ফাইলে থাকা কিন্তু এখনো না-পড়া লাইনগুলো মেমোরিতে আনো |
| `-r` | পুরো ফাইল পড়ে মেমোরির তালিকায় যোগ করো |
| `-w` | মেমোরির পুরো তালিকা ফাইলে লেখো (overwrite) |
| `-s arg` | কমান্ড না চালিয়েই history-তে একটা এন্ট্রি যোগ করো |
| `-p arg` | history expansion করে ফল দেখাও, চালায় না এবং সেভ করে না |

### উদাহরণ

```bash
$ history 5
  496  cd /etc/nginx
  497  sudo nano nginx.conf
  498  sudo nginx -t
  499  sudo systemctl reload nginx
  500  history 5
```

```bash
$ history | grep ssh          # পাইপ আর grep শিখবে অধ্যায় ১৯ ও ২০-এ
  212  ssh rahim@192.168.1.50
  340  ssh -p 2222 deploy@example.com
```

ভুল করে পাসওয়ার্ড কমান্ডে লিখে ফেলেছ? মুছে দাও:

```bash
$ mysql -u root -pMySecret123        # ওহ না!
$ history 2
  612  mysql -u root -pMySecret123
  613  history 2
$ history -d 612
$ history -w                         # ফাইলেও হালনাগাদ করো
```

> ⚠️ **সতর্কতা:** `history -c` শুধু মেমোরি মোছে। ফাইল `~/.bash_history` থেকে মুছতে এরপর `history -w` চালাও। আর মনে রেখো, ব্যাকআপ বা অন্য খোলা শেলে কপি থেকে যেতে পারে। পাসওয়ার্ড কখনো কমান্ড লাইনে লিখবে না — `ps` দিয়েও অন্য ইউজার সেটা দেখতে পারে।

### `fc` — Fix Command

`fc` একটা পুরনো (POSIX) কিন্তু দারুণ builtin।

| কমান্ড | কাজ |
|---|---|
| `fc -l` | শেষ ১৬টা কমান্ডের তালিকা |
| `fc -l -5` | শেষ ৫টা |
| `fc -l 100 110` | ১০০ থেকে ১১০ নম্বর |
| `fc` | শেষ কমান্ড এডিটরে খোলো, সেভ করলে চলবে |
| `fc 498` | ৪৯৮ নম্বর কমান্ড এডিটরে খোলো |
| `fc 495 499` | ৪৯৫–৪৯৯ সবগুলো এডিটরে, সেভ করলে পরপর চলবে |
| `fc -s` | শেষ কমান্ড আবার চালাও (`!!`-এর মতো) |
| `fc -s old=new` | শেষ কমান্ডে old কে new বানিয়ে চালাও (সব জায়গায়) |
| `fc -s ssh` | `ssh` দিয়ে শুরু হওয়া শেষ কমান্ড চালাও |
| `fc -e nano` | নির্দিষ্ট এডিটর দিয়ে খোলো |

```bash
$ echo dev dev dev
dev dev dev
$ fc -s dev=prod
echo prod prod prod
prod prod prod
```

---

## ৬.১০ History Expansion — `!` এর জাদু

History expansion হলো একটা "টেক্সট-রিপ্লেস" সুবিধা: Enter চাপার পর, কমান্ড চালানোর **আগে**, bash `!` দিয়ে শুরু হওয়া চিহ্নগুলো history থেকে আসল লেখা দিয়ে বদলে দেয় — আর বদলানো কমান্ডটা স্ক্রিনে প্রিন্ট করে দেখায়।

এর তিনটা অংশ: **event designator** (কোন কমান্ড), **word designator** (সেই কমান্ডের কোন শব্দ), **modifier** (শব্দটাকে কীভাবে বদলাবে)। ফরম্যাট: `!event:word:modifier`

### ধাপ ১: Event designator — কোন কমান্ড?

| চিহ্ন | মানে | উদাহরণ |
|---|---|---|
| `!!` | ঠিক আগের কমান্ড | `sudo !!` |
| `!n` | history-র `n` নম্বর কমান্ড | `!498` |
| `!-n` | `n` ধাপ আগের কমান্ড | `!-2` = দুই কমান্ড আগে |
| `!string` | `string` দিয়ে **শুরু** হওয়া সর্বশেষ কমান্ড | `!ssh` |
| `!?string?` | `string` **আছে** এমন সর্বশেষ কমান্ড | `!?nginx?` |
| `^old^new` | আগের কমান্ডে `old`-এর **প্রথম** উপস্থিতি `new` দিয়ে বদলে চালাও | `^8080^9090` |
| `!#` | এখন পর্যন্ত লেখা বর্তমান লাইন | `cp file.txt !#:1.bak` |

```bash
$ cat /etc/ssh/sshd_config
cat: /etc/ssh/sshd_config: Permission denied
$ sudo !!
sudo cat /etc/ssh/sshd_config
# ... ফাইলের লেখা ...
```

```bash
$ curl http://localhost:8080/health
curl: (7) Failed to connect to localhost port 8080
$ ^8080^9090
curl http://localhost:9090/health
OK
```

### ধাপ ২: Word designator — কোন শব্দ?

একটা কমান্ডের শব্দগুলো ০ থেকে গোনা হয়:

```
  cp   -r   photos   /mnt/backup
  ↑    ↑    ↑        ↑
  0    1    2        3 ($)
       ^ (প্রথম argument)
```

| চিহ্ন | মানে |
|---|---|
| `!$` | আগের কমান্ডের **শেষ** শব্দ (`!!:$`-এর সংক্ষেপ) |
| `!^` | আগের কমান্ডের **প্রথম argument** (শব্দ ১) |
| `!*` | আগের কমান্ডের **সব argument** (১ থেকে শেষ, কমান্ডের নাম বাদে) |
| `!:0` | আগের কমান্ডের নাম |
| `!:2` | আগের কমান্ডের ২ নম্বর শব্দ |
| `!:1-3` | ১ থেকে ৩ নম্বর শব্দ |
| `!:2*` | ২ থেকে শেষ পর্যন্ত (`!:2-$`) |
| `!:2-` | ২ থেকে শেষের আগের শব্দ পর্যন্ত |
| `!ssh:$` | `ssh` দিয়ে শুরু হওয়া শেষ কমান্ডের শেষ শব্দ |

```bash
$ mkdir /var/www/mysite
$ cd !$
cd /var/www/mysite

$ touch a.txt b.txt c.txt
$ ls -l !*
ls -l a.txt b.txt c.txt

$ cp -r photos /mnt/backup
$ ls !:3
ls /mnt/backup
```

> 💡 **টিপস:** `!$` আর `Alt+.` প্রায় একই কাজ করে। পার্থক্য: `Alt+.` লেখাটা **সাথে সাথে** লাইনে বসায়, তুমি দেখে-শুনে Enter চাপো। `!$` দেখা যায় Enter চাপার পর। নিরাপত্তার জন্য `Alt+.` ভালো।

### ধাপ ৩: Modifier — শব্দটা বদলাও

| Modifier | কাজ | উদাহরণ (শব্দ = `/home/rahim/song.mp3`) |
|---|---|---|
| `:h` | head — শেষ পাথ-অংশ বাদ (dirname) | `/home/rahim` |
| `:t` | tail — শুধু শেষ অংশ (basename) | `song.mp3` |
| `:r` | remove extension | `/home/rahim/song` |
| `:e` | শুধু extension | `.mp3` |
| `:p` | **শুধু প্রিন্ট করো, চালিও না** | — |
| `:s/old/new/` | প্রথম `old` কে `new` বানাও | — |
| `:gs/old/new/` | সব `old` কে `new` বানাও | — |
| `:&` | আগের substitution আবার করো | — |
| `:q` | quote করো | `'/home/rahim/song.mp3'` |

```bash
$ ls /home/rahim/song.mp3
/home/rahim/song.mp3
$ cd !$:h
cd /home/rahim
$ ffmpeg -i !-2:$ !-2:$:r.ogg
ffmpeg -i /home/rahim/song.mp3 /home/rahim/song.ogg
```

`^old^new` শুধু **প্রথম** উপস্থিতি বদলায়। সব বদলাতে:

```bash
$ echo cat cat cat
cat cat cat
$ !!:gs/cat/dog/
echo dog dog dog
dog dog dog
```

### নিরাপদে history expansion: `:p` আর `histverify`

`rm` জাতীয় বিপজ্জনক কমান্ডে `!` ব্যবহার করার আগে দেখে নাও:

```bash
$ !rm:p
rm -rf ./build/*
```

এটা শুধু প্রিন্ট করে (আর history-তে যোগ করে, যাতে `↑` চেপে চালাতে পারো)। আরও ভালো উপায় — `histverify` চালু করো:

```bash
$ shopt -s histverify
$ sudo !!
$ sudo apt update█        ← চলল না! expand হয়ে লাইনে বসল, দেখে Enter চাপো
```

> ⚠️ **সতর্কতা:** `!` বিশেষ চিহ্ন বলে ডাবল কোটের ভেতরে ঝামেলা করে:
> ```bash
> $ echo "Hello World!"            # bash 4.3+ এ ঠিক আছে, কারণ ! এর পরে " আছে
> $ echo "Wow!great"
> bash: !great: event not found
> ```
> সমাধান: সিঙ্গেল কোট ব্যবহার করো (`'Wow!great'`) অথবা `\!` লেখো। স্ক্রিপ্টের ভেতরে (non-interactive শেলে) history expansion ডিফল্টভাবে বন্ধ থাকে, তাই সেখানে এই সমস্যা নেই। চাইলে interactive শেলেও বন্ধ করতে পারো: `set +H`।

---

## ৬.১১ History কনফিগার করা: HIST* ভেরিয়েবল

এই ভেরিয়েবলগুলো সাধারণত `~/.bashrc`-এ সেট করা হয় (`.bashrc` নিয়ে বিস্তারিত [অধ্যায় ৪৪](44-environment-bashrc.md)-এ)। Ubuntu-র ডিফল্ট `~/.bashrc`-এ আগে থেকেই কয়েকটা আছে:

```bash
$ grep -i hist ~/.bashrc
# don't put duplicate lines or lines starting with space in the history.
HISTCONTROL=ignoreboth
# append to the history file, don't overwrite it
shopt -s histappend
# for setting history length see HISTSIZE and HISTFILESIZE in bash(1)
HISTSIZE=1000
HISTFILESIZE=2000
```

### ভেরিয়েবলের তালিকা

| ভেরিয়েবল | কাজ | ডিফল্ট |
|---|---|---|
| `HISTFILE` | history ফাইলের পাথ | `~/.bash_history` |
| `HISTSIZE` | **মেমোরিতে** কতগুলো কমান্ড থাকবে | ৫০০ (Ubuntu-র `.bashrc`-এ ১০০০) |
| `HISTFILESIZE` | **ফাইলে** সর্বোচ্চ কত লাইন থাকবে | `HISTSIZE`-এর মান (Ubuntu-তে ২০০০) |
| `HISTCONTROL` | কোন কমান্ড সেভ হবে না | সেট নেই (Ubuntu-তে `ignoreboth`) |
| `HISTIGNORE` | কোন প্যাটার্নের কমান্ড বাদ যাবে | সেট নেই |
| `HISTTIMEFORMAT` | সময় দেখানোর ফরম্যাট (সেট থাকলে timestamp সেভ হয়) | সেট নেই |

> 📝 **মনে রাখো:** `HISTSIZE` বা `HISTFILESIZE`-এ ঋণাত্মক মান (যেমন `-1`) দিলে **সীমাহীন** (bash 4.3+)। আর `HISTSIZE=0` দিলে history পুরোপুরি বন্ধ।

### `HISTCONTROL`-এর মানগুলো

| মান | কাজ |
|---|---|
| `ignorespace` | **স্পেস দিয়ে শুরু** হওয়া কমান্ড সেভ হবে না |
| `ignoredups` | ঠিক আগের কমান্ডের সাথে হুবহু এক হলে সেভ হবে না |
| `ignoreboth` | ওপরের দুটোই |
| `erasedups` | নতুন কমান্ড সেভের আগে history থেকে তার সব পুরনো কপি মুছে ফেলো |

একাধিক মান কোলন দিয়ে: `HISTCONTROL=ignoreboth:erasedups`

`ignorespace` চালু থাকলে গোপন কমান্ড এভাবে চালাও (শুরুতে একটা স্পেস):

```bash
$  export API_TOKEN=abc123secret
$ history 2
  701  history 2          ← export কমান্ডটা নেই!
```

### `HISTIGNORE` — প্যাটার্ন দিয়ে বাদ দাও

কোলন দিয়ে আলাদা করা glob প্যাটার্ন। পুরো কমান্ড লাইনটা প্যাটার্নের সাথে মিলতে হবে।

```bash
HISTIGNORE="ls:ll:cd:pwd:exit:clear:history*"
```

এখন শুধু `ls` বা `cd` সেভ হবে না, কিন্তু `ls -la /etc` সেভ হবে (কারণ পুরো লাইন `ls`-এর সাথে মেলে না)। `ls *` লিখলে `ls` দিয়ে শুরু হওয়া সব বাদ।

### `HISTTIMEFORMAT` — কখন কোন কমান্ড চালিয়েছিলাম?

সার্ভারে সমস্যা হলে প্রায়ই প্রশ্ন ওঠে "কে, কখন এটা চালিয়েছিল?" `HISTTIMEFORMAT` সেট করলে bash প্রতিটা কমান্ডের সাথে সময় (Unix timestamp, `#` দিয়ে শুরু লাইনে) ফাইলে সেভ করে, আর `history` সেটা দেখায়। ফরম্যাট `strftime`-এর মতো (`date` কমান্ডের `%` কোডের মতো)।

```bash
$ export HISTTIMEFORMAT="%F %T  "
$ history 3
  702  2026-11-12 10:15:32  sudo systemctl restart nginx
  703  2026-11-12 10:16:01  tail -f /var/log/nginx/error.log
  704  2026-11-12 10:17:45  history 3
```

| কোড | মানে | উদাহরণ |
|---|---|---|
| `%F` | তারিখ (`%Y-%m-%d`) | `2026-11-12` |
| `%T` | সময় (`%H:%M:%S`) | `10:15:32` |
| `%d/%m/%y` | দিন/মাস/বছর | `12/11/26` |

> 📝 **মনে রাখো:** `HISTTIMEFORMAT` সেট করার **আগে** চালানো কমান্ডের timestamp ছিল না, তাই সেগুলোতে ফাইলটা প্রথম পড়ার সময়টাই দেখাবে — ভুল তথ্য মনে কোরো না।

### একাধিক টার্মিনালে history শেয়ার করা

ডিফল্টভাবে শেল বন্ধ হলে history ফাইলে লেখা হয়, আর `histappend` না থাকলে শেষ যে শেল বন্ধ হয় সে ফাইলটা **overwrite** করে — অন্য টার্মিনালের কমান্ড হারিয়ে যায়। সমাধান:

```bash
# ~/.bashrc
shopt -s histappend                 # overwrite না করে append করো
PROMPT_COMMAND="history -a${PROMPT_COMMAND:+; $PROMPT_COMMAND}"
```

`PROMPT_COMMAND` প্রতিবার প্রম্পট দেখানোর আগে চলে। `history -a` প্রতিটা কমান্ডের পরপরই ফাইলে লিখে দেয় — ক্র্যাশ হলেও হারায় না। অন্য টার্মিনালের কমান্ড এই টার্মিনালে আনতে চাইলে `history -n` চালাও (অথবা `PROMPT_COMMAND`-এ `history -a; history -n` দাও, তবে তাতে ↑ চাপলে অন্য টার্মিনালের কমান্ড মিশে যায় — কারো ভালো লাগে, কারো না)।

### একটা ভালো সেটআপ (~/.bashrc)

```bash
# ===== History সেটিং =====
HISTSIZE=50000
HISTFILESIZE=100000
HISTCONTROL=ignoreboth:erasedups
HISTIGNORE="ls:ll:pwd:exit:clear:history"
HISTTIMEFORMAT="%F %T  "
shopt -s histappend      # append করো
shopt -s cmdhist         # মাল্টি-লাইন কমান্ড এক এন্ট্রিতে সেভ করো (ডিফল্ট চালু)
shopt -s histverify      # !! ইত্যাদি expand করে দেখাও, সাথে সাথে চালিও না
PROMPT_COMMAND="history -a${PROMPT_COMMAND:+; $PROMPT_COMMAND}"
stty -ixon 2>/dev/null   # Ctrl+S দিয়ে forward search চালু
```

পরিবর্তন সক্রিয় করতে: `source ~/.bashrc`

> ⚠️ **সতর্কতা:** root ইউজারের আলাদা history থাকে (`/root/.bash_history`)। `sudo -i` বা `sudo su` দিয়ে root হলে যা চালাবে, তা তোমার নিজের history-তে যাবে না।

---

## ৬.১২ `Ctrl+R` — Reverse Search বিস্তারিত

এটা টার্মিনালের সবচেয়ে শক্তিশালী শর্টকাট। "আমার মনে আছে কমান্ডে `docker` আর `postgres` ছিল, কিন্তু পুরোটা মনে নেই" — `Ctrl+R` এর জন্যই।

### কীভাবে কাজ করে

1. `Ctrl+R` চাপো। প্রম্পট বদলে যাবে:
   ```
   (reverse-i-search)`':
   ```
2. টাইপ করা শুরু করো। প্রতিটা অক্ষরের সাথে সাথে (incremental — তাই নামে "i") সর্বশেষ মিলে যাওয়া কমান্ড দেখাবে:
   ```
   (reverse-i-search)`post': docker run -d --name db -e POSTGRES_PASSWORD=x postgres:16
   ```
3. মিলে যাওয়া অংশ শব্দের **যেকোনো জায়গায়** থাকতে পারে, শুরুতে হওয়া জরুরি না।

### Search চলাকালীন কী কী চাপা যায়

| কী | কাজ |
|---|---|
| আরও অক্ষর টাইপ | খোঁজা আরও নির্দিষ্ট করো |
| `Ctrl+R` আবার | একই শব্দের **আরও পুরনো** মিল দেখাও |
| `Ctrl+S` | উল্টো দিকে — **নতুনতর** মিল (`stty -ixon` লাগবে) |
| `Backspace` | খোঁজার শব্দ থেকে শেষ অক্ষর মোছো |
| `Enter` | পাওয়া কমান্ড **সাথে সাথে চালাও** |
| `Esc`, `←`, `→`, `Ctrl+A`, `Ctrl+E` ইত্যাদি | খোঁজা থামাও, কমান্ডটা লাইনে রেখে **এডিট করার সুযোগ দাও** |
| `Ctrl+G` | খোঁজা বাতিল, আগের (খালি) লাইনে ফেরো |
| `Ctrl+C` | খোঁজা ও লাইন দুটোই বাতিল |

```
 তুমি টাইপ করছ:   Ctrl+R  →  "ssh"
                       │
     history (নতুন থেকে পুরনো দিকে খোঁজে)
     ┌────────────────────────────────────┐
     │ 905  ls                            │
     │ 904  ssh deploy@10.0.0.7   ◄── ১ম মিল │
     │ 903  vim app.py                    │
     │ 870  ssh -p 2222 rahim@vps ◄── Ctrl+R আবার: ২য় মিল │
     │ ...                                │
     └────────────────────────────────────┘
```

### খুব কাজের কৌশল

1. **কমান্ডে "ট্যাগ" দাও:** লম্বা কমান্ডের শেষে কমেন্ট লিখে রাখো:
   ```bash
   $ rsync -avz --delete ~/site/ deploy@vps:/var/www/site/  # deploy-site
   ```
   পরে `Ctrl+R` চেপে `deploy-site` লিখলেই পেয়ে যাবে।

2. **পেয়ে এডিট করো, তারপর চালাও:** `Enter` না চেপে `→` বা `Ctrl+E` চাপো, বদলাও, তারপর Enter।

3. **`Ctrl+O` দিয়ে সিরিজ চালাও:** `Ctrl+R` দিয়ে কোনো পুরনো কমান্ড খুঁজে `Ctrl+O` চাপলে সেটা চলবে **এবং** history-তে তার পরের কমান্ডটা লাইনে চলে আসবে। আবার `Ctrl+O` — তার পরেরটা। একটা পুরনো কাজের ধারাবাহিকতা হুবহু আবার চালাতে চমৎকার।

4. **ইতিহাসে prefix-ভিত্তিক খোঁজা Up-arrow দিয়ে** (অনেকের প্রিয়) — `~/.inputrc`-এ:
   ```
   "\e[A": history-search-backward
   "\e[B": history-search-forward
   ```
   এখন `git` লিখে `↑` চাপলে শুধু `git` দিয়ে **শুরু** হওয়া কমান্ডগুলোই ঘুরবে।

> 💡 **টিপস:** আরও আধুনিক অভিজ্ঞতা চাইলে `fzf` টুল (Ubuntu/Debian/Fedora-র রিপোজিটরিতে আছে) ইনস্টল করে তার bash ইন্টিগ্রেশন চালু করতে পারো — তখন `Ctrl+R` fuzzy খোঁজার একটা তালিকা দেখায়। তবে আগে বিল্ট-ইন `Ctrl+R` ভালোভাবে শেখো, কারণ যেকোনো সার্ভারে এটাই পাবে।

---

## ৬.১৩ Terminal Emulator-এর শর্টকাট

এগুলো শেলের নয়, GUI টার্মিনাল অ্যাপের নিজস্ব। ডেস্কটপ পরিবেশ নিয়ে বিস্তারিত [অধ্যায় ৪](04-desktop-environment.md)-এ।

### কেন `Ctrl+Shift`?

কারণ সাধারণ `Ctrl+অক্ষর` কম্বিনেশনগুলো (`Ctrl+C`, `Ctrl+V`, `Ctrl+W`...) শেল ও প্রোগ্রামের জন্য সংরক্ষিত — ৬.২ অংশের ডায়াগ্রাম মনে করো। টার্মিনাল যদি `Ctrl+C`-কে "কপি" বানাত, তাহলে কোনো প্রোগ্রাম থামানো যেত না। তাই টার্মিনাল অ্যাপগুলো `Shift` যোগ করে নিজের কাজ করে।

### GNOME Terminal / Ptyxis (Ubuntu, Fedora Workstation) — সাধারণ ডিফল্ট

| শর্টকাট | কাজ |
|---|---|
| `Ctrl+Alt+T` | নতুন টার্মিনাল খোলো (Ubuntu ডেস্কটপের সিস্টেম শর্টকাট) |
| `Ctrl+Shift+C` | কপি |
| `Ctrl+Shift+V` | পেস্ট |
| `Shift+Insert` | পেস্ট (প্রায় সব টার্মিনালে) |
| `Ctrl+Shift+T` | নতুন ট্যাব |
| `Ctrl+Shift+N` | নতুন উইন্ডো |
| `Ctrl+Shift+W` | বর্তমান ট্যাব বন্ধ |
| `Ctrl+Shift+Q` | পুরো উইন্ডো বন্ধ |
| `Ctrl+Page Up` / `Ctrl+Page Down` | আগের/পরের ট্যাবে যাও |
| `Ctrl+Shift+Page Up` / `Page Down` | ট্যাবকে বামে/ডানে সরাও |
| `Alt+1` … `Alt+9` | নির্দিষ্ট নম্বরের ট্যাবে যাও (GNOME Terminal) |
| `Ctrl+Shift+F` | টার্মিনালের লেখার মধ্যে খোঁজো (Find) |
| `Ctrl++` (`Ctrl+Shift+=`) | লেখা বড় করো (zoom in) |
| `Ctrl+-` | লেখা ছোট করো |
| `Ctrl+0` | স্বাভাবিক সাইজ |
| `F11` | ফুলস্ক্রিন |
| `Shift+Page Up` / `Shift+Page Down` | scrollback-এ ওপরে/নিচে স্ক্রল |

### Konsole (KDE) — কিছু পার্থক্য

| শর্টকাট | কাজ |
|---|---|
| `Ctrl+Shift+T` | নতুন ট্যাব |
| `Ctrl+Shift+N` | নতুন উইন্ডো |
| `Shift+←` / `Shift+→` | আগের/পরের ট্যাব |
| `Ctrl+(` / `Ctrl+)` | উইন্ডো উপরে-নিচে/পাশাপাশি ভাগ (split view) |
| `Ctrl+Shift+F` | খোঁজো |

> 📝 **মনে রাখো:** শর্টকাটগুলো অ্যাপ ও ভার্সনভেদে একটু আলাদা হতে পারে। সঠিক তালিকা সবসময় টার্মিনালের **Preferences → Shortcuts** (বা Keyboard Shortcuts) মেনুতে দেখা যায়, আর সেখান থেকে বদলানোও যায়। Ubuntu 25.10 ও সাম্প্রতিক Fedora-তে ডিফল্ট টার্মিনাল **Ptyxis**; Ubuntu 24.04-এ এখনো GNOME Terminal।

### লিনাক্সের "দুটো ক্লিপবোর্ড"

লিনাক্সে (X11 ও বেশিরভাগ Wayland ডেস্কটপে) কপি-পেস্টের দুটো আলাদা ব্যবস্থা আছে:

| ব্যবস্থা | কপি | পেস্ট |
|---|---|---|
| **CLIPBOARD** (সাধারণ) | `Ctrl+Shift+C` (টার্মিনালে) / `Ctrl+C` (অন্য অ্যাপে) | `Ctrl+Shift+V` / `Ctrl+V` |
| **PRIMARY selection** | শুধু মাউস দিয়ে **সিলেক্ট** করলেই কপি | মাউসের **মাঝের বাটন** (চাকা) ক্লিক |

> 💡 **টিপস:** টার্মিনালে কোনো শব্দে **ডাবল-ক্লিক** করলে শব্দটা সিলেক্ট হয়, **ট্রিপল-ক্লিক** করলে পুরো লাইন। তারপর মাঝের বাটনে ক্লিক — পেস্ট! কোনো কী চাপতেই হলো না।

> ⚠️ **সতর্কতা:** ওয়েবসাইট থেকে কমান্ড কপি করে সরাসরি টার্মিনালে পেস্ট করা বিপজ্জনক — পেজে লুকানো লেখা (যেমন একটা newline সহ খারাপ কমান্ড) থাকতে পারে যা পেস্ট হওয়া মাত্র চলে যায়। আধুনিক bash (5.1+) ও readline ডিফল্টভাবে **bracketed paste** চালু রাখে, যাতে পেস্ট করা লেখা Enter না চাপা পর্যন্ত চলে না। তবুও অপরিচিত উৎসের কমান্ড আগে কোনো টেক্সট এডিটরে পেস্ট করে পড়ে নাও।

---

## ৬.১৪ Emacs Mode বনাম Vi Mode

এতক্ষণ যা শিখলে সব **emacs mode**-এর শর্টকাট — bash-এর ডিফল্ট। যারা Vim এডিটরে ([অধ্যায় ১৪](14-vim.md)) অভ্যস্ত, তারা কমান্ড লাইনেও Vim-এর মতো এডিট করতে পারে।

```bash
$ set -o vi        # vi mode চালু
$ set -o emacs     # আবার emacs mode-এ ফেরো
$ set -o | grep -E '^(emacs|vi) '
emacs           off
vi              on
```

### Vi mode কীভাবে কাজ করে

Vi mode-এ দুটো অবস্থা (mode) আছে:

```
                Esc
  ┌──────────────┐ ──────────► ┌──────────────────┐
  │ Insert mode  │             │ Command mode     │
  │ (সাধারণ টাইপ)  │ ◄────────── │ (h j k l, w b...) │
  └──────────────┘  i, a, A, I └──────────────────┘
   লাইন শুরু হয়               Enter চাপলে যেকোনো
   এই মোডে                    মোড থেকে কমান্ড চলে
```

### Vi command mode-এর প্রধান কী

| কী | কাজ |
|---|---|
| `Esc` | command mode-এ যাও |
| `i` / `a` | কার্সরের আগে / পরে insert |
| `I` / `A` | লাইনের শুরুতে / শেষে insert |
| `h` / `l` | এক অক্ষর বামে / ডানে |
| `w` / `b` / `e` | পরের শব্দ / আগের শব্দ / শব্দের শেষ |
| `0` / `^` / `$` | লাইনের শুরু / প্রথম অক্ষর / শেষ |
| `k` / `j` | history-তে আগের / পরের কমান্ড |
| `/text` তারপর Enter | history-তে পেছনের দিকে খোঁজো |
| `n` / `N` | পরের / উল্টো দিকের মিল |
| `x` | কার্সরের নিচের অক্ষর মোছো |
| `dw` / `db` / `dd` / `D` | শব্দ মোছো / আগের শব্দ / পুরো লাইন / শেষ পর্যন্ত |
| `cw` / `cc` / `C` | শব্দ / লাইন / শেষ পর্যন্ত বদলাও (মুছে insert) |
| `r` তারপর অক্ষর | একটা অক্ষর প্রতিস্থাপন |
| `~` | বড়/ছোট হাতের অক্ষর উল্টাও |
| `u` | undo |
| `p` / `P` | কার্সরের পরে / আগে পেস্ট |
| `.` | শেষ পরিবর্তন আবার করো |
| `v` | বর্তমান লাইন এডিটরে খোলো (emacs mode-এর `Ctrl+X Ctrl+E`-এর মতো) |
| `fX` / `FX` | সামনে / পেছনে অক্ষর `X`-এ লাফ দাও |

> 📝 **মনে রাখো:** vi mode-এ `Ctrl+A`, `Ctrl+E`, `Ctrl+R`-এর মতো emacs শর্টকাট ডিফল্টভাবে **কাজ করে না** (insert mode-এ কয়েকটা — যেমন `Ctrl+W`, `Ctrl+U` — tty-র কারণে কাজ করে)। তাই অনেকে vi mode চালু করে কিছু emacs শর্টকাট `~/.inputrc`-এ যোগ করে নেয় (নিচে দেখো)। তবে `Ctrl+C`, `Ctrl+Z`, `Ctrl+D` কার্নেল/tty-র বলে দুই মোডেই একই।

### কোনটা বেছে নেবে?

| দিক | Emacs mode | Vi mode |
|---|---|---|
| ডিফল্ট? | হ্যাঁ, সব সার্ভারে | না, চালু করতে হয় |
| শেখা | সহজ — টাইপ করো আর Ctrl চাপো | Vim জানলে স্বাভাবিক, না জানলে কঠিন |
| সার্ভারে কাজ | অন্যের মেশিনেও একই | প্রতিবার `set -o vi` লাগবে |
| পরামর্শ | **নতুনদের জন্য এটাই** | Vim দক্ষতা আসার পর চেষ্টা করো |

স্থায়ীভাবে vi mode চাইলে `~/.bashrc`-এ `set -o vi`, অথবা `~/.inputrc`-এ `set editing-mode vi` (দ্বিতীয়টা readline-ব্যবহারকারী সব প্রোগ্রামে প্রযোজ্য)।

---

## ৬.১৫ `~/.inputrc` — Readline-কে নিজের মতো সাজাও

Readline চালু হওয়ার সময় একটা কনফিগারেশন ফাইল পড়ে:
1. `$INPUTRC` ভেরিয়েবলে দেওয়া ফাইল, না থাকলে
2. `~/.inputrc`, না থাকলে
3. `/etc/inputrc` (সিস্টেম-ব্যাপী)

> ⚠️ **সতর্কতা:** তুমি `~/.inputrc` বানালে readline আর `/etc/inputrc` পড়বে **না**। ফলে distro-র দেওয়া সুবিধা (যেমন `Ctrl+←/→` দিয়ে শব্দ লাফানো, `Home`/`End` কী) হারিয়ে যেতে পারে। তাই নিজের ফাইলের প্রথম লাইনে সবসময় লেখো:
> ```
> $include /etc/inputrc
> ```

### ফাইলের গঠন

তিন রকম লাইন থাকে:

```
# ১. কমেন্ট — # দিয়ে শুরু

# ২. ভেরিয়েবল সেট:  set নাম মান
set completion-ignore-case on

# ৩. কী-বাইন্ডিং:  "কী-সিকোয়েন্স": ফাংশন-নাম   অথবা   "কী": "ম্যাক্রো লেখা"
"\C-x\C-r": re-read-init-file
"\e[A": history-search-backward
```

কী লেখার নিয়ম: `\C-` = Ctrl, `\M-` বা `\e` = Meta/Alt/Esc। যেমন `"\C-a"`, `"\M-f"`, `"\e."`।

### একটা কাজের `~/.inputrc` উদাহরণ

```
$include /etc/inputrc

# ---- Completion ----
set completion-ignore-case on        # Tab-এ বড়-ছোট হাতের পার্থক্য উপেক্ষা: doc<Tab> → Documents/
set completion-map-case on           # - আর _ কে সমান ধরো
set show-all-if-ambiguous on         # একবার Tab-এই সব অপশন দেখাও (double Tab লাগবে না)
set colored-stats on                 # completion তালিকা ls-এর মতো রঙিন
set visible-stats on                 # ফাইলের ধরন চিহ্ন (/ * @) দেখাও
set mark-symlinked-directories on    # symlink ডিরেক্টরির পরে / বসাও
set colored-completion-prefix on     # মিলে যাওয়া অংশ রঙিন করো
set completion-query-items 200       # কতগুলোর বেশি হলে "Display all?" জিজ্ঞেস করবে

# ---- অন্যান্য ----
set bell-style none                  # বিপ বন্ধ (visible দিলে স্ক্রিন ঝলকাবে)
set enable-bracketed-paste on        # পেস্ট নিরাপদ রাখো

# ---- History খোঁজা ↑/↓ দিয়ে (লেখা শুরুর সাথে মিলিয়ে) ----
"\e[A": history-search-backward
"\e[B": history-search-forward

# ---- শুধু bash-এর জন্য ----
$if Bash
  # Alt+S চাপলে লাইনের শুরুতে "sudo " বসাও (ম্যাক্রো)
  "\es": "\C-asudo \C-e"
$endif
```

শেষের `$if Bash ... $endif` অংশটা শুধু bash-এ কাজ করবে, `python3` বা `mysql`-এ না। `$if mode=vi` দিয়ে শুধু vi mode-এর জন্যও বাইন্ডিং দেওয়া যায়।

Vi mode ব্যবহারকারীদের জন্য উপকারী সংযোজন:

```
set editing-mode vi
set show-mode-in-prompt on           # প্রম্পটের আগে (ins)/(cmd) দেখাও
$if mode=vi
  set keymap vi-insert
  "\C-l": clear-screen
  "\C-a": beginning-of-line
  "\C-e": end-of-line
$endif
```

### পরিবর্তন কার্যকর করা

- নতুন টার্মিনাল খোলো, **অথবা**
- বর্তমান শেলে `Ctrl+X Ctrl+R` চাপো (`re-read-init-file`), **অথবা**
- `bind -f ~/.inputrc`

### `bind` কমান্ড — বর্তমান বাইন্ডিং দেখা ও বদলানো

`bind` হলো bash-এর builtin যা দিয়ে readline-এর সেটিং দেখা ও বদলানো যায়।

| কমান্ড | কাজ |
|---|---|
| `bind -P` | সব ফাংশন ও তাদের কী-বাইন্ডিং পড়ার মতো করে দেখাও |
| `bind -p` | ওই তালিকা `.inputrc` ফরম্যাটে দেখাও |
| `bind -l` | সব readline ফাংশনের নাম |
| `bind -v` | সব ভেরিয়েবল ও তাদের মান |
| `bind -q ফাংশন` | ফাংশনটা কোন কী-তে বাঁধা |
| `bind -u ফাংশন` | ফাংশনের সব বাইন্ডিং সরাও |
| `bind -r "কী"` | একটা কী-এর বাইন্ডিং সরাও |
| `bind '"কী": ফাংশন'` | নতুন বাইন্ডিং (এই সেশনে) |
| `bind -x '"কী": শেল-কমান্ড'` | কী চাপলে একটা শেল কমান্ড চালাও |
| `bind -f ফাইল` | একটা inputrc ফাইল পড়ো |

```bash
$ bind -q yank-last-arg
yank-last-arg can be invoked via "\e.", "\e_".

$ bind -P | grep -w kill-line
kill-line can be invoked via "\C-k".

$ bind -x '"\C-xt": date'      # Ctrl+X তারপর T চাপলে তারিখ দেখাবে
$ <Ctrl+X t>
Thu Nov 12 10:30:00 AM +06 2026
```

### একটা কী কোন সিকোয়েন্স পাঠায় জানব কীভাবে?

`Ctrl+V` চেপে তারপর কী-টা চাপো (`quoted-insert`), বা `cat` চালিয়ে কী চাপো:

```bash
$ cat
^[[A        ← ↑ কী (^[ মানে Esc, তাই "\e[A")
^[[1;5C     ← Ctrl+→
^C
```

এই সিকোয়েন্সই `.inputrc`-এ লেখো: `"\e[1;5C": forward-word`।

---

## হাতে-কলমে অনুশীলন

একটা টার্মিনাল খোলো (`Ctrl+Alt+T`)। প্রতিটা ল্যাব নিজে টাইপ করে করো — পড়ে শেখা যায় না, আঙুলে শিখতে হয়।

### ল্যাব ১: কার্সর আর মোছা

1. টাইপ করো (Enter চাপবে না):
   ```bash
   echo one two three four five
   ```
2. `Ctrl+A` — শুরুতে। `Ctrl+E` — শেষে।
3. `Alt+B` তিনবার চাপো — কার্সর `three`-এর শুরুতে।
4. `Alt+D` — `three` মুছে গেল। `Ctrl+Y` — ফিরে এল।
5. `Ctrl+E`, তারপর `Ctrl+W` — `five` গেল। `Ctrl+W` আবার — `four` গেল।
6. `Ctrl+Y` — দেখো কী আসে (শুধু শেষের kill, নাকি দুটো একসাথে?)। পরপর kill করলে readline সেগুলো kill ring-এর একই এন্ট্রিতে জুড়ে দেয়, তাই `four five` একসাথে ফেরত আসবে।
7. `Ctrl+_` কয়েকবার — undo।
8. `Ctrl+U` — পুরো লাইন গেল। `Ctrl+Y` — ফিরে এল। Enter।

### ল্যাব ২: অক্ষর বদল

1. টাইপ করো: `echo hello world`
2. `Alt+B` (world-এর শুরু), `Alt+U` → `WORLD`।
3. `Alt+B` দুবার (hello), `Alt+C` → `Hello`।
4. `Ctrl+E`, তারপর `Alt+T` → শেষ দুটো শব্দ অদলবদল। ফলাফল দেখো, Enter।

### ল্যাব ৩: History expansion

```bash
$ mkdir -p ~/lab6/a/b/c
$ cd !$
$ pwd
$ touch file1.txt file2.txt file3.txt
$ ls -l !*
$ echo ~/lab6/report.final.pdf
$ echo !$:t
$ echo !-2:$:r
$ echo red green blue
$ ^green^yellow
$ !!:gs/e/E/
$ !mkdir:p
$ history 12
```

প্রতিটা লাইনের পর bash কী expand করে প্রিন্ট করল, খেয়াল করো। শেষ লাইনে `history 12`-এ নম্বরগুলো দেখে `!<নম্বর>` দিয়ে একটা কমান্ড আবার চালাও।

### ল্যাব ৪: Ctrl+R

1. এই কমান্ডটা চালাও: `echo "my secret project location" # proj-tag`
2. কয়েকটা অন্য কমান্ড চালাও: `ls`, `date`, `whoami`।
3. `Ctrl+R` চাপো, `proj-tag` লেখো। কমান্ডটা দেখা দেবে।
4. `→` চাপো (চালাবে না) — `secret` শব্দটা বদলে `public` করো, Enter।
5. আবার `Ctrl+R`, `echo` লেখো, তারপর `Ctrl+R` বারবার চাপো — পুরনো echo-গুলো ঘুরে দেখো। `Ctrl+G` দিয়ে বাতিল করো।

### ল্যাব ৫: History কনফিগার

```bash
$ cp ~/.bashrc ~/.bashrc.backup-ch6        # আগে ব্যাকআপ!
$ nano ~/.bashrc                           # একদম শেষে ৬.১১-এর "ভালো সেটআপ" অংশ যোগ করো
$ source ~/.bashrc
$  echo "this has a leading space"
$ ls
$ ls
$ history 5                                # leading-space কমান্ড নেই, ls একবারই
```

এখন দুটো টার্মিনাল খোলো। প্রথমটায় `echo from-terminal-1` চালাও। দ্বিতীয়টায় `history -n` চালিয়ে তারপর `Ctrl+R` দিয়ে `terminal-1` খোঁজো — পেয়ে যাবে।

### ল্যাব ৬: Tab completion ও inputrc

1. `ls /usr/share/doc/` লিখে Tab Tab — তালিকা দেখো।
2. `cd ~/doc` লিখে Tab চাপো — `Documents` বড় হাতের `D`, তাই কিছু হবে না।
3. `~/.inputrc` বানাও:
   ```bash
   $ printf '$include /etc/inputrc\nset completion-ignore-case on\nset show-all-if-ambiguous on\n' > ~/.inputrc
   ```
4. `Ctrl+X Ctrl+R` চাপো। এখন আবার `cd ~/doc<Tab>` — `Documents/` complete হবে!
5. `bind -v | grep completion-ignore-case` চালিয়ে যাচাই করো।

### ল্যাব ৭: Vi mode এক্সপেরিমেন্ট

```bash
$ set -o vi
$ echo vi mode is fun
```
1. `↑` চেপে কমান্ডটা আনো। `Esc` চাপো।
2. `0` (শুরু), `w` দুবার (`mode`), `cw` চেপে `editing` লেখো, `Esc`।
3. `$` (শেষে), `A` চেপে ` indeed` লেখো, Enter।
4. `Esc`, তারপর `/vi` লিখে Enter — history খোঁজা।
5. শেষে `set -o emacs` দিয়ে ফেরো।

### ল্যাব ৮: ফ্রিজ রহস্য

1. `Ctrl+S` চাপো। এখন `ls` লিখে Enter চাপো — কিছু দেখা যাচ্ছে না!
2. `Ctrl+Q` চাপো — `ls` চলে আউটপুট দেখা দেবে।
3. `stty -ixon` চালাও, তারপর `Ctrl+R` দিয়ে কিছু খুঁজে `Ctrl+S` চাপো — এবার forward search কাজ করছে।

পরিষ্কার করা: `rm -r ~/lab6`

---

## সারসংক্ষেপ

### সবচেয়ে জরুরি ২০টা

| শর্টকাট / কমান্ড | কাজ |
|---|---|
| `Ctrl+A` / `Ctrl+E` | লাইনের শুরু / শেষ |
| `Alt+B` / `Alt+F` | এক শব্দ পেছনে / সামনে |
| `Ctrl+U` / `Ctrl+K` | শুরু পর্যন্ত / শেষ পর্যন্ত কাটো |
| `Ctrl+W` / `Alt+D` | আগের শব্দ / পরের শব্দ কাটো |
| `Alt+Backspace` | আগের শব্দ কাটো (`/`-এ থামে) |
| `Ctrl+Y` / `Alt+Y` | পেস্ট / kill ring ঘোরাও |
| `Ctrl+_` | undo |
| `Ctrl+T` / `Alt+T` | অক্ষর / শব্দ অদলবদল |
| `Alt+U` / `Alt+L` / `Alt+C` | UPPER / lower / Capitalize |
| `Ctrl+L` | স্ক্রিন পরিষ্কার |
| `Ctrl+C` | প্রোগ্রাম থামাও (SIGINT) / লাইন বাতিল |
| `Ctrl+D` | EOF — খালি লাইনে শেল থেকে বেরোও |
| `Ctrl+Z` | প্রোগ্রাম suspend (`fg` দিয়ে ফেরত) |
| `Ctrl+S` / `Ctrl+Q` | আউটপুট ফ্রিজ / চালু |
| `Tab` / `Tab Tab` | complete / সব অপশন দেখাও |
| `Ctrl+R` | history-তে খোঁজো |
| `Ctrl+G` | খোঁজা বাতিল |
| `Alt+.` | আগের কমান্ডের শেষ argument |
| `Ctrl+X Ctrl+E` | লাইন এডিটরে খোলো |
| `Alt+#` | লাইন কমেন্ট করে history-তে রাখো |

### History expansion

| চিহ্ন | মানে |
|---|---|
| `!!` | আগের কমান্ড |
| `!n` / `!-n` | n নম্বর / n ধাপ আগের কমান্ড |
| `!str` / `!?str?` | str দিয়ে শুরু / str আছে এমন শেষ কমান্ড |
| `!$` / `!^` / `!*` | শেষ / প্রথম / সব argument |
| `!:n` / `!:n-m` | n নম্বর শব্দ / রেঞ্জ |
| `^old^new` | প্রথম old → new করে আগের কমান্ড চালাও |
| `!!:gs/old/new/` | সব old → new |
| `:h :t :r :e :p` | dirname / basename / extension বাদ / শুধু extension / শুধু প্রিন্ট |

### কমান্ড ও সেটিং

| কমান্ড / ভেরিয়েবল | কাজ |
|---|---|
| `history [n]`, `history -c/-d/-a/-n/-w` | history দেখা ও ব্যবস্থাপনা |
| `fc -l`, `fc`, `fc -s old=new` | তালিকা / এডিটরে খোলো / বদলে চালাও |
| `HISTSIZE`, `HISTFILESIZE` | মেমোরি / ফাইলে কত কমান্ড |
| `HISTCONTROL=ignoreboth:erasedups` | স্পেস-শুরু ও ডুপ্লিকেট বাদ |
| `HISTIGNORE`, `HISTTIMEFORMAT` | প্যাটার্ন বাদ / timestamp |
| `shopt -s histappend histverify` | append করো / expand করে যাচাই |
| `set -o vi` / `set -o emacs` | এডিটিং মোড বদল |
| `~/.inputrc` + `$include /etc/inputrc` | readline কনফিগ |
| `bind -P`, `bind -q`, `bind -x` | বাইন্ডিং দেখো / খোঁজো / কমান্ড বাঁধো |
| `stty -a`, `stty -ixon` | tty কী দেখো / Ctrl+S ফ্রিজ বন্ধ |
| `Ctrl+Shift+C/V/T/N/W` | টার্মিনাল: কপি / পেস্ট / ট্যাব / উইন্ডো / ট্যাব বন্ধ |

---

## নিজেকে যাচাই করো

1. টার্মিনালে `Ctrl+C` চাপলে কেন লেখা কপি হয় না? কপি করতে কী চাপবে?
2. `Ctrl+W` আর `Alt+Backspace`-এর মধ্যে পার্থক্য কী? `cd /etc/nginx/sites-available` লাইনের শেষে দুটো চাপলে কী কী মুছবে?
3. তুমি `apt update` চালালে, "Permission denied" এলো। সবচেয়ে কম টাইপে কীভাবে sudo দিয়ে আবার চালাবে?
4. হঠাৎ টার্মিনালে কিছু টাইপ করলে দেখা যাচ্ছে না। সম্ভাব্য কারণ কী আর সমাধান কী?
5. `HISTSIZE` আর `HISTFILESIZE`-এর পার্থক্য কী?
6. একটা কমান্ড চালাতে চাও যেটা history-তে সেভ হবে না। কোন সেটিং থাকলে কীভাবে করবে?
7. `Ctrl+R` দিয়ে কমান্ড পেলে, সেটা না চালিয়ে এডিট করতে কী চাপবে? আর খোঁজা পুরো বাতিল করতে?
8. `~/.inputrc` বানানোর পর `Ctrl+←` কাজ করা বন্ধ করে দিল। কেন, আর কীভাবে ঠিক করবে?
9. `!$` আর `Alt+.`-এর মধ্যে মূল পার্থক্য কী? কোনটা বেশি নিরাপদ?
10. `ls /tmp/report.tar.gz` চালানোর ঠিক পরে `echo !$:r` চালালে কী দেখাবে? আর তার ঠিক পরে `echo !$:t` চালালে?

<details><summary>উত্তর দেখো</summary>

1. টার্মিনালে `Ctrl+C` বাইট `0x03` পাঠায়, যা কার্নেলের tty line discipline ধরে চলমান প্রোগ্রামকে `SIGINT` signal পাঠায় — প্রোগ্রাম থামানোর জন্য এটা সংরক্ষিত। কপি করতে `Ctrl+Shift+C` (পেস্ট `Ctrl+Shift+V`), অথবা মাউসে সিলেক্ট করে মাঝের বাটনে পেস্ট।
2. `Ctrl+W` স্পেস পর্যন্ত মোছে, তাই পুরো `/etc/nginx/sites-available` মুছে যাবে। `Alt+Backspace` শুধু alphanumeric শব্দ মোছে — প্রথমবার `available` মুছবে (`-` এ থামবে), থাকবে `cd /etc/nginx/sites-`।
3. `sudo !!` (`histverify` চালু থাকলে একবার বাড়তি Enter লাগবে)। বিকল্প: `↑`, `Ctrl+A`, `sudo ` লিখে Enter।
4. সম্ভবত ভুলে `Ctrl+S` চাপা হয়েছে, যা XON/XOFF flow control দিয়ে আউটপুট ফ্রিজ করে। `Ctrl+Q` চাপলে ঠিক হবে। স্থায়ীভাবে এড়াতে `~/.bashrc`-এ `stty -ixon`।
5. `HISTSIZE` = মেমোরিতে (বর্তমান সেশনে) কতগুলো কমান্ড থাকবে; `HISTFILESIZE` = `~/.bash_history` ফাইলে সর্বোচ্চ কত লাইন রাখা হবে।
6. `HISTCONTROL`-এ `ignorespace` বা `ignoreboth` থাকলে কমান্ডের আগে একটা স্পেস দিয়ে লেখো। (ভুলে সেভ হয়ে গেলে `history -d <নম্বর>` তারপর `history -w`।)
7. `Esc`, `→`, `←`, `Ctrl+E` বা `Ctrl+A`-এর মতো যেকোনো movement কী চাপলে কমান্ড লাইনে থেকে যাবে এডিটের জন্য। পুরো বাতিল করতে `Ctrl+G`।
8. `~/.inputrc` থাকলে readline `/etc/inputrc` আর পড়ে না, আর distro-র `Ctrl+←/→` বাইন্ডিং সেখানেই ছিল। সমাধান: `~/.inputrc`-এর প্রথম লাইনে `$include /etc/inputrc` লিখে `Ctrl+X Ctrl+R` চাপো।
9. `Alt+.` লেখাটা সাথে সাথে লাইনে বসায়, তাই Enter চাপার আগে দেখে নেওয়া যায়; `!$` Enter চাপার পর expand হয়ে সরাসরি চলে (যদি না `histverify` চালু থাকে)। তাই `Alt+.` বেশি নিরাপদ।
10. প্রথমটা: `!$` = `/tmp/report.tar.gz`, আর `:r` শুধু **শেষ** extension সরায় → `/tmp/report.tar`। দ্বিতীয়টা: এখন ঠিক আগের কমান্ড হলো `echo /tmp/report.tar`, তাই `!$` = `/tmp/report.tar`, আর `:t` শুধু শেষ অংশ রাখে → `report.tar`। শিক্ষা: `!$` সবসময় **ঠিক আগের** কমান্ড দেখে — মাঝে অন্য কমান্ড চালালে ফল বদলে যায়।

</details>

---

[← আগের অধ্যায়](05-terminal-o-shell.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](07-filesystem-hierarchy.md)

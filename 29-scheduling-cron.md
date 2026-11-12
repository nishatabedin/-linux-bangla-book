# অধ্যায় ২৯: কাজ শিডিউল করা — cron, at ও systemd timers

> "যে কাজ দুইবার হাতে করেছ, তৃতীয়বার সেটা কম্পিউটারকে দিয়ে করাও।" — সিস্টেম অ্যাডমিনদের পুরোনো কথা

ভাবো তোমার একটা সার্ভার আছে। প্রতি রাত ২টায় ডাটাবেসের ব্যাকআপ নিতে হবে, প্রতি সপ্তাহে পুরোনো লগ মুছতে হবে, প্রতি ৫ মিনিটে ওয়েবসাইট চালু আছে কিনা দেখতে হবে। তুমি কি প্রতি রাতে ২টায় অ্যালার্ম দিয়ে উঠে কমান্ড চালাবে? অবশ্যই না। লিনাক্সে এই কাজগুলো **শিডিউল** (schedule — নির্দিষ্ট সময়ে স্বয়ংক্রিয়ভাবে চালানো) করে রাখা যায়। এই অধ্যায়ে আমরা শিখব সেই তিনটি প্রধান হাতিয়ার: **cron**, **at** এবং **systemd timers**।

## এই অধ্যায়ে যা শিখবে

- শিডিউলিং কেন দরকার, আর লিনাক্সে কোন কাজের জন্য কোন টুল
- cron daemon কী, ভেতরে কীভাবে প্রতি মিনিটে কাজ খোঁজে
- `crontab -e`, `-l`, `-r`, `-u` — ইউজার crontab ম্যানেজ করা
- cron-এর ৫ ফিল্ডের সিনট্যাক্স — ডায়াগ্রাম, অপারেটর (`*` `,` `-` `/`) ও প্রচুর উদাহরণ
- বিশেষ স্ট্রিং: `@reboot`, `@daily`, `@hourly` ইত্যাদি
- সিস্টেম crontab: `/etc/crontab`, `/etc/cron.d/`, `/etc/cron.daily/` ইত্যাদি
- cron-এর সবচেয়ে বড় ফাঁদ: environment/PATH সমস্যা, `%` চিহ্ন, আউটপুট ও লগ
- anacron — যে কম্পিউটার সবসময় চালু থাকে না তার জন্য
- `at`, `atq`, `atrm`, `batch` — একবারের কাজ শিডিউল করা
- systemd timer — `OnCalendar`, `Persistent`, `OnBootSec` সহ সম্পূর্ণ উদাহরণ
- cron বনাম systemd timer — কখন কোনটা বেছে নেবে
- কে cron/at ব্যবহার করতে পারবে — `cron.allow`/`cron.deny`

---

## ২৯.১ শিডিউলিং কেন? বড় ছবিটা

দৈনন্দিন উপমা দিয়ে শুরু করি। তোমার মোবাইলে তিন ধরনের অ্যালার্ম/রিমাইন্ডার থাকে:

| মোবাইলের জিনিস | লিনাক্সে সমতুল্য | কাজ |
|---|---|---|
| প্রতিদিন সকাল ৭টার অ্যালার্ম (repeat) | **cron** / **systemd timer** | বারবার, নির্দিষ্ট সময়ে |
| "আজ বিকেল ৫টায় মাকে ফোন করো" রিমাইন্ডার (একবার) | **at** | একবারই, নির্দিষ্ট সময়ে |
| "ফোন চার্জে থাকলে আপডেট করো" | **batch** | একবার, যখন সিস্টেম ফ্রি থাকবে |
| "ফোন বন্ধ ছিল, অ্যালার্ম মিস হয়েছে — চালু হলে জানাও" | **anacron** / `Persistent=true` | মিস হওয়া কাজ পরে চালানো |

লিনাক্সে শিডিউলিং-এর কিছু সাধারণ ব্যবহার:

- ব্যাকআপ নেওয়া ([অধ্যায় ৩৫](35-backup-rsync.md))
- লগ রোটেট ও পরিষ্কার করা ([অধ্যায় ২৮](28-logs.md)-এর logrotate নিজেই cron/timer দিয়ে চলে)
- SSL সার্টিফিকেট রিনিউ করা (certbot)
- প্যাকেজ আপডেট চেক করা
- রিপোর্ট বানিয়ে ইমেইল করা
- মনিটরিং স্ক্রিপ্ট চালানো

### টুলগুলোর পরিবার

```
                    শিডিউলিং টুল
                         │
        ┌────────────────┼──────────────────┐
        │                │                  │
   বারবার চলে       একবার চলে         systemd-এর অংশ
        │                │                  │
   ┌────┴────┐       ┌───┴───┐         systemd timer
 cron     anacron    at    batch     (.timer + .service)
 (মিনিট-  (দিন-      (নির্দিষ্ট (লোড কম    (বারবার বা একবার,
  ভিত্তিক) ভিত্তিক)   সময়)    হলে)       দুটোই পারে)
```

---

## ২৯.২ cron daemon — ভেতরে কীভাবে কাজ করে

### daemon কী?

**daemon** (ডিমন) হলো এমন একটা প্রোগ্রাম যা ব্যাকগ্রাউন্ডে সবসময় চলতে থাকে, কোনো টার্মিনাল ছাড়াই ([অধ্যায় ২৪](24-process-management.md) ও [অধ্যায় ২৬](26-systemd-services.md) দেখো)। cron হলো এমনই একটা daemon। নাম এসেছে গ্রিক শব্দ *chronos* (সময়) থেকে।

উপমা: cron হলো অফিসের একজন অত্যন্ত নিয়মনিষ্ঠ দারোয়ান। তার হাতে একটা খাতা (crontab)। প্রতি মিনিটে সে ঘড়ি দেখে, খাতা খুলে মিলিয়ে দেখে — "এই মিনিটে কারো কোনো কাজ আছে?" থাকলে সে কাজটা করিয়ে দেয়, তারপর আবার পরের মিনিটের জন্য অপেক্ষা করে।

### distro-ভেদে নাম

| Distro | প্যাকেজ | সার্ভিসের নাম | ইউজার crontab কোথায় থাকে |
|---|---|---|---|
| Ubuntu 24.04 / Debian 12 | `cron` | `cron.service` | `/var/spool/cron/crontabs/<user>` |
| Fedora / RHEL 9 | `cronie` | `crond.service` | `/var/spool/cron/<user>` |

সার্ভিস চলছে কিনা দেখো:

```
$ systemctl status cron          # Ubuntu/Debian
● cron.service - Regular background program processing daemon
     Loaded: loaded (/usr/lib/systemd/system/cron.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-12 09:01:12 +06; 3h ago
       Docs: man:cron(8)
   Main PID: 812 (cron)
```

```
$ systemctl status crond         # Fedora/RHEL
```

না থাকলে ইনস্টল করো ([অধ্যায় ৩০](30-package-management.md)):

```
$ sudo apt install cron                       # Ubuntu/Debian
$ sudo dnf install cronie                     # Fedora/RHEL
$ sudo systemctl enable --now crond           # Fedora/RHEL-এ চালু করো
```

> 📝 মনে রাখো: কিছু মিনিমাল কন্টেইনার ইমেজ বা ক্লাউড ইমেজে cron ইনস্টল করা থাকে না। Fedora-র নতুন ভার্সনগুলো অনেক কাজ systemd timer দিয়ে করে, তাই সেখানেও cronie আলাদা করে লাগতে পারে।

### ভেতরের কাজের ধারা (under the hood)

```
   ┌──────────────────────────────────────────────────────────┐
   │                    cron daemon (PID 812)                  │
   │                                                          │
   │  ১. চালু হওয়ার সময় সব crontab পড়ে মেমোরিতে রাখে:        │
   │     /var/spool/cron/crontabs/*   (ইউজার crontab)          │
   │     /etc/crontab                  (সিস্টেম crontab)       │
   │     /etc/cron.d/*                 (প্যাকেজের crontab)     │
   │                                                          │
   │  ২. প্রতি মিনিটের শুরুতে জেগে ওঠে                         │
   │  ৩. ফাইলগুলোর modification time বদলেছে? → আবার পড়ে       │
   │  ৪. এই মিনিটের সাথে মেলে এমন প্রতিটি লাইনের জন্য:         │
   │       fork() → child process                             │
   │       child: ইউজার বদলায় (setuid) → /bin/sh -c "কমান্ড"  │
   │  ৫. child-এর আউটপুট ধরে → MAILTO-তে মেইল / লগ             │
   │  ৬. আবার ঘুম, পরের মিনিট পর্যন্ত                          │
   └──────────────────────────────────────────────────────────┘
```

গুরুত্বপূর্ণ বিষয়গুলো:

- cron-এর সবচেয়ে ছোট একক হলো **১ মিনিট**। সেকেন্ড-ভিত্তিক শিডিউল cron পারে না (systemd timer পারে)।
- cron নিজে কমান্ড চালায় না — প্রতিটা কাজের জন্য একটা নতুন child process বানিয়ে `/bin/sh` দিয়ে চালায়। তাই কমান্ডটা **sh** সিনট্যাক্সে হতে হবে, bash-এর বিশেষ জিনিস (যেমন `[[ ]]`) সরাসরি নাও চলতে পারে। (Ubuntu/Debian-এ `/bin/sh` আসলে `dash`।)
- crontab ফাইল বদলালে cron রিস্টার্ট করতে হয় না — সে নিজেই প্রতি মিনিটে চেক করে।
- cron কাজ শেষ হওয়ার জন্য অপেক্ষা করে না। আগের রান শেষ না হতেই পরের মিনিটে আবার শুরু হতে পারে — এটা একটা বড় ফাঁদ (২৯.১০-এ `flock` দেখো)।

---

## ২৯.৩ crontab কমান্ড — নিজের কাজের খাতা

**crontab** শব্দের দুটো অর্থ: (১) একটা ফাইল যাতে শিডিউল লেখা থাকে (cron table), (২) সেই ফাইল এডিট করার কমান্ড।

> ⚠️ সতর্কতা: `/var/spool/cron/...` ফাইল কখনো সরাসরি এডিট করবে না। সবসময় `crontab` কমান্ড ব্যবহার করো। কারণ এটি সিনট্যাক্স চেক করে, সঠিক পারমিশন দেয়, আর cron-কে জানায়।

### সিনট্যাক্স

```
crontab [-u user] -e | -l | -r | -i
crontab [-u user] file
```

| অপশন | কাজ |
|---|---|
| `-e` | নিজের crontab এডিটরে খোলে (না থাকলে নতুন বানায়) |
| `-l` | crontab-এর বিষয়বস্তু দেখায় (list) |
| `-r` | পুরো crontab **মুছে ফেলে** (remove) — কোনো প্রশ্ন ছাড়াই! |
| `-i` | `-r`-এর সাথে ব্যবহার করলে মোছার আগে নিশ্চিত হতে চায় |
| `-u user` | অন্য ইউজারের crontab (শুধু root/sudo) |
| `file` | একটা ফাইল থেকে crontab ইনস্টল করে — **পুরোনোটা পুরো বদলে যায়** |
| `-` | stdin থেকে crontab পড়ে |

### প্রথমবার `crontab -e`

Ubuntu/Debian-এ প্রথমবার চালালে এডিটর বাছতে বলবে:

```
$ crontab -e
no crontab for nishat - using an empty one

Select an editor.  To change later, run 'select-editor'.
  1. /bin/nano        <---- easiest
  2. /usr/bin/vim.basic
  3. /usr/bin/vim.tiny
  4. /bin/ed

Choose 1-4 [1]: 1
```

নতুন হলে `nano` বেছে নাও ([অধ্যায় ১৩](13-nano-editor.md))। পরে বদলাতে `select-editor` চালাও, অথবা `EDITOR` ভ্যারিয়েবল সেট করো:

```
$ EDITOR=vim crontab -e
```

Fedora/RHEL-এ ডিফল্ট এডিটর সাধারণত `vi` (RHEL 9) বা `EDITOR` ভ্যারিয়েবল অনুযায়ী (Fedora-য় ডিফল্ট `nano`)।

এডিটরে একটা লাইন লেখো, সেভ করে বের হও। তখন দেখবে:

```
crontab: installing new crontab
```

ভুল সিনট্যাক্স হলে:

```
"/tmp/crontab.Xb3kQe/crontab":23: bad minute
errors in crontab file, can't install.
Do you want to retry the same edit? (y/n)
```

### `crontab -l` — দেখো

```
$ crontab -l
# m h  dom mon dow   command
0 2 * * * /home/nishat/bin/backup.sh
```

crontab না থাকলে:

```
$ crontab -l
no crontab for nishat
```

### `crontab -r` — ভয়ংকর অপশন

```
$ crontab -r
$ crontab -l
no crontab for nishat
```

কোনো প্রশ্ন নেই, কোনো undo নেই! আর কীবোর্ডে `-e` আর `-r` পাশাপাশি — একটু হাত ফসকালেই সব শেষ।

> 💡 টিপস: নিয়মিত ব্যাকআপ রাখো: `crontab -l > ~/crontab.backup`। ফেরত আনতে: `crontab ~/crontab.backup`। আর নিরাপদ থাকতে মুছতে চাইলে `crontab -ri` ব্যবহার করো:
> ```
> $ crontab -ri
> crontab: really delete nishat's crontab? (y/n) n
> ```

### অন্য ইউজারের crontab

```
$ sudo crontab -u www-data -l
$ sudo crontab -u www-data -e
```

### ফাইল থেকে ইনস্টল ও স্ক্রিপ্টে লাইন যোগ করা

```
$ crontab mycron.txt                  # পুরোনো crontab মুছে এটা বসে যায়
$ (crontab -l 2>/dev/null; echo "*/5 * * * * /usr/local/bin/check.sh") | crontab -
```

দ্বিতীয় কমান্ডটা পুরোনো লাইন রেখে একটা নতুন লাইন যোগ করে — স্ক্রিপ্টে অটোমেশনের জন্য খুব কাজের ([অধ্যায় ১৯](19-redirection-pipe.md)-এর পাইপ ব্যবহার করে)।

---

## ২৯.৪ cron-এর ৫ ফিল্ড সিনট্যাক্স — সবচেয়ে গুরুত্বপূর্ণ অংশ

প্রতিটি crontab লাইনে ৫টি সময়-ফিল্ড, তারপর কমান্ড:

```
 ┌───────────── মিনিট (minute)        0 - 59
 │ ┌─────────── ঘণ্টা (hour)          0 - 23   (২৪ ঘণ্টার হিসাব)
 │ │ ┌───────── মাসের দিন (day of month) 1 - 31
 │ │ │ ┌─────── মাস (month)          1 - 12  অথবা jan - dec
 │ │ │ │ ┌───── সপ্তাহের দিন (day of week) 0 - 7  (0 ও 7 দুটোই রবিবার) অথবা sun - sat
 │ │ │ │ │
 │ │ │ │ │
 * * * * *  কমান্ড যা চালাতে হবে
```

মনে রাখার কৌশল: **"মি-ঘ-দি-মা-স"** — মিনিট, ঘণ্টা, দিন, মাস, সপ্তাহ। ছোট থেকে বড়, শুধু শেষেরটা (সপ্তাহের দিন) আলাদা।

### সপ্তাহের দিনের সংখ্যা

| সংখ্যা | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|---|
| দিন | রবি (sun) | সোম (mon) | মঙ্গল (tue) | বুধ (wed) | বৃহঃ (thu) | শুক্র (fri) | শনি (sat) | রবি (sun) |

### অপারেটর

| চিহ্ন | অর্থ | উদাহরণ | ব্যাখ্যা |
|---|---|---|---|
| `*` | প্রতিটি মান | `*` ঘণ্টায় | প্রতি ঘণ্টা |
| `,` | তালিকা | `1,15,30` | ১, ১৫ আর ৩০ তারিখে |
| `-` | পরিসর (range) | `1-5` সপ্তাহে | সোম থেকে শুক্র |
| `/` | ধাপ (step) | `*/10` মিনিটে | প্রতি ১০ মিনিট (0,10,20,...) |
| মিশ্র | একসাথে | `0-30/10` | 0, 10, 20, 30 মিনিটে |
| নাম | মাস/দিনের নাম | `jan`, `mon` | তিন অক্ষর, case-insensitive |

> 📝 মনে রাখো: নাম (যেমন `mon`) range-এ ব্যবহার করা নিয়ে cron বাস্তবায়নে পার্থক্য আছে — Debian-এর cron-এর man page বলে নামের range বা list চলবে না। নিরাপদে থাকতে range-এ সংখ্যা ব্যবহার করো: `1-5`, `mon-fri` নয়।

`*/10` কীভাবে কাজ করে: `*` মানে 0-59, `/10` মানে সেখান থেকে প্রতি ১০টা পরপর নাও → 0, 10, 20, 30, 40, 50। তাই `*/7` মিনিটে মানে 0, 7, 14, ..., 56 — তারপর আবার 0। অর্থাৎ ঘণ্টার শেষে 56 থেকে 0-তে মাত্র ৪ মিনিটের ফাঁক! `/` ধাপ প্রতি ঘণ্টায় নতুন করে গোনা শুরু হয়, একটানা না।

### উদাহরণ টেবিল (মুখস্থ না, বুঝে নাও)

| crontab লাইন | কখন চলবে |
|---|---|
| `* * * * *` | প্রতি মিনিটে |
| `*/5 * * * *` | প্রতি ৫ মিনিটে |
| `0 * * * *` | প্রতি ঘণ্টার শুরুতে (১:০০, ২:০০ ...) |
| `30 * * * *` | প্রতি ঘণ্টার ৩০ মিনিটে |
| `0 2 * * *` | প্রতিদিন রাত ২:০০টায় |
| `30 14 * * *` | প্রতিদিন দুপুর ২:৩০-এ |
| `0 9 * * 1-5` | সোম–শুক্র সকাল ৯টায় |
| `0 9 * * 0,6` | শনি ও রবিবার সকাল ৯টায় |
| `0 0 1 * *` | প্রতি মাসের ১ তারিখ মধ্যরাতে |
| `0 0 1 1 *` | প্রতি বছর ১ জানুয়ারি মধ্যরাতে |
| `15 10 * * 5` | প্রতি শুক্রবার সকাল ১০:১৫-এ |
| `0 */6 * * *` | প্রতি ৬ ঘণ্টায় (০, ৬, ১২, ১৮টায়) |
| `0 9-17 * * 1-5` | অফিস টাইমে (৯টা–৫টা) প্রতি ঘণ্টায়, কর্মদিবসে |
| `*/15 9-17 * * 1-5` | অফিস টাইমে প্রতি ১৫ মিনিটে |
| `0 0 * * 0` | প্রতি রবিবার মধ্যরাতে |
| `0 3 1,15 * *` | প্রতি মাসের ১ ও ১৫ তারিখ রাত ৩টায় |
| `0 0 1 */3 *` | প্রতি তিন মাসে একবার (জানু, এপ্রিল, জুলাই, অক্টো ১ তারিখ) |
| `5 4 * * sun` | প্রতি রবিবার ভোর ৪:০৫-এ |
| `0 22 * * 1-5` | কর্মদিবসে রাত ১০টায় |

### ফাঁদ: দিন আর সপ্তাহের দিন একসাথে (OR লজিক)

ধরো লিখলে:

```
0 9 13 * 5    /usr/local/bin/script.sh
```

তুমি হয়তো ভাবছ "১৩ তারিখ যদি শুক্রবার হয় তবে চলবে"। **ভুল!** যখন মাসের দিন (৩য় ফিল্ড) আর সপ্তাহের দিন (৫ম ফিল্ড) দুটোই `*` না, cron দুটোর মধ্যে **OR** করে: প্রতি মাসের ১৩ তারিখে **অথবা** প্রতি শুক্রবারে চলবে।

"মাসের শেষ শুক্রবার" বা "১৩ তারিখ শুক্রবার" এর মতো শর্ত দিতে স্ক্রিপ্টের ভেতরে চেক করো:

```
0 9 13 * *   [ "$(date +\%u)" -eq 5 ] && /usr/local/bin/script.sh
```

(এখানে `\%` কেন — পরের অংশে দেখো।)

### `%` চিহ্নের ফাঁদ

crontab-এর কমান্ড অংশে `%` চিহ্নের বিশেষ অর্থ আছে: এটা **নতুন লাইন** হিসেবে গণ্য হয়, আর প্রথম `%`-এর পরের সব কিছু কমান্ডের stdin-এ চলে যায়। তাই `date +%Y-%m-%d` crontab-এ লিখলে কাজ করবে না।

```
# ভুল — কমান্ড ভেঙে যাবে
0 2 * * * tar czf /backup/home-$(date +%F).tar.gz /home

# ঠিক — % এর আগে ব্যাকস্ল্যাশ
0 2 * * * tar czf /backup/home-$(date +\%F).tar.gz /home
```

> 💡 টিপস: সবচেয়ে ভালো অভ্যাস হলো জটিল কমান্ড crontab-এ না লিখে একটা স্ক্রিপ্ট ফাইলে লেখা ([অধ্যায় ৪১](41-shell-scripting-1.md)), আর crontab-এ শুধু স্ক্রিপ্টটা ডাকা। তাহলে `%`, quoting, PATH — সব ঝামেলা কমে যায়।

### crontab ফাইলে আর কী লেখা যায়

```
# '#' দিয়ে শুরু হওয়া লাইন কমেন্ট
# ফাঁকা লাইনও চলে

# environment ভ্যারিয়েবল সেট করা (লাইনের শুরুতে NAME=value)
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=admin@example.com

# কাজ
0 2 * * * /home/nishat/bin/backup.sh
```

> ⚠️ সতর্কতা: crontab ফাইলের শেষ লাইনের পরে অবশ্যই একটা newline থাকতে হবে। না থাকলে কিছু cron বাস্তবায়ন শেষ লাইনটা উপেক্ষা করে (Debian cron "premature EOF" এরর দেয়)। `crontab -e` দিয়ে এডিট করলে সাধারণত এডিটর নিজেই newline দেয়।

> 📝 মনে রাখো: crontab-এর ভ্যারিয়েবল লাইনে shell expansion হয় না। `PATH=$PATH:/opt/bin` লিখলে `$PATH` আক্ষরিক টেক্সট হিসেবেই থাকবে। পুরো পাথ নিজে লিখে দাও।

### টাইমজোন

cron সিস্টেমের লোকাল টাইমজোন অনুযায়ী চলে (`timedatectl` দিয়ে দেখো)। সার্ভার UTC-তে থাকলে "রাত ২টা" মানে UTC রাত ২টা — বাংলাদেশে তখন সকাল ৮টা!

```
$ timedatectl | grep "Time zone"
                Time zone: Asia/Dhaka (+06, +0600)
```

cronie (Fedora/RHEL) crontab-এর ভেতরে `CRON_TZ=Asia/Dhaka` ভ্যারিয়েবল সমর্থন করে, যা পরের লাইনগুলোর সময় সেই টাইমজোনে ধরে। Debian/Ubuntu-র cron-এ এটা নির্ভরযোগ্য নয় — সেখানে সিস্টেম টাইমজোনের উপর ভরসা করো, বা systemd timer ব্যবহার করো (যা `OnCalendar`-এ টাইমজোন নিতে পারে)।

ডেলাইট সেভিং টাইম (DST) আছে এমন দেশে ঘড়ি এগোনো/পেছানোর সময় কিছু কাজ একবার বাদ পড়তে বা দুবার চলতে পারে। বাংলাদেশে DST নেই, তবে বিদেশি সার্ভারে কাজ করলে মাথায় রেখো — DST-এর সময় (সাধারণত রাত ১টা–৩টা) জরুরি কাজ শিডিউল না করাই ভালো।

---

## ২৯.৫ বিশেষ স্ট্রিং — `@reboot`, `@daily` ইত্যাদি

৫টা ফিল্ডের বদলে কিছু শর্টকাট শব্দ ব্যবহার করা যায়:

| স্ট্রিং | অর্থ | সমতুল্য |
|---|---|---|
| `@reboot` | cron daemon চালু হওয়ার সময় একবার (অর্থাৎ বুটে) | — |
| `@yearly` | বছরে একবার | `0 0 1 1 *` |
| `@annually` | `@yearly`-এর মতোই | `0 0 1 1 *` |
| `@monthly` | মাসে একবার | `0 0 1 * *` |
| `@weekly` | সপ্তাহে একবার (রবিবার মধ্যরাত) | `0 0 * * 0` |
| `@daily` | দিনে একবার | `0 0 * * *` |
| `@midnight` | `@daily`-এর মতোই | `0 0 * * *` |
| `@hourly` | ঘণ্টায় একবার | `0 * * * *` |

উদাহরণ:

```
@reboot   /home/nishat/bin/start-my-app.sh
@daily    /usr/local/bin/cleanup-tmp.sh
@hourly   /usr/local/bin/sync-data.sh
```

> ⚠️ সতর্কতা: `@reboot` আসলে "cron daemon চালু হলে" চলে, এটা নিশ্চিত করে না যে নেটওয়ার্ক বা ডাটাবেস তখন তৈরি। বুটে কোনো সার্ভিস চালাতে হলে systemd service ([অধ্যায় ২৬](26-systemd-services.md)) অনেক ভালো — সেখানে `After=network-online.target` দিয়ে নির্ভরতা বলা যায়। আর Debian cron-এ `cron` সার্ভিস রিস্টার্ট করলে `@reboot` কাজ আবার চলে না (এটা বুট শনাক্ত করে একটা ফাইল দিয়ে)।

---

## ২৯.৬ সিস্টেম crontab — `/etc/crontab`, `/etc/cron.d/` ও `cron.daily`

ইউজার crontab ছাড়াও সিস্টেম-লেভেলের শিডিউল থাকে। এগুলো root এডিট করে, আর এদের একটা বড় পার্থক্য আছে: **ইউজারের নাম লেখার একটা অতিরিক্ত ফিল্ড**।

```
ইউজার crontab (crontab -e):
  মিনিট ঘণ্টা দিন মাস সপ্তাহ        কমান্ড
  0     2     *   *   *              /home/nishat/backup.sh

সিস্টেম crontab (/etc/crontab, /etc/cron.d/*):
  মিনিট ঘণ্টা দিন মাস সপ্তাহ  ইউজার  কমান্ড
  0     2     *   *   *       root   /usr/local/bin/backup.sh
                              ^^^^
                              কোন ইউজার হিসেবে চলবে
```

### পুরো কাঠামো এক নজরে

```
/etc/
├── crontab              ← প্রধান সিস্টেম crontab (ইউজার ফিল্ড সহ)
├── cron.d/              ← প্যাকেজগুলো নিজেদের crontab এখানে রাখে (ইউজার ফিল্ড সহ)
│   ├── e2scrub_all
│   ├── php              (php ইনস্টল থাকলে, সেশন পরিষ্কারের জন্য)
│   └── certbot          (কিছু ইনস্টলেশনে)
├── cron.hourly/         ← এখানে রাখা স্ক্রিপ্ট ঘণ্টায় একবার চলে
├── cron.daily/          ← দিনে একবার
│   ├── apt-compat
│   ├── dpkg
│   ├── logrotate        (অনেক সিস্টেমে এখন systemd timer দিয়ে চলে)
│   └── man-db
├── cron.weekly/         ← সপ্তাহে একবার
├── cron.monthly/        ← মাসে একবার
├── cron.allow / cron.deny  ← কে crontab ব্যবহার করতে পারবে
└── anacrontab           ← anacron-এর কনফিগ
```

### `/etc/crontab` — Ubuntu/Debian

```
$ cat /etc/crontab
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.daily; }
47 6    * * 7   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.weekly; }
52 6    1 * *   root    test -x /usr/sbin/anacron || { cd / && run-parts --report /etc/cron.monthly; }
```

(ভার্সন ভেদে `{ ...; }`-এর বদলে `( ... )` দেখতে পারো — অর্থ একই।)

লাইনগুলো ভেঙে বুঝি:

- **প্রথম লাইন**: প্রতি ঘণ্টার ১৭ মিনিটে `run-parts` দিয়ে `/etc/cron.hourly/`-এর সব স্ক্রিপ্ট চালায়।
- **বাকি তিন লাইন**: `test -x /usr/sbin/anacron ||` মানে — "যদি anacron ইনস্টল থাকে, তাহলে কিছু করো না (anacron নিজেই daily/weekly/monthly সামলাবে); না থাকলে এখানেই চালাও।" ২৯.৮-এ anacron বিস্তারিত দেখব।

কেন ১৭, ২৫, ৪৭, ৫২-এর মতো অদ্ভুত মিনিট? যাতে হাজার হাজার সার্ভার একই সাথে ঠিক ০ মিনিটে ভারী কাজ শুরু না করে, আর বিভিন্ন কাজ একে অপরের সাথে ধাক্কা না খায়।

### `/etc/crontab` — Fedora/RHEL

RHEL 9-এ `/etc/crontab`-এ শুধু কমেন্ট আর ভ্যারিয়েবল থাকে, কোনো কাজ থাকে না। ঘণ্টার কাজটা আসে `/etc/cron.d/0hourly` থেকে:

```
$ cat /etc/cron.d/0hourly
# Run the hourly jobs
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
01 * * * * root run-parts /etc/cron.hourly
```

আর `/etc/cron.hourly/0anacron` স্ক্রিপ্টটি anacron চালায়, যা daily/weekly/monthly সামলায়।

### `run-parts` কী?

`run-parts` একটা ছোট প্রোগ্রাম যা একটা ডিরেক্টরির ভেতরের সব executable ফাইল বর্ণানুক্রমে একে একে চালায়। তাই `/etc/cron.daily/`-এ একটা স্ক্রিপ্ট রেখে দিলেই সেটা দিনে একবার চলবে — কোনো সময় লিখতে হয় না।

```
$ run-parts --test /etc/cron.daily        # কী কী চলবে দেখায়, চালায় না (Debian)
/etc/cron.daily/apt-compat
/etc/cron.daily/dpkg
/etc/cron.daily/man-db
```

> ⚠️ সতর্কতা: Debian/Ubuntu-র `run-parts` ফাইলের নামে **ডট (`.`) থাকলে সেই ফাইল উপেক্ষা করে**! নাম শুধু অক্ষর, সংখ্যা, `_` আর `-` দিয়ে হতে হবে। তাই `/etc/cron.daily/backup.sh` কখনো চলবে না — নাম রাখো `backup`। এটা নতুনদের সবচেয়ে বেশি হওয়া ভুলগুলোর একটা। একই নিয়ম `/etc/cron.d/`-এর ফাইলের নামেও প্রযোজ্য। আর ফাইলটা অবশ্যই executable হতে হবে: `sudo chmod +x /etc/cron.daily/backup`।

### `/etc/cron.d/` — নিজের সিস্টেম কাজ রাখার সেরা জায়গা

`/etc/crontab` সরাসরি এডিট করার চেয়ে `/etc/cron.d/`-এ আলাদা ফাইল বানানো ভালো — প্যাকেজ আপডেটে হারায় না, আর প্রতিটা কাজ আলাদা ফাইলে গোছানো থাকে।

```
$ sudo nano /etc/cron.d/db-backup
```

```
# প্রতিদিন রাত ২:৩০-এ postgres ইউজার হিসেবে ডাটাবেস ব্যাকআপ
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
MAILTO=admin@example.com

30 2 * * * postgres /usr/local/bin/pg-backup >> /var/log/pg-backup.log 2>&1
```

শর্ত:
- ফাইলের মালিক root হতে হবে, group/others লিখতে পারবে না (`sudo chmod 644 /etc/cron.d/db-backup`)
- ফাইলের নামে ডট নয় (Debian/Ubuntu)
- শেষে newline
- `/etc/cron.d/`-এর ফাইল executable হওয়ার দরকার নেই (এগুলো crontab, স্ক্রিপ্ট নয়)

### কোনটা কখন ব্যবহার করবে

| পরিস্থিতি | জায়গা |
|---|---|
| নিজের ব্যক্তিগত কাজ (ইউজার হিসেবে) | `crontab -e` |
| সিস্টেমের কাজ, নির্দিষ্ট সময় দরকার | `/etc/cron.d/<নাম>` |
| সিস্টেমের কাজ, "দিনে একবার" হলেই হলো | `/etc/cron.daily/<নাম>` (executable স্ক্রিপ্ট) |
| `/etc/crontab` | সাধারণত হাত দিও না |

### সব cron কাজ এক জায়গায় দেখা

```
$ sudo ls /var/spool/cron/crontabs/          # কোন কোন ইউজারের crontab আছে (Ubuntu/Debian)
nishat  www-data
$ sudo ls /var/spool/cron/                   # Fedora/RHEL
$ ls /etc/cron.d/ /etc/cron.daily/
$ for u in $(cut -d: -f1 /etc/passwd); do sudo crontab -u "$u" -l 2>/dev/null | sed "s/^/$u: /"; done
```

শেষ লাইনটা প্রতিটা ইউজারের crontab দেখায় — সার্ভার অডিটের সময় কাজের ([অধ্যায় ৪৫](45-security-hardening.md))। আক্রমণকারীরা প্রায়ই cron-এ লুকিয়ে কাজ বসায়!

---

## ২৯.৭ cron-এর environment ও PATH সমস্যা, আউটপুট ও লগ

### "টার্মিনালে চলে, cron-এ চলে না!" — এক নম্বর অভিযোগ

এটা লিনাক্সে সবচেয়ে বেশি জিজ্ঞাসা করা প্রশ্নগুলোর একটা। কারণ বোঝো:

```
   তোমার টার্মিনাল (interactive login shell)          cron-এর কাজ
   ───────────────────────────────────────           ─────────────────────
   ~/.bashrc, ~/.profile পড়া হয়েছে                  কিছুই পড়া হয় না
   PATH=/usr/local/sbin:/usr/local/bin:              PATH=/usr/bin:/bin
        /usr/sbin:/usr/bin:/sbin:/bin:                    (খুবই ছোট!)
        /snap/bin:/home/nishat/.local/bin
   SHELL=/bin/bash                                   SHELL=/bin/sh (dash)
   তোমার alias, function আছে                         কিছু নেই
   current directory = যেখানে আছো                    current directory = HOME
   DISPLAY, টার্মিনাল আছে                            কোনো টার্মিনাল নেই
   nvm/pyenv/conda সেট করা                           কিছুই সেট নেই
```

cron একদম ন্যূনতম environment দেয় — সাধারণত শুধু `SHELL=/bin/sh`, `PATH=/usr/bin:/bin`, `HOME`, `LOGNAME` (আর `USER`)। তাই `/usr/local/bin`, `/usr/sbin`, `~/.local/bin`, `/snap/bin`-এর কোনো কমান্ড পাওয়া যাবে না।

cron আসলে কী environment দেয় নিজের চোখে দেখো:

```
* * * * * env > /tmp/cron-env.txt
```

এক মিনিট পরে:

```
$ cat /tmp/cron-env.txt
HOME=/home/nishat
LOGNAME=nishat
PATH=/usr/bin:/bin
LANG=en_US.UTF-8
SHELL=/bin/sh
PWD=/home/nishat
```

(পরে লাইনটা মুছে দিও!)

### সমাধানগুলো

**১. কমান্ডের পুরো পাথ লেখো** (সবচেয়ে নিরাপদ):

```
$ which mysqldump
/usr/bin/mysqldump
```

```
0 2 * * * /usr/bin/mysqldump mydb > /backup/mydb.sql
```

**২. crontab-এর উপরে PATH সেট করো:**

```
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**৩. bash লাগলে SHELL বদলাও:**

```
SHELL=/bin/bash
```

**৪. স্ক্রিপ্টের ভেতরেই environment ঠিক করো:**

```bash
#!/bin/bash
export PATH=/usr/local/bin:/usr/bin:/bin
cd /home/nishat/myproject || exit 1
source /home/nishat/myproject/.env
./run-report
```

**৫. রিলেটিভ পাথ এড়াও** — cron-এ current directory হলো HOME, তাই `./data/file.txt` ভুল জায়গা খুঁজবে। সবসময় absolute path, অথবা স্ক্রিপ্টের শুরুতে `cd`।

**৬. GUI কমান্ড** (যেমন `notify-send`) cron থেকে চালাতে `DISPLAY` ও `DBUS_SESSION_BUS_ADDRESS` লাগে — ঝামেলার। ডেস্কটপ রিমাইন্ডারের জন্য systemd user timer সহজ।

### cron-এর আউটপুট কোথায় যায়?

cron কাজের stdout বা stderr-এ যা আসে, cron সেটা **ইমেইল** করে পাঠায় crontab-এর মালিকের কাছে (বা `MAILTO`-তে)। কিন্তু আধুনিক ডেস্কটপ/সার্ভারে সাধারণত মেইল সার্ভার (MTA — Mail Transfer Agent, যেমন postfix) থাকে না, তাই আউটপুট হারিয়ে যায়। Ubuntu/Debian-এ লগে দেখবে:

```
CRON[4521]: (CRON) info (No MTA installed, discarding output)
```

এই লাইনটা দেখলে বুঝবে — কাজ চলেছে, কিছু বলেছে (হয়তো এরর!), কিন্তু তুমি সেটা দেখতে পাওনি।

### আউটপুট নিজে লগ ফাইলে পাঠাও

রিডাইরেকশন ([অধ্যায় ১৯](19-redirection-pipe.md)) দিয়ে:

```
# stdout আর stderr দুটোই লগ ফাইলে যোগ (append) করো
0 2 * * * /usr/local/bin/backup >> /var/log/backup.log 2>&1

# শুধু এরর রাখো, সাধারণ আউটপুট ফেলে দাও
0 2 * * * /usr/local/bin/backup > /dev/null

# সব ফেলে দাও (মেইলও যাবে না) — সাবধানে, এরর লুকিয়ে যাবে
*/5 * * * * /usr/local/bin/ping-check > /dev/null 2>&1

# systemd journal-এ পাঠাও, ট্যাগ দিয়ে
0 2 * * * /usr/local/bin/backup 2>&1 | logger -t mybackup
```

> ⚠️ সতর্কতা: `2>&1 >> file` আর `>> file 2>&1` এক জিনিস না! ক্রম গুরুত্বপূর্ণ। সঠিক হলো `>> file 2>&1` — আগে stdout ফাইলে পাঠাও, তারপর stderr-কে stdout-এর জায়গায় পাঠাও।

শেষ উদাহরণের `logger` দিয়ে পাঠানো লগ দেখতে:

```
$ journalctl -t mybackup --since today
```

### MAILTO

```
MAILTO=admin@example.com      # এই ঠিকানায় আউটপুট মেইল করো (MTA লাগবে)
MAILTO=""                     # কোনো মেইল পাঠাবে না
```

### cron নিজে কী লগ করে

cron প্রতিটা কাজ শুরুর একটা লাইন লগ করে। কোথায় দেখবে:

| Distro | কমান্ড/ফাইল |
|---|---|
| Ubuntu 24.04 / Debian 12 | `journalctl -u cron` (আর rsyslog থাকলে `/var/log/syslog`) |
| Fedora / RHEL 9 | `journalctl -u crond` অথবা `/var/log/cron` |

```
$ journalctl -u cron --since "1 hour ago"
Nov 12 14:00:01 myhost CRON[5012]: pam_unix(cron:session): session opened for user nishat(uid=1000) by nishat(uid=0)
Nov 12 14:00:01 myhost CRON[5013]: (nishat) CMD (/home/nishat/bin/backup.sh)
Nov 12 14:00:02 myhost CRON[5012]: pam_unix(cron:session): session closed for user nishat
```

```
$ grep CRON /var/log/syslog | tail -5          # Ubuntu, rsyslog থাকলে
$ sudo tail -f /var/log/cron                   # RHEL/Fedora
```

> 📝 মনে রাখো: Debian 12-এ ডিফল্টে rsyslog ইনস্টল থাকে না, তাই `/var/log/syslog` নাও থাকতে পারে — journalctl ব্যবহার করো ([অধ্যায় ২৮](28-logs.md))।

### cron ডিবাগ করার চেকলিস্ট

```
কাজ চলছে না?
   │
   ├─ ১. cron daemon চলছে?            → systemctl status cron / crond
   ├─ ২. crontab ইনস্টল হয়েছে?         → crontab -l
   ├─ ৩. cron চালানোর চেষ্টা করেছে?    → journalctl -u cron  (CMD লাইন আছে?)
   ├─ ৪. সময়ের ফিল্ড ঠিক?              → crontab.guru-তে মিলিয়ে দেখো, টাইমজোন দেখো
   ├─ ৫. PATH/পুরো পাথ?                → কমান্ডের absolute path দাও
   ├─ ৬. % এস্কেপ করা?                 → \%
   ├─ ৭. স্ক্রিপ্ট executable?          → chmod +x, প্রথম লাইনে #!/bin/bash
   ├─ ৮. ফাইলের নামে ডট? (cron.d/daily) → নাম বদলাও
   ├─ ৯. আউটপুট/এরর কী বলছে?           → >> /tmp/debug.log 2>&1 যোগ করো
   └─ ১০. cron-এর মতো environment-এ টেস্ট করো:
          env -i HOME=$HOME PATH=/usr/bin:/bin /bin/sh -c '/path/to/script'
```

শেষ কমান্ডটা দারুণ কাজের: `env -i` সব environment মুছে ফেলে, তারপর শুধু cron-এর মতো ন্যূনতম ভ্যারিয়েবল দিয়ে স্ক্রিপ্ট চালায়। এখানে চললে cron-এও চলার সম্ভাবনা বেশি।

> 💡 টিপস: ৫ ফিল্ড মেলানো নিয়ে সন্দেহ হলে https://crontab.guru ওয়েবসাইটে লাইনটা বসিয়ে দেখো — সাধারণ ভাষায় বলে দেয় কখন চলবে।

---

## ২৯.৮ anacron — যে কম্পিউটার রাতে বন্ধ থাকে

### সমস্যা

cron ধরে নেয় কম্পিউটার ২৪ ঘণ্টা চালু। কিন্তু তোমার ল্যাপটপ রাতে বন্ধ থাকে। `/etc/cron.daily` যদি ভোর ৬:২৫-এ চলার কথা থাকে আর তুমি ল্যাপটপ চালু করো সকাল ৯টায় — cron সেই কাজ **কখনো চালাবে না**। মিস হওয়া মিনিট সে ফিরে দেখে না।

### সমাধান: anacron

**anacron** (নাম এসেছে "anachronistic" থেকে — সময়ের সাথে না মেলা) সময় না, **পর্যায়কাল (period)** দিয়ে ভাবে: "শেষবার কবে চলেছিল? এক দিনের বেশি হয়ে গেলে এখন চালাও।"

উপমা: cron হলো ট্রেনের সময়সূচি — ৯:০০-এর ট্রেন ৯:০০-এই ছাড়বে, তুমি না থাকলে মিস। anacron হলো ওষুধ খাওয়ার নিয়ম — "দিনে একবার"; সকালে ভুলে গেলে দুপুরে মনে পড়লেই খেয়ে নাও।

```
            cron                                anacron
  ┌─────────────────────────┐          ┌──────────────────────────────┐
  │ "রাত ২:০০ টায় চালাও"    │          │ "প্রতি ১ দিনে একবার চালাও"   │
  │                          │          │                              │
  │ ২:০০-তে PC বন্ধ? → মিস   │          │ চালু হলে চেক: শেষবার কবে?    │
  │ মিনিট-স্তরের নির্ভুলতা   │          │ /var/spool/anacron/cron.daily│
  │ daemon, সবসময় চলে        │          │ ≥১ দিন আগে? → delay পরে চালাও │
  │ যেকোনো ইউজার             │          │ দিন-স্তরের, daemon না         │
  └─────────────────────────┘          │ সাধারণত শুধু root             │
                                        └──────────────────────────────┘
```

anacron daemon না — এটা চালু হয়, কাজ চেক করে/চালায়, তারপর বেরিয়ে যায়। একে চালায় cron (RHEL-এ `/etc/cron.hourly/0anacron`), অথবা systemd (`anacron.timer`, Debian/Ubuntu-তে), অথবা বুটের সময়।

### `/etc/anacrontab` — ফরম্যাট

```
period   delay   job-identifier   command
(দিন)   (মিনিট)  (নাম)            (কী চালাবে)
```

| ফিল্ড | অর্থ |
|---|---|
| period | কত দিন পরপর; `@monthly` ও লেখা যায় |
| delay | চালানোর আগে কত মিনিট অপেক্ষা (বুটের সময় সব একসাথে চাপ না দিতে) |
| job-identifier | নাম; `/var/spool/anacron/<নাম>` ফাইলে শেষ চলার তারিখ রাখে |
| command | কমান্ড |

Ubuntu/Debian:

```
$ cat /etc/anacrontab
SHELL=/bin/sh
HOME=/root
LOGNAME=root

# These replace cron's entries
1	5	cron.daily	run-parts --report /etc/cron.daily
7	10	cron.weekly	run-parts --report /etc/cron.weekly
@monthly	15	cron.monthly	run-parts --report /etc/cron.monthly
```

RHEL 9 / Fedora:

```
$ cat /etc/anacrontab
SHELL=/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
# the maximal random delay added to the base delay of the jobs
RANDOM_DELAY=45
# the jobs will be started during the following hours only
START_HOURS_RANGE=3-22

#period in days   delay in minutes   job-identifier   command
1	5	cron.daily		nice run-parts /etc/cron.daily
7	25	cron.weekly		nice run-parts /etc/cron.weekly
@monthly 45	cron.monthly		nice run-parts /etc/cron.monthly
```

এখানে `START_HOURS_RANGE=3-22` মানে শুধু ভোর ৩টা থেকে রাত ১০টার মধ্যে কাজ শুরু হবে, আর `RANDOM_DELAY=45` মানে delay-এর সাথে ০–৪৫ মিনিটের এলোমেলো দেরি যোগ হবে।

টাইমস্ট্যাম্প ফাইল:

```
$ sudo cat /var/spool/anacron/cron.daily
20261112
```

### anacron কমান্ড

| কমান্ড | কাজ |
|---|---|
| `anacron -T` | anacrontab-এর সিনট্যাক্স পরীক্ষা |
| `anacron -n` | delay উপেক্ষা করে এখনই (দরকারি কাজ) চালাও |
| `anacron -f` | সময় না হলেও জোর করে সব চালাও (force) |
| `anacron -u` | কিছু না চালিয়ে শুধু টাইমস্ট্যাম্প আজকের তারিখে আপডেট করো |
| `anacron -d` | foreground-এ চালাও, মেসেজ টার্মিনালে দেখাও (debug) |
| `anacron -s` | কাজগুলো একটার পর একটা (serialize) চালাও |

```
$ sudo anacron -T && echo "সিনট্যাক্স ঠিক আছে"
সিনট্যাক্স ঠিক আছে
$ sudo anacron -fnd         # সব কাজ এখনই, foreground-এ
```

> 📝 মনে রাখো: anacron ইনস্টল না থাকলে (`test -x /usr/sbin/anacron` ব্যর্থ), Ubuntu/Debian-এর `/etc/crontab` নিজেই cron.daily ইত্যাদি চালায়। Ubuntu ডেস্কটপে anacron সাধারণত ইনস্টল থাকে; মিনিমাল সার্ভারে নাও থাকতে পারে: `sudo apt install anacron`। RHEL/Fedora-তে এটা `cronie-anacron` প্যাকেজে আসে।

---

## ২৯.৯ at, atq, atrm ও batch — একবারের কাজ

cron বারবারের জন্য। কিন্তু যদি বলো "আজ রাত ১১টায় একবার সার্ভার রিস্টার্ট করো" — তখন crontab-এ লাইন দিয়ে পরে মুছে ফেলা ঝামেলা। এর জন্য **at**।

### ইনস্টল ও চালু করা

`at` সব জায়গায় ডিফল্টে থাকে না:

```
$ sudo apt install at                          # Ubuntu/Debian (atd নিজে চালু হয়)
$ sudo dnf install at && sudo systemctl enable --now atd    # Fedora/RHEL
$ systemctl status atd
```

ভেতরে: `at` কমান্ড তোমার কাজটা (সাথে তোমার বর্তমান environment ও current directory!) একটা স্ক্রিপ্ট হিসেবে `/var/spool/cron/atjobs/` (Debian) বা `/var/spool/at/` (RHEL)-এ রেখে দেয়। `atd` daemon সময় হলে সেটা চালায়।

> 📝 মনে রাখো: cron-এর বিপরীতে, `at` তোমার বর্তমান environment ভ্যারিয়েবল (PATH সহ) আর current directory **সংরক্ষণ করে**। তাই at-এ PATH সমস্যা অনেক কম। তবে টার্মিনাল থাকে না — আউটপুট মেইলে যায় (MTA থাকলে)।

### at-এর ব্যবহার

```
$ at 23:00
warning: commands will be executed using /bin/sh
at> /usr/local/bin/backup
at> echo "done" >> /tmp/at-test.log
at> <EOT>                           ← এখানে Ctrl+D চাপো
job 3 at Thu Nov 12 23:00:00 2026
```

কমান্ড পাইপ বা ফাইল দিয়েও দেওয়া যায়:

```
$ echo "/usr/local/bin/backup" | at now + 30 minutes
job 4 at Thu Nov 12 15:42:00 2026

$ at -f /home/nishat/bin/task.sh 02:00 tomorrow
job 5 at Fri Nov 13 02:00:00 2026
```

### সময় লেখার ধরন

`at` খুব সহজ ইংরেজি সময় বোঝে:

| লেখা | অর্থ |
|---|---|
| `now + 5 minutes` | ৫ মিনিট পর |
| `now + 2 hours` | ২ ঘণ্টা পর |
| `now + 3 days` | ৩ দিন পর (এখনকার সময়ে) |
| `now + 1 week` | ১ সপ্তাহ পর |
| `14:30` | আজ ২:৩০-এ (পেরিয়ে গেলে কাল) |
| `2:30 PM` | একই, ১২ ঘণ্টার ফরম্যাটে |
| `noon` / `midnight` | দুপুর ১২টা / রাত ১২টা |
| `teatime` | বিকেল ৪টা |
| `tomorrow` | কাল এই সময়ে |
| `10:00 tomorrow` | কাল সকাল ১০টায় |
| `9am Monday` | পরের সোমবার সকাল ৯টায় |
| `10:00 2026-12-31` | নির্দিষ্ট তারিখে (ISO) |
| `10:00 12/31/2026` | নির্দিষ্ট তারিখে (MM/DD/YYYY) |
| `-t 202612311000` | `touch`-এর মতো `[[CC]YY]MMDDhhmm` ফরম্যাট |

পুরো তালিকা: `/usr/share/doc/at/timespec` (Debian/Ubuntu)।

### at-এর অপশন

| অপশন | কাজ |
|---|---|
| `-f file` | stdin-এর বদলে ফাইল থেকে কমান্ড |
| `-l` | কিউর কাজ দেখাও (`atq`-এর সমান) |
| `-d job` | কাজ মুছো (`atrm`-এর সমান) |
| `-c job` | কাজের পুরো স্ক্রিপ্ট দেখাও (environment সহ) |
| `-m` | আউটপুট না থাকলেও শেষ হলে মেইল করো |
| `-M` | কখনো মেইল করবে না |
| `-q queue` | কিউ বাছো (a–z, A–Z; বড় অক্ষর = বেশি nice) |
| `-t time` | `[[CC]YY]MMDDhhmm[.ss]` ফরম্যাটে সময় |

### atq — কিউ দেখা

```
$ atq
3	Thu Nov 12 23:00:00 2026 a nishat
4	Thu Nov 12 15:42:00 2026 a nishat
5	Fri Nov 13 02:00:00 2026 a nishat
```

কলাম: job নম্বর, সময়, কিউ (`a`), ইউজার। root সবার কাজ দেখে।

কাজের ভেতরে কী আছে:

```
$ at -c 4 | tail -3
cd /home/nishat || {
	 echo 'Execution directory inaccessible' >&2
	 exit 1
}
/usr/local/bin/backup
```

### atrm — কাজ বাতিল

```
$ atrm 4
$ atq
3	Thu Nov 12 23:00:00 2026 a nishat
5	Fri Nov 13 02:00:00 2026 a nishat
```

### batch — সিস্টেম ফ্রি হলে চালাও

`batch` কোনো সময় নেয় না। কাজটা কিউতে রাখে, আর যখন সিস্টেমের **load average** একটা সীমার নিচে নামে (atd-এর ডিফল্ট সাধারণত 1.5; `atd -l` দিয়ে বদলানো যায়), তখন চালায়। load average কী — [অধ্যায় ৪৬](46-performance-troubleshooting.md) দেখো।

```
$ batch
warning: commands will be executed using /bin/sh
at> tar czf /backup/big-archive.tar.gz /data
at> <EOT>
job 6 at Thu Nov 12 15:20:00 2026
$ atq
6	Thu Nov 12 15:20:00 2026 b nishat
```

লক্ষ্য করো কিউ `b` — batch-এর কিউ। ভারী কাজ (ভিডিও কনভার্ট, বড় কম্প্রেশন) যা জরুরি না, কিন্তু সিস্টেম ব্যস্ত থাকলে চালালে অন্যদের সমস্যা হবে — তার জন্য batch।

### একটা বাস্তব উদাহরণ: নিরাপদ ফায়ারওয়াল পরীক্ষা

রিমোট সার্ভারে ফায়ারওয়াল নিয়ম বদলাচ্ছ ([অধ্যায় ৩৯](39-firewall.md))? ভুল হলে নিজেকেই লক আউট করে ফেলবে। আগে একটা "সেফটি নেট" বসাও:

```
$ echo "ufw disable" | sudo at now + 10 minutes
job 7 at Thu Nov 12 15:30:00 2026
$ sudo ufw enable       # নতুন নিয়ম পরীক্ষা করো
# সব ঠিক থাকলে, সেফটি নেট বাতিল:
$ sudo atrm 7
```

ভুল হলে আর ঢুকতে না পারলেও ১০ মিনিট পর ফায়ারওয়াল নিজেই বন্ধ হবে।

### at-এর কীবোর্ড শর্টকাট

| শর্টকাট | কাজ |
|---|---|
| `Enter` | `at>` প্রম্পটে পরের কমান্ড লাইন |
| `Ctrl+D` | ইনপুট শেষ, কাজ কিউতে জমা (`<EOT>` দেখায়) |
| `Ctrl+C` | কিছু জমা না দিয়ে বাতিল |

---

## ২৯.১০ কে cron/at ব্যবহার করতে পারবে — allow/deny ফাইল

| ফাইল | নিয়ম |
|---|---|
| `/etc/cron.allow` | থাকলে **শুধু** এখানে লেখা ইউজাররা crontab ব্যবহার করতে পারবে |
| `/etc/cron.deny` | (allow না থাকলে) এখানে লেখা ইউজাররা পারবে না |
| `/etc/at.allow` | at-এর জন্য একই নিয়ম |
| `/etc/at.deny` | at-এর জন্য একই নিয়ম |

নিয়মের ক্রম:

```
cron.allow আছে?
   ├── হ্যাঁ → শুধু তালিকাভুক্ত ইউজার পারবে (deny দেখা হয় না)
   └── না → cron.deny আছে?
              ├── হ্যাঁ → তালিকাভুক্ত ছাড়া সবাই পারবে
              └── না → distro-ভেদে: Debian-এ সবাই পারে, কিছু সিস্টেমে শুধু root
```

ফাইলে প্রতি লাইনে একটা ইউজারনেম। root সবসময় পারে। ব্লক করা ইউজার চেষ্টা করলে:

```
$ crontab -e
You (guest) are not allowed to use this program (crontab)
See crontab(1) for more information
```

> 📝 মনে রাখো: এই ফাইলগুলো শুধু crontab/at **কমান্ড** ব্যবহার আটকায়। `/etc/cron.d`-এ root যা রাখে সেটা চলবেই।

### একই কাজ দুবার একসাথে না চলুক — `flock`

ধরো একটা কাজ প্রতি ৫ মিনিটে চলে, কিন্তু কোনো দিন সেটা ৭ মিনিট লাগল। তখন আগেরটা শেষ না হতেই নতুনটা শুরু হবে — দুটো একই ফাইলে লিখবে, ডেটা নষ্ট হবে। cron এটা আটকায় না। সমাধান `flock` (util-linux প্যাকেজে, প্রায় সব জায়গায় থাকে):

```
*/5 * * * * /usr/bin/flock -n /tmp/sync.lock /usr/local/bin/sync-data
```

`-n` (non-block) মানে: লক আগে থেকে নেওয়া থাকলে অপেক্ষা না করে সাথে সাথে বেরিয়ে যাও। (systemd timer-এ এই সমস্যা নেই — সার্ভিস আগে থেকে চললে timer নতুন করে শুরু করে না।)

---

## ২৯.১১ systemd timers — আধুনিক বিকল্প

### ধারণা

systemd ([অধ্যায় ২৬](26-systemd-services.md)) শুধু সার্ভিস চালায় না, শিডিউলও করতে পারে। systemd timer দুটো unit ফাইল দিয়ে তৈরি:

```
   ┌────────────────────┐   সময় হলে চালু করে   ┌────────────────────┐
   │  backup.timer      │ ────────────────────► │  backup.service    │
   │  "কখন"             │                        │  "কী"              │
   │  OnCalendar=...    │                        │  ExecStart=...     │
   │  Persistent=true   │                        │  Type=oneshot      │
   └────────────────────┘                        └────────────────────┘
            ▲                                              │
            │ systemctl enable --now backup.timer          ▼
       timers.target                              journal-এ সব আউটপুট
                                                  (journalctl -u backup)
```

উপমা: `.service` হলো রান্নার রেসিপি (কী রান্না হবে), `.timer` হলো রান্নাঘরের টাইমার (কখন চুলা জ্বলবে)। একই নামের হলে (`backup.timer` → `backup.service`) timer নিজে বুঝে নেয় কোন সার্ভিস চালাবে; আলাদা নাম হলে `Unit=` দিয়ে বলে দিতে হয়।

আসলে তোমার সিস্টেমে আগে থেকেই অনেক timer চলছে:

```
$ systemctl list-timers
NEXT                            LEFT LAST                              PASSED UNIT                         ACTIVATES
Thu 2026-11-12 16:00:00 +06    24min Thu 2026-11-12 15:00:04 +06      35min ago anacron.timer                anacron.service
Fri 2026-11-13 00:00:00 +06       8h Thu 2026-11-12 00:00:01 +06          15h ago logrotate.timer              logrotate.service
Fri 2026-11-13 00:00:00 +06       8h Thu 2026-11-12 00:00:01 +06          15h ago dpkg-db-backup.timer         dpkg-db-backup.service
Fri 2026-11-13 06:24:51 +06      14h Thu 2026-11-12 06:41:12 +06           8h ago apt-daily-upgrade.timer      apt-daily-upgrade.service
Sun 2026-11-15 03:10:52 +06   2 days Sun 2026-11-08 03:10:07 +06       4 days ago e2scrub_all.timer            e2scrub_all.service
Mon 2026-11-16 00:47:21 +06   3 days Mon 2026-11-09 00:15:33 +06       3 days ago fstrim.timer                 fstrim.service

6 timers listed.
Pass --all to see loaded but inactive timers, too.
```

(তোমার আউটপুট ভিন্ন হবে।) `systemctl list-timers --all` নিষ্ক্রিয়গুলোও দেখায়।

### সম্পূর্ণ উদাহরণ: প্রতিদিন রাত ২টার ব্যাকআপ

**ধাপ ১: স্ক্রিপ্ট**

```
$ sudo nano /usr/local/bin/mybackup
```

```bash
#!/bin/bash
set -euo pipefail
tar czf "/backup/etc-$(date +%F).tar.gz" /etc
echo "Backup finished at $(date)"
```

```
$ sudo chmod +x /usr/local/bin/mybackup
$ sudo mkdir -p /backup
```

(লক্ষ্য করো — এখানে `%` এস্কেপ করতে হয়নি, কারণ এটা crontab না!)

**ধাপ ২: service unit**

```
$ sudo nano /etc/systemd/system/mybackup.service
```

```ini
[Unit]
Description=Daily backup of /etc

[Service]
Type=oneshot
ExecStart=/usr/local/bin/mybackup
```

`Type=oneshot` মানে একবার চলে শেষ হয়ে যায় (দীর্ঘক্ষণ চলা সার্ভিস না)। এখানে `[Install]` সেকশন লাগে না, কারণ সার্ভিসটা timer চালু করবে, বুটে নয়।

**ধাপ ৩: timer unit**

```
$ sudo nano /etc/systemd/system/mybackup.timer
```

```ini
[Unit]
Description=Run mybackup daily at 02:00

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

**ধাপ ৪: চালু করো**

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now mybackup.timer
Created symlink /etc/systemd/system/timers.target.wants/mybackup.timer → /etc/systemd/system/mybackup.timer.
```

> ⚠️ সতর্কতা: enable করবে **`.timer`**, `.service` না! service enable করলে (যদি `[Install]` থাকত) সেটা বুটে চলত, শিডিউলে না।

**ধাপ ৫: যাচাই**

```
$ systemctl list-timers mybackup.timer
NEXT                         LEFT LAST PASSED UNIT           ACTIVATES
Fri 2026-11-13 02:04:37 +06   10h -    -      mybackup.timer mybackup.service

1 timers listed.
```

এখনই একবার পরীক্ষা করতে (timer-এর অপেক্ষা না করে):

```
$ sudo systemctl start mybackup.service
$ journalctl -u mybackup.service -n 5
Nov 12 15:40:11 myhost systemd[1]: Starting mybackup.service - Daily backup of /etc...
Nov 12 15:40:12 myhost mybackup[6120]: Backup finished at Thu Nov 12 15:40:12 +06 2026
Nov 12 15:40:12 myhost systemd[1]: mybackup.service: Deactivated successfully.
Nov 12 15:40:12 myhost systemd[1]: Finished mybackup.service - Daily backup of /etc.
```

দেখো — কোনো মেইল সার্ভার ছাড়াই আউটপুট সুন্দরভাবে journal-এ! এটাই timer-এর বড় সুবিধা।

### `OnCalendar` সিনট্যাক্স

সাধারণ ফরম্যাট:

```
   DayOfWeek  Year-Month-Day  Hour:Minute:Second  [Timezone]
   Mon..Fri   *-*-*           09:00:00            Asia/Dhaka
   ────────   ──────────────  ─────────────────
   (ঐচ্ছিক)   * = যেকোনো      সেকেন্ড বাদ দিলে :00
```

| OnCalendar | অর্থ | cron সমতুল্য |
|---|---|---|
| `minutely` | প্রতি মিনিটে | `* * * * *` |
| `hourly` | প্রতি ঘণ্টার শুরুতে | `0 * * * *` |
| `daily` | প্রতিদিন মধ্যরাতে | `0 0 * * *` |
| `weekly` | সোমবার মধ্যরাতে (রবিবার না!) | `0 0 * * 1` |
| `monthly` | মাসের ১ তারিখ মধ্যরাতে | `0 0 1 * *` |
| `yearly` / `annually` | ১ জানুয়ারি মধ্যরাতে | `0 0 1 1 *` |
| `quarterly` | জানু/এপ্রিল/জুলাই/অক্টো ১ তারিখে | `0 0 1 1,4,7,10 *` |
| `*-*-* 02:00:00` | প্রতিদিন রাত ২টা | `0 2 * * *` |
| `*:0/15` | প্রতি ১৫ মিনিটে | `*/15 * * * *` |
| `*:*:0/30` | প্রতি ৩০ সেকেন্ডে | (cron পারে না) |
| `Mon..Fri 09:00` | কর্মদিবসে সকাল ৯টায় | `0 9 * * 1-5` |
| `Sat,Sun 10:00` | সাপ্তাহিক ছুটিতে ১০টায় | `0 10 * * 0,6` |
| `*-*-01 03:00` | প্রতি মাসের ১ তারিখ রাত ৩টায় | `0 3 1 * *` |
| `*-*~01` | প্রতি মাসের **শেষ** দিনে (`~` = শেষ থেকে গোনা) | (cron পারে না) |
| `Fri *-*~07/1 18:00` | মাসের শেষ শুক্রবার সন্ধ্যা ৬টায় | (cron পারে না) |
| `*-*-* 02:00 Asia/Dhaka` | ঢাকার সময়ে রাত ২টা | — |

ধাপ (`/`) আর পরিসর (`..`) ব্যবহার হয়; তালিকা কমা দিয়ে। একাধিক `OnCalendar=` লাইন দেওয়া যায় — যেকোনোটা মিললেই চলবে।

### `systemd-analyze calendar` — লেখার আগে যাচাই

এটা অসাধারণ একটা টুল — বলে দেয় তোমার লেখা এক্সপ্রেশন কখন চলবে:

```
$ systemd-analyze calendar "Mon..Fri 09:00"
  Original form: Mon..Fri 09:00
Normalized form: Mon..Fri *-*-* 09:00:00
    Next elapse: Fri 2026-11-13 09:00:00 +06
       (in UTC): Fri 2026-11-13 03:00:00 UTC
       From now: 17h left
```

পরের কয়েকবার দেখতে:

```
$ systemd-analyze calendar --iterations=3 "*-*~01"
  Original form: *-*~01
Normalized form: *-*~01 00:00:00
    Next elapse: Mon 2026-11-30 00:00:00 +06
       (in UTC): Sun 2026-11-29 18:00:00 UTC
       From now: 2 weeks 3 days left
       Iter. #2: Thu 2026-12-31 00:00:00 +06
       (in UTC): Wed 2026-12-30 18:00:00 UTC
       From now: 1 month 18 days left
       Iter. #3: Sun 2027-01-31 00:00:00 +06
       (in UTC): Sat 2027-01-30 18:00:00 UTC
       From now: 2 months 18 days left
```

### `[Timer]` সেকশনের গুরুত্বপূর্ণ অপশন

| অপশন | কাজ | উদাহরণ |
|---|---|---|
| `OnCalendar=` | ক্যালেন্ডার/ঘড়ির সময় (wall-clock timer) | `OnCalendar=daily` |
| `Persistent=true` | মেশিন বন্ধ থাকায় মিস হলে, চালু হওয়ার পর একবার চালাও (anacron-এর মতো) — শুধু `OnCalendar`-এর সাথে কাজ করে | `Persistent=true` |
| `OnBootSec=` | বুটের এতক্ষণ পরে | `OnBootSec=5min` |
| `OnStartupSec=` | systemd (বা user manager) চালুর এতক্ষণ পরে | `OnStartupSec=10min` |
| `OnActiveSec=` | timer নিজে চালু হওয়ার এতক্ষণ পরে | `OnActiveSec=1h` |
| `OnUnitActiveSec=` | সার্ভিস শেষবার **চালু** হওয়ার এতক্ষণ পরে (পুনরাবৃত্তি) | `OnUnitActiveSec=15min` |
| `OnUnitInactiveSec=` | সার্ভিস শেষবার **শেষ** হওয়ার এতক্ষণ পরে | `OnUnitInactiveSec=30min` |
| `RandomizedDelaySec=` | ০ থেকে এই সময়ের মধ্যে এলোমেলো দেরি (অনেক সার্ভার একসাথে চাপ না দিতে) | `RandomizedDelaySec=15min` |
| `AccuracySec=` | কতটা নির্ভুল হবে; ডিফল্ট 1min (systemd বিদ্যুৎ বাঁচাতে কাছাকাছি timer একসাথে চালায়) | `AccuracySec=1s` |
| `Unit=` | কোন unit চালাবে (ডিফল্ট: একই নামের .service) | `Unit=other.service` |
| `WakeSystem=true` | suspend থেকে মেশিন জাগিয়ে চালাও (হার্ডওয়্যার সমর্থন লাগে) | `WakeSystem=true` |

`OnBootSec` আর `OnUnitActiveSec` মিলিয়ে "বুটের ৫ মিনিট পর একবার, তারপর প্রতি ১ ঘণ্টায়" — এমন **monotonic timer** (ঘড়ির সময় না, আপেক্ষিক সময়) বানানো যায়:

```ini
[Timer]
OnBootSec=5min
OnUnitActiveSec=1h
```

সময়ের একক: `us`, `ms`, `s`/`sec`, `m`/`min`, `h`/`hr`, `d`/`day`, `w`/`week`, `month`, `y`/`year` — যেমন `1h 30min`।

### `Persistent=true` — ভেতরে কীভাবে কাজ করে

systemd প্রতিবার timer চলার সময় একটা স্ট্যাম্প ফাইল আপডেট করে: `/var/lib/systemd/timers/stamp-<নাম>.timer`। বুটের পর timer চালু হলে সে দেখে — "শেষবার চলা আর এখনকার মধ্যে কি কোনো নির্ধারিত সময় পার হয়ে গেছে?" হ্যাঁ হলে সাথে সাথে একবার চালায়। এক সপ্তাহ বন্ধ থাকলেও একবারই চালায়, সাতবার না।

```
$ ls /var/lib/systemd/timers/
stamp-apt-daily.timer  stamp-fstrim.timer  stamp-logrotate.timer  stamp-mybackup.timer
```

### systemd-run — এক লাইনে অস্থায়ী timer (at-এর বিকল্প)

unit ফাইল না লিখেই:

```
$ sudo systemd-run --on-active=30min /usr/local/bin/mybackup
Running timer as unit: run-r8c1b2e3f.timer
Will run service as unit: run-r8c1b2e3f.service

$ sudo systemd-run --on-calendar="*-*-* 23:00" --unit=nightly-test /usr/bin/touch /tmp/hello
Running timer as unit: nightly-test.timer
Will run service as unit: nightly-test.service
```

এগুলো অস্থায়ী (transient) — রিবুটে হারিয়ে যায়। বাতিল করতে: `sudo systemctl stop run-r8c1b2e3f.timer`।

### ইউজার timer — root ছাড়া

নিজের ইউজারের জন্য timer রাখো `~/.config/systemd/user/`-এ, আর সব কমান্ডে `--user`:

```
$ mkdir -p ~/.config/systemd/user
$ nano ~/.config/systemd/user/remind.service
```

```ini
[Unit]
Description=Drink water reminder

[Service]
Type=oneshot
ExecStart=/usr/bin/notify-send "পানি খাও!"
```

```
$ nano ~/.config/systemd/user/remind.timer
```

```ini
[Unit]
Description=Every hour reminder

[Timer]
OnCalendar=hourly

[Install]
WantedBy=timers.target
```

```
$ systemctl --user daemon-reload
$ systemctl --user enable --now remind.timer
$ systemctl --user list-timers
```

ডেস্কটপ সেশনের ভেতরে চলে বলে `notify-send` এখানে কাজ করে (cron-এ যা ঝামেলার)।

> 📝 মনে রাখো: ইউজার timer শুধু তুমি লগইন থাকা অবস্থায় চলে। লগআউট থাকলেও (যেমন সার্ভারে) চালাতে চাইলে: `sudo loginctl enable-linger nishat`।

### timer ম্যানেজমেন্ট কমান্ড

| কমান্ড | কাজ |
|---|---|
| `systemctl list-timers` | সক্রিয় timer, পরের/শেষ চলার সময় |
| `systemctl list-timers --all` | নিষ্ক্রিয়গুলোও |
| `systemctl status mybackup.timer` | timer-এর অবস্থা, Trigger সময় |
| `systemctl enable --now X.timer` | চালু ও বুটে সক্রিয় |
| `systemctl disable --now X.timer` | বন্ধ ও নিষ্ক্রিয় |
| `systemctl start X.service` | এখনই একবার কাজটা চালাও |
| `systemctl cat X.timer` | unit ফাইল দেখাও |
| `systemctl edit X.timer` | override (drop-in) বানিয়ে বদলাও |
| `journalctl -u X.service` | কাজের আউটপুট/লগ |
| `systemd-analyze calendar "..."` | OnCalendar যাচাই |
| `systemd-analyze verify X.timer` | unit ফাইলে ভুল আছে কিনা দেখো |
| `systemctl daemon-reload` | unit ফাইল বদলানোর পর অবশ্যই |

---

## ২৯.১২ cron বনাম systemd timer — তুলনা

| বিষয় | cron | systemd timer |
|---|---|---|
| সেটআপ | এক লাইন — খুব সহজ | দুটো ফাইল — একটু বেশি লেখা |
| সর্বনিম্ন একক | ১ মিনিট | মাইক্রোসেকেন্ড পর্যন্ত (ডিফল্ট নির্ভুলতা ১ মিনিট) |
| লগিং | মেইল (MTA লাগে) বা নিজে রিডাইরেক্ট | স্বয়ংক্রিয় journal-এ |
| মিস হওয়া কাজ | না (anacron লাগে) | `Persistent=true` |
| বুটের পরে আপেক্ষিক সময় | শুধু `@reboot` | `OnBootSec`, `OnUnitActiveSec` |
| একই কাজ দুবার একসাথে | আটকায় না (`flock` লাগে) | আটকায় (চলমান থাকলে আবার শুরু হয় না) |
| নির্ভরতা (যেমন নেটওয়ার্ক) | নেই | `After=`, `Requires=` |
| রিসোর্স লিমিট | নেই (`nice` দিয়ে কিছুটা) | `CPUQuota=`, `MemoryMax=`, `Nice=`, `IOSchedulingClass=` |
| নিরাপত্তা স্যান্ডবক্স | নেই | `ProtectSystem=`, `PrivateTmp=` ইত্যাদি |
| এখনই হাতে চালানো | কমান্ড কপি করে চালাও (environment ভিন্ন!) | `systemctl start X.service` (হুবহু একই environment) |
| পরের রান কখন | নিজে হিসাব করো | `systemctl list-timers` |
| এলোমেলো দেরি | নেই (cronie-র anacron-এ `RANDOM_DELAY`) | `RandomizedDelaySec=` |
| টাইমজোন প্রতি কাজে | cronie-তে `CRON_TZ` | `OnCalendar=... Asia/Dhaka` |
| পোর্টেবিলিটি | প্রায় সব Unix (BSD, macOS, পুরোনো লিনাক্স) | শুধু systemd-ভিত্তিক লিনাক্স |
| নতুনদের জন্য | সহজ | একটু শেখা লাগে |

### সিদ্ধান্ত নেওয়ার সহজ নিয়ম

```
একবারের কাজ?
   ├── হ্যাঁ → at  (বা systemd-run --on-active)
   └── না (বারবার)
         │
         ├── দ্রুত, সাধারণ, ব্যক্তিগত কাজ / পুরোনো বা non-systemd সিস্টেম?
         │      → cron
         │
         └── প্রোডাকশন সার্ভার, ভালো লগ চাই, মিস হলে চালাতে হবে,
             নির্ভরতা/রিসোর্স লিমিট দরকার, সেকেন্ড-স্তরের সময়?
                → systemd timer
```

> 💡 টিপস: দুটোই শেখো। ইন্টারভিউ আর পুরোনো সার্ভারে cron সর্বত্র; আধুনিক distro-র নিজের কাজগুলো (logrotate, apt-daily, fstrim) বেশিরভাগ এখন timer দিয়ে চলে।

### এক নজরে: একই কাজ তিনভাবে

"প্রতিদিন রাত ২টায় `/usr/local/bin/mybackup` চালাও":

```
# ১. ইউজার crontab (crontab -e)
0 2 * * * /usr/local/bin/mybackup >> /var/log/mybackup.log 2>&1

# ২. /etc/cron.d/mybackup  (ইউজার ফিল্ড সহ)
0 2 * * * root /usr/local/bin/mybackup >> /var/log/mybackup.log 2>&1

# ৩. systemd timer  (mybackup.timer)
[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
```

আর "শুধু আজ রাত ২টায় একবার":

```
$ echo /usr/local/bin/mybackup | at 02:00
```

---

## ২৯.১৩ কাজের কীবোর্ড শর্টকাট

শিডিউলিং-এর নিজস্ব শর্টকাট কম, কিন্তু এডিটর আর প্রম্পটের শর্টকাট লাগবেই:

| পরিস্থিতি | শর্টকাট | কাজ |
|---|---|---|
| `crontab -e` (nano) | `Ctrl+O`, তারপর `Enter` | সেভ |
| `crontab -e` (nano) | `Ctrl+X` | বের হও (সেভ করতে বলবে) |
| `crontab -e` (nano) | `Ctrl+K` | পুরো লাইন কাটো |
| `crontab -e` (nano) | `Alt+3` | লাইন কমেন্ট/আনকমেন্ট (নতুন nano-তে) |
| `crontab -e` (vim) | `i` → লেখো → `Esc` → `:wq` | লেখো ও সেভ করে বের হও |
| `crontab -e` (vim) | `:q!` | সেভ না করে বের হও (কিছুই বদলাবে না) |
| `crontab -e` (vim) | `yy` `p` | লাইন কপি ও পেস্ট |
| `at>` প্রম্পট | `Ctrl+D` | কাজ জমা দাও |
| `at>` প্রম্পট | `Ctrl+C` | বাতিল |
| `journalctl -u ...` | `Shift+G` / `g` | শেষে / শুরুতে যাও |
| `journalctl -f` / `tail -f` | `Ctrl+C` | লাইভ দেখা বন্ধ |
| `systemctl list-timers` | `q` | pager থেকে বের হও |

বিস্তারিত এডিটর শর্টকাট: [অধ্যায় ১৩](13-nano-editor.md) ও [অধ্যায় ১৪](14-vim.md)।

---

## হাতে-কলমে অনুশীলন

### ল্যাব ১: প্রথম cron কাজ

১. প্রতি মিনিটে সময় লিখে রাখা একটা কাজ বানাও:

```
$ crontab -e
```

যোগ করো:

```
* * * * * /usr/bin/date >> /tmp/cron-test.log 2>&1
```

২. সেভ করে বের হও, তারপর যাচাই:

```
$ crontab -l
* * * * * /usr/bin/date >> /tmp/cron-test.log 2>&1
```

৩. ২–৩ মিনিট অপেক্ষা করে দেখো:

```
$ cat /tmp/cron-test.log
Thu Nov 12 15:51:01 +06 2026
Thu Nov 12 15:52:01 +06 2026
Thu Nov 12 15:53:01 +06 2026
```

৪. লগে cron কী বলছে দেখো: `journalctl -u cron -n 5` (RHEL-এ `crond`)।

৫. এবার `*/2` করে দাও — প্রতি ২ মিনিটে চলছে কিনা দেখো।

### ল্যাব ২: `%` আর environment-এর ফাঁদ নিজের চোখে দেখো

১. crontab-এ যোগ করো (ইচ্ছা করে ভুল):

```
* * * * * date +%H:%M >> /tmp/pct.log
```

২. এক মিনিট পর `cat /tmp/pct.log` — আশা অনুযায়ী আউটপুট আসবে না (কমান্ডটা `%`-এ ভেঙে গেছে)। এবার ঠিক করো:

```
* * * * * date +\%H:\%M >> /tmp/pct.log
```

৩. environment দেখো:

```
* * * * * env > /tmp/cron-env.txt
```

তারপর `diff <(sort /tmp/cron-env.txt) <(env | sort)` দিয়ে তোমার টার্মিনালের সাথে তুলনা করো। PATH-এর পার্থক্য লক্ষ্য করো।

৪. সব পরীক্ষার লাইন মুছে দাও (`crontab -e` দিয়ে, `-r` দিয়ে না!)।

### ল্যাব ৩: crontab ব্যাকআপ ও রিস্টোর

```
$ crontab -l > ~/my-crontab.bak
$ crontab -ri
crontab: really delete nishat's crontab? (y/n) y
$ crontab -l
no crontab for nishat
$ crontab ~/my-crontab.bak
$ crontab -l                     # আগের সব ফিরে এসেছে
```

### ল্যাব ৪: `/etc/cron.daily`-তে স্ক্রিপ্ট (ডট-এর ফাঁদ)

```
$ printf '#!/bin/sh\necho "daily ran $(date)" >> /tmp/daily.log\n' | sudo tee /etc/cron.daily/mytest.sh
$ sudo chmod +x /etc/cron.daily/mytest.sh
$ run-parts --test /etc/cron.daily | grep mytest     # Ubuntu/Debian: কিছুই দেখাবে না!
$ sudo mv /etc/cron.daily/mytest.sh /etc/cron.daily/mytest
$ run-parts --test /etc/cron.daily | grep mytest
/etc/cron.daily/mytest
$ sudo run-parts --regex '^mytest$' /etc/cron.daily  # শুধু এটাকে এখনই চালাও (Debian)
$ cat /tmp/daily.log
$ sudo rm /etc/cron.daily/mytest                     # পরিষ্কার করো
```

### ল্যাব ৫: at দিয়ে একবারের কাজ

```
$ echo 'echo "at worked at $(date)" > /tmp/at.log' | at now + 2 minutes
$ atq
$ at -c <job-নম্বর> | tail -3
# আরেকটা কাজ বানিয়ে বাতিল করো
$ echo 'touch /tmp/should-not-exist' | at now + 1 hour
$ atq
$ atrm <job-নম্বর>
$ atq
```

২ মিনিট পর `cat /tmp/at.log` দেখো।

### ল্যাব ৬: systemd timer বানাও

২৯.১১-এর `mybackup` উদাহরণটা হুবহু তৈরি করো, কিন্তু `OnCalendar=*:0/2` দাও (প্রতি ২ মিনিটে, শুধু পরীক্ষার জন্য)। তারপর:

```
$ systemd-analyze calendar "*:0/2"
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now mybackup.timer
$ systemctl list-timers mybackup.timer
$ journalctl -u mybackup.service -f        # Ctrl+C দিয়ে বের হও
$ ls /backup
```

শেষে পরিষ্কার করো:

```
$ sudo systemctl disable --now mybackup.timer
$ sudo rm /etc/systemd/system/mybackup.{timer,service}
$ sudo systemctl daemon-reload
```

### ল্যাব ৭: `Persistent=true` পরীক্ষা (VM-এ করো)

১. একটা timer বানাও `OnCalendar` দিয়ে এমন সময়ে যা কয়েক মিনিট পরে, আর `Persistent=true`।
২. সেই সময়ের আগেই VM বন্ধ করো, সময় পার হলে চালু করো।
৩. `journalctl -u <নাম>.service -b` দেখো — বুটের পরপরই কাজটা চলেছে।
৪. এবার `Persistent=false` দিয়ে একই পরীক্ষা করো, পার্থক্য দেখো।

### ল্যাব ৮: চ্যালেঞ্জ

নিচের প্রতিটার জন্য cron লাইন **এবং** OnCalendar লেখো, তারপর `systemd-analyze calendar` দিয়ে যাচাই করো:

১. প্রতি কর্মদিবসে (সোম–শুক্র) বিকেল ৫:৩০
২. প্রতি ঘণ্টার ১৫ ও ৪৫ মিনিটে
৩. প্রতি মাসের ১০ তারিখ সকাল ৮টা
৪. প্রতি ১০ মিনিটে, শুধু রাত ১০টা থেকে রাত ১১:৫০ পর্যন্ত

(উত্তর: ১. `30 17 * * 1-5` / `Mon..Fri 17:30`; ২. `15,45 * * * *` / `*:15,45`; ৩. `0 8 10 * *` / `*-*-10 08:00`; ৪. `*/10 22-23 * * *` / `22,23:0/10`)

---

## সারসংক্ষেপ

| কমান্ড / জিনিস | কাজ |
|---|---|
| `crontab -e` | নিজের crontab এডিট |
| `crontab -l` | crontab দেখো |
| `crontab -r` / `-ri` | crontab মুছো (`-i` = নিশ্চিত হয়ে) |
| `crontab -u user -e` | অন্য ইউজারের crontab (root) |
| `crontab file` | ফাইল থেকে crontab বসাও (পুরোনোটা বদলে যায়) |
| `মি ঘ দি মা স কমান্ড` | ৫ ফিল্ড: 0-59, 0-23, 1-31, 1-12, 0-7 |
| `*` `,` `-` `/` | সব, তালিকা, পরিসর, ধাপ |
| `@reboot @hourly @daily @weekly @monthly @yearly` | বিশেষ স্ট্রিং |
| `\%` | crontab-এ `%` এস্কেপ |
| `>> log 2>&1` | আউটপুট ও এরর লগে |
| `/etc/crontab`, `/etc/cron.d/` | সিস্টেম crontab (ইউজার ফিল্ড সহ) |
| `/etc/cron.{hourly,daily,weekly,monthly}/` | executable স্ক্রিপ্ট রাখো, নামে ডট নয় |
| `run-parts --test DIR` | কোন স্ক্রিপ্ট চলবে দেখো |
| `/etc/cron.allow`, `/etc/cron.deny` | কে crontab পারবে |
| `journalctl -u cron` / `crond` | cron-এর লগ |
| `/etc/anacrontab` | period delay id command |
| `anacron -T`, `anacron -fn` | সিনট্যাক্স চেক, জোর করে এখনই |
| `at TIME`, `at -f file TIME` | একবারের কাজ |
| `atq` / `at -l` | at কিউ |
| `at -c N` | কাজের বিষয়বস্তু |
| `atrm N` / `at -d N` | কাজ বাতিল |
| `batch` | লোড কম হলে চালাও |
| `flock -n lockfile cmd` | একসাথে দুবার চলা আটকাও |
| `X.timer` + `X.service` | systemd timer জোড়া |
| `OnCalendar=`, `Persistent=true` | ক্যালেন্ডার সময়, মিস হলে পরে চালাও |
| `OnBootSec=`, `OnUnitActiveSec=` | আপেক্ষিক (monotonic) সময় |
| `RandomizedDelaySec=`, `AccuracySec=` | এলোমেলো দেরি, নির্ভুলতা |
| `systemctl list-timers [--all]` | সব timer |
| `systemctl enable --now X.timer` | timer চালু |
| `systemd-analyze calendar "..."` | OnCalendar যাচাই |
| `systemd-run --on-active=30min cmd` | অস্থায়ী timer |
| `systemctl --user ...`, `loginctl enable-linger` | ইউজার timer |

---

## নিজেকে যাচাই করো

১. `*/20 8-18 * * 1-5` লাইনটা কখন চলবে?

২. ইউজার crontab আর `/etc/cron.d/`-এর ফাইলের ফরম্যাটে মূল পার্থক্য কী?

৩. একটা স্ক্রিপ্ট টার্মিনালে ঠিকঠাক চলে, কিন্তু cron-এ "command not found" দেয়। সম্ভাব্য কারণ ও দুটো সমাধান বলো।

৪. `0 2 * * * tar czf /backup/$(date +%F).tar.gz /home` লাইনে সমস্যা কী?

৫. Ubuntu-তে `/etc/cron.daily/cleanup.sh` নামে executable স্ক্রিপ্ট রাখলে, সেটা কি চলবে? কেন?

৬. `0 0 1 * 1` লাইনটা কখন চলবে — "মাসের ১ তারিখ যদি সোমবার হয়"?

৭. cron আর anacron-এর মূল পার্থক্য কী? ল্যাপটপে কোনটা দরকার?

৮. `at` দিয়ে ৪৫ মিনিট পরে একবার `/usr/local/bin/report` চালানোর কমান্ড লেখো, আর কিউ দেখা ও বাতিল করার কমান্ড বলো।

৯. systemd timer-এ `Persistent=true` কী করে? এটা কোন ধরনের timer-এর সাথে কাজ করে?

১০. systemd timer বানানোর পর কোন unit enable করতে হয়, আর timer-এর পরের রান কখন সেটা কীভাবে দেখবে?

<details>
<summary>উত্তর দেখো</summary>

১. সোমবার থেকে শুক্রবার, সকাল ৮টা থেকে সন্ধ্যা ৬টার ঘণ্টাগুলোর (৮–১৮) প্রতিটিতে ০, ২০ ও ৪০ মিনিটে। অর্থাৎ শেষ রান ১৮:৪০-এ।

২. `/etc/cron.d/` (এবং `/etc/crontab`)-এর ফাইলে ৫টি সময়-ফিল্ডের পরে একটা অতিরিক্ত **ইউজার** ফিল্ড থাকে — কাজটা কোন ইউজার হিসেবে চলবে। ইউজার crontab-এ এই ফিল্ড নেই, কারণ মালিকের নামেই চলে। এছাড়া `/etc/cron.d/` ফাইল root-এর মালিকানায় ও নামে ডট ছাড়া হতে হয়।

৩. cron ন্যূনতম environment দেয় — `PATH` সাধারণত শুধু `/usr/bin:/bin`, `.bashrc`/`.profile` পড়ে না, shell `/bin/sh`। তাই `/usr/local/bin`, `~/.local/bin` বা `/usr/sbin`-এর কমান্ড পায় না। সমাধান: (ক) কমান্ডের পুরো পাথ লেখো (`which` দিয়ে খুঁজে), (খ) crontab-এর উপরে `PATH=...` সেট করো, (গ) স্ক্রিপ্টের ভেতরে `export PATH=...` করো।

৪. crontab-এ `%` মানে নতুন লাইন; প্রথম `%`-এর পরের অংশ stdin হয়ে যায়, ফলে কমান্ড ভেঙে যায়। ঠিক করতে `date +\%F` লেখো, অথবা কমান্ডটা স্ক্রিপ্টে রাখো।

৫. না। Debian/Ubuntu-র `run-parts` ফাইলের নামে ডট (`.`) থাকলে সেটা উপেক্ষা করে। নাম `cleanup` রাখতে হবে। `run-parts --test /etc/cron.daily` দিয়ে যাচাই করা যায়।

৬. না। মাসের দিন আর সপ্তাহের দিন দুটোই নির্দিষ্ট হলে cron OR করে — এটা প্রতি মাসের ১ তারিখে **এবং** প্রতি সোমবার মধ্যরাতে চলবে। "দুটোই" শর্ত চাইলে কমান্ডের ভেতরে `date +\%u` দিয়ে চেক করতে হয় (বা systemd-এ `Mon *-*-01 00:00`)।

৭. cron নির্দিষ্ট সময়ে চালায়, মেশিন তখন বন্ধ থাকলে কাজ মিস হয়; মিনিট-স্তরের, daemon হিসেবে সবসময় চলে। anacron পর্যায়কাল (দিন) দিয়ে ভাবে — শেষবার কবে চলেছে মনে রাখে (`/var/spool/anacron/`), মিস হলে চালু হওয়ার পরে চালায়। ল্যাপটপ/ডেস্কটপে anacron (বা `Persistent=true` timer) দরকার, যাতে দৈনিক/সাপ্তাহিক কাজ বাদ না পড়ে।

৮. `echo /usr/local/bin/report | at now + 45 minutes` (বা `at now + 45 minutes` লিখে প্রম্পটে কমান্ড দিয়ে `Ctrl+D`)। কিউ দেখতে `atq` (বা `at -l`), বাতিল করতে `atrm <job-নম্বর>` (বা `at -d`)।

৯. মেশিন বন্ধ (বা timer নিষ্ক্রিয়) থাকার কারণে কোনো নির্ধারিত সময় মিস হলে, পরের বার timer চালু হওয়ার সাথে সাথে কাজটা একবার চালায়। শেষ চলার সময় `/var/lib/systemd/timers/stamp-*.timer`-এ রাখা হয়। এটা শুধু `OnCalendar=` (realtime/wall-clock) timer-এর সাথে কাজ করে।

১০. `.timer` unit enable করতে হয় (`sudo systemctl enable --now X.timer`), `.service` না। পরের রান দেখতে `systemctl list-timers` (বা `systemctl status X.timer`); আর লেখার আগে এক্সপ্রেশন যাচাই করতে `systemd-analyze calendar "..."`।

</details>

---

[← আগের অধ্যায়](28-logs.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](30-package-management.md)

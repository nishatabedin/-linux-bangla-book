# অধ্যায় ১৮: root, sudo ও প্রশাসনিক ক্ষমতা

## এই অধ্যায়ে যা শিখবে

- root ইউজার কে, তার ক্ষমতা কতটা, আর সারাক্ষণ root হয়ে কাজ করা কেন বিপজ্জনক
- ভেতরে কীভাবে "ক্ষমতা" যাচাই হয়: UID 0, setuid বিট আর PAM
- `su`, `su -`, `sudo`, `sudo -i`, `sudo -s`, এদের মধ্যে আসল পার্থক্য
- `sudo`-র গুরুত্বপূর্ণ অপশন: `-u`, `-l`, `-v`, `-k`, `-e`, `-E`, `-n`, `-b`
- `/etc/sudoers` ফাইলের সিনট্যাক্স, Alias, `Defaults`, Tag (`NOPASSWD` ইত্যাদি)
- `visudo` দিয়ে নিরাপদে এডিট করা, আর `/etc/sudoers.d/` ডিরেক্টরি ব্যবহার
- `sudo` গ্রুপ (Debian/Ubuntu) আর `wheel` গ্রুপ (Fedora/RHEL)
- কোনো ইউজারকে শুধু নির্দিষ্ট কয়েকটা কমান্ডের অনুমতি দেওয়া, এবং তার ফাঁদগুলো
- `sudo -l` দিয়ে নিজের অনুমতি দেখা, sudo-র লগ কোথায় থাকে আর কীভাবে পড়তে হয়
- পাসওয়ার্ড মনে রাখার সময় (timestamp timeout) কীভাবে কাজ করে
- `pkexec` ও polkit কী, GUI-তে পাসওয়ার্ড চাওয়ার পেছনের ব্যবস্থা
- Least privilege (ন্যূনতম ক্ষমতা) নীতি, আর বাস্তবে সেটা কীভাবে মানতে হয়

---

## ১৮.১ root কে?

লিনাক্সে প্রতিটি ইউজারের একটা সংখ্যা থাকে, যাকে বলে **UID (User ID)**। [অধ্যায় ১৫](15-users-groups.md)-এ এটা দেখেছ। এদের মধ্যে একজন বিশেষ: যার UID **0**। প্রথা অনুযায়ী তার নাম `root`।

```
$ id root
uid=0(root) gid=0(root) groups=0(root)
```

**root** হলো সিস্টেমের superuser (সর্বোচ্চ ক্ষমতার ইউজার)। কার্নেল যখন কোনো কাজের অনুমতি যাচাই করে, যেমন একটা ফাইল খোলা, অন্য ইউজারের process মেরে ফেলা বা নেটওয়ার্ক পোর্ট ৮০ খোলা, তখন সে প্রথমে দেখে: যে process অনুরোধ করছে তার effective UID কি 0? যদি হয়, তাহলে প্রায় সব পারমিশন চেক বাদ দেওয়া হয়।

> 📝 **মনে রাখো:** কার্নেল নাম দেখে না, সংখ্যা দেখে। কোনো ইউজারের নাম `root` না হয়েও UID 0 হলে সে-ও পুরো root। এজন্য নিরাপত্তা অডিটে `awk -F: '$3==0' /etc/passwd` চালিয়ে দেখা হয় UID 0 আর কারো আছে কি না।

### উপমা: অফিসের মাস্টার চাবি

একটা অফিস ভবন ভাবো। সাধারণ কর্মচারীর কাছে শুধু নিজের রুমের চাবি থাকে। ম্যানেজারের কাছে তার বিভাগের কয়েকটা রুমের চাবি থাকে। আর কেয়ারটেকারের কাছে থাকে **মাস্টার চাবি**, যা দিয়ে সব দরজা খোলে, সার্ভার রুম, ভল্ট, এমনকি বিদ্যুতের মেইন সুইচ বোর্ডও।

root হলো সেই মাস্টার চাবি। মাস্টার চাবি দরকারি, কিন্তু কেউ সারাদিন সেটা পকেটে নিয়ে ঘোরে না। হারালে বা ভুল দরজা খুললে বিপদ। `sudo` হলো সেই ব্যবস্থা যেখানে তুমি রেজিস্টার খাতায় সই করে কিছুক্ষণের জন্য মাস্টার চাবি নাও, কাজ শেষে ফেরত দাও, আর খাতায় লেখা থাকে কে কখন কোন দরজা খুলেছিল।

### root আসলে কী কী পারে

| কাজ | সাধারণ ইউজার | root |
|-----|---------------|------|
| অন্যের ফাইল পড়া/মোছা (পারমিশন না থাকলে) | না | হ্যাঁ |
| `/etc`, `/usr`, `/boot`-এ লেখা | না | হ্যাঁ |
| সফটওয়্যার ইনস্টল/আনইনস্টল | না | হ্যাঁ |
| অন্য ইউজারের process `kill` করা | না | হ্যাঁ |
| ১০২৪-এর নিচের পোর্ট খোলা (যেমন ৮০, ৪৪৩) | না | হ্যাঁ |
| ডিস্ক ফরম্যাট, পার্টিশন বদলানো | না | হ্যাঁ |
| কার্নেল মডিউল লোড করা | না | হ্যাঁ |
| যেকোনো ইউজারের পাসওয়ার্ড বদলানো | না | হ্যাঁ |
| ফায়ারওয়াল নিয়ম বদলানো | না | হ্যাঁ |

### root-ও কি সব পারে?

প্রায় সব, কিন্তু পুরোপুরি না। আধুনিক লিনাক্সে root-এর ক্ষমতা ছোট ছোট টুকরোয় ভাগ করা, এগুলোকে বলে **capabilities** (যেমন `CAP_NET_BIND_SERVICE`, `CAP_SYS_ADMIN`)। root-এর সবগুলো থাকে, কিন্তু কন্টেইনারের ভেতরের "root"-এর অনেকগুলো থাকে না ([অধ্যায় ৪৮](48-containers-virtualization.md))। এছাড়া SELinux/AppArmor-এর মতো MAC (Mandatory Access Control) ব্যবস্থা root-কেও আটকাতে পারে ([অধ্যায় ৪৫](45-security-hardening.md))। আর `chattr +i` দেওয়া ফাইল root-ও সরাসরি মুছতে পারে না, আগে অ্যাট্রিবিউট সরাতে হয় ([অধ্যায় ১৭](17-special-permissions-acl.md))।

---

## ১৮.২ সারাক্ষণ root হয়ে থাকা কেন বিপজ্জনক

নতুনরা প্রায়ই ভাবে: "বারবার পাসওয়ার্ড দেওয়া ঝামেলা, root হয়েই সব কাজ করি।" এটা সবচেয়ে বড় ভুলগুলোর একটা। কারণগুলো দেখো:

**১. কোনো "আনডু" নেই।** লিনাক্সে `rm` করা ফাইল রিসাইকেল বিনে যায় না। একটা স্পেসের ভুল যথেষ্ট:

```
# rm -rf / home/rahim/old-project     ← ভুল: "/" আর "home" এর মাঝে স্পেস
```

সাধারণ ইউজার হিসেবে এই কমান্ড হাজার হাজার "Permission denied" দিয়ে থেমে যেত, ক্ষতি হতো শুধু নিজের ফাইলের। root হিসেবে এটা পুরো সিস্টেম মুছে ফেলার চেষ্টা করবে। (আধুনিক GNU `rm` সরাসরি `rm -rf /` আটকায় `--preserve-root` ডিফল্ট থাকায়, কিন্তু `rm -rf /*` আটকায় না।)

**২. ম্যালওয়্যার আর বাগের ক্ষতি সীমাহীন হয়।** ব্রাউজার বা কোনো স্ক্রিপ্ট যদি root হিসেবে চলে আর তাতে নিরাপত্তা ত্রুটি থাকে, আক্রমণকারী সাথে সাথে পুরো সিস্টেমের মালিক হয়ে যায়। সাধারণ ইউজার হিসেবে চললে ক্ষতি সেই ইউজারের ফাইলের মধ্যে আটকে থাকে।

**৩. কোনো জবাবদিহি থাকে না।** পাঁচজন অ্যাডমিন যদি একই root পাসওয়ার্ড শেয়ার করে, লগে শুধু লেখা থাকবে "root এটা করেছে"। কে করেছে বোঝার উপায় নেই। `sudo` প্রতিটি কমান্ডের সাথে আসল ইউজারের নাম লগে লিখে রাখে।

**৪. পাসওয়ার্ড শেয়ার করা মানেই ঝুঁকি।** একজন কর্মী চাকরি ছাড়লে root পাসওয়ার্ড বদলাতে হয় আর বাকি সবাইকে নতুনটা জানাতে হয়। sudo-তে শুধু ওই ইউজারকে গ্রুপ থেকে সরালেই হয়।

**৫. ক্লান্ত মাথা ভুল করে।** রাত দুটোয় সার্ভার ঠিক করার সময় প্রম্পটে `#` দেখেও খেয়াল না করে ভুল টার্মিনালে কমান্ড চালানো খুব সাধারণ ঘটনা।

> ⚠️ **সতর্কতা:** প্রম্পটের শেষ চিহ্নটা খেয়াল করো। `$` মানে সাধারণ ইউজার, `#` মানে root। এই বইয়ে উদাহরণে `$` দেখলে সাধারণ ইউজার, `#` দেখলে root shell বোঝাবে।

### Ubuntu কেন root অ্যাকাউন্ট "লক" রাখে

Ubuntu ইনস্টল করার সময় root-এর পাসওয়ার্ড চাওয়া হয় না। root অ্যাকাউন্ট আছে, কিন্তু তার পাসওয়ার্ড **লক** করা। ফলে কেউ সরাসরি root হিসেবে লগইন করতে পারে না। প্রথম যে ইউজার তৈরি হয়, তাকে `sudo` গ্রুপে রাখা হয়।

```
$ sudo passwd -S root
root L 2024-04-23 0 99999 7 -1
```

এখানে দ্বিতীয় কলামের `L` মানে Locked। (`P` হলে usable password আছে, `NP` হলে কোনো পাসওয়ার্ড নেই।)

বিভিন্ন distro-র অবস্থা:

| Distro | root পাসওয়ার্ড | প্রথম ইউজারের অ্যাডমিন গ্রুপ |
|--------|-----------------|------------------------------|
| Ubuntu 24.04 | ডিফল্টে লক | `sudo` |
| Debian 12 | ইনস্টলার জিজ্ঞেস করে। root পাসওয়ার্ড দিলে প্রথম ইউজার `sudo` গ্রুপে যায় না, এমনকি `sudo` প্যাকেজই নাও থাকতে পারে। ফাঁকা রাখলে root লক হয় আর প্রথম ইউজার `sudo` গ্রুপে যায় | `sudo` |
| Fedora | ইনস্টলারে root ডিফল্টে লক। ইউজার তৈরির সময় "Make this user administrator" টিক দিলে `wheel` গ্রুপে যায় | `wheel` |
| RHEL 9 / Rocky / Alma | ইনস্টলারে root পাসওয়ার্ড সেট করা যায় বা লক রাখা যায়। ডিফল্টে root-এর পাসওয়ার্ড দিয়ে SSH লগইন বন্ধ | `wheel` |

---

## ১৮.৩ ভেতরে কীভাবে কাজ করে: UID, setuid আর PAM

`sudo` বা `su` কীভাবে তোমাকে root বানায়, এটা বুঝলে বাকি সব সহজ হয়ে যায়।

প্রতিটি process-এর সাথে কার্নেল কয়েকটা পরিচয় রাখে:

- **Real UID (RUID):** আসলে কে process টা চালু করেছে
- **Effective UID (EUID):** পারমিশন যাচাইয়ে কার পরিচয় ব্যবহার হবে
- **Saved UID:** পরে ফিরে যাওয়ার জন্য সংরক্ষিত পরিচয়

সাধারণত তিনটাই এক। এখন প্রশ্ন: সাধারণ ইউজার (UID 1000) চালানো একটা process কীভাবে UID 0 হয়? উত্তর **setuid বিট** ([অধ্যায় ১৭](17-special-permissions-acl.md))।

```
$ ls -l /usr/bin/sudo /usr/bin/su
-rwsr-xr-x 1 root root 277936 Apr  8  2024 /usr/bin/sudo
-rwsr-xr-x 1 root root  55680 Apr  9  2024 /usr/bin/su
```

`rws`-এর `s` হলো setuid। এর মানে: যে-ই এই প্রোগ্রাম চালাক, প্রোগ্রামটা চলবে ফাইলের **মালিকের** (root) effective UID নিয়ে। তাই `sudo` চালু হওয়ার মুহূর্তেই তার EUID 0। কিন্তু sudo বোকা না। root ক্ষমতা হাতে পেয়েও সে নিজে নিয়ম মেনে যাচাই করে তুমি অনুমতি পাওয়ার যোগ্য কি না।

```
 তুমি টাইপ করলে:  $ sudo apt update
          │
          ▼
 ┌────────────────────────────────────────────────────────┐
 │ shell fork+exec করে /usr/bin/sudo                       │
 │ setuid বিটের কারণে sudo চালু হয় EUID=0 নিয়ে            │
 │ (RUID তখনও তোমার, যেমন 1000)                            │
 └───────────────────────┬────────────────────────────────┘
                         ▼
 ┌────────────────────────────────────────────────────────┐
 │ ১. /etc/sudoers (+ /etc/sudoers.d/*) পড়ে                │
 │    "এই ইউজার কি এই কমান্ড, এই হোস্টে, এই                  │
 │     target user হিসেবে চালাতে পারবে?"                    │
 │    না হলে → অস্বীকার + লগে লেখা                          │
 └───────────────────────┬────────────────────────────────┘
                         ▼
 ┌────────────────────────────────────────────────────────┐
 │ ২. timestamp আছে ও মেয়াদ আছে? (গত ১৫ মিনিটে পাসওয়ার্ড?) │
 │    না থাকলে → PAM (/etc/pam.d/sudo) দিয়ে তোমার            │
 │    নিজের পাসওয়ার্ড যাচাই                                  │
 └───────────────────────┬────────────────────────────────┘
                         ▼
 ┌────────────────────────────────────────────────────────┐
 │ ৩. লগ লেখে: কে, কোন tty, কোন ডিরেক্টরি, কোন কমান্ড       │
 │ ৪. environment পরিষ্কার করে (env_reset, secure_path)    │
 │ ৫. setuid(0)/setgid(0) করে RUID-ও 0 বানায়               │
 │ ৬. exec করে /usr/bin/apt update                          │
 └────────────────────────────────────────────────────────┘
```

**PAM (Pluggable Authentication Modules)** হলো লিনাক্সের প্রমাণীকরণের একটা প্লাগইন ব্যবস্থা। `sudo`, `su`, `login`, `sshd` নিজেরা পাসওয়ার্ড যাচাইয়ের কোড লেখে না, তারা PAM-কে জিজ্ঞেস করে। PAM কনফিগারেশন থাকে `/etc/pam.d/`-এ। এজন্যই ফিঙ্গারপ্রিন্ট বা দুই-ধাপের যাচাই (2FA) সহজে sudo-তে যোগ করা যায়, sudo-র কোড না বদলেই।

> 💡 **টিপস:** `/etc/sudoers` ফাইলের পারমিশন `0440` (`-r--r-----`) আর মালিক root। মানে সাধারণ ইউজার এটা পড়তেও পারে না। sudo setuid root হয়ে চলে বলেই নিজে ফাইলটা পড়তে পারে।

---

## ১৮.৪ `su` — ইউজার বদলানো

`su` মানে **substitute user** (অনেকে বলে "switch user")। এটা তোমাকে অন্য ইউজারে বদলে দেয়, আর **সেই target ইউজারের পাসওয়ার্ড** চায়।

**সিনট্যাক্স:**

```
su [options] [-] [user [argument...]]
```

user না দিলে ধরে নেওয়া হয় `root`।

| অপশন | কাজ |
|-------|------|
| `-` , `-l`, `--login` | পূর্ণ login shell: environment রিসেট, target-এর home-এ চলে যায়, তার profile ফাইল পড়ে |
| `-c "command"` | shell না খুলে শুধু একটা কমান্ড চালিয়ে ফিরে আসে |
| `-s /path/shell` | নির্দিষ্ট shell ব্যবহার করে |
| `-m`, `-p` | environment অপরিবর্তিত রাখে (`--preserve-environment`) |
| `-g group` | primary group বদলায় (শুধু root ব্যবহার করতে পারে) |

### `su` বনাম `su -` : আসল পার্থক্য

এটা ইন্টারভিউয়ের প্রিয় প্রশ্ন। পার্থক্য হলো **environment** (পরিবেশ চলক, যেমন `PATH`, `HOME`, `USER`) আর **বর্তমান ডিরেক্টরি**।

```
$ pwd
/home/rahim/projects
$ su
Password:
# pwd
/home/rahim/projects          ← একই ডিরেক্টরিতে রয়ে গেছ
# echo $HOME
/root                         ← util-linux su, HOME আর SHELL বদলায়
# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin   ← distro ও কনফিগ অনুযায়ী ভিন্ন হতে পারে
# exit
$ su -
Password:
# pwd
/root                         ← root-এর home-এ চলে গেছ
# echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

| বিষয় | `su` | `su -` |
|-------|------|--------|
| ডিরেক্টরি | যেখানে ছিলে সেখানেই | target-এর home (`/root`) |
| Shell ধরন | non-login shell | login shell |
| পড়ে | `~/.bashrc` | `/etc/profile`, `~/.bash_profile` বা `~/.profile` (আর সেখান থেকে সাধারণত `.bashrc`) |
| পুরনো environment | বেশির ভাগ থেকে যায় | প্রায় সব মুছে নতুন করে সেট হয় |
| নিরাপদ? | কম (তোমার `PATH`-এর কোনো ক্ষতিকর ফোল্ডার থেকে যেতে পারে) | বেশি |

> 📝 **মনে রাখো:** root হতে চাইলে সবসময় `su -` ব্যবহার করো, শুধু `su` নয়। শুধু `su`-তে তোমার নিজের environment (যেমন অদ্ভুত `PATH` বা alias) root shell-এ চলে আসতে পারে, যা বিভ্রান্তি আর নিরাপত্তা ঝুঁকি তৈরি করে।

### অন্য ইউজার হওয়া আর একটা কমান্ড চালানো

```
$ su - karim                       # karim হও (karim-এর পাসওয়ার্ড লাগবে)
$ su -c "systemctl restart nginx"  # root হিসেবে একটা কমান্ড চালাও, তারপর ফিরে এসো
Password:
$
```

### Ubuntu-তে `su` কেন কাজ করে না

```
$ su -
Password:
su: Authentication failure
```

কারণ root-এর পাসওয়ার্ড লক। `su` সবসময় **target**-এর পাসওয়ার্ড চায়, আর root-এর তো কোনো ব্যবহারযোগ্য পাসওয়ার্ডই নেই। সমাধান: `sudo -i` ব্যবহার করো (নিচে দেখো)। root-এর পাসওয়ার্ড সেট করা (`sudo passwd root`) সম্ভব, কিন্তু সাধারণত দরকার নেই, বরং অপ্রয়োজনীয় ঝুঁকি।

root shell থেকে বের হতে `exit` লেখো বা `Ctrl+D` চাপো।

---

## ১৮.৫ `sudo` — একটা কমান্ড, অল্প সময়ের জন্য ক্ষমতা

`sudo` মানে "**su**bstitute user **do**" (আগে বলা হতো "superuser do")। এটা **একটা কমান্ড** অন্য ইউজার (ডিফল্টে root) হিসেবে চালায়, আর চায় **তোমার নিজের পাসওয়ার্ড**।

### `su` আর `sudo`-র মূল দর্শনগত পার্থক্য

| বিষয় | `su` | `sudo` |
|-------|------|--------|
| কার পাসওয়ার্ড চায় | target ইউজারের (root-এর) | তোমার নিজের |
| কতক্ষণ ক্ষমতা | যতক্ষণ shell খোলা | শুধু ওই একটা কমান্ড (+ কিছুক্ষণ পাসওয়ার্ড মনে রাখে) |
| কে পারবে তার নিয়ন্ত্রণ | যে পাসওয়ার্ড জানে, সে-ই | `/etc/sudoers` ফাইলে সূক্ষ্ম নিয়ম |
| নির্দিষ্ট কমান্ডে সীমাবদ্ধ করা | সম্ভব না | সম্ভব |
| লগ | "কেউ root হয়েছে" | প্রতিটি কমান্ড, আসল ইউজারের নামসহ |
| পাসওয়ার্ড শেয়ার | করতে হয় | করতে হয় না |

### প্রথম sudo

```
$ apt update
Reading package lists... Done
E: Could not open lock file /var/lib/apt/lists/lock - open (13: Permission denied)
E: Unable to lock directory /var/lib/apt/lists/
$ sudo apt update
[sudo] password for rahim:
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
...
```

> 💡 **টিপস:** পাসওয়ার্ড টাইপ করার সময় স্ক্রিনে কিছু দেখা যায় না, এমনকি তারকা চিহ্নও না। এটা স্বাভাবিক, কিবোর্ড নষ্ট না। টাইপ করে Enter চাপো। (কিছু distro যেমন Linux Mint `pwfeedback` চালু রাখে, তখন `*` দেখায়।)

প্রথমবার sudo চালালে একটা "lecture" (উপদেশ) দেখায়:

```
We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.
```

তিনটা কথাই সত্যি, বিশেষ করে দ্বিতীয়টা।

### অনুমতি না থাকলে

```
$ sudo apt update
[sudo] password for karim:
karim is not in the sudoers file.
```

(পুরনো sudo ভার্সনে এর সাথে "This incident will be reported." লেখা থাকত।) এই চেষ্টাটা লগে রেকর্ড হয়।

### `sudo`-র গুরুত্বপূর্ণ অপশন

**সিনট্যাক্স:**

```
sudo [options] command [arguments]
```

| অপশন | কাজ |
|-------|------|
| `-u user` | root-এর বদলে নির্দিষ্ট ইউজার হিসেবে চালাও |
| `-g group` | নির্দিষ্ট গ্রুপকে primary group করে চালাও |
| `-i` | target ইউজারের **login shell** খোলো (অথবা login shell-এর মাধ্যমে কমান্ড চালাও) |
| `-s` | `$SHELL`-এ থাকা shell খোলো (non-login) |
| `-l` | তুমি কী কী চালাতে পারো তার তালিকা (`-ll` দিলে বিস্তারিত) |
| `-l command` | নির্দিষ্ট কমান্ডটা অনুমোদিত কি না দেখাও (পূর্ণ path দেখায়) |
| `-U user -l` | অন্য ইউজারের অনুমতি দেখাও (এর জন্য নিজের যথেষ্ট অধিকার লাগে) |
| `-v` | কোনো কমান্ড না চালিয়ে timestamp নবায়ন করো (পাসওয়ার্ড আবার মনে রাখাও) |
| `-k` | বর্তমান session-এর timestamp বাতিল করো (পরের বার আবার পাসওয়ার্ড চাইবে) |
| `-K` | ইউজারের সব timestamp পুরোপুরি মুছে ফেলো |
| `-e file` | ফাইল নিরাপদে এডিট করো (`sudoedit`-এর মতো) |
| `-E` | তোমার environment রেখে দাও (sudoers-এ অনুমতি থাকলে) |
| `--preserve-env=VAR1,VAR2` | শুধু নির্দিষ্ট চলক রেখে দাও |
| `-n` | non-interactive: পাসওয়ার্ড লাগলে না চেয়ে ভুল হিসেবে বের হয়ে যাও (স্ক্রিপ্টে কাজের) |
| `-b` | কমান্ড background-এ চালাও |
| `-H` | `HOME` চলক target-এর home-এ সেট করো (আধুনিক sudo-তে এটা সাধারণত ডিফল্ট) |
| `-S` | পাসওয়ার্ড stdin থেকে পড়ো |
| `-V` | sudo-র ভার্সন দেখাও (root হিসেবে চালালে কনফিগ তথ্যও) |

উদাহরণ:

```
$ sudo -u postgres psql                 # postgres ইউজার হিসেবে psql চালাও
$ sudo -u www-data touch /var/www/html/test.txt
$ sudo -n true && echo "পাসওয়ার্ড ছাড়াই sudo চলবে" || echo "পাসওয়ার্ড লাগবে"
$ sudo -v                               # এখনই পাসওয়ার্ড দিয়ে রাখো, লম্বা স্ক্রিপ্টের আগে কাজের
$ sudo -k                               # কম্পিউটার ছেড়ে ওঠার আগে "চাবি ফেরত দাও"
```

### `sudo !!` — আগের কমান্ডটা sudo দিয়ে

কমান্ড চালিয়ে "Permission denied" পেলে পুরোটা আবার টাইপ করতে হবে না:

```
$ cat /etc/shadow
cat: /etc/shadow: Permission denied
$ sudo !!
sudo cat /etc/shadow
[sudo] password for rahim:
root:*:19836:0:99999:7:::
...
```

`!!` হলো bash-এর history expansion, মানে "আগের পুরো কমান্ড" ([অধ্যায় ৬](06-terminal-shortcuts.md))।

---

## ১৮.৬ `sudo -i`, `sudo -s`, `sudo su -`: root shell পাওয়ার নানা উপায়

কখনো কখনো অনেকগুলো প্রশাসনিক কমান্ড পরপর চালাতে হয়। তখন প্রতিবার `sudo` না লিখে একটা root shell নেওয়া যায়।

```
$ pwd
/home/rahim/projects

$ sudo -s
# pwd
/home/rahim/projects       ← ডিরেক্টরি বদলায়নি
# echo $HOME
/root                      ← আধুনিক sudo HOME বদলে দেয়
# exit

$ sudo -i
# pwd
/root                      ← root-এর home-এ
# exit
```

| কমান্ড | কার পাসওয়ার্ড | shell-এর ধরন | ডিরেক্টরি | environment |
|--------|----------------|--------------|-----------|-------------|
| `su` | root-এর | non-login | একই থাকে | অনেকটা তোমারটাই |
| `su -` | root-এর | login | `/root` | root-এর পূর্ণ login environment |
| `sudo -s` | তোমার | non-login (`$SHELL`) | একই থাকে | sudo রিসেট করা environment, root-এর `.bashrc` পড়ে |
| `sudo -i` | তোমার | login (root-এর shell, `/etc/passwd` থেকে) | `/root` | root-এর পূর্ণ login environment |
| `sudo su -` | তোমার (sudo-র জন্য) | login | `/root` | `su -`-এর মতো |
| `sudo bash` | তোমার | non-login | একই থাকে | `sudo -s`-এর কাছাকাছি |

> 💡 **টিপস:** `sudo su -` পুরনো অভ্যাস, কাজ করে, কিন্তু এখানে দুটো setuid প্রোগ্রাম একটার ভেতর আরেকটা চলে। একই কাজ `sudo -i` সরাসরি করে, এটাই পরিষ্কার উপায়।

**কোনটা কখন?**
- একটা কমান্ড: `sudo command` (সবচেয়ে ভালো)
- পরপর অনেক অ্যাডমিন কাজ, পরিষ্কার environment চাই: `sudo -i`
- বর্তমান ডিরেক্টরিতে থেকে কয়েকটা কমান্ড: `sudo -s`
- কাজ শেষ হলেই `exit`। root shell খোলা রেখে উঠে যেও না।

### একটা সাধারণ ফাঁদ: sudo আর redirection

```
$ sudo echo "nameserver 1.1.1.1" > /etc/resolv.conf
bash: /etc/resolv.conf: Permission denied
```

কেন? কারণ `>` (redirection) করে **তোমার shell**, sudo নয়। shell প্রথমে সাধারণ ইউজার হিসেবে ফাইলটা লেখার জন্য খুলতে চায়, ব্যর্থ হয়। sudo শুধু `echo` কে root বানায়, যার কোনো দরকারই ছিল না।

```
   তোমার shell (rahim)                         sudo → echo (root)
 ┌───────────────────────────┐
 │ > /etc/resolv.conf খোলো    │  ← এখানেই ব্যর্থ, rahim লিখতে পারে না
 └───────────────────────────┘
```

সমাধান:

```
$ echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf       # ওভাররাইট
$ echo "nameserver 8.8.8.8" | sudo tee -a /etc/resolv.conf    # যোগ (append)
$ sudo sh -c 'echo "nameserver 1.1.1.1" > /etc/resolv.conf'   # পুরো কমান্ড root shell-এ
```

`tee` সম্পর্কে বিস্তারিত [অধ্যায় ১৯](19-redirection-pipe.md)-এ।

### `sudoedit` / `sudo -e` — root ফাইল নিরাপদে এডিট করা

`sudo vim /etc/hosts` চালালে পুরো এডিটরটাই root হিসেবে চলে। vim-এর ভেতর থেকে `:!bash` দিয়ে root shell পাওয়া যায়, প্লাগইনও root হিসেবে চলে। নিরাপদ উপায়:

```
$ sudoedit /etc/hosts
```

ভেতরে যা ঘটে:
1. sudo ফাইলটার একটা অস্থায়ী কপি বানায়, তোমার মালিকানায়
2. তোমার এডিটর (`SUDO_EDITOR`, `VISUAL` বা `EDITOR` চলক অনুযায়ী) **তোমার** নামে চলে
3. সেভ করে বের হলে sudo কপিটা আসল জায়গায় ফেরত লেখে

এডিটর কখনো root হয় না। নিজের পছন্দের এডিটর সেট করতে: `export SUDO_EDITOR=nano` (স্থায়ী করতে `.bashrc`-এ, [অধ্যায় ৪৪](44-environment-bashrc.md))।

---

## ১৮.৭ কে sudo পাবে: `sudo` আর `wheel` গ্রুপ

প্রতিটি ইউজারের জন্য আলাদা নিয়ম না লিখে, distro-গুলো একটা গ্রুপকে পূর্ণ অ্যাডমিন অধিকার দিয়ে রাখে:

| Distro পরিবার | গ্রুপ | `/etc/sudoers`-এর লাইন |
|---------------|-------|------------------------|
| Debian, Ubuntu, Mint | `sudo` | `%sudo   ALL=(ALL:ALL) ALL` |
| Fedora, RHEL, Rocky, Alma, CentOS | `wheel` | `%wheel  ALL=(ALL)       ALL` |
| Arch | `wheel` (লাইনটা ডিফল্টে comment করা, নিজে চালু করতে হয়) | `# %wheel ALL=(ALL:ALL) ALL` |

"wheel" নামটা এসেছে পুরনো ইউনিক্স যুগের "big wheel" (গুরুত্বপূর্ণ ব্যক্তি) কথা থেকে।

### কাউকে অ্যাডমিন বানানো

```
# Debian/Ubuntu
$ sudo usermod -aG sudo karim

# Fedora/RHEL
$ sudo usermod -aG wheel karim

# অথবা gpasswd দিয়ে
$ sudo gpasswd -a karim sudo
Adding user karim to group sudo
```

> ⚠️ **সতর্কতা:** `usermod`-এ `-a` (append) দিতে ভুলবে না। শুধু `-G sudo` দিলে karim-এর বাকি সব secondary group মুছে গিয়ে শুধু `sudo` থাকবে। বিস্তারিত [অধ্যায় ১৫](15-users-groups.md)-এ।

গ্রুপ পরিবর্তন সাথে সাথে চলমান session-এ প্রযোজ্য হয় না। karim-কে লগআউট করে আবার লগইন করতে হবে (বা SSH session নতুন করে খুলতে হবে)। যাচাই:

```
$ id karim
uid=1001(karim) gid=1001(karim) groups=1001(karim),27(sudo)
$ groups karim
karim : karim sudo
```

### কারো অ্যাডমিন অধিকার কেড়ে নেওয়া

```
$ sudo gpasswd -d karim sudo
Removing user karim from group sudo
# অথবা
$ sudo deluser karim sudo        # Debian/Ubuntu
```

> 📝 **মনে রাখো:** অধিকার কেড়ে নেওয়ার পর karim-এর আগে থেকে খোলা root shell (যদি থাকে) বন্ধ হয় না। গুরুতর ক্ষেত্রে তার চলমান session শেষ করে দাও (`sudo loginctl terminate-user karim`)।

---

## ১৮.৮ `/etc/sudoers` ফাইলের গঠন

এবার মূল নিয়ম-বইটা খুলি। Ubuntu 24.04-এ ডিফল্ট ফাইলটা (comment বাদে) মোটামুটি এরকম:

```
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
Defaults        use_pty

# User privilege specification
root    ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

@includedir /etc/sudoers.d
```

RHEL 9 / Fedora-তে মূল লাইনগুলো:

```
root    ALL=(ALL)       ALL
%wheel  ALL=(ALL)       ALL
## %wheel        ALL=(ALL)       NOPASSWD: ALL
#includedir /etc/sudoers.d
```

> ⚠️ **সতর্কতা:** `#includedir`-এর শুরুতে `#` থাকলেও এটা comment **না**! এটা পুরনো সিনট্যাক্সের একটা নির্দেশ। sudo 1.9.1 থেকে নতুন রূপ `@includedir` চালু হয়েছে, দুটোই কাজ করে। ভুল করে এই লাইন মুছবে না।

### একটা নিয়মের লাইন পড়া (User Specification)

```
  %sudo     ALL   =   (ALL : ALL)       ALL
   │         │          │     │          │
   │         │          │     │          └─ কোন কমান্ড চালাতে পারবে
   │         │          │     └──────────── কোন গ্রুপ হিসেবে চালাতে পারবে (Runas group)
   │         │          └────────────────── কোন ইউজার হিসেবে চালাতে পারবে (Runas user)
   │         └───────────────────────────── কোন হোস্টে (মেশিনে) এই নিয়ম প্রযোজ্য
   └─────────────────────────────────────── কে (ইউজার, বা % দিলে গ্রুপ)
```

সাধারণ রূপ:

```
WHO   WHERE = (AS_USER:AS_GROUP)  [TAG:]  COMMANDS
```

পড়ো এভাবে: "**sudo গ্রুপের সদস্যরা, যেকোনো হোস্টে, যেকোনো ইউজার ও যেকোনো গ্রুপ হিসেবে, যেকোনো কমান্ড চালাতে পারবে।**"

অংশগুলো বিস্তারিত:

| অংশ | কী লেখা যায় | উদাহরণ |
|-----|---------------|---------|
| WHO | ইউজারনেম, `%group`, `#UID`, `%#GID`, User_Alias | `rahim`, `%devops`, `#1005` |
| WHERE | হোস্টনেম, IP, নেটওয়ার্ক, Host_Alias, `ALL` | `web01`, `192.168.1.0/24`, `ALL` |
| AS_USER | ইউজারনেম, Runas_Alias, `ALL` | `(root)`, `(postgres)`, `(ALL)` |
| AS_GROUP | গ্রুপনেম, `ALL` (ঐচ্ছিক) | `(ALL:ALL)`, `(root:adm)` |
| TAG | `NOPASSWD:`, `PASSWD:`, `NOEXEC:`, `EXEC:`, `SETENV:`, `NOSETENV:` ইত্যাদি | `NOPASSWD:` |
| COMMANDS | **পূর্ণ path**সহ কমান্ড, Cmnd_Alias, `ALL` | `/usr/bin/systemctl restart nginx` |

`(AS_USER)` একদম না লিখলে ডিফল্ট ধরা হয় `(root)`।

**WHERE বা হোস্টের অংশ কেন?** বড় প্রতিষ্ঠানে একই `sudoers` ফাইল শত শত সার্ভারে কপি করা হয় (Ansible বা অন্য কোনো টুল দিয়ে)। তখন এক লাইনেই বলা যায় "rahim শুধু web সার্ভারগুলোতে nginx রিস্টার্ট করতে পারবে"। একক মেশিনে সাধারণত `ALL` লেখা হয়।

### উদাহরণ নিয়ম

```
# rahim পূর্ণ অ্যাডমিন
rahim   ALL=(ALL:ALL) ALL

# karim শুধু nginx রিস্টার্ট ও রিলোড করতে পারবে
karim   ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx

# dbteam গ্রুপ শুধু postgres ইউজার হিসেবে যেকোনো কমান্ড
%dbteam ALL=(postgres) ALL

# backup ইউজার পাসওয়ার্ড ছাড়া rsync চালাতে পারবে
backup  ALL=(root) NOPASSWD: /usr/bin/rsync

# deploy ইউজার: একটা কমান্ডে পাসওয়ার্ড লাগবে না, অন্যটায় লাগবে
deploy  ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp, PASSWD: /usr/bin/apt
```

শেষ লাইনে দেখো, একবার Tag দিলে সেটা পরের সব কমান্ডে প্রযোজ্য থাকে যতক্ষণ না উল্টো Tag আসে।

> 📝 **মনে রাখো:** কমান্ড সবসময় **পূর্ণ path** দিয়ে লিখতে হয়। `systemctl` না, `/usr/bin/systemctl`। path জানতে `command -v systemctl` বা `which systemctl` চালাও ([অধ্যায় ১২](12-file-khoja.md))। নইলে কেউ নিজের ফোল্ডারে `systemctl` নামে একটা ক্ষতিকর স্ক্রিপ্ট রেখে দিতে পারত।

### কমান্ডে আর্গুমেন্টের নিয়ম

| লেখা | মানে |
|------|------|
| `/usr/bin/systemctl` | যেকোনো আর্গুমেন্টসহ systemctl (বা কোনো আর্গুমেন্ট ছাড়া) |
| `/usr/bin/systemctl restart nginx` | ঠিক এই আর্গুমেন্টগুলো, অন্য কিছু না |
| `/usr/bin/systemctl ""` | শুধু আর্গুমেন্ট ছাড়া |
| `/usr/bin/systemctl restart *` | `restart` এর পর যেকোনো কিছু (wildcard, বিপজ্জনক হতে পারে) |
| `!/usr/bin/su` | এই কমান্ড বাদ (negation, নিচে ফাঁদ দেখো) |
| `sudoedit /etc/nginx/nginx.conf` | এই ফাইলটা `sudoedit` দিয়ে এডিট করতে পারবে |

### Alias — নিয়মকে সংগঠিত করা

অনেক ইউজার আর অনেক কমান্ড থাকলে Alias দিয়ে নাম দেওয়া যায়। Alias-এর নাম **বড় হাতের অক্ষরে** (A-Z, 0-9, _) হতে হয়।

```
# কারা
User_Alias   WEBADMINS = rahim, karim, %webteam

# কোন মেশিনে
Host_Alias   WEBSERVERS = web01, web02, 10.0.5.0/24

# কার হিসেবে
Runas_Alias  APPUSERS = www-data, nginx

# কোন কমান্ড
Cmnd_Alias   WEBCTL = /usr/bin/systemctl restart nginx, \
                      /usr/bin/systemctl reload nginx, \
                      /usr/bin/systemctl status nginx
Cmnd_Alias   PKG    = /usr/bin/apt, /usr/bin/apt-get, /usr/bin/dpkg

# নিয়ম
WEBADMINS  WEBSERVERS = (root) WEBCTL
WEBADMINS  WEBSERVERS = (APPUSERS) NOPASSWD: ALL
```

লম্বা লাইন ভাঙতে শেষে `\` দেওয়া যায়।

### `Defaults` — sudo-র আচরণ বদলানো

`Defaults` লাইন দিয়ে sudo-র সেটিংস বদলানো হয়। এগুলো সবার জন্য, বা নির্দিষ্ট ইউজার/হোস্ট/কমান্ডের জন্য দেওয়া যায়:

```
Defaults              env_reset                 # সবার জন্য
Defaults:rahim        timestamp_timeout=30      # শুধু rahim-এর জন্য
Defaults:%devops      !lecture                  # devops গ্রুপের জন্য lecture বন্ধ
Defaults@web01        log_output                # শুধু web01 হোস্টে
Defaults>postgres     !requiretty               # postgres হিসেবে চালালে
Defaults!/usr/bin/apt !log_output               # নির্দিষ্ট কমান্ডের জন্য
```

গুরুত্বপূর্ণ `Defaults` অপশন (সবগুলো `man sudoers`-এ আছে):

| অপশন | কাজ |
|-------|------|
| `env_reset` | environment পরিষ্কার করে শুধু নিরাপদ কিছু চলক রাখে (ডিফল্টে চালু) |
| `env_keep += "VAR"` | নির্দিষ্ট চলক রেখে দাও (যেমন `http_proxy`) |
| `secure_path="..."` | sudo কমান্ডের জন্য নির্দিষ্ট নিরাপদ `PATH` |
| `timestamp_timeout=N` | কত মিনিট পাসওয়ার্ড মনে রাখবে (নিচে বিস্তারিত) |
| `timestamp_type=tty` | timestamp কীভাবে আলাদা রাখবে: `tty`, `ppid`, `global`, `kernel` |
| `passwd_tries=N` | ভুল পাসওয়ার্ডের সর্বোচ্চ চেষ্টা (ডিফল্ট ৩) |
| `passwd_timeout=N` | পাসওয়ার্ড প্রম্পট কত মিনিট অপেক্ষা করবে (০ মানে অসীম) |
| `badpass_message="..."` | ভুল পাসওয়ার্ডে দেখানো বার্তা |
| `insults` | ভুল পাসওয়ার্ডে মজার অপমানজনক বার্তা (মজা, তবে পেশাদার সার্ভারে নয়) |
| `pwfeedback` | পাসওয়ার্ড টাইপের সময় `*` দেখায় |
| `lecture=always/once/never` | lecture কখন দেখাবে |
| `logfile="/var/log/sudo.log"` | syslog ছাড়াও আলাদা লগ ফাইলে লেখো |
| `log_input`, `log_output` | পুরো session-এর ইনপুট/আউটপুট রেকর্ড করো (`sudoreplay` দিয়ে দেখা যায়) |
| `use_pty` | কমান্ড আলাদা pseudo-terminal-এ চালাও (নিরাপত্তার জন্য, Ubuntu-তে চালু) |
| `requiretty` | শুধু আসল terminal থেকে sudo চলবে (কিছু পুরনো RHEL কনফিগে ছিল) |
| `mail_badpass` | ভুল পাসওয়ার্ড দিলে `mailto`-তে মেইল পাঠাও |
| `editor=/usr/bin/nano` | `visudo` কোন এডিটর ব্যবহার করবে |
| `rootpw` | ইউজারের নিজের বদলে root-এর পাসওয়ার্ড চাও (openSUSE-এর মতো `targetpw` ও আছে) |

বুলিয়ান অপশন বন্ধ করতে আগে `!` দাও: `Defaults !lecture`।

### নিয়মের ক্রম: শেষ মিলটাই জেতে

একজন ইউজারের সাথে একাধিক লাইন মিলে গেলে, **ফাইলে সবার শেষে যে লাইন মেলে**, সেটা কার্যকর। এজন্য নির্দিষ্ট নিয়মগুলো সাধারণ নিয়মের **পরে** লিখতে হয়।

```
%sudo   ALL=(ALL:ALL) ALL
rahim   ALL=(ALL) NOPASSWD: /usr/bin/apt        ← rahim sudo গ্রুপে, এটা পরে আছে, তাই apt-এ পাসওয়ার্ড লাগবে না
```

উল্টো ক্রমে লিখলে `%sudo` লাইন পরে মিলত, আর rahim-কে apt-এর জন্যও পাসওয়ার্ড দিতে হতো।

`/etc/sudoers.d/`-এর ফাইলগুলো `@includedir` লাইনের জায়গায় **নামের বর্ণানুক্রমে** (lexical order) পড়া হয়। তাই `10-admins` আগে, `90-overrides` পরে। এজন্য ফাইলের নামের আগে সংখ্যা দেওয়া একটা ভালো অভ্যাস।

---

## ১৮.৯ `visudo` — sudoers এডিটের একমাত্র নিরাপদ উপায়

> ⚠️ **সতর্কতা:** `/etc/sudoers` কখনো সরাসরি `nano /etc/sudoers` বা `vim` দিয়ে এডিট করবে না। একটা টাইপো করলে sudo পুরো ফাইলটা পড়তেই অস্বীকার করবে, আর **কেউই** আর sudo চালাতে পারবে না। Ubuntu-তে root লক থাকায় তুমি নিজেকে সিস্টেম থেকে পুরোপুরি তালাবদ্ধ করে ফেলবে।

ভুল sudoers হলে যা দেখবে:

```
$ sudo ls
>>> /etc/sudoers: syntax error near line 25 <<<
sudo: parse error in /etc/sudoers near line 25
sudo: no valid sudoers sources found, quitting
```

`visudo` এই বিপদ ঠেকায়:

```
 $ sudo visudo
      │
      ▼
 ┌──────────────────────────────────────────┐
 │ ১. /etc/sudoers লক করে (একসাথে দুজন       │
 │    এডিট করতে পারবে না)                    │
 │ ২. একটা অস্থায়ী কপি বানায় (/etc/sudoers.tmp)│
 │ ৩. এডিটরে খোলে                            │
 └───────────────────┬──────────────────────┘
                     ▼  তুমি সেভ করে বের হলে
 ┌──────────────────────────────────────────┐
 │ ৪. সিনট্যাক্স যাচাই                        │
 │    ঠিক আছে → আসল ফাইলে বসিয়ে দেয়         │
 │    ভুল → "What now?" জিজ্ঞেস করে           │
 └──────────────────────────────────────────┘
```

ভুল থাকলে:

```
$ sudo visudo
/etc/sudoers:25:20: syntax error
karim ALL=(root) systemctl restart nginx
                   ^~~~~~~~~
What now?
```

এখানে তিনটা বিকল্প:

| কী চাপবে | মানে |
|----------|------|
| `e` | আবার এডিটরে ফিরে গিয়ে ঠিক করো (**এটাই সাধারণত করা উচিত**) |
| `x` | কিছু সেভ না করে বের হও, আগের ফাইল অক্ষত থাকে |
| `Q` | ভুলসহই সেভ করো (**বিপজ্জনক, কখনো চাপবে না**) |

উপরের ভুলটা হলো `systemctl`-এর পূর্ণ path না দেওয়া।

### `visudo`-র অপশন

| কমান্ড | কাজ |
|--------|------|
| `sudo visudo` | `/etc/sudoers` এডিট |
| `sudo visudo -f /etc/sudoers.d/webteam` | নির্দিষ্ট ফাইল এডিট (নতুন হলে তৈরি) |
| `sudo visudo -c` | সব sudoers ফাইলের সিনট্যাক্স শুধু যাচাই করো, এডিট না |
| `sudo visudo -cf /path/file` | একটা নির্দিষ্ট ফাইল যাচাই করো |
| `sudo visudo -s` | কঠোর (strict) যাচাই, যেমন অব্যবহৃত বা অসংজ্ঞায়িত Alias ধরা |

```
$ sudo visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/webteam: parsed OK
```

### কোন এডিটর খুলবে?

Ubuntu/Debian-এ `visudo` সাধারণত **nano** খোলে (ডিফল্ট `editor` বিকল্প অনুযায়ী)। Fedora/RHEL-এ খোলে **vi**। বদলাতে:

```
$ sudo EDITOR=nano visudo           # একবারের জন্য
$ sudo update-alternatives --config editor   # Debian/Ubuntu-তে সিস্টেম ডিফল্ট বদলাও
```

nano-র শর্টকাট [অধ্যায় ১৩](13-nano-editor.md)-এ, vim-এর [অধ্যায় ১৪](14-vim.md)-এ।

> 💡 **টিপস:** sudoers বদলানোর সময় একটা **আলাদা টার্মিনালে** আগে থেকে `sudo -i` দিয়ে root shell খোলা রাখো। কিছু ভুল হলে সেই shell থেকে ঠিক করতে পারবে। নতুন নিয়ম কাজ করছে কি না অন্য টার্মিনাল থেকে পরীক্ষা করো, তারপরই root shell বন্ধ করো।

### সব শেষ হয়ে গেলে: তালাবদ্ধ হয়ে গেলে কী করবে

যদি sudoers ভেঙে যায় আর root shell খোলা না থাকে:
1. **Ubuntu desktop-এ:** `pkexec visudo` চেষ্টা করো (polkit sudo-র উপর নির্ভর করে না, নিচে দেখো)।
2. **Recovery mode:** রিবুট করে GRUB মেনুতে "Advanced options → recovery mode → root" নাও, তারপর `mount -o remount,rw /` আর `visudo`।
3. **Live USB:** লাইভ ইউএসবি থেকে বুট করে ডিস্ক মাউন্ট করে ফাইল ঠিক করো ([অধ্যায় ৩২](32-disk-partition-mount.md))।
4. **ক্লাউড VM:** প্রোভাইডারের serial console বা রেসকিউ মোড।

বুট প্রসেস আর recovery নিয়ে [অধ্যায় ২৭](27-boot-process.md)-এ বিস্তারিত আছে।

---

## ১৮.১০ `/etc/sudoers.d/` — নিজের নিয়ম আলাদা ফাইলে

মূল `/etc/sudoers` ফাইল প্যাকেজ আপডেটের সময় বদলাতে পারে (Debian/Ubuntu তখন জিজ্ঞেস করে কোনটা রাখবে)। নিজের নিয়ম এতে লিখলে conflict হয়। সমাধান: `/etc/sudoers.d/`-এ আলাদা ছোট ফাইল।

সুবিধা:
- প্রতিটি দলের জন্য আলাদা ফাইল, সহজে যোগ/মোছা যায়
- Ansible, Puppet-এর মতো টুল একটা ফাইল ফেলে দিলেই হয়
- মূল ফাইল অপরিবর্তিত থাকে

```
$ sudo visudo -f /etc/sudoers.d/10-webteam
```

ভেতরে লেখো:

```
# Web team: nginx নিয়ন্ত্রণ
Cmnd_Alias NGINXCTL = /usr/bin/systemctl restart nginx, \
                      /usr/bin/systemctl reload nginx, \
                      /usr/bin/systemctl status nginx, \
                      /usr/sbin/nginx -t
%webteam ALL=(root) NGINXCTL
```

### `sudoers.d`-এর অলিখিত নিয়ম (এগুলো না জানলে ফাইল নিঃশব্দে উপেক্ষিত হয়)

| নিয়ম | বিস্তারিত |
|-------|-----------|
| ফাইলের নামে `.` থাকা চলবে না | `webteam.conf` **পড়া হবে না**! `webteam` বা `10-webteam` লেখো |
| `~` দিয়ে শেষ হলে উপেক্ষিত | `webteam~` (এডিটরের ব্যাকআপ) পড়া হয় না |
| পারমিশন `0440` | `visudo -f` এটা নিজেই ঠিক রাখে। হাতে তৈরি করলে `sudo chmod 0440 file` |
| মালিক root | `root:root` |
| বর্ণানুক্রমে পড়া হয় | পরের ফাইলের নিয়ম আগের ফাইলকে override করতে পারে (শেষ মিল জেতে) |

> ⚠️ **সতর্কতা:** `.conf` এক্সটেনশন দেওয়ার ভুলটা খুব সাধারণ। তুমি নিয়ম লিখলে, `visudo -c` হয়তো ঠিক বলল, কিন্তু sudo সেটা কখনো পড়ছেই না। `sudo -l -U ইউজার` দিয়ে সবসময় যাচাই করো নিয়মটা সত্যিই কার্যকর হয়েছে কি না।

স্ক্রিপ্ট থেকে নিরাপদে ফাইল তৈরি করার প্যাটার্ন:

```
$ echo 'backup ALL=(root) NOPASSWD: /usr/bin/rsync' > /tmp/backup-sudo
$ sudo visudo -cf /tmp/backup-sudo && \
  sudo install -m 0440 -o root -g root /tmp/backup-sudo /etc/sudoers.d/20-backup
/tmp/backup-sudo: parsed OK
```

আগে যাচাই, তারপর ঠিক পারমিশনসহ বসানো।

---

## ১৮.১১ `NOPASSWD` — পাসওয়ার্ড ছাড়া sudo

কিছু ক্ষেত্রে পাসওয়ার্ড চাওয়া অসম্ভব বা অর্থহীন:
- cron job বা স্ক্রিপ্ট যা রাতে নিজে নিজে চলে
- CI/CD পাইপলাইন (deploy ইউজার)
- ক্লাউড VM যেখানে লগইন শুধু SSH key দিয়ে, ইউজারের কোনো পাসওয়ার্ডই নেই (যেমন AWS-এর `ubuntu` ইউজার, যা `/etc/sudoers.d/90-cloud-init-users`-এ `NOPASSWD:ALL` পায়)

```
# ভালো: সীমিত কমান্ডে NOPASSWD
deploy  ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp

# ঝুঁকিপূর্ণ: সবকিছুতে NOPASSWD
deploy  ALL=(ALL) NOPASSWD: ALL
```

> ⚠️ **সতর্কতা:** `NOPASSWD: ALL` মানে ওই ইউজারের অ্যাকাউন্ট যে দখল করবে (চুরি যাওয়া SSH key, ব্রাউজারের ত্রুটি, ক্ষতিকর npm প্যাকেজ), সে কোনো বাধা ছাড়াই root। পাসওয়ার্ড হলো শেষ একটা দেয়াল। পারলে NOPASSWD শুধু নির্দিষ্ট কমান্ডে দাও।

---

## ১৮.১২ নির্দিষ্ট কমান্ডের অনুমতি, এবং তার ফাঁদ

"karim শুধু লগ দেখতে পারবে" শুনতে সহজ। কিন্তু এখানেই সবচেয়ে বেশি নিরাপত্তা ছিদ্র তৈরি হয়। কারণ অনেক প্রোগ্রামের ভেতর থেকে **shell escape** (shell চালু করা) সম্ভব।

### ফাঁদ ১: যেসব প্রোগ্রাম shell দেয়

```
karim ALL=(root) /usr/bin/less /var/log/syslog
```

নিরীহ মনে হয়। কিন্তু `less`-এর ভেতরে `!bash` টাইপ করলে একটা shell খোলে, আর সেটা root! একই সমস্যা:

| প্রোগ্রাম | কীভাবে root shell পাওয়া যায় |
|-----------|-------------------------------|
| `less`, `more`, `man` | `!sh` |
| `vi`, `vim` | `:!sh` বা `:shell` |
| `find` | `-exec /bin/sh \;` |
| `awk` | `'BEGIN {system("/bin/sh")}'` |
| `tar` | `--checkpoint-action=exec=/bin/sh` |
| `python`, `perl`, `ruby` | সরাসরি কোড চালানো |
| `cp`, `tee`, `dd` | যেকোনো ফাইলে লেখা, যেমন `/etc/sudoers` বা `/etc/shadow` ওভাররাইট |
| `chmod`, `chown` | যেকোনো ফাইলের পারমিশন বদলানো |
| `apt`, `dnf` | ক্ষতিকর প্যাকেজ ইনস্টল বা hook চালানো |
| `systemctl` (কোনো সীমা ছাড়া) | নতুন service ফাইল দিয়ে যেকোনো কমান্ড |
| `bash`, `sh`, `su` | সরাসরি shell |

এই ধরনের প্রোগ্রামের একটা বিখ্যাত তালিকা হলো **GTFOBins** (gtfobins.github.io)। sudo-তে কোনো কমান্ড দেওয়ার আগে সেখানে দেখে নেওয়া ভালো অভ্যাস।

**নিরাপদ বিকল্প:**
- লগ পড়ার জন্য: ইউজারকে `adm` (Debian/Ubuntu) গ্রুপে দাও, তাহলে sudo ছাড়াই `/var/log/syslog` পড়তে পারবে। `systemd-journal` গ্রুপে দিলে পুরো journal পড়তে পারবে।
- ফাইল এডিটের জন্য: `sudoedit` অনুমতি দাও (`karim ALL=(root) sudoedit /etc/nginx/nginx.conf`)।
- `NOEXEC:` Tag ব্যবহার করো, যা কমান্ডকে আরেকটা প্রোগ্রাম exec করতে বাধা দেয় (`karim ALL=(root) NOEXEC: /usr/bin/less /var/log/syslog`)। তবে এটা শুধু dynamically linked প্রোগ্রামে কাজ করে আর শতভাগ নিশ্চয়তা দেয় না।

### ফাঁদ ২: wildcard

```
karim ALL=(root) /usr/bin/cat /var/log/*
```

karim চালাতে পারে: `sudo cat /var/log/../../etc/shadow`। `*` path-এর `/` এবং `..`-ও মিলিয়ে নেয়। আরও: `sudo cat /var/log/syslog /etc/shadow`, কারণ `*` স্পেসসহ বাকি সব আর্গুমেন্ট মেলায়।

### ফাঁদ ৩: negation (`!`) দিয়ে নিষেধ

```
rahim ALL=(ALL) ALL, !/usr/bin/su, !/bin/bash
```

এটা কাজ করে না। rahim `cp /bin/bash /tmp/mybash` করে `sudo /tmp/mybash` চালাবে। "সব দাও, শুধু এগুলো বাদ" ধরনের নিয়ম মৌলিকভাবে ভাঙা যায়। সবসময় উল্টোটা করো: **শুধু যা দরকার তা-ই দাও** (allowlist)।

### ফাঁদ ৪: ইউজার-লেখযোগ্য স্ক্রিপ্ট

```
deploy ALL=(root) NOPASSWD: /home/deploy/restart.sh
```

`deploy` ইউজার নিজেই `restart.sh`-এর ভেতরটা বদলাতে পারে, তাই কার্যত সে root। sudo-তে দেওয়া যেকোনো স্ক্রিপ্ট (এবং তার ডিরেক্টরি) **root-এর মালিকানায়** আর অন্যদের জন্য লেখার অযোগ্য হতে হবে, যেমন `/usr/local/sbin/restart-myapp` মোড `0755`, মালিক `root:root`।

### একটা ভালো নির্দিষ্ট-কমান্ড উদাহরণ

```
# /etc/sudoers.d/30-support
Cmnd_Alias SUPPORT_CMDS = /usr/bin/systemctl status *, \
                          /usr/bin/systemctl restart myapp, \
                          /usr/bin/journalctl -u myapp *
%support ALL=(root) SUPPORT_CMDS
```

এখানেও `journalctl` ডিফল্টে pager হিসেবে `less` খোলে, আর `systemctl status`-ও। sudo-র অধীনে আধুনিক systemd "secure mode"-এ less চালায় (shell escape বন্ধ), তবুও সতর্কতার জন্য `--no-pager` যোগ করা ভালো। আর `systemctl status *`-এর wildcard বাড়তি আর্গুমেন্ট মিলায়, তাই নীতির দিক থেকে যত নির্দিষ্ট সম্ভব লেখো। নিরাপত্তা হলো খুঁটিনাটির খেলা।

---

## ১৮.১৩ `sudo -l` — আমি কী কী পারি?

```
$ sudo -l
[sudo] password for karim:
Matching Defaults entries for karim on web01:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User karim may run the following commands on web01:
    (root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl reload nginx
```

পূর্ণ অ্যাডমিনের ক্ষেত্রে:

```
User rahim may run the following commands on web01:
    (ALL : ALL) ALL
```

কোনো অনুমতি না থাকলে:

```
Sorry, user guest may not run sudo on web01.
```

আরও কিছু রূপ:

```
$ sudo -ll                                   # লম্বা, বিস্তারিত ফরম্যাট
$ sudo -l /usr/bin/systemctl restart nginx   # নির্দিষ্ট কমান্ড অনুমোদিত?
/usr/bin/systemctl restart nginx
$ sudo -l -U karim                           # অ্যাডমিন হিসেবে অন্যের অনুমতি দেখো
```

নির্দিষ্ট কমান্ড অনুমোদিত হলে তার পূর্ণ path প্রিন্ট করে আর exit status 0 দেয়, না হলে কিছু প্রিন্ট না করে exit status 1।

> 💡 **টিপস:** নতুন সার্ভারে লগইন করে সবার আগে `sudo -l` চালাও, জেনে নাও তোমার কতটুকু ক্ষমতা আছে। নিরাপত্তা পরীক্ষকরাও (pentester) এটা সবার আগে চালায়, কারণ ভুল sudoers হলো privilege escalation-এর সবচেয়ে সাধারণ পথ।

---

## ১৮.১৪ timestamp timeout — পাসওয়ার্ড কতক্ষণ মনে থাকে

একবার পাসওয়ার্ড দিলে sudo কিছু সময়ের জন্য আর পাসওয়ার্ড চায় না। ডিফল্ট **১৫ মিনিট** (Ubuntu, Debian, Fedora, RHEL সবখানে, যদি না distro বদলে দেয়)।

**ভেতরে কীভাবে:** sudo একটা timestamp রেকর্ড রাখে `/run/sudo/ts/<username>` ফাইলে (কিছু পুরনো সিস্টেমে `/var/run/sudo/ts/` বা `/var/lib/sudo/ts/`)। `/run` হলো tmpfs (RAM-এ), তাই রিবুটে মুছে যায়।

ডিফল্টে timestamp প্রতিটি **terminal (tty) অনুযায়ী আলাদা** (`timestamp_type=tty`)। মানে একটা টার্মিনালে পাসওয়ার্ড দিলে পাশের নতুন টার্মিনাল ট্যাবে আবার চাইবে। এটা ইচ্ছাকৃত: একটা টার্মিনালে ক্ষতিকর কোনো প্রক্রিয়া চলতে থাকলে সে তোমার অন্য টার্মিনালের "খোলা দরজা" ব্যবহার করতে পারবে না।

```
  টার্মিনাল ১ (pts/0)            টার্মিনাল ২ (pts/1)
  $ sudo apt update              $ sudo ls /root
  [sudo] password: ****          [sudo] password:    ← আবার চাইছে, আলাদা tty
  ...
  $ sudo systemctl restart x     
  (চাইল না, ১৫ মিনিটের মধ্যে)
```

### বদলানো

```
Defaults timestamp_timeout=5        # ৫ মিনিট
Defaults timestamp_timeout=0        # প্রতিবার পাসওয়ার্ড চাও (সবচেয়ে কড়া)
Defaults timestamp_timeout=-1       # কখনো মেয়াদ শেষ হবে না (সুপারিশ করা হয় না)
Defaults:rahim timestamp_timeout=30 # শুধু rahim-এর জন্য
Defaults timestamp_type=global      # সব terminal-এ একটাই timestamp
```

হাতে নিয়ন্ত্রণ:

| কমান্ড | কাজ |
|--------|------|
| `sudo -v` | পাসওয়ার্ড দিয়ে timestamp নবায়ন (আবার ১৫ মিনিট) |
| `sudo -k` | এই session-এর timestamp বাতিল |
| `sudo -K` | এই ইউজারের সব timestamp মুছে ফেলো |
| `sudo -n true` | পাসওয়ার্ড ছাড়া এখনই sudo চলবে কি না যাচাই (exit status দিয়ে) |

> 💡 **টিপস:** কম্পিউটার রেখে চা খেতে উঠলে `sudo -k` চালাও (আর স্ক্রিন লক করো: GNOME-এ `Super+L`)। লম্বা ইনস্টল স্ক্রিপ্ট চালানোর আগে `sudo -v` দিয়ে রাখলে মাঝপথে পাসওয়ার্ড চেয়ে আটকে থাকবে না।

---

## ১৮.১৫ sudo লগ — কে কী করেছে

sudo প্রতিটি সফল ও ব্যর্থ চেষ্টা syslog-এ পাঠায় (`authpriv` facility)। কোথায় জমা হয়:

| Distro | লগ ফাইল |
|--------|---------|
| Debian, Ubuntu | `/var/log/auth.log` |
| Fedora, RHEL, Rocky | `/var/log/secure` |
| সব systemd সিস্টেম | systemd journal (`journalctl`) |

> 📝 **মনে রাখো:** Fedora-র নতুন ভার্সনে ডিফল্টে rsyslog ইনস্টল নাও থাকতে পারে, তখন `/var/log/secure` থাকবে না, শুধু journal। লগ ব্যবস্থা নিয়ে পুরোটা [অধ্যায় ২৮](28-logs.md)-এ।

### লগ দেখা

```
$ sudo grep sudo /var/log/auth.log | tail -3
2026-11-12T10:15:01.482913+06:00 web01 sudo:    rahim : TTY=pts/0 ; PWD=/home/rahim ; USER=root ; COMMAND=/usr/bin/apt update
2026-11-12T10:15:01.486120+06:00 web01 sudo: pam_unix(sudo:session): session opened for user root(uid=0) by rahim(uid=1000)
2026-11-12T10:15:09.031277+06:00 web01 sudo: pam_unix(sudo:session): session closed for user root
```

(Ubuntu 24.04 থেকে `auth.log`-এ এই ISO-8601 সময় ফরম্যাট, পুরনো সংস্করণে `Nov 12 10:15:01` ধরনের ছিল।)

একটা লাইন ভেঙে দেখি:

| অংশ | মানে |
|-----|------|
| `rahim` | আসল ইউজার, যে sudo চালিয়েছে |
| `TTY=pts/0` | কোন terminal থেকে |
| `PWD=/home/rahim` | তখন কোন ডিরেক্টরিতে ছিল |
| `USER=root` | কার হিসেবে কমান্ড চলেছে |
| `COMMAND=/usr/bin/apt update` | ঠিক কোন কমান্ড, পূর্ণ path সহ |

ব্যর্থ চেষ্টা:

```
2026-11-12T10:20:44.112233+06:00 web01 sudo:    karim : user NOT in sudoers ; TTY=pts/1 ; PWD=/home/karim ; USER=root ; COMMAND=/usr/bin/cat /etc/shadow
2026-11-12T10:21:02.554411+06:00 web01 sudo:    karim : 3 incorrect password attempts ; TTY=pts/1 ; PWD=/home/karim ; USER=root ; COMMAND=/usr/bin/apt update
2026-11-12T10:22:13.998877+06:00 web01 sudo:    karim : command not allowed ; TTY=pts/1 ; PWD=/home/karim ; USER=root ; COMMAND=/usr/bin/bash
```

journal থেকে:

```
$ journalctl -t sudo --since today          # sudo-র নিজের বার্তা
$ journalctl _COMM=sudo -n 20               # sudo process থেকে আসা শেষ ২০টি
$ journalctl -t sudo -g "NOT in sudoers"    # নির্দিষ্ট প্যাটার্ন খোঁজা (-g = grep)
```

### পুরো session রেকর্ড: `log_output` ও `sudoreplay`

শুধু কমান্ড না, স্ক্রিনে কী হয়েছিল তাও রেকর্ড করা যায় (অডিট-কঠোর পরিবেশে, যেমন ব্যাংকিং):

```
Defaults log_input, log_output
Defaults iolog_dir=/var/log/sudo-io
```

তারপর:

```
$ sudo sudoreplay -l                 # রেকর্ড করা session-এর তালিকা
$ sudo sudoreplay 000001             # ভিডিওর মতো ঠিক আগের মতো আবার দেখো
```

> ⚠️ **সতর্কতা:** `log_input` চালু করলে টাইপ করা **পাসওয়ার্ডও** (যেমন sudo session-এর ভেতর `mysql -p`) রেকর্ড হতে পারে। সংবেদনশীল তথ্যের লগ সাবধানে সুরক্ষিত রাখতে হবে।

মনে রাখো: `sudo -i` বা `sudo -s` দিয়ে root shell নিলে, লগে শুধু লেখা থাকবে `COMMAND=/bin/bash`। এর ভেতরে কী কী চালানো হয়েছে sudo-র সাধারণ লগে আসবে না (log_output না থাকলে)। এটা root shell এড়িয়ে আলাদা আলাদা `sudo command` চালানোর আরেকটা কারণ।

---

## ১৮.১৬ environment ও sudo: কেন আমার `PATH` কাজ করছে না?

```
$ ls ~/bin
mytool
$ mytool
Hello!
$ sudo mytool
sudo: mytool: command not found
```

কারণ sudo তোমার `PATH` না নিয়ে `secure_path` ব্যবহার করে। এটা ইচ্ছাকৃত নিরাপত্তা: কেউ তোমার `PATH`-এর শুরুতে `/tmp/evil` ঢুকিয়ে `ls` নামে ম্যালওয়্যার রাখলে, sudo সেটা চালাত না।

সমাধান:

```
$ sudo ~/bin/mytool                  # পূর্ণ path দাও
$ sudo "$(command -v mytool)"        # path খুঁজে দাও
```

একইভাবে `env_reset`-এর কারণে বেশির ভাগ চলক মুছে যায়। প্রক্সির মতো কিছু রাখতে হলে:

```
$ sudo --preserve-env=http_proxy,https_proxy apt update
```

অথবা sudoers-এ `Defaults env_keep += "http_proxy https_proxy no_proxy"`। পুরো environment রাখার `sudo -E` সম্ভব হলে এড়িয়ে চলো, ক্ষতিকর চলক (যেমন `LD_PRELOAD` বাদে অন্য অনেক কিছু) ঢুকে যেতে পারে। (sudo নিজে কিছু বিপজ্জনক চলক যেমন `LD_PRELOAD` সবসময় সরিয়ে ফেলে।)

sudo কমান্ডের ভেতর কিছু চলক নিজে থেকেই সেট হয়, স্ক্রিপ্টে কাজের:

| চলক | মান |
|-----|-----|
| `SUDO_USER` | আসল ইউজারের নাম |
| `SUDO_UID`, `SUDO_GID` | আসল ইউজারের UID/GID |
| `SUDO_COMMAND` | যে কমান্ড চালানো হয়েছে |

```
$ sudo bash -c 'echo "আসলে চালিয়েছে: $SUDO_USER, এখন আমি: $(whoami)"'
আসলে চালিয়েছে: rahim, এখন আমি: root
```

---

## ১৮.১৭ polkit ও `pkexec` — GUI-র প্রশাসনিক ক্ষমতা

desktop-এ সফটওয়্যার আপডেট, Wi-Fi কনফিগ বা তারিখ-সময় বদলাতে গেলে একটা ছোট উইন্ডো এসে পাসওয়ার্ড চায়। এর পেছনে sudo নেই, আছে **polkit** (আগের নাম PolicyKit)।

### কেন আলাদা ব্যবস্থা?

sudo-র মডেল: "পুরো একটা প্রোগ্রাম root হিসেবে চালাও"। GUI-তে এটা খারাপ, কারণ পুরো Settings অ্যাপকে root বানালে বিশাল কোড root ক্ষমতা পায়। polkit-এর মডেল ভিন্ন: GUI অ্যাপ সাধারণ ইউজার হিসেবেই চলে। সে একটা ছোট, বিশেষায়িত root **service** (যেমন `systemd-timedated`, `NetworkManager`, `packagekitd`)-কে D-Bus দিয়ে অনুরোধ পাঠায়: "সময় বদলাও"। service জিজ্ঞেস করে polkit-কে: "এই ইউজার কি `org.freedesktop.timedate1.set-time` কাজটা করতে পারবে?"

```
 ┌───────────────┐   D-Bus অনুরোধ     ┌──────────────────────┐
 │ Settings অ্যাপ │ ────────────────▶ │ systemd-timedated    │
 │ (ইউজার rahim)  │  "সময় বদলাও"      │ (root হিসেবে চলে)     │
 └───────────────┘                    └──────────┬───────────┘
                                                 │ "rahim কি পারবে?"
                                                 ▼
                                      ┌──────────────────────┐
                                      │ polkitd              │
                                      │ নিয়ম পড়ে:            │
                                      │ /usr/share/polkit-1/  │
                                      │   actions/*.policy   │
                                      │ /etc/polkit-1/       │
                                      │   rules.d/*.rules    │
                                      └──────────┬───────────┘
                                                 │ "পাসওয়ার্ড লাগবে"
                                                 ▼
                                      ┌──────────────────────┐
                                      │ Authentication Agent │
                                      │ (GNOME/KDE-এর পপআপ)   │
                                      │ পাসওয়ার্ড চায়        │
                                      └──────────────────────┘
```

polkit-এর মূল অংশ:

| অংশ | কাজ |
|-----|------|
| **Action** | একটা নির্দিষ্ট কাজের নাম, যেমন `org.freedesktop.login1.reboot`। সংজ্ঞা `/usr/share/polkit-1/actions/*.policy` (XML) ফাইলে |
| **Rules** | কে কোন action করতে পারবে, JavaScript-এ লেখা। থাকে `/etc/polkit-1/rules.d/` (অ্যাডমিনের) আর `/usr/share/polkit-1/rules.d/` (প্যাকেজের) |
| **polkitd** | daemon যে সিদ্ধান্ত নেয় |
| **Authentication agent** | পাসওয়ার্ড চাওয়ার পপআপ (GUI) বা টেক্সট এজেন্ট |

উত্তর তিন রকম হতে পারে: `yes` (সরাসরি অনুমতি), `no` (নিষেধ), `auth_self` (নিজের পাসওয়ার্ড দাও) বা `auth_admin` (অ্যাডমিনের পাসওয়ার্ড লাগবে)। এগুলোর `_keep` সংস্করণও আছে, যা কিছুক্ষণ মনে রাখে।

polkit "অ্যাডমিন" কাকে ধরে? Ubuntu-তে `sudo` গ্রুপ, Fedora/RHEL-এ `wheel` গ্রুপের সদস্যকে।

### একটা polkit নিয়ম উদাহরণ

`devops` গ্রুপকে পাসওয়ার্ড ছাড়া service রিস্টার্ট করতে দেওয়া:

```
// /etc/polkit-1/rules.d/50-devops-units.rules
polkit.addRule(function(action, subject) {
    if (action.id == "org.freedesktop.systemd1.manage-units" &&
        subject.isInGroup("devops")) {
        return polkit.Result.YES;
    }
});
```

(Debian 12-এর polkit 122 সংস্করণে এই JavaScript `.rules` ফরম্যাট সমর্থিত। খুব পুরনো Debian/Ubuntu-তে `.pkla` ফরম্যাট ছিল।)

### polkit কমান্ড

| কমান্ড | কাজ |
|--------|------|
| `pkexec command` | polkit দিয়ে যাচাই করে command root হিসেবে চালাও |
| `pkexec --user karim command` | karim হিসেবে চালাও |
| `pkaction` | সব নিবন্ধিত action-এর তালিকা |
| `pkaction --action-id ID --verbose` | একটা action-এর বিস্তারিত (কে পারবে, ডিফল্ট কী) |
| `pkcheck` | একটা process কোনো action-এর অনুমতি পায় কি না যাচাই |
| `pkttyagent` | টেক্সট টার্মিনালের জন্য authentication agent |

```
$ pkaction --action-id org.freedesktop.login1.reboot --verbose
org.freedesktop.login1.reboot:
  description:       Reboot the system
  message:           Authentication is required to reboot the system.
  vendor:            The systemd Project
  vendor_url:        https://systemd.io
  icon:
  implicit any:      auth_admin_keep
  implicit inactive: auth_admin_keep
  implicit active:   yes
```

`implicit active: yes` মানে যে ইউজার সামনে বসে লগইন করা (active session), সে পাসওয়ার্ড ছাড়াই রিবুট করতে পারে। SSH দিয়ে ঢুকলে (any) অ্যাডমিন পাসওয়ার্ড লাগবে।

### `pkexec`

```
$ pkexec visudo
==== AUTHENTICATING FOR org.freedesktop.policykit.exec ====
Authentication is needed to run `/usr/sbin/visudo' as the super user
Authenticating as: Rahim Uddin,,, (rahim)
Password:
==== AUTHENTICATION COMPLETE ====
```

desktop-এ থাকলে পপআপ আসবে। `pkexec`-ও sudo-র মতো setuid root প্রোগ্রাম, কিন্তু sudoers পড়ে না, polkit পড়ে। এজন্য sudoers ভেঙে গেলে এটা উদ্ধারের উপায় হতে পারে।

> ⚠️ **সতর্কতা:** ২০২২ সালে `pkexec`-এ "PwnKit" (CVE-2021-4034) নামে একটা গুরুতর ত্রুটি পাওয়া গিয়েছিল, যা দিয়ে যেকোনো সাধারণ ইউজার root হতে পারত। ১২ বছর ধরে ত্রুটিটা ছিল। এখন সব distro-তে ঠিক করা, কিন্তু এটা দেখায় setuid প্রোগ্রাম কতটা স্পর্শকাতর। সার্ভারে GUI না থাকলে অনেকে polkit প্যাকেজই রাখে না বা `pkexec`-এর setuid বিট সরিয়ে দেয়। সিস্টেম আপডেট রাখো ([অধ্যায় ৩০](30-package-management.md))।

### আরও কিছু বিকল্প (জেনে রাখো)

- **`doas`:** OpenBSD থেকে আসা খুব ছোট sudo-বিকল্প। কনফিগ `/etc/doas.conf`, যেমন `permit persist :wheel`। Debian/Ubuntu-তে `opendoas` প্যাকেজ হিসেবে পাওয়া যায়, ডিফল্ট নয়।
- **`run0`:** systemd 256 থেকে আসা নতুন টুল, setuid ছাড়াই polkit দিয়ে কাজ করে (Fedora 40+ এ পাওয়া যায়, Ubuntu 24.04-এর systemd 255-এ নেই)।
- **`sudo-rs`:** Rust-এ নতুন করে লেখা sudo, মেমরি-নিরাপদ। Ubuntu 25.10 থেকে ডিফল্ট হিসেবে আসছে। ব্যবহারকারীর জন্য কমান্ড প্রায় একই।

---

## ১৮.১৮ Least Privilege — ন্যূনতম ক্ষমতার নীতি

**Principle of Least Privilege (PoLP):** প্রতিটি ইউজার, প্রোগ্রাম আর process ঠিক ততটুকু ক্ষমতা পাবে যতটুকু তার কাজের জন্য দরকার, তার এক বিন্দু বেশি নয়, আর ঠিক যতক্ষণ দরকার ততক্ষণই।

উপমা: হোটেলের কী-কার্ড। অতিথির কার্ড শুধু তার রুম আর জিম খোলে, তাও চেক-আউট পর্যন্ত। ক্লিনারের কার্ড তার ফ্লোরের রুমগুলো খোলে, শুধু শিফটের সময়। কারো কার্ডই ভল্ট খোলে না, শুধু ম্যানেজারেরটা খোলে।

### বাস্তবে কীভাবে মানবে

| ক্ষেত্র | ভুল পদ্ধতি | সঠিক পদ্ধতি |
|---------|------------|-------------|
| দৈনন্দিন কাজ | root হয়ে লগইন | সাধারণ ইউজার, দরকারে `sudo command` |
| দলের সদস্য | সবাইকে `sudo` গ্রুপে | নির্দিষ্ট কাজের জন্য `Cmnd_Alias` |
| লগ পড়া | sudo দিয়ে `less` | `adm` বা `systemd-journal` গ্রুপ |
| ওয়েব অ্যাপ | root হিসেবে চলে | নিজস্ব ইউজার (`www-data`), systemd-এর `User=` |
| পোর্ট ৮০ | অ্যাপ root হিসেবে | `CAP_NET_BIND_SERVICE` বা reverse proxy (nginx) |
| deploy স্ক্রিপ্ট | `NOPASSWD: ALL` | `NOPASSWD:` শুধু নির্দিষ্ট root-মালিকানাধীন স্ক্রিপ্টে |
| SSH | root লগইন চালু | `PermitRootLogin no`, key দিয়ে সাধারণ ইউজার ([অধ্যায় ৩৮](38-ssh-remote.md)) |
| ফাইল এডিট | `sudo vim` | `sudoedit` |
| অস্থায়ী অধিকার | গ্রুপে যোগ করে ভুলে যাওয়া | কাজ শেষে সরানো, নিয়মিত অডিট |
| root shell | সারাদিন খোলা | কাজ শেষে সাথে সাথে `exit` |

### নিয়মিত অডিট চেকলিস্ট

```
$ getent group sudo wheel                # কারা অ্যাডমিন গ্রুপে আছে
$ awk -F: '$3==0 {print $1}' /etc/passwd # UID 0 আর কে আছে (শুধু root থাকা উচিত)
$ sudo grep -rn NOPASSWD /etc/sudoers /etc/sudoers.d/   # পাসওয়ার্ডহীন নিয়ম
$ sudo visudo -c                          # সব ফাইল ঠিক আছে?
$ ls -l /etc/sudoers.d/                   # অচেনা ফাইল আছে কি?
$ sudo find / -xdev -perm -4000 -type f 2>/dev/null   # setuid প্রোগ্রামের তালিকা
```

পুরো নিরাপত্তা চেকলিস্ট আর হার্ডেনিং [অধ্যায় ৪৫](45-security-hardening.md)-এ।

---

## ১৮.১৯ কীবোর্ড শর্টকাট ও দ্রুত কৌশল

| শর্টকাট / কৌশল | কাজ |
|----------------|------|
| `sudo !!` | আগের কমান্ডটা sudo দিয়ে আবার চালাও |
| `Ctrl+A` তারপর `sudo ` টাইপ | বর্তমান লাইনের শুরুতে গিয়ে sudo যোগ করো |
| `Alt+.` (বা `Esc` তারপর `.`) | আগের কমান্ডের শেষ আর্গুমেন্ট বসাও, যেমন `sudo nano ` + `Alt+.` |
| `Ctrl+D` বা `exit` | root shell (বা `su`) থেকে বের হও |
| `Ctrl+C` (পাসওয়ার্ড প্রম্পটে) | sudo বাতিল করো |
| `Ctrl+U` (পাসওয়ার্ড প্রম্পটে) | এ পর্যন্ত টাইপ করা পাসওয়ার্ড মুছে নতুন করে শুরু করো |
| `Esc` `Esc` (zsh + Oh My Zsh-এর `sudo` প্লাগইন) | লাইনের শুরুতে `sudo` যোগ/বাদ |
| `Super+L` (GNOME), `Super+L` বা `Ctrl+Alt+L` (KDE) | উঠে যাওয়ার আগে স্ক্রিন লক করো |
| `visudo`-তে `e` / `x` / `Q` | ভুল হলে: আবার এডিট / সেভ না করে বের হও / (বিপজ্জনক) জোর করে সেভ |
| nano-তে (visudo): `Ctrl+O` তারপর `Enter`, `Ctrl+X` | সেভ, বের হওয়া |
| vi-তে (visudo): `Esc` `:wq` `Enter` / `:q!` | সেভ করে বের হওয়া / সেভ না করে বের হওয়া |

> 💡 **টিপস:** bash-এ নিজের "লাইনের শুরুতে sudo" শর্টকাট বানাতে পারো। `~/.bashrc`-এ যোগ করো: `bind '"\e\e": "\C-asudo \C-e"'`। তারপর `Esc` দুবার চাপলে লাইনের শুরুতে `sudo ` বসে যাবে। (কিছু টার্মিনালে দুটো `Esc` অন্য কাজে লাগে, পরীক্ষা করে দেখো।) bash কাস্টমাইজেশন বিস্তারিত [অধ্যায় ৪৪](44-environment-bashrc.md)-এ।

---

## হাতে-কলমে অনুশীলন

> ⚠️ **সতর্কতা:** এই ল্যাব একটা **ভার্চুয়াল মেশিন** বা পরীক্ষামূলক সিস্টেমে করো ([অধ্যায় ৩](03-distro-o-installation.md))। শুরুর আগে একটা আলাদা টার্মিনালে `sudo -i` দিয়ে root shell খুলে রাখো, বিপদে কাজে লাগবে। উদাহরণ Ubuntu/Debian-এর, Fedora/RHEL-এ `sudo` গ্রুপের জায়গায় `wheel` ব্যবহার করো।

### ল্যাব ১: নিজের পরিচয় ও ক্ষমতা জানো

```
$ whoami
$ id
$ sudo -l
$ sudo whoami
root
$ sudo id
uid=0(root) gid=0(root) groups=0(root)
$ ls -l /usr/bin/sudo /etc/sudoers
```

খেয়াল করো sudo-তে `s` বিট আর sudoers-এর `0440` পারমিশন।

### ল্যাব ২: `su`, `sudo -s`, `sudo -i` তুলনা

```
$ cd /tmp
$ sudo -s
# pwd ; echo $HOME ; echo $SUDO_USER
# exit
$ sudo -i
# pwd ; echo $HOME ; echo $SUDO_USER
# exit
```

দুই ক্ষেত্রে `pwd`-এর ফল লিখে রাখো। এবার root-এর পাসওয়ার্ড লক আছে কি না দেখো:

```
$ sudo passwd -S root
$ su -
```

### ল্যাব ৩: পরীক্ষামূলক ইউজার তৈরি

```
$ sudo useradd -m -s /bin/bash tester
$ sudo passwd tester
New password:
Retype new password:
passwd: password updated successfully
$ su - tester
Password:
tester$ sudo ls /root
[sudo] password for tester:
tester is not in the sudoers file.
tester$ exit
```

লগে চেষ্টাটা খুঁজে দেখো:

```
$ sudo grep tester /var/log/auth.log | tail -2     # Fedora/RHEL: /var/log/secure
$ journalctl -t sudo -n 5
```

### ল্যাব ৪: নির্দিষ্ট কমান্ডের অনুমতি দাও

```
$ sudo visudo -f /etc/sudoers.d/50-tester
```

ভেতরে লেখো (path মিলিয়ে নিতে আগে `command -v systemctl` চালাও):

```
tester ALL=(root) /usr/bin/systemctl status cron, /usr/bin/systemctl restart cron
```

(Fedora/RHEL-এ service-এর নাম `crond`।) সেভ করে বের হও, তারপর:

```
$ sudo visudo -c
$ ls -l /etc/sudoers.d/50-tester
-r--r----- 1 root root 77 Nov 12 11:02 /etc/sudoers.d/50-tester
$ sudo -l -U tester
...
User tester may run the following commands on web01:
    (root) /usr/bin/systemctl status cron, /usr/bin/systemctl restart cron
```

tester হিসেবে পরীক্ষা করো:

```
$ su - tester
tester$ sudo systemctl restart cron        # কাজ করবে
tester$ sudo systemctl stop cron           # অনুমতি নেই
Sorry, user tester is not allowed to execute '/usr/bin/systemctl stop cron' as root on web01.
tester$ exit
```

### ল্যাব ৫: `.conf` ফাঁদ নিজের চোখে দেখো

```
$ sudo visudo -f /etc/sudoers.d/60-tester.conf
```

ভেতরে লেখো `tester ALL=(root) /usr/bin/whoami`, সেভ করো। তারপর:

```
$ sudo -l -U tester
```

`whoami` তালিকায় **নেই**, কারণ নামে `.` থাকায় ফাইলটা উপেক্ষিত। এবার নাম ঠিক করো:

```
$ sudo mv /etc/sudoers.d/60-tester.conf /etc/sudoers.d/60-tester
$ sudo -l -U tester
```

এখন আছে।

### ল্যাব ৬: ভুল সিনট্যাক্স ও visudo-র সুরক্ষা

```
$ sudo visudo -f /etc/sudoers.d/70-broken
```

ইচ্ছা করে ভুল লেখো: `tester ALL=(root) whoami` (পূর্ণ path ছাড়া)। সেভ করলে `What now?` আসবে। `x` চাপো। নিশ্চিত হও ফাইলটা তৈরি হয়নি:

```
$ ls /etc/sudoers.d/
```

### ল্যাব ৭: timestamp নিয়ে খেলা

```
$ sudo -k
$ sudo -n true && echo "পাসওয়ার্ড লাগবে না" || echo "পাসওয়ার্ড লাগবে"
পাসওয়ার্ড লাগবে
$ sudo -v
[sudo] password for rahim:
$ sudo -n true && echo "পাসওয়ার্ড লাগবে না" || echo "পাসওয়ার্ড লাগবে"
পাসওয়ার্ড লাগবে না
```

নতুন একটা টার্মিনাল ট্যাব খুলে আবার `sudo -n true` চালাও, tty আলাদা বলে পাসওয়ার্ড লাগবে।

এবার `/etc/sudoers.d/80-timeout` ফাইলে (visudo দিয়ে) লেখো `Defaults:tester timestamp_timeout=0`, তারপর tester হিসেবে পরপর দুবার `sudo systemctl status cron` চালাও। দুবারই পাসওয়ার্ড চাইবে।

### ল্যাব ৮: redirection ফাঁদ

```
$ sudo echo "test" > /etc/lab-test.txt
bash: /etc/lab-test.txt: Permission denied
$ echo "test" | sudo tee /etc/lab-test.txt
test
$ cat /etc/lab-test.txt
test
$ sudo rm /etc/lab-test.txt
```

### ল্যাব ৯: polkit অন্বেষণ

```
$ pkaction | wc -l
$ pkaction | grep -i reboot
$ pkaction --action-id org.freedesktop.login1.reboot --verbose
$ pkexec whoami
root
```

### ল্যাব ১০: পরিষ্কার করো

```
$ sudo rm /etc/sudoers.d/50-tester /etc/sudoers.d/60-tester /etc/sudoers.d/80-timeout
$ sudo visudo -c
$ sudo userdel -r tester
```

---

## সারসংক্ষেপ

| কমান্ড / ফাইল | কাজ |
|---------------|------|
| `id`, `whoami` | বর্তমান পরিচয় (UID 0 = root) |
| `su - [user]` | target-এর পাসওয়ার্ড দিয়ে পূর্ণ login shell-এ ইউজার বদল |
| `su -c "cmd"` | অন্য ইউজার হিসেবে একটা কমান্ড |
| `sudo cmd` | নিজের পাসওয়ার্ড দিয়ে root হিসেবে একটা কমান্ড |
| `sudo -u user cmd` | নির্দিষ্ট ইউজার হিসেবে |
| `sudo -i` | root-এর login shell (`/root`-এ) |
| `sudo -s` | root shell, বর্তমান ডিরেক্টরিতে |
| `sudo !!` | আগের কমান্ড sudo দিয়ে |
| `sudo -l`, `sudo -l -U user` | অনুমতির তালিকা |
| `sudo -v` / `-k` / `-K` | timestamp নবায়ন / বাতিল / সব মোছা |
| `sudo -n` | পাসওয়ার্ড না চেয়ে চেষ্টা (স্ক্রিপ্টে) |
| `sudoedit file` / `sudo -e` | নিরাপদে root-এর ফাইল এডিট |
| `cmd \| sudo tee file` | root-এর ফাইলে লেখা |
| `sudo visudo` | `/etc/sudoers` নিরাপদে এডিট |
| `sudo visudo -f /etc/sudoers.d/name` | ড্রপ-ইন ফাইল এডিট (নামে `.` নয়) |
| `sudo visudo -c` | সিনট্যাক্স যাচাই |
| `usermod -aG sudo/wheel user` | কাউকে অ্যাডমিন বানানো |
| `gpasswd -d user sudo` | অ্যাডমিন অধিকার সরানো |
| `user HOST=(RUNAS) [TAG:] CMDS` | sudoers নিয়মের গঠন |
| `NOPASSWD:` | পাসওয়ার্ড ছাড়া (সীমিত কমান্ডে) |
| `Defaults timestamp_timeout=N` | পাসওয়ার্ড মনে রাখার সময় (ডিফল্ট ১৫ মিনিট) |
| `/var/log/auth.log` / `/var/log/secure` | sudo লগ (Debian-পরিবার / RHEL-পরিবার) |
| `journalctl -t sudo` | journal থেকে sudo লগ |
| `sudoreplay` | রেকর্ড করা sudo session আবার দেখা |
| `pkexec cmd` | polkit দিয়ে root হিসেবে চালানো |
| `pkaction --action-id ID --verbose` | polkit action-এর নিয়ম দেখা |
| `/etc/polkit-1/rules.d/` | অ্যাডমিনের polkit নিয়ম |

---

## নিজেকে যাচাই করো

1. `su` আর `sudo` কার পাসওয়ার্ড চায়? এই পার্থক্য নিরাপত্তায় কী সুবিধা দেয়?
2. `su` আর `su -`-এর মধ্যে দুটো ব্যবহারিক পার্থক্য বলো।
3. `sudo -i` আর `sudo -s` চালানোর পর `pwd` আলাদা হয় কেন?
4. এই লাইনটা সহজ বাংলায় পড়ো: `%ops web01=(postgres) NOPASSWD: /usr/bin/psql`
5. `/etc/sudoers.d/admins.conf` নামে ফাইল তৈরি করে নিয়ম লিখলে, কিন্তু কাজ করছে না। কেন?
6. `sudo echo hi > /etc/test` কেন "Permission denied" দেয়? দুটো সমাধান লেখো।
7. কোনো ইউজারকে শুধু `sudo vim /etc/hosts` চালানোর অনুমতি দেওয়া কেন বিপজ্জনক? ভালো বিকল্প কী?
8. sudo-র ডিফল্ট timestamp timeout কত, আর প্রতিবার পাসওয়ার্ড চাওয়ার জন্য কী লিখবে?
9. Ubuntu আর RHEL-এ sudo-র লগ কোথায় পাবে? journal থেকে দেখার কমান্ড কী?
10. polkit আর sudo-র কাজের ধরনে মূল পার্থক্য কী? GUI-তে polkit কেন ভালো?

<details>
<summary>উত্তর দেখো</summary>

1. `su` চায় **target ইউজারের** (সাধারণত root-এর) পাসওয়ার্ড, `sudo` চায় **নিজের** পাসওয়ার্ড। ফলে root পাসওয়ার্ড কাউকে জানাতে হয় না, প্রত্যেকের কাজ তার নিজের নামে লগ হয়, আর কারো অধিকার সরাতে শুধু তাকে গ্রুপ/sudoers থেকে সরালেই হয়।

2. (ক) `su -` target-এর home ডিরেক্টরিতে নিয়ে যায়, `su` বর্তমান ডিরেক্টরিতেই রাখে। (খ) `su -` login shell, environment রিসেট করে target-এর profile পড়ে; `su`-তে তোমার অনেক চলক (যেমন `PATH`) থেকে যায়।

3. `sudo -i` login shell চালায়, যা target ইউজারের home (`/root`)-এ চলে যায়। `sudo -s` non-login shell, তাই বর্তমান ডিরেক্টরি বদলায় না।

4. "`ops` গ্রুপের সদস্যরা, শুধু `web01` নামের মেশিনে, `postgres` ইউজার হিসেবে, পাসওয়ার্ড ছাড়াই, `/usr/bin/psql` কমান্ড (যেকোনো আর্গুমেন্টসহ) চালাতে পারবে।"

5. `/etc/sudoers.d/`-এ যেসব ফাইলের নামে `.` থাকে (বা যা `~` দিয়ে শেষ হয়), sudo সেগুলো উপেক্ষা করে। নাম বদলে `admins` বা `10-admins` রাখো, পারমিশন `0440` নিশ্চিত করো, আর `sudo -l -U user` দিয়ে যাচাই করো।

6. redirection (`>`) করে তোমার নিজের shell, sudo নয়। sudo শুধু `echo`-কে root বানায়। সমাধান: `echo hi | sudo tee /etc/test` অথবা `sudo sh -c 'echo hi > /etc/test'`।

7. vim-এর ভেতর থেকে `:!bash` বা `:shell` দিয়ে root shell পাওয়া যায়, আবার `:e /etc/shadow` দিয়ে যেকোনো ফাইল খোলা যায়। ফলে কার্যত পূর্ণ root। ভালো বিকল্প: `tester ALL=(root) sudoedit /etc/hosts`, তাতে এডিটর ইউজারের নিজের নামে চলে।

8. ১৫ মিনিট। প্রতিবার চাওয়ার জন্য `Defaults timestamp_timeout=0` (visudo দিয়ে)। হাতে বাতিল করতে `sudo -k`।

9. Debian/Ubuntu: `/var/log/auth.log`; Fedora/RHEL: `/var/log/secure` (rsyslog থাকলে)। journal থেকে: `journalctl -t sudo` (বা `journalctl _COMM=sudo`)।

10. sudo পুরো একটা প্রোগ্রামকে root হিসেবে চালায়। polkit-এ অ্যাপ সাধারণ ইউজার হিসেবেই থাকে, শুধু নির্দিষ্ট একটা "action" (যেমন সময় বদলানো) একটা ছোট root service-কে D-Bus দিয়ে করতে বলে, আর polkit ঠিক করে সেটা অনুমোদিত কি না। GUI-তে বিশাল অ্যাপকে root বানাতে হয় না বলে এটা নিরাপদ, আর নিয়ম action ধরে সূক্ষ্মভাবে দেওয়া যায়।

</details>

---

[← আগের অধ্যায়](17-special-permissions-acl.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](19-redirection-pipe.md)

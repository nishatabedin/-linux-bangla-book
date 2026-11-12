# অধ্যায় ৪৫: লিনাক্স নিরাপত্তা ও হার্ডেনিং

> "নিরাপত্তা কোনো প্রোডাক্ট না, এটা একটা প্রক্রিয়া।" — Bruce Schneier

এতদিন আমরা শিখেছি লিনাক্স দিয়ে কীভাবে কাজ করতে হয়। এই অধ্যায়ে শিখব কীভাবে সেই সিস্টেমকে **সুরক্ষিত** রাখতে হয়। একটা নতুন সার্ভার ইন্টারনেটে চালু করলে কয়েক মিনিটের মধ্যেই সারা দুনিয়ার bot তার SSH পোর্টে পাসওয়ার্ড অনুমান করা শুরু করে দেয়। তাই **hardening** (হার্ডেনিং — সিস্টেমকে "শক্ত" করা, মানে আক্রমণের সুযোগ যতটা পারা যায় কমিয়ে আনা) প্রতিটা লিনাক্স অ্যাডমিনের রোজকার কাজ।

## এই অধ্যায়ে যা শিখবে

- নিরাপত্তার স্তর (defense in depth) ধারণা ও threat model
- সিস্টেম আপডেট রাখা এবং `unattended-upgrades` / `dnf-automatic` দিয়ে স্বয়ংক্রিয় সিকিউরিটি আপডেট
- শক্তিশালী পাসওয়ার্ড নীতি, পাসওয়ার্ডের মেয়াদ (`chage`) এবং PAM কীভাবে কাজ করে
- SSH সার্ভার হার্ডেনিং — key-based login, root login বন্ধ, ইত্যাদি
- `fail2ban` দিয়ে brute-force আক্রমণ ঠেকানো
- অপ্রয়োজনীয় সার্ভিস বন্ধ করা এবং খোলা পোর্ট অডিট (`ss`, `lsof`, `nmap`)
- DAC বনাম MAC — দুই ধরনের অ্যাক্সেস কন্ট্রোল
- SELinux: mode, `getenforce`, `setenforce`, context, `restorecon`, boolean, `audit2why`
- AppArmor: `aa-status`, profile, enforce/complain mode
- `auditd` দিয়ে কে কী করল তার হিসাব রাখা
- AIDE দিয়ে ফাইল ইন্টেগ্রিটি যাচাই, `rkhunter` দিয়ে rootkit খোঁজা
- `sha256sum` ও GPG দিয়ে ডাউনলোড করা ফাইল যাচাই
- Lynis দিয়ে পুরো সিস্টেমের নিরাপত্তা অডিট

---

## ৪৫.১ নিরাপত্তার মূল ধারণা

### CIA ট্রায়াড

নিরাপত্তার তিনটা মূল লক্ষ্যকে বলা হয় **CIA triad**:

| লক্ষ্য | মানে | উদাহরণ |
|---|---|---|
| **Confidentiality** (গোপনীয়তা) | যার দেখার অধিকার নেই, সে যেন দেখতে না পারে | `/etc/shadow` শুধু root পড়তে পারে |
| **Integrity** (অখণ্ডতা) | ডেটা যেন অনুমতি ছাড়া বদলে না যায় | AIDE দিয়ে `/bin/ls` বদলেছে কি না দেখা |
| **Availability** (প্রাপ্যতা) | সার্ভিস যেন দরকারের সময় চালু থাকে | DDoS ঠেকানো, ব্যাকআপ রাখা |

### Defense in Depth — নিরাপত্তার স্তর

একটা পুরোনো দুর্গের কথা ভাবো। শুধু একটা দেয়াল থাকে না — বাইরে পরিখা (moat), তারপর উঁচু দেয়াল, তারপর ফটকে প্রহরী, ভেতরে আরেকটা দেয়াল, আর রাজার ঘরে তালা। শত্রু একটা স্তর ভাঙলেও পরের স্তরে আটকে যায়। লিনাক্স নিরাপত্তাও ঠিক এভাবে স্তরে স্তরে সাজানো:

```
   ইন্টারনেট (আক্রমণকারী)
          │
          ▼
 ┌──────────────────────────────────────────────┐
 │ স্তর ১: নেটওয়ার্ক — firewall, fail2ban          │  (অধ্যায় ৩৯)
 │ ┌──────────────────────────────────────────┐ │
 │ │ স্তর ২: সার্ভিস — শুধু দরকারি পোর্ট খোলা,     │ │
 │ │         SSH hardening                    │ │
 │ │ ┌──────────────────────────────────────┐ │ │
 │ │ │ স্তর ৩: অথেন্টিকেশন — শক্ত পাসওয়ার্ড,   │ │ │
 │ │ │         SSH key, PAM, sudo            │ │ │
 │ │ │ ┌──────────────────────────────────┐ │ │ │
 │ │ │ │ স্তর ৪: অ্যাক্সেস কন্ট্রোল —        │ │ │ │
 │ │ │ │   DAC (permission) + MAC         │ │ │ │
 │ │ │ │   (SELinux / AppArmor)           │ │ │ │
 │ │ │ │ ┌──────────────────────────────┐ │ │ │ │
 │ │ │ │ │ স্তর ৫: ডেটা ও কার্নেল —        │ │ │ │ │
 │ │ │ │ │  আপডেট, encryption, sysctl   │ │ │ │ │
 │ │ │ │ └──────────────────────────────┘ │ │ │ │
 │ │ │ └──────────────────────────────────┘ │ │ │
 │ │ └──────────────────────────────────────┘ │ │
 │ └──────────────────────────────────────────┘ │
 └──────────────────────────────────────────────┘
   সব স্তরের পাশে: নজরদারি — auditd, logs, AIDE, rkhunter, Lynis
```

> 📝 **মনে রাখো:** কোনো একটা টুল একা তোমাকে নিরাপদ রাখবে না। firewall আছে বলে SSH-এ দুর্বল পাসওয়ার্ড রাখা চলবে না, আবার SELinux আছে বলে আপডেট বাদ দেওয়া চলবে না।

### Threat Model ও Attack Surface

- **Threat model** — "আমি কাকে থেকে, কী রক্ষা করছি?" একটা ব্যক্তিগত ল্যাপটপ আর একটা ব্যাংকের সার্ভারের threat model এক না।
- **Attack surface** (আক্রমণের পৃষ্ঠতল) — আক্রমণকারী যতগুলো দরজা দিয়ে ঢোকার চেষ্টা করতে পারে: খোলা পোর্ট, চালু সার্ভিস, ইউজার অ্যাকাউন্ট, ইনস্টল করা সফটওয়্যার। হার্ডেনিং-এর মূল কাজ হলো এই পৃষ্ঠতল **ছোট করা**।
- **Principle of least privilege** (ন্যূনতম অধিকারের নীতি) — প্রত্যেক ইউজার ও প্রোগ্রাম শুধু ততটুকু ক্ষমতা পাবে যতটুকু তার কাজের জন্য একান্ত দরকার। (sudo-র বিস্তারিত [অধ্যায় ১৮](18-sudo-root.md)-এ।)

---

## ৪৫.২ সিস্টেম আপডেট — প্রথম এবং সবচেয়ে গুরুত্বপূর্ণ ধাপ

বাস্তবে বেশিরভাগ সফল আক্রমণ হয় এমন দুর্বলতা (vulnerability) দিয়ে যার patch আগেই বের হয়ে গিয়েছিল, কিন্তু অ্যাডমিন আপডেট করেননি। প্রতিটা দুর্বলতার একটা আইডি থাকে, যাকে বলে **CVE** (Common Vulnerabilities and Exposures), যেমন `CVE-2024-3094` (xz backdoor)।

### ম্যানুয়াল আপডেট

প্যাকেজ ম্যানেজারের বিস্তারিত [অধ্যায় ৩০](30-package-management.md)-এ আছে। এখানে নিরাপত্তার দিকটা দেখি:

```bash
# Debian / Ubuntu
$ sudo apt update && sudo apt upgrade

# Fedora / RHEL
$ sudo dnf upgrade --refresh

# RHEL/Fedora: শুধু সিকিউরিটি আপডেট দেখা ও নেওয়া
$ dnf updateinfo list --security
$ sudo dnf upgrade --security
```

কার্নেল বা `glibc`-এর মতো জিনিস আপডেট হলে রিবুট লাগতে পারে। কীভাবে বুঝবে?

```bash
# Debian / Ubuntu: এই ফাইলটা থাকলে রিবুট দরকার
$ cat /var/run/reboot-required
*** System restart required ***
$ cat /var/run/reboot-required.pkgs
linux-image-6.8.0-48-generic

# RHEL / Fedora (dnf-utils / dnf-plugins-core প্যাকেজ)
$ sudo needs-restarting -r
Core libraries or services have been updated since boot-up:
  * kernel
Reboot is required to fully utilize these updates.
```

> 💡 **টিপস:** Ubuntu-তে `needrestart` প্যাকেজ আপডেটের পর জানিয়ে দেয় কোন কোন সার্ভিস পুরোনো লাইব্রেরি নিয়ে চলছে এবং রিস্টার্ট দরকার।

### unattended-upgrades (Debian / Ubuntu)

সার্ভারে প্রতিদিন নিজে লগইন করে আপডেট করা বাস্তবসম্মত না। `unattended-upgrades` প্যাকেজ প্রতিদিন নিজে থেকে সিকিউরিটি আপডেট ইনস্টল করে দেয়। Ubuntu-তে এটা সাধারণত আগে থেকেই ইনস্টল ও চালু থাকে; Debian-এ নিজে ইনস্টল করতে হয়।

**ভেতরে কীভাবে কাজ করে:**

```
 systemd timer: apt-daily.timer ──────► apt-daily.service
   (দিনে ২ বার, এলোমেলো সময়ে)            └─ package list আপডেট + ডাউনলোড
                                              (20auto-upgrades-এর সেটিং দেখে)

 systemd timer: apt-daily-upgrade.timer ─► apt-daily-upgrade.service
   (সকাল ~৬টার দিকে)                       └─ unattended-upgrade চালায়
                                              └─ 50unattended-upgrades-এ
                                                 অনুমোদিত origin থেকে আপডেট
                                              └─ লগ: /var/log/unattended-upgrades/
```

ইনস্টল ও চালু করা:

```bash
$ sudo apt install unattended-upgrades apt-listchanges
$ sudo dpkg-reconfigure -plow unattended-upgrades
```

দ্বিতীয় কমান্ডটা একটা ডায়ালগ দেখায়, "Yes" দিলে `/etc/apt/apt.conf.d/20auto-upgrades` ফাইল তৈরি হয়:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

এখানে `"1"` মানে "প্রতি ১ দিনে একবার"। `"0"` দিলে বন্ধ।

কোন উৎস (origin) থেকে আপডেট নেওয়া হবে, সেটা ঠিক হয় `/etc/apt/apt.conf.d/50unattended-upgrades` ফাইলে:

```
Unattended-Upgrade::Allowed-Origins {
        "${distro_id}:${distro_codename}";
        "${distro_id}:${distro_codename}-security";
        "${distro_id}ESMApps:${distro_codename}-apps-security";
        "${distro_id}ESM:${distro_codename}-infra-security";
//      "${distro_id}:${distro_codename}-updates";
};

// দরকার হলে নির্দিষ্ট প্যাকেজ বাদ দাও
Unattended-Upgrade::Package-Blacklist {
//      "linux-";
};

Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// রিবুট লাগলে নিজে থেকে রিবুট (সার্ভারে সাবধানে!)
Unattended-Upgrade::Automatic-Reboot "false";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";

// ইমেইলে রিপোর্ট (mail সিস্টেম কনফিগার থাকলে)
//Unattended-Upgrade::Mail "admin@example.com";
```

> 📝 **মনে রাখো:** `//` দিয়ে শুরু লাইন কমেন্ট। ডিফল্টভাবে শুধু `-security` চালু থাকে — এটাই নিরাপদ পছন্দ, কারণ সাধারণ ফিচার আপডেট মাঝে মাঝে জিনিস ভাঙতে পারে।

টেস্ট করা (আসলে কিছু ইনস্টল না করে দেখা কী হতো):

```bash
$ sudo unattended-upgrade --dry-run --debug
...
Allowed origins are: o=Ubuntu,a=noble, o=Ubuntu,a=noble-security, ...
Packages that will be upgraded: libssl3t64 openssl
...

$ systemctl list-timers 'apt-daily*'
NEXT                        LEFT     LAST                        PASSED  UNIT                    ACTIVATES
Wed 2026-11-12 18:21:04 UTC 5h left  Wed 2026-11-12 06:40:11 UTC 6h ago  apt-daily.timer         apt-daily.service
Thu 2026-11-13 06:12:33 UTC 17h left Wed 2026-11-12 06:05:02 UTC 7h ago  apt-daily-upgrade.timer apt-daily-upgrade.service

$ sudo tail /var/log/unattended-upgrades/unattended-upgrades.log
```

### dnf-automatic (Fedora / RHEL)

RHEL/Fedora-তে একই কাজ করে `dnf-automatic`:

```bash
$ sudo dnf install dnf-automatic
$ sudo vi /etc/dnf/automatic.conf
```

```ini
[commands]
# default = সব আপডেট, security = শুধু সিকিউরিটি
upgrade_type = security
download_updates = yes
apply_updates = yes
```

```bash
$ sudo systemctl enable --now dnf-automatic.timer
$ systemctl list-timers '*automatic*'
```

> ⚠️ **সতর্কতা:** Fedora 41 থেকে `dnf5` এসেছে, সেখানে প্যাকেজ ও টাইমারের নাম একটু ভিন্ন হতে পারে (`dnf5-plugin-automatic`)। নিশ্চিত হতে `systemctl list-timers '*automatic*'` দিয়ে আসল নাম দেখে নাও।

---

## ৪৫.৩ শক্তিশালী পাসওয়ার্ড ও PAM

### ভালো পাসওয়ার্ড কেমন?

পাসওয়ার্ডের শক্তি মূলত তার **দৈর্ঘ্য** আর **অনিশ্চয়তা** (entropy)-র ওপর নির্ভর করে। `P@ssw0rd1` দেখতে জটিল মনে হলেও অভিধান-ভিত্তিক আক্রমণে মুহূর্তে ভেঙে যায়। অন্যদিকে `nodi-tara-cha-boi-megh` এর মতো চার-পাঁচটা এলোমেলো শব্দের **passphrase** মনে রাখা সহজ, কিন্তু ভাঙা অনেক কঠিন।

> 💡 **টিপস:** একটা password manager (KeePassXC, Bitwarden) ব্যবহার করো আর সার্ভারে যতটা সম্ভব পাসওয়ার্ডের বদলে SSH key দিয়ে লগইন করো (৪৫.৪ অংশ)।

পাসওয়ার্ড কোথায় থাকে, `/etc/shadow`-এর গঠন কী — এসব [অধ্যায় ১৫](15-users-groups.md)-এ বিস্তারিত আছে। এখানে নীতি (policy) চাপানোর দিকটা দেখব।

### PAM কী?

**PAM** (Pluggable Authentication Modules) হলো লিনাক্সের অথেন্টিকেশনের "সুইচবোর্ড"। `login`, `sshd`, `sudo`, `su`, `passwd` — এরা কেউ নিজে পাসওয়ার্ড যাচাই করে না। তারা PAM-কে জিজ্ঞেস করে, "এই ইউজারকে ঢুকতে দেব?" আর PAM কনফিগ ফাইল দেখে ঠিক করে কোন কোন মডিউল (প্লাগইন) দিয়ে যাচাই হবে।

উপমা: একটা অফিস ভবনের প্রবেশদ্বারের কথা ভাবো। প্রতিটা দরজা (প্রোগ্রাম) নিজের তালা বানায় না — সবাই একই নিরাপত্তা ডেস্ক (PAM) ব্যবহার করে। ডেস্কের নিয়মের খাতা (`/etc/pam.d/`) বদলালেই সব দরজার নিয়ম বদলে যায়, কোনো দরজা নতুন করে বানাতে হয় না।

```
 ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
 │ login  │  │  sshd  │  │  sudo  │  │ passwd │   ← PAM-aware অ্যাপ্লিকেশন
 └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘
     └───────────┴─────┬─────┴───────────┘
                       ▼
               ┌───────────────┐
               │  libpam (PAM) │ ← /etc/pam.d/<service> পড়ে
               └───────┬───────┘
     ┌─────────────┬───┴─────────┬──────────────┐
     ▼             ▼             ▼              ▼
 pam_unix.so  pam_pwquality.so  pam_faillock.so  pam_limits.so ...
 (shadow       (পাসওয়ার্ডের     (বারবার ভুল     (রিসোর্স সীমা)
  পাসওয়ার্ড)    শক্তি যাচাই)      হলে লক)
```

### PAM কনফিগ ফাইলের গঠন

প্রতিটা সার্ভিসের জন্য `/etc/pam.d/`-তে একটা ফাইল থাকে। প্রতিটা লাইনের গঠন:

```
<type>    <control>    <module>    [arguments]
```

**চার ধরনের type (management group):**

| Type | কাজ |
|---|---|
| `auth` | তুমি কে, তা প্রমাণ করা (পাসওয়ার্ড, OTP) |
| `account` | অ্যাকাউন্ট ব্যবহারযোগ্য কি না (মেয়াদ শেষ? লক?) |
| `password` | পাসওয়ার্ড বদলানোর নিয়ম |
| `session` | লগইনের আগে-পরে কাজ (হোম ডিরেক্টরি মাউন্ট, লগ, limits) |

**Control flag:**

| Flag | মানে |
|---|---|
| `required` | পাস করতেই হবে; ফেল করলেও বাকি মডিউল চলে, শেষে ফল "ব্যর্থ" |
| `requisite` | পাস করতেই হবে; ফেল করলে সঙ্গে সঙ্গে থেমে যায় |
| `sufficient` | পাস করলে (এবং আগে কোনো required ফেল না করলে) সঙ্গে সঙ্গে সফল; ফেল করলে উপেক্ষা |
| `optional` | ফল সাধারণত গুরুত্ব পায় না (একমাত্র মডিউল হলে ছাড়া) |
| `include` / `substack` | অন্য একটা ফাইলের নিয়ম এখানে টেনে আনা |

Ubuntu-তে উদাহরণ:

```bash
$ cat /etc/pam.d/sshd | grep -v '^#' | grep -v '^$'
@include common-auth
account    required     pam_nologin.so
@include common-account
session [success=ok ignore=ignore module_unknown=ignore default=bad] pam_selinux.so close
session    required     pam_loginuid.so
...
@include common-password
```

> ⚠️ **সতর্কতা:** PAM ফাইলে একটা ভুল লাইন দিলে তুমি নিজেই সিস্টেম থেকে লকড আউট হয়ে যেতে পারো — sudo-ও কাজ করবে না! PAM ফাইল এডিট করার সময় **সবসময় একটা root শেল আলাদা খোলা রাখো**, আর অন্য টার্মিনালে নতুন লগইন টেস্ট করার পরেই পুরোনোটা বন্ধ করো। RHEL/Fedora-তে PAM ফাইল হাতে না বদলে `authselect` টুল ব্যবহার করা উচিত; Debian/Ubuntu-তে `pam-auth-update`।

### pam_pwquality — পাসওয়ার্ডের শক্তি চাপানো

```bash
# Debian / Ubuntu (RHEL/Fedora-তে আগে থেকেই থাকে)
$ sudo apt install libpam-pwquality
```

কনফিগ ফাইল `/etc/security/pwquality.conf` (বা `/etc/security/pwquality.conf.d/*.conf`):

```ini
minlen = 14          # ন্যূনতম দৈর্ঘ্য
minclass = 3         # ছোট হাতের, বড় হাতের, সংখ্যা, চিহ্ন — অন্তত ৩ ধরনের
maxrepeat = 3        # একই অক্ষর পরপর ৩ বারের বেশি না
dictcheck = 1        # অভিধানের শব্দ নিষেধ
usercheck = 1        # ইউজারনেম থাকা চলবে না
difok = 5            # পুরোনো পাসওয়ার্ড থেকে অন্তত ৫টা অক্ষর আলাদা
enforce_for_root     # root নিজেও এই নিয়ম মানবে
```

`dcredit`, `ucredit`, `lcredit`, `ocredit` অপশনে ঋণাত্মক মান দিলে (যেমন `dcredit = -1`) মানে "অন্তত ১টা সংখ্যা থাকতেই হবে"।

```bash
$ passwd
Current password:
New password: abc123
BAD PASSWORD: The password is shorter than 14 characters
```

### pam_faillock — বারবার ভুল পাসওয়ার্ড দিলে লক

কনফিগ: `/etc/security/faillock.conf`

```ini
deny = 5             # ৫ বার ভুল হলে লক
unlock_time = 900    # ৯০০ সেকেন্ড (১৫ মিনিট) পরে নিজে খুলবে
fail_interval = 900  # এই সময়ের মধ্যে ভুলগুলো গোনা হবে
# even_deny_root     # root-কেও লক করবে (সাবধান)
```

- RHEL 9 / Fedora: `sudo authselect enable-feature with-faillock`
- Ubuntu/Debian-এ ডিফল্টভাবে চালু থাকে না; `common-auth`-এ মডিউল যোগ করতে হয় (সাবধানে, ম্যান পেজ `man pam_faillock` দেখে)।

```bash
$ sudo faillock --user rahim
rahim:
When                Type  Source                                           Valid
2026-11-12 10:21:33 RHOST 203.0.113.50                                         V
2026-11-12 10:21:40 RHOST 203.0.113.50                                         V

$ sudo faillock --user rahim --reset      # লক খুলে দেওয়া
```

### পাসওয়ার্ডের মেয়াদ — `chage`

```bash
$ sudo chage -l rahim
Last password change                                    : Nov 01, 2026
Password expires                                        : never
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires       : 7
```

| অপশন | কাজ |
|---|---|
| `-l` | মেয়াদের তথ্য দেখানো |
| `-M 90` | সর্বোচ্চ ৯০ দিন পর পাসওয়ার্ড বদলাতে হবে |
| `-m 1` | অন্তত ১ দিন পর আবার বদলানো যাবে |
| `-W 7` | মেয়াদ শেষের ৭ দিন আগে সতর্ক করবে |
| `-I 30` | মেয়াদ শেষের ৩০ দিন পর অ্যাকাউন্ট নিষ্ক্রিয় |
| `-E 2027-06-30` | ঐ তারিখে পুরো অ্যাকাউন্টের মেয়াদ শেষ |
| `-d 0` | পরের লগইনে পাসওয়ার্ড বদলাতে বাধ্য করা |

নতুন ইউজারদের ডিফল্ট মান আসে `/etc/login.defs` থেকে (`PASS_MAX_DAYS`, `PASS_MIN_DAYS`, `PASS_WARN_AGE`)।

> 💡 **টিপস:** আধুনিক নির্দেশিকা (যেমন NIST SP 800-63B) বলে, বারবার জোর করে পাসওয়ার্ড বদলানোর চেয়ে লম্বা, অনন্য পাসওয়ার্ড + MFA বেশি কার্যকর। তাই অন্ধভাবে `-M 30` বসিয়ে দিও না; তোমার প্রতিষ্ঠানের নীতি অনুযায়ী ঠিক করো।

### অ্যাকাউন্ট পরিচ্ছন্নতা

```bash
# খালি পাসওয়ার্ডওয়ালা অ্যাকাউন্ট খোঁজা (ভয়ংকর!)
$ sudo awk -F: '($2 == "") {print $1}' /etc/shadow

# UID 0 (root ক্ষমতা) আছে কার কার — শুধু root থাকা উচিত
$ awk -F: '($3 == 0) {print $1}' /etc/passwd
root

# অব্যবহৃত অ্যাকাউন্ট লক করা
$ sudo usermod -L -e 1 olduser     # পাসওয়ার্ড লক + অ্যাকাউন্টের মেয়াদ শেষ
$ sudo passwd -S olduser
olduser L 2026-01-10 0 99999 7 -1
```

---

## ৪৫.৪ SSH হার্ডেনিং

SSH-এর মৌলিক ব্যবহার ও key তৈরি [অধ্যায় ৩৮](38-ssh-remote.md)-এ আছে। এখানে শুধু **সার্ভার দিকটা শক্ত করা** দেখব।

### কেন দরকার?

ইন্টারনেটে খোলা যেকোনো সার্ভারের লগ দেখো:

```bash
$ sudo journalctl -u ssh --since "1 hour ago" | grep -c "Failed password"
1874
```

এক ঘণ্টায় প্রায় দুই হাজার ব্যর্থ চেষ্টা! এগুলো স্বয়ংক্রিয় bot।

### ধাপ ১: আগে key-based login চালু ও টেস্ট করো

```bash
# নিজের কম্পিউটার থেকে
$ ssh-keygen -t ed25519 -C "rahim@laptop"
$ ssh-copy-id rahim@server.example.com
$ ssh rahim@server.example.com      # পাসওয়ার্ড ছাড়াই ঢুকতে পারছ? তাহলে ঠিক আছে
```

> ⚠️ **সতর্কতা:** key দিয়ে লগইন **নিশ্চিত হওয়ার আগে** কখনো `PasswordAuthentication no` দিও না। আর কনফিগ বদলানোর সময় একটা SSH সেশন খোলা রেখে **দ্বিতীয় একটা টার্মিনাল থেকে** নতুন লগইন টেস্ট করো। এক সেশনে ভুল করলে পুরোনো সেশন দিয়ে ঠিক করতে পারবে।

### ধাপ ২: drop-in কনফিগ ফাইল

মূল ফাইল `/etc/ssh/sshd_config` সরাসরি না বদলে আধুনিক distro-তে `/etc/ssh/sshd_config.d/` ডিরেক্টরিতে আলাদা `.conf` ফাইল রাখা ভালো। মূল ফাইলের একদম শুরুতে থাকে:

```
Include /etc/ssh/sshd_config.d/*.conf
```

> 📝 **মনে রাখো — sshd-এর "প্রথম মান জেতে" নিয়ম:** `sshd_config`-এ কোনো অপশন একাধিকবার থাকলে **প্রথমবার যে মান পাওয়া যায় সেটাই কার্যকর** (বেশিরভাগ অপশনের ক্ষেত্রে)। drop-in ফাইলগুলো বর্ণানুক্রমে পড়া হয়। Ubuntu cloud image-এ প্রায়ই `50-cloud-init.conf` ফাইলে `PasswordAuthentication yes` থাকে! তাই তোমার ফাইলের নাম `00-hardening.conf` রাখো যাতে সেটা আগে পড়া হয়।

```bash
$ sudo nano /etc/ssh/sshd_config.d/00-hardening.conf
```

```
# --- অথেন্টিকেশন ---
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
AuthenticationMethods publickey

# --- শুধু নির্দিষ্ট ইউজার/গ্রুপ ---
AllowGroups sshusers

# --- brute-force কঠিন করা ---
MaxAuthTries 3
LoginGraceTime 30
MaxStartups 10:30:60

# --- অপ্রয়োজনীয় ফিচার বন্ধ ---
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
PermitTunnel no

# --- নিষ্ক্রিয় সেশন কেটে দেওয়া (৩০০ সে × ২ = ~১০ মিনিট) ---
ClientAliveInterval 300
ClientAliveCountMax 2

# --- লগ ---
LogLevel VERBOSE
```

**অপশনগুলোর ব্যাখ্যা:**

| অপশন | মান | কেন |
|---|---|---|
| `PermitRootLogin` | `no` | root নাম সবাই জানে — আক্রমণকারীর অর্ধেক কাজ সহজ। সাধারণ ইউজারে ঢুকে `sudo` ব্যবহার করো। (`prohibit-password` দিলে root শুধু key দিয়ে ঢুকতে পারবে) |
| `PasswordAuthentication` | `no` | পাসওয়ার্ড অনুমান করে ঢোকার পথ পুরো বন্ধ |
| `KbdInteractiveAuthentication` | `no` | keyboard-interactive (পুরোনো নাম `ChallengeResponseAuthentication`) পথেও পাসওয়ার্ড চাওয়া বন্ধ |
| `AllowUsers` / `AllowGroups` | ইউজার/গ্রুপ | তালিকার বাইরে কেউ ঢুকতে পারবে না |
| `MaxAuthTries` | `3` | এক সংযোগে সর্বোচ্চ ৩ বার চেষ্টা |
| `LoginGraceTime` | `30` | ৩০ সেকেন্ডে লগইন শেষ না হলে সংযোগ কাটা |
| `MaxStartups` | `10:30:60` | অসম্পূর্ণ সংযোগ ১০টার বেশি হলে ৩০% হারে বাতিল, ৬০ হলে সব বাতিল |
| `X11Forwarding` | `no` | দরকার না হলে গ্রাফিক্স ফরওয়ার্ডিং বন্ধ |
| `ClientAliveInterval` | `300` | ৫ মিনিট পরপর ক্লায়েন্ট বেঁচে আছে কি না দেখা |

> ⚠️ **সতর্কতা:** `AllowGroups sshusers` দিলে আগে গ্রুপটা বানিয়ে নিজেকে যোগ করো: `sudo groupadd sshusers && sudo usermod -aG sshusers rahim`। নাহলে তুমি নিজেই আটকে যাবে।

### ধাপ ৩: যাচাই করে রিলোড

```bash
# সিনট্যাক্স পরীক্ষা — কিছু না দেখালে মানে ঠিক আছে
$ sudo sshd -t

# আসলে কোন মান কার্যকর হচ্ছে (সব ফাইল মিলিয়ে)
$ sudo sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|maxauthtries'
permitrootlogin no
passwordauthentication no
maxauthtries 3

# রিলোড (চলমান সেশন কাটে না)
$ sudo systemctl reload ssh      # Debian / Ubuntu
$ sudo systemctl reload sshd     # RHEL / Fedora
```

> 📝 **মনে রাখো:** সার্ভিসের নাম Debian/Ubuntu-তে `ssh`, RHEL/Fedora-তে `sshd`।

### পোর্ট বদলানো — দরকার আছে কি?

`Port 2222` দিলে লগের "শব্দ" (bot-এর চেষ্টা) অনেক কমে যায়, কিন্তু এটা আসল নিরাপত্তা না — এক `nmap` স্ক্যানেই নতুন পোর্ট ধরা পড়ে (একে বলে *security through obscurity*)। key-only লগইন আসল সুরক্ষা। তবু বদলাতে চাইলে তিনটা জায়গায় খেয়াল রাখো:

```bash
# ১. firewall-এ নতুন পোর্ট খোলো (পুরোনোটা বন্ধের আগে!)
$ sudo ufw allow 2222/tcp                                  # Ubuntu
$ sudo firewall-cmd --permanent --add-port=2222/tcp && sudo firewall-cmd --reload  # RHEL

# ২. RHEL/Fedora-তে SELinux-কে জানাও যে sshd এই পোর্ট ব্যবহার করবে
$ sudo semanage port -a -t ssh_port_t -p tcp 2222

# ৩. Ubuntu 22.10+ (24.04 সহ) ডিফল্টভাবে ssh.socket (socket activation) ব্যবহার করে;
#    সেখানে পোর্ট বদলের পর:
$ sudo systemctl daemon-reload
$ sudo systemctl restart ssh.socket
```

### SSH-এর জন্য দরকারি কীবোর্ড "শর্টকাট" — escape sequence

হার্ডেনিং-এর পর সেশন আটকে গেলে (যেমন নেটওয়ার্ক কেটে গেছে) এগুলো কাজে লাগে। এগুলো কাজ করে **নতুন লাইনের শুরুতে** (আগে `Enter` চাপো):

| কী | কাজ |
|---|---|
| `Enter` তারপর `~.` | ঝুলে থাকা SSH সেশন জোর করে বন্ধ |
| `Enter` তারপর `~?` | সব escape sequence-এর তালিকা |
| `Enter` তারপর `~#` | চলমান forwarded সংযোগের তালিকা |
| `Enter` তারপর `~Ctrl+Z` | SSH সেশন background-এ পাঠানো |
| `Super+L` (GNOME/KDE) | ডেস্কটপ স্ক্রিন লক — কম্পিউটার ছেড়ে উঠলে অবশ্যই! |

### আরও এক ধাপ: 2FA

SSH-এ key-এর সাথে TOTP (Google Authenticator ধরনের) যোগ করা যায় `libpam-google-authenticator` (Debian/Ubuntu) বা `google-authenticator` (EPEL/Fedora) প্যাকেজ দিয়ে, যেখানে `AuthenticationMethods publickey,keyboard-interactive` দিতে হয়। এটা PAM-এর একটা চমৎকার বাস্তব উদাহরণ।

---

## ৪৫.৫ fail2ban — brute-force আক্রমণকারীকে ব্যান

**কী করে:** লগ ফাইল পড়ে দেখে কোন IP বারবার ভুল লগইন করছে, তারপর firewall-এ সেই IP কিছু সময়ের জন্য ব্লক করে দেয়।

উপমা: পাড়ার দারোয়ান খাতায় লিখে রাখছে কে কতবার ভুল চাবি দিয়ে গেট খোলার চেষ্টা করল। ৫ বারের বেশি হলে তাকে এক ঘণ্টার জন্য গেটের সামনে থেকে সরিয়ে দেয়।

```
  ┌─────────────┐   পড়ে   ┌──────────────┐  filter (regex)  ┌──────────┐
  │ journal /   │ ───────► │ fail2ban-    │ ───────────────► │ জেল (jail)│
  │ auth.log    │          │ server       │   মিলে গেলে গোনা  │  sshd     │
  └─────────────┘          └──────────────┘                  └────┬─────┘
                                                maxretry ছাড়ালে   │ action
                                                                  ▼
                                    nftables / iptables / firewalld-এ
                                    "203.0.113.50 DROP" নিয়ম যোগ
                                    bantime পরে নিয়ম মুছে ফেলা
```

- **filter** — `/etc/fail2ban/filter.d/*.conf`, regex দিয়ে লগের কোন লাইন "ব্যর্থতা" তা চেনে
- **action** — `/etc/fail2ban/action.d/*.conf`, ব্যান কীভাবে করবে
- **jail** — filter + action + সংখ্যার সীমা মিলিয়ে একটা নিয়ম

### ইনস্টল

```bash
$ sudo apt install fail2ban                     # Debian / Ubuntu
$ sudo dnf install epel-release && sudo dnf install fail2ban   # RHEL (EPEL থেকে)
$ sudo dnf install fail2ban                     # Fedora
```

### কনফিগ

> ⚠️ **সতর্কতা:** `/etc/fail2ban/jail.conf` সরাসরি এডিট করো না — প্যাকেজ আপডেটে বদলে যেতে পারে। নিজের সেটিং রাখো `/etc/fail2ban/jail.local` বা `/etc/fail2ban/jail.d/*.local` ফাইলে।

```bash
$ sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
# নিজের IP কখনো ব্যান হবে না
ignoreip = 127.0.0.1/8 ::1 192.168.1.0/24
bantime  = 1h
findtime = 10m
maxretry = 5
# বারবার অপরাধীর ব্যানের সময় বাড়তে থাকবে
bantime.increment = true
# journald থেকে লগ পড়া (Debian 12-এ /var/log/auth.log ডিফল্টে নেই)
backend = systemd

[sshd]
enabled = true
port    = ssh
maxretry = 3
```

মানে: **findtime** (১০ মিনিট)-এর মধ্যে **maxretry** (৩) বার ব্যর্থ হলে **bantime** (১ ঘণ্টা) ব্যান।

```bash
$ sudo systemctl enable --now fail2ban
$ sudo fail2ban-client status
Status
|- Number of jail:      1
`- Jail list:   sshd

$ sudo fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 2
|  |- Total failed:     147
|  `- Journal matches:  _SYSTEMD_UNIT=ssh.service + _COMM=sshd
`- Actions
   |- Currently banned: 4
   |- Total banned:     19
   `- Banned IP list:   203.0.113.50 198.51.100.7 ...
```

| কমান্ড | কাজ |
|---|---|
| `fail2ban-client status` | সব jail-এর তালিকা |
| `fail2ban-client status sshd` | নির্দিষ্ট jail-এর অবস্থা |
| `fail2ban-client set sshd banip 1.2.3.4` | হাতে IP ব্যান |
| `fail2ban-client set sshd unbanip 1.2.3.4` | IP আনব্যান |
| `fail2ban-client unban --all` | সব আনব্যান |
| `fail2ban-client reload` | কনফিগ রিলোড |
| `fail2ban-regex <log> <filter>` | filter টেস্ট করা |

> 💡 **টিপস:** nginx, Apache, Postfix ইত্যাদির জন্যও তৈরি filter আছে (`ls /etc/fail2ban/filter.d/`)। শুধু `jail.local`-এ `[nginx-http-auth]` ধরনের সেকশনে `enabled = true` দিলেই হয়।

> 📝 **মনে রাখো:** key-only SSH থাকলে fail2ban আসলে নিরাপত্তা খুব বেশি বাড়ায় না (পাসওয়ার্ড অনুমান তো কাজই করবে না), তবে লগের আবর্জনা ও সার্ভারের লোড কমায়।

---

## ৪৫.৬ অপ্রয়োজনীয় সার্ভিস বন্ধ করা

যে সার্ভিস চলছেই না, তাকে হ্যাক করা যায় না। তাই নিয়ম: **যা দরকার নেই, তা বন্ধ — সম্ভব হলে আনইনস্টল।** systemd-এর বিস্তারিত [অধ্যায় ২৬](26-systemd-services.md)-এ।

```bash
# এখন কোন সার্ভিস চলছে
$ systemctl list-units --type=service --state=running

# বুটে কোনগুলো চালু হবে
$ systemctl list-unit-files --type=service --state=enabled
UNIT FILE                    STATE   PRESET
avahi-daemon.service         enabled enabled
cups.service                 enabled enabled
ssh.service                  enabled enabled
...
```

সার্ভারে সাধারণত দরকার নেই এমন কিছু উদাহরণ:

| সার্ভিস | কী করে | সার্ভারে লাগে? |
|---|---|---|
| `cups` | প্রিন্টার | প্রায়ই না |
| `avahi-daemon` | লোকাল নেটওয়ার্কে mDNS/Bonjour ঘোষণা | না |
| `bluetooth` | ব্লুটুথ | না |
| `rpcbind` | NFS-এর জন্য | NFS না থাকলে না |
| `telnet`, `rsh`, `vsftpd` (অনিরাপদ সেটআপে) | এনক্রিপশনহীন প্রোটোকল | কখনোই না |

```bash
$ sudo systemctl disable --now cups.service cups.socket cups.path
$ sudo systemctl mask avahi-daemon.service avahi-daemon.socket
$ sudo apt purge telnetd         # একেবারে মুছে ফেলা
```

> 📝 **মনে রাখো:** `disable` = বুটে চালু হবে না, কিন্তু অন্য কেউ চাইলে চালু করতে পারে। `mask` = `/dev/null`-এর সাথে symlink করে দেয়, কেউই চালু করতে পারবে না। অনেক সার্ভিস `.socket` ইউনিট দিয়েও জেগে ওঠে — তাই socket-টাও বন্ধ করো।

---

## ৪৫.৭ খোলা পোর্ট অডিট

প্রতিটা খোলা (listening) পোর্ট একটা দরজা। কোনগুলো খোলা আছে আর কে খুলে রেখেছে, জানা জরুরি। (কমান্ডগুলোর পূর্ণ বিবরণ [অধ্যায় ৩৭](37-network-commands.md)-এ।)

### ভেতর থেকে: `ss`

```bash
$ sudo ss -tulpn
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
udp   UNCONN 0      0      127.0.0.53%lo:53         0.0.0.0:*     users:(("systemd-resolve",pid=612,fd=13))
tcp   LISTEN 0      4096   0.0.0.0:22               0.0.0.0:*     users:(("sshd",pid=901,fd=3))
tcp   LISTEN 0      511    0.0.0.0:80               0.0.0.0:*     users:(("nginx",pid=1210,fd=6))
tcp   LISTEN 0      151    127.0.0.1:3306           0.0.0.0:*     users:(("mysqld",pid=1033,fd=21))
tcp   LISTEN 0      4096   0.0.0.0:6379             0.0.0.0:*     users:(("redis-server",pid=1101,fd=6))
```

| অপশন | মানে |
|---|---|
| `-t` | TCP |
| `-u` | UDP |
| `-l` | শুধু listening |
| `-p` | কোন প্রসেস (root লাগে) |
| `-n` | নাম না বানিয়ে সংখ্যায় দেখানো |

**কীভাবে পড়বে:**

- `127.0.0.1:3306` — শুধু নিজের মেশিন থেকে ঢোকা যায়। ভালো।
- `0.0.0.0:6379` বা `[::]:6379` — **সব নেটওয়ার্ক ইন্টারফেস** থেকে খোলা! Redis-এর মতো ডেটাবেস ইন্টারনেটে খোলা থাকা বড় বিপদ। এটা `bind 127.0.0.1` করে দিতে হবে।

```bash
# বিকল্প
$ sudo lsof -i -P -n | grep LISTEN
$ sudo lsof -i :6379          # নির্দিষ্ট পোর্ট কে ব্যবহার করছে
```

### বাইরে থেকে: `nmap`

ভেতর থেকে যা দেখছ, বাইরে থেকে firewall পেরিয়ে আসলে কী দেখা যায় — সেটা বোঝার জন্য **অন্য একটা মেশিন থেকে** স্ক্যান করো:

```bash
$ sudo nmap -sS -sV -p- server.example.com
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
```

| অপশন | মানে |
|---|---|
| `-sS` | SYN স্ক্যান (দ্রুত, root লাগে) |
| `-sV` | সার্ভিসের ভার্সন বের করা |
| `-p-` | ১ থেকে ৬৫৫৩৫ সব পোর্ট |
| `-sU` | UDP স্ক্যান (ধীর) |

> ⚠️ **সতর্কতা:** শুধু **নিজের** বা অনুমতি পাওয়া সিস্টেমে `nmap` চালাও। অন্যের সার্ভারে অনুমতি ছাড়া স্ক্যান অনেক দেশে আইনত অপরাধ হতে পারে এবং ক্লাউড প্রোভাইডারের নীতিভঙ্গ।

**পোর্ট অডিটের সিদ্ধান্ত তালিকা:** প্রতিটা listening পোর্টের জন্য জিজ্ঞেস করো —
1. এই সার্ভিসটা কি দরকার? না → বন্ধ/আনইনস্টল।
2. বাইরে থেকে দরকার? না → `127.0.0.1`-এ bind করো।
3. বাইরে থেকে দরকার, কিন্তু সবার জন্য না? → firewall-এ নির্দিষ্ট IP-কে অনুমতি ([অধ্যায় ৩৯](39-firewall.md))।

---

## ৪৫.৮ DAC বনাম MAC — অ্যাক্সেস কন্ট্রোলের দুই দর্শন

### DAC — Discretionary Access Control

এটা আমাদের চেনা `rwx` পারমিশন ([অধ্যায় ১৬](16-permissions.md))। "Discretionary" মানে **ফাইলের মালিক নিজের ইচ্ছামতো** (at their discretion) ঠিক করে কে ঢুকবে। সমস্যা:

1. মালিক ভুল করে `chmod 777` দিলে সবাই সব পারে।
2. **root সবকিছুর ঊর্ধ্বে।** কোনো সার্ভিস (যেমন root হিসেবে চলা ওয়েব সার্ভার) হ্যাক হলে আক্রমণকারী পুরো সিস্টেম পায়।
3. একটা প্রোগ্রাম সেই ইউজারের **সব** ফাইল ছুঁতে পারে — তোমার ব্রাউজার হ্যাক হলে সে তোমার `~/.ssh/id_ed25519` পড়তে পারে, কারণ দুটোই তোমার।

### MAC — Mandatory Access Control

এখানে **সিস্টেম-ব্যাপী নীতি (policy)** ঠিক করে কোন প্রোগ্রাম কী করতে পারবে — মালিক বা এমনকি root-ও সাধারণ উপায়ে তা এড়াতে পারে না। "Mandatory" মানে বাধ্যতামূলক।

উপমা: একটা হাসপাতাল। DAC হলো প্রতিটা রুমের চাবি — যার হাতে চাবি, সে ঢুকতে পারে, আর চাবির মালিক যাকে খুশি চাবি দিতে পারে। MAC হলো হাসপাতালের নিয়ম: "ক্যান্টিনের কর্মী (চাবি থাকুক বা না থাকুক) অপারেশন থিয়েটারে ঢুকতে পারবে না।" এমনকি হাসপাতালের পরিচালকের দেওয়া চাবিও এই নিয়ম ভাঙতে পারে না।

```
   প্রসেস "nginx" (root বা www-data হিসেবে চলছে)
   চাইছে: /etc/shadow পড়তে
            │
            ▼
   ┌──────────────────────────┐
   │ ১. DAC চেক               │  rwx পারমিশন, owner, group
   │   (UID/GID ভিত্তিক)       │  root হলে প্রায় সবসময় পাস!
   └────────────┬─────────────┘
          পাস   │   ফেল → ❌ EACCES (Permission denied)
                ▼
   ┌──────────────────────────┐
   │ ২. MAC চেক (LSM hook)     │  SELinux: httpd_t কি shadow_t পড়তে পারে?
   │   SELinux / AppArmor     │  AppArmor: nginx profile-এ /etc/shadow আছে?
   └────────────┬─────────────┘
          পাস   │   ফেল → ❌ EACCES + audit log-এ "denied"
                ▼
           ✅ অ্যাক্সেস দেওয়া হলো
```

> 📝 **মনে রাখো:** MAC **DAC-এর পরে** চেক হয় এবং শুধু **আরও সীমাবদ্ধ** করতে পারে, কখনো বাড়তি অনুমতি দেয় না। DAC "না" বললে MAC পর্যন্ত যাওয়াই হয় না।

**ভেতরে কীভাবে:** কার্নেলে আছে **LSM** (Linux Security Modules) framework — কার্নেলের গুরুত্বপূর্ণ জায়গাগুলোতে (ফাইল খোলা, সকেট বানানো, প্রসেস চালানো) "hook" বসানো আছে। SELinux আর AppArmor হলো এই hook-এ লাগানো দুটো ভিন্ন LSM। কোনটা চালু আছে দেখতে:

```bash
$ cat /sys/kernel/security/lsm
lockdown,capability,landlock,yama,apparmor,bpf           # Ubuntu
lockdown,capability,yama,selinux,bpf,landlock            # Fedora/RHEL (উদাহরণ)
```

| বিষয় | SELinux | AppArmor |
|---|---|---|
| ডিফল্ট distro | RHEL, Fedora, CentOS Stream, Rocky, Alma | Ubuntu, Debian (10 থেকে), openSUSE (পুরোনো ভার্সন) |
| ভিত্তি | **label** (প্রতিটা ফাইল/প্রসেসের গায়ে context লেখা) | **path** (ফাইলের পাথ দিয়ে নিয়ম) |
| জটিলতা | বেশি, কিন্তু অনেক সূক্ষ্ম নিয়ন্ত্রণ | কম, শেখা সহজ |
| কভারেজ | প্রায় পুরো সিস্টেম | শুধু যেসব প্রোগ্রামের profile আছে |
| মূল টুল | `getenforce`, `semanage`, `restorecon` | `aa-status`, `aa-enforce`, `aa-complain` |

---

## ৪৫.৯ SELinux

**SELinux** (Security-Enhanced Linux) মূলত আমেরিকার NSA তৈরি করে, ২০০৩ সালে কার্নেল ২.৬-এ যুক্ত হয়। RHEL/Fedora-তে এটা ডিফল্টে চালু থাকে।

> ⚠️ **সতর্কতা:** অনেক টিউটোরিয়ালে প্রথম লাইনেই লেখা থাকে "SELinux বন্ধ করে দিন"। **এটা কোরো না।** SELinux সমস্যা করলে সেটা প্রায় সবসময় একটা ভুল label বা বন্ধ boolean — এই অধ্যায় পড়ে তুমি ২ মিনিটে ঠিক করতে পারবে।

### SELinux mode

| Mode | কী হয় |
|---|---|
| **Enforcing** | নীতি চাপানো হয় — নিষিদ্ধ কাজ আটকানো হয় এবং লগ হয় |
| **Permissive** | কিছু আটকানো হয় না, শুধু "এটা আটকানো উচিত ছিল" লগ হয় — ডিবাগের জন্য |
| **Disabled** | SELinux একদম বন্ধ — কোনো label-ও রক্ষণাবেক্ষণ হয় না |

```bash
$ getenforce
Enforcing

$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33
```

**সাময়িকভাবে mode বদলানো** (রিবুটে আবার আগের মতো):

```bash
$ sudo setenforce 0      # Permissive
$ getenforce
Permissive
$ sudo setenforce 1      # Enforcing
```

`setenforce` শুধু Enforcing ⇄ Permissive-এর মধ্যে বদলাতে পারে, Disabled করতে পারে না।

**স্থায়ীভাবে:** `/etc/selinux/config` ফাইলে:

```ini
SELINUX=enforcing
SELINUXTYPE=targeted
```

> 📝 **মনে রাখো:** RHEL 9-এ পুরোপুরি disable করতে হলে config ফাইলের বদলে কার্নেল প্যারামিটার ব্যবহারের পরামর্শ দেওয়া হয়: `sudo grubby --update-kernel ALL --args selinux=0`। কিন্তু আবার বলছি — disable কোরো না। আর disabled থেকে আবার চালু করলে পুরো ফাইলসিস্টেম relabel করতে হয় (`sudo touch /.autorelabel` তারপর রিবুট — বড় ডিস্কে অনেক সময় নেয়)।

> 💡 **টিপস:** পুরো সিস্টেম permissive না করে শুধু একটা সার্ভিসের domain permissive করা যায়: `sudo semanage permissive -a httpd_t` (সরাতে `-d`)। বাকি সিস্টেম সুরক্ষিত থাকে।

### Security Context — SELinux-এর "লেবেল"

SELinux-এ প্রতিটা ফাইল, প্রসেস, পোর্টের গায়ে একটা লেবেল থাকে, যাকে বলে **context**। দেখতে `-Z` অপশন:

```bash
$ ls -Z /var/www/html/index.html
system_u:object_r:httpd_sys_content_t:s0 /var/www/html/index.html

$ ps -eZ | grep httpd
system_u:system_r:httpd_t:s0       1234 ?        00:00:00 httpd

$ id -Z
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

Context-এর চারটা অংশ:

```
   system_u : object_r : httpd_sys_content_t : s0
   ────┬───   ───┬────   ─────────┬────────   ─┬─
       │         │                │            └─ level (MLS/MCS সংবেদনশীলতা স্তর)
       │         │                └─ TYPE  ★ সবচেয়ে গুরুত্বপূর্ণ
       │         └─ role
       └─ SELinux user (Linux user না!)
```

**Type Enforcement** — SELinux-এর মূল ধারণা: প্রসেসের type-কে বলে **domain** (যেমন `httpd_t`)। নীতিতে লেখা থাকে কোন domain কোন type-এর ফাইলে কী করতে পারবে:

```
 allow httpd_t httpd_sys_content_t : file { read open getattr };
 └─ httpd_t domain-এর প্রসেস httpd_sys_content_t type-এর ফাইল পড়তে পারবে
```

নীতিতে যা স্পষ্টভাবে `allow` নেই, তা নিষিদ্ধ (default deny)। তাই হ্যাক হওয়া Apache `/etc/shadow` (type `shadow_t`) বা `/home/rahim` (type `user_home_t`) পড়তে পারবে না — প্রসেসটা root হলেও।

```bash
$ ls -Z /etc/shadow
system_u:object_r:shadow_t:s0 /etc/shadow
```

### সবচেয়ে সাধারণ SELinux সমস্যা: ভুল label

একটা ক্লাসিক দৃশ্য:

```bash
$ echo "Hello" > ~/index.html
$ sudo mv ~/index.html /var/www/html/
$ curl http://localhost/index.html
<title>403 Forbidden</title>
```

পারমিশন তো ঠিক আছে, তাহলে 403 কেন?

```bash
$ ls -Z /var/www/html/index.html
unconfined_u:object_r:user_home_t:s0 /var/www/html/index.html
```

**কারণ:** `mv` ফাইলের পুরোনো label (`user_home_t`) সাথে নিয়ে এসেছে। Apache `user_home_t` পড়তে পারে না। (`cp` করলে নতুন জায়গার ডিফল্ট label পেত।)

**সমাধান — `restorecon`:** নীতিতে লেখা "এই পাথের ডিফল্ট label" ফিরিয়ে দেয়।

```bash
$ sudo restorecon -v /var/www/html/index.html
Relabeled /var/www/html/index.html from unconfined_u:object_r:user_home_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0

$ sudo restorecon -Rv /var/www/html     # পুরো ডিরেক্টরি, recursive
```

| `restorecon` অপশন | মানে |
|---|---|
| `-R` | recursive |
| `-v` | কী বদলাল দেখানো |
| `-n` | কিছু না বদলে শুধু দেখানো (dry run) |
| `-F` | user/role/level সহ পুরো context জোর করে রিসেট |

### নিজের পাথের জন্য স্থায়ী label: `semanage fcontext`

ধরো ওয়েবসাইট রাখবে `/srv/mysite`-এ। এই পাথ নীতিতে নেই, তাই label হবে `var_t` বা `default_t`, Apache পড়তে পারবে না।

```bash
# ভুল উপায় (অস্থায়ী): chcon — পরের restorecon বা relabel-এ মুছে যাবে
$ sudo chcon -R -t httpd_sys_content_t /srv/mysite

# সঠিক উপায়: নীতিতে নিয়ম যোগ, তারপর প্রয়োগ
$ sudo semanage fcontext -a -t httpd_sys_content_t "/srv/mysite(/.*)?"
$ sudo restorecon -Rv /srv/mysite

# নিয়মটা দেখা
$ sudo semanage fcontext -l | grep mysite
/srv/mysite(/.*)?    all files    system_u:object_r:httpd_sys_content_t:s0
```

> 📝 **মনে রাখো:** `chcon` = অস্থায়ী, সরাসরি ফাইলে লেবেল লেখে। `semanage fcontext` + `restorecon` = স্থায়ী, নীতিতে নিয়ম রাখে। সবসময় দ্বিতীয়টা ব্যবহার করো।

> 💡 **টিপস:** `semanage` কমান্ড না পেলে প্যাকেজ ইনস্টল করো: `sudo dnf install policycoreutils-python-utils`। আর `setroubleshoot-server` ইনস্টল করলে `sealert` দিয়ে সহজ ভাষায় সমস্যা ও সমাধান পাবে।

### SELinux Boolean — নীতির অন/অফ সুইচ

পুরো নীতি নতুন করে লেখা ছাড়াই কিছু সাধারণ আচরণ চালু/বন্ধ করার জন্য boolean আছে।

```bash
$ getsebool -a | grep httpd_can_network
httpd_can_network_connect --> off
httpd_can_network_connect_db --> off

# nginx/Apache যদি reverse proxy হিসেবে অন্য পোর্টে সংযোগ করতে চায়:
$ sudo setsebool -P httpd_can_network_connect on
```

`-P` না দিলে রিবুটে আগের মানে ফিরে যাবে। `semanage boolean -l` দিয়ে প্রতিটা boolean-এর বর্ণনা দেখা যায়।

### পোর্ট label

```bash
$ sudo semanage port -l | grep http_port_t
http_port_t     tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000

$ sudo semanage port -a -t http_port_t -p tcp 8088     # Apache-কে 8088 শোনার অনুমতি
```

### SELinux ডিবাগিং: AVC, `ausearch`, `audit2why`, `audit2allow`

SELinux কিছু আটকালে একটা **AVC** (Access Vector Cache) "denied" বার্তা লগ হয় — `auditd` চালু থাকলে `/var/log/audit/audit.log`-এ।

```bash
$ sudo ausearch -m AVC -ts recent
type=AVC msg=audit(1731400000.123:456): avc:  denied  { read } for  pid=1234
  comm="httpd" name="index.html" dev="dm-0" ino=393220
  scontext=system_u:system_r:httpd_t:s0
  tcontext=unconfined_u:object_r:user_home_t:s0 tclass=file permissive=0
```

পড়ার কৌশল: **কে** (`scontext` = `httpd_t`) **কী করতে চেয়েছিল** (`{ read }`) **কার ওপর** (`tcontext` = `user_home_t`, `tclass=file`)।

**`audit2why`** — কেন আটকানো হলো, সহজ ভাষায় বলে:

```bash
$ sudo ausearch -m AVC -ts recent | audit2why
type=AVC msg=audit(...): avc:  denied  { read } for ... comm="httpd" ...

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
```

boolean দিয়ে ঠিক হওয়ার মতো সমস্যা হলে `audit2why` সরাসরি বলে দেয় কোন boolean চালু করতে হবে — যেমন `setsebool -P httpd_can_network_connect 1`।

**`audit2allow`** — denial থেকে নতুন নীতি মডিউল বানায়:

```bash
$ sudo ausearch -m AVC -ts recent | audit2allow -M mywebfix
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i mywebfix.pp

$ cat mywebfix.te        # আগে পড়ে দেখো কী অনুমতি দিচ্ছ!
$ sudo semodule -i mywebfix.pp
$ sudo semodule -l | grep mywebfix     # লোড হয়েছে কি না
$ sudo semodule -r mywebfix            # সরাতে
```

> ⚠️ **সতর্কতা:** `audit2allow` হলো শেষ উপায়। উপরের উদাহরণে আসল সমস্যা ছিল ভুল label — `audit2allow` চালালে Apache-কে সব ইউজারের হোম ফাইল পড়ার অনুমতি দিয়ে বসতে! সমস্যা সমাধানের ক্রম মনে রাখো:

```
 SELinux denial হলো?
   │
   ├─► ১. label ঠিক আছে?          → না  → restorecon / semanage fcontext
   │
   ├─► ২. কোনো boolean আছে?       → হ্যাঁ → setsebool -P  (audit2why বলে দেবে)
   │
   ├─► ৩. পোর্ট label?            → না  → semanage port -a
   │
   └─► ৪. কিছুই না মিললে          → audit2allow দিয়ে ছোট, পড়ে-দেখা মডিউল
```

---

## ৪৫.১০ AppArmor

Ubuntu ও Debian-এ ডিফল্ট MAC হলো **AppArmor**। এখানে label-এর বদলে প্রতিটা প্রোগ্রামের জন্য একটা **profile** থাকে, যাতে পাথ দিয়ে লেখা থাকে সেই প্রোগ্রাম কোন ফাইলে কী করতে পারবে।

### অবস্থা দেখা: `aa-status`

```bash
$ sudo aa-status
apparmor module is loaded.
121 profiles are loaded.
33 profiles are in enforce mode.
   /usr/bin/man
   /usr/sbin/cupsd
   /usr/sbin/tcpdump
   lsb_release
   nvidia_modprobe
   ...
5 profiles are in complain mode.
   ...
4 processes have profiles defined.
4 processes are in enforce mode.
   /usr/sbin/cupsd (1102)
   ...
0 processes are in complain mode.
0 processes are unconfined but have a profile defined.
```

(`apparmor_status` একই কমান্ডের পুরোনো নাম।)

### দুই mode

| Mode | কাজ |
|---|---|
| **enforce** | নিয়ম চাপানো হয়, নিষিদ্ধ কাজ আটকানো ও লগ |
| **complain** | কিছু আটকানো হয় না, শুধু লগ (SELinux-এর permissive-এর মতো) |

### Profile দেখতে কেমন

Profile থাকে `/etc/apparmor.d/`-তে, নাম সাধারণত প্রোগ্রামের পাথের `/` বদলে `.` দিয়ে, যেমন `/usr/sbin/tcpdump` → `usr.sbin.tcpdump`।

```
# /etc/apparmor.d/usr.sbin.tcpdump (সংক্ষেপিত)
abi <abi/4.0>,
include <tunables/global>

profile tcpdump /usr/sbin/tcpdump {
  include <abstractions/base>
  include <abstractions/nameservice>

  capability net_raw,
  capability setuid,
  capability setgid,

  network raw,
  network packet,

  /usr/sbin/tcpdump mr,
  /etc/ld.so.cache r,
  # pcap ফাইল লেখা যাবে
  /**.[pP][cC][aA][pP] rw,
}
```

**পারমিশন অক্ষর:** `r` পড়া, `w` লেখা, `a` append, `m` মেমরিতে executable ম্যাপ, `k` lock, `l` link, `ix`/`px`/`cx`/`ux` — অন্য প্রোগ্রাম চালানোর নিয়ম (inherit / আলাদা profile / child / unconfined)।

### Profile ম্যানেজ করা

```bash
$ sudo apt install apparmor-utils apparmor-profiles   # aa-* টুল ও বাড়তি profile

$ sudo aa-complain /usr/sbin/tcpdump     # complain mode-এ
$ sudo aa-enforce  /usr/sbin/tcpdump     # enforce mode-এ
$ sudo aa-disable  /usr/sbin/tcpdump     # profile নিষ্ক্রিয়

# profile ফাইল হাতে বদলালে কার্নেলে আবার লোড
$ sudo apparmor_parser -r /etc/apparmor.d/usr.sbin.tcpdump
$ sudo systemctl reload apparmor          # সব profile রিলোড
```

### নতুন profile তৈরি

```bash
$ sudo aa-genprof /usr/local/bin/myapp
# এক টার্মিনালে aa-genprof চলবে, অন্য টার্মিনালে myapp ব্যবহার করো
# তারপর 'S' (Scan) চেপে লগ থেকে নিয়ম প্রস্তাব নাও, প্রতিটা Allow/Deny করো, শেষে 'F'

$ sudo aa-logprof     # পরে নতুন denial দেখে profile আপডেট
```

### AppArmor denial দেখা

```bash
$ sudo journalctl -k | grep -i 'apparmor="DENIED"'
audit: type=1400 audit(1731400000.555:88): apparmor="DENIED" operation="open"
  profile="/usr/sbin/tcpdump" name="/etc/shadow" pid=4321 comm="tcpdump"
  requested_mask="r" denied_mask="r" fsuid=0 ouid=0
```

`auditd` চালু থাকলে `/var/log/audit/audit.log`-এও আসে।

> 📝 **মনে রাখো:** Ubuntu 24.04 থেকে AppArmor দিয়ে **unprivileged user namespace** সীমিত করা হয়েছে (`kernel.apparmor_restrict_unprivileged_userns=1`)। কিছু অ্যাপ (যেমন কিছু AppImage/ব্রাউজার sandbox) এ কারণে চলতে না পারলে সেই অ্যাপের জন্য profile দরকার হয়। পুরো ফিচার বন্ধ করে দেওয়া নিরাপদ সমাধান না।

---

## ৪৫.১১ auditd — কে, কখন, কী করল

MAC আটকায়, কিন্তু অনেক সময় দরকার **হিসাব রাখা**: কে `/etc/passwd` বদলাল? কে রাত ২টায় `sudo` চালাল? এর জন্য লিনাক্সের **Audit framework**।

উপমা: ব্যাংকের CCTV ক্যামেরা। এটা চোর আটকায় না, কিন্তু পরে কী ঘটেছিল তার অকাট্য প্রমাণ রাখে।

```
   ইউজার স্পেস                       কার্নেল স্পেস
 ┌─────────────────┐   নিয়ম লোড    ┌──────────────────────────┐
 │ auditctl /      │ ────────────► │ kernel audit subsystem    │
 │ augenrules      │               │ (syscall, ফাইল অ্যাক্সেসে  │
 └─────────────────┘               │  hook — নিয়ম মিললে ইভেন্ট)  │
                                   └────────────┬─────────────┘
 ┌─────────────────┐   netlink সকেট             │
 │ auditd (daemon) │ ◄──────────────────────────┘
 └────────┬────────┘
          ▼
   /var/log/audit/audit.log ──► ausearch (খোঁজা) / aureport (রিপোর্ট)
```

### ইনস্টল

```bash
$ sudo apt install auditd audispd-plugins     # Debian / Ubuntu
$ sudo dnf install audit                      # RHEL / Fedora (সাধারণত আগে থেকেই আছে)
$ sudo systemctl enable --now auditd
```

> ⚠️ **সতর্কতা:** `auditd` সার্ভিস `systemctl restart/stop` দিয়ে বন্ধ করতে দেয় না (ইউনিট ফাইলে `RefuseManualStop=yes`) — যাতে আক্রমণকারী সহজে নজরদারি বন্ধ করতে না পারে। রিস্টার্ট দরকার হলে: `sudo service auditd restart`।

### নিয়ম (rules) — দুই ধরন

**১. ফাইল watch** (`-w`):

```bash
$ sudo auditctl -w /etc/passwd -p wa -k identity
$ sudo auditctl -w /etc/sudoers -p wa -k sudoers_change
$ sudo auditctl -w /etc/ssh/sshd_config -p wa -k sshd_config
```

| অংশ | মানে |
|---|---|
| `-w <path>` | এই ফাইল/ডিরেক্টরি নজরে রাখো |
| `-p rwxa` | কোন অ্যাক্সেস: `r` পড়া, `w` লেখা, `x` চালানো, `a` attribute বদল |
| `-k <key>` | পরে খোঁজার জন্য একটা নাম-ট্যাগ |

**২. syscall নিয়ম** (`-a`):

```bash
# সাধারণ ইউজার (auid>=1000) কোনো ফাইল মুছলে লগ
$ sudo auditctl -a always,exit -F arch=b64 -S unlink,unlinkat,rename,renameat \
    -F auid>=1000 -F auid!=unset -k delete
```

`auid` মানে **audit UID** — লগইনের সময় যে ইউজার ছিল। কেউ `sudo su -` করে root হলেও auid আসল ইউজারটাই থাকে। তাই "কে আসলে করেছে" ধরা যায়।

```bash
$ sudo auditctl -l          # বর্তমান নিয়ম দেখা
-w /etc/passwd -p wa -k identity
...
$ sudo auditctl -s          # অবস্থা (enabled, lost ইভেন্ট ইত্যাদি)
$ sudo auditctl -D          # সব নিয়ম মুছে ফেলা (অস্থায়ী)
```

### স্থায়ী নিয়ম

`auditctl` দিয়ে দেওয়া নিয়ম রিবুটে হারিয়ে যায়। স্থায়ী করতে `/etc/audit/rules.d/`-তে ফাইল বানাও:

```bash
$ sudo nano /etc/audit/rules.d/50-hardening.rules
```

```
-w /etc/passwd -p wa -k identity
-w /etc/group -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k sudoers_change
-w /etc/sudoers.d/ -p wa -k sudoers_change
-w /etc/ssh/sshd_config -p wa -k sshd_config
-w /var/log/lastlog -p wa -k logins
-a always,exit -F arch=b64 -S execve -F euid=0 -F auid>=1000 -F auid!=unset -k root_cmd
```

```bash
$ sudo augenrules --load    # rules.d-র সব ফাইল জোড়া দিয়ে audit.rules বানিয়ে লোড
```

> 💡 **টিপস:** একদম শেষ নিয়ম হিসেবে `-e 2` দিলে নিয়মগুলো **immutable** হয়ে যায় — রিবুট ছাড়া কেউ (root-ও না) বদলাতে পারবে না। প্রোডাকশনের জন্য ভালো, কিন্তু টেস্টের সময় দিও না।

### লগ খোঁজা ও রিপোর্ট

```bash
$ sudo ausearch -k identity -i
----
type=PROCTITLE msg=audit(11/12/2026 10:45:12.331:812) : proctitle=useradd hacker
type=PATH msg=audit(...) : item=0 name=/etc/passwd ...
type=SYSCALL msg=audit(...) : arch=x86_64 syscall=openat success=yes ...
  auid=rahim uid=root comm=useradd exe=/usr/sbin/useradd key=identity
```

এখান থেকে স্পষ্ট: `rahim` (auid) root হয়ে `useradd hacker` চালিয়েছে।

| কমান্ড | কাজ |
|---|---|
| `ausearch -k <key>` | key দিয়ে খোঁজা |
| `ausearch -i` | সংখ্যা (UID, syscall) মানুষের পড়ার মতো করে দেখানো |
| `ausearch -ua 1000` | নির্দিষ্ট UID/auid-এর ইভেন্ট |
| `ausearch -m USER_LOGIN --success no` | ব্যর্থ লগইন |
| `ausearch -ts today` / `-ts recent` | আজকের / গত ১০ মিনিটের |
| `aureport --summary` | সারাংশ |
| `aureport -au` | অথেন্টিকেশন রিপোর্ট |
| `aureport -l --failed` | ব্যর্থ লগইন রিপোর্ট |
| `aureport -x --summary` | কোন প্রোগ্রাম কতবার চলেছে |

কনফিগ `/etc/audit/auditd.conf`-এ লগের আকার ও ঘোরানো (`max_log_file`, `num_logs`, `max_log_file_action`) ঠিক করা যায়। (সাধারণ লগ ব্যবস্থার জন্য দেখো [অধ্যায় ২৮](28-logs.md)।)

---

## ৪৫.১২ AIDE — ফাইল ইন্টেগ্রিটি যাচাই

আক্রমণকারী সিস্টেমে ঢুকে প্রায়ই `/bin/ls`, `/usr/bin/ps`, `/usr/sbin/sshd`-এর মতো ফাইল বদলে দেয় (যাতে নিজের প্রসেস লুকানো যায় বা পাসওয়ার্ড চুরি করা যায়)। **AIDE** (Advanced Intrusion Detection Environment) এমন পরিবর্তন ধরে।

**কীভাবে কাজ করে:** পরিষ্কার অবস্থায় প্রতিটা গুরুত্বপূর্ণ ফাইলের "আঙুলের ছাপ" (hash, size, permission, owner, inode, mtime) একটা ডেটাবেসে রাখে। পরে আবার মিলিয়ে দেখে কী বদলেছে।

উপমা: বাড়ি থেকে বের হওয়ার আগে প্রতিটা ঘরের ছবি তুলে রাখলে, ফিরে এসে ছবির সাথে মিলিয়ে বোঝা যায় কিছু সরানো হয়েছে কি না।

```
  পরিষ্কার সিস্টেম                          পরে (নিয়মিত)
 ┌───────────────┐   aide --init    ┌──────────────────┐
 │ /bin, /etc,   │ ───────────────► │ aide.db (baseline)│──┐
 │ /usr/sbin ... │                  └──────────────────┘  │ তুলনা
 └───────────────┘                                        ▼
                      aide --check  ┌─────────────────────────────┐
     বর্তমান ফাইল ─────────────────►│ পার্থক্য রিপোর্ট:              │
                                    │ added / removed / changed   │
                                    └─────────────────────────────┘
```

### RHEL / Fedora

```bash
$ sudo dnf install aide
$ sudo aide --init                 # baseline তৈরি (কয়েক মিনিট লাগে)
AIDE initialized database at /var/lib/aide/aide.db.new.gz
$ sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz

$ sudo aide --check
AIDE found NO differences between database and filesystem. Looks okay!!
```

### Debian / Ubuntu

```bash
$ sudo apt install aide
$ sudo aideinit                    # ডেটাবেস বানায় ও /var/lib/aide/aide.db-তে কপি করে
$ sudo aide --config /etc/aide/aide.conf --check
```

Debian প্যাকেজ একটা দৈনিক চেক (cron/systemd timer `dailyaidecheck`) সাথে দেয়।

### কিছু বদলালে

```bash
$ sudo aide --check
AIDE found differences between database and filesystem!!
Summary:
  Total number of entries:      145032
  Added entries:                0
  Removed entries:              0
  Changed entries:              1
---------------------------------------------------
Changed entries:
---------------------------------------------------
f   ...    .C... : /usr/bin/ls
---------------------------------------------------
Detailed information about changes:
---------------------------------------------------
File: /usr/bin/ls
  SHA512   : 3bQ1...                         | 9xZk...
```

বৈধ পরিবর্তনের (যেমন প্যাকেজ আপডেট) পর baseline হালনাগাদ:

```bash
$ sudo aide --update               # নতুন ডেটাবেস বানায় (aide.db.new.gz)
$ sudo mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
```

কী নজরে রাখবে তা ঠিক হয় `/etc/aide.conf` (RHEL) বা `/etc/aide/aide.conf` + `/etc/aide/aide.conf.d/` (Debian) ফাইলে।

> ⚠️ **সতর্কতা:** আক্রমণকারী root পেলে AIDE-এর ডেটাবেসও বদলে দিতে পারে! তাই baseline ডেটাবেসের একটা কপি **অফলাইনে / রিড-অনলি মিডিয়ায়** রাখো, আর আসল চেক সম্ভব হলে বিশ্বস্ত মিডিয়া থেকে চালাও।

> 📝 **মনে রাখো:** Debian/Ubuntu-তে প্যাকেজের ফাইল দ্রুত যাচাইয়ের জন্য `debsums` (`sudo debsums -c`) আর RHEL/Fedora-তে `rpm -Va` আছে — এরা প্যাকেজ ডেটাবেসের hash-এর সাথে মিলিয়ে দেখে।

---

## ৪৫.১৩ rkhunter — rootkit খোঁজা

**Rootkit** হলো এমন ম্যালওয়্যার যা root অ্যাক্সেস পাওয়ার পর নিজেকে লুকিয়ে রাখে — `ps`-এ নিজের প্রসেস দেখায় না, `ls`-এ নিজের ফাইল দেখায় না, কখনো কার্নেল মডিউল হিসেবে কার্নেলের ভেতরেই বসে থাকে।

**rkhunter** (Rootkit Hunter) যা পরীক্ষা করে:
- পরিচিত rootkit-এর ফাইল ও সিগনেচার
- গুরুত্বপূর্ণ কমান্ডের (`ls`, `ps`, `netstat`...) hash ও পারমিশন বদলেছে কি না
- লুকানো ফাইল, সন্দেহজনক কার্নেল মডিউল, অদ্ভুত খোলা পোর্ট
- SSH কনফিগ (যেমন root লগইন চালু কি না)

```bash
$ sudo apt install rkhunter        # Debian / Ubuntu
$ sudo dnf install rkhunter        # Fedora; RHEL-এ EPEL থেকে

$ sudo rkhunter --update           # সিগনেচার ডেটাবেস আপডেট
$ sudo rkhunter --propupd          # বর্তমান ফাইলগুলোর property baseline
$ sudo rkhunter --check --sk       # স্ক্যান (--sk = প্রতিটা ধাপে Enter চাপতে হবে না)
...
System checks summary
=====================
File properties checks...
    Files checked: 142
    Suspect files: 0

Rootkit checks...
    Rootkits checked : 498
    Possible rootkits: 0

Applications checks...
    All checks skipped

The system checks took: 1 minute and 52 seconds

All results have been written to the log file: /var/log/rkhunter.log
```

| অপশন | কাজ |
|---|---|
| `--update` | ডেটা ফাইল আপডেট |
| `--propupd` | ফাইল property baseline আপডেট (সিস্টেম আপডেটের পর) |
| `-c`, `--check` | স্ক্যান চালানো |
| `--sk`, `--skip-keypress` | কী চাপার অপেক্ষা না করা |
| `--rwo`, `--report-warnings-only` | শুধু সতর্কতা দেখানো |
| `--cronjob` | cron-এর জন্য (রঙ ও কী-চাপা ছাড়া) |

```bash
$ sudo rkhunter --check --sk --rwo
Warning: The SSH and rkhunter configuration options should be the same:
         SSH configuration option 'PermitRootLogin': no
         Rkhunter configuration option 'ALLOW_SSH_ROOT_USER': unset
```

> 💡 **টিপস:** rkhunter-এ অনেক **false positive** (ভুল সতর্কতা) আসে, বিশেষত প্যাকেজ আপডেটের পর। প্রতিটা সতর্কতা `/var/log/rkhunter.log`-এ পড়ে দেখো, আর বৈধ আপডেটের পর `--propupd` চালাও। কনফিগ: `/etc/rkhunter.conf` (Debian-এ `/etc/default/rkhunter`-এ দৈনিক চালানোর সেটিং)। বিকল্প টুল: `chkrootkit`।

> ⚠️ **সতর্কতা:** সত্যিকারের rootkit সন্দেহ হলে সংক্রমিত সিস্টেমের নিজের টুলের ওপর ভরসা করা যায় না — rootkit সেগুলোকেও ধোঁকা দিতে পারে। সবচেয়ে নিরাপদ পথ: সিস্টেম নেটওয়ার্ক থেকে আলাদা করো, live USB থেকে বুট করে ডিস্ক পরীক্ষা করো, ডেটা ব্যাকআপ নিয়ে **নতুন করে ইনস্টল** করো। পাসওয়ার্ড ও key সব বদলাও।

---

## ৪৫.১৪ ডাউনলোড যাচাই — checksum ও GPG

ধরো Ubuntu-র ISO ডাউনলোড করলে। দুটো প্রশ্ন:
1. **ফাইলটা কি নষ্ট হয়নি?** (ডাউনলোডে ভুল) → **checksum** দিয়ে যাচাই
2. **ফাইলটা কি সত্যিই Ubuntu-র, নাকি কেউ মাঝপথে বদলে দিয়েছে?** → **GPG signature** দিয়ে যাচাই

### Hash / Checksum কী?

**Hash function** যেকোনো আকারের ডেটা থেকে একটা নির্দিষ্ট দৈর্ঘ্যের "আঙুলের ছাপ" বানায়। ফাইলের একটা বিটও বদলালে hash পুরো বদলে যায়।

```bash
$ echo "hello" | sha256sum
5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03  -
$ echo "hellp" | sha256sum
# সম্পূর্ণ ভিন্ন একটা মান আসবে — একটা অক্ষর বদলালেই
```

| টুল | অ্যালগরিদম | এখন নিরাপদ? |
|---|---|---|
| `md5sum` | MD5 (128-bit) | ❌ ভাঙা — শুধু দুর্ঘটনাজনিত নষ্ট ধরার জন্য |
| `sha1sum` | SHA-1 (160-bit) | ❌ collision দেখানো হয়েছে |
| `sha256sum` | SHA-256 | ✅ |
| `sha512sum` | SHA-512 | ✅ |
| `b2sum` | BLAKE2b | ✅ |

### `sha256sum` ব্যবহার

```bash
# একটা ফাইলের hash
$ sha256sum ubuntu-24.04.1-desktop-amd64.iso
c2e6f4dc37ac944e2ed507f87c6188dd4d3179bf4a3f9e110d3c88d1f3294bdc  ubuntu-24.04.1-desktop-amd64.iso
# (উপরের hash উদাহরণমাত্র — আসল মান Ubuntu-র SHA256SUMS ফাইলে দেখো)

# প্রকাশকের দেওয়া SHA256SUMS ফাইল দিয়ে স্বয়ংক্রিয় মিলানো
$ sha256sum -c SHA256SUMS --ignore-missing
ubuntu-24.04.1-desktop-amd64.iso: OK
```

| অপশন | কাজ |
|---|---|
| `-c FILE` | FILE-এ লেখা hash-এর সাথে মিলিয়ে দেখা |
| `--ignore-missing` | তালিকার যেসব ফাইল ডাউনলোড করোনি, সেগুলো উপেক্ষা |
| `--quiet` | শুধু ব্যর্থতা দেখানো |
| `--status` | কিছু না দেখিয়ে শুধু exit code (স্ক্রিপ্টের জন্য) |

> ⚠️ **সতর্কতা:** checksum একা **যথেষ্ট না**। আক্রমণকারী যদি ওয়েবসাইট হ্যাক করে ISO বদলাতে পারে, তাহলে পাশের `SHA256SUMS` ফাইলটাও বদলে দিতে পারে! এজন্যই দরকার GPG signature।

### GPG — ডিজিটাল স্বাক্ষর

**GPG** (GNU Privacy Guard) **public-key cryptography** ব্যবহার করে। প্রত্যেকের দুটো key:
- **Private key** — নিজের কাছে গোপন রাখো, এটা দিয়ে **সাইন** করো (আর ডিক্রিপ্ট)।
- **Public key** — সবাইকে দাও, এটা দিয়ে অন্যরা **স্বাক্ষর যাচাই** করে (আর তোমার জন্য এনক্রিপ্ট)।

উপমা: private key হলো তোমার হাতের সই করার ক্ষমতা, যা কেউ নকল করতে পারে না। public key হলো ব্যাংকে জমা রাখা তোমার সইয়ের নমুনা, যা দেখে যে কেউ মিলিয়ে নিতে পারে।

```
  প্রকাশক (Ubuntu)                                তুমি
 ┌──────────────────┐                     ┌────────────────────────┐
 │ SHA256SUMS ফাইল   │──────────┐          │ ১. public key আনো        │
 │       +           │          │          │    (fingerprint মিলিয়ে!) │
 │ private key দিয়ে  │          ▼          │ ২. gpg --verify          │
 │ সাইন              │──► SHA256SUMS.gpg ─►│    → "Good signature"    │
 └──────────────────┘   (স্বাক্ষর)          │ ৩. sha256sum -c          │
                                          │    → ISO: OK              │
                                          └────────────────────────┘
   বিশ্বাসের শৃঙ্খল: key-এর fingerprint → স্বাক্ষর → SHA256SUMS → ISO
```

### ধাপে ধাপে ISO যাচাই (Ubuntu উদাহরণ)

```bash
# ১. তিনটা ফাইল একই ফোল্ডারে: ISO, SHA256SUMS, SHA256SUMS.gpg

# ২. স্বাক্ষর যাচাইয়ের চেষ্টা — প্রথমবার key নেই বলে কোন key লাগবে তা বলবে
$ gpg --keyid-format long --verify SHA256SUMS.gpg SHA256SUMS
gpg: Signature made ...
gpg:                using RSA key <KEY-ID>
gpg: Can't check signature: No public key

# ৩. ঐ key keyserver থেকে আনো
$ gpg --keyid-format long --keyserver hkps://keyserver.ubuntu.com --recv-keys <KEY-ID>

# ৪. key-এর fingerprint দেখো এবং প্রকাশকের অফিসিয়াল ডকুমেন্টেশনের সাথে মিলাও
$ gpg --fingerprint <KEY-ID>

# ৫. আবার যাচাই
$ gpg --keyid-format long --verify SHA256SUMS.gpg SHA256SUMS
gpg: Good signature from "Ubuntu CD Image Automatic Signing Key (2012) <cdimage@ubuntu.com>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.

# ৬. এবার checksum
$ sha256sum -c SHA256SUMS --ignore-missing
ubuntu-24.04.1-desktop-amd64.iso: OK
```

> 📝 **মনে রাখো:** "**Good signature**" মানে ফাইলটা ঐ key দিয়েই সাইন করা। "WARNING: not certified" মানে তুমি নিজে এখনো ঐ key-কে "বিশ্বস্ত" হিসেবে চিহ্নিত করোনি — এটা স্বাভাবিক, **যদি তুমি fingerprint প্রকাশকের অফিসিয়াল সূত্রের সাথে মিলিয়ে থাকো**। কিন্তু "**BAD signature**" দেখলে ফাইলটা কখনো ব্যবহার কোরো না।

### নিজের GPG key ও দৈনন্দিন ব্যবহার

```bash
$ gpg --full-generate-key                 # নতুন key (ডিফল্ট ECC/ed25519 বা RSA বেছে নাও)
$ gpg --list-keys                         # public key তালিকা
$ gpg --list-secret-keys --keyid-format long
$ gpg --armor --export rahim@example.com > rahim.pub.asc   # public key শেয়ারের জন্য
$ gpg --import someone.pub.asc            # অন্যের public key যোগ

$ gpg --detach-sign --armor report.pdf    # report.pdf.asc স্বাক্ষর তৈরি
$ gpg --verify report.pdf.asc report.pdf  # যাচাই

$ gpg -e -r karim@example.com secret.txt  # karim-এর জন্য এনক্রিপ্ট → secret.txt.gpg
$ gpg -d secret.txt.gpg > secret.txt      # ডিক্রিপ্ট
$ gpg -c backup.tar                       # পাসওয়ার্ড দিয়ে symmetric এনক্রিপ্ট
$ gpg --gen-revoke rahim@example.com > revoke.asc   # key হারালে বাতিলের সার্টিফিকেট (আগেই বানিয়ে রাখো)
```

> 📝 **মনে রাখো:** প্যাকেজ ম্যানেজারও ঠিক এই কাজটাই প্রতিবার নিজে নিজে করে — `apt` রিপোজিটরির `Release` ফাইলের GPG স্বাক্ষর যাচাই করে (key থাকে `/etc/apt/keyrings/` বা `/usr/share/keyrings/`-এ), আর `dnf`/`rpm` প্রতিটা RPM-এর স্বাক্ষর (`gpgcheck=1`)। তাই কোনো রিপোতে `gpgcheck=0` বা `[trusted=yes]` দেখলে সতর্ক হও ([অধ্যায় ৩০](30-package-management.md))।

---

## ৪৫.১৫ Lynis — পুরো সিস্টেমের নিরাপত্তা অডিট

এতক্ষণ যা শিখলাম, সব হাতে মিলিয়ে দেখা কষ্টসাধ্য। **Lynis** একটা ওপেন-সোর্স অডিট টুল যা শত শত পরীক্ষা চালিয়ে বলে দেয় কোথায় দুর্বলতা আছে আর কী করা উচিত। এটা নিজে কিছু বদলায় না — শুধু রিপোর্ট দেয়।

```bash
$ sudo apt install lynis           # Debian / Ubuntu
$ sudo dnf install lynis           # Fedora; RHEL-এ EPEL থেকে

$ sudo lynis audit system
[ Lynis 3.x ]
...
[+] Boot and services
------------------------------------
  - Service Manager                                           [ systemd ]
  - Checking UEFI boot                                        [ ENABLED ]
  - Checking Secure Boot                                      [ ENABLED ]
...
[+] SSH Support
------------------------------------
  - Checking running SSH daemon                               [ FOUND ]
    - OpenSSH option: PermitRootLogin                         [ OK ]
    - OpenSSH option: PasswordAuthentication                  [ OK ]
    - OpenSSH option: AllowTcpForwarding                      [ OK ]
...
  -[ Lynis 3.x Results ]-

  Warnings (1):
  ----------------------------
  ! Found one or more vulnerable packages. [PKGS-7392]
      https://cisofy.com/lynis/controls/PKGS-7392/

  Suggestions (34):
  ----------------------------
  * Install a PAM module for password strength testing like pam_cracklib or pam_passwdqc [AUTH-9262]
  * Consider hardening SSH configuration [SSH-7408]
    - Details  : MaxSessions (10 --> 2)
  ...

  Lynis security scan details:

  Hardening index : 67 [#############       ]
  Tests performed : 262
  Plugins enabled : 0

  - Test and debug information      : /var/log/lynis.log
  - Report data                     : /var/log/lynis-report.dat
```

**কীভাবে পড়বে:**
- **Warnings** — জরুরি, আগে ঠিক করো।
- **Suggestions** — উন্নতির পরামর্শ; প্রতিটা অন্ধভাবে মানতে হবে না, তোমার সিস্টেমের প্রয়োজন বুঝে সিদ্ধান্ত নাও।
- **Hardening index** — ০–১০০, একটা মোটামুটি নির্দেশক। লক্ষ্য হলো সময়ের সাথে বাড়ানো, ১০০ পাওয়া না।
- বন্ধনীর কোড (যেমন `[SSH-7408]`) হলো test ID।

| কমান্ড | কাজ |
|---|---|
| `lynis audit system` | পূর্ণ অডিট |
| `lynis audit system --quick` | মাঝে থামবে না |
| `lynis audit system --cronjob` | cron-এর জন্য (রঙ ও বিরতি ছাড়া) |
| `lynis show details SSH-7408` | একটা টেস্টের বিস্তারিত |
| `lynis show commands` | সব কমান্ড |
| `lynis show version` | ভার্সন |
| `lynis update info` | ভার্সন হালনাগাদ কি না |

> 💡 **টিপস:** distro-র রিপোতে Lynis প্রায়ই পুরোনো ভার্সন থাকে। সর্বশেষ ভার্সনের জন্য CISOfy-র অফিসিয়াল রিপো বা GitHub ব্যবহার করা যায়। কাজের ধারা: অডিট → কয়েকটা জিনিস ঠিক করো → আবার অডিট → hardening index তুলনা।

---

## ৪৫.১৬ আরও কিছু হার্ডেনিং কৌশল (সংক্ষেপে)

এগুলোর বিস্তারিত অন্য অধ্যায়ে আছে, কিন্তু একটা হার্ডেনিং চেকলিস্টে অবশ্যই থাকা উচিত:

**কার্নেল প্যারামিটার (sysctl)** — `/etc/sysctl.d/99-hardening.conf` ([অধ্যায় ৪৭](47-kernel-proc-sys.md)):

```ini
net.ipv4.tcp_syncookies = 1               # SYN flood প্রতিরোধ
net.ipv4.conf.all.rp_filter = 1           # spoofed সোর্স IP বাদ
net.ipv4.conf.all.accept_redirects = 0    # ICMP redirect গ্রহণ না
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
kernel.kptr_restrict = 2                  # কার্নেল pointer লুকানো
kernel.dmesg_restrict = 1                 # সাধারণ ইউজার dmesg পড়তে পারবে না
fs.protected_symlinks = 1
fs.protected_hardlinks = 1
```

```bash
$ sudo sysctl --system      # সব .conf ফাইল লোড
```

**মাউন্ট অপশন** — `/tmp` বা `/dev/shm`-এ `nodev,nosuid,noexec` দিলে সেখান থেকে প্রোগ্রাম চালানো বা SUID কাজে লাগানো কঠিন হয় ([অধ্যায় ৩২](32-disk-partition-mount.md))।

**SUID ফাইল অডিট** ([অধ্যায় ১৭](17-special-permissions-acl.md)):

```bash
$ sudo find / -xdev -perm -4000 -type f 2>/dev/null
```

**অন্যান্য:**
- ফায়ারওয়াল ডিফল্ট "deny incoming" ([অধ্যায় ৩৯](39-firewall.md))
- `sudo` কনফিগ সীমিত রাখা, `NOPASSWD` এড়ানো ([অধ্যায় ১৮](18-sudo-root.md))
- ডিস্ক এনক্রিপশন (LUKS) — ল্যাপটপ চুরি হলে ডেটা রক্ষা
- UEFI Secure Boot ও BIOS/GRUB পাসওয়ার্ড — শারীরিক অ্যাক্সেস থেকে সুরক্ষা ([অধ্যায় ২৭](27-boot-process.md))
- নিয়মিত, পরীক্ষিত, অফলাইন ব্যাকআপ — ransomware-এর শেষ প্রতিরক্ষা ([অধ্যায় ৩৫](35-backup-rsync.md))
- কেন্দ্রীয় লগ সংগ্রহ — হ্যাকার লোকাল লগ মুছে দিলেও কপি থাকবে ([অধ্যায় ২৮](28-logs.md))
- প্রতিষ্ঠানে বড় পরিসরে: **CIS Benchmark** ও **OpenSCAP** (`oscap`) দিয়ে মানদণ্ড অনুযায়ী স্বয়ংক্রিয় যাচাই

### নিরাপত্তা সংক্রান্ত দরকারি শর্টকাট

| শর্টকাট / কৌশল | কাজ |
|---|---|
| `Super+L` (GNOME/KDE) | স্ক্রিন লক |
| `Ctrl+Alt+Delete` | systemd-এ ডিফল্টে রিবুট! সার্ভারে বন্ধ করতে: `sudo systemctl mask ctrl-alt-del.target` |
| `Enter` `~.` | আটকে যাওয়া SSH সেশন বন্ধ |
| কমান্ডের আগে একটা Space | `HISTCONTROL=ignorespace` থাকলে ঐ কমান্ড history-তে যায় না (পাসওয়ার্ডসহ কমান্ডের জন্য; [অধ্যায় ৬](06-terminal-shortcuts.md)) |
| `Ctrl+D` / `exit` | কাজ শেষে root শেল সঙ্গে সঙ্গে ছেড়ে দাও |
| `less`-এ `/denied` তারপর `n` | বড় লগে denial খোঁজা ও পরেরটায় যাওয়া |
| `journalctl -f` + `Ctrl+C` | লাইভ লগ দেখা ও বন্ধ |

---

## হাতে-কলমে অনুশীলন

> ⚠️ এই ল্যাবগুলো একটা **ভার্চুয়াল মেশিনে** (VirtualBox/KVM, [অধ্যায় ৪৮](48-containers-virtualization.md)) করো, যার snapshot নেওয়া আছে। আসল সার্ভারে প্রথমবার পরীক্ষা কোরো না।

### ল্যাব ১: আপডেট স্বয়ংক্রিয় করা (Ubuntu/Debian)

```bash
$ sudo apt update && sudo apt upgrade -y
$ sudo apt install -y unattended-upgrades
$ sudo dpkg-reconfigure -plow unattended-upgrades     # "Yes"
$ cat /etc/apt/apt.conf.d/20auto-upgrades
$ sudo unattended-upgrade --dry-run --debug 2>&1 | tail -20
$ systemctl list-timers 'apt-daily*'
$ ls /var/run/reboot-required 2>/dev/null && echo "রিবুট দরকার" || echo "রিবুট লাগবে না"
```

### ল্যাব ২: পাসওয়ার্ড নীতি

```bash
$ sudo apt install -y libpam-pwquality
$ echo -e "minlen = 12\nminclass = 3" | sudo tee /etc/security/pwquality.conf.d/lab.conf
$ sudo useradd -m testuser
$ sudo passwd testuser          # "abc" দিয়ে চেষ্টা — BAD PASSWORD আসবে (root হিসেবে শুধু সতর্কতা দেখাবে)
$ sudo chage -M 90 -W 7 testuser
$ sudo chage -l testuser
```

> 💡 root হিসেবে `passwd` চালালে দুর্বল পাসওয়ার্ডে সতর্কতা দেখিয়েও মেনে নেয় (`enforce_for_root` না দিলে)। আসল প্রভাব দেখতে `su - testuser` হয়ে `passwd` চালাও।

### ল্যাব ৩: SSH হার্ডেনিং

```bash
# ১. নিজের key আছে কি না নিশ্চিত হও (ক্লায়েন্ট থেকে ssh-copy-id আগে করো)
$ sudo groupadd sshusers
$ sudo usermod -aG sshusers $USER

# ২. কনফিগ
$ sudo tee /etc/ssh/sshd_config.d/00-hardening.conf >/dev/null <<'CONF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowGroups sshusers
CONF

# ৩. যাচাই ও রিলোড
$ sudo sshd -t && sudo sshd -T | grep -Ei 'permitroot|passwordauth|allowgroups'
$ sudo systemctl reload ssh

# ৪. পুরোনো সেশন খোলা রেখে নতুন টার্মিনাল থেকে লগইন টেস্ট করো
#    পাসওয়ার্ড দিয়ে চেষ্টা করলে: Permission denied (publickey).
```

### ল্যাব ৪: fail2ban

```bash
$ sudo apt install -y fail2ban
$ sudo tee /etc/fail2ban/jail.local >/dev/null <<'CONF'
[DEFAULT]
bantime = 10m
findtime = 10m
maxretry = 3
backend = systemd

[sshd]
enabled = true
CONF
$ sudo systemctl enable --now fail2ban
$ sudo fail2ban-client status sshd
# অন্য একটা VM থেকে ইচ্ছে করে ৩–৪ বার ভুল ইউজারনেমে ssh করো, তারপর:
$ sudo fail2ban-client status sshd        # Banned IP list-এ ঐ IP
$ sudo fail2ban-client set sshd unbanip <ঐ-IP>
```

### ল্যাব ৫: পোর্ট ও সার্ভিস অডিট

```bash
$ sudo ss -tulpn
$ systemctl list-unit-files --type=service --state=enabled
# প্রতিটা listening পোর্টের পাশে লিখে ফেলো: দরকার? বাইরে থেকে দরকার?
$ sudo systemctl disable --now cups.service cups.socket 2>/dev/null
$ sudo ss -tulpn          # আবার দেখো, কী কমল
```

### ল্যাব ৬: SELinux (Fedora / RHEL / Rocky VM)

```bash
$ getenforce
$ sudo dnf install -y httpd policycoreutils-python-utils setroubleshoot-server
$ sudo systemctl enable --now httpd
$ echo "Hello SELinux" > ~/index.html
$ sudo mv ~/index.html /var/www/html/
$ curl -s localhost | head -3          # 403 Forbidden
$ ls -Z /var/www/html/index.html       # user_home_t
$ sudo ausearch -m AVC -ts recent | audit2why
$ sudo restorecon -v /var/www/html/index.html
$ curl -s localhost                    # Hello SELinux

# নিজের ডিরেক্টরি
$ sudo mkdir -p /srv/mysite && echo "srv site" | sudo tee /srv/mysite/index.html
$ ls -Z /srv/mysite/index.html
$ sudo semanage fcontext -a -t httpd_sys_content_t "/srv/mysite(/.*)?"
$ sudo restorecon -Rv /srv/mysite
$ getsebool httpd_can_network_connect
```

### ল্যাব ৭: AppArmor (Ubuntu VM)

```bash
$ sudo aa-status | head -5
$ sudo apt install -y apparmor-utils
$ sudo aa-complain /usr/sbin/tcpdump 2>/dev/null || echo "tcpdump profile নেই"
$ sudo aa-status | grep -A3 complain
$ sudo aa-enforce /usr/sbin/tcpdump
```

### ল্যাব ৮: auditd

```bash
$ sudo apt install -y auditd
$ echo "-w /etc/passwd -p wa -k identity" | sudo tee /etc/audit/rules.d/50-lab.rules
$ sudo augenrules --load
$ sudo auditctl -l
$ sudo useradd labuser2
$ sudo ausearch -k identity -i | tail -20     # auid তোমার নাম দেখাবে
$ sudo aureport --summary
```

### ল্যাব ৯: AIDE, rkhunter, Lynis

```bash
$ sudo apt install -y aide rkhunter lynis
$ sudo aideinit                                # সময় লাগবে
$ sudo touch /usr/local/bin/suspicious && sudo chmod 4755 /usr/local/bin/suspicious
$ sudo aide --config /etc/aide/aide.conf --check | head -30   # /usr/local নজরে থাকলে নতুন SUID ফাইলটা "Added entries"-এ আসবে
$ sudo rm /usr/local/bin/suspicious

$ sudo rkhunter --update ; sudo rkhunter --propupd
$ sudo rkhunter --check --sk --rwo

$ sudo lynis audit system --quick | tail -30   # hardening index লিখে রাখো
```

### ল্যাব ১০: ডাউনলোড যাচাই

```bash
$ mkdir ~/verify && cd ~/verify
$ echo "important data" > file.txt
$ sha256sum file.txt > SHA256SUMS
$ sha256sum -c SHA256SUMS                  # file.txt: OK
$ echo "tampered" >> file.txt
$ sha256sum -c SHA256SUMS                  # file.txt: FAILED

$ gpg --quick-generate-key "Lab User <lab@example.com>" default default 1y
$ gpg --detach-sign --armor SHA256SUMS
$ gpg --verify SHA256SUMS.asc SHA256SUMS   # Good signature
$ echo "x" >> SHA256SUMS
$ gpg --verify SHA256SUMS.asc SHA256SUMS   # BAD signature
```

---

## সারসংক্ষেপ

| বিষয় | মূল কমান্ড / ফাইল | এক লাইনে |
|---|---|---|
| আপডেট (Debian/Ubuntu) | `apt upgrade`, `unattended-upgrades`, `/etc/apt/apt.conf.d/50unattended-upgrades` | সিকিউরিটি আপডেট স্বয়ংক্রিয় |
| আপডেট (RHEL/Fedora) | `dnf upgrade --security`, `dnf-automatic.timer`, `needs-restarting -r` | একই কাজ |
| পাসওয়ার্ড নীতি | `/etc/security/pwquality.conf`, `chage -l/-M/-W` | শক্তি ও মেয়াদ |
| PAM | `/etc/pam.d/`, `auth/account/password/session`, `faillock` | অথেন্টিকেশনের সুইচবোর্ড |
| SSH | `/etc/ssh/sshd_config.d/00-*.conf`, `sshd -t`, `sshd -T` | key-only, root বন্ধ |
| fail2ban | `jail.local`, `fail2ban-client status sshd`, `unbanip` | brute-force ব্যান |
| সার্ভিস | `systemctl list-unit-files --state=enabled`, `disable --now`, `mask` | যা লাগে না তা বন্ধ |
| পোর্ট | `ss -tulpn`, `lsof -i`, `nmap -sV` | কোন দরজা খোলা |
| SELinux | `getenforce`, `sestatus`, `setenforce 0/1`, `ls -Z`, `restorecon -Rv`, `semanage fcontext/port/boolean`, `setsebool -P`, `ausearch -m AVC`, `audit2why`, `audit2allow` | label-ভিত্তিক MAC |
| AppArmor | `aa-status`, `aa-enforce`, `aa-complain`, `apparmor_parser -r`, `aa-genprof`, `aa-logprof` | path-ভিত্তিক MAC |
| auditd | `auditctl -w ... -p wa -k`, `/etc/audit/rules.d/`, `augenrules --load`, `ausearch -k -i`, `aureport` | কে কী করল |
| AIDE | `aide --init`, `aide --check`, `aide --update`, `aideinit` | ফাইল বদল ধরা |
| rkhunter | `rkhunter --update`, `--propupd`, `--check --sk --rwo` | rootkit খোঁজা |
| checksum | `sha256sum`, `sha256sum -c SHA256SUMS --ignore-missing` | ফাইল অক্ষত কি না |
| GPG | `gpg --verify`, `--recv-keys`, `--fingerprint`, `--detach-sign`, `-e -r`, `-c` | উৎস আসল কি না |
| Lynis | `lynis audit system`, `lynis show details <ID>` | সামগ্রিক অডিট |

---

## নিজেকে যাচাই করো

1. "Defense in depth" বলতে কী বোঝায়? অন্তত চারটা স্তরের নাম ও প্রতিটার একটা করে টুল বলো।
2. Ubuntu-তে `unattended-upgrades` কোন দুটো কনফিগ ফাইল ব্যবহার করে, আর প্রতিটার কাজ কী?
3. PAM-এ `required` আর `requisite`-এর মধ্যে পার্থক্য কী?
4. তুমি `/etc/ssh/sshd_config.d/99-my.conf`-এ `PasswordAuthentication no` লিখেছ, কিন্তু তবু পাসওয়ার্ড দিয়ে ঢোকা যাচ্ছে। সম্ভাব্য কারণ কী, আর কীভাবে নিশ্চিত হবে?
5. fail2ban-এ `findtime`, `maxretry`, `bantime` কীভাবে একসাথে কাজ করে?
6. `ss -tulpn` আউটপুটে `0.0.0.0:6379` আর `127.0.0.1:3306` — কোনটা বেশি চিন্তার এবং কেন?
7. DAC ও MAC-এর মূল পার্থক্য কী? root হিসেবে চলা একটা হ্যাক হওয়া প্রসেসের ক্ষেত্রে MAC কেন গুরুত্বপূর্ণ?
8. Apache একটা ফাইলে 403 দিচ্ছে, পারমিশন ঠিক আছে, SELinux enforcing। তুমি কোন ক্রমে সমস্যা খুঁজবে ও ঠিক করবে?
9. `chcon` আর `semanage fcontext` + `restorecon`-এর পার্থক্য কী?
10. শুধু `sha256sum` মিলে গেলেই কেন একটা ডাউনলোড নিরাপদ বলা যায় না? GPG কোন সমস্যার সমাধান করে?

<details>
<summary>উত্তর দেখো</summary>

1. একাধিক স্তরে নিরাপত্তা সাজানো, যাতে একটা স্তর ভাঙলেও পরেরটা আটকায়। যেমন: নেটওয়ার্ক — firewall/fail2ban; সার্ভিস — অপ্রয়োজনীয় সার্ভিস বন্ধ, SSH hardening; অথেন্টিকেশন — PAM/pwquality/SSH key; অ্যাক্সেস কন্ট্রোল — permission + SELinux/AppArmor; ডেটা/কার্নেল — আপডেট, LUKS, sysctl; নজরদারি — auditd/AIDE/Lynis।
2. `/etc/apt/apt.conf.d/20auto-upgrades` — স্বয়ংক্রিয় চালু/বন্ধ ও কত দিন পরপর (`Update-Package-Lists`, `Unattended-Upgrade`); `/etc/apt/apt.conf.d/50unattended-upgrades` — কোন origin থেকে আপডেট নেবে, blacklist, স্বয়ংক্রিয় রিবুট, ইমেইল ইত্যাদি।
3. দুটোতেই মডিউল পাস করা বাধ্যতামূলক। `required` ফেল করলেও বাকি মডিউলগুলো চলে (আক্রমণকারী বুঝতে পারে না ঠিক কোথায় ফেল হলো), শেষে ফল ব্যর্থ। `requisite` ফেল করলে সঙ্গে সঙ্গে থেমে ব্যর্থতা ফেরত দেয়।
4. sshd-তে প্রথম পাওয়া মান কার্যকর হয়, আর drop-in ফাইলগুলো বর্ণানুক্রমে পড়া হয়। হয়তো আগের কোনো ফাইল (যেমন `50-cloud-init.conf`) বা মূল ফাইলের Include-এর আগের অংশে `PasswordAuthentication yes` আছে। `sudo sshd -T | grep passwordauthentication` দিয়ে কার্যকর মান দেখো, `grep -ri passwordauth /etc/ssh/` দিয়ে উৎস খোঁজো, নিজের ফাইলের নাম `00-hardening.conf` করো, তারপর `sshd -t` ও reload। আর `KbdInteractiveAuthentication no`-ও দিয়েছ কি না দেখো।
5. `findtime` সময়সীমার মধ্যে কোনো IP থেকে `maxretry` সংখ্যক ব্যর্থতা হলে সেই IP `bantime` সময়ের জন্য firewall-এ ব্লক হয়। যেমন ১০ মিনিটে ৩ বার → ১ ঘণ্টা ব্যান।
6. `0.0.0.0:6379` — Redis সব ইন্টারফেসে খোলা, firewall না থাকলে ইন্টারনেট থেকে ঢোকা যায়; এটা বেশি চিন্তার। `127.0.0.1:3306` শুধু লোকাল মেশিন থেকে অ্যাক্সেসযোগ্য।
7. DAC-এ ফাইলের মালিক নিজের ইচ্ছামতো অনুমতি দেয় এবং root প্রায় সব বাধা পেরোতে পারে। MAC-এ সিস্টেম-ব্যাপী নীতি ঠিক করে প্রোগ্রাম কী করতে পারবে, যা মালিক বা root সাধারণ উপায়ে এড়াতে পারে না। তাই root হিসেবে চলা হ্যাক হওয়া ওয়েব সার্ভারও তার domain-এর বাইরে (যেমন `/etc/shadow`) যেতে পারে না।
8. (ক) `ausearch -m AVC -ts recent` দিয়ে denial দেখা ও `audit2why` দিয়ে কারণ বোঝা; (খ) `ls -Z` দিয়ে label দেখা — ভুল হলে `restorecon -Rv` (অ-ডিফল্ট পাথে `semanage fcontext -a` তারপর `restorecon`); (গ) boolean দরকার হলে `setsebool -P`; (ঘ) অ-ডিফল্ট পোর্ট হলে `semanage port -a`; (ঙ) কিছুই না মিললে শেষ উপায় হিসেবে `audit2allow -M` দিয়ে মডিউল, আগে `.te` ফাইল পড়ে।
9. `chcon` সরাসরি ফাইলের label বদলায় — অস্থায়ী, পরের `restorecon` বা পূর্ণ relabel-এ মুছে যায়। `semanage fcontext` নীতিতে স্থায়ী নিয়ম যোগ করে, আর `restorecon` সেই নিয়ম অনুযায়ী label প্রয়োগ করে — এটাই সঠিক পদ্ধতি।
10. আক্রমণকারী ISO বদলাতে পারলে পাশের `SHA256SUMS` ফাইলটাও বদলে দিতে পারে; checksum শুধু ফাইল অক্ষত কি না বলে, উৎস আসল কি না বলে না। GPG স্বাক্ষর প্রমাণ করে যে `SHA256SUMS` প্রকাশকের private key দিয়েই সাইন করা (যদি তুমি public key-এর fingerprint অফিসিয়াল সূত্রের সাথে মিলিয়ে থাকো) — অর্থাৎ authenticity নিশ্চিত করে।

</details>

---

[← আগের অধ্যায়](44-environment-bashrc.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](46-performance-troubleshooting.md)

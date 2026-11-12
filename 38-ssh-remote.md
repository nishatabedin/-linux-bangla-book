# অধ্যায় ৩৮: SSH ও রিমোট অ্যাক্সেস

> একজন লিনাক্স অ্যাডমিনের দিনের বেশিরভাগ সময় কাটে এমন সব মেশিনে, যেগুলো সে কোনোদিন চোখেও দেখেনি। ডেটা সেন্টারে, ক্লাউডে, অন্য শহরে বা অন্য দেশে সার্ভারগুলো চলছে, আর সে নিজের ল্যাপটপে বসে সেগুলো চালাচ্ছে। এই কাজের দরজা হলো **SSH**।

## এই অধ্যায়ে যা শিখবে

- SSH কী, কেন Telnet/FTP-এর বদলে SSH ব্যবহার করা হয়
- SSH কানেকশনের ভেতরে ধাপে ধাপে কী ঘটে: TCP handshake, key exchange, host key যাচাই, user authentication
- `ssh user@host` দিয়ে লগইন, রিমোট কমান্ড চালানো, ভিন্ন port ব্যবহার
- `known_hosts` কী, "REMOTE HOST IDENTIFICATION HAS CHANGED" সতর্কবার্তার মানে কী
- `ssh-keygen` দিয়ে ed25519 key তৈরি, passphrase, fingerprint
- `ssh-copy-id` আর `authorized_keys` দিয়ে পাসওয়ার্ড ছাড়া লগইন
- `~/.ssh/config` দিয়ে লম্বা কমান্ড ছোট করা
- `ssh-agent` ও `ssh-add` দিয়ে passphrase বারবার না লেখা
- সার্ভার সাইডে `sshd` ইনস্টল, `sshd_config` হার্ডেনিং (`PasswordAuthentication no`, `PermitRootLogin` ইত্যাদি)
- ফাইল আদান-প্রদান: `scp`, `sftp`, `rsync` over SSH
- Port forwarding: local (`-L`), remote (`-R`), dynamic/SOCKS (`-D`)
- Jump host / bastion (`-J`, `ProxyJump`)
- SSH escape sequences (`~.`, `~?` ইত্যাদি)
- পারমিশন সমস্যা আর সাধারণ ট্রাবলশুটিং (`ssh -v`)

---

## ৩৮.১ SSH কী এবং কেন?

**SSH (Secure Shell)** হলো একটি নেটওয়ার্ক প্রোটোকল, যা দিয়ে তুমি অন্য একটি কম্পিউটারে নিরাপদে লগইন করতে পারো, কমান্ড চালাতে পারো, ফাইল পাঠাতে পারো — আর পুরো আলাপটা থাকে **encrypted** (এনক্রিপ্টেড, মানে তালাবদ্ধ সংকেতে)।

### আগে কী ছিল? Telnet-এর সমস্যা

SSH-এর আগে মানুষ `telnet`, `rlogin`, `rsh`, `ftp` ব্যবহার করত। এদের সবচেয়ে বড় সমস্যা: সবকিছু যেত **plain text**-এ। তুমি যে পাসওয়ার্ড টাইপ করছ, নেটওয়ার্কের মাঝখানে বসে থাকা যে কেউ (একই Wi-Fi-তে থাকা কেউ, ISP, বা কোনো হ্যাকড রাউটার) সেটা হুবহু পড়ে ফেলতে পারত।

**উপমা:** Telnet হলো পোস্টকার্ডে চিঠি পাঠানো — ডাকপিয়ন থেকে শুরু করে যে-ই হাতে পাক, পড়ে ফেলতে পারে। SSH হলো একটা তালাবদ্ধ বাক্সে চিঠি পাঠানো, যার চাবি শুধু তোমার আর প্রাপকের কাছে আছে — আর বাক্সের গায়ে সিল আছে, কেউ খুললে বা বদলালে ধরা পড়ে যাবে।

SSH তিনটা জিনিস নিশ্চিত করে:

| বৈশিষ্ট্য | মানে | কীভাবে |
|---|---|---|
| **Confidentiality** (গোপনীয়তা) | মাঝখানে কেউ পড়তে পারবে না | Symmetric encryption (যেমন AES, ChaCha20) |
| **Integrity** (অখণ্ডতা) | মাঝখানে কেউ ডেটা বদলাতে পারবে না | MAC / AEAD (যেমন `chacha20-poly1305`, `aes-gcm`) |
| **Authentication** (পরিচয় যাচাই) | তুমি সত্যিই ঠিক সার্ভারে, আর সার্ভার নিশ্চিত যে তুমিই তুমি | Host key + user key/password |

লিনাক্সে যে SSH সফটওয়্যারটা প্রায় সবাই ব্যবহার করে তার নাম **OpenSSH**। এর দুটো অংশ:

- **ক্লায়েন্ট** (`ssh`, `scp`, `sftp`, `ssh-keygen`, `ssh-agent` ...) — প্যাকেজ `openssh-client` (Debian/Ubuntu) বা `openssh-clients` (Fedora/RHEL)
- **সার্ভার** (`sshd` — "SSH daemon", যে প্রোগ্রামটা ব্যাকগ্রাউন্ডে বসে কানেকশনের অপেক্ষা করে) — প্যাকেজ `openssh-server`

ডিফল্টভাবে SSH চলে **TCP port 22**-এ।

```bash
$ ssh -V
OpenSSH_9.6p1 Ubuntu-3ubuntu13.5, OpenSSL 3.0.13 30 Jan 2024
```

> 📝 **মনে রাখো:** "SSH" দিয়ে প্রোটোকল বোঝায়, `ssh` হলো ক্লায়েন্ট কমান্ড, আর `sshd` হলো সার্ভার প্রোগ্রাম। নেটওয়ার্কের মূল ধারণা (IP, port, TCP) জানতে [অধ্যায় ৩৬](36-networking-basics.md) দেখো।

---

## ৩৮.২ ভেতরে কীভাবে কাজ করে — SSH আর্কিটেকচার

SSH প্রোটোকল তিনটি স্তরে (layer) সাজানো:

```
 ┌─────────────────────────────────────────────────────────┐
 │  Connection Layer   (RFC 4254)                          │
 │  একটা কানেকশনের ভেতরে অনেকগুলো "channel":                  │
 │  shell session, exec command, sftp, port forwarding...  │
 ├─────────────────────────────────────────────────────────┤
 │  User Authentication Layer   (RFC 4252)                 │
 │  "তুমি কে?" — publickey / password / keyboard-interactive  │
 ├─────────────────────────────────────────────────────────┤
 │  Transport Layer   (RFC 4253)                           │
 │  key exchange, server (host) authentication,            │
 │  encryption, integrity, compression                     │
 ├─────────────────────────────────────────────────────────┤
 │  TCP  (সাধারণত port 22)                                  │
 └─────────────────────────────────────────────────────────┘
```

এখন দেখি `ssh rahim@server.example.com` লিখে Enter চাপলে ঠিক কী কী হয়।

### ধাপ ১: TCP কানেকশন ও ভার্সন বিনিময়

ক্লায়েন্ট সার্ভারের port 22-এ TCP কানেকশন খোলে। তারপর দুই পক্ষ নিজেদের ভার্সন স্ট্রিং পাঠায়:

```
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.5
```

তুমি নিজেও এটা দেখতে পারো:

```bash
$ nc server.example.com 22
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.5
```

### ধাপ ২: Algorithm negotiation

দুই পক্ষ নিজেদের সমর্থিত algorithm-এর তালিকা পাঠায় (key exchange, host key, cipher, MAC, compression)। দুই তালিকায় মিল আছে এমন সবচেয়ে পছন্দের algorithm বেছে নেওয়া হয়।

### ধাপ ৩: Key Exchange (KEX) — অচেনা দুজনের গোপন চাবি বানানো

এটাই SSH-এর জাদু। ক্লায়েন্ট আর সার্ভার আগে কখনো কোনো গোপন কিছু ভাগাভাগি করেনি, অথচ খোলা নেটওয়ার্কে কথা বলেই তারা এমন একটা **shared secret** তৈরি করে ফেলে যেটা মাঝখানের কেউ জানতে পারে না। এর পদ্ধতির নাম **Diffie-Hellman key exchange** (আধুনিক রূপ: elliptic curve `curve25519-sha256`; নতুন OpenSSH-এ quantum-resistant hybrid `sntrup761x25519-sha512` বা `mlkem768x25519-sha256`)।

**রঙের উপমা:**

```
   ক্লায়েন্ট                                    সার্ভার
   ─────────                                   ─────────
   সবার জানা রং: হলুদ  ◄────── public ──────►  সবার জানা রং: হলুদ

   গোপন রং: লাল (কাউকে বলে না)               গোপন রং: নীল (কাউকে বলে না)

   হলুদ + লাল = কমলা                          হলুদ + নীল = সবুজ
          │                                          │
          └──────── কমলা পাঠায় ─────────►            │
                    ◄────────── সবুজ পাঠায় ──────────┘

   সবুজ + লাল = বাদামি                        কমলা + নীল = বাদামি
        ▲                                           ▲
        └──────── দুজনের কাছেই একই "বাদামি" ────────┘
                (মাঝখানের লোক শুধু হলুদ, কমলা, সবুজ দেখেছে;
                 মিশ্রণ আলাদা করা প্রায় অসম্ভব, তাই বাদামি বানাতে পারে না)
```

এই "বাদামি" shared secret থেকে **session key** তৈরি হয়, আর বাকি পুরো কানেকশন সেই key দিয়ে symmetric encryption-এ চলে। প্রতিটা কানেকশনে নতুন session key — তাই আজকের key চুরি হলেও গতকালের রেকর্ড করা ট্রাফিক খোলা যায় না (একে **forward secrecy** বলে)।

### ধাপ ৪: Server authentication — "এটা কি সত্যিই আমার সার্ভার?"

Key exchange-এর সময় সার্ভার তার **host key** (সার্ভারের নিজস্ব স্থায়ী key pair, থাকে `/etc/ssh/ssh_host_*_key`-এ) দিয়ে exchange-এর ডেটায় একটা **digital signature** দেয়। ক্লায়েন্ট সার্ভারের public host key দিয়ে সেটা যাচাই করে, আর মিলিয়ে দেখে `~/.ssh/known_hosts`-এ এই সার্ভারের যে key আগে সেভ করা আছে তার সাথে মেলে কিনা।

এটা না থাকলে **Man-in-the-Middle (MITM)** আক্রমণ সম্ভব হতো: কেউ মাঝখানে বসে নিজেকে সার্ভার হিসেবে উপস্থাপন করত, তুমি তার সাথে key exchange করতে, আর সে সব পড়ত।

### ধাপ ৫: User authentication — "তুমি কে?"

এবার encrypted চ্যানেলের ভেতরে ক্লায়েন্ট নিজের পরিচয় প্রমাণ করে। সাধারণ পদ্ধতি:

| পদ্ধতি | কীভাবে | নিরাপত্তা |
|---|---|---|
| `publickey` | তোমার private key দিয়ে signature | সবচেয়ে ভালো (প্রস্তাবিত) |
| `password` | পাসওয়ার্ড পাঠানো (encrypted চ্যানেলে) | brute-force ঝুঁকি |
| `keyboard-interactive` | সার্ভার প্রশ্ন করে (পাসওয়ার্ড, OTP ইত্যাদি) | 2FA-এর জন্য কাজে লাগে |
| `gssapi-with-mic` | Kerberos | এন্টারপ্রাইজ নেটওয়ার্কে |

### ধাপ ৬: Session — channel খোলা

সফল হলে ক্লায়েন্ট একটা **channel** খোলে, সার্ভারের `sshd` তোমার ইউজার হিসেবে একটা প্রসেস চালু করে (সাধারণত তোমার লগইন শেল, আর একটা pseudo-terminal বা **PTY**)। এখন তুমি যা টাইপ করো সেটা encrypted হয়ে যায়, শেল চালায়, output encrypted হয়ে ফেরত আসে।

```
তোমার ল্যাপটপ                                          সার্ভার
┌──────────────┐                                  ┌───────────────────┐
│ টার্মিনাল    │                                  │ sshd (root, :22)  │
│   │          │      encrypted TCP stream        │   └─ sshd (তোমার)  │
│   ▼          │ ◄══════════════════════════════► │       └─ bash      │
│ ssh client   │   channel 0: shell (PTY)         │           └─ top   │
│              │   channel 1: port forward ...    │                   │
└──────────────┘                                  └───────────────────┘
```

> 💡 **টিপস:** একটা SSH কানেকশনের ভেতর দিয়ে একসাথে অনেকগুলো channel যেতে পারে — এজন্যই একই কানেকশনে শেলও চলছে, আবার port forwarding-ও হচ্ছে।

### Public key authentication বিস্তারিত

এটা বুঝলে SSH key-এর পুরো ব্যাপারটা পরিষ্কার হয়ে যাবে।

**Key pair** মানে দুটো গাণিতিকভাবে জোড়া লাগানো key:
- **Private key** (`~/.ssh/id_ed25519`) — শুধু তোমার কাছে থাকে, কাউকে দেবে না। এটা হলো **সিলমোহর**।
- **Public key** (`~/.ssh/id_ed25519.pub`) — যাকে খুশি দিতে পারো। এটা হলো সেই সিলমোহরের **ছাপের নমুনা** — যে কেউ মিলিয়ে দেখতে পারে কোনো ছাপ আসল সিলমোহর থেকে এসেছে কিনা, কিন্তু নমুনা থেকে সিলমোহর বানানো যায় না।

```
  ক্লায়েন্ট (তুমি)                                     সার্ভার
  ┌─────────────────────┐                     ┌──────────────────────────────┐
  │ ~/.ssh/id_ed25519   │                     │ ~rahim/.ssh/authorized_keys  │
  │   (PRIVATE key)     │                     │   ssh-ed25519 AAAAC3... ←──┐ │
  │ ~/.ssh/id_ed25519.pub│ ── আগেই কপি করা ──► │   (তোমার PUBLIC key)       │ │
  └─────────────────────┘                     └────────────────────────────┼─┘
                                                                           │
  ১. "আমি rahim, এই public key দিয়ে লগইন করতে চাই"  ──────────────►        │
                                                   সার্ভার দেখে key-টা ───┘
                                                   authorized_keys-এ আছে কিনা
  ২.            ◄────────── "ঠিক আছে, তাহলে প্রমাণ দাও" (key OK)

  ৩. ক্লায়েন্ট session-এর অনন্য ডেটা (session ID সহ) private key
     দিয়ে SIGN করে পাঠায়  ──────────────────────────────────►
                                                   সার্ভার public key দিয়ে
                                                   signature VERIFY করে
  ৪.            ◄────────── সফল! শেল খুলে দিচ্ছি
```

গুরুত্বপূর্ণ ব্যাপার: **private key কখনো নেটওয়ার্কে যায় না।** শুধু signature যায়, আর সেটা এই একটা session-এর ডেটার উপর, তাই চুরি করে পরে আবার ব্যবহার (replay) করা যায় না।

> 📝 **মনে রাখো:** দুই ধরনের key আছে, গুলিয়ে ফেলো না।
> - **Host key** — সার্ভারের পরিচয়। ক্লায়েন্ট সেভ করে রাখে `~/.ssh/known_hosts`-এ।
> - **User key** — তোমার পরিচয়। সার্ভার সেভ করে রাখে `~/.ssh/authorized_keys`-এ।

---

## ৩৮.৩ প্রথম লগইন: `ssh user@host`

### Syntax

```
ssh [options] [user@]host [command]
```

- `user` না দিলে তোমার লোকাল ইউজারনেম ধরে নেয়।
- `host` হতে পারে IP (`192.168.1.50`), hostname (`server1`), বা ডোমেইন (`server.example.com`)।
- `command` দিলে লগইন শেল না খুলে শুধু সেই কমান্ড চালিয়ে বেরিয়ে আসে।

### সার্ভারে SSH চালু করা (যদি না থাকে)

যে মেশিনে লগইন করবে, সেখানে `sshd` চলতে হবে।

```bash
# Debian / Ubuntu
$ sudo apt install openssh-server
$ sudo systemctl enable --now ssh

# Fedora / RHEL / Rocky / Alma
$ sudo dnf install openssh-server
$ sudo systemctl enable --now sshd
```

> ⚠️ **সতর্কতা:** সার্ভিসের নাম distro-ভেদে আলাদা: Debian/Ubuntu-তে `ssh`, Fedora/RHEL-এ `sshd`। Ubuntu 22.10 থেকে (24.04 সহ) SSH ডিফল্টভাবে **socket activation** দিয়ে চলে — `ssh.socket` port-এ শোনে, কানেকশন এলে `ssh.service` চালু হয়। systemd-এর বিস্তারিত [অধ্যায় ২৬](26-systemd-services.md)-এ।

চলছে কিনা দেখো:

```bash
$ sudo ss -tlnp | grep :22
LISTEN 0  4096   0.0.0.0:22   0.0.0.0:*   users:(("systemd",pid=1,fd=xx))
LISTEN 0  4096      [::]:22      [::]:*   users:(("systemd",pid=1,fd=xx))
```

(socket activation-এ শোনে `systemd` নিজে; ক্লাসিক সেটআপে দেখাবে `sshd`।)

### প্রথমবার কানেক্ট করা

```bash
$ ssh rahim@192.168.1.50
The authenticity of host '192.168.1.50 (192.168.1.50)' can't be established.
ED25519 key fingerprint is SHA256:Xq3b7mQ1u0dPq9yJmHkV2cGx8N6zVvE4tR1lKpW0aBc.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.50' (ED25519) to the list of known hosts.
rahim@192.168.1.50's password:
Welcome to Ubuntu 24.04.1 LTS (GNU/Linux 6.8.0-45-generic x86_64)
...
rahim@server1:~$
```

এই "authenticity can't be established" প্রশ্নটা হলো **TOFU — Trust On First Use**। প্রথমবার ক্লায়েন্ট সার্ভারকে চেনে না, তাই তোমাকে জিজ্ঞেস করছে। সত্যিকারের নিরাপদ পদ্ধতি হলো সার্ভারের কনসোলে বসে (বা অ্যাডমিনের কাছ থেকে) fingerprint মিলিয়ে নেওয়া:

```bash
# সার্ভারে চালাও
$ ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
256 SHA256:Xq3b7mQ1u0dPq9yJmHkV2cGx8N6zVvE4tR1lKpW0aBc root@server1 (ED25519)
```

মিললে `yes` লেখো। প্রশ্নের উত্তরে সরাসরি fingerprint পেস্ট করলেও চলে — মিললে ক্লায়েন্ট নিজেই গ্রহণ করে।

বের হতে: `exit` লেখো বা `Ctrl+D` চাপো।

```bash
rahim@server1:~$ exit
logout
Connection to 192.168.1.50 closed.
```

### গুরুত্বপূর্ণ `ssh` অপশন

| অপশন | কাজ | উদাহরণ |
|---|---|---|
| `-p PORT` | ভিন্ন port (ছোট হাতের p) | `ssh -p 2222 rahim@host` |
| `-l USER` | ইউজার আলাদাভাবে দেওয়া | `ssh -l rahim host` |
| `-i FILE` | নির্দিষ্ট private key ব্যবহার | `ssh -i ~/.ssh/work_key host` |
| `-v`, `-vv`, `-vvv` | verbose — ডিবাগিং তথ্য | `ssh -v host` |
| `-t` | জোর করে PTY দেওয়া (interactive রিমোট কমান্ডের জন্য) | `ssh -t host top` |
| `-T` | PTY না দেওয়া | `ssh -T git@github.com` |
| `-C` | compression চালু (ধীর লিংকে কাজে লাগে) | `ssh -C host` |
| `-X` / `-Y` | X11 forwarding (রিমোট GUI অ্যাপ লোকালে দেখানো) | `ssh -X host xclock` |
| `-A` | agent forwarding (সাবধানে!) | `ssh -A host` |
| `-L`, `-R`, `-D` | port forwarding (নিচে বিস্তারিত) | |
| `-J` | jump host দিয়ে যাওয়া | `ssh -J bastion internal` |
| `-N` | কোনো রিমোট কমান্ড চালাবে না (শুধু forwarding) | `ssh -N -L ...` |
| `-f` | authentication-এর পর background-এ চলে যাবে | `ssh -fN -L ...` |
| `-o KEY=VALUE` | যেকোনো config অপশন কমান্ড লাইনে | `ssh -o ConnectTimeout=5 host` |
| `-F FILE` | ভিন্ন config ফাইল | `ssh -F ./cfg host` |
| `-q` | quiet — সতর্কবার্তা কম দেখাবে | `ssh -q host` |
| `-4` / `-6` | শুধু IPv4 / IPv6 | `ssh -4 host` |
| `-G` | কোনো host-এর জন্য শেষ পর্যন্ত কী config প্রযোজ্য, সেটা দেখায় (কানেক্ট করে না) | `ssh -G web1` |

### রিমোট কমান্ড চালানো (লগইন না করেই)

```bash
$ ssh rahim@server1 'uptime'
 14:22:07 up 12 days,  3:41,  1 user,  load average: 0.08, 0.05, 0.01

$ ssh rahim@server1 'df -h / && free -h'

# লোকাল ফাইল রিমোটে পাইপ করা
$ cat notes.txt | ssh rahim@server1 'cat > /tmp/notes.txt'

# রিমোট output লোকালে প্রসেস করা
$ ssh rahim@server1 'cat /var/log/syslog' | grep -i error | tail -5
```

> ⚠️ **সতর্কতা:** কোটেশন খেয়াল করো। `ssh host "echo $HOME"` লিখলে `$HOME` তোমার **লোকাল** শেল আগেই expand করে ফেলবে (তোমার লোকাল home দেখাবে)। রিমোটের `$HOME` চাইলে single quote দাও: `ssh host 'echo $HOME'`। কোটেশনের নিয়ম জানতে [অধ্যায় ৪১](41-shell-scripting-1.md) দেখো।

`top`, `htop`, `vim` এর মতো interactive প্রোগ্রাম রিমোট কমান্ড হিসেবে চালাতে `-t` দরকার:

```bash
$ ssh -t rahim@server1 htop
$ ssh -t rahim@server1 'sudo systemctl restart nginx'   # sudo পাসওয়ার্ড চাইবে, তাই -t
```

---

## ৩৮.৪ `known_hosts` — সার্ভারের পরিচয়পত্রের খাতা

প্রথমবার `yes` বলার পর সার্ভারের public host key সেভ হয় `~/.ssh/known_hosts`-এ:

```bash
$ cat ~/.ssh/known_hosts
|1|kV3h...=|q8Zp...= ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGq...
```

শুরুর `|1|...` অংশটা হলো **hashed hostname** — Debian/Ubuntu-তে `HashKnownHosts yes` ডিফল্ট, যাতে কেউ ফাইলটা চুরি করলেও জানতে না পারে তুমি কোন কোন সার্ভারে যাও। Fedora/RHEL-এ সাধারণত hostname সরাসরি দেখা যায়।

পরের প্রতিবার কানেক্ট করার সময় ক্লায়েন্ট মিলিয়ে দেখে। না মিললে এই ভয়ংকর বার্তা:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
...
Offending ED25519 key in /home/rahim/.ssh/known_hosts:3
Host key verification failed.
```

**এর মানে কী?** দুটোর একটা:
1. **নিরীহ কারণ:** সার্ভার রিইনস্টল হয়েছে, নতুন VM একই IP পেয়েছে, অ্যাডমিন host key বদলেছে।
2. **বিপজ্জনক কারণ:** কেউ MITM আক্রমণ করছে।

তাই অন্ধভাবে মুছে ফেলবে না — আগে নিশ্চিত হও কারণটা নিরীহ। নিশ্চিত হলে পুরোনো entry সরাও:

```bash
$ ssh-keygen -R 192.168.1.50
# Host 192.168.1.50 found: line 3
/home/rahim/.ssh/known_hosts updated.
Original contents retained as /home/rahim/.ssh/known_hosts.old
```

অন্য কাজের কমান্ড:

```bash
# known_hosts-এ কোনো host আছে কিনা খোঁজা (hashed হলেও কাজ করে)
$ ssh-keygen -F 192.168.1.50

# কানেক্ট না করেই সার্ভারের host key আনা
$ ssh-keyscan -t ed25519 192.168.1.50
192.168.1.50 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGq...
```

`StrictHostKeyChecking` অপশনটা নিয়ন্ত্রণ করে নতুন/বদলানো key-এর ক্ষেত্রে কী হবে:

| মান | নতুন host | বদলানো key |
|---|---|---|
| `ask` (ডিফল্ট) | জিজ্ঞেস করে | কানেক্ট করে না |
| `accept-new` | চুপচাপ যোগ করে | কানেক্ট করে না |
| `yes` | কানেক্ট করে না | কানেক্ট করে না |
| `no` / `off` | চুপচাপ যোগ করে | সতর্ক করে, কিছু ঝুঁকিপূর্ণ ফিচার বন্ধ করে কানেক্ট করে |

> 💡 **টিপস:** অটোমেশন স্ক্রিপ্টে `-o StrictHostKeyChecking=accept-new` ভালো আপস — নতুন সার্ভার যোগ করে, কিন্তু key বদলালে থামে। `StrictHostKeyChecking=no` আর `UserKnownHostsFile=/dev/null` একসাথে দেওয়া মানে MITM-এর দরজা পুরো খুলে দেওয়া; শুধু ল্যাবের ক্ষণস্থায়ী VM-এর জন্য।

---

## ৩৮.৫ SSH Key তৈরি: `ssh-keygen`

পাসওয়ার্ড দিয়ে লগইনের সমস্যা: দুর্বল পাসওয়ার্ড অনুমান করা যায়, ইন্টারনেটে খোলা যেকোনো SSH সার্ভারে দিনে হাজার হাজার bot পাসওয়ার্ড চেষ্টা করে। Key-based লগইন অনেক বেশি নিরাপদ এবং বেশি সুবিধাজনক।

### কোন ধরনের key?

| Type | মন্তব্য |
|---|---|
| **ed25519** | আধুনিক, দ্রুত, ছোট, নিরাপদ — **এটাই ব্যবহার করো** |
| rsa | পুরোনো সিস্টেমের সাথে compatible; ব্যবহার করলে কমপক্ষে 3072 বা 4096 bit |
| ecdsa | চলে, কিন্তু ed25519 বেশি পছন্দনীয় |
| ed25519-sk / ecdsa-sk | হার্ডওয়্যার security key (YubiKey ইত্যাদি)-তে বাঁধা |
| dsa | বাতিল; নতুন OpenSSH-এ সমর্থনই নেই |

### Key তৈরি

```bash
$ ssh-keygen -t ed25519 -C "rahim@laptop"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/rahim/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/rahim/.ssh/id_ed25519
Your public key has been saved in /home/rahim/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:9fT2nC8yW1aRkL0pM3xQe7HvB5uJdZ4sN6gO2iYtK1w rahim@laptop
The key's randomart image is:
+--[ED25519 256]--+
|        .o+=*o.  |
|       . ..=+=.  |
|        o *.B..  |
|       . O * =   |
|        S = = .  |
|         + + o   |
|        . = E .  |
|         o.B o   |
|         .=*B.   |
+----[SHA256]-----+
```

তৈরি হলো দুটো ফাইল:

```bash
$ ls -l ~/.ssh/
-rw------- 1 rahim rahim 411 Nov 12 10:01 id_ed25519       # private — কাউকে দেবে না!
-rw-r--r-- 1 rahim rahim  98 Nov 12 10:01 id_ed25519.pub   # public — নির্দ্বিধায় শেয়ার করো

$ cat ~/.ssh/id_ed25519.pub
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJm8x... rahim@laptop
```

Public key-এর তিনটা অংশ: `type` `base64-key` `comment`। Comment শুধু চেনার সুবিধার জন্য, কাজে কোনো প্রভাব নেই।

### Passphrase — দিতে হবে কেন?

Passphrase দিলে private key ফাইলটা নিজেই encrypted থাকে। ল্যাপটপ চুরি হলে বা ব্যাকআপ ফাঁস হলে চোর key ফাইল পেলেও passphrase ছাড়া ব্যবহার করতে পারবে না। বারবার লেখার ঝামেলা? সেটার সমাধান `ssh-agent` (৩৮.৮ অংশে দেখো)।

> 💡 **টিপস:** পাসওয়ার্ডবিহীন key শুধু অটোমেশনের জন্য (cron job, CI) — আর সেই key-কে `authorized_keys`-এ সীমাবদ্ধ করে দাও (৩৮.৬-এ দেখো)।

### `ssh-keygen`-এর গুরুত্বপূর্ণ অপশন

| অপশন | কাজ |
|---|---|
| `-t TYPE` | key type (`ed25519`, `rsa`, `ecdsa`, `ed25519-sk`) |
| `-b BITS` | key size (RSA-র জন্য, যেমন `-b 4096`) |
| `-C COMMENT` | comment |
| `-f FILE` | ফাইলের নাম/পাথ |
| `-N PASS` | passphrase কমান্ডেই (`-N ""` = খালি; স্ক্রিপ্টে) |
| `-a ROUNDS` | passphrase brute-force কঠিন করতে KDF rounds (ডিফল্ট 16) |
| `-p` | বিদ্যমান key-এর passphrase বদলানো |
| `-l` | fingerprint দেখানো |
| `-y` | private key থেকে public key আবার তৈরি করা |
| `-R HOST` | known_hosts থেকে host মুছে ফেলা |
| `-F HOST` | known_hosts-এ host খোঁজা |
| `-E md5` | পুরোনো MD5 ফরম্যাটে fingerprint |

```bash
# কাজের জন্য আলাদা key, আলাদা নামে
$ ssh-keygen -t ed25519 -C "rahim@work" -f ~/.ssh/id_work

# passphrase বদলানো
$ ssh-keygen -p -f ~/.ssh/id_ed25519

# fingerprint দেখা
$ ssh-keygen -lf ~/.ssh/id_ed25519.pub
256 SHA256:9fT2nC8yW1aRkL0pM3xQe7HvB5uJdZ4sN6gO2iYtK1w rahim@laptop (ED25519)

# .pub ফাইল হারিয়ে গেলে
$ ssh-keygen -y -f ~/.ssh/id_ed25519 > ~/.ssh/id_ed25519.pub
```

---

## ৩৮.৬ সার্ভারে public key বসানো: `ssh-copy-id` ও `authorized_keys`

### সহজ পথ: `ssh-copy-id`

```bash
$ ssh-copy-id -i ~/.ssh/id_ed25519.pub rahim@192.168.1.50
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/rahim/.ssh/id_ed25519.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
rahim@192.168.1.50's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'rahim@192.168.1.50'"
and check to make sure that only the key(s) you wanted were added.
```

এটা একবার পাসওয়ার্ড দিয়ে লগইন করে, সার্ভারে `~/.ssh` না থাকলে তৈরি করে, তোমার public key `~/.ssh/authorized_keys`-এর শেষে যোগ করে, আর পারমিশন ঠিক করে দেয়। ভিন্ন port হলে: `ssh-copy-id -p 2222 rahim@host`।

এখন:

```bash
$ ssh rahim@192.168.1.50
Welcome to Ubuntu 24.04.1 LTS ...
rahim@server1:~$          # পাসওয়ার্ড চায়নি!
```

### হাতে করে (ssh-copy-id না থাকলে)

```bash
$ cat ~/.ssh/id_ed25519.pub | ssh rahim@server1 \
  'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

> ⚠️ **সতর্কতা:** `>>` (append) দিও, `>` না — নইলে আগের সব key মুছে যাবে এবং অন্যরা (বা তুমি নিজেই অন্য মেশিন থেকে) লক-আউট হবে। রিডাইরেকশন নিয়ে [অধ্যায় ১৯](19-redirection-pipe.md) দেখো।

### `authorized_keys` ফাইলের ফরম্যাট

প্রতি লাইনে একটা key:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJm8x... rahim@laptop
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKp2q... rahim@desktop
```

কোনো key সরিয়ে দিতে চাইলে (যেমন পুরোনো ল্যাপটপ হারিয়ে গেছে) শুধু সেই লাইনটা মুছে দাও।

### Key-এর সামনে সীমাবদ্ধতা (options)

Key-এর আগে option বসিয়ে ক্ষমতা কমানো যায় — অটোমেশন key-এর জন্য দারুণ:

```
from="10.0.0.0/24",restrict,command="/usr/local/bin/backup.sh" ssh-ed25519 AAAA... backup-bot
```

| Option | কাজ |
|---|---|
| `from="pattern"` | শুধু এই IP/host থেকে key গ্রহণযোগ্য |
| `command="cmd"` | যা-ই চাওয়া হোক, শুধু এই কমান্ড চলবে |
| `restrict` | port/agent/X11 forwarding ও PTY সব বন্ধ |
| `no-port-forwarding`, `no-agent-forwarding`, `no-X11-forwarding`, `no-pty` | আলাদা আলাদা ভাবে বন্ধ |
| `expiry-time="YYYYMMDD"` | নির্দিষ্ট তারিখের পর key অচল |
| `permitopen="host:port"` | শুধু নির্দিষ্ট গন্তব্যে local forwarding |

---

## ৩৮.৭ `~/.ssh/config` — শর্টকাটের রাজা

ধরো তুমি রোজ লেখো:

```bash
$ ssh -p 2222 -i ~/.ssh/id_work deploy@203.0.113.25
```

এর বদলে `~/.ssh/config` ফাইলে:

```
# ~/.ssh/config

Host web1
    HostName 203.0.113.25
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_work
    IdentitiesOnly yes

Host db1
    HostName 10.0.5.12
    User admin
    ProxyJump web1

Host github.com
    User git
    IdentityFile ~/.ssh/id_github

Host *.lab.local
    User student
    StrictHostKeyChecking accept-new

# সব host-এর জন্য ডিফল্ট — সবসময় শেষে রাখো
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    AddKeysToAgent yes
```

এখন শুধু:

```bash
$ ssh web1
$ scp report.pdf web1:/tmp/
$ rsync -av site/ web1:/var/www/
```

`scp`, `sftp`, `rsync`, `git` — সবাই এই config পড়ে।

### কীভাবে পড়া হয় (খুব গুরুত্বপূর্ণ)

```
  ssh web1
     │
     ▼
  ১. কমান্ড লাইনের -o / -p / -i ইত্যাদি
  ২. ~/.ssh/config (উপর থেকে নিচে)
  ৩. /etc/ssh/ssh_config (+ /etc/ssh/ssh_config.d/*.conf)
     │
     ▼
  প্রতিটা অপশনের জন্য যে মান **প্রথমে** পাওয়া যায় সেটাই চূড়ান্ত
```

এজন্যই নির্দিষ্ট `Host` ব্লক উপরে আর `Host *` সবার শেষে রাখতে হয়। `Host *` উপরে রাখলে সেখানে দেওয়া `User` সব host-এর জন্য "প্রথম মান" হয়ে যাবে, নিচের ব্লকের `User` উপেক্ষিত হবে।

```bash
# কোনো host-এর চূড়ান্ত config দেখা
$ ssh -G web1 | grep -E '^(hostname|user|port|identityfile) '
hostname 203.0.113.25
user deploy
port 2222
identityfile ~/.ssh/id_work
```

### দরকারি config অপশন

| অপশন | কাজ |
|---|---|
| `Host` | alias/pattern (`*`, `?`, `!` চলে) |
| `HostName` | আসল host/IP |
| `User`, `Port` | ইউজার, port |
| `IdentityFile` | কোন private key |
| `IdentitiesOnly yes` | শুধু উল্লেখিত key চেষ্টা করবে (agent-এ অনেক key থাকলে "Too many authentication failures" এড়ায়) |
| `ProxyJump` | jump host |
| `LocalForward`, `RemoteForward`, `DynamicForward` | স্থায়ী port forwarding |
| `ServerAliveInterval` | এত সেকেন্ড পরপর keepalive পাঠায় (idle কানেকশন কেটে যাওয়া ঠেকায়) |
| `ForwardAgent` | agent forwarding |
| `AddKeysToAgent yes` | প্রথমবার ব্যবহারে key agent-এ যোগ |
| `ControlMaster`, `ControlPath`, `ControlPersist` | একই host-এ কানেকশন শেয়ার (multiplexing) |
| `Include` | অন্য config ফাইল যোগ করা |

### Multiplexing — দ্বিতীয় কানেকশন চোখের পলকে

```
Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm-%r@%h:%p
    ControlPersist 10m
```

প্রথম `ssh web1` সাধারণভাবে কানেক্ট করে একটা socket রেখে দেয়। পরের `ssh web1`, `scp` ইত্যাদি সেই খোলা কানেকশনের ভেতরে নতুন channel খোলে — নতুন handshake বা authentication লাগে না। শেষ কানেকশন বন্ধ হওয়ার পর ১০ মিনিট পর্যন্ত master টিকে থাকে।

```bash
$ ssh -O check web1     # master চলছে কিনা
Master running (pid=41234)
$ ssh -O exit web1      # master বন্ধ করা
```

> 📝 **মনে রাখো:** `~/.ssh/config` ফাইলটা অন্যরা লিখতে পারলে ssh সেটা ব্যবহার করতে অস্বীকার করে ("Bad owner or permissions")। `chmod 600 ~/.ssh/config` দাও।

---

## ৩৮.৮ `ssh-agent` — passphrase একবার লেখো

Key-এ passphrase দিয়েছ (ভালো!), কিন্তু প্রতিবার `ssh` চালালে passphrase চাইছে (বিরক্তিকর!)। সমাধান **`ssh-agent`** — একটা ছোট ব্যাকগ্রাউন্ড প্রোগ্রাম যা তোমার decrypted private key মেমরিতে ধরে রাখে।

**উপমা:** অফিসের রিসেপশনিস্ট। সকালে একবার তাকে তোমার সিলমোহর দিয়ে দাও (passphrase দিয়ে খুলে)। সারাদিন কোনো কাগজে সিল লাগাতে হলে তুমি কাগজটা তার কাছে পাঠাও, সে সিল মেরে ফেরত দেয় — সিলমোহর কখনো তার টেবিল ছেড়ে যায় না।

```
  ssh client ──"এই ডেটায় sign করে দাও"──► ssh-agent (মেমরিতে key)
       ▲                                        │
       └──────────── signature ◄────────────────┘
          (যোগাযোগ হয় একটা UNIX socket দিয়ে: $SSH_AUTH_SOCK)
```

GNOME/KDE-এর মতো ডেস্কটপে সাধারণত লগইনের সাথে সাথেই একটা agent চালু থাকে (GNOME-এ `gcr-ssh-agent` বা পুরোনোতে `gnome-keyring`)। চেক করো:

```bash
$ echo $SSH_AUTH_SOCK
/run/user/1000/gcr/ssh
```

খালি হলে (যেমন সার্ভারে বা শুধু-টার্মিনাল সেশনে) নিজে চালু করো:

```bash
$ eval "$(ssh-agent -s)"
Agent pid 52811
```

`ssh-agent -s` কিছু শেল কমান্ড প্রিন্ট করে (`SSH_AUTH_SOCK=...; export SSH_AUTH_SOCK; ...`), আর `eval` সেগুলো বর্তমান শেলে চালিয়ে environment variable সেট করে দেয়।

### `ssh-add`

| কমান্ড | কাজ |
|---|---|
| `ssh-add` | ডিফল্ট key-গুলো (`~/.ssh/id_ed25519` ইত্যাদি) যোগ করে |
| `ssh-add ~/.ssh/id_work` | নির্দিষ্ট key যোগ |
| `ssh-add -t 3600` | ১ ঘণ্টা পর key নিজে থেকে মুছে যাবে |
| `ssh-add -l` | লোড করা key-এর fingerprint তালিকা |
| `ssh-add -L` | লোড করা key-এর public key তালিকা |
| `ssh-add -d FILE` | একটা key সরানো |
| `ssh-add -D` | সব key সরানো |
| `ssh-add -x` / `-X` | agent লক / আনলক (পাসওয়ার্ড দিয়ে) |

```bash
$ ssh-add ~/.ssh/id_ed25519
Enter passphrase for /home/rahim/.ssh/id_ed25519:
Identity added: /home/rahim/.ssh/id_ed25519 (rahim@laptop)

$ ssh-add -l
256 SHA256:9fT2nC8yW1aRkL0pM3xQe7HvB5uJdZ4sN6gO2iYtK1w rahim@laptop (ED25519)
```

এখন যত খুশি `ssh` চালাও, passphrase আর চাইবে না।

### Agent forwarding (`-A`) — সুবিধা আর ঝুঁকি

`ssh -A server1` দিলে server1-এ বসে আরেকটা সার্ভারে (বা GitHub-এ) ssh করার সময় তোমার **লোকাল** agent ব্যবহার হয় — key সার্ভারে কপি করতে হয় না।

> ⚠️ **সতর্কতা:** server1-এর root (বা server1 হ্যাক করা কেউ) তোমার কানেকশন চলাকালীন তোমার forwarded agent socket ব্যবহার করে তোমার নামে অন্য সার্ভারে লগইন করতে পারে। অবিশ্বস্ত সার্ভারে কখনো `-A` দিও না। বেশিরভাগ ক্ষেত্রে jump host-এর জন্য `-J` (৩৮.১৫) agent forwarding-এর চেয়ে নিরাপদ বিকল্প।

---

## ৩৮.৯ সার্ভার সাইড: `sshd_config` ও হার্ডেনিং

সার্ভারের `sshd` কনফিগার হয় `/etc/ssh/sshd_config` দিয়ে (ক্লায়েন্টের `/etc/ssh/ssh_config`-এর সাথে গুলিয়ো না — নামে শুধু একটা `d`-এর পার্থক্য)।

### ফাইলের গঠন — `Include` আর "প্রথম মান জেতে"

আধুনিক Ubuntu, Debian 12, Fedora, RHEL 9 — সবার `sshd_config`-এর একদম উপরের দিকে আছে:

```
Include /etc/ssh/sshd_config.d/*.conf
```

আর `sshd`-এর নিয়ম হলো: **বেশিরভাগ keyword-এর ক্ষেত্রে যে মান প্রথমে পাওয়া যায় সেটাই কার্যকর।** যেহেতু `Include` উপরে, তাই `sshd_config.d/`-এর ফাইলগুলো মূল ফাইলের চেয়ে অগ্রাধিকার পায়, আর সেখানে ফাইলগুলো বর্ণানুক্রমে পড়া হয়।

> ⚠️ **সতর্কতা (বাস্তবে অনেকেই এখানে আটকায়):** তুমি `sshd_config`-এ `PasswordAuthentication no` লিখলে, কিন্তু তবুও পাসওয়ার্ড দিয়ে লগইন হচ্ছে! কারণ প্রায়ই ক্লাউড ইমেজে `/etc/ssh/sshd_config.d/50-cloud-init.conf` ফাইলে `PasswordAuthentication yes` আছে — সেটা আগে পড়া হয় তাই জিতে যায়। সমাধান: নিজের সেটিং একটা কম নম্বরের ফাইলে রাখো, যেমন `/etc/ssh/sshd_config.d/00-hardening.conf`, অথবা সেই cloud-init ফাইলটা ঠিক করো। সবসময় `sshd -T` দিয়ে কার্যকর মান যাচাই করো।

### হার্ডেনিং কনফিগারেশন — একটা ভালো উদাহরণ

```
# /etc/ssh/sshd_config.d/00-hardening.conf

# root সরাসরি লগইন করতে পারবে না (sudo ব্যবহার করো)
PermitRootLogin no

# শুধু key দিয়ে লগইন
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no

# শুধু এই ইউজার/গ্রুপ লগইন করতে পারবে
AllowGroups sshusers

# brute-force কঠিন করা
MaxAuthTries 3
LoginGraceTime 30

# idle কানেকশন সামলানো (৩০০s × ২ = ~১০ মিনিট সাড়া না পেলে কেটে দেবে)
ClientAliveInterval 300
ClientAliveCountMax 2

# দরকার না থাকলে বন্ধ
X11Forwarding no
AllowAgentForwarding no
# AllowTcpForwarding no      # port forwarding লাগলে এটা বন্ধ করো না
```

### গুরুত্বপূর্ণ `sshd_config` অপশন

| অপশন | মান ও অর্থ |
|---|---|
| `Port` | শোনার port (ডিফল্ট 22) |
| `ListenAddress` | কোন IP-তে শুনবে |
| `PermitRootLogin` | `yes` / `prohibit-password` (root শুধু key দিয়ে; upstream ডিফল্ট) / `forced-commands-only` / `no` |
| `PasswordAuthentication` | `yes`/`no` — পাসওয়ার্ড লগইন |
| `KbdInteractiveAuthentication` | keyboard-interactive (পুরোনো নাম `ChallengeResponseAuthentication`) |
| `PubkeyAuthentication` | key লগইন (ডিফল্ট `yes`) |
| `AuthorizedKeysFile` | key ফাইলের পাথ (ডিফল্ট `.ssh/authorized_keys .ssh/authorized_keys2`) |
| `AllowUsers` / `DenyUsers` | ইউজারের সাদা/কালো তালিকা (`rahim@10.0.0.*` ধরনের pattern চলে) |
| `AllowGroups` / `DenyGroups` | গ্রুপের সাদা/কালো তালিকা |
| `MaxAuthTries` | একটা কানেকশনে কতবার চেষ্টা |
| `MaxSessions` | একটা কানেকশনে কয়টা session (multiplexing) |
| `LoginGraceTime` | লগইন শেষ করার সময়সীমা |
| `ClientAliveInterval` / `ClientAliveCountMax` | সার্ভার থেকে keepalive |
| `X11Forwarding` | X11 forwarding |
| `AllowTcpForwarding` | port forwarding (`yes`/`no`/`local`/`remote`) |
| `GatewayPorts` | remote forward অন্য মেশিনের জন্য খোলা হবে কিনা |
| `AllowAgentForwarding` | agent forwarding |
| `StrictModes` | ইউজারের home/.ssh ফাইলের পারমিশন যাচাই (ডিফল্ট `yes`, রেখে দাও) |
| `UsePAM` | PAM ব্যবহার (Debian/Ubuntu/RHEL-এ `yes`) |
| `Banner` | লগইনের আগে দেখানো বার্তার ফাইল |
| `Match` | শর্তসাপেক্ষ সেটিং (নিচে দেখো) |

`Match` দিয়ে নির্দিষ্ট ইউজার/গ্রুপ/IP-এর জন্য আলাদা নিয়ম:

```
# শুধু অফিস নেটওয়ার্ক থেকে পাসওয়ার্ড লগইন চলবে
Match Address 10.10.0.0/16
    PasswordAuthentication yes

# sftponly গ্রুপ শুধু ফাইল আদান-প্রদান করতে পারবে, শেল পাবে না
Match Group sftponly
    ChrootDirectory /srv/sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```

> 📝 **মনে রাখো:** `Match` ব্লক পরের `Match` বা ফাইলের শেষ পর্যন্ত চলে। তাই `Match` সবসময় ফাইলের শেষের দিকে রাখো। `ChrootDirectory`-র পাথটা root-এর মালিকানায় থাকতে হবে এবং অন্য কেউ লিখতে পারবে না।

### নিরাপদে পরিবর্তন প্রয়োগ — লক-আউট এড়ানোর নিয়ম

রিমোট সার্ভারের SSH কনফিগ ভুল করা মানে নিজের ঘরের চাবি ভেতরে রেখে দরজা বন্ধ করা। এই ধাপগুলো মেনে চলো:

```bash
# ১. key লগইন আগে থেকেই কাজ করছে কিনা নিশ্চিত হও (নতুন টার্মিনাল থেকে)
$ ssh rahim@server1 'echo key-login-OK'

# ২. সিনট্যাক্স পরীক্ষা (কিছু না দেখালে ঠিক আছে)
$ sudo sshd -t

# ৩. কার্যকর মানগুলো দেখো
$ sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|port) '
port 22
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no

# ৪. reload (চলমান কানেকশন কাটে না)
$ sudo systemctl reload ssh      # Debian/Ubuntu
$ sudo systemctl reload sshd     # Fedora/RHEL

# ৫. পুরোনো সেশন খোলা রেখেই, নতুন টার্মিনাল থেকে লগইন পরীক্ষা করো
```

> ⚠️ **সতর্কতা:** পরীক্ষা সফল না হওয়া পর্যন্ত বর্তমান SSH সেশন **বন্ধ করবে না**। `sshd` restart/reload করলেও চলমান সেশন টিকে থাকে — এটাই তোমার জীবনরক্ষাকারী দড়ি।

### Port বদলানো — distro-ভেদে বিশেষ ধাপ

Port 22 থেকে অন্য port-এ নিলে bot-এর লগের আবর্জনা কমে (নিরাপত্তা তেমন বাড়ে না, তবে লগ পরিষ্কার থাকে)।

```
Port 2222
```

তারপর:

- **ফায়ারওয়াল:** নতুন port খোলো, পুরোনোটা বন্ধ করার আগে পরীক্ষা করো ([অধ্যায় ৩৯](39-firewall.md))।
  ```bash
  $ sudo ufw allow 2222/tcp                                         # Ubuntu
  $ sudo firewall-cmd --permanent --add-port=2222/tcp && sudo firewall-cmd --reload   # Fedora/RHEL
  ```
- **SELinux (Fedora/RHEL):** SELinux জানে sshd শুধু `ssh_port_t` টাইপের port-এ বসতে পারে। নতুন port যোগ না করলে sshd চালুই হবে না:
  ```bash
  $ sudo semanage port -a -t ssh_port_t -p tcp 2222
  $ sudo systemctl restart sshd
  ```
- **Ubuntu 24.04 (socket activation):** port শোনে `ssh.socket`। `sshd_config`-এর `Port` থেকে একটা systemd generator socket সেটিং তৈরি করে, তাই বদলানোর পর:
  ```bash
  $ sudo systemctl daemon-reload
  $ sudo systemctl restart ssh.socket
  ```

> 💡 **টিপস:** আরও নিরাপত্তার জন্য fail2ban (বারবার ভুল চেষ্টা করা IP ব্লক করা), 2FA ইত্যাদি [অধ্যায় ৪৫](45-security-hardening.md)-এ আছে। SSH লগ দেখতে: `journalctl -u ssh` (Ubuntu) বা `journalctl -u sshd` (RHEL); ফাইলে `/var/log/auth.log` (Debian/Ubuntu) বা `/var/log/secure` (RHEL) — বিস্তারিত [অধ্যায় ২৮](28-logs.md)-এ।

---

## ৩৮.১০ পারমিশন সমস্যা — "কেন key কাজ করছে না?"

Key ঠিকঠাক বসিয়েছ, তবুও পাসওয়ার্ড চাইছে বা "Permission denied (publickey)" দেখাচ্ছে? ৯০% ক্ষেত্রে কারণ **ফাইল পারমিশন**।

`sshd`-এর `StrictModes yes` (ডিফল্ট) নিয়ম: যদি অন্য কেউ তোমার `authorized_keys` বদলাতে পারে, তাহলে সে নিজের key ঢুকিয়ে তোমার অ্যাকাউন্টে ঢুকতে পারবে। তাই sshd এমন অবস্থায় key-টা বিশ্বাসই করে না।

```
সার্ভার সাইড (যার অ্যাকাউন্টে ঢুকছ):
  /home/rahim                    755 বা 750 বা 700  (group/other লিখতে পারবে না!)
  /home/rahim/.ssh               700  drwx------
  /home/rahim/.ssh/authorized_keys  600  -rw-------
  মালিক: rahim (root না!)

ক্লায়েন্ট সাইড:
  ~/.ssh                         700
  ~/.ssh/id_ed25519   (private)  600  -rw-------
  ~/.ssh/id_ed25519.pub          644
  ~/.ssh/config                  600
  ~/.ssh/known_hosts             644 বা 600
```

ঠিক করার কমান্ড:

```bash
# সার্ভারে
$ chmod go-w ~
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys
$ sudo chown -R rahim:rahim /home/rahim/.ssh     # root দিয়ে বানালে মালিক ভুল হয়

# ক্লায়েন্টে
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/id_ed25519 ~/.ssh/config
```

ক্লায়েন্টের private key বেশি খোলা থাকলে ssh নিজেই প্রত্যাখ্যান করে:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for '/home/rahim/.ssh/id_ed25519' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
```

পারমিশনের মূল ধারণা জানতে [অধ্যায় ১৬](16-permissions.md) দেখো।

> 💡 **টিপস (Fedora/RHEL):** পারমিশন ঠিক থাকলেও key কাজ না করলে SELinux context দেখো — বিশেষ করে `.ssh` ফোল্ডার অন্য জায়গা থেকে কপি/move করলে। ঠিক করতে: `restorecon -Rv ~/.ssh`।

### ট্রাবলশুটিং: `ssh -v`

সমস্যা হলে সবার আগে verbose মোড:

```bash
$ ssh -v rahim@server1
OpenSSH_9.6p1 Ubuntu-3ubuntu13.5, OpenSSL 3.0.13 30 Jan 2024
debug1: Reading configuration data /home/rahim/.ssh/config
debug1: Connecting to server1 [192.168.1.50] port 22.
debug1: Connection established.
...
debug1: Authentications that can continue: publickey,password
debug1: Offering public key: /home/rahim/.ssh/id_ed25519 ED25519 SHA256:9fT2... agent
debug1: Authentications that can continue: publickey,password
debug1: Next authentication method: password
```

"Offering public key" এর পর সার্ভার আবার "can continue" বললে মানে সার্ভার key-টা গ্রহণ করেনি — সার্ভারের পারমিশন বা `authorized_keys`-এর লেখা দেখো। সার্ভারের দিকের কারণ সার্ভারের লগে পাবে:

```bash
$ sudo journalctl -u ssh -n 20
... sshd[1234]: Authentication refused: bad ownership or modes for directory /home/rahim/.ssh
```

| Error | সম্ভাব্য কারণ |
|---|---|
| `Connection refused` | sshd চলছে না, বা ভুল port |
| `Connection timed out` | ফায়ারওয়াল ব্লক করছে, host বন্ধ, রাউটিং সমস্যা |
| `No route to host` | নেটওয়ার্ক/ফায়ারওয়াল (REJECT) |
| `Permission denied (publickey)` | key মেলেনি, পারমিশন সমস্যা, ভুল ইউজার, পাসওয়ার্ড লগইন বন্ধ |
| `Host key verification failed` | known_hosts-এর key মিলছে না |
| `Too many authentication failures` | agent-এ অনেক key; `IdentitiesOnly yes` + `IdentityFile` দাও |
| `Could not resolve hostname` | DNS সমস্যা বা ভুল নাম ([অধ্যায় ৩৭](37-network-commands.md)) |

---

## ৩৮.১১ ফাইল কপি: `scp`

`scp` (secure copy) হলো SSH-এর উপর দিয়ে `cp`।

### Syntax

```
scp [options] SOURCE... DESTINATION
```

রিমোট পাথ লেখার নিয়ম: `user@host:path`। কোলন (`:`) না দিলে scp এটাকে লোকাল ফাইলের নাম ভাববে!

```bash
# লোকাল → রিমোট
$ scp report.pdf rahim@server1:/home/rahim/docs/
report.pdf                         100%  2048KB  10.5MB/s   00:00

# রিমোট → লোকাল (. = বর্তমান ডিরেক্টরি)
$ scp rahim@server1:/var/log/nginx/error.log .

# পুরো ফোল্ডার
$ scp -r ./website rahim@server1:/tmp/

# ভিন্ন port (বড় হাতের P!)
$ scp -P 2222 file.txt rahim@server1:~

# দুই রিমোটের মধ্যে, তোমার মেশিন দিয়ে ঘুরে
$ scp -3 server1:/data/a.tar server2:/backup/
```

| অপশন | কাজ |
|---|---|
| `-r` | recursive (ফোল্ডার) |
| `-P PORT` | port (বড় হাতের, ssh-এর `-p`-র মতো না!) |
| `-p` | modification time ও mode রক্ষা করে |
| `-i FILE` | identity key |
| `-C` | compression |
| `-l KBIT` | bandwidth সীমা (Kbit/s) |
| `-3` | দুই রিমোটের মধ্যে কপি লোকাল দিয়ে ঘুরিয়ে |
| `-J HOST` | jump host |
| `-O` | পুরোনো SCP প্রোটোকল ব্যবহার |
| `-q` | progress bar বন্ধ |

> 📝 **মনে রাখো:** OpenSSH 9.0 থেকে `scp` ভেতরে ভেতরে পুরোনো SCP প্রোটোকলের বদলে **SFTP প্রোটোকল** ব্যবহার করে (পুরোনোটার নিরাপত্তা ত্রুটি ছিল)। খুব পুরোনো সার্ভারে SFTP না থাকলে `-O` দাও।

> ⚠️ **সতর্কতা:** `scp` বারবার সিঙ্ক বা বিশাল ডেটার জন্য ভালো না — মাঝপথে কেটে গেলে আবার শুরু থেকে কপি করে। এ কাজে `rsync` (৩৮.১৩) ব্যবহার করো।

---

## ৩৮.১২ `sftp` — ইন্টারঅ্যাকটিভ ফাইল ট্রান্সফার

`sftp` হলো FTP-র মতো ইন্টারঅ্যাকটিভ সেশন, কিন্তু SSH-এর ভেতরে (FTP-র সাথে প্রোটোকলগত কোনো সম্পর্ক নেই)। সার্ভারে ফাইল ব্রাউজ করে দেখে তারপর নামাতে চাইলে সুবিধাজনক।

```bash
$ sftp rahim@server1
Connected to server1.
sftp> pwd
Remote working directory: /home/rahim
sftp> ls
docs   backup.tar.gz   notes.txt
sftp> lcd ~/Downloads
sftp> get backup.tar.gz
Fetching /home/rahim/backup.tar.gz to backup.tar.gz
backup.tar.gz                      100%   45MB  11.2MB/s   00:04
sftp> put photo.jpg docs/
Uploading photo.jpg to /home/rahim/docs/photo.jpg
sftp> bye
```

ভিন্ন port: `sftp -P 2222 rahim@server1` (এখানেও বড় হাতের P)।

| sftp কমান্ড | কাজ |
|---|---|
| `ls`, `cd`, `pwd`, `mkdir`, `rm`, `rmdir`, `rename` | **রিমোটে** কাজ |
| `lls`, `lcd`, `lpwd`, `lmkdir` | **লোকালে** কাজ (শুরুতে `l` = local) |
| `get FILE` | রিমোট → লোকাল |
| `get -r DIR` | ফোল্ডার নামানো |
| `put FILE` | লোকাল → রিমোট |
| `put -r DIR` | ফোল্ডার তোলা |
| `reget` / `reput` | মাঝপথে থেমে যাওয়া ট্রান্সফার আবার চালু (resume) |
| `chmod`, `chown` | রিমোট পারমিশন |
| `df -h` | রিমোট ডিস্কের ফাঁকা জায়গা |
| `!cmd` | লোকাল শেলে কমান্ড |
| `help` বা `?` | সাহায্য |
| `bye` / `exit` / `quit` | বের হওয়া |

স্ক্রিপ্টে ব্যবহারের জন্য batch মোড:

```bash
$ cat cmds.txt
cd /var/backups
get db-latest.sql.gz
$ sftp -b cmds.txt rahim@server1
```

> 💡 **টিপস:** GUI চাইলে বেশিরভাগ ফাইল ম্যানেজারে (GNOME Files, Dolphin) address bar-এ `sftp://rahim@server1/` লিখলেই রিমোট ফোল্ডার খুলে যায়। আর রিমোট ফোল্ডারকে লোকাল ফোল্ডারের মতো mount করতে `sshfs` আছে: `sshfs rahim@server1:/var/www ~/mnt/www` (খোলা: `fusermount3 -u ~/mnt/www`)।

---

## ৩৮.১৩ `rsync` over SSH — স্মার্ট সিঙ্ক

`rsync` শুধু **পরিবর্তিত অংশটুকু** পাঠায় — তাই দ্বিতীয়বার চালালে অনেক দ্রুত। রিমোট পাথ দিলে (`host:path`) ডিফল্টভাবেই SSH ব্যবহার করে। এখানে SSH-সংক্রান্ত দিকটা দেখাচ্ছি; rsync-এর পুরো গাইড [অধ্যায় ৩৫](35-backup-rsync.md)-এ।

```bash
# লোকাল ফোল্ডার → সার্ভার
$ rsync -avz --progress ./website/ rahim@server1:/var/www/html/

# ভিন্ন port বা key — -e দিয়ে ssh কমান্ড বলে দাও
$ rsync -avz -e "ssh -p 2222 -i ~/.ssh/id_work" ./data/ deploy@web1:/srv/data/

# config-এ alias থাকলে আরও সহজ
$ rsync -avz ./data/ web1:/srv/data/

# আগে শুধু দেখো কী হবে (dry run), তারপর --delete দিয়ে হুবহু মিরর
$ rsync -avzn --delete ./website/ web1:/var/www/html/
$ rsync -avz  --delete ./website/ web1:/var/www/html/
```

| অপশন | কাজ |
|---|---|
| `-a` | archive: recursive + পারমিশন, সময়, symlink ইত্যাদি রক্ষা |
| `-v` | verbose |
| `-z` | ট্রান্সফারের সময় compression |
| `-P` | `--partial --progress` (resume + progress) |
| `-n` / `--dry-run` | কিছু না করে শুধু দেখায় |
| `--delete` | গন্তব্যে যা উৎসে নেই তা মুছে ফেলে |
| `-e "ssh ..."` | কোন remote shell ও তার অপশন |
| `--exclude=PATTERN` | বাদ দেওয়া |

> ⚠️ **সতর্কতা:** উৎসের শেষে `/` থাকা-না-থাকায় ফল আলাদা: `./website/` মানে "ফোল্ডারের **ভেতরের জিনিস**", `./website` মানে "ফোল্ডারটা **সহ**"। `--delete`-এর সাথে ভুল করলে গন্তব্যের ডেটা মুছে যেতে পারে — আগে `-n` দিয়ে দেখো।

> 📝 **মনে রাখো:** rsync-এর জন্য দুই মেশিনেই `rsync` ইনস্টল থাকতে হবে। নইলে "rsync: command not found" দেখাবে।

---

## ৩৮.১৪ Port Forwarding (SSH Tunneling)

SSH কানেকশনটাকে একটা **নিরাপদ সুড়ঙ্গ (tunnel)** হিসেবে ব্যবহার করে অন্য TCP ট্রাফিক পাঠানো যায়। এর তিন ধরন।

### ক) Local forwarding: `-L` — "দূরের জিনিস কাছে আনো"

```
ssh -L [bind_address:]LOCAL_PORT:DEST_HOST:DEST_PORT user@ssh_server
```

**উদাহরণ:** server1-এ MySQL চলছে, কিন্তু শুধু `127.0.0.1:3306`-এ শোনে (বাইরে থেকে বন্ধ — নিরাপত্তার জন্য ভালো)। তুমি তোমার ল্যাপটপের GUI ক্লায়েন্ট দিয়ে সেটা ব্যবহার করতে চাও:

```bash
$ ssh -N -L 3307:localhost:3306 rahim@server1
```

এখন ল্যাপটপে `127.0.0.1:3307`-এ কানেক্ট করলে সেটা আসলে server1-এর MySQL-এ পৌঁছায়।

```
  ল্যাপটপ                                       server1
 ┌──────────────────────┐                ┌──────────────────────────┐
 │ MySQL GUI            │                │                          │
 │   │ connect to       │                │                          │
 │   ▼ 127.0.0.1:3307   │                │   "localhost:3306"       │
 │ ssh client ══════════╪══ SSH tunnel ══╪══► sshd ──► MySQL :3306  │
 │ (শুনছে :3307)        │  (port 22,     │  (DEST_HOST-কে server1-এর │
 └──────────────────────┘   encrypted)   │   দৃষ্টিকোণ থেকে বোঝা হয়) │
                                         └──────────────────────────┘
```

> 📝 **মনে রাখো:** `-L`-এ `DEST_HOST` টা **SSH সার্ভারের দৃষ্টিকোণ থেকে** বোঝা হয়। `localhost` মানে server1 নিজে। তুমি চাইলে server1-এর পাশের কোনো মেশিনও দিতে পারো: `-L 8080:10.0.5.20:80` — তখন server1 ঐ অভ্যন্তরীণ মেশিনে কানেক্ট করবে।

ডিফল্টভাবে লোকাল port শুধু `127.0.0.1`-এ খোলে (শুধু তোমার মেশিন ব্যবহার করতে পারে)। LAN-এর অন্যদের জন্য খুলতে চাইলে `-L 0.0.0.0:3307:localhost:3306` (সাবধানে!)।

### খ) Remote forwarding: `-R` — "কাছের জিনিস দূরে দেখাও"

```
ssh -R [bind_address:]REMOTE_PORT:DEST_HOST:DEST_PORT user@ssh_server
```

**উদাহরণ:** তোমার ল্যাপটপে `localhost:3000`-এ একটা ডেভেলপমেন্ট ওয়েবসাইট চলছে। একটা পাবলিক সার্ভার (vps1) এর মাধ্যমে বন্ধুকে দেখাতে চাও, অথচ তোমার ল্যাপটপ NAT/রাউটারের পেছনে, বাইরে থেকে সরাসরি পৌঁছানো যায় না:

```bash
$ ssh -N -R 9000:localhost:3000 rahim@vps1
```

এখন vps1-এ `localhost:9000`-এ গেলে সেটা তোমার ল্যাপটপের :3000-এ পৌঁছায়।

```
  ল্যাপটপ (NAT-এর পেছনে)                          vps1 (পাবলিক)
 ┌────────────────────────┐                ┌─────────────────────────┐
 │ dev server :3000  ◄────┼═══ SSH tunnel ═╪═══ sshd (শুনছে :9000)   │
 │                        │  (ল্যাপটপই     │        ▲                │
 │ ssh client ────────────┼── কানেকশন     │        │ curl            │
 └────────────────────────┘   শুরু করেছে)   │   localhost:9000        │
                                           └─────────────────────────┘
```

এখানে `DEST_HOST` বোঝা হয় **ক্লায়েন্টের (তোমার ল্যাপটপের) দৃষ্টিকোণ থেকে**।

> ⚠️ **সতর্কতা:** ডিফল্টভাবে vps1-এর :9000 শুধু vps1-এর loopback-এ খোলে। পুরো ইন্টারনেটের জন্য খুলতে vps1-এর `sshd_config`-এ `GatewayPorts yes` (বা `clientspecified`) লাগবে — এবং এটা তোমার ল্যাপটপের সার্ভিস সারা দুনিয়ার কাছে উন্মুক্ত করে দেয়। ভেবেচিন্তে করো।

### গ) Dynamic forwarding: `-D` — নিজের SOCKS proxy

```bash
$ ssh -N -D 1080 rahim@server1
```

এখন তোমার ল্যাপটপের `127.0.0.1:1080`-এ একটা **SOCKS5 proxy** চলছে। ব্রাউজারকে এই proxy ব্যবহার করতে বললে সব ওয়েব ট্রাফিক server1 হয়ে যাবে — যেন তুমি server1-এ বসে ব্রাউজ করছ। পাবলিক Wi-Fi-তে নিরাপদে ব্রাউজ করতে বা অফিসের ভেতরের ওয়েবসাইট দেখতে কাজে লাগে।

```bash
$ curl --socks5-hostname 127.0.0.1:1080 http://intranet.company.local/
```

```
  ব্রাউজার ──SOCKS──► ssh client :1080 ══ tunnel ══► server1 ──► যেকোনো গন্তব্য
                                                     (প্রতিটা অনুরোধে গন্তব্য
                                                      আলাদা — তাই "dynamic")
```

### তিন ধরনের তুলনা

| | `-L` Local | `-R` Remote | `-D` Dynamic |
|---|---|---|---|
| port খোলে কোথায় | ক্লায়েন্টে | সার্ভারে | ক্লায়েন্টে |
| গন্তব্য | নির্দিষ্ট একটা (সার্ভারের দিক থেকে) | নির্দিষ্ট একটা (ক্লায়েন্টের দিক থেকে) | যেকোনো (SOCKS অনুযায়ী) |
| সাধারণ ব্যবহার | রিমোট DB/অ্যাডমিন প্যানেল | NAT-এর পেছনের সার্ভিস বাইরে দেখানো | proxy / নিরাপদ ব্রাউজিং |

### সাথে দরকারি অপশন

```bash
# -N: শেল খুলবে না, -f: background-এ চলে যাবে
$ ssh -fN -L 8080:localhost:80 rahim@server1

# পরে বন্ধ করতে
$ pgrep -af 'ssh -fN -L 8080'
48211 ssh -fN -L 8080:localhost:80 rahim@server1
$ kill 48211
```

`~/.ssh/config`-এ স্থায়ীভাবে:

```
Host dbtunnel
    HostName server1.example.com
    User rahim
    LocalForward 3307 localhost:3306
    ExitOnForwardFailure yes
```

তারপর শুধু `ssh -N dbtunnel`।

> 💡 **টিপস:** সার্ভারের `sshd_config`-এ `AllowTcpForwarding no` থাকলে কোনো forwarding কাজ করবে না। আর 1024-এর নিচের লোকাল port (যেমন 80) bind করতে root লাগে — তাই 8080, 3307 ধরনের উঁচু port ব্যবহার করো।

---

## ৩৮.১৫ Jump Host / Bastion: `-J`

বড় প্রতিষ্ঠানে অভ্যন্তরীণ সার্ভারগুলো ইন্টারনেট থেকে সরাসরি পৌঁছানো যায় না। একটাই প্রবেশদ্বার থাকে — **bastion** বা **jump host**। আগে সেখানে লগইন করে, তারপর ভেতরে যেতে হতো:

```bash
$ ssh rahim@bastion        # ধাপ ১
rahim@bastion:~$ ssh rahim@10.0.5.12    # ধাপ ২ — key বা agent bastion-এ লাগবে, ঝামেলা!
```

`-J` (ProxyJump) একবারে করে দেয়:

```bash
$ ssh -J rahim@bastion.example.com rahim@10.0.5.12

# একাধিক jump, কমা দিয়ে
$ ssh -J user@jump1,user@jump2 user@target

# scp/rsync-ও একইভাবে
$ scp -J bastion file.txt 10.0.5.12:/tmp/
$ rsync -av -e "ssh -J bastion" ./app/ 10.0.5.12:/srv/app/
```

```
  ল্যাপটপ                 bastion                      10.0.5.12 (ভেতরের)
 ┌─────────┐   SSH-1    ┌─────────┐   TCP forward    ┌────────────┐
 │  ssh  ══╪════════════╪═► sshd ─╪─────────────────►│   sshd     │
 │    ║    │            └─────────┘                  │            │
 │    ╚════╪════════ SSH-2 (end-to-end encrypted) ═══╪══► শেল     │
 └─────────┘   SSH-2 যায় SSH-1-এর ভেতর দিয়ে          └────────────┘
```

কীভাবে কাজ করে: ক্লায়েন্ট প্রথমে bastion-এ কানেক্ট করে, bastion-কে বলে "10.0.5.12:22-এ একটা TCP সংযোগ জুড়ে দাও" (local forwarding-এর মতো)। তারপর ঐ পাইপের ভেতর দিয়ে **তোমার ল্যাপটপ থেকে সরাসরি** target-এর সাথে আলাদা SSH session চালায়। ফলাফল:

- Authentication দুটোই তোমার ল্যাপটপের key দিয়ে হয় — bastion-এ কোনো private key রাখতে হয় না।
- bastion শুধু encrypted বাইট পার করে, ভেতরের সেশন পড়তে পারে না।
- `-A` (agent forwarding)-এর ঝুঁকি নেই।

`~/.ssh/config`-এ:

```
Host bastion
    HostName bastion.example.com
    User rahim

Host 10.0.5.* app-*
    User rahim
    ProxyJump bastion
```

এখন `ssh 10.0.5.12` নিজে থেকেই bastion হয়ে যাবে। (bastion-এ `AllowTcpForwarding` চালু থাকতে হবে।)

> 💡 **টিপস:** পুরোনো পদ্ধতি `ProxyCommand ssh -W %h:%p bastion` — এটাও একই কাজ করে, পুরোনো config-এ দেখতে পাবে।

---

## ৩৮.১৬ SSH Escape Sequences — জমে যাওয়া সেশন থেকে মুক্তি

সার্ভার রিবুট হলো বা Wi-Fi কেটে গেল — তোমার টার্মিনাল জমে আছে, `Ctrl+C` কাজ করছে না (কারণ `Ctrl+C` রিমোটে পাঠানো হয়, আর রিমোট সাড়া দিচ্ছে না)। কী করবে?

**Enter চাপো, তারপর `~` তারপর `.`** — কানেকশন সাথে সাথে কেটে যাবে।

ssh ক্লায়েন্ট তোমার কীবোর্ড ইনপুটের দিকে নজর রাখে: **নতুন লাইনের একদম শুরুতে** `~` টাইপ করলে সেটা রিমোটে না পাঠিয়ে ক্লায়েন্টের নিজের জন্য একটা কমান্ড হিসেবে ধরে নেয়। এজন্যই আগে Enter চাপতে হয়।

| Sequence | কাজ |
|---|---|
| `~.` | কানেকশন তৎক্ষণাৎ কেটে দাও (জমে যাওয়া সেশনের ওষুধ) |
| `~?` | সব escape sequence-এর তালিকা দেখাও |
| `~^Z` (`~` তারপর `Ctrl+Z`) | ssh নিজেকে suspend করো (লোকাল শেলে ফেরো; `fg` দিয়ে ফেরত) |
| `~&` | ssh-কে background-এ পাঠাও (forwarded কানেকশন শেষ হওয়ার অপেক্ষায় থাকলে) |
| `~#` | চলমান forwarded কানেকশনের তালিকা |
| `~C` | ssh কমান্ড লাইন খোলো — চলমান সেশনে নতুন forward যোগ করা যায় (যেমন `-L 8080:localhost:80`) |
| `~R` | rekey (নতুন session key) অনুরোধ |
| `~V` / `~v` | ক্লায়েন্টের log verbosity কমাও / বাড়াও |
| `~B` | রিমোটে BREAK পাঠাও |
| `~~` | আসল একটা `~` অক্ষর পাঠাও |

```
rahim@server1:~$          ← (নেটওয়ার্ক কেটে গেছে, কিছুই সাড়া দিচ্ছে না)
[Enter] ~ .
Connection to server1 closed.
rahim@laptop:~$
```

**নেস্টেড SSH** (ssh-এর ভেতরে ssh): প্রতিটা `~` প্রথম ssh ক্লায়েন্ট খেয়ে ফেলে। ভেতরের সেশন কাটতে `~~.` দাও (প্রথম ক্লায়েন্ট `~~`-কে একটা `~` হিসেবে পাঠায়, দ্বিতীয় ক্লায়েন্ট তারপর `~.` পায়)।

> 📝 **মনে রাখো:** OpenSSH 9.2 থেকে `~C` কমান্ড লাইন ডিফল্টভাবে বন্ধ; ব্যবহার করতে ক্লায়েন্ট config-এ `EnableEscapeCommandline yes` দাও। escape character বদলাতে `-e` অপশন (`ssh -e '^' host`), পুরো বন্ধ করতে `-e none`।

---

## ৩৮.১৭ আরও কিছু দরকারি জিনিস

### সংযোগ টিকিয়ে রাখা

NAT রাউটার বা ফায়ারওয়াল idle কানেকশন কিছুক্ষণ পর ভুলে যায়, ফলে সেশন জমে যায়। ক্লায়েন্ট সাইডে:

```
Host *
    ServerAliveInterval 60      # প্রতি ৬০ সেকেন্ডে encrypted "আছো?" বার্তা
    ServerAliveCountMax 3       # ৩ বার সাড়া না পেলে কানেকশন বন্ধ
```

দুর্বল/মোবাইল নেটওয়ার্কের জন্য `mosh` নামে আলাদা টুল আছে (SSH দিয়ে লগইন করে, তারপর UDP-তে চলে; নেটওয়ার্ক বদলালেও সেশন টেকে)। আর কানেকশন কাটলেও কাজ চালু রাখতে সার্ভারে `tmux` চালাও — [অধ্যায় ২৫](25-jobs-tmux.md)।

### X11 forwarding — রিমোট GUI

```bash
$ ssh -X rahim@server1 gedit
```

রিমোটের GUI অ্যাপ তোমার স্ক্রিনে আসে। সার্ভারে `X11Forwarding yes` আর `xauth` প্যাকেজ লাগবে। `-Y` হলো "trusted" মোড — কম সীমাবদ্ধ, কম নিরাপদ।

### Windows ও অন্যান্য

Windows 10/11-এ OpenSSH ক্লায়েন্ট বিল্ট-ইন — PowerShell-এ `ssh user@host` সরাসরি চলে। macOS-এও একই OpenSSH।

### SSH certificate (অ্যাডভান্সড ঝলক)

হাজারো সার্ভারে প্রতিটার `authorized_keys`-এ key বসানো ঝামেলা। বড় প্রতিষ্ঠানে একটা **SSH CA (Certificate Authority)** key দিয়ে ইউজারের key-তে সময়সীমা-সহ সই করা হয় (`ssh-keygen -s ca_key -I id -n rahim -V +1d id_ed25519.pub`), আর সার্ভার শুধু CA-কে বিশ্বাস করে (`TrustedUserCAKeys`)। এতে key বিতরণ ও বাতিল দুটোই সহজ।

---

## হাতে-কলমে অনুশীলন

> দুটো মেশিন লাগবে: একটা লিনাক্স VM (VirtualBox/KVM, [অধ্যায় ৪৮](48-containers-virtualization.md)) বা একটা সস্তা ক্লাউড VPS হলে চলবে। নিচে "ক্লায়েন্ট" = তোমার মেশিন, "সার্ভার" = VM, ধরে নিচ্ছি সার্ভারের IP `192.168.56.10`, ইউজার `student`।

### ল্যাব ১: সার্ভার প্রস্তুত ও প্রথম লগইন

```bash
# সার্ভারে
$ sudo apt install openssh-server        # বা: sudo dnf install openssh-server
$ sudo systemctl enable --now ssh        # RHEL/Fedora-তে sshd
$ ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub   # fingerprint লিখে রাখো
$ ip -4 addr                             # IP দেখো

# ক্লায়েন্টে
$ ssh student@192.168.56.10              # fingerprint মিলিয়ে yes
$ ssh student@192.168.56.10 'hostname; uptime'
```

### ল্যাব ২: Key-based লগইন

```bash
# ক্লায়েন্টে
$ ssh-keygen -t ed25519 -C "student-lab" -f ~/.ssh/id_lab
$ ssh-copy-id -i ~/.ssh/id_lab.pub student@192.168.56.10
$ ssh -i ~/.ssh/id_lab student@192.168.56.10     # passphrase চাইবে, পাসওয়ার্ড না
$ eval "$(ssh-agent -s)"; ssh-add ~/.ssh/id_lab
$ ssh -i ~/.ssh/id_lab student@192.168.56.10     # এবার কিছুই চাইবে না
```

### ল্যাব ৩: config ফাইল

`~/.ssh/config`-এ লেখো:

```
Host lab
    HostName 192.168.56.10
    User student
    IdentityFile ~/.ssh/id_lab
    IdentitiesOnly yes
```

```bash
$ chmod 600 ~/.ssh/config
$ ssh lab
$ ssh -G lab | grep -E '^(user|hostname|identityfile) '
```

### ল্যাব ৪: ইচ্ছাকৃত পারমিশন ভাঙা ও সারানো

```bash
$ ssh lab 'chmod 777 ~/.ssh'
$ ssh -v lab 2>&1 | grep -i -E 'offering|password'   # key প্রত্যাখ্যাত, পাসওয়ার্ড চাইবে
# পাসওয়ার্ড দিয়ে ঢুকে সার্ভারের লগ দেখো:
$ sudo journalctl -u ssh -n 5          # "bad ownership or modes" খোঁজো
$ chmod 700 ~/.ssh                     # সারাও, তারপর বের হয়ে আবার key-তে লগইন পরীক্ষা
```

### ল্যাব ৫: হার্ডেনিং (পুরোনো সেশন খোলা রেখে!)

```bash
# সার্ভারে
$ echo -e "PermitRootLogin no\nPasswordAuthentication no\nKbdInteractiveAuthentication no" \
  | sudo tee /etc/ssh/sshd_config.d/00-hardening.conf
$ sudo sshd -t && sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication) '
$ sudo systemctl reload ssh

# ক্লায়েন্টে, নতুন টার্মিনাল থেকে
$ ssh lab 'echo still-works'
$ ssh -o PubkeyAuthentication=no student@192.168.56.10
student@192.168.56.10: Permission denied (publickey).    # পাসওয়ার্ড লগইন বন্ধ, প্রমাণিত
```

### ল্যাব ৬: ফাইল ট্রান্সফার

```bash
$ mkdir -p ~/demo && for i in 1 2 3; do echo "file $i" > ~/demo/f$i.txt; done
$ scp -r ~/demo lab:/tmp/
$ sftp lab          # ভেতরে: cd /tmp/demo, ls, get f1.txt, bye
$ echo "changed" >> ~/demo/f2.txt
$ rsync -avz ~/demo/ lab:/tmp/demo/    # লক্ষ করো, শুধু f2.txt যাচ্ছে
```

### ল্যাব ৭: Tunnel

```bash
# সার্ভারে একটা ছোট ওয়েব সার্ভার, শুধু localhost-এ
$ python3 -m http.server 8000 --bind 127.0.0.1

# ক্লায়েন্টে
$ curl -m 3 http://192.168.56.10:8000/      # ব্যর্থ হবে — বাইরে থেকে বন্ধ
$ ssh -fN -L 9000:localhost:8000 lab
$ curl http://localhost:9000/               # কাজ করছে! টানেল দিয়ে
$ pkill -f 'ssh -fN -L 9000'
```

### ল্যাব ৮: Escape sequence

```bash
$ ssh lab
student@server:~$          # এখন Enter, তারপর ~?  — তালিকা দেখো
student@server:~$          # Enter, তারপর ~.     — কানেকশন কেটে যাবে
```

---

## সারসংক্ষেপ

| কমান্ড / শর্টকাট | কাজ |
|---|---|
| `ssh user@host` | রিমোট লগইন |
| `ssh -p 2222 user@host` | ভিন্ন port |
| `ssh host 'cmd'` / `ssh -t host cmd` | রিমোট কমান্ড / interactive কমান্ড |
| `ssh -v host` | ডিবাগ |
| `ssh -G host` | চূড়ান্ত config দেখা |
| `ssh-keygen -t ed25519 -C "comment"` | নতুন key |
| `ssh-keygen -p -f key` | passphrase বদলানো |
| `ssh-keygen -lf file.pub` | fingerprint |
| `ssh-keygen -R host` | known_hosts থেকে host মুছে ফেলা |
| `ssh-keyscan host` | host key আনা |
| `ssh-copy-id -i key.pub user@host` | public key সার্ভারে বসানো |
| `eval "$(ssh-agent -s)"` | agent চালু |
| `ssh-add`, `ssh-add -l`, `ssh-add -D` | key যোগ / তালিকা / সব মোছা |
| `sudo sshd -t` / `sudo sshd -T` | সার্ভার config পরীক্ষা / কার্যকর মান |
| `systemctl reload ssh` / `sshd` | config প্রয়োগ (Ubuntu / RHEL) |
| `scp [-r] [-P port] src user@host:dst` | ফাইল কপি |
| `sftp [-P port] user@host` | ইন্টারঅ্যাকটিভ ট্রান্সফার |
| `rsync -avz -e "ssh -p 2222" src/ host:dst/` | সিঙ্ক over SSH |
| `ssh -L lport:dhost:dport host` | local forward |
| `ssh -R rport:dhost:dport host` | remote forward |
| `ssh -D 1080 host` | SOCKS proxy |
| `ssh -fN ...` | শুধু টানেল, background-এ |
| `ssh -J jump target` | jump host |
| `Enter ~.` | জমে যাওয়া সেশন কাটা |
| `Enter ~?` | escape তালিকা |
| `Enter ~^Z` | ssh suspend |
| `Enter ~#` | forwarded কানেকশন তালিকা |
| `Ctrl+D` / `exit` | লগআউট |

**পারমিশনের চাবি-সংখ্যা:** `~/.ssh` = 700, `authorized_keys` = 600, private key = 600, `config` = 600, `.pub` = 644, home-এ group/other লিখতে পারবে না।

---

## নিজেকে যাচাই করো

1. SSH-এ key exchange কী সমস্যার সমাধান করে? Private key কি কখনো নেটওয়ার্কে যায়?
2. `known_hosts` আর `authorized_keys` — কোনটা কোথায় থাকে, কার পরিচয় রাখে?
3. "REMOTE HOST IDENTIFICATION HAS CHANGED" দেখলে প্রথমে কী করবে, আর নিশ্চিত হওয়ার পর কোন কমান্ড চালাবে?
4. তুমি `/etc/ssh/sshd_config`-এ `PasswordAuthentication no` দিয়েছ, reload করেছ, তবুও পাসওয়ার্ড লগইন হচ্ছে। সম্ভাব্য কারণ কী, কীভাবে যাচাই করবে?
5. `ssh -p` আর `scp -P` — পার্থক্য কী? `scp -p` কী করে?
6. সার্ভারের PostgreSQL শুধু `127.0.0.1:5432`-এ শোনে। তোমার ল্যাপটপের :5433 থেকে সেটা ব্যবহার করার কমান্ড লেখো।
7. `-L`, `-R`, `-D`-এর মধ্যে কোনটায় port সার্ভারে খোলে?
8. Key ঠিকমতো বসানো, তবুও "Permission denied (publickey)"। কোন তিনটা জিনিস প্রথমে দেখবে?
9. নেটওয়ার্ক কেটে গেছে, SSH টার্মিনাল জমে আছে। কী চাপবে, আর কেন আগে Enter চাপতে হয়?
10. `ssh -A` দিয়ে bastion হয়ে যাওয়ার চেয়ে `ssh -J` কেন নিরাপদ?

<details>
<summary>উত্তর দেখো</summary>

1. খোলা নেটওয়ার্কে আগে কোনো গোপন কিছু ভাগ না করেও দুই পক্ষ এমন একটা shared secret বানায় যা মাঝখানের কেউ জানতে পারে না। সেখান থেকে session key তৈরি হয়। না, private key কখনো নেটওয়ার্কে যায় না, শুধু তা দিয়ে বানানো signature যায়।
2. `known_hosts` থাকে **ক্লায়েন্টে** (`~/.ssh/known_hosts`), সার্ভারের host key রাখে, মানে সার্ভারের পরিচয়। `authorized_keys` থাকে **সার্ভারে**, যে ইউজারের অ্যাকাউন্টে ঢুকবে তার `~/.ssh/authorized_keys`-এ। এটা লগইন করার অনুমতিপ্রাপ্ত ইউজারদের public key রাখে।
3. প্রথমে নিশ্চিত হও কারণটা নিরীহ (সার্ভার রিইনস্টল, নতুন VM, অ্যাডমিন key বদলেছে)। সম্ভব হলে কনসোল থেকে নতুন fingerprint মিলিয়ে নাও। তারপর `ssh-keygen -R host` চালাও আর আবার কানেক্ট করে নতুন fingerprint যাচাই করো।
4. `sshd_config`-এর উপরের `Include /etc/ssh/sshd_config.d/*.conf` ফাইলগুলো আগে পড়া হয়, আর প্রথম মানই কার্যকর থাকে। যেমন `50-cloud-init.conf`-এ `PasswordAuthentication yes` থাকতে পারে। `sudo sshd -T | grep passwordauthentication` দিয়ে কার্যকর মান দেখো, `grep -r PasswordAuthentication /etc/ssh/` দিয়ে উৎস খোঁজো। সমাধান হলো `00-*.conf` ফাইলে সেটিং রাখা।
5. দুটোই port নির্দেশ করে, শুধু অক্ষরের কেস আলাদা: ssh-এ ছোট হাতের `-p`, scp/sftp-তে বড় হাতের `-P`। `scp -p` modification time ও mode রক্ষা করে।
6. `ssh -N -L 5433:localhost:5432 user@server`। এরপর ল্যাপটপে `127.0.0.1:5433`-এ কানেক্ট করো।
7. `-R`। Remote forwarding-এ port SSH সার্ভারে খোলে। `-L` আর `-D` খোলে ক্লায়েন্টে।
8. (ক) সার্ভারে `~/.ssh` 700 আর `authorized_keys` 600 কিনা, home ডিরেক্টরি group/other-writable কিনা, মালিক ঠিক আছে কিনা। (খ) `ssh -v` দিয়ে দেখো সঠিক key offer করা হচ্ছে কিনা আর ইউজারনেম ঠিক আছে কিনা। (গ) সার্ভারের লগ দেখো (`journalctl -u ssh`/`sshd`, `/var/log/auth.log`/`secure`)। RHEL-এ SELinux context-ও দেখো (`restorecon -Rv ~/.ssh`)।
9. `Enter`, তারপর `~`, তারপর `.`। ssh ক্লায়েন্ট escape character শুধু নতুন লাইনের শুরুতে চেনে, তাই আগে Enter দিতে হয়।
10. `-A` দিলে bastion-এর root বা আক্রমণকারী কানেকশন চলাকালীন তোমার agent socket ব্যবহার করে তোমার নামে অন্য সার্ভারে লগইন করতে পারে। `-J`-এ bastion শুধু TCP বাইট পার করে। ভেতরের SSH session তোমার ল্যাপটপ থেকে target পর্যন্ত end-to-end encrypted থাকে, আর authentication-ও তোমার ল্যাপটপেই হয়।

</details>

---

[← আগের অধ্যায়](37-network-commands.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](39-firewall.md)

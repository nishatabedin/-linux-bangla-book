# অধ্যায় ৪০: সার্ভার সেটআপ — ওয়েব সার্ভার ও সাধারণ সার্ভিস

> এতক্ষণ তুমি লিনাক্সের খুঁটিনাটি শিখেছ: ফাইল, পারমিশন, প্রসেস, systemd, নেটওয়ার্ক, SSH, ফায়ারওয়াল। এই অধ্যায়ে সব একসাথে জোড়া লাগবে। একদম ফাঁকা একটা লিনাক্স সার্ভার নিয়ে আমরা সেটাকে এমন জায়গায় নিয়ে যাব যেখানে ইন্টারনেটে একটা ওয়েবসাইট HTTPS-সহ চলছে, পেছনে একটা ডেটাবেস আছে, আর অ্যাপটা systemd দেখাশোনা করছে।

## এই অধ্যায়ে যা শিখবে

- নতুন লিনাক্স সার্ভার হাতে পেলে প্রথম ৩০ মিনিটে কী কী করতে হয় (চেকলিস্ট)
- ওয়েব সার্ভার আসলে কী, আর একটা HTTP request ভেতরে কীভাবে চলে
- nginx ইনস্টল, কনফিগ ফাইলের গঠন: `http`, `server` block, `location`, `root`, `index`, `try_files`
- nginx দিয়ে reverse proxy (Node/Python অ্যাপের সামনে বসানো)
- Apache (httpd) পরিচিতি: VirtualHost, মডিউল, `a2ensite`/`a2enmod`, nginx-এর সাথে তুলনা
- Let's Encrypt আর `certbot` দিয়ে বিনামূল্যে HTTPS, আর অটো-রিনিউ
- MySQL/MariaDB ইনস্টল, নিরাপদ করা, ডেটাবেস ও ইউজার তৈরি
- PostgreSQL ইনস্টল, `psql`, role আর `pg_hba.conf`
- PHP-FPM কী এবং nginx-এর সাথে কীভাবে যুক্ত হয়
- Node.js অ্যাপকে systemd service বানিয়ে চালানো
- nginx, Apache, ডেটাবেস আর অ্যাপের লগ কোথায় থাকে, কীভাবে পড়বে
- LAMP আর LEMP স্ট্যাকের ছবি ও তুলনা
- সাধারণ সমস্যা (502, 403, 404, পোর্ট দখল) ধরে ফেলার কৌশল

---

## ৪০.১ সার্ভার কী, সার্ভিস কী — ছোট্ট ভূমিকা

**Server** (সার্ভার) শব্দটার দুটো মানে আছে:

1. **হার্ডওয়্যার/মেশিন** অর্থে: একটা কম্পিউটার (প্রায়ই ক্লাউডের VPS), যেটা দিনরাত চালু থাকে আর অন্যদের সেবা দেয়।
2. **সফটওয়্যার** অর্থে: একটা প্রোগ্রাম যেটা কোনো পোর্টে কান পেতে (listen করে) বসে থাকে, request এলে উত্তর দেয়। যেমন nginx (ওয়েব সার্ভার), MySQL (ডেটাবেস সার্ভার), sshd (SSH সার্ভার)।

এই দ্বিতীয় ধরনের প্রোগ্রামগুলোকে লিনাক্সে **service** বা **daemon** বলে। এরা ব্যাকগ্রাউন্ডে চলে, কোনো টার্মিনালের সাথে বাঁধা থাকে না, আর systemd এদের চালু-বন্ধ-রিস্টার্ট করে ([অধ্যায় ২৬](26-systemd-services.md))।

**উপমা:** একটা রেস্টুরেন্ট ভাবো। ভবনটা হলো মেশিন। দরজার রিসেপশনিস্ট হলো nginx — সে অতিথিকে (request) স্বাগত জানায়, ঠিক টেবিলে পাঠায়। রান্নাঘরের শেফ হলো তোমার অ্যাপ (PHP/Node)। আর স্টোররুম হলো ডেটাবেস। ম্যানেজার (systemd) খেয়াল রাখে কেউ অসুস্থ হয়ে পড়লে (crash) যেন আবার কাজে ফেরে।

### একটা ওয়েব request-এর যাত্রা

ব্রাউজারে `https://example.com/products` লিখে Enter চাপলে কী হয়?

```
 ব্রাউজার                                সার্ভার (লিনাক্স)
 ────────                                ─────────────────
 1. DNS: example.com -> 203.0.113.10
 2. TCP connect 203.0.113.10:443  ───────>  kernel (ফায়ারওয়াল পার হলো)
 3. TLS handshake (সার্টিফিকেট)   <──────>  nginx (port 443 এ listen করছে)
 4. HTTP: GET /products           ───────>  nginx: কোন server block? কোন location?
                                              │
                                              ├── static ফাইল? -> ডিস্ক থেকে পাঠাও
                                              │
                                              └── ডাইনামিক? -> proxy_pass / fastcgi_pass
                                                        │
                                                        v
                                                  অ্যাপ (Node / PHP-FPM)
                                                        │
                                                        v
                                                  ডেটাবেস (MySQL / PostgreSQL)
 5. HTTP 200 + HTML               <───────  উত্তর ফেরত একই পথে
```

DNS, TCP, পোর্টের ধারণা [অধ্যায় ৩৬](36-networking-basics.md)-এ আছে; ফায়ারওয়াল [অধ্যায় ৩৯](39-firewall.md)-এ।

---

## ৪০.২ নতুন সার্ভার প্রস্তুতির চেকলিস্ট

ক্লাউড প্রোভাইডার (DigitalOcean, AWS, Hetzner ইত্যাদি) থেকে নতুন Ubuntu 24.04 সার্ভার পেলে সাধারণত শুধু একটা IP আর root লগইন (বা SSH key) পাও। কোনো অ্যাপ বসানোর আগে নিচের কাজগুলো করো। প্রতিটি ধাপের বিস্তারিত অন্য অধ্যায়ে আছে, এখানে শুধু ক্রমটা মনে রাখার জন্য একসাথে দেওয়া হলো।

| # | কাজ | কমান্ড (Ubuntu/Debian) | কেন | বিস্তারিত |
|---|-----|------------------------|-----|-----------|
| 1 | সিস্টেম আপডেট | `apt update && apt full-upgrade -y` | পুরোনো প্যাকেজে জানা নিরাপত্তা-ছিদ্র থাকে | [অধ্যায় ৩০](30-package-management.md) |
| 2 | hostname ঠিক করা | `hostnamectl set-hostname web1` | লগ আর prompt-এ মেশিন চেনা যায় | [অধ্যায় ৩৬](36-networking-basics.md) |
| 3 | timezone | `timedatectl set-timezone Asia/Dhaka` | লগের সময় বোঝা সহজ | [অধ্যায় ২৮](28-logs.md) |
| 4 | সাধারণ ইউজার তৈরি | `adduser deploy` | সবসময় root হয়ে কাজ করা বিপজ্জনক | [অধ্যায় ১৫](15-users-groups.md) |
| 5 | sudo দেওয়া | `usermod -aG sudo deploy` (RHEL: `wheel`) | দরকার হলে তবেই প্রশাসনিক ক্ষমতা | [অধ্যায় ১৮](18-sudo-root.md) |
| 6 | SSH key কপি | `ssh-copy-id deploy@IP` (নিজের পিসি থেকে) | পাসওয়ার্ডের চেয়ে অনেক নিরাপদ | [অধ্যায় ৩৮](38-ssh-remote.md) |
| 7 | SSH শক্ত করা | `PermitRootLogin no`, `PasswordAuthentication no` | brute-force আক্রমণ বন্ধ | [অধ্যায় ৩৮](38-ssh-remote.md) |
| 8 | ফায়ারওয়াল | `ufw allow OpenSSH && ufw enable` | শুধু দরকারি পোর্ট খোলা থাকবে | [অধ্যায় ৩৯](39-firewall.md) |
| 9 | অটো সিকিউরিটি আপডেট | `apt install unattended-upgrades` | ঘুমের মধ্যেও প্যাচ বসবে | [অধ্যায় ৪৫](45-security-hardening.md) |
| 10 | fail2ban (ঐচ্ছিক) | `apt install fail2ban` | বারবার ভুল লগইন করা IP ব্লক | [অধ্যায় ৪৫](45-security-hardening.md) |
| 11 | swap (ছোট RAM হলে) | `fallocate -l 2G /swapfile` ... | RAM শেষ হলে OOM-killer এড়াতে | [অধ্যায় ৩২](32-disk-partition-mount.md) |
| 12 | সময় সিঙ্ক যাচাই | `timedatectl status` | TLS আর লগের জন্য সঠিক সময় জরুরি | — |
| 13 | ব্যাকআপ পরিকল্পনা | — | ডেটা হারালে কিছুই বাঁচে না | [অধ্যায় ৩৫](35-backup-rsync.md) |

Fedora/RHEL 9-এ সমতুল্য: `dnf upgrade -y`, `usermod -aG wheel deploy`, ফায়ারওয়াল `firewall-cmd`, আর অটো আপডেটের জন্য `dnf install dnf-automatic` তারপর `systemctl enable --now dnf-automatic.timer`।

> ⚠️ **সতর্কতা:** SSH-এ `PasswordAuthentication no` করার **আগে** নতুন একটা টার্মিনাল থেকে key দিয়ে লগইন হচ্ছে কিনা পরীক্ষা করো। পুরোনো সেশন খোলা রাখো। নাহলে নিজেকেই সার্ভারের বাইরে তালাবদ্ধ করে ফেলবে। ufw চালু করার আগে `ufw allow OpenSSH` দিতেও ভুলবে না।

> 💡 **টিপস:** চেকলিস্টের ধাপগুলো একটা স্ক্রিপ্টে লিখে রাখলে প্রতিটি নতুন সার্ভারে একই কাজ মিনিটে হয়ে যায়। স্ক্রিপ্ট লেখা শিখবে [অধ্যায় ৪১](41-shell-scripting-1.md) থেকে। বড় পরিসরে মানুষ Ansible-এর মতো টুল ব্যবহার করে।

সার্ভারে এখন কোন কোন পোর্টে কে listen করছে তা দেখার অভ্যাস করো:

```
$ sudo ss -tlnp
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
LISTEN  0       4096    0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=812,fd=3))
LISTEN  0       4096    127.0.0.53%lo:53    0.0.0.0:*          users:(("systemd-resolve",pid=530,fd=14))
```

এই অধ্যায় শেষে এখানে 80, 443, 3306/5432, 3000 ইত্যাদি দেখবে।

---

## ৪০.৩ nginx — আধুনিক ওয়েব সার্ভার

**nginx** (উচ্চারণ "engine-x") একটা ওয়েব সার্ভার, reverse proxy আর load balancer। এটা খুব কম মেমরিতে একসাথে হাজার হাজার কানেকশন সামলাতে পারে, তাই আজকের বেশিরভাগ ওয়েবসাইটের সামনে এটাই থাকে।

### nginx ভেতরে কীভাবে কাজ করে

```
                ┌──────────────────────────────┐
                │  master process (root হিসেবে) │  কনফিগ পড়ে, পোর্ট 80/443 খোলে,
                │  pid: 1201                    │  worker তৈরি করে, signal সামলায়
                └──────┬───────────┬───────────┘
                       │ fork      │ fork
          ┌────────────┘           └────────────┐
          v                                     v
 ┌─────────────────────┐              ┌─────────────────────┐
 │ worker (www-data)   │              │ worker (www-data)   │   সাধারণত প্রতি CPU core-এ
 │ event loop (epoll)  │              │ event loop (epoll)  │   একটা worker
 │ হাজারো কানেকশন      │              │ হাজারো কানেকশন      │
 └─────────────────────┘              └─────────────────────┘
```

- **master process** root হিসেবে চলে, কারণ 1024-এর নিচের পোর্ট (80, 443) খুলতে বিশেষ ক্ষমতা লাগে। এটা নিজে কোনো request সামলায় না।
- **worker process** কম ক্ষমতার ইউজার হিসেবে চলে (Ubuntu/Debian-এ `www-data`, Fedora/RHEL-এ `nginx`)। ফলে কোনো worker হ্যাক হলেও আক্রমণকারী root পায় না।
- প্রতিটি worker একটা **event loop** চালায়। প্রতি কানেকশনের জন্য আলাদা thread না বানিয়ে kernel-এর `epoll` ব্যবস্থা দিয়ে জিজ্ঞেস করে, "কোন কানেকশনে এখন ডেটা এসেছে?" তারপর শুধু সেগুলোর কাজ করে। এটাকে **event-driven, non-blocking** আর্কিটেকচার বলে।

**উপমা:** পুরোনো ধাঁচের সার্ভার (যেমন Apache-এর prefork) প্রতিটি অতিথির জন্য একজন করে ওয়েটার রাখে, অতিথি মেনু পড়তে ১০ মিনিট নিলে ওয়েটারও ১০ মিনিট দাঁড়িয়ে থাকে। nginx-এর একজন ওয়েটার ঘুরে ঘুরে দেখে কোন টেবিল অর্ডার দিতে প্রস্তুত, শুধু সেখানেই যায়। তাই অল্প লোকে বেশি টেবিল সামলানো যায়।

`systemctl reload nginx` দিলে master নতুন কনফিগ পড়ে, নতুন worker চালু করে আর পুরোনো worker-দের হাতের কাজ শেষ করে বিদায় নিতে বলে। এজন্য reload-এ কোনো কানেকশন কাটে না।

### ইনস্টল

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install nginx

# Fedora / RHEL 9
sudo dnf install nginx
sudo systemctl enable --now nginx     # RHEL-এ নিজে থেকে চালু হয় না
```

Ubuntu/Debian-এ ইনস্টলের সাথে সাথেই service চালু ও enable হয়ে যায়। যাচাই করো:

```
$ systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-11-11 10:12:03 +06; 8s ago
   Main PID: 2311 (nginx)
      Tasks: 3 (limit: 2276)
     Memory: 2.6M
     CGroup: /system.slice/nginx.service
             ├─2311 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─2312 "nginx: worker process"
             └─2313 "nginx: worker process"

$ curl -I http://localhost
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Content-Type: text/html
...
```

ফায়ারওয়ালে পোর্ট খোলো:

```bash
# Ubuntu (ufw) — nginx প্যাকেজ নিজস্ব app profile দেয়
sudo ufw app list            # Nginx Full, Nginx HTTP, Nginx HTTPS
sudo ufw allow 'Nginx Full'  # 80 আর 443 দুটোই

# Fedora / RHEL (firewalld)
sudo firewall-cmd --permanent --add-service=http --add-service=https
sudo firewall-cmd --reload
```

এখন ব্রাউজারে `http://সার্ভারের-IP` খুললে "Welcome to nginx!" পাতা দেখবে।

### nginx-এর ফাইল কোথায় থাকে

| জিনিস | Ubuntu / Debian | Fedora / RHEL |
|-------|-----------------|---------------|
| মূল কনফিগ | `/etc/nginx/nginx.conf` | `/etc/nginx/nginx.conf` |
| সাইটের কনফিগ | `/etc/nginx/sites-available/` + `sites-enabled/` (symlink) | `/etc/nginx/conf.d/*.conf` |
| অতিরিক্ত কনফিগ | `/etc/nginx/conf.d/*.conf`, `/etc/nginx/snippets/` | `/etc/nginx/default.d/` |
| ডিফল্ট ওয়েব রুট | `/var/www/html` | `/usr/share/nginx/html` |
| লগ | `/var/log/nginx/access.log`, `error.log` | একই |
| worker ইউজার | `www-data` | `nginx` |

Debian-এর `sites-available` / `sites-enabled` ব্যবস্থাটা সহজ: সব সাইটের কনফিগ `sites-available`-এ রাখো, যেগুলো চালু চাও শুধু তাদের symlink ([অধ্যায় ১১](11-links-inode.md)) `sites-enabled`-এ দাও। বন্ধ করতে symlink মুছে ফেলো, আসল ফাইল থেকে যায়।

```bash
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default      # ডিফল্ট সাইট বন্ধ
```

### কনফিগের গঠন: context আর directive

nginx কনফিগ দুই রকম জিনিস দিয়ে তৈরি:

- **directive**: `নাম মান;` (শেষে সেমিকোলন বাধ্যতামূলক), যেমন `worker_processes auto;`
- **block / context**: `নাম { ... }`, যার ভেতরে আরও directive থাকে।

block-গুলো একটার ভেতরে আরেকটা, পেঁয়াজের খোসার মতো:

```
nginx.conf
┌─────────────────────────────────────────────────────────┐
│ main context: user, worker_processes, error_log, pid     │
│                                                          │
│ events { worker_connections 768; }                       │
│                                                          │
│ http {                                                   │
│   sendfile, gzip, log_format, include mime.types ...     │
│   include /etc/nginx/conf.d/*.conf;                      │
│   include /etc/nginx/sites-enabled/*;                    │
│                                                          │
│   ┌── server { listen 80; server_name a.com; ──────┐     │
│   │     root /var/www/a;                           │     │
│   │     location / { ... }                         │     │
│   │     location /api/ { proxy_pass ...; }         │     │
│   └────────────────────────────────────────────────┘     │
│   ┌── server { listen 80; server_name b.com; ──────┐     │
│   │     location / { ... }                         │     │
│   └────────────────────────────────────────────────┘     │
│ }                                                        │
└─────────────────────────────────────────────────────────┘
```

বাইরের block-এ লেখা directive ভেতরের block উত্তরাধিকারসূত্রে পায় (inherit করে), যদি না ভেতরে আবার নতুন করে লেখা হয়।

Ubuntu-র `/etc/nginx/nginx.conf`-এর গুরুত্বপূর্ণ অংশ (সংক্ষেপে):

```nginx
user www-data;
worker_processes auto;          # CPU core অনুযায়ী worker সংখ্যা
pid /run/nginx.pid;

events {
    worker_connections 768;     # প্রতি worker সর্বোচ্চ কতগুলো কানেকশন
}

http {
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;   # (নতুন সংস্করণে এটা main context-এ থাকতে পারে)

    gzip on;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

> 📝 **মনে রাখো:** মূল `nginx.conf` সাধারণত হাত দেওয়ার দরকার হয় না। নিজের সাইটের জন্য আলাদা ফাইল বানাও (Ubuntu-তে `sites-available/`, RHEL-এ `conf.d/নাম.conf`)। প্যাকেজ আপগ্রেডের সময় তাতে ঝামেলা কম হয়।

### প্রথম server block: একটা static সাইট

ধরো ডোমেইন `example.com`, ফাইল রাখব `/var/www/example`-এ।

```bash
sudo mkdir -p /var/www/example
echo '<h1>Hello from nginx</h1>' | sudo tee /var/www/example/index.html
sudo chown -R deploy:www-data /var/www/example
sudo chmod -R 755 /var/www/example
```

`/etc/nginx/sites-available/example` ফাইল:

```nginx
server {
    listen 80;                         # IPv4 পোর্ট 80
    listen [::]:80;                    # IPv6 পোর্ট 80
    server_name example.com www.example.com;

    root /var/www/example;             # ফাইল কোথায় খুঁজবে
    index index.html index.htm;        # ডিরেক্টরি চাইলে কোন ফাইল দেখাবে

    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log;

    location / {
        try_files $uri $uri/ =404;     # ফাইল আছে? ডিরেক্টরি আছে? নাহলে 404
    }
}
```

চালু করো:

```
$ sudo ln -s /etc/nginx/sites-available/example /etc/nginx/sites-enabled/
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
$ sudo systemctl reload nginx
$ curl -H 'Host: example.com' http://localhost
<h1>Hello from nginx</h1>
```

`curl -H 'Host: ...'` দিয়ে ডোমেইন DNS-এ না থাকলেও নির্দিষ্ট server block পরীক্ষা করা যায়।

> ⚠️ **সতর্কতা:** কনফিগ বদলানোর পর **সবসময়** আগে `sudo nginx -t`, তারপর reload। ভুল কনফিগে `restart` দিলে nginx উঠবেই না, আর তোমার সব সাইট বন্ধ হয়ে যাবে। `reload` ভুল কনফিগ পেলে পুরোনো কনফিগেই চলতে থাকে, তবুও `nginx -t` আগে চালানোই নিয়ম।

### nginx কোন server block বেছে নেয়?

একটা সার্ভারে দশটা সাইট থাকতে পারে, সবাই পোর্ট 80-তে। nginx ঠিক করে এভাবে:

1. যে IP:port-এ request এসেছে, সেটার সাথে মেলে এমন `listen` থাকা server block-গুলো আলাদা করে।
2. তাদের মধ্যে request-এর `Host` header-কে `server_name`-এর সাথে মেলায়: আগে হুবহু নাম, তারপর সামনে wildcard (`*.example.com`), তারপর পেছনে wildcard (`mail.*`), তারপর regex (`~^...`)।
3. কেউ না মিললে ওই পোর্টের **default server**-এ যায়: যেটায় `listen 80 default_server;` লেখা, না থাকলে কনফিগে প্রথম যেটা পাওয়া গেছে।

অচেনা Host বা সরাসরি IP দিয়ে আসা request ফেলে দিতে একটা "ক্যাচ-অল" রাখা ভালো অভ্যাস:

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    return 444;     # nginx-এর বিশেষ কোড: কোনো উত্তর না দিয়ে কানেকশন বন্ধ
}
```

### location block — URL অনুযায়ী আচরণ

`location` বলে দেয় URL-এর কোন অংশে কী করতে হবে। এর কয়েকটা ধরন আছে:

| সিনট্যাক্স | মানে | উদাহরণ |
|-----------|------|--------|
| `location /path/` | prefix মিল (সবচেয়ে লম্বা মিলটা জেতে) | `location /images/` |
| `location = /path` | হুবহু মিল, পেলে সাথে সাথে থামে | `location = /favicon.ico` |
| `location ^~ /path/` | prefix মিল; এটা সবচেয়ে লম্বা prefix হলে regex আর দেখে না | `location ^~ /static/` |
| `location ~ regex` | regex, বড়-ছোট হাতের অক্ষর আলাদা ধরে | `location ~ \.php$` |
| `location ~* regex` | regex, বড়-ছোট হাতের অক্ষর একই ধরে | `location ~* \.(jpg\|png)$` |

**nginx যেভাবে বেছে নেয়:**

```
request URI: /static/logo.png
        │
        v
 ১. কোনো "= " হুবহু মিল আছে? ── হ্যাঁ ──> সেটাই, শেষ
        │ না
        v
 ২. সব prefix location থেকে সবচেয়ে লম্বা মিলটা মনে রাখো
        │
        ├── সেটায় "^~" আছে? ── হ্যাঁ ──> সেটাই, শেষ
        │ না
        v
 ৩. regex location গুলো কনফিগে যে ক্রমে লেখা সেই ক্রমে পরীক্ষা করো
        │
        ├── প্রথম যেটা মিলল ──> সেটাই, শেষ
        │ কোনোটা না
        v
 ৪. ধাপ ২-এর সবচেয়ে লম্বা prefix ব্যবহার করো
```

> 📝 **মনে রাখো:** regex location-এ **ক্রম** গুরুত্বপূর্ণ, prefix location-এ **দৈর্ঘ্য** গুরুত্বপূর্ণ। এই পার্থক্যটা না বুঝলে "আমার location কাজ করছে না কেন" প্রশ্নে অনেক সময় নষ্ট হয়। regex নিয়ে বিস্তারিত [অধ্যায় ২০](20-grep-regex.md)-এ।

### root বনাম alias

দুটোই বলে ফাইল ডিস্কের কোথায় আছে, কিন্তু হিসাব আলাদা:

```nginx
location /images/ {
    root /var/www/data;
}
# /images/cat.png  ->  /var/www/data/images/cat.png   (location অংশ যোগ হয়)

location /images/ {
    alias /var/www/pictures/;
}
# /images/cat.png  ->  /var/www/pictures/cat.png      (location অংশ বদলে যায়)
```

> ⚠️ **সতর্কতা:** `alias` ব্যবহার করলে location-এর শেষে `/` থাকলে alias-এর শেষেও `/` দাও। `location /img { alias /var/www/pictures/; }`-এর মতো অমিল থাকলে path traversal নিরাপত্তা-ছিদ্র তৈরি হতে পারে (যেমন `/img../` দিয়ে উপরের ডিরেক্টরিতে ঢোকা)।

### try_files — ফাইল খোঁজার ক্রম

`try_files` বাম থেকে ডানে একেকটা অপশন চেষ্টা করে, প্রথম যেটা ডিস্কে পাওয়া যায় সেটা দেয়; শেষ অপশনটা হলো fallback (একটা URI বা `=কোড`)।

```nginx
# সাধারণ static সাইট
try_files $uri $uri/ =404;

# React/Vue/Angular single-page app: সব অচেনা পথ index.html-এ
try_files $uri $uri/ /index.html;

# PHP ফ্রেমওয়ার্ক (Laravel, WordPress)
try_files $uri $uri/ /index.php?$query_string;
```

### দরকারি আরও কিছু directive

```nginx
server {
    ...
    client_max_body_size 20M;        # আপলোডের সর্বোচ্চ সাইজ (ডিফল্ট 1M; বেশি হলে 413 এরর)

    location = /old-page {
        return 301 /new-page;        # স্থায়ী রিডাইরেক্ট
    }

    location /admin/ {
        allow 203.0.113.5;           # শুধু এই IP ঢুকতে পারবে
        deny all;
    }

    location ~ /\. {
        deny all;                    # .git, .env জাতীয় লুকানো ফাইল লুকাও
    }

    location ~* \.(css|js|png|jpg|svg|woff2)$ {
        expires 30d;                 # ব্রাউজার ৩০ দিন ক্যাশ রাখবে
        access_log off;
    }

    error_page 404 /404.html;
}
```

nginx-এর ভ্যারিয়েবল (`$` দিয়ে শুরু) যেগুলো বারবার দেখবে:

| ভ্যারিয়েবল | মান |
|-------------|-----|
| `$uri` | query string ছাড়া বর্তমান path, যেমন `/products/5` |
| `$request_uri` | ক্লায়েন্ট যা পাঠিয়েছে হুবহু, query সহ |
| `$args` / `$query_string` | `?`-এর পরের অংশ |
| `$host` | request-এর Host header (না থাকলে server_name) |
| `$remote_addr` | ক্লায়েন্টের IP |
| `$scheme` | `http` বা `https` |
| `$http_<নাম>` | যেকোনো request header, যেমন `$http_user_agent` |

---

## ৪০.৪ nginx দিয়ে reverse proxy

Node.js, Python (Django/Flask/FastAPI), Go বা Java অ্যাপ সাধারণত নিজেই একটা ছোট HTTP সার্ভার চালায়, যেমন `127.0.0.1:3000`-এ। তাহলে সরাসরি পোর্ট 80-তে চালাই না কেন? কারণ সামনে nginx রাখলে:

- **TLS/HTTPS** nginx একাই সামলায়, অ্যাপকে সার্টিফিকেট নিয়ে ভাবতে হয় না।
- **static ফাইল** (ছবি, CSS) nginx অনেক দ্রুত দেয়, অ্যাপের CPU বাঁচে।
- একই সার্ভারে **একাধিক অ্যাপ** আলাদা ডোমেইন বা path-এ চালানো যায়।
- অ্যাপকে root হিসেবে চালাতে হয় না (1024-এর উপরের পোর্টে চলে)।
- ধীর ক্লায়েন্ট (খারাপ মোবাইল নেটওয়ার্ক) সামলানোর ভার nginx নেয়: সে পুরো উত্তর বাফারে নিয়ে ধীরে ধীরে পাঠায়, অ্যাপ ততক্ষণে পরের request ধরে।
- **load balancing**: একাধিক অ্যাপ instance-এ ট্রাফিক ভাগ করা।

**উপমা:** "proxy" মানে প্রতিনিধি। সাধারণ (forward) proxy ক্লায়েন্টের হয়ে বাইরে যায়। **reverse proxy** সার্ভারের হয়ে দরজায় দাঁড়ায়, ক্লায়েন্ট জানেই না ভেতরে আসলে কে কাজ করছে।

```
 ইন্টারনেট                       সার্ভার
   │                ┌──────────────────────────────────────┐
   │  :443 (HTTPS)  │  nginx                                │
   └──────────────> │   ├─ /          -> static /var/www    │
                    │   ├─ /api/      -> 127.0.0.1:3000 ──> Node অ্যাপ
                    │   └─ app2.com   -> 127.0.0.1:8000 ──> Python অ্যাপ
                    └──────────────────────────────────────┘
                     (3000 আর 8000 বাইরে থেকে খোলা নয়)
```

### একটা সম্পূর্ণ reverse proxy কনফিগ

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        # আসল ক্লায়েন্টের তথ্য অ্যাপকে জানাও
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket সাপোর্ট (Socket.IO ইত্যাদি)
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 60s;
    }
}
```

কেন এই header-গুলো? nginx-এর পেছনে থাকা অ্যাপের চোখে প্রতিটি request আসছে `127.0.0.1` থেকে (nginx থেকে)। `X-Real-IP` আর `X-Forwarded-For` না দিলে অ্যাপ আসল ব্যবহারকারীর IP জানতে পারবে না; `X-Forwarded-Proto` না দিলে অ্যাপ ভাববে request HTTP-তে এসেছে আর অনেক সময় অসীম রিডাইরেক্ট লুপ বানাবে।

> 💡 **টিপস:** WebSocket না থাকলে সব request-এ `Connection "upgrade"` পাঠানো আদর্শ নয়। প্রফেশনাল কনফিগে `http` context-এ একটা `map` বানানো হয়:
> ```nginx
> map $http_upgrade $connection_upgrade {
>     default upgrade;
>     ''      close;
> }
> ```
> তারপর location-এ `proxy_set_header Connection $connection_upgrade;`।

### proxy_pass-এর শেষ স্ল্যাশের রহস্য

এটা নতুনদের সবচেয়ে প্রিয় ফাঁদ:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;     # URI অংশ নেই
}
# /api/users  ->  অ্যাপ পায় /api/users

location /api/ {
    proxy_pass http://127.0.0.1:3000/;    # শেষে "/" (URI অংশ আছে)
}
# /api/users  ->  অ্যাপ পায় /users   (/api/ অংশ "/" দিয়ে বদলে গেল)
```

নিয়ম: `proxy_pass`-এ হোস্ট:পোর্টের পরে কোনো path (এমনকি একটা `/`) থাকলে location-এর মিলে যাওয়া অংশটা সেই path দিয়ে প্রতিস্থাপিত হয়। না থাকলে URI যেমন ছিল তেমনই যায়।

### load balancing (upstream)

```nginx
upstream app_backend {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
    server 10.0.0.12:3000 backup;   # বাকিরা মরলে তবেই
}

server {
    listen 80;
    server_name app.example.com;
    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
    }
}
```

ডিফল্টে nginx পালাক্রমে (round-robin) পাঠায়। `least_conn;` লিখলে যার হাতে কম কানেকশন তাকে দেয়, `ip_hash;` লিখলে একই IP সবসময় একই সার্ভারে যায়।

> ⚠️ **সতর্কতা (RHEL/Fedora):** SELinux ([অধ্যায় ৪৫](45-security-hardening.md)) ডিফল্টে ওয়েব সার্ভারকে অন্য পোর্টে নেটওয়ার্ক কানেকশন খুলতে দেয় না। reverse proxy-তে **502 Bad Gateway** আর error.log-এ `connect() ... failed (13: Permission denied)` দেখলে চালাও:
> ```bash
> sudo setsebool -P httpd_can_network_connect 1
> ```
> ওয়েব রুট `/var/www`-এর বাইরে রাখলে ফাইলের SELinux লেবেলও ঠিক করতে হয় (`semanage fcontext` + `restorecon`)।

---

## ৪০.৫ Apache HTTP Server পরিচিতি

**Apache HTTP Server** ১৯৯৫ সাল থেকে চলছে, দীর্ঘদিন বিশ্বের এক নম্বর ওয়েব সার্ভার ছিল। এখনো শেয়ার্ড হোস্টিং, cPanel আর পুরোনো PHP প্রজেক্টে খুব প্রচলিত। প্যাকেজের নাম distro ভেদে আলাদা, এটা সবচেয়ে বড় বিভ্রান্তি:

| জিনিস | Ubuntu / Debian | Fedora / RHEL |
|-------|-----------------|---------------|
| প্যাকেজ ও service | `apache2` | `httpd` |
| মূল কনফিগ | `/etc/apache2/apache2.conf` | `/etc/httpd/conf/httpd.conf` |
| সাইট কনফিগ | `/etc/apache2/sites-available/` + `sites-enabled/` | `/etc/httpd/conf.d/*.conf` |
| মডিউল | `/etc/apache2/mods-available/` + `mods-enabled/` | `/etc/httpd/conf.modules.d/` |
| পোর্ট | `/etc/apache2/ports.conf` | `httpd.conf`-এর `Listen` |
| লগ | `/var/log/apache2/` | `/var/log/httpd/` |
| কনফিগ পরীক্ষা | `sudo apache2ctl configtest` | `sudo apachectl configtest` |
| ইউজার | `www-data` | `apache` |

```bash
# Ubuntu
sudo apt install apache2
sudo ufw allow 'Apache Full'

# RHEL / Fedora
sudo dnf install httpd
sudo systemctl enable --now httpd
```

> ⚠️ **সতর্কতা:** nginx আর Apache দুটোই ডিফল্টে পোর্ট 80 চায়। একসাথে ইনস্টল করলে যে পরে চালু হবে সে `Address already in use` এরর দিয়ে ব্যর্থ হবে। একটা রাখো, অথবা একটাকে অন্য পোর্টে সরাও (যেমন nginx সামনে 80-তে, Apache পেছনে 8080-এ)।

### VirtualHost

nginx-এর `server` block-এর Apache সংস্করণ হলো `<VirtualHost>`। `/etc/apache2/sites-available/example.conf`:

```apache
<VirtualHost *:80>
    ServerName example.com
    ServerAlias www.example.com
    DocumentRoot /var/www/example

    <Directory /var/www/example>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/example_error.log
    CustomLog ${APACHE_LOG_DIR}/example_access.log combined
</VirtualHost>
```

- `Options -Indexes`: ফোল্ডারে index ফাইল না থাকলে ফাইলের তালিকা দেখাবে না।
- `AllowOverride All`: ডিরেক্টরির ভেতরে `.htaccess` ফাইল দিয়ে কনফিগ বদলানো যাবে (WordPress-এর জন্য দরকার)।
- `Require all granted`: সবাইকে ঢুকতে দাও।

### Debian-এর সহায়ক কমান্ড

| কমান্ড | কাজ |
|--------|-----|
| `sudo a2ensite example` | `sites-enabled`-এ symlink বানায় (সাইট চালু) |
| `sudo a2dissite 000-default` | সাইট বন্ধ |
| `sudo a2enmod rewrite` | মডিউল চালু (URL rewrite) |
| `sudo a2enmod proxy proxy_http headers ssl` | একসাথে কয়েকটা মডিউল |
| `sudo a2dismod autoindex` | মডিউল বন্ধ |
| `sudo a2enconf নাম` / `a2disconf` | `conf-available`-এর কনফিগ চালু/বন্ধ |
| `apache2ctl -S` | কোন VirtualHost কোন পোর্টে, তার তালিকা |
| `apache2ctl -M` | লোড হওয়া মডিউলের তালিকা |

এরপর `sudo systemctl reload apache2`। RHEL-এ এসব helper নেই; সরাসরি `conf.d`-তে ফাইল রাখো বা মুছে ফেলো।

### Apache-এর MPM — ভেতরের কাজের ধরন

Apache কীভাবে request সামলাবে তা ঠিক করে **MPM (Multi-Processing Module)**:

| MPM | কৌশল | কখন |
|-----|------|-----|
| `prefork` | প্রতি কানেকশনে একটা আলাদা process, thread নেই | পুরোনো `mod_php` (যেটা thread-safe নয়) |
| `worker` | process-এর ভেতরে অনেক thread | মাঝামাঝি |
| `event` | `worker`-এর উন্নত রূপ; idle keep-alive কানেকশন আলাদা thread সামলায় | আধুনিক ডিফল্ট, PHP-FPM-এর সাথে |

Ubuntu 24.04 আর RHEL 9-এ ডিফল্ট `event`। কিন্তু Ubuntu-তে `libapache2-mod-php` ইনস্টল করলে সেটা নির্ভরতা হিসেবে `prefork`-এ বদলে দেয়।

### nginx বনাম Apache

| বিষয় | nginx | Apache |
|-------|-------|--------|
| আর্কিটেকচার | event-driven, অল্প worker | MPM (process/thread), event MPM-ও আছে |
| static ফাইল ও অনেক কানেকশন | খুব দ্রুত, কম মেমরি | ভালো, তবে বেশি মেমরি |
| PHP | সবসময় বাইরে (PHP-FPM) | `mod_php` (ভেতরে) বা PHP-FPM |
| `.htaccess` | নেই (সব কনফিগ কেন্দ্রে) | আছে, প্রতি ফোল্ডারে কনফিগ |
| কনফিগ ভাষা | সংক্ষিপ্ত block | XML-ঘেঁষা ট্যাগ |
| মডিউল | বেশিরভাগ কম্পাইল-টাইমে, কিছু dynamic | রানটাইমে চালু/বন্ধ |
| সাধারণ ব্যবহার | reverse proxy, আধুনিক স্ট্যাক | শেয়ার্ড হোস্টিং, পুরোনো অ্যাপ |

`.htaccess` সুবিধাজনক কিন্তু ধীর: প্রতিটি request-এ Apache-কে পথের প্রতিটি ফোল্ডারে `.htaccess` খুঁজতে হয়। nginx এই নমনীয়তা ছেড়ে গতি নিয়েছে।

---

## ৪০.৬ HTTPS — Let's Encrypt ও certbot

### HTTPS কেন, আর কীভাবে কাজ করে

HTTP-তে সব ডেটা খোলা চিঠির মতো যায়; মাঝপথে যে কেউ (একই Wi-Fi, ISP) পাসওয়ার্ড পড়তে পারে। **HTTPS** = HTTP + **TLS** (Transport Layer Security)। TLS দুটো কাজ করে:

1. **এনক্রিপশন:** ডেটা খামে ভরে সিলমোহর দেওয়া।
2. **পরিচয় যাচাই:** সার্ভার একটা **certificate** দেখায়, যেটা একটা বিশ্বস্ত **CA (Certificate Authority)** সই করেছে। ব্রাউজারে আগে থেকেই CA-দের তালিকা থাকে।

**উপমা:** সার্টিফিকেট হলো জাতীয় পরিচয়পত্র, CA হলো নির্বাচন কমিশন। যে কেউ বলতে পারে "আমি example.com", কিন্তু কমিশনের সই করা কার্ড দেখাতে পারে শুধু আসল মালিক।

```
ব্রাউজার                                   nginx
   │── ClientHello (সাপোর্টেড cipher) ────────>│
   │<── ServerHello + certificate (public key) ─│
   │   ব্রাউজার যাচাই: CA-র সই ঠিক? ডোমেইন মেলে? মেয়াদ আছে?
   │── key exchange ───────────────────────────>│
   │        দুই পক্ষ একই session key পায়        │
   │<══════ এখন থেকে সব ডেটা এনক্রিপ্টেড ══════>│
```

সার্ভারে দুটো ফাইল লাগে: **certificate** (পাবলিক, সবাইকে দেখানো হয়) আর **private key** (গোপন, কখনো শেয়ার করবে না)।

**Let's Encrypt** একটা অলাভজনক CA যারা বিনামূল্যে, স্বয়ংক্রিয়ভাবে সার্টিফিকেট দেয়। এরা **ACME** প্রোটোকলে যাচাই করে যে ডোমেইনটা সত্যিই তোমার নিয়ন্ত্রণে। সার্টিফিকেটের মেয়াদ বর্তমানে **৯০ দিন** (Let's Encrypt ভবিষ্যতে আরও কমানোর ঘোষণা দিয়েছে), তাই অটো-রিনিউ বাধ্যতামূলক। **certbot** হলো সবচেয়ে প্রচলিত ACME ক্লায়েন্ট।

### পূর্বশর্ত

- ডোমেইনের DNS A (এবং IPv6 থাকলে AAAA) রেকর্ড তোমার সার্ভারের IP-তে নির্দেশ করছে (`dig +short example.com` দিয়ে যাচাই, [অধ্যায় ৩৭](37-network-commands.md))।
- পোর্ট 80 বাইরে থেকে খোলা (HTTP-01 যাচাই পোর্ট 80 দিয়েই হয়)।
- nginx-এ ওই ডোমেইনের `server_name` সহ server block আছে।

### certbot ইনস্টল

```bash
# Ubuntu — certbot প্রকল্পের নিজস্ব সুপারিশ: snap
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# অথবা distro-র প্যাকেজ (Ubuntu/Debian)
sudo apt install certbot python3-certbot-nginx     # Apache হলে python3-certbot-apache

# Fedora
sudo dnf install certbot python3-certbot-nginx
# RHEL 9: আগে EPEL রিপোজিটরি চালু করো (অধ্যায় ৩০ দেখো), তারপর একই কমান্ড
```

### সার্টিফিকেট নেওয়া

```
$ sudo certbot --nginx -d example.com -d www.example.com
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): admin@example.com
...
Requesting a certificate for example.com and www.example.com

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/example.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/example.com/privkey.pem
This certificate expires on 2027-02-09.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

Deploying certificate
Successfully deployed certificate for example.com to /etc/nginx/sites-enabled/example
Successfully deployed certificate for www.example.com to /etc/nginx/sites-enabled/example
Congratulations! You have successfully enabled HTTPS on https://example.com and https://www.example.com
```

`--nginx` plugin নিজেই তিনটা কাজ করে: যাচাই করে, সার্টিফিকেট আনে, আর তোমার server block-এ `listen 443 ssl` ও certificate-এর লাইন যোগ করে, সাথে HTTP থেকে HTTPS-এ রিডাইরেক্ট বসায়। কনফিগ শেষে দেখতে মোটামুটি এমন হয়:

```nginx
server {
    server_name example.com www.example.com;
    root /var/www/example;
    ...
    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = example.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
    listen 80;
    server_name example.com www.example.com;
    return 404; # managed by Certbot
}
```

### certbot-এর গুরুত্বপূর্ণ কমান্ড ও অপশন

| কমান্ড / অপশন | কাজ |
|---------------|-----|
| `certbot --nginx -d DOMAIN` | nginx-এর জন্য সার্টিফিকেট নেওয়া ও কনফিগ বসানো |
| `certbot --apache -d DOMAIN` | Apache-এর জন্য একই |
| `certbot certonly --nginx -d DOMAIN` | শুধু সার্টিফিকেট আনো, কনফিগে হাত দিও না |
| `certbot certonly --webroot -w /var/www/example -d DOMAIN` | চলমান সার্ভারের ওয়েব রুটে ফাইল রেখে যাচাই |
| `certbot certonly --standalone -d DOMAIN` | certbot নিজেই পোর্ট 80-তে সাময়িক সার্ভার চালায় (ওয়েব সার্ভার বন্ধ থাকতে হবে) |
| `certbot certonly --manual --preferred-challenges dns -d '*.example.com'` | DNS TXT রেকর্ড দিয়ে যাচাই; wildcard-এর জন্য DNS যাচাই বাধ্যতামূলক |
| `certbot certificates` | সব সার্টিফিকেট আর মেয়াদ দেখাও |
| `certbot renew` | যেগুলোর মেয়াদ শেষের কাছাকাছি সেগুলো নবায়ন |
| `certbot renew --dry-run` | পরীক্ষামূলক নবায়ন (staging সার্ভারে, আসল কিছু বদলায় না) |
| `certbot delete --cert-name DOMAIN` | সার্টিফিকেট মুছে ফেলা |
| `--non-interactive --agree-tos -m EMAIL` | স্ক্রিপ্টে প্রশ্ন ছাড়া চালানো |
| `--redirect` / `--no-redirect` | HTTP→HTTPS রিডাইরেক্ট বসাবে কিনা |
| `--test-cert` (বা `--staging`) | পরীক্ষার জন্য staging CA (rate limit এড়াতে) |

### অটো-রিনিউ কীভাবে চলে

certbot ইনস্টলের সাথে একটা systemd timer ([অধ্যায় ২৯](29-scheduling-cron.md)) বা cron job আসে, যেটা দিনে দুবার `certbot renew` চালায়। মেয়াদ শেষ হওয়ার কাছাকাছি (সাধারণত ৩০ দিনের কম) থাকলে তবেই নবায়ন হয়।

```
$ systemctl list-timers | grep -i certbot
Thu 2026-11-12 03:41:00 +06  6h left  ...  snap.certbot.renew.timer   snap.certbot.renew.service
# apt/dnf প্যাকেজে timer-এর নাম certbot.timer (বা certbot-renew.timer)

$ sudo certbot renew --dry-run
...
Congratulations, all simulated renewals succeeded:
  /etc/letsencrypt/live/example.com/fullchain.pem (success)
```

`--nginx` plugin দিয়ে নেওয়া সার্টিফিকেট নবায়নের পর certbot নিজেই nginx reload করে। `certonly`/`webroot` দিয়ে নিলে reload করাতে hook লাগে:

```bash
sudo certbot renew --deploy-hook "systemctl reload nginx"
# স্থায়ীভাবে: /etc/letsencrypt/renewal-hooks/deploy/ ফোল্ডারে একটা executable স্ক্রিপ্ট রাখো
```

> ⚠️ **সতর্কতা:** Let's Encrypt-এর **rate limit** আছে (একই ডোমেইন-সেটের জন্য সপ্তাহে সীমিত সংখ্যক সার্টিফিকেট, বারবার ব্যর্থ যাচাইয়েরও সীমা)। শেখার সময় বা পরীক্ষায় `--test-cert`/`--dry-run` ব্যবহার করো, নাহলে কয়েক ঘণ্টা বা দিনের জন্য আটকে যেতে পারো।

> 📝 **মনে রাখো:** `/etc/letsencrypt/live/` ফোল্ডারের ফাইলগুলো আসলে `../archive/`-এর সর্বশেষ সংস্করণের symlink। কনফিগে সবসময় `live/` পথ ব্যবহার করো, তাহলে নবায়নের পর কিছু বদলাতে হবে না। `privkey.pem` শুধু root পড়তে পারে, এটা এমনই থাকা উচিত।

যাচাই:

```
$ curl -I https://example.com
HTTP/2 200
server: nginx/1.24.0 (Ubuntu)
...
$ echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -dates -issuer
notBefore=Nov 11 03:12:00 2026 GMT
notAfter=Feb  9 03:11:59 2027 GMT
issuer=C = US, O = Let's Encrypt, CN = R12
```

(issuer-এর `CN` মান Let's Encrypt-এর কোন intermediate সই করেছে তার উপর নির্ভর করে আলাদা হতে পারে।)

HTTPS চালু হলে চাইলে HSTS header যোগ করো, যাতে ব্রাউজার এরপর থেকে সবসময় HTTPS-এই আসে:

```nginx
add_header Strict-Transport-Security "max-age=31536000" always;
```

> ⚠️ **সতর্কতা:** HSTS দেওয়ার পর HTTPS নষ্ট হলে দর্শকরা `max-age` শেষ না হওয়া পর্যন্ত HTTP-তেও ঢুকতে পারবে না। প্রথমে ছোট `max-age` (যেমন 300) দিয়ে পরীক্ষা করো।

---

## ৪০.৭ ডেটাবেস সার্ভার: MySQL / MariaDB

**MySQL** সবচেয়ে জনপ্রিয় ওপেন-সোর্স রিলেশনাল ডেটাবেস (এখন Oracle-এর মালিকানায়)। **MariaDB** হলো MySQL-এর মূল নির্মাতাদের তৈরি একটা fork; কমান্ড ও প্রোটোকল প্রায় একই, তাই বেশিরভাগ অ্যাপ দুটোতেই চলে। Debian 12 আর RHEL/Fedora-তে ডিফল্ট "MySQL" প্যাকেজ আসলে MariaDB; Ubuntu দুটোই দেয়।

### ডেটাবেস সার্ভারের গঠন

```
 অ্যাপ (PHP/Node)        mysql ক্লায়েন্ট (তুমি)
       │ TCP 127.0.0.1:3306      │ Unix socket /run/mysqld/mysqld.sock
       └──────────┬──────────────┘
                  v
        ┌──────────────────────────┐
        │ mysqld / mariadbd        │  একটা process, অনেক thread
        │  ├─ কানেকশন ও যাচাই      │  (ইউজার@হোস্ট + পাসওয়ার্ড/প্লাগইন)
        │  ├─ SQL parser/optimizer │
        │  └─ storage engine InnoDB│
        └────────────┬─────────────┘
                     v
           /var/lib/mysql/   (ডেটা ফাইল — এটাই আসল সম্পদ)
```

### ইনস্টল ও চালু

```bash
# Ubuntu 24.04 — MySQL 8.0
sudo apt install mysql-server            # service: mysql

# Ubuntu/Debian — MariaDB
sudo apt install mariadb-server          # service: mariadb

# Fedora / RHEL 9 — MariaDB
sudo dnf install mariadb-server
sudo systemctl enable --now mariadb

# RHEL 9 — MySQL
sudo dnf install mysql-server
sudo systemctl enable --now mysqld       # লক্ষ করো: service-এর নাম mysqld
```

```
$ sudo systemctl status mysql
● mysql.service - MySQL Community Server
     Active: active (running) ...
$ sudo ss -tlnp | grep 3306
LISTEN 0  151  127.0.0.1:3306  0.0.0.0:*  users:(("mysqld",pid=4410,fd=23))
```

লক্ষ করো, Ubuntu-তে MySQL ডিফল্টে শুধু `127.0.0.1`-এ listen করে। মানে বাইরে থেকে কেউ ঢুকতে পারবে না। এটাই ভালো।

### নিরাপদ করা: mysql_secure_installation

```bash
sudo mysql_secure_installation       # MySQL
sudo mariadb-secure-installation     # MariaDB (পুরোনো নাম mysql_secure_installation-ও কাজ করে)
```

এই স্ক্রিপ্ট প্রশ্ন করে করে যা করে:

| প্রশ্ন | উত্তর | কারণ |
|-------|-------|------|
| VALIDATE PASSWORD component (MySQL) | ঐচ্ছিক | দুর্বল পাসওয়ার্ড আটকায় |
| Remove anonymous users? | Y | নাম ছাড়া লগইন বন্ধ |
| Disallow root login remotely? | Y | root শুধু লোকাল থেকে |
| Remove test database? | Y | সবার জন্য খোলা `test` ডেটাবেস মুছে ফেলো |
| Reload privilege tables? | Y | পরিবর্তন কার্যকর |

> 📝 **মনে রাখো:** Ubuntu-র MySQL আর Debian/Ubuntu-র MariaDB-তে ডেটাবেসের root ইউজার পাসওয়ার্ড দিয়ে নয়, **unix socket authentication** দিয়ে লগইন করে: লিনাক্সের root (sudo) হলে ডেটাবেসের root-এ সরাসরি ঢোকা যায়। তাই `sudo mysql` লিখলেই হয়, পাসওয়ার্ড লাগে না। এটা নিরাপদ, বদলানোর দরকার নেই; অ্যাপের জন্য আলাদা ইউজার বানাও।

### ডেটাবেস ও অ্যাপ ইউজার তৈরি

```
$ sudo mysql
mysql> CREATE DATABASE shopdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
Query OK, 1 row affected (0.01 sec)

mysql> CREATE USER 'shopapp'@'localhost' IDENTIFIED BY 'Str0ng-Pass!2026';
Query OK, 0 rows affected (0.02 sec)

mysql> GRANT ALL PRIVILEGES ON shopdb.* TO 'shopapp'@'localhost';
Query OK, 0 rows affected (0.01 sec)

mysql> SHOW GRANTS FOR 'shopapp'@'localhost';
+------------------------------------------------------------------+
| Grants for shopapp@localhost                                     |
+------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `shopapp`@`localhost`                      |
| GRANT ALL PRIVILEGES ON `shopdb`.* TO `shopapp`@`localhost`      |
+------------------------------------------------------------------+

mysql> exit
Bye
```

- `'shopapp'@'localhost'`: MySQL-এ ইউজার মানে **নাম + কোথা থেকে আসছে** জোড়া। `'shopapp'@'%'` মানে যেকোনো হোস্ট থেকে, যেটা দরকার না হলে দেবে না।
- `utf8mb4`: বাংলা আর ইমোজি সহ সব Unicode অক্ষর ঠিকভাবে রাখতে। (পুরোনো `utf8` আসলে ৩-বাইট সীমিত।)
- `GRANT` দিয়ে দেওয়া অধিকার সাথে সাথে কার্যকর হয়; `FLUSH PRIVILEGES` শুধু তখন লাগে যখন কেউ সরাসরি `mysql.user` টেবিল এডিট করে।
- **least privilege নীতি:** অ্যাপকে শুধু তার নিজের ডেটাবেসের অধিকার দাও, কখনো `*.*` নয়।

নতুন ইউজার দিয়ে পরীক্ষা:

```
$ mysql -u shopapp -p shopdb
Enter password:
mysql> SELECT CURRENT_USER(), DATABASE();
+-------------------+------------+
| CURRENT_USER()    | DATABASE() |
+-------------------+------------+
| shopapp@localhost | shopdb     |
+-------------------+------------+
```

### mysql ক্লায়েন্টের দরকারি কমান্ড

| কমান্ড | কাজ |
|--------|-----|
| `SHOW DATABASES;` | সব ডেটাবেস |
| `USE shopdb;` | ডেটাবেস বাছাই |
| `SHOW TABLES;` | টেবিলের তালিকা |
| `DESCRIBE orders;` | টেবিলের কলাম |
| `SELECT user, host FROM mysql.user;` | সব ইউজার |
| `SHOW PROCESSLIST;` | এখন কী query চলছে |
| `SELECT ... \G` | ফলাফল লম্বালম্বি (চওড়া টেবিল পড়তে সুবিধা) |
| `DROP USER 'x'@'localhost';` | ইউজার মুছে ফেলা |
| `ALTER USER 'x'@'localhost' IDENTIFIED BY 'নতুন';` | পাসওয়ার্ড বদল |
| `exit` / `\q` / `Ctrl+D` | বের হওয়া |

শেল থেকে ব্যাকআপ ও রিস্টোর ([অধ্যায় ৩৫](35-backup-rsync.md)):

```bash
mysqldump -u shopapp -p --single-transaction shopdb > shopdb-$(date +%F).sql
mysql -u shopapp -p shopdb < shopdb-2026-11-11.sql
```

### কনফিগ ও দূর থেকে কানেকশন

| | Ubuntu MySQL | Debian/Ubuntu MariaDB | RHEL/Fedora MariaDB |
|-|--------------|----------------------|---------------------|
| কনফিগ | `/etc/mysql/mysql.conf.d/mysqld.cnf` | `/etc/mysql/mariadb.conf.d/50-server.cnf` | `/etc/my.cnf.d/mariadb-server.cnf` |
| ডেটা | `/var/lib/mysql` | `/var/lib/mysql` | `/var/lib/mysql` |
| এরর লগ | `/var/log/mysql/error.log` | journald / `/var/log/mysql/` | `/var/log/mariadb/mariadb.log` |

অন্য সার্ভার থেকে ঢুকতে হলে `bind-address = 127.0.0.1`-কে নির্দিষ্ট প্রাইভেট IP-তে বদলাও, `'user'@'10.0.0.%'` ধরনের ইউজার বানাও, আর ফায়ারওয়ালে শুধু সেই IP-কে 3306 খুলে দাও:

```bash
sudo ufw allow from 10.0.0.20 to any port 3306 proto tcp
```

> ⚠️ **সতর্কতা:** ডেটাবেস পোর্ট (3306, 5432) কখনো পুরো ইন্টারনেটে খুলবে না। ইন্টারনেটে বট সারাক্ষণ এই পোর্টগুলো স্ক্যান করে। দূর থেকে নিজে কাজ করতে চাইলে SSH tunnel ব্যবহার করো ([অধ্যায় ৩৮](38-ssh-remote.md)): `ssh -L 3306:127.0.0.1:3306 deploy@server`।

---

## ৪০.৮ ডেটাবেস সার্ভার: PostgreSQL

**PostgreSQL** ("Postgres") আরেকটা শক্তিশালী ওপেন-সোর্স রিলেশনাল ডেটাবেস। SQL মান মেনে চলায়, জটিল query, JSON, এক্সটেনশন (যেমন PostGIS) ইত্যাদিতে এটা বিখ্যাত। Django, Rails এবং আধুনিক অনেক প্রজেক্টে এটাই প্রথম পছন্দ।

### ইনস্টল

```bash
# Ubuntu 24.04 (PostgreSQL 16) / Debian 12 (PostgreSQL 15)
sudo apt install postgresql
# ইনস্টলের সাথেই একটা "cluster" তৈরি হয়ে চালু হয়ে যায়

# Fedora / RHEL 9
sudo dnf install postgresql-server
sudo postgresql-setup --initdb          # প্রথমবার ডেটা ডিরেক্টরি তৈরি — RHEL-এ বাধ্যতামূলক
sudo systemctl enable --now postgresql
```

```
$ sudo ss -tlnp | grep 5432
LISTEN 0  200  127.0.0.1:5432  0.0.0.0:*  users:(("postgres",pid=5120,fd=6))
```

Ubuntu/Debian-এ একাধিক সংস্করণ পাশাপাশি চালানো যায়; প্রতিটা হলো একটা **cluster**। তালিকা দেখো:

```
$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

### PostgreSQL-এর ভেতরের গঠন

MySQL-এর মতো একটা process-এ অনেক thread নয়; PostgreSQL **প্রতি কানেকশনে একটা আলাদা process** বানায়:

```
postgres (postmaster / main)  ── কানেকশন গ্রহণ করে
  ├── checkpointer, background writer, walwriter, autovacuum ... (রক্ষণাবেক্ষণ)
  ├── postgres: shopapp shopdb 127.0.0.1(51234) idle      <- ক্লায়েন্ট ১
  └── postgres: shopapp shopdb 127.0.0.1(51240) SELECT    <- ক্লায়েন্ট ২
```

`ps aux | grep postgres` চালালেই এগুলো দেখতে পাবে ([অধ্যায় ২৪](24-process-management.md))। এজন্যই খুব বেশি কানেকশনের অ্যাপে PgBouncer-এর মতো connection pooler ব্যবহার হয়।

### role আর লগইন

PostgreSQL-এ ইউজার আর গ্রুপ দুটোই **role**। ইনস্টলের সময় `postgres` নামে একটা লিনাক্স ইউজার আর একই নামের superuser role তৈরি হয়। ডিফল্ট লোকাল লগইন হয় **peer authentication** দিয়ে: লিনাক্স ইউজারের নাম আর ডেটাবেস role-এর নাম মিললে ঢুকতে দেয়।

```
$ sudo -u postgres psql
psql (16.4 (Ubuntu 16.4-0ubuntu0.24.04.2))
Type "help" for help.

postgres=# CREATE ROLE shopapp WITH LOGIN PASSWORD 'Str0ng-Pass!2026';
CREATE ROLE
postgres=# CREATE DATABASE shopdb OWNER shopapp;
CREATE DATABASE
postgres=# \l shopdb
                                    List of databases
  Name  |  Owner  | Encoding | ...
--------+---------+----------+----
 shopdb | shopapp | UTF8     | ...
postgres=# \q
```

শেল থেকেও একই কাজ করা যায়:

```bash
sudo -u postgres createuser --pwprompt shopapp
sudo -u postgres createdb -O shopapp shopdb
```

পাসওয়ার্ড দিয়ে TCP-তে পরীক্ষা (লক্ষ করো `-h 127.0.0.1`; এটা না দিলে socket আর peer auth চেষ্টা হবে):

```
$ psql -h 127.0.0.1 -U shopapp -d shopdb
Password for user shopapp:
shopdb=> SELECT current_user, version();
```

> 💡 **টিপস:** PostgreSQL 15 থেকে `public` schema-তে সাধারণ ইউজার ডিফল্টে টেবিল বানাতে পারে না। ডেটাবেসের `OWNER` অ্যাপ ইউজারকে বানালে (যেমন উপরে) এই সমস্যা হয় না।

### psql-এর meta-command

| কমান্ড | কাজ |
|--------|-----|
| `\l` | সব ডেটাবেস |
| `\c shopdb` | অন্য ডেটাবেসে যাও |
| `\dt` | টেবিলের তালিকা |
| `\d orders` | টেবিলের গঠন |
| `\du` | সব role |
| `\dn` | schema-র তালিকা |
| `\x` | লম্বালম্বি আউটপুট চালু/বন্ধ (MySQL-এর `\G`-এর মতো) |
| `\timing` | প্রতিটি query-র সময় দেখাও |
| `\e` | বর্তমান query `$EDITOR`-এ খোলো |
| `\i file.sql` | ফাইল থেকে SQL চালাও |
| `\?` / `\h` | meta-command / SQL সাহায্য |
| `\q` বা `Ctrl+D` | বের হও |

### কনফিগ: postgresql.conf ও pg_hba.conf

| | Ubuntu/Debian | RHEL/Fedora |
|-|---------------|-------------|
| কনফিগ ফোল্ডার | `/etc/postgresql/16/main/` | `/var/lib/pgsql/data/` |
| ডেটা | `/var/lib/postgresql/16/main` | `/var/lib/pgsql/data` |
| লগ | `/var/log/postgresql/` | `/var/lib/pgsql/data/log/` |

- **`postgresql.conf`**: সার্ভারের সেটিং। `listen_addresses = 'localhost'` (ডিফল্ট) মানে শুধু লোকাল। দূর থেকে চাইলে নির্দিষ্ট IP দাও।
- **`pg_hba.conf`** (host-based authentication): কে, কোন ডেটাবেসে, কোথা থেকে, কোন পদ্ধতিতে ঢুকতে পারবে। ওপর থেকে নিচে প্রথম মেলা লাইনটাই খাটে।

```
# TYPE  DATABASE  USER     ADDRESS         METHOD
local   all       postgres                 peer
local   all       all                      peer
host    all       all      127.0.0.1/32    scram-sha-256
host    all       all      ::1/128         scram-sha-256
host    shopdb    shopapp  10.0.0.20/32    scram-sha-256   # দূরের অ্যাপ সার্ভার
```

বদলানোর পর `sudo systemctl reload postgresql` (`listen_addresses` বদলালে restart লাগে)।

> ⚠️ **সতর্কতা:** `pg_hba.conf`-এ কখনো `trust` মেথড দূরের ঠিকানার জন্য দেবে না; `trust` মানে পাসওয়ার্ড ছাড়াই যে কেউ ঢুকতে পারবে।

ব্যাকআপ:

```bash
pg_dump -h 127.0.0.1 -U shopapp -Fc shopdb > shopdb.dump     # custom format
pg_restore -h 127.0.0.1 -U shopapp -d shopdb --clean shopdb.dump
```

### MySQL বনাম PostgreSQL এক নজরে

| বিষয় | MySQL / MariaDB | PostgreSQL |
|-------|-----------------|------------|
| পোর্ট | 3306 | 5432 |
| অ্যাডমিন হিসেবে ঢোকা | `sudo mysql` | `sudo -u postgres psql` |
| ক্লায়েন্ট | `mysql` (MariaDB-তে `mariadb`ও) | `psql` |
| ডিফল্ট লোকাল auth | unix socket (Ubuntu/Debian) | peer |
| process মডেল | এক process, অনেক thread | প্রতি কানেকশনে এক process |
| ব্যাকআপ টুল | `mysqldump` | `pg_dump`, `pg_dumpall` |
| প্রচলিত সঙ্গী | PHP, WordPress, Laravel | Django, Rails, অ্যানালিটিক্স |

---

## ৪০.৯ PHP-FPM — nginx-এর সাথে PHP চালানো

nginx নিজে PHP বোঝে না; সে শুধু ফাইল দিতে আর অন্যকে request পাঠাতে পারে। PHP চালায় **PHP-FPM** (FastCGI Process Manager): একটা আলাদা service, যেটা কয়েকটা PHP worker process প্রস্তুত রাখে আর **FastCGI** প্রোটোকলে nginx-এর কাছ থেকে কাজ নেয়।

```
nginx                     php-fpm (master, root)
  │  fastcgi_pass            ├── pool [www]  (ইউজার www-data)
  │  unix socket  ─────────> │     ├── worker 1  (index.php চালাচ্ছে)
  │  /run/php/php8.3-fpm.sock│     ├── worker 2  (idle)
  │                          │     └── worker 3  (idle)
  │ <──── HTML output ────── │
```

### ইনস্টল

```bash
# Ubuntu 24.04 (PHP 8.3) — Debian 12-এ PHP 8.2, তখন নামে 8.2 বসাও
sudo apt install php-fpm php-mysql php-pgsql php-curl php-mbstring php-xml php-zip
systemctl status php8.3-fpm

# Fedora / RHEL 9
sudo dnf install php-fpm php-mysqlnd php-pgsql php-mbstring php-xml
sudo systemctl enable --now php-fpm
```

| | Ubuntu/Debian | RHEL/Fedora |
|-|---------------|-------------|
| service | `php8.3-fpm` | `php-fpm` |
| pool কনফিগ | `/etc/php/8.3/fpm/pool.d/www.conf` | `/etc/php-fpm.d/www.conf` |
| php.ini | `/etc/php/8.3/fpm/php.ini` | `/etc/php.ini` |
| socket | `/run/php/php8.3-fpm.sock` | `/run/php-fpm/www.sock` |
| লগ | `/var/log/php8.3-fpm.log` | `/var/log/php-fpm/` |

> 📝 **মনে রাখো:** RHEL/Fedora-তে PHP-FPM pool-এর ডিফল্ট ইউজার `apache`। nginx-এর সাথে চালালে `www.conf`-এ `user`/`group` আর socket-এর `listen.acl_users`-এ nginx ইউজার আছে কিনা মিলিয়ে নাও, নাহলে socket-এ permission denied (502) পাবে।

### nginx-এ PHP সংযোগ (Ubuntu)

```nginx
server {
    listen 80;
    server_name blog.example.com;
    root /var/www/blog;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;              # SCRIPT_FILENAME ইত্যাদি সেট করে
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

পরীক্ষা: `/var/www/blog/info.php` ফাইলে `<?php phpinfo();` লিখে ব্রাউজারে খোলো। কাজ করলে **সাথে সাথে ফাইলটা মুছে ফেলো**, কারণ এটা সার্ভারের অনেক গোপন তথ্য দেখায়।

### pool টিউনিং

`www.conf`-এর গুরুত্বপূর্ণ লাইন:

```ini
pm = dynamic              ; static | dynamic | ondemand
pm.max_children = 5       ; একসাথে সর্বোচ্চ কতগুলো PHP worker
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

`pm.max_children` আন্দাজ করার সূত্র: (PHP-এর জন্য বরাদ্দ RAM) ÷ (একটা worker গড়ে কত MB খায়)। খুব বেশি দিলে RAM শেষ হয়ে swap/OOM, খুব কম দিলে request লাইনে দাঁড়িয়ে থাকে আর লগে `server reached pm.max_children setting` দেখায়।

বদলানোর পর:

```bash
sudo php-fpm8.3 -t            # কনফিগ পরীক্ষা (RHEL: sudo php-fpm -t)
sudo systemctl reload php8.3-fpm
```

একাধিক সাইট থাকলে প্রতিটি সাইটের জন্য আলাদা pool (আলাদা ইউজার ও socket) বানানো নিরাপদ: এক সাইট হ্যাক হলে অন্যটার ফাইল ছুঁতে পারবে না।

---

## ৪০.১০ Node.js অ্যাপকে systemd দিয়ে চালানো

টার্মিনালে `node server.js` চালিয়ে SSH বন্ধ করলে অ্যাপও মরে যায় (কারণ জানতে [অধ্যায় ২৫](25-jobs-tmux.md))। `nohup` বা `tmux` কাজ চালানোর উপায়, কিন্তু প্রোডাকশনে ঠিক উপায় হলো **systemd service**: বুটে নিজে চালু, crash হলে নিজে রিস্টার্ট, লগ journald-এ জমা। unit ফাইলের পূর্ণ ব্যাখ্যা [অধ্যায় ২৬](26-systemd-services.md)-এ আছে; এখানে বাস্তব রেসিপি।

### ধাপ ১: অ্যাপের জন্য আলাদা ইউজার

```bash
sudo useradd --system --create-home --home-dir /srv/myapp --shell /usr/sbin/nologin myapp
```

`--system` আর `nologin` মানে এই ইউজার দিয়ে কেউ লগইন করতে পারবে না; শুধু অ্যাপ চালানোর জন্য।

### ধাপ ২: Node ও অ্যাপ

```bash
# Ubuntu-র রিপোজিটরির nodejs পুরোনো হতে পারে; প্রোডাকশনে প্রায়ই NodeSource বা nvm থেকে LTS নেওয়া হয়
sudo apt install nodejs npm
node --version
```

`/srv/myapp/server.js` (ছোট উদাহরণ):

```javascript
const http = require('http');
const port = process.env.PORT || 3000;
http.createServer((req, res) => {
  res.end('Hello from Node, ip=' + (req.headers['x-real-ip'] || 'unknown') + '\n');
}).listen(port, '127.0.0.1', () => console.log(`listening on ${port}`));
```

লক্ষ করো `'127.0.0.1'`: অ্যাপ শুধু লোকালে শোনে, বাইরে থেকে পৌঁছানো যায় শুধু nginx দিয়ে।

### ধাপ ৩: environment ফাইল (গোপন তথ্য)

`/etc/myapp.env`:

```
NODE_ENV=production
PORT=3000
DATABASE_URL=postgres://shopapp:Str0ng-Pass!2026@127.0.0.1:5432/shopdb
```

```bash
sudo chown root:myapp /etc/myapp.env
sudo chmod 640 /etc/myapp.env      # অন্যরা পাসওয়ার্ড পড়তে পারবে না
```

### ধাপ ৪: unit ফাইল

`/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My Node.js app
After=network.target
# ডেটাবেস একই সার্ভারে থাকলে: After=network.target postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/srv/myapp
EnvironmentFile=/etc/myapp.env
ExecStart=/usr/bin/node /srv/myapp/server.js
Restart=on-failure
RestartSec=5

# অতিরিক্ত নিরাপত্তা (ঐচ্ছিক কিন্তু ভালো)
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full

[Install]
WantedBy=multi-user.target
```

| লাইন | মানে |
|------|------|
| `After=network.target` | নেটওয়ার্ক প্রস্তুত হওয়ার পরে শুরু |
| `User=`/`Group=` | কোন ইউজার হিসেবে চলবে (root নয়!) |
| `WorkingDirectory=` | অ্যাপের কাজের ফোল্ডার |
| `EnvironmentFile=` | ফাইল থেকে environment ভ্যারিয়েবল |
| `ExecStart=` | পুরো পথসহ চালানোর কমান্ড |
| `Restart=on-failure` | crash (non-zero exit) হলে আবার চালু |
| `RestartSec=5` | রিস্টার্টের আগে ৫ সেকেন্ড অপেক্ষা |
| `NoNewPrivileges=true` | process নিজের ক্ষমতা বাড়াতে পারবে না |
| `ProtectSystem=full` | `/usr`, `/boot`, `/etc` read-only |
| `WantedBy=multi-user.target` | `enable` করলে বুটে চালু হবে |

> ⚠️ **সতর্কতা:** `ExecStart`-এ সবসময় **পূর্ণ পথ** দাও (`which node` দিয়ে দেখে নাও)। nvm দিয়ে ইনস্টল করা node ইউজারের home-এ থাকে (`~/.nvm/versions/...`), systemd সেটা নিজে খুঁজে পায় না।

### ধাপ ৫: চালু করো

```
$ sudo systemctl daemon-reload
$ sudo systemctl enable --now myapp
Created symlink /etc/systemd/system/multi-user.target.wants/myapp.service → /etc/systemd/system/myapp.service.
$ systemctl status myapp
● myapp.service - My Node.js app
     Loaded: loaded (/etc/systemd/system/myapp.service; enabled; preset: enabled)
     Active: active (running) since ...
   Main PID: 6021 (node)
$ curl http://127.0.0.1:3000
Hello from Node, ip=unknown
```

এখন ৪০.৪-এর reverse proxy কনফিগ দিয়ে nginx-এর পেছনে বসাও, তারপর certbot দিয়ে HTTPS। ব্যস, প্রোডাকশন-মানের ডিপ্লয়মেন্ট।

কোড আপডেটের পর:

```bash
sudo systemctl restart myapp
journalctl -u myapp -f        # লাইভ লগ দেখো
```

> 💡 **টিপস:** একই পদ্ধতি Python (`ExecStart=/srv/app/venv/bin/gunicorn -b 127.0.0.1:8000 app:app`), Go বা Java অ্যাপের জন্যও খাটে, শুধু `ExecStart` বদলাবে। Node জগতে PM2 নামে একটা জনপ্রিয় process manager আছে; সেটাও শেষ পর্যন্ত systemd দিয়েই বুটে চালু করা হয় (`pm2 startup`)। একটা সার্ভারে একটা-দুটো অ্যাপ হলে সরাসরি systemd সবচেয়ে সরল।

---

## ৪০.১১ লগ দেখা — সমস্যার প্রথম সূত্র

লগ সম্পর্কে সাধারণ জ্ঞান আর `journalctl`-এর পুরো ব্যাখ্যা [অধ্যায় ২৮](28-logs.md)-এ। এখানে সার্ভিস-নির্দিষ্ট লগের মানচিত্র:

| সার্ভিস | লগ ফাইল | journald |
|---------|---------|----------|
| nginx | `/var/log/nginx/access.log`, `error.log` | `journalctl -u nginx` (শুধু start/stop বার্তা) |
| Apache (Debian) | `/var/log/apache2/access.log`, `error.log` | `journalctl -u apache2` |
| Apache (RHEL) | `/var/log/httpd/access_log`, `error_log` | `journalctl -u httpd` |
| MySQL (Ubuntu) | `/var/log/mysql/error.log` | `journalctl -u mysql` |
| MariaDB | distro-ভেদে ফাইল বা journald | `journalctl -u mariadb` |
| PostgreSQL | `/var/log/postgresql/postgresql-16-main.log` (Ubuntu) | `journalctl -u postgresql@16-main` |
| PHP-FPM | `/var/log/php8.3-fpm.log` | `journalctl -u php8.3-fpm` |
| তোমার Node অ্যাপ | (stdout/stderr) | `journalctl -u myapp` |
| certbot | `/var/log/letsencrypt/letsencrypt.log` | — |

### access log পড়া

```
$ sudo tail -n 2 /var/log/nginx/access.log
203.0.113.77 - - [11/Nov/2026:10:31:02 +0600] "GET /products HTTP/1.1" 200 5123 "https://google.com/" "Mozilla/5.0 ..."
198.51.100.9 - - [11/Nov/2026:10:31:05 +0600] "GET /.env HTTP/1.1" 404 162 "-" "curl/8.5.0"
```

এটা **combined** ফরম্যাট:

```
203.0.113.77  -  -  [সময়]  "GET /products HTTP/1.1"  200  5123  "referer"  "user-agent"
   │                          │                       │     │
 ক্লায়েন্ট IP                method path protocol    status bytes
```

দ্বিতীয় লাইনটা খেয়াল করো: কেউ `.env` ফাইল খুঁজছে। এটা প্রতিটি পাবলিক সার্ভারে প্রতিদিন হয়; তাই লুকানো ফাইলে `deny all` দেওয়া জরুরি।

status code মনে রাখার সহজ নিয়ম: **2xx** সফল, **3xx** রিডাইরেক্ট, **4xx** ক্লায়েন্টের ভুল (404 নেই, 403 নিষেধ, 413 ফাইল বড়), **5xx** সার্ভারের ভুল (500 অ্যাপ ভেঙেছে, 502 পেছনের অ্যাপ সাড়া দেয়নি, 504 সময় শেষ)।

### বাস্তব লগ-বিশ্লেষণ এক-লাইনার

[অধ্যায় ২২](22-awk.md) আর [অধ্যায় ২৩](23-text-tools.md)-এর টুল এখানে কাজে আসে:

```bash
# লাইভ দেখো
sudo tail -f /var/log/nginx/access.log /var/log/nginx/error.log

# সবচেয়ে বেশি request পাঠানো ১০টা IP
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# status code অনুযায়ী গণনা
sudo awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# শুধু 5xx এরর
sudo awk '$9 ~ /^5/' /var/log/nginx/access.log | tail

# সবচেয়ে বেশি চাওয়া path
sudo awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head

# আজকের অ্যাপ লগে "error" শব্দ
journalctl -u myapp --since today | grep -i error
```

পুরোনো লগ `logrotate` প্রতিদিন/সপ্তাহে ঘুরিয়ে `.1`, `.2.gz` নামে রাখে; gzip করা লগ পড়তে `zcat`/`zgrep`/`zless` ব্যবহার করো।

### সাধারণ সমস্যা ও সমাধানের মানচিত্র

| লক্ষণ | সম্ভাব্য কারণ | কোথায় দেখবে / কী করবে |
|-------|---------------|----------------------|
| ব্রাউজারে কানেক্টই হয় না | service বন্ধ, ফায়ারওয়াল, DNS ভুল | `systemctl status nginx`, `ss -tlnp`, `ufw status`, `dig` |
| nginx চালু হয় না | কনফিগ ভুল বা পোর্ট দখল | `nginx -t`, `journalctl -u nginx`, `ss -tlnp 'sport = :80'` |
| 403 Forbidden | ফাইল/ফোল্ডারে পারমিশন নেই, index ফাইল নেই, SELinux | error.log; `namei -l /var/www/site/index.html` |
| 404 Not Found | `root` ভুল, `try_files` ভুল | error.log-এ nginx কোন পথ খুঁজেছে দেখো |
| 502 Bad Gateway | পেছনের অ্যাপ/PHP-FPM বন্ধ, ভুল পোর্ট/socket, SELinux | `systemctl status myapp`, `curl 127.0.0.1:3000`, error.log |
| 504 Gateway Timeout | অ্যাপ খুব ধীর (প্রায়ই ধীর DB query) | অ্যাপ লগ, `proxy_read_timeout` |
| 413 Request Entity Too Large | আপলোড বড় | `client_max_body_size`; PHP-তে `upload_max_filesize`, `post_max_size` |
| HTTPS সার্টিফিকেট মেয়াদোত্তীর্ণ | renew ব্যর্থ | `certbot renew --dry-run`, `/var/log/letsencrypt/` |
| অ্যাপ DB-তে ঢুকতে পারে না | ভুল পাসওয়ার্ড, ভুল host, `pg_hba.conf` | DB লগ, হাতে `mysql`/`psql` দিয়ে একই তথ্যে চেষ্টা |

`namei -l` একটা চমৎকার কমান্ড: পথের প্রতিটি ধাপের পারমিশন দেখায়, যাতে বোঝা যায় কোন ফোল্ডারে worker ইউজার আটকে যাচ্ছে ([অধ্যায় ১৬](16-permissions.md))।

```
$ namei -l /var/www/example/index.html
f: /var/www/example/index.html
drwxr-xr-x root   root     /
drwxr-xr-x root   root     var
drwxr-xr-x root   root     www
drwxr-x--- deploy deploy   example      <- www-data এখানে আটকে যাবে!
-rw-r--r-- deploy deploy   index.html
```

---

## ৪০.১২ LAMP ও LEMP স্ট্যাক

"স্ট্যাক" মানে একসাথে কাজ করা সফটওয়্যারের সেট। নামগুলো আদ্যক্ষর:

- **LAMP** = **L**inux + **A**pache + **M**ySQL/MariaDB + **P**HP (কখনো Perl/Python)
- **LEMP** = **L**inux + **E**nginx ("engine-x"-এর উচ্চারণ থেকে E) + **M**ySQL/MariaDB + **P**HP

### LAMP — mod_php সহ

```
                ┌──────────────────────── Linux সার্ভার ─────────────────────────┐
 ব্রাউজার       │                                                                 │
   │  :80/:443  │  ┌──────────────────────────────┐                               │
   └──────────> │  │ Apache (httpd / apache2)      │                               │
                │  │  ├─ static ফাইল সরাসরি        │                               │
                │  │  └─ .php ->  mod_php          │  PHP ইন্টারপ্রেটার Apache-এর   │
                │  │             (Apache-এর ভেতরে)  │  প্রতিটি process-এর ভেতরেই    │
                │  └──────────────┬───────────────┘                               │
                │                 │ SQL (socket বা 127.0.0.1:3306)                │
                │                 v                                               │
                │  ┌──────────────────────────────┐                               │
                │  │ MySQL / MariaDB  -> /var/lib/mysql                            │
                │  └──────────────────────────────┘                               │
                └─────────────────────────────────────────────────────────────────┘
```

(আধুনিক LAMP-এ প্রায়ই mod_php-এর বদলে Apache-ও `proxy_fcgi` দিয়ে PHP-FPM ব্যবহার করে, event MPM-এর সাথে।)

### LEMP — PHP-FPM সহ

```
                ┌──────────────────────── Linux সার্ভার ─────────────────────────┐
 ব্রাউজার       │                                                                 │
   │  :443      │  ┌─────────────────────┐  FastCGI   ┌──────────────────────┐    │
   └──────────> │  │ nginx               │  (socket)  │ PHP-FPM              │    │
                │  │  ├─ TLS (certbot)   │ ─────────> │  pool www            │    │
                │  │  ├─ static ফাইল     │ <───────── │  worker × N          │    │
                │  │  └─ *.php ──────────┘            └──────────┬───────────┘    │
                │  └─────────────────────┘                       │ SQL            │
                │                                                v                │
                │                                   ┌──────────────────────┐      │
                │                                   │ MySQL / MariaDB      │      │
                │                                   └──────────────────────┘      │
                │   সবকিছু systemd দেখাশোনা করে: nginx, php8.3-fpm, mysql        │
                └─────────────────────────────────────────────────────────────────┘
```

### অন্য জনপ্রিয় স্ট্যাক

| স্ট্যাক | উপাদান | উদাহরণ |
|---------|--------|--------|
| LAMP | Linux, Apache, MySQL, PHP | পুরোনো WordPress, cPanel হোস্টিং |
| LEMP | Linux, nginx, MySQL, PHP | আধুনিক WordPress, Laravel |
| MERN/MEAN | MongoDB, Express, React/Angular, Node | JavaScript-সর্বস্ব অ্যাপ |
| Node + Postgres | nginx → Node (systemd) → PostgreSQL | API সার্ভার |
| Django | nginx → gunicorn (systemd) → PostgreSQL | Python ওয়েব অ্যাপ |

### একটা LEMP স্ট্যাক এক নিঃশ্বাসে (Ubuntu 24.04)

```bash
sudo apt update
sudo apt install nginx mysql-server php-fpm php-mysql
sudo mysql_secure_installation
sudo ufw allow 'Nginx Full'
# তারপর: ডেটাবেস + ইউজার (৪০.৭), server block + PHP location (৪০.৯), certbot (৪০.৬)
```

> 💡 **টিপস:** এখনকার অনেক টিম এই একই স্ট্যাক কন্টেইনারে চালায় (nginx, php-fpm, mysql আলাদা container), যা [অধ্যায় ৪৮](48-containers-virtualization.md)-এ দেখবে। কিন্তু container-এর ভেতরেও ঠিক এই কনফিগ আর ধারণাই কাজ করে, তাই এখানে যা শিখলে তা সরাসরি কাজে লাগবে।

---

## ৪০.১৩ সংশ্লিষ্ট কীবোর্ড শর্টকাট

সার্ভার সেটআপে সবচেয়ে বেশি সময় যায় লগ দেখা, ডেটাবেস শেল আর কনফিগ এডিটে। সেগুলোর শর্টকাট:

**`less` / `journalctl` (লগ পড়ার সময়)**

| কী | কাজ |
|----|-----|
| `Space` / `b` | এক পাতা নিচে / উপরে |
| `G` / `g` | শেষে / শুরুতে |
| `/শব্দ` তারপর `n` / `N` | খোঁজা, পরের / আগের মিল |
| `F` | `tail -f`-এর মতো লাইভ অনুসরণ; থামাতে `Ctrl+C` |
| `q` | বের হওয়া |

**`tail -f`, চলমান কমান্ড**

| কী | কাজ |
|----|-----|
| `Ctrl+C` | `tail -f` / `journalctl -f` থামানো |
| `Ctrl+S` / `Ctrl+Q` | টার্মিনাল আউটপুট সাময়িক থামানো / আবার চালু (দ্রুত চলা লগ ধরতে) |

**`mysql` ও `psql` শেল**

| কী | কাজ |
|----|-----|
| `Ctrl+D` | শেল থেকে বের হওয়া (মুহূর্তে) |
| `Ctrl+C` | mysql: চলমান query বাতিল (৫.৭+/৮); psql: বর্তমান query বাতিল |
| `↑` / `↓` | আগের কমান্ড |
| `Ctrl+R` | হিস্ট্রিতে খোঁজা (readline সাপোর্ট থাকলে) |
| `Tab` | টেবিল/কলামের নাম auto-complete |
| `\c` (mysql) | অর্ধেক লেখা statement বাতিল |
| `\G` (mysql) / `\x` (psql) | লম্বালম্বি আউটপুট |

**কনফিগ এডিট** — `nano`-র শর্টকাট [অধ্যায় ১৩](13-nano-editor.md)-এ আর `vim`-এর [অধ্যায় ১৪](14-vim.md)-এ। দ্রুত মনে রাখার জন্য: nano-তে `Ctrl+O` সেভ, `Ctrl+X` বের, `Ctrl+W` খোঁজা; vim-এ `:w` সেভ, `:q` বের, `/` খোঁজা।

**SSH সেশন** (সার্ভারে কাজের সময়): ঝুলে যাওয়া SSH সেশন থেকে বের হতে নতুন লাইনে `~.` চাপো ([অধ্যায় ৩৮](38-ssh-remote.md))।

---

## হাতে-কলমে অনুশীলন

একটা পরীক্ষামূলক Ubuntu 24.04 মেশিন নাও (VM, [অধ্যায় ৩](03-distro-o-installation.md), বা সস্তা VPS)। ডোমেইন না থাকলে ল্যাব ১–৪ আর ৬–৭ লোকালি করা যায়; ল্যাব ৫-এর জন্য আসল ডোমেইন লাগবে।

### ল্যাব ১: nginx static সাইট

1. `sudo apt install nginx` এবং `curl -I http://localhost` দিয়ে `200 OK` দেখো।
2. `/var/www/lab` বানিয়ে ভেতরে `index.html` রাখো।
3. `/etc/nginx/sites-available/lab` ফাইলে `server_name lab.local;` আর `root /var/www/lab;` দিয়ে server block লেখো।
4. symlink দাও, `sudo nginx -t`, `sudo systemctl reload nginx`।
5. `curl -H 'Host: lab.local' http://localhost` চালাও; নিজের HTML দেখা উচিত।
6. ইচ্ছে করে কনফিগে একটা `;` মুছে `nginx -t` চালাও, এরর বার্তা আর লাইন নম্বর পড়ো। ঠিক করো।

### ল্যাব ২: location ও try_files পরীক্ষা

1. `/var/www/lab/docs/a.html` বানাও।
2. `location = /hello { return 200 "exact\n"; }` আর `location /hello { return 200 "prefix\n"; }` যোগ করো।
3. `curl -H 'Host: lab.local' localhost/hello` আর `.../hello/world` চালিয়ে কোনটা কোন উত্তর দেয় ব্যাখ্যা করো।
4. `location /` -এ `try_files $uri $uri/ /index.html;` দাও; `.../no-such-page` চাইলে কী আসে দেখো।

### ল্যাব ৩: Node অ্যাপ systemd + reverse proxy

1. `myapp` system ইউজার বানাও, ৪০.১০-এর `server.js` রাখো।
2. unit ফাইল লেখো, `daemon-reload`, `enable --now`।
3. `sudo systemctl kill -s SIGKILL myapp` দিয়ে অ্যাপ "crash" করাও; ৫ সেকেন্ড পর `systemctl status myapp` দেখো, নতুন PID নিয়ে আবার চালু হয়েছে কিনা।
4. nginx-এ `app.local`-এর জন্য reverse proxy server block বানাও।
5. `curl -H 'Host: app.local' localhost` চালাও; আউটপুটে `ip=127.0.0.1` (X-Real-IP থেকে) দেখবে।
6. `sudo systemctl stop myapp` করে আবার curl করো: **502** পাবে। `/var/log/nginx/error.log`-এ `connect() failed (111: Connection refused)` খুঁজে বের করো। তারপর অ্যাপ আবার চালু করো।

### ল্যাব ৪: ডেটাবেস

1. `sudo apt install mysql-server`, `sudo mysql_secure_installation`।
2. `labdb` ডেটাবেস আর `labuser@localhost` ইউজার বানাও, শুধু `labdb.*`-এ অধিকার দাও।
3. `mysql -u labuser -p labdb`-এ ঢুকে একটা টেবিল বানাও, দুটো row ঢোকাও।
4. `mysqldump` দিয়ে ব্যাকআপ নাও, টেবিল `DROP` করো, ব্যাকআপ থেকে ফেরত আনো।
5. একই কাজ PostgreSQL-এ করো: `sudo apt install postgresql`, `sudo -u postgres createuser --pwprompt labuser`, `createdb -O labuser labdb`, `psql -h 127.0.0.1 -U labuser labdb`।

### ল্যাব ৫: HTTPS (আসল ডোমেইন লাগবে)

1. DNS-এ A রেকর্ড দাও, `dig +short` দিয়ে যাচাই।
2. `sudo certbot --nginx --test-cert -d তোমার.ডোমেইন` (staging) দিয়ে প্রথমে পরীক্ষা।
3. সফল হলে `--test-cert` ছাড়া আবার চালাও (প্রয়োজনে `--force-renewal`)।
4. `sudo certbot certificates` আর `sudo certbot renew --dry-run`।
5. `curl -I http://তোমার.ডোমেইন` দিয়ে 301 রিডাইরেক্ট দেখো।

### ল্যাব ৬: LEMP-এ PHP

1. `sudo apt install php-fpm php-mysql`।
2. `lab` server block-এ ৪০.৯-এর `location ~ \.php$` যোগ করো।
3. `/var/www/lab/db.php` ফাইলে PDO দিয়ে `labdb`-তে কানেক্ট করে row গুলো প্রিন্ট করো।
4. `curl -H 'Host: lab.local' localhost/db.php` চালাও।
5. `sudo systemctl stop php8.3-fpm` করে 502 দেখো, error.log-এ socket-এর এরর পড়ো, আবার চালু করো।

### ল্যাব ৭: লগ গোয়েন্দাগিরি

1. একটা লুপ দিয়ে কিছু ট্রাফিক বানাও: `for i in $(seq 50); do curl -s -o /dev/null -H 'Host: lab.local' localhost/; curl -s -o /dev/null -H 'Host: lab.local' localhost/nope; done`
2. ৪০.১১-এর awk কমান্ড দিয়ে status code-এর গণনা বের করো (200 আর 404 দুটোই ৫০ হওয়া উচিত)।
3. `journalctl -u myapp --since "10 min ago"` দিয়ে অ্যাপের রিস্টার্টের বার্তা খুঁজে বের করো।

---

## সারসংক্ষেপ

| কাজ | কমান্ড |
|-----|--------|
| nginx ইনস্টল | `sudo apt install nginx` / `sudo dnf install nginx` |
| nginx কনফিগ পরীক্ষা | `sudo nginx -t` |
| পুরো কনফিগ (include সহ) দেখা | `sudo nginx -T` |
| নতুন কনফিগ লোড | `sudo systemctl reload nginx` |
| সাইট চালু (Debian) | `sudo ln -s /etc/nginx/sites-available/X /etc/nginx/sites-enabled/` |
| Apache সাইট/মডিউল চালু | `sudo a2ensite X`, `sudo a2enmod rewrite` |
| Apache কনফিগ পরীক্ষা | `sudo apache2ctl configtest` / `sudo apachectl configtest` |
| HTTPS সার্টিফিকেট | `sudo certbot --nginx -d example.com` |
| সার্টিফিকেটের তালিকা | `sudo certbot certificates` |
| রিনিউ পরীক্ষা | `sudo certbot renew --dry-run` |
| MySQL/MariaDB ইনস্টল | `sudo apt install mysql-server` / `mariadb-server` |
| MySQL নিরাপদ করা | `sudo mysql_secure_installation` |
| MySQL অ্যাডমিন শেল | `sudo mysql` |
| MySQL ব্যাকআপ | `mysqldump -u U -p --single-transaction DB > f.sql` |
| PostgreSQL ইনস্টল | `sudo apt install postgresql` / `dnf install postgresql-server` + `postgresql-setup --initdb` |
| PostgreSQL অ্যাডমিন শেল | `sudo -u postgres psql` |
| Postgres ইউজার/ডেটাবেস | `createuser --pwprompt U`, `createdb -O U DB` |
| Postgres ব্যাকআপ | `pg_dump -Fc DB > f.dump` |
| PHP-FPM কনফিগ পরীক্ষা | `sudo php-fpm8.3 -t` |
| অ্যাপ service চালু | `sudo systemctl daemon-reload && sudo systemctl enable --now myapp` |
| অ্যাপের লাইভ লগ | `journalctl -u myapp -f` |
| ওয়েব লগ লাইভ | `sudo tail -f /var/log/nginx/access.log` |
| কে কোন পোর্টে | `sudo ss -tlnp` |
| HTTP header পরীক্ষা | `curl -I https://example.com` |
| নির্দিষ্ট Host পরীক্ষা | `curl -H 'Host: site' http://localhost` |
| পথের পারমিশন | `namei -l /path/to/file` |
| SELinux proxy অনুমতি (RHEL) | `sudo setsebool -P httpd_can_network_connect 1` |

| শর্টকাট | কাজ |
|---------|-----|
| `Ctrl+C` | `tail -f`/`journalctl -f` থামানো, psql/mysql-এ query বাতিল |
| `Ctrl+D` | mysql/psql থেকে বের হওয়া |
| `F` (less-এ) | লাইভ অনুসরণ |
| `/শব্দ`, `n` | less/journalctl-এ খোঁজা |
| `\G` / `\x` | mysql / psql-এ লম্বালম্বি আউটপুট |

---

## নিজেকে যাচাই করো

1. nginx-এর master আর worker process-এর কাজ কী? worker কেন root হিসেবে চলে না?
2. কনফিগ বদলানোর পর `restart`-এর বদলে `nginx -t` আর `reload` কেন ব্যবহার করা উচিত?
3. `location /api/ { proxy_pass http://127.0.0.1:3000/; }` থাকলে `/api/users` request অ্যাপের কাছে কোন path হিসেবে পৌঁছাবে?
4. `root` আর `alias`-এর পার্থক্য উদাহরণ দিয়ে বলো।
5. reverse proxy-তে `X-Forwarded-Proto` header না দিলে কী সমস্যা হতে পারে?
6. Let's Encrypt সার্টিফিকেটের মেয়াদ কত, আর নবায়ন কে করে? নবায়ন ঠিকমতো চলবে কিনা কীভাবে পরীক্ষা করবে?
7. Ubuntu-তে MySQL ইনস্টলের পর `sudo mysql` লিখলে পাসওয়ার্ড ছাড়াই root হিসেবে ঢোকা যায় কেন?
8. PostgreSQL-এ `pg_hba.conf`-এর কাজ কী? `peer` আর `scram-sha-256` মেথডের পার্থক্য কী?
9. nginx `502 Bad Gateway` দিচ্ছে। কোন তিনটা জিনিস প্রথমে পরীক্ষা করবে?
10. LAMP আর LEMP স্ট্যাকে PHP চালানোর পদ্ধতিতে মূল পার্থক্য কী?

<details><summary>উত্তর দেখো</summary>

1. **master** root হিসেবে চলে, কনফিগ পড়ে, 80/443-এর মতো privileged পোর্ট খোলে, worker তৈরি করে এবং reload/stop signal সামলায়। **worker** আসল request সামলায় event loop দিয়ে। worker কম ক্ষমতার ইউজার (`www-data`/`nginx`) হিসেবে চলে যাতে কোনো worker হ্যাক হলেও আক্রমণকারী root ক্ষমতা না পায় (least privilege)।

2. `nginx -t` আগে সিনট্যাক্স ও ফাইল পরীক্ষা করে, ফলে ভুল ধরা পড়ে সার্ভিস বন্ধ হওয়ার আগেই। `restart` পুরো process বন্ধ করে আবার চালায়: চলমান কানেকশন কাটে, আর কনফিগ ভুল হলে nginx আর উঠবেই না। `reload` নতুন worker দিয়ে ধীরে ধীরে বদলায়, কানেকশন কাটে না, আর ভুল কনফিগ পেলে পুরোনোটাতেই চলতে থাকে।

3. `/users`। `proxy_pass`-এ হোস্ট:পোর্টের পরে `/` (URI অংশ) আছে, তাই location-এর মিলে যাওয়া অংশ `/api/` সেটা দিয়ে প্রতিস্থাপিত হয়। শেষের `/` না থাকলে `/api/users`-ই যেত।

4. `root` হলে location-এর path যোগ হয়: `location /img/ { root /var/www/data; }` → `/img/a.png` = `/var/www/data/img/a.png`। `alias` হলে location-এর অংশ বদলে যায়: `location /img/ { alias /var/www/pics/; }` → `/img/a.png` = `/var/www/pics/a.png`।

5. অ্যাপ বুঝতে পারবে না ক্লায়েন্ট আসলে HTTPS দিয়ে এসেছে (nginx থেকে অ্যাপে কানেকশন সাধারণ HTTP)। ফলে অ্যাপ "HTTPS-এ যাও" রিডাইরেক্ট বারবার দিয়ে অসীম রিডাইরেক্ট লুপ তৈরি করতে পারে, অথবা ভুল `http://` লিংক বানাতে পারে, বা secure cookie ঠিকমতো সেট না-ও করতে পারে।

6. বর্তমানে ৯০ দিন। certbot ইনস্টলের সাথে আসা systemd timer (`snap.certbot.renew.timer` বা `certbot.timer`) বা cron দিনে দুবার `certbot renew` চালায় এবং মেয়াদ শেষের কাছাকাছি হলে নবায়ন করে। পরীক্ষা: `sudo certbot renew --dry-run` এবং `systemctl list-timers | grep certbot`।

7. Ubuntu-র MySQL-এ (এবং Debian/Ubuntu-র MariaDB-তে) ডেটাবেসের root ইউজার **unix socket authentication** ব্যবহার করে: কানেক্ট করা লিনাক্স ইউজার root হলে MySQL তাকে বিশ্বাস করে। `sudo` দিয়ে তুমি লিনাক্সের root, তাই পাসওয়ার্ড লাগে না। বাইরের কেউ এভাবে ঢুকতে পারে না, কারণ socket শুধু লোকাল।

8. `pg_hba.conf` ঠিক করে কোন ধরনের কানেকশন (local socket/host TCP), কোন ডেটাবেসে, কোন ইউজার, কোন ঠিকানা থেকে, কোন পদ্ধতিতে যাচাই হবে; ওপর থেকে প্রথম মেলা লাইন খাটে। `peer`: লিনাক্স ইউজারের নাম আর DB role-এর নাম মিললেই ঢুকতে দেয় (শুধু লোকাল socket-এ)। `scram-sha-256`: নিরাপদ challenge-response পদ্ধতিতে পাসওয়ার্ড যাচাই করে।

9. (ক) পেছনের অ্যাপ/PHP-FPM চলছে কিনা: `systemctl status myapp` বা `php8.3-fpm`। (খ) nginx যে ঠিকানায় পাঠাচ্ছে সেখানে সত্যিই কেউ শুনছে কিনা: `ss -tlnp`, `curl 127.0.0.1:3000`, socket-এর পথ মিলছে কিনা। (গ) `/var/log/nginx/error.log` পড়া: `Connection refused`, `No such file` (socket), বা `Permission denied` (socket পারমিশন বা RHEL-এ SELinux, তখন `httpd_can_network_connect`)।

10. LAMP-এ ঐতিহ্যগতভাবে PHP চলে Apache-এর **ভেতরে** `mod_php` মডিউল হিসেবে (প্রতিটি Apache process-এ PHP লোড থাকে, prefork MPM লাগে)। LEMP-এ nginx PHP চালাতে পারে না; PHP চলে **আলাদা service** PHP-FPM-এ, আর nginx FastCGI দিয়ে (সাধারণত unix socket) তাকে request পাঠায়। আধুনিক Apache-ও চাইলে PHP-FPM ব্যবহার করতে পারে।

</details>

---

[← আগের অধ্যায়](39-firewall.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](41-shell-scripting-1.md)

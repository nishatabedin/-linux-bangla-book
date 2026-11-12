# অধ্যায় ৩৯: ফায়ারওয়াল — ufw, firewalld, iptables ও nftables

## এই অধ্যায়ে যা শিখবে

- ফায়ারওয়াল (firewall) আসলে কী, কেন লাগে, আর কোনটা ফায়ারওয়াল পারে না
- লিনাক্স কার্নেলের ভেতরের **netfilter** ফ্রেমওয়ার্ক: hooks, tables, chains — ডায়াগ্রাম সহ
- একটা প্যাকেট কম্পিউটারে ঢুকে কোন পথে কোথায় যায় (packet flow)
- **stateful** ফায়ারওয়াল আর **conntrack** (connection tracking) কীভাবে কাজ করে
- **iptables**: INPUT/OUTPUT/FORWARD chain, `-A -I -D -L -n -v`, ACCEPT/DROP/REJECT, রুল সেভ করা
- **nftables** (`nft`): আধুনিক সিনট্যাক্স, set, handle দিয়ে রুল মোছা, কনফিগ ফাইল
- **ufw** (Ubuntu/Debian): enable, allow, deny, limit, status numbered, delete, app profiles
- **firewalld** (Fedora/RHEL): zones, services, `--permanent`, `--reload`, rich rules
- **NAT** আর **port forwarding** — iptables, nftables, ufw ও firewalld চারটাতেই
- রিমোট সার্ভারে ফায়ারওয়াল চালু করতে গিয়ে **নিজেকে লক-আউট না করার** কৌশল
- কোন টুল কখন ব্যবহার করবে, আর কোন দুটো একসাথে চালানো বিপজ্জনক

---

## ৩৯.১ ফায়ারওয়াল কী এবং কেন?

### একটা উপমা দিয়ে শুরু করি

ধরো তুমি একটা বড় অফিস বিল্ডিংয়ের সিকিউরিটি গার্ড। বিল্ডিংয়ে অনেকগুলো রুম — রুম ২২-এ IT টিম (SSH), রুম ৮০ আর ৪৪৩-এ রিসেপশন (ওয়েবসাইট), রুম ৩৩০৬-এ হিসাব বিভাগ (MySQL ডাটাবেস)। তোমার হাতে একটা **রুল বুক**:

1. "রিসেপশনে (৮০, ৪৪৩) যে কেউ আসতে পারবে।"
2. "IT রুমে (২২) শুধু অফিসের নিজের লোক (নির্দিষ্ট IP) আসতে পারবে।"
3. "হিসাব বিভাগে (৩৩০৬) বাইরের কেউ ঢুকবে না।"
4. "যে ভেতর থেকে বের হয়ে গিয়েছিল, সে ফিরে আসলে ঢুকতে দাও।"
5. "বাকি সবাইকে — চুপচাপ ফিরিয়ে দাও।"

প্রত্যেক আগন্তুককে তুমি রুল বুকের **উপর থেকে নিচে** মিলিয়ে দেখো; প্রথম যে রুলটা মিলে যায়, সেটাই সিদ্ধান্ত। কোনো রুল না মিললে শেষ নিয়ম (default policy) খাটে।

এই গার্ডই হলো **ফায়ারওয়াল**। আর আগন্তুকরা হলো **নেটওয়ার্ক প্যাকেট** (packet — নেটওয়ার্কে পাঠানো ডেটার ছোট ছোট টুকরো)।

### সংজ্ঞা

**ফায়ারওয়াল** হলো এমন একটা ব্যবস্থা যা নেটওয়ার্ক ট্রাফিক (আসা-যাওয়া করা প্যাকেট) পরীক্ষা করে কিছু নিয়মের ভিত্তিতে সিদ্ধান্ত নেয় — প্যাকেটটা যেতে দেবে, ফেলে দেবে, নাকি ফেরত পাঠাবে।

প্যাকেটের যেসব তথ্য দেখে সিদ্ধান্ত নেওয়া হয় (নেটওয়ার্কিং-এর বেসিক জানতে [অধ্যায় ৩৬](36-networking-basics.md) দেখো):

| তথ্য | উদাহরণ |
|---|---|
| Source IP (কোথা থেকে এসেছে) | `203.0.113.5` |
| Destination IP (কোথায় যাবে) | `192.168.1.10` |
| Protocol | TCP, UDP, ICMP |
| Source/Destination port | `22`, `80`, `443`, `53` |
| Interface (কোন নেটওয়ার্ক কার্ড দিয়ে) | `eth0`, `wlan0`, `lo` |
| Connection state | নতুন কানেকশন, নাকি পুরনো কানেকশনের অংশ |

### কেন ফায়ারওয়াল লাগে?

- **Attack surface কমানো:** তোমার সার্ভারে হয়তো ভুল করে একটা ডাটাবেস বা ডিবাগ সার্ভার সব IP-তে (`0.0.0.0`) listen করছে। ফায়ারওয়াল থাকলে সেটা ইন্টারনেট থেকে দেখা যাবে না।
- **Brute-force কমানো:** SSH-এ সেকেন্ডে শত শত লগইন চেষ্টা আসে। rate-limit রুল দিয়ে সেটা ধীর করা যায়।
- **সেগমেন্টেশন:** ডাটাবেস সার্ভারে শুধু অ্যাপ সার্ভার থেকেই কানেকশন আসবে, আর কোথাও থেকে না।
- **রাউটার/গেটওয়ে বানানো:** লিনাক্স মেশিনকে রাউটার বানিয়ে NAT করে পুরো LAN-কে ইন্টারনেট দেওয়া যায়।

> 📝 **মনে রাখো:** ফায়ারওয়াল কোনো সার্ভিসের **ভেতরের** দুর্বলতা ঠিক করে না। পোর্ট ৮০ খোলা থাকলে ওয়েব অ্যাপের SQL injection ফায়ারওয়াল আটকাবে না। ফায়ারওয়াল হলো নিরাপত্তার একটা স্তর মাত্র — বাকি স্তরগুলো [অধ্যায় ৪৫](45-security-hardening.md)-এ।

### ফায়ারওয়ালের ধরন

| ধরন | কী দেখে | উদাহরণ |
|---|---|---|
| **Packet filter (stateless)** | প্রতিটা প্যাকেট আলাদাভাবে, শুধু হেডার | পুরনো ACL, সাধারণ রাউটার |
| **Stateful firewall** | প্যাকেট কোন কানেকশনের অংশ সেটাও মনে রাখে | netfilter + conntrack (লিনাক্স) |
| **Application firewall / WAF** | HTTP-এর ভেতরের ডেটা | ModSecurity, Cloudflare WAF |
| **Host-based** | নিজের মেশিনকে রক্ষা করে | ufw, firewalld |
| **Network-based** | পুরো নেটওয়ার্ককে রক্ষা করে | লিনাক্স রাউটার, pfSense, ক্লাউডের Security Group |

লিনাক্সের ফায়ারওয়াল মূলত **stateful packet filter**, যেটা host-based আর network-based দুইভাবেই ব্যবহার করা যায়।

---

## ৩৯.২ বড় ছবি: টুলগুলোর পারস্পরিক সম্পর্ক

নতুনরা সবচেয়ে বেশি যেটায় গুলিয়ে ফেলে: "iptables, nftables, ufw, firewalld — এগুলো কি চারটা আলাদা ফায়ারওয়াল?"

**না।** আসল ফায়ারওয়াল একটাই — লিনাক্স **কার্নেলের ভেতরে** থাকা **netfilter**। বাকি সব হলো ইউজার-স্পেসের (user space — সাধারণ প্রোগ্রাম যেখানে চলে) টুল, যেগুলো কার্নেলকে রুল লিখে দেয়।

```
   ┌─────────────────────── USER SPACE ────────────────────────┐
   │                                                            │
   │   ufw (Ubuntu/Debian)        firewalld (Fedora/RHEL)       │
   │   "সহজ ফ্রন্টএন্ড"            "zone-ভিত্তিক ডেমন"            │
   │        │                            │                      │
   │        ▼                            ▼                      │
   │   iptables / ip6tables         nft (nftables backend)      │
   │   (আসলে iptables-nft)              │                       │
   │        │                            │                      │
   └────────┼────────────────────────────┼──────────────────────┘
            │   netlink সকেট দিয়ে রুল পাঠানো │
   ┌────────▼────────────────────────────▼──────────────────────┐
   │                    KERNEL SPACE                             │
   │   ┌──────────────────────────────────────────────────┐     │
   │   │  nf_tables  (আধুনিক রুল ইঞ্জিন)                    │     │
   │   │  x_tables   (পুরনো iptables-legacy ইঞ্জিন)          │     │
   │   └──────────────────────────────────────────────────┘     │
   │                 NETFILTER hooks + conntrack                 │
   └─────────────────────────────────────────────────────────────┘
```

উপমা: netfilter হলো গাড়ির **ইঞ্জিন**। iptables আর nftables হলো দুই রকম **স্টিয়ারিং-গিয়ার** (ম্যানুয়াল কন্ট্রোল)। ufw আর firewalld হলো **অটোমেটিক গিয়ার** — সহজ, কিন্তু ভেতরে একই ইঞ্জিন চালায়।

### আধুনিক ডিস্ট্রোতে কী ডিফল্ট?

| ডিস্ট্রো | ডিফল্ট ফ্রন্টএন্ড | ব্যাকএন্ড |
|---|---|---|
| Ubuntu 24.04 | ufw (ইনস্টলড, কিন্তু **disabled**) | iptables-nft → nf_tables |
| Debian 12 | কিছুই চালু নেই (ufw/firewalld ইনস্টল করতে হয়) | nftables; `iptables` কমান্ড = iptables-nft |
| Fedora | firewalld (চালু) | nftables |
| RHEL 9 / Rocky / Alma | firewalld (চালু) | nftables; `iptables` = iptables-nft |

চেক করো তোমার `iptables` কোন ব্যাকএন্ড:

```bash
$ iptables --version
iptables v1.8.10 (nf_tables)
```

`(nf_tables)` মানে iptables সিনট্যাক্সে লিখছ কিন্তু রুলগুলো আসলে nftables ইঞ্জিনে যাচ্ছে। `(legacy)` লেখা থাকলে পুরনো x_tables ইঞ্জিন।

> ⚠️ **সতর্কতা:** একই মেশিনে **ufw আর firewalld একসাথে চালু রাখবে না।** দুটোই নিজের মতো রুল লেখে, একে অপরের রুল মুছে দিতে পারে বা অপ্রত্যাশিত আচরণ হয়। একটা বেছে নাও, অন্যটা `systemctl disable --now` করো। একইভাবে ufw/firewalld চালু থাকা অবস্থায় হাতে iptables/nft রুল দিলে reload-এর সময় সেগুলো হারিয়ে যেতে পারে।

---

## ৩৯.৩ Netfilter আর্কিটেকচার — ভেতরে কী হয়

### Hooks: প্যাকেটের পথে পাঁচটা চেকপোস্ট

কার্নেলের নেটওয়ার্ক স্ট্যাকে প্যাকেট যে পথ দিয়ে চলে, সেই পথে netfilter পাঁচটা জায়গায় "হুক" (hook — আঁকড়া, যেখানে নিজের কোড ঝুলিয়ে দেওয়া যায়) বসিয়েছে। প্রতিটা হুকে প্যাকেট থামে, রুল মিলিয়ে দেখা হয়, তারপর এগোয়।

| Hook | কখন প্যাকেট এখানে আসে |
|---|---|
| **PREROUTING** | প্যাকেট নেটওয়ার্ক কার্ডে ঢোকার সাথে সাথে, routing সিদ্ধান্তের **আগে** |
| **INPUT** | routing ঠিক করেছে প্যাকেটটা **এই মেশিনের জন্যই** — লোকাল প্রসেসে পৌঁছানোর আগে |
| **FORWARD** | প্যাকেটটা অন্য কারো জন্য, এই মেশিন শুধু **পার করে দিচ্ছে** (রাউটার হিসেবে) |
| **OUTPUT** | এই মেশিনের কোনো লোকাল প্রসেস প্যাকেট **পাঠাচ্ছে** |
| **POSTROUTING** | প্যাকেট নেটওয়ার্ক কার্ড দিয়ে **বের হওয়ার ঠিক আগে** |

(nftables-এ এর বাইরে `ingress` নামে আরও একটা আগের হুক আছে, যা ইন্টারফেস-স্তরে কাজ করে — অ্যাডভান্সড কাজে লাগে।)

### Packet flow ডায়াগ্রাম

```
                    নেটওয়ার্ক থেকে প্যাকেট আসলো
                              │
                              ▼
                     ┌─────────────────┐
                     │   PREROUTING    │  ← raw, conntrack, mangle, nat(DNAT)
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │ ROUTING DECISION│  "এই প্যাকেট কি আমার জন্য?"
                     └───┬─────────┬───┘
               হ্যাঁ, আমার │         │ না, অন্য কারো
                         ▼         ▼
                ┌──────────┐   ┌──────────┐
                │  INPUT   │   │ FORWARD  │ ← mangle, filter
                └────┬─────┘   └────┬─────┘
                     │              │
                     ▼              │
          ┌────────────────────┐    │
          │ লোকাল প্রসেস         │    │
          │ (sshd, nginx ...)   │    │
          └─────────┬──────────┘    │
                    │ উত্তর পাঠালো    │
                    ▼               │
             ┌──────────┐           │
             │  OUTPUT  │ ← raw, mangle, nat, filter
             └────┬─────┘           │
                  │                 │
                  ▼                 │
          ┌─────────────────┐       │
          │ ROUTING DECISION│       │
          └────────┬────────┘       │
                   └───────┬────────┘
                           ▼
                  ┌─────────────────┐
                  │   POSTROUTING   │  ← mangle, nat(SNAT/MASQUERADE)
                  └────────┬────────┘
                           ▼
                  নেটওয়ার্ক কার্ড দিয়ে বের হলো
```

তিনটা পথ মনে রাখো:

1. **আমার কাছে আসা প্যাকেট:** PREROUTING → INPUT → লোকাল প্রসেস
2. **আমার পাঠানো প্যাকেট:** লোকাল প্রসেস → OUTPUT → POSTROUTING
3. **আমার ওপর দিয়ে পার হওয়া প্যাকেট:** PREROUTING → FORWARD → POSTROUTING

> 💡 **টিপস:** সাধারণ সার্ভার সুরক্ষায় ৯০% কাজ হয় **INPUT** chain-এ। FORWARD লাগে রাউটার/Docker/VM হোস্টে। PREROUTING/POSTROUTING লাগে NAT-এ।

### Tables: কাজ অনুযায়ী রুলের ভাগ

iptables-এ রুলগুলো কাজের ধরন অনুযায়ী আলাদা **table**-এ রাখা হয়। প্রতিটা table কিছু নির্দিষ্ট হুকে বসে।

| Table | কাজ | যে chain গুলো আছে |
|---|---|---|
| **filter** (ডিফল্ট) | প্যাকেট যেতে দেবে কি না | INPUT, FORWARD, OUTPUT |
| **nat** | ঠিকানা বদলানো (NAT) | PREROUTING, INPUT, OUTPUT, POSTROUTING |
| **mangle** | প্যাকেট হেডার বদলানো (TTL, TOS, mark) | পাঁচটাই |
| **raw** | conntrack থেকে বাদ দেওয়া (`NOTRACK`) | PREROUTING, OUTPUT |
| **security** | SELinux লেবেলিং | INPUT, FORWARD, OUTPUT |

একই হুকে একাধিক table থাকলে ক্রম: **raw → (conntrack) → mangle → nat → filter**। (nat-এর ক্রম হুক অনুযায়ী একটু আলাদা — PREROUTING-এ DNAT filter-এর আগে, POSTROUTING-এ SNAT সবার শেষে।)

### Chains আর Rules

- **Chain** = রুলের একটা ক্রমবদ্ধ তালিকা। বিল্ট-ইন chain (INPUT ইত্যাদি) হুকের সাথে যুক্ত; তুমি নিজেও user-defined chain বানাতে পারো (যেমন `SSH_RULES`) এবং সেখানে `-j` দিয়ে লাফ দিতে পারো।
- **Rule** = "যদি এই শর্ত মেলে (match), তাহলে এই কাজ করো (target)"।
- রুল **উপর থেকে নিচে** মেলানো হয়। প্রথম terminating target (ACCEPT/DROP/REJECT) পেলেই থামে।
- কোনো রুল না মিললে chain-এর **policy** খাটে (ACCEPT বা DROP)।

```
INPUT chain (policy: DROP)
 ┌───┬─────────────────────────────────────────┬─────────┐
 │ # │ match (শর্ত)                             │ target  │
 ├───┼─────────────────────────────────────────┼─────────┤
 │ 1 │ interface lo                             │ ACCEPT  │ ← প্যাকেট এখানে মিললে থামে
 │ 2 │ state ESTABLISHED,RELATED                │ ACCEPT  │
 │ 3 │ tcp dport 22                             │ ACCEPT  │
 │ 4 │ tcp dport 80,443                         │ ACCEPT  │
 └───┴─────────────────────────────────────────┴─────────┘
   কিছুই মেলেনি? → policy DROP
```

### Targets (কাজ)

| Target | কী করে | ক্লায়েন্ট কী দেখে |
|---|---|---|
| `ACCEPT` | যেতে দাও | কানেকশন হয় |
| `DROP` | চুপচাপ ফেলে দাও, কোনো উত্তর না | timeout পর্যন্ত অপেক্ষা ("hang") |
| `REJECT` | ফেলে দাও এবং ICMP/TCP RST দিয়ে জানিয়ে দাও | সাথে সাথে "Connection refused" |
| `LOG` | কার্নেল লগে লিখে **পরের রুলে যাও** (non-terminating) | — |
| `RETURN` | এই chain ছেড়ে আগের chain-এ ফিরে যাও | — |
| `DNAT` / `SNAT` / `MASQUERADE` / `REDIRECT` | ঠিকানা বদলাও (nat table) | — |
| `<chain-name>` | user-defined chain-এ লাফ দাও | — |

> 💡 **DROP না REJECT?** ইন্টারনেট-মুখী পোর্টে অনেকে DROP পছন্দ করে (স্ক্যানার ধীর হয়)। ভেতরের নেটওয়ার্কে REJECT ভালো — ভুল কনফিগ হলে ক্লায়েন্ট সাথে সাথে বুঝতে পারে, অযথা timeout-এ বসে থাকে না। দুটোই নিরাপত্তার দিক থেকে মোটামুটি সমান।

---

## ৩৯.৪ Stateful ফায়ারওয়াল ও conntrack

### সমস্যাটা কী?

তুমি তোমার ল্যাপটপ থেকে `google.com`-এ HTTPS রিকোয়েস্ট পাঠালে। তোমার প্যাকেট যায় `তোমার IP:51234 → google:443`। উত্তর আসে `google:443 → তোমার IP:51234`। এখন যদি INPUT chain-এ নিয়ম থাকে "সব DROP, শুধু ২২ খোলা", তাহলে google-এর উত্তর তো পোর্ট 51234-এ আসছে — সেটা তো ফেলে দেওয়া হবে!

Stateless ফায়ারওয়ালে এর সমাধান ছিল হাজারটা high port খুলে রাখা — যা বিপজ্জনক। **Stateful** ফায়ারওয়াল মনে রাখে: "এই কানেকশনটা তো আমিই শুরু করেছিলাম, উত্তরটা আসতে দাও।"

### conntrack কীভাবে কাজ করে

কার্নেলের **conntrack** (connection tracking) মডিউল প্রতিটা কানেকশনের একটা এন্ট্রি টেবিলে রাখে — উপমা: অফিসের গেটে "ভিজিটর রেজিস্টার"। যে বের হয়েছে তার নাম লেখা থাকে, ফিরে এলে রেজিস্টার দেখে ঢুকতে দেওয়া হয়।

```
   ক্লায়েন্ট                 conntrack টেবিল                  সার্ভার
      │  SYN ──────────▶  [NEW]  10.0.0.5:51234→93.184.216.34:443   │
      │                                                            │
      │  ◀──────── SYN/ACK  [ESTABLISHED] (উত্তর দেখা গেছে)          │
      │  ACK, ডেটা ⇄  ⇄   [ESTABLISHED]                             │
      │  FIN ...          [কিছুক্ষণ পর এন্ট্রি মুছে যায়]                 │
```

প্রতিটা প্যাকেট এই **state** গুলোর একটা পায়:

| State | মানে |
|---|---|
| `NEW` | একটা নতুন কানেকশনের প্রথম প্যাকেট |
| `ESTABLISHED` | এমন কানেকশনের প্যাকেট যেখানে দুই দিক থেকেই ট্রাফিক দেখা গেছে |
| `RELATED` | নতুন, কিন্তু কোনো বিদ্যমান কানেকশনের সাথে সম্পর্কিত (যেমন ICMP error, FTP data channel) |
| `INVALID` | কোনো কানেকশনের সাথে মেলে না / ভাঙা প্যাকেট — সাধারণত DROP করা উচিত |
| `UNTRACKED` | raw table-এ `NOTRACK` করা প্যাকেট |

> 📝 **মনে রাখো:** conntrack TCP ছাড়াও **UDP আর ICMP**-কে "কানেকশন" হিসেবে ট্র্যাক করে (যদিও এগুলো connectionless) — তাই DNS (UDP 53)-এর উত্তরও ESTABLISHED হিসেবে ঢোকে।

এ কারণেই প্রায় প্রতিটা ভালো ফায়ারওয়ালের প্রথম রুল:

```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

এই এক লাইনের ফলে তুমি বাকি রুলে শুধু **NEW** কানেকশন নিয়ে চিন্তা করলেই হয়। পারফরম্যান্সও ভালো হয় — বেশিরভাগ প্যাকেট প্রথম রুলেই মিলে বেরিয়ে যায়।

### conntrack টেবিল দেখা

`conntrack` টুল আলাদা প্যাকেজে আসে (`sudo apt install conntrack` বা `sudo dnf install conntrack-tools`):

```bash
$ sudo conntrack -L | head -3
tcp      6 431999 ESTABLISHED src=192.168.1.20 dst=192.168.1.10 sport=53122 dport=22 src=192.168.1.10 dst=192.168.1.20 sport=22 dport=53122 [ASSURED] mark=0 use=1
udp      17 25 src=192.168.1.10 dst=1.1.1.1 sport=41003 dport=53 src=1.1.1.1 dst=192.168.1.10 sport=53 dport=41003 mark=0 use=1
conntrack v1.4.7 (conntrack-tools): 2 flow entries have been shown.

$ sudo conntrack -C            # মোট এন্ট্রি সংখ্যা
2
$ cat /proc/sys/net/netfilter/nf_conntrack_max   # সর্বোচ্চ সীমা
262144
```

> ⚠️ **সতর্কতা:** খুব ব্যস্ত সার্ভারে conntrack টেবিল ভরে গেলে কার্নেল লগে `nf_conntrack: table full, dropping packet` দেখা যায় এবং নতুন কানেকশন ড্রপ হয়। তখন `net.netfilter.nf_conntrack_max` বাড়াতে হয় (sysctl নিয়ে [অধ্যায় ৪৭](47-kernel-proc-sys.md))।

---

## ৩৯.৫ iptables — ক্লাসিক টুল

যদিও নতুন সিস্টেমে nftables-ই ভবিষ্যৎ, iptables এখনও সবচেয়ে বেশি টিউটোরিয়াল, স্ক্রিপ্ট, Docker, Kubernetes আর ইন্টারভিউতে দেখা যায়। তাই ভালো করে শেখা দরকার। সব iptables কমান্ড **root** হিসেবে চালাতে হয় (`sudo`, দেখো [অধ্যায় ১৮](18-sudo-root.md))।

> 📝 IPv4-এর জন্য `iptables`, IPv6-এর জন্য **আলাদা** কমান্ড `ip6tables` (একই সিনট্যাক্স)। শুধু iptables সেট করে IPv6 খোলা রেখে দেওয়া একটা ক্লাসিক ভুল।

### সিনট্যাক্স

```
iptables [-t table] COMMAND chain [match ...] [-j TARGET [target-options]]
```

`-t` না দিলে ডিফল্ট table `filter`।

### কমান্ড (কী করতে চাও)

| অপশন | পূর্ণরূপ | কাজ |
|---|---|---|
| `-A chain` | `--append` | chain-এর **শেষে** রুল যোগ |
| `-I chain [n]` | `--insert` | n নম্বর অবস্থানে (ডিফল্ট ১ = সবার উপরে) রুল ঢোকাও |
| `-D chain rule` / `-D chain n` | `--delete` | রুল মোছো (হুবহু রুল লিখে, বা নম্বর দিয়ে) |
| `-R chain n rule` | `--replace` | n নম্বর রুল বদলাও |
| `-L [chain]` | `--list` | রুল দেখাও |
| `-S [chain]` | `--list-rules` | রুলগুলো কমান্ড আকারে দেখাও |
| `-F [chain]` | `--flush` | সব রুল মুছে ফেলো (policy বদলায় না!) |
| `-Z [chain]` | `--zero` | প্যাকেট/বাইট কাউন্টার শূন্য করো |
| `-P chain target` | `--policy` | chain-এর ডিফল্ট policy সেট |
| `-N chain` | `--new-chain` | নতুন user-defined chain |
| `-X [chain]` | `--delete-chain` | খালি user-defined chain মোছো |
| `-C chain rule` | `--check` | রুলটা আছে কি না যাচাই (স্ক্রিপ্টে কাজে লাগে) |

### দেখার অপশন

| অপশন | কাজ |
|---|---|
| `-n` | IP/পোর্টকে নামে (DNS/service) রূপান্তর করবে না — **দ্রুত**, সবসময় ব্যবহার করো |
| `-v` | বিস্তারিত: interface, প্যাকেট ও বাইট কাউন্টার |
| `--line-numbers` | প্রতিটা রুলের নম্বর দেখাও (মোছার জন্য দরকার) |
| `-x` | কাউন্টার সঠিক সংখ্যায় (K/M ছাড়া) |

### ম্যাচ (শর্ত)

| অপশন | মানে | উদাহরণ |
|---|---|---|
| `-p proto` | প্রোটোকল | `-p tcp`, `-p udp`, `-p icmp` |
| `-s addr` | source IP/নেটওয়ার্ক | `-s 192.168.1.0/24` |
| `-d addr` | destination | `-d 10.0.0.5` |
| `-i iface` | ইনকামিং ইন্টারফেস (INPUT/FORWARD/PREROUTING) | `-i eth0` |
| `-o iface` | আউটগোয়িং ইন্টারফেস (OUTPUT/FORWARD/POSTROUTING) | `-o eth1` |
| `--dport n` | destination port (`-p tcp/udp` লাগবে) | `--dport 22` |
| `--sport n` | source port | `--sport 53` |
| `-m multiport --dports a,b` | একাধিক পোর্ট | `--dports 80,443` |
| `-m conntrack --ctstate S` | কানেকশন state | `--ctstate NEW` |
| `-m limit --limit r` | rate limit | `--limit 5/min` |
| `-m comment --comment "..."` | রুলে মন্তব্য | `--comment "web"` |
| `!` | উল্টো (negation) | `! -s 10.0.0.0/8` |

পোর্ট রেঞ্জ: `--dport 6000:6007`।

### রুল দেখা

```bash
$ sudo iptables -L -n -v --line-numbers
Chain INPUT (policy DROP 12 packets, 720 bytes)
num   pkts bytes target  prot opt in   out  source          destination
1      120  9600 ACCEPT  all  --  lo   *    0.0.0.0/0       0.0.0.0/0
2     5340  412K ACCEPT  all  --  *    *    0.0.0.0/0       0.0.0.0/0     ctstate RELATED,ESTABLISHED
3       14   840 ACCEPT  tcp  --  *    *    0.0.0.0/0       0.0.0.0/0     tcp dpt:22

Chain FORWARD (policy DROP 0 packets, 0 bytes)
num   pkts bytes target  prot opt in   out  source          destination

Chain OUTPUT (policy ACCEPT 4210 packets, 501K bytes)
num   pkts bytes target  prot opt in   out  source          destination
```

কলামগুলো: `pkts/bytes` = এই রুল কতগুলো প্যাকেট মিলিয়েছে (ডিবাগিংয়ে সোনা!), `in/out` = ইন্টারফেস (`*` = যেকোনো), `opt` = IP fragment সম্পর্কিত ফ্ল্যাগ।

`-S` দিয়ে দেখলে কমান্ড আকারে পাবে — কপি করে অন্য সার্ভারে ব্যবহার করা যায়:

```bash
$ sudo iptables -S INPUT
-P INPUT DROP
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
```

nat table দেখতে: `sudo iptables -t nat -L -n -v`।

### রুল যোগ করা: -A বনাম -I

```bash
# শেষে যোগ (append)
$ sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# সবার উপরে ঢোকাও (insert at 1)
$ sudo iptables -I INPUT -s 203.0.113.66 -j DROP

# ৩ নম্বর অবস্থানে ঢোকাও
$ sudo iptables -I INPUT 3 -p tcp --dport 443 -j ACCEPT
```

> ⚠️ **ক্রমই সব:** ধরো শেষে আছে `-A INPUT -j DROP` (সব ফেলে দাও)। এখন তুমি `-A INPUT -p tcp --dport 80 -j ACCEPT` দিলে সেটা DROP-এর **পরে** বসবে — কোনোদিন কাজ করবে না! এমন ক্ষেত্রে `-I` ব্যবহার করো। নিয়ম: নির্দিষ্ট (specific) রুল উপরে, সাধারণ (generic) রুল নিচে।

### রুল মোছা

```bash
# পদ্ধতি ১: নম্বর দিয়ে
$ sudo iptables -L INPUT -n --line-numbers
num  target  prot opt source         destination
1    ACCEPT  all  --  0.0.0.0/0      0.0.0.0/0
2    ACCEPT  tcp  --  0.0.0.0/0      0.0.0.0/0     tcp dpt:80
3    ACCEPT  tcp  --  0.0.0.0/0      0.0.0.0/0     tcp dpt:22
$ sudo iptables -D INPUT 2

# পদ্ধতি ২: হুবহু রুল লিখে (-A এর জায়গায় -D)
$ sudo iptables -D INPUT -p tcp --dport 80 -j ACCEPT
```

> 💡 একটা রুল মুছলে নিচের রুলগুলোর নম্বর এক করে কমে যায়। একাধিক রুল নম্বর দিয়ে মুছতে চাইলে **বড় নম্বর থেকে ছোটর দিকে** মোছো।

### Policy

```bash
$ sudo iptables -P INPUT DROP
$ sudo iptables -P FORWARD DROP
$ sudo iptables -P OUTPUT ACCEPT
```

Policy শুধু `ACCEPT` বা `DROP` হতে পারে (REJECT না)। শুধু বিল্ট-ইন chain-এর policy থাকে।

> ⚠️ **সবচেয়ে বিপজ্জনক কমান্ড কম্বো:** রিমোট সার্ভারে `iptables -P INPUT DROP` দিয়ে তারপর `iptables -F` দিলে — সব ACCEPT রুল মুছে গেল, policy DROP রয়ে গেল → তোমার SSH সাথে সাথে বন্ধ। **`-F` policy রিসেট করে না।** রিসেট করতে চাইলে আগে `-P INPUT ACCEPT`, তারপর `-F`।

### ACCEPT, DROP, REJECT হাতে-কলমে

```bash
$ sudo iptables -A INPUT -p tcp --dport 8080 -j DROP
# অন্য মেশিন থেকে:
$ curl --max-time 5 http://192.168.1.10:8080
curl: (28) Connection timed out after 5001 milliseconds

# (ধরে নিচ্ছি 8080-এর রুলটা INPUT-এর ১ নম্বর রুল; --line-numbers দিয়ে মিলিয়ে নাও)
$ sudo iptables -R INPUT 1 -p tcp --dport 8080 -j REJECT --reject-with tcp-reset
$ curl http://192.168.1.10:8080
curl: (7) Failed to connect to 192.168.1.10 port 8080 after 2 ms: Connection refused
```

`--reject-with` এর মানগুলো: `icmp-port-unreachable` (ডিফল্ট), `icmp-host-unreachable`, `icmp-net-unreachable`, `icmp-admin-prohibited`, `tcp-reset` (শুধু TCP-তে)।

### LOG দিয়ে ডিবাগ

```bash
$ sudo iptables -A INPUT -p tcp --dport 23 -j LOG --log-prefix "TELNET-TRY: " --log-level 4
$ sudo iptables -A INPUT -p tcp --dport 23 -j DROP

$ sudo journalctl -k -g TELNET-TRY
Nov 12 10:15:01 srv kernel: TELNET-TRY: IN=eth0 OUT= MAC=... SRC=203.0.113.9 DST=192.168.1.10 LEN=60 ... PROTO=TCP SPT=40122 DPT=23 ... SYN
```

LOG non-terminating, তাই তার পরের DROP রুলটাও প্যাকেট পায়। লগ ফ্লাড এড়াতে `-m limit --limit 5/min` যোগ করো। (লগ দেখা নিয়ে বিস্তারিত [অধ্যায় ২৮](28-logs.md)।)

### একটা সম্পূর্ণ সার্ভার ফায়ারওয়াল (iptables)

```bash
#!/bin/bash
# web-server-firewall.sh — একটা সাধারণ ওয়েব সার্ভারের জন্য
set -e

# ১. আগে policy ACCEPT করে রুল পরিষ্কার (নিরাপদ রিসেট)
iptables -P INPUT ACCEPT
iptables -F
iptables -X

# ২. loopback সবসময় খোলা (অনেক প্রোগ্রাম 127.0.0.1-এ কথা বলে)
iptables -A INPUT -i lo -j ACCEPT

# ৩. পুরনো কানেকশনের উত্তর ঢুকতে দাও, ভাঙা প্যাকেট ফেলো
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# ৪. ping (ICMP echo) — ডিবাগিংয়ে কাজে লাগে
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# ৫. SSH শুধু অফিস নেটওয়ার্ক থেকে
iptables -A INPUT -p tcp -s 198.51.100.0/24 --dport 22 -m conntrack --ctstate NEW -j ACCEPT

# ৬. ওয়েব সবার জন্য
iptables -A INPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW -j ACCEPT

# ৭. বাকি সব — শেষে policy DROP
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
```

### user-defined chain

রুল গুছিয়ে রাখতে:

```bash
$ sudo iptables -N SSH_GUARD
$ sudo iptables -A SSH_GUARD -s 198.51.100.0/24 -j ACCEPT
$ sudo iptables -A SSH_GUARD -j DROP
$ sudo iptables -A INPUT -p tcp --dport 22 -j SSH_GUARD     # INPUT থেকে লাফ
```

মুছতে: আগে রেফারেন্স (`-D INPUT ...`) মোছো, তারপর `-F SSH_GUARD`, তারপর `-X SSH_GUARD`।

### SSH brute-force rate limit (`recent` মডিউল)

```bash
# ৬০ সেকেন্ডে একই IP থেকে ৪টার বেশি নতুন SSH কানেকশন হলে ফেলে দাও
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -m recent --set --name SSH
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -m recent --update --seconds 60 --hitcount 4 --name SSH -j DROP
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
```

### রুল স্থায়ী (persistent) করা

iptables রুল **মেমরিতে** থাকে — রিবুট দিলে হারিয়ে যায়। সেভ/রিস্টোর:

```bash
$ sudo iptables-save > rules.v4          # বর্তমান রুল ফাইলে
$ sudo iptables-restore < rules.v4       # ফাইল থেকে লোড (পুরনোগুলো প্রতিস্থাপন করে)
$ sudo iptables-restore --test < rules.v4   # শুধু সিনট্যাক্স চেক
```

`iptables-save` এর আউটপুট দেখতে এমন:

```
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
COMMIT
```

বুট-এ অটো লোড:

| ডিস্ট্রো | প্যাকেজ | ফাইল | সেভ কমান্ড |
|---|---|---|---|
| Debian/Ubuntu | `iptables-persistent` (`netfilter-persistent` সার্ভিস) | `/etc/iptables/rules.v4`, `/etc/iptables/rules.v6` | `sudo netfilter-persistent save` |
| RHEL/Fedora | `iptables-services` (firewalld বন্ধ করতে হবে) | `/etc/sysconfig/iptables`, `/etc/sysconfig/ip6tables` | `sudo iptables-save > /etc/sysconfig/iptables` |

> ⚠️ ufw বা firewalld চালু থাকলে iptables-persistent ইনস্টল করবে না — ঝামেলা বাড়বে। Ubuntu-তে iptables-persistent ইনস্টল করলে apt ufw সরিয়ে দিতে পারে (দুটো conflict করে)।

### `iptables-apply` — লক-আউট থেকে বাঁচার বিল্ট-ইন উপায়

```bash
$ sudo iptables-apply /etc/iptables/rules.v4
Applying new ruleset... done.
Can you establish NEW connections to the machine? (y/N)
```

নতুন রুল লোড করে প্রশ্ন করে; নির্দিষ্ট সময়ে (ডিফল্ট ১০ সেকেন্ড, `-t` দিয়ে বদলানো যায়) `y` না দিলে পুরনো রুলে ফিরে যায়। নতুন SSH সেশন খুলে টেস্ট করে তারপর `y` দাও।

---

## ৩৯.৬ nftables — আধুনিক ফায়ারওয়াল

### কেন nftables এলো?

iptables-এর কিছু সমস্যা ছিল:

- IPv4, IPv6, ARP, bridge-এর জন্য চারটা আলাদা টুল (`iptables`, `ip6tables`, `arptables`, `ebtables`) আর আলাদা কার্নেল কোড।
- একটা রুল বদলালেও পুরো টেবিল কার্নেলে নতুন করে পাঠাতে হতো — হাজার রুলে ধীর।
- ৫০০টা IP ব্লক করতে ৫০০টা রুল (ipset ছাড়া)।

**nftables** (কার্নেল ৩.১৩ থেকে, ২০১৪) সমাধান করে:

- একটা টুল `nft`, একটা সিনট্যাক্স — **`inet` family** দিয়ে IPv4 আর IPv6 একসাথে।
- কার্নেলের ভেতরে একটা ছোট **virtual machine** রুল চালায়; incremental (আংশিক) আপডেট।
- **set** আর **map** বিল্ট-ইন — হাজার IP এক রুলে।
- কোনো table/chain ডিফল্টে থাকে না — তুমি যা বানাবে তাই।

### গঠন

```
ruleset
 └── table  <family> <name>         (family: ip, ip6, inet, arp, bridge, netdev)
      ├── chain <name> { type <filter|nat|route> hook <hook> priority <n>; policy <accept|drop>; }
      │     ├── rule
      │     └── rule
      ├── set  <name> { type ipv4_addr; elements = {...} }
      └── map  ...
```

**Base chain** (হুকে যুক্ত) বনাম **regular chain** (শুধু `jump`/`goto` দিয়ে যাওয়া যায়)। **priority** ঠিক করে একই হুকে কোন chain আগে চলবে — ছোট সংখ্যা আগে। নাম দিয়েও লেখা যায়: `raw` (-300), `mangle` (-150), `dstnat` (-100), `filter` (0), `security` (50), `srcnat` (100)।

> ⚠️ **একটা গুরুত্বপূর্ণ ব্যাপার:** একই হুকে একাধিক base chain থাকলে, প্যাকেটকে **সবগুলো** chain পার হতে হয়। একটায় `accept` মানে শুধু ঐ chain থেকে ছাড়, অন্য chain-এ `drop` হলে প্যাকেট ড্রপ হবে। তাই firewalld/Docker-এর table থাকা অবস্থায় নিজের table যোগ করলে সবগুলোর রুল একসাথে খাটে।

### কমান্ড-লাইন থেকে ধাপে ধাপে

```bash
# পুরো ruleset দেখো
$ sudo nft list ruleset

# table বানাও (inet = IPv4 + IPv6)
$ sudo nft add table inet myfw

# base chain বানাও — শেলের জন্য { } ও ; কোট করতে হয়
$ sudo nft add chain inet myfw input '{ type filter hook input priority filter; policy accept; }'

# রুল যোগ
$ sudo nft add rule inet myfw input iif lo accept
$ sudo nft add rule inet myfw input ct state established,related accept
$ sudo nft add rule inet myfw input ct state invalid drop
$ sudo nft add rule inet myfw input tcp dport 22 accept
$ sudo nft add rule inet myfw input tcp dport { 80, 443 } accept
$ sudo nft add rule inet myfw input meta l4proto { icmp, ipv6-icmp } accept

# সব ঠিক থাকলে policy drop
$ sudo nft chain inet myfw input '{ policy drop; }'
```

> 💡 IPv6-এ **ICMPv6 ব্লক করবে না** — neighbor discovery (IPv6-এর ARP) ICMPv6 দিয়ে চলে; ব্লক করলে IPv6 নেটওয়ার্কিং-ই ভেঙে যায়।

### দেখা ও handle দিয়ে মোছা

nftables-এ রুল মোছা হয় **handle** (প্রতিটা রুলের অনন্য আইডি) দিয়ে:

```bash
$ sudo nft -a list chain inet myfw input
table inet myfw {
	chain input { # handle 1
		type filter hook input priority filter; policy drop;
		iif "lo" accept # handle 2
		ct state established,related accept # handle 3
		ct state invalid drop # handle 4
		tcp dport 22 accept # handle 5
		tcp dport { 80, 443 } accept # handle 7
		meta l4proto { icmp, ipv6-icmp } accept # handle 9
	}
}

$ sudo nft delete rule inet myfw input handle 7
```

| কাজ | কমান্ড |
|---|---|
| শেষে রুল | `nft add rule <family> <table> <chain> ...` |
| শুরুতে রুল | `nft insert rule <family> <table> <chain> ...` |
| নির্দিষ্ট রুলের পরে | `nft add rule ... <chain> position <handle> ...` |
| রুল বদলানো | `nft replace rule ... <chain> handle <n> ...` |
| রুল মোছা | `nft delete rule ... <chain> handle <n>` |
| chain খালি করা | `nft flush chain inet myfw input` |
| table মোছা | `nft delete table inet myfw` |
| সব মুছে ফেলা | `nft flush ruleset` |
| table তালিকা | `nft list tables` |
| handle সহ দেখা | `nft -a list ruleset` |
| IP-কে DNS নামে দেখাও | `nft -N list ruleset` (ডিফল্টে nft IP সংখ্যাতেই দেখায়) |
| ফাইল থেকে লোড | `nft -f /etc/nftables.conf` |
| শুধু চেক, লোড না | `nft -c -f file.nft` |

> ⚠️ `nft flush ruleset` **সব** টেবিল মুছে দেয় — firewalld, Docker, libvirt-এর রুলও! সার্ভারে সাবধানে।

### Sets — হাজার IP এক রুলে

```bash
$ sudo nft add set inet myfw blocklist '{ type ipv4_addr; flags interval; }'
$ sudo nft add element inet myfw blocklist '{ 203.0.113.0/24, 198.51.100.7 }'
$ sudo nft insert rule inet myfw input ip saddr @blocklist drop
$ sudo nft list set inet myfw blocklist
```

`flags interval` দিলে CIDR রেঞ্জ রাখা যায়। `flags timeout` দিয়ে এলিমেন্ট নির্দিষ্ট সময় পর নিজেই মুছে যায় — অটো-ব্যান সিস্টেম বানাতে দারুণ।

### কনফিগ ফাইল — আসল ব্যবহার

বাস্তবে nftables রুল একটা ফাইলে লেখা হয় এবং একবারে (atomic) লোড হয় — অর্ধেক রুল লোড হয়ে আটকে থাকার ঝুঁকি নেই।

| ডিস্ট্রো | ফাইল |
|---|---|
| Debian 12 / Ubuntu | `/etc/nftables.conf` |
| Fedora / RHEL 9 | `/etc/sysconfig/nftables.conf` (যেটা `/etc/nftables/*.nft` include করে) |

```
#!/usr/sbin/nft -f
# /etc/nftables.conf

flush ruleset

table inet filter {
    set ssh_allowed {
        type ipv4_addr
        flags interval
        elements = { 198.51.100.0/24, 192.168.1.0/24 }
    }

    chain input {
        type filter hook input priority filter; policy drop;

        iif "lo" accept
        ct state established,related accept
        ct state invalid drop

        meta l4proto { icmp, ipv6-icmp } accept

        ip saddr @ssh_allowed tcp dport 22 ct state new accept
        tcp dport { 80, 443 } ct state new accept

        # SSH rate limit: প্রতি IP মিনিটে ১০টার বেশি নতুন কানেকশন হলে ড্রপ
        tcp dport 22 ct state new meter ssh_meter { ip saddr limit rate over 10/minute } drop

        limit rate 5/minute log prefix "nft-drop: "
        counter
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

```bash
$ sudo nft -c -f /etc/nftables.conf        # আগে সিনট্যাক্স চেক
$ sudo nft -f /etc/nftables.conf           # লোড
$ sudo systemctl enable --now nftables     # বুটে অটো লোড
```

(নতুন nft ভার্সনে `meter`-এর জায়গায় dynamic set-এর সিনট্যাক্সও আছে; `meter` এখনও কাজ করে।)

> 📝 ফাইলের শুরুতে `flush ruleset` থাকলে প্রতিবার লোডে পরিষ্কার শুরু হয়। কিন্তু Docker/libvirt থাকলে তাদের রুলও মুছবে — তখন `flush ruleset`-এর বদলে শুধু নিজের table মুছে আবার বানাও (`table inet filter` আগে define করে তারপর `delete table inet filter`)।

### iptables থেকে nftables-এ অনুবাদ

```bash
$ iptables-translate -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT
nft 'add rule ip filter INPUT tcp dport 22 ct state new counter accept'

$ sudo iptables-save > old.v4
$ iptables-restore-translate -f old.v4 > new.nft
```

### iptables বনাম nftables পাশাপাশি

| কাজ | iptables | nftables |
|---|---|---|
| রুল দেখা | `iptables -L -n -v` | `nft list ruleset` |
| loopback | `-A INPUT -i lo -j ACCEPT` | `iif lo accept` |
| stateful | `-m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT` | `ct state established,related accept` |
| একাধিক পোর্ট | `-m multiport --dports 80,443` | `tcp dport { 80, 443 }` |
| IP থেকে | `-s 10.0.0.0/8` | `ip saddr 10.0.0.0/8` |
| ড্রপ | `-j DROP` | `drop` |
| রিজেক্ট | `-j REJECT` | `reject` |
| কাউন্টার | ডিফল্টে থাকে | `counter` লিখতে হয় |
| IPv4+IPv6 | দুটো কমান্ড | `inet` family-তে একটা |
| মোছা | `-D INPUT 3` | `delete rule ... handle 5` |
| সেভ | `iptables-save` | `nft list ruleset > file` |

---

## ৩৯.৭ ufw — Uncomplicated Firewall (Ubuntu/Debian)

নামেই পরিচয় — "জটিলতাহীন ফায়ারওয়াল"। ufw ভেতরে iptables (Ubuntu 24.04-এ iptables-nft) রুল লেখে, কিন্তু তোমাকে মানুষের ভাষার কাছাকাছি কমান্ড দেয়। ডেস্কটপ আর সাধারণ সার্ভারের জন্য সবচেয়ে সহজ।

```bash
# Ubuntu-তে আগে থেকেই থাকে; Debian-এ:
$ sudo apt install ufw
```

### ডিফল্ট policy

ufw-এর ডিফল্ট: **incoming deny, outgoing allow, routed disabled** — মানে বাইরে থেকে কেউ ঢুকতে পারবে না, তুমি যেখানে খুশি যেতে পারবে। চাইলে স্পষ্টভাবে সেট করো:

```bash
$ sudo ufw default deny incoming
Default incoming policy changed to 'deny'
(be sure to update your rules accordingly)
$ sudo ufw default allow outgoing
```

### ⚠️ চালু করার আগে সবচেয়ে জরুরি ধাপ

রিমোট সার্ভারে **আগে SSH allow করো, তারপর enable**:

```bash
$ sudo ufw allow OpenSSH          # বা: sudo ufw allow 22/tcp
Rules updated
Rules updated (v6)

$ sudo ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

SSH যদি অন্য পোর্টে চলে (ধরো 2222), তাহলে `sudo ufw allow 2222/tcp` — `OpenSSH` প্রোফাইল শুধু ২২ খোলে!

### স্ট্যাটাস দেখা

```bash
$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
OpenSSH (v6)               ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)

$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
...
```

`ufw enable` করার আগে (inactive অবস্থায়) যোগ করা রুলগুলো দেখতে: `sudo ufw show added`।

### allow / deny / reject / limit — বিভিন্ন রূপ

```bash
# পোর্ট (TCP ও UDP দুটোই)
$ sudo ufw allow 53
# শুধু TCP
$ sudo ufw allow 443/tcp
# পোর্ট রেঞ্জ (প্রোটোকল বলতেই হবে)
$ sudo ufw allow 6000:6007/tcp
# /etc/services থেকে সার্ভিসের নাম
$ sudo ufw allow http
# নির্দিষ্ট IP থেকে সব
$ sudo ufw allow from 192.168.1.50
# নির্দিষ্ট নেটওয়ার্ক থেকে নির্দিষ্ট পোর্ট
$ sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp
# নির্দিষ্ট ইন্টারফেসে
$ sudo ufw allow in on eth1 to any port 3306 proto tcp
# ব্লক — চুপচাপ ফেলে দাও
$ sudo ufw deny from 203.0.113.66
# ব্লক — জানিয়ে দাও
$ sudo ufw reject 23/tcp
# আউটগোয়িং ব্লক
$ sudo ufw deny out 25/tcp
# rate limit (brute-force সুরক্ষা)
$ sudo ufw limit ssh
# রুলে মন্তব্য
$ sudo ufw allow 8080/tcp comment 'dev server'
```

`ufw limit`: একই IP থেকে **৩০ সেকেন্ডে ৬ বা তার বেশি** কানেকশন চেষ্টা হলে সেটা deny করে। SSH সার্ভারের জন্য চমৎকার।

| ufw সিনট্যাক্স অংশ | মানে |
|---|---|
| `allow` / `deny` / `reject` / `limit` | কাজ |
| `in` / `out` | দিক (ডিফল্ট `in`) |
| `on <iface>` | ইন্টারফেস |
| `from <addr>` | source (`any` = যেকোনো) |
| `to <addr>` | destination (`any` = এই মেশিনের যেকোনো IP) |
| `port <n>` | পোর্ট |
| `proto tcp\|udp` | প্রোটোকল |
| `comment '...'` | মন্তব্য |

> ⚠️ **ক্রম এখানেও গুরুত্বপূর্ণ:** ufw রুলও প্রথম-মিল-জেতে নিয়মে চলে। আগে `allow 22` থাকলে পরে `deny from 203.0.113.66` যোগ করলে সেই IP তবুও ২২-এ ঢুকতে পারবে! ব্লক রুল উপরে বসাতে `insert` ব্যবহার করো:
> ```bash
> $ sudo ufw insert 1 deny from 203.0.113.66
> ```

### রুল মোছা — numbered

```bash
$ sudo ufw status numbered
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 80/tcp                     ALLOW IN    Anywhere
[ 3] 8080/tcp                   ALLOW IN    Anywhere                   # dev server
[ 4] 22/tcp (v6)                ALLOW IN    Anywhere (v6)
[ 5] 80/tcp (v6)                ALLOW IN    Anywhere (v6)
[ 6] 8080/tcp (v6)              ALLOW IN    Anywhere (v6)              # dev server

$ sudo ufw delete 3
Deleting:
 allow 8080/tcp comment 'dev server'
Proceed with operation (y|n)? y
Rule deleted
```

লক্ষ করো, IPv6 রুল (`[6]`) এখনও আছে — নম্বর দিয়ে মুছলে শুধু ঐ একটাই মোছে। **রুলের নাম দিয়ে** মুছলে v4 আর v6 দুটোই একসাথে যায়:

```bash
$ sudo ufw delete allow 8080/tcp
Rule deleted
Rule deleted (v6)
```

### অ্যাপ প্রোফাইল (application profiles)

প্যাকেজগুলো (openssh-server, nginx, apache2 ইত্যাদি) `/etc/ufw/applications.d/`-এ প্রোফাইল ফাইল রাখে, যাতে পোর্ট নম্বর মনে না রেখে নাম দিয়ে খোলা যায়।

```bash
$ sudo ufw app list
Available applications:
  Nginx Full
  Nginx HTTP
  Nginx HTTPS
  OpenSSH

$ sudo ufw app info 'Nginx Full'
Profile: Nginx Full
Title: Web Server (Nginx, HTTP + HTTPS)
Description: Small, but very powerful and efficient web server

Ports:
  80,443/tcp

$ sudo ufw allow 'Nginx Full'      # নামে স্পেস থাকলে কোট দাও
```

নিজের অ্যাপের প্রোফাইল বানাও — `/etc/ufw/applications.d/myapp`:

```ini
[MyApp]
title=My Node.js App
description=Internal API and admin panel
ports=3000/tcp|9229/tcp
```

```bash
$ sudo ufw app update MyApp
$ sudo ufw allow MyApp
```

(`ports=` এ একাধিক পোর্ট কমা দিয়ে একই প্রোটোকলে, যেমন `80,443/tcp`; ভিন্ন প্রোটোকল `|` দিয়ে আলাদা, যেমন `53/tcp|53/udp`।)

### অন্যান্য দরকারি ufw কমান্ড

| কমান্ড | কাজ |
|---|---|
| `ufw disable` | ফায়ারওয়াল বন্ধ (রুল থেকে যায়) |
| `ufw reload` | রুল আবার লোড |
| `ufw reset` | সব রুল মুছে ডিফল্টে ফেরো (ফায়ারওয়াল disable হয়; পুরনো রুল ফাইল ব্যাকআপ রাখে) |
| `ufw logging on\|off\|low\|medium\|high\|full` | লগিং লেভেল |
| `ufw --dry-run allow 80` | আসলে না করে কী হতো দেখাও |
| `ufw show raw` | ভেতরের আসল iptables রুল দেখাও |
| `ufw show listening` | কোন পোর্টে প্রোগ্রাম listen করছে আর কোন রুল প্রযোজ্য |
| `ufw route allow in on eth1 out on eth0` | FORWARD (রাউটিং) ট্রাফিকের রুল |

লগ যায় `/var/log/ufw.log` এ (বা `journalctl -k`), ব্লক হওয়া প্যাকেট `[UFW BLOCK]` প্রিফিক্স সহ:

```bash
$ sudo grep 'UFW BLOCK' /var/log/ufw.log | tail -1
Nov 12 11:02:44 srv kernel: [UFW BLOCK] IN=eth0 OUT= MAC=... SRC=45.155.205.3 DST=192.168.1.10 LEN=40 ... PROTO=TCP SPT=48211 DPT=3389 WINDOW=1024 RES=0x00 SYN URGP=0
```

### ufw-এর ফাইলগুলো

| ফাইল | কী আছে |
|---|---|
| `/etc/default/ufw` | ডিফল্ট policy (`DEFAULT_FORWARD_POLICY` ইত্যাদি), `IPV6=yes` |
| `/etc/ufw/ufw.conf` | `ENABLED=yes`, লগিং লেভেল |
| `/etc/ufw/user.rules`, `user6.rules` | তোমার `ufw allow ...` রুলগুলো (হাতে এডিট করবে না) |
| `/etc/ufw/before.rules`, `after.rules` | ufw রুলের আগে/পরে চলা কাঁচা iptables রুল — NAT এখানে লেখা হয় |
| `/etc/ufw/sysctl.conf` | ufw লোডের সময় সেট হওয়া kernel প্যারামিটার |
| `/etc/ufw/applications.d/` | অ্যাপ প্রোফাইল |

### ⚠️ Docker আর ufw — একটা কুখ্যাত ফাঁদ

Docker-এ `docker run -p 3306:3306 mysql` দিলে Docker নিজে iptables-এর `nat` table-এ DNAT রুল আর `FORWARD` chain-এ (তার `DOCKER` chain-এ) রুল বসায়। এই প্যাকেট **INPUT** chain-এ যায়ই না (কারণ DNAT-এর পর এটা কন্টেইনারের দিকে forward হয়), তাই ufw-এর `deny 3306` **কাজ করে না** — পোর্টটা পুরো ইন্টারনেটে খোলা!

সমাধান:
- পোর্ট শুধু localhost-এ বাঁধো: `-p 127.0.0.1:3306:3306`
- অথবা Docker-এর `DOCKER-USER` chain-এ নিজের রুল দাও (Docker সেটা ছোঁয় না)।

(কন্টেইনার নিয়ে বিস্তারিত [অধ্যায় ৪৮](48-containers-virtualization.md)-এ।)

---

## ৩৯.৮ firewalld — zone-ভিত্তিক ডাইনামিক ফায়ারওয়াল (Fedora/RHEL)

firewalld একটা **ডেমন** (daemon — ব্যাকগ্রাউন্ডে চলা সার্ভিস) যা রুল সামলায় এবং D-Bus দিয়ে কমান্ড নেয়। এর কমান্ড-লাইন টুল `firewall-cmd`, GUI `firewall-config`। Fedora আর RHEL 9-এ এটা ডিফল্টে চালু, ব্যাকএন্ড nftables।

```bash
$ sudo systemctl status firewalld
$ sudo firewall-cmd --state
running
# Ubuntu/Debian-এ চাইলে: sudo apt install firewalld (তখন ufw disable করো)
```

### মূল ধারণা ১: Zones

**Zone** = একটা বিশ্বাসের স্তর (trust level), সাথে সেই স্তরের রুলসেট। প্রতিটা **interface** বা **source IP** কোনো একটা zone-এর সাথে যুক্ত থাকে। উপমা: একটা বাড়িতে ড্রয়িং রুম (অতিথি আসতে পারে), বেডরুম (শুধু পরিবার), রান্নাঘর (শুধু রাঁধুনি) — দরজা দিয়ে কে এল সেই অনুযায়ী কোন রুমের নিয়ম খাটবে।

```
   eth0 (ইন্টারনেট) ─────▶ zone: public   → শুধু ssh, dhcpv6-client
   eth1 (অফিস LAN) ─────▶ zone: internal → ssh, samba, mdns ...
   source 10.8.0.0/24 ──▶ zone: trusted  → সবকিছু অনুমোদিত
```

প্যাকেট কোন zone পাবে: আগে **source** মেলানো হয়, না মিললে **interface**-এর zone, তাও না থাকলে **default zone**।

| প্রি-ডিফাইন্ড zone | আচরণ (বিশ্বাস কম → বেশি) |
|---|---|
| `drop` | সব ইনকামিং চুপচাপ ড্রপ, কোনো উত্তর নেই; শুধু বাইরে যাওয়া চলে |
| `block` | সব ইনকামিং reject (icmp-host-prohibited) |
| `public` | পাবলিক নেটওয়ার্ক; শুধু বেছে নেওয়া সার্ভিস (RHEL ডিফল্ট zone) |
| `external` | রাউটারের বাইরের দিক; masquerading চালু |
| `dmz` | DMZ-এর পাবলিক সার্ভার |
| `work` | অফিস নেটওয়ার্ক |
| `home` | বাসার নেটওয়ার্ক |
| `internal` | ভেতরের নেটওয়ার্ক |
| `trusted` | সব কানেকশন অনুমোদিত |

> 📝 Fedora Workstation-এর ডিফল্ট zone `FedoraWorkstation` — এটা ১০২৫-৬৫৫৩৫ পর্যন্ত সব high port খোলা রাখে (ডেস্কটপের সুবিধার জন্য)। Fedora Server ও RHEL-এ `public`।

### মূল ধারণা ২: Runtime বনাম Permanent

এটাই firewalld-এর সবচেয়ে বড় ধাঁধা:

```
            firewall-cmd --add-...              firewall-cmd --add-... --permanent
                    │                                       │
                    ▼                                       ▼
          ┌───────────────────┐                  ┌───────────────────────┐
          │  RUNTIME কনফিগ     │                  │  PERMANENT কনফিগ        │
          │  (এখন চালু, মেমরিতে) │                  │  (/etc/firewalld/ XML)  │
          └───────────────────┘                  └───────────────────────┘
                    ▲      │                                 │
                    │      └── --runtime-to-permanent ──────▶│
                    └──────────────── --reload ──────────────┘
                    (রিবুট/reload-এ runtime হারিয়ে permanent লোড হয়)
```

- `--permanent` ছাড়া: সাথে সাথে কাজ করে, কিন্তু reload/রিবুটে হারিয়ে যায়।
- `--permanent` সহ: ফাইলে লেখা হয়, কিন্তু **এখনই কাজ করে না** — `--reload` লাগবে।

দুটো সাধারণ কৌশল:

```bash
# কৌশল ক: permanent লেখো, তারপর reload
$ sudo firewall-cmd --permanent --add-service=http
success
$ sudo firewall-cmd --reload
success

# কৌশল খ: runtime-এ টেস্ট করো, ঠিক থাকলে permanent বানাও (নিরাপদ!)
$ sudo firewall-cmd --add-service=http
$ ... টেস্ট ...
$ sudo firewall-cmd --runtime-to-permanent
```

> 💡 কৌশল খ লক-আউটের বিরুদ্ধে দারুণ: runtime-এ কিছু ভুল করলে `firewall-cmd --reload` দিলেই পুরনো permanent অবস্থায় ফিরে যাবে।

### zone দেখা ও বদলানো

```bash
$ firewall-cmd --get-default-zone
public
$ firewall-cmd --get-zones
block dmz drop external home internal nm-shared public trusted work
$ firewall-cmd --get-active-zones
public
  interfaces: eth0

$ sudo firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: cockpit dhcpv6-client ssh
  ports:
  protocols:
  forward: yes
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:

$ sudo firewall-cmd --list-all --zone=internal    # অন্য zone
$ sudo firewall-cmd --list-all-zones              # সব zone

$ sudo firewall-cmd --set-default-zone=home       # ডিফল্ট বদলাও (runtime+permanent দুটোই)
$ sudo firewall-cmd --zone=internal --change-interface=eth1 --permanent
$ sudo firewall-cmd --zone=trusted --add-source=10.8.0.0/24 --permanent
$ sudo firewall-cmd --reload
```

> 📝 NetworkManager চালিত সিস্টেমে ইন্টারফেসের zone NetworkManager-এর connection প্রোফাইলেও থাকে (`nmcli connection modify eth1 connection.zone internal`) — দুই জায়গায় দুই রকম থাকলে বিভ্রান্তি হয়।

### services ও ports

firewalld-এ **service** = নাম দেওয়া পোর্ট/প্রোটোকলের গুচ্ছ (ufw-এর অ্যাপ প্রোফাইলের মতো), XML ফাইলে সংজ্ঞায়িত।

```bash
$ firewall-cmd --get-services | tr ' ' '\n' | grep -E '^(http|https|ssh|mysql)$'
http
https
mysql
ssh

$ firewall-cmd --info-service=https
https
  ports: 443/tcp
  protocols:
  source-ports:
  modules:
  destination:
  includes:
  helpers:

$ sudo firewall-cmd --permanent --add-service={http,https}   # bash brace expansion
$ sudo firewall-cmd --permanent --remove-service=cockpit
$ sudo firewall-cmd --permanent --add-port=8080/tcp
$ sudo firewall-cmd --permanent --add-port=60000-61000/udp
$ sudo firewall-cmd --permanent --remove-port=8080/tcp
$ sudo firewall-cmd --reload
$ firewall-cmd --list-services
dhcpv6-client http https ssh
$ firewall-cmd --list-ports
60000-61000/udp
```

সার্ভিস ফাইলের জায়গা:

| ডিরেক্টরি | কী |
|---|---|
| `/usr/lib/firewalld/` | প্যাকেজের দেওয়া ডিফল্ট zone/service (হাতে বদলাবে না) |
| `/etc/firewalld/` | তোমার কনফিগ — এখানকার ফাইল ডিফল্টকে override করে |
| `/etc/firewalld/firewalld.conf` | মূল সেটিং (`DefaultZone`, `FirewallBackend=nftables`, `LogDenied`) |

নিজের সার্ভিস বানাও:

```bash
$ sudo firewall-cmd --permanent --new-service=myapp
$ sudo firewall-cmd --permanent --service=myapp --set-short="My App"
$ sudo firewall-cmd --permanent --service=myapp --add-port=3000/tcp
$ sudo firewall-cmd --reload
$ sudo firewall-cmd --permanent --add-service=myapp && sudo firewall-cmd --reload
```

### Rich rules — সূক্ষ্ম নিয়ন্ত্রণ

শুধু "সার্ভিস খোলো" দিয়ে না হলে (যেমন নির্দিষ্ট IP থেকে, লগ সহ, rate limit সহ) **rich rule**:

```bash
# শুধু অফিস নেটওয়ার্ক থেকে SSH
$ sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="198.51.100.0/24" service name="ssh" accept'

# একটা IP পুরো ব্লক
$ sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="203.0.113.66" drop'

# MySQL শুধু অ্যাপ সার্ভার থেকে, লগ সহ
$ sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="10.0.0.20" port port="3306" protocol="tcp" log prefix="mysql-ok " level="info" accept'

# HTTP-তে মিনিটে ২৫টা নতুন কানেকশন
$ sudo firewall-cmd --permanent --add-rich-rule='rule service name="http" accept limit value="25/m"'

$ sudo firewall-cmd --reload
$ firewall-cmd --list-rich-rules
```

মুছতে একই স্ট্রিং দিয়ে `--remove-rich-rule='...'`।

> ⚠️ ssh সার্ভিস zone-এ আগে থেকেই খোলা থাকলে শুধু rich rule যোগ করলে সীমাবদ্ধতা হবে না — `--remove-service=ssh` করতে হবে, নইলে সবাই সার্ভিস দিয়েই ঢুকে যাবে।

### অন্যান্য দরকারি firewall-cmd অপশন

| অপশন | কাজ |
|---|---|
| `--timeout=5m` | runtime রুল ৫ মিনিট পর নিজেই মুছে যাবে (`--permanent`-এর সাথে চলে না) |
| `--panic-on` / `--panic-off` / `--query-panic` | সব ট্রাফিক সাথে সাথে বন্ধ (আক্রমণের সময়) — রিমোট থেকে দিলে তুমিও কাটা পড়বে! |
| `--check-config` | permanent কনফিগ ফাইলের সিনট্যাক্স যাচাই |
| `--complete-reload` | পুরোপুরি reload, চলমান কানেকশনের state সহ মুছে যায় |
| `--query-service=ssh` | সার্ভিস চালু আছে কি না (`yes`/`no`) |
| `--set-log-denied=all` | ড্রপ হওয়া প্যাকেট লগ করো (`unicast`, `off` ইত্যাদি) |
| `--add-icmp-block=echo-request` | ping ব্লক |
| `--zone=X` | যেকোনো অপশনে নির্দিষ্ট zone |

> 💡 firewalld-এর নতুন ভার্সনে zone-এর মধ্যে ট্রাফিক নিয়ন্ত্রণের জন্য **policy objects** (`--new-policy`, `--policy=...`) আছে; পুরনো `--direct` ইন্টারফেস deprecated। জটিল রাউটার সেটআপে policy ব্যবহার করো।

---

## ৩৯.৯ NAT ও Port Forwarding

### NAT কী?

**NAT** (Network Address Translation) = প্যাকেটের IP ঠিকানা (এবং পোর্ট) পথের মাঝে বদলে দেওয়া। (ধারণাগত ভিত্তি [অধ্যায় ৩৬](36-networking-basics.md)-এ।) দুই ধরন:

- **SNAT / MASQUERADE** (source বদলানো, POSTROUTING-এ): ভেতরের প্রাইভেট IP-গুলোকে রাউটারের পাবলিক IP-র পেছনে লুকানো — বাসার Wi-Fi রাউটার ঠিক এটাই করে। MASQUERADE হলো SNAT যা ইন্টারফেসের বর্তমান IP নিজেই বের করে নেয় (DHCP/ডাইনামিক IP-তে সুবিধাজনক); IP স্থির হলে `SNAT --to-source` একটু দ্রুত।
- **DNAT** (destination বদলানো, PREROUTING-এ): "পোর্ট ফরওয়ার্ডিং" — বাইরের পোর্ট ৮০৮০-তে আসা ট্রাফিক ভেতরের কোনো মেশিনের ৮০-তে পাঠানো।

```
                     লিনাক্স গেটওয়ে
  ইন্টারনেট          ┌───────────────────┐          LAN 192.168.50.0/24
                    │ eth0: 203.0.113.10│
  ক্লায়েন্ট ──:8080──▶│  PREROUTING DNAT   │──:80──▶ ওয়েব সার্ভার 192.168.50.10
                    │  → 192.168.50.10:80│
                    │                    │
  google ◀──────────│  POSTROUTING       │◀─────── ল্যাপটপ 192.168.50.23
  (দেখে src=        │  MASQUERADE        │         (src=192.168.50.23)
   203.0.113.10)    │ eth1: 192.168.50.1 │
                    └───────────────────┘
```

conntrack মনে রাখে কোন কানেকশনের ঠিকানা কী বদলানো হয়েছিল, তাই **উত্তরের প্যাকেট অটোমেটিক উল্টো অনুবাদ হয়** — তোমাকে উল্টো দিকের রুল লিখতে হয় না।

### ধাপ ০: IP forwarding চালু করো

ডিফল্টে লিনাক্স অন্যের প্যাকেট forward করে না। রাউটার বানাতে:

```bash
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0
$ sudo sysctl -w net.ipv4.ip_forward=1                 # এখনই
$ echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-forward.conf
$ sudo sysctl --system                                 # স্থায়ী
```

(IPv6-এর জন্য `net.ipv6.conf.all.forwarding = 1`।)

### iptables দিয়ে

```bash
# LAN (eth1) কে ইন্টারনেট (eth0) দাও
sudo iptables -t nat -A POSTROUTING -s 192.168.50.0/24 -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT

# পোর্ট ফরওয়ার্ড: বাইরের 8080 → ভেতরের 192.168.50.10:80
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.50.10:80
sudo iptables -A FORWARD -i eth0 -o eth1 -p tcp -d 192.168.50.10 --dport 80 -m conntrack --ctstate NEW -j ACCEPT

# একই মেশিনে পোর্ট রিডাইরেক্ট: 80 → লোকাল 8080
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-ports 8080
```

> 📝 লক্ষ করো, FORWARD রুলে destination port **80** (DNAT-এর পরের ঠিকানা), 8080 না — কারণ FORWARD chain DNAT-এর **পরে** আসে (ডায়াগ্রাম মনে করো)। আর গেটওয়ে মেশিন **নিজে** থেকে `curl localhost:8080` করলে সেটা PREROUTING দিয়ে যায় না (OUTPUT দিয়ে যায়), তাই টেস্ট করবে অন্য মেশিন থেকে।

### nftables দিয়ে

```
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        iifname "eth0" tcp dport 8080 dnat to 192.168.50.10:80
    }
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 192.168.50.0/24 oifname "eth0" masquerade
    }
}

table inet filter {
    chain forward {
        type filter hook forward priority filter; policy drop;
        ct state established,related accept
        iifname "eth1" oifname "eth0" accept
        iifname "eth0" oifname "eth1" ip daddr 192.168.50.10 tcp dport 80 ct state new accept
    }
}
```

`ct status dnat accept` লিখলে সব DNAT হওয়া কানেকশন একবারে অনুমোদন করা যায়।

### ufw দিয়ে

ufw-এ NAT-এর জন্য সরাসরি কমান্ড নেই; ফাইল এডিট করতে হয়:

১. `/etc/default/ufw`-এ: `DEFAULT_FORWARD_POLICY="ACCEPT"` (অথবা DROP রেখে `ufw route allow` দিয়ে নির্দিষ্ট forward অনুমোদন)।

২. `/etc/ufw/sysctl.conf`-এ এই লাইন আনকমেন্ট করো: `net/ipv4/ip_forward=1`

৩. `/etc/ufw/before.rules`-এর **একদম শুরুতে** (`*filter` লাইনের আগে):

```
*nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A PREROUTING -i eth0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.50.10:80
-A POSTROUTING -s 192.168.50.0/24 -o eth0 -j MASQUERADE
COMMIT
```

৪. `sudo ufw disable && sudo ufw enable` (বা `sudo ufw reload`)।

### firewalld দিয়ে

```bash
# masquerade (ইন্টারনেটের দিকের zone-এ)
$ sudo firewall-cmd --permanent --zone=external --change-interface=eth0
$ sudo firewall-cmd --permanent --zone=internal --change-interface=eth1
$ sudo firewall-cmd --permanent --zone=external --add-masquerade

# পোর্ট ফরওয়ার্ড অন্য হোস্টে
$ sudo firewall-cmd --permanent --zone=external \
    --add-forward-port=port=8080:proto=tcp:toport=80:toaddr=192.168.50.10

# একই হোস্টে পোর্ট বদল (toaddr ছাড়া)
$ sudo firewall-cmd --permanent --zone=public --add-forward-port=port=80:proto=tcp:toport=8080

$ sudo firewall-cmd --reload
$ firewall-cmd --zone=external --list-forward-ports
port=8080:proto=tcp:toport=80:toaddr=192.168.50.10
```

masquerade চালু করলে firewalld IPv4 forwarding নিজেই চালু করে নেয়। ভেতরের zone থেকে বাইরের zone-এ forward অনুমতি নিয়ন্ত্রণে নতুন ভার্সনে policy object ব্যবহার হয়।

---

## ৩৯.১০ নিজেকে লক-আউট করো না! (রিমোট সার্ভারের সোনালি নিয়ম)

যেকোনো সিস্টেম অ্যাডমিনের জীবনে অন্তত একবার এটা ঘটে: SSH দিয়ে ক্লাউড সার্ভারে ঢুকে ফায়ারওয়াল চালু করলে — আর টার্মিনাল জমে গেল। সার্ভার ৫০০ কিলোমিটার দূরে, কিংবা ক্লাউডে। এখন?

### প্রতিরোধের চেকলিস্ট

1. **আগে allow, পরে deny/enable।** SSH পোর্ট (যেটা আসলে ব্যবহার করছ — `ss -tlnp | grep sshd` দিয়ে দেখো) সবার আগে খোলো।
2. **ESTABLISHED,RELATED রুল রাখো।** এটা না থাকলে চলমান SSH সেশনের উত্তর প্যাকেটও আটকে যাবে।
3. **বর্তমান সেশন বন্ধ করবে না।** নতুন টার্মিনাল থেকে নতুন SSH কানেকশন খুলে টেস্ট করো; সফল হলে তবেই পুরনোটা ছাড়ো।
4. **টাইমার-ভিত্তিক রোলব্যাক রাখো** — রুল দেওয়ার আগে একটা "ডেড-ম্যানস সুইচ":

```bash
# ufw: ৫ মিনিট পর নিজেই disable হবে
$ echo "ufw disable" | sudo at now + 5 minutes      # at প্যাকেজ লাগবে
# অথবা systemd দিয়ে (at ছাড়াই):
$ sudo systemd-run --on-active=5min /usr/sbin/ufw disable

# iptables: আগে ব্যাকআপ, তারপর টাইমার
$ sudo iptables-save > /root/fw-backup.v4
$ sudo systemd-run --on-active=5min /bin/sh -c 'iptables-restore < /root/fw-backup.v4'

# nftables
$ sudo nft list ruleset > /root/nft-backup.nft
$ sudo systemd-run --on-active=5min /bin/sh -c 'nft flush ruleset; nft -f /root/nft-backup.nft'

# firewalld: runtime-এ timeout দিয়ে টেস্ট, অথবা কৌশল খ (runtime → টেস্ট → runtime-to-permanent)
```

সব ঠিক থাকলে টাইমার বাতিল করো: `atq` / `atrm <id>`, বা `systemctl list-timers` দেখে `sudo systemctl stop run-XXXX.timer`।

5. **`iptables-apply` ব্যবহার করো** (৩৯.৫ দেখো)।
6. **আউট-অফ-ব্যান্ড অ্যাক্সেস জেনে রাখো:** ক্লাউডের ওয়েব কনসোল (AWS EC2 Serial Console, DigitalOcean Droplet Console ইত্যাদি), ডেটা সেন্টারের IPMI/iDRAC/KVM। লক-আউট হলে এখান দিয়ে ঢুকে `ufw disable` বা `systemctl stop firewalld`।
7. **ক্লাউড ফায়ারওয়াল আলাদা স্তর:** AWS Security Group / GCP firewall rules হোস্টের বাইরে কাজ করে। হোস্টে পোর্ট খুললেও ক্লাউডের দিকে বন্ধ থাকলে কানেকশন আসবে না — দুই জায়গাতেই দেখো।
8. **`-F` এর আগে `-P ACCEPT`** — আবার মনে করিয়ে দিলাম।

> ⚠️ **সতর্কতা:** `firewall-cmd --panic-on`, `nft flush ruleset` (DROP policy-র chain রেখে), `iptables -P INPUT DROP` — এগুলো রিমোট থেকে দেওয়ার আগে তিনবার ভাবো।

---

## ৩৯.১১ ফায়ারওয়াল টেস্টিং ও ট্রাবলশুটিং

| প্রশ্ন | কীভাবে দেখবে |
|---|---|
| সার্ভিস আদৌ চলছে ও কোন IP-তে listen করছে? | `sudo ss -tulnp` (বিস্তারিত [অধ্যায় ৩৭](37-network-commands.md)) |
| বাইরে থেকে পোর্ট খোলা? | অন্য মেশিন থেকে `nc -zv 192.168.1.10 22` বা `nmap -p 22,80,443 192.168.1.10` |
| কোন রুল প্যাকেট ধরছে? | `iptables -L -n -v` এর কাউন্টার; nft-এ রুলে `counter` যোগ করো |
| ড্রপ হওয়া প্যাকেট | LOG রুল / `ufw logging on` / `firewall-cmd --set-log-denied=all`, তারপর `journalctl -k -f` |
| সব রুল আসলে কী (সব টুলের) | `sudo nft list ruleset` — iptables-nft, ufw, firewalld, Docker সবার রুল এখানে দেখা যায় |

```bash
$ nc -zv 192.168.1.10 22
Connection to 192.168.1.10 22 port [tcp/ssh] succeeded!
$ nc -zv -w 3 192.168.1.10 3306
nc: connect to 192.168.1.10 port 3306 (tcp) timed out: Operation now in progress
```

"timed out" = সম্ভবত DROP; "Connection refused" = REJECT অথবা পোর্টে কেউ listen করছে না।

> 💡 ট্রাবলশুটিং-এর দ্রুত পরীক্ষা: কয়েক সেকেন্ডের জন্য ফায়ারওয়াল বন্ধ করে দেখো সমস্যা যায় কি না (লোকাল মেশিনে, বা কনসোল অ্যাক্সেস থাকলে)। গেলে সমস্যা ফায়ারওয়ালে, না গেলে অন্য কোথাও (সার্ভিস, routing, ক্লাউড ফায়ারওয়াল)।

---

## ৩৯.১২ কোন টুল কখন?

| পরিস্থিতি | সুপারিশ |
|---|---|
| Ubuntu ডেস্কটপ/সাধারণ সার্ভার | **ufw** |
| Fedora / RHEL / Rocky / Alma | **firewalld** (ডিফল্ট, বদলানোর দরকার নেই) |
| একাধিক ইন্টারফেস, ল্যাপটপ যা বিভিন্ন নেটওয়ার্কে যায় | **firewalld** (zone) |
| জটিল রাউটার/গেটওয়ে, হাজার IP-র blocklist, পূর্ণ নিয়ন্ত্রণ | **nftables** সরাসরি, ফাইলে |
| পুরনো স্ক্রিপ্ট, Docker/Kubernetes ভেতরের রুল বোঝা, ইন্টারভিউ | **iptables** জানতেই হবে |

---

## ৩৯.১৩ কীবোর্ড শর্টকাট ও দ্রুত কাজের কৌশল

ফায়ারওয়াল টুলগুলোর নিজস্ব কীবোর্ড শর্টকাট নেই (এগুলো কমান্ড-লাইন টুল), তবে টার্মিনালের এই শর্টকাটগুলো ফায়ারওয়াল নিয়ে কাজ করার সময় খুব কাজে লাগে (সব শর্টকাট [অধ্যায় ৬](06-terminal-shortcuts.md)-এ):

| শর্টকাট | কাজে লাগে যখন |
|---|---|
| `Ctrl + R` তারপর `iptables` টাইপ | আগের লম্বা iptables/firewall-cmd কমান্ড খুঁজে আবার চালাতে |
| `Alt + .` | আগের কমান্ডের শেষ argument বসাতে (যেমন একই IP বা ফাইল) |
| `Ctrl + A` / `Ctrl + E` | লম্বা রুলের শুরু/শেষে যেতে — `-A` কে `-D` বানাতে |
| `Ctrl + W` | এক শব্দ মুছতে |
| `!!` | আগের কমান্ড — `sudo !!` (sudo দিতে ভুলে গেলে) |
| `^old^new` | আগের কমান্ডে প্রথম `old` কে `new` দিয়ে বদলে চালাও — যেমন `^-A^-D` রুল মুছতে |
| `Ctrl + C` | ufw-এর `(y\|n)` প্রশ্ন বা ঝুলে থাকা `nc`/`curl` বাতিল করতে |
| `watch -n1 'sudo iptables -L -n -v'` | কাউন্টার লাইভ দেখতে (`Ctrl + C` দিয়ে বের হও) |

GUI টুল: Ubuntu-তে **gufw** (`sudo apt install gufw`), Fedora/RHEL-এ **firewall-config** — দুটোই মাউস দিয়ে রুল দেওয়ার জন্য; শেখার সময় ভেতরে কী রুল তৈরি হলো তা `nft list ruleset` দিয়ে মিলিয়ে দেখো।

---

## হাতে-কলমে অনুশীলন

> ⚠️ এই ল্যাবগুলো একটা **লোকাল VM**-এ করো (VirtualBox/KVM — [অধ্যায় ৪৮](48-containers-virtualization.md)), প্রোডাকশন সার্ভারে না। দুটো VM থাকলে ভালো (একটা থেকে অন্যটা টেস্ট করবে)।

### ল্যাব ১: ufw দিয়ে ওয়েব সার্ভার সুরক্ষা (Ubuntu)

```bash
# ১. একটা টেস্ট ওয়েব সার্ভার
$ sudo apt install -y nginx
$ curl -sI http://localhost | head -1
HTTP/1.1 200 OK

# ২. ফায়ারওয়ালের অবস্থা
$ sudo ufw status
Status: inactive

# ৩. ডিফল্ট আর SSH
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw limit OpenSSH

# ৪. চালু
$ sudo ufw enable

# ৫. অন্য VM থেকে: curl --max-time 3 http://<VM-IP>  → timeout হওয়ার কথা
# ৬. HTTP খোলো
$ sudo ufw allow 'Nginx HTTP'
# ৭. আবার অন্য VM থেকে curl → এবার 200 OK
# ৮. দেখো ও মোছো
$ sudo ufw status numbered
$ sudo ufw delete allow 'Nginx HTTP'
```

### ল্যাব ২: iptables-এর কাউন্টার দিয়ে প্যাকেট গোনা

```bash
$ sudo ufw disable                       # ল্যাবের জন্য ufw সরাও
$ sudo iptables-save > ~/before.v4       # ব্যাকআপ
$ sudo iptables -I INPUT -p icmp --icmp-type echo-request -j ACCEPT
$ sudo iptables -L INPUT -n -v --line-numbers | head -4
# অন্য VM থেকে: ping -c 5 <VM-IP>
$ sudo iptables -L INPUT -n -v --line-numbers | head -4
# প্রথম রুলের pkts কলাম ৫ বেড়েছে কি?
$ sudo iptables -R INPUT 1 -p icmp --icmp-type echo-request -j DROP
# আবার ping → এবার কোনো উত্তর নেই, কিন্তু কাউন্টার বাড়ছে
$ sudo iptables-restore < ~/before.v4    # আগের অবস্থায় ফেরো
```

### ল্যাব ৩: nftables কনফিগ ফাইল লিখে লোড

১. ৩৯.৬-এর `/etc/nftables.conf` উদাহরণটা কপি করো, `ssh_allowed` set-এ নিজের নেটওয়ার্ক বসাও।
২. `sudo nft -c -f /etc/nftables.conf` — কোনো আউটপুট না এলে সিনট্যাক্স ঠিক।
৩. রোলব্যাক টাইমার: `sudo systemd-run --on-active=3min /usr/sbin/nft flush ruleset`
৪. `sudo nft -f /etc/nftables.conf`
৫. নতুন টার্মিনাল থেকে SSH করো। সফল? তাহলে টাইমার বন্ধ করো (`systemctl list-timers | grep run-` দেখে `systemctl stop`)।
৬. `sudo nft -a list ruleset` দিয়ে handle দেখো, ICMP রুলটা handle দিয়ে মুছে ping টেস্ট করো।

### ল্যাব ৪: firewalld zone ও rich rule (Fedora/Rocky)

```bash
$ sudo firewall-cmd --get-active-zones
$ sudo firewall-cmd --add-service=http --timeout=2m        # ২ মিনিটের জন্য
$ firewall-cmd --list-services                             # http আছে
# ২ মিনিট পর আবার দেখো — http চলে গেছে
$ sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<অন্য VM IP>" port port="8000" protocol="tcp" accept'
$ sudo firewall-cmd --reload
$ python3 -m http.server 8000                               # টেস্ট সার্ভার
# অন্য VM থেকে curl → কাজ করে; তৃতীয় কোনো মেশিন থেকে → reject
```

### ল্যাব ৫ (চ্যালেঞ্জ): দুই VM দিয়ে NAT গেটওয়ে

VM-A-তে দুটো নেটওয়ার্ক কার্ড (একটা NAT/ইন্টারনেট, একটা internal network), VM-B শুধু internal network-এ। VM-A-তে `ip_forward` চালু করে MASQUERADE দাও, VM-B-র default gateway VM-A দাও (`sudo ip route add default via <VM-A internal IP>`)। VM-B থেকে `ping 1.1.1.1` কাজ করলে তুমি একটা রাউটার বানিয়ে ফেলেছ! তারপর VM-A-র ৮০৮০ পোর্ট VM-B-র ৮০-তে DNAT করো।

---

## সারসংক্ষেপ

লিনাক্সের আসল ফায়ারওয়াল কার্নেলের **netfilter**; iptables/nftables তাকে সরাসরি রুল দেয়, ufw/firewalld সহজ ফ্রন্টএন্ড। প্যাকেট **PREROUTING → INPUT/FORWARD → OUTPUT → POSTROUTING** হুক দিয়ে চলে; রুল উপর থেকে নিচে মেলে, প্রথম মিলই জেতে, না মিললে policy। **conntrack**-এর `ESTABLISHED,RELATED` রুল stateful ফায়ারওয়ালের ভিত্তি। রিমোটে সবসময় আগে SSH খোলো এবং রোলব্যাক টাইমার রাখো।

### দ্রুত রেফারেন্স

| কাজ | iptables | nftables | ufw | firewalld |
|---|---|---|---|---|
| রুল দেখা | `iptables -L -n -v --line-numbers` | `nft -a list ruleset` | `ufw status numbered` | `firewall-cmd --list-all` |
| পোর্ট খোলা | `iptables -A INPUT -p tcp --dport 80 -j ACCEPT` | `nft add rule inet f input tcp dport 80 accept` | `ufw allow 80/tcp` | `firewall-cmd --permanent --add-port=80/tcp` |
| সার্ভিস নামে | — | — | `ufw allow 'Nginx Full'` | `firewall-cmd --permanent --add-service=http` |
| IP ব্লক | `iptables -I INPUT -s IP -j DROP` | `nft insert rule inet f input ip saddr IP drop` | `ufw insert 1 deny from IP` | `--add-rich-rule='rule family="ipv4" source address="IP" drop'` |
| রুল মোছা | `iptables -D INPUT 3` | `nft delete rule inet f input handle 5` | `ufw delete 3` | `--remove-port=80/tcp` |
| ডিফল্ট policy | `iptables -P INPUT DROP` | `policy drop;` | `ufw default deny incoming` | zone বেছে নাও (`--set-default-zone`) |
| প্রয়োগ/লোড | তাৎক্ষণিক | `nft -f file` | তাৎক্ষণিক | `firewall-cmd --reload` |
| স্থায়ী করা | `iptables-save` / `netfilter-persistent save` | `systemctl enable nftables` | অটো | `--permanent` / `--runtime-to-permanent` |
| চালু/বন্ধ | — | `systemctl start/stop nftables` | `ufw enable/disable` | `systemctl start/stop firewalld` |
| NAT | `-t nat ... -j MASQUERADE / DNAT` | `masquerade`, `dnat to` | `/etc/ufw/before.rules` | `--add-masquerade`, `--add-forward-port` |

| অন্যান্য দরকারি কমান্ড | কাজ |
|---|---|
| `iptables --version` | ব্যাকএন্ড (nf_tables/legacy) |
| `iptables-apply file` | টাইমআউট-সহ নিরাপদ প্রয়োগ |
| `iptables-translate ...` | iptables → nft অনুবাদ |
| `conntrack -L` / `-C` | কানেকশন টেবিল |
| `ufw app list` / `app info X` | অ্যাপ প্রোফাইল |
| `firewall-cmd --get-active-zones` | কোন ইন্টারফেস কোন zone-এ |
| `firewall-cmd ... --timeout=5m` | সাময়িক রুল |
| `systemd-run --on-active=5min CMD` | লক-আউট রোলব্যাক টাইমার |
| `sysctl -w net.ipv4.ip_forward=1` | রাউটিং চালু |

---

## নিজেকে যাচাই করো

1. ufw, firewalld, iptables, nftables — এদের মধ্যে কোনটা আসলে প্যাকেট ফিল্টার করে? সম্পর্কটা এক লাইনে বলো।
2. একটা প্যাকেট এই মেশিনের nginx-এর জন্য এসেছে। কোন কোন হুক দিয়ে যাবে? আর যদি মেশিনটা রাউটার হয়ে প্যাকেট পার করে দেয়?
3. DROP আর REJECT-এর পার্থক্য কী? ক্লায়েন্টের দিক থেকে কী দেখা যায়?
4. `iptables -A INPUT -j DROP` থাকার পর `iptables -A INPUT -p tcp --dport 80 -j ACCEPT` দিলে পোর্ট ৮০ কাজ করবে না কেন? কীভাবে ঠিক করবে?
5. `ct state established,related accept` রুল না থাকলে কী সমস্যা হবে?
6. firewalld-এ `--permanent` দিয়ে সার্ভিস যোগ করার পরেও পোর্ট খুলছে না কেন? দুটো সমাধান বলো।
7. ufw-এ `ufw status numbered` থেকে নম্বর দিয়ে IPv4 রুল মুছলে IPv6 রুলের কী হয়?
8. Docker কন্টেইনারের পাবলিশ করা পোর্ট ufw-এর deny রুল মানে না কেন?
9. বাইরের 8080 পোর্টকে ভেতরের 192.168.50.10:80-তে পাঠাতে কোন হুকে কোন ধরনের NAT লাগবে? আর কোন kernel সেটিং চালু করতে হবে?
10. রিমোট সার্ভারে নতুন ফায়ারওয়াল রুল প্রয়োগের আগে লক-আউট এড়াতে অন্তত তিনটা কৌশল বলো।

<details><summary>উত্তর দেখো</summary>

1. আসল ফিল্টারিং করে কার্নেলের **netfilter**। iptables ও nftables (`nft`) সরাসরি তাকে রুল দেয়; ufw (ভেতরে iptables) আর firewalld (ভেতরে nftables) হলো সহজ ফ্রন্টএন্ড।
2. লোকাল: **PREROUTING → INPUT** → nginx। রাউটার হিসেবে: **PREROUTING → FORWARD → POSTROUTING**।
3. DROP চুপচাপ ফেলে দেয় — ক্লায়েন্ট timeout পর্যন্ত অপেক্ষা করে। REJECT একটা ICMP error বা TCP RST পাঠায় — ক্লায়েন্ট সাথে সাথে "Connection refused" (বা unreachable) পায়।
4. রুল উপর থেকে নিচে মেলে; `-A` রুলটা DROP-এর পরে বসেছে, তাই প্যাকেট আগেই DROP-এ মিলে যায়। সমাধান: `iptables -I INPUT -p tcp --dport 80 -j ACCEPT` (বা DROP রুলের আগের নম্বরে `-I INPUT n`), অথবা শেষের DROP রুল বাদ দিয়ে policy DROP ব্যবহার।
5. তোমার নিজের শুরু করা কানেকশনের (DNS, apt update, curl) উত্তর প্যাকেট INPUT policy DROP-এ আটকে যাবে; চলমান SSH সেশনও ভেঙে যেতে পারে, কারণ সার্ভারে আসা প্রতিটা প্যাকেট আলাদা রুলে মেলাতে হবে।
6. `--permanent` শুধু ফাইলে লেখে, runtime-এ প্রয়োগ করে না। সমাধান: `firewall-cmd --reload`; অথবা `--permanent` ছাড়া একই কমান্ডটা আবার চালাও (runtime-এর জন্য)। বিকল্প হিসেবে runtime-এ যোগ করে `--runtime-to-permanent`।
7. IPv6 রুলটা থেকে যায় — নম্বর দিয়ে মুছলে শুধু ঐ নির্দিষ্ট রুল মোছে। দুটো একসাথে মুছতে রুলের নাম দিয়ে মোছো: `ufw delete allow 8080/tcp`।
8. Docker nat PREROUTING-এ DNAT করে প্যাকেট কন্টেইনারের দিকে **FORWARD** করে, তাই প্যাকেট INPUT chain-এ যায়ই না, যেখানে ufw-এর রুল থাকে। সমাধান: `-p 127.0.0.1:PORT:PORT`, অথবা `DOCKER-USER` chain-এ রুল।
9. **PREROUTING** হুকে **DNAT** (`--to-destination 192.168.50.10:80` / `dnat to`), সাথে FORWARD chain-এ অনুমতি, আর উত্তরের রাস্তা ঠিক রাখতে দরকার হলে POSTROUTING-এ MASQUERADE। kernel-এ `net.ipv4.ip_forward=1`।
10. যেকোনো তিনটা: (ক) enable/policy DROP-এর আগে SSH allow; (খ) ESTABLISHED,RELATED রুল রাখা; (গ) `systemd-run --on-active=5min` বা `at` দিয়ে রোলব্যাক টাইমার; (ঘ) `iptables-apply`; (ঙ) firewalld-এ runtime/`--timeout`-এ আগে টেস্ট; (চ) পুরনো সেশন খোলা রেখে নতুন সেশনে টেস্ট; (ছ) ক্লাউড/IPMI কনসোল অ্যাক্সেস প্রস্তুত রাখা; (জ) `-F` এর আগে `-P INPUT ACCEPT`।

</details>

---

[← আগের অধ্যায়](38-ssh-remote.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](40-web-server-services.md)

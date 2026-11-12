# অধ্যায় ৩৭: নেটওয়ার্ক কমান্ড ও ট্রাবলশুটিং

> আগের অধ্যায়ে ([অধ্যায় ৩৬](36-networking-basics.md)) আমরা শিখেছি নেটওয়ার্ক *কীভাবে কাজ করে*: IP address, subnet, gateway, DNS, port, TCP/UDP, OSI/TCP-IP মডেল। এই অধ্যায়ে সেই জ্ঞানকে হাতে নেব। এবার টার্মিনালে বসে নেটওয়ার্ক **দেখব, মাপব, পরীক্ষা করব আর ঠিক করব**।

## এই অধ্যায়ে যা শিখবে

- `ip` কমান্ড দিয়ে interface, IP address, route আর ARP/neighbor টেবিল দেখা ও বদলানো (`ip addr`, `ip link`, `ip route`, `ip neigh`)
- পুরনো `ifconfig`, `route`, `netstat`, `arp` কোন নতুন কমান্ডে বদলে গেছে, তার পুরো তুলনা টেবিল
- `hostname` আর `hostnamectl` দিয়ে কম্পিউটারের নাম দেখা ও বদলানো
- `ping` দিয়ে কোনো হোস্ট বেঁচে আছে কি না যাচাই করা, আর ভেতরে ICMP কীভাবে কাজ করে
- `traceroute`, `tracepath` আর `mtr` দিয়ে প্যাকেটের পুরো রাস্তা দেখা, আর TTL-এর কৌশলটা বোঝা
- `ss -tulpn` আর `lsof -i` দিয়ে কোন প্রোগ্রাম কোন port খুলে রেখেছে তা বের করা
- `dig`, `nslookup`, `host` দিয়ে DNS প্রশ্ন করা
- `curl` (`-I -L -o -O -X -H -d -v -w` সহ) আর `wget` দিয়ে HTTP টেস্ট করা আর ফাইল নামানো
- `nc` (netcat) দিয়ে port টেস্ট, ছোট চ্যাট সার্ভার আর ফাইল পাঠানো
- `tcpdump` দিয়ে নেটওয়ার্কের প্যাকেট ধরা ও পড়া
- `nmap` পরিচিতি (শুধু নিজের সিস্টেমে, আইনি সীমার মধ্যে)
- নেটওয়ার্ক সমস্যা ধাপে ধাপে সমাধানের একটা পদ্ধতি, যা পেশাদার সিস্টেম অ্যাডমিনরা ব্যবহার করেন

---

## ৩৭.১ শুরুর আগে: টুলবক্স সাজিয়ে নাও

সব টুল সব distro-তে আগে থেকে ইনস্টল থাকে না। নিচের টেবিল থেকে দেখে নাও কোন প্যাকেজে কোন কমান্ড আছে:

| কমান্ড | Ubuntu/Debian প্যাকেজ | Fedora/RHEL প্যাকেজ |
|---|---|---|
| `ip`, `ss` | `iproute2` (সাধারণত আগে থেকেই থাকে) | `iproute` (আগে থেকেই থাকে) |
| `ifconfig`, `route`, `netstat`, `arp` | `net-tools` | `net-tools` |
| `ping`, `tracepath` | `iputils-ping`, `iputils-tracepath` | `iputils` |
| `traceroute` | `traceroute` | `traceroute` |
| `mtr` | `mtr` (GUI-সহ) বা `mtr-tiny` (শুধু টার্মিনাল) | `mtr` |
| `dig`, `nslookup`, `host` | `dnsutils` বা `bind9-dnsutils` | `bind-utils` |
| `curl` | `curl` | `curl` |
| `wget` | `wget` | `wget` (Fedora 40 থেকে আসলে `wget2`) |
| `nc` | `netcat-openbsd` | `nmap-ncat` (এটা `ncat`, `nc` নামেও চলে) |
| `tcpdump` | `tcpdump` | `tcpdump` |
| `nmap` | `nmap` | `nmap` |
| `lsof` | `lsof` | `lsof` |

একবারে সব ইনস্টল করে নাও:

```bash
# Ubuntu / Debian
$ sudo apt update
$ sudo apt install iproute2 net-tools iputils-ping iputils-tracepath traceroute mtr-tiny \
      dnsutils curl wget netcat-openbsd tcpdump nmap lsof

# Fedora / RHEL 9
$ sudo dnf install iproute net-tools iputils traceroute mtr bind-utils curl wget \
      nmap-ncat tcpdump nmap lsof
```

> 📝 **মনে রাখো:** প্যাকেজ ম্যানেজার নিয়ে বিস্তারিত আছে [অধ্যায় ৩০](30-package-management.md)-এ।

### কোন টুল কোন লেয়ারে কাজ করে

নেটওয়ার্ক সমস্যা ধরতে হলে আগে জানতে হবে কোন টুল কোন "তলায়" কাজ করে। একটা বহুতল ভবন ভাবো: নিচতলায় তার আর সিগন্যাল, ওপরতলায় ওয়েবসাইট। যে তলায় সমস্যা, সেই তলার যন্ত্র দিয়ে মাপতে হয়।

```
 ┌──────────────────────────────────────────────────────────────┐
 │ Application (HTTP, DNS, SSH...)                              │
 │   curl, wget, dig, nslookup, host, nc (টেক্সট পাঠানো)          │
 ├──────────────────────────────────────────────────────────────┤
 │ Transport (TCP / UDP, port)                                  │
 │   ss, netstat, lsof -i, nc -zv, nmap                         │
 ├──────────────────────────────────────────────────────────────┤
 │ Network / Internet (IP, ICMP, routing)                       │
 │   ip addr, ip route, ping, traceroute, tracepath, mtr        │
 ├──────────────────────────────────────────────────────────────┤
 │ Link (Ethernet, Wi-Fi, MAC, ARP)                             │
 │   ip link, ip neigh, ethtool, iw                             │
 └──────────────────────────────────────────────────────────────┘
          tcpdump সব লেয়ারের প্যাকেট একসাথে দেখতে পারে
```

---

## ৩৭.২ `ip` — আধুনিক লিনাক্সের নেটওয়ার্ক "সুইস আর্মি নাইফ"

### `ip` কেন এল? ভেতরে কীভাবে কাজ করে?

আগে লিনাক্সে নেটওয়ার্ক দেখতে `ifconfig`, `route`, `arp` আলাদা আলাদা কমান্ড ছিল (`net-tools` প্যাকেজ)। এগুলো কার্নেলের সাথে পুরনো `ioctl()` ইন্টারফেস দিয়ে কথা বলত। এগুলোর ২০০১ সালের পর থেকে প্রায় কোনো উন্নতি হয়নি, তাই অনেক কিছু দেখাতে পারে না। যেমন একটা interface-এ একাধিক IP থাকলে `ifconfig` ঠিকমতো দেখায় না, আর policy routing বা network namespace-এর মতো নতুন ফিচার সে চেনেই না।

`ip` কমান্ড (প্যাকেজ `iproute2`) কার্নেলের সাথে কথা বলে **Netlink socket** দিয়ে। Netlink হলো user space আর kernel-এর মধ্যে নেটওয়ার্ক তথ্য আদান-প্রদানের আধুনিক চ্যানেল।

```
   ┌───────────────┐                        ┌────────────────────────────┐
   │  ip / ss      │ ── Netlink socket ───► │  Linux Kernel              │
   │ (iproute2)    │ ◄── উত্তর (মেসেজ) ──── │  - interface টেবিল          │
   └───────────────┘                        │  - IP address টেবিল        │
                                            │  - routing টেবিল (FIB)      │
   ┌───────────────┐                        │  - neighbor (ARP) টেবিল     │
   │ ifconfig/route│ ── পুরনো ioctl() ────► │  - socket টেবিল            │
   │ (net-tools)   │                        └────────────────────────────┘
   └───────────────┘
```

> 💡 **উপমা:** কার্নেল হলো একটা ব্যাংক। `ifconfig` পুরনো কাউন্টারে লাইনে দাঁড়িয়ে কাগজে ফর্ম জমা দেয়। `ip` মোবাইল অ্যাপ দিয়ে একই ব্যাংকের সব হিসাব একসাথে, বিস্তারিতভাবে দেখে।

### `ip` কমান্ডের গঠন

```
ip [OPTIONS] OBJECT COMMAND [ARGUMENTS]
```

| OBJECT | সংক্ষেপ | কী দেখায়/বদলায় |
|---|---|---|
| `address` | `addr`, `a` | interface-এর IP address |
| `link` | `l` | interface নিজে (নেটওয়ার্ক কার্ড): up/down, MAC, MTU |
| `route` | `r` | routing টেবিল: কোন প্যাকেট কোন পথে যাবে |
| `neighbour` | `neigh`, `n` | ARP/NDP টেবিল: IP থেকে MAC-এর ম্যাপ |
| `netns` | | network namespace (কন্টেইনারে কাজে লাগে, [অধ্যায় ৪৮](48-containers-virtualization.md)) |
| `maddress` | `maddr` | multicast address |

`ip`-এর গুরুত্বপূর্ণ সাধারণ অপশন:

| অপশন | কাজ |
|---|---|
| `-br` / `-brief` | এক লাইনে সংক্ষেপে দেখায় (সবচেয়ে কাজের) |
| `-c` / `-color` | রঙিন আউটপুট |
| `-4` / `-6` | শুধু IPv4 / শুধু IPv6 |
| `-s` / `-stats` | পরিসংখ্যান (কত প্যাকেট গেল-এল, error) |
| `-d` / `-details` | আরও বিস্তারিত |
| `-j` / `-json` | JSON আউটপুট (স্ক্রিপ্টে কাজে লাগে) |
| `-p` / `-pretty` | JSON সুন্দর করে সাজানো |

> 💡 **টিপস:** `ip`-এর সব কমান্ড সংক্ষেপে লেখা যায়। `ip address show` = `ip addr show` = `ip a`। যেকোনো জায়গায় `help` লিখলে সাহায্য পাবে: `ip addr help`, `ip route help`।

### ৩৭.২.১ `ip addr` — IP address দেখা ও বসানো

```bash
$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 3c:7c:3f:1a:2b:4c brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.105/24 brd 192.168.1.255 scope global dynamic noprefixroute enp3s0
       valid_lft 86142sec preferred_lft 86142sec
    inet6 fe80::3e7c:3fff:fe1a:2b4c/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
3: wlp2s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
    link/ether a4:c3:f0:11:22:33 brd ff:ff:ff:ff:ff:ff
```

আউটপুট লাইন ধরে ধরে বোঝো:

| অংশ | মানে |
|---|---|
| `2: enp3s0` | interface নম্বর ২, নাম `enp3s0` |
| `<...UP,LOWER_UP>` | `UP` = সফটওয়্যারে চালু; `LOWER_UP` = তার লাগানো/সিগন্যাল আছে (physical link) |
| `NO-CARRIER` | তার নেই বা Wi-Fi যুক্ত নয় |
| `mtu 1500` | এক প্যাকেটে সর্বোচ্চ কত byte যাবে |
| `state UP/DOWN` | interface-এর অবস্থা |
| `link/ether 3c:7c:...` | MAC address (হার্ডওয়্যার ঠিকানা) |
| `inet 192.168.1.105/24` | IPv4 address আর subnet (`/24` = `255.255.255.0`) |
| `brd 192.168.1.255` | broadcast address |
| `scope global` | বাইরের নেটওয়ার্কে ব্যবহারযোগ্য; `scope link` = শুধু লোকাল লিংকে; `scope host` = শুধু এই কম্পিউটারে |
| `dynamic` | DHCP থেকে পাওয়া IP |
| `valid_lft 86142sec` | DHCP lease আর কত সেকেন্ড বৈধ |
| `inet6 fe80::...` | IPv6 link-local address |

#### interface-এর নাম এমন অদ্ভুত কেন? (`enp3s0`, `wlp2s0`)

আগে নাম ছিল `eth0`, `wlan0`। কিন্তু একাধিক কার্ড থাকলে বুটের সময় কোনটা আগে চিনবে তার ওপর নাম বদলে যেত। এখন systemd/udev **Predictable Network Interface Names** ব্যবহার করে। নামটা হার্ডওয়্যারের অবস্থান থেকে তৈরি হয়:

| নাম | মানে |
|---|---|
| `en` | Ethernet |
| `wl` | Wireless LAN (Wi-Fi) |
| `ww` | WWAN (মোবাইল ব্রডব্যান্ড) |
| `p3s0` | PCI bus 3, slot 0 |
| `o1` (`eno1`) | মাদারবোর্ডে বসানো (onboard) ডিভাইস নম্বর ১ |
| `x3c7c3f...` (`enx...`) | MAC address থেকে নাম (সাধারণত USB adapter) |

ক্লাউড VM-এ প্রায়ই `eth0` বা `ens5` দেখবে।

#### সংক্ষিপ্ত আর রঙিন ভিউ

প্রতিদিনের কাজে এটাই সবচেয়ে বেশি লাগবে:

```bash
$ ip -br -c addr
lo               UNKNOWN        127.0.0.1/8 ::1/128
enp3s0           UP             192.168.1.105/24 fe80::3e7c:3fff:fe1a:2b4c/64
wlp2s0           DOWN
```

শুধু একটা interface বা শুধু IPv4:

```bash
$ ip addr show dev enp3s0
$ ip -4 addr
$ ip -4 -br addr show enp3s0
enp3s0           UP             192.168.1.105/24
```

#### IP যোগ করা ও মোছা (অস্থায়ী)

```bash
# একটা অতিরিক্ত IP বসাও (একটা interface-এ একাধিক IP থাকতে পারে)
$ sudo ip addr add 192.168.1.200/24 dev enp3s0

# মুছে ফেলো
$ sudo ip addr del 192.168.1.200/24 dev enp3s0

# interface-এর সব IP মুছে ফেলো (সাবধান!)
$ sudo ip addr flush dev enp3s0
```

> ⚠️ **সতর্কতা:** `ip` দিয়ে করা সব পরিবর্তন **অস্থায়ী**। রিবুট করলে বা NetworkManager নতুন DHCP lease নিলে সব মুছে যায়। স্থায়ী কনফিগারেশন করতে হয় distro-র টুল দিয়ে: Ubuntu-তে **Netplan** (`/etc/netplan/*.yaml`), Fedora/RHEL-এ **NetworkManager** (`nmcli`), Debian সার্ভারে `/etc/network/interfaces`। এগুলো [অধ্যায় ৩৬](36-networking-basics.md)-এ আছে।

> ⚠️ **সতর্কতা:** রিমোট সার্ভারে SSH দিয়ে ঢুকে যে interface দিয়ে ঢুকেছ সেটার IP মুছলে বা interface down করলে সংযোগ সঙ্গে সঙ্গে কেটে যাবে, আর ফিরে ঢুকতে পারবে না। তখন cloud console বা সার্ভারের সামনে গিয়ে ঠিক করতে হবে।

### ৩৭.২.২ `ip link` — নেটওয়ার্ক কার্ড নিজেকে নিয়ন্ত্রণ

`ip addr` দেখায় ঠিকানা (লেয়ার ৩), আর `ip link` দেখায় কার্ডের অবস্থা (লেয়ার ২)।

```bash
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 3c:7c:3f:1a:2b:4c brd ff:ff:ff:ff:ff:ff

$ ip -br link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
enp3s0           UP             3c:7c:3f:1a:2b:4c <BROADCAST,MULTICAST,UP,LOWER_UP>
wlp2s0           DOWN           a4:c3:f0:11:22:33 <NO-CARRIER,BROADCAST,MULTICAST,UP>
```

গুরুত্বপূর্ণ কাজগুলো:

```bash
# interface বন্ধ ও চালু
$ sudo ip link set dev enp3s0 down
$ sudo ip link set dev enp3s0 up

# MTU বদলানো
$ sudo ip link set dev enp3s0 mtu 9000

# MAC address বদলানো (আগে down করতে হয়)
$ sudo ip link set dev enp3s0 down
$ sudo ip link set dev enp3s0 address 02:11:22:33:44:55
$ sudo ip link set dev enp3s0 up

# promiscuous mode (সব প্যাকেট গ্রহণ, tcpdump নিজেই চালু করে)
$ sudo ip link set dev enp3s0 promisc on
```

#### পরিসংখ্যান: প্যাকেট হারাচ্ছে কি না

```bash
$ ip -s link show dev enp3s0
2: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 3c:7c:3f:1a:2b:4c brd ff:ff:ff:ff:ff:ff
    RX:  bytes  packets errors dropped  missed   mcast
     987654321   812345      0      12       0    1543
    TX:  bytes  packets errors dropped carrier collsns
     123456789   456789      0       0       0       0
```

`RX` = গ্রহণ (receive), `TX` = পাঠানো (transmit)। `errors` বা `dropped` সংখ্যা যদি দ্রুত বাড়তে থাকে তাহলে তার, পোর্ট, ড্রাইভার বা ডুপ্লেক্স সেটিংয়ে সমস্যা আছে। `-s -s` (দুবার) দিলে আরও বিস্তারিত error ভাগ দেখায়।

> 💡 **টিপস:** তারের স্পিড আর ডুপ্লেক্স দেখতে `sudo ethtool enp3s0` চালাও (`Speed: 1000Mb/s`, `Duplex: Full`, `Link detected: yes`)। Wi-Fi-র সিগন্যাল দেখতে `iw dev wlp2s0 link`।

#### ভার্চুয়াল interface (পরিচিতি)

`ip link` দিয়ে সফটওয়্যারের তৈরি interface-ও বানানো যায়। Docker, VPN আর VM এগুলো নিজে থেকেই বানায়:

```bash
$ sudo ip link add br0 type bridge        # একটা ভার্চুয়াল সুইচ
$ sudo ip link add veth0 type veth peer name veth1   # দুই মাথার ভার্চুয়াল তার
$ sudo ip link delete br0
```

তুমি হয়তো `docker0`, `virbr0`, `tun0`, `wg0`, `vethXXXX` নামের interface দেখবে। এগুলো যথাক্রমে Docker bridge, libvirt bridge, OpenVPN, WireGuard আর কন্টেইনারের veth। বিস্তারিত [অধ্যায় ৪৮](48-containers-virtualization.md)-এ।

### ৩৭.২.৩ `ip route` — প্যাকেট কোন রাস্তায় যাবে

#### ভেতরে কীভাবে কাজ করে

প্রতিটা প্যাকেট পাঠানোর আগে কার্নেল **routing table** দেখে। নিয়মটা হলো **longest prefix match**: যে route গন্তব্যের সাথে সবচেয়ে নির্দিষ্টভাবে মেলে (সবচেয়ে বড় `/` সংখ্যা), সেটাই জেতে। কোনোটা না মিললে `default` route ব্যবহার হয়।

> 💡 **উপমা:** পোস্ট অফিসের কর্মী চিঠির ঠিকানা দেখে। "ঢাকা, মিরপুর-১০" এর জন্য আলাদা বাক্স থাকলে সেখানে রাখে, না থাকলে "ঢাকা" বাক্সে। কিছুই না মিললে "অন্যান্য, কেন্দ্রীয় অফিসে পাঠাও" বাক্সে রাখে। এই শেষ বাক্সটাই `default` route, আর কেন্দ্রীয় অফিস হলো gateway (রাউটার)।

```bash
$ ip route show
default via 192.168.1.1 dev enp3s0 proto dhcp src 192.168.1.105 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
192.168.1.0/24 dev enp3s0 proto kernel scope link src 192.168.1.105 metric 100
```

| অংশ | মানে |
|---|---|
| `default via 192.168.1.1` | অচেনা সব গন্তব্যের জন্য রাউটার `192.168.1.1`-এ পাঠাও |
| `dev enp3s0` | এই interface দিয়ে বের হবে |
| `proto dhcp` | route-টা DHCP থেকে এসেছে; `proto kernel` = IP বসানোর সময় কার্নেল নিজে বানিয়েছে; `proto static` = হাতে বসানো |
| `scope link` | গন্তব্য সরাসরি এই লিংকে আছে, gateway লাগবে না |
| `src 192.168.1.105` | এই route দিয়ে যাওয়ার সময় কোন source IP বসবে |
| `metric 100` | একই গন্তব্যের একাধিক route থাকলে কম metric জেতে (যেমন তার আর Wi-Fi দুটোই থাকলে) |
| `linkdown` | interface-এর লিংক নেই |

#### একটা নির্দিষ্ট গন্তব্যে কোন পথ নেবে?

এটা ট্রাবলশুটিংয়ের সোনার কমান্ড। কার্নেলকে সরাসরি জিজ্ঞেস করো:

```bash
$ ip route get 8.8.8.8
8.8.8.8 via 192.168.1.1 dev enp3s0 src 192.168.1.105 uid 1000
    cache

$ ip route get 192.168.1.50
192.168.1.50 dev enp3s0 src 192.168.1.105 uid 1000
    cache
```

প্রথমটায় `via` আছে, মানে রাউটার হয়ে যাবে। দ্বিতীয়টায় `via` নেই, মানে সরাসরি একই LAN-এ।

#### route যোগ/মোছা (অস্থায়ী)

```bash
# default gateway বসানো
$ sudo ip route add default via 192.168.1.1 dev enp3s0

# একটা নির্দিষ্ট নেটওয়ার্কের জন্য আলাদা রাউটার
$ sudo ip route add 10.10.0.0/16 via 192.168.1.254

# বদলানো (না থাকলে যোগ করে)
$ sudo ip route replace default via 192.168.1.2

# মোছা
$ sudo ip route del 10.10.0.0/16
```

> 📝 **মনে রাখো:** লিনাক্সে একাধিক routing table থাকতে পারে (`ip route show table all`) আর `ip rule` দিয়ে "কোন প্যাকেট কোন টেবিল দেখবে" ঠিক করা যায়। একে **policy routing** বলে। VPN আর multi-ISP সার্ভারে এটা কাজে লাগে। সাধারণ কাজে শুধু `main` টেবিলই দরকার।

### ৩৭.২.৪ `ip neigh` — ARP/Neighbor টেবিল

#### ভেতরে কীভাবে কাজ করে

একই LAN-এ প্যাকেট আসলে পৌঁছায় **MAC address** দিয়ে, IP দিয়ে নয়। তাই `192.168.1.1`-এ পাঠাতে হলে কম্পিউটারকে আগে জানতে হয় ওই IP-র MAC কত। এর জন্য সে পুরো LAN-এ চিৎকার করে জিজ্ঞেস করে (ARP broadcast): *"192.168.1.1 কার? জানাও!"* রাউটার উত্তর দেয়: *"আমার, আমার MAC 08:00:27:aa:bb:cc"*। কার্নেল উত্তরটা কিছুক্ষণ মনে রাখে। এই মনে রাখার খাতাটাই neighbor table। IPv6-এ একই কাজ করে **NDP** (Neighbor Discovery Protocol)।

```
 তোমার PC                                  রাউটার (192.168.1.1)
   │  ARP Request (broadcast ff:ff:ff:ff:ff:ff)  │
   │  "Who has 192.168.1.1? Tell 192.168.1.105"  │
   │ ─────────────────────────────────────────►  │
   │                                             │
   │  ARP Reply (unicast)                        │
   │  "192.168.1.1 is at 08:00:27:aa:bb:cc"      │
   │ ◄─────────────────────────────────────────  │
   │                                             │
   └─► neighbor table-এ লিখে রাখে
```

```bash
$ ip neigh show
192.168.1.1 dev enp3s0 lladdr 08:00:27:aa:bb:cc REACHABLE
192.168.1.20 dev enp3s0 lladdr 5c:e9:1e:44:55:66 STALE
192.168.1.77 dev enp3s0 FAILED
fe80::1 dev enp3s0 lladdr 08:00:27:aa:bb:cc router STALE
```

| অবস্থা (state) | মানে |
|---|---|
| `REACHABLE` | সম্প্রতি নিশ্চিত হয়েছে যে বেঁচে আছে |
| `STALE` | MAC জানা, তবে কিছুক্ষণ যোগাযোগ হয়নি; পরের বার ব্যবহারের সময় আবার যাচাই হবে |
| `DELAY` / `PROBE` | যাচাই চলছে |
| `INCOMPLETE` | ARP প্রশ্ন পাঠানো হয়েছে, উত্তর আসেনি এখনো |
| `FAILED` | উত্তর আসেনি, ওই IP-তে কেউ নেই (বা সাড়া দিচ্ছে না) |
| `PERMANENT` | হাতে বসানো স্থায়ী এন্ট্রি |

```bash
# পুরো টেবিল খালি করা (সমস্যা হলে নতুন করে ARP শিখবে)
$ sudo ip neigh flush dev enp3s0

# স্থায়ী এন্ট্রি বসানো
$ sudo ip neigh add 192.168.1.50 lladdr 00:11:22:33:44:55 dev enp3s0 nud permanent

# মোছা
$ sudo ip neigh del 192.168.1.50 dev enp3s0
```

> 💡 **টিপস:** gateway-কে ping করতে পারছ না? `ip neigh` দেখো। gateway-এর পাশে `FAILED` বা `INCOMPLETE` মানে লেয়ার ২-তেই সমস্যা: তার, সুইচ, Wi-Fi বা ভুল subnet। `REACHABLE` থাকলে লিংক ঠিক আছে, সমস্যা ওপরের কোথাও।

### ৩৭.২.৫ `ip monitor` — লাইভ পরিবর্তন দেখা

```bash
$ ip monitor
[LINK]2: enp3s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default
[ROUTE]Deleted default via 192.168.1.1 dev enp3s0 proto dhcp metric 100
...
```

তার খুলে ফেললে বা Wi-Fi কেটে গেলে সঙ্গে সঙ্গে এখানে দেখা যায়। থামাতে `Ctrl+C`।

---

## ৩৭.৩ পুরনো বনাম নতুন: `ifconfig`, `route`, `netstat`, `arp`

পুরনো টিউটোরিয়াল, Stack Overflow উত্তর আর পুরনো সার্ভারে তুমি `net-tools`-এর কমান্ড প্রচুর দেখবে। এগুলো এখন **deprecated** (পরিত্যক্ত ঘোষিত)। Ubuntu, Fedora, RHEL ৯-এর নতুন ইনস্টলে সাধারণত এগুলো থাকে না। তবে চিনে রাখা জরুরি, কারণ অনেক পুরনো স্ক্রিপ্ট আর বইয়ে এগুলোই আছে।

```bash
$ ifconfig
enp3s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.105  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::3e7c:3fff:fe1a:2b4c  prefixlen 64  scopeid 0x20<link>
        ether 3c:7c:3f:1a:2b:4c  txqueuelen 1000  (Ethernet)
        RX packets 812345  bytes 987654321 (987.6 MB)
        TX packets 456789  bytes 123456789 (123.4 MB)

$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    100    0        0 enp3s0
192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 enp3s0
```

### বদল টেবিল (মুখস্থ রাখার মতো)

| পুরনো (`net-tools`) | নতুন (`iproute2`) | কাজ |
|---|---|---|
| `ifconfig` | `ip addr` / `ip -br a` | IP address দেখা |
| `ifconfig -a` | `ip link` | সব interface (down সহ) |
| `ifconfig eth0 up` | `ip link set eth0 up` | interface চালু |
| `ifconfig eth0 down` | `ip link set eth0 down` | interface বন্ধ |
| `ifconfig eth0 192.168.1.5 netmask 255.255.255.0` | `ip addr add 192.168.1.5/24 dev eth0` | IP বসানো |
| `ifconfig eth0 mtu 9000` | `ip link set eth0 mtu 9000` | MTU |
| `ifconfig eth0 hw ether 02:..` | `ip link set eth0 address 02:..` | MAC বদল |
| `route -n` / `netstat -rn` | `ip route` | routing table |
| `route add default gw 192.168.1.1` | `ip route add default via 192.168.1.1` | default gateway |
| `route del -net 10.0.0.0/8` | `ip route del 10.0.0.0/8` | route মোছা |
| `arp -n` | `ip neigh` | ARP টেবিল |
| `arp -d 192.168.1.5` | `ip neigh del 192.168.1.5 dev eth0` | ARP এন্ট্রি মোছা |
| `netstat -tulpn` | `ss -tulpn` | listening port |
| `netstat -an` | `ss -an` | সব socket |
| `netstat -s` | `ss -s` (সারাংশ) / `nstat` | পরিসংখ্যান |
| `netstat -i` | `ip -s link` | interface পরিসংখ্যান |
| `netstat -g` | `ip maddr` | multicast group |
| `iwconfig` | `iw` | Wi-Fi |
| `nameif` | `ip link set eth0 name lan0` | interface-এর নাম বদল |

> 📝 **মনে রাখো:** `ifconfig` subnet দেখায় `netmask 255.255.255.0` আকারে, আর `ip` দেখায় CIDR আকারে `/24`। দুটো একই জিনিস। হিসাবটা [অধ্যায় ৩৬](36-networking-basics.md)-এ আছে।

---

## ৩৭.৪ `hostname` ও `hostnamectl` — কম্পিউটারের নাম

প্রতিটা কম্পিউটারের একটা নাম থাকে। prompt-এ `nishat@mylaptop:~$`-এর `mylaptop` অংশটা সেই নাম।

```bash
$ hostname
mylaptop

$ hostname -I          # সব IP (loopback বাদে), শুধু সংখ্যা
192.168.1.105 172.17.0.1

$ hostname -f          # FQDN (পূর্ণ ডোমেইন নাম), যদি কনফিগার করা থাকে
mylaptop.home.lan

$ hostname -s          # ছোট নাম (প্রথম ডট পর্যন্ত)
mylaptop
```

| অপশন | কাজ |
|---|---|
| `-I` (বড় হাতের i) | সব interface-এর IP |
| `-i` | hostname resolve করে যে IP পাওয়া যায় (প্রায়ই `127.0.1.1`, তাই বিভ্রান্তিকর) |
| `-f` | Fully Qualified Domain Name |
| `-s` | short name |
| `-d` | শুধু domain অংশ |

### নাম স্থায়ীভাবে বদলানো

আধুনিক systemd-ভিত্তিক সব distro-তে:

```bash
$ hostnamectl
 Static hostname: mylaptop
       Icon name: computer-laptop
         Chassis: laptop
      Machine ID: 4f2a...
         Boot ID: 9c1e...
Operating System: Ubuntu 24.04.1 LTS
          Kernel: Linux 6.8.0-45-generic
    Architecture: x86-64

$ sudo hostnamectl set-hostname webserver01
$ hostname
webserver01
```

`hostnamectl set-hostname` নামটা `/etc/hostname` ফাইলে লেখে, তাই রিবুটের পরেও থাকে। (`sudo hostname newname` শুধু অস্থায়ীভাবে বদলায়।)

> ⚠️ **সতর্কতা:** নাম বদলানোর পর `/etc/hosts` ফাইলেও পুরনো নাম বদলে দাও (Debian/Ubuntu-তে সাধারণত `127.0.1.1  mylaptop` লাইন থাকে)। না দিলে `sudo` চালানোর সময় `unable to resolve host webserver01` সতর্কবার্তা আসবে।

```bash
$ cat /etc/hosts
127.0.0.1   localhost
127.0.1.1   webserver01
::1         localhost ip6-localhost ip6-loopback
```

> 📝 **মনে রাখো:** `/etc/hosts` হলো ছোট্ট লোকাল ফোনবুক। নাম থেকে IP খোঁজার সময় সাধারণত এটা DNS-এর **আগে** দেখা হয় (ক্রম ঠিক করে `/etc/nsswitch.conf`-এর `hosts:` লাইন)। টেস্টের জন্য এখানে লিখে দিলে কোনো ডোমেইনকে নিজের পছন্দের IP-তে পাঠানো যায়।

---

## ৩৭.৫ `ping` — "তুমি কি বেঁচে আছ?"

### ভেতরে কীভাবে কাজ করে

`ping` **ICMP** (Internet Control Message Protocol) ব্যবহার করে। সে গন্তব্যে একটা **Echo Request** প্যাকেট পাঠায়, আর গন্তব্য (যদি বেঁচে থাকে আর firewall আটকে না রাখে) একটা **Echo Reply** ফেরত পাঠায়। যাওয়া-আসার সময়টাকে বলে **RTT** (Round Trip Time)।

> 💡 **উপমা:** সাবমেরিনের সোনার। শব্দ পাঠাও, প্রতিধ্বনি ফিরে এলে বুঝবে সামনে কিছু আছে, আর কত সময়ে ফিরল তা দিয়ে দূরত্ব আন্দাজ করবে।

```
 তোমার PC                                   8.8.8.8
   │ ICMP Echo Request (type 8) seq=1 ──────────► │
   │ ◄────────── ICMP Echo Reply (type 0) seq=1  │
   │        RTT = পাঠানো থেকে উত্তর পাওয়া পর্যন্ত সময়   │
```

### সিনট্যাক্স ও ব্যবহার

```
ping [OPTIONS] HOST
```

লিনাক্সে `ping` থামানো পর্যন্ত চলতেই থাকে (Windows-এর মতো ৪টায় থামে না)। থামাতে `Ctrl+C`।

```bash
$ ping google.com
PING google.com (142.250.193.110) 56(84) bytes of data.
64 bytes from del12s08-in-f14.1e100.net (142.250.193.110): icmp_seq=1 ttl=117 time=24.6 ms
64 bytes from del12s08-in-f14.1e100.net (142.250.193.110): icmp_seq=2 ttl=117 time=23.9 ms
64 bytes from del12s08-in-f14.1e100.net (142.250.193.110): icmp_seq=3 ttl=117 time=25.1 ms
^C
--- google.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 23.912/24.533/25.104/0.487 ms
```

আউটপুট পড়তে শেখো:

| অংশ | মানে |
|---|---|
| `(142.250.193.110)` | DNS থেকে পাওয়া IP। নাম থেকে IP পেলে বোঝা যায় DNS কাজ করছে |
| `56(84) bytes` | ৫৬ byte ডেটা; IP header (২০) আর ICMP header (৮) যোগ করে ৮৪ |
| `icmp_seq=1` | ক্রমিক নম্বর। মাঝে নম্বর বাদ পড়লে প্যাকেট হারিয়েছে |
| `ttl=117` | ফেরত প্যাকেটের অবশিষ্ট TTL |
| `time=24.6 ms` | RTT |
| `0% packet loss` | কোনো প্যাকেট হারায়নি |
| `mdev` | সময়ের ওঠানামা (jitter-এর মতো); কম হলে সংযোগ স্থির |

### গুরুত্বপূর্ণ অপশন

| অপশন | কাজ | উদাহরণ |
|---|---|---|
| `-c N` | N টা প্যাকেট পাঠিয়ে থামবে | `ping -c 4 8.8.8.8` |
| `-i SEC` | দুই প্যাকেটের মাঝে বিরতি (০.২ সেকেন্ডের কম দিতে root লাগে) | `ping -i 0.5 host` |
| `-W SEC` | প্রতিটা উত্তরের জন্য কতক্ষণ অপেক্ষা | `ping -c1 -W2 host` |
| `-w SEC` | পুরো ping মোট কত সেকেন্ড চলবে (deadline) | `ping -w 10 host` |
| `-s BYTES` | প্যাকেটের ডেটা সাইজ | `ping -s 1472 host` |
| `-t TTL` | পাঠানো প্যাকেটের TTL | `ping -t 5 host` |
| `-4` / `-6` | জোর করে IPv4 / IPv6 | `ping -6 google.com` |
| `-q` | শুধু শেষের সারাংশ | `ping -c 10 -q host` |
| `-n` | IP-কে নামে উল্টো রূপান্তর (reverse DNS) করবে না | `ping -n 8.8.8.8` |
| `-I IFACE` | নির্দিষ্ট interface দিয়ে পাঠাও | `ping -I wlp2s0 8.8.8.8` |
| `-M do` | fragment করতে দেবে না (MTU পরীক্ষা) | `ping -M do -s 1472 host` |
| `-D` | প্রতিটা লাইনের আগে timestamp | `ping -D host` |
| `-a` | উত্তর এলে বিপ শব্দ | `ping -a host` |
| `-f` | flood ping (root লাগে, শুধু নিজের নেটওয়ার্কে টেস্টে) | `sudo ping -f -c 1000 192.168.1.1` |

### সাধারণ ফলাফল ও তার মানে

```bash
$ ping -c 2 192.168.1.77
PING 192.168.1.77 (192.168.1.77) 56(84) bytes of data.
From 192.168.1.105 icmp_seq=1 Destination Host Unreachable
From 192.168.1.105 icmp_seq=2 Destination Host Unreachable

--- 192.168.1.77 ping statistics ---
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 1024ms
```

| ফলাফল | সম্ভাব্য কারণ |
|---|---|
| `Destination Host Unreachable` (নিজের IP থেকে) | একই LAN-এ ওই IP-র কাছ থেকে ARP উত্তর আসেনি; হোস্ট বন্ধ বা নেই |
| কিছুই আসে না, শুধু শেষে `100% packet loss` | firewall ICMP আটকাচ্ছে, বা পথে কোথাও প্যাকেট হারাচ্ছে |
| `Network is unreachable` | কোনো route নেই (default gateway নেই?) → `ip route` দেখো |
| `ping: google.com: Temporary failure in name resolution` | DNS কাজ করছে না → IP দিয়ে ping করে দেখো |
| `ping: google.com: Name or service not known` | নামটাই পাওয়া যায়নি (বানান ভুল বা DNS-এ নেই) |
| `Request timeout` / বেশি `time` আর `packet loss` | নেটওয়ার্ক ব্যস্ত বা দুর্বল সিগন্যাল |

> ⚠️ **সতর্কতা:** ping-এর উত্তর না আসা মানেই হোস্ট মৃত নয়। অনেক সার্ভার আর ক্লাউড (যেমন AWS-এর ডিফল্ট security group) ICMP আটকে রাখে। তখন `nc -zv host 443` বা `curl` দিয়ে আসল সার্ভিসটা পরীক্ষা করো।

> 💡 **টিপস:** ping চলার সময় `Ctrl+\` চাপলে ping না থামিয়েই এখন পর্যন্ত পরিসংখ্যান দেখায়।

> 📝 **মনে রাখো:** আধুনিক লিনাক্সে সাধারণ ইউজার `ping` চালাতে পারে, কারণ কার্নেল `net.ipv4.ping_group_range` দিয়ে unprivileged ICMP socket অনুমতি দেয় (বা পুরনো সিস্টেমে `ping`-এ `cap_net_raw` capability দেওয়া থাকে)। capability নিয়ে আছে [অধ্যায় ১৭](17-special-permissions-acl.md) আর [অধ্যায় ৪৫](45-security-hardening.md)-এ।

### MTU সমস্যা ধরার কৌশল

Ethernet-এর MTU ১৫০০। ICMP+IP header ২৮ byte, তাই সর্বোচ্চ ডেটা ১৫০০ − ২৮ = ১৪৭২।

```bash
$ ping -c 2 -M do -s 1472 8.8.8.8      # কাজ করা উচিত
$ ping -c 2 -M do -s 1473 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 1473(1501) bytes of data.
ping: local error: message too long, mtu=1500
```

VPN বা PPPoE সংযোগে MTU কম থাকে (যেমন ১৪৯২ বা আরও কম)। তখন "কিছু ওয়েবসাইট খোলে, কিছু আটকে থাকে"-জাতীয় অদ্ভুত সমস্যা হয়। এই কৌশলে সঠিক MTU বের করা যায়।

---

## ৩৭.৬ `traceroute`, `tracepath` ও `mtr` — রাস্তার প্রতিটা মোড় দেখা

### TTL-এর চতুর কৌশল

প্রতিটা IP প্যাকেটে একটা **TTL** (Time To Live) সংখ্যা থাকে। প্রতিটা রাউটার প্যাকেটটা পার করার সময় TTL থেকে ১ কমায়। TTL ০ হয়ে গেলে রাউটার প্যাকেটটা ফেলে দেয় আর প্রেরককে একটা ICMP **"Time Exceeded"** বার্তা পাঠায়, যাতে রাউটারের নিজের IP থাকে। (এই ব্যবস্থা আছে যাতে ভুল route-এ প্যাকেট অনন্তকাল ঘুরতে না থাকে।)

`traceroute` এই নিয়মের সুবিধা নেয়:

```
  TTL=1 পাঠাও → ১ম রাউটার TTL শূন্য করে → "Time Exceeded" ফেরত → ১ম hop জানা গেল
  TTL=2 পাঠাও → ১ম পার, ২য় রাউটারে শূন্য → "Time Exceeded" → ২য় hop জানা গেল
  TTL=3 ...
  ... গন্তব্যে পৌঁছালে সে ভিন্ন উত্তর দেয় (Port Unreachable / Echo Reply / SYN-ACK)

  তোমার PC ──► [রাউটার ১] ──► [ISP রাউটার] ──► [ব্যাকবোন] ──► ... ──► গন্তব্য
               TTL 1-এ মরে      TTL 2-এ মরে      TTL 3-এ মরে
```

> 💡 **উপমা:** একটা বল ছুঁড়ছ যেটা ১ বার হাত বদলের পর থেমে যায়। যার হাতে থামল, সে চিৎকার করে নিজের নাম বলে। এরপর ২ বার হাত বদলের বল ছোঁড়ো... এভাবে লাইনের প্রত্যেকের নাম জেনে যাও।

### `traceroute`

```bash
$ traceroute google.com
traceroute to google.com (142.250.193.110), 30 hops max, 60 byte packets
 1  _gateway (192.168.1.1)  1.123 ms  1.045 ms  0.998 ms
 2  10.20.0.1 (10.20.0.1)  4.512 ms  4.498 ms  4.470 ms
 3  103.12.44.1 (103.12.44.1)  6.210 ms  6.188 ms  6.150 ms
 4  * * *
 5  72.14.218.10 (72.14.218.10)  22.877 ms  22.840 ms  22.812 ms
 6  del12s08-in-f14.1e100.net (142.250.193.110)  24.301 ms  24.255 ms  24.230 ms
```

- প্রতিটা লাইন একটা **hop** (রাউটার)। প্রতিটায় ৩টা প্রোব পাঠানো হয়, তাই ৩টা সময়।
- `* * *` মানে ওই রাউটার উত্তর দেয়নি। এটা সবসময় সমস্যা নয়। অনেক রাউটার "Time Exceeded" পাঠায় না বা কম অগ্রাধিকার দেয়। পরের hop-গুলো ঠিক থাকলে চিন্তা নেই।
- কোনো hop থেকে শুরু করে **শেষ পর্যন্ত** সব `* * *` হলে সমস্যা সম্ভবত সেখান থেকেই।

লিনাক্সের `traceroute` ডিফল্টভাবে **UDP** প্যাকেট পাঠায় (port 33434 থেকে শুরু)। কিছু firewall UDP আটকায়, তাই অন্য পদ্ধতিও আছে:

| অপশন | কাজ |
|---|---|
| `-n` | DNS নাম খুঁজবে না (অনেক দ্রুত) |
| `-I` | UDP-র বদলে ICMP Echo (Windows-এর `tracert`-এর মতো; root লাগতে পারে) |
| `-T` | TCP SYN প্রোব (firewall পার হতে সবচেয়ে ভালো; root লাগে) |
| `-p PORT` | port নির্ধারণ (যেমন `-T -p 443`) |
| `-m N` | সর্বোচ্চ hop (ডিফল্ট ৩০) |
| `-q N` | প্রতি hop-এ কতটা প্রোব (ডিফল্ট ৩) |
| `-w SEC` | উত্তরের অপেক্ষা |
| `-4` / `-6` | IPv4 / IPv6 |

```bash
$ sudo traceroute -n -T -p 443 example.com
```

### `tracepath` — root ছাড়া, MTU সহ

`tracepath` প্রায় `traceroute`-এর মতো, কিন্তু root লাগে না এবং পথের **PMTU** (Path MTU) দেখায়।

```bash
$ tracepath -n 8.8.8.8
 1?: [LOCALHOST]                      pmtu 1500
 1:  192.168.1.1                                           1.201ms
 1:  192.168.1.1                                           1.087ms
 2:  10.20.0.1                                             4.660ms
 3:  103.12.44.1                                           6.302ms asymm  4
 4:  no reply
 5:  8.8.8.8                                              21.904ms reached
     Resume: pmtu 1500 hops 5 back 5
```

`asymm 4` মানে ফেরার পথে hop সংখ্যা ভিন্ন (যাওয়া-আসার রাস্তা আলাদা)।

### `mtr` — `ping` + `traceroute` একসাথে, লাইভ

`mtr` (My Traceroute) একটানা প্রতিটা hop-কে ping করতে থাকে আর প্রতিটার packet loss ও সময় লাইভ দেখায়। ISP-কে "আপনার নেটওয়ার্কের কোথায় প্যাকেট হারাচ্ছে" প্রমাণ দেখাতে এটাই সবচেয়ে কাজের।

```bash
$ mtr google.com            # ইন্টারঅ্যাকটিভ, লাইভ আপডেট (q চেপে বের হও)

$ mtr -rwn -c 100 8.8.8.8   # ১০০টা প্রোব পাঠিয়ে রিপোর্ট
Start: 2026-11-12T10:15:02+0600
HOST: mylaptop                    Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 192.168.1.1                0.0%   100    1.1   1.2   0.9   3.4   0.3
  2.|-- 10.20.0.1                  0.0%   100    4.5   4.8   4.1  12.0   0.9
  3.|-- 103.12.44.1               35.0%   100    6.3   6.4   6.0   9.8   0.5
  4.|-- 72.14.218.10               0.0%   100   22.9  23.0  22.5  27.1   0.6
  5.|-- 8.8.8.8                    0.0%   100   21.9  22.1  21.6  25.0   0.4
```

| কলাম | মানে |
|---|---|
| `Loss%` | এই hop-এ কত শতাংশ প্রোবের উত্তর আসেনি |
| `Snt` | কতটা পাঠানো হয়েছে |
| `Last/Avg/Best/Wrst` | শেষ/গড়/সেরা/সবচেয়ে খারাপ RTT (ms) |
| `StDev` | ওঠানামা |

> 📝 **মনে রাখো (খুব গুরুত্বপূর্ণ):** ওপরে hop ৩-এ `35% Loss`, কিন্তু hop ৪ আর ৫-এ `0%`। তার মানে **আসল ক্ষতি নেই**। hop ৩-এর রাউটার শুধু নিজের কাছে আসা প্রোবের উত্তর দিতে কৃপণতা করছে (ICMP rate-limiting), কিন্তু অন্যদের ট্রাফিক ঠিকঠাক পার করছে। **আসল packet loss তখনই, যখন loss একটা hop থেকে শুরু হয়ে শেষ পর্যন্ত সব hop-এ থাকে।**

| `mtr` অপশন | কাজ |
|---|---|
| `-r` / `--report` | রিপোর্ট মোড (শেষে একবার প্রিন্ট) |
| `-w` / `--report-wide` | পূর্ণ hostname সহ চওড়া রিপোর্ট |
| `-c N` | কতটা প্রোব |
| `-n` | DNS বন্ধ |
| `-b` | নাম ও IP দুটোই দেখাও |
| `-T` / `-u` | TCP / UDP প্রোব (ডিফল্ট ICMP) |
| `-P PORT` | TCP/UDP port |
| `-4` / `-6` | IPv4 / IPv6 |

#### `mtr`-এর ইন্টারঅ্যাকটিভ কীবোর্ড শর্টকাট

| কী | কাজ |
|---|---|
| `q` | বের হও |
| `p` | থামাও (pause) |
| `Space` | আবার চালাও (resume) |
| `r` | পরিসংখ্যান রিসেট |
| `n` | DNS নাম দেখানো চালু/বন্ধ |
| `d` | ডিসপ্লে মোড বদলাও (টেবিল/গ্রাফ) |
| `j` | latency আর jitter কলামের মধ্যে বদলাও |
| `o` | কলামের ক্রম বদলাও |
| `h` / `?` | সাহায্য |

---

## ৩৭.৭ `ss` — কোন port খোলা, কে কার সাথে কথা বলছে

### socket কী? ভেতরে কীভাবে কাজ করে

প্রোগ্রাম নেটওয়ার্কে কথা বলে **socket** দিয়ে। socket হলো কার্নেলের ভেতরে একটা যোগাযোগের "প্লাগ পয়েন্ট"। প্রতিটা TCP সংযোগ চারটা জিনিস দিয়ে আলাদা করে চেনা যায়: **(local IP, local port, remote IP, remote port)**।

একটা ওয়েব সার্ভার port 80-এ **listen** করে, মানে দরজা খুলে বসে থাকে। কেউ সংযোগ করলে কার্নেল একটা নতুন **ESTABLISHED** socket তৈরি করে।

> 💡 **উপমা:** একটা অফিসের অভ্যর্থনা ডেস্ক (LISTEN socket) সবসময় খোলা থাকে। অতিথি এলে অভ্যর্থনাকারী তাকে একটা আলাদা মিটিং রুমে (ESTABLISHED socket) বসিয়ে দেয়, আর নিজে আবার পরের অতিথির অপেক্ষায় থাকে।

`ss` (socket statistics) কার্নেলের socket টেবিল সরাসরি Netlink দিয়ে পড়ে। পুরনো `netstat` `/proc/net/tcp` ফাইল পার্স করত, তাই হাজার হাজার সংযোগ থাকলে খুব ধীর হতো। `ss` সেখানে অনেক দ্রুত।

### সবচেয়ে বিখ্যাত কমান্ড: `ss -tulpn`

```bash
$ sudo ss -tulpn
Netid State  Recv-Q Send-Q  Local Address:Port   Peer Address:Port Process
udp   UNCONN 0      0       127.0.0.53%lo:53          0.0.0.0:*     users:(("systemd-resolve",pid=612,fd=13))
udp   UNCONN 0      0             0.0.0.0:68          0.0.0.0:*     users:(("NetworkManager",pid=801,fd=27))
tcp   LISTEN 0      4096    127.0.0.53%lo:53          0.0.0.0:*     users:(("systemd-resolve",pid=612,fd=14))
tcp   LISTEN 0      128           0.0.0.0:22          0.0.0.0:*     users:(("sshd",pid=1023,fd=3))
tcp   LISTEN 0      511           0.0.0.0:80          0.0.0.0:*     users:(("nginx",pid=1240,fd=6))
tcp   LISTEN 0      151         127.0.0.1:3306        0.0.0.0:*     users:(("mysqld",pid=1310,fd=21))
tcp   LISTEN 0      128              [::]:22             [::]:*     users:(("sshd",pid=1023,fd=4))
```

অপশনগুলো মনে রাখার সহজ উপায় **"টিউলিপ-এন"** (tulpn):

| অপশন | মানে |
|---|---|
| `-t` | TCP socket |
| `-u` | UDP socket |
| `-l` | শুধু listening (যারা অপেক্ষা করছে) |
| `-p` | কোন process (নাম, PID)। অন্য ইউজারের process দেখতে `sudo` লাগে |
| `-n` | numeric: port নম্বর আর IP দেখাও, নাম নয় (`:22`, `:ssh` নয়) |

**Local Address কলাম পড়তে শেখো (নিরাপত্তার জন্য খুব জরুরি):**

| Local Address | মানে |
|---|---|
| `0.0.0.0:80` | সব IPv4 interface-এ খোলা, বাইরের যে কেউ সংযোগ করতে পারবে (firewall না থাকলে) |
| `[::]:22` বা `*:22` | সব IPv6 (আর প্রায়ই IPv4-ও) interface-এ খোলা |
| `127.0.0.1:3306` | শুধু এই কম্পিউটার থেকেই সংযোগ করা যাবে। ডেটাবেসের জন্য এটাই নিরাপদ |
| `127.0.0.53%lo:53` | শুধু loopback-এ (systemd-resolved-এর DNS stub) |
| `192.168.1.105:8080` | শুধু ওই নির্দিষ্ট IP-তে |

> ⚠️ **সতর্কতা:** ডেটাবেস (MySQL 3306, PostgreSQL 5432, Redis 6379, MongoDB 27017) যদি `0.0.0.0`-এ listen করে আর firewall না থাকে, তাহলে পুরো ইন্টারনেট সেটায় ঢোকার চেষ্টা করতে পারবে। সার্ভার সেটআপের পর প্রথম কাজ `sudo ss -tulpn` দেখা। firewall নিয়ে [অধ্যায় ৩৯](39-firewall.md), নিরাপত্তা নিয়ে [অধ্যায় ৪৫](45-security-hardening.md)।

`Recv-Q`/`Send-Q`: LISTEN socket-এ `Send-Q` হলো backlog-এর সর্বোচ্চ সীমা, আর `Recv-Q` হলো এখন কতটা সংযোগ accept হওয়ার অপেক্ষায়। ESTABLISHED socket-এ এগুলো হলো এখনো না পড়া/না পাঠানো byte। এগুলো বারবার বড় দেখালে অ্যাপ্লিকেশন বা নেটওয়ার্ক ধীর।

### `ss`-এর অন্য অপশন

| অপশন | কাজ |
|---|---|
| `-a` | সব socket (listening + সংযুক্ত) |
| `-x` | Unix domain socket (একই মেশিনের প্রোগ্রামের মধ্যে যোগাযোগ) |
| `-w` | raw socket |
| `-4` / `-6` | শুধু IPv4 / IPv6 |
| `-s` | সারাংশ পরিসংখ্যান |
| `-e` | বর্ধিত তথ্য (uid, inode) |
| `-o` | টাইমার (keepalive, retransmit) |
| `-i` | TCP-র ভেতরের তথ্য (rtt, cwnd, retrans) |
| `-m` | socket-এর মেমরি ব্যবহার |
| `-K` | মেলানো socket জোর করে বন্ধ (root, কার্নেল সাপোর্ট লাগে) |
| `-H` | হেডার লাইন বাদ (স্ক্রিপ্টের জন্য) |

```bash
$ ss -s
Total: 812
TCP:   34 (estab 12, closed 6, orphaned 0, timewait 5)

Transport Total     IP        IPv6
RAW       1         0         1
UDP       9         6         3
TCP       28        21        7
INET      38        27        11
FRAG      0         0         0
```

### ফিল্টার: `ss`-এর আসল শক্তি

```bash
# শুধু ESTABLISHED TCP সংযোগ
$ ss -tn state established
Recv-Q Send-Q   Local Address:Port      Peer Address:Port
0      0        192.168.1.105:51432    142.250.193.110:443
0      0        192.168.1.105:22         192.168.1.20:60122

# কে আমার port 22-তে সংযুক্ত?
$ ss -tn state established '( sport = :22 )'

# আমি কোন কোন সার্ভারের 443 port-এ সংযুক্ত?
$ ss -tn '( dport = :443 )'

# port 8080-এ কে listen করছে?
$ sudo ss -tlnp 'sport = :8080'

# একটা নির্দিষ্ট IP-র সাথে সব সংযোগ
$ ss -tn dst 192.168.1.20

# TIME-WAIT অবস্থার সংযোগ গোনা
$ ss -tan state time-wait | wc -l
```

TCP-র অবস্থা (state) নাম: `established`, `syn-sent`, `syn-recv`, `fin-wait-1`, `fin-wait-2`, `time-wait`, `closed`, `close-wait`, `last-ack`, `listening`, `closing`। এছাড়া কিছু দলগত নামও আছে: `connected` (listening আর closed বাদে সব), `synchronized`, `bucket`, `big`।

> 💡 **টিপস:** অনেক `CLOSE-WAIT` জমে থাকলে বুঝবে অ্যাপ্লিকেশন নিজের দিক থেকে সংযোগ বন্ধ করতে ভুলে যাচ্ছে (কোডে bug)। অনেক `SYN-RECV` থাকলে সম্ভবত SYN flood আক্রমণ চলছে।

### `netstat` (পুরনো) — চিনে রাখো

```bash
$ sudo netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1023/sshd
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1240/nginx: master
```

অপশনগুলো `ss`-এর মতোই। এছাড়া `netstat -r` (route), `netstat -i` (interface), `netstat -s` (প্রোটোকল পরিসংখ্যান)।

---

## ৩৭.৮ `lsof -i` — নেটওয়ার্ক ব্যবহারকারী ফাইল খোঁজা

লিনাক্সে "সবকিছুই ফাইল"। তাই একটা নেটওয়ার্ক socket-ও প্রসেসের একটা খোলা file descriptor। `lsof` (LiSt Open Files) সব খোলা ফাইলের তালিকা দেয়, আর `-i` দিলে শুধু নেটওয়ার্কেরগুলো। ([অধ্যায় ২৪](24-process-management.md)-এ process আর file descriptor নিয়ে আরও আছে।)

```bash
$ sudo lsof -i :80
COMMAND  PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx   1240     root    6u  IPv4  23456      0t0  TCP *:http (LISTEN)
nginx   1241 www-data    6u  IPv4  23456      0t0  TCP *:http (LISTEN)
```

| কমান্ড | কাজ |
|---|---|
| `lsof -i` | সব নেটওয়ার্ক সংযোগ |
| `lsof -i :22` | port 22 ব্যবহার করছে যারা |
| `lsof -i tcp` / `lsof -i udp` | শুধু TCP / UDP |
| `lsof -i tcp:443` | TCP port 443 |
| `lsof -i @192.168.1.20` | এই হোস্টের সাথে সংযোগ |
| `lsof -i -n -P` | `-n` = IP নাম বানাবে না, `-P` = port নাম বানাবে না (দ্রুত) |
| `lsof -iTCP -sTCP:LISTEN` | শুধু listening TCP |
| `lsof -i -a -p 1240` | PID 1240-এর নেটওয়ার্ক সংযোগ (`-a` = AND) |
| `lsof -i -a -u www-data` | একজন ইউজারের নেটওয়ার্ক সংযোগ |
| `lsof -t -i :8080` | শুধু PID প্রিন্ট (`-t` = terse) |

বাস্তব উদাহরণ: "port 3000 already in use" এরর এল। কে ধরে রেখেছে, আর তাকে বন্ধ করো:

```bash
$ lsof -t -i :3000
48211
$ kill 48211          # বা এক লাইনে: kill $(lsof -t -i :3000)
```

> ⚠️ **সতর্কতা:** `sudo` ছাড়া `lsof` শুধু তোমার নিজের প্রসেস দেখায়। ফলে "কিছুই নেই" দেখালেও আসলে অন্য ইউজারের প্রসেস port দখল করে থাকতে পারে।

> 💡 **টিপস:** আরেকটা ছোট টুল `fuser`: `sudo fuser 80/tcp` port 80 ব্যবহারকারী PID দেখায়, আর `sudo fuser -k 80/tcp` তাদের মেরে ফেলে (সাবধানে!)।

---

## ৩৭.৯ DNS টুল: `dig`, `nslookup`, `host`

DNS হলো ইন্টারনেটের ফোনবুক: `google.com` নাম থেকে IP বের করে। কীভাবে কাজ করে (resolver, root, TLD, authoritative server, record-এর ধরন) তা [অধ্যায় ৩৬](36-networking-basics.md)-এ আছে। এখানে আমরা প্রশ্ন করব।

### আগে জানো: তোমার সিস্টেম কোন DNS সার্ভারকে জিজ্ঞেস করে

```bash
$ cat /etc/resolv.conf
# This is /run/systemd/resolve/stub-resolv.conf managed by man:systemd-resolved(8).
nameserver 127.0.0.53
options edns0 trust-ad
search home.lan
```

Ubuntu আর Fedora-তে **systemd-resolved** চলে। সে `127.0.0.53`-এ একটা লোকাল DNS stub চালায়, যা উত্তর cache করে আর আসল DNS সার্ভারে প্রশ্ন পাঠায়। আসল সার্ভার দেখতে:

```bash
$ resolvectl status
Global
       Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub

Link 2 (enp3s0)
    Current Scopes: DNS
  Current DNS Server: 192.168.1.1
         DNS Servers: 192.168.1.1 8.8.8.8
          DNS Domain: home.lan
```

```
 প্রোগ্রাম (curl, firefox)
        │ getaddrinfo("google.com")
        ▼
 glibc → /etc/nsswitch.conf ("hosts: files ... dns")
        │ ১) /etc/hosts দেখে
        │ ২) না পেলে /etc/resolv.conf-এর nameserver-কে জিজ্ঞেস করে
        ▼
 systemd-resolved (127.0.0.53)  ── cache-এ থাকলে সাথে সাথে উত্তর
        │
        ▼
 রাউটার / ISP / 8.8.8.8 (recursive resolver)
        │
        ▼
 root (.) → TLD (.com) → authoritative (google.com-এর নিজের সার্ভার)
```

```bash
$ resolvectl query google.com          # resolved দিয়ে প্রশ্ন
$ resolvectl flush-caches              # DNS cache খালি করো
$ resolvectl statistics                # cache hit/miss
```

> 📝 **মনে রাখো:** `dig`, `nslookup` আর `host` সরাসরি DNS সার্ভারকে প্রশ্ন করে, `/etc/hosts` দেখে না। আর সাধারণ প্রোগ্রাম (`ping`, `curl`) glibc হয়ে যায়, তাই `/etc/hosts`-ও দেখে। "dig ঠিক উত্তর দিচ্ছে কিন্তু ব্রাউজার ভুল জায়গায় যাচ্ছে" হলে `/etc/hosts` দেখো। glibc-র মতো করে নাম খুঁজতে: `getent hosts google.com`।

### `dig` — পেশাদারের DNS টুল

```bash
$ dig google.com

; <<>> DiG 9.18.28-0ubuntu0.24.04.1-Ubuntu <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31415
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             187     IN      A       142.250.193.110

;; Query time: 3 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Thu Nov 12 10:20:11 +06 2026
;; MSG SIZE  rcvd: 55
```

| অংশ | মানে |
|---|---|
| `status: NOERROR` | সফল। `NXDOMAIN` = ডোমেইনই নেই; `SERVFAIL` = সার্ভার উত্তর দিতে পারেনি (প্রায়ই DNSSEC বা authoritative সার্ভারের সমস্যা); `REFUSED` = সার্ভার প্রশ্ন নিতে রাজি নয় |
| `flags: qr rd ra` | `qr` = এটা উত্তর; `rd` = recursion চাওয়া হয়েছে; `ra` = recursion পাওয়া যায়; `aa` = authoritative উত্তর |
| `QUESTION SECTION` | কী জিজ্ঞেস করা হয়েছিল (`A` record) |
| `ANSWER SECTION` | উত্তর: নাম, **TTL (187 সেকেন্ড, এতক্ষণ cache থাকবে)**, class `IN`, type `A`, মান |
| `Query time` | কত সময় লাগল (cache থেকে এলে খুব কম) |
| `SERVER` | কোন DNS সার্ভার উত্তর দিল |

#### `dig`-এর সবচেয়ে কাজের রূপগুলো

```bash
$ dig +short google.com                 # শুধু উত্তর
142.250.193.110

$ dig google.com AAAA +short            # IPv6 ঠিকানা
2404:6800:4002:82d::200e

$ dig gmail.com MX +short               # মেইল সার্ভার (সংখ্যা = অগ্রাধিকার, কম = আগে)
5 gmail-smtp-in.l.google.com.
10 alt1.gmail-smtp-in.l.google.com.

$ dig google.com NS +short              # ডোমেইনের name server
$ dig google.com TXT +short             # TXT (SPF, ডোমেইন যাচাই)
$ dig www.github.com CNAME +short       # অন্য নামের ছদ্মনাম
$ dig google.com SOA +short             # zone-এর মূল তথ্য
$ dig google.com ANY                    # অনেক সার্ভার এখন ANY-র পূর্ণ উত্তর দেয় না (RFC 8482)

$ dig @8.8.8.8 example.com              # নির্দিষ্ট DNS সার্ভারকে প্রশ্ন (Google)
$ dig @1.1.1.1 example.com +short       # Cloudflare

$ dig -x 8.8.8.8 +short                 # reverse lookup: IP → নাম (PTR)
dns.google.

$ dig +noall +answer google.com         # শুধু ANSWER সেকশন, পূর্ণ লাইনে
google.com.             187     IN      A       142.250.193.110

$ dig +trace example.com                # root থেকে শুরু করে প্রতিটা ধাপ নিজে হেঁটে দেখাও
```

| `dig` অপশন | কাজ |
|---|---|
| `@SERVER` | কোন DNS সার্ভারকে জিজ্ঞেস করবে |
| `TYPE` (`A`, `AAAA`, `MX`, `NS`, `TXT`, `CNAME`, `SOA`, `PTR`, `SRV`, `CAA`) | record-এর ধরন |
| `-x IP` | reverse lookup |
| `+short` | সংক্ষিপ্ত উত্তর |
| `+noall +answer` | শুধু উত্তর সেকশন |
| `+trace` | root থেকে পুরো পথ |
| `+tcp` | UDP-র বদলে TCP |
| `+dnssec` | DNSSEC record চাও |
| `-4` / `-6` | IPv4 / IPv6 দিয়ে প্রশ্ন পাঠাও |
| `-p PORT` | ভিন্ন port |
| `+norecurse` | recursion চাইবে না |

> 💡 **টিপস:** নতুন ডোমেইন বা DNS পরিবর্তন করেছ কিন্তু কাজ করছে না? `dig @8.8.8.8 yourdomain.com` আর `dig @1.1.1.1 yourdomain.com` দুটো চালিয়ে মেলাও, তারপর `dig yourdomain.com NS +short` দিয়ে authoritative সার্ভার বের করে সরাসরি তাকে জিজ্ঞেস করো (`dig @ns1.example-dns.com yourdomain.com`)। authoritative সার্ভার নতুন উত্তর দিলে বাকিরা পুরনো TTL শেষ হলেই আপডেট পাবে।

### `nslookup` — পুরনো, সব OS-এ পাওয়া যায়

Windows-এও আছে, তাই অনেকে চেনে।

```bash
$ nslookup google.com
Server:         127.0.0.53
Address:        127.0.0.53#53

Non-authoritative answer:
Name:   google.com
Address: 142.250.193.110
Name:   google.com
Address: 2404:6800:4002:82d::200e

$ nslookup -type=MX gmail.com
$ nslookup google.com 8.8.8.8           # নির্দিষ্ট সার্ভার (শেষে দাও)
```

"Non-authoritative answer" মানে উত্তর এসেছে cache বা recursive resolver থেকে, ডোমেইনের নিজের সার্ভার থেকে নয়। এটা স্বাভাবিক।

ইন্টারঅ্যাকটিভ মোডও আছে: শুধু `nslookup` লিখে Enter দাও, তারপর `set type=MX`, `server 8.8.8.8`, ডোমেইন নাম লেখো; বের হতে `exit`।

### `host` — সবচেয়ে সহজ, পরিষ্কার উত্তর

```bash
$ host google.com
google.com has address 142.250.193.110
google.com has IPv6 address 2404:6800:4002:82d::200e
google.com mail is handled by 10 smtp.google.com.

$ host 8.8.8.8
8.8.8.8.in-addr.arpa domain name pointer dns.google.

$ host -t NS google.com
$ host -t TXT google.com
$ host google.com 1.1.1.1
$ host -a google.com           # বিস্তারিত (dig-এর মতো)
```

### তিনটার তুলনা

| বৈশিষ্ট্য | `dig` | `nslookup` | `host` |
|---|---|---|---|
| আউটপুট | বিস্তারিত, কাঁচা DNS | মাঝামাঝি | সংক্ষিপ্ত, ইংরেজি বাক্যে |
| স্ক্রিপ্টে ব্যবহার | চমৎকার (`+short`) | ভালো নয় | ভালো |
| `+trace` / DNSSEC | হ্যাঁ | না | সীমিত |
| কখন ব্যবহার করবে | ট্রাবলশুটিং, পেশাদার কাজ | Windows-এর অভ্যাস থাকলে | দ্রুত এক নজরে |

---

## ৩৭.১০ `curl` — HTTP-র ছুরি-কাঁচি

`curl` (Client URL) টার্মিনাল থেকে URL-এ রিকোয়েস্ট পাঠায়। HTTP, HTTPS, FTP, SFTP, SCP সহ বহু প্রোটোকল সাপোর্ট করে। API টেস্ট, ওয়েবসাইট চেক, ফাইল নামানো, সার্ভারের হেডার দেখা: ডেভেলপার আর DevOps-এর প্রতিদিনের সঙ্গী।

### একটা HTTP রিকোয়েস্ট-রেসপন্স কেমন দেখতে

```
 curl (client)                                       সার্ভার
   │ ১. DNS: example.com → 93.184.215.14               │
   │ ২. TCP handshake (SYN, SYN-ACK, ACK) port 443      │
   │ ৩. TLS handshake (সার্টিফিকেট যাচাই)                │
   │ ৪. রিকোয়েস্ট ───────────────────────────────────►    │
   │    GET /index.html HTTP/1.1                       │
   │    Host: example.com                              │
   │    User-Agent: curl/8.5.0                         │
   │                                                   │
   │ ৫. রেসপন্স ◄────────────────────────────────────    │
   │    HTTP/1.1 200 OK                                │
   │    Content-Type: text/html                        │
   │    (ফাঁকা লাইন)                                    │
   │    <html>...body...</html>                        │
```

### বেসিক

```bash
$ curl https://example.com                 # body টার্মিনালে প্রিন্ট
$ curl -s https://api.ipify.org            # -s = silent (progress bar নেই); তোমার পাবলিক IP
103.12.44.87
```

### `-I` — শুধু হেডার দেখো (HEAD রিকোয়েস্ট)

```bash
$ curl -I https://example.com
HTTP/2 200
content-type: text/html
etag: "84238dfc8092e5d9c0dac8ef93371a07:1736799080.121134"
last-modified: Mon, 13 Jan 2025 20:11:20 GMT
cache-control: max-age=2357
date: Thu, 12 Nov 2026 04:20:11 GMT
content-length: 1256
```

প্রথম লাইনের **status code** সবচেয়ে গুরুত্বপূর্ণ:

| কোড | মানে |
|---|---|
| `200` | OK, সফল |
| `201` | Created (নতুন কিছু তৈরি হয়েছে, POST-এর পরে) |
| `204` | No Content (সফল, কিন্তু body নেই) |
| `301` / `308` | স্থায়ীভাবে অন্য জায়গায় সরে গেছে |
| `302` / `307` | অস্থায়ী redirect |
| `304` | Not Modified (cache-এরটাই চলবে) |
| `400` | Bad Request (রিকোয়েস্ট ভুল) |
| `401` | Unauthorized (লগইন/টোকেন লাগবে) |
| `403` | Forbidden (অনুমতি নেই) |
| `404` | Not Found |
| `405` | Method Not Allowed |
| `429` | Too Many Requests (rate limit) |
| `500` | Internal Server Error (সার্ভারের কোডে সমস্যা) |
| `502` | Bad Gateway (proxy-র পেছনের অ্যাপ সাড়া দেয়নি) |
| `503` | Service Unavailable |
| `504` | Gateway Timeout |

> 📝 **মনে রাখো:** `-I` আসলে HEAD মেথড পাঠায়। কিছু সার্ভার HEAD-এ ঠিকমতো উত্তর দেয় না। তখন `curl -s -D - -o /dev/null URL` চালাও: GET পাঠাবে, হেডার (`-D -` = stdout-এ dump) দেখাবে আর body ফেলে দেবে। অথবা `-i` (ছোট হাতের) দিলে হেডার আর body দুটোই দেখায়।

### `-L` — redirect অনুসরণ করো

```bash
$ curl -I http://github.com
HTTP/1.1 301 Moved Permanently
Content-Length: 0
Location: https://github.com/

$ curl -sIL http://github.com | grep -i -E '^(HTTP|location)'
HTTP/1.1 301 Moved Permanently
Location: https://github.com/
HTTP/2 200
```

ডিফল্টভাবে `curl` redirect অনুসরণ করে না। `-L` দিলে `Location` হেডার ধরে শেষ পর্যন্ত যায় (সর্বোচ্চ সংখ্যা `--max-redirs`)।

### `-o` আর `-O` — ফাইলে সংরক্ষণ

```bash
$ curl -o ubuntu.iso https://releases.ubuntu.com/24.04/ubuntu-24.04.1-desktop-amd64.iso
$ curl -O https://example.com/files/report.pdf      # সার্ভারের নামেই (report.pdf) সেভ
$ curl -LO https://github.com/user/repo/releases/download/v1.0/tool.tar.gz   # redirect + সেভ
$ curl -C - -O https://example.com/big.iso          # থেমে যাওয়া ডাউনলোড আবার শুরু (resume)
$ curl -o /dev/null -s -w "%{http_code}\n" https://example.com   # body ফেলে শুধু status
200
```

### `-X`, `-H`, `-d` — API নিয়ে কাজ

```bash
# -X: HTTP method (GET, POST, PUT, PATCH, DELETE)
$ curl -X DELETE https://api.example.com/users/42

# -H: হেডার যোগ করো (একাধিকবার দেওয়া যায়)
$ curl -H "Accept: application/json" \
       -H "Authorization: Bearer eyJhbGciOi..." \
       https://api.example.com/me

# -d: body-তে ডেটা পাঠাও। -d দিলে method নিজে থেকেই POST হয়,
#     আর Content-Type হয় application/x-www-form-urlencoded
$ curl -d "name=Rahim&age=25" https://httpbin.org/post

# JSON পাঠানো (Content-Type নিজে বলে দিতে হবে)
$ curl -X POST https://httpbin.org/post \
       -H "Content-Type: application/json" \
       -d '{"name":"Rahim","age":25}'

# ফাইল থেকে body পড়ো (@ চিহ্ন)
$ curl -H "Content-Type: application/json" -d @user.json https://httpbin.org/post

# curl 7.82+ এ শর্টকাট: --json নিজেই Content-Type ও Accept হেডার বসায় আর POST করে
$ curl --json '{"name":"Rahim"}' https://httpbin.org/post

# PUT দিয়ে আপডেট
$ curl -X PUT -H "Content-Type: application/json" -d '{"age":26}' https://api.example.com/users/42

# ফর্মে ফাইল আপলোড (multipart/form-data)
$ curl -F "photo=@/home/rahim/pic.jpg" -F "title=Holiday" https://httpbin.org/post
```

> ⚠️ **সতর্কতা:** `-X POST`-এর সাথে `-d` দিলে `-X` আসলে অপ্রয়োজনীয়, তবে ক্ষতি নেই। কিন্তু `-X POST`-এর সাথে `-L` দিলে redirect-এর পরেও POST-ই পাঠাবে, যা প্রায়ই ভুল ফল দেয়। redirect-এ সঠিক আচরণ চাইলে `-X` বাদ দিয়ে শুধু `-d` ব্যবহার করো।

> ⚠️ **সতর্কতা:** কমান্ড লাইনে পাসওয়ার্ড বা টোকেন লিখলে সেটা shell history-তে (`~/.bash_history`) আর `ps` আউটপুটে অন্যরা দেখতে পারে। লাইনের শুরুতে একটা স্পেস দিলে (যদি `HISTCONTROL=ignorespace` থাকে) history-তে যায় না, অথবা `-H @headers.txt` দিয়ে ফাইল থেকে হেডার পড়াও (curl 7.55+)। বিস্তারিত [অধ্যায় ৬](06-terminal-shortcuts.md) আর [অধ্যায় ৪৪](44-environment-bashrc.md)-এ।

### `-v` আর `-w` — ভেতরে কী হচ্ছে দেখো

```bash
$ curl -v https://example.com -o /dev/null
*   Trying 93.184.215.14:443...
* Connected to example.com (93.184.215.14) port 443
* ALPN: curl offers h2,http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
...
* Server certificate:
*  subject: C=US; ST=California; L=Los Angeles; O=Internet Corporation for Assigned Names and Numbers; CN=*.example.com
*  expire date: Jan 15 23:59:59 2027 GMT
> GET / HTTP/2
> Host: example.com
> User-Agent: curl/8.5.0
> Accept: */*
>
< HTTP/2 200
< content-type: text/html
...
```

`*` = curl-এর নিজের তথ্য, `>` = পাঠানো রিকোয়েস্ট, `<` = পাওয়া রেসপন্স। সার্টিফিকেটের মেয়াদ আর TLS সমস্যা এখানেই ধরা পড়ে।

`-w` (write-out) দিয়ে সময় মাপো, কোন ধাপে দেরি হচ্ছে বোঝো:

```bash
$ curl -o /dev/null -s -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nFirstByte: %{time_starttransfer}s\nTotal: %{time_total}s\nCode: %{http_code}\n" https://example.com
DNS: 0.004512s
Connect: 0.152330s
TLS: 0.318842s
FirstByte: 0.471220s
Total: 0.471580s
Code: 200
```

এই সময়গুলো **শুরু থেকে জমা হিসাবে** গোনা। DNS বেশি হলে DNS সমস্যা; Connect বেশি হলে নেটওয়ার্ক বা দূরত্ব; FirstByte আর TLS-এর পার্থক্য বেশি হলে সার্ভারের অ্যাপ ধীর।

### `curl`-এর গুরুত্বপূর্ণ অপশন — সম্পূর্ণ টেবিল

| অপশন | পূর্ণ নাম | কাজ |
|---|---|---|
| `-I` | `--head` | শুধু হেডার (HEAD method) |
| `-i` | `--include` | হেডার + body |
| `-L` | `--location` | redirect অনুসরণ |
| `-o FILE` | `--output` | নির্দিষ্ট ফাইলে সেভ |
| `-O` | `--remote-name` | URL-এর নামে সেভ |
| `-X METHOD` | `--request` | HTTP method |
| `-H "K: V"` | `--header` | হেডার যোগ |
| `-d DATA` | `--data` | body ডেটা (POST) |
| `--data-urlencode` | | ডেটা URL-encode করে পাঠাও |
| `--json DATA` | | JSON পাঠাও (7.82+) |
| `-F` | `--form` | multipart ফর্ম/ফাইল আপলোড |
| `-u user:pass` | `--user` | Basic authentication |
| `-v` | `--verbose` | বিস্তারিত ডিবাগ |
| `-s` | `--silent` | progress/error চুপ |
| `-S` | `--show-error` | `-s`-এর সাথে দিলে শুধু error দেখাবে |
| `-f` | `--fail` | HTTP 4xx/5xx এ কোনো body না দেখিয়ে non-zero exit code (স্ক্রিপ্টে দরকারি) |
| `-k` | `--insecure` | TLS সার্টিফিকেট যাচাই করবে না (শুধু টেস্টে!) |
| `-w FORMAT` | `--write-out` | শেষে নিজের মতো তথ্য প্রিন্ট |
| `-C -` | `--continue-at` | ডাউনলোড resume |
| `-A AGENT` | `--user-agent` | User-Agent বদলাও |
| `-e URL` | `--referer` | Referer হেডার |
| `-b` / `-c` | `--cookie` / `--cookie-jar` | cookie পাঠাও / সেভ করো |
| `-x PROXY` | `--proxy` | proxy দিয়ে যাও |
| `-m SEC` | `--max-time` | মোট সময়সীমা |
| `--connect-timeout SEC` | | সংযোগ স্থাপনের সময়সীমা |
| `--retry N` | | ব্যর্থ হলে N বার চেষ্টা |
| `-4` / `-6` | | IPv4 / IPv6 |
| `--resolve host:port:IP` | | DNS বাদ দিয়ে নির্দিষ্ট IP-তে পাঠাও (নতুন সার্ভার টেস্টে দারুণ) |
| `-Z` | `--parallel` | একাধিক URL একসাথে (7.66+) |

```bash
# DNS না বদলেই নতুন সার্ভারে ওয়েবসাইট টেস্ট
$ curl --resolve example.com:443:203.0.113.10 https://example.com -I

# স্ক্রিপ্টে: সাইট ডাউন হলে exit code non-zero
$ curl -fsS -m 10 https://example.com/health > /dev/null && echo "UP" || echo "DOWN"
UP
```

---

## ৩৭.১১ `wget` — বিশ্বস্ত ডাউনলোডার

`curl` সব কাজের ছুরি, আর `wget` বিশেষজ্ঞ ডাউনলোডার। এর বিশেষত্ব হলো ডিফল্টে ফাইলে সেভ করা, নেটওয়ার্ক কাটলে নিজে আবার চেষ্টা করা, আর পুরো ওয়েবসাইট recursive ভাবে নামাতে পারা।

```bash
$ wget https://example.com/file.zip
--2026-11-12 10:30:01--  https://example.com/file.zip
Resolving example.com (example.com)... 93.184.215.14
Connecting to example.com (example.com)|93.184.215.14|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 10485760 (10M) [application/zip]
Saving to: ‘file.zip’

file.zip            100%[===================>]  10.00M  2.31MB/s    in 4.3s

2026-11-12 10:30:06 (2.31 MB/s) - ‘file.zip’ saved [10485760/10485760]
```

| অপশন | কাজ |
|---|---|
| `-O FILE` | নির্দিষ্ট নামে সেভ (`-O -` = stdout-এ) — **খেয়াল করো: curl-এর `-O`-এর মানে আলাদা!** |
| `-P DIR` | কোন ফোল্ডারে সেভ |
| `-c` | আধা-নামানো ফাইল resume |
| `-b` | ব্যাকগ্রাউন্ডে চালাও (লগ `wget-log`-এ) |
| `-q` | চুপচাপ |
| `-i FILE` | ফাইলে লেখা URL-গুলো একে একে নামাও |
| `-t N` | কতবার চেষ্টা (`0` = অসীম) |
| `-T SEC` | timeout |
| `--limit-rate=500k` | স্পিড সীমিত করো |
| `-r` | recursive |
| `-l N` | recursive-এর গভীরতা |
| `-np` | parent ডিরেক্টরিতে উঠবে না |
| `-k` | ডাউনলোড করা পেজের লিংক লোকাল করে দাও |
| `-p` | পেজ দেখাতে যা লাগে (ছবি, CSS) সব নামাও |
| `-m` / `--mirror` | পুরো সাইট mirror (`-r -N -l inf --no-remove-listing`) |
| `--spider` | ডাউনলোড না করে শুধু আছে কি না দেখো |
| `--user` / `--password` | লগইন |
| `--no-check-certificate` | TLS যাচাই বাদ (শুধু টেস্টে) |

```bash
$ wget -c https://releases.ubuntu.com/24.04/ubuntu-24.04.1-desktop-amd64.iso
$ wget -P ~/Downloads -i links.txt
$ wget --mirror -k -p -np https://docs.example.com/guide/
$ wget --spider https://example.com/file.zip && echo "ফাইল আছে"
$ wget -qO- https://api.ipify.org; echo     # ফাইলে না রেখে টার্মিনালে দেখাও
```

> 📝 **মনে রাখো:** Fedora 40 থেকে `wget` কমান্ডটা আসলে **wget2** (নতুন সংস্করণ, একাধিক সংযোগে দ্রুত)। প্রায় সব অপশন একই, তবে আউটপুট দেখতে কিছুটা আলাদা হতে পারে।

### `curl` বনাম `wget`

| বিষয় | `curl` | `wget` |
|---|---|---|
| ডিফল্ট আউটপুট | stdout (টার্মিনাল) | ফাইল |
| redirect | `-L` দিতে হয় | নিজেই অনুসরণ করে |
| recursive/mirror | না | হ্যাঁ |
| API (POST/PUT/হেডার) | খুব শক্তিশালী | সীমিত |
| প্রোটোকল | অনেক (HTTP, FTP, SFTP, SMTP, ...) | মূলত HTTP/HTTPS/FTP |
| লাইব্রেরি | `libcurl` (অসংখ্য প্রোগ্রাম ব্যবহার করে) | না |

---

## ৩৭.১২ `nc` (netcat) — নেটওয়ার্কের "TCP/IP সুইস আর্মি নাইফ"

`nc` যেকোনো TCP বা UDP port-এ কাঁচা সংযোগ খোলে আর কীবোর্ড থেকে লেখা পাঠায়, বা কোনো port-এ শুনে বসে থাকে। কোনো প্রোটোকল নিজে বোঝে না, শুধু byte আদান-প্রদান করে। তাই যেকোনো সার্ভিস হাতে-কলমে পরীক্ষা করা যায়।

> 📝 **মনে রাখো (ভ্যারিয়েন্টের ঝামেলা):** তিন রকম netcat আছে, অপশনেও কিছু পার্থক্য:
> - **OpenBSD netcat** (`netcat-openbsd`): Ubuntu/Debian-এর ডিফল্ট। listen করতে `nc -l 1234`।
> - **ncat** (Nmap প্রকল্পের, `nmap-ncat`): Fedora/RHEL-এর ডিফল্ট, `nc` নামেও চলে। `nc -l 1234` আর `nc -l -p 1234` দুটোই চলে।
> - **traditional netcat** (`netcat-traditional`): পুরনো; listen করতে `nc -l -p 1234`।
>
> সন্দেহ হলে `nc -h` বা `man nc` দেখো।

| অপশন | কাজ |
|---|---|
| `-l` | listen মোড (সার্ভার হয়ে বসে থাকো) |
| `-p PORT` | source/listen port (variant ভেদে) |
| `-z` | কোনো ডেটা না পাঠিয়ে শুধু port খোলা কি না দেখো (scan; খুব পুরনো ncat সংস্করণে নেই, তখন `nmap` ব্যবহার করো) |
| `-v` | বিস্তারিত (সফল/ব্যর্থ জানাবে) |
| `-u` | TCP-র বদলে UDP |
| `-w SEC` | timeout |
| `-n` | DNS lookup করবে না |
| `-k` | একটা সংযোগ শেষ হলেও listen চালিয়ে যাও (OpenBSD ও ncat) |
| `-N` | stdin শেষ হলে (EOF) সংযোগ বন্ধ করো (OpenBSD) |
| `-4` / `-6` | IPv4 / IPv6 |

### ১. port খোলা কি না (সবচেয়ে বেশি ব্যবহৃত)

```bash
$ nc -zv 192.168.1.10 22
Connection to 192.168.1.10 22 port [tcp/ssh] succeeded!

$ nc -zv 192.168.1.10 3306
nc: connect to 192.168.1.10 port 3306 (tcp) failed: Connection refused

$ nc -zv -w 3 example.com 8080
nc: connect to example.com port 8080 (tcp) timed out: Operation now in progress

$ nc -zv 192.168.1.10 20-25        # একটা রেঞ্জ (OpenBSD)
```

**"refused" আর "timed out"-এর পার্থক্য বোঝা খুব জরুরি:**

| ফলাফল | ভেতরে কী হয়েছে | মানে |
|---|---|---|
| `succeeded` | SYN → SYN-ACK পাওয়া গেছে | port খোলা, কেউ listen করছে |
| `Connection refused` | SYN → **RST** ফেরত এসেছে | হোস্টে পৌঁছানো গেছে, কিন্তু ওই port-এ কেউ listen করছে না (সার্ভিস বন্ধ, বা শুধু `127.0.0.1`-এ listen করছে) |
| `timed out` | SYN → কোনো উত্তর নেই | পথে firewall প্যাকেট চুপচাপ ফেলে দিচ্ছে (DROP), বা হোস্টই নেই |
| `No route to host` | ICMP host unreachable, বা firewall REJECT করছে | route/ARP সমস্যা বা firewall reject |

### ২. দুই কম্পিউটারের মধ্যে চ্যাট

```bash
# কম্পিউটার A (সার্ভার, IP 192.168.1.10):
$ nc -l 5000

# কম্পিউটার B (ক্লায়েন্ট):
$ nc 192.168.1.10 5000
হ্যালো!           ← B-তে লিখলে A-তে দেখা যাবে, উল্টোটাও
```

বের হতে `Ctrl+C` (বা ক্লায়েন্টে `Ctrl+D` দিয়ে EOF পাঠাও)।

### ৩. ফাইল পাঠানো

```bash
# গ্রহণকারী (A):
$ nc -l 5000 > received.tar.gz

# প্রেরক (B):
$ nc -N 192.168.1.10 5000 < backup.tar.gz      # -N: শেষ হলে বন্ধ (OpenBSD)
```

> ⚠️ **সতর্কতা:** `nc` সব কিছু **এনক্রিপশন ছাড়া** পাঠায়। বিশ্বস্ত লোকাল নেটওয়ার্কের বাইরে গোপন কিছু পাঠাতে `scp`/`rsync` over SSH ব্যবহার করো ([অধ্যায় ৩৮](38-ssh-remote.md), [অধ্যায় ৩৫](35-backup-rsync.md))। কাজ শেষে listener বন্ধ করতে ভুলো না।

### ৪. হাতে HTTP বা SMTP কথা বলা

```bash
$ printf 'GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n' | nc example.com 80
HTTP/1.1 200 OK
Content-Type: text/html
...
```

এভাবে বোঝা যায় HTTP আসলে শুধু সাধারণ টেক্সট লাইন। (HTTPS-এর জন্য TLS লাগে, তখন `openssl s_client -connect example.com:443` ব্যবহার করো।)

### ৫. একটা সাধারণ সার্ভার বানিয়ে firewall টেস্ট

নতুন একটা port firewall-এ খুলেছ, কিন্তু অ্যাপ এখনো বসাওনি। তাহলে খোলা ঠিক হলো কি না কীভাবে বুঝবে?

```bash
# সার্ভারে:
$ nc -lk 8080

# অন্য মেশিন থেকে:
$ nc -zv server-ip 8080     # succeeded হলে firewall ঠিক আছে
```

UDP টেস্ট: সার্ভারে `nc -lu 5353`, ক্লায়েন্টে `echo hi | nc -u -w1 server-ip 5353`। (UDP-তে কোনো handshake নেই, তাই `-zv` প্রায় সবসময় "succeeded" বলে। আসল ডেটা গেল কি না দেখে নিশ্চিত হও।)

---

## ৩৭.১৩ `tcpdump` — প্যাকেট নিজের চোখে দেখা

বাকি সব টুল যখন বলে "কিছু ঠিক নেই" কিন্তু কেন তা বলে না, তখন `tcpdump` তারের ভেতর দিয়ে যাওয়া আসল প্যাকেট দেখায়। এটা হলো নেটওয়ার্কের **এক্স-রে**।

### ভেতরে কীভাবে কাজ করে

```
  নেটওয়ার্ক কার্ড ──► কার্নেল নেটওয়ার্ক স্ট্যাক ──► অ্যাপ্লিকেশন
                        │
                        │  AF_PACKET socket (প্রতিটা প্যাকেটের কপি)
                        ▼
                   BPF ফিল্টার (কার্নেলের ভেতরে চলে, যেমন "port 80")
                        │  শুধু মিলে যাওয়া প্যাকেট
                        ▼
                   libpcap ──► tcpdump (প্রিন্ট বা .pcap ফাইলে লেখা)
```

ফিল্টারটা (BPF, Berkeley Packet Filter) কার্নেলের ভেতরেই চলে, তাই অপ্রয়োজনীয় প্যাকেট user space পর্যন্ত আসে না। ফলে ব্যস্ত সার্ভারেও tcpdump চালানো যায়। কাঁচা প্যাকেট পড়তে root (`CAP_NET_RAW`) লাগে।

### বেসিক ব্যবহার

```bash
$ sudo tcpdump -D                  # কোন কোন interface আছে
1.enp3s0 [Up, Running, Connected]
2.any (Pseudo-device that captures on all interfaces) [Up, Running]
3.lo [Up, Running, Loopback]
...

$ sudo tcpdump -i enp3s0 -nn -c 5 icmp
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp3s0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
10:40:01.123456 IP 192.168.1.105 > 8.8.8.8: ICMP echo request, id 7, seq 1, length 64
10:40:01.145678 IP 8.8.8.8 > 192.168.1.105: ICMP echo reply, id 7, seq 1, length 64
...
5 packets captured
5 packets received by filter
0 packets dropped by kernel
```

(অন্য টার্মিনালে `ping 8.8.8.8` চালালে এগুলো দেখা যাবে।)

| অপশন | কাজ |
|---|---|
| `-i IFACE` | কোন interface (`-i any` = সব) |
| `-D` | interface তালিকা |
| `-n` | IP-কে নামে বদলাবে না |
| `-nn` | IP আর port কোনোটাই নামে বদলাবে না (সবচেয়ে দ্রুত, সবচেয়ে প্রচলিত) |
| `-c N` | N টা প্যাকেট ধরে থামো |
| `-v` / `-vv` / `-vvv` | বেশি বিস্তারিত (TTL, IP id, checksum) |
| `-A` | প্যাকেটের ভেতরের ডেটা ASCII-তে (HTTP দেখতে কাজে লাগে) |
| `-X` | hex + ASCII দুটোই |
| `-e` | Ethernet (MAC) header দেখাও |
| `-w FILE.pcap` | প্যাকেট ফাইলে সেভ (পরে Wireshark-এ খোলা যায়) |
| `-r FILE.pcap` | সেভ করা ফাইল পড়ো |
| `-s N` | প্রতিটা প্যাকেটের কত byte ধরবে (ডিফল্ট 262144, মানে পুরোটা) |
| `-q` | সংক্ষিপ্ত আউটপুট |
| `-t` | timestamp দেখাবে না |
| `-l` | line-buffered (পাইপ দিয়ে `grep`-এ পাঠালে লাগে) |

### ফিল্টার (BPF expression)

| ফিল্টার | মানে |
|---|---|
| `host 192.168.1.20` | এই হোস্টের যাওয়া বা আসা |
| `src host 10.0.0.5` / `dst host 10.0.0.5` | শুধু উৎস / শুধু গন্তব্য |
| `net 192.168.1.0/24` | পুরো subnet |
| `port 443` | port 443 (উৎস বা গন্তব্য) |
| `src port 53` / `dst port 80` | নির্দিষ্ট দিকের port |
| `portrange 8000-8100` | port রেঞ্জ |
| `tcp` / `udp` / `icmp` / `arp` | প্রোটোকল |
| `and` / `or` / `not` (`&&` `||` `!`) | যুক্ত করা |
| `'tcp[tcpflags] & tcp-syn != 0'` | শুধু SYN flag থাকা প্যাকেট |

```bash
# DNS প্রশ্ন আর উত্তর দেখো
$ sudo tcpdump -i any -nn port 53
10:41:10.001 IP 192.168.1.105.48211 > 192.168.1.1.53: 31415+ A? example.com. (29)
10:41:10.019 IP 192.168.1.1.53 > 192.168.1.105.48211: 31415 1/0/0 A 93.184.215.14 (45)

# একটা হোস্টের সাথে HTTP ট্রাফিক, SSH বাদ দিয়ে
$ sudo tcpdump -i enp3s0 -nn 'host 192.168.1.20 and not port 22'

# HTTP (এনক্রিপ্ট না হওয়া) রিকোয়েস্টের লেখা পড়ো
$ sudo tcpdump -i any -nn -A 'tcp port 80'

# পরে বিশ্লেষণের জন্য ফাইলে রাখো
$ sudo tcpdump -i enp3s0 -nn -w /tmp/capture.pcap 'port 443'
$ tcpdump -nn -r /tmp/capture.pcap | head
```

> ⚠️ **সতর্কতা:** রিমোট সার্ভারে SSH দিয়ে ঢুকে ফিল্টার ছাড়া `tcpdump` চালালে সে নিজের SSH ট্রাফিকও ধরবে, যা আবার আউটপুট হয়ে SSH দিয়ে আসবে... এক অনন্ত লুপে স্ক্রিন ভরে যাবে। সবসময় `not port 22` যোগ করো।

> ⚠️ **সতর্কতা (আইনি):** অন্যের নেটওয়ার্ক বা অন্যের ট্রাফিক অনুমতি ছাড়া capture করা বেশিরভাগ দেশে অপরাধ। বাংলাদেশে ডিজিটাল/সাইবার নিরাপত্তা আইনেও এর শাস্তি আছে। শুধু নিজের মেশিন বা যেখানে লিখিত অনুমতি আছে সেখানে ব্যবহার করো। `.pcap` ফাইলে পাসওয়ার্ড বা ব্যক্তিগত তথ্য থাকতে পারে, তাই সাবধানে রাখো।

### TCP flag পড়া: 3-way handshake নিজের চোখে

```bash
$ sudo tcpdump -i any -nn 'tcp port 80 and host 93.184.215.14'
10:42:00.100 IP 192.168.1.105.51500 > 93.184.215.14.80: Flags [S], seq 1000, win 64240, length 0
10:42:00.250 IP 93.184.215.14.80 > 192.168.1.105.51500: Flags [S.], seq 5000, ack 1001, win 65535, length 0
10:42:00.251 IP 192.168.1.105.51500 > 93.184.215.14.80: Flags [.], ack 5001, win 502, length 0
10:42:00.252 IP 192.168.1.105.51500 > 93.184.215.14.80: Flags [P.], seq 1001:1076, ack 5001, length 75: HTTP: GET / HTTP/1.1
...
10:42:00.600 IP 192.168.1.105.51500 > 93.184.215.14.80: Flags [F.], seq 1076, ack 6300, length 0
```

| tcpdump flag | মানে |
|---|---|
| `[S]` | SYN: "সংযোগ করতে চাই" |
| `[S.]` | SYN-ACK: "ঠিক আছে, আমিও রাজি" (`.` = ACK) |
| `[.]` | শুধু ACK |
| `[P.]` | PUSH + ACK: ডেটা পাঠাচ্ছে |
| `[F.]` | FIN: "আমার কথা শেষ" |
| `[R]` / `[R.]` | RST: "সংযোগ বাতিল / এখানে কেউ নেই" |

> 💡 **টিপস:** `[S]` যাচ্ছে কিন্তু কোনো উত্তর নেই = firewall DROP বা হোস্ট নেই। `[S]` যাচ্ছে আর `[R.]` আসছে = port বন্ধ (refused)। বারবার একই `[S]` পাঠাচ্ছে = retransmission, উত্তর হারিয়ে যাচ্ছে।

> 💡 **টিপস:** GUI-তে আরও আরামে বিশ্লেষণ করতে চাইলে **Wireshark** ইনস্টল করো (`sudo apt install wireshark`), আর `tcpdump -w` দিয়ে বানানো `.pcap` ফাইলটা খোলো। শুধু টার্মিনাল থাকলে `tshark` আছে।

---

## ৩৭.১৪ `nmap` পরিচিতি — নিজের সিস্টেম স্ক্যান

`nmap` (Network Mapper) একটা শক্তিশালী নেটওয়ার্ক স্ক্যানার: কোন হোস্ট বেঁচে আছে, কোন port খোলা, কোন সার্ভিস ও কোন সংস্করণ চলছে।

> ⚠️ **সতর্কতা (খুব গুরুত্বপূর্ণ):** অনুমতি ছাড়া অন্যের সার্ভার, কলেজ/অফিস নেটওয়ার্ক বা ইন্টারনেটের কোনো আইপি স্ক্যান করা অনৈতিক, অনেক দেশে বেআইনি, আর ISP তোমার সংযোগ বন্ধ করে দিতে পারে। **শুধু নিজের মেশিন (`localhost`), নিজের বাসার ল্যাব, বা যেখানে লিখিত অনুমতি আছে সেখানে চালাও।** অনুশীলনের জন্য Nmap প্রকল্প নিজেই `scanme.nmap.org` নামের একটা সার্ভার রেখেছে, যেটা হালকাভাবে স্ক্যান করার অনুমতি দেওয়া আছে।

```bash
$ nmap localhost
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-11-12 10:50 +06
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000080s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
631/tcp  open  ipp
3306/tcp open  mysql

Nmap done: 1 IP address (1 host up) scanned in 0.05 seconds
```

ডিফল্টভাবে `nmap` সবচেয়ে প্রচলিত ১০০০টা TCP port দেখে।

| অপশন | কাজ |
|---|---|
| `-sT` | TCP connect scan (পূর্ণ সংযোগ; root ছাড়া এটাই ডিফল্ট) |
| `-sS` | SYN scan ("half-open"; root লাগে, root হলে এটাই ডিফল্ট) |
| `-sU` | UDP scan (ধীর, root লাগে) |
| `-sn` | শুধু কোন হোস্ট বেঁচে আছে (port scan নেই; আগে নাম ছিল `-sP`) |
| `-p 22,80,443` | নির্দিষ্ট port |
| `-p 1-1024` / `-p-` | রেঞ্জ / সব ৬৫৫৩৫ port |
| `-F` | দ্রুত: শুধু শীর্ষ ১০০টা port |
| `-sV` | সার্ভিসের সংস্করণ বের করো |
| `-O` | অপারেটিং সিস্টেম অনুমান (root) |
| `-A` | aggressive: `-sV -O -sC --traceroute` একসাথে |
| `-Pn` | হোস্ট বেঁচে আছে কি না যাচাই না করেই স্ক্যান |
| `-T0`…`-T5` | গতি (`-T4` দ্রুত; বেশি দ্রুত মানে বেশি শব্দ) |
| `-oN file` / `-oX file` | ফলাফল সাধারণ লেখায় / XML-এ সেভ |

```bash
$ nmap -sV -p 22,80 localhost
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)

# নিজের বাসার নেটওয়ার্কে কোন কোন ডিভাইস চালু (শুধু নিজের নেটওয়ার্ক!)
$ sudo nmap -sn 192.168.1.0/24
Nmap scan report for _gateway (192.168.1.1)
Host is up (0.0011s latency).
MAC Address: 08:00:27:AA:BB:CC (Router Vendor)
...
```

port-এর অবস্থা বোঝো:

| STATE | মানে |
|---|---|
| `open` | কেউ listen করছে |
| `closed` | পৌঁছানো গেছে, কেউ listen করছে না (RST এসেছে) |
| `filtered` | firewall উত্তর আটকে দিচ্ছে, nmap নিশ্চিত হতে পারছে না |
| `open\|filtered` | UDP-তে প্রায়ই; খোলা নাকি filtered বোঝা যাচ্ছে না |

> 💡 **টিপস:** নিজের সার্ভার বাইরে থেকে কেমন দেখায় তা জানতে **অন্য একটা মেশিন থেকে** নিজের সার্ভারকে `nmap` করো। `localhost` থেকে স্ক্যান করলে firewall পার হতে হয় না, তাই সব খোলা দেখায়, যা বাস্তব চিত্র নয়।

---

## ৩৭.১৫ নেটওয়ার্ক সমস্যা সমাধানের ধাপে ধাপে পদ্ধতি

"ইন্টারনেট চলছে না" খুব অস্পষ্ট একটা অভিযোগ। পেশাদাররা অনুমান করে এলোমেলো কমান্ড চালান না; **নিচ থেকে ওপরে** (bottom-up), একটা একটা লেয়ার পরীক্ষা করেন। যে ধাপে প্রথম ব্যর্থ হবে, সমস্যা সেখানেই বা তার ঠিক নিচে।

> 💡 **উপমা:** বাতি জ্বলছে না? আগে দেখো মেইন সুইচ চালু কি না, তারপর তার, তারপর হোল্ডার, শেষে বাল্ব। একেবারে নতুন বাল্ব কিনে আনা প্রথম কাজ নয়।

```
  ┌───────────────────────────────────────────────────────────────────┐
  │ ধাপ ১: লিংক আছে?        ip -br link        → UP / LOWER_UP?       │
  │        না → তার/Wi-Fi/ড্রাইভার, nmcli, ethtool, iw                 │
  ├───────────────────────────────────────────────────────────────────┤
  │ ধাপ ২: IP আছে?          ip -br addr        → ঠিক subnet-এর IP?     │
  │        না / 169.254.x.x → DHCP সমস্যা                              │
  ├───────────────────────────────────────────────────────────────────┤
  │ ধাপ ৩: gateway আছে?     ip route           → default via ... ?   │
  │        ping -c3 <gateway>; ip neigh                              │
  ├───────────────────────────────────────────────────────────────────┤
  │ ধাপ ৪: ইন্টারনেট (IP)?   ping -c3 8.8.8.8 / 1.1.1.1                │
  │        না → রাউটার/ISP; mtr 8.8.8.8 দিয়ে কোথায় থামছে দেখো           │
  ├───────────────────────────────────────────────────────────────────┤
  │ ধাপ ৫: DNS?             dig google.com / resolvectl status        │
  │        IP-তে ping চলে কিন্তু নামে চলে না → DNS সমস্যা                │
  ├───────────────────────────────────────────────────────────────────┤
  │ ধাপ ৬: port পৌঁছায়?     nc -zv host 443 / traceroute -T            │
  │        refused → সার্ভিস নেই; timeout → firewall                  │
  ├───────────────────────────────────────────────────────────────────┤
  │ ধাপ ৭: অ্যাপ ঠিক?        curl -v / curl -I                        │
  │        HTTP 5xx → সার্ভারের অ্যাপ; TLS error → সার্টিফিকেট/সময়       │
  ├───────────────────────────────────────────────────────────────────┤
  │ শেষ অস্ত্র: tcpdump দিয়ে আসল প্যাকেট দেখো                           │
  └───────────────────────────────────────────────────────────────────┘
```

### ধাপগুলো বিস্তারিত

**ধাপ ১ — লিংক:**

```bash
$ ip -br link
$ nmcli device status          # NetworkManager থাকলে (Desktop, Fedora, RHEL)
DEVICE   TYPE      STATE         CONNECTION
enp3s0   ethernet  connected     Wired connection 1
wlp2s0   wifi      disconnected  --
```

`NO-CARRIER`/`DOWN` হলে: তার লাগানো? Wi-Fi কানেক্টেড? `rfkill list` দেখো (Wi-Fi সফটওয়্যারে বা হার্ডওয়্যার সুইচে blocked কি না)। ড্রাইভার লোড হয়েছে কি না দেখতে `sudo dmesg | grep -i -E 'eth|wlan|firmware'` ([অধ্যায় ৪৭](47-kernel-proc-sys.md))।

**ধাপ ২ — IP:**

```bash
$ ip -4 -br addr
enp3s0           UP             169.254.23.7/16
```

`169.254.x.x` (link-local / APIPA) বা কোনো IP না থাকা মানে DHCP সার্ভার থেকে IP পাওয়া যায়নি। NetworkManager-এ সংযোগ আবার চালু করো: `nmcli connection up "Wired connection 1"`, বা লগ দেখো: `journalctl -u NetworkManager -b --since "10 min ago"` ([অধ্যায় ২৮](28-logs.md))।

**ধাপ ৩ — gateway:**

```bash
$ ip route | grep default
default via 192.168.1.1 dev enp3s0 proto dhcp metric 100
$ ping -c 3 192.168.1.1
$ ip neigh show 192.168.1.1
```

default route নেই → `Network is unreachable` আসবে। gateway-এর ARP `FAILED` → লেয়ার ২ সমস্যা বা ভুল subnet।

**ধাপ ৪ — ইন্টারনেট (IP দিয়ে, DNS ছাড়া):**

```bash
$ ping -c 3 8.8.8.8
$ mtr -rwn -c 20 8.8.8.8
```

gateway পর্যন্ত ঠিক, কিন্তু এর পরে সব হারাচ্ছে → রাউটারের WAN/ISP সমস্যা।

**ধাপ ৫ — DNS:**

```bash
$ ping -c 1 google.com
ping: google.com: Temporary failure in name resolution
$ dig google.com +short                  # কিছু আসে না
$ dig @8.8.8.8 google.com +short         # এটায় আসে → লোকাল DNS সার্ভার সমস্যা
142.250.193.110
$ resolvectl status
```

`8.8.8.8` দিয়ে কাজ করে কিন্তু ডিফল্ট দিয়ে করে না → রাউটারের DNS বা resolved-এর কনফিগারেশন ঠিক করো (অস্থায়ীভাবে `resolvectl dns enp3s0 1.1.1.1 8.8.8.8`)।

**ধাপ ৬ — নির্দিষ্ট সার্ভিসের port:**

```bash
$ nc -zv -w 5 myserver.com 443
$ sudo traceroute -T -p 443 -n myserver.com
```

নিজে সার্ভার হলে উল্টো দিক থেকেও দেখো: সার্ভারে `sudo ss -tlnp | grep 443` দিয়ে সার্ভিস listen করছে কি না, আর কোন IP-তে (`127.0.0.1` না `0.0.0.0`)। তারপর firewall দেখো: `sudo ufw status` / `sudo firewall-cmd --list-all` / `sudo nft list ruleset` ([অধ্যায় ৩৯](39-firewall.md))। ক্লাউডে থাকলে প্রোভাইডারের security group-ও দেখো।

**ধাপ ৭ — অ্যাপ্লিকেশন:**

```bash
$ curl -v https://myserver.com/
$ curl -o /dev/null -s -w "%{http_code} %{time_total}\n" https://myserver.com/
502 0.031
```

`502` → nginx চলছে কিন্তু পেছনের অ্যাপ (PHP-FPM, Node ইত্যাদি) সাড়া দিচ্ছে না: `systemctl status`, `journalctl -u` দেখো ([অধ্যায় ২৬](26-systemd-services.md), [অধ্যায় ৪০](40-web-server-services.md))। `certificate has expired` বা `not yet valid` → সার্টিফিকেট নবায়ন করো, অথবা সিস্টেমের ঘড়ি ভুল কি না দেখো (`timedatectl`)।

### সাধারণ উপসর্গ → সম্ভাব্য কারণ

| উপসর্গ | প্রথমে কী সন্দেহ করবে | কোন কমান্ড |
|---|---|---|
| IP দিয়ে চলে, নাম দিয়ে চলে না | DNS | `dig`, `resolvectl status`, `/etc/resolv.conf` |
| একটা নির্দিষ্ট সাইট ভুল জায়গায় যাচ্ছে | `/etc/hosts`, DNS cache | `getent hosts site`, `resolvectl flush-caches` |
| ছোট পেজ খোলে, বড় পেজ/ফাইল আটকে যায় (বিশেষ করে VPN-এ) | MTU | `ping -M do -s 1472`, `tracepath` |
| মাঝে মাঝে ধীর, প্যাকেট হারায় | Wi-Fi সিগন্যাল, তার, ISP | `mtr`, `ip -s link`, `iw dev wlp2s0 link` |
| "Connection refused" | সার্ভিস বন্ধ বা ভুল IP-তে listen করছে | `ss -tlnp`, `systemctl status` |
| "Connection timed out" | firewall DROP | `nc -zv`, `ufw`/`firewalld`, cloud security group |
| "Address already in use" | অন্য প্রসেস port ধরে আছে | `ss -tlnp 'sport = :PORT'`, `lsof -i :PORT` |
| SSL/TLS error | সার্টিফিকেটের মেয়াদ, ভুল সময়, নাম না মেলা | `curl -v`, `timedatectl`, `openssl s_client` |
| হঠাৎ একটা নতুন হোস্টে যাওয়া যাচ্ছে না, বাকিগুলোয় যায় | route/ARP | `ip route get IP`, `ip neigh` |

> 📝 **মনে রাখো:** যা পরীক্ষা করছ তা লিখে রাখো (কোন ধাপে কী ফল এল)। ISP, হোস্টিং সাপোর্ট বা সহকর্মীকে জানাতে গেলে `mtr -rwn` রিপোর্ট আর `curl -v` আউটপুট দিলে সমস্যা অনেক দ্রুত সমাধান হয়।

---

## ৩৭.১৬ কীবোর্ড শর্টকাট (এই অধ্যায়ের টুলগুলোর জন্য)

| কী | কোথায় | কাজ |
|---|---|---|
| `Ctrl+C` | `ping`, `tcpdump`, `nc`, `ip monitor`, `traceroute` | থামাও (ping/tcpdump শেষে সারাংশ দেখায়) |
| `Ctrl+\` | `ping` (iputils) | না থামিয়ে এখন পর্যন্ত পরিসংখ্যান দেখাও |
| `Ctrl+D` | `nc` ক্লায়েন্ট, `nslookup` ইন্টারঅ্যাকটিভ | EOF পাঠাও / বের হও |
| `Ctrl+Z` | যেকোনো চলমান কমান্ড | সাময়িক থামিয়ে ব্যাকগ্রাউন্ডে রাখো (`fg`/`bg`, [অধ্যায় ২৫](25-jobs-tmux.md)) |
| `q` | `mtr`, `less`-এ খোলা আউটপুট | বের হও |
| `p` / `Space` | `mtr` | pause / resume |
| `r` | `mtr` | পরিসংখ্যান রিসেট |
| `n` | `mtr` | DNS নাম চালু/বন্ধ |
| `d` | `mtr` | ডিসপ্লে মোড বদল |
| `Ctrl+R` | shell | আগে চালানো লম্বা `curl`/`tcpdump` কমান্ড history থেকে খুঁজে বের করো ([অধ্যায় ৬](06-terminal-shortcuts.md)) |
| `Tab` | shell | `ip` সাব-কমান্ড আর interface নামের auto-complete (bash-completion থাকলে) |

> 💡 **টিপস:** কোনো কমান্ডের আউটপুট লাইভ আপডেট দেখতে `watch` ব্যবহার করো: `watch -n 1 'ss -s'` বা `watch -n 2 'ip -s link show enp3s0'`। বের হতে `Ctrl+C`।

---

## হাতে-কলমে অনুশীলন

> নিচের সব ল্যাব শুধু তোমার নিজের কম্পিউটার আর নিজের নেটওয়ার্কে করবে।

### ল্যাব ১: নিজের নেটওয়ার্কের পরিচয়পত্র বানাও

1. `ip -br -c addr` চালাও। তোমার প্রধান interface-এর নাম, IP আর `/` সংখ্যা লিখে রাখো।
2. `ip -br link` দিয়ে MAC address বের করো।
3. `ip route | grep default` দিয়ে gateway বের করো।
4. `ip route get 1.1.1.1` চালাও; কোন interface আর কোন source IP দিয়ে যাবে মিলিয়ে দেখো।
5. `ping -c 2 <gateway>` চালিয়ে তারপর `ip neigh show` দেখো; gateway-এর MAC আর state লিখে রাখো।
6. `resolvectl status` থেকে আসল DNS সার্ভার বের করো।
7. `hostname`, `hostname -I` আর `hostnamectl` চালাও।
8. `curl -s https://api.ipify.org; echo` দিয়ে পাবলিক IP বের করো। লোকাল IP থেকে আলাদা কেন? (উত্তর: NAT, [অধ্যায় ৩৬](36-networking-basics.md))

### ল্যাব ২: রাস্তা মাপো

1. `ping -c 5 8.8.8.8` আর `ping -c 5 google.com`। গড় RTT তুলনা করো।
2. `tracepath -n 8.8.8.8` চালাও। কতটা hop? pmtu কত?
3. `mtr -rwn -c 30 1.1.1.1` চালাও। কোনো hop-এ loss আছে? সেটা কি শেষ পর্যন্ত টিকে আছে, নাকি শুধু মাঝখানে? আসল সমস্যা কি না সিদ্ধান্ত নাও।
4. `ping -c 2 -M do -s 1472 8.8.8.8` আর `-s 1473` দিয়ে MTU-র সীমা নিজে প্রমাণ করো।

### ল্যাব ৩: একটা সার্ভার বানাও, খুঁজে বের করো, টেস্ট করো

1. টার্মিনাল ১-এ একটা ছোট ওয়েব সার্ভার চালাও:
   ```bash
   $ mkdir -p ~/netlab && cd ~/netlab && echo "<h1>Hello Linux</h1>" > index.html
   $ python3 -m http.server 8000
   Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
   ```
2. টার্মিনাল ২-এ খুঁজে বের করো: `ss -tlnp 'sport = :8000'` আর `lsof -i :8000`। কোন Local Address-এ listen করছে?
3. `curl -I http://localhost:8000/`, তারপর `curl -s -w "\n%{http_code}\n" http://localhost:8000/nope` চালাও। status code কী এল?
4. `nc -zv localhost 8000` (succeeded) আর `nc -zv localhost 8001` (refused) দেখো, পার্থক্য ব্যাখ্যা করো।
5. টার্মিনাল ৩-এ `sudo tcpdump -i lo -nn -A 'tcp port 8000'` চালিয়ে আবার `curl http://localhost:8000/` করো। `[S]`, `[S.]`, `[.]`, `[P.]` আর `GET / HTTP/1.1` লেখাটা খুঁজে বের করো।
6. সার্ভার বন্ধ করে (`Ctrl+C`) আবার `curl` চালাও। কী error এল? (`Connection refused`)

### ল্যাব ৪: DNS গোয়েন্দা

1. `dig example.com` চালাও; status, TTL আর SERVER লিখে রাখো। দুবার চালাও। দ্বিতীয়বার `Query time` আর TTL কেন কম?
2. `dig gmail.com MX +short`, `dig google.com NS +short`, `dig -x 1.1.1.1 +short`।
3. `dig nosuchdomain-xyz-12345.com` চালাও; status কী? (`NXDOMAIN`)
4. `sudo` দিয়ে `/etc/hosts`-এ `127.0.0.1 mytest.local` লাইন যোগ করো। তারপর `getent hosts mytest.local` আর `dig mytest.local +short` চালাও। পার্থক্যের কারণ কী? কাজ শেষে লাইনটা মুছে দাও।
5. `sudo tcpdump -i any -nn port 53` চালু রেখে অন্য টার্মিনালে `dig @8.8.8.8 example.com` চালাও আর প্রশ্ন-উত্তর প্যাকেট দেখো।

### ল্যাব ৫: netcat চ্যাট ও ফাইল ট্রান্সফার (একই মেশিনে দুই টার্মিনালে)

```bash
# টার্মিনাল ১
$ nc -l 5000
# টার্মিনাল ২
$ nc localhost 5000
```

দুই দিক থেকে লিখে দেখো। তারপর ফাইল পাঠাও:

```bash
# টার্মিনাল ১
$ nc -l 5001 > copy.txt
# টার্মিনাল ২
$ nc -N localhost 5001 < /etc/os-release
# যাচাই
$ diff /etc/os-release copy.txt && echo "একদম এক!"
```

(Fedora-র `ncat`-এ `-N` না চললে `--send-only` ব্যবহার করো।)

### ল্যাব ৬: নিজের মেশিনের নিরাপত্তা অডিট

1. `sudo ss -tulpn` চালাও। প্রতিটা listening port-এর জন্য লেখো: কোন প্রোগ্রাম, `127.0.0.1` না `0.0.0.0`, তোমার দরকার কি না।
2. `nmap localhost` আর `sudo nmap -sV localhost` চালিয়ে ফল মেলাও।
3. দরকার নেই এমন সার্ভিস পেলে নোট করো। [অধ্যায় ২৬](26-systemd-services.md)-এ শিখবে কীভাবে `systemctl disable --now` দিয়ে বন্ধ করতে হয়, আর [অধ্যায় ৪৫](45-security-hardening.md)-এ পুরো হার্ডেনিং।

### ল্যাব ৭: সমস্যা বানাও, সমাধান করো (VM বা নিজের ল্যাপটপে, রিমোট সার্ভারে নয়!)

1. DNS ভাঙো: `sudo resolvectl dns enp3s0 192.0.2.1` (192.0.2.0/24 ডকুমেন্টেশনের জন্য সংরক্ষিত, সেখানে কোনো DNS নেই)। এবার ৩৭.১৫-এর ধাপগুলো দিয়ে সমস্যা খুঁজে বের করো। ঠিক করতে: `sudo resolvectl revert enp3s0` (বা `nmcli connection up "<নাম>"`)।
2. route ভাঙো: `sudo ip route del default`। `ping 8.8.8.8` কী বলে? `ip route` দিয়ে কারণ বের করো। ঠিক করতে: `sudo ip route add default via <তোমার gateway>` বা NetworkManager দিয়ে সংযোগ আবার চালু করো।

---

## সারসংক্ষেপ

| কাজ | কমান্ড |
|---|---|
| IP দেখা (সংক্ষেপে, রঙিন) | `ip -br -c addr` |
| interface চালু/বন্ধ | `sudo ip link set dev IFACE up/down` |
| interface পরিসংখ্যান | `ip -s link show IFACE` |
| অস্থায়ী IP যোগ/মোছা | `sudo ip addr add/del 192.168.1.50/24 dev IFACE` |
| routing table | `ip route` |
| কোন পথে যাবে | `ip route get 8.8.8.8` |
| default gateway বসানো | `sudo ip route add default via GW` |
| ARP/neighbor টেবিল | `ip neigh` |
| লাইভ পরিবর্তন | `ip monitor` |
| hostname দেখা/বদলানো | `hostname -I`, `sudo hostnamectl set-hostname NAME` |
| হোস্ট বেঁচে আছে কি না | `ping -c 4 HOST` |
| MTU পরীক্ষা | `ping -M do -s 1472 HOST` |
| পথ দেখা | `traceroute -n HOST`, `tracepath HOST` |
| লাইভ পথ + loss | `mtr HOST`, `mtr -rwn -c 100 HOST` |
| listening port ও প্রসেস | `sudo ss -tulpn` |
| established সংযোগ | `ss -tn state established` |
| socket সারাংশ | `ss -s` |
| port কে ধরে আছে | `sudo lsof -i :PORT`, `lsof -t -i :PORT` |
| DNS প্রশ্ন | `dig NAME +short`, `dig @8.8.8.8 NAME`, `dig -x IP` |
| DNS-এর পুরো পথ | `dig +trace NAME` |
| সহজ DNS উত্তর | `host NAME`, `nslookup NAME` |
| সিস্টেম DNS | `resolvectl status`, `resolvectl flush-caches`, `getent hosts NAME` |
| HTTP হেডার | `curl -I URL` |
| redirect অনুসরণ | `curl -L URL` |
| ডাউনলোড | `curl -O URL`, `curl -o FILE URL`, `wget -c URL` |
| API রিকোয়েস্ট | `curl -X POST -H "Content-Type: application/json" -d '{..}' URL` |
| ডিবাগ | `curl -v URL` |
| status/সময় মাপা | `curl -o /dev/null -s -w "%{http_code} %{time_total}\n" URL` |
| পুরো সাইট mirror | `wget --mirror -k -p -np URL` |
| port খোলা কি না | `nc -zv HOST PORT` |
| port-এ শোনা | `nc -l PORT` |
| প্যাকেট ধরা | `sudo tcpdump -i any -nn port 53` |
| প্যাকেট ফাইলে | `sudo tcpdump -i IFACE -w file.pcap` |
| নিজের port স্ক্যান | `nmap localhost`, `sudo nmap -sV localhost` |
| LAN-এ কে আছে (নিজের) | `sudo nmap -sn 192.168.1.0/24` |
| ট্রাবলশুটিং ক্রম | link → IP → gateway → ইন্টারনেট IP → DNS → port → অ্যাপ |

---

## নিজেকে যাচাই করো

1. `ip` কমান্ড কেন `ifconfig`-এর চেয়ে ভালো? দুটো কার্নেলের সাথে কীভাবে কথা বলে?
2. `ip addr` আউটপুটে `UP` আর `LOWER_UP`-এর পার্থক্য কী? `NO-CARRIER` দেখলে কী বুঝবে?
3. `netstat -tulpn`, `route -n` আর `arp -n`-এর আধুনিক বিকল্প কী কী?
4. `ping google.com` বলছে `Temporary failure in name resolution`, কিন্তু `ping 8.8.8.8` কাজ করছে। সমস্যা কোথায়, আর পরের দুটো কমান্ড কী চালাবে?
5. `traceroute` ভেতরে কীভাবে প্রতিটা রাউটারের ঠিকানা জানে?
6. `mtr` রিপোর্টে hop ৪-এ ৫০% loss, কিন্তু শেষ hop-এ ০%। ইন্টারনেট কি খারাপ?
7. `ss -tulpn`-এ একটা লাইনে `127.0.0.1:5432` আর আরেকটায় `0.0.0.0:5432`। নিরাপত্তার দিক থেকে কোনটা কেমন?
8. `nc -zv server 443` চালালে "Connection refused" আর "timed out"-এর মানে কী আলাদা?
9. `curl`-এ `-I`, `-L`, `-o`, `-O`, `-X`, `-H`, `-d` প্রত্যেকটা কী করে? শুধু `-d` দিলে method কী হয়?
10. রিমোট সার্ভারে SSH দিয়ে ঢুকে `tcpdump` চালানোর সময় কোন ফিল্টার দেওয়া উচিত, আর কেন?

<details><summary>উত্তর দেখো</summary>

1. `ifconfig` (net-tools) পুরনো `ioctl()` দিয়ে কার্নেলের সাথে কথা বলে আর বহু বছর ধরে প্রায় স্থবির; একাধিক IP, policy routing, namespace ঠিকমতো দেখাতে পারে না। `ip` (iproute2) আধুনিক **Netlink socket** দিয়ে কথা বলে, সব নতুন ফিচার সাপোর্ট করে, আর একটা কমান্ডেই address, link, route, neigh সব সামলায়।
2. `UP` মানে সফটওয়্যারে (অ্যাডমিন) interface চালু করা হয়েছে; `LOWER_UP` মানে physical লিংক আছে (তার লাগানো/Wi-Fi যুক্ত)। `NO-CARRIER` মানে interface চালু, কিন্তু তার বা সিগন্যাল নেই।
3. `netstat -tulpn` → `ss -tulpn`; `route -n` → `ip route`; `arp -n` → `ip neigh`।
4. IP-তে চলছে মানে লিংক, IP, gateway আর ইন্টারনেট ঠিক আছে; সমস্যা **DNS**-এ। পরের কমান্ড: `resolvectl status` (কোন DNS সার্ভার সেট আছে) আর `dig @8.8.8.8 google.com` (অন্য DNS দিয়ে কাজ করে কি না)। এছাড়া `/etc/resolv.conf` দেখা যায়।
5. TTL ১, ২, ৩... করে প্যাকেট পাঠায়। প্রতিটা রাউটার TTL ১ কমায়, আর ০ হলে প্যাকেট ফেলে দিয়ে প্রেরককে ICMP "Time Exceeded" পাঠায়, যার উৎস ঠিকানাই ওই রাউটারের IP। গন্তব্যে পৌঁছালে অন্য ধরনের উত্তর (Port Unreachable/Echo Reply/SYN-ACK) আসে, তখন থামে।
6. না। loss শেষ পর্যন্ত টিকে না থাকলে সেটা আসল ক্ষতি নয়; hop ৪-এর রাউটার শুধু নিজের কাছে আসা ICMP-র উত্তর কম দিচ্ছে (rate-limit), অন্যদের ট্রাফিক ঠিকঠাক পার করছে। আসল loss হলে সেই hop থেকে শেষ পর্যন্ত সব hop-এ loss দেখা যেত।
7. `127.0.0.1:5432` শুধু ওই মেশিন থেকেই সংযোগ নেয়, তাই নিরাপদ। `0.0.0.0:5432` সব interface-এ খোলা; firewall না থাকলে নেটওয়ার্ক বা ইন্টারনেট থেকে যে কেউ ডেটাবেসে ঢোকার চেষ্টা করতে পারে, তাই ঝুঁকিপূর্ণ।
8. **Refused**: হোস্টে পৌঁছানো গেছে আর সে RST ফেরত দিয়েছে, মানে ওই port-এ কোনো সার্ভিস listen করছে না (বা অন্য IP-তে করছে)। **Timed out**: কোনো উত্তরই আসেনি; সাধারণত firewall প্যাকেট DROP করছে, বা হোস্ট/রাস্তাই নেই।
9. `-I` শুধু হেডার (HEAD); `-L` redirect অনুসরণ; `-o FILE` নির্দিষ্ট নামে সেভ; `-O` URL-এর নামে সেভ; `-X` HTTP method ঠিক করে; `-H` হেডার যোগ করে; `-d` body-তে ডেটা পাঠায়। শুধু `-d` দিলে method নিজে থেকেই **POST** হয় (Content-Type `application/x-www-form-urlencoded`)।
10. `not port 22` দেওয়া উচিত। নইলে tcpdump নিজের SSH সেশনের প্যাকেটও ধরবে, সেই আউটপুট আবার SSH দিয়ে পাঠাতে নতুন প্যাকেট তৈরি হবে, আর এক অনন্ত চক্রে স্ক্রিন ভরে যাবে।

</details>

---

[← আগের অধ্যায়](36-networking-basics.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](38-ssh-remote.md)

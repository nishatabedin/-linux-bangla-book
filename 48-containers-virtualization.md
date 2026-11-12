# অধ্যায় ৪৮: কন্টেইনার ও ভার্চুয়ালাইজেশন

## এই অধ্যায়ে যা শিখবে

- Virtualization (ভার্চুয়ালাইজেশন) কী, কেন দরকার, আর Virtual Machine (VM) ও Container-এর মধ্যে আসল পার্থক্য কোথায়
- Hypervisor কী — Type 1 আর Type 2-এর পার্থক্য
- KVM, QEMU, libvirt কীভাবে একসাথে কাজ করে, আর `virsh` / `virt-install` দিয়ে VM চালানো
- কন্টেইনার আসলে কোনো "জাদু" নয় — লিনাক্স kernel-এর কয়েকটা ফিচার জোড়া লাগিয়ে বানানো: `chroot`, **namespaces**, **cgroups v2**, **OverlayFS**
- `unshare` দিয়ে হাতে-কলমে নিজের "মিনি কন্টেইনার" বানানো
- Docker: ইনস্টল, image ও container, `run -d -p -v --name`, `ps`, `logs`, `exec`, `stop`, `rm`, volume, network
- `Dockerfile` লেখা, image build করা, multi-stage build
- `docker compose` দিয়ে একাধিক কন্টেইনার একসাথে চালানো
- Podman — daemon ছাড়া কন্টেইনার, আর rootless container কীভাবে কাজ করে
- Kubernetes এক নজরে — কী, কেন, আর মূল ধারণাগুলো
- কন্টেইনার ও VM-সংক্রান্ত সব দরকারি কীবোর্ড শর্টকাট

---

## ৪৮.১ ভার্চুয়ালাইজেশন কেন দরকার?

ধরো তোমার একটা বিশাল বাড়ি আছে (একটা শক্তিশালী সার্ভার — ৬৪ কোর CPU, ২৫৬ GB RAM)। তুমি সেখানে একটা মাত্র ওয়েবসাইট চালাচ্ছ, যেটা CPU-র ৫% ব্যবহার করে। বাকি ৯৫% অলস পড়ে আছে। আবার, তোমার আরেকটা অ্যাপ লাগবে যেটা পুরনো Python ভার্সন চায়, আর প্রথম অ্যাপটা নতুন ভার্সন চায় — একই মেশিনে দুজনের ঝগড়া!

সমাধান দুটো:

1. **Virtualization (ভার্চুয়ালাইজেশন):** বাড়িটাকে আলাদা আলাদা **ফ্ল্যাটে** ভাগ করো। প্রতিটা ফ্ল্যাটের নিজস্ব রান্নাঘর, বাথরুম, পানির লাইন — সম্পূর্ণ আলাদা সংসার। এটাই **Virtual Machine** — প্রতিটা VM-এর নিজের পুরো অপারেটিং সিস্টেম, নিজের kernel।
2. **Containerization (কন্টেইনারাইজেশন):** বাড়িটাকে **হোস্টেলের রুমে** ভাগ করো। প্রত্যেকের নিজের রুম, নিজের তালা — কিন্তু রান্নাঘর, পানির লাইন, বিদ্যুৎ সবাই শেয়ার করে। এটাই **Container** — সবাই হোস্টের একটাই kernel শেয়ার করে, শুধু চারপাশে দেয়াল তোলা।

> 📝 **মনে রাখো:** VM = আলাদা kernel সহ পুরো কম্পিউটারের নকল। Container = একই kernel, কিন্তু আলাদা করে রাখা (isolated) process।

### সুবিধাগুলো এক নজরে

| প্রয়োজন | ভার্চুয়ালাইজেশন/কন্টেইনার কীভাবে সাহায্য করে |
|---|---|
| হার্ডওয়্যারের পুরো ব্যবহার | এক মেশিনে অনেক VM/container চালানো যায় |
| আলাদা আলাদা পরিবেশ | প্রতিটা অ্যাপের নিজস্ব লাইব্রেরি ভার্সন |
| "আমার মেশিনে তো চলে!" সমস্যা | Container image-এ সব dependency বাঁধা থাকে, সব জায়গায় একইভাবে চলে |
| নিরাপদ পরীক্ষা | VM নষ্ট হলে মুছে আবার বানাও, হোস্টের কিছু হয় না |
| দ্রুত deploy ও scale | নতুন container সেকেন্ডে চালু হয় |
| ভিন্ন OS চালানো | লিনাক্সের ভেতরে Windows VM চালানো যায় (container দিয়ে যায় না) |

---

## ৪৮.২ VM বনাম Container — আর্কিটেকচার ডায়াগ্রাম

```
        VIRTUAL MACHINES                          CONTAINERS
 ┌─────────┐┌─────────┐┌─────────┐      ┌─────────┐┌─────────┐┌─────────┐
 │  App A  ││  App B  ││  App C  │      │  App A  ││  App B  ││  App C  │
 ├─────────┤├─────────┤├─────────┤      ├─────────┤├─────────┤├─────────┤
 │Bins/Libs││Bins/Libs││Bins/Libs│      │Bins/Libs││Bins/Libs││Bins/Libs│
 ├─────────┤├─────────┤├─────────┤      └────┬────┘└────┬────┘└────┬────┘
 │Guest OS ││Guest OS ││Guest OS │           │          │          │
 │(Kernel) ││(Kernel) ││(Kernel) │      ┌────┴──────────┴──────────┴────┐
 └────┬────┘└────┬────┘└────┬────┘      │  Container Runtime (Docker,   │
 ┌────┴──────────┴──────────┴────┐      │  Podman, containerd + runc)   │
 │          HYPERVISOR           │      ├───────────────────────────────┤
 │      (KVM, VMware, Xen...)    │      │   Host OS — একটাই Linux Kernel │
 ├───────────────────────────────┤      │ (namespaces + cgroups + overlay)│
 │   Host OS (Type 2 হলে থাকে)   │      ├───────────────────────────────┤
 ├───────────────────────────────┤      │           Hardware            │
 │           Hardware            │      └───────────────────────────────┘
 └───────────────────────────────┘
```

| বিষয় | Virtual Machine | Container |
|---|---|---|
| Kernel | প্রতিটা VM-এর নিজস্ব | হোস্টের kernel শেয়ার |
| আকার | কয়েক GB | কয়েক MB থেকে কয়েকশো MB |
| চালু হতে সময় | ৩০ সেকেন্ড – কয়েক মিনিট (পুরো OS বুট) | মিলিসেকেন্ড – কয়েক সেকেন্ড (শুধু একটা process) |
| Isolation (বিচ্ছিন্নতা) | খুব শক্তিশালী (হার্ডওয়্যার-লেভেল) | তুলনামূলক দুর্বল (kernel শেয়ার, kernel bug হলে ঝুঁকি) |
| ভিন্ন OS | হ্যাঁ — Linux-এ Windows, BSD চলে | না — লিনাক্স হোস্টে শুধু লিনাক্স container |
| Overhead | বেশি (প্রতিটার আলাদা RAM, kernel) | প্রায় শূন্য |
| এক সার্ভারে কতগুলো | ডজনখানেক | শত শত |
| ব্যবহার | ভিন্ন OS, শক্ত isolation, পুরনো সিস্টেম | microservices, CI/CD, অ্যাপ প্যাকেজিং |

> 💡 **টিপস:** বাস্তবে দুটো একসাথে ব্যবহার হয়। AWS/GCP-তে তুমি একটা VM ভাড়া নাও, আর সেই VM-এর ভেতরে Docker container চালাও।

> ⚠️ **সতর্কতা:** Mac বা Windows-এ Docker Desktop আসলে গোপনে একটা ছোট লিনাক্স VM চালায়, তার ভেতরে container চলে — কারণ লিনাক্স container-এর জন্য লিনাক্স kernel লাগবেই।

---

## ৪৮.৩ Hypervisor — VM-এর পরিচালক

**Hypervisor** (হাইপারভাইজার, অন্য নাম VMM — Virtual Machine Monitor) হলো সেই সফটওয়্যার যে আসল হার্ডওয়্যারকে ভাগ করে প্রতিটা VM-কে মনে করায় "এই পুরো কম্পিউটারটা আমার"। উপমা: একজন বাড়িওয়ালা, যে প্রতিটা ফ্ল্যাটের ভাড়াটেকে তার ভাগের বিদ্যুৎ-পানি মেপে দেয় আর কেউ যেন অন্যের ফ্ল্যাটে ঢুকতে না পারে তা দেখে।

### Type 1 বনাম Type 2

```
   TYPE 1 (Bare-metal)                 TYPE 2 (Hosted)
 ┌──────┐ ┌──────┐ ┌──────┐        ┌──────┐ ┌──────┐
 │ VM 1 │ │ VM 2 │ │ VM 3 │        │ VM 1 │ │ VM 2 │   ┌──────────┐
 └──┬───┘ └──┬───┘ └──┬───┘        └──┬───┘ └──┬───┘   │Browser,  │
 ┌──┴────────┴────────┴───┐        ┌──┴────────┴───┐   │অন্য অ্যাপ │
 │  Hypervisor            │        │ VirtualBox /  │   └────┬─────┘
 │  (ESXi, Xen, Hyper-V)  │        │ VMware Wkstn  │        │
 ├────────────────────────┤        ├───────────────┴────────┴──┐
 │       Hardware         │        │   Host OS (Windows/Linux) │
 └────────────────────────┘        ├───────────────────────────┤
                                   │          Hardware         │
                                   └───────────────────────────┘
```

| | Type 1 (Bare-metal) | Type 2 (Hosted) |
|---|---|---|
| কোথায় চলে | সরাসরি হার্ডওয়্যারের উপর | একটা সাধারণ OS-এর উপর অ্যাপ হিসেবে |
| উদাহরণ | VMware ESXi, Xen, Microsoft Hyper-V, **KVM** | Oracle VirtualBox, VMware Workstation, Parallels |
| পারফরম্যান্স | বেশি | তুলনামূলক কম |
| ব্যবহার | ডেটা সেন্টার, ক্লাউড | ডেস্কটপে শেখা/টেস্ট |

### KVM কোন টাইপ?

**KVM** (Kernel-based Virtual Machine) হলো লিনাক্স kernel-এর একটা মডিউল (`kvm.ko` আর `kvm_intel.ko` বা `kvm_amd.ko`)। এটা লোড হলে লিনাক্স kernel নিজেই hypervisor হয়ে যায়। যেহেতু এটা kernel-এর অংশ এবং সরাসরি হার্ডওয়্যার নিয়ন্ত্রণ করে, তাই KVM-কে সাধারণত **Type 1** ধরা হয় — যদিও লিনাক্স একসাথে সাধারণ OS হিসেবেও কাজ করে, তাই কেউ কেউ একে "হাইব্রিড" বলে। AWS (Nitro), Google Cloud — সবাই KVM-ভিত্তিক।

### হার্ডওয়্যার সাপোর্ট — VT-x / AMD-V

আধুনিক CPU-তে virtualization-এর জন্য বিশেষ নির্দেশ (instruction) থাকে: Intel-এ **VT-x** (`vmx` flag), AMD-তে **AMD-V** (`svm` flag)। এগুলো ছাড়া KVM চলবে না।

```bash
$ grep -Ec '(vmx|svm)' /proc/cpuinfo
8
```

০-এর বেশি হলে CPU সাপোর্ট করে (সংখ্যাটা মূলত logical CPU-র সংখ্যা)। ০ হলে BIOS/UEFI সেটিংসে গিয়ে "Intel Virtualization Technology" বা "SVM Mode" চালু করো।

```bash
$ lsmod | grep kvm
kvm_intel             487424  0
kvm                  1404928  1 kvm_intel

$ ls -l /dev/kvm
crw-rw----+ 1 root kvm 10, 232 Nov 12 09:10 /dev/kvm
```

Ubuntu/Debian-এ সহজ চেকার:

```bash
$ sudo apt install cpu-checker
$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

### KVM + QEMU + libvirt — কে কী করে?

এই তিনটা নাম প্রায় সবসময় একসাথে আসে, তাই ভূমিকা পরিষ্কার করে নিই:

```
 ┌────────────────────────────────────────────────────────────┐
 │  ব্যবহারকারী:  virt-manager (GUI)   virsh (CLI)  virt-install │
 └──────────────────────────┬─────────────────────────────────┘
                            │ (API কল)
 ┌──────────────────────────▼─────────────────────────────────┐
 │  libvirtd / virtqemud  — ম্যানেজমেন্ট ডেমন (libvirt)          │
 │  VM-এর XML কনফিগ, নেটওয়ার্ক, স্টোরেজ পুল সামলায়              │
 └──────────────────────────┬─────────────────────────────────┘
                            │ (প্রতিটা VM-এর জন্য একটা process চালায়)
 ┌──────────────────────────▼─────────────────────────────────┐
 │  qemu-system-x86_64  — USER SPACE                           │
 │  ভার্চুয়াল ডিস্ক, নেটওয়ার্ক কার্ড, USB, ডিসপ্লে ইত্যাদি        │
 │  ডিভাইস "নকল" (emulate) করে                                  │
 └──────────────────────────┬─────────────────────────────────┘
                            │ ioctl(/dev/kvm)
 ┌──────────────────────────▼─────────────────────────────────┐
 │  KVM kernel module  — KERNEL SPACE                          │
 │  CPU ও মেমরি virtualization (VT-x/AMD-V ব্যবহার করে)          │
 └──────────────────────────┬─────────────────────────────────┘
 ┌──────────────────────────▼─────────────────────────────────┐
 │              Hardware (CPU with VT-x / AMD-V)               │
 └────────────────────────────────────────────────────────────┘
```

| উপাদান | কাজ | উপমা |
|---|---|---|
| **KVM** | CPU আর RAM-কে নিরাপদে VM-এর হাতে তুলে দেয়, প্রায় native গতিতে | বাড়ির মূল কাঠামো আর বিদ্যুতের লাইন |
| **QEMU** | বাকি সব হার্ডওয়্যার (ডিস্ক, নেটওয়ার্ক কার্ড, মাউস, স্ক্রিন) emulate করে। KVM ছাড়াও একা চলতে পারে (তখন খুব ধীর, কিন্তু ARM-এর মতো অন্য আর্কিটেকচারও চালাতে পারে) | ফ্ল্যাটের আসবাবপত্র |
| **libvirt** | সব VM ম্যানেজ করার একটা কমন API ও ডেমন | বাড়ির ম্যানেজার অফিস |
| **virsh** | libvirt-এর কমান্ড-লাইন ক্লায়েন্ট | ম্যানেজারের সাথে কথা বলার ফোন |
| **virt-manager** | libvirt-এর GUI | ম্যানেজার অফিসের রিসেপশন ডেস্ক |

প্রতিটা চলমান VM হোস্টে আসলে একটা সাধারণ লিনাক্স process মাত্র — `ps aux | grep qemu` দিলে দেখতে পাবে। VM-এর প্রতিটা vCPU সেই process-এর একটা thread।

### KVM/libvirt ইনস্টল

```bash
# Ubuntu 24.04 / Debian 12
$ sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virtinst virt-manager
$ sudo usermod -aG libvirt,kvm $USER     # এরপর logout/login করো

# Fedora
$ sudo dnf install @virtualization
$ sudo systemctl enable --now libvirtd

# RHEL 9 / Rocky / Alma
$ sudo dnf install qemu-kvm libvirt virt-install virt-viewer
$ sudo systemctl enable --now libvirtd
```

> 📝 **মনে রাখো:** নতুন libvirt "modular daemon" ব্যবহার করে (`virtqemud`, `virtnetworkd` ইত্যাদি)। কিছু distro-তে `libvirtd` একক ডেমন, কিছুতে modular। `virsh` দুটোর সাথেই একইভাবে কাজ করে।

`virsh` দুই রকম "connection"-এ কাজ করে: `qemu:///system` (root-লেভেল, সিস্টেমের সব VM — `sudo` বা `libvirt` গ্রুপ লাগে) আর `qemu:///session` (শুধু তোমার ইউজারের VM)। `virsh` কোনটায় যাচ্ছে না বুঝলে স্পষ্ট করে বলো: `virsh -c qemu:///system list --all`।

### `qemu-img` — ভার্চুয়াল ডিস্ক

```bash
$ qemu-img create -f qcow2 ~/vms/test.qcow2 20G
Formatting '/home/rahim/vms/test.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=21474836480 lazy_refcounts=off refcount_bits=16

$ qemu-img info ~/vms/test.qcow2
image: /home/rahim/vms/test.qcow2
file format: qcow2
virtual size: 20 GiB (21474836480 bytes)
disk size: 196 KiB
...
```

**qcow2** (QEMU Copy-On-Write v2) ফরম্যাটের মজা হলো — ২০ GB "ভার্চুয়াল সাইজ" হলেও আসল ডিস্কে মাত্র ১৯৬ KB! ডেটা লেখার সাথে সাথে ফাইল বড় হয় (thin provisioning)। snapshot-ও সাপোর্ট করে। অন্য ফরম্যাট `raw` — দ্রুত, কিন্তু কোনো ফিচার নেই।

সরাসরি QEMU দিয়ে VM চালানো (libvirt ছাড়া — শেখার জন্য ভালো):

```bash
$ qemu-system-x86_64 -enable-kvm -m 2048 -smp 2 \
    -drive file=~/vms/test.qcow2,format=qcow2 \
    -cdrom ~/Downloads/ubuntu-24.04-live-server-amd64.iso -boot d
```

| অপশন | অর্থ |
|---|---|
| `-enable-kvm` (বা `-accel kvm`) | KVM hardware acceleration ব্যবহার করো (না দিলে খুব ধীর) |
| `-m 2048` | 2048 MB RAM |
| `-smp 2` | ২টা vCPU |
| `-drive file=...` | ভার্চুয়াল হার্ডডিস্ক |
| `-cdrom file.iso` | ISO-কে CD ড্রাইভ হিসেবে |
| `-boot d` | CD থেকে বুট করো |
| `-nographic` | কোনো উইন্ডো না, সিরিয়াল কনসোল টার্মিনালেই |

### `virt-install` — libvirt দিয়ে নতুন VM

```bash
$ sudo virt-install \
    --name ubuntu-vm \
    --memory 2048 \
    --vcpus 2 \
    --disk size=20 \
    --cdrom ~/Downloads/ubuntu-24.04-live-server-amd64.iso \
    --os-variant ubuntu24.04 \
    --network network=default \
    --graphics vnc
```

`--os-variant`-এর সঠিক নাম জানতে: `osinfo-query os | grep -i ubuntu` (`libosinfo-bin` প্যাকেজ)। `--disk size=20` দিলে libvirt ডিফল্ট storage pool-এ (সাধারণত `/var/lib/libvirt/images/`) ২০ GB-র qcow2 ডিস্ক বানিয়ে নেয়।

### `virsh` — VM নিয়ন্ত্রণের রিমোট কন্ট্রোল

```bash
$ sudo virsh list --all
 Id   Name        State
----------------------------
 1    ubuntu-vm   running
 -    debian-vm   shut off
```

| কমান্ড | কাজ |
|---|---|
| `virsh list` / `virsh list --all` | চলমান / সব VM দেখাও |
| `virsh start NAME` | VM চালু করো |
| `virsh shutdown NAME` | ভদ্রভাবে বন্ধ (ACPI সিগন্যাল — যেন পাওয়ার বাটন চাপলে) |
| `virsh destroy NAME` | জোর করে বন্ধ (পাওয়ার কর্ড টানার মতো — ডিস্ক মোছে **না**) |
| `virsh reboot NAME` | রিবুট |
| `virsh suspend NAME` / `virsh resume NAME` | VM থামিয়ে রাখো (RAM-এ) / আবার চালাও |
| `virsh dominfo NAME` | VM-এর তথ্য (CPU, RAM, state) |
| `virsh domifaddr NAME` | VM-এর IP ঠিকানা |
| `virsh console NAME` | সিরিয়াল কনসোলে ঢোকো (বের হতে `Ctrl+]`) |
| `virsh autostart NAME` | হোস্ট বুট হলে VM-ও চালু হবে |
| `virsh edit NAME` | VM-এর XML কনফিগ এডিটরে খোলো |
| `virsh dumpxml NAME` | XML কনফিগ দেখাও |
| `virsh snapshot-create-as NAME snap1` | snapshot নাও |
| `virsh snapshot-list NAME` | snapshot তালিকা |
| `virsh snapshot-revert NAME snap1` | snapshot-এ ফিরে যাও |
| `virsh undefine NAME` | VM-এর সংজ্ঞা মুছে ফেলো (`--remove-all-storage` দিলে ডিস্কও) |
| `virsh net-list --all` | ভার্চুয়াল নেটওয়ার্ক তালিকা |
| `virsh pool-list --all` | storage pool তালিকা |

> ⚠️ **সতর্কতা:** `virsh destroy` নাম শুনে ভয় পেও না — এটা VM মুছে ফেলে না, শুধু হঠাৎ বন্ধ করে। কিন্তু `virsh undefine --remove-all-storage` সত্যিই সব মুছে ফেলে, ফিরিয়ে আনা যায় না।

libvirt-এর `default` নেটওয়ার্ক একটা NAT নেটওয়ার্ক — হোস্টে `virbr0` নামে একটা ব্রিজ ইন্টারফেস বানায় (সাধারণত `192.168.122.0/24`), আর `dnsmasq` দিয়ে VM-দের IP দেয়। VM বাইরের ইন্টারনেটে যেতে পারে, কিন্তু বাইরে থেকে সরাসরি VM-এ আসা যায় না (bridged নেটওয়ার্ক বানালে যায়)। নেটওয়ার্কিং-এর মূলনীতির জন্য দেখো [অধ্যায় ৩৬](36-networking-basics.md)।

> 💡 **টিপস:** GUI পছন্দ হলে `virt-manager` চালাও — ক্লিক করে VM বানানো, কনসোল দেখা, snapshot নেওয়া সব করা যায়। GNOME-এ সহজ বিকল্প **GNOME Boxes**।

---

## ৪৮.৪ কন্টেইনার আসলে কী দিয়ে তৈরি?

এখানেই সবচেয়ে গুরুত্বপূর্ণ কথাটা: **লিনাক্স kernel-এ "container" নামে কোনো জিনিস নেই।** Container হলো একটা সাধারণ process, যাকে kernel-এর কয়েকটা ফিচার দিয়ে ঘিরে রাখা হয়েছে:

```
            একটা "Container" = সাধারণ লিনাক্স process +
 ┌───────────────────────────────────────────────────────────┐
 │  NAMESPACES  → সে কী "দেখতে" পায়                          │
 │               (নিজের PID তালিকা, নিজের নেটওয়ার্ক, নিজের hostname) │
 ├───────────────────────────────────────────────────────────┤
 │  CGROUPS     → সে কতটা "ব্যবহার" করতে পারে                  │
 │               (সর্বোচ্চ RAM, CPU, process সংখ্যা)             │
 ├───────────────────────────────────────────────────────────┤
 │  ROOT FS     → তার "দুনিয়া" কোন ফোল্ডার                      │
 │  (chroot/pivot_root + OverlayFS image layers)             │
 ├───────────────────────────────────────────────────────────┤
 │  SECURITY    → সে কী "করতে" পারে                           │
 │  (capabilities, seccomp, AppArmor/SELinux)                │
 └───────────────────────────────────────────────────────────┘
```

হোস্টেলের উপমায়:
- **Namespace** = রুমের দেয়াল আর পর্দা — তুমি পাশের রুমের লোকদের দেখতে পাও না।
- **cgroup** = বিদ্যুতের মিটার আর ফিউজ — তুমি তোমার বরাদ্দের বেশি বিদ্যুৎ টানতে পারবে না।
- **chroot/OverlayFS** = তোমার রুমের আসবাব — তোমার জগৎ ওই রুমটুকু।
- **capabilities/seccomp** = হোস্টেলের নিয়মকানুন — তুমি মেইন সুইচ বন্ধ করতে পারবে না।

চলো একটা একটা করে দেখি।

### ৪৮.৪.১ chroot — সবচেয়ে পুরনো "কারাগার"

`chroot` (change root, ১৯৭৯ সালের Unix V7 থেকে আছে) একটা process-এর কাছে `/` (root ডিরেক্টরি) কোথায় সেটা বদলে দেয়। process মনে করে একটা ফোল্ডারই পুরো ফাইল সিস্টেম।

```bash
$ mkdir -p ~/jail
$ cp --parents /bin/bash /bin/ls ~/jail/
$ for lib in $(ldd /bin/bash /bin/ls | grep -o '/[^ ]*'); do
>     cp --parents "$lib" ~/jail/
> done
$ sudo chroot ~/jail /bin/bash
bash-5.2# ls /
bin  lib  lib64
bash-5.2# cd ..
bash-5.2# pwd
/
bash-5.2# exit
```

এখানে `ldd` দেখায় `bash` আর `ls` কোন কোন shared library-র উপর নির্ভর করে, আর `cp --parents` পুরো পাথসহ কপি করে। jail-এর ভেতরে `cd ..` দিয়েও বাইরে যাওয়া যায় না।

কিন্তু chroot একা যথেষ্ট নয়:
- jail-এর ভেতরের process এখনও `ps` চালালে (যদি `/proc` মাউন্ট করা থাকে) হোস্টের সব process দেখবে।
- নেটওয়ার্ক পুরোটাই হোস্টের।
- root ইউজার chroot থেকে কিছু কৌশলে বেরিয়ে আসতে পারে।
- কোনো রিসোর্স লিমিট নেই।

তাই আধুনিক container runtime `chroot`-এর বদলে নিরাপদ `pivot_root` system call ব্যবহার করে, আর তার সাথে যোগ করে namespaces ও cgroups।

### ৪৮.৪.২ Namespaces — কে কী দেখতে পাবে

**Namespace** হলো kernel-এর একটা ফিচার যা কোনো একটা গ্লোবাল রিসোর্সকে "মোড়ক" দিয়ে আলাদা করে — যাতে namespace-এর ভেতরের process-রা মনে করে রিসোর্সটা শুধু তাদেরই।

| Namespace | `unshare` অপশন | কী আলাদা করে | উদাহরণ |
|---|---|---|---|
| **mnt** (Mount) | `--mount` / `-m` | মাউন্ট পয়েন্টের তালিকা | container-এর নিজের `/` |
| **pid** (Process ID) | `--pid` / `-p` | process ID নম্বর | container-এর প্রথম process-এর PID = 1 |
| **net** (Network) | `--net` / `-n` | নেটওয়ার্ক ইন্টারফেস, IP, routing table, firewall rule, port | container-এর নিজের `eth0`, নিজের port 80 |
| **uts** (UNIX Time-Sharing) | `--uts` / `-u` | hostname ও domain name | container-এর আলাদা hostname |
| **ipc** (Inter-Process Comm.) | `--ipc` / `-i` | System V IPC, POSIX message queue, shared memory | এক container অন্যটার shared memory পায় না |
| **user** | `--user` / `-U` | UID/GID ম্যাপিং | ভেতরে root (UID 0), বাইরে সাধারণ ইউজার |
| **cgroup** | `--cgroup` / `-C` | cgroup ট্রি-র দৃশ্য | container নিজের cgroup-কেই root দেখে |
| **time** | `--time` / `-T` | `CLOCK_MONOTONIC` ও `CLOCK_BOOTTIME` অফসেট | container migration-এ uptime ঠিক রাখা |

> 📝 **মনে রাখো:** `time` namespace-ও ঘড়ির "wall-clock" সময় (তারিখ/সময়) বদলাতে পারে না — শুধু monotonic আর boottime ঘড়ি।

প্রতিটা process কোন namespace-এ আছে তা `/proc/PID/ns/`-এ দেখা যায়:

```bash
$ ls -l /proc/$$/ns
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:00 ipc -> 'ipc:[4026531839]'
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:00 mnt -> 'mnt:[4026531841]'
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:00 net -> 'net:[4026531840]'
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:00 pid -> 'pid:[4026531836]'
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:00 pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:00 time -> 'time:[4026531834]'
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:00 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:00 user -> 'user:[4026531837]'
lrwxrwxrwx 1 rahim rahim 0 Nov 12 10:00 uts -> 'uts:[4026531838]'
```

বন্ধনীর সংখ্যাটা namespace-এর পরিচয় (inode নম্বর)। দুটো process-এর সংখ্যা একই হলে তারা একই namespace-এ। (`$$` = বর্তমান shell-এর PID; `/proc` নিয়ে বিস্তারিত [অধ্যায় ৪৭](47-kernel-proc-sys.md)-এ।)

সিস্টেমের সব namespace দেখতে `lsns`:

```bash
$ sudo lsns -t net
        NS TYPE NPROCS   PID USER      NETNSID NSFS COMMAND
4026531840 net     312     1 root   unassigned      /sbin/init splash
4026532501 net       2  4120 root            0      nginx: master process nginx -g daemon off;
```

#### Namespace-এর পেছনের system call

| System call | কাজ |
|---|---|
| `clone()` / `clone3()` | নতুন process বানাও, চাইলে নতুন namespace-এ (`CLONE_NEWPID`, `CLONE_NEWNET` ইত্যাদি flag) |
| `unshare()` | বর্তমান process-কে নতুন namespace-এ সরিয়ে নাও |
| `setns()` | একটা বিদ্যমান namespace-এ ঢোকো (`docker exec` এটাই করে!) |

#### `unshare` ডেমো — নিজের হাতে container বানাও

`unshare` কমান্ড (util-linux প্যাকেজ) নতুন namespace-এ একটা প্রোগ্রাম চালায়।

**ধাপ ১ — UTS namespace (আলাদা hostname):**

```bash
$ hostname
my-laptop
$ sudo unshare --uts bash
root@my-laptop:~# hostname container1
root@my-laptop:~# hostname
container1
root@my-laptop:~# exit
$ hostname
my-laptop          # হোস্টের hostname অক্ষত!
```

(prompt-এ পুরনো নাম দেখাচ্ছে কারণ prompt shell শুরুর সময় তৈরি; নতুন shell খুললে নতুন নাম দেখাবে।)

**ধাপ ২ — PID namespace (নিজের process ট্রি):**

```bash
$ sudo unshare --pid --fork --mount-proc bash
root@my-laptop:~# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   9184  5248 pts/2    S    10:05   0:00 bash
root           8  0.0  0.0  11320  4608 pts/2    R+   10:05   0:00 ps aux
root@my-laptop:~# exit
```

এখানে:
- `--pid` — নতুন PID namespace।
- `--fork` — দরকারি! `unshare` নিজে নতুন PID namespace-এ ঢোকে না, শুধু তার *child*-রা ঢোকে। তাই fork করে child হিসেবে bash চালাতে হয়, যে PID 1 পায়।
- `--mount-proc` — নতুন mount namespace বানিয়ে সেখানে নতুন `/proc` মাউন্ট করে। নাহলে `ps` পুরনো `/proc` পড়ে হোস্টের সব process দেখাত।

দেখো — container-এর ভেতরে bash হলো **PID 1**, আর মাত্র দুটো process দেখা যাচ্ছে! অথচ হোস্ট থেকে (অন্য টার্মিনালে) `ps aux | grep bash` দিলে এই bash-কে সাধারণ একটা বড় PID (যেমন 5123) হিসেবে দেখবে। **একই process, দুই জগতে দুই নম্বর।**

> 📝 **মনে রাখো:** PID 1 বিশেষ — সে মারা গেলে পুরো PID namespace-এর সব process kernel মেরে ফেলে। এজন্যই container-এর মূল process বন্ধ হলে container-ও বন্ধ হয়ে যায়।

**ধাপ ৩ — Network namespace:**

```bash
$ sudo unshare --net bash
root@my-laptop:~# ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
root@my-laptop:~# ping -c1 8.8.8.8
ping: connect: Network is unreachable
root@my-laptop:~# exit
```

নতুন network namespace সম্পূর্ণ খালি — শুধু একটা বন্ধ (DOWN) loopback। কোনো ইন্টারনেট নেই। Docker এখানে একটা **veth pair** (ভার্চুয়াল ইথারনেট তার — এক মাথা container-এ, অন্য মাথা হোস্টের ব্রিজে) জুড়ে দেয়।

`ip netns` দিয়ে নাম দেওয়া network namespace বানানো ও জোড়া লাগানো যায়:

```bash
$ sudo ip netns add red
$ sudo ip link add veth-host type veth peer name veth-red
$ sudo ip link set veth-red netns red
$ sudo ip addr add 10.0.0.1/24 dev veth-host
$ sudo ip link set veth-host up
$ sudo ip netns exec red ip addr add 10.0.0.2/24 dev veth-red
$ sudo ip netns exec red ip link set veth-red up
$ sudo ip netns exec red ip link set lo up
$ ping -c2 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.061 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.048 ms
$ sudo ip netns delete red        # পরিষ্কার করো (veth জোড়াও মুছে যায়)
```

```
   HOST network namespace          "red" network namespace
 ┌─────────────────────────┐      ┌─────────────────────────┐
 │  veth-host 10.0.0.1  ●──┼──────┼──● veth-red 10.0.0.2    │
 │                         │ veth │                         │
 │  eth0 (ইন্টারনেট)        │ pair │  lo                     │
 └─────────────────────────┘      └─────────────────────────┘
```

এটাই হুবহু Docker-এর bridge নেটওয়ার্কের মূলনীতি — শুধু Docker মাঝখানে `docker0` নামে একটা ব্রিজ রাখে আর NAT rule যোগ করে।

**ধাপ ৪ — সব একসাথে:**

```bash
$ sudo unshare --uts --pid --net --mount --ipc --fork --mount-proc bash
root@my-laptop:~# hostname mini-container
root@my-laptop:~# ps aux | wc -l
3
```

অভিনন্দন — তুমি Docker ছাড়াই প্রায় একটা container বানিয়ে ফেলেছ! বাকি শুধু: আলাদা root ফাইলসিস্টেম, রিসোর্স লিমিট (cgroups), আর নিরাপত্তা নিয়ম।

**ধাপ ৫ — User namespace (root ছাড়া "root"):**

```bash
$ id
uid=1000(rahim) gid=1000(rahim) groups=1000(rahim),27(sudo)
$ unshare --user --map-root-user bash
root@my-laptop:~# id
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
root@my-laptop:~# touch /etc/hack
touch: cannot touch '/etc/hack': Permission denied
root@my-laptop:~# exit
```

`sudo` ছাড়াই ভেতরে তুমি "root"! কিন্তু এই root শুধু namespace-এর ভেতরে — বাইরের দুনিয়ায় সে এখনও UID 1000, তাই হোস্টের `/etc`-তে লিখতে পারে না। এটাই **rootless container**-এর ভিত্তি (৪৮.১০-এ বিস্তারিত)।

> ⚠️ **সতর্কতা (Ubuntu 23.10+/24.04):** Ubuntu AppArmor দিয়ে unprivileged user namespace সীমিত করে (`kernel.apparmor_restrict_unprivileged_userns = 1`)। উপরের `id` ডেমো চলবে, কিন্তু namespace-এর ভেতরে mount-এর মতো ক্ষমতা কাজ নাও করতে পারে। Podman/Docker rootless-এর জন্য প্যাকেজের সাথে AppArmor প্রোফাইল আসে, তাই সেগুলো ঠিকমতো চলে।

#### `nsenter` — চলমান container-এ ঢোকা

```bash
$ PID=$(docker inspect -f '{{.State.Pid}}' web)
$ sudo nsenter --target "$PID" --net ip addr
```

এটা `web` container-এর network namespace-এ ঢুকে **হোস্টের** `ip` কমান্ড চালায় — container image-এ `ip` টুল না থাকলেও! ডিবাগিংয়ে খুব কাজের। `docker exec` ভেতরে এই একই `setns()` কৌশল ব্যবহার করে।

### ৪৮.৪.৩ cgroups v2 — কে কতটা ব্যবহার করবে

Namespace ঠিক করে process কী **দেখবে**; কিন্তু একটা container যদি সার্ভারের সব RAM খেয়ে ফেলে? তখন দরকার **cgroups** (Control Groups) — process-দের গ্রুপে ভাগ করে প্রতিটা গ্রুপের রিসোর্স ব্যবহার **সীমিত, মাপা ও নিয়ন্ত্রণ** করার kernel ফিচার।

উপমা: হোস্টেলের প্রতিটা রুমে আলাদা বিদ্যুৎ মিটার আর ফিউজ। কেউ বেশি টানলে তার রুমের ফিউজ কাটবে, পুরো হোস্টেলের না।

**v1 বনাম v2:** পুরনো cgroups v1-এ প্রতিটা রিসোর্সের (cpu, memory…) আলাদা আলাদা ট্রি ছিল — জটিল ও বিশৃঙ্খল। **cgroups v2**-এ একটাই ইউনিফাইড ট্রি। Ubuntu 22.04+, Debian 11+, Fedora 31+, RHEL 9 — সবাই ডিফল্টে v2 ব্যবহার করে।

```bash
$ stat -fc %T /sys/fs/cgroup
cgroup2fs          # v2 (যদি "tmpfs" দেখায়, তাহলে v1 বা hybrid)

$ cat /sys/fs/cgroup/cgroup.controllers
cpuset cpu io memory hugetlb pids rdma misc
```

cgroup ট্রি আসলে `/sys/fs/cgroup`-এ মাউন্ট করা একটা ভার্চুয়াল ফাইলসিস্টেম। **ফোল্ডার বানানো = নতুন cgroup বানানো**, আর ফাইলে লেখা = লিমিট সেট করা।

```
 /sys/fs/cgroup/                         ← root cgroup
 ├── cgroup.controllers                  ← কোন কোন controller আছে
 ├── cgroup.subtree_control              ← child-দের জন্য কোনগুলো চালু
 ├── system.slice/                       ← systemd সার্ভিসগুলো
 │   ├── docker.service/
 │   ├── ssh.service/
 │   └── docker-4f3a...e1.scope/         ← একটা Docker container!
 │       ├── cgroup.procs                ← এই গ্রুপের PID তালিকা
 │       ├── memory.max                  ← RAM লিমিট
 │       ├── memory.current              ← এখন কত RAM ব্যবহার করছে
 │       ├── cpu.max                     ← CPU লিমিট
 │       └── pids.max                    ← সর্বোচ্চ process সংখ্যা
 └── user.slice/                         ← লগইন করা ইউজারদের process
```

গুরুত্বপূর্ণ ফাইলগুলো:

| ফাইল | অর্থ | উদাহরণ মান |
|---|---|---|
| `cgroup.procs` | এই cgroup-এর process-দের PID; PID লিখলে process এখানে সরে আসে | `4120` |
| `memory.max` | সর্বোচ্চ RAM; বেশি হলে OOM killer মারে | `512M` বা `max` |
| `memory.high` | "নরম" সীমা — পেরোলে kernel জোর করে মেমরি ফেরত নেয় (throttle) | `400M` |
| `memory.current` | বর্তমান ব্যবহার (শুধু পড়া যায়) | `104857600` |
| `memory.swap.max` | সর্বোচ্চ swap | `0` |
| `cpu.max` | `QUOTA PERIOD` (মাইক্রোসেকেন্ড): প্রতি PERIOD-এ সর্বোচ্চ QUOTA সময় CPU | `50000 100000` = আধা CPU |
| `cpu.weight` | CPU ভাগাভাগির অনুপাত (১–১০০০০, ডিফল্ট ১০০) | `200` = দ্বিগুণ ভাগ |
| `io.max` | ডিস্ক I/O লিমিট (device অনুযায়ী) | `8:0 rbps=1048576` |
| `pids.max` | সর্বোচ্চ process/thread সংখ্যা (fork bomb ঠেকায়) | `100` |
| `cpu.stat`, `memory.stat` | পরিসংখ্যান | — |

#### হাতে-কলমে: নিজের cgroup বানাও

```bash
$ sudo mkdir /sys/fs/cgroup/demo
$ ls /sys/fs/cgroup/demo | head -5
cgroup.controllers
cgroup.events
cgroup.freeze
cgroup.kill
cgroup.max.depth

$ echo 100M | sudo tee /sys/fs/cgroup/demo/memory.max
100M
$ echo "20000 100000" | sudo tee /sys/fs/cgroup/demo/cpu.max     # ২০% CPU
20000 100000
$ echo 20 | sudo tee /sys/fs/cgroup/demo/pids.max
20

# বর্তমান shell-কে এই cgroup-এ ঢোকাও
$ echo $$ | sudo tee /sys/fs/cgroup/demo/cgroup.procs
$ cat /proc/$$/cgroup
0::/demo
```

এখন এই shell থেকে যা-ই চালাবে, সব মিলিয়ে ১০০ MB RAM, ২০% CPU, আর ২০টার বেশি process পাবে না। যেমন `yes > /dev/null` চালিয়ে অন্য টার্মিনালে `top` দেখলে দেখবে ওটা ~২০% CPU-তে আটকে আছে।

পরিষ্কার করতে: shell বন্ধ করো (cgroup খালি হবে), তারপর `sudo rmdir /sys/fs/cgroup/demo`। (খালি না হলে `rmdir` "Device or resource busy" দেবে।)

> 💡 **টিপস:** হাতে ফাইল না লিখে systemd দিয়েই একই কাজ করা যায় — যে কোনো কমান্ডকে লিমিটসহ চালাও:
> ```bash
> $ systemd-run --user --scope -p MemoryMax=200M -p CPUQuota=30% stress-ng --cpu 2
> ```
> আর পুরো cgroup ট্রি দেখতে `systemd-cgls`, লাইভ রিসোর্স ব্যবহার দেখতে `systemd-cgtop`। systemd নিয়ে বিস্তারিত [অধ্যায় ২৬](26-systemd-services.md)-এ।

Docker যখন `docker run --memory 512m --cpus 1.5` চালায়, ঠিক এই ফাইলগুলোতেই লেখে: `memory.max = 536870912` আর `cpu.max = 150000 100000`।

### ৪৮.৪.৪ OverlayFS — স্তরে স্তরে ফাইলসিস্টেম

একটা Ubuntu image ~৮০ MB। ১০০টা container চালালে কি ৮ GB ডিস্ক লাগবে? না! কারণ **OverlayFS** (union filesystem)।

উপমা: একটা ছাপানো বইয়ের উপর **ট্রেসিং পেপার** রাখো। তুমি ট্রেসিং পেপারে যা খুশি লেখো, কাটো — নিচের বই অক্ষত থাকে। উপর থেকে তাকালে দুটো মিলিয়ে একটা পাতা দেখায়। ১০০ জন ছাত্র একই বই শেয়ার করতে পারে, প্রত্যেকের শুধু নিজের ট্রেসিং পেপার।

```
          merged  (container যা দেখে — একীভূত দৃশ্য)
 ┌──────────────────────────────────────────────────┐
 │  /etc/nginx.conf(নতুন)  /app/data  /bin/sh  ...    │
 └──────────────────────────────────────────────────┘
                     ▲  উপরের স্তর জেতে
 ┌──────────────────────────────────────────────────┐
 │ upperdir  (লেখা যায়, container-এর নিজস্ব)          │ ← container layer
 │   etc/nginx.conf (পরিবর্তিত কপি), app/data          │
 ├──────────────────────────────────────────────────┤
 │ lowerdir 3 (read-only)  — COPY app/ ...           │ ┐
 ├──────────────────────────────────────────────────┤ │ image layers
 │ lowerdir 2 (read-only)  — apt install nginx       │ │ (সব container
 ├──────────────────────────────────────────────────┤ │  শেয়ার করে)
 │ lowerdir 1 (read-only)  — ubuntu base             │ ┘
 └──────────────────────────────────────────────────┘
```

নিয়মগুলো:
- **পড়া:** ফাইল উপরের স্তরে থাকলে সেটা, নাহলে নিচের স্তর থেকে।
- **লেখা (Copy-on-Write):** নিচের স্তরের কোনো ফাইল বদলালে আগে পুরো ফাইলটা upperdir-এ কপি হয়, তারপর বদলায়। নিচেরটা কখনো বদলায় না।
- **মোছা:** নিচের স্তরের ফাইল মুছলে upperdir-এ একটা **whiteout** ফাইল (বিশেষ character device, 0/0) তৈরি হয়, যা ওই ফাইলকে "লুকিয়ে" রাখে।

#### নিজে OverlayFS মাউন্ট করো

```bash
$ mkdir -p ~/ov/{lower,upper,work,merged}
$ echo "আমি নিচের স্তরে" > ~/ov/lower/a.txt
$ echo "আমিও নিচে"     > ~/ov/lower/b.txt

$ sudo mount -t overlay overlay \
    -o lowerdir=$HOME/ov/lower,upperdir=$HOME/ov/upper,workdir=$HOME/ov/work \
    ~/ov/merged

$ ls ~/ov/merged
a.txt  b.txt

$ echo "বদলে দিলাম" | sudo tee ~/ov/merged/a.txt
$ sudo rm ~/ov/merged/b.txt
$ sudo touch ~/ov/merged/c.txt

$ cat ~/ov/lower/a.txt
আমি নিচের স্তরে                 # নিচের স্তর অক্ষত!
$ ls -l ~/ov/upper
-rw-r--r-- 1 rahim rahim    29 Nov 12 11:00 a.txt   # পরিবর্তিত কপি
c---------  1 root  root  0, 0 Nov 12 11:00 b.txt   # whiteout!
-rw-r--r-- 1 root  root     0 Nov 12 11:00 c.txt   # নতুন ফাইল

$ sudo umount ~/ov/merged
```

| অপশন | কাজ |
|---|---|
| `lowerdir=` | read-only স্তর(গুলো); একাধিক হলে `:` দিয়ে, **বামেরটা সবচেয়ে উপরে** |
| `upperdir=` | লেখার স্তর |
| `workdir=` | kernel-এর অভ্যন্তরীণ কাজের জন্য খালি ডিরেক্টরি; upperdir-এর সাথে একই ফাইলসিস্টেমে হতে হবে |

Docker-এর ক্লাসিক storage driver `overlay2` ঠিক এটাই করে। চলমান container-এর মাউন্ট দেখো:

```bash
$ mount | grep overlay
overlay on /var/lib/docker/overlay2/7c1e.../merged type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/ABC...:/var/lib/docker/overlay2/l/DEF...,upperdir=...,workdir=...)
```

> 📝 **মনে রাখো:** container মুছলে তার upperdir মুছে যায় — অর্থাৎ container-এর ভেতরে লেখা সব ডেটা হারিয়ে যায়! স্থায়ী ডেটার জন্য **volume** লাগে (৪৮.৬.৬-এ দেখব)।

> 💡 **টিপস:** নতুন Docker ভার্সনে (v29 থেকে নতুন ইনস্টলে) image রাখার দায়িত্ব containerd-এর "image store"-কে দেওয়া হয়েছে, তখন `docker info`-তে storage driver হিসেবে `overlayfs` (containerd snapshotter) দেখাতে পারে। নাম যা-ই হোক, ভেতরের কৌশল এই OverlayFS-ই।

### ৪৮.৪.৫ নিরাপত্তার স্তর — capabilities, seccomp, LSM

Namespace আর cgroup ছাড়াও runtime আরও কয়েকটা দেয়াল দেয়:

| স্তর | কাজ |
|---|---|
| **Capabilities** | root-এর ক্ষমতাকে ~৪০টা টুকরোয় ভাগ করা (`CAP_NET_ADMIN`, `CAP_SYS_ADMIN`…)। Docker ডিফল্টে container-এর root-কে মাত্র ~১৪টা দেয়। |
| **seccomp** | কোন system call চালানো যাবে তার ফিল্টার। Docker-এর ডিফল্ট প্রোফাইল ৪০+ বিপজ্জনক syscall বন্ধ রাখে (যেমন `reboot`, kernel module লোড)। |
| **AppArmor / SELinux** | Mandatory Access Control — Ubuntu-তে AppArmor, Fedora/RHEL-এ SELinux। বিস্তারিত [অধ্যায় ৪৫](45-security-hardening.md)-এ। |
| **read-only rootfs** | `--read-only` দিলে container নিজের ফাইলসিস্টেমে লিখতে পারে না। |

> ⚠️ **সতর্কতা:** `docker run --privileged` এই সব দেয়াল ভেঙে দেয় — container প্রায় হোস্টের root-এর সমান ক্ষমতা পায়। একান্ত দরকার না হলে কখনো ব্যবহার করবে না।

---

## ৪৮.৫ Container ইকোসিস্টেম — কে কোন কাজ করে

### Image বনাম Container

| | Image | Container |
|---|---|---|
| কী | read-only টেমপ্লেট (স্তরের সমষ্টি + মেটাডেটা) | image থেকে চালানো একটা জীবন্ত instance |
| উপমা | রান্নার রেসিপি / class | রান্না করা খাবার / object |
| অবস্থা | স্থির | চলে, থামে, মরে |
| সংখ্যা | একটা image | তা থেকে অসংখ্য container |

**Registry** হলো image-এর লাইব্রেরি/অ্যাপ-স্টোর: Docker Hub (`docker.io`), GitHub (`ghcr.io`), Quay (`quay.io`), নিজস্ব প্রাইভেট registry।

Image-এর পুরো নাম: `registry/namespace/repository:tag`

```
 docker.io / library / nginx : 1.27-alpine
 └registry┘ └namespace┘ └repo┘ └── tag ──┘
```

`nginx` লিখলে আসলে মানে `docker.io/library/nginx:latest`। (`latest` মানে "সবচেয়ে নতুন" নয় — শুধু একটা ডিফল্ট tag নাম! প্রোডাকশনে সবসময় নির্দিষ্ট ভার্সন tag দাও।)

### OCI — সবার জন্য এক নিয়ম

**OCI** (Open Container Initiative) image ফরম্যাট আর runtime-এর স্ট্যান্ডার্ড ঠিক করে। তাই Docker দিয়ে বানানো image Podman, Kubernetes সব জায়গায় চলে।

### Docker-এর ভেতরের আর্কিটেকচার

```
 ┌──────────────┐  REST API over /var/run/docker.sock
 │ docker (CLI) │ ─────────────────────────────────────┐
 └──────────────┘                                      ▼
                                          ┌─────────────────────────┐
                                          │ dockerd (Docker daemon) │
                                          │ image build, network,   │
                                          │ volume, API             │
                                          └────────────┬────────────┘
                                                       │ gRPC
                                          ┌────────────▼────────────┐
                                          │ containerd              │
                                          │ image pull/store,       │
                                          │ container lifecycle     │
                                          └────────────┬────────────┘
                                                       │
                                          ┌────────────▼────────────┐
                                          │ containerd-shim-runc-v2 │ (প্রতি container-এ একটা;
                                          └────────────┬────────────┘  dockerd রিস্টার্ট হলেও
                                                       │               container বেঁচে থাকে)
                                          ┌────────────▼────────────┐
                                          │ runc  (OCI runtime)     │ namespaces + cgroups
                                          │ container তৈরি করে বিদায়  │ সেট করে, তারপর exit
                                          └────────────┬────────────┘
                                                       ▼
                                          ┌─────────────────────────┐
                                          │ তোমার process (PID 1)   │
                                          └─────────────────────────┘
```

| উপাদান | ভূমিকা |
|---|---|
| `docker` | শুধু একটা ক্লায়েন্ট — তুমি যা টাইপ করো API-তে পাঠায় |
| `dockerd` | মূল daemon, root হিসেবে চলে |
| `containerd` | container-এর জীবনচক্র সামলানোর "ইন্ডাস্ট্রি-স্ট্যান্ডার্ড" runtime (Kubernetes-ও সরাসরি এটা ব্যবহার করে) |
| `runc` | নিম্ন-স্তরের runtime — `clone()`, `unshare()`, cgroup ফাইলে লেখা, `pivot_root` — ঠিক যা আমরা উপরে হাতে করলাম |

---

## ৪৮.৬ Docker — হাতে-কলমে

### ৪৮.৬.১ ইনস্টল

> ⚠️ **সতর্কতা:** Ubuntu-র নিজস্ব `docker.io` প্যাকেজ আর Docker-এর অফিশিয়াল `docker-ce` প্যাকেজ একসাথে রেখো না। আগে পুরনো/বিরোধী প্যাকেজ সরাও:
> ```bash
> $ for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove -y $pkg; done
> ```

**Ubuntu 24.04 (অফিশিয়াল Docker রিপোজিটরি থেকে):**

```bash
$ sudo apt update
$ sudo apt install ca-certificates curl
$ sudo install -m 0755 -d /etc/apt/keyrings
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
$ sudo chmod a+r /etc/apt/keyrings/docker.asc
$ echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt update
$ sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Debian 12-এ একই ধাপ, শুধু URL-এ `ubuntu`-র বদলে `debian`।

**Fedora:**

```bash
$ sudo dnf install dnf-plugins-core
$ sudo dnf config-manager addrepo --from-repofile=https://download.docker.com/linux/fedora/docker-ce.repo
$ sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
$ sudo systemctl enable --now docker
```

**RHEL 9 / Rocky / Alma:**

```bash
$ sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
$ sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
$ sudo systemctl enable --now docker
```

(Fedora 41+-এর `dnf5`-এ `config-manager addrepo` সিনট্যাক্স, পুরনো `dnf4`-এ `--add-repo`। RHEL-এ Red Hat নিজে Docker-এর বদলে **Podman** দেয় — ৪৮.৯ দেখো।)

**যাচাই:**

```bash
$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

`hello-world` চালাতে ভেতরে যা ঘটল: CLI → dockerd-কে বলল; dockerd দেখল image লোকালে নেই → Docker Hub থেকে pull করল → container বানাল → container বার্তা ছাপিয়ে exit করল।

**sudo ছাড়া docker:**

```bash
$ sudo usermod -aG docker $USER
$ newgrp docker          # অথবা logout/login
$ docker ps
```

> ⚠️ **সতর্কতা:** `docker` গ্রুপের সদস্য হওয়া মানে কার্যত **root** হওয়া! কারণ `docker run -v /:/host -it ubuntu chroot /host` দিয়ে যে কেউ হোস্টের পুরো ফাইলসিস্টেমে root হয়ে ঢুকতে পারে। শুধু বিশ্বস্ত ইউজারকে এই গ্রুপে দাও, অথবা rootless Docker/Podman ব্যবহার করো।

### ৪৮.৬.২ Image নিয়ে কাজ

```bash
$ docker pull nginx:1.27
$ docker images            # বা: docker image ls
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
nginx         1.27      3b25b682ea82   3 weeks ago    192MB
hello-world   latest    d2c94e258dcb   18 months ago  13.3kB

$ docker image history nginx:1.27      # কোন স্তর কীভাবে তৈরি
$ docker image inspect nginx:1.27      # সব মেটাডেটা (JSON)
$ docker rmi hello-world               # image মোছো
$ docker image prune                   # dangling (নামহীন) image মোছো
$ docker search postgres               # Docker Hub-এ খোঁজো
$ docker tag nginx:1.27 myrepo/nginx:v1  # নতুন নাম দাও (কপি না, শুধু লেবেল)
$ docker login && docker push myrepo/nginx:v1
```

### ৪৮.৬.৩ `docker run` — সবচেয়ে গুরুত্বপূর্ণ কমান্ড

**Syntax:** `docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`

```bash
$ docker run -d -p 8080:80 -v ~/site:/usr/share/nginx/html:ro --name web nginx:1.27
7f3c2a9b1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a
```

এক লাইনে ভেঙে দেখি:

| অংশ | অর্থ |
|---|---|
| `-d` (`--detach`) | ব্যাকগ্রাউন্ডে চালাও, container ID ছাপিয়ে টার্মিনাল ফেরত দাও |
| `-p 8080:80` (`--publish`) | **হোস্টের** port 8080 → **container-এর** port 80 |
| `-v ~/site:/usr/share/nginx/html:ro` (`--volume`) | হোস্টের `~/site` ফোল্ডার container-এর ভেতরে ওই পাথে; `:ro` = read-only |
| `--name web` | container-এর নাম `web` (না দিলে Docker মজার র‍্যান্ডম নাম দেয়, যেমন `quirky_turing`) |
| `nginx:1.27` | কোন image থেকে |

এখন ব্রাউজারে `http://localhost:8080` খোলো — nginx পেজ!

**`docker run`-এর অন্যান্য দরকারি অপশন:**

| অপশন | অর্থ |
|---|---|
| `-it` | `-i` (stdin খোলা) + `-t` (pseudo-terminal) — ইন্টারঅ্যাক্টিভ shell-এর জন্য |
| `--rm` | container থামলেই নিজে মুছে যাবে |
| `-e KEY=VALUE` (`--env`) | environment variable |
| `--env-file .env` | ফাইল থেকে env var |
| `-p 127.0.0.1:8080:80` | শুধু localhost-এ খোলো (বাইরে থেকে না) |
| `-P` | image-এর সব `EXPOSE` করা port র‍্যান্ডম হোস্ট port-এ |
| `--network NAME` | নির্দিষ্ট নেটওয়ার্কে যুক্ত করো |
| `--restart unless-stopped` | crash/রিবুটে আবার চালু হবে (`no`, `on-failure[:N]`, `always`, `unless-stopped`) |
| `--memory 512m` / `-m 512m` | RAM লিমিট (cgroup `memory.max`) |
| `--cpus 1.5` | CPU লিমিট (cgroup `cpu.max`) |
| `--pids-limit 100` | process সংখ্যা লিমিট |
| `-w /app` (`--workdir`) | ভেতরের working directory |
| `-u 1000:1000` (`--user`) | কোন UID:GID হিসেবে চালাবে |
| `--read-only` | rootfs read-only |
| `--hostname NAME` | container-এর hostname (UTS namespace) |
| `--entrypoint CMD` | image-এর ENTRYPOINT বদলাও |
| `--mount type=bind,src=...,dst=...` | `-v`-এর আরও স্পষ্ট, নতুন সিনট্যাক্স |

```bash
# ইন্টারঅ্যাক্টিভ, কাজ শেষে মুছে যাবে
$ docker run -it --rm ubuntu:24.04 bash
root@a1b2c3d4e5f6:/# cat /etc/os-release | head -2
PRETTY_NAME="Ubuntu 24.04.1 LTS"
NAME="Ubuntu"
root@a1b2c3d4e5f6:/# exit

# ডেটাবেস — env var সহ
$ docker run -d --name db -e POSTGRES_PASSWORD=secret -p 5432:5432 postgres:16
```

> 💡 **টিপস:** hostname-এ যে হিজিবিজি `a1b2c3d4e5f6` — ওটা container ID-র প্রথম ১২ অক্ষর। Docker UTS namespace দিয়ে এটা সেট করে।

### ৪৮.৬.৪ চলমান container দেখা ও নিয়ন্ত্রণ

```bash
$ docker ps
CONTAINER ID   IMAGE        COMMAND                  CREATED         STATUS         PORTS                                   NAMES
7f3c2a9b1d4e   nginx:1.27   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   web

$ docker ps -a          # থেমে থাকাগুলোও (Exited)
$ docker ps -q          # শুধু ID (স্ক্রিপ্টে কাজের)
```

| কমান্ড | কাজ |
|---|---|
| `docker logs web` | container-এর stdout/stderr দেখাও |
| `docker logs -f web` | লাইভ ফলো করো (`tail -f`-এর মতো) |
| `docker logs --tail 50 --since 10m web` | শেষ ৫০ লাইন, গত ১০ মিনিটের |
| `docker exec -it web bash` | চলমান container-এর ভেতরে নতুন shell (bash না থাকলে `sh`) |
| `docker exec web nginx -t` | ভেতরে একটা কমান্ড চালাও |
| `docker stop web` | ভদ্রভাবে থামাও: SIGTERM, ১০ সেকেন্ড পর SIGKILL |
| `docker kill web` | সঙ্গে সঙ্গে SIGKILL |
| `docker start web` / `docker restart web` | আবার চালু / রিস্টার্ট |
| `docker pause web` / `unpause` | freeze (cgroup freezer) |
| `docker rm web` | থামা container মোছো |
| `docker rm -f web` | চলমান হলেও জোর করে থামিয়ে মোছো |
| `docker inspect web` | সব বিস্তারিত JSON |
| `docker inspect -f '{{.NetworkSettings.IPAddress}}' web` | নির্দিষ্ট ফিল্ড |
| `docker stats` | লাইভ CPU/RAM/নেটওয়ার্ক ব্যবহার (`top`-এর মতো) |
| `docker top web` | container-এর process তালিকা |
| `docker cp web:/etc/nginx/nginx.conf .` | container ↔ হোস্ট ফাইল কপি |
| `docker diff web` | upperdir-এ কী কী বদলেছে (A=যোগ, C=পরিবর্তন, D=মোছা) |
| `docker port web` | port ম্যাপিং |
| `docker attach web` | container-এর মূল process-এর stdin/stdout-এ যুক্ত হও |
| `docker rename web web2` | নাম বদলাও |
| `docker update --memory 1g web` | চলমান container-এর লিমিট বদলাও |

```bash
$ docker logs --tail 3 web
172.17.0.1 - - [12/Nov/2026:11:20:01 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.5.0" "-"
...

$ docker stats --no-stream
CONTAINER ID   NAME   CPU %     MEM USAGE / LIMIT     MEM %     NET I/O         BLOCK I/O   PIDS
7f3c2a9b1d4e   web    0.00%     8.4MiB / 15.5GiB      0.05%     5.2kB / 2.1kB   0B / 8.19kB  9
```

> 📝 **মনে রাখো — container-এর জীবনচক্র:**
> ```
>  docker create    docker start        docker stop/kill        docker rm
>  ──────────► Created ──────► Running ────────────────► Exited ──────► (মুছে গেল)
>                                │  ▲                       │
>                  docker pause  ▼  │ unpause   docker start│
>                              Paused                       └──► Running
>  docker run = create + start
> ```

> 💡 **টিপস:** container ID পুরোটা লিখতে হয় না — যতটুকু লিখলে অনন্য হয় ততটুকুই যথেষ্ট: `docker stop 7f3`।

### ৪৮.৬.৫ Docker-এ নেটওয়ার্কিং

ডিফল্টে Docker হোস্টে `docker0` নামে একটা Linux bridge বানায় (সাধারণত `172.17.0.0/16`)। প্রতিটা container-এর network namespace থেকে একটা veth জোড়া এই ব্রিজে যুক্ত হয়, আর iptables/nftables-এ NAT (MASQUERADE) rule দিয়ে বাইরে যায়। `-p 8080:80` দিলে Docker একটা DNAT rule যোগ করে: হোস্টের 8080-এ আসা প্যাকেট → container-এর IP:80।

```
   ইন্টারনেট/LAN
        │
 ┌──────┴──────────────────────────────────────────────┐
 │ HOST  eth0 192.168.1.10                             │
 │        │   iptables: DNAT :8080 → 172.17.0.2:80     │
 │        │             MASQUERADE 172.17.0.0/16       │
 │   ┌────┴─────────── docker0 (bridge) 172.17.0.1 ──┐ │
 │   │ vethA                         vethB           │ │
 │   └──┬────────────────────────────────┬───────────┘ │
 │  ┌───┴───────────┐            ┌───────┴───────┐     │
 │  │ web           │            │ db            │     │
 │  │ eth0          │            │ eth0          │     │
 │  │ 172.17.0.2:80 │            │ 172.17.0.3    │     │
 │  └───────────────┘            └───────────────┘     │
 └─────────────────────────────────────────────────────┘
```

| নেটওয়ার্ক ড্রাইভার | অর্থ |
|---|---|
| `bridge` | ডিফল্ট; একই হোস্টে container-রা নিজেদের মধ্যে কথা বলে |
| `host` | container আলাদা network namespace পায় না, হোস্টের নেটওয়ার্কই ব্যবহার করে (port ম্যাপিং লাগে না) |
| `none` | শুধু loopback, কোনো নেটওয়ার্ক নেই |
| `macvlan` / `ipvlan` | container LAN-এ নিজের আলাদা MAC/IP পায় |
| `overlay` | একাধিক হোস্ট জুড়ে (Swarm) |

```bash
$ docker network create mynet
$ docker run -d --name db  --network mynet -e POSTGRES_PASSWORD=secret postgres:16
$ docker run -it --rm --network mynet postgres:16 psql -h db -U postgres
```

> 💡 **টিপস:** **নিজের বানানো** (user-defined) bridge নেটওয়ার্কে Docker একটা বিল্ট-ইন DNS দেয় — তাই container-রা একে অপরকে **নাম দিয়ে** খুঁজে পায় (`-h db`)। ডিফল্ট `bridge` নেটওয়ার্কে এই সুবিধা নেই। তাই সবসময় নিজের নেটওয়ার্ক বানাও।

> ⚠️ **সতর্কতা:** Docker নিজের iptables rule সরাসরি যোগ করে, যা অনেক সময় **ufw-কে পাশ কাটিয়ে যায়**। ufw-তে port বন্ধ থাকলেও `-p 5432:5432` দিলে ডেটাবেস ইন্টারনেটে খোলা হয়ে যেতে পারে! সমাধান: `-p 127.0.0.1:5432:5432` ব্যবহার করো, অথবা `DOCKER-USER` chain-এ rule দাও। ফায়ারওয়াল নিয়ে [অধ্যায় ৩৯](39-firewall.md) দেখো।

### ৪৮.৬.৬ Volumes — স্থায়ী ডেটা

আগেই দেখেছি container মুছলে upperdir মুছে যায়। ডেটা বাঁচানোর তিনটা উপায়:

| ধরন | সিনট্যাক্স | কোথায় থাকে | কখন |
|---|---|---|---|
| **Named volume** | `-v pgdata:/var/lib/postgresql/data` | `/var/lib/docker/volumes/pgdata/_data` — Docker সামলায় | ডেটাবেস, প্রোডাকশন ডেটা |
| **Bind mount** | `-v /home/rahim/site:/usr/share/nginx/html` | হোস্টের যেকোনো পাথ | ডেভেলপমেন্ট (কোড এডিট করে সাথে সাথে দেখা), কনফিগ ফাইল |
| **tmpfs** | `--tmpfs /tmp` | শুধু RAM-এ | অস্থায়ী গোপন ডেটা |

`-v`-এর প্রথম অংশ `/` বা `./` দিয়ে শুরু হলে bind mount, নাহলে named volume।

```bash
$ docker volume create pgdata
$ docker volume ls
DRIVER    VOLUME NAME
local     pgdata
$ docker volume inspect pgdata
$ docker run -d --name db -e POSTGRES_PASSWORD=secret -v pgdata:/var/lib/postgresql/data postgres:16
$ docker rm -f db           # container গেল...
$ docker run -d --name db -e POSTGRES_PASSWORD=secret -v pgdata:/var/lib/postgresql/data postgres:16
                            # ...কিন্তু ডেটা আগের মতোই আছে!
$ docker volume rm pgdata   # volume মোছো (সাবধানে!)
$ docker volume prune       # অব্যবহৃত volume মোছো
```

> ⚠️ **সতর্কতা:** Fedora/RHEL-এ SELinux চালু থাকলে bind mount-এ "Permission denied" আসতে পারে। তখন `:z` (শেয়ার্ড) বা `:Z` (প্রাইভেট) সাফিক্স দাও: `-v ~/site:/usr/share/nginx/html:Z` — এটা ফোল্ডারের SELinux লেবেল ঠিক করে দেয়।

### ৪৮.৬.৭ পরিষ্কার-পরিচ্ছন্নতা

```bash
$ docker system df            # কে কতটা ডিস্ক খাচ্ছে
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          12        3         4.1GB     3.2GB (78%)
Containers      5         2         120MB     80MB (66%)
Local Volumes   4         2         1.2GB     300MB (25%)
Build Cache     40        0         900MB     900MB

$ docker container prune      # সব থামা container মোছো
$ docker system prune         # থামা container + অব্যবহৃত নেটওয়ার্ক + dangling image + build cache
$ docker system prune -a --volumes   # ⚠️ সব অব্যবহৃত image ও volume-ও!
```

> ⚠️ **সতর্কতা:** `docker system prune -a --volumes` এক কমান্ডে তোমার ডেটাবেসের volume-ও মুছে দিতে পারে যদি সেই মুহূর্তে কোনো container সেটা ব্যবহার না করে। চালানোর আগে দুবার ভাবো।

---

## ৪৮.৭ Dockerfile — নিজের image বানাও

**Dockerfile** হলো image বানানোর রেসিপি — একটা সাধারণ টেক্সট ফাইল, যার প্রতিটা লাইন একটা নির্দেশ। `docker build` এটা উপর থেকে নিচে পড়ে, আর ফাইলসিস্টেম বদলায় এমন প্রতিটা নির্দেশ (`RUN`, `COPY`, `ADD`) একটা নতুন **স্তর (layer)** তৈরি করে।

### একটা বাস্তব উদাহরণ — Python Flask অ্যাপ

প্রজেক্টের গঠন:

```
myapp/
├── app.py
├── requirements.txt
├── .dockerignore
└── Dockerfile
```

`app.py`:

```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def home():
    return "আসসালামু আলাইকুম, Docker থেকে!\n"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

`requirements.txt`:

```
flask==3.0.3
```

`Dockerfile`:

```dockerfile
# ১. বেস image — কোথা থেকে শুরু
FROM python:3.12-slim

# ২. মেটাডেটা
LABEL maintainer="rahim@example.com"

# ৩. environment variable
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# ৪. ভেতরের কাজের ফোল্ডার (না থাকলে তৈরি হবে)
WORKDIR /app

# ৫. আগে শুধু dependency তালিকা কপি — ক্যাশের সুবিধার জন্য
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ৬. তারপর বাকি কোড
COPY . .

# ৭. root হিসেবে না চালিয়ে সাধারণ ইউজার হিসেবে
RUN useradd --create-home appuser
USER appuser

# ৮. ডকুমেন্টেশন: অ্যাপ এই port শোনে
EXPOSE 5000

# ৯. container চালু হলে কী চলবে
CMD ["python", "app.py"]
```

`.dockerignore` (build context থেকে এগুলো বাদ — `.gitignore`-এর মতো):

```
.git
__pycache__/
*.pyc
.env
venv/
```

Build ও চালানো:

```bash
$ cd myapp
$ docker build -t myapp:1.0 .
[+] Building 12.3s (11/11) FINISHED
 => [internal] load build definition from Dockerfile
 => [1/5] FROM docker.io/library/python:3.12-slim
 => [2/5] WORKDIR /app
 => [3/5] COPY requirements.txt .
 => [4/5] RUN pip install --no-cache-dir -r requirements.txt
 => [5/5] COPY . .
 ...
 => => naming to docker.io/library/myapp:1.0

$ docker run -d -p 5000:5000 --name myapp myapp:1.0
$ curl localhost:5000
আসসালামু আলাইকুম, Docker থেকে!
```

শেষের `.` হলো **build context** — এই ফোল্ডারের সব ফাইল (`.dockerignore` বাদে) builder-এর কাছে পাঠানো হয়। `COPY` শুধু এর ভেতরের ফাইলই কপি করতে পারে।

### Dockerfile নির্দেশাবলী

| নির্দেশ | কাজ | উদাহরণ |
|---|---|---|
| `FROM` | বেস image (প্রথম নির্দেশ হতে হবে, `ARG` ছাড়া) | `FROM debian:12-slim` |
| `RUN` | build-এর সময় কমান্ড চালাও, ফলাফল নতুন স্তর | `RUN apt-get update && apt-get install -y curl` |
| `COPY` | context থেকে image-এ ফাইল কপি | `COPY src/ /app/src/` |
| `ADD` | `COPY`-এর মতো, কিন্তু URL ও লোকাল tar স্বয়ংক্রিয় extract করে | `ADD app.tar.gz /opt/` |
| `WORKDIR` | পরের নির্দেশগুলোর working directory | `WORKDIR /app` |
| `ENV` | environment variable (build ও run দুই সময়েই) | `ENV NODE_ENV=production` |
| `ARG` | শুধু build-time variable | `ARG VERSION=1.0` → `docker build --build-arg VERSION=2.0` |
| `EXPOSE` | কোন port শোনে তার ডকুমেন্টেশন (নিজে port খোলে **না**) | `EXPOSE 80` |
| `USER` | কোন ইউজার হিসেবে চলবে | `USER 1000` |
| `VOLUME` | এই পাথ volume হবে বলে চিহ্নিত | `VOLUME /data` |
| `CMD` | ডিফল্ট কমান্ড — `docker run IMAGE অন্য-কমান্ড` দিলে বদলে যায় | `CMD ["nginx", "-g", "daemon off;"]` |
| `ENTRYPOINT` | মূল executable — run-এর আর্গুমেন্ট এর পরে যোগ হয় | `ENTRYPOINT ["ping"]` |
| `HEALTHCHECK` | container সুস্থ কিনা যাচাইয়ের কমান্ড | `HEALTHCHECK CMD curl -f http://localhost/ \|\| exit 1` |
| `LABEL` | মেটাডেটা | `LABEL version="1.0"` |
| `SHELL` | `RUN`-এর shell বদলাও | `SHELL ["/bin/bash", "-c"]` |
| `STOPSIGNAL` | `docker stop` কোন signal পাঠাবে | `STOPSIGNAL SIGQUIT` |

### CMD বনাম ENTRYPOINT

```dockerfile
ENTRYPOINT ["ping", "-c", "3"]
CMD ["localhost"]
```

```bash
$ docker run pinger              # চলবে: ping -c 3 localhost
$ docker run pinger google.com   # চলবে: ping -c 3 google.com  (CMD বদলে গেল)
```

ENTRYPOINT = "কী চালাবে", CMD = "ডিফল্ট আর্গুমেন্ট"।

**exec form বনাম shell form:**

| ফর্ম | লেখা | কী ঘটে |
|---|---|---|
| exec form ✅ | `CMD ["python", "app.py"]` | python সরাসরি PID 1, `docker stop`-এর SIGTERM ঠিকমতো পায় |
| shell form | `CMD python app.py` | আসলে চলে `/bin/sh -c "python app.py"` — sh হয় PID 1, signal অনেক সময় অ্যাপে পৌঁছায় না, তাই `docker stop` ১০ সেকেন্ড অপেক্ষা করে SIGKILL দেয় |

> 💡 **টিপস:** CMD/ENTRYPOINT-এ সবসময় exec form (JSON array) ব্যবহার করো।

### Layer cache — কেন ক্রম গুরুত্বপূর্ণ

Docker প্রতিটা স্তর ক্যাশ করে। কোনো নির্দেশ বা তার ইনপুট ফাইল বদলালে **সেই স্তর ও তার পরের সব স্তর** আবার বানাতে হয়।

```
 ❌ খারাপ ক্রম                    ✅ ভালো ক্রম
 COPY . .           ← কোড বদলালেই    COPY requirements.txt .
 RUN pip install    ← আবার সব       RUN pip install    ← ক্যাশ থেকে (দ্রুত!)
                      install!      COPY . .           ← শুধু এটা নতুন
```

যা কম বদলায় সেটা উপরে, যা বেশি বদলায় সেটা নিচে।

### Multi-stage build — ছোট image

Go বা Java-র মতো ভাষায় compiler লাগে build-এর সময়, কিন্তু চালানোর সময় না। Multi-stage build-এ এক stage-এ build করে শুধু ফলাফল পরের stage-এ নাও:

```dockerfile
# Stage 1: build (বড় image, compiler সহ ~৮০০ MB)
FROM golang:1.23 AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/server .

# Stage 2: run (ছোট image)
FROM alpine:3.20
COPY --from=builder /out/server /usr/local/bin/server
USER nobody
ENTRYPOINT ["/usr/local/bin/server"]
```

চূড়ান্ত image-এ শুধু alpine (~৮ MB) + তোমার binary। compiler, সোর্স কোড কিছুই নেই — ছোট, দ্রুত, আর আক্রমণের সুযোগ কম।

### ভালো অভ্যাস (Best practices)

1. ছোট বেস image: `-slim`, `alpine`, বা `distroless`।
2. নির্দিষ্ট ভার্সন tag: `FROM python:3.12-slim`, `latest` নয়।
3. `RUN` একত্র করো ও পরিষ্কার করো:
   ```dockerfile
   RUN apt-get update && \
       apt-get install -y --no-install-recommends curl && \
       rm -rf /var/lib/apt/lists/*
   ```
   (আলাদা `RUN`-এ মুছলে লাভ নেই — আগের স্তরে ফাইল থেকেই যায়।)
4. root হিসেবে চালিও না — `USER` ব্যবহার করো।
5. পাসওয়ার্ড/কী image-এ রেখো না (`docker history`-তে দেখা যায়!) — runtime-এ env/secret দিয়ে দাও।
6. `.dockerignore` রাখো।
7. এক container = এক কাজ (একটা মূল process)।

### Build-সংক্রান্ত কমান্ড

| কমান্ড | কাজ |
|---|---|
| `docker build -t name:tag .` | build ও নাম দাও |
| `docker build -f Dockerfile.prod -t app .` | অন্য নামের Dockerfile |
| `docker build --no-cache -t app .` | ক্যাশ ছাড়া নতুন করে |
| `docker build --target builder -t app-dev .` | নির্দিষ্ট stage পর্যন্ত |
| `docker buildx build --platform linux/amd64,linux/arm64 -t user/app --push .` | একাধিক CPU আর্কিটেকচারের জন্য |
| `docker builder prune` | build cache মোছো |
| `docker commit web myimage:snap` | চলমান container থেকে image (শুধু পরীক্ষার জন্য; Dockerfile-ই সঠিক পথ) |
| `docker save -o app.tar app:1.0` / `docker load -i app.tar` | image ফাইলে রাখো / ফাইল থেকে আনো (ইন্টারনেট ছাড়া সরানো) |

---

## ৪৮.৮ Docker Compose — অনেক container একসাথে

বাস্তব অ্যাপে থাকে ওয়েব সার্ভার + ডেটাবেস + ক্যাশ। প্রতিবার লম্বা `docker run` লেখা ক্লান্তিকর আর ভুলপ্রবণ। **Docker Compose** দিয়ে সবকিছু একটা YAML ফাইলে বর্ণনা করে এক কমান্ডে চালানো যায়।

> 📝 **মনে রাখো:** পুরনো `docker-compose` (হাইফেন সহ, Python-এ লেখা, v1) বাতিল হয়ে গেছে। এখন **`docker compose`** (স্পেস দিয়ে, v2, Docker CLI-এর plugin)। ফাইলের পছন্দের নাম `compose.yaml` (পুরনো `docker-compose.yml`-ও চলে)। ফাইলের শুরুতে `version: "3.8"` লাইন এখন আর লাগে না (obsolete)।

`compose.yaml`:

```yaml
services:
  web:
    build: .                    # এই ফোল্ডারের Dockerfile থেকে build
    ports:
      - "5000:5000"
    environment:
      DATABASE_URL: postgresql://app:secret@db:5432/appdb
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      retries: 5

  cache:
    image: redis:7-alpine

volumes:
  pgdata:
```

Compose স্বয়ংক্রিয়ভাবে প্রজেক্টের জন্য একটা নেটওয়ার্ক বানায় (`myapp_default`), তাই `web` container `db` আর `cache` নামে অন্যদের খুঁজে পায়।

```bash
$ docker compose up -d
[+] Running 5/5
 ✔ Network myapp_default    Created
 ✔ Volume "myapp_pgdata"    Created
 ✔ Container myapp-db-1     Healthy
 ✔ Container myapp-cache-1  Started
 ✔ Container myapp-web-1    Started

$ docker compose ps
NAME            IMAGE           COMMAND                  SERVICE   CREATED          STATUS                    PORTS
myapp-cache-1   redis:7-alpine  "docker-entrypoint.s…"   cache     30 seconds ago   Up 29 seconds             6379/tcp
myapp-db-1      postgres:16     "docker-entrypoint.s…"   db        30 seconds ago   Up 29 seconds (healthy)   5432/tcp
myapp-web-1     myapp-web       "python app.py"          web       30 seconds ago   Up 23 seconds             0.0.0.0:5000->5000/tcp
```

| কমান্ড | কাজ |
|---|---|
| `docker compose up -d` | সব সার্ভিস ব্যাকগ্রাউন্ডে চালাও (দরকার হলে build/pull) |
| `docker compose up -d --build` | আগে image আবার build করো |
| `docker compose down` | container ও নেটওয়ার্ক মোছো (volume থাকে) |
| `docker compose down -v` | ⚠️ volume-ও মোছো |
| `docker compose ps` | অবস্থা |
| `docker compose logs -f web` | একটা সার্ভিসের লগ ফলো |
| `docker compose exec db psql -U app appdb` | চলমান সার্ভিসে কমান্ড |
| `docker compose run --rm web python manage.py migrate` | এককালীন কাজের জন্য নতুন container |
| `docker compose stop` / `start` / `restart` | থামাও / চালাও / রিস্টার্ট |
| `docker compose pull` | নতুন image নামাও |
| `docker compose build` | শুধু build |
| `docker compose config` | চূড়ান্ত (variable বসানো) কনফিগ দেখাও — ভুল ধরতে দারুণ |
| `docker compose up -d --scale web=3` | একটা সার্ভিসের ৩টা কপি (তখন হোস্ট port ফিক্সড রাখা যায় না) |

> 💡 **টিপস:** একই ফোল্ডারে `.env` ফাইল রাখলে compose সেখান থেকে variable পড়ে, আর YAML-এ `${POSTGRES_PASSWORD}` লিখে ব্যবহার করা যায়। পাসওয়ার্ড তখন YAML-এ লিখতে হয় না (`.env`-কে git-এ commit কোরো না)।

---

## ৪৮.৯ Podman — daemon ছাড়া container

**Podman** (Pod Manager) Red Hat-এর তৈরি, Docker-এর প্রায় হুবহু বিকল্প। RHEL 8/9 ও Fedora-তে এটাই ডিফল্ট container টুল।

```
        DOCKER                                 PODMAN
 ┌──────────┐                           ┌──────────┐
 │ docker   │                           │ podman   │ ← নিজেই fork করে
 └────┬─────┘                           └────┬─────┘   container চালায়
      │ socket                               │
 ┌────▼──────────────┐                  ┌────▼─────┐
 │ dockerd (root,    │ ← সব container   │ conmon   │ ← প্রতি container-এ
 │ সবসময় চলমান)      │   এর বাবা         └────┬─────┘   ছোট মনিটর
 └────┬──────────────┘                  ┌────▼─────┐
 ┌────▼─────┐                           │ crun /   │
 │containerd│                           │ runc     │
 └────┬─────┘                           └────┬─────┘
 ┌────▼─────┐                           ┌────▼─────┐
 │  runc    │                           │container │
 └────┬─────┘                           └──────────┘
 ┌────▼─────┐
 │container │
 └──────────┘
```

| বৈশিষ্ট্য | Docker | Podman |
|---|---|---|
| Daemon | `dockerd` সবসময় root হিসেবে চলে | কোনো daemon নেই |
| ডিফল্ট মোড | rootful | **rootless** (সাধারণ ইউজার হিসেবেই চলে) |
| Single point of failure | daemon মরলে ঝামেলা | নেই |
| Pod সাপোর্ট | নেই | আছে (Kubernetes-এর মতো) |
| systemd ইন্টিগ্রেশন | সীমিত | চমৎকার (Quadlet) |
| CLI | `docker ...` | প্রায় হুবহু একই: `podman ...` |
| ডিফল্ট OCI runtime | `runc` | Fedora/RHEL-এ `crun` (C-তে লেখা, দ্রুত) |

```bash
# ইনস্টল
$ sudo apt install podman        # Ubuntu/Debian
$ sudo dnf install podman        # Fedora/RHEL

# ব্যবহার — docker-এর মতোই, কিন্তু sudo ছাড়া!
$ podman run -d -p 8080:80 --name web docker.io/library/nginx:1.27
$ podman ps
CONTAINER ID  IMAGE                           COMMAND               CREATED        STATUS        PORTS                 NAMES
a3f1c2d4e5b6  docker.io/library/nginx:1.27    nginx -g daemon o...  5 seconds ago  Up 5 seconds  0.0.0.0:8080->80/tcp  web
$ podman logs web
$ podman exec -it web sh
$ podman stop web && podman rm web
```

> 💡 **টিপস:** পুরো image নাম (`docker.io/library/nginx`) লেখা ভালো অভ্যাস। ছোট নাম (`nginx`) দিলে Podman `/etc/containers/registries.conf`-এর তালিকা থেকে registry বাছাইয়ের জন্য প্রশ্ন করতে পারে।

> 💡 **টিপস:** অভ্যাস না বদলাতে চাইলে `alias docker=podman` — বেশিরভাগ কমান্ড একইভাবে চলবে। Compose ফাইলের জন্য আছে `podman compose` (ভেতরে `podman-compose` বা `docker-compose` টুল ব্যবহার করে)।

Podman-এর বাড়তি টুলভাই: **Buildah** (image build), **Skopeo** (registry-তে image কপি/inspect, image নামানো ছাড়াই)।

### Pods

একটা **pod** হলো এমন কয়েকটা container-এর দল যারা একই network namespace শেয়ার করে (একে অপরকে `localhost` দিয়ে পায়) — ঠিক Kubernetes-এর pod-এর মতো।

```bash
$ podman pod create --name mypod -p 8080:80
$ podman run -d --pod mypod --name web docker.io/library/nginx:1.27
$ podman run -d --pod mypod --name sidecar docker.io/library/alpine sleep infinity
$ podman pod ps
$ podman generate kube mypod > mypod.yaml     # Kubernetes YAML বানাও!
$ podman kube play mypod.yaml                 # YAML থেকে চালাও
```

### Quadlet — container-কে systemd সার্ভিস বানাও

পুরনো `podman generate systemd` এখন deprecated; নতুন পথ **Quadlet** (Podman 4.4+)। একটা `.container` ফাইল লেখো, systemd নিজেই সার্ভিস বানিয়ে নেবে:

`~/.config/containers/systemd/web.container` (rootless; সিস্টেম-ব্যাপী হলে `/etc/containers/systemd/`):

```ini
[Unit]
Description=My Nginx container

[Container]
Image=docker.io/library/nginx:1.27
PublishPort=8080:80
Volume=%h/site:/usr/share/nginx/html:ro,Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

```bash
$ systemctl --user daemon-reload
$ systemctl --user start web.service
$ systemctl --user status web.service
$ loginctl enable-linger $USER     # লগআউটের পরও ইউজার সার্ভিস চলুক
```

---

## ৪৮.১০ Rootless container — root ছাড়া container

**Rootless container** মানে container runtime আর container — দুটোই সাধারণ ইউজার হিসেবে চলে, কোথাও আসল root লাগে না। কেউ container ভেঙে বেরিয়ে এলেও সে শুধু একজন সাধারণ ইউজার — পুরো সিস্টেম নয়।

### কীভাবে কাজ করে — user namespace + subordinate ID

container-এর ভেতরে অনেক ইউজার লাগে (root=0, `www-data`=33, `postgres`=999…)। কিন্তু তুমি তো একজন মাত্র ইউজার (UID 1000)! সমাধান: সিস্টেম তোমাকে আরও ৬৫,৫৩৬টা "অধীনস্থ" (subordinate) UID বরাদ্দ দেয়:

```bash
$ cat /etc/subuid
rahim:100000:65536
$ cat /etc/subgid
rahim:100000:65536
```

মানে rahim ১০০০০০ থেকে ১৬৫৫৩৫ পর্যন্ত UID ব্যবহার করতে পারবে। (নতুন ইউজার `useradd` দিয়ে বানালে সাধারণত এটা স্বয়ংক্রিয় হয়; না থাকলে `sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 rahim`।)

```
   container-এর ভেতরে (user namespace)       হোস্টে আসল UID
 ┌──────────────────────────┐            ┌──────────────────┐
 │ UID 0     (root)         │ ─────────► │ 1000  (rahim)    │
 │ UID 1                    │ ─────────► │ 100000           │
 │ UID 33    (www-data)     │ ─────────► │ 100032           │
 │ UID 999   (postgres)     │ ─────────► │ 100998           │
 │ ...                      │            │ ...              │
 │ UID 65536                │ ─────────► │ 165535           │
 └──────────────────────────┘            └──────────────────┘
```

```bash
$ podman unshare cat /proc/self/uid_map
         0       1000          1
         1     100000      65536
```

(প্রথম সংখ্যা = ভেতরের UID, দ্বিতীয় = বাইরের UID, তৃতীয় = কতগুলো।) `newuidmap`/`newgidmap` নামের ছোট setuid প্রোগ্রাম (`uidmap` প্যাকেজ) এই ম্যাপিং লেখে।

```bash
$ podman run -d --name sleeper docker.io/library/alpine sleep 1000
$ podman exec sleeper id
uid=0(root) gid=0(root) groups=0(root),...
$ ps -o user,pid,cmd -C sleep
USER         PID CMD
rahim       8812 sleep 1000      # হোস্ট থেকে দেখলে — সাধারণ ইউজার!
```

### Rootless-এর সীমাবদ্ধতা

| সমস্যা | কারণ | সমাধান |
|---|---|---|
| ১০২৪-এর নিচের port (৮০, ৪৪৩) bind করা যায় না | ওগুলো privileged port | `-p 8080:80` ব্যবহার করো, বা `sudo sysctl net.ipv4.ip_unprivileged_port_start=80` |
| নেটওয়ার্ক একটু ধীর | আসল bridge/veth বানাতে root লাগে, তাই user-space নেটওয়ার্ক স্ট্যাক ব্যবহার হয় | Podman 5+-এ ডিফল্ট `pasta` (আগে `slirp4netns`) যথেষ্ট দ্রুত |
| bind mount-এ ফাইলের মালিক অদ্ভুত (যেমন 100032) | UID ম্যাপিং | `podman unshare chown`, বা `--userns=keep-id` |
| cgroup লিমিট কাজ না করা | cgroups v2 + systemd delegation লাগে | আধুনিক distro-তে সাধারণত ঠিক থাকে |
| image ইউজারভেদে আলাদা | প্রতিটা ইউজারের নিজস্ব স্টোরেজ | rootless: `~/.local/share/containers/storage`, rootful: `/var/lib/containers/storage` |

**Docker rootless মোড:** Docker-ও rootless চালানো যায়:

```bash
$ sudo apt install uidmap docker-ce-rootless-extras
$ dockerd-rootless-setuptool.sh install
$ systemctl --user enable --now docker
$ export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
$ docker run -d -p 8080:80 nginx
```

---

## ৪৮.১১ Kubernetes এক নজরে

একটা সার্ভারে ১০টা container সামলানো সহজ। কিন্তু ৫০টা সার্ভারে ৫,০০০ container? কোনটা কোন সার্ভারে চলবে, কোনটা crash করলে কে আবার চালাবে, ট্রাফিক কীভাবে ভাগ হবে, নতুন ভার্সন কীভাবে downtime ছাড়া deploy হবে? এই কাজের নাম **container orchestration**, আর তার রাজা **Kubernetes** (সংক্ষেপে **K8s** — K আর s-এর মাঝে ৮টা অক্ষর)। Google-এর ভেতরের "Borg" সিস্টেমের অভিজ্ঞতা থেকে তৈরি, এখন CNCF-এর অধীনে।

উপমা: Docker হলো একজন জাহাজের নাবিক যে কন্টেইনার লোড করে। Kubernetes হলো পুরো **বন্দরের কন্ট্রোল টাওয়ার** — কোন জাহাজে কোন কন্টেইনার যাবে, কোন জাহাজ ডুবলে মাল কোথায় সরবে, সব সে ঠিক করে। (গ্রিক ভাষায় Kubernetes মানেই "জাহাজের কাণ্ডারি"।)

### মূল ধারণা: Declarative

তুমি Kubernetes-কে বলো **"আমি কী চাই"** (desired state) — যেমন "nginx-এর ৩টা কপি সবসময় চলুক"। Kubernetes ক্রমাগত দেখে আসল অবস্থা (actual state) কী, আর পার্থক্য থাকলে ঠিক করে (reconciliation loop)। একটা crash করলে নিজেই নতুন একটা চালায় — একে বলে **self-healing**।

### আর্কিটেকচার

```
 ┌──────────────────────── CONTROL PLANE ─────────────────────────┐
 │                                                                │
 │  ┌───────────────┐   ┌──────────┐   ┌──────────────────────┐   │
 │  │ kube-apiserver│◄─►│   etcd   │   │ kube-scheduler       │   │
 │  │ (সবার প্রবেশদ্বার) │   │(key-value│   │ (pod কোন node-এ যাবে)  │   │
 │  └───────▲───────┘   │ ডেটাবেস)  │   └──────────────────────┘   │
 │          │           └──────────┘   ┌──────────────────────┐   │
 │          │                          │ kube-controller-     │   │
 │          │                          │ manager (desired =   │   │
 │          │                          │ actual রাখার লুপ)      │   │
 │          │                          └──────────────────────┘   │
 └──────────┼─────────────────────────────────────────────────────┘
   kubectl ─┤
            │
   ┌────────┴─────────────────┐        ┌──────────────────────────┐
   │ WORKER NODE 1            │        │ WORKER NODE 2            │
   │  kubelet  (node-এর এজেন্ট) │        │  kubelet                 │
   │  kube-proxy (Service     │        │  kube-proxy              │
   │   নেটওয়ার্কিং)           │        │  containerd / CRI-O      │
   │  containerd / CRI-O      │        │  ┌─────┐ ┌─────┐         │
   │  ┌─────┐ ┌─────┐         │        │  │ Pod │ │ Pod │         │
   │  │ Pod │ │ Pod │         │        │  └─────┘ └─────┘         │
   │  └─────┘ └─────┘         │        │                          │
   └──────────────────────────┘        └──────────────────────────┘
```

| উপাদান | কাজ |
|---|---|
| **kube-apiserver** | সব যোগাযোগের কেন্দ্র; `kubectl` এর সাথেই কথা বলে |
| **etcd** | পুরো ক্লাস্টারের অবস্থা রাখা distributed key-value ডেটাবেস |
| **kube-scheduler** | নতুন pod-কে কোন node-এ বসাবে তা ঠিক করে (রিসোর্স দেখে) |
| **kube-controller-manager** | Deployment, ReplicaSet, Node ইত্যাদির controller লুপ চালায় |
| **kubelet** | প্রতিটা node-এ এজেন্ট — container runtime-কে বলে pod চালাতে |
| **kube-proxy** | Service-এর জন্য নেটওয়ার্ক rule (iptables/IPVS/nftables) বসায় |
| **Container runtime** | containerd বা CRI-O (CRI ইন্টারফেস দিয়ে) — Kubernetes 1.24 থেকে dockershim সরানো হয়েছে, কিন্তু Docker-এ বানানো image ঠিকই চলে |

### মূল অবজেক্টগুলো

| অবজেক্ট | কী |
|---|---|
| **Pod** | সবচেয়ে ছোট একক — এক বা একাধিক container, একই IP ও নেটওয়ার্ক শেয়ার করে |
| **ReplicaSet** | নির্দিষ্ট সংখ্যক একই pod চালু রাখে |
| **Deployment** | ReplicaSet-কে সামলায়; rolling update ও rollback |
| **Service** | pod-গুলোর সামনে একটা স্থায়ী IP/DNS নাম ও load balancer (pod-এর IP বদলায়, Service-এর না) |
| **Ingress** | বাইরের HTTP/HTTPS ট্রাফিককে ডোমেইন/পাথ অনুযায়ী Service-এ পাঠায় |
| **ConfigMap / Secret** | কনফিগ ও গোপন তথ্য |
| **PersistentVolume(Claim)** | স্থায়ী স্টোরেজ |
| **Namespace** | ক্লাস্টারের ভেতরে লজিক্যাল ভাগ (dev, prod) — লিনাক্স kernel namespace-এর সাথে গুলিয়ে ফেলো না! |
| **StatefulSet / DaemonSet / Job / CronJob** | ডেটাবেস / প্রতিটা node-এ একটা করে / এককালীন কাজ / শিডিউলড কাজ |

### একটা Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            limits:
              memory: "128Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

(`cpu: "500m"` = ৫০০ millicore = আধা CPU — ভেতরে সেই cgroup `cpu.max`!)

### শেখার জন্য লোকাল ক্লাস্টার

| টুল | বৈশিষ্ট্য |
|---|---|
| **minikube** | এক-node ক্লাস্টার, VM বা container-এ; শুরু করার জন্য সবচেয়ে সহজ |
| **kind** (Kubernetes IN Docker) | Docker container-কে node বানায়; CI-তে জনপ্রিয় |
| **k3s** | হালকা, প্রোডাকশন-যোগ্য; Raspberry Pi/edge-এ চলে |
| **MicroK8s** | Canonical-এর, `snap` দিয়ে Ubuntu-তে |

### `kubectl` — প্রাথমিক কমান্ড

```bash
$ minikube start
$ kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.31.0

$ kubectl apply -f web.yaml
deployment.apps/web created
service/web created

$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
web-7d9c6b5f8d-4xkqz   1/1     Running   0          20s
web-7d9c6b5f8d-9hjtl   1/1     Running   0          20s
web-7d9c6b5f8d-mzp2r   1/1     Running   0          20s

$ kubectl delete pod web-7d9c6b5f8d-4xkqz      # একটা মেরে দেখো...
$ kubectl get pods                              # ...নতুন একটা নিজে চলে এসেছে!
```

| কমান্ড | কাজ |
|---|---|
| `kubectl get pods/deploy/svc/nodes` | তালিকা (`-o wide` বিস্তারিত, `-A` সব namespace) |
| `kubectl describe pod NAME` | বিস্তারিত + events (ডিবাগিংয়ের প্রথম ধাপ) |
| `kubectl logs -f POD` | লগ |
| `kubectl exec -it POD -- sh` | pod-এর ভেতরে shell |
| `kubectl apply -f file.yaml` | YAML প্রয়োগ (তৈরি বা আপডেট) |
| `kubectl delete -f file.yaml` | মুছে ফেলো |
| `kubectl create deployment web --image=nginx:1.27 --replicas=3` | YAML ছাড়া দ্রুত Deployment |
| `kubectl expose deployment web --port=80 --type=NodePort` | Service বানাও |
| `kubectl scale deployment web --replicas=5` | কপি বাড়াও/কমাও |
| `kubectl set image deployment/web nginx=nginx:1.28` | নতুন image — rolling update |
| `kubectl rollout status deployment/web` / `rollout undo deployment/web` | আপডেটের অবস্থা / আগের ভার্সনে ফেরো |
| `kubectl port-forward svc/web 8080:80` | লোকাল port থেকে Service-এ সুড়ঙ্গ |

> 💡 **টিপস:** Kubernetes নিজেই একটা পুরো বই-সমান বিষয়। এই অধ্যায় শেষে লিনাক্স, নেটওয়ার্কিং আর Docker-এ পাকা হলে তবেই Kubernetes-এ ঝাঁপ দাও — তখন বুঝবে এর প্রতিটা অংশ আসলে এই বইয়ের কোনো না কোনো অধ্যায়ের ধারণার উপর দাঁড়িয়ে।

---

## ৪৮.১২ আরও কিছু প্রযুক্তি — নাম জেনে রাখো

| প্রযুক্তি | কী |
|---|---|
| **LXC / Incus (LXD)** | "system container" — পুরো একটা OS (systemd সহ) container-এ চালায়, অনেকটা হালকা VM-এর মতো। Incus হলো LXD-এর কমিউনিটি fork। |
| **systemd-nspawn** | systemd-এর নিজস্ব হালকা container টুল; `machinectl` দিয়ে সামলানো |
| **Firecracker** | AWS-এর "microVM" — VM-এর নিরাপত্তা, container-এর গতি (AWS Lambda এটা ব্যবহার করে) |
| **gVisor / Kata Containers** | container-কে বাড়তি sandbox/হালকা VM-এ মুড়ে শক্ত isolation |
| **Vagrant** | কোড দিয়ে VM বানানো ও সামলানোর টুল |
| **Proxmox VE** | KVM + LXC-ভিত্তিক ওপেন-সোর্স virtualization প্ল্যাটফর্ম (Debian-ভিত্তিক) |
| **Flatpak / Snap** | ডেস্কটপ অ্যাপ প্যাকেজিং — ভেতরে namespace আর sandbox-এর একই কৌশল ([অধ্যায় ৩০](30-package-management.md)) |

---

## ৪৮.১৩ কীবোর্ড শর্টকাট

| শর্টকাট | কোথায় | কাজ |
|---|---|---|
| `Ctrl+P` তারপর `Ctrl+Q` | `docker run -it` / `docker attach` / `podman attach` | container না থামিয়ে **detach** করো (বেরিয়ে আসো) |
| `Ctrl+C` | `docker run` (foreground, `-it` সহ) | মূল process-এ SIGINT — সাধারণত container থেমে যায় |
| `Ctrl+D` / `exit` | container-এর ভেতরের shell | shell থেকে বের হও (`docker exec`-এর shell হলে container চলতেই থাকে; `docker run -it`-এর মূল shell হলে container থেমে যায়) |
| `Ctrl+C` | `docker logs -f`, `docker stats`, `kubectl logs -f` | ফলো করা বন্ধ (container থামে না) |
| `Ctrl+]` | `virsh console` | VM-এর সিরিয়াল কনসোল থেকে বের হও |
| `Ctrl+A` তারপর `X` | `qemu-system-* -nographic` | QEMU বন্ধ করে বের হও |
| `Ctrl+A` তারপর `C` | `qemu -nographic` | QEMU monitor ↔ সিরিয়াল কনসোল টগল |
| `Ctrl+A` তারপর `H` | `qemu -nographic` | এই escape শর্টকাটগুলোর সাহায্য |
| `Ctrl+Alt+G` | QEMU GTK উইন্ডো | মাউস/কীবোর্ড grab ছাড়ো |
| `Ctrl+Alt+F` | QEMU GTK উইন্ডো | ফুলস্ক্রিন টগল |
| `Ctrl+Alt+2` / `Ctrl+Alt+1` | QEMU GTK উইন্ডো | QEMU monitor / গেস্ট ডিসপ্লে |
| বাম `Ctrl+Alt` | virt-manager / virt-viewer কনসোল | মাউস/কীবোর্ড grab ছাড়ো |
| ডান `Ctrl` (Host key) | VirtualBox | মাউস/কীবোর্ড grab ছাড়ো |
| `Ctrl+L` | কোনো shell-এ | স্ক্রিন পরিষ্কার ([অধ্যায় ৬](06-terminal-shortcuts.md)) |

> 💡 **টিপস:** detach key বদলাতে চাইলে: `docker run -it --detach-keys="ctrl-x" ...`, অথবা `~/.docker/config.json`-এ `"detachKeys": "ctrl-x"`। (`Ctrl+P` bash-এর হিস্ট্রি শর্টকাটের সাথে সংঘর্ষ করে বলে অনেকে বদলায়।)

---

## হাতে-কলমে অনুশীলন

### ল্যাব ১: Docker ছাড়া "container" বানাও (namespace + cgroup)

```bash
# ১. একটা Alpine root ফাইলসিস্টেম নামাও (মাত্র ~৩ MB)
$ mkdir -p ~/mycontainer && cd ~/mycontainer
$ curl -LO https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
$ mkdir rootfs && sudo tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs

# ২. cgroup বানাও: ৫০ MB RAM, ২০টা process
$ sudo mkdir /sys/fs/cgroup/mybox
$ echo 50M | sudo tee /sys/fs/cgroup/mybox/memory.max
$ echo 20  | sudo tee /sys/fs/cgroup/mybox/pids.max

# ৩. shell-কে cgroup-এ ঢুকিয়ে, নতুন namespace-এ, alpine root-এ চালাও
$ sudo bash -c 'echo $$ > /sys/fs/cgroup/mybox/cgroup.procs &&
    exec unshare --uts --pid --net --mount --ipc --fork \
    chroot rootfs /bin/sh -c "mount -t proc proc /proc && hostname mybox && exec /bin/sh"'

# ৪. ভেতরে পরীক্ষা করো
/ # hostname
mybox
/ # ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    2 root      0:00 ps
/ # cat /etc/os-release | head -1
NAME="Alpine Linux"
/ # ip addr        # শুধু lo
/ # exit

# ৫. পরিষ্কার
$ sudo rmdir /sys/fs/cgroup/mybox
```

(Alpine-এর ভার্সন নম্বর বদলে থাকলে https://dl-cdn.alpinelinux.org/alpine/ থেকে সর্বশেষ minirootfs ফাইলের নাম দেখে নাও।) লক্ষ করো: তুমি হোস্টে চালিয়েও ভেতরে পেলে আলাদা OS (Alpine), আলাদা hostname, PID 1, নেটওয়ার্ক নেই, আর RAM/process সীমিত — এটাই container!

### ল্যাব ২: Docker দিয়ে একটা স্ট্যাটিক ওয়েবসাইট

1. `mkdir ~/site && echo '<h1>আমার প্রথম Docker সাইট</h1>' > ~/site/index.html`
2. `docker run -d -p 8080:80 -v ~/site:/usr/share/nginx/html:ro --name web nginx:1.27`
3. `curl localhost:8080` — তোমার লেখা দেখাবে।
4. `index.html` এডিট করো, আবার `curl` করো — সাথে সাথে বদল (bind mount!)।
5. `docker logs web` দিয়ে access log দেখো।
6. `docker exec -it web sh` → ভেতরে `cat /etc/os-release`, `ps` (না থাকলে `ls /proc`) দেখো, `exit`।
7. হোস্ট থেকে `ps aux | grep nginx` — container-এর nginx হোস্টেও একটা process হিসেবে দেখা যাচ্ছে!
8. `docker inspect -f '{{.State.Pid}}' web` দিয়ে PID বের করে `sudo ls -l /proc/<PID>/ns` — নিজের shell-এর সাথে namespace নম্বর তুলনা করো।
9. `cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' web).scope/memory.current` — container-এর RAM ব্যবহার সরাসরি cgroup থেকে! (systemd cgroup driver-এ এই পাথ।)
10. `docker stop web && docker rm web`।

### ল্যাব ৩: নিজের image বানাও ও compose দিয়ে চালাও

1. ৪৮.৭-এর Flask অ্যাপ (`app.py`, `requirements.txt`, `Dockerfile`, `.dockerignore`) বানাও।
2. `docker build -t myapp:1.0 .` — সময়টা খেয়াল করো।
3. `app.py`-এর বার্তা বদলে আবার build করো — দেখো `pip install` ধাপ `CACHED` দেখাচ্ছে।
4. `docker image history myapp:1.0` দিয়ে স্তরগুলো দেখো।
5. ৪৮.৮-এর `compose.yaml` বানিয়ে `docker compose up -d`।
6. `docker compose ps`, `docker compose logs -f web`।
7. `docker compose exec db psql -U app appdb -c '\l'`।
8. `docker compose down`, তারপর আবার `up -d` — ডেটাবেস ডেটা volume-এ টিকে আছে কিনা দেখো। শেষে `docker compose down -v`।

### ল্যাব ৪: Podman rootless

1. `podman run -d --name sleeper docker.io/library/alpine sleep 1000`
2. `podman exec sleeper id` — ভেতরে root।
3. `ps -o user,pid,cmd -C sleep` — বাইরে তুমি।
4. `podman unshare cat /proc/self/uid_map` — ম্যাপিং দেখো।
5. `podman run -d -p 80:80 docker.io/library/nginx` চেষ্টা করো — কী error আসে? কেন? এবার `-p 8080:80` দিয়ে চালাও।
6. `podman rm -f -a` দিয়ে সব পরিষ্কার।

### ল্যাব ৫: KVM VM (ঐচ্ছিক, হার্ডওয়্যার সাপোর্ট লাগবে)

1. `grep -Ec '(vmx|svm)' /proc/cpuinfo` ও `kvm-ok` দিয়ে যাচাই।
2. libvirt ইনস্টল করে নিজেকে `libvirt` গ্রুপে যোগ করো।
3. একটা ছোট ISO (যেমন Alpine "virtual") দিয়ে `virt-install` চালাও।
4. `virsh list --all`, `virsh dominfo`, `virsh domifaddr` চালাও।
5. `virsh snapshot-create-as VM clean` → VM-এ কিছু ফাইল মুছে → `virsh snapshot-revert VM clean`।
6. হোস্টে `ps aux | grep qemu` — VM আসলে একটা process!

---

## সারসংক্ষেপ

| বিষয় | মূল কমান্ড / ধারণা |
|---|---|
| VM বনাম Container | VM = আলাদা kernel (hypervisor); Container = শেয়ার্ড kernel (namespace + cgroup) |
| Hypervisor | Type 1 (ESXi, Xen, Hyper-V, KVM) / Type 2 (VirtualBox, VMware Workstation) |
| KVM সাপোর্ট | `grep -Ec '(vmx\|svm)' /proc/cpuinfo`, `kvm-ok`, `lsmod \| grep kvm` |
| QEMU | `qemu-img create -f qcow2 d.qcow2 20G`, `qemu-system-x86_64 -enable-kvm ...` |
| libvirt | `virt-install`, `virsh list --all/start/shutdown/destroy/console/snapshot-create-as/undefine` |
| chroot | `sudo chroot DIR /bin/bash` |
| Namespaces | `unshare --uts --pid --net --mount --ipc --user --fork --mount-proc`, `lsns`, `nsenter`, `ip netns` |
| cgroups v2 | `/sys/fs/cgroup/`, `memory.max`, `cpu.max`, `pids.max`, `cgroup.procs`, `systemd-cgls`, `systemd-cgtop`, `systemd-run -p` |
| OverlayFS | `mount -t overlay overlay -o lowerdir=,upperdir=,workdir= MERGED` |
| Docker image | `pull`, `images`, `rmi`, `build -t`, `tag`, `push`, `image history`, `save/load` |
| Docker container | `run -d -p -v --name -e --rm -it`, `ps -a`, `logs -f`, `exec -it`, `stop`, `start`, `rm -f`, `inspect`, `stats`, `cp` |
| Docker volume/network | `volume create/ls/rm`, `network create`, `--network` |
| পরিষ্কার | `docker system df`, `docker system prune` |
| Dockerfile | `FROM RUN COPY WORKDIR ENV ARG EXPOSE USER CMD ENTRYPOINT`; multi-stage `COPY --from=` |
| Compose | `docker compose up -d / ps / logs -f / exec / down / down -v / config` |
| Podman | `podman run/ps/exec/pod create/generate kube/kube play`, Quadlet `.container` |
| Rootless | `/etc/subuid`, `/etc/subgid`, `podman unshare`, `dockerd-rootless-setuptool.sh install` |
| Kubernetes | `kubectl get/describe/logs/exec/apply/delete/scale/rollout`, minikube/kind/k3s |
| শর্টকাট | `Ctrl+P Ctrl+Q` detach, `Ctrl+]` virsh console থেকে বের, `Ctrl+A X` QEMU থেকে বের, `Ctrl+Alt+G` মাউস ছাড়ো |

---

## নিজেকে যাচাই করো

1. VM আর container-এর সবচেয়ে মৌলিক পার্থক্য কী? লিনাক্স হোস্টে কেন Windows container চালানো যায় না, অথচ Windows VM চালানো যায়?
2. Type 1 আর Type 2 hypervisor-এর পার্থক্য কী? KVM, QEMU আর libvirt — প্রত্যেকে কী দায়িত্ব পালন করে?
3. `virsh destroy` আর `virsh undefine --remove-all-storage`-এর মধ্যে পার্থক্য কী?
4. `sudo unshare --pid bash` চালালে `ps` কেন এখনও হোস্টের সব process দেখায়? ঠিক করতে কোন দুটো অপশন লাগবে, আর কেন?
5. cgroup-এ `cpu.max` ফাইলে `25000 100000` লিখলে কী হবে?
6. OverlayFS-এ নিচের স্তরের একটা ফাইল container মুছে ফেললে আসলে কী ঘটে?
7. `docker run -d -p 8080:80 -v data:/data --name app myimg` — প্রতিটা অংশ ব্যাখ্যা করো। `data` কি bind mount না named volume?
8. Dockerfile-এ `COPY requirements.txt .` আর `RUN pip install` কেন `COPY . .`-এর আগে রাখা হয়?
9. `CMD python app.py` না লিখে `CMD ["python", "app.py"]` লেখা কেন ভালো?
10. Rootless container-এ ভেতরের root (UID 0) হোস্টে কে? এটা কোন kernel ফিচার আর কোন ফাইলের উপর নির্ভর করে? আর কেন `docker` গ্রুপে ইউজার যোগ করা নিরাপত্তা-ঝুঁকি?

<details>
<summary>উত্তর দেখো</summary>

1. VM-এর নিজস্ব পূর্ণ OS ও kernel থাকে, hypervisor হার্ডওয়্যার ভাগ করে দেয়; container হোস্টের kernel শেয়ার করে শুধু namespace/cgroup দিয়ে আলাদা করা process। Windows container-এর জন্য Windows kernel লাগে, যা লিনাক্স হোস্টে নেই; কিন্তু VM নিজের kernel নিজে বুট করে, তাই VM-এ Windows চলে।

2. Type 1 সরাসরি হার্ডওয়্যারে চলে (ESXi, Xen, Hyper-V, KVM); Type 2 একটা সাধারণ OS-এর উপর অ্যাপ হিসেবে চলে (VirtualBox, VMware Workstation)। KVM = kernel মডিউল, CPU/মেমরি virtualization (VT-x/AMD-V দিয়ে); QEMU = user-space-এ ডিস্ক, নেটওয়ার্ক কার্ড ইত্যাদি ডিভাইস emulate করে; libvirt = সব VM সামলানোর API/ডেমন, যার ক্লায়েন্ট `virsh`/`virt-manager`।

3. `virsh destroy` শুধু VM-কে জোর করে বন্ধ করে (পাওয়ার কাটার মতো) — VM ও ডিস্ক থেকে যায়, আবার `virsh start` করা যায়। `virsh undefine --remove-all-storage` VM-এর সংজ্ঞা ও ডিস্ক ফাইল স্থায়ীভাবে মুছে দেয়।

4. (ক) `unshare` নিজে নতুন PID namespace-এ ঢোকে না, শুধু তার child-রা — তাই `--fork` লাগে। (খ) `ps` তথ্য পড়ে `/proc` থেকে, যা এখনও হোস্টের PID namespace-এর; তাই নতুন mount namespace-এ নতুন `/proc` মাউন্ট করতে `--mount-proc` লাগে।

5. প্রতি ১০০,০০০ µs (১০০ ms) সময়ে ওই cgroup-এর process-রা মিলে সর্বোচ্চ ২৫,০০০ µs (২৫ ms) CPU সময় পাবে — অর্থাৎ একটা CPU-র ২৫%।

6. নিচের (read-only) স্তর বদলায় না; upperdir-এ ওই নামে একটা **whiteout** ফাইল (character device 0/0) তৈরি হয়, যা merged দৃশ্যে ফাইলটাকে লুকিয়ে রাখে।

7. `-d` ব্যাকগ্রাউন্ডে; `-p 8080:80` হোস্টের 8080 → container-এর 80; `-v data:/data` — `data` `/` বা `./` দিয়ে শুরু হয়নি, তাই এটা **named volume**, container-এর `/data`-তে মাউন্ট; `--name app` container-এর নাম; `myimg` image।

8. Layer cache-এর জন্য। dependency তালিকা কম বদলায়; কোড বেশি বদলায়। আগে `requirements.txt` কপি করে install করলে, শুধু কোড বদলালে install স্তর ক্যাশ থেকে আসে — build অনেক দ্রুত হয়।

9. JSON (exec) form-এ `python` সরাসরি PID 1 হয় এবং `docker stop`-এর SIGTERM পায়, ভদ্রভাবে বন্ধ হতে পারে। shell form-এ `/bin/sh -c` PID 1 হয়, signal প্রায়ই অ্যাপে পৌঁছায় না, ফলে ১০ সেকেন্ড পর SIGKILL দিয়ে জোর করে মারা হয়।

10. ভেতরের UID 0 হোস্টে তোমার নিজের সাধারণ UID (যেমন 1000); বাকি UID-গুলো `/etc/subuid`/`/etc/subgid`-এ বরাদ্দ subordinate রেঞ্জে (যেমন 100000+) ম্যাপ হয়। এটা **user namespace**-এর উপর নির্ভর করে (`newuidmap`/`newgidmap` ম্যাপিং লেখে)। আর `docker` গ্রুপের সদস্য root হিসেবে চলা `dockerd`-কে যেকোনো আদেশ দিতে পারে, যেমন `docker run -v /:/host ...` দিয়ে হোস্টের পুরো ফাইলসিস্টেম root হিসেবে মাউন্ট করা — তাই কার্যত root-এর সমান।

</details>

---

[← আগের অধ্যায়](47-kernel-proc-sys.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](appendix-a-shortcuts-cheatsheet.md)

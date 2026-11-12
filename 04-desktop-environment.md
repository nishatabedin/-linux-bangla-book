# অধ্যায় ৪: ডেস্কটপ পরিবেশ ও GUI পরিচিতি

লিনাক্স ইনস্টল করে প্রথমবার যখন কম্পিউটার চালু করো, তখন যা চোখে পড়ে তা হলো লগইন স্ক্রিন, তারপর একটা সুন্দর ডেস্কটপ, প্যানেল, আইকন আর উইন্ডো। এগুলো একসাথে মিলে **GUI** (Graphical User Interface, মানে ছবি আর মাউস দিয়ে কম্পিউটার চালানোর ব্যবস্থা)। Windows বা macOS-এ এই GUI অপারেটিং সিস্টেমের সাথে একদম জোড়া লাগানো। লিনাক্সে ব্যাপারটা আলাদা। এখানে GUI কয়েকটা আলাদা সফটওয়্যারের স্তর দিয়ে তৈরি, আর প্রতিটা স্তর চাইলে বদলে ফেলা যায়।

এই অধ্যায়ে আমরা এই স্তরগুলো খুলে দেখব। কোনটা কী কাজ করে, কেন এভাবে বানানো, আর দৈনন্দিন কাজে কোন কী-বোর্ড শর্টকাটগুলো তোমার সময় বাঁচাবে, সব এখানে আছে।

## এই অধ্যায়ে যা শিখবে

- লিনাক্স GUI-এর স্তরগুলো: kernel, display server, compositor, window manager, display manager ও desktop environment, এবং এদের পার্থক্য
- X11 আর Wayland কী, কেন Wayland আসছে, আর তোমার সিস্টেম কোনটা চালাচ্ছে তা কীভাবে বুঝবে
- জনপ্রিয় desktop environment: GNOME, KDE Plasma, XFCE, Cinnamon (এবং আরও কিছু)
- ফাইল ম্যানেজার (Nautilus, Dolphin, Thunar, Nemo) ব্যবহার ও শর্টকাট
- সফটওয়্যার সেন্টার (GNOME Software, Ubuntu App Center, KDE Discover), Flatpak ও Snap-এর পরিচয়
- সেটিংস অ্যাপ, আর কমান্ড লাইন থেকে `gsettings` দিয়ে সেটিংস বদলানো
- workspace / virtual desktop দিয়ে কাজ গুছিয়ে রাখা
- GNOME ও KDE Plasma-র সম্পূর্ণ কী-বোর্ড শর্টকাট টেবিল
- টার্মিনাল খোলার সব উপায়, যা পরের অধ্যায়ের দরজা খুলে দেবে

---

## ৪.১ GUI আসলে কী, আর লিনাক্সে এটা আলাদা কেন

কম্পিউটারের আসল কাজ চালায় **kernel** (অপারেটিং সিস্টেমের মূল অংশ, যে হার্ডওয়্যার আর প্রোগ্রামের মাঝে মধ্যস্থতা করে; বিস্তারিত [অধ্যায় ২](02-linux-architecture.md)-এ আছে)। kernel নিজে কোনো উইন্ডো, বাটন বা মাউস পয়েন্টার আঁকে না। সে শুধু জানে কীভাবে গ্রাফিক্স কার্ডে পিক্সেল পাঠাতে হয় আর কী-বোর্ড-মাউসের ইনপুট পড়তে হয়।

উইন্ডো আঁকা, কোন উইন্ডো কার উপরে থাকবে ঠিক করা, প্যানেল-মেনু দেখানো, এসব কাজ করে **user space**-এর (kernel-এর বাইরের সাধারণ প্রোগ্রামের জগৎ) কিছু প্রোগ্রাম। লিনাক্সে এই প্রোগ্রামগুলো আলাদা আলাদা প্রজেক্ট, আলাদা দল বানায়। তাই:

- একই Ubuntu-তে তুমি GNOME, KDE বা XFCE, যেকোনোটা চালাতে পারো।
- সার্ভারে কোনো GUI-ই রাখতে হয় না। এতে RAM আর CPU বাঁচে, নিরাপত্তার ঝুঁকিও কমে।
- GUI ক্র্যাশ করলেও kernel আর বাকি সিস্টেম সাধারণত চলতেই থাকে।

> 💡 **উপমা:** একটা রেস্টুরেন্টের কথা ভাবো। রান্নাঘর হলো kernel। খাবার পরিবেশনের জন্য টেবিল-চেয়ার (display server), কোন টেবিলে কে বসবে ঠিক করা ম্যানেজার (window manager), দরজায় দাঁড়ানো রিসেপশনিস্ট (display manager), আর পুরো ডেকোরেশন, মেনুকার্ড, লাইটিং (desktop environment)। Windows-এ পুরো রেস্টুরেন্ট একটা কোম্পানির প্যাকেজ। লিনাক্সে তুমি রিসেপশনিস্ট বা ডেকোরেশন আলাদাভাবে বদলাতে পারো।

---

## ৪.২ GUI-এর স্তরগুলো: কে কী করে

এই অংশটা এই অধ্যায়ের সবচেয়ে গুরুত্বপূর্ণ জায়গা। নামগুলো একরকম শোনায়, তাই নতুনরা প্রায়ই গুলিয়ে ফেলে।

### ৪.২.১ Display Server (ডিসপ্লে সার্ভার)

**Display server** হলো সেই প্রোগ্রাম যে GUI প্রোগ্রাম আর হার্ডওয়্যারের মাঝখানে বসে থাকে। তার কাজ:

1. কী-বোর্ড, মাউস, টাচপ্যাডের ইনপুট kernel থেকে নিয়ে সঠিক উইন্ডোতে পৌঁছে দেওয়া।
2. অ্যাপগুলো যা আঁকতে চায় তা স্ক্রিনে দেখানোর ব্যবস্থা করা।

লিনাক্সে দুটো display প্রযুক্তি চলে: পুরোনো **X11** (X Window System, যার প্রধান বাস্তবায়ন **Xorg**) আর নতুন **Wayland**। এ নিয়ে ৪.৩ অংশে বিস্তারিত বলছি।

> 📝 **মনে রাখো:** "সার্ভার" শব্দ শুনে ইন্টারনেট সার্ভার ভেবো না। এখানে সার্ভার মানে "সেবা দেওয়া প্রোগ্রাম"। অ্যাপগুলো ক্লায়েন্ট, তারা display server-এর কাছে অনুরোধ পাঠায়: "আমার উইন্ডোটা দেখাও।"

### ৪.২.২ Window Manager (উইন্ডো ম্যানেজার)

**Window manager (WM)** ঠিক করে:

- উইন্ডো কোথায় বসবে, কত বড় হবে
- উইন্ডোর টাইটেল বার, minimize/maximize/close বাটন (এগুলোকে বলে *decoration*)
- কোন উইন্ডো focus পাবে (মানে কী-বোর্ডে টাইপ করলে কোন উইন্ডোতে যাবে)
- উইন্ডো সরানো, resize, পাশাপাশি সাজানো (tiling)

WM দুই রকম:

| ধরন | কীভাবে কাজ করে | উদাহরণ |
|---|---|---|
| Stacking (floating) | উইন্ডো একটার উপর আরেকটা কাগজের মতো স্তূপ হয়ে থাকে | Mutter, KWin, Xfwm4, Muffin, Openbox |
| Tiling | উইন্ডোগুলো নিজে থেকে স্ক্রিন ভাগ করে টাইলের মতো বসে, ওভারল্যাপ হয় না | i3, Sway, Hyprland, bspwm, dwm |

### ৪.২.৩ Compositor (কম্পোজিটর)

আগে প্রতিটা অ্যাপ সরাসরি স্ক্রিনে আঁকত। **Compositor** আসার পর প্রতিটা উইন্ডো আগে নিজের জন্য আলাদা মেমরিতে (off-screen buffer) আঁকে, তারপর compositor সব উইন্ডোর ছবি একসাথে জুড়ে (composite করে) চূড়ান্ত স্ক্রিন বানায়। এতেই সম্ভব হয়েছে স্বচ্ছতা (transparency), ছায়া, অ্যানিমেশন, আর tearing ছাড়া মসৃণ ছবি।

X11-এ compositor একটা আলাদা ঐচ্ছিক জিনিস ছিল (যেমন `picom`)। Wayland-এ compositor-ই display server, আর সাধারণত সে-ই window manager। GNOME-এর **Mutter** আর KDE-র **KWin** একাই তিনটা কাজ করে: display server (Wayland-এ), compositor, window manager।

### ৪.২.৪ Display Manager (ডিসপ্লে ম্যানেজার / লগইন ম্যানেজার)

বুট শেষে যে গ্রাফিক্যাল লগইন স্ক্রিন আসে, ইউজারের নাম বাছো, পাসওয়ার্ড দাও, সেটাই **display manager (DM)**। নাম "display manager" হলেও এটা আসলে **লগইন ম্যানেজার**। এর কাজ:

- ইউজারের পাসওয়ার্ড যাচাই করা (PAM-এর মাধ্যমে, [অধ্যায় ৪৫](45-security-hardening.md) দেখো)
- কোন session (GNOME, Plasma, XFCE...) চালু হবে তা বাছতে দেওয়া
- সেই session চালু করে দেওয়া

| Display Manager | সাধারণত যার সাথে আসে |
|---|---|
| GDM (GNOME Display Manager; Debian/Ubuntu-তে প্যাকেজ নাম `gdm3`) | GNOME: Ubuntu, Fedora Workstation, RHEL, Debian GNOME |
| SDDM (Simple Desktop Display Manager) | KDE Plasma: Kubuntu, Fedora KDE |
| LightDM | XFCE, Cinnamon, MATE: Xubuntu, Linux Mint |
| plasma-login-manager / greetd ইত্যাদি | নতুন বা হালকা বিকল্প |

display manager নিজে একটা systemd service। এটা কীভাবে বুটের সময় চালু হয় তা [অধ্যায় ২৬](26-systemd-services.md) আর [অধ্যায় ২৭](27-boot-process.md)-এ দেখবে।

### ৪.২.৫ Desktop Environment (ডেস্কটপ এনভায়রনমেন্ট)

**Desktop environment (DE)** একটা পুরো প্যাকেজ। এর ভেতরে থাকে:

- একটা window manager/compositor
- প্যানেল, টাস্কবার, অ্যাপ মেনু বা লঞ্চার
- নোটিফিকেশন সিস্টেম
- ফাইল ম্যানেজার
- সেটিংস অ্যাপ
- টার্মিনাল, টেক্সট এডিটর, ইমেজ ভিউয়ারের মতো নিজস্ব অ্যাপ
- একই রকম দেখতে থিম, আইকন, ফন্ট

মানে window manager শুধু উইন্ডো সামলায়, আর desktop environment তোমাকে পুরো "কম্পিউটার চালানোর অভিজ্ঞতা" দেয়।

### ৪.২.৬ সব স্তর একসাথে: ডায়াগ্রাম

```
 ┌───────────────────────────────────────────────────────────────┐
 │                     তুমি (ইউজার)                                │
 │          চোখে দেখো স্ক্রিন  ▲          ▼ হাতে চাপো কী-বোর্ড/মাউস    │
 └──────────────────────────┼──────────┼─────────────────────────┘
                            │          │
 ┌──────────────────────────┴──────────┴─────────────────────────┐
 │  Applications: Firefox, LibreOffice, Terminal, Files ...      │
 │  (GTK বা Qt টুলকিট দিয়ে বানানো)                                  │
 ├───────────────────────────────────────────────────────────────┤
 │  Desktop Environment: GNOME / KDE Plasma / XFCE / Cinnamon    │
 │  (প্যানেল, মেনু, নোটিফিকেশন, সেটিংস, ফাইল ম্যানেজার)                 │
 ├───────────────────────────────────────────────────────────────┤
 │  Window Manager + Compositor: Mutter / KWin / Xfwm4 / Muffin  │
 ├───────────────────────────────────────────────────────────────┤
 │  Display Server: Wayland compositor  অথবা  Xorg (X11)        │
 ├───────────────────────────────────────────────────────────────┤
 │  গ্রাফিক্স লাইব্রেরি: Mesa (OpenGL/Vulkan), libinput (ইনপুট)          │
 ├═══════════════════════ user space / kernel ═══════════════════┤
 │  Linux Kernel: DRM/KMS (গ্রাফিক্স ড্রাইভার), evdev (ইনপুট ডিভাইস)     │
 ├───────────────────────────────────────────────────────────────┤
 │  হার্ডওয়্যার: GPU, মনিটর, কী-বোর্ড, মাউস, টাচপ্যাড                   │
 └───────────────────────────────────────────────────────────────┘

 Display Manager (GDM/SDDM/LightDM) এই পুরো স্ট্যাক শুরু হওয়ার আগে
 লগইন স্ক্রিন দেখায়, তারপর তোমার বাছাই করা session চালু করে দেয়।
```

বুট থেকে ডেস্কটপ পর্যন্ত ক্রমটা এরকম:

```
 kernel বুট ─► systemd ─► graphical.target ─► display-manager.service (GDM)
                                                        │
                                       লগইন স্ক্রিন, পাসওয়ার্ড যাচাই
                                                        │
                                              session চালু (gnome-shell)
                                                        │
                                   Wayland compositor + প্যানেল + অ্যাপ
```

> 💡 **টিপস:** **GTK** আর **Qt** হলো দুটো "টুলকিট", মানে বাটন, মেনু, টেক্সটবক্স বানানোর লাইব্রেরি। GNOME, XFCE, Cinnamon GTK ব্যবহার করে, আর KDE Plasma ব্যবহার করে Qt। যেকোনো DE-তে যেকোনো টুলকিটের অ্যাপ চলে, শুধু দেখতে একটু বেমানান লাগতে পারে।

---

## ৪.৩ X11 বনাম Wayland

### ৪.৩.১ X11 (X Window System): পুরোনো সৈনিক

X11 এর জন্ম ১৯৮৪ সালে MIT-তে। এর সংস্করণ ১১ (X11) ১৯৮৭ থেকে চলছে। লিনাক্সে এর বাস্তবায়ন **Xorg server**।

X11-এর ডিজাইন **নেটওয়ার্ক-কেন্দ্রিক**। তখনকার দিনে একটা বড় কম্পিউটারে প্রোগ্রাম চলত, আর অনেক দুর্বল টার্মিনাল নেটওয়ার্ক দিয়ে তার ছবি দেখত। তাই X-এ:

```
  X11 মডেল (সরলীকৃত)

  ┌─────────┐   আঁকার অনুরোধ    ┌──────────────┐   ছবি ফেরত    ┌─────────────┐
  │  অ্যাপ    │ ───────────────► │  Xorg server │ ────────────► │ Compositor  │
  │ (client)│ ◄─────────────── │  (মধ্যস্থতাকারী) │ ◄──────────── │ (+ WM, আলাদা)│
  └─────────┘   ইনপুট ইভেন্ট      └──────┬───────┘  চূড়ান্ত ছবি     └─────────────┘
                                       │
                                  kernel / স্ক্রিন
```

সমস্যাগুলো:

- **নিরাপত্তা:** X11-এ যেকোনো অ্যাপ অন্য যেকোনো অ্যাপের কী-বোর্ড ইনপুট পড়তে পারে, স্ক্রিনশট নিতে পারে। একটা দুষ্ট প্রোগ্রাম তোমার ব্রাউজারে টাইপ করা পাসওয়ার্ড শুনে ফেলতে পারে (keylogging)।
- **জটিলতা:** কয়েক দশকের পুরোনো কোড আর এমন অনেক ফিচার যা এখন কেউ ব্যবহার করে না।
- **অতিরিক্ত ধাপ:** ছবি app → X server → compositor → X server → স্ক্রিন, এভাবে ঘুরে যায়। এতে বিলম্ব আর tearing হয়।
- আধুনিক জিনিস যেমন বিভিন্ন মনিটরে বিভিন্ন স্কেলিং (একটায় 100%, আরেকটায় 150%), HDR, variable refresh rate এসবে দুর্বল।

### ৪.৩.২ Wayland: নতুন প্রজন্ম

Wayland ২০০৮ সালে শুরু হয়। এটা কোনো একক প্রোগ্রাম নয়, একটা **protocol** (অ্যাপ আর compositor কীভাবে কথা বলবে তার নিয়মকানুন)। প্রতিটা DE নিজের compositor-এ এই protocol বাস্তবায়ন করে: GNOME-এর Mutter, KDE-র KWin, Sway, Hyprland ইত্যাদি।

```
  Wayland মডেল

  ┌─────────┐  "আমার উইন্ডোর ছবি এই buffer-এ আছে"  ┌──────────────────────────┐
  │  অ্যাপ    │ ───────────────────────────────►    │  Wayland Compositor       │
  │ (client)│ ◄───────────────────────────────    │ (display server + WM +    │
  └─────────┘   শুধু নিজের উইন্ডোর ইনপুট              │  compositor, একই প্রোগ্রাম) │
                                                  └────────────┬─────────────┘
                                                               │ KMS/DRM
                                                          kernel / স্ক্রিন
```

সুবিধা:

- **নিরাপত্তা:** একটা অ্যাপ শুধু নিজের উইন্ডোর ইনপুট পায়। স্ক্রিনশট বা স্ক্রিন শেয়ার করতে হলে **xdg-desktop-portal** আর **PipeWire**-এর মাধ্যমে অনুমতি চাইতে হয়। এজন্যই Zoom/Meet-এ স্ক্রিন শেয়ার করলে Wayland-এ একটা "কোন স্ক্রিন শেয়ার করবে?" ডায়ালগ আসে।
- **মসৃণতা:** "every frame is perfect", মানে tearing হয় না।
- **আধুনিক ফিচার:** প্রতি মনিটরে আলাদা fractional scaling, HDR (নতুন GNOME/Plasma-তে), touchpad gesture।

**XWayland:** যেসব পুরোনো অ্যাপ এখনো শুধু X11 বোঝে, তাদের জন্য Wayland session-এর ভেতরে একটা ছোট X server চলে, নাম **XWayland**। তাই পুরোনো অ্যাপও Wayland-এ সাধারণত ঠিকঠাক চলে।

### ৪.৩.৩ পাশাপাশি তুলনা

| বিষয় | X11 (Xorg) | Wayland |
|---|---|---|
| শুরু | ১৯৮৪ (X11: ১৯৮৭) | ২০০৮ |
| গঠন | আলাদা X server + আলাদা WM + আলাদা compositor | compositor-ই server, WM সাধারণত একই প্রোগ্রামে |
| নিরাপত্তা | যেকোনো অ্যাপ অন্যের ইনপুট/স্ক্রিন দেখতে পারে | অ্যাপগুলো একে অপরের থেকে আলাদা |
| স্ক্রিন শেয়ার/স্ক্রিনশট | সরাসরি | portal + PipeWire দিয়ে, অনুমতি নিয়ে |
| Tearing | হতে পারে | মূলত হয় না |
| Fractional / মিশ্র স্কেলিং | দুর্বল | ভালো |
| নেটওয়ার্কে GUI (`ssh -X`) | বিল্ট-ইন | নেই; বিকল্প `waypipe`, RDP |
| পুরোনো অ্যাপ | সব চলে | XWayland দিয়ে চলে |
| `xdotool`, `xrandr`, `xkill` জাতীয় টুল | কাজ করে | অনেকগুলো কাজ করে না (Wayland-নিজস্ব বিকল্প লাগে) |

### ৪.৩.৪ Distro-ভেদে বর্তমান অবস্থা

| Distro | ডিফল্ট |
|---|---|
| Ubuntu 24.04 LTS (GNOME 46) | Wayland। লগইন স্ক্রিনে "Ubuntu on Xorg" বিকল্প এখনো আছে |
| Debian 12 (GNOME 43) | Wayland। কিছু পুরোনো NVIDIA ড্রাইভারে Xorg-এ ফিরে যায় |
| Fedora Workstation (GNOME) | Wayland। Fedora 43 থেকে GNOME-এর X11 session আর নেই |
| Fedora KDE (Plasma 6) | Wayland only (Fedora 40 থেকে X11 session নেই) |
| RHEL 9 | Wayland ডিফল্ট, Xorg session এখনো আছে (deprecated) |
| RHEL 10 | Xorg server সরানো হয়েছে, শুধু Wayland (পুরোনো অ্যাপের জন্য XWayland) |
| Linux Mint (Cinnamon) | X11 ডিফল্ট, Wayland পরীক্ষামূলক |
| Xubuntu / XFCE | X11 ডিফল্ট, XFCE 4.20 থেকে Wayland আংশিক/পরীক্ষামূলক |

> ⚠️ **সতর্কতা:** পুরোনো টিউটোরিয়ালে প্রায়ই `xrandr`, `xdotool`, `xkill` এর মতো কমান্ড পাবে। এগুলো X11-এর জন্য বানানো। Wayland session-এ এরা হয় কাজ করে না, নয়তো শুধু XWayland-এ চলা অ্যাপের উপর কাজ করে। কোনো কমান্ড "কিছু হচ্ছে না" মনে হলে আগে দেখো তুমি কোন session-এ আছো।

### ৪.৩.৫ তুমি কোন session-এ আছো, কীভাবে বুঝবে

টার্মিনাল খোলো (কীভাবে খুলবে তা ৪.১০ অংশে আছে; শর্টকাটে Ubuntu-তে `Ctrl+Alt+T`) আর লেখো:

```
$ echo $XDG_SESSION_TYPE
wayland
```

`x11` দেখালে তুমি X11-এ আছো, `tty` দেখালে টেক্সট কনসোলে।

কোন desktop environment চলছে:

```
$ echo $XDG_CURRENT_DESKTOP
ubuntu:GNOME
```

Fedora-তে শুধু `GNOME`, KDE-তে `KDE`, XFCE-তে `XFCE`, Mint-এ `X-Cinnamon` দেখাবে।

আরেকটু বিস্তারিত দেখতে `loginctl` ব্যবহার করো (এটা systemd-এর লগইন ম্যানেজার `systemd-logind`-এর টুল):

```
$ loginctl
SESSION  UID USER  SEAT  LEAVER TTY  STATE  IDLE SINCE
      2 1000 rahim seat0 -      tty2 active no   -

1 sessions listed.

$ loginctl show-session 2 -p Type -p Desktop -p Service
Type=wayland
Desktop=ubuntu
Service=gdm-password
```

> 📝 **মনে রাখো:** `loginctl`-এর আউটপুটের কলাম systemd-এর সংস্করণভেদে একটু আলাদা হতে পারে। তোমার session নম্বর `2` না হয়ে অন্য কিছু হতে পারে; প্রথম কমান্ডের আউটপুট থেকে নম্বরটা নিয়ে দ্বিতীয় কমান্ডে বসাও।

Wayland-এ থাকলে দুটো environment variable থাকে:

```
$ echo $WAYLAND_DISPLAY
wayland-0
$ echo $DISPLAY
:0
```

`DISPLAY` থাকে কারণ XWayland চালু থাকে পুরোনো অ্যাপের জন্য। Environment variable জিনিসটা কী, তা [অধ্যায় ৪৪](44-environment-bashrc.md)-এ শিখবে।

### ৪.৩.৬ Session বদলানো

1. Log out করো।
2. লগইন স্ক্রিনে তোমার নামে ক্লিক করো।
3. GDM-এ নিচের ডান কোণে একটা গিয়ার (⚙) আইকন আসবে; SDDM-এ নিচের বাম দিকে session-এর ড্রপডাউন; LightDM-এ সাধারণত উপরের প্যানেলে আইকন।
4. সেখান থেকে "Ubuntu" (Wayland) বা "Ubuntu on Xorg" বা অন্য DE বেছে নাও, তারপর পাসওয়ার্ড দাও।

> 💡 **টিপস:** নতুন হিসেবে ডিফল্ট Wayland-ই রাখো। শুধু যদি কোনো নির্দিষ্ট অ্যাপ (পুরোনো স্ক্রিন রেকর্ডার, রিমোট সাপোর্ট টুল, কিছু গেম) ঠিকমতো না চলে, তখন Xorg session চেষ্টা করো।

---

## ৪.৪ জনপ্রিয় Desktop Environment

### ৪.৪.১ GNOME

Ubuntu, Fedora Workstation, Debian (ডিফল্ট), RHEL-এর ডিফল্ট DE। ২০১১ সালের GNOME 3 থেকে এর ডিজাইন বেশ আলাদা: কোনো ক্লাসিক স্টার্ট মেনু নেই, কোনো টাস্কবার নেই।

মূল অংশগুলো:

```
 ┌──────────────────────────────────────────────────────────────────┐
 │ Activities         বুধ ১২ নভেম্বর ১০:৩০            🔊 🔋 ⏻  ◄── Top bar │
 ├────┬─────────────────────────────────────────────────────────────┤
 │ 🦊 │                                                             │
 │ 📁 │          (Ubuntu-তে বাম দিকে এই Dock থাকে।                     │
 │ 📝 │           খাঁটি GNOME, যেমন Fedora-তে, Dock শুধু                │
 │ ⚙  │           Activities overview খুললে নিচে দেখা যায়)              │
 │    │                                                             │
 │ ⣿  │ ◄── Show Apps (সব অ্যাপের grid)                              │
 └────┴─────────────────────────────────────────────────────────────┘
```

- **Top bar:** বাম দিকে "Activities" বা workspace নির্দেশক, মাঝে ঘড়ি (ক্লিক করলে ক্যালেন্ডার আর নোটিফিকেশন), ডানে **Quick Settings** (Wi-Fi, ব্লুটুথ, শব্দ, ব্যাটারি, ডার্ক মোড, পাওয়ার)।
- **Activities Overview:** `Super` কী (কী-বোর্ডের Windows লোগো কী) চাপলে খোলে। সব খোলা উইন্ডো, workspace আর একটা সার্চ বক্স একসাথে দেখা যায়।
- **Search:** Overview খুলে সরাসরি টাইপ করো, অ্যাপ, ফাইল, সেটিংস, এমনকি হিসাব (`25*4`) পর্যন্ত খুঁজে দেয়।
- **Dash/Dock:** প্রিয় অ্যাপগুলোর সারি।
- **Extensions:** GNOME-এর চেহারা আর আচরণ বদলানো যায় extension দিয়ে। Ubuntu-র Dock আর ডেস্কটপ আইকন আসলে Ubuntu-র যোগ করা extension।

শক্তি: পরিষ্কার, কম বিভ্রান্তিকর, টাচপ্যাড gesture চমৎকার (তিন আঙুলে উপরে সোয়াইপ করলে overview, পাশে করলে workspace বদল)।
দুর্বলতা: অনেক জিনিস কাস্টমাইজ করতে extension বা **GNOME Tweaks** (`gnome-tweaks`) লাগে। RAM বেশি লাগে (খালি অবস্থায় মোটামুটি ১–১.৫ GB)।

### ৪.৪.২ KDE Plasma

Kubuntu, Fedora KDE, openSUSE, KDE neon-এর ডিফল্ট। বর্তমানে **Plasma 6** চলছে। Qt টুলকিটে বানানো।

- Windows-এর মতো চেনা লেআউট: নিচে প্যানেল (টাস্কবার), বাম কোণে **Application Launcher** (Kickoff), ডানে **System Tray** আর ঘড়ি।
- **KRunner:** `Alt+Space` বা `Alt+F2` চাপলে উপরে একটা সার্চ বার আসে; অ্যাপ চালানো, হিসাব, ইউনিট কনভার্শন, ফাইল খোঁজা সব এখান থেকে।
- **Widgets:** ডেস্কটপ আর প্যানেলে ঘড়ি, সিস্টেম মনিটর, নোট ইত্যাদি বসানো যায়।
- **Activities:** virtual desktop-এর চেয়েও বড় ধারণা। "অফিস" আর "বাড়ি" নামে দুটো Activity বানালে প্রতিটার আলাদা ওয়ালপেপার, উইজেট আর খোলা অ্যাপ থাকে।
- **Overview:** `Meta+W` চাপলে সব উইন্ডো আর ডেস্কটপ একসাথে দেখা যায় (KDE-তে Windows কী-কে বলে **Meta**)।

শক্তি: প্রায় সবকিছু কাস্টমাইজ করা যায়, নিজস্ব অ্যাপগুলো শক্তিশালী (Dolphin, Konsole, Kate, Spectacle, Okular)। আধুনিক Plasma বেশ হালকাও।
দুর্বলতা: অপশন এত বেশি যে নতুনদের মাথা ঘুরতে পারে।

### ৪.৪.৩ XFCE

Xubuntu-র ডিফল্ট; Debian, Fedora, Mint-এ ঐচ্ছিক সংস্করণ আছে। GTK-ভিত্তিক।

- হালকা আর স্থির: খালি অবস্থায় সাধারণত ৫০০ MB-এর কম RAM-এ চলে।
- পুরোনো ধাঁচের পরিচিত লেআউট: প্যানেল, Whisker মেনু, ডেস্কটপ আইকন।
- উইন্ডো ম্যানেজার **Xfwm4**, ফাইল ম্যানেজার **Thunar**।
- পুরোনো ল্যাপটপ বা কম RAM-এর মেশিনের জন্য আদর্শ।
- দুর্বলতা: এখনো মূলত X11-নির্ভর; চেহারা কিছুটা পুরোনো ধাঁচের।

### ৪.৪.৪ Cinnamon

Linux Mint-এর বানানো, Mint-এর সবচেয়ে জনপ্রিয় সংস্করণ। GNOME থেকে জন্ম নিলেও দেখতে Windows 7/10-এর মতো:

- নিচে প্যানেল, বাম কোণে মেনু, ডানে system tray।
- উইন্ডো ম্যানেজার **Muffin** (Mutter-এর fork), ফাইল ম্যানেজার **Nemo**।
- Windows থেকে আসা নতুনদের জন্য সবচেয়ে কম ধাক্কা দেয়।
- Wayland সাপোর্ট এখনো পরীক্ষামূলক।

### ৪.৪.৫ আরও কিছু DE ও WM

| নাম | ধরন | কাদের জন্য |
|---|---|---|
| MATE | পুরোনো GNOME 2-এর ধারাবাহিকতা, হালকা | ক্লাসিক লুক পছন্দ যাদের |
| LXQt | খুব হালকা, Qt-ভিত্তিক | খুব পুরোনো হার্ডওয়্যার (Lubuntu) |
| Budgie | আধুনিক, সরল | Ubuntu Budgie |
| COSMIC | System76-এর Rust-এ লেখা নতুন DE, Wayland-only | Pop!_OS |
| i3 / Sway | tiling WM (Sway হলো Wayland-এ i3-এর সমতুল্য) | কী-বোর্ড প্রেমী পাওয়ার ইউজার |
| Hyprland | অ্যানিমেশনসমৃদ্ধ tiling Wayland compositor | কাস্টমাইজেশন প্রেমী |

### ৪.৪.৬ কোনটা বেছে নেবে: তুলনা টেবিল

| বৈশিষ্ট্য | GNOME | KDE Plasma | XFCE | Cinnamon |
|---|---|---|---|---|
| টুলকিট | GTK | Qt | GTK | GTK |
| WM/Compositor | Mutter | KWin | Xfwm4 | Muffin |
| Display manager (সাধারণত) | GDM | SDDM | LightDM | LightDM |
| ফাইল ম্যানেজার | Files (Nautilus) | Dolphin | Thunar | Nemo |
| টার্মিনাল | GNOME Terminal / Ptyxis | Konsole | xfce4-terminal | GNOME Terminal |
| সেটিংস অ্যাপ | Settings (`gnome-control-center`) | System Settings (`systemsettings`) | `xfce4-settings-manager` | `cinnamon-settings` |
| সফটওয়্যার সেন্টার | GNOME Software / Ubuntu App Center | Discover | (distro অনুযায়ী) | Mint Software Manager |
| Wayland | পূর্ণ, ডিফল্ট | পূর্ণ, ডিফল্ট | পরীক্ষামূলক | পরীক্ষামূলক |
| RAM (আনুমানিক, খালি অবস্থায়) | বেশি | মাঝারি | কম | মাঝারি |
| কাস্টমাইজেশন | extension দিয়ে | সর্বোচ্চ | অনেক | ভালো |
| কার জন্য | সরল, আধুনিক কাজ | পাওয়ার ইউজার, Windows-অভ্যস্ত | পুরোনো PC | Windows থেকে আসা নতুনরা |

> 📝 **মনে রাখো:** RAM-এর সংখ্যাগুলো আনুমানিক। distro, সংস্করণ আর কোন অ্যাপ background-এ চলছে তার উপর অনেক বদলায়। নিজের মেশিনে মাপতে `free -h` ব্যবহার করো ([অধ্যায় ৪৬](46-performance-troubleshooting.md))।

### ৪.৪.৭ একাধিক DE ইনস্টল করা

একই সিস্টেমে একাধিক DE রাখা যায়, লগইন স্ক্রিন থেকে বেছে নেওয়া যায়। প্যাকেজ ম্যানেজার নিয়ে বিস্তারিত [অধ্যায় ৩০](30-package-management.md)-এ; এখানে শুধু উদাহরণ:

```
# Ubuntu-তে KDE Plasma
$ sudo apt install kubuntu-desktop

# Ubuntu-তে XFCE
$ sudo apt install xubuntu-desktop

# Debian-এ (tasksel-এর task প্যাকেজ)
$ sudo apt install task-kde-desktop

# Fedora-তে KDE Plasma
$ sudo dnf install @kde-desktop-environment
```

ইনস্টলের সময় Ubuntu/Debian জিজ্ঞেস করতে পারে কোন display manager ডিফল্ট হবে (`gdm3` না `sddm`)। পরে বদলাতে:

```
$ sudo dpkg-reconfigure gdm3
```

এটা একটা মেনু খুলবে, সেখানে ইনস্টল থাকা display manager-গুলো থেকে বেছে নাও। Debian/Ubuntu-তে বর্তমান ডিফল্টটা এই ফাইলে লেখা থাকে:

```
$ cat /etc/X11/default-display-manager
/usr/sbin/gdm3
```

Fedora/RHEL-এ display manager হলো `display-manager.service` নামের একটা symlink। কোনটা চলছে দেখতে:

```
$ systemctl status display-manager
● gdm.service - GNOME Display Manager
     Loaded: loaded (/usr/lib/systemd/system/gdm.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-11-12 09:02:11 +06; 1h 28min ago
...
```

> ⚠️ **সতর্কতা:** "kubuntu-desktop" বা "xubuntu-desktop" এর মতো meta-package অনেক অ্যাপ নিয়ে আসে, ডাউনলোড ১–২ GB হতে পারে, আর মেনুতে প্রতিটা কাজের দুটো করে অ্যাপ দেখা যাবে। শুধু চেখে দেখতে চাইলে আগে Live USB বা virtual machine-এ ([অধ্যায় ৪৮](48-containers-virtualization.md)) চেষ্টা করো।

---

## ৪.৫ ফাইল ম্যানেজার

ফাইল ম্যানেজার হলো Windows-এর File Explorer বা macOS-এর Finder-এর সমতুল্য। ভেতরে ভেতরে এটা ঠিক সেই কাজগুলোই করে যা তুমি টার্মিনালে `ls`, `cp`, `mv`, `rm` দিয়ে করবে ([অধ্যায় ৮](08-navigation.md), [অধ্যায় ৯](09-file-management.md))। পার্থক্য শুধু মাউস দিয়ে করা।

| DE | ফাইল ম্যানেজার | কমান্ড |
|---|---|---|
| GNOME | Files (আগের নাম Nautilus) | `nautilus` |
| KDE Plasma | Dolphin | `dolphin` |
| XFCE | Thunar | `thunar` |
| Cinnamon | Nemo | `nemo` |

### ৪.৫.১ সাধারণ ধারণা

- **Home folder:** প্রতিটা ইউজারের নিজস্ব ফোল্ডার, `/home/তোমার-নাম`, সংক্ষেপে `~`। ফাইল ম্যানেজার খুললে সাধারণত এখানেই শুরু হয়। ভেতরে থাকে `Desktop`, `Documents`, `Downloads`, `Music`, `Pictures`, `Videos` ইত্যাদি। এসব ফোল্ডারের নাম `~/.config/user-dirs.dirs` ফাইলে সংরক্ষিত থাকে।
- **Hidden file:** যে ফাইল বা ফোল্ডারের নাম ডট (`.`) দিয়ে শুরু, যেমন `.bashrc` বা `.config`, সেটা লুকানো থাকে। এটা কোনো নিরাপত্তা ব্যবস্থা নয়, শুধু চোখের আরামের জন্য। `Ctrl+H` চাপলে দেখা যায়।
- **Trash (রিসাইকেল বিন):** ফাইল ম্যানেজারে Delete চাপলে ফাইল আসলে মোছে না, `~/.local/share/Trash/files/`-এ সরে যায়। কিন্তু টার্মিনালের `rm` কমান্ড সরাসরি মুছে ফেলে, Trash-এ যায় না।
- **Other Locations / Devices:** USB ড্রাইভ, অন্য পার্টিশন, নেটওয়ার্ক শেয়ার এখানে দেখা যায়। USB ঢোকালে ফাইল ম্যানেজার নিজেই সাধারণত `/media/তোমার-নাম/ড্রাইভের-নাম`-এ (Fedora/RHEL-এ `/run/media/তোমার-নাম/...`) mount করে। Mount জিনিসটা কী তা [অধ্যায় ৩২](32-disk-partition-mount.md)-এ।
- **সার্ভারে সংযোগ:** অ্যাড্রেস বারে `sftp://user@192.168.1.10/` বা `smb://server/share` লিখে রিমোট ফোল্ডার খোলা যায় (GNOME-এ এটা **GVfs** নামের স্তর সামলায়, KDE-তে **KIO**)।

> ⚠️ **সতর্কতা:** `Shift+Delete` চাপলে ফাইল Trash-এ না গিয়ে সরাসরি মুছে যায়। টার্মিনালে `rm` করলেও তাই। এর কোনো "undo" নেই।

### ৪.৫.২ ফাইল ম্যানেজার থেকে টার্মিনালে যাওয়া

- **Nautilus:** ফোল্ডারের খালি জায়গায় রাইট-ক্লিক → "Open in Terminal" (Ubuntu-তে বিল্ট-ইন; অন্যান্য distro-তে `nautilus-extension-gnome-terminal` বা distro-ভেদে সমতুল্য প্যাকেজ লাগতে পারে)।
- **Dolphin:** `F4` চাপলে নিচে একটা ছোট টার্মিনাল প্যানেল আসে, যেটা ফোল্ডার বদলালে নিজেও সেই ফোল্ডারে যায়। `Shift+F4` আলাদা Konsole উইন্ডো খোলে।
- **Thunar / Nemo:** রাইট-ক্লিক → "Open Terminal Here"।

উল্টো দিকে, টার্মিনাল থেকে বর্তমান ফোল্ডার ফাইল ম্যানেজারে খুলতে:

```
$ xdg-open .
```

`xdg-open` যেকোনো ফাইল তার ডিফল্ট অ্যাপে খোলে: `xdg-open report.pdf` PDF ভিউয়ারে, `xdg-open https://kernel.org` ব্রাউজারে। এটা DE-নিরপেক্ষ, সব DE-তে চলে।

### ৪.৫.৩ ফাইল ম্যানেজারের কী-বোর্ড শর্টকাট

| কাজ | Nautilus (GNOME) | Dolphin (KDE) | Thunar / Nemo |
|---|---|---|---|
| লুকানো ফাইল দেখাও/লুকাও | `Ctrl+H` | `Alt+.` (অথবা `Ctrl+H`) | `Ctrl+H` |
| অ্যাড্রেস বারে পাথ লেখা | `Ctrl+L` | `Ctrl+L` | `Ctrl+L` |
| নতুন ফোল্ডার | `Ctrl+Shift+N` | `F10` | `Ctrl+Shift+N` |
| নাম বদলানো | `F2` | `F2` | `F2` |
| Trash-এ পাঠানো | `Delete` | `Delete` | `Delete` |
| স্থায়ীভাবে মোছা | `Shift+Delete` | `Shift+Delete` | `Shift+Delete` |
| কপি / কাট / পেস্ট | `Ctrl+C` / `Ctrl+X` / `Ctrl+V` | একই | একই |
| সব নির্বাচন | `Ctrl+A` | `Ctrl+A` | `Ctrl+A` |
| নতুন ট্যাব | `Ctrl+T` | `Ctrl+T` | `Ctrl+T` |
| নতুন উইন্ডো | `Ctrl+N` | `Ctrl+N` | `Ctrl+N` |
| উপরের (parent) ফোল্ডার | `Alt+↑` | `Alt+↑` বা `Backspace` | `Alt+↑` |
| পিছনে / সামনে | `Alt+←` / `Alt+→` | `Alt+←` / `Alt+→` | `Alt+←` / `Alt+→` |
| Home ফোল্ডার | `Alt+Home` | `Alt+Home` | `Alt+Home` |
| ফাইল খোঁজা | `Ctrl+F` (বা সরাসরি টাইপ) | `Ctrl+F` | `Ctrl+F` |
| Properties | `Alt+Enter` | `Alt+Enter` | `Alt+Enter` |
| রিফ্রেশ | `F5` বা `Ctrl+R` | `F5` | `F5` বা `Ctrl+R` |
| Undo | `Ctrl+Z` | `Ctrl+Z` | `Ctrl+Z` |
| Split view (দুই প্যানে) | নেই | `F3` | `F3` |
| সাইডবার দেখাও/লুকাও | `F9` | `F9` (Places প্যানেল) | `Ctrl+B` (Thunar), `F9` (Nemo) |
| টার্মিনাল প্যানেল | নেই | `F4` | নেই |
| বড়/ছোট আইকন | `Ctrl+=` / `Ctrl+-` | `Ctrl+=` / `Ctrl+-` | `Ctrl+=` / `Ctrl+-` |

> 📝 **মনে রাখো:** Dolphin-এর বেশির ভাগ শর্টকাট Settings → Configure Keyboard Shortcuts থেকে বদলানো যায়, আর সংস্করণভেদে দু-একটা ডিফল্ট বদলে যেতে পারে। মেনুতে প্রতিটা অপশনের পাশে তার বর্তমান শর্টকাট লেখা থাকে। সন্দেহ হলে মেনু খুলে দেখে নাও।

---

## ৪.৬ সফটওয়্যার সেন্টার

সফটওয়্যার সেন্টার হলো লিনাক্সের "অ্যাপ স্টোর"। পার্থক্য হলো, এখানের প্রায় সব সফটওয়্যার বিনামূল্যে, আর ভেতরে এটা আসলে **প্যাকেজ ম্যানেজারের** একটা গ্রাফিক্যাল মুখ।

```
  সফটওয়্যার সেন্টার (GUI)
        │
        ▼
  PackageKit  (DE-নিরপেক্ষ মধ্যস্থতাকারী সার্ভিস)
        │
   ┌────┴─────┬──────────┬──────────┐
   ▼          ▼          ▼          ▼
  apt/dpkg   dnf/rpm   flatpak     snap
 (Debian,   (Fedora,   (Flathub)  (Snap Store)
  Ubuntu)    RHEL)
```

| DE / Distro | সফটওয়্যার সেন্টার | কী কী সামলায় |
|---|---|---|
| Ubuntu 24.04 | **App Center** (`snap-store` snap প্যাকেজ) | Snap ডিফল্ট, `.deb` ফাইলও |
| Fedora, Debian GNOME | **GNOME Software** (`gnome-software`) | rpm/deb + Flatpak |
| KDE Plasma | **Discover** (`plasma-discover`) | distro প্যাকেজ + Flatpak (+ Snap, যদি প্লাগইন থাকে) |
| Linux Mint | **Software Manager** (`mintinstall`) | deb + Flatpak |

### ৪.৬.১ তিন ধরনের প্যাকেজ: সংক্ষিপ্ত পরিচয়

- **Distro প্যাকেজ (`.deb`, `.rpm`):** distro-র নিজস্ব repository থেকে আসে। সিস্টেমের লাইব্রেরি শেয়ার করে, তাই ছোট; কিন্তু LTS distro-তে সংস্করণ কিছুটা পুরোনো হতে পারে।
- **Flatpak:** distro-নিরপেক্ষ। প্রতিটা অ্যাপ একটা স্যান্ডবক্সে চলে, নিজের "runtime" নিয়ে আসে। প্রধান ভাণ্ডার **Flathub**। Fedora, Mint, Debian-এ জনপ্রিয়।
- **Snap:** Canonical-এর (Ubuntu-র কোম্পানি) বানানো, ধারণা Flatpak-এর মতো। Ubuntu-তে ডিফল্ট; Firefox-ও Ubuntu-তে snap হিসেবে আসে।

এদের ভেতরের কাজ, কমান্ড লাইনে `apt`, `dnf`, `flatpak`, `snap` ব্যবহার — সব [অধ্যায় ৩০](30-package-management.md)-এ বিস্তারিত।

> 💡 **টিপস:** সফটওয়্যার সেন্টারে একই অ্যাপ (যেমন VLC) দুই-তিনবার দেখা যেতে পারে। অ্যাপের পাতায় "Source" বা ড্রপডাউনে দেখা যায় কোনটা Snap, কোনটা Flatpak, কোনটা distro-র deb/rpm। একটাই ইনস্টল করো, নইলে মেনুতে দুটো আইকন পাবে।

> ⚠️ **সতর্কতা:** ইন্টারনেট থেকে এলোমেলো `.deb`/`.rpm` ফাইল বা স্ক্রিপ্ট ডাউনলোড করে ইনস্টল কোরো না। সফটওয়্যার সেন্টার বা অফিশিয়াল repository-র সফটওয়্যার স্বাক্ষরযুক্ত (signed) আর যাচাই করা।

### ৪.৬.২ সিস্টেম আপডেট

GUI-তেও আপডেট খুব সহজ:

- **Ubuntu:** "Software Updater" অ্যাপ নিজে থেকেই নোটিফিকেশন দেয়।
- **Fedora (GNOME Software):** Updates ট্যাব। Fedora সিস্টেম আপডেট ডাউনলোড করে রিস্টার্টের সময় ইনস্টল করে ("offline update"), যাতে চলন্ত প্রোগ্রামের ফাইল বদলাতে না হয়।
- **KDE Discover:** Updates অংশ, প্যানেলেও আপডেট আইকন আসে।

---

## ৪.৭ সেটিংস অ্যাপ

প্রতিটা DE-র নিজস্ব সেটিংস অ্যাপ আছে। সাধারণ যা যা পাবে:

| অংশ | কী বদলানো যায় |
|---|---|
| Network / Wi-Fi | Wi-Fi, ইথারনেট, VPN, প্রক্সি (ভেতরে NetworkManager, দেখো [অধ্যায় ৩৬](36-networking-basics.md)) |
| Bluetooth | ডিভাইস জোড়া লাগানো |
| Displays | রেজোলিউশন, স্কেলিং, একাধিক মনিটর, Night Light |
| Sound | আউটপুট/ইনপুট ডিভাইস, ভলিউম (ভেতরে PipeWire) |
| Power | স্ক্রিন বন্ধ হওয়ার সময়, সাসপেন্ড, পাওয়ার মোড |
| Appearance | লাইট/ডার্ক থিম, ওয়ালপেপার, অ্যাকসেন্ট রং |
| Keyboard | লেআউট/ইনপুট সোর্স (যেমন বাংলা), শর্টকাট |
| Mouse & Touchpad | গতি, natural scrolling, tap-to-click |
| Users | ইউজার যোগ করা, পাসওয়ার্ড বদলানো ([অধ্যায় ১৫](15-users-groups.md)) |
| Region & Language | ভাষা, তারিখ/সংখ্যার ফরম্যাট |
| Date & Time | টাইমজোন, স্বয়ংক্রিয় সময় |
| Privacy / Security | লক স্ক্রিন, লোকেশন, ফাইল হিস্ট্রি |
| Default Apps | কোন ব্রাউজার, মেইল, মিউজিক প্লেয়ার ডিফল্ট |
| About / System | হার্ডওয়্যার তথ্য, OS সংস্করণ |

টার্মিনাল থেকে সেটিংস খোলা:

```
$ gnome-control-center            # GNOME Settings
$ gnome-control-center display    # সরাসরি Displays প্যানেল
$ systemsettings                  # KDE System Settings
$ xfce4-settings-manager          # XFCE
$ cinnamon-settings               # Cinnamon
```

### ৪.৭.১ বাংলায় লেখার জন্য কী-বোর্ড যোগ করা

GNOME-এ: Settings → Keyboard → Input Sources → "+" → Bangla (Bengali) → পছন্দের লেআউট (যেমন Bangla (Probhat))। এরপর `Super+Space` চাপলে ইংরেজি আর বাংলার মধ্যে বদল হবে।
Avro-র মতো ফোনেটিক টাইপিং চাইলে **IBus** বা **Fcitx5** input method framework-এর সাথে `ibus-m17n` বা OpenBangla Keyboard ব্যবহার করা হয়। KDE-তে System Settings → Keyboard → Layouts, বা Input Method থেকে।

### ৪.৭.২ ভেতরে কীভাবে কাজ করে: GSettings ও dconf

GNOME-এর প্রায় সব সেটিংস সংরক্ষিত থাকে একটা কী-ভ্যালু ডাটাবেজে, নাম **dconf**, ফাইল `~/.config/dconf/user`। এটা binary ফাইল, হাতে এডিট করা যায় না। অ্যাপগুলো এটা পড়ে **GSettings** API দিয়ে। টার্মিনাল থেকে একই কাজ করে `gsettings` কমান্ড।

```
gsettings get <schema> <key>
gsettings set <schema> <key> <value>
gsettings reset <schema> <key>
gsettings list-keys <schema>
gsettings list-recursively [schema]
```

| সাব-কমান্ড | কাজ |
|---|---|
| `get` | একটা সেটিং-এর বর্তমান মান দেখায় |
| `set` | মান বদলায় (তৎক্ষণাৎ কার্যকর) |
| `reset` | ডিফল্টে ফিরিয়ে দেয় |
| `list-schemas` | সব schema-র নাম |
| `list-keys` | একটা schema-র সব key |
| `list-recursively` | schema-র সব key আর মান একসাথে |
| `range` | একটা key কোন কোন মান নিতে পারে |
| `monitor` | কোনো সেটিং বদলালে লাইভ দেখায় |

উদাহরণ: ডার্ক মোড চালু করা

```
$ gsettings get org.gnome.desktop.interface color-scheme
'default'
$ gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'
$ gsettings get org.gnome.desktop.interface color-scheme
'prefer-dark'
```

কোন মান বৈধ তা দেখতে:

```
$ gsettings range org.gnome.desktop.interface color-scheme
enum
'default'
'prefer-dark'
'prefer-light'
```

আরও কিছু কাজের উদাহরণ:

```
# ঘড়িতে সেকেন্ড দেখানো
$ gsettings set org.gnome.desktop.interface clock-show-seconds true

# ব্যাটারি শতাংশ দেখানো
$ gsettings set org.gnome.desktop.interface show-battery-percentage true

# টাইটেল বারে minimize, maximize, close তিনটা বাটন
$ gsettings set org.gnome.desktop.wm.preferences button-layout ':minimize,maximize,close'

# Settings অ্যাপে কোনো কিছু বদলালে কোন key বদলাল তা লাইভ দেখা
$ gsettings monitor org.gnome.desktop.interface
color-scheme: 'prefer-dark'
```

> 💡 **টিপস:** `gsettings monitor` চালু রেখে Settings অ্যাপে কোনো অপশন বদলাও। টার্মিনালে দেখবে ঠিক কোন key বদলাল। এভাবে যেকোনো GUI সেটিং-এর কমান্ড খুঁজে বের করে স্ক্রিপ্টে ব্যবহার করতে পারবে। গ্রাফিক্যালভাবে সব key ঘাঁটতে চাইলে `dconf-editor` অ্যাপ আছে।

> ⚠️ **সতর্কতা:** `dconf reset -f /` তোমার সব GNOME সেটিংস মুছে ডিফল্ট করে দেয়। না বুঝে চালিও না।

KDE Plasma-র সেটিংস থাকে `~/.config/` ফোল্ডারে সাধারণ টেক্সট ফাইলে (যেমন `kdeglobals`, `kwinrc`)। কমান্ড লাইন থেকে বদলাতে Plasma 6-এ `kwriteconfig6` / `kreadconfig6` ব্যবহার হয়। XFCE-র সেটিংস সামলায় `xfconf`, কমান্ড `xfconf-query`।

---

## ৪.৮ Workspace / Virtual Desktop

ধরো একসাথে ১৫টা উইন্ডো খোলা: ব্রাউজার, কোড এডিটর, তিনটা টার্মিনাল, মিউজিক প্লেয়ার, চ্যাট। Alt+Tab করে খুঁজতে খুঁজতে মাথা খারাপ। **Workspace** (KDE-তে **Virtual Desktop**) এই সমস্যার সমাধান: একই মনিটরে একাধিক "ভার্চুয়াল স্ক্রিন"।

> 💡 **উপমা:** তোমার পড়ার টেবিলে চারটা ড্রয়ার আছে। একটায় অঙ্কের খাতা, একটায় ইংরেজি, একটায় গল্পের বই। যখন যেটা পড়বে সেই ড্রয়ার খুলে টেবিলে রাখো। বাকিগুলো হারায়নি, শুধু চোখের সামনে নেই।

```
  Workspace 1        Workspace 2         Workspace 3        Workspace 4
 ┌───────────┐     ┌───────────┐      ┌───────────┐      ┌───────────┐
 │ ব্রাউজার    │     │ কোড এডিটর   │      │ টার্মিনাল ×৩ │      │ চ্যাট, গান   │
 │ (রিসার্চ)    │ ◄─► │           │ ◄──► │           │ ◄──► │           │
 └───────────┘     └───────────┘      └───────────┘      └───────────┘
   Super+PgUp / Super+PgDn (GNOME)   ·   Ctrl+Meta+← / → (KDE)
```

### ৪.৮.১ GNOME-এ workspace

- ডিফল্টভাবে **dynamic**: একটা workspace-এ কোনো উইন্ডো রাখলে ডান পাশে স্বয়ংক্রিয়ভাবে একটা নতুন খালি workspace তৈরি হয়; খালি হলে মুছে যায়। সবসময় শেষে একটা খালি থাকে।
- workspace-গুলো পাশাপাশি (horizontal) সাজানো।
- Overview-তে উইন্ডো টেনে অন্য workspace-এ ফেলা যায়।
- টাচপ্যাডে তিন আঙুলে বামে/ডানে সোয়াইপ করলে workspace বদলায়।
- একাধিক মনিটরে ডিফল্টভাবে workspace বদলায় শুধু প্রধান মনিটরে; বাকি মনিটর স্থির থাকে (Settings → Multitasking থেকে বদলানো যায়)।

নির্দিষ্ট সংখ্যক workspace চাইলে (Settings → Multitasking-এও আছে):

```
$ gsettings set org.gnome.mutter dynamic-workspaces false
$ gsettings set org.gnome.desktop.wm.preferences num-workspaces 4
```

### ৪.৮.২ KDE Plasma-য় virtual desktop

- System Settings → Window Management → Virtual Desktops থেকে সংখ্যা, নাম আর কয় সারিতে সাজানো হবে ঠিক করো।
- `Meta+G` (Grid view) সব desktop একসাথে দেখায়; উইন্ডো টেনে সরানো যায়।
- `Ctrl+F1` ... `Ctrl+F4` সরাসরি নির্দিষ্ট desktop-এ নিয়ে যায়।
- উইন্ডোর টাইটেল বারে রাইট-ক্লিক → "Move to Desktop" দিয়ে সরাও।
- এর উপরে **Activities** (`Meta+Tab` দিয়ে বদল), যা ৪.৪.২-এ বলেছি।

### ৪.৮.৩ Workspace ব্যবহারের ভালো অভ্যাস

1. প্রতিটা workspace-এর একটা নির্দিষ্ট কাজ রাখো (১ = ব্রাউজার, ২ = কোড, ৩ = টার্মিনাল, ৪ = যোগাযোগ)।
2. মাউসের বদলে শর্টকাট দিয়ে বদলাও। কয়েক দিনে হাতের অভ্যাস হয়ে যাবে।
3. পরে [অধ্যায় ২৫](25-jobs-tmux.md)-এ শিখবে `tmux`, যেটা টার্মিনালের ভেতরে একই রকম "workspace" দেয়, এমনকি SSH দিয়ে সার্ভারেও।

---

## ৪.৯ GNOME কী-বোর্ড শর্টকাট (সম্পূর্ণ টেবিল)

`Super` = কী-বোর্ডের Windows/লোগো কী। নিচের শর্টকাটগুলো GNOME 46–48 (Ubuntu 24.04, Fedora 40–42, Debian 13) অনুযায়ী; Debian 12-এর GNOME 43-এও প্রায় সবই এক। কোনো শর্টকাট শুধু Ubuntu-তে থাকলে তা উল্লেখ করেছি। Settings → Keyboard → View and Customize Shortcuts থেকে সব দেখা ও বদলানো যায়।

### ৪.৯.১ সিস্টেম ও নেভিগেশন

| শর্টকাট | কাজ |
|---|---|
| `Super` | Activities Overview খোলা/বন্ধ (সার্চ সহ) |
| `Super+A` | সব অ্যাপের grid (Show Apps) |
| `Super+S` | Quick Settings মেনু |
| `Super+V` বা `Super+M` | নোটিফিকেশন তালিকা ও ক্যালেন্ডার |
| `Super+N` | নোটিফিকেশনে focus |
| `Alt+F2` | কমান্ড চালানোর ছোট বক্স (Run a command) |
| `Super+L` | স্ক্রিন লক |
| `Ctrl+Alt+Delete` | Log out ডায়ালগ |
| `Super+Space` | ইনপুট সোর্স/কী-বোর্ড লেআউট বদল (যেমন English ⇄ বাংলা) |
| `Ctrl+Alt+Tab` | সিস্টেম কন্ট্রোলে (top bar, dock) কী-বোর্ড focus |
| `Ctrl+Alt+T` | টার্মিনাল খোলা (Ubuntu-তে ডিফল্ট; Fedora/Debian-এ নিজে সেট করতে হয়) |
| `Super+D` | সব উইন্ডো লুকিয়ে ডেস্কটপ দেখানো (Ubuntu-তে ডিফল্ট; upstream GNOME-এ বন্ধ) |
| `Super+E` | Home ফোল্ডার ফাইল ম্যানেজারে (Ubuntu-তে) |

### ৪.৯.২ অ্যাপ ও উইন্ডো বদলানো

| শর্টকাট | কাজ |
|---|---|
| `Alt+Tab` / `Super+Tab` | অ্যাপের মধ্যে বদল (একই অ্যাপের সব উইন্ডো একসাথে গোনা হয়) |
| `Alt+Shift+Tab` | উল্টো দিকে অ্যাপ বদল |
| `` Alt+` `` / `` Super+` `` | একই অ্যাপের উইন্ডোগুলোর মধ্যে বদল (`` ` `` হলো Tab-এর উপরের কী) |
| `Alt+Esc` | কোনো তালিকা না দেখিয়ে সরাসরি উইন্ডো বদল |
| `Super+1` ... `Super+9` | Dock/Dash-এর ১ম ... ৯ম অ্যাপ খোলা বা তাতে যাওয়া |

### ৪.৯.৩ উইন্ডো ম্যানেজমেন্ট

| শর্টকাট | কাজ |
|---|---|
| `Super+↑` | উইন্ডো maximize |
| `Super+↓` | maximize থেকে আগের আকারে / restore |
| `Super+←` / `Super+→` | স্ক্রিনের বাম/ডান অর্ধেকে বসানো (tile) |
| `Super+H` | উইন্ডো minimize (hide) |
| `Alt+F4` | উইন্ডো বন্ধ |
| `Alt+F10` | maximize টগল |
| `Alt+F7` | কী-বোর্ড দিয়ে উইন্ডো সরানো (তারপর তীর কী, শেষে Enter) |
| `Alt+F8` | কী-বোর্ড দিয়ে উইন্ডো resize |
| `Alt+Space` | উইন্ডো মেনু (Move, Resize, Always on Top...) |
| `Super+Shift+←` / `→` | উইন্ডো পাশের মনিটরে সরানো |
| `Super` চেপে মাউস দিয়ে টানা | উইন্ডোর যেকোনো জায়গা ধরে সরানো |

### ৪.৯.৪ Workspace

| শর্টকাট | কাজ |
|---|---|
| `Super+Page Up` / `Super+Page Down` | আগের/পরের workspace |
| `Ctrl+Alt+←` / `Ctrl+Alt+→` | আগের/পরের workspace (GNOME 40+) |
| `Super+Shift+Page Up` / `Page Down` | বর্তমান উইন্ডো আগের/পরের workspace-এ নিয়ে যাওয়া |
| `Ctrl+Alt+Shift+←` / `→` | একই কাজ, তীর দিয়ে |
| `Super+Home` / `Super+End` | প্রথম/শেষ workspace |
| `Super+Shift+Home` / `End` | উইন্ডো প্রথম/শেষ workspace-এ |

### ৪.৯.৫ স্ক্রিনশট ও রেকর্ডিং (GNOME 42+)

| শর্টকাট | কাজ |
|---|---|
| `Print` | স্ক্রিনশট টুল খোলে (অংশ/পুরো স্ক্রিন/উইন্ডো বাছো, রেকর্ডিং মোডও আছে) |
| `Shift+Print` | সঙ্গে সঙ্গে পুরো স্ক্রিনের স্ক্রিনশট |
| `Alt+Print` | সঙ্গে সঙ্গে বর্তমান উইন্ডোর স্ক্রিনশট |
| `Ctrl+Shift+Alt+R` | স্ক্রিন রেকর্ডিং শুরু/বন্ধ |

স্ক্রিনশট `~/Pictures/Screenshots/`-এ সেভ হয় আর ক্লিপবোর্ডেও কপি হয়; রেকর্ডিং যায় `~/Videos/Screencasts/`-এ।

### ৪.৯.৬ Accessibility

| শর্টকাট | কাজ |
|---|---|
| `Super+Alt+8` | Zoom চালু/বন্ধ |
| `Super+Alt+=` / `Super+Alt+-` | Zoom in / out |
| `Super+Alt+S` | Screen reader (Orca) চালু/বন্ধ |

### ৪.৯.৭ টাচপ্যাড gesture (GNOME, Wayland)

| Gesture | কাজ |
|---|---|
| তিন আঙুলে উপরে সোয়াইপ | Overview; আবার উপরে = App grid |
| তিন আঙুলে নিচে সোয়াইপ | ফিরে আসা |
| তিন আঙুলে বামে/ডানে | workspace বদল |
| দুই আঙুলে স্ক্রল | স্ক্রল (natural scrolling Settings-এ) |

---

## ৪.১০ KDE Plasma কী-বোর্ড শর্টকাট (সম্পূর্ণ টেবিল)

`Meta` = Windows/লোগো কী। নিচের তালিকা Plasma 6 অনুযায়ী (Fedora KDE, Kubuntu 24.10+, KDE neon)। Kubuntu 24.04-এ Plasma 5.27 আছে; পার্থক্যগুলো টেবিলে বলেছি। System Settings → Keyboard → Shortcuts থেকে সব দেখা ও বদলানো যায়।

### ৪.১০.১ সিস্টেম ও লঞ্চার

| শর্টকাট | কাজ |
|---|---|
| `Meta` | Application Launcher (Kickoff মেনু) |
| `Alt+Space` বা `Alt+F2` | KRunner (সার্চ ও কমান্ড চালানো) |
| `Meta+L` বা `Ctrl+Alt+L` | স্ক্রিন লক |
| `Ctrl+Alt+Delete` | Log out / Shut down স্ক্রিন |
| `Ctrl+Alt+T` | Konsole টার্মিনাল |
| `Meta+E` | Dolphin ফাইল ম্যানেজার |
| `Ctrl+Esc` | System Monitor (প্রসেস তালিকা) |
| `Meta+V` | ক্লিপবোর্ড হিস্ট্রি (Klipper) |
| `Meta+.` | ইমোজি নির্বাচক |
| `Meta+P` | ডিসপ্লে/প্রজেক্টর কনফিগারেশন |
| `Meta+Alt+K` | পরের কী-বোর্ড লেআউটে বদল |

> 📝 **মনে রাখো:** KDE-তে লেআউট বদলের শর্টকাট System Settings → Keyboard → Layouts-এ নিজে সেট করতে হতে পারে; distro-ভেদে ডিফল্ট আলাদা।

### ৪.১০.২ উইন্ডো বদল ও ওভারভিউ

| শর্টকাট | কাজ |
|---|---|
| `Alt+Tab` / `Alt+Shift+Tab` | উইন্ডোর মধ্যে বদল (সামনে/পিছনে) |
| `` Alt+` `` | একই অ্যাপের উইন্ডোর মধ্যে বদল |
| `Meta+W` | Overview (সব উইন্ডো + desktop; Plasma 6) |
| `Meta+G` | Grid view (সব virtual desktop; Plasma 6) |
| `Meta+1` ... `Meta+9` | টাস্ক ম্যানেজারের ১ম ... ৯ম অ্যাপে যাওয়া/খোলা |
| `Meta+D` | ডেস্কটপ উঁকি দেওয়া (Plasma 6; Plasma 5-এ `Ctrl+F12`) |
| `Meta+Tab` | Activity বদল |

### ৪.১০.৩ উইন্ডো ম্যানেজমেন্ট

| শর্টকাট | কাজ |
|---|---|
| `Meta+Page Up` | maximize |
| `Meta+Page Down` | minimize |
| `Meta+←` / `Meta+→` | বাম/ডান অর্ধেকে quick tile |
| `Meta+↑` / `Meta+↓` | উপরের/নিচের অর্ধেকে quick tile (কোণে বসাতে পাশের তীরের সাথে মেলাও) |
| `Meta+T` | Tile editor (Plasma 6, নিজের লেআউট বানানো) |
| `Alt+F4` | উইন্ডো বন্ধ |
| `Alt+F3` | উইন্ডো অপারেশন মেনু |
| `Ctrl+Alt+Esc` | উইন্ডো "kill" মোড: পয়েন্টার খুলির মতো হয়, যে উইন্ডোতে ক্লিক করবে সেটা জোর করে বন্ধ (Esc চাপলে বাতিল) |
| `Meta+Shift+←` / `→` | উইন্ডো পাশের মনিটরে সরানো |
| `Meta` + মাউস টানা | উইন্ডোর যেকোনো জায়গা ধরে সরানো (Plasma 6; Plasma 5-এ modifier ছিল `Alt`) |

### ৪.১০.৪ Virtual Desktop

| শর্টকাট | কাজ |
|---|---|
| `Ctrl+F1` ... `Ctrl+F4` | ১ ... ৪ নম্বর desktop-এ যাওয়া |
| `Ctrl+Meta+←` / `Ctrl+Meta+→` | আগের/পরের desktop |
| `Ctrl+Meta+↑` / `↓` | উপরের/নিচের সারির desktop (গ্রিডে সাজানো থাকলে) |
| `Meta+G` | সব desktop গ্রিডে দেখা |

উইন্ডোকে নির্দিষ্ট desktop-এ পাঠানোর শর্টকাট ("Window to Desktop 2" ইত্যাদি) ডিফল্টভাবে খালি; শর্টকাট সেটিংসে নিজে দিয়ে নাও।

### ৪.১০.৫ স্ক্রিনশট (Spectacle)

| শর্টকাট | কাজ |
|---|---|
| `Print` | Spectacle খোলা |
| `Meta+Print` | সক্রিয় উইন্ডোর স্ক্রিনশট |
| `Meta+Shift+Print` বা `Meta+Shift+S` | নির্বাচিত অংশের স্ক্রিনশট |
| `Shift+Print` | পুরো স্ক্রিনের স্ক্রিনশট |

### ৪.১০.৬ GNOME বনাম KDE: একই কাজ, আলাদা শর্টকাট

| কাজ | GNOME | KDE Plasma |
|---|---|---|
| অ্যাপ খোঁজা/চালানো | `Super`, টাইপ | `Meta` বা `Alt+Space` |
| সব উইন্ডো দেখা | `Super` | `Meta+W` |
| স্ক্রিন লক | `Super+L` | `Meta+L` |
| টার্মিনাল | `Ctrl+Alt+T` (Ubuntu) | `Ctrl+Alt+T` |
| ফাইল ম্যানেজার | `Super+E` (Ubuntu) | `Meta+E` |
| maximize | `Super+↑` | `Meta+Page Up` |
| অর্ধেক স্ক্রিনে tile | `Super+←/→` | `Meta+←/→` |
| পরের workspace | `Super+Page Down` | `Ctrl+Meta+→` |
| ক্লিপবোর্ড হিস্ট্রি | নেই (extension লাগে) | `Meta+V` |
| স্ক্রিনশট | `Print` | `Print` |

### ৪.১০.৭ XFCE ও Cinnamon-এর দরকারি শর্টকাট

| কাজ | XFCE (Xubuntu) | Cinnamon (Mint) |
|---|---|---|
| মেনু | `Super` (Whisker menu) | `Super` |
| টার্মিনাল | `Ctrl+Alt+T` | `Ctrl+Alt+T` |
| কমান্ড চালানো | `Alt+F2` | `Alt+F2` |
| স্ক্রিন লক | `Ctrl+Alt+L` | `Ctrl+Alt+L` |
| workspace বদল | `Ctrl+Alt+←/→` | `Ctrl+Alt+←/→` |
| উইন্ডো অন্য workspace-এ | `Ctrl+Alt+Shift+←/→` | `Ctrl+Alt+Shift+←/→` |
| সব workspace দেখা | নেই (ডিফল্টে) | `Ctrl+Alt+↑` (Expo) |
| সব উইন্ডো দেখা | নেই (ডিফল্টে) | `Ctrl+Alt+↓` (Scale) |
| ডেস্কটপ দেখানো | `Ctrl+Alt+D` | `Super+D` |
| tile | `Super+←/→` (Xubuntu-তে সংস্করণভেদে) | `Super+←/→` |

### ৪.১০.৮ সব DE-তে প্রায় একই সর্বজনীন শর্টকাট

| শর্টকাট | কাজ |
|---|---|
| `Ctrl+C` / `Ctrl+X` / `Ctrl+V` | কপি / কাট / পেস্ট (GUI অ্যাপে; টার্মিনালে আলাদা, নিচে দেখো) |
| `Ctrl+Z` / `Ctrl+Shift+Z` বা `Ctrl+Y` | Undo / Redo |
| `Ctrl+S` | সেভ |
| `Ctrl+Q` | অ্যাপ পুরো বন্ধ |
| `Ctrl+W` | ট্যাব/ডকুমেন্ট বন্ধ |
| `Ctrl+F` | খোঁজা |
| `F11` | ফুলস্ক্রিন |
| `Alt+F4` | উইন্ডো বন্ধ |
| `Alt+Tab` | উইন্ডো বদল |
| মাউসের মাঝের বাটন (চাকা) ক্লিক | নির্বাচিত লেখা পেস্ট (লিনাক্সের "PRIMARY selection", নিচে দেখো) |
| `Ctrl+Alt+F3` ... `Ctrl+Alt+F6` | টেক্সট কনসোল (TTY)-এ যাওয়া |

> 💡 **লিনাক্সের দুটো ক্লিপবোর্ড:** লিনাক্সে কোনো লেখা মাউস দিয়ে শুধু **সিলেক্ট** করলেই সেটা একটা আলাদা ক্লিপবোর্ডে (PRIMARY selection) চলে যায়। তারপর যেকোনো জায়গায় মাউসের **মাঝের বাটন** চাপলে পেস্ট হয়। `Ctrl+C`/`Ctrl+V`-এর সাধারণ ক্লিপবোর্ড (CLIPBOARD) আলাদা। দুটো একে অপরকে বদলায় না। এটা X11 যুগের ঐতিহ্য, Wayland-এও বেশির ভাগ DE রেখেছে।

> ⚠️ **সতর্কতা:** টার্মিনালের ভেতরে `Ctrl+C` মানে কপি না, চলন্ত প্রোগ্রাম থামানো! টার্মিনালে কপি-পেস্ট হলো `Ctrl+Shift+C` / `Ctrl+Shift+V`। বিস্তারিত [অধ্যায় ৬](06-terminal-shortcuts.md)-এ।

---

## ৪.১১ টার্মিনাল খোলা

এই বইয়ের বাকি অংশের বেশির ভাগ কাজ টার্মিনালে। তাই টার্মিনাল খোলার সব উপায় জেনে রাখো। টার্মিনাল আসলে কী, শেল কী, তা পরের অধ্যায়ে ([অধ্যায় ৫](05-terminal-o-shell.md)) বিস্তারিত।

| DE / Distro | ডিফল্ট টার্মিনাল অ্যাপ | কমান্ড |
|---|---|---|
| Ubuntu 24.04, Debian 12 (GNOME) | GNOME Terminal | `gnome-terminal` |
| Fedora 41+ Workstation | Ptyxis | `ptyxis` |
| কিছু GNOME distro | Console | `kgx` |
| KDE Plasma | Konsole | `konsole` |
| XFCE | Xfce Terminal | `xfce4-terminal` |
| Cinnamon (Mint) | GNOME Terminal | `gnome-terminal` |

### ৪.১১.১ টার্মিনাল খোলার উপায়

1. **শর্টকাট:** `Ctrl+Alt+T` (Ubuntu, Mint, Kubuntu, Xubuntu-তে ডিফল্ট)।
2. **সার্চ করে:** GNOME-এ `Super` চেপে "terminal" লেখো, Enter। KDE-তে `Meta` চেপে "konsole" বা "terminal"।
3. **অ্যাপ মেনু থেকে:** GNOME-এ `Super+A` → Terminal; KDE/XFCE/Cinnamon-এ মেনু → System → Terminal।
4. **ফাইল ম্যানেজার থেকে:** রাইট-ক্লিক → "Open in Terminal" (৪.৫.২ দেখো)।
5. **Run dialog থেকে:** `Alt+F2` চেপে `gnome-terminal` (বা `konsole`, `ptyxis`) লিখে Enter।
6. **ডেস্কটপে রাইট-ক্লিক:** Ubuntu-তে ডেস্কটপের খালি জায়গায় রাইট-ক্লিক → "Open in Terminal"।

যদি `Ctrl+Alt+T` কাজ না করে (যেমন Fedora বা Debian GNOME-এ), নিজে বানাও: Settings → Keyboard → View and Customize Shortcuts → Custom Shortcuts → "+" → Name: `Terminal`, Command: `ptyxis` (বা `gnome-terminal`), Shortcut: `Ctrl+Alt+T`।

> 💡 **টিপস:** টার্মিনাল Dock/প্যানেলে pin করে রাখো (আইকনে রাইট-ক্লিক → "Pin to Dash" বা "Add to Favorites")। তাহলে `Super+1`-এর মতো শর্টকাটেও খুলবে।

### ৪.১১.২ GUI ছাড়া টেক্সট কনসোল (TTY)

GUI জমে গেলে বা ক্র্যাশ করলে `Ctrl+Alt+F3` চাপো। একটা পুরো কালো টেক্সট স্ক্রিন আসবে, যার নাম **TTY** (virtual console)। এখানে ইউজারনেম আর পাসওয়ার্ড দিয়ে লগইন করে কমান্ড চালানো যায়, যেমন জমে থাকা প্রোগ্রাম বন্ধ করা ([অধ্যায় ২৪](24-process-management.md))।

```
Ubuntu 24.04.1 LTS rahim-pc tty3

rahim-pc login: rahim
Password:
rahim@rahim-pc:~$
```

GUI-তে ফিরতে: Ubuntu/Fedora-তে GDM সাধারণত `Ctrl+Alt+F1` (লগইন স্ক্রিন) আর তোমার session `Ctrl+Alt+F2`-এ থাকে। কোনটায় আছে তা distro আর display manager অনুযায়ী আলাদা; `loginctl`-এর `TTY` কলাম দেখে নিশ্চিত হতে পারো।

> 📝 **মনে রাখো:** TTY-তে পাসওয়ার্ড টাইপ করলে কিছুই দেখা যায় না, এমনকি তারকা চিহ্নও না। এটা স্বাভাবিক নিরাপত্তা ব্যবস্থা। টাইপ করে Enter চাপো।

### ৪.১১.৩ GUI চালু থাকবে কি না: systemd target

সিস্টেম বুট হয়ে GUI-তে যাবে নাকি শুধু টেক্সট মোডে থাকবে, তা ঠিক করে systemd-এর **default target**:

```
$ systemctl get-default
graphical.target
```

সার্ভারে সাধারণত `multi-user.target` থাকে (GUI ছাড়া)। বদলানো এবং target-এর বিস্তারিত [অধ্যায় ২৬](26-systemd-services.md) ও [অধ্যায় ২৭](27-boot-process.md)-এ।

---

## ৪.১২ GUI-সংক্রান্ত দরকারি কমান্ড ও সমস্যা সমাধান

| কমান্ড | কাজ |
|---|---|
| `echo $XDG_SESSION_TYPE` | wayland / x11 / tty |
| `echo $XDG_CURRENT_DESKTOP` | কোন DE চলছে |
| `loginctl` / `loginctl show-session N` | session-এর তথ্য |
| `systemctl status display-manager` | কোন display manager চলছে |
| `xdg-open FILE_OR_URL` | ডিফল্ট অ্যাপে খোলা |
| `xdg-mime query default application/pdf` | PDF-এর ডিফল্ট অ্যাপ কোনটা |
| `gsettings get/set ...` | GNOME সেটিং পড়া/বদলানো |
| `gnome-extensions list` | ইনস্টল করা GNOME extension-এর তালিকা |
| `gnome-extensions disable UUID` | কোনো extension বন্ধ করা |
| `gnome-shell --version` | GNOME সংস্করণ |
| `plasmashell --version` | Plasma সংস্করণ |
| `journalctl -b --user` | তোমার session-এর লগ ([অধ্যায় ২৮](28-logs.md)) |

```
$ xdg-mime query default application/pdf
org.gnome.Evince.desktop

$ gnome-shell --version
GNOME Shell 46.0
```

### ৪.১২.১ ভেতরে কী: `.desktop` ফাইল

অ্যাপ মেনুতে যে আইকন আর নাম দেখো, প্রতিটার পেছনে একটা ছোট টেক্সট ফাইল আছে, যার এক্সটেনশন `.desktop`। থাকে `/usr/share/applications/` (সিস্টেমজুড়ে), `~/.local/share/applications/` (শুধু তোমার), আর Flatpak অ্যাপের জন্য `/var/lib/flatpak/exports/share/applications/`।

```
$ cat /usr/share/applications/org.gnome.Terminal.desktop
[Desktop Entry]
Name=Terminal
Comment=Use the command line
Exec=gnome-terminal --window
Icon=org.gnome.Terminal
Type=Application
Categories=GNOME;GTK;System;TerminalEmulator;
...
```

নিজের স্ক্রিপ্টের জন্য মেনুতে আইকন চাইলে `~/.local/share/applications/`-এ এমন একটা ফাইল বানালেই হয়। এটা freedesktop.org-এর একটা মানদণ্ড, তাই সব DE বোঝে। একইভাবে লগইনের সময় স্বয়ংক্রিয়ভাবে কোনো অ্যাপ চালাতে `.desktop` ফাইল রাখা হয় `~/.config/autostart/`-এ।

### ৪.১২.২ সাধারণ সমস্যা ও সমাধান

| সমস্যা | সম্ভাব্য কারণ ও সমাধান |
|---|---|
| স্ক্রিন শেয়ারে কালো স্ক্রিন | Wayland-এ অ্যাপটা portal সাপোর্ট করে না। অ্যাপ আপডেট করো, `xdg-desktop-portal` চলছে কি না দেখো, শেষ উপায় Xorg session |
| একটা উইন্ডো জমে গেছে | `Alt+F4`; না হলে GNOME নিজেই "Force Quit" জিজ্ঞেস করবে; KDE-তে `Ctrl+Alt+Esc` |
| পুরো GUI জমে গেছে | `Ctrl+Alt+F3` → TTY-তে লগইন → সমস্যার প্রসেস `kill` ([অধ্যায় ২৪](24-process-management.md)) |
| GNOME extension-এর পর অদ্ভুত আচরণ | `gnome-extensions disable` দিয়ে বন্ধ করে দেখো |
| লগইনের পর আবার লগইন স্ক্রিনে ফেরত (login loop) | প্রায়ই home ফোল্ডারের ফাইলের মালিকানা বা ডিস্ক ভর্তি। TTY থেকে `df -h` আর `ls -la ~` দেখো ([অধ্যায় ১৬](16-permissions.md)) |
| বিভিন্ন মনিটরে আলাদা স্কেলিং ঠিক হচ্ছে না | Wayland session ব্যবহার করো |

> 💡 **টিপস:** পুরোনো X11 যুগে GNOME জমে গেলে `Alt+F2` → `r` দিয়ে GNOME Shell রিস্টার্ট করা যেত। Wayland-এ এটা কাজ করে না, কারণ GNOME Shell নিজেই display server; ওটা বন্ধ হলে পুরো session বন্ধ হয়ে যায়। তাই Wayland-এ log out/log in করাই উপায়।

---

## হাতে-কলমে অনুশীলন

### ল্যাব ১: তোমার GUI স্ট্যাক চেনো

1. `Ctrl+Alt+T` (বা ৪.১১.১-এর যেকোনো উপায়ে) টার্মিনাল খোলো।
2. নিচের কমান্ডগুলো চালাও আর উত্তরগুলো খাতায় লেখো:
   ```
   $ echo $XDG_SESSION_TYPE
   $ echo $XDG_CURRENT_DESKTOP
   $ systemctl status display-manager --no-pager | head -3
   $ gnome-shell --version     # অথবা KDE হলে: plasmashell --version
   ```
3. এবার নিচের টেবিলটা পূরণ করো:

   | স্তর | আমার সিস্টেমে |
   |---|---|
   | Display server / protocol | ? |
   | Desktop environment ও সংস্করণ | ? |
   | Display manager | ? |
   | Window manager / compositor (৪.৪.৬-এর টেবিল দেখে) | ? |

### ল্যাব ২: শর্টকাট দিয়ে একটা দিন

মাউস একপাশে সরিয়ে রাখো, শুধু কী-বোর্ড দিয়ে:

1. `Super` (KDE: `Meta`) চেপে "files" লিখে ফাইল ম্যানেজার খোলো।
2. `Ctrl+H` দিয়ে লুকানো ফাইল দেখো। `.bashrc` আর `.config` খুঁজে বের করো। আবার `Ctrl+H`।
3. `Ctrl+Shift+N` (Dolphin-এ `F10`) দিয়ে `linux-practice` নামে ফোল্ডার বানাও।
4. `Ctrl+L` চেপে অ্যাড্রেস বারে `/etc` লিখে Enter। এটা সিস্টেম কনফিগারেশনের ফোল্ডার ([অধ্যায় ৭](07-filesystem-hierarchy.md))।
5. `Alt+Home` দিয়ে Home-এ ফেরো।
6. `Super+←` দিয়ে ফাইল ম্যানেজার বাম অর্ধেকে রাখো, তারপর টার্মিনাল খুলে `Super+→` দিয়ে ডান অর্ধেকে (KDE-তে `Meta+←/→`)।
7. `Super+Page Down` (KDE: `Ctrl+Meta+→`) দিয়ে পরের workspace-এ যাও, সেখানে ব্রাউজার খোলো।
8. `Super+Page Up` দিয়ে ফেরো। `Super` চেপে overview-তে দেখো দুটো workspace-এ কী কী আছে।
9. `Super+L` দিয়ে স্ক্রিন লক করো, আবার আনলক করো।

### ল্যাব ৩: GUI আর টার্মিনালের যোগাযোগ

1. টার্মিনালে `gsettings monitor org.gnome.desktop.interface` চালাও।
2. Settings → Appearance-এ গিয়ে Light/Dark বদলাও। টার্মিনালে কী দেখা গেল?
3. `Ctrl+C` দিয়ে monitor থামাও।
4. এবার কমান্ড দিয়ে ডার্ক মোড চালু করো:
   ```
   $ gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'
   ```
5. আগের অবস্থায় ফেরাতে:
   ```
   $ gsettings reset org.gnome.desktop.interface color-scheme
   ```
6. `mkdir -p ~/linux-practice && cd ~/linux-practice && xdg-open .` চালাও। ফাইল ম্যানেজারে ফোল্ডারটা খুলে গেল কি?

(KDE ব্যবহারকারী হলে ধাপ ১–৫ বাদ দিয়ে System Settings → Colors & Themes থেকে থিম বদলাও, আর `cat ~/.config/kdeglobals | head` দিয়ে দেখো সেটিং কোথায় লেখা হলো।)

### ল্যাব ৪: TTY থেকে ঘুরে আসা

1. সব কাজ সেভ করো।
2. `Ctrl+Alt+F3` চাপো। ইউজারনেম আর পাসওয়ার্ড দিয়ে লগইন করো।
3. `loginctl` চালাও। এখন দুটো session দেখা যাওয়ার কথা: একটা GUI-র, একটা এই TTY-র।
4. `exit` লিখে লগআউট করো।
5. `Ctrl+Alt+F2` (না হলে `F1`, `F7`) চেপে GUI-তে ফেরো।

### ল্যাব ৫: মেনুতে নিজের লঞ্চার (ঐচ্ছিক)

1. `~/.local/share/applications/hello-terminal.desktop` নামে একটা ফাইল বানাও (এডিটর হিসেবে আপাতত GUI টেক্সট এডিটর বা [অধ্যায় ১৩](13-nano-editor.md)-এর `nano` ব্যবহার করো):
   ```
   [Desktop Entry]
   Type=Application
   Name=Hello Terminal
   Comment=আমার প্রথম লঞ্চার
   Exec=gnome-terminal
   Icon=utilities-terminal
   Terminal=false
   Categories=Utility;
   ```
   (KDE-তে `Exec=konsole`, Fedora 41+-এ `Exec=ptyxis` লেখো।)
2. অ্যাপ মেনু খুলে "Hello Terminal" খোঁজো। পাওয়া গেলে ক্লিক করো।
3. কাজ শেষে ফাইলটা মুছে ফেলো।

---

## সারসংক্ষেপ

**মূল ধারণা:**

| পরিভাষা | এক লাইনে | উদাহরণ |
|---|---|---|
| Display server | অ্যাপ, ইনপুট আর স্ক্রিনের মধ্যস্থতাকারী | Xorg, Wayland compositor |
| Compositor | সব উইন্ডোর ছবি জুড়ে চূড়ান্ত স্ক্রিন বানায় | Mutter, KWin, picom |
| Window manager | উইন্ডোর অবস্থান, আকার, focus, decoration | Mutter, KWin, Xfwm4, i3 |
| Display manager | গ্রাফিক্যাল লগইন স্ক্রিন, session চালু করে | GDM, SDDM, LightDM |
| Desktop environment | পুরো প্যাকেজ: WM + প্যানেল + অ্যাপ + সেটিংস | GNOME, KDE Plasma, XFCE, Cinnamon |
| XWayland | Wayland-এ পুরোনো X11 অ্যাপ চালানোর সেতু | |

**দ্রুত রেফারেন্স কমান্ড:**

| কমান্ড | কাজ |
|---|---|
| `echo $XDG_SESSION_TYPE` | Wayland না X11 |
| `echo $XDG_CURRENT_DESKTOP` | কোন DE |
| `loginctl` | session তালিকা |
| `systemctl status display-manager` | display manager |
| `xdg-open .` | বর্তমান ফোল্ডার ফাইল ম্যানেজারে |
| `gsettings get/set/reset/monitor` | GNOME সেটিংস |
| `gnome-control-center` / `systemsettings` | সেটিংস অ্যাপ |
| `gnome-extensions list` | GNOME extension |

**সবচেয়ে দরকারি শর্টকাট:**

| কাজ | GNOME | KDE Plasma |
|---|---|---|
| লঞ্চার/সার্চ | `Super` | `Meta`, `Alt+Space` |
| সব উইন্ডো | `Super` | `Meta+W` |
| টার্মিনাল | `Ctrl+Alt+T` (Ubuntu) | `Ctrl+Alt+T` |
| অ্যাপ বদল | `Alt+Tab` | `Alt+Tab` |
| অর্ধেক স্ক্রিন | `Super+←/→` | `Meta+←/→` |
| maximize | `Super+↑` | `Meta+Page Up` |
| workspace বদল | `Super+Page Up/Down` | `Ctrl+Meta+←/→` |
| উইন্ডো অন্য workspace-এ | `Super+Shift+Page Up/Down` | (নিজে সেট করতে হয়) |
| লক | `Super+L` | `Meta+L` |
| স্ক্রিনশট | `Print` | `Print` |
| লুকানো ফাইল (ফাইল ম্যানেজার) | `Ctrl+H` | `Alt+.` / `Ctrl+H` |
| TTY | `Ctrl+Alt+F3` | `Ctrl+Alt+F3` |

---

## নিজেকে যাচাই করো

1. Window manager আর desktop environment-এর মধ্যে পার্থক্য কী? একটা করে উদাহরণ দাও।
2. Display manager কী কাজ করে? Ubuntu আর Kubuntu-তে ডিফল্ট display manager কোনটা?
3. X11-এর সবচেয়ে বড় নিরাপত্তা সমস্যা কী, আর Wayland কীভাবে তা সমাধান করে?
4. তুমি Wayland-এ আছো না X11-এ, কোন কমান্ড দিয়ে জানবে?
5. XWayland কী এবং কেন দরকার?
6. ফাইল ম্যানেজারে `Delete` আর `Shift+Delete`-এর পার্থক্য কী? আর টার্মিনালের `rm` কোনটার মতো আচরণ করে?
7. GNOME-এ টার্মিনাল থেকে ডার্ক মোড চালু করার কমান্ড কী? কোনো GUI সেটিং-এর key খুঁজে বের করার সহজ উপায় কী?
8. GNOME-এ বর্তমান উইন্ডোটাকে পরের workspace-এ পাঠানোর শর্টকাট কী? KDE-তে সব virtual desktop গ্রিডে দেখার শর্টকাট কী?
9. পুরো GUI জমে গেলে কী করবে?
10. অ্যাপ মেনুর আইকনগুলোর পেছনে কোন ধরনের ফাইল থাকে, আর সেগুলো কোথায় থাকে?

<details><summary>উত্তর দেখো</summary>

1. Window manager শুধু উইন্ডোর অবস্থান, আকার, focus আর decoration সামলায় (যেমন i3, Openbox, Xfwm4)। Desktop environment একটা পুরো প্যাকেজ, যার ভেতরে একটা window manager ছাড়াও প্যানেল, মেনু, নোটিফিকেশন, ফাইল ম্যানেজার, সেটিংস আর নিজস্ব অ্যাপ থাকে (যেমন GNOME, KDE Plasma)।
2. Display manager গ্রাফিক্যাল লগইন স্ক্রিন দেখায়, পাসওয়ার্ড যাচাই করায় (PAM দিয়ে), কোন session চলবে তা বাছতে দেয় এবং সেটা চালু করে। Ubuntu-তে GDM (প্যাকেজ `gdm3`), Kubuntu-তে SDDM।
3. X11-এ যেকোনো অ্যাপ অন্য অ্যাপের কী-বোর্ড ইনপুট পড়তে আর স্ক্রিন দেখতে পারে, তাই keylogging সহজ। Wayland-এ প্রতিটা অ্যাপ শুধু নিজের উইন্ডোর ইনপুট পায়; স্ক্রিনশট/স্ক্রিন শেয়ারের জন্য xdg-desktop-portal আর PipeWire-এর মাধ্যমে ইউজারের অনুমতি লাগে।
4. `echo $XDG_SESSION_TYPE` — `wayland` বা `x11` দেখাবে। বিকল্প: `loginctl show-session <নম্বর> -p Type`।
5. XWayland হলো Wayland session-এর ভেতরে চলা একটা X server, যাতে যেসব পুরোনো অ্যাপ শুধু X11 বোঝে সেগুলোও চলতে পারে।
6. `Delete` ফাইলকে Trash-এ (`~/.local/share/Trash/`) পাঠায়, পরে ফেরত আনা যায়। `Shift+Delete` সরাসরি স্থায়ীভাবে মুছে ফেলে। টার্মিনালের `rm` `Shift+Delete`-এর মতো, Trash-এ যায় না।
7. `gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'`। key খোঁজার উপায়: `gsettings monitor <schema>` (বা `dconf watch /`) চালু রেখে Settings অ্যাপে অপশন বদলানো; অথবা `dconf-editor` দিয়ে ঘাঁটা।
8. GNOME: `Super+Shift+Page Down` (বা `Ctrl+Alt+Shift+→`)। KDE Plasma 6: `Meta+G`।
9. `Ctrl+Alt+F3` চেপে TTY-তে যাও, লগইন করো, জমে থাকা প্রোগ্রাম খুঁজে `kill` করো (অধ্যায় ২৪); দরকার হলে সেখান থেকে রিবুট। KDE-তে একটা উইন্ডো জমলে `Ctrl+Alt+Esc` দিয়ে সেটা বন্ধ করা যায়।
10. `.desktop` ফাইল (freedesktop.org মানদণ্ড)। থাকে `/usr/share/applications/` (সিস্টেমজুড়ে), `~/.local/share/applications/` (ব্যক্তিগত) এবং Flatpak-এর জন্য `/var/lib/flatpak/exports/share/applications/`।

</details>

---

[← আগের অধ্যায়](03-distro-o-installation.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](05-terminal-o-shell.md)

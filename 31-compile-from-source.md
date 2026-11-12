# অধ্যায় ৩১: সোর্স কোড থেকে সফটওয়্যার বিল্ড করা

> আগের অধ্যায়ে ([অধ্যায় ৩০](30-package-management.md)) আমরা `apt`/`dnf` দিয়ে "রান্না করা খাবার" কিনে এনেছি। এই অধ্যায়ে আমরা নিজেরাই রান্না করব: কাঁচা উপকরণ (সোর্স কোড) থেকে চলার মতো প্রোগ্রাম বানাব। আর রান্নাঘরের ভেতরে আসলে কী কী হয়, সেটাও দেখব।

## এই অধ্যায়ে যা শিখবে

- সোর্স কোড, binary, compile, link আসলে কী জিনিস
- compile-এর চার ধাপ: preprocess → compile → assemble → link (ডায়াগ্রামসহ)
- `gcc` দিয়ে প্রথম C প্রোগ্রাম (hello world) বানানো, প্রতিটা ধাপ আলাদা করে দেখা
- static library (`.a`) আর shared library (`.so`) এর পার্থক্য, আর দুটোই নিজে বানানো
- প্রোগ্রাম চালু হলে library কে খুঁজে দেয়: `ld.so`, `ldd`, `ldconfig`, `LD_LIBRARY_PATH`
- বিল্ড টুল ইনস্টল: `build-essential` (Debian/Ubuntu), `Development Tools` (Fedora/RHEL)
- tarball নামিয়ে ক্লাসিক `./configure && make && sudo make install`
- `Makefile` লেখার বেসিক
- `cmake` পরিচিতি
- `/usr/local` কেন ব্যবহার করা হয়
- `checkinstall`, `make uninstall`, `DESTDIR`
- কেন প্যাকেজ ম্যানেজারই প্রথম পছন্দ হওয়া উচিত

---

## ৩১.১ সোর্স কোড আর binary: আসল কথা

### সোর্স কোড কী?

**Source code** মানে মানুষের পড়ার মতো করে লেখা প্রোগ্রাম, যেমন C, C++, Rust বা Go ভাষায় লেখা টেক্সট ফাইল। নিচেরটা একটা C সোর্স কোড:

```c
#include <stdio.h>

int main(void) {
    printf("Hello, World!\n");
    return 0;
}
```

এটা পড়ে তুমি বুঝতে পারছ এটা কী করবে। কিন্তু CPU কিছুই বোঝে না। CPU বোঝে শুধু **machine code**, মানে `0` আর `1` দিয়ে লেখা নির্দেশনা।

### Binary কী?

**Binary** (বা executable, মানে চালানোর যোগ্য ফাইল) হলো সেই machine code-এর ফাইল। `/usr/bin/ls`, `/usr/bin/bash` সবই binary। লিনাক্সে এগুলো **ELF** (Executable and Linkable Format) নামের একটা নির্দিষ্ট ফরম্যাটে থাকে।

```console
$ file /usr/bin/ls
/usr/bin/ls: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=..., for GNU/Linux 3.2.0, stripped
```

এই লাইনটা পরে আবার কাজে লাগবে। "dynamically linked" আর "interpreter /lib64/ld-linux-x86-64.so.2" অংশ দুটো খেয়াল রাখো।

### Compile মানে কী?

**Compile** করা মানে সোর্স কোডকে machine code-এ অনুবাদ করা। যে প্রোগ্রাম অনুবাদ করে তাকে বলে **compiler**। লিনাক্সে সবচেয়ে বেশি চলে **GCC** (GNU Compiler Collection), আর তার পাশাপাশি **Clang**।

> 💡 **উপমা:** সোর্স কোড হলো বাংলায় লেখা রান্নার রেসিপি। CPU হলো এমন এক রাঁধুনি যে শুধু জাপানি বোঝে। compiler হলো অনুবাদক, যে পুরো রেসিপি জাপানিতে অনুবাদ করে দেয়। একবার অনুবাদ হয়ে গেলে রাঁধুনি যতবার খুশি সেটা দেখে রান্না করতে পারে, প্রতিবার অনুবাদক লাগে না।

### কেন সোর্স থেকে বিল্ড করতে হয়?

সাধারণত দরকার পড়ে না, প্যাকেজ ম্যানেজারই যথেষ্ট। তবু কিছু ক্ষেত্রে লাগে:

| পরিস্থিতি | উদাহরণ |
|---|---|
| distro-র রিপোজিটরিতে সফটওয়্যারটা নেই | ছোট কোনো GitHub প্রজেক্ট |
| রিপোজিটরির ভার্সন অনেক পুরনো | Debian stable-এ নতুন ফিচার দরকার |
| বিশেষ compile অপশন লাগবে | কোনো ফিচার চালু/বন্ধ করে বিল্ড, যেমন nginx-এ নির্দিষ্ট module |
| নিজের লেখা কোড | তুমি C/C++ ডেভেলপার |
| শেখা ও ডিবাগ করা | সফটওয়্যারটা ভেতরে কীভাবে কাজ করে বোঝা |
| প্যাচ লাগানো | একটা bug fix এখনো রিলিজ হয়নি |

---

## ৩১.২ Compile-এর চার ধাপ: ভেতরে কী হয়

আমরা মুখে বলি "compile করলাম", কিন্তু আসলে `gcc` একটা নয়, চারটা আলাদা ধাপ চালায়:

```
  hello.c   (তোমার লেখা সোর্স কোড)
     │
     │  ১. Preprocess  (cpp)      gcc -E
     │     #include ফাইল ঢোকানো, #define বদলানো, comment মুছে ফেলা
     ▼
  hello.i   (বিশাল, বিস্তারিত C কোড)
     │
     │  ২. Compile     (cc1)      gcc -S
     │     C কোড → assembly ভাষা (CPU-নির্দিষ্ট, মানুষ পড়তে পারে)
     ▼
  hello.s   (assembly)
     │
     │  ৩. Assemble    (as)       gcc -c
     │     assembly → machine code, কিন্তু অসম্পূর্ণ
     ▼
  hello.o   (object file: printf কোথায় আছে এখনো জানে না)
     │
     │  ৪. Link        (ld)       gcc
     │     অনেক .o + library (libc) জোড়া লাগিয়ে চূড়ান্ত প্রোগ্রাম
     ▼
  hello     (ELF executable, চালানোর যোগ্য)
```

### ১. Preprocessing (প্রি-প্রসেসিং)

`#` দিয়ে শুরু হওয়া লাইনগুলো (**directive**) এই ধাপে প্রসেস হয়:

- `#include <stdio.h>`: `/usr/include/stdio.h` ফাইলের পুরো লেখাটা ওই জায়গায় কপি করে বসিয়ে দেয়
- `#define PI 3.14`: কোডে যেখানে `PI` আছে সেখানে `3.14` বসায়
- `#ifdef ... #endif`: শর্ত অনুযায়ী কোডের অংশ রাখে বা বাদ দেয়
- comment মুছে ফেলে

### ২. Compilation (আসল অনুবাদ)

C কোড বিশ্লেষণ করে (syntax check, type check, optimization) তারপর **assembly language**-এ রূপান্তর করে। এখানেই তুমি syntax error-এর বার্তা পাও। `-O2` এর মতো optimization অপশন এই ধাপে কাজ করে।

### ৩. Assembly (অ্যাসেম্বলি)

**Assembler** (`as`) assembly-কে machine code-এ রূপান্তর করে **object file** (`.o`) বানায়। এই ফাইলে তোমার কোডের machine code আছে, কিন্তু `printf` এর মতো বাইরের ফাংশনগুলো কোথায় আছে তা নেই। ওগুলো "unresolved symbol" হিসেবে চিহ্নিত থাকে।

### ৪. Linking (লিংকিং)

**Linker** (`ld`) সব object file আর library জোড়া লাগায়, unresolved symbol গুলোর ঠিকানা মিলিয়ে দেয় (`printf` → `libc`-এর ভেতরে), তারপর একটা পূর্ণ executable বানায়।

> 📝 **মনে রাখো:** "undefined reference to `xyz`" error আসে **link** ধাপে, মানে কোড ঠিক আছে কিন্তু ফাংশনটা কোন library-তে আছে linker সেটা পাচ্ছে না। "expected ';' before ..." এর মতো error আসে **compile** ধাপে। কোন ধাপের error সেটা চিনতে পারলে ঠিক করা অনেক সহজ হয়।

---

## ৩১.৩ বিল্ড টুল ইনস্টল করা

কোনো কিছু compile করার আগে compiler, linker, `make` ইত্যাদি লাগবে। প্রতিটা distro একটা "মেটা-প্যাকেজ" বা "গ্রুপ" দেয়, যেটা একবারে সব নামিয়ে দেয়।

### Debian / Ubuntu: `build-essential`

```console
$ sudo apt update
$ sudo apt install build-essential
```

`build-essential` নিজে কিছু না, এটা অন্য প্যাকেজের একটা তালিকা: `gcc`, `g++`, `make`, `libc6-dev` (C library-র header), `dpkg-dev`।

### Fedora / RHEL 9 / Rocky / AlmaLinux: `Development Tools` গ্রুপ

```console
# Fedora 41 বা নতুন (dnf5)
$ sudo dnf group install development-tools

# RHEL 9 / Rocky 9 / AlmaLinux 9 এবং পুরনো Fedora (dnf4)
$ sudo dnf groupinstall "Development Tools"
```

> 💡 **টিপস:** গ্রুপের নাম/ID দেখতে `dnf group list` চালাও। RHEL-এ `dnf groupinstall` আর `dnf group install` দুটোই কাজ করে।

### আরও যা প্রায়ই লাগে

| টুল | কাজ | Debian/Ubuntu প্যাকেজ | Fedora/RHEL প্যাকেজ |
|---|---|---|---|
| `cmake` | আধুনিক বিল্ড সিস্টেম | `cmake` | `cmake` |
| `pkg-config` | library-র compile/link ফ্ল্যাগ বলে দেয় | `pkg-config` (বা `pkgconf`) | `pkgconf-pkg-config` |
| `autoconf`, `automake`, `libtool` | `configure` স্ক্রিপ্ট তৈরি করে | একই নাম | একই নাম |
| `git` | সোর্স ক্লোন করা | `git` | `git` |
| `ninja` | দ্রুত বিল্ড (cmake/meson-এর সাথে) | `ninja-build` | `ninja-build` |
| `meson` | আরেকটা আধুনিক বিল্ড সিস্টেম | `meson` | `meson` |

যাচাই করো:

```console
$ gcc --version
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
Copyright (C) 2023 Free Software Foundation, Inc.
...
$ make --version
GNU Make 4.3
...
```

(ভার্সন নম্বর distro অনুযায়ী আলাদা হবে, যেমন Debian 12-এ gcc 12, RHEL 9-এ gcc 11।)

---

## ৩১.৪ `gcc` দিয়ে Hello World

### প্রথম প্রোগ্রাম

একটা ফোল্ডার বানাও আর ফাইল লেখো ([অধ্যায় ১৩](13-nano-editor.md)-এর `nano` বা [অধ্যায় ১৪](14-vim.md)-এর `vim` দিয়ে):

```console
$ mkdir -p ~/build-lab/hello && cd ~/build-lab/hello
$ nano hello.c
```

```c
#include <stdio.h>

int main(void) {
    printf("Hello, World!\n");
    return 0;
}
```

compile করো আর চালাও:

```console
$ gcc hello.c -o hello
$ ./hello
Hello, World!
```

> 📝 **মনে রাখো:** `./hello` লিখতে হয় কারণ বর্তমান ডিরেক্টরি (`.`) `PATH`-এ থাকে না (নিরাপত্তার কারণে)। শুধু `hello` লিখলে shell সেটা `PATH`-এর ডিরেক্টরিগুলোতে খুঁজবে। বিস্তারিত [অধ্যায় ৪৪](44-environment-bashrc.md)-এ।

`-o` না দিলে আউটপুট ফাইলের নাম হয় `a.out` (ঐতিহাসিক নাম)।

### `gcc` সিনট্যাক্স

```
gcc [অপশন] সোর্স-ফাইল... [-o আউটপুট]
```

### গুরুত্বপূর্ণ `gcc` অপশন

| অপশন | অর্থ |
|---|---|
| `-o FILE` | আউটপুট ফাইলের নাম |
| `-E` | শুধু preprocess করে থামো (stdout-এ লেখে) |
| `-S` | compile পর্যন্ত, `.s` (assembly) বানাও |
| `-c` | assemble পর্যন্ত, `.o` বানাও, link কোরো না |
| `-Wall` | সাধারণ সব warning দেখাও (সবসময় ব্যবহার করো) |
| `-Wextra` | আরও কিছু বাড়তি warning |
| `-Werror` | warning-কেও error ধরো |
| `-g` | debug তথ্য রাখো (`gdb`-এর জন্য) |
| `-O0`, `-O1`, `-O2`, `-O3`, `-Os` | optimization লেভেল (`-O0` = কোনো optimization নেই, ডিফল্ট; `-Os` = সাইজ ছোট) |
| `-I DIR` | header (`.h`) খোঁজার জন্য বাড়তি ডিরেক্টরি |
| `-L DIR` | library খোঁজার জন্য বাড়তি ডিরেক্টরি (link সময়) |
| `-lNAME` | `libNAME.so` বা `libNAME.a` এর সাথে link করো (যেমন `-lm` = `libm`) |
| `-D NAME=VAL` | `#define NAME VAL` এর মতো কাজ |
| `-std=c17` | C স্ট্যান্ডার্ড নির্বাচন (`c99`, `c11`, `c17`, `gnu17` ...) |
| `-fPIC` | position-independent code (shared library-র জন্য) |
| `-shared` | shared library (`.so`) বানাও |
| `-static` | পুরো প্রোগ্রাম static link করো |
| `-v` | প্রতিটা ধাপে কী চালাচ্ছে দেখাও |
| `-save-temps` | `.i`, `.s`, `.o` মধ্যবর্তী ফাইলগুলো রেখে দাও |

C++ এর জন্য একই অপশন দিয়ে `g++` ব্যবহার করো।

### চার ধাপ নিজের চোখে দেখো

```console
$ gcc -E hello.c -o hello.i      # ১. preprocess
$ wc -l hello.c hello.i
   6 hello.c
 795 hello.i
 801 total
```

মাত্র ৬ লাইন থেকে প্রায় ৮০০ লাইন! (সংখ্যাটা distro আর glibc ভার্সন অনুযায়ী আলাদা হবে।) পুরোটাই `stdio.h` আর তার ভেতরের অন্য header থেকে এসেছে।

```console
$ gcc -S hello.i -o hello.s      # ২. compile → assembly
$ cat hello.s
        .file   "hello.c"
        .text
        .section        .rodata
.LC0:
        .string "Hello, World!"
        .text
        .globl  main
        .type   main, @function
main:
        ...
        call    puts@PLT
        ...
```

> 💡 **মজার তথ্য:** তুমি `printf` লিখেছ, কিন্তু assembly-তে `puts` এসেছে। কারণ শুধু একটা স্থির string আর শেষে `\n` থাকলে GCC নিজেই বুঝে নেয় যে `puts` দিয়েই একই কাজ হবে আর সেটা দ্রুত। compiler এরকম অনেক বুদ্ধি খাটায়।

```console
$ gcc -c hello.s -o hello.o      # ৩. assemble → object file
$ file hello.o
hello.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped

$ nm hello.o                     # object-এর symbol তালিকা
0000000000000000 T main
                 U puts
```

`nm` দেখাচ্ছে: `main` এখানে **T** (text, মানে কোড এই ফাইলেই আছে), আর `puts` **U** (undefined, অন্য কোথাও আছে)। linker এই `U` গুলো পূরণ করবে।

```console
$ gcc hello.o -o hello           # ৪. link
$ ./hello
Hello, World!
```

এক কমান্ডে সব মধ্যবর্তী ফাইল রাখতে চাইলে: `gcc -save-temps hello.c -o hello`।

### একাধিক ফাইলের প্রোগ্রাম

বাস্তবে প্রোগ্রাম অনেক ফাইলে ভাগ করা থাকে:

```c
/* mathutil.h */
#ifndef MATHUTIL_H
#define MATHUTIL_H
int add(int a, int b);
int square(int x);
#endif
```

```c
/* mathutil.c */
#include "mathutil.h"
int add(int a, int b) { return a + b; }
int square(int x)     { return x * x; }
```

```c
/* main.c */
#include <stdio.h>
#include "mathutil.h"

int main(void) {
    printf("2 + 3 = %d\n", add(2, 3));
    printf("4^2  = %d\n", square(4));
    return 0;
}
```

```console
$ gcc -Wall -c mathutil.c        # mathutil.o
$ gcc -Wall -c main.c            # main.o
$ gcc main.o mathutil.o -o calc  # link
$ ./calc
2 + 3 = 5
4^2  = 16
```

আলাদা করে `.o` বানানোর সুবিধা: শুধু `main.c` বদলালে শুধু সেটাই আবার compile করতে হয়, `mathutil.o` আগেরটাই চলে। বড় প্রজেক্টে (যেমন Linux kernel, যার হাজার হাজার ফাইল) এটা ঘণ্টার পর ঘণ্টা সময় বাঁচায়। এই "কোনটা আবার বিল্ড করতে হবে" হিসাবটাই `make` করে (৩১.৮-এ দেখব)।

> `<stdio.h>` (কোণ বন্ধনী) মানে সিস্টেম header, `/usr/include` এর মতো ডিরেক্টরিতে খোঁজে। `"mathutil.h"` (উদ্ধৃতি চিহ্ন) মানে আগে বর্তমান ডিরেক্টরিতে খোঁজে।

---

## ৩১.৫ Library: static বনাম shared

### Library কী?

**Library** হলো আগে থেকে compile করা কিছু ফাংশনের সংগ্রহ, যেটা অনেক প্রোগ্রাম ব্যবহার করতে পারে। যেমন `printf`, `malloc`, `strlen` সব আছে **glibc** (GNU C Library)-তে, যার ফাইলের নাম `libc.so.6`। প্রায় প্রতিটা লিনাক্স প্রোগ্রাম এটা ব্যবহার করে।

লিনাক্সে library দুই রকম:

| বৈশিষ্ট্য | Static library | Shared library |
|---|---|---|
| এক্সটেনশন | `.a` (archive) | `.so` (shared object) |
| উইন্ডোজে সমতুল্য | `.lib` | `.dll` |
| কখন জোড়া লাগে | **link time**-এ, প্রোগ্রামের ভেতরে কপি হয়ে যায় | **run time**-এ, প্রোগ্রাম চালুর সময় লোড হয় |
| binary সাইজ | বড় | ছোট |
| RAM ব্যবহার | প্রতিটা প্রোগ্রাম নিজের কপি রাখে | এক কপি RAM-এ, অনেক প্রোগ্রাম ভাগাভাগি করে |
| library আপডেট (যেমন security fix) | সব প্রোগ্রাম আবার link করতে হয় | শুধু `.so` বদলালেই সবাই নতুনটা পায় |
| নির্ভরতা | binary একা চলে | `.so` না থাকলে প্রোগ্রাম চালুই হয় না |
| উদাহরণ | `libc.a` | `libc.so.6`, `libssl.so.3` |

> 💡 **উপমা:** **Static library** হলো একটা বইয়ের পাতা ফটোকপি করে নিজের খাতায় আঠা দিয়ে লাগিয়ে নেওয়া। খাতা ভারী হয়, কিন্তু লাইব্রেরি বন্ধ থাকলেও পড়তে পারবে। বইয়ে ভুল সংশোধন হলে তোমার কপিতে সেটা আসবে না। **Shared library** হলো খাতায় শুধু লিখে রাখা "পাবলিক লাইব্রেরির ৩ নম্বর তাকের বই, পৃষ্ঠা ৪২ দেখো"। খাতা হালকা, সবাই একই বই পড়ে, বই আপডেট হলে সবাই নতুনটা পায়। কিন্তু লাইব্রেরি থেকে বই সরিয়ে ফেললে তুমি আর পড়তে পারবে না।

এজন্যই লিনাক্স distro-গুলো প্রায় সবকিছু shared library দিয়ে বিল্ড করে: OpenSSL-এ একটা security bug ঠিক হলে একটা প্যাকেজ আপডেট করলেই শত শত প্রোগ্রাম নিরাপদ হয়ে যায়।

### Static library বানানো

```console
$ gcc -c mathutil.c -o mathutil.o
$ ar rcs libmathutil.a mathutil.o
$ ar t libmathutil.a              # ভেতরে কী আছে
mathutil.o
```

`ar` হলো archiver, অনেক `.o` কে একটা `.a` ফাইলে বাঁধে। `r` = যোগ/প্রতিস্থাপন, `c` = দরকার হলে তৈরি, `s` = symbol index লেখা।

```console
$ gcc main.c -L. -lmathutil -o calc_static
$ ./calc_static
2 + 3 = 5
4^2  = 16
```

`-L.` মানে "বর্তমান ডিরেক্টরিতেও library খোঁজো", `-lmathutil` মানে `libmathutil` (সামনের `lib` আর পেছনের `.a`/`.so` linker নিজেই যোগ করে)।

> ⚠️ **সতর্কতা:** `-l` অপশন কমান্ডের **শেষের দিকে**, সোর্স/অবজেক্ট ফাইলের **পরে** দাও। GNU linker বাম থেকে ডানে পড়ে; library আগে দিলে তখনো কোনো unresolved symbol না থাকায় সে library টা থেকে কিছু নেয় না, আর পরে "undefined reference" error দেয়।

### Shared library বানানো

```console
$ gcc -fPIC -c mathutil.c -o mathutil.o
$ gcc -shared -Wl,-soname,libmathutil.so.1 -o libmathutil.so.1.0.0 mathutil.o
$ ln -s libmathutil.so.1.0.0 libmathutil.so.1
$ ln -s libmathutil.so.1 libmathutil.so
$ ls -l libmathutil.so*
lrwxrwxrwx 1 rahim rahim    16 ... libmathutil.so -> libmathutil.so.1
lrwxrwxrwx 1 rahim rahim    20 ... libmathutil.so.1 -> libmathutil.so.1.0.0
-rwxrwxr-x 1 rahim rahim 15560 ... libmathutil.so.1.0.0
```

এখানে কয়েকটা নতুন জিনিস:

- **`-fPIC`** (Position Independent Code): কোডটা মেমোরির যেকোনো ঠিকানায় লোড হলেও চলবে। shared library বিভিন্ন প্রোগ্রামে বিভিন্ন ঠিকানায় বসে, তাই এটা দরকার।
- **`-Wl,...`**: `gcc`-কে বলে "এই অপশনগুলো linker (`ld`) কে পাঠাও"।
- **soname**: library-র "অফিশিয়াল নাম", যেটা প্রোগ্রামের ভেতরে লেখা থাকে।
- symlink ([অধ্যায় ১১](11-links-inode.md)) দিয়ে তিন স্তরের নাম:

```
  libmathutil.so            ← "linker name": শুধু বিল্ডের সময় -lmathutil এটা খোঁজে
        │ (symlink)
        ▼
  libmathutil.so.1          ← "soname": চালানোর সময় প্রোগ্রাম এটা চায়
        │ (symlink)              (major version 1 = ABI সামঞ্জস্য)
        ▼
  libmathutil.so.1.0.0      ← "real name": আসল ফাইল (major.minor.patch)
```

major version (`.1`) বদলায় শুধু যখন library-তে এমন পরিবর্তন হয় যেটা পুরনো প্রোগ্রাম ভেঙে দেবে (**ABI**, Application Binary Interface, পরিবর্তন)। তাই `libssl.so.3` আর `libssl.so.1.1` একসাথে সিস্টেমে থাকতে পারে, আর পুরনো প্রোগ্রাম পুরনোটা, নতুন প্রোগ্রাম নতুনটা ব্যবহার করে।

এখন link করো:

```console
$ gcc main.c -L. -lmathutil -o calc_shared
$ ./calc_shared
./calc_shared: error while loading shared libraries: libmathutil.so.1: cannot open shared object file: No such file or directory
```

বিল্ড হয়েছে, কিন্তু চলছে না! কেন? কারণ `-L.` শুধু **বিল্ডের সময়** linker-কে বলেছিল কোথায় খুঁজতে হবে। চালানোর সময় library খোঁজে আরেকজন: **dynamic linker** `ld.so`। আর সে বর্তমান ডিরেক্টরিতে খোঁজে না। চলো তাকে চিনি।

> 📝 **মনে রাখো:** একই নামের `.so` আর `.a` দুটোই থাকলে `-l` ডিফল্টভাবে `.so` (shared) বেছে নেয়।

---

## ৩১.৬ Dynamic linker `ld.so`: প্রোগ্রাম চালুর সময় কী হয়

### Under the hood

তুমি যখন `./calc_shared` চালাও:

```
  তুমি: ./calc_shared
     │
     ▼
  ┌─────────────── Kernel (execve system call) ───────────────┐
  │ ১. ফাইলটা ELF কিনা দেখে                                     │
  │ ২. ELF header-এ "interpreter" লেখা পায়:                     │
  │       /lib64/ld-linux-x86-64.so.2                          │
  │ ৩. প্রোগ্রাম + এই interpreter মেমোরিতে লোড করে               │
  │ ৪. নিয়ন্ত্রণ দেয় interpreter (ld.so)-কে, প্রোগ্রামকে না       │
  └────────────────────────────┬──────────────────────────────┘
                               ▼
  ┌──────────────── ld.so (dynamic linker/loader) ─────────────┐
  │ ৫. প্রোগ্রামের "NEEDED" তালিকা পড়ে:                          │
  │       libmathutil.so.1, libc.so.6                          │
  │ ৬. প্রতিটা library খোঁজে (নিচের ক্রমে)                      │
  │ ৭. খুঁজে পেলে মেমোরিতে map করে (mmap)                        │
  │ ৮. symbol মেলায়: add → libmathutil-এর ঠিকানা,               │
  │                  puts → libc-এর ঠিকানা                      │
  │ ৯. নিয়ন্ত্রণ দেয় প্রোগ্রামের শুরুতে → main()                 │
  └────────────────────────────────────────────────────────────┘
```

`ld.so` (বা `ld-linux.so`) নিজেও একটা shared library, আবার নিজেই চলতে পারে। man page: `man ld.so`। এর আসল পাথ distro ও আর্কিটেকচার অনুযায়ী বদলায়: x86_64-এ `/lib64/ld-linux-x86-64.so.2`, 64-bit ARM-এ `/lib/ld-linux-aarch64.so.1`।

প্রোগ্রাম কোন কোন library চায় দেখতে:

```console
$ readelf -d calc_shared | grep NEEDED
 0x0000000000000001 (NEEDED)             Shared library: [libmathutil.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
```

দেখো, এখানে **soname** (`libmathutil.so.1`) লেখা আছে, linker name বা real name না।

### `ld.so` কোন ক্রমে library খোঁজে

1. binary-র ভেতরে লেখা **`DT_RPATH`** (শুধু যদি `DT_RUNPATH` না থাকে; পুরনো পদ্ধতি)
2. **`LD_LIBRARY_PATH`** environment variable-এর ডিরেক্টরিগুলো (setuid প্রোগ্রামে উপেক্ষা করা হয়)
3. binary-র ভেতরে লেখা **`DT_RUNPATH`**
4. **`/etc/ld.so.cache`** ফাইল (যেটা `ldconfig` বানায়)
5. ডিফল্ট ডিরেক্টরি: `/lib`, `/usr/lib` (64-bit সিস্টেমে distro অনুযায়ী `/lib64`, `/usr/lib64` বা `/usr/lib/x86_64-linux-gnu`)

> 💡 **উপমা:** `ld.so` হলো একজন কুরিয়ার। প্যাকেটে (প্রোগ্রামে) লেখা আছে "libmathutil.so.1 লাগবে"। সে আগে দেখে প্যাকেটের গায়ে কোনো বিশেষ ঠিকানা (RPATH) লেখা আছে কিনা, তারপর তোমার হাতে লেখা চিরকুট (`LD_LIBRARY_PATH`), তারপর তার নিজের ফোনবুক (`ld.so.cache`), শেষে পরিচিত গুদামগুলো (`/usr/lib`)। কোথাও না পেলে হাল ছেড়ে দেয়, আর সেটাই ওই "cannot open shared object file" error।

---

## ৩১.৭ `ldd`, `LD_LIBRARY_PATH`, `ldconfig`

### `ldd`: কোন library লাগবে আর কোথায় পাওয়া গেল

**কাজ:** একটা প্রোগ্রাম বা library কোন shared library-র উপর নির্ভর করে, আর সেগুলো এখন কোথায় resolve হবে দেখায়।

**সিনট্যাক্স:** `ldd [অপশন] ফাইল...`

| অপশন | অর্থ |
|---|---|
| `-v` | বিস্তারিত, symbol version তথ্যসহ |
| `-u` | যেসব library link হয়েছে কিন্তু আসলে ব্যবহার হয় না |
| `-d` / `-r` | data/function relocation চালিয়ে missing symbol রিপোর্ট করো |

```console
$ ldd /usr/bin/ls
        linux-vdso.so.1 (0x00007ffd5a1f2000)
        libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1 (0x00007f8c3a5e0000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f8c3a200000)
        libpcre2-8.so.0 => /lib/x86_64-linux-gnu/libpcre2-8.so.0 (0x00007f8c3a540000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f8c3a640000)
```

(Fedora/RHEL-এ পাথ হবে `/lib64/libc.so.6`।)

- `linux-vdso.so.1`: কোনো ফাইল না, kernel নিজেই প্রতিটা প্রসেসে এটা ঢুকিয়ে দেয় যাতে `gettimeofday` এর মতো কিছু system call দ্রুত হয় ([অধ্যায় ৪৭](47-kernel-proc-sys.md))
- `=>` এর পরে আসল পাথ
- শেষেরটা `ld.so` নিজে

আমাদের ভাঙা প্রোগ্রাম:

```console
$ ldd calc_shared
        linux-vdso.so.1 (0x00007ffc...)
        libmathutil.so.1 => not found
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f...)
        /lib64/ld-linux-x86-64.so.2 (0x00007f...)
```

**`not found`** দেখেই সমস্যা ধরা গেল। বাস্তবে কোনো প্রোগ্রাম "error while loading shared libraries" দিলে প্রথম কাজ: `ldd` চালাও।

> ⚠️ **সতর্কতা:** অবিশ্বস্ত (অচেনা জায়গা থেকে নামানো) binary-তে `ldd` চালাবে না। কিছু ক্ষেত্রে `ldd` প্রোগ্রামটাকে আসলেই চালিয়ে ফেলতে পারে (`man ldd`-এ এই সতর্কতা লেখা আছে)। নিরাপদ বিকল্প: `objdump -p FILE | grep NEEDED` বা `readelf -d FILE`।

### `LD_LIBRARY_PATH`: সাময়িক সমাধান

```console
$ LD_LIBRARY_PATH=. ./calc_shared
2 + 3 = 5
4^2  = 16

$ LD_LIBRARY_PATH=$PWD ldd calc_shared | grep mathutil
        libmathutil.so.1 => /home/rahim/build-lab/hello/libmathutil.so.1 (0x00007f...)
```

একাধিক ডিরেক্টরি `:` দিয়ে আলাদা করো: `LD_LIBRARY_PATH=/opt/foo/lib:/opt/bar/lib`।

> ⚠️ **সতর্কতা:** `LD_LIBRARY_PATH` স্থায়ীভাবে `.bashrc`-এ `export` করে রাখা খারাপ অভ্যাস। এতে সিস্টেমের সব প্রোগ্রাম আগে ওই ডিরেক্টরিতে খুঁজবে। ওখানে কোনো ভুল ভার্সনের `libssl.so.3` থাকলে অন্য প্রোগ্রামও ভেঙে যেতে পারে, ধীরও হয়। এটা টেস্ট আর ডিবাগের জন্য, অথবা একটা নির্দিষ্ট প্রোগ্রামের wrapper স্ক্রিপ্টের ভেতরে ব্যবহারের জন্য।

**RPATH/RUNPATH** দিয়ে binary-র ভেতরেই পাথ লিখে দেওয়া যায়:

```console
$ gcc main.c -L. -lmathutil -Wl,-rpath,'$ORIGIN' -o calc_rpath
$ ./calc_rpath
2 + 3 = 5
4^2  = 16
$ readelf -d calc_rpath | grep -E 'RPATH|RUNPATH'
 0x000000000000001d (RUNPATH)            Library runpath: [$ORIGIN]
```

`$ORIGIN` মানে "binary যে ডিরেক্টরিতে আছে"। আধুনিক toolchain ডিফল্টভাবে `RUNPATH` লেখে। (single quote দিতে হয় যাতে shell `$ORIGIN`-কে variable ভেবে বদলে না ফেলে।)

### `ldconfig` আর `/etc/ld.so.conf`: স্থায়ী সমাধান

প্রতিবার `ld.so` যদি পুরো ডিস্কের সব ডিরেক্টরি ঘেঁটে library খুঁজত, প্রোগ্রাম চালু হতে অনেক সময় লাগত। তাই একটা **cache** (`/etc/ld.so.cache`) থাকে, যেটা "soname → পাথ" এর তালিকা। এই cache বানায় **`ldconfig`**।

```
  /etc/ld.so.conf
     └── include /etc/ld.so.conf.d/*.conf
             ├── libc.conf              → /usr/local/lib   (Debian/Ubuntu)
             ├── x86_64-linux-gnu.conf  → /usr/local/lib/x86_64-linux-gnu, ...
             └── myapp.conf             → /opt/myapp/lib   (তুমি যোগ করবে)
                       │
                       ▼   sudo ldconfig  (ডিরেক্টরি স্ক্যান করে)
               /etc/ld.so.cache   (binary ফাইল, ld.so এটা পড়ে)
```

**সিনট্যাক্স:** `ldconfig [অপশন] [ডিরেক্টরি...]`

| অপশন | অর্থ |
|---|---|
| (কোনো অপশন নেই) | cache নতুন করে বানাও, soname symlink আপডেট করো (root লাগে) |
| `-p` | cache-এ বর্তমানে কী আছে তা দেখাও |
| `-v` | কী করছে বিস্তারিত দেখাও |
| `-n DIR` | শুধু ওই ডিরেক্টরিতে symlink বানাও, cache ছুঁয়ো না |
| `-N` | cache আপডেট কোরো না (শুধু symlink) |

```console
$ ldconfig -p | grep libssl
        libssl.so.3 (libc6,x86-64) => /lib/x86_64-linux-gnu/libssl.so.3
$ ldconfig -p | head -1
1234 libs found in cache `/etc/ld.so.cache'
```

আমাদের library সিস্টেম-ওয়াইড ইনস্টল করি:

```console
$ sudo cp libmathutil.so.1.0.0 /usr/local/lib/
$ sudo ldconfig
$ ls -l /usr/local/lib/libmathutil*
lrwxrwxrwx 1 root root    20 ... /usr/local/lib/libmathutil.so.1 -> libmathutil.so.1.0.0
-rwxr-xr-x 1 root root 15560 ... /usr/local/lib/libmathutil.so.1.0.0
$ ./calc_shared
2 + 3 = 5
4^2  = 16
```

খেয়াল করো, `ldconfig` নিজেই soname symlink (`libmathutil.so.1`) বানিয়ে দিয়েছে।

> ⚠️ **Distro পার্থক্য:** Debian/Ubuntu-তে `/usr/local/lib` আগে থেকেই `/etc/ld.so.conf.d/libc.conf`-এ আছে। কিন্তু **Fedora/RHEL-এ `/usr/local/lib` বা `/usr/local/lib64` ডিফল্টভাবে খোঁজার তালিকায় নেই।** সেখানে নিজে যোগ করতে হবে:
>
> ```console
> $ echo '/usr/local/lib' | sudo tee /etc/ld.so.conf.d/usr-local.conf
> $ echo '/usr/local/lib64' | sudo tee -a /etc/ld.so.conf.d/usr-local.conf
> $ sudo ldconfig
> ```

> 📝 **মনে রাখো:** নতুন shared library ইনস্টল করার পরে **`sudo ldconfig`** চালাও। প্যাকেজ ম্যানেজার এটা নিজে করে, কিন্তু `make install` অনেক সময় করে না।

### Static link করা binary

```console
$ gcc -static hello.c -o hello_static
$ ls -lh hello hello_static
-rwxrwxr-x 1 rahim rahim  16K ... hello
-rwxrwxr-x 1 rahim rahim 880K ... hello_static
$ ldd hello_static
        not a dynamic executable
$ file hello_static
hello_static: ELF 64-bit LSB executable, x86-64, ..., statically linked, ...
```

৫০ গুণের বেশি বড়, কারণ libc-এর দরকারি অংশ ভেতরে কপি হয়ে গেছে। (সাইজ ভার্সন অনুযায়ী আলাদা হবে।)

> ⚠️ **Distro পার্থক্য:** Fedora/RHEL-এ `-static` কাজ করতে `glibc-static` প্যাকেজ লাগে (`sudo dnf install glibc-static`; RHEL 9-এ এটা CRB রিপোজিটরিতে থাকে)। না থাকলে "cannot find -lc" error আসে। Debian/Ubuntu-তে `libc6-dev`-এর সাথেই `libc.a` আসে।

---

## ৩১.৮ `make` আর `Makefile`: বিল্ড স্বয়ংক্রিয় করা

### সমস্যা

১০০টা `.c` ফাইল থাকলে হাতে ১০০ বার `gcc -c` লিখবে? একটা ফাইল বদলালে কোনগুলো আবার compile করতে হবে মনে রাখবে? না, এটা করে **`make`**।

`make` একটা `Makefile` পড়ে, যেখানে লেখা থাকে কোন ফাইল কোন ফাইল থেকে তৈরি হয় (**dependency**) আর কীভাবে তৈরি হয় (**recipe**)। তারপর ফাইলের **modification time** (শেষ পরিবর্তনের সময়) মিলিয়ে দেখে: target-এর চেয়ে কোনো dependency নতুন হলে শুধু সেটাই আবার বিল্ড করে।

> 💡 **উপমা:** `Makefile` হলো রান্নাঘরে টাঙানো একটা চার্ট: "পোলাও বানাতে লাগবে সেদ্ধ চাল আর ভাজা পেঁয়াজ; সেদ্ধ চাল বানাতে লাগবে কাঁচা চাল"। `make` এমন এক বুদ্ধিমান সহকারী যে দেখে পেঁয়াজ আগেই ভাজা আছে কিনা। থাকলে আবার ভাজে না, শুধু যা দরকার তাই করে।

### মৌলিক গঠন

```make
target: dependency1 dependency2
	recipe-command
```

> ⚠️ **সবচেয়ে বিখ্যাত ভুল:** recipe লাইনের শুরুতে **TAB** অক্ষর লাগবে, স্পেস না। স্পেস দিলে error আসে:
> `Makefile:2: *** missing separator.  Stop.`
> `vim`-এ `:set list` দিলে TAB দেখা যায় `^I` হিসেবে। `cat -A Makefile` দিয়েও দেখতে পারো।

### আমাদের calc প্রজেক্টের Makefile

```make
# Makefile
CC      = gcc
CFLAGS  = -Wall -Wextra -O2
LDFLAGS =
PREFIX ?= /usr/local

OBJS = main.o mathutil.o

calc: $(OBJS)
	$(CC) $(LDFLAGS) $^ -o $@

main.o: main.c mathutil.h
	$(CC) $(CFLAGS) -c $< -o $@

mathutil.o: mathutil.c mathutil.h
	$(CC) $(CFLAGS) -c $< -o $@

install: calc
	install -D -m 755 calc $(DESTDIR)$(PREFIX)/bin/calc

uninstall:
	rm -f $(DESTDIR)$(PREFIX)/bin/calc

clean:
	rm -f calc $(OBJS)

.PHONY: install uninstall clean
```

### Makefile-এর উপাদান

| উপাদান | অর্থ |
|---|---|
| `VAR = value` | variable সংজ্ঞা; ব্যবহার করার সময় মান বের হয় (recursive) |
| `VAR := value` | তখনই একবার মান বের করে রাখে (simple) |
| `VAR ?= value` | শুধু যদি আগে সেট না থাকে (কমান্ড লাইন বা environment থেকে বদলানো যায়) |
| `$(VAR)` | variable-এর মান |
| `$@` | বর্তমান target-এর নাম |
| `$<` | প্রথম dependency |
| `$^` | সব dependency (ডুপ্লিকেট বাদে) |
| `.PHONY` | এই target গুলো আসলে ফাইল না, শুধু কাজের নাম |
| `#` | comment |
| `%.o: %.c` | pattern rule: যেকোনো `.c` থেকে `.o` |
| `@command` | কমান্ডটা চালানোর আগে স্ক্রিনে দেখিও না |

pattern rule দিয়ে আরও ছোট:

```make
%.o: %.c mathutil.h
	$(CC) $(CFLAGS) -c $< -o $@
```

> 💡 **টিপস:** `make`-এর নিজের অনেক **implicit rule** আছে। কিছু না লিখলেও সে জানে `.c` থেকে `.o` কীভাবে বানাতে হয় (`$(CC) $(CPPFLAGS) $(CFLAGS) -c`)। `make -p` দিয়ে সব built-in rule দেখতে পারো।

`.PHONY` কেন? ধরো ডিরেক্টরিতে কোনোভাবে `clean` নামে একটা ফাইল চলে এলো। তখন `make clean` বলবে "`clean` is up to date" আর কিছু মুছবে না। `.PHONY` বলে দেয় এটা ফাইল না, সবসময় চালাও।

### ব্যবহার

```console
$ make
gcc -Wall -Wextra -O2 -c main.c -o main.o
gcc -Wall -Wextra -O2 -c mathutil.c -o mathutil.o
gcc  main.o mathutil.o -o calc

$ make
make: 'calc' is up to date.

$ touch main.c        # main.c "বদলালাম"
$ make
gcc -Wall -Wextra -O2 -c main.c -o main.o
gcc  main.o mathutil.o -o calc
```

শুধু `main.c` আবার compile হলো, `mathutil.c` না। এটাই `make`-এর জাদু।

```
              calc
             /    \
        main.o    mathutil.o
        /    \       /     \
   main.c  mathutil.h  mathutil.c
                (দুজনেই এটার উপর নির্ভর করে)

   main.c বদলালে → main.o → calc  আবার বিল্ড
   mathutil.h বদলালে → main.o + mathutil.o → calc  সব আবার বিল্ড
```

### `make` অপশন

| অপশন | অর্থ |
|---|---|
| `make` | প্রথম target বিল্ড করো (সাধারণত `all`) |
| `make TARGET` | নির্দিষ্ট target |
| `-j N` | একসাথে N টা কাজ চালাও (parallel build) |
| `-j$(nproc)` | যতগুলো CPU core আছে ততগুলো কাজ |
| `-n` | dry run: কী চালাবে শুধু দেখাও, চালিও না |
| `-B` | সব কিছু জোর করে আবার বিল্ড করো |
| `-C DIR` | আগে ওই ডিরেক্টরিতে গিয়ে তারপর make চালাও |
| `-f FILE` | অন্য নামের Makefile ব্যবহার করো |
| `-k` | error হলেও যতটা সম্ভব চালিয়ে যাও |
| `VAR=value` | variable বদলাও, যেমন `make CFLAGS=-g` |
| `V=1` বা `VERBOSE=1` | অনেক প্রজেক্টে (automake: `V=1`, cmake: `VERBOSE=1`) পুরো কমান্ড দেখায় |

```console
$ nproc
8
$ make -j$(nproc)
```

> 💡 **টিপস:** বড় প্রজেক্টে `-j$(nproc)` বিল্ড কয়েক গুণ দ্রুত করে। তবে RAM কম হলে (যেমন ২ GB) বেশি parallel job সিস্টেম জমিয়ে দিতে পারে; তখন `-j2` দাও।

---

## ৩১.৯ `/usr/local`: নিজের বিল্ড করা সফটওয়্যার কোথায় যাবে

[অধ্যায় ৭](07-filesystem-hierarchy.md)-এ FHS দেখেছ। মূল নিয়ম:

```
  /usr        ← প্যাকেজ ম্যানেজারের এলাকা (apt/dnf এর)। হাত দেবে না।
  │   ├── bin, lib, include, share
  │
  /usr/local  ← স্থানীয় অ্যাডমিনের এলাকা (তুমি)। প্যাকেজ ম্যানেজার এখানে হাত দেয় না।
  │   ├── bin        → এক্সিকিউটেবল (PATH-এ /usr/bin এর আগে থাকে)
  │   ├── sbin
  │   ├── lib        → library
  │   ├── include    → header
  │   ├── share      → man page, ডকুমেন্টেশন, ডেটা
  │   ├── etc        → কনফিগ (কিছু প্রজেক্ট /usr/local/etc ব্যবহার করে)
  │   └── src        → সোর্স কোড রাখার জায়গা (প্রথা)
  │
  /opt/APPNAME ← স্বয়ংসম্পূর্ণ তৃতীয়-পক্ষের সফটওয়্যার (একটা ফোল্ডারে সব)
  │
  ~/.local    ← শুধু তোমার ইউজারের জন্য (root লাগে না), ~/.local/bin
```

কেন `/usr/local`?

1. **সংঘর্ষ এড়ানো:** তুমি `make install` দিয়ে `/usr/bin/python3` ওভাররাইট করলে পরের `apt upgrade` সেটা আবার বদলে দেবে, অথবা ডজন ডজন সিস্টেম টুল (যেগুলো distro-র python-এর উপর নির্ভর করে) ভেঙে যাবে।
2. **ট্র্যাকিং:** প্যাকেজ ম্যানেজার `/usr`-এর প্রতিটা ফাইলের হিসাব রাখে। সেখানে অচেনা ফাইল রাখলে বিভ্রান্তি তৈরি হয়।
3. **অগ্রাধিকার:** ডিফল্ট `PATH`-এ `/usr/local/bin` সাধারণত `/usr/bin`-এর **আগে** থাকে, তাই নিজের বিল্ড করা নতুন ভার্সন আগে চলে।

```console
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

প্রায় সব `configure` স্ক্রিপ্ট আর CMake-এর ডিফল্ট install prefix হলো `/usr/local`। root ছাড়া ইনস্টল করতে চাইলে `--prefix=$HOME/.local` দাও।

> 💡 **টিপস:** নতুন কোনো প্রোগ্রাম ইনস্টলের পরেও shell পুরনো পাথ থেকে চালাচ্ছে মনে হলে `hash -r` চালাও। bash কমান্ডের পাথ মনে রেখে দেয় (cache)। কোনটা চলছে দেখতে `type -a calc` বা `which -a calc` ([অধ্যায় ১২](12-file-khoja.md))।

---

## ৩১.১০ বাস্তব প্রজেক্ট: tarball থেকে GNU hello বিল্ড

**GNU hello** একটা ছোট প্রোগ্রাম যেটা GNU প্রজেক্ট বানিয়েছে, ঠিক এই উদ্দেশ্যে: GNU বিল্ড সিস্টেম কীভাবে কাজ করে তার উদাহরণ দেখানো।

### ক্লাসিক তিন ধাপ, এক নজরে

```
  hello-2.12.1.tar.gz
        │  tar xf       (অধ্যায় ৩৪)
        ▼
  hello-2.12.1/   (configure, Makefile.in, src/, README, INSTALL ...)
        │
        │  ./configure      → তোমার সিস্টেম পরীক্ষা করে, Makefile তৈরি করে
        │                     (compiler আছে? কোন library আছে? কোথায় ইনস্টল?)
        ▼
  Makefile, config.h
        │
        │  make             → compile + link (সোর্স ডিরেক্টরির ভেতরে)
        ▼
  hello (binary)
        │
        │  sudo make install → ফাইলগুলো /usr/local/... তে কপি করে
        ▼
  /usr/local/bin/hello, /usr/local/share/man/man1/hello.1, ...
```

### ধাপ ১: সোর্স ডাউনলোড

```console
$ mkdir -p ~/build-lab && cd ~/build-lab
$ wget https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz
$ wget https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz.sig
```

### ধাপ ২: যাচাই (সতর্ক অভ্যাস)

ইন্টারনেট থেকে নামানো কোড `sudo` দিয়ে ইনস্টল করবে, তাই সেটা আসল কিনা যাচাই করা জরুরি। GNU প্রজেক্ট GPG স্বাক্ষর দেয়:

```console
$ gpg --verify hello-2.12.1.tar.gz.sig hello-2.12.1.tar.gz
gpg: Signature made ...
gpg: Can't check signature: No public key
```

"No public key" মানে রক্ষণাবেক্ষণকারীর public key তোমার কাছে নেই। GNU-র keyring (`https://ftp.gnu.org/gnu/gnu-keyring.gpg`) নামিয়ে import করে আবার যাচাই করতে পারো। অনেক প্রজেক্ট শুধু checksum দেয়; তখন:

```console
$ sha256sum hello-2.12.1.tar.gz
```

আর ওয়েবসাইটে দেওয়া মানের সাথে মিলিয়ে দেখো।

> ⚠️ **সতর্কতা:** অচেনা ওয়েবসাইট বা ব্লগের লিংক থেকে সোর্স নামিয়ে `sudo make install` চালানো মানে সেই কোডকে তোমার পুরো সিস্টেমের মালিকানা দিয়ে দেওয়া। `Makefile`-এর `install` target-এ যেকোনো কমান্ড লেখা থাকতে পারে। সবসময় অফিশিয়াল সোর্স ব্যবহার করো।

### ধাপ ৩: খোলা আর পড়া

```console
$ tar xf hello-2.12.1.tar.gz
$ cd hello-2.12.1
$ ls
ABOUT-NLS  ChangeLog  INSTALL  Makefile.am  NEWS  README  build-aux  configure     contrib  doc  lib  m4  man  po  src  tests ...
```

> 📝 **মনে রাখো:** বিল্ড করার আগে **`README`** আর **`INSTALL`** ফাইল পড়বে। প্রতিটা প্রজেক্টের নিজস্ব নির্দেশনা, dependency তালিকা আর বিশেষ অপশন সেখানে লেখা থাকে।

### ধাপ ৪: `./configure`

`configure` একটা বিশাল shell script (সাধারণত **Autoconf** দিয়ে তৈরি)। এটা তোমার সিস্টেম পরীক্ষা করে:

```console
$ ./configure
checking for a BSD-compatible install... /usr/bin/install -c
checking whether build environment is sane... yes
checking for a thread-safe mkdir -p... /usr/bin/mkdir -p
checking for gawk... no
checking for mawk... mawk
checking whether make sets $(MAKE)... yes
checking for gcc... gcc
checking whether the C compiler works... yes
checking for C compiler default output file name... a.out
...
config.status: creating Makefile
config.status: creating config.h
```

এটা আসলে কী করে:

- ছোট ছোট টেস্ট প্রোগ্রাম লিখে compile করে দেখে (যেমন "এই সিস্টেমে `strnlen` ফাংশন আছে কি?")
- ফলাফল লেখে `config.h`-এ (`#define HAVE_STRNLEN 1`)
- `Makefile.in` টেমপ্লেট থেকে আসল `Makefile` বানায়, তোমার prefix ও অপশন বসিয়ে
- সব লগ রাখে `config.log`-এ

**গুরুত্বপূর্ণ `configure` অপশন:**

| অপশন | অর্থ |
|---|---|
| `--help` | এই প্রজেক্টের সব অপশন দেখাও (সবচেয়ে কাজের) |
| `--prefix=DIR` | ইনস্টলের মূল ডিরেক্টরি (ডিফল্ট `/usr/local`) |
| `--bindir=`, `--libdir=`, `--sysconfdir=` | আলাদা আলাদা ডিরেক্টরি সেট করা |
| `--enable-FEATURE` / `--disable-FEATURE` | ঐচ্ছিক ফিচার চালু/বন্ধ |
| `--with-PACKAGE[=DIR]` / `--without-PACKAGE` | বাইরের কোনো library ব্যবহার করবে কি না |
| `CC=clang` | অন্য compiler |
| `CFLAGS="-O2 -g"` | compiler ফ্ল্যাগ |

```console
$ ./configure --prefix=$HOME/.local        # root ছাড়া, শুধু নিজের জন্য
$ ./configure --help | less
```

**`configure` ব্যর্থ হলে** (সবচেয়ে সাধারণ সমস্যা):

```console
configure: error: Package requirements (libxml-2.0 >= 2.9) were not met:
No package 'libxml-2.0' found
```

মানে একটা library-র **development প্যাকেজ** নেই। চলার জন্য library (`libxml2`) আর বিল্ডের জন্য header + `.so` symlink + `.pc` ফাইল আলাদা প্যাকেজে থাকে:

| distro | নামের ধরন | উদাহরণ |
|---|---|---|
| Debian/Ubuntu | `lib...-dev` | `sudo apt install libxml2-dev` |
| Fedora/RHEL | `...-devel` | `sudo dnf install libxml2-devel` |

কোন প্যাকেজে কোন ফাইল আছে খুঁজতে: Ubuntu-তে `apt-file search libxml/parser.h` (আগে `sudo apt install apt-file && sudo apt-file update`), Fedora-তে `dnf provides '*/libxml/parser.h'` ([অধ্যায় ৩০](30-package-management.md))।

> 💡 **টিপস:** distro-র প্যাকেজটা যে dependency দিয়ে বিল্ড হয়, সব একবারে নামাতে পারো:
> - Debian/Ubuntu: `sudo apt build-dep PACKAGE` (এর জন্য `deb-src` সোর্স চালু থাকতে হয়; Ubuntu 24.04-এ `/etc/apt/sources.list.d/ubuntu.sources` ফাইলে `Types: deb deb-src` করে `sudo apt update`)
> - Fedora/RHEL: `sudo dnf builddep PACKAGE` (`dnf-plugins-core` লাগে; Fedora 41+ এ dnf5-এ built-in)

**`pkg-config`** হলো সেই টুল যেটা `configure` library খুঁজতে ব্যবহার করে:

```console
$ pkg-config --cflags --libs libxml-2.0
-I/usr/include/libxml2 -lxml2
$ pkg-config --modversion libxml-2.0
2.9.14
```

নিজের প্রোগ্রামেও: `gcc app.c $(pkg-config --cflags --libs libxml-2.0) -o app`।

### ধাপ ৫: `make`

```console
$ make -j$(nproc)
make  all-recursive
make[1]: Entering directory '/home/rahim/build-lab/hello-2.12.1'
...
  CC       src/hello.o
  CCLD     hello
...
$ ./hello
Hello, world!
$ ./hello --greeting="আসসালামু আলাইকুম"
আসসালামু আলাইকুম
```

অনেক প্রজেক্টে `make check` বা `make test` দিয়ে টেস্ট চালানো যায়:

```console
$ make check
...
PASS: tests/greeting-1
PASS: tests/hello-1
...
# PASS:  ...
# FAIL:  0
```

### ধাপ ৬: `sudo make install`

```console
$ sudo make install
...
 /usr/bin/mkdir -p '/usr/local/bin'
  /usr/bin/install -c hello '/usr/local/bin'
 /usr/bin/mkdir -p '/usr/local/share/man/man1'
 /usr/bin/install -c -m 644 man/hello.1 '/usr/local/share/man/man1'
...
$ which hello
/usr/local/bin/hello
$ hello --version | head -1
hello (GNU Hello) 2.12.1
$ man hello
```

> 📝 **মনে রাখো:** শুধু `make install` ধাপটায় `sudo` লাগে (কারণ `/usr/local` root-এর)। `./configure` আর `make` কখনো `sudo` দিয়ে চালাবে না। তাহলে সোর্স ডিরেক্টরির সব ফাইলের মালিক root হয়ে যাবে, আর বিল্ড প্রসেসের কোনো ভুল পুরো সিস্টেমে ক্ষতি করতে পারবে ([অধ্যায় ১৮](18-sudo-root.md))।

> 💡 **টিপস:** তিনটা এক লাইনে: `./configure && make -j$(nproc) && sudo make install`। `&&` মানে আগেরটা সফল হলেই পরেরটা চলবে ([অধ্যায় ১৯](19-redirection-pipe.md))।

### `git` থেকে নামানো সোর্সে `configure` না থাকলে

tarball-এ `configure` আগে থেকে তৈরি থাকে, কিন্তু git রিপোজিটরিতে সাধারণত শুধু `configure.ac` আর `Makefile.am` থাকে। তখন আগে `configure` তৈরি করতে হয়:

```console
$ ./autogen.sh        # প্রজেক্ট দিলে, অথবা:
$ autoreconf -fi      # autoconf/automake/libtool লাগবে
$ ./configure
```

### `make clean` আর `make distclean`

| কমান্ড | কী মোছে |
|---|---|
| `make clean` | বিল্ড করা ফাইল (`.o`, binary) |
| `make distclean` | উপরেরটা + `configure`-এর তৈরি ফাইল (`Makefile`, `config.h`, `config.log`); tarball খোলার পরের অবস্থায় ফেরে |

ভিন্ন `--prefix` বা অপশন দিয়ে আবার configure করতে চাইলে আগে `make distclean` করে নাও।

---

## ৩১.১১ আনইনস্টল: সবচেয়ে বড় দুর্বলতা

প্যাকেজ ম্যানেজার জানে কোন প্যাকেজ কোন ফাইল বসিয়েছে, তাই `apt remove` পরিষ্কার করে মুছে দেয়। কিন্তু `make install` কোনো হিসাব রাখে না। ফাইলগুলো শুধু কপি হয়ে যায়।

### `make uninstall`

Automake দিয়ে বানানো প্রজেক্টে (GNU hello সহ) একটা `uninstall` target থাকে:

```console
$ cd ~/build-lab/hello-2.12.1
$ sudo make uninstall
 ( cd '/usr/local/bin' && rm -f hello )
 ...
$ which hello
$
```

শর্ত আর সমস্যা:

- **একই সোর্স ডিরেক্টরি আর একই `configure` অপশন** লাগবে। সোর্স ফোল্ডার মুছে ফেললে বা ভিন্ন `--prefix` দিয়ে আবার configure করলে কাজ করবে না।
- **সব প্রজেক্টে এটা থাকে না।** হাতে লেখা Makefile-এ প্রায়ই নেই। থাকলে `make -n uninstall` দিয়ে আগে দেখে নাও কী মুছবে।
- নতুন ভার্সনের সোর্স দিয়ে পুরনো ভার্সন আনইনস্টল করলে কিছু ফাইল থেকে যেতে পারে।

> 💡 **টিপস:** সোর্স ডিরেক্টরিটা মুছবে না, `/usr/local/src`-এ রেখে দাও। পরে আনইনস্টলের সময় কাজে লাগবে।

### `DESTDIR` দিয়ে আগে দেখে নেওয়া

প্রায় সব ভালো বিল্ড সিস্টেম `DESTDIR` বোঝে। এটা পুরো ইনস্টলটাকে একটা আলাদা ফোল্ডারের ভেতরে করে, প্যাকেজ বানানোর সময় distro-রা ঠিক এটাই করে:

```console
$ make DESTDIR=/tmp/hello-stage install
$ find /tmp/hello-stage -type f
/tmp/hello-stage/usr/local/bin/hello
/tmp/hello-stage/usr/local/share/man/man1/hello.1
/tmp/hello-stage/usr/local/share/info/hello.info
/tmp/hello-stage/usr/local/share/locale/bn/LC_MESSAGES/hello.mo
...
```

এখানে `sudo` লাগেনি, আর তুমি জেনে গেলে ঠিক কোন কোন ফাইল ইনস্টল হবে। এই তালিকা সংরক্ষণ করে রাখলে পরে নিজেই মুছতে পারবে।

### `checkinstall`: `make install` থেকে প্যাকেজ

`checkinstall` হলো `make install` কে ঘিরে চলা একটা টুল। এটা ইনস্টল কোন ফাইলগুলো বসাচ্ছে নজর রাখে, তারপর সেগুলো দিয়ে একটা `.deb` (বা `.rpm`) প্যাকেজ বানিয়ে প্যাকেজ ম্যানেজারের মাধ্যমে ইনস্টল করে। ফলে পরে `apt remove` দিয়ে পরিষ্কার মোছা যায়।

```console
$ sudo apt install checkinstall          # Debian/Ubuntu
$ cd ~/build-lab/hello-2.12.1
$ ./configure && make
$ sudo checkinstall --pkgname=hello-local --pkgversion=2.12.1 --default
...
**********************************************************************
 Done. The new package has been installed and saved to
 /home/rahim/build-lab/hello-2.12.1/hello-local_2.12.1-1_amd64.deb

 You can remove it from your system anytime using:
      dpkg -r hello-local
**********************************************************************

$ dpkg -L hello-local | head
$ sudo apt remove hello-local            # পরিষ্কার আনইনস্টল
```

| অপশন | অর্থ |
|---|---|
| `--pkgname=NAME` | প্যাকেজের নাম |
| `--pkgversion=VER` | ভার্সন |
| `--default` | সব প্রশ্নে ডিফল্ট উত্তর, প্রশ্ন না করে |
| `--install=no` | শুধু `.deb` বানাও, ইনস্টল কোরো না |
| `-D` / `-R` | Debian / RPM প্যাকেজ বানাও |

> ⚠️ **সীমাবদ্ধতা:** `checkinstall` অনেক বছর ধরে সক্রিয়ভাবে উন্নত হচ্ছে না। এটা Ubuntu-র `universe` রিপোজিটরিতে আছে, কিন্তু **Fedora/RHEL-এ অফিশিয়াল রিপোজিটরিতে নেই**। জটিল ইনস্টল স্ক্রিপ্টে মাঝে মাঝে ভুল করে। এর বানানো প্যাকেজে সঠিক dependency তথ্যও থাকে না। নিজের মেশিনে ছোটখাটো জিনিসের জন্য ঠিক আছে; সার্ভারে বা অনেক মেশিনে বিতরণের জন্য আসল প্যাকেজ বানানো শেখো (Debian-এ `dpkg-buildpackage`, Fedora/RHEL-এ `rpmbuild` ও spec ফাইল)।

### আরেকটা কৌশল: প্রতিটা প্রোগ্রাম আলাদা prefix-এ

```console
$ ./configure --prefix=/opt/hello-2.12.1
$ make && sudo make install
$ sudo ln -s /opt/hello-2.12.1/bin/hello /usr/local/bin/hello
```

আনইনস্টল মানে শুধু `sudo rm -rf /opt/hello-2.12.1` আর symlink মোছা। একাধিক ভার্সন পাশাপাশিও রাখা যায়। (GNU Stow নামের টুল এই symlink ব্যবস্থাপনা স্বয়ংক্রিয় করে।)

---

## ৩১.১২ CMake পরিচিতি

Autotools (`configure`) পুরনো আর জটিল। নতুন অনেক C/C++ প্রজেক্ট (যেমন LLVM, KDE, OpenCV) **CMake** ব্যবহার করে। CMake নিজে বিল্ড করে না; সে একটা **build system generator**: `CMakeLists.txt` পড়ে `Makefile` (বা Ninja-র `build.ninja`) বানায়।

```
  CMakeLists.txt  ──cmake (configure)──►  build/Makefile বা build/build.ninja
                                                   │
                                       cmake --build (make/ninja)
                                                   ▼
                                               binary
```

### একটা সহজ `CMakeLists.txt`

আমাদের calc প্রজেক্টের জন্য:

```cmake
cmake_minimum_required(VERSION 3.16)
project(calc VERSION 1.0 LANGUAGES C)

add_library(mathutil SHARED mathutil.c)
add_executable(calc main.c)
target_link_libraries(calc PRIVATE mathutil)

install(TARGETS calc mathutil)
```

### বিল্ড করা ("out-of-source" বিল্ড)

```console
$ cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
-- The C compiler identification is GNU 13.3.0
-- Detecting C compiler ABI info - done
...
-- Configuring done
-- Generating done
-- Build files have been written to: /home/rahim/build-lab/hello/build

$ cmake --build build -j$(nproc)
[ 25%] Building C object CMakeFiles/mathutil.dir/mathutil.c.o
[ 50%] Linking C shared library libmathutil.so
[ 75%] Building C object CMakeFiles/calc.dir/main.c.o
[100%] Linking C executable calc
[100%] Built target calc

$ ./build/calc
2 + 3 = 5
4^2  = 16

$ sudo cmake --install build
-- Install configuration: "Release"
-- Installing: /usr/local/bin/calc
-- Installing: /usr/local/lib/libmathutil.so
...
```

`-S` = source ডিরেক্টরি, `-B` = build ডিরেক্টরি। সব বিল্ড ফাইল `build/` এর ভেতরে থাকে, সোর্স পরিষ্কার থাকে। নতুন করে শুরু করতে শুধু `rm -rf build`।

> 📝 **মনে রাখো:** পুরনো টিউটোরিয়ালে দেখবে `mkdir build && cd build && cmake .. && make && sudo make install`। এটা একই কাজ করে। `cmake --build` আর `cmake --install` (CMake 3.15+) আধুনিক, generator-নিরপেক্ষ পদ্ধতি।

| CMake অপশন / কমান্ড | অর্থ |
|---|---|
| `-S DIR` / `-B DIR` | সোর্স / বিল্ড ডিরেক্টরি |
| `-DCMAKE_INSTALL_PREFIX=/opt/x` | ইনস্টল prefix (ডিফল্ট `/usr/local`) |
| `-DCMAKE_BUILD_TYPE=Release` | `Debug`, `Release`, `RelWithDebInfo`, `MinSizeRel` |
| `-DVAR=VALUE` | প্রজেক্টের কোনো অপশন সেট করা (যেমন `-DBUILD_TESTING=OFF`) |
| `-G Ninja` | Makefile-এর বদলে Ninja ব্যবহার |
| `cmake -LH build` | প্রজেক্টের সব cache variable ও তার ব্যাখ্যা দেখাও |
| `ccmake build` | টার্মিনালে মেনু দিয়ে অপশন বদলানো (`cmake-curses-gui` প্যাকেজ) |
| `cmake --build build --target clean` | clean |
| `cmake --build build -v` | পুরো কমান্ড দেখাও |
| `cmake --install build --prefix ~/.local` | ভিন্ন prefix-এ ইনস্টল |

### CMake-এ আনইনস্টল

CMake ডিফল্টভাবে `uninstall` target দেয় না। তবে `cmake --install` ইনস্টল করা ফাইলের তালিকা রাখে `build/install_manifest.txt`-এ:

```console
$ cat build/install_manifest.txt
/usr/local/bin/calc
/usr/local/lib/libmathutil.so
$ xargs -d '\n' sudo rm -v < build/install_manifest.txt
```

([অধ্যায় ২৩](23-text-tools.md)-এ `xargs` দেখো।) এটা শুধু ফাইল মোছে, তৈরি হওয়া খালি ডিরেক্টরি না।

### অন্যান্য বিল্ড সিস্টেম, এক নজরে

| প্রজেক্টে থাকলে | বিল্ড সিস্টেম | সাধারণ কমান্ড |
|---|---|---|
| `configure` / `configure.ac` | Autotools | `./configure && make && sudo make install` |
| `CMakeLists.txt` | CMake | `cmake -S . -B build && cmake --build build && sudo cmake --install build` |
| `meson.build` | Meson | `meson setup build && meson compile -C build && sudo meson install -C build` |
| শুধু `Makefile` | হাতে লেখা make | `make && sudo make install` (README দেখো) |
| `Cargo.toml` | Rust / Cargo | `cargo build --release` বা `cargo install --path .` |
| `go.mod` | Go | `go build` বা `go install` |
| `pyproject.toml` / `setup.py` | Python | `pip install .` (virtualenv-এর ভেতরে) |

---

## ৩১.১৩ কেন প্যাকেজ ম্যানেজারই প্রথম পছন্দ

এতকিছু শেখার পরে সবচেয়ে গুরুত্বপূর্ণ উপদেশ: **যদি প্যাকেজ ম্যানেজারে পাওয়া যায়, সেটাই ব্যবহার করো।** কারণ:

| বিষয় | প্যাকেজ ম্যানেজার (`apt`/`dnf`) | সোর্স থেকে বিল্ড |
|---|---|---|
| Security আপডেট | `apt upgrade`-এ স্বয়ংক্রিয় | তোমাকে নিজে খবর রেখে আবার বিল্ড করতে হবে |
| Dependency | নিজে খুঁজে ইনস্টল করে | হাতে খুঁজে খুঁজে `-dev` প্যাকেজ ইনস্টল |
| আনইনস্টল | `apt remove`, পরিষ্কার | `make uninstall` থাকলে ভাগ্য ভালো |
| ফাইলের হিসাব | `dpkg -L` / `rpm -ql` | কোনো হিসাব নেই |
| সত্যতা যাচাই | GPG-স্বাক্ষরিত রিপোজিটরি, স্বয়ংক্রিয় | নিজে করতে হবে |
| টেস্টিং | distro আর হাজারো ইউজার পরীক্ষা করেছে | তুমিই প্রথম পরীক্ষক হতে পারো |
| distro-র সাথে মিল | distro-র নিয়ম মেনে কনফিগ, systemd unit, SELinux/AppArmor নীতি | এগুলো নিজে করতে হবে |
| সময় | কয়েক সেকেন্ড | কয়েক মিনিট থেকে কয়েক ঘণ্টা |

### সোর্সে যাওয়ার আগে বিকল্পগুলো দেখো

1. distro-র অফিশিয়াল রিপোজিটরি (`apt search`, `dnf search`)
2. Debian-এ **backports**, Fedora/RHEL-এ **EPEL** (RHEL/Rocky-র জন্য অতিরিক্ত প্যাকেজ)
3. প্রজেক্টের নিজস্ব অফিশিয়াল রিপোজিটরি (যেমন Docker, PostgreSQL, Node.js এর নিজস্ব apt/dnf repo)
4. Ubuntu PPA বা Fedora COPR (তৃতীয় পক্ষের, বিশ্বাসের প্রশ্ন আছে)
5. **Flatpak**, **Snap**, **AppImage** (মূলত ডেস্কটপ অ্যাপ)
6. **কন্টেইনার** ([অধ্যায় ৪৮](48-containers-virtualization.md)): সার্ভারে প্রায়ই সবচেয়ে পরিষ্কার উপায়
7. প্রজেক্টের দেওয়া pre-built binary
8. শেষ উপায়: সোর্স থেকে বিল্ড, আর তখন `/usr/local` বা `/opt` এ, `checkinstall` বা আসল প্যাকেজ দিয়ে

> ⚠️ **সতর্কতা:** distro-র মূল লাইব্রেরি (`glibc`, `openssl`) বা মূল টুল (`python3`, `gcc`) সোর্স থেকে বিল্ড করে `/usr`-এ বসানো সিস্টেম ভাঙার সবচেয়ে নিশ্চিত উপায়। glibc ভুলভাবে বদলালে প্রায় কোনো প্রোগ্রামই চলবে না, `sudo` বা `ls` ও না। নতুন Python দরকার হলে `./configure --prefix=/usr/local` দিয়ে বিল্ড করে **`sudo make altinstall`** ব্যবহার করো (CPython-এর README-তে এটা বলা আছে), যাতে `python3` নামটা ওভাররাইট না হয়ে `python3.13` এর মতো ভার্সন-নির্দিষ্ট নামে ইনস্টল হয়।

---

## ৩১.১৪ সাধারণ সমস্যা ও সমাধান

| সমস্যা / error | কারণ | সমাধান |
|---|---|---|
| `gcc: command not found` | বিল্ড টুল নেই | `build-essential` / `Development Tools` |
| `fatal error: zlib.h: No such file or directory` | header নেই, মানে `-dev`/`-devel` প্যাকেজ নেই | `zlib1g-dev` / `zlib-devel` |
| `configure: error: ... not found` | dependency নেই | `config.log` পড়ো, তারপর `-dev` প্যাকেজ |
| `undefined reference to 'sqrt'` | link-এ library দেওয়া হয়নি | `-lm` যোগ করো (কমান্ডের শেষে) |
| `cannot find -lfoo` | link time-এ `libfoo.so`/`.a` নেই | `-dev` প্যাকেজ, বা `-L DIR` |
| `error while loading shared libraries: libfoo.so.1` | run time-এ `ld.so` খুঁজে পাচ্ছে না | `ldd` দিয়ে দেখো; `sudo ldconfig`, `/etc/ld.so.conf.d/`, বা `LD_LIBRARY_PATH` |
| `Makefile:N: *** missing separator.` | recipe-তে TAB-এর বদলে স্পেস | TAB দাও |
| `make: *** No targets specified and no makefile found.` | `Makefile` নেই | আগে `./configure` / `cmake` চালাও |
| `./configure: Permission denied` | execute permission নেই | `chmod +x configure` বা `sh configure` |
| বিল্ডের মাঝে সিস্টেম জমে যাওয়া / `Killed` | RAM শেষ (OOM killer) | `-j` কমাও, swap বাড়াও ([অধ্যায় ৪৬](46-performance-troubleshooting.md)) |
| নতুন ইনস্টল করলাম কিন্তু পুরনো ভার্সন চলছে | shell-এর hash cache / PATH ক্রম | `hash -r`, `type -a PROG` |
| `version 'GLIBC_2.38' not found` | নতুন glibc-ওয়ালা সিস্টেমে বিল্ড করা binary পুরনো সিস্টেমে চালানো | যে সিস্টেমে চালাবে সেখানেই (বা পুরনো glibc-এ) বিল্ড করো |

> 💡 **টিপস:** লম্বা বিল্ডের আউটপুটে আসল error খুঁজে পাওয়া কঠিন। প্রথম `error:` লাইনটাই সাধারণত আসল কারণ, পরেরগুলো তার ফলাফল। লগ ফাইলে সংরক্ষণ করো: `make 2>&1 | tee build.log`, তারপর `grep -n 'error' build.log`। দীর্ঘ বিল্ড SSH-এ চালালে `tmux`-এর ভেতরে চালাও ([অধ্যায় ২৫](25-jobs-tmux.md))।

---

## ৩১.১৫ কাজের কীবোর্ড শর্টকাট

এই অধ্যায়ের নিজস্ব কোনো শর্টকাট নেই, তবে বিল্ড করার সময় এগুলো সবচেয়ে বেশি কাজে লাগে (বিস্তারিত [অধ্যায় ৬](06-terminal-shortcuts.md)):

| শর্টকাট | কাজ |
|---|---|
| `Ctrl+C` | চলমান `make`/`configure` থামাও |
| `Ctrl+Z` তারপর `bg` | বিল্ডকে background-এ পাঠাও |
| `Ctrl+R` | আগের লম্বা `./configure --prefix=...` কমান্ড খুঁজে বের করো |
| `Ctrl+L` | স্ক্রিন পরিষ্কার |
| `Shift+PageUp` / `Shift+PageDown` | অনেক টার্মিনালে আউটপুট উপরে-নিচে স্ক্রল |
| `Ctrl+Shift+F` | GNOME Terminal-এ আউটপুটে খোঁজা |
| `!!` | আগের কমান্ড আবার (যেমন `sudo !!`) |
| `Alt+.` | আগের কমান্ডের শেষ argument (যেমন tarball-এর নাম) |
| `Tab` | ফাইলের নাম auto-complete (`hello-2.12.1.tar.gz` টাইপ করতে হয় না) |

---

## হাতে-কলমে অনুশীলন

### ল্যাব ১: চার ধাপ নিজের চোখে

```console
$ sudo apt install build-essential          # বা: sudo dnf group install development-tools
$ mkdir -p ~/build-lab/lab1 && cd ~/build-lab/lab1
$ printf '#include <stdio.h>\n#define NAME "Linux"\nint main(void){ printf("Hello, %%s!\\n", NAME); return 0; }\n' > hi.c
$ cat hi.c
$ gcc -E hi.c | tail -5        # NAME কি "Linux" দিয়ে বদলে গেছে?
$ gcc -S hi.c && less hi.s      # assembly দেখো
$ gcc -c hi.c && nm hi.o        # T main, U printf
$ gcc hi.o -o hi && ./hi
Hello, Linux!
$ file hi && ldd hi
```

প্রশ্ন: `nm hi.o`-তে এবার `printf` এলো নাকি `puts`? কেন? (ইঙ্গিত: format string-এ `%s` আছে।)

### ল্যাব ২: static ও shared library

1. ৩১.৪-এর `mathutil.h`, `mathutil.c`, `main.c` তিনটা ফাইল `~/build-lab/lab2`-এ বানাও।
2. `ar rcs` দিয়ে `libmathutil.a` বানিয়ে `calc_static` link করো। `ldd calc_static`-এ `libmathutil` আছে কিনা দেখো (থাকার কথা না, ওটা ভেতরে ঢুকে গেছে; libc কিন্তু থাকবে)।
3. ৩১.৫-এর মতো `libmathutil.so.1.0.0` আর symlink বানাও, `calc_shared` link করো।
4. `./calc_shared` চালিয়ে error দেখো, `ldd` দিয়ে `not found` নিশ্চিত করো।
5. `LD_LIBRARY_PATH=. ./calc_shared` দিয়ে চালাও।
6. এবার মজা: `mathutil.c`-তে `square` বদলে `return x * x * x;` করো, শুধু `.so` আবার বিল্ড করো (main আবার link না করে)। `LD_LIBRARY_PATH=. ./calc_shared` চালাও। ফল বদলেছে! `./calc_static` চালাও। বদলায়নি! shared আর static-এর পার্থক্য হাতেনাতে দেখলে।

### ল্যাব ৩: নিজের Makefile

1. ল্যাব ২-এর ফোল্ডারে ৩১.৮-এর `Makefile` লেখো (TAB মনে রেখো)।
2. `make`, আবার `make` (up to date), `touch mathutil.h`, `make` চালিয়ে দেখো কোনগুলো আবার বিল্ড হলো।
3. `make -n clean` দিয়ে dry run দেখো, তারপর `make clean`।
4. `make install PREFIX=$HOME/.local` দিয়ে root ছাড়া ইনস্টল করো, `ls ~/.local/bin`।
5. `make DESTDIR=/tmp/stage install` দিয়ে দেখো ফাইল কোথায় যায়।

### ল্যাব ৪: GNU hello, পুরো চক্র

```console
$ cd ~/build-lab
$ wget https://ftp.gnu.org/gnu/hello/hello-2.12.1.tar.gz
$ sha256sum hello-2.12.1.tar.gz
$ tar xf hello-2.12.1.tar.gz && cd hello-2.12.1
$ less README INSTALL
$ ./configure --help | less
$ ./configure --prefix=$HOME/.local
$ make -j$(nproc)
$ make check
$ make install                    # ~/.local, তাই sudo লাগবে না
$ ~/.local/bin/hello --greeting="শুভ সকাল"
$ make uninstall
$ ls ~/.local/bin/hello           # No such file or directory
```

বোনাস (Debian/Ubuntu): `make distclean`, তারপর ডিফল্ট prefix দিয়ে configure করে `sudo checkinstall --pkgname=hello-local --pkgversion=2.12.1 --default` চালাও। `dpkg -L hello-local` দিয়ে ফাইলের তালিকা দেখো, শেষে `sudo apt remove hello-local`।

### ল্যাব ৫: CMake

ল্যাব ২-এর ফোল্ডারে ৩১.১২-এর `CMakeLists.txt` লেখো, `sudo apt install cmake` (বা `dnf`), তারপর:

```console
$ cmake -S . -B build -DCMAKE_INSTALL_PREFIX=$HOME/.local
$ cmake --build build -j$(nproc)
$ ./build/calc
$ cmake --install build
$ cat build/install_manifest.txt
$ ldd ~/.local/bin/calc | grep mathutil
```

শেষের কমান্ডে কী দেখলে? CMake ইনস্টলের সময় RPATH মুছে দেয় (ডিফল্ট আচরণ), তাই `libmathutil.so` `not found` দেখাতে পারে, কারণ `~/.local/lib` লাইব্রেরি খোঁজার তালিকায় নেই। এটা ঠিক করার দুটো উপায় ভাবো: `LD_LIBRARY_PATH`, নাকি `-DCMAKE_INSTALL_RPATH=$HOME/.local/lib`? কোনটা ভালো, কেন?

শেষে পরিষ্কার: `xargs -d '\n' rm -v < build/install_manifest.txt`।

---

## সারসংক্ষেপ

মূল ধারণা: সোর্স → **preprocess** (`.i`) → **compile** (`.s`) → **assemble** (`.o`) → **link** (executable)। shared library চালানোর সময় `ld.so` লোড করে; static library বিল্ডের সময় ভেতরে কপি হয়। নিজের বিল্ড করা জিনিস `/usr/local`-এ যায়, `/usr`-এ না। আর সম্ভব হলে প্যাকেজ ম্যানেজারই ব্যবহার করো।

| কমান্ড | কাজ |
|---|---|
| `sudo apt install build-essential` | Debian/Ubuntu-তে বিল্ড টুল |
| `sudo dnf group install development-tools` | Fedora 41+ এ বিল্ড টুল (RHEL 9: `dnf groupinstall "Development Tools"`) |
| `gcc file.c -o prog` | compile + link |
| `gcc -Wall -g -O2` | warning, debug তথ্য, optimization |
| `gcc -E` / `-S` / `-c` | preprocess / assembly / object পর্যন্ত থামো |
| `gcc ... -L DIR -lNAME` | `libNAME` এর সাথে link |
| `ar rcs libX.a a.o b.o` | static library |
| `gcc -fPIC -shared -o libX.so a.c` | shared library |
| `nm file.o` | object-এর symbol তালিকা |
| `file prog` | ELF ধরন, static/dynamic |
| `readelf -d prog` | NEEDED library, RUNPATH |
| `ldd prog` | কোন `.so` লাগবে, কোথায় পাওয়া গেল |
| `LD_LIBRARY_PATH=DIR prog` | সাময়িকভাবে library ডিরেক্টরি যোগ |
| `sudo ldconfig` | library cache নতুন করে বানাও |
| `ldconfig -p` | cache-এ কী আছে |
| `/etc/ld.so.conf.d/*.conf` | স্থায়ী library ডিরেক্টরি |
| `pkg-config --cflags --libs NAME` | library-র compile/link ফ্ল্যাগ |
| `tar xf X.tar.gz` | tarball খোলা |
| `./configure --prefix=DIR` | সিস্টেম পরীক্ষা, Makefile তৈরি |
| `make -j$(nproc)` | parallel বিল্ড |
| `make check` | টেস্ট |
| `sudo make install` | ইনস্টল |
| `make DESTDIR=/tmp/x install` | আলাদা ফোল্ডারে ইনস্টল (দেখে নেওয়া/প্যাকেজিং) |
| `sudo make uninstall` | আনইনস্টল (থাকলে) |
| `make clean` / `make distclean` | বিল্ড ফাইল মোছা / configure-এর আগের অবস্থা |
| `sudo checkinstall` | `make install` থেকে `.deb` বানিয়ে ইনস্টল |
| `cmake -S . -B build` | CMake configure |
| `cmake --build build` | CMake বিল্ড |
| `sudo cmake --install build` | CMake ইনস্টল |
| `apt build-dep PKG` / `dnf builddep PKG` | বিল্ড dependency একবারে |
| `hash -r` | shell-এর কমান্ড পাথ cache মোছা |

---

## নিজেকে যাচাই করো

1. compile প্রক্রিয়ার চারটা ধাপের নাম ক্রমানুসারে বলো, আর প্রতিটা ধাপের আউটপুট ফাইলের এক্সটেনশন কী?
2. "undefined reference to `pow`" error কোন ধাপে আসে, আর সাধারণ সমাধান কী?
3. static library আর shared library-র দুটো সুবিধা ও দুটো অসুবিধা বলো। distro-রা কেন shared library পছন্দ করে?
4. একটা প্রোগ্রাম চালালে "error while loading shared libraries: libfoo.so.2: cannot open shared object file" আসছে। তুমি ধাপে ধাপে কী করবে?
5. `-L` অপশন আর `LD_LIBRARY_PATH` দুটোই "library ডিরেক্টরি" বলে দেয়। পার্থক্য কী?
6. `/usr/local/lib`-এ নতুন `.so` কপি করার পরে কোন কমান্ড চালাতে হয়? Fedora-তে বাড়তি কী করতে হতে পারে?
7. `./configure`, `make` আর `make install` আলাদা আলাদা কী কাজ করে? কোনটায় শুধু `sudo` লাগে, আর কেন বাকিগুলোয় দেওয়া উচিত না?
8. Makefile-এ "missing separator" error কেন আসে? `.PHONY` কেন ব্যবহার করা হয়?
9. নিজের বিল্ড করা সফটওয়্যার `/usr`-এ না রেখে `/usr/local`-এ রাখা হয় কেন?
10. `make install` দিয়ে বসানো সফটওয়্যার পরে পরিষ্কারভাবে মোছার তিনটা উপায় বলো।

<details><summary>উত্তর দেখো</summary>

1. Preprocess (`.i`) → Compile (`.s`, assembly) → Assemble (`.o`, object file) → Link (executable, ডিফল্ট নাম `a.out`)।
2. **Link** ধাপে। `pow` আছে math library `libm`-এ, তাই কমান্ডের শেষে `-lm` যোগ করতে হয়: `gcc prog.c -o prog -lm`। কোড ঠিক আছে, শুধু linker ফাংশনটার সংজ্ঞা পাচ্ছে না।
3. Static: সুবিধা হলো binary একা চলে, dependency সমস্যা নেই, আর চালুর সময় খোঁজাখুঁজি নেই। অসুবিধা হলো binary বড় হয়, আর library-তে security fix এলে প্রতিটা প্রোগ্রাম আবার link করতে হয়। Shared: সুবিধা হলো binary ছোট, RAM-এ এক কপি ভাগাভাগি হয়, আর library আপডেট করলেই সব প্রোগ্রাম ঠিক হয়ে যায়। অসুবিধা হলো `.so` না থাকলে বা ভুল ভার্সন হলে প্রোগ্রাম চলে না। distro-রা shared পছন্দ করে মূলত security আপডেটের সুবিধার জন্য: একটা `libssl` প্যাকেজ আপডেট করলেই শত শত প্রোগ্রাম নিরাপদ হয়।
4. (ক) `ldd prog` চালিয়ে দেখো কোনটা `not found`। (খ) ফাইলটা সিস্টেমে আছে কিনা খোঁজো (`find / -name 'libfoo.so*' 2>/dev/null`, বা `apt-file search` / `dnf provides`)। (গ) না থাকলে প্যাকেজ ম্যানেজার দিয়ে সংশ্লিষ্ট library প্যাকেজ ইনস্টল করো। (ঘ) থাকলে কিন্তু অপ্রচলিত ডিরেক্টরিতে থাকলে সেই ডিরেক্টরি `/etc/ld.so.conf.d/` এর একটা `.conf` ফাইলে লিখে `sudo ldconfig` চালাও। শুধু টেস্টের জন্য হলে `LD_LIBRARY_PATH` ব্যবহার করো।
5. `-L` কাজ করে **বিল্ডের (link) সময়**, linker (`ld`)-এর জন্য। `LD_LIBRARY_PATH` কাজ করে **চালানোর সময়**, dynamic linker (`ld.so`)-এর জন্য। একটা দিয়ে অন্যটার কাজ হয় না, তাই `-L.` দিয়ে বিল্ড হওয়া প্রোগ্রামও চালানোর সময় library খুঁজে না পেতে পারে।
6. `sudo ldconfig`, যেটা `/etc/ld.so.cache` আপডেট করে আর soname symlink বানায়। Fedora/RHEL-এ `/usr/local/lib` (আর `/usr/local/lib64`) ডিফল্ট খোঁজার তালিকায় নেই, তাই আগে `/etc/ld.so.conf.d/usr-local.conf` ফাইলে এগুলো যোগ করতে হয়।
7. `./configure` সিস্টেম পরীক্ষা করে (compiler, library, header) আর তোমার অপশন অনুযায়ী `Makefile` ও `config.h` বানায়। `make` সোর্স compile আর link করে সোর্স ডিরেক্টরির ভেতরেই binary বানায়। `make install` সেগুলো prefix-এর (যেমন `/usr/local`) সঠিক জায়গায় কপি করে। শুধু `make install`-এ `sudo` লাগে, কারণ `/usr/local` root-এর মালিকানায়। বাকিগুলো `sudo` দিয়ে চালালে বিল্ড ফাইলের মালিক root হয়ে যায়, আর বিল্ড প্রসেসের কোনো ভুল বা ক্ষতিকর কোড পুরো সিস্টেমের ক্ষমতা পেয়ে যায়।
8. recipe লাইনের শুরুতে TAB না দিয়ে স্পেস দিলে "missing separator" আসে, কারণ `make` শুধু TAB দেখেই recipe চেনে। `.PHONY` বলে দেয় `clean`/`install` এর মতো target আসলে ফাইল না। নাহলে ওই নামে কোনো ফাইল থাকলে `make` ভাববে target "up to date" আর কাজটা চালাবে না।
9. `/usr` প্যাকেজ ম্যানেজারের এলাকা। সেখানে নিজের ফাইল রাখলে পরের আপডেটে ওভাররাইট হতে পারে, distro-র ফাইল ভেঙে যেতে পারে (যেমন সিস্টেম python বদলে ফেলা), আর কোন ফাইল কার সেই হিসাব গুলিয়ে যায়। `/usr/local` স্থানীয় অ্যাডমিনের জন্য সংরক্ষিত; প্যাকেজ ম্যানেজার সেখানে হাত দেয় না, আর `PATH`-এ এটা `/usr/bin`-এর আগে থাকে।
10. (ক) একই সোর্স ডিরেক্টরি ও একই configure অপশন রেখে `sudo make uninstall` (যদি Makefile দেয়)। (খ) শুরুতেই `checkinstall` দিয়ে ইনস্টল করা, তারপর `apt remove` / `dpkg -r`। (গ) প্রতিটা সফটওয়্যারকে আলাদা prefix-এ (`/opt/NAME-VERSION`) ইনস্টল করে পরে পুরো ফোল্ডার মুছে ফেলা। এছাড়া CMake-এ `install_manifest.txt` থেকে, অথবা `make DESTDIR=... install` দিয়ে আগে সংরক্ষণ করা তালিকা থেকে ফাইল মোছা যায়।

</details>

---

[← আগের অধ্যায়](30-package-management.md) | [সূচিপত্র](README.md) | [পরের অধ্যায় →](32-disk-partition-mount.md)

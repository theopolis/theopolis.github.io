---
title: "Create highly portable ELF binaries using the build-anywhere toolchain"
created: 2018-12-26
tags: 
  - compilers
---

This post describes the basic requirements for compiling highly portable ELF binaries. Essentially using a newer Linux distro like Ubuntu 18.10 to build complex projects that run on older distros like CentOS 6. The details are limited to C/C++ projects and to x86\_64 architectures.

The low-level solution is to use a C++ runtime that requires only glibc 2.13+ runtime linkage and link all third-party libraries as well as the compiler runtime and C++ implementation statically. Do not make a "fully static" binary. You will most likely find a glibc newer than 2.13 on every Linux distribution released since 2011. The high-level solution is to use the `build-anywhere` scripts to build a easy-to-use toolchain and set compiler flags.

<!--more-->

## What is build-anywhere?

I often find I need to build a set of third-party librarys for an executable I'd like to run on a lot of heterogenious Linux installs. I want a "really really static binary" but know in practice this is hard to acheive.

Introducing '[build-anywhere](https://github.com/theopolis/build-anywhere)', a x86\_64 compiler and glibc runtime designed to help you build ultra-portable binaries. The `build-anywhere` toolchain is new, but it has had intial successes and should not change much beyond stabilization.

There are two bits of magic inside:

- Both a static GCC libstdc++ and static LLVM libc++ implementation with C linkage against a glibc 2.13 and zlib 1.2.11.
- Scripts you can source to pick the right CC/CXX/CFLAGS/LDFLAGS options.

Quick TL;DR

- Download [x86\_64-anywhere-linux-gnu-v2.tar.xz](https://github.com/theopolis/build-anywhere/releases/download/v1/x86_64-anywhere-linux-gnu-v2.tar.xz) from the GitHub releases page
- Untar and install where ever you want
- Run `source ./x86_64-anywhere-linux-gnu/scripts/anywhere-setup-security.sh`
- Build your third-party libraries and `make install`
- Build you project
- Check your work with `willitrun`

In practice your third-party dependencies will install shared libraries and your final project will want to link those dispite your efforts to configure the build. You can "cheat" this by removing the shared libraries installed into the `build-anywhere` prefix.

## A few examples

I have installed my anywhere runtime into `/opt/rt`. After sourcing the above script, some of my environment variables change.

The goal is now to work with a few library build systems so they pick up these variables and install static libraries into `$PREFIX`.

```bash
$ source ./x86_64-anywhere-linux-gnu/scripts/anywhere-setup-security.sh
$ env | grep anywhere
PATH=/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot/usr/bin:/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot/sbin:...
CC=clang --gcc-toolchain=/opt/rt/x86_64-anywhere-linux-gnu --sysroot=/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot
CXX=clang++ --gcc-toolchain=/opt/rt/x86_64-anywhere-linux-gnu --sysroot=/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot
CONFIG_SITE=/opt/rt/x86_64-anywhere-linux-gnu/config.site
PKG_CONFIG_PATH=/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot/usr/lib/pkgconfig
LIBRARY_PATH=/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot/usr/lib
CFLAGS=--gcc-toolchain=/opt/rt/x86_64-anywhere-linux-gnu --sysroot=/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot -march=x86-64 -fPIC -fPIE -fstack-protector-all -fsanitize=safe-stack
CXXFLAGS=--gcc-toolchain=/opt/rt/x86_64-anywhere-linux-gnu --sysroot=/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot -march=x86-64 -fPIC -fPIE -fstack-protector-all -fsanitize=safe-stack -stdlib=libc++
PREFIX=/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot/usr
```

What does all of this mean?

- Auto tools, CMake, and other build configuration managers will usually respect the environment's `CC`, `CXX`, `CFLAGS`, `CXXFLAGS`, and `LDFLAGS` settings
- If `pkg-config` is needed, it should prefer configurations in your `build-anywhere` runtime.
- The "prefix" that packages should install into is available as `$PREFIX`.

A lot of things can go wrong here. Off the top of my head, `CPPFLAGS` is not set, `AS` and other assembler flags are not set, you will most likely get duplicate-flag or unused-flag warnings, nothing controls package preference for static or shared library usage or production.

Never-the-less, let's jump in and build `ssdeep`.

```bash
$ wget https://github.com/ssdeep-project/ssdeep/releases/download/release-2.14.1/ssdeep-2.14.1.tar.gz
$ tar xf ssdeep-2.14.1.tar.gz
$ cd ssdeep-2.14.1
$ ./configure --disabled-shared
$ make && make install
```

This installed an `ssdeep` binary and a `libfuzzy.a` static library into the `$PREFIX`. The only non-standard thing here is adding `--disable-shared`.

Let's inspect the resulting `ssdeep` binary's runtime linkage requirements:

```bash
$ ldd -v $PREFIX/bin/ssdeep
	linux-vdso.so.1 (0x00007ffc813e1000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f0ecf0c6000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f0eceea7000)
	librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f0ecec9f000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f0ecea9b000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f0ece6aa000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f0ecf464000)

	Version information:
	./ssdeep:
		libdl.so.2 (GLIBC_2.2.5) => /lib/x86_64-linux-gnu/libdl.so.2
		ld-linux-x86-64.so.2 (GLIBC_2.3) => /lib64/ld-linux-x86-64.so.2
		ld-linux-x86-64.so.2 (GLIBC_2.2.5) => /lib64/ld-linux-x86-64.so.2
		libpthread.so.0 (GLIBC_2.2.5) => /lib/x86_64-linux-gnu/libpthread.so.0
		libc.so.6 (GLIBC_2.4) => /lib/x86_64-linux-gnu/libc.so.6
		libc.so.6 (GLIBC_2.3) => /lib/x86_64-linux-gnu/libc.so.6
		libc.so.6 (GLIBC_2.7) => /lib/x86_64-linux-gnu/libc.so.6
		libc.so.6 (GLIBC_2.2.5) => /lib/x86_64-linux-gnu/libc.so.6
```

This binary depends on glibc 2.7+. This should run anywhere.

Use the provided `willitrun` to double check. This is a quick-script to apply basic linkage dependency checks.

```bash
$ willitrun $PREFIX/bin/ssdeep
✓ /opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot/usr/bin/ssdeep: yes
```

The next example is OpenSSL, the build system requires inspection and will not work out of the box. Here is a working example.

```bash
$ wget https://dl.bintray.com/homebrew/mirror/openssl-1.0.2o.tar.gz 
$ tar xf openssl-1.0.2o.tar.gz
$ cd openssl-1.0.2o
$ ./Configure --prefix=$PREFIX \
  no-ssl3 no-asm no-weak-ssl-ciphers \
  zlib-dynamic no-shared linux-x86_64 \
  "$CFLAGS -Wno-unused-command-line-argument" \
  "$LDFLAGS"
$ make depend
$ make && make install
```

Next try a more complicated build like cmake.

```bash
$ wget https://github.com/Kitware/CMake/releases/download/v3.13.2/cmake-3.13.2.tar.gz
$ tar xf cmake-3.13.2.tar.gz
$ cd cmake-3.13.2
$ ./bootstrap --prefix=$PREFIX
$ make && make install
```

The only non-standard option is adding `--prefix` to `cmake`'s bootstrap script.

Use the provided `Vagrantfile` to be extra sure the resulting `cmake` can run anywhere. Copy the project's `Vagrantfile` to the folder you installed `build-anywhere`, in my example this is `/opt/rt`.

```bash
$ cp Vagrantfile /opt/rt
$ vagrant up centos6
$ vagrant ssh centos6
[vagrant@localhost ~]$ /vagrant/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot/usr/bin/cmake 
Usage

  cmake [options] 
  cmake [options] 
  cmake [options] -S  -B 

Specify a source directory to (re-)generate a build system for it in the
current working directory.  Specify an existing build directory to
re-generate its build system.

Run 'cmake --help' for more information.
```

## Use libc++ by default

The setup scripts will use `libc++` by default because there are no shared versions of these libraries installed with the `build-anywhere` toolchain(!)

Builds like `sleuthkit`'s make assumptions about how you want to link. In these cases the build tries very hard to link the dynamic library. The 'forcing function' flags to use `libc++` over GCC's `libstdc++` are essentially the following (already set with build-anywhere).

```bash
LDFLAGS=$LDFLAGS -l:libc++.a -l:libc++abi.a -l:libunwind.a -lpthread
CXXFLAGS=$CXXFLAGS -stdlib=libc++
```

So if you try to build `sleuthkit`:

```bash
$ wget https://github.com/sleuthkit/sleuthkit/archive/sleuthkit-4.6.1.tar.gz
$ tar xf sleuthkit-4.6.1.tar.gz
$ cd sleuthkit-sleuthkit-4.6.1
$ ./configure --disable-shared
$ make && make install
$ willitrun $PREFIX/bin/icat 
✓ /opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot/usr/bin/icat: yes
```

And if you are curious to compare binary size impact:

- 1,096,968 bytes with shared linkage (normal)
- 1,277,240 bytes with static linkage (+14%)
- 1,039,968 bytes total if you add `-Oz`

The impact is not trivial. You should not be optimizing for binary size and portability simultaniously.

## Add Clang's Control Flow Integrity

It is fairly simple to add additional features like [CFI](https://clang.llvm.org/docs/ControlFlowIntegrity.html).

```bash
$ export LDFLAGS="$LFDLAGS -flto -fsanitize=cfi"
$ export CFLAGS="$CFLAGS -fvisibility=hidden -fsanitize=cfi -flto"
$ export CXXFLAGS="$CXXFLAGS -fvisibility=hidden -fsanitize=cfi -flto"
```

These are not on by default yet. But adding them and building `ssdeep` again works fine.

```bash
$ ssdeep `which ssdeep`
ssdeep,1.1--blocksize:hash:hash,filename
98304:Cj4OKS9OwIrgvVzDnW20B2TgMW8dkcIBN1/hvPWMkCCz51U8B:z0wvh261/VTXCzI,"/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot/usr/bin/ssdeep"
```

Add ASAN by adding `-fsanitize=address` to the three sets of flags, and finds a leak in `ssdeep`:

```bash
=================================================================
==18673==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 32767 byte(s) in 1 object(s) allocated from:
    #0 0x555c30d36857 in __interceptor_malloc /opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot/usr/src/llvm/projects/compiler-rt/lib/asan/asan_malloc_linux.cc:146:3
    #1 0x555c30dc7613 in main /opt/build-anywhere/ssdeep-2.14.1/main.cpp:282:5
    #2 0x7f5bd7458b96 in __libc_start_main /build/glibc-OTsEL5/glibc-2.27/csu/../csu/libc-start.c:310

SUMMARY: AddressSanitizer: 32767 byte(s) leaked in 1 allocation(s).
```

Fixed with:

```diff
--- a/main.cpp
+++ b/main.cpp
@@ -315,6 +315,8 @@ int main(int argc, char **argv)
       ++count;
     }
 
+    free(cwd);
+
     // If we processed files, but didn't find anything large enough
     // to be meaningful, we should display a warning message to the user.
     // This happens mostly when people are testing very small files
```

## Use Clang's libFuzzer

Like above, this is very simple, and the result is portable as well. A helper script you can source is `x86_64-anywhere-linux-gnu/scripts/anywhere-setup-fuzzing.sh`.

Build `ssdeep`'s `libfuzzy` and install, then build a smaller fuzzing harness:

```c
#include <fuzzy.h>
#include <stdlib.h>

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  auto state = fuzzy_new();
  fuzzy_set_total_input_length(state, Size);
  fuzzy_update(state, Data, Size);
  auto result = (char*)malloc(FUZZY_MAX_RESULT);
  fuzzy_digest(state, result, 0);
  fuzzy_free(state);
  free(result);

  return 0;  // Non-zero return values are reserved for future use.
}
```

```bash
$ clang++ \
  --gcc-toolchain=/opt/rt/x86_64-anywhere-linux-gnu \
  --sysroot=/opt/rt/x86_64-anywhere-linux-gnu/x86_64-anywhere-linux-gnu/sysroot \
  -march=x86-64 -fPIC -g -fstack-protector-all -stdlib=libc++ \
  -fsanitize=address,fuzzer \
  -fno-omit-frame-pointer \
  -fsanitize-coverage=trace-cmp fuzz.cpp \
  -static-libgcc -static-libstdc++ -fuse-ld=lld -l:libc++.a -l:libc++abi.a -l:libunwind.a -lpthread \
  -l:libfuzzy.a
```

Essentially the goal is to have all of clang and LLVM's features at your fingertips.

Please reach out if you would like help building complex C/C++ projects such that the output binaries are portable.

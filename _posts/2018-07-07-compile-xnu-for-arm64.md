---
layout: post
title:  "Compiling XNU 4570.41.2 for ARM64"
date:   2018-07-07
categories: xnu arm64
---

Compiling XNU for x86_64 is quite easy, particularly when armed with a [good guide](https://kernelshaman.blogspot.com/2018/01/building-xnu-for-macos-high-sierra-1013.html).
Aside from an extremely helpful [gist](https://gist.github.com/Proteas/fe7bbb4c1b35a50de5e44d7c121d9601) and a
[Twitter thread](https://twitter.com/proteaswang/status/914067270397157376?lang=en) from Proteas, there didn't seem to 
be much on compiling XNU for arm64.

Why would you want to compile XNU for arm64? Well, you definitely can't run it on your i-device. But there could be
many other reasons. For me, a clean compile means I can generate a clean `compile-commands.json` file for use with 
[Woboq](https://fergofrog.com/code/cbowser/xnu/).

# tl;dr
The long and short of this process is, follow the steps from [this blog](https://kernelshaman.blogspot.com/2018/01/building-xnu-for-macos-high-sierra-1013.html),
with the following ammendments:
  1. Use my repository for [xnu](https://github.com/fergofrog/xnu/tree/xnu-4570.41.2) - it contains modifications to the xnu source 
     necessary for building 
     * _Note_: if you would rather build from Apple's source, check the [diff](https://github.com/fergofrog/xnu/commit/3f9807a1601c982580b77859e0aae6a915252c05)
       from my repo for changes I made
  2. a copy of the `TargetConditionals.h` header must be made from the iPhoneOS SDK's `/usr/include` directory to the 
     iPhoneOS SDK's `/System/Library/Frameworks/Kernel.framework/Versions/A/Headers` directory in order to build
     libfirehose
  3. When building: the SDK is "`iphoneos`", the ARCH is "`arm64`", and XNU's build also needs `BUILD_WERROR=0`

# The Details
This process worked for me compiling XNU 4570.41.2 with a clean install of macOS High Sierra (10.13.5) and XCode 9.4, 
however, your mileage may vary.

## CTF Tools
The CTF tools (`ctfconvert`, `ctfdump` and `ctfmerge`) are executed as part of XNU's build and are expected to be in 
your hosts's SDK path. Build is per the blog.

```bash
tar zxvf dtrace-262.tar.gz
cd dtrace-262
mkdir obj sym dst
xcodebuild install \
    -target ctfconvert -target ctfdump -target ctfmerge \
    ARCHS=x86_64 SRCROOT=$PWD OBJROOT=$PWD/obj \
    SYMROOT=$PWD/sym DSTROOT=$PWD/dst
sudo ditto \
    $PWD/dst/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain \
    /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain
cd ..
```

## AvailabilityVersions
AvailabilityVersions provides a perl script used as part of the XNU build. Build and installation is per the blog, 
except the install target is the `iphoneos` SDK.

```bash
tar zxvf AvailabilityVersions-32.30.1.tar.gz
cd AvailabilityVersions-32.30.1
mkdir dst
make install SRCROOT=$PWD DSTROOT=$PWD/dst
sudo ditto \
    $PWD/dst/usr/local/libexec \
    $(xcrun -sdk iphoneos -show-sdk-path)/usr/local/libexec
cd ..
```

## libplatform Headers
Just three header files are required from this project. 

```bash
tar zxvf libplatform-161.20.1.tar.gz
cd libplatform-161.20.1
sudo mkdir -p \
    $(xcrun -sdk iphoneos -show-sdk-path)/usr/local/include/os/internal
sudo ditto $PWD/private/os/internal \
    $(xcrun -sdk iphoneos -show-sdk-path)/usr/local/include/os/internal
cd ..
```

## XNU Headers
The headers are ready to be built now. This is where the first modifications are required. Proteas' suggestion of 
copying the compile and link flags for the ARM64 architecture at this point just works. I'd recommend using my repo and
check the [diff](https://github.com/fergofrog/xnu/commit/3f9807a1601c982580b77859e0aae6a915252c05) to see what I did.

After the headers are installed, a copy of the `TargetConditionals.h` header must be made from the SDK's `/usr/include`
directory to the SDK's `/System/Library/Frameworks/Kernel.framework/Versions/A/Headers` directory in order to build
libfirehose.

```bash
git clone -b xnu-4570.41.2 https://github.com/fergofrog/xnu.git
cd xnu
make LOGCOLORS=y SDKROOT=iphoneos ARCH_CONFIGS=ARM64 KERNEL_CONFIGS=RELEASE installhdrs
sudo ditto $PWD/BUILD/dst $(xcrun -sdk iphoneos -show-sdk-path)
sudo cp $(xcrun -sdk iphoneos -show-sdk-path)/usr/include/TargetConditionals.h \
    $(xcrun -sdk iphoneos -show-sdk-path)/System/Library/Frameworks/Kernel.framework/Versions/A/Headers/TargetConditionals.h
cd ..
```

## libfirehose_kernel From libdispatch
The kernel libfirehose header and objects are required to build XNU. With the XNU headers and `TargetConditionals.h`
header in the right place, this step is the same as before, however `ENABLE_BITCODE` must be set to "no" in the build.

```bash
tar zxvf libdispatch-913.30.4.tar.gz
cd libdispatch-913.30.4
mkdir obj sym dst
xcodebuild install -sdk iphoneos -target libfirehose_kernel \
    SRCROOT=$PWD OBJROOT=$PWD/obj SYMROOT=$PWD/sym DSTROOT=$PWD/dst \
    ENABLE_BITCODE=no
sudo ditto $PWD/dst/usr/local \
    $(xcrun -sdk iphoneos -show-sdk-path)/usr/local
cd ..
```
## XNU
Now it's time to build XNU. With several code modifications (one fix and some code stubbing) and help to the make 
scripts, this is as simple as running one make command.

```bash
cd xnu
make LOGCOLORS=y SDKROOT=iphoneos ARCH_CONFIGS=ARM64 KERNEL_CONFIGS=RELEASE BUILD_WERROR=0
```

From there, you'll have your very own XNU kernel targetting ARM64 located at `BUILD/obj/RELEASE_ARM64/mach` (and an 
unstripped version at `BUILD/obj/RELEASE_ARM64/mach.unstripped`)


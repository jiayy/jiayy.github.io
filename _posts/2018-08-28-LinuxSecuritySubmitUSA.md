---
layout:     post
title:      Linux Security Submit USA 2018
subtitle:   Some new features of kernel security
date:       2018-08-28
author:     jiayy
header-img: img/post-android-bulletin-2017.jpg
catalog: true
tags:
    - android
    - CFI
    - linux security submit
---

Yesterday (20180827) on the Linux Security Submit USA 2018, Jeff Vander Stoep and Sami Tolvanen from Google brought
their talk [Year in Review: Android Kernel Security](https://events.linuxfoundation.jp/wp-content/uploads/2017/11/LSS2018.pdf) 

In this talk, they released some valuable information about android kernel security:

- 1/3 of android vulnerabilities belong to kernel
- the attack surface reduction mitigation (such as selinux) works very well
- other userspace-> kernel mitigations: hardened usercopy and PAN
- other access vectors such as : wifi/usb/dsp/bluetooth/modem lack mitigations
- first android devices with LTO+CFI kernels will ship this year [CFI of LLVM](https://clang.llvm.org/docs/ControlFlowIntegrity.html)

The most important thing is the introduction of CFI

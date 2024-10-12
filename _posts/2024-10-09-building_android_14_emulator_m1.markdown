---
layout: post
title:  "Building an Android 14 (AOSP) Emulator on Apple M1"
date:   2024-10-09 16:00:00 -0000
categories: aosp android
---
![](https://i.imgur.com/lMkLN3v.png)

One day, I suddenly felt like doing something with [AOSP(Android Open Source Project)](https://source.android.com/), for example, building various versions of AVDs, and running some tests. But then, I found that Google officially stopped supporting AOSP builds on macOS from June 22, 2021. Oooppps! But I started looking for other ways to build AOSP, and that’s when I found OrbStack and decided to give it a try!

## OrbStack

[OrbStack](https://orbstack.dev/) is a lightweight macOS app that makes it quick and easy to run Docker containers and Linux. For people frustrated with Docker Desktop’s slow speed and high resource usage, OrbStack has become very popular. Although it’s now a paid service after the beta ended, personal users can still use it for free.

## Why OrbStack

OrbStack has a feature that lets you create Linux virtual machines. But unlike Parallels or UTM, these VMs are more smoothly integrated with macOS. It’s kind of like WSL (Windows Subsystem for Linux) on Windows, where everything works well with the host system’s files and network. OrbStack’s VMs are just like that—they’re easy to set up and use, and they boot up in just a second!

You can also use Rosetta to create x86_64/amd64 machines. And this is where we get a hint for building AOSP! We’re going to create an Intel Ubuntu machine (using Rosetta) in OrbStack and build AOSP in that environment.

![](https://i.imgur.com/4SFIYXP.png)

## Build Environment

Here’s my setup:
* MacBook M1 Pro
* Sonoma 14.7
* Memory 16 GB
* OrbStack 1.7.5
* Ubuntu (Intel) 20.04 (Focal Fossa)

## Build Process

I’ll skip the basic AOSP setup since there are plenty of guides out there. Let’s jump right into the commands. We’ll be downloading the code with `android14-gsi` tag for Android 14’s stable Generic System Image. (I haven’t tested other tags yet!)

Oh, one important thing: <u>don’t use your host machine’s disk</u>. Some older guides suggest creating a case-sensitive partition with diskutil, but you may meet errors when building some image files that way. Instead, download and build the code directly inside the Linux disk that OrbStack provides.

```bash
$ cd ~/aosp/android14-gsi/
$ repo init --depth=1 -u https://android.googlesource.com/platform/manifest -b android14-gsi
$ repo sync -c --no-clone-bundle --no-tags -j2
```

After this, the AOSP code should be downloaded. As for `-j` option (which controls how many parallel jobs are run), don’t set it too high if you have only 16 GB of RAM like me—you might run out of memory and the build will fail. If you have more RAM, feel free to increase the number.

```bash
$ . build/envsetup.sh
$ lunch sdk_phone64_arm64-userdebug
$ m emu_img_zip -j 2
```

### Error 1: Insufficient Memory

That’s the full command for building AOSP for AVD. But as expected, errors might start popping up. The first issue, as mentioned earlier, is memory. If you have 32 GB or more, you probably won’t have any problems. But with 16 GB, you might hit memory issues that cause the build to fail.

To solve this, I increased my swap memory to 32 GB.
```bash
$ sudo swapon --show
NAME       TYPE       SIZE USED  PRIO
/dev/zram0 partition 11.7G 2.7G 32767
/dev/vdc   partition 1024M 2.7M     1
/swapfile  file        32G 6.3M    -2
```

### Error 2: lunch infinite loop

Next, when running the lunch command, I ran into an issue where it seemed stuck in a loop. After waiting for 1 hour, 2 hours... nothing happened. I checked the CPU usage and saw that a process called nsjail was stuck at 100%. [nsjail](https://github.com/google/nsjail) is a tool made by Google to isolate processes during the AOSP build.

```bash
$ top
top - 11:00:03 up 2 days, 53 min,  0 users,  load average: 5.33, 5.87, 5.92
Tasks:  28 total,   2 running,  26 sleeping,   0 stopped,   0 zombie
%Cpu(s): 34.2 us, 12.7 sy,  0.0 ni, 51.3 id,  1.9 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :  12009.5 total,   2693.3 free,   7870.9 used,   1445.4 buff/cache
MiB Swap:  45801.5 total,  43037.4 free,   2764.1 used.   3918.7 avail Mem

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
1291927 d3xter    20   0 7867392   1.7g  28224 S 253.3  14.6   1:10.33 java
1292357 d3xter    20   0 6769036 412832  28180 S 113.3   3.4   0:03.98 java
  30644 d3xter    20   0 1199424    696    608 R 100.0   0.0   1273:48 nsjail
      1 root      20   0 1362268   3268    384 S   0.0   0.0   0:02.42 systemd
      8 root      20   0 1234244   1208    388 S   0.0   0.0   0:01.23 orbstack-helper
    118 root      19  -1 1228916   6156   4068 S   0.0   0.1   0:07.71 systemd-journal
    139 systemd+  20   0 1218636   2176    760 S   0.0   0.0   0:01.57 systemd-network
    156 systemd+  20   0 1216372   2032   1068 S   0.0   0.0   0:01.50 systemd-resolve
    158 root      20   0 1200208   1088    628 S   0.0   0.0   0:01.96 cron
    .
    .
```
But the thing is, you don’t really need it.

```bash
$ mv prebuilds/build-tools/linux-x86/bin/nsjail prebuilds/build-tools/linux-x86/bin/nsjail.old
```

Skipping nsjail allowed the lunch command to run successfully without any errors. And you may get the following warning message. Everything worked as expected!

```bash
20:00:03 ************************************************************
20:00:03 You are building on a machine with 11.7GB of RAM
20:00:03
20:00:03 The minimum required amount of free memory is around 16GB,
20:00:03 and even with that, some configurations may not work.
20:00:03
20:00:03 If you run into segfaults or other errors, try reducing your
20:00:03 -j value.
20:00:03 ************************************************************
20:00:03 Build sandboxing disabled due to nsjail error.
```

### Error 3: Dex2oat failed to compile a boot image.

After fixing above 2 errors, the build was almost done(maybe 80-90%?). But suddenly I got a Dex2oat error.

```bash
ERROR: Dex2oat failed to compile a boot image.It is likely that the boot classpath is inconsistent.Rebuild with ART_BOOT_IMAGE_EXTRA_ARGS="--runtime-arg -verbose:verifier" to see verification errors.
```

Dex2oat is the process of converting DEX files in the boot image to native machine code. This error seems to be caused by that process. Some posts on StackOverflow suggest that it’s related to the kernel version, but since we’re using OrbStack (which runs Linux kernel 6.10.12), downgrading the kernel to 5.x.x isn’t really an option.

So, instead, we’ll modify the code to turn off Dex2oat, skipping pre-optimization for the boot image.

1. `build/core/board_config.mk`
```bash
 205 # Conditional to building on linux, as dex2oat currently does not work on darwin.
 206 ifeq ($(HOST_OS),linux)
 207   WITH_DEXPREOPT := false # true -> false
 208 endif
```
2. `build/core/dex_preopt_config.mk`
```bash
 71   # Non eng linux builds must have preopt enabled so that system server doesn't run as interpreter
 72   # make all as comments!
 73   #ifeq (,$(filter eng, $(TARGET_BUILD_VARIANT)))
 74   #  ifneq (true,$(WITH_DEXPREOPT))
 75   #    ifneq (true,$(WITH_DEXPREOPT_BOOT_IMG_AND_SYSTEM_SERVER_ONLY))
 76   #      $(call pretty-error, DEXPREOPT must be enabled for user and userdebug builds)
 77   #    endif
 78   #  endif
 79   #endif
```

## AVD

### Install AVD Image

Once you fix these 3 errors, you should be able to finish the AOSP build for AVD on Apple M1! The log said it took 6 hours, but in reality, with all the retries due to the errors, it took me about 2 days. I’m sure it’ll be faster on a more powerful machine.

```bash
[ 99% 12962/12964] Create system-qemu.img now
removing out/target/product/emulator64_arm64/system-qemu.img.qcow2
out/host/linux-x86/bin/sgdisk --clear out/target/product/emulator64_arm64/system-qemu.img
[100% 12964/12964] Package: out/target/product/emulator64_arm64/sdk-repo-linux-system-images-eng.d3xter.zip

#### build completed successfully (06:08:40 (hh:mm:ss)) ####
```

After the build is complete, you’ll need to move the built image to the Android SDK path (e.g., /Source/Android/sdk/).

```bash
$ mkdir -p /Source/Android/sdk/system-images/android-34/aosp_custom/ && unzip -o out/target/product/emulator64_arm64/sdk-repo-linux-system-images-eng.d3xter.zip -d /Source/Android/sdk/system-images/android-34/aosp_custom/
```

When you extract the compressed file and check the image, it should look something like this:

```bash
$ d3xter@ubuntu:/Source/Android/sdk/system-images/android-34/aosp_custom$ tree ./
./
└── arm64-v8a
    ├── NOTICE.txt
    ├── VerifiedBootParams.textproto
    ├── advancedFeatures.ini
    ├── build.prop
    ├── data
    │   ├── local.prop
    │   └── misc
    │       ├── apns
    │       │   └── apns-conf.xml
    │       ├── emulator
    │       │   ├── config
    │       │   │   └── radioconfig.xml
    │       │   └── version.txt
    │       └── modem_simulator
    │           ├── etc
    │           │   └── modem_simulator
    │           │       └── files
    │           │           └── numeric_operator.xml
    │           ├── iccprofile_for_carrierapitests.xml
    │           └── iccprofile_for_sim0.xml
    ├── encryptionkey.img
    ├── kernel-ranchu
    ├── ramdisk.img
    ├── source.properties
    ├── system.img
    ├── userdata.img
    └── vendor.img

10 directories, 18 files
```

### Create Image Profile

In the folder where the other emulator images are stored, you’ll find a `package.xml` file. This file helps Android Studio recognize the image. I copied and modified the package.xml file from the android-34 > google_api_playstore image. Make sure the path in the localPackage tag matches exactly with the extracted path.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns2:repository xmlns:ns2="http://schemas.android.com/repository/android/common/02"
    xmlns:ns3="http://schemas.android.com/repository/android/common/01"
    xmlns:ns4="http://schemas.android.com/repository/android/generic/01"
    xmlns:ns5="http://schemas.android.com/repository/android/generic/02"
    xmlns:ns6="http://schemas.android.com/sdk/android/repo/addon2/01"
    xmlns:ns7="http://schemas.android.com/sdk/android/repo/addon2/02"
    xmlns:ns8="http://schemas.android.com/sdk/android/repo/addon2/03"
    xmlns:ns9="http://schemas.android.com/sdk/android/repo/repository2/01"
    xmlns:ns10="http://schemas.android.com/sdk/android/repo/repository2/02"
    xmlns:ns11="http://schemas.android.com/sdk/android/repo/repository2/03"
    xmlns:ns12="http://schemas.android.com/sdk/android/repo/sys-img2/04"
    xmlns:ns13="http://schemas.android.com/sdk/android/repo/sys-img2/03"
    xmlns:ns14="http://schemas.android.com/sdk/android/repo/sys-img2/02"
    xmlns:ns15="http://schemas.android.com/sdk/android/repo/sys-img2/01">
    <license id="android-sdk-arm-dbt-license" type="text">Terms and Conditions

        This is the Android Software Development Kit License Agreement

        1. Introduction

        1.1 The Android Software Development Kit (referred to in the License Agreement as the "SDK" and specifically
        including the Android system files, packaged APIs, and Google APIs add-ons) is licensed to you subject to the
        terms of the License Agreement. The License Agreement forms a legally binding contract between you and Google in
        relation to your use of the SDK.

        1.2 "Android" means the Android software stack for devices, as made available under the Android Open Source
        Project, which is located at the following URL: http://source.android.com/, as updated from time to time.

        1.3 A "compatible implementation" means any Android device that (i) complies with the Android Compatibility
        Definition document, which can be found at the Android compatibility website
        (http://source.android.com/compatibility) and which may be updated from time to time; and (ii) successfully
        passes the Android Compatibility Test Suite (CTS).

        1.4 "Google" means Google Inc., a Delaware corporation with principal place of business at 1600 Amphitheatre
        Parkway, Mountain View, CA 94043, United States.


        2. Accepting the License Agreement

        2.1 In order to use the SDK, you must first agree to the License Agreement. You may not use the SDK if you do
        not accept the License Agreement.

        2.2 By clicking to accept, you hereby agree to the terms of the License Agreement.

        2.3 You may not use the SDK and may not accept the License Agreement if you are a person barred from receiving
        the SDK under the laws of the United States or other countries, including the country in which you are resident
        or from which you use the SDK.

        2.4 If you are agreeing to be bound by the License Agreement on behalf of your employer or other entity, you
        represent and warrant that you have full legal authority to bind your employer or such entity to the License
        Agreement. If you do not have the requisite authority, you may not accept the License Agreement or use the SDK
        on behalf of your employer or other entity.


        3. SDK License from Google

        3.1 Subject to the terms of the License Agreement, Google grants you a limited, worldwide, royalty-free,
        non-assignable, non-exclusive, and non-sublicensable license to use the SDK solely to develop applications for
        compatible implementations of Android.

        3.2 You may not use this SDK to develop applications for other platforms (including non-compatible
        implementations of Android) or to develop another SDK. You are of course free to develop applications for other
        platforms, including non-compatible implementations of Android, provided that this SDK is not used for that
        purpose.

        3.3 You agree that Google or third parties own all legal right, title and interest in and to the SDK, including
        any Intellectual Property Rights that subsist in the SDK. "Intellectual Property Rights" means any and all
        rights under patent law, copyright law, trade secret law, trademark law, and any and all other proprietary
        rights. Google reserves all rights not expressly granted to you.

        3.4 You may not use the SDK for any purpose not expressly permitted by the License Agreement. Except to the
        extent required by applicable third party licenses, you may not copy (except for backup purposes), modify,
        adapt, redistribute, decompile, reverse engineer, disassemble, or create derivative works of the SDK or any part
        of the SDK.

        3.5 Use, reproduction and distribution of components of the SDK licensed under an open source software license
        are governed solely by the terms of that open source software license and not the License Agreement.

        3.6 You agree that the form and nature of the SDK that Google provides may change without prior notice to you
        and that future versions of the SDK may be incompatible with applications developed on previous versions of the
        SDK. You agree that Google may stop (permanently or temporarily) providing the SDK (or any features within the
        SDK) to you or to users generally at Google's sole discretion, without prior notice to you.

        3.7 Nothing in the License Agreement gives you a right to use any of Google's trade names, trademarks, service
        marks, logos, domain names, or other distinctive brand features.

        3.8 You agree that you will not remove, obscure, or alter any proprietary rights notices (including copyright
        and trademark notices) that may be affixed to or contained within the SDK.


        4. Use of the SDK by You

        4.1 Google agrees that it obtains no right, title or interest from you (or your licensors) under the License
        Agreement in or to any software applications that you develop using the SDK, including any intellectual property
        rights that subsist in those applications.

        4.2 You agree to use the SDK and write applications only for purposes that are permitted by (a) the License
        Agreement and (b) any applicable law, regulation or generally accepted practices or guidelines in the relevant
        jurisdictions (including any laws regarding the export of data or software to and from the United States or
        other relevant countries).

        4.3 You agree that if you use the SDK to develop applications for general public users, you will protect the
        privacy and legal rights of those users. If the users provide you with user names, passwords, or other login
        information or personal information, you must make the users aware that the information will be available to
        your application, and you must provide legally adequate privacy notice and protection for those users. If your
        application stores personal or sensitive information provided by users, it must do so securely. If the user
        provides your application with Google Account information, your application may only use that information to
        access the user's Google Account when, and for the limited purposes for which, the user has given you permission
        to do so.

        4.4 You agree that you will not engage in any activity with the SDK, including the development or distribution
        of an application, that interferes with, disrupts, damages, or accesses in an unauthorized manner the servers,
        networks, or other properties or services of any third party including, but not limited to, Google or any mobile
        communications carrier.

        4.5 You agree that you are solely responsible for (and that Google has no responsibility to you or to any third
        party for) any data, content, or resources that you create, transmit or display through Android and/or
        applications for Android, and for the consequences of your actions (including any loss or damage which Google
        may suffer) by doing so.

        4.6 You agree that you are solely responsible for (and that Google has no responsibility to you or to any third
        party for) any breach of your obligations under the License Agreement, any applicable third party contract or
        Terms of Service, or any applicable law or regulation, and for the consequences (including any loss or damage
        which Google or any third party may suffer) of any such breach.

        4.7 This software enables the execution of intellectual property owned by Arm Limited. You agree that your use
        of the software, that allows execution of ARM Instruction Set Architecture (“ISA”) compliant executables for
        application development and debug only on x86 desktop, laptop, customer on-premise servers, and
        customer-procured cloud-based environments.

        5. Your Developer Credentials

        5.1 You agree that you are responsible for maintaining the confidentiality of any developer credentials that may
        be issued to you by Google or which you may choose yourself and that you will be solely responsible for all
        applications that are developed under your developer credentials.

        6. Privacy and Information

        6.1 In order to continually innovate and improve the SDK, Google may collect certain usage statistics from the
        software including but not limited to a unique identifier, associated IP address, version number of the
        software, and information on which tools and/or services in the SDK are being used and how they are being used.
        Before any of this information is collected, the SDK will notify you and seek your consent. If you withhold
        consent, the information will not be collected.

        6.2 The data collected is examined in the aggregate to improve the SDK and is maintained in accordance with
        Google's Privacy Policy.


        7. Third Party Applications

        7.1 If you use the SDK to run applications developed by a third party or that access data, content or resources
        provided by a third party, you agree that Google is not responsible for those applications, data, content, or
        resources. You understand that all data, content or resources which you may access through such third party
        applications are the sole responsibility of the person from which they originated and that Google is not liable
        for any loss or damage that you may experience as a result of the use or access of any of those third party
        applications, data, content, or resources.

        7.2 You should be aware the data, content, and resources presented to you through such a third party application
        may be protected by intellectual property rights which are owned by the providers (or by other persons or
        companies on their behalf). You may not modify, rent, lease, loan, sell, distribute or create derivative works
        based on these data, content, or resources (either in whole or in part) unless you have been specifically given
        permission to do so by the relevant owners.

        7.3 You acknowledge that your use of such third party applications, data, content, or resources may be subject
        to separate terms between you and the relevant third party. In that case, the License Agreement does not affect
        your legal relationship with these third parties.


        8. Using Android APIs

        8.1 Google Data APIs

        8.1.1 If you use any API to retrieve data from Google, you acknowledge that the data may be protected by
        intellectual property rights which are owned by Google or those parties that provide the data (or by other
        persons or companies on their behalf). Your use of any such API may be subject to additional Terms of Service.
        You may not modify, rent, lease, loan, sell, distribute or create derivative works based on this data (either in
        whole or in part) unless allowed by the relevant Terms of Service.

        8.1.2 If you use any API to retrieve a user's data from Google, you acknowledge and agree that you shall
        retrieve data only with the user's explicit consent and only when, and for the limited purposes for which, the
        user has given you permission to do so. If you use the Android Recognition Service API, documented at the
        following URL: https://developer.android.com/reference/android/speech/RecognitionService, as updated from time
        to time, you acknowledge that the use of the API is subject to the Data Processing Addendum for Products where
        Google is a Data Processor, which is located at the following URL:
        https://privacy.google.com/businesses/gdprprocessorterms/, as updated from time to time. By clicking to accept,
        you hereby agree to the terms of the Data Processing Addendum for Products where Google is a Data Processor.


        9. Terminating the License Agreement

        9.1 The License Agreement will continue to apply until terminated by either you or Google as set out below.

        9.2 If you want to terminate the License Agreement, you may do so by ceasing your use of the SDK and any
        relevant developer credentials.

        9.3 Google may at any time, terminate the License Agreement with you if: (A) you have breached any provision of
        the License Agreement; or (B) Google is required to do so by law; or (C) the partner with whom Google offered
        certain parts of SDK (such as APIs) to you has terminated its relationship with Google or ceased to offer
        certain parts of the SDK to you; or (D) Google decides to no longer provide the SDK or certain parts of the SDK
        to users in the country in which you are resident or from which you use the service, or the provision of the SDK
        or certain SDK services to you by Google is, in Google's sole discretion, no longer commercially viable.

        9.4 When the License Agreement comes to an end, all of the legal rights, obligations and liabilities that you
        and Google have benefited from, been subject to (or which have accrued over time whilst the License Agreement
        has been in force) or which are expressed to continue indefinitely, shall be unaffected by this cessation, and
        the provisions of paragraph 14.7 shall continue to apply to such rights, obligations and liabilities
        indefinitely.


        10. DISCLAIMER OF WARRANTIES

        10.1 YOU EXPRESSLY UNDERSTAND AND AGREE THAT YOUR USE OF THE SDK IS AT YOUR SOLE RISK AND THAT THE SDK IS
        PROVIDED "AS IS" AND "AS AVAILABLE" WITHOUT WARRANTY OF ANY KIND FROM GOOGLE.

        10.2 YOUR USE OF THE SDK AND ANY MATERIAL DOWNLOADED OR OTHERWISE OBTAINED THROUGH THE USE OF THE SDK IS AT YOUR
        OWN DISCRETION AND RISK AND YOU ARE SOLELY RESPONSIBLE FOR ANY DAMAGE TO YOUR COMPUTER SYSTEM OR OTHER DEVICE OR
        LOSS OF DATA THAT RESULTS FROM SUCH USE.

        10.3 GOOGLE FURTHER EXPRESSLY DISCLAIMS ALL WARRANTIES AND CONDITIONS OF ANY KIND, WHETHER EXPRESS OR IMPLIED,
        INCLUDING, BUT NOT LIMITED TO THE IMPLIED WARRANTIES AND CONDITIONS OF MERCHANTABILITY, FITNESS FOR A PARTICULAR
        PURPOSE AND NON-INFRINGEMENT.


        11. LIMITATION OF LIABILITY

        11.1 YOU EXPRESSLY UNDERSTAND AND AGREE THAT GOOGLE, ITS SUBSIDIARIES AND AFFILIATES, AND ITS LICENSORS SHALL
        NOT BE LIABLE TO YOU UNDER ANY THEORY OF LIABILITY FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, CONSEQUENTIAL
        OR EXEMPLARY DAMAGES THAT MAY BE INCURRED BY YOU, INCLUDING ANY LOSS OF DATA, WHETHER OR NOT GOOGLE OR ITS
        REPRESENTATIVES HAVE BEEN ADVISED OF OR SHOULD HAVE BEEN AWARE OF THE POSSIBILITY OF ANY SUCH LOSSES ARISING.


        12. Indemnification

        12.1 To the maximum extent permitted by law, you agree to defend, indemnify and hold harmless Google, its
        affiliates and their respective directors, officers, employees and agents from and against any and all claims,
        actions, suits or proceedings, as well as any and all losses, liabilities, damages, costs and expenses
        (including reasonable attorneys fees) arising out of or accruing from (a) your use of the SDK, (b) any
        application you develop on the SDK that infringes any copyright, trademark, trade secret, trade dress, patent or
        other intellectual property right of any person or defames any person or violates their rights of publicity or
        privacy, and (c) any non-compliance by you with the License Agreement.


        13. Changes to the License Agreement

        13.1 Google may make changes to the License Agreement as it distributes new versions of the SDK. When these
        changes are made, Google will make a new version of the License Agreement available on the website where the SDK
        is made available.


        14. General Legal Terms

        14.1 The License Agreement constitutes the whole legal agreement between you and Google and governs your use of
        the SDK (excluding any services which Google may provide to you under a separate written agreement), and
        completely replaces any prior agreements between you and Google in relation to the SDK.

        14.2 You agree that if Google does not exercise or enforce any legal right or remedy which is contained in the
        License Agreement (or which Google has the benefit of under any applicable law), this will not be taken to be a
        formal waiver of Google's rights and that those rights or remedies will still be available to Google.

        14.3 If any court of law, having the jurisdiction to decide on this matter, rules that any provision of the
        License Agreement is invalid, then that provision will be removed from the License Agreement without affecting
        the rest of the License Agreement. The remaining provisions of the License Agreement will continue to be valid
        and enforceable.

        14.4 You acknowledge and agree that each member of the group of companies of which Google is the parent shall be
        third party beneficiaries to the License Agreement and that such other companies shall be entitled to directly
        enforce, and rely upon, any provision of the License Agreement that confers a benefit on (or rights in favor of)
        them. Other than this, no other person or company shall be third party beneficiaries to the License Agreement.

        14.5 EXPORT RESTRICTIONS. THE SDK IS SUBJECT TO UNITED STATES EXPORT LAWS AND REGULATIONS. YOU MUST COMPLY WITH
        ALL DOMESTIC AND INTERNATIONAL EXPORT LAWS AND REGULATIONS THAT APPLY TO THE SDK. THESE LAWS INCLUDE
        RESTRICTIONS ON DESTINATIONS, END USERS AND END USE.

        14.6 The rights granted in the License Agreement may not be assigned or transferred by either you or Google
        without the prior written approval of the other party. Neither you nor Google shall be permitted to delegate
        their responsibilities or obligations under the License Agreement without the prior written approval of the
        other party.

        14.7 The License Agreement, and your relationship with Google under the License Agreement, shall be governed by
        the laws of the State of California without regard to its conflict of laws provisions. You and Google agree to
        submit to the exclusive jurisdiction of the courts located within the county of Santa Clara, California to
        resolve any legal matter arising from the License Agreement. Notwithstanding this, you agree that Google shall
        still be allowed to apply for injunctive remedies (or an equivalent type of urgent legal relief) in any
        jurisdiction.


        January 16, 2019</license>
    <localPackage path="system-images;android-34;aosp_custom;arm64-v8a" obsolete="false">
        <type-details xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="ns13:sysImgDetailsType">
            <api-level>34</api-level>
            <base-extension>true</base-extension>
            <tag>
                <id>aosp_custom</id>
                <display>AOSP</display>
            </tag>
            <vendor>
                <id>d3xter</id>
                <display>d3xter</display>
            </vendor>
            <abi>arm64-v8a</abi>
        </type-details>
        <revision>
            <major>1</major>
        </revision>
        <display-name>AOSP ARM 64 v8a System Image</display-name>
    </localPackage>
</ns2:repository>
```

### Create AVD

Launch Android Studio, go to `Device Manager`, and open the AVD creation window. I chose the Pixel 7 hardware profile.

![](https://i.imgur.com/XkULSgI.png)

In the ARM Images tab, you’ll see the profile we just created. Go ahead and select it. You can adjust the AVD settings to your liking.

![](https://i.imgur.com/qEyUNaZ.png)

When you run the AVD, you should finally see the Android 14 image we built! Woohoo! Now it’s time to play around with AOSP and have fun customizing it!

<p align="center">
	<img src="https://i.imgur.com/wFAa8yP.png" width=300>
	<img src="https://i.imgur.com/XYq5vpT.png" width=300>
</p>

## Reference

* https://shumxin.github.io/2024/04/05/build-aosp-in-mackbook-pro-m3-max/
* https://grapeup.com/blog/android-automotive-os-14-is-out-build-your-own-emulator-from-scratch/
* https://medium.com/@imitiyaz125/build-aosp-emulator-image-fd4ae86a39cc
* https://stackoverflow.com/questions/77404675/aosp-emulator-images-not-running
* https://www.mobibrw.com/2024/39521
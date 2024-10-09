---
layout: post
title:  "Building an Android 14 (AOSP) Emulator on Apple M1"
date:   2024-10-09 16:00:00 -0000
categories: aosp android
---
![](https://i.imgur.com/lMkLN3v.png)

맥북을 사용중인데 어느날 문득 [AOSP(Android Open Source Project)](https://source.android.com/) 코드를 수정하여 다양한 버전의 AVD를 만들면서 이것저것 테스트하고 싶어졌습니다. 그런데 Google에서 2021년 6월 22일부터 macOS에서의 AOSP 빌드 지원을 공식적으로 종료해버렸어요. 그래서 AOSP 빌드가 가능한 방법이 없을까 알아보던 중에 orbstack을 발견하여 삽질을 시작했습니다!

## OrbStack

[OrbStack](https://orbstack.dev/)은 프로그램 자체가 엄청 가볍고 Docker 컨테이너와 Linux를 빠르고 쉽게 실행하도록 도와주는 macOS 전용 앱입니다. Docker Desktop의 느린 속도와 짜증나는 리소스 점유율에 질린 유저들이 OrbStack을 사용하고는 모두 찬사를 아끼지 않았었죠. 지금은 베타버전 종료 이후 유료모델을 도입했으나 개인 사용자들은 무료로 사용이 가능합니다.

## Why OrbStack

OrbStack에는 가상머신이라고 리눅스 머신을 생성하는 기능이 있습니다. 그러나 기존 Parallels, UTM 등과 다르게 macOS 호스트와 자연스럽게 통합된 방식으로 가상 머신 사용이 가능합니다. 정확하게는 비교 대상이 Parallels, UTM 보다는 윈도우의 WSL을 생각하면 될 것 같습니다. WSL 리눅스는 생성/삭제가 간편하고 윈도우의 파일시스템 등과 굉장히 결합이 잘 되어있죠? OrbStack 가상머신이 딱 그렇습니다. 파일시스템/네트워크 등 호스트 시스템과 밀접하게 결합되어 세팅 및 사용이 매우 쉽습니다. 부팅은 1초컷이죠.

여기에 더해 Rosetta을 이용한 x86_64/amd64 머신도 생성이 가능합니다. 여기서 이제 AOSP 빌드에 대한 힌트를 얻을 수 있습니다. 우리는 OrbStack의 가상머신 중 Rosetta를 이용한 우분투 리눅스(Intel)를 생성하여 해당 환경에서 AOSP 빌드를 진행할 것입니다.

![](https://i.imgur.com/4SFIYXP.png)

## Build Environment

빌드 환경은 아래와 같습니다.
* MacBook M1 Pro
* Sonoma 14.7
* Memory 16 GB
* OrbStack 1.7.5
* Ubuntu (Intel) 20.04 (Focal Fossa)

## Build Process

AOSP 빌드를 위한 기본적인 설정은 생략할게요. 이미 다양한 글에서 이 부분을 다루고 있기 때문에 바로 커멘드부터 시작하겠습니다. 그리고 Android 14의 안정화 버전인 Generic System Image 를 위한 android14-gsi 태그로 코드를 다운로드하도록 하겠습니다. `(다른 태그는.. 테스트안해봄)`

아! 여기서 <u>주의할 점은 절대로 호스트의 디스크를 사용하지 마세요</u>. diskutil로 case-sensitive 파티션을 생성해서 진행하면 된다는 이전 AOSP 빌드 관련 글들이 있으나 그렇게 진행했을때 특정 img 파일 빌드할때 계속 에러가 납니다. 그냥 OrbStack이 만들어준 리눅스 디스크 안에서 코드를 다운로드하고 빌드합시다.

```bash
$ cd ~/aosp/android14-gsi/
$ repo init --depth=1 -u https://android.googlesource.com/platform/manifest -b android14-gsi
$ repo sync -c --no-clone-bundle --no-tags -j2
```

여기까지 하면 AOSP 코드 다운로드가 완료됩니다. `-j` 옵션의 경우, 병렬 작업 수를 지정하는건데 메모리 16 GB 가지고 너무 많이 지정하면 메모리 부족으로 빌드에 실패합니다ㅠ Max 쓰시거나 램이 더 많으시면 더 늘리셔도 됩니다.

```bash
$ . build/envsetup.sh
$ lunch sdk_phone64_arm64-userdebug
$ m emu_img_zip -j 2
```

### Error 1: Insufficient Memory

일단 AVD를 위한 AOSP 빌드하는 전체 커멘드는 위와 같습니다. 그런데 이제 에러들이 툭툭 나오게 될거에요. 우선 이전에 언급한 메모리 부분. 32GB 이상을 사용하시는 분은 아마 별 문제 없을건데 저같은 16GB 메모리의 경우, 메모리 문제로 빌드에 실패하는 경우가 있습니다.

그래서 저는 swap 메모리를 늘려주었습니다. 저의 경우, 32G를 잡아주었습니다.
```bash
$ sudo swapon --show
NAME       TYPE       SIZE USED  PRIO
/dev/zram0 partition 11.7G 2.7G 32767
/dev/vdc   partition 1024M 2.7M     1
/swapfile  file        32G 6.3M    -2
```

### Error 2: lunch infinite loop

lunch 커맨드 실행시 무한루프에 빠집니다. 1시간.. 2시간... 4시간... 아무리 기다려도 다음으로 안 넘어갑니다. 그래서 CPU 사용률을 보면,

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

nsjail 이라는 프로세스가 CPU 100%를 먹고 변화가 없는 것을 볼 수 있어요. [nsjail](https://github.com/google/nsjail)은 Google에서 만든 리눅스를 위한 프로세스 격리 도구입니다. AOSP 빌드 시에 빌드 프로세스를 호스트 시스템과 격리하기 위하여 구글에서 도입한 절차로 예상됩니다. 즉, 없어도 됩니다.

```bash
$ mv prebuilds/build-tools/linux-x86/bin/nsjail prebuilds/build-tools/linux-x86/bin/nsjail.old
```

이렇게 하면 lunch 가 아주 빠르게 동작을 하게 되고 다음과 같은 경고 메시지가 보일 것입니다. 그럼 성공!

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

저 위에 에러 2개가 잡으면 빌드가 80-90% 까지 문제없이 진행됩니다. 그러다 갑자기 Dex2oat 실패 메시지가 나옵니다.

```bash
ERROR: Dex2oat failed to compile a boot image.It is likely that the boot classpath is inconsistent.Rebuild with ART_BOOT_IMAGE_EXTRA_ARGS="--runtime-arg -verbose:verifier" to see verification errors.
```

부트 이미지 내 DEX 파일을 Dex2oat를 통해 native machine code로 변환하는 과정에서 에러가 난 것으로 보입니다. 이 문제가 리눅스 커널 버전과 관련된다는 코멘트가 stackoverflow에 있으나 우리는 OrbStack(Linux kernel 6.10.12)을 통하여 리눅스를 사용하고 있어서 리눅스 커널을 다운그레이드(Linux kernel 5.x.x)하기 쉽지 않습니다.

그래서 우리는 아예 코드를 수정하여 Dex2oat를 비활성화시켜 부트 이미지에 대한 사전 최적화를 수행하지 않도록 하려고 합니다.

1. `build/core/board_config.mk`
```bash
 205 # Conditional to building on linux, as dex2oat currently does not work on darwin.
 206 ifeq ($(HOST_OS),linux)
 207   WITH_DEXPREOPT := false # true -> false 변경
 208 endif
```
2. `build/core/dex_preopt_config.mk`
```bash
 71   # Non eng linux builds must have preopt enabled so that system server doesn't run as interpreter
 72   # 전부 주석 처리
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

위 3개의 에러를 잡아주면 드디어 M1 에서 AVD를 위한 AOSP 빌드가 완료됩니다. 다음 메시지에는 6시간 걸렸다고 나오지만.. 실제로는 위 에러때문에 이어서 빌드하고.. 이어서 빌드하고.. 다 합치면 2일 정도 걸린것같네요? 맥북 사양이 좋으면 더 빠를 것 같아요.

```bash
[ 99% 12962/12964] Create system-qemu.img now
removing out/target/product/emulator64_arm64/system-qemu.img.qcow2
out/host/linux-x86/bin/sgdisk --clear out/target/product/emulator64_arm64/system-qemu.img
[100% 12964/12964] Package: out/target/product/emulator64_arm64/sdk-repo-linux-system-images-eng.d3xter.zip

#### build completed successfully (06:08:40 (hh:mm:ss)) ####
```

완료되면 AVD 생성을 위해 빌드된 이미지를 sdk 경로(예, /Source/Android/sdk/)에 넣어줘야합니다.

```bash
$ mkdir -p /Source/Android/sdk/system-images/android-34/aosp_custom/ && unzip -o out/target/product/emulator64_arm64/sdk-repo-linux-system-images-eng.d3xter.zip -d /Source/Android/sdk/system-images/android-34/aosp_custom/
```

빌드된 압축파일을 풀어주고 이미지를 확인해보면 다음과 같습니다.

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

다른 에뮬레이터 이미지 경로에 가면 `package.xml`이라는 파일이 있습니다. 해당 파일은 에뮬레이터 이미지 프로파일 파일로 Android Studio 에서 해당 파일을 기반으로 이미지를 인식합니다. 저는 android-34 > google_api_playstore 이미지의 파일을 복사해서 수정하였습니다. localPackage 태그의 path 값은 압축해제한 경로와 반드시 일치해야 합니다.

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

Android Studio 을 실행하고 `Device Manager`을 실행하여 일반적인 AVD 생성 창을 불러옵니다. 저는 Pixel 7 하드웨어 프로필을 선택하겠습니다.

![](https://i.imgur.com/XkULSgI.png)

ARM Images 탭에 가면 우리가 만들어준 프로파일을 기반으로 한 이미지가 표시되는 것을 확인할 수 있습니다. 가볍게 선택하시고 넘어가주세요. 다음 AVD 설정은 입맛에 맞게 설정해주세요.

![](https://i.imgur.com/qEyUNaZ.png)

생성된 AVD를 실행하면 드디어 빌드된 안드로이드가 나타나는 것을 볼 수 있어요! 안드로이드14 만세!!
이제 AOSP 코드 이것저것 만져봅시다~
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
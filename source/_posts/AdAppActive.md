---
title: A Short Analysis of AdAppActive
date: 2017-09-22 15:52:54
tags: RE
---

During my random browsing of PPHelper, China's largest "assistant" that provids download of cracked softwares as well as serving
as a 2nd AppStore itself, we found one of the games contains a strange Adware not seen in its AppStore Version.

Overall it's a pretty boring case since the native calls are not obfuscated
<!-- more -->
[Sample](AdAppActive.framework/AdAppActive)
Running `codesign -dvvv` yields the following information:
```
Identifier=dujj.com.AdAppActive
Format=bundle with Mach-O universal (armv7 arm64)
CodeDirectory v=20200 size=1424 flags=0x0(none) hashes=64+3 location=embedded
OSPlatform=37
OSSDKVersion=590592
OSVersionMin=458752
Hash type=sha1 size=20
CandidateCDHash sha1=84318837fcc69584ccf6ab194011e707691e8240
Hash choices=sha1
Page size=4096
CDHash=84318837fcc69584ccf6ab194011e707691e8240
Signature size=4378
Authority=iPhone Distribution: Shandong Meibang Information Technology Co., Ltd.
Authority=Apple Worldwide Developer Relations Certification Authority
Authority=Apple Root CA
Signed Time=2017年1月12日 上午2:37:38
Info.plist entries=21
TeamIdentifier=KQE7R93SMR
Sealed Resources version=2 rules=12 files=2
Internal requirements count=1 size=208
```

Apparently a cracked app will have its CSBlobs removed so now we can be sure that this application has been resigned, which is strange considering this is published on "Applications for Jailbroken Devices"

Running `otool -l` against the main binary also yields a new load command:
```
Load command 57
          cmd LC_LOAD_DYLIB
      cmdsize 80
         name @executable_path/AdAppActive.framework/AdAppActive (offset 24)
   time stamp 2 Thu Jan  1 01:00:02 1970
      current version 1.0.0
compatibility version 1.0.0
```
# The Binary Itself
The main functionality of AdAppActive took place in ``-[PPNiubilitySimpleFactory createClientInfo]``, starting from the EntryPoint and in turn calls ``[TRPPNiubilityManager share]``

Inside ``-[PPNiubilitySimpleFactory createClientInfo]``, it calls:  
- ``[PPNiubilityConfiguration sharedConfiguration]``
- ``[UIDevice isJailBreak]``
- ``[PPReachability reachabilityForLocalWiFi]``

to collect various device information and upload them to ``http://applog.uc.cn:9081/collect``

# Dangerous Behaviours
## UDID
  ``-[PPNiubilitySimpleFactory createClientInfo]`` calls ``+[UnitlTool uuid]`` which in turn abuses ``MGCopyAnswer`` in libMobileGestalt to upload the UDID of the device, although theoratically this private API is no longer usable without proper entitlements, and our samples seems to be missing those entitlements, it only contains:
```
  <?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
        <key>application-identifier</key>
        <string>K5X9BM5TQ2.com.yinhan.SpaceHunterPP.IPad-260064-6</string>
        <key>com.apple.developer.team-identifier</key>
        <string>K5X9BM5TQ2</string>
        <key>get-task-allow</key>
        <false/>
        <key>keychain-access-groups</key>
        <array>
                <string>K5X9BM5TQ2.com.yinhan.SpaceHunterPP.IPad-260064-6</string>
        </array>
</dict>
</plist>
```
![](MG.png)
## IOKit
  Various methods in the class ``UIDeviceAdditions`` seems to be abusing IOKit and sysctl to fetch ECID/SerialNumber/MAC Address/Platform.
  This is also no longer usable in sandbox since around iOS 9.3.X
![](IOKit.png)

## Various Thoughts
  Judging by:
  - The history of this kind of "assistants" and their commercial mode
  - The sample is marking its ``promotion channel/APP ID`` to the server
  - The original developer hasn't started a lawsuit yet
  - I've also examined another 12 games on PPHelper and none of them contains AdAppActive
  - IPAs of this exact game found elsewhere doesn't contain AdAppActive as well

  I think it's safe to assume the developer is involved in this mess. I've already emailed the developer regarding this

## TODO
  - Dynamic binary instrumention
  - Report this to AAPL's product security team to invalidate KQE7R93SMR

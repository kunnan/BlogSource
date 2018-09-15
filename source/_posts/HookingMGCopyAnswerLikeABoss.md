---
layout: default
title:  "Hooking MGCopyAnswer Like A Boss"
date:   2017-06-26 21:36:33 +0800
tags: RE
---

  This is a note following [my post](/2017/04/05/Random-Thoughts-On-Hooking-Short-Functions/) back in april regarding short function hooking.
  <!-- more -->

  In the aforementioned post we talked about *Analyzing assembly* and its usage scenario.
  Here in China the advertisement/analytics SDK developers abuses various private iOS APIs to uniquely identify a device.
  And hackers bypass these checks to fake data in order to squeeze every single penny of investors.
  (Actually it's much more than that but this is not our main topic)

  One of the most abused API is `MGCopyAnswer` in libMobileGestalt, but directly hooking it will instantly crash the process with an `invalid instruction`.
  However if you've ever analyzed the library in disassemblers you would notice that this function is actually using another private symbol-less routine to perform its tasks and this routine is always the first branching instruction in `MGCopyAnswer`.

  ![](Hopper.png)

  As such, a pretty obvious method would be hooking this subroutine, as mentioned in my previous post.But how do you know the address if it doesn't have a symbol to begin with? Relative offset? Possible, but maybe not stable.
  Fortunately, we have [Capstone Engine](http://www.capstone-engine.org), which is a powerful disassembler based on LLVM's MC to save the day.  
  Now we have the "signature"(First B in MGCopyAnswer),and a disassembler. All we need is to disassemble `MGCopyAnswer` and locate the subroutine for hooking. A sample PoC code can be found at <https://github.com/Naville/MobileGestaltHooking/>
  ![](Log.png)
  ![](Log2.png)


# NOTE
  Apparently this PoC code is not using any architecture detection and it currently works in AArch64 only.
  You might want to add a few marco definition checks in production environment and adjust `cs_open`'s arguments accordingly

  @Ouroboros mentioned that `insn[j].id == ARM64_INS_BL` might be a better solution than `strcmp()`

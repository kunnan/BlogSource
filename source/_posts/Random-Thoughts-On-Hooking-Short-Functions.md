---
layout: default
title:  "Random Thoughts On Hooking Short Functions"
date:   2017-04-04 18:10:17 +0000
tags: RE
---


  For languages that doesn't have a reflection mechanism that benefits developers (and hackers), (I'm looking at you Objective-C) function hooking inevitably involves in inline hooking, or to be more precise, assembly patching. Unfortunately not every C function has enough length for us to do our dirty work.   
  This post is my $0.02 regarding this issue. Theoretically following analysis should hold true on most platforms, although the implementation might differ

<!-- more -->

### Hook(Patch)ing Lazy Symbol table
  This is the simplest solution, by replacing LazySymbol pointers using tools like fishhook.
  * `+` Easy to use.
  * `+` Not restricted by target function length.
  * `+` Not intrusive. Thus won't trigger security mechanisms
  * `-` Only works to lazy symbols linked at runtime. Won't work for dynamically resolved symbols using functions like dlsym() or direct function call from the same binary/library
  * `-` Requires thorough parsing of executable. Hard to implement


### Patching dlsym()
  Not much worth mentioning. On iOS this is a  decent replacement for certain functions that is too short to patch and usually resolved at runtime. (**COUGH COUGH**)

### Analyzing assembly
  Most functions that are too short to use inline hook are barely a wrapper around some other functions that carries out the actual task. By analyzing assembly from function prologue and search for **JUMP** instructions we can locate the inner function that is likely long enough for inline hook. Note that this process can be automated using open-source disassembling frameworks like [capstone-engine](http://www.capstone-engine.org/)
  * `+` (Probably the only solution that)Works for short functions
  * `-` Relies on the function matches the characteristics described above, there are quite some functions don't match those. (We will describe below)
  * `-` The binary/library/whatever container will have to contain a huge disassembling library
  * `-` False-Positives from "effort-less" disassembling, or requires too much effort if the function is analyzed by a human

### Patching The Kernel
  There is a bunch of C function in libC that are barebone wrappers around syscalls. For (**RARE**) situations like this (and LazySymbol Patching Does't Work), the only feasible solution would be patching the sysent table in the kernel

  * `+` Accurate function hooking, as all other high-level wrappers are still using syscalls under-the-hood
  * `?` Can intercept corresponding call in all processes
  * `-` SUPER Hard to implement.
  * `-` Requires Kernel Access.
  * `-` Affects system performance.
  * `-` (Probably) unstable

### Patch Calling Instructions(Purely Theoretical)
  Well there are unfortunate scenarios that none of the above is feasible. The last possibility would be searching Calling Instructions to our target and modify those instructions to jump to our proxy function.
  * `+` For certain usage scenarios this can be statically applied and thus won't trigger various protection mechanism.
  * `+` You probably don't have a choice
  * `-` Requires full binary analyzing && disassembling && patching. Nightmare
  * `-` For each new __TEXT loaded, this process has to be repeated. *HUGE* performance impact

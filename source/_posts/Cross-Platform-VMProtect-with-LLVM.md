---
title: Cross-Platform VMProtect with LLVM
date: 2018-08-08 16:24:45
tags: LLVM
---
I finally had enough fun with building LLVM Transform Obfuscation Passes and decided to build a VMProtect-Like obfuscation mechanism.

<!-- more -->
Below is the emulated CPU's control flow graph. Extra protection disabled, otherwise you won't even see this graph.  
![](VMP.png)


## Current Implementations

### Assembly Interpreter
- Mostly used design pattern
- Takes a large amount of time to add support for a new platform
- Not Fast Enough
- Hard to maintain/debug

### LLVM IR Interpreter
- Similar to Assembly Interpreter, expect more trouble to solve. For example using reg2mem to resolve PhiNodes
- Bridging with native code. Global Variable/Local Memory Address mapping
- var-arg support
- Emulating stack?

### High-Level Language Interpreter
- Not safe enough(Dump out string and it's over.)
- JIT implementation requires a full compile chain available at runtime.
- Raw Interpreter Implementation needs to handle structure coercion in order to properly support foreign function calling[^1]



## Hikari's implementation

In order to achieve cross platform with minimum effort, Hikari's VM Protection emulates a specialized CPU with full fetch-decode-execute cycle.

### Supporting LLVM Instructions

- Cast Instructions are essentially no-op
- CallInsts are handled by using LLVM to generate proper assembly CallSites
- BinaryOperators are obviously easy to handle
- Invoke Instructions are lowered to CallInsts (Essentially unsupported atm)
- Load/Stores are properly bitcasted
- Everything else is left as-is

The arguments are passed as pointers and each op's handling block is responsible for specializing itself to handle each variation of the arguments.

### Finding Candidate Instructions

Since our VM implementation is pretty dumb and doesn't support all the instructions(yet), we need to find instruction sequences to patch and replace. This algorithm is pretty straightforward at the moment. Do note, however, that we need to run various lowering passes to make sure:  
- PhiNodes are properly lowered
- Intrinsics are either stripped out or lowered into platform specific library calls
- ConstantExprs are properly resolved
- etc...


The actual source code won't be open-source but you should be able to craft yourself one after reading this.


Zhang






[^1]: External Reading: [DragonFFI: FFI/JIT for the C language using Clang/LLVM](http://blog.llvm.org/2018/03/dragonffi-ffijit-for-c-language-using.html)

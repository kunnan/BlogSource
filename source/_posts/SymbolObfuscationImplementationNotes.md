---
title: FunctionCallSite/Symbol Obfuscation Implementation Notes
date: 2017-12-21 20:16:59
tags: LLVM
---

By design,Function CallSite Obfuscation aims to replace direct call/invoke instructions with ``dlsym()`` equivalents at IR Level.
Symbol Obfuscation also suffers from the exact same issues
<!-- more -->

## Source of Symbol
  We can not accurately determine a symbol's origin at IR Level.For example, the Module might refers to a symbol in a third party library.In which case there is a chance that the symbol is stripped out in Release build and thus crash our compiling process.  
  A possible approach would be a after-install analysis of system libraries(dyld cache on Darwin/Apple platforms) plus optional arguments at compile time.

## Messed-up Symbols
  Again, on darwin platforms, a bunch of libSystem functions are marked as ``__DARWIN_ALIAS_C`` which would screw-up the symbol in various ways.  
  A possible approach, since these kind of modification has a pattern, would be stripping out prefix/suffixs in our pass.
## Insecure Implementation of system symbol searching mechanism
  One simple runtime code injection plus some decompiler script is all it takes to defeat our protection.Implementing a custom-baked replacement and mark it as inline instead of using platform implementation would be (kinda) safer.
  However this is not possible for ObjC since we can't ship a custom implemented ObjC Runtime.(Or can we?)


T.B.C

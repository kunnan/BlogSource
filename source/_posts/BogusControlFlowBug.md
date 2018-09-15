---
layout: default
title:  "Bug in Obfuscator-LLVM's Bogus Control Flow"
date:   2017-12-27 21:36:33 +0800
tags: LLVM
---
@Ouroburos sent me a [IR](https://gist.github.com/Naville/37c61f653be529faceeb875554790c10) which crashes Obfuscator-LLVM. So the investigation begin.
<!-- more -->
Below is a generated Control-Flow-Graph for the affected function.  
![](OriginalCFG.png)

However running bcf on this IR using ``bin/opt -boguscf -bcf_prob=100`` won't pass IRVerifier, yielding the following message:  

```
The unwind destination does not have an exception handling instruction!
  %10 = invoke %1* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %1* (i8*, i8*)*)(i8* %7, i8* %9) #3
          to label %11 unwind label %16
Block containing LandingPadInst must be jumped to only by the unwind edge of an invoke.
  %25 = landingpad { i8*, i32 }
          cleanup
Block containing LandingPadInst must be jumped to only by the unwind edge of an invoke.
  %34 = landingpad { i8*, i32 }
          cleanup
LLVM ERROR: Broken function found, compilation aborted!
```
Now that if you have enough knowledge of LLVM you will surely recognize this error as a InvokeInst going wild and jumping to illegal destination block. First, let's bootup opt again and dump the CFG of the obfuscated IR, in order to do that we need to disable IR verifier first by using ``-disable-verify``. The full command would be:

```
bin/opt -boguscf -bcf_prob=100 -S original.ll -disable-verify -o Obfuscated.ll  
opt -dot-cfg Obfuscated.ll -disable-verify  
dot -Tpng callgraph.dot -o cfg.png
```
Yielding the following image.I did the annotation part for you : )  

![](ObfuscatedCFG.png)   

Let's explain this CFG a little bit more.Among other things, the root cause of the issue is the basicblock marked as ``originalBBpart2``, which I've circled out with blue. If you remember our CFG for the un-obfuscated version, this block is the terminator of the first BB that got seperated out.Below is the code responsible for this extracted from Obfuscator-LLVM ``78e056391160dab834682f7aaabe3a7a2afadcf6``

```cpp
// Split at this point (we only want the terminator in the second part)
Twine * var5 = new Twine("originalBBpart2");
BasicBlock * originalBBpart2 = originalBB->splitBasicBlock(--i , *var5);
DEBUG_WITH_TYPE("gen", errs() << "bcf: Terminator part of the original basic block"<< " is isolated\n");
// the first part go either on the return statement or on the begining
// of the altered block.. So we erase the terminator created when splitting.
originalBB->getTerminator()->eraseFromParent();
// We add at the end a new always true condition
Twine * var6 = new Twine("condition2");
FCmpInst * condition2 = new FCmpInst(*originalBB, CmpInst::FCMP_TRUE , LHS, RHS, *var6);
BranchInst::Create(originalBBpart2, alteredBB, (Value *)condition2, originalBB);
DEBUG_WITH_TYPE("gen", errs() << "bcf: Terminator original basic block: ok\n");
DEBUG_WITH_TYPE("gen", errs() << "bcf: End of addBogusFlow().\n");
```

Initially, this piece of code does seem to be correct. However, in order to loop though all BasicBlocks,the author used a ``set<BasicBlock*>`` in ``void bogus(Function &F)`` following codes that insert all BBs from this function into the set. That's how things went horribly wrong.   
At later stage in the obfuscation, BasicBlock ``%16`` also got obfuscated into ``%48``. Remember the BCF uses ``BasicBlock::getFirstNonPHIOrDbgOrLifetime()`` to locate the split point. And that, unfortunately, means our poor ``LandingPad`` also got splited to ``originalBB6``.   
Now ``%26`` 's terminating ``InvokeInst`` is using a BB that doesn't have LandingPad. And the original LandingPad BasicBlock is now referenced by an unconditional BranchInst (aka the one after ``%56``), this newly generated IR will surely fail to pass IRVerifier and thus crash the whole compiling process.

# Potential fix
The easist solution would be inserting codes checking if the BB is a landing pad before obfuscating. However I do feel like constant based branching condition is far from effective so here I purpose a new solution that generates random mathematical expressions as LHS and use ``tinyexpr`` to evaluate the RHS

---
title: PhiNode in LLVM
date: 2018-06-04 21:18:23
tags: LLVM
---

PhiNode is one of the most confusing concepts in compiler intermediate languages. This post attempts to introduces its concept and various fun aspects of it.
<!-- more -->
# Basics
SSA form, which stands for ``Static Single Assignment``. Basically it means each variable is assigned exactly once. This is related to what we'll be talking about today, as we will shortly see.
PhiNode refers to a special Instruction/Value/Node in compiler IR that changes its value depending on the control flow, which is usually used when the diverged control flow of a program merges back together.

# Main Dish
Now we can craft a simple function to demostrate PhiNode:   

````
int foooooo(int bar){
  int i=0;
  if(bar%2==0){
    i=1; //BasicBlock 1
  }
  else{
    i=2; //BasicBlock 2
  }
  return i;
}
````
It's pretty obvious to see that in this case the variable ``i`` is assigned twice (``i=1 and i=2``), which breaks the definition of SSA (assigned **exactly once**). (However real world compilers won't generate PhiNode for this piece of code because here ``i`` is allocated on stack as a temporary variables and BB1&BB2 are simply storing into it.)
That's when PhiNode kicks in. For example, we could rewrite ``foooooo`` in the following form:

````
 if(bar%2==0){
    //BasicBlock1
  }
  else{
    //BasicBlock2
  }
  int i=Phi([BasicBlock1,1],[BasicBlock2,2])
  return i;
````

However, since no real world processors supports PhiNode, the compiler must generates code without PhiNode semantics, and this is where PhiNode Resolving (otherwise known as SSA Destruction) Process kicks in.   
At LLVM IR level, this is achieved by using [DemotePHIToStack](https://github.com/llvm-mirror/llvm/blob/master/lib/Transforms/Utils/DemoteRegToStack.cpp#L109) or its wrapper Pass [reg2mem](https://github.com/llvm-mirror/llvm/blob/master/lib/Transforms/Scalar/Reg2Mem.cpp). These two passes are both doing one similar thing, as we will shortly see. One thing to keep in mind is that SSA form IR could be easily imagined as running on a virtual CPU with infinite registers and each of the SSA values corresponds to one of the virtual registers.

# Rev(s)olver
As ``DemotePHIToStack`` and ``reg2mem`` 's name suggests, they convert the SSA values into a memory location. But what does that mean? Essentially, our simple example could now be represented in the following C-style pseudo-code:   

````
 int* i=malloc(sizeof(int));
 if(bar%2==0){
	Store Value 1 to the memory location pointed to by i
  }
  else{
    Store Value 2 to the memory location pointed to by i
  }
  int j=load from the address pointed by i
  return j;
````
This is essentially achieved by inserting ``alloca``, ``store`` and ``load`` LLVM Assembly instructions at places.

Other possible places for PhiResolving includes the backend, or to be precise, the Register Allocation stage, while SSA values are assigned the same register if possible, or insert move instructions to the end of affected BasicBlock if possible.

To revert this process, there are [PromoteMemoryToRegister](https://github.com/llvm-mirror/llvm/blob/master/lib/Transforms/Utils/PromoteMemoryToRegister.cpp) or its wrapper class [mem2reg](https://github.com/llvm-mirror/llvm/blob/master/lib/Transforms/Utils/Mem2Reg.cpp) for this.

# External Reading
- [Static single assignment form](https://en.wikipedia.org/wiki/Static_single_assignment_form)
- [Static Single Assignment Book](http://ssabook.gforge.inria.fr/latest/book.pdf)

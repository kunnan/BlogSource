---
layout: default
title:  "LLVM Hacking-0x2 Manipulating GlobalVariable and Obfuscate ObjC Control Flow"
date:   2017-06-02 07:34:18 +0000
tags: LLVM
---


  This is part of my planned LLVM-Hacking blog posts, intended for new compiler hobbists like myself
  <del>PoC codes are available at <https://github.com/Naville/LLVM-Hacking-Tut-Source></del>.
  The sample codes have been removed due to inaccuracy and stuff. A more robust Obfuscator implementation is open-source at [HikariObfuscator/Hikari](https://github.com/HikariObfuscator/Hikari)
  <!-- more -->

In our second chapter I'm gonna demostrate some basic usage of `GlobalVariable` and using these techniques to build a pass that will obfuscate ObjC control flow.

In order to achieve this, we'd like to hide `SEL` and `Class` references. First let's compile some sample ObjC codes into LLVM IR to have a general idea about what happened under the hood.

```cpp
#import <Foundation/Foundation.h>
int main(){
  [[NSMutableData new] increaseLengthBy:20];
  return 0;
}


```

generates the following LLVM IR:

```
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.12.0"

%0 = type opaque
%struct._class_t = type { %struct._class_t*, %struct._class_t*, %struct._objc_cache*, i8* (i8*, i8*)**, %struct._class_ro_t* }
%struct._objc_cache = type opaque
%struct._class_ro_t = type { i32, i32, i32, i8*, i8*, %struct.__method_list_t*, %struct._objc_protocol_list*, %struct._ivar_list_t*, i8*, %struct._prop_list_t* }
%struct.__method_list_t = type { i32, i32, [0 x %struct._objc_method] }
%struct._objc_method = type { i8*, i8*, i8* }
%struct._objc_protocol_list = type { i64, [0 x %struct._protocol_t*] }
%struct._protocol_t = type { i8*, i8*, %struct._objc_protocol_list*, %struct.__method_list_t*, %struct.__method_list_t*, %struct.__method_list_t*, %struct.__method_list_t*, %struct._prop_list_t*, i32, i32, i8**, i8*, %struct._prop_list_t* }
%struct._ivar_list_t = type { i32, i32, [0 x %struct._ivar_t] }
%struct._ivar_t = type { i64*, i8*, i8*, i32, i32 }
%struct._prop_list_t = type { i32, i32, [0 x %struct._prop_t] }
%struct._prop_t = type { i8*, i8* }

@"OBJC_CLASS_$_NSMutableData" = external global %struct._class_t
@"OBJC_CLASSLIST_REFERENCES_$_" = private global %struct._class_t* @"OBJC_CLASS_$_NSMutableData", section "__DATA, __objc_classrefs, regular, no_dead_strip", align 8
@OBJC_METH_VAR_NAME_ = private global [4 x i8] c"new\00", section "__TEXT,__objc_methname,cstring_literals", align 1
@OBJC_SELECTOR_REFERENCES_ = private externally_initialized global i8* getelementptr inbounds ([4 x i8], [4 x i8]* @OBJC_METH_VAR_NAME_, i32 0, i32 0), section "__DATA, __objc_selrefs, literal_pointers, no_dead_strip", align 8
@OBJC_METH_VAR_NAME_.1 = private global [18 x i8] c"increaseLengthBy:\00", section "__TEXT,__objc_methname,cstring_literals", align 1
@OBJC_SELECTOR_REFERENCES_.2 = private externally_initialized global i8* getelementptr inbounds ([18 x i8], [18 x i8]* @OBJC_METH_VAR_NAME_.1, i32 0, i32 0), section "__DATA, __objc_selrefs, literal_pointers, no_dead_strip", align 8
@llvm.compiler.used = appending global [5 x i8*] [i8* bitcast (%struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_" to i8*), i8* getelementptr inbounds ([4 x i8], [4 x i8]* @OBJC_METH_VAR_NAME_, i32 0, i32 0), i8* bitcast (i8** @OBJC_SELECTOR_REFERENCES_ to i8*), i8* getelementptr inbounds ([18 x i8], [18 x i8]* @OBJC_METH_VAR_NAME_.1, i32 0, i32 0), i8* bitcast (i8** @OBJC_SELECTOR_REFERENCES_.2 to i8*)], section "llvm.metadata"

; Function Attrs: norecurse ssp uwtable
define i32 @main() #0 {
  %1 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  %2 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_", align 8
  %3 = load i8*, i8** @OBJC_SELECTOR_REFERENCES_, align 8, !invariant.load !7
  %4 = bitcast %struct._class_t* %2 to i8*
  %5 = call i8* bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to i8* (i8*, i8*)*)(i8* %4, i8* %3)
  %6 = bitcast i8* %5 to %0*
  %7 = load i8*, i8** @OBJC_SELECTOR_REFERENCES_.2, align 8, !invariant.load !7
  %8 = bitcast %0* %6 to i8*
  call void bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to void (i8*, i8*, i64)*)(i8* %8, i8* %7, i64 20)
  ret i32 0
}

; Function Attrs: nonlazybind
declare i8* @objc_msgSend(i8*, i8*, ...) #1

attributes #0 = { norecurse ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { nonlazybind }

!llvm.module.flags = !{!0, !1, !2, !3, !4, !5}
!llvm.ident = !{!6}

!0 = !{i32 1, !"Objective-C Version", i32 2}
!1 = !{i32 1, !"Objective-C Image Info Version", i32 0}
!2 = !{i32 1, !"Objective-C Image Info Section", !"__DATA, __objc_imageinfo, regular, no_dead_strip"}
!3 = !{i32 4, !"Objective-C Garbage Collection", i32 0}
!4 = !{i32 1, !"Objective-C Class Properties", i32 64}
!5 = !{i32 1, !"PIC Level", i32 2}
!6 = !{!"Apple LLVM version 8.1.0 (clang-802.0.42)"}
!7 = !{}

```

Note the two `LoadInst` referencing `OBJC_CLASSLIST_REFERENCES_` and `OBJC_SELECTOR_REFERENCES_`.
Now let's walk away from LLVM Core for a while and focus on the frontend,`Clang`, you can find a unofficial web-friendly mirror here:  
<https://github.com/llvm-mirror/clang/blob/b0c092f298d809acb814934c0ef593104d633713/lib/CodeGen/CGObjCMac.cpp>

Search for `OBJC_CLASSLIST_REFERENCES` brings us to Line 7125, which shows GV `OBJC_CLASSLIST_REFERENCES` is constructed as:

```cpp
Entry = new llvm::GlobalVariable(CGM.getModule(), ObjCTypes.ClassnfABIPtrTy,
                                     false, llvm::GlobalValue::PrivateLinkage,
                                     ClassGV, "OBJC_CLASSLIST_REFERENCES_$_");
```

which corresponds to the following `GlobalVariable` constructor:

```cpp
GlobalVariable (Module &M, Type *Ty, bool isConstant, LinkageTypes Linkage,
Constant *Initializer, const Twine &Name="", GlobalVariable   *InsertBefore=nullptr,
ThreadLocalMode=NotThreadLocal, unsigned AddressSpace=0,   bool isExternallyInitialized=false)
```


Now we can write a simple LLVM Pass:

```cpp
...OMITTED...
      for(auto G=M.getGlobalList().begin();G!=M.getGlobalList().end();G++){
        GlobalVariable &GV=*G;
			errs()<<GV.getName()<<"\n";
        }
...OMITTED...
```

prints out:

```
llvm.compiler.used
OBJC_CLASSLIST_REFERENCES_$_  	
OBJC_METH_VAR_NAME_
OBJC_SELECTOR_REFERENCES_
OBJC_METH_VAR_NAME_.1
OBJC_SELECTOR_REFERENCES_.2
OBJC_CLASS_$_NSMutableData
OBJC_METACLASS_$_Utils
OBJC_METACLASS_$_NSObject
_objc_empty_cache
l_OBJC_METACLASS_RO_$_Utils
OBJC_CLASS_NAME_
l_OBJC_$_CLASS_METHODS_Utils
OBJC_METH_VAR_NAME_.2
OBJC_METH_VAR_TYPE_
_unnamed_cfstring_
__CFConstantStringClassReference
.str
OBJC_CLASS_$_Utils
OBJC_CLASS_$_NSObject
l_OBJC_CLASS_RO_$_Utils
OBJC_LABEL_CLASS_$
```

Now we can improve our sample pass a little bit to collect `OBJC_CLASSLIST_REFERENCES_` 's initializers:

```cpp
...OMITTED...
      for(auto G=M.getGlobalList().begin();G!=M.getGlobalList().end();G++){
        GlobalVariable &GV=*G;
			if (GV.getName().str().find("OBJC_CLASSLIST_REFERENCES")==0){
          string className=GV.getInitializer ()->getName();
          errs()<<className<<"\n";
        }
        }
...OMITTED...
```

yielding `OBJC_CLASS_$_NSMutableData` , strip out the prefix`className.replace(className.find("OBJC_CLASS_$_"),strlen("OBJC_CLASS_$_"),"");` and we now have the Class Name:`NSMutableData`

At last, we want to find all IRs referencing to our `GV` and replace them with a `objc_getClass()` call.
We can first obtain the original Instruction, through GV iteration,construct the Function,using IRBuilder to build CallInst and finally replace uses

```cpp
for(auto G=M.global_begin(); G!=M.global_end(); G++) {
        GlobalVariable &GV=*G;
        if (GV.getName().str().find("OBJC_CLASSLIST_REFERENCES")==0) {
                if(GV.hasInitializer()) {
                        string className=GV.getInitializer ()->getName();
                        className.replace(className.find("OBJC_CLASS_$_"),strlen("OBJC_CLASS_$_"),"");
                        for(auto U=GV.user_begin (); U!=GV.user_end(); U++) {
                                if (Instruction* I = dyn_cast<Instruction>(*U)) {
                                        IRBuilder<> builder(I);
                                        FunctionType *objc_getClass_type =FunctionType::get(I->getType(), {Type::getInt8PtrTy(M.getContext())}, false);
                                        Function *objc_getClass_Func = cast<Function>(M.getOrInsertFunction("objc_getClass", objc_getClass_type ) );
                                        Value* newClassName=builder.CreateGlobalStringPtr(StringRef(className));
                                        CallInst* CI=builder.CreateCall(objc_getClass_Func,{newClassName});
                                        I->replaceAllUsesWith(CI);
                                        I->eraseFromParent ();
                                }
                        }
                }
        }
}
```

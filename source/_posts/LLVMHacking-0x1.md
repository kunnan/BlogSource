---
layout: default
title:  "LLVM Hacking-0x1 Symbol Renaming"
date:   2017-06-01 20:18:27 +0000
tags: LLVM
---


  This is part of my planned LLVM-Hacking blog posts, intended for new compiler hobbists like myself
  <del>PoC codes are available at <https://github.com/Naville/LLVM-Hacking-Tut-Source></del>.
  The sample codes have been removed due to inaccuracy and stuff. A more robust Obfuscator implementation is open-source at [HikariObfuscator/Hikari](https://github.com/HikariObfuscator/Hikari)
  <!-- more -->
  To start from the easy ones, our first task is to rename function symbols to serve as a better obfuscator,to replace existing pre-processing based obfuscations.

  The core idea is pretty simple, we just create something inherits one of the pass classes and override the `runOnXXX` methods.
  Since functions (and the majority,if not all) LLVM IR components inherits from `llvm::Value`, we can use `llvm::Value::getName()` and `llvm::Value::setName()` for function name manipulating.

  Since LLVM Passes *DO NOT* preserve states between TU(Translation Unit)s, it becomes pretty obvious that we can obtain a one and only global TU Module by injecting our pass at LTO stage. If you have no idea about LTO, please refer to <http://llvm.org/docs/LinkTimeOptimization.html>

## How to register LTO Pass

You need to modify `llvm/Transforms/IPO/PassManagerBuild.cpp` and add your pass in `populateLTOPassManager`
Also add your transform module to `llvm/Transforms/IPO/LLVMBuild.txt`

## The pass itself

```cpp
/*
 *  LLVM SymbolObfuscation Pass
 *  Zhang@University of Glasgow
 *
 */
#include "llvm/IR/Instructions.h"
#include "llvm/Pass.h"
#include "llvm/IR/Module.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/Transforms/IPO/PassManagerBuilder.h"
#include "llvm/IR/LegacyPassManager.h"
#include "llvm/Transforms/Obfuscation/SymbolObfuscation.h"
#include <string>
#include <iostream>
#include <cstdlib>
using namespace llvm;
using namespace std;
static string obfcharacters="qwertyuiopasdfghjklzxcvbnm1234567890";
namespace llvm{
  struct SymbolObfuscation : public ModulePass {
    static char ID;
    SymbolObfuscation() : ModulePass(ID) {}
    string randomString(int length){
      string name;
      name.resize(length);
      for(int i=0;i<length;i++){
        name[i]=obfcharacters[rand()%(obfcharacters.length()+1)];
      }
      return name;
    }
    bool runOnModule(Module &M) override {
      //F.setName(randomString(16));
      errs()<<"Do not go gentle into that good night\n";
      //Remember each Module has an iterator of Functions
      //Each Function has an iterator of BasicBlocks
      //Each BB has an iterator of instructions
      for(Module::iterator Fun=M.begin();Fun!=M.end();Fun++){
        Function &F=*Fun;
        //Rename
        errs()<<"Renaming Function: "<<F.getName()<<"\n";
        F.setName(randomString(16));

      }
      return true;
    }
  };
  Pass * createSymbolObf() {return new SymbolObfuscation();}
}

char SymbolObfuscation::ID = 0;
```

Compile, it works, however external function calls are also messed up,so does `main` , so we add required checks and now our `runOnModule` looks like this:

Note that F.empty() is underlying checking if the function has any BasicBlock, an external function apparently don't. Thus we can skip renaming.

```cpp
    bool runOnModule(Module &M) override {
      //F.setName(randomString(16));
      errs()<<"Do not go gentle into that good night\n";
      for(Module::iterator Fun=M.begin();Fun!=M.end();Fun++){
        Function &F=*Fun;
        if (F.getName().str().compare("main")==0){
          errs()<<"Skipping main\n";
        }
        else if(F.empty()==false){
          //Rename
          errs()<<"Renaming Function: "<<F.getName()<<"\n";
          F.setName(randomString(16));
        }
        else{
          errs()<<"Skipping External Function: "<<F.getName()<<"\n";
        }
      }
      return true;
    }
```

Now it works fine on our test code `test.mm`. Yielding the following LLVM IR for our test case

```
source_filename = "test.mm"
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.12.0"

%0 = type opaque
%struct.__NSConstantString_tag = type { i32*, i32, i8*, i64 }
%struct._objc_cache = type opaque
%struct._class_t = type { %struct._class_t*, %struct._class_t*, %struct._objc_cache*, i8* (i8*, i8*)**, %struct._class_ro_t* }
%struct._class_ro_t = type { i32, i32, i32, i8*, i8*, %struct.__method_list_t*, %struct._objc_protocol_list*, %struct._ivar_list_t*, i8*, %struct._prop_list_t* }
%struct.__method_list_t = type { i32, i32, [0 x %struct._objc_method] }
%struct._objc_method = type { i8*, i8*, i8* }
%struct._objc_protocol_list = type { i64, [0 x %struct._protocol_t*] }
%struct._protocol_t = type { i8*, i8*, %struct._objc_protocol_list*, %struct.__method_list_t*, %struct.__method_list_t*, %struct.__method_list_t*, %struct.__method_list_t*, %struct._prop_list_t*, i32, i32, i8**, i8*, %struct._prop_list_t* }
%struct._ivar_list_t = type { i32, i32, [0 x %struct._ivar_t] }
%struct._ivar_t = type { i64*, i8*, i8*, i32, i32 }
%struct._prop_list_t = type { i32, i32, [0 x %struct._prop_t] }
%struct._prop_t = type { i8*, i8* }

@__CFConstantStringClassReference = external global [0 x i32]
@.str = private unnamed_addr constant [4 x i8] c"foo\00", section "__TEXT,__cstring,cstring_literals", align 1
@_unnamed_cfstring_ = private global %struct.__NSConstantString_tag { i32* getelementptr inbounds ([0 x i32], [0 x i32]* @__CFConstantStringClassReference, i32 0, i32 0), i32 1992, i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i64 3 }, section "__DATA,__cfstring", align 8
@_objc_empty_cache = external global %struct._objc_cache
@"OBJC_METACLASS_$_NSObject" = external global %struct._class_t
@OBJC_CLASS_NAME_ = private unnamed_addr constant [6 x i8] c"Utils\00", section "__TEXT,__objc_classname,cstring_literals", align 1
@OBJC_METH_VAR_NAME_ = private unnamed_addr constant [4 x i8] c"foo\00", section "__TEXT,__objc_methname,cstring_literals", align 1
@OBJC_METH_VAR_TYPE_ = private unnamed_addr constant [8 x i8] c"v16@0:8\00", section "__TEXT,__objc_methtype,cstring_literals", align 1
@"\01l_OBJC_$_CLASS_METHODS_Utils" = private global { i32, i32, [1 x %struct._objc_method] } { i32 24, i32 1, [1 x %struct._objc_method] [%struct._objc_method { i8* getelementptr inbounds ([4 x i8], [4 x i8]* @OBJC_METH_VAR_NAME_, i32 0, i32 0), i8* getelementptr inbounds ([8 x i8], [8 x i8]* @OBJC_METH_VAR_TYPE_, i32 0, i32 0), i8* bitcast (void (i8*, i8*)* @"pig7\003urf5gjv1cj" to i8*) }] }, section "__DATA, __objc_const", align 8
@"\01l_OBJC_METACLASS_RO_$_Utils" = private global %struct._class_ro_t { i32 1, i32 40, i32 40, i8* null, i8* getelementptr inbounds ([6 x i8], [6 x i8]* @OBJC_CLASS_NAME_, i32 0, i32 0), %struct.__method_list_t* bitcast ({ i32, i32, [1 x %struct._objc_method] }* @"\01l_OBJC_$_CLASS_METHODS_Utils" to %struct.__method_list_t*), %struct._objc_protocol_list* null, %struct._ivar_list_t* null, i8* null, %struct._prop_list_t* null }, section "__DATA, __objc_const", align 8
@"OBJC_METACLASS_$_Utils" = global %struct._class_t { %struct._class_t* @"OBJC_METACLASS_$_NSObject", %struct._class_t* @"OBJC_METACLASS_$_NSObject", %struct._objc_cache* @_objc_empty_cache, i8* (i8*, i8*)** null, %struct._class_ro_t* @"\01l_OBJC_METACLASS_RO_$_Utils" }, section "__DATA, __objc_data", align 8
@"OBJC_CLASS_$_NSObject" = external global %struct._class_t
@"\01l_OBJC_CLASS_RO_$_Utils" = private global %struct._class_ro_t { i32 0, i32 8, i32 8, i8* null, i8* getelementptr inbounds ([6 x i8], [6 x i8]* @OBJC_CLASS_NAME_, i32 0, i32 0), %struct.__method_list_t* null, %struct._objc_protocol_list* null, %struct._ivar_list_t* null, i8* null, %struct._prop_list_t* null }, section "__DATA, __objc_const", align 8
@"OBJC_CLASS_$_Utils" = global %struct._class_t { %struct._class_t* @"OBJC_METACLASS_$_Utils", %struct._class_t* @"OBJC_CLASS_$_NSObject", %struct._objc_cache* @_objc_empty_cache, i8* (i8*, i8*)** null, %struct._class_ro_t* @"\01l_OBJC_CLASS_RO_$_Utils" }, section "__DATA, __objc_data", align 8
@"OBJC_CLASSLIST_REFERENCES_$_" = private global %struct._class_t* @"OBJC_CLASS_$_Utils", section "__DATA, __objc_classrefs, regular, no_dead_strip", align 8
@OBJC_SELECTOR_REFERENCES_ = private externally_initialized global i8* getelementptr inbounds ([4 x i8], [4 x i8]* @OBJC_METH_VAR_NAME_, i32 0, i32 0), section "__DATA, __objc_selrefs, literal_pointers, no_dead_strip", align 8
@.str.1 = private unnamed_addr constant [4 x i8] c"CXX\00", section "__TEXT,__cstring,cstring_literals", align 1
@_unnamed_cfstring_.2 = private global %struct.__NSConstantString_tag { i32* getelementptr inbounds ([0 x i32], [0 x i32]* @__CFConstantStringClassReference, i32 0, i32 0), i32 1992, i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str.1, i32 0, i32 0), i64 3 }, section "__DATA,__cfstring", align 8
@"OBJC_LABEL_CLASS_$" = private global [1 x i8*] [i8* bitcast (%struct._class_t* @"OBJC_CLASS_$_Utils" to i8*)], section "__DATA, __objc_classlist, regular, no_dead_strip", align 8
@llvm.compiler.used = appending global [7 x i8*] [i8* getelementptr inbounds ([6 x i8], [6 x i8]* @OBJC_CLASS_NAME_, i32 0, i32 0), i8* getelementptr inbounds ([4 x i8], [4 x i8]* @OBJC_METH_VAR_NAME_, i32 0, i32 0), i8* getelementptr inbounds ([8 x i8], [8 x i8]* @OBJC_METH_VAR_TYPE_, i32 0, i32 0), i8* bitcast ({ i32, i32, [1 x %struct._objc_method] }* @"\01l_OBJC_$_CLASS_METHODS_Utils" to i8*), i8* bitcast (%struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_" to i8*), i8* bitcast (i8** @OBJC_SELECTOR_REFERENCES_ to i8*), i8* bitcast ([1 x i8*]* @"OBJC_LABEL_CLASS_$" to i8*)], section "llvm.metadata"

; Function Attrs: noinline optnone ssp uwtable
define internal void @"pig7\003urf5gjv1cj"(i8*, i8*) #0 {
  %3 = alloca i8*, align 8
  %4 = alloca i8*, align 8
  store i8* %0, i8** %3, align 8
  store i8* %1, i8** %4, align 8
  notail call void (%0*, ...) @NSLog(%0* bitcast (%struct.__NSConstantString_tag* @_unnamed_cfstring_ to %0*))
  ret void
}

declare void @NSLog(%0*, ...) #1

; Function Attrs: noinline norecurse optnone ssp uwtable
define i32 @main() #2 {
  %1 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  %2 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_", align 8
  %3 = load i8*, i8** @OBJC_SELECTOR_REFERENCES_, align 8, !invariant.load !8
  %4 = bitcast %struct._class_t* %2 to i8*
  call void bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to void (i8*, i8*)*)(i8* %4, i8* %3)
  call void @hluet9h72idwmo2f()
  ret i32 0
}

; Function Attrs: nonlazybind
declare i8* @objc_msgSend(i8*, i8*, ...) #3

; Function Attrs: noinline optnone ssp uwtable
define linkonce_odr void @hluet9h72idwmo2f() #0 align 2 {
  notail call void (%0*, ...) @NSLog(%0* bitcast (%struct.__NSConstantString_tag* @_unnamed_cfstring_.2 to %0*))
  ret void
}

attributes #0 = { noinline optnone ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #1 = { "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #2 = { noinline norecurse optnone ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
attributes #3 = { nonlazybind }

!llvm.module.flags = !{!0, !1, !2, !3, !4, !5, !6}
!llvm.ident = !{!7}

!0 = !{i32 1, !"Objective-C Version", i32 2}
!1 = !{i32 1, !"Objective-C Image Info Version", i32 0}
!2 = !{i32 1, !"Objective-C Image Info Section", !"__DATA, __objc_imageinfo, regular, no_dead_strip"}
!3 = !{i32 4, !"Objective-C Garbage Collection", i32 0}
!4 = !{i32 1, !"Objective-C Class Properties", i32 64}
!5 = !{i32 1, !"wchar_size", i32 4}
!6 = !{i32 7, !"PIC Level", i32 2}
!7 = !{!"clang version 5.0.0 (trunk 304373)"}
!8 = !{}

```

That being said, Objective-C class information can still be retrieved by parsing Objective-C Structures, the solution to that problem involves some other more advanced IR manipulating and will be discussed in the next post

Until next time.


**UPDATE:**

  Example has been update with ObjC ClassName/MethodName obfuscating codes.
  The key is to obtain `OBJC_METH_VAR_NAME_` and `OBJC_CLASS_NAME_` referenced by a bunch of other ObjC runtime structs and `replaceAllUsesWith`
  Note that this code **DOES NOT** separate external and local class/selectors because there is no way to directly do those.
  A reasonable approach would be parse and extract those info from unobfuscated symbols.


**UPDATE2:**
  Or, simply GV iterate through `l_OBJC_$_CLASS_METHODS_CLASSNAME` and parse/replace the structs,which contains pointer to string names, one could theoretically create new method/classname string Constants and replace those structs, which is guaranteed *NOT* to have the `SEL` / `ClassName` collision overhead

---
title: Bug in StringEncryption Pass Armariris
date: 2017-12-26 15:21:12
tags: LLVM
---

I'm preparing a new StringEncryption pass for Hikari, aim to fix various drawbacks from previous implementations.  
The most famous one among those implementations would be [GoSSIP-SJTU/Armariris](https://github.com/GoSSIP-SJTU/Armariris/)  
You can read the pass's full source [HERE](https://github.com/GoSSIP-SJTU/Armariris/blob/master/lib/Transforms/Obfuscation/StringObfuscation.cpp)  
<!-- more -->

# Drawbacks

- ``Armariris`` uses one single uint8_t as the encryption key across the full [ConstantAggregate](http://llvm.org/doxygen/classllvm_1_1ConstantAggregate.html) , <del>this is trival to bypass due to there are maximum 256 possible values for an ``uint8_t``, even brute force is a feasible option. Plus extracting a XOR instruction from assemblies shouldn't be too hard.</del>  
	We know that:
	- Last character of the unencrypted c string is 0x00
	- We can extract the last character of the encrypted string.
	- We know one single key is used for all characters
	- We know the "encryption" is XOR.

	Which means? : )

- ``Armariris`` inject its global decryptor at ``llvm.global_ctors``, for the C/ObjC/C++ guys reading this, this is equivalent to ``__attribute__((constructor))``.(LLVM also has a global destructor,which is ``llvm.global_dtors``, but that's not the topic of this post). This makes it trival to `dump` decrypted strings at main executable's entrypoint, where the system loader(``dyld`` on Darwin) has finished calling ctors and thus decrypting our string GVs

- ``Armariris`` is not handling ObjC Strings correctly.Below is GV filtering code extracted from ``Armariris``:  

```
std::string section(gv->getSection());
if (gv->isConstant() && gv->hasInitializer() &&   
isa<ConstantDataSequential>(gv->getInitializer())  
&& section != "llvm.metadata" &&  
section.find("__objc_methname") == std::string::npos) {  
```

This piece of code did filter out LLVM Metadata usage,Non-CDS usage and GVs that are stored in ``__objc_methname``.
However, consider the following code:  

```objc
@interface foo:NSObject  
+(void)foo;  
@end  
@implementation foo  
+(void)foo{  
  NSLog(@"FOOOO");  
}  
@end  
```

Which yields the following LLVM IR(Unrelated part stripped out):  

```
@.str = private unnamed_addr constant [6 x i8] c"FOOOO\00", section "__TEXT,__cstring,cstring_literals", align 1
@_unnamed_cfstring_ = private global %struct.__NSConstantString_tag { i32* getelementptr inbounds ([0 x i32], [0 x i32]* @__CFConstantStringClassReference, i32 0, i32 0), i32 1992, i8* getelementptr inbounds ([6 x i8], [6 x i8]* @.str, i32 0, i32 0), i64 5 }, section "__DATA,__cfstring", align 8  
@"OBJC_METACLASS_$_NSObject" = external global %struct._class_t
@OBJC_CLASS_NAME_ = private unnamed_addr constant [4 x i8] c"foo\00", section "__TEXT,__objc_classname,cstring_literals", align 1  
@OBJC_METH_VAR_NAME_ = private unnamed_addr constant [4 x i8] c"foo\00", section "__TEXT,__objc_methname,cstring_literals", align 1  
@OBJC_METH_VAR_TYPE_ = private unnamed_addr constant [8 x i8] c"v16@0:8\00", section "__TEXT,__objc_methtype,cstring_literals", align 1  
@"\01l_OBJC_$_CLASS_METHODS_foo" = private global { i32, i32, [1 x %struct._objc_method] } { i32 24, i32 1, [1 x %struct._objc_method] [%struct._objc_method { i8* getelementptr inbounds ([4 x i8], [4 x i8]* @OBJC_METH_VAR_NAME_, i32 0, i32 0), i8* getelementptr inbounds ([8 x i8], [8 x i8]* @OBJC_METH_VAR_TYPE_, i32 0, i32 0), i8* bitcast (void (i8*, i8*)* @"\01+[foo foo]" to i8*) }] }, section "__DATA, __objc_const", align 8  
@"\01l_OBJC_METACLASS_RO_$_foo" = private global %struct._class_ro_t { i32 1, i32 40, i32 40, i8* null, i8* getelementptr inbounds ([4 x i8], [4 x i8]* @OBJC_CLASS_NAME_, i32 0, i32 0), %struct.__method_list_t* bitcast ({ i32, i32, [1 x %struct._objc_method] }* @"\01l_OBJC_$_CLASS_METHODS_foo" to %struct.__method_list_t*), %struct._objc_protocol_list* null, %struct._ivar_list_t* null, i8* null, %struct._prop_list_t* null }, section "__DATA, __objc_const", align 8  
@"OBJC_METACLASS_$_foo" = global %struct._class_t { %struct._class_t* @"OBJC_METACLASS_$_NSObject", %struct._class_t* @"OBJC_METACLASS_$_NSObject", %struct._objc_cache* @_objc_empty_cache, i8* (i8*, i8*)** null, %struct._class_ro_t* @"\01l_OBJC_METACLASS_RO_$_foo" }, section "__DATA, __objc_data", align 8  
@"OBJC_CLASS_$_NSObject" = external global %struct._class_t  
@"\01l_OBJC_CLASS_RO_$_foo" = private global %struct._class_ro_t { i32 0, i32 8, i32 8, i8* null, i8* getelementptr inbounds ([4 x i8], [4 x i8]*   @OBJC_CLASS_NAME_, i32 0, i32 0), %struct.__method_list_t* null, %struct._objc_protocol_list* null, %struct._ivar_list_t* null, i8* null, %struct._prop_list_t* null }, section "__DATA, __objc_const", align 8  
@"OBJC_CLASS_$_foo" = global %struct._class_t { %struct._class_t* @"OBJC_METACLASS_$_foo", %struct._class_t* @"OBJC_CLASS_$_NSObject", %struct._objc_cache* @_objc_empty_cache, i8* (i8*, i8*)** null, %struct._class_ro_t* @"\01l_OBJC_CLASS_RO_$_foo" }, section "__DATA, __objc_data", align 8  
@"OBJC_LABEL_CLASS_$" = private global [1 x i8*] [i8* bitcast (%struct._class_t* @"OBJC_CLASS_$_foo" to i8*)], section "__DATA,__objc_classlist,regular,no_dead_strip", align 8  
@llvm.compiler.used = appending global [5 x i8*] [i8* getelementptr inbounds ([4 x i8], [4 x i8]* @OBJC_CLASS_NAME_, i32 0, i32 0), i8* getelementptr inbounds ([4 x i8], [4 x i8]* @OBJC_METH_VAR_NAME_, i32 0, i32 0), i8* getelementptr inbounds ([8 x i8], [8 x i8]* @OBJC_METH_VAR_TYPE_, i32 0, i32 0), i8* bitcast ({ i32, i32, [1 x %struct._objc_method] }* @"\01l_OBJC_$_CLASS_METHODS_foo" to i8*), i8* bitcast ([1 x i8*]* @"OBJC_LABEL_CLASS_$" to i8*)], section "llvm.metadata"  
```

Note

```
@OBJC_CLASS_NAME_ = private unnamed_addr constant [4 x i8] c"foo\00", section "__TEXT,__objc_classname,cstring_literals", align 1  
@OBJC_METH_VAR_TYPE_ = private unnamed_addr constant [8 x i8] c"v16@0:8\00", section "__TEXT,__objc_methtype,cstring_literals", align 1
```

The ``OBJC_CLASS_NAME_`` here is also handled by ``Armariris``.In other words, the class is registered in the ObjC Runtime with completely wrong names and types.The result could be catastrophic.  
Let's see [dyld.cpp](https://github.com/opensource-apple/dyld/blob/3f928f32597888c5eac6003b9199d972d49857b5/src/dyld.cpp). Note line 1090:  

```
void initializeMainExecutable()
{
	// record that we've reached this step
	gLinkContext.startedInitializingMainExecutable = true;

	// run initialzers for any inserted dylibs
	ImageLoader::InitializerTimingList initializerTimes[sAllImages.size()];
	initializerTimes[0].count = 0;
	const size_t rootCount = sImageRoots.size();
	if ( rootCount > 1 ) {
		for(size_t i=1; i < rootCount; ++i) {
			sImageRoots[i]->runInitializers(gLinkContext, initializerTimes[0]);
		}
	}

	// run initializers for main executable and everything it brings up
	sMainExecutable->runInitializers(gLinkContext, initializerTimes[0]);

	// register cxa_atexit() handler to run static terminators in all loaded images when this process exits
	if ( gLibSystemHelpers != NULL )
		(*gLibSystemHelpers->cxa_atexit)(&runAllStaticTerminators, NULL, NULL);

	// dump info if requested
	if ( sEnv.DYLD_PRINT_STATISTICS )
		ImageLoaderMachO::printStatistics((unsigned int)sAllImages.size(), initializerTimes[0]);
}
```
and the following comment right above ``static void addRootImage(ImageLoader* image)``

```
In order for register_func_for_add_image() callbacks to to be called bottom up,  
we need to maintain a list of root images. The main executable is usally the  
first root. Any images dynamically added are also roots (unless already  loaded).  
If DYLD_INSERT_LIBRARIES is used, those libraries are first.
```

In other words, dyld run initializers for libraries first, then the main executable, that means when our binary is starting up, the string constants like ObjC's class name are still encrypted, which could be troublesome for ObjC and various system libraries.

# Improvements
## Spliting
The correct implementation would use this to distinguish ObjC strings and C strings:  

```cpp
  set<GlobalVariable *> cstrings;
    set<GlobalVariable *> objcstrings;
    // Collect GVs
    for (auto g = M.global_begin(); g != M.global_end(); g++) {
      GlobalVariable *GV = &(*g);
      // We only handle NonMetadata&&NonObjC&&LocalInitialized&&CDS
      if (GV->hasInitializer() && GV->isConstant() &&
          GV->getSection() != StringRef("llvm.metadata") &&
          GV->getSection().find(StringRef("__objc")) == string::npos &&
          GV->getName().find("OBJC") == string::npos) {
        if (GV->hasInitializer() &&
            GV->getType()->getElementType() ==
                M.getTypeByName("struct.__NSConstantString_tag")) {
          objcstrings.insert(GV);
          continue;
        }
        // isString() asssumes the array has type i8, which should hold on all
        // major platforms  We don't care about some custom os written by 8yo
        // Bob that uses arbitrary ABI
        if (ConstantDataSequential *CDS =
                dyn_cast<ConstantDataSequential>(GV->getInitializer())) {
          if (CDS->isString()) {
            cstrings.insert(GV);
          }
        }
      }
    }
    // FIXME: Correctly Handle ObjC String Constants
    // Probably recreate at runtime and replace pointers?
    for (GlobalVariable *GV : objcstrings) {
      //@_unnamed_cfstring_ = private global %struct.__NSConstantString_tag {
      // i32* getelementptr inbounds ([0 x i32], [0 x i32]*
      //@__CFConstantStringClassReference, i32 0, i32 0),  i32 1992, i8*
      // getelementptr inbounds ([2 x i8], [2 x i8]* @.str, i32 0, i32 0), i64 1
      // },  section "__DATA,__cfstring", align 8
      ConstantStruct *CS = dyn_cast<ConstantStruct>(GV->getInitializer());
      Constant *CE = CS->getOperand(2); // This is GEP CE
      GlobalVariable *referencedGV =
          dyn_cast<GlobalVariable>(CE->getOperand(0));
      cstrings.erase(referencedGV);
    }
```

which correctly split CFStrings and C-Style strings. This works by collecting all strings first, then iterate all CFStrings and remove the C-Style strings referenced by the CFString from the list


# Crypto
First of all we need to analyze the GV's def-use chain and locate any instructions referencing it. This is not as trival as it seems because direct users are usually BitCast ConstantExprs, we need to iterate through the def-use chain.
Usually the use-def chain we are looking for look like this:  
``ConstantExpr->Instruction->BasicBlock->Function``
Then we can either create new AllocaInst at function entrypoint or re-use existing GVs.  The decryption can be done at function entrypoint, then possibly re-encrypt GVs back at terminators. Unless we are dealing with malformed BasicBlocks, which shouldn't happen unless frontend has gone wild.


There is a lot to do to make a workable obfuscator and ``GoSSIP-SJTU`` surely did some remarkable work, props to them.

Zhang

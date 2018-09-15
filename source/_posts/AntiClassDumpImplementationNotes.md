---
title: Anti class-dump Implementation Notes
date: 2017-12-21 17:19:36
tags: LLVM
---

The core idea of acd is to extract ObjC class informations and create everything at a program-controlled stage instead of letting libObjC do the initialization for us.
<!-- more -->
### Dependency Resolving
OOP languages have a core concept called "Classes", ObjC is no difference.For our transform to work as intended, we need to create classes strictly following the dependency graph.And that means:  

- Classes that don't have a base class should be created first.
- Following classes that has external dependency, which can be distinguished at IR level by checking if the BaseClass is definition or declaration.This is done using ``GlobalVariable::hasInitializer ()``
- Loop through the remaining classes using a deque.Which we repeat the following process until the deque is empty:
    - Pop the front-most class
    - Check if this class's dependency is available
        - If available,process the class
        - Otherwise,push the class to the back of the queue

The same also applies for protocols. Note we are running our pass at LTO stage to make sure local classes are sorted correctly

## Platform-Dependent Data Sizes
Certain ObjC Runtime API uses type like ``size_t`` which size is platform dependent.
Critical DataStructures like ``class_ro_t`` has platform specific definition:

```c++
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif
    const uint8_t * ivarLayout;
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
```


These issues are resolved by parsing Module's Triple.

## Handle IvarLayout
  Just add ``class_setIvarLayout`` and ``class_setWeakIvarLayout`` calls

## Provide Mode Switching
  Full LTO is not always possible. As such we should provide three modes:

- Injecting at global initializers
- Replace original metadata_ro_t and leave only ``+initialize``,perform method adding there. Ivars and props are unchanged
- User Custom Initializing using ``__attribute__((annotate("FLAGS")))`` , which is converted to ``llvm.global.annotations`` in ``llvm.metadata``

## Fix ro_flags
Google ``RO_IS_ARR``, we need to add instructions to copy flags over after creating class

## Misc Structs
``CGObjCMac.cpp`` 's comments explained the structs and their meanings in great detail.Below is IR-Level struct and their original definitions


``%struct._ivar_t = type { i64*, i8*, i8*, i32, i32 }``

```
struct _ivar_t {
  unsigned [long] int *offset;  // pointer to ivar offset location
  char *name;
  char *type;
  uint32_t alignment;
  uint32_t size;
}
```

``%struct._ivar_list_t = type { i32, i32, [0 x %struct._ivar_t] }``

```
struct _ivar_list_t {
  uint32 entsize;  // sizeof(struct _ivar_t)
  uint32 count;
  struct _iver_t list[count];
}
```

```
%struct._protocol_t =
type { i8*, i8*,
		%struct._objc_protocol_list*,
		%struct.__method_list_t*,
		%struct.__method_list_t*,
		%struct.__method_list_t*,
		%struct.__method_list_t*,
		%struct._prop_list_t*,
		i32, i32, i8**, i8*,
		%struct._prop_list_t*
		}
```

```
struct _protocol_t {
id isa;  // NULL
	const char * const protocol_name;
	const struct _protocol_list_t * protocol_list; // super protocols
	const struct method_list_t * const instance_methods;
	const struct method_list_t * const class_methods;
	const struct method_list_t *optionalInstanceMethods;
	const struct method_list_t *optionalClassMethods;
	const struct _prop_list_t * properties;
	const uint32_t size;  // sizeof(struct _protocol_t)
	const uint32_t flags;  // = 0
	const char ** extendedMethodTypes;
	const char *demangledName;
	const struct _prop_list_t * class_properties;
}
```

```
%struct._objc_protocol_list = type { i64, [0 x %struct._protocol_t*] }
```

```
struct _protocol_list_t {
	long protocol_count;   // Note, this is 32/64 bit
	struct _protocol_t[protocol_count];
}
```

```
%struct._prop_t
```

```
struct _prop_t {
	char *name;
	char *attributes;
}
```


T.B.C

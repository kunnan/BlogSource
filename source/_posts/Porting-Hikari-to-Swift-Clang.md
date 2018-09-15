---
title: Porting Hikari to Swift/Clang and Build for Production
date: 2018-02-13 06:03:00
tags: LLVM
---

Since Apple's Development Tools contains a bunch of modifications that will probably never see its day in LLVM upstream and Swift contains the latest open-source Apple fork of LLVM at the moment. It's probably wise to port Hikari to Swift instead of using the upstream.
<!-- more -->
# Obtaining the source code
```
mkdir swiftsrc \
&& cd swiftsrc \
&& git clone https://github.com/apple/swift.git \
&& ./swift/utils/update-checkout --clone
```

# Porting the Hikari Core
Hikari's core is located at ``lib/Transforms/Obfuscation``, plus headers at ``include/llvm/Transforms/Obfuscation/``.   
``HikariObfuscator/Mirai`` is the modified LLVM Core. So we first need to locate the Swift LLVM Core we just cloned, which is ``swiftsrc/llvm``.  

- Copy ``Hikari/lib/Transforms/Obfuscation`` to ``swiftsrc/llvm/lib/Transforms/Obfuscation``
- Copy ``Hikari/include/llvm/Transforms/Obfuscation`` to ``swiftsrc/llvm/include/llvm/Transforms/Obfuscation``
- Edit ``swiftsrc/llvm/lib/Transforms/LLVMBuild.txt``, add ``Obfuscation`` at the end of ``subdirectories``
- Edit ``swiftsrc/llvm/lib/Transforms/CMakeLists.txt``, add ``add_subdirectory(Obfuscation)`` at the end
- Edit ``swiftsrc/llvm/lib/Transforms/IPO/LLVMBuild.txt``, add ``Obfuscation`` at the end of ``required_libraries``
- Edit ``swiftsrc/llvm/lib/IPO/PassManagerBuilder.cpp``, add ``#include "llvm/Transforms/Obfuscation/Obfuscation.h"`` at the end of those ``#include`` statements, add ``MPM.add(createObfuscationPass());`` at the beginning of ``void PassManagerBuilder::populateModulePassManager``
- Edit ``swiftsrc/llvm/include/llvm/InitializePasses.h``, add the following right after the last ``void initializeXXXX`` definition:  

```
void initializeStringEncryptionPass(PassRegistry &);
void initializeFunctionCallObfuscatePass(PassRegistry &);
void initializeAntiDebuggingPass(PassRegistry &);
void initializeAntiClassDumpPass(PassRegistry &);
void initializeBogusControlFlowPass(PassRegistry &);
void initializeFlatteningPass(PassRegistry &);
void initializeSplitBasicBlockPass(PassRegistry &);
void initializeSubstitutionPass(PassRegistry &);
void initializeAntiDebuggingPass(PassRegistry &);
void initializeAntiHookPass(PassRegistry &);
void initializeIndirectBranchPass(PassRegistry &);
void initializeObfuscationPass(PassRegistry &);
```
- Edit ``swiftsrc/llvm/include/llvm/LinkAllPasses.h``, add ``#include "llvm/Transforms/Obfuscation/Obfuscation.h"`` at the end of those ``#include`` statements,then add the following right after the last ``(void) llvm::createXXXXX`` definition:

```
      (void) llvm::createStringEncryptionPass();
      (void) llvm::createFunctionCallObfuscatePass();
      (void) llvm::createAntiClassDumpPass();
      (void) llvm::createBogusControlFlowPass();
      (void) llvm::createFlatteningPass();
      (void) llvm::createSplitBasicBlockPass();
      (void) llvm::createSubstitutionPass();
      (void) llvm::createAntiDebuggingPass();
      (void) llvm::createAntiHookPass();
      (void) llvm::createIndirectBranchPass();
      (void) llvm::createObfuscationPass();
```

# Building

## Effortless Building
  Apple provided ``swiftsrc/swift/utils/build-toolchain``
## Crafted Build Options
``utils/build-toolchain`` provides a dumb wrapper around the core build script and doesn't provide fine tuning. We could it a little to provide new Toolchain name and stuff. I personally replaced the ``./utils/build-script`` command to something like this:   

```
./utils/build-script ${DRY_RUN} --preset-file="utils/HikariSwift" --preset="Hikari"\
        install_destdir="${SWIFT_INSTALL_DIR}" \
        installable_package="${SWIFT_INSTALLABLE_PACKAGE}" \
        install_toolchain_dir="${SWIFT_TOOLCHAIN_DIR}" \
        install_symroot="${SWIFT_INSTALL_SYMROOT}" \
        symbols_package="${SYMBOLS_PACKAGE}" \
```
Alone with a hand-crafted preset,stored at ``utils/HikariSwift`` :  

```
[preset: Hikari]

dash-dash
no-assertions
ios
tvos
watchos

lldb
llbuild
swiftpm
playgroundsupport
release
compiler-vendor=none
dash-dash
lldb-no-debugserver
lldb-use-system-debugserver
extra-cmake-options=-DCMAKE_BUILD_TYPE=MinSizeRel
verbose-build
build-ninja
build-swift-static-stdlib
build-swift-static-sdk-overlay
build-swift-stdlib-unittest-extra
playgroundsupport-build-type=Release

install-swift
install-lldb
install-llbuild
install-swiftpm
install-playgroundsupport

install-destdir=%(install_destdir)s

darwin-install-extract-symbols

# Path where debug symbols will be installed.
install-symroot=%(install_symroot)s

# Path where the compiler, the runtime and the standard libraries will be
# installed.
install-prefix=%(install_toolchain_dir)s/usr
skip-test-swift
skip-test-swiftpm
skip-test-llbuild
skip-test-lldb
skip-test-cmark
skip-test-playgroundsupport
swift-install-components=compiler;clang-builtin-headers;stdlib;swift-syntax;sdk-overlay;license;sourcekit-xpc-service;swift-remote-mirror;swift-remote-mirror-headers
llvm-install-components=libclang;libclang-headers

# Path to the .tar.gz package we would create.
installable-package=%(installable_package)s

# Path to the .tar.gz symbols package
symbols-package=%(symbols_package)s

# Info.plist
darwin-toolchain-bundle-identifier="com.naville.hikariswift"
darwin-toolchain-display-name="HikariSwift"
darwin-toolchain-display-name-short="HikariSwift"
darwin-toolchain-name="HikariSwift"
darwin-toolchain-version="1.0.0"
darwin-toolchain-alias="Local"

```

# Miscellaneous Notes  

- AAPL didn't ship LLVM's libcxx and libcxxabi. You might want to setup those yourself
- Hikari is pretty much untested on Swift projects
- Swift frontend uses a slightly different argument to pass LLVM flags

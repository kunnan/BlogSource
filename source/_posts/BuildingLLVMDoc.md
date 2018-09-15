---
layout: default
title:  "Building LLVM Doc(set)"
date:   2017-04-06 20:18:27 +0000
tags: LLVM
---

  LLVM has a large doc base which takes literally 12hours+ to build on my configuration.
  This post is a collection of techniques I found that could improve either the build doc size/build time/quality
<!-- more -->
# Generic Changes
  - Change `VERBATIM_HEADERS` and `SOURCE_BROWSER` to `NO`. This would remove embedded LLVM source, results in much smaller build size and shorter build time.
  - Change `STRIP_FROM_PATH` to correct file-system folder base, this would fix some issues where absolute path is included in the product

# Speedup Indexing
After generating the docs, you can generate  Apple Docset, in which the indexing process would take more than 12hours normally.   
  Edit the generated Makefile
    - Clang:``$(BUILD_ROOT)/tools/clang/docs/doxygen/html/Makefile``  
    - LLVM: ``$(BUILD_ROOT)/docs/doxygen/html/Makefile``  
  Change ``$(XCODE_INSTALL)/usr/bin/docsetutil index $(DOCSET_NAME)`` to
  ``$(XCODE_INSTALL)/usr/bin/docsetutil index -skip-text $(DOCSET_NAME)`` will decrease the indexing time to roughly an hour or so without negative impact as far as I know

# Optional

  - Change `CREATE_SUBDIRS` to `YES`  
  	This option is described as:		

> If the CREATE_SUBDIRS tag is set to YES, then doxygen will create 4096 sub-directories (in 2 levels) under the output directory of each output format and will distribute the generated files over these directories. Enabling this  
option can be useful when feeding doxygen a huge amount of source files, where putting all generated files in the same directory would otherwise causes performance problems for the file system.  

  - Edit `DOCSET_FEEDNAME`,  
    `DOCSET_BUNDLE_ID`,`DOCSET_PUBLISHER_ID`,  
    `DOCSET_PUBLISHER_NAME` if `GENERATE_DOCSET` is set to `YES` (Used for building Apple Docset
  - Change `DOT_IMAGE_FORMAT` to `svg` and `INTERACTIVE_SVG` to `YES`, the former would use SVGs instead of PNGs, the latter would generate interactive,zoom-able inherit diagrams


  - For Apple Docset users, refer to <https://kapeli.com/docsets#doxygen> for more specific instructions

---
layout: default
title:  "Implementing A Custom MachO Parser"
date:   2017-03-09 07:04:41 +0000
tags: RE
---


  Despite Mach-O being a (somewhat) popular executable format. Current open-source implementations lacks either  cross-platform support or some obscure LoadCommand types. Furthermore, there is no library (not that I know of) supports deserialize and **re**serialize of MachO executables, which would have been a useful tool for Reverse Engineers

  Thus a personal research project [NyachOKit](https://github.com/Naville/NyachOKit) was initialized in an attempt to solve this issues once and for all.

**Note: This article is more like a technical note for implementing my own parser.**

<!-- more -->

# Design Principles

The main difficulties of building a library capable of solving the issues proposed above, in my humble opinion , is the over-complexed internal dependencies of a MachO Executable. As such, one of the design principles is to **strip away redundant information as much as possible and reconstruct them on-the-fly during re-construction**.

Another issue of MachO parsing is endian conversions. As such each ADT contains a flag for the endians.

# ADT

Theoretically a tree might be a better ADT than the current implementation (a list of objects) , however since we've stripped away redundant information and linked raw data to their corresponding load commands, the tree is effectively a unsorted tree, thus current ADT design is suitable (in my humble opinion) and the time-consuming and overkill tree design can be abandoned

## implementation Specifications
Everything is mapped into a corresponding C++ Object. The core is a thin MachO Parser. Fat Mach-Os are mapped into an object containing fat headers and a list of thin Mach-O objects.

Thin MachO parser is iterable object so malformed MachOs will only stop parsing at a point (Useful for fixing malformed ones), insteading of crashing the whole analyzing process.

Each Load Command is mapped into a corresponding LC Object.

### LoadCommand Classes
-	The usual header fields (mapped into a hashmap with field name being the struct field name) as well as a pointer to data (If this LC type contains associated data) or NULL.
- 	Method for serializing data to specified file offsets
-  String description

 Note the HashMap design is for better frontend GUIs

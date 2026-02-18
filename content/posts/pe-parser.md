---
title: "Building a PE Parser in C"
date: 2026-02-18
draft: false
---

## Introduction

A PE (Portable Executable) file is the format Windows uses for executables and DLLs. Understanding PE structure is fundamental for malware analysis and reverse engineering.

## PE Structure
```
┌─────────────────────────────────────┐
│ DOS Header                          │
├─────────────────────────────────────┤
│ NT Headers                          │
├─────────────────────────────────────┤
│ Section Headers                     │
├─────────────────────────────────────┤
│ Sections                            │
└─────────────────────────────────────┘
```

## Parsing the DOS Header
```c
PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)Buffer;

if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
{
    printf("Not a valid PE file\n");
    return FALSE;
}
```

## What I Learned

- PE files always start with "MZ"
- The `e_lfanew` field points to NT headers
- Section headers describe code and data layout

## Source Code

Full code available on [GitHub](https://github.com/jashidsany/pe-parser).

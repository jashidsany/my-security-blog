---
title: "Building a PE Parser in C"
date: 2026-02-18
draft: false
description: "A deep dive into Windows PE file structure. Learn how to parse DOS headers, NT headers, and sections by building a parser from scratch in C."
---

## Introduction

A PE (Portable Executable) file is the format Windows uses for executables (.exe), DLLs (.dll), and other binary files. If you want to understand how Windows works under the hood — whether for malware analysis, reverse engineering, or offensive security — understanding PE structure is essential.

In this post, I'll walk through building a PE parser from scratch in C, explaining each component along the way.

---

## What is a PE File?

Every time you run a program on Windows, the OS loader reads the PE file and maps it into memory. The PE format tells Windows:

- Where the code is
- Where the data is
- What DLLs are needed
- Where to start execution

---

## PE Structure Overview

A PE file is organized into headers and sections:

| Component | Purpose |
|-----------|---------|
| DOS Header | Legacy DOS compatibility, pointer to NT headers |
| DOS Stub | "This program cannot be run in DOS mode" |
| NT Headers | PE signature, file info, memory layout |
| Section Headers | Describes each section (.text, .data, etc.) |
| Sections | Actual code and data |

---

## Setting Up the Project

First, let's set up our includes and create a function to read a file into memory:
```c
#include <Windows.h>
#include <stdio.h>
```

**Why these includes?**
- `Windows.h` — Gives us PE structures (`IMAGE_DOS_HEADER`, `IMAGE_NT_HEADERS`) and Windows API functions
- `stdio.h` — For `printf` output

---

## Reading the File Into Memory

Before we can parse anything, we need to read the PE file into a buffer:
```c
BOOL
ReadFileIntoBuffer(
    _In_  LPCWSTR FilePath,
    _Out_ PBYTE   *Buffer,
    _Out_ PDWORD  FileSize
)
{
    HANDLE hFile       = INVALID_HANDLE_VALUE;
    DWORD  dwBytesRead = 0;
    PBYTE  pBuffer     = NULL;
    DWORD  dwFileSize  = 0;

    hFile = CreateFileW(
        FilePath,
        GENERIC_READ,
        FILE_SHARE_READ,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hFile == INVALID_HANDLE_VALUE)
    {
        printf("[-] CreateFileW failed. Error: %d\n", GetLastError());
        return FALSE;
    }

    dwFileSize = GetFileSize(hFile, NULL);
    pBuffer = (PBYTE)HeapAlloc(GetProcessHeap(), HEAP_ZERO_MEMORY, dwFileSize);

    if (pBuffer == NULL)
    {
        printf("[-] HeapAlloc failed. Error: %d\n", GetLastError());
        CloseHandle(hFile);
        return FALSE;
    }

    if (!ReadFile(hFile, pBuffer, dwFileSize, &dwBytesRead, NULL))
    {
        printf("[-] ReadFile failed. Error: %d\n", GetLastError());
        HeapFree(GetProcessHeap(), 0, pBuffer);
        CloseHandle(hFile);
        return FALSE;
    }

    *Buffer   = pBuffer;
    *FileSize = dwFileSize;

    CloseHandle(hFile);
    return TRUE;
}
```

**Key points:**
- `CreateFileW` opens the file for reading
- `GetFileSize` tells us how much memory to allocate
- `HeapAlloc` allocates memory on the process heap
- `ReadFile` copies file contents into our buffer
- Always check return values and clean up on failure

---

## Parsing the DOS Header

The DOS header is always at offset 0 of a PE file. It starts with the magic bytes "MZ" (0x5A4D).
```c
BOOL
ParseDosHeader(
    _In_ PBYTE Buffer
)
{
    PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)Buffer;

    if (pDosHeader->e_magic != IMAGE_DOS_SIGNATURE)
    {
        printf("[-] Invalid DOS signature. Not a PE file.\n");
        return FALSE;
    }

    printf("[+] DOS Header:\n");
    printf("    e_magic:    0x%X (MZ)\n", pDosHeader->e_magic);
    printf("    e_lfanew:   0x%X (Offset to NT Headers)\n", pDosHeader->e_lfanew);

    return TRUE;
}
```

**What's happening:**
- We cast the buffer to `PIMAGE_DOS_HEADER` because the DOS header is at offset 0
- `e_magic` must be 0x5A4D ("MZ") — named after Mark Zbikowski who designed the format
- `e_lfanew` is the most important field — it tells us where the NT headers are

---

## Parsing the NT Headers

The NT headers contain the PE signature and two important structures: the File Header and Optional Header.
```c
BOOL
ParseNtHeaders(
    _In_ PBYTE Buffer
)
{
    PIMAGE_DOS_HEADER pDosHeader = (PIMAGE_DOS_HEADER)Buffer;
    PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(Buffer + pDosHeader->e_lfanew);

    if (pNtHeaders->Signature != IMAGE_NT_SIGNATURE)
    {
        printf("[-] Invalid NT signature. Not a PE file.\n");
        return FALSE;
    }

    printf("[+] NT Headers:\n");
    printf("    Signature:  0x%X (PE)\n", pNtHeaders->Signature);

    // File Header
    printf("[+] File Header:\n");
    printf("    Machine:              0x%X\n", pNtHeaders->FileHeader.Machine);
    printf("    NumberOfSections:     %d\n", pNtHeaders->FileHeader.NumberOfSections);
    printf("    TimeDateStamp:        0x%X\n", pNtHeaders->FileHeader.TimeDateStamp);
    printf("    Characteristics:      0x%X\n", pNtHeaders->FileHeader.Characteristics);

    // Optional Header
    printf("[+] Optional Header:\n");
    printf("    Magic:                0x%X (%s)\n", 
        pNtHeaders->OptionalHeader.Magic,
        pNtHeaders->OptionalHeader.Magic == 0x20B ? "64-bit" : "32-bit"
    );
    printf("    AddressOfEntryPoint:  0x%X\n", pNtHeaders->OptionalHeader.AddressOfEntryPoint);
    printf("    ImageBase:            0x%llX\n", pNtHeaders->OptionalHeader.ImageBase);
    printf("    SizeOfImage:          0x%X\n", pNtHeaders->OptionalHeader.SizeOfImage);

    return TRUE;
}
```

**Key fields explained:**

| Field | What It Tells Us |
|-------|------------------|
| `Machine` | Target CPU (0x8664 = x64, 0x14C = x86) |
| `NumberOfSections` | How many sections to parse |
| `AddressOfEntryPoint` | Where execution begins (RVA) |
| `ImageBase` | Preferred load address in memory |
| `SizeOfImage` | Total size when loaded |

**Finding the NT headers:**
```c
PIMAGE_NT_HEADERS pNtHeaders = (PIMAGE_NT_HEADERS)(Buffer + pDosHeader->e_lfanew);
```

This is pointer arithmetic: Base address + offset = NT headers address.

---

## Parsing Section Headers

Sections contain the actual code and data. Common sections include:

| Section | Purpose |
|---------|---------|
| `.text` | Executable code |
| `.data` | Initialized global data |
| `.rdata` | Read-only data (strings, constants) |
| `.rsrc` | Resources (icons, dialogs) |
| `.reloc` | Relocation information |
```c
VOID
ParseSectionHeaders(
    _In_ PBYTE Buffer
)
{
    PIMAGE_DOS_HEADER     pDosHeader     = (PIMAGE_DOS_HEADER)Buffer;
    PIMAGE_NT_HEADERS     pNtHeaders     = (PIMAGE_NT_HEADERS)(Buffer + pDosHeader->e_lfanew);
    PIMAGE_SECTION_HEADER pSectionHeader = IMAGE_FIRST_SECTION(pNtHeaders);
    WORD                  wNumSections   = pNtHeaders->FileHeader.NumberOfSections;

    printf("[+] Sections (%d):\n", wNumSections);
    printf("    %-8s %-10s %-10s %-10s %-10s\n", 
        "Name", "VirtAddr", "VirtSize", "RawAddr", "RawSize"
    );

    for (WORD i = 0; i < wNumSections; i++)
    {
        printf("    %-8.8s 0x%-8X 0x%-8X 0x%-8X 0x%-8X\n",
            pSectionHeader[i].Name,
            pSectionHeader[i].VirtualAddress,
            pSectionHeader[i].Misc.VirtualSize,
            pSectionHeader[i].PointerToRawData,
            pSectionHeader[i].SizeOfRawData
        );
    }
}
```

**Understanding section fields:**
- `VirtualAddress` — Where the section is loaded in memory (RVA)
- `VirtualSize` — Size in memory
- `PointerToRawData` — Where the section is in the file
- `SizeOfRawData` — Size on disk

---

## Putting It All Together
```c
INT
wmain(
    INT    argc,
    PWCHAR argv[]
)
{
    PBYTE pFileBuffer = NULL;
    DWORD dwFileSize  = 0;

    if (argc != 2)
    {
        wprintf(L"[!] Usage: %s <path to PE file>\n", argv[0]);
        return -1;
    }

    wprintf(L"[+] Parsing: %s\n\n", argv[1]);

    if (!ReadFileIntoBuffer(argv[1], &pFileBuffer, &dwFileSize))
    {
        return -1;
    }

    printf("[+] File size: %d bytes\n\n", dwFileSize);

    if (!ParseDosHeader(pFileBuffer))
    {
        goto Cleanup;
    }

    if (!ParseNtHeaders(pFileBuffer))
    {
        goto Cleanup;
    }

    ParseSectionHeaders(pFileBuffer);

Cleanup:
    if (pFileBuffer != NULL)
    {
        HeapFree(GetProcessHeap(), 0, pFileBuffer);
    }

    return 0;
}
```

---

## Example Output

Running the parser against notepad.exe:
```
[+] Parsing: C:\Windows\System32\notepad.exe

[+] File size: 201216 bytes

[+] DOS Header:
    e_magic:    0x5A4D (MZ)
    e_lfanew:   0xF0 (Offset to NT Headers)

[+] NT Headers:
    Signature:  0x4550 (PE)
[+] File Header:
    Machine:              0x8664
    NumberOfSections:     7
    TimeDateStamp:        0x5E2C4A4B
    Characteristics:      0x22
[+] Optional Header:
    Magic:                0x20B (64-bit)
    AddressOfEntryPoint:  0x1A4B0
    ImageBase:            0x140000000
    SizeOfImage:          0x39000

[+] Sections (7):
    Name     VirtAddr   VirtSize   RawAddr    RawSize   
    .text    0x1000     0x1A3E4    0x400      0x1A400   
    .rdata   0x1C000    0xCE2C     0x1A800    0xD000    
    .data    0x29000    0x1690     0x27800    0x600     
    .pdata   0x2B000    0x1B54     0x27E00    0x1C00    
    .didat   0x2D000    0x130      0x29A00    0x200     
    .rsrc    0x2E000    0x9CE8     0x29C00    0x9E00    
    .reloc   0x38000    0x298      0x33A00    0x400     
```

---

## What I Learned

Building this parser taught me:

1. **PE structure is logical** — Each header points to the next, like a chain
2. **Pointer arithmetic is essential** — Base + offset = target address
3. **Always validate** — Check magic numbers before parsing further
4. **Windows API patterns** — Error checking, cleanup with goto, handle management
5. **Why this matters for security** — Malware analysis, packing, injection all require PE knowledge

---

## What's Next

Future improvements I'm planning:
- Parse Import Table (see what DLLs a PE uses)
- Parse Export Table (see what functions a DLL exports)
- Detect packed executables
- Add entropy analysis

---

## Source Code

Full source code available on [GitHub](https://github.com/jashidsany/pe-parser).

---

## References

- [Microsoft PE Format Documentation](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format)
- [PE Format - Wikipedia](https://en.wikipedia.org/wiki/Portable_Executable)
- Windows Internals by Mark Russinovich

---
layout: post
author: UnpackingNoobs
title: "Lection 4: Proc, where are you?"
date: 2025-05-05
tags:
    - "Binary unpacking"
---

### In this lection we will learn:

* how to get details on specific functions of the WinAPI
* how to perform late-loading and create the IAT ourself
* learn a bit more about the PE file structure
* write our own minimalistic IAT loader

### What you need for this lection:

* The examples from the _[Lection1](https://github.com/UnpackingNoobs/Lection1)_ and _[Lection4](https://github.com/UnpackingNoobs/Lection4)_ repos
* A debugger of your choice (preferrably [x64dbg](https://x64dbg.com/))
* A tool to edit PE files (preferrably [PE-bear](https://github.com/hasherezade/pe-bear))

# Recap

In [lection 3](/lections/lection_3) we successfully unpacked a program and fixed the imoprts so that it runs properly.<br/>
In this lection we will see how the packer is capable of late-loading the original IAT at runtime.

# You only need two

As we figured out in the last lection, the IAT of a program is loaded/initialized by the OS when the program starts. That means that if we would like to use a function from a DLL that was not known at program startup, we have no chance to get the address of that function - right? Well, not exactly. Windows offers a mechanism to late-load a DLL during runtime and to retrieve the address of a function within this DLL. As a sidenote: The functions that are provided by a DLL are actually called Exports and these Exports - just as Imports - have their own section in the PE file structure, but thats just a minor detail right now.<br/><br/>

In order to find out how late-loading is done, we could search the internet and find the solution pretty fast and that would be absolutely fine, but I would like to go the more hands-on route in order to see everything in action and to get a better understanding on how a packer works on the inside.<br/><br/>

By the way: In the previous lections I always said something like _packer_ or _packer-stuff_, what I meant by that is the code that runs at program startup and unpacks the original program. It's quite common to call this small piece of software the _packer stub_ or sometimes just _stub_, from now on I will also call it like that.<br/><br/>

Ok, back on the topic. Since the stub has to perform late-loading in order to load the original IAT, it definitely needs to call some WinAPI functions. In the previous lection we saw that the _packed.exe_ only has 4 imports, so we can guess that among these imports there might be functions than can perfom late-loading.<br/>
So open up PE-bear again, load the _packed.exe_ and have a look at the imports, what do we have?<br/><br/>

* [LoadLibraryA](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibrarya)
* [ExitProcess](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-exitprocess)
* [GetProcAddress](https://learn.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress)
* [VirtualProtect](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualprotect)

Well, we can instantly rule out _ExitProcess_, so we are down to three.<br/><br/>

As you might have already noticed, I linked the WinAPI documentation for every function.<br/>
I expect that you have consulted them many times, if not, here's a short guide:

* At the top under _Syntax_ you will find the signature of the function.
* Under _Parameters_, just as the name implies, every parameter is explained in detail.
* _Return value_ explains the value that will be returned from the function and wether some special values might be returned, like _NULL_.

Time to read the description of the remaining functions:

**LoadLibraryA**: Loads the specified module into the address space of the calling process.<br/>
**GetProcAddress**: Retrieves the address of an exported function (also known as a procedure) or variable from the specified dynamic-link library (DLL).<br/>
**VirtualProtect**: Changes the protection on a region of committed pages in the virtual address space of the calling process.<br/>

Out of these, _LoadLibraryA_ and _GetProcAddress_ sound very promising! Time to make ourselfes a bit familiar with the two functions.<br/><br/>

As we know that the _original.exe_ uses functions from the KERNEL32.dll and since GetProcAddress uses the retrieved handle from LoadLibraryA, we expect to see something like this:

```c
HMODULE hModule = LoadLibraryA("KERNEL32.dll");

// Check handle

for (int i = 0; i < NUM_PROCS; i++)
{
    FARPROC procAddress = GetProcAddress(hModule, procNames[i]);

    // Check proc

    // Store procAddress in IAT
}
```

Time to start up the debugger and see if we are right.<br/>
For this, place a breakpoint on every call to LoadLibraryA and GetProcAddress and let the program run freely... But wait... Strangely we can not find the places where these functions are called. It looks like even parts of the stub are packed or somewhat obscured. So we go a different route and place a breakpoint directly at the start of the functions via the debugger commands:<br/><br/>

_bp LoadLibraryA_ and _bp GetProcAddress_<br/><br/>

Hit run and we should break in _LoadLibraryA_. Have a look at the stack, it's exactly as we expected it: _KERNEL32.dll_ was passed as the parameter. Double click the return address to see how we came here. Aha, the function was called at address 0x00420C82. Press F9 and see where we break next. Just as expected, we land in _GetProcAddress_ and the second parameter is a pointer to "QueryPerformanceCounter", our first import. Again, we check where the function was called (0x00420C97) and should realize that the code looks exactly like we expected it:

![Loading the IAT]({{site.url}}/assets/lection_4/load_iat.png)

One thing I did not told you so far is that the packer that I use in the examples is [UPX](https://upx.github.io/), a very well known packer that is open source, so we can have a look at the real code. Unluckily the [import part](https://github.com/upx/upx/blob/5ed1d5b2b36b51b50ceb11b33820404881e6a731/src/stub/src/i386-win32.pe.S#L103) is also written in assembly.<br/><br/>

By the way, _LoadLibrary_ and _GetProcAddress_ are the only two functions you need, all other functions can be imported via them.

# Virtual RVAlity

In order to familiarize us a bit more on how a PE-file looks like and how a packer stub creates the IAT, we are now going to build a very simple and crude PE-file parser that will get the imports and then request the addresses from the WinAPI.<br/><br/>

You can find many details about the PE format in the internet, but you can mainly stick to the [Official Microsoft Documentation}(https://learn.microsoft.com/en-us/windows/win32/debug/pe-format). Nevertheless, the illustration from [corkami](https://github.com/corkami/pics/tree/master/binary/pe101) is also very well done.<br/><br/>
If we read the docs, we will find that a PE file starts with a _MS-DOS stub_. The stub is a valid MS-DOS mini-program usually only printing the famous message _'This program cannot be run in DOS mode'_. At offset 0x3c (_e_lfanew_) in the stub we find the offset to the image NT headers. There are two versions of these headers. A 32 Bit and a 64 Bit version. To know what we are dealing with, we need to check the _Machine_ member of the _IMAGE_FILE_HEADER_, so something like:

```c
/** Get offset of PE Header */
PIMAGE_DOS_HEADER DOSHeader = (PIMAGE_DOS_HEADER)peFileData;
DWORD peHeaderOffset = _DOSHeader->e_lfanew;

/* Get machine type */
PIMAGE_FILE_HEADER pImageFileHeader = (PIMAGE_FILE_HEADER)&peFileData[peHeaderOffset + sizeof(DWORD)];  // Skip signature

if (pImageFileHeader->Machine == 0x14c)
{
    // 32 Bit
}
else if (pImageFileHeader->Machine == 0x8664)
{
    // 64 Bit
}
```

In order to get to the imports, we first need to get the Virtual Address of the _Import Directory Table_ (**IDT**, aka the list with the module-names we see in PE-bear under _Imports_). To get this address, we need to have a look into the second _IMAGE_DATA_DIRECTORY_ entry in the _IMAGE_OPTIONAL_HEADER_. Why the second you may ask. Well, because as per definition there are 16 _IMAGE_DATA_DIRECTORY_ entries and the indicies are defined as follows:

```c
// Directory Entries

#define IMAGE_DIRECTORY_ENTRY_EXPORT          0   // Export Directory
#define IMAGE_DIRECTORY_ENTRY_IMPORT          1   // Import Directory
#define IMAGE_DIRECTORY_ENTRY_RESOURCE        2   // Resource Directory
#define IMAGE_DIRECTORY_ENTRY_EXCEPTION       3   // Exception Directory
#define IMAGE_DIRECTORY_ENTRY_SECURITY        4   // Security Directory
#define IMAGE_DIRECTORY_ENTRY_BASERELOC       5   // Base Relocation Table
#define IMAGE_DIRECTORY_ENTRY_DEBUG           6   // Debug Directory
//      IMAGE_DIRECTORY_ENTRY_COPYRIGHT       7   // (X86 usage)
#define IMAGE_DIRECTORY_ENTRY_ARCHITECTURE    7   // Architecture Specific Data
#define IMAGE_DIRECTORY_ENTRY_GLOBALPTR       8   // RVA of GP
#define IMAGE_DIRECTORY_ENTRY_TLS             9   // TLS Directory
#define IMAGE_DIRECTORY_ENTRY_LOAD_CONFIG    10   // Load Configuration Directory
#define IMAGE_DIRECTORY_ENTRY_BOUND_IMPORT   11   // Bound Import Directory in headers
#define IMAGE_DIRECTORY_ENTRY_IAT            12   // Import Address Table
#define IMAGE_DIRECTORY_ENTRY_DELAY_IMPORT   13   // Delay Load Import Descriptors
#define IMAGE_DIRECTORY_ENTRY_COM_DESCRIPTOR 14   // COM Runtime descriptor
```

Note: In theory there are not always 16 entries (see _NumberOfRvaAndSizes_ for more on that), but we will ignore this detail for now.<br/><br/>
An _IMAGE_DATA_DIRECTORY_ is just a struct that looks like the following:

```c
typedef struct _IMAGE_DATA_DIRECTORY {
    DWORD   VirtualAddress;
    DWORD   Size;
} IMAGE_DATA_DIRECTORY, *PIMAGE_DATA_DIRECTORY;
```

So in order to get the _VirtualAddress_ of the IDT, we can do the following:

```c
// 32 Bit
PIMAGE_NT_HEADERS32 pImageNTHeaders32 = (PIMAGE_NT_HEADERS32)&peFileData[peHeaderOffset];
PIMAGE_OPTIONAL_HEADER32 pImageOptionalHeader32 = &pImageNTHeaders32->OptionalHeader;
PIMAGE_DATA_DIRECTORY pDataDirectoryImports = &pImageOptionalHeader32->DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT];
DWORD vaImports = pDataDirectoryImports->VirtualAddress;
```

The place where the IDT is located in the PE-file is usually within the sections in a section called _.idata_. Some compilers, especially when optimization options are enabled, will merge various sections together, so you will not always find a section that is really called _.idata_. Moreover, the _VirtualAddress_ we talked about earlier is the address in the mapped (loaded) PE-file in memory! For example in the _original.exe_ that comes with the [Lection1](https://github.com/UnpackingNoobs/Lection1) material, PE-bear shows us, that the IDT shall be located at 0x1C1FC (Virtual Address):

![IDT RVA]({{site.url}}/assets/lection_4/idt_rva.png)

But the real address on disc (File Offset or Raw Address) is 0x1AFFC:

![IDT File Offset]({{site.url}}/assets/lection_4/idt_disc.png)

You can easily verify this if you load _original.exe_ in your debugger and have a look at 0x0041C1FC (Image Base of 0x00400000 + 0x1C1FC RVA, mapped) and compare this to the hex view at address 0x1AFFC (unmapped):

![IDT RVA]({{site.url}}/assets/lection_4/idt_mapped.png)<br/>
![IDT RVA]({{site.url}}/assets/lection_4/idt_unmapped.png)

But the question is now, how is 0x1AFFC calculated? It's actually surprisingly complicated (but actually not too much). We have to go through all sections, and check if the RVA we are looking for is within the Virtual Address Space of the section. If so, calculate the offset from the start of the mapped section to the RVA and apply that offset to the start of the unmapped section (Raw address). Best we see this directly in code, step by step:

First, we get the offset of the PE Header

```c
PIMAGE_DOS_HEADER DOSHeader = (PIMAGE_DOS_HEADER)peFileData;
DWORD peHeaderOffset = DOSHeader->e_lfanew;
```

then we get the Image NT Headers to get the Optional Headers where we find the Data Directory of the Imports 

```c
PIMAGE_NT_HEADERS32 pImageNTHeaders32 = (PIMAGE_NT_HEADERS32)&peFileData[peHeaderOffset];
PIMAGE_OPTIONAL_HEADER32 pImageOptionalHeader32 = &pImageNTHeaders32->OptionalHeader;
PIMAGE_DATA_DIRECTORY pDataDirectoryImports = &pImageOptionalHeader32->DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT];
```

In the directory we will find the Virtual Address of the Imports

```c
DWORD virtualAddressIData = pDataDirectoryImports->VirtualAddress;
```

From the Image NT Headers we also get the number of sections

```c
WORD numberOfSections = pImageNTHeaders32->FileHeader.NumberOfSections;
```

The section headers start right after the Image NT Headers

```c
PIMAGE_SECTION_HEADER pImageSectionHeaders = (PIMAGE_SECTION_HEADER)((PBYTE)pImageNTHeaders32 + sizeof(IMAGE_NT_HEADERS32));
```

From there we can loop the sections until we have found the one containing the Imports.

```c
DWORD rawOffsetIData;

for (size_t i = 0; i < numberOfSections; i++)
{
    /* Check if RVA within section */
    if (
            (pImageSectionHeaders->VirtualAddress <= virtualAddressIData) &&
            (virtualAddressIData < (pImageSectionHeaders->VirtualAddress + pImageSectionHeaders->Misc.VirtualSize))
        )
    {
        DWORD offset = virtualAddressIData - pImageSectionHeaders->VirtualAddress;

        rawOffsetIData = pImageSectionHeaders->PointerToRawData + offset;

        break;
    }

    /* Next section */
    pImageSectionHeaders++;
}
```

Once we know where the IDT is, we can cast a pointer to the location and loop the IDT until the Characteristics of an Import Descriptor is zero (last Directory is always zero)

```c
PIMAGE_IMPORT_DESCRIPTOR pImportDescriptor = (PIMAGE_IMPORT_DESCRIPTOR)&peFileData[rawOffsetIData];

while (pImportDescriptor->Characteristics != 0)
{
    pImportDescriptor++;
}
```

To get the Imports (Thunks) itself, cast a PIMAGE_THUNK_DATA to the address of pImportDescriptor->FirstThunk, but keep in mind that that address is again a mapped address, so calculate the raw address first! Then we can loop the thunks until the Ordinal is zero.

```c
DWORD firstThunkRaw = RVA2Raw(peFileData, pImportDescriptor->FirstThunk);

PIMAGE_THUNK_DATA pThunk = (PIMAGE_THUNK_DATA)&peFileData[firstThunkRaw];

while (pThunk->u1.Ordinal != 0)
{
    pThunk++;
}
```

There is actually much more to it (Forwarding, Bound/Delayed Imports, ...), but we'll keep it simple this time.<br/><br/>

Under the _[Lection4](https://github.com/UnpackingNoobs/Lection4)_ repo you will find a bit longer example that will handle 32/64 Bit. There are probably some errors in the code and nearly no error checking, but I guess you'll get the idea.<br/><br/>

That's it for today. In the next lection we will try to find the OEP even if we don't have the _original.exe_.


### Conclusion

* The official Microsoft Documentation is actually quite comprehensive
* All you ever need is _LoadLibrary_ and _GetProcAddress_
* The PE-File format is somewhat tricky, but luckily many people have struggled with it before so you find many good explanations online 

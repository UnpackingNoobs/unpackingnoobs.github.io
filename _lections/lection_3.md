---
layout: post
author: UnpackingNoobs
title: "Lection 3: To the packer and back"
date: 2025-05-03
tags:
    - "Binary unpacking"
---

### In this lection we will learn:

* about the IAT
* how to properly perform Step 3

### What you need for this lection:

* The examples from the _[Lection1](https://github.com/UnpackingNoobs/Lection1)_ repo
* A debugger of your choice (preferrably [x64dbg](https://x64dbg.com/))
* A tool to edit PE files (preferrably [PE-bear](https://github.com/hasherezade/pe-bear))

# Recap

In [lection 2](/lections/lection_2) we successfully dumped the packed program and fixed the entry point, but so far we did not end up with a properly running program.<br/>
In this lection we will try to figure out the steps needed to make everything run again.

# Finding the elephant in the room

In order to figure out what is the main reason why the dumped program does not want to run, we can compare the _original.exe_ from [Lection1](https://github.com/UnpackingNoobs/Lection1) and _dumped.exe_ in a PE viewer (I will use PE-bear). The DOS Header, Rich Header and File Header should all look the same, so they are of no help. In the Optional Header we can see some differences, but nothing special, some offsets and sizes are different, but this can probably be explained by the fact that the packed program also holds the packer-stuff and thus needs to be a bit larger. Same with the Section Headers. They have different names and the size varies to some degree, but we can ignore that for now. The first obvious difference is in the _Imports_. PE-bear even highlights the list in red. And the list in the lower window is much shorter and looks kinda empty.

![Original Imports]({{site.url}}/assets/lection_3/original_imports.png)<br/>
![Dumped Imports]({{site.url}}/assets/lection_3/dumped_imports.png)

We get the feeling that something with the imports is not right. But wait a second. What are imports you might ask?

# Excursus: Imports

Most - if not all - programs need to communicate with _'the outside world'_ at some point. And by outside world I mean everything that that is beyond basic operations on internal memory. So, for example if a program reads/writes a file, asks the OS what time it is, wants to print something on the screen, reads a user input, requests more memory or simply wants to exit prematurely. All this (and much more) makes it necessary to communicate with the outside world. In most scenarios 'the outside world' is the Operating System (WinAPI), but also calling a method in a DLL - even if the DLL is from the same folder as the exe - leads to 'the outside world' from the point of view of the program.<br/>
Since I assume you have a background in reversing or at least in reading/writing assembler code, I probably don't need to tell you, that in order to get to 'the outside world' you need to either _CALL_ or _JMP_ to a specific memory address. Within the bounds of a program, the target address of a JMP/CALL is determined by either the compiler or linker at compile-/link-time. But what about external calls? Does every linker ship with the exact address of every method of the WinAPI? That sounds unrealistic and impractical. There must be some dynamic process going on that tells the program at runtime where a certain method is.<br/>

![Linked Calls]({{site.url}}/assets/lection_3/imports_simple.png)

But wait a second. Does this mean that the address of the CALL is altered at runtime? Let's have a quick check. For simplicity load the _original.exe_ in your debugger and have a quick look at the CALL to _WriteFile_ at address 0x0040DF05, the call that will print the text to the console. The opcode is _FF 15 78 60 41 00_. Open _original.exe_ in a hex editor and have a look at offset 0xD305 what do you see? It's exactly our code. So that means the assembler code itself is not altered. But then how does the program know the address of _WriteFile_? Well, have a closer look at the instruction. It's not just an ordinary direct jump to an absolute address, but rather an indirect jump that takes whatever address is at 0x00416078 and jumps there. If we have a look at 0x00416078 in the dump view of our debugger, we will notice that the neighbouring DWORDS all look kinda similar.

![Jump table]({{site.url}}/assets/lection_3/iat.png)

They all seem to be addresses in the 0x76A... memory range. Highlight the next 4 bytes in the dump (for me it's _A0 0B AE 76_) and do a rightclick and then _Follow DWORD in Disassembler_. You should land in the kernel32.dll module at address 0x76AE0BA0 at the start of the _GetModuleFileNameW_ method.

![GetModuleFileNameW]({{site.url}}/assets/lection_3/getmodulefilenamew.png)

The next address points to _ExitProcess_ and after that comes _GetModuleHandleExW_.<br/>
Go ahead and try some more addresses and see where they will take you.<br/>
While this itself does not give us any new information, we can start to understand how imports work. What we have found at 0x00416078 seems to be an entry of some sort of array that holds the addresses of external methods and whenever we want to call that method, instead of calling it directly, we instead call the address at a specific index in that array. You can visualize it a bit via the following Code:

```c
typedef void (*FUNCPTR_t)(void);

enum Methods
{
	ID_METHOD_0,
	ID_METHOD_1,
	ID_METHOD_2,

	NUM_METHODS
};

FUNCPTR_t methods[NUM_METHODS];

int main()
{
	// 'methods' gets initialized by someone else, similar to:
    // methods[0] = &method0;
    // methods[1] = &method1;
    // methods[2] = &method2;

	/* Indirect Call method 2 */
	methods[ID_METHOD_2]();

	return 0;
}
```

The question is now, who initializes this array and how does he know which address to put at which index.<br/>
To answer that question, have a look back into PE-bear. Scroll down a bit in the imports and you should find this:

![Imports]({{site.url}}/assets/lection_3/imports.png)

Well, that looks just like the methods we just discovered and also in the exact same order. Also if you add the Image Base Address (0x00400000) to the _Call Via_ address, you end up with the memory address of the indirect call. So it looks like we have found the _how_.<br/>
For a normal, unpacked program the _who_ is actually quite easy to figure out. It's the OS itself and if you think about it, it totally makes sense, since a program can not figure out the addresses on it's own. So whenever we load a program - in a debugger or without - the OS will go through the imports, figure out the addresses and place them in this special array. By the way, the array is called the _Import Address Table (IAT)_ and I will call it by this name from now on. We will not try to understand the exact format of the IAT as of now, but just accept the fact that it holds all the addresses of methods in external modules (DLLs).<br/><br/>

# What's wrong with our imports

Ok, so back on the topic. Since we now know how a good IAT shall look like, we can think about how we can re-construct it. But wait, how did it get corrupted in the first place? After all, we were able to dump the program with all its headers etc., why was the IAT not dumped properly? Well, actually it was dumped properly, but not in the way we expected it. To see this, load _packed.exe_ in PE-bear and have a look at the imports. Then open the dumped exe in PE-bear and also have a look at the imports. Do you see the similarity? We still have 4 imports and they are all at the exact same addresses starting at 0x21028.<br/>
The only main difference is the value under _'Thunk'_, if you look closely you will recognize these values as the addresses from the 0x76A... memory space that we discovered earlier. So that means that during the loading process when the program is loaded from file to memory, the OS overwrites the IAT as found in the file on disc with the module addresses. If we then dump the program from memory back to a file, the original IAT data is lost. So in order to reconstruct the IAT, we need to overwrite the addresses with some data that is meaningful to the loader again. We will find out what this data is in a short second, but in the meantime we have to think about something else.<br/><br/>
Do you remember how we loaded the _original.exe_ in PE-bear earlier? If not, do it again and have a look at the imports. What do you notice? There are much more imports than just 4 - to be precise, PE-bear tells us that 73 methods are imported from KERNEL32.dll.
So what about the other 69 imports in the _packed.exe_ where have they gone? Don't we need them anymore? The answer to this question is not directly clear but if you think about it, you might come to the conclusion, that the _packed.exe_ really only needs a limited set of imports - at least when it starts - because right after the start the packer-code is active and does not need fancy stuff like writing to the console etc. So, the 4 imports we see are not the imports of our unpacked program but still the original imports by the packer.

![Imports]({{site.url}}/assets/lection_3/iat_packer.png)

This means that we must first find the original IAT in the unpacked data, then modify the dumped exe so that it uses the original IAT and finally reconstruct the IAT.<br/><br/>

# Finally: Step 3

This all sounds kinda complicated, but luckily Scylla can do all this for us in just a few clicks.<br/><br/>
To see this in action, load the _packed.exe_ in x64dbg, let it run until the OEP, just as we did in [lection 2](/lections/lection_2) then open Scylla. Dump the process just like before and then press _'IAT Autosearch'_ (if a message pops up, just click _yes_). After that, press _'Get Imports'_ and you should see the 73 imports. Sometimes there are some garbage imports detected, we need to remove them first via clicking _'Show Invalid'_ and then perform a rightlick _'Delete tree node'_ on the invalid import. Once all imports are valid (green or yellow) click on _'Fix Dump'_ and select the previously dumped file. If everything worked well, a new file with the extension '\_SCY' is created. Finally click on _'PE Rebuild'_ and select the file with the '\_SCY' extension to rebuild it.<br/><br/>

That's it. We have successfully unpacked, dumped and fixed our first executeable!<br/><br/>

In the next lection we will find a solution to the following problem: If the OS only loads the packer IAT at program start, but the original IAT is still packed at that time, how can we ever end up with a fully loaded original IAT during runtime? Or in other word, how does the packer load the original IAT on its own?

### Conclusion

* If a program needs to call external methods it does this via the IAT
* The addresses in the IAT are placed there by the OS, overwriting the original content
* In order to restore the original content we need to fix the IAT

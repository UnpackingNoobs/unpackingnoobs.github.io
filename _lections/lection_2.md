---
layout: post
author: UnpackingNoobs
title: "Lection 2: What does it mean to be un-packed?"
date: 2025-05-03
tags:
    - "Binary unpacking"
---

### In this lection we will learn:

* A short definition of what an un-packed binary is
* What the OEP is
* A bit about the PE file structure

### What you need for this lection:

* The examples from the _[Lection1](https://github.com/UnpackingNoobs/Lection1)_ repo
* A debugger of your choice (preferrably [x64dbg](https://x64dbg.com/))
* A tool to edit PE files (preferrably [PE-bear](https://github.com/hasherezade/pe-bear))

# Recap

In [lection 1](/lections/lection_1) we have learned that a packed program consists of a piece of code that is executed right at the start and will somehow 'generate' the real program code, strings and probably much more:

![A sketch on how a packed program works]({{site.url}}/assets/lection_1/packer_simple.png)

We also realized that a packed program hides it's real content and that we can't just use a hex editor to read out strings etc. So we must find a way to restore the program to it's original state.

# Where one thing ends, a new thing starts

To someone who has already a good understanding of the unpacking process the question may sound trivial, but let's think about it for a moment - we're noobs after all: When or shall we say where is the place in the following graphic:

![Original entry point]({{site.url}}/assets/lection_2/entry_point_simple.png)

It's the exact moment when the magic packer-code has done it's thing and has fully generated the real program code and is just about to execute it. If we could stop execution in this very moment we should - at least that's what we can assume - land right at the same place as if we would have started the original program.<br/><br/>

But where is this place you might ask. Well, that's actually the whole point - the holy grail - of the art of unpacking. That, my friend, is called the _OEP (Original Entry Point)_. Once we've found it, we can call it a day and are done with the unpacking process ... well ... not quite! Although we are now left with the original program code, it only lives in memory and if we were to restart the debugger, it would be gone. So we need to perform two additional steps. First, read the program from memory and store it back to the hard drive and second, modify the address of the entry point so that it will start execution at the OEP.

![Unpacking process]({{site.url}}/assets/lection_2/unpacking.png)

There is actually a third step involved, but we will get to that later, for the general understanding we don't need it right now.

# Finding our first OEP

Finding the OEP can be quite challenging sometimes, especially since we usually don't know how it looks like since there is no guaranteed _'you have reached the OEP'_ indicator, but there are some methods that we can use that will guide us in the right directions. We will explore them in the next lections, for now we just want to get a feeling of the steps that need to be done once we have found the OEP.<br/>
As an example we will use the `packed.exe` from lection 1, because it gives us an advantage we usually don't have: access to the `original.exe`.
How can the _original.exe_ be of help you might ask. Well, thats easy: Although we've seen in lection 1 that the packed code starts at a different address (0x00420C80 instead of 0x004012C1), the generated/unpacked code seems to be located at the same address. This means that, according to our theory, once the code is fully unpacked, it will start the execution at the same address entry point address (0x004012C1).<br/>
To see if we are right, place a _hardware breakpoint on execution_ at the address 0x004012C1 and let the program run freely (F9). Once the debugger stops there, re-analyze the code (Ctrl+A) and have a look at the code. Does it look familiar? It should, because it's exactly the same as the code that is found at the entry point of _original.exe_. Well done, you have found your first OEP!<br/>
To safe the program from memory back to disc - we will call it _dumping_ from now on - open Scylla (the little black/red 'S' icon) and click the _'Dump'_ button.<br/>
That's step 1 done.<br/><br/>
You could now try to execute the dumped program, but it will likely crash or won't even start at all. Why is that so? Well first - as we've seen earlier - we need to fix the address of the entry point. For that, open up your PE editor and load the _original.exe_ first, so that we can get a feeling how it should look like.<br/>
Note: At this point I assume that you have seen the general PE structure before, if not - don't worry we'll have a much deeper look in the following lections.<br/>
Under the _Optional Headers_ we will find the Entry Point which is 0x12C1. Note that this is relative to the base address at which the program is loaded to memory (0x00400000), to get the address of the entry point as we will see it later in the debugger, simply add these two values together.<br/>

![Entry point in the PE editor]({{site.url}}/assets/lection_2/pe_editor.png)

You could repeat this process for the _packed.exe_ and would find the previously seen entry point of 0x20C80.<br/>
Now load the dumped exe into the PE editor and have a look at the address of the entry point. Intrestingly it is already at the OEP (0x12C1), so it looks like Scylla has already fixed this for us. That means that Step 2 is also done.<br/>
But what are we missing then? Looks like more fixes need to be done in order to get a fully functional unpacked program. We will see the necessary third step in the next lection.<br/>
But is there actually anything we can do at the moment? Well yes, you could already load the dumped exe to e.g. Ghidra and perform a static analysis of the code or have a look with the hex editor and search for promising strings.<br/><br/>

That's it for now for our second tiny step into the world of unpacking. In the next lection we will try to make the dumped program run again.


### Conclusion

* Once the execution of the packer-stuff is over, the real program gets executed. We call this place the OEP
* Dumping the program and fixing the address of the entry point is not enough

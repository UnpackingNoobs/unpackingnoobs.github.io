---
layout: post
author: UnpackingNoobs
title: "Lection 1: What does it mean to be packed?"
date: 2025-05-02
tags:
    - "Binary unpacking"
---

### In this lection we will learn:

* A short definition of what a packed binary is
* How a packed binary looks like from the outside
* How a packed binary looks like from the inside

### What you need for this lection:

* The examples from the _[Lection1](https://github.com/UnpackingNoobs/Lection1)_ repo
* A hex-editor of your choice
* A debugger of your choice (preferrably [x64dbg](https://x64dbg.com/))

# A first observation

Since the whole series is from a noob for noobs, we will not dive too deep into the topic in this first lection. We will just make some observations and explain the exact reasons later.\
For now, let's just have a look at the two examples provided with the material for this lection. In the _bin_ folder you will find two files called `original.exe` and `packed.exe`.\
Open them up and compare them. Do they behave different? What are their file sizes, are the different?\
You should come to the conclusion that both programs do exactly the same thing, they both ask you for the secret code (which is _123456_ by the way) and give you a Goodboy/Badboy message depending on what you've put in. But why is the packed version only half the size you might ask. Of course there can be many reasons for a different file size like a different compiler, compiler settings, runtime, programming language etc. but that's all not the case here.\
To investigate further, we can open the two programs in a hex editor and search for the strings that are used, e.g. _'Please insert the secret code'_. For the _original.exe_ you should have no problems to locate the string in the file, but what's with the _packed.exe_? The strings are nowhere to be found.<br/>

![The strings can easily be found in the original.exe]({{site.url}}/assets/lection_1/hexeditor.png)

Next, open up your favorite debugger, load the two programs and let them run to the entry point of the program and compare the assembly. That's strange, they look completely different. Also, if you search for string references, you will find nothing for the _packed.exe_. Moreover they seem to start at different addresses (0x004012C1 and 0x00420C80).

![The entry point of original.exe]({{site.url}}/assets/lection_1/ep_original.png)<br/><br/>
![The entry point of packed.exe]({{site.url}}/assets/lection_1/ep_packed.png)

Now let the _packed.exe_ run freely (F9/Play Button in x64dbg) and search for string references again. Well that's interesting - now there are strings. Where did they come from? It looks like they have been created at runtime.<br/>
Now compare the locations where the strings are used in the _original.exe_ and the _packed.exe_ - are they different? You should make the - maybe unexpected - observation that both programs use the exact same code, even the addresses are the same (0x004158C6). How is this possible, just seconds ago it looked like we had two different programs and now they look absolutely identical?<br/>

![The place where the strings are used]({{site.url}}/assets/lection_1/code_unpacked.png)

Make one more observation by remembering the address of the _PUSH_ instruction that pushes the _'Please insert the...'_ string onto the stack (0x004158C6), reset the program (but only run it up until the entry point) and then have a look at the remembered address (Ctrl+G in x64dbg). For the _original.exe_ everything looks just fine, just as we expected it. But what's up with the _packed.exe_ ? The instructions look like garbage and there are bytes that are not even a valid instruction. Let the program run again (F9) and tell your debugger to re-analyze the code and ... boom, the valid code is back.<br/>

![Scrambled data]({{site.url}}/assets/lection_1/code_packed.png)

At this point it should be obvious that somewhere within the program the real instructions have been restored and were placed here. This gives us a first idea of what a packed program looks like and how is behaves: There seems to be some sort of magic code that is executed at the start of the program that will then somehow generate and execute the real program code thus making the program behave just like we would have executed the original version.

![A sketch on how a packed program works]({{site.url}}/assets/lection_1/packer_simple.png)

That's it for now for our first tiny step into the world of unpacking. In the [next lection](/lections/lection_2) we will try to gain an understanding how we can put the _un_ in unpacking.

### What have we observed so far?

A packed program...

* can vary in size compared to the original
* hides it's real content
* will generate it's strings and instructions during runtime
* will have different assembly code (at least at the entry point)
* will in the end still behave just like the original program

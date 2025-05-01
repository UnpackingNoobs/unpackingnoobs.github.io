---
layout: default
---

If you've stubled across this page you might ask yourself _what's that about?_ The answer is simple: It's unpacking (as in unpacking a binary) with noobs ;)

# Background

As someone who grew up in the 90s and early 2000s I lived through the - what I call - _golden age of software piracy_. I fondly remember the moment when you opened up a keygen and were greeted with a catchy chiptune and some nice graphics.
I was always fascinated by people who were capable of the black art of `Reverse Code Engineering`, so naturally I myself started to learn how to code and later started to dive into the world of RCE.
Throughout the years I have written many Keygens and Cracks for my personal pleasure (I see them as elaborated crossword puzzles) but one thing I never fully got into was the even darker art of manually unpacking binaries that were protected, scrambled, encrypted or simply compressed by a packer.
After a few years of RCE-abstinence I recently got interested in the topic again and thought that this time I should focus more on unpacking since it is also of interest in malware analysis.
I'm actually not quite interested in malware analysis, but you never know where life will take you so it's always a good idea to learn something new each day ;)

## My plans for this blog

I try to teach myself how to manually unpack binaries and will document the process via short digestable lections that will hopefully be helpful to anyone.
Along the way I will also explore the different techniques on how packer-authors try to keep little snoopy reversers from reversing their code (anti-debugging-techniques) and how to get around these techniques (anti-anti).
My plan is to also develop my own simple packer to better understand the process of how a program is packed.
Hopefully one day I will be able to unpack a real-life target that uses a Virtual Machine (VM) based packer as this seems to be where all the fun is these days :)

## Target audience

You have a background in reverse engineering, can read a bit of Assembly and C-Code but are an absolute bloody beginner when it comes to unpacking? Nice! This blog will be made for you.

---
layout: post
title:  Getting Acquainted with LLVM Codegen
date:   2024-12-9 15:06:35 -0400
categories: llvm compilers
---

I am currently looking at chapter 3 of the LLVM Kaleidoscope Language tutorial, where a simple language called Kaleidoscope is implemented to learn about how LLVM works. Here is a link to the tutorial: [LLVM Tutorial Chapter 3](https://llvm.org/docs/tutorial/MyFirstLanguageFrontend/LangImpl03.html).

Chapte 3 does an initial codegen pass using the lexer and parser from the previous two chapters. I have written many a lexer and parser, so I wasn't too interested in writing those myself, so it's nice they provide the code and it is easy enough to do some hacking on top of it. This post serves as an extension of the chapter 3 tutorial -- reading code is useful but there is nothing like solving problems on your own. So I added some extra challenges, and if you're reading this and are interested in learning about LLVM, I urge you to try them out yourself! Then I will walk through how I solved each challenge.

## Challenge 1: Reorganize Code 

Take the code from chapter 3 and reorganize it into separate modules. I just prefer it this way to having multiple files. This starting point is available on my github as the "beginnings" branch, which just splits up this code into separate modules, and also gets rid of the global LLVM context, to avoid polluting the global namespace of the program. The build script is in build.sh, and it assumed llvm is installed. https://github.com/artjomPlaunov/kaleidoscope

Get acquainted with the code base, and see you soon for the next challenge, where we will make some extensions to the codegen pass!





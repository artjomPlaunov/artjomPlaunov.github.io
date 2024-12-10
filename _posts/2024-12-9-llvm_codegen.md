---
layout: post
title:  Getting Acquainted with LLVM Codegen
date:   2024-12-9 15:06:35 -0400
categories: llvm compilers
---

Taking a small reprieve from databases while my friend Tony is out of town -- still reading for the Phil Eaton bookclub, though, and I am currently reading the chapter on planning. I have always wanted to get better acquainted with LLVM, since it seems like a powerful tool to have handy; if you can turn source code into LLVM IR, then you have yourself a compiler. 

I am currently looking at chapter 3, which does an initial codegen pass using the lexer and parser from the previous two chapters. I have written many a lexer and parser, so I wasn't too interested in writing those myself, so it's nice they provide the code and it is easy enough to do some hacking on top of it. This post serves as an extension of the chapter 3 tutorial -- reading code is useful but there is nothing like solving problems on your own. So I added some extra challenges, and if you're reading this and are interested in learning about LLVM, I urge you to try them out yourself! Then I will walk through how I solved each challenge. 
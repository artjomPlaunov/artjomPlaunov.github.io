---
layout: post
title:  Getting Acquainted with LLVM, Part 3
date:   2024-12-10 15:06:35 -0400
categories: llvm compilers
---

Before moving onto chapter 4, let's tinker with the code we have so far. I am posting some easy challenges here (and the solutions will be posted as well shortly).

## Challenge 3: Add Unary Negation

This one requires modifying the AST and parser to allow for such a construct; note that we need it to work on expressions in general, for instance -(a + b) can be negated. For simplicity, we can also have a custom negation character such as ~, like in Haskell. So our negation operations will look like ~7, ~(a + ~b), and so forth. Hint: Look at the CreateFNeg in LLVM. 

## Challenge 4: Add exponentiation

May have to look into Intrinsic for this one. 
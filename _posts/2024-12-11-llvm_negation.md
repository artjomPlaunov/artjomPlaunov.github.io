---
layout: post
title:  Getting Acquainted with LLVM, Part 4
date:   2024-12-11 15:06:35 -0400
categories: llvm compilers
---

Part 4 covered two big topics: registering optimization passes and setting up a JIT to compile our code on the fly. 
## Challenge #5 - Find Examples for Optimization Passes. 

We register new passes in the initialization function, as follows:

```cpp
// Add transform passes.
// Do simple "peephole" optimizations and bit-twiddling optzns.
TheFPM->addPass(InstCombinePass());
// Reassociate expressions.
TheFPM->addPass(ReassociatePass());
// Eliminate Common SubExpressions.
TheFPM->addPass(GVNPass());
// Simplify the control flow graph (deleting unreachable blocks, etc).
TheFPM->addPass(SimplifyCFGPass());
```

Find some example programs that showcase each of these passes. Comment out and uncomment to see the difference in the generated code. Here is an example of the common subexpressions pass, with and without the optimization:

```bash
ready> def foo(a b) (a + b) * (b + a);
ready> Read function definition:define double @foo(double %a, double %b) {
entry:
  %addtmp = fadd double %a, %b
  %addtmp1 = fadd double %b, %a
  %multmp = fmul double %addtmp, %addtmp1
  ret double %multmp
}
```

As we can see, a+b is being recomputed twice here. If we turn back on the GVNPass (Global Value Numbering algorithm used for common subexpressions), we get: 

```bash
ready> def foo(a b) (a + b) * (b + a);
ready> Read function definition:define double @foo(double %a, double %b) {
entry:
  %addtmp = fadd double %a, %b
  %multmp = fmul double %addtmp, %addtmp
  ret double %multmp
}
```

We are currently only doing function level passes, which fits well with our interactive REPL/JIT workflow. The optimization is invoked in the FunctionAST codegen function. After the function is constructed and verified, TheFPM->run function is called, which modifies the AST in place with the passes. 

As for the JIT, we are currently concerned with what it is providing for us, not how it actually works yet. We have a template JIT implementation that works for now. It's good to see the abstraction level that the JIT operates, however, by inspecting the API and usage. We first initialize our target for the JIT: 

```cpp
InitializeNativeTarget();
InitializeNativeTargetAsmPrinter();
InitializeNativeTargetAsmParser()
```



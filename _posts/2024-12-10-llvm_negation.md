---
layout: post
title:  Getting Acquainted with LLVM, Part 3
date:   2024-12-10 15:06:35 -0400
categories: llvm compilers
---

Before moving onto chapter 4, let's tinker with the code we have so far. I am posting some easy challenges here (and the solutions will be posted as well shortly).

## Challenge 3: Add Unary Negation

This one requires modifying the AST and parser to allow for such a construct; note that we need it to work on expressions in general, for instance -(a + b) can be negated. For simplicity, we can also have a custom negation character such as ~, like in Haskell. So our negation operations will look like ~7, ~(a + ~b), and so forth. Hint: Look at the CreateFNeg in LLVM. 

First lets add a unary expression node in the AST. This will be useful later on if we want to add more unary operators, and will make it easier to fit into the current parser. Here is the node, modeled after the BinaryExprAST: 

```cpp
class UnaryExprAST : public ExprAST {
    char Op;
    std::unique_ptr<ExprAST> Expr;
public:
    UnaryExprAST(char Op, std::unique_ptr<ExprAST> Expr) 
        : Op(Op), Expr(std::move(Expr)) {}
    llvm::Value *codegen(CodeGen &CG) override;
};
```

Remember I said I didn't want to work on the lexer or parser, but of course we have to extend it as necessary if we want more features. If you look back at the lexer code, the token representation is very simple. There are some custom token types defined as an enum, and all other single character tokens are simply just the same character. If modifying the lexer and parser becomes too unwieldly in the future, I may substitute in a lexer and parser generator, which will require only working with the specification file. I think it's also better to have a custom token type, but we will leave that alone for now. 

This means we don't need to modify the lexer, since it will just return the '~' character as is, and it is up to the parser to see the '~' and parse a unary expression accordingly. 

Now we have to modify the parser. First we create a parsing function for unary expressions, which will accomodate any more unary operators we add in the future: 

```cpp
std::unique_ptr<ExprAST> Parser::ParseUnaryExpr() {
    char op = lexer.getCurrentToken();
    lexer.getNextToken();
    auto Expr = ParsePrimary();
    if (!Expr) {
        return nullptr;
    }
    std::make_unique<UnaryExprAST>(op, std::move(Expr));
}
```

The parser is simple enough: just eat up the operator character, then parse a primary expression, and construct the AST node. We also modify the ParsePrimary code to handle our new negation expression:

```cpp
std::unique_ptr<ExprAST> Parser::ParsePrimary() {
    switch (lexer.getCurrentToken()) {
    default:
        return LogError("unknown token when expecting an expression");
    case tok_identifier:
        return ParseIdentifierExpr();
    case tok_number:
        return ParseNumberExpr();
    case '~': 
        return ParseUnaryExpr();
    case '(':
        return ParseParenExpr();
    }
}
```

Finally, we need to add the codegen method: 

```cpp
Value *UnaryExprAST::codegen(CodeGen &CG) {
    Value *R = Expr->codegen(CG);
    if (!R) {
        return nullptr;
    }
    switch (Op) {
        case '~':
            return CG.getBuilder()->CreateFNeg(R, "negtemp");
        default:
            return CG.LogErrorV("invalid unary operator");
    }
}
```

This is very similar to the BinaryExprAST codegen. Here is our newly added feature in action:

```bash
ready> def foo(a) ~(a + ~a);
ready> Read function definition:define double @foo(double %a) {
entry:
  %negtemp = fneg double %a
  %addtmp = fadd double %a, %negtemp
  %negtemp1 = fneg double %addtmp
  ret double %negtemp1
}
```

## Challenge 4: Add exponentiation

May have to look into Intrinsic for this one. 
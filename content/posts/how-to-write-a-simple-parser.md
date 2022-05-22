---
title: "怎样写一个简单的Parser（1）"
date: 2022-02-22T15:14:45+08:00
draft: false
tags: ["parser", "programming language"]
---

## 介绍
作为一个程序员，多少都有听过"paser"、 "interpreter"、 "complier"这些概念，虽然我们可能没有了解过一个编程语言的parser是怎么实现的，但其实日常编程中我们使用过很多种parser，像JSON的parser、YAML的parser，各个语言中都有相应的实现，它们的工作都是把文本字符串转换成某种数据结构。而计算机语言自身的parser也是一样，它的输入是程序员编写的代码，输出也是一个数据结构：“抽象语法树”（AST, Abstract Syntax Tree）。 和JSON parser的不一样的是，AST对程序员来说可能是一个相对陌生的结构，不像JSON数据一样，可以让我们直观的理解。
比如下面这个简单的表达式，经过parser处理后就可以生产一个对应的AST。

``` txt
                             +
                 Parser     / \
 "1 + 2 * 3"    ------->   1   *
                              / \
                             2   3
```

Python中的ast模块可以用来方便地查看Python代码生成的AST
```Python
In [1]: import ast

In [2]: print(ast.dump(ast.parse('if a>b: c'), indent=4))
Module(
    body=[
        If(
            test=Compare(
                left=Name(id='a', ctx=Load()),
                ops=[
                    Gt()],
                comparators=[
                    Name(id='b', ctx=Load())]),
            body=[
                Expr(
                    value=Name(id='c', ctx=Load()))],
            orelse=[])],
    type_ignores=[])
```

## BNF -- 描述编程语言语法的语法
计算机语言的语法规则通常使用某种上下文无关文法(CFG, Context Free Grammar)的形式语言来描述。最常用的CFG是BNF或者EBNF，BNF最早是John Backus、 Peter Naur 开发用来描述Algol 60语言的，BNF也就是Backus-Naur Form，B和N分别代表他们两个的名字，Form是形式的意思。BNF由一系列规则（更学术的叫法是生成式，Production Rule）组成，比如用来描述类似上面算式"1+2*3"的语法可以被表示为：

```txt
Expr ::= Factor | Expr '+' Factor
Factor ::= Number | Factor '*' Number
Number ::= '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9'
```
左边的Expr、Factor和Number都是非终止符，他们要被替换成右边的形式，而`+`、`*`，数字`1`到`9`都是终止符，终止符不需要再被替换。这样一系列的规则就能够描述一个编程语言涉及到的所有关键字、优先级以及支持的语法。

有了这些规则，我们就可以手写代码来实现一个parser来验证代码是否合法，以及生成AST。计算机科学已经发展了很多年，有很多种成熟的的parser生成器可以拿来用，像是YACC、Bison或者ANTLR等。parser生成器用形式语言描述的规则作为输入，生成一个parser的代码，生成的代码被编译后可以用语言的代码作为输入，输出对应的语法树。

## Parser的解析过程

1. 自顶向下
2. 自底向上

Parser的解析过程有两种：自顶向下和自底向上。自顶向下的的过程更符合我们的思维习惯，只需要模拟按照规则从上往下的不断替换就能解析一个语法。自顶向下的最直观的过程就是递归下降（recursive descent），针对每一个规则提供一个递归的解析函数，如果有多种匹配就尝试其中一个，匹配失败再回溯，它是最通用也是最容易理解的方式，所以手写的parser通常都是用的递归下降的方式来实现。

递归下降可能会有效率的问题，会有多层的递归和回溯。另外一种自顶向下的分析方法叫LL(Left-to-right, Left derivation), 通常它也需要提前向后看k个符号，所以LL文法通常用LL(k)表示，常见的就是LL(1).
LL(1)根据规则通过提前创建一个预测的分析表，消除了回溯，在从上往下分析的过程中，通过匹配分析表而不是回溯的尝试所有可能。

LL文法也有它的限制，它能表达的语法有限，相比之下与之对应的自底向上的分析方法LR（Left-to-right, Rightmost derivation)可以表达更丰富的语法，不过LR的parser相对来说也更难以构造，所以LR的parser都是用生成器自动生成的，像Yacc就是基于LALR文法的parser生成器。LL和LR的区别有点像树的前序遍历和后序遍历。

## 生成器还是手写？

上面提到了实现parser有两种方式：1. 手写，2. parser生成器。虽然生成器很方便，不过并不是所有语言都用生成器来生成parser、解析语法。Python现在用的是[PEG语法](https://devguide.python.org/parser/)的生成器来生成parser。[Rust](https://rustc-dev-guide.rust-lang.org/the-parser.html)使用的是手写的parser。 Go[1.6之前](https://github.com/golang/go/blob/go1.5/src/cmd/compile/internal/gc/y.go)使用的是基于Yacc生成的parser，[1.6之后](https://go-review.googlesource.com/c/go/+/16665/)开始改成了手写的parser。Rob Pike给出的[原因](https://www.reddit.com/r/golang/comments/46bd5h/ama_we_are_the_go_contributors_ask_us_anything/d03zx6f/?context=8&depth=9)是：1. 不需要维护额外的工具和语言，2. 递归下降的parser可以有更好的错误信息，3. 手写的parser也可以做更多的优化，也就更快。

编程语言为了更好的支持各种语法、性能和友好的错误信息，更多的都会选择手写parser。与之相比SQL的parser一般都是自动生成，不管是[MySQL](https://github.com/mysql/mysql-server/blob/8.0/sql/sql_yacc.yy), [PostgreSQL](https://github.com/postgres/postgres/blob/REL_13_STABLE/src/backend/parser/gram.y) 还是[SQLite](https://github.com/sqlite/sqlite/blob/version-3.36.0/src/parse.y)都是使用的基于Yacc的自动生成的parser，当然，这也可能跟它们被开发出来的年代有关。

在[下一篇文章](https://shellfly.org/posts/how-to-write-a-simple-parser-part2/)中，可以看到如何实际的手写一个简单的parser。

Ref:

1. [BNF](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)
2. [Parsing](https://en.wikipedia.org/wiki/Parsing)
3. [LL and LR Parsing Demystified](https://blog.reverberate.org/2013/07/ll-and-lr-parsing-demystified.html)
4. [Parser generators vs. handwritten parsers: surveying major language implementations in 2021](https://notes.eatonphil.com/parser-generators-vs-handwritten-parsers-survey-2021.html)

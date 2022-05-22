---
title: "怎样写一个简单的Parser（2）"
date: 2022-05-22T11:14:45+08:00
draft: false
tags: ["parser", "programming language", "Rust"]
---

## 介绍
在[上一篇文章](https://shellfly.org/posts/how-to-write-a-simple-parser/)中，简单介绍了一下parser的背景知识，这一次我们来手写一个简单的parser，还是用这个简单的例子开始，我们的任务是实现一个paser, 它可以接受输入左边的表达式，生成右边的AST。

``` txt
                             +
                 Parser     / \
 "1 + 2 * 3"    ------->   1   *
                              / \
                             2   3
```
然后我们会继续扩展我们的parser，让它支持前缀表达式和括号，使得它可以解析更复杂的表达式，比如：`( -1 + 2 ) * 3 - -4`

## TODP

Parser的实现的算法是基于TDOP(Top Down Operator Precedence)，这是[Vaughan Pratt](https://en.wikipedia.org/wiki/Vaughan_Pratt)发表于1973年的论文，论文中他提出了一种“易于理解，实现简单，使用方便，非常高效并且足够灵活的适应各种语法需求”的方法，并且提到，这种明显的乌托邦式的方法之所以没有被广泛采用，就是因为大家都关注在BNF以及相关LL，LR文法上，他并不认为这是一个有前途的方向。2007年Douglas Crockford（《JavaScript: The Good Parts》的作者）基于TDOP实现了JSLint的parser，并且写了一篇[blog](http://crockford.com/javascript/tdop/tdop.html)来介绍这个算法，之后开始有更多人关注并使用TDOP来实现parser。

TDOP也是一种递归下降的算法，它的特别之处在于以下几点：
1. 递归的函数和具体的操作符（+-*/、if、else等）关联（而不是像其他基于BNF语法parser，和一条规则关联）
2. 根据操作符的位置，每个符号可以有两种不同的解析方法，在Pratt的论文中称作 `nud`（null denotation）和 `led` （left denotation）， 作为 `nud` 时，操作符不关心左边的符号，用来处理前缀（prefix），`led` 需要关心它左边的符号，用来处理中缀（infix）和后缀（suffix）的情况
3. 每个符号有两个优先级 `lbp` 和 `rbp`，分别表示left binding power和right binding power。比如 `1+2*3` 中， `+` 和 `*` 是符号，2是一个表达式，在解析的时候 `+` 的 `rbp` 小于 `*` 号的 `lbp` 那么 `2` 就应该跟 `*` 先关联
4. `lbp` 由当前符号查询获得，`rbp` 由递归的函数调用传递过来，初始为0


## 代码

代码实现使用的是Rust，选择Rust一是主要因为最近在学Rust，二也是觉得Rust有丰富的类型系统，尤其是Enum，非常适合用来表达这里面的概念，类型也能帮助我们理解数据结构和函数的输入输出。

### 1. 定义数据结构。
第一步我们需要定义数据结构来表示输入的Token，以及最终的AST。为了方便，我们用lisp中的S表达式来代替AST，第一步的任务就是把 `1+2*3` 转换成 `(+ 1 (* 2 3))` 的形式。

``` rust {linenos=table,linenostart=1}
// Token is either a Symbol like "1", "2", or an Op like "+", "*"
enum Token {
    Symbol(String),
    Op(String),
}

// Expr is a lisp S-expression, it's either an atom or a list of atom.
enum Expr {
    Atom(Token),
    Cons(Token, Vec<Expr>),
}

// parse receive an input text and transform it to Expr
fn parse(input: &str) -> Expr {
    todo!()
}

#[test]
fn test_parse() {
    let s = parse("1 + 2 * 3");
    assert_eq!(s.to_string(), "(+ 1 (* 2 3))");

    let s = parse("1 * 2 + 3");
    assert_eq!(s.to_string(), "(+ (* 1 2) 3)");
}
```

### 2. 简单的Lexer。
Parser之前还需要一个Lexing的过程，它负责把输入字符串变成一个一个的符号传给Parser，我们的Lexer比较简单，只需要按照空格把字符串分开即可，为了配合后面的Parser使用，也定义了 `pop` 和 `peek` 方法。

``` rust  {linenos=table,linenostart=1}
struct Lexer {
    tokens: Vec<Token>,
}
impl Lexer {
    fn new(input: &str) -> Lexer {
        let mut tokens = input
            .split_whitespace()
            .map(|c| match c {
                "+" | "-" | "*" | "/" => Token::Op(c.to_string()),
                _ => Token::Symbol(c.to_string()),
            })
            .collect::<Vec<_>>();
        // reverse tokens here so that we can pop out first token without
        // shifting all elements.
        tokens.reverse();
        Lexer { tokens }
    }

    // pop the first token of origin input, from left to right
    fn pop(&mut self) -> Option<Token> {
        self.tokens.pop()
    }
    // check first token without pop out it
    fn peek(&mut self) -> Option<&Token> {
        self.tokens.last()
    }
}
```

### 3. Parse
解析的主要过程是先对输入生成Lexer，然后递归调用 `parse_bp`，解析的过程我们需要比较操作符的优先级，所以也需要实现一个方法用来查询符号的优先级，第一步中我们只支持中缀操作符，所以只实现 `led` 方法，返回操作符的左右优先级。

``` rust {linenos=table,hl_lines=["13-22", "33-37","39", "41-46"],linenostart=1}
// Token is either a Symbol like "1", "2", or an Op like "+", "*"
enum Token {
    Symbol(String),
    Op(String),
}

impl Token {
    // nud return right binding power
    fn nud(&self) -> u8 {
        todo!()
    }
    // led return left and right binding power
    fn led(&self) -> Option<(u8, u8)> {
        match self {
            Token::Op(s) => match s.as_str() {
                "+" | "-" => Some((10, 10)),
                "*" | "/" => Some((20, 20)),
                _ => None,
            },
            _ => None,
        }
    }
}


// parse receive an input text and transform it to Expr
fn parse(input: &str) -> Expr {
    let mut lexer = Lexer::new(input);
    parse_bp(&mut lexer, 0)
}

fn parse_bp(lexer: &mut Lexer, rbp: u8) -> Expr {
    let token = lexer.pop().expect("unexpected eof");
    let mut left = match token {
        Token::Symbol(_) => Expr::Atom(token),
        _ => panic!("bad token: {:?}", token),
    };

    while let Some(token) = lexer.peek() {
        let token = token.clone();
        let (lbp, new_rbp) = token
            .led()
            .unwrap_or_else(|| panic!("unexpected token: {:?}", token));
        if lbp < rbp {
            break;
        }

        lexer.pop(); // pop out operator
        let right = parse_bp(lexer, new_rbp);
        left = Expr::Cons(token, vec![left, right])
    }
    left
}
```
- 13-22: 定义加和减的优先级是10，乘和除的优先级是20
- 33-37: 每次递归调用开始，表达式的开头应该是一个数字（因为我们还没有支持前缀操作符）
- 39: 一直循环直到Lexer中没有剩余的符号
- 41-46: 获取当前符号的 `lbp` 和新的 `rbp`，如果当前符号的优先级更低，则返回left作为上一层的right，否则说明需要和后面的操作关联，递归调用获取 `right` ， 组合Expr，继续查看后面的token

### 4. 通过测试

这个时候第一步的算法已经算实现完了，不过Rust中的结构体没有默认的打印方式，所以我们需要告诉它我们希望输出的格式。

```rust {linenos=table,hl_lines=["1", "10-17", "25-38"],linenostart=1}
use std::fmt;

// Token is either a Symbol like "1", "2", or an Op like "+", "*"
#[derive(Debug, Clone)]
enum Token {
    Symbol(String),
    Op(String),
}

impl fmt::Display for Token {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Token::Symbol(s) => write!(f, "{}", s),
            Token::Op(op) => write!(f, "{}", op),
        }
    }
}

// Expr is a lisp S-expression, it's either an atom or a list of atom.
enum Expr {
    Atom(Token),
    Cons(Token, Vec<Expr>),
}

impl fmt::Display for Expr {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Expr::Atom(t) => write!(f, "{}", t),
            Expr::Cons(head, rest) => {
                write!(f, "({}", head)?;
                for s in rest {
                    write!(f, " {}", s)?
                }
                write!(f, ")")
            }
        }
    }
}

#[test]
fn test_parse() {
    let s = parse("1 + 2 * 3");
    assert_eq!(s.to_string(), "(+ 1 (* 2 3))");

    let s = parse("1 * 2 + 3");
    assert_eq!(s.to_string(), "(+ (* 1 2) 3)");
}
```

### 5. 支持前缀操作符

前缀操作只有向右绑定的优先级，在函数 `nud` 里实现，在我们的语法中，当 `+` 和 `-` 作为前缀时，它们有更高的优先级。

```rust {linenos=table,hl_lines=["4-12", "26-32"],linenostart=1}
...
impl Token {
    // nud return right binding power
    fn nud(&self) -> Option<u8> {
        match self {
            Token::Op(s) => match s.as_str() {
                "+" | "-" => Some(30),
                _ => None,
            },
            _ => None,
        }
    }   
    ...
}
...
// parse receive an input text and transform it to Expr
fn parse(input: &str) -> Expr {
    let mut lexer = Lexer::new(input);
    parse_bp(&mut lexer, 0)
}

fn parse_bp(lexer: &mut Lexer, rbp: u8) -> Expr {
    let token = lexer.pop().expect("unexpected eof");
    let mut left = match token {
        Token::Symbol(_) => Expr::Atom(token),
        Token::Op(_) => {
            let new_rbp = token
                .nud()
                .unwrap_or_else(|| panic!("unexpected prefix op: {:?}", token));
            let right = parse_bp(lexer, new_rbp);
            Expr::Cons(token, vec![right])
        }
    };

    while let Some(token) = lexer.peek() {
        let token = token.clone();
        let (lbp, new_rbp) = token
            .led()
            .unwrap_or_else(|| panic!("unexpected token: {:?}", token));
        if lbp < rbp {
            break;
        }

        lexer.pop(); // pop out operator
        let right = parse_bp(lexer, new_rbp);
        left = Expr::Cons(token, vec![left, right])
    }
    left
}

#[test]
fn test_prefix() {
    let s = parse("-1 * 2 + 3");
    assert_eq!(s.to_string(), "(+ (* (- 1) 2) 3)");
}
```

### 6. 支持括号
当出现左括号时，递归开始新的一组parse过程，出现右括号时，停止递归，返回上层处理。

```rust {linenos=table,hl_lines=["2-13","28-30", "41-44"],linenostart=1}
impl Token {
    fn is_left_paren(&self) -> bool {
        match self {
            Self::Op(op) => op == "(",
            _ => false,
        }
    }
    fn is_right_paren(&self) -> bool {
        match self {
            Self::Op(op) => op == ")",
            _ => false,
        }
    }
    ...
}

// parse receive an input text and transform it to Expr
fn parse(input: &str) -> Expr {
    let mut lexer = Lexer::new(input);
    parse_bp(&mut lexer, 0)
}

fn parse_bp(lexer: &mut Lexer, rbp: u8) -> Expr {
    let token = lexer.pop().expect("unexpected eof");
    let mut left = match token {
        Token::Symbol(_) => Expr::Atom(token),
        Token::Op(_) => {
            if token.is_left_paren() {
                parse_bp(lexer, 0);
            } else {
                let new_rbp = token
                    .nud()
                    .unwrap_or_else(|| panic!("unexpected prefix op: {:?}", token));
                let right = parse_bp(lexer, new_rbp);
                Expr::Cons(token, vec![right])
            }
        }
    };

    while let Some(token) = lexer.peek() {
        if token.is_right_paren() {
            lexer.pop();
            break;
        }
        let token = token.clone();
        let (lbp, new_rbp) = token
            .led()
            .unwrap_or_else(|| panic!("unexpected token: {:?}", token));
        if lbp < rbp {
            break;
        }

        lexer.pop(); // pop out operator
        let right = parse_bp(lexer, new_rbp);
        left = Expr::Cons(token, vec![left, right])
    }
    left
}

#[test]
fn test_group() {
    let s = parse("( -1 + 2 ) * 3");
    assert_eq!(s.to_string(), "(* (+ (- 1) 2) 3)");
}
```

## 总结

完整的代码放在了这个[repo](https://github.com/shellfly/simple-parser/blob/main/src/parser.rs)。相比于基于BNF语法规则的解析器，TDOP确实容易实现，也更容易理解，只需要理解简单的递归，用很简单的代码就实现一个完整的parser。

Ref:
1. [Top Down Operator Precedence](https://tdop.github.io/)
2. [Top Down Operator Precedence by Douglas Crockford](http://crockford.com/javascript/tdop/tdop.html)
3. [Top-Down operator precedence parsing](https://eli.thegreenplace.net/2010/01/02/top-down-operator-precedence-parsing/)
4. [Pratt Parsers: Expression Parsing Made Easy](http://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/)
5. [TDOP / Pratt parser in pictures](http://l-lang.org/blog/TDOP---Pratt-parser-in-pictures/)
6. [Pratt Parsing Index and Updates](http://www.oilshell.org/blog/2017/03/31.html)
7. [Top-down Operator Precedence Parsing in Rust](https://willspeak.me/2016/09/03/top-down-operator-precedence-parsing-in-rust.html)
8. [Simple but Powerful Pratt Parsing](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html)

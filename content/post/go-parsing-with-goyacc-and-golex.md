---
title: "Parsing in Go with golex and goyacc"
date: "2019-07-06"
tags:
- "golang"
draft: true

---

This blog post walks through the use of [golex] and [goyacc] to generate a Go language parser for simple arithmatic
expressions with the goal of providing an end-to-end example of using tools together to parse languages in Go.

* how to wire these two tools together to generate a lexer and a parser
* how to improve the error messaging of the parser
* TODO other things

You can see the finished example...

I assume you already have some familiarity with lexer and parser generators, but [Writing an Interpreter with Lex, Yacc,
and Memphis](http://memphis.compilertools.net/interpreter.html) should give sufficient background.

Until recently, I had not used these a parser generator since college so it is likely that my implementation will not be
optimal. I welcome any feedback in the comments!

---

## Background

Recently I was tasked with creating a parser for a simplified [Lucene
syntax](https://lucene.apache.org/core/2_9_4/queryparsersyntax.html).

While, for such a simple language, I could have written a custom parser, I thought this would be good opportunity to
utilize a [parser generator](https://en.wikipedia.org/wiki/Compiler-compiler) for an easier to maintain implementation
for our team given:

* the clarity in Yacc's description of the language
* less code to maintain

Happily, I discovered that there were already Go implementations of the tools I had used before: [Lex] and [Yacc].
Unhappily, I did not find a complete example of using [`goyacc`](goyacc) and [`golex`](golex) to generata a parse tree.
This post aims to fill that gap.

TODO: some reasons you might not want to use a parser generator

## Example language

For this example, we will generate a parser for a simple arithmetic operations.

To show why you might want to generate a parse tree rather than have the parser do the arithmetic evaluation, we will
also allow for placeholder variables in the expression that are filled in later. Typically the parsing of an expression
is the expensive step so it makes sense to only do this once if you will be evaluating the expression many times.

The operators we will support are:

* `+`: addition
* `-`: subtraction
* `*`: multiplication
* `/`: division

The precedence (or order of operations) will be the typical ordering:

* `*`, `/`
* `+`, `-`

but we will also support the use of `()` to group operations.

Operators will be left-associative (so that, for example, `10 / 5 * 2` would effectively be `(10 / 5) * 2`).

For this example we will build a simple arithmatic expression parser to evaluate expressions such as `4 * (3 + 1) - 2`.
It will generate an intermediate parse tree that will then be walked to perfrom the evaluation.

An example expression would be `10 * x / (5 + y)`.When `x` is `4` and `y` is `3` the result would be `5`.

## Gettng started

To get follow along with the example:

* install [`goyacc`](goyacc) and [`golex`](golex) via `go get modernc.org/golex/lex golang.org/x/tools/cmd/goyacc`
* create a new directory to hold the source code

## Representing the parse tree

We will start by creating types to hold a representation of a parsed arithmetic expression in our language.

Create a file called `expr.go` with the following contents:

```go
package main

import (
	"fmt"
	"math/big"
)

// Expr is a component of an arithmetic expression
type Expr interface {
	Evaluate(variables map[string]*big.Rat) (*big.Rat, []error)
}

// BinaryOp is a binary operation that has a left and a right operand
type BinaryOp int

// A list of the binary operations we support
const (
	BinaryOpAdd BinaryOp = iota
	BinaryOpSubtract
	BinaryOpMultiply
	BinaryOpDivide
)

// BinaryExpr represents a binary expression with two operands and an operator
type BinaryExpr struct {
	BinaryOp BinaryOp
	Left     Expr
	Right    Expr
}

// Evaluate evaluates the binary expression using the given variables and returns the result
//
// All operands are left-associative
func (e BinaryExpr) Evaluate(variables map[string]*big.Rat) (r *big.Rat, errs []error) {
	left, lerrs := e.Left.Evaluate(variables)
	errs = append(errs, lerrs...)

	right, rerrs := e.Right.Evaluate(variables)
	errs = append(errs, rerrs...)

	if len(errs) > 0 {
		return nil, errs
	}

	r = &big.Rat{}

	switch e.BinaryOp {
	case BinaryOpAdd:
		r = r.Add(left, right)
	case BinaryOpSubtract:
		r = r.Sub(left, right)
	case BinaryOpMultiply:
		r = r.Mul(left, right)
	case BinaryOpDivide:
		r = r.Quo(left, right)
	default:
		panic("unknown operation")
	}

	return r, nil
}

// ConstExpr represents a number
type ConstExpr struct {
	r *big.Rat
}

// Evaluate returns the constant
func (e ConstExpr) Evaluate(variables map[string]*big.Rat) (r *big.Rat, errs []error) {
	return e.r, nil
}

// VarExpr represents a variable in the expression to be filled in later
type VarExpr struct {
	name string
}

// Evaluate returns the variable's value or an error if the variable is not defined in the map
func (e VarExpr) Evaluate(variables map[string]*big.Rat) (r *big.Rat, errs []error) {
	r, ok := variables[e.name]
	if !ok {
		return nil, []error{fmt.Errorf("variable %q undefined", e.name)}
	}
	return r, nil
}
```

Here you will see that we model an expression as a simple interface with one method: `Evaluate`.

The implementors are:

* `BinaryExpr`: a binary expression consisting of two operands and an operator
* `ConstExpr`: a number in the expression
* `VarExpr`: a variable in the expression

The use of an interface allows us to neatly build up a parse tree made up `BinaryExpr` where the leaf nodes are either
a `ConstExpr` or a `VarExpr`.

For example, we might represent the expression: `5 + x * 2` as:

```go
expr := BinaryExpr{
	Op:   BinaryOpAdd,
	Left: ConstExpr{r: big.NewRat(5, 1)},
	Right: BinaryExpr{
		Op:    BinaryOpMultiply,
		Left:  VarExpr{name: "x"},
		Right: ConstExpr{r: big.NewRat(2, 1)},
	},
}
```

And evaluate it with a value of `2` for `x` as:

```go
res, errs := expr.Evaluate(map[string]*big.Rat{"x": big.NewRat(2, 1)})
```

## Generating a parser

Now that we have a container to hold our parse tree, `Expr`, we can define our parser specification that constructs this
tree and use [`goyacc`](goyacc) to generate the parser for us.  The specification language is largely the same as that
of [Yacc], but with using Go code where Yacc uses a C dialect.

Create a file called `expr.y` with the following contents:

```yacc
%{
package main

import (
	"math/big"
)
%}


/* define containers to hold the types we will need to pass between rules */
%union {
  expr Expr
  num *big.Rat
  s string
}

/* define the set of tokens in our language */
%token tVAR tNUM tPLUS tMINUS tSTAR tSLASH tLPAREN tRPAREN

/* associate Go types with tokens that hold a value */
%type  <expr> expr
%type  <num>  tNUM
%type  <s>    tVAR

/* define precedence (lowest to highest) and associativity of our operators */
%left tPLUS tMINUS
%left tSTAR tSLASH
%nonassoc tLPAREN tRPAREN

%%

top:
  expr
  {
    /* save our expression for later evaluation */
    yylex.(*lexerWrapper).expr = $1
  }

expr:
  tNUM
  {
    $$ = ConstExpr{r: $1}
  }
| tVAR
  {
    $$ = VarExpr{name: $1}
  }
| expr tPLUS expr
  {
    $$ = BinaryExpr{BinaryOp: BinaryOpAdd, Left: $1, Right: $3}
  }
| expr tMINUS expr
  {
    $$ = BinaryExpr{BinaryOp: BinaryOpSubtract, Left: $1, Right: $3}
  }
| expr tSTAR expr
  {
    $$ = BinaryExpr{BinaryOp: BinaryOpMultiply, Left: $1, Right: $3}
  }
| expr tSLASH expr
  {
    $$ = BinaryExpr{BinaryOp: BinaryOpDivide, Left: $1, Right: $3}
  }
| tLPAREN expr tRPAREN
  {
    $$ = $2
  }
```

Here we define a simple grammar that builds up a parse tree (`Expr`). The top rule assigns this expression to a field on
the lexer. I found this necessary to decouple the parsing and evaluation of an expression.

We can now generate the parser using `goyacc -o expr.y.go expr.y` to create `expr.y.go`. If you open this file up, you
will see the generated parser. You typically won't need to look at this file, but the most important part is the
generated `yyParse` function:

```golang
func yyParse(yylex yyLexer) int {
	return yyNewParser().Parse(yylex)
}
```

This is the entrypoint for the parser and what we will call to do the actual parsing. It takes a `yyLexer` which is an
interface it defines for what it needs from a lexer:

```golang
type yyLexer interface {
	Lex(lval *yySymType) int
	Error(s string)
}
```

We will use [`golex`](golex) to generate a lexer matching this interface.

Some other notable parts of this generated file are:

### Token constants

Towards the top you will see that it has defined constants for our tokens:

```golang
const tVAR = 57346
const tNUM = 57347
const tPLUS = 57348
const tMINUS = 57349
const tSTAR = 57350
const tSLASH = 57351
const tLPAREN = 57352
const tRPAREN = 57353
```

We will use these constants when generating our lexer so that we return the values that the parser is expecting.

### Symbol values

You will also find a definition of `yySymType`:

```golang
type yySymType struct {
	yys  int
	expr Expr
	num  *big.Rat
	s    string
}
```

created from our `%union` definition. These field names (which are the same as the names in the `%union`), will also be
important in returning the value to the parser from the lexer.

<aside class="notice">
When creating your own grammar you might run into
[conflicts](https://docs.oracle.com/cd/E19504-01/802-5880/6i9k05dh2/index.html) if it is ambiguous (multiple rules could
be applied at a given stage). `goyacc` deals with
these deterministically (shifting for shift-reduce, using the earliest defined rule for reduce-reduce), but you will
probably want to run modify your grammar to be unambigious to be easier to reason about. You can `goyacc` with `-v
y.output` to see the generated parse tables which will tell you which rules it had trouble disambiguating.
unambigious.
</aside>

## Generating a lexer

Now that we have generated a parser, we need to create a lexer that we can give it to tokenize the input stream. For
this, we will use [`golex`](golex) to generate a parser that satisfies the `yyLexer` interface.

Create a file named `expr.l` with the following contents:

```lex
%{
package main

import (
	"bytes"
	"fmt"
	"math/big"
)

const (
	// 0 is expected by the goyacc generated parser to indicate EOF
	eof = 0
)

// lexer holds the state of the lexer
type lexer struct {
	src *bytes.Reader

	buf     []byte
	current byte
	pos     int

	errs []string
}

func newLexer(b []byte) *lexer {
	l := &lexer{
		src: bytes.NewReader(b),
	}
	// queue up a byte
	l.next()
	return l
}

func (l *lexer) Error(s string) {
	l.errs = append(l.errs, fmt.Sprintf("%s at character %d", s, l.pos))
}

func (l *lexer) next() {
	if l.current != 0 {
		l.buf = append(l.buf, l.current)
	}
	l.current = 0
	if b, err := l.src.ReadByte(); err == nil {
		l.current = b
	}
	l.pos++
}

func (l *lexer) Lex(lval *yySymType) int {
%}

/* give some regular expressions more semantic names for use below */

/* tell golex how to determine the current byte */
%yyc l.current
/* tell golex how to advance to the next byte */
%yyn l.next()

%%
	// runs before each token is parsed
	l.buf = l.buf[:0]

[ \t]+
  // ignore whitespace

[a-zA-Z]+
  lval.s = string(l.buf)
  return tVAR

(\*?)[0-9]+(\.[0-9]+)?
	lval.num = &big.Rat{}
	_, ok := lval.num.SetString(string(l.buf))
	if !ok {
    l.Error(fmt.Sprintf("bad number %q", string(l.buf)))
	}
  return tNUM

[+]
  return tPLUS

[-]
  return tMINUS

[*]
  return tSTAR

[/]
  return tSLASH

[(]
  return tLPAREN

[)]
  return tRPAREN

.
  /* catch all for other tokens */
  return int(l.buf[0])

\0
	return eof

%%

// should never get here
panic("scanner internal error")
}
```

Here we create a lexer that

TODO:

* main.go
* Ask Mark to review
* Note that Golang used to use goyacc
* Link to recursive generator podcast
* This is similar to the example shipped with `goyacc` but with the exception of using `lex` to generate the lexer and creating a parse tree
* YACC syntax highlighting


[golex]: https://godoc.org/modernc.org/golex/lex
[goyacc]: https://godoc.org/golang.org/x/tools/cmd/goyacc
[Lex]: http://dinosaur.compilertools.net/lex/index.html
[Yacc]: http://dinosaur.compilertools.net/yacc/index.html

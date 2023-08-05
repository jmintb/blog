+++
title = "Building a programming language part 1: Hello World!"
date = 2023-07-17
+++

# Building a programming language with Rust and MLIR part 1: hello world

TODO: disclaimer I am not an expert blablabla

## Overview

This blog post is the first in (hopefully) a series on building and designing a programming language. For now the primary
motivation is to satisfy an urge I have had for a while. To dig my teeth into language and compiler design/implementation.
As well as try to solve issues I have seen in other programming languages and experiment with novel concepts
within the space. I will make a separate post motivating the language, for now lets focus on getting started.

The first couple of posts will serve as a sort of tutorial/introduction to the practical and theoretical aspects.
Once the technical foundations have been set, I will start desigining and implementing more involved aspects. 

I hope you stay for the ride :)

Today's goal is to compile and run the following code and have it print to stdout:

```rust
println("Hello World!");
```

## An introduction to compiling programming languages

Before we start it is important to have some intuition around the problem space. I assume that you have experience with using one. If you do not then you have my respect for starting your journey with how to build a programming language.

We will focus on compiling and not interpreting with a [just in time(JIT)](https://en.wikipedia.org/wiki/Just-in-time_compilation) option for dynamic purposes. Lets build an intuition around compilers.

Often and in the case of our compiler the goal is to take input often in the form of a progam and lower it to a different 
and hopefully well optimized representation that can be used in context, for general purpose programming languages this
 will often be some type of machine code that the target environment can execute like an X86 processor or virtual macine 
byte code. 

This will usualy invovle parsing the input to build an abstract syntax tree (AST), lowering to some intermediate representation or multiple intermerdiate representation until we end up with a format executing in our environment. 

Lets start at the start.

## The parser

The first step in our compiler pipeline is the parser. This step will build an AST and verify that the syntax is valid. 
We could implement our own lexer, and for educational purposes I think that is a good exercise. However I will be using
the PEG parsing crate [pest](https://pest.rs/), it is an incredible tool for generating a parser based on a PEG grammar in Rust.
This also forces us to provide a specification for our language in the form of a grammar, win win. The grammar syntax
can be a bit daunting at first. Lets develop at simple grammar that will be suffcient.

To print hello world we need a few pieces are syntax in out grammar. Function calls and quoted strings. For now print will be a built in provided by the compiler.
PEST provides helpfull builtins for common string related cases.

Lets define strings first, we start with the string defined for pest's JSON [example](https://pest.rs/book/examples/json.html) and strip it down a bit.

```
// grammar.pest

char = {
    !("\"" | "\\") ~ ANY
}

inner = @{ char* }

string = { "\""	~ inner ~ "\"" }

```


Pest has a rich syntax for specifing 
the grammar but unless you are in the habit of writig grammars often you will want to check out the grammar chapter of the pest book 
[here](https://pest.rs/book/grammars/grammars.html). 

We now have two rules, one for single characters `char` and one for a double quoted string `string`.
We won't cover everything pest can do but breaking down the grammar for today hopefully provides a good starting point. The char rule 
allows any characters except `"` and `\`. Looking at the char rule from left to write the `!` acts as a negation, `( "\"" | "\\" )` makes our parser first try to match `"` and then afterwards if that fails "\". This uses the [choice](https://pest.rs/book/grammars/syntax.html#ordered-choice) operator. It is important to remember that the ordering matters. The surrounding parantethese are important to not include the fol, this uses the [negative predicate](https://pest.rs/book/grammars/syntax.html#predicates). This tells our parse that is the no matches are found inside the parentheses, consume one character which in the `char` can be anything denoted by the builtin `ANY` rule.

 TODO: explain why we are  escaping these characters.


A string contains an arbitrary number of characters, therefore the `inner` rule denotes exactly that with the `char*`. The `@` marks this rule as [atomic](https://pest.rs/book/grammars/syntax.html#atomic). Atomic rules have the following special properties:
1. `~` means immediately follow by. 
2. Repetition operators (* and +) have no implicit separation.
3. Rules inside an atomic rule are treated as atomic.

This is practical for parsing strings as we do not want to produce a token for each individual character. This also removes the implicit
allowing of whitespace that normal rules have with `~`. 

We now finally get to the `string` rule. A string is delimited by `"`, therefore our rule does exactly that 
`string = { "\""	~ inner ~ "\"" }'.  We look for a pair of `"` with an `inner` between then.

Lets fire up Rust and test our grammar.

Init a cargo project, if you have not already and add the [`pest`](https://pest.rs) dependecies:

```sh
cargo init --bin somelang 
cargo add pest pest_derive 
```
The `cargo add` command can be installed with [`cargo-edit`](https://github.com/killercup/cargo-edit), other wise just add the dependencies manually to `Cargo.toml` as shown below :)

You `Cargo.toml` should look like this:
```toml
[dependencies]
pest = "2.6"
pest_derive = "2.6"
```



We can now jump into `src/main.rs` and use our parser. To get an intuition of what is happening, lets start by printing what a parsed string looks like.

Remember to put the grammar defined earlier into `grammar.pest` in the same directory as `Cargo.toml`. You directory should look like this:

```
Cargo.toml
grammar.pest
src/main.rs
```

In `main.rs` we:  

```rust

// Add the pest depdendencies
use pest::Parser;
use pest_derive::Parser;


// Declare our Parser struct and derive the pest Parser trait.
// As well as reference the file containing our grammar.
#[derive(Parser)]
#[grammar = "grammar.pest"]
pub struct LangParser;

fn main() {
    let input = "\"hello world\"".to_string();

    // This parses the "hello world" string using the string rule.
    let parsed_input = LangParser::parse(Rule::string, &input).unwrap();

    println!("{parsed_input:?}");
}

```

Running `cargo run` should produce the following output, note that I have formatted the output abit to make it easier to read it should appear on one long tine in the terminal output.

```
[Pair { 
   rule: string, 
   span: Span { str: "\"hello world\"", start: 0, end: 13 }, 
   inner: [Pair { rule: inner, 
                  span: Span { str: "hello world", start: 1, end: 12 }, 
                  inner: [] }] 
       }]
```

Here we can see the [Pair](https://docs.rs/pest/latest/pest/iterators/struct.Pair.html) which represents a matching pair of tokens and everything inbetween. In our case `"` is the token
matched by the first Pair, see `span: Span { str: "\"hello world\"", start: 0, end: 13 }`. The inner field contains what ever the tokens spanned. In this case Another `Pair`, spanning the actual 
 string contents. In this case the matching token pair is `h` and `d`. This might seem a bit counter intutive but our string contains any number of `CHAR` due to `inner = { CHAR* }` and char matches
 on almostany character `CHAR = { !("\"" | "\\") ~ ANY }`. Therefore an important point is that the matching tokens for a `Pair` do not need to be the same, just match the same rule.  

### Function calls

Next up let us expand our grammar to support function calls with a single input value. This will be the last addition required to print `Hello World!`.
We will start by extending our input in `src/main.rs`.

```rust
    ...
    let input = "print(\"hello world\")".to_string();
    ...
```

Now if we run `cargo run` we will get an error as our grammar does not support function calls yet. Lets fix that!

We will add 3 new rules, first  `val = _{ string }`. `val` represents a value, right now we only support string values
so val only contains the `string` rule. `val` is also silent using `_{ }` as we are not interested in parseing the `val` rule, we just need a
general wy of representing values in our grammar.  This is importing for calling the print function as we need to parse the desired output in 
as a string value.


`id = { ASCII_ALPHA+  }` will represent an identifier of  variables,  functions, modules etc. `print` will be an identifer for example. `ASCII_ALPHA+` indicates matches on one or more
letters from a-z and A-Z. In future iterations we want to be support more characters but this will do for today.  

Finally `call = { id ~  "(" ~ val  ~ ")" }`, which matches on a function call with an identifer followed by braces with a single value argument. Exactly what we need :fireworks:.

`grammar.pest` should now look like this.

```
// grammar.pest

char = {
    !("\"" | "\\" ) ~ ANY
}

inner = @{ char* }

string = { "\""	~ inner ~ "\"" }

val = _{ string }

id = { ASCII_ALPHA*  }

call = { id ~  "(" ~ val  ~ ")" }

```

Lets take it for a spin!. If you run `cargo run` now you should still get an error, as we have not updated which rule we use to parse this.
In `src/main.rs` update the parsing line to use `Rule::call`. 

```rust
    // src/main.rs
    ...
    let parsed_input = LangParser::parse(Rule::call, &input).unwrap();
    ...
```

Now you should be able to run `cargo run` and get the parsed function call like this:

```shell
[
    Pair {
        rule: call,
        span: Span {
            str: "print(\"hello world\")",
            start: 0,
            end: 20,
        },
        inner: [
            Pair {
                rule: id,
                span: Span {
                    str: "print",
                    start: 0,
                    end: 5,
                },
                inner: [],
            },
            Pair {
                rule: string,
                span: Span {
                    str: "\"hello world\"",
                    start: 6,
                    end: 19,
                },
                inner: [
                    Pair {
                        rule: inner,
                        span: Span {
                            str: "hello world",
                            start: 7,
                            end: 18,
                        },
                        inner: [],
                    },
                ],
            },
        ],
    },
]
```

And with that we are finished with the grammar(for now :) ) and can move to implementing support for this in our compiler.

## Compiler

We are now able to parse `print("Hello World!")` but can not compile anything yet.  We will start by implementing a builtin print function.
You can think of this as a sort of standard library provided print function. First we need to get some preliminaries in order.

### Preliminaries

Before we dive in we need to establish a bit of intuition and setup our projet to work with [MLIR](https://mlir.llvm.org/) and [LLVM](https://llvm.org/). 
What we want to accomplish here is to produce an executable machine code from `print("Hello World!")`. This involes many steps of lower the representation
a user of our language sees to something the hardware can execute. That as where compiler backends can do some of the heavy lifting. Instead of targeting marchine
code our compiler can act as a frontend and target the intermediate representation(IR) of a backend such as [LLVM](todo) and the backend can then handle the heavy lifting 
of optimizing and target specific hardware. This has many upsides and downsides but for bootstrapping a new language in a single blog post it is very helpful :). So what
about MLIR? Well MLIR is developed by the LLVM project and attempts to address some of the limitations of LLVM, such as better support for parallell compilation and better
support for heterogenus compute. Heterogenuis compute is a fancy way of saying multiple different types of hardware, like if you want to build a language that targets both
CPUs and GPU, which I do :eyes". MLIR stands for Multi-Level intermediate representation and if mean to replace the need for language to have their own IR. I will dive into
the technical and theoretical side of this in a separate post as there is a lot to cover. Hopefully this gave you an idea of why we are using this approach. If not let me know
and I will try to elaborate. Otherwise don't worry even if this doesn't make complete sense following along with the implementation should be just fine anyways and might be a better
way to learn what I just tried to explain haha.
### The print function

Now the question how do we want to implement print? And how does printing even work?. Printing works by writing to standard out. Standard out differs 
depending on the environment our code is executed in but we can think of it as a pointer to a file we can keep writing to and whatever is controller standard out,
for example our terminal we read from stdout and display it on the screen.

 One option would be to use `printf` from the os provided c library. 
That would work just fine but I was curious how exactly to get a hold of standard out directly, so we will be implement this our selves. It is relatively simple once you
understand what is happening so it will not expand the implementation by much.


+++
title = "Building a programming language part 1: Hello World!"
date = 2023-07-17
+++

# Building a programming language with Rust and MLIR part 1: hello world

Before diving in, I want to make it clear that I am not(yet) an expert in compiler and language development. I have a bit of experience
from my bachelor project which invovled implementing a JIT compiler for a language developed by my university. This experience made me hungry
for more and served as inpsiration for all sorts of ideas I want to try out. I will make plenty of mistakes and have to learn along the way.
By documentating the process you can learn from my mistakes as well :)

## Overview

This blog post is the first in (hopefully) a series on building and designing a programming language. For now the primary
motivation is to satisfy an urge I have had for a while. To dig my teeth into language and compiler design/implementation.
As well as try to solve issues I have seen in other programming languages and experiment with novel concepts
within the space. I will make a separate post properly motivate the language I intend to build, for now lets focus on getting started.

The first couple of posts will serve as a sort of tutorial/introduction to the practical and theoretical aspects.
Once the technical foundations have been set, I will start desigining and implementing more involved aspects. 

I hope you stay for the journey :)

The first milestone is very simple and mostly serves as a way to get our project up and running. We will start by 
building a minimal compiler that can compile: 

```rust
println("Hello World!");
```

and have "Hellp World!" output to standard out.

This should be conceptually simple enough to wrap our heads around but invovled enough to kickstart the project and get the technical requirements inplace.

This first post will focus on the defining a grammar needed for this and generating a parser. We will also go over some general intution around compiler development.

Lets go!

## An introduction to compiling programming languages

Before we start it is important to have some intuition around the problem space. I assume that you have experience with using one. 
If you do not then you have my respect for starting your journey with how to build a programming language.

We will focus on compiling. I am confident that a [just in time(JIT)](https://en.wikipedia.org/wiki/Just-in-time_compilation) option will be sufficient to support dynamic/scripting purposes. 
This will also force us to keep compile times low, win win.
Lets build an intuition around compilers.

Often and in the case of our compiler the goal is to take input often in the form of a progam and lower it to a different 
and hopefully well optimized representation that can be executed in some target environment. For general purpose programming languages this
 will often be some type of machine code that the target environment can execute like an X86 processor or virtual machine like the [JVM](TODO).  

This will usualy invovle parsing the input to build an abstract syntax tree (AST), lowering to some intermediate representation or multiple intermerdiate representations until we end up with a
representation the target environment can execute. 

We will start with the beginning, parsing our input.

## The parser

The first step in our compiler pipeline is the parser. This step will build an AST and verify that the syntax is valid. 
We could implement our own lexer, and for educational purposes I think that is a good exercise. However I will be using
the PEG parsing crate [pest](https://pest.rs/), it is an incredible tool for generating a parser based on a PEG grammar in Rust.
This also forces us to provide a specification for our language in the form of a grammar, win win. The grammar syntax
can be a bit daunting at first. Lets develop at simple grammar that will be sufficient.

To parse `println("Hello World!")` we need a few pieces of syntax in our grammar. Function calls, identifiers and values in the form of strings. 
For now print will be a built in provided by the compiler, so we do not need syntax for declaring functions, only calling them.

Pest provides helpfull builtins for common string related cases.

Lets define strings first, we will start with the string defined for pest's JSON [example](https://pest.rs/book/examples/json.html) and strip it down a bit.

Our grammar now looks like this, we will initialise a Rust project in a moment:

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

We now have three rules, one for single characters `char`, one for the string contents `inner` and one for the whole string surrounded by double quotes `string`.
We won't cover everything pest can do but breaking down the grammar for hopefully provides a good starting point. 

The `char` rule uses the [`any character but`](https://pest.rs/book/grammars/syntax.html#predicates) idiom. In our case this means if the following characters is not `"` or `\` consume one character.
Looking at the char rule from left to write the `!` acts as a negation, so `!( "\"" | "\\" )` says reject the patterns in the parenthses, where `|` is the 
[choice operator](https://pest.rs/book/grammars/syntax.html?highlight=choice#ordered-choice). This operator will try to match on the provided options from left to right. Finally if the pattern is rejected 
a single character is consumed using the builtin consumes one character denoted by the builtin [`ANY` rule](https://pest.rs/book/grammars/built-ins.html).

If you are wondering why these characters are not allowed, `"` is used to define our strings and thus can not be included without being escaped and `\` will be used to escape characters.
Escaping these characters is not strictly necessary just yet, but it sets the foundation nicely and allows us to try more of pest so why not. 

A string contains an arbitrary number of characters, therefore the `inner` rule denotes exactly that with `char*`, where `*` indicates zero or more. The `@` marks this rule 
as [atomic](https://pest.rs/book/grammars/syntax.html#atomic). Atomic rules have the following special properties:
1. `~` means immediately follow by, so no whitespace. 
2. Repetition operators (* and +) have no implicit separation.
3. Rules inside an atomic rule are treated as atomic.

This is practical for parsing strings as we do not want to produce a token for each individual character. This also removes the implicit
allowing of whitespace that normal rules have with `~`. 

We now finally get to the `string` rule. A string is delimited by `"`, therefore our rule does exactly that 
`string = { "\""	~ inner ~ "\"" }'.  We look for a pair of `"` with an `inner` between then.

Lets write some Rust and test our grammar.

Init a cargo project, if you have not already and add the [`pest`](https://pest.rs) dependecies:

```sh
cargo init --bin somelang 
cargo add pest pest_derive 
```

I believe that `cargo add` is a builtin option now but if you are on an older version of cargo it can be installed with [`cargo-edit`](https://github.com/killercup/cargo-edit). 
Other wise just add the dependencies manually to `Cargo.toml` as shown below :)

You `Cargo.toml` should look like this:
```toml
[dependencies]
pest = "2.6"
pest_derive = "2.6"
```

We can now jump into `src/main.rs` and use our parser. To get an intuition of what is happening, lets start by printing what a parsed string looks like.

Remember to put the grammar defined earlier into `grammar.pest` in the same directory as `Cargo.toml`. You directory should have the following structure:

```
Cargo.toml
Cargo.lock
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

    // We use #? to pretty the output for easy reading.
    println!("{parsed_input:#?}");
}

```

Running `cargo run` should produce the following output.

```
[
    Pair {
        rule: string,
        span: Span {
            str: "\"hello world\"",
            start: 0,
            end: 13,
        },
        inner: [
            Pair {
                rule: inner,
                span: Span {
                    str: "hello world",
                    start: 1,
                    end: 12,
                },
                inner: [],
            },
        ],
    },
]
```

Here we can see the [Pair](https://docs.rs/pest/latest/pest/iterators/struct.Pair.html) which represents a matching pair of tokens and everything in between. In our case `"` is the token
matched by the first Pair, see `span: Span { str: "\"hello world\"", start: 0, end: 13 }`. The inner field contains what ever the tokens spanned. In this case another `Pair`, spanning the actual 
 string contents. In this case the matching token pair is `h` and `d`. This might seem a bit counter intutive but our string contains any number of `CHAR` due to `inner = { CHAR* }` and char matches
 on almost any character. An important objservation here is that matching tokens for a `Pair` do not need to be the same, only match the same rule. 

Note how the `inner` does not contain any `CHAR` rules but just the characters it spans. This is due the fact that `inner` is [atomic](https://pest.rs/book/grammars/syntax.html#atomic) and does not permit 
whitespace between. `string` on the other hand is a [compound atomic](https://pest.rs/book/grammars/syntax.html#atomic) rule. 
It therefore will have inner rules but still does not permit any whitespace between matches.  


### Function calls

Next up let us expand our grammar to support function calls with a single input value. This will be the last addition required to print `Hello World!`.
We will start by extending our input in `src/main.rs`.

```rust
    ...
    let input = "print(\"hello world\")".to_string();
    ...
```

Now if we run `cargo run` we will get an error as our grammar does not support function calls yet. Lets fix that!

We will add 3 new rules, first  `val = _{ string }`. `val` represents a value. Right now we only support string values
so `val` only contains the `string` rule. `val` is also silent using denoted by u `_`, as it is just a way of representing values in our grammar, we only want to parse it's contents. 
In our sample input `"Hello World!" is a value.

`id = { ASCII_ALPHA+  }` will represent an identifier of variables, functions, modules etc. `print` is identifer. `ASCII_ALPHA+` indicates matches on one or more
letters from a-z and A-Z. In future iterations we want to be support more characters but this will do for today. 

Finally `call = { id ~  "(" ~ val  ~ ")" }`, which matches on a function call containing a single argument. `call` concists of an identifer followed by a single argument surrouned by parantethes. 
Exactly what we need :fireworks:. We also need to make this rule compound atomic as we do not want whitespace between the identifer and parantethese by we do want to parse the inner rules.

`call = ${ id ~ "(" ~ val ~ ")" }`, there we go.

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

call = ${ id ~  "(" ~ val  ~ ")" }

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

As the final step we will extract the identifier and argument. We can traverse the parsed input as nested iterators.


```rust
// src/main.rs
...

    // Get an iterator over the pairs matched by Rule::call
    let mut call_pairs = parsed_input.into_iter().next().unwrap().into_inner();

    // We know the id is comes first.
    let id = call_pairs.next().unwrap();

    // And the function argument second.
    // We extract the `inner` rule inside the matched `string`.
    let argument = call_pairs.next().unwrap().into_inner();

    println!("id: {}, argument: {}", id.as_str(), argument.as_str());

}

 ```

`cargo run` should now produce:

```sh
id: print, argument: hello world
```

Note how `hello world` does not have surrounding quotes. This is because we went one level deeper. If we stoppe at  `argument = call_pairs.next().unwrap` we woud have the `string` rule which
contains `"`. By callingo `into_inner` we go one rule deeper and extract the matched `inner` which only holds the contents of a string. 

Traversing our parsed input in this manner will allow us to build a propper AST one we are ready. I hope this example showed how powerful and simple [pest](todo) allows us to get started parsing.


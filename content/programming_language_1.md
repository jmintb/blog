+++
title = "Building a programming language Rust and MLIR part 1: pest the parser"
date = 2023-07-17
+++

Before diving in, I want to make it clear that I am not(yet) an expert in compiler and language development. I have a bit of experience
from my bachelor project which invovled implementing a JIT compiler for a language developed by my university. This experience made me hungry
for more and served as inpsiration for all sorts of ideas I want to try out. I will make plenty of mistakes and have to learn along the way.
By documentating the process you can learn from my mistakes as well :)

## Overview

This blog post is the first in (hopefully) a series on building and designing a programming language. For now the primary
motivation is to satisfy an urge I have To dig my teeth into language + compiler design and implementation.
I have some existing ideas for the language I plan to build, those will get a separate post. for I will focus on getting 
the practical setup out of the way.

The first couple of posts will serve as a sort of tutorial/introduction to the practical and theoretical aspects using the tools I have chosen (Rust, MLIR and pest).
Once the foundations have been set, I will start desigining and implementing more involved aspects specific to the my language. I really need to find a name. 

I hope you stay for the journey :)

The first milestone is very simple and mostly serves as a way to get our project up and running. We will start by 
building a minimal compiler that can compile: 

```rust
println("Hello World!");
```

and have "Hellp World!" output to standard out.

This should be conceptually simple enough to wrap our heads around but invovled enough to kickstart the project and get our compiler stack setup.

This first post will focus on the defining a grammar powerful enough for `prinln("Hello World!")` and generating a parser. A small introduction to compiler development
will precede the parser development, to help provide general compiler intuition for the uninitiated.

Lets go!

## An introduction to compiling programming languages

Before we start it is important to have some intuition around the problem space. I assume that you have experience with using a programming language. 
If not then you have my respect for starting your journey with how to build a programming language!

Both compiled and interpreted languages are quite common today. For example Python is a popular interpreted language and Rust is a compiled language.
We won't go deep into interpreted languages but the short version is, instead of preparing machine instructions a head of time, they are generated on
the fly as new code is fed to the interpreter. A longer explanation is found [here](https://en.wikipedia.org/wiki/Interpreter_(computing). One advantage
here is that code can be run immediately without the need to invoke a compiler. Two major disadvantages are poor performance and no compile time checks, for
example like checking types. 

We will be focusing on the design and implementation of a compiled language.
I am confident that a [just in time(JIT)](https://en.wikipedia.org/wiki/Just-in-time_compilation) option will be 
sufficient to support dynamic/scripting purposes. This will also force us to keep compile times low, win win.

What is a compiler?

The goal is to take input, often in the form of high level progam, think Rust, C, Java etc and lower it to a different 
and hopefully well optimized representation that can be executed in some target environment. For general purpose programming languages this
 will often be some type of machine code that the target environment can execute like an X86 processor or virtual machine like the [JVM](https://en.wikipedia.org/wiki/Java_virtual_machine).  

This will usualy invovle parsing the input to build an abstract syntax tree (AST), lowering to some intermediate representation or multiple intermerdiate representations 
until we end up with a representation the target environment can execute. 

We will start with at the beginning, parsing our input.

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

string = ${ "\""	~ inner ~ "\"" }

```

Pest has a rich syntax for specifing 
grammar. Unless you are in the habit of writig grammars often you will want to check out the the pest book 
[here](https://pest.rs/book/grammars/grammars.html) to familiarize your self with the options. 

We now have three rules, one for single characters `char`, one for the string contents `inner` and one for the whole string surrounded by double quotes `string`.
We won't cover everything pest can do but breaking down our grammar hopefully provides a good starting point. 

The `char` rule uses the [`any character but`](https://pest.rs/book/grammars/syntax.html#predicates) idiom. 
In our case this means if the following characters is not `"` or `\` consume one character.
Looking at the char rule from left to write the `!` acts as a negation, so `!( "\"" | "\\" )` says reject the patterns in the parenthses, where `|` is the 
[choice operator](https://pest.rs/book/grammars/syntax.html?highlight=choice#ordered-choice). This operator will try to match on the provided options from left to right. Finally if the pattern is rejected 
a single character is consumed using the builtin consumes one character denoted by the builtin [`ANY` rule](https://pest.rs/book/grammars/built-ins.html).

If you are wondering why these characters are not allowed, `"` is used to define our strings and thus can not be included without being escaped 
and `\` will be used to escape characters.
Escaping these characters is not strictly necessary yet. However it does set a nice foundation and shows of more of pest. So why not? 

A string contains an arbitrary number of characters, therefore the `inner` rule denotes exactly that with `char*`, where `*` indicates zero or more. The `@` marks this rule 
as [atomic](https://pest.rs/book/grammars/syntax.html#atomic). Atomic rules have the following special properties:
1. `~` means immediately follow by, so no whitespace. 
2. Repetition operators (* and +) have no implicit separation.
3. Rules inside an atomic rule are treated as atomic.

This is practical for parsing strings as we do not want to produce a token for each individual character. This also removes the implicit
allowance of whitespace that normal rules have with `~`. If you are wondering why have the `inner` rule at all and just put `char*` directly
inside the string rule, that is a fair question. The `inner` rule contains the contents of a string without the surrounding`"`, if we removed it we would have to manually
remove the surrounding quotes.

We now finally get to the `string` rule. A string is delimited by `"`, therefore our rule does exactly that 
`string = ${ "\""	~ inner ~ "\"" }`. We look for a pair of `"` with an `inner` between then. Note the `$`, it denotes `string` as
a [compound atomic](https://pest.rs/book/grammars/syntax.html#atomic). This ensures that no implicit whitespace is permitted while
still collecting the inner rules for parsing. The example below will show this in action :)

Lets write some Rust and test our grammar.

Init a cargo project, if you have not already and add the [`pest`](https://pest.rs) dependecies:

```sh
cargo init --bin somelang 
cargo add pest pest_derive 
```

I believe that `cargo add` is a builtin option these days. If you are on an older version of cargo it can be installed with [`cargo-edit`](https://github.com/killercup/cargo-edit). 

Otherwise just add the dependencies manually to `Cargo.toml` as shown below :)

Your `Cargo.toml` should look like this:
```toml
[dependencies]
pest = "2.6"
pest_derive = "2.6"
```

We can now jump into `src/main.rs` and use our parser. Lets start by printing what a parsed string looks like.

Remember to put the grammar defined earlier into `grammar.pest` in the same directory as `Cargo.toml`. Your directory should have the following structure:

```
Cargo.toml
Cargo.lock
grammar.pest
src/main.rs
...
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

Here we can see the [Pair](https://docs.rs/pest/latest/pest/iterators/struct.Pair.html) which 
represents a matching pair of tokens and everything in between. In our case `"` is the token
matched by the first Pair: `span: Span { str: "\"hello world\"", start: 0, end: 13 }`. 
The inner field contains what ever the tokens spanned. In this case another `Pair` spanning the actual 
 string contents. The matching token pair is `h` and `d`. This might seem a bit counter intuitve
but our string contains any number of `CHAR` due to `inner = { CHAR* }` and char matches
 on almost any character. An important observation here is that matching tokens for a `Pair` do not need to be the same, 
only match the same rule. 

Note how the `inner` does not contain any `CHAR` rules but just the characters it spans. 
This is due the fact that `inner` is [atomic](https://pest.rs/book/grammars/syntax.html#atomic) and does not permit 
whitespace between rules. `string` on the other hand is a [compound atomic](https://pest.rs/book/grammars/syntax.html#atomic) rule. 
It will therefore have inner rules but still does not permit any whitespace between matches.

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
so `val` only contains the `string` rule. `val` is silent denoted by `_`. This means that it will note be visible in our
parser. Instead we will see instances it's inner rules. This is nice as it saves us having to "unwrap" val instances, since we
are always interested in it's contents.

`id = { ASCII_ALPHA+  }` will represent an identifier of variables, functions, modules etc. For example `println` is an identifer. 
`ASCII_ALPHA+` indicates matches on one or more
letters from a-z and A-Z. In future iterations we want to be support more characters but this will do for now. 

Finally `call = ${ id ~  "(" ~ val  ~ ")" }`, which matches on a function call containing a single argument. 
`call` concists of an identifer followed by a single argument surrounded by parentheses. 
Exactly what we need! We make this rule compound atomic as we do not want whitespace between the 
identifer and parentheses.


`grammar.pest` should now look like this.

```
// grammar.pest

char = {
    !("\"" | "\\" ) ~ ANY
}

inner = @{ char* }

string = ${ "\""	~ inner ~ "\"" }

val = _{ string }

id = { ASCII_ALPHA*  }

call = ${ id ~  "(" ~ val  ~ ")" }

```

Lets take it for a spin!. Running `cargo run` should still produce an error, as we have not updated which rule we use to parse this.
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

And with that we are finished with the gramma, for now ;)

This post has grown quite long so we will continue in the next one. 
Where we will actually use our parser and hopefully finish `prinln("Hello World!")`.

As a final step we will extract the identifier and argument, by traversing the parsed input as nested iterators.
This will help show how to work with pest.

```rust
// src/main.rs
...

    // Get an iterator over the pairs matched by Rule::call
    let mut call_pairs = parsed_input.into_iter().next().unwrap().into_inner();

    // We know the id comes first.
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

Note how `hello world` does not have surrounding quotes. This is because we went one level deeper. If we stoppe at  `argument = 
call_pairs.next().unwrap` we woud have the `string` rule which
contains `"`. By callingo `into_inner` we go one rule deeper and extract the matched `inner` which only holds the contents of a 
string. 

Traversing our parsed input in this manner will allow us to build a propper AST one we are ready. I hope this example showed how 
powerful and simple [pest](todo) allows us to get started parsing.

### Closing notes

Thanks for reading! I hope this post was informative and fun. The code for this post is available [here](TODO).

If you are interested in this sort of thing subscribe to the [rss](TODO). I do not have a propper about section
yet, so I will put my socials here:

mastodon: https://hachyderm.io/@jmintb

github: https://github.com/jmintb

youtube: https://www.youtube.com/channel/UCiktIroKtzNNLqyRgPxvnfQ

twitch: https://www.twitch.tv/teainspace

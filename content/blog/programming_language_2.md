+++
title = "Building a programming language part 2: Hello World!"
date = 2023-07-17
+++

## Compiler

We are now able to parse `print("Hello World!")` but can not compile anything yet.  We will start by implementing a builtin print function.
You can think of this as a sort of standard library provided print function. First we need to get some preliminaries in order.

### Preliminaries

Before we dive in we need to establish a bit of intuition and setup our projet to work with [MLIR](https://mlir.llvm.org/) and [LLVM](https://llvm.org/). 
What we want to accomplish here is to produce executable machine code from `print("Hello World!")`. This involes many steps of lowering the representation
 to something the hardware can execute. That as where compiler backends can do some of the heavy lifting. Instead of targeting marchine
code our compiler can act as a frontend and target the intermediate representation(IR) of a backend such as [LLVM](todo) and the backend can then handle the heavy lifting 
of optimizing and targeting specific hardware. This has many upsides and downsides but for bootstrapping a new language in a single blog post it is very helpful :). So what
about MLIR? Well MLIR is developed by the LLVM project and attempts to address some of the limitations of LLVM, such as better support for parallell compilation and better
support for heterogenus compute. Heterogenuis compute is a fancy way of saying multiple different types of hardware, like if you want to build a language that targets both
CPUs and GPU, which I do. MLIR stands for Multi-Level intermediate representation and if mean to replace the need for language to have their own IR. I will dive into
the technical and theoretical side of this in a separate post as there is a lot to cover. Hopefully this gave you an idea of why we are using this approach. If not let me know
and I will try to elaborate. Otherwise don't worry even if this doesn't make complete sense following along with the implementation should be just fine anyways and might be a better
way to learn what I just tried to explain haha.

### The print function

Now the question how do we want to implement print? And how does printing even work?. Printing works by writing to standard out. Standard out differs 
depending on the environment our code is executed in but we can think of it as a pointer to a file we can keep writing to. 
 One option would be to use `printf` from the os provided c libraries. 
That would work just fine but I was curious how exactly to get a hold of standard out directly, so we will be implement this our selves. It is relatively simple once you understand what is happening so it will not expand the implementation by much. Instead we will use the `write` function, which allows us to write bytes directly to file or something file like using a [file descriptor](https://en.wikipedia.org/wiki/File_descriptor). Under the hood I think most printf implementations will endup using write calls. We loose a lot of functionality though. Printf allows us to format strings and handles buffering. Where as `write` will do no such thing, but it will allow us to define our own String types which are not necessarily null delimited. Also the point is to learn how to do these things so working with the underlying primitives is kind of the point :).

Now this will invovle a few steps:

1. We need to expose `write` to our language.
2. We need to represent strings in our compiler.
3. We need to be able to pass a pointer to pointer to the bytes for our string and an integer representing the stdout file descriptor.


#### Understanding `write`

First lets try to understand what `write` is doing. `write` is a [syscall](https://en.wikipedia.org/wiki/System_call) which essentially means a function which is executed by the OS kernel. At least in the contex to Linux. The reason being that userspace code is not allowed to write directly to things like stdout.(TODO why is this?).

The signature is shown below. The first parameter fd is an integer  representing a specifc file(like) object which can be written to. The file descriptor for stdout 1 by default, at least in most cases. The buf parameter points to the bytes we want to write. The size parameter indicates the size of the data we want to write. This will allow `write` to start from the the buf pointer and write the number of bytes according to the supplied size. The offset parameter is not relevant in our case but if we were writing to an actual file we could specify that we want `write` to start from a specific offset within the file.     

```c
ssize_t write(int fd, const void *buf,  size_t size, off_t offset);
 ```
Both `fdopen` and `fwrite` are provided by the operating system's `stdio.h` C headers. `fdopen` returns a pointer to a file based on the file descriptor we provide. File descriptors are integeres that act as handles to files or or other I/O resources. The file descriptor for stdout is 1 on linux. We can therefore call `fdopen(1)`

#### Exposing `write`

Now how do we support calling `write` from our language? Calling into other languages is generally referred to as `ffi`. It is quite common for languages to support `ffi` with C. For example see an [example](https://doc.rust-lang.org/rust-by-example/std_misc/ffi.html) with Rust. We want to make this as easy as possible to implement at the moment. Therefore we will hardcode the foreign interface to `write` for now. Thankfully this is easy to with `MLIR`. *Note that MLIR/LLVM does not always call C correctly but for our purposes this will suffice*(TODO: source). The `MLIR` we need to produce for exposing `write` look like this:

```mlir

```

This is simple to do from `Rust` once you know how:

```rust
```

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
That would work just fine but I was curious how exactly to get a hold of standard out directly, so we will be implement this our selves. It is relatively simple once you understand what is happening so it will not expand the implementation by much. Our print function will work in the following steps:
1. Retrieve a pointer to stdout using `fdopen`
2. Write bytes to stdout using `fwrite` 

```c
FILE *fdopen(int fd, const char *mode);

size_t fwrite(const void *ptr, size_t size, size_t nmemb, FILE *stream);
 ```
Both `fdopen` and `fwrite` are provided by the operating system's `stdio.h` C headers. `fdopen` returns a pointer to a file based on the file descriptor we provide. File descriptors are integeres that act as handles to files or or other I/O resources. The file descriptor for stdout is 1 on linux. We can therefore call `fdopen(1)`

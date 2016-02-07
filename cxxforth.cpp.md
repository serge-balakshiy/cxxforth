
cxxforth: A Simple Forth Implementation in C++
==============================================

by Kristopher Johnson

----

This is free and unencumbered software released into the public domain.

Anyone is free to copy, modify, publish, use, compile, sell, or distribute this
software, either in source code form or as a compiled binary, for any purpose,
commercial or non-commercial, and by any means.

In jurisdictions that recognize copyright laws, the author or authors of this
software dedicate any and all copyright interest in the software to the public
domain. We make this dedication for the benefit of the public at large and to
the detriment of our heirs and successors. We intend this dedication to be an
overt act of relinquishment in perpetuity of all present and future rights to
this software under copyright law.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL THE
AUTHORS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN
ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

For more information, please refer to <http://unlicense.org/>

----

`cxxforth` is a simple implementation of a [Forth][forth] system in C++.  There
are many examples of Forth implementations available on the Internet, but most
of them are written in assembly language or low-level C, with a focus in
maximizing efficiency and demonstrating traditional Forth implementation
techniques.  This Forth is different: Our goal is to use modern C++ to create a
Forth implementation that is easy to understand, easy to port, and easy to
extend.  We're not going to talk about register assignments or addressing modes
or opcodes or the trade-offs between indirect threaded code, direct threaded
code, subroutine threaded code, and token threaded code.  We are just going to
build a working Forth system in a couple thousand lines of code.

An inspiration for this implementation is Richard W.M. Jones's
[JONESFORTH][jonesforth].  JONESFORTH is a low-level assembly-language Forth
implementation written as a very readable tutorial, and we are going to copy
its style for our higher-level implementation.  Our Forth kernel is written as
C++ file with large comment blocks, and we have a utility, `cpp2md`, that will
take that C++ file and convert it to a [Markdown][markdown] format document
with nicely formatted commentary sections between the C++ code blocks.  If you
are currently reading `cxxforth.cpp` online, consider reading the Markdown file
[cxxforth.cpp.md](cxxforth.cpp.md) instead, as BitBucket or GitHub will
automtically display the Markdown file as formatted HTML.

As in other Forth systems, the basic design of this Forth is to create a small
kernel in native code (C++, in our case), and then implement the rest of the
system with Forth code.  The kernel has to provide the basic primitives needed
for memory access, arithmetic, logical operations, and operating system access.
With those primitives, we can then write Forth code to flesh out the system.

We are writing C++ conforming to the C++14 standard.  If your C++ compiler
does not support C++14 yet, you may need to make some modifications to the code
to get it to build.

The Forth words provided by cxxforth are based on those in the [ANS Forth draft
standard][dpans].  We don't claim conformance to the standard, but the reader
can use the draft standard as a crude form of documentation for the Forth words
we implement.  cxxforth implements many of the words from the standard's Core
and Core Extension word sets, and a smattering of words from other standard
word sets.

It is assumed that the reader has some familiarity in C++ and Forth.  It is
recommended that the reader first read the JONESFORTH source or the source of
some other Forth implementation to get the basic gist of how Forth is usually
implemented.

[forth]: https://en.wikipedia.org/wiki/Forth_(programming_language) "Forth (programming language)"

[jonesforth]: http://git.annexia.org/?p=jonesforth.git;a=blob;f=jonesforth.S;h=45e6e854a5d2a4c3f26af264dfce56379d401425;hb=HEAD

[markdown]: https://daringfireball.net/projects/markdown/ "Markdown"

[dpans]: http://forth.sourceforge.net/std/dpans/dpansf.htm "Alphabetic list of words"

----

Building cxxforth
-----------------

Building the `cxxforth` executable and other targets is easiest if you are on a
UNIX-ish system that has `make`, `cmake`, and `clang` or `gcc`.  If you have
those components, you can probably build by just entering these commands:

    cd myfiles/cxxforth
    make

If successful, the `cxxforth` executable will be built in the `cxxforth/build/`
subdirectory.

If you don't have one of those components, or if 'make' doesn't work, then it's
not too hard to build it manually.  You will need to create a file called
`cxxforthconfig.h`, which can be empty, then you need to invoke your C++
compiler on the `cxxforth.cpp` source file, enabling whatever options might be
needed for C++14 compatibility.  For example, on a Linux system with gcc, you
should be able to build it by entering these commands:

    cd myfiles/cxxforth
    touch cxxforthconfig.h
    g++ -std=c++14 -o cxxforth cxxforth.cpp

----

The Code
--------

In `cxxforth.cpp`, we start by including headers from the C++ Standard Library.
We also include `cxxforth.h`, which declares exported functions and includes
the `cxxforthconfig.h` file produced by the CMake build.

    
    #include "cxxforth.h"
    
    #include <algorithm>
    #include <cctype>
    #include <chrono>
    #include <cstdlib>
    #include <cstring>
    #include <ctime>
    #include <iostream>
    #include <stdexcept>
    #include <string>
    #include <thread>
    

We have a few macros to define the size of the Forth data space, the maximum
numbers of cells on the data and return stacks, and the maximum number of word
definitions in the dictionary.

These macros are usually defined in the `cxxforthconfig.h` header that is
generated by CMake and included by `cxxforth.h`, but we provide default values
in case they have not been defined.

    
    #ifndef CXXFORTH_DATASPACE_SIZE
    #define CXXFORTH_DATASPACE_SIZE (16 * 1024 * sizeof(Cell))
    #endif
    
    #ifndef CXXFORTH_DSTACK_COUNT
    #define CXXFORTH_DSTACK_COUNT (256)
    #endif
    
    #ifndef CXXFORTH_RSTACK_COUNT
    #define CXXFORTH_RSTACK_COUNT (256)
    #endif
    
    #ifndef CXXFORTH_DEFINITIONS_COUNT
    #define CXXFORTH_DEFINITIONS_COUNT (4096)
    #endif
    
  
We'll start by defining some basic types.  A `Cell` is the basic Forth type.
We define our cell using the C++ `uintptr_t` type to ensure it is large enough
to hold an address.  This generally means that it will be a 32-bit value on
32-bit platforms and a 64-bit value on 64-bit platforms.  (If you want to build
a 32-bit Forth on a 64-bit platform with clang or gcc, you can pass the `-m32`
flag to the compiler and linker.)

Forth doesn't require type declarations; a cell can be used as an address, an
unsigned integer, a signed integer, or a variety of other uses.  However, in
C++ we will have to be explicit about types to perform the operations we want
to perform.  So, we define a few types, and a few macros to avoid littering our
code with `reinterpret_cast<>()`.

By the way, we won't be providing any of the double-cell operations that
traditional Forths provide.  Double-cell operations were important in the days
of 8-bit and 16-bit Forths, but with cells of 32 bits or more, many
applications have no need for them.

We also aren't dealing with floating-point values.  Floating-point support
would be useful, but we're trying to keep this simple.

    
    namespace {
    
    using Cell = uintptr_t;
    using SCell = intptr_t;
    
    using Char = unsigned char;
    using SChar = signed char;
    
    using CAddr = Char*;  // Any address
    using AAddr = Cell*;  // Cell-aligned address
    
    #define CELL(x)  reinterpret_cast<Cell>(x)
    #define CADDR(x) reinterpret_cast<Char*>(x)
    #define AADDR(x) reinterpret_cast<AAddr>(x)
    
    constexpr auto CellSize = sizeof(Cell);
    

We define constants for Forth `TRUE` and `FALSE` flag values. Note that the
Forth standard says that a true flag is a cell with all bits set, unlike C++,
so we need to be sure to use these constants for all Forth words that return a
boolean flag.

    
    constexpr auto True = static_cast<Cell>(-1);
    constexpr auto False = static_cast<Cell>(0);
    

Our first big difference from most traditional Forth implementations is how
we'll store the word definitions for our Forth dictionary.  Traditional Forths
intersperse the word names in the shared data space along with code and data,
using a linked list to navigate through them.  We are going to just have an
array of `Definition` structs.

One of the members of `Definition` is a `std::string` to hold the name, so we
won't need to worry about managing the memory for that variable-length field.
We can use its `data()` function when we need a pointer to the raw character
data.

A `Definition` also needs a `code` field that points to the native code
associated with the word, a `parameter` field that points to the data-space
elements associated with the word, and some bit flags to keep track of whether
the word is `IMMEDIATE` and/or `HIDDEN`.  We will explore the use of these
fields later when we talk about the interpreter.

Note that the `code` field is the first member of the `Definition` struct.
This means that the address of a `Definition` is also the address of a code
field.  We will use such an address as our Forth _execution token_ (xt).

    
    using Code = void(*)();
    
    struct Definition {
        Code        code      = nullptr;
        AAddr       parameter = nullptr;
        Cell        flags     = 0;   
        std::string name;
    
        static constexpr Cell FlagHidden    = (1 << 1);
        static constexpr Cell FlagImmediate = (1 << 2);
    
        const char* nameAddress() const {
            return name.data();
        }
    
        std::size_t nameLength() const {
            return name.length();
        }
    
        bool isHidden() const {
            return (flags & FlagHidden) != 0;
        }
    
        void setHidden(bool hidden) {
            flags = hidden ? (flags | FlagHidden) : (flags & ~FlagHidden);
        }
    
        bool isImmediate() const {
            return (flags & FlagImmediate) != 0;
        }
    
        void setImmediate(bool immediate) {
            flags = immediate ? (flags | FlagImmediate) : (flags & ~FlagImmediate);
        }
    
        void reset() {
            code = nullptr;
            parameter = nullptr;
            flags = 0;
            name.clear();
        }
    };
    

With our types defined, we can define our global variables.  We have the Forth
data space, data and return stacks, and the array of `Definitions`.

For each of these arrays, we also define constants that point to the end of the
array, so we can easily test whether we have filled them and need to report an
overflow.

    
    Char       dataSpace[CXXFORTH_DATASPACE_SIZE];
    Cell       dStack[CXXFORTH_DSTACK_COUNT];
    Cell       rStack[CXXFORTH_RSTACK_COUNT];
    Definition dictionary[CXXFORTH_DEFINITIONS_COUNT];
    
    constexpr CAddr       dataSpaceLimit  = &dataSpace[CXXFORTH_DATASPACE_SIZE];
    constexpr AAddr       dStackLimit     = &dStack[CXXFORTH_DSTACK_COUNT];
    constexpr AAddr       rStackLimit     = &rStack[CXXFORTH_RSTACK_COUNT];
    constexpr Definition* dictionaryLimit = &dictionary[CXXFORTH_DEFINITIONS_COUNT];
    

For each of the global arrays, we need a pointer to the current location.

For the data space, we have the `dataPointer`, which corresponds to Forth's
`HERE`.

For each of the stacks, we need a pointer to the element at the top of the
stack.  The stacks grow upward.  When a stack is empty, the associated pointer
points to an address below the actual bottom of the array, so we will need to
avoid dereferencing these pointers under those circumstances.

Finally, we need a pointer to the last member of the `Definition` array.  As
with the stack pointers, this pointer is not valid when the dictionary is
empty, but we don't need to worry much about it because the array won't be
empty after it is initialized with the kernel's built-in words.

    
    CAddr       dataPointer      = nullptr;
    AAddr       dTop             = nullptr;
    AAddr       rTop             = nullptr;
    Definition* latestDefinition = nullptr;
    

Our interpreter needs to keep track of the word that is currently being
executed, and the next instruction to be executed when it is finished.

We also need a flag to track whether we are in interpreting or compiling state
(corresponding to Forth's `STATE` variable).

    
    Definition* currentInstruction = nullptr;
    Definition* nextInstruction    = nullptr;
    
    Cell isCompiling = True;
    
    std::string inputBuffer;
    Cell inputOffset = 0;
    
    size_t commandLineArgCount = 0;
    const char** commandLineArgVector = nullptr;
    
    

Runtime Safety Checks
---------------------

Old-school Forths are apparently implemented by super-programmers who never
make coding mistakes and so don't want the overhead of bounds-checking or other
nanny hand-holding.  However, we're just dumb C++ programmers here, and we'd
like some help to catch our mistakes.

To that end, we have a set of macros and functions that verify that we have the
expected number of arguments available on our stacks, that we aren't going to
run off the end of an array, that we aren't going to try to divide by zero, and
so on.

We can define the macro `CXXFORTH_SKIP_RUNTIME_CHECKS` to generate an
executable that doesn't include these checks, so when we have a fully debugged
Forth application we can run it on that optimized executable for improvied
performance.

When the `CXXFORTH_SKIP_RUNTIME_CHECKS` macro is not defined, these macros
will check conditions and throw a `std::runtime_error` if the assertions fail.
We won't go into the details of these macros here.  Later you will see them
used in the definitions of our primitive Forth words.

    
    std::ptrdiff_t dStackDepth() {
        return dTop - dStack + 1;
    }
    
    std::ptrdiff_t rStackDepth() {
        return rTop - rStack + 1;
    }
    
    #ifdef CXXFORTH_SKIP_RUNTIME_CHECKS
    
    #define RUNTIME_ERROR(msg)                   do { } while (0)
    #define RUNTIME_ERROR_IF(cond, msg)          do { } while (0)
    #define REQUIRE_DSTACK_DEPTH(n, name)        do { } while (0)
    #define REQUIRE_DSTACK_AVAILABLE(n, name)    do { } while (0)
    #define REQUIRE_RSTACK_DEPTH(n, name)        do { } while (0)
    #define REQUIRE_RSTACK_AVAILABLE(n, name)    do { } while (0)
    #define REQUIRE_ALIGNED(addr, name)          do { } while (0)
    #define REQUIRE_VALID_HERE(name)             do { } while (0)
    #define REQUIRE_DATASPACE_AVAILABLE(n, name) do { } while (0)
    
    #else
    
    #define RUNTIME_ERROR(msg)                   do { throw std::runtime_error(msg); } while (0)
    #define RUNTIME_ERROR_IF(cond, msg)          do { if (cond) RUNTIME_ERROR(msg); } while (0)
    #define REQUIRE_DSTACK_DEPTH(n, name)        requireDStackDepth(n, name)
    #define REQUIRE_DSTACK_AVAILABLE(n, name)    requireDStackAvailable(n, name)
    #define REQUIRE_RSTACK_DEPTH(n, name)        requireRStackDepth(n, name)
    #define REQUIRE_RSTACK_AVAILABLE(n, name)    requireRStackAvailable(n, name)
    #define REQUIRE_ALIGNED(addr, name)          checkAligned(addr, name)
    #define REQUIRE_VALID_HERE(name)             checkValidHere(name)
    #define REQUIRE_DATASPACE_AVAILABLE(n, name) requireDataSpaceAvailable(n, name)
    
    template<typename T>
    void checkAligned(T addr, const char* name) {
        RUNTIME_ERROR_IF((CELL(addr) % CellSize) != 0,
                         std::string(name) + ": unaligned address");
    }
    
    void requireDStackDepth(std::size_t n, const char* name) {
        RUNTIME_ERROR_IF(dStackDepth() < static_cast<std::ptrdiff_t>(n),
                         std::string(name) + ": stack underflow");
    }
    
    void requireDStackAvailable(std::size_t n, const char* name) {
        RUNTIME_ERROR_IF((dTop + n) >= dStackLimit,
                         std::string(name) + ": stack overflow");
    }
    
    void requireRStackDepth(std::size_t n, const char* name) {
        RUNTIME_ERROR_IF(rStackDepth() < std::ptrdiff_t(n),
                         std::string(name) + ": return stack underflow");
    }
    
    void requireRStackAvailable(std::size_t n, const char* name) {
        RUNTIME_ERROR_IF((rTop + n) >= rStackLimit,
                         std::string(name) + ": return stack overflow");
    }
    
    void checkValidHere(const char* name) {
        RUNTIME_ERROR_IF(dataPointer < dataSpace || dataSpaceLimit <= dataPointer,
                         std::string(name) + ": HERE outside data space");
    }
    
    void requireDataSpaceAvailable(std::size_t n, const char* name) {
        RUNTIME_ERROR_IF((dataPointer + n) >= dataSpaceLimit,
                         std::string(name) + ": data space overflow");
    }
    
    #endif // CXXFORTH_SKIP_RUNTIME_CHECKS
    

Stack Manipulation Primitives
-----------------------------

We will be spending a lot of time pushing and popping values to and from our
data and return stacks, so in lieu of sprinkling pointer arithmetic all through
our code, we'll define a few simple functions to handle those operations.  We
can expect the compiler to expand calls to these functions inline, so we aren't
losing any efficiency.

    
    // Push cell onto data stack.
    void push(Cell x) {
        *(++dTop) = x;
    }
    
    // Pop cell from data stack.
    void pop() {
        --dTop;
    }
    
    // Push cell onto return stack.
    void rpush(Cell x) {
        *(++rTop) = x;
    }
    
    // Pop cell from return stack.
    void rpop() {
        --rTop;
    }
    

Interpreter
-----------

Now we get to some of the most important operations in a Forth system.  They
look very simple, but these functions are going to implement the interpreter
that executes compiled Forth definitions.

There are basically two kinds of words in a Forth system:

- primitive code: native code fragments that are executed directly by the CPU
- colon definition: a sequence of Forth words compiled by `:` (colon), `:NONAME`, or `DOES>`.

Every defined word has a `code` field that points to native code.  In the case
of "code" words, the `code` field points to a routine that performs the
operation.  In the case of a colon definition, the `code` field points to the
`doColon()` function, which saves the current program state and then starts
executing the words that make up the colon definition.

Each colon definition ends with a call to `EXIT`, which sets up a return to the
colon definition that called the current word.

    
    void processInstructions() {
        while (nextInstruction != nullptr) {
            currentInstruction = nextInstruction++;
            currentInstruction->code();
        }
        throw std::runtime_error("No instructions to process");
    }
    
    void doColon() {
        REQUIRE_RSTACK_AVAILABLE(1, "(colon)");
        rpush(CELL(nextInstruction));
        nextInstruction = reinterpret_cast<Definition*>(currentInstruction->parameter);
    }
    
    // EXIT ( -- ) ( R: nest-sys -- )
    void exit() {
        REQUIRE_RSTACK_DEPTH(1, "EXIT");
        nextInstruction = reinterpret_cast<Definition*>(*rTop);
        rpop();
    }
    
    // (literal) ( -- x )
    // Compiled by LITERAL.
    // Not an ANS Forth word.
    void doLiteral() {
        REQUIRE_DSTACK_AVAILABLE(1, "(literal)");
        push(CELL(nextInstruction));
        ++nextInstruction;
    }
    

Next we'll define the basic Forth stack manipulation words.  When changing the
stack, we don't change the stack depth any more than necessary.  For example,
`SWAP` and `DROP` just rearrange elements on the stack, rather than doing any
popping or pushing.

    
    // DEPTH ( -- +n )
    void depth() {
        REQUIRE_DSTACK_AVAILABLE(1, "DEPTH");
        push(static_cast<Cell>(dStackDepth()));
    }
    
    // DROP ( x -- )
    void drop() {
        REQUIRE_DSTACK_DEPTH(1, "DROP");
        pop();
    }
    
    // DUP ( x -- x x )
    void dup() {
        REQUIRE_DSTACK_DEPTH(1, "DUP");
        REQUIRE_DSTACK_AVAILABLE(1, "DUP");
        push(*dTop);
    }
    
    // OVER ( x1 x2 -- x1 x2 x1 )
    void over() {
        REQUIRE_DSTACK_DEPTH(2, "OVER");
        REQUIRE_DSTACK_AVAILABLE(1, "OVER");
        push(*(dTop - 1));
    }
    
    // SWAP ( x1 x2 -- x2 x1 )
    void swap() {
        REQUIRE_DSTACK_DEPTH(2, "SWAP");
        auto temp = *dTop;
        *dTop = *(dTop - 1);
        *(dTop - 1) = temp;
    }
    
    // ROT ( x1 x2 x3 -- x2 x3 x1 )
    void rot() {
        REQUIRE_DSTACK_DEPTH(3, "ROT");
        auto x1 = *(dTop - 2);
        auto x2 = *(dTop - 1);
        auto x3 = *dTop;
        *dTop = x1;
        *(dTop - 1) = x3;
        *(dTop - 2) = x2;
    }
    
    // PICK ( xu ... x1 x0 u -- xu ... x1 x0 xu )
    void pick() {
        REQUIRE_DSTACK_DEPTH(1, "PICK");
        auto index = *dTop;
        REQUIRE_DSTACK_DEPTH(index + 2, "PICK");
        *dTop = *(dTop - index - 1);
    }
    
    // ROLL ( xu xu-1 ... x0 u -- xu-1 ... x0 xu )
    void roll() {
        REQUIRE_DSTACK_DEPTH(1, "ROLL");
        auto index = *dTop;
        pop();
        if (index > 0) {
            REQUIRE_DSTACK_DEPTH(index + 1, "ROLL");
            auto x = *(dTop - index - 1);
            std::memmove(dTop - index - 1, dTop - index, index * sizeof(Cell));
            *dTop = x;
        }
    }
    
    // ?DUP ( x -- 0 | x x )
    void qdup() {
        REQUIRE_DSTACK_DEPTH(1, "?DUP");
        Cell top = *dTop;
        if (top != 0) {
            REQUIRE_DSTACK_AVAILABLE(1, "?DUP");
            push(top);
        }
    }
    
    // >R ( x -- ) ( R:  -- x )
    void toR() {
        REQUIRE_DSTACK_DEPTH(1, ">R");
        REQUIRE_RSTACK_AVAILABLE(1, ">R");
        rpush(*dTop);
        pop();
    }
    
    // R> ( -- x ) ( R: x -- )
    void rFrom() {
        REQUIRE_RSTACK_DEPTH(1, "R>");
        REQUIRE_DSTACK_AVAILABLE(1, "R>");
        push(*rTop);
        rpop();
    }
    
    // R@ ( -- x ) ( R: x -- x )
    void rFetch() {
        REQUIRE_RSTACK_DEPTH(1, "R@");
        REQUIRE_DSTACK_AVAILABLE(1, "R@");
        push(*rTop);
    }
    

Another important set of Forth primitives are those for reading and writing
values in data space.

    
    // ! ( x a-addr -- )
    void store() {
        REQUIRE_DSTACK_DEPTH(2, "!");
        auto aaddr = AADDR(*dTop);
        REQUIRE_ALIGNED(aaddr, "!");
        pop();
        auto x = *dTop;
        pop();
        *aaddr = x;
    }
    
    // @ ( a-addr -- x )
    void fetch() {
        REQUIRE_DSTACK_DEPTH(1, "@");
        auto aaddr = AADDR(*dTop);
        REQUIRE_ALIGNED(aaddr, "@");
        *dTop = *aaddr;
    }
    
    // c! ( char c-addr -- )
    void cstore() {
        REQUIRE_DSTACK_DEPTH(2, "C!");
        auto caddr = CADDR(*dTop);
        pop();
        auto x = *dTop;
        pop();
        *caddr = static_cast<Char>(x);
    }
    
    // c@ ( c-addr -- char )
    void cfetch() {
        REQUIRE_DSTACK_DEPTH(1, "C@");
        auto caddr = CADDR(*dTop);
        REQUIRE_ALIGNED(caddr, "C@");
        *dTop = static_cast<Cell>(*caddr);
    }
    
 
Now we will define the Forth words for accessing and manipulating the data
space pointer.

    
    template<typename T>
    AAddr alignAddress(T addr) {
        auto offset = CELL(addr) % CellSize;
        return (offset == 0) ? AADDR(addr) : AADDR(CADDR(addr) + (CellSize - offset));
    }
    
    void alignDataPointer() {
        dataPointer = CADDR(alignAddress(dataPointer));
    }
    
    // ALIGN ( -- )
    void align() {
        alignDataPointer();
    }
    
    // ALIGNED ( addr -- a-addr )
    void aligned() {
        REQUIRE_DSTACK_DEPTH(1, "ALIGNED");
        *dTop = CELL(alignAddress(*dTop));
    }
    
    // HERE ( -- addr )
    void here() {
        REQUIRE_DSTACK_AVAILABLE(1, "HERE");
        push(CELL(dataPointer));
    }
    
    // ALLOT ( n -- )
    void allot() {
        REQUIRE_DSTACK_DEPTH(1, "ALLOT");
        REQUIRE_VALID_HERE("ALLOT");
        REQUIRE_DATASPACE_AVAILABLE(CellSize, "ALLOT");
        dataPointer += *dTop;
        pop();
    }
    
    // CELL+ ( a-addr1 -- a-addr2 )
    void cellPlus() {
        REQUIRE_DSTACK_DEPTH(1, "CELL+");
        *dTop += CellSize;
    }
    
    // CELLS ( n1 -- n2 )
    void cells() {
        REQUIRE_DSTACK_DEPTH(1, "CELLS");
        *dTop *= CellSize;
    }
    
    // Store a cell to data space.
    void data(Cell x) {
        REQUIRE_VALID_HERE(",");
        REQUIRE_DATASPACE_AVAILABLE(CellSize, ",");
        REQUIRE_ALIGNED(dataPointer, ",");
        *(AADDR(dataPointer)) = x;
        dataPointer += CellSize;
    }
    
    // Store a char to data space.
    void cdata(Cell x) {
        REQUIRE_VALID_HERE("C,");
        REQUIRE_DATASPACE_AVAILABLE(1, "C,");
        *dataPointer = static_cast<Char>(x);
        dataPointer += 1;
    }
    
    // UNUSED ( -- u )
    void unused() {
        REQUIRE_DSTACK_AVAILABLE(1, "UNUSED");
        push(static_cast<Cell>(dataSpaceLimit - dataPointer));
    }
    

Here we will define our I/O primitives.  We keep things easy and portable by
using C++ iostream objects.

    
    // EMIT ( x -- )
    void emit() {
        REQUIRE_DSTACK_DEPTH(1, "EMIT");
        auto cell = *dTop;
        pop();
        std::cout.put(static_cast<char>(cell));
    }
    
    // >IN ( -- a-addr )
    void in() {
        REQUIRE_DSTACK_AVAILABLE(1, ">IN");
        push(CELL(&inputOffset));
    }
    
    // SOURCE ( -- c-addr u )
    void source() {
        REQUIRE_DSTACK_AVAILABLE(2, "SOURCE");
        push(CELL(inputBuffer.data()));
        push(inputBuffer.length());
    }
    
    // REFILL ( -- flag )
    void refill() {
        REQUIRE_DSTACK_AVAILABLE(1, "REFILL");
        if (std::getline(std::cin, inputBuffer)) {
            inputOffset = 0;
            push(True);
        }
        else {
            push(False);
        }
    }
    

Define arithmetic primitives.

    
    // + ( n1 n2 -- n3 )
    void plus() {
        REQUIRE_DSTACK_DEPTH(2, "+");
        auto n2 = static_cast<SCell>(*dTop);
        pop();
        auto n1 = static_cast<SCell>(*dTop);
        *dTop = static_cast<Cell>(n1 + n2);
    }
    
    // - ( n1 n2 -- n3 )
    void minus() {
        REQUIRE_DSTACK_DEPTH(2, "-");
        auto n2 = static_cast<SCell>(*dTop);
        pop();
        auto n1 = static_cast<SCell>(*dTop);
        *dTop = static_cast<Cell>(n1 - n2);
    }
    
    // * ( n1 n2 -- n3 )
    void star() {
        REQUIRE_DSTACK_DEPTH(2, "*");
        auto n2 = static_cast<SCell>(*dTop);
        pop();
        auto n1 = static_cast<SCell>(*dTop);
        *dTop = static_cast<Cell>(n1 * n2);
    }
    
    // / ( n1 n2 -- n3 )
    void slash() {
        REQUIRE_DSTACK_DEPTH(2, "/");
        auto n2 = static_cast<SCell>(*dTop);
        pop();
        auto n1 = static_cast<SCell>(*dTop);
        RUNTIME_ERROR_IF(n2 == 0, "/: zero divisor");
        *dTop = static_cast<Cell>(n1 / n2);
    }
    
    // /MOD ( n1 n2 -- n3 n4 )
    void slashMod() {
        REQUIRE_DSTACK_DEPTH(2, "/MOD");
        auto n2 = static_cast<SCell>(*dTop);
        auto n1 = static_cast<SCell>(*(dTop - 1));
        RUNTIME_ERROR_IF(n2 == 0, "/MOD: zero divisor");
        auto result = std::ldiv(n1, n2);
        *(dTop - 1) = static_cast<Cell>(result.rem);
        *dTop = static_cast<Cell>(result.quot);
    }
    
    // NEGATE ( n1 -- n2 )
    void negate() {
        REQUIRE_DSTACK_DEPTH(1, "NEGATE");
        *dTop = static_cast<Cell>(-static_cast<SCell>(*dTop));
    }
    

Define logical and relational primitives.

    
    // AND ( x1 x2 -- x3 )
    void bitwiseAnd() {
        REQUIRE_DSTACK_DEPTH(2, "AND");
        auto x2 = *dTop;
        pop();
        *dTop = *dTop & x2;
    }
    
    // OR ( x1 x2 -- x3 )
    void bitwiseOr() {
        REQUIRE_DSTACK_DEPTH(2, "OR");
        auto x2 = *dTop;
        pop();
        *dTop = *dTop | x2;
    }
    
    // XOR ( x1 x2 -- x3 )
    void bitwiseXor() {
        REQUIRE_DSTACK_DEPTH(2, "XOR");
        auto x2 = *dTop;
        pop();
        *dTop = *dTop ^ x2;
    }
    
    // INVERT ( x1 -- x2 )
    void invert() {
        REQUIRE_DSTACK_DEPTH(1, "INVERT");
        *dTop = ~*dTop;
    }
    
    // LSHIFT ( x1 u -- x2 )
    void lshift() {
        REQUIRE_DSTACK_DEPTH(2, "LSHIFT");
        auto n = *dTop;
        pop();
        *dTop <<= n;
    }
    
    // RSHIFT ( x1 u -- x2 )
    void rshift() {
        REQUIRE_DSTACK_DEPTH(2, "RSHIFT");
        auto n = *dTop;
        pop();
        *dTop >>= n;
    }
    
    // = ( x1 x2 -- flag )
    void equals() {
        REQUIRE_DSTACK_DEPTH(2, "=");
        auto n2 = *dTop;
        pop();
        *dTop = *dTop == n2 ? True : False;
    }
    
    // < ( n1 n2 -- flag )
    void lessThan() {
        REQUIRE_DSTACK_DEPTH(2, "<");
        auto n2 = static_cast<SCell>(*dTop);
        pop();
        *dTop = static_cast<SCell>(*dTop) < n2 ? True : False;
    }
    
    // > ( n1 n2 -- flag )
    void greaterThan() {
        REQUIRE_DSTACK_DEPTH(2, ">");
        auto n2 = static_cast<SCell>(*dTop);
        pop();
        *dTop = static_cast<SCell>(*dTop) > n2 ? True : False;
    }
    

Define system and environmental primitives

    
    // #ARG ( -- n )
    // Provide count of command-line arguments.
    // Not an ANS Forth word.
    void argCount() {
        REQUIRE_DSTACK_AVAILABLE(1, "#ARG");
        push(commandLineArgCount);
    }
    
    // ARG ( n -- c-addr u )
    // Provide the Nth command-line argument.
    // Not an ANS Forth word.`
    void argAtIndex() {
        REQUIRE_DSTACK_DEPTH(1, "ARG");
        REQUIRE_DSTACK_AVAILABLE(1, "ARG");
        auto index = static_cast<size_t>(*dTop);
        RUNTIME_ERROR_IF(index >= commandLineArgCount, "ARG: invalid index");
        auto value = commandLineArgVector[index];
        *dTop = CELL(value);
        push(std::strlen(value));
    }
    
    // BYE ( -- )
    void bye() {
        std::exit(EXIT_SUCCESS);
    }
    
    // MS ( u -- )
    void ms() {
        REQUIRE_DSTACK_DEPTH(1, "MS");
        auto period = *dTop;
        pop();
        std::this_thread::sleep_for(std::chrono::milliseconds(period));
    }
    
    // TIME&DATE ( -- +n1 +n2 +n3 +n4 +n5 +n6 )
    void timeAndDate () {
        REQUIRE_DSTACK_AVAILABLE(6, "TIME&DATE");
        auto t = std::time(0);
        auto tm = std::localtime(&t);
        push(static_cast<Cell>(tm->tm_sec));
        push(static_cast<Cell>(tm->tm_min));
        push(static_cast<Cell>(tm->tm_hour));
        push(static_cast<Cell>(tm->tm_mday));
        push(static_cast<Cell>(tm->tm_mon + 1));
        push(static_cast<Cell>(tm->tm_year + 1900));
    }
    
    // UTCTIME&DATE ( -- +n1 +n2 +n3 +n4 +n5 +n6 )
    // Not an ANS Forth word.
    // Like TIME&DATE, but returns UTC rather than local time.
    void utcTimeAndDate () {
        REQUIRE_DSTACK_AVAILABLE(6, "UTCTIME&DATE");
        auto t = std::time(0);
        auto tm = std::gmtime(&t);
        push(static_cast<Cell>(tm->tm_sec));
        push(static_cast<Cell>(tm->tm_min));
        push(static_cast<Cell>(tm->tm_hour));
        push(static_cast<Cell>(tm->tm_mday));
        push(static_cast<Cell>(tm->tm_mon + 1));
        push(static_cast<Cell>(tm->tm_year + 1900));
    }
    
 
Compilation
-----------

    
    // STATE ( -- a-addr )
    void state() {
        REQUIRE_DSTACK_AVAILABLE(1, "STATE");
        push(CELL(&isCompiling));
    }
    
    void defineCodeWord(const char* name, Code code) {
        alignDataPointer();
    
        auto word = ++latestDefinition;
        word->code = code;
        word->parameter = AADDR(dataPointer);
        word->name = name;
    }
    
    bool doNamesMatch(CAddr name1, CAddr name2, Cell nameLength) {
        for (size_t i = 0; i < nameLength; ++i) {
            if (std::toupper(name1[i]) != std::toupper(name2[i])) {
                return false;
            }
        }
        return true;
    }
    
    Definition* findDefinition(CAddr nameToFind, Cell nameLength) {
        for (auto defn = latestDefinition; defn >= dictionary; ++defn) {
            if (defn->isHidden())
                continue;
            auto& name = defn->name;
            if (name.length() == nameLength) {
                auto nameCAddr = CADDR(const_cast<char*>(name.data()));
                if (doNamesMatch(nameToFind, nameCAddr, nameLength)) {
                    return defn;
                }
            }
        }
        return nullptr;
    }
    
    // FIND ( c-addr -- c-addr 0  |  xt 1  |  xt -1 )
    void find() {
        REQUIRE_DSTACK_DEPTH(1, "FIND");
        REQUIRE_DSTACK_AVAILABLE(1, "FIND");
        auto caddr = CADDR(*dTop);
        auto length = static_cast<Cell>(*caddr);
        auto name = caddr + 1;
        auto word = findDefinition(name, length);
        if (word == nullptr) {
            push(0);
        }
        else {
            *dTop = CELL(word);
            push(word->isImmediate() ? 1 : Cell(-1));
        }
    }
    
    // WORDS ( -- )
    void words() {
        for (auto word = latestDefinition; word >= dictionary; ++word) {
            std::cout << word->name << " ";
        }
    }
    

In `initializeDictionary()`, we set up the initial contents of the dictionary.
This is the Forth kernel that Forth code can use to implement the rest of a
working system.

    
    void definePrimitives() {
        static struct { const char* name; Code code; } codeWords[] = {
            {"!",             store},
            {"#ARG",          argCount},
            {"(literal)",     doLiteral},
            {"*",             star},
            {"+",             plus},
            {"-",             minus},
            {"/",             slash},
            {"/MOD",          slashMod},
            {"<",             lessThan},
            {"=",             equals},
            {">",             greaterThan},
            {">IN",           in},
            {">R",            toR},
            {"?DUP",          qdup},
            {"@",             fetch},
            {"ALIGN",         align},
            {"ALIGNED",       aligned},
            {"ALLOT",         allot},
            {"AND",           bitwiseAnd},
            {"ARG",           argAtIndex},
            {"BYE",           bye},
            {"C!",            cstore},
            {"C@",            cfetch},
            {"CELL+",         cellPlus},
            {"CELLS",         cells},
            {"DEPTH",         depth},
            {"DROP",          drop},
            {"DUP",           dup},
            {"EMIT",          emit},
            {"EXIT",          exit},
            {"FIND",          find},
            {"HERE",          here},
            {"INVERT",        invert},
            {"LSHIFT",        lshift},
            {"MS",            ms},
            {"NEGATE",        negate},
            {"OR",            bitwiseOr},
            {"OVER",          over},
            {"PICK",          pick},
            {"R>",            rFrom},
            {"R@",            rFetch},
            {"REFILL",        refill},
            {"ROLL",          roll},
            {"ROT",           rot},
            {"RSHIFT",        rshift},
            {"SOURCE",        source},
            {"STATE",         state},
            {"SWAP",          swap},
            {"TIME&DATE",     timeAndDate},
            {"UNUSED",        unused},
            {"UTCTIME&DATE",  utcTimeAndDate},
            {"WORDS",         words},
            {"XOR",           bitwiseXor}
        };
        for (auto& w: codeWords) {
            defineCodeWord(w.name, w.code);
        }
    }
    
    void initializeDictionary() {
        latestDefinition = dictionary - 1;
        for (auto word = dictionary; word < dictionaryLimit; ++word) {
            word->reset();
        }
    
        definePrimitives();
    }
    
    } // end anonymous namespace
    
    extern "C" void resetForth() {
    
        std::memset(dStack, 0, sizeof(dStack));
        dTop = dStack - 1;
    
        std::memset(rStack, 0, sizeof(rStack));
        rTop = rStack - 1;
    
        std::memset(dataSpace, 0, sizeof(dataSpace));
        dataPointer = dataSpace;
    
        initializeDictionary();
    }
    
    extern "C" int runForth(int argc, const char** argv) {
        try {
            commandLineArgCount = static_cast<size_t>(argc);
            commandLineArgVector = argv;
    
            resetForth();
    
            processInstructions();
    
            return 0;
        }
        catch (const std::exception& ex) {
            std::cerr << "Exception: " << ex.what() << std::endl;
            return -1;
        }
    }
    
    #ifndef CXXFORTH_NO_MAIN
    int main(int argc, const char** argv) {
        return runForth(argc, argv);
    }
    #endif
    
    
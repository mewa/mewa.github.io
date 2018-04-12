---
categories:
- Hello world
- x86 assembly
comments: true
date: "2017-05-14T00:00:00Z"
title: The somewhat lengthy "Hello world" post
---

It is a long honored tradition for IT texts to start with these famous words. Over the years, the languages have evolved, so did the paradigms by which we write our code -- yet almost every book or tutorial, on any language, still begins with this simple piece of *software*.

Since I don't want to stir things up and be some kind of a rebel this blog shall not be different -- in this particular case. 
<!--more-->

Now that we know what we're up to and what all this "Hello world" stuff is about, let's settle down to the tiny bits (or should I say *bytes*?) that get things going -- *the code*.

### The Code

I've been meaning to write a couple blog posts for a long time now. The main obstacle here being the fact that -- up until now -- I hadn't had a blog.

As I have just mentioned, any "Hello world" is a piece of *software*. Software that is beautiful in its simplicity.
Well, maybe for some of you, what I'm going to start with isn't the exact definition of simplicity... but the thing is -- it has one purpose, it was written in a single pass, it does its job -- and does it well. You're highly unlikely to **ever** need to patch a "Hello world" program once you have written one!
After all -- what can go wrong in a "Hello world" program? It's like the simplest of the simplest. It ought to be, at least.

Let's see for ourselves.

```nasm
global _start

section .text
_start:
	push 0xa
	push 0x21646c72
	push 0x6f77206f
	push 0x6c6c6548
	mov eax, esp
	push 13
	push eax
	push 1
	sub esp, 4
	mov eax, 4
	int 0x80
	
	push 0
	sub esp, 4
	mov eax, 1
	int 0x80
```

I have this magic `make` command that will create a binary out of this, so we can check if it really is a "Hello world".
```sh
mewa-osx$ make
nasm -fmacho32 hello.asm
ld -e _start hello.o
mewa-osx$ ./a.out
Hello world!
```
Unbelievable! Did you really think it was going to be something else? ;)

Now that we know it really is a "Hello world" let's analyse what is happening **exactly**.

### The bits

As you may have noticed, the example above is written in assembly language - in particular, x86 assembly, since I have an x86-64 Intel CPU. The assembly language is actually **dead simple** -- it consists of instructions which are just mnemonics for respective machine code, and that machine code does a thing and just that. The barrier that often makes people think it is hard is, that you have to be constantly aware of your *environment* - system, hardware, etc. And yeah, it is verbose - it is even more verbose than **Java**. :O

Anyway, I'm using NASM -- the Netwide Assembler -- to assemble an `.asm` file to an object blob (the `.o` file). 

The first line defines what entry point our binary is going to use.
```nasm
global _start
```
It basically makes it readable to other common tools such as `ld` -- linker. It's no coincidence that later on `ld -e _start hello.o` is invoked. Had we not included this line, this step would've failed showing an error message indicating symbol `_start` was not found. No wonder -- executables are just bits and bytes -- and they don't care what we call them. They just execute. That's why we have to mark it that this particular label has to be globally-readable and have its name saved, so others can find out about it. 

Usually, computer programs (which are going to be used interchangeably with PC programs throughout this blog) are divided into several `sections`. The most common sections you are likely to deal with are the `.data` section, and the `.text` section.
I shall elaborate on this distinction in the upcoming posts, but for the time being just remember, that the `.text` segment contains our program's code - i.e. flow of execution.

Hence, the next line marks the point that says *"Hey, all subsequent bytes are code!"*.
```nasm
section .text
```

Following up is a label `_start`, which we can refer to, just like we did in the first line of our program. 

Now the interesting part -- we're going to construct our greeting:
```nasm
push 0x0a
push 0x21646c72
push 0x6f77206f
push 0x6c6c6548
```
We're pushing individual words onto the stack, which will together form a character string. Let's see what do these words form.

So it's a hexadecimal number, where every 2 digits encode a byte.

```sh
mewa-osx$ echo $(cat hello.asm | grep "push 0x" | tr -d "push0x" | tr -d '\n\t ')
0a21646c726f7726f6c6c6548
```

Let's decode it to something human-readable -- it's a string after all. You can use any language -- or even a piece of paper and a pencil -- but I'm gonna use Haskell, because why not. Plus I like Haskell.

The following script takes our hex-string and decodes it to bytes that this string represents. 
```sh
mewa-osx$ ghci
GHCi, version 8.0.2: http://www.haskell.org/ghc/  :? for help
```
```haskell
Prelude> import Numeric as N
Prelude N> import Data.List.Split as S
Prelude N S> import Data.Char as C
Prelude N S C> map (chr . fst . head . readHex) $ reverse . chunksOf 2 $ "0a21646c726f77206f6c6c6548"
"Hello world!\n"
```

As you can see, the bytes we'd been pushing before really do make the greeting string. You don't have to completely understand what is going on in my little Haskell script, even though it's quite simple, however, you've probably noticed the `reverse` in the middle. This is due to x86 architecture being [little endian](https://en.wikipedia.org/wiki/Endianness), so when pushing our string we have to actually push it in reverse (and we did).

Now that we have constructed our string we have to get a way to refer to it somehow. Since we'd been pushing values onto the stack that's exactly where our string lies. Hence we need to retrieve the address pointing to the top of our stack. Luckily, this is pretty straightforward since there's the `esp` register which holds just that value. Let's copy it before we futher modify our stack. 
```nasm
mov eax, esp
```

Some of you, who already know x86 assembly language, may have noticed that the way I'm creating and using the greeting string isn't probably the same you would do when writing a generic computer program. However, I did it on purpose -- to unfold in the next posts.

Okay, we already have our greeting -- let's display it. We'll use the `write` syscall, which takes 3 arguments -- descriptor of the file to write to, address of the things we want to write and finfally length of the content in bytes.
```c
ssize_t write(int fildes, const void *buf, size_t nbyte);
```

We need to push these arguments on the stack *in reverse order*.
```nasm
push 13		; length of our string
push eax	; previously saved pointer to our string
push 1		; stdout
```

Last, but not least -- and this is specific to BSD derivatives -- we need to align the stack. 
```nasm
sub esp, 4
```
The origins of this empty gap are quite interesting, however, are a bit outside of the scope of this article. If you're keen on the nitty-gritty you can follow up [here](https://www.freebsd.org/doc/en/books/developers-handbook/x86-system-calls.html).

Now that we have our arguments on the stack, let's (finally) write our greeting. We need to tell the system what it should do with them. This is done by passing the syscall number in the `eax` register. On most *nix-like systems `write` syscall number is going to be 4.

```nasm
mov eax, 4
```

While the most common syscalls, such as the `write` syscall, usually have the same syscall numbers, you should always consult your system's headers or manuals to select the appropriate number, since even when different OSes expose the same functions (i.e. under the same "names"), they often have a different syscall number assigned and thus the code would break.

Anyway, we're ready to write stuff to the standard output, yay!

```nasm
int 0x80
```

Now, this will print out our message, but the (maybe) unexpected thing is it will still crash after doing so. So after saying "hi" and exchanging courtesy the other person would witness your seizures. Since we do not want anybody to see us having seizures and, as a matter of fact, we do not want to experience seizures at all, we'll need to fix it, before our software goes live.

Ind order to do so after printing our message we're going to exit gracefully using, well, the `exit` syscall -- which takes just one parameter and its the exit code.
```nasm
push 0			; exit code
sub esp, 4		; align stack
mov eax, 1		; exit syscall number
int 0x80 		; exit 0
```

That's it. Our software is complete -- we can now go out on the street and sell it to random people!

### Summary

While I tried to explain everything at hand some things may still not be clear if you've never done any assembly programming. If you still have trouble understanding some parts of it, I recommend to read through a basic assembly tutorial and then probably this post will be pretty straghtforward to understand.

It's true that the assembly is not the most common language, but it is pretty interesting, nevertheless. And powerful.

In the [upcoming posts]({{ "/articles/2017-07/polyglot-assembly-101" | absolute_url }}) we'll see however that sometimes it comes in very handy.

Last but not least, since this is my very first blog post -- please give me your feedback, whether you liked it -- or not (and why); comment, share, etc. etc.!
```sh
mewa-osx$ ./a.out
Hello world!
```

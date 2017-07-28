---
layout: post
title: Polyglot assembly 101
categories: [x86 assembly, polyglot]
comments: true
---

As promised in my [somewhat lengthy "Hello World" post]({{ "articles/2017-05/hello-world" | absolute_url }}), although much later than intended, I finally got down to writing a follow-up post. 

I remember one of the very first lectures during my uni course -- when `assembly` language was being introduced. Nothing too deep, really. However, I recall a statement being made, that probably still lives in the minds of the many that heard it (and perhaps even more that didn't). Namely -- that `assembly` code is not *portable*.

In this post we're going to take a look at how misleading that statement is and explore writing **polyglot assembly code**.
<!--more-->

### On portability
First of all let's take a look at the words *portability* and *polyglot* themselves. According to [Wikipedia](https://en.wikipedia.org/wiki/Software_portability), portability

> is the usability of the same software in different environments

whereas [polyglot code](https://en.wikipedia.org/wiki/Polyglot_(computing))

> is a computer program written in a valid form of multiple programming languages, which performs the same operations or output independent of the programming language used to compile or interpret it

Knowing that, let us all agree -- for the sake of this article -- that the following is not a far-fetched statement: *Polyglot code is a subset of portable code*. While polyglot code might be used for different reasons, *polyglot machine code* most likely won't. 

### So, what's the fuss?

I bet almost everyone (but the developers maybe) appreciates portable software. No matter the architecture, one can just compile what they need and use it. But what if there's a better way?

Obviously, I left a (blunt) hint in the previous section! *Polyglot code* is the ***better*** way to portability (at least from the end user's perspective). Have you ever been under the impression that compiling again is not necessary? Well, I have. What if I could just compile my code once and never have to worry about doing it again? 

That's the whole idea behind *polyglot machine code*: write once -- run everywhere. 

But before we continue with writing our little polyglot, let's examine whether we need it at all.

### The code

For our tests we're going to use the following program, which just reads machine code from a file given as its argument and executes it.

{% highlight c %}
#include <sys/mman.h>
#include <unistd.h>
#include <fcntl.h>
#include <limits.h>
#include <stdio.h>

#define MAX_SIZE 2048

char buf[MAX_SIZE];

int main(int argc, char** argv) {
  char* filename = argv[1];

  int f = open(filename, O_RDONLY);  
  int size = read(f, buf, MAX_SIZE);

  size_t pagesize = getpagesize();
  size_t start = ((size_t) buf) & -pagesize;
  
  mprotect((void*) start, MAX_SIZE, PROT_WRITE | PROT_EXEC);

  size_t bits = sizeof(void*) * CHAR_BIT;
  printf("Compiled for x86-%d\n", bits);
  printf("Loaded %d bytes of code\n", size);
    
  void (*fun)(void) = (void (*)(void)) buf;

  fun();
  
  return 0;
}

{% endhighlight %}

Now, let's write a piece of assembly that will generate our machine code. 

One thing to note before that: as I'm writing this post, I'm sitting on a Linux machine, hence the [ABI](https://en.wikipedia.org/wiki/Application_binary_interface) is different from what you could see in the [previous post]({{ "/articles/2017-05/hello-world" | absolute_url }}) -- for now you don't have to worry about it, I'm going to cover this in the following post. 

{% highlight nasm %}
BITS 32
	
	mov 	eax, 4
	push 	word 0x0a74
	push 	dword 0x69623233
	mov 	ebx, 1
	mov 	ecx, esp
	mov 	edx, 6
	int 	0x80
	mov 	eax, 1
	mov 	ebx, 123
	int 	0x80
{% endhighlight %}

Since we only want to translate our assembly code into machine code there are no sections and global symbols, instead we're going to turn it into raw bytes using

{% highlight sh %}
mewa@sea$ nasm poly32.asm -o poly32 && hexdump -C poly32 && echo --- hex end && ndisasm -b32 poly32
00000000  b8 04 00 00 00 66 68 74  0a 68 33 32 62 69 bb 01  |.....fht.h32bi..|
00000010  00 00 00 89 e1 ba 06 00  00 00 cd 80 b8 01 00 00  |................|
00000020  00 bb 7b 00 00 00 cd 80                           |..{.....|
00000028
--- hex end
00000000  B804000000        mov eax,0x4
00000005  6668740A          push word 0xa74
00000009  6833326269        push dword 0x69623233
0000000E  BB01000000        mov ebx,0x1
00000013  89E1              mov ecx,esp
00000015  BA06000000        mov edx,0x6
0000001A  CD80              int 0x80
0000001C  B801000000        mov eax,0x1
00000021  BB7B000000        mov ebx,0x7b
00000026  CD80              int 0x80
{% endhighlight %}

Let's compile our tester program for 32 bit x86 architecture and check the result

{% highlight sh %}
mewa@sea$ gcc -m32 tester.c -o tester
mewa@sea$ ./tester poly32; echo $?
Compiled for x86-32
Loaded 40 bytes of code
32bit
123
{% endhighlight %}

As expected, it worked. Now let's try it again for x86-64

{% highlight sh %}
mewa@sea$ gcc -m64 tester.c -o tester
mewa@sea$ ./tester poly32; echo $?
Compiled for x86-64
Loaded 40 bytes of code
123
{% endhighlight %}

Ok, so we got the exit code at least. But where is the string that was supposed to be printed? Let's analyze what is happening.

* we are running a 64 bit program
* we are then injecting 32 bit machine code into it
* the `exit` syscall is working, since we got the `123` status code
* the `write` syscall is somehow failing

{% highlight sh %}
mewa@sea$ gdb ./tester
{% endhighlight %}
{% highlight gdb %}
(gdb) disas main
...
0x0000000000000815 <+171>:lea    0xbd(%rip),%rdi        # 0x8d9
0x000000000000081c <+178>:mov    $0x0,%eax
0x0000000000000821 <+183>:callq  0x600 <printf@plt>
0x0000000000000826 <+188>:lea    0x200853(%rip),%rax        # 0x201080 <buf>
0x000000000000082d <+195>:mov    %rax,-0x8(%rbp)
0x0000000000000831 <+199>:mov    -0x8(%rbp),%rax
0x0000000000000835 <+203>:callq  *%rax
0x0000000000000837 <+205>:mov    $0x0,%eax
...
(gdb) break *main+203
(gdb) disp/4i $rip
(gdb) r poly32
1: x/4i $rip
=> 	0x555555554835 <main+203>:callq  *%rax
	0x555555554837 <main+205>:mov    $0x0,%eax
	0x55555555483c <main+210>:leaveq 
	0x55555555483d <main+211>:retq   
(gdb) si
1: x/4i $rip
=> 	0x555555755080 <buf>:mov      $0x4,%eax
	0x555555755085 <buf+5>:pushw  $0xa74
	0x555555755089 <buf+9>:pushq  $0x69623233
	0x55555575508e <buf+14>:mov   $0x1,%ebx
(gdb) si 4
=> 	0x555555755093 <buf+19>:mov    %esp,%ecx
	0x555555755095 <buf+21>:mov    $0x6,%edx
	0x55555575509a <buf+26>:int    $0x80		<-- write syscall
	0x55555575509c <buf+28>:mov    $0x1,%eax
# let's examine the return code
(gdb) disp $eax
2: $eax = 4
(gdb) si 3
1: x/4i $rip
=> 	0x55555575509c <buf+28>:mov    $0x1,%eax
	0x5555557550a1 <buf+33>:mov    $0x7b,%ebx
	0x5555557550a6 <buf+38>:int    $0x80
	0x5555557550a8 <buf+40>:add    %al,(%rax)
2: $eax = -14		<-- we got an error! errno == 14
{% endhighlight %}

Aha! According to the headers it's `EFAULT`, meaning something is wrong with address of the string passed. 

~~~
#define EFAULT          14      /* Bad address */
~~~
{% highlight gdb %}
(gdb) info reg esp
esp	0xffffdf0e	-8434
(gdb) x/s $esp
0xffffffffffffdf0e:	<error: Cannot access memory at address 0xffffffffffffdf0e>
{% endhighlight %}

Let's see. What is the stack pointer register pointing to? The **stack** obviously. It's an address, which is 64 bits wide, due to our program having been compiled for x86-64. And `esp` is a *32 bit* register. That's why the address pointed to by the lower half of the 64 bit `rsp`, which holds the stack pointer in 64 bit programs, is not within our process' address space and hence invalid! 

On a side note -- this is obviously a very lucky case, since we hit the backwards compatibility mode and our `int 0x80` syscalls were actually called, had it not existed it would've result in a **total disaster** (a.k.a a crash).

Anyway, now that we know it could be useful to distinguish between our runtime architecture let's get to the whole point of this article.

### The polyglot
In order to proceed we'll need 64 bit specific machine code
{% highlight nasm %}
BITS 64
	
	mov 	rax, 1
	sub	rsp, 0x06
	mov	word [rsp+4], 0x0a74
	mov 	dword [rsp], 0x69623436
	mov	rdi, 1
	mov 	rsi, rsp
	mov 	rdx, 6
	syscall
	mov 	rax, 60
	mov 	rdi, 123
	syscall
{% endhighlight %}

Let's see if it works
{% highlight sh %}
mewa@sea$ ./tester poly64; echo $?
Compiled for x86-64
Loaded 49 bytes of code
64bit
123
{% endhighlight %}

Now we have 2 separate machine codes and we need run them correspondingly. 

In order to detect what architecture we are currently running on, we have to extract the difference in information available from running the same piece of machine code based on the architecture. This implies that such piece of code has to be valid for both x86-32 and x86-64 but at the same time give different results. Sounds tricky? It certainly is if you're not familiar with the opcodes and their extensions.

But first things first -- I haven't mentioned what machine code *reallly* is. In order to give you a better view let's look closely what a CPU is and what it is not. First of all, it's not magic! It's entirely possible, although not the most convenient way imaginable, to write a program using nothing but raw sequences of bytes. That's because a CPU is just an electronic circuit that acts upon *instructions* given at its *input*. Based on what instruction is given it performs some computations, such as addition, decrementation, etc. While humans usually talk about instructions in general terms such as "addition", CPUs are given numbers which describe (encode the information) what action to perform, e.g. for a x86-32 machine `inc eax` would be translated to byte `0x40`, whereas `inc ebx` to `0x41`. When the CPU encounters `0x40` at its input it will perform incrementation of the `eax` register. Such numbers are called *opcodes* and machine code is just a *sequence of opcodes*. 

Luckily enough if you want to look for a specific opcode you don't have to dig through AMD's or Intel's manuals -- there already exist dedicated resources that have this information extracted. If you're curious you can look [here](http://ref.x86asm.net) for more information on this subject. 

To distinguish between the x86-32 and x86-64 architectures I'm going to use the exact `0x40` opcode. On x86-32, as mentioned above, it simply encodes an [incrementation](http://ref.x86asm.net/coder32.html#x40) instruction. But on x86-64 it [describes](http://ref.x86asm.net/coder64.html#x40) a REX prefix, so an opcode that is used to change the behaviour of an instruction. REX prefixes were only introduced on the x86-64 architecture and we're going to exploit that fact. 

We're going to deliberately place a REX prefix that will change the way our program behaves. 

{% highlight nasm %}
BITS 32
	
	xor	eax, eax
	db	0x40
	mov	bh, 1
	test	eax, eax
	jz	near x64
x86:
	nop			; x86-86 relevant code
x64:
	nop			; x86-64 relevant code
{% endhighlight %}

Let's use `ndisasm` to discover how our code would be interpreted. 
{% highlight sh %}
mewa@sea:polyglot$ nasm stub.asm -o stub && echo --- 32 bit && ndisasm -b32 stub && echo --- 64 bit && ndisasm -b64 stub
--- 32 bit
00000000  31C0              xor eax,eax
00000002  40                inc eax
00000003  B701              mov bh,0x1
00000005  85C0              test eax,eax
00000007  0F8401000000      jz near 0xe
0000000D  90                nop
0000000E  90                nop
--- 64 bit
00000000  31C0              xor eax,eax
00000002  40B701            mov dil,0x1
00000005  85C0              test eax,eax
00000007  0F8401000000      jz near 0xe
0000000D  90                nop
0000000E  90                nop
{% endhighlight %}

As you can see, even though we have the same sequence of bytes, they will act differently. The 32 bit version will increment the `eax` register, while the 64 bit version won't. We're then testing against eax and jump to 64 bit code if it's zero. 

Let's create our fully functional polyglot at last.

Here's the contents of `poly32.asm`

{% highlight nasm %}
BITS 32

	xor 	eax, eax
	db	0x40
	mov	bh, 1
	test 	eax, eax
	jz	x64
	
	mov 	eax, 4
	push 	word 0x0a74
	push 	dword 0x69623233
	mov 	ebx, 0
	mov 	ecx, esp
	mov 	edx, 6
	int 	0x80
	mov	eax, 1
	mov 	ebx, 123
	int 	0x80
x64:
{% endhighlight %}

and `poly64.asm`

{% highlight nasm %}
BITS 64

	mov 	rax, 1
	sub	rsp, 0x06
	mov	word [rsp+4], 0x0a74
	mov 	dword [rsp], 0x69623436
	xor	edi, edi
	mov 	rsi, rsp
	xor 	edx, edx
	mov 	rdx, 6
	syscall
	mov 	rax, 60
	mov 	rdi, 123
	syscall
{% endhighlight %}

Let's combine it together

{% highlight sh %}
mewa@sea$ nasm poly32.asm -o poly32 && nasm poly64.asm -o poly64
mewa@sea$ cat poly32 > poly && cat poly64 >> poly
{% endhighlight %}

The last thing remaining on our list is to test our little polyglot and see if it works!

{% highlight sh %}
mewa@sea$ gcc -m32 tester.c -o tester && ./tester poly; echo $?
Compiled for x86-32
Loaded 103 bytes of code
32bit
123
mewa@sea$ gcc -m64 tester.c -o tester && ./tester poly; echo $?
Compiled for x86-64
Loaded 103 bytes of code
64bit
123
{% endhighlight %}

### Great success!
We have successfully constructed our first polyglot! 

If you want to give it a try yourself all the sources are available on my [GitHub](https://github.com/mewa/polyglot-101).

As promised, we'll continue to explore the world of *machine code polyglots* and in the upcoming post we're going to take a look at how to handle different operating systems and tackle with their ABIs.

I hope you enjoyed this mini article and as always -- I encourage you to discuss further and leave your comments below!

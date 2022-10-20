---
layout: post
title: Something Boots!
---


# Progress report: Hello World in Aarch64 Assembly on Bare Metal

I got something to boot last night.


## Part 1: Hello World! (in virtual userspace)

I started with the arm assembly hello world [here](https://peterdn.com/post/2020/08/22/hello-world-in-arm64-assembly/). It looks something like this:

    /*hello.s*/
    .data
    
    /* Data segment: define our message string and calculate its length. */
    msg:
        .ascii        "Hello, ARM64!\n"
    len = . - msg
    
    .text
    
    /* Our application's entry point. */
    .globl _start
    _start:
        /* syscall write(int fd, const void *buf, size_t count) */
        mov     x0, #1      /* fd := STDOUT_FILENO */
        ldr     x1, =msg    /* buf := msg */
        ldr     x2, =len    /* count := len */
        mov     w8, #64     /* write is syscall #64 */
        svc     #0          /* invoke syscall */
    
        /* syscall exit(int status) */
        mov     x0, #0      /* status := 0 */
        mov     w8, #93     /* exit is syscall #93 */
        svc     #0          /* invoke syscall */

Install `binutils-arm-linux-gnueabihf` then:

    aarch64-linux-gnueabihf-as hello.s -o hello.o
    aarch64-linux-gnueabihf-ld hello.o -o hello

And now run it!

    qemu-aarch64 ./hello

But that&rsquo;s not very satsifying right? Those svc instructions are interrupts to invoke syscalls. Syscalls that are handled by an operating system. We want something that runs on bare metal, without an operating system. I mean, we&rsquo;re writing an operating system, right? (Yeah, yeah, an OS is a collection of things that don&rsquo;t fit into a language&#x2026; there shouldn&rsquo;t be one, and so on and so forth.) A bare metal lisp, a lisp from scratch, in the sense of LISP being the primary model of computation, where shutting down your lisp meant shutting down your computer, because to engage in computing was to compute, not merely to use an appliance as one uses a dishwasher.

And this does not run on bare metal. It presumes an operating system.

Anyway. So let&rsquo;s go further:


## Part 2: h(ello world!) on Bare (Virtual) Metal

I started with OS Dev&rsquo;s handy dandy [QEMU AArch64 Virt Bare Bones](https://wiki.osdev.org/QEMU_AArch64_Virt_Bare_Bones) explanation. Now, I&rsquo;m really not interested in writing a whole lot of C. I&rsquo;d like to go from assembly to lisp as quickly as possible with no intermediaries. So let&rsquo;s just look at the assembly to find the relevant instructions. Here&rsquo;s what the original kernel.c looks like:

    #include <stdint.h>
    
    volatile uint8_t *uart = (uint8_t *) 0x09000000;
    
    void putchar(char c) {
        *uart = c;
    }
    
    void print(const char *s) {
        while(*s != '\0') {
            putchar(*s);
            s++;
        }
    }
    
    void kmain(void) {
         print("Hello world!\n");
    }

And if you run  `aarch64-elf-gcc -ffreestanding -c kernel.c  -S` to look at the assembly you&rsquo;ll see a lot of junk. And I didn&rsquo;t understand that junk, so I pared kernel.c down to something approachable.

    #include <stdint.h>
    
    volatile uint8_t *uart = (uint8_t *) 0x09000000;
    
    void kmain(void) {
        *uart = 'h';
        //*uart = 'e';
        //*uart = 'l';
        //*uart = 'l';
        //*uart = 'o';
    }

And now we get this significantly more approachable bit of assembly:

    	.arch armv8-a
    	.file	"kernel.c"
    	.text
    	.global	uart
    	.data
    	.align	3
    	.type	uart, %object
    	.size	uart, 8
    uart:
    	.xword	150994944
    	.text
    	.align	2
    	.global	kmain
    	.type	kmain, %function
    kmain:
    .LFB0:
    	.cfi_startproc
    	adrp	x0, uart
    	add	x0, x0, :lo12:uart
    	ldr	x0, [x0]
    	mov	w1, 104
    	strb	w1, [x0]
    	nop
    	ret
    	.cfi_endproc
    .LFE0:
    	.size	kmain, .-kmain
    	.ident	"GCC: (Ubuntu 11.2.0-5ubuntu1) 11.2.0"
    	.section	.note.GNU-stack,"",@progbits

Now, that&rsquo;s quite alot. I don&rsquo;t think we can simplify too much further than that though.

There are a few relevant bits to understand here.


### Declaring the UART register

        .text
        .global uart
    uart:
        .xword 150994944

Now, 0x09000000 is the hex representation of 150994944, so it looks like `as` converted it to decimal here. So this snippet defines the address for the register uart&#x2026; or something (don&rsquo;t ask me, I&rsquo;m figuring it out as I go!)


### Getting the Correct Register for UART into x0

    adrp	x0, uart
    add	x0, x0, :lo12:uart
    ldr	x0, [x0]
    mov	w1, 104
    strb	w1, [x0]

I&rsquo;ve tried removing particular instructions from this segment, and they all seem to be essential.


### Actually Printing a Character to UART

    mov	w1, 104
    strb	w1, [x0]

The strb is what does the actual reading. [This stack overflow post](https://stackoverflow.com/a/25508561) was helpful, even though it&rsquo;s for an older form of ARM assembly.


### Ending the Procedure(?)

    nop
    ret

I think this is how you end procedures.


### Putting it all Together

So I tried to take it to bare essentials and this is what I got:

    /* hello_1_5 */
        .text
        .global uart
    uart:
        .xword 150994944
    
        .text
        .global _start
    _start:
        adrp x0, uart
        add x0, x0, :lo12:uart
        ldr x0, [x0]
    
        mov w1, 104
        strb w1, [x0]
    
        nop
        ret

    aarch64-linux-gnu-as hello_1_5.s -o hello_1_5.o && aarch64-linux-gnu-ld hello_1_5.o -o hello_1_5

    qemu-system-aarch64 -machine virt -cpu cortex-a57 -kernel hello_1_5 -nographic

And it prints `h`!


## Part 3: prompt (=>) hello world!

Now let&rsquo;s throw a couple more things in. With the `.text` and the `.local` declarations we can define procedures and/or variables; `ldrb` can read from the uart register the same way that `strb` writes to it; and procedures defined in assembly can be called recursively. What does that spell?! A listener loop!

    /* hello_2.s */
        .text
        .global uart
    uart:
        .xword 150994944
    
        .text
        .local listen
    listen:
        /* let's listen for a character typed in the terminal*/
        adrp x0, uart
        add x0, x0, :lo12:uart
        ldr x0, [x0]
        /* now we know that the actual address for uart is in x0 */
    
        ldrb w1, [x0] /* load the read character into w1 from uart */
    
        strb w1, [x0] /* echo the character loaded in w1 back to uart */
    
        b listen /* recur, b means branch, which I think is a procedure call */
    
        .text
        .global _start
    _start:
        adrp x0, uart
        add x0, x0, :lo12:uart
        ldr x0, [x0]
    
        mov w1, 61 /* 61 is '=' */
        mov w2, 62 /* 62 is '>' */
    
        strb w1, [x0] /* let's print our mock prompt */
        strb w2, [x0]
    
        b listen /* start the listener loop */
    
        nop
        ret

And now when we build that, we get a program that prints `=>` for a mock prompt, and then echoes whatever you type back to you. In other words&#x2026;

We have I/O running on (virtual) bare metal!


## Update: A Response and Clarification from /u/anydalch

I posted this to reddit a little while ago and /u/anydalch provided [this very helpful response](https://www.reddit.com/r/lisp/comments/t9zgd3/started_a_dev_blog_building_a_lisp_on_bare_metal/hzz186n/) clarifying how addressing works in AArch64.

> neat! if you haven&rsquo;t found it already, the arm architecture reference manual will be an invaluable resource on this journey: <https://developer.arm.com/documentation/ddi0487/ha/?lang=en> .
> 
> i also want to explain the `adrp` / `add` pattern you noticed in your c compiler&rsquo;s output, because it&rsquo;s something you&rsquo;ll want to be used to.
> 
> arm64 is designed for writing position-independent relocatable code, where it shouldn&rsquo;t matter what address you load your binary into; you just put it wherever and jump into it. as a result, all the immediate memory-access instructions use program-counter-relative offsets instead of absolute addresses. so you say, &ldquo;load from 128 bytes before this instruction,&rdquo; not &ldquo;load from the absolute address `0xabcdef`.&rdquo; your assembler will mostly make this transparent to you; when you write a symbol as the immediate argument to an instruction like your `b listen`, it will be automatically converted into a pc-relative offset. but if you want to put the address of a symbol into a register, you use a variant of the `adr` (&ldquo;address&rdquo;) instruction, which calculates a pc-relative address and stores that address in a register. (it&rsquo;s sort of similar to intel&rsquo;s `lea` instruction, if you&rsquo;re familiar with that, in that both of them offer an interface to the system&rsquo;s address calculator which gives you back the address instead of immediately using it in another operation.)
> 
> the bare `adr` instruction has a relatively short range, since the offset has to fit in an immediate value that gets encoded in a 32-bit instruction &#x2013; iirc, `adr` gives you a signed 12-bit offset. this poses a problem, because linkers often arrange for your code and data segments to be stored somewhat far apart, outside that range. enter `adrp` (&ldquo;address page&rdquo;), which calculates a 12-bit-aligned page address. the pattern you&rsquo;ll see a lot in code generated by c compilers is to use `adrp` to calculate a base, followed by an add to get an offset into that page.
> 
> the `adrp` / `add` pattern is so common, in fact, that arm defines a &ldquo;pseudo-instruction&rdquo; called `adrl` (&ldquo;address long&rdquo;) which expands to both. not all assemblers implement it (the clang assembler notably does not), but i think gnu as does. so you should be able to replace:
> 
>     adrp x0, uart
>     add x0, x0, :lo12:uart
> 
> with:
> 
>     adrl x0, uart


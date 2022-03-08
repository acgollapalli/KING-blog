---
layout: post
title: something boots!
---
#+TITLE: something boots!


# Progress report

I got something to boot last night.
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

There are a few relevant bits to understand here:

        .text
        .global uart
    uart:
        .xword 150994944

Now, 0x09000000 is the hex representation of 150994944, so it looks like `as` converted it to decimal here. So this snippet defines the address for the register uart&#x2026; or something (don&rsquo;t ask me, I&rsquo;m figuring it out as I go!)

    adrp	x0, uart
    add	x0, x0, :lo12:uart
    ldr	x0, [x0]
    mov	w1, 104
    strb	w1, [x0]

I&rsquo;ve tried removing particular instructions from this segment, and they all seem to be essential. The strb is what does the actual reading. [This stack overflow post](https://stackoverflow.com/a/25508561) was helpful, even though it&rsquo;s for an older form of ARM assembly.

    nop
    ret

I think this is how you end programs.

So I tried to take it to bare essentials

    /* hello_1_5 /*
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

And it prints H!

Now let&rsquo;s throw a couple more things in. With the `.text` and the `.local` declarations we can define procedures and/or variables; `ldrb` can read from the uart register the same way that `strb` writes to it; and procedures defined in assembly can be called recursively. What does that spell?! A listener loop!

    /* hello2.s */
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
        ldrb w1, [x0] /* store the read character into w1 */
    
        adrp x0, uart
        add x0, x0, :lo12:uart
        ldr x0, [x0]
        strb w1, [x0] /* echo the character stored in w1 back to uart */
    
        b listen /* recur, b means branch, which I think is a procedure call */
    
        .text
        .global _start
    _start:
        adrp x0, uart
        add x0, x0, :lo12:uart
        ldr x0, [x0]
        mov w1, 104
        strb w1, [x0]
    
        b listen /* start the listener loop */
    
        nop
        ret

And now when we build that, we get a program that prints h, and then when you type something in the terminal it will echo it back to you.

We have I/O running on (virtual) bare metal!


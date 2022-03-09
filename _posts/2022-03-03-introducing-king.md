---
layout: post
title: Introducing KING
---


# KING Is Not Genera

Well, that&rsquo;s the first thing I wanted to say, anyway. If you know what Symbolics Genera is, that tells you a lot about what this is, how ambitious it is and its likelihood of success or failure.


# Some Preliminaries


## What is this blog?

It&rsquo;s a dev blog.


## For what?

KING Is Not Genera.


## You told me that already. I don&rsquo;t know what it means.

It&rsquo;s a recursive acronym. KING for short.


## So if it&rsquo;s not Genera (whatever that is), what is it?

It&rsquo;s a Lisp.


## Okay, so it&rsquo;s a programming language.

No.


## What?

Lisp is not a programming language, Lisp is a way of interfacing with the computer. It&rsquo;s as much an operating system as it is a programming language.
For instance. Emacs Lisp is not a lisp. Emacs is a lisp, which has Emacs Lisp as its programming language.


## What?

Think of it like this: Lisp is a process that you interface with to read lisp, write lisp, edit lisp, etc., and everything else that you do with a computer is just a higher-order usage of lisp. For instance your graphics driver is taking a lazy-list of frames from a video-file, which is a list of frames, each of which is an image, which is just an array of pixel values, and maybe some timecode metadata, to sync it to the audio, which is just a list of different amplitude values and timecode values, and both your video and audio can be handled the same way as any other kind of lisp data type, because they&rsquo;re just lisp data types, because it&rsquo;s all part of one program, lisp, that runs on your computer, with nothing in between you and it, or between it and the machine. Lisp is the program that you teach you to do everything you want it to, including write programs.


## Okay&#x2026; What&rsquo;s Genera?

Genera is [awesome](https://www.youtube.com/watch?v=jACcgLfyiyM).


## So why is KING not Genera?

Because Genera is not open source, and even though Symbolics (the company that made Genera) shut down the year I was born, you **still can&rsquo;t legally get your hands on Genera**. So I&rsquo;m going to write my own lisp. A lisp from scratch. The king of all programs. And I&rsquo;ll call it KING.

I&rsquo;d recommend you read part 4 of Stephen Levy&rsquo;s book [Hackers](http://index-of.es/Hack/Steven%20Levy%20-%20Hackers%20Heroes%20of%20the%20Computer%20Revolution%20-%202010.pdf), about how the GNU project got started, and the relationship between that project and the Lisp Machine.


## So has anyone ever done this before?

-   Well, it&rsquo;s [definitely](https://github.com/mntmn/interim) [possible](https://github.com/vygr/ChrysaLisp) [that](https://github.com/whily/yalo) [I&rsquo;m](https://github.com/froggey/Mezzano) [the](https://movitz.common-lisp.dev/) [first](https://www.bogodyne.com/category/software/) [one](https://tumbleweed.nu/lm-3/) [in](https://github.com/fjames86/flisp) [the](https://www.researchgate.net/publication/228351943_KnowOS_The_re_birth_of_the_knowledge_operating_system) [history](https://github.com/akkartik/mu) [of](http://metamodular.com/lispos.pdf) [ever](https://github.com/tokamach/beige) [to](http://armpit.sourceforge.net/) [even](https://youtu.be/I_4Fb7mOtDc) [think](https://github.com/GitoriousLispBackup/lambdapi) [about](http://www.loper-os.org/?p=8) [attempting](http://tunes.org/) [it](https://www.mail-archive.com/picolisp@software-lab.de/msg04823.html), [totally](https://picolisp.com/wiki/?PilOS) [100%](https://luksamuk.codes/posts/lispm-001.html) [visionary](https://www.makerlisp.com/).


## So do you have a plan?

Kind of. We&rsquo;ll make it up as we go.


# A Glimmer of a Plan


## Step 0: Architecture

We&rsquo;re targeting the [MNT hardware](https://mntre.com/media/reform_md/2020-05-08-the-much-more-personal-computer.html). Their laptop has a hyper key. That&rsquo;s good enough for us. It means we&rsquo;re targeting ARMv8. Eventually (hopefully), it means ARMv8.5+ with the Memory Tagging Extension.

Also, Lukas at MNT is kinda [our guy](https://github.com/mntmn/interim). And there are [other projects](https://www.youtube.com/watch?v=I_4Fb7mOtDc) targeting similar hardware, so it looks like it&rsquo;s kind of the place to be.


## Step 1: A Bootstrap Lisp

We need to create a bootstrap lisp.
Why? So we&rsquo;re not writing the whole damn thing in assembly.
Eventually, it&rsquo;d be nice to have an assembly function, which takes something like this:

    (asm (:mov x0 x2)
         (:add x0 x1))

and turns it into machine code.
But for right now, so that we aren&rsquo;t completely stuck, we&rsquo;re writing it using GNU assembler.


## Step 2: EXCALIBUR

All lisps are the same in their bare essentials. The sort of data structures, data literals, valid symbols, scoping (dynamic or lexical), macros vs fexprs, lisp-1 vs lisp-2, define vs defun vs defn, Common-Lisp vs Scheme vs Clojure-like vs pico-lisp style vs kernel or something else entirely? Different question. If all anyone gets out of this is the bootstrap lisp, well that would make me very sad, but it would be good for everyone else, because it would mean that every young lisp wizard would be empowered to make their own lisp from our bootstrap lisp, and evolve that lisp into the reflection of their own mind, up to the point where it became what they used for both work and play, building in standards from libraries in order to do real work, but nevertheless in all things which they may choose, that which they *have* chosen rather than the appliance-like default selected for them by some OS designer external to themselves. Wait&#x2026; what were we talking about? Right, this is about *my* lisp. more to come on this.


### Step 2a: YOUR-KINGDOM (persistence)

Instead of having a filesystem, we&rsquo;ll store all data in namespaces as lisp objects under symbols. Namespaces can be stored within other namespaces if they&rsquo;re part of the same larger project. The top level namespace is your castle, sort of like a symbolics world.

Now dealing with that is a bit of a pain in a multi-address-space world, which is why KING is single address space. I have no idea how security will be handled, but there has been work on the topic, and I suspect that given sufficiently advanced metadata semantics, the fact that all data in memory is lisp data and shares lisp semantics will provide us adequate defenses.


## Step 3: Draw a pixel

We can only communicate like a teletype for so long before we get sick of that. So eventually we&rsquo;re going to need to be able to draw to the screen. That means a graphics library.


## Step 4: GALAHAD (Editor)

More to come&#x2026;


## Step 4: ROUND TABLE (Process Control)


## Step ?: LANCELOT (Networking and Browser)


## Step ?: PERCIVAL (other languages)


## Step ?: MERLIN (Compiler)


# Some caveats

This is meant to be a long term project. It&rsquo;s also meant to be a way for me to learn, and for all others who are interested to learn with me. If it ends up being anything more than that, I&rsquo;ll be ecstatic. Each knight (individual subprocess) will be minimalistic in scope, simply because it&rsquo;s too much for one person. If it actually gets done, then it would be&#x2026; well honestly it&rsquo;d be pretty comparable to Temple OS. Let&rsquo;s follow that comparison for a bit. My hope is that, if Temple OS be the Altar of Computation, that KING shall be its Throne.


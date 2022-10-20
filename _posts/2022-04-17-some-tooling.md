---
layout: post
title: Some Tooling
---

I&rsquo;ve been working primarily on a couple posts for memory management. Keep an eye out for those. The trouble is that, in order to actually so much as cons a pair, you basically need to have memory all figured out. You need to:

-   Parse the DeviceTreeBlob to find areas of reserved memory
-   Have a way of allocating and de-allocating memory

And *that* is a very large topic. One that will take me some time.

But, we can still make progress (so we can blog about it), taking the advice of Sussman and Abelson and using the wonderful power of wishful thinking.

We&rsquo;re going to build some tooling. And we&rsquo;re going to do it in emacs lisp.


# So Introducing: Our as yet unnamed lisp compiler


## Why Emacs Lisp?

-   It&rsquo;s readily available on every platform one could ever want to work on.
-   This blog is written in org-mode, using emacs, meaning that emacs lisp is very convenient.
-   It is the most widely distributed, continuously developed, and often used lisp in the history of lisp.
-   It probably has more people hacking in it than scheme, common lisp, and clojure combined.
-   I want to.


## So let&rsquo;s get started then:


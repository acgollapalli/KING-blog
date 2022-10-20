---
layout: post
title: Some Tooling
---

EXPORT<sub>FILE</sub><sub>NAME</sub>: ../<sub>posts</sub>/2022-04-16-some-tooling.md

Okay, so I&rsquo;ve been working on posts on memory management, because to do almost anything lispy you need to be able to do some memory management. Lisp is a dynamic language after all. You can check[ the repo for this blog](https://github.com/acgollapalli/KING-blog/tree/gh-pages/posts) to see the progress on those, but in the meanwhile, I&rsquo;m taking Gerald Sussman&rsquo;s advice to &ldquo;use wishful thinking&rdquo;. How? By pretending I&rsquo;ve got an assembly lisp.

For instance:

    (defλ some-fn
      [x0 x1 x2 x3 & x4]
      (some-stuff x1 x3)
      (compute-return-value x0 x4))

Becomes something like:

        .text
        .global some-fn
    some-fn:
        /* store frame pointer and return address on stack */
        stp x29, x30 [sp, #-32]!
        /* store stack pointer as new frame pointer */
        mov x29, [sp]
    
        /* store all the arguments on the stack*/
        /* (we're assuming all arguments are coming to us evaluated) */
        stp x0, x1 [sp, #-32]!
        stp x2, x3 [sp, #-32]!
        str x4 [sp, #-16]
    
        /* getting the arguments for some-stuff and calling it */
        ldr x0, [sp], #64         // get x1 (second arg)
        ldr x1, [sp], #32         // get x3 (fourth arg)
        bl some-stuff             // call some-stuff
    
        /* getting the arguments for compute-return-value and calling it */
        ldr x0, [sp], #80         // get x0 (first arg)
        ldr x1, [sp], #16         // get x4 (& args)
        bl compute-return-value   // call compute-return-value
    
        /* pointer to return value is in x0 */
    
        ldp x29, x30, [sp], #112 /* get back frame pointer and return address */
        ret /* return to point in x30 */

[This resource](https://diveintosystems.org/book/C9-ARM64/functions.html) was very helpful.

We don&rsquo;t have a heap. So our pointers don&rsquo;t actually point to anything yet. But We actually have something that looks like a procedure call now.

Let&rsquo;s actually try to write some emacs-lisp to turn our pseudo-lisp into assembly.

    (with-current-buffer (find-file-noselect
                          (concat blog-directory
                                  "posts/Some Tooling.org"))
        (goto-char 650)
        (read (current-buffer)))

The actual source code for our pseudo-lisp starts at char 650 in this file. The above snippet returns a parsed version of our pseudo-lisp as actual s-expressions. And that means that we can treat our pseudo-lisp like any other s-expressions, which emacs-lisp, like all lisps, happens to be exceptionally good at parsing.

For instance, let&rsquo;s say we want the very first value of our pseudo-lisp.

    (with-current-buffer (find-file-noselect
                          (concat blog-directory
                                  "posts/Some Tooling.org"))
        (goto-char 650)
        (car (read (current-buffer)))) ;=> defλ

How about the argument list?

    (with-current-buffer (find-file-noselect
                          (concat blog-directory
                                  "posts/Some Tooling.org"))
        (goto-char 650)
        (caddr (read (current-buffer)))) ;=> [x0 x1 x2 x3 & x4]

And the second argument?

    (with-current-buffer (find-file-noselect
                          (concat blog-directory
                                  "posts/Some Tooling.org"))
        (goto-char 650)
        (aref
         (caddr (read (current-buffer)))
         1)) ;=> x1

For this example, the second argument is just the name of the register that the argument is representing.


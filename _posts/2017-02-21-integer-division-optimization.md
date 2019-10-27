---
layout: post
status: publish
published: true
title: Fun with integer division optimizations.
author: Travis Mick
date: 2017-02-21
---
I recently stumbled across a [post about some crazy optimization](https://zneak.github.io/fcd/2017/02/19/divisions.html) that clang does to divisions by a constant. If you aren't interested in reading it yourself, the summary is as follows:

* Arbitrary integer division is slow.
* Division by powers of 2 is fast.
* Given a divisor $$n$$, the compiler finds some $$a, b$$ such that $$a/2^b$$ approximates $$1/n$$.
* This approximation gives exact results for any 32-bit integer.

<!-- more -->

I was interested in seeing just how much faster this seemingly-magic optimization was than the regular `div` instruction, so I set up a simple test framework:

```c
#include <stdio.h>
#include <sys/time.h>

extern unsigned udiv19(unsigned arg);

int main(int argc, char **argv) {

    struct timeval start, finish;
    
    unsigned long long accum = 0;
    int j;

    for(j = 0; j < 1000; j++){
        gettimeofday(&amp;start, NULL);

        int i;
        for(i = 0; i < 1000000; i++){
            udiv19(1000);
        }
        
        gettimeofday(&amp;finish, NULL);

        accum += ((1000000l * finish.tv_sec + finish.tv_usec) - (1000000l * start.tv_sec + start.tv_usec));
    }
    
    printf("%f microseconds per 1M operations\n", (float) accum / 1000);

    return 0;

}
```

If you don't feel like reading C, the algorithm is basically as follows:

* Repeat 1,000 times:
  * Start a stopwatch.
  * Repeat 1,000,000 times:
    * Call the assembly function.
  * Stop the stopwatch and add the time elapsed to the total.
* Spit out the total time elapsed, divided by 1,000. This gives the average time for 1,000,000 calls to the assembly function.

I took the assembly from the other blog post and marked it up to be able to compile it into this scaffolding:

```asm
.intel_syntax noprefix
.text
.global udiv19
udiv19:
    mov     ecx, edi
    mov     eax, 2938661835
    imul    rax, rcx
    shr     rax, 32
    sub     edi, eax
    shr     edi
    add     eax, edi
    shr     eax, 4
    ret
```

I also wrote a naive version using the `div` instruction (this took a bit of googling to refresh my knowledge of the x86 ABI and instruction set...):

```asm
.intel_syntax noprefix
.text
.global udiv19
udiv19:
    mov     edx, 0
    mov     ecx, 19
    mov     eax, edi
    div     ecx
    ret
```

To compile:

```bash
$ gcc -o optimized main.c optimized.s 
$ gcc -o simple main.c simple.s
```

The results:

```bash
$ ./optimized 
1861.276001 microseconds per 1M operations
$ ./simple 
2783.295898 microseconds per 1M operations
```

We can clearly see a ~1.5x speedup (at least on my i5-6400). However, there's a caveat: we're only really testing the *latency* of the implementation, not its *throughput*. Modern processors have sophisticated ways to increase performance by pipelining instructions and executing them out of order. In a real application, this would drastically reduce the impact of the *div* instruction's latency. However, it's still interesting to see this optimization technique and prove that it theoretically *could* be beneficial, maybe, in some particular instances.


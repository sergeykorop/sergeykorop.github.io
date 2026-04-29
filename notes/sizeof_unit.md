# Pitfall of `sizeof()` or How Wide is Your Byte?

Published: 2026-04-29.

## Introduction
This note has been inspired by
[post](https://www.linkedin.com/posts/prisin6_include-cprogramming-embeddedsystems-activity-7406991651350540288-Vb9V)
brought to my newsfeed by a magic of LinkedIn recommender. We will
discuss a C code snippet presented in that post which could be a good
showcase for some problem with _portability_.

The intended audience for this note is beginning software developers,
so we will start from defining the idea of portability per se, then
present the code and explain the problem it has. Finally, we refer to
the C language reference to see the ground truth behind this case.

## The Idea of Portability
Informally, the idea of portability is simple: some program is
portable if it behaves reasonably the same when it runs on different
hardware platforms, OSes or language implementations.

The C language always put the efficiency first, so the idea of
portable C program has one more property: it should also be as
efficient across different platforms as possible. This requirement is
crucial: in order to fulfill it, some language properties, such as
data type value ranges, were made _implementation-dependent_ and
portable program should take this into account.

## Is This Code Portable?
So, it's time to dive into our code. The post which we are discussing
here, explained how to discover the internal representation of some
data types with the pointer to `char`. The code below slightly differs
from the original: I've changed it to expose more interesting effects
related to portability but left intact the problem which inspired me
on this writing.
```c
#include <stdio.h>

int main(void)
{
    unsigned long n = 0x12345678UL;
    unsigned char *p = (unsigned char *)&n;
    
    printf("Numeric value: 0x%lX\n", n);
    printf("In-memory representation, bytes:\n");
    
    for(size_t i = 0; i < sizeof(n); ++i)
    {
        printf("Address: %p, value: 0x%02x\n", (void *)(p + i), *(p + i));
    }
    
    return 0;
}
```

### When Portability Works ...
At the first glance, this code looks pretty good, right? Let's try
running it with different C implementations on two hardware platforms:
32-bit (x86) and 64-bit (amd64, also known as x64).

For x86 platform and 32-bit Visual C++ 2022 (19.44.35213) we have the
following output:
```text
Numeric value: 0x12345678
In-memory representation, bytes:
Address: 006FFE40, value: 0x78
Address: 006FFE41, value: 0x56
Address: 006FFE42, value: 0x34
Address: 006FFE43, value: 0x12
```
Everything looks correct: we can see how our number is stored in
memory. Additionally, we have one more evidence that our hardware is
[little-endian](https://en.wikipedia.org/wiki/Endianness).

Going further, let's try the same code built with the same version of
the compiler but this time targeted to produce the executable file for
amd64 architecture:
```text
Numeric value: 0x12345678
In-memory representation, bytes:
Address: 0000001C98EFFEB4, value: 0x78
Address: 0000001C98EFFEB5, value: 0x56
Address: 0000001C98EFFEB6, value: 0x34
Address: 0000001C98EFFEB7, value: 0x12
```
Do you see the difference? Yes, indeed, the memory addresses are now
64-bit wide but our data are the same.

Finally, we will run our code on 64-bit Debian 12 with GCC 12.2.0 
(also amd64): 
```text
Numeric value: 0x12345678
In-memory representation, bytes:
Address: 0x7ffd57938988, value: 0x78
Address: 0x7ffd57938989, value: 0x56
Address: 0x7ffd5793898a, value: 0x34
Address: 0x7ffd5793898b, value: 0x12
Address: 0x7ffd5793898c, value: 0x00
Address: 0x7ffd5793898d, value: 0x00
Address: 0x7ffd5793898e, value: 0x00
Address: 0x7ffd5793898f, value: 0x00
```
Surprise! Now we have even more difference: the output includes some
additional memory filled with zeros.

What does it all mean, anyway? Can we conclude that something is wrong
and our code is not portable? This time the answer would be "no". The
difference we see comes from the C language property we have mentioned
earlier: for the sake of efficiency, the same data types can have
different value ranges (and, therefore, vary in size) on different
platforms.  This is exactly that we can see: when we switch from x86
to amd64, our 32-bit pointers become 64-bit, which is quite natural,
while our numeric value, remaining 32-bit on Windows, suddenly turns
into 64-bit on Linux.

These sets of predefined data type sizes are known as _data models_.
Some of possible combinations are summarized
[here](https://en.wikipedia.org/wiki/64-bit_computing#64-bit_data_models).
They usually referred by their names which speak for themselves.
In our case we have `ILP32` (`int`, `long int` and pointers, all 32-bit),
`LLP64` (64-bits begin at `long long int` up to pointers, so
unmentioned `int` and `long int` remain 32-bit), and `LP64`
(everything from `long int` to pointers are 64-bit), respectively.

Taking this into account, we can conclude that our program, while
working differently, produces some output which is _reasonably the
same_ anyway.

How our code manages all these differences in data type sizes?
Obviously, using `sizeof()` which determines these platform-dependent
numbers and arranges the appropriate number of iterations.

### ... And When It Fails
So far, so good, we have tested our code running with different
compilers, OSes and, to some extent, hardware. We could see that
differences among these environments exist, but they were successfully
handled without making any changes in source code. Is it enough to
conclude that everything is fine?

It's time to reveal our secret and say "yes... but no". There is a
family of specialized DSPs from Texas Instruments, known as TMS320C3x.
If we could put our hands on some of those systems and run our code
there, we could see something like this (with some real address value
instead of question marks):
```text
Numeric value: 0x12345678
In-memory representation, bytes:
Address: ????????, value: 0x12345678
```
What? We have done just one iteration, printed the entire number instead
of its components and that's all? There should definitely be something
wrong with compiler!

Fortunately, everything works properly. If we take a look into
[TMS320C3x/C4x Optimizing C Compiler User’s Guide](https://www.ti.com/lit/ug/spru034h/spru034h.pdf),
pg. 3-5, we can read:

> **Note: A TMS320C3x/C4x Byte Is 32 Bits**
>
> The ANSI definition specifies that the sizeof operator yields the number
> of **bytes** required to store an object. ANSI further stipulates that
> when the sizeof operator is applied to type char, the result is one.
> Since the TMS320C3x/C4x char is 32 bits (to make it separately addressable),
> a byte is also 32 bits. This yields results that you may not expect; 
> for example, sizeof (int) == 1 (not 4).
> TMS320C3x/C4x bytes and words are equivalent (32 bits).

This definition perfectly explains what we have just seen, and we must
agree that it doesn't look like _reasonably the same_ to previous
results, so we should conclude that our goal of writing portable code
was not reached.

The natural reaction to this case could be something like: "Ok, we
indeed have this problem, but isn't this platform a kind of niche
solution?  Why should we bother to support them if they don't ever
know that byte has 8 bits?"

Such opinion really makes sense. Any seasoned software developer
working on real-life projects clearly understands the importance of
keeping the cost-benefit ratio in good shape. Putting it simple, if
your code works well in environments specified by project
requirements, nobody will care about making it 100% portable.

However, if we take the other side of this dispute and put ourselves
into the shoes of the portability purists, we must also agree that
there is no gray zone: your code is either portable and, therefore,
works everythere, or, being honest, it is not portable.

So, if we decided to take this maximalist point of view (just for the
time you are reading this note), how should we achieve our goal? How
can we make sure we can write something portably without testing it
with all possible hardware and compilers which existed sometimes,
exist now or will probably be created?

Fortunately, we don't need such a collection. The recipe of writing
portable code is easy and hard at the same time: _use the Standard,
Luke!_ As long as your language implementation is standard-conformant
and your program is written rigorously following its requirements,
this program should work in the same way everytime everywhere, varying
only within well-specified boundaries (or, as we said a thousand
times, behaving reasonably the same).

## The Ground Truth

_"The time has come," the Walrus said, "To talk of many things..."_
In our case, this would be fulfilling the promise and get the ultimate
verdict in our case from the C language specification.

The official ISO C standard is not freely available online, so I would
refer to [cppreference](https://cppreference.com/) which, despite its
name, also has information about C.

Let's see what is said about
[`sizeof()`](https://en.cppreference.com/c/language/sizeof):

> Queries size of the object or type.
>
> Used when actual size of the object must be known.  
> **Syntax**  
> `sizeof(` _type_ `)` 	 (1)  
> `sizeof` _expression_  (2)  
>
> Both versions return a value of type `size_t`.
> 
> **Explanation**  
> 1) Returns the size, in bytes, of the object representation of _type_  
> 2) Returns the size, in bytes, of the object representation of the type of _expression_.
>    No implicit conversions are applied to _expression_.
>
> **Notes**
>
> Depending on the computer architecture, a byte may consist of 8 or more bits, 
> the exact number provided as `CHAR_BIT`.
>
> `sizeof(char)`, `sizeof(signed char)`, and `sizeof(unsigned char)` always return 1.

The reference explicitly states that:

1) The size of data types is measured in _bytes_, but...

2) These bytes **do not have** to be always 8-bit wide!

This leads us a pretty contradicting conclusion: if you ask someone
which units are used by `sizeof`, the answer "bytes" could be either
right or wrong depending on _what_ byte definition is assumed. And
with probability of 99% or higher, it would mean the ubiquitous 8-bit
byte which is totally irrelevant to this particular context.

Such peculiar treatment of bytes is, probably, one of the most
surprising discoveries someone could make learning C nowadays. As IT
history hobbyist, I knew that old hardware architectures indeed have
different byte sizes. I also knew that network protocol specifications
use word "octet" instead of "byte" to make these specifications more
machine-independent. Finally, I knew about this pitfall with
`sizeof`... But I could never imagine, however, that those subjects
are so closely related to each other! If someone asked me this
question before I began digging the corresponding part of the language
reference preparing this note, I would answer that `sizeof` measures sizes
in units of type `char`. A bit closer to the ground truth but not to
the point.

One more question which we should also clarify would be: okay, we have
learned that byte can have different size, but how could it happen
that size of `char` type is equal to the size of `long`? Aren't they
dedicated for representing quantities of different magnitude in
memory-efficient manner?

Fortunately, reading the 
[reference](https://en.cppreference.com/c/language/arithmetic_types#Integer_types)
we can explain that as well: C language definition specifies only
_minimal_ bit width of each integer type, from `char` to `long long`
and finally states that

> Besides the minimal bit counts, the C Standard guarantees that
> ```text
>    1 == sizeof(char) ≤ sizeof(short) ≤ sizeof(int) ≤ sizeof(long) ≤ sizeof(long long).
> ```
> Note: this allows the extreme case in which byte are sized 64 bits, all types (including char)
> are 64 bits wide, and `sizeof` returns 1 for every type. 

Note that `≤` sign and the final remark. TMS320 data model passes this
exam, again.

Finally, we could have one more question: is this definition of byte
specific to this very language standard or has some wider use? A kind
of answer can be found in
[ISO/IEC 2382:2015(en) Information technology — Vocabulary](https://www.iso.org/obp/ui/en/#iso:std:iso-iec:2382:ed-1:v2:en):
> **byte**
> 
> string that consists of a number of bits, treated as a unit, and usually representing 
> a character or a part of a character  
> Note 1 to entry: The number of bits in a byte is fixed for a given data processing system.  
> Note 2 to entry: The number of bits in a byte is usually 8.  
> Note 3 to entry: byte: term and definition standardized by ISO/IEC \[ISO/IEC 2382-1:1993; ISO/IEC 2382-4:1999].  
> Note 4 to entry: 01.02.09 (2382)

So, we can conclude that the meaning of byte from the C Language
Standard is a bit unusual but to some extent accepted in the IT
community. From the other hand, if we refer to
[POSIX.1-2024](https://pubs.opengroup.org/onlinepubs/9799919799/basedefs/limits.h.html)
we can see that standard header file `limits.h` should contain
> ```text
> {CHAR_BIT}  
>     Number of bits in a type char.  
>     Value: 8
>```

making byte size _exactly_ 8 bits.

## Can We Fix It?
This time the answer would be my favourite: "it depends". As far as we
are aware of the problem with varying byte size, can we try to
overcome it using
[fixed-width types](https://en.cppreference.com/c/types/integer),
such as `uint8_t`?  Unfortunately, we can't: these types are only
defined for the platforms where some type containing exactly 8 bits
actually exists. So, in case of TMS320, code using `uint8_t` simply
couldn't be built.

The key to the solution would be using `unsigned char` to divide some
more complex data type to bytes but then determine their actual bit
width from `CHAR_BIT` and extract octets with shifts and bit
masks. This could be quite tricky, though: while byte size is always
at least 8 bits, nobody promised it will also be a multiple of 8 so
portable code working with octets should be ready to cross the byte
boundaries.

However, even using bit-fiddling we won't be able to solve a problem
which assumes that octets are individually addressable. That is, we
can change our original code to output binary representation of the
number in 8-bit chunks, but we remain unable to print them together
with their addresses.

## Conclusion
In this note, we have considered the notion of portability both in
general and specifically for C language. We could also see how balance
between portability and efficiency leads to writing C programs which
can behave differently on different platforms. Some of these
differences were quite clear and expected while others looked so
unusual that we suspected some problem with particular language
implementation.

At this point we figured out that the only reliable way to reason
about portability would be following the language standard with full
rigor. We could see, how unusual, from the common sense point of view,
could it be: some of us learned that bytes are not always 8-bit wide,
while others, including me, that the notion of platform-specific byte
width still be used today.

Finally, we considered the price of being 100% portable, and saw how
high could it be. Should we always try at our best and fight for that
goal? In real-world projects, of course, we shouldn't. However, in my
opinion, understanding those dark corners of the language is highly
desirable and expected from a good engineer.
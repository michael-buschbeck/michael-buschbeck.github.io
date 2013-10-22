---
layout: post
category: arduino
tags: [arduino, avr, c++]
title: There's always an edge case
---

> This post is a follow-up to [Your fla- Your flash is now clean](/arduino/2013/10/20/string-merging-pstr/).
> An updated, improved version of the modified `PSTR()` macro is at the bottom of this page.

******

I solicited feedback in the [Arduino forums](http://forum.arduino.cc//index.php?topic=194603.0)
to my de-duplicating `PSTR()` macro replacement described in last Sunday's [Your fla- Your flash is now clean](/arduino/2013/10/20/string-merging-pstr/) post.
It didn't take long until someone had found an edge case where my construct broke the build.

Darryl [points out](http://forum.arduino.cc//index.php?topic=194603.msg1437819#msg1437819)
that the following (perfectly reasonable) use of the updated `PSTR()` macro (via the `F()` macro) makes the compiler barf:

{% highlight cpp %}
Serial.println(F("%"));
{% endhighlight %}

This innocuous line causes the compiler to say this:

    PSTR.ino: In function 'void setup()':
    PSTR.ino:15:6: error: invalid 'asm': invalid %-code

Uhh. Well, darn.

In hindsight, that's not so very surprising, though.
Given `"%"`, the updated `PSTR()` macro expands to something like this:

{% highlight cpp %}
PGM_P ptr;
asm volatile
(
  ".pushsection .progmem.data, \"SM\", @progbits, 1" "\n\t"
  "PSTR%=: .string \"%\""  /* oops! */               "\n\t"
  ".popsection"                                      "\n\t"
  "ldi %A0, lo8(PSTR%=)"                             "\n\t"
  "ldi %B0, hi8(PSTR%=)"                             "\n\t"
  : "=d" (ptr)
);
{% endhighlight %}

Can you see it? The `%\"` next to the `.string` directive looks a lot like those `%A0` and `%B0` and `%=` placeholders, except it doesn't.
It's a good thing Darryl didn't use a string like "&pound;2&times;50%=&pound;1", or he'd have ended up with gibberish instead of a proper compiler error.

The good news is that GCC's `asm` directive only attempts to interpret "%-codes" if any outputs are declared.

Of course, I *do* need outputs to get the string address out.
But I can split the `asm` statement into two consecutive ones, the first one defining the string, the second one getting its address.

There's a problem with that, too, though, because now I can't use the `%=` construct to generate a unique label for the string.
(And `%=` would resolve to different values in those two `asm` blocks anyway.)
But fortunately, the assembler has a concept of [local labels](http://tigcc.ticalc.org/doc/gnuasm.html#SEC48L) that seems to work even in my case.

Long story short, here's an updated version that works even in the `F("%")` case:

{% highlight cpp %}
#define PSTR(str) \
  (__extension__({ \
    PGM_P ptr;  \
    asm volatile \
    ( \
      ".pushsection .progmem.data, \"SM\", @progbits, 1" "\n\t" \
      "0: .string " #str                                 "\n\t" \
      ".popsection"                                      "\n\t" \
    ); \
    asm volatile \
    ( \
      "ldi %A0, lo8(0b)"                                 "\n\t" \
      "ldi %B0, hi8(0b)"                                 "\n\t" \
      : "=d" (ptr) \
    ); \
    ptr; \
  }))
{% endhighlight %}

******

**Side note:**
I tested this with some statements like these:

{% highlight cpp %}
Serial.print(F("%"));
Serial.print(F("%%"));
Serial.print(F("%%%"));
{% endhighlight %}

and it turns out the linker actually cleverly merges those three strings *into each other*.
In the resulting ELF file, only "%%%" remains, and "%" and "%%" end up pointing *into* that string at the appropriate offset.
Nice!

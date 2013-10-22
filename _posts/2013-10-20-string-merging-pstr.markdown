---
layout: post
category: arduino
tags: [arduino, avr, c++]
title: Your fla- Your flash is now clean
---

> **Update (22 October):**
> The solution shown at the end of this page has an important edge-case issue that was pointed out on the [Arduino forums](http://forum.arduino.cc//index.php?topic=194603).
> The follow-up post, [There's always an edge case](/arduino/2013/10/22/string-merging-pstr-percent-codes/), has an updated version.

******

> **tl;dr:**
> The `PSTR()` macro wastes flash memory because it doesn't support de-duplication of identical string literals (string merging).
> At the very bottom of this page you'll find a drop-in replacement of the `PSTR()` macro that doesn't have this limitation.

******

(The title is a reference to [this excellent movie](http://en.wikipedia.org/wiki/Idiocracy), specifically [this scene](http://www.youtube.com/watch?v=OpFUrjq8nWE).)

I'm currently working on an Arduino project that uses a knock sensor and the [Music Shield](https://github.com/michael-buschbeck/arduino/tree/master/Music) to interact with its user.

The Arduino app prompts the user by playing MP3 audio files.
There are quite a handful of those files on the SD card, organized in a directory hierarchy, and to play them I've got a lot of lines that go more or less like this:

{% highlight cpp %}
play(F("ENROLL/"), F("I-CHALL/"), F("PROMPT"), F(".MP3"));

play(F("ENROLL/"), F("I-CHALL/"), F("X-NM1"), F(".MP3"));

switch (escalation) {
  case 1:  play(F("ENROLL/"), F("I-CONF/"), F("X-NM1"), F(".MP3"));
  case 2:  play(F("ENROLL/"), F("I-CONF/"), F("X-NM2"), F(".MP3"));
  default: play(F("ENROLL/"), F("I-CONF/"), F("X-NM3"), F(".MP3"));
}
{% endhighlight %}

The paths are concatenated at runtime within `play()` because the [SD](http://arduino.cc/en/Reference/SD) library wants full path names.
The `F()` macro uses a macro named `PSTR()` that places its string argument in flash memory -- SRAM is precious, and there's no point in keeping constant strings there.

I've even split the full path names into tokens to take advantage of the fact that the C++ compiler is clever enough to
save only a *single* actual copy of each distinct string literal in the executable image, even if it appears several times in code.

...or so I thought. Imagine my surprise when I looked into the compiled ELF file and found this mess:

![ELF hexdump showing string duplication](/assets/2013-10-20-string-merging-pstr/hexdump-duplication.png)

Seriously? The *same* strings repeated over and over and over again? In *my* precious flash memory?
What am I feeding this compiler all those wholesome, tasty cycles for?!

At this point, someone feeling a lesser amount of righteous indignation than I might've just shrugged their shoulders
and started compiling -- possibly by hand -- a global table of string constants or somesuch.

But not I. This compiler was just being *lazy*, and I can't *stand* things or people being lazy at my expense.

Here is how the `PSTR()` macro (which is used by the `F()` macro) is defined:

{% highlight cpp %}
#define PSTR(s) (__extension__({static const char __c[] PROGMEM = (s); &__c[0];}))
{% endhighlight %}

The `PROGMEM` attribute ensures that its subject is placed in flash memory, but it can't be applied to a string literal --
it must be [attached to a variable](http://gcc.gnu.org/onlinedocs/gcc/Variable-Attributes.html).
The `PSTR()` macro works around this limitation by creating a static local variable named `__c`, making it `PROGMEM`,
initializing it with the given string literal, and returning the variable's address.

That's a decent workaround, but it has one annoying limitation:
In a *method* (a member function of a C++ class), a static local -- like that `__c` thing -- has more complicated semantics than just those of a locally namespaced, global variable.
The compiler makes it a *weak symbol*, presumably because the linker may be faced with several instances of it stemming from multiple translation units,
and the linker doesn't seem to like doing string merging on weak symbols even if they point to read-only memory.

That's how I explained it to myself, anyway.
I tested that idea by experimentally including `-fno-weak` in the `avr-g++` command line, which prevents the compiler from emitting weak symbols.
(That switch is documented as ["only for testing"](http://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Dialect-Options.html#C_002b_002b-Dialect-Options) and noted to create "inferior code".)
When I ran the compiler with it, I got loads and loads of curiously apologetic warnings
telling me that the switch was making the semantics all wrong and that I'd "wind up with multiple copies" of all my `__c` variables.
However, it *did* make string merging work, so I guess I was right --
but making a sweeping change to essential C++ semantics didn't seem like such a great approach to solving my string duplication problem.

Well...

...now I'd like to just gloss over one or two days of trying to get the *compiler* to de-duplicate strings.
There were plenty of [red herrings](http://gcc.gnu.org/bugzilla/show_bug.cgi?id=43746#c8) in that direction,
but at some point it began to dawn on me that I was looking at the wrong suspect.
The *compiler* doesn't do string merging. The *linker* does. The compiler just sets everything up. It made a lot of sense once I had realized that.

So... if the compiler isn't helping because of all those semantic subtleties that are far beyond my simple needs, perhaps I should bypass it.
Hmm. Can you actually use assembler *directives* in inline assembler?
All documentation examples and tutorials I've seen just do opcodes and the occasional label.

Let's just give it a try:

{% highlight cpp %}
asm volatile
(
  ".pushsection .progmem.data, \"SM\", @progbits, 1" "\n\t"
  ".string \"MOO MOO MOO\""                          "\n\t"
  ".popsection"                                      "\n\t"
);
{% endhighlight %}

The first line switches to a section called `.progmem.data` that contains C strings (indicated by the `"S"` flag) that should be merged if possible (the `"M"` flag).
The `@progbits` attribute says that it "contains data", which is to say that it should be placed in the executable image,
and the `1` after that declares that each character of those mergeable strings is represented by one byte.
The `.string` directive then adds a null-terminated C string to that section,
and `.popsection` switches back to whatever section we were in prior to `.pushsection`.

It compiles. That's a promising start, but not very useful yet. I have "MOO MOO MOO" somewhere in flash memory now, but I have no idea where.
I need to get the address of that string constant.
Since it's just a proof of concept at this stage, I don't mind if it's ugly if it gets the job done; avert your eyes if you're squeamish.

{% highlight cpp %}
asm volatile
(
  ".pushsection .progmem.data, \"SM\", @progbits, 1" "\n\t"
);
MOO_STRING:
asm volatile
(
  ".string \"MOO MOO MOO\""                          "\n\t"
  ".popsection"                                      "\n\t"
);
PGM_P ptr = reinterpret_cast<PGM_P>(&&MOO_STRING);
{% endhighlight %}

(You can re-open your eyes now.)

I've put a C++ label in the middle of my assembler code and taken its address.
When run, I can see that `ptr` contains a smallish non-zero value now, which is encouraging.
But when I cross-check it against the string offset in the actual flash image, I find that `ptr` is only *half* of what it should be.

Why?

As far as the compiler is concerned, `MOO_STRING` is a code label, and `&&MOO_STRING` is a code pointer; and code is organized and addressed in 16-bit words on an AVR.
My string is at the n-th *byte* for some value of n, so that label points to the n/2-th *code word*, and that is what the compiler is telling me.

Obviously, a bit shift can fix that. And having done that, my test code actually works: `strlen_P()` returns 11 as it should,
and with some minor `F()`-macro-style massaging of `ptr`, even `Serial.print()` cooperates and happily moos straight from flash to the serial console.

I think I'm on to something here!

One last thing is bothering me:
Duplicating a compile-time constant value at *runtime* to get a proper byte offset seems stupid. There must be a way to do this trivial transformation at compile time.

A short bit later I had figured out that the *assembler* would actually happily give me the byte offset of any label I want unless I *specifically* ask for a code pointer.
So, if I had put the `MOO_STRING` label into assembler code, I'd just have to say something like this:

    ldi r25, lo8(MOO_STRING)
    ldi r26, hi8(MOO_STRING)

Let's try to put that into context:

{% highlight cpp %}
PGM_P ptr;
asm volatile
(
  ".pushsection .progmem.data, \"SM\", @progbits, 1" "\n\t"
  "MOO_STRING: .string \"MOO MOO MOO\""              "\n\t"
  ".popsection"                                      "\n\t"
  "ldi %A0, lo8(MOO_STRING)"                         "\n\t"
  "ldi %B0, hi8(MOO_STRING)"                         "\n\t"
  : "=d" (ptr)
);
{% endhighlight %}

Compile, link, disassemble, check, upload, run. **It works!**
And even in duplicate and triplicate, there's only ever one single instance of that "MOO MOO MOO" string to be found in the executable image.

*And* it works even with GCC 4.3.7, the one bundled with the Arduino IDE, with all the standard command line switches passed by the IDE.
That's good news. I'm not a big fan of complicated solutions.

So that is it. All that's left to do is to make it into a macro:

******

> **Update (22 October):**
> The code shown below has been superseded with an [updated version](/arduino/2013/10/22/string-merging-pstr-percent-codes/) that fixes an important edge-case issue.

******

{% highlight cpp %}
#define PSTR(str) \
  (__extension__({ \
    PGM_P ptr;  \
    asm volatile \
    ( \
      ".pushsection .progmem.data, \"SM\", @progbits, 1" "\n\t" \
      "PSTR%=: .string " #str                            "\n\t" \
      ".popsection"                                      "\n\t" \
      "ldi %A0, lo8(PSTR%=)"                             "\n\t" \
      "ldi %B0, hi8(PSTR%=)"                             "\n\t" \
      : "=d" (ptr) \
    ); \
    ptr; \
  }))
{% endhighlight %}

Edit `pgmspace.h` and replace the existing `PSTR()` macro with that, and you're good to go. (Or just put it somewhere near the top of your own source files.)

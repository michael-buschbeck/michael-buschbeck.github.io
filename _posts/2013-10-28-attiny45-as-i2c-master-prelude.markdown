---
layout: post
category: arduino
tags: [arduino, attiny, avr]
title: Prelude to a bus
---

> I want to attach all my Arduino peripherals to a single I&sup2;C bus,
> and it turns out that one of my sensors needs to get its own ATtiny45 brain to make that work.

******

My current Arduino project needs quite a few peripherals:

* a knock detector,
* a whole bunch of electromechanical sensors (switches) and actors (magnets),
* a 10-position binary code switch,
* and an [audio output](https://github.com/michael-buschbeck/arduino/tree/master/Music) shield.

The latter is important here because it occupies a whole **nine** out of Arduino Uno's twenty general-purpose input/output (GPIO) pins,
including four of the six available "analog in" pins.
(And six more digital inputs for its "multifunction button", but that only matters if you intend to actually use it.)
There still remain a few pins for other uses, but certainly not enough for all those components listed above.

In addition, in my project's setup, most peripherals are located an arm's length away from the Arduino.
Even if I could [conjure up more GPIO ports](http://playground.arduino.cc/Code/I2CPortExpander8574) on my Arduino board,
I'd rather avoid stringing dozens of individual signal cables across my entire installation.

So, clearly, what I want is some sort of data bus.

I've read plenty about the disadvantages of using [I&sup2;C](http://en.wikipedia.org/wiki/I2C) for long-distance communications &ndash;
notably that it was never intended, let alone designed, for that kind of use.
All it was designed for is inter-IC communication over traces on a PCB, not meters of dangling cable.

But I don't need anywhere near that much. I also don't require any bandwidth to speak of.

And I&sup2;C seems just *so very convenient*. I can't resist giving it a try.

The [ATmega328](http://www.atmel.com/devices/atmega328.aspx) microprocessor that's on my Arduino Uno board has built-in I&sup2;C hardware support
and there's Arduino's standard [Wire](http://arduino.cc/en/reference/Wire) library that promises to make it easy to use an Arduino as an I&sup2;C master, slave,
or [even both](http://forum.arduino.cc//index.php?topic=13579.msg101244#msg101244)
(which may require a bit of [tweaking](http://www.robotroom.com/Atmel-AVR-TWI-I2C-Multi-Master-Problem.html), perhaps).

Those mechanical sensors and actors and the binary code switch lend themselves to taking a passive role &ndash;
the Arduino just needs to *confirm* that a switch is actuated, and it just wants *query* the state of the code switch at some point.
So I've bought a handful of [I&sup2;C port expanders](http://www.nxp.com/pip/PCF8574P.html) that effectively work like remote-controllable open-drain I/O ports.

That leaves... the knock detector.

The core of the knock detector is a [piezoelectric sensor](http://en.wikipedia.org/wiki/Piezoelectric_sensor) &ndash;
a piece of material that generates voltage (but not much current) when pressed or bent.
To detect a knock, I have to watch for a voltage spike in the piezo's output.
The amplitude of that spike is strongly dependent on the amount of pressure excerted on the piezo &ndash;
anything in the range of a few millivolts (have the sensor lie flat on the table, and knock against the table) and dozens of volts (tap directly on the sensor).
Its duration is somewhere in the range of milliseconds.

With a few additional components (a pair of diodes, an analog comparator, a trimmer),
I can make the piezo's analog output into a tunable digital one (HIGH when above a given adjustable threshold, LOW otherwise).
I could attach that to one of those I&sup2;C port expanders and have the Arduino poll it.
Those voltage spikes are rather short, so I'd have to poll *really* frequently, or I'll probably miss most.

And that, well, that just doesn't seem like a very good idea.
Swamping the I&sup2;C bus with hundreds of identical read requests *per second*, almost all of them returning *nothing* at all,
is just so resoundingly brutish and ugly that I'll leave it to someone who actually has no other choice. At all.

So. What's the plan, then?

The knock detector needs to **locally monitor** the piezo voltage;
**actively push** information about knock events to the Arduino;
and possibly even send **locally-gathered timing** information along,
which also makes it possible for it to **locally buffer** detected knock events just in case it can't get detected events to the Arduino as fast as they occur.

Unless somebody unearthes some ready-made knock detection IC somewhere, this calls for a dedicated microprocessor.
(Finally a reason to get my hands dirty with an AVR [without the training wheels](http://forum.arduino.cc//index.php?topic=111309.msg837103#msg837103).
Yay!)

So, today, I've ordered a handful of [ATtiny45](http://www.atmel.com/devices/attiny45.aspx)s.
My brand-new [AVRISP mkII](http://www.atmel.com/tools/AVRISPMKII.aspx) programmer is already sitting next to me on my desk.

The ATtiny45 is physically small (even if I stick to the DIP form factor &ndash; soldering SMD by hand is still black magic to me),
and it doesn't make me feel like three quarters of the pins are going to waste. It's *juuust* big enough for this task.
In addition, the ATtiny25/45/85 models all have built-in hardware support for I&sup2;C by means of their "Universal Serial Interface" (USI) module.

And also, it's cheap: I've ordered mine for less than an Euro per unit.

Here's the pinout, straight from the [datasheet](http://www.atmel.com/Images/Atmel-2586-AVR-8-bit-Microcontroller-ATtiny25-ATtiny45-ATtiny85_Datasheet.pdf):

![ATtiny25/45/85 PDIP/SOIC/TSSOP pinout](/assets/2013-10-28-attiny45-as-i2c-master-prelude/attiny45-pinout.png)

Huh. That's what you get when you have so very few pins: massive, massive overloading.

Let's brainstorm. What are my options?

1. Use two diodes and an analog comparator (see above) to externally digitize the piezo's output. Feed that into PB1, PB3 or PB4.
   Use SDA and SCL (alias PB0 and PB2) for I&sup2;C communications, taking advantage of the USI hardware, which talks through these pins.
    * **Pros:** Can use USI and the PCINT0 interrupt.
    * **Cons:** Need an external analog comparator and two clamping diodes.

2. Feed the piezo output directly into the *internal* analog comparator behind AIN0 and AIN1 (alias PB0 and PB1), 
   exploiting any AVR's internal [clamping diodes](http://www.atmel.com/images/doc2508.pdf).
    * **Pros:** No external components, and can use the analog comparator's ANA\_COMP interrupt.
    * **Cons:** Need to bit-bang I&sup2;C because AIN0 and SDA are both mapped to PB0, so I can't use USI.

3. Directly attach the piezo to PB1, PB3 or PB4, still exploiting the internal clamping diodes.
   Use the internal ADC to read analog voltages. Implement the threshold in software.
   Use SDA and SCL for I&sup2;C.
    * **Pros:** No external components, and can use USI.
    * **Cons:** No interrupt.

All three cases support the use of an external trimmer to adjust the detection voltage threshold &ndash;
either attached to the analog comparator's *other* input or, in the latter case, to a second ADC pin.

The first two options absolutely *require* a trimmer (or a fixed voltage divider) to supply a reference voltage to the analog comparator, external or internal.

The third one is different in that it implements the threshold in software, so here's a crazy idea:
Instead of using a mechanical trimmer, I could code up an I&sup2;C-triggerable "training mode" where the knock detector just listens for a while,
gathering some sensor statistics (like baseline noise and absolute maximum amplitude), and then uses those to configure itself.

Hmm.

I currently tend towards option&nbsp;3 and that fancy training mode thingy, so I guess I'll try that first.
Let's see how much of that functionality I'll actually be able to squeeze into ATtiny45's 512&nbsp;bytes of RAM and 8&nbsp;KiB of program memory...

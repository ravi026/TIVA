libti2cit
=========

This library was born because the Connected Launchpad i2c hardware state machine is complicated.
The i2c state machine of the MSP430 and AVR lines are so simple it hardly makes sense that it should
be this hard to use i2c on the Connected Launchpad.

Hopefully you can at least use this library as a starting point, since it is small and well documented.

**Table of Contents**
- [Installation](#installation)
- [Writing Your Own i2c Application](#writing-your-own-i2c-application)
- [libti2cit HOWTO](#libti2cit-howto)
- [Understanding i2c](#understanding-i2c)
- [License](#license)

Installation
------------

You need to extract one file from inside the Zip file
[SW-EK-TM4C1294XL-2.1.0.12573.exe](http://www.ti.com/tool/sw-ek-tm4c1294xl) (Yes, it is a
zip file and any unzip program on any modern OS will extract the contents. It *does* spew its
files into the current dir though. Fair warning.)

`project.ld` <-- Find this file.

It is used by `ld` to assemble `example-main.bin` when you type `make`. It is the formal definition
of how to assemble the flash image for a Connected Launchpad, so get it from the zip file available
on TI.com. I found it under `examples/project/project.ld`. Copy `project.ld` into the current
directory where this README lives.

Then run `make`. You should see:

```
$ make
  CC    example-main.c
  CC    startup_gcc.c
  LD    example-main.elf 
$
```

Finally you can run `make lm4flash` or just run `lm4flash` yourself. This will program the
flash image to your device.

Writing Your Own i2c Application
--------------------------------

The easiest way to write an i2c application from this code is to use `example-main.c` as a template
and iterate until it meets your requirements.

Embedded development protip: compile and flash a working build to your device, early and often.

libti2cit is designed to facilitate that.

In `Makefile` you can replace all occurrences of "example-main" with your own app name. Then
rename the `example-main.c` file to that name.


libti2cit HOWTO
---------------

So how do you use libti2cit in your application?

1. Do the [installation](#installation) step to copy project.ld into the source directory.

2. Use the Tivaware DriverLib to initialize the hardware. It really does just fine at that.

  a. Find the base address for the right i2c port. For example, I2C0_BASE, I2C1_BASE, I2C2_BASE, etc.

  b. Reset the i2c hardware, wait for it to complete its reset, configure the pin functions, etc.
     (This is all part of the Tivaware DriverLib initialization.)

3. You probably want to [understand i2c](#understanding-i2c) to make the best decisions for
   your application design.

  a. Master or Slave mode.

  b. Polling or Interrupts.

  c. FIFO Burst or not.

  d. uDMA or not.

4. In Master mode:

  a. The address you send must be left-shifted by 1. The LSB or 1's bit is used to
     signal a read or write, and is not available for addresses. In other words, i2c addresses can
     be viewed as 0-127 (before the left-shift by 1) which become only the even numbers 0-254
     (after the left-shift by 1).

  b. Only call `libti2cit_m_send()` and `libti2cit_m_recv()` (do not call any `libti2cit_s_...` functions)

  c. Signal that you intend to read by first calling `libti2cit_m_send()` with the address LSB (1's bit)
     set to 1. You may also send bytes at the same time. If signalling a read, do not forget to call
     `libti2cit_m_recv()`. The downside is that if you don't follow these rules, the Connected Launchpad
     i2c hardware never flips the expected bits and libti2cit freezes.

  d. Do not blindly ignore an error returned from any libti2cit function. If an error is returned,
     the master must give up and restart from the first `libti2cit_m_send()`.

5. In Slave mode:

  a. Only call `libti2cit_s_send()` and `libti2cit_s_recv()` (do not call any `libti2cit_m_...` functions)

  b.

  c.

  d. Do not blindly ignore an error returned from any libti2cit function.  If a slave receives an
     error, it must give up and restart from the first `libti2cit_s_recv()`.

Understanding i2c
-----------------

[i2cis a brilliantly simple protocol](http://en.wikipedia.org/wiki/I2C) that only uses 2 wires.

The 2 wires are:
- SDA ("Serial Data")
- SCL ("Serial Clock")

Both wires must have one pull-up resistor each. It turns out the Connected Launchpad can even generate an
"internal pullup" if you want. Since that is part of the Tivaware DriverLib initialization step, it is
not described here.

After all your devices are connected to SDA and SCL, you must decide which devices are to be a "master"
and which are to be a "slave." Don't worry: electronic devices don't have feelings and are perfectly happy
to be enslaved on an i2c bus. **Master devices are in charge of initiating everything.** Also, most i2c
devices you might try to hook up to your Connected Launchpad are slaves and cannot be a master anyway.

An i2c slave can never spontaneously send data to its master. This may mean the master has to check
back with the slave constantly, which becomes a lot of work for the master. But it can keep your code
clean because the master controls all the timing.

If you start to have a lot of devices you might want to tell your Connected Launchpad to sometimes
be a slave and sometimes be a master. In that case you must carefully test your collision logic (the
stuff that i2c does when two masters try to initiate something at the same time.) And if you're doing
it that way, you're way past needing me to explain things to you. :)

**The simple case: master mode**

Since almost everyone using libti2cit wants to run in master mode, this explanation assumes the
Connected Launchpad is in master mode. But libti2cit works both as a master and as a slave.

In master mode, you call `libti2cit_m_send()`. The `m` is for master, and `libti2cit_m_send()` is what wakes up
all the slaves and takes command of the bus. This is called an I2C START.

First the master sends I2C START, then it sends an address, then some bytes, and last of all
(if the master wants to receive bytes), a slave starts talking and sends bytes to the
master.

The master can "let go of the i2c bus" by sending an I2C STOP at any time. A slave can only talk when
the master asks to receive bytes, and asks for the address of that slave device.

**Receiving as Master**

Say you are the master. Say you want to read something. You can't just `libti2cit_m_recv()` anywhere.
You actually must `libti2cit_m_send()` first, then `libti2cit_m_recv()`, and probably you must tell the
slave during the `libti2cit_m_send()` how many bytes you will be reading.

`libti2cit_m_send()` takes as a parameter an address. The LSB (with a value of 1) of the address must
be set to 1 to tell the slave to prepare for a read. Carefully study the slave's datasheet and
documentation to find out if you must also supply the number of bytes to the slave.

The `libti2cit_m_recv()` will happily read forever, even bogus data the slave did not send.
The i2c bus does not include a signal **from the slave** saying that `libti2cit_m_recv()` succeeded or
failed. Only the received data itself can be taken as a success or a failure.
(Typically a value of 255 or 0xff indicates failure because that is what the bus looks like
when no one is talking at all.)

**Scanning the Bus as Master**

One interesting thing the i2c Master can do is scan the bus. If the master sends an I2C START,
an address, and immediately an I2C STOP, it receives only one bit from each address. Almost all
i2c Slaves set the bit to say "I am here," so this can be used to scan the bus.

Here is some example code:

```
  init_i2c();

  uint32_t addr;
  for (addr = 0; addr < 128; addr++) {
    if (libti2cit_m_send(base, addr << 1, 0, 0)) continue;
    UARTprintf("found %u\r\n", addr);
  }
```

**Sending and receiving as Slave**

You call different functions when in slave mode: `libti2cit_s_send()` and `libti2cit_s_recv()`. You
may notice they do not need an address. You set the slave address using the Tivaware DriverLib,
and wait for the master to talk to you.

Advanced features like dual slave addresses and clock stretching are also possible, but not
discussed here.

License
-------

The tiva-ussh i2c library is licensed under the GNU Lesser General Public License, either version 3.0,
or any later version.


GNU LGPL v3
-----------

GNU LESSER GENERAL PUBLIC LICENSE
                       Version 3, 29 June 2007

 Copyright (C) 2007 Free Software Foundation, Inc. <http://fsf.org/>
 Everyone is permitted to copy and distribute verbatim copies
 of this license document, but changing it is not allowed.


  This version of the GNU Lesser General Public License incorporates
the terms and conditions of version 3 of the GNU General Public
License, supplemented by the additional permissions listed below.

  0. Additional Definitions.

  As used herein, "this License" refers to version 3 of the GNU Lesser
General Public License, and the "GNU GPL" refers to version 3 of the GNU
General Public License.

  "The Library" refers to a covered work governed by this License,
other than an Application or a Combined Work as defined below.

  An "Application" is any work that makes use of an interface provided
by the Library, but which is not otherwise based on the Library.
Defining a subclass of a class defined by the Library is deemed a mode
of using an interface provided by the Library.

  A "Combined Work" is a work produced by combining or linking an
Application with the Library.  The particular version of the Library
with which the Combined Work was made is also called the "Linked
Version".

  The "Minimal Corresponding Source" for a Combined Work means the
Corresponding Source for the Combined Work, excluding any source code
for portions of the Combined Work that, considered in isolation, are
based on the Application, and not on the Linked Version.

  The "Corresponding Application Code" for a Combined Work means the
object code and/or source code for the Application, including any data
and utility programs needed for reproducing the Combined Work from the
Application, but excluding the System Libraries of the Combined Work.

  1. Exception to Section 3 of the GNU GPL.

  You may convey a covered work under sections 3 and 4 of this License
without being bound by section 3 of the GNU GPL.

  2. Conveying Modified Versions.

  If you modify a copy of the Library, and, in your modifications, a
facility refers to a function or data to be supplied by an Application
that uses the facility (other than as an argument passed when the
facility is invoked), then you may convey a copy of the modified
version:

   a) under this License, provided that you make a good faith effort to
   ensure that, in the event an Application does not supply the
   function or data, the facility still operates, and performs
   whatever part of its purpose remains meaningful, or

   b) under the GNU GPL, with none of the additional permissions of
   this License applicable to that copy.

  3. Object Code Incorporating Material from Library Header Files.

  The object code form of an Application may incorporate material from
a header file that is part of the Library.  You may convey such object
code under terms of your choice, provided that, if the incorporated
material is not limited to numerical parameters, data structure
layouts and accessors, or small macros, inline functions and templates
(ten or fewer lines in length), you do both of the following:

   a) Give prominent notice with each copy of the object code that the
   Library is used in it and that the Library and its use are
   covered by this License.

   b) Accompany the object code with a copy of the GNU GPL and this license
   document.

  4. Combined Works.

  You may convey a Combined Work under terms of your choice that,
taken together, effectively do not restrict modification of the
portions of the Library contained in the Combined Work and reverse
engineering for debugging such modifications, if you also do each of
the following:

   a) Give prominent notice with each copy of the Combined Work that
   the Library is used in it and that the Library and its use are
   covered by this License.

   b) Accompany the Combined Work with a copy of the GNU GPL and this license
   document.

   c) For a Combined Work that displays copyright notices during
   execution, include the copyright notice for the Library among
   these notices, as well as a reference directing the user to the
   copies of the GNU GPL and this license document.

   d) Do one of the following:

       0) Convey the Minimal Corresponding Source under the terms of this
       License, and the Corresponding Application Code in a form
       suitable for, and under terms that permit, the user to
       recombine or relink the Application with a modified version of
       the Linked Version to produce a modified Combined Work, in the
       manner specified by section 6 of the GNU GPL for conveying
       Corresponding Source.

       1) Use a suitable shared library mechanism for linking with the
       Library.  A suitable mechanism is one that (a) uses at run time
       a copy of the Library already present on the user's computer
       system, and (b) will operate properly with a modified version
       of the Library that is interface-compatible with the Linked
       Version.

   e) Provide Installation Information, but only if you would otherwise
   be required to provide such information under section 6 of the
   GNU GPL, and only to the extent that such information is
   necessary to install and execute a modified version of the
   Combined Work produced by recombining or relinking the
   Application with a modified version of the Linked Version. (If
   you use option 4d0, the Installation Information must accompany
   the Minimal Corresponding Source and Corresponding Application
   Code. If you use option 4d1, you must provide the Installation
   Information in the manner specified by section 6 of the GNU GPL
   for conveying Corresponding Source.)

  5. Combined Libraries.

  You may place library facilities that are a work based on the
Library side by side in a single library together with other library
facilities that are not Applications and are not covered by this
License, and convey such a combined library under terms of your
choice, if you do both of the following:

   a) Accompany the combined library with a copy of the same work based
   on the Library, uncombined with any other library facilities,
   conveyed under the terms of this License.

   b) Give prominent notice with the combined library that part of it
   is a work based on the Library, and explaining where to find the
   accompanying uncombined form of the same work.

  6. Revised Versions of the GNU Lesser General Public License.

  The Free Software Foundation may publish revised and/or new versions
of the GNU Lesser General Public License from time to time. Such new
versions will be similar in spirit to the present version, but may
differ in detail to address new problems or concerns.

  Each version is given a distinguishing version number. If the
Library as you received it specifies that a certain numbered version
of the GNU Lesser General Public License "or any later version"
applies to it, you have the option of following the terms and
conditions either of that published version or of any later version
published by the Free Software Foundation. If the Library as you
received it does not specify a version number of the GNU Lesser
General Public License, you may choose any version of the GNU Lesser
General Public License ever published by the Free Software Foundation.

  If the Library as you received it specifies that a proxy can decide
whether future versions of the GNU Lesser General Public License shall
apply, that proxy's public statement of acceptance of any version is
permanent authorization for you to choose that version for the
Library.
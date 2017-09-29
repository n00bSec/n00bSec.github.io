---
layout: post
title:  "Learning Format Strings With Lestrade"
date:   2017-9-28 19:20:00 -0400
categories: update post exploitation reverse-engineering stringformat 
---


<h1>Introduction</h1>

When I managed to get started on the ASIS 2017 Finals CTF, I was most definitely not in the right state of mind to do any challenge solving. On top of that, this particular bug class I never really took that seriously ever since <a href="http://phrack.org/issues/67/9.html">I read some time ago</a> that it was disappearing in the wild, and had been mitigated to being an information leak. Who would've known that they were still a common CTF challenge? 

Anyways, without that practice, I started on Greg Lestrade, but hadn't finished it before I decided I needed to get to sleep. I instead solved it a while after, with a different methodology for completing the exploit.

<h1>Preparation - Tools</h1>
Firstly, the setup I used:
<ul>
<li> <a href="https://github.com/radare/radare2">radare2</a> - For disassembly and patching the local binary.</li>
<li> Python and <a href="http://pwntools.readthedocs.io/en/stable/">pwntools</a> - For scripting the exploit.</li>
<li> <a href="https://github.com/hugsy/gef">GDB w/GEF</a> - For debugging.</li>
</ul>

These are probably replacable by whatever your favorite disassembler, scripting language, and debugger are. I just like what's open source and easy to run inside of tmux, with an Ubuntu 16.04 machine.

<h1>Preparation - Material</h1>
I've written this with an assumed basic knowledge of the x86 instruction set from readers. If you need to get acquainted, you can take a disassembler to compiled C code, or if you're a normal person you can ask around for a book.

There's no way I could cover format string exploitation as well as other resources already have. I'll do a little bit of overview here, but I'd rather point out some primers on it. Any of them will do. Pick one and follow along until you're either bored or ready to jump into it.
<ul>
<li>For book reading, you can pick up Hacking: The Art of Exploitation.</li>
<li><a href="https://www.youtube.com/watch?v=0WvrSfcdq1I">LiveOverflow's videos</a> on string format exploitation are great.</li>
<li><a href="https://www.youtube.com/watch?v=xAdjDEwENCQ">Gynvael Coldwind has a stream</a> on the topic, with CTFs in mind.</li>
<li><a href="http://phrack.org/issues/59/7.html">Phrack</a>, <a href="http://phrack.org/issues/67/9.html">Phrack</a>, <a href="http://phrack.org/issues/67/13.html">Phrack</a></li>
</ul>

The Phrack articles may not be great primers for strict beginners, but they're worth reading still.

<h1>Reversing Greg Lestrade</h1>

You can get my patched binary and final exploit <a href="https://github.com/n00bSec/ASIS-Finals2017/tree/master/lestrade">here</a>.

First things first, let pwntools/checksec give the low down on what protections the binary is running. pwntools' ELF object gives the proper output when constructed, like.

{% highlight Python %}
>>> from pwn import *
>>> e = ELF('./greg_lestrade')
[*] '~/Programming/ASIS-Finals2017/lestrade/greg_lestrade'
	Arch:     amd64-64-little
	RELRO:    Partial RELRO
	Stack:    Canary found
	NX:       NX enabled
	PIE:      No PIE
{% endhighlight %}

Then open it up in radare2.

<img title="r2 - Login graph" src="/pics/LestradeLogin.png">

The main function starts out with a call to a function that removes input buffering and sets a time limit with a SIGALARM. Patching out the signal for local debugging is just a matter of entering *wx 9090909090@0x4008d1* in r2. This is followed by a call to a function that loads from static memory the credential to check against the user's input, *7h15_15_v3ry_53cr37_1_7h1nk*.

Following that, the user is presented a menu in which they can either exit, or enter some command. The first bug of interest is here, within the admin command menu.

<img src="/pics/LestradeReadStrlenBug.png">

In the check for the command entered (passed in through read()), we can see that only the lowest byte of strlen()+1's result is stored on the stack. This byte serves as the upper limit of a following loop to ensure that the characters in the buffer are lowercase ASCII, which provides two weaknesses in that none of the 0x3ff characters beyond the first 256 are checked for, and no checking will happen at all when the number of characters read in is a multiple of 0x100 minus 1.

Bypassing this, we're rewarded with another bug.

<img src="/pics/LestradePrintfOnRawBuffer.png">

The buffer we read into has its address loaded into the rax register, which is mov'd into rdi, which is then passed directly into printf. This allows us to pass in delimeter sequences and perform a string format exploit attack.

<h1>String Format Attack</h1>
The handiest pair of things you could achieve when doing binary exploitation is an arbitrary read and write, and format string attacks allow you to do just that. What format strings are are C strings passed into the printf family of functions (see man 3 printf) with a set of sequences that interact with potential additional arguments. As an example, you could expect normal usage to look like...

{% highlight C %}
int x = 1;
printf("x = %d\n", x); //prints "x = 1"
char* buffer = "dog";
printf("I want %d %s(s) for Christmas.\n", x, buffer); //prints "I want 1 dog(s) for Christmas."
printf("%2$d, %3$8d, %1$4d\n", 1,2,3); //should print "2,        3,    1"
{% endhighlight %}

The format of a delimiter is typically %(#1)$(#2)([hh]character), where #1 specifies which argument it is you're targeting, #2 specifies how many characters should be printed in formatting that argument, and the character specifies how the argument should be treated. *h* or *hh* are size specifiers, that will be explained further shortly.

The printf family can have any number of parameters by taking the first three in through registers, and the rest will be pushed onto the stack. Know what else is also on our stack? The buffer we're performing the attack with. Thus, we can use high positional arguments to specify our own pieces of string data as an argument. As an experiment, I suggest you try out the following C code:

{% highlight C %}
#include <stdio.h>

int main(void){
printf("%x%x%x%x%x%x%x%x%x%x%x%x%x\n");
}
{% endhighlight %}

After enough "%x"s, catch any patterns? Do you see where your data is coming from?

This is especially handy with the %n delimeter, which treats the argument it targets as an int pointer, and writes into the memory pointed to the number of characters printed from the format string so far. 

%n typically writes 4 bytes, but the h specifier allows one to shorten the perceived data storage size by half. %hn thusly writes only two bytes, and %hhn writes 1. We can combine these with data in our stack buffer and other delimeters to print out characters until printf's internal counter is ready to perform writes to any arbitrary memory we choose.

<h1>The Exploit</h1>
Here is where I stopped on the night of the CTF. I was spending quite a bit of time working out just how one could completely automate the generation of format strings in a way that they'd have a repeatable challenge solver. That turns out to be a nightmarishly finicky task, as paddings, alignments, and positional arguments bring changes to the pointers you place in such a way that I couldn't avoid eventually causing an off-by-one error within an hour and a half of writing. This took time and energy I could've put to better use in writing the format string by hand.

Writing by hand was much simpler anyhow, and pwntools helped me out a little in getting started. Pwntools' automated format string exploitation module can easily get the starting offset of a buffer (8 in my case), and from there I calculated that I should pad 256/8 double dwords worth of 'b's as well as the first delimeter and move the first positional paremter accordingly. The decimal equivalend of 0x876-256 I had placed in my %#c delimeter to prepend my write, and my %n delimeter's position was placed onto an arbitrary pointer.

What's a good place to write to? Looking back at the protections pwntools informed about, RELRO was only partial on this binary. This meant that the Global Offset Table was writable, and any function listed within was hijackable. It just so happens that puts() was the first function to be called in it after printf() gets finished, and puts() was indeed in the GOT...

{% highlight Python %}
>>> for x in e.got:
...   print str(x) + ":" + hex(e.got[x])
...
strncmp:0x602018
puts:0x602020
read:0x602050
alarm:0x602048
system:0x602038
__stack_chk_fail:0x602030
__libc_start_main:0x602058
__isoc99_scanf:0x602068
printf:0x602040
strlen:0x602028
setvbuf:0x602060
{% endhighlight %}

...so I made puts my target.

What should we write into the GOT with the rest of the protections preventing us from executing shellcode, and with Address Space Layout Randomization? Well there's a __win__ function that was placed in the binary to save a lot of work, and just read out the challenge's flag when called.

<img src="/pics/LestradeWinFunction.png">

I decided to just go with this, with a backup plan of leaking another GOT pointer, and bruteforcing the offset to a <a href="https://github.com/david942j/one_gadget">magic gadget</a> within system for a shell, or finding a stack pivot big enough to do ROP.

What followed after was just a lot of finagling and aligning of my buffer's data, as well as fixing put the positional parameters after I successfully tested each write in my debugger, and the finished format string is thus the first real format string exploit I've done that wasn't just an information leak. 

{% highlight Python %}
fmt_payload = ('b'*256) + '%1910c%47$hn' + '%202c%48$hhn' + '%192c%49$hhn' + '%50$hhn' + '%51$hhn ' + ' '*5 + p64(e.got['puts']) + p64(e.got['puts']+2) + p64(e.got['puts']+3) + p64(e.got['puts']+4) + p64(e.got['puts']+5)
{% endhighlight %}

Having all that done, I suggest in repeating that after the first short write (%hn - 2 bytes), use short-short-writes (%hhn - 1 byte) so that you need only print something less than 0x100 versus 0x10000 to specify the data you want to write, even if you'd then have to go at it a single byte at a time.

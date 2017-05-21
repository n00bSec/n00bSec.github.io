---
layout: post
title: "Be Careful With Your Temporary Email"
date: 2017-05-21  21:00:00 -0400
categories: xss post update email
---

<p>Before finals came upon us, I managed to find several XSS vulnerabilities in a variety of sites. I've reported all of them (most through <a href="https://www.openbugbounty.org">OpenBugBounty</a>) and now feel like I have something short and interesting to share.</p>

<h1>Demonstration</h1>
<video height="400px" width="600px" src="/pics/xss-mailbox.mp4" controls></video>

<h1>Explanation</h1>
<p>For years now I've been making regular use of temporary email services to keep spam out of my personal inbox. A lot of websites hold services that I typically need in a pinch but wall of anyone without a new valid email to register with. Of course you can expect whatever email you give to be filled up with newsletters and get abused after being sold off to advertisers. </p>
<p>During some bug hunting, I was very curious about the potential for a stored XSS in a website that allowed the creation of user profiles. <a href="http://223119.xssposed.net/mirror/">My hunch was correct in the end,</a> but to create profiles to demonstrate I used a temporary email from 10minutemail.net. And maybe you can imagine my surprise when I found that the potentially malicious Javascript I placed within the registration form was relayed perfectly unfiltered and intact back to the browser for execution.</p>
<p>So I sent an email then to the website administrator along with a proof of concept, and moved on with my night.
Or at least I did until I noticed someone in IRC talking about finding XSS in a temporary email provider he was looking to use that same night.
We had a great chat in private, talking about looking towards the possibility of working together in the future, and sharing tidbits on the services we were poking holes in. 
I wish I could fully remember his handle, but from our chat I realized that 10minutemail.net wasn't a unique service.</p>
<p>Most of these services switch between somewhat unique domains that could be recognized without much effort, so it's not completely unreasonable to expect . If an unscrupulous service were to want to attack a user, it'd be as simple as slipping in some Javascript into the HTML emails the user would expect to receive. </p>
<h1>Testing</h1>
So, I've so far yet recognized this same vulnerability within:
<ul>
<li><h2>10minutemail.net</h2>
<br> <i>w/PoC above...</i> </li>
<li><h2>10minutemail.com</h2> <br> 
<img src="/pics/10minutemail-com-xss.png"> <br> </li>
<li><h2>tempmail.net</h2>
<br> (Notified of this one by a separate bug hunter)<br> <img src="/pics/tempmail-net.png"> 
<br> <i> and it comes with options...</i>
<br> <img src="/pics/tempmail-net-again.png"><br> </li>
<li><h2>20minutemail.com</h2> 
<br> <img src="/pics/20minutemail-com.png"> <br> </li>
<li><h2>my10minutemail.com</h2> <br> <img src="/pics/my10minutemail-com.png"> </li>
</ul>
What I haven't managed to trigger XSS in within five minutes of discovery include:
<ul>
	<li>dropmail.me</li>
	<li>mailinator.com</li>
</ul>

HideMyAss's service seemed secure enough to be worth mentioning, but sadly it won't be continuing for much longer. 

<h1>Takeaway</h1>
Just be REAL careful with temporary email addresses. Not all of them are insecure, but plenty are horribly so, 
and I've reason to believe plenty of the maintainers in charge of them don't usually care enough to fix anything.
Writing a script to test for this in temporary inboxes shouldn't be too difficult, if you feel the best way to be sure
about your safety in using one is to test it yourself. I'll probably update this post if I get to writing that script myself.
 

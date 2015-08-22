---
layout: post
title: "Deanonymization made simple"
date: 2015-08-22 21:44:25 +0200
comments: true
categories: 
---

<a href="https://twitter.com/c0rdis/status/630705659848302592">cbcf9dde327c475d99627c87f58cab7ac6689164bf2fe7734c10c78005ed118e</a> == sha256("[10.08.2015] I've discovered that about 2% of the known darkweb is controlled by one organization.")

{% img center /images/5_dark_web.jpg 600 'image' 'images' %}

<!--more-->

Reading articles of deanonymization of hidden services by <a href="http://40.media.tumblr.com/cd025e4b9b6db50cf53d21a7af5e5568/tumblr_nq7y4s0aMQ1uygu1vo1_1280.png">controlling certain nodes</a> 
or conducting <a href="http://conference.hitb.org/hitbsecconf2015ams/wp-content/uploads/2015/02/D2T2-Filippo-Valsorda-and-George-Tankersly-Non-Hidden-Hidden-Services-Considered-Harmful.pdf">correlation attacks</a>, 
I came to an idea that in certain cases it might be much easier to break anonymity. Just by having the same vulnerabilities as in "clearnet", applications can expose sensitive information
and let an attacker gather data from the system and deanonymize the target, with certain "darknet" specifics in the approach.

According to the results of the recent HyperionGray research of <a href="http://alex.hyperiongray.com/posts/289994-scanning-the-dark-web">scanning the darkweb with PunkSPIDER</a>,
approximate number of alive dark services is about 7000. The guys took alive and not-so-hidden services and started to scan those for serious vulnerabilities.
I've started my own research with slightly different approach - in opposite to searching for critical vulnerabilities like OSCI/SQLi, 
I've taken a closer look to conventionally low-risk information disclosure. 

For that I've written a simple Python script which, when provided with server/framework, would enumerate accessible files and folders and probably discover certain leaks of server information.
To my surprise, fair amount of them actually had quite lame generic server authorization/configuration issues up to world-readable */phpinfo.php*.

{% img center /images/5_phpinfo_hacking.png 900 'image' 'images' %}

The most helpful and common fail pattern was, however, the default Apache pages such as */server-info* and */server-status*. Whereas the first
one would give you a nice picture of the server information with current settings, modules and its configuration (and IP address, of course), 
the second is more valuable in terms of current connections. **In a given set of 7k+ alive services almost 500 of them (about 7%) appeared to be 
vulnerable.** Further analysis showed that large-traffic applications are affected, too.

<table>
<tr>
  <td>{% img left /images/5_status_traffic.jpg 350 'image' 'images' %}</td>
  <td>{% img right /images/5_status_traffic_2.png 420 'image' 'images' %}</td>
</tr>
</table>   

For one of the websites I've noticed, that it has several other hosts with completely different kinds
of subjects. The only thing which was the same, were those */server-status* pages all among them. Quick gather of references on those revealed more than 300
unique services with traffic as much as 50+ Gb per day. Interestingly enough, most of them were referenced from HiddenWiki page,
which also resided on the same server. A weaver! As appeared later, it was a hidden hosting service, where anybody could pay certain amount of BTC and rent
it for his own dark intentions. Obviously, such disclosure makes it possible for deanonymizer to list all the queries to a particular domain on the hosting 
server and view parameters with corresponding values for GET requests with full paths to closed parts of the application.

I was lucky again when my script warned me of an external IP address, which accessed *"vps.server.com"*.
If you've ever had a look to access.log of your web server, for sure you've noticed a lot of connections of all kinds of bots which scan the Internet for 
vulnerabilities. That was probably the first time in my life, when I was really thankful to them. It meant the following:

- clearnet service is also available on port 80
- if I manage to access it, my watcher script can isolate it

One of the options to hit that is to basically try to scan the whole Internet on port 80. Sounds crazy? Hold on, check these projects first: 
<a href="https://zmap.io/">Zmap</a> and <a href="https://github.com/robertdavidgraham/masscan">Massscan</a>!

What's basically needed, is to access a specific IP address with certain marker, which would identify this IP address uniquely, and monitor 
such access on */server-status* of a target server. I assumed that probably the easiest way to do it is to use the following vector: http://xx.xx.xx.xx/xx.xx.xx.xx.
Results haven't made me wait too long:

{% img center /images/5_ip_access.png 800 'image' 'images' %}

Of course, this is not the only way to achieve that. The following scenario is even simpler: many clearnet hosts on the same server are used to redirect traffic to darknet, and
this also helps a lot to deanonymize the target. One approach is quite similar to the previous one but more universal in a way that you don't really need to have control over status page. 
It is enough to parse those responses, which return *30x* code, and check for presense of ".onion" string in the "Location:" header:

{% img center /images/5_location.png 800 'image' 'images' %}

For the laziest of researchers, Shodan might help, too:

{% img center /images/5_shodan.png 800 'image' 'images' %}

Finally, researcher can always find a vulnerability in one weak service, and get access to the whole hosting server. Let's say, I believe it's possible ;)

{% img center /images/5_shell.jpg 800 'image' 'images' %}

## Conclusion

The goal of my research was to show that often deanonymization of a hidden service (or even a network) can be done trivially by applying the same pentest approach as in clearnet.
Main difference here is that usually non-critical information disclosure plays much more significant role than for "normal" web applications.
To summarize, at least the following easy ways may let researcher deanonymize a darknet service:

- instant win (server-info, phpinfo, ...)
- status page access (x.x.x.x/x.x.x.x)
- (un)expected redirect (30x clearnet to darknet)
- app-level pwnage (missing patches, vulnerabilities in the code, default framework pages...)


P.S. If you're interested in the topic, you may also want to check <a href="https://www.thecthulhu.com/setting-up-a-hidden-service-with-nginx/">TheCtulhu's blog</a> and find decent instructions on configuring nginx server to host a hidden service in a more secure way.
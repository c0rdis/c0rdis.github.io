---
layout: post
title: "&quot;Cryptosocial network&quot; from the inside"
date: 2015-01-17 23:45:57 +0300
comments: true
categories: crypto social
---
*Disclaimer: all vulnerabilities described here were reported to developers and published with their consent*

"Get a public key, safely, starting just with someone's social media username(s)." - this is what you likely to see if you visit the main page of an ambitious project named <a href="https://keybase.io">Keybase</a>. 
A great idea to (finally) bring public-key cryptography en masse and make its use easy and fun. 
The project is in fact a public key directory wrapped by well-worked model of social networking and tightly bound to those networks itself.

{% img center /images/0_header_maria.jpg 600 'image' 'images' %}

How does it work? From the high level, a user registers, uploads a key to the server (or creates a key pair right on the website - this is what will likely to be a common scenario) and then verifies his or her identity via popular social networks and personal websites by placing signed proofs there. 
Then, when by some reasons the key is not valid anymore, the user has to upload a new one and get verified again. In order to make such mechanism more efficient, creators implemented "tracking" - kind of following a user and receiving all the updates regarding his activity.

###From cryptographical point, authors realized an interesting concept of "TripleSec" - conjunction of three-in-one sound crypto algorithms to make the protected data storage even more secure.
"TripleSec is a simple, triple-paranoid, symmetric encryption library for Python, Node.js, Go, C#, and the browser. It encrypts data with Salsa 20, AES, and Twofish, so that a compromise of one or two of the ciphers will not expose the secret. Of course, encryption is only part of the story. TripleSec also: derives keys with scrypt to defend against password-cracking and rainbow tables; authenticates with HMAC to protect against adaptive chosen-ciphertext attacks; and in the JavaScript version supplements the native entropy sources for fear they are weak." (<a href="https://keybase.io/triplesec/">link</a>)

###Currently the project is in its beta - only invited people are allowed to join and test it.
So that was double interesting for me to be one of them... Well, and grab a nice nickname ;)
Surprisingly, check of tweets marked "keybase" showed quite a number of people ready to share the invite - in less than 10 mins I got one and started crawling around.

I'm not going to publish all the issues which currently exist there - some of them (seem) non-critical now, some XSS (seem) non-exploitable, and let's be honest to ourselves,
 
 "*Lookup failed in query SELECT hash,val,ctime,type FROM merkle_blocks WHERE hash LIKE ? w/ [\"bad%\"]*"  is not enough why would anyone ever write a blog post about it :)

After a short overview, I spotted two major design problems which looked quite serious to me.

### 1) User password as a single point of failure

There were enough articles on the Internet criticizing upload of private key to the server - I'm not going to repeat it here. But what's really worrying here is the fact how it is presented to the end users: 

 - Most likely, "normal" people will not be using Terminal and will work in the browser - that's pretty much fine here and that's what creators were counting for.
 - Most likely, "normal" people will generate the key in their browser - in order to do so, they will have to confirm their "passphrase", which for instance is their password. Sure, no problem.
 - Most likely, "normal" people will follow instructions and save the encrypted private key on the server. (Presumably) still ok.

{% img center /images/0_keybase_private.png 500 350 'image' 'images' %}

The user clicks "Done" and he's finished and ready to tweet everybody how cool it is to dive in the world of cryptography.
Did you actually spot the weak point? *The user was never asked to set the passphrase, which by default is the same as his password!*
I mean, yeah, having several layers of properly implemented encryption might help to withstand attacks on one of algorithms for quite some time. 
But in fact, *all* of those assumptions are made on the fact the user password is not leaked. If that password by any means is obtained by an attacker - the game is pretty much over. 
An attacker will not only have access to the social network account (which is bad but criticality here is not that huge) but also to the stored secret key which could have been used in quite a lot of other applications.
No "AnyNumSec" algorithms would keep the user safe from a single stolen account password (DNS? Phishing? Social engineering?) given the key is the same, which is the case. My suggestion here is at least to segregate account password and private key passphrase and/or put additional controls in place.

### 2) Golden "backdoorish" session 

Another major concern here is about "golden key". The structure of almost all the emails sent from the server - i.e. reminders to verify social network account, changed e-mail, password reminders, post-registration and other services, contain a small footer with links - "Change Mail Settings" and "1-Click Unsubscribe".

{% img center /images/0_keybase_change_settings.png 600 'image' 'images' %}

Both of them look similar and have parameter "a" included: 

*https://keybase.io/_/user/account?a=lgGS[base64-encoded id, email and several hashes]*.

This parameter is responsible for a permanent user session - it is a replacement for the session token itself, and it doesn't expire unless the e-mail is not changed.
Just to clarify: persistent session (= persistent access to user's account) is transferred in plaintext to the provided e-mail and stored in plaintext on the server. 
Consider forwarding any of those service e-mail by unaware user, unauthorized access to the mailbox or simple typo...
Ok, what one could do with it?
Well, it is possible to login under another account but victim still controls it... Or not? 

###It appeared that due to insufficient authorization (client-side checks only) it is possible to fully take control over victim's account following the steps:

0) Copy victim's encrypted private key. This step is not mandatory to achieve the goal but still good-to-have one ;)

1) Once logged-in, an attacker is able to remove private/public keys without any confirmation. At this moment all the public proofs are gone.

2) It is also possible to generate a new key pair with chosen passphrase - this is where auth bypass matters. 
Normally, to be able to generate a new pair, one should know current password. 
An attacker can simply spoof the response and make it look valid - and that would be enough for him to proceed. 
Two responses (valid/invalid) for the same functionality of checking password are presented on the screenshot below.

{% img center /images/0_keybase_possible_responses.png 800 350 'image' 'images' %}


3) From this point, an attacker can exploit the fact passphrase and password are used interchargeably in the application - knowing correct passphrase (which an attacker has just set - see p.2), he is able to change *both* the password and the passphrase to any string.

4) Finally, he can also change the e-mail now, making the "golden session" futile - and so the chances for a good guy to take his account back.

In such scenario an attacker doesn't break confidentiality and doesn't reveal the secret key (probably only saved an encrypted copy for good). 
However, from the point of second part of "cryptosocialness", losing an account in a social network is still not the most pleasant thing, especially if you already got it set up and working for some time.



###As a conclusion: 
there are certain design defects, however, it doesn't change the fact of how brilliant the idea itself is. 
Besides, it is beta now, so this is exactly the time when good guys could make possibly-next-big-thing safer.
If you want to get an invite, I still have a couple, so drop me a line... And if you're already registered, feel free to <a href="https://keybase.io/my">track me</a> ;)

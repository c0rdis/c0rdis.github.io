---
layout: post
title: "Lifting the veil, or dark does NOT always mean secure"
date: 2015-10-09 17:02:10 +0200
comments: true
categories: 
---
This post can be treated as a continuation of previously published article of "Deanonymization made simple". As mentioned, more than five hundred 
of publicly gathered hidden services were misconfigured to disclose */server-status* page. I've analyzed all of them, and the results looked quite
interesting to me to publish those.

*I would like to thank my friends <a href="https://twitter.com/josephfcox">@josephfcox</a> and <a href="https://twitter.com/flexlibris">@flexlibris</a> for providing me with invites 
to Riseup and making this article possible.*

<!--more-->

In my previous post the main target of such attacks were hidden services themselves, or servers. It sounds obvious that clients are under attack and
can potentially be disclosed, too. However, this is not exactly the case. Whereas all GET-requests are stored and available to any unauthenticated
attacker, it is often hard to find a correlation between the user and his actions. Accessing a service under Tor will log a record with 127.0.0.1 as
visitor's IP address - and even if there were external IP-addresses, they would be just exit nodes.
For example, on the following screenshot one can see current requests to a variant of "Dark Google": 

{% img center /images/6_dark_google.png 900 'image' 'images' %}

This information can be rather used for statistics than for actual de-anonymization. I don't feel to 
publish it here but my observations show that probably there is a reason why such Tor services are called "dark". 

The following scenario is more of help though, since knowing the "key" parameter value serves as auth method and 
lets anybody knowing it see the confirmation of the order - along with the cost, BTC addresses of both parties, 
purchased product and customer details. Data from log:

{% img center /images/6_log_client.png 900 'image' 'images' %}

Actual order along with client data:

{% img center /images/6_client_info.png 900 'image' 'images' %}

Although "pure" hidden service can rarely be used for successful DA with this method (probably if not taking in consideration stupid authorization mistakes and
poor authentication schemes), it becomes trivial in case of ordinary non-Tor services are in place. Moreover, in the next examples presence of a hidden 
service actually decreases overall security!
 

### Riseup
*Your riseup.net email account is a wonderful thing. Although we don't provide as much storage quota as surveillance-funded corporate email providers, riseup.net
email has many unusual features: <...> we do not log internet addresses of anyone using riseup.net services, including email.* *

<i><p align="right">** welcome email for newcomers*</p></i>

Riseup has three types of accounts sorted by security level: green (lists, wiki), red (email, shell, OpenVPN) and black (Bitmask enhanced security).
In this section I will concentrate on red and black accounts, since green ones do not seem to have that much importance in terms of privacy. 


Brief information:
incorrect configuration of nearly all of HTTP(S) services (listed
here: https://help.riseup.net/security/network-security/tor/hs-addresses-signed.txt)
allows an attacker to disclose IP addresses of currently active users,
their contacts from address book, login name at the moment, and in case of black.* service - to
find a correlation between unique hashed ID of the user and his real
login, to further spy on his activities. According to the earliest
references I was able to find, this vulnerability was in place at
least from 2012, which presents critical privacy risk to all Riseup
users. Since it is quite trivial to set up monitoring of all the
requests made to the systems, probably many of the users were already
successfully de-anonymized during those 3+ years.



#### Vulnerable services in "darknet":

* http://nzh3fv6jc6jskki3.onion/server-status - help.\*, lyre.*, riseup.net
* http://cwoiopiifrlzcuos.onion/server-status - black.\*, api.black.\*
* http://zsolxunfmbfuq7wf.onion/server-status - cotinga.\*, mail.\*
* http://yfm6sdhnfbulplsw.onion/server-status - labs.\*, bugs.otr.im
* http://xpgylzydxykgdqyg.onion/server-status - lists.\*, whimbrel.\*
* http://j6uhdvbhz74oefxf.onion/server-status - user.\*
* http://7lvd7fa5yfbdqaii.onion/server-status - we.\*


#### Sample data, which can be disclosed:
*RED* : remote IP address of the current user, his actions and address book contacts

{% img center /images/6_real_ip_address_book.png 900 'image' 'images' %}

*RED* : currently logged in user, and his actions

{% img center /images/6_current_user_login.png 900 'image' 'images' %}

*BLACK* : correlation between real user login and his unique hash ID, which
is used later to anonymize all the activities he makes

{% img center /images/6_login_hash_id.png 900 'image' 'images' %}



### Megafon
As you might deduce, Megafon, one of the largest Russian mobile operators, is affected too. In this case,
it was set of old subscription services along with WAP.

{% img center /images/6_megafon_general_info.png 900 'image' 'images' %}

Brief information:
it is possible for non-authenticated user to disclose list of all current connections (active users w/mobile phone numbers),
internal pages of vulnerable services, information about current transactions and admin credentials for the services.

#### Vulnerable hosts on the same server:

* wap.megafonpro.ru
* podpiskipro.ru
* iclickpro.ru


#### Sample data, which can be disclosed:
General user activity with phone numbers:

{% img center /images/6_megafon_user_activity.png 900 'image' 'images' %}

Admin credentials to vulnerable services:

{% img center /images/6_megafon_admin_passwords.png 900 'image' 'images' %}

*Disclaimer: admin credentials were not used by me to break into the system, however, log analysis has shown
that further attack on other Megafon systems is very likely from there.*

### Why is it happening?
The answer looks simple - because of default Apache configuration. For the sake of an experiment I've setup a freshly installed
Apache server to host both normal and hidden services. Here are the results for normal service:

{% img center /images/6_normal_service.png 900 'image' 'images' %}

And here are those for hidden one:

{% img center /images/6_hidden_service.png 900 'image' 'images' %}

The reason is that for Apache any connection from Tor browser is considered to be initiated from localhost (we've seen that on all 
previous faulty screenshots, remember?). Default configuration of *status.conf* is as follows:

```
<Location /server-status>
    SetHandler server-status
    Order deny,allow
    Deny from all
    Allow from 127.0.0.1 ::1
#    Allow from 192.0.2.0/24
</Location>
```

In other words, since connection seems to come from a trustworthy source, it simply allows anyone to access it!
Moreover, any user navigating on Tor seems to be more trusted than in non-Tor conditions, as both server and many applications just trust 127.0.0.1 more as local.
<a href="http://blog.ircmaxell.com/2012/11/anatomy-of-attack-how-i-hacked.html">How I hacked StackOverflow</a>, indeed.

So, for this specific case it might be enough to disable */server-status* or comment the line "Allow from 127.0.0.1 ::1". However, the problem is deeper - due 
to the architecture of Tor solution, **all the applications in darknet** have to be reviewed to make sure there is no excessive trust to "local attacker", and 
it sounds like the fight "Security vs. Privacy" continues.

{% img center /images/6_tor_project.png 900 'image' 'images' %}
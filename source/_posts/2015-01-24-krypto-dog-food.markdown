---
layout: post
title: "Easy way to get KDF (Krypto-Dog Food)"
date: 2015-01-26 22:57:42 +0300
comments: true
categories: crypto KDF
---
My recent <a href="/blog/2015/01/17/cryptosocial-network-from-the-inside/">Keybase overview</a> gave me an impulse to read more about KDFs, their implementations and modern applications, which I'm going to present in the following post.

{% img center /images/2_krypto_dog.jpg 400 'image' 'images' %}

KDF is a Key Derivation Function. As follows from the definition, such function is used to derive one or more keys from some secret value - *source of initial keying material*.
Derived keys can then be used in different ways, such as to encrypt other important data, to built a MAC, or even as-is. 
One example of using KDF is to generate a session key during TLS handshake. 

Main functionality built in KDFs can be described as X-X: *extract-and-expand* paradigm.
*Extraction* module takes non-uniformly random or pseudorandom source keying material as input and, by applying some function, "extracts" uniform key to operate with as primary input.
This step can be omitted in case of initial keying material is already uniform, but it's often not the case.
*Expansion* module, in turn, operates with previously generated (pseudo)random key and uses it to seed some function (not just any - see <a href="https://crypto.stanford.edu/~dabo/cs255/lectures/PRP-PRF.pdf">PRPs and PRFs</a>) to produce additional keys - those we expect to be derived.

Based on provided initial keying material, KDFs are divided into two large groups:

###KDFs based on some source key
These KDFs take source key as input, which is assumed to have enough entropy.

"Traditional" KDF scheme operates with perfect source keys (those which do not need extraction step). 
Its additional input parameters are CTX (context string, depends on current application) and CTR (counter), and the scheme is based on simple concatenation of pseudorandom function (secure PRF) output. 
With such a function one could generate as many bits/keys as needed, and just cut off the rest when the goal is achieved.
It can generally be described with the following pseudocode: 

{% codeblock lang:python %}
KDF(SK,CTX,CTR) = K(0) || K(1) || ... || K(CTR)
    where
    K(0)   = PRF(SK,(CTX||0)) 
    K(1)   = PRF(SK,(CTX||1)) 
    ...
    K(CTR) = PRF(SK,(CTX||CTR))
{% endcodeblock %}

The best known example of "non-perfect-key-input" KDFs is HKDF, or <a href="https://eprint.iacr.org/2010/264.pdf">HMAC-based Key Derivation Function.</a>
Steps to derive the keys include extraction (full X-X scheme) and use of HMAC as secure PRF in traditional KDF scheme.
Additionally, previously derived key is used as input to generate the succeeding one:

{% codeblock lang:python %}
Extraction : k = HMAC(salt, SK)
Expansion  : HKDF(k,CTX,CTR) = K(0) || K(1) || ... || K(CTR)
                where
                K(0)   = HMAC(k, (CTX||0))
                K(1)   = HMAC(k, (K(0) || CTX || 1))
                ...
                K(CTR) = HMAC(k, (K(CTR-1) || CTX || CTR))
{% endcodeblock %}

KDFs of this type are commonly used for key diversification and are not acceptable for password storage.


###Password-Based KDF
Another group of key derivation functions is based on user-supplied password as input.
Since passwords do not have sufficient entropy, traditional KDFs as well as HKDF should not be used in this case, as derived keys will be vulnerable to dictionary attacks.
To deal with a problem and compensate input weakness, two main PBKDF defenses were developed: use of salt and *slow hash functions*.

Traditional approach to slow down the calculations here is based on increased number of iterations - so fast hash function runs over and over again until acceptable latency is reached.
PCKS#5 describes PBKDF as follows:

{% codeblock lang:python %}
PBKDF1 = H{c}(password||salt) = H(H(H(H(H...(H(password || salt))...))))


PBKDF2 = T(0) || T(1) || ... || T(L)
    where
    L = len(desired_key) / len(PRF_output)
    T(i) = U(0) ^ U(1) ^ ... ^ U(c)
    U(0) = PRF( password, salt || INT_32_BE(i) )
    U(1) = PRF( password, U(0) )
    ...
    U(c) = PRF( password, U(c-1) )
{% endcodeblock %}

PBKDF1 is considered obsolete and currently replaced by its successor PBKDF2, as it could only produce derived keys of fixed, limited length.
PBKDF2 in turn is used in many popular encryption implementations, including WPA/WPA2 to secure WiFi networks, Mac OS X for user passwords, Android filesystem encryption and many more.
Typical WPA2 is based on HMAC-SHA1 PRF with network SSID as salt and number of iterations c = 4096, and produces 256-bit key.

Nevertheless, since these functions require very little RAM, ASIC/GPU brute-force attacks are relatively cheap and effective against them, so more powerful KDFs were needed.
The two most popular implementation of enhanced PBKDFs are bcrypt and scrypt which are described below.

###bcrypt
First presented in 1999, this KDF is the default password hash algorithm in BSD and many other systems.
It is based on Blowfish - fast block cipher with a notable remark - key changing is very slow there. 
Each new key requires pre-processing equivalent to encrypting about 4 kilobytes of text, which is very slow compared to other block ciphers. 
Altough that might be a problem for small embedded systems (e.g. some smartcards), such approach turned into benefit in PBKDFs: in order to conduct successful dictionary attack, much more time is needed.

Simplified algorithm of bcrypt is presented below.

{% codeblock lang:c %}
EksBlowfishSetup(cost, salt, key)
{
   state = InitState()
   state = ExpandKey(state,salt,key)
   repeat 2^cost:
      state = ExpandKey(state,0,salt)
      state = ExpandKey(state,0,key)
   return state
}


bcrypt(cost, salt, pwd)
{
   state = EksBlowfishSetup(cost,salt,key)
   ctext = "OrpheanBeholderScryDoubt"
   repeat 64:
      ctext = EncryptECB(state,ctext)
   return Concatenate(cost,salt,ctext)
}
{% endcodeblock %}

Comparing to "classic" PBKDF, bcrypt requires larger (but still fixed to 4KB) amount of RAM and is slightly stronger against brute-force attacks.
However, while bcrypt does a decent job at making life difficult for a GPU-enhanced attacker, it does little against a <a href="http://www.openwall.com/presentations/Passwords14-Energy-Efficient-Cracking/">FPGA-wielding attacker.</a>


###scrypt

The scrypt key derivation function was originally developed for use in the Tarsnap online backup system.
It can use arbitrarily large amounts of memory and is therefore much more resistant to hardware brute-force attacks than alternative functions such as PBKDF2 or bcrypt.
An amazing concept of *sequential memory-hard functions* is applied there - those can be computed by algorithms which use largest amount of storage possible, and cannot be parallelized effectively.

This can really be considered as breakthrough - by estimations, cost of cracking scrypted 8-char password in a year is <a href="http://www.tarsnap.com/scrypt/scrypt.pdf">approximately 4400 times more expensive</a> 
than one converted with bcrypt.

Its simplified structure can be described with the following pseudocode:

{% codeblock lang:c %}
scrypt(password, salt, N, p, r, keyLen)
// N - number of iterations for slow hash function      (CPU cost)
// p - how many blocks is used                          (parallelization cost)
// r - size of each block                               (memory cost)
{
   blockLen = 1024*r 
   iterCount = 1
   B = PBKDF2(HMAC_SHA256, password, salt, iterCount, p*blockLen) 
   for i in range(p):
      B[i] = ROMix(Salsa20(B[i]), N)
   return PBKDF2(HMAC_SHA256, password, B, iterCount, keyLen) 
}
{% endcodeblock %}

Let's look at the internals.
The large memory requirements of scrypt come from a large vector of pseudorandom bit strings that are generated as part of the algorithm (ROMix routine iteratively calls *BlockMix()* function which is desribed below).
Once the vector is generated, the elements of it are accessed in a pseudo-random order and combined to produce the derived key. 

{% codeblock lang:python %}
BlockMix( Salsa20, B )
{
   X = inverse(B)
   Y = []
   for bi in B:
      X = Salsa20(X ^ bi)
      Y += X
   return Y[0::2] + Y[1::2]
}
{% endcodeblock %}

In order to get rid of large memory requirements, there is a significant trade-off in speed.
This sort of time–memory trade-off can often be met in computer algorithms: you can increase speed at the cost of using more memory, or decrease memory requirements at the cost of performing more operations and taking longer. 
The idea behind scrypt is to deliberately make this trade-off costly in either direction. 
Thus an attacker could use an implementation that doesn't require many resources (and can therefore be massively parallelized with limited expense) but runs very slowly, or use an implementation that runs more quickly but 
has very large memory requirements and is therefore more expensive to parallelize. The main risk here (as always) is wrong implementation / poorly chosen parameters, which could reduce its <a href="http://blog.ircmaxell.com/2014/03/why-i-dont-recommend-scrypt.html">comparative benefits to zero.</a>

Although scrypt is somewhat new KDF, it's already rather common in the real-world applications. 
The first application was of course Tarsnap (N=2^14, p=1, r=8) - secure online backup service, company-creator of scrypt.
Probably the most well-known implementations are cryptocurrencies - Litecoin (N=2^10, p=1, r=1), YACoin (N=2^15, p=1, r=1) and many other altcoins. 
I feel I should mention Keybase application (N=2^15, p=1, r=8) too, as it was the reason why I actually decided to write the article :)


##Resume
KDFs is a nice cryptographic concept which, being properly implemented, can significantly improve overall level of security.
It is important to understand when it is better to use KDFs in place of other crypto primitives.
Notable applications of key derivation functions are:

1. Key diversification - obtaining additional keys from source key
2. Key stretching / key strengthening - password hashing approach to replace existing common hash functions used for verification
3. Basis for a system RNG to seed a pseudorandom generator (PRG).
4. Components of multiparty key-agreement protocols



## Q&A section
#### What is the actual difference between hash and KDF?
First of all, KDFs are more general in their applications, and not all KDFs should actually replace hashes (for example, HKDF is primarily used for secondary keys derivation).
Comparing to password-based KDFs, hashes are usually weaker (they do not satisfy randomness requirements) and, most important, are easier to bruteforce.
Modern PBKDFs are much slower by design and thus should be given preference over simple hashing.

#### Why do we need additional keys if we already have strong source key?
This is quite common scenario when additional keys are needed - e.g. in TLS (unidirectional keys: MAC key, encryption key, IV...) or in CBC (nonces).
In case an attacker obtains a derived key, he or she is not able to deduce either the input secret value or any of the other derived keys.

#### Why in the world do we need our input to be uniformly random in order to derive additional keys?
Well, it turns out that if the condition of input randomness is not met, the output might not look random. 
In context of keys used to secure the sessions, it might be possible for an attacker to anticipate some of the session keys and thereby break the session.


P.S. If you still have any questions, feel free to ask, and I'll publish answers here.
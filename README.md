# The Socat backdoor

This repo contains some research I'm currently doing on the Socat backdoor.

On February 1st 2016, a security advisory was posted to Openwall by a [Socat](http://www.dest-unreach.org/socat/) developer: [Socat security advisory 7 - Created new 2048bit DH modulus](http://www.openwall.com/lists/oss-security/2016/02/01/4)

> In the OpenSSL address implementation the hard coded 1024 bit DH p parameter **was not prime**. The effective cryptographic strength of a key exchange using these parameters was weaker than the one one could get by using a prime p. Moreover, since there is no indication of how these parameters were chosen, the existence of a trapdoor that makes possible for an eavesdropper to recover the shared secret from a key exchange that uses them cannot be ruled out.  
> A new prime modulus p parameter has been generated by Socat developer using OpenSSL dhparam command.  
> In addition the new parameter is 2048 bit long.

This is a pretty weird message with a [Juniper](http://forums.juniper.net/t5/Security-Incident-Response/Important-Announcement-about-ScreenOS/ba-p/285554) feeling to it. 

[Socat's README](http://www.dest-unreach.org/socat/doc/README) tells us that you can use their free software to setup an encrypted tunnel for data transfer between two peers.

Looking at the commit logs you can see that they used a 512 bits Diffie-Hellman modulus until last year (2015) january when [it was replaced with a 1024 bits one](http://repo.or.cz/socat.git/commitdiff/281d1bd6515c2f0f8984fc168fb3d3b91c20bdc0).

> Socat did not work in FIPS mode because 1024 instead of 512 bit DH prime is required. Thanks to **Zhigang Wang** for reporting and sending a patch.

The person who pushed the commit is *Gerhard Rieger* who is the same person who fixed it a year later. In the comment he refers to *Zhigang Wang*, an Oracle employee at the time who has yet to comment on his mistake.

This research tries to understand how this could be a backdoor. And more particularly, a [NOBUS](https://en.wikipedia.org/wiki/NOBUS) one (*Nobody But Us*). Here are the objectives of this research:

* Build a proof of concept of such a NOBUS backdoor
* In failures of building NOBUS backdoors, check if I can use the socat backdoor
* Try to answer: "does it look like a backdoor?"

# Human errors

How likely is it a human error?

* I've tried reversing the prime to see if the developer made a mistake, but nothing.

![reversed dh](http://i.imgur.com/L0VxosD.png)

* [Someone proposed](https://www.reddit.com/r/crypto/comments/43wh7h/the_socat_backdoor/czlxydf) "Swap each 4-byte section or 8-byte section as though someone messed up endian conversion." I haven't tried that yet.

# Pohlig Hellman and small subgroup attacks

I won't write an explanation of these attacks here, but they are the reason why a non-prime modulus could be a NOBUS backdoor.

If a backdoor allowing such attacks was generated with a prime modulus, then anyone could reveal the order (by computing `modulus - 1`) and see if it is smooth by simply factoring it (and then take advantage of the backdoor).

# How to implement a NOBUS DH

There is a working proof of concept in PoC.sage that implements one way of doing it (I expect more ways to generate backdoored primes)

It creates a non-prime modulus `p = p_1 * p_2` with `p_i` primes, such that
`p_i - 1` is smooth. Since the order of the group will be `(p_1 - 1)(p_2 - 1)` (smooth) and known only to the malicious person who generated `p`, Pohlig-Hellman can be used to recover the private key

In the proof of concept the small subgroup attack is implemented instead of Pohlig-Hellman just because it seemed easier to code. They are relatively equivalent except that in practice an ephemeral key is used and such small subgroup attacks are not that practical.

Note that these issues should not arrise if the DH parameters were generated properly, that is the order and subgroups orders should be known (and used to verify that the public key received lies in the correct subgroup). See [rfc2785](https://tools.ietf.org/html/rfc2785) for more information.

![proof of concept](http://i.imgur.com/CL2wk5V.png)

The proof of concept is a step by step explanation of what's happening. Above you can see the generation of the backdoored modulus, bellow you can see the attack tacking place and recovering discrete logs of each of the subgroups

![discrete logs](http://i.imgur.com/KojNtVY.png)

To run it yourself you will need Sage. You can also use an online version of it a [cloud.sagemath.com](http://cloud.sagemath.com).

# How to reverse socat's non-prime dh1024_p

from what we learned in implementing such a backdoor, we will see how we can reverse it to use it ourselves.

* What are the chances that if this was non-prime was a mistake, it generated factors large enough so that no one can reverse it?

# How to reverse socat's new prime dh2048_p's order

A new order has been generated, but we know nothing about its order. This doesn't sound good and this section should explain why (from a backdoor point of view) and explain how we can try to reverse it...

![unknown order](http://i.imgur.com/AKbKna3.png)

The above shows that the new modulus was not generated as `p = 2q + 1`

# Resources

* [Thai Duong's blogpost](http://vnhacker.blogspot.com/2016/02/exploiting-diffie-hellman-bug-in-socat.html)
* [crypto stackexchange's post](http://crypto.stackexchange.com/questions/32415/how-does-a-non-prime-modulus-for-diffie-hellman-allow-for-a-backdoor/32431?noredirect=1)
* [reddit's thread](https://www.reddit.com/r/crypto/comments/43wh7h/the_socat_backdoor/)

---
layout: post
title:  Alice, Bob, and their shared secret
date:   2017-12-11 12:35:00 -0500
published: true
---
Have you ever wondered what encryption is, or how it works? Me too! In this post I'll discuss this topic, and walk you through some [C code](https://github.com/greenteadigital/C-primes) I wrote to generate large prime numbers. Large primes are the basis of much (if not all) of today's current cryptography.

There are many excellent resources online for learning about cryptography, and I don't intend to reproduce them here. Instead, I want to focus on a type of key agreement protocol called Diffie-Hellman ("DH").

In a normal browsing session, you probably use DH many, many times without knowing it. Sometime when you're connected to a website over HTTPS, click on the little green lock icon in the upper left corner of your browser. Depending on your browser and your connection, you may see something like TLS\_EC**_DH_**E\_RSA\_WITH\_AES\_128\_GCM\_SHA256. Those two letters in bold stand for Diffie-Hellman.

 The really cool thing about DH is that it allows for two parties who wish to establish encrypted communications to agree on an encryption key over an insecure channel. This means that even if a passive foe is recording all the traffic exchanged during a DH key agreement "handshake", it will be practically impossible for them to determine the agreed-upon key, assuming that an adequately large key size is chosen. Notice I said a "passive foe". If you are facing an active foe with the ability play the role of man-in-the-middle, you will need an additional verification method to be sure that the party with which you are communicating is the one you intend. That subject can be the topic of a future post.
 
(to be continued)
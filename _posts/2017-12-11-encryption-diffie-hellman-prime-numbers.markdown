---
layout: post
title:  Alice, Bob, and their shared secret
date:   2017-12-11 12:35:00 -0500
published: true
---
Have you ever wondered what encryption is, or how it works? Me too! In this post I'll discuss this topic, and walk you through some [C code](https://github.com/greenteadigital/C-primes) I wrote to generate large prime numbers. Large primes are the basis of much (if not all) of today's current cryptography.

### Diffie-what? Is that a mayonnaise?
There are many excellent resources online for learning about cryptography, and I don't intend to reproduce them here. Instead, I want to focus on a type of key agreement protocol called Diffie-Hellman ("DH").

In a normal browsing session, you probably use DH many, many times without knowing it. Sometime when you're connected to a website over HTTPS, click on the little green lock icon in the upper left corner of your browser. Depending on your browser and your connection, you may see something like TLS\_EC**DH**E\_RSA\_WITH\_AES\_128\_GCM\_SHA256. Those two letters in bold stand for Diffie-Hellman.

 The really cool thing about DH is that it allows for two parties who wish to establish encrypted communications to agree on an encryption key over an insecure channel. This means that even if a passive foe is recording all the traffic exchanged during a DH key agreement "handshake", it will be practically impossible for them to determine the agreed-upon key. Notice I said a "passive foe". If you are facing an active foe with the ability play the role of man-in-the-middle, you will need additional verification to be sure that the party you're communicating with is who you think they are. That can be a topic for a future post :)
 
Two other things are required to make DH secure: the parameters used during the DH computation must be of adequate size, and they must be prime. This last requirement is where my code comes in. What I've created is a prime number generating daemon for Unix-likes.

Now lemme just say up-front: when it comes to writing C, I'm still a n00b. So, if you're an old hand at C, don't be surprised if something doesn't look quite right ;) That being said, I've taken significant time to test and verify that the code works as intended without memory leaks or data corruption. So, if you find an issue along those lines, please tweet me at the link in the header!

### On to the code...
Uh... not so fast. Before we dive into the code, a word on the architecture. The daemon's main thread recieves incoming UDP traffic on port 31397 (itself a prime number, more on this later). Clients wishing to recieve a generated prime send the bit length of the requested prime to the daemon on said port. Only 4 bit lengths are currently supported: 1024, 2048, 3072, and 4096. The daemon will spawn the number of child threads configured in primes.h to do the heavy lifting. The number of threads should be equal to the number of _physical_ cores available to the process. While the child threads race to be the first to find a random prime of the given length, the main thread goes back to receiving new requests. While the daemon is capable of servicing multiple requests concurrently, performance will undoubtedly suffer due to the CPU-bound nature of the operation.

(to be continued)
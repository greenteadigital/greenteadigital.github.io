---
layout: post
title:  Alice, Bob, and their shared secret
date:   2017-12-11 12:35:00 -0500
published: true
---
Have you ever wondered what encryption is, or how it works? Me too! In this post I'll discuss this topic, and walk you through some [C code](https://github.com/greenteadigital/C-primes) I wrote to generate large prime numbers. Large primes are the basis of much (if not all) of today's current cryptography.

### Diffie-what? Is that a mayonnaise?
There are many excellent resources online for learning about cryptography, and I don't intend to reproduce them here. Instead, I want to focus on a type of key agreement protocol called Diffie-Hellman ("DH").

In a normal browsing session, you probably use DH many, many times without knowing it. Sometime when you're connected to a website over HTTPS, click on the little green lock icon in the upper left corner of your browser. Depending on your browser and your connection, you may see something like <code>TLS_EC<b>DH</b>E_RSA_WITH_AES_128_GCM_SHA256</code>. Those two letters in bold stand for Diffie-Hellman.

 The really cool thing about DH is that it allows for two parties who wish to establish encrypted communications to agree on an encryption key over an insecure channel. This means that even if a passive foe is recording all the traffic exchanged during a DH key agreement "handshake", it will be practically impossible for them to determine the agreed-upon key. Notice I said a "passive foe". If you are facing an active foe with the ability play the role of man-in-the-middle, you will need additional verification to be sure that the party you're communicating with is who you think they are. That can be a topic for a future post :)
 
Two other things are required to make DH secure: the parameters used during the DH computation must be of adequate size, and they must be prime. This last requirement is where my code comes in. What I've created is a prime number generating daemon (`primesd`) for Unix-like OSes.

Now lemme just say up-front: when it comes to writing C, I'm still a `n00b`. So, if you're an old hand at C, don't be surprised if something doesn't look quite right ;) That being said, I've taken significant time to test and verify that the code works as intended without memory leaks or data corruption. So, if you find an issue along those lines, send me a tweet at the link in the header!

### High-level architecture and performance
Before we dive into the code, a word on the architecture. The daemon's main thread receives incoming UDP traffic on port 31397 (itself a prime number, more on this later). Clients wishing to recieve a generated prime send the bit length of the requested prime to `primesd` on said port. Only 4 bit lengths are currently supported: 1024, 2048, 3072, and 4096. `primesd` will spawn the number of child threads configured in [primes.h](https://github.com/greenteadigital/C-primes/blob/master/primes/primes.h) to do the heavy lifting. The number of threads should be equal to the number of _physical_ cores available to the process. While the child threads race to be the first to find a random prime of the given length, the main thread goes back to receiving new requests. While the daemon is capable of servicing multiple requests concurrently, performance will undoubtedly suffer due to the CPU-bound nature of the operation.

On the subject of performance, I ran the [reference client](https://github.com/greenteadigital/C-primes/blob/master/primes/getprime.c) in a loop of 100 iterations using the following command at a bash prompt (replacing `$BITLEN` with the length being tested):
````bash
time for n in {1..100}; do ./getprime -b $BITLEN; done
````
These are the times my quad-core i7 @2.5GHz put up to generate 100 primes, by bitlength requested:

Bit Length | Real (Clock) Time | Average Time
---------- | ----------------- | ---------------
1024       | 1.810s  | 18ms
2048       | 14.281s | 143ms
3072       | 58.870s | 589ms
4096       | 2m12.984s | 1.33s
{:.times}
<br/>
Judging by those times, the sweet spot for security and performance is 2048-bit primes. Of course, your mileage will vary.

One final note before we walk the code, while the daemon and client do use sockets, the intention was that all traffic between daemon and client would travel the loopback device. And I would probably agree with those that criticize the choice of UDP. At least in the sense that packets may be dropped. However, since primes are returned to clients as hexadecimal strings, even in the worst case of 4096-bit primes, the response will only be 1024 bytes in length. If you run `sysctl -a | grep udp`, it's unlikely you have `maxdgram` or `recvspace` set to less than 1024. But that is no guarantee a packet will never be dropped, so tread with caution.

### Let's you and me take a little walk(through)
It's not a long trip. `primesd` requires only 2 source files: a header file with some constants and a `struct` definition, and primesd.c which contains `main`. Let's take a look at the header, its pretty simple:
````c
#pragma once

#include <stdbool.h>

#define PRIMES_DAEMON_PORT 31397
#define LOCALHOST "127.0.0.1"
#define NTHREADS 4
#define SEED_SZ_BYTES 16	// 16 * 8 = 128 bits for random seed

struct thread_params {
	struct sockaddr_in r_addr;
	volatile bool served_prime;
	short bitsize;
	volatile char live_thread_count;
};
````
The only thing here that might benefit from some explanation is the `struct`. A new struct of this type is created for every request, and is shared among child threads. The two non-volatile members hold the socket address of the client, and the requested bitlength, repectively. The volatile members keep track of whether a prime has been found and returned to the client, and how many child threads are still alive. Both are protected by mutexes.

Which brings us to the main file, [primesd.c](https://github.com/greenteadigital/C-primes/blob/master/primes/primesd.c). It consists of the `main` function, the `getPrime` function which child threads run, one function to create threads and another to destroy them, and a final function which gets a random seed. Six variables are declared at the file level: a boolean, a file pointer, an integer, and 3 mutexes. Only the boolean, `DBG`, gets initialized. If you want debug output printed to stdout, set that to true.
````c
bool DBG = false;

FILE *urandom;
int sockfd; // local socket

pthread_mutex_t urandom_lock;
pthread_mutex_t served_lock;
pthread_mutex_t tcnt_lock;
````
The remaining five will be initialized in `main`:
````c
int main(int argc, char *argv[]) {
	
	urandom = fopen("/dev/urandom", "r");
	
	pthread_mutex_init(&urandom_lock, NULL);
	pthread_mutex_init(&served_lock, NULL);
	pthread_mutex_init(&tcnt_lock, NULL);
	
	sockfd = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
````
Notice that the previous line created a socket and returned the file descriptor. However, this socket has to be bound to an address to be useful. Helpfully, by inluding `<netinet/in.h>`, we have a struct available to hold the adress: `struct sockaddr_in`. So we declare one, and then zero out its memory.
````c
    struct sockaddr_in l_addr;	// daemon socket address
    memset((void *) &l_addr, 0, sizeof(l_addr));
````
Now we're ready to set the address parameters: the address family, the port, and the IP address. Two helper functions make dealing with differences in endian-ness and machine vs. human-readable formats easy: `htons` (host to network short) and `inet_pton` (inet presentation to network). And then finally the address struct is bound, via its file descriptor, to the socket we created earlier.
````c
    l_addr.sin_family = AF_INET;
    l_addr.sin_port = htons(PRIMES_DAEMON_PORT);
    inet_pton(AF_INET, LOCALHOST, &(l_addr.sin_addr.s_addr));

    bind(sockfd, (struct sockaddr*) &l_addr, sizeof(l_addr));
````
So now we have a UDP socket bound to an address. If you're used to using TCP sockets, this is a little different. We don't have to listen for and accept connections, we just `recvfrom()`. But before we can do that, there's a bit more setup to do. We need a buffer to hold the bitlength the client requests, and another zeroed-out `struct sockaddr_in` to hold the client's address info. Like so:
````c
	char buffsz = 4;
	char client_req_bitlen[buffsz];
	
	struct sockaddr_in r_addr;	// remote client address
	socklen_t remotesz = sizeof(r_addr);
	memset((void *) &r_addr, 0, sizeof(r_addr));
````
Before the main thread enters its infinite `recvfrom()` loop, we'll declare two more variables: a `struct thread_params` which...

(to be continued)
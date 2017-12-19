---
layout: post
title:  Alice, Bob, and their shared secret
date:   2017-12-11 12:35:00 -0500
published: true
---
Have you ever wondered what encryption is, or how it works? Me too! In this post I'll discuss this topic, and walk you through some [C code](https://github.com/greenteadigital/C-primes) I wrote to generate large prime numbers. Large primes are the basis of much (if not all) of today's current cryptography.

### Diffie-what? Is that a mayonnaise?
There are many excellent resources online for learning about cryptography, and I don't intend to reproduce them here. Instead, I want to focus on a type of key agreement protocol called Diffie-Hellman ("DH") and the generation of large prime numbers commonly used within.

In a normal browsing session, you probably use DH many, many times without knowing it. Sometime when you're connected to a website over HTTPS, click on the little green lock icon in the upper left corner of your browser. Depending on your browser and your connection, you may see something like <code>TLS_EC<b>DH</b>E_RSA_WITH_AES_128_GCM_SHA256</code>. Those two letters in bold stand for Diffie-Hellman.

 The really cool thing about DH is that it allows for two parties who wish to establish encrypted communications to agree on an encryption key over an insecure channel. This means that even if a passive foe is recording all the traffic exchanged during a DH key agreement "handshake", it will be practically impossible for them to determine the agreed-upon key. Notice I said a "passive foe". If you are facing an active foe with the ability play the role of man-in-the-middle, you will need additional verification to be sure that the party you're communicating with is who you think they are. That can be a topic for a future post :)
 
Two other things are required to make DH secure: the parameters used during the DH computation must be of adequate size, and they must be prime. This last requirement is where my code comes in. What I've created is a prime number generating daemon (`primesd`) for Unix-like OSes.

Now lemme just say up-front: when it comes to writing C, I'm still a `n00b`. So, if you're an old hand at C, don't be surprised if something doesn't look quite right ;) That being said, I've taken significant time to test and verify that the code works as intended without memory leaks or data corruption. So, if you find an issue along those lines, send me a tweet at the link in the header!

### High-level architecture and performance
Before we dive into the code, a word on the architecture. The daemon's main thread receives incoming UDP traffic on port 31397 (itself a prime number, <a href="#fun-picking-ports">more on this</a> later). Clients wishing to recieve a generated prime send the bit length of the requested prime to `primesd` on said port. Only 4 bit lengths are currently supported: 1024, 2048, 3072, and 4096. `primesd` will spawn the number of child threads configured in [primes.h](https://github.com/greenteadigital/C-primes/blob/master/primes/primes.h) to do the heavy lifting. The number of threads should be equal to the number of _physical_ cores available to the process. While the child threads race to be the first to find a random prime of the given length, the main thread goes back to receiving new requests. While the daemon is capable of servicing multiple requests concurrently, performance will undoubtedly suffer due to the CPU-bound nature of the operation.

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
Judging by those times, the sweet spot for security and performance is 2048-bit primes. Of course, your mileage will vary depending on your hardware, load, and security requirements.

One final note before we walk the code, while the daemon and client do use sockets, the intention was that all traffic between daemon and client would travel the loopback device. And I would probably agree with those that criticize the choice of UDP. At least in the sense that packets may be dropped. However, since primes are returned to clients as hexadecimal strings, even in the worst case of 4096-bit primes, the response will only be 1024 bytes in length. If you run `sysctl -a | grep udp`, it's unlikely you have `maxdgram` or `recvspace` set to less than 1024. But that is no guarantee a packet will never be dropped, so tread with caution.

### Let's you and me take a little walk(through)
`primesd` requires only 2 source files: a header file with some constants and a `struct` definition, and primesd.c which contains `main`. Let's take a look at the header, its pretty simple:
````c
#pragma once

#include <stdbool.h>

#define PRIMES_DAEMON_PORT 31397
#define LOCALHOST "127.0.0.1"
#define NTHREADS 4
#define SEED_SZ_BYTES 16	// 16 * 8 = 128 bits for random seed
#define DBG false

struct thread_params {
	struct sockaddr_in r_addr;
	volatile bool served_prime;
	short bitsize;
	volatile char live_thread_count;
};
````
The only thing here that might benefit from some explanation is the `struct`. A new struct of this type is created for every request, and is shared among child threads. The two non-volatile members hold the socket address of the client, and the requested bitlength, repectively. The volatile members keep track of whether a prime has been found and returned to the client, and how many child threads are still alive. Both volatiles are protected by mutexes.

Which brings us to the main file, [primesd.c](https://github.com/greenteadigital/C-primes/blob/master/primes/primesd.c). It consists of the `main` function, the `getPrime` function which child threads run, one function to create threads and another to destroy them, and a final function which gets a random seed. Five variables are declared at the file level: a file pointer, an integer, and 3 mutexes.
````c
FILE *urandom;
int sockfd; // local socket

pthread_mutex_t urandom_lock;
pthread_mutex_t served_lock;
pthread_mutex_t tcnt_lock;
````
Those five will be initialized in `main`:
````c
int main(int argc, char *argv[]) {
	
	urandom = fopen("/dev/urandom", "r");
	
	pthread_mutex_init(&urandom_lock, NULL);
	pthread_mutex_init(&served_lock, NULL);
	pthread_mutex_init(&tcnt_lock, NULL);
	
	sockfd = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
````
Notice that the previous line created a socket and returned the file descriptor. However, this socket has to be bound to an address to be useful. Helpfully, by including `<netinet/in.h>`, we have a struct available to hold the adress: `struct sockaddr_in`. So we declare one, and then zero out its memory.
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
	
	struct sockaddr_in r_addr;	// "remote" client address
	socklen_t remotesz = sizeof(r_addr);
	memset((void *) &r_addr, 0, sizeof(r_addr));
````
Before the main thread enters its infinite `recvfrom()` loop, we'll declare two more variables: a `struct thread_params` instance to hold the data shared across worker threads, and `char recv_count` to track the number of bytes received from each client request.

Now we enter the main loop of the main thread:

````c
while(1) {
    
    recv_count = 0;
    while (recv_count < buffsz) {
        recv_count += recvfrom (
                sockfd,
                (void *) (&client_req_bitlen + (recv_count * sizeof(char))),
                sizeof(client_req_bitlen) - recv_count,
                0,
                (struct sockaddr *) &r_addr,
                &remotesz );
    }
````
...which starts out looping over the input from the requesting client. Although the 4 byte request is likely fully ingested by the end of the first iteration, pointer arithmetic in the 2nd argument will assure the full request is received and stored.

Once we get the full request, we validate it. If it fails, we ignore it and continue with the next iteration.
````c
    if ( ! ( strncmp(&client_req_bitlen[0], "1024", 4) == 0
        || strncmp(&client_req_bitlen[0], "2048", 4) == 0
        || strncmp(&client_req_bitlen[0], "3072", 4) == 0
        || strncmp(&client_req_bitlen[0], "4096", 4) == 0)) {
        
        continue;
    }
````
We then initialize the `thread_params` struct which is passed to child threads, and copy it out onto the heap. Otherwise it could be clobbered by the next request, as the instance created in `main()` is reused.
````c
    params.bitsize = strtol(&client_req_bitlen[0], 0, 0);
    params.r_addr = r_addr;
    params.served_prime = false;
    params.live_thread_count = 0;
    		
    void *pp = malloc(sizeof(params));
    memcpy(pp, &params, sizeof(params));
````
Then we have an optional diagnostic print just before child threads get initialized in `initThreads`.
````c
    if (DBG) printf("\nPreparing to send %s-bit prime to port %d\n", &client_req_bitlen[0],
                    ntohs(((struct thread_params *) pp)->r_addr.sin_port));
    
    initThreads(pp);
}
````

### You're still here!?
Awesome :) You deserve some internet points for making it this far! Take a break, refresh yourself... Done? Ok, good. Let's continue.

### In which the sausage gets made
The `initThreads` function does little more than spawn the configured number of threads and increment `live_thread_count` in the shared struct.
````c
void initThreads(void *params) {
	
	for (int i = 0; i != NTHREADS; i++) {
		pthread_t new_thread;
		pthread_create(&new_thread, NULL, (void *) &getPrime, params);
		pthread_mutex_lock(&tcnt_lock);
		((struct thread_params *) params)->live_thread_count++;
		pthread_mutex_unlock(&tcnt_lock);
		pthread_detach(new_thread);
	}
}
````
The function each thread executes is `getPrime`. It relies on the fantastic [GNU Multiple Precision Arithmetic Library](https://gmplib.org/) to handle calculations involving the very large numbers required for DH. 
````c
void getPrime(void *thread_params) {
	
	struct thread_params *params = (struct thread_params *) thread_params;
	unsigned char seed_buff[SEED_SZ_BYTES];
	
	gmp_randstate_t state;
	gmp_randinit_mt(state);
	
	mpz_t seed, randnum, one;
	mpz_init(seed);
	mpz_init(randnum);
	mpz_init(one);
	
	unsigned char single = 1;
	mpz_import(one, 1, -1, 1, 0, 0, &single);
````
As you see above, we initialize quite a few things to begin with: a buffer to hold a random seed, and several types provided by GMPlib. Those include a random state and several integer types. More about those later.
Next a call is made to [`getUrandom`](https://github.com/greenteadigital/C-primes/blob/a9cdc82f752efb49774a1c91bcdae08060a9deea/primes/primesd.c#L25), which reads the system CSPRNG to seed the random state we already established. A mutex in `getUrandom` protects reads to assure each thread's seed is distinct. 
````c
	getUrandom(&seed_buff[0], SEED_SZ_BYTES);
	
	mpz_import(seed, SEED_SZ_BYTES, -1, 1, 0, 0, &seed_buff[0]);
	gmp_randseed(state, seed);
```` 
Then the code enters what can be considered the core loop: first check to see if another thread has already found and returned a prime number to the client. If yes, function `thread_exit` is called. Otherwise, a random number is generated of the requested length, bitwise OR'd with one to assure that it's odd, and then probabilistically tested for primality.
````c
while(1) {
    
    pthread_mutex_lock(&served_lock);
    if (params->served_prime) thread_exit(params, &state, &seed, &randnum, &one);
    pthread_mutex_unlock(&served_lock);
    
    mpz_urandomb(randnum, state, params->bitsize);
    mpz_ior(randnum, randnum, one);	// make sure randnum is odd by bitwise-ORing with 1
    if (mpz_probab_prime_p(randnum, 17)) break;
}
````
Yes, in fact, this does mean that the probability that the dameon returns a composite number is non-zero, but it is still quite small at <0.000000000058207661. If you want greater assurance, use an integer larger than 17 as the second argument to `mpz_probab_prime_p` (you can [RTFM](https://gmplib.org/manual/Number-Theoretic-Functions.html)). The remainder of `getPrime`, listed below, is concerned with storing a hexidecimal string representation of the random prime `randnum`, returning it to the client if that hasn't happened yet, and terminating by calling `thread_exit`.
````c
	char outstr[(params->bitsize / 4) + 1];
	memset((void *) &outstr[0], 0, sizeof(outstr));
	int send_size = gmp_sprintf((void *) &outstr[0], "%Zx", randnum);
	socklen_t remotesz = sizeof(params->r_addr);
	
	pthread_mutex_lock(&served_lock);
	
	if (!params->served_prime) {
		sendto(sockfd, (void *) &outstr, send_size, 0, (struct sockaddr *) &(params->r_addr), remotesz);
		params->served_prime = true;
		if (DBG) {
			printf("\nDaemon: sent prime over port %d: %s\n", ntohs(params->r_addr.sin_port), &outstr[0]);
		}
		thread_exit(params, &state, &seed, &randnum, &one);
	} else {
		thread_exit(params, &state, &seed, &randnum, &one);
	}
````
The `thread_exit` function, below, reads a mutex-protected counter to determine when it can `free()` the shared `thread_params` struct, releases resources allocated by GMPlib types, and unlocks the `served_lock` mutex locked in the previous block above. And then, it's _sayonara_.
````c
void thread_exit(struct thread_params *params,
				 gmp_randstate_t *state,
				 mpz_t *seed,
				 mpz_t *randnum,
				 mpz_t *one) {
	
	pthread_mutex_lock(&tcnt_lock);
	
	if (params->live_thread_count == 1) {
		pthread_mutex_unlock(&tcnt_lock);
		free(params);
	} else {
		params->live_thread_count -= 1;
		pthread_mutex_unlock(&tcnt_lock);
	}
	mpz_clear(*seed);
	mpz_clear(*randnum);
	mpz_clear(*one);
	gmp_randclear(*state);
	pthread_mutex_unlock(&served_lock);
	pthread_exit(0);
}
````
### Fun picking ports<a name="fun-picking-ports"></a>
I had a bit of geeky fun deciding which port the daemon would receive requests on. I decided to do a search of the non-privileged port space to see which port numbers are prime. For this I used Python, one of my favorite langs to code in.
````python
import miller_rabin as mr

def getPrimePorts():
    count = 0
    previous = -1
    gaps = {}
    
    for port in xrange(65535, 1024, -1):
        if mr.isPrime(port):
            if previous > -1:
                gap = previous - port
                if gap in gaps:
                    gaps[gap] += 1
                else:
                    gaps[gap] = 1
                print '      ' + str(gap)
                print port
#                if gap == 72:
#                    break
            previous = port
            count += 1
    print 'gap size, count'
    print '---------------'
    for gap in gaps.items():
        print gap
    print "\ncount", count
    print "max gap", max(gaps)

getPrimePorts()
````
This code uses the same type of probabilistic primality test (Miller-Rabin) that `primesd` does. The imported module is [here](https://github.com/greenteadigital/pycrypto/blob/master/miller_rabin.py). If you run the code above, the following will be printed:
````python
gap size, count
---------------
(2, 824)
(4, 818)
(6, 1332)
(8, 525)
(10, 608)
(12, 639)
(14, 322)
(16, 224)
(18, 357)
(20, 141)
(22, 145)
(24, 112)
(26, 52)
(28, 60)
(30, 91)
(32, 19)
(34, 21)
(36, 25)
(38, 10)
(40, 15)
(42, 8)
(44, 3)
(48, 1)
(50, 3)
(52, 6)
(54, 3)
(58, 2)
(60, 1)
(62, 1)
(72, 1)

count 6370
max gap 72
````
And if you then uncomment the two lines that are commented, you'll see:
````python
...
31469
      72
31397
...
````
So, of all the prime unprivileged ports, the largest gap separating two primes is 72, and the prime just above that gap is the port the daemon `recv()`s on: 31397. Do I get my nerd badge now? :smile:

Thanks for reading! I hope you learned something interesting, or at least enjoyed yourself. Talk to you soon!

Ben.
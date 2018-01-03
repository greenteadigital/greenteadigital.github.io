---
layout: post
title:  Password-based file encryption with Python 2
date:   2017-12-20 17:13:00 -0500
published: true
---
### Crypto train keeps rollin' on...
Seeing as I'm already on the subject of crypto with my [last post](/2017/12/11/encryption-diffie-hellman-prime-numbers.html), I'd like to swalk you through some [Python code](https://github.com/greenteadigital/pycrypto/blob/master/PBKDF2.py) I wrote to provide password-based file encryption and decryption capabilities. Of course, all the usual [disclaimers](https://github.com/greenteadigital/pycrypto/blob/master/README.md) apply.

The idea here is simple: choose a file that you want to encrypt. Launch `PBKDF2.py` with the path to the file as the first arg (drag-drop), answer some questions, supply a password, and _Voilà_! You now have a secure version of said file you can attach to an email, or transmit through other insecure means. Of course, it will be only as secure as your chosen password, so make it [strong](https://support.google.com/accounts/answer/32040?hl=en)! And of course, you did read the [disclaimer](https://github.com/greenteadigital/pycrypto/blob/master/README.md), _right_?

### Ewww, did you step in some code?
Enough pretext, let's step into the code :)
````python
import getpass
from hashlib import sha512, sha384, sha256, sha224
import os
import struct
import sys
import zlib
import bz2
from cStringIO import StringIO as strio
import userinput as usr

ALGOS = {
	0: sha224,
	1: sha256,
	2: sha384,
	3: sha512
}
DIGESTSIZES = {
	0: ALGOS[0]().digest_size,
	1: ALGOS[1]().digest_size,
	2: ALGOS[2]().digest_size,
	3: ALGOS[3]().digest_size
}
COMPRESSION = {
	0: None,
	1: zlib,
	2: bz2
}
CRYPT_EXT = '.phse'
MAGIC = "PBKDF2-HMAC-SHA2"
PWD_HASH_MULT = 20
sha2 = None	# set later
````
Other than the [`userinput` module](https://github.com/greenteadigital/pycrypto/blob/master/userinput.py), all the imports come from Python's "batteries included" standard library. Some constants are initialized to support a choice of hashing and compression algorithms. An uninitialized `sha2` variable is also declared. Then we have some function `def`s...
````python
def _exit():
	raw_input("\npress Enter to exit...")
	sys.exit()
````
...`_exit`, which should be self-explanatory. Then two functions for packing and unpacking metadata to and from bitfields. The `bitPack` function is used to store user-selected preferences, like hashing algorithm and compression type, and `bitUnpack` which extracts the stored preferences during the decryption phase.
````python
def bitPack(algonum, exp_incr, compressornum):
	bitstr = (bin(algonum)[2:].zfill(2)
			+ bin(exp_incr)[2:].zfill(2)
			+ bin(compressornum)[2:].zfill(2)
			+ '00')	## two bits left for storing metadata
	
	r = int('0b' + bitstr, 2)
	return r

def bitUnpack(_int):
	bitstr = bin(_int)[2:].zfill(8)
	algonum = int('0b' + bitstr[:2], 2)
	increment_by = int('0b' + bitstr[2:4], 2)
	compressornum = int('0b' + bitstr[4:6], 2)
	
	r = (algonum, increment_by, compressornum)
	return r 
````
Following that is a function for doing constant-time comparisons. This is used to mitigate a type of [side-channel attack](https://en.wikipedia.org/wiki/Side-channel_attack) called a [timing attack](https://en.wikipedia.org/wiki/Timing_attack). So, what is being compared? During the encryption phase the provided password is salted and hashed [many, many](https://github.com/greenteadigital/pycrypto/blob/42ca526462554898accddf0d4464984b1bcbdfb2/userinput.py#L30) times and stored in the encrypted file. It's during the decryption phase when a comparison is made of the salted hash of the provided password, and the hash stored in the encrypted `.phse` file. Putting on my critic's hat for a moment, I can see credible arguments being made that either:
1. It's an unacceptable risk to include ANY form of the password in the encrypted file, or
2. Constant-time comparison of hashes is not necessary because a small change in the input (`hunter1` -> `hunter2`) will produce dramtically different hashes, making the timing variance of a naive comparison of little use.

While I tend to agree with the spirit of criticism #1, the addition of a random salt and a large, tunable iteration count make it less applicable. My reponse to criticism #2 is that yes, while it's true that the comparison of _secure hashes_ leaves no visible opening for a timing attack, the future evolution of the software may require comparisons not yet contemplated. So let's do the right thing right off the bat.
````python
def constTimeCompare(val1, val2):
	if len(val1) != len(val2):
		return False
	result = 0
	for x, y in zip(val1, val2):
		result |= ord(x) ^ ord(y)

	return not result
````
Following that comparison function is a core function which generates cryptograhic keying material.
````python
def genKeyBlock(password, salt):
	blksz = sha2().block_size
	passlen = len(password)
	if passlen < blksz:
		password += (chr(0) * (blksz - passlen))
	else:
		password = sha2(password).digest()
	o_pad = ''.join(chr(0x5c ^ ord(char)) for char in password)
	i_pad = ''.join(chr(0x36 ^ ord(char)) for char in password)

	return sha2(o_pad + sha2(i_pad + salt).digest()).digest()
````



(to be continued)
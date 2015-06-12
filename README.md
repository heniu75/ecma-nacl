# ecma-nacl: Pure JavaScript (ECMAScript) version of NaCl cryptographic library.

[NaCl](http://nacl.cr.yp.to/) is a great crypto library that is not placing a burden of crypto-math choices onto developers, providing only solid high-level functionality (box - for public-key, and secret_box - for secret key  authenticated encryption), in a let's [stop blaming users](http://cr.yp.to/talks/2012.08.08/slides.pdf) of cryptographic library (e.g. end product developers, or us) manner.
Take a look at details of NaCl's design  "[The security impact of a new cryptographic
library](http://cr.yp.to/highspeed/coolnacl-20120725.pdf)".

ecma-nacl is a re-write of NaCl in TypeScript, which is ECMAScript with compile-time types.
Library implements NaCl's box and secret box.
Signing code comes from SUPERCOP version 2014-11-24.

The following have been added to ecma-nacl beyond NaCl.
Firstly, it is a re-write of [scrypt](http://www.tarsnap.com/scrypt.html) key derivation function.
Scrypt is a highly valuable thing for services that allow users to have passwords, while doing proper work with crypto keys, derived from those passwords.
Secondly, it is an XSP file format for objects encrypted with NaCl. 

Re-writes are based on C sources, included in this repository.
Tests are written to correspond those in C code, to make sure that output of this library is the same as that of C's version.
Scrypt's tests a taken from an [RFC draft](http://tools.ietf.org/html/draft-josefsson-scrypt-kdf-01).
Besides this, we added comparison runs between ecma-nacl, and js-nacl with js-scrypt, which is an [Emscripten](https://github.com/kripken/emscripten)-compilation of C respective libraries.
These comparison runs can be done in both node and browsers.

## Get ecma-nacl

### NPM Package

This library is registered on
[npmjs.org](https://npmjs.org/package/ecma-nacl). To install it, do:

    npm install ecma-nacl

Package comes with already compiled library code in dist/ folder. For testing, bring up a build environment (see below), and run necessary [gulp](http://gulpjs.com/) tasks.

### Browser Package

[Browserified](http://browserify.org/) version can be found in dist/ folder, and is generated by one of [gulp](http://gulpjs.com/)'s tasks.

### Building

Once you get package, or this repo, do in the folder

    npm install

which will install dev-dependencies.

Building is done with [gulp](http://gulpjs.com/), so it has to also be installed globally, as this is how gulp functions.
From there on do in the folder

    gulp help

and have fun with it.

## ecma-nacl API

### API for secret-key authenticated encryption

Add module into code as
```javascript
var nacl = require('ecma-nacl');
```

[Secret-key authenticated](http://nacl.cr.yp.to/secretbox.html) encryption is provided by secret_box, which implements XSalsa20+Poly1305.

When encrypting, or packing, NaCl does following things. First, it encrypts plain text bytes using XSalas20 algorithm. Secondly, it creates 16 bytes of authentication Poly1305 code, and places these infront of the cipher. Thus, regular byte layout is 16 bytes of Poly1305 code, followed by cipher with actual message, having exactly the same length as plain text message.

Decrypting, or opening goes through these steps in reverse. First, Poly1305 code is read and is compared with code, generated by reading cipher. When these do not match, it means either that key+nonce pair is incorrect, or that cipher with message has been damaged/changed. Our code will throw an exception in such a case. When verification is successful, XSalsa20 will do decryption, producing message bytes.
```javascript
// all incoming and outgoing things are Uint8Array's;
// to encrypt, or pack plain text bytes into cipher bytes, use
var cipher_bytes = nacl.secret_box.pack(plain_bytes, nonce, key);

// decryption, or opening is done by
var result_bytes = nacl.secret_box.open(cipher_bytes, nonce, key);
```

Above pack method will produce an Uint8Array with cipher, offset by 16 zero bytes in the underlying buffer. Deciphered bytes, on the other hand, are offset by 32 zero bytes. This should always be kept in mind, when transferring raw buffers to/from web-workers. In all other places, this padding is never noticed, thanks to [typed array api](https://developer.mozilla.org/en-US/docs/Web/API/Uint8Array).

Key is 32 bytes long. Nonce is 24 bytes. Nonce means number-used-once, i.e. it should be unique for every segment encrypted by the same key.

Sometimes, when storing things, it is convenient to pack cipher together with nonce (WN) into the same array.

    +-------+ +------+ +---------------+
    | nonce | | poly | |  data cipher  |
    +-------+ +------+ +---------------+
    | <----       WN format      ----> |

For this, secret_box has formatWN object, which is used analogously:
```javascript
// encrypting, and placing nonce as first 24 bytes infront NaCl's byte output layout
var cipher_bytes = nacl.secret_box.formatWN.pack(plain_bytes, nonce, key);

// decryption, or opening is done by
var result_bytes = nacl.secret_box.formatWN.open(cipher_bytes, key);

// extraction of nonce from cipher can be done as follows
var extracted_nonce = nacl.secret_box.formatWN.copyNonceFrom(cipher_bytes);
```

Cipher array here has no offset in the buffer, but decrypted array does have the same 32 zero bytes offset, as mentioned above.

It is important to always use different nonce, when encrypting something new with the same key. Object nonce contains functions to advance nonce, to calculate consequent nonces, etc. The 24 bytes of a nnce are taken as three 32-bit integers, and are advanced by 1 (oddly), by 2 (evenly), or by n. So, when encrypting many segments of a huge file, one can advance nonce oddly every time. When key is shared, and is used for communication between two parties, one party's initial nonce may be oddly advanced initial nonce, received from the second party, and all other respective nonces are advanced evenly on both sides of communication. This way, unique nonces are used for every message send.
```javascript
// nonce changed in place oddly
nacl.nonce.advanceOddly(nonce);

// nonce changed in place evenly
nacl.nonce.advanceEvenly(nonce);

// nonce changed in place by delta
nacl.nonce.advance(nonce, delta);

// calculate related nonce
var relatedNonce = nacl.nonce.calculateNonce(nonce, delta, arrayFactory);

// find delta between nonces (null result is for unrelated nonces)
var delta = nacl.nonce.calculateDelta(n1, n2);
```

It is common, that certain code needs to be given encryption/decryption functionality, but according to [principle of least authority](https://en.wikipedia.org/wiki/Principle_of_least_privilege) such code does not necessarily need to know secret key, with which encryption is done. So, one may make an encryptor and a decryptor. These are made to produce and read ciphers with-nonce format.
```javascript
// delta is optional, defaults to one
var encryptor = nacl.secret_box.formatWN.makeEncryptor(key, nextNonce, delta);

// packing bytes is done with
var cipher_bytes = encryptor.pack(plain_bytes);

// when encryptor is no longer needed, key should be properly wiped from memory
encryptor.destroy();

var decryptor = nacl.secret_box.formatWN.makeDecryptor(key);

// opening is done with
var result_bytes = decryptor.open(cipher_bytes);

// when encryptor is no longer needed, key should be properly wiped from memory
decryptor.destroy();
```

### API for public-key authenticated encryption

[Public-key authenticated](http://nacl.cr.yp.to/box.html) encryption is provided by box, which implements Curve25519+XSalsa20+Poly1305. Given pairs of secret-public keys, corresponding shared, in Diffie–Hellman sense, key is calculated (Curve25519) and is used for data encryption with secret_box (XSalsa20+Poly1305).

Given any random secret key, we can generate corresponding public key:
```javascript
var public_key = nacl.box.generate_pubkey(secret_key);
```

Secret key may come from browser's crypto.getRandomValues(array), or be derived from a passphrase with scrypt.

There are two ways to use box. The first way is to always do two things, calculation of DH-shared key and subsequent packing/opening, in one step.
```javascript
// Alice encrypts message for Bob
var cipher_bytes = nacl.box.pack(msg_bytes, nonce, bob_pkey, alice_skey);

// Bob opens the message
var msg_bytes = nacl.box.open(cipher_bytes, nonce, alice_pkey, bob_skey);
```

The second way is to calculate DH-shared key once and use it for packing/opening multiple messages, with box.stream.pack and box.stream.open, which are just nicknames of described above secret_box.pack and secret_box.open.
```javascript
// Alice calculates DH-shared key
var dhshared_key = nacl.box.calc_dhshared_key(bob_pkey, alice_skey);
// Alice encrypts message for Bob
var cipher_bytes = nacl.box.stream.pack(msg_bytes, nonce, dhshared_key);

// Bob calculates DH-shared key
var dhshared_key = nacl.box.calc_dhshared_key(alice_pkey, bob_skey);
// Bob opens the message
var msg_bytes = nacl.box.stream.open(cipher_bytes, nonce, dhshared_key);
```

Or, we may use box encryptors that do first step of DH-shared key calculation only at creation.

Alice's side:
```javascript
// generate nonce, browser example
var nonce = new Uint8Array(24);
crypto.getRandomValues(nonce);

// make encryptor to produce with-nonce format (default delta is two)
var encryptor = nacl.box.formatWN.makeEncryptor(bob_pkey, alice_skey, nonce);

// pack messages to Bob
var cipher_to_send = encryptor.pack(msg_bytes);

// open mesages from Bob
var decryptor = nacl.box.formatWN.makeDecryptor(bob_pkey, alice_skey);
var msg_from_bob = decryptor.open(received_cipher);
    
// when encryptor is no longer needed, key should be properly wiped from memory
encryptor.destroy();
decryptor.destroy();
```

Bob's side:
```javascript
// get nonce from Alice's first message, advance it oddly, and
// use for encryptor, as encryptors on both sides advance nonces evenly
var nonce = nacl.box.formatWN.copyNonceFrom(cipher1_from_alice);
nacl.nonce.advanceOddly(nonce);

// make encryptor to produce with-nonce format (default delta is two)
var encryptor = nacl.box.formatWN.makeEncryptor(alice_pkey, bob_skey, nonce);

// pack messages to Alice
var cipher_to_send = encryptor.pack(msg_bytes);

// open mesages from Alice
var decryptor = nacl.box.formatWN.makeDecryptor(alice_pkey, bob_skey);
var msg_from_alice = encryptor.open(received_cipher);
    
// when encryptor is no longer needed, key should be properly wiped from memory
encryptor.destroy();
decryptor.destroy();
```

### Signing

Code for signing is a re-write of Ed25519 C version from [SUPERCOP's](http://bench.cr.yp.to/supercop.html), referenced in [NaCl](http://nacl.cr.yp.to/sign.html).

signing object contains all related functionality.
```javascript
// signing key pair can be generated from some seed array, which can
// either be random itself, or be generated from a password
var pair = nacl.signing.generate_keypair(seed);

// make signature bytes, for msg
var msgSig = nacl.signing.signature(msg, pair.skey);

// verify signature
var sigIsOK = nacl.signing.verify(msgSig, msg, pair.pkey);
```

There are functions like [NaCl's](http://nacl.cr.yp.to/sign.html) sign and sign_open methods, which place signature and message into one array, and expect the same for opening (verification).
In a context of [JWK](http://self-issued.info/docs/draft-ietf-jose-json-web-key.html), abovementioned functions seem to be more flexible and useful than C's API.

### Random number generation

NaCl does not do it. The randombytes in the original code is a unix shim with the following rational, given in the comment, quote: "it's really stupid that there isn't a syscall for this".

So, you should obtain cryptographically strong random bytes yourself. In node, there is crypto. There is crypto in browser. IE6? IE6 must die! Stop supporting insecure crap! Respect your users, and tell them truth, that they need modern secure browser(s).

### Scrypt - key derivation from passphrases

Scrypt derives a key from users password.
Algorithm is memory-hard, which means it uses lots and lots of memory.
There are three parameters that go into derivation: ``` N```,
``` r``` and
``` p```.

Amount of memory used is roughly ``` 128 * N * r == r * 2^(7+logN)``` bytes.
With ``` r = 8```,
when ``` logN``` is 10, it is a MB range of memory,
when ``` logN``` is 20, it is a GB range of memory in use.

Parameter ``` p``` says how many times should the whole operation occur.
So, when running out of memory (js is not giving enough memory for ``` logN = 20```
), one may up ``` p``` value.

It goes without saying, that such operations take time, and this implementation has a callback for progress reporting.

```javascript
   // given pass (secret), salt and other less-secret parameters
   // key of length keyLen is generated as follows:
   var logN = 17;
   var r = 8;
   var p = 2;
   var key = nacl.scrypt(pass, salt, logN, r, p, keyLen, function(pDone) {
       console.log('derivation progress: ' + pDone + '%');
   }); 
```

Colin Percival's [paper](http://www.tarsnap.com/scrypt/scrypt.pdf) about scrypt makes for a lovely weekend-long reading. 

### XSP file format

Each NaCl's cipher must be read completely, before any plain text output.
Such requirement makes reading big files awkward.
Therefore, we need a file format, which satisfies the following requirements:

 * file should be split into segments;
 * it should be possible to randomly read segments;
 * segment should have poly+cipher, allowing to check segment's integrity;
 * it should be possible to randomly remove and add segments without re-encrypting the whole file;
 * segments' nonces should never be reused, even when performing partial changes to a file, without complete file re-encryption;
 * it should be possible to detect segment reshuffling;
 * there should be cryptographic proof of a complete file size, for files with
known size;
 * file should be encrypted by its own key, encrypted with some external (master) key;
 * there should be a stream-like setting, where segments can be encrypted and read without knowledge of a final length;
 * when complete length of a stream is finally known, switching to known-length setting should be cheap.

We call this format XSP, to stand for XSalsa+Poly, to indicate that file layout
is specifically tailored for storing NaCl's secret box's ciphers.
XSP file has segments, which are NaCl's packs of Poly and XSalsa cipher, and a header.
Header is stored at the end of the file, when segments and header are stored in a single file object.
Sometimes, XSP segments and a header can be stored in separate files.

Complete XSP file looks as following (starting with UTF-8 'xsp'):

    +-----+ +---------------+ +----------+ +--------+
    | xsp | | header offset | | segments | | header |
    +-----+ +---------------+ +----------+ +--------+

When file parts are stored separately, header file starts with UTF-8 'hxsp', and segments' file starts with UTF-8 'sxsp'.

Header can be of two types, distinguished by header's size.

#### Endless file

(72+65)-byte header is used for a file with unknown length (endless file), and looks as follows:

    |<-  72 bytes ->| |<-          65 bytes          ->|
    +---------------+ +--------------------------------+
    |   file key    | | seg size | first segment nonce |
    +---------------+ +--------------------------------+
    |<- WN format ->| |<-          WN format         ->|

 * WN pack of a file key is encrypted with master key. Everything else is encrypted with the file key.
 * Seg size is a single byte. Its value, times 256, gives segment's length. This sets a shortest segment to be 256 bytes, and sets a longest one to be 65280 bytes.
 * All segments have given size except for the last segment, which may be shorter.
 * Nonces for segments related to the initial nonce through delta, equal to segment's index, which starts from 0 for the first segment.

Let's define a segment chain to be a series of segments, encrypted with related nonces.
Chain requires three parameters for a comfortable use:

 * first segment's nonce;
 * number of segments in the chain;
 * length of the last segment in the chain, which can be smaller than common segment length.

Endless file has only one segment chain, and both length of the last segment, and a number of segments cannot be known. 

#### Finite file

(72+46+n*30)-byte header is used for a file with known length (finite file).
n is a number of segment chains in the file.
Header layout:

    |<-  72 bytes ->| |<-              46+n*30 bytes                ->|
    +---------------+ +-----------------------------------------------+
    |   file key    | | total segs length | | seg size | | seg chains |
    +---------------+ +-----------------------------------------------+
    |<- WN format ->| |<-                WN format                  ->|

 * Total segments length, is just that. It excludes header and other file elements.
 * Total segments length is encoded little-endian way into 5 bytes, allowing up to 2^40-1 bytes, or 1TB - 1byte.
 * Content length is equal to total segments length minus 16*s, where s is a total number of segments (16 bytes is poly's code length).

Segment chain bytes look as following:

    |<-   4 bytes  ->| |<-  2 bytes  ->| |<-     24 bytes    ->|
    +----------------+ +---------------+ +---------------------+
    | number of segs | | last seg size | | first segment nonce |
    +----------------+ +---------------+ +---------------------+


#### API for packing/opening XSP segments 

There is a sub-module, with XSP-related functionality:
```javascript
var xsp = nacl.fileXSP;
```

Packing segments and header:  
```javascript
// new file writer needs segment size and a function to get random bytes
var writer = xsp.segments.makeNewWriter(segSizein256bs, getRandom);

// file has its own key, which is encrypted by a master key encryptor
var masterKeyEncr = nacl.secret_box.formatWN.makeEncryptor(masterKey, someNonce);

// header is produced by
var header = writer.packHeader(masterKeyEncr);

// segments are packed with
var sInfo = writer.packSeg(content, segInd);
// where sInfo.dataLen is a number of content bytes packed,
// and sInfo.seg is an array with segment bytes.

// initial endless file can be set to be finite, this changes header information
writer.setContentLength(contentLen);

// writer of existing file should read existing header with master key's decryptor
var masterKeyDecr = nacl.secret_box.formatWN.makeDecryptor(masterKey);
var writer = xsp.segments.makeWriter(header, masterKeyDecr, getRandom);

// writer should be destroyed, when no longer needed
writer.destroy();
```

Currently, at version 2.0.0, efficient splicing functionality is not implemented, but it shall exist, as file format allows for it.
We also plan to add some fool-proof restrictions into implementation to disallow packing the same segment twice.
For now, users are advised to be careful, and to pack each segment only once.

Reader is used for reading:
```javascript
var reader = xsp.segments.makeReader(header, masterKeyDecr);

var dInfo = reader.openSeg(seg, segInd);
// where dInfo.segLen is a number of segment files read,
// where dInfo.last is true for the last segment of this file,
// and dInfo.data is an array with data, read from the segment.

// reader should be destroyed, when no longer needed
reader.destroy();
```


## License

This code is provided here under [Mozilla Public License Version 2.0](https://www.mozilla.org/MPL/2.0/).

NaCl C library is public domain code by Daniel J. Bernstein and others crypto gods and semi-gods. We thank thy wisdom of giving us developer-friendly library.

Scrypt C library is created by Colin Percival. We thank thee for bringing us strong new ideas in key derivation tasks.

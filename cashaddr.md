# Address format for Bitcoin Cash

Version 1.0, 2017-10-13

## Abstract

This document describes a proposed address format to be used on Bitcoin Cash. It is a base32 encoded format using BCH[[1]](#bch) codes as checksum and that can be used directly in links or QR codes.

This format reuses the work done for Bech32[[2]](#bip173) and is similar in some aspects, but improves on others.

## Specification

The address is composed of
1. A prefix indicating the network on which this address is valid.
2. A separator, always `:`
3. A base32 encoded payload indicating the destination of the address and containing a checksum.

### Prefix

The prefix indicates the network on which this addess is valid. It is set to `bitcoincash` for Bitcoin Cash main net, `xbctest` for bitcoin cash testnet and `xbcreg` for bitcoin cash regtest.

The prefix is followed by the separator `:`.

When presented to users, the prefix may be omitted as it is part of the checksum computation. The checksum ensures that addresses on different networks will remain incompatible, even in the absence of an explicit prefix.

### Payload

The payload is a base32 encoded stream of data.

|     | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
| --: | - | - | - | - | - | - | - | - |
|  +0 | q | p | z | r | y | 9 | x | 8 |
|  +8 | g | f | 2 | t | v | d | w | 0 |
| +16 | s | 3 | j | n | 5 | 4 | k | h |
| +24 | c | e | 6 | m | u | a | 7 | l |

The payload is composed of 3 elements:
1. A version byte indicating the type of address.
2. A hash.
3. A 40 bits checksum.

#### Version byte

The version byte's most signficant bit is reserved and must be 0. The 4 next bits indicate the type of address and the 3 least significant bits indicate the size of the hash.

| Size bits | Hash size in bits |
| --------: | ----------------: |
|         0 |               160 |
|         1 |               192 |
|         2 |               224 |
|         3 |               256 |
|         4 |               320 |
|         5 |               384 |
|         6 |               448 |
|         7 |               512 |

Encoding the size of the hash in the version field ensure that it is possible to check that the length of the address is correct.

| Type bits |      Meaning      | Version byte value |
| --------: | :---------------: | -----------------: |
|         0 |       P2KH        |                  0 |
|         1 |       P2SH        |                  8 |

Further types will be added as news features are added.

#### Hash

The hash part really deserves not much explanation as its meaning is dependent on the version field. It is the hash that represents the data, namely a hash of the public key for P2KH and the hash of the reedemScript for P2SH.

#### Checksum

The checksum is a 40 bits BCH codes defined over GF(2^5). It ensures the detection of up to 6 errors in the address and 8 in a row. Combined with the length check, this provides very strong guarantee against errors.

The checksum is computed per the code below:

````cpp
uint64_t PolyMod(const data &v) {
    uint64_t c = 1;
    for (uint8_t d : v) {
        uint8_t c0 = c >> 35;
        c = ((c & 0x07ffffffff) << 5) ^ d;
        
        if (c & 0x01) c ^= 0x98f2bc8e61;
        if (c & 0x02) c ^= 0x79b76d99e2;
        if (c & 0x04) c ^= 0xf33e5fb3c4;
        if (c & 0x08) c ^= 0xae2eabe2a8;
        if (c & 0x10) c ^= 0x1e4f43e470;
    }
    
    return c ^ 1;
}
````

The checksum is calculated over the following data:
1. The lower 5 bits of each characters of the prefix.
2. A zero for the separator (5 zero bits).
3. The payload by chunks of 5 bits. The payload is padded with zero bits up to the point where it is a multiple of 5 bits.

The following adresses can be used as test vector for checksum computation:
 - prefix:x64nx6hz
 - p:gpf8m4h7
 - bitcoincash:qpzry9x8gf2tvdw0s3jn54khce6mua7lcw20ayyn
 - xbctest:testnetaddressa4dxsgzr
 - xbcreg:555555555555555555555555555555555555555555555n5nuyrz8

NB: These addresses do not have valid payload on purpose.

## Error correction

BCH codes allows for error correction. However, it is strongly advised that error correction is not done in an automatic manner as it may cause funds to be lost irrecoverably if done incorrectly. It may however be used to hint a user at a possible error.

## Uppercase/lowercase

Lower case is preferred for cashaddr, but uppercase is accepted. A mixture of lower case and uppercase must be rejected.

Allowing for uppercase ensure that the address can be encoded efficiently in QR codes using the alphanumeric mode[[3]](#alphanumqr).

## Double prefix

In some contexts, such as payment URLs or QR codes, the addresses are currently prefixed with `bitcoincash:`. In these contexts, the address must not be double prefixed.

## References

<a name="bch">[1]</a> https://en.wikipedia.org/wiki/BCH_code

<a name="bip173">[2]</a> https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki

<a name="alphanumqr">[3]</a> http://www.thonky.com/qr-code-tutorial/alphanumeric-mode-encoding

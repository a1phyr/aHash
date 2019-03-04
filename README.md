# aHash

aHash is a high speed keyed hashing algorithm intended for use in in-memory hashmaps. It provides a very quality 64 bit hash.
aHash is designed for performance and is *not cryptographically secure*.

When it is available aHash takes advantage of the [hardware AES instruction](https://en.wikipedia.org/wiki/AES_instruction_set)
on X86 processors. If it is not available it falls back on a slightly lower quality algorithm based on a xor a rotation and a 64bit multiply. 

Similar to Sip_hash it is a keyed hash, so two hashers initialized with different keys will produce completely different
hashes and the resulting hashes cannot be predicted without knowing the keys. 
This prevents DOS attackes where an attacker sends a large number of items that get used as keys in a hashmap and the hashes collide.

## Speed

When it is available aHash uses two rounds of AES encryption using the AES-NI instruction per 16 bytes of input.
On an intel i5-6200u this is as fast as a 64 bit multiplication, 
but it has the advantages of being a much stronger permutation. It also handles 16 bytes at a time. This is obviously
much faster than most standard approaches to hashing, and does a much better job of scrambling data than most non-secure hashes.

On an intel i5-6200u compiled with flags `-C opt-level=3 -C target-cpu=native -C codegen-units=1 -C llvm-args=-unroll-threshold=1000`:

| Input   | SipHash 3-1 time | FnvHash time|FxHash time| aHash time| aHash Fallback time|
|----------------|-----------|-----------|-----------|-----------|---------------|
| u8             | 12.415 ns | 2.0967 ns | **1.1531 ns** | 1.4853 ns | 1.3422 ns |
| u16            | 13.095 ns | 1.3030 ns | **1.1589 ns** | 1.4858 ns | 1.3329 ns |
| u32            | 12.303 ns | 2.1232 ns | **1.1491 ns** | 1.4871 ns | 1.3244 ns |
| u64            | 14.648 ns | 4.3945 ns | **1.1623 ns** | 1.4874 ns | 1.3242 ns |
| u128           | 17.207 ns | 9.5498 ns | **1.4231 ns** | 1.7187 ns | 1.7170 ns |
| 1 byte string  | 16.042 ns | 1.9192 ns | 2.5481 ns | 2.5548 ns | **1.8353 ns** |
| 3 byte string  | 16.775 ns | 3.5305 ns | 4.5138 ns | 2.9186 ns | **2.3882 ns** |
| 4 byte string  | 15.726 ns | 3.8268 ns | **1.2745 ns** | 2.5415 ns | 2.1907 ns |
| 7 byte string  | 19.970 ns | 5.9849 ns | 3.9006 ns | 3.0936 ns | **2.7905 ns** |
| 8 byte string  | 18.103 ns | 4.5923 ns | **2.2808 ns** | 2.5501 ns | 2.9154 ns |
| 15 byte string | 22.637 ns | 10.361 ns | 6.0990 ns | **3.2825 ns** | 4.1184 ns |
| 16 byte string | 19.882 ns | 9.8525 ns | **2.7562 ns** | 4.0007 ns | 3.4502 ns |
| 24 byte string | 21.893 ns | 16.640 ns | **3.2014 ns** | 4.1262 ns | 4.1600 ns |
| 68 byte string | 33.370 ns | 65.900 ns | 6.4713 ns | **5.9960 ns** | 8.0634 ns |
| 132 byte string| 52.996 ns | 158.34 ns | 14.245 ns | **5.9262 ns** | 16.704 ns |
|1024 byte string| 337.01 ns | 1453.1 ns | 205.60 ns | **52.789 ns** | 209.22 ns |

As you can see above aHash provides the similar (~5x) speeup over SipHash that FxHash provides.

Rust by default uses SipHash because faster hash functions such as FxHash are predictable and vulnerable to denial of service attacks.
aHash has both very strong scrambling (inherited from AES) as well as very high performance, but is architecture speffic.

Similar to FxHash it perfroms well when dealing with large inputs because aHash reads 16 bytes at a time. 
(This is how it is able to out perfrom FxHash with large strings) It also provides the reasonabably good performance when
dealing with unaligned input. (notice the big performance gaps between 3 vs 4, 7 vs 8 and 15 vs 16 above.)

## Security

aHash is designed to prevent keys from being guessable. This means:
- It is a high quality hash that produces results that look highly random.
- It obays the '[strict avalanche criterion](https://en.wikipedia.org/wiki/Avalanche_effect#Strict_avalanche_criterion)': 
Each bit of input can has the potential to flip every bit of the output.
    - Additionally, whether or not it does so depends on every bit in the Key.

This prevents DOS attacks that attempt to produce hash collisions by knowing how the hash wroks.
It is however not recomended to assume this property can hold if the attaker is allowed to SEE the hashed value.
AES is desinged to prevent an attacker from learning the key being used even if they can see the encryped output and 
select the plain text that is used. *However* this property holds when 10 rounds are used. aHash uses only 2 rounds, so 
it likely won't hold up to this sort of attack. For DOS prevention, this should not be a problem, as an attacker trying 
to produce collisions in a hashmap does not get to see the hash values that are beeing used inside of the map.

### aHash is not cryptographically secure

aHash should not be used for situations where cryptographic security is needed for several reasons.

1. It has not been analyzed for vulnerabilities and may leak bits of the key in its output.
2. It only uses 2 rounds of AES as opposed to the standard of 10. This likely makes it possible to guess the key by observing a large number of hashes.
3. Like any cypher based hash, it will show certain statistical deviations from truely random output when comparing a (VERY) large number of hashes.

There are several efforts to build a secure hash function that uses AES-NI for acceleration, but this is not one of them.

## Compatibility

Currently aHash is pre 1.0. So new versions may change the algorithm slightly resulting in the new version producing 
different hashes than the old version even with the same keys. Additionally aHash does not currently guarantee that it 
won't produce different hash values for the same data on different machines. 

## Supported CPUs

Hardware AES instructions are built into Intel processors built after 2010 and AMD processors after 2012.
It is also available on [many other CPUs](https://en.wikipedia.org/wiki/AES_instruction_set) should in eventually
be able to get aHash to work. However only X86 and X86-64 are the only supported architectures at the moment, as currently
they are the only architectures for which Rust provides an intrinsic.
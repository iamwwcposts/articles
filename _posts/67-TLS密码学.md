---
issueid: 67
tags:
- net
title: TLS密码学
date: 2024-06-05
updated: 2024-06-12
---
## 前言
不管引入什么概念，什么算法，密码的核心永远不变
让原文获得密文容易，没有key的帮助反向**解密出原文并正确**很难
各种算法/机制都是在加强这句话

传输安全的目标：在不可靠的信道中建立可靠通讯

传输安全的三个要素：加密+校验+基于证书的身份识别机制，缺一不可
- 真实数据不会被看到：需要加解密
- 数据不被篡改：需要完整性校验
- 确定对端的真实身份：防止中间人攻击

## 对称加密(symmetric cryptography)
加解密使用相同key。密文=\= key XOR 原文，原文=\=密文 XOR key

分类
- 分组密码(block cipher): 加解密以固定块大小为单元，不足一个块需要填充(填充)
    - 熟知的算法
        - AES-CBC
- 流式密码(stream cipher):  可以加解密一个最小单元(比如字节)，像无尽的水流，无须padding数据随来随加/解
    - 熟知的算法
        - RC4
        - AES-GCM
        - AES-CTR
    
问题：
为什么AES既可以当分组密码，又可以当流密码？
对称加密都是block cipher，所以AES是block cipher。但AES有很多模式，使得分组密码可以很容易的转成流式模式使用

> [为什么分组密码可以很容易地转换成流式密码使用？](https://eitca.org/cybersecurity/eitc-is-ccf-classical-cryptography-fundamentals/stream-ciphers/stream-ciphers-random-numbers-and-the-one-time-pad/a-block-cipher-can-be-easily-turned-into-a-stream-cipher-while-the-opposite-is-not-the-case/)
> A block cipher can be indeed easily turned into a stream cipher while the opposite is not the case. This is due to the fundamental differences between block ciphers and stream ciphers, as well as the properties and requirements of each.
> 
> To better understand this problem, let's first define what block ciphers and stream ciphers are. A block cipher is a cryptographic algorithm that operates on fixed-size blocks of data, typically 64 or 128 bits. It encrypts or decrypts these blocks independently, using a fixed key. Examples of block ciphers include the Data Encryption Standard (DES), Advanced Encryption Standard (AES), and Triple Data Encryption Standard (3DES).
> 
> On the other hand, a **stream cipher** is a cryptographic algorithm that encrypts or decrypts data one bit or one byte at a time. **It uses a key and a pseudorandom number generator (PRNG) to generate a stream of bits or bytes, which are then combined with the plaintext or ciphertext using an XOR operation**. The resulting stream is used to encrypt or decrypt the data. Stream ciphers are often used for real-time communication and have applications in wireless networks, satellite communication, and secure voice transmission.
> 
> Now, let's delve into the reasons why a block cipher can be easily turned into a stream cipher, while the opposite is not true. One of the main reasons is that the structure and design of a block cipher inherently allow for the transformation into a stream cipher. **Block ciphers are designed to operate on fixed-size blocks of data, but they can also be used in a mode called "counter mode" or "CTR mode" to generate a stream of key bits or bytes. In this mode, the block cipher is used as a PRNG, where the key and a counter value are input into the block cipher encryption function to generate the stream of key bits or bytes. This stream can then be used as the keystream in a stream cipher.**
> 
> For example, let's consider the AES block cipher. AES operates on 128-bit blocks and supports key sizes of 128, 192, or 256 bits. To turn AES into a stream cipher, we can use it in CTR mode. In CTR mode, we select a nonce (a unique value) and a counter value. We then encrypt the nonce concatenated with the counter using AES, and the resulting ciphertext is used as the keystream. The keystream is XORed with the plaintext to produce the ciphertext, and vice versa for decryption. By incrementing the counter value for each block of plaintext or ciphertext, we can generate a stream of key bits or bytes.
> 
> On the other hand, the opposite transformation, turning a stream cipher into a block cipher, is not straightforward. Stream ciphers are designed to operate on individual bits or bytes, and their encryption and decryption processes are based on the assumption of a continuous stream of data. Block ciphers, on the other hand, require fixed-size blocks of data and operate on these blocks independently. The block cipher encryption and decryption functions are not designed to handle individual bits or bytes.
> 
> If we were to try to turn a stream cipher into a block cipher, we would need to define a fixed block size and determine how to handle the encryption and decryption of individual bits or bytes within the block. This would require significant modifications to the stream cipher algorithm, potentially compromising its security and efficiency.
> 
> A block cipher can be easily turned into a stream cipher by using it in counter mode, where the block cipher is used as a PRNG to generate a stream of key bits or bytes. However, the opposite transformation, turning a stream cipher into a block cipher, is not straightforward due to the inherent differences in their design and operation.

理解：计数器模式下，把AES当称伪随机数生成器，
1. 输入 key + nonce + count 获得加密后的块以满足原文长度
2. 如果块长度不够，提升count的值并重复 step 1，源源不断生成加密块
3. 用密文与原文进行XOR运算，就能得到最后的密文

## 非对称加密(asymmetric cryptography)
又称 公钥加密(Public-key cryptography)，看名称就知道，加解密由公私钥组成。
公钥加密的，只能私钥解开获得真正原文。私钥加密的，只能公钥解开获得原文

具体实现
- Diffie–Hellman key exchange protocol （DH）
- DSS (Digital Signature Standard), which incorporates the Digital Signature Algorithm
- Elliptic-curve cryptography（ECC）
    - Elliptic Curve Digital Signature Algorithm (ECDSA)
    - Elliptic-curve Diffie–Hellman (ECDH)
    - Ed25519 and Ed448 (EdDSA)
    - X25519 and X448 (ECDH/EdDH)
- RSA encryption algorithm (PKCS#1)

## 数字签名算法(Digital Signature Algorithm/DSA)
DSA首先是个公钥密码学体系，然后定义了一些算法套件实现数字签名能力
数字签名有签名和验签，所以DSA必须非对称加密和hash算法配合使用

配对使用方式
1. 非对称算法（提供public/private key pair）可选
- ECDSA
- RSA
2. hash算法：提供验签能力
- sha-1
- sha-2

注：数字签名中hash算法是必须的

签名需要的材料
- private key
- 原文
- hash 算法（sha-1, sha256, shaxxx）

```js
const {
  generateKeyPairSync,
  createSign,
  createVerify,
} = require('node:crypto');

// 生成key pair
const { privateKey, publicKey } = generateKeyPairSync('ec', {
  namedCurve: 'sect239k1',
});
// 指定hash算法
const sign = createSign('SHA256');
sign.write('some data to sign');
sign.end();
// 使用私钥签名
const signature = sign.sign(privateKey, 'hex');

const verify = createVerify('SHA256');
verify.write('some data to sign');
verify.end();
// 验签
console.log(verify.verify(publicKey, signature, 'hex'));
// Prints: trueCOPY

```

hash then encrypt
encrypt after
encrypt then hash

[rsa - Hash-Then-Encrypt or Encrypt-Then-Hash? - Cryptography Stack Exchange](https://crypto.stackexchange.com/questions/62163/hash-then-encrypt-or-encrypt-then-hash)

sign和encrypt有什么区别？
[rsa - What is the difference between encrypting and signing in asymmetric encryption? - Stack Overflow](https://stackoverflow.com/questions/454048/what-is-the-difference-between-encrypting-and-signing-in-asymmetric-encryption)

参考
>[I'm trying to explain how a digital signature works to my friend and I got stuck... : r/cryptography](https://www.reddit.com/r/cryptography/comments/xvom3z/im_trying_to_explain_how_a_digital_signature/)
> [crypto.createSign(algorithm[, options])](https://nodejs.org/api/crypto.html#cryptocreatesignalgorithm-options:~:text=or%20Hmac.-,crypto.createSign(,-algorithm%5B%2C%20options%5D))

### EdDSA(Edwards-curve Digital Signature Algorithm)签名算法

具体算法
- Ed25519: elliptic curve signing algorithm using EdDSA and Curve25519

RFC
[RFC 8032 - Edwards-Curve Digital Signature Algorithm (EdDSA)](https://datatracker.ietf.org/doc/html/rfc8032)


### RSASSA-PSS(RSA Probabilistic Signature Scheme)
一种数字签名算法
## 密钥协商(key agreement)
重在协商，相互施加影响。大多数不严谨的场景，密钥交换和密钥协商指的是一件事。利用某种机制在不安全的环境协商出只有双方才知道的一个数

## 密钥交换(key exchange)
叫密钥交换而不是密钥交换算法，是因为密钥交换是依赖其他算法一个过程，因此 key exchange 本身并不是算法

密钥交换要解决的问题是，为了进行对称加解密，需要双方有相同的key。为了在不安全的信道协商出key，需要先引入非对称加密算法。

key exchange一定基于公私钥加密，也就是非对称加密。`RSA`, `ECC椭圆曲线` 是实现上述公私钥加密的数学实现之一。

密钥交换算法

- DH key exchange 家族
- RSA

**密钥交换算法可以单独使用。在一些需要消息完整性（TLS场景）的场景，key exchange算法必须配合签名算法**
因此TLS cipher suite经常会有
> TLS\_**ECDHE\_ECDSA**\_WITH_AES_256_CBC_SHA384
> TLS\_**ECDHE\_ECDSA**\_WITH_AES_128_CBC_SHA256
> TLS\_**ECDHE\_ECDSA**\_WITH_AES_256_CBC_SHA384
> TLS\_**ECDHE\_RSA**\_WITH_AES_128_GCM_SHA256
> TLS\_**ECDHE\_RSA**\_WITH_AES_256_GCM_SHA384
> TLS\_**ECDHE\_RSA**\_WITH_AES_128_CBC_SHA256
> TLS\_**ECDHE\_RSA**\_WITH_AES_256_CBC_SHA384
> TLS\_**ECDHE\_RSA**\_WITH_AES_128_CBC_SHA256
> TLS\_**ECDHE\_RSA**\_WITH_AES_256_CBC_SHA384
> TLS\_**DHE\_RSA**\_WITH_AES_128_GCM_SHA256

### DH(Diffie–Hellman key exchange) 家族
> [Diffie–Hellman key exchange - Wikipedia](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)
DH是密钥交换的一种具体交换流程，利用已有公钥加密算法而实现的一种交换形式，因此都是非对称加密
![](/assets/2024-05-31.png)

#### DHE(Diffie–Hellman ephemeral)
DH的升级版，E表明key是当时现场生成的，不会横跨现在和未来多个会话使用

TLS中配对使用的方式

- DHE-RSA: DHE key exchange 并配合 RSA 签名算法使用
- DHE-DSA: DHE key exchange 并配合 RSA 

#### ECDH(Elliptic-curve Diffie–Hellman)

通常用于密钥协商。基于椭圆曲线实现的DH密钥交换形式，不仅仅是加密过程，还定义了key的交换方式。
可以这么理解，如果说DH算法定义的公钥生成规则是 `g^(私钥) mod p = 公钥`
那ECDH使用椭圆曲线替代了上述计算规则，计算方式变为在椭圆曲线上求解，相比RSA，ECC所用的密钥更短，运行速度更快

> ECC is more secure than RSA and is in its adaptive phase. Its usage is expected to scale up in the near future. RSA requires much bigger key lengths to implement encryption. ECC requires much shorter key lengths compared to RSA.

#### ECDHE(Elliptic-curve Diffie–Hellman ephemeral)
比ECDH多了E(短暂的，ephemeral)。生成的key是短暂的很快失效，不再具备生成的session key解密之前数据的能力。具有前向安全性。
因此https场景中，key是private key，而session key是key exchange之后的用于对称加解密的key。哪怕private key泄漏，之前的数据也无法被解密

配对使用方式
- ECDHE-RSA: ECDHE key exchange 并配合 RSA签名算法使用
- ECDHE-ECDSA: ECDHE key exchange 并配合 ECDSA 签名算法使用

如果以ECDHE进行，serverhello选择的TLS1.2套件类似 `TLS_ECDHE_RSA_WITH_RSA_128_GCM_SHA256`
server key exchange 会为ECDHE提供ECDH 参数
![](/assets/2024-06-03-4.png)
## AEAD(加解密的同时能保证消息完整性/authenticated encryption with associated data)
AE（authenticated Encryption），AD（associated data）

使用场景如下
```
[encrypted-data][encrypted-data's MAC][destination address]
|<-------AE(都是加密的)--------------->|<-----AD(明文)------>|
```
其实AE就够了，但某些场景比如网络数据包，destination address不能加密，否则中间人无法路由，只能暴露，但需要加完整性检查

AEAD具体算法实现
- AES_128_GCM
- AES_256_GCM

## HASH算法(SHA-XXX)
SHA-1，SHA-2家族

SHA384是SHA-2长度为384bit的hash算法
## MAC(message authentication code)
MAC是一个概念，用来检验消息完成性的一小段数据，类似tcp的checksum。

可以用块加密算法实现，比如AES-GCM。根据原文生成一段加密后的数据，其他人也可以加密，并验证

而用hash函数(SHA-1，SHA-2)实现MAC的方式称为 HMAC
### HMAC(hash-based message authentication code)
和数字签名很像，hash时引入secret生成digest
数字签名依赖非对称加密的公私钥，为了能安全分发公钥，需要PKI(public key infrastructure)体系保驾护航
而HMAC用的secret双方都知道

## KDF( 密钥派生函数/key derivation function)

目的：有时输入的key密码强度太低，比如 `helloworld`，无法满足很多加密算法的密钥长度和强度要求，需要一种机制将输入的key进行计算，获得密码学意义上的高强度key

### HKDF(HMAC-based Extract-and-Expand Key Derivation Function)
RFC全文不长，可以通读
[RFC 5869 - HMAC-based Extract-and-Expand Key Derivation Function (HKDF)](https://datatracker.ietf.org/doc/html/rfc5869)

TLS HKDF的H是 ciphersuite 最后指定的hash算法

## TLS1.3-pre-shared  key extension(PSK)
![](/assets/2024-05-31-2.png)
这就是常说的TLS 1.3 0RTT握手。client携带PKS尝试与server恢复会话，如果server可以恢复，serverhello也会携带PSK extension，指定选中的key。接下来正常收发数据

根据RFC，如果client 有 PSK extension，那也必须有psk-key-exchange-modes
![](/assets/2024-06-01-1.png)
> A **client MUST provide a "psk_key_exchange_modes" extension if it
   offers a "pre_shared_key" extension**.  If clients offer
   "pre_shared_key" without a "psk_key_exchange_modes" extension,
   servers MUST abort the handshake.  Servers MUST NOT select a key
   exchange mode that is not listed by the client.  This extension also
   restricts the modes for use with PSK resumption.  Servers SHOULD NOT
   send NewSessionTicket with tickets that are not compatible with the
   advertised modes; however, if a server does so, the impact will just
   be that the client's attempts at resumption fail.
[TLS-1.3 pre-shared key](https://datatracker.ietf.org/doc/html/rfc8446#page-55)
## session id-session ticket-psk区别
session id需要server维护状态
session ticket将关键数据加密发给client，client请求携带session ticket extension进行连接复用
TLS1.3废弃了session ticket，并引入psk。
[Resumption and Pre-Shared Key (PSK)](https://datatracker.ietf.org/doc/html/rfc8446#section-2.2)
>  This document supersedes and obsoletes previous versions of TLS,
>  including version 1.2 [RFC5246].  **It also obsoletes the TLS ticket
>  mechanism defined in [RFC5077] and replaces it with the mechanism
>  defined in Section 2.2.**  Because TLS 1.3 changes the way keys are
>  derived, it updates [RFC5705] as described in Section 7.5.  It also
>  changes how Online Certificate Status Protocol (OCSP) messages are
>  carried and therefore updates [RFC6066] and obsoletes [RFC6961] as
>  described in Section 4.4.2.1.
>  Although TLS PSKs can be established out of band, PSKs can also be
>  established in a previous connection and then used to establish a new
>  connection ("session resumption" or "resuming" with a PSK).  Once a
>  handshake has completed, the server can send the client a PSK
>  identity that corresponds to a unique key derived from the initial
>  handshake (see Section 4.6.1).  The client can then use that PSK
>  identity in future handshakes to negotiate the use of the associated
>  PSK.  If the server accepts the PSK, then the security context of the
>  new connection is cryptographically tied to the original connection
>  and the key derived from the initial handshake is used to bootstrap
>  the cryptographic state instead of a full handshake.  **In TLS 1.2 and
>  below, this functionality was provided by "session IDs" and "session
>  tickets" [RFC5077].  Both mechanisms are obsoleted in TLS 1.3.**

## TLS1.3-key share extension

DH虽然只有1RTT，但传递的数据还是用public key加密了。具体实现是 tls1.3的key share extension
TLS1.3主体保持与1.2的兼容，如果最终选定TLS1.3握手，key share则必须
client hello阶段client通过key share extension发送很多类型的public key
## TLS1.2-premaster secret
- 如果TLS key exchange 以RSA算法进行，那premaster secret 是client exchange 阶段以server public key加密的 48字节数据
[tls 1.2 的第三个随机数](https://datatracker.ietf.org/doc/html/rfc5246#section-8.1:~:text=byte%20pre_master_secret%20is%20generated%20by%20the%20client)
[RSA key exchange逐渐弃用，没有前向安全性，建议用DHE](https://datatracker.ietf.org/doc/html/rfc5246#appendix-F.1.1.2)
- 如果以DH密钥交换方式进行，premaster secret是通过DH 1RTT交换双方public后推导出的数据（server hello public key + client key exchange public key）
![server key exchange public key](/assets/2024-06-03-2.png)
![client key exchange public key](/assets/2024-06-03.png)
## master secret

```
master_secret = PRF(pre_master_secret, "master secret",
                  ClientHello.random + ServerHello.random)
                  [0..47];
```

[master secret 用以生成TLS传输过程中各种key](https://datatracker.ietf.org/doc/html/rfc5246#section-6.3:~:text=The%20master%20secret%20is%20expanded%20into%20a%20sequence%20of%20secure%20bytes)
- client write MAC key
- server write MAC key
- client write encryption key
- server write encryption key

>  To generate the key material, compute
> 
>       key_block = PRF(SecurityParameters.master_secret,
>                       "key expansion",
>                       SecurityParameters.server_random +
>                       SecurityParameters.client_random);
> 
>    until enough output has been generated.  Then, the key_block is
>    partitioned as follows:
> 
>       client_write_MAC_key[SecurityParameters.mac_key_length]
>       server_write_MAC_key[SecurityParameters.mac_key_length]
>       client_write_key[SecurityParameters.enc_key_length]
>       server_write_key[SecurityParameters.enc_key_length]
>       client_write_IV[SecurityParameters.fixed_iv_length]
>       server_write_IV[SecurityParameters.fixed_iv_length]
> 

以上这些key都依据master secret key生成，又称为 session key。

生成过程依赖PRF（类似HKDF）
[RFC 5246 - The Transport Layer Security (TLS) Protocol Version 1.2](https://datatracker.ietf.org/doc/html/rfc5246#section-8.1:~:text=To%20generate%20the%20key%20material%2C%20compute)
### session key
session key 是会话中用到的，根据KDF生成的各种会话key
![](/assets/2024-05-31-3.png)

## FAQ

### TLS 1.2 和 TLS 1.3 从cipher suites上有什么不同

tls1.2的一个算法 TLS_RSA_WITH_AES_256_CBC_SHA
tls1.3 的一个算法 TLS_AES_128_GCM_SHA256
### TLS1.3的cipher suite为什么比TLS1.2的一些短？

TLS1.2
```
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
TLS_DHE_RSA_WITH_AES_128_GCM_SHA256
TLS_DHE_RSA_WITH_AES_256_GCM_SHA384
TLS_DHE_RSA_WITH_AES_128_CBC_SHA
TLS_DHE_RSA_WITH_AES_256_CBC_SHA
TLS_DHE_RSA_WITH_AES_128_CBC_SHA256
TLS_DHE_RSA_WITH_AES_256_CBC_SHA256
TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
```
TLS1.3

```
TLS_AES_256_GCM_SHA384
TLS_CHACHA20_POLY1305_SHA256
TLS_AES_128_GCM_SHA256
TLS_AES_128_CCM_8_SHA256
TLS_AES_128_CCM_SHA256
```

拿 1.3 的 `TLS_AES_256_GCM_SHA384` 和tls1.2的 `TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384`
![](/assets/2024-05-31-1.png)
cipher suite比1.2少了密钥交换和签名算法
是因为signature algorithm放到了extension，支持多个，server选其一
[cipher selection - How are key exchange and signature algorithms negotiated in TLS 1.3 - Information Security Stack Exchange](https://security.stackexchange.com/questions/257776/how-are-key-exchange-and-signature-algorithms-negotiated-in-tls-1-3)
### tls1.2
#### 为什么tls1.2的套件的key exchange 和 signature 用了两个不同的有两个非对称算法，
![](/assets/2024-05-31-1.png)

#### TLS1.2弃用RSA作为key exchange

拿现在流行的TLS1.2 ciphersuite 举例
[Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=intermediate&openssl=1.1.1k&guideline=5.7)
```
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
TLS_DHE_RSA_WITH_AES_128_GCM_SHA256
TLS_DHE_RSA_WITH_AES_256_GCM_SHA384
TLS_DHE_RSA_WITH_AES_128_CBC_SHA
TLS_DHE_RSA_WITH_AES_256_CBC_SHA
TLS_DHE_RSA_WITH_AES_128_CBC_SHA256
TLS_DHE_RSA_WITH_AES_256_CBC_SHA256
TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
```
可以看到 `TLS_FOO` 的 FOO 都是末尾都是`DHE`，没有 `TLS_RSA_WITH_AES_256_CBC_SHA` 这些套件了。
RSA作为key exchange算法没有前向安全性，如果泄露私钥，之前的会话都能解密。因此大家都弃用了RSA作为密钥交换算法，转而使用前向安全的DHE
[reddit.com/r/networking/comments/16vcnh4/chrome\_says\_rsa\_key\_exchange\_is\_obsolete\_under/](https://www.reddit.com/r/networking/comments/16vcnh4/chrome_says_rsa_key_exchange_is_obsolete_under/)
[Deprecating Obsolete Key Exchange Methods in TLS 1.2](https://www.ietf.org/archive/id/draft-ietf-tls-deprecate-obsolete-kex-02.html#name-rsa)
### TLS 1.3
#### TLS1.3如何防止中间人攻击
#### TLS1.3 key exchange 步骤

> In the Key Exchange phase, the client sends the ClientHello
> (Section 4.1.2) message, which contains a random nonce
> (ClientHello.random); its offered protocol versions; a list of
> symmetric cipher/HKDF hash pairs; either a set of Diffie-Hellman key
> shares (in the "key_share" (Section 4.2.8) extension), a set of
> pre-shared key labels (in the "pre_shared_key" (Section 4.2.11)
> extension), or both; and potentially additional extensions.
> Additional fields and/or messages may also be present for middlebox
> compatibility.
> The server processes the ClientHello and determines the appropriate
> cryptographic parameters for the connection.  It then responds with
> its own ServerHello (Section 4.1.3), which indicates the negotiated
> connection parameters.  The combination of the ClientHello and the
> ServerHello determines the shared keys.  **If (EC)DHE key establishment
> is in use, then the ServerHello contains a "key_share" extension with
> the server's ephemeral Diffie-Hellman share; the server's share MUST
> be in the same group as one of the client's shares.  If PSK key
> establishment is in use, then the ServerHello contains a
> "pre_shared_key" extension indicating which of the client's offered
> PSKs was selected.**  Note that implementations can use (EC)DHE and PSK
> together, in which case both extensions will be supplied.

![client hello key share extension](/assets/2024-06-01.png)

![server hello 选择client key share其一](/assets/2024-06-02-98.png)
#### 为什么TLS1.3会有Client.random和Server.random
client并不知道server是否支持tls 1.3，为了兼容且正常握手，client会按照1.2的要求进行握手。并提供1.3需要的key share extension。使得clienthello同时支持1.2/1.3协商。

serverhello决定最后采用多少版本握手。如果1.2，则random末尾8字节用固定数据。是防止1.2/1.3的client/server被要求降低版本到1.1及以下导致降级攻击

>  TLS 1.3 has a **downgrade protection mechanism** embedded in the server's
>    random value.  TLS 1.3 servers which negotiate TLS 1.2 or below in
>    response to a ClientHello MUST set the last 8 bytes of their Random
>    value specially in their ServerHello.
> 
>    If negotiating TLS 1.2, TLS 1.3 servers MUST set the last 8 bytes of
>    their Random value to the bytes:
> 
>      44 4F 57 4E 47 52 44 01
> 
>    If negotiating TLS 1.1 or below, TLS 1.3 servers MUST, and TLS 1.2
>    servers SHOULD, set the last 8 bytes of their ServerHello.Random
>    value to the bytes:
> 
>      44 4F 57 4E 47 52 44 00
> 
>    **TLS 1.3 clients receiving a ServerHello indicating TLS 1.2 or below
>    MUST check that the last 8 bytes are not equal to either of these
>    values. TLS 1.2 clients SHOULD also check that the last 8 bytes are
>    not equal to the second value if the ServerHello indicates TLS 1.1 or
>    below.  If a match is found, the client MUST abort the handshake** with
>    an "illegal_parameter" alert.  This mechanism provides limited
>    protection against downgrade attacks over and above what is provided
>    by the Finished exchange: because the ServerKeyExchange, a message
>    present in TLS 1.2 and below, includes a signature over both random
>    values, it is not possible for an active attacker to modify the
> 
> 
> 
> Rescorla                     Standards Track                   [Page 32]
> 
> RFC 8446                           TLS                       August 2018
> 
> 
>    random values without detection as long as ephemeral ciphers are
>    used.  It does not provide downgrade protection when static RSA
>    is used.
#### TLS1.3 supported-group extension
client hello `supported group` extension 给出支持的 key exchange 椭圆曲线方程，比如curve-25519曲线
`y2 = x3 + 486662x2 + x`
![](/assets/2024-06-02-97.png)
![](/assets/2024-06-02.png)
并通过key share给出不同曲线方程用到的函数参数
![](/assets/2024-06-02-2.png)
#### 证书里的public key有什么用？
1.2握手第三步骤，client的pre master secret通过server certificate 的public key加密传给server。这叫公钥加密，私钥解密
1.3握手1RTT，server hello之后的内容就被加密了，所以server早就协商出了密钥。client收到证书public key还有什么用呢？
TLS1.3引入 CertificateVerify message，在certificate之后发送，CertificateVerify 会包含 `certificate private key+之前所发全部内容的hash` 做的签名，client用证书里的public key验签，来确保之前的消息都未被修改
#### TLS废弃RSA做kx，证书还有什么用？
这涉及到rsa证书和ecc证书区别是什么
rsa和ecc都属于非对称（公钥）加密算法，rsa证书采用rsa做非对称加密加密。ecc采用椭圆曲线做非对称加密

TLS1.2 RSA kx涉及到pre-master-secret加密，server用私钥解密。此时证书有kx和身份验证能力
废弃RSA kx后，TLS1.2，TLS1.3使用RSA，只使用的RSA的身份验证能力。算个数（或者handshake transcript）用证书的public key加密，或者private key签名，对端用private key解密，或者public key验签名
## 握手涉及到加解密/签名的地方
TLS1.3 CertificateVerify
## TLS1.3握手详解
TODO 必读，然后总结出来
- [The Illustrated TLS 1.3 Connection: Every Byte Explained](https://tls13.xargs.org/#client-key-exchange-generation)
- [cipher selection - How are key exchange and signature algorithms negotiated in TLS 1.3 - Information Security Stack Exchange](https://security.stackexchange.com/questions/257776/how-are-key-exchange-and-signature-algorithms-negotiated-in-tls-1-3)

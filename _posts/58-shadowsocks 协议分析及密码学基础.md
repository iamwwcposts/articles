---
date: 2022-05-24
updated: 2022-05-25
issueid: 58
tags:
- net
title: shadowsocks 协议分析及密码学基础
---
# shadowsocks 协议分析

## 基础知识

### nonce 和 salt 区别

salt 只用一次，用于生成 cipher，和 salt 类似的概念叫initial vector（初始化向量），可认为 salt  和 iv 是等价的概念。

nonce 每次 encrypt，decrypt都需要，而且 decrypt 时使用的nonce必须和那次encrypt的nonce一致。由于加解密的nonce必须一一对应，所以nonce往往采用双方提前约定的生成规则，一般都是每次使用完自加1，为下次使用做准备

### 信息完整性校验

针对 AEAD 加密，密文加密生成 HMAC tag。解密需提供 校验tag 用于数据完整性校验。所以加密方负责将 tag 传给解密方。一般来说，tag会附加到密文之后。

## shadowsocks 协议格式

解密的首个数据包头会携带 salt 用于接下来计算解密key

```
[salt][encrypted length][tag][encrypted payload][tag]
```

之后的数据流格式为

```
[encrypted length][tag][encrypted payload][tag]
```

由于key创建时需 salt，所以 shadowsocks-server 的 decryptor 的 key 是在收到首次数据包后才能创建出来的

加解密的函数式表达

psk 的推导需要 user_password，核心是OpenSSL的 [EVP_BytesToKey](https://www.openssl.org/docs/man1.0.2/man3/EVP_BytesToKey.html)

我贴下原文，D_i 代表第 i 轮，每一轮的计算为：`[上次计算的结果 + data + salt]`，直到 digest 之后的结果符合key长度要求

```
KEY DERIVATION ALGORITHM
The key and IV is derived by concatenating D_1, D_2, etc until enough data is available for the key and IV. D_i is defined as:

        D_i = HASH^count(D_(i-1) || data || salt)
where || denotes concatentaion, D_0 is empty, HASH is the digest algorithm in use, HASH^1(data) is simply HASH(data), HASH^2(data) is HASH(HASH(data)) and so on.
```

以下是其他语言针对 psk 计算的实现

```
https://github.com/shadowsocks/shadowsocks/blob/master/shadowsocks/cryptor.py#L54
https://github.com/shadowsocks/shadowsocks-org/issues/98
```

```

// key_length 为加密算法要求的key length
// 比如 AES-128-GCM 要求 key length 128 bits，即 16 字节, derive_pre_shared_key 必须保证 psk.len() == 16
// psk length 与 key length 相等
derive_pre_shared_key(user_password， key_length) => psk
derive_key(psk, salt) => key

// message 为原文
// tag 为完整性检验数据
AE_Encrypt(key, message, nonce) => (ciphertext, tag)

// ciphertext 密文
AE_Decrypt(key, ciphertext, nonce, tag) => message
```

对于 shadowsocks#inbound, 第一个 encrypted payload 是目标地址，之后的则是 要转发的数据

```
[target_address][payload length][payload]
```

对于 shadowsocks#outbound

首次数据包构造中，第一个 encrypted payload 必须是 `target_address`，之后则是要转发的数据

加解密只是真实的传递双方原始的数据，针对 shadowsocks#inbound

我们设定 EncryptedChunk 格式如下

```
[encrypted length][tag][encrypted payload][tag]
```

则原始协议对应如下

```
| target_address    | payload length    | payload           |
| ----------------- | ----------------- | ----------------- |
| variable length   | 2bytes            | variable length   |
| ----------------- | ----------------- | ----------------- |
| EncryptedChunk    | EncryptedChunk    | EncryptedChunk    |
| ----------------- | ----------------- | ----------------- |
|                   | <--------------无限重复--------------->|
```

加密后的 shadowsocks 数据流格式为

```
[salt]（[Encrypted Chunk]） * n
```

target_address 为 socks5的格式，定义如下

| ATYP  | Target          | Port   |
| ----- | --------------- | ------ |
| 1byte | variable length | 2bytes |

shadowsocks-client => shadowsocks-server\
shadowsocks-client <= shadowsocks-server
连接都以 salt 开头

client => server 在 salt 之后的第一个 EncryptedChunk 为 target_address

server => client 在 salt 之后的 EncryptedChunk 则都为要转发的数据

### 密钥导出

#### pre shared master key（PSK） 生成

user passwd 加 md5计算而成
<https://github.com/shadowsocks/shadowsocks-windows/blob/b52472ebd7f519032cbf8a0ed688d090eeb75b2e/Shadowsocks.Net/Crypto/Stream/StreamCrypto.cs#L54>

#### session key 生成（真正用于加解密的key）

整个流程可简述为

HKDF-SHA1(key, salt, info) => subkey

HKDF 包含两个阶段，extract，expand

```
HKDF extract(algorithmName, salt, key) => HKDF-struct

HKDF-struct#expand(info, key_len) => session key
```

<https://github.com/shadowsocks/shadowsocks-windows/blob/8fafef7c5751809751d8eef361e3c39caef75261/Shadowsocks.Net/Crypto/AEAD/AEADCrypto.cs#L90>

```cs
// masterKey 就是 PSK，sessionKey则为生成结果，salt 从shadowsocks server的第一个chunk的头取，InfoBytes 始终为 ss-subkey 的 bytes 表示
//HKDF 包含 extract，expand 两个阶段，下面代码是两个阶段一起都做了
HKDF.DeriveKey(HashAlgorithmName.SHA1, masterKey, sessionKey, salt, InfoBytes);
```

上面的 `HKDF.DeriveKey` 同时包含extract, expand

如果是分两步进行，大致下面模样

```rs
pub fn hkdf_sha1(key: &[u8], salt: &[u8], info: Vec<u8>, size: usize) -> Result<Vec<u8>> {
    let (_, h) = Hkdf::<Sha1>::extract(Some(salt), key);
    let mut okm = vec![0u8; size];
    h.expand(&info, &mut okm)
        .map_err(|_|anyhow!("hkdf expand failed"))?;
    Ok(okm.to_vec())
}
```

这里要说明下，由于用户输入的密码往往是简单的字符串，所以大多数实现并不直接使用 password 作为 key，而是先进行 KDF（key derivation function），将KDF得到的值作为key，

常见的 KDF 算法有 PBKDF2， HKDF,这些算法的特点是，都可以生成指定长度的 key

--------
拿 `AES-128-GCM` 举个例子

128 bit 的 key 长度，也就是 16 字节

其他的 salt， nonce 长度并未有强制性要求，有些实现库支持 variable length

<http://shadowsocks.org/en/wiki/AEAD-Ciphers.html>

具体长度则由应用自行决定

shadowsocks 要求如下

| Name                   | Alias                  | Key Size | Salt Size | Nonce Size | Tag Size |
| ---------------------- | ---------------------- | -------- | --------- | ---------- | -------- |
| AEAD_CHACHA20_POLY1305 | chacha20-ietf-poly1305 | 32       | 32        | 12         | 16       |
| AEAD_AES_256_GCM       | aes-256-gcm            | 32       | 32        | 12         | 16       |
| EAD_AES_128_GCM        | aes-128-gcm            | 16       | 16        | 12         | 16       |

不同的加密算法所要求的 key size 不同，这就要求我们能任意生成不同长度的key

举个例子，HKDF 分extract，expand 两个阶段。extract 之后会得到一串伪随机数，expand 阶段可以指定长度，从而得到任意长度的 key。

# 密码学基础

## MAC (Message authentication code)

消息验证码，是附加到原文之后的数据，用于持有密钥的接收方进行校验，可察觉数据是否被篡改，只是一个概念，并不对应具体的实现算法。

为什么 mac 需要一个key呢？直接做一边hash当成tag不就可以了吗？

假设

```
小明 => 小红，小明用原文 A 做hash后得到 tag A 发给小红

hacker 劫持后，用伪造的原文B做hash后得到 tag B，同时将 原文A，tagA替换掉发给小红，小红获取 原文A（实际被替换成原文B）计算后与 tagA（实际被替换为tagB）比较，发现相同，认为数据没有篡改 !!

所以必须引入一个只有小红，小明才知道的key
```

新阶段为

```
小明

mac(key, textA) => tagA

由于 hacker不知道 key，假设随便使用一个 key1，伪造 textB

mac(key1, textB) => tagB

当小红
mac(key, textB) => tagC != tagB

则小红会发现数据被篡改
```

## HMAC (hash based message authentication code)

是 MAC 的一种实现算法，底层使用 hash 函数生成 mac

所以HMAC也需要一个key

HMAC(key, text) => tag

## HKDF (HMAC-based Extract-and-Expand Key Derivation Function)

将 HMAC 作为底层实现的密钥派生算法

```
//  https://datatracker.ietf.org/doc/html/rfc5869
A key derivation function (KDF) is a basic and essential component of
cryptographic systems.  Its goal is to take some source of initial
keying material and derive from it one or more cryptographically
strong secret keys.
```

KDF，即密钥派生算法，旨在通过一些初始的密码材料，派生出一个至多个密码学意义上，足够健壮的 密钥

HKDF 包含两个阶段，extract， expand

extract(salt, user_weak_password) => HKDF_struct

// If you don't have any info to pass, use an empty slice.
HKDF_struct.expand(info) => out_key

以下引用自 <https://datatracker.ietf.org/doc/html/rfc5869>， 我非常建议阅读，十分简单明了的RFC文档，介绍了每个数据的作用

```

In many applications, the input keying material is not necessarily
distributed uniformly, and the attacker may have some partial
knowledge about it (for example, a Diffie-Hellman value computed by a
key exchange protocol) or even partial control of it (as in some
entropy-gathering applications).  Thus, the goal of the "extract"
stage is to "concentrate" the possibly dispersed entropy of the input
keying material into a short, but cryptographically strong,
pseudorandom key.  In some applications, the input may already be a
good pseudorandom key; in these cases, the "extract" stage is not
necessary, and the "expand" part can be used alone.

The second stage "expands" the pseudorandom key to the desired
length; the number and lengths of the output keys depend on the
specific cryptographic algorithms for which the keys are needed.

2.2.  Step 1: Extract

   HKDF-Extract(salt, IKM) -> PRK

   Options:
      Hash     a hash function; HashLen denotes the length of the
               hash function output in octets

   Inputs:
      salt     optional salt value (a non-secret random value);
               if not provided, it is set to a string of HashLen zeros.
      IKM      input keying material

   Output:
      PRK      a pseudorandom key (of HashLen octets)

   The output PRK is calculated as follows:

   PRK = HMAC-Hash(salt, IKM)

2.3.  Step 2: Expand

   HKDF-Expand(PRK, info, L) -> OKM

   Options:
      Hash     a hash function; HashLen denotes the length of the
               hash function output in octets







Krawczyk & Eronen             Informational                     [Page 3]

RFC 5869                 Extract-and-Expand HKDF                May 2010


   Inputs:
      PRK      a pseudorandom key of at least HashLen octets
               (usually, the output from the extract step)
      info     optional context and application specific information
               (can be a zero-length string)
      L        length of output keying material in octets
               (<= 255*HashLen)

   Output:
      OKM      output keying material (of L octets)

   The output OKM is calculated as follows:

   N = ceil(L/HashLen)
   T = T(1) | T(2) | T(3) | ... | T(N)
   OKM = first L octets of T

   where:
   T(0) = empty string (zero length)
   T(1) = HMAC-Hash(PRK, T(0) | info | 0x01)
   T(2) = HMAC-Hash(PRK, T(1) | info | 0x02)
   T(3) = HMAC-Hash(PRK, T(2) | info | 0x03)
   ...
```

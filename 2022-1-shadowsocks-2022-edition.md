# Shadowsocks 2022版：基于对称加密的安全L4隧道

## 摘要

本文档定义了Shadowsocks协议的2022版。相比于Shadowsocks AEAD（2017版），Shadowsocks 2022解决了前几个版本的已知问题，摒弃了过时的加密方法，在安全性和性能上进行了优化，并为未来的扩展预留了空间。

## 1. 概述

Shadowsocks 2022是一种用于TCP和UDP流量的安全代理协议。该协议使用[AEAD](https://zh.wikipedia.org/wiki/带鉴别的加密)和预共享的对称密钥来确保数据的完整性和机密性。代理流量与随机字节流无异，因此能够绕过依赖于[深度包检测 (DPI)](https://zh.wikipedia.org/wiki/深度封包检测)的防火墙和互联网审查。

与[前几版](https://github.com/shadowsocks/shadowsocks-org/blob/master/whitepaper/whitepaper.md)相比，Shadowsocks 2022实现了强制性的重放保护。每条消息都有唯一的类型，不能用于其他用途。基于会话的UDP代理显著减少了协议开销，并提高了可靠性和效率。此外，过时的加密函数已被更现代的方案取代。

与以往一样，Shadowsocks 2022并不提供前向保密性。考虑到其应用场景，使用无需握手的预共享密钥被认为是最佳选择。

Shadowsocks 2022的实现包括服务器、客户端和可选的中继节点。本文档明确规定了实现时必须遵循的要求。

### 1.1. 文档结构

本文件描述了Shadowsocks 2022版，结构如下：

- 第2节介绍了加密密钥的要求及会话子密钥的派生方式。
- 第3节定义了AES-GCM方法的编码细节及请求和响应的处理过程。
- 第4节定义了可选的ChaCha-Poly1305方法的编码细节。

### 1.2. 术语和定义

本文档中的术语 "MUST"、"MUST NOT"、"REQUIRED"、"SHALL"、"SHALL NOT"、"SHOULD"、"SHOULD NOT"、"RECOMMENDED"、"NOT RECOMMENDED"、"MAY" 和 "OPTIONAL" 的解释需按照BCP 14 [RFC2119](https://www.rfc-editor.org/info/rfc2119)和[RFC8174](https://www.rfc-editor.org/info/rfc8174)中的定义进行，仅在这些术语以大写形式出现时具有上述含义。

下列术语在本文档中常用：

- Shadowsocks AEAD：2017年标准化的Shadowsocks AEAD构造。

## 2. 加密/解密密钥

预共享密钥（PSK）用于派生会话子密钥，以加密/解密会话中的流量。在某些场景中，PSK也可直接使用。

### 2.1. PSK

与旧版本不同，Shadowsocks 2022要求用户直接提供一个具有加密安全性的固定长度PSK。实现时**不得**使用旧的`EVP_BytesToKey`函数或其他密码生成密钥的方式。

为了方便起见，PSK采用Base64编码表示。可以使用以下命令生成：

```bash
openssl rand -base64 <key_size>
```

密钥大小取决于所选的加密方法。此设计受WireGuard启发。

| 方法                      | 密钥字节数 | 盐字节数 |
| ------------------------- | ---------: | -------: |
| 2022-blake3-aes-128-gcm   |        16  |       16 |
| 2022-blake3-aes-256-gcm   |        32  |       32 |

### 2.2. 子密钥派生

Shadowsocks 2022采用[BLAKE3](https://raw.githubusercontent.com/BLAKE3-team/BLAKE3-specs/master/blake3.pdf)的密钥派生模式，替换了旧版中的HKDF_SHA1函数。派生子密钥时，使用随机生成的盐附加到PSK后作为密钥材料。盐的长度与PSK相同。

```
session_subkey := blake3::derive_key(
    context: "shadowsocks 2022 session subkey", 
    key_material: key + salt
)
```

## 3. 必须实现的方法

所有实现必须支持`2022-blake3-aes-128-gcm`和`2022-blake3-aes-256-gcm`。`2022`反映了该协议的快速演进和灵活性。

### 3.1. TCP

Shadowsocks 2022中的TCP连接与代理连接一一映射。每个代理连接包含两个流：请求流和响应流。客户端通过启动请求流发起代理连接，服务器通过响应流返回数据。这些流中的数据块使用会话子密钥加密传输。

Shadowsocks 2022沿用了Shadowsocks AEAD的长度-数据块模型，并进行了少量调整以提升性能。请求和响应流的头部数据块已独立发送，提升了安全性，并避免了重放攻击。

#### 3.1.1. 加密和解密

每个代理流使用一个随机盐来派生用于加密和解密的会话子密钥。加密/解密操作中，使用12字节的小端整数作为计数器，每次操作后递增。

```
u96le counter
aead := aead_new(key: session_subkey)
ciphertext := aead.seal(nonce: counter, plaintext)
plaintext := aead.open(nonce: counter, ciphertext)
```

#### 3.1.2. 数据格式

请求流以一个随机盐和两个加密头部数据块开头，随后是若干长度数据块和负载数据块。响应流同样以随机盐开头，但只有一个固定长度的头部数据块。

```
请求流：
+--------+------------------------+---------------------------+...+
|  salt  | 加密头部数据块 (1)     | 加密头部数据块 (2)        |...|
+--------+------------------------+---------------------------+...|

响应流：
+--------+------------------------+---------------------------+...+
|  salt  | 固定长度头部数据块     | 加密负载数据块            |...|
+--------+------------------------+---------------------------+...|
```

#### 3.1.3. 头部格式

```
请求固定长度头部：
+------+------------------+--------+
| type |     timestamp    | length |
+------+------------------+--------+

响应固定长度头部：
+------+------------------+----------------+--------+
| type |     timestamp    |  request salt  | length |
+------+------------------+----------------+--------+
```

- `type`：用于区分客户端与服务器消息。请求流的类型为`0`，响应流为`1`。
- `timestamp`：Unix时间戳，大于30秒的时间差必须视为重放攻击。
- `length`：指示下一个数据块的明文长度。

#### 3.1.4. 重放保护

服务器必须在60秒内存储所有接收到的盐。建立新的TCP会话时，首个消息解密后，需检查其时间戳是否与系统时间在30秒内，并确保盐未重复。

---

### 3.2. UDP

Shadowsocks 2022对UDP代理进行了彻底重构。每个UDP会话都有一个唯一的会话ID，并使用其作为盐派生会话子密钥。每个UDP包也包含一个包ID作为计数器。

#### 3.2.1. 加密和解密

UDP包由一个单独的加密头部和一个AEAD加密的主体组成。

---

## 4. 可选方法

实现可以选择支持`2022-blake3-chacha20-poly1305`等方法，特别适用于不支持AES指令的CPU。

---

## 致谢

特别感谢@zonyitoo、@xiaokangwang和@nekohasekai对本协议设计的贡献。

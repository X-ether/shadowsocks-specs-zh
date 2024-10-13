# Shadowsocks 2022 扩展身份头（Extensible Identity Headers）

**身份头**是一个或多个额外的头层，每一层都包含下一层 PSK（预共享密钥）的哈希值。每层身份头的下一层可以是另一个身份头，或者如果是最后一层，则是协议头。身份头通过当前层的身份 PSK 使用 AES 分组加密进行加密。

身份头的实现方式完全向现有的 Shadowsocks 2022 版本**兼容**。每一个身份处理器对下一层来说都是透明的。

- **iPSKn**：表示第 n 层的身份 PSK，用于标识当前层。
- **uPSKn**：表示第 n 层用户 PSK，用于标识服务器上的某个用户。

---

## TCP

在 TCP 请求中，身份头位于**盐（salt）**和 **AEAD 块**之间。

```
identity_subkey := blake3::derive_key(context: "shadowsocks 2022 identity subkey", key_material: iPSKn + salt)
plaintext := blake3::hash(iPSKn+1)[0..16] // 取下一层 iPSK 哈希的前 16 字节。
identity_header := aes_encrypt(key: identity_subkey, plaintext: plaintext)
```

---

## UDP

在 UDP 请求包中，身份头位于**独立头部（session ID 和 packet ID）**和 **AEAD 密文**之间。

响应包不包含身份头。

```
plaintext := blake3::hash(iPSKn+1)[0..16] ^ session_id_packet_id // 通过异或操作确保每个包的内容不同。
identity_header := aes_encrypt(key: iPSKn, plaintext: plaintext)
```

在 UDP 请求包中，如果使用了 iPSKs，则**独立头部必须使用第一个 iPSK 加密**。每一层的身份处理器必须用下一层的 PSK 解密并重新加密独立头部。

在响应包中，独立头部不需要特殊处理。服务器会根据基础规范使用用户的 PSK 加密独立头部。身份处理器必须**直接转发**响应包给客户端，而无需解析或修改。

---

## 使用场景

```
      client0       >---+
(iPSK0:iPSK1:uPSK0)      \
                          \
      client1       >------\                        +--->    server0 [iPSK1]
(iPSK0:iPSK1:uPSK1)         \                      /      [uPSK0, uPSK1, uPSK2]
                             >-> relay0 [iPSK0] >-<
      client2               /    [iPSK1, uPSK3]    \
(iPSK0:iPSK1:uPSK2) >------/                        +--->    server1 [uPSK3]
                          /
      client3            /
   (iPSK0:uPSK3)    >---+
```

每个客户端会分配一组用冒号分隔的 PSK。客户端发送请求时，必须为每个 iPSK 生成一个身份头。

- **relay0** 通过其身份密钥解密第一个身份头，从 PSK 哈希表中找到目标服务器，并将请求的剩余部分转发出去。
- **支持多用户单端口**的服务器使用其身份密钥解密身份头，然后根据用户 PSK 哈希表找到用户 PSK 的加密方式，并处理请求的其余部分。

在上图中：
- `client0`、`client1` 和 `client2` 是通过 `relay0` 中继访问 `server0` 的用户。
- `server1` 是一个不支持身份头的简单服务器，`client3` 通过 `relay0` 连接到它。

---

## TCP 会话示例

1. **client0** 生成一个随机盐。
2. 使用 **iPSK0** 衍生的子密钥加密 **iPSK1** 的哈希，作为第一个身份头。
3. 使用 **iPSK1** 衍生的子密钥加密 **uPSK0** 的哈希，作为第二个身份头。
4. 按照原始规范完成请求的其余部分。

**relay0** 处理 TCP 请求时：
1. 使用 **iPSK0** 衍生的子密钥解密第一个身份头。
2. 从 PSK 哈希表中查找目标服务器，将盐和请求的其余部分（不包括已处理的身份头）发送给 **server0**。

---

## UDP 包示例

1. **client0** 使用 **iPSK0** 加密独立头部。
2. 使用 **iPSK0** 加密 `(iPSK1 的哈希 ^ session_id_packet_id)` 作为第一个身份头。
3. 使用 **iPSK1** 加密 `(uPSK0 的哈希 ^ session_id_packet_id)` 作为第二个身份头。
4. 按照原始规范完成请求。

**relay0** 处理 UDP 包时：
1. 原地解密 **独立头部**（使用 iPSK0）。
2. 使用 **iPSK0** 解密第一个身份头，从 PSK 哈希表中找到目标服务器。
3. 将重新加密的独立头部放置在第一个身份头的位置，并将包的剩余部分发送给 **server0**。

---

## 总结

Shadowsocks 2022 的**扩展身份头**机制通过多层 PSK 加密，为中继和服务器之间的身份验证提供了更灵活的支持。  
- **中继节点**（如 relay0）可以解析和转发多层身份头，确保请求到达正确的目标服务器。  
- **服务器**可以在支持多用户单端口的情况下，使用用户 PSK 进行加密和解密。

这种设计确保了协议的安全性和灵活性，同时保持与现有 Shadowsocks 2022 实现的**向后兼容性**。

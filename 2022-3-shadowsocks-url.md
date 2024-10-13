# Shadowsocks 服务器配置 URL

本文档定义了用于共享 Shadowsocks 服务器配置的 URL 格式。

大多数编程语言的标准库都提供了用于解析和组装 URL 的工具。可以将以下信息输入 URL 构建器，以生成所需的 URL。

- **协议（Scheme）**：`ss`  
- **用户信息（Userinfo）**  
  - **用户名**：加密方法（`method`）  
  - **密码**：密码（`password`）  
- **片段（Fragment）**：服务器名称  

### 示例：

```
ss://2022-blake3-aes-128-gcm:5mOQSa20Kt6ay2LXruBoHQ%3D%3D@example.com:443/#name
ss://2022-blake3-aes-256-gcm:t7XRzLCvgsH4r4r669cyqPnVNFG2c%2FHC5Tt%2BMjINJB0%3D@[2001:db8:1f74:3c86:aef9:a75:5d2a:425e]:20220/#name
```

### 说明：

1. **加密方法**（method）：放在 URL 的用户名部分。
2. **密码**（password）：放在 URL 的密码部分，并进行 URL 编码（如 `%3D` 表示 `=`）。
3. **服务器地址**：可以是域名或 IPv4/IPv6 地址。例如：`example.com` 或 `[2001:db8:...]`。
4. **端口号**：放在服务器地址之后，使用 `:` 分隔。
5. **服务器名称**（name）：放在 URL 的片段部分（`#` 后）。

这些配置 URL 可用于快速分享服务器信息，方便客户端解析和连接。

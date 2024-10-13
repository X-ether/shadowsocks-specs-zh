# Shadowsocks 2022 实现

本文档跟踪和比较 Shadowsocks 2022 的各种实现，涵盖功能与规范的符合情况，并展示一些基准测试和测速结果。

## 1. 实现

### 1.1. 参考实现

Shadowsocks 2022 的参考实现完全符合规范，经过性能优化，可用于生产环境。

|      | shadowsocks-go        | shadowsocks-rust         | sing-box            |
| ---- | --------------------: | -----------------------: | ------------------: |
| 描述 | 高效的安全通信代理平台 | 用 Rust 编写的 CLI 和库   | 通用代理平台        |
| URL  | [GitHub](https://github.com/database64128/shadowsocks-go) | [GitHub](https://github.com/shadowsocks/shadowsocks-rust) | [GitHub](https://github.com/SagerNet/sing-shadowsocks), [GitHub](https://github.com/SagerNet/sing-box) |
| 许可证 | AGPLv3 | MIT | GPLv3 |
| 服务器 | ✅ | ✅ | ✅ |
| 中继   | ❌ | ❌ | ✅ |
| 客户端 | ✅ | ✅ | ✅ |
| [EIH (扩展身份头)](2022-2-shadowsocks-2022-extensible-identity-headers.md) | ✅ | ✅ | ✅ |

### 1.2. 衍生实现

这些实现基于参考实现，由下游开发者独立开发。

|      | v2ray-core (SagerNet 分支) | Xray-core           |
| ---- | -------------------------: | ------------------: |
| URL  | [GitHub](https://github.com/SagerNet/v2ray-core) | [GitHub](https://github.com/XTLS/Xray-core) |
| 协议实现 | sing-shadowsocks | sing-shadowsocks |
| 许可证   | GPLv3 | MPLv2 |
| 服务器   | ✅ | ✅ |
| 中继     | ✅ | ✅ |
| 客户端   | ✅ | ✅ |
| [EIH (扩展身份头)](2022-2-shadowsocks-2022-extensible-identity-headers.md) | ✅ | ✅ |

### 1.3. 图形用户界面客户端 (GUI Clients)  

（此部分待完善）

### 1.4. 其他实现  

（此部分待完善）

---

## 2. 基准测试

### 2.1. 头部性能基准测试  

代码地址：[测试代码](https://github.com/database64128/cubic-go-playground/blob/main/shadowsocks/udpheader/udpheader_test.go)  

```
BenchmarkGenSaltHkdfSha1-8          	  372712	      3283 ns/op	    1264 B/op	      18 allocs/op
BenchmarkGenSaltBlake3-8            	  770426	      1554 ns/op	       0 B/op	       0 allocs/op
BenchmarkAesEcbHeaderEncryption-8   	99224395	        12.84 ns/op	       0 B/op	       0 allocs/op
BenchmarkAesEcbHeaderDecryption-8   	100000000	        11.79 ns/op	       0 B/op	       0 allocs/op
```

### 2.2. 完整 UDP 数据包构建性能基准测试  

代码地址：[测试代码](https://github.com/database64128/cubic-go-playground/blob/main/shadowsocks/udp_test.go)  

```
BenchmarkShadowsocksAEADAes256GcmEncryption-8             	  252819	      4520 ns/op	    2160 B/op	      23 allocs/op
BenchmarkShadowsocksAEADAes256GcmWithBlake3Encryption-8   	  320718	      3367 ns/op	     896 B/op	       5 allocs/op
BenchmarkDraftSeparateHeaderAes256GcmEncryption-8         	 2059383	       590.5 ns/op	       0 B/op	       0 allocs/op
BenchmarkDraftXChaCha20Poly1305Encryption-8               	  993336	      1266 ns/op	       0 B/op	       0 allocs/op
```

### 2.3. shadowsocks-rust 测速结果  

|       shadowsocks-rust        |   TCP 速度  |   UDP 速度  |
| ----------------------------- | ----------: | ----------: |
| 2022-blake3-aes-128-gcm       | 12.2Gbps    | 14.2Gbps    |
| 2022-blake3-aes-256-gcm       | 10.9Gbps    | 12.5Gbps    |
| 2022-blake3-chacha20-poly1305 | 8.05Gbps    | 2.35Gbps    |
| 2022-blake3-chacha8-poly1305  | 8.36Gbps    | 2.60Gbps    |
| aes-128-gcm                   | 8.99Gbps    | 13.5Gbps    |
| aes-256-gcm                   | 8.21Gbps    | 11.9Gbps    |
| chacha20-poly1305             | 6.55Gbps    | 8.66Gbps    |

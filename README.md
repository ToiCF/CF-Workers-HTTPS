# CF-Workers-HTTPS

`HTTPS.js` 是一个基于 Cloudflare Workers 的 HTTPS 代理 `CONNECT` 出站实验。

它的重点不只是“走 HTTPS 代理”，而是把两条完全不同的路径同时做进了 Worker：

1. **原生快路径**：直接使用 Workers 原生 `secureTransport: 'on'`。
2. **自定义 TLS 路径**：先走原始 TCP，再在 Worker 内自己完成 TLS 握手，用来覆盖原生路径做不到的场景。

这也是本项目和普通 `CONNECT` 转发代码的本质区别。

> 这是研究性、学习性代码。它主要用于验证 Workers 在 HTTPS 代理场景下的能力边界：什么时候可以直接吃 Cloudflare 原生 TLS，什么时候必须自己接管 TLS。实际稳定性与代理质量、证书情况、网络环境和 Workers 运行时限制有关。

## 核心结论

Cloudflare 原生提供的：

```js
connect({ hostname: proxy.host, port: proxy.port }, { secureTransport: 'on' })
```

只能解决 **Worker → HTTPS 代理** 这一跳的标准 TLS 场景。

也就是说，原生快路径适合：

- 代理地址是正常可校验的域名；
- 证书链正常；
- 证书与目标主机名匹配；
- 不需要跳过证书校验。

但 Cloudflare 没有提供“跳过 TLS 校验”的原生开关。

因此一旦遇到下面这些场景，原生路径就不够了：

- 代理只有 `IP:PORT`；
- 自签证书；
- 证书名不匹配；
- 需要忽略证书校验继续建立 HTTPS 代理隧道。

这时就必须改成：

```text
原始 TCP → Worker 内自定义 TLS 握手 → 发送 CONNECT → 建立隧道
```

而这正是 `TLSClientMini.js` 的意义。

## 文件结构

| 文件 | 说明 |
| --- | --- |
| [HTTPS.js](./HTTPS.js) | 源码版 Worker：同时保留原生 `secureTransport:on` 快路径和自定义 TLS 路径。 |
| [TLSClientMini.js](../../TLS/TLSClientMini.js) | 最小 TLS 核心，面向特殊 HTTPS 代理场景使用。 |
| [HTTPSMini.js](./HTTPSMini.js) | 合并版单文件 Worker：把 HTTPS 代理逻辑和 Mini TLS 路径压进一个可直接部署的单文件里。 |

## 两条路径分别在干什么

### 1. 原生快路径

`HTTPS.js` 在标准场景下直接使用：

```js
connect({ hostname: proxy.host, port: proxy.port }, { secureTransport: 'on', allowHalfOpen: false })
```

然后向代理发送：

```text
CONNECT target:port HTTP/1.1
Host: target:port
Proxy-Authorization: Basic xxxxxxxx
```

这条路径的特点是：

- TLS 握手由 Cloudflare 原生实现；
- 路径最短；
- 延迟和开销最低；
- 适合证书正常、域名匹配的 HTTPS 代理。

可以把它理解成：

```text
Worker → 原生 TLS → HTTPS Proxy → CONNECT target → TCP 隧道
```

### 2. 自定义 TLS 兼容路径

如果 URL 带了 `insecure` 标记，代码不会再走原生 `secureTransport:on`，而是改成：

```text
Worker → 原始 TCP → TLSClientMini 握手 → CONNECT target → TCP 隧道
```

这条路径的意义在于：

- 不再依赖 Cloudflare 原生 TLS 校验；
- 可以接管握手逻辑；
- 可以处理证书不匹配、自签或纯 `IP:PORT` 的代理入口；
- 用来补齐原生 HTTPS 代理路径的能力空白。

一句话说：

**原生路径负责快，自定义 TLS 路径负责补原生做不到的兼容性。**

需要区分两种实现：

- `HTTPS.js` 这条源码路径使用的是完整 `TLSClient.js`，并通过 `insecure: true` 明确关闭证书 / 主机名校验。
- `HTTPSMini.js` 内联的是 `TLSClientMini.js` 这一套最小 TLS 实现；它本身就不做完整证书校验，因此天然属于兼容 / 跳过验证路径。

## TLSClientMini 的定位

`TLSClientMini.js` 不是浏览器级完整 TLS 栈，也不是为了做完整 PKI 验证器。

它的目标非常明确：

- 只保留 Worker 场景最需要的黄金路径；
- 只做最小必要的 TLS 1.2 / TLS 1.3；
- 只保留关键 AEAD 套件与握手收发；
- 不实现完整证书链 / 主机名校验；
- 服务于“原生 `secureTransport` 不够用”的 HTTPS 代理场景。

也因此，`HTTPSMini.js` 的意义不是“重复造一个 HTTPS 代理”，而是把：

```text
HTTPS CONNECT 逻辑 + Mini TLS 握手能力
```

压成一个更适合直接部署、直接分发的单文件 Worker。

## 代理参数格式

请求 URL 中使用：

```text
?https://proxy.example.com:443
```

如果代理需要认证：

```text
?https://user:pass@proxy.example.com:443
```

如果要走自定义 TLS 路径：

```text
?https://proxy.example.com:443&insecure
```

或：

```text
?https://user:pass@1.2.3.4:443&insecure
```

代码会自动解析：

- 代理主机
- 代理端口
- 用户名 / 密码
- `Proxy-Authorization: Basic ...`
- 是否启用 `insecure` 路径

## 建立流程

### 标准证书代理

```text
客户端 → Worker
  → secureTransport:on
  → HTTPS Proxy
  → CONNECT target:port
  → 建立隧道
```

### 跳过证书校验 / 纯 IP 代理

```text
客户端 → Worker
  → raw TCP
  → TLSClientMini
  → CONNECT target:port
  → 建立隧道
```

## Worker 节点格式

WebSocket path 形式：

```text
/?https://proxy.example.com:443
```

认证代理：

```text
/?https://user:pass@proxy.example.com:443
```

走自定义 TLS 路径：

```text
/?https://1.2.3.4:443&insecure
```

VLESS 分享链接模板：

```text
vless://<uuid>@<front-host>:<front-port>/?type=ws&encryption=none&host=<worker-host>&path=%2F%3Fhttps%3A%2F%2Fproxy.example.com%3A443&security=tls&sni=<worker-host>&fp=random#HTTPS
```

自定义 TLS 路径模板：

```text
vless://<uuid>@<front-host>:<front-port>/?type=ws&encryption=none&host=<worker-host>&path=%2F%3Fhttps%3A%2F%2F1.2.3.4%3A443%26insecure&security=tls&sni=<worker-host>&fp=random#HTTPS-Insecure
```

## 原生路径与 Mini 路径的关系

这套实现不是“二选一”，而是故意双轨并存：

- **能走 Cloudflare 原生 TLS 时，就走原生快路径**；
- **原生路径做不到时，再切换到自定义 TLS 路径**。

所以 `HTTPSMini` / `TLSClientMini` 的定位，从来都不是替代原生快路径，而是：

**保住原生路径性能的同时，把特殊 HTTPS 代理场景补齐。**

## 已知限制

- `secureTransport` 只负责 `Worker → 代理` 这一跳是否使用 Cloudflare 原生 TLS。
- 原生接口没有“跳过证书验证”的开关。
- 标准 HTTPS 代理更适合 `domain:port + 正常证书` 的场景。
- 纯 `IP:PORT`、自签、证书不匹配等情况，通常要走自定义 TLS 路径。
- `TLSClientMini` 的目标是最小可用，不是完整浏览器 TLS 能力复刻，也不提供完整证书校验能力。
- 这是协议研究实现，不保证所有代理都稳定可用。

## 适合场景

- HTTPS 代理 `CONNECT` 出站实验。
- Cloudflare 原生 TLS 快路径与自定义 TLS 路径对照研究。
- 纯 IP HTTPS 代理 / 跳过证书验证场景验证。
- Worker 内最小 TLS 客户端能力研究。
- 单文件 HTTPS 代理 Worker 部署。

## 相关链接

- 开源协议：[GPL-3.0](./LICENSE)
- 频道 / 交流群组：<https://t.me/Enkelte_notif>

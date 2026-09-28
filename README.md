# myproxy

> **已归档 / Archived:** 本仓库仅保留历史代码，不再作为维护中的项目。后续主线是 [AsterFerry](https://github.com/afterlune/asterferry)。AsterFerry 使用了新的架构；本仓库的 YAML 配置不能直接导入 AsterFerry。

> This repository is archived and kept for historical use. It is no longer actively maintained. The successor project is [AsterFerry](https://github.com/afterlune/asterferry), which uses a different architecture and does not directly import this project's YAML configuration.

## 项目简介 / Overview

`myproxy` 是一个配置驱动的 HTTP 和 SOCKS5 代理。入口可按路由规则直接连接目标，或通过 QUIC 连接配置的远端 endpoint。endpoint 和代理入口可以分别运行在不同主机上。

`myproxy` is a configuration-driven HTTP and SOCKS5 proxy. Inbound requests can connect directly to their destination or use a configured remote endpoint over QUIC. The endpoint and proxy inbounds can run on separate hosts.

项目内置 GeoIP 数据库供路由规则查询目的地址的国家/地区代码。仓库中的 `config-c.yaml` 是示例配置，使用前需要按本机网络和路由意图修改。

The project embeds a GeoIP database for routing by destination country/region code. The root `config-c.yaml` is an example and should be adjusted for the local network and intended routing behavior.

## 要求 / Requirements

- Go 1.22 或更新版本 / Go 1.22 or newer
- 代理主机与远端 endpoint 之间可达的 UDP 网络 / UDP connectivity from the proxy host to the remote endpoint
- 为 SOCKS5 或 HTTP 客户端配置对应的本地代理地址和端口 / Configure clients to use the local SOCKS5 or HTTP proxy address and port

## 构建与运行 / Build and run

仓库根目录中的 `config.yaml` 是 endpoint 示例；`config-c.yaml` 是代理入口和 outbound 示例。启动参数 `-c` 指定配置文件，默认值是 `config.yaml`。

The root `config.yaml` is the endpoint example; `config-c.yaml` defines proxy inbounds and an outbound. Use `-c` to select a configuration file; the default is `config.yaml`.

分别在远端 endpoint 主机和代理主机克隆仓库。/ Clone the repository on both the remote endpoint host and the proxy host.

```sh
git clone https://github.com/afterlune/myproxy.git
cd myproxy
```

在远端主机启动 endpoint：

Start the endpoint on the remote host:

```sh
go build -o myproxy .
./myproxy -c config.yaml
```

在代理主机上，把 `config-c.yaml` 中 outbound 的 `address` 改成 endpoint 主机的可达地址，然后运行：

On the proxy host, set the outbound `address` in `config-c.yaml` to the reachable address of the endpoint host, then run:

```sh
./myproxy -c config-c.yaml
```

也可以使用 `go run . -c config.yaml` 或 `go run . -c config-c.yaml`，不先构建二进制。

You can also run `go run . -c config.yaml` or `go run . -c config-c.yaml` without building a binary first.

默认 `config-c.yaml` 在 `127.0.0.1:1080` 提供 SOCKS5，在 `127.0.0.1:1081` 提供 HTTP 代理。将浏览器或应用的代理设置指向相应地址和端口。

By default, `config-c.yaml` provides SOCKS5 on `127.0.0.1:1080` and HTTP proxy on `127.0.0.1:1081`. Point the browser or application proxy settings to the appropriate address and port.

## 配置说明 / Configuration

| 配置项 / Setting | 作用 / Purpose |
| --- | --- |
| `endpoint.address`, `endpoint.port` | QUIC/UDP endpoint 的监听地址和端口。/ QUIC/UDP listen address and port for the endpoint. |
| `inbounds[]` | 本地 HTTP 或 SOCKS5 监听器，`protocol` 使用 `http` 或 `socks`；`setting.user` 与 `setting.pass` 可配置入站凭据。/ Local HTTP or SOCKS5 listeners (`protocol` is `http` or `socks`); `setting.user` and `setting.pass` configure inbound credentials. |
| `outbounds[]` | 远端 endpoint 的 `address` 和 `port`，以及在该节点上使用的 `nodePort`。`tag` 用来被路由规则引用。/ Remote endpoint `address` and `port`, plus the `nodePort` used on that node. `tag` is referenced by routing rules. |
| `routing.rules[]` | 按入站 tag 和 GeoIP 国家代码选择 outbound，或选择 `direct`。/ Select an outbound or `direct` based on inbound tag and GeoIP country code. |
| `transfer.tls` | 配置 endpoint 证书，或为测试设置客户端的 `insecure: true`。/ Configure the endpoint certificate or set client `insecure: true` for testing. |

默认 endpoint 示例监听 `0.0.0.0:23456`。默认代理示例把 outbound 指向 `127.0.0.1:23456`，并请求远端 `nodePort: 21086`；部署到不同主机时要改为 endpoint 的可达地址。防火墙需要允许代理主机访问 endpoint UDP 端口和配置的 `nodePort` UDP 端口。默认 endpoint 生成的证书是自签名的；连接示例前，请先按下面的 TLS 说明配置证书。

The default endpoint example listens on `0.0.0.0:23456`. The proxy example points its outbound at `127.0.0.1:23456` and requests remote `nodePort: 21086`; change the address when the endpoint runs on another host. Allow the proxy host to reach both the endpoint UDP port and the configured `nodePort` UDP port through the firewall. The default endpoint generates a self-signed certificate; configure TLS as described below before connecting with the sample configuration.

当前路由实现比较 GeoIP 返回的 ISO 国家代码。比如美国代码是 `US`，如果要匹配该代码，规则值应使用 `!US`；仓库示例中的 `!USA` 与当前实现不匹配。其他地区请按数据库返回的 ISO 代码填写。此规则不是通用 IP/CIDR 匹配器。

The current routing implementation compares values against the ISO country code returned by GeoIP. For example, the US code is `US`, so use `!US` to match it; the `!USA` value in the checked-in example does not match the current implementation. Use the ISO code returned for other regions as well. These rules are not general IP/CIDR matching rules.

## TLS 与安全 / TLS and security

endpoint 未配置证书时会生成自签名证书。客户端默认验证系统证书链，因此仓库示例在默认 TLS 设置下可能无法完成连接。生产部署应使用客户端信任且名称匹配的证书；仅在可信的临时测试环境中，才在客户端配置中设置：

When no certificate is configured, the endpoint generates a self-signed certificate. The client verifies the system certificate chain by default, so the checked-in examples may fail to connect with their default TLS settings. For production, use a certificate trusted by the client and matching the endpoint name. Only for a trusted, temporary test environment, set this in the client configuration:

```yaml
transfer:
  tls:
    insecure: true
```

`insecure: true` 会跳过服务器证书验证，不应用于不可信网络。`transfer.obfuscation.xorKey` 和 `padding` 只影响数据帧混淆/填充，不是加密或身份验证机制；安全性依赖 QUIC/TLS 配置。

`insecure: true` skips server certificate verification and should not be used over untrusted networks. `transfer.obfuscation.xorKey` and `padding` only affect frame obfuscation/padding; they do not provide encryption or authentication. Security depends on the QUIC/TLS configuration.

默认 inbounds 只监听 loopback，且没有配置用户凭据。若改为监听局域网或公网地址，请配置 `setting.user` 与 `setting.pass` 并通过防火墙限制来源。HTTP Basic 和 SOCKS5 用户名/密码认证不会加密代理入口这一段连接；不要在不可信网络上直接暴露这些监听端口。QUIC/TLS 保护的是代理与 endpoint 之间的连接。

The default inbounds listen only on loopback and have no credentials configured. If you bind them to a LAN or public address, configure `setting.user` and `setting.pass` and restrict access with a firewall. HTTP Basic and SOCKS5 username/password authentication do not encrypt the connection to the proxy inbound; do not expose these listeners directly on an untrusted network. QUIC/TLS protects the connection between the proxy and the endpoint.

## 许可证 / License

详见 [`LICENSE`](LICENSE)。 / See [`LICENSE`](LICENSE).

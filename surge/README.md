# Surge

[← 返回总览](../README.md) · 配置文件：`surge.conf`

完整配置，可从 URL 导入。本文件覆盖导入步骤，以及 DNS 限制、APNs、测速、广告拦截、规则求值顺序这些 Surge 特有的细节。

## 使用方法

1. 打开 Surge → 首页顶部 / Mac 侧边栏最下方的 [更多] → [从 URL 下载配置]，填：

```text
https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/surge/surge.conf
```

也可在已安装 Surge 的设备上用一键导入（把配置 URL 交给 Surge 打开）：

```text
surge:///install-config?url=https%3A%2F%2Fgh-proxy.com%2Fhttps%3A%2F%2Fraw.githubusercontent.com%2F2253845067%2FProxy-Config%2Fmain%2Fsurge%2Fsurge.conf
```

镜像打不开时，把上面两处的 `https://gh-proxy.com/` 前缀去掉，改用 GitHub 原始地址：

```text
https://raw.githubusercontent.com/2253845067/Proxy-Config/main/surge/surge.conf
```

2. 导入并切换到这份配置后，打开 [策略组] → `机场订阅`（该组已隐藏，可在策略组列表里找到），把 `policy-path` 里的占位串 `这里填入你的机场订阅 URL` 换成你的机场订阅 URL 并保存。**只需要改这一处**：`Proxy` / `Auto` 通过 `include-other-group` 复用同一份订阅。
3. 更新外部资源，确认 `Proxy` / `Auto` 里能看到节点。若你更习惯在 Surge 里单独添加外部资源 / 订阅，也可以不改 `policy-path`——`include-all-proxies=true` 会把这些手动添加的节点一并收进来（同一套过滤正则仍然生效）。注意：**组内一个节点都没有时 Surge 不会报错，而是把该组回落成 `DIRECT`**（日志里显示 `SUBSTITUTE`），表现就是"全部直连"，看到这个现象先回去查订阅。
4. `场景分流` 已将 `YUQI` 设为直连。若要配置第二个路由器，可把 `"WiFi2"` 改为实际 SSID，并取消 `[SSID Setting]` 中对应模板行的注释、填入路由器 IP；连上该 Wi-Fi 时 DNS 会交给路由器解析。不使用第二个路由器时无需更改模板。
5. 开启 Surge（iOS 用 VPN / 增强模式，Mac 用增强模式或接管系统代理），并在 [工具] 里更新一次规则集与 GeoIP 数据库。

## DNS

**防泄漏**：配置只写 IP 形式的国内 DoH（`223.5.5.5` / `1.12.12.12` / `223.6.6.6`），**故意不写明文 `dns-server`**，避免 DoH 主机名还要先走一遍明文 bootstrap；并用 `hijack-dns = *:53` 把应用自己发起的明文 DNS 查询交给 Surge。

`encrypted-dns-follow-outbound-mode = false`，因为这三条 DoH 国内可直连，不必再经 `Proxy`，也可避免和节点解析形成循环依赖（代价是 `PROTOCOL,DOH` / `DOQ` 这类规则不会生效，加密 DNS 查询根本不进规则系统）。

连上透明代理路由器时例外：`[SSID Setting]` 里 `SSID:YUQI dns-server=192.168.77.1,encrypted-dns-server=off` 会把该网络的 DNS 交给路由器，路由器的 DNS 分流 / 广告拦截才会生效；全局已配加密 DNS，所以 `encrypted-dns-server=off` 必须显式写，否则切不回传统 DNS。

**DNS 限制**（相对 mihomo）：Surge 没有 `nameserver-policy`，不能把海外域名的 DoH 经 `Proxy` 出境，因此这里采用和 Loon 相同的思路——全程走国内加密 DNS，而不是 mihomo 那套「国内走 223.5.5.5、海外走 1.1.1.1#Proxy」。

真要给个别域名换解析器，Surge 的出口是 `[Host]` 的 `server:` 映射，例如 `example.com = server:https://1.1.1.1/dns-query`（只对写死的域名生效，不是按规则分流）。

加密 DNS 失败时 Surge 仍可能回落到系统 DNS（与 Loon 类似）；若必须留明文兜底，可取消配置里 `dns-server` 那一行的注释。另外 Surge 不是 mihomo 那种可精细控制的 fake-ip 模式，只用 `always-real-ip` 保证局域网 / Apple 等域名拿到真实 IP；`read-etc-hosts = false`，本机 hosts 不会改解析结果。

## 其他关键项

`udp-policy-not-supported-behaviour = reject`（节点不支持 UDP 时直接拒绝，而不是回落到直连）、`allow-wifi-access = false`（不向局域网共享代理）、`ipv6 = false` + `ipv6-vif = auto`。

局域网流量由 `skip-proxy` 与 `[Rule]` 的私网规则兜住，`skip-proxy` 已连带写上 IPv6 的 `fc00::/7` / `fe80::/10`——现在 `ipv6 = false` 用不上，将来开 IPv6 时不用再改这一行。

为满足 APNs 接管要求，iOS 配置开启了 `include-all-networks`；`include-local-networks` 仍保持默认关闭，因此不会额外把发往局域网的请求纳入 Surge VIF。

## APNs 推送

iOS 配置启用了 `include-all-networks = true` 与 `include-apns = true`，否则 Surge VIF 不会接管 APNs。

APNs 域名规则走 `场景分流`，优先于 `AppleCN`；已知 IPv4 / IPv6 网段放在 IP 兜底区，避免过早触发 DNS。

`include-all-networks` 会扩大隧道接管范围，官方提示可能影响 AirDrop、Xcode 调试和 USB Dashboard；Mac 会忽略这两个 iOS 选项。若不需要代理 APNs，可关闭 `include-apns` 并移除对应规则；若保留 `include-apns`，`include-all-networks` 也必须保持开启。

## 测速与通知

`proxy-test-url` 用 `http://cp.cloudflare.com/generate_204`，不用 gstatic——测速 URL 不通会把好节点判成不可用、被 `Auto` 直接跳过，而 Cloudflare 的 204 端点可达性最好（国内也能直连）。

`internet-test-url` 仍用国内 hicloud，因为 DIRECT 策略的测速必须国内可达。

`Auto` 另加了 `no-alert=true`：最优节点变化时不再逐条推送通知，组本身照常自动切换；想恢复通知删掉这一项即可。

## 广告拦截

用 `REJECT` + `pre-matching`，在 DNS 与 TCP SYN 阶段就拒掉，不必等连接建好再拒。`REJECT` 自带频率保护（同一域名 30 秒内被拒 50 次后自动升级为 `REJECT-DROP`），既快速失败又不怕重试风暴。

**不要改成 `REJECT-DROP`**：它在 DNS 阶段会丢弃查询不回包，应用只能等超时，反而更慢；也不要用 `REJECT-NO-DROP` 当"静默丢弃"——它的语义是"照常拒绝、但不自动升级"，与 mihomo 的 `REJECT-DROP` 并不等价。

## QUIC

用 `[General]` 的 `block-quic = all` 拦截所有 UDP/443，让浏览器和应用回落 TCP，与 Mihomo、Loon、Egern、sing-box 的全局拦截保持一致。需要 HTTP/3 时改成 `always-allow`。

## 规则求值顺序

`Reject` / `Ads`（`pre-matching`）→ 私有网络（`Lan.list` 带 `no-resolve`）→ `PROTOCOL,STUN,REJECT` → CN 域名段 → Proxy 域名段 → CN IP 兜底（`ChinaASN` / `ChinaIP` / `GEOIP`，**不带** `no-resolve`）→ `FINAL`。

- **不要**去掉 `Lan.list` 的 `no-resolve`：它内部有 8 条 `IP-CIDR`，去掉后每个域名请求求值到那里都会先做一次真实 DNS 解析。
- **也不要**把 CN IP 三层挪到 Proxy 域名段之前：那等于又变回"每个请求都要解析"，而且会把域名表判定该走代理的流量抢成直连。
- `PROTOCOL,STUN,REJECT` 用来减少 WebRTC 泄漏真实 IP（与 Egern / Loon 的 STUN 处理对齐），已挪到私有网络之后让局域网内的 STUN 走直连；FaceTime / 语音会议异常时可先注释它。

## iOS / Mac 共用一份

`read-etc-hosts` 只在 Mac 生效，`allow-wifi-access` / `all-hybrid` 只在 iOS 生效，另一端会直接忽略、不会报错，所以不要为了"两端通用"删掉它们。`icon-url` 需要 Surge Mac 6.5.0+，旧版本只是不显示图标。

## GeoIP / ASN

`GEOIP,CN` 使用 `geoip-maxmind-url` 指向的 Loyalsoldier 国家库（经 `gh-proxy.com` 中转）。

`ChinaASN`（`IP-ASN`）用的是 Surge **自带**的 GeoLite2 ASN 库，只随 App 更新，配置里没有像 Egern `asn_db_url` 那样的更换入口——这是 Surge 的能力边界，不是漏配。库过旧时部分国内 IP 可能被误走代理，优先升级 Surge。

## 本配置不含模块 / MITM 证书

没有移植 Loon 插件或 Egern 模块，`[Proxy]`、证书相关字段都是空的。需要去广告脚本或 MITM 时，在客户端自行添加模块，不要把 `ca-p12` / 订阅写回这个仓库。

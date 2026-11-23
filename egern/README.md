# Egern

[← 返回总览](../README.md) · 配置文件：`egern.yaml`

完整配置，可一键导入。本文件覆盖导入步骤，以及 DNS、策略组顺序、场景（SSID）分流、局域网排除、模块与 MITM 这些 Egern 特有的细节。

## 使用方法

1. 点击一键导入 [配置](https://egernapp.com/profiles/new?name=egern&url=https%3A%2F%2Fgh-proxy.com%2Fhttps%3A%2F%2Fraw.githubusercontent.com%2F2253845067%2FProxy-Config%2Fmain%2Fegern%2Fegern.yaml)。
2. 如果无法跳转导入，打开 Egern 底部 [设置] -> 最上面 [配置] -> 右上角 [+号] -> [下载]，URL 填写：

```text
https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/egern/egern.yaml
```

3. 导入完成后，切换并使用 `egern` 配置。
4. 打开 [策略组] -> `机场订阅`，把 `urls` 里的占位串 `这里填入你的机场订阅 URL` 换成你的机场订阅 URL 并保存（该组已从旧的 `external` 写法迁移为官方推荐的 `select + urls`，两者等价）。**只需要改这一处**：`Proxy` 组用的是同一个 YAML 锚点 `*SubURL`，会自动跟着变。
5. 打开 [策略组] -> `场景`。`YUQI` 已作为默认透明代理路由器 SSID，`WiFi2` 保留为第二个路由器的模板，需要时可手动改名；不用场景分流时，可把 `rules` 末尾 `default` 的 `policy` 改回 `Proxy`。
6. 回到策略组页面更新外部资源，确认 `Proxy` / `Auto` 里能看到节点。
7. 回到仪表界面打开 Egern 开关即可。

**镜像打不开时**：把上面两处地址的 `https://gh-proxy.com/` 前缀去掉（一键导入链接里的 `%2F` 编码前缀同理），改用 GitHub 原始地址：

```text
https://raw.githubusercontent.com/2253845067/Proxy-Config/main/egern/egern.yaml
```

## DNS

`dns.proxy_nameservers` 的连接按官方语义是**强制直连**、且会绕过 `forward` 规则，所以必须填国内可达的 DoH（当前为 223.5.5.5 / 1.12.12.12）。若填 `8.8.8.8`、`1.1.1.1`，在国内网络下节点域名解析会失败，所有以域名下发的节点会直接不可用。

`bootstrap`（223.5.5.5 / 119.29.29.29）只用于解析 DoH 主机名，同样必须是国内可达的地址。

`forward` 已让 Lan、Direct、WeChat、Bilibili、AppleCN、OneDrive、DownloadCDN_CN 和 ChinaDomain 使用国内 DNS，其他域名经 Proxy 使用海外 DNS。

## 运行模式与端口

`vif_only: true` 表示只走 VIF（TUN）模式，不接管系统代理；`http_port: 3080` / `socks_port: 3090` 只监听本机（`allow_external_connections: false`）；`compat_route: true` 打开兼容路由，减少路由冲突。

## Proxy 组：Auto 在前，节点直接跟在后

`Proxy` 里只有 `Auto` 一个静态子策略，之后就是全部订阅节点（按名称去重），所以手动挑单个节点只需点开 `Proxy` 一次，不用再进一层汇总组。

顺序不是猜的——官方语义是「`urls` 与 `policies` 混用时，订阅节点排在静态子策略之后」。这条规则是**单向的**：静态项只能排在节点前面，**没法排到节点后面**，所以「`Auto` → 节点 → `DIRECT`/`REJECT`」这种顺序在 Egern 里写不出来。当前取舍是让节点紧跟 `Auto`，直连 / 拒绝改由 `CN` 组切换（那里仍是 `DIRECT` / `Proxy` / `REJECT` 三项）；想把它们加回来就在 `policies` 里补 `- DIRECT` / `- REJECT`，位置会是 `Auto` 之后、节点之前。

这里**故意没有用 `flatten`**：文档说 flatten 会展开「所有」嵌套策略组，`Auto` 会被一起展开、从列表里消失，正好和「第一个是 Auto」冲突。

## 场景分流（SSID 直连）

`场景` 是 `conditional` 组，连上 `YUQI` 或模板 SSID `WiFi2` 时整体走 `DIRECT`，其他网络走 `Proxy`。国内与局域网规则仍使用 `DIRECT` / `CN`，所有海外分类规则以及兜底规则都指向 `场景`，因此连上指定 Wi-Fi 时海外分类流量也交给路由器，其他网络才走代理节点。

`close_connections_on_policy_change: true` 会在策略切换时断开旧连接，使切换及时生效。

## 局域网排除

分两层：

- `vif_excluded_routes` 是**路由级**（只收 CIDR，这些网段压根不进 VIF 路由表），覆盖 `127/8`、`169.254/16`、`100.64/10`、`10/8`、`172.16/12`、`192.168/16`；
- `bypass_tunnel_proxy` 是**目标级**（域名 + CIDR），只留 `*.local` / `*.lan` / `localhost` 这类路由级表达不了的。

当前 `include_all_networks: true` 接管全部网络，这两层是局域网设备不被抓进隧道的唯一保障——访问 `192.168.10.1`、NAS、打印机异常时先查这里。同一个 CIDR 不要同时写进两处：重复声明不报错，但会让人误判到底哪一层在生效。

## 广告拦截与 STUN

`Reject` / `Ads_AWAvenue` 走 `REJECT-NO-DROP`（丢弃不回 RST），与 mihomo 的 `REJECT-DROP` 语义对齐——回 RST 等于告诉广告 SDK「这里有拦截器」，会触发重试甚至上报特征；代价是拦截表现为超时，个别 App 首屏可能慢 1-3 秒，疑似误杀就去看这两条规则的命中日志。

`rules` 第一条是 `protocol: stun → REJECT`，用于防 WebRTC 泄漏真实 IP；这条**故意用 `REJECT` 而不是 `REJECT-NO-DROP`**，因为 WebRTC 需要快速失败才能及时回落 TURN，静默丢弃会让通话建立多等 5-10 秒。重度用 FaceTime / 语音会议的人若发现通话质量下降，先注释这一条。

## 出口 IP 探测

`public_ip_lookup_url: https://ipv4.ddnspod.com` 固定走 IPv4（与 `ipv6: false` 一致），该域名国内可达，所以 `CN` / `DIRECT` 策略下也能查到结果；换成 `ip.sb`、`ipify` 这类境外站点会在这两种策略下超时。

## 模块（MITM 重写）

`modules` 段启用 16 个模块，按用途分五组，排障时可整组禁用。

| 分组 | 模块 | 作用 |
| --- | --- | --- |
| 基础设施 | `Script-Hub` | `script.hub` 是个假域名，靠它在本机把 Conf 转成 sgmodule；关掉它，「开屏广告」「高德去广告」会一起失效 |
| 第一波 | 开屏广告、B站去广告、B站界面增强、YouTube净化、百度网页去广告 | 覆盖率最高的一批，开屏广告覆盖 400+ App；百度那条只作用于 `m.baidu.com` / `www.baidu.com` 的搜索首页，不含网盘与贴吧 |
| 第二波 | 百度地图去广告、高德去广告、夸克去广告、夸克扫描王去广告、网易云音乐增强 | 单 App 界面净化 |
| 第三波 | 微信公众号去广告、淘宝去广告、拼多多去广告、京东去广告比价 | 改响应体，有风控风险 |
| 其他 | missav网页净化 | 网页净化 |

### MITM 相关注意

`mitm` 段的 `ca_p12` / `ca_passphrase` 必须留空，证书由 Egern 自己管理；MITM 域名由各模块自己声明、客户端自动合并（实测合计 31 个具体域名，无通配），**不要在 `mitm` 里写 `*`**。

首次启用会要求安装并信任根证书，装好之前所有模块都不生效。做到这里的代价是：证书固定（pinning）的 App（银行、12306、部分政务/游戏）被 MITM 后会**直接没有任何网络**——出现「某个 App 突然全断」时先禁用对应模块，不要先怀疑规则。

另外，部分模块带有作者未公开含义的 `PR*` 开关，出现界面异常时可先禁用对应模块。微信的两个重写方案会覆盖同一批接口，因此只启用一个；番茄小说的两个模块也不要同时开。百度网页模块只作用于 `m.baidu.com` / `www.baidu.com` 搜索首页，不含网盘与贴吧。

### 模块自带的 `[Rule]` 段

模块会把自身的拦截规则合并进主规则，且多为 `pre-matching`（优先级高于 `rules` 段），所以它不在 `rules` 里出现却能生效。三条需要留意的误伤：

- 拼多多会 REJECT `cdl-1/cdl-p2/cd-1.pddpic.com`（图片 CDN，商品图可能加载不出）；
- 淘宝会 REJECT `ut.taobao.com` / `ossgw.alicdn.com` / `hudong.alicdn.com`（埋点与 CDN）；
- 微信会 REJECT `DOMAIN-SUFFIX,wxs.qq.com`（小程序静态资源域，小程序白屏时第一个怀疑它）。

其余注入规则都是纯广告或埋点域名。`script.hub`（开屏广告、高德两条走的中转）是 Script-Hub 模块伪造的本地域名，**离开客户端用 curl 访问必然失败，这不是配置错误**。

## QUIC

Egern 不是用规则拦截，而是 `block_quic: true`——全局一份，`机场订阅` / `Proxy` / `Auto` 三个组各一份。需要 HTTP/3 时要一次性把这些开关都关掉，只关一个不生效。

## 订阅与过滤

订阅 URL 只在 `机场订阅` 的 `urls` 里写一次（带 YAML 锚点 `&SubURL`），`Proxy` 用 `*SubURL` 复用，**占位串只需要改一处**。

`filter` 正则与其余三端同一套，用来剔除机场信息节点，两个组共用同一份（锚点 `&FilterAll`）。`机场订阅` 设了 `hidden: true`，只作为订阅入口，不在首页策略列表里显示。

`Proxy` 自己也会按 `update_interval` 拉一次订阅（同一间隔、同一份），所以订阅实际被请求两次；这是为了拿到「Auto 在前、节点在后」的顺序——官方文档保证 `urls` 与 `policies` 混用时订阅节点排在静态子策略之后并按名称去重。

## ASN / GEO 库

`ChinaASN` 规则依赖 `asn_db_url`（P3TERX 的 GeoLite2-ASN），`GEOIP` 兜底依赖 `geoip_db_url`（xream 的 ipinfo.country.mmdb）。这两项下载失败时不会报错，只是 ASN / GEOIP 规则静默失效，表现为部分国内 IP 被误走代理；排查分流异常时先确认这两个库能下载。

## APNs 与网络接管范围

官方语义里 `include_apns` / `include_cellular_services` / `include_local_networks` 都是主开关 `include_all_networks` 的**附属项**，主开关为 `false` 时它们不生效。

当前配置是主开关 `true` + `include_apns: true` + 另外两项 `false`：蜂窝下的 APNs 因此能进隧道、命中 `AppleServers` 规则，而 VoLTE/IMS 与局域网流量仍不被接管（`bypass_tunnel_proxy` 再兜一层）。代价是除默认网络外还接管了"全部网络"这一档，遇到热点共享或投屏异常时优先怀疑这里。

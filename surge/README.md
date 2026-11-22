# Surge

[← 返回总览](../README.md) · 配置文件：`surge.conf`

完整配置，可从 URL 导入。本文件覆盖导入步骤，以及 DNS 限制、APNs、测速、广告拦截、规则求值顺序这些 Surge 特有的细节。

## 使用方法

1. 打开 Surge → 首页顶部 / Mac 侧边栏最下方的 [更多] → [从 URL 下载配置]，填：

```text
https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/surge/surge.conf
```

想一键导入（把配置地址交给 Surge 打开）就走安装页顶部那两个按钮：

**<https://2253845067.github.io/Proxy-Config/surge/install.html>**

一个走 `gh-proxy.com` 镜像，一个直连 GitHub。镜像打不开就用直连那条；也可以在 Surge 里直接填原始地址：

```text
https://raw.githubusercontent.com/2253845067/Proxy-Config/main/surge/surge.conf
```

2. **一键导入 `local.dconf`（先装这个，只做一次）**：安装页「配置」区第二排的两个按钮——镜像 / 直连。它是一份只含 `[Proxy Group]` 与 `[Tailscale *]` 两个段的普通配置，会被存进 Profiles 目录；装完在 [更多] → [文件] 里打开它，把 `机场订阅` 那行的占位串 `这里填入你的机场订阅 URL` 换成真实订阅 URL（**只需要改这一处**，`Proxy` / `Auto` 通过 `include-other-group` 复用同一份订阅）。Tailscale 段的登录见下面的 [Tailscale](#tailscale) 一节。

```text
# 不走安装页时，也可以用 Surge 的 URL scheme 直接装
surge:///install-config?url=https%3A%2F%2Fraw.githubusercontent.com%2F2253845067%2FProxy-Config%2Fmain%2Fsurge%2Flocal.dconf
```

   > 代价：`install-config` 装上的是普通 profile，Surge 会给它套用 `#!name` 默认名，所以 Profiles 列表里会出现一份叫 `local.dconf` 的配置。别用它当主配置，也别删除——删掉后 `surge.conf` 引用的段就没了。

3. 导入并切换到 `surge.conf`。`机场订阅` 组在 `local.dconf` 里定义，导入后可在 [策略组] 列表里看到它（隐藏）。`surge.conf` 里的引用写的是 `../Profiles/local.dconf`，如果两台设备上 `local.dconf` 没落在同一层目录导致找不到，改这两个 `#!include` 行的相对路径即可。
4. 更新外部资源，确认 `Proxy` / `Auto` 里能看到节点。若你更习惯在 Surge 里单独添加外部资源 / 订阅，也可以不改 `policy-path`——`include-all-proxies=true` 会把这些手动添加的节点一并收进来（同一套过滤正则仍然生效）。注意：**组内一个节点都没有时 Surge 不会报错，而是把该组回落成 `DIRECT`**（日志里显示 `SUBSTITUTE`），表现就是"全部直连"，看到这个现象先回去查订阅。
5. `场景分流` 已将 `YUQI` 设为直连。若要配置第二个路由器，可把 `"WiFi2"` 改为实际 SSID，并取消 `[SSID Setting]` 中对应模板行的注释、填入路由器 IP；连上该 Wi-Fi 时 DNS 会交给路由器解析。不使用第二个路由器时无需更改模板。
6. 开启 Surge（iOS 用 VPN / 增强模式，Mac 用增强模式或接管系统代理），并在 [工具] 里更新一次规则集与 GeoIP 数据库。

## 更新配置时如何保住本地设置

从 URL 导入的是 **managed profile**，来源是仓库里的 `surge.conf`。只要重新下载，文件就会被整份替换，所以**凡是写进 `surge.conf` 的私人内容，更新后都要重填**。解决办法是把私人内容拆到 `local.dconf`，用 `#!include` 引用（官方 *Detached Profile Section* 机制）：

| 内容 | 更新后会丢吗 | 处理方式 |
|---|---|---|
| 机场订阅 URL（`机场订阅` 组） | 会 | 放 `local.dconf` 的 `[Proxy Group]`，`surge.conf` 里只有 `#!include local.dconf` |
| Tailscale 的 `auth-key` / `interactive-login`（`[Tailscale tailscale]` 段） | 会 | 放 `local.dconf` 的 `[Tailscale *]`，用通配引用整段引入 |
| MITM 的 CA 证书（`[MITM] ca-p12` / `ca-passphrase`） | 会 | `local.dconf` 里留了 `[MITM]` 模板段（默认注释掉），把当前配置里的证书与口令贴进去即可；模块仍然负责 `hostname` |
| 手动添加的外部资源 / 单个节点 | 不会 | 它们存于 Surge 自己的资源列表，`include-all-proxies=true` 会自动收进 Proxy / Auto |
| Tailscale 的**机器身份**（登录会话） | 不会 | 存在 App 本地状态（授权登录按段名、auth-key 按 key 哈希），只要段名仍是 `tailscale`，重新加载配置不用再登录 |
| `[SSID Setting]`、规则、策略组结构 | 会（但本来就在仓库里维护） | 想本机临时改，就把它一起搬进 `local.dconf`（同名段会被 include 的覆盖，别两边重复定义） |

注意事项：

- `local.dconf` 必须放在 **Profiles 目录**、文件名保持 `local.dconf`；`surge.conf` 里用 `#!include ../Profiles/local.dconf` 引用（Surge 按当前 profile 所在目录解析相对路径，所以要从配置文件目录往上一层再进 `Profiles/`）。文件缺失时 `机场订阅` 组不存在、Tailscale 策略不可用（`section-name` 对应的段找不到）。
- managed profile 里被 include 的段在 UI 里是**只读**的，改订阅 / 登录后要看效果，直接编辑 `local.dconf` 文件即可。
- 想把 `[Rule]` 之类也搬到本地：`mixed content`（段内容 + `#!include` 混写）需要 iOS 5.22.0+ / Mac 6.9.0+，`[Tailscale *]` / `[Ruleset *]` 通配引用同样如此。注意**同名段不能重复定义**（Surge 会加载失败），所以搬哪一段，就要把 `surge.conf` 里那段的内容删掉、只留 `#!include`。
- 如果干脆不用本仓库配置，也可以自己做一份本地 profile，把本文件当上游 include 进来（`#!include` 支持远程 URL，Mac 6.0.0+ 的 linked profile），本地那份就能在 UI 里随便改。

## DNS

**防泄漏**：配置只写 IP 形式的国内 DoH（`223.5.5.5` / `1.12.12.12` / `223.6.6.6`），**故意不写明文 `dns-server`**，避免 DoH 主机名还要先走一遍明文 bootstrap；并用 `hijack-dns = *:53` 把应用自己发起的明文 DNS 查询交给 Surge。

`encrypted-dns-follow-outbound-mode = false`，因为这三条 DoH 国内可直连，不必再经 `Proxy`，也可避免和节点解析形成循环依赖（代价是 `PROTOCOL,DOH` / `DOQ` 这类规则不会生效，加密 DNS 查询根本不进规则系统）。

连上透明代理路由器时例外：`[SSID Setting]` 里 `SSID:YUQI dns-server=192.168.77.1,encrypted-dns-server=off` 会把该网络的 DNS 交给路由器，路由器的 DNS 分流 / 广告拦截才会生效；全局已配加密 DNS，所以 `encrypted-dns-server=off` 必须显式写，否则切不回传统 DNS。

**DNS 限制**（相对 mihomo）：Surge 没有 `nameserver-policy`，不能把海外域名的 DoH 经 `Proxy` 出境，因此这里采用和 Loon 相同的思路——全程走国内加密 DNS，而不是 mihomo 那套「国内走 223.5.5.5、海外走 1.1.1.1#Proxy」。

真要给个别域名换解析器，Surge 的出口是 `[Host]` 的 `server:` 映射，例如 `example.com = server:https://1.1.1.1/dns-query`（只对写死的域名生效，不是按规则分流）。

加密 DNS 失败时 Surge 仍可能回落到系统 DNS（与 Loon 类似）；若必须留明文兜底，可取消配置里 `dns-server` 那一行的注释。另外 Surge 不是 mihomo 那种可精细控制的 fake-ip 模式，只用 `always-real-ip` 保证局域网 / Apple 等域名拿到真实 IP；`read-etc-hosts = false`，本机 hosts 不会改解析结果。

## Tailscale

配置里带了一个 Tailscale 节点（`hostname = Proxy-Tool`）：策略行在 `surge.conf` 的 `[Proxy]`，授权段 `[Tailscale tailscale]` 在 `local.dconf`。Surge 的 `tailscale` 必须「策略行 + `[Tailscale <name>]` 段」两段式本地声明，Sub-Store 没有 Surge 的 tailscale 解析 / 输出（只有 Loon 的），托管订阅也塞不进来。

要求 Surge iOS 5.20.0+ / Mac 6.7.0+；交互式登录与自动 MagicDNS 路由要 iOS 5.21.0+ / Mac 6.8.0+。早于后者时只能用 `auth-key`，且没有自动路由，套用下面的手写规则。

**登录（二选一）**

- 交互登录：Surge → 策略 → 选中 `Tailscale` → 策略编辑器里点登录，浏览器授权后 Surge 会把 **`local.dconf` 里那段**改写成 `interactive-login = true`（所以更新配置后不用重新登录，改段名才需要）。此时删掉 `auth-key` 那行，两者互斥。
- `auth-key`：把 `local.dconf` 里 `interactive-login = true` 注释掉，再取消 `#auth-key =` 那行并填真实 key（`tskey-auth-…`，建议一次性、短期、预授权、带 tag）。**这是密钥，不要提交到仓库**；需要交互批准的 key 走 `auth-key` 注册会失败，只能交互登录。Surge 按 key 的哈希存本机身份，换 key 等于换节点。

**不要留占位符**：`auth-key = tskey-auth-替换为你的…` 这类值会被当成真实 key 去注册，Surge 直接报「配置存在错误，请联系配置提供商：[Tailscale *] 字段 auth-key 必须被提供」，整份 profile 都不加载。要么填真 key，要么用交互登录（`local.dconf` 里默认就是 `interactive-login = true`）。

**会自动下发什么**：`auto-add-magic-dns-rule = true` 时，MagicDNS 后缀、每个可见 peer 的 100.x 地址、本机自己的 tailnet 地址都会自动路由到该策略，不用写规则。**子网路由、出口节点不会自动下发**，要自己写：

```ini
IP-CIDR,192.168.1.0/24,Tailscale,no-resolve   # 已在 [Rule] 第 2 段写好
exit-node = auto                              # 可选：tailnet 里恰好一个可用出口节点时
```

子网规则**必须排在 `Lan.list`（内含 `IP-CIDR,192.168.0.0/16`）之前**，否则访问 192.168.1.x 时会被私有网络段判成 DIRECT（见上面的「规则求值顺序」）。

**DNS**：`[Host]` 里的 `ts.net = server:100.100.100.100` 只是兜底，让 MagicDNS 名字不会掉进国内 DoH 解析失败；策略自己下发的解析器与 `*.ts.net` 规则优先。子网设备（不在 MagicDNS 里）的局域网名字用 `[Host]` 映射，如 `nas.lan = 192.168.1.10`，解出来的 192.168.1.x 交给上面的 IP 规则。

**注意**：这是应用层出站策略，不接管系统全局路由，也不替代系统 Tailscale 客户端；`underlying-proxy` 一旦指向代理，peer 流量就只走 DERP 中继（延迟更高）；`allow-peer-access` / `advertise-*` 是让别的节点反向连本机，iOS 端要 6.0.0+。

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

`Reject` / `Ads`（`pre-matching`）→ `IP-CIDR,192.168.1.0/24,Tailscale` → 私有网络（`Lan.list` 带 `no-resolve`）→ `PROTOCOL,STUN,REJECT` → CN 域名段 → Proxy 域名段 → CN IP 兜底（`ChinaASN` / `ChinaIP` / `GEOIP`，**不带** `no-resolve`）→ `FINAL`。

- **不要**去掉 `Lan.list` 的 `no-resolve`：它内部有 8 条 `IP-CIDR`，去掉后每个域名请求求值到那里都会先做一次真实 DNS 解析。
- **也不要**把 CN IP 三层挪到 Proxy 域名段之前：那等于又变回"每个请求都要解析"，而且会把域名表判定该走代理的流量抢成直连。
- **Tailscale 子网规则必须排在 `Lan.list` 之前**：`Lan.list` 第 13 行就是 `IP-CIDR,192.168.0.0/16`（它自己没有 `no-resolve`，`192.168.1.1` 这类地址照样匹配），它又排在第一段私有网络里；`192.168.1.0/24` 若挂在它后面，访问 192.168.1.1 会被判成 DIRECT 走掉。同理也别挪到 `Direct.list` / 显式私网 `IP-CIDR` 之后。它只写 /24，是为了不影响自家路由器（192.168.77.x）的直连。
- `PROTOCOL,STUN,REJECT` 用来减少 WebRTC 泄漏真实 IP（与 Egern / Loon 的 STUN 处理对齐），已挪到私有网络之后让局域网内的 STUN 走直连；FaceTime / 语音会议异常时可先注释它。

## iOS / Mac 共用一份

`read-etc-hosts` 只在 Mac 生效，`allow-wifi-access` / `all-hybrid` 只在 iOS 生效，另一端会直接忽略、不会报错，所以不要为了"两端通用"删掉它们。`icon-url` 需要 Surge Mac 6.5.0+，旧版本只是不显示图标。

## GeoIP / ASN

`GEOIP,CN` 使用 `geoip-maxmind-url` 指向的 Loyalsoldier 国家库（经 `gh-proxy.com` 中转）。

`ChinaASN`（`IP-ASN`）用的是 Surge **自带**的 GeoLite2 ASN 库，只随 App 更新，配置里没有像 Egern `asn_db_url` 那样的更换入口——这是 Surge 的能力边界，不是漏配。库过旧时部分国内 IP 可能被误走代理，优先升级 Surge。

## 本配置不含 MITM 段

`surge.conf` 里没有 `[MITM]`，解密域名（`hostname`）由模块下发——这也是更新配置不会影响解密范围的原因。**CA 证书不放模块里**（官方限制：模块不能改 CA），要么放 `local.dconf` 的 `[MITM]` 段（模板已备好），要么每次更新配置时把旧的 `ca-p12` / `ca-passphrase` 抄回来。证书只存在本地 profile 与系统钥匙串，没有云端备份。

`[MITM]` 段在模块里被改的字段只有 `hostname`、`hostname-disabled`、`client-source-address`、`skip-server-cert-verify`、`tcp-connection`、`h2`；若你在 `local.dconf` 里也写了 `hostname`，它会作为基础值，模块在其上覆盖 / 追加。

## 本配置不含模块 / MITM 证书

没有移植 Loon 插件或 Egern 模块，`[Proxy]`、证书相关字段都是空的。需要去广告脚本或 MITM 时，在客户端自行添加模块（对应关系见下面的[模块对照表](#模块对照表loon-插件--surge-模块)），不要把 `ca-p12` / 订阅写回这个仓库。

## 模块对照表（Loon 插件 → Surge 模块）

本配置本身**不含模块**。下面这张表是把 `loon/loon.lcf` 里启用的插件逐个对上 Surge 模块的结果——这些 Loon 插件绝大部分来自 [kelee.one](https://kelee.one)，而 [surge.qingr.moe](https://surge.qingr.moe)（[LoonKissSurge](https://github.com/QingRex/LoonKissSurge) 项目）做的就是把这套 Loon 插件批量转成 Surge 模块，所以 42 个启用插件里有 40 个能一一对应。

有 5 行不是 LoonKissSurge 的转换结果，需要单独说明：**天气**那两条直接引用上游 [NSRingo/WeatherKit](https://github.com/NSRingo/WeatherKit) 的官方模块（LoonKissSurge 里的转换版是半年前的快照，版本旧、参数也少），**YouTube** 那两条换成上游原生 sgmodule 和它衍生出来的 Maasea 原版（转换版功能不全）。这两组各自都是「二选一」，不能全装，详见表格后的[天气模块](#天气模块不走-loonkisssurge用上游官方地址)与 [YouTube 模块](#youtube-模块gholts-与-maasea-二选一)小节。

这张表只作索引：模块名用来对照 Loon 插件，实际安装全部在上面那个页面里。「对应 Loon 插件」列写的是 `loon/loon.lcf` 里的原始文件名，方便逐条核对。

**安装顺序**：先装表格最上面两条标了「前置依赖」的模块，再装其他的。缺了它们，后面那些去广告模块可能整体不生效。

**装模块请用这个页面：[2253845067.github.io/Proxy-Config/surge/install.html](https://2253845067.github.io/Proxy-Config/surge/install.html)**（源码 `surge/install.html`）。它是真 HTML，46 个模块 + 2 条配置导入做成了可点卡片，带搜索框、前置依赖标红、备选单独分组，点一下直接唤起 Surge 安装。

之所以不用下面这张表直接点：`surge://` 不是 http(s) 协议，**Markdown 渲染器会按协议白名单把这种链接整个丢掉**（GitHub 实测会把 `[x](surge://…)` 渲染成纯文本 `<p>x</p>`，连 `<a>` 都不生成），微信 / GitHub App 的内嵌浏览器也会拦这类跳转。所以**表格里不再放任何 `surge://` 链接**，只留模块名和 Loon 插件名作索引；要装就去那个页面点。

| 分类 | 模块 | 对应 Loon 插件 | 备注 |
| --- | --- | --- | --- |
| 系统基础 | 拦截HTTPDNS | `Block_HTTPDNS` | 前置依赖 |
| 系统基础 | 广告平台拦截器 | `BlockAdvertisers` | 前置依赖 |
| 系统基础 | DNS防泄露 | `Prevent_DNS_Leaks` | ⚠️ 转换后为**空模块**，装了不生效，见下方说明 |
| 系统基础 | 快捷搜索 | `QuickSearch` |  |
| 系统基础 | BoxJs | `BoxJs` |  |
| 系统基础 | Sub-Store | `Sub-Store` |  |
| 系统基础 | Script Hub | `Script-Hub` |  |
| Apple | iRingo WeatherKit（Rewrite） | `iRingo.WeatherKit.Rewrite` | **当前用这个**；纯 URL 重写，不需要 token |
| Apple | iRingo WeatherKit（脚本版） | `iRingo.WeatherKit` | 备选；可接彩云 / 和风 / WAQI，需要 token |
| 微信 | 微信小程序去广告 | `WexinMiniPrograms_Remove_ads` |  |
| 微信 | 微信公众号去广告 | `Weixin_Official_Accounts_remove_ads` |  |
| 微信 | 微信外部链接解锁 | `Weixin_external_links_unlock` |  |
| 社交 | 小红书去广告 | `RedPaper_remove_ads` |  |
| 社交 | Soul 去广告 | `Soul_remove_ads` |  |
| 社交 | 贴吧去广告 | `Tieba_remove_ads` |  |
| 社交 | 酷安去广告 | `CoolApk_remove_ads` |  |
| 电商生活 | 淘宝去广告 | `Taobao_remove_ads` |  |
| 电商生活 | 拼多多去广告 | `PinDuoDuo_remove_ads` |  |
| 电商生活 | 京东去广告 | `JD_remove_ads` |  |
| 电商生活 | 京东外卖去广告 | `JDWaimai_remove_ads` |  |
| 电商生活 | 京东比价 | `JD_Price` |  |
| 电商生活 | 闲鱼去广告 | `FleaMarket_remove_ads` |  |
| 电商生活 | 什么值得买去广告 | `smzdm_remove_ads` |  |
| 电商生活 | 菜鸟去广告 | `Cainiao_remove_ads` |  |
| 出行 | 高德地图去广告 | `Amap_remove_ads` |  |
| 出行 | 滴滴出行去广告 | `DiDi_remove_ads` |  |
| 出行 | 航旅纵横去广告 | `Umetrip_remove_ads` |  |
| 出行 | 12306 去广告 | `12306_remove_ads` |  |
| 影音 | 网易云音乐去广告 | `NeteaseCloudMusic_remove_ads` |  |
| 影音 | 汽水音乐去广告 | `SodaMusic_remove_ads` |  |
| 影音 | 哔哩哔哩去广告 | `Bilibili_remove_ads` |  |
| 影音 | YouTube Enhance（gholts） | `YouTube_remove_ads` | **当前用这个**；去广告 + 后台/PiP/4x/自动高清，**仅 iOS** |
| 影音 | Youtube (Music) Enhance（Maasea） | `YouTube_remove_ads` | 备选；iOS / Mac 都能跑，开销更轻，与上面二选一 |
| 影音 | YouTube 双字幕 | `YouTubeSubtitlesTranslation` |  |
| 阅读 | 书旗小说去广告 | `ShuQiCenterReader_remove_ads` |  |
| 阅读 | 七猫小说去广告 | `SevenCat_remove_ads` |  |
| 阅读 | 得间小说去广告 | `iReaderDejian_remove_ads` |  |
| 阅读 | 番茄小说去广告 | `fanqie.plugin` | 非同一第三方版本 |
| 网盘浏览器 | 百度网盘去广告 | `BaiduNetDisk_remove_ads` |  |
| 网盘浏览器 | 123 云盘去广告 | `123NetWorkDisk_remove_ads` |  |
| 网盘浏览器 | 中国移动云盘去广告 | `mobileClouds_remove_ads` |  |
| 网盘浏览器 | 夸克去广告 | `QuarkBrowser_remove_ads` |  |
| 网盘浏览器 | 夸克扫描王去广告 | `QuarkScanking_remove_ads` |  |
| 网盘浏览器 | 百度搜索网页去广告 | `BaiduSearchWebpage_remove_ads` |  |

模块地址前缀是 `https://gh-proxy.com/https://raw.githubusercontent.com/QingRex/LoonKissSurge/main/Surge/`，镜像失效时把前缀去掉、改用 GitHub 原始地址 `https://raw.githubusercontent.com/QingRex/LoonKissSurge/main/Surge/`。

**`DNS防泄露` 是唯一一个转换后为空的模块**：主目录和 `Beta/` 两个版本下载下来都只有 `#!name` / `#!desc` 之类的元数据，没有任何 `[Rule]` / `[URL Rewrite]` / `[Script]` 段，装了等于没装（原插件写的是 Loon 自己那套 DNS 参数，转换器映射不到 Surge）。Surge 这边的防泄漏本来就由 `surge.conf` 自己做——只写 IP 形式的国内 DoH、不写明文 `dns-server`、`hijack-dns = *:53`，见上面的 [DNS](#dns) 节，不需要模块配合。这一条留着只为和 `loon/loon.lcf` 对齐，嫌碍事可以不装。

### 天气模块（不走 LoonKissSurge，用上游官方地址）

表格里 Apple 那两条不是 LoonKissSurge 的转换结果，直接引用上游 [NSRingo/WeatherKit](https://github.com/NSRingo/WeatherKit) 的官方 Surge 模块，和 `loon/loon.lcf` 里那对 iRingo 插件一一对应：

- **Rewrite 版**（当前使用，对应 Loon 里启用的 `iRingo.WeatherKit.Rewrite.lpx`）：
  `https://github.com/NSRingo/WeatherKit/raw/main/modules/iRingo.WeatherKit.Rewrite.sgmodule`
- **脚本版**（备选，对应 Loon 里禁用的 `iRingo.WeatherKit.lpx`）：
  `https://github.com/NSRingo/WeatherKit/releases/latest/download/iRingo.WeatherKit.sgmodule`

两个版本**只能装一个**——都改写 `weatherkit.apple.com`，同时开会互相覆盖。表格里点开的安装链接走 `gh-proxy.com` 镜像，镜像失效时去掉前缀、换用上面两条原始地址（这个 Surge 版 release 走 `gh-proxy` 是通的，不像 Loon 那个 `.lpx` release 会返 520）。

Rewrite 版的可选端点由参数 `endpoint` 控制，默认 `weatherkit.pages.dev`（作者标注首选、可直连、无需代理），另有 `dev.weatherkit.pages.dev` 与 `weather.nanocat.cloud`（Worker 版，需要代理）。代价是天气请求连同坐标一起经过第三方服务。

装完 Rewrite 版后，`surge.conf` 里那行 `DOMAIN-SUFFIX,weather-analytics-events.apple.com,CN` 会被模块的 `DOMAIN,weather-analytics-events.apple.com,REJECT-DROP` 盖掉（模块规则插在规则列表最顶部）——这正是 Loon 配置注释里想要的效果，**配置不用改**，只是要知道那一行届时成了死规则：删掉模块它会重新生效。

### YouTube 模块（gholts 与 Maasea 二选一）

表格里那两条也不是 LoonKissSurge 的转换结果：gholts 的是上游原生 sgmodule，Maasea 那条是 LoonKissSurge 转出来的。两者同源（gholts 标注 *Based on Maasea's code, Apache-2.0*），都改写 `youtubei.googleapis.com/youtubei/v1/(browse|next|player|search|…)`，**同时开启会对同一个响应跑两遍脚本，必须二选一**。

gholts 那版是超集，多做了四件事：拦 `youtubei/v1/player/ad_break`（服务端广告请求，去广告比纯改响应彻底）、处理 `initplayback` 的 UMP 广告、`auto_hd` 自动选最高分辨率、jump ahead / 屏蔽 store / 屏蔽 AI 摘要等。默认参数即可用，想调在模块参数的 arguments 里改。

**两个需要知道的代价：**

- gholts 版带 `#!requirement=SYSTEM = 'iOS'`，**Mac 上不生效**（模块列表里会显示为不适用）。这份配置 iOS / Mac 共用，所以 Mac 端要么装 Maasea 那条，要么就空着；模块开关状态不跨设备同步，各装各的不会打架。
- gholts 版的 `auto_hd` 会在 `*.googlevideo.com/videoplayback` 上跑 `http-request` 脚本（`requires-body=1` + `binary-body-mode=1`），也就是说**视频分片流量要过一遍 Surge 解密和脚本**。iOS 上看 YouTube 如果明显变慢、发热变高，先把 `auto_hd` 改成 `false`——这条脚本本身带 `#!REQUIREMENT "'{{{auto_hd}}}' == 'true'"`，关掉后 `videoplayback` 就不进脚本了。Maasea 那条只在 `initplayback` 上做 `Map Local`，开销小得多。

顺带：Maasea 那条自带 `AND,((DOMAIN-SUFFIX,googlevideo.com),(PROTOCOL,UDP)),REJECT` 拦截 UDP，和本配置的 `block-quic = all` 重复，但无害。

### 没有直接对应的两个

| Loon 插件 | 替代做法 |
| --- | --- |
| `TelegramRedirect` | Telegram 跳转（[son.zhiaon.top](https://son.zhiaon.top)）——把 `t.me` 链接转到指定客户端 |
| `Node_detection_tool` | 面板·节点信息（[yfamilys.com/surge](https://yfamilys.com/surge)）；LoonKissSurge 里的 `Official/节点测速` 是 NetSpeed，功能不同 |

son.zhiaon.top 上还有一批「Telegram 访问增强 / 优化 Pro」模块，**不建议装**：它们靠成段的 `IP-CIDR` / `IP-ASN` 规则实现，而模块的 Rule 会被插到规则列表最顶部（见下节），等于给每个请求强加一次 DNS 解析，正好破坏本配置「域名规则前置、IP 规则后置」的优化。

### 装模块前必须知道的四件事

1. **模块的 Rule 插在规则列表最顶部**，优先于 `surge.conf` 的 `[Rule]` 求值。纯域名 `REJECT` 的去广告模块没影响；带 `IP-CIDR` / `IP-ASN` 且不带 `no-resolve` 的模块会把「遇到第一条 IP 规则才解析一次」变成「每个请求都解析」，装之前先把模块内容看一遍。
2. **模块只能使用配置里已存在的策略，不能新建策略**，也不能修改 `[Proxy]` / `[Proxy Group]` / `[Rule]` 段。写 `DIRECT` / `REJECT` / `REJECT-TINYGIF` 一定可用；写 `场景分流` 这种你配置里有的策略组也能用，但模块作者没法预知你的组名，所以公开的去广告模块基本是硬 `REJECT`——想让用户自选策略的作者会用 `#!arguments=policy:DIRECT` 加 `{{{policy}}}` 占位，在模块参数里改。反过来，**模块里写了你配置里没有的策略名，Surge 会直接报 unknown policy**，这时要么改模块，要么先在配置里加同名策略组。另外模块的 Rule 只能插在规则列表最顶部，没法指定「插在第几条」。
3. **模块会往 `[Mitm]` 追加 `hostname`**。装得越多解密范围越大，首次「更新外部资源」也越慢。某个 App 出现证书错误 / 网络异常时，先关掉它对应的模块再排查。
4. **想吃到最新特性就用 Beta 版**。主目录故意不加 JQ 重写这类「不订阅就用不了」的特性；把地址里的 `Surge/` 换成 `Surge/Beta/`、文件名末尾加 `.beta`（如 `淘宝去广告.beta.sgmodule`）即可。本配置已经在用 `pre-matching`，说明 Surge 版本够新，可以优先考虑 Beta。

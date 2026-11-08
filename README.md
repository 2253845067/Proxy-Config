# Proxy-Config

个人代理客户端配置模板仓库。

## 目录结构

- `mihomo/`: Mihomo 配置模板。
- `mihomo/mihomo.yaml`: 当前使用的 Mihomo 主配置。
- `loon/`: Loon 配置文件。
- `loon/loon.lcf`: 当前使用的 Loon 主配置。
- `egern/`: Egern 配置文件。
- `egern/egern.yaml`: 当前使用的 Egern 主配置。
- `singbox/`: sing-box 配置模板。
- `singbox/config.json`: sing-box 主配置（1.15+，需使用 reF1nd 核心）。

## Mihomo 使用方法

1. 点击 Sub-Store 前端第二个 [文件] tab，新建 Mihomo 配置文件。
2. 进行配置：设置订阅、添加脚本操作，远程链接填写：

```text
https://raw.githubusercontent.com/2253845067/Proxy-Config/main/mihomo/mihomo.yaml
```

3. 保存后，将生成的链接导入 Mihomo Party 等客户端。
   若预览提示“无法下载脚本”，先检查模板地址是否可访问；`gh-proxy.com` 中转地址可能返回 403，可改用上面的 GitHub 原始地址（Sub-Store 后端需能访问 GitHub）。
4. 在客户端开启 TUN 模式并启用系统代理接管；模板已开启自动路由和严格路由，只有 TUN 真正运行后才能阻止应用绕过内置 DNS。

> **规则来源**：规则集全部取自 [Repcz/Tool](https://github.com/Repcz/Tool)（分支 `X`）对应的 `mihomo/Rules`、`Loon/Rules`、`Egern/Rules`、`sing-box/Rules` 目录，经 gh-proxy 中转引用，上游每天自动重建，本仓库只引用不镜像内容。mihomo / Loon / Egern 各 46 个，sing-box 45 个（上游没有 `ChinaASN.srs`，sing-box 也不支持 ASN 匹配）。原先的 ACL4SSR 大列表（`BanEasyList` 4.1 万条 + `BanEasyPrivacy` 4 万条 + 其余三项，合计约 8.7 万条）已整体下线，换成 `Reject`（6200 条精选）+ `Ads_AWAvenue`（国内 App 净化），内存占用与启动解析量大幅下降。四端共用同一套规则源，不再出现同一域名在不同客户端分流不一致的问题。
>
> 国内判定为四层递进：`ChinaDomain`（1.3 万条域名）→ `ChinaASN`（5076 条运营商 ASN，补 `ChinaIP` 漏掉的国内 IP）→ `ChinaIP`（CIDR 列表）→ `GEOIP,CN` 兜底。`ChinaASN` 依赖真实目标 IP，因此带 `no-resolve`，只在已拿到 IP 的连接上生效。
>
> 规则集为经典文本列表（`classical` / `behavior: classical`），不再依赖 mihomo 1.18+ 的 mrs 格式，旧内核也能跑。
>
> **DNS 说明**：fake-ip 过滤为 `rule` 模式，只对明确标记 `real-ip` 的域名下发真实 IP；海外域名走 `#Proxy` 的 Cloudflare/Google DoH，国内域名与节点域名走国内 DoH（223.5.5.5 / doh.pub）。`use-system-hosts: false` 会忽略本机 hosts，若依赖 hosts 里的自定义解析，请改写进配置的 `hosts:` 段。内置 DNS 监听 `0.0.0.0:1053`（内核默认端口）：53 端口由 TUN 的 `dns-hijack` 接管，因此既不占 53（避免与 Windows ICS/Docker 冲突），也不会因为 `allow-lan: true` 在局域网暴露一个开放解析器。
>
> **QUIC 拦截**：规则里有 `AND,((NETWORK,udp),(DST-PORT,443)),REJECT`，让浏览器回落到 TCP。既然 QUIC 在规则层已被整体拒绝，嗅探段就不再声明 `QUIC` 类型，避免留下永远不执行的分支。若某些应用必须用 HTTP/3，注释掉这一条即可（同时可按需加回 `sniffer.sniff.QUIC`）。

## Loon 使用方法

1. 点击一键导入 [配置](https://www.nsloon.com/openloon/import?sub=https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/loon/loon.lcf)。
2. 导入完成后，点击切换至 [自动分流](https://www.nsloon.com/openloon/flowmodel=filter) 模式。
3. 点击切换代理模式至 [TUN Only](https://www.nsloon.com/openloon/proxymode=tun) 模式，跳转至 Loon 后即为切换成功。
4. 打开 [MitM]、[脚本]、[复写] 三个功能的开关。
5. 打开 [MitM] 里的 [MitM over HTTP/2] 和 [QUIC 回退保护] 开关。
6. 保证 Safari 是默认浏览器的情况下，安装并信任证书。
7. 添加你的订阅。
8. 点击 Loon 底部导航栏的 [配置] -> 右上角的 [...]，打开 [始终开启] 的开关。
9. 打开 Loon 的开关后，点击 [一键更新所有外部资源](https://www.nsloon.com/openloon/update?sub=all)。
10. 待更新完毕之后，回到仪表界面重新打开一次 Loon 的开关即可。

> **DNS 防泄漏**：配置使用 DoH，不再声明系统 DNS 或明文 DNS，并用 `hijack-dns = *:53` 把应用自己发起的明文 DNS 查询交给 Loon 处理。三条 DoH 全部写成 IP 形式（`223.5.5.5` / `1.12.12.12` / `223.6.6.6`）：DoH 主机名本身也要先解析，用域名就等于先依赖一条明文 DNS 做 bootstrap；同时不使用 `cloudflare-dns.com` 这类国内不可达的境外 DoH。Loon 的官方文档说明，加密 DNS 查询失败时仍可能回落到普通 DNS；若要完全禁止回落，请在 Loon 的 DNS 服务器设置中关闭“加密 DNS 失败时使用普通 DNS”。
>
> **Fake IP 例外**：`real-ip` 除 Apple 推送域名外，还包含 `*.lan` / `*.local` / `*.arpa`，否则路由器、NAS、打印机等局域网域名会被解析成 Fake IP（其余三端模板都做了同样的排除）。
>
> **QUIC 拦截**：本地规则里有 `AND,((PROTOCOL,UDP),(DEST-PORT,443)),REJECT`，掐掉 QUIC 让浏览器回落 TCP（与 Mihomo 模板保持一致）。若某些应用必须用 HTTP/3，注释掉这一条即可。
>
> **Apple 推送（APNs / iMessage / FaceTime）**：本地 `[Rule]` 里把 APNs 的域名（`identity.apple.com`、`push.apple.com`、`push-apple.com.akadns.net`）和 Apple 的 `17.x` / IPv6 网段单独指向 `场景分流`，用于解决 iMessage、FaceTime 激活或收发异常。本地规则优先于订阅规则，因此它会盖住订阅里 `Apple.list → CN` 的整体直连；想无条件走代理就把这些行的 `场景分流` 换成 `Proxy`，不需要时整段注释掉。**不要照抄社区规则里的 `DOMAIN-SUFFIX,akadns.net` 和 `DOMAIN-KEYWORD,apple.com.edgekey.net`**：Loon 会按 CNAME 链匹配，而天气（`weather-data/weatherkit/weather-map.apple.com`）、App Store（`apps.apple.com`）、系统更新（`mesu.apple.com`）、iTunes、Support 全部 CNAME 到 `*.akadns.net`，加进来等于把这些流量一起拽到代理上，实测会直接导致 Apple Watch 天气、App Store 加载失败。改完规则后要开关一次飞行模式断开所有连接才会生效；仍不行再到 Loon → 设置 → 隧道，打开「包含所有网络」和「APNs」开关。
>
> **插件与 MITM**：插件自带的 MITM 域名会合并进配置；番茄系去广告插件的域名已在 `[Mitm]` 里显式声明，禁用该插件后记得把这行一起删掉。插件行里的 `pin=true` 只表示“把插件固定在列表前部、不参与自动重排”（官方示例配置用它保护核心防泄漏插件），与是否跟随上游更新无关。`ca-p12` / `ca-passphrase` 留空由 Loon 管理证书，本仓库是公开仓库，切勿填入自己的证书内容。

> **WiFi 分流（可选）**：若你使用透明代理路由器，把配置中的 `WiFi1` / `WiFi2` 替换为路由器 WiFi 名称、`10.0.0.1` / `10.0.0.2` 替换为对应路由器管理 IP。连上这些 WiFi 时 Loon 自动走直连（由路由器代理翻墙），其他网络由 Loon 自动分流，避免双重代理。
>
> 有多个透明代理路由器 WiFi 时：
> - `[Proxy Group]` 的 `场景分流` 组里继续追加 `"SSID"=DIRECT`（如 `"WiFi3"=DIRECT`）；
> - `[Host]` 里每个 WiFi 单独一行 `ssid:SSID = server:路由器IP`（`ssid:` 与 `server:` 后都要顶格写，不能留空格，否则 SSID 名会带前导空格匹配不到）；
> - 普通 WiFi（没有路由器翻墙）不用列出，自动走 default 的 Proxy/Auto。

## Egern 使用方法

1. 点击一键导入 [配置](https://egernapp.com/profiles/new?name=egern&url=https%3A%2F%2Fgh-proxy.com%2Fhttps%3A%2F%2Fraw.githubusercontent.com%2F2253845067%2FProxy-Config%2Fmain%2Fegern%2Fegern.yaml)。
2. 如果无法跳转导入，打开 Egern 底部 [设置] -> 最上面 [配置] -> 右上角 [+号] -> [下载]，URL 填写：

```text
https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/egern/egern.yaml
```

3. 导入完成后，切换并使用 `egern` 配置。
4. 打开 [策略组] -> `机场订阅`，填入你的机场订阅 URL 并保存（该组已从官方归为旧式的 `external` 写法迁移为 `select + urls`，两者等价）。
5. 回到策略组页面更新外部资源，确认 `Proxy` / `Auto` 里能看到节点。
6. 回到仪表界面打开 Egern 开关即可。

> **DNS 注意**：`dns.proxy_nameservers` 的连接按官方语义是**强制直连**、且会绕过 `forward` 规则，所以必须填国内可达的 DoH（当前为 223.5.5.5 / 1.12.12.12）。若填 `8.8.8.8`、`1.1.1.1`，在国内网络下节点域名解析会失败，所有以域名下发的节点会直接不可用。

## sing-box 使用方法

`singbox/config.json` 对应 Mihomo 的 `Proxy` / `Auto` / `CN` 三个策略组，并保留广告拦截、国内直连、海外代理、TUN 和 DNS 分流逻辑。

本配置使用 sing-box_reF1nd 分支的 Provider 功能，请使用对应的 reF1nd 核心运行；官方 sing-box 核心不支持配置中的 `providers` 字段。

机场订阅链接填写在 `singbox/config.json` 的 `providers[0].url`。订阅节点会自动加入 `Proxy` / `Auto` 策略组。

订阅首次加载使用 `provider-download` 直连以避免与 `Proxy` 策略组形成循环依赖；规则集下载使用 `download` 并通过 `Proxy`。请确保订阅地址可直连访问，并在启动前将示例地址替换为真实订阅链接。

配置使用 reF1nd 的并发 DNS 组、Provider HTTP 客户端和 `resolve.match_only`：先匹配域名规则，解析后再匹配 IP 规则，同时保留原始域名用于实际连接。

TUN 入站已启用自动路由和严格路由（`auto_route` / `strict_route`），启动客户端时请授予 TUN/管理员权限；否则系统流量不会被接管。

QUIC 拦截与其余三端保持一致：规则里有一条 `{"network":"udp","port":443,"action":"reject"}`（位于私网直连规则之后），让浏览器回落到 TCP。不需要可整条删掉。

控制面板已启用：`clash_api` 监听 `127.0.0.1:9090`，面板下载到 `external_ui: ui`，启动后打开 `http://127.0.0.1:9090/ui` 即可。注意 sing-box 的 `external_ui_download_url` 只在 `external_ui` 指定的目录**存在且为空**时才会触发下载，两者必须成对出现，只写下载地址不会有任何效果。

`CN` 策略组只有 `direct` / `Proxy` 两个选项：官方已弃用 `block` 出站，REJECT 需通过规则 `action: reject` 实现，因此不再往组里塞 REJECT（这点与 Mihomo / Egern 的 `CN` 组略有不同）。

规则集使用 sing-box 原生 `.srs` 文件，同样来自 Repcz/Tool 的 `sing-box/Rules/`，首次启动时经 `ghfast.top` 下载。若运行时提示规则集解析失败，说明本地 reF1nd 核心版本落后于规则集格式，优先升级核心，不要改回旧的 SagerNet 规则集（域名覆盖差很多）。导入前可用对应的 reF1nd 核心检查配置：

```text
sing-box check -c singbox/config.json
```

相关参考：

- [sing-box_reF1nd 核心仓库](https://github.com/reF1nd/sing-box)
- [resolve.match_only 特性说明与配置示例](https://gist.github.com/CHIZI-0618/35f59df7b17bf66ea988d775aaf76152)

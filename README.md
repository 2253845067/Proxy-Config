# Proxy-Config

个人代理客户端配置模板仓库。

四份配置共用同一套规则集（Repcz/Tool `X` 分支）、同一套策略组命名（`Proxy` / `Auto` / `CN`，Loon 多一个 `场景分流`、Egern 多一个 `场景`）和一致的 DNS / QUIC 处理思路；剩下的差异只来自客户端能力边界，逐项列在下面的「四端差异」表里。

## 目录结构

- `mihomo/`：Mihomo 配置模板（Sub-Store 远程模板，文件里不含节点）。
- `mihomo/mihomo.yaml`：当前使用的 Mihomo 主配置。
- `loon/`：Loon 配置文件（完整配置，可直接一键导入）。
- `loon/loon.lcf`：当前使用的 Loon 主配置。
- `egern/`：Egern 配置文件（完整配置，可直接一键导入）。
- `egern/egern.yaml`：当前使用的 Egern 主配置。
- `singbox/`：sing-box 配置模板（完整配置，仅 reF1nd 分支核心可用）。
- `singbox/config.json`：sing-box 主配置。
- `.gitignore`：忽略 `.DS_Store` 与本地 `.workbuddy/` 工作区数据。
- `.gitattributes`：统一按文本处理，提交时做 LF 归一化，避免 Windows 换行符把配置改成 CRLF。

## 通用说明

### 节点从哪里来

| 客户端 | 节点接入方式 | 你需要改的地方 |
| --- | --- | --- |
| Mihomo | 节点由 Sub-Store 输出时注入，模板本身不写 `proxies` / `proxy-providers` | 在 Sub-Store 里配订阅与脚本操作（见下文） |
| Loon | `AllNodes` 来自 Loon 内添加的订阅 | Loon 内添加机场订阅 |
| Egern | `机场订阅` 组的 `urls` | 把占位串换成真实订阅 URL（只需改这一处，`Proxy` 用 YAML 锚点复用） |
| sing-box | `providers[0].url` | 把占位串换成真实订阅 URL |

Egern 与 sing-box 仓库文件里写的是占位串（`这里填入你的机场订阅 URL` / `https://example.com/your-subscription`），**不替换就会一直加载失败**，这不是配置残缺。

### 端口与监听

| 客户端 | 监听 | 说明 |
| --- | --- | --- |
| Mihomo | `7890`（mixed）、`0.0.0.0:1053`（DNS） | `allow-lan: true` + `bind-address: "*"`，局域网内其他设备可用这两个端口，且没有认证 |
| sing-box | `7890`（mixed）、`127.0.0.1:9090`（面板）、TUN `172.19.0.1/30` | 与 Mihomo 抢同一个 7890，**同一台机器不要同时启用** |
| Egern | `3080`（http）、`3090`（socks） | 只监听本机，`allow_external_connections: false` |
| Loon | 无固定端口 | 走 Network Extension 隧道 |

### 下载镜像

四端引用的规则集、图标、面板都经第三方 GitHub 镜像：

- mihomo / Loon / Egern：`https://gh-proxy.com/https://raw.githubusercontent.com/...`
- sing-box：`https://ghfast.top/https://github.com/Repcz/Tool/raw/X/sing-box/Rules/*.srs`

镜像失效时整体替换即可（两种镜像的前缀形式相同，可以直接互换）：

```bash
grep -rl 'gh-proxy.com' mihomo loon egern | xargs sed -i 's#gh-proxy.com#ghfast.top#g'
```

Sub-Store 拉取模板这一步例外：请用 GitHub 原始地址，`gh-proxy.com` 对某些路径会返回 403。

### 公开仓库提醒

配置里不含任何节点、订阅链接或证书内容。`ca-p12` / `ca-passphrase` / `secret` 一律留空，由客户端自己管理，**不要把证书或订阅写进这个仓库**。

## 规则源与分流逻辑（四端通用）

> **规则来源**：规则集全部取自 [Repcz/Tool](https://github.com/Repcz/Tool)（分支 `X`）对应的 `mihomo/Rules`、`Loon/Rules`、`Egern/Rules`、`sing-box/Rules` 目录，经镜像中转引用，上游每天自动重建，本仓库只引用不镜像内容。mihomo / Loon / Egern 各 46 个，sing-box 45 个（上游没有 `ChinaASN.srs`，官方 sing-box 也不支持 ASN 匹配）。原先的 ACL4SSR 大列表（`BanEasyList` 4.1 万条 + `BanEasyPrivacy` 4 万条 + 其余三项，合计约 8.7 万条）已整体下线，换成 `Reject`（6200 条精选）+ `Ads_AWAvenue`（国内 App 净化），内存占用与启动解析量大幅下降。
>
> 规则集**同源**，但本地补充规则**不完全相同**，看分流结果时不要只看规则集：Loon 额外强制两条业务域名直连（`report-online.sh.wxgateway.com`、`diancan.xibeilangsaoma.com`），同时少了其他三端都写死的 `weather.apple.com` / `weather-analytics-events.apple.com → CN`——Loon 上这两个域名会落到 `场景分流`（默认代理），这是当前**有意保留**的差异；需要对齐时自己在 `[Rule]` 里补两行 `DOMAIN-SUFFIX` 即可。
>
> **国内判定**：mihomo / Loon / Egern 是四层递进——`ChinaDomain`（1.3 万条域名）→ `ChinaASN`（5076 条运营商 ASN，补 `ChinaIP` 漏掉的国内 IP）→ `ChinaIP`（CIDR 列表）→ `GEOIP,CN` 兜底。`ChinaASN` 依赖真实目标 IP，mihomo 里显式带 `no-resolve`，只在已拿到 IP 的连接上生效。sing-box 只有三层（`ChinaDomain` → `ChinaIP` → 末条 `final: Proxy`），没有 ASN 和 GEOIP 兜底。
>
> **规则格式**：mihomo 用文本 `classical` 列表（`behavior: classical` + `format: text`），不依赖 mrs 格式，旧内核也能跑；Loon 是文本 list；Egern 是 yaml 规则集；sing-box 是编译后的 `.srs` 二进制。

### 四端差异

| 项目 | Mihomo | Loon | Egern | sing-box |
| --- | --- | --- | --- | --- |
| 策略组 | `Proxy` / `Auto` / `CN` | 同 + `场景分流` | `场景` / `Proxy` / `Auto` / `CN`（+ 隐藏的 `机场订阅`） | `Proxy` / `Auto` / `CN` |
| 广告拦截 | `REJECT-DROP` | `REJECT` | `REJECT-NO-DROP` | `action: reject` |
| QUIC 拦截 | 规则 `AND,((NETWORK,udp),(DST-PORT,443))` | 规则 `AND,((PROTOCOL,UDP),(DEST-PORT,443))` | `block_quic: true` 开关（不是规则） | 规则 `{"network":"udp","port":443}` |
| 国内判定 | 四层 | 四层 | 四层 | 三层（无 ASN） |
| `CN` 组可选项 | `DIRECT` / `Proxy` / `REJECT` | 同 | 同 | `direct` / `Proxy` |
| 规则格式 | classical text | text | yaml | srs |

## Mihomo 使用方法

1. 点击 Sub-Store 前端第二个 [文件] tab，新建 Mihomo 配置文件。
2. 进行配置：设置订阅、添加脚本操作，远程链接填写：

```text
https://raw.githubusercontent.com/2253845067/Proxy-Config/main/mihomo/mihomo.yaml
```

3. 保存后，将生成的链接导入 Mihomo Party 等客户端。
   若预览提示“无法下载脚本”，先检查模板地址是否可访问；`gh-proxy.com` 中转地址可能返回 403，可改用上面的 GitHub 原始地址（Sub-Store 后端需能访问 GitHub）。
4. 在客户端开启 TUN 模式并启用系统代理接管；模板已开启自动路由和严格路由，只有 TUN 真正运行后才能阻止应用绕过内置 DNS。TUN 栈用的是 `mips`（mihomo 1.19.x 起的官方默认栈，用户态、开销低、不需要放行防火墙）；内核较旧不认这个值时改成 `mixed`。

> **策略组**：`Proxy` 手动选择（内含 `Auto` 与全部订阅节点）、`Auto` 自动测速（300 秒一轮、容差 50ms）、`CN` 默认 `DIRECT`（可切 `Proxy` / `REJECT`）。节点由 Sub-Store 注入，`filter` 正则会把「群 / 剩余流量 / 到期 / 官网」这类机场信息节点剔除；广告规则统一走 `REJECT-DROP`（丢弃不回 RST，减少被探测）。

> **DNS / hosts**：`fake-ip` 过滤为 `rule` 模式，只对明确标记 `real-ip` 的域名下发真实 IP；海外域名走 `#Proxy` 的 Cloudflare/Google DoH，国内域名与节点域名走国内 DoH（223.5.5.5 / doh.pub）。内网域名（`rule-set:Lan`）交给 `system` 解析，路由器/NAS 域名才能通。配置里同时关了 `use-system-hosts` 和 `use-hosts`：前者忽略本机 hosts，后者忽略配置内的 `hosts:` 段——所以**现在往配置里加 `hosts:` 段不会生效**。要恢复静态映射，需要把 `use-hosts` 改回 `true` 并新增 `hosts:` 段；要走本机 hosts 则需同时把 `use-system-hosts` 改回 `true`；也可以不改开关，直接用 `nameserver-policy` 或 `DOMAIN` 规则解决。

> **DNS 端口**：内置 DNS 监听 `0.0.0.0:1053`（内核默认端口）。53 端口由 TUN 的 `dns-hijack` 接管，因此不占 53，不会和 Windows ICS / Docker 抢端口。但要注意 `allow-lan: true` 下 `0.0.0.0:1053` 对局域网是开放的（同样开放的还有 `7890`，且都不带认证）；不需要给局域网用时，把 `listen` 改成 `127.0.0.1:1053`。

> **QUIC 拦截**：规则里有 `AND,((NETWORK,udp),(DST-PORT,443)),REJECT`，位置在局域网 / 直连规则之后，让浏览器回落到 TCP。既然 QUIC 在规则层已被整体拒绝，嗅探段就不再声明 `QUIC` 类型，避免留下永远不执行的分支。若某些应用必须用 HTTP/3，注释掉这一条即可（同时可按需加回 `sniffer.sniff.QUIC`）。

## Loon 使用方法

1. 点击一键导入 [配置](https://www.nsloon.com/openloon/import?sub=https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/loon/loon.lcf)。
   链接里的 `sub=` 参数没有做百分号编码，个别环境会把内层 URL 截断；打不开时改用 Loon → [配置] → 右上角 [+]，直接从 URL 下载，填 `https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/loon/loon.lcf`。
2. 导入完成后，点击切换至 [自动分流](https://www.nsloon.com/openloon/flowmodel=filter) 模式。
3. 点击切换代理模式至 [TUN Only](https://www.nsloon.com/openloon/proxymode=tun) 模式，跳转至 Loon 后即为切换成功。
4. 打开 [MitM]、[脚本]、[复写] 三个功能的开关（这三项是给插件用的，`loon.lcf` 的 `[Rewrite]` / `[Script]` 段本身是空的）。
5. 打开 [MitM] 里的 [MitM over HTTP/2] 和 [QUIC 回退保护] 开关。
6. 保证 Safari 是默认浏览器的情况下，安装并信任证书。
7. 添加你的订阅。
8. 点击 Loon 底部导航栏的 [配置] -> 右上角的 [...]，打开 [始终开启] 的开关。
9. 打开 Loon 的开关后，点击 [一键更新所有外部资源](https://www.nsloon.com/openloon/update?sub=all)。
10. 待更新完毕之后，回到仪表界面重新打开一次 Loon 的开关即可。

> **DNS 防泄漏**：配置使用 DoH，不再声明系统 DNS 或明文 DNS，并用 `hijack-dns = *:53` 把应用自己发起的明文 DNS 查询交给 Loon 处理。三条 DoH 全部写成 IP 形式（`223.5.5.5` / `1.12.12.12` / `223.6.6.6`）：DoH 主机名本身也要先解析，用域名就等于先依赖一条明文 DNS 做 bootstrap；同时不使用 `cloudflare-dns.com` 这类国内不可达的境外 DoH。Loon 的官方文档说明，加密 DNS 查询失败时仍可能回落到普通 DNS；若要完全禁止回落，请在 Loon 的 DNS 服务器设置中关闭“加密 DNS 失败时使用普通 DNS”。

> **其他关键项**：`udp-fallback-mode = REJECT`（节点不支持 UDP 时直接拒绝，而不是回落到直连，否则 UDP 会绕开代理）、`disable-stun = true`（直连时丢弃 STUN）、`dns-reject-mode = LOOPBACKIP` + `domain-reject-mode = DNS`（拦截回环 IP，避免部分 App 立刻重试；官方取值是大写 `LOOPBACKIP` / `NOANSWER` / `NXDOMAIN`）、`allow-wifi-access = false`（不向局域网共享代理）、`ip-mode = dual` + `ipv6-vif = auto`。`skip-proxy` / `bypass-tun` 覆盖全部 RFC1918 与保留网段，所以局域网设备流量根本不进 Loon。

> **规则求值顺序**：Loon 是「本地 `[Rule]` → `[Remote Rule]`」，全部未命中时才回本地执行 `FINAL`；本地规则优先于订阅规则。所以 `[Rule]` 里的 APNs 段和 QUIC 拦截能盖住订阅里的同名判断，`FINAL,场景分流` 放在 `[Rule]` 末尾是正确位置——**不要**挪到 `[Remote Rule]` 后面，那样会让远程规则提前终止。

> **Fake IP 例外**：`real-ip` 除 Apple 推送域名外，还包含 `*.lan` / `*.local` / `*.arpa`，否则路由器、NAS、打印机等局域网域名会被解析成 Fake IP（其余三端模板都做了同样的排除）。

> **QUIC 拦截**：本地规则里有 `AND,((PROTOCOL,UDP),(DEST-PORT,443)),REJECT`，掐掉 QUIC 让浏览器回落 TCP（与其余三端同一思路）。Loon 的拦截写在本地 `[Rule]` 里，优先级高于订阅规则，但局域网流量已被 `bypass-tun` 排除在隧道外，实际效果与其他三端一致。若某些应用必须用 HTTP/3，注释掉这一条即可。

> **Apple 推送（APNs / iMessage / FaceTime）**：本地 `[Rule]` 里把 APNs 的域名（`identity.apple.com`、`push.apple.com`、`push-apple.com.akadns.net`）和 Apple 的 5 个 `17.x` 网段 + 4 个 IPv6 `/48` 网段单独指向 `场景分流`，用于解决 iMessage、FaceTime 激活或收发异常。本地规则优先于订阅规则，因此它会盖住订阅里 `Apple.list → CN` 的整体直连；想无条件走代理就把这些行的 `场景分流` 换成 `Proxy`，不需要时整段注释掉。**不要照抄社区规则里的 `DOMAIN-SUFFIX,akadns.net` 和 `DOMAIN-KEYWORD,apple.com.edgekey.net`**：Loon 会按 CNAME 链匹配，而天气（`weather-data/weatherkit/weather-map.apple.com`）、App Store（`apps.apple.com`）、系统更新（`mesu.apple.com`）、iTunes、Support 全部 CNAME 到 `*.akadns.net`，加进来等于把这些流量一起拽到代理上，实测会直接导致 Apple Watch 天气、App Store 加载失败。改完规则后要开关一次飞行模式断开所有连接才会生效；仍不行再到 Loon → 设置 → 隧道，打开「包含所有网络」和「APNs」开关。

> **插件与 MITM**：插件自带的 MITM 域名会合并进配置；番茄系去广告插件的域名已在 `[Mitm]` 里显式声明，禁用该插件后记得把这行一起删掉。插件行里的 `pin=true` 只表示“把插件固定在列表前部、不参与自动重排”（官方示例配置用它保护核心防泄漏插件），与是否跟随上游更新无关。`ca-p12` / `ca-passphrase` 留空由 Loon 管理证书，本仓库是公开仓库，切勿填入自己的证书内容。

> **天气增强插件（iRingo:WeatherKit）**：提供 Rewrite 版和脚本版两个文件，**二选一，不要同时启用**（都会 MITM `weatherkit.apple.com`，同时开等于同一批请求被改两次）。当前启用的是 Rewrite 版，脚本版保持禁用：Rewrite 版把 `weatherkit.apple.com` 的 API 请求（含定位坐标）重写到第三方中转 `weatherkit.pages.dev`，该域名在 Cloudflare Pages 上、需走代理才通，且依赖别人的服务可用性；脚本版在本地改写响应体，配彩云 / 和风 / WAQI 的 token 才有最佳效果，脚本从 GitHub releases 拉取（该 release 走 `gh-proxy.com` 会返回 520，所以用的是 `ghfast.top`）。旧的 `AppleWeatherEnhancer.lpx` 是同类插件，保持禁用即可。

> **WiFi 分流（可选）**：若你使用透明代理路由器，把配置中的 `WiFi1` / `WiFi2` 替换为路由器 WiFi 名称、`10.0.0.1` / `10.0.0.2` 替换为对应路由器管理 IP。连上这些 WiFi 时 Loon 自动走直连（由路由器代理翻墙），其他网络由 Loon 自动分流，避免双重代理。**不用透明代理路由器就把 `"WiFi1"=DIRECT` / `"WiFi2"=DIRECT` 和 `[Host]` 里对应两行一起删掉**，否则遇到同名 SSID（公共热点叫 `WiFi1` 这种）会被误判成直连。
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
4. 打开 [策略组] -> `机场订阅`，把 `urls` 里的占位串 `这里填入你的机场订阅 URL` 换成你的机场订阅 URL 并保存（该组已从旧的 `external` 写法迁移为官方推荐的 `select + urls`，两者等价）。**只需要改这一处**：`Proxy` 组用的是同一个 YAML 锚点 `*SubURL`，会自动跟着变。
5. 打开 [策略组] -> `场景`，把 `WiFi1` / `WiFi2` 换成透明代理路由器的真实 SSID（详见下文「场景分流」；不用透明代理路由器就把这个组删掉，并把 `rules` 末尾 `default` 的 `policy` 改回 `Proxy`）。
6. 回到策略组页面更新外部资源，确认 `Proxy` / `Auto` 里能看到节点。
7. 回到仪表界面打开 Egern 开关即可。

> **DNS 注意**：`dns.proxy_nameservers` 的连接按官方语义是**强制直连**、且会绕过 `forward` 规则，所以必须填国内可达的 DoH（当前为 223.5.5.5 / 1.12.12.12）。若填 `8.8.8.8`、`1.1.1.1`，在国内网络下节点域名解析会失败，所有以域名下发的节点会直接不可用。`bootstrap`（223.5.5.5 / 119.29.29.29）只用于解析 DoH 主机名，同样必须是国内可达的地址。

> **运行模式与端口**：`vif_only: true` 表示只走 VIF（TUN）模式，不接管系统代理；`http_port: 3080` / `socks_port: 3090` 只监听本机（`allow_external_connections: false`）；`compat_route: true` 打开兼容路由，减少路由冲突。

> **Proxy 组：Auto 在前，节点直接跟在后**：`Proxy` 里只有 `Auto` 一个静态子策略，之后就是全部订阅节点（按名称去重），所以手动挑单个节点只需点开 `Proxy` 一次，不用再进一层汇总组。顺序不是猜的——官方语义是「`urls` 与 `policies` 混用时，订阅节点排在静态子策略之后」。这条规则是**单向的**：静态项只能排在节点前面，**没法排到节点后面**，所以「`Auto` → 节点 → `DIRECT`/`REJECT`」这种顺序在 Egern 里写不出来。当前取舍是让节点紧跟 `Auto`，直连 / 拒绝改由 `CN` 组切换（那里仍是 `DIRECT` / `Proxy` / `REJECT` 三项）；想把它们加回来就在 `policies` 里补 `- DIRECT` / `- REJECT`，位置会是 `Auto` 之后、节点之前。这里**故意没有用 `flatten`**：文档说 flatten 会展开「所有」嵌套策略组，`Auto` 会被一起展开、从列表里消失，正好和「第一个是 Auto」冲突。

> **场景分流（SSID 直连）**：`场景` 是 `conditional` 组——连上 `WiFi1` / `WiFi2` 时整体走 `DIRECT`，其他网络走 `Proxy`，而 `rules` 末尾的 `default` 指向它，于是「在家交给路由器、出门自己扛」变成一次开关级切换。两个 SSID 都是**占位名，必须替换**：SSID 是精确匹配，留着 `WiFi1` 不但不生效，遇到同名公共热点还会把流量误判成直连（Loon 端 `[Proxy Group]` 里的同名占位有一模一样的坑）。这一步依赖 `close_connections_on_policy_change: true`——没有它，切到 `DIRECT` 之后旧长连接仍会继续走代理节点。

> **局域网排除**：分两层。`vif_excluded_routes` 是**路由级**（只收 CIDR，这些网段压根不进 VIF 路由表），覆盖 `127/8`、`169.254/16`、`100.64/10`、`10/8`、`172.16/12`、`192.168/16`；`bypass_tunnel_proxy` 是**目标级**（域名 + CIDR），只留 `*.local` / `*.lan` / `localhost` 这类路由级表达不了的。当前 `include_all_networks: true` 接管全部网络，这两层是局域网设备不被抓进隧道的唯一保障——访问 `192.168.10.1`、NAS、打印机异常时先查这里。同一个 CIDR 不要同时写进两处：重复声明不报错，但会让人误判到底哪一层在生效。

> **广告拦截与 STUN**：`Reject` / `Ads_AWAvenue` 走 `REJECT-NO-DROP`（丢弃不回 RST），与 mihomo 的 `REJECT-DROP` 语义对齐——回 RST 等于告诉广告 SDK「这里有拦截器」，会触发重试甚至上报特征；代价是拦截表现为超时，个别 App 首屏可能慢 1-3 秒，疑似误杀就去看这两条规则的命中日志。`rules` 第一条是 `protocol: stun → REJECT`，用于防 WebRTC 泄漏真实 IP；这条**故意用 `REJECT` 而不是 `REJECT-NO-DROP`**，因为 WebRTC 需要快速失败才能及时回落 TURN，静默丢弃会让通话建立多等 5-10 秒。重度用 FaceTime / 语音会议的人若发现通话质量下降，先注释这一条。

> **出口 IP 探测**：`public_ip_lookup_url: https://ipv4.ddnspod.com` 固定走 IPv4（与 `ipv6: false` 一致），该域名国内可达，所以 `CN` / `DIRECT` 策略下也能查到结果；换成 `ip.sb`、`ipify` 这类境外站点会在这两种策略下超时。

> **模块（MITM 重写）**：`modules` 段启用 16 个模块，按用途分五组写在配置注释里，**排障时整组注释，不要逐个试**：

| 分组 | 模块 | 作用 |
| --- | --- | --- |
| 基础设施 | `Script-Hub` | `script.hub` 是个假域名，靠它在本机把 Conf 转成 sgmodule；关掉它，「开屏广告」「高德去广告」会一起失效 |
| 第一波 | 开屏广告、B站去广告、B站界面增强、YouTube净化、百度网页去广告 | 覆盖率最高的一批，开屏广告覆盖 400+ App；百度那条只作用于 `m.baidu.com` / `www.baidu.com` 的搜索首页，不含网盘与贴吧 |
| 第二波 | 百度地图去广告、高德去广告、夸克去广告、夸克扫描王去广告、网易云音乐增强 | 单 App 界面净化 |
| 第三波 | 微信公众号去广告、淘宝去广告、拼多多去广告、京东去广告比价 | 改响应体，有风控风险 |
| 其他 | missav网页净化 | 网页净化 |

> **MITM 相关注意**：`mitm` 段的 `ca_p12` / `ca_passphrase` 必须留空，证书由 Egern 自己管理；MITM 域名由各模块自己声明、客户端自动合并（实测合计 31 个具体域名，无通配），**不要在 `mitm` 里写 `*`**。首次启用会要求安装并信任根证书，装好之前所有模块都不生效。做到这里的代价是：证书固定（pinning）的 App（银行、12306、部分政务/游戏）被 MITM 后会**直接没有任何网络**——出现「某个 App 突然全断」时先禁用对应模块，不要先怀疑规则。另外这批模块只保留 GitHub 源（经 `gh-proxy`）与 `script.hub` 中转，B 配置里那些挂在个人域名（`loon.103516.xyz`、`yfamilys.com`、`mock.parsingbook.workers.dev`、`reven.jsforbaby.workers.dev`、`one-api.zzxu.de`）上的模块都没有引入：它们挂掉时是静默失效，不报错，只会某天功能突然回来。

> **模块自带的 `[Rule]` 段**：模块会把自身的拦截规则合并进主规则，且多为 `pre-matching`（优先级高于 `rules` 段），所以它不在 `rules` 里出现却能生效。三条需要留意的误伤：拼多多会 REJECT `cdl-1/cdl-p2/cd-1.pddpic.com`（图片 CDN，商品图可能加载不出）、淘宝会 REJECT `ut.taobao.com` / `ossgw.alicdn.com` / `hudong.alicdn.com`（埋点与 CDN）、微信会 REJECT `DOMAIN-SUFFIX,wxs.qq.com`（小程序静态资源域，小程序白屏时第一个怀疑它）。其余注入规则都是纯广告或埋点域名。`script.hub`（开屏广告、高德两条走的中转）是 Script-Hub 模块伪造的本地域名，**离开客户端用 curl 访问必然失败，这不是配置错误**。

> **QUIC**：Egern 不是用规则拦截，而是 `block_quic: true`——全局一份，`机场订阅` / `Proxy` / `Auto` 三个组各一份。需要 HTTP/3 时要一次性把这些开关都关掉，只关一个不生效。

> **订阅与过滤**：订阅 URL 只在 `机场订阅` 的 `urls` 里写一次（带 YAML 锚点 `&SubURL`），`Proxy` 用 `*SubURL` 复用，**占位串只需要改一处**。`filter` 正则与其余三端同一套，用来剔除机场信息节点，两个组共用同一份（锚点 `&FilterAll`）。`机场订阅` 设了 `hidden: true`，只作为订阅入口，不在首页策略列表里显示。`Proxy` 自己也会按 `update_interval` 拉一次订阅（同一间隔、同一份），所以订阅实际被请求两次；这是为了拿到「Auto 在前、节点在后」的顺序——官方文档保证 `urls` 与 `policies` 混用时订阅节点排在静态子策略之后并按名称去重。

> **ASN / GEO 库**：`ChinaASN` 规则依赖 `asn_db_url`（P3TERX 的 GeoLite2-ASN），`GEOIP` 兜底依赖 `geoip_db_url`（xream 的 ipinfo.country.mmdb）。这两项下载失败时不会报错，只是 ASN / GEOIP 规则静默失效，表现为部分国内 IP 被误走代理；排查分流异常时先确认这两个库能下载。

> **APNs 与网络接管范围**：官方语义里 `include_apns` / `include_cellular_services` / `include_local_networks` 都是主开关 `include_all_networks` 的**附属项**，主开关为 `false` 时它们不生效。当前配置是主开关 `true` + `include_apns: true` + 另外两项 `false`：蜂窝下的 APNs 因此能进隧道、命中 `AppleServers` 规则，而 VoLTE/IMS 与局域网流量仍不被接管（`bypass_tunnel_proxy` 再兜一层）。代价是除默认网络外还接管了"全部网络"这一档，遇到热点共享或投屏异常时优先怀疑这里。

## sing-box 使用方法

`singbox/config.json` 对应 Mihomo 的 `Proxy` / `Auto` / `CN` 三个策略组（`Proxy` 默认 `Auto`、`CN` 默认 `direct`），并保留广告拦截、国内直连、海外代理、TUN 和 DNS 分流逻辑。

本配置使用 sing-box_reF1nd 分支的 Provider 功能，**必须用 reF1nd 核心运行**；文件里的 `$schema` 也指向 `reF1nd/sing-box` 的 `reF1nd-testing-next` 分支文档。官方 sing-box 核心会因 7 处 fork 专属字段直接启动失败：`providers`、`dns.servers[].type: group`（两处）、outbound 的 `providers` / `use_all_providers`、`route.rules[].match_only`、`experimental.urltest_unified_delay`。

机场订阅链接填写在 `singbox/config.json` 的 `providers[0].url`；`exclude` 正则与其余三端同一套，用于剔除机场信息节点。订阅节点会自动加入 `Proxy` / `Auto` 策略组。

订阅首次加载使用 `provider-download` 直连以避免与 `Proxy` 策略组形成循环依赖；规则集下载使用 `download` 并通过 `Proxy`。请确保订阅地址可直连访问，并在启动前将示例地址替换为真实订阅链接。

配置使用 reF1nd 的并发 DNS 组、Provider HTTP 客户端和 `resolve.match_only`：先匹配域名规则，解析后再匹配 IP 规则，同时保留原始域名用于实际连接。`default_domain_resolver: cn` 决定未被 DNS 规则命中的域名走国内 DoH 解析，只有明确要出海的域名才交给 `remote`（8.8.8.8 / 1.1.1.1，经 `Proxy`）。

TUN 入站已启用自动路由和严格路由（`auto_route` / `strict_route`），地址为 `172.19.0.1/30`、接口名 `sing-box`；启动客户端时请授予 TUN/管理员权限，否则系统流量不会被接管。`mixed` 入站监听 `7890` 但 `set_system_proxy: false`，即不自动改系统代理，需要时手动开或只用 TUN——注意这个端口和 Mihomo 的 `mixed-port` 相同，两个客户端不能同时运行。

QUIC 拦截与其他三端一致：规则里有一条 `{"network":"udp","port":443,"action":"reject"}`（位于私网直连规则之后），让浏览器回落到 TCP。不需要可整条删掉。Egern 走的是 `block_quic` 开关，不是规则。

控制面板已启用：`clash_api` 监听 `127.0.0.1:9090`，`external_ui` 指定为 `ui`，面板本体是 metacubexd（经 `ghfast.top` 下载）。启动后打开 `http://127.0.0.1:9090/ui` 即可。注意 sing-box 的 `external_ui_download_url` 只在 `external_ui` 指定的目录**存在且为空**时才会触发下载，两者必须成对出现，只写下载地址不会有任何效果。

`CN` 策略组只有 `direct` / `Proxy` 两个选项：官方已弃用 `block` 出站，REJECT 需通过规则 `action: reject` 实现，因此不再往组里塞 REJECT（这点与 Mihomo / Egern 的 `CN` 组略有不同）。

规则集使用 sing-box 原生 `.srs` 文件，同样来自 Repcz/Tool 的 `sing-box/Rules/`，首次启动时经 `ghfast.top` 下载。若运行时提示规则集解析失败，说明本地 reF1nd 核心版本落后于规则集格式，优先升级核心，不要改回旧的 SagerNet 规则集（域名覆盖差很多）。导入前可用对应的 reF1nd 核心检查配置：

```text
sing-box check -c singbox/config.json
```

`experimental.cache_file` 已开启（`cache.db`），用于缓存规则集、节点测速结果与上次选择；想完全重置就删掉这个文件。

## 改动后自检

改完配置至少跑一遍语法校验，再提交：

```text
mihomo -t -f mihomo/mihomo.yaml
sing-box check -c singbox/config.json
```

Loon / Egern 没有本地校验器，导入后在客户端 [配置] 页确认无红色报错，再逐项核对规则集数量（46 / 46 / 46 / 45）。

相关参考：

- [sing-box_reF1nd 核心仓库](https://github.com/reF1nd/sing-box)
- [resolve.match_only 特性说明与配置示例](https://gist.github.com/CHIZI-0618/35f59df7b17bf66ea988d775aaf76152)
- [Repcz/Tool 规则集仓库](https://github.com/Repcz/Tool)

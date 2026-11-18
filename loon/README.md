# Loon

[← 返回总览](../README.md) · 配置文件：`loon.lcf`

完整配置，可直接一键导入。本文件覆盖导入步骤，以及 DNS、规则求值、APNs、插件、场景分流这些 Loon 特有的细节。

## 使用方法

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

**镜像打不开时**：把上面两处地址的 `https://gh-proxy.com/` 前缀去掉，改用 GitHub 原始地址（`sub=` 里同样去掉前缀）：

```text
https://raw.githubusercontent.com/2253845067/Proxy-Config/main/loon/loon.lcf
```

## DNS

**防泄漏**：配置使用 DoH，不再声明系统 DNS 或明文 DNS，并用 `hijack-dns = *:53` 把应用自己发起的明文 DNS 查询交给 Loon 处理。

三条 DoH 全部写成 IP 形式（`223.5.5.5` / `1.12.12.12` / `223.6.6.6`）：DoH 主机名本身也要先解析，用域名就等于先依赖一条明文 DNS 做 bootstrap；同时不使用 `cloudflare-dns.com` 这类国内不可达的境外 DoH。

Loon 的官方文档说明，加密 DNS 查询失败时仍可能回落到普通 DNS；若要完全禁止回落，请在 Loon 的 DNS 服务器设置中关闭“加密 DNS 失败时使用普通 DNS”。

**Fake IP 例外**：Loon / Egern 显式保留 Apple 推送域名与 `*.lan` / `*.local` / `*.arpa` 的真实 IP；Mihomo 只对局域网、NTP 与 Direct 规则集保留真实 IP，Apple 中国区和中国域名仍使用 Fake IP，减少真实地址例外带来的兼容性差异。

## 其他关键项

`udp-fallback-mode = REJECT`（节点不支持 UDP 时直接拒绝，而不是回落到直连，否则 UDP 会绕开代理）、`disable-stun = true`（直连时丢弃 STUN）、`dns-reject-mode = LOOPBACKIP` + `domain-reject-mode = DNS`（拦截回环 IP，避免部分 App 立刻重试；官方取值是大写 `LOOPBACKIP` / `NOANSWER` / `NXDOMAIN`）、`allow-wifi-access = false`（不向局域网共享代理）、`ip-mode = dual` + `ipv6-vif = auto`。`skip-proxy` / `bypass-tun` 覆盖全部 RFC1918 与保留网段，所以局域网设备流量根本不进 Loon。

## 规则求值顺序

Loon 是「本地 `[Rule]` → `[Remote Rule]`」，全部未命中时才回本地执行 `FINAL`；本地规则优先于订阅规则。

所以 `[Rule]` 里的 APNs 段和 QUIC 拦截能盖住订阅里的同名判断，`FINAL,场景分流` 放在 `[Rule]` 末尾是正确位置——**不要**挪到 `[Remote Rule]` 后面，那样会让远程规则提前终止。

## QUIC 拦截

本地规则里有 `AND,((PROTOCOL,UDP),(DEST-PORT,443)),REJECT`，掐掉 QUIC 让浏览器回落 TCP（与其余三端同一思路）。

Loon 的拦截写在本地 `[Rule]` 里，优先级高于订阅规则，但局域网流量已被 `bypass-tun` 排除在隧道外，实际效果与其他三端一致。若某些应用必须用 HTTP/3，注释掉这一条即可。

## Apple 推送（APNs / iMessage / FaceTime）

本地 `[Rule]` 里把 APNs 的域名（`identity.apple.com`、`push.apple.com`、`push-apple.com.akadns.net`）和 Apple 的 5 个 `17.x` 网段 + 4 个 IPv6 `/48` 网段单独指向 `场景分流`，用于解决 iMessage、FaceTime 激活或收发异常。

本地规则优先于订阅规则，因此它会盖住订阅里 `Apple.list → CN` 的整体直连；想无条件走代理就把这些行的 `场景分流` 换成 `Proxy`，不需要时整段注释掉。

**不要照抄社区规则里的 `DOMAIN-SUFFIX,akadns.net` 和 `DOMAIN-KEYWORD,apple.com.edgekey.net`**：Loon 会按 CNAME 链匹配，而天气（`weather-data/weatherkit/weather-map.apple.com`）、App Store（`apps.apple.com`）、系统更新（`mesu.apple.com`）、iTunes、Support 全部 CNAME 到 `*.akadns.net`，加进来等于把这些流量一起拽到代理上，实测会直接导致 Apple Watch 天气、App Store 加载失败。

改完规则后要开关一次飞行模式断开所有连接才会生效；仍不行再到 Loon → 设置 → 隧道，打开「包含所有网络」和「APNs」开关。

## 插件与 MITM

插件自带的 MITM 域名会合并进配置；番茄系去广告插件的域名已在 `[Mitm]` 里显式声明，禁用该插件后记得把这行一起删掉。

插件行里的 `pin=true` 只表示“把插件固定在列表前部、不参与自动重排”（官方示例配置用它保护核心防泄漏插件），与是否跟随上游更新无关。

`ca-p12` / `ca-passphrase` 留空由 Loon 管理证书，本仓库是公开仓库，切勿填入自己的证书内容。

### 天气增强插件（iRingo:WeatherKit）

提供 Rewrite 版和脚本版两个文件，**二选一，不要同时启用**（都会 MITM `weatherkit.apple.com`，同时开等于同一批请求被改两次）。

当前启用的是 Rewrite 版，脚本版保持禁用：Rewrite 版把 `weatherkit.apple.com` 的 API 请求（含定位坐标）重写到第三方中转 `weatherkit.pages.dev`，该域名在 Cloudflare Pages 上、需走代理才通，且依赖别人的服务可用性；脚本版在本地改写响应体，配彩云 / 和风 / WAQI 的 token 才有最佳效果，脚本从 GitHub releases 拉取（该 release 走 `gh-proxy.com` 会返回 520，所以用的是 `ghfast.top`）。旧的 `AppleWeatherEnhancer.lpx` 是同类插件，保持禁用即可。

### 去广告插件

`[Plugin]` 里按类别分了九组——系统与基础、Apple/天气、微信、社交与社区、电商与生活服务、出行、影音、阅读、网盘/浏览器/搜索，共 43 个 kelee 插件（总 46 条，含 3 个非 kelee）。

它们都会把自家域名合并进 `[Mitm]`，一次全开等于解密范围大幅扩张，首次「一键更新所有外部资源」也会明显变慢；某个 App 出现证书错误或网络异常时，先关掉它对应的插件再排查。

微信那三个（小程序去广告 / 外链解锁 / 公众号去广告）会改写同一批响应，微信报网络异常就先关它们。

番茄小说两套去广告都保留在阅读组，默认启用 `fanqie.plugin`（番茄三系去广告），kelee 的 `DragonRead` 设为禁用——两套会改写同一批响应，同时开只会互相干扰；想换另一套就把这两条的 `enabled` 对调。

## WiFi 分流（可选）

`场景分流` 已把 `YUQI`（透明代理路由器）配成直连，`[Host]` 里 `ssid:YUQI = server:192.168.77.1` 指定了它在连上该 WiFi 时用路由器自身解析。`WiFi2` 保留为第二个路由器的模板，需要时手动改名并填写对应路由器 IP；不用第二个路由器时可以保留模板不动。

有多个透明代理路由器 WiFi 时：

- `[Proxy Group]` 的 `场景分流` 组里继续追加 `"SSID"=DIRECT`（如 `"WiFi3"=DIRECT`）；
- `[Host]` 里每个 WiFi 单独一行 `ssid:SSID = server:路由器IP`（`ssid:` 与 `server:` 后都要顶格写，不能留空格，否则 SSID 名会带前导空格匹配不到）；
- 普通 WiFi（没有路由器翻墙）不用列出，自动走 default 的 Proxy/Auto。

# Proxy-Config

个人代理客户端配置模板仓库。

五份配置共用同一套规则集（Repcz/Tool `X` 分支）和基本一致的策略组命名。DNS 与 QUIC 按各客户端能力分别配置，差异列在下面的[五端差异](#五端差异)表里。

本文件只放**跨端通用**的内容：目录、节点接入、端口、镜像、规则源与分流逻辑。各客户端的导入步骤、DNS / QUIC / 插件 / 分流细节见对应目录的 README。

## 目录结构

| 目录 | 说明 | 文档 |
| --- | --- | --- |
| `mihomo/` | Mihomo 配置模板（Sub-Store 远程模板，文件里不含节点） | [mihomo/README.md](mihomo/README.md) |
| `mihomo/mihomo.yaml` | 当前使用的 Mihomo 主配置 | ↑ |
| `loon/` | Loon 配置文件（完整配置，可直接一键导入） | [loon/README.md](loon/README.md) |
| `loon/loon.lcf` | 当前使用的 Loon 主配置 | ↑ |
| `surge/` | Surge 配置文件（本地入口按段引用远程核心） | [surge/README.md](surge/README.md) |
| `surge/surge.conf` | 当前使用的 Surge 主配置 | ↑ |
| `egern/` | Egern 配置文件（完整配置，可直接一键导入） | [egern/README.md](egern/README.md) |
| `egern/egern.yaml` | 当前使用的 Egern 主配置 | ↑ |
| `singbox/` | sing-box 配置模板（完整配置，仅 reF1nd 分支核心可用） | [singbox/README.md](singbox/README.md) |
| `singbox/config.json` | sing-box 主配置 | ↑ |
| `.gitignore` | 忽略 `.DS_Store` 与本地 `.workbuddy/` 工作区数据 | — |
| `.gitattributes` | 统一按文本处理，提交时做 LF 归一化，避免 Windows 换行符把配置改成 CRLF | — |

## 通用说明

### 节点从哪里来

| 客户端 | 节点接入方式 | 你需要改的地方 |
| --- | --- | --- |
| Mihomo | 节点由 Sub-Store 输出时注入，模板本身不写 `proxies` / `proxy-providers` | 在 Sub-Store 里配订阅与脚本操作（见 [mihomo/README.md](mihomo/README.md)） |
| Loon | `AllNodes` 来自 Loon 内添加的订阅 | Loon 内添加机场订阅 |
| Egern | `机场订阅` 组的 `urls` | 把占位串换成真实订阅 URL（只需改这一处，`Proxy` 用 YAML 锚点复用） |
| Surge | `local.conf` 的 `policy-path` | 本地入口保存真实订阅 URL（只需改这一处，`Proxy` / `Auto` 用 `include-other-group` 复用） |
| sing-box | `providers[0].url` | 把占位串换成真实订阅 URL |

Egern、Surge 与 sing-box 仓库文件里写的是占位串（`这里填入你的机场订阅 URL` / `https://example.com/your-subscription`），**不替换就会一直加载失败**，这不是配置残缺。

### 端口与监听

| 客户端 | 监听 | 说明 |
| --- | --- | --- |
| Mihomo | `7890`（mixed）、`0.0.0.0:1053`（DNS） | `allow-lan: true` + `bind-address: "*"`，局域网内其他设备可用这两个端口，且没有认证 |
| sing-box | `0.0.0.0:7890`（mixed）、`127.0.0.1:9090`（面板）、TUN `172.19.0.1/30` | mixed 代理开放给局域网且无认证；与 Mihomo 抢同一个 7890，**同一台机器不要同时启用** |
| Egern | `3080`（http）、`3090`（socks） | 只监听本机，`allow_external_connections: false` |
| Loon | 无固定端口 | 走 Network Extension 隧道 |
| Surge | 无固定端口 | 走 Network Extension / Enhanced Mode 隧道；`allow-wifi-access = false`，不向局域网共享代理 |

### 下载镜像

五端引用的规则集、图标、面板都经第三方 GitHub 镜像：

- mihomo / Loon / Egern / Surge：`https://gh-proxy.com/https://raw.githubusercontent.com/...`
- sing-box：`https://ghfast.top/https://github.com/Repcz/Tool/raw/X/sing-box/Rules/*.srs`

镜像失效时整体替换即可（两种镜像的前缀形式相同，可以直接互换）：

```bash
grep -rl 'gh-proxy.com' mihomo loon egern surge | xargs sed -i 's#gh-proxy.com#ghfast.top#g'
```

Sub-Store 拉取模板这一步例外：优先用 GitHub 原始地址，`gh-proxy.com` 对某些路径会返回 403（备用写法见 [mihomo/README.md](mihomo/README.md)）。

### 公开仓库提醒

配置里不含任何节点、订阅链接或证书内容。`ca-p12` / `ca-passphrase` / `secret` 一律留空，由客户端自己管理，**不要把证书或订阅写进这个仓库**。

## 规则源与分流逻辑（五端通用）

> **规则来源**：规则集全部取自 [Repcz/Tool](https://github.com/Repcz/Tool)（分支 `X`）对应的 `mihomo/Rules`、`Loon/Rules`、`Egern/Rules`、`Surge/Rules`、`sing-box/Rules` 目录，经镜像中转引用，上游每天自动重建，本仓库只引用不镜像内容。mihomo / Loon / Egern / Surge 各 46 个，sing-box 45 个（上游没有 `ChinaASN.srs`，官方 sing-box 也不支持 ASN 匹配）。原先的 ACL4SSR 大列表（`BanEasyList` 4.1 万条 + `BanEasyPrivacy` 4 万条 + 其余三项，合计约 8.7 万条）已整体下线，换成 `Reject`（6200 条精选）+ `Ads_AWAvenue`（国内 App 净化），内存占用与启动解析量大幅下降。
>
> 规则集**同源**，但本地补充规则**不完全相同**，看分流结果时不要只看规则集：Loon 额外强制两条业务域名直连（`report-online.sh.wxgateway.com`、`diancan.xibeilangsaoma.com`），天气域名方面，`weather.apple.com → CN` 已在 `[Rule]` 里写死，与其他四端一致；只有 `weather-analytics-events.apple.com` **有意不写** CN——它交给 WeatherKit 插件做 REJECT-DROP，本地补 `DOMAIN-SUFFIX,CN` 会盖掉插件处理，不要补。Surge 的天气域名补丁与 mihomo / Egern / sing-box 对齐。
>
> **国内判定**：mihomo / Loon / Egern / Surge 都是四层——`ChinaDomain`（1.3 万条域名）→ `ChinaASN`（5076 条运营商 ASN，补 `ChinaIP` 漏掉的国内 IP）→ `ChinaIP`（CIDR 列表）→ `GEOIP,CN` 兜底，但 **Surge 的排列位置不同**：Surge 求值到第一条 IP 规则时会暂停并做一次真实 DNS 解析，所以 Surge 里 IP 三层被整体挪到「所有域名规则之后、`FINAL` 之前」，并且**不带** `no-resolve`——只有「域名表全没命中」的流量才解析一次做 IP 兜底；命中域名表时零解析。mihomo / Loon / Egern 仍是 `ChinaASN` / `ChinaIP` 紧跟在 `ChinaDomain` 后面并带 `no-resolve`（它们的规则引擎不因 IP 规则中断求值）。Surge 的 `IP-ASN` 用的是客户端自带的 GeoLite2 ASN 库（随 App 更新，配置里无法更换）。sing-box 只有三层（`ChinaDomain` → `ChinaIP` → 末条 `final: Proxy`），没有 ASN 和 GEOIP 兜底。
>
> **规则格式**：mihomo 用文本 `classical` 列表（`behavior: classical` + `format: text`），不依赖 mrs 格式，旧内核也能跑；Loon / Surge 是文本 list；Egern 是 yaml 规则集；sing-box 是编译后的 `.srs` 二进制。

### 五端差异

| 项目 | Mihomo | Loon | Egern | Surge | sing-box |
| --- | --- | --- | --- | --- | --- |
| 策略组 | `Proxy` / `Auto` / `CN` | 同 + `场景分流` | `场景` / `Proxy` / `Auto` / `CN`（+ 隐藏的 `机场订阅`） | 同 + `场景分流`（+ 隐藏的 `机场订阅`） | `Proxy` / `Auto` / `CN` |
| 广告拦截 | `REJECT-DROP` | `REJECT` | `REJECT-NO-DROP` | `REJECT` + `pre-matching`（DNS / SYN 阶段即拒，命中风暴时自动升级 `REJECT-DROP`） | `action: reject` |
| QUIC 拦截 | 规则 `AND,((NETWORK,udp),(DST-PORT,443))` | 规则 `AND,((PROTOCOL,UDP),(DEST-PORT,443))` | `block_quic: true` 开关（不是规则） | `block-quic = all` 开关（不是规则；拦截所有 UDP/443） | 规则 `{"network":"udp","port":443}` |
| 国内判定 | 四层 | 四层 | 四层 | 四层（ASN 库随 App 更新，配置无法更换） | 三层（无 ASN） |
| `CN` 组可选项 | `DIRECT` / `Proxy` / `REJECT` | 同 | 同 | 同 | `direct` / `Proxy` |
| 规则格式 | classical text | text | yaml | text | srs |

各端的实现细节（DNS 方案、求值顺序、插件、场景分流等）见对应目录的 README。

## 各端文档索引

| 客户端 | 文档 | 一句话定位 |
| --- | --- | --- |
| Mihomo | [mihomo/README.md](mihomo/README.md) | Sub-Store 模板，`fake-ip` + 分流 DNS，最灵活 |
| Loon | [loon/README.md](loon/README.md) | 完整配置 + 46 个插件，APNs / 天气 / 去广告重点 |
| Surge | [surge/README.md](surge/README.md) | 完整配置，`pre-matching` 拦截，iOS / Mac 共用 |
| Egern | [egern/README.md](egern/README.md) | 完整配置 + 16 个模块，场景（SSID）分流 |
| sing-box | [singbox/README.md](singbox/README.md) | reF1nd 核心专属，Provider + 并发 DNS |

## 改动后自检

改完配置至少跑一遍语法校验，再提交：

```text
mihomo -t -f mihomo/mihomo.yaml
sing-box check -c singbox/config.json
```

Loon / Egern / Surge 没有本地校验器，导入后在客户端 [配置] 页确认无红色报错，再逐项核对规则集数量（46 / 46 / 46 / 46 / 45）。

相关参考：

- [sing-box_reF1nd 核心仓库](https://github.com/reF1nd/sing-box)
- [resolve.match_only 特性说明与配置示例](https://gist.github.com/CHIZI-0618/35f59df7b17bf66ea988d775aaf76152)
- [Repcz/Tool 规则集仓库](https://github.com/Repcz/Tool)

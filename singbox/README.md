# sing-box

[← 返回总览](../README.md) · 配置文件：`config.json`

完整配置，仅 reF1nd 分支核心可用。本文件覆盖核心要求、订阅接入、DNS、TUN、面板与规则集。

## 获取配置

sing-box 没有 URL 一键导入，把 `singbox/config.json` 下载到本地再导入即可。直接取仓库文件不方便时，可用镜像地址下载：

```text
https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/singbox/config.json
```

镜像打不开就去掉 `https://gh-proxy.com/` 前缀，改用 GitHub 原始地址：

```text
https://raw.githubusercontent.com/2253845067/Proxy-Config/main/singbox/config.json
```

## 概览

`singbox/config.json` 对应 Mihomo 的 `Proxy` / `Auto` / `CN` 三个策略组（`Proxy` 默认 `Auto`、`CN` 默认 `direct`），并保留广告拦截、国内直连、海外代理、TUN 和 DNS 分流逻辑。

## 核心要求

本配置使用 sing-box_reF1nd 分支的 Provider 功能，**必须用 reF1nd 核心运行**；文件里的 `$schema` 也指向 `reF1nd/sing-box` 的 `reF1nd-testing-next` 分支文档。

官方 sing-box 核心会因 7 处 fork 专属字段直接启动失败：`providers`、`dns.servers[].type: group`（两处）、outbound 的 `providers` / `use_all_providers`、`route.rules[].match_only`、`experimental.urltest_unified_delay`。

导入前可用对应的 reF1nd 核心检查配置：

```text
sing-box check -c singbox/config.json
```

## 订阅接入

机场订阅链接填写在 `singbox/config.json` 的 `providers[0].url`；`exclude` 正则与其余三端同一套，用于剔除机场信息节点。订阅节点会自动加入 `Proxy` / `Auto` 策略组。

订阅首次加载使用 `provider-download` 直连以避免与 `Proxy` 策略组形成循环依赖；规则集下载使用 `download` 并通过 `Proxy`。请确保订阅地址可直连访问，并在启动前将示例地址替换为真实订阅链接。

## DNS

配置使用 reF1nd 的并发 DNS 组、Provider HTTP 客户端和 `resolve.match_only`：先匹配域名规则，解析后再匹配 IP 规则，同时保留原始域名用于实际连接。

`default_domain_resolver: cn` 决定未被 DNS 规则命中的域名走国内 DoH 解析；DNS rule-set 与 CN / 直连分流一致，只有海外分类规则命中的域名才交给 `remote`（8.8.8.8 / 1.1.1.1，经 `Proxy`）。

## TUN 与入站

TUN 入站已启用自动路由和严格路由（`auto_route` / `strict_route`），地址为 `172.19.0.1/30`、接口名 `sing-box`；启动客户端时请授予 TUN/管理员权限，否则系统流量不会被接管。

`mixed` 入站监听 `0.0.0.0:7890` 且无认证，局域网设备可连接；`set_system_proxy: false` 表示不自动改系统代理，需要时手动开或只用 TUN。

配置没有 Wi-Fi 分流，端口和 Mihomo 的 `mixed-port` 相同，两个客户端不能同时运行。

## QUIC 拦截

与其他四端一致：规则里有一条 `{"network":"udp","port":443,"action":"reject"}`（位于私网直连规则之后），让浏览器回落到 TCP。不需要可整条删掉。Egern 走的是 `block_quic` 开关，不是规则。

## 控制面板

已启用：`clash_api` 监听 `127.0.0.1:9090`，`external_ui` 指定为 `ui`，面板本体是 metacubexd（经 `ghfast.top` 下载）。启动后打开 `http://127.0.0.1:9090/ui` 即可。

注意 sing-box 的 `external_ui_download_url` 只在 `external_ui` 指定的目录**存在且为空**时才会触发下载，两者必须成对出现，只写下载地址不会有任何效果。

## 策略组与规则集

`CN` 策略组只有 `direct` / `Proxy` 两个选项：官方已弃用 `block` 出站，REJECT 需通过规则 `action: reject` 实现，因此不再往组里塞 REJECT（这点与 Mihomo / Egern 的 `CN` 组略有不同）。

规则集使用 sing-box 原生 `.srs` 文件，同样来自 Repcz/Tool 的 `sing-box/Rules/`，首次启动时经 `ghfast.top` 下载。若运行时提示规则集解析失败，说明本地 reF1nd 核心版本落后于规则集格式，优先升级核心，不要改回旧的 SagerNet 规则集（域名覆盖差很多）。

## 缓存

`experimental.cache_file` 已开启（`cache.db`），用于缓存规则集、节点测速结果与上次选择；想完全重置就删掉这个文件。

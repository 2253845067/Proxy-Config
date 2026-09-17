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
https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/mihomo/mihomo.yaml
```

3. 保存后，将生成的链接导入 Mihomo Party 等客户端。

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

> **WiFi 分流（可选）**：若你使用透明代理路由器，把配置中的 `WiFi1` / `WiFi2` 替换为路由器 WiFi 名称、`10.0.0.1` / `10.0.0.2` 替换为对应路由器管理 IP。连上这些 WiFi 时 Loon 自动走直连（由路由器代理翻墙），其他网络由 Loon 自动分流，避免双重代理。
>
> 有多个透明代理路由器 WiFi 时：
> - `[Proxy Group]` 的 `场景分流` 组里继续追加 `"SSID"=DIRECT`（如 `"WiFi3"=DIRECT`）；
> - `[Host]` 里每个 WiFi 单独一行 `ssid: SSID=server: 路由器IP`；
> - 普通 WiFi（没有路由器翻墙）不用列出，自动走 default 的 Proxy/Auto。

## Egern 使用方法

1. 点击一键导入 [配置](https://egernapp.com/profiles/new?name=egern&url=https%3A%2F%2Fgh-proxy.com%2Fhttps%3A%2F%2Fraw.githubusercontent.com%2F2253845067%2FProxy-Config%2Fmain%2Fegern%2Fegern.yaml)。
2. 如果无法跳转导入，打开 Egern 底部 [设置] -> 最上面 [配置] -> 右上角 [+号] -> [下载]，URL 填写：

```text
https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/egern/egern.yaml
```

3. 导入完成后，切换并使用 `egern` 配置。
4. 打开 [工具] -> [代理] -> `机场订阅`，填入你的机场订阅并保存。
5. 回到策略组页面更新外部资源，确认 `Proxy` / `Auto` 里能看到节点。
6. 回到仪表界面打开 Egern 开关即可。

## sing-box 使用方法

`singbox/config.json` 对应 Mihomo 的 `Proxy` / `Auto` / `CN` 三个策略组，并保留广告拦截、国内直连、海外代理、TUN 和 DNS 分流逻辑。

本配置使用 sing-box_reF1nd 分支的 Provider 功能，请使用对应的 reF1nd 核心运行；官方 sing-box 核心不支持配置中的 `providers` 字段。

机场订阅链接填写在 `singbox/config.json` 的 `providers[0].url`。订阅节点会自动加入 `Proxy` / `Auto` 策略组。

订阅首次加载使用 `provider-download` 直连以避免与 `Proxy` 策略组形成循环依赖；规则集下载使用 `download` 并通过 `Proxy`。请确保订阅地址可直连访问，并在启动前将示例地址替换为真实订阅链接。

配置使用 reF1nd 的并发 DNS 组、Provider HTTP 客户端和 `resolve.match_only`：先匹配域名规则，解析后再匹配 IP 规则，同时保留原始域名用于实际连接。

TUN 入站已启用自动路由和严格路由（`auto_route` / `strict_route`），启动客户端时请授予 TUN/管理员权限；否则系统流量不会被接管。

规则集使用 sing-box 原生 `.srs` 文件，并在首次启动时经 `ghfast.top` 从 SagerNet 仓库下载。导入前可用对应的 reF1nd 核心检查配置：

```text
sing-box check -c singbox/config.json
```

相关参考：

- [sing-box_reF1nd 核心仓库](https://github.com/reF1nd/sing-box)
- [resolve.match_only 特性说明与配置示例](https://gist.github.com/CHIZI-0618/35f59df7b17bf66ea988d775aaf76152)

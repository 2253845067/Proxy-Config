# Mihomo

[← 返回总览](../README.md) · 配置文件：`mihomo.yaml`

Mihomo 配置模板，供 Sub-Store 远程引用，文件里不含节点。节点、策略组注入、DNS 与 QUIC 的细节都在下面。

## 使用方法

1. 点击 Sub-Store 前端第二个 [文件] tab，新建 Mihomo 配置文件。
2. 进行配置：设置订阅、添加脚本操作，远程链接填写：

```text
https://raw.githubusercontent.com/2253845067/Proxy-Config/main/mihomo/mihomo.yaml
```

3. 保存后，将生成的链接导入 Mihomo Party 等客户端。
   若预览提示“无法下载脚本”，先检查模板地址是否可访问；`gh-proxy.com` 中转地址可能返回 403，可改用上面的 GitHub 原始地址（Sub-Store 后端需能访问 GitHub）。
4. 在客户端开启 TUN 模式并启用系统代理接管；模板已开启自动路由和严格路由，只有 TUN 真正运行后才能阻止应用绕过内置 DNS。TUN 栈用的是 `mips`（mihomo 1.19.x 起的官方默认栈，用户态、开销低、不需要放行防火墙）；内核较旧不认这个值时改成 `mixed`。

**模板地址**：优先用上面的 GitHub 原始地址。Sub-Store 后端直连 GitHub 困难时，可以临时换成镜像地址：

```text
https://gh-proxy.com/https://raw.githubusercontent.com/2253845067/Proxy-Config/main/mihomo/mihomo.yaml
```

> **注意**：Sub-Store 拉取模板这一步镜像不保证可用——`gh-proxy.com` 对某些路径会返回 403，遇到就换回 GitHub 原始地址。这一步和配置里的规则集下载不同：`mihomo.yaml` 内 50 个规则集外链本来就走 `gh-proxy.com`，只有「Sub-Store 拉模板」这一跳用原始地址最稳。

## 策略组

`Proxy` 手动选择（内含 `Auto` 与全部订阅节点）、`Auto` 自动测速（300 秒一轮、容差 50ms）、`CN` 默认 `DIRECT`（可切 `Proxy` / `REJECT`）。

节点由 Sub-Store 注入，`filter` 正则会把「群 / 剩余流量 / 到期 / 官网」这类机场信息节点剔除；广告规则统一走 `REJECT-DROP`（丢弃不回 RST，减少被探测）。

## DNS / hosts

`fake-ip` 过滤为 `rule` 模式，只对局域网、NTP 和 `Direct` 规则集明确标记 `real-ip` 的域名下发真实 IP；Apple 中国区和中国域名仍使用 Fake IP。

海外域名走 `#Proxy` 的 Cloudflare/Google DoH，`Direct`、微信、Bilibili、AppleCN、OneDrive、国内下载 CDN 和 ChinaDomain 走国内 DoH（223.5.5.5 / doh.pub），使 DNS 出口与 CN/直连规则一致。内网域名（`rule-set:Lan`）交给 `system` 解析，路由器/NAS 域名才能通。

配置里同时关了 `use-system-hosts` 和 `use-hosts`：前者忽略本机 hosts，后者忽略配置内的 `hosts:` 段——所以**现在往配置里加 `hosts:` 段不会生效**。要恢复静态映射，需要把 `use-hosts` 改回 `true` 并新增 `hosts:` 段；要走本机 hosts 则需同时把 `use-system-hosts` 改回 `true`；也可以不改开关，直接用 `nameserver-policy` 或 `DOMAIN` 规则解决。

## DNS 端口

内置 DNS 监听 `0.0.0.0:1053`（内核默认端口）。53 端口由 TUN 的 `dns-hijack` 接管，因此不占 53，不会和 Windows ICS / Docker 抢端口。

但要注意 `allow-lan: true` 下 `0.0.0.0:1053` 对局域网是开放的（同样开放的还有 `7890`，且都不带认证）；不需要给局域网用时，把 `listen` 改成 `127.0.0.1:1053`。

## QUIC 拦截

规则里有 `AND,((NETWORK,udp),(DST-PORT,443)),REJECT`，位置在局域网 / 直连规则之后，让浏览器回落到 TCP。既然 QUIC 在规则层已被整体拒绝，嗅探段就不再声明 `QUIC` 类型，避免留下永远不执行的分支。

若某些应用必须用 HTTP/3，注释掉这一条即可（同时可按需加回 `sniffer.sniff.QUIC`）。

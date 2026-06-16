# Proxy-Config

个人代理客户端配置模板仓库。

## 目录结构

- `mihomo/`: Mihomo 配置模板。
- `mihomo/mihomo.yaml`: 当前使用的 Mihomo 主配置。
- `loon/`: Loon 配置文件。
- `loon/loon.lcf`: 当前使用的 Loon 主配置。
- `egern/`: Egern 配置文件。
- `egern/egern.yaml`: 当前使用的 Egern 主配置。

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

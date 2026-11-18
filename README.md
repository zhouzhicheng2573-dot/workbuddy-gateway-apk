# WorkBuddy 网关（Android APK）

把 WorkBuddy / CodeBuddy 的网页登录态变成**本机 OpenAI 兼容接口**的 Android 应用。
装到手机上，起一个只监听 `127.0.0.1:8788` 的反代，任何支持自定义 Base URL 的客户端
（Cherry Studio、ChatBox、CCSM、Codex、OpenWebUI……）都能直接接上。

> **本仓库不包含任何账号、Token 或密钥。** 安装包里只有一个占位用的
> `config.example.json`，首次启动时复制成你自己的 `config.json`，
> 账号需要你自己登录。

---

## 下载

| | |
|---|---|
| 最新版本 | [Releases](../../releases/latest) |
| 文件名 | `wbgateway-signed.apk` |
| 大小 | 34.7 MB（含 Node 运行时，解压后约 95 MB） |
| 版本 | 1.5（versionCode 6） |
| 支持 | Android 7.0+（minSdk 24 / targetSdk 33） |
| 架构 | arm64-v8a |

安装前请允许「安装未知来源应用」。装好后打开 App，点「启动」，
网关就开始在 `http://127.0.0.1:8788` 上跑了。

校验下载完整性：

```sh
sha256sum wbgateway-signed.apk
# 与 Release 页面里的 .sha256 文件比对
```

---

## 它能做什么

- **OpenAI 兼容**：`/v1/chat/completions`、`/v1/models`
- **Anthropic 兼容**：`/v1/messages`、`/v1/messages/count_tokens`
- **Responses API**：`/v1/responses`
- **多站点**：国内版（copilot.tencent.com）、国际版 CLI（codebuddy.ai）、国际版 WorkBuddy（workbuddy.ai）
- **号池轮转**：同一站点可以放多个账号，失败自动换号
- **站点降级**：某个站点没额度/连不上，自动换到另一个确实有该模型的站点
- **用量统计**：按天 / 按模型统计调用数、token、credit，**含缓存命中率**
- **上下文压缩**：超长对话自动收缩，避免上游 400
- **自定义壁纸 + 液态玻璃界面**：AGSL 折射 shader（Android 12+）

服务只监听本机回环地址，局域网和公网都访问不到。

---

## 首次配置

1. 打开 App → 点「登录」→ 选站点 → 跳系统浏览器授权 → 自动回到 App 入池
2. 点「复制配置」，会得到本机随机生成的一串 `sk-local-xxxx` 密钥和 Base URL
3. 把 Base URL 和密钥填进你的客户端

```
Base URL: http://127.0.0.1:8788/v1
API Key : sk-local-xxxxxxxx（App 里「复制配置」直接给你）
```

> 注意：同一台手机上，客户端和网关都在本机，所以 `127.0.0.1` 是通的。
> 如果你想让**同一 Wi-Fi 下的电脑**也用它，本 APK 没有开放对外监听
> （安全考虑，避免账号被局域网内其他人白嫖）。

---

## 配置项

首次启动会把 `config.example.json` 复制成私有目录下的 `config.json`，
可以在 App 内的控制台里改，也可以直接用文本编辑器改。

几个常用的：

| 字段 | 说明 |
|---|---|
| `apiKey` | 本机密钥，随便填一串，客户端要用同样的值 |
| `defaultSite` / `defaultModel` | 默认走哪个站点、哪个模型 |
| `sites.<名字>.enabled` | 启用/禁用某个站点 |
| `allowModels` / `excludeModels` | 模型白/黑名单，支持 `*` 通配 |
| `modelRoutes` | 把某模型钉死在某个站点 |

完整说明见 App 内控制台的「服务」分页，或 `docs/` 目录。

---

## 首次启动会做什么

App 第一次运行时，会把内置的网关服务端代码（`assets/gateway/`）解压到私有目录：

```
/data/data/com.shizi.wbgateway/files/gateway/
├── src/*.mjs          # 19 个模块
├── server.mjs
├── console/index.html # 控制台
├── config.json        # ← 你自己的配置（首启从 example 复制）
├── auth.*.pool.json   # ← 你登录后的账号池
└── server.log
```

升级 APK 时**只覆盖随包代码**，账号池、`config.json`、日志原样保留，
不需要重新登录。

---

## 常见问题

**网关显示运行中，但电脑连不上？**
正常。它只监听 `127.0.0.1`，设计上就只给本机用。

**后台一会儿就断了？**
ColorOS / MIUI 这类系统会掐后台网络。到「设置 → 电池 → 应用 → WorkBuddy 网关」
允许后台运行、关闭省电限制。App 的「⋯」菜单里有「允许后台运行（保活）」引导。

**提示 `usage exceeded` / 429？**
是上游账号的频率限制，不是你本地的问题。换号、换模型或等额度重置。

**模型列表是空的？**
先确认站点已登录（控制台「账号登录」分页），登录后目录才会拉取。

**APK 装不上 / 签名冲突？**
如果之前装过别的版本，需要先卸载。不同签名不能覆盖安装。

---

## 从源码构建

本仓库的 APK 是在 Android 设备上用 Termux 构建的。需要的工具：

```sh
pkg install aapt apksigner openjdk-17 python
```

然后：

```sh
cd apkbuild
sh build.sh
# 产物：build/wbgateway-signed.apk
```

`build.sh` 会依次做：`aapt2 compile/link` → `javac` → `d8` → 打包 assets
→ `apksigner sign`。签名用的 `keystore.jks` 密码是 `wbgateway`
（自签名，只用于本地安装，不上架任何商店）。

---

## 目录结构（仓库）

```
apkbuild/
├── AndroidManifest.xml
├── build.sh              # 构建脚本
├── pack.py               # 打包 assets + libnode.so
├── keystore.jks          # 自签名密钥库
├── src/com/shizi/wbgateway/
│   ├── MainActivity.java
│   ├── GatewayService.java
│   ├── Gw.java           # 解包 / 启停 node
│   ├── GlassPanel.java   # AGSL 玻璃面板
│   ├── GlassDlg.java     # 玻璃弹窗
│   └── AuroraBg.java     # 极光背景
├── assets/
│   ├── gateway/          # 内置网关服务端（Node，零依赖）
│   ├── ui/               # 主界面 WebView
│   ├── agsl/             # 液态玻璃 shader
│   └── libs/             # native 依赖
└── res/
```

---

## 免责声明

本项目是一个**本地反向代理工具**，用于把你自己已登录的网页端会话转换成本机 API，
方便在本地客户端里使用。它不提供、不分发任何账号或额度。

使用者需自行遵守上游服务的使用条款。因使用本工具导致的账号风控、限流、
封禁等后果，由使用者自行承担。

请勿将本项目用于转售、共享账号、批量注册等用途。

---

## License

MIT

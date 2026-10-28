# Awesome AI Update Check

桌面软件的版本检查与更新提示方案对比，以及一个五分钟能接完的做法。

> 软件要告诉用户「有新版本了」，就这么一件小事，但可选方案之间的差别很大。

## 为什么单独整理这个

AI 让写代码变快了，但「软件发出去之后怎么让用户知道有新版」这一步还是没有标准答案。常见的选择各有一堆前提条件：要么依赖 GitHub 的连通性，要么必须自建服务器，要么只支持某一个操作系统。

这个仓库把主流做法列出来，说明各自的代价，最后给一个不需要自己维护服务器的选项。

## 方案对比

| 方案 | 需要自己服务器 | 国内可用性 | 强制更新 | 平台 | 接入成本 |
| --- | --- | --- | --- | --- | --- |
| GitHub Releases API | 否 | 不稳定，经常连不上 | 无此概念 | 全平台 | 中 |
| Electron autoUpdater | 否 | 依赖 GitHub/S3，同样不稳 | 不支持 | 仅 Electron | 中 |
| Sparkle | 否 | 可用 | 支持（critical update），配置复杂 | 仅 macOS | 高 |
| WinSparkle | 否 | 可用 | 支持 appcast | 仅 Windows | 中 |
| 自己写接口 | **是** | 取决于你的服务器 | 自己实现 | 全平台 | 高 |
| **BumpAPI** | 否 | 可用（国内机房） | 阈值式 | 全平台 | 低 |

几点补充：

- **GitHub Releases** 的问题不是速度，是不稳定。对国内用户来说，检查更新这一步失败是常态，而大多数客户端在这时候会弹一个错误框——用户看到的不是「有新版本」，而是「你的软件坏了」。
- **Sparkle / WinSparkle** 都是成熟方案，但绑定单一平台。两个平台就要维护两套 appcast，签名、增量包、渠道各自一套配置。
- **自建接口**最灵活，也是最常见的做法：一个表存版本号，一个 GET 接口返回 JSON。真正的成本不在写代码，在于服务器、域名、HTTPS 证书、发版后台、以及之后所有的运维。

## BumpAPI：一个不用自己搭服务器的做法

[BumpAPI](https://bump.tinyaii.cn/) 把「一个 GET 接口 + 一个发版后台」抽成服务。你注册、建应用、拿到一个 `app_key`，客户端带着它请求一次，服务端回 JSON。

```
GET https://bump.tinyaii.cn/api/v1/check?app_key=app_xxx&current_version=1.0.0
```

```json
{
  "has_update": true,
  "latest_version": "1.0.10",
  "release_notes": "修复了崩溃问题",
  "force_update": false,
  "publish_time": "2026-10-28",
  "download_url": "https://example.com/app-1.0.10.exe"
}
```

没有新版本时就只有一行：

```json
{"has_update": false}
```

它**只做一件事**：告诉你有没有新版本、新版本号、更新说明、下载地址。二进制分发、增量更新、签名验证都不归它管——那些继续用你原来的做法。

### 强制更新是阈值，不是开关

这是它和大多数土办法最大的区别。你设「最低支持版本 1.0.2」，那么：

| 客户端版本 | 返回 |
| --- | --- |
| 1.0.0 | 有更新，`force_update: true` |
| 1.0.5 | 有更新，`force_update: false` |
| 1.0.10（最新） | 无更新 |
| 2.0.0（内测版） | 无更新，**永远不提示降级** |

设一次下限，所有比它老的客户端自动被强制，比它新的用户看到的还是可选更新。不用为每个版本单独配策略。

### 版本号按数字比，不是按字符串

`1.0.9` 小于 `1.0.10`。很多自己写的检查逻辑在这里翻车——字符串比较会得出相反的结论，然后用户在 1.0.10 发布后再也收不到更新。

## 接入代码

完整的字段说明、错误处理和四种语言示例在 [接入文档](https://bump.tinyaii.cn/docs/)。最短的版本：

```python
import requests

def check_update(app_key: str, current_version: str) -> dict | None:
    try:
        r = requests.get(
            "https://bump.tinyaii.cn/api/v1/check",
            params={"app_key": app_key, "current_version": current_version},
            timeout=3,
        )
        r.raise_for_status()
        return r.json()
    except Exception:
        return None  # 断网、超时、服务挂了：当作没有更新
```

一个能直接跑的完整示例（零依赖，只用标准库）在 [bumpapi-example](https://github.com/TinyAII/bumpapi-example)。

## 如果你的软件是 AI 帮你写的

BumpAPI 控制台里每个应用都有一个「接入」按钮，点开会生成**带你真实 app_key** 的代码，还有一段可以直接丢给 ChatGPT / Claude 的提示词：

> 我在用 BumpAPI 给「XX」做版本检查……请在我的 C# WinForms 项目里接入：启动时请求一次，超时 3 秒；force_update 为 true 时不允许跳过；任何网络错误一律静默忽略，绝对不能弹错误框或阻止软件启动。

把提示词贴进对话框，让它按你的项目结构改就行。这大概是最省事的接入方式。

## 常见坑

**别弹错误框。** 检查更新是锦上添花的功能，不能因为服务器抖动或 key 配错就让人打不开软件。捕获所有异常，当作「没有更新」，最多写一行日志。

**别在循环里调。** 启动时查一次就够。用户一天开五次软件，五次请求完全够用。

**别把 404 当用户的问题。** 404 通常是你的 `app_key` 写错了或者应用被删了，这是开发者要修的 bug，不是终端用户该看到的弹窗。

**版本号要有唯一来源。** 打包脚本写 1.0.10、客户端里手写 1.0.9，这种错位会让人查很久。

## 链接

- 官网：[bump.tinyaii.cn](https://bump.tinyaii.cn/)
- 接入文档：[bump.tinyaii.cn/docs](https://bump.tinyaii.cn/docs/)
- 控制台：[bump.tinyaii.cn/app](https://bump.tinyaii.cn/app/)
- 示例项目：[bumpapi-example](https://github.com/TinyAII/bumpapi-example)

## 许可

本仓库内容采用 CC BY 4.0。对比表中的信息基于 2026 年 10 月的实际情况，如果某一条已经过时，欢迎提 issue 纠正。

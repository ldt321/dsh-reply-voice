# 回复语音（Reply voice）

把 DSH 的助手回复读出来：用 [edge-tts](https://github.com/rany2/edge-tts) 在**内存里**合成 MP3，直接交给打开 DSH 页面的浏览器播放。**全程不生成任何音频文件**——不写临时目录，也不留残留。

## 功能

每条助手消息的操作行里都有三个动作：

- **朗读**（喇叭图标）：合成这一条并开始播放。
- **暂停 / 继续**（两条竖线 / 三角图标）：就地暂停，再点一下从原位置接着读。
- **停止**（方块图标，播放中或暂停时出现）：结束播放并立即释放后台缓冲，下次从头开始。

自动朗读默认开启：每轮回复结束时自动读这一轮的最终文字。`autoPlay: false` 时只保留按钮，不自动出声。

其他行为：

- 声音在**打开 DSH 页面的那个浏览器**里播放，浏览器不需要额外权限；Host 只负责合成和把字节递过去。
- 同一时间只播一条：手动点另一条会打断当前朗读。
- 工具调用的中间步骤不朗读，只读这一轮最终给你的文字；代码块、链接、表格和 Markdown 记号会被去掉，超长内容按配置截断。
- 子代理（subagent）的回复默认不读。
- 插件激活时会清理旧版本留在 `%TEMP%\dsh-reply-voice` 的临时 MP3。

## 安装

从 npm 安装（发布后）：

```powershell
dsh plugin --profile desktop add dsh-reply-voice
dsh --profile desktop
```

从本地 tarball 安装：

```powershell
dsh plugin --profile desktop add .\dsh-reply-voice-1.1.0.tgz
dsh --profile desktop
```

开发时也可以直接链接源码目录：

```powershell
dsh plugin --profile desktop add C:\path\to\reply-voice
```

安装后，插件会出现在 profile 的 `dsh.profile.bundles` 里，并通过 `cordis.patch.yml` 插入 `reply-voice` 行。

## 依赖

需要一个装有 `edge_tts` 的 Python 3：

```powershell
python -m pip install edge-tts
python -c "import edge_tts; print(edge_tts.__version__)"
```

插件按 `config.pythonPath` → `python` → `py` → 自动发现的 `%LOCALAPPDATA%\Programs\Python\Python*\python.exe` 顺序查找解释器，找到能用的就固定下来。
## 配置

在 profile 的 `cordis.patch.yml` 里覆盖 `reply-voice` 行的 `config`：

```yaml
- id: reply-voice
  name: dsh-reply-voice
  config:
    enabled: true
    voice: zh-TW-HsiaoChenNeural
    rate: 0
    volume: 0
    maxChars: 800
    includeSubagents: false
    autoPlay: true
    pythonPath: ''
```

| 键 | 默认值 | 说明 |
| --- | --- | --- |
| `enabled` | `true` | 关掉后不再朗读，按钮也不会注册（等价于禁用插件） |
| `voice` | `zh-TW-HsiaoChenNeural` | edge-tts 音色，例如 `zh-CN-XiaoxiaoNeural` |
| `rate` | `0` | 语速百分比，范围 -100 ~ 100 |
| `volume` | `0` | 音量百分比，范围 -100 ~ 100 |
| `maxChars` | `800` | 每次最多朗读的字符数，超出部分截断 |
| `includeSubagents` | `false` | 是否也朗读子代理的回复 |
| `autoPlay` | `true` | 每轮回复结束是否自动朗读；`false` 时只有点按钮才读 |
| `pythonPath` | `空` | Python 解释器的绝对路径；留空则自动查找 |
> 自动朗读依赖浏览器的自动播放许可。如果页面没被点过，浏览器可能拒绝自动出声；此时按钮的悬停提示会写明原因，点一下「朗读」即可正常播放。

## 接口（插件内部）

| 路由 | 说明 |
| --- | --- |
| `POST /reply-voice/speak` | 请求体 `{ sessionId, messageId }`（或直接 `{ text }`）；替换当前朗读，合成完成后返回 `{ ok, playbackId }` |
| `GET /reply-voice/audio?id=` | 返回内存里的 MP3 字节，支持 `Range` |
| `POST /reply-voice/stop` | 请求体 `{ id }` 释放这段缓冲；不带 `id` 时取消正在合成的那一段 |
| `GET /reply-voice/autoplay?since=` | 取「这一轮读完了」的提示 `{ sessionId, messageId, key, at }`，`at <= since` 时返回 `null` |
| `GET /reply-voice/state` | 诊断用：当前缓冲的音频与最新提示 |
## 排错

- 完全没声音：先确认 Python 能导入 `edge_tts`，再确认 `enabled` 没有被关掉。
- 按钮悬停显示「朗读失败：...」时：
  - 「找不到可用的 Python（需要装 edge_tts）...」：解释器没找到或缺库，设置 `config.pythonPath`。
  - 「语音合成失败（...）」：edge-tts 调用失败，括号里是子进程的最后一行错误。
  - 「浏览器阻止了自动播放...」：自动朗读被自动播放策略拦下，点一下朗读按钮。
  - 「这条消息没有可朗读的文字」：该回复只有工具调用或代码，没有正文。
  - 「插件后端没有注册路由，重启 DSH 后再试」：插件在 `webServer` 服务就绪前启动过；本插件已用 `inject: ['webServer']`，正常安装不会出现。
- Host 控制台会打印 `[reply-voice]` 开头的错误。
## 开发与发布

```powershell
npm test                 # 模块契约、客户端注册、TTS 空输入测试
npm pack                 # 生成 dsh-reply-voice-1.1.0.tgz
npm run release         # 跑测试、打包到 release/ 并写 SHA256SUMS.txt
```

`npm pack` 前会自动运行测试（`prepack`）。仓库还包含 `.github/workflows/release.yml`：推送 `v*` 标签后，会用 `NPM_TOKEN` 发布到 npm 并把 tarball 上传到 GitHub Release。发布到 npm：

```powershell
npm login
npm publish
```

生成插件索引或上传 GitHub Release 时，可直接使用 `npm pack` 生成的 tarball 和 `SHA256SUMS.txt`。

## 隐私

只把待朗读的文字发送给 edge-tts 的在线语音服务；插件不保存音频、不写临时文件，也不收集遥测。
## English

Reads every finished assistant reply aloud and adds read-aloud actions to each assistant message: click the speaker to speak it, pause in place, resume where it stopped, or stop and free the buffer. Synthesis happens in memory on the Host (`scripts/tts.py` streams MP3 bytes on stdout), and the page plays them through one shared `<audio>` element, so nothing is written to disk and pause/resume are native.

Install with `dsh plugin --profile desktop add dsh-reply-voice`, then start `dsh --profile desktop`. Configuration is the `config` block of the `reply-voice` row (`voice`, `rate`, `volume`, `maxChars`, `includeSubagents`, `autoPlay`, `pythonPath`). Automatic reading depends on the browser's autoplay permission; when it is refused, the tooltip says so and the button still works. Requires Python 3 with the `edge_tts` package.

MIT licensed.
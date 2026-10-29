# pi-aside

[English](README.md) | [简体中文](README.zh-CN.md)

[Pi](https://pi.dev) 扩展：**在主对话旁边阅读与提问**。

主代理跑它的，你在浮层面板里问你的。问答不进主 session、不 steer/queue 主代理，
也不会滚动或重排 transcript —— 关掉面板，你还在原来那一屏。

适合这种场景：AI 正在写长文档或跑任务，你想边看边问「这个词什么意思」
「这段代码为什么这么写」，又不想弄乱主对话上下文、不想让正在看的输出被顶走。

## 特性

- **上下文隔离**：侧边问答有独立消息数组，直接调用 `ctx.modelRegistry.streamSimple()`，
  不经过 `session.prompt()`，不写 session 文件。
- **不打断主代理**：独立请求；主代理流式输出时也能提问，不会插进当前 run。
- **能真正离开**：`Esc` 只关闭面板，回答在后台继续；footer 显示进度，
  回答完成可弹通知。多个 tab 可以并行跑（最多 9 个）。
- **不挤走阅读位置**：overlay 面板，主 transcript 不动，再打开回到原位置。
- **自动附带只读快照**：打开 tab 时抓取最后一条助手回复 + 它之前的用户请求，
  作为 side model 的参考资料。
- **可选写回**：`Ctrl+Y` 才把某一轮问答写回主对话（只有这一步会进入主上下文）。
- **可单独选模型**：用便宜/快的模型做解释，不必占用主模型额度。
- **双语界面**：默认跟随系统语言 —— 只有系统语言为中文时才用中文，其它语言一律英文。面板标题会显示当前模式和快捷键（`Language: Auto (EN) · Ctrl+L` / `语言：跟随系统（中文）· Ctrl+L`）；按 `Ctrl+L`（全屏模式下也可点击标签）在 跟随系统 / 英语 / 中文 之间循环，选择会写入 `aside.json`。

## 安装

```bash
git clone https://github.com/meatRoll/pi-aside.git
cp pi-aside/aside.ts ~/.pi/agent/extensions/aside.ts
```

之后在 pi 里执行 `/reload`（或重启 pi）即可。

更新：

```bash
cd pi-aside && git pull && cp aside.ts ~/.pi/agent/extensions/aside.ts
```

不想装到全局，也可以只在需要时加载：

```bash
pi --extension /path/to/pi-aside/aside.ts
```

卸载：

```bash
rm ~/.pi/agent/extensions/aside.ts
```

> 只保留一份。同一扩展如果既装在用户扩展目录、又由 package 加载，Pi 会注册成
> `aside:1` / `aside:2`，旧的那份可能反而生效。删掉其中一份再 `/reload`。

## 使用

- `Ctrl+Shift+S` 打开面板
- 或在主编辑器里输入 `/aside 这条命令是什么意思？` 打开并带上第一个问题
- 面板标题会显示当前模式和快捷键（`Language: Auto (EN) · Ctrl+L` / `Language: EN · Ctrl+L` / `语言：跟随系统（中文）· Ctrl+L` / `语言：中文 · Ctrl+L`）；按 `Ctrl+L` 在 跟随系统 / 英语 / 中文 之间循环，全屏模式下也可点击标签
- `/aside-model [provider/model]` 选择侧边模型

主代理正在输出时也可以直接用。

| 键 | 作用 |
|---|---|
| `Enter` | 发送问题 |
| `Shift+Enter` / `Ctrl+J` | 换行 |
| `Esc` | 关闭面板（请求在后台继续） |
| `Ctrl+C` | 取消当前 tab 的生成 |
| `Ctrl+N` | 新建 tab |
| `Ctrl+X` | 关闭当前 tab |
| `Shift+←` / `Shift+→` | 切换 tab |
| `PageUp` / `PageDown` | 滚动侧边回答 |
| `Alt+↑` / `Alt+↓` | 单行滚动 |
| `Ctrl+End` | 回到最新 |
| `Ctrl+R` | 重新抓取主对话快照 |
| `Ctrl+T` | 开关快照（关掉后侧边模型只看得到你的问题） |
| `Ctrl+L` | 循环切换界面语言：跟随系统 / 英语 / 中文（写入 `aside.json`） |
| `Ctrl+Y` | 把最近一轮问答写回主对话（**这一步才会进入主上下文**） |

鼠标滚轮也可以滚动。

## 配置

可选文件：`~/.pi/agent/aside.json`

```json
{
  "language": "auto",
  "model": "provider/model-id",
  "maxContextChars": 24000,
  "maxAnswerTokens": 4000,
  "notifyOnComplete": true
}
```

- `language`：界面和系统提示词语言 —— `"auto"`（默认）跟随系统：只有系统语言为中文时用中文，其它语言一律英文。`"en"` / `"zh"` 强制指定语言。在面板里按 `Ctrl+L` 在 跟随系统 / 英语 / 中文 之间循环，选择结果会写回这个文件。
- `model`：侧边问答单独用的模型；不填则跟随主对话当前模型。
  也可以随时用 `/aside-model` 选择，选择结果会写回这个文件。
- `maxContextChars`：快照最多保留多少字符（超出时保留头 65% + 尾 35%，中间标注省略）。
- `maxAnswerTokens`：单次侧边回答的 token 上限。
- `notifyOnComplete`：面板关闭时，回答完成是否弹通知。

侧边请求固定使用 `reasoning: "low"`，优先速度。

## 它是怎么隔离的

1. 侧边问答有独立的消息数组，直接调用
   `ctx.modelRegistry.streamSimple(model, ...)`，**不经过** `session.prompt()`，
   所以不会往 session 文件里写任何消息。
2. 请求 context 里不声明 tools，模型没有改文件或执行命令的能力。
3. 打开 tab 时抓一份**只读快照**：当前分支最后一条助手回复 + 它之前的一条用户请求。
   快照作为 system prompt 的一部分发送，是副本；主对话后续的变化不会自动同步
   （需要 `Ctrl+R`）。
4. 面板是 overlay：主 transcript 不滚动、不重绘位置，阅读位置保持不变。

唯一会写回主对话的操作是 `Ctrl+Y`，它用 `pi.sendMessage()` 添加一条
`aside` 类型的 custom message（会进入后续模型上下文）。

## 已知边界

- 快照只包含主对话的文本，不含工具输出、文件内容和图片。
  想解释某个文件，把相关片段贴进问题里更可靠。
- 侧边请求的 token 用量显示在面板状态行，但不计入主 session 的用量统计。
- 侧边历史保存在内存里；切换 session（`/new`、`/resume`、`/fork`）或退出 pi 后清空。
- 只支持 TUI 模式；RPC/print 模式下 `/aside` 会给出提示。
- `Ctrl+Shift+S` 可能被个别终端占用；用 `/aside` 命令总能打开。

## 开发

类型检查可以复用 pi 源码仓库的 `tsconfig.json` 与路径映射：

```bash
cd /path/to/pi
cat > tsconfig.aside.json <<'JSON'
{ "extends": "./tsconfig.json", "include": ["/path/to/pi-aside/aside.ts"] }
JSON
./node_modules/.bin/tsc -p tsconfig.aside.json --noEmit 2>&1 | grep aside
rm tsconfig.aside.json
```

在 pi 里测试本地修改：

```bash
cp aside.ts ~/.pi/agent/extensions/aside.ts
# 然后在 pi 里执行 /reload
```

> 注意：`pi` 的启动脚本在源码仓库存在未跟踪文件时会拒绝 `git pull`，
> 检查完记得删掉临时的 `tsconfig.aside.json`。

## License

MIT

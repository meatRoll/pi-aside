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
- **选中的文字会作为引用**：在主对话里选中文字后打开 Aside，片段会开在一个单独的 tab 里
  （当前 tab 还是空的就先复用），并且只随这个 tab 的下一个问题发送一次，发完就丢掉：
  不会带进新 tab，重新打开弹窗也不会重复引用；按 `Alt+S` 可以让下个问题不带这段引用。
- **只读工具**：快照不够时，侧边模型可以用 `read` / `grep` / `find` / `ls` 查看仓库和系统文件，
  用 `git` 读取提交历史与 commit 数据，用 `web_fetch` 抓取具体 URL。没有 `bash`、`edit`、`write` 工具，所以不能修改文件。
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

想引用主对话里的文字：先在主 transcript 里选中（全屏模式，默认；由 Pi 管理选区），
再按 `Ctrl+Shift+S` 或执行 `/aside`。选中的片段会开在一个单独的 tab 里，只随这个 tab
的**第一个**问题发送一次，发完从 tab 上删掉：新 tab 不会继承，关掉面板再打开也不会
出现。选区本身在打开时就被消费掉，不重新选不会重复引用；按 `Alt+S` 可以让下个问题
不带这段引用。

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
| `Ctrl+T` | 开关快照（关掉后侧边模型只看得到你的问题和选中的文字） |
| `Alt+S` | 开关当前 tab 的选中文字引用 |
| `Ctrl+G` | 开关当前 tab 的只读工具 |
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
  "maxSelectionChars": 8000,
  "notifyOnComplete": true,
  "tools": true,
  "git": true,
  "network": true,
  "maxToolResultChars": 12000
}
```

- `language`：界面和系统提示词语言 —— `"auto"`（默认）跟随系统：只有系统语言为中文时用中文，其它语言一律英文。`"en"` / `"zh"` 强制指定语言。在面板里按 `Ctrl+L` 在 跟随系统 / 英语 / 中文 之间循环，选择结果会写回这个文件。
- `model`：侧边问答单独用的模型；不填则跟随主对话当前模型。
  也可以随时用 `/aside-model` 选择，选择结果会写回这个文件。
- `maxContextChars`：快照最多保留多少字符（超出时保留头 65% + 尾 35%，中间标注省略）。
- `maxSelectionChars`：主对话中选中的文字最多保留多少字符（默认 8000；超出时中间标注省略）。
- `maxAnswerTokens`：可选，单次侧边回答的 token 上限；不填则用模型自己的输出上限（和主对话一致）。
- `notifyOnComplete`：面板关闭时，回答完成是否弹通知。
- `tools`：启用只读工具（`read`、`grep`、`find`、`ls`、`git`），默认 `true`。
  在面板里按 `Ctrl+G` 可以按 tab 开关。
- `git`：启用只读 git 工具，默认 `true`；只有 `tools` 开启时才会注册。
  支持 `show`、`log`、`diff`、`blame`、`file`、`status`、`branches`、`tags`、`ls`，
  不提供任意 git 命令执行。
- `network`：启用 `web_fetch` 抓取具体 http(s) URL，默认 `true`；只有 `tools` 开启时才会注册。
  没有网页搜索。
- `maxToolResultChars`：单次工具结果最多保留多少字符（默认 12000）。
  内置工具本身已经会截断大输出，这是第二道 token 预算。

除了两条刻意的限制（没有 `write`/`edit`/`bash` 工具；独立的消息数组，不碰 session 文件），
aside 的行为和主对话一致：模型默认跟随主模型（可用 `model` 覆盖），思考强度跟随主对话，
而且没有工具调用预算 —— 工具在整轮里一直声明着，provider 才能一直把调用解析成结构化字段。

压缩不自己定义策略，而是跟随主对话当前的压缩方案：

- **pi-compactor 接管压缩时**（`~/.pi/agent/compactor/lifecycle.json` 里 enabled）：
  用它 `~/.pi/agent/compactor/settings.json` 里的 `trigger` / `target`，包括仓库层规则和
  按模型的覆盖，所以阈值和主对话完全一致。
- **否则**：用 pi 自己的 `compaction` 配置（`enabled`、`reserveTokens`、`keepRecentTokens`、
  `modelOverrides`）。

摘要本身走 pi 的摘要器（`generateSummary`），模型和思考强度与主对话相同。遇到 context
overflow 报错时，aside 会折叠一次并重试这一轮，和主对话一样。面板里仍然显示完整历史，
只有发给模型的消息数组被压缩。

## 它是怎么隔离的

1. 侧边问答有独立的消息数组，直接调用
   `ctx.modelRegistry.streamSimple(model, ...)`，**不经过** `session.prompt()`，
   所以不会往 session 文件里写任何消息。
2. 侧边 agent 自己跑工具循环，工具集是封闭的：`read`、`grep`、`find`、`ls`、`git`，
   以及可选的 `web_fetch`。`bash`、`edit`、`write` 从来不会注册，所以模型没有任何
   能修改文件的工具。`git` 工具只接受固定的只读动作和固定参数；`web_fetch` 只发 GET，
   并拒绝 localhost/内网地址。
3. 打开面板时抓一份**只读快照**：当前分支最后一条助手回复 + 它之前的一条用户请求，
   外加主对话里当前选中的文字。它们都作为 system prompt 的一部分发送，是副本；
   主对话后续的变化不会自动同步（快照按 `Ctrl+R` 刷新；选中文字需要重新选中并打开，
   引用过的选区在打开时就被清掉）。
4. 面板是 overlay：主 transcript 不滚动、不重绘位置，阅读位置保持不变。

唯一会写回主对话的操作是 `Ctrl+Y`，它用 `pi.sendMessage()` 添加一条
`aside` 类型的 custom message（会进入后续模型上下文）。

## 已知边界

- 没有网页搜索。`web_fetch` 只在你已经有具体 http(s) URL 时可用；搜索需要搜索 API，
  目前没有接入。
- 读取范围没有限制在项目内：侧边模型能读 pi 进程能读的任何文件（和主代理一样）。
  它仍然不能写，但不要把不想发给模型提供商的密钥文件指给它。
- `grep` 和 `find` 依赖 `rg` / `fd`；和主代理一样，缺失时 pi 可能会在首次使用时自动下载。
- `git` 工具刻意做得很窄：可以看 commit、diff、文件和历史，但不能 commit、checkout、
  stash、建分支/tag，也不能执行任意 git 命令。
- 引用选中文字需要全屏模式（由 Pi 管理鼠标选区）。普通模式下选区归终端管理，
  Aside 读不到。引用只会被发送一次：选区在打开时从 transcript 清掉，片段在随问题
  发送后从 tab 删掉；要再引用就重新选。
- 选中的文字和快照一样会发给模型提供商。不要把不想发给提供商的密钥选进来。
- 工具调用会增加 token 和延迟。只用快照就够时可以用 `Ctrl+G` 关掉。
- 侧边请求的 token 用量显示在面板状态行，但不计入主 session 的用量统计。
- 侧边历史保存在内存里；切换 session（`/new`、`/resume`、`/fork`）或退出 pi 后清空。
- 没有单独的侧边上下文上限：触发点就是主对话的。装了 pi-compactor 时那个 `trigger`
  （按模型的百分比或 token 数）说了算；没装时才是 `compaction` 配置里的
  `contextWindow - reserveTokens`。保留多少近期原文看 `target` / `keepRecentTokens`。
  pi-compactor 的 **session 级**覆盖只存在内存里，aside 读不到。
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

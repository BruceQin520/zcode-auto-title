# zcode-auto-title

给 **ZCode 会话自动命名补全**。ZCode 引擎自带生成式会话命名，但触发条件很窄，实际用起来
总有一批会话永远顶着「首句原文截断」的标题。这个插件补的就是那批。

> Fills in the session titles that ZCode's built-in namer skips. The built-in
> `session_title_generation` only fires once, on the first turn, for top-level interactive
> sessions with a first message of ≥10 chars — everything else keeps its raw first prompt
> forever. This plugin adds a `Stop` hook that names those sessions, plus a backfill for
> history. See [背景](#背景内置命名为什么不够) for the mechanism.

## 背景：内置命名为什么不够

引擎里的 `session_title_generation`（`generateAndPersistSessionTitle`）只在**会话第一轮**
尝试一次，并且要求：

| 条件 | 说明 |
|---|---|
| 只在第 0 轮 | `turnNumber===0`，之后不再重试 |
| 无父会话 | 子智能体会话（`parent_id` 非空）一律跳过 |
| `task_type=interactive` | fork / side-chat / 子会话等跳过 |
| 首条消息 ≥10 字 | `Iba=10`，`你好`、`继续` 这类开场白直接放弃 |
| 一次成功 | provider runtime headers 触发「延迟生成」时，要等下一轮模型请求才执行 —— 单轮会话往往等不到 |

任何一条不满足，该会话的标题就永久留在首句截断状态（`title_source='first_input'`）。
实际跑下来，顶层会话里常年有一成左右属于这种情况（短开场白、单轮任务、延迟生成没等到
下一轮）。

**本插件（`auto-title`）** 在 `Stop` 钩子里补生成：引擎没覆盖或漏掉的会话，用同一条
提示词（与内置逐字一致，保证风格统一）生成 3–7 词的语义标题，写回引擎会话库和 App
任务列表；另提供历史会话批量补名。

## 安装

### 方式 A：配置钩子（不装插件也能用）

`~/.zcode/cli/config.json`：

```json
{
  "hooks": {
    "enabled": true,
    "events": {
      "Stop": [
        { "hooks": [ {
            "type": "command",
            "command": "python3 /绝对路径/plugins/auto-title/hooks/auto_title.py",
            "timeout": 30,
            "statusMessage": "Auto-title session"
        } ] }
      ]
    }
  }
}
```

钩子在**会话开始时快照**，改完要开新会话才生效。

### 方式 B：插件市场安装（可发布）

1. 设置 → 插件 → 创建 → 添加插件市场，填本仓库根目录（含 `marketplace.json`）。
2. 在「个人」分段安装 `auto-title`。插件是复制到
   `~/.zcode/cli/plugins/cache/` 加载的，改源文件后要刷新市场并重装。
3. 插件自带 `hooks/hooks.json`（`Stop` 事件），安装即注册，无需再配 config.json。
   —— 两种方式二选一，同时开会让脚本跑两遍（第二次会被 `title_source` 判重跳过，但没必要）。

## 用法

```bash
# S = 插件内脚本的绝对路径，例如
#   ~/.zcode/cli/plugins/cache/<marketplace>/auto-title/<version>/hooks/auto_title.py
S=<插件目录>/hooks/auto_title.py

python3 $S --backfill --dry-run        # 看哪些会话会被补名（不调用模型、不写入）
python3 $S --backfill                  # 批量补齐历史会话
python3 $S --session sess_xxx          # 立刻给某个会话命名
python3 $S --force --session sess_xxx  # 强制重新命名（含已生成的）
python3 $S --backfill --include-subagents   # 连子智能体会话一起命名（默认跳过）
```

也可以在会话里用 `/retitle` 命令让 agent 调用脚本。

## 配置

`~/.zcode/auto-title/config.json`（可选，缺省即用默认值；示例见
`plugins/auto-title/config.example.json`）：

| 键 | 默认 | 说明 |
|---|---|---|
| `enabled` | `true` | 总开关 |
| `provider` | 自动 | `~/.zcode/v2/config.json` 里的 provider id，如 `builtin:bigmodel` |
| `model` | 自动挑 flash | 模型 id，如 `GLM-5.3-Flash` |
| `maxTitleChars` | `48` | 标题上限，按词边界截断 |
| `minInputChars` | `6` | 取素材时的最小长度；短开场白会向后扫前 10 条用户消息 |
| `nameOnFirstStop` | `false` | `true` = 第一次 Stop 就补，不等内置命名 |
| `includeSubagents` | `false` | 是否给子智能体会话命名（它们在 App 列表里是隐藏的） |
| `httpTimeoutSec` | `20` | 模型调用超时 |

日志：`~/.zcode/auto-title/auto-title.log`（保留最近 400 行）。环境变量
`ZCODE_AUTO_TITLE_OFF=1` 可单次禁用。

## 写回与安全规则

- 引擎库 `~/.zcode/cli/db/db.sqlite` → `session.title` + `title_source='generated'` +
  `time_title_updated`；**`title_source='custom'`（用户手动改过）永不覆盖**。
- App 任务列表 `~/.zcode/v2/tasks-index.sqlite` → `tasks.title`；只写
  `title_overridden=0` 的行，用户改过名的（=1）不碰。
- 两个库都是 WAL + `busy_timeout=5000` 短事务，写不进去只记日志，不影响会话。

## 已知限制

- App 任务列表的刷新时机取决于客户端：外部写入的标题可能要切换/重开列表才显示
  （内置命名走的是引擎事件，能即时刷新）。
- 生成依赖 `~/.zcode/v2/config.json` 里已配置好的 provider；全部不可用时会记
  `no usable provider` 并跳过。
- 思考型模型偶尔返回空文本（思考块占满预算），脚本会重试一次。

## 发布

仓库根目录就是插件市场（`marketplace.json` + `plugins/auto-title`），推到 GitHub 后
任何人在「设置 → 插件 → 添加插件市场」里填仓库地址即可安装。发版要同时改
`marketplace.json` 条目和 `.zcode-plugin/plugin.json` 的 `version`。

MIT.

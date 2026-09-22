# 给 ZCode 的 feature request 草稿（可直接贴到官方仓库 issue）

**标题**：希望会话重命名有外部可用的通道，或让内置命名在第一轮开始时就发 `session_title_updated`

## 背景

ZCode 桌面端的会话标题只在收到引擎事件 `session.titleUpdated` 时更新（App 侧 `applyTitleChange`）。
外部进程直接写 `~/.zcode/cli/db/db.sqlite` 的 `session.title` 与 `~/.zcode/v2/tasks-index.sqlite`
的 `tasks.title`，数据层立即生效，但界面不会刷新——已实测：写入后列表无变化，直到引擎自己
改名（第一轮结束时的内置命名）才更新。

同时，内置命名 `session_title_generation` 只在**第一轮结束时**落地标题，并且只覆盖
「顶层 + interactive + 首条消息 ≥10 字 + 一次尝试成功」的会话。

## 痛点

长任务（几十分钟）或中途被中断的会话，在整个执行期间列表里一直显示首句原文截断，
用户从外面无法判断这个会话在做什么；被中断的会话如果内置命名没跑完，就永远没有可读标题。

## 请求（任选其一即可解决）

1. 给外部进程一个受支持的改名通道（例如让 hook 能返回一个 `sessionTitle` 指令，
   或把 `renameSession` 命令暴露到本地 IPC / app-server 协议）；
2. 或者让内置命名在**第一轮开始**（首个用户输入到达）时就先落一个标题并发事件，
   结束时再按需要精修（现在的行为是只在结束时写一次）；
3. 或者在会话列表侧支持重新读取 `tasks-index.sqlite`（例如定时/聚焦时刷新），
   让外部写入能自然生效。

## 复现环境

- ZCode 0.16.9（macOS）
- 实测：插件在提交后 3 秒写入两个库；App 列表无变化；引擎在 93 秒后（第一轮结束）
  写入自己的标题并发事件，列表此时才更新。

---
description: 立刻重新生成当前会话标题（调用 auto-title 脚本）
---

重新生成当前会话的标题，步骤：

1. 定位脚本：优先 `~/.zcode/workspace/default/zcode-auto-title/plugins/auto-title/hooks/auto_title.py`；
   若不存在，在 `~/.zcode/cli/plugins/cache/` 下查找 `auto-title` 插件目录里的同名脚本。
2. 用 Bash 运行（把 `<脚本>` 替换成上一步的真实路径）：

   ```
   python3 <脚本> --force --session "$CLAUDE_SESSION_ID"
   ```

3. 把脚本输出的结果（新标题 / 跳过原因）回报给我，不要自己编标题。

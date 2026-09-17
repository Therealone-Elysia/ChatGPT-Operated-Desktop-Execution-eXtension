# ChatGPT Operated Desktop Execution eXtension（C.O.D.E.X.）

**C**hatGPT **O**perated **D**esktop **E**xecution e**X**tension

个人维护的 Mac 工作流 / Skill 集合，不是 OpenAI 官方产品，也不包含驱动实现。

## 能力范围

- Word 正文与表格读取、最小文字修改及保存核验。
- 桌面文件定位与打开。
- 使用现有 Safari 访问网页、输入、下载及网页正文保存。
- 通过 GitHub 网页发布通用 Skill 文件。

## 使用原则

优先使用原生 MCP；宿主缺少直接入口时，通过单次执行中继调用已有的 Hiro/Hermes 原生工具。

区分实际 Mac 与聊天容器，不改动承载当前聊天的窗口。动作报错先核对实际结果；用户确认成功后停止重试。

工作流文件：[`skills/hiro-mac-workflow/SKILL.md`](skills/hiro-mac-workflow/SKILL.md)。

依赖用户自己已有的 Hiro/Hermes 环境及相应应用授权。安装 Skill 不会自动授予应用控制、文件访问或其他权限。

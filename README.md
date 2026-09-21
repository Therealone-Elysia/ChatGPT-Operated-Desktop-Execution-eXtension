# ChatGPT Operated Desktop Execution eXtension（C.O.D.E.X.）

**C**hatGPT **O**perated **D**esktop **E**xecution e**X**tension

个人维护的 Mac 工作流 / Skill 集合，不是 OpenAI 官方产品，也不包含驱动实现。

## 能力范围

- Word 正文与表格读取、最小文字修改及保存核验。
- 桌面文件定位与打开。
- 使用现有 Safari 访问网页、输入、下载及网页正文保存。
- 通过 GitHub 网页发布通用 Skill 文件。
- 在另一台 Mac、另一 ChatGPT 账号或另一 GitHub/网站账号上部署、迁移并调试 Hiro/Hermes/C.O.D.E.X. 桥接，包括 Operator/Codex、macOS 权限、Safari 应用清单、分层验收和常见故障定位。

## 部署与迁移

`hiro-mac-workflow` v2.1.6 包含完整的跨设备/跨账号部署与调试指南：区分 ChatGPT 账号连接、网站账号登录与新 Mac 本机部署；不复制旧账号 Cookie/OAuth/token，不迁移 TCC 数据库，也不通过关闭认证或系统保护来“修复”连接。部署后按服务、MCP 工具、文件、Codex、原生桌面、Safari 和 ChatGPT 客户端逐层验收。

v2.1.6 还补充了两类 native-ui 恢复流程：一是 capability manifest 空闲超时后，关闭旧 native session、同名 workspace 重开、重新 `desktop_open` 并获取新的窗口/snapshot/token；二是修改 `expires_after` / `idle_timeout` 等已批准 manifest 后，区分 Hermes gateway 与承载 `bridge.py` 的 `local.hermes-gpt.server`，完成 approved hash、服务重载、新 session 与 bounded read-only 调用四层验收。全桌面观察被 `bounded_resource_outside_manifest` 拒绝时不扩大权限，而是先通过已批准应用的 `desktop_open(new_instance=false)` 注册目标，再做 PID 范围内的只读验证。

## 使用原则

优先使用原生 MCP；宿主缺少直接入口时，通过单次执行中继调用已有的 Hiro/Hermes 原生工具。

区分实际 Mac 与聊天容器，不改动承载当前聊天的窗口。需要浏览 GitHub、发布 Skill 或做网页核验时，默认在 Safari 新建独立窗口执行并复用该任务窗口。动作报错先核对实际结果；用户确认成功后停止重试。

工作流文件：[`skills/hiro-mac-workflow/SKILL.md`](skills/hiro-mac-workflow/SKILL.md)。仓库根目录的 `SKILL.md` 保留为同内容入口，便于直接查看。

跨平台移植（Windows / Linux）说明：[`docs/cross-platform.md`](docs/cross-platform.md)。

## 平台说明

原始目标平台是 **macOS**。底层驱动 [cua-driver](https://github.com/trycua/cua) 本身跨平台（macOS / Windows / Linux），但本仓库的工作流文件包含大量 macOS 专有内容：
TCC 权限章节、`com.apple.*` 应用标识、Safari 桌面路线、launchd 自启动。

移植到其他平台时：

- **可移植**：能力分层思路、会话与身份校验流程、故障定位表、分层验收清单、Word/文件读写。
- **不可移植**：TCC 授权、`bundle_id` 标识、Safari 分支、`LaunchAgent` 自启动、本机绝对路径。
- **替换路线**：桌面动作走 cua-driver 统一接口；浏览器任务改用 `browser_*` 系列而非 Safari 桌面控制；自启动改用目标平台机制（Windows: `cua-driver autostart`）。

具体步骤与自检清单见 [`docs/cross-platform.md`](docs/cross-platform.md)。该文档区分「本机已验证」与「待目标机验证」两类结论 —— 移植时不要把前者当后者用。

依赖用户自己已有的 Hiro/Hermes 环境及相应应用授权。安装 Skill 不会自动授予应用控制、文件访问或其他权限。

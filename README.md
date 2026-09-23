# ChatGPT Operated Desktop Execution eXtension（C.O.D.E.X.）

**C**hatGPT **O**perated **D**esktop **E**xecution e**X**tension

个人维护的工作流与架构说明，不是 OpenAI 官方产品。目标是在用户自己的 PC / Mac 上复用**已经安装、已授权的 Hermes-GPT 通道**，通过自建 Web Chat 或 ChatGPT 插件调用 Hermes 的 `computer_use`，实现从网页发起、在本机执行并回传结果的桌面操作。

本仓库目前发布的是 Skill 和移植文档；**不包含** Hiro/Hermes-GPT 桥接服务、Web Chat 后端/前端、认证配置或桌面驱动。因此克隆仓库或安装 Skill 不会自动建立上述链路。完整组件边界、两种网页入口和验收方法见 [架构与接入说明](docs/architecture.md)。

## 核心调用链

```text
用户网页请求
  → 自建 Web Chat 后端（OpenAI API 工具调用）
     或 ChatGPT 插件（自建 MCP 服务）
  → 已认证的 Hiro/Hermes-GPT 桥接
  → 目标电脑上已有的 Hermes computer_use
  → cua-driver / 系统桌面
  → 观察结果逐层返回网页
```

OpenAI API 提供模型和工具调用流程；实际桌面动作由用户部署的服务、Hermes 和驱动执行。Web Chat 与 ChatGPT 插件是两种入口，不能把其中一种的安装步骤当成另一种的部署证明。

## 自建插件的名称与调用

创建网页端电脑控制插件时，由使用者给它取一个**独立、明确的名称**；不要把新插件命名为 `Hiro`。`Hiro` 在本文中只指现有本机桥接/插件，不能当作新插件的别名。调用错同名插件可能把动作送到另一台电脑，包括原有 Hiro 所连接的本机。

每次网页聊天要使用新插件，都应在**本次请求**中明确写出自己取的插件名，并在客户端选择/启用对应连接。例如：“使用我创建的「〔你取的插件名〕」插件，在它绑定的电脑上查看当前窗口。”其中方括号内容要替换成实际名称。如果请求只写“用 Hiro”、只说“控制电脑”或没有点名新插件，就不要默认选择 Hiro 或任何其他电脑控制连接。服务端还必须核对连接身份和绑定的设备；**名字与提示词只是防误选措施，不能代替服务端授权**。详细规则见 [架构与接入说明](docs/architecture.md#插件命名与每次显式调用)。

## 当前 Mac Skill 的能力范围

- Word 正文与表格读取、最小文字修改及保存核验。
- 桌面文件定位与打开。
- 使用现有 Safari 访问网页、输入、下载及网页正文保存。
- 通过 GitHub 网页发布通用 Skill 文件。
- 在另一台 Mac、另一 ChatGPT 账号或另一 GitHub/网站账号上部署、迁移并调试 Hiro/Hermes/C.O.D.E.X. 桥接，包括 Operator/Codex、macOS 权限、Safari 应用清单、分层验收和常见故障定位。

## 部署与迁移

`hiro-mac-workflow` v2.2.1 包含跨设备/跨账号部署与调试指南：区分 ChatGPT 账号连接、网站账号登录与新 Mac 本机部署；不复制旧账号 Cookie/OAuth/token，不迁移 TCC 数据库，也不通过关闭认证或系统保护来“修复”连接。部署后按服务、MCP 工具、文件、Codex、原生桌面、Safari 和实际网页客户端逐层验收。

该 Skill 还补充了两类 native-ui 恢复流程：一是 capability manifest 空闲超时后，关闭旧 native session、同名 workspace 重开、重新 `desktop_open` 并获取新的窗口/snapshot/token；二是修改 `expires_after` / `idle_timeout` 等已批准 manifest 后，区分 Hermes gateway 与承载 `bridge.py` 的 `local.hermes-gpt.server`，完成 approved hash、服务重载、新 session 与 bounded read-only 调用四层验收。全桌面观察被 `bounded_resource_outside_manifest` 拒绝时不扩大权限，而是先通过已批准应用的 `desktop_open(new_instance=false)` 注册目标，再做 PID 范围内的只读验证。

## 使用原则

网页入口由自建后端或 MCP 服务调用已认证的本机桥接；现有 Mac Skill 在宿主直接提供原生 MCP 时优先使用它。宿主缺少直接入口且已有受授权中继时，可通过单次执行中继调用现有 Hiro/Hermes 工具。

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

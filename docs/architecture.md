# 网页调用本机 Hermes computer use：架构与接入边界

## 目标与现状

目标是让用户在网页聊天中提出任务，模型通过受控工具调用用户自己电脑上**已经运行、已经授权**的 Hermes-GPT/Hiro 通道，再由 Hermes 的 `computer_use` 调用 `cua-driver` 操作 Windows 或 macOS 桌面。桌面观察和动作结果沿原链路返回聊天。这里的“类似 Codex 控制电脑”指用户体验和任务闭环，不表示使用 Codex 的内部实现。

本仓库现有文件是 Skill 与部署经验。以下架构描述**接入合同与还需实现的组件**；仓库尚未提供 Web Chat、Hiro/Hermes-GPT 桥接、MCP 服务、凭据管理或可一键安装的 Windows 程序。现有私有部署中的 `hiro_native_*` 工具名与 `capabilities.yaml` 结构不能推定为公开 Hermes 的固定接口；接入时必须以目标设备的实际 `tools/list`、Hermes 版本和授权策略为准。

## 两种网页入口

| 入口 | 模型与工具调用由谁负责 | 本机连接 | 适用场景 |
| --- | --- | --- | --- |
| 自建 Web Chat | 自建后端调用 OpenAI API，并处理模型返回的工具调用及结果 | 后端通过已认证桥接转发到用户设备上的 Hermes | 需要自己的网页界面、账号体系或设备选择 |
| ChatGPT 网页插件 | ChatGPT 连接开发者提供的 MCP 工具服务 | MCP 服务通过已认证桥接转发到用户设备上的 Hermes | 希望直接在受支持的 ChatGPT 网页入口使用 |

两种入口可以共用同一个本机适配器，但**OpenAI API 密钥不使云端模型自动获得本机桌面权限**。自建 Web Chat 的后端需要显式执行工具调用；ChatGPT 插件需要可连接、可认证的 MCP 服务。对应的官方接口分别见 [OpenAI API 工具调用](https://developers.openai.com/api/docs/guides/function-calling) 与 [ChatGPT 插件 MCP 快速入门](https://developers.openai.com/plugins/quickstart)。插件可通过公有 HTTPS MCP 端点或开发模式支持的安全隧道接入；具体可用性取决于账号与工作区策略，见 [连接和测试说明](https://developers.openai.com/plugins/deploy/connect-chatgpt)。

## 插件命名与每次显式调用

**新建的网页端电脑控制插件必须由使用者自行取一个与现有插件区分的名称，不能命名为 `Hiro`。** 这里的 `Hiro` 保留给既有本机桥接/插件；它可以继续作为内部后端的名称，但不得被新网页插件借用为显示名或调用别名。误选现有 Hiro 连接可能把桌面动作送往它绑定的另一台电脑（例如原来的本机）。

每次网页聊天希望使用新插件时，用户都应在**当前这条请求**中明确写出所取的插件名，并在客户端选择/启用这个连接。例如：

> 使用我创建的「〔你取的插件名〕」插件，在它绑定的电脑上查看当前窗口。

使用时将方括号内容替换为实际插件名。

在 ChatGPT 网页入口，可通过插件选择器或 `@` 选择对应插件，然后在请求中写明同一名称；具体入口以当前客户端界面为准。[OpenAI 插件快速入门](https://developers.openai.com/plugins/quickstart)演示了 `@` 选择，[连接测试指南](https://developers.openai.com/plugins/deploy/connect-chatgpt)要求从工具菜单添加目标 MCP 连接。自建 Web Chat 应在自己的界面显示插件名称与目标设备，并把用户的选择传给后端。

客户端和服务端按以下顺序处理每一次电脑控制请求：

1. 确认当前请求明确点名**这个自建插件的准确名称**，且客户端选中的插件/连接与之相同。只出现 `Hiro`、只说“控制电脑”、名称不一致或未选中连接时，不推断、不自动回退到 Hiro 或别的电脑控制插件；先让用户指出要用的插件。
2. 在任何桌面工具调用前，使用**只读**身份/设备查询确认该连接当前绑定的目标电脑，并向用户明确目标。若返回设备与用户预期不同，停止调用。可为 MCP 连接提供稳定的只读 profile 工具帮助区分账号；[OpenAI 认证指南](https://developers.openai.com/plugins/build/auth)说明了这种连接识别方式。
3. 后端依据已验证的用户凭据和持久绑定关系决定目标设备，核对每次工具调用所属的插件/连接及会话；拒绝目标不匹配的调用。不可只用插件显示名、用户输入的设备 ID 或模型的选择作为授权依据。

显式命名和选择是**防误用规则**，服务端设备绑定与逐请求授权是**实际隔离边界**。重新命名插件、重连账号或新增设备后，应重新验证显示名称、连接身份和绑定设备；原有 `Hiro` 连接保持独立。

## 调用与信任边界

```text
浏览器中的用户
  │ 请求、选择目标设备、查看动作结果
  ▼
网页入口：自建 Web Chat 后端 / ChatGPT MCP 插件
  │ 验证用户身份；把请求限制到该用户已绑定的设备
  ▼
已认证的远程入口或安全隧道
  │ 传输受限工具调用；不直接公开本机驱动端口
  ▼
用户 PC / Mac 上的 Hiro/Hermes-GPT 适配器
  │ 验证工具名、参数、应用/路径授权、会话与目标窗口
  ▼
Hermes computer_use → cua-driver → 当前登录会话中的桌面
  │ 截图/可访问性树/动作结果
  └───────────────────────────────→ 沿原链路返回网页
```

适配器至少要处理四件事：

1. **发现能力**：启动时发现目标 Hermes 的真实工具及参数；外层只暴露经过设计的观察、定位、输入等工具，不把私有 `hiro_native_*` 名称当作跨设备协议。
2. **绑定身份与设备**：服务端验证每次请求属于哪位用户、哪个自建插件连接、哪台设备及哪段会话；浏览器不能凭自填设备 ID 或插件显示名获得其他设备控制权。OpenAI API 密钥与本机连接凭据只保存在各自服务端。
3. **约束动作**：对应用、文件路径和工具参数实施服务端授权；写入、发送、安装等有后果的动作按实际风险确认。模型提示词、Skill 与 MCP 工具注释不能替代服务端校验。[OpenAI MCP 服务指南](https://developers.openai.com/plugins/build/mcp-server)也要求服务端逐请求授权。
4. **闭环验证**：每次桌面动作记录目标会话、窗口和结果；必要时重新观察桌面，确认实际变化后再向网页报告。超时或结果不确定时先查状态，避免重复点击、输入或提交。

## 需要部署的组件

| 组件 | 在目标电脑还是服务端 | 仓库现状 |
| --- | --- | --- |
| Hermes Agent / 现有 Hermes-GPT 通道及其 `computer_use` | 目标电脑 | 外部依赖，需在目标机安装并验收 |
| `cua-driver` 与桌面权限/登录会话 | 目标电脑 | 外部依赖，需按平台安装并验收 |
| Hiro/Hermes-GPT 适配器及 `hiro_native_*` 扩展（若采用） | 目标电脑或与其可信连接的服务 | 未发布；不能通过安装本仓库重建 |
| 自建 Web Chat 后端和前端 | 开发者服务端/网页 | 未发布；采用 OpenAI API 路线时必需 |
| MCP 服务与认证接入 | 开发者服务端/可信隧道 | 未发布；采用 ChatGPT 插件路线时必需 |
| 本仓库的 Skill 与说明 | 入口/agent 的指令层 | 已发布；只指导选择与调用已存在的能力 |

如果只是让 Hermes **在本机**控制桌面，可以直接使用其内建 `computer_use`；要实现“从网页跨网络控制自己的电脑”，仍须完成上表中的网页入口、认证连接和设备端适配。一个只安装了 `cua-driver` 的 Windows 用户尚不能从 ChatGPT 网页调用自己的电脑。

## Windows 落地与验收

Windows 需要在有图形桌面的交互式登录会话中运行桌面驱动。上游提供 Windows 安装与自启动方法；`cua-driver autostart` 只启动驱动，不负责 Hermes-GPT、Hiro 适配器或隧道。这些服务应分别配置启动、认证和健康检查。Windows 的应用标识、权限及窗口投递与 macOS 不同，不能复用 `com.apple.*`、TCC 或 Safari 规则。详见 [跨平台移植说明](cross-platform.md)、[Hermes computer use](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/computer-use.md) 和 [Cua Driver 安装说明](https://github.com/trycua/cua/blob/main/docs/content/docs/how-to-guides/driver/install.mdx)。

按同一台 Windows 目标机逐层留存成功或失败记录：

1. Hermes 与驱动版本、驱动 `status` / `doctor` 正常，`list_apps` 能看到该登录会话中已打开的测试应用。`doctor` 正常本身不证明能操作窗口。[上游入门验收](https://github.com/trycua/cua/blob/main/docs/content/docs/tutorials/drive-your-first-app.mdx)
2. 从 Hermes 内建 `computer_use` 对测试应用完成一次观察和安全动作，再观察确认效果；记录实际工具 schema 与失败原因。
3. 从本机适配器通过受限会话执行同样操作，核对目标应用/窗口、会话续用、超时恢复和拒绝越界动作。
4. 从自建 Web Chat 或 ChatGPT 插件的**真实网页入口**点名新插件调用一次，确认显示名、所选连接、只读设备身份、设备绑定、工具结果和最终回复均连通；另测试只写 `Hiro` 或省略插件名时不会误调新插件或旧 Hiro 连接。
5. 重新登录或重启相关服务后重复关键路径，分别验证驱动、Hermes-GPT/适配器和远程入口能恢复。

这些项目须在实际 Windows 设备上完成后才能声明“Windows 已验证”。Mac 上的运行记录、上游跨平台声明或只通过安装检查，都不能替代这项结果。

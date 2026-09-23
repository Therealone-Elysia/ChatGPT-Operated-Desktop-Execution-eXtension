# 跨平台移植：macOS → Windows / Linux

本工作流原始目标平台是 macOS。底层驱动 `cua-driver` 本身是**跨平台**的
（[`cua-driver manifest`](https://github.com/trycua/cua) 自述: *cross-platform computer-use automation driver*），
但本仓库的 `SKILL.md` 里有大量 macOS 专有名词与路径。移植到 Windows 前，
必须先分清**哪些是可移植的抽象、哪些是 macOS 实现细节**。

项目目标是“网页请求 → 自建 Web Chat/OpenAI API 工具调用或 ChatGPT MCP 插件 →
已认证的 Hiro/Hermes-GPT 桥接 → 目标机 Hermes `computer_use` → `cua-driver`”。
完整链路、两种网页入口及尚未发布的组件见 [架构与接入边界](architecture.md)。
本页只讨论目标机移植，安装驱动不等于网页端已经可以调用该机器。
网页入口另须遵守[自建插件命名与每次显式调用规则](architecture.md#插件命名与每次显式调用)：
新插件由用户另行命名，不使用 `Hiro`；每次请求点名并选中该连接，服务端核对绑定设备。

> 本文档中的事实分两类，请按标注区别对待：
> **[已验证]** = 在 macOS 本机实测或从本机已安装的二进制/Hermes 源码中读出；
> **[待目标机验证]** = 必须在目标平台上实测，不要在迁移时当假设用。

---

## 1. 能力分层与可移植性

| 层 | 现状 | 可移植性 |
| --- | --- | --- |
| `cua-driver` 驱动本体 | 跨平台二进制 | **可** — 平台各自安装 |
| 桌面动作（点击/输入/滚动/热键/AX 树） | 驱动统一接口 | **可** — 驱动保证三平台同一套工具名 [已验证] |
| `hiro_native_*` MCP 扩展层 | 本机自研 bridge，转调 Hermes `computer_use` 后端；本仓库未发布实现 | **待实现与目标机实调用验收**，不是 Hermes 的固定公开接口 |
| 应用白名单（`capabilities.yaml` 的 `bundle_id`） | `com.apple.*` | **不可** — `bundle_id` 是 macOS 概念 |
| Safari 操作 | `com.apple.Safari` | **不可** — Windows 无 Safari |
| 浏览器自动化（Chromium 路线） | 可选的 agent-browser + Chromium | **有条件可移植** — 只有目标桥接实际提供 `hiro_native_browser_*` 时才能调用 |
| 服务自启动 | macOS launchd `LaunchAgent` | **须分别移植**驱动、Hermes-GPT/桥接和隧道；Windows 驱动可用计划任务 |
| 系统权限模型 | TCC（辅助功能 + 屏幕录制） | **不可** — Windows/Linux 无 TCC |
| Word / 文件读写 | 文件层，与 OS 无关 | **可** |
| GitHub 发布流程 | 网页流程 | **可**（浏览器换成目标机浏览器） |

结论：**方法论文档可移植，macOS 专有名词必须逐条替换。**
不要指望把本仓库原样 clone 到 Windows 就能跑——里面的路径、应用标识、权限章全是 Mac 的。

---

## 2. Windows 落地步骤

### 2.1 安装驱动

[上游 Windows 安装说明](https://github.com/trycua/cua/blob/main/docs/content/docs/how-to-guides/driver/install.mdx)：

```powershell
# Windows (PowerShell)
irm https://cua.ai/driver/install.ps1 | iex
cua-driver autostart kick
```

macOS 对照（本机当前用它装的）：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/cua-driver/scripts/install.sh)"
```

如当前 PowerShell 尚未识别新 PATH，另开窗口。然后在目标机的交互式桌面会话里确认可见应用：

```powershell
cua-driver --version
cua-driver status
cua-driver doctor --json
cua-driver call list_apps
```

`cua-driver permissions status` 检查的是 macOS TCC 授权，Windows 不以它作为就绪条件。
`doctor` 通过或返回零退出码也不能代替 `list_apps` 与一次真实桌面操作。

### 2.2 自启动（这一步和 macOS 完全不同）

[上游自启动说明](https://github.com/trycua/cua/blob/main/docs/content/docs/how-to-guides/driver/keep-running.mdx)：安装器通常会尝试注册自启动；若未注册，须在交互式会话中补做 `enable`。

```powershell
cua-driver autostart enable    # 注册一个登录时触发的计划任务
cua-driver autostart status    # 查看是否已注册 + 守护进程是否在跑
cua-driver autostart kick      # 不重新登录，立刻启动
```

macOS 对应物是 `~/Library/LaunchAgents/*.plist` + `launchctl`。
**不要把 plist 搬到 Windows**，也不要在 Windows 上模仿 `launchctl kickstart`。

[已验证] 驱动 `serve` 子命令自述区分了两条路径：
*macOS 走 proxy/auto-relaunch，Windows 走 autostart 的 Session 1+ 守护进程*。
即 Windows 上有一个登录会话内的常驻守护，权限边界与 macOS 的 app-daemon 代理不同。
`cua-driver autostart` 只管理**驱动**；Hermes-GPT、本机 Hiro 桥接与远程入口/隧道
需要各自的 Windows 启动与健康检查，不能由驱动计划任务代替。

### 2.3 权限模型

| | macOS [已验证] | Windows | Linux [已验证] |
| --- | --- | --- | --- |
| 系统授权 | TCC：辅助功能 + 屏幕录制 | 无 TCC | 无 TCC |
| 授权归属 | 驱动自身身份 `com.trycua.driver`（CuaDriver.app），**不是** Hermes | — | — |
| 就绪判据 | TCC 授权与真实桌面动作 | 驱动健康、应用可枚举且真实动作成功 | 驱动健康、桌面可达且真实动作成功 |
| 可能弹窗 | 系统设置里的授权对话框 | UIAccess worker `cua-driver-uia.exe` 可能触发 SmartScreen | 无 |
| 授予命令 | `cua-driver permissions grant` | 不适用 | 不适用 |

要点：
- **`SKILL.md` 第 D 节整节是 macOS TCC 内容，Windows 上跳过**；确认 `status`、`doctor`、`list_apps` 和一次实际动作及回读。
- Linux 走 X11 / XWayland 辅助控制栈；纯 Wayland 支持程度需在目标机实测。
- Windows 的 UIAccess 与提权窗口限制要在目标权限级别上实测；`doctor` 可帮助定位环境问题，不能单独证明提权窗口可操作。

### 2.4 应用标识：把 `bundle_id` 换成什么

`capabilities.yaml` 当前长这样 [已验证]：

```yaml
resources:
  apps:
    - bundle_id: com.apple.Safari
      launch: true
      windows: all
```

`bundle_id` 以及驱动 `launch_app` / `list_apps` 的语义都是 **macOS `.app` bundle**：

[已验证：`cua-driver describe launch_app` 首句即为 *Launch a macOS app in the background*；
`list_apps` 说明中列出扫描 `/Applications`、`/System/Applications` 等 macOS 目录，`kind` 只返回 `"desktop"`]

因此 Windows 上：

1. **不要沿用任何 `com.apple.*` 值**，包括 Safari。
2. 用**目标平台驱动的真实 schema** 决定应用怎么标 —— 先 `cua-driver list-tools` /
   `cua-driver describe <tool>` 看该平台上 `launch_app` 的入参定义，再写白名单。
   **不要照抄本文档编一个 Windows 应用标识格式。**
3. 如果目标机已有同一套私有桥接，白名单的哈希锁设计可作为移植候选：
   `capabilities.yaml`、`approved.sha256`、`approval.json` 与 `app_policy.py` 的实际行为
   必须在目标部署中核对。本仓库没有这些文件，不能通过克隆本仓库得到该机制。

### 2.5 浏览器路线

Windows 上没有 Safari，**桌面 Safari 分支整段作废**，改用：

- 如果目标桥接的 `tools/list` **确实提供** `hiro_native_browser_*` 系列，可按真实 schema
  使用；本仓库不提供这些工具的实现。使用 Hermes 内建浏览器能力时按其当前文档接入。
- 驱动侧另有 `browser_prepare`（为浏览器准备 DevTools 端点）、`browser_download`
  （把下载强制保存到已批准目录）等能力，做网页任务时优先用它们而非坐标点击。

注意：`SKILL.md` 第 5 节关于 Safari 的窗口发现与双模态身份校验（`hiro_native_app_windows`
+ `hiro_native_verify_window_identity`）**思路可移植、参数不可移植** —— 换成目标浏览器的
可执行体/窗口即可，但具体标识要以目标机 `list_apps` 的真实返回为准。

---

## 3. 迁移时必须改掉的本机绝对路径

以下路径和组件来自原部署的说明，**不在本仓库中**；应在持有原部署的目标机上
逐项核对，不可声称在本仓库 `grep` 就能复现：

| 位置 | 内容 | 迁移动作 |
| --- | --- | --- |
| `config.env` | `HERMES_GPT_OPERATOR_ALLOWED_PATHS=/Users/<user>/Documents,...` | 换成目标机实际目录；**不要**改成整个盘符或用户目录 |
| `config.env` | `HERMES_GPT_CODEX_EXE=/Users/<user>/.hermes/hermes-gpt/hiro-codex` | 换成目标机可执行文件路径 |
| `start-server.sh` | `#!/bin/zsh`，`exec .../venv/bin/python native-ui/main.py` | Windows 用 `.ps1`/`.cmd` 重写；注意 venv 的 `Scripts/python.exe` 而非 `bin/python` |
| `native-ui/bridge.py` | 子进程用 `.../hermes-agent/venv/bin/python` + `start_new_session=True` | 路径改 `Scripts/python.exe`；`start_new_session` 在 Windows 上语义不同，需实测会话/子进程回收行为 |
| `native-ui/worker.py` | 硬编码 `/Applications/Google Chrome.app/Contents/MacOS/Google Chrome` | 换成目标机浏览器真实路径，或用 Hermes 的浏览器注册表 |
| LaunchAgent plists | `local.hermes-gpt.{server,actions,tunnel-login}.plist` | 为 Hermes-GPT/桥接/隧道分别配置 Windows 启动；驱动单独使用 `cua-driver autostart` |
| `tunnel-client/` | `cloudflared` + `*-darwin-arm64.*` | 换对应平台二进制 |

**通用建议：不要在源码里硬编码新平台的绝对路径。**
`SKILL.md` 第 C 节已经写了原则（"复用已验收的本机解释器、工作目录"），
移植时把路径集中到配置文件（如 `config.env`），源码只读配置。

---

## 4. 已验证的坑（跨平台相关）

1. **Windows 上版本号可能自相矛盾** [已验证：Hermes `tools/computer_use/doctor.py` 注释明确记载]
   —— `health_report` 报的版本可能与 `cua-driver --version` 和磁盘上的实际版本不一致
   （上游在 Windows 观察到 health_report 报 0.8.3 而实际是 0.12.6）。
   调试会话问题时**以 `--version` 和 doctor 并列输出为准**，别被单一版本串带偏。
   macOS 本机未复现此现象。

2. **Windows 的 stdout/stderr 编码** [已验证：`doctor.py` 注释]
   —— Windows 会把输出包上系统 ANSI 编码，解析 `cua-driver doctor --json` 时
   必须显式按 UTF-8 解码，否则会抛 `UnicodeDecodeError`。

3. **WSL 混合部署的路径翻译** [已验证：Hermes `cua_backend.py` 中
   `_wsl_windows_path_to_posix()`] —— 当 Hermes 跑在 WSL 里、驱动装在 Windows 侧时，
   驱动清单会返回 `C:\Users\...\cua-driver.exe`，而 Hermes 需要 `/mnt/c/...`。
   Hermes 已内建该翻译；**用第三方脚本自己拼路径时记得补这一步**。

4. **不要用 WSL 里的 Linux 版驱动去驱动 Windows 桌面** —— 两者是不同的驱动实例，
   TCC/权限/应用列表都不是一套。要么全 Windows，要么明确走上级的 WSL 翻译路径。

5. **移植后必须重跑分层验收** —— `SKILL.md` 第 G 节的八层验收（服务/工具/文件/Codex/
   桌面/Safari→浏览器/持久性/客户端）在各平台都适用，但**"Safari 层"要改名为
   "浏览器层"，且不能拿 macOS 的成功记录当作新平台的证据**。
   每层单独标 `verified / blocked / unknown`。

---

## 5. 给 Windows 用户的最短路径建议

如果只是想让 agent 在 Windows 上动桌面，**不必移植本仓库**：

1. 装 Hermes Agent（自带 `computer_use` 工具集，`_RUNTIME_PLATFORMS` 已包含 `win32` [已验证]）。
2. 按 2.1 节安装并启动 `cua-driver`，检查 `status`、`doctor` 和 `list_apps`，再做一次安全动作及回读。
3. 检查 `cua-driver autostart status`；若安装器未注册，在交互式会话中执行 `autostart enable` 和 `kick`。
4. 用上游维护的 skill pack 而不是本仓库：

   ```powershell
   cua-driver skills install    # 从 GitHub Releases 拉版本化 skill pack
   cua-driver skills status     # 查看已链接到哪些 agent（Claude Code / Codex / OpenCode / Hermes 等）
   ```

   [已验证：本机 `skills status` 显示它支持 Claude Code、Codex、Prime Agent、OpenClaw、
   OpenCode、Antigravity、Hermes 七个位置，幂等且不覆盖用户已有链接]

   上游 pack 跟着驱动版本走，跨平台一致性比本仓库这个 macOS 定制工作流好得多。

以上只完成了**本机 agent → Windows 桌面**这部分。如果目标是本项目的网页链路，
还需要在该 PC 上部署并验证已有 Hermes-GPT/Hiro 适配器，给自建 Web Chat 后端或
ChatGPT MCP 服务建立经过认证、绑定该设备的连接，最后从真实网页会话调用一次。
OpenAI API 工具调用由自建后端执行；ChatGPT 插件由 MCP 服务提供工具。
两条入口的要求见 [架构与接入边界](architecture.md)。

本仓库的价值在于**方法论与故障定位**（尤其是 `SKILL.md` 第 H 节的报错码处理表和
"文件更新 / 服务重载 / session 重建 / 受限实调用"四层验收），这些与平台无关，值得照搬思路。

---

## 6. 移植自检清单

- [ ] `cua-driver --version` / `status` / `doctor --json` 输出留档；macOS 再检查 `permissions status`
- [ ] Windows 交互式登录会话中 `cua-driver call list_apps` 非空，且真实桌面动作和回读成功
- [ ] 自启动用目标平台的机制（Windows: `autostart enable`），**没搬 plist**
- [ ] Hermes-GPT/桥接与隧道分别有启动、认证和健康检查，未误认为由驱动自启动管理
- [ ] `capabilities.yaml` 里所有 `com.apple.*` 已换成目标平台真实标识，且已重新走批准流程
- [ ] `approved.sha256` 与 `capabilities.yaml` 实际哈希一致（哈希锁没被绕过）
- [ ] `config.env` / `start-server.sh` 里没有残留 `/Users/...`、`/Applications/...`
- [ ] venv 解释器路径是 `Scripts/python.exe` 而非 `bin/python`
- [ ] 浏览器任务走 `browser_*` 而不是 Safari 桌面路线
- [ ] 第 G 节八层验收**在目标机上重跑**，每层单独标记状态
- [ ] 从选定的自建 Web Chat 或 ChatGPT 插件网页入口完成一次端到端调用和结果回传
- [ ] 自建插件名称与 `Hiro` 区分；每次请求点名并选中它，且只读检查返回预期目标设备
- [ ] 名称缺失、只写 `Hiro`、选错连接或设备不匹配时拒绝桌面动作，不自动切换到其他电脑
- [ ] 报告里不含 token / Cookie / 密码 / 完整个人目录清单

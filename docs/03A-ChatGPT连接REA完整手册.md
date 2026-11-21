# 03A｜ChatGPT ↔ REA MCP 连接完整记录（核心文档）

> **这是整个项目最重要的基础设施文档。** 大肥鱼的逆向分析之所以能由 ChatGPT 直接协助完成，关键不是单独安装了 REA，而是让 ChatGPT 的 MCP 工具请求能够安全、稳定地到达用户电脑上的 REA 服务。本文件尽量把连接结构、组件职责、配置、启动、验证和排错讲清楚。
>
> 记录日期：2026-11-21。内容根据此前实际配置记录整理；如果重新安装或更换版本，应以机器上的当前状态为准。

## 1. 最终要实现的效果

我们希望实现的是：

1. REA 在用户自己的 Windows 电脑上运行。
2. REA 能访问本机文件，例如大肥鱼 EXE。
3. ChatGPT 能看到并调用 REA 暴露出来的 MCP 工具。
4. ChatGPT 发起工具调用后，请求能经过安全通道回到本机 REA。
5. REA 在本机执行分析，再把结果返回 ChatGPT。
6. 不需要把整个大肥鱼程序、磁盘或所有源码上传到 ChatGPT 云端。

简单说：**ChatGPT 负责理解任务、决定调用哪个工具和解释结果；REA 负责在本机真正检查程序。**

## 2. 完整连接结构

```text
┌──────────────────────────────────────────────┐
│ ChatGPT                                      │
│ 已安装的 REA MCP App / 工具入口              │
└──────────────────────┬───────────────────────┘
                       │ MCP 工具请求
                       ▼
             OpenAI Secure MCP Tunnel
                       │
                       │ 云端隧道服务与本机客户端之间
                       │ 的受控连接
                       ▼
┌──────────────────────────────────────────────┐
│ 用户的 Windows 电脑                           │
│                                              │
│  tunnel-client.exe（本机隧道客户端）          │
│          │                                   │
│          │ 转发到本机 MCP URL                 │
│          ▼                                   │
│  REA MCP Server                               │
│  http://127.0.0.1:8765/mcp                   │
│          │                                   │
│          ▼                                   │
│  REA Runtime / 263 个工具（此前 list tools）  │
│          │                                   │
│          ├── 二进制分析 Provider              │
│          ├── Ghidra Provider（需单独验证）    │
│          └── 其他可用分析 Provider            │
│          │                                   │
│          ▼                                   │
│  本机目标文件：DeepSeek Harness.exe 等        │
└──────────────────────────────────────────────┘
```

### 每一段分别干什么？

| 组件 | 大白话解释 | 作用 |
|---|---|---|
| ChatGPT | 指挥和分析的人 | 理解用户要求、选择工具、阅读结果 |
| REA MCP App | ChatGPT 里的工具入口 | 让 ChatGPT 知道可以调用 REA |
| OpenAI Secure MCP Tunnel | 云端和本机之间的安全通道 | 把 ChatGPT 的 MCP 请求送到本机 |
| tunnel-client | 本机接线员 | 接收隧道请求并转发给本地 REA |
| REA MCP Server | 本机工具服务 | 接收 MCP 请求、执行 REA 工具 |
| REA Provider | 具体的分析引擎 | 执行二进制、反汇编等任务 |
| Ghidra | 反汇编/反编译工具 | 对机器码进行更深入的静态分析 |

## 3. 本机已记录的关键配置

### 3.1 REA Runtime

- REA Runtime 根目录：`%USERPROFILE%\\.dsh\\rea-runtime`
- REA CLI：`%USERPROFILE%\\.dsh\\rea-runtime\\npm-global\\rea.cmd`
- REA CLI 版本：6.10.1
- Node.js：26.12.1
- REA MCP 地址：`http://127.0.0.1:8765/mcp`
- MCP 协议版本：此前服务信息报告为 `2025-03-26`
- 此前 `tools/list` 返回 263 个工具

Node.js 26.12.1 的目录此前记录为：

`%USERPROFILE%\\.dsh\\rea-runtime\\node\\node-v26.12.1-win-x64`

REA CLI 需要 Node.js 至少 26.12.1。启动时应确保 PATH 优先使用这个版本，避免误用电脑上较旧的 Node.js。

### 3.2 OpenAI Secure MCP Tunnel Client

- 客户端版本：0.0.20（Windows x64）
- 程序：`%USERPROFILE%\\.dsh\\rea-runtime\\tunnel-client\\bin\\tunnel-client.exe`
- Profile 名称：`rea-local`
- Profile 文件：`%USERPROFILE%\\.dsh\\rea-runtime\\tunnel-client\\profiles\\rea-local.yaml`
- Profile 中的 MCP 目标：`http://127.0.0.1:8765/mcp`
- 本机健康检查监听地址：`127.0.0.1:8787`
- Tunnel ID：配置记录中有对应 ID；出于公开仓库最小披露原则，这里不重复公开完整标识。

### 3.3 启动脚本

此前创建的脚本：

`%USERPROFILE%\\.dsh\\rea-runtime\\tunnel-client\\run-rea-tunnel.ps1`

脚本设计和此前执行记录：

1. 用 PowerShell 的 `Read-Host -AsSecureString` 提示输入 Runtime API Key，输入内容不会明文显示在屏幕上。
2. 将 Key 临时放入当前启动进程需要的环境变量。
3. 运行 Tunnel Client 的 doctor / preflight 检查。
4. 预检查通过后，在后台启动 tunnel-client。
5. 轮询本机 `/readyz`，等待客户端就绪。
6. 输出 PID、healthz、readyz 和日志路径。
7. 清理父 PowerShell 进程中的临时环境变量和 SecureString 转换产生的内存缓冲。

**安全说明：**隐藏输入只能减少屏幕回显风险，不代表密钥可以随意保存或分享。不要把 Key 写进脚本、Profile、README、GitHub、截图或聊天内容。曾经暴露过的旧 Key 应撤销，不应继续使用。

## 4. Runtime API Key 的权限要求

此前配置记录表明，Tunnel Client 使用的 Runtime API Key 需要具备组织级别的 Tunnels Read 和 Tunnels Use 权限。

重要原则：

- 只在本机安全提示中输入。
- 不要把密钥发给 ChatGPT。
- 不要把密钥放进 YAML 或 PowerShell 文件。
- 如果怀疑泄露，立即撤销旧 Key，检查使用记录，再创建权限尽可能小的新 Key。
- GitHub 仓库里只记录“需要什么权限”，不记录 Key 本身。

## 5. 为什么要让 REA 只监听 127.0.0.1？

REA MCP 服务此前监听：

`127.0.0.1:8765`

这表示服务只接受本机回环接口上的连接。此前配置记录指出 REA HTTP MCP 没有额外 TLS / 用户鉴权层，因此不应把 8765 端口直接暴露到公网。

安全设计应当是：

- REA 只在本机回环地址提供服务。
- 由经过配置的 Tunnel Client 主动建立受控连接。
- 不要在路由器上做 8765 端口映射。
- 不要把 REA 直接绑定到 `0.0.0.0`。
- 不要为了“让 ChatGPT 能访问”而关闭防火墙或公开整个本机服务。

## 6. 实际验证记录：不只是配置文件看起来正确

此前做过以下验证：

### 6.1 Tunnel Client doctor

曾运行：

```powershell
tunnel-client.exe doctor --profile rea-local --profile-dir <profile-directory> --explain
```

此前结果：`RESULT ok`。

其中有两项容易误判：

- MCP endpoint 探测显示 HTTP 406：这是对 Streamable HTTP MCP 端点进行普通 GET/HEAD 探测时可能出现的预期响应，不能单独据此判断 MCP POST 初始化失败。
- OAuth metadata 返回 404：REA 当前没有配置 OAuth 时，这个结果可能是预期的，不应单独当成隧道故障。

真正验证仍应看 MCP 初始化和工具调用。

### 6.2 Tunnel Client 就绪状态

此前一次成功启动返回：

```text
Preflight passed. Starting the tunnel client in the background...
Tunnel client is READY. PID: 5752
Health: http://127.0.0.1:8787/healthz
Ready:  http://127.0.0.1:8787/readyz
```

这证明那一次本机客户端完成了启动，并报告 READY。PID 会随每次启动改变，不能把 5752 当成永久 PID。

### 6.3 日志中观察到的状态

此前日志检查记录包括：

- 本机 health check 返回 HTTP 200。
- REA MCP session initialize 成功。
- OpenAI Tunnel metadata fetch 成功。
- 没有观察到 HTTP 401。

这些是此前那次连接的状态记录，不代表电脑重启后服务永远在线。

### 6.4 最关键：ChatGPT 实际调用 REA 工具

ChatGPT 曾调用：

```text
mcp__rea__list_documents({})
```

REA 返回：

```text
No app is open. Ask the user which app to investigate,
then call open_binary with its local path.
```

这条响应非常关键：它不是网络超时，也不是 Tunnel 无法连接，而是 REA 服务收到请求后，按自己的业务逻辑返回“当前没有打开分析目标”。

因此，**那次测试已经验证 ChatGPT → OpenAI Tunnel → 本机 Tunnel Client → REA MCP Server → REA 工具 的调用路径到达了 REA 应用层。**

但它还没有证明 Ghidra 对大肥鱼 EXE 的完整分析成功。连接成功和分析成功是两件事。

## 7. 正确的启动和检查顺序

如果重启电脑后要恢复连接，建议按以下顺序排查：

### 第 1 步：确认 REA 服务

检查本机 8765 端口是否在监听：

```powershell
Test-NetConnection 127.0.0.1 -Port 8765
```

如果失败，先检查 Windows 计划任务 `REA Local MCP` 和 REA 启动脚本。

### 第 2 步：确认 Node 版本

```powershell
& '<REA_NODE_DIRECTORY>\\node.exe' --version
```

应确认使用的是 26.12.1，而不是 PATH 中的其他旧版本。

### 第 3 步：启动 Tunnel Client

运行：

```powershell
& '<USERPROFILE>\\.dsh\\rea-runtime\\tunnel-client\\run-rea-tunnel.ps1'
```

按提示在本机安全输入 Runtime API Key，不要把 Key 复制到聊天窗口。

### 第 4 步：确认 READY

检查脚本输出是否包含：

- Preflight passed
- Tunnel client is READY
- 本机 healthz / readyz 地址

如未 READY，先看本机 Tunnel Client 日志和错误信息。

### 第 5 步：从 ChatGPT 实际调用一个 REA 工具

例如列出文档或检查当前打开的分析目标。只有拿到 REA 工具的明确响应，才能证明整个调用链到达了 REA。

### 第 6 步：打开真正的分析目标

连接恢复后，再让 REA 打开目标文件，例如：

`G:\\big_blue_fish\\DeepSeek Harness.exe`

之后检查二进制信息、Provider 状态、入口点和反汇编结果。

## 8. 常见故障怎么判断？

| 现象 | 可能问题 | 排查方向 |
|---|---|---|
| 8765 端口不通 | REA MCP 没启动 | 检查计划任务、启动脚本、REA 日志 |
| REA 正常但 Tunnel 不 READY | 隧道凭据、权限或网络问题 | 检查 Key 是否有效、Tunnels Read/Use 权限、doctor 输出 |
| healthz 正常但 ChatGPT 工具不可用 | 本机客户端活着，但云端连接或 App 配置可能有问题 | 查看 tunnel 日志、确认 ChatGPT 中连接的 MCP App |
| ChatGPT 调用后提示 No app is open | 请求已经到达 REA，但没有打开分析目标 | 调用 open_binary 打开目标路径 |
| 工具能调用但 Ghidra 分析失败 | MCP 连接正常，分析 Provider 或环境可能有问题 | 单独检查 Ghidra Provider、Java 和安装路径 |
| 工具返回文件不存在 | 路径或运行账户不一致 | 确认 G 盘挂载、路径拼写、REA 运行用户权限 |

## 9. Tailscale 为什么不是这条主连接？

此前还安装配置了 Tailscale：

- Tailscale 服务运行在 Windows 本机。
- 代理监听 `127.0.0.1:8766`。
- 代理转发到 REA 的 `127.0.0.1:8765`。
- Tailscale Serve 对应设备间访问地址。

但 Tailscale 网络主要是用户自己的设备之间互通。ChatGPT 云端不会因为用户电脑安装了 Tailscale 就自动加入该网络。

因此两条路线要分清：

- **ChatGPT 调用 REA：**OpenAI Secure MCP Tunnel → 本机 tunnel-client → REA。
- **用户自己的其他设备访问 REA：**Tailscale Serve → 本地代理 → REA。

Tailscale 不是 ChatGPT 云端访问本机 REA 的替代通道。

## 10. ChatGPT 端的 REA App

此前记录显示，ChatGPT 中安装的 REA App 标识为：

`app://asdk_app_6b013b58429c8191a2b6339e3a48060f`

它是 ChatGPT 端的工具入口。App 能否正常显示、连接状态如何，仍应以当前 ChatGPT 界面为准。

此前通过该入口成功触发过 REA 工具调用，因此不是仅仅“安装了一个 App”而已。

## 11. 哪些内容不能放进 GitHub？

绝对不要提交：

- Runtime API Key 或任何 API Key
- Tunnel Client 的认证材料、私钥、Token
- 含密钥的环境变量导出
- 未清理的 Tunnel 日志
- 带有 Authorization 请求头的网络抓包
- 完整个人配置目录（可能含用户信息和秘密）

可以提交：

- 脱敏后的配置结构示例
- 不带密钥的启动步骤
- 版本号、端口、组件关系
- 清理过的错误信息和成功验证摘要

## 12. 当前状态总结

| 项目 | 状态 | 说明 |
|---|---|---|
| REA 本机 MCP 服务 | 曾成功启动 | 端口 8765 |
| Tunnel Client | 曾报告 READY | 端口 8787 健康检查 |
| Runtime API Key | 曾成功用于连接 | 不记录密钥 |
| ChatGPT REA App | 已安装 | 曾真实调用工具 |
| ChatGPT 到 REA 的请求链 | **曾验证到达 REA 应用层** | `list_documents` 返回 REA 自身提示 |
| Ghidra Provider | 仍需复核 | 连接通不代表 Provider 正常 |
| 大肥鱼 EXE 深度分析 | 尚未在本文件中证明完成 | 后续单独记录分析证据 |

**最重要的结论：**我们已经有过一次真实的 ChatGPT → REA 工具调用成功记录。后续应优先确保这条连接稳定、重启后可恢复，并把每次连接验证结果记录下来。只有这条基础设施稳定，才能持续让 ChatGPT 帮忙分析本机的大肥鱼程序。

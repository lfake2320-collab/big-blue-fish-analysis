# REA 长期复用与换账号快速配置方案

> 目标：把“ChatGPT 连接本机 REA”做成长期可维护的基础设施。以后换 ChatGPT 账号、重新登录或重新添加 MCP App 时，尽量只做账号侧授权，不重装 REA、不重新搭建本机环境。

## 1. 设计原则：本机层与账号层分离

整个连接分成两层，不能混为一谈。

### A. 本机常驻层（尽量只配置一次）

- REA Runtime / REA MCP Server
- 本机 MCP 地址：`http://127.0.0.1:8765/mcp`
- OpenAI Secure MCP Tunnel Client
- Tunnel Profile（当前名称：`rea-local`）
- 启动、健康检查、日志和恢复脚本
- 大肥鱼目标程序路径与分析环境

这层属于 Windows 电脑。只要电脑、目录和配置没有变化，换 ChatGPT 账号通常不需要重新安装这些组件。

### B. ChatGPT 账号授权层（每个账号可能需要单独做）

- 目标账号所属的 Platform 组织权限
- 该组织是否有 Tunnels Read / Tunnels Use 权限
- Tunnel 是否关联目标 ChatGPT 工作区
- 目标工作区是否允许自定义 MCP Server
- 目标账号是否已添加并启用对应 MCP App

这层属于账号/组织/工作区。换账号时，通常只需要检查和补齐这一层。

## 2. 推荐的最终结构

```text
Windows 电脑（长期运行）
├─ REA MCP Server :8765
├─ Tunnel Client
│  └─ Profile: rea-local
├─ 一套固定的启动/检查/恢复脚本
└─ 本机分析目标：G:\\big_blue_fish\\DeepSeek Harness.exe
       │
       ▼
OpenAI Secure MCP Tunnel（长期连接）
       │
       ├─ ChatGPT 账号 A / 工作区 A（完成授权后可用）
       ├─ ChatGPT 账号 B / 工作区 B（完成授权后可用）
       └─ 未来新增账号 / 工作区（按同一清单授权）
```

说明：一条 Tunnel 能否被多个账号/工作区使用，取决于 OpenAI 当前支持的关联方式、组织权限和工作区策略。不能仅凭复制 Tunnel ID 就认为授权完成；也不能假设所有个人账号或企业工作区都能无条件接入。首次实施多账号方案时必须逐个真实验证。

## 3. 换账号时的快速配置流程

目标是将未来每次换账号的操作压缩为以下步骤：

1. 登录新的 ChatGPT 账号。
2. 确认该账号所属 Platform 组织具有所需的 Tunnel 权限。
3. 确认 Tunnel 已关联目标 ChatGPT 工作区；若未关联，按官方管理界面完成关联。
4. 在目标 ChatGPT 工作区添加自定义 MCP Server，选择 Tunnel 连接方式。
5. 选择或填写现有 Tunnel 标识（只使用非敏感 ID，不复制 Runtime API Key）。
6. 启用 REA App，完成必要的确认。
7. 调用一个只读、低风险的 REA 工具，确认工具请求确实到达本机 REA。
8. 将账号别名、工作区、验证日期、结果记入本地的账号接入清单；不要记录密码、API Key 或 Token。

如果第 2～4 步失败，优先检查账号/组织/工作区权限，不要先重装本机 REA。

## 4. 本机一键检查脚本应做什么

建议把检查功能整合到一个入口脚本，例如：

`check-rea-connection.ps1`

建议检查顺序：

1. 检查 REA MCP 服务进程或服务端口是否存在。
2. 请求本机 REA 健康检查接口（如当前版本提供）。
3. 检查 Tunnel Client 进程是否存在。
4. 请求 Tunnel Client 本地 ready/health 状态接口（以当前实际 Profile 和版本为准）。
5. 检查配置文件是否存在、Profile 名称是否正确。
6. 输出清晰状态：REA 正常 / Tunnel 正常 / 需要账号侧授权 / 本机服务未启动。
7. 不打印环境变量中的密钥，不读取或回显 API Key。

注意：本机脚本不能可靠判断某个 ChatGPT 账号是否已经完成工作区授权。账号侧状态仍需在对应账号中做一次实际工具调用验证。

## 5. 启动与恢复入口

建议最终提供三个明确入口：

- `start-rea-connection.ps1`：启动或确认 REA 与 Tunnel，避免重复启动。
- `check-rea-connection.ps1`：只检查状态，不修改配置。
- `repair-rea-connection.ps1`：仅在用户确认后执行可逆修复；不自动重置 Tunnel、不删除配置、不轮换密钥。

当前已有脚本名称和行为记录在《ChatGPT 连接 REA 完整手册》中。**在读取并核对本机真实脚本之前，不应直接重命名或重写现有脚本。**本方案是目标设计，不代表上述三个入口已经全部实现。

## 6. 配置备份与恢复

建议备份以下非敏感信息：

- Tunnel Profile 的脱敏模板
- REA MCP Server 的启动配置
- 启动、检查和恢复脚本
- 软件版本清单
- 端口和本机路径说明
- 账号接入步骤（只记录账号别名和工作区名称，不记录身份凭据）
- 最近一次成功验证的日期和结果

以下内容不能进入公开 GitHub：

- Runtime API Key、Bearer Token、密码、Cookie
- 含认证头或完整凭据的日志
- 私人聊天记录、个人文件和未脱敏路径
- 可直接用于接管账号或隧道的凭据文件

公开仓库只保存脱敏模板和说明。真正的密钥继续保存在本机安全位置，并由用户在本机安全提示中输入。

## 7. 账号接入登记模板

每次新增账号时，在本机私密笔记中记录：

| 字段 | 记录内容 |
|---|---|
| 账号别名 | 例如：主账号 / 备用账号 |
| ChatGPT 工作区 | 个人工作区或企业工作区名称 |
| Platform 组织 | 组织别名，不记录密钥 |
| Tunnel 关联 | 已确认 / 待确认 |
| MCP App | 已添加 / 未添加 |
| 只读工具测试 | 成功 / 失败 |
| 验证日期 | YYYY-MM-DD |
| 备注 | 错误代码和脱敏后的原因 |

不要把真实邮箱、账号 ID、组织 ID、API Key 或 Token 放进公开仓库。

## 8. 什么时候才需要重新配置本机层？

以下情况才可能需要动本机配置：

- 更换电脑或 Windows 用户环境
- REA Runtime 或 Tunnel Client 安装目录发生变化
- 本机端口变化
- REA MCP Server 启动方式发生变化
- Tunnel 被删除、吊销或需要重新创建
- 凭据被撤销或轮换
- Profile 文件损坏

单纯退出 ChatGPT、换一个 ChatGPT 登录账号，原则上不应触发本机环境重装。

## 9. 实施顺序（先验证，再自动化）

- [ ] 读取并核对当前真实的 `rea-local.yaml`
- [ ] 读取并核对当前真实的 `run-rea-tunnel.ps1`
- [ ] 读取并核对 REA MCP Server 启动脚本
- [ ] 明确本机健康检查端点和进程识别方式
- [ ] 创建只读检查脚本
- [ ] 创建幂等启动脚本（重复执行不会启动多个冲突实例）
- [ ] 编写安全恢复脚本，所有破坏性动作必须二次确认
- [ ] 在第二个 ChatGPT 账号/工作区真实验证授权流程
- [ ] 记录第二账号连接成功的证据
- [ ] 更新 README 和本手册中的实际状态

在完成上述测试之前，不能把“多账号快速切换”标记为已实现。当前状态是：**设计方案已建立；多账号授权流程和一键检查工具待实施、待验证。**

## 10. 官方文档

- Secure MCP Tunnel： https://developers.openai.com/api/docs/guides/secure-mcp-tunnels
- ChatGPT 自定义 MCP Server 连接说明： https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt

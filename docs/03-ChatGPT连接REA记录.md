# 03｜ChatGPT 是怎样连上本机 REA 的？

## 1. 想实现什么？

目标是：REA 在用户自己的 Windows 电脑上运行，ChatGPT 通过安全连接调用 REA 的工具。这样就可以让 ChatGPT 协助检查本机程序，而不是把所有文件都上传到网上。

## 2. 连接结构（大白话版）

```text
ChatGPT
   |
   | 通过 OpenAI Secure MCP Tunnel 发起工具请求
   v
本机 tunnel-client
   |
   | 转发到本机地址
   v
REA MCP 服务（127.0.0.1:8765/mcp）
   |
   v
REA 分析工具 / Ghidra 等 Provider
   |
   v
本机目标程序文件
```

可以把 Tunnel 理解成一条由本机客户端主动建立的安全通道。它避免把 REA 的无鉴权 HTTP 服务直接开到公网。

## 3. Tunnel 配置记录

- Tunnel ID：`tunnel_6b0137ae43088191842c377ceed280f7`
- 客户端版本：0.0.20（Windows x64）
- 客户端路径：`C:\Users\23209\.dsh\rea-runtime\tunnel-client\bin\tunnel-client.exe`
- Profile：`rea-local`
- Profile 文件：`C:\Users\23209\.dsh\rea-runtime\tunnel-client\profiles\rea-local.yaml`
- MCP 目标：`http://127.0.0.1:8765/mcp`
- 本地健康检查端口：8787

启动脚本：`C:\Users\23209\.dsh\rea-runtime\tunnel-client\run-rea-tunnel.ps1`

脚本会通过隐藏输入提示读取 Runtime API Key，并在当前进程环境中使用。**不要把 API Key 写入仓库、聊天记录、截图或普通日志。**

## 4. 如何确认连接真的通了？

最有价值的验证不是“配置文件看起来没问题”，而是实际调用一个 REA 工具。

此前 ChatGPT 调用过：

```text
REA list_documents
```

REA 返回了明确的应用层错误：

```text
No app is open. Ask the user which app to investigate,
then call open_binary with its local path.
```

这条返回说明请求确实到达 REA，并由 REA 自己处理。它不是网络连接失败，而是当时没有打开分析目标。

因此可以确认：在那次测试时，ChatGPT 到 REA 的工具调用链已经打通到应用层。

## 5. 需要分清的几种状态

- Tunnel 客户端显示 READY：表示隧道客户端报告就绪。
- 本机 /healthz 返回 200：表示本地健康检查端点正常。
- REA 工具实际返回业务信息：才是工具链真正可用的强证据。
- 成功打开目标并读取函数、字符串等：才代表具体分析任务已经开始。

不能把“Tunnel READY”直接等同于“已经完成程序分析”。

## 6. Tailscale 和 OpenAI Tunnel 的区别

此前还配置过 Tailscale：

- Tailscale IP：`100.122.78.113`
- MagicDNS：`l.tail3aed70.ts.net`
- Serve 使用本机 8766 代理端口。

Tailscale 主要方便用户自己的设备互相访问；ChatGPT 云端不会自动加入用户的 Tailscale 网络。ChatGPT 的连接路线是 OpenAI Secure MCP Tunnel，不是 Tailscale Serve。

## 7. 密钥安全

之前曾有一把 API Key 意外出现在聊天内容中。此前已提醒用户撤销泄露的旧 Key 并更换。不要在本仓库记录新 Key，也不要在聊天里粘贴它。

如果怀疑 Key 泄露：到对应平台撤销旧 Key，检查使用记录，再创建权限尽可能小的新 Key。

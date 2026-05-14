---
name: cc-connect-debug
description: cc-connect 远程调试指南 - 用于飞书/钉钉等 IM 平台连接问题定位
license: MIT
compatibility: opencode
---

# cc-connect 远程调试指南

本 skill 用于 cc-connect 远程使用场景（飞书、钉钉等 IM 平台）的问题定位。

## 一、调试流程概述

### 1.1 消息流转路径

```
用户消息 → IM 平台 → cc-connect → OpenCode → 响应
         ↓          ↓            ↓           ↓
    [平台日志]  [会话JSON]   [数据库part]  [stdout]
```

### 1.2 标准调试步骤

按以下顺序排查：

| 步骤 | 检查点 | 方法 |
|------|--------|------|
| 1 | 进程状态 | `Get-Process -Name cc-connect, opencode` |
| 2 | 消息到达 cc-connect | 会话 JSON 文件的 `history` 数组 |
| 3 | 消息到达 OpenCode | 数据库 `part` 表最新 `text` 类型事件 |
| 4 | OpenCode 开始处理 | 对应的 `step_start` 事件 |
| 5 | OpenCode 处理完成 | `step_finish` 事件，`reason=stop` |
| 6 | 是否有错误 | `error` 类型事件 |

### 1.3 启动命令（推荐 daemon 模式）

```powershell
# 推荐：daemon 模式（系统级服务管理，稳定性和性能最优）
.\cc-connect.exe daemon install

# daemon 安装后会自动启动，日志位置：
# C:\Users\{用户名}\.cc-connect\logs\cc-connect.log
# 最大 10 MB，自动轮转

# 查看 daemon 状态
.\cc-connect.exe daemon status

# 实时查看日志（不阻塞当前终端）
.\cc-connect.exe daemon logs --lines 50

# 其他 daemon 命令
.\cc-connect.exe daemon restart   # 重启
.\cc-connect.exe daemon stop      # 停止
.\cc-connect.exe daemon uninstall # 移除
```

**daemon 优势**：
- 系统级服务管理（Windows 使用 schtasks）
- cc-connect 直接写入日志文件，无进程间通信开销
- 内置日志轮转（最大 10 MB）
- 系统重启后自动恢复
- 不依赖外部进程（pm2 是 Node.js 进程，占用额外资源）

**注意**：Windows 上会显示一个 PowerShell 窗口滚动输出日志。请勿关闭此窗口，否则 cc-connect 服务会停止。

### 1.3.1 进程管理

```powershell
# 查看运行状态
Get-Process -Name cc-connect -ErrorAction SilentlyContinue | Select-Object Id, ProcessName, StartTime

# 停止所有进程
Stop-Process -Name cc-connect -Force

# 重启 daemon 服务
.\cc-connect.exe daemon restart
```

### 1.4 调试信息位置

| 信息类型 | 位置 | 说明 |
|---------|------|------|
| 配置文件 | `--config` 参数或 `./config.toml` 或 `~/.cc-connect/config.toml` | 优先级递减 |
| cc-connect 日志 | `%USERPROFILE%\.cc-connect\logs\cc-connect.log` | daemon 模式下生成（推荐） |
| cc-connect 会话 | `%USERPROFILE%\.cc-connect\sessions\*.json` | 用户对话历史、AgentSessionID |

**获取 cc-connect 日志（不阻塞当前终端）**：

```powershell
# 查看最近日志
Get-Content "$env:USERPROFILE\.cc-connect\logs\cc-connect.log" -Tail 50

# 实时查看日志（daemon 命令）
.\cc-connect.exe daemon logs --lines 50

# 查看日志文件元数据
Get-Item "$env:USERPROFILE\.cc-connect\logs\cc-connect.log" | Select-Object Length, LastWriteTime
```

## 二、日志位置和查询命令

### 2.1 日志路径

| 类型 | 路径 | 内容 |
|------|------|------|
| cc-connect 日志 | `%USERPROFILE%\.cc-connect\logs\cc-connect.log` | 运行日志（daemon 模式，推荐） |
| cc-connect 会话 | `%USERPROFILE%\.cc-connect\sessions\*.json` | 用户对话历史、AgentSessionID |
| OpenCode 数据库 | `%USERPROFILE%\.local\share\opencode\opencode.db` | 所有事件记录（最关键） |
| OpenCode 日志 | `%USERPROFILE%\.local\share\opencode\log\*.log` | 运行日志 |

### 2.2 常用查询命令

```powershell
# 查看最新 OpenCode 事件（最常用）
sqlite3 $env:USERPROFILE\.local\share\opencode\opencode.db "SELECT id, json_extract(data, '$.type'), substr(data, 1, 300) FROM part ORDER BY time_created DESC LIMIT 10"

# 查看特定 session 的完整活动
sqlite3 $env:USERPROFILE\.local\share\opencode\opencode.db "SELECT time_created, json_extract(data, '$.type'), substr(data, 1, 200) FROM part WHERE session_id = 'ses_xxx' ORDER BY time_created"

# 查看最近的 step_finish 状态
sqlite3 $env:USERPROFILE\.local\share\opencode\opencode.db "SELECT id, json_extract(data, '$.reason') FROM part WHERE json_extract(data, '$.type') = 'step_finish' ORDER BY time_created DESC LIMIT 5"

# 查看是否有错误
sqlite3 $env:USERPROFILE\.local\share\opencode\opencode.db "SELECT time_created, substr(data, 1, 500) FROM part WHERE json_extract(data, '$.type') = 'error' ORDER BY time_created DESC LIMIT 3"

# 查看 cc-connect 会话信息
Get-Content $env:USERPROFILE\.cc-connect\sessions\project_*.json | ConvertFrom-Json
```

### 2.3 快速检查脚本

```powershell
# 一键检查脚本
Get-Process -Name cc-connect -ErrorAction SilentlyContinue
sqlite3 $env:USERPROFILE\.local\share\opencode\opencode.db "SELECT json_extract(data, '$.type'), substr(data, 1, 100) FROM part ORDER BY time_created DESC LIMIT 5"
```

## 三、OpenCode 事件格式

### 3.1 stdout 事件类型

cc-connect 解析的是 OpenCode stdout 输出，**使用下划线格式**：

| 顶层 type | 说明 | 关键字段位置 |
|-----------|------|-------------|
| `text` | 文本响应 | `part.text` |
| `reasoning` | 思考过程 | `part.text` |
| `tool_use` | 工具调用 | `part.tool`, `part.state` |
| `step_start` | 开始处理 | 顶层 `sessionID` |
| `step_finish` | 处理完成 | `part.reason` (stop/tool-calls) |
| `error` | 错误事件 | 顶层 `error.message` |

**注意**：数据库存储格式使用连字符（`step-start`），但 stdout 输出使用下划线（`step_start`）。cc-connect 必须解析 stdout 格式。

### 3.2 关键字段位置示例

```json
{"type": "step_start", "sessionID": "ses_xxx", "part": {...}}
{"type": "step_finish", "part": {"reason": "stop"}}
{"type": "text", "part": {"text": "响应内容"}}
{"type": "error", "error": {"data": {"message": "错误信息"}}}
```

## 四、常见问题排查

### 4.1 用户消息无响应

**排查步骤**：

1. 检查进程：`Get-Process -Name cc-connect`
2. 查询数据库最新事件，确认消息是否到达 OpenCode
3. 查找 `error` 类型事件，确认是否有错误
4. 手动测试 OpenCode：
   ```powershell
   opencode run --format json --dir work_dir --model model_id "test"
   ```

**常见原因**：
- API 认证失败（查看 error 事件的 message）
- 事件类型未处理（检查 handleEvent switch）
- 进程未正常启动

### 4.2 响应中断或不完整

**排查步骤**：

1. 查询 `step_finish` 事件的 `reason` 值
2. 如果 `reason=tool-calls`，说明还在处理中，继续等待
3. 如果没有 `step_finish`，检查是否有 `error` 事件

**常见原因**：
- `reason=tool-calls`：正常的多轮工具调用，等待完成
- 缺少 `step_finish reason=stop`：处理未正常结束
- 未实现 error 事件处理：错误被静默忽略

### 4.3 多条消息阻塞

**排查步骤**：

1. 检查 cc-connect 是否在处理上一条消息
2. 查询最新 `step_finish` 是否为 `reason=stop`
3. 如果一直为 `tool-calls`，可能工具调用卡住

**常见原因**：
- OpenCode 工具调用未完成
- 会话锁未释放（需等待上一条消息完成）

## 五、代码调试要点

### 5.1 事件处理完整性

确保 `handleEvent` 处理所有事件类型：
- `text`, `reasoning`, `tool_use`, `step_start`, `step_finish`, `error`

### 5.2 字段获取位置

| 字段 | 获取位置 |
|------|---------|
| `sessionID` | `raw["sessionID"]`（顶层） |
| `text` | `raw["part"]["text"]` |
| `reason` | `raw["part"]["reason"]` |
| `error.message` | `raw["error"]["data"]["message"]` |

### 5.3 会话结束判断

只有 `step_finish` 且 `part.reason="stop"` 才表示处理完成，发送 `EventResult`。`reason="tool-calls"` 表示继续处理。
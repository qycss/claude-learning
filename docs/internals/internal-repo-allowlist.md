# Internal Repo Allowlist

> **Source**: `src/utils/commitAttribution.ts`

## Overview

`INTERNAL_MODEL_REPOS` is a whitelist of ~30 Anthropic private repositories where Undercover Mode is **disabled** (attribution is shown normally). This list is the boundary between "internal" and "public" for Anthropic employees.

## The Allowlist

Both SSH (`git@github.com:anthropics/`) and HTTPS (`https://github.com/anthropics/`) formats:

| Repository | Probable Purpose |
|-----------|-----------------|
| `claude-cli-internal` | Claude Code internal development |
| `anthropic` | Main Anthropic monorepo |
| `apps` | Internal applications |
| `casino` | Unknown internal project |
| `dbt` | Data transformation (analytics) |
| `dotfiles` | Developer environment configs |
| `terraform-config` | Infrastructure as code |
| `hex-export` | Unknown internal tool |
| `feedback-v2` | User feedback system |
| `labs` | Research experiments |
| `argo-rollouts` | Kubernetes deployment |
| `starling-configs` | Configuration management |
| `ts-tools` | TypeScript tooling |
| `ts-capsules` | TypeScript modules |
| `feldspar-testing` | Testing infrastructure |
| `trellis` | Unknown internal platform |
| `claude-for-hiring` | Hiring/interview tool |
| `forge-web` | Internal web platform |
| `infra-manifests` | Infrastructure manifests |
| `mycro_manifests` | Microservice manifests |
| `mycro_configs` | Microservice configs |
| `mobile-apps` | Mobile applications |

## Critical Design Note

From the source comments:

> "`anthropics` and `anthropic-experimental` orgs contain **PUBLIC** repos... Undercover mode must stay ON in those"

This means:
- Having a repo under `anthropics/` GitHub org does NOT make it "internal"
- Only repos explicitly listed in `INTERNAL_MODEL_REPOS` disable undercover mode
- Public repos like `anthropic-cookbook`, `courses`, `claude-code` itself remain in undercover territory

## Security Implications

This allowlist reveals:
1. The internal project names and their probable purposes
2. Anthropic's infrastructure stack (Terraform, Argo, Kubernetes)
3. The microservice architecture (`mycro_*` repos)
4. The distinction between public and private repositories within the same GitHub org

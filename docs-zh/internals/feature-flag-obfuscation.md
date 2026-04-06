# Feature Flag 混淆

## 概述

Claude Code 使用**故意不透明的** feature flag 名称，以防止外部观察者了解正在测试或推出的功能。命名规范遵循 `tengu_<word1>_<word2>` 模式，其中单词与功能没有任何语义关联。

## 命名模式

```
tengu_<word1>_<word2>
```

- `tengu` — 产品代号（天狗）
- `<word1>` 和 `<word2>` — 随机的、无关的英文单词

## 已知示例

| Flag 名称 | 实际用途 |
|-----------|----------|
| `tengu_penguins_off` | Fast/Penguin Mode 紧急开关 |
| `tengu_frond_boric` | 分析数据通道紧急开关（按通道控制：datadog、firstParty） |
| `tengu_auto_mode_config` | YOLO 分类器配置（enabled、twoStageClassifier 等） |
| `tengu_event_sampling_config` | 遥测事件采样率 |
| `tengu_log_datadog_events` | Datadog 事件日志门控 |
| `tengu_ant_model_override` | 内部模型覆盖配置 |

## 为什么要混淆？

Feature flag 通常在以下场景中可见：
1. **网络流量** — GrowthBook API 调用包含 flag 名称
2. **源代码** — 即使在编译/打包后的代码中，字符串字面量仍然保留
3. **错误信息** — flag 名称可能出现在日志中
4. **客户端存储** — 缓存的 feature 值会写入磁盘

混淆命名可以防止：
- 竞争对手了解 A/B 测试策略
- 用户手动覆盖实验性功能
- 安全研究人员推断尚未发布的能力

## GrowthBook 集成

Flag 通过 `remoteEval` 模式进行求值——服务器根据用户属性预先计算所有 feature 的结果并返回。Flag 名称是响应中的键。

覆盖优先级：
```
环境变量 (CLAUDE_INTERNAL_FC_OVERRIDES)      <- 最高
  > 配置覆盖 (仅限 ant 用户 /config Gates)
    > 远程求值结果
      > 磁盘缓存                              <- 最低
```

## 与直接命名的对比

部分 flag 在不需要混淆时使用可识别的名称：
- `tengu_auto_mode_config` — 半可识别（auto mode 是公开功能）
- `tengu_penguins_off` — 半混淆（penguins = 内部代号）
- `tengu_frond_boric` — 完全混淆（与分析功能毫无关联）

# 回合系统

## 概述

单局由多个**回合（Round）**串联。每个回合是一段限时生存战斗，时间结束即强制结束，不依赖清场。

## 回合配置

每回合由数据驱动，核心字段示例：

| 字段 | 类型 | 说明 |
|------|------|------|
| `round_id` | int | 回合序号 |
| `duration_sec` | 30 \| 60 \| 90 | 本回合时长（秒） |
| `spawn_table` | SpawnEntry[] | 怪物生成表 |
| `difficulty_multiplier` | float | 怪物血量/伤害倍率（可选） |
| `token_reward_base` | int | 回合结束基础代币 |
| `is_boss_round` | bool | 是否为 Boss 回合 |

### 刷怪表 SpawnEntry

```json
{
  "enemy_id": "slime_small",
  "weight": 40,
  "spawn_interval": 2.0,
  "max_concurrent": 15,
  "spawn_pattern": "ring" 
}
```

- **weight**：在本回合刷怪池中的权重。
- **spawn_interval**：该类型怪物的生成间隔（秒）。
- **max_concurrent**：场上该类型怪物数量上限。
- **spawn_pattern**：生成位置策略（如 `ring` 屏幕外环形、`random_edge` 随机边缘）。

### 回合间差异

| 维度 | 早期回合 | 后期回合 |
|------|----------|----------|
| 怪物种类 | 1~2 种 | 3~5 种混合 |
| 同屏数量 | 较少 | 较多 |
| 精英/Boss | 无 | 按配置出现 |
| 时长 | 可偏短（30s） | 可偏长（60s/90s） |

## 回合状态机

```
ROUND_IDLE
  → ROUND_COUNTDOWN（可选 3-2-1）
  → ROUND_ACTIVE（计时 + 刷怪 + 战斗）
  → ROUND_ENDING（停刷怪、结算掉落/代币）
  → SHOP（移交商店系统）
```

### ROUND_ACTIVE 规则

- 全局倒计时从 `duration_sec` 递减至 0。
- 刷怪系统按 `spawn_table` 持续生成，受 `max_concurrent` 与全场怪物上限约束。
- 玩家死亡 → 本局游戏结束（非单回合重开）。
- 时间到 0 → 进入 `ROUND_ENDING`，**不要求**击杀剩余怪物。

## 时长配置

时长在回合配置中逐回合指定，也可由**难度预设**批量映射：

| 预设 | 典型映射 |
|------|----------|
| 快节奏 | 前 3 回合 30s，之后 60s |
| 标准 | 全程 60s |
| 耐力 | 后半段 90s |

全局默认值建议：`60s`；可在 `project settings` 或关卡表中覆盖。

## 回合结束奖励

```
代币 = token_reward_base
     + floor(存活秒数 × 存活系数)    // 可选
     + 击杀奖励总和                  // 按怪物类型表
```

奖励在 `ROUND_ENDING` 结算完成后写入玩家代币，再打开商店。

## 与商店的衔接

- 回合结束 → 暂停战斗场景（或切换至商店场景）。
- 玩家点击「继续」且商店关闭后，加载下一回合配置并进入 `ROUND_COUNTDOWN`。

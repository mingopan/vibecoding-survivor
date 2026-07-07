# 武器攻击执行序列

## 概述

武器攻击不是单一「发射子弹」函数，而是一条可配置、可扩展的**执行序列（Execution Sequence）**。核心玩法在于组合不同步骤，实现多样化的子弹、发射方式与命中效果。

## 设计目标

- **数据驱动**：新武器 / 扩展主要通过配置序列或补丁，少写硬编码。
- **可组合**：扩展可插入、替换、修改序列中的步骤。
- **确定性**：同帧内多武器并行，各武器序列独立实例，便于回放与联机扩展。
- **可读**：策划能看懂步骤列表，程序员实现有限步骤类型库。

## 序列结构

一次「开火」触发一个 **Fire Context**，按顺序执行 **Step** 列表；任一步可 `abort` 终止后续步骤。

```
FireContext
├── weapon_instance
├── owner (player)
├── target_hint (optional)
├── rng_seed
└── steps: Step[]
```

### Step 通用字段

| 字段 | 说明 |
|------|------|
| `step_id` | 步骤类型，如 `spawn_projectile` |
| `params` | 类型相关参数 |
| `conditions` | 可选，满足才执行 |
| `on_complete` | 可选，跳转或触发子序列 |

## 标准步骤类型库

### 1. 瞄准与准备

| step_id | 作用 |
|---------|------|
| `acquire_target` | 最近敌人 / 鼠标方向 / 移动方向 |
| `apply_spread` | 根据 params 计算多个发射角 |
| `charge` | 蓄力，修改后续伤害/弹体大小 |
| `consume_ammo` | 扣弹药或冷却（无弹药武器可跳过） |

### 2. 发射体生成

| step_id | 作用 |
|---------|------|
| `spawn_projectile` | 生成直线弹体 |
| `spawn_arc` | 抛物线 / 弧形 |
| `spawn_beam` | 即时射线，持续 tick |
| `spawn_orbital` | 环绕玩家旋转体 |
| `spawn_zone` | 地面持续区域（毒圈、减速场） |
| `spawn_summon` | 召唤物（自主寻敌） |

**params 示例**（`spawn_projectile`）：

```json
{
  "count": 3,
  "speed": 400,
  "lifetime": 2.0,
  "pierce": 1,
  "visual": "polygon_triangle",
  "size": 8
}
```

### 3. 发射模式（由扩展修改 params 或插入步骤）

| 模式 | 实现方式 |
|------|----------|
| 单发 | `count: 1` |
| 扇形散射 | `apply_spread` + `spawn_projectile` |
| 连发 | 重复 `spawn_projectile` 或 `burst` 包装步骤 |
| 环绕 | `spawn_orbital` |
| 反弹 | projectile 行为 + 墙壁反弹 |
| 追踪 | projectile 行为 `homing` |

### 4. 命中与效果

| step_id | 作用 |
|---------|------|
| `on_hit` | 子序列入口：弹体命中时触发 |
| `deal_damage` | 伤害结算 |
| `apply_status` | 燃烧、减速、感电等 |
| `spawn_sub_projectile` | 分裂弹、连锁闪电 |
| `knockback` | 击退 |
| `lifesteal` | 吸血 |
| `area_explosion` | 范围二次伤害 |

**on_hit 子序列示例**：

```
on_hit
  → deal_damage
  → apply_status (burn, 3s)
  → chance(0.3) → area_explosion
```

### 5. 收尾

| step_id | 作用 |
|---------|------|
| `play_vfx` | 枪口焰、辉光闪 |
| `play_sfx` | 音效 |
| `start_cooldown` | 武器 CD |
| `trigger_passive` | 通知其他扩展「本次开火已发生」 |

## 执行规则

### 顺序与并行

- 序列内步骤默认**串行**。
- 标记 `parallel: true` 的步骤可与下一并行组合并同时执行（如三发子弹同时 `spawn_projectile`）。
- `on_hit` / `on_expire` 子序列在事件发生时**异步**触发，不阻塞主序列（除非 `wait: true`）。

### 扩展如何修改序列

| 扩展类型 | 修改方式 |
|----------|----------|
| 数值 | `patch_params`：`spawn_projectile.count += 2` |
| 插入 | `insert_step_after("spawn_projectile", "apply_status")` |
| 替换 | `replace_step("spawn_projectile", "spawn_beam")` |
| 条件分支 | 增加 `conditions: { ext: "arc_mode" }` 的步骤 |

扩展在**装备时**编译进武器的运行时序列（缓存），卸下时还原。

### 冷却与攻速

- 一次完整序列执行完毕 → `start_cooldown`。
- 攻速影响：`cooldown = base_cooldown / fire_rate`。
- 部分步骤可 `scale_with_attack_speed: true`（连发间隔缩短）。

### 冲突与优先级

1. 武器模板默认序列  
2. 武器品质修正（数值 patch）  
3. 扩展按槽位顺序应用 patch（先左后右，或按 `priority` 字段）  
4. 同一步骤多次 patch → 累加或取 max，由 `merge_rule` 声明

## 完整示例：脉冲步枪

```yaml
weapon_type: pulse_rifle
fire_sequence:
  - step_id: acquire_target
    params: { mode: nearest_enemy }
  - step_id: apply_spread
    params: { angle: 12, count: 1 }
  - step_id: spawn_projectile
    params: { speed: 500, pierce: 0, visual: polygon_bar }
  - step_id: on_hit
    sub_sequence:
      - step_id: deal_damage
      - step_id: apply_status
        params: { status: shock, duration: 0.5 }
  - step_id: play_vfx
    params: { preset: muzzle_glow_blue }
  - step_id: start_cooldown
    params: { base: 0.4 }
```

装备扩展「双发」后编译结果：`apply_spread.count` 变为 2，或插入第二个 `spawn_projectile`。

## 运行时 API（概念）

```gdscript
class_name WeaponExecutionRunner

func compile_sequence(weapon: WeaponInstance) -> CompiledSequence
func fire(weapon: WeaponInstance, context: FireContext) -> void
func _run_step(step: Step, ctx: FireContext) -> StepResult
```

弹体命中时调用 `runner.trigger_event("on_hit", hit_context)` 执行子序列。

## 调试

- 开发模式：开火时在屏幕打印当前步骤链（折叠显示）。
- 录制 `rng_seed` + 输入 → 便于复现穿透/暴击问题。

## 与视觉的衔接

- 每步 `play_vfx` 使用 [visual-style.md](visual-style.md) 中的多边形 + 辉光预设。
- 弹体 `visual` 字段映射到 mesh 与 shader 参数（颜色、尾迹长度）。

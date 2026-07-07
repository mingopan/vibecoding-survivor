# GDScript 编码风格

> **vibecoding_survivor** 项目完整编码规范。所有 GDScript、场景脚本与新增代码必须遵守本文档。  
> 禁区与反模式见 [bad-patterns.md](bad-patterns.md)。

---

## 1. 适用范围

| 范围 | 路径 |
|------|------|
| 游戏场景 | `scenes/`（战斗、商店、菜单等流程场景） |
| 预制体 | `prefabs/`（实体、UI 控件、特效等可复用场景） |
| 场景伴随脚本 | 与对应 `.tscn` **同目录**（见 §8） |
| 系统逻辑脚本 | `scripts/systems/`、`scripts/autoload/` |
| 数据资源 | `resources/`（`.tres` 配置与数据） |
| 原始素材 | `assets/`（贴图、音频、字体、shader 等） |
| 自动加载 | `scripts/autoload/` |

引擎版本：**Godot 4.7**。不使用已废弃的 Godot 3.x API。

---

## 2. 命名

| 类型 | 风格 | 示例 |
|------|------|------|
| 类名 `class_name` | `PascalCase` | `WeaponRunner`、`RoundManager` |
| 函数 / 方法 | `snake_case` | `try_purchase()`、`compile_sequence()` |
| 成员变量 | `snake_case` | `current_tokens`、`fire_rate` |
| 常量 | `UPPER_SNAKE_CASE` | `MAX_WEAPON_SLOTS`、`QUALITY_MYTHIC` |
| 信号 | `snake_case`，事件用过去式 | `round_ended`、`weapon_fired`、`item_purchased` |
| 私有成员 | `_` 前缀 | `_spawn_timer`、`_cached_sequence` |
| 脚本文件 | `snake_case.gd` | `weapon_runner.gd` |
| 场景文件 | `snake_case.tscn` | `player.tscn`、`shop_panel.tscn` |
| Resource 文件 | `snake_case.tres` | `pulse_rifle.tres`、`round_03.tres` |
| 枚举 | `PascalCase` 类型名 + `UPPER_SNAKE` 成员 | `Quality.EPIC` |

### 命名语义

- **布尔**变量用 `is_` / `has_` / `can_` 前缀：`is_round_active`、`can_merge`。
- **回调**以 `_on_` 开头：`_on_round_timer_timeout`。
- **工厂 / 构建**用 `create_` / `build_` / `compile_`：`compile_sequence()`。
- 标识符一律**英文**；面向玩家的文案走本地化，不写死在逻辑深处。

---

## 3. 类型注解

- 函数**参数**与**返回值**必须声明类型；无返回写 `-> void`。
- 成员变量**尽量**声明类型；`@export` **必须**声明类型。
- 使用 Godot 4 泛型集合：`Array[String]`、`Array[Enemy]`、`Dictionary[String, int]`。
- 可为空的节点引用显式标注：`var _target: Enemy = null`。

```gdscript
# ✅
func apply_damage(amount: float, source: Node2D) -> bool:
	if amount <= 0.0:
		return false
	return true

# ❌
func apply_damage(amount, source):
	if not amount > 0: return false
```

### 空值与有效性

```gdscript
# 对象 / 节点
if enemy == null or not is_instance_valid(enemy):
	return

# 数组、字符串
if extensions.is_empty():
	return
```

- 对象判空用 `== null`，不用 `if not enemy` 混用风格。
- 数组是否为空用 `is_empty()`，不用 `size() == 0`。

---

## 4. 文件内成员顺序

每个 `.gd` 文件按以下顺序组织（没有的段落跳过）：

| 顺序 | 成员 |
|------|------|
| 1 | `extends` / `class_name`（`extends` 必须在首行） |
| 2 | 常量 `const`、枚举 `enum` |
| 3 | `@onready` 变量 |
| 4 | `@export` 变量 |
| 5 | 公开变量 |
| 6 | 私有变量（`_` 前缀） |
| 7 | 信号 `signal` |
| 8 | 生命周期（`_init`、`_ready`、`_process`…） |
| 9 | 公开方法 |
| 10 | 私有方法 |
| 11 | 信号绑定方法（`_on_*` 回调） |
| 12 | 静态方法 `static func` |

```gdscript
extends Node2D
class_name WeaponRunner

## 类文档（可选，仅复杂系统类）

# --- 常量 ---
const MAX_STEPS := 32

enum StepPhase { AIM, SPAWN, ON_HIT, COOLDOWN }

# --- @onready ---
@onready var _muzzle: Marker2D = %Muzzle

# --- @export ---
@export var debug_log: bool = false

# --- 公开变量 ---
var is_compiling: bool = false

# --- 私有变量 ---
var _steps: Array[Dictionary] = []

# --- 信号 ---
signal sequence_completed(weapon: WeaponInstance)

# --- 生命周期 ---
func _ready() -> void:
	pass

func _physics_process(delta: float) -> void:
	pass

func _exit_tree() -> void:
	pass

# --- 公开方法 ---
func compile(weapon: WeaponInstance) -> CompiledSequence:
	pass

# --- 私有方法 ---
func _run_step(step: Dictionary) -> void:
	pass

# --- 信号绑定方法 ---
func _on_hitbox_area_entered(area: Area2D) -> void:
	pass

# --- 静态方法 ---
static func create_default() -> WeaponRunner:
	return WeaponRunner.new()
```

- `class_name` 仅全局可复用类需要。
- 信号绑定方法统一 `_on_` 前缀，集中在私有方法之后。
- 不用 `pass` 占位空函数，除非编辑器强制要求。

---

## 5. 格式化

与 `.editorconfig` 一致：

| 项 | 规则 |
|----|------|
| 缩进 | **Tab**（展示宽度 4，由编辑器处理） |
| 编码 | UTF-8 |
| 行宽 | 建议 ≤ 100 字符，过长时按语义换行 |
| 字符串 | 默认双引号 `"`；无插值时用字面量 |
| 尾随逗号 | 多行字典 / 数组最后一项保留逗号（便于 diff） |

### 控制流

```gdscript
# ✅ 早返回，减少嵌套
func try_purchase(item_id: String) -> bool:
	if _tokens < _get_price(item_id):
		return false
	_tokens -= _get_price(item_id)
	return true

# ❌ 一行塞多种逻辑
func try_purchase(item_id):
	if not _tokens >= _get_price(item_id): return false
```

- 禁止无括号的单行 `if` 叠在同行。
- 三元表达式少用；可读性优先时用完整 `if/else`。

---

## 6. 节点与场景访问

```gdscript
# ✅ 缓存引用
@onready var _hud: Control = %HUD
@onready var _spawn_point: Marker2D = $SpawnPoint

# ❌ 热路径或深层路径查找
func _process(_delta: float) -> void:
	var label = get_node("../../UI/Label")
```

- 优先 `%UniqueName`（场景内勾选「唯一名称」）。
- **禁止**在 `_process` / `_physics_process` 中调用 `get_node` / `find_child`。
- `_init()` 中不访问子节点；在 `_ready()` 或 `@onready` 中访问。

---

## 7. 信号

```gdscript
func _ready() -> void:
	EventBus.round_ended.connect(_on_round_ended)

func _exit_tree() -> void:
	if EventBus.round_ended.is_connected(_on_round_ended):
		EventBus.round_ended.disconnect(_on_round_ended)
```

- 连接：`signal.connect(callable)`。
- 在 `_exit_tree()` 中断开**本节点发起**的连接，避免泄漏。
- 跨模块通信用 **EventBus**（Autoload）或明确注入的依赖，禁止 `get_parent().get_parent()`。

---

## 8. 目录与职责

```
scenes/                 # 仅「游戏场景」：流程级场景
  combat/               # 战斗场景
  shop/                 # 商店场景
  menu/                 # 菜单场景

prefabs/                # 预制体：可复用实体与控件
  combat/               # 玩家、敌人、弹体…
  ui/                   # UI 控件、面板组件
  vfx/                  # 特效预制

scripts/                # 无配套 .tscn 的纯逻辑脚本
  autoload/             # EventBus、GameState 等单例
  systems/              # RoundManager、ShopService、WeaponRunner…
  resources/            # Resource 子类定义（.gd，非 .tres）

assets/                 # 原始素材（引擎导入前的源文件）
  sprites/
  audio/
  fonts/
  shaders/

resources/              # .tres 配置与数据资源
  weapons/
  rounds/
  extensions/
  quality/

docs/                   # 设计 & 技术文档（不参与运行时）
```

### 脚本与场景同目录

**游戏场景、预制体、UI 控件的根节点脚本**与对应 `.tscn` 放在**同一目录**，不放入 `scripts/`：

```
prefabs/combat/
  player.tscn
  player.gd          # ✅ 与 player.tscn 同目录

scenes/shop/
  shop.tscn
  shop.gd            # ✅ 与 shop.tscn 同目录

scripts/systems/
  shop_service.gd    # ✅ 无 tscn 的纯逻辑
```

| 规则 | 说明 |
|------|------|
| 场景 vs 预制体 | `scenes/` 只放流程场景；可复用实体放 `prefabs/` |
| 脚本归属 | 有 `.tscn` 的根脚本跟 tscn 走；无 tscn 的逻辑放 `scripts/` |
| 素材 vs 配置 | 原始素材进 `assets/`；`.tres` 进 `resources/` 对应子目录 |
| 数据与逻辑分离 | 数值、刷怪表、序列步骤用 `resources/*.tres`，不写死在脚本里 |
| UI 不写公式 | 伤害、价格、合成结果由 `scripts/systems/` 计算，UI 只调用并显示 |
| 单预制体单实体 | 一个 prefab `.tscn` 对应一个可复用实体 |
| 根节点名与文件名一致 | `player.tscn` → 根节点 `Player` |

---

## 9. 本项目专用约定

### 9.1 配置驱动

回合时长、武器槽数量、扩展槽数量、品质系数等来自 `resources/` 下的 Resource，脚本只读配置：

```gdscript
@export var config: RoundConfig

func get_duration_sec() -> int:
	return config.duration_sec
```

魔法数字放入 `const` 或 `@export`，并注明含义。

### 9.2 武器执行序列

- 步骤类型用 `StringName` 或枚举，集中注册在 `WeaponExecutionRunner`。
- 扩展对序列的修改在**装备时编译**为 `CompiledSequence`，战斗时只读缓存。
- 命中子序列通过 `trigger_event("on_hit", ctx)` 触发，不在弹体脚本里硬编码全套效果。

### 9.3 性能（幸存者同屏）

- 弹体、敌人、伤害数字使用**对象池**，禁止每帧 `instantiate` + `queue_free` 刷屏。
- 寻敌、商店刷新权重等用 `Timer`，间隔 ≥ 0.1s，不在每帧 `_process` 全量遍历。
- 大量实体禁用逐节点 `Area2D` 重叠检测时，考虑空间哈希或 Godot 内置 broadphase 合理分组。

### 9.4 品质与视觉

- 品质颜色、辉光强度从 `resources/quality/` 或等价 `.tres` 读取，与 [item-quality.md](../design/features/item-quality.md) 一致。
- VFX 通过 `play_vfx` 步骤或统一 `VfxSpawner` 生成，不散落各脚本。

---

## 10. 资源路径

- 一律使用 `res://` 绝对路径。
- 禁止 `../` 相对路径、`preload` 非项目内资源。
- 动态加载频繁的资源在启动或关卡加载时 `preload` / 缓存。

```gdscript
const BULLET_SCENE: PackedScene = preload("res://prefabs/combat/bullet.tscn")
```

---

## 11. 注释

- **好的注释**解释「为什么」，不复述「做了什么」。
- 公共 API（`systems/`、`autoload/`）的复杂函数用 `##` 文档注释。
- TODO 格式：`# TODO(模块): 简述`，带责任人或工单号更佳。
- 禁止大段注释掉的废代码；删除用 git 追溯。

---

## 12. 函数体量与拆分

- 单函数建议 **≤ 30 行**；超过则提取私有方法或独立类。
- 单文件建议 **≤ 300 行**；超过则按职责拆文件。
- `_process` / `_physics_process` 内只保留每帧必须逻辑，其余拆到系统或 Timer。

---

## 13. 版本控制

- 不提交 `.godot/`、`.import/`、本地 IDE 配置。
- `docs/` 已有 `.gdignore`，文档变更与代码分开提交即可。
- 一次提交聚焦一个主题（与项目提交习惯一致）。

---

## 14. 自检清单（提交前）

- [ ] 参数与返回值均有类型
- [ ] 无 `_process` 内 `get_node`
- [ ] 信号在 `_exit_tree` 断开
- [ ] 玩法数值来自 `resources/` 或 `@export`
- [ ] 新场景已设唯一名称或 `@onready` 缓存
- [ ] 同屏实体考虑池化
- [ ] 标识符英文，用户可见文本可本地化

---

## 相关文档

- [game-design.md](../design/game-design.md) — 玩法总览
- [weapon-execution-sequence.md](../design/features/weapon-execution-sequence.md) — 攻击序列
- [bad-patterns.md](bad-patterns.md) — 禁区清单

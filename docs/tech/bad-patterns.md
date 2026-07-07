# 禁区：常见自坑模式

> 已验证会出问题的做法。**任何人（含 AI）都不应重复。**

完整编码规范见 [code-style.md](code-style.md)。

---

## 1. `get_node()` 硬编码深层路径

```gdscript
# ❌ 节点改名或移动就崩
var hp_label = get_node("../../UI/HUD/HealthLabel")

# ✅
@onready var _health_label: Label = %HealthLabel
```

---

## 2. 信号只连不断

```gdscript
# ❌ 场景切换后仍触发，空引用崩溃
func _ready() -> void:
	EventBus.round_ended.connect(_on_round_ended)

# ✅
func _exit_tree() -> void:
	if EventBus.round_ended.is_connected(_on_round_ended):
		EventBus.round_ended.disconnect(_on_round_ended)
```

---

## 3. `_process()` 里做昂贵运算

```gdscript
# ❌ 每帧全量寻敌
func _process(_delta: float) -> void:
	_target = _find_nearest_enemy()

# ✅ Timer 或物理帧降频
```

---

## 4. 在 `_init()` 访问子节点

子节点尚未入树，`$Node` 为 `null`。使用 `@onready` 或 `_ready()`。

---

## 5. 相对路径加载资源

只用 `res://`，不用 `"../assets/foo.png"`。

---

## 6. UI 脚本里写伤害 / 价格公式

```gdscript
# ❌
func _on_buy_pressed() -> void:
	player.tokens -= item.price * 1.5  # 隐藏平衡逻辑

# ✅
func _on_buy_pressed() -> void:
	ShopService.try_purchase(item_id)
```

---

## 7. 每发子弹 `instantiate` + `queue_free`

幸存者同屏弹体多，必须对象池，否则 GC 与实例化拖垮帧率。

---

## 8. 武器效果写死在弹体 `area_entered` 里

命中燃烧、分裂、爆炸应走 **执行序列 `on_hit` 子序列**，否则扩展系统无法组合。

---

## 9. `get_parent().get_parent()` 跨层拿管理器

用 Autoload、`@export var manager` 注入，或信号上行。

---

## 10. 循环依赖（A 引 B，B 引 A）

系统层单向依赖：`ui → systems → resources/data`。EventBus 传事件，不互持引用。

---

## 11. 把回合结束条件写成「清场」

设计为**时间到即结束**；勿在代码里要求 `get_tree().get_nodes_in_group("enemies").is_empty()` 才进商店。

---

## 12. 品质色写死在 UI 脚本

颜色与 `glow_strength` 从品质表 Resource 读取，与 [item-quality.md](../design/features/item-quality.md) 一致。

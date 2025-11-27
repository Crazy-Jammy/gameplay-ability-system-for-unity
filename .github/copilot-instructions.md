# EX-GAS (Unity 游戏能力系统) - AI 代理指南

## 项目概览

**EX-GAS** 是虚幻引擎 Gameplay Ability System 的 Unity 移植版 - 一个用于管理能力、效果和游戏交互的复杂的属性驱动框架。这是一个**复杂的数据驱动架构**，需要理解多个系统之间的相互作用。

**核心理念**: "WHO DO WHAT"（谁做什么）
- **WHO（谁）**: `AbilitySystemComponent` (ASC) - 执行操作的实体
- **DO（做）**: `Ability` - 执行的动作  
- **WHAT（什么）**: `GameplayEffect` - 应用的结果/修改

## 关键依赖

- **Odin Inspector 3.2+** (付费资源) - 自定义检视器必需
- **Unity 2022.3+**
- 框架源于中文；文档混合中英文

## 架构：五大支柱

### 1. GameplayTag 系统（层级状态管理）
**位置**: `Assets/GAS/Runtime/Tags/`, `Assets/GAS/Editor/Tags/`

标签用树形结构（`Parent.Child.Grandchild`）替代布尔值/枚举状态标记。

**关键模式**:
- 标签使用点号表示法（`State.Buff.PowerUp`），但在生成的代码中变为下划线（`State_Buff_PowerUp`）
- **关键**: 在 Tag Manager 中编辑标签后，务必点击**"生成TagLib"**重新生成 `GTagLib.gen.cs`
- 标签层级结构支持强大的过滤功能（一次性移除所有 `Debuff.*` 标签）

**常见标签分类**:
```
State.Buff.*          # 正面状态效果
State.Debuff.*        # 负面状态效果
Ability.Type.*        # 技能分类
GameplayEffect.Type.* # 效果分类
```

### 2. Attribute 与 AttributeSet 系统
**位置**: `Assets/GAS/Runtime/Attribute/`, `Assets/GAS/Runtime/AttributeSet/`

属性**仅在 AttributeSet 内唯一**（类似姓氏+名字）:
- `AS_Fight.Health` ≠ `AS_Weapon.Health`（武器耐久度）
- 一个 ASC 可以有多个 AttributeSet，但**建议每个单位只用一个 AttributeSet**

**代码生成工作流**:
1. 在 Attribute Manager 中定义属性 → 点击**"生成AttrLib"**
2. 在 AttributeSet Manager 中将属性分配到 AttributeSet → 点击**"生成AttrSetLib"**
3. 生成的类：`AS_[集合名]`（例如，"Fight" AttributeSet 对应 `AS_Fight`）

**属性访问模式**:
```csharp
asc.GetAttributeCurrentValue("AS_Fight", "Health");
asc.AttrSet<AS_Fight>().Health.CurrentValue;
```

### 3. GameplayEffect (GE) - 唯一的属性修改器
**位置**: `Assets/GAS/Runtime/Effects/`

**黄金法则**: 在 GAS 系统中，只有 GameplayEffect 可以修改属性（初始化不算修改）。

**持续时间策略**:
- `Instant`: 执行一次，自我销毁（伤害/治疗）
- `Duration`: 带计时器的临时 buff
- `Infinite`: 永久存在直到手动移除

**效果施加 vs 激活**（关键区别）:
- **施加（Applied）**: GE 附加到目标 ASC
- **激活（Activated）**: GE 实际运行并修改数值
- 示例：被动回血 GE 在被 debuff 阻止治疗时保持*施加*状态，debuff 结束后*重新激活*

**堆叠系统**:
- `stackingCodeName`: 基于哈希的可堆叠效果标识符
- `StackingType.AggregateBySource`: 每个施法者单独堆叠计数
- `StackingType.AggregateByTarget`: 共享堆叠计数器
- 溢出触发器可以应用额外效果（`overflowEffects[]`）

**基于标签的逻辑**:
- `ApplicationRequiredTags`: 目标必须拥有所有这些标签
- `OngoingRequiredTags`: 如果缺少标签，GE 失活（不是移除）
- `RemoveGameplayEffectsWithTags`: 清除拥有任意匹配标签的效果
- `ApplicationImmunityTags`: 目标拥有任意这些标签时免疫该 GE

### 4. ModifierMagnitudeCalculation (MMC)
**位置**: `Assets/GAS/Runtime/Effects/Modifier/`

MMC 计算属性修改数值。四种内置类型：

1. **ScalableFloatModCalculation**: 线性函数 `Magnitude * k + b`
2. **AttributeBasedModCalculation**: 读取属性 `AttributeValue * k + b`
   - `captureType = Track`: 应用时读取当前值
   - `captureType = SnapShot`: GE 创建时捕获
3. **SetByCallerModCalculation**: 运行时通过 `spec.RegisterValue(key, value)` 设置值
4. **CustomCalculation**: 继承 `ModifierMagnitudeCalculation` 实现复杂逻辑

**操作类型**:
- `Add`: 加法（使用负数实现减法）
- `Multiply`: 乘法（使用倒数实现除法）  
- `Override`: 覆写属性值

### 5. Ability 系统
**位置**: `Assets/GAS/Runtime/Ability/`

**三类模式**（始终一起实现）:
```
AbilityAsset     → ScriptableObject 配置（面向设计师）
Ability          → 运行时数据包装器
AbilitySpec      → 运行时实例及游戏逻辑（面向开发者）
```

**代码生成**:
- 在 Ability Asset 中编辑 `U-Name` 后，通过 Asset Aggregator 重新生成 `AbilityLib.gen.cs`
- 运行时技能查找：`AbilityLib.CreateAbility("AbilityName")`

**基于标签的技能控制**:
- `CancelAbilityWithTags`: 激活时取消拥有任意这些标签的技能
- `BlockAbilityWithTags`: 阻止激活拥有任意这些标签的技能
- `ActivationRequiredTags`: 必须拥有所有标签才能激活
- `ActivationBlockedTags`: 拥有任意这些标签时无法激活

**TimelineAbility**（高级）:
- 基于序列的技能的可视化时间轴编辑器
- 6 种轨道类型：即时 Cue、释放效果、即时任务、持续 Cue、Buff、持续任务
- 用于攻击连招、引导技能、脚本序列

**从 GE 授予的技能**:
GameplayEffect 可以授予带生命周期策略的临时技能：
- `ActivationPolicy`: 何时激活（None/WhenAdded/SyncWithEffect）
- `DeactivationPolicy`: 何时取消激活（None/SyncWithEffect）
- `RemovePolicy`: 何时移除技能（None/SyncWithEffect/WhenEnd/WhenCancel/WhenCancelOrEnd）

## AbilitySystemComponent (ASC) - 核心枢纽

**初始化模式**:
```csharp
asc.InitWithPreset(level: 1, ascPreset);
GameplayAbilitySystem.GAS.Register(asc);
```

**关键容器**（ASC 组成）:
- `AbilityContainer`: 管理拥有的技能
- `GameplayEffectContainer`: 活动效果（可以理解为 buff 管理器）
- `AttributeSetContainer`: 保存属性数据
- `GameplayTagAggregator`: 固定标签 + 动态标签

**常用操作**:
```csharp
// 对目标施加 GE
var spec = asc.ApplyGameplayEffectTo(geData, targetASC);

// 激活技能
asc.TryActivateAbility("AbilityName", args);

// 标签查询
asc.HasTag(tag);
asc.HasAllTags(tagSet);
asc.HasAnyTags(tagSet);
```

## GameplayCue 系统（表现层）
**位置**: `Assets/GAS/Runtime/Cue/`

**黄金法则**: Cue 不得影响数值游戏逻辑（属性、标签等）。仅用于视觉/音效。

**两种类型**:
1. `GameplayCueInstant` + `GameplayCueInstantSpec`: 一次性（打击特效、声音）
2. `GameplayCueDurational` + `GameplayCueDurationalSpec`: 持续性（buff 视效、状态动画）

**自定义实现**:
```csharp
public class MyCue : GameplayCueInstant { }
public class MyCueSpec : GameplayCueInstantSpec {
    public override void Trigger() { /* VFX 逻辑 */ }
}
```

## 关键工作流程

### 添加新技能
1. 创建 `MyAbilityAsset.cs`（ScriptableObject）继承自 `AbilityAsset`
2. 创建 `MyAbility.cs`，包含配对的 `Ability` 和 `AbilitySpec` 类
3. 在 Unity 中创建资源，设置 `U-Name`
4. **前往 Asset Aggregator → C-Ability → 点击"生成AbilityLib"**
5. 在代码中使用 `AbilityLib.CreateAbility("MyAbilityName")`

### 创建 GameplayEffect
1. 通过 Asset Aggregator 或 Project 窗口创建资源
2. 通过自定义编辑器配置（标签、修改器、cues、堆叠）
3. 在 Ability 代码中引用或直接应用：
   ```csharp
   var spec = source.ApplyGameplayEffectTo(myGE, target);
   ```

### 预缓存优化
在游戏初始化时调用以避免 `Type.Name` 产生 GC：
```csharp
GasCache.CacheAttributeSetName(GAttrSetLib.TypeToName);
```

## 编辑器工具导航

**主菜单**: `EX-GAS` → Asset Aggregator
- **GameplayTag Manager**: 标签层级树编辑器
- **Attribute Manager**: 定义属性名称  
- **AttributeSet Manager**: 将属性分组到集合中
- **Asset Aggregator**: 查看/创建 GE、Abilities、MMC、Cues

**运行时监视器**: `EX-GAS` → Runtime Watcher
- 实时查看 ASC 状态（技能、属性、效果、标签）

**项目设置**: `Edit` → Project Settings → EX Gameplay Ability System
- 配置文件资源路径
- 脚本生成路径
- **"检查子目录文件夹"**: 确保所有文件夹存在

## 常见陷阱

1. **编辑 Tags/Attributes/Abilities 后忘记重新生成 Lib 文件**
2. **破坏堆叠机制** - 忘记 `stackingCodeName` 必须匹配才能共享堆叠
3. **混淆 GE 移除与失活** - GE 只是失活时不会被移除（检查标签要求）
4. **在 GE 系统外修改属性** 会破坏追踪/调试
5. **在 TargetCatcher/AbilityTask 中引用 ScriptableObject** 会导致 JSON GUID 问题
6. **创建 ASC 后未调用 `GameplayAbilitySystem.GAS.Register(asc)`**

## 文件结构约定

```
Assets/GAS/
├── Runtime/
│   ├── Ability/          # Ability 类、容器
│   ├── Attribute/        # Attribute 基类
│   ├── AttributeSet/     # AttributeSet 容器
│   ├── Component/        # AbilitySystemComponent
│   ├── Core/             # GameplayAbilitySystem 单例
│   ├── Cue/              # GameplayCue 实现
│   ├── Effects/          # GameplayEffect、MMC
│   ├── Tags/             # GameplayTag 运行时
│   └── Utils/            # 辅助工具
├── Editor/
│   ├── Ability/          # Ability 资源编辑器
│   ├── Effect/           # GE 自定义编辑器
│   ├── Tags/             # Tag 树编辑器
│   └── GameplayAbilitySystem/ # 主编辑器窗口
└── General/              # 共享工具（依赖 Odin）
```

## 测试与调试

- 使用 **Runtime Watcher**（`EX-GAS` 菜单）检查实时 ASC 状态
- 检查 `GameplayEffectContainer.GameplayEffects()` 查看活动效果
- 在调试 ability/GE 失败前，通过 `GameplayTagAggregator` 验证标签存在
- 启用 `UNITY_EDITOR` 条件编译符号用于编辑器专用调试

## 语言与文档说明

- 主要文档：中文（`README.md`、内联注释）
- 教程系列：知乎文章（链接见 README）
- QQ 反馈群：616570103（bug/支持）
- 生成的代码为英文，带中文 XML 注释
- 编辑器标签使用 `GASTextDefine` 常量（混合中英文）

## 性能考虑

- 使用 `CatchTargetsNonAlloc()` 而非 `CatchTargets()` 以避免 GC
- 启动时通过 `GasCache` 预缓存 AttributeSet 名称
- 限制 GameplayCueDurationalSpec 中 `OnTick()` 的操作
- 避免频繁的 ASC 注册/注销

---

## 回合制游戏适配指南

EX-GAS 原本设计面向实时战斗游戏（每帧驱动），但**完全可以应用于回合制游戏**。以下是适配方案和建议：

### 核心差异与优势

**回合制游戏的特点**:
- ✅ **事件驱动** 而非帧驱动 - 行动由玩家/AI 决策触发，非时间流逝
- ✅ **精确的数值计算** - 不需要考虑帧率、插值等实时问题
- ✅ **状态管理简单** - 回合结束时状态明确，易于保存/回滚
- ✅ **网络同步友好** - 只需同步决策和结果，非每帧状态

**GAS 的天然优势**:
- GameplayTag 系统非常适合回合制的状态管理（眩晕、禁锢、嘲讽等）
- GameplayEffect 的堆叠机制完美契合回合制 buff 设计
- Ability 系统可以表达技能释放的完整流程
- 属性系统支持复杂的数值计算（暴击、闪避、伤害计算等）

### 适配方案

#### 方案一：暂停 GAS Tick，改用手动触发（推荐）

**实现步骤**:

1. **关闭自动 Tick**
```csharp
// 游戏初始化时暂停 GAS 自动更新
GameplayAbilitySystem.GAS.Pause();
```

2. **在回合/行动节点手动 Tick**
```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    void OnTurnStart()
    {
        // 回合开始时，手动 Tick 一次处理持续效果
        foreach (var unit in battleUnits)
        {
            unit.ASC.Tick();
        }
    }
    
    void OnActionExecute(BattleUnit actor, Ability action, BattleUnit target)
    {
        // 执行技能
        actor.ASC.TryActivateAbility(action.Name, target.ASC);
        
        // 可选：立即 Tick 一次确保效果生效
        target.ASC.Tick();
    }
    
    void OnTurnEnd()
    {
        // 回合结束时处理 buff 计时
        foreach (var unit in battleUnits)
        {
            DecrementBuffDurations(unit.ASC);
        }
    }
}
```

3. **修改持续时间机制** - 用回合数代替秒数
```csharp
// 自定义 GameplayEffect 包装类
public class TurnBasedGameplayEffect
{
    public GameplayEffect Effect;
    public int RemainingTurns; // 剩余回合数
    
    public void OnTurnEnd()
    {
        RemainingTurns--;
        if (RemainingTurns <= 0)
        {
            // 移除效果
        }
    }
}
```

**优点**:
- ✅ 完全控制执行时机
- ✅ 性能最优（按需执行）
- ✅ 易于调试和日志记录
- ✅ 便于网络同步

**缺点**:
- ⚠️ 需要手动管理 GE 的持续时间（不能直接用 Duration 字段）
- ⚠️ Period 周期机制需要重新设计

#### 方案二：保留 Tick，使用虚拟时间（适合有动画表现的回合制）

如果你的回合制游戏有技能动画、过场演出，可以保留 GAS 的 Tick 机制：

```csharp
public class TurnBasedTimeController : MonoBehaviour
{
    private bool isAnimating = false;
    
    void Update()
    {
        if (isAnimating)
        {
            // 动画播放期间，GAS 正常运行
            // TimelineAbility 可以播放技能动画
        }
        else
        {
            // 等待玩家输入时，暂停 GAS
            GameplayAbilitySystem.GAS.Pause();
        }
    }
    
    public void PlaySkillAnimation(Ability ability)
    {
        isAnimating = true;
        GameplayAbilitySystem.GAS.Unpause();
        
        // TimelineAbility 自动播放
        // 完成后回调
    }
}
```

### 回合制特定的 GAS 使用模式

#### 1. 标签用于状态控制

```csharp
// 定义回合制特有标签
State.Control.Stun        // 眩晕（跳过回合）
State.Control.Sleep       // 睡眠（攻击唤醒）
State.Control.Taunt       // 嘲讽（强制目标）
State.Phase.Acting        // 正在行动
State.Phase.Waiting       // 等待回合
Ability.Limit.OncePerTurn // 每回合限一次
```

```csharp
// 检查是否可以行动
public bool CanAct(AbilitySystemComponent unit)
{
    return !unit.HasAnyTags(new[] {
        GTagLib.State_Control_Stun,
        GTagLib.State_Control_Sleep,
        GTagLib.State_Phase_Acting
    });
}
```

#### 2. GameplayEffect 用于 Buff/Debuff

```csharp
// 创建持续 N 回合的 buff
public void ApplyBuff(AbilitySystemComponent caster, AbilitySystemComponent target, 
                      GameplayEffect buffEffect, int turns)
{
    var spec = caster.ApplyGameplayEffectTo(buffEffect, target);
    
    // 使用 Infinite 策略，手动管理回合计数
    spec.SetDurationPolicy(EffectsDurationPolicy.Infinite);
    
    // 存储回合数（可以用自定义组件或字典管理）
    buffManager.RegisterBuff(spec, turns);
}

// 回合结束时更新
public void OnTurnEnd()
{
    foreach (var (spec, turns) in buffManager.ActiveBuffs)
    {
        turns--;
        if (turns <= 0)
        {
            spec.Owner.RemoveGameplayEffect(spec);
        }
    }
}
```

#### 3. Ability 用于技能释放

```csharp
// 回合制技能示例
public class TurnBasedAttackAbility : Ability
{
    // 配置在 AbilityAsset 中
    public int ActionPointCost;      // 行动点消耗
    public int CooldownTurns;        // 冷却回合数
    public bool CanTargetSelf;       // 能否自我施放
    public TargetType targetType;    // 单体/范围/全体
}

public class TurnBasedAttackAbilitySpec : AbilitySpec<TurnBasedAttackAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // 消耗行动点（通过 Cost GE）
        // 计算伤害（通过 MMC）
        // 应用效果（GE）
        var damageGE = Ability.DataReference.DamageEffect;
        Owner.ApplyGameplayEffectTo(damageGE, target);
        
        // 触发表现（Cue）
        // ...
        
        EndAbility();
    }
}
```

#### 4. MMC 用于复杂伤害计算

```csharp
// 回合制伤害计算示例
public class TurnBasedDamageCalculation : ModifierMagnitudeCalculation
{
    public override float CalculateMagnitude(GameplayEffectSpec spec)
    {
        var attacker = spec.Source;
        var defender = spec.Owner;
        
        // 读取攻击力
        float attack = attacker.GetAttributeCurrentValue("AS_Combat", "Attack") ?? 0;
        
        // 读取防御力
        float defense = defender.GetAttributeCurrentValue("AS_Combat", "Defense") ?? 0;
        
        // 暴击判定
        float critRate = attacker.GetAttributeCurrentValue("AS_Combat", "CritRate") ?? 0;
        bool isCrit = Random.value < critRate;
        float critMultiplier = isCrit ? 1.5f : 1.0f;
        
        // 属性克制（通过 Tag 判断）
        float elementBonus = 1.0f;
        if (attacker.HasTag(GTagLib.Element_Fire) && 
            defender.HasTag(GTagLib.Element_Ice))
        {
            elementBonus = 1.5f;
        }
        
        // 最终伤害 = (攻击力 - 防御力) * 暴击倍率 * 属性克制
        float damage = Mathf.Max(0, attack - defense) * critMultiplier * elementBonus;
        
        return damage;
    }
}
```

### 核心机制详解

#### 机制 1: Buff Duration 和 Period 的回合制改造

**问题**: GE 的 Duration 和 Period 是基于时间（秒）的，回合制需要基于回合数。

**解决方案**: 扩展 GameplayEffectSpec，添加回合计数

```csharp
// 1. 创建回合制 GE 包装类
public class TurnBasedEffectData
{
    public GameplayEffectSpec Spec;
    public int TotalTurns;          // 总回合数
    public int RemainingTurns;      // 剩余回合数
    public int TickInterval;        // 每几回合触发一次（Period 替代）
    public int NextTickTurn;        // 下次触发回合
    public EffectTimingType Timing; // 结算时机
    
    public enum EffectTimingType
    {
        OnTurnStart,    // 回合开始时结算
        OnTurnEnd,      // 回合结束时结算
        Immediate       // 立即生效（自身增益）
    }
}

// 2. 创建回合制效果管理器
public class TurnBasedEffectManager : MonoBehaviour
{
    // 按持有者分组管理效果
    private Dictionary<AbilitySystemComponent, List<TurnBasedEffectData>> allEffects = new();
    
    /// <summary>
    /// 应用回合制效果
    /// </summary>
    public void ApplyEffect(AbilitySystemComponent source, AbilitySystemComponent target, 
                           GameplayEffect effect, int turns, 
                           TurnBasedEffectData.EffectTimingType timing = TurnBasedEffectData.EffectTimingType.OnTurnStart,
                           int tickInterval = 0)
    {
        // 创建 GE Spec（使用 Infinite 策略）
        var spec = source.ApplyGameplayEffectTo(effect, target);
        if (spec == null) return;
        
        spec.SetDurationPolicy(EffectsDurationPolicy.Infinite);
        
        // 包装为回合制数据
        var turnData = new TurnBasedEffectData
        {
            Spec = spec,
            TotalTurns = turns,
            RemainingTurns = turns,
            TickInterval = tickInterval,
            NextTickTurn = tickInterval > 0 ? tickInterval : 0,
            Timing = timing
        };
        
        // 立即生效类型（自身增益）
        if (timing == TurnBasedEffectData.EffectTimingType.Immediate)
        {
            // 已经在 ApplyGameplayEffectTo 时生效了
        }
        
        // 注册到管理器
        if (!allEffects.ContainsKey(target))
        {
            allEffects[target] = new List<TurnBasedEffectData>();
        }
        allEffects[target].Add(turnData);
    }
    
    /// <summary>
    /// 回合开始时处理效果
    /// </summary>
    public void ProcessTurnStart(AbilitySystemComponent unit)
    {
        if (!allEffects.ContainsKey(unit)) return;
        
        var effects = allEffects[unit];
        var toRemove = new List<TurnBasedEffectData>();
        
        foreach (var effectData in effects)
        {
            // 只处理回合开始结算的效果
            if (effectData.Timing != TurnBasedEffectData.EffectTimingType.OnTurnStart)
                continue;
            
            // 处理周期性效果（Period）
            if (effectData.TickInterval > 0)
            {
                effectData.NextTickTurn--;
                if (effectData.NextTickTurn <= 0)
                {
                    // 触发周期效果（如果 GE 配置了 PeriodExecution）
                    ExecutePeriodEffect(effectData);
                    effectData.NextTickTurn = effectData.TickInterval;
                }
            }
            
            // 减少回合计数
            effectData.RemainingTurns--;
            
            if (effectData.RemainingTurns <= 0)
            {
                toRemove.Add(effectData);
            }
        }
        
        // 移除过期效果
        foreach (var data in toRemove)
        {
            data.Spec.Owner.RemoveGameplayEffect(data.Spec);
            effects.Remove(data);
        }
    }
    
    /// <summary>
    /// 回合结束时处理效果
    /// </summary>
    public void ProcessTurnEnd(AbilitySystemComponent unit)
    {
        if (!allEffects.ContainsKey(unit)) return;
        
        var effects = allEffects[unit];
        var toRemove = new List<TurnBasedEffectData>();
        
        foreach (var effectData in effects)
        {
            // 只处理回合结束结算的效果
            if (effectData.Timing != TurnBasedEffectData.EffectTimingType.OnTurnEnd)
                continue;
            
            // 处理周期性效果
            if (effectData.TickInterval > 0)
            {
                effectData.NextTickTurn--;
                if (effectData.NextTickTurn <= 0)
                {
                    ExecutePeriodEffect(effectData);
                    effectData.NextTickTurn = effectData.TickInterval;
                }
            }
            
            // 减少回合计数
            effectData.RemainingTurns--;
            
            if (effectData.RemainingTurns <= 0)
            {
                toRemove.Add(effectData);
            }
        }
        
        // 移除过期效果
        foreach (var data in toRemove)
        {
            data.Spec.Owner.RemoveGameplayEffect(data.Spec);
            effects.Remove(data);
        }
    }
    
    /// <summary>
    /// 执行周期性效果（Period 替代）
    /// </summary>
    private void ExecutePeriodEffect(TurnBasedEffectData effectData)
    {
        var periodGE = effectData.Spec.GameplayEffect.PeriodExecution;
        if (periodGE != null)
        {
            // 应用周期效果（通常是 Instant 类型的伤害/治疗）
            effectData.Spec.Source.ApplyGameplayEffectTo(periodGE, effectData.Spec.Owner);
        }
    }
    
    /// <summary>
    /// 清理单位的所有效果
    /// </summary>
    public void ClearUnitEffects(AbilitySystemComponent unit)
    {
        if (!allEffects.ContainsKey(unit)) return;
        
        foreach (var data in allEffects[unit])
        {
            data.Spec.Owner.RemoveGameplayEffect(data.Spec);
        }
        allEffects[unit].Clear();
    }
}
```

**Buff 结算时机示例**:

```csharp
// 敌方施加的 Debuff - 下回合开始结算
effectManager.ApplyEffect(
    source: enemy.ASC,
    target: player.ASC,
    effect: poisonEffect,
    turns: 3,
    timing: TurnBasedEffectData.EffectTimingType.OnTurnStart
);

// 自身增益 Buff - 当前回合立即生效，下回合结束计数-1
effectManager.ApplyEffect(
    source: player.ASC,
    target: player.ASC,
    effect: attackBoostEffect,
    turns: 2,
    timing: TurnBasedEffectData.EffectTimingType.OnTurnEnd
);

// 持续回血 - 每回合开始时触发
effectManager.ApplyEffect(
    source: healer.ASC,
    target: player.ASC,
    effect: regenEffect,
    turns: 5,
    timing: TurnBasedEffectData.EffectTimingType.OnTurnStart,
    tickInterval: 1  // 每回合触发一次
);
```


#### 机制 2: 回合制目标选择与技能释放

**问题**: 需要支持多种目标选择模式（单体、AOE、附近单位等）。

**解决方案**: 扩展 TargetCatcher 系统

```csharp
// 1. 单体目标选择
public class CatchSingleTarget : TargetCatcherBase
{
    protected override void CatchTargetsNonAlloc(AbilitySystemComponent mainTarget, 
                                                  List<AbilitySystemComponent> results)
    {
        if (mainTarget != null)
        {
            results.Add(mainTarget);
        }
    }
}

// 2. 全体敌方 AOE
public class CatchAllEnemies : TargetCatcherBase
{
    protected override void CatchTargetsNonAlloc(AbilitySystemComponent mainTarget, 
                                                  List<AbilitySystemComponent> results)
    {
        var battleManager = TurnBasedBattleManager.Instance;
        var ownerTeam = battleManager.GetTeam(Owner);
        
        // 获取所有敌方单位
        foreach (var unit in battleManager.AllUnits)
        {
            if (battleManager.GetTeam(unit.ASC) != ownerTeam && unit.IsAlive)
            {
                results.Add(unit.ASC);
            }
        }
    }
}

// 3. 全体友方 AOE（如群体治疗）
public class CatchAllAllies : TargetCatcherBase
{
    protected override void CatchTargetsNonAlloc(AbilitySystemComponent mainTarget, 
                                                  List<AbilitySystemComponent> results)
    {
        var battleManager = TurnBasedBattleManager.Instance;
        var ownerTeam = battleManager.GetTeam(Owner);
        
        foreach (var unit in battleManager.AllUnits)
        {
            if (battleManager.GetTeam(unit.ASC) == ownerTeam && unit.IsAlive)
            {
                results.Add(unit.ASC);
            }
        }
    }
}

// 4. 目标 + 附近单位（如溅射伤害）
public class CatchTargetAndNearby : TargetCatcherBase
{
    public int nearbyCount = 2; // 附近单位数量
    
    protected override void CatchTargetsNonAlloc(AbilitySystemComponent mainTarget, 
                                                  List<AbilitySystemComponent> results)
    {
        if (mainTarget == null) return;
        
        var battleManager = TurnBasedBattleManager.Instance;
        var targetTeam = battleManager.GetTeam(mainTarget);
        
        // 添加主目标
        results.Add(mainTarget);
        
        // 获取同队的其他单位（按位置排序）
        var teammates = battleManager.AllUnits
            .Where(u => battleManager.GetTeam(u.ASC) == targetTeam && 
                       u.ASC != mainTarget && 
                       u.IsAlive)
            .OrderBy(u => battleManager.GetDistance(mainTarget, u.ASC))
            .Take(nearbyCount);
        
        foreach (var unit in teammates)
        {
            results.Add(unit.ASC);
        }
    }
}

// 5. 随机 N 个敌方目标
public class CatchRandomEnemies : TargetCatcherBase
{
    public int targetCount = 3;
    
    protected override void CatchTargetsNonAlloc(AbilitySystemComponent mainTarget, 
                                                  List<AbilitySystemComponent> results)
    {
        var battleManager = TurnBasedBattleManager.Instance;
        var ownerTeam = battleManager.GetTeam(Owner);
        
        var enemies = battleManager.AllUnits
            .Where(u => battleManager.GetTeam(u.ASC) != ownerTeam && u.IsAlive)
            .OrderBy(x => Random.value) // 随机排序
            .Take(targetCount);
        
        foreach (var enemy in enemies)
        {
            results.Add(enemy.ASC);
        }
    }
}

// 6. 血量最低的友方单位
public class CatchLowestHealthAlly : TargetCatcherBase
{
    protected override void CatchTargetsNonAlloc(AbilitySystemComponent mainTarget, 
                                                  List<AbilitySystemComponent> results)
    {
        var battleManager = TurnBasedBattleManager.Instance;
        var ownerTeam = battleManager.GetTeam(Owner);
        
        var lowestHpAlly = battleManager.AllUnits
            .Where(u => battleManager.GetTeam(u.ASC) == ownerTeam && u.IsAlive)
            .OrderBy(u => u.ASC.GetAttributeCurrentValue("AS_Combat", "Health"))
            .FirstOrDefault();
        
        if (lowestHpAlly != null)
        {
            results.Add(lowestHpAlly.ASC);
        }
    }
}

// 7. 前排/后排目标
public class CatchFrontRow : TargetCatcherBase
{
    public bool targetEnemies = true; // true=敌方前排, false=友方前排
    
    protected override void CatchTargetsNonAlloc(AbilitySystemComponent mainTarget, 
                                                  List<AbilitySystemComponent> results)
    {
        var battleManager = TurnBasedBattleManager.Instance;
        var ownerTeam = battleManager.GetTeam(Owner);
        var targetTeam = targetEnemies ? (ownerTeam == 0 ? 1 : 0) : ownerTeam;
        
        var frontRow = battleManager.AllUnits
            .Where(u => battleManager.GetTeam(u.ASC) == targetTeam && 
                       u.IsAlive && 
                       battleManager.IsInFrontRow(u.ASC));
        
        foreach (var unit in frontRow)
        {
            results.Add(unit.ASC);
        }
    }
}
```

**技能释放示例**:

```csharp
// 单体攻击技能
public class SingleAttackAbilitySpec : AbilitySpec<SingleAttackAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // 检查目标有效性
        if (target == null || !IsValidTarget(target))
        {
            Debug.LogError("无效目标");
            EndAbility();
            return;
        }
        
        // 应用伤害 GE
        var damageGE = Ability.DataReference.DamageEffect;
        Owner.ApplyGameplayEffectTo(damageGE, target);
        
        // 触发 Cue
        TriggerAttackCue(target);
        
        EndAbility();
    }
}

// AOE 技能（全体攻击）
public class AOEAttackAbilitySpec : AbilitySpec<AOEAttackAbility>
{
    private CatchAllEnemies targetCatcher = new CatchAllEnemies();
    
    public override void ActivateAbility(params object[] args)
    {
        targetCatcher.Init(Owner);
        
        var targets = new List<AbilitySystemComponent>();
        targetCatcher.CatchTargetsNonAlloc(null, targets);
        
        var damageGE = Ability.DataReference.DamageEffect;
        
        foreach (var target in targets)
        {
            Owner.ApplyGameplayEffectTo(damageGE, target);
            TriggerHitCue(target);
        }
        
        EndAbility();
    }
}

// 溅射伤害技能（主目标 + 附近 2 个单位）
public class SplashAttackAbilitySpec : AbilitySpec<SplashAttackAbility>
{
    private CatchTargetAndNearby targetCatcher;
    
    public SplashAttackAbilitySpec(SplashAttackAbility ability, AbilitySystemComponent owner) 
        : base(ability, owner)
    {
        targetCatcher = new CatchTargetAndNearby { nearbyCount = 2 };
        targetCatcher.Init(owner);
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var mainTarget = args[0] as AbilitySystemComponent;
        
        var targets = new List<AbilitySystemComponent>();
        targetCatcher.CatchTargetsNonAlloc(mainTarget, targets);
        
        var primaryDamageGE = Ability.DataReference.PrimaryDamageEffect;
        var splashDamageGE = Ability.DataReference.SplashDamageEffect; // 溅射伤害较低
        
        for (int i = 0; i < targets.Count; i++)
        {
            var target = targets[i];
            
            // 第一个是主目标，造成全额伤害
            if (i == 0)
            {
                Owner.ApplyGameplayEffectTo(primaryDamageGE, target);
            }
            // 其他是溅射目标，造成较低伤害
            else
            {
                Owner.ApplyGameplayEffectTo(splashDamageGE, target);
            }
            
            TriggerHitCue(target, i == 0);
        }
        
        EndAbility();
    }
}
```

**目标选择的最佳实践**:

**推荐方案：传入主目标，在 AbilitySpec 内部使用 TargetCatcher 计算**

```csharp
// ✅ 推荐：UI 层只传主目标
public void OnPlayerSelectTarget(BattleUnit selectedUnit)
{
    // UI 层只负责选择主目标
    currentUnit.ASC.TryActivateAbility("SplashAttack", selectedUnit.ASC);
}

// AbilitySpec 内部处理目标选择逻辑
public class SplashAttackAbilitySpec : AbilitySpec<SplashAttackAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var mainTarget = args[0] as AbilitySystemComponent;
        
        // 在技能内部使用 TargetCatcher 计算所有目标
        var targetCatcher = new CatchTargetAndNearby { nearbyCount = 2 };
        targetCatcher.Init(Owner);
        
        var targets = new List<AbilitySystemComponent>();
        targetCatcher.CatchTargetsNonAlloc(mainTarget, targets);
        
        // 对所有目标应用效果
        foreach (var target in targets)
        {
            Owner.ApplyGameplayEffectTo(damageGE, target);
        }
        
        EndAbility();
    }
}
```

**为什么这样设计？**

1. **职责分离**:
   - UI 层：负责玩家交互，选择主目标
   - Ability 层：负责游戏逻辑，计算实际受影响的目标
   
2. **灵活性**:
   ```csharp
   // 同一个技能，可以根据不同条件使用不同的 TargetCatcher
   public override void ActivateAbility(params object[] args)
   {
       var mainTarget = args[0] as AbilitySystemComponent;
       
       // 根据技能等级或状态选择不同范围
       TargetCatcherBase catcher;
       if (Owner.HasTag(GTagLib.State_Buff_PowerUp))
       {
           catcher = new CatchTargetAndNearby { nearbyCount = 4 }; // 增强状态范围更大
       }
       else
       {
           catcher = new CatchTargetAndNearby { nearbyCount = 2 };
       }
       
       catcher.Init(Owner);
       // ...
   }
   ```

3. **易于测试和 AI 使用**:
   ```csharp
   // AI 只需要选择一个主目标，不需要关心范围计算
   var bestTarget = aiSystem.SelectBestTarget();
   asc.TryActivateAbility("SplashAttack", bestTarget);
   ```

4. **支持技能预览**:
   ```csharp
   // UI 可以提前预览受影响的目标
   public List<AbilitySystemComponent> PreviewTargets(AbilitySystemComponent mainTarget)
   {
       var catcher = new CatchTargetAndNearby { nearbyCount = 2 };
       catcher.Init(Owner);
       
       var targets = new List<AbilitySystemComponent>();
       catcher.CatchTargetsNonAlloc(mainTarget, targets);
       return targets;
   }
   
   // UI 调用
   void OnTargetHover(BattleUnit hoveredUnit)
   {
       var previewTargets = abilitySpec.PreviewTargets(hoveredUnit.ASC);
       HighlightTargets(previewTargets); // 高亮显示受影响的目标
   }
   ```

**特殊情况：全体 AOE 技能**

```csharp
// 全体 AOE 不需要主目标
public void OnPlayerUseAOE()
{
    // 可以传 null 或不传参数
    currentUnit.ASC.TryActivateAbility("AOEAttack");
}

public class AOEAttackAbilitySpec : AbilitySpec<AOEAttackAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        // 不需要主目标参数
        var catcher = new CatchAllEnemies();
        catcher.Init(Owner);
        
        var targets = new List<AbilitySystemComponent>();
        catcher.CatchTargetsNonAlloc(null, targets);
        
        foreach (var target in targets)
        {
            Owner.ApplyGameplayEffectTo(damageGE, target);
        }
        
        EndAbility();
    }
}
```

**对比：不推荐的做法**

```csharp
// ❌ 不推荐：在外部计算所有目标再传入
public void OnPlayerSelectTarget(BattleUnit selectedUnit)
{
    // UI 层需要了解目标选择逻辑
    var targetCatcher = new CatchTargetAndNearby { nearbyCount = 2 };
    targetCatcher.Init(currentUnit.ASC);
    
    var targets = new List<AbilitySystemComponent>();
    targetCatcher.CatchTargetsNonAlloc(selectedUnit.ASC, targets);
    
    // 传入所有目标
    currentUnit.ASC.TryActivateAbility("SplashAttack", targets.ToArray());
}

// 问题：
// 1. UI 层耦合了游戏逻辑
// 2. 无法根据技能状态动态调整范围
// 3. AI 也需要复制这套逻辑
// 4. 测试困难
```

**完整的技能使用流程示例**:

```csharp
public class BattleUIController : MonoBehaviour
{
    private AbilitySpec currentAbility;
    
    // 1. 玩家选择技能
    public void OnAbilityButtonClick(string abilityName)
    {
        currentAbility = playerASC.AbilityContainer.AbilitySpecs()[abilityName];
        
        // 检查技能是否可用
        if (currentAbility.CanActivate() != AbilityActivateResult.Success)
        {
            ShowAbilityNotAvailable();
            return;
        }
        
        // 根据技能类型决定 UI 交互
        if (RequiresTarget(abilityName))
        {
            EnterTargetSelectionMode();
        }
        else
        {
            // 全体 AOE 直接释放
            ExecuteAbility(null);
        }
    }
    
    // 2. 玩家选择目标
    public void OnTargetSelect(BattleUnit target)
    {
        ExecuteAbility(target.ASC);
    }
    
    // 3. 执行技能
    private void ExecuteAbility(AbilitySystemComponent mainTarget)
    {
        // 只传主目标（或 null），内部计算实际目标
        TurnBasedBattleManager.Instance.ExecuteAbility(
            playerUnit,
            currentAbility.Ability.Name,
            mainTarget
        );
        
        ExitTargetSelectionMode();
    }
    
    // 4. 目标预览（可选）
    public void OnTargetHover(BattleUnit hoveredUnit)
    {
        if (currentAbility is SplashAttackAbilitySpec splashAbility)
        {
            var previewTargets = splashAbility.PreviewTargets(hoveredUnit.ASC);
            HighlightTargets(previewTargets);
        }
    }
}
```



#### 机制 3: 完整的回合制战斗流程

将所有机制整合到完整的战斗管理器中：

```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    public static TurnBasedBattleManager Instance { get; private set; }
    
    [System.Serializable]
    public class BattleUnit
    {
        public AbilitySystemComponent ASC;
        public int Team; // 0=玩家方, 1=敌方
        public int Position; // 战场位置（用于前后排判断）
        public bool IsAlive => ASC.GetAttributeCurrentValue("AS_Combat", "Health") > 0;
    }
    
    public List<BattleUnit> AllUnits = new();
    private TurnBasedEffectManager effectManager;
    private int currentUnitIndex = 0;
    
    void Awake()
    {
        Instance = this;
        effectManager = GetComponent<TurnBasedEffectManager>();
    }
    
    void Start()
    {
        // 暂停 GAS 自动更新
        GameplayAbilitySystem.GAS.Pause();
        
        InitBattle();
        StartBattle();
    }
    
    void InitBattle()
    {
        // 初始化所有单位的 ASC
        foreach (var unit in AllUnits)
        {
            unit.ASC.InitWithPreset(1, unit.ASC.Preset);
            GameplayAbilitySystem.GAS.Register(unit.ASC);
        }
        
        // 按速度排序决定行动顺序
        AllUnits = AllUnits
            .OrderByDescending(u => u.ASC.GetAttributeCurrentValue("AS_Combat", "Speed"))
            .ToList();
    }
    
    void StartBattle()
    {
        Debug.Log("=== 战斗开始 ===");
        StartNextTurn();
    }
    
    void StartNextTurn()
    {
        // 跳过已死亡单位
        while (currentUnitIndex < AllUnits.Count && !AllUnits[currentUnitIndex].IsAlive)
        {
            currentUnitIndex++;
        }
        
        if (currentUnitIndex >= AllUnits.Count)
        {
            // 一轮结束，进入下一轮
            OnRoundEnd();
            currentUnitIndex = 0;
            StartNextTurn();
            return;
        }
        
        var currentUnit = AllUnits[currentUnitIndex];
        
        Debug.Log($">>> {currentUnit.ASC.name} 的回合开始");
        
        // 1. 回合开始时处理 Buff（敌方施加的 debuff 在这里结算）
        effectManager.ProcessTurnStart(currentUnit.ASC);
        
        // 2. 检查控制状态
        if (IsControlled(currentUnit.ASC))
        {
            Debug.Log($"{currentUnit.ASC.name} 被控制，跳过回合");
            EndCurrentTurn();
            return;
        }
        
        // 3. 等待行动选择
        if (currentUnit.Team == 0) // 玩家方
        {
            ShowPlayerActionMenu(currentUnit);
        }
        else // 敌方
        {
            StartCoroutine(AISelectAction(currentUnit));
        }
    }
    
    bool IsControlled(AbilitySystemComponent asc)
    {
        return asc.HasAnyTags(new[] {
            GTagLib.State_Control_Stun,
            GTagLib.State_Control_Freeze,
            GTagLib.State_Control_Sleep
        });
    }
    
    /// <summary>
    /// 执行技能
    /// </summary>
    public void ExecuteAbility(BattleUnit actor, string abilityName, AbilitySystemComponent target)
    {
        Debug.Log($"{actor.ASC.name} 使用 {abilityName}");
        
        bool success = actor.ASC.TryActivateAbility(abilityName, target);
        
        if (success)
        {
            // 技能执行后立即 Tick 一次，确保效果应用
            actor.ASC.Tick();
            
            // 检查战斗结束
            if (CheckBattleEnd())
            {
                return;
            }
        }
        
        EndCurrentTurn();
    }
    
    /// <summary>
    /// 结束当前回合
    /// </summary>
    void EndCurrentTurn()
    {
        var currentUnit = AllUnits[currentUnitIndex];
        
        Debug.Log($"<<< {currentUnit.ASC.name} 的回合结束");
        
        // 回合结束时处理 Buff（自身增益 buff 在这里计数）
        effectManager.ProcessTurnEnd(currentUnit.ASC);
        
        // 下一个单位
        currentUnitIndex++;
        
        // 延迟开始下一回合（给玩家查看信息的时间）
        StartCoroutine(DelayedNextTurn(0.5f));
    }
    
    IEnumerator DelayedNextTurn(float delay)
    {
        yield return new WaitForSeconds(delay);
        StartNextTurn();
    }
    
    /// <summary>
    /// 一整轮结束（所有单位都行动过）
    /// </summary>
    void OnRoundEnd()
    {
        Debug.Log("=== 回合轮次结束 ===");
        
        // 可以在这里处理回合轮次相关的逻辑
        // 例如：毒圈缩小、环境效果等
    }
    
    bool CheckBattleEnd()
    {
        var team0Alive = AllUnits.Any(u => u.Team == 0 && u.IsAlive);
        var team1Alive = AllUnits.Any(u => u.Team == 1 && u.IsAlive);
        
        if (!team0Alive)
        {
            Debug.Log("=== 战斗失败 ===");
            OnBattleEnd(false);
            return true;
        }
        
        if (!team1Alive)
        {
            Debug.Log("=== 战斗胜利 ===");
            OnBattleEnd(true);
            return true;
        }
        
        return false;
    }
    
    void OnBattleEnd(bool victory)
    {
        // 清理所有效果
        foreach (var unit in AllUnits)
        {
            effectManager.ClearUnitEffects(unit.ASC);
        }
        
        // 显示结算界面等
    }
    
    // 辅助方法
    public int GetTeam(AbilitySystemComponent asc)
    {
        return AllUnits.First(u => u.ASC == asc).Team;
    }
    
    public float GetDistance(AbilitySystemComponent from, AbilitySystemComponent to)
    {
        var fromUnit = AllUnits.First(u => u.ASC == from);
        var toUnit = AllUnits.First(u => u.ASC == to);
        return Mathf.Abs(fromUnit.Position - toUnit.Position);
    }
    
    public bool IsInFrontRow(AbilitySystemComponent asc)
    {
        var unit = AllUnits.First(u => u.ASC == asc);
        // 假设位置 0-2 是前排
        return unit.Position <= 2;
    }
}
```

**Buff 结算时机完整示例**:

```csharp
public class BuffApplicationExamples : MonoBehaviour
{
    private TurnBasedEffectManager effectManager;
    
    void Example_PlayerSelfBuff()
    {
        // 玩家给自己上增益 buff（攻击力提升 2 回合）
        // 当前回合立即生效，在第 2 个回合结束时移除
        effectManager.ApplyEffect(
            source: playerASC,
            target: playerASC,
            effect: attackBoostGE,
            turns: 2,
            timing: TurnBasedEffectData.EffectTimingType.Immediate
        );
        
        // 注意：Immediate 效果会立即应用，但计数在回合结束时-1
        // 回合 1（当前）：buff 生效，回合结束计数变为 1
        // 回合 2：buff 继续生效，回合结束计数变为 0，移除
    }
    
    void Example_EnemyDebuffOnPlayer()
    {
        // 敌人给玩家上 debuff（中毒，持续 3 回合）
        // 下回合开始才结算第一次伤害
        effectManager.ApplyEffect(
            source: enemyASC,
            target: playerASC,
            effect: poisonGE,
            turns: 3,
            timing: TurnBasedEffectData.EffectTimingType.OnTurnStart,
            tickInterval: 1  // 每回合触发一次
        );
        
        // 时间线：
        // 当前回合：buff 施加，不触发
        // 玩家回合 1 开始：触发第一次毒伤，计数变为 2
        // 玩家回合 2 开始：触发第二次毒伤，计数变为 1
        // 玩家回合 3 开始：触发第三次毒伤，计数变为 0，移除
    }
    
    void Example_HealOverTime()
    {
        // 治疗术（回复 5 回合）
        // 施放回合结束时第一次回血
        effectManager.ApplyEffect(
            source: healerASC,
            target: playerASC,
            effect: regenGE,
            turns: 5,
            timing: TurnBasedEffectData.EffectTimingType.OnTurnEnd,
            tickInterval: 1
        );
        
        // 时间线：
        // 当前回合结束：触发第一次回血，计数变为 4
        // 回合 1 结束：触发第二次回血，计数变为 3
        // ... 以此类推
    }
    
    void Example_ShieldBuff()
    {
        // 护盾（立即生效，持续到下回合结束）
        effectManager.ApplyEffect(
            source: playerASC,
            target: playerASC,
            effect: shieldGE,
            turns: 1,
            timing: TurnBasedEffectData.EffectTimingType.Immediate
        );
        
        // 时间线：
        // 当前回合：立即获得护盾
        // 下回合：护盾依然有效
        // 下回合结束：护盾移除
    }
}
```

### 关键要点总结

#### Buff/Debuff 结算规则

1. **敌方施加的 Debuff**:
   - 使用 `EffectTimingType.OnTurnStart`
   - 在目标的下一个回合开始时第一次结算
   - 示例：中毒、灼烧、流血

2. **自身施加的增益 Buff**:
   - 使用 `EffectTimingType.Immediate`
   - 立即生效，当前回合就能享受属性提升
   - 回合计数在回合**结束**时递减
   - 示例：攻击强化、防御强化

3. **持续治疗/伤害**:
   - OnTurnStart: 回合开始时触发（如持续掉血）
   - OnTurnEnd: 回合结束时触发（如持续回血）
   - 配合 `tickInterval` 控制触发频率

4. **护盾/临时 Buff**:
   - 使用 `Immediate` 立即生效
   - 持续时间根据游戏设计调整

#### Duration 和 Period 改造总结

| 原 GAS 机制 | 回合制改造 | 说明 |
|------------|-----------|------|
| Duration (秒) | TotalTurns (回合数) | 效果持续的总回合数 |
| Period (秒) | TickInterval (回合) | 每几回合触发一次周期效果 |
| Infinite | 保留使用 | 所有回合制 GE 都用 Infinite，手动管理 |
| OngoingRequiredTags | 保留使用 | Tag 失活机制在回合制中同样有用 |

#### 技能目标选择总结

| 技能类型 | TargetCatcher | 示例 |
|---------|--------------|------|
| 单体攻击 | CatchSingleTarget | 普通攻击 |
| 全体 AOE | CatchAllEnemies | 全屏大招 |
| 溅射伤害 | CatchTargetAndNearby | 主目标+周围2个 |
| 随机多目标 | CatchRandomEnemies | 随机3个敌人 |
| 治疗最低血量 | CatchLowestHealthAlly | 奶妈技能 |
| 前排攻击 | CatchFrontRow | 坦克克星 |
| 全体友方 | CatchAllAllies | 群体护盾 |

### 推荐的回合制 Tag 设计

```
# 回合状态
State.Turn.Active          # 当前行动单位
State.Turn.Waiting         # 等待回合
State.Turn.ActionDone      # 已行动（本回合）

# 控制状态
State.Control.Stun         # 眩晕（无法行动）
State.Control.Freeze       # 冰冻
State.Control.Sleep        # 睡眠
State.Control.Charm        # 魅惑（攻击友军）
State.Control.Taunt        # 嘲讽（强制攻击目标）
State.Control.Silence      # 沉默（无法施法）
State.Control.Disarm       # 缴械（无法普攻）

# 属性状态
State.Buff.AttackUp        # 攻击提升
State.Buff.DefenseUp       # 防御提升
State.Buff.SpeedUp         # 速度提升
State.Debuff.AttackDown    # 攻击降低
State.Debuff.DefenseDown   # 防御降低

# 特殊机制
State.Special.Counter      # 反击状态
State.Special.Dodge        # 必定闪避
State.Special.Shield       # 护盾
State.Special.Immune       # 免疫控制

# 技能限制
Ability.Limit.OncePerTurn  # 每回合限一次
Ability.Limit.OncePerbattle # 每战斗限一次
```

### 总结建议

**强烈推荐回合制使用 EX-GAS，因为**:
1. ✅ Tag 系统完美契合回合制的状态管理
2. ✅ GE 的堆叠、免疫、移除机制非常适合 buff 设计
3. ✅ MMC 支持复杂的数值计算公式
4. ✅ 不需要关心实时性能问题

**关键调整**:
1. 关闭自动 Tick，改用事件驱动
2. Duration 改为回合计数
3. 不使用 TimelineAbility（除非需要动画）
4. 创建 TurnBasedEffectManager 管理效果生命周期

**额外收益**:
- 易于实现战斗回放（记录决策即可）
- 易于网络同步（只需同步指令）
- 易于平衡调整（数值配置化）
- 易于存档（状态明确）

---

## 回合制游戏重点关注模块

除了 Buff 管理和目标选择，以下模块在回合制游戏中需要特别设计：

### 模块 1: **行动顺序系统（速度/优先级）**

**为什么需要关注？**
- 回合制的核心体验依赖行动顺序
- 速度属性、Buff 加速/减速、优先级技能都会影响顺序
- 需要支持插队、延后、额外回合等特殊机制

**设计方案**:

```csharp
public class TurnOrderSystem
{
    public class TurnEntry
    {
        public AbilitySystemComponent Unit;
        public float InitiativeValue; // 行动值（速度累计）
        public int Priority;          // 优先级（同行动值时比较）
        public bool IsExtraTurn;      // 是否额外回合
    }
    
    private List<TurnEntry> turnQueue = new();
    
    /// <summary>
    /// 初始化回合顺序（基于速度属性）
    /// </summary>
    public void InitializeTurnOrder(List<AbilitySystemComponent> units)
    {
        turnQueue.Clear();
        
        foreach (var unit in units)
        {
            // 读取速度属性
            float speed = unit.GetAttributeCurrentValue("AS_Combat", "Speed") ?? 100;
            
            // 检查速度 Buff（通过 Tag 标记）
            if (unit.HasTag(GTagLib.State_Buff_SpeedUp))
            {
                speed *= 1.5f; // 加速 50%
            }
            if (unit.HasTag(GTagLib.State_Debuff_Slow))
            {
                speed *= 0.5f; // 减速 50%
            }
            
            turnQueue.Add(new TurnEntry
            {
                Unit = unit,
                InitiativeValue = speed,
                Priority = 0
            });
        }
        
        // 按行动值降序排序（速度快的先行动）
        SortTurnQueue();
    }
    
    /// <summary>
    /// 获取下一个行动单位
    /// </summary>
    public AbilitySystemComponent GetNextActor()
    {
        if (turnQueue.Count == 0) return null;
        
        var entry = turnQueue[0];
        turnQueue.RemoveAt(0);
        
        // 如果不是额外回合，将单位重新加入队列（下一轮）
        if (!entry.IsExtraTurn)
        {
            var speed = entry.Unit.GetAttributeCurrentValue("AS_Combat", "Speed") ?? 100;
            turnQueue.Add(new TurnEntry
            {
                Unit = entry.Unit,
                InitiativeValue = entry.InitiativeValue + speed, // 累加行动值
                Priority = 0
            });
            SortTurnQueue();
        }
        
        return entry.Unit;
    }
    
    /// <summary>
    /// 插入额外回合（如：连击技能）
    /// </summary>
    public void GrantExtraTurn(AbilitySystemComponent unit, int priority = 999)
    {
        turnQueue.Insert(0, new TurnEntry
        {
            Unit = unit,
            InitiativeValue = float.MaxValue,
            Priority = priority,
            IsExtraTurn = true
        });
    }
    
    /// <summary>
    /// 延后行动（如：冰冻延迟）
    /// </summary>
    public void DelayTurn(AbilitySystemComponent unit, float penaltyValue)
    {
        var entry = turnQueue.FirstOrDefault(e => e.Unit == unit);
        if (entry != null)
        {
            entry.InitiativeValue -= penaltyValue;
            SortTurnQueue();
        }
    }
    
    private void SortTurnQueue()
    {
        turnQueue = turnQueue
            .OrderByDescending(e => e.InitiativeValue)
            .ThenByDescending(e => e.Priority)
            .ToList();
    }
}
```

**GAS 集成要点**:
- 速度属性定义在 AttributeSet 中：`AS_Combat.Speed`
- 速度 Buff 通过 Tag 标记：`State.Buff.SpeedUp`, `State.Debuff.Slow`
- 优先级技能通过 Ability 授予额外回合

**使用示例**:

```csharp
// 连击技能 - 立即获得额外回合
public class ComboAttackAbilitySpec : AbilitySpec<ComboAttackAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // 造成伤害
        Owner.ApplyGameplayEffectTo(damageGE, target);
        
        // 50% 概率触发连击
        if (Random.value < 0.5f)
        {
            TurnBasedBattleManager.Instance.TurnOrderSystem.GrantExtraTurn(Owner);
            Debug.Log($"{Owner.name} 触发连击！");
        }
        
        EndAbility();
    }
}

// 冰冻技能 - 延迟目标行动
public class FreezeAbilitySpec : AbilitySpec<FreezeAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // 施加冰冻 Debuff
        Owner.ApplyGameplayEffectTo(freezeGE, target);
        
        // 延迟目标行动值
        var speedPenalty = target.GetAttributeCurrentValue("AS_Combat", "Speed") ?? 100;
        TurnBasedBattleManager.Instance.TurnOrderSystem.DelayTurn(target, speedPenalty * 0.5f);
        
        EndAbility();
    }
}
```

**为什么这样设计？**
1. **行动值累加机制**：支持不同速度的单位自然交错行动
2. **优先级系统**：处理同速度单位的先后顺序
3. **额外回合标记**：确保连击技能不会无限循环
4. **动态调整**：速度 Buff 可以实时影响行动顺序

---

### 模块 2: **技能冷却与行动点系统**

**为什么需要关注？**
- 回合制不能用时间冷却，需要用回合计数
- 需要支持行动点（AP）、技力（MP）等资源消耗
- 某些技能可能有次数限制（每战斗/每回合）

**设计方案**:

```csharp
public class TurnBasedAbilityCooldownManager
{
    private Dictionary<AbilitySpec, int> cooldowns = new(); // 技能 -> 剩余冷却回合数
    private Dictionary<string, int> usageCount = new();     // 技能名 -> 本回合使用次数
    
    /// <summary>
    /// 检查技能是否在冷却中
    /// </summary>
    public bool IsOnCooldown(AbilitySpec ability)
    {
        return cooldowns.ContainsKey(ability) && cooldowns[ability] > 0;
    }
    
    /// <summary>
    /// 使用技能后设置冷却
    /// </summary>
    public void StartCooldown(AbilitySpec ability, int cooldownTurns)
    {
        if (cooldownTurns > 0)
        {
            cooldowns[ability] = cooldownTurns;
        }
    }
    
    /// <summary>
    /// 回合结束时递减冷却
    /// </summary>
    public void DecrementCooldowns(AbilitySystemComponent unit)
    {
        var abilitiesToUpdate = cooldowns.Keys
            .Where(a => a.Owner == unit)
            .ToList();
        
        foreach (var ability in abilitiesToUpdate)
        {
            cooldowns[ability]--;
            if (cooldowns[ability] <= 0)
            {
                cooldowns.Remove(ability);
            }
        }
    }
    
    /// <summary>
    /// 检查每回合使用次数限制
    /// </summary>
    public bool CanUseThisTurn(string abilityName, int maxUsesPerTurn)
    {
        if (!usageCount.ContainsKey(abilityName))
        {
            usageCount[abilityName] = 0;
        }
        
        return usageCount[abilityName] < maxUsesPerTurn;
    }
    
    /// <summary>
    /// 记录技能使用
    /// </summary>
    public void RecordUsage(string abilityName)
    {
        if (!usageCount.ContainsKey(abilityName))
        {
            usageCount[abilityName] = 0;
        }
        usageCount[abilityName]++;
    }
    
    /// <summary>
    /// 回合开始时重置使用次数
    /// </summary>
    public void ResetTurnUsage()
    {
        usageCount.Clear();
    }
}
```

**GAS 集成：使用 Cost GameplayEffect**

```csharp
// 在 AbilityAsset 中配置消耗
public class ManaAbilityAsset : AbilityAsset
{
    [Header("资源消耗")]
    public GameplayEffect ManaCostEffect; // Instant GE，减少 Mana 属性
    public int CooldownTurns = 3;         // 冷却回合数
    
    [Header("使用限制")]
    public int MaxUsesPerTurn = 1;        // 每回合限用次数
    public int MaxUsesPerBattle = 999;    // 每战斗限用次数
}

// AbilitySpec 中检查和消耗
public class ManaAbilitySpec : AbilitySpec<ManaAbility>
{
    private int battleUsageCount = 0;
    
    public override AbilityActivateResult CanActivate()
    {
        var asset = Ability.DataReference;
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        
        // 1. 检查冷却
        if (cooldownMgr.IsOnCooldown(this))
        {
            return AbilityActivateResult.Fail;
        }
        
        // 2. 检查每回合次数限制
        if (!cooldownMgr.CanUseThisTurn(Ability.Name, asset.MaxUsesPerTurn))
        {
            return AbilityActivateResult.Fail;
        }
        
        // 3. 检查每战斗次数限制
        if (battleUsageCount >= asset.MaxUsesPerBattle)
        {
            return AbilityActivateResult.Fail;
        }
        
        // 4. 检查 Mana 是否足够
        var currentMana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
        var manaCost = GetManaCost(); // 从 MMC 计算
        if (currentMana < manaCost)
        {
            return AbilityActivateResult.Fail;
        }
        
        return base.CanActivate();
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var asset = Ability.DataReference;
        
        // 1. 消耗 Mana（应用 Cost GE）
        Owner.ApplyGameplayEffectTo(asset.ManaCostEffect, Owner);
        
        // 2. 技能效果
        var target = args[0] as AbilitySystemComponent;
        Owner.ApplyGameplayEffectTo(Ability.DataReference.DamageEffect, target);
        
        // 3. 记录使用
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        cooldownMgr.StartCooldown(this, asset.CooldownTurns);
        cooldownMgr.RecordUsage(Ability.Name);
        battleUsageCount++;
        
        EndAbility();
    }
    
    private float GetManaCost()
    {
        // 可以通过 MMC 动态计算消耗
        // 例如：技能等级越高消耗越多
        return 50f;
    }
}
```

**为什么这样设计？**
1. **分离关注点**：冷却、次数限制、资源消耗各自独立管理
2. **利用 GE 系统**：资源消耗用 Cost GE，符合 GAS 理念
3. **可扩展性**：支持减 CD Buff（直接操作 cooldowns 字典）
4. **易于 UI 显示**：cooldowns 字典可直接查询显示剩余回合

**减 CD Buff 示例**:

```csharp
// 减少所有技能冷却 1 回合的 GE
public class ReduceCooldownEffect : GameplayEffect
{
    // 配置为 Instant，通过 GameplayCue 触发逻辑
}

public class ReduceCooldownCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        var target = parameters.Target;
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        
        // 减少所有技能冷却 1 回合
        cooldownMgr.ReduceAllCooldowns(target, 1);
    }
}
```

---

### 模块 3: **反击与触发式技能（被动技能）**

**为什么需要关注？**
- 回合制中常见反击、格挡反击、受击触发等被动机制
- 需要在不是自己回合时也能执行技能
- 需要条件判断系统（何时触发）

**设计方案**:

```csharp
// 被动技能触发器
public enum PassiveTriggerType
{
    OnBeingAttacked,    // 被攻击时
    OnDamageTaken,      // 受到伤害时
    OnDodge,            // 闪避时
    OnBlock,            // 格挡时
    OnCriticalHit,      // 暴击敌人时
    OnAllyDeath,        // 友军死亡时
    OnEnemyDeath,       // 敌人死亡时
    OnTurnStart,        // 回合开始时
    OnTurnEnd,          // 回合结束时
    OnHealthBelow50,    // 血量低于 50% 时
}

public class PassiveAbilitySystem
{
    // 注册被动技能
    private Dictionary<AbilitySystemComponent, List<PassiveAbilityData>> passiveAbilities = new();
    
    public class PassiveAbilityData
    {
        public AbilitySpec Ability;
        public PassiveTriggerType TriggerType;
        public float TriggerChance;    // 触发概率
        public int CooldownTurns;      // 冷却回合数
        public int RemainingCooldown;  // 剩余冷却
        public System.Func<GameplayEventData, bool> Condition; // 额外条件
    }
    
    /// <summary>
    /// 注册被动技能
    /// </summary>
    public void RegisterPassive(AbilitySystemComponent owner, PassiveAbilityData data)
    {
        if (!passiveAbilities.ContainsKey(owner))
        {
            passiveAbilities[owner] = new List<PassiveAbilityData>();
        }
        passiveAbilities[owner].Add(data);
    }
    
    /// <summary>
    /// 触发被动技能
    /// </summary>
    public void TriggerPassives(PassiveTriggerType triggerType, GameplayEventData eventData)
    {
        var owner = eventData.Target; // 触发者
        
        if (!passiveAbilities.ContainsKey(owner)) return;
        
        foreach (var passive in passiveAbilities[owner])
        {
            // 1. 检查触发类型
            if (passive.TriggerType != triggerType) continue;
            
            // 2. 检查冷却
            if (passive.RemainingCooldown > 0) continue;
            
            // 3. 检查概率
            if (Random.value > passive.TriggerChance) continue;
            
            // 4. 检查额外条件
            if (passive.Condition != null && !passive.Condition(eventData)) continue;
            
            // 5. 执行被动技能
            Debug.Log($"{owner.name} 触发被动技能 {passive.Ability.Ability.Name}");
            passive.Ability.TryActivateAbility(eventData.Source);
            
            // 6. 设置冷却
            passive.RemainingCooldown = passive.CooldownTurns;
        }
    }
    
    /// <summary>
    /// 回合结束时递减被动技能冷却
    /// </summary>
    public void DecrementPassiveCooldowns(AbilitySystemComponent owner)
    {
        if (!passiveAbilities.ContainsKey(owner)) return;
        
        foreach (var passive in passiveAbilities[owner])
        {
            if (passive.RemainingCooldown > 0)
            {
                passive.RemainingCooldown--;
            }
        }
    }
}

// 游戏事件数据
public class GameplayEventData
{
    public AbilitySystemComponent Source;  // 攻击者
    public AbilitySystemComponent Target;  // 受击者
    public float Damage;                   // 伤害值
    public bool IsCritical;                // 是否暴击
    public bool IsDodged;                  // 是否闪避
    public bool IsBlocked;                 // 是否格挡
    public GameplayEffect SourceEffect;    // 来源效果
}
```

**GAS 集成：通过 GameplayCue 触发事件**

```csharp
// 伤害 GE 配置 Cue
public class DamageGameplayEffect : GameplayEffect
{
    // 在编辑器中配置 OnAppliedCue
}

// DamageCue 中触发被动检查
public class DamageCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        var eventData = new GameplayEventData
        {
            Source = parameters.Source,
            Target = parameters.Target,
            Damage = parameters.MagnitudeData.Magnitude,
            IsCritical = parameters.MagnitudeData.IsCritical,
            // ...
        };
        
        // 播放受击动画
        PlayHitAnimation(parameters.Target);
        
        // 触发被动技能
        var passiveSystem = TurnBasedBattleManager.Instance.PassiveSystem;
        
        // 受击时触发（如：反击）
        passiveSystem.TriggerPassives(PassiveTriggerType.OnBeingAttacked, eventData);
        
        // 受到伤害时触发（如：荆棘反伤）
        if (eventData.Damage > 0)
        {
            passiveSystem.TriggerPassives(PassiveTriggerType.OnDamageTaken, eventData);
        }
        
        // 闪避时触发
        if (eventData.IsDodged)
        {
            passiveSystem.TriggerPassives(PassiveTriggerType.OnDodge, eventData);
        }
    }
}
```

**被动技能示例**:

```csharp
// 反击技能
public class CounterAttackAbilitySpec : AbilitySpec<CounterAttackAbility>
{
    public override void OnGranted()
    {
        base.OnGranted();
        
        // 注册为被动技能
        var passiveSystem = TurnBasedBattleManager.Instance.PassiveSystem;
        passiveSystem.RegisterPassive(Owner, new PassiveAbilitySystem.PassiveAbilityData
        {
            Ability = this,
            TriggerType = PassiveTriggerType.OnBeingAttacked,
            TriggerChance = 0.5f, // 50% 触发
            CooldownTurns = 0,    // 无冷却
            Condition = (eventData) =>
            {
                // 额外条件：只有物理攻击才反击
                return eventData.SourceEffect.HasTag(GTagLib.Damage_Physical);
            }
        });
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var attacker = args[0] as AbilitySystemComponent;
        
        Debug.Log($"{Owner.name} 反击 {attacker.name}!");
        
        // 造成 50% 攻击力的反击伤害
        Owner.ApplyGameplayEffectTo(counterDamageGE, attacker);
        
        EndAbility();
    }
}

// 荆棘反伤技能
public class ThornsDamageAbilitySpec : AbilitySpec<ThornsDamageAbility>
{
    public override void OnGranted()
    {
        base.OnGranted();
        
        var passiveSystem = TurnBasedBattleManager.Instance.PassiveSystem;
        passiveSystem.RegisterPassive(Owner, new PassiveAbilitySystem.PassiveAbilityData
        {
            Ability = this,
            TriggerType = PassiveTriggerType.OnDamageTaken,
            TriggerChance = 1.0f, // 100% 触发
            CooldownTurns = 0,
            Condition = null // 无额外条件
        });
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var attacker = args[0] as AbilitySystemComponent;
        
        // 反伤 20% 受到的伤害
        var eventData = args[1] as GameplayEventData;
        var reflectDamage = eventData.Damage * 0.2f;
        
        // 使用 SetByCaller 传递伤害值
        var spec = GameplayEffect.CreateSpec(thornsGE, Owner, attacker);
        spec.RegisterValue("Damage", reflectDamage);
        attacker.ApplyGameplayEffectSpec(spec);
        
        EndAbility();
    }
}
```

**为什么这样设计？**
1. **事件驱动**：通过 GameplayCue 系统自然触发被动检查
2. **灵活的条件系统**：支持概率、冷却、自定义条件
3. **符合 GAS 理念**：被动技能也是 Ability，可以应用 GE
4. **易于扩展**：新增触发类型只需添加枚举值

---

### 模块 4: **战斗日志与事件系统**

**为什么需要关注？**
- 回合制需要详细的战斗日志供玩家回顾
- AI 决策需要基于战斗事件
- 战斗回放、成就系统需要事件记录

**设计方案**:

```csharp
// 战斗事件类型
public enum BattleEventType
{
    TurnStart,
    TurnEnd,
    AbilityUsed,
    DamageDealt,
    HealingDone,
    BuffApplied,
    BuffRemoved,
    UnitDeath,
    BattleEnd
}

// 战斗事件
public class BattleEvent
{
    public BattleEventType Type;
    public int TurnNumber;
    public float Timestamp;
    public AbilitySystemComponent Actor;
    public AbilitySystemComponent Target;
    public string AbilityName;
    public float Value; // 伤害/治疗量
    public Dictionary<string, object> CustomData;
}

public class BattleEventLogger
{
    private List<BattleEvent> events = new();
    public IReadOnlyList<BattleEvent> Events => events;
    
    /// <summary>
    /// 记录事件
    /// </summary>
    public void LogEvent(BattleEvent evt)
    {
        events.Add(evt);
        
        // 触发事件监听器
        OnEventLogged?.Invoke(evt);
        
        // 生成战斗日志文本
        GenerateCombatLog(evt);
    }
    
    /// <summary>
    /// 生成战斗日志文本
    /// </summary>
    private void GenerateCombatLog(BattleEvent evt)
    {
        string log = evt.Type switch
        {
            BattleEventType.AbilityUsed => 
                $"[回合 {evt.TurnNumber}] {evt.Actor.name} 使用 {evt.AbilityName} 对 {evt.Target.name}",
            
            BattleEventType.DamageDealt => 
                $"  ⚔️ 造成 {evt.Value:F0} 点伤害",
            
            BattleEventType.HealingDone => 
                $"  💚 恢复 {evt.Value:F0} 点生命",
            
            BattleEventType.BuffApplied => 
                $"  ✨ 施加 {evt.CustomData["BuffName"]} ({evt.CustomData["Duration"]} 回合)",
            
            BattleEventType.UnitDeath => 
                $"[回合 {evt.TurnNumber}] ☠️ {evt.Actor.name} 阵亡",
            
            _ => $"[回合 {evt.TurnNumber}] {evt.Type}"
        };
        
        Debug.Log(log);
        OnCombatLogGenerated?.Invoke(log);
    }
    
    /// <summary>
    /// 获取单位的战斗统计
    /// </summary>
    public BattleStatistics GetStatistics(AbilitySystemComponent unit)
    {
        var stats = new BattleStatistics();
        
        foreach (var evt in events)
        {
            if (evt.Actor == unit)
            {
                if (evt.Type == BattleEventType.DamageDealt)
                    stats.TotalDamageDealt += evt.Value;
                if (evt.Type == BattleEventType.HealingDone)
                    stats.TotalHealingDone += evt.Value;
                if (evt.Type == BattleEventType.AbilityUsed)
                    stats.AbilitiesUsed++;
            }
            
            if (evt.Target == unit && evt.Type == BattleEventType.DamageDealt)
            {
                stats.TotalDamageTaken += evt.Value;
            }
        }
        
        return stats;
    }
    
    public event System.Action<BattleEvent> OnEventLogged;
    public event System.Action<string> OnCombatLogGenerated;
}

public class BattleStatistics
{
    public float TotalDamageDealt;
    public float TotalDamageTaken;
    public float TotalHealingDone;
    public int AbilitiesUsed;
    public int KillCount;
}
```

**GAS 集成：在关键位置记录事件**

```csharp
// 在 TurnBasedBattleManager 中集成
public class TurnBasedBattleManager : MonoBehaviour
{
    public BattleEventLogger EventLogger { get; private set; }
    private int currentTurn = 1;
    
    void StartNextTurn()
    {
        // ... 原有逻辑
        
        // 记录回合开始事件
        EventLogger.LogEvent(new BattleEvent
        {
            Type = BattleEventType.TurnStart,
            TurnNumber = currentTurn,
            Timestamp = Time.time,
            Actor = currentUnit.ASC
        });
    }
    
    public void ExecuteAbility(BattleUnit actor, string abilityName, AbilitySystemComponent target)
    {
        // 记录技能使用事件
        EventLogger.LogEvent(new BattleEvent
        {
            Type = BattleEventType.AbilityUsed,
            TurnNumber = currentTurn,
            Timestamp = Time.time,
            Actor = actor.ASC,
            Target = target,
            AbilityName = abilityName
        });
        
        // ... 原有技能执行逻辑
    }
}

// 在 DamageCue 中记录伤害事件
public class DamageCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        // 记录伤害事件
        TurnBasedBattleManager.Instance.EventLogger.LogEvent(new BattleEvent
        {
            Type = BattleEventType.DamageDealt,
            TurnNumber = TurnBasedBattleManager.Instance.CurrentTurn,
            Timestamp = Time.time,
            Actor = parameters.Source,
            Target = parameters.Target,
            Value = parameters.MagnitudeData.Magnitude
        });
        
        // ... 原有 Cue 逻辑
    }
}
```

**为什么这样设计？**
1. **中心化记录**：所有战斗事件统一记录，易于查询
2. **结构化数据**：事件对象包含完整信息，支持复杂分析
3. **事件订阅**：外部系统可以监听事件（如：成就系统）
4. **战斗回放**：events 列表可以直接用于回放系统

---

### 模块 5: **AI 决策系统**

**为什么需要关注？**
- 回合制 AI 需要评估多个技能选择
- 需要考虑目标选择、技能优先级、资源管理
- 需要适应玩家策略（学习型 AI）

**简化方案（评分系统）**:

```csharp
public class TurnBasedAI
{
    public class AIDecision
    {
        public AbilitySpec Ability;
        public AbilitySystemComponent Target;
        public float Score;
    }
    
    /// <summary>
    /// AI 选择最佳行动
    /// </summary>
    public AIDecision SelectBestAction(AbilitySystemComponent aiUnit)
    {
        var allDecisions = new List<AIDecision>();
        
        // 遍历所有可用技能
        foreach (var ability in aiUnit.AbilityContainer.AbilitySpecs().Values)
        {
            if (ability.CanActivate() != AbilityActivateResult.Success)
                continue;
            
            // 获取潜在目标
            var potentialTargets = GetPotentialTargets(aiUnit, ability);
            
            foreach (var target in potentialTargets)
            {
                // 评分
                float score = EvaluateAction(aiUnit, ability, target);
                
                allDecisions.Add(new AIDecision
                {
                    Ability = ability,
                    Target = target,
                    Score = score
                });
            }
        }
        
        // 返回得分最高的决策
        return allDecisions.OrderByDescending(d => d.Score).FirstOrDefault();
    }
    
    /// <summary>
    /// 评估行动的得分
    /// </summary>
    private float EvaluateAction(AbilitySystemComponent aiUnit, AbilitySpec ability, AbilitySystemComponent target)
    {
        float score = 0;
        
        // 1. 基础伤害评分
        var estimatedDamage = EstimateDamage(ability, target);
        score += estimatedDamage * 1.0f;
        
        // 2. 击杀奖励（优先击杀残血敌人）
        var targetHealth = target.GetAttributeCurrentValue("AS_Combat", "Health") ?? 100;
        if (estimatedDamage >= targetHealth)
        {
            score += 500; // 高额击杀奖励
        }
        
        // 3. 目标威胁度（优先攻击威胁高的敌人）
        var targetThreat = EvaluateThreat(target);
        score += targetThreat * 50;
        
        // 4. 技能效率（AOE 打多个目标得分更高）
        var targetCount = EstimateTargetCount(ability, target);
        score += targetCount * 20;
        
        // 5. Buff/Debuff 价值
        if (ability.Ability.Name.Contains("Buff"))
        {
            score += EvaluateBuffValue(ability, target) * 100;
        }
        
        // 6. 资源保留（Mana 不足时降低高消耗技能得分）
        var manaCost = GetManaCost(ability);
        var currentMana = aiUnit.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
        if (manaCost > currentMana * 0.5f)
        {
            score *= 0.5f; // 惩罚高消耗技能
        }
        
        // 7. 随机性（避免 AI 太死板）
        score += Random.Range(-10f, 10f);
        
        return score;
    }
    
    private float EstimateDamage(AbilitySpec ability, AbilitySystemComponent target)
    {
        // 简化：基于攻击力估算
        var attack = ability.Owner.GetAttributeCurrentValue("AS_Combat", "Attack") ?? 0;
        var defense = target.GetAttributeCurrentValue("AS_Combat", "Defense") ?? 0;
        return Mathf.Max(0, attack - defense * 0.5f);
    }
    
    private float EvaluateThreat(AbilitySystemComponent target)
    {
        // 威胁度 = 攻击力 / 血量（攻高血少 = 威胁大）
        var attack = target.GetAttributeCurrentValue("AS_Combat", "Attack") ?? 0;
        var health = target.GetAttributeCurrentValue("AS_Combat", "Health") ?? 1;
        return attack / health;
    }
    
    private List<AbilitySystemComponent> GetPotentialTargets(AbilitySystemComponent aiUnit, AbilitySpec ability)
    {
        // 简化：根据技能类型返回敌方或友方
        if (ability.Ability.Name.Contains("Heal") || ability.Ability.Name.Contains("Buff"))
        {
            return TurnBasedBattleManager.Instance.GetTeamUnits(aiUnit, sameTeam: true);
        }
        else
        {
            return TurnBasedBattleManager.Instance.GetTeamUnits(aiUnit, sameTeam: false);
        }
    }
}
```

**为什么这样设计？**
1. **评分机制透明**：每个因素的权重可调
2. **易于平衡**：调整评分权重改变 AI 行为
3. **支持多样化**：随机性避免 AI 太机械
4. **可扩展**：新增评分因素只需添加规则

---

### 模块 6: **存档与战斗回放**

**为什么需要关注？**
- 回合制游戏玩家期望随时存档
- 战斗失败后需要重试
- 需要支持战斗回放功能

**设计方案**:

```csharp
[System.Serializable]
public class BattleSaveData
{
    public int TurnNumber;
    public List<UnitSaveData> Units;
    public List<EffectSaveData> ActiveEffects;
    public List<BattleEvent> EventHistory;
    
    [System.Serializable]
    public class UnitSaveData
    {
        public string UnitId;
        public Dictionary<string, float> Attributes; // 属性值
        public List<string> Tags;                   // 当前 Tags
        public List<string> Abilities;              // 拥有的技能
        public Dictionary<string, int> Cooldowns;   // 冷却状态
    }
    
    [System.Serializable]
    public class EffectSaveData
    {
        public string EffectName;
        public string SourceUnitId;
        public string TargetUnitId;
        public int RemainingTurns;
        public int StackCount;
    }
}

public class BattleSaveSystem
{
    /// <summary>
    /// 保存战斗状态
    /// </summary>
    public BattleSaveData SaveBattle()
    {
        var saveData = new BattleSaveData
        {
            TurnNumber = TurnBasedBattleManager.Instance.CurrentTurn,
            Units = new List<BattleSaveData.UnitSaveData>(),
            ActiveEffects = new List<BattleSaveData.EffectSaveData>(),
            EventHistory = new List<BattleEvent>(TurnBasedBattleManager.Instance.EventLogger.Events)
        };
        
        // 保存所有单位状态
        foreach (var unit in TurnBasedBattleManager.Instance.AllUnits)
        {
            var unitData = new BattleSaveData.UnitSaveData
            {
                UnitId = unit.ASC.name,
                Attributes = SaveAttributes(unit.ASC),
                Tags = SaveTags(unit.ASC),
                Abilities = SaveAbilities(unit.ASC),
                Cooldowns = SaveCooldowns(unit.ASC)
            };
            saveData.Units.Add(unitData);
        }
        
        // 保存所有 Buff/Debuff
        var effectMgr = TurnBasedBattleManager.Instance.EffectManager;
        foreach (var (owner, effects) in effectMgr.AllEffects)
        {
            foreach (var effect in effects)
            {
                saveData.ActiveEffects.Add(new BattleSaveData.EffectSaveData
                {
                    EffectName = effect.Spec.GameplayEffect.name,
                    SourceUnitId = effect.Spec.Source.name,
                    TargetUnitId = effect.Spec.Owner.name,
                    RemainingTurns = effect.RemainingTurns,
                    StackCount = effect.Spec.StackCount
                });
            }
        }
        
        return saveData;
    }
    
    /// <summary>
    /// 加载战斗状态
    /// </summary>
    public void LoadBattle(BattleSaveData saveData)
    {
        // 1. 恢复回合数
        TurnBasedBattleManager.Instance.CurrentTurn = saveData.TurnNumber;
        
        // 2. 恢复单位状态
        foreach (var unitData in saveData.Units)
        {
            var unit = TurnBasedBattleManager.Instance.FindUnit(unitData.UnitId);
            LoadAttributes(unit.ASC, unitData.Attributes);
            LoadTags(unit.ASC, unitData.Tags);
            LoadCooldowns(unit.ASC, unitData.Cooldowns);
        }
        
        // 3. 恢复 Buff/Debuff
        var effectMgr = TurnBasedBattleManager.Instance.EffectManager;
        foreach (var effectData in saveData.ActiveEffects)
        {
            var source = TurnBasedBattleManager.Instance.FindUnit(effectData.SourceUnitId).ASC;
            var target = TurnBasedBattleManager.Instance.FindUnit(effectData.TargetUnitId).ASC;
            var effect = LoadEffect(effectData.EffectName);
            
            effectMgr.ApplyEffect(source, target, effect, effectData.RemainingTurns);
            // 设置堆叠数
            // ...
        }
        
        // 4. 恢复事件历史
        TurnBasedBattleManager.Instance.EventLogger.LoadHistory(saveData.EventHistory);
    }
    
    private Dictionary<string, float> SaveAttributes(AbilitySystemComponent asc)
    {
        var attrs = new Dictionary<string, float>();
        // 遍历所有 AttributeSet，保存当前值
        // ...
        return attrs;
    }
}
```

**为什么这样设计？**
1. **完整状态**：保存属性、标签、冷却、Buff 等所有状态
2. **事件重放**：保存事件历史支持战斗回放
3. **序列化友好**：使用 [System.Serializable] 支持 JSON/Binary
4. **易于调试**：存档可以人工查看和修改

---

### 总结：回合制重点模块清单

| 模块 | 核心问题 | GAS 集成方式 | 优先级 |
|------|---------|------------|-------|
| **行动顺序系统** | 速度、插队、延后 | 速度属性 + Tag 加速/减速 | ⭐⭐⭐ 必需 |
| **技能冷却** | 回合计数、次数限制 | Cost GE + 自定义 CooldownManager | ⭐⭐⭐ 必需 |
| **反击/被动** | 触发时机、条件判断 | GameplayCue 事件触发 | ⭐⭐ 常见 |
| **战斗日志** | 事件记录、统计分析 | 在 Cue/Manager 中记录事件 | ⭐⭐ 推荐 |
| **AI 决策** | 技能评分、目标选择 | 读取属性/标签进行评估 | ⭐⭐ 推荐 |
| **存档/回放** | 状态保存、重试机制 | 序列化 ASC 状态 + 事件历史 | ⭐ 可选 |

**设计思路总结**:
1. **优先利用 GAS 现有机制**（Tag、Attribute、GE、Cue）
2. **在 GAS 之上构建回合制逻辑**（TurnOrderSystem、CooldownManager）
3. **通过事件系统解耦模块**（GameplayCue 触发被动、记录日志）
4. **保持数据驱动**（AI 评分权重、技能配置都可在编辑器调整）

---

**遇到疑问时**: 查阅相应的 UE4 GAS 文档（[BillEliot/GASDocumentation_Chinese](https://github.com/BillEliot/GASDocumentation_Chinese)）- 本框架紧密模仿 UE 的实现。

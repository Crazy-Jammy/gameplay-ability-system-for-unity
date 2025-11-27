# 回合制游戏实现路线图

基于 EX-GAS 框架实现复杂回合制游戏的完整 TODO 清单

**目标**：将实时 GAS 框架改造为支持复杂回合制逻辑的系统

**设计原则**：
- 🎯 **最小侵入**：优先扩展而非修改框架核心代码
- 🔄 **高扩展性**：模块化设计，易于添加新机制
- 📊 **数据驱动**：配置化优于硬编码
- 🧩 **解耦合**：各系统独立，通过事件通信

---

## 📋 总览：核心改造点

### ⭐⭐⭐ 必需改造（P0 - 框架基础）
- [ ] **时间系统改造**：暂停 Tick → 回合驱动
- [ ] **Buff 生命周期改造**：Duration/Period 秒 → 回合计数
- [ ] **技能冷却改造**：时间 CD → 回合 CD

### ⭐⭐ 核心系统（P1 - 游戏逻辑）
- [ ] **行动顺序系统**：速度计算、插队、延迟
- [ ] **目标选择系统**：单体、AOE、条件筛选
- [ ] **战斗流程管理器**：回合流转、胜负判定

### ⭐ 高级功能（P2 - 体验增强）
- [ ] **被动技能系统**：反击、触发式技能
- [ ] **战斗事件系统**：日志、统计、回放
- [ ] **AI 决策系统**：技能评分、目标选择

### 🎨 可选扩展（P3 - 额外特性）
- [ ] **连携/组合技系统**：多单位协作技能
- [ ] **地形/站位系统**：前后排、阵型
- [ ] **存档/回放系统**：状态保存、战斗重演
- [ ] **网络同步支持**：指令同步、断线重连

---

## 📝 详细 TODO 清单

---

## P0-1: 时间系统改造 ⭐⭐⭐

**问题**：GAS 默认使用 `GasHost.Update()` 每帧调用 `GAS.Tick()`，所有效果基于 `Time.deltaTime` 计时

**目标**：改为事件驱动，在回合关键节点手动触发更新

### 方案选择

#### 方案 A：暂停全局 Tick + 手动触发（推荐 ⭐）

**实现**：
```csharp
// 游戏初始化
GameplayAbilitySystem.GAS.Pause();

// 在回合关键节点手动 Tick
void OnTurnStart(AbilitySystemComponent unit)
{
    unit.Tick(); // 只 Tick 当前单位
}

void OnAbilityExecuted(AbilitySystemComponent source, AbilitySystemComponent target)
{
    source.Tick();
    target.Tick();
}
```

**优点**：
- ✅ 完全控制执行时机
- ✅ 性能最优（按需执行）
- ✅ 易于调试（精确控制）
- ✅ 不修改框架代码

**缺点**：
- ⚠️ 需要手动管理 Tick 调用点

**扩展性**：
- 支持暂停/恢复（战斗外切换到实时模式）
- 支持慢动作/快进（调整虚拟时间缩放）

**为什么这样设计**：
- 回合制本质是"离散事件"而非"连续时间"
- 手动 Tick 让每个游戏动作都有明确的因果关系
- 避免"半个回合"的中间状态

---

#### 方案 B：虚拟时间控制

**实现**：
```csharp
public class VirtualTimeController
{
    private bool isPaused = true;
    private float timeScale = 1.0f;
    
    void Update()
    {
        if (isPaused) 
        {
            GameplayAbilitySystem.GAS.Pause();
        }
        else
        {
            // 动画/过场播放时恢复 Tick
            GameplayAbilitySystem.GAS.Unpause();
        }
    }
}
```

**优点**：
- ✅ 支持技能动画播放（TimelineAbility）
- ✅ 动画期间 GE 仍在运行

**缺点**：
- ❌ 动画期间时间流逝不可控
- ❌ 需要区分"逻辑时间"和"表现时间"

**适用场景**：需要华丽技能演出的游戏

---

### 实现步骤

- [ ] 1.1 在战斗管理器初始化时调用 `GAS.Pause()`
- [ ] 1.2 在 TurnBasedBattleManager 中标记所有需要 Tick 的关键节点
  - 回合开始
  - 技能执行前
  - 技能执行后
  - 回合结束
- [ ] 1.3 编写测试用例验证 Tick 控制
  - [ ] 测试：暂停状态下 GE 不自动过期
  - [ ] 测试：手动 Tick 后 GE 正常更新

---

## P0-2: Buff 生命周期改造 ⭐⭐⭐

**问题**：GameplayEffect 的 Duration 和 Period 基于秒数，回合制需要回合计数

**目标**：创建回合制效果管理器，用回合数管理 Buff 生命周期

### 方案选择

#### 方案 A：包装器模式（推荐 ⭐）

**实现**：
```csharp
public class TurnBasedEffectData
{
    public GameplayEffectSpec Spec;        // 原始 GE Spec
    public int TotalTurns;                 // 总回合数
    public int RemainingTurns;             // 剩余回合数
    public int TickInterval;               // 每几回合触发一次
    public EffectTimingType Timing;        // OnTurnStart/OnTurnEnd/Immediate
}

public class TurnBasedEffectManager
{
    private Dictionary<ASC, List<TurnBasedEffectData>> effects;
    
    public void ApplyEffect(ASC source, ASC target, GE effect, int turns, EffectTimingType timing);
    public void ProcessTurnStart(ASC unit);
    public void ProcessTurnEnd(ASC unit);
}
```

**优点**：
- ✅ 不修改 GE 核心代码
- ✅ 可以混用原生 GE（用于其他模式）
- ✅ 易于扩展新的计时逻辑

**缺点**：
- ⚠️ 需要额外的数据结构

**扩展性**：
- 支持"真实回合"/"行动次数"等多种计时模式
- 支持 Buff 暂停/恢复（冰冻期间计时停止）

**为什么这样设计**：
- 包装器模式保留原生 GE 的所有功能（堆叠、Tag 过滤等）
- 只扩展生命周期管理，不影响数值计算

---

#### 方案 B：继承 GameplayEffect

**实现**：
```csharp
public class TurnBasedGameplayEffect : GameplayEffect
{
    public int DurationInTurns;
    public int PeriodInTurns;
    public EffectTimingType Timing;
}
```

**优点**：
- ✅ 直接在 GE 中配置回合数
- ✅ 编辑器中可视化

**缺点**：
- ❌ 需要修改框架核心
- ❌ 原生 GE 和回合制 GE 分裂
- ❌ 升级框架时可能冲突

**适用场景**：完全自主维护框架的团队

---

### 设计要点

#### Buff 结算时机设计

```csharp
public enum EffectTimingType
{
    Immediate,     // 立即生效（自身增益 Buff）
    OnTurnStart,   // 回合开始结算（敌方 Debuff）
    OnTurnEnd      // 回合结束结算（持续治疗）
}
```

**为什么需要三种时机**：

1. **Immediate**（立即生效）
   - **用途**：自身施加的增益 Buff
   - **示例**：攻击强化（当前回合就要享受加成）
   - **计数规则**：当前回合立即生效，回合**结束时**计数-1

2. **OnTurnStart**（回合开始结算）
   - **用途**：敌方施加的 Debuff
   - **示例**：中毒（下回合开始第一次受伤）
   - **计数规则**：回合**开始时**先结算效果，再计数-1

3. **OnTurnEnd**（回合结束结算）
   - **用途**：持续回复类效果
   - **示例**：回血术（回合结束时触发）
   - **计数规则**：回合**结束时**先结算效果，再计数-1

**时间线示例**：

```
[玩家回合]
- OnTurnStart: 中毒触发第1次伤害（敌人施加，剩余2回合）
- 玩家行动: 使用攻击强化（Immediate，持续2回合，立即生效）
- OnTurnEnd: 中毒计数变为1，攻击强化计数变为1

[玩家下回合]
- OnTurnStart: 中毒触发第2次伤害（剩余1回合）
- 玩家行动: 攻击强化仍有效
- OnTurnEnd: 中毒计数变为0移除，攻击强化计数变为0移除
```

---

### 实现步骤

- [ ] 2.1 创建 TurnBasedEffectData 数据类
  - [ ] 定义回合计数字段
  - [ ] 定义结算时机枚举
  - [ ] 添加 Period 替代（TickInterval）

- [ ] 2.2 创建 TurnBasedEffectManager 管理器
  - [ ] 实现 ApplyEffect() 包装 GE 应用
  - [ ] 实现 ProcessTurnStart() 处理回合开始效果
  - [ ] 实现 ProcessTurnEnd() 处理回合结束效果
  - [ ] 实现 ExecutePeriodEffect() 处理周期触发

- [ ] 2.3 集成到战斗管理器
  - [ ] 在回合开始调用 ProcessTurnStart()
  - [ ] 在回合结束调用 ProcessTurnEnd()

- [ ] 2.4 编写测试用例
  - [ ] 测试：Immediate Buff 当前回合生效，下回合结束移除
  - [ ] 测试：OnTurnStart Debuff 下回合开始第一次触发
  - [ ] 测试：Period 效果每 N 回合触发一次
  - [ ] 测试：Buff 堆叠正确计数

---

## P0-3: 技能冷却改造 ⭐⭐⭐

**问题**：原生 Ability 没有内置冷却系统，需要自行设计回合制冷却

**目标**：创建回合制冷却管理器，支持回合计数、次数限制、资源消耗

### 方案选择

#### 方案 A：独立冷却管理器（推荐 ⭐）

**实现**：
```csharp
public class TurnBasedAbilityCooldownManager
{
    private Dictionary<AbilitySpec, int> cooldowns;        // 技能 -> 剩余CD回合
    private Dictionary<string, int> turnUsageCount;       // 技能名 -> 本回合使用次数
    private Dictionary<AbilitySpec, int> battleUsageCount; // 技能 -> 战斗总使用次数
    
    public bool IsOnCooldown(AbilitySpec ability);
    public void StartCooldown(AbilitySpec ability, int turns);
    public bool CanUseThisTurn(string abilityName, int maxPerTurn);
    public void DecrementCooldowns(ASC unit);
}
```

**优点**：
- ✅ 集中管理所有冷却状态
- ✅ 易于实现减 CD 效果
- ✅ 易于 UI 查询显示
- ✅ 支持复杂的冷却机制（共享 CD、组 CD）

**扩展性**：
- 支持全局 CD（所有技能共享）
- 支持 CD 组（同组技能共享 CD）
- 支持 CD 回复速度调整（Buff 加速）

**为什么这样设计**：
- 冷却是战斗状态，应由管理器统一管理
- 便于实现"重置所有技能 CD"等全局操作
- 易于序列化保存

---

#### 方案 B：集成到 AbilitySpec

**实现**：
```csharp
public class TurnBasedAbilitySpec : AbilitySpec
{
    private int remainingCooldown;
    private int turnUsageCount;
    
    public override bool CanActivate()
    {
        if (remainingCooldown > 0) return false;
        // ...
    }
}
```

**优点**：
- ✅ 数据和逻辑耦合紧密
- ✅ 不需要额外查询

**缺点**：
- ❌ 难以实现全局 CD 操作
- ❌ 每个 AbilitySpec 需要继承修改
- ❌ UI 查询不方便

---

### 资源消耗设计

#### Cost GameplayEffect 模式（推荐 ⭐）

```csharp
public class ManaAbilityAsset : AbilityAsset
{
    [Header("消耗")]
    public GameplayEffect ManaCostEffect; // Instant GE, Add -50 to Mana
    public int CooldownTurns;
    
    [Header("限制")]
    public int MaxUsesPerTurn = 1;
    public int MaxUsesPerBattle = 999;
}

public class ManaAbilitySpec : AbilitySpec<ManaAbility>
{
    public override bool CanActivate()
    {
        // 检查 Mana 是否足够
        var currentMana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana");
        var manaCost = CalculateManaCost(); // 可用 MMC 计算
        
        if (currentMana < manaCost) return false;
        
        // 检查冷却、次数限制
        // ...
    }
    
    public override void ActivateAbility(params object[] args)
    {
        // 消耗 Mana
        Owner.ApplyGameplayEffectTo(Ability.DataReference.ManaCostEffect, Owner);
        
        // 技能效果
        // ...
    }
}
```

**为什么用 GE 而非直接减属性**：
- ✅ 符合 GAS 理念（只有 GE 修改属性）
- ✅ 支持消耗减免（Buff 降低 Mana 消耗 20%）
- ✅ 便于追踪和调试

---

### 实现步骤

- [ ] 3.1 创建 TurnBasedAbilityCooldownManager
  - [ ] 实现冷却计数字典
  - [ ] 实现回合使用次数字典
  - [ ] 实现战斗使用次数字典

- [ ] 3.2 实现冷却检查逻辑
  - [ ] IsOnCooldown() 查询冷却状态
  - [ ] CanUseThisTurn() 检查次数限制
  - [ ] CanUseBattle() 检查战斗总次数

- [ ] 3.3 实现冷却更新逻辑
  - [ ] StartCooldown() 设置冷却
  - [ ] DecrementCooldowns() 回合结束递减
  - [ ] ResetTurnUsage() 回合开始重置次数

- [ ] 3.4 集成到 AbilitySpec
  - [ ] 在 CanActivate() 中检查冷却
  - [ ] 在 ActivateAbility() 后记录使用

- [ ] 3.5 实现减 CD 机制
  - [ ] ReduceCooldown() 单个技能减 CD
  - [ ] ReduceAllCooldowns() 所有技能减 CD
  - [ ] 通过 GameplayCue 触发

- [ ] 3.6 编写测试用例
  - [ ] 测试：技能使用后正确进入冷却
  - [ ] 测试：回合结束冷却递减
  - [ ] 测试：每回合限用 1 次正确工作
  - [ ] 测试：减 CD Buff 正确生效

---

## P1-1: 行动顺序系统 ⭐⭐

**问题**：需要根据速度、优先级、Buff 等因素决定行动顺序

**目标**：创建灵活的行动顺序系统，支持插队、延迟、额外回合

### 方案选择

#### 方案 A：行动值累加系统（推荐 ⭐）

**实现**：
```csharp
public class TurnOrderSystem
{
    public class TurnEntry
    {
        public ASC Unit;
        public float InitiativeValue; // 行动值（累加）
        public int Priority;          // 优先级
        public bool IsExtraTurn;      // 额外回合标记
    }
    
    private List<TurnEntry> turnQueue;
    
    public ASC GetNextActor()
    {
        var entry = turnQueue[0];
        turnQueue.RemoveAt(0);
        
        if (!entry.IsExtraTurn)
        {
            // 重新加入队列，行动值累加
            entry.InitiativeValue += GetSpeed(entry.Unit);
            turnQueue.Add(entry);
            SortQueue();
        }
        
        return entry.Unit;
    }
}
```

**优点**：
- ✅ 自然模拟速度差异（快 2 倍的单位行动 2 次）
- ✅ 支持动态速度变化（Buff 加速/减速）
- ✅ 支持插队（直接修改 InitiativeValue）

**扩展性**：
- 支持"行动条"UI 显示
- 支持预测未来 N 回合的行动顺序
- 支持回溯（撤销操作）

**为什么这样设计**：
- 模拟真实时间流逝（速度快=行动值增长快）
- 避免"固定轮次"的僵化体验
- 类似《最终幻想战略版》的 CT 系统

---

#### 方案 B：固定轮次系统

**实现**：
```csharp
public class SimpleTurnSystem
{
    private List<ASC> units;
    private int currentIndex;
    
    public ASC GetNextActor()
    {
        var unit = units[currentIndex];
        currentIndex = (currentIndex + 1) % units.Count;
        return unit;
    }
}
```

**优点**：
- ✅ 实现简单
- ✅ 易于理解

**缺点**：
- ❌ 速度属性无意义
- ❌ 不支持插队
- ❌ 体验单调

**适用场景**：简单的卡牌游戏

---

### 特殊机制设计

#### 额外回合设计

```csharp
public void GrantExtraTurn(ASC unit, int priority = 999)
{
    turnQueue.Insert(0, new TurnEntry
    {
        Unit = unit,
        InitiativeValue = float.MaxValue,
        Priority = priority,
        IsExtraTurn = true // 标记为额外回合
    });
}
```

**为什么需要 IsExtraTurn 标记**：
- 防止无限连击（额外回合不累加行动值，用完就移除）
- 区分"正常回合"和"特殊回合"（UI 显示不同）

---

#### 延迟/加速设计

```csharp
public void DelayTurn(ASC unit, float penalty)
{
    var entry = turnQueue.Find(e => e.Unit == unit);
    entry.InitiativeValue -= penalty;
    SortQueue();
}

public void HasteTurn(ASC unit, float bonus)
{
    var entry = turnQueue.Find(e => e.Unit == unit);
    entry.InitiativeValue += bonus;
    SortQueue();
}
```

**示例场景**：
- 冰冻技能：延迟目标行动值（相当于跳过半个回合）
- 加速 Buff：提升行动值（更早行动）

---

### 实现步骤

- [ ] 1.1 创建 TurnOrderSystem 类
  - [ ] 定义 TurnEntry 数据结构
  - [ ] 实现优先级队列（按 InitiativeValue 排序）

- [ ] 1.2 实现行动顺序初始化
  - [ ] 读取单位速度属性
  - [ ] 检查速度 Buff（Tag 加速/减速）
  - [ ] 生成初始队列

- [ ] 1.3 实现获取下一个行动者
  - [ ] GetNextActor() 取队首
  - [ ] 重新计算行动值并入队
  - [ ] 处理额外回合

- [ ] 1.4 实现特殊操作
  - [ ] GrantExtraTurn() 插队
  - [ ] DelayTurn() 延迟
  - [ ] HasteTurn() 加速

- [ ] 1.5 集成到战斗管理器
  - [ ] 战斗开始时初始化顺序
  - [ ] 每回合调用 GetNextActor()

- [ ] 1.6 UI 支持
  - [ ] 显示当前回合单位
  - [ ] 显示未来 5 个行动者预览
  - [ ] 行动条可视化

- [ ] 1.7 编写测试用例
  - [ ] 测试：速度 200 的单位行动 2 次，速度 100 的行动 1 次
  - [ ] 测试：额外回合不影响行动值
  - [ ] 测试：延迟技能正确推后行动顺序

---

## P1-2: 目标选择系统 ⭐⭐

**问题**：需要支持多种目标选择模式（单体、AOE、条件筛选等）

**目标**：扩展 TargetCatcher 系统，创建回合制专用的目标选择器

### 设计原则

**核心理念**：UI 层传主目标，AbilitySpec 内部计算实际目标

**为什么**：
- 职责分离（UI 负责交互，Ability 负责逻辑）
- 支持动态范围（Buff 状态影响技能范围）
- 易于 AI 使用（AI 只需选一个目标）
- 支持技能预览（提前高亮受影响单位）

---

### 方案设计

#### 方案 A：继承 TargetCatcherBase（推荐 ⭐）

**实现**：
```csharp
// 1. 单体目标
public class CatchSingleTarget : TargetCatcherBase
{
    protected override void CatchTargetsNonAlloc(ASC mainTarget, List<ASC> results)
    {
        if (mainTarget != null) results.Add(mainTarget);
    }
}

// 2. 全体敌方
public class CatchAllEnemies : TargetCatcherBase
{
    protected override void CatchTargetsNonAlloc(ASC mainTarget, List<ASC> results)
    {
        var myTeam = BattleManager.GetTeam(Owner);
        foreach (var unit in BattleManager.AllUnits)
        {
            if (BattleManager.GetTeam(unit.ASC) != myTeam && unit.IsAlive)
            {
                results.Add(unit.ASC);
            }
        }
    }
}

// 3. 主目标 + 附近单位（溅射）
public class CatchTargetAndNearby : TargetCatcherBase
{
    public int nearbyCount = 2;
    
    protected override void CatchTargetsNonAlloc(ASC mainTarget, List<ASC> results)
    {
        if (mainTarget == null) return;
        
        results.Add(mainTarget);
        
        var nearby = BattleManager.AllUnits
            .Where(u => u.ASC != mainTarget && u.IsAlive)
            .OrderBy(u => BattleManager.GetDistance(mainTarget, u.ASC))
            .Take(nearbyCount);
        
        foreach (var unit in nearby) results.Add(unit.ASC);
    }
}
```

**优点**：
- ✅ 复用框架现有的 TargetCatcher 架构
- ✅ 易于扩展新的选择模式
- ✅ 支持 GC-free（NonAlloc 版本）

**扩展性**：
- 支持条件筛选（只选血量 <50% 的）
- 支持 Tag 筛选（只选有 Debuff 的）
- 支持位置筛选（前排/后排）

---

#### 方案 B：策略模式 + 配置化

**实现**：
```csharp
[CreateAssetMenu]
public class TargetSelectionConfig : ScriptableObject
{
    public TargetType type; // Single, AllEnemies, Nearby, etc.
    public int range;
    public List<GameplayTag> requiredTags;
    public HealthCondition healthCondition;
}

public class ConfigurableTargetCatcher : TargetCatcherBase
{
    public TargetSelectionConfig config;
    
    protected override void CatchTargetsNonAlloc(ASC mainTarget, List<ASC> results)
    {
        // 根据 config 动态选择
    }
}
```

**优点**：
- ✅ 设计师可在编辑器配置
- ✅ 不需要写代码

**缺点**：
- ❌ 复杂逻辑难以配置化
- ❌ 调试困难

**适用场景**：技能模板化程度高的游戏

---

### 常用目标选择器清单

| 选择器 | 用途 | 实现要点 |
|-------|------|---------|
| CatchSingleTarget | 单体攻击 | 直接返回主目标 |
| CatchAllEnemies | 全体 AOE | 遍历敌方阵营 |
| CatchAllAllies | 群体治疗 | 遍历友方阵营 |
| CatchTargetAndNearby | 溅射伤害 | 主目标 + 距离排序 |
| CatchRandomEnemies | 随机多目标 | 随机抽取 N 个 |
| CatchLowestHealthAlly | 智能治疗 | 血量排序取最低 |
| CatchFrontRow | 前排攻击 | 位置过滤 |
| CatchBackRow | 后排刺杀 | 位置过滤 |
| CatchByTag | 条件筛选 | Tag 过滤 |
| CatchExcludeSelf | 排除自身 | 友方 - 自己 |

---

### 实现步骤

- [ ] 2.1 创建基础目标选择器
  - [ ] CatchSingleTarget
  - [ ] CatchAllEnemies
  - [ ] CatchAllAllies

- [ ] 2.2 创建高级目标选择器
  - [ ] CatchTargetAndNearby（溅射）
  - [ ] CatchRandomEnemies（随机）
  - [ ] CatchLowestHealthAlly（智能治疗）

- [ ] 2.3 创建位置相关选择器
  - [ ] CatchFrontRow
  - [ ] CatchBackRow
  - [ ] CatchByPosition（自定义位置过滤）

- [ ] 2.4 创建条件选择器
  - [ ] CatchByTag（Tag 过滤）
  - [ ] CatchByHealthPercent（血量条件）
  - [ ] CatchByBuffCount（Buff 数量）

- [ ] 2.5 实现目标预览功能
  - [ ] PreviewTargets() 方法
  - [ ] UI 高亮显示

- [ ] 2.6 集成到 AbilitySpec
  - [ ] 在技能内部初始化 TargetCatcher
  - [ ] 在 ActivateAbility() 中使用

- [ ] 2.7 编写测试用例
  - [ ] 测试：单体正确选择主目标
  - [ ] 测试：溅射正确选中主目标 + 附近 2 个
  - [ ] 测试：全体 AOE 选中所有敌人
  - [ ] 测试：条件筛选正确过滤

---

## P1-3: 战斗流程管理器 ⭐⭐

**问题**：需要统一管理回合流转、胜负判定、战斗初始化等

**目标**：创建核心战斗管理器，协调所有系统

### 设计架构

```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    // 核心系统引用
    public TurnOrderSystem TurnOrderSystem { get; private set; }
    public TurnBasedEffectManager EffectManager { get; private set; }
    public TurnBasedAbilityCooldownManager CooldownManager { get; private set; }
    public BattleEventLogger EventLogger { get; private set; }
    
    // 战斗状态
    public List<BattleUnit> AllUnits;
    public int CurrentTurn { get; private set; }
    public BattleState State { get; private set; }
    
    // 核心流程
    public void InitBattle();
    public void StartBattle();
    public void StartNextTurn();
    public void ExecuteAbility(BattleUnit actor, string abilityName, ASC target);
    public void EndCurrentTurn();
    public void OnRoundEnd();
    public void EndBattle(bool victory);
}
```

---

### 回合流程设计

```
[战斗开始]
  ↓
InitBattle()
  - 初始化所有单位 ASC
  - 暂停 GAS.Tick()
  - 初始化行动顺序
  - 注册到 GAS
  ↓
StartBattle()
  - 触发战斗开始事件
  ↓
┌─────────────────┐
│  StartNextTurn() │ ←──────────┐
└─────────────────┘            │
  ↓                            │
获取下一个行动者                 │
  ↓                            │
ProcessTurnStart(unit)          │
  - 处理回合开始 Buff            │
  - 减少技能冷却                │
  - 重置回合使用次数             │
  ↓                            │
检查控制状态                     │
  - 眩晕/冰冻/睡眠？            │
  ↓                            │
[是] → 跳过行动 ─────────────────┤
[否] → 等待行动选择              │
  ↓                            │
执行技能/行动                    │
  ↓                            │
ProcessTurnEnd(unit)            │
  - 处理回合结束 Buff            │
  - 检查胜负                    │
  ↓                            │
[战斗结束] → EndBattle()        │
[继续] ───────────────────────┘
```

---

### 实现步骤

- [ ] 3.1 创建 BattleManager 单例
  - [ ] 初始化所有子系统
  - [ ] 管理战斗状态

- [ ] 3.2 实现战斗初始化
  - [ ] InitBattle() 初始化单位
  - [ ] 暂停 GAS Tick
  - [ ] 初始化行动顺序
  - [ ] 分配队伍/阵营

- [ ] 3.3 实现回合流程
  - [ ] StartNextTurn() 获取下一个行动者
  - [ ] ProcessTurnStart() 回合开始处理
  - [ ] ProcessTurnEnd() 回合结束处理

- [ ] 3.4 实现技能执行
  - [ ] ExecuteAbility() 执行技能
  - [ ] 记录战斗事件
  - [ ] 触发 GameplayCue

- [ ] 3.5 实现胜负判定
  - [ ] CheckBattleEnd() 检查条件
  - [ ] EndBattle() 战斗结算

- [ ] 3.6 实现辅助方法
  - [ ] GetTeam() 查询阵营
  - [ ] GetDistance() 计算距离
  - [ ] IsInFrontRow() 位置判断
  - [ ] FindUnit() 查找单位

- [ ] 3.7 编写测试用例
  - [ ] 测试：完整战斗流程不报错
  - [ ] 测试：胜负判定正确
  - [ ] 测试：控制状态正确跳过回合

---

## P2-1: 被动技能系统 ⭐

**问题**：需要支持反击、触发式技能等在非自己回合执行的技能

**目标**：创建事件驱动的被动技能系统

### 设计方案

#### 触发类型设计

```csharp
public enum PassiveTriggerType
{
    // 战斗事件
    OnBeingAttacked,    // 被攻击时
    OnDamageTaken,      // 受到伤害时
    OnCriticalHit,      // 暴击敌人时
    OnKillEnemy,        // 击杀敌人时
    OnDodge,            // 闪避时
    OnBlock,            // 格挡时
    
    // 回合事件
    OnTurnStart,        // 回合开始时
    OnTurnEnd,          // 回合结束时
    OnRoundStart,       // 轮次开始时
    
    // 状态事件
    OnHealthBelow50,    // 血量 <50%
    OnHealthBelow20,    // 血量 <20%
    OnManaFull,         // 满 Mana
    OnBuffApplied,      // 获得 Buff 时
    OnDebuffApplied,    // 获得 Debuff 时
    
    // 队友事件
    OnAllyDeath,        // 友军死亡时
    OnAllyHurt,         // 友军受伤时
}
```

---

#### 被动数据结构

```csharp
public class PassiveAbilityData
{
    public AbilitySpec Ability;
    public PassiveTriggerType TriggerType;
    public float TriggerChance;          // 触发概率（0-1）
    public int CooldownTurns;            // 冷却回合数
    public int MaxTriggerPerTurn;        // 每回合最大触发次数
    public Func<GameplayEventData, bool> Condition; // 额外条件
}
```

**为什么这样设计**：
- TriggerChance：支持概率触发（50% 反击）
- CooldownTurns：防止频繁触发（反击 CD 2 回合）
- MaxTriggerPerTurn：平衡性（荆棘每回合最多触发 3 次）
- Condition：复杂条件（只反击物理攻击）

---

### 实现步骤

- [ ] 1.1 创建 PassiveAbilitySystem
  - [ ] 定义触发类型枚举
  - [ ] 定义被动数据结构
  - [ ] 实现注册/注销机制

- [ ] 1.2 实现触发逻辑
  - [ ] TriggerPassives() 触发检查
  - [ ] 检查触发类型
  - [ ] 检查冷却
  - [ ] 检查概率
  - [ ] 检查额外条件
  - [ ] 执行被动技能

- [ ] 1.3 集成到 GameplayCue
  - [ ] 在 DamageCue 中触发 OnDamageTaken
  - [ ] 在 HealCue 中触发 OnHealReceived
  - [ ] 在 BuffCue 中触发 OnBuffApplied

- [ ] 1.4 集成到战斗流程
  - [ ] 回合开始触发 OnTurnStart
  - [ ] 回合结束触发 OnTurnEnd
  - [ ] 血量变化触发 OnHealthBelow

- [ ] 1.5 实现常见被动技能
  - [ ] 反击技能
  - [ ] 荆棘反伤
  - [ ] 闪避触发技能
  - [ ] 濒死技能

- [ ] 1.6 编写测试用例
  - [ ] 测试：反击 50% 概率正确触发
  - [ ] 测试：反击 CD 正确工作
  - [ ] 测试：条件过滤正确（只反击物理攻击）

---

## P2-2: 战斗事件系统 ⭐

**问题**：需要记录战斗过程用于日志、统计、回放、成就等

**目标**：创建事件记录系统和统计分析系统

### 实现步骤

- [ ] 2.1 创建事件数据结构
  - [ ] 定义 BattleEventType 枚举
  - [ ] 定义 BattleEvent 类
  - [ ] 定义 GameplayEventData 类

- [ ] 2.2 创建 BattleEventLogger
  - [ ] LogEvent() 记录事件
  - [ ] GenerateCombatLog() 生成文本日志
  - [ ] 事件订阅机制

- [ ] 2.3 集成到战斗流程
  - [ ] 回合开始/结束记录
  - [ ] 技能使用记录
  - [ ] 在 GameplayCue 中记录伤害/治疗

- [ ] 2.4 实现战斗统计
  - [ ] GetStatistics() 获取单位统计
  - [ ] 总伤害、总治疗、击杀数等

- [ ] 2.5 UI 支持
  - [ ] 战斗日志面板
  - [ ] 实时文本显示
  - [ ] 战斗结算统计面板

- [ ] 2.6 编写测试用例
  - [ ] 测试：事件正确记录
  - [ ] 测试：统计数据准确

---

## P2-3: AI 决策系统 ⭐

**问题**：敌方 AI 需要智能选择技能和目标

**目标**：创建基于评分的 AI 决策系统

### 实现步骤

- [ ] 3.1 创建 TurnBasedAI 类
  - [ ] 定义 AIDecision 数据结构
  - [ ] 实现 SelectBestAction()

- [ ] 3.2 实现评分系统
  - [ ] EstimateDamage() 伤害估算
  - [ ] EvaluateThreat() 威胁度计算
  - [ ] EvaluateBuffValue() Buff 价值评估
  - [ ] 击杀奖励
  - [ ] AOE 效率
  - [ ] 资源保留

- [ ] 3.3 实现目标选择
  - [ ] GetPotentialTargets() 获取候选目标
  - [ ] 根据技能类型筛选

- [ ] 3.4 支持多种 AI 策略
  - [ ] 进攻型（优先伤害）
  - [ ] 防御型（优先治疗/Buff）
  - [ ] 平衡型

- [ ] 3.5 编写测试用例
  - [ ] 测试：AI 优先击杀残血
  - [ ] 测试：AI 不浪费 Mana

---

## P3-1: 连携/组合技系统 🎨

**问题**：多个单位协作释放组合技

**目标**：创建连携系统

### 设计方案

#### 方案 A：条件触发型

```csharp
public class ComboAbility : Ability
{
    public List<string> RequiredAbilities; // 需要前置技能
    public int TurnWindow;                 // 触发时间窗口（回合数）
}

public class ComboTracker
{
    private Dictionary<string, int> recentAbilities; // 技能 -> 回合数
    
    public bool CanTriggerCombo(ComboAbility combo)
    {
        foreach (var required in combo.RequiredAbilities)
        {
            if (!recentAbilities.ContainsKey(required)) return false;
            if (recentAbilities[required] > combo.TurnWindow) return false;
        }
        return true;
    }
}
```

**示例**：火球术 + 冰箭术 = 蒸汽爆炸

---

#### 方案 B：队友协作型

```csharp
public class CooperativeAbility : Ability
{
    public List<GameplayTag> RequiredAllyTags; // 队友需要的 Tag
    public int RequiredAllyCount;              // 需要的队友数
}
```

**示例**：3 个剑士同时攻击触发"三连斩"

---

### 实现步骤

- [ ] 1.1 创建 ComboTracker 系统
- [ ] 1.2 记录最近使用的技能
- [ ] 1.3 检查组合触发条件
- [ ] 1.4 UI 提示可用组合技
- [ ] 1.5 编写测试用例

---

## P3-2: 地形/站位系统 🎨

**问题**：需要支持前后排、阵型、地形效果等

**目标**：创建战场位置系统

### 设计方案

```csharp
public class BattlePosition
{
    public int Row;    // 行（0=前排, 1=后排）
    public int Column; // 列
    public TerrainType Terrain; // 地形类型
}

public class FormationSystem
{
    public bool IsInFrontRow(ASC unit);
    public bool IsAdjacent(ASC unit1, ASC unit2);
    public float GetDistancePenalty(ASC attacker, ASC target);
}
```

---

### 实现步骤

- [ ] 2.1 创建 BattlePosition 数据结构
- [ ] 2.2 实现位置管理
- [ ] 2.3 实现地形效果
- [ ] 2.4 集成到目标选择（前排保护）
- [ ] 2.5 UI 显示战场布局
- [ ] 2.6 编写测试用例

---

## P3-3: 存档/回放系统 🎨

**问题**：玩家需要随时存档和回放战斗

**目标**：创建完整的状态序列化系统

### 实现步骤

- [ ] 3.1 创建 BattleSaveData 数据结构
- [ ] 3.2 实现状态保存
  - [ ] SaveAttributes()
  - [ ] SaveTags()
  - [ ] SaveBuffs()
  - [ ] SaveCooldowns()
- [ ] 3.3 实现状态加载
- [ ] 3.4 实现战斗回放
- [ ] 3.5 UI 支持
- [ ] 3.6 编写测试用例

---

## P3-4: 网络同步支持 🎨

**问题**：回合制游戏需要支持 PVP

**目标**：创建指令同步系统

### 设计方案

```csharp
public class BattleCommand
{
    public string PlayerId;
    public int TurnNumber;
    public string AbilityName;
    public string TargetUnitId;
}

public class NetworkBattleManager
{
    public void SendCommand(BattleCommand cmd);
    public void ReceiveCommand(BattleCommand cmd);
    public void ExecuteCommand(BattleCommand cmd);
}
```

**为什么用指令同步而非状态同步**：
- ✅ 数据量小（只传决策）
- ✅ 确定性（双方执行相同逻辑）
- ✅ 易于防作弊（服务端验证）

---

### 实现步骤

- [ ] 4.1 设计指令协议
- [ ] 4.2 实现指令序列化
- [ ] 4.3 实现锁步同步
- [ ] 4.4 实现断线重连
- [ ] 4.5 服务端验证
- [ ] 4.6 编写测试用例

---

## 📊 实现优先级总结

### 第一阶段（必需，2-3 周）
1. ✅ P0-1: 时间系统改造
2. ✅ P0-2: Buff 生命周期改造
3. ✅ P0-3: 技能冷却改造
4. ✅ P1-1: 行动顺序系统
5. ✅ P1-2: 目标选择系统
6. ✅ P1-3: 战斗流程管理器

**里程碑**：可以进行完整的回合制战斗

---

### 第二阶段（推荐，1-2 周）
7. ✅ P2-1: 被动技能系统
8. ✅ P2-2: 战斗事件系统
9. ✅ P2-3: AI 决策系统

**里程碑**：游戏体验完整，AI 可玩

---

### 第三阶段（可选，按需）
10. ⭐ P3-1: 连携/组合技系统
11. ⭐ P3-2: 地形/站位系统
12. ⭐ P3-3: 存档/回放系统
13. ⭐ P3-4: 网络同步支持

**里程碑**：高级特性，提升深度

---

## 🎯 下一步行动

**建议开发顺序**：
1. 先完成 P0-1 时间系统（暂停 Tick）
2. 并行开发 P0-2 和 P0-3（Buff 和冷却）
3. 实现 P1-3 战斗管理器框架
4. 实现 P1-1 和 P1-2（顺序和目标）
5. 测试完整战斗流程
6. 逐步添加 P2、P3 功能

**风险点**：
- ⚠️ GAS Tick 控制需要仔细测试
- ⚠️ Buff 时机需要严格验证
- ⚠️ 冷却系统需要考虑边界情况

**成功指标**：
- ✅ 完整回合制战斗可玩
- ✅ Buff 正确结算
- ✅ 技能冷却正常
- ✅ AI 行为合理

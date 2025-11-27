# P0-1: 时间系统改造详细设计文档

**优先级**：⭐⭐⭐ P0 - 框架基础（必需）

**目标**：将 EX-GAS 框架从实时驱动（每帧 Tick）改造为回合驱动（事件触发）

**影响范围**：
- GameplayAbilitySystem 核心 Tick 循环
- GameplayEffectContainer 的时间计算
- AbilityContainer 的持续技能更新
- GASTimer 时间管理器

---

## 📋 目录

0. [**基础概念：什么是 Tick？**](#基础概念什么是-tick) ← 新手必读！
1. [问题分析](#问题分析)
2. [设计原则](#设计原则)
3. [方案对比](#方案对比)
4. [推荐方案详细设计](#推荐方案详细设计)

---

## 基础概念：什么是 Tick？ 🎯

### Tick 的本质

**Tick = "时间推进一帧"**

想象一下手表的秒针：
- 每滴答一下，时间前进 1 秒
- 每次 Tick，游戏时间前进 `deltaTime` 秒（通常是 0.016 秒，即 1/60 秒）

在 GAS 系统中，**Tick 就是让所有"基于时间的机制"往前走一步**。

---

### 调用一次 Tick 做了什么？

让我们用一个实际例子说明：

**场景**：玩家身上有一个"持续回血 Buff"
- 总持续时间：10 秒
- 每 2 秒回复 50 HP
- 当前已经过了 3.5 秒

**调用一次 `ASC.Tick()` 会发生什么？**

```csharp
public void Tick()  // ← 这是 AbilitySystemComponent 的 Tick 方法
{
    // 假设这次 Tick 的 deltaTime = 0.016 秒（1帧）
    float deltaTime = Time.deltaTime;
    
    // ========== 步骤 1：更新所有 GameplayEffect ==========
    foreach (var ge in ActiveGameplayEffects)
    {
        // 1.1 更新持续时间
        ge.RemainingDuration -= deltaTime;  // 10秒 → 9.984秒
        
        // 1.2 检查是否到期
        if (ge.RemainingDuration <= 0)
        {
            RemoveGameplayEffect(ge);  // 移除过期的 buff
            continue;
        }
        
        // 1.3 更新周期计时器（Period）
        if (ge.HasPeriod)  // 如果有周期效果（如每 2 秒回血）
        {
            ge.PeriodTimer += deltaTime;  // 1.516 → 1.532
            
            // 检查是否到周期触发点
            if (ge.PeriodTimer >= ge.PeriodInterval)  // 1.532 >= 2.0? 否
            {
                // 触发周期效果（回血）
                ApplyPeriodEffect(ge);
                ge.PeriodTimer = 0;  // 重置计时器
            }
        }
    }
    
    // ========== 步骤 2：更新所有技能冷却 ==========
    foreach (var ability in Abilities)
    {
        if (ability.IsOnCooldown)
        {
            ability.CooldownRemaining -= deltaTime;  // 5.0 → 4.984
            
            if (ability.CooldownRemaining <= 0)
            {
                ability.IsOnCooldown = false;  // 冷却结束
            }
        }
    }
    
    // ========== 步骤 3：其他时间相关逻辑 ==========
    // 例如：检查持续施法技能是否完成
    // 例如：更新 AttributeSet 的某些时间相关计算
}
```

**一次 Tick 的结果**：
- ⏱️ 所有 Buff 的剩余时间减少 0.016 秒
- ⏱️ 所有周期计时器增加 0.016 秒（如果到达周期点，触发效果）
- ⏱️ 所有技能冷却减少 0.016 秒
- ⏱️ 检查是否有 Buff 到期（移除）
- ⏱️ 检查是否有技能冷却结束（解锁）

---

### 为什么要"持续调用"Tick？

**在实时游戏中**（如 MOBA、射击游戏）：

```csharp
void Update()  // Unity 的 Update，每帧调用（每秒 60 次）
{
    GameplayAbilitySystem.GAS.Tick();  // ← 每帧都调用
}
```

**为什么要每帧都调用？**

因为游戏是连续进行的，时间在不断流逝！

```
第 1 帧（0.000s）：Buff 剩余 10.000s
第 2 帧（0.016s）：Buff 剩余 9.984s  ← Tick 一次
第 3 帧（0.032s）：Buff 剩余 9.968s  ← Tick 一次
第 4 帧（0.048s）：Buff 剩余 9.952s  ← Tick 一次
...
第 625 帧（10.000s）：Buff 到期，移除  ← Tick 一次
```

如果不持续调用，时间就"冻结"了：

```
第 1 帧（0.000s）：Buff 剩余 10.000s
第 2 帧（0.016s）：没有调用 Tick，Buff 剩余时间还是 10.000s ❌
第 3 帧（0.032s）：没有调用 Tick，Buff 剩余时间还是 10.000s ❌
...
Buff 永远不会过期！
```

---

### 回合制游戏的问题

**问题**：回合制游戏**不是连续进行的**，而是"一段一段"进行的。

```
[玩家回合] → [等待输入...] → [执行技能] → [敌人回合] → [等待 AI...] → [执行技能]
    ↑              ↑               ↑
  需要 Tick      不需要 Tick      需要 Tick
```

**矛盾**：
- ❌ 如果持续调用 Tick（每帧 60 次）：
  - 等待玩家输入时，Buff 持续减少（玩家思考 10 秒，Buff 就过期了）❌
  - CPU 一直在工作（浪费性能）❌

- ❌ 如果不调用 Tick：
  - Buff 永远不会过期 ❌
  - 周期效果不会触发 ❌

**解决方案**：
✅ **暂停自动 Tick，改为"手动 Tick"** —— 只在需要的时候调用一次

---

### "调用一次" vs "持续调用"

| 场景 | 调用方式 | 原因 |
|------|---------|------|
| **实时游戏**（MOBA、FPS） | 持续调用（每帧） | 时间连续流逝 |
| **回合制游戏**（等待输入） | 不调用 | 时间暂停 |
| **回合制游戏**（执行技能） | 调用一次 | 只推进这一刻的逻辑 |
| **回合制游戏**（回合开始） | 调用一次 | 处理回合开始的 Buff 结算 |
| **回合制游戏**（回合结束） | 调用一次 | 处理回合结束的 Buff 结算 |

---

### 手动 Tick 的工作流程（回合制）

**示例：玩家攻击敌人**

```csharp
// 1. 玩家选择攻击技能
void OnPlayerAttack(BattleUnit target)
{
    // 2. 执行技能逻辑
    playerASC.TryActivateAbility("Attack", target.ASC);
    
    // 技能内部会：
    // - 创建伤害 GE
    // - 应用到目标
    // - 触发 Cue
    
    // 3. ========== 关键：手动 Tick 一次 ==========
    // 只调用一次，让刚才应用的 GE 生效
    playerASC.Tick();   // Tick 攻击者（可能有"攻击后回血"的 buff）
    target.ASC.Tick();  // Tick 目标（让伤害 GE 生效）
    
    // 4. 此时伤害已经扣除，可以显示血量变化
    UpdateHealthBar(target);
}
```

**为什么只调用一次？**

因为在这个时刻，我们只需要：
- ✅ 让刚才应用的伤害 GE 生效
- ✅ 检查是否有 Buff 在这个时刻触发
- ✅ 更新属性值

我们**不需要**：
- ❌ 让所有 Buff 的持续时间减少（它们还在"暂停"中）
- ❌ 让周期效果提前触发（它们按回合计数，不是时间）

---

### 实际代码对比

**实时游戏（GAS 原本的设计）**：

```csharp
// Unity 引擎每帧调用（每秒 60 次）
void Update()
{
    float deltaTime = Time.deltaTime;  // 0.016 秒
    
    // 每帧都 Tick，时间持续流逝
    GameplayAbilitySystem.GAS.Tick();
    
    // 结果：
    // - Buff 持续时间每秒减少 1 秒
    // - 周期效果每 2 秒触发一次（自动）
    // - 技能冷却每秒减少 1 秒
}
```

**回合制游戏（手动 Tick）**：

```csharp
// Unity 引擎每帧调用，但我们不 Tick
void Update()
{
    // 什么都不做，GAS 暂停
}

// 只在特定时刻手动 Tick
void OnTurnStart()
{
    Debug.Log("回合开始，处理 Buff");
    
    // ========== 手动 Tick 一次 ==========
    foreach (var unit in allUnits)
    {
        unit.ASC.Tick();  // ← 只调用一次
    }
    
    // 这一次 Tick 做了什么？
    // - 检查 Buff 是否到期（但我们用回合计数，不是时间）
    // - 触发"回合开始"的周期效果（如回血）
    // - 更新属性值
}

void OnPlayerAttack(BattleUnit target)
{
    // 执行攻击...
    
    // ========== 手动 Tick 一次 ==========
    playerASC.Tick();   // ← 只调用一次
    target.ASC.Tick();  // ← 只调用一次
    
    // 这一次 Tick 做了什么？
    // - 让刚才的伤害 GE 生效
    // - 触发被动技能（如反击）
    // - 更新血量
}
```

---

### Tick 的三个关键点总结

1. **Tick 是什么？**
   - 让所有"基于时间的逻辑"往前走一步
   - 更新 Buff 持续时间、周期计时器、技能冷却等

2. **为什么实时游戏要持续 Tick？**
   - 因为时间在连续流逝
   - 每帧推进 0.016 秒，一秒钟 Tick 60 次

3. **为什么回合制只调用一次？**
   - 因为时间是"暂停"的，只在特定时刻"推进一下"
   - 调用一次 = 处理当前时刻的逻辑，然后继续暂停

---

### 通俗比喻 💡

**实时游戏的 Tick**：
> 就像钟表的秒针，滴答滴答不停地走，每滴答一下就是一次 Tick。

**回合制游戏的 Tick**：
> 就像国际象棋的棋钟，只有玩家按下按钮（执行行动）时，时钟才走一格。平时时钟是停止的。

---

### 接下来的内容

现在您理解了 Tick 的本质，接下来我们会讲解：
- 当前 GAS 的 Tick 机制有什么问题（为什么不适合回合制）
- 如何改造（暂停自动 Tick，改为手动控制）
- 何时调用 Tick（7 个关键时机）
- 如何优化性能（批量 Tick、延迟 Tick）

---

### 💡 重要澄清：为什么回合制还要用 Tick？

**您的疑问是对的**：回合制游戏中，Buff 和冷却应该按**回合数**计算，而不是**时间**！

例如：
- ❌ "攻击强化持续 5 秒" → 不合理（玩家思考时间不同）
- ✅ "攻击强化持续 3 回合" → 合理

**那为什么我们还要保留 Tick 机制？**

---

#### 原因 1：Tick 不仅仅是时间计算

**Tick 的本质作用**：

```csharp
public void Tick()
{
    // ========== 1. 时间相关计算（需要改造）==========
    ge.RemainingDuration -= deltaTime;  // ← 这部分要改成回合计数
    
    // ========== 2. 状态更新（必须保留）==========
    UpdateAttributeModifiers();         // ← 重新计算属性修改器
    RefreshAggregatedTags();            // ← 刷新聚合 Tag
    ApplyPendingEffects();              // ← 应用待处理的效果
    
    // ========== 3. 事件检查（必须保留）==========
    CheckEffectExpiration();            // ← 检查 GE 是否到期
    TriggerPeriodicEffects();           // ← 触发周期效果
    UpdateAbilityStates();              // ← 更新技能状态
}
```

**关键点**：即使改成回合计数，**仍然需要一个机制来触发这些更新**，这就是 Tick！

---

#### 原因 2：Tick = "触发更新"，而非"时间流逝"

**Tick 的真正含义**可以理解为两种：

| 含义 | 实时游戏 | 回合制游戏 |
|------|---------|-----------|
| **字面含义** | 时间滴答一下（0.016 秒） | 不适用 ❌ |
| **实际含义** | **执行一次状态更新** | **执行一次状态更新** ✅ |

在回合制中，我们**重新定义 Tick**：
- ❌ ~~Tick = 时间前进 deltaTime~~
- ✅ **Tick = 执行一次 GAS 系统的状态更新**

---

#### 原因 3：最小侵入原则

**如果不用 Tick，需要做什么？**

**方案 A**：完全重写 GAS 核心（不用 Tick）
```csharp
// 需要修改框架核心代码
public class GameplayEffectContainer
{
    // 重写所有时间相关逻辑
    public void UpdateEffectsByTurnCount() { ... }  // 新方法
    public void ApplyAttributeModifiers() { ... }   // 新方法
    public void CheckExpiration() { ... }           // 新方法
    // ... 可能影响上百处代码
}
```

❌ **问题**：
- 工作量巨大（可能需要 2-4 周）
- 破坏框架设计，失去官方支持
- 难以升级到新版本

---

**方案 B**：保留 Tick，改造其行为（推荐）
```csharp
// 只需要在外部控制 Tick 的调用时机
public class TurnBasedBattleManager
{
    void OnTurnStart()
    {
        // 手动触发 Tick，执行状态更新
        foreach (var unit in allUnits)
        {
            unit.ASC.Tick();  // ← 复用框架的 Tick 机制
        }
    }
}

// Tick 内部的时间计算由 TurnBasedEffectManager 接管
public class TurnBasedEffectManager
{
    void OnTurnEnd()
    {
        // 用回合计数替代时间
        effectData.RemainingTurns--;  // ← 外部管理回合数
        
        if (effectData.RemainingTurns <= 0)
        {
            asc.RemoveGameplayEffect(effectData.Spec);  // ← 调用框架的移除方法
        }
    }
}
```

✅ **优点**：
- 零框架修改（最小侵入）
- 工作量小（只需 1-2 天）
- 保持官方支持，可升级

---

#### 原因 4：Tick 触发的关键功能仍然需要

**即使改成回合制，以下功能仍然需要 Tick 触发**：

**1. 属性修改器重新计算**

```csharp
// 场景：玩家施加了"攻击力 +50"的 Buff
playerASC.ApplyGameplayEffectTo(attackBoostGE, playerASC);

// 问题：此时属性还没更新！
var attack = playerASC.GetAttributeCurrentValue("AS_Combat", "Attack");
Debug.Log(attack);  // 输出：100（原始值）❌

// 解决：调用 Tick，触发属性重新计算
playerASC.Tick();

var attack = playerASC.GetAttributeCurrentValue("AS_Combat", "Attack");
Debug.Log(attack);  // 输出：150（100 + 50）✅
```

**为什么？** 因为 GAS 的属性修改器是**延迟计算**的，需要 Tick 触发。

---

**2. Tag 聚合更新**

```csharp
// 场景：施加"眩晕"GE（带 State.Control.Stun Tag）
enemyASC.ApplyGameplayEffectTo(stunGE, enemyASC);

// 问题：此时 Tag 还没生效！
bool isStunned = enemyASC.HasTag(GTagLib.State_Control_Stun);
Debug.Log(isStunned);  // 输出：false ❌

// 解决：调用 Tick，触发 Tag 聚合更新
enemyASC.Tick();

bool isStunned = enemyASC.HasTag(GTagLib.State_Control_Stun);
Debug.Log(isStunned);  // 输出：true ✅
```

---

**3. 周期效果触发**

```csharp
// 场景：持续回血 Buff（每回合回复 50 HP）
// 在回合开始时触发

void OnTurnStart()
{
    // 调用 Tick，检查并触发周期效果
    playerASC.Tick();  // ← 内部会检查哪些 GE 需要触发周期效果
}
```

虽然我们用**回合计数**管理持续时间，但**触发周期效果的逻辑**仍然在 Tick 中。

---

**4. 待处理效果的应用**

```csharp
// 某些 GE 可能被延迟应用
public class DelayedDamageGE : GameplayEffect
{
    // 标记为"下次 Tick 时应用"
}

// 调用 Tick 时统一处理
asc.Tick();  // ← 应用所有待处理的效果
```

---

#### 改造策略：分离"时间"和"触发"

**核心思路**：
1. **时间管理**：从 GAS 的 Tick 中剥离，交给 `TurnBasedEffectManager` 管理回合计数
2. **触发机制**：保留 Tick，用于触发状态更新

**改造前**（原 GAS）：
```
Tick() {
    ├─ 时间计算（Duration -= deltaTime）     ← 不适合回合制
    ├─ 属性更新（重新计算修改器）             ← 仍需要
    ├─ Tag 聚合（刷新聚合 Tag）                ← 仍需要
    ├─ 周期触发（检查 Period 计时器）          ← 仍需要
    └─ 过期检查（移除到期 GE）                 ← 仍需要
}
```

**改造后**（回合制）：
```
Tick() {
    ├─ 时间计算 → 跳过（TurnBasedEffectManager 已处理）
    ├─ 属性更新（重新计算修改器）             ← 保留
    ├─ Tag 聚合（刷新聚合 Tag）                ← 保留
    ├─ 周期触发（检查是否到触发点）           ← 保留
    └─ 过期检查（移除到期 GE）                 ← 保留
}
```

**外部 TurnBasedEffectManager**：
```csharp
void OnTurnEnd()
{
    // 手动管理回合计数
    effectData.RemainingTurns--;
    
    if (effectData.RemainingTurns <= 0)
    {
        // 标记为过期（下次 Tick 时移除）
        effectData.Spec.Owner.RemoveGameplayEffect(effectData.Spec);
    }
}
```

---

#### 实际工作流程对比

**改造前（实时游戏）**：
```
每帧（0.016 秒）：
  ↓
调用 Tick()
  ├─ Duration -= 0.016
  ├─ PeriodTimer += 0.016
  ├─ 检查是否到期
  └─ 更新属性/Tag
```

**改造后（回合制）**：
```
回合开始：
  ↓
TurnBasedEffectManager:
  ├─ RemainingTurns--（回合计数）
  ├─ 检查是否到期
  └─ 标记需要触发的周期效果
  ↓
调用 Tick()（状态更新）:
  ├─ 跳过时间计算（已由外部处理）
  ├─ 触发标记的周期效果
  ├─ 更新属性/Tag
  └─ 移除过期 GE
```

**关键区别**：
- **时间管理**：从 Tick 内部移到外部（TurnBasedEffectManager）
- **Tick 作用**：从"时间推进"变为"状态更新触发器"

---

#### 总结：为什么回合制还用 Tick？

| 原因 | 说明 |
|------|------|
| **1. Tick ≠ 时间** | Tick 的本质是"触发状态更新"，而非"时间流逝" |
| **2. 最小侵入** | 复用框架机制，避免重写核心代码 |
| **3. 功能依赖** | 属性计算、Tag 聚合、周期触发等都需要 Tick |
| **4. 分离关注点** | 时间管理外置（TurnBasedEffectManager），触发机制内置（Tick） |

**类比**：
- **Tick** 就像 Unity 的 `Update()` 方法
- 实时游戏：每帧自动调用（60 FPS）
- 回合制游戏：手动调用（按需触发）

**最终方案**：
- ✅ 保留 Tick 作为"状态更新触发器"
- ✅ 暂停自动 Tick（`GAS.Pause()`）
- ✅ 在关键时刻手动调用 Tick
- ✅ 用 TurnBasedEffectManager 管理回合计数

这样既保持了框架的完整性，又实现了回合制的需求！🎯

---


5. [实现步骤](#实现步骤)
6. [测试方案](#测试方案)
7. [风险与应对](#风险与应对)

---

## 问题分析

### 当前框架的时间机制

#### 1. 核心 Tick 循环

**源码分析**（`Assets/GAS/Runtime/Core/GameplayAbilitySystem.cs`）：

```csharp
public class GameplayAbilitySystem : MonoBehaviour
{
    private static GameplayAbilitySystem instance;
    private List<AbilitySystemComponent> registeredASCs = new();
    
    void Update()
    {
        if (!isPaused)
        {
            GASTimer.UpdateCurrentFrameCount();
            Tick();
        }
    }
    
    private void Tick()
    {
        foreach (var asc in registeredASCs)
        {
            asc.Tick(); // 每帧调用所有 ASC
        }
    }
}
```

**问题**：
- ❌ **每帧都执行**：回合制中大部分时间处于等待状态，无意义的 CPU 消耗
- ❌ **基于帧率**：`Time.deltaTime` 计算 GE 持续时间，回合制不应依赖真实时间
- ❌ **难以控制**：无法精确控制在某个逻辑点执行

---

#### 2. GameplayEffect 时间计算

**源码分析**（`Assets/GAS/Runtime/Effects/GameplayEffectContainer.cs`）：

```csharp
public class GameplayEffectPeriodTicker
{
    private float periodTimer;
    
    public void Tick(float deltaTime)
    {
        periodTimer += deltaTime;
        
        if (periodTimer >= period)
        {
            ExecutePeriodEffect();
            periodTimer -= period;
        }
    }
}

public class GameplayEffectDurationHandler
{
    private float remainingDuration;
    
    public void Tick(float deltaTime)
    {
        remainingDuration -= deltaTime;
        
        if (remainingDuration <= 0)
        {
            RemoveEffect();
        }
    }
}
```

**问题**：
- ❌ **基于秒数**：`period` 和 `duration` 使用浮点秒数，回合制需要整数回合
- ❌ **累加误差**：浮点数累加可能导致时间漂移
- ❌ **难以同步**：网络回合制游戏需要精确的整数计数

---

#### 3. AbilityContainer 的 Tick

**源码分析**（`Assets/GAS/Runtime/Ability/AbilityContainer.cs`）：

```csharp
public void Tick()
{
    foreach (var spec in runningAbilitySpecs)
    {
        spec.Tick(); // 持续技能每帧更新
    }
}
```

**问题**：
- ❌ **持续技能**：TimelineAbility 依赖 Tick 播放动画，回合制中不适用
- ❌ **状态更新**：一些技能可能在 Tick 中检查条件，需要改为事件驱动

---

### 回合制需求分析

| 需求 | 当前实现 | 期望实现 |
|------|---------|---------|
| **时间驱动** | 每帧自动 Tick | 事件手动触发 |
| **GE 持续时间** | 浮点秒数 | 整数回合数 |
| **Period 周期** | 秒数间隔 | 回合数间隔 |
| **性能** | 每帧运行所有 ASC | 按需运行单个 ASC |
| **确定性** | 依赖帧率 | 完全确定（整数计数） |
| **网络同步** | 时间同步困难 | 回合同步简单 |

---

## 设计原则

### 核心原则

1. **🎯 最小侵入**
   - 优先使用框架提供的 API（Pause/Unpause）
   - 避免修改框架核心代码
   - 保持与原框架的兼容性

2. **🔄 事件驱动**
   - 从"时间流逝触发更新"改为"游戏事件触发更新"
   - 明确的触发点（回合开始、技能执行、回合结束）
   - 可追踪的执行流程

3. **📊 确定性优先**
   - 使用整数计数替代浮点时间
   - 相同输入必定产生相同输出
   - 便于网络同步和回放

4. **⚡ 性能优化**
   - 按需执行（只 Tick 必要的 ASC）
   - 避免每帧开销
   - 减少内存分配

---

## 方案对比

### 🎯 推荐方案总览

基于明确的需求（**带动画演出的回合制** + **SmartTickManager 自动化**），本文档推荐：

**方案 A（SmartTickManager 自动触发） + 分层时间控制（动画支持）**

核心特点：
- ✅ **99% 自动化**：SmartTickManager 在 7 个关键点自动触发 Tick
- ✅ **支持动画**：分层时间控制，Animator 始终运行，GAS 按需启用
- ✅ **零手动调用**：开发者无需关心 Tick，专注业务逻辑
- ✅ **华丽演出**：支持待机、技能、大招等各种动画
- ✅ **性能优秀**：批量 Tick、智能去重、按需执行

详见以下章节：
- [方案 A：SmartTickManager 自动触发](#方案-a智能自动触发推荐-⭐⭐⭐)（核心机制）
- [特殊场景：带动画演出的回合制游戏](#特殊场景带动画演出的回合制游戏-🎬)（动画集成）

---

### 方案 A：智能自动触发（推荐 ⭐⭐⭐）

#### 核心思路

利用框架提供的 `Pause()/Unpause()` API，全局暂停自动 Tick。通过 **SmartTickManager** 在 7 个关键节点**自动触发** Tick，开发者无需手动调用。

#### 实现概览

```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    private SmartTickManager tickManager;
    
    void Start()
    {
        // 1. 初始化时暂停全局 Tick
        GameplayAbilitySystem.GAS.Pause();
        
        // 2. 创建 SmartTickManager
        tickManager = new SmartTickManager();
    }
    
    void OnTurnStart(AbilitySystemComponent unit)
    {
        // 自动触发点 1：回合开始
        tickManager.AutoFlush(TickTriggerPoint.TurnStart);
        // 开发者无需手动调用 Tick！
    }
    
    void OnAbilityExecuted(AbilitySystemComponent source, AbilitySystemComponent target)
    {
        // 自动触发点 2：技能执行后
        tickManager.AutoFlush(TickTriggerPoint.AfterAbility);
        // SmartTickManager 已自动收集 source 和 target！
    }
    
    void OnTurnEnd(AbilitySystemComponent unit)
    {
        // 自动触发点 3：回合结束
        tickManager.AutoFlush(TickTriggerPoint.TurnEnd);
    }
}
```

---

#### 详细设计

**1. SmartTickManager 完整实现**

```csharp
public class SmartTickManager
{
    // 待更新的 ASC 集合（HashSet 自动去重）
    private HashSet<AbilitySystemComponent> dirtyASCs = new();
    
    // 性能监控
    private Dictionary<TickTriggerPoint, int> tickCountStats = new();
    
    /// <summary>
    /// 标记 ASC 需要更新（由系统自动调用，开发者无需关心）
    /// </summary>
    public void MarkDirty(AbilitySystemComponent asc, string reason = "")
    {
        if (asc == null) return;
        
        dirtyASCs.Add(asc);
        
        #if UNITY_EDITOR
        Debug.Log($"[MarkDirty] {asc.name} - Reason: {reason}");
        #endif
    }
    
    /// <summary>
    /// 自动刷新（在 7 个触发点自动调用）
    /// </summary>
    public void AutoFlush(TickTriggerPoint triggerPoint)
    {
        if (dirtyASCs.Count == 0) return;
        
        Debug.Log($"[AutoFlush] {triggerPoint} - Flushing {dirtyASCs.Count} ASCs");
        
        // 批量 Tick
        foreach (var asc in dirtyASCs)
        {
            asc.Tick();
        }
        
        // 统计
        if (!tickCountStats.ContainsKey(triggerPoint))
            tickCountStats[triggerPoint] = 0;
        tickCountStats[triggerPoint] += dirtyASCs.Count;
        
        // 清空
        dirtyASCs.Clear();
    }
    
    /// <summary>
    /// 立即 Tick（1% 的边缘场景手动使用）
    /// </summary>
    public void TickImmediate(AbilitySystemComponent asc, string reason = "Manual")
    {
        Debug.Log($"[TickImmediate] {asc.name} - Reason: {reason}");
        asc.Tick();
    }
    
    /// <summary>
    /// 批量 Tick（初始化等场景）
    /// </summary>
    public void TickAll(IEnumerable<AbilitySystemComponent> ascs, TickTriggerPoint triggerPoint)
    {
        int count = 0;
        foreach (var asc in ascs)
        {
            if (asc != null)
            {
                asc.Tick();
                count++;
            }
        }
        
        Debug.Log($"[TickAll] {triggerPoint} - Ticked {count} ASCs");
    }
    
    /// <summary>
    /// 获取性能统计
    /// </summary>
    public void PrintStats()
    {
        Debug.Log("=== SmartTickManager 统计 ===");
        foreach (var (point, count) in tickCountStats)
        {
            Debug.Log($"{point}: {count} 次 Tick");
        }
    }
}
```

**关键特性**：
- ✅ **HashSet 去重**：同一个 ASC 在一个触发点内只 Tick 一次
- ✅ **性能监控**：统计每个触发点的 Tick 次数
- ✅ **调试友好**：详细的日志输出（编辑器模式）
- ✅ **支持手动**：保留 TickImmediate 用于 1% 的边缘场景

---

**2. 7 个自动触发点**

```csharp
public enum TickTriggerPoint
{
    BattleStart,      // 战斗开始
    TurnStart,        // 回合开始（处理回合开始 Buff）
    BeforeAbility,    // 技能执行前（检查前置条件）
    AfterAbility,     // 技能执行后（应用效果）
    TurnEnd,          // 回合结束（处理回合结束 Buff）
    RoundEnd,         // 轮次结束（所有单位都行动过）
    BattleEnd         // 战斗结束
}
```

**为什么需要这些触发点**：
- **BattleStart**：初始化战场 Buff（地形效果、开场 Buff）
- **TurnStart**：处理敌方施加的 Debuff（中毒等）
- **BeforeAbility**：检查技能是否可用（冷却、资源）
- **AfterAbility**：确保效果立即生效（伤害结算、Buff 应用）⭐ **最重要（80% 覆盖率）**
- **TurnEnd**：处理自身 Buff 计数、持续治疗
- **RoundEnd**：触发"每轮一次"的效果
- **BattleEnd**：清理战场状态

**💡 关键设计**：开发者只需在这 7 个地方调用 `AutoFlush()`，无需关心哪些 ASC 需要 Tick！

---

**3. GameplayCue 自动 MarkDirty 集成**

**问题**：技能释放后，source 和 target 的 ASC 需要 Tick，如何自动收集？

**解决方案**：在 GameplayCue 中自动 MarkDirty

```csharp
// 伤害 Cue（技能造成伤害时触发）
public class DamageCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        var source = parameters.Source;
        var target = parameters.Target;
        
        // 1. 播放受击特效
        PlayHitVFX(target);
        
        // 2. 自动标记需要更新的 ASC
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        tickManager.MarkDirty(source, "造成伤害");
        tickManager.MarkDirty(target, "受到伤害");
        
        // 3. 触发被动技能检查
        var passiveSystem = TurnBasedBattleManager.Instance.PassiveSystem;
        passiveSystem.TriggerPassives(PassiveTriggerType.OnDamageTaken, new GameplayEventData
        {
            Source = source,
            Target = target,
            Damage = parameters.MagnitudeData.Magnitude
        });
    }
}

// Buff 应用 Cue
public class BuffAppliedCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        var target = parameters.Target;
        
        // 播放 Buff 特效
        PlayBuffVFX(target);
        
        // 自动标记
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        tickManager.MarkDirty(target, "Buff 应用");
    }
}
```

**为什么这样设计**：
- ✅ **自动收集**：每次应用 GE，自动标记相关 ASC
- ✅ **零遗漏**：所有受影响的单位都会被收集
- ✅ **解耦合**：业务逻辑无需知道 Tick 的存在

---

**4. 被动技能自动 MarkDirty**

```csharp
public class PassiveAbilitySystem
{
    private SmartTickManager tickManager;
    
    public void TriggerPassives(PassiveTriggerType triggerType, GameplayEventData eventData)
    {
        var owner = eventData.Target; // 触发者
        
        if (!passiveAbilities.ContainsKey(owner)) return;
        
        foreach (var passive in passiveAbilities[owner])
        {
            // 检查触发条件...
            if (ShouldTrigger(passive, triggerType, eventData))
            {
                // 执行被动技能
                passive.Ability.TryActivateAbility(eventData.Source);
                
                // 自动标记相关 ASC
                tickManager.MarkDirty(owner, $"被动技能触发: {passive.Ability.Ability.Name}");
                tickManager.MarkDirty(eventData.Source, "被动技能目标");
            }
        }
    }
}
```

**为什么这样设计**：
- ✅ **连锁反应**：反击技能触发后，自动标记攻击者和反击者
- ✅ **无需手动**：被动系统自己负责标记，业务逻辑无感知

---

**5. 战斗流程完整集成**

```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    public SmartTickManager TickManager { get; private set; }
    public PassiveAbilitySystem PassiveSystem { get; private set; }
    
    void InitBattle()
    {
        // 1. 暂停自动 Tick
        GameplayAbilitySystem.GAS.Pause();
        
        // 2. 创建管理器
        TickManager = new SmartTickManager();
        PassiveSystem = new PassiveAbilitySystem(TickManager);
        
        // 3. 初始化所有单位
        foreach (var unit in AllUnits)
        {
            unit.ASC.InitWithPreset(1, unit.ASC.Preset);
            GameplayAbilitySystem.GAS.Register(unit.ASC);
        }
        
        // 4. 触发点 1：战斗开始
        TickManager.TickAll(AllUnits.Select(u => u.ASC), TickTriggerPoint.BattleStart);
    }
    
    void StartNextTurn()
    {
        var currentUnit = TurnOrderSystem.GetNextActor();
        
        Debug.Log($">>> {currentUnit.ASC.name} 的回合开始");
        
        // 触发点 2：回合开始
        TickManager.MarkDirty(currentUnit, "回合开始");
        TickManager.AutoFlush(TickTriggerPoint.TurnStart);
        
        // 处理回合开始的 Buff（TurnBasedEffectManager 管理）
        effectManager.ProcessTurnStart(currentUnit);
        
        // 检查控制状态
        if (IsControlled(currentUnit))
        {
            Debug.Log($"{currentUnit.name} 被控制，跳过回合");
            EndCurrentTurn();
            return;
        }
        
        // 等待行动选择
        if (currentUnit.Team == 0) // 玩家方
        {
            ShowPlayerActionMenu(currentUnit);
        }
        else // 敌方
        {
            StartCoroutine(AISelectAction(currentUnit));
        }
    }
    
    public void ExecuteAbility(BattleUnit actor, string abilityName, AbilitySystemComponent target)
    {
        Debug.Log($"{actor.ASC.name} 使用 {abilityName}");
        
        // 可选：触发点 3：技能执行前（检查状态）
        // TickManager.MarkDirty(actor.ASC, "技能执行前");
        // TickManager.AutoFlush(TickTriggerPoint.BeforeAbility);
        
        // 执行技能（内部会应用 GE，触发 Cue，Cue 会自动 MarkDirty）
        bool success = actor.ASC.TryActivateAbility(abilityName, target);
        
        if (success)
        {
            // 触发点 4：技能执行后（最重要！80% 的 Tick 发生在这里）
            TickManager.AutoFlush(TickTriggerPoint.AfterAbility);
            
            // 检查战斗结束
            if (CheckBattleEnd())
            {
                return;
            }
        }
        
        EndCurrentTurn();
    }
    
    void EndCurrentTurn()
    {
        var currentUnit = AllUnits[currentUnitIndex];
        
        Debug.Log($"<<< {currentUnit.ASC.name} 的回合结束");
        
        // 处理回合结束的 Buff（计数递减）
        effectManager.ProcessTurnEnd(currentUnit.ASC);
        
        // 触发点 5：回合结束
        TickManager.MarkDirty(currentUnit.ASC, "回合结束");
        TickManager.AutoFlush(TickTriggerPoint.TurnEnd);
        
        // 下一个单位
        currentUnitIndex++;
        
        // 延迟开始下一回合
        StartCoroutine(DelayedNextTurn(0.5f));
    }
    
    void OnRoundEnd()
    {
        Debug.Log("=== 回合轮次结束 ===");
        
        // 触发点 6：轮次结束
        TickManager.TickAll(AllUnits.Select(u => u.ASC), TickTriggerPoint.RoundEnd);
    }
    
    void OnBattleEnd(bool victory)
    {
        Debug.Log(victory ? "=== 战斗胜利 ===" : "=== 战斗失败 ===");
        
        // 触发点 7：战斗结束
        TickManager.TickAll(AllUnits.Select(u => u.ASC), TickTriggerPoint.BattleEnd);
        
        // 清理所有效果
        effectManager.ClearAllEffects();
    }
}
```

**完整工作流程**：

```
战斗开始
  ↓
[BattleStart] TickAll(所有单位)
  ↓
┌─────────── 回合循环 ───────────┐
│                                │
│ 回合开始                        │
│   ↓                            │
│ [TurnStart] MarkDirty(当前单位) │
│   ↓                            │
│ AutoFlush()                    │
│   ↓                            │
│ 玩家/AI 选择技能                │
│   ↓                            │
│ [可选 BeforeAbility]            │
│   ↓                            │
│ 执行技能                        │
│   ├─ ApplyGE                   │
│   ├─ Cue.Trigger()             │
│   │    └─ MarkDirty(source)    │
│   │    └─ MarkDirty(target)    │
│   ├─ PassiveTrigger()          │
│   │    └─ MarkDirty(反击者)     │
│   ↓                            │
│ [AfterAbility] AutoFlush()     │ ← 最重要！
│   ↓                            │
│ 回合结束                        │
│   ↓                            │
│ [TurnEnd] MarkDirty(当前单位)   │
│   ↓                            │
│ AutoFlush()                    │
│   ↓                            │
│ 下一个单位                      │
│                                │
└─────────── 循环 ──────────────┘
  ↓
所有单位都行动过
  ↓
[RoundEnd] TickAll(所有单位)
  ↓
继续下一轮或结束战斗
  ↓
[BattleEnd] TickAll(所有单位)
```

**关键点**：
- ✅ **开发者无需手动 Tick**：只需在 7 个地方调用 AutoFlush()
- ✅ **Cue 自动收集**：每次应用 GE，Cue 自动 MarkDirty
- ✅ **被动技能自动收集**：触发反击等被动时，自动 MarkDirty
- ✅ **智能去重**：HashSet 确保同一 ASC 只 Tick 一次
- ✅ **99% 覆盖**：绝大多数场景都自动处理
---

#### 优点分析

✅ **1. 99% 自动化，零心智负担**
- SmartTickManager 在 7 个关键点自动触发
- Cue 系统自动收集受影响的 ASC
- 被动技能自动标记相关单位
- **开发者完全无需关心 Tick**

✅ **2. 零侵入框架核心**
- 使用官方 API（`Pause()/Unpause()`）
- 不修改 GameplayAbilitySystem 源码
- 框架升级时无冲突

✅ **3. 完全控制执行时机**
- 明确知道每个 Tick 发生的时机（7 个触发点）
- 易于调试（详细日志）
- 可以在 AutoFlush 前后插入逻辑

✅ **4. 性能最优**
- 只在必要时 Tick（事件驱动）
- 避免每帧开销（60FPS → 0FPS）
- HashSet 智能去重（同一 ASC 只 Tick 一次）
- 批量执行（TickAll 优化）

✅ **5. 易于扩展**
- 可以添加新的触发点
- 支持条件 Tick（只 Tick 有 Buff 的单位）
- 支持性能监控（PrintStats）

✅ **6. 网络友好**
- 确定性执行（不依赖帧率）
- 易于同步（只同步 AutoFlush 触发点）
- 支持断线重连（重放触发序列）

✅ **7. 支持动画演出**（结合分层时间控制）
- Animator 始终运行（待机动画）
- TimelineAbility 临时启用 GAS（技能动画）
- 大招演出完全可控

---

#### 缺点分析

⚠️ **1. 需要在 7 个地方调用 AutoFlush**
- **问题**：虽然自动收集 ASC，但仍需调用 AutoFlush
- **应对**：
  - 封装到 TurnBasedBattleManager 的生命周期方法中
  - 使用事件系统进一步自动化（可选）
  - 7 个触发点非常清晰，不易遗漏

⚠️ **2. Cue 需要集成 MarkDirty 逻辑**
- **问题**：需要修改所有 GameplayCue 添加 MarkDirty 调用
- **应对**：
  - 创建基类 `TurnBasedCueBase` 自动处理
  - 使用代码生成工具批量添加
  - 只需改一次，后续无需关心

⚠️ **3. 初次学习成本**
- **问题**：需要理解 MarkDirty + AutoFlush 的概念
- **应对**：
  - 提供完整文档和示例（本文档）
  - 设计直观的 API（MarkDirty, AutoFlush）
  - 一旦理解，使用非常简单

---

#### 扩展性分析

**1. 支持混合模式（战斗内/外切换）**

```csharp
public class GameModeManager
{
    public void EnterBattle()
    {
        // 进入战斗，切换到回合制
        GameplayAbilitySystem.GAS.Pause();
        tickManager.enabled = true;
    }
    
    public void ExitBattle()
    {
        // 退出战斗，恢复实时模式
        GameplayAbilitySystem.GAS.Unpause();
        tickManager.enabled = false;
    }
}
```

**2. 支持性能监控**

```csharp
void OnBattleEnd()
{
    // 打印统计信息
    tickManager.PrintStats();
    
    // 输出示例：
    // === SmartTickManager 统计 ===
    // BattleStart: 8 次 Tick
    // TurnStart: 45 次 Tick
    // AfterAbility: 123 次 Tick  ← 最多！
    // TurnEnd: 45 次 Tick
    // RoundEnd: 16 次 Tick
    // BattleEnd: 8 次 Tick
}
```

**3. 支持暂停/恢复**

```csharp
public void PauseBattle()
{
    // 已经是 Paused 状态，只需停止触发 AutoFlush
    isPaused = true;
}

public void ResumeBattle()
{
    isPaused = false;
    // 继续触发 AutoFlush
}
```

---

#### 适用场景

✅ **最适合（推荐）**：
- 回合制游戏（纯回合或带动画）
- 需要网络同步的回合制
- 性能敏感的移动平台
- 需要战斗回放的游戏
- **任何追求简洁和自动化的项目** ⭐

⚠️ **不太适合**：
- 实时 MOBA/ARPG（应保持原 GAS 自动 Tick）
- 极端复杂的混合战斗系统（需要方案 C 混合模式）

---

### 动画集成方案（配合方案 A）

#### 核心问题

**需求**：
- ✅ 角色有**待机动画**（Idle、呼吸、眨眼等）
- ✅ 技能有**释放动画**（挥剑、施法、射击等）
- ✅ 大招有**特殊演出**（镜头切换、特写、慢动作等）
- ✅ 回合制逻辑（等待玩家输入时 GAS 暂停）

**矛盾点**：
- ❌ 方案 A（GAS 暂停）如何播放待机动画？
- ❌ 技能动画期间如何处理音效、特效、Timeline？

**解决方案**：分层时间控制

---

#### 分层时间控制设计

**将系统分为三层，各自独立控制**：

1. **GAS 逻辑层**：使用 SmartTickManager（事件触发），默认暂停
2. **角色动画层**：使用 Unity Animator，**始终运行**（不受 GAS 影响）
3. **演出控制层**：使用 TimelineAbility，临时启用 GAS

```
┌─────────────────────────────────────┐
│  演出控制层（TimelineAbility）       │ ← 需要时临时启用 GAS
├─────────────────────────────────────┤
│  角色动画层（Animator）              │ ← 始终运行（不受 GAS 影响）
├─────────────────────────────────────┤
│  GAS 逻辑层（SmartTickManager）      │ ← 默认暂停（事件触发）
└─────────────────────────────────────┘
```

**关键原则**：
- Unity 的 Animator 有独立的更新循环，**不依赖 GAS**
- GAS.Pause() 只暂停 GAS 的 Tick，**不影响 Unity 引擎的 Update**
- 待机动画、呼吸、眨眼等可以正常播放
            tickManager.FlushDirtyASCs();
            
            // 如果有其他单位受影响（AOE），也需要 Tick
            var affectedUnits = GetAffectedUnits(abilityName);
            tickManager.TickAll(affectedUnits);
        }
        
        EndCurrentTurn();
    }
    
    void EndCurrentTurn()
    {
        var currentUnit = GetCurrentUnit();
        
        Debug.Log($"<<< {currentUnit.name} 的回合结束");
        
        // 回合结束 Tick
        tickManager.TickImmediate(currentUnit);
        
        // 检查战斗是否结束
        if (CheckBattleEnd()) return;
        
        // 进入下一回合
        StartNextTurn();
    }
    
    void OnRoundEnd()
    {
        Debug.Log("=== 轮次结束 ===");
        
        // 轮次结束时 Tick 所有单位（处理"每轮触发"的效果）
        tickManager.TickAll(AllUnits.Select(u => u.ASC));
    }
}
```

---

#### 优点分析

✅ **1. 零侵入框架核心**
- 使用官方 API（`Pause()/Unpause()`）
- 不修改 GameplayAbilitySystem 源码
- 框架升级时无冲突

✅ **2. 完全控制执行时机**
- 明确知道每个 Tick 发生的时机
- 易于调试（日志清晰）
- 可以在 Tick 前后插入逻辑

✅ **3. 性能最优**
- 只在必要时 Tick
- 避免每帧开销（60FPS → 0FPS）
- 智能管理器避免重复 Tick

✅ **4. 易于扩展**
- 可以添加新的触发点
- 支持条件 Tick（只 Tick 有 Buff 的单位）
- 支持批量优化

✅ **5. 网络友好**
- 确定性执行（不依赖帧率）
- 易于同步（只同步触发点）
- 支持断线重连（重放触发序列）

---

#### 缺点分析

⚠️ **1. 需要手动管理 Tick 调用**
- **问题**：开发者需要记住在所有关键点调用 Tick
- **应对**：
  - 封装 SmartTickManager 统一管理
  - 使用事件系统自动触发
  - 编写单元测试确保不遗漏

⚠️ **2. 可能遗漏 Tick 调用点**
- **问题**：新增功能时可能忘记 Tick
- **应对**：
  - 设计时明确 Tick 触发点
  - 使用 Dirty 标记自动收集
  - 代码审查检查

⚠️ **3. 调试时需要追踪 Tick 链**
- **问题**：复杂场景下不知道哪里触发了 Tick
- **应对**：
  ```csharp
  public void TickImmediate(ASC asc, string reason)
  {
      Debug.Log($"[Tick] {asc.name} - Reason: {reason}");
      asc.Tick();
  }
  ```

---

#### 扩展性分析

**1. 支持混合模式（战斗内/外切换）**

```csharp
public class GameModeManager
{
    public void EnterBattle()
    {
        // 进入战斗，切换到回合制
        GameplayAbilitySystem.GAS.Pause();
        tickManager.enabled = true;
    }
    
    public void ExitBattle()
    {
        // 退出战斗，恢复实时模式
        GameplayAbilitySystem.GAS.Unpause();
        tickManager.enabled = false;
    }
}
```

**2. 支持慢动作/快进**

```csharp
public class BattleSpeedController
{
    private float timeScale = 1.0f;
    
    public void SetSpeed(float scale)
    {
        timeScale = scale;
        // 调整动画播放速度
        Time.timeScale = scale;
        
        // Tick 频率不变（仍然是事件驱动）
    }
}
```

**3. 支持暂停/恢复**

```csharp
public void PauseBattle()
{
    // 已经是 Paused 状态，只需停止触发 Tick
    isPaused = true;
}

public void ResumeBattle()
{
    isPaused = false;
    // 继续触发 Tick
}
```

---

#### 适用场景

✅ **最适合**：
- 纯回合制游戏（无动画演出）
✅ **最适合（推荐）**：
- 回合制游戏（纯回合或带动画）
- 需要网络同步的回合制
- 性能敏感的移动平台
- 需要战斗回放的游戏
- **任何追求简洁和自动化的项目** ⭐

⚠️ **不太适合**：
- 实时 MOBA/ARPG（应保持原 GAS 自动 Tick）
- 极端复杂的混合战斗系统（需要自定义方案）

---

### 其他方案说明

#### 方案 B：虚拟时间控制（不推荐）

**核心思路**：保留框架的 Tick 循环，通过虚拟时间控制器调整时间流速。

**为什么不推荐**：
- ❌ 需要修改框架核心代码（GASTimer）
- ❌ 时间控制复杂，难以调试
- ❌ 网络同步困难
- ❌ 无法获得方案 A 的自动化优势

**结论**：方案 A（SmartTickManager + 分层动画）已完全替代此方案。

---

#### 方案 C：混合模式（不推荐）

**核心思路**：平时用手动 Tick，播放动画时用虚拟时间。

**为什么不推荐**：
- ❌ 复杂度高，需要管理两种模式
- ❌ 仍需修改框架（如果要用虚拟时间）
- ❌ 切换时机需要精心设计

**结论**：方案 A + 分层动画控制更简洁高效。

---

#### 方案 D：完全重写时间系统（强烈不推荐）

**核心思路**：Fork 框架，重写 GASTimer 和 GameplayEffectContainer。

**为什么强烈不推荐**：
- ❌ 工作量巨大（可能需要 2-4 周）
- ❌ 失去官方支持，无法升级
- ❌ 维护成本高
- ❌ 可能破坏其他功能

**结论**：除非有非常特殊的需求，否则绝不推荐。

---

## 带动画演出的回合制游戏完整方案 🎬

### 核心需求

- ✅ 角色有**待机动画**（Idle、呼吸、眨眼等）
- ✅ 技能有**释放动画**（挥剑、施法、射击等）
- ✅ 大招有**特殊演出**（镜头切换、特写、慢动作等）
- ✅ 回合制逻辑（等待玩家输入时 GAS 暂停）

### 解决方案：分层时间控制 ⭐⭐⭐

**核心思路**：将系统分为三层，各自独立控制

1. **GAS 逻辑层**：使用 SmartTickManager（事件触发），默认暂停
2. **角色动画层**：使用 Unity Animator，**始终运行**（不受 GAS 影响）
3. **演出控制层**：使用 TimelineAbility，临时启用 GAS

```
┌─────────────────────────────────────┐
│  演出控制层（TimelineAbility）       │ ← 需要时临时启用 GAS
├─────────────────────────────────────┤
│  角色动画层（Animator）              │ ← 始终运行（不受 GAS 影响）
├─────────────────────────────────────┤
│  GAS 逻辑层（SmartTickManager）      │ ← 默认暂停（事件触发）
└─────────────────────────────────────┘
```

**关键原则**：
- Unity 的 Animator 有独立的更新循环，**不依赖 GAS**
- GAS.Pause() 只暂停 GAS 的 Tick，**不影响 Unity 引擎的 Update**
- 待机动画、呼吸、眨眼等可以正常播放

---

### 详细实现

**关键原则**：角色动画不依赖 GAS 系统

```csharp
public class BattleCharacterController : MonoBehaviour
{
    private Animator animator;
    private AbilitySystemComponent asc;
    
    void Awake()
    {
        animator = GetComponent<Animator>();
        asc = GetComponent<AbilitySystemComponent>();
    }
    
    void Update()
    {
        // Animator 始终运行，不受 GAS.Pause() 影响
        // Unity 的 Animator 有自己的更新循环
        UpdateIdleAnimation();
    }
    
    void UpdateIdleAnimation()
    {
        // 根据战斗状态切换待机动画
        if (IsMyTurn())
        {
            animator.SetBool("IsActive", true); // 轮到我，播放激活待机
        }
        else if (IsWaitingForInput())
        {
            animator.SetBool("IsActive", false); // 等待输入，播放普通待机
        }
    }
}
```

**为什么这样设计**：
- Unity 的 Animator 组件有独立的更新循环，不依赖 GAS
- GAS.Pause() 只暂停 GAS 的 Tick，不影响 Unity 引擎的 Update
- 待机动画、呼吸、眨眼等可以正常播放

---

**2. 技能动画管理器**

```csharp
public class SkillAnimationController : MonoBehaviour
{
    private Animator animator;
    private AbilitySystemComponent asc;
    
    /// <summary>
    /// 播放技能动画（不依赖 GAS Tick）
    /// </summary>
    public void PlaySkillAnimation(string skillName, Action onComplete)
    {
        // 1. 触发动画
        animator.SetTrigger(skillName);
        
        // 2. 等待动画完成（通过 AnimationEvent 或计时器）
        StartCoroutine(WaitForAnimationEnd(skillName, onComplete));
    }
    
    IEnumerator WaitForAnimationEnd(string skillName, Action onComplete)
    {
        // 方法 1：通过动画长度计时
        var clipLength = GetAnimationClipLength(skillName);
        yield return new WaitForSeconds(clipLength);
        
        // 方法 2：通过 AnimationEvent 回调（更精确）
        // 在动画结尾帧添加 Event，调用 OnAnimationComplete()
        
        onComplete?.Invoke();
    }
    
    /// <summary>
    /// 获取动画片段长度
    /// </summary>
    float GetAnimationClipLength(string skillName)
    {
        var clips = animator.runtimeAnimatorController.animationClips;
        var clip = System.Array.Find(clips, c => c.name == skillName);
        return clip ? clip.length : 0f;
    }
    
    /// <summary>
    /// AnimationEvent 回调（在动画编辑器中配置）
    /// </summary>
    public void OnAnimationComplete()
    {
        Debug.Log("动画播放完成");
        // 触发回调
    }
}
```

**为什么这样设计**：
- 技能动画完全由 Unity Animator 驱动，不依赖 GAS Tick
- 使用 AnimationEvent 或计时器控制回调时机
- 灵活支持各种动画长度

---

**3. 大招演出管理器**

```csharp
public class UltimateSkillDirector : MonoBehaviour
{
    private CinemachineVirtualCamera mainCamera;
    private CinemachineVirtualCamera closeupCamera;
    
    /// <summary>
    /// 播放大招演出（复杂流程）
    /// </summary>
    public void PlayUltimateSequence(BattleUnit caster, BattleUnit target, Action onComplete)
    {
        StartCoroutine(UltimateSequenceCoroutine(caster, target, onComplete));
    }
    
    IEnumerator UltimateSequenceCoroutine(BattleUnit caster, BattleUnit target, Action onComplete)
    {
        // === 第一阶段：镜头切换 ===
        Debug.Log("[演出] 切换到特写镜头");
        closeupCamera.Priority = 20; // 提高优先级切换镜头
        yield return new WaitForSeconds(0.5f);
        
        // === 第二阶段：蓄力动画 ===
        Debug.Log("[演出] 播放蓄力动画");
        caster.AnimationController.PlaySkillAnimation("Ultimate_Charge", null);
        yield return new WaitForSeconds(1.0f);
        
        // === 第三阶段：释放技能逻辑（临时启用 GAS Tick）===
        Debug.Log("[演出] 释放技能效果");
        
        // 临时启用 GAS Tick（让 GE 生效）
        GameplayAbilitySystem.GAS.Unpause();
        
        // 执行技能逻辑
        caster.ASC.TryActivateAbility("UltimateSkill", target.ASC);
        
        // 手动 Tick 确保效果立即生效
        caster.ASC.Tick();
        target.ASC.Tick();
        
        // 立即重新暂停
        GameplayAbilitySystem.GAS.Pause();
        
        // === 第四阶段：特效播放 ===
        Debug.Log("[演出] 播放特效");
        PlayVFX("UltimateExplosion", target.transform.position);
        yield return new WaitForSeconds(1.5f);
        
        // === 第五阶段：慢动作效果 ===
        Debug.Log("[演出] 慢动作");
        Time.timeScale = 0.3f;
        yield return new WaitForSecondsRealtime(0.5f);
        Time.timeScale = 1.0f;
        
        // === 第六阶段：镜头恢复 ===
        Debug.Log("[演出] 镜头恢复");
        closeupCamera.Priority = 5;
        yield return new WaitForSeconds(0.5f);
        
        // === 演出结束 ===
        Debug.Log("[演出] 结束");
        onComplete?.Invoke();
    }
    
    void PlayVFX(string vfxName, Vector3 position)
    {
        // 播放粒子特效
        var vfx = Instantiate(vfxPrefab, position, Quaternion.identity);
        Destroy(vfx, 2.0f);
    }
}
```

**为什么这样设计**：
- **分阶段控制**：每个阶段职责清晰（镜头、动画、逻辑、特效）
- **临时启用 GAS**：只在需要应用技能效果时短暂启用，立即暂停
- **Coroutine 编排**：灵活控制时序，易于调整演出节奏
- **Time.timeScale**：支持慢动作效果（不影响 GAS 逻辑）

---

**4. 战斗流程集成**

```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    private UltimateSkillDirector ultimateDirector;
    private SkillAnimationController animationController;
    
    /// <summary>
    /// 执行技能（区分普通技能和大招）
    /// </summary>
    public void ExecuteAbility(BattleUnit actor, string abilityName, BattleUnit target)
    {
        if (IsUltimateSkill(abilityName))
        {
            // 大招：播放完整演出
            ExecuteUltimateSkill(actor, abilityName, target);
        }
        else
        {
            // 普通技能：简单动画
            ExecuteNormalSkill(actor, abilityName, target);
        }
    }
    
    /// <summary>
    /// 执行普通技能
    /// </summary>
    void ExecuteNormalSkill(BattleUnit actor, string abilityName, BattleUnit target)
    {
        Debug.Log($"{actor.name} 使用 {abilityName}");
        
        // 1. 播放技能动画（不依赖 GAS）
        actor.AnimationController.PlaySkillAnimation(abilityName, () =>
        {
            // 动画播放完成后的回调
            Debug.Log($"{abilityName} 动画播放完成");
            
            // 继续战斗流程
            EndCurrentTurn();
        });
        
        // 2. 在动画播放到一半时应用技能效果
        StartCoroutine(ApplySkillEffectAtMidpoint(actor, abilityName, target, 0.3f));
    }
    
    IEnumerator ApplySkillEffectAtMidpoint(BattleUnit actor, string abilityName, 
                                            BattleUnit target, float delay)
    {
        // 等待动画播放到一半（例如剑挥到目标时）
        yield return new WaitForSeconds(delay);
        
        Debug.Log($"[{abilityName}] 应用技能效果");
        
        // 临时启用 GAS Tick（可选，如果需要立即看到效果）
        // GameplayAbilitySystem.GAS.Unpause();
        
        // 执行技能逻辑
        actor.ASC.TryActivateAbility(abilityName, target.ASC);
        
        // 手动 Tick 确保效果生效
        TickManager.MarkDirty(actor.ASC, "SkillCaster");
        TickManager.MarkDirty(target.ASC, "SkillTarget");
        TickManager.FlushDirtyASCs();
        
        // 重新暂停（如果之前启用了）
        // GameplayAbilitySystem.GAS.Pause();
        
        // 播放受击特效
        target.AnimationController.PlayHitReaction();
    }
    
    /// <summary>
    /// 执行大招
    /// </summary>
    void ExecuteUltimateSkill(BattleUnit actor, string abilityName, BattleUnit target)
    {
        Debug.Log($"{actor.name} 使用大招 {abilityName}");
        
        // 播放完整演出
        ultimateDirector.PlayUltimateSequence(actor, target, () =>
        {
            Debug.Log("大招演出结束");
            EndCurrentTurn();
        });
    }
    
    bool IsUltimateSkill(string abilityName)
    {
        // 通过技能名称或 Tag 判断
        return abilityName.Contains("Ultimate") || abilityName.Contains("大招");
    }
}
```

---

#### 完整工作流程

```
[玩家选择技能]
  ↓
判断技能类型？
  ↓
┌─────────────────────────────────────────┐
│ 普通技能                                │
├─────────────────────────────────────────┤
│ 1. 播放角色技能动画（Animator）         │
│ 2. 在动画关键帧应用技能效果（手动 Tick）│
│ 3. 播放受击动画和特效                   │
│ 4. 等待动画结束                         │
│ 5. 回调结束回合                         │
└─────────────────────────────────────────┘
  ↓
[回合结束]

┌─────────────────────────────────────────┐
│ 大招                                    │
├─────────────────────────────────────────┤
│ 1. 切换镜头（Cinemachine）              │
│ 2. 播放蓄力动画                         │
│ 3. 临时启用 GAS → 应用效果 → 重新暂停  │
│ 4. 播放特效、慢动作                     │
│ 5. 镜头恢复                             │
│ 6. 回调结束回合                         │
└─────────────────────────────────────────┘
  ↓
[回合结束]
```

---

#### 技术要点

**1. 动画与逻辑分离**

```csharp
// ❌ 错误：在 GAS Tick 中播放动画
public override void ActivateAbility(params object[] args)
{
    animator.SetTrigger("Attack"); // GAS 暂停时不会执行！
    // ...
}

// ✅ 正确：在外部控制动画，GAS 只负责逻辑
public override void ActivateAbility(params object[] args)
{
    // 只处理数值逻辑
    Owner.ApplyGameplayEffectTo(damageGE, target);
}

// 动画在 BattleManager 中播放
actor.AnimationController.PlaySkillAnimation("Attack", () => {
    actor.ASC.TryActivateAbility("Attack", target.ASC);
});
```

---

**2. 临时启用 GAS 的时机**

```csharp
// 原则：只在需要立即看到效果时临时启用

// 场景 1：普通技能 - 不需要临时启用
ExecuteNormalSkill()
{
    // 播放动画...
    actor.ASC.TryActivateAbility(abilityName, target); // 技能逻辑
    TickManager.FlushDirtyASCs(); // 手动 Tick 即可
}

// 场景 2：大招演出 - 需要临时启用（因为可能有持续施法 GE）
ExecuteUltimateSkill()
{
    GAS.Unpause(); // 启用
    actor.ASC.TryActivateAbility("Ultimate", target);
    yield return new WaitForSeconds(2.0f); // GE 在这期间正常运行
    GAS.Pause(); // 暂停
}

// 场景 3：持续引导技能 - 需要临时启用
ExecuteChannelingSkill()
{
    GAS.Unpause();
    actor.ASC.TryActivateAbility("Channeling", target);
    
    // 引导 3 秒，期间 GE 每 0.5 秒触发一次
    yield return new WaitForSeconds(3.0f);
    
    GAS.Pause();
}
```

---

**3. 动画事件系统**

**方法 1：AnimationEvent（推荐）**

在 Unity 动画编辑器中，在关键帧添加 Event：

```
Timeline: [||||||||||||||||||||||||]
          0s  0.3s(Event)      1.0s
             ↓
         OnAttackHit() // 调用脚本方法
```

```csharp
public class SkillAnimationController : MonoBehaviour
{
    private System.Action onHitCallback;
    
    public void PlaySkillAnimation(string skillName, System.Action onHit)
    {
        onHitCallback = onHit;
        animator.SetTrigger(skillName);
    }
    
    /// <summary>
    /// AnimationEvent 回调（在动画编辑器中配置）
    /// </summary>
    public void OnAttackHit()
    {
        Debug.Log("攻击命中时刻！");
        onHitCallback?.Invoke();
    }
}
```

**方法 2：Timeline + Signal（适合复杂演出）**

使用 Unity Timeline 系统，在关键帧发送 Signal：

```csharp
public class TimelineSignalReceiver : MonoBehaviour, INotificationReceiver
{
    public void OnNotify(Playable origin, INotification notification, object context)
    {
        if (notification is AttackHitSignal)
        {
            Debug.Log("Timeline 发出攻击命中信号");
            // 应用技能效果
        }
    }
}
```

---

#### 优点分析

✅ **1. 动画表现完整**
- 待机动画正常播放（不受 GAS 影响）
- 技能动画流畅自然
- 大招演出可以任意复杂

✅ **2. 性能最优**
- 大部分时间 GAS 暂停，无 CPU 消耗
- 只在必要时短暂启用

✅ **3. 灵活控制**
- 普通技能用简单流程
- 大招用复杂演出
- 易于扩展新的演出类型

✅ **4. 易于调试**
- 动画和逻辑分离，各自独立调试
- 演出流程用 Coroutine 编排，易于修改节奏

✅ **5. 网络友好**
- 只同步技能逻辑，不同步动画
- 各客户端独立播放动画，不影响确定性

---

#### 缺点分析

⚠️ **1. 需要额外的动画管理代码**
- **问题**：需要编写 SkillAnimationController、UltimateSkillDirector
- **应对**：封装成可复用组件，一次编写多处使用

⚠️ **2. 动画和逻辑需要对齐**
- **问题**：需要确保动画播放到一半时才应用效果
- **应对**：使用 AnimationEvent 精确控制时机

⚠️ **3. 大招演出需要精心设计**
- **问题**：Coroutine 复杂，容易出错
- **应对**：使用 Unity Timeline 可视化编排

---

#### 适用场景

✅ **最适合**：
- 带动画表现的回合制游戏（如：《火焰纹章》《XCOM》）
- 需要大招演出的游戏（如：《宝可梦》）
- 需要待机动画的游戏（角色有生命感）

✅ **推荐指数**：⭐⭐⭐⭐⭐

---

#### 实施建议

**第一阶段（基础）**：
1. 实现方案 A（全局暂停 + 手动 Tick）
2. 验证待机动画正常播放
3. 实现简单的技能动画播放

**第二阶段（增强）**：
4. 实现 SkillAnimationController
5. 使用 AnimationEvent 控制效果应用时机
6. 测试动画和逻辑的同步

**第三阶段（演出）**：
7. 实现 UltimateSkillDirector
8. 设计大招演出流程
9. 集成镜头、特效、慢动作

**第四阶段（优化）**：
10. 使用 Unity Timeline 替代 Coroutine（可选）
11. 性能优化和测试
12. 添加更多演出效果

---

#### 代码示例总结

**核心文件**：
```
BattleCharacterController.cs      # 角色控制器（待机动画）
SkillAnimationController.cs       # 技能动画管理器
UltimateSkillDirector.cs          # 大招演出管理器
TurnBasedBattleManager.cs         # 战斗流程（集成动画）
```

**核心原则**：
1. 动画由 Unity Animator 驱动（不依赖 GAS）
2. 逻辑由 GAS 管理（默认暂停，手动 Tick）
3. 演出由 Coroutine/Timeline 编排（灵活控制）
4. 临时启用 GAS 只在必要时使用

---

### 与其他方案对比

| 方案 | 待机动画 | 技能动画 | 大招演出 | 性能 | 网络同步 | 推荐度 |
|------|---------|---------|---------|------|---------|--------|
| 方案 A（纯手动 Tick） | ❌ | ❌ | ❌ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| 方案 B（虚拟时间） | ✅ | ✅ | ⚠️ | ⭐ | ⭐ | ⭐ |
| 方案 C（混合模式） | ⚠️ | ✅ | ⚠️ | ⭐⭐ | ⭐⭐ | ⭐⭐ |
| **分层时间控制** | ✅ | ✅ | ✅ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

**结论**：对于带动画演出的回合制游戏，**分层时间控制是最佳方案**。

---

### TimelineAbility 集成方案 🎬

#### 问题描述

**实际需求**：
- ✅ 技能释放包含：动画、音效、特效、镜头、UI 提示等
- ✅ 需要在 Unity Timeline 中可视化编排这些元素
- ✅ 框架已有 TimelineAbility 系统，想直接使用
- ❌ 但 TimelineAbility 依赖 GAS Tick 驱动

**核心矛盾**：
- TimelineAbility 需要 GAS.Tick() 驱动时间轴
- 手动 Tick 模式下 GAS 默认暂停
- 需要在技能释放期间临时启用 GAS

---

#### 解决方案：技能释放时临时启用 GAS

**核心思路**：
1. **默认状态**：GAS 暂停，等待玩家输入
2. **技能释放时**：临时启用 GAS，让 TimelineAbility 正常运行
3. **技能结束后**：重新暂停 GAS，继续回合流程

```csharp
public class TimelineAbilityIntegration : MonoBehaviour
{
    /// <summary>
    /// 执行 TimelineAbility（自动管理 GAS 暂停/启用）
    /// </summary>
    public void ExecuteTimelineAbility(BattleUnit actor, TimelineAbilitySpec abilitySpec, 
                                       BattleUnit target, System.Action onComplete)
    {
        StartCoroutine(ExecuteTimelineAbilityCoroutine(actor, abilitySpec, target, onComplete));
    }
    
    IEnumerator ExecuteTimelineAbilityCoroutine(BattleUnit actor, TimelineAbilitySpec abilitySpec, 
                                                 BattleUnit target, System.Action onComplete)
    {
        Debug.Log($"[Timeline] 开始播放技能 Timeline：{abilitySpec.Ability.Name}");
        
        // ========== 第一步：临时启用 GAS ==========
        var wasGASPaused = !GameplayAbilitySystem.GAS.IsRunning;
        if (wasGASPaused)
        {
            GameplayAbilitySystem.GAS.Unpause();
            Debug.Log("[Timeline] 临时启用 GAS Tick");
        }
        
        // ========== 第二步：激活 TimelineAbility ==========
        abilitySpec.TryActivateAbility(target.ASC);
        
        // ========== 第三步：等待 Timeline 播放完成 ==========
        // 方法 1：通过 TimelineAbility 的 Duration 属性
        var duration = abilitySpec.GetTimelineDuration();
        yield return new WaitForSeconds(duration);
        
        // 方法 2：监听 TimelineAbility 的完成事件（更精确）
        // var completed = false;
        // abilitySpec.OnAbilityEnded += () => completed = true;
        // yield return new WaitUntil(() => completed);
        
        Debug.Log($"[Timeline] Timeline 播放完成");
        
        // ========== 第四步：重新暂停 GAS ==========
        if (wasGASPaused)
        {
            GameplayAbilitySystem.GAS.Pause();
            Debug.Log("[Timeline] 重新暂停 GAS");
        }
        
        // ========== 第五步：回调完成 ==========
        onComplete?.Invoke();
    }
}
```

---

#### TimelineAbility 内部结构示例

**在 Unity Timeline 中编排技能表现**：

```
Timeline: 技能释放完整流程 (Duration: 3.0s)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
0.0s  角色动画轨道：播放"施法动画"
      ┃
0.3s  音效轨道：播放"吟唱音效"
      ┃
0.5s  特效轨道：手部出现魔法阵
      ┃
1.0s  ◆ GE 应用轨道：施加伤害效果（InstantCue Track）
      ┃  ↓ 此时 GAS 正在运行，GE 正常生效
      ┃
1.2s  特效轨道：释放火球
      ┃
1.5s  镜头轨道：镜头震动
      ┃
2.0s  ◆ 目标动画轨道：播放受击动画（Cue Track）
      ┃
2.5s  UI 轨道：显示伤害数字
      ┃
3.0s  [Timeline 结束]
```

**对应的 TimelineAbility 配置**：

```csharp
public class FireballTimelineAbility : TimelineAbility
{
    // 在 Inspector 中配置：
    // - Animation Track: 施法动画
    // - Audio Track: 音效
    // - VFX Track: 特效
    // - InstantCue Track (1.0s): 应用伤害 GE
    // - DurationalCue Track: 持续特效
}

public class FireballTimelineAbilitySpec : TimelineAbilitySpec<FireballTimelineAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // TimelineAbility 会自动播放 Timeline
        // 在 1.0s 时触发 InstantCue，应用 GE
        base.ActivateAbility(target);
    }
    
    /// <summary>
    /// 获取 Timeline 总时长
    /// </summary>
    public float GetTimelineDuration()
    {
        // 从 PlayableDirector 获取时长
        return (float)Ability.DataReference.TimelineAsset.duration;
    }
}
```

---

#### 战斗流程中的使用

```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    private TimelineAbilityIntegration timelineIntegration;
    
    /// <summary>
    /// 执行技能（自动判断是否为 TimelineAbility）
    /// </summary>
    public void ExecuteAbility(BattleUnit actor, string abilityName, BattleUnit target)
    {
        var abilitySpec = actor.ASC.AbilityContainer.AbilitySpecs()[abilityName];
        
        // 判断是否为 TimelineAbility
        if (abilitySpec is TimelineAbilitySpec timelineSpec)
        {
            Debug.Log($"[战斗] {actor.name} 使用 TimelineAbility: {abilityName}");
            
            // 使用 TimelineAbility 执行流程
            timelineIntegration.ExecuteTimelineAbility(actor, timelineSpec, target, () =>
            {
                Debug.Log($"[战斗] TimelineAbility 执行完成");
                EndCurrentTurn();
            });
        }
        else
        {
            Debug.Log($"[战斗] {actor.name} 使用普通 Ability: {abilityName}");
            
            // 普通技能：立即执行，手动 Tick
            actor.ASC.TryActivateAbility(abilityName, target.ASC);
            TickManager.MarkDirty(actor.ASC, "SkillCaster");
            TickManager.MarkDirty(target.ASC, "SkillTarget");
            TickManager.FlushDirtyASCs();
            
            EndCurrentTurn();
        }
    }
}
```

---

#### TimelineAbility vs 普通 Ability 对比

| 维度 | TimelineAbility | 普通 Ability |
|------|----------------|-------------|
| **表现层** | Timeline 可视化编排 | 代码控制 |
| **音效特效** | Timeline 轨道管理 | 手动播放 |
| **GAS Tick** | 需要临时启用 | 手动 Tick 即可 |
| **性能** | 略低（Timeline 开销） | 最优 |
| **网络同步** | 略复杂（需同步时长） | 简单 |
| **适用场景** | 复杂演出技能、大招 | 简单技能、被动 |

---

#### 优化建议

**1. 预加载 Timeline 资源**

```csharp
public class TimelinePreloader : MonoBehaviour
{
    void Start()
    {
        // 预加载所有技能 Timeline，避免运行时卡顿
        foreach (var ability in allTimelineAbilities)
        {
            ability.PreloadTimeline();
        }
    }
}
```

**2. Timeline 时长缓存**

```csharp
public class TimelineAbilitySpec : AbilitySpec
{
    private float cachedDuration = -1f;
    
    public float GetTimelineDuration()
    {
        if (cachedDuration < 0)
        {
            cachedDuration = (float)Ability.DataReference.TimelineAsset.duration;
        }
        return cachedDuration;
    }
}
```

**3. GAS 启用/暂停优化**

```csharp
// 如果连续多个单位都使用 TimelineAbility
public void ExecuteMultipleTimelineAbilities(List<TimelineAbilityExecution> executions)
{
    StartCoroutine(BatchExecuteTimelines(executions));
}

IEnumerator BatchExecuteTimelines(List<TimelineAbilityExecution> executions)
{
    // 只启用一次 GAS
    GameplayAbilitySystem.GAS.Unpause();
    
    // 顺序执行所有 Timeline
    foreach (var exec in executions)
    {
        exec.abilitySpec.TryActivateAbility(exec.target);
        yield return new WaitForSeconds(exec.abilitySpec.GetTimelineDuration());
    }
    
    // 全部完成后暂停
    GameplayAbilitySystem.GAS.Pause();
}
```

---

### 被动技能触发机制 ⚡

#### 问题描述

**典型被动技能场景**：
- 反击：被攻击时有概率反击攻击者
- 保护：友军被攻击时替其承受伤害
- 连击：攻击后有概率再次行动
- 吸血：造成伤害时恢复生命
- 荆棘：受到伤害时反弹部分伤害

**核心问题**：
- 被动技能不在自己回合触发
- 需要在其他单位行动时插入额外逻辑
- 手动 Tick 模式下如何确保被动触发？

---

#### 解决方案：事件驱动 + 即时 Tick

**设计原则**：
1. **事件驱动触发**：通过 GameplayCue 系统触发被动检查
2. **即时 Tick**：被动技能触发后立即 Tick，不等待回合结束
3. **不影响流程**：被动执行在当前技能的 Cue 回调中，不中断战斗流程

---

#### 实现架构

**1. 被动技能注册系统**（复用之前的设计）

```csharp
public enum PassiveTriggerType
{
    OnBeingAttacked,    // 被攻击时（伤害应用前）
    OnDamageTaken,      // 受到伤害时（伤害应用后）
    OnDealDamage,       // 造成伤害时
    OnAllyAttacked,     // 友军被攻击时
    OnDodge,            // 闪避时
    OnCriticalHit,      // 暴击时
    OnKill,             // 击杀敌人时
    OnHealthBelow50,    // 血量低于 50% 时
}

public class PassiveAbilitySystem
{
    public class PassiveAbilityData
    {
        public AbilitySpec Ability;
        public PassiveTriggerType TriggerType;
        public float TriggerChance;
        public int CooldownTurns;
        public int RemainingCooldown;
        public System.Func<PassiveTriggerContext, bool> Condition;
    }
    
    public class PassiveTriggerContext
    {
        public AbilitySystemComponent Source;      // 触发源（攻击者）
        public AbilitySystemComponent Target;      // 触发目标（受击者）
        public AbilitySystemComponent PassiveOwner; // 被动技能持有者
        public float Damage;
        public bool IsCritical;
        public GameplayEffect SourceEffect;
    }
    
    private Dictionary<AbilitySystemComponent, List<PassiveAbilityData>> passiveAbilities = new();
    
    /// <summary>
    /// 触发被动技能
    /// </summary>
    public void TriggerPassives(PassiveTriggerType triggerType, PassiveTriggerContext context)
    {
        // 检查被动持有者的被动技能
        if (!passiveAbilities.ContainsKey(context.PassiveOwner)) return;
        
        foreach (var passive in passiveAbilities[context.PassiveOwner])
        {
            if (ShouldTriggerPassive(passive, triggerType, context))
            {
                ExecutePassiveAbility(passive, context);
            }
        }
    }
    
    bool ShouldTriggerPassive(PassiveAbilityData passive, PassiveTriggerType triggerType, 
                              PassiveTriggerContext context)
    {
        // 1. 检查触发类型
        if (passive.TriggerType != triggerType) return false;
        
        // 2. 检查冷却
        if (passive.RemainingCooldown > 0) return false;
        
        // 3. 检查概率
        if (UnityEngine.Random.value > passive.TriggerChance) return false;
        
        // 4. 检查额外条件
        if (passive.Condition != null && !passive.Condition(context)) return false;
        
        return true;
    }
    
    void ExecutePassiveAbility(PassiveAbilityData passive, PassiveTriggerContext context)
    {
        Debug.Log($"[被动] {context.PassiveOwner.name} 触发被动技能 {passive.Ability.Ability.Name}");
        
        // 执行被动技能
        passive.Ability.TryActivateAbility(context.Source, context.Target);
        
        // ========== 关键：立即 Tick，确保效果生效 ==========
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        tickManager.MarkDirty(context.PassiveOwner, "PassiveOwner");
        tickManager.MarkDirty(context.Source, "PassiveTarget");
        tickManager.FlushDirtyASCs();
        
        // 设置冷却
        passive.RemainingCooldown = passive.CooldownTurns;
    }
}
```

---

#### 被动技能示例

**示例 1：反击技能**

```csharp
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
            TriggerChance = 0.5f, // 50% 概率
            CooldownTurns = 0,    // 无冷却
            Condition = (context) =>
            {
                // 只有物理攻击才反击
                return context.SourceEffect.HasTag(GTagLib.Damage_Physical);
            }
        });
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var attacker = args[0] as AbilitySystemComponent;
        
        Debug.Log($"[反击] {Owner.name} 反击 {attacker.name}!");
        
        // 造成反击伤害（50% 攻击力）
        var damageGE = Ability.DataReference.CounterDamageEffect;
        Owner.ApplyGameplayEffectTo(damageGE, attacker);
        
        // 触发反击 Cue（播放动画、特效）
        TriggerCounterCue(attacker);
        
        EndAbility();
    }
}
```

---

**示例 2：保护技能**（替友军承受伤害）

```csharp
public class ProtectAllyAbilitySpec : AbilitySpec<ProtectAllyAbility>
{
    public override void OnGranted()
    {
        base.OnGranted();
        
        var passiveSystem = TurnBasedBattleManager.Instance.PassiveSystem;
        passiveSystem.RegisterPassive(Owner, new PassiveAbilitySystem.PassiveAbilityData
        {
            Ability = this,
            TriggerType = PassiveTriggerType.OnAllyAttacked,
            TriggerChance = 1.0f, // 100% 触发
            CooldownTurns = 1,    // 每回合限一次
            Condition = (context) =>
            {
                // 只保护相邻的友军
                var battleMgr = TurnBasedBattleManager.Instance;
                return battleMgr.IsAdjacent(Owner, context.Target);
            }
        });
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var attacker = args[0] as AbilitySystemComponent;
        var originalTarget = args[1] as AbilitySystemComponent;
        
        Debug.Log($"[保护] {Owner.name} 替 {originalTarget.name} 承受攻击!");
        
        // 将伤害转移到自己身上
        // 这需要在伤害应用前触发，修改目标
        // 实现方式：通过 GE 的 TargetRedirection 机制
        
        // 触发保护 Cue（播放冲刺动画、盾牌特效）
        TriggerProtectCue(originalTarget);
        
        EndAbility();
    }
}
```

**保护技能的特殊处理**：

```csharp
// 在 DamageCue 中检查保护被动
public class DamageCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        var source = parameters.Source;
        var target = parameters.Target;
        
        // ========== 关键：在伤害应用前检查保护被动 ==========
        var context = new PassiveAbilitySystem.PassiveTriggerContext
        {
            Source = source,
            Target = target,
            PassiveOwner = null, // 稍后填充
            Damage = parameters.MagnitudeData.Magnitude,
            SourceEffect = parameters.GameplayEffectSpec.GameplayEffect
        };
        
        // 检查所有友军的保护被动
        var passiveSystem = TurnBasedBattleManager.Instance.PassiveSystem;
        var allies = TurnBasedBattleManager.Instance.GetAllies(target);
        
        AbilitySystemComponent protector = null;
        foreach (var ally in allies)
        {
            context.PassiveOwner = ally;
            
            // 尝试触发保护被动
            if (passiveSystem.TryTriggerPassive(PassiveTriggerType.OnAllyAttacked, context))
            {
                protector = ally;
                break; // 只有一个单位可以保护
            }
        }
        
        // 如果有保护者，修改目标
        if (protector != null)
        {
            Debug.Log($"[保护] {protector.name} 替 {target.name} 承受伤害");
            target = protector; // 重定向目标
        }
        
        // 应用伤害到最终目标
        // ... 原有伤害逻辑
        
        // 触发受击被动（反击、荆棘等）
        context.PassiveOwner = target;
        passiveSystem.TriggerPassives(PassiveTriggerType.OnDamageTaken, context);
    }
}
```

---

**示例 3：连击技能**（攻击后额外回合）

```csharp
public class ComboAttackAbilitySpec : AbilitySpec<ComboAttackAbility>
{
    public override void OnGranted()
    {
        base.OnGranted();
        
        var passiveSystem = TurnBasedBattleManager.Instance.PassiveSystem;
        passiveSystem.RegisterPassive(Owner, new PassiveAbilitySystem.PassiveAbilityData
        {
            Ability = this,
            TriggerType = PassiveTriggerType.OnDealDamage,
            TriggerChance = 0.3f, // 30% 概率
            CooldownTurns = 2,
            Condition = null
        });
    }
    
    public override void ActivateAbility(params object[] args)
    {
        Debug.Log($"[连击] {Owner.name} 触发连击，获得额外回合!");
        
        // 立即获得额外回合
        var turnOrderSystem = TurnBasedBattleManager.Instance.TurnOrderSystem;
        turnOrderSystem.GrantExtraTurn(Owner, priority: 999);
        
        EndAbility();
    }
}
```

---

#### 触发时机与流程

**完整的伤害应用流程**（含被动触发）：

```
[攻击者使用技能]
  ↓
应用伤害 GE
  ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DamageCue.Trigger() 被调用
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ↓
1. 检查"友军保护"被动（OnAllyAttacked）
   ├─ 有保护者 → 重定向目标
   └─ 无保护者 → 继续
  ↓
2. 应用最终伤害到目标
  ↓
3. 触发攻击者的"造成伤害"被动（OnDealDamage）
   ├─ 吸血被动 → 恢复生命 → 立即 Tick
   ├─ 连击被动 → 获得额外回合
   └─ ...
  ↓
4. 触发目标的"受到伤害"被动（OnDamageTaken）
   ├─ 反击被动 → 反击攻击者 → 立即 Tick
   ├─ 荆棘被动 → 反弹伤害 → 立即 Tick
   └─ ...
  ↓
5. 检查目标是否死亡
   ├─ 已死亡 → 触发攻击者的"击杀"被动（OnKill）
   └─ 未死亡 → 继续
  ↓
6. 播放受击动画、特效
  ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DamageCue.Trigger() 结束
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ↓
[技能执行完成，回合继续]
```

---

#### 关键代码：在 Cue 中触发被动

```csharp
public class DamageCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        var source = parameters.Source;
        var target = parameters.Target;
        var damage = parameters.MagnitudeData.Magnitude;
        
        var passiveSystem = TurnBasedBattleManager.Instance.PassiveSystem;
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        
        // 构建上下文
        var context = new PassiveAbilitySystem.PassiveTriggerContext
        {
            Source = source,
            Target = target,
            Damage = damage,
            IsCritical = parameters.MagnitudeData.IsCritical,
            SourceEffect = parameters.GameplayEffectSpec.GameplayEffect
        };
        
        // ========== 1. 攻击前：检查保护被动 ==========
        context.PassiveOwner = null; // 遍历所有友军
        var finalTarget = CheckProtectionPassive(passiveSystem, context, target);
        
        // ========== 2. 应用伤害 ==========
        // ... 实际伤害逻辑
        
        // ========== 3. 攻击后：触发攻击者的被动 ==========
        context.PassiveOwner = source;
        passiveSystem.TriggerPassives(PassiveTriggerType.OnDealDamage, context);
        
        // ========== 4. 受击后：触发受击者的被动 ==========
        context.PassiveOwner = finalTarget;
        passiveSystem.TriggerPassives(PassiveTriggerType.OnDamageTaken, context);
        
        // ========== 5. 立即 Tick 所有相关单位 ==========
        // 这确保被动技能的效果立即生效，不等待回合结束
        tickManager.MarkDirty(source, "Attacker");
        tickManager.MarkDirty(finalTarget, "Victim");
        tickManager.FlushDirtyASCs();
        
        // ========== 6. 检查击杀 ==========
        if (finalTarget.GetAttributeCurrentValue("AS_Combat", "Health") <= 0)
        {
            context.PassiveOwner = source;
            passiveSystem.TriggerPassives(PassiveTriggerType.OnKill, context);
        }
        
        // ========== 7. 播放表现 ==========
        PlayHitAnimation(finalTarget);
        PlayHitVFX(finalTarget);
    }
}
```

---

#### 被动技能不会影响战斗流程的原因

**关键设计**：
1. **同步执行**：被动在 Cue 回调中触发，不是异步协程
2. **即时 Tick**：被动效果立即生效，不等待回合结束
3. **不中断流程**：所有被动处理在当前技能的 Cue 内完成

**流程对比**：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
没有被动技能的流程：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. 玩家选择技能
2. 执行技能 AbilitySpec.ActivateAbility()
3. 应用 GE
4. 触发 Cue（播放特效）
5. 手动 Tick
6. 回合结束

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
有被动技能的流程：
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. 玩家选择技能
2. 执行技能 AbilitySpec.ActivateAbility()
3. 应用 GE
4. 触发 Cue（播放特效）
   ├─ 4.1 检查被动触发（保护）
   ├─ 4.2 应用伤害
   ├─ 4.3 触发攻击者被动（吸血、连击）
   ├─ 4.4 触发受击者被动（反击、荆棘）
   ├─ 4.5 立即 Tick 所有相关单位 ← 关键！
   └─ 4.6 播放动画特效
5. 手动 Tick（已经在 4.5 完成，可选）
6. 回合结束

流程总时长：相同（被动在步骤 4 内部完成）
```

**为什么不会影响流程**：
- ✅ 被动触发是**同步的**，在 Cue.Trigger() 方法内完成
- ✅ 被动效果通过**立即 Tick** 生效，不需要等待
- ✅ 战斗管理器只需等待 Cue 完成，不关心内部细节
- ✅ 被动的动画/特效可以与主技能的特效同时播放

---

#### 被动技能动画处理

**问题**：反击、保护等被动有自己的动画，如何处理？

**方案 1：简化动画（推荐）**

```csharp
public class CounterAttackAbilitySpec : AbilitySpec<CounterAttackAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var attacker = args[0] as AbilitySystemComponent;
        
        // 播放简化的反击动画（快速挥剑，不使用 Timeline）
        Owner.GetComponent<Animator>().SetTrigger("QuickCounter");
        
        // 应用反击伤害
        Owner.ApplyGameplayEffectTo(counterDamageGE, attacker);
        
        EndAbility();
    }
}
```

**方案 2：延迟战斗流程（复杂被动）**

```csharp
public class ProtectAllyAbilitySpec : AbilitySpec<ProtectAllyAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var attacker = args[0] as AbilitySystemComponent;
        var ally = args[1] as AbilitySystemComponent;
        
        // 启动保护演出（Coroutine）
        TurnBasedBattleManager.Instance.StartCoroutine(PlayProtectSequence(attacker, ally));
        
        EndAbility();
    }
    
    IEnumerator PlayProtectSequence(AbilitySystemComponent attacker, AbilitySystemComponent ally)
    {
        // 1. 冲刺到友军身前
        yield return MoveToPosition(Owner, ally.transform.position);
        
        // 2. 播放格挡动画
        Owner.GetComponent<Animator>().SetTrigger("Block");
        yield return new WaitForSeconds(0.5f);
        
        // 3. 承受伤害（已经在 Cue 中重定向了目标）
        
        // 4. 返回原位
        yield return MoveToOriginalPosition(Owner);
    }
}
```

**注意**：如果使用方案 2，需要在 Cue 中等待被动动画完成：

```csharp
public class DamageCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        // ... 前面的逻辑
        
        // 如果触发了复杂被动，等待其完成
        if (triggeredComplexPassive)
        {
            StartCoroutine(WaitForPassiveComplete(() =>
            {
                // 被动完成后继续
                PlayHitAnimation(target);
            }));
        }
        else
        {
            // 直接播放受击动画
            PlayHitAnimation(target);
        }
    }
}
```

---

#### 性能优化

**1. 被动检查优化**

```csharp
// 使用哈希表按触发类型分组
private Dictionary<PassiveTriggerType, List<PassiveAbilityData>> passivesByType = new();

public void TriggerPassives(PassiveTriggerType triggerType, PassiveTriggerContext context)
{
    // 只检查匹配类型的被动
    if (!passivesByType.ContainsKey(triggerType)) return;
    
    foreach (var passive in passivesByType[triggerType])
    {
        if (passive.Owner == context.PassiveOwner && ShouldTriggerPassive(passive, context))
        {
            ExecutePassiveAbility(passive, context);
        }
    }
}
```

**2. Tick 批处理**

```csharp
// 在一次伤害事件中，可能触发多个被动
// 使用 MarkDirty + FlushDirtyASCs 批量处理，而非每个被动都 Tick 一次

// ❌ 低效
passiveSystem.TriggerPassives(PassiveTriggerType.OnDealDamage, context);
source.Tick(); // Tick 1
passiveSystem.TriggerPassives(PassiveTriggerType.OnDamageTaken, context);
target.Tick(); // Tick 2

// ✅ 高效
passiveSystem.TriggerPassives(PassiveTriggerType.OnDealDamage, context);
passiveSystem.TriggerPassives(PassiveTriggerType.OnDamageTaken, context);
tickManager.MarkDirty(source);
tickManager.MarkDirty(target);
tickManager.FlushDirtyASCs(); // 只 Tick 一次
```

---

#### 总结

**被动技能在手动 Tick 模式下的关键点**：

1. ✅ **事件驱动**：通过 GameplayCue 系统触发，不依赖 GAS Tick
2. ✅ **即时生效**：触发后立即 MarkDirty + FlushDirtyASCs，不等待回合结束
3. ✅ **不影响流程**：所有被动处理在 Cue 回调内同步完成
4. ✅ **性能优化**：批量 Tick，避免重复
5. ✅ **动画灵活**：简单被动用 Trigger 动画，复杂被动用 Coroutine

**被动触发流程**：
```
技能释放 → 应用 GE → Cue.Trigger()
                        ↓
           ┌────────────┴────────────┐
           │ 1. 检查保护被动         │
           │ 2. 应用伤害             │
           │ 3. 触发攻击者被动       │
           │ 4. 触发受击者被动       │
           │ 5. MarkDirty + Flush    │ ← 立即 Tick
           │ 6. 播放动画特效         │
           └────────────┬────────────┘
                        ↓
                    Cue 结束
                        ↓
                   回合继续
```

---

## 推荐方案详细设计

**综合评估**：方案 A（全局暂停 + 手动触发）

**推荐理由**：
1. ✅ 零侵入框架核心
2. ✅ 性能最优
3. ✅ 易于调试
4. ✅ 网络友好
5. ✅ 确定性强

---

### 完整实现架构

#### 1. 核心组件设计

```csharp
/// <summary>
/// 智能 Tick 管理器
/// </summary>
public class SmartTickManager
{
    private HashSet<AbilitySystemComponent> dirtyASCs = new();
    private Dictionary<AbilitySystemComponent, List<string>> tickReasons = new();
    
    public bool EnableLogging { get; set; } = false;
    
    /// <summary>
    /// 标记 ASC 需要更新
    /// </summary>
    public void MarkDirty(AbilitySystemComponent asc, string reason = "Unknown")
    {
        if (asc == null) return;
        
        dirtyASCs.Add(asc);
        
        if (EnableLogging)
        {
            if (!tickReasons.ContainsKey(asc))
            {
                tickReasons[asc] = new List<string>();
            }
            tickReasons[asc].Add(reason);
        }
    }
    
    /// <summary>
    /// 执行所有待更新的 ASC
    /// </summary>
    public void FlushDirtyASCs()
    {
        if (dirtyASCs.Count == 0) return;
        
        if (EnableLogging)
        {
            Debug.Log($"[SmartTickManager] Flushing {dirtyASCs.Count} dirty ASCs");
        }
        
        foreach (var asc in dirtyASCs)
        {
            if (EnableLogging && tickReasons.ContainsKey(asc))
            {
                Debug.Log($"[Tick] {asc.name} - Reasons: {string.Join(", ", tickReasons[asc])}");
            }
            
            asc.Tick();
        }
        
        dirtyASCs.Clear();
        tickReasons.Clear();
    }
    
    /// <summary>
    /// 立即 Tick 指定 ASC
    /// </summary>
    public void TickImmediate(AbilitySystemComponent asc, string reason = "Immediate")
    {
        if (asc == null) return;
        
        if (EnableLogging)
        {
            Debug.Log($"[Tick] {asc.name} - Reason: {reason}");
        }
        
        asc.Tick();
    }
    
    /// <summary>
    /// 批量 Tick
    /// </summary>
    public void TickAll(IEnumerable<AbilitySystemComponent> ascs, string reason = "BatchTick")
    {
        if (ascs == null) return;
        
        foreach (var asc in ascs)
        {
            TickImmediate(asc, reason);
        }
    }
    
    /// <summary>
    /// 条件 Tick（只 Tick 满足条件的）
    /// </summary>
    public void TickWhere(IEnumerable<AbilitySystemComponent> ascs, 
                          Func<AbilitySystemComponent, bool> predicate, 
                          string reason = "ConditionalTick")
    {
        if (ascs == null) return;
        
        foreach (var asc in ascs.Where(predicate))
        {
            TickImmediate(asc, reason);
        }
    }
}
```

---

#### 2. 战斗流程集成

```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    public static TurnBasedBattleManager Instance { get; private set; }
    
    // 核心系统
    public SmartTickManager TickManager { get; private set; }
    public TurnOrderSystem TurnOrderSystem { get; private set; }
    public TurnBasedEffectManager EffectManager { get; private set; }
    
    // 战斗数据
    public List<BattleUnit> AllUnits;
    public int CurrentTurn { get; private set; }
    
    void Awake()
    {
        Instance = this;
        
        TickManager = new SmartTickManager();
        TurnOrderSystem = new TurnOrderSystem();
        EffectManager = new TurnBasedEffectManager();
    }
    
    void Start()
    {
        InitBattle();
        StartBattle();
    }
    
    /// <summary>
    /// 初始化战斗
    /// </summary>
    void InitBattle()
    {
        Debug.Log("=== 初始化战斗 ===");
        
        // 1. 暂停 GAS 自动 Tick
        GameplayAbilitySystem.GAS.Pause();
        
        // 2. 初始化所有单位的 ASC
        foreach (var unit in AllUnits)
        {
            unit.ASC.InitWithPreset(1, unit.Preset);
            GameplayAbilitySystem.GAS.Register(unit.ASC);
        }
        
        // 3. 初始化行动顺序
        TurnOrderSystem.InitializeTurnOrder(AllUnits.Select(u => u.ASC).ToList());
        
        // 4. 触发战斗开始 Tick（初始化战场 Buff）
        TickManager.TickAll(AllUnits.Select(u => u.ASC), "BattleStart");
    }
    
    /// <summary>
    /// 开始战斗
    /// </summary>
    void StartBattle()
    {
        Debug.Log("=== 战斗开始 ===");
        CurrentTurn = 1;
        StartNextTurn();
    }
    
    /// <summary>
    /// 开始下一个回合
    /// </summary>
    void StartNextTurn()
    {
        // 1. 获取下一个行动者
        var currentUnit = TurnOrderSystem.GetNextActor();
        
        if (currentUnit == null)
        {
            // 所有单位都死亡？
            EndBattle(false);
            return;
        }
        
        Debug.Log($">>> {currentUnit.name} 的回合开始（回合 {CurrentTurn}）");
        
        // 2. 回合开始 Tick
        TickManager.TickImmediate(currentUnit, "TurnStart");
        
        // 3. 处理回合开始效果（敌方 Debuff）
        EffectManager.ProcessTurnStart(currentUnit);
        
        // 4. 检查控制状态
        if (IsControlled(currentUnit))
        {
            Debug.Log($"{currentUnit.name} 被控制，跳过回合");
            EndCurrentTurn();
            return;
        }
        
        // 5. 等待行动选择
        if (IsPlayerUnit(currentUnit))
        {
            ShowPlayerActionMenu(currentUnit);
        }
        else
        {
            AISelectAction(currentUnit);
        }
    }
    
    /// <summary>
    /// 执行技能
    /// </summary>
    public void ExecuteAbility(BattleUnit actor, string abilityName, AbilitySystemComponent target)
    {
        Debug.Log($"{actor.ASC.name} 使用 {abilityName}");
        
        // 1. 技能执行前 Tick（标记相关单位）
        TickManager.MarkDirty(actor.ASC, "BeforeAbility");
        if (target != null)
        {
            TickManager.MarkDirty(target, "BeforeAbility_Target");
        }
        
        // 2. 执行技能
        bool success = actor.ASC.TryActivateAbility(abilityName, target);
        
        if (success)
        {
            // 3. 技能执行后 Tick（确保效果立即生效）
            TickManager.FlushDirtyASCs();
            
            // 如果是 AOE，Tick 所有受影响单位
            // TODO: 这里需要从技能获取实际目标列表
            
            // 4. 检查战斗结束
            if (CheckBattleEnd())
            {
                return;
            }
        }
        
        // 5. 结束当前回合
        EndCurrentTurn();
    }
    
    /// <summary>
    /// 结束当前回合
    /// </summary>
    void EndCurrentTurn()
    {
        var currentUnit = GetCurrentUnit();
        
        Debug.Log($"<<< {currentUnit.name} 的回合结束");
        
        // 1. 回合结束 Tick
        TickManager.TickImmediate(currentUnit, "TurnEnd");
        
        // 2. 处理回合结束效果（自身 Buff 计数）
        EffectManager.ProcessTurnEnd(currentUnit);
        
        // 3. 增加回合计数
        CurrentTurn++;
        
        // 4. 检查轮次结束（所有单位都行动过）
        if (ShouldTriggerRoundEnd())
        {
            OnRoundEnd();
        }
        
        // 5. 延迟开始下一回合（给玩家查看的时间）
        StartCoroutine(DelayedNextTurn(0.5f));
    }
    
    IEnumerator DelayedNextTurn(float delay)
    {
        yield return new WaitForSeconds(delay);
        StartNextTurn();
    }
    
    /// <summary>
    /// 轮次结束（所有单位都行动过）
    /// </summary>
    void OnRoundEnd()
    {
        Debug.Log("=== 轮次结束 ===");
        
        // Tick 所有单位（处理"每轮触发"的效果）
        TickManager.TickAll(AllUnits.Select(u => u.ASC), "RoundEnd");
    }
    
    /// <summary>
    /// 检查战斗结束
    /// </summary>
    bool CheckBattleEnd()
    {
        var team0Alive = AllUnits.Any(u => u.Team == 0 && u.IsAlive);
        var team1Alive = AllUnits.Any(u => u.Team == 1 && u.IsAlive);
        
        if (!team0Alive)
        {
            EndBattle(false);
            return true;
        }
        
        if (!team1Alive)
        {
            EndBattle(true);
            return true;
        }
        
        return false;
    }
    
    /// <summary>
    /// 结束战斗
    /// </summary>
    void EndBattle(bool victory)
    {
        Debug.Log($"=== 战斗结束 - {(victory ? "胜利" : "失败")} ===");
        
        // 1. 最后一次 Tick（清理状态）
        TickManager.TickAll(AllUnits.Select(u => u.ASC), "BattleEnd");
        
        // 2. 清理所有效果
        foreach (var unit in AllUnits)
        {
            EffectManager.ClearUnitEffects(unit.ASC);
        }
        
        // 3. 显示结算界面
        // ...
    }
    
    // 辅助方法
    bool IsControlled(AbilitySystemComponent asc)
    {
        return asc.HasAnyTags(new[] {
            GTagLib.State_Control_Stun,
            GTagLib.State_Control_Freeze,
            GTagLib.State_Control_Sleep
        });
    }
    
    bool IsPlayerUnit(AbilitySystemComponent asc)
    {
        return AllUnits.First(u => u.ASC == asc).Team == 0;
    }
    
    AbilitySystemComponent GetCurrentUnit()
    {
        // 简化：从行动顺序系统获取
        return TurnOrderSystem.CurrentUnit;
    }
    
    bool ShouldTriggerRoundEnd()
    {
        // 简化：每 N 个回合触发一次
        return CurrentTurn % AllUnits.Count == 0;
    }
}
```

---

#### 3. 配置与调试支持

```csharp
[CreateAssetMenu(fileName = "BattleTickConfig", menuName = "EX-GAS/Battle Tick Config")]
public class BattleTickConfig : ScriptableObject
{
    [Header("调试选项")]
    public bool enableTickLogging = false;
    public bool logOnlyPlayer = false;
    public bool showTickReasons = true;
    
    [Header("性能选项")]
    public bool useDirtyMarking = true;    // 使用脏标记优化
    public bool batchTickOnRoundEnd = true; // 轮次结束批量 Tick
    
    [Header("Tick 触发点")]
    public bool tickOnBattleStart = true;
    public bool tickOnTurnStart = true;
    public bool tickOnBeforeAbility = false; // 可选
    public bool tickOnAfterAbility = true;
    public bool tickOnTurnEnd = true;
    public bool tickOnRoundEnd = true;
    public bool tickOnBattleEnd = true;
}
```

---

## 实现步骤

### 阶段一：基础准备（1 天）

- [ ] **Step 1.1**: 创建 `SmartTickManager.cs`
  - [ ] 实现 `MarkDirty()`
  - [ ] 实现 `FlushDirtyASCs()`
  - [ ] 实现 `TickImmediate()`
  - [ ] 实现 `TickAll()`
  - [ ] 添加日志功能

- [ ] **Step 1.2**: 创建 `BattleTickConfig.cs`
  - [ ] 定义配置项
  - [ ] 创建默认配置资源

- [ ] **Step 1.3**: 编写单元测试
  - [ ] 测试 `MarkDirty()` 去重
  - [ ] 测试 `FlushDirtyASCs()` 批量执行
  - [ ] 测试日志输出

---

### 阶段二：战斗流程集成（2-3 天）

- [ ] **Step 2.1**: 修改 `TurnBasedBattleManager`
  - [ ] 在 `InitBattle()` 中暂停 GAS
  - [ ] 在 `StartBattle()` 中触发初始 Tick

- [ ] **Step 2.2**: 实现 Tick 触发点
  - [ ] `StartNextTurn()` 中添加 TurnStart Tick
  - [ ] `ExecuteAbility()` 前后添加 Tick
  - [ ] `EndCurrentTurn()` 中添加 TurnEnd Tick
  - [ ] `OnRoundEnd()` 中添加 RoundEnd Tick

- [ ] **Step 2.3**: 集成到现有代码
  - [ ] 替换所有直接 `asc.Tick()` 调用为 `TickManager.TickImmediate()`
  - [ ] 在需要延迟 Tick 的地方使用 `MarkDirty()`

---

### 阶段三：测试与优化（2-3 天）

- [ ] **Step 3.1**: 功能测试
  - [ ] 测试完整战斗流程
  - [ ] 测试 Buff 在各个时机正确结算
  - [ ] 测试控制状态正确跳过回合
  - [ ] 测试战斗结束正确触发

- [ ] **Step 3.2**: 性能测试
  - [ ] 对比暂停前后 CPU 占用
  - [ ] 测试大量单位战斗（10v10）
  - [ ] 优化 Tick 调用次数

- [ ] **Step 3.3**: 边界情况测试
  - [ ] 测试所有单位同时死亡
  - [ ] 测试额外回合期间的 Tick
  - [ ] 测试 AOE 技能的多目标 Tick

---

### 阶段四：文档与工具（1 天）

- [ ] **Step 4.1**: 编写使用文档
  - [ ] Tick 触发点说明
  - [ ] SmartTickManager API 文档
  - [ ] 常见问题 FAQ

- [ ] **Step 4.2**: 开发调试工具
  - [ ] Runtime Watcher 显示 Tick 历史
  - [ ] 可视化 Tick 触发链
  - [ ] 性能分析器

---

## 测试方案

### 1. 单元测试

```csharp
[TestFixture]
public class SmartTickManagerTests
{
    private SmartTickManager tickManager;
    private MockASC mockASC;
    
    [SetUp]
    public void Setup()
    {
        tickManager = new SmartTickManager();
        mockASC = new MockASC();
    }
    
    [Test]
    public void MarkDirty_ShouldAvoidDuplicates()
    {
        // Arrange
        tickManager.MarkDirty(mockASC, "Reason1");
        tickManager.MarkDirty(mockASC, "Reason2");
        
        // Act
        tickManager.FlushDirtyASCs();
        
        // Assert
        Assert.AreEqual(1, mockASC.TickCallCount);
    }
    
    [Test]
    public void TickImmediate_ShouldExecuteImmediately()
    {
        // Act
        tickManager.TickImmediate(mockASC);
        
        // Assert
        Assert.AreEqual(1, mockASC.TickCallCount);
    }
    
    [Test]
    public void FlushDirtyASCs_ShouldClearQueue()
    {
        // Arrange
        tickManager.MarkDirty(mockASC);
        tickManager.FlushDirtyASCs();
        
        // Act
        tickManager.FlushDirtyASCs(); // 第二次调用
        
        // Assert
        Assert.AreEqual(1, mockASC.TickCallCount); // 应该只调用了一次
    }
}
```

---

### 2. 集成测试

```csharp
[TestFixture]
public class BattleFlowTickTests
{
    private TurnBasedBattleManager battleManager;
    
    [SetUp]
    public void Setup()
    {
        // 创建测试战斗
        battleManager = CreateTestBattle(2, 2); // 2v2
    }
    
    [Test]
    public void BattleStart_ShouldTickAllUnits()
    {
        // Arrange
        var units = battleManager.AllUnits;
        
        // Act
        battleManager.InitBattle();
        
        // Assert
        foreach (var unit in units)
        {
            Assert.IsTrue(unit.WasTickedOnBattleStart);
        }
    }
    
    [Test]
    public void TurnStart_ShouldTickCurrentUnit()
    {
        // Arrange
        battleManager.InitBattle();
        var firstUnit = battleManager.AllUnits[0];
        
        // Act
        battleManager.StartBattle();
        
        // Assert
        Assert.IsTrue(firstUnit.WasTickedOnTurnStart);
    }
    
    [Test]
    public void ExecuteAbility_ShouldTickSourceAndTarget()
    {
        // Arrange
        battleManager.InitBattle();
        battleManager.StartBattle();
        var actor = battleManager.GetCurrentUnit();
        var target = battleManager.AllUnits.First(u => u.Team != actor.Team);
        
        // Act
        battleManager.ExecuteAbility(actor, "BasicAttack", target.ASC);
        
        // Assert
        Assert.IsTrue(actor.WasTickedAfterAbility);
        Assert.IsTrue(target.WasTickedAfterAbility);
    }
}
```

---

### 3. 性能测试

```csharp
[TestFixture]
public class TickPerformanceTests
{
    [Test]
    public void LargeBattle_ShouldCompleteInReasonableTime()
    {
        // Arrange
        var battleManager = CreateTestBattle(10, 10); // 10v10
        
        // Act
        var stopwatch = System.Diagnostics.Stopwatch.StartNew();
        battleManager.InitBattle();
        battleManager.StartBattle();
        
        // 模拟 100 回合
        for (int i = 0; i < 100; i++)
        {
            SimulateTurn(battleManager);
        }
        
        stopwatch.Stop();
        
        // Assert
        Assert.Less(stopwatch.ElapsedMilliseconds, 1000); // 应在 1 秒内完成
    }
    
    [Test]
    public void PausedState_ShouldHaveZeroCPUUsage()
    {
        // Arrange
        var battleManager = CreateTestBattle(5, 5);
        battleManager.InitBattle();
        
        // Act
        var beforeCPU = GetCPUUsage();
        System.Threading.Thread.Sleep(1000); // 等待 1 秒
        var afterCPU = GetCPUUsage();
        
        // Assert
        Assert.Less(afterCPU - beforeCPU, 0.1f); // CPU 增长应小于 0.1%
    }
}
```

---

## 风险与应对

### 风险 1：遗漏 Tick 调用点

**风险描述**：
开发者可能在新增功能时忘记在关键点调用 Tick，导致 Buff 不更新、技能效果不生效。

**影响等级**：⚠️ 中

**应对策略**：

1. **代码审查清单**
   ```
   [ ] 技能执行前是否 Tick？
   [ ] 技能执行后是否 Tick？
   [ ] 回合开始是否 Tick？
   [ ] 回合结束是否 Tick？
   [ ] AOE 技能是否 Tick 所有目标？
   ```

2. **自动化检查工具**
   ```csharp
   public class TickPointValidator
   {
       public static void ValidateBattleFlow(TurnBasedBattleManager manager)
       {
           var missingPoints = new List<string>();
           
           if (!manager.TickOnBattleStart)
               missingPoints.Add("BattleStart");
           
           if (!manager.TickOnTurnStart)
               missingPoints.Add("TurnStart");
           
           // ...
           
           if (missingPoints.Any())
           {
               Debug.LogWarning($"Missing Tick points: {string.Join(", ", missingPoints)}");
           }
       }
   }
   ```

3. **单元测试覆盖**
   - 为每个 Tick 触发点编写测试
   - CI/CD 中强制运行测试

---

### 风险 2：Tick 顺序错误

**风险描述**：
在错误的时机 Tick，导致效果结算顺序不符合预期（如：先扣血后检查死亡）。

**影响等级**：⚠️ 中

**应对策略**：

1. **明确 Tick 顺序规范**
   ```
   正确顺序：
   1. 回合开始 Tick → 处理回合开始 Buff
   2. 检查控制状态 → 决定是否跳过
   3. 执行技能 → 应用效果
   4. 技能后 Tick → 确保效果生效
   5. 回合结束 Tick → 处理回合结束 Buff
   6. 检查死亡 → 移除死亡单位
   ```

2. **日志追踪**
   ```csharp
   TickManager.EnableLogging = true; // 开启详细日志
   ```

3. **集成测试验证**
   - 测试"中毒 → 扣血 → 死亡"完整流程
   - 测试"治疗 → 回血 → 复活"流程

---

### 风险 3：性能问题

**风险描述**：
大规模战斗（20v20）时，频繁 Tick 可能导致卡顿。

**影响等级**：⚠️ 低（回合制对性能要求不高）

**应对策略**：

1. **延迟批量 Tick**
   ```csharp
   public void TickAllDelayed(List<ASC> ascs, float delay)
   {
       StartCoroutine(TickAllCoroutine(ascs, delay));
   }
   
   IEnumerator TickAllCoroutine(List<ASC> ascs, float delay)
   {
       foreach (var asc in ascs)
       {
           asc.Tick();
           yield return new WaitForSeconds(delay);
       }
   }
   ```

2. **条件 Tick**
   ```csharp
   // 只 Tick 有 Buff 的单位
   TickManager.TickWhere(AllUnits, u => u.HasAnyBuffs(), "BuffUpdate");
   ```

3. **性能监控**
   ```csharp
   var stopwatch = Stopwatch.StartNew();
   TickManager.FlushDirtyASCs();
   stopwatch.Stop();
   
   if (stopwatch.ElapsedMilliseconds > 16)
   {
       Debug.LogWarning($"Tick took {stopwatch.ElapsedMilliseconds}ms");
   }
   ```

---

### 风险 4：与 TimelineAbility 不兼容

**风险描述**：
如果项目需要使用 TimelineAbility，暂停 Tick 会导致动画无法播放。

**影响等级**：⚠️ 低（回合制通常不用 TimelineAbility）

**应对策略**：

1. **临时启用 Tick**
   ```csharp
   public void PlayTimelineAbility(AbilitySpec ability)
   {
       // 临时启用自动 Tick
       GameplayAbilitySystem.GAS.Unpause();
       
       ability.Activate();
       
       // 等待动画结束
       StartCoroutine(PauseAfterAnimation(ability.Duration));
   }
   ```

2. **使用混合模式**（参见方案 C）

3. **重新设计技能表现**
   - 使用 Animator + Coroutine 替代 TimelineAbility
   - 将动画和逻辑分离

---

## 总结

### 推荐方案：方案 A（全局暂停 + 手动触发）

**核心优势**：
1. ✅ 零侵入框架核心代码
2. ✅ 性能最优（按需执行）
3. ✅ 易于调试和追踪
4. ✅ 网络同步友好
5. ✅ 确定性强（不依赖帧率）

**实施建议**：
1. 优先实现 `SmartTickManager`
2. 在战斗管理器中明确定义 Tick 触发点
3. 编写完善的测试用例
4. 添加详细的日志和调试工具

**预期工作量**：
- 开发：4-5 天
- 测试：2-3 天
- 文档：1 天
- **总计**：7-9 天

**成功指标**：
- ✅ 暂停状态下 CPU 占用 < 1%
- ✅ 所有 Buff 在正确时机结算
- ✅ 完整战斗流程无卡顿
- ✅ 单元测试覆盖率 > 80%

---

**下一步**：
阅读 [P0-2: Buff 生命周期改造](./P0-2-BuffLifecycle-Reform.md) 文档，了解如何将 Duration/Period 从秒数改为回合计数。

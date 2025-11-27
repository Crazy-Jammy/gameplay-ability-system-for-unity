# P0-3: 技能冷却改造 (Ability Cooldown Reform)

**文档版本**: 1.0  
**创建日期**: 2025-11-27  
**状态**: ✅ 设计完成  
**优先级**: ⭐⭐⭐ P0 - 框架基础

---

## 📋 目录

1. [文档概述](#一文档概述)
2. [核心需求与设计原则](#二核心需求与设计原则)
3. [TurnBasedAbilityCooldownManager 核心设计](#三turnbasedabilitycooldownmanager-核心设计)
4. [冷却机制详细设计](#四冷却机制详细设计)
5. [资源消耗系统 (Cost GE)](#五资源消耗系统-cost-ge)
6. [减 CD 与特殊机制](#六减cd与特殊机制)
7. [与 AbilitySpec 集成](#七与abilityspec集成)
8. [优缺点、扩展性和测试方案](#八优缺点扩展性和测试方案)
9. [备选方案对比](#九备选方案对比)
10. [实现步骤与完整示例](#十实现步骤与完整示例)
11. [附录：快速参考](#附录快速参考)

---

## 一、文档概述

### 1.1 背景

**核心问题**：
- EX-GAS 框架的 `Ability` 系统**没有内置冷却机制**
- 原生 GAS 没有时间概念，需要开发者自行设计
- 回合制游戏需要**回合计数的冷却**，而非实时秒数

**当前状态**：
```csharp
public abstract class AbilitySpec
{
    // 只有基础的激活检查
    public virtual AbilityActivateResult CanActivate()
    {
        // 检查 Owner、Cost GE 等
        // ❌ 没有冷却检查
    }
}
```

**目标**：
创建回合制冷却管理器，支持：
- ✅ 回合计数冷却（3 回合 CD）
- ✅ 次数限制（每回合限 1 次）
- ✅ 战斗总次数（每战斗限 3 次）
- ✅ 资源消耗（Mana、体力等）
- ✅ 减 CD 机制（Buff 降低冷却）
- ✅ 共享 CD 组（同组技能共享 CD）

---

### 1.2 文档结构说明

本文档分为 10 个主要章节：

**基础部分**（第 1-2 章）：
- 问题分析、核心需求、设计原则

**核心解决方案**（第 3-7 章）：
- CooldownManager 完整实现
- 三种冷却机制
- Cost GE 资源消耗
- 减 CD 特殊机制
- AbilitySpec 集成

**分析与实战**（第 8-10 章）：
- 优缺点、扩展性、测试
- 备选方案对比
- 3 天实现计划 + 完整战斗示例

---

## 二、核心需求与设计原则

### 2.1 GAS 原生冷却机制分析

**UE4 GAS 的冷却设计**：
- 使用 `GameplayEffect` 的 `DurationPolicy` 模拟冷却
- 通过 `CooldownTags` 标记技能冷却状态
- 示例：技能 CD 3 秒 → 施加持续 3 秒的 `Cooldown.Skill.Fireball` Tag

**EX-GAS 当前状态**：
- ❌ 没有内置 `CooldownTags` 机制
- ❌ 没有 `GameplayEffect` 的 Duration 管理（已在 P0-2 改造）
- ✅ 但保留了 `GameplayTag` 系统（可以用于标记冷却）

**为什么不用 GE 模拟 CD**：
- ❌ 回合制不需要时间流逝（已暂停 Tick）
- ❌ GE 的 Duration 已改为回合计数（P0-2）
- ❌ 冷却状态与 Buff 本质不同（冷却是技能状态，Buff 是单位状态）

---

### 2.2 回合制冷却的 5 大核心问题

| 问题 | 描述 | 示例 |
|------|------|------|
| **1. 冷却计数单位** | 秒数 → 回合数 | 3 秒 CD → 3 回合 CD |
| **2. 冷却递减时机** | 何时减少 CD？ | 回合开始？回合结束？ |
| **3. 次数限制** | 每回合限用次数 | 普通攻击无限制，大招每回合限 1 次 |
| **4. 战斗总次数** | 每战斗限用次数 | 终极技能每战斗限 3 次 |
| **5. 资源消耗** | Mana、体力等 | 释放技能消耗 50 Mana |

---

### 2.3 核心改造需求

#### 需求 1：回合冷却系统

```csharp
// 技能配置
public class FireballAbility : AbilityAsset
{
    public int CooldownTurns = 3; // 3 回合冷却
}

// 使用后
playerASC.TryActivateAbility("Fireball", target);
// → 技能进入 3 回合冷却
// 回合 1 结束：CD 剩余 2
// 回合 2 结束：CD 剩余 1
// 回合 3 结束：CD 剩余 0，可再次使用
```

---

#### 需求 2：次数限制

```csharp
public class UltimateAbility : AbilityAsset
{
    public int MaxUsesPerTurn = 1;        // 每回合限 1 次
    public int MaxUsesPerBattle = 3;      // 每战斗限 3 次
}

// 使用场景
playerASC.TryActivateAbility("Ultimate", target); // 成功
playerASC.TryActivateAbility("Ultimate", target); // 失败：本回合已用过

// 下回合
playerASC.TryActivateAbility("Ultimate", target); // 成功
// ...战斗中第 3 次使用后
playerASC.TryActivateAbility("Ultimate", target); // 失败：战斗总次数已用完
```

---

#### 需求 3：资源消耗

```csharp
public class ManaSkill : AbilityAsset
{
    public GameplayEffect ManaCostEffect; // Instant GE, Add -50 to Mana
}

// 检查 Mana
if (currentMana < 50) 
{
    // 无法释放
}

// 消耗 Mana
Owner.ApplyGameplayEffectTo(ManaCostEffect, Owner);
```

---

#### 需求 4：减 CD 机制

```csharp
// Buff 效果：所有技能冷却 -1 回合
cooldownManager.ReduceAllCooldowns(playerASC, 1);

// 特定技能冷却重置
cooldownManager.ResetCooldown(fireballAbilitySpec);
```

---

### 2.4 设计原则

#### 原则 1：集中管理

**理念**：冷却状态由 `TurnBasedAbilityCooldownManager` 统一管理，而非分散在各个 `AbilitySpec` 中

**为什么**：
- ✅ 便于实现全局操作（重置所有 CD、减 CD Buff）
- ✅ 易于 UI 查询（显示所有技能冷却）
- ✅ 易于序列化保存

**对比**：
```csharp
// ❌ 分散管理（不推荐）
public class FireballAbilitySpec : AbilitySpec
{
    private int remainingCooldown; // 每个技能自己管理
}

// ✅ 集中管理（推荐）
public class TurnBasedAbilityCooldownManager
{
    private Dictionary<AbilitySpec, int> cooldowns; // 统一管理所有技能
}
```

---

#### 原则 2：易于扩展

**支持的扩展场景**：
- 共享 CD 组（同组技能共享冷却）
- 全局 CD（所有技能共享）
- CD 回复速度调整（Buff 加速冷却恢复）
- 条件 CD（特定情况下 CD 减半）

---

#### 原则 3：与 GAS 理念一致

**资源消耗用 Cost GE**：
```csharp
// ✅ 符合 GAS 理念
Owner.ApplyGameplayEffectTo(ManaCostEffect, Owner);

// ❌ 不符合 GAS 理念
Owner.AttrSet<AS_Combat>().Mana.CurrentValue -= 50;
```

**为什么**：
- ✅ 只有 GE 可以修改属性（GAS 黄金法则）
- ✅ 支持消耗减免（Buff 降低 Mana 消耗 20%）
- ✅ 便于追踪和调试

---

#### 原则 4：明确的递减时机

**冷却递减时机**：回合**结束时**

**为什么选择回合结束**：
```
回合 1:
- 使用技能 Fireball（CD 3 回合）
- 回合结束：CD 变为 2

回合 2:
- 回合开始：CD 仍为 2（不可用）
- 回合结束：CD 变为 1

回合 3:
- 回合开始：CD 仍为 1（不可用）
- 回合结束：CD 变为 0

回合 4:
- 回合开始：CD 为 0（✅ 可用）
```

**设计理由**：
- ✅ 玩家直观理解（"冷却 3 回合" = 需要等 3 个回合结束）
- ✅ 与 Buff 计数一致（回合结束时统一更新）
- ✅ 避免"当前回合立即可用"的混乱

---

### 2.5 核心设计总结

**TurnBasedAbilityCooldownManager** 是一个：
- **集中式管理器**：统一管理所有技能冷却状态
- **多维度限制**：支持回合 CD、次数限制、战斗总次数
- **易于扩展**：支持减 CD、共享 CD、全局 CD
- **符合 GAS 理念**：资源消耗用 Cost GE

**与其他系统的关系**：
- 依赖 `TurnBasedBattleManager`（回合结束时递减 CD）
- 集成到 `AbilitySpec`（`CanActivate()` 检查冷却）
- 配合 `Cost GE`（资源消耗）
- 支持 `GameplayCue`（减 CD Buff）

---

## 三、TurnBasedAbilityCooldownManager 核心设计

### 3.1 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│           TurnBasedAbilityCooldownManager                   │
│  (技能冷却管理器 - 集中管理所有技能冷却状态)                  │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 回合冷却字典  │    │ 回合使用字典  │    │ 战斗使用字典  │
│ cooldowns    │    │turnUsageCount│    │battleUsage   │
│              │    │              │    │   Count      │
│AbilitySpec   │    │string        │    │AbilitySpec   │
│  ↓           │    │ abilityName  │    │  ↓           │
│int remaining │    │  ↓           │    │int usedCount │
│   Cooldown   │    │int usedCount │    │              │
└──────────────┘    └──────────────┘    └──────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                ┌─────────────┴─────────────┐
                │      核心 API 方法         │
                ├──────────────────────────┤
                │ IsOnCooldown()           │ 查询冷却状态
                │ CanUseThisTurn()         │ 检查次数限制
                │ StartCooldown()          │ 设置冷却
                │ DecrementCooldowns()     │ 回合结束递减
                │ RecordUsage()            │ 记录使用
                │ ResetTurnUsage()         │ 回合开始重置
                │ ReduceCooldown()         │ 减 CD
                │ ResetCooldown()          │ 重置 CD
                └──────────────────────────┘
                              │
                ┌─────────────┴─────────────┐
                │      集成点               │
                ├──────────────────────────┤
                │ AbilitySpec.CanActivate()│ ← 检查冷却
                │ AbilitySpec.Activate()   │ → 记录使用
                │ BattleManager.TurnEnd()  │ → 递减 CD
                │ BattleManager.TurnStart()│ → 重置次数
                │ GameplayCue             │ → 减 CD Buff
                └──────────────────────────┘
```

---

### 3.2 核心数据结构

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using UnityEngine;
using GAS;

/// <summary>
/// 回合制技能冷却管理器
/// 集中管理所有技能的冷却状态、使用次数限制
/// </summary>
public class TurnBasedAbilityCooldownManager : MonoBehaviour
{
    #region 数据结构
    
    /// <summary>
    /// 技能冷却字典
    /// Key: AbilitySpec（技能实例）
    /// Value: 剩余冷却回合数
    /// </summary>
    private Dictionary<AbilitySpec, int> cooldowns = new Dictionary<AbilitySpec, int>();
    
    /// <summary>
    /// 回合使用次数字典
    /// Key: 技能名称（允许同类技能共享次数限制）
    /// Value: 本回合已使用次数
    /// </summary>
    private Dictionary<string, int> turnUsageCount = new Dictionary<string, int>();
    
    /// <summary>
    /// 战斗总使用次数字典
    /// Key: AbilitySpec（每个技能实例独立计数）
    /// Value: 战斗中已使用次数
    /// </summary>
    private Dictionary<AbilitySpec, int> battleUsageCount = new Dictionary<AbilitySpec, int>();
    
    /// <summary>
    /// 共享冷却组字典
    /// Key: 冷却组名称
    /// Value: 剩余冷却回合数
    /// </summary>
    private Dictionary<string, int> sharedCooldownGroups = new Dictionary<string, int>();
    
    #endregion
    
    #region 调试开关
    
    [Header("调试设置")]
    public bool EnableDebugLog = true;
    
    private void DebugLog(string message)
    {
        if (EnableDebugLog)
        {
            Debug.Log($"<color=orange>[CooldownMgr]</color> {message}");
        }
    }
    
    #endregion
    
    // ... 方法实现见下文
}
```

**为什么这样设计**：

1. **三层冷却机制**：
   - `cooldowns`：技能冷却（技能特定）
   - `turnUsageCount`：回合次数（技能名称共享）
   - `battleUsageCount`：战斗总次数（实例独立）

2. **Key 的选择**：
   - `AbilitySpec` 用于技能冷却（每个技能实例独立）
   - `string` 用于回合次数（同名技能共享限制）
   - `AbilitySpec` 用于战斗次数（实例独立计数）

3. **扩展性**：
   - `sharedCooldownGroups`：支持共享 CD 组（预留）

---

### 3.3 完整类实现

#### 3.3.1 冷却检查 API

```csharp
public class TurnBasedAbilityCooldownManager : MonoBehaviour
{
    // ... 数据结构
    
    /// <summary>
    /// 检查技能是否在冷却中
    /// </summary>
    public bool IsOnCooldown(AbilitySpec ability)
    {
        if (ability == null) return false;
        
        if (cooldowns.ContainsKey(ability))
        {
            return cooldowns[ability] > 0;
        }
        
        return false;
    }
    
    /// <summary>
    /// 获取剩余冷却回合数
    /// </summary>
    public int GetRemainingCooldown(AbilitySpec ability)
    {
        if (ability == null) return 0;
        
        if (cooldowns.ContainsKey(ability))
        {
            return cooldowns[ability];
        }
        
        return 0;
    }
    
    /// <summary>
    /// 检查本回合是否还能使用（次数限制）
    /// </summary>
    public bool CanUseThisTurn(string abilityName, int maxUsesPerTurn)
    {
        if (maxUsesPerTurn <= 0) return true; // 无限制
        
        if (!turnUsageCount.ContainsKey(abilityName))
        {
            return true; // 本回合未使用过
        }
        
        return turnUsageCount[abilityName] < maxUsesPerTurn;
    }
    
    /// <summary>
    /// 检查战斗中是否还能使用（战斗总次数限制）
    /// </summary>
    public bool CanUseBattle(AbilitySpec ability, int maxUsesPerBattle)
    {
        if (ability == null) return false;
        if (maxUsesPerBattle <= 0) return true; // 无限制
        
        if (!battleUsageCount.ContainsKey(ability))
        {
            return true; // 战斗中未使用过
        }
        
        return battleUsageCount[ability] < maxUsesPerBattle;
    }
}
```

---

#### 3.3.2 冷却设置与更新 API

```csharp
public class TurnBasedAbilityCooldownManager : MonoBehaviour
{
    // ... 检查 API
    
    /// <summary>
    /// 设置技能冷却
    /// </summary>
    public void StartCooldown(AbilitySpec ability, int cooldownTurns)
    {
        if (ability == null || cooldownTurns <= 0) return;
        
        cooldowns[ability] = cooldownTurns;
        
        DebugLog($"技能 {ability.Ability.Name} 进入冷却，剩余 {cooldownTurns} 回合");
    }
    
    /// <summary>
    /// 记录技能使用（回合次数 + 战斗次数）
    /// </summary>
    public void RecordUsage(AbilitySpec ability)
    {
        if (ability == null) return;
        
        // 记录回合使用次数
        string abilityName = ability.Ability.Name;
        if (!turnUsageCount.ContainsKey(abilityName))
        {
            turnUsageCount[abilityName] = 0;
        }
        turnUsageCount[abilityName]++;
        
        // 记录战斗总使用次数
        if (!battleUsageCount.ContainsKey(ability))
        {
            battleUsageCount[ability] = 0;
        }
        battleUsageCount[ability]++;
        
        DebugLog($"记录使用：{abilityName}（本回合 {turnUsageCount[abilityName]} 次，战斗总计 {battleUsageCount[ability]} 次）");
    }
    
    /// <summary>
    /// 回合结束：递减所有冷却
    /// </summary>
    public void DecrementCooldowns(AbilitySystemComponent unit)
    {
        if (unit == null) return;
        
        // 获取该单位的所有技能
        var abilities = unit.AbilityContainer.AbilitySpecs().Values
            .Where(a => cooldowns.ContainsKey(a))
            .ToList();
        
        foreach (var ability in abilities)
        {
            if (cooldowns[ability] > 0)
            {
                cooldowns[ability]--;
                
                DebugLog($"{unit.name} 的 {ability.Ability.Name} 冷却递减，剩余 {cooldowns[ability]} 回合");
                
                if (cooldowns[ability] <= 0)
                {
                    cooldowns.Remove(ability);
                    DebugLog($"{ability.Ability.Name} 冷却结束");
                }
            }
        }
    }
    
    /// <summary>
    /// 回合开始：重置本回合使用次数
    /// </summary>
    public void ResetTurnUsage()
    {
        if (turnUsageCount.Count > 0)
        {
            DebugLog($"重置回合使用次数（{turnUsageCount.Count} 个技能）");
            turnUsageCount.Clear();
        }
    }
}
```

---

#### 3.3.3 减 CD 与特殊操作 API

```csharp
public class TurnBasedAbilityCooldownManager : MonoBehaviour
{
    // ... 基础 API
    
    /// <summary>
    /// 减少单个技能冷却
    /// </summary>
    public void ReduceCooldown(AbilitySpec ability, int turns)
    {
        if (ability == null || turns <= 0) return;
        
        if (cooldowns.ContainsKey(ability))
        {
            int oldCooldown = cooldowns[ability];
            cooldowns[ability] = Mathf.Max(0, cooldowns[ability] - turns);
            
            DebugLog($"{ability.Ability.Name} 冷却减少 {turns} 回合：{oldCooldown} → {cooldowns[ability]}");
            
            if (cooldowns[ability] <= 0)
            {
                cooldowns.Remove(ability);
                DebugLog($"{ability.Ability.Name} 冷却已重置");
            }
        }
    }
    
    /// <summary>
    /// 减少单位所有技能冷却
    /// </summary>
    public void ReduceAllCooldowns(AbilitySystemComponent unit, int turns)
    {
        if (unit == null || turns <= 0) return;
        
        var abilities = unit.AbilityContainer.AbilitySpecs().Values
            .Where(a => cooldowns.ContainsKey(a))
            .ToList();
        
        DebugLog($"减少 {unit.name} 所有技能冷却 {turns} 回合");
        
        foreach (var ability in abilities)
        {
            ReduceCooldown(ability, turns);
        }
    }
    
    /// <summary>
    /// 重置单个技能冷却
    /// </summary>
    public void ResetCooldown(AbilitySpec ability)
    {
        if (ability == null) return;
        
        if (cooldowns.ContainsKey(ability))
        {
            cooldowns.Remove(ability);
            DebugLog($"{ability.Ability.Name} 冷却已重置");
        }
    }
    
    /// <summary>
    /// 重置单位所有技能冷却
    /// </summary>
    public void ResetAllCooldowns(AbilitySystemComponent unit)
    {
        if (unit == null) return;
        
        var abilities = unit.AbilityContainer.AbilitySpecs().Values
            .Where(a => cooldowns.ContainsKey(a))
            .ToList();
        
        DebugLog($"重置 {unit.name} 所有技能冷却");
        
        foreach (var ability in abilities)
        {
            cooldowns.Remove(ability);
        }
    }
    
    /// <summary>
    /// 战斗开始：清空所有计数
    /// </summary>
    public void OnBattleStart()
    {
        cooldowns.Clear();
        turnUsageCount.Clear();
        battleUsageCount.Clear();
        sharedCooldownGroups.Clear();
        
        DebugLog("战斗开始，清空所有冷却数据");
    }
    
    /// <summary>
    /// 战斗结束：清空所有计数
    /// </summary>
    public void OnBattleEnd()
    {
        cooldowns.Clear();
        turnUsageCount.Clear();
        battleUsageCount.Clear();
        
        DebugLog("战斗结束，清空所有冷却数据");
    }
}
```

---

### 3.4 基础使用示例

#### 示例 1：普通技能（有冷却）

```csharp
// 1. 在 AbilityAsset 中配置冷却
public class FireballAbilityAsset : AbilityAsset
{
    [Header("冷却设置")]
    public int CooldownTurns = 3; // 3 回合冷却
}

// 2. 在 AbilitySpec 中集成冷却管理器
public class FireballAbilitySpec : AbilitySpec<FireballAbility>
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    public FireballAbilitySpec(FireballAbility ability, AbilitySystemComponent owner) 
        : base(ability, owner)
    {
        cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
    }
    
    public override AbilityActivateResult CanActivate()
    {
        // 检查基础条件
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success)
        {
            return baseResult;
        }
        
        // 检查冷却
        if (cooldownMgr.IsOnCooldown(this))
        {
            Debug.Log($"技能冷却中，剩余 {cooldownMgr.GetRemainingCooldown(this)} 回合");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // 技能效果
        Owner.ApplyGameplayEffectTo(Ability.DataReference.DamageEffect, target);
        
        // 设置冷却
        var asset = Ability.DataReference as FireballAbilityAsset;
        cooldownMgr.StartCooldown(this, asset.CooldownTurns);
        
        EndAbility();
    }
}

// 3. 在战斗管理器中递减冷却
public class TurnBasedBattleManager : MonoBehaviour
{
    public TurnBasedAbilityCooldownManager CooldownManager { get; private set; }
    
    void EndCurrentTurn()
    {
        var currentUnit = AllUnits[currentUnitIndex];
        
        // 回合结束时递减冷却
        CooldownManager.DecrementCooldowns(currentUnit.ASC);
        
        // ...
    }
}
```

**使用流程**：
```
回合 1: 使用 Fireball → 进入 3 回合冷却
回合 1 结束: CD 剩余 2
回合 2 结束: CD 剩余 1
回合 3 结束: CD 剩余 0
回合 4: 可再次使用 Fireball ✅
```

---

#### 示例 2：次数限制技能

```csharp
// 1. 配置次数限制
public class UltimateAbilityAsset : AbilityAsset
{
    [Header("使用限制")]
    public int MaxUsesPerTurn = 1;        // 每回合限 1 次
    public int MaxUsesPerBattle = 3;      // 每战斗限 3 次
    public int CooldownTurns = 5;         // 5 回合冷却
}

// 2. 检查次数限制
public class UltimateAbilitySpec : AbilitySpec<UltimateAbility>
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        var asset = Ability.DataReference as UltimateAbilityAsset;
        
        // 检查冷却
        if (cooldownMgr.IsOnCooldown(this))
        {
            return AbilityActivateResult.Fail;
        }
        
        // 检查本回合次数限制
        if (!cooldownMgr.CanUseThisTurn(Ability.Name, asset.MaxUsesPerTurn))
        {
            Debug.Log("本回合已使用过此技能");
            return AbilityActivateResult.Fail;
        }
        
        // 检查战斗总次数限制
        if (!cooldownMgr.CanUseBattle(this, asset.MaxUsesPerBattle))
        {
            Debug.Log("战斗中已达到使用次数上限");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as UltimateAbilityAsset;
        
        // 技能效果
        Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        
        // 记录使用
        cooldownMgr.RecordUsage(this);
        
        // 设置冷却
        cooldownMgr.StartCooldown(this, asset.CooldownTurns);
        
        EndAbility();
    }
}
```

**使用流程**：
```
回合 1: 使用 Ultimate → 进入冷却，记录使用 (1/3)
回合 1: 再次使用 Ultimate → ❌ 失败（本回合限 1 次）
回合 6: 冷却结束，使用 Ultimate → 记录使用 (2/3)
回合 11: 使用 Ultimate → 记录使用 (3/3)
回合 16: 冷却结束，尝试使用 → ❌ 失败（战斗总次数已用完）
```

---

#### 示例 3：无冷却技能（普通攻击）

```csharp
// 1. 配置无冷却
public class BasicAttackAbilityAsset : AbilityAsset
{
    [Header("冷却设置")]
    public int CooldownTurns = 0; // 无冷却
}

// 2. 简单实现
public class BasicAttackAbilitySpec : AbilitySpec<BasicAttackAbility>
{
    public override AbilityActivateResult CanActivate()
    {
        // 普通攻击无冷却，只检查基础条件
        return base.CanActivate();
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // 技能效果
        Owner.ApplyGameplayEffectTo(Ability.DataReference.DamageEffect, target);
        
        // 无需设置冷却
        
        EndAbility();
    }
}
```

**使用流程**：
```
回合 1: 使用 BasicAttack → 成功
回合 1: 再次使用 BasicAttack → 成功（无冷却，无限制）
回合 2: 使用 BasicAttack → 成功
```

---

### 3.5 核心设计总结

**TurnBasedAbilityCooldownManager 特性**：

| 特性 | 说明 | 价值 |
|------|------|------|
| **集中管理** | 所有技能冷却状态统一管理 | 易于全局操作（减 CD、重置 CD） |
| **三层限制** | 冷却、回合次数、战斗次数 | 覆盖所有设计需求 |
| **易于集成** | 在 `CanActivate()` 检查即可 | 开发者友好 |
| **易于查询** | UI 可直接查询冷却状态 | 实时显示技能 CD |
| **易于扩展** | 支持减 CD、共享 CD 等 | 满足复杂设计 |
| **易于序列化** | 三个字典可直接保存 | 存档系统友好 |

**关键 API 速查**：

```csharp
// 检查
bool isOnCD = cooldownMgr.IsOnCooldown(ability);
int remaining = cooldownMgr.GetRemainingCooldown(ability);
bool canUse = cooldownMgr.CanUseThisTurn(abilityName, maxPerTurn);
bool canUseBattle = cooldownMgr.CanUseBattle(ability, maxPerBattle);

// 设置
cooldownMgr.StartCooldown(ability, turns);
cooldownMgr.RecordUsage(ability);

// 更新
cooldownMgr.DecrementCooldowns(unit);   // 回合结束
cooldownMgr.ResetTurnUsage();           // 回合开始

// 特殊操作
cooldownMgr.ReduceCooldown(ability, turns);
cooldownMgr.ResetCooldown(ability);
cooldownMgr.ReduceAllCooldowns(unit, turns);
cooldownMgr.ResetAllCooldowns(unit);
```

---

## 四、冷却机制详细设计

### 4.1 三种冷却机制对比

| 冷却机制 | 用途 | 计数单位 | 重置时机 | 典型技能 |
|---------|------|---------|---------|---------|
| **回合冷却** | 限制技能使用频率 | 回合数 | 回合结束递减 | 火球术（3 回合 CD） |
| **回合次数限制** | 限制单回合使用次数 | 次数 | 回合开始重置 | 大招（每回合限 1 次） |
| **战斗总次数** | 限制整场战斗使用次数 | 次数 | 战斗结束清空 | 终极技（每战斗限 3 次） |

**设计理由**：

- **回合冷却**：最常见的限制方式，模拟技能"充能时间"
- **回合次数**：防止同一技能在一个回合内连续使用（破坏平衡）
- **战斗总次数**：限制强力技能的总使用次数，增加策略性

---

### 4.2 回合冷却（Turn Cooldown）

#### 4.2.1 核心机制

**定义**：技能使用后，需要等待 N 个回合才能再次使用

**计数规则**：
```
使用技能 → 设置 CD = N
回合 1 结束 → CD = N - 1
回合 2 结束 → CD = N - 2
...
回合 N 结束 → CD = 0（可用）
```

**时间线示例**（3 回合冷却）：
```
玩家回合 1:
  - 使用 Fireball（CD 3）
  - 回合结束：CD 变为 2

玩家回合 2:
  - 回合开始：CD 仍为 2（❌ 不可用）
  - 尝试使用 Fireball → 失败
  - 回合结束：CD 变为 1

玩家回合 3:
  - 回合开始：CD 仍为 1（❌ 不可用）
  - 回合结束：CD 变为 0

玩家回合 4:
  - 回合开始：CD 为 0（✅ 可用）
  - 使用 Fireball → CD 重新设置为 3
```

---

#### 4.2.2 实现代码

```csharp
// 1. AbilityAsset 配置
public class FireballAbilityAsset : AbilityAsset
{
    [Header("冷却设置")]
    [Tooltip("技能冷却回合数")]
    public int CooldownTurns = 3;
}

// 2. AbilitySpec 集成
public class FireballAbilitySpec : AbilitySpec<FireballAbility>
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    public FireballAbilitySpec(FireballAbility ability, AbilitySystemComponent owner) 
        : base(ability, owner)
    {
        cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
    }
    
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        // 检查冷却
        if (cooldownMgr.IsOnCooldown(this))
        {
            int remaining = cooldownMgr.GetRemainingCooldown(this);
            Debug.Log($"技能冷却中，剩余 {remaining} 回合");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // 技能效果
        Owner.ApplyGameplayEffectTo(Ability.DataReference.DamageEffect, target);
        
        // 设置冷却
        var asset = Ability.DataReference as FireballAbilityAsset;
        cooldownMgr.StartCooldown(this, asset.CooldownTurns);
        
        EndAbility();
    }
}

// 3. 战斗管理器中递减冷却
public class TurnBasedBattleManager : MonoBehaviour
{
    void EndCurrentTurn()
    {
        var currentUnit = AllUnits[currentUnitIndex];
        
        // 回合结束时递减冷却
        CooldownManager.DecrementCooldowns(currentUnit.ASC);
        
        // 继续下一回合
        currentUnitIndex++;
        StartNextTurn();
    }
}
```

---

#### 4.2.3 使用场景

| 技能类型 | 冷却回合 | 设计意图 |
|---------|---------|---------|
| 普通攻击 | 0（无冷却） | 可无限使用 |
| 常规技能 | 2-3 回合 | 主要输出手段 |
| 强力技能 | 4-5 回合 | 关键时刻使用 |
| 终极技能 | 6-8 回合 | 扭转战局 |

**平衡建议**：
- 冷却越长，技能威力应越强
- 考虑战斗平均回合数（10-15 回合）设计冷却
- 冷却 > 5 回合的技能，建议配合减 CD 机制

---

### 4.3 回合次数限制（Usage Limitation）

#### 4.3.1 核心机制

**定义**：技能在单个回合内只能使用 N 次

**计数规则**：
```
回合开始 → 重置使用次数为 0
使用技能 → 次数 + 1
再次使用 → 如果次数 >= 上限，禁止使用
回合结束 → 次数保留（下回合开始重置）
```

**时间线示例**（每回合限 1 次）：
```
玩家回合 1:
  - 回合开始：使用次数重置为 0
  - 使用 Ultimate → 次数变为 1
  - 尝试再次使用 → ❌ 失败（本回合已用过）
  - 回合结束

玩家回合 2:
  - 回合开始：使用次数重置为 0
  - 使用 Ultimate → ✅ 成功
```

**为什么需要次数限制**：
- 防止同一技能在一个回合内连续使用（如：连续 3 次大招）
- 增加策略性（玩家需要选择使用时机）
- 平衡性（强力技能不应该短时间内多次触发）

---

#### 4.3.2 实现代码

```csharp
// 1. AbilityAsset 配置
public class UltimateAbilityAsset : AbilityAsset
{
    [Header("使用限制")]
    [Tooltip("每回合最大使用次数（0 = 无限制）")]
    public int MaxUsesPerTurn = 1;
    
    [Tooltip("冷却回合数")]
    public int CooldownTurns = 5;
}

// 2. AbilitySpec 集成
public class UltimateAbilitySpec : AbilitySpec<UltimateAbility>
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        var asset = Ability.DataReference as UltimateAbilityAsset;
        
        // 检查冷却
        if (cooldownMgr.IsOnCooldown(this))
        {
            return AbilityActivateResult.Fail;
        }
        
        // 检查本回合次数限制
        if (!cooldownMgr.CanUseThisTurn(Ability.Name, asset.MaxUsesPerTurn))
        {
            Debug.Log($"本回合已使用过 {Ability.Name}");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as UltimateAbilityAsset;
        
        // 技能效果
        Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        
        // 记录本回合使用
        cooldownMgr.RecordUsage(this);
        
        // 设置冷却
        cooldownMgr.StartCooldown(this, asset.CooldownTurns);
        
        EndAbility();
    }
}

// 3. 战斗管理器中重置次数
public class TurnBasedBattleManager : MonoBehaviour
{
    void StartNextTurn()
    {
        // 回合开始时重置使用次数
        CooldownManager.ResetTurnUsage();
        
        // 获取下一个行动者
        var currentUnit = TurnOrderSystem.GetNextActor();
        
        // ...
    }
}
```

---

#### 4.3.3 使用场景

| 技能类型 | 每回合限制 | 设计意图 |
|---------|-----------|---------|
| 普通攻击 | 无限制 | 基础输出 |
| 常规技能 | 无限制（但有 CD） | 灵活使用 |
| 大招 | 每回合限 1 次 | 防止连续释放 |
| 辅助技能 | 每回合限 2 次 | 增加策略性 |

**设计建议**：
- 大部分技能不需要次数限制（冷却已足够）
- 只对"瞬间爆发"的技能限制（如：无冷却的强力技能）
- 配合"减 CD"机制时，次数限制可以防止无限循环

---

### 4.4 战斗总次数限制（Battle Usage Limitation）

#### 4.4.1 核心机制

**定义**：技能在整场战斗中只能使用 N 次

**计数规则**：
```
战斗开始 → 战斗使用次数为 0
使用技能 → 次数 + 1
再次使用 → 如果次数 >= 上限，禁止使用
战斗结束 → 清空次数
```

**时间线示例**（每战斗限 3 次）：
```
战斗开始：使用次数 = 0

回合 1: 使用 LimitBreak → 次数 = 1
回合 6: 使用 LimitBreak → 次数 = 2
回合 11: 使用 LimitBreak → 次数 = 3
回合 16: 尝试使用 LimitBreak → ❌ 失败（已达上限）

战斗结束：清空次数
```

**为什么需要战斗总次数限制**：
- 限制超强技能的总使用次数
- 增加资源管理策略（何时使用珍贵的技能）
- 适合"限定资源"设定（如：魔法卷轴、终极技）

---

#### 4.4.2 实现代码

```csharp
// 1. AbilityAsset 配置
public class LimitBreakAbilityAsset : AbilityAsset
{
    [Header("使用限制")]
    [Tooltip("每战斗最大使用次数（0 = 无限制）")]
    public int MaxUsesPerBattle = 3;
    
    [Tooltip("每回合最大使用次数")]
    public int MaxUsesPerTurn = 1;
    
    [Tooltip("冷却回合数")]
    public int CooldownTurns = 4;
}

// 2. AbilitySpec 集成
public class LimitBreakAbilitySpec : AbilitySpec<LimitBreakAbility>
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        var asset = Ability.DataReference as LimitBreakAbilityAsset;
        
        // 检查冷却
        if (cooldownMgr.IsOnCooldown(this))
        {
            return AbilityActivateResult.Fail;
        }
        
        // 检查本回合次数限制
        if (!cooldownMgr.CanUseThisTurn(Ability.Name, asset.MaxUsesPerTurn))
        {
            Debug.Log("本回合已使用过此技能");
            return AbilityActivateResult.Fail;
        }
        
        // 检查战斗总次数限制
        if (!cooldownMgr.CanUseBattle(this, asset.MaxUsesPerBattle))
        {
            Debug.Log($"战斗中已达到 {Ability.Name} 使用上限");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as LimitBreakAbilityAsset;
        
        // 技能效果
        Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        
        // 记录使用（同时更新回合次数和战斗次数）
        cooldownMgr.RecordUsage(this);
        
        // 设置冷却
        cooldownMgr.StartCooldown(this, asset.CooldownTurns);
        
        // 显示剩余次数
        int used = cooldownMgr.GetBattleUsageCount(this);
        Debug.Log($"{Ability.Name} 剩余使用次数：{asset.MaxUsesPerBattle - used}");
        
        EndAbility();
    }
}

// 3. 战斗管理器中清空计数
public class TurnBasedBattleManager : MonoBehaviour
{
    void InitBattle()
    {
        // 战斗开始时清空所有计数
        CooldownManager.OnBattleStart();
        
        // ...
    }
    
    void OnBattleEnd(bool victory)
    {
        // 战斗结束时清空计数
        CooldownManager.OnBattleEnd();
        
        // ...
    }
}
```

**扩展实现**（支持查询战斗使用次数）：

```csharp
public class TurnBasedAbilityCooldownManager : MonoBehaviour
{
    // ... 已有代码
    
    /// <summary>
    /// 获取技能在战斗中已使用次数
    /// </summary>
    public int GetBattleUsageCount(AbilitySpec ability)
    {
        if (ability == null) return 0;
        
        if (battleUsageCount.ContainsKey(ability))
        {
            return battleUsageCount[ability];
        }
        
        return 0;
    }
}
```

---

#### 4.4.3 使用场景

| 技能类型 | 每战斗限制 | 设计意图 |
|---------|-----------|---------|
| 普通技能 | 无限制 | 常规使用 |
| 终极技能 | 每战斗限 3-5 次 | 关键时刻使用 |
| 消耗品技能 | 每战斗限 1 次 | 珍贵资源 |
| 复活技能 | 每战斗限 1 次 | 防止无限复活 |

**设计建议**：
- 只对极强力技能使用（如：一击必杀、全体复活）
- 配合 UI 显示剩余次数，增加紧迫感
- 考虑战斗平均回合数，避免限制过严

---

### 4.5 冷却递减时机分析

#### 4.5.1 三种可选时机

| 时机 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **回合开始** | 玩家直观（立即知道是否可用） | "CD 3 回合"实际只需等 2 个回合结束 | 简单游戏 |
| **回合结束**（推荐） | 符合直觉（"CD 3 回合"=等 3 个回合结束） | 需要玩家理解机制 | 复杂游戏 |
| **行动后** | 与 Buff 机制统一 | 玩家可能困惑 | 特殊设计 |

---

#### 4.5.2 推荐方案：回合结束递减

**实现逻辑**：
```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    void EndCurrentTurn()
    {
        var currentUnit = AllUnits[currentUnitIndex];
        
        // 1. 处理回合结束 Buff
        EffectManager.ProcessTurnEnd(currentUnit.ASC);
        
        // 2. 递减技能冷却
        CooldownManager.DecrementCooldowns(currentUnit.ASC);
        
        // 3. 检查胜负
        if (CheckBattleEnd()) return;
        
        // 4. 下一回合
        currentUnitIndex++;
        StartNextTurn();
    }
    
    void StartNextTurn()
    {
        // 回合开始时重置使用次数
        CooldownManager.ResetTurnUsage();
        
        // 处理回合开始 Buff
        EffectManager.ProcessTurnStart(currentUnit.ASC);
        
        // ...
    }
}
```

**时间线对比**：

**方案 A：回合结束递减（推荐）**
```
回合 1: 使用技能（CD 3）
回合 1 结束: CD → 2
回合 2 结束: CD → 1
回合 3 结束: CD → 0
回合 4: 可用 ✅
```

**方案 B：回合开始递减**
```
回合 1: 使用技能（CD 3）
回合 2 开始: CD → 2
回合 3 开始: CD → 1
回合 4 开始: CD → 0，可用 ✅

问题：玩家以为"CD 3 回合"，但实际只需等 2 个回合结束
```

---

### 4.6 完整战斗场景示例

```csharp
/// <summary>
/// 完整战斗场景：展示三种冷却机制
/// </summary>
public class CooldownBattleExample : MonoBehaviour
{
    void SimulateBattle()
    {
        var player = playerASC;
        var enemy = enemyASC;
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        
        Log("=== 战斗开始 ===");
        cooldownMgr.OnBattleStart();
        
        // ========== 回合 1：玩家使用 Fireball（CD 3） ==========
        Log("\n【回合 1】玩家回合");
        player.TryActivateAbility("Fireball", enemy); // 成功
        Log($"  Fireball CD: {cooldownMgr.GetRemainingCooldown(fireballSpec)} 回合");
        
        // 回合结束
        cooldownMgr.DecrementCooldowns(player);
        Log($"  回合结束，Fireball CD: {cooldownMgr.GetRemainingCooldown(fireballSpec)}");
        
        // ========== 回合 2：玩家使用 Ultimate（每回合限 1 次） ==========
        Log("\n【回合 2】玩家回合");
        cooldownMgr.ResetTurnUsage();
        
        player.TryActivateAbility("Ultimate", enemy); // 成功
        Log($"  Ultimate 本回合使用次数: 1");
        
        player.TryActivateAbility("Ultimate", enemy); // ❌ 失败
        Log($"  再次使用 Ultimate → 失败（每回合限 1 次）");
        
        cooldownMgr.DecrementCooldowns(player);
        
        // ========== 回合 3：玩家尝试使用 Fireball ==========
        Log("\n【回合 3】玩家回合");
        cooldownMgr.ResetTurnUsage();
        
        player.TryActivateAbility("Fireball", enemy); // ❌ 失败
        Log($"  Fireball CD: {cooldownMgr.GetRemainingCooldown(fireballSpec)} → 仍在冷却");
        
        cooldownMgr.DecrementCooldowns(player);
        
        // ========== 回合 4：Fireball 冷却结束 ==========
        Log("\n【回合 4】玩家回合");
        cooldownMgr.ResetTurnUsage();
        
        player.TryActivateAbility("Fireball", enemy); // ✅ 成功
        Log($"  Fireball 冷却结束，成功使用");
        
        cooldownMgr.DecrementCooldowns(player);
        
        // ========== 回合 11：使用 LimitBreak（每战斗限 3 次） ==========
        Log("\n【回合 11】玩家回合");
        player.TryActivateAbility("LimitBreak", enemy); // 成功（1/3）
        
        // ... 回合 16
        player.TryActivateAbility("LimitBreak", enemy); // 成功（2/3）
        
        // ... 回合 21
        player.TryActivateAbility("LimitBreak", enemy); // 成功（3/3）
        
        // ... 回合 26
        player.TryActivateAbility("LimitBreak", enemy); // ❌ 失败
        Log($"  LimitBreak 已用完（3/3）");
        
        Log("\n=== 战斗结束 ===");
        cooldownMgr.OnBattleEnd();
    }
}
```

**预期输出**：
```
=== 战斗开始 ===

【回合 1】玩家回合
  Fireball CD: 3 回合
  回合结束，Fireball CD: 2

【回合 2】玩家回合
  Ultimate 本回合使用次数: 1
  再次使用 Ultimate → 失败（每回合限 1 次）

【回合 3】玩家回合
  Fireball CD: 1 → 仍在冷却

【回合 4】玩家回合
  Fireball 冷却结束，成功使用

【回合 11】玩家回合
  LimitBreak 已用完（3/3）

=== 战斗结束 ===
```

---

### 4.7 冷却机制设计总结

**设计清单**：

| 机制 | 何时使用 | 配置参数 | 检查时机 | 更新时机 |
|------|---------|---------|---------|---------|
| **回合冷却** | 限制使用频率 | `CooldownTurns` | `CanActivate()` | 回合结束 |
| **回合次数** | 防止连续使用 | `MaxUsesPerTurn` | `CanActivate()` | 回合开始重置 |
| **战斗次数** | 限制总使用次数 | `MaxUsesPerBattle` | `CanActivate()` | 战斗结束清空 |

**最佳实践**：
1. ✅ 大部分技能只需要回合冷却
2. ✅ 强力技能配合回合次数限制（防止爆发）
3. ✅ 终极技能配合战斗次数限制（增加策略性）
4. ✅ 所有冷却在回合结束时统一递减
5. ✅ 回合次数在回合开始时统一重置

---

## 五、资源消耗系统（Cost GE）

### 5.1 为什么用 GE 而非直接减属性

#### 5.1.1 错误的做法 ❌

```csharp
// ❌ 不符合 GAS 理念
public class BadManaAbilitySpec : AbilitySpec
{
    public override bool CanActivate()
    {
        var mana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
        if (mana < 50) return false;
        
        return base.CanActivate();
    }
    
    public override void ActivateAbility(params object[] args)
    {
        // ❌ 直接修改属性
        Owner.AttrSet<AS_Combat>().Mana.CurrentValue -= 50;
        
        // 技能效果
        // ...
    }
}
```

**问题**：
1. ❌ 违反 GAS 黄金法则（只有 GE 可以修改属性）
2. ❌ 无法追踪消耗来源（调试困难）
3. ❌ 不支持消耗减免（Buff 降低消耗）
4. ❌ 不触发 AttributeChanged 事件
5. ❌ 无法通过 GameplayCue 播放消耗特效

---

#### 5.1.2 正确的做法 ✅

```csharp
// ✅ 符合 GAS 理念
public class GoodManaAbilityAsset : AbilityAsset
{
    [Header("资源消耗")]
    public GameplayEffect ManaCostEffect; // Instant GE, Add -50 to Mana
}

public class GoodManaAbilitySpec : AbilitySpec<GoodManaAbility>
{
    public override bool CanActivate()
    {
        var asset = Ability.DataReference as GoodManaAbilityAsset;
        
        // 检查 Mana 是否足够
        var mana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
        var cost = CalculateManaCost(); // 可通过 MMC 动态计算
        
        if (mana < cost) return false;
        
        return base.CanActivate();
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var asset = Ability.DataReference as GoodManaAbilityAsset;
        
        // ✅ 通过 Cost GE 消耗 Mana
        Owner.ApplyGameplayEffectTo(asset.ManaCostEffect, Owner);
        
        // 技能效果
        // ...
    }
}
```

**优势**：
1. ✅ 符合 GAS 黄金法则
2. ✅ 可追踪消耗来源（Runtime Watcher 显示）
3. ✅ 支持消耗减免（通过 MMC）
4. ✅ 触发 AttributeChanged 事件（UI 更新）
5. ✅ 可配置 GameplayCue（消耗特效）
6. ✅ 支持 Tag 条件（免费施法 Buff）

---

### 5.2 Cost GE 模式详解

#### 5.2.1 基础配置

**创建 Cost GE**：
1. 在 Unity 中创建 GameplayEffect ScriptableObject
2. 配置为 `Instant` 策略
3. 添加 Modifier：`Attribute = Mana`, `Operation = Add`, `Magnitude = -50`

**配置示例**：
```csharp
// ManaCost_50.asset
DurationPolicy: Instant
Modifiers:
  - Attribute: AS_Combat.Mana
    Operation: Add
    Magnitude: ScalableFloat(-50)
```

**在技能中引用**：
```csharp
public class FireballAbilityAsset : AbilityAsset
{
    [Header("资源消耗")]
    public GameplayEffect ManaCostEffect; // 拖入 ManaCost_50.asset
}
```

---

#### 5.2.2 动态消耗计算（MMC）

**场景**：技能等级越高，消耗越多

```csharp
// 1. 创建自定义 MMC
public class ScalingManaCostCalculation : ModifierMagnitudeCalculation
{
    public float BaseManaCost = 30f;
    public float CostPerLevel = 10f;
    
    public override float CalculateMagnitude(GameplayEffectSpec spec)
    {
        // 获取技能等级（假设存储在 Spec 的 Context 中）
        int abilityLevel = GetAbilityLevel(spec);
        
        // 计算消耗：基础消耗 + 等级消耗
        float cost = BaseManaCost + (abilityLevel - 1) * CostPerLevel;
        
        // 返回负数（消耗）
        return -cost;
    }
    
    private int GetAbilityLevel(GameplayEffectSpec spec)
    {
        // 从 Spec 的 Context 中获取技能等级
        if (spec.Context != null && spec.Context.TryGetValue("AbilityLevel", out var level))
        {
            return (int)level;
        }
        return 1;
    }
}

// 2. 在 GE 中使用 MMC
// ManaCost_Scaling.asset
DurationPolicy: Instant
Modifiers:
  - Attribute: AS_Combat.Mana
    Operation: Add
    Magnitude: CustomCalculation(ScalingManaCostCalculation)

// 3. 在技能中设置等级
public class ScalingManaAbilitySpec : AbilitySpec<ScalingManaAbility>
{
    public int AbilityLevel = 1;
    
    public override void ActivateAbility(params object[] args)
    {
        var asset = Ability.DataReference as ScalingManaAbilityAsset;
        
        // 创建 Spec 并设置等级
        var costSpec = GameplayEffect.CreateSpec(asset.ManaCostEffect, Owner, Owner);
        costSpec.Context = new Dictionary<string, object>
        {
            { "AbilityLevel", AbilityLevel }
        };
        
        // 应用消耗
        Owner.ApplyGameplayEffectSpec(costSpec);
        
        // 技能效果
        // ...
    }
}
```

**消耗示例**：
```
等级 1: 30 + (1-1)*10 = 30 Mana
等级 2: 30 + (2-1)*10 = 40 Mana
等级 3: 30 + (3-1)*10 = 50 Mana
```

---

### 5.3 消耗减免机制

#### 5.3.1 通过 Buff 减免消耗

**场景**：Buff 使所有技能消耗减少 20%

```csharp
// 1. 创建消耗减免 MMC
public class CostReductionCalculation : ModifierMagnitudeCalculation
{
    public float BaseCost = 50f;
    
    public override float CalculateMagnitude(GameplayEffectSpec spec)
    {
        var target = spec.Owner;
        
        // 检查是否有减免 Buff
        float reduction = 1.0f;
        if (target.HasTag(GTagLib.State_Buff_CostReduction))
        {
            reduction = 0.8f; // 减少 20%
        }
        
        // 计算最终消耗
        float finalCost = BaseCost * reduction;
        
        return -finalCost;
    }
}

// 2. 创建减免 Buff GE
// CostReductionBuff.asset
DurationPolicy: Infinite
GrantedTags:
  - State.Buff.CostReduction

// 3. 应用 Buff
player.ASC.ApplyGameplayEffectTo(costReductionBuffGE, player.ASC);

// 4. 使用技能时自动减免
player.ASC.TryActivateAbility("Fireball", enemy.ASC);
// → Mana 消耗：50 * 0.8 = 40
```

---

#### 5.3.2 免费施法机制

**场景**：Buff 使下一个技能免费

```csharp
// 1. 在 CanActivate() 中检查免费施法 Tag
public class ManaAbilitySpec : AbilitySpec<ManaAbility>
{
    public override bool CanActivate()
    {
        // 检查是否有免费施法 Buff
        if (Owner.HasTag(GTagLib.State_Buff_FreeCast))
        {
            return true; // 免费，无需检查 Mana
        }
        
        // 正常检查 Mana
        var cost = CalculateManaCost();
        var mana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
        
        return mana >= cost;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        // 检查是否免费施法
        if (Owner.HasTag(GTagLib.State_Buff_FreeCast))
        {
            // 移除免费施法 Buff
            RemoveFreeCastBuff();
        }
        else
        {
            // 正常消耗 Mana
            Owner.ApplyGameplayEffectTo(Ability.DataReference.ManaCostEffect, Owner);
        }
        
        // 技能效果
        // ...
    }
    
    private void RemoveFreeCastBuff()
    {
        var buffToRemove = Owner.GameplayEffectContainer.GameplayEffects()
            .FirstOrDefault(ge => ge.GameplayEffect.GrantedTags.Contains(GTagLib.State_Buff_FreeCast));
        
        if (buffToRemove != null)
        {
            Owner.RemoveGameplayEffect(buffToRemove);
        }
    }
}

// 2. 免费施法 Buff
// FreeCastBuff.asset
DurationPolicy: Infinite
GrantedTags:
  - State.Buff.FreeCast
RemoveGameplayEffectsWithTags:
  - State.Buff.FreeCast  // 使用后自动移除
```

---

### 5.4 完整技能示例

#### 5.4.1 Mana 消耗技能

```csharp
// 1. AbilityAsset 配置
public class FireballAbilityAsset : AbilityAsset
{
    [Header("资源消耗")]
    public GameplayEffect ManaCostEffect; // ManaCost_50.asset
    
    [Header("技能效果")]
    public GameplayEffect DamageEffect;
    
    [Header("冷却设置")]
    public int CooldownTurns = 3;
}

// 2. AbilitySpec 实现
public class FireballAbilitySpec : AbilitySpec<FireballAbility>
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        var asset = Ability.DataReference as FireballAbilityAsset;
        
        // 检查冷却
        if (cooldownMgr.IsOnCooldown(this))
        {
            return AbilityActivateResult.Fail;
        }
        
        // 检查 Mana（除非有免费施法 Buff）
        if (!Owner.HasTag(GTagLib.State_Buff_FreeCast))
        {
            var mana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
            var cost = CalculateManaCost();
            
            if (mana < cost)
            {
                Debug.Log($"Mana 不足：需要 {cost}，当前 {mana}");
                return AbilityActivateResult.Fail;
            }
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as FireballAbilityAsset;
        
        // 消耗 Mana（除非免费施法）
        if (Owner.HasTag(GTagLib.State_Buff_FreeCast))
        {
            RemoveFreeCastBuff();
        }
        else
        {
            Owner.ApplyGameplayEffectTo(asset.ManaCostEffect, Owner);
        }
        
        // 技能效果
        Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        
        // 设置冷却
        cooldownMgr.StartCooldown(this, asset.CooldownTurns);
        
        EndAbility();
    }
    
    private float CalculateManaCost()
    {
        // 可以通过 MMC 动态计算，这里简化
        return 50f;
    }
    
    private void RemoveFreeCastBuff()
    {
        // 实现见上文
    }
}
```

---

#### 5.4.2 体力消耗技能

```csharp
// 1. AbilityAsset 配置
public class ChargeAttackAbilityAsset : AbilityAsset
{
    [Header("资源消耗")]
    public GameplayEffect StaminaCostEffect; // StaminaCost_30.asset
    
    [Header("技能效果")]
    public GameplayEffect DamageEffect;
    
    [Header("冷却设置")]
    public int CooldownTurns = 2;
}

// 2. 体力消耗 GE
// StaminaCost_30.asset
DurationPolicy: Instant
Modifiers:
  - Attribute: AS_Combat.Stamina
    Operation: Add
    Magnitude: ScalableFloat(-30)

// 3. AbilitySpec 实现
public class ChargeAttackAbilitySpec : AbilitySpec<ChargeAttackAbility>
{
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        // 检查体力
        var stamina = Owner.GetAttributeCurrentValue("AS_Combat", "Stamina") ?? 0;
        if (stamina < 30)
        {
            Debug.Log($"体力不足：需要 30，当前 {stamina}");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as ChargeAttackAbilityAsset;
        
        // 消耗体力
        Owner.ApplyGameplayEffectTo(asset.StaminaCostEffect, Owner);
        
        // 技能效果
        Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        
        EndAbility();
    }
}
```

---

#### 5.4.3 怒气消耗技能（特殊机制）

**场景**：怒气通过攻击积累，使用技能消耗

```csharp
// 1. 怒气积累 GE（被攻击时获得）
// RageGain_10.asset
DurationPolicy: Instant
Modifiers:
  - Attribute: AS_Combat.Rage
    Operation: Add
    Magnitude: ScalableFloat(10)  // 获得 10 点怒气

// 2. 怒气消耗技能
public class RageAbilityAsset : AbilityAsset
{
    [Header("资源消耗")]
    public int RageCost = 50;  // 需要 50 怒气
    public GameplayEffect RageCostEffect; // RageCost_50.asset
    
    [Header("技能效果")]
    public GameplayEffect DamageEffect;
}

// 3. AbilitySpec 实现
public class RageAbilitySpec : AbilitySpec<RageAbility>
{
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        var asset = Ability.DataReference as RageAbilityAsset;
        
        // 检查怒气
        var rage = Owner.GetAttributeCurrentValue("AS_Combat", "Rage") ?? 0;
        if (rage < asset.RageCost)
        {
            Debug.Log($"怒气不足：需要 {asset.RageCost}，当前 {rage}");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as RageAbilityAsset;
        
        // 消耗怒气
        Owner.ApplyGameplayEffectTo(asset.RageCostEffect, Owner);
        
        // 技能效果（通常更强力）
        Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        
        EndAbility();
    }
}

// 4. 在普通攻击中积累怒气
public class BasicAttackAbilitySpec : AbilitySpec<BasicAttackAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // 攻击效果
        Owner.ApplyGameplayEffectTo(damageEffect, target);
        
        // 获得怒气
        Owner.ApplyGameplayEffectTo(rageGainEffect, Owner);
        
        EndAbility();
    }
}
```

---

### 5.5 边缘场景处理

#### 5.5.1 消耗后资源不足（负数）

**问题**：Mana 剩余 30，技能消耗 50，应该禁止使用还是允许负数？

**方案 A**：禁止使用（推荐）
```csharp
public override AbilityActivateResult CanActivate()
{
    var cost = CalculateManaCost();
    var mana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
    
    // ✅ 检查是否足够
    if (mana < cost)
    {
        return AbilityActivateResult.Fail;
    }
    
    return base.CanActivate();
}
```

**方案 B**：允许负数（透支）
```csharp
// 在 AttributeSet 中限制最小值
public class AS_Combat : AttributeSet
{
    public Attribute Mana = new Attribute
    {
        MinValue = 0,  // 最小值为 0，不会变负数
        MaxValue = 100
    };
}
```

---

#### 5.5.2 多资源消耗技能

**场景**：技能同时消耗 Mana 和体力

```csharp
public class MultiCostAbilityAsset : AbilityAsset
{
    [Header("资源消耗")]
    public GameplayEffect ManaCostEffect;    // -50 Mana
    public GameplayEffect StaminaCostEffect; // -30 Stamina
}

public class MultiCostAbilitySpec : AbilitySpec<MultiCostAbility>
{
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        // 检查 Mana
        var mana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
        if (mana < 50)
        {
            return AbilityActivateResult.Fail;
        }
        
        // 检查体力
        var stamina = Owner.GetAttributeCurrentValue("AS_Combat", "Stamina") ?? 0;
        if (stamina < 30)
        {
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var asset = Ability.DataReference as MultiCostAbilityAsset;
        
        // 消耗 Mana
        Owner.ApplyGameplayEffectTo(asset.ManaCostEffect, Owner);
        
        // 消耗体力
        Owner.ApplyGameplayEffectTo(asset.StaminaCostEffect, Owner);
        
        // 技能效果
        // ...
    }
}
```

---

#### 5.5.3 消耗失败回滚

**场景**：技能使用过程中失败（目标死亡），需要退还消耗

```csharp
public class RefundableAbilitySpec : AbilitySpec<RefundableAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as RefundableAbilityAsset;
        
        // 先消耗资源
        Owner.ApplyGameplayEffectTo(asset.ManaCostEffect, Owner);
        
        // 检查目标是否有效
        if (!target.IsAlive)
        {
            Debug.LogWarning("目标已死亡，退还 Mana");
            
            // 退还资源
            Owner.ApplyGameplayEffectTo(asset.ManaRefundEffect, Owner); // +50 Mana
            
            CancelAbility();
            return;
        }
        
        // 技能效果
        Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        
        EndAbility();
    }
}

// ManaRefund_50.asset
DurationPolicy: Instant
Modifiers:
  - Attribute: AS_Combat.Mana
    Operation: Add
    Magnitude: ScalableFloat(50)  // 退还 50 Mana
```

---

### 5.6 资源消耗系统总结

**核心原则**：
1. ✅ **始终用 GE 消耗资源**（符合 GAS 理念）
2. ✅ **在 CanActivate() 中检查**（避免无效操作）
3. ✅ **支持消耗减免**（通过 MMC 或 Tag）
4. ✅ **处理边缘场景**（负数、多资源、退还）

**推荐资源类型**：
- **Mana**：法术技能消耗
- **Stamina（体力）**：物理技能消耗
- **Rage（怒气）**：战斗积累，爆发技能消耗
- **Energy（能量）**：通用资源，每回合恢复

**Cost GE 配置速查**：
```
消耗 50 Mana:
  DurationPolicy: Instant
  Modifier: AS_Combat.Mana, Add, -50

消耗减免 20%:
  使用 CustomCalculation(CostReductionMMC)
  检查 Tag: State.Buff.CostReduction

免费施法:
  CanActivate() 检查 Tag: State.Buff.FreeCast
  使用后移除 Buff
```

---

## 六、减 CD 与特殊机制

### 6.1 ReduceCooldown API 详解

#### 6.1.1 核心 API

```csharp
/// <summary>
/// 减少指定技能的冷却
/// </summary>
/// <param name="ability">技能实例</param>
/// <param name="reduction">减少的回合数</param>
public void ReduceCooldown(AbilitySpec ability, int reduction)
{
    if (!cooldowns.ContainsKey(ability)) return;
    
    cooldowns[ability] = Mathf.Max(0, cooldowns[ability] - reduction);
    
    if (cooldowns[ability] <= 0)
    {
        cooldowns.Remove(ability);
        Debug.Log($"技能 {ability.Ability.Name} 冷却已就绪");
    }
    else
    {
        Debug.Log($"技能 {ability.Ability.Name} 剩余冷却：{cooldowns[ability]} 回合");
    }
}

/// <summary>
/// 重置指定技能的冷却（立即可用）
/// </summary>
public void ResetCooldown(AbilitySpec ability)
{
    if (cooldowns.ContainsKey(ability))
    {
        cooldowns.Remove(ability);
        Debug.Log($"技能 {ability.Ability.Name} 冷却已重置");
    }
}
```

---

#### 6.1.2 使用示例

**场景 1：减少单个技能冷却**

```csharp
// 使用道具减少 Fireball 冷却 2 回合
var fireballSpec = player.ASC.AbilityContainer.AbilitySpecs()["Fireball"];
cooldownManager.ReduceCooldown(fireballSpec, 2);

// 示例：冷却从 5 回合减至 3 回合
// Before: 剩余冷却 5 回合
// After:  剩余冷却 3 回合
```

**场景 2：重置冷却（立即可用）**

```csharp
// 大招道具：立即重置 Ultimate 技能冷却
var ultimateSpec = player.ASC.AbilityContainer.AbilitySpecs()["Ultimate"];
cooldownManager.ResetCooldown(ultimateSpec);

// 示例：
// Before: Ultimate 剩余冷却 6 回合
// After:  Ultimate 立即可用
```

---

### 6.2 批量操作 API

#### 6.2.1 减少所有技能冷却

```csharp
/// <summary>
/// 减少单位所有技能的冷却
/// </summary>
public void ReduceAllCooldowns(AbilitySystemComponent owner, int reduction)
{
    var abilitiesToUpdate = cooldowns.Keys
        .Where(a => a.Owner == owner)
        .ToList();
    
    foreach (var ability in abilitiesToUpdate)
    {
        ReduceCooldown(ability, reduction);
    }
    
    Debug.Log($"{owner.name} 所有技能冷却减少 {reduction} 回合");
}

/// <summary>
/// 重置单位所有技能冷却
/// </summary>
public void ResetAllCooldowns(AbilitySystemComponent owner)
{
    var abilitiesToReset = cooldowns.Keys
        .Where(a => a.Owner == owner)
        .ToList();
    
    foreach (var ability in abilitiesToReset)
    {
        cooldowns.Remove(ability);
    }
    
    Debug.Log($"{owner.name} 所有技能冷却已重置");
}
```

---

#### 6.2.2 使用场景

**场景 1：回合结束减少所有冷却（已在 BattleManager 中实现）**

```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    void EndCurrentTurn()
    {
        var currentUnit = AllUnits[currentUnitIndex];
        
        // ✅ 回合结束时减少所有冷却 1 回合
        cooldownManager.DecrementCooldowns(currentUnit.ASC);
        
        // ...
    }
}
```

**场景 2：Buff 效果 - 减少所有冷却**

```csharp
// "刷新" Buff：使用后减少所有技能冷却 2 回合
public class RefreshBuffCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        var target = parameters.Target;
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        
        // 减少所有技能冷却 2 回合
        cooldownMgr.ReduceAllCooldowns(target, 2);
        
        Debug.Log($"{target.name} 使用刷新 Buff，所有技能冷却 -2 回合");
    }
}
```

**场景 3：终极技能 - 重置所有冷却**

```csharp
public class TimeRewindAbilitySpec : AbilitySpec<TimeRewindAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        
        // 重置自己所有技能冷却（除了本技能）
        cooldownMgr.ResetAllCooldowns(Owner);
        
        // 本技能进入超长冷却
        cooldownMgr.StartCooldown(this, 10);
        
        Debug.Log($"{Owner.name} 使用时间倒流，所有技能冷却重置！");
        
        EndAbility();
    }
}
```

---

### 6.3 共享冷却组（Shared Cooldown Groups）

#### 6.3.1 核心机制

**问题**：某些技能共享冷却（如：3 个不同的火球术共享冷却）

**解决方案**：共享冷却组

```csharp
// 在 TurnBasedAbilityCooldownManager 中添加
private Dictionary<string, int> sharedCooldownGroups = new();

/// <summary>
/// 启动共享冷却组
/// </summary>
public void StartSharedCooldown(string groupName, int cooldownTurns)
{
    sharedCooldownGroups[groupName] = cooldownTurns;
    Debug.Log($"共享冷却组 '{groupName}' 进入冷却：{cooldownTurns} 回合");
}

/// <summary>
/// 检查共享冷却组是否在冷却中
/// </summary>
public bool IsSharedCooldownActive(string groupName)
{
    return sharedCooldownGroups.ContainsKey(groupName) && sharedCooldownGroups[groupName] > 0;
}

/// <summary>
/// 获取共享冷却组剩余冷却
/// </summary>
public int GetSharedCooldownRemaining(string groupName)
{
    if (sharedCooldownGroups.ContainsKey(groupName))
    {
        return sharedCooldownGroups[groupName];
    }
    return 0;
}

/// <summary>
/// 在回合结束时递减共享冷却
/// </summary>
public void DecrementCooldowns(AbilitySystemComponent owner)
{
    // 递减单独冷却
    var abilitiesToUpdate = cooldowns.Keys
        .Where(a => a.Owner == owner)
        .ToList();
    
    foreach (var ability in abilitiesToUpdate)
    {
        cooldowns[ability]--;
        if (cooldowns[ability] <= 0)
        {
            cooldowns.Remove(ability);
        }
    }
    
    // ✅ 递减共享冷却组
    var groupsToUpdate = sharedCooldownGroups.Keys.ToList();
    foreach (var group in groupsToUpdate)
    {
        sharedCooldownGroups[group]--;
        if (sharedCooldownGroups[group] <= 0)
        {
            sharedCooldownGroups.Remove(group);
            Debug.Log($"共享冷却组 '{group}' 已就绪");
        }
    }
}
```

---

#### 6.3.2 技能配置

```csharp
// 1. AbilityAsset 配置
public class SharedCooldownAbilityAsset : AbilityAsset
{
    [Header("共享冷却设置")]
    public string SharedCooldownGroup = "Fireball"; // 共享组名
    public int SharedCooldownTurns = 2;             // 共享冷却回合数
}

// 2. AbilitySpec 实现
public class SharedCooldownAbilitySpec : AbilitySpec<SharedCooldownAbility>
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        var asset = Ability.DataReference as SharedCooldownAbilityAsset;
        
        // ✅ 检查共享冷却
        if (cooldownMgr.IsSharedCooldownActive(asset.SharedCooldownGroup))
        {
            var remaining = cooldownMgr.GetSharedCooldownRemaining(asset.SharedCooldownGroup);
            Debug.Log($"共享冷却中：{remaining} 回合");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var asset = Ability.DataReference as SharedCooldownAbilityAsset;
        
        // 技能效果
        // ...
        
        // ✅ 启动共享冷却
        cooldownMgr.StartSharedCooldown(asset.SharedCooldownGroup, asset.SharedCooldownTurns);
        
        EndAbility();
    }
}
```

---

#### 6.3.3 使用场景

**示例：三个火球术共享冷却**

```csharp
// Fireball_Small.asset
SharedCooldownGroup: "Fireball"
SharedCooldownTurns: 2

// Fireball_Medium.asset
SharedCooldownGroup: "Fireball"
SharedCooldownTurns: 2

// Fireball_Large.asset
SharedCooldownGroup: "Fireball"
SharedCooldownTurns: 2

// 使用流程：
// 回合 1：使用 Fireball_Small → 共享组 "Fireball" 进入 2 回合冷却
// 回合 2：Fireball_Medium/Large 都无法使用（共享冷却中）
// 回合 3：Fireball_Medium/Large 都无法使用（共享冷却剩余 1 回合）
// 回合 4：所有 Fireball 技能可用
```

**示例：职业技能共享冷却**

```csharp
// 战士的 3 个姿态技能共享冷却
// DefensiveStance.asset
SharedCooldownGroup: "WarriorStance"
SharedCooldownTurns: 1  // 切换姿态后需等待 1 回合

// OffensiveStance.asset
SharedCooldownGroup: "WarriorStance"
SharedCooldownTurns: 1

// BerserkerStance.asset
SharedCooldownGroup: "WarriorStance"
SharedCooldownTurns: 1

// 防止玩家频繁切换姿态
```

---

### 6.4 全局冷却（GCD）

#### 6.4.1 核心机制

**问题**：所有技能共享 1 回合冷却（使用任何技能后，下回合才能再次行动）

**解决方案**：全局冷却组

```csharp
// 在 TurnBasedAbilityCooldownManager 中
private const string GlobalCooldownGroup = "GlobalCooldown";

/// <summary>
/// 启动全局冷却
/// </summary>
public void StartGlobalCooldown(int cooldownTurns = 1)
{
    StartSharedCooldown(GlobalCooldownGroup, cooldownTurns);
}

/// <summary>
/// 检查全局冷却
/// </summary>
public bool IsGlobalCooldownActive()
{
    return IsSharedCooldownActive(GlobalCooldownGroup);
}
```

**在 AbilitySpec 中使用**：

```csharp
public class GCDAbilitySpec : AbilitySpec<GCDAbility>
{
    public override AbilityActivateResult CanActivate()
    {
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        
        // ✅ 检查全局冷却
        if (cooldownMgr.IsGlobalCooldownActive())
        {
            Debug.Log("全局冷却中，无法使用任何技能");
            return AbilityActivateResult.Fail;
        }
        
        return base.CanActivate();
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        
        // 技能效果
        // ...
        
        // ✅ 启动全局冷却
        cooldownMgr.StartGlobalCooldown(1);
        
        EndAbility();
    }
}
```

---

### 6.5 通过 GameplayCue 触发减 CD

#### 6.5.1 Buff 减 CD 示例

**场景**：使用 "冷却缩减药水" Buff，每回合减少所有技能冷却 1 回合

```csharp
// 1. 创建 Buff GE
// CooldownReductionBuff.asset
DurationPolicy: Infinite  // 持续到战斗结束
GrantedTags:
  - State.Buff.CooldownReduction
OnAppliedCue: CooldownReductionCue  // 应用时触发

// 2. 创建 Cue
public class CooldownReductionCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        var target = parameters.Target;
        
        Debug.Log($"{target.name} 获得冷却缩减 Buff");
    }
}

// 3. 在回合结束时额外减 CD
public class TurnBasedBattleManager : MonoBehaviour
{
    void EndCurrentTurn()
    {
        var currentUnit = AllUnits[currentUnitIndex];
        
        // 正常减少冷却 1 回合
        cooldownManager.DecrementCooldowns(currentUnit.ASC);
        
        // ✅ 如果有冷却缩减 Buff，额外减少 1 回合
        if (currentUnit.ASC.HasTag(GTagLib.State_Buff_CooldownReduction))
        {
            cooldownManager.ReduceAllCooldowns(currentUnit.ASC, 1);
            Debug.Log($"{currentUnit.ASC.name} 冷却缩减 Buff 生效，额外减少 1 回合");
        }
        
        // ...
    }
}
```

**效果**：
```
普通情况：回合结束冷却 -1
有 Buff：回合结束冷却 -2（1 + 1）

示例：
- Fireball CD 5 回合
- 回合 1 结束：CD 5 → 4
- 回合 2 结束（有 Buff）：CD 4 → 2（减少 2 回合）
- 回合 3 结束（有 Buff）：CD 2 → 0（可用）
```

---

#### 6.5.2 击杀减 CD 示例

**场景**：击杀敌人后，重置所有技能冷却

```csharp
// 在 DamageCue 中检测击杀
public class DamageCue : GameplayCueInstant
{
    public override void Trigger(GameplayCueParameters parameters)
    {
        var source = parameters.Source;
        var target = parameters.Target;
        
        // 播放受击动画
        // ...
        
        // ✅ 检测击杀
        var targetHealth = target.GetAttributeCurrentValue("AS_Combat", "Health") ?? 0;
        if (targetHealth <= 0)
        {
            OnEnemyKilled(source, target);
        }
    }
    
    private void OnEnemyKilled(AbilitySystemComponent killer, AbilitySystemComponent victim)
    {
        Debug.Log($"{killer.name} 击杀了 {victim.name}");
        
        // ✅ 检查击杀者是否有 "击杀重置" Buff
        if (killer.HasTag(GTagLib.State_Buff_KillReset))
        {
            var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
            cooldownMgr.ResetAllCooldowns(killer);
            
            Debug.Log($"{killer.name} 击杀重置：所有技能冷却重置！");
        }
    }
}

// KillResetBuff.asset
DurationPolicy: Infinite
GrantedTags:
  - State.Buff.KillReset
```

---

### 6.6 完整战斗示例

#### 6.6.1 场景设计

**单位**：
- 玩家（Player）拥有技能：
  - Fireball（CD 3 回合）
  - Heal（CD 2 回合）
  - Ultimate（CD 5 回合）

**道具**：
- "刷新药水"：减少所有技能冷却 2 回合
- "冷却缩减 Buff"：每回合额外减少冷却 1 回合

---

#### 6.6.2 战斗流程

```csharp
public class CooldownReductionBattleExample : MonoBehaviour
{
    void Start()
    {
        var player = FindObjectOfType<PlayerUnit>().ASC;
        var enemy = FindObjectOfType<EnemyUnit>().ASC;
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        
        Debug.Log("=== 战斗开始 ===");
        
        // --- 回合 1 ---
        Debug.Log("\n[回合 1] 玩家使用 Fireball");
        player.TryActivateAbility("Fireball", enemy);
        // → Fireball 进入 3 回合冷却
        // → Heal CD 0, Ultimate CD 0
        
        // 回合结束
        cooldownMgr.DecrementCooldowns(player);
        // → Fireball CD 3 → 2
        
        // --- 回合 2 ---
        Debug.Log("\n[回合 2] 玩家使用 Ultimate");
        player.TryActivateAbility("Ultimate", enemy);
        // → Ultimate 进入 5 回合冷却
        // → Fireball CD 2, Heal CD 0
        
        // 回合结束
        cooldownMgr.DecrementCooldowns(player);
        // → Fireball CD 2 → 1
        // → Ultimate CD 5 → 4
        
        // --- 回合 3 ---
        Debug.Log("\n[回合 3] 玩家使用刷新药水");
        UseRefreshPotion(player);
        // → 所有技能冷却 -2
        // → Fireball CD 1 → 0（可用）
        // → Ultimate CD 4 → 2
        
        Debug.Log("玩家使用 Fireball");
        player.TryActivateAbility("Fireball", enemy);
        // → Fireball 进入 3 回合冷却
        
        // 回合结束
        cooldownMgr.DecrementCooldowns(player);
        // → Fireball CD 3 → 2
        // → Ultimate CD 2 → 1
        
        // --- 回合 4 ---
        Debug.Log("\n[回合 4] 玩家获得冷却缩减 Buff");
        ApplyCooldownReductionBuff(player);
        // → GrantedTags: State.Buff.CooldownReduction
        
        // 回合结束
        cooldownMgr.DecrementCooldowns(player);
        // → 正常减少：Fireball CD 2 → 1, Ultimate CD 1 → 0
        
        if (player.HasTag(GTagLib.State_Buff_CooldownReduction))
        {
            cooldownMgr.ReduceAllCooldowns(player, 1);
            // → 额外减少：Fireball CD 1 → 0, Ultimate CD 0 → 0
        }
        
        // --- 回合 5 ---
        Debug.Log("\n[回合 5] 所有技能已就绪");
        Debug.Log($"Fireball CD: {cooldownMgr.GetRemainingCooldown(fireballSpec)}");
        Debug.Log($"Ultimate CD: {cooldownMgr.GetRemainingCooldown(ultimateSpec)}");
        // → 输出: Fireball CD: 0, Ultimate CD: 0
    }
    
    void UseRefreshPotion(AbilitySystemComponent player)
    {
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        cooldownMgr.ReduceAllCooldowns(player, 2);
        Debug.Log("使用刷新药水，所有技能冷却 -2 回合");
    }
    
    void ApplyCooldownReductionBuff(AbilitySystemComponent player)
    {
        player.ApplyGameplayEffectTo(cooldownReductionBuffGE, player);
        Debug.Log("获得冷却缩减 Buff，每回合额外减少 1 回合冷却");
    }
}
```

---

#### 6.6.3 预期输出

```
=== 战斗开始 ===

[回合 1] 玩家使用 Fireball
→ Fireball 进入冷却：3 回合
回合结束：Fireball CD 3 → 2

[回合 2] 玩家使用 Ultimate
→ Ultimate 进入冷却：5 回合
回合结束：Fireball CD 2 → 1, Ultimate CD 5 → 4

[回合 3] 玩家使用刷新药水
→ 所有技能冷却 -2 回合
→ Fireball CD 1 → 0（可用）
→ Ultimate CD 4 → 2

玩家使用 Fireball
→ Fireball 进入冷却：3 回合
回合结束：Fireball CD 3 → 2, Ultimate CD 2 → 1

[回合 4] 玩家获得冷却缩减 Buff
回合结束：
  正常减少：Fireball CD 2 → 1, Ultimate CD 1 → 0
  额外减少（Buff）：Fireball CD 1 → 0
  
[回合 5] 所有技能已就绪
Fireball CD: 0
Ultimate CD: 0
```

---

### 6.7 减 CD 机制总结

**核心 API**：
- `ReduceCooldown(ability, turns)`: 减少单个技能冷却
- `ResetCooldown(ability)`: 重置单个技能冷却
- `ReduceAllCooldowns(owner, turns)`: 减少所有技能冷却
- `ResetAllCooldowns(owner)`: 重置所有技能冷却

**共享冷却**：
- `StartSharedCooldown(groupName, turns)`: 启动共享冷却组
- `IsSharedCooldownActive(groupName)`: 检查共享冷却
- 用途：多个技能共享冷却（如：3 种火球术）

**全局冷却（GCD）**：
- `StartGlobalCooldown()`: 所有技能共享 1 回合冷却
- 用途：防止玩家单回合使用多个技能

**Buff 集成**：
- 通过 Tag 检查 Buff（如：`State.Buff.CooldownReduction`）
- 在回合结束时额外减少冷却
- 通过 GameplayCue 触发减 CD 逻辑

**常见场景**：
- 道具减 CD（刷新药水）
- Buff 持续减 CD（冷却缩减药水）
- 击杀重置 CD（杀敌刷新）
- 终极技能重置所有 CD（时间倒流）

---

**下一章**：与 AbilitySpec 集成（CanActivate + ActivateAbility 完整流程）

---

## 七、与 AbilitySpec 集成

### 7.1 集成架构

#### 7.1.1 职责划分

| 组件 | 职责 |
|------|------|
| **TurnBasedAbilityCooldownManager** | 冷却状态管理、次数限制、减 CD 操作 |
| **AbilitySpec** | 技能逻辑、检查冷却、记录使用、消耗资源 |
| **TurnBasedBattleManager** | 回合流程、调用冷却递减、重置次数 |
| **GameplayEffect** | 资源消耗（Cost GE） |

**关键原则**：
- ✅ **AbilitySpec 不存储冷却数据**（全部由 Manager 管理）
- ✅ **AbilitySpec 通过 Manager 查询冷却状态**
- ✅ **AbilitySpec 在使用后通知 Manager 记录**

---

#### 7.1.2 数据流图

```
┌─────────────────────────────────────────────────────────────┐
│                      玩家/AI 选择技能                         │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────────┐
         │  AbilitySpec.CanActivate()  │ ◄─── 检查各项条件
         └──────────┬──────────────────┘
                    │
      ┌─────────────┼─────────────┐
      │             │             │
      ▼             ▼             ▼
┌──────────┐  ┌──────────┐  ┌──────────────┐
│ 检查冷却 │  │ 检查资源 │  │ 检查次数限制 │
│（Manager）│  │（Attribute）│  │（Manager）   │
└──────────┘  └──────────┘  └──────────────┘
      │             │             │
      └─────────────┼─────────────┘
                    │
                    ▼
              ┌──────────┐
              │ 条件通过 │
              └─────┬────┘
                    │
                    ▼
    ┌────────────────────────────────┐
    │ AbilitySpec.ActivateAbility()  │ ◄─── 执行技能
    └────────────┬───────────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
    ▼            ▼            ▼
┌─────────┐ ┌─────────┐ ┌──────────┐
│ 消耗资源│ │ 技能效果│ │ 记录冷却 │
│（Cost GE）│ │（GE）  │ │（Manager）│
└─────────┘ └─────────┘ └──────────┘
                 │
                 ▼
         ┌───────────────┐
         │   回合结束    │
         └───────┬───────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
    ▼            ▼            ▼
┌─────────┐ ┌─────────┐ ┌──────────┐
│ 冷却 -1 │ │Buff计数-1│ │重置次数  │
│（Manager）│ │（Manager）│ │（Manager）│
└─────────┘ └─────────┘ └──────────┘
```

---

### 7.2 CanActivate() 检查模式

#### 7.2.1 完整检查流程

```csharp
public class TurnBasedAbilitySpec : AbilitySpec<TurnBasedAbility>
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    protected override void OnAbilityConstruct()
    {
        base.OnAbilityConstruct();
        
        // 获取 CooldownManager 引用
        cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
    }
    
    public override AbilityActivateResult CanActivate()
    {
        // 1. 基础检查（Tag 条件等）
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success)
        {
            return baseResult;
        }
        
        var asset = Ability.DataReference as TurnBasedAbilityAsset;
        
        // 2. ✅ 检查回合冷却
        if (cooldownMgr.IsOnCooldown(this))
        {
            var remaining = cooldownMgr.GetRemainingCooldown(this);
            Debug.Log($"技能冷却中：{remaining} 回合");
            return AbilityActivateResult.Fail;
        }
        
        // 3. ✅ 检查每回合次数限制
        if (asset.MaxUsesPerTurn > 0)
        {
            if (!cooldownMgr.CanUseThisTurn(Ability.Name, asset.MaxUsesPerTurn))
            {
                Debug.Log($"已达到每回合使用上限：{asset.MaxUsesPerTurn}");
                return AbilityActivateResult.Fail;
            }
        }
        
        // 4. ✅ 检查每战斗次数限制
        if (asset.MaxUsesPerBattle > 0)
        {
            if (!cooldownMgr.CanUseBattle(this, asset.MaxUsesPerBattle))
            {
                var used = cooldownMgr.GetBattleUsageCount(this);
                Debug.Log($"已达到每战斗使用上限：{used}/{asset.MaxUsesPerBattle}");
                return AbilityActivateResult.Fail;
            }
        }
        
        // 5. ✅ 检查资源（Mana/Stamina 等）
        if (asset.ManaCostEffect != null)
        {
            // 除非有免费施法 Buff
            if (!Owner.HasTag(GTagLib.State_Buff_FreeCast))
            {
                var cost = CalculateManaCost();
                var mana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
                
                if (mana < cost)
                {
                    Debug.Log($"Mana 不足：需要 {cost}，当前 {mana}");
                    return AbilityActivateResult.Fail;
                }
            }
        }
        
        // 6. ✅ 检查共享冷却组
        if (!string.IsNullOrEmpty(asset.SharedCooldownGroup))
        {
            if (cooldownMgr.IsSharedCooldownActive(asset.SharedCooldownGroup))
            {
                var remaining = cooldownMgr.GetSharedCooldownRemaining(asset.SharedCooldownGroup);
                Debug.Log($"共享冷却组 '{asset.SharedCooldownGroup}' 冷却中：{remaining} 回合");
                return AbilityActivateResult.Fail;
            }
        }
        
        // 7. ✅ 检查全局冷却（GCD）
        if (asset.RequiresGlobalCooldown)
        {
            if (cooldownMgr.IsGlobalCooldownActive())
            {
                Debug.Log("全局冷却中，无法使用技能");
                return AbilityActivateResult.Fail;
            }
        }
        
        return AbilityActivateResult.Success;
    }
}
```

---

#### 7.2.2 检查顺序优化

**推荐顺序**（从快到慢）：
1. **基础 Tag 检查**（最快，本地查询）
2. **回合冷却检查**（快，字典查询）
3. **次数限制检查**（快，字典查询）
4. **共享冷却检查**（快，字典查询）
5. **资源检查**（稍慢，需要计算 MMC）
6. **全局冷却检查**（最快，单个字典查询）

**为什么这样排序？**
- 早失败原则（Fail Fast）：最常见的失败条件放前面
- 性能优化：避免不必要的资源计算

---

### 7.3 ActivateAbility() 记录模式

#### 7.3.1 完整执行流程

```csharp
public class TurnBasedAbilitySpec : AbilitySpec<TurnBasedAbility>
{
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as TurnBasedAbilityAsset;
        
        // 1. ✅ 消耗资源（通过 Cost GE）
        if (asset.ManaCostEffect != null)
        {
            // 检查免费施法 Buff
            if (Owner.HasTag(GTagLib.State_Buff_FreeCast))
            {
                RemoveFreeCastBuff();
                Debug.Log("免费施法 Buff 触发，无需消耗 Mana");
            }
            else
            {
                Owner.ApplyGameplayEffectTo(asset.ManaCostEffect, Owner);
            }
        }
        
        // 2. ✅ 技能效果
        if (asset.DamageEffect != null)
        {
            Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        }
        
        // 3. ✅ 记录冷却
        if (asset.CooldownTurns > 0)
        {
            cooldownMgr.StartCooldown(this, asset.CooldownTurns);
        }
        
        // 4. ✅ 记录使用次数
        if (asset.MaxUsesPerTurn > 0)
        {
            cooldownMgr.RecordUsage(Ability.Name);
        }
        
        if (asset.MaxUsesPerBattle > 0)
        {
            cooldownMgr.RecordBattleUsage(this);
        }
        
        // 5. ✅ 启动共享冷却
        if (!string.IsNullOrEmpty(asset.SharedCooldownGroup))
        {
            cooldownMgr.StartSharedCooldown(asset.SharedCooldownGroup, asset.SharedCooldownTurns);
        }
        
        // 6. ✅ 启动全局冷却
        if (asset.RequiresGlobalCooldown)
        {
            cooldownMgr.StartGlobalCooldown();
        }
        
        EndAbility();
    }
}
```

---

#### 7.3.2 执行顺序说明

**推荐顺序**：
1. **先消耗资源**（避免无资源仍触发效果）
2. **再执行技能效果**（主要逻辑）
3. **最后记录冷却/次数**（确保技能成功执行）

**为什么这样排序？**
- 资源优先：确保有足够资源（虽然 CanActivate 已检查，但双重保险）
- 效果居中：主要逻辑
- 记录最后：确保技能完整执行后再进入冷却

---

### 7.4 完整技能实现示例

#### 7.4.1 普通攻击（无冷却）

```csharp
// 1. AbilityAsset 配置
public class BasicAttackAbilityAsset : AbilityAsset
{
    [Header("技能效果")]
    public GameplayEffect DamageEffect;
    
    [Header("冷却设置")]
    public int CooldownTurns = 0;  // 无冷却
    public int MaxUsesPerTurn = 0; // 无限制
}

// 2. AbilitySpec 实现
public class BasicAttackAbilitySpec : AbilitySpec<BasicAttackAbility>
{
    public override AbilityActivateResult CanActivate()
    {
        // 普通攻击只需基础检查
        return base.CanActivate();
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as BasicAttackAbilityAsset;
        
        // 攻击效果
        Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        
        // 无需记录冷却
        
        EndAbility();
    }
}
```

---

#### 7.4.2 大招（CD + 每回合限 1 次）

```csharp
// 1. AbilityAsset 配置
public class UltimateAbilityAsset : AbilityAsset
{
    [Header("资源消耗")]
    public GameplayEffect ManaCostEffect; // ManaCost_100.asset
    
    [Header("技能效果")]
    public GameplayEffect DamageEffect;
    
    [Header("冷却设置")]
    public int CooldownTurns = 5;      // CD 5 回合
    public int MaxUsesPerTurn = 1;     // 每回合限 1 次
    public int MaxUsesPerBattle = 0;   // 无限制
}

// 2. AbilitySpec 实现
public class UltimateAbilitySpec : AbilitySpec<UltimateAbility>
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        var asset = Ability.DataReference as UltimateAbilityAsset;
        
        // 检查冷却
        if (cooldownMgr.IsOnCooldown(this))
        {
            return AbilityActivateResult.Fail;
        }
        
        // 检查每回合次数限制
        if (!cooldownMgr.CanUseThisTurn(Ability.Name, asset.MaxUsesPerTurn))
        {
            Debug.Log("本回合已使用大招");
            return AbilityActivateResult.Fail;
        }
        
        // 检查 Mana
        var mana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
        if (mana < 100)
        {
            Debug.Log($"Mana 不足：需要 100，当前 {mana}");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as UltimateAbilityAsset;
        
        // 消耗 Mana
        Owner.ApplyGameplayEffectTo(asset.ManaCostEffect, Owner);
        
        // 技能效果
        Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        
        // 记录冷却
        cooldownMgr.StartCooldown(this, asset.CooldownTurns);
        
        // 记录使用次数
        cooldownMgr.RecordUsage(Ability.Name);
        
        Debug.Log($"{Owner.name} 使用大招！进入 {asset.CooldownTurns} 回合冷却");
        
        EndAbility();
    }
}
```

---

#### 7.4.3 终极技能（每战斗限 3 次 + 超长 CD）

```csharp
// 1. AbilityAsset 配置
public class LimitBreakAbilityAsset : AbilityAsset
{
    [Header("资源消耗")]
    public int RageCost = 100;  // 需要满怒气
    public GameplayEffect RageCostEffect;
    
    [Header("技能效果")]
    public GameplayEffect DamageEffect;
    
    [Header("冷却设置")]
    public int CooldownTurns = 8;      // CD 8 回合
    public int MaxUsesPerTurn = 0;     // 无限制
    public int MaxUsesPerBattle = 3;   // 每战斗限 3 次
}

// 2. AbilitySpec 实现
public class LimitBreakAbilitySpec : AbilitySpec<LimitBreakAbility>
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        var asset = Ability.DataReference as LimitBreakAbilityAsset;
        
        // 检查冷却
        if (cooldownMgr.IsOnCooldown(this))
        {
            return AbilityActivateResult.Fail;
        }
        
        // 检查每战斗次数限制
        if (!cooldownMgr.CanUseBattle(this, asset.MaxUsesPerBattle))
        {
            var used = cooldownMgr.GetBattleUsageCount(this);
            Debug.Log($"已达到战斗使用上限：{used}/{asset.MaxUsesPerBattle}");
            return AbilityActivateResult.Fail;
        }
        
        // 检查怒气
        var rage = Owner.GetAttributeCurrentValue("AS_Combat", "Rage") ?? 0;
        if (rage < asset.RageCost)
        {
            Debug.Log($"怒气不足：需要 {asset.RageCost}，当前 {rage}");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        var asset = Ability.DataReference as LimitBreakAbilityAsset;
        
        // 消耗怒气
        Owner.ApplyGameplayEffectTo(asset.RageCostEffect, Owner);
        
        // 技能效果（通常非常强力）
        Owner.ApplyGameplayEffectTo(asset.DamageEffect, target);
        
        // 记录冷却
        cooldownMgr.StartCooldown(this, asset.CooldownTurns);
        
        // 记录战斗使用次数
        cooldownMgr.RecordBattleUsage(this);
        
        var used = cooldownMgr.GetBattleUsageCount(this);
        Debug.Log($"{Owner.name} 使用终极技能！剩余次数：{asset.MaxUsesPerBattle - used}");
        
        EndAbility();
    }
}
```

---

### 7.5 与 Cost GE 协作

#### 7.5.1 消耗减免 Buff 集成

**场景**：Buff 使所有技能消耗减少 20%

```csharp
// 1. Cost GE 使用 MMC
public class CostReductionMMC : ModifierMagnitudeCalculation
{
    public float BaseCost = 50f;
    
    public override float CalculateMagnitude(GameplayEffectSpec spec)
    {
        var target = spec.Owner;
        
        float reduction = 1.0f;
        
        // ✅ 检查消耗减免 Buff
        if (target.HasTag(GTagLib.State_Buff_CostReduction))
        {
            reduction = 0.8f; // 减少 20%
        }
        
        return -(BaseCost * reduction);
    }
}

// 2. 在 CanActivate() 中计算实际消耗
public class ManaAbilitySpec : AbilitySpec<ManaAbility>
{
    public override AbilityActivateResult CanActivate()
    {
        var baseResult = base.CanActivate();
        if (baseResult != AbilityActivateResult.Success) return baseResult;
        
        // 计算实际消耗（考虑减免）
        var baseCost = 50f;
        var actualCost = baseCost;
        
        if (Owner.HasTag(GTagLib.State_Buff_CostReduction))
        {
            actualCost *= 0.8f; // 减少 20%
        }
        
        var mana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
        if (mana < actualCost)
        {
            Debug.Log($"Mana 不足：需要 {actualCost:F0}（原本 {baseCost}），当前 {mana}");
            return AbilityActivateResult.Fail;
        }
        
        return AbilityActivateResult.Success;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        var asset = Ability.DataReference as ManaAbilityAsset;
        
        // ✅ Cost GE 会自动应用减免（通过 MMC）
        Owner.ApplyGameplayEffectTo(asset.ManaCostEffect, Owner);
        
        // 技能效果
        // ...
    }
}
```

---

### 7.6 集成流程图

```
┌──────────────────────────────────────────────────────────┐
│                     技能使用完整流程                      │
└──────────────────────────────────────────────────────────┘

1. 玩家/AI 选择技能
   ↓
2. AbilitySpec.CanActivate()
   ├─ 检查基础 Tag 条件（base.CanActivate()）
   ├─ 检查回合冷却（cooldownMgr.IsOnCooldown()）
   ├─ 检查次数限制（cooldownMgr.CanUseThisTurn/Battle()）
   ├─ 检查共享冷却（cooldownMgr.IsSharedCooldownActive()）
   ├─ 检查资源（GetAttributeCurrentValue()）
   └─ 检查全局冷却（cooldownMgr.IsGlobalCooldownActive()）
   ↓
3. ✅ 条件通过，执行 AbilitySpec.ActivateAbility()
   ├─ 消耗资源（Owner.ApplyGameplayEffectTo(costGE)）
   ├─ 技能效果（Owner.ApplyGameplayEffectTo(damageGE)）
   ├─ 记录冷却（cooldownMgr.StartCooldown()）
   ├─ 记录次数（cooldownMgr.RecordUsage/BattleUsage()）
   ├─ 启动共享冷却（cooldownMgr.StartSharedCooldown()）
   └─ 启动全局冷却（cooldownMgr.StartGlobalCooldown()）
   ↓
4. 回合结束（BattleManager.EndCurrentTurn()）
   ├─ 冷却递减（cooldownMgr.DecrementCooldowns()）
   ├─ Buff 计数递减（effectManager.ProcessTurnEnd()）
   └─ 重置每回合次数（cooldownMgr.ResetTurnUsage()）
   ↓
5. 下一回合开始
   └─ 重复步骤 1
```

---

### 7.7 集成总结

**核心模式**：
1. **AbilitySpec 负责检查和记录**（通过 Manager API）
2. **Manager 负责存储和计算**（冷却数据、次数限制）
3. **BattleManager 负责驱动**（回合结束递减、回合开始重置）

**关键 API**：
- **检查类**：`IsOnCooldown`, `CanUseThisTurn`, `CanUseBattle`, `IsSharedCooldownActive`
- **记录类**：`StartCooldown`, `RecordUsage`, `RecordBattleUsage`, `StartSharedCooldown`
- **查询类**：`GetRemainingCooldown`, `GetBattleUsageCount`, `GetSharedCooldownRemaining`
- **操作类**：`DecrementCooldowns`, `ResetTurnUsage`, `ReduceCooldown`, `ResetCooldown`

**设计优势**：
- ✅ 职责明确：AbilitySpec 不存储冷却数据
- ✅ 易于扩展：新增冷却类型只需修改 Manager
- ✅ 统一管理：所有技能冷却集中查询、统一递减
- ✅ 符合 GAS：资源消耗用 Cost GE，符合黄金法则

---

## 八、优缺点、扩展性和测试

### 8.1 方案优点

#### 8.1.1 设计优势

| 优点 | 说明 | 对比传统方案 |
|------|------|-------------|
| **集中化管理** | 所有冷却数据集中在 Manager，易于全局操作（重置所有 CD） | 传统方案：分散在各 AbilitySpec，难以批量操作 |
| **职责明确** | AbilitySpec 不存储冷却，只负责逻辑 | 传统方案：AbilitySpec 既存储又计算，职责混乱 |
| **易于扩展** | 新增冷却类型（如：共享 CD）只需修改 Manager | 传统方案：需修改所有 AbilitySpec |
| **符合 GAS 理念** | 资源消耗用 Cost GE，冷却通过 Manager，无直接属性修改 | 传统方案：常见直接减属性，违反 GAS 黄金法则 |
| **三层冷却机制** | 回合 CD + 每回合限制 + 每战斗限制，覆盖所有场景 | 传统方案：通常只有简单 CD |
| **支持特殊机制** | 减 CD、重置 CD、共享 CD、全局 CD | 传统方案：难以实现这些机制 |
| **回合制友好** | 冷却以回合计数，非实时秒数 | 传统方案：常用秒数，不适合回合制 |
| **易于调试** | Runtime Watcher 查看所有冷却状态 | 传统方案：需逐个 AbilitySpec 查看 |

---

#### 8.1.2 性能优势

1. **无 GC 压力**：
   - 使用 `Dictionary<AbilitySpec, int>` 存储冷却
   - 无频繁分配/释放对象
   - 回合制游戏性能要求低，完全满足

2. **查询高效**：
   - `IsOnCooldown()` O(1) 字典查询
   - `GetRemainingCooldown()` O(1) 查询
   - 批量操作使用 LINQ Where 过滤，性能可接受

3. **内存占用低**：
   - 只存储在冷却中的技能（CD=0 自动移除）
   - 三个字典总内存占用 < 1KB（典型战斗）

---

#### 8.1.3 可维护性优势

1. **代码清晰**：
   ```csharp
   // ✅ 清晰的 API
   cooldownMgr.IsOnCooldown(ability);
   cooldownMgr.StartCooldown(ability, 3);
   cooldownMgr.ReduceCooldown(ability, 1);
   ```

2. **易于测试**：
   - Manager 可独立单元测试
   - 不依赖 Unity 运行时

3. **易于扩展**：
   - 新增冷却类型：修改 Manager
   - 新增特殊机制：添加 API

---

### 8.2 方案缺点与应对策略

#### 8.2.1 缺点 1：需要额外的 Manager 类

**问题**：
- 引入了新的依赖（TurnBasedAbilityCooldownManager）
- 需要在 BattleManager 中初始化和驱动

**应对策略**：
```csharp
// 将 Manager 集成到 BattleManager，统一管理
public class TurnBasedBattleManager : MonoBehaviour
{
    public TurnBasedAbilityCooldownManager CooldownManager { get; private set; }
    
    void Awake()
    {
        // 自动初始化
        CooldownManager = new TurnBasedAbilityCooldownManager();
    }
}

// AbilitySpec 中通过单例访问
private TurnBasedAbilityCooldownManager cooldownMgr => 
    TurnBasedBattleManager.Instance.CooldownManager;
```

---

#### 8.2.2 缺点 2：AbilitySpec 需要引用 Manager

**问题**：
- 每个 AbilitySpec 需要获取 Manager 引用
- 增加了一点耦合

**应对策略**：
```csharp
// 方案 A：通过单例访问（推荐）
private TurnBasedAbilityCooldownManager cooldownMgr => 
    TurnBasedBattleManager.Instance.CooldownManager;

// 方案 B：在 AbilitySpec 基类中提供
public abstract class TurnBasedAbilitySpec : AbilitySpec
{
    protected TurnBasedAbilityCooldownManager CooldownMgr => 
        TurnBasedBattleManager.Instance.CooldownManager;
}

// 方案 C：依赖注入（高级）
public class AbilitySpecFactory
{
    public AbilitySpec CreateSpec(Ability ability, AbilitySystemComponent owner)
    {
        var spec = new MyAbilitySpec(ability, owner);
        spec.InjectCooldownManager(TurnBasedBattleManager.Instance.CooldownManager);
        return spec;
    }
}
```

---

#### 8.2.3 缺点 3：共享冷却需要字符串 Key

**问题**：
- `sharedCooldownGroups` 使用字符串 Key
- 可能出现拼写错误

**应对策略**：
```csharp
// 方案 A：使用常量
public static class SharedCooldownGroups
{
    public const string Fireball = "Fireball";
    public const string WarriorStance = "WarriorStance";
    public const string GlobalCooldown = "GlobalCooldown";
}

// 使用
cooldownMgr.StartSharedCooldown(SharedCooldownGroups.Fireball, 2);

// 方案 B：使用 GameplayTag（更符合 GAS 理念）
public class SharedCooldownAbilityAsset : AbilityAsset
{
    public GameplayTag SharedCooldownTag; // 用 Tag 代替字符串
}

// 修改 Manager
private Dictionary<GameplayTag, int> sharedCooldownGroups = new();
```

---

### 8.3 扩展性分析

#### 8.3.1 网络同步支持

**场景**：多人回合制游戏需要同步冷却状态

```csharp
// 1. 序列化冷却数据
[System.Serializable]
public class CooldownSyncData
{
    public string AbilityName;
    public int RemainingCooldown;
}

public class TurnBasedAbilityCooldownManager
{
    /// <summary>
    /// 导出冷却数据（用于网络同步）
    /// </summary>
    public List<CooldownSyncData> ExportCooldowns(AbilitySystemComponent owner)
    {
        var data = new List<CooldownSyncData>();
        
        foreach (var (ability, cooldown) in cooldowns)
        {
            if (ability.Owner == owner)
            {
                data.Add(new CooldownSyncData
                {
                    AbilityName = ability.Ability.Name,
                    RemainingCooldown = cooldown
                });
            }
        }
        
        return data;
    }
    
    /// <summary>
    /// 导入冷却数据（从服务器同步）
    /// </summary>
    public void ImportCooldowns(AbilitySystemComponent owner, List<CooldownSyncData> data)
    {
        foreach (var item in data)
        {
            var ability = owner.AbilityContainer.AbilitySpecs()[item.AbilityName];
            cooldowns[ability] = item.RemainingCooldown;
        }
    }
}

// 2. 网络同步示例
public class NetworkBattleManager : MonoBehaviour
{
    void OnTurnEnd(int playerId)
    {
        if (IsServer)
        {
            var player = GetPlayer(playerId);
            
            // 服务器端递减冷却
            cooldownManager.DecrementCooldowns(player.ASC);
            
            // 同步给客户端
            var syncData = cooldownManager.ExportCooldowns(player.ASC);
            RpcSyncCooldowns(playerId, syncData);
        }
    }
    
    [ClientRpc]
    void RpcSyncCooldowns(int playerId, List<CooldownSyncData> data)
    {
        var player = GetPlayer(playerId);
        cooldownManager.ImportCooldowns(player.ASC, data);
    }
}
```

---

#### 8.3.2 存档系统支持

**场景**：保存/加载战斗状态

```csharp
// 1. 扩展 BattleSaveData
[System.Serializable]
public class BattleSaveData
{
    // ... 已有字段 ...
    
    public List<CooldownSaveData> Cooldowns;
    
    [System.Serializable]
    public class CooldownSaveData
    {
        public string UnitId;
        public string AbilityName;
        public int RemainingCooldown;
        public int TurnUsageCount;
        public int BattleUsageCount;
    }
}

// 2. 保存冷却状态
public void SaveCooldowns(BattleSaveData saveData)
{
    saveData.Cooldowns = new List<BattleSaveData.CooldownSaveData>();
    
    foreach (var (ability, cooldown) in cooldowns)
    {
        saveData.Cooldowns.Add(new BattleSaveData.CooldownSaveData
        {
            UnitId = ability.Owner.name,
            AbilityName = ability.Ability.Name,
            RemainingCooldown = cooldown,
            TurnUsageCount = turnUsageCount.GetValueOrDefault(ability.Ability.Name, 0),
            BattleUsageCount = battleUsageCount.GetValueOrDefault(ability, 0)
        });
    }
}

// 3. 加载冷却状态
public void LoadCooldowns(BattleSaveData saveData)
{
    cooldowns.Clear();
    turnUsageCount.Clear();
    battleUsageCount.Clear();
    
    foreach (var data in saveData.Cooldowns)
    {
        var unit = FindUnit(data.UnitId);
        var ability = unit.ASC.AbilityContainer.AbilitySpecs()[data.AbilityName];
        
        cooldowns[ability] = data.RemainingCooldown;
        turnUsageCount[data.AbilityName] = data.TurnUsageCount;
        battleUsageCount[ability] = data.BattleUsageCount;
    }
}
```

---

#### 8.3.3 UI 集成

**场景**：显示技能冷却状态

```csharp
// 1. UI 控制器
public class AbilityButtonUI : MonoBehaviour
{
    public AbilitySpec Ability;
    public Text CooldownText;
    public Image CooldownMask;
    public Button Button;
    
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    void Update()
    {
        if (cooldownMgr == null)
        {
            cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
            return;
        }
        
        // ✅ 查询冷却状态
        bool isOnCooldown = cooldownMgr.IsOnCooldown(Ability);
        
        if (isOnCooldown)
        {
            int remaining = cooldownMgr.GetRemainingCooldown(Ability);
            CooldownText.text = remaining.ToString();
            CooldownMask.fillAmount = remaining / (float)maxCooldown;
            Button.interactable = false;
        }
        else
        {
            CooldownText.text = "";
            CooldownMask.fillAmount = 0;
            Button.interactable = Ability.CanActivate() == AbilityActivateResult.Success;
        }
    }
}

// 2. 技能栏管理器
public class AbilityBarUI : MonoBehaviour
{
    public List<AbilityButtonUI> AbilityButtons;
    
    public void RefreshCooldowns(AbilitySystemComponent owner)
    {
        var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        
        foreach (var button in AbilityButtons)
        {
            var remaining = cooldownMgr.GetRemainingCooldown(button.Ability);
            button.UpdateCooldown(remaining);
        }
    }
}
```

---

#### 8.3.4 技能书系统

**场景**：学习新技能、升级技能

```csharp
// 1. 学习技能
public class SkillBookSystem
{
    public void LearnAbility(AbilitySystemComponent owner, string abilityName)
    {
        var ability = AbilityLib.CreateAbility(abilityName);
        owner.AbilityContainer.GiveAbility(ability);
        
        // ✅ 冷却系统无需额外配置，自动支持新技能
        Debug.Log($"学会技能：{abilityName}");
    }
}

// 2. 升级技能（减少冷却）
public class SkillUpgradeSystem
{
    public void UpgradeAbilityCooldown(AbilityAsset asset, int reduction)
    {
        asset.CooldownTurns -= reduction;
        asset.CooldownTurns = Mathf.Max(0, asset.CooldownTurns);
        
        Debug.Log($"技能升级：冷却减少 {reduction} 回合");
    }
}

// 3. 天赋系统（永久减少所有技能冷却）
public class TalentSystem
{
    public void ApplyCooldownReductionTalent(AbilitySystemComponent owner)
    {
        // 在回合结束时额外减少冷却
        // 参考 8.3.1 网络同步示例中的实现
    }
}
```

---

### 8.4 测试方案

#### 8.4.1 单元测试

```csharp
using NUnit.Framework;

[TestFixture]
public class CooldownManagerTests
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    private AbilitySpec testAbility;
    
    [SetUp]
    public void Setup()
    {
        cooldownMgr = new TurnBasedAbilityCooldownManager();
        testAbility = CreateMockAbility("TestAbility");
    }
    
    [Test]
    public void StartCooldown_ShouldSetCooldown()
    {
        // Arrange & Act
        cooldownMgr.StartCooldown(testAbility, 3);
        
        // Assert
        Assert.IsTrue(cooldownMgr.IsOnCooldown(testAbility));
        Assert.AreEqual(3, cooldownMgr.GetRemainingCooldown(testAbility));
    }
    
    [Test]
    public void DecrementCooldowns_ShouldReduceCooldown()
    {
        // Arrange
        cooldownMgr.StartCooldown(testAbility, 3);
        
        // Act
        cooldownMgr.DecrementCooldowns(testAbility.Owner);
        
        // Assert
        Assert.AreEqual(2, cooldownMgr.GetRemainingCooldown(testAbility));
    }
    
    [Test]
    public void DecrementCooldowns_ShouldRemoveWhenZero()
    {
        // Arrange
        cooldownMgr.StartCooldown(testAbility, 1);
        
        // Act
        cooldownMgr.DecrementCooldowns(testAbility.Owner);
        
        // Assert
        Assert.IsFalse(cooldownMgr.IsOnCooldown(testAbility));
    }
    
    [Test]
    public void CanUseThisTurn_ShouldRespectLimit()
    {
        // Arrange
        string abilityName = "TestAbility";
        int maxUses = 2;
        
        // Act & Assert
        Assert.IsTrue(cooldownMgr.CanUseThisTurn(abilityName, maxUses));
        
        cooldownMgr.RecordUsage(abilityName);
        Assert.IsTrue(cooldownMgr.CanUseThisTurn(abilityName, maxUses));
        
        cooldownMgr.RecordUsage(abilityName);
        Assert.IsFalse(cooldownMgr.CanUseThisTurn(abilityName, maxUses));
    }
    
    [Test]
    public void ResetTurnUsage_ShouldClearUsageCount()
    {
        // Arrange
        string abilityName = "TestAbility";
        cooldownMgr.RecordUsage(abilityName);
        cooldownMgr.RecordUsage(abilityName);
        
        // Act
        cooldownMgr.ResetTurnUsage();
        
        // Assert
        Assert.IsTrue(cooldownMgr.CanUseThisTurn(abilityName, 1));
    }
    
    [Test]
    public void ReduceCooldown_ShouldReduceRemaining()
    {
        // Arrange
        cooldownMgr.StartCooldown(testAbility, 5);
        
        // Act
        cooldownMgr.ReduceCooldown(testAbility, 2);
        
        // Assert
        Assert.AreEqual(3, cooldownMgr.GetRemainingCooldown(testAbility));
    }
    
    [Test]
    public void ResetCooldown_ShouldRemoveCooldown()
    {
        // Arrange
        cooldownMgr.StartCooldown(testAbility, 5);
        
        // Act
        cooldownMgr.ResetCooldown(testAbility);
        
        // Assert
        Assert.IsFalse(cooldownMgr.IsOnCooldown(testAbility));
    }
}
```

---

#### 8.4.2 集成测试

```csharp
[TestFixture]
public class AbilityIntegrationTests
{
    private TurnBasedBattleManager battleMgr;
    private AbilitySystemComponent player;
    private AbilitySystemComponent enemy;
    
    [SetUp]
    public void Setup()
    {
        // 初始化战斗环境
        battleMgr = CreateMockBattleManager();
        player = CreateMockPlayer();
        enemy = CreateMockEnemy();
    }
    
    [Test]
    public void Fireball_ShouldEnterCooldown()
    {
        // Arrange
        var fireballSpec = player.AbilityContainer.AbilitySpecs()["Fireball"];
        
        // Act
        player.TryActivateAbility("Fireball", enemy);
        
        // Assert
        Assert.IsTrue(battleMgr.CooldownManager.IsOnCooldown(fireballSpec));
        Assert.AreEqual(3, battleMgr.CooldownManager.GetRemainingCooldown(fireballSpec));
    }
    
    [Test]
    public void CooldownShouldDecrement_AfterTurnEnd()
    {
        // Arrange
        var fireballSpec = player.AbilityContainer.AbilitySpecs()["Fireball"];
        player.TryActivateAbility("Fireball", enemy);
        
        // Act
        battleMgr.EndCurrentTurn();
        
        // Assert
        Assert.AreEqual(2, battleMgr.CooldownManager.GetRemainingCooldown(fireballSpec));
    }
    
    [Test]
    public void Ultimate_ShouldRespectPerTurnLimit()
    {
        // Arrange
        var ultimateSpec = player.AbilityContainer.AbilitySpecs()["Ultimate"];
        
        // Act - 第一次使用
        bool firstUse = player.TryActivateAbility("Ultimate", enemy);
        
        // Act - 第二次尝试使用
        bool secondUse = player.TryActivateAbility("Ultimate", enemy);
        
        // Assert
        Assert.IsTrue(firstUse);
        Assert.IsFalse(secondUse); // 每回合限 1 次
    }
}
```

---

#### 8.4.3 边缘场景测试

```csharp
[TestFixture]
public class EdgeCaseTests
{
    [Test]
    public void Cooldown_ShouldNotGoNegative()
    {
        // Arrange
        var cooldownMgr = new TurnBasedAbilityCooldownManager();
        var ability = CreateMockAbility("TestAbility");
        
        cooldownMgr.StartCooldown(ability, 1);
        
        // Act - 递减 3 次
        cooldownMgr.DecrementCooldowns(ability.Owner);
        cooldownMgr.DecrementCooldowns(ability.Owner);
        cooldownMgr.DecrementCooldowns(ability.Owner);
        
        // Assert - 应该是 0，不是负数
        Assert.AreEqual(0, cooldownMgr.GetRemainingCooldown(ability));
    }
    
    [Test]
    public void ReduceCooldown_BeyondZero_ShouldClampToZero()
    {
        // Arrange
        var cooldownMgr = new TurnBasedAbilityCooldownManager();
        var ability = CreateMockAbility("TestAbility");
        
        cooldownMgr.StartCooldown(ability, 2);
        
        // Act - 减少 5 回合（超过当前冷却）
        cooldownMgr.ReduceCooldown(ability, 5);
        
        // Assert - 应该移除冷却
        Assert.IsFalse(cooldownMgr.IsOnCooldown(ability));
    }
    
    [Test]
    public void BattleUsage_ShouldPersistAcrossTurns()
    {
        // Arrange
        var cooldownMgr = new TurnBasedAbilityCooldownManager();
        var ability = CreateMockAbility("LimitBreak");
        
        // Act - 使用 3 次（跨多个回合）
        cooldownMgr.RecordBattleUsage(ability);
        cooldownMgr.ResetTurnUsage(); // 回合重置不影响战斗次数
        
        cooldownMgr.RecordBattleUsage(ability);
        cooldownMgr.ResetTurnUsage();
        
        cooldownMgr.RecordBattleUsage(ability);
        
        // Assert
        Assert.AreEqual(3, cooldownMgr.GetBattleUsageCount(ability));
        Assert.IsFalse(cooldownMgr.CanUseBattle(ability, 3));
    }
}
```

---

### 8.5 优缺点与测试总结

**优点总结**：
- ✅ 集中化管理，易于全局操作
- ✅ 职责明确，AbilitySpec 不存储冷却
- ✅ 三层冷却机制，覆盖所有场景
- ✅ 支持特殊机制（减 CD、共享 CD、全局 CD）
- ✅ 易于扩展（网络同步、存档、UI、技能书）
- ✅ 易于测试（Manager 可独立单元测试）

**缺点与应对**：
- ⚠️ 需要额外 Manager 类 → 集成到 BattleManager
- ⚠️ AbilitySpec 需要引用 Manager → 通过单例或基类
- ⚠️ 共享冷却用字符串 Key → 用常量或 GameplayTag

**测试覆盖**：
- ✅ 单元测试：Manager 核心功能
- ✅ 集成测试：与 Ability 协作
- ✅ 边缘场景：负数、超限、跨回合

---

**下一章**：备选方案对比（3 种方案对比分析）

---

## 九、备选方案对比

### 9.1 方案 A：独立 CooldownManager（本文档推荐方案）

#### 9.1.1 架构

```
┌──────────────────────────────────────────┐
│    TurnBasedAbilityCooldownManager       │
│  ┌────────────────────────────────────┐  │
│  │ Dictionary<AbilitySpec, int>       │  │ ← 冷却数据
│  │ Dictionary<string, int>            │  │ ← 每回合次数
│  │ Dictionary<AbilitySpec, int>       │  │ ← 每战斗次数
│  └────────────────────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │ IsOnCooldown(), StartCooldown()   │  │ ← 核心 API
│  │ ReduceCooldown(), ResetCooldown()  │  │ ← 特殊操作
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
                    ▲
                    │ 引用
    ┌───────────────┴───────────────┐
    │                               │
┌───────────────┐         ┌──────────────────┐
│ AbilitySpec   │         │ BattleManager    │
│ - CanActivate │         │ - DecrementCDs   │
│ - Activate    │         │ - ResetTurnUsage │
└───────────────┘         └──────────────────┘
```

**核心代码**：
```csharp
// Manager 存储数据
public class TurnBasedAbilityCooldownManager
{
    private Dictionary<AbilitySpec, int> cooldowns = new();
    
    public void StartCooldown(AbilitySpec ability, int turns)
    {
        cooldowns[ability] = turns;
    }
    
    public bool IsOnCooldown(AbilitySpec ability)
    {
        return cooldowns.ContainsKey(ability) && cooldowns[ability] > 0;
    }
}

// AbilitySpec 查询 Manager
public class FireballAbilitySpec : AbilitySpec
{
    public override bool CanActivate()
    {
        return !cooldownMgr.IsOnCooldown(this);
    }
    
    public override void ActivateAbility(params object[] args)
    {
        // 技能效果
        // ...
        
        cooldownMgr.StartCooldown(this, 3);
    }
}
```

---

#### 9.1.2 优缺点

**优点**：
- ✅ 职责明确：Manager 存储，Spec 查询
- ✅ 易于全局操作：重置所有 CD、减少所有 CD
- ✅ 易于扩展：新增冷却类型只需修改 Manager
- ✅ 易于测试：Manager 可独立单元测试
- ✅ 支持特殊机制：共享 CD、全局 CD、减 CD

**缺点**：
- ⚠️ 需要额外的 Manager 类
- ⚠️ AbilitySpec 需要引用 Manager
- ⚠️ 轻微性能开销（字典查询）

**推荐度**：⭐⭐⭐⭐⭐（强烈推荐）

---

### 9.2 方案 B：集成到 AbilitySpec（分散管理）

#### 9.2.1 架构

```
┌──────────────────────────────────────────┐
│          AbilitySpec（每个技能）          │
│  ┌────────────────────────────────────┐  │
│  │ int cooldownTurns                  │  │ ← 最大冷却
│  │ int remainingCooldown              │  │ ← 当前冷却
│  │ int turnUsageCount                 │  │ ← 本回合使用次数
│  │ int battleUsageCount               │  │ ← 本战斗使用次数
│  └────────────────────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │ CanActivate()  → 检查自己的冷却    │  │
│  │ ActivateAbility() → 设置自己的冷却 │  │
│  │ DecrementCooldown() → 递减自己冷却 │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
                    ▲
                    │ BattleManager 逐个递减
    ┌───────────────┴───────────────┐
    │ foreach (var ability in abilities) │
    │     ability.DecrementCooldown();   │
    └────────────────────────────────────┘
```

**核心代码**：
```csharp
// 每个 AbilitySpec 存储自己的冷却
public class FireballAbilitySpec : AbilitySpec
{
    private int cooldownTurns = 3;
    private int remainingCooldown = 0;
    private int turnUsageCount = 0;
    
    public override bool CanActivate()
    {
        if (remainingCooldown > 0)
        {
            Debug.Log($"冷却中：{remainingCooldown} 回合");
            return false;
        }
        
        return true;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        // 技能效果
        // ...
        
        // 设置冷却
        remainingCooldown = cooldownTurns;
        turnUsageCount++;
    }
    
    public void DecrementCooldown()
    {
        if (remainingCooldown > 0)
        {
            remainingCooldown--;
        }
    }
    
    public void ResetTurnUsage()
    {
        turnUsageCount = 0;
    }
}

// BattleManager 需要逐个递减
public class TurnBasedBattleManager : MonoBehaviour
{
    void EndCurrentTurn()
    {
        var currentUnit = AllUnits[currentUnitIndex];
        
        // ❌ 需要遍历所有技能
        foreach (var ability in currentUnit.ASC.AbilityContainer.AbilitySpecs().Values)
        {
            if (ability is FireballAbilitySpec fireballSpec)
            {
                fireballSpec.DecrementCooldown();
            }
            // ... 其他技能类型
        }
    }
}
```

---

#### 9.2.2 优缺点

**优点**：
- ✅ 无需额外 Manager 类
- ✅ 数据封装在 AbilitySpec 内（符合 OOP）
- ✅ 无字典查询开销

**缺点**：
- ❌ 职责混乱：AbilitySpec 既存储又计算
- ❌ 难以全局操作：重置所有 CD 需要遍历所有技能
- ❌ 难以扩展：新增冷却类型需要修改所有 AbilitySpec
- ❌ 难以实现共享 CD：需要外部状态
- ❌ 难以测试：需要完整的 ASC 环境
- ❌ 代码重复：每个技能都需要实现相同的冷却逻辑

**推荐度**：⭐⭐（不推荐）

---

### 9.3 方案 C：用 GE 模拟冷却（滥用 GE）

#### 9.3.1 架构

```
┌──────────────────────────────────────────┐
│       用 GameplayEffect 模拟冷却          │
│  ┌────────────────────────────────────┐  │
│  │ CooldownGE (Duration = 3 turns)    │  │ ← 冷却 GE
│  │ GrantedTags: Cooldown.Fireball     │  │ ← 冷却 Tag
│  └────────────────────────────────────┘  │
│  ┌────────────────────────────────────┐  │
│  │ CanActivate():                     │  │
│  │   if (Owner.HasTag(Cooldown.Fireball))│
│  │       return Fail;                 │  │
│  │ ActivateAbility():                 │  │
│  │   Owner.ApplyGE(CooldownGE);       │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

**核心代码**：
```csharp
// 1. 创建冷却 GE
// FireballCooldownGE.asset
DurationPolicy: Duration
DurationMagnitude: 3  // 3 回合
GrantedTags:
  - Cooldown.Fireball

// 2. 在 AbilitySpec 中使用
public class FireballAbilitySpec : AbilitySpec
{
    public GameplayEffect CooldownGE; // 拖入 FireballCooldownGE.asset
    
    public override bool CanActivate()
    {
        // ❌ 检查冷却 Tag
        if (Owner.HasTag(GTagLib.Cooldown_Fireball))
        {
            Debug.Log("冷却中");
            return false;
        }
        
        return true;
    }
    
    public override void ActivateAbility(params object[] args)
    {
        // 技能效果
        // ...
        
        // ❌ 应用冷却 GE
        Owner.ApplyGameplayEffectTo(CooldownGE, Owner);
    }
}

// 3. ❌ 问题：Duration GE 用的是实时秒数，非回合数
// 需要修改 GE 系统支持回合计时，或者手动管理 Duration
```

---

#### 9.3.2 优缺点

**优点**：
- ✅ 符合 UE4 GAS 原生冷却机制
- ✅ 利用 Tag 系统检查冷却
- ✅ 可以通过 RemoveGameplayEffectsWithTags 批量移除

**缺点**：
- ❌ GE 的 Duration 是实时秒数，非回合数（需要改造）
- ❌ 滥用 GE 系统：冷却不是属性修改，不应该用 GE
- ❌ 性能开销：每个冷却创建一个 GE Spec
- ❌ 难以实现次数限制（每回合限 N 次）
- ❌ 难以实现减 CD（需要修改 GE Duration）
- ❌ Tag 污染：Cooldown.* Tag 会占用 Tag 命名空间
- ❌ 不支持查询剩余冷却（GE 没有"剩余时间"概念）

**推荐度**：⭐（强烈不推荐）

---

### 9.4 三种方案对比表

| 维度 | 方案 A：CooldownManager | 方案 B：集成到 AbilitySpec | 方案 C：用 GE 模拟 |
|------|------------------------|--------------------------|-------------------|
| **侵入性** | ⭐⭐ 需引用 Manager | ⭐ 无需外部依赖 | ⭐⭐⭐ 需改造 GE 系统 |
| **职责分离** | ⭐⭐⭐ 清晰 | ⭐ 混乱 | ⭐⭐ 一般 |
| **全局操作** | ⭐⭐⭐ 简单 | ⭐ 困难 | ⭐⭐ 中等 |
| **扩展性** | ⭐⭐⭐ 优秀 | ⭐ 差 | ⭐ 差 |
| **性能** | ⭐⭐ 字典查询 | ⭐⭐⭐ 直接访问 | ⭐ GE Spec 开销 |
| **可测试性** | ⭐⭐⭐ 优秀 | ⭐⭐ 一般 | ⭐ 困难 |
| **支持共享 CD** | ⭐⭐⭐ 原生支持 | ⭐ 需要外部状态 | ⭐ 困难 |
| **支持全局 CD** | ⭐⭐⭐ 原生支持 | ⭐ 困难 | ⭐⭐ 用 Tag 可实现 |
| **支持减 CD** | ⭐⭐⭐ 简单 | ⭐⭐ 需要接口 | ⭐ 非常困难 |
| **支持次数限制** | ⭐⭐⭐ 原生支持 | ⭐⭐ 需要额外字段 | ⭐ 无法实现 |
| **查询剩余冷却** | ⭐⭐⭐ 简单 | ⭐⭐⭐ 简单 | ❌ 无法实现 |
| **符合 GAS 理念** | ⭐⭐⭐ 符合 | ⭐⭐ 一般 | ⭐ 滥用 GE |
| **代码重复** | ⭐⭐⭐ 无重复 | ⭐ 每个技能重复 | ⭐⭐ 一般 |
| **推荐度** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐ |

---

### 9.5 方案选择建议

#### 9.5.1 推荐 方案 A 的场景（90% 情况）

- ✅ 回合制游戏（卡牌、策略、RPG）
- ✅ 需要特殊机制（减 CD、共享 CD、全局 CD）
- ✅ 需要全局操作（重置所有 CD）
- ✅ 需要 UI 显示（查询剩余冷却）
- ✅ 需要网络同步/存档
- ✅ 重视代码质量和可维护性

**示例游戏**：
- 卡牌游戏（炉石传说、万智牌）
- 回合制 RPG（最终幻想战略版）
- 回合制策略（XCOM、火焰纹章）

---

#### 9.5.2 可以考虑 方案 B 的场景（少数情况）

- ⚠️ 技能数量极少（< 5 个）
- ⚠️ 无需特殊机制（只有简单 CD）
- ⚠️ 无需全局操作
- ⚠️ 项目极简，不想引入额外类

**示例游戏**：
- 简单的解谜游戏（技能只有"提示"）
- 极简的原型项目

---

#### 9.5.3 不推荐 方案 C 的原因

- ❌ GE 的 Duration 是实时秒数，非回合数
- ❌ 无法查询剩余冷却
- ❌ 无法实现次数限制
- ❌ 滥用 GE 系统
- ❌ 性能和内存开销
- ❌ 代码复杂度高

**唯一例外**：
- 如果你的游戏是实时制（ARPG、MOBA），且愿意改造 GE 的 Duration 系统支持自定义时间单位

---

### 9.6 方案对比总结

**最佳实践**：
1. **回合制游戏 → 方案 A（CooldownManager）** ✅
2. **实时制游戏 → UE4 GAS 原生冷却（CooldownTags + Duration GE）**
3. **极简项目 → 方案 B（集成到 AbilitySpec）**
4. **任何情况都不要用方案 C**（除非改造 GE 系统）

**本文档推荐**：方案 A（TurnBasedAbilityCooldownManager）
- ✅ 职责清晰、易于扩展、功能完整
- ✅ 支持三层冷却机制（回合 CD、每回合限制、每战斗限制）
- ✅ 支持特殊机制（减 CD、共享 CD、全局 CD）
- ✅ 符合 GAS 理念，易于测试

---

## 十、实现步骤与完整示例

### 10.1 三天实现计划

#### Day 1：基础冷却系统（4-6 小时）

**目标**：实现 TurnBasedAbilityCooldownManager 核心功能

**步骤**：
1. **创建 Manager 类**（1 小时）
   ```csharp
   // Assets/GAS/Runtime/Ability/TurnBasedAbilityCooldownManager.cs
   public class TurnBasedAbilityCooldownManager
   {
       private Dictionary<AbilitySpec, int> cooldowns = new();
       
       public void StartCooldown(AbilitySpec ability, int turns) { }
       public bool IsOnCooldown(AbilitySpec ability) { }
       public int GetRemainingCooldown(AbilitySpec ability) { }
       public void DecrementCooldowns(AbilitySystemComponent owner) { }
   }
   ```

2. **集成到 BattleManager**（1 小时）
   ```csharp
   public class TurnBasedBattleManager : MonoBehaviour
   {
       public TurnBasedAbilityCooldownManager CooldownManager { get; private set; }
       
       void Awake()
       {
           CooldownManager = new TurnBasedAbilityCooldownManager();
       }
       
       void EndCurrentTurn()
       {
           var currentUnit = AllUnits[currentUnitIndex];
           CooldownManager.DecrementCooldowns(currentUnit.ASC);
           // ...
       }
   }
   ```

3. **修改 AbilityAsset 和 AbilitySpec**（2 小时）
   ```csharp
   // AbilityAsset 添加字段
   public class MyAbilityAsset : AbilityAsset
   {
       public int CooldownTurns = 3;
   }
   
   // AbilitySpec 集成冷却
   public class MyAbilitySpec : AbilitySpec
   {
       private TurnBasedAbilityCooldownManager cooldownMgr;
       
       public override AbilityActivateResult CanActivate()
       {
           if (cooldownMgr.IsOnCooldown(this))
               return AbilityActivateResult.Fail;
           
           return base.CanActivate();
       }
       
       public override void ActivateAbility(params object[] args)
       {
           // 技能效果
           // ...
           
           var asset = Ability.DataReference as MyAbilityAsset;
           cooldownMgr.StartCooldown(this, asset.CooldownTurns);
       }
   }
   ```

4. **测试基础功能**（1 小时）
   - 技能使用后进入冷却
   - 回合结束冷却递减
   - 冷却结束后可再次使用

**交付物**：
- ✅ 基础冷却系统可用
- ✅ 至少 1 个技能支持冷却

---

#### Day 2：次数限制与资源消耗（4-6 小时）

**目标**：实现次数限制和 Cost GE 系统

**步骤**：
1. **添加次数限制功能**（2 小时）
   ```csharp
   // Manager 添加方法
   public class TurnBasedAbilityCooldownManager
   {
       private Dictionary<string, int> turnUsageCount = new();
       private Dictionary<AbilitySpec, int> battleUsageCount = new();
       
       public bool CanUseThisTurn(string abilityName, int maxUses) { }
       public bool CanUseBattle(AbilitySpec ability, int maxUses) { }
       public void RecordUsage(string abilityName) { }
       public void RecordBattleUsage(AbilitySpec ability) { }
       public void ResetTurnUsage() { }
   }
   
   // BattleManager 调用
   void StartNextTurn()
   {
       CooldownManager.ResetTurnUsage();
       // ...
   }
   ```

2. **创建 Cost GE**（1 小时）
   ```csharp
   // Unity 中创建 GameplayEffect ScriptableObject
   // ManaCost_50.asset
   DurationPolicy: Instant
   Modifiers:
     - Attribute: AS_Combat.Mana
       Operation: Add
       Magnitude: ScalableFloat(-50)
   
   // AbilityAsset 引用
   public class ManaAbilityAsset : AbilityAsset
   {
       public GameplayEffect ManaCostEffect; // 拖入 ManaCost_50.asset
   }
   ```

3. **修改 AbilitySpec 检查资源**（2 小时）
   ```csharp
   public override AbilityActivateResult CanActivate()
   {
       var asset = Ability.DataReference as ManaAbilityAsset;
       
       // 检查冷却
       if (cooldownMgr.IsOnCooldown(this))
           return AbilityActivateResult.Fail;
       
       // 检查每回合次数
       if (!cooldownMgr.CanUseThisTurn(Ability.Name, asset.MaxUsesPerTurn))
           return AbilityActivateResult.Fail;
       
       // 检查 Mana
       var mana = Owner.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
       if (mana < 50)
           return AbilityActivateResult.Fail;
       
       return base.CanActivate();
   }
   
   public override void ActivateAbility(params object[] args)
   {
       var asset = Ability.DataReference as ManaAbilityAsset;
       
       // 消耗 Mana
       Owner.ApplyGameplayEffectTo(asset.ManaCostEffect, Owner);
       
       // 技能效果
       // ...
       
       // 记录冷却和次数
       cooldownMgr.StartCooldown(this, asset.CooldownTurns);
       cooldownMgr.RecordUsage(Ability.Name);
   }
   ```

4. **测试次数限制和资源消耗**（1 小时）
   - 每回合限制生效
   - 每战斗限制生效
   - Mana 不足时无法使用
   - Mana 消耗后正确扣除

**交付物**：
- ✅ 次数限制系统可用
- ✅ Cost GE 资源消耗可用
- ✅ 至少 2 个技能支持（普通攻击 + 大招）

---

#### Day 3：特殊机制与优化（4-6 小时）

**目标**：实现减 CD、共享 CD、UI 集成

**步骤**：
1. **添加减 CD 功能**（1.5 小时）
   ```csharp
   // Manager 添加方法
   public void ReduceCooldown(AbilitySpec ability, int reduction) { }
   public void ResetCooldown(AbilitySpec ability) { }
   public void ReduceAllCooldowns(AbilitySystemComponent owner, int reduction) { }
   public void ResetAllCooldowns(AbilitySystemComponent owner) { }
   
   // 测试：使用道具减 CD
   cooldownMgr.ReduceAllCooldowns(player.ASC, 2);
   ```

2. **添加共享冷却组**（1.5 小时）
   ```csharp
   // Manager 添加方法
   private Dictionary<string, int> sharedCooldownGroups = new();
   
   public void StartSharedCooldown(string groupName, int turns) { }
   public bool IsSharedCooldownActive(string groupName) { }
   public int GetSharedCooldownRemaining(string groupName) { }
   
   // AbilityAsset 配置
   public class SharedCooldownAbilityAsset : AbilityAsset
   {
       public string SharedCooldownGroup = "Fireball";
       public int SharedCooldownTurns = 2;
   }
   
   // AbilitySpec 检查
   if (cooldownMgr.IsSharedCooldownActive(asset.SharedCooldownGroup))
       return AbilityActivateResult.Fail;
   
   cooldownMgr.StartSharedCooldown(asset.SharedCooldownGroup, asset.SharedCooldownTurns);
   ```

3. **UI 集成**（2 小时）
   ```csharp
   // AbilityButtonUI.cs
   public class AbilityButtonUI : MonoBehaviour
   {
       public AbilitySpec Ability;
       public Text CooldownText;
       public Image CooldownMask;
       
       void Update()
       {
           var cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
           
           if (cooldownMgr.IsOnCooldown(Ability))
           {
               int remaining = cooldownMgr.GetRemainingCooldown(Ability);
               CooldownText.text = remaining.ToString();
               CooldownMask.fillAmount = remaining / (float)maxCooldown;
           }
           else
           {
               CooldownText.text = "";
               CooldownMask.fillAmount = 0;
           }
       }
   }
   ```

4. **性能优化与测试**（1 小时）
   - 使用 Profiler 检查性能
   - 测试完整战斗流程（5 回合）
   - 测试边缘场景（冷却负数、次数溢出）

**交付物**：
- ✅ 减 CD 功能可用
- ✅ 共享 CD 可用
- ✅ UI 显示冷却状态
- ✅ 性能满足要求

---

### 10.2 完整战斗示例

#### 10.2.1 场景设计

**单位**：
- **玩家（Player）**
  - Attribute: Health=100, Mana=150
  - 技能：
    1. BasicAttack（普通攻击）：无冷却，无消耗
    2. Fireball（火球术）：CD 3 回合，消耗 50 Mana
    3. Heal（治疗）：CD 2 回合，消耗 30 Mana
    4. Ultimate（大招）：CD 5 回合，每回合限 1 次，消耗 100 Mana

- **敌人（Enemy）**
  - Attribute: Health=80
  - 技能：BasicAttack

**道具**：
- "刷新药水"：减少所有技能冷却 2 回合

---

#### 10.2.2 完整流程代码

```csharp
public class CompleteBattleExample : MonoBehaviour
{
    private TurnBasedBattleManager battleMgr;
    private TurnBasedAbilityCooldownManager cooldownMgr;
    private BattleUnit player;
    private BattleUnit enemy;
    
    void Start()
    {
        InitBattle();
        StartCoroutine(SimulateBattle());
    }
    
    void InitBattle()
    {
        battleMgr = TurnBasedBattleManager.Instance;
        cooldownMgr = battleMgr.CooldownManager;
        
        player = battleMgr.AllUnits.Find(u => u.Team == 0);
        enemy = battleMgr.AllUnits.Find(u => u.Team == 1);
        
        Debug.Log("=== 战斗初始化 ===");
        Debug.Log($"玩家：Health={GetHealth(player.ASC)}, Mana={GetMana(player.ASC)}");
        Debug.Log($"敌人：Health={GetHealth(enemy.ASC)}");
    }
    
    IEnumerator SimulateBattle()
    {
        yield return new WaitForSeconds(1);
        
        // --- 回合 1 ---
        Debug.Log("\n[回合 1] 玩家使用 Fireball");
        player.ASC.TryActivateAbility("Fireball", enemy.ASC);
        LogCooldowns();
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 1] 敌人使用 BasicAttack");
        enemy.ASC.TryActivateAbility("BasicAttack", player.ASC);
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 1] 回合结束");
        EndTurn();
        yield return new WaitForSeconds(1);
        
        // --- 回合 2 ---
        Debug.Log("\n[回合 2] 玩家使用 Heal");
        player.ASC.TryActivateAbility("Heal", player.ASC);
        LogCooldowns();
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 2] 敌人使用 BasicAttack");
        enemy.ASC.TryActivateAbility("BasicAttack", player.ASC);
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 2] 回合结束");
        EndTurn();
        yield return new WaitForSeconds(1);
        
        // --- 回合 3 ---
        Debug.Log("\n[回合 3] 玩家尝试使用 Fireball（冷却中）");
        bool canUse = player.ASC.TryActivateAbility("Fireball", enemy.ASC);
        Debug.Log($"→ 结果：{(canUse ? "成功" : "失败（冷却中）")}");
        
        Debug.Log("[回合 3] 玩家使用 BasicAttack");
        player.ASC.TryActivateAbility("BasicAttack", enemy.ASC);
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 3] 敌人使用 BasicAttack");
        enemy.ASC.TryActivateAbility("BasicAttack", player.ASC);
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 3] 回合结束");
        EndTurn();
        yield return new WaitForSeconds(1);
        
        // --- 回合 4 ---
        Debug.Log("\n[回合 4] 玩家使用刷新药水");
        UseRefreshPotion();
        LogCooldowns();
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 4] 玩家使用 Fireball（冷却已重置）");
        player.ASC.TryActivateAbility("Fireball", enemy.ASC);
        LogCooldowns();
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 4] 敌人使用 BasicAttack");
        enemy.ASC.TryActivateAbility("BasicAttack", player.ASC);
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 4] 回合结束");
        EndTurn();
        yield return new WaitForSeconds(1);
        
        // --- 回合 5 ---
        Debug.Log("\n[回合 5] 玩家使用 Ultimate");
        player.ASC.TryActivateAbility("Ultimate", enemy.ASC);
        LogCooldowns();
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 5] 敌人血量检查");
        if (GetHealth(enemy.ASC) <= 0)
        {
            Debug.Log("=== 战斗胜利 ===");
            yield break;
        }
        
        Debug.Log("[回合 5] 敌人使用 BasicAttack");
        enemy.ASC.TryActivateAbility("BasicAttack", player.ASC);
        yield return new WaitForSeconds(1);
        
        Debug.Log("[回合 5] 回合结束");
        EndTurn();
        
        Debug.Log("\n=== 战斗模拟完成 ===");
    }
    
    void EndTurn()
    {
        // 递减冷却
        cooldownMgr.DecrementCooldowns(player.ASC);
        cooldownMgr.DecrementCooldowns(enemy.ASC);
        
        // 重置每回合使用次数
        cooldownMgr.ResetTurnUsage();
        
        Debug.Log("→ 冷却递减完成");
    }
    
    void UseRefreshPotion()
    {
        cooldownMgr.ReduceAllCooldowns(player.ASC, 2);
        Debug.Log("→ 使用刷新药水，所有技能冷却 -2 回合");
    }
    
    void LogCooldowns()
    {
        var fireballSpec = player.ASC.AbilityContainer.AbilitySpecs()["Fireball"];
        var healSpec = player.ASC.AbilityContainer.AbilitySpecs()["Heal"];
        var ultimateSpec = player.ASC.AbilityContainer.AbilitySpecs()["Ultimate"];
        
        Debug.Log($"→ 冷却状态：Fireball={cooldownMgr.GetRemainingCooldown(fireballSpec)}, " +
                  $"Heal={cooldownMgr.GetRemainingCooldown(healSpec)}, " +
                  $"Ultimate={cooldownMgr.GetRemainingCooldown(ultimateSpec)}");
        
        Debug.Log($"→ 资源状态：Mana={GetMana(player.ASC)}");
    }
    
    float GetHealth(AbilitySystemComponent asc)
    {
        return asc.GetAttributeCurrentValue("AS_Combat", "Health") ?? 0;
    }
    
    float GetMana(AbilitySystemComponent asc)
    {
        return asc.GetAttributeCurrentValue("AS_Combat", "Mana") ?? 0;
    }
}
```

---

#### 10.2.3 预期输出

```
=== 战斗初始化 ===
玩家：Health=100, Mana=150
敌人：Health=80

[回合 1] 玩家使用 Fireball
→ 消耗 50 Mana（剩余 100）
→ 造成 30 伤害（敌人剩余 50）
→ Fireball 进入 3 回合冷却
→ 冷却状态：Fireball=3, Heal=0, Ultimate=0
→ 资源状态：Mana=100

[回合 1] 敌人使用 BasicAttack
→ 造成 10 伤害（玩家剩余 90）

[回合 1] 回合结束
→ 冷却递减完成
→ Fireball CD 3 → 2

[回合 2] 玩家使用 Heal
→ 消耗 30 Mana（剩余 70）
→ 恢复 20 生命（玩家 90 → 100）
→ Heal 进入 2 回合冷却
→ 冷却状态：Fireball=2, Heal=2, Ultimate=0
→ 资源状态：Mana=70

[回合 2] 敌人使用 BasicAttack
→ 造成 10 伤害（玩家剩余 90）

[回合 2] 回合结束
→ 冷却递减完成
→ Fireball CD 2 → 1
→ Heal CD 2 → 1

[回合 3] 玩家尝试使用 Fireball（冷却中）
→ 结果：失败（冷却中）
→ Fireball 剩余冷却：1 回合

[回合 3] 玩家使用 BasicAttack
→ 造成 15 伤害（敌人剩余 35）

[回合 3] 敌人使用 BasicAttack
→ 造成 10 伤害（玩家剩余 80）

[回合 3] 回合结束
→ 冷却递减完成
→ Fireball CD 1 → 0（可用）
→ Heal CD 1 → 0（可用）

[回合 4] 玩家使用刷新药水
→ 使用刷新药水，所有技能冷却 -2 回合
→ 冷却状态：Fireball=0, Heal=0, Ultimate=0

[回合 4] 玩家使用 Fireball（冷却已重置）
→ 消耗 50 Mana（剩余 20）
→ 造成 30 伤害（敌人剩余 5）
→ Fireball 进入 3 回合冷却
→ 冷却状态：Fireball=3, Heal=0, Ultimate=0
→ 资源状态：Mana=20

[回合 4] 敌人使用 BasicAttack
→ 造成 10 伤害（玩家剩余 70）

[回合 4] 回合结束
→ 冷却递减完成
→ Fireball CD 3 → 2

[回合 5] 玩家使用 Ultimate
→ Mana 不足：需要 100，当前 20
→ 结果：失败（Mana 不足）

[回合 5] 玩家使用 BasicAttack
→ 造成 15 伤害（敌人剩余 -10）

[回合 5] 敌人血量检查
=== 战斗胜利 ===

=== 战斗模拟完成 ===
```

---

### 10.3 UI 集成示例

#### 10.3.1 技能按钮 UI

```csharp
// AbilityButtonUI.cs
using UnityEngine;
using UnityEngine.UI;

public class AbilityButtonUI : MonoBehaviour
{
    [Header("引用")]
    public AbilitySpec Ability;
    public Button Button;
    public Image Icon;
    public Image CooldownMask;
    public Text CooldownText;
    public Text ManaCostText;
    
    [Header("配置")]
    public int MaxCooldownTurns = 5; // 用于计算 fillAmount
    
    private TurnBasedAbilityCooldownManager cooldownMgr;
    private AbilitySystemComponent owner;
    
    void Start()
    {
        cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
        owner = Ability.Owner;
        
        // 绑定点击事件
        Button.onClick.AddListener(OnButtonClick);
    }
    
    void Update()
    {
        UpdateCooldownDisplay();
        UpdateButtonState();
    }
    
    void UpdateCooldownDisplay()
    {
        if (cooldownMgr.IsOnCooldown(Ability))
        {
            int remaining = cooldownMgr.GetRemainingCooldown(Ability);
            CooldownText.text = remaining.ToString();
            CooldownMask.fillAmount = remaining / (float)MaxCooldownTurns;
            CooldownMask.gameObject.SetActive(true);
        }
        else
        {
            CooldownText.text = "";
            CooldownMask.fillAmount = 0;
            CooldownMask.gameObject.SetActive(false);
        }
    }
    
    void UpdateButtonState()
    {
        // 检查技能是否可用
        bool canActivate = Ability.CanActivate() == AbilityActivateResult.Success;
        
        Button.interactable = canActivate;
        
        // 根据状态改变颜色
        Icon.color = canActivate ? Color.white : Color.gray;
    }
    
    void OnButtonClick()
    {
        // 通知 BattleUIController 选择了这个技能
        BattleUIController.Instance.OnAbilitySelected(Ability);
    }
}
```

---

#### 10.3.2 技能栏 UI

```csharp
// AbilityBarUI.cs
using UnityEngine;
using System.Collections.Generic;

public class AbilityBarUI : MonoBehaviour
{
    [Header("引用")]
    public AbilityButtonUI BasicAttackButton;
    public AbilityButtonUI FireballButton;
    public AbilityButtonUI HealButton;
    public AbilityButtonUI UltimateButton;
    
    private List<AbilityButtonUI> allButtons;
    
    void Start()
    {
        allButtons = new List<AbilityButtonUI>
        {
            BasicAttackButton,
            FireballButton,
            HealButton,
            UltimateButton
        };
    }
    
    public void RefreshCooldowns()
    {
        // UI 自动通过 Update() 刷新，这里可以用于手动强制刷新
        foreach (var button in allButtons)
        {
            button.UpdateCooldownDisplay();
            button.UpdateButtonState();
        }
    }
    
    public void DisableAllButtons()
    {
        foreach (var button in allButtons)
        {
            button.Button.interactable = false;
        }
    }
    
    public void EnableAllButtons()
    {
        foreach (var button in allButtons)
        {
            button.UpdateButtonState();
        }
    }
}
```

---

### 10.4 调试技巧

#### 10.4.1 Runtime Watcher 集成

```csharp
// 在 Runtime Watcher 中显示冷却状态
public class CooldownDebugger : MonoBehaviour
{
    private TurnBasedAbilityCooldownManager cooldownMgr;
    
    void OnGUI()
    {
        if (cooldownMgr == null)
        {
            cooldownMgr = TurnBasedBattleManager.Instance.CooldownManager;
            return;
        }
        
        GUILayout.BeginArea(new Rect(10, 10, 300, 400));
        GUILayout.Label("=== 冷却状态 ===");
        
        // 显示所有冷却中的技能
        foreach (var (ability, cooldown) in cooldownMgr.Cooldowns)
        {
            GUILayout.Label($"{ability.Ability.Name}: {cooldown} 回合");
        }
        
        GUILayout.Label("\n=== 使用次数 ===");
        
        // 显示每回合使用次数
        foreach (var (abilityName, count) in cooldownMgr.TurnUsageCounts)
        {
            GUILayout.Label($"{abilityName}: {count} 次");
        }
        
        GUILayout.EndArea();
    }
}
```

---

#### 10.4.2 日志输出优化

```csharp
// 在 TurnBasedAbilityCooldownManager 中添加调试日志
public void StartCooldown(AbilitySpec ability, int cooldownTurns)
{
    if (cooldownTurns <= 0) return;
    
    cooldowns[ability] = cooldownTurns;
    
    #if UNITY_EDITOR
    Debug.Log($"[Cooldown] {ability.Ability.Name} 进入冷却：{cooldownTurns} 回合");
    #endif
}

public void DecrementCooldowns(AbilitySystemComponent owner)
{
    var abilitiesToUpdate = cooldowns.Keys
        .Where(a => a.Owner == owner)
        .ToList();
    
    foreach (var ability in abilitiesToUpdate)
    {
        cooldowns[ability]--;
        
        #if UNITY_EDITOR
        Debug.Log($"[Cooldown] {ability.Ability.Name} 冷却递减：{cooldowns[ability] + 1} → {cooldowns[ability]}");
        #endif
        
        if (cooldowns[ability] <= 0)
        {
            cooldowns.Remove(ability);
            
            #if UNITY_EDITOR
            Debug.Log($"[Cooldown] {ability.Ability.Name} 冷却结束，可以使用");
            #endif
        }
    }
}
```

---

### 10.5 快速参考

#### 10.5.1 API 速查表

| API | 用途 | 示例 |
|-----|------|------|
| `StartCooldown(ability, turns)` | 启动冷却 | `cooldownMgr.StartCooldown(this, 3)` |
| `IsOnCooldown(ability)` | 检查冷却 | `if (cooldownMgr.IsOnCooldown(this))` |
| `GetRemainingCooldown(ability)` | 查询剩余冷却 | `int cd = cooldownMgr.GetRemainingCooldown(this)` |
| `CanUseThisTurn(name, max)` | 检查每回合次数 | `cooldownMgr.CanUseThisTurn("Ultimate", 1)` |
| `CanUseBattle(ability, max)` | 检查每战斗次数 | `cooldownMgr.CanUseBattle(this, 3)` |
| `RecordUsage(name)` | 记录使用次数 | `cooldownMgr.RecordUsage(Ability.Name)` |
| `RecordBattleUsage(ability)` | 记录战斗使用 | `cooldownMgr.RecordBattleUsage(this)` |
| `DecrementCooldowns(owner)` | 递减冷却 | `cooldownMgr.DecrementCooldowns(player.ASC)` |
| `ResetTurnUsage()` | 重置每回合次数 | `cooldownMgr.ResetTurnUsage()` |
| `ReduceCooldown(ability, turns)` | 减少冷却 | `cooldownMgr.ReduceCooldown(this, 2)` |
| `ResetCooldown(ability)` | 重置冷却 | `cooldownMgr.ResetCooldown(this)` |
| `ReduceAllCooldowns(owner, turns)` | 减少所有冷却 | `cooldownMgr.ReduceAllCooldowns(player.ASC, 2)` |
| `ResetAllCooldowns(owner)` | 重置所有冷却 | `cooldownMgr.ResetAllCooldowns(player.ASC)` |
| `StartSharedCooldown(group, turns)` | 启动共享冷却 | `cooldownMgr.StartSharedCooldown("Fireball", 2)` |
| `IsSharedCooldownActive(group)` | 检查共享冷却 | `if (cooldownMgr.IsSharedCooldownActive("Fireball"))` |

---

#### 10.5.2 常见配置模板

**模板 1：普通攻击（无冷却）**
```csharp
public class BasicAttackAbilityAsset : AbilityAsset
{
    public int CooldownTurns = 0;  // 无冷却
    public int MaxUsesPerTurn = 0; // 无限制
}
```

**模板 2：常规技能（CD 3 回合 + Mana 消耗）**
```csharp
public class FireballAbilityAsset : AbilityAsset
{
    [Header("冷却")]
    public int CooldownTurns = 3;
    
    [Header("消耗")]
    public GameplayEffect ManaCostEffect; // ManaCost_50.asset
}
```

**模板 3：大招（CD 5 回合 + 每回合限 1 次 + 高消耗）**
```csharp
public class UltimateAbilityAsset : AbilityAsset
{
    [Header("冷却")]
    public int CooldownTurns = 5;
    public int MaxUsesPerTurn = 1;
    
    [Header("消耗")]
    public GameplayEffect ManaCostEffect; // ManaCost_100.asset
}
```

**模板 4：终极技能（超长 CD + 每战斗限 3 次）**
```csharp
public class LimitBreakAbilityAsset : AbilityAsset
{
    [Header("冷却")]
    public int CooldownTurns = 8;
    public int MaxUsesPerBattle = 3;
    
    [Header("消耗")]
    public int RageCost = 100;
    public GameplayEffect RageCostEffect; // RageCost_100.asset
}
```

**模板 5：共享冷却技能**
```csharp
public class SharedCooldownAbilityAsset : AbilityAsset
{
    [Header("共享冷却")]
    public string SharedCooldownGroup = "Fireball";
    public int SharedCooldownTurns = 2;
}
```

---

### 10.6 实现总结

**核心文件**（需要创建/修改）：
1. ✅ `TurnBasedAbilityCooldownManager.cs`（~300 行）
2. ✅ `TurnBasedBattleManager.cs`（集成 CooldownManager）
3. ✅ `AbilityAsset`（添加冷却配置字段）
4. ✅ `AbilitySpec`（集成冷却检查和记录）
5. ✅ `Cost GE`（资源消耗 ScriptableObject）
6. ✅ `AbilityButtonUI.cs`（UI 显示）

**工作量估算**：
- **Day 1**：基础冷却系统（4-6 小时）
- **Day 2**：次数限制 + Cost GE（4-6 小时）
- **Day 3**：特殊机制 + UI（4-6 小时）
- **总计**：12-18 小时（1-2 个工作日）

**测试覆盖**：
- ✅ 单元测试：Manager 核心功能
- ✅ 集成测试：与 Ability 协作
- ✅ UI 测试：冷却显示
- ✅ 完整战斗测试：5 回合流程

**扩展性**：
- ✅ 网络同步：导出/导入冷却数据
- ✅ 存档系统：序列化冷却状态
- ✅ UI 集成：显示冷却、次数、资源
- ✅ 技能书系统：动态学习技能

---

## 十一、总结与最佳实践

### 11.1 核心改造内容

**问题**：EX-GAS Ability 系统没有内置冷却机制

**解决方案**：TurnBasedAbilityCooldownManager

**三层冷却机制**：
1. **回合冷却**：技能使用后进入 N 回合冷却
2. **每回合次数限制**：技能每回合最多使用 N 次
3. **每战斗次数限制**：技能每战斗最多使用 N 次

**核心 API**：
- 检查类：`IsOnCooldown`, `CanUseThisTurn`, `CanUseBattle`
- 记录类：`StartCooldown`, `RecordUsage`, `RecordBattleUsage`
- 特殊类：`ReduceCooldown`, `ResetCooldown`, `StartSharedCooldown`

---

### 11.2 最佳实践

**DO（推荐做法）**：
- ✅ 使用 TurnBasedAbilityCooldownManager 集中管理冷却
- ✅ 资源消耗用 Cost GE，符合 GAS 理念
- ✅ 在 CanActivate() 中检查所有条件
- ✅ 在 ActivateAbility() 中记录冷却和次数
- ✅ 回合结束时统一递减冷却
- ✅ 回合开始时重置每回合次数
- ✅ 使用常量或 GameplayTag 代替共享冷却字符串

**DON'T（不推荐做法）**：
- ❌ 不要在 AbilitySpec 中存储冷却数据
- ❌ 不要直接修改属性消耗资源
- ❌ 不要用 GE 模拟冷却（除非改造 Duration 系统）
- ❌ 不要在多个地方递减冷却（统一在 BattleManager）
- ❌ 不要忘记检查冷却后再使用技能

---

### 11.3 扩展方向

**已实现**：
- ✅ 回合冷却、次数限制、资源消耗
- ✅ 减 CD、重置 CD、共享 CD、全局 CD
- ✅ UI 集成、调试工具

**可扩展**：
- 🔧 网络同步（导出/导入冷却数据）
- 🔧 存档系统（序列化冷却状态）
- 🔧 技能书系统（动态学习技能）
- 🔧 天赋系统（永久减 CD）
- 🔧 消耗减免 Buff（通过 MMC）
- 🔧 击杀重置 CD（通过 GameplayCue）

---

### 11.4 结语

本文档提供了 **EX-GAS 回合制技能冷却系统** 的完整设计方案：

- **第一至三章**：基础理念、核心需求、Manager 架构
- **第四章**：三层冷却机制详细设计
- **第五章**：资源消耗系统（Cost GE）
- **第六章**：减 CD 与特殊机制
- **第七章**：与 AbilitySpec 集成
- **第八章**：优缺点、扩展性、测试
- **第九章**：三种方案对比
- **第十章**：实现步骤与完整示例

**文档特点**：
- 📖 理论与实践结合
- 💻 完整可运行代码
- 🎯 覆盖所有使用场景
- 🔧 详细的实现步骤
- ✅ 完善的测试方案

**适用场景**：
- ✅ 回合制 RPG
- ✅ 回合制策略游戏
- ✅ 卡牌游戏
- ✅ 回合制战棋

**推荐阅读顺序**：
1. **快速上手**：直接看第十章完整示例
2. **深入理解**：从第一章开始完整阅读
3. **参考实现**：查阅第三至七章具体代码
4. **方案对比**：阅读第九章选择最适合方案

---

**完。**


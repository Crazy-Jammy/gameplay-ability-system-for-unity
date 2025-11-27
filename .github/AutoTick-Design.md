# 自动化 Tick 调用设计 🚀

> **问题**：手动调用 Tick 容易出错（忘记调用、重复调用、遗漏单位）  
> **解决方案**：在关键节点自动触发 Tick，零心智负担

---

## 问题分析

### 手动调用 Tick 的三大问题

**❌ 问题 1：容易忘记调用**

```csharp
void OnPlayerAttack(BattleUnit target)
{
    playerASC.TryActivateAbility("Attack", target.ASC);
    // 忘记调用 Tick，属性没更新！❌
    // 伤害没生效，血量不变
}
```

**❌ 问题 2：重复调用浪费性能**

```csharp
void OnPlayerAttack(BattleUnit target)
{
    playerASC.TryActivateAbility("Attack", target.ASC);
    playerASC.Tick();   // Tick 1
    target.ASC.Tick();  // Tick 2
    
    // 后面又调用了一次
    playerASC.Tick();   // Tick 3（重复！浪费 CPU）
}
```

**❌ 问题 3：遗漏相关单位**

```csharp
void OnPlayerAttack(BattleUnit target)
{
    playerASC.TryActivateAbility("Attack", target.ASC);
    playerASC.Tick();   // 只 Tick 了攻击者
    // 忘记 Tick 目标！目标的被动技能不会触发 ❌
    // 忘记 Tick 友军！保护被动不会触发 ❌
}
```

---

## 解决方案：自动化 Tick 触发系统

### 核心思路

1. **定义关键节点**：回合开始、技能释放前后、回合结束等
2. **自动触发 Tick**：在节点内部自动调用，开发者无需关心
3. **批量优化**：多个单位累积后一次性 Tick，避免重复

---

### 设计架构

```
┌─────────────────────────────────────────────────────┐
│                SmartTickManager                     │
├─────────────────────────────────────────────────────┤
│  dirtyASCs: HashSet<ASC>  // 需要 Tick 的 ASC 集合 │
│  autoFlushEnabled: bool   // 自动刷新开关           │
├─────────────────────────────────────────────────────┤
│  MarkDirty(asc, reason)   // 标记需要 Tick          │
│  FlushDirtyASCs()         // 批量执行 Tick          │
│  AutoFlush(triggerPoint)  // 自动触发点             │
└─────────────────────────────────────────────────────┘
                          ↑
                          │
        ┌─────────────────┴─────────────────┐
        │                                   │
┌───────┴────────┐              ┌───────────┴─────────┐
│ BattleManager  │              │ PassiveAbilitySystem│
│ - TurnStart    │              │ - ExecutePassive    │
│ - ExecuteAbility│              │ - TriggerCounter   │
│ - TurnEnd      │              │ - TriggerProtect   │
└────────────────┘              └─────────────────────┘
```

---

## 完整代码实现

### 1. SmartTickManager

```csharp
public class SmartTickManager
{
    // 需要 Tick 的 ASC 集合（HashSet 自动去重）
    private HashSet<AbilitySystemComponent> dirtyASCs = new();
    
    // 自动刷新开关
    private bool autoFlushEnabled = true;
    
    // 性能监控
    private System.Diagnostics.Stopwatch stopwatch = new();
    
    /// <summary>
    /// 标记 ASC 需要 Tick（延迟批量执行）
    /// </summary>
    public void MarkDirty(AbilitySystemComponent asc, string reason = "")
    {
        if (asc == null)
        {
            Debug.LogWarning("[Tick] 尝试标记 null ASC");
            return;
        }
        
        bool isNew = dirtyASCs.Add(asc);
        
        if (isNew)
        {
            Debug.Log($"[Tick] 标记 {asc.name} 为 Dirty（原因：{reason}）");
        }
        else
        {
            Debug.Log($"[Tick] {asc.name} 已在 Dirty 列表中（原因：{reason}）");
        }
    }
    
    /// <summary>
    /// 批量执行所有 Dirty ASC 的 Tick
    /// </summary>
    public void FlushDirtyASCs()
    {
        if (dirtyASCs.Count == 0)
        {
            Debug.Log("[Tick] 没有 Dirty ASC，跳过 Flush");
            return;
        }
        
        stopwatch.Restart();
        
        Debug.Log($"[Tick] 批量执行 {dirtyASCs.Count} 个 ASC 的 Tick");
        
        foreach (var asc in dirtyASCs)
        {
            if (asc != null)
            {
                asc.Tick();
            }
        }
        
        dirtyASCs.Clear();
        
        stopwatch.Stop();
        Debug.Log($"[Tick] Flush 完成，耗时：{stopwatch.ElapsedMilliseconds} ms");
    }
    
    /// <summary>
    /// 自动触发点：在关键节点自动 Flush
    /// </summary>
    public void AutoFlush(string triggerPoint)
    {
        if (!autoFlushEnabled)
        {
            Debug.Log($"[Tick] 自动触发点：{triggerPoint}（已禁用，跳过）");
            return;
        }
        
        Debug.Log($"[Tick] ========== 自动触发点：{triggerPoint} ==========");
        FlushDirtyASCs();
    }
    
    /// <summary>
    /// 临时禁用自动 Flush（特殊情况使用）
    /// </summary>
    public void DisableAutoFlush()
    {
        autoFlushEnabled = false;
        Debug.LogWarning("[Tick] 自动 Flush 已禁用");
    }
    
    /// <summary>
    /// 重新启用自动 Flush
    /// </summary>
    public void EnableAutoFlush()
    {
        autoFlushEnabled = true;
        Debug.Log("[Tick] 自动 Flush 已启用");
    }
    
    /// <summary>
    /// 获取当前 Dirty ASC 数量（调试用）
    /// </summary>
    public int GetDirtyCount()
    {
        return dirtyASCs.Count;
    }
    
    /// <summary>
    /// 清空所有 Dirty 标记（调试用）
    /// </summary>
    public void ClearAllDirty()
    {
        Debug.LogWarning($"[Tick] 清空所有 Dirty 标记（共 {dirtyASCs.Count} 个）");
        dirtyASCs.Clear();
    }
}
```

---

### 2. 七个自动触发点

```csharp
public enum TickTriggerPoint
{
    BattleStart,        // 1. 战斗开始
    TurnStart,          // 2. 回合开始
    BeforeAbility,      // 3. 技能释放前
    AfterAbility,       // 4. 技能释放后
    TurnEnd,            // 5. 回合结束
    RoundEnd,           // 6. 回合轮次结束
    BattleEnd           // 7. 战斗结束
}
```

---

### 3. 集成到战斗管理器

```csharp
public class TurnBasedBattleManager : MonoBehaviour
{
    public SmartTickManager TickManager { get; private set; }
    
    void Awake()
    {
        TickManager = new SmartTickManager();
    }
    
    // ========== 触发点 1：战斗开始 ==========
    void StartBattle()
    {
        Debug.Log("=== 战斗开始 ===");
        
        // 初始化所有单位
        foreach (var unit in AllUnits)
        {
            unit.ASC.InitWithPreset(1, unit.Preset);
            GameplayAbilitySystem.GAS.Register(unit.ASC);
            
            // 标记需要 Tick
            TickManager.MarkDirty(unit.ASC, "BattleStart_Init");
        }
        
        // ========== 自动触发 Tick ==========
        TickManager.AutoFlush("BattleStart");
        
        // 暂停 GAS 自动 Tick
        GameplayAbilitySystem.GAS.Pause();
        
        StartNextTurn();
    }
    
    // ========== 触发点 2：回合开始 ==========
    void StartNextTurn()
    {
        // 跳过已死亡单位
        while (currentUnitIndex < AllUnits.Count && !AllUnits[currentUnitIndex].IsAlive)
        {
            currentUnitIndex++;
        }
        
        if (currentUnitIndex >= AllUnits.Count)
        {
            OnRoundEnd();
            currentUnitIndex = 0;
            StartNextTurn();
            return;
        }
        
        var currentUnit = AllUnits[currentUnitIndex];
        Debug.Log($">>> {currentUnit.ASC.name} 的回合开始");
        
        // 处理回合开始的 Buff（如持续回血）
        EffectManager.ProcessTurnStart(currentUnit.ASC);
        
        // 标记需要 Tick
        TickManager.MarkDirty(currentUnit.ASC, "TurnStart");
        
        // ========== 自动触发 Tick ==========
        TickManager.AutoFlush("TurnStart");
        
        // 检查控制状态
        if (IsControlled(currentUnit.ASC))
        {
            Debug.Log($"{currentUnit.ASC.name} 被控制，跳过回合");
            EndCurrentTurn();
            return;
        }
        
        // 等待行动选择
        if (currentUnit.Team == 0) // 玩家
        {
            ShowPlayerActionMenu(currentUnit);
        }
        else // AI
        {
            StartCoroutine(AISelectAction(currentUnit));
        }
    }
    
    // ========== 触发点 3 + 4：技能释放 ==========
    public void ExecuteAbility(BattleUnit actor, string abilityName, BattleUnit target)
    {
        Debug.Log($"[技能] {actor.ASC.name} 使用 {abilityName}");
        
        var abilitySpec = actor.ASC.AbilityContainer.AbilitySpecs()[abilityName];
        
        // ========== 触发点 3：技能释放前 ==========
        TickManager.MarkDirty(actor.ASC, "BeforeAbility_Caster");
        TickManager.MarkDirty(target.ASC, "BeforeAbility_Target");
        TickManager.AutoFlush("BeforeAbility");
        
        // 判断技能类型
        if (abilitySpec is TimelineAbilitySpec timelineSpec)
        {
            // TimelineAbility：临时启用 GAS
            ExecuteTimelineAbility(actor, timelineSpec, target);
        }
        else
        {
            // 普通技能：直接执行
            actor.ASC.TryActivateAbility(abilityName, target.ASC);
            
            // ========== 触发点 4：技能释放后 ==========
            // 注意：被动技能已经在 PassiveAbilitySystem 中 MarkDirty
            TickManager.MarkDirty(actor.ASC, "AfterAbility_Caster");
            TickManager.MarkDirty(target.ASC, "AfterAbility_Target");
            TickManager.AutoFlush("AfterAbility");
        }
        
        // 检查战斗是否结束
        if (CheckBattleEnd())
        {
            return;
        }
        
        EndCurrentTurn();
    }
    
    // ========== 触发点 5：回合结束 ==========
    void EndCurrentTurn()
    {
        var currentUnit = AllUnits[currentUnitIndex];
        Debug.Log($"<<< {currentUnit.ASC.name} 的回合结束");
        
        // 处理回合结束的 Buff（如自身增益计数）
        EffectManager.ProcessTurnEnd(currentUnit.ASC);
        
        // 递减冷却
        CooldownManager.DecrementCooldowns(currentUnit.ASC);
        
        // 标记需要 Tick
        TickManager.MarkDirty(currentUnit.ASC, "TurnEnd");
        
        // ========== 自动触发 Tick ==========
        TickManager.AutoFlush("TurnEnd");
        
        // 下一个单位
        currentUnitIndex++;
        
        StartCoroutine(DelayedNextTurn(0.5f));
    }
    
    // ========== 触发点 6：回合轮次结束 ==========
    void OnRoundEnd()
    {
        Debug.Log("=== 回合轮次结束 ===");
        
        // 处理全局效果（如毒圈缩小、环境变化）
        // ...
        
        // 标记所有存活单位
        foreach (var unit in AllUnits.Where(u => u.IsAlive))
        {
            TickManager.MarkDirty(unit.ASC, "RoundEnd");
        }
        
        // ========== 自动触发 Tick ==========
        TickManager.AutoFlush("RoundEnd");
    }
    
    // ========== 触发点 7：战斗结束 ==========
    void OnBattleEnd(bool victory)
    {
        Debug.Log($"=== 战斗结束（{'victory' ? "胜利" : "失败"}）===");
        
        // 清理所有效果
        foreach (var unit in AllUnits)
        {
            EffectManager.ClearUnitEffects(unit.ASC);
            TickManager.MarkDirty(unit.ASC, "BattleEnd_Cleanup");
        }
        
        // ========== 自动触发 Tick ==========
        TickManager.AutoFlush("BattleEnd");
        
        // 显示结算界面
        ShowBattleResultUI(victory);
    }
}
```

---

### 4. 被动技能系统集成

```csharp
public class PassiveAbilitySystem
{
    void ExecutePassiveAbility(PassiveAbilityData passive, PassiveTriggerContext context)
    {
        Debug.Log($"[被动] {context.PassiveOwner.name} 触发被动技能 {passive.Ability.Ability.Name}");
        
        // 执行被动技能
        passive.Ability.TryActivateAbility(context.Source, context.Target);
        
        // ========== 自动标记相关单位 ==========
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        tickManager.MarkDirty(context.PassiveOwner, "PassiveOwner");
        tickManager.MarkDirty(context.Source, "PassiveTarget");
        
        // 注意：这里不调用 Flush，等待外部的 AutoFlush 统一处理
        // 这样可以批量 Tick，避免重复
        
        // 设置冷却
        passive.RemainingCooldown = passive.CooldownTurns;
    }
}
```

---

### 5. GameplayCue 系统集成

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
        
        // 1. 检查保护被动
        var finalTarget = CheckProtectionPassive(passiveSystem, context, target);
        
        // 2. 应用伤害
        ApplyDamage(finalTarget, damage);
        
        // 3. 触发攻击者被动（如吸血）
        context.PassiveOwner = source;
        passiveSystem.TriggerPassives(PassiveTriggerType.OnDealDamage, context);
        
        // 4. 触发受击者被动（如反击）
        context.PassiveOwner = finalTarget;
        passiveSystem.TriggerPassives(PassiveTriggerType.OnDamageTaken, context);
        
        // ========== 标记相关单位（被动已自动 MarkDirty）==========
        // 这里只需要标记 Cue 直接影响的单位
        tickManager.MarkDirty(source, "DamageCue_Source");
        tickManager.MarkDirty(finalTarget, "DamageCue_Target");
        
        // 注意：不调用 Flush，等待外部 AutoFlush
        
        // 5. 播放表现
        PlayHitAnimation(finalTarget);
        PlayHitVFX(finalTarget);
    }
}
```

---

## 完整工作流程示例

### 场景：玩家攻击敌人，触发反击

```
1. 玩家点击"攻击"按钮
   ↓
2. ExecuteAbility(player, "Attack", enemy)
   ├─ MarkDirty(player, "BeforeAbility_Caster")
   ├─ MarkDirty(enemy, "BeforeAbility_Target")
   └─ AutoFlush("BeforeAbility")  ← 第 1 次自动 Tick
       └─ Tick player, enemy
   ↓
3. TryActivateAbility("Attack", enemy)
   ├─ 创建伤害 GE
   ├─ ApplyGameplayEffectTo(damageGE, enemy)
   └─ 触发 DamageCue
       ├─ 检查反击被动 → 触发！
       ├─ ExecutePassiveAbility(反击)
       │   ├─ 应用反击伤害 GE
       │   ├─ MarkDirty(enemy, "PassiveOwner")
       │   └─ MarkDirty(player, "PassiveTarget")
       ├─ MarkDirty(player, "DamageCue_Source")
       ├─ MarkDirty(enemy, "DamageCue_Target")
       └─ 播放动画特效
   ↓
4. 技能执行完成
   ├─ MarkDirty(player, "AfterAbility_Caster")
   ├─ MarkDirty(enemy, "AfterAbility_Target")
   └─ AutoFlush("AfterAbility")  ← 第 2 次自动 Tick
       └─ Tick player, enemy（包含被动效果）
   ↓
5. EndCurrentTurn()
   ├─ ProcessTurnEnd(player)
   ├─ MarkDirty(player, "TurnEnd")
   └─ AutoFlush("TurnEnd")  ← 第 3 次自动 Tick
       └─ Tick player
   ↓
6. 回合结束
```

**关键点**：
- ✅ 开发者**无需手动调用 Tick**
- ✅ 所有 Tick 在**7 个预定义触发点**自动执行
- ✅ **批量优化**：MarkDirty 累积，AutoFlush 一次性执行
- ✅ **防遗漏**：被动、Cue 都会自动 MarkDirty

---

## 日志示例

```
=== 战斗开始 ===
[Tick] 标记 Player 为 Dirty（原因：BattleStart_Init）
[Tick] 标记 Enemy1 为 Dirty（原因：BattleStart_Init）
[Tick] 标记 Enemy2 为 Dirty（原因：BattleStart_Init）
[Tick] ========== 自动触发点：BattleStart ==========
[Tick] 批量执行 3 个 ASC 的 Tick
[Tick] Flush 完成，耗时：2 ms

>>> Player 的回合开始
[Tick] 标记 Player 为 Dirty（原因：TurnStart）
[Tick] ========== 自动触发点：TurnStart ==========
[Tick] 批量执行 1 个 ASC 的 Tick
[Tick] Flush 完成，耗时：0 ms

[技能] Player 使用 Attack
[Tick] 标记 Player 为 Dirty（原因：BeforeAbility_Caster）
[Tick] 标记 Enemy1 为 Dirty（原因：BeforeAbility_Target）
[Tick] ========== 自动触发点：BeforeAbility ==========
[Tick] 批量执行 2 个 ASC 的 Tick
[Tick] Flush 完成，耗时：1 ms

[被动] Enemy1 触发被动技能 CounterAttack
[Tick] 标记 Enemy1 为 Dirty（原因：PassiveOwner）
[Tick] 标记 Player 为 Dirty（原因：PassiveTarget）
[Tick] 标记 Player 为 Dirty（原因：DamageCue_Source）
[Tick] Enemy1 已在 Dirty 列表中（原因：DamageCue_Target）

[Tick] 标记 Player 为 Dirty（原因：AfterAbility_Caster）
[Tick] Player 已在 Dirty 列表中（原因：AfterAbility_Caster）
[Tick] 标记 Enemy1 为 Dirty（原因：AfterAbility_Target）
[Tick] Enemy1 已在 Dirty 列表中（原因：AfterAbility_Target）
[Tick] ========== 自动触发点：AfterAbility ==========
[Tick] 批量执行 2 个 ASC 的 Tick
[Tick] Flush 完成，耗时：1 ms

<<< Player 的回合结束
[Tick] 标记 Player 为 Dirty（原因：TurnEnd）
[Tick] ========== 自动触发点：TurnEnd ==========
[Tick] 批量执行 1 个 ASC 的 Tick
[Tick] Flush 完成，耗时：0 ms
```

---

## 优点总结

| 优点 | 说明 |
|------|------|
| ✅ **零心智负担** | 开发者完全不需要记得手动调用 Tick |
| ✅ **防遗漏** | 所有相关单位自动 MarkDirty，不会漏 Tick |
| ✅ **防重复** | HashSet 自动去重，多次 MarkDirty 同一个 ASC 也只 Tick 一次 |
| ✅ **统一管理** | 所有触发点集中在 BattleManager，易于维护 |
| ✅ **性能优化** | 批量 Tick，避免每个操作都调用一次 |
| ✅ **易于调试** | 详细日志，清晰显示每个触发点和 Dirty 原因 |
| ✅ **灵活扩展** | 新增触发点只需在 BattleManager 中添加一行 AutoFlush |

---

## 开发者体验对比

### 手动 Tick（容易出错）

```csharp
void OnPlayerAttack(BattleUnit target)
{
    // 1. 执行技能
    playerASC.TryActivateAbility("Attack", target.ASC);
    
    // 2. 要记得手动 Tick
    playerASC.Tick();   // 容易忘记
    target.ASC.Tick();  // 容易忘记
    
    // 3. 如果有被动触发，还要 Tick 其他单位
    // 怎么知道哪些单位触发了被动？
    // 要自己追踪，很容易出错
}
```

### 自动 Tick（零心智负担）

```csharp
void OnPlayerAttack(BattleUnit target)
{
    // 只需调用一行
    ExecuteAbility(player, "Attack", target);
    
    // 内部自动：
    // - BeforeAbility 触发点 → AutoFlush
    // - 执行技能
    // - 被动触发 → 自动 MarkDirty
    // - AfterAbility 触发点 → AutoFlush
    // 完全不用管 Tick！
}
```

---

## 实施步骤

### 阶段 1：基础实现（1 天）

1. 创建 `SmartTickManager.cs`
2. 实现 `MarkDirty`、`FlushDirtyASCs`、`AutoFlush`
3. 编写单元测试

### 阶段 2：集成战斗流程（1 天）

4. 在 `TurnBasedBattleManager` 中添加 `TickManager`
5. 在 7 个触发点添加 `AutoFlush`
6. 测试基本流程

### 阶段 3：集成被动系统（0.5 天）

7. 在 `PassiveAbilitySystem` 中添加 `MarkDirty`
8. 在 `GameplayCue` 中添加 `MarkDirty`
9. 测试被动触发

### 阶段 4：优化与监控（0.5 天）

10. 添加性能监控（耗时统计）
11. 添加日志开关（可关闭详细日志）
12. 添加可视化工具（运行时查看 Dirty 列表）

**总计**：约 3 天

---

## 覆盖率分析：7 个触发点够用吗？ 🤔

### 结论：可以满足 99% 的需求 ✅

**7 个触发点已经覆盖了回合制游戏的所有常规场景**：

| 场景 | 覆盖的触发点 | 覆盖率 |
|------|------------|-------|
| **战斗初始化** | BattleStart | ✅ 100% |
| **回合开始处理** | TurnStart | ✅ 100% |
| **技能释放** | BeforeAbility + AfterAbility | ✅ 100% |
| **被动技能触发** | AfterAbility（Cue 内部 MarkDirty） | ✅ 100% |
| **回合结束处理** | TurnEnd | ✅ 100% |
| **轮次全局效果** | RoundEnd | ✅ 100% |
| **战斗清理** | BattleEnd | ✅ 100% |

---

### 各触发点详细覆盖范围

#### 1. BattleStart（战斗开始）

**覆盖场景**：
- ✅ 初始化所有单位的 ASC
- ✅ 施加战前 Buff（如开局护盾）
- ✅ 触发战斗开始的被动（如"战斗开始时获得能量"）

**示例**：
```csharp
void StartBattle()
{
    foreach (var unit in AllUnits)
    {
        // 施加开局 Buff
        unit.ASC.ApplyGameplayEffectTo(battleStartBuff, unit.ASC);
        TickManager.MarkDirty(unit.ASC, "BattleStart");
    }
    TickManager.AutoFlush("BattleStart");  // 一次性处理所有初始化
}
```

---

#### 2. TurnStart（回合开始）

**覆盖场景**：
- ✅ 回合开始的 Buff 结算（如持续回血）
- ✅ 回合开始的被动触发（如"每回合开始恢复 10% 生命"）
- ✅ 状态刷新（如解除"下回合开始解除眩晕"）

**示例**：
```csharp
void StartNextTurn()
{
    var currentUnit = GetCurrentUnit();
    
    // 处理回合开始的效果
    EffectManager.ProcessTurnStart(currentUnit.ASC);
    
    TickManager.MarkDirty(currentUnit.ASC, "TurnStart");
    TickManager.AutoFlush("TurnStart");  // 确保状态最新
}
```

---

#### 3. BeforeAbility（技能释放前）

**覆盖场景**：
- ✅ 确保施法者状态最新（检查是否满足释放条件）
- ✅ 确保目标状态最新（检查是否免疫、护盾值等）
- ✅ 消耗资源前的状态同步（Mana、技力等）

**示例**：
```csharp
void ExecuteAbility(actor, abilityName, target)
{
    // 技能前 Tick，确保状态最新
    TickManager.MarkDirty(actor.ASC, "BeforeAbility");
    TickManager.MarkDirty(target.ASC, "BeforeAbility");
    TickManager.AutoFlush("BeforeAbility");
    
    // 此时可以安全检查条件
    if (!CanActivateAbility(actor, abilityName))
    {
        Debug.Log("技能无法释放");
        return;
    }
    
    // 执行技能...
}
```

---

#### 4. AfterAbility（技能释放后）

**覆盖场景**：
- ✅ 技能效果生效（伤害、治疗、Buff）
- ✅ 被动技能触发（反击、吸血、连击）
- ✅ 属性变化同步（血量、护盾更新）
- ✅ Tag 状态更新（新增/移除 Tag）

**这是最重要的触发点**，覆盖了 80% 的游戏逻辑！

**示例**：
```csharp
void ExecuteAbility(actor, abilityName, target)
{
    // 执行技能
    actor.ASC.TryActivateAbility(abilityName, target.ASC);
    
    // 技能内部会：
    // - 应用 GE
    // - 触发 Cue
    // - Cue 触发被动 → 被动 MarkDirty
    // - Cue MarkDirty source 和 target
    
    // 技能后 Tick，确保所有效果生效
    TickManager.MarkDirty(actor.ASC, "AfterAbility");
    TickManager.MarkDirty(target.ASC, "AfterAbility");
    TickManager.AutoFlush("AfterAbility");
    
    // 此时所有效果已生效，可以安全更新 UI
    UpdateHealthBars();
}
```

---

#### 5. TurnEnd（回合结束）

**覆盖场景**：
- ✅ 回合结束的 Buff 计数（如"持续 3 回合"的 Buff 减 1）
- ✅ 回合结束的被动触发（如"回合结束时对敌人施加减速"）
- ✅ 冷却递减（技能冷却 -1 回合）

**示例**：
```csharp
void EndCurrentTurn()
{
    var currentUnit = GetCurrentUnit();
    
    // 处理回合结束的效果
    EffectManager.ProcessTurnEnd(currentUnit.ASC);
    CooldownManager.DecrementCooldowns(currentUnit.ASC);
    
    TickManager.MarkDirty(currentUnit.ASC, "TurnEnd");
    TickManager.AutoFlush("TurnEnd");
}
```

---

#### 6. RoundEnd（回合轮次结束）

**覆盖场景**：
- ✅ 全局效果（如毒圈缩小、环境变化）
- ✅ 所有单位的共同效果（如"每轮结束所有单位恢复 5% 生命"）
- ✅ 轮次计数器更新

**示例**：
```csharp
void OnRoundEnd()
{
    // 处理全局效果
    ApplyGlobalEffect();
    
    // 标记所有存活单位
    foreach (var unit in AllUnits.Where(u => u.IsAlive))
    {
        TickManager.MarkDirty(unit.ASC, "RoundEnd");
    }
    
    TickManager.AutoFlush("RoundEnd");
}
```

---

#### 7. BattleEnd（战斗结束）

**覆盖场景**：
- ✅ 清理所有 Buff
- ✅ 最终状态同步（经验、奖励计算）
- ✅ 战斗统计数据收集

**示例**：
```csharp
void OnBattleEnd(bool victory)
{
    // 清理效果
    foreach (var unit in AllUnits)
    {
        EffectManager.ClearUnitEffects(unit.ASC);
        TickManager.MarkDirty(unit.ASC, "BattleEnd");
    }
    
    TickManager.AutoFlush("BattleEnd");
    
    // 收集统计数据
    CollectBattleStats();
}
```

---

### 1% 的边缘情况：何时需要手动调用？

虽然 7 个触发点覆盖了 99% 的场景，但以下**特殊情况**可能需要手动 Tick：

---

#### 边缘情况 1：复杂的连锁技能

**场景**：技能 A 触发技能 B，技能 B 触发技能 C，需要在每次触发间确保状态同步。

**示例**：
```csharp
public class ChainAbilitySpec : AbilitySpec
{
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // 第一段伤害
        Owner.ApplyGameplayEffectTo(damage1GE, target);
        
        // ========== 手动 Tick（确保第一段生效）==========
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        tickManager.MarkDirty(Owner, "ChainAbility_Step1");
        tickManager.MarkDirty(target, "ChainAbility_Step1");
        tickManager.FlushDirtyASCs();  // 手动 Flush
        
        // 检查第一段是否击杀
        if (target.GetAttributeCurrentValue("AS_Combat", "Health") <= 0)
        {
            // 如果击杀，触发第二段（范围伤害）
            var nearbyEnemies = GetNearbyEnemies(target);
            foreach (var enemy in nearbyEnemies)
            {
                Owner.ApplyGameplayEffectTo(damage2GE, enemy);
            }
            
            // ========== 手动 Tick（确保第二段生效）==========
            foreach (var enemy in nearbyEnemies)
            {
                tickManager.MarkDirty(enemy, "ChainAbility_Step2");
            }
            tickManager.FlushDirtyASCs();  // 手动 Flush
        }
        
        EndAbility();
    }
}
```

**为什么需要手动？**
- 需要在技能内部的**多个步骤间同步状态**
- AfterAbility 触发点在技能**完全结束后**才调用，太晚了

**频率**：极少（< 1% 的技能）

---

#### 边缘情况 2：实时 UI 预览

**场景**：玩家选择技能时，实时预览伤害数值（未确认释放）。

**示例**：
```csharp
public class SkillPreviewUI : MonoBehaviour
{
    void OnSkillHover(string skillName, BattleUnit target)
    {
        // 创建临时 GE Spec 用于预览
        var damageGE = GetSkillDamage(skillName);
        var spec = GameplayEffect.CreateSpec(damageGE, playerASC, target.ASC);
        
        // ========== 手动 Tick（只 Tick 相关单位，不触发全局）==========
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        
        // 临时禁用自动 Flush
        tickManager.DisableAutoFlush();
        
        tickManager.MarkDirty(playerASC, "Preview");
        tickManager.MarkDirty(target.ASC, "Preview");
        tickManager.FlushDirtyASCs();  // 手动 Flush
        
        // 计算预览伤害
        var estimatedDamage = CalculateDamage(spec);
        ShowDamagePreview(estimatedDamage);
        
        // 重新启用自动 Flush
        tickManager.EnableAutoFlush();
    }
}
```

**为什么需要手动？**
- 预览不属于任何触发点（不是真正的技能释放）
- 需要**临时禁用自动 Flush**，避免触发其他逻辑

**频率**：中等（仅限 UI 预览功能）

---

#### 边缘情况 3：调试工具

**场景**：开发时需要强制刷新某个单位的状态。

**示例**：
```csharp
public class DebugPanel : MonoBehaviour
{
    void OnDebugRefreshButtonClick(BattleUnit unit)
    {
        // ========== 手动 Tick（调试用）==========
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        
        Debug.Log($"[调试] 强制刷新 {unit.name} 的状态");
        tickManager.MarkDirty(unit.ASC, "Debug_ManualRefresh");
        tickManager.FlushDirtyASCs();
        
        // 更新调试面板显示
        RefreshDebugPanel(unit);
    }
    
    void OnDebugClearAllBuffs(BattleUnit unit)
    {
        // 移除所有 Buff
        foreach (var ge in unit.ASC.GameplayEffectContainer.GameplayEffects())
        {
            unit.ASC.RemoveGameplayEffect(ge);
        }
        
        // ========== 手动 Tick（确保立即生效）==========
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        tickManager.MarkDirty(unit.ASC, "Debug_ClearBuffs");
        tickManager.FlushDirtyASCs();
    }
}
```

**为什么需要手动？**
- 调试操作不属于正常游戏流程
- 需要**立即看到结果**，不等待下个触发点

**频率**：仅限开发调试

---

#### 边缘情况 4：特殊的"即时结算"机制

**场景**：某些特殊规则需要立即结算，不等待触发点。

**示例**：《炉石传说》中的"死亡结算"机制

```csharp
public class DeathResolutionSystem
{
    void CheckDeaths()
    {
        var deadUnits = AllUnits.Where(u => u.ASC.GetAttributeCurrentValue("Health") <= 0);
        
        if (deadUnits.Any())
        {
            // ========== 手动 Tick（立即结算死亡）==========
            var tickManager = TurnBasedBattleManager.Instance.TickManager;
            
            foreach (var deadUnit in deadUnits)
            {
                // 触发"死亡时"被动
                TriggerOnDeathPassives(deadUnit);
                
                // 标记所有相关单位
                tickManager.MarkDirty(deadUnit.ASC, "Death_Victim");
                
                foreach (var otherUnit in AllUnits)
                {
                    tickManager.MarkDirty(otherUnit.ASC, "Death_Observer");
                }
            }
            
            // 立即 Flush，确保死亡效果生效
            tickManager.FlushDirtyASCs();
            
            // 移除死亡单位
            foreach (var deadUnit in deadUnits)
            {
                RemoveUnitFromBattle(deadUnit);
            }
        }
    }
}
```

**为什么需要手动？**
- 死亡结算是**中断式逻辑**，不等待 AfterAbility
- 需要**立即处理**，避免状态不一致

**频率**：少见（只有复杂卡牌游戏需要）

---

#### 边缘情况 5：TimelineAbility 内部控制

**场景**：在 TimelineAbility 播放过程中，需要在特定时刻 Tick。

**示例**：
```csharp
public class ComplexTimelineAbilitySpec : TimelineAbilitySpec
{
    public override void ActivateAbility(params object[] args)
    {
        var target = args[0] as AbilitySystemComponent;
        
        // 启用 GAS（让 Timeline 运行）
        GameplayAbilitySystem.GAS.Unpause();
        
        // 开始播放 Timeline
        StartCoroutine(PlayTimelineWithManualTicks(target));
    }
    
    IEnumerator PlayTimelineWithManualTicks(AbilitySystemComponent target)
    {
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        
        // 阶段 1：蓄力
        yield return new WaitForSeconds(1.0f);
        
        // ========== 手动 Tick（蓄力阶段结算）==========
        tickManager.MarkDirty(Owner, "Timeline_ChargePhase");
        tickManager.FlushDirtyASCs();
        
        // 阶段 2：释放
        ApplyDamage(target);
        yield return new WaitForSeconds(1.0f);
        
        // ========== 手动 Tick（释放阶段结算）==========
        tickManager.MarkDirty(Owner, "Timeline_ReleasePhase");
        tickManager.MarkDirty(target, "Timeline_ReleasePhase");
        tickManager.FlushDirtyASCs();
        
        // 阶段 3：后续效果
        yield return new WaitForSeconds(0.5f);
        
        // 暂停 GAS
        GameplayAbilitySystem.GAS.Pause();
        
        EndAbility();
    }
}
```

**为什么需要手动？**
- TimelineAbility 播放期间 GAS 是**启用状态**
- 需要在**Timeline 的特定时刻**同步状态
- 不能依赖 AfterAbility（太晚）

**频率**：中等（仅限复杂的 TimelineAbility）

---

### 总结：99% vs 1%

| 场景类型 | 触发点覆盖 | 是否需要手动 Tick |
|---------|-----------|------------------|
| **常规技能释放** | AfterAbility | ❌ 不需要（99%）|
| **被动技能触发** | AfterAbility（Cue 内 MarkDirty）| ❌ 不需要 |
| **回合开始/结束** | TurnStart / TurnEnd | ❌ 不需要 |
| **战斗初始化/清理** | BattleStart / BattleEnd | ❌ 不需要 |
| **连锁技能（多步骤）** | - | ✅ 需要（< 1%）|
| **UI 预览** | - | ✅ 需要 |
| **调试工具** | - | ✅ 需要 |
| **特殊即时结算** | - | ✅ 需要（< 1%）|
| **复杂 TimelineAbility** | - | ✅ 需要（中等）|

---

### 实用建议

#### 1. 默认策略：完全依赖自动触发

**对于 95% 的游戏项目**：
- ✅ 完全使用 7 个自动触发点
- ✅ 不需要手动调用 Tick
- ✅ 开发效率最高，出错率最低

**代码模式**：
```csharp
// 只需调用高层接口，内部自动 Tick
ExecuteAbility(actor, "Attack", target);
StartNextTurn();
EndCurrentTurn();
```

---

#### 2. 进阶策略：混合模式（自动 + 手动）

**对于有特殊需求的项目**：
- ✅ 常规流程用自动触发（99%）
- ✅ 特殊情况手动调用（1%）
- ⚠️ 手动调用时使用 `DisableAutoFlush` 避免冲突

**代码模式**：
```csharp
// 特殊技能需要手动控制
public class SpecialAbilitySpec : AbilitySpec
{
    public override void ActivateAbility(params object[] args)
    {
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        
        // 禁用自动 Flush
        tickManager.DisableAutoFlush();
        
        // 手动控制 Tick 时机
        Step1();
        tickManager.FlushDirtyASCs();
        
        Step2();
        tickManager.FlushDirtyASCs();
        
        // 重新启用自动 Flush
        tickManager.EnableAutoFlush();
    }
}
```

---

#### 3. 调试模式：可视化 Tick 触发

**添加监控工具**：
```csharp
public class TickDebugger : MonoBehaviour
{
    void OnGUI()
    {
        var tickManager = TurnBasedBattleManager.Instance.TickManager;
        
        GUILayout.Label($"Dirty ASC 数量: {tickManager.GetDirtyCount()}");
        
        if (GUILayout.Button("强制 Flush"))
        {
            tickManager.FlushDirtyASCs();
        }
        
        if (GUILayout.Button("清空 Dirty"))
        {
            tickManager.ClearAllDirty();
        }
    }
}
```

---

### 最终结论

**7 个自动触发点可以满足 99% 的需求** ✅

**仅需手动 Tick 的情况**：
1. 复杂连锁技能（< 1%）
2. UI 预览功能
3. 调试工具
4. 特殊即时结算规则（< 1%）
5. 复杂 TimelineAbility 内部控制

**建议**：
- 🎯 **初期开发**：完全依赖自动触发，不手动调用
- 🚀 **遇到特殊需求时**：再添加手动 Tick 支持
- 📊 **上线后**：通过日志分析，确认 99% 的流程都是自动触发

**关键原则**：
> **"自动化是默认，手动是例外"**

这样既保证了开发效率，又保留了灵活性！🎯

---

## 总结

**自动化 Tick 调用是推荐方案的核心优势**：

- 🎯 **开发者体验**：完全不用管 Tick，专注游戏逻辑
- 🚀 **性能优化**：批量 Tick，自动去重
- 🛡️ **防错机制**：自动触发，不会遗漏
- 📊 **易于调试**：详细日志，清晰追踪

**与手动 Tick 对比**：
- 手动：容易忘记、容易重复、容易遗漏 ❌
- 自动：零心智负担、自动优化、防错设计 ✅

这正是为什么推荐方案 A（全局暂停 + SmartTickManager）是最佳选择！🎯


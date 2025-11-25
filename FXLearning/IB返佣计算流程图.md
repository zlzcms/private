# IB返佣计算流程图

## 🔄 返佣计算决策树

```
开始计算返佣
    │
    ├─→ 获取代理信息
    │   ├─→ 查询代理的IB等级 (ib_rank_id)
    │   └─→ 确定代理类型 (lv1/lv2)
    │
    ├─→ 获取交易信息
    │   ├─→ 账户类型 (account_type)
    │   ├─→ 交易品种 (symbol)
    │   └─→ 交易手数 (volume)
    │
    └─→ 查询返佣规则
        │
        ├─→ [步骤1] 检查交易品种特殊规则
        │   │
        │   └─→ 查询 IbCommissionSharingRuleSymbol
        │       WHERE rule_id = ? 
        │       AND account_type = ?
        │       AND symbol = ?
        │       │
        │       ├─→ [存在] ✓ 使用特殊规则
        │       │   ├─→ one_value (一级返佣)
        │       │   └─→ two_value (二级返佣)
        │       │   └─→ 返佣 = value × volume
        │       │
        │       └─→ [不存在] → 继续下一步
        │
        ├─→ [步骤2] 查询账户类型规则
        │   │
        │   └─→ 查询 IbCommissionSharingRuleItem
        │       WHERE rule_id = ?
        │       AND account_type = ?
        │       │
        │       ├─→ [FIXED类型] 固定手数返佣
        │       │   ├─→ 检查 symbol_item_type 是否匹配
        │       │   ├─→ 使用 next_common_value
        │       │   └─→ 返佣 = next_common_value × volume
        │       │
        │       └─→ [SCALE类型] 比例返佣
        │           ├─→ 使用 next_common_value (%)
        │           └─→ 返佣 = volume × (next_common_value / 100)
        │
        └─→ [步骤3] 计算最终返佣金额
            │
            ├─→ 一级代理 (lv1)
            │   └─→ 使用查询到的返佣值
            │
            └─→ 二级代理 (lv2)
                └─→ 使用查询到的返佣值（通常较低）
```

---

## 📊 返佣规则匹配优先级

```
优先级从高到低：

1. 【最高优先级】交易品种特殊规则
   └─→ IbCommissionSharingRuleSymbol
       └─→ 条件：rule_id + account_type + symbol 完全匹配
       └─→ 返回：one_value 或 two_value

2. 【中等优先级】固定手数返佣规则
   └─→ IbCommissionSharingRuleItem (FIXED)
       └─→ 条件：rule_id + account_type + symbol_item_type 匹配
       └─→ 返回：next_common_value

3. 【基础优先级】比例返佣规则
   └─→ IbCommissionSharingRuleItem (SCALE)
       └─→ 条件：rule_id + account_type 匹配
       └─→ 返回：next_common_value (%)
```

---

## 🎯 实际计算示例

### 示例1：有特殊品种规则的情况

```
场景：
- 代理等级：黄金代理 (ib_rank_id=5, rule_id=10)
- 账户类型：STDAUD
- 交易品种：EURUSD
- 交易手数：10手

计算流程：
1. 查询 IbCommissionSharingRuleSymbol
   WHERE rule_id=10 AND account_type='STDAUD' AND symbol='EURUSD'
   → 找到记录：one_value=0.15, two_value=0.10

2. 计算结果：
   - 一级返佣 = 0.15 × 10 = 1.5手
   - 二级返佣 = 0.10 × 10 = 1.0手
```

### 示例2：使用账户类型规则的情况

```
场景：
- 代理等级：黄金代理 (ib_rank_id=5, rule_id=10)
- 账户类型：STDAUD
- 交易品种：GBPUSD (无特殊规则)
- 交易手数：10手
- 规则类型：SCALE (比例返佣)
- 返佣比例：5%

计算流程：
1. 查询 IbCommissionSharingRuleSymbol
   WHERE rule_id=10 AND account_type='STDAUD' AND symbol='GBPUSD'
   → 未找到记录

2. 查询 IbCommissionSharingRuleItem
   WHERE rule_id=10 AND account_type='STDAUD' AND next_common_type='SCALE'
   → 找到记录：next_common_value=5.0

3. 计算结果：
   - 返佣 = 10 × (5.0 / 100) = 0.5手
```

### 示例3：固定手数返佣的情况

```
场景：
- 代理等级：黄金代理 (ib_rank_id=5, rule_id=10)
- 账户类型：STDAUD
- 交易品种：XAUUSD (黄金)
- 交易手数：10手
- 规则类型：FIXED (固定手数)
- 固定返佣：每手0.2

计算流程：
1. 查询 IbCommissionSharingRuleSymbol
   WHERE rule_id=10 AND account_type='STDAUD' AND symbol='XAUUSD'
   → 未找到记录

2. 查询 IbCommissionSharingRuleItem
   WHERE rule_id=10 AND account_type='STDAUD' AND next_common_type='FIXED'
   → 找到记录：next_common_value=0.2, symbol_item_type包含'黄金'

3. 计算结果：
   - 返佣 = 0.2 × 10 = 2.0手
```

---

## 🔀 数据流转图

```
[前端请求]
    │
    ↓
[控制器: IbCommissionSharingRule]
    │
    ├─→ add() / edit()
    │   ├─→ 验证数据 (IbCommissionSharingRuleRequest)
    │   ├─→ 保存主规则 (IbCommissionSharingRule)
    │   └─→ 触发事件 (IbCommissionSharingRuleSaveEvent)
    │
    ├─→ setSymbolRule()
    │   └─→ 保存/更新交易品种规则 (IbCommissionSharingRuleSymbol)
    │
    └─→ getSymbolRuleList()
        └─→ 查询并返回交易品种规则列表
            │
            ↓
[事件监听器: IbCommissionSharingRuleSavelistener]
    │
    ├─→ saveScales()
    │   └─→ 保存/更新比例返佣规则 (IbCommissionSharingRuleItem)
    │
    └─→ saveFixeds()
        └─→ 保存/更新固定手数返佣规则 (IbCommissionSharingRuleItem)
            └─→ 删除关联的交易品种规则 (如果规则被删除)
```

---

## 📋 返佣规则配置检查清单

### 创建返佣规则前需要确认：

- [ ] IB等级是否存在 (ib_rank_id)
- [ ] 代理类型是否正确 (lv1/lv2)
- [ ] 账户类型是否有效 (account_type)
- [ ] 返佣类型选择 (SCALE/FIXED)
- [ ] 返佣值是否合理 (0 < value <= 100)
- [ ] 如果是FIXED类型，symbol_item_type是否配置
- [ ] 是否需要为特定交易品种设置特殊规则

### 返佣计算时需要确认：

- [ ] 代理的IB等级是否匹配
- [ ] 账户类型是否匹配
- [ ] 交易品种是否有特殊规则
- [ ] 返佣规则是否启用 (status=0)
- [ ] 返佣值计算是否正确

---

## 🎨 返佣规则配置示例

### 完整配置示例

```json
{
  "主规则": {
    "type": "lv1",
    "ib_rank_id": 5,
    "status": 0
  },
  "比例返佣规则": [
    {
      "account_type": "STDAUD",
      "next_common_type": "SCALE",
      "next_common_value": 5.0
    },
    {
      "account_type": "STDUSD",
      "next_common_type": "SCALE",
      "next_common_value": 4.5
    }
  ],
  "固定手数返佣规则": [
    {
      "account_type": "STDAUD",
      "next_common_type": "FIXED",
      "next_common_value": 0.2,
      "symbol_item_type": ["外汇", "黄金"],
      "currency": "USD",
      "next_level_fixed_common_type": "SCALE",
      "next_level_fixed_common_value": 3.0
    }
  ],
  "交易品种特殊规则": [
    {
      "account_type": "STDAUD",
      "symbol": "EURUSD",
      "one_value": 0.15,
      "two_value": 0.10
    },
    {
      "account_type": "STDAUD",
      "symbol": "XAUUSD",
      "one_value": 0.25,
      "two_value": 0.15
    }
  ]
}
```

---

## ⚡ 性能优化建议

1. **缓存机制**：
   - 返佣规则查询结果可以缓存（当前已有部分缓存）
   - 建议缓存时间：5-10分钟

2. **索引优化**：
   - `ib_commission_sharing_rule`: (ib_rank_id, type, status)
   - `ib_commission_sharing_rule_item`: (rule_id, account_type, next_common_type)
   - `ib_commission_sharing_rule_symbol`: (rule_id, account_type, symbol)

3. **批量查询**：
   - 计算返佣时，尽量批量查询规则，避免N+1查询问题

4. **数据预加载**：
   - 在计算返佣前，预加载所有相关规则到内存


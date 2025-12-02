# IB返佣规则功能详细分析

## 📋 功能概述

IB返佣规则系统是一个多层级、多维度的代理返佣配置系统，支持按代理等级、账户类型、交易品种等不同维度设置返佣规则。

---

## 🏗️ 系统架构

### 核心数据表结构

#### 1. **主规则表 (IbCommissionSharingRule)**
- **作用**：关联IB代理等级和返佣规则
- **关键字段**：
  - `id`: 规则ID
  - `type`: 代理类型（lv1=一级代理, lv2=二级代理）
  - `ib_rank_id`: IB等级ID（关联到IbRank表）
  - `status`: 状态（0=启用, -1=禁用）

#### 2. **规则明细表 (IbCommissionSharingRuleItem)**
- **作用**：存储具体的返佣规则配置
- **关键字段**：
  - `rule_id`: 关联主规则ID
  - `account_type`: 账户类型（如：STDAUD, STDUSD等）
  - `next_common_type`: 返佣类型
    - `SCALE`: 比例返佣（按百分比）
    - `FIXED`: 固定手数返佣
  - `next_common_value`: 返佣值（比例或固定手数）
  - `symbol_item_type`: 交易品种类型（JSON格式，用于FIXED类型）
  - `currency`: 货币类型（用于FIXED类型）
  - `next_level_fixed_common_type`: 下一级固定返佣类型（SCALE）
  - `next_level_fixed_common_value`: 下一级固定返佣值

#### 3. **交易品种规则表 (IbCommissionSharingRuleSymbol)**
- **作用**：为特定交易品种设置个性化返佣规则
- **关键字段**：
  - `rule_id`: 关联主规则ID
  - `account_type`: 账户类型
  - `symbol`: 交易品种（如：EURUSD, GBPUSD等）
  - `one_value`: 一级返佣值
  - `two_value`: 二级返佣值

---

## 💰 返佣机制详解

### 返佣类型说明

#### 1. **SCALE（比例返佣）**
- **适用场景**：按交易手数的百分比返佣
- **配置方式**：
  - 为每个账户类型设置一个返佣比例
  - 例如：STDAUD账户类型，返佣比例5%
- **特点**：
  - 简单直接，按比例计算
  - 适用于所有交易品种（除非有特殊品种规则覆盖）

#### 2. **FIXED（固定手数返佣）**
- **适用场景**：按固定手数返佣，更灵活
- **配置方式**：
  - 设置固定返佣手数（`next_common_value`）
  - 可选择特定交易品种类型（`symbol_item_type`）
  - 可设置下一级的固定返佣类型和值
- **特点**：
  - 可以为不同交易品种类型设置不同的固定返佣
  - 支持多层级返佣配置

### 返佣优先级

```
交易品种特殊规则 (IbCommissionSharingRuleSymbol)
    ↓ (如果存在，优先使用)
账户类型规则 (IbCommissionSharingRuleItem)
    ↓ (基础规则)
代理等级规则 (IbCommissionSharingRule)
```

---

## 🔄 返佣计算流程

### 场景1：使用比例返佣（SCALE）

```
1. 查询代理的IB等级 → 获取对应的返佣规则ID
2. 根据账户类型查询规则明细 → 获取SCALE类型的返佣比例
3. 计算返佣 = 交易手数 × 返佣比例
```

**示例**：
- 代理等级：黄金代理（ib_rank_id=5）
- 账户类型：STDAUD
- 返佣比例：5%
- 交易手数：10手
- **返佣金额 = 10 × 5% = 0.5手**

### 场景2：使用固定手数返佣（FIXED）

```
1. 查询代理的IB等级 → 获取对应的返佣规则ID
2. 根据账户类型和交易品种类型查询规则明细 → 获取FIXED类型的固定返佣值
3. 计算返佣 = 固定返佣值 × 交易手数
```

**示例**：
- 代理等级：黄金代理（ib_rank_id=5）
- 账户类型：STDAUD
- 交易品种类型：外汇
- 固定返佣：每手0.1
- 交易手数：10手
- **返佣金额 = 0.1 × 10 = 1手**

### 场景3：交易品种特殊规则

```
1. 查询代理的IB等级 → 获取对应的返佣规则ID
2. 检查是否存在该交易品种的特殊规则（IbCommissionSharingRuleSymbol）
3. 如果存在，使用特殊规则中的one_value或two_value
4. 如果不存在，回退到账户类型规则
```

**示例**：
- 代理等级：黄金代理（ib_rank_id=5）
- 账户类型：STDAUD
- 交易品种：EURUSD
- 特殊规则：one_value=0.15, two_value=0.10
- 交易手数：10手
- **一级返佣 = 0.15 × 10 = 1.5手**
- **二级返佣 = 0.10 × 10 = 1.0手**

---

## 🎯 代理等级区分

### LV1（一级代理）
- **特点**：直接发展客户的代理
- **返佣来源**：从自己发展的客户交易中获得返佣
- **配置**：需要配置`scales`和`fixeds`两种返佣类型

### LV2（二级代理）
- **特点**：通过一级代理间接发展客户的代理
- **返佣来源**：从一级代理的客户交易中获得返佣
- **配置**：只需要配置`scales`比例返佣

---

## 📊 数据流转过程

### 1. 创建/编辑返佣规则

```php
// 控制器：IbCommissionSharingRule::add() 或 edit()
1. 验证数据（验证器：IbCommissionSharingRuleRequest）
2. 保存主规则（IbCommissionSharingRule）
3. 触发事件（IbCommissionSharingRuleSaveEvent）
4. 事件监听器处理（IbCommissionSharingRuleSavelistener）
   - 保存比例返佣规则（saveScales）
   - 保存固定手数返佣规则（saveFixeds）
```

### 2. 保存规则明细逻辑

**比例返佣（SCALE）保存逻辑**：
- 检查是否存在相同`rule_id`、`account_type`、`next_common_type='SCALE'`的记录
- 如果存在，更新；如果不存在，创建
- 删除不在新数据中的旧记录（软删除）

**固定手数返佣（FIXED）保存逻辑**：
- 检查是否存在相同`rule_id`、`account_type`、`next_common_value`、`next_common_type='FIXED'`的记录
- 如果存在，更新；如果不存在，创建
- 删除不在新数据中的旧记录（软删除）
- 同时删除对应的交易品种规则（IbCommissionSharingRuleSymbol）

### 3. 交易品种规则设置

**单个设置**（`setSymbolRule`）：
- 为特定交易品种设置`one_value`和`two_value`
- 如果记录存在则更新，不存在则创建

**批量设置**（`setBatchSymbolRule`）：
- 支持两种模式：
  - `type=1`：为选定的交易品种批量设置
  - `type=2`：为所有交易品种批量设置

**导入设置**（`importSymbolRule`）：
- 从Excel文件导入交易品种返佣规则
- 格式：symbol, one_value, two_value

---

## 🔍 关键功能点

### 1. 规则查询（`index`）
- 支持分页查询
- 自动关联查询：
  - IB等级信息（rankData）
  - 比例返佣规则（scales）
  - 固定手数返佣规则（fixeds）
  - 管理员信息（adminData）

### 2. 固定手数返佣详情（`getFixedDetail`）
- 查询指定规则下的所有固定手数返佣配置
- 支持按交易品种展开显示

### 3. 交易品种规则列表（`getSymbolRuleList`）
- 支持搜索和排序
- 关联查询交易品种规则
- 显示`one_value`和`two_value`

### 4. 导出功能（`exportSymbolRule`）
- 导出指定规则和账户类型下的所有交易品种返佣规则
- 格式：symbol, one_value, two_value

---

## 📝 数据验证规则

### 主规则验证
- `type`: 必须是lv1或lv2
- `ib_rank_id`: 必须存在且为整数
- `status`: 必须是0或-1

### 比例返佣验证
- `scales`: 必须是非空数组
- `scales.*.account_type`: 必填
- `scales.*.next_common_type`: 必须是SCALE
- `scales.*.next_common_value`: 必须是数字，大于0，最大100

### 固定手数返佣验证
- `fixeds`: 必须是非空数组
- `fixeds.*.account_type`: 必填
- `fixeds.*.symbol_item_type`: 必填
- `fixeds.*.next_common_type`: 必须是FIXED
- `fixeds.*.next_common_value`: 必须是数字，大于0，最大100
- `fixeds.*.next_level_fixed_common_type`: 必须是SCALE
- `fixeds.*.next_level_fixed_common_value`: 必须是数字，大于0，最大100

---

## 🎨 使用示例

### 示例1：为黄金代理设置比例返佣

```php
// 请求数据
{
    "type": "lv1",
    "ib_rank_id": 5,  // 黄金代理等级ID
    "status": 0,
    "scales": [
        {
            "account_type": "STDAUD",
            "next_common_type": "SCALE",
            "next_common_value": 5.0  // 5%返佣
        },
        {
            "account_type": "STDUSD",
            "next_common_type": "SCALE",
            "next_common_value": 4.5  // 4.5%返佣
        }
    ],
    "fixeds": [...]
}
```

### 示例2：为EURUSD设置特殊返佣规则

```php
// 请求数据
{
    "rule_id": 1,
    "account_type": "STDAUD",
    "symbol": "EURUSD",
    "one_value": 0.15,  // 一级返佣每手0.15
    "two_value": 0.10   // 二级返佣每手0.10
}
```

---

## ⚠️ 注意事项

1. **唯一性约束**：
   - 每个IB等级只能有一个返佣规则（通过`ib_rank_id`唯一性保证）

2. **软删除机制**：
   - 规则明细使用软删除，删除主规则时会同步删除相关明细

3. **数据一致性**：
   - 删除固定手数返佣规则时，会同步删除对应的交易品种规则

4. **二级代理限制**：
   - LV2代理只能配置比例返佣（SCALE），不能配置固定手数返佣（FIXED）

5. **交易品种规则优先级**：
   - 如果为某个交易品种设置了特殊规则，会覆盖账户类型的通用规则

---

## 🔗 相关文件

- **控制器**：`app/admin/controller/System/MtSetting/IbCommissionSharingRule.php`
- **主规则模型**：`app/admin/model/Ib/Commission/IbCommissionSharingRule.php`
- **规则明细模型**：`app/admin/model/Ib/Commission/IbCommissionSharingRuleItem.php`
- **交易品种规则模型**：`app/admin/model/Ib/Commission/IbCommissionSharingRuleSymbol.php`
- **IB等级模型**：`app/admin/model/Ib/Rank/IbRank.php`
- **验证器**：`app/admin/validate/IbCommissionSharingRuleRequest.php`
- **事件**：`app/event/IbCommissionSharingRuleSaveEvent.php`
- **监听器**：`app/listener/IbCommissionSharingRuleSavelistener.php`

---

## 📈 总结

这个IB返佣规则系统设计得非常灵活，支持：

1. **多层级代理**：区分一级和二级代理
2. **多账户类型**：为不同账户类型设置不同返佣规则
3. **多返佣方式**：支持比例返佣和固定手数返佣
4. **精细化控制**：可以为特定交易品种设置特殊返佣规则
5. **批量操作**：支持批量设置和Excel导入导出

整个系统通过事件驱动的方式处理数据保存，保证了数据的一致性和完整性。


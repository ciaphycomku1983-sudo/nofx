# 数量格式化和最小名义价值修复

## 问题描述

遇到两个关键问题：

### 1. BTCUSDT 开空单失败
```
❌ BTCUSDT open_short 失败: 格式化后的数量必须大于0，当前: 0.00000000 (原始数量: 0.00040619)
```

**原因**：`roundToTickSize` 函数使用 `math.Round`（四舍五入），导致小数量被向下取整为 0。
- 原始数量：0.00040619
- StepSize：0.001
- 计算：0.00040619 / 0.001 = 0.40619
- 四舍五入：Round(0.40619) = 0
- 结果：0 * 0.001 = 0

### 2. DOGEUSDT 开空单失败
```
❌ DOGEUSDT open_short 失败: HTTP 400: {"code":-4164,"msg":"Order's notional must be no smaller than 5.0 (unless you choose reduce only)"}
```

**原因**：虽然有最小名义价值调整逻辑，但调整后的数量再次经过 `formatQuantity` 时，可能被向下取整，导致名义价值再次低于 5 USDT。

## 解决方案

### 1. 修改 roundToTickSize 函数
将 `math.Round`（四舍五入）改为 `math.Ceil`（向上取整）：

```go
func roundToTickSize(value float64, tickSize float64) float64 {
    if tickSize <= 0 {
        return value
    }
    // 计算有多少个tick size
    steps := value / tickSize
    // 向上取整到最近的整数（确保不会变成0）
    roundedSteps := math.Ceil(steps)
    // 乘回tick size
    return roundedSteps * tickSize
}
```

**效果**：
- 原始数量：0.00040619
- StepSize：0.001
- 计算：0.00040619 / 0.001 = 0.40619
- 向上取整：Ceil(0.40619) = 1
- 结果：1 * 0.001 = 0.001 ✓

### 2. 添加二次验证
在所有开仓/平仓函数中，调整最小名义价值后添加验证：

```go
// 确保名义价值不小于5.0 USDT（AsterDex最小要求）
minNotional := 5.0
notionalValue := formattedQty * price
if notionalValue < minNotional {
    log.Printf("  ⚠️ 名义价值 %.2f USDT 小于最小值 %.2f USDT，自动调整", notionalValue, minNotional)
    formattedQty = minNotional / price
    // 重新格式化调整后的数量
    formattedQty, err = t.formatQuantity(symbol, formattedQty)
    if err != nil {
        return nil, err
    }
    // 再次验证调整后的数量是否大于0
    if formattedQty <= 0 {
        return nil, fmt.Errorf("调整后的数量仍为0，无法满足最小名义价值要求 %.2f USDT", minNotional)
    }
    notionalValue = formattedQty * price
    log.Printf("  ✓ 调整后数量: %.8f, 名义价值: %.2f USDT", formattedQty, notionalValue)
}
```

## 修改的函数
1. `roundToTickSize` - 改为向上取整
2. `OpenLong` - 添加二次验证
3. `OpenShort` - 添加二次验证
4. `CloseLong` - 添加二次验证
5. `CloseShort` - 添加二次验证

## 预期效果
- ✓ 小数量不会被格式化为 0
- ✓ 满足 AsterDex 的最小名义价值要求（5 USDT）
- ✓ 向上取整确保订单可以成交
- ✓ 明确的错误提示，避免向交易所发送无效订单

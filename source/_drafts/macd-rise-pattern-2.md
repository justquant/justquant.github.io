---
title: MACD起涨模式2 - MACD靠近0轴金叉
tags: macd
---
## 形态解读

在长期的看盘的经验中，我发现有一种MACD形态一旦出现，大概率会迎来不小的上涨幅度。这个形态就是MACD靠近0轴金叉。

这种形态有好几个含义：

1. MACD金叉，代表股价短期强势
2. diff靠近0轴，同时拐头向上，说明长期趋势开始上升。有两种细分形态：
    - diff线首次上0轴，一般是长期下跌后开始上升的初期
    - diff从0轴之上一直下降在0轴获得支撑，让后拐头向上。一般出现在上升调整结束后

## 强化条件

1. 收阳线
2. K线突破重要均线：5,10,20,60
3. 有出现连续红量（连续5天，或者6天中只有1天阴量，或者10天中只有2天阴量）

## 量化

那么该如何量化这个形态呢？MACD金叉非常好量化。那么diff靠近0轴呢？由于MACD是一个和股价相关的量，因此距离0轴多远算是靠近，应该是一个和价格有关的函数。

根据我的经验：取价格除以100比较合适。例如当前价格是12元，那么当diff位于[0,0.12]时，可以认为diff很靠近0轴。

当然如果能够用机器学习的办法，算出这个参数，那就更好了（等我学完scikit-learn之后再做）

## 介入时间

如果形态出现当天，涨停或者涨幅巨大，可在在接下来的几天回调后介入。一般要结合均线是否开始多头排列(ma5 > ma10 > ma20 > ma30 > ma60)

## 例子





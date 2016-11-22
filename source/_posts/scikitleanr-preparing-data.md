---
title: scikit-learn笔记——准备数据
tags:
  - python
  - scikit-learn
  - 机器学习
date: 2016-11-22 14:25:01
---

> 种一棵树最好的时间，是十年前和现在

终于决定踏入机器学习的世界。只为了那个梦想：养一个机器人在金融市场挖矿，然后去环游世界。

选择scikit-learn，是因为python已经成为事实上的数据分析和挖掘的语言，是很多数据科学家的首选。而且scikit-learn还可以与Spark集成，感谢Databricks的工程师。

参考：[Auto-scaling scikit-learn with Apache Spark](https://databricks.com/blog/2016/02/08/auto-scaling-scikit-learn-with-apache-spark.html)

俗话说：兵马未动粮草先行。要学习scikit-learn，首先需要有数据。作为学习，Scikit-learn提供了三种方式获取数据。

# 内部自带的数据

scikit-learn包自带的数据在datasets模块当中。

```python
from sklearn import datasets
import numpy as np
```
在IPython中，通过输入datasets.*?会列出datasets包含的所有API。

```python
boston = d.load_boston()
print(boston.DESCR)
```

内部数据的接口用：datasets.load_*?()

# 外部数据

datasets模块也包含了API获取外部的数据。这些API以 fetch_*? 开头。

```python
housing = datasets.fetch_california_housing()
```

# 造数据

除了已有的数据，scikit-learn还提供了丰富的API来生成少量的数据。

这些接口都以datasets.make_*?开头。

```python
datasets.make_regression(1000,10,5,2,1.0)
```

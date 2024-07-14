---
title: Alfred 精确计算器
description: 扩展 Alfred 原生计算器能力
date: 2024-07-14 18:44:59
categories: [Technology]
tags:
  - mac
  - alfred
cover: /img/alfred-accurate-calculator/calculator.png
top_img: /img/alfred-accurate-calculator/alfred.png
---

## 背景

Alfred 自带了一个计算器，可以在激活后直接输入运算符进行计算：

![alfred-calculator-with-big-number](/img/alfred-accurate-calculator/alfred-calculator-with-big-number.png)

但是正如上图所示，Alfred 原生的计算器并不能处理大数场景，在公司内部开发时，我们经常需要跟 snowflake 算法生成的 ID 打交道（非常长的数字），因此就经常需要进行大数间的运算。

## 问题

如上图的计算结果是：`1.23456789e18`，实际上等价于 `1234567890000000000`，是一个错误的结果。

我们期望的正确结果是：`1234567890123456790`，Alfred 原生计算器丢失了精度信息。

## 解决方案

Python 在这种情况下工作正常：

```python
$ python3 -c "print(eval('1234567890123456789+1'))"
1234567890123456790
```

Python 做了很多努力来尽可能使它的数字类型（int）与数学层面上的数字等价，即从用户视角看，Python 的 int 并没有 Java 的 byte、char、int、long、bigint 的区分，是无限大的。

因此我们使用 Python 进行数字运算时，不需要考虑精度问题，这也为解决这个问题提供了一个方案。

## 实现

如上一节的示例，我们只需要一个简单的 `print(eval('${expression}'))` 即可，这里我将其封装为 Alfred 的 workflow，以便可以更简单的复用：

https://github.com/akiakise/alfred-accurate-calculator/releases/tag/v1.0

下载 release 界面的 `Accurate.Calculator.alfredworkflow` 然后双击，Alfred 会自动完成安装进程。

## 使用

呼出 Alfred 菜单后使用 `calc` 前缀激活 workflow，然后输入你的计算语句即可：

![Usage 1](/img/alfred-accurate-calculator/usage-1.png)

![Usage 2](/img/alfred-accurate-calculator/usage-2.png)

![Usage 3](/img/alfred-accurate-calculator/usage-3.png)

## 源码

https://github.com/akiakise/alfred-accurate-calculator

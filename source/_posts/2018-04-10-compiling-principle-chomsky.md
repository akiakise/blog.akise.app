---
title: 编译原理之 Chomsky 文法的判断 —— Java 实现
date: 2018-04-10 16:58:38 +0800
categories: [Learn]
tags: [learn]
mathjax: true
---

文法的定义和记号

$$ G = (V_N, V_T, P, S) \qquad (V_N \cap V_T = \varnothing, V_N \cup V_T = V) $$

是 N.Chomsky 在 1956 年描述形式语言时首先给出的。同时，Chomsky 还对产生式的形式给以不同的限制而定义了四类基本的文法，分别称之为 0 型文法，1 型文法，2 型文法和 3 型文法。

<!-- more -->

## 明确定义

- 0 型文法：对任一产生式 $\alpha \to \beta$，都有 $\alpha \in V^+, \quad \beta \in V^*$
- 1 型文法：对任一产生式 $\alpha \to \beta$，都有 $\mid\beta\mid \ge \mid\alpha\mid, \quad \alpha,\beta \in V^+$，仅仅 $\alpha \to \epsilon$ 除外
- 2 型文法：对任一产生式 $A \to \beta$，都有 $A \in V_N, \quad \beta \in V^+$
- 3 型文法：任一产生式都形如 $A \to Ba$ 或 $A \to a$，其中 $A,B \in V_N, \quad a \in V_T$，该文法称为右线性文法。类似可定义左线性文法。

## 判断思路

### 0 型文法

- 字符串 $\alpha$ 的范围是 $V^+$，是全符号集的一个正闭包，即符号集中所有符号的任意组合，且不包含 $\epsilon$ 元素

- 字符串 $\beta$ 的范围是 $V^*$，是全符号集的自反传递闭包，也即 $V^+ \cup \{\epsilon\}$

- 要判断一个文法是否是 0 型文法，只需要判断**左侧非空且不全为小写**即可

- 任何 0 型语言都是递归可枚举的，故 0 型语言又称**递归可枚举集**

### 1 型文法

- 首先 1 型文法必须是 0 型文法

- 1 型文法除了 $\alpha \to \epsilon$ 这个特例外，其他情况都满足 **$\beta$ 的长度大于 $\alpha$ 的长度**

- 1 型文法也叫**上下文相关（敏感）文法**

### 2 型文法

- 首先 2 型文法必须是 1 型文法

- 2 型文法**左侧必须是一个非终结符**

- 2 型文法也叫**上下文无关文法**

### 3 型文法

- 首先 3 型文法必须是 2 型文法

- 3 型文法必须满足以下两种形式之一

1. $A \to aB$ 或 $A \to a$
1. $A \to Ba$ 或 $A \to a$

- 3 型文法也叫正规文法

## 代码实现

### 保存产生式集

可以使用 `List` 来保存所有产生式，用内部类来表示具体的产生式：

```java
public class Chomsky {
    private List<Producer> producers = new ArrayList<>();

    private static final class Producer {
        String left;
        String right;

        public Producer(String left, String right) {
            this.left = left;
            this.right = right;
        }
    }
}
```

### 判断 0 型文法

遍历产生式集，如果有一个产生式不符合条件，则直接退出。

```java
public boolean isZero() {
    for (Producer producer : producers) {
        if (producer.left.length() == 0 || producer.left.equals(producer.left.toLowerCase())) {
            // 判断产生式的左部是否为空或者全部是小写，如果是则不是 0 型文法
            return false;
        }
    }
    return true;
}
```

关于判断左部是否全为小写，使用 `String.toLowerCase()` 将其转换为小写后再与原产生式比较，相等则原产生式为小写。

### 判断 1 型文法

```java
public boolean isFirst() {
    if (isZero()) {
        for (Producer producer : producers) {
            if (producer.right.length() != 0 && producer.right.length() < producer.left.length()) {
                // 1 型文法必须右部长度大于左部，除了右部为ε的情况
                return false;
            }
        }
        return true;
    } else {
        return false;
    }
}
```

1 型文法必须先是 0 型文法。

1 型文法的判断跳过了右部为 $\epsilon$ 的情况，即允许右部为 $\epsilon$。

### 判断 2 型文法

```java
public boolean isSecond() {
    if (isFirst()) {
        for (Producer producer : producers) {
            if (producer.left.length() != 1 || producer.left.matches("[a-z]")) {
                // 2 型文法左部必须为 1，且左部为 1 时不能为非终结符
                return false;
            }
        }
        return true;
    } else {
        return false;
    }
}
```

2 型文法必须先是 1 型文法。

首先限制左部长度必须为 1，然后利用正则判断是否为小写，`[a-z]` 匹配一个小写字符。

### 判断 3 型文法

```java
public boolean isThird() {
    if (isSecond()) {
        int countLeft = 0, countRight = 0;
        for (Producer producer : producers) {
            if (producer.right.matches("[A-Z]?[a-z]")) {
                countLeft++;
            }
            if (producer.right.matches("[a-z][A-Z]?")) {
                countRight++;
            }
        }
        if (countLeft == producers.size()) {
            System.out.println("此文法是 3 型文法，并且是左线性文法！");
            return true;
        }
        if (countRight == producers.size()) {
            System.out.println("此文法是 3 型文法，并且是右线性文法！");
            return true;
        }
    }
    return false;
}
```

3 型文法必须先是 2 型文法。

## 全部源代码

Chomsky.java:

```java
import java.util.*;

public class Chomsky {
    private List<Producer> producers = new ArrayList<>();

    public void add(Producer producer) {
        producers.add(producer);
    }

    public boolean isZero() {
        for (Producer producer : producers) {
            if (producer.left.length() == 0 || producer.left.equals(producer.left.toLowerCase())) {
                // 判断产生式的左部是否为空或者全部是小写，如果是则不是 0 型文法
                return false;
            }
        }
        return true;
    }

    public boolean isFirst() {
        if (isZero()) {
            for (Producer producer : producers) {
                if (producer.right.length() != 0 && producer.right.length() < producer.left.length()) {
                    // 1 型文法必须右部长度大于左部，除了右部为ε的情况
                    return false;
                }
            }
            return true;
        } else {
            return false;
        }
    }

    public boolean isSecond() {
        if (isFirst()) {
            for (Producer producer : producers) {
                if (producer.left.length() != 1 || producer.left.matches("[a-z]")) {
                    // 2 型文法左部必须为 1，且左部为 1 时不能为非终结符
                    return false;
                }
            }
            return true;
        } else {
            return false;
        }
    }

    public boolean isThird() {
        if (isSecond()) {
            int countLeft = 0, countRight = 0;
            for (Producer producer : producers) {
                if (producer.right.matches("[A-Z]?[a-z]")) {
                    countLeft++;
                }
                if (producer.right.matches("[a-z][A-Z]?")) {
                    countRight++;
                }
            }
            if (countLeft == producers.size()) {
                System.out.println("此文法是 3 型文法，并且是左线性文法！");
                return true;
            }
            if (countRight == producers.size()) {
                System.out.println("此文法是 3 型文法，并且是右线性文法！");
                return true;
            }
        }
        return false;
    }

    /**
    * 判断文法的具体类型，从 3 到 0 逐步判断
    */
    public void test() {
        if (!isThird()) {
            if (isSecond()) {
                System.out.println("此文法是 2 型文法");
            } else if (isFirst()) {
                System.out.println("此文法是 1 型文法");
            } else if (isZero()) {
                System.out.println("此文法是 0 型文法");
            } else {
                System.out.println("此文法不是 0 型文法！");
            }
        }
    }

    private static final class Producer {
        String left;
        String right;

        public Producer(String left, String right) {
            this.left = left;
            this.right = right;
        }
    }

    public static void main(String[] args) {
        int count;
        String[] input;
        Scanner scanner = new Scanner(System.in);
        Chomsky chomsky = new Chomsky();

        System.out.println("请输入产生式的个数:");
        count = scanner.nextInt();
        System.out.println("请依次输入产生式（示例：A->ab，一行一个）：");
        for (int i = 0; i < count; i++) {
            input = scanner.next().split("->");
            try {
                chomsky.add(new Producer(input[0], input[1]));
            } catch (ArrayIndexOutOfBoundsException e) {
                System.out.println("输入非法，请重新运行程序！！！");
                System.exit(1);
            }
        }

        chomsky.test();

        scanner.close();
    }
}
```

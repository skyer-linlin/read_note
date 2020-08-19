# darts 一道非常简单的题目，但我还是没能写好

题目描述如下:

简单来说，就是一个飞镖游戏

- 离原点 10 单位外，得 0 分
- 离原点 10 单位内，得 1 分
- 离原点 5 单位内，得 5 分
- 离原点 1 单位内，得 10 分

最初给出得类模板如下：

```java
class Darts {
    Darts(double x, double y) {
        // todo
    }

    int score() {
        // todo
    }
}
```

不假思索，实现如下：

```java
class Darts {
    double x;
    double y;

    Darts(double x, double y) {
        this.x = x;
        this.y = y;
    }

    int score() {
        double exp = Math.sqrt(x * x + y * y);
        System.out.println("exp is: " + (x * x + y * y));
        return exp > 10 ? 0 : (exp > 5 ? 1 : (exp > 1 ? 5 : 10));
    }

}
```

判断得分这里很啰嗦，但是能少打几个字，也没想到更好的实现...

比较欣赏的星星多的解法：

```java
class Darts {
    double distanceSquared;

    Darts(double x, double y) {
        this.distanceSquared = x * x + y * y;
    }

    int score() {
        if (withinCircle(1)) {
            return 10;
        } else if (withinCircle(5)) {
            return 5;
        } else if (withinCircle(10)) {
            return 1;
        } else {
            return 0;
        }
    }

    private boolean withinCircle(int radius) {
        return distanceSquared <= radius * radius;
    }
}
```

对比：

1. 构造器：我的实现使用了 2 个 double，这个实现只存储了 1 个 double，空间占用完胜我
2. 计算量：
   - 很明显，我的实现使用了 sqrt 来求平方根，这个实现我所知道的高效方法就是使用牛顿迭代法，要算出一个精确到 10 位以后的值，估计至少得迭代 10 次，不知道 JDK 底层是不是有什么黑魔法，总之 sqrt 算法复杂度可以很高
   - 对比之下，只使用 `x * x + y * y` 来比较是否落在圆内，只需要两个乘法运算，直接就有 cpu 指令级别的支持，消耗几个时钟周期就足够了，这差距可以说是天差地别
3. 对于分数的条件分支，看结果很简单，模式抽象的非常清晰，但我是依照题意从外圆开始判断，这种就必须要内部嵌套判断了，从内圆依次判断则如参考实现如此简单清晰

虽然只是几行比 1+1 稍微复杂了一点的代码，但我还是没能写好，真是惭愧。没有好的设计就没有好的实现，写好代码先从认真思考一个脱离编码层面的设计开始。

---
title: "横向控制：Pure Pursuit"
description: "用前视点计算转向角。"
date: 2026-05-09
draft: false
series: "自动驾驶控制：入门篇"
weight: 5
summary: "用前视点计算转向角。"
tags: ["自动驾驶", "控制算法", "入门篇"]
---

## 这篇文章讲什么

上一篇讲了纵向控制：

```text
目标速度 - 当前速度 → P 控制器 → 加速度 a
```

这一篇讲横向控制。

横向控制要解决的问题是：

```text
车应该往左打多少，或者往右打多少？
```

在自动驾驶控制里，横向控制通常负责输出：

```text
转向角 δ
```

这一篇只讲一个最适合入门的横向控制方法：

```text
Pure Pursuit
```

## 横向控制控制什么

车辆控制可以先拆成两部分：

```text
纵向控制：控制速度，输出加速度 a
横向控制：控制方向，输出转向角 δ
```

横向控制不是直接控制车辆的 `x` 或 `y`。

它真正输出的是转向角。

也就是说，横向控制器要根据：

- 当前车辆位置
- 当前车辆朝向
- 前方参考轨迹

算出：

```text
方向盘应该往哪边打，打多少
```

## Pure Pursuit 的直觉

Pure Pursuit 的中文可以理解成“纯追踪”。

它的直觉非常简单：

```text
不要盯着脚下最近点，而是看前方轨迹上的一个目标点。
然后让车朝这个目标点开过去。
```

这个目标点就叫：

```text
前视点
```

车辆和前视点之间的距离叫：

```text
前视距离
```

Pure Pursuit 的核心动作就是：

```text
找前视点 → 计算目标点相对车辆的角度 → 输出转向角
```

![](/images/auto-control-beginner/05_pure_pursuit.png)

## 前视点是什么

参考轨迹是一串点。

车辆控制时，通常先找到离当前车辆最近的轨迹点。

然后沿着轨迹往前找一段距离，找到一个前方目标点。

这个点就是前视点。

可以简单理解成：

```text
车辆不是看脚下，而是看前方。
```

这样做的好处是车辆不会为了贴住最近点而频繁左右修正。

它会更平滑地沿着轨迹走。

![](/images/auto-control-beginner/05_pure_pursuit_tracking.gif)

## 前视距离怎么理解

前视距离通常记作：

```text
Ld
```

`Ld` 太小，车辆会盯得太近。

结果可能是：

- 反应很快
- 转向频繁
- 容易抖动

`Ld` 太大，车辆会看得太远。

结果可能是：

- 行驶更平滑
- 反应变慢
- 弯道里可能切弯或跟踪不准

所以前视距离也不是越大越好。

入门阶段可以先这样记：

```text
Ld 小：跟得紧，但容易抖
Ld 大：更平滑，但反应慢
```

实际工程里，前视距离经常会和速度有关。

速度越快，看远一点；速度越慢，看近一点。

最简单可以写成：


$$
L_d = k v + L_0
$$


其中：

| 符号 | 含义 |
| --- | --- |
| `Ld` | 前视距离 |
| `v` | 当前速度 |
| `k` | 速度系数 |
| `L0` | 最小前视距离 |

## 转向角怎么计算

Pure Pursuit 的目标是让车辆走向前视点。

这里需要一个角度：

```text
α
```

`α` 表示前视点相对于车辆当前朝向的夹角。

如果前视点在车头左侧，车辆就向左打方向。

如果前视点在车头右侧，车辆就向右打方向。

常见的 Pure Pursuit 转向角公式是：


$$
\delta = \arctan \left( \frac{2L\sin(\alpha)}{L_d} \right)
$$


其中：

| 符号 | 含义 |
| --- | --- |
| `δ` | 转向角 |
| `L` | 车辆轴距 |
| `α` | 前视点相对车辆朝向的夹角 |
| `Ld` | 前视距离 |

这篇不展开公式推导。

只需要理解这个公式在做什么：

```text
前视点偏得越多，转向角越大。
前视距离越远，转向会更平缓。
```

## 工程实现思路

Pure Pursuit 的工程流程可以先写成这样：

```text
1. 读取当前车辆状态 x, y, yaw, v
2. 在参考轨迹中找到最近点
3. 从最近点沿轨迹往前找前视点
4. 把前视点转换到车辆坐标系
5. 计算夹角 α
6. 根据公式计算转向角 δ
7. 限制 δ 的范围
8. 输出 δ
```

最小伪代码可以这样写：

```text
def pure_pursuit_controller(state, reference_path):
    nearest = find_nearest_point(reference_path, state)
    lookahead = find_lookahead_point(reference_path, nearest, Ld)

    alpha = angle_to_target(state, lookahead)
    delta = atan(2 * L * sin(alpha) / Ld)
    delta = clamp(delta, -max_steer, max_steer)

    return delta
```

这就是 Pure Pursuit 的核心。

它不需要复杂优化，也不需要很重的模型。

只要能找到前视点，就能算出一个直观的转向角。

## 参数怎么调

Pure Pursuit 里最关键的参数是前视距离 `Ld`。

可以先按这个思路调：

| 现象 | 可能原因 | 调整方向 |
| --- | --- | --- |
| 车辆左右抖动 | 前视距离太小 | 增大 `Ld` |
| 车辆转弯反应慢 | 前视距离太大 | 减小 `Ld` |
| 高速时不稳定 | 看得太近 | 让 `Ld` 随速度增大 |
| 低速转弯不准 | 看得太远 | 减小最小前视距离 |

入门阶段可以先用固定前视距离。

比如：

```text
Ld = 3.0 m
```

等系统跑起来后，再改成和速度相关：

```text
Ld = k * v + L0
```

## 放到控制循环里

Pure Pursuit 会和上一篇的 P 控制器一起工作。

在控制循环里可以这样理解：

```text
while running:
    state = get_vehicle_state()
    target_point = find_target_point(reference_path, state)

    a = p_speed_controller(target_point.v, state.v, kp)
    delta = pure_pursuit_controller(state, reference_path)

    apply_control(a, delta)
```

其中：

- `a` 负责车速
- `delta` 负责方向

到这里，最小控制系统已经基本成型。

下一篇只需要把这些模块拼起来。

## 小结

这一篇只需要记住一句话：

Pure Pursuit 的核心就是让车辆追踪前方轨迹上的一个目标点。

它的流程是：

```text
找最近点 → 找前视点 → 计算夹角 α → 输出转向角 δ
```

公式是：


$$
\delta = \arctan \left( \frac{2L\sin(\alpha)}{L_d} \right)
$$


有了上一篇的加速度 `a`，再加上这一篇的转向角 `δ`，下一篇就可以把最小轨迹跟踪系统完整拼起来。

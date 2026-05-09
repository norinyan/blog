---
title: "纵向控制：P 控制器"
description: "用速度误差计算加速度。"
date: 2026-05-09
draft: false
series: "自动驾驶控制：入门篇"
weight: 4
summary: "用速度误差计算加速度。"
tags: ["自动驾驶", "控制算法", "入门篇"]
---

## 这篇文章讲什么

前面已经讲了：

- 控制器需要参考轨迹和车辆状态
- 车辆模型描述车辆怎么动

这一篇开始真正写控制器。

先从最简单的纵向控制开始。

纵向控制要解决的问题是：

```text
车速应该怎么控制？
```

在自动驾驶控制里，纵向控制通常负责输出：

```text
加速度 a
```

也就是让车：

- 该快的时候快一点
- 该慢的时候慢一点
- 该保持速度的时候尽量稳定

这一篇只讲最简单的 P 控制器。

## 纵向控制控制什么

车辆运动可以简单拆成两个方向：

```text
纵向：车沿着前进方向快慢变化
横向：车往左还是往右转
```

纵向控制管的是速度。

它不关心车现在应该往左还是往右，它只关心：

```text
目标速度是多少？
当前速度是多少？
我应该加速还是减速？
```

所以纵向控制器的输入可以先理解成：

```text
目标速度 v_ref
当前速度 v
```

输出是：

```text
加速度 a
```

![](/images/auto-control-beginner/04_p_controller_io.png)

## 目标速度和当前速度

目标速度来自参考轨迹。

比如某个参考点上写着：

```text
v_ref = 10 m/s
```

意思是车辆经过这个点附近时，最好以 `10 m/s` 左右的速度行驶。

当前速度来自车辆状态。

比如当前车辆状态里写着：

```text
v = 8 m/s
```

说明现在车速只有 `8 m/s`。

这时候控制器看到：

```text
目标速度 10 m/s
当前速度 8 m/s
```

就知道车慢了，应该加速。

如果当前速度是 `12 m/s`，那就说明车快了，应该减速。

## 速度误差

P 控制器的核心是误差。

纵向速度误差定义为：


$$
e_v = v_{ref} - v
$$


其中：

| 符号 | 含义 |
| --- | --- |
| `e_v` | 速度误差 |
| `v_ref` | 目标速度 |
| `v` | 当前速度 |

这个公式很直观：

- `e_v > 0`：当前速度比目标速度低，需要加速
- `e_v < 0`：当前速度比目标速度高，需要减速
- `e_v = 0`：当前速度接近目标速度，不需要明显调整

![](/images/auto-control-beginner/04_speed_error.png)

## P 控制器公式

P 控制器就是把误差乘上一个系数。

公式是：


$$
a = K_p e_v
$$


也就是：


$$
a = K_p (v_{ref} - v)
$$


其中：

| 符号 | 含义 |
| --- | --- |
| `a` | 输出加速度 |
| `K_p` | 比例系数 |
| `e_v` | 速度误差 |

它的直觉很简单：

```text
速度差得越多，加速度越大
速度差得越少，加速度越小
```

比如：

| 目标速度 | 当前速度 | 速度误差 | 控制结果 |
| --- | --- | --- | --- |
| 10 | 8 | 2 | 加速 |
| 10 | 10 | 0 | 基本不变 |
| 10 | 12 | -2 | 减速 |

这就是 P 控制器最核心的思想。

## Kp 怎么理解

`K_p` 决定控制器有多“激进”。

如果 `K_p` 太小：

- 加速很慢
- 速度追踪反应迟钝
- 目标速度变了以后，车辆很久才跟上

如果 `K_p` 太大：

- 加速度变化太猛
- 速度可能来回波动
- 乘坐体验会变差

所以 `K_p` 不是越大越好。

入门阶段可以先这样理解：

```text
Kp 小：稳，但慢
Kp 大：快，但容易抖
```

实际调参时，通常先给一个小值，让车能稳定追速度，再慢慢加大。

![](/images/auto-control-beginner/04_p_speed_tracking.gif)

## 从加速度到油门刹车

控制算法里算出来的是加速度 `a`。

但真实车辆或者仿真器通常接收的是：

- 油门
- 刹车
- 方向盘

所以还需要一步转换。

最简单的处理方式是：

```text
如果 a > 0：给油门
如果 a < 0：给刹车
```

比如可以先写成：

```text
if a > 0:
    throttle = map_acc_to_throttle(a)
    brake = 0
else:
    throttle = 0
    brake = map_acc_to_brake(-a)
```

在入门系统里，不需要一开始就把油门刹车映射做得特别精细。

先把“速度误差 → 加速度 → 油门/刹车”这条链路跑通最重要。

## 工程实现思路

一个最小 P 速度控制器可以这样写：

```text
def p_speed_controller(target_speed, current_speed, kp):
    error = target_speed - current_speed
    acceleration = kp * error
    return acceleration
```

真实使用时，一般还要给加速度加限制。

比如：

```text
acceleration = clamp(acceleration, min_acc, max_acc)
```

原因很简单：

如果速度误差很大，P 控制器可能算出一个很大的加速度。

但车辆不可能无限加速，也不应该猛加速。

所以通常会限制输出范围：

```text
最大加速度
最大减速度
```

一个更完整一点的版本可以写成：

```text
def p_speed_controller(target_speed, current_speed, kp):
    error = target_speed - current_speed
    acceleration = kp * error
    acceleration = clamp(acceleration, -max_brake_acc, max_acc)
    return acceleration
```

## 放到控制循环里

纵向控制器不是单独运行的。

它会放在整个自动驾驶控制循环里：

```text
while running:
    state = get_vehicle_state()
    target_point = find_target_point(reference_path, state)

    target_speed = target_point.v
    current_speed = state.v

    a = p_speed_controller(target_speed, current_speed, kp)
```

这一步算出来的 `a`，后面会和横向控制器算出来的转向角 `δ` 一起发送给车辆。

也就是：

```text
纵向控制器 → 加速度 a
横向控制器 → 转向角 δ
```

这一篇只解决前半部分。

## 小结

这一篇只需要记住一句话：

P 控制器用速度误差来计算加速度。

公式就是：


$$
e_v = v_{ref} - v
$$



$$
a = K_p e_v
$$


它简单，但足够帮我们搭起最小自动驾驶控制系统里的纵向控制部分。

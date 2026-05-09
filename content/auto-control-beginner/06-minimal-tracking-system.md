---
title: "最小轨迹跟踪系统实现"
description: "把前面模块拼成一个闭环系统。"
date: 2026-05-09
draft: false
series: "自动驾驶控制：入门篇"
weight: 6
summary: "把前面模块拼成一个闭环系统。"
tags: ["自动驾驶", "控制算法", "入门篇"]
---

## 这篇文章讲什么

前面 5 篇其实已经把一个最小控制系统拆开讲完了：

- 第 1 篇：控制系统整体是什么
- 第 2 篇：参考轨迹和车辆状态是什么
- 第 3 篇：车辆模型描述车辆怎么动
- 第 4 篇：纵向 P 控制器怎么控制速度
- 第 5 篇：横向 Pure Pursuit 怎么控制方向

这一篇把它们拼起来。

目标不是写一个复杂自动驾驶系统，而是搭一个最小可实现的轨迹跟踪闭环：

```text
读取车辆状态
找到目标轨迹点
P 控制器算加速度
Pure Pursuit 算转向角
把控制量发给车辆
循环执行
```

只要这条链路跑通，就已经有了一个最小自动驾驶控制系统的雏形。

## 最小系统包含哪些模块

最小轨迹跟踪系统可以拆成 5 个模块：

| 模块 | 作用 |
| --- | --- |
| 参考轨迹 | 告诉车辆应该沿哪里走 |
| 车辆状态 | 告诉控制器车辆现在在哪里 |
| 目标点搜索 | 找最近点和前视点 |
| 纵向控制器 | 根据速度误差输出加速度 `a` |
| 横向控制器 | 根据前视点输出转向角 `δ` |

整体流程是：

```text
参考轨迹 + 车辆状态
        ↓
目标点搜索
        ↓
纵向 P 控制器 + 横向 Pure Pursuit
        ↓
加速度 a + 转向角 δ
        ↓
车辆执行
        ↓
新的车辆状态
```

![](/images/auto-control-beginner/06_minimal_tracking_system.png)

## 读取参考轨迹

参考轨迹通常是一组离散点。

最小系统里，每个点可以先包含：

```text
x, y, yaw, v
```

其中：

- `x, y`：参考点位置
- `yaw`：参考方向
- `v`：参考速度

可以把参考轨迹理解成一个列表：

```text
reference_path = [
    {x, y, yaw, v},
    {x, y, yaw, v},
    {x, y, yaw, v},
    ...
]
```

后面控制器每一帧都要从这条轨迹里找到当前车辆应该追踪的目标点。

## 读取车辆状态

车辆状态是控制器每一帧读取的当前信息。

最小系统里，需要：

```text
x, y, yaw, v
```

也就是：

- 当前车辆位置
- 当前车辆朝向
- 当前车辆速度

在不同平台里，这些状态的来源不一样。

它可能来自仿真器，也可能来自真实车辆的定位、里程计、IMU 或底盘反馈。

但从控制算法角度看，先记住这一句就够：

```text
控制器每一帧都要知道车辆当前在哪里、朝哪、速度多少。
```

## 找最近点和前视点

拿到车辆状态之后，要先在参考轨迹中找到车辆附近的点。

最小做法是：

```text
1. 遍历参考轨迹
2. 找到离车辆当前位置最近的点
3. 从最近点沿轨迹向前找一段距离
4. 得到前视点
```

最近点用于判断车辆当前处在轨迹的哪个位置。

前视点用于 Pure Pursuit 横向控制。

这里可以先不追求搜索效率。

入门阶段即使用遍历，也足够帮助理解系统流程。

## 纵向 P 控制器

纵向控制器负责速度。

输入是：

```text
目标速度 v_ref
当前速度 v
```

输出是：

```text
加速度 a
```

公式是：


$$
e_v = v_{ref} - v
$$



$$
a = K_p e_v
$$


其中 `v_ref` 来自前视点或目标参考点，`v` 来自当前车辆状态。

最小代码就是：

```text
error = target_speed - current_speed
a = kp * error
a = clamp(a, min_acc, max_acc)
```

## 横向 Pure Pursuit 控制器

横向控制器负责方向。

输入是：

```text
当前车辆状态
前视点
```

输出是：

```text
转向角 δ
```

核心公式是：


$$
\delta = \arctan \left( \frac{2L\sin(\alpha)}{L_d} \right)
$$


其中：

- `L` 是车辆轴距
- `α` 是前视点相对车辆朝向的夹角
- `Ld` 是前视距离

最小代码思路是：

```text
lookahead = find_lookahead_point(reference_path, state)
alpha = angle_to_target(state, lookahead)
delta = atan(2 * L * sin(alpha) / Ld)
delta = clamp(delta, -max_steer, max_steer)
```

## 合成控制输出

到这里，两个控制器分别给出了两个控制量：

```text
纵向 P 控制器 → 加速度 a
横向 Pure Pursuit → 转向角 δ
```

然后把它们合成一条控制指令：

```text
control = {
    acceleration: a,
    steering: delta
}
```

如果是仿真器或真实车辆，还要转换成执行器能接收的格式。

常见执行器接口可能不是直接接收 `a` 和 `δ`，而是接收：

```text
throttle
brake
steer
```

最简单转换逻辑是：

```text
if a >= 0:
    throttle = map_acc_to_throttle(a)
    brake = 0
else:
    throttle = 0
    brake = map_acc_to_brake(-a)

steer = map_delta_to_steer(delta)
```

这一步不追求特别精确。

入门阶段先把控制链路跑通最重要。

## 控制循环

整个最小系统的核心就是一个循环。

伪代码如下：

```text
reference_path = load_reference_path()

while running:
    state = get_vehicle_state()

    nearest_point = find_nearest_point(reference_path, state)
    target_point = find_lookahead_point(reference_path, nearest_point, Ld)

    target_speed = target_point.v
    current_speed = state.v

    a = p_speed_controller(target_speed, current_speed, kp)
    delta = pure_pursuit_controller(state, target_point, Ld)

    control = convert_to_actuator_command(a, delta)
    apply_control(control)
```

这段伪代码就是这个入门系列的核心成果。

它把前面所有概念串起来了。

![](/images/auto-control-beginner/06_minimal_tracking.gif)

## 把前 5 篇串起来

现在回头看，这个系统并不神秘。

```text
参考轨迹：告诉车应该去哪
车辆状态：告诉控制器车现在在哪
车辆模型：告诉我们车会怎么动
P 控制器：根据速度误差算加速度
Pure Pursuit：根据前视点算转向角
控制循环：把这些模块反复执行
```

每一篇都只是其中一块。

第六篇做的事情，就是把它们接成一个闭环。

这个闭环一旦跑起来，车辆就不再是只算一次控制量，而是不断：

```text
观察当前状态
计算控制量
执行控制量
得到新的状态
继续下一轮
```

这就是自动驾驶控制里最核心的“闭环”。

## 最小系统先不追求什么

这个入门系统的目标是跑通，不是最优。

所以先不追求：

- 高速极限控制
- 复杂车辆动力学
- 最优控制
- 精细油门刹车标定
- 各种复杂道路场景

先让车辆能沿着一条参考轨迹稳定走起来。

这一步完成后，再去升级 PID、Stanley、LQR、MPC，思路会清楚很多。

因为你知道自己到底在升级哪个模块。

## 小结

到这里，入门篇的最小系统就完整了。

整个系统可以压缩成一句话：

```text
参考轨迹和车辆状态进来，P 控制器算加速度，Pure Pursuit 算转向角，然后车辆执行，循环更新。
```

这套系统不复杂，但它已经包含了自动驾驶控制最核心的闭环思想。

后面再学 PID、Stanley、LQR、MPC，本质上都是在升级其中某个模块。

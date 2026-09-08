# 多相机 + IMU + Mocap 标定现场采集 Checklist

## 今天需要录 5 个数据包

```text
□ cam_static_01
□ cam_static_02

□ joint_dynamic_01
□ joint_dynamic_02

□ validation_01
```

---

# 录制前：必须全部打勾

### Rig

```text
□ 相机没有移动
□ IMU 没有移动
□ Mocap marker 没有移动
□ 所有螺丝锁紧
□ 镜头 focus 没有变化
```

### Camera

```text
□ 分辨率正确
□ FPS 正确
□ Exposure 正确
□ Gain 正确
□ 无明显过曝
□ 无明显 motion blur
□ 所有 Camera 图像正常
```

### IMU

```text
□ IMU 正常输出
□ 频率正确
□ Timestamp 连续
□ 无明显数据 gap
```

### Mocap

```text
□ rigid body 正确
□ marker tracking 稳定
□ Mocap 坐标正常
□ Timestamp 正常
```

### AprilGrid

```text
□ 使用正确的标定板
□ tag size 参数正确
□ tag spacing 参数正确
□ 标定板平整
```

---

# ① cam_static_01 / 02

## 设备怎么动？

```text
Rig：       不动
AprilGrid：人拿着移动
```

### 必须覆盖

```text
□ 图像中心

□ 上
□ 下
□ 左
□ 右

□ 左上
□ 右上
□ 左下
□ 右下
```

### 必须改变距离

```text
□ 近
□ 中
□ 远
```

### 必须改变标定板角度

```text
□ 正视
□ 左倾
□ 右倾
□ 上倾
□ 下倾
□ Roll
```

### 多 Camera 最重要

```text
□ cam0 + cam1 同时看到板
□ cam1 + cam2 同时看到板
□ cam2 + cam3 同时看到板
□ cam3 + cam0 同时看到板
```

至少保证所有 Camera 形成连通观测关系。

### 动作方式

推荐：

```text
移动
 ↓
停一下
 ↓
换位置
 ↓
停一下
```

不要快速挥动标定板。

### 时间

```text
约 1–2 min / 条
```

---

# ② joint_dynamic_01 / 02

## 设备怎么动？

```text
AprilGrid：固定不动
Rig：       人拿着运动
Mocap：     全程 tracking Rig
```

## 开始

```text
□ 静止 3–5 s
```

## Basalt 示例动作顺序

```text
前后 → 上下 → 左右 → Roll → Pitch → Yaw → 8 字 / 圆周运动
```

## Translation / Acceleration

必须全部做：

```text
□ 前后 X
□ 上下 Z
□ 左右 Y
```

重点：

> 要有加速和减速，不是匀速慢慢移动。

## Rotation

必须全部做：

```text
□ Roll  +
□ Roll  -

□ Pitch +
□ Pitch -

□ Yaw   +
□ Yaw   -
```

不要只有 yaw。

## Combined Motion

再做：

```text
□ 8 字
□ 圆周
□ 平移 + Rotation
□ 多轴组合运动
```

最后的 8 字或圆周运动可以适当增大幅度。IMU 与 Mocap marker 安装位置较近时，增大 Rig 的轨迹幅度有助于提供更充分的观测激励。

### 动作要求

```text
平滑
连续
有明显激励
```

不要：

```text
猛甩
碰撞
敲击
突然停止
```

### 同时注意

```text
□ AprilGrid 始终在参与当前标定的 Camera 视野内并可检测
□ 图像没有严重 blur
□ Mocap marker 没有大量遮挡
□ IMU 没有数据 gap
```

### 最后

```text
□ 静止 3–5 s
```

### 时间

```text
约 60–90 s / 条
```

---

# ③ validation_01

录制方式：

```text
AprilGrid：保持场景中
Rig：正常运动
Camera + IMU + Mocap：全部记录
```

运动包含：

```text
□ 正常走动
□ 转身
□ Roll / Pitch
□ 上下移动
□ 多轴组合动作
```

但：

> 不需要完全复制 joint_dynamic 的动作。

最重要：

```text
validation_01
绝对不能参与 calibration optimization
```

它只用于检查标定是否真正正确。

---

# 每录完一条立即检查

不要等全部录完才检查。

### Camera

```text
□ 文件数量正常
□ Timestamp 单调递增
□ 无 duplicate timestamp
□ 无明显连续丢帧
□ 无严重 blur
□ AprilGrid 能正常检测
```

### IMU

```text
□ Rate 正常
□ Timestamp 正常
□ 无明显 gap
```

### Mocap

```text
□ Tracking 正常
□ 无大段 lost
□ Pose 连续
```

---

# 出现下面任何情况：重录

```text
□ Camera 大量丢帧
□ IMU 出现明显 gap
□ Mocap 大量 lost
□ AprilGrid 基本检测不到

□ static 数据只覆盖图像中心

□ 相邻 Camera 没有共同看到标定板

□ dynamic 数据只有 Yaw
□ 没有 Roll / Pitch
□ 没有 XYZ acceleration

□ 图像严重 motion blur
□ 标定板过曝
□ 标定板参数填错

□ Rig / Camera / IMU / marker 中途发生机械移动
```

原则：

> 数据有明显问题直接重录，不要指望后续 optimizer 把坏数据救回来。

---

# 五个包录完以后

必须得到：

```text
□ cam_static_01        QA PASS
□ cam_static_02        QA PASS

□ joint_dynamic_01     QA PASS
□ joint_dynamic_02     QA PASS

□ validation_01        QA PASS
```

并确认每个数据包均包含：

```text
□ mav0/
□ meta/manifest.yaml
□ meta/recorder_stats.json
□ meta/checksums.sha256
```

完成后才能进入正式标定。

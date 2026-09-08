# 多相机 + IMU + Mocap 时空标定数据采集与标定 SOP

**版本：v0.1**

**适用对象：多相机 + IMU + Mocap marker 刚性传感器 Rig**

**标定内容：**

- Camera intrinsics
- Camera–Camera 外参
- Camera–IMU 外参及时间偏移
- IMU–Mocap marker 外参及时间偏移
- 标定结果重复性及独立数据验证

---

# 1. 总体原则

整个标定流程遵循以下原则：

1. **原始数据只采集一次，尽可能完整保存。**
2. 原始数据统一保存为项目规定的 **扩展 EuRoC 格式**。
3. Kalibr、Basalt、CO-Calib、hand-eye 等工具需要的 ROS bag、CSV 或其他格式，全部从原始数据转换生成。
4. **禁止为了某个标定软件直接修改原始数据。**
5. Camera–Camera 与 Camera–IMU 使用不同运动方式采集。
6. Camera–IMU 与 IMU–Mocap 可以使用同一条动态标定数据。
7. 至少录制两条独立标定序列，并额外保留一条 **Validation 数据**。
8. Validation 数据不得参与参数优化。
9. 一个标定结果不能只看 reprojection error，必须同时检查：
   - 数据质量；
   - 标定残差；
   - 多次标定重复性；
   - 不同标定方法一致性；
   - held-out 数据验证结果。

Kalibr 官方推荐将相机静态标定与 Camera–IMU 动态标定分开：Camera calibration 数据可降低到约 4 Hz，而 Camera–IMU 数据典型使用约 20 Hz Camera + 200 Hz IMU，并要求充分激励 IMU 各轴。

---

# 2. 坐标系命名

整个项目统一使用以下坐标系名称：

| Frame | 含义 |
|---|---|
| `cam0` | 主相机 |
| `cam1` | 第 2 个相机 |
| `cam2` | 第 3 个相机 |
| `cam3` | 第 4 个相机 |
| `imu0` | IMU 测量坐标系 |
| `mocap0` | 固定在 Rig 上的 Mocap rigid body / marker frame |
| `mocap_world` | 动捕系统世界坐标系 |
| `aprilgrid` | AprilGrid 标定板坐标系 |

所有输出矩阵必须明确：

- `from_frame`
- `to_frame`
- 单位
- quaternion 顺序
- 时间偏移定义

**禁止只写 `Tbc`、`Tci`、`extrinsic` 而不解释方向。**

项目统一约定：

> `T_A_B` 表示：将 B 坐标系中的点变换到 A 坐标系。

即：

`p_A = T_A_B × p_B`

不同标定软件自己的矩阵定义可能不同，转换到项目最终配置文件时必须统一成上述定义。

---

# 3. 标准数据包格式

每一次独立录制对应一个 sequence，不允许把多次录制人工拼成一个数据包。

建议目录：

```text
sequence_name/
├── mav0/
│   ├── cam0/
│   │   ├── data/
│   │   │   ├── 1788....png
│   │   │   └── ...
│   │   └── data.csv
│   ├── cam1/
│   │   ├── data/
│   │   └── data.csv
│   ├── cam2/
│   │   ├── data/
│   │   └── data.csv
│   ├── cam3/
│   │   ├── data/
│   │   └── data.csv
│   ├── imu0/
│   │   └── data.csv
│   ├── mocap0/
│   │   └── data.csv
│   └── state_groundtruth_estimate0/
│       └── data.csv
│
└── meta/
    ├── manifest.yaml
    ├── aprilgrid.yaml
    ├── events.csv
    ├── recorder_stats.json
    └── checksums.sha256
```

不是每个 sequence 都必须包含全部 sensor。

例如 Camera-only 标定中：

```text
imu0/
mocap0/
```

可以为空或不存在。

但**如果采集系统能够方便地同步保存这些数据，建议仍然全部保存。**

Basalt 原生支持 `bag` 和 `euroc` dataset type，因此项目可以将 EuRoC-like 数据作为 canonical raw format，再针对 Kalibr 等工具生成派生数据。Kalibr 本身要求 ROS1 bag，但官方提供从图像文件和 IMU CSV 创建 bag 的工具。

---

# 4. 数据包命名

统一采用：

```text
YYYYMMDD_<rig_id>_<purpose>_<index>
```

例如：

```text
20260908_rig01_cam_static_01
20260908_rig01_cam_static_02

20260908_rig01_joint_dynamic_01
20260908_rig01_joint_dynamic_02

20260908_rig01_validation_01
```

禁止使用：

```text
test1
test_new
final
final2
good
new_good
```

这类无法追踪的数据包名称。

---

# 5. 一套完整标定至少需要哪些数据

推荐每次 Rig 完整标定至少录下面 **5 个包**：

| 数据包 | 用途 | 是否参与最终标定 |
|---|---|---:|
| `cam_static_01` | Camera intrinsic + Camera–Camera | 是 |
| `cam_static_02` | Camera 标定重复性检查 | 是/对比 |
| `joint_dynamic_01` | Camera–IMU + IMU–Mocap | 是 |
| `joint_dynamic_02` | 第二条独立动态数据，检查重复性 | 是/对比 |
| `validation_01` | 独立验证最终标定结果 | **否** |

另外建议长期维护：

```text
imu_static_xx
```

用于 Allan deviation / IMU noise 参数估计。

Kalibr Camera–IMU 标定需要 IMU noise density 和 bias random walk 等参数，并建议提前获得 IMU 的误差模型。

---

# 6. 录制前检查

每次正式录制前必须逐项确认。

## 6.1 机械结构

- Camera、IMU、Mocap marker 已完全固定。
- 螺丝已经锁紧。
- Camera focus 已调好并固定。
- 镜头不得在不同数据包之间重新调焦。
- Mocap marker 不允许移动。
- Camera 支架角度不允许调整。
- IMU PCB/模组不允许重新安装。

**只要任何刚性关系发生变化，之前的外参全部视为失效。**

---

## 6.2 AprilGrid

确认：

- Tag 行数、列数正确；
- `tagSize` 正确；
- `tagSpacing` 正确；
- 实际打印尺寸与配置一致；
- 标定板平整，无明显弯曲；
- 最好固定在刚性平板上；
- 场景中不能出现其他可能被误识别的 AprilTag。

Kalibr 推荐使用 AprilGrid，因为允许标定板部分出画面，而且不存在棋盘格姿态翻转歧义。

**注意：标定板尺寸错误会直接造成平移尺度错误。**

因此不要只根据“Tag6-500-55”之类型号猜尺寸，必须测量或确认供应商参数后写入 `aprilgrid.yaml`。

---

# 7. 相机参数设置

正式标定前固定：

- resolution
- FPS
- exposure
- gain
- focus
- ISP 设置
- 图像裁剪方式
- 图像 resize/downsample 方法

原则：

### 推荐

稳定室内照明下尽量使用：

- 固定 exposure；
- 固定 gain；
- 固定 focus；
- AWB 可关闭或锁定；
- 避免曝光过程中参数持续变化。

标定板：

- 白色区域不能大面积过曝；
- 黑色 tag 必须有足够对比度；
- 快速运动过程中不能出现明显 motion blur。

Camera–IMU 动态标定尤其应优先保证短曝光。

---

# 8. 时间戳检查

正式采集前至少检查一次：

### Camera

检查：

- FPS 是否正确；
- 是否丢帧；
- timestamp 是否严格递增；
- 是否存在 duplicate timestamp；
- Camera 之间是否同步。

### IMU

检查：

- 实际频率；
- timestamp interval 分布；
- 是否存在 burst；
- 是否存在几十毫秒级 gap；
- 是否存在系统时间戳代替 sensor timestamp。

Kalibr 官方特别提醒检查 IMU timestamp DT；数据若表现为一批 IMU 消息瞬间发布，然后中间出现明显 gap，会严重影响 Camera–IMU 标定。

### Mocap

检查：

- rigid body 是否持续 tracked；
- timestamp 是否连续；
- 是否存在 lost tracking；
- Mocap PC 与采集 PC 的时钟关系。

如果 Camera/IMU 与 Mocap 在不同电脑：

优先级：

```text
硬件同步 / 同一硬件时钟
        ↓
PTP
        ↓
稳定网络时钟同步
        ↓
普通 NTP
```

即使后续算法能够估计 constant time offset，也不代表可以接受明显的 clock drift。

---

# 9. 数据包 A：Camera–Camera 标定

## 9.1 数据包

```text
cam_static_01
cam_static_02
```

主要用于：

- 每个 Camera intrinsic；
- distortion 参数；
- Camera–Camera rigid extrinsic。

---

# 10. Camera–Camera 采集方式

## Rig

**Rig 固定不动。**

## AprilGrid

**人移动 AprilGrid。**

这也是 Kalibr 官方推荐的 camera calibration 采集方式。用于离线标定时可将相机流降采样到约 4 Hz，以减少大量近似重复图像。

但项目规定：

> **采集时仍保存完整原始帧率，4 Hz 只在生成标定输入数据时进行抽帧。**

不要在数据源阶段丢掉原始图像。

---

# 11. Camera–Camera 动作

每个 Camera 都必须覆盖：

```text
中心
上
下
左
右
左上
右上
左下
右下
```

即标定板角点不能永远只出现在画面中央。

尤其是鱼眼/超广角相机：

> **图像边缘的数据比一直在中心移动标定板重要得多。**

建议包含：

### 距离变化

至少：

- 近；
- 中；
- 远。

### 姿态变化

标定板应包含：

- 正视；
- 向上倾斜；
- 向下倾斜；
- 左倾；
- 右倾；
- roll 旋转。

不要全部保持正对相机。

---

# 12. 多 Camera 特别要求

Camera–Camera extrinsic 的核心不是：

> “每个 Camera 都拍到很多板。”

而是：

> **Camera 之间必须建立足够的共同观测。**

例如四目：

```text
cam0 ↔ cam1
cam1 ↔ cam2
cam2 ↔ cam3
cam3 ↔ cam0
```

至少必须形成一个连通的 Camera graph。

因此需要专门让 AprilGrid 在相邻 Camera 的 overlap FOV 中移动。

对于超广角四目系统，应特别检查：

- cam0/cam1 同时看到板；
- cam1/cam2 同时看到板；
- cam2/cam3 同时看到板；
- cam3/cam0 同时看到板。

如果相机时间同步尚未完全确认，Camera calibration 时建议：

> 移动 → 停住约 0.5–1 s → 再移动。

不要快速连续挥动标定板。

否则 Camera 之间存在毫秒级时间差时，运动中的板会被不同 Camera 看到在不同位置，从而污染 Camera–Camera extrinsic。

---

# 13. Camera–Camera 推荐录制时长

每条约：

**1–2 min**

比“录 10 min 然后全部送进标定器”更重要的是：

- 图像覆盖；
- 距离变化；
- board orientation；
- Camera overlap。

不建议用“至少 100 张/Camera”作为唯一判断标准。

100 张高度重复的图：

> 不如 40–60 张空间分布良好的图。

---

# 14. Camera–IMU + IMU–Mocap 动态数据

对应：

```text
joint_dynamic_01
joint_dynamic_02
```

这一条数据同时记录：

```text
cam0
cam1
cam2
cam3
imu0
mocap0
```

它同时用于：

### Camera–IMU

估计：

```text
Camera ↔ IMU rotation
Camera ↔ IMU translation
Camera ↔ IMU time offset
```

### IMU–Mocap

估计：

```text
IMU ↔ Mocap marker rotation
IMU ↔ Mocap marker translation
Mocap/rig 时间关系
```

Basalt 的 Camera + IMU + Mocap calibration 正是针对这种数据，并提供 Camera–IMU 初始化、Camera time offset refinement、Mocap 初始化及 Mocap optimization。

---

# 15. Dynamic 数据的场景布置

这一次和 Camera calibration 相反。

## AprilGrid

**固定不动。**

最好固定在：

- 墙上；
- 刚性支架；
- Mocap 场景中的稳定位置。

录制过程中禁止移动。

## Rig

**人拿着 Rig 运动。**

## Mocap marker

固定在 Rig 上，并确保：

> 整条序列基本都可以被 Mocap 稳定追踪。

---

# 16. Camera–IMU 动作要求

Camera–IMU 标定最重要的不是走很大范围，而是：

> **充分 excitation。**

必须包含：

### Rotation

分别绕：

```text
X / roll
Y / pitch
Z / yaw
```

运动。

不能只有 yaw。

### Translation / acceleration

分别产生：

```text
X
Y
Z
```

方向的加减速。

不能只是匀速平移。

Kalibr 明确建议 Camera–IMU 标定时充分激励所有 IMU rotation 和 acceleration axes，并避免突然碰撞或冲击。

---

# 17. 推荐的一套动态动作

动作示范参考：[Kalibr 推荐的 Camera–IMU 标定序列（5:03 起）](https://www.youtube.com/watch?v=puNXsnrYWTY&t=303s)。视频中的核心动作落实为下面的三轴旋转、三轴加减速和多轴组合运动。

Basalt Camera–IMU–Mocap 标定序列参见[本地示例视频](../../artifacts/calibration/20260908_121012_basalt_cam_imu_mocap_sequence.webm)。示例动作顺序为：

```text
前后 → 上下 → 左右 → roll → pitch → yaw → 8 字 / 圆周运动
```

全过程保持 AprilGrid 在相机视野内。最后的 8 字或圆周运动可以适当增大幅度；当 IMU 与 Mocap marker 的安装位置较近时，增大 Rig 的运动轨迹幅度有助于提供更充分的观测激励，但仍须保证 AprilGrid 可检测且 Mocap marker 稳定可见。

建议一条数据约 **60–90 s**。

### 0–5 s

Rig 静止。

### 5–20 s：Rotation

依次：

```text
roll ±
pitch ±
yaw ±
```

动作连续、平滑。

不要猛甩。

### 20–35 s：Translation

前后：

```text
+X / -X
```

左右：

```text
+Y / -Y
```

上下：

```text
+Z / -Z
```

需要明显加速和减速。

### 35–60 s：组合运动

执行：

- 8 字；
- 圆周；
- 前后 + rotation；
- 上下 + rotation；
- 多轴组合。

### 60–75 s

适当提高 angular velocity / acceleration。

但前提是：

> AprilGrid 仍然能稳定检测，图像没有严重 motion blur。

### 最后 3–5 s

重新静止。

---

# 18. 动态标定过程中 Camera 的要求

目标不是让标定板 100% 时间处于 Camera 中心。

应该让标定板：

- 在多个 Camera 中出现；
- 在不同画面位置出现；
- 在不同距离出现；
- 有一定斜视角。

但需要保证存在足够数量的有效 AprilGrid detection。

如果动作导致：

```text
80% 时间 AprilGrid 都看不到
```

即使 IMU excitation 很丰富，数据也没有意义。

---

# 19. Mocap 特别要求

Dynamic sequence 中：

- Mocap rigid body 不允许修改；
- marker 不允许有松动；
- Mocap tracking 不应频繁 lost；
- 不要用严重 marker 遮挡的动作；
- Mocap world frame 在整个标定过程中保持不变。

建议在 `events.csv` 中记录：

```text
START
STATIC_BEGIN
ROLL
PITCH
YAW
XYZ_TRANSLATION
COMBINED
STATIC_END
STOP
```

方便后续快速检查异常。

---

# 20. Validation 数据

另录：

```text
validation_01
```

这是整个流程非常重要的一条数据。

原则：

> **绝对不能拿它重新优化 calibration。**

用途只有：

> 检查已经得到的 calibration 是否真的有效。

动作应与 calibration sequence 不完全相同。

例如包含：

- 正常手持走动；
- 转身；
- 上下运动；
- 大范围 yaw；
- roll/pitch；
- 组合运动。

建议同时保存：

```text
Camera
IMU
Mocap
```

如果条件允许，AprilGrid 仍放在场景中，可用于独立 reprojection / pose validation。

---

# 21. Raw Data QA

数据录完以后，**先验数据，不要立刻跑标定。**

每个 sequence 至少检查以下内容：

| 检查项                   | 要求               |
| --------------------- | ---------------- |
| Timestamp             | monotonic        |
| Duplicate timestamp   | 0                |
| Camera dropped frames | 无明显连续丢帧          |
| IMU gap               | 无异常大 gap         |
| Mocap lost tracking   | 尽量无              |
| Image blur            | 无大范围严重 blur      |
| Over-exposure         | 标定板无明显大面积饱和      |
| AprilGrid detection   | 数量充足             |
| Image coverage        | 中心与边缘均覆盖         |
| Camera overlap        | Camera graph 连通  |
| 文件数量                  | data.csv 与实际文件一致 |
| checksum              | 必须生成             |

存在严重问题：

> 直接重新录制。

不要指望“标定软件优化一下就好了”。

---

# 22. `manifest.yaml` 至少记录什么

建议：

```yaml
dataset_version: 1

sequence:
  name: 20260908_rig01_joint_dynamic_01
  purpose: cam_imu_mocap_calibration
  operator: xxx

rig:
  id: rig01

cameras:
  cam0:
    serial: xxx
    resolution: [1280, 800]
    fps: 20
    exposure_us: xxx
    gain: xxx
    ae: false
    awb: false

imu:
  imu0:
    model: xxx
    serial: xxx
    rate_hz: 200

mocap:
  rigid_body: rig01_marker
  rate_hz: xxx

time:
  camera_timestamp_source: sensor
  imu_timestamp_source: sensor
  mocap_timestamp_source: mocap_server
  host_clock_sync: NTP

target:
  type: aprilgrid
  config: meta/aprilgrid.yaml

notes: ""
```

Recorder 软件版本、firmware、Git commit 等也建议写入。

---

# 23. Camera–Camera 标定方法

建议至少使用两套方法交叉检查。

例如：

### 方法 A

CO-Calib

针对本项目的 220° 鱼眼，工程标定优先使用：

```text
EUCM (eucm-none)
```

`omni+radtan` 可用于交叉比较，但当前 VIO 算法暂不支持，因此不作为交付模型。

### 方法 B

Kalibr 或 Basalt 对应相机模型。

如果最终 Basalt 需要 DS，可以使用同一条 raw sequence 再做：

```text
DS calibration
```

---

# 24. 不同 Camera model 得到的 extrinsic 是否必须一样

不要求数值完全一致。

Camera intrinsic 和 extrinsic 在优化中存在耦合。

因此：

```text
omni+radtan
DS
EUCM
KB/equidistant
```

得到的 Camera–Camera extrinsic 会有小范围差异。

**禁止把不同模型得到的 extrinsic 直接取平均值。**

应该比较：

1. reprojection residual；
2. residual 在整个 image plane 上的分布；
3. image edge 是否存在系统误差；
4. cam–cam extrinsic repeatability；
5. held-out validation 数据表现；
6. 最终 VIO 表现。

最后选择：

> 与下游算法兼容，同时 held-out validation 最好的 calibration。

---

# 25. Camera–IMU 标定

建议至少：

## 方法 A：Kalibr

使用：

```text
cam_static
```

已经得到的 Camera intrinsics + cam–cam extrinsic。

然后输入：

```text
joint_dynamic_01
```

估计：

```text
T_cam_imu
camera–imu time offset
```

Kalibr 使用连续时间 spline 做 Camera–IMU spatial + temporal calibration。

重点检查：

- reprojection；
- gyro residual；
- accel residual；
- estimated time offset；
- IMU timestamp DT；
- convergence。

---

# 26. Camera–IMU Basalt 交叉验证

对于 Basalt-compatible camera model，可以用同一条 raw dynamic data：

```bash
basalt_calibrate_imu \
    --dataset-path <dataset> \
    --dataset-type euroc \
    ...
```

进行独立标定。

Basalt 文档建议 Camera–IMU calibration 中通常关闭 intrinsic optimization；Camera time offset 也应在整体 optimization 已基本收敛后再进行 refinement。

因此不要一开始：

```text
intrinsic
extrinsic
time offset
IMU scale
mocap
```

全部同时打开。

---

# 27. IMU–Mocap 主标定方法

推荐：

## 方法 A：Basalt joint calibration

使用：

```text
Camera observation
IMU
Mocap pose
```

共同建立连续时间轨迹。

在已经完成：

```text
Camera calibration
Camera–IMU initialization
```

之后：

```text
init_mocap
→ opt_mocap
```

得到 Mocap marker 与 IMU 的 rigid transformation。

Basalt 官方 Camera + IMU + Mocap 流程明确提供 `init_mocap`、`opt_mocap` 以及 `save_mocap_calib`。

---

# 28. IMU–Mocap 独立验证：Hand-eye

推荐把 hand-eye 作为独立第二方法。

准备两条 trajectory：

### Trajectory A

由 Mocap 得到：

```text
mocap_world → mocap0
```

### Trajectory B

由 AprilGrid Camera pose 得到：

```text
aprilgrid → camera
```

再利用已经得到的：

```text
camera ↔ imu
```

转换成 IMU trajectory。

然后进行：

```text
time alignment
       ↓
hand-eye initialization
       ↓
batch refinement
```

ETH `hand_eye_calibration` 工具支持 timestamped pose trajectory、time-offset alignment、dual-quaternion hand-eye calibration 和后续 batch refinement。

其基本 trajectory 格式为：

```text
t, x, y, z, qx, qy, qz, qw
```

因此我们的 EuRoC mocap 数据可以非常方便地转换成该格式。

---

# 29. 不推荐的 IMU–Mocap 方法

以下方法：

```text
Mocap position
      ↓
一次微分
      ↓
velocity
      ↓
再次微分
      ↓
acceleration
      ↓
与 IMU acceleration 对齐
```

**不作为正式 extrinsic calibration 方法。**

原因：

二阶微分会明显放大：

- Mocap position noise；
- timestamp jitter；
- lost frame；
- interpolation error。

同时还受到：

- gravity；
- IMU bias；
- sensor axis；
- low-pass filtering；
- time offset

等因素影响。

因此它最多用于：

```text
coarse initialization
sanity check
debug
```

不能替代：

```text
pose-based hand-eye
```

或：

```text
continuous-time joint optimization
```

---

# 30. 标定结果必须做“重复性比较”

不要只跑：

```text
cam_static_01
joint_dynamic_01
```

然后直接发布结果。

至少比较：

```text
cam_static_01
vs
cam_static_02
```

以及：

```text
joint_dynamic_01
vs
joint_dynamic_02
```

---

# 31. Extrinsic 比较方法

假设两个 calibration 得到：

```text
T1
T2
```

计算：

```text
ΔT = inv(T1) × T2
```

然后分别报告：

### Translation difference

```text
|Δt|
```

单位：

```text
mm
```

### Rotation difference

```text
angle(ΔR)
```

单位：

```text
degree
```

最终报告不要直接比较：

```text
tx ty tz
roll pitch yaw
```

逐元素差值。

特别是 Euler angle 很容易受到表示方式影响。

---

# 32. 时间偏移比较

每次 calibration 都记录：

```text
dt_cam_imu
dt_imu_mocap
```

并明确：

> `timestamp_B_corrected = timestamp_B + dt`

还是相反。

禁止只记录：

```text
time offset = 0.003
```

却没有说明符号定义。

需要比较：

```text
dynamic_01 dt
dynamic_02 dt
```

如果同一硬件结构重复录制后 time offset 明显漂移：

首先检查：

- timestamp source；
- driver；
- buffering；
- exposure timestamp 定义；
- Camera/IMU clock；
- PC clock synchronization。

不要首先怀疑 optimizer。

---

# 33. 推荐内部工程验收指标

以下不是数学上的统一行业标准，而是建议作为第一版项目工程 gate，后续应根据实际 Rig 的多次统计再调整。

| 项目 | 推荐目标 | 明显异常 |
|---|---:|---:|
| Camera reprojection RMS | 尽量 <0.5 px | >1 px 需重点检查 |
| cam–cam rotation repeatability | <0.2° | >0.5° |
| cam–cam translation repeatability | <2–3 mm | >5 mm |
| cam–IMU rotation repeatability | <0.2–0.3° | >0.5° |
| cam–IMU translation repeatability | <3–5 mm | >10 mm |
| IMU–mocap rotation repeatability | <0.3° | >0.5–1° |
| IMU–mocap translation repeatability | <5 mm | >10 mm |

注意：

> reprojection RMSE 不能跨不同软件简单横向比较。

不同软件：

- residual 定义；
- robust kernel；
- corner weighting；
- outlier rejection

可能不同。

真正有价值的是：

> 同工具 repeatability + held-out validation。

---

# 34. Validation 怎么做

最终 calibration 完成以后，对：

```text
validation_01
```

执行验证。

## Camera

检查：

- AprilGrid reprojection；
- 尤其检查 image edge；
- 是否存在某个 Camera 单独偏差明显。

## Camera–Camera

通过已知 extrinsic：

- 把同一个 AprilGrid pose 跨 Camera 转换；
- 检查不同 Camera 对同一目标的 pose consistency。

## Camera–IMU

使用最终 calibration 跑 VIO / continuous trajectory：

检查：

- 初始化是否稳定；
- rotation 是否出现系统性偏差；
- 高频 motion 下是否异常；
- timestamp correction 后 residual 是否改善。

## IMU–Mocap

把 IMU trajectory 通过：

```text
T_mocap_imu
```

变换到 Mocap 系统。

然后比较：

```text
position residual
rotation residual
```

随时间是否存在：

- 常量偏差；
- rotation-dependent error；
- phase lag；
- gradual drift。

其中：

### 固定 translation error

优先怀疑 extrinsic translation。

### 姿态变化时 position error 呈周期变化

非常典型地提示：

```text
lever-arm / rotation extrinsic
```

错误。

### 两条曲线形状一致但存在时间平移

优先怀疑：

```text
time offset
```

。

---

# 35. 正式发布 calibration 前必须形成一个对比表

示例：

| 参数 | Method A | Method B | Repeat run | Final |
|---|---:|---:|---:|---:|
| cam0–cam1 ΔR | CO-Calib | Kalibr | run02 | xxx |
| cam0–cam1 Δt | CO-Calib | Kalibr | run02 | xxx |
| cam0–IMU ΔR | Kalibr | Basalt | run02 | xxx |
| cam0–IMU Δt | Kalibr | Basalt | run02 | xxx |
| cam–IMU dt | Kalibr | Basalt | run02 | xxx |
| IMU–mocap ΔR | Basalt | hand-eye | run02 | xxx |
| IMU–mocap Δt | Basalt | hand-eye | run02 | xxx |

并给出：

```text
ACCEPT
```

或：

```text
REJECT
```

结论。

---

# 36. 最终 calibration 文件

最终不要只保存某个软件输出的：

```text
camchain.yaml
calibration.json
```

而应该另外生成项目统一格式：

```text
canonical_calibration.yaml
```

至少包含：

```yaml
calibration_version: 1

reference_frame: imu0

cameras:

  cam0:
    model: ...
    intrinsics: [...]
    distortion: [...]
    T_imu_cam:
      [...]

  cam1:
    model: ...
    intrinsics: [...]
    distortion: [...]
    T_imu_cam:
      [...]

time_offsets:

  cam0_to_imu0:
    value_s: ...
    definition: "..."

mocap:

  frame: mocap0

  T_imu_mocap:
    [...]

  time_offset:
    value_s: ...
    definition: "..."
```

同时保存：

```text
原始工具输出
完整 command
软件 Git commit
输入 sequence 名称
标定日期
operator
validation report
```

保证半年以后还能完整复现。

---

# 37. 以下情况必须重新录制

出现任意一种：

- Camera 调过 focus；
- Camera 支架发生移动；
- IMU 重新安装；
- Mocap marker 重新安装；
- 标定板尺寸填写错误；
- 标定板明显弯曲；
- Camera 大面积过曝；
- motion blur 严重；
- IMU timestamp 有明显 burst/gap；
- Camera 大量丢帧；
- Mocap 大量 lost tracking；
- Camera overlap 不足；
- Camera coverage 明显只集中在中心；
- Dynamic sequence 只有 yaw、没有 roll/pitch；
- Dynamic sequence 基本没有 XYZ acceleration；
- 两次独立标定 extrinsic 差异明显。

原则：

> **重新录 2 分钟数据，通常比花几个小时尝试“救坏数据”更划算。**

---

# 38. 现场执行版 Checklist

## 开机后

- [ ] Rig 机械结构无变化
- [ ] Camera focus 锁定
- [ ] AprilGrid 参数正确
- [ ] Camera FPS 正确
- [ ] Exposure/Gain 正确
- [ ] IMU FPS 正确
- [ ] Mocap tracking 正常
- [ ] Timestamp 正常
- [ ] 磁盘空间足够

## Camera calibration

录：

```text
cam_static_01
cam_static_02
```

操作：

- [ ] Rig 固定
- [ ] 人移动 AprilGrid
- [ ] 每个 Camera 中心覆盖
- [ ] 每个 Camera 四边覆盖
- [ ] 每个 Camera 四角覆盖
- [ ] 近/中/远
- [ ] 多种 tilt
- [ ] Camera overlap 区域覆盖
- [ ] 每条约 1–2 min

## Camera–IMU–Mocap

录：

```text
joint_dynamic_01
joint_dynamic_02
```

操作：

- [ ] AprilGrid 固定
- [ ] Mocap marker 稳定 tracked
- [ ] roll
- [ ] pitch
- [ ] yaw
- [ ] X acceleration
- [ ] Y acceleration
- [ ] Z acceleration
- [ ] multi-axis motion
- [ ] 无明显 blur
- [ ] 每条约 60–90 s

## Validation

录：

```text
validation_01
```

- [ ] 与 calibration motion 不完全相同
- [ ] Camera + IMU + Mocap 全部记录
- [ ] **禁止加入 calibration optimization**

---

# 39. 最终推荐的标定链路

整个流程建议固定为：

```text
                    ┌─────────────────────┐
                    │   cam_static_01/02  │
                    └──────────┬──────────┘
                               │
                               ▼
              Camera intrinsics + Cam-Cam
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
                 ▼                           ▼
             CO-Calib                   Kalibr/Basalt
                 │                           │
                 └─────────────┬─────────────┘
                               │
                         结果一致性检查
                               │
                               ▼
                    固定 Camera calibration
                               │
                               ▼
                 joint_dynamic_01 / 02
                               │
               ┌───────────────┴────────────────┐
               │                                │
               ▼                                ▼
       Camera–IMU Kalibr                Basalt joint calibration
               │                                │
               └───────────┬────────────────────┘
                           │
                  Cam–IMU 一致性检查
                           │
                           ▼
                    固定 Cam–IMU
                           │
               ┌───────────┴────────────┐
               │                        │
               ▼                        ▼
       Basalt IMU–Mocap         Pose-based Hand-eye
               │                        │
               └───────────┬────────────┘
                           │
                  IMU–Mocap 一致性
                           │
                           ▼
                    validation_01
                           │
                           ▼
                     ACCEPT / REJECT
```

---

# 40. 最核心的四句话

只需要牢牢记住：

**Camera–Camera：Rig 不动，移动板，重点覆盖整个 FOV 和 Camera overlap。**

**Camera–IMU：板不动，移动 Rig，重点充分激励 roll / pitch / yaw + XYZ acceleration。**

**IMU–Mocap：和 Camera–IMU 同时录 Mocap，marker 必须刚性固定且持续 tracked。**

**标定数据和 Validation 数据必须分开；没有独立 Validation 的 calibration 不算完成。**

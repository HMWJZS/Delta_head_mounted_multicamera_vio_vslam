## P0–P4 数据采集、标定与 Ground Truth Pipeline

## 1. 目标

建立一套可重复使用的头戴多目视觉定位数据采集与标定流程，为后续 VIO/VSLAM 测试提供：

- 明确定义且可复现的相机 Layout；
- 经过质检的 Camera / IMU 数据流；
- 可靠的多目相机及 Camera–IMU 时空标定；
- 标准化的 EuRoC 风格原始数据集；
- 与 IMU/body frame 对齐并完成时间同步的 Mocap Ground Truth。

本阶段只负责 **P0–P4**：

```text
P0  Rig / Layout Definition
        ↓
P1  Sensor & Timestamp Qualification
        ↓
P2  Spatial + Temporal Calibration
        ↓
P3  RK3588 EuRoC Recorder
        ↓
P4  Mocap Ground Truth Generator
        ↓
P5  Algorithm Adapters
        ↓
P6  Unified Evaluator
        ↓
P7  Experiment Matrix + Report
```

后续 VIO/VSLAM 算法适配、实验设计及性能评测不在本文档范围内。

第一阶段硬件：

```text
OAK-4P-New
	4 × B033501
	1280 × 800
	OV9782
	RGB Global Shutter
	220° fisheye
	内置 IMU
RK3588 + SSD Recorder
```

---

# P0. Rig / Layout Definition

## P0.1 目标

把相机机械布局定义成一个**可编号、可恢复、可比较的实验配置**。

任何以下参数发生改变：

```text
camera position
yaw
pitch
roll
baseline
```

都视为一个新的 Layout，例如：

```text
L001
L002
L003
...
```

Layout 改变后，不能继续沿用旧 Layout 的标定结果。

---

## P0.2 坐标系

统一定义：

```text
I = IMU frame
C0~C3 = Camera frame
M = Mocap marker rigid-body frame
```

统一使用：

```text
T_A_B
```

表示：

> 将 B frame 中的坐标转换到 A frame。

即：

```text
p_A = T_A_B · p_B
```

后续所有数据、标定和 Ground Truth 均使用这一 convention。

IMU frame `I` 作为后续定位评测的统一 body frame。

---

## P0.3 Mocap marker 与机械结构

Mocap marker rigid body 必须与 Camera + IMU 固定在同一刚性结构上。

要求：

- marker 不安装在弹性长杆或活动相机支架上；
- 相机、IMU、marker 之间不能在正常头部运动下发生可测相对形变；
- 相机线材做好 strain relief，避免线材拉扯改变相机姿态；
- marker 布局保证常见头部姿态下具有良好的 Mocap 可见性。

相机安装工装需要满足：

- **可调**：position / yaw / pitch 等可以调整；
- **可量化**：调整量可以记录；
- **可重复**：具有刻度、定位孔、固定槽位或 CAD configuration；
- **可锁死**：快速转头、下蹲、跑跳过程中结构不能松动。

---

## P0.4 Layout metadata

每个 Layout 建议保存：

```text
layouts/
└── L001/
    ├── layout.yaml
    ├── layout.step
    ├── layout.png
    └── notes.md
```

`layout.yaml` 至少记录：

```yaml
layout_id: L001

camera:
  cam0:
    role: front_left
    nominal_position_mm: [...]
    yaw_deg: ...
    pitch_deg: ...
    roll_deg: ...

  cam1:
    ...

mocap_marker:
  rigid_body_name: ...

mechanical_version: V0.1
```

### P0-GATE

进入标定之前需要确认：

- 四相机固定可靠；
- Camera ID 与物理位置一一对应；
- Mocap marker 固定完成；
- 线材完成应力释放；
- Layout 已编号；
- CAD / 照片 / 参数已归档。

---

# P1. Sensor & Timestamp Qualification

## P1.1 目标

在进行标定和正式采集前，首先验证：

> Camera / IMU 输出的数据本身是否可靠。

重点检查四相机：

```text
FPS
timestamp monotonic
sequence continuity
duplicate timestamp
frame drop
inter-camera synchronization
```

针对每组同步图像统计：

```text
mean Δt
std Δt
p95 Δt
max Δt
frame drops
```

输出：

```text
camera_sync_stats.json
```

不能只依据设备标称同步精度，需要对实际采集数据进行验证。

---

## P1.2 Camera / IMU timestamp

Recorder 必须保存设备侧 timestamp，而不能只保存 host receive time。

Camera 至少保存：

```text
device timestamp
sequence number
```

IMU 每个 packet 至少保存：

```text
device timestamp
gyro xyz
acc xyz
```

需要确认：

- Camera timestamp 的定义；
- IMU timestamp 的定义；
- Camera 与 IMU 是否来自统一设备时钟；
- 数据读取、USB 传输及 host 调度是否会改变最终记录 timestamp。

---

## P1.3 Camera–IMU 时间偏移

统一定义：

```text
t_imu = t_cam + dt_cam_imu
```

第一阶段不直接假设：

```text
dt_cam_imu = 0
```

而是通过 Camera–IMU 时空标定实际估计，并通过多次独立标定检查稳定性。

重点关注：

```text
dt_cam_imu repeatability
```

如果多次结果出现明显毫秒级随机变化，应优先排查 timestamp pipeline，而不是继续进行后续 Benchmark。

---

## P1.4 Recorder Stress Test

正式进入 Mocap 场地前，RK3588 Recorder 应连续记录约：

```text
30–60 min
```

自动检查：

```text
camera FPS
camera frame count
inter-camera timestamp
IMU sample rate
IMU timestamp interval

duplicate timestamp
out-of-order timestamp
frame drop

USB throughput
SSD throughput

CPU load
memory
temperature
```

输出：

```text
recorder_qualification_report.json
```

同时需要确认最终实际记录的图像格式，例如 RAW / YUV / RGB / ISP output 等。

Benchmark 原始数据原则上应尽量避免 H.264/H.265 等有损视频压缩，以免把编码影响和 Sensor/Layout 本身影响混在一起。

### P1-GATE

要求：

- 四相机连续稳定工作；
- Camera–Camera timestamp 正常；
- IMU 输出频率和时间间隔正常；
- 无明显掉帧；
- 无 duplicate / out-of-order timestamp；
- RK3588 长时间记录稳定；
- 图像数据格式明确；
- device timestamp 定义明确。

---

# P2. Spatial + Temporal + IMU Calibration

## P2.1 每个 Layout 需要获得的参数

### Camera intrinsics

```text
intrinsics_i
distortion_i
```

### Camera extrinsics

统一表达为：

```text
T_I_C0
T_I_C1
T_I_C2
T_I_C3
```

### Camera–IMU temporal offset

```text
dt_cam_imu
```

### IMU noise parameters

至少包括：

```text
gyro_noise_density
gyro_random_walk

acc_noise_density
acc_random_walk
```

必要时进一步考虑：

```text
IMU scale
axis misalignment
```

此外还需要得到：

```text
T_I_M
```

用于后续 Mocap GT 转换。

---

## P2.2 标定要求

针对 220° 多鱼眼系统，标定不能只检查 overall reprojection RMS。

本项目优先使用 **EUCM**（`eucm-none`）作为工程相机模型。`omni+radtan` 可用于标定结果交叉比较，但当前 VIO 算法暂不支持，因此不作为交付模型。

至少需要同时检查：

```text
overall reprojection error
reprojection error vs image radius
```

尤其关注鱼眼边缘区域是否存在明显 systematic residual。

Camera–Camera / Camera–IMU 外参还需要检查物理合理性及重复性。

---

## P2.3 Repeatability

每个候选 Layout 建议至少进行 3 次独立标定：

```text
Calibration A
Calibration B
Calibration C
```

比较：

```text
intrinsic repeatability

extrinsic repeatability
    translation
    rotation

dt_cam_imu repeatability
```

第一阶段阈值作为 **Pipeline QA 指标** 使用，待首批数据完成后再根据实际结果冻结正式验收标准。

标定通过后冻结为版本，例如：

```text
CALIB_L001_V001
```

正式数据采集后不允许无记录地修改该标定版本。

### P2-GATE

至少确认：

```text
intrinsics PASS
camera-camera extrinsics PASS
camera-IMU extrinsics PASS
dt_cam_imu PASS
IMU noise model PASS
repeatability PASS
```

---

# P3. RK3588 EuRoC Recorder

## P3.1 原则

Recorder 的主要职责是：

> 忠实、稳定地记录 Sensor Measurement。

采集过程中尽量不要同时运行与数据记录无关的高负载算法，避免影响：

```text
timestamp
frame drop
USB transmission
SSD write
CPU scheduling
temperature
```

---

## P3.2 Dataset Format

采用扩展 EuRoC 格式：

```text
sequence/
└── mav0/
    ├── cam0/
    │   ├── data/
    │   └── data.csv
    ├── cam1/
    ├── cam2/
    ├── cam3/
    ├── imu0/
    │   └── data.csv
    ├── mocap0/
    │   └── data.csv
    └── state_groundtruth_estimate0/
        └── data.csv

sequence/
└── meta/
    ├── manifest.yaml
    ├── events.csv
    ├── recorder_stats.json
    └── checksums.sha256
```

---

## P3.3 Raw Mocap 与 Ground Truth 分离

```text
mocap0/
```

保存 Mocap 系统原始输出的 marker rigid-body trajectory。

```text
state_groundtruth_estimate0/
```

保存经过：

```text
Mocap–IMU time alignment
+
T_I_M transformation
```

得到的统一 IMU/body frame Ground Truth。

Raw Mocap 数据不能被覆盖。

---

## P3.4 Sequence ID

统一命名：

```text
<sensor>_<layout>_<scene>_<motion>_<repeat>
```

例如：

```text
OAK4P_L001_HOME_M03_R01
```

避免：

```text
test1
final_test
final_test2
```

---

## P3.5 Manifest

每条 sequence 至少记录：

```yaml
sequence_id:
sensor_id:
layout_id:
calibration_id:

scene:
motion:
repeat:

camera_fps:
imu_rate:

exposure_mode:
exposure_us:
gain:
white_balance:

recorder_version:
depthai_version:

start_time:
duration:
```

保证后续所有测试结果均可以反查至：

```text
Sensor
Layout
Calibration
Recorder Version
Capture Configuration
```

---

## P3.6 Event Marker

每条正式动作 sequence 建议包含：

```text
static
↓
initialization / excitation
↓
TEST_START
↓
actual motion
↓
TEST_END
↓
static
```

Recorder 支持生成：

```text
events.csv
```

例如：

```text
INIT_START
INIT_END
TEST_START
TEST_END
```

用于后续统一截取正式评测区间。

---

## P3.7 Dataset Validator

每条 sequence 采集完成后自动检查：

```text
image count
camera timestamp
camera sync

IMU rate
IMU gaps

mocap gaps
duration

corrupted images
disk checksum
```

输出 Dataset Validation 结果。

### P3-GATE

数据集 Validator PASS 后，才认为该条 sequence 采集有效。

---

# P4. Mocap Ground Truth Generator

## P4.1 目标

将 Mocap 系统输出的：

```text
T_W_M
```

转换成与后续定位结果统一的：

```text
T_W_I
```

核心需要解决两个问题：

```text
1. Spatial Alignment
   T_I_M

2. Temporal Alignment
   dt_mocap_imu
```

最终生成：

```text
state_groundtruth_estimate0/data.csv
```

原始：

```text
mocap0/data.csv
```

保持不变。

---

## P4.2 Mocap–IMU 时间偏移

需要估计：

```text
dt_mocap_imu
```

推荐使用具有充分旋转激励的数据，通过：

```text
Mocap rotation
→ angular velocity
```

与：

```text
IMU gyro
```

进行时间对齐。

不能直接假定：

```text
Mocap timestamp == IMU timestamp
```

---

## P4.3 Clock Drift

第一次系统测试需要使用一条较长动态 sequence。

分段估计：

```text
dt_mocap_imu
```

例如：

```text
0–2 min
2–4 min
4–6 min
...
```

判断 offset 是否随时间变化。

如果单一 offset 足够，则采用：

```text
t_corrected = t + dt
```

如果存在不可忽略的 clock drift，则需要考虑：

```text
t_corrected = a · t + b
```

或进一步采用 shared clock / trigger / PTP 等同步方案。

---

## P4.4 Ground Truth Qualification

Ground Truth 本身也存在误差来源：

```text
Mocap pose noise
+
marker rigid-body error
+
T_I_M calibration error
+
dt_mocap_imu error
+
rig flex
```

因此需要至少通过：

```text
static
slow rotation
fast rotation
```

检查：

```text
position noise
orientation noise
temporal alignment
trajectory continuity
```

### P4-GATE

Ground Truth 至少满足：

```text
Mocap coverage PASS
T_I_M PASS
time alignment PASS
clock drift PASS
GT continuity PASS
```

---

# 2. P0–P4 最终交付物

完成一个 Layout 后，应形成以下完整交付：

```text
Layout Definition
    ↓
Lxxx metadata / CAD / photo

Sensor Qualification
    ↓
camera_sync_stats.json
recorder_qualification_report.json

Calibration
    ↓
CALIB_Lxxx_Vxxx

Canonical Dataset
    ↓
Extended EuRoC Dataset
+ manifest
+ events
+ validator report

Ground Truth
    ↓
raw mocap trajectory
+
T_I_M
+
dt_mocap_imu
+
aligned T_W_I trajectory
```

以上 P0–P4 全部通过 Gate 后，该 Layout 即认为具备进入后续 VIO/VSLAM Benchmark 的条件。

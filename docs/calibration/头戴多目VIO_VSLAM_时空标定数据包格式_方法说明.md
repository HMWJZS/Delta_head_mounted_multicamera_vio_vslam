# 附录 A：标定数据包格式规范

## A.1 目标

所有用于：

- Camera intrinsic
- Camera–Camera extrinsic
- Camera–IMU extrinsic / time offset
- IMU–Mocap extrinsic / time offset
- VIO / VSLAM validation
- Mocap ground truth evaluation

的数据，统一保存为 **Extended EuRoC Format**。

原则：

> 采集程序只负责生成统一原始数据包；Kalibr ROS bag、Basalt dataset、hand-eye trajectory 等均由后处理脚本生成。

---

# A.2 标准目录结构

```text
<sequence_name>/
├── mav0/
│   ├── cam0/
│   │   ├── data/
│   │   │   ├── <timestamp_ns>.png
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

其中：

- `mav0/`：传感器原始测量；
- `meta/`：描述此次录制的数据和系统状态；
- 一个目录只对应一次连续录制；
- 禁止人工拼接两次独立录制。

---

# A.3 时间戳统一规定

所有 CSV 第一列统一为：

```text
timestamp_ns
```

定义：

> 从某一明确时间基准起的整数纳秒时间戳。

必须满足：

```text
int64
```

例如：

```text
1788839123456789000
```

禁止使用：

```text
1788839123.456789
```

这种浮点秒作为 canonical timestamp。

原因是浮点数在大 epoch timestamp 下可能损失精度。

---

# A.4 时间戳语义必须明确

对于每种传感器，`manifest.yaml` 必须记录 timestamp 的含义。

Camera 尤其需要注明：

```yaml
timestamp_reference: exposure_start
```

或：

```yaml
timestamp_reference: exposure_mid
```

或：

```yaml
timestamp_reference: exposure_end
```

不能只写：

```yaml
timestamp_source: camera
```

因为 Camera–IMU time offset 与曝光时刻定义直接相关。

IMU 也需要说明：

```yaml
timestamp_reference: measurement_time
```

Mocap 需要说明：

```yaml
timestamp_reference: mocap_measurement_time
```

---

# A.5 Camera `data.csv`

格式：

```csv
#timestamp [ns],filename
1788839123456789000,1788839123456789000.png
1788839123506789000,1788839123506789000.png
```

建议：

```text
filename = timestamp_ns + extension
```

保证文件名天然唯一且方便追踪。

不得使用：

```text
000001.png
000002.png
```

作为唯一信息来源。

---

# A.6 Camera 原始图像

标定数据优先保存：

```text
PNG
```

或者其他：

> 无损、未经视频编码的单帧图像。

不建议直接把：

```text
H.264
H.265
```

解码图像作为 Camera calibration 的 canonical raw data。

因为视频编码可能引入：

- compression artifact；
- sharpening；
- temporal prediction artifact；
- 色彩处理差异。

如果硬件采集系统只能输出视频：

> 同时保存原始视频及提取后的 calibration images，并在 manifest 中明确来源。

---

# A.7 IMU `data.csv`

统一采用 EuRoC 风格：

```csv
#timestamp [ns],w_RS_S_x [rad s^-1],w_RS_S_y [rad s^-1],w_RS_S_z [rad s^-1],a_RS_S_x [m s^-2],a_RS_S_y [m s^-2],a_RS_S_z [m s^-2]
1788839123456000000,0.0012,-0.0021,0.0007,0.12,-0.05,9.79
```

即：

| Column | 含义 | 单位 |
|---|---|---|
| timestamp | IMU measurement timestamp | ns |
| wx | angular velocity X | rad/s |
| wy | angular velocity Y | rad/s |
| wz | angular velocity Z | rad/s |
| ax | specific force X | m/s² |
| ay | specific force Y | m/s² |
| az | specific force Z | m/s² |

注意：

### Gyroscope

必须统一为：

```text
rad/s
```

禁止直接存：

```text
deg/s
```

### Accelerometer

必须统一为：

```text
m/s²
```

禁止使用：

```text
g
```

作为 canonical data。

---

# A.8 IMU 坐标系

必须明确：

```text
+x
+y
+z
```

分别指向哪里。

例如：

```yaml
imu:
  imu0:
    frame: imu0
    axis_convention:
      x: forward
      y: left
      z: up
```

实际定义以传感器硬件为准。

禁止仅依赖：

> “BMI088 默认坐标系应该是什么。”

最终 canonical 数据必须明确已经采用哪个坐标系。

---

# A.9 Mocap `data.csv`

推荐格式：

```csv
#timestamp [ns],p_RS_R_x [m],p_RS_R_y [m],p_RS_R_z [m],q_RS_w [],q_RS_x [],q_RS_y [],q_RS_z []
1788839123456000000,1.245,0.351,1.102,0.9998,0.003,-0.015,0.008
```

定义：

```text
T_mocap_world_mocap0
```

即 Mocap rigid body `mocap0` 在 Mocap world 中的 pose。

必须在 manifest 中明确：

```yaml
pose_definition: T_mocap_world_mocap0
```

---

# A.10 Quaternion 统一规定

Canonical format 统一使用：

```text
qw, qx, qy, qz
```

并在 CSV header 中明确。

禁止不同数据包混用：

```text
qx qy qz qw
```

和：

```text
qw qx qy qz
```

转换给 hand-eye 工具时，如果工具要求：

```text
qx qy qz qw
```

由转换程序负责转换。

不要修改 canonical raw data。

---

# A.11 Mocap tracking 状态

建议额外增加：

```text
tracking_valid
```

例如：

```csv
#timestamp [ns],px,py,pz,qw,qx,qy,qz,tracking_valid
...,1.2,0.3,1.1,0.99,...,1
```

其中：

```text
1 = valid
0 = lost / invalid
```

如果 Mocap 系统还提供：

- residual；
- marker count；
- tracking quality；

建议一并保存。

例如：

```text
marker_count
mean_marker_error_m
```

这样后续可以自动剔除差的 Mocap pose。

---

# A.12 `state_groundtruth_estimate0`

`mocap0` 和 `state_groundtruth_estimate0` 不建议混为一谈。

定义：

### `mocap0`

保存：

> Mocap 系统原始 rigid-body measurement。

### `state_groundtruth_estimate0`

保存：

> 根据最终标定结果，将 Mocap marker pose 转换后的 Rig/IMU ground truth。

例如：

```text
T_world_imu
```

因此：

```text
mocap0
```

属于 **raw measurement**；

```text
state_groundtruth_estimate0
```

属于 **derived data**。

首次采集数据包时后者可以不存在。

---

# A.13 `events.csv`

建议：

```csv
#timestamp_ns,event,note
1788839123000000000,START,
1788839128000000000,STATIC_BEGIN,
1788839133000000000,ROLL_BEGIN,
1788839143000000000,PITCH_BEGIN,
1788839153000000000,YAW_BEGIN,
1788839163000000000,TRANSLATION_X,
1788839173000000000,TRANSLATION_Y,
1788839183000000000,TRANSLATION_Z,
1788839193000000000,COMBINED,
1788839213000000000,STATIC_END,
1788839218000000000,STOP,
```

不是强制要求人工精确记录到毫秒。

主要用途：

- 快速检查动作覆盖；
- 定位异常；
- 自动生成报告；
- 选择某些 motion segment。

---

# A.14 `manifest.yaml`

建议正式固定为：

```yaml
format:
  name: extended_euroc
  version: 1.0

sequence:
  name: 20260908_rig01_joint_dynamic_01
  purpose: cam_imu_mocap_calibration
  created_at: "2026-09-08"
  operator: xxx

rig:
  id: rig01
  hardware_revision: rev_a

cameras:

  cam0:
    frame: cam0
    model: OAK_B033501
    serial: xxx

    resolution:
      width: 1280
      height: 800

    fps_nominal: 20

    image_format: png

    exposure:
      mode: manual
      exposure_us: xxx

    gain:
      mode: manual
      value: xxx

    awb:
      enabled: false

    timestamp:
      source: device
      clock_domain: oak_device_clock
      reference: exposure_mid

  cam1:
    frame: cam1
    model: OAK_B033501
    serial: xxx
    resolution:
      width: 1280
      height: 800
    fps_nominal: 20
    image_format: png
    timestamp:
      source: device
      clock_domain: oak_device_clock
      reference: exposure_mid

imu:

  imu0:
    frame: imu0
    model: xxx
    serial: xxx

    rate_hz_nominal: 200

    gyro_unit: rad/s
    accel_unit: m/s2

    timestamp:
      source: device
      clock_domain: oak_device_clock
      reference: measurement_time

mocap:

  mocap0:
    frame: mocap0
    world_frame: mocap_world

    rigid_body_name: rig01_marker

    rate_hz_nominal: 200

    pose_definition: T_mocap_world_mocap0

    quaternion_order: wxyz

    timestamp:
      source: mocap_server
      clock_domain: mocap_pc_clock
      reference: measurement_time

target:

  type: aprilgrid
  config: meta/aprilgrid.yaml

clock_sync:

  camera_imu:
    mechanism: hardware_shared_clock

  sensor_pc_to_mocap_pc:
    mechanism: ntp

software:

  recorder:
    repository: xxx
    git_commit: xxx

  firmware:
    version: xxx

notes: ""
```

---

# A.15 `recorder_stats.json`

由 recorder 自动生成，不建议人工填写。

至少包含：

```json
{
  "duration_s": 76.42,

  "cam0": {
    "frames": 1528,
    "mean_rate_hz": 19.995,
    "dropped_frames": 0,
    "duplicate_timestamps": 0,
    "max_dt_ms": 50.8
  },

  "imu0": {
    "samples": 15283,
    "mean_rate_hz": 199.98,
    "max_dt_ms": 6.1
  },

  "mocap0": {
    "samples": 15281,
    "tracking_invalid": 3,
    "tracking_valid_ratio": 0.9998
  }
}
```

后续 QA 脚本首先检查这个文件。

---

# A.16 `checksums.sha256`

每个数据包完成后自动运行 checksum。

例如：

```text
sha256sum mav0/cam0/data/*.png
sha256sum mav0/imu0/data.csv
...
```

最终形成：

```text
meta/checksums.sha256
```

目的：

> 数据从采集机拷贝到服务器后可以确认文件没有损坏或漏传。

---

# A.17 数据状态定义

建议一个 sequence 只有以下三种状态：

```text
RAW
QA_PASSED
REJECTED
```

而不要使用：

```text
good
better
final
```

例如在 manifest 中：

```yaml
qa:
  status: QA_PASSED
```

---

# A.18 严格区分 Raw 与 Derived Data

建议规定：

```text
sequence/
```

中的 `mav0`：

> 永远只保存 canonical measurement。

派生结果放在：

```text
derived/
```

例如：

```text
derived/
├── rosbag/
│   └── kalibr.bag
├── kalibr/
├── cocalib/
├── basalt/
├── handeye/
└── validation/
```

因此整个数据包进一步可以是：

```text
sequence/
├── mav0/
├── meta/
└── derived/
```

其中：

```text
mav0 + meta
```

不可修改。

`derived` 可以随时重新生成。

---

# A.19 推荐数据处理链

```text
OAK / Camera / IMU / Mocap
            │
            ▼
      Recorder
            │
            ▼
    Extended EuRoC
      canonical raw
            │
      ┌─────┼───────────────┐
      │     │               │
      ▼     ▼               ▼
   ROS bag Basalt       hand-eye CSV
      │     │               │
      ▼     ▼               ▼
   Kalibr Basalt        hand-eye
```

任何新增算法都从：

```text
canonical raw
```

生成自己的输入格式。

---

# A.20 数据包验收原则

负责标定的同事提交数据时，不提交：

> “一个 bag”。

而应提交：

```text
20260908_rig01_cam_static_01/
20260908_rig01_cam_static_02/
20260908_rig01_joint_dynamic_01/
20260908_rig01_joint_dynamic_02/
20260908_rig01_validation_01/
```

每个均具有：

```text
mav0/
meta/
```

并通过自动 QA。

这样之后：

- 换 Camera model；
- 换标定软件；
- 重新估 time offset；
- 重跑 hand-eye；
- 增加新 VIO；

都不需要重新寻找原始 ROS bag。
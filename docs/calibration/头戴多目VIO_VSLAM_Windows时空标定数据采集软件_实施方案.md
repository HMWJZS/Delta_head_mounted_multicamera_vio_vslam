# 头戴多目 VIO-VSLAM Windows 时空标定数据采集软件实施方案

> 状态：设计要求 v0.1
>
> 运行平台：64 位 Windows 10/11
>
> 适用硬件：OAK-4P 四相机、板载 IMU、刚性安装的 Mocap marker
>
> 目标：采集 Camera + IMU + Mocap 时空标定所需的可追溯原始数据

## 1. 依据与定位

本软件的设计以以下项目文档为准：

- [标定现场采集动作清单](头戴多目VIO_VSLAM_标定现场采集_动作清单.md)
- [多相机、IMU 与 Mocap 时空标定现场操作手册](头戴多目VIO_VSLAM_多相机IMU与Mocap时空标定_现场操作手册.md)
- [时空标定数据包格式](头戴多目VIO_VSLAM_时空标定数据包格式_方法说明.md)
- [标定与数据采集测试方案](头戴多目VIO_VSLAM_标定与数据采集_测试方案.md)

软件只负责：

1. Windows 主机上的传感器连接、预检和同步采集；
2. 引导操作者完成规定动作；
3. 无损保存原始数据与时间信息；
4. 自动生成采集统计、校验和及初步 QA 结果。

软件不负责运行 CO-Calib、Kalibr、Basalt、Hand-eye 或修改标定结果。这些工具的输入必须由采集后的 canonical raw data 转换生成。

## 2. v1 交付范围

v1 必须支持一次完整标定任务中的五条独立 sequence：

| 模式 | 默认名称后缀 | 必需传感器 | 现场动作 | 用途 |
| --- | --- | --- | --- | --- |
| Camera 静态 1 | `cam_static_01` | cam0..cam3 | Rig 固定，移动 AprilGrid | 相机内参与 Camera–Camera 外参 |
| Camera 静态 2 | `cam_static_02` | cam0..cam3 | 独立重复上一流程 | 相机标定重复性 |
| 联合动态 1 | `joint_dynamic_01` | cam0..cam3、imu0、mocap0 | AprilGrid 固定，移动 Rig | Camera–IMU 与 IMU–Mocap 标定 |
| 联合动态 2 | `joint_dynamic_02` | cam0..cam3、imu0、mocap0 | 独立重复上一流程 | 动态标定重复性 |
| 独立验证 | `validation_01` | cam0..cam3、imu0、mocap0 | 不完全重复标定动作 | 只验证，不参与优化 |

软件还应提供独立的 `imu_static` 模式，用于长时间静止采集和 Allan deviation。它不计入上述五包。

每次点击开始只能创建一条新的连续 sequence。软件不得把两次录制自动拼接为一个数据包，也不得覆盖已有目录。

## 3. 固定工程约定

### 3.1 传感器与坐标系

固定命名：

```text
cam0 = CAM_A
cam1 = CAM_B
cam2 = CAM_C
cam3 = CAM_D
imu0 = OAK-4P 板载 IMU 测量坐标系
mocap0 = 固定在 Rig 上的 Mocap rigid-body frame
mocap_world = 动捕系统世界坐标系
```

软件必须在界面和 `manifest.yaml` 中显示并记录物理设备到逻辑 frame 的映射，不允许采集后重新排序。

### 3.2 图像模型与原始数据

220° 鱼眼的工程标定模型为 `eucm-none`，但采集软件不得对图像做 EUCM 去畸变。软件必须保存未经 resize、crop、去畸变和有损视频编码的原始单帧图像；`eucm-none` 只作为目标标定模型写入 metadata。

### 3.3 变换与时间偏移

项目统一使用：

```text
T_A_B：将 B frame 中的点变换到 A frame
```

采集软件不计算最终外参，但输出中必须明确 frame 名称、坐标轴、单位、四元数顺序和时间戳语义，禁止只写含义不明的 `Tbc`、`extrinsic` 或 `time offset`。

## 4. 硬件与运行前提

### 4.1 Windows 主机

- 64 位 Windows 10/11；
- USB 3.x 控制器、接口和合格数据线；
- 本机 SSD，不允许把正式采集直接写到网络盘；
- 采集期间关闭睡眠、自动更新、实时同步和明显占用磁盘的程序；
- 软件启动时检查输出盘剩余容量和持续写入能力。

四路 1280×800 灰度图像在 30 Hz 下的未压缩数据率约为 123 MB/s。软件必须按所选时长估算最坏空间，并至少预留 10% 余量。

### 4.2 Camera 与 IMU

- 四路相机固定为 1280×800、30 Hz；
- 四路使用同一个持续外部 FSIN；
- 传感器侧 FSIN 电平和引脚以实际 OAK-4P 板卡为准，当前目标为 1.8 V 上升沿；
- 四路相机使用无损灰度 PNG；
- IMU 保存 RAW gyro 和 accelerometer，目标频率为 400 Hz；
- Camera 与 IMU 必须使用设备测量时间，不得以 USB 到达时间替代。

没有持续 FSIN、USB 不是 SuperSpeed、相机映射不完整或设备预检不通过时，正式五包模式不得开始录制。

### 4.3 Mocap

正式实现前必须确定实际 Mocap 品牌、SDK/网络接口、rigid body 名称、输出频率、pose 定义和源时间戳格式。

v1 只实现已确定的一个 Mocap 接口，不建设通用插件系统。无论底层接口为何，采集侧必须得到：

```text
source_timestamp
position_m
quaternion
tracking_valid
```

若系统可提供，还必须保存：

```text
marker_count
mean_marker_error_m
tracking_quality
```

`joint_dynamic_*` 和 `validation_01` 中，Mocap 未连接、rigid body 名称不匹配或 tracking 无效时必须禁止开始或立即显示录制无效。

## 5. 时钟与同步要求

### 5.1 原则

软件必须保留每个传感器的原始测量时间戳及其 clock domain，不能把 host receive time 冒充 sensor timestamp，也不能在采集时静默改写时间轴。

至少区分：

```text
oak_device_clock
windows_monotonic_clock
windows_utc_clock
mocap_clock
```

### 5.2 Camera–Camera

- 使用同一 FSIN 脉冲触发四路相机；
- 以真实设备时间戳配组，不使用各相机独立 sequence number 跨相机关联；
- 每组保存四路时间戳和 `max_skew_us`；
- 严格门槛为组内最大偏差不超过 1000 µs；
- 超门槛、缺帧、重复时间戳和乱序必须计数并显示。

### 5.3 Camera/IMU–Mocap

优先使用 PTP 或同等稳定的硬件/网络时钟方案；普通 NTP 只能作为降级方案并必须在 manifest 中标记。

软件必须周期性记录跨 clock domain 的对应关系，至少包含：

```text
sensor/source timestamp
Windows monotonic timestamp
Windows UTC timestamp
host receive timestamp
```

这些数据用于离线估计 `dt_cam_imu`、`dt_imu_mocap` 和 clock drift。网络接收时间只能用于诊断，不能替代 Mocap measurement time。

## 6. 软件工作流与状态机

### 6.1 完整任务向导

主界面按固定顺序显示：

```text
设备与 Rig 配置
  → 预检
  → cam_static_01
  → cam_static_02
  → joint_dynamic_01
  → joint_dynamic_02
  → validation_01
  → 完整性汇总
```

操作者可以退出后继续未完成的任务，但已完成的 sequence 只读，不能继续追加。

### 6.2 单条 sequence 状态

```text
IDLE
  → PREFLIGHT
  → ARMED
  → RECORDING
  → FINALIZING
  → QA_PASSED 或 REJECTED
```

- `PREFLIGHT`：检查硬件、参数、磁盘和数据流；
- `ARMED`：传感器稳定、等待操作者开始；
- `RECORDING`：只追加写入；
- `FINALIZING`：排空队列、关闭文件、生成统计和 checksum；
- `QA_PASSED`：所有硬门槛通过；
- `REJECTED`：数据保留，但不得作为正式标定输入。

程序异常退出后，不得把未完成目录标为成功。下次启动应识别未完成 sequence，允许只读检查和重新生成统计，但不得把新数据追加到旧 sequence。

## 7. 操作界面要求

v1 使用现有 Python、DepthAI、OpenCV 和 PyInstaller 路线，不新增 Web 服务、数据库或云端依赖。

### 7.1 任务配置

开始完整任务前必须填写或确认：

- 输出根目录；
- `rig_id`、hardware revision 和 operator；
- OAK 设备序列号及 `cam0..cam3` 映射；
- Mocap rigid body 名称；
- AprilGrid `tagRows`、`tagCols`、`tagSize`、`tagSpacing`；
- exposure、ISO、focus、FPS、IMU rate；
- Camera、IMU、Mocap 的 timestamp source、reference 和 clock domain；
- Windows 与 Mocap 主机的时钟同步方式。

上一次配置可以作为默认值，但开始前必须重新确认。五包之间不得改变焦点、曝光策略、图像处理、相机映射或 Rig 配置。

### 7.2 实时监控

录制界面至少显示：

- 四路 2×2 实时灰度预览及固定相机编号；
- 每路 FPS、设备 timestamp、sequence number、队列深度和丢帧数；
- 当前相机组 `max_skew_us` 及 FSIN PASS/FAIL；
- IMU gyro/accel 频率、最新间隔、最大 gap 和三轴活动状态；
- Mocap rigid body、频率、tracking 状态、invalid 数和最长失踪时间；
- 当前数据包名称、录制时长、写盘速率、剩余空间和估计可录时长；
- AprilGrid 检测状态和当前动作提示；
- 已记录事件及当前 QA 错误。

预览允许缩放显示，但写盘必须始终使用原始分辨率图像。界面卡顿不得阻塞采集线程。

### 7.3 操作控制

只提供必要操作：

```text
开始
安全停止
废弃本条并标记 REJECTED
记录动作事件
退出
```

不得提供“覆盖已有数据”“继续写入已完成数据包”或“忽略丢帧并标为成功”的按钮。

## 8. 模式化动作引导

### 8.1 `cam_static_01/02`

界面提示 Rig 固定、移动 AprilGrid，并追踪：

- 每路图像中心、四边和四角覆盖；
- 近、中、远距离；
- 正视、左右倾、上下倾和 roll；
- 相邻相机共视；所有相机必须形成连通观测图。

移动标定板后提示停稳 0.5–1 s，再记录有效观测。建议时长为每条 1–2 min；软件保存完整 30 Hz 原始数据，离线工具再抽帧。

### 8.2 `joint_dynamic_01/02`

界面按以下顺序提示动作并写入 `events.csv`：

```text
静止 3–5 s
→ 前后
→ 上下
→ 左右
→ roll 正/负
→ pitch 正/负
→ yaw 正/负
→ 8 字 / 圆周 / 多轴组合
→ 静止 3–5 s
```

全过程要求：

- AprilGrid 在参与当前标定的相机视野内并可检测；
- Mocap marker 持续 tracked；
- 平移包含明显加速和减速；
- 动作平滑连续，无碰撞、敲击、猛甩和突然停止；
- 图像无严重 motion blur。

最后的 8 字或圆周运动允许适当增大幅度。IMU 与 Mocap marker 安装位置较近时，应提示增大 Rig 轨迹幅度以增强观测，但不能以丢失 AprilGrid 或 Mocap tracking 为代价。

建议每条 60–90 s。软件不得因为动作阶段切换而停止或拆分传感器数据流。

### 8.3 `validation_01`

界面明确显示：

> Validation 只用于验证，禁止参与 calibration optimization。

动作包含正常走动、转身、上下移动、roll/pitch 和多轴组合，并且不完全复制 `joint_dynamic_*`。

## 9. 数据写入架构

采集核心沿用已有 `record_oak4p` 的设备读取、时间戳配组、有界队列和并行 PNG 写盘逻辑，在其上增加模式向导、Mocap 输入和 Extended EuRoC writer。

线程职责必须分离：

```text
OAK 设备读取
Camera 时间戳配组
IMU 读取
Mocap 读取
磁盘写入
实时统计
UI 显示
```

要求：

- 所有采集队列有界，队列深度实时可见；
- UI 和 AprilGrid 检测不得阻塞设备读取和写盘；
- 队列溢出不得静默丢弃，必须累计错误并使本条无法通过 QA；
- 图片写入使用临时文件后原子改名，避免留下看似完整的半张图片；
- 停止时先停止接收新样本，再排空写盘队列，最后生成统计和 checksum；
- 原始测量只追加，不在录制后就地修改。

## 10. 标准输出格式

每条 sequence 名称固定为：

```text
YYYYMMDD_<rig_id>_<purpose>_<index>
```

例如：

```text
20260908_rig01_joint_dynamic_01
```

目录必须直接符合 Extended EuRoC：

```text
<sequence_name>/
├── mav0/
│   ├── cam0/data/ + data.csv
│   ├── cam1/data/ + data.csv
│   ├── cam2/data/ + data.csv
│   ├── cam3/data/ + data.csv
│   ├── imu0/data.csv
│   └── mocap0/data.csv
└── meta/
    ├── manifest.yaml
    ├── aprilgrid.yaml
    ├── events.csv
    ├── clock_samples.csv
    ├── recorder_stats.json
    └── checksums.sha256
```

`state_groundtruth_estimate0` 是标定后的派生结果，采集软件不得生成或伪装成原始 Mocap 数据。

### 10.1 Camera

```csv
#timestamp [ns],filename
1788839123456789000,1788839123456789000.png
```

- 第一列为 `int64` 纳秒；
- 文件名使用设备 timestamp；
- 图片为 1280×800 无损 PNG；
- exposure、ISO、sequence number、host 时间映射可以写入额外 sidecar，但不得改变 canonical timestamp 的含义。

### 10.2 IMU

```csv
#timestamp [ns],w_RS_S_x [rad s^-1],w_RS_S_y [rad s^-1],w_RS_S_z [rad s^-1],a_RS_S_x [m s^-2],a_RS_S_y [m s^-2],a_RS_S_z [m s^-2]
```

- gyro 单位为 rad/s；
- accelerometer 单位为 m/s²；
- `manifest.yaml` 明确 IMU 三轴方向；
- 若设备分别产生 accel 和 gyro 样本，软件必须同时保留两路原始样本及其时间戳；生成统一 `data.csv` 所用的配对或插值规则必须写入 manifest，禁止丢弃无法配对的原始样本。

### 10.3 Mocap

```csv
#timestamp [ns],p_RS_R_x [m],p_RS_R_y [m],p_RS_R_z [m],q_RS_w [],q_RS_x [],q_RS_y [],q_RS_z [],tracking_valid
```

- pose 定义为 `T_mocap_world_mocap0`；
- position 单位为 m；
- quaternion 固定为 `qw,qx,qy,qz`；
- lost tracking 样本不得伪造 pose 或直接删除，必须保留状态；
- 源 SDK 的原始 timestamp 和附加质量字段必须保留。

### 10.4 Metadata

`manifest.yaml` 至少记录：

- format name/version、sequence name/purpose、operator；
- rig ID、hardware revision 和设备序列号；
- 相机映射、分辨率、FPS、图像格式、曝光、ISO、focus；
- Camera/IMU/Mocap 的 frame、轴定义、单位、频率；
- 每类 timestamp 的 source、reference 和 clock domain；
- Mocap rigid body、world frame、pose 定义和 quaternion 顺序；
- AprilGrid 参数；
- FSIN 和跨主机 clock sync 机制；
- recorder 版本、Git commit、DepthAI/firmware/SDK 版本；
- QA 状态及失败原因。

## 11. 实时与结束后 QA

### 11.1 硬失败

以下任一情况发生后，本条 sequence 不得标为 `QA_PASSED`：

- 相机、IMU 或必需的 Mocap 数据流中断；
- Camera timestamp duplicate、乱序或缺少对应图像；
- 四相机组内最大设备时间戳偏差超过 1000 µs；
- 设备读取或写盘队列丢包；
- IMU 出现异常长 gap 或单位/轴定义不明确；
- Mocap rigid body 错误或出现大段 lost tracking；
- 图片文件损坏、数量与 CSV 不一致；
- Rig、Camera、IMU、marker 或 focus 在五包中途发生变化；
- `manifest.yaml`、统计或 checksum 生成失败。

数据必须保留并标记为 `REJECTED`，不得静默删除。

### 11.2 质量警告

软件应实时提示但不擅自修复：

- AprilGrid 检测率低；
- 图像覆盖集中在中心或相机共视图不连通；
- motion blur 或过曝；
- dynamic 数据只有单轴运动；
- IMU 三轴 rotation/acceleration excitation 不充分；
- Mocap tracking invalid 比例升高；
- Windows 与 Mocap clock offset 漂移。

首批实测完成前，不为这些指标编造固定阈值。软件必须输出原始统计，阈值作为版本化配置写入 manifest，冻结后再用于自动 PASS/REJECT。

### 11.3 `recorder_stats.json`

至少报告：

- 实际时长；
- 各相机帧数、平均 FPS、drop、duplicate、最大间隔；
- 相机同步组数、skew 的 mean/p95/max；
- IMU 样本数、平均频率、最大 gap 和三轴激励摘要；
- Mocap 样本数、平均频率、invalid 数、valid ratio 和最长 lost 区间；
- AprilGrid 检测、画面区域覆盖和相机共视摘要；
- 写盘峰值速率、最大队列深度和剩余磁盘空间；
- 最终 `RAW`、`QA_PASSED` 或 `REJECTED` 状态及原因。

### 11.4 Checksum

安全停止后自动为 `mav0/` 和除 checksum 自身之外的 `meta/` 文件生成 SHA-256。数据复制到服务器后必须可用同一文件验证完整性。

## 12. 数据安全与故障恢复

- 新 sequence 先写入带 `.recording` 标记的唯一目录；正常 finalizing 完成后才移除标记；
- `q`、Esc、停止按钮和 Windows 关闭事件必须走同一安全收尾路径；
- 断电或崩溃后的数据保持只读可检查，不自动删除；
- 程序重启后显示未完成原因，并建议重录；
- 不提供原始数据编辑功能；
- 重新 QA 或生成派生格式时写入 `derived/`，不得修改 `mav0/`；
- 日志不得只存在控制台，必须随 sequence 保存。

## 13. 配置与发布

### 13.1 配置

硬件相关参数使用一个版本化配置文件，不为每个字段单独创造配置层。至少包含：

```text
camera mapping
resolution / FPS
FSIN mode / skew gate
IMU rate / units / axes
Mocap source / rigid body / axes
AprilGrid parameters
exposure / ISO / focus
QA thresholds
```

每条 sequence 在开始时复制一份实际生效配置到 `manifest.yaml`，不能只引用采集机上的外部配置路径。

### 13.2 发布

- 保留 Python 源码入口，便于开发和诊断；
- 使用 PyInstaller 在 Windows 上构建单文件 EXE；
- EXE 不要求目标机安装 Python、ROS、Docker 或 WSL；
- 软件版本、Git commit 和依赖版本写入输出；
- 发布包包含快速启动说明和默认配置，不包含标定优化环境。

## 14. 现有实现与增量

现有 `Delta_D2_4cam_calibration` 已具备：

- Windows 10/11 Python/EXE 入口；
- OAK-4P `cam0..cam3 = CAM_A..CAM_D` 固定映射；
- 30 Hz FSIN、1 ms 相机同步门槛；
- 四路 1280×800 无损 PNG 和 400 Hz RAW IMU；
- USB 3、磁盘空间、有界队列、安全停止和基础验证。

完成本方案还必须增加：

1. 五包任务向导及按模式生成标准 sequence 名称；
2. Mocap 实时接入、tracking 质量和源时间戳保存；
3. Extended EuRoC 直接写入，不再以现有 `session.json + groups.csv + split IMU CSV` 作为最终交付格式；
4. `manifest.yaml`、`aprilgrid.yaml`、`events.csv`、`clock_samples.csv` 和 `checksums.sha256`；
5. AprilGrid 覆盖、相机共视和动态动作提示；
6. Camera/IMU 与 Mocap 的 clock mapping 和 drift 诊断；
7. 完整任务汇总与明确的 `QA_PASSED/REJECTED` 状态。

现有采集核心应复用，不重新实现 DepthAI 管线、PNG writer 或时间戳配组。

## 15. 验收标准

软件实现完成后，至少通过以下验收：

### 15.1 功能验收

- 可在一台干净的 64 位 Windows 10/11 主机运行 EXE；
- 可按向导创建五个名称正确且互不覆盖的数据包；
- `joint_dynamic_*` 和 `validation_01` 同时包含四相机、IMU 和 Mocap；
- 每包目录、CSV、metadata 和 checksum 符合本文档；
- Validation 在界面和 manifest 中明确标为不参与优化；
- 所有动作事件、设备配置和软件版本可追溯。

### 15.2 数据验收

- 四相机均为 1280×800、30 Hz、无损 PNG；
- Camera timestamp 单调，duplicate 为 0；
- FSIN 组内最大设备时间戳偏差不超过 1000 µs；
- 写盘和设备队列 drop 为 0；
- IMU 为 RAW 数据，目标 400 Hz，单位与轴定义明确；
- Mocap pose、measurement timestamp 和 tracking 状态完整；
- 文件数量与 CSV 一致，SHA-256 全部通过；
- 未通过任一硬门槛的数据只能是 `REJECTED`。

### 15.3 压力与恢复验收

- 以正式参数连续采集至少 30 min，无内存持续增长、队列溢出或静默丢包；
- 磁盘空间不足时在开始前拒绝，录制中空间不足时安全停止并标记失败；
- 模拟 Mocap 断流、USB 断开、写盘变慢和程序强退后，均不产生假 `QA_PASSED`；
- 正常停止后所有队列排空，CSV、统计和 checksum 可重复验证。

## 16. v1 明确不做

- 不在 Windows 软件中运行相机、Camera–IMU 或 IMU–Mocap 优化；
- 不生成 ROS bag、Basalt dataset 或 Hand-eye CSV，这些属于 `derived/`；
- 不实时计算最终外参或 `canonical_calibration.yaml`；
- 不支持多种未知 Mocap 协议的插件市场；
- 不引入数据库、云服务、用户系统或远程控制；
- 不对原始图像去畸变、resize、crop 或有损压缩。

先完成一个已确定 Mocap 系统、一个 OAK-4P Rig 和五包流程的可靠闭环；只有实际出现第二种 Mocap 接口时，再抽象通用适配层。

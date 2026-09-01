# 头戴多目 VIO-VSLAM Basalt 标定与数据参考

## 定位

[Basalt](../../references/basalt/) 是一套 C++ 视觉惯性工具链，覆盖相机、IMU 与动捕标定，离线/实时 VIO 和建图。当前本地参考固定于提交 `0f3b2b5`；阅读入口为：[README](../../references/basalt/README.md)、[Realsense 教程](../../references/basalt/doc/Realsense.md)和[标定教程](../../references/basalt/doc/Calibration.md)。

它在本项目中最有价值的不是直接驱动 OAK-4P，而是提供一条完整且可检查的链路：**原始测量录制 → EuRoC 数据组织 → 相机/IMU/动捕标定 → 时间对齐 → VIO 评测**。这正对应本项目 P1–P5 的数据与验证要求。

## 相关组件

| 上游入口 | 职责 | 本项目可学习内容 |
| --- | --- | --- |
| `src/rs_t265_record.cpp` | T265 双目、IMU 与设备姿态录制器 | 时间戳命名、异步写入队列、图像/IMU CSV 组织和录制期间的丢帧观测 |
| `scripts/basalt_verify_dataset.py` | 数据集完整性检查 | 用实际频率和缺失图像判定录制质量，而不是只检查目录是否存在 |
| `src/calibrate.cpp` | 多相机几何、暗角标定 | AprilGrid 观测、内参、相机间外参和优化收敛流程 |
| `src/calibrate_imu.cpp` | 相机–IMU–Mocap 联合标定 | IMU 噪声模型、相机–IMU 外参与 marker–IMU 外参的初始化和优化顺序 |
| `src/time_alignment.cpp` | 动捕与 IMU 时间对齐 | 从动捕旋转推导角速度，再与 gyro 对齐估计时间差 |
| `src/vio.cpp`、`src/rs_t265_vio.cpp` | 离线与设备实时 VIO | EuRoC 数据集读入与后续算法适配的参考 |

## EuRoC 数据契约

`basalt_rs_t265_record` 在每次录制下创建 `mav0/`，并为每帧以纳秒时间戳命名图像，再在 `data.csv` 中保存“时间戳、文件名”映射；IMU CSV 保存 gyro 与 accel。其 T265 实现只创建 `cam0`、`cam1` 和 `imu0`，图像使用 WebP 或 JPEG。

自研四目 Recorder 应复用这种**时间戳和 CSV 契约**，但不是照抄 T265 文件数或编码：

```text
sequence/
├── mav0/
│   ├── cam0..cam3/data/ + data.csv
│   ├── imu0/data.csv
│   ├── mocap0/data.csv
│   └── state_groundtruth_estimate0/data.csv
└── meta/
    ├── manifest.yaml
    ├── events.csv
    ├── recorder_stats.json
    └── checksums.sha256
```

其中前两项是与 EuRoC 生态兼容的核心；`mocap0`、已对齐 Ground Truth 与 `meta` 是本项目为追溯性添加的扩展。Basalt 的 WebP 写入是其自身读取链路的一部分，不能因此假定所有 EuRoC 工具都接受 WebP；最终图像编码需由后续算法适配器实际验证。

## 标定与 Ground Truth 的关键流程

1. **相机几何标定**：使用静态 AprilGrid，覆盖各相机完整视场和多种观察角度；完成角点检测、内参初始化、相机位姿/外参初始化与优化。
2. **相机–IMU 标定**：`init_cam_imu` 通过相机角速度与 gyro 初始化旋转；在主要优化收敛后，才把相机–IMU 时间差和 IMU scale 等选项作为细化项。
3. **marker–IMU 外参**：先执行 `init_mocap`，再启用 `opt_mocap`；结果保存为 `mocap_calibration.json`。这为项目的 `T_I_M` 提供方法参考。
4. **Mocap–IMU 时间差**：这不是 `opt_cam_time_offset` 的职责。Basalt 另用动捕姿态推导角速度，与 IMU gyro 误差最小化，生成对齐且转换到 IMU frame 的轨迹。
5. **数据质量检查**：录制结束运行验证器，确认实际频率与图像文件齐全；队列堆积意味着写入速度不足，需按丢帧处理而非继续使用该 sequence。

所有 IMU 噪声密度与 random walk 必须来自本模组实测；教程中的 T265 数值仅为示例，不能迁移到 OAK-4P 的内置 IMU。

## 与 P0–P5 的关系

| 项目阶段 | Basalt 的参考价值 | 不应替代的项目工作 |
| --- | --- | --- |
| P0 Layout | 刚性安装 marker 与相机/IMU 的要求 | Layout ID、CAD、照片和结构验收 |
| P1 传感器质检 | 录制队列、实际频率和缺失文件检查 | OAK-4P 相机同步、USB/SSD 压力与温漂测试 |
| P2 时空标定 | AprilGrid、相机–IMU、marker–IMU 优化流程 | 四目模型适配、重复性与外参闭环验收 |
| P3 EuRoC Recorder | 目录、CSV、纳秒命名和验证器设计 | 四路相机写入、manifest、events 与 checksum |
| P4 Ground Truth | marker–IMU 标定与 gyro 对齐方法 | 原始动捕保留、clock drift 检验和 `T_W_I` 交付 |
| P5 算法适配 | EuRoC 读取和 Basalt VIO 的参考 | 目标算法的数据读取、相机模型与评测接口 |

## 适用边界

Basalt 的 RealSense 录制器依赖 `librealsense2` 与 T265 设备接口，且实现是双目；它不能直接连接 OAK-4P，也不能直接成为四目 RK3588 Recorder。可复用的是数据模型、验证规则和标定思路，不是设备代码。

建议阅读顺序为：先看 [Realsense 教程的数据录制段落](../../references/basalt/doc/Realsense.md#recording-own-dataset)，再看 [Camera + IMU + Mocap calibration](../../references/basalt/doc/Calibration.md#camera--imu--mocap-calibration)，最后对照本项目的 [P0–P4 测试方案](头戴多目VIO_VSLAM_标定与数据采集_测试方案.md)。

# 头戴多目 VIO-VSLAM COCalib 多相机标定参考

## 定位

[CO-Calib](../../references/CO-Calib/)（其 CLI 包名为 `omnicalib-open`）是面向任意相机数量的多相机标定工具箱，当前本地参考固定于提交 `8b894ee`。它的重点不是替代 Kalibr，而是在 Kalibr 前增加**输入统一、标定板检测、Datawash 与共视观测筛选**，最终生成标准 ROS1 标定 bag 并调用 Kalibr。

它最适合本项目 P2 的“相机内参与相机间外参”学习：如何将四目数据清洗成可观测、可追溯的标定子集，并输出可检查的 `camchain` 结果。其默认工作流不等同于相机–IMU、marker–IMU 或 Mocap–IMU 时间对齐；这些仍分别参考 tools-quarterKalibr 与 Basalt。

## 输入、配置与输出

| 类别 | CO-Calib 支持内容 | 对四目项目的含义 |
| --- | --- | --- |
| 输入 | 图像序列、ROS1 bag、ROS2 bag | 可以直接复用 P3 录制结果，也可先转为它定义的图像序列 |
| 图像序列 | 每相机一个目录、`images/` 和 `timestamps.csv`；CSV 为 `frame_id,timestamp_ns,filename` | 四路相机各自保存；不要求完全同频，避免拼接图成为唯一输入格式 |
| `rig.yaml` | 相机数量、ID、话题、图像目录、frame ID、相机模型、同步容差 | 用 `cam0`–`cam3` 明确固定物理相机，并把同步容差显式版本化 |
| `datawash.yaml` | 角点置信度、最少有效点数、anchor/covisible/mono-fill 选择规则与预算 | 将“视场覆盖”和“相机对共视”变成可复查的数据筛选条件 |
| `target.yaml` | 当前检测器使用的 6 × 6 AprilGrid | `tagSize`、`tagSpacing` 必须改为实物尺寸 |
| 输出 | `calibration_clean.bag`、`selected_roles.csv`、清洗摘要、`calibration-camchain.yaml`、报告 PDF | 可把输入、选帧和最终外参串为同一份可审计产物 |

当前示例 `rig.yaml` 使用 `sync_tolerance_ms: 10.0`，它只是样例值，不能直接作为 OAK-4P 的同步判据。相机模型可选 `omni-*`、`eucm-*`、`ds-none`、`pinhole-*` 等；模型选择必须基于实际镜头覆盖范围、标定残差和下游算法支持，而不是按名称猜测。

## Datawash 如何补足普通 Kalibr 流程

传统流程常把整段 bag 直接送入标定器，坏帧、低覆盖度、运动模糊或某些相机未共视的帧会增加优化难度。CO-Calib 先检测标定板，再为每个时刻赋予三类角色：

- **anchor**：保证单相机观测在图像有效区域有足够径向跨度；
- **covisible**：为每个相机对保留共同看到标定板的帧；
- **mono-fill**：补充单相机覆盖不足的区域。

`selected_roles.csv` 记录每个被选时刻和实际参与相机，可用来复查“某个外参到底由哪些共视观测支撑”。这比只保存一个最终 `camchain.yaml` 更适合调试四目 Layout 或查找局部视场覆盖不足。

## 与 P2 的推荐关系

```text
P1：确认四路时间戳、帧率和无丢帧
    ↓
P2：CO-Calib Datawash 选出有效 AprilGrid 观测
    ↓
Kalibr：估计四目内参和相机间外参
    ↓
检查重投影误差、相机对共视、外参环闭合和重复性
    ↓
tools-quarterKalibr / Basalt：补充相机–IMU、marker–IMU 与时间对齐
```

因此 CO-Calib 与 tools-quarterKalibr 是互补关系：前者擅长任意相机数的输入规范化与观测筛选，后者为固定四鱼眼提供拆包、相邻相机对和相机–IMU 原型；Basalt 则覆盖数据集验证、动捕和时间对齐。三者的输出都必须在同一 Layout ID 与 calibration ID 下归档，避免混用不同机械布局的参数。

## 运行与适配边界

CO-Calib 的运行环境是 Linux、Conda 与 Docker；NN-Detector 可使用 GPU，无法使用时会回退 CPU。其参考代码包含 Docker 化 Kalibr、检测模型与浏览器外参可视化，但不应在尚未验证的 RK3588 目标机上直接部署。

首次验证应选一个 Layout 的短标定序列，明确以下事实后再扩大数据规模：

1. 四路图像能否按 `rig.yaml` 正确读入，且相机 ID 与物理位置一致；
2. 选定相机模型与实际鱼眼图像匹配，AprilGrid 尺寸完全正确；
3. Datawash 后每个相机都有足够覆盖、每个邻接相机对都有共视观测；
4. `calibration-camchain.yaml` 的外参闭环与独立重复采集相符；
5. 输出目录、清洗 CSV 和报告已绑定 Layout ID 与校准记录。

建议先读中文 [README](../../references/CO-Calib/README.zh-CN.md)，然后依次查看 `configs/rig_stereo.yaml`、`configs/datawash.yaml` 与 `configs/target_aprilgrid_6x6.yaml`，最后检查 `visualization/` 对 `camchain.yaml` 的外参展示逻辑。

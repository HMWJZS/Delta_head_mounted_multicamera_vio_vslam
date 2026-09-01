# 头戴多目 VIO-VSLAM tools-quarterKalibr 四目 IMU 标定参考

## 定位

[tools-quarterKalibr](../../references/tools-quarterKalibr/) 是针对四目鱼眼模组的上游参考实现，当前本地参考固定于提交 `5a2be86`。其核心思路是：不将四个大视场鱼眼一次放入一个联合优化，而是拆成四个单目和四个相邻相机对，降低失败后重录整包数据的成本。

该项目适合学习 OAK-4P 这类“单张拼接图包含四路相机”的数据拆包、邻接相机对标定、逐相机相机–IMU 标定和虚拟双目生成。它是 P2 的候选原型，不是已验证的生产标定流水线。

## 输入假设与固定约定

| 项目项 | 上游实现的假设 | 接入前需要确认 |
| --- | --- | --- |
| 图像输入 | 一个 ROS1 bag 中的拼接四目图，默认话题为 `/oak_ffc_4p/assemble_image/compressed` | 实际话题、编码、拼接顺序与每子图的尺寸 |
| 相机顺序 | 从左到右垂直切分为 `CAM_A`、`CAM_B`、`CAM_C`、`CAM_D` | 物理相机 ID、坐标系、镜头方向和 Layout ID 的对应关系 |
| 标定板 | AprilGrid 6 × 6；提取器写入 `tagSize: 0.088`、`tagSpacing: 0.3` | 印刷板尺寸与间距必须与实际一致，不能沿用默认值 |
| IMU | ROS 话题与 `imu.yaml` 中噪声模型、更新率相匹配 | `rostopic`、速率、连续时间噪声 density/random walk 均需实测 |
| 运行环境 | ROS1、`rosbag`、Docker、TartanCalib/Kalibr 及图形显示环境 | RK3588/主机架构、容器可用性与 `DISPLAY` 依赖 |

`config.py` 提供话题与路径样例；实际 Notebook 使用自身的 `global_*` 变量，因此两者必须同步核对。源码中的 `imu_toopic` 是拼写为此的配置键，不应在自研接口中继续沿用。

## 实际工作流

```text
输入 ROS1 bag
    ↓
检查图像与 IMU 话题
    ↓
按 AprilTag 在四子图中的可见组合拆出 4 个单目包和 4 个相邻双目包
    ↓
并行单目鱼眼内参（TartanCalib，omni-radtan）
    ↓
相邻双目外参：D–A、A–B、B–C、C–D
    ↓
四个相机各自相机–IMU 标定（Kalibr）
    ↓
汇总为 `fisheye_cams.yaml`
    ↓
可选：按虚拟视角重采样为虚拟双目并再次标定
```

`calibrate.ipynb` 是这一流程的编排入口。`BagExtractor.py` 会把拼接图切为四个 ROS 图像消息，并依据 AprilTag 的可见相机组合推进采集阶段；`Calibration.py` 对四个单目和四个邻接对并行运行容器内的标定；`Utils.py` 汇总每个相机的 `T_cam_imu`；`VirtualStereoCalibration.py` 在已有鱼眼配置基础上生成虚拟双目图像与 bag。

## 主要输出与含义

| 输出 | 来源 | 用途 |
| --- | --- | --- |
| `CAM_A` … `CAM_D` 的 `log1-camchain.yaml` | 单目 TartanCalib | 单个鱼眼内参及模型参数 |
| `CAM_D-CAM_A`、`CAM_A-CAM_B`、`CAM_B-CAM_C`、`CAM_C-CAM_D` 的标定目录 | 相邻双目 TartanCalib | 邻接相机间外参和双目报告 |
| `<CAM>-camchain-imucam.yaml` | Kalibr 相机–IMU标定 | 每个相机相对 IMU 的外参与时差结果 |
| `fisheye_cams.yaml` | `Utils.generate_d2vins_cinfiguration` | 面向下游多目/虚拟双目程序的四相机参数汇总 |
| `virtual_stereo_calibration_*` | 虚拟双目模块 | 虚拟视角、AprilGrid 与后续双目标定产物 |

相邻对结果构成一个环，理论上可检查 `D→A→B→C→D` 的闭环一致性。这是本项目 P2 的重要验收项：不能只因每个 YAML 已生成就视为外参可靠。

## 对本项目的价值与限制

它解决的是四鱼眼联合优化难收敛、某一路失败导致全量重采以及标定包不可恢复的问题。对每个相机/相机对独立保存结果，也利于定位失败源和重复采集。

但源码的 `check_calibration` 与 `check_imu_calibration` 只检查预期 YAML 文件是否存在；它们不检查重投影误差、相机同步、环闭合、时间差、IMU 激励或重复性。因此本项目仍必须以 [P2-GATE](头戴多目VIO_VSLAM_标定与数据采集_测试方案.md#p2-gate) 的质量指标判定结果，不能把文件存在当作 PASS。

此外，虚拟双目模块是下游深度/双目用途的可选扩展，并不构成四目内参、相邻外参或相机–IMU 标定的前置条件。先完成真实相机模型与外参验证，再决定是否需要该模块。

## 建议学习顺序

1. 阅读 [README](../../references/tools-quarterKalibr/README.md) 了解“相邻对替代四目联合优化”的动机。
2. 阅读 [Notebook](../../references/tools-quarterKalibr/calibrate.ipynb) 的八个执行单元，明确每一步会生成什么数据。
3. 对照 `utils/BagExtractor.py` 和 `utils/PrivateGlobalValue.py`，核对拼接顺序与相机对定义。
4. 对照 `utils/Calibration.py` 和 `imu.yaml`，确认容器命令、相机模型、AprilGrid 与 IMU 噪声假设。
5. 最后才研究 `VirtualStereoCalibration.py`，并只在真实四目参数已通过 P2 门控后启用。

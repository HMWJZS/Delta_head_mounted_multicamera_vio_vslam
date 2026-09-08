# 文档索引

## 当前任务：Camera + IMU + Mocap 标定

目标是在同一刚性 Rig 上得到四相机内参与相机间外参、Camera–IMU 外参与时间偏移、IMU–Mocap marker 外参与时间偏移，并通过独立数据验证后生成 `canonical_calibration.yaml`。

### 固定决策

- 220° 鱼眼工程模型使用 **EUCM**（`eucm-none`）；`omni+radtan` 只作交叉比较，当前 VIO 不使用。
- 统一采用 `T_A_B` 表示从 B frame 变换到 A frame；所有时间偏移必须记录符号定义。
- 原始数据使用 Extended EuRoC，`mav0/` 与 `meta/` 不修改，工具输入和结果放入 `derived/`。
- 完整采集包含 `cam_static_01/02`、`joint_dynamic_01/02` 和 `validation_01`；Validation 不参与优化。
- Camera–Camera 主采集固定 Rig、移动 AprilGrid；动态采集固定 AprilGrid、移动 Rig，并同时记录 Camera、IMU 和 Mocap。

### 执行顺序

| 阶段 | 执行动作 | 通过条件或输出 |
| --- | --- | --- |
| 0. 冻结 Rig | 锁定相机、IMU、Mocap marker、焦点、相机编号和 AprilGrid 参数 | 任一刚性关系变化即重新开始 |
| 1. 打通时间链 | 确认 FSIN、Camera/IMU 设备时间、Mocap 时间源，以及 Mocap 导入 `mav0/mocap0/data.csv` 的方法 | 时间戳单调、无异常 gap，偏移与漂移可估计 |
| 2. IMU 噪声 | 额外录制 `imu_static`，计算 Allan deviation | 生成可供 Kalibr/Basalt 使用的 IMU noise YAML |
| 3. 采集五个包 | 按[现场采集动作清单](calibration/头戴多目VIO_VSLAM_标定现场采集_动作清单.md)逐条录制 | 五个包均完整落盘 |
| 4. 逐包 QA | 每录完一包立即检查同步、丢帧、IMU gap、Mocap tracking、AprilGrid 覆盖和 checksum | 状态为 `QA_PASSED`；失败包直接重录 |
| 5. 相机标定 | 用 `cam_static_01/02` 分别运行 CO-Calib/Kalibr，固定使用 `eucm-none` | 内参、Camera–Camera 外参、边缘残差和重复性通过 |
| 6. Camera–IMU | 用 `joint_dynamic_01/02` 分别运行 Kalibr，并用 Basalt 交叉验证 | `T_imu_cam*`、`dt_cam_imu` 及重复性通过 |
| 7. IMU–Mocap | 先做 Basalt `init_mocap → opt_mocap`，再用 Hand-eye 独立验证 | `T_imu_mocap`、`dt_imu_mocap` 及重复性通过 |
| 8. 独立验收 | 只在 `validation_01` 上检查重投影、跨相机一致性、VIO 和 Mocap 残差 | 输出 ACCEPT/REJECT、验证报告和 `canonical_calibration.yaml` |

### 当前工程边界（2026-09-08）

[Delta_D2_4cam_calibration](https://github.com/c1lz/Delta_D2_4cam_calibration) 已有四相机 + IMU 采集与验证、EUCM 相机链、IMU Allan、Kalibr Camera–IMU 和联合评估入口；现有 EUCM 结果属于诊断结果，不是最终标定。

正式采集前仍需确认或补齐：

- Mocap 原始轨迹采集及 `mav0/mocap0/data.csv` 导入；
- Camera/IMU 与 Mocap 的时间偏移、clock drift 估计；
- Basalt `init_mocap/opt_mocap` 执行入口；
- Hand-eye 独立验证与 `canonical_calibration.yaml` 汇总。

## 文档分工

- [Windows 时空标定数据采集软件实施方案](calibration/头戴多目VIO_VSLAM_Windows时空标定数据采集软件_实施方案.md)：Windows 五包采集软件的功能、数据、同步、界面、QA 和验收要求。
- [测试方案](calibration/头戴多目VIO_VSLAM_标定与数据采集_测试方案.md)：查看 P0–P4 范围、Gate 和最终交付物。
- [现场操作手册](calibration/头戴多目VIO_VSLAM_多相机IMU与Mocap时空标定_现场操作手册.md)：执行完整采集、标定、重复性比较和 Validation。
- [现场采集动作清单](calibration/头戴多目VIO_VSLAM_标定现场采集_动作清单.md)：现场逐项打勾。
- [数据包格式](calibration/头戴多目VIO_VSLAM_时空标定数据包格式_方法说明.md)：实现 Recorder、转换器和 QA 时遵循。
- [Basalt 参考](calibration/头戴多目VIO_VSLAM_Basalt标定与数据参考_方法说明.md)：Camera–IMU–Mocap 联合标定和时间对齐。
- [tools-quarterKalibr 参考](calibration/头戴多目VIO_VSLAM_toolsQuarterKalibr四目IMU标定参考_方法说明.md)：四目拆包和逐相机 Camera–IMU 原型。
- [CO-Calib 参考](calibration/头戴多目VIO_VSLAM_COCalib多相机标定参考_方法说明.md)：多相机 Datawash、共视选帧和相机链。

## 参考项目解析

- [参考项目解析索引](references/README.md)
- [Basalt 项目详细解析](references/头戴多目VIO_VSLAM_Basalt项目解析_方法说明.md)
- [tools-quarterKalibr 项目详细解析](references/头戴多目VIO_VSLAM_toolsQuarterKalibr项目解析_方法说明.md)
- [CO-Calib 项目详细解析](references/头戴多目VIO_VSLAM_COCalib项目解析_方法说明.md)

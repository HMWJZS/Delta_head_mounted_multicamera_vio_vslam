# 头戴多目 VIO-VSLAM 测试

头戴式四目相机与 IMU 的 VIO/VSLAM 标定、数据采集、算法测试和评测项目。

## 文档

- [当前 Camera + IMU + Mocap 标定执行入口](docs/README.md)：固定决策、执行顺序、验收出口和当前工具缺口。
- [标定与数据采集测试方案](docs/calibration/头戴多目VIO_VSLAM_标定与数据采集_测试方案.md)：P0–P4 的 Layout、传感器质检、时空标定、EuRoC 数据录制与动捕 Ground Truth 流程。
- [多相机、IMU 与 Mocap 时空标定现场操作手册](docs/calibration/头戴多目VIO_VSLAM_多相机IMU与Mocap时空标定_现场操作手册.md)：完整采集流程、标定方法、重复性比较和独立数据验证。
- [标定现场采集动作清单](docs/calibration/头戴多目VIO_VSLAM_标定现场采集_动作清单.md)：五个标准数据包的现场检查与动作要求。
- [时空标定数据包格式](docs/calibration/头戴多目VIO_VSLAM_时空标定数据包格式_方法说明.md)：Extended EuRoC 目录、时间戳、CSV、元数据和验收约定。
- [Basalt 标定与数据参考](docs/calibration/头戴多目VIO_VSLAM_Basalt标定与数据参考_方法说明.md)：EuRoC 录制、数据质检、动捕外参与时间对齐。
- [tools-quarterKalibr 四目 IMU 标定参考](docs/calibration/头戴多目VIO_VSLAM_toolsQuarterKalibr四目IMU标定参考_方法说明.md)：拼接四鱼眼拆包、相邻相机对、相机–IMU 与虚拟双目原型。
- [CO-Calib 多相机标定参考](docs/calibration/头戴多目VIO_VSLAM_COCalib多相机标定参考_方法说明.md)：Datawash、任意相机数配置与 Kalibr 相机链输出。

## 参考项目解析

- [参考项目解析索引](docs/references/README.md)：三个本地上游仓库的版本、能力边界和阅读路线。
- [Basalt 项目详细解析](docs/references/头戴多目VIO_VSLAM_Basalt项目解析_方法说明.md)：工程架构、标定、EuRoC 风格录制、Mocap 时间对齐、光流、VIO、Mapper 与测试。
- [tools-quarterKalibr 项目详细解析](docs/references/头戴多目VIO_VSLAM_toolsQuarterKalibr项目解析_方法说明.md)：四拼接图拆包状态机、Notebook 真实路径、Docker 标定、虚拟双目和当前缺口。
- [CO-Calib 项目详细解析](docs/references/头戴多目VIO_VSLAM_COCalib项目解析_方法说明.md)：多格式输入、同步、检测缓存、质量指标、Datawash 选帧与 Kalibr 桥接。

后续按需添加算法适配、实验数据、运行日志和评测产物。

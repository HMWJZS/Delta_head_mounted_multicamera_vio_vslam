# 头戴多目 VIO-VSLAM 测试

头戴式四目相机与 IMU 的 VIO/VSLAM 标定、数据采集、算法测试和评测项目。

## 文档

- [标定与数据采集测试方案](docs/calibration/头戴多目VIO_VSLAM_标定与数据采集_测试方案.md)：P0–P4 的 Layout、传感器质检、时空标定、EuRoC 数据录制与动捕 Ground Truth 流程。
- [Basalt 标定与数据参考](docs/calibration/头戴多目VIO_VSLAM_Basalt标定与数据参考_方法说明.md)：EuRoC 录制、数据质检、动捕外参与时间对齐。
- [tools-quarterKalibr 四目 IMU 标定参考](docs/calibration/头戴多目VIO_VSLAM_toolsQuarterKalibr四目IMU标定参考_方法说明.md)：拼接四鱼眼拆包、相邻相机对、相机–IMU 与虚拟双目原型。
- [CO-Calib 多相机标定参考](docs/calibration/头戴多目VIO_VSLAM_COCalib多相机标定参考_方法说明.md)：Datawash、任意相机数配置与 Kalibr 相机链输出。

## 参考项目解析

- [参考项目解析索引](docs/references/README.md)：三个本地上游仓库的版本、能力边界和阅读路线。
- [Basalt 项目详细解析](docs/references/头戴多目VIO_VSLAM_Basalt项目解析_方法说明.md)：工程架构、标定、EuRoC 风格录制、Mocap 时间对齐、光流、VIO、Mapper 与测试。
- [tools-quarterKalibr 项目详细解析](docs/references/头戴多目VIO_VSLAM_toolsQuarterKalibr项目解析_方法说明.md)：四拼接图拆包状态机、Notebook 真实路径、Docker 标定、虚拟双目和当前缺口。
- [CO-Calib 项目详细解析](docs/references/头戴多目VIO_VSLAM_COCalib项目解析_方法说明.md)：多格式输入、同步、检测缓存、质量指标、Datawash 选帧与 Kalibr 桥接。

后续按需添加算法适配、实验数据、运行日志和评测产物。

# 头戴多目 VIO-VSLAM 测试

头戴式四目相机与 IMU 的 VIO/VSLAM 标定、数据采集、算法测试和评测项目。

## 文档

- [标定与数据采集测试方案](docs/calibration/头戴多目VIO_VSLAM_标定与数据采集_测试方案.md)：P0–P4 的 Layout、传感器质检、时空标定、EuRoC 数据录制与动捕 Ground Truth 流程。
- [Basalt 标定与数据参考](docs/calibration/头戴多目VIO_VSLAM_Basalt标定与数据参考_方法说明.md)：EuRoC 录制、数据质检、动捕外参与时间对齐。
- [tools-quarterKalibr 四目 IMU 标定参考](docs/calibration/头戴多目VIO_VSLAM_toolsQuarterKalibr四目IMU标定参考_方法说明.md)：拼接四鱼眼拆包、相邻相机对、相机–IMU 与虚拟双目原型。
- [CO-Calib 多相机标定参考](docs/calibration/头戴多目VIO_VSLAM_COCalib多相机标定参考_方法说明.md)：Datawash、任意相机数配置与 Kalibr 相机链输出。

后续按需添加算法适配、实验数据、运行日志和评测产物。

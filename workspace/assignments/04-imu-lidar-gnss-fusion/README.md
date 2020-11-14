# Sensor Fusion: IMU-Lidar-GNSS Fusion -- 多传感器融合定位: 基于滤波的融合定位

This is the solution of Assignment 04 of Sensor Fusion from [深蓝学院](https://www.shenlanxueyuan.com/course/261).

深蓝学院从多传感器融合定位第4节IMU-Lidar-GNSS Fusion for Localization答案. 版权归深蓝学院所有. 请勿抄袭.

---

## Problem Statement

---

### 1. 基于滤波的融合定位

任选一种滤波模型和方法, 在`KITTI`数据集上, 实现基于`IMU`以及`点云地图匹配`的融合定位

#### ANS

此处选用`基于误差信息的Kalman Filtering`进行融合定位. 解决方案的架构图如下:

<img src="doc/images/01-error-based-kalman-filtering--architect.png" alt="Trajectory Estimation using Scan Context" width="100%" />

算法的`理论推导`以及`针对KITTI Road Test Data`的`伪代码实现`参考 [here](doc/derivations)

为了产生`未降频的IMU测量值`:

1. 下载`extract.zip`
2. 解压后, 将其中的`oxts`替换`sync`中的`oxts`
3. 修复`extract oxts`测量值的`timestamp`异常. 原始`timestamp`差分的统计描述如下:

    ```bash
    count                       45699
    mean       0 days 00:00:00.010318
    std        0 days 00:00:00.022812
    min      -1 days +23:59:59.949989
    25%        0 days 00:00:00.009967
    50%        0 days 00:00:00.009996
    75%        0 days 00:00:00.010025
    max        0 days 00:00:01.919595
    ```

    使用如下脚本进行修复

    ```python
    import pandas as pd
    
    #
    # read:
    #
    df_timestamps = pd.read_csv('timestamps.with.disorder.txt', header=None)
    
    #
    # modify:
    #
    # get first timestamp from velodyne since scan context is used for localization init:
    df_timestamps = df_timestamps.loc[df_timestamps[0] >= '2011-10-03 12:55:34.934675232', :]
    
    # parse timestamp
    df_timestamps[0] = df_timestamps[0].apply(lambda x: pd.to_datetime(x))
    # create time deltas:
    ss_timedelta = pd.Series(df_timestamps.index.values).apply(lambda x: pd.Timedelta(0.01*x, unit='sec'))

    # sync timestamps:
    df_timestamps[0] = df_timestamps.iloc[0, 0] + ss_timedelta

    #
    # write back:
    #
    df_timestamps.to_csv('timestamps.txt', header=None, index=False)
    ```

4. 修改`kitti2bag`源代码, 将`IMU` `Linear Acceleration / Angular Velocity`测量值的坐标系改为`Body Frame(xyz)` [here](src/lidar_localization/scripts/kitti2bag.py#L39)
    
    ```python
    def save_imu_data(bag, kitti, imu_frame_id, topic):
    print("Exporting IMU")
    for timestamp, oxts in zip(kitti.timestamps, kitti.oxts):
        q = tf.transformations.quaternion_from_euler(oxts.packet.roll, oxts.packet.pitch, oxts.packet.yaw)
        imu = Imu()
        imu.header.frame_id = imu_frame_id
        imu.header.stamp = rospy.Time.from_sec(float(timestamp.strftime("%s.%f")))
        imu.orientation.x = q[0]
        imu.orientation.y = q[1]
        imu.orientation.z = q[2]
        imu.orientation.w = q[3]
        # change here:
        imu.linear_acceleration.x = oxts.packet.ax
        imu.linear_acceleration.y = oxts.packet.ay
        imu.linear_acceleration.z = oxts.packet.az
        imu.angular_velocity.x = oxts.packet.wx
        imu.angular_velocity.y = oxts.packet.wy
        imu.angular_velocity.z = oxts.packet.wz
        bag.write(topic, imu, t=imu.header.stamp)
    ```
5. 然后运行`kitti2bag`, 产生用于`Filtering/Graph Optimization`的ROS Bag

在`kitti_2011_10_03_drive_0027_sync`上得到的ROS Bag Info如下:

```bash
$ rosbag info kitti_2011_10_03_drive_0027_synced.bag

path:        kitti_2011_10_03_drive_0027_synced.bag
version:     2.0
duration:    7:49s (469s)
start:       Oct 03 2011 12:55:36.39 (1317646536.39)
end:         Oct 03 2011 13:03:25.83 (1317647005.83)
size:        24.1 GB
messages:    269396
compression: none [18259/18259 chunks]
types:       geometry_msgs/TwistStamped [98d34b0043a2093cf9d9345ab6eef12e]
             sensor_msgs/CameraInfo     [c9a58c1b0b154e0e6da7578cb991d214]
             sensor_msgs/Image          [060021388200f6f0f447d0fcd9c64743]
             sensor_msgs/Imu            [6a62c6daae103f4ff57a132d6f95cec2]
             sensor_msgs/NavSatFix      [2d3a8cd499b9b4a0249fb98fd05cfa48]
             sensor_msgs/PointCloud2    [1158d486dd51d683ce2f1be655c3c181]
             tf2_msgs/TFMessage         [94810edda583a504dfda3829e70d7eec]
topics:      /kitti/camera_color_left/camera_info     4544 msgs    : sensor_msgs/CameraInfo    
             /kitti/camera_color_left/image_raw       4544 msgs    : sensor_msgs/Image         
             /kitti/camera_color_right/camera_info    4544 msgs    : sensor_msgs/CameraInfo    
             /kitti/camera_color_right/image_raw      4544 msgs    : sensor_msgs/Image         
             /kitti/camera_gray_left/camera_info      4544 msgs    : sensor_msgs/CameraInfo    
             /kitti/camera_gray_left/image_raw        4544 msgs    : sensor_msgs/Image         
             /kitti/camera_gray_right/camera_info     4544 msgs    : sensor_msgs/CameraInfo    
             /kitti/camera_gray_right/image_raw       4544 msgs    : sensor_msgs/Image         
             /kitti/oxts/gps/fix                     45700 msgs    : sensor_msgs/NavSatFix     
             /kitti/oxts/gps/vel                     45700 msgs    : geometry_msgs/TwistStamped
             /kitti/oxts/imu                         45700 msgs    : sensor_msgs/Imu           
             /kitti/velo/pointcloud                   4544 msgs    : sensor_msgs/PointCloud2   
             /tf                                     45700 msgs    : tf2_msgs/TFMessage        
             /tf_static                              45700 msgs    : tf2_msgs/TFMessage
```

基于`Eigen`与`Sophus`的`Error-State Kalman Fusion`实现参考[here](src/lidar_localization/src/models/kalman_filter/kalman_filter.cpp)

`IMU-Lidar Error-State Kalman Fusion Odometry`与`GNSS Groud Truth`的对比如下图所示. 其中`黄色`为`GNSS Groud Truth`, `红色`为`Lidar Odometry`, `蓝色`为`IMU-Lidar Fusion Odometry`:

<img src="doc/images/01-IMU-lidar-fusion.png" alt="IMU-Lidar Fusion v.s. GNSS" width="100%">

---

### 2. GNSS/IMU融合分析

推导组合导航(GNSS + IMU)的滤波模型. 要求:

* 对静止, 匀速, 转向, 加减速等不同运动状态下各状态量的可观测性和可观测度进行分析.

* 使用第三章所述数据仿真软件, 产生对应运动状态的数据, 进行Kalman滤波.

* 统计Kalman滤波中各状态量的收敛速度和收敛精度, 并与可观测度分析的结果汇总比较.

#### ANS
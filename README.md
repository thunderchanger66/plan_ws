# 组件关系
`plan_ws` 不是独立完整系统。它提供 EGO、轨迹到 PX4 Offboard 的控制桥、FAST-LIO 到 PX4 EKF 的外部里程计桥。完整仿真另外需要：
- Gazebo Harmonic 和 `ros_gz_bridge`。
- `ws_livox`：`livox_ros_driver2` 的 `CustomMsg` 消息定义。
- `mid360_ws`：Gazebo MID-360 插件、X500+MID-360 模型及仿真 launch。
- `fastlio2_pgo_ws`：FAST-LIO2、PGO 和对应配置。
- `plan_ws`：EGO、`px4_msgs`、`quadrotor_msgs` 和两个 bridge。
- `factory_world_2026_9_22`：当前默认的工厂世界和模型资源。

# 构建
```bash
cd .../plan_ws
source .../ws_livox/install/setup.bash
colcon build --symlink-install
```

# 每次仿真启动
每条命令使用独立终端。先关闭上一次的 Gazebo、PX4、Micro XRCE-DDS Agent、FAST-LIO/PGO 和 ROS launch；UDP 端口 `8888` 只能有一个 Micro XRCE-DDS Agent。所有终端使用同一个 `ROS_DOMAIN_ID` 和 DDS 实现。

**1. 启动：Gazebo、PX4 SITL、MID-360**  
~~**当前 SITL**：Gazebo launch 在 PX4 进程启动时通过 `PX4_PARAM_*` 环境变量注入一组仿真参数，不要求操作员每次手动进入 PX4 shell 执行 `param set`。当前默认值为：~~  
启动PX4后手动在`QGroundControl`找到：`左上角->Configure->Parameters`搜寻如下参数修改:

| PX4 参数 | 当前所需值 | 用途 |
|---|---:|---|
| `EKF2_EV_CTRL` | `7` | 融合外部视觉  水平/垂直位置及速度 |
| `EKF2_GPS_CTRL` | `0` | 当前无 GNSS aiding |
| `EKF2_HGT_REF` | `3` | 使用 external vision 高度参考 |
| `NAV_DLL_ACT` | `0` | 当前无 GCS 的 SITL 行为 |
| `COM_RCL_EXCEPT`、`COM_DLL_EXCEPT` | `4` | 当前 SITL Offboard 测试对 RC/GCS loss 的例外位 |
| `COM_LOW_BAT_ACT`、`SIM_BAT_DRAIN` | `0` | 关闭确定性仿真中的低电量动作/电量消耗 |

**2. 启动：FAST-LIO2/PGO**  
当前 EGO 使用 FAST-LIO 的 `/fastlio2/lio_odom`、`/fastlio2/world_cloud`

**3. 启动：EGO 和 PX4 bridge**
```bash
cd .../plan_ws
source .../ws_livox/install/setup.bash
source .../plan_ws/install/setup.bash
ros2 launch px4_bridge bringup.launch.py start_rviz:=true
```
该 bringup 默认启动 EGO、Micro XRCE-DDS Agent（UDP `8888`）、PX4 Offboard bridge、FAST-LIO 外部里程计 bridge 和 RViz。启动后飞机会自动悬停在`0.5m`，稳定后再开启发送目标。

**4. 启动：发送目标**
```bash
cd .../plan_ws
source .../ws_livox/install/setup.bash
source .../plan_ws/install/setup.bash
ros2 run px4_bridge manual_goal 4.0 -4.0 0.5 90
```
`manual_goal` 会等到 goal/yaw 订阅者出现、bridge 确认 yaw 模式并报告启动起飞稳定后才发目标；坐标参数x, y, z, yaw 为`lidar` 局部坐标，单位米，最后的 `90` 是该局部坐标中的 yaw 角度。自动朝向可以用 `--auto-yaw`，但不能同时传 yaw 数值。


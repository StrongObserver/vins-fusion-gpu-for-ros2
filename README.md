# vins-fusion-gpu-for-ros2
## 一、项目背景
随着室内外复杂环境对灵活飞行平台的需求增加，小型视觉无人机以体积小、机动性高的优势逐渐受到关注。本项目旨在自主设计并开发一款室内、室外均可使用的3.5寸小型无人机，充分利用Orin nx super的GPU算力优势，使用VINS-Fusion-GPU版进行定位、UWB进行通信，并对定位和通信分别进行ROS2适配，为将来的分布式集群通信打下基础，实现无人机在多场景下的可靠自主飞行。
## 二、项目目标
1. 设计并搭建一款室内室外均可使用的3.5寸小型无人机
2. 对vins-fusion-GPU版进行二次开发，使其适配ROS2
3. 对UWB驱动包进行修改，使其适配ROS2
## 三、开发日志
11.23 解决驱动bug & imu没话题 & ros1 & ros2转换
1.解决重新进入容器后d435i驱动突然又启动不起来的问题；（进入docker的命令和进入后的export不对）
2.解决启动驱动后没有camera/imu这个话题的问题（需要自己写rs_camera_vins.launch这个节点并让ai写成py版本方便ros2执行）
3.录制ros2的数据集并使用bridge将其进行转换（需要一个同时又ros1 & ros2的机器，我选择我的nuc 20.04）

---
11.24 标定IMU
参考https://zhuanlan.zhihu.com/p/20824959866
使用昨天录制并转换的数据集进行标定，得到的imu的最终结果（录制两个小时）

---
11.30 标定双目 & 联合标定
1.去楼下录制包，标定前擦一遍镜头跟标定版，把灯都打开，录制的时候，在本地电脑输入ros2 run rviz2 rviz2打开rviz2,订阅对应话题的image，确保标定板始终处于相机视野内
转ros1（坑：一般需要录制compress，如果录制原始视频流，record那个终端需要使用nvme+扩大缓存区（2G）才能不溢出，所以我用我笔记本新建了一个20.04的docker，配置好ros1&2和ros1_bridge）
还是开四个终端，record那个终端使用命令如下：
```
rosbag record -O double_cam_ros1.bag \
  --buffsize=2000000000 \
  /camera/camera/infra1/image_rect_raw \
  /camera/camera/infra2/image_rect_raw
```
<div align="center">
  <img width="548" height="560" alt="image" src="https://github.com/user-attachments/assets/3b69900c-5c97-4e37-8628-395bbc7f451e" />
</div>

12.1 解决宿主机能topic list但echo/hz不了容器话题bug & 解决vins报错wait for imu...bug & 完成部署并手持测试
1.宿主机能topic list但echo/hz不了容器话题bug，gpt说是DDS的问题：
FastDDS 在某些环境（特别是 Jetson + Docker + 多映射 volume）下，确实会出现“只能发现不能收数据”的坑
2.vins报错wait for imu...
解决方案：https://github.com/zinuok/VINS-Fusion-ROS2/issues/10
3.手持测试vins 并录制 rosbag包：
明确一遍最终的工作流命令：
```
/**************************************D435i驱动*********************************************/
sudo docker run --rm --runtime nvidia --gpus all --network=host --ipc host -it \
  --name vins_fusion_gpu_camera_driver_final \
  -v /home/cyf/my_ws:/workspace \
  -v /home/cyf/vins_ws:/root/vins_ws \
  -v /home/cyf/uwb_ws:/root/uwb_ws \
  -v /home/cyf/ch343ser_linux:/root/ch343ser_linux \
  -v /home/cyf/ros2_ws:/root/realsense_driver \
  -v /usr/lib:/usr/lib \
  -v /usr/local/lib:/usr/local/lib \
  -v /home/cyf/my_ws/src/opencv410/opencv-4.10.0/build/lib:/usr/local/opencv410/lib \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /home/cyf/cyclonedds.xml:/home/cyf/cyclonedds.xml \
  -e DISPLAY=$DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -e HTTP_PROXY="http://127.0.0.1:7897" \
  -e HTTPS_PROXY="http://127.0.0.1:7897" \
  -e NO_PROXY="localhost,127.0.0.1,::1" \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  -e CYCLONEDDS_URI=file:///home/cyf/cyclonedds.xml \
  -w /workspace \
  --device=/dev/bus/usb/002/002 \
  --device=/dev/ttyCH343USB0:/dev/ttyCH343USB0 \
  strongobserver/sm_uav_d435i_all_ok_1203:latest
  /*****************************************DDS*********************************************/
  export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
  echo $RMW_IMPLEMENTATION # 确认使用的是cyclone DDS 
  ros2 doctor --report | grep -i rmw   # 进一步确认是使用 cyclone DDS 

  cd realsense_driver/
  source install/setup.bash
  ros2 launch realsense2_camera rs_camera_vins.py
  /**********************************vins_fusion_gpu*****************************************/
  source ~/APP/opencv410/opencv410_env.sh
  cd vins_ws
  source install/setup.bash
  ros2 run vins vins_node /home/cyf/vins_ws/src/VINS-Fusion-ROS2/config/realsense_d435i/realsense_stereo_imu_config.yaml
```
  
残留问题：
1.“多播被防火墙/VPN/网络隔离”——我们用 CycloneDDS 显式配置绕过
2.bag包录不到rect_raw那个话题
注意使用riviz可视化的时候，需要改成world坐标系（默认是map）

---
1.4 解决 CycloneDDS 模式下笔记本接受不到话题的BUG & 找到动捕小球并安装& 研究磁罗盘校准为什么在QGC消失 & 固定好硬件 & 重新配对遥控器并校准 & 重新测试下UWB
笔记本端和NX端都运行
```
RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI=file:///home/cyf/cyclonedds.xml
```
cyclonedds.xml内容如下
```
<CycloneDDS>
  <Domain>
    <General>
      <AllowMulticast>true</AllowMulticast>
    </General>
    <Discovery>
      <Peers>
        <Peer address="192.168.8.101"/>
        <Peer address="192.168.8.100"/>
      </Peers>
    </Discovery>
  </Domain>
</CycloneDDS>
```
注意启动docker的命令也稍微改了一下，用下面的命令启动docker之后就不用执行RMW_IMPLEMENTATION=rmw_cyclonedds_cpp和export CYCLONEDDS_URI=file:///home/cyf/cyclonedds.xml了,因为已经在命令里设置好环境变量了:
```
sudo docker run --rm --runtime nvidia --gpus all --network=host --ipc host -it \
  --name vins_fusion_gpu_camera_driver_final \
  -v /home/cyf/my_ws:/workspace \
  -v /home/cyf/vins_ws:/root/vins_ws \
  -v /home/cyf/uwb_ws:/root/uwb_ws \
  -v /home/cyf/ch343ser_linux:/root/ch343ser_linux \
  -v /home/cyf/ros2_ws:/root/realsense_driver \
  -v /usr/lib:/usr/lib \
  -v /usr/local/lib:/usr/local/lib \
  -v /home/cyf/my_ws/src/opencv410/opencv-4.10.0/build/lib:/usr/local/opencv410/lib \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v /home/cyf/cyclonedds.xml:/home/cyf/cyclonedds.xml \
  -e DISPLAY=$DISPLAY \
  -e QT_X11_NO_MITSHM=1 \
  -e HTTP_PROXY="http://127.0.0.1:7897" \
  -e HTTPS_PROXY="http://127.0.0.1:7897" \
  -e NO_PROXY="localhost,127.0.0.1,::1" \
  -e RMW_IMPLEMENTATION=rmw_cyclonedds_cpp \
  -e CYCLONEDDS_URI=file:///home/cyf/cyclonedds.xml \
  -w /workspace \
  --device=/dev/bus/usb/002/002 \
  --device=/dev/ttyCH343USB0:/dev/ttyCH343USB0 \
  strongobserver/sm_uav_d435i_all_ok_1203:latest
```
super模式下各资源占用，运行良好：
<div align="center">
  <img width="1280" height="497" alt="image" src="https://github.com/user-attachments/assets/dfc185c9-fd21-4003-8b7b-f55843e76b15" />
</div>

2.6 整理底部走线布局 & 解决QGC不显示磁罗盘校准选项的问题 & 调节PX4参数重新调通nx与PX4的通信 & 重新设置好电机的转向
1.QGC不显示磁罗盘校准选项的问题
SYS_HAS_MAG设置为1就行了
2.调节PX4参数重新调通nx与PX4的通信
MAV_0_CONFIG默认是 TELEM1没问题
SER_TEL1_BAUD和SER_TEL2_BAUD都设置为921600
UXRCE_DDS_CFG设置为TELEM2

## 四、实现效果
1.建模-实物
<div align="center">
  <img src="https://github.com/user-attachments/assets/74fbecbc-f1c9-40ca-84e4-a3154275ef27" alt="3月5日 (1)(4)" />
</div>


# FAKE GPS

官网英文原文地址：http://dev.px4.io/advanced-fake-gps.html

This page shows you how to use mocap data to fake gps. 
本文将会向您展示如何使用motioncapture相机的数据来伪造gps数据（适用于室内等场景）-by yanhuizhang@nuaa.edu.cn
The setup looks as follows:
这里需要安装vicon相机 + [ROS](http://www.ros.org/) + [网络摄像机vicon](https://github.com/ethz-asl/vicon_bridge) 通过网络发送数据到你的PC端.
On "your computer" you should have ROS + MAVROS installed确认你的电脑已经安装了ROS和MAVROS.
In MAVROS there is a script which simulates gps data out of mocap data.
在MAVROS有一个脚本，使用相机数据来模拟gps数据。
"Your computer" then sends the data over 3DR radiometry to the pixhawk.
然后你的电脑通过3DR的电台发送数据到pixhawk。
*NOTE*: The "VICON computer" and "your computer" can be the same, of course.
注意:监视VICON的电脑当然也可以是你发送数据的电脑。
## Prerequisites必备软硬件环境

- MOCAP system (in this example, VICON is used)--动捕系统（本文以VICON为例进行说明）
- Computer with ROS, MAVROS and Vicon_bridge --运行ROSMAVROS和vicon网络的电脑
- 3DR radiometry set-- 电台

## Procedure步骤

### Step 1第一步

Make sure "your computer" is in the same network as the "VICON computer" (maybe you need a wireless adapter).
确认你的电脑和相机电脑处在同一个局域网中（可能需要无线适配器）
Create two files on the "VICON computer": "launch_fake_gps.sh" and "launch_fake_gps_distorted.sh" 
在相机电脑上创建两个文件："launch_fake_gps.sh" 和 "launch_fake_gps_distorted.sh"
Add the following two lines to the "launch_fake_gps.sh" file and replace xxx.xxx.x.xxx with the IP adress of "your computer" (you can get the IP adress by typing "ifconfig" in the terminal).
将以下两行添加到“launch_fake_gps.sh”文件中，并用IP地址“您的计算机”替换xxx.xxx.x.xxx（您可以通过在终端中键入“ifconfig”获取IP地址）。
```sh
export ROS_MASTER_URI=http://xxx.xxx.x.xxx:11311
roslaunch vicon_bridge vicon.launch $@
```

Next, add the following two lines to the "launch_fake_gps_distorted.sh" file and replace xxx.xxx.x.xxx with the IP adress of "your computer".
接下来，将以下两行添加到“launch_fake_gps_distorted.sh”文件中，并用IP地址“your computer”替换xxx.xxx.x.xxx。
```sh
export ROS_MASTER_URI=http://xxx.xxx.x.xxx:11311
rosrun vicon_bridge tf_distort $@
```

Put markers on your drone and create a model in the MOCAP system (later referred to as yourModelName).
将标记点放在无人机上，并在MOCAP系统中创建一个模型（以后作为您的模型名称）。
### Step 2第二步
在你的PC上运行

```sh
$ roscore
```


### Step 3第三步

在您创建两个文件的目录中的“VICON计算机”上的两个不同终端中
运行

```sh
$ sh launch_fake_gps.sh
```

和

```sh
$ sh launch_fake_gps_distorted.sh
```


### Step 4

在你的电脑上运行

```sh
$ rosrun rqt_reconfigure rqt_reconfigure
```

应该打开一个新窗口，您可以在其中选择“tf_distort”。使用此工具，您可以编辑参数来加工动捕数据。 
使用以下参数来模拟gps信号：

- publish rate = 5.0Hz
- tf_frame_in = vicon/yourModelName/yourModelName (e.g. vicon/DJI_450/DJI_450)
- delay = 200ms
- sigma_xy = 0.05m
- sigma_z = 0.05m

### Step 5第五步


连接pixhawk到QGC。点击参数选项->系统参数SYS_COMPANION 为 257600（开启魔法模式）

然后点击参数->mavlink  将MAV_USEHILGPS 改为1 （使能HIL GPS）

然后 参数 ->姿态四元数估计 并将 ATT_EXT_HDG_M 设为2（使用动捕系统的航向）

最后，参数->惯导估计参数 并改变 INAV_DISAB_MOCAP 改为1 （禁用动捕估计）

注意：如果发现找不到上述状态参数，检查参数->默认组
### Step 6第六步


打开"mocap_fake_gps.cpp"文件，你可以在yourCatkinWS/src/mavros/mavros_extras/src/plugins/mocap_fake_gps.cpp找到

替换DJI_450/DJI_450为你自己的模型名字（如：/vicon/yourModelName/yourModelname_drop），下一步将会解释"_drop" 的意义。
```sh
mocap_tf_sub = mp_nh.subscribe("/vicon/DJI_450/DJI_450_drop", 1, &MocapFakeGPSPlugin::mocap_tf_cb, this);
```

### Step 7

In step 5, we enabled heading from motion capture. Therefore pixhawk does not use the original North, East direction, but the one from the motion capture system. Because the 3DR radiometry device is not fast enought, we have to limit the rate of our MOCAP data. To do this run
在步骤5中，我们启用了从动捕。因此pixhawk不使用原北东方向，而是运动捕获系统的。因为3DR电台存在通信延迟，我们必须限制我们的MOCAP数据率。为此运行
```sh
$ rosrun topic_tools drop /vicon/yourModelName/yourModelName 9 10
```

这意味着，从rostopic / vicon / yourModelName / yourModelName中，10个消息中的9个将被删除并以主题名称“/ vicon / yourModelName / yourModelName_drop”发布。
### 第8步


使用电脑（USB）将3DR电台与pixhawk TELEM2连接。

### 第9步
切换到catkinWS然后运行

```sh
$ catkin build
```

然后运行

```sh
$ roslaunch mavros px4.launch fcu_url:=/dev/ttyUSB0:57600
```
现在你的pixhawk已经获得了gps数据并且指示灯为绿灯闪烁

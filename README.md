# HIT-heli-ROV 水下机器人，哈深河狸队

![](https://github.com/GeXu66/HIT-heli-ROV/blob/main/png/underwater-model.gif)


水下机器人模型百度网盘链接：
链接：https://pan.baidu.com/s/1DfWbYRaOcvUyCyCEIduUzQ?pwd=r28p 
提取码：r28p
![](https://github.com/GeXu66/HIT-heli-ROV/blob/main/png/underwater.gif)

## 电路部分

###  **1**、检测及控制设计思路

1）检测：

检测管道：用opencv分别对BGR图片中的G、R颜色空间进行二值化，然后进行与运算得到比较稳定的二值化图像，从而可以检测出管道。得到管道图像后，对所有的白色点用numpy进行拟合可以得到管道简化为的走向图，然后用中心点创建垂直的直线判断管道是左倾还是右倾，然后用中心点与图像中间的间距判断机器人的水平位置偏移量。

检测黑色物块：用yoloV5目标检测模型进行目标检测。

(2) 上位机控制：

上位机：Jetsonnano

信息获取：获取图像，并将图像传给检测代码并得到返回信息；从串口读取stm32传上来的深度信息和IMU方位角信息

过程控制:使用增量式pid对上述信息进行处理控制电机的转速，使用ros的串口通讯将电机转速传给stm32

(3) 下位机控制：

下位机：STM32F103ZET6最小系统板

信息获取与发送：通过设计通讯数据包，使用STM32与上位机Jestonnano进行串口通讯，获取六个推进器的转速和控制标志，将转速传给推进器实现运动；同时STM32分别通过串口中断和轮询获取IMU的欧拉角信息和深度传感器的深度信息，并将其通过串口发送给上位机。过程控制：通过STM32定时器产生PWM波输出给推进器连接的电调，驱动推进器工作

![](https://github.com/GeXu66/HIT-heli-ROV/blob/main/png/%E6%95%B4%E4%BD%93%E7%BB%93%E6%9E%84.png)

###   **2**、器件选择及实施方案

1. 推进器：选用与30A电调相连的2216型推进器马达,通过STM32输出50HzPWM波进行控制

2. IMU:选用 HiPNUC 生产的9轴姿态传感器HI229，进行配置后利用串口中断读取其输出的欧拉角信息

3. 深度传感器：选用MS5837压力传感器，通过串口轮询读取其输出的温度和深度信息

4. 下位机：选用STM32F103ZET6最小系统板

5. 电源：选用11.6V的4S电池为推进器供电，使用5V输出的充电宝为Jestonnano供电，STM32F103的供电由Nano引出，其他传感器的电源从STM32引出

   ![](https://github.com/GeXu66/HIT-heli-ROV/blob/main/png/%E6%9D%BF%E5%AD%90.png)

###   **3**、总结和体会

（1）管道检测：首先采用的想法是用图像分割模型，例如Unet，DeeplabV3+等，但是运行后发现对算力要求太大，如果同时使用目标检测模型和图像分割模型，一秒仅有几帧图像，远远无法满足检测要求。而且用opencv进行传统的形态学处理后发现得到的结果几乎和图像分割没有太大区别。所以我们放弃了图像分割模型，使用形态学处理+目标检测yolov5模型。形态学处理采用了分离颜色空间，再进行二值化+与运算而非直接灰度处理，高斯模糊+二值化。前者得到的图像更加稳定。然后用numpy的polyfit进行拟合求解方向。

（2）物体检测

物体检测采用了新发布的yolov5目标检测模型。首先，这是YOLO家族中，第一个先用PyTorch的原生版本，而非PJ Reddie的Darknet编写的模型。Darknet是一个非常灵活的研究框架，但它并没有考虑到生产环境的构建，而且用户社区也较小。这导致Darknet需要在配置上花费不少功夫，而且生产准备不足。

由于YOLOv5是在PyTorch中实现的，它受益于成熟的PyTorch生态系统：支持更简单，部署更容易。此外，作为一个更广为人知的研究框架，YOLOv5 的迭代对更广泛的研究社区来说可能更容易。这也使得部署到移动设备上更加简单，因为该模型可以轻松编译成ONNX和CoreML。

其次，YOLOv5的速度快得惊人。在YOLOv5 Colab notebook上，运行Tesla P100，每张图像的推理时间仅需0.007秒，这意味着每秒140帧（FPS）。相比之下，YOLOv4在被转换到相同的Ultralytics PyTorch库后的速度是50FPS。YOLOv5的速度是YOLOv4的2倍还多。

第三，YOLOv5精度超高。在Roboflow对血细胞计数和检测（BCCD）数据集的测试中，只训练了100个epochs就达到了大约0.895的平均精度（mAP）。诚然EfficientDet和YOLOv4的性能相当，但在准确率没有任何损失的情况下，看到如此全面的性能提升是非常罕见的。

第四，YOLOv5的体积很小。具体来说，YOLOv5的权重文件是27兆字节。YOLOv4（采用Darknet架构）的权重文件是244兆。YOLOv5比YOLOv4小了近90%。这意味着YOLOv5可以更容易地部署q到嵌入式设备上。

<img src="https://github.com/GeXu66/HIT-heli-ROV/blob/main/png/%E8%AF%AF%E5%B7%AE%E5%9B%BE.png" style="zoom:48%;" />

![](https://github.com/GeXu66/HIT-heli-ROV/blob/main/png/%E8%A7%86%E8%A7%89%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%9B%BE.jpg)

<img src="https://github.com/GeXu66/HIT-heli-ROV/blob/main/png/STM32%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%9B%BE.png" style="zoom:75%;" />

### 4.说 明

（1）STM32控制部分：  STM32主要负责推进器的驱动和传感器信息的获取，在程序中，首先对STM32各项基本外设进行初始化，然后通过IO口和串口对推进器和传感器进行初始化配置，配置完成后，STM32定时获得传感器数据，并通过串口通讯协议以数据包的形式发送给上位机。同时，STM32会持续等待上位机发送的速度和状态控制信息，并更新对推进器的控制信号

（2）视觉控制部分：

由摄像头不断捕获水下图像，然后用OpenCv提取G、R颜色空间并进行二值化，然后进行与运算，得到管道为白，其余为黑的单通道二值化图像。随后进行numpy的直线拟合，得到管道的走势和水平偏移，从而控制机器人运动。在运动过程中对黑色点进行判断，达到一定阈值即判断为有黑色物体，然后加载yolov5模型文件进行目标检测，进行蜂鸣器报警和RGB灯闪烁。  （3）jetson  nano控制部分：  

1.信息获取：摄像头信息，通过串口通讯从stm32获取的方位角信息和深度信息
2.管道检测模式：将信息传给视觉检测程序获取当前管道位置，将管道位置和从stm32获取的信息和当前电机转速传入pid控制程序中，获取下一步的电机转速。
3.物体检测模式：当视觉检测程序检测到图像中有物块时，程序会实时检测目标物块位置，并在机器人到达物块正上方时停下来，进行报警和识别，并控制机器人下潜完成回收操作。

![](https://github.com/GeXu66/HIT-heli-ROV/blob/main/png/Jetson%20nano%E6%8E%A7%E5%88%B6%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

### 5.电路设计方案创新特色说明

电路设计：  选用STM32F103ZET6核心板，引出必要的IO口，且IO接口设计能用于方便合理接线  理由如下：  (1)引出TIM2-TIM3的PWM通道且安排与相近位置，方便推进器驱动信号统一接线  (2)引出五路串口UART1-UART5,且大部分串口RX-TX引脚位置相邻，方便与通讯模块连接  (3)核心板预留出5V与3.3V电源输出，便于给深度传感器和IMU供电，无需再引入电源模块

![](https://github.com/GeXu66/HIT-heli-ROV/blob/main/png/final.png)

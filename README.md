# WFR07_decode
WFR07 7通道接收机解调，PPM转换成串口数字信号。  
本工程在windows下如果出现乱码等问题，请尝试更换编辑器，如notepad++。  
在tools目录下有一个python小程序可以直观地显示出各个通道的数据。

**我使用的是美国手（左边油门）遥控器，后面的功能介绍都是基于美国手遥控器。**  
对于各个通道信号解码来说并不关心各个通道的意义，解码后每个通道的作用由你自己后续的程序决定。

对于航模的多通道遥控器，通常有 直升机 和 固定翼 两种模式，在直升机模式下各个通道会相互关联，在固定翼模式下各个通道独立。  
**解码的目的是在接收端还原遥控器的操作，所以选择各个通道独立操作的固定翼模式，以下所有操作都在固定翼模式下。**

# 功能
WFR07是天地飞（WFly）7通道航模遥控器的接收机，其7个通道输出PPM信号。  
本工程使用单片机提取其7个通道的信号，解码后以数字信号通过UART串口输出。

# WFR07特性
关于WFT07遥控器和WFR07接收机的详细介绍可访问 <http://nicekwell.net/blog/20161223/wft07he-wfr07de-gong-neng-he-shi-yong.html>。  
关于PPM信号的详细介绍可访问 <http://nicekwell.net/blog/20161223/ppmxin-hao-jie-shao.html>。

7个通道的默认配置：

通道 | 功能 | 备注 | 信号特性
:-: | :-: | :-: | :-:
CH1 | 副翼 | 右手左右调节 | 1000us~2000us连续变化，**精度20us**
CH2 | 升降舵 | 右手上下调节 | 1000us~2000us连续变化，**精度20us**
CH3 | 油门 | 左手上下调节 | 1000us~2000us连续变化，**精度20us**
CH4 | 方向舵 | 左手左右调节 | 1000us~2000us连续变化，**精度20us**
CH5 | 起落架 | 一个三档开关 | 1000us、1500us、2000us间切换
CH6 | 螺距 | 连续可调旋钮 | 1000us~2000us连续变化，**精度20us**
CH7 | 辅助通道 | 一个三档开关 | 1000us、1500us、2000us间切换

由于输出精度是20us，所以单片机在采集信号时精度达到20us即可。

**示波器测量所有通道周期是21.2ms**

每个通道的高电平脉宽范围大约在1000us~2000us之间。  
在一个周期内，各个通道的高电平脉冲一个紧接着一个依次输出，7个通道信号全部输出结束最多约占用2ms*7=14ms，周期内剩余约7ms的时间所有通道输出全为低电平。

# 硬件
【单片机】STC12C5A60S2  
【晶振】24MHz  
注：此晶振可产生精确地定时器中断，方便监测各个通道，但串口波特率会有0.16%的误差，不会影响使用。  
【引脚连接】  
CH1：P1.6  
CH2：P1.5  
CH3：P1.4  
CH4：P1.3  
CH5：P1.2  
CH6：P1.1  
CH7：P1.0  
TXD：P3.1

# 输出格式
【波特率】115200  
实测发送一个字节大约需要13us，这样算的话一帧发送8字节大约需要104us。  
这里测量的13us是程序把一字节数据送入缓存，并等待发送完成标志所用的时间，不是实际串口的工作时间。
【数据格式】  
每个周期内，当采集完7个通道的高电平后（最长约14ms）会立刻通过串口发送7个通道的数据信息。  
每个周期的数据为一帧，一帧数据有8个字节：  
第一字节固定为0x01，标志一帧数据开始。（后面7个字节不可能为这个值）  
后面7个字节依次表示CH1到CH7的脉宽，单位是10us。如输出150表示脉宽为1500us。  
注：  
1、接收机输出的脉宽范围大约在1000us~2000us之间，所以7个脉宽的数据范围大约在100~200之间。  
2、解码后输出的数据单位是10us，但实际接收机输出的精度是20us，单片机程序也是按照20us的精度采样的。

对于此接收机，不会出现信号丢失的情况，当遇到遥控器信号丢失时，接收机会输出预先设定好的信号，对于解码器来说不能区分当前遥控器信号是否丢失。

# 程序结构介绍
两个进程：定时器中断和主循环。

定时器20us一次中断，有两个状态：  
1、信号采集中：  
&emsp;&emsp;1、采集各个通道高电平时间。  
&emsp;&emsp;2、判断当前所有通道是否采集完成（所有通道信号结束后，所有通道都会输出低电平。  
&emsp;&emsp;&emsp;&emsp;如果连续100us（5个周期）检测到所有通道都是低电平，则认为一帧信号结束，此时对采集到的信号进行判断：  
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;如果所有通道脉宽大于500us，则认为数据有效，通知主进程发送数据，并进入状态2。  
2、本周期信号已结束，等待下一周期：  
&emsp;&emsp;任意通道采集到高电平则进入状态1。

主循环进程只干一件事，等待定时器进程发送指令，接收到指令后发送数据。  
但主循环会忽略第一帧数据，因为第一帧数据可能采集不完整。

更多关于单片机编程结构的文章请访问 <http://nicekwell.net/pages/dan-pian-ji-bian-cheng.html>

做好之后效果演示视频：<http://v.youku.com/v_show/id_XMTg3OTU4ODYxNg==.html>


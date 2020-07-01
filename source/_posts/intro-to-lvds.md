---
title: 浅谈LVDS
date: 2020-01-11 04:04:05
tags: LVDS
category: 嵌入式系统
---

## 差分信号与低压差分信号  

LVDS，全称Low-Voltage Differential Signaling，即“低电压差分信号”，是一种在功耗、误码率、串扰和辐射等方面非常优越的差分信号技术。想要介绍LVDS就不得不从差分信号开始讲起。  

差分传输是一种高速信号传输的技术，区别于传统的一根信号线一根地线的做法，差分传输在这两根线上都传输信号，这两个信号的振幅相同，相位相反。在这两根线上的传输的信号就是差分信号。信号接收端比较这两个电压的差值来判断发送端发送的逻辑状态。  

<div align="center">
<img src = "ds.png" width = 50% height = 50% alt="差分信号示意图">
<p><i>差分信号示意图，A为差分信号，B为差分信号A的实际载荷</i></p>
</div>

差分信号和普通的单端信号走线相比，最明显的优势体现在以下三个方面[<sup>1</sup>](#refer-1)：  

1. 抗干扰能力强。因为两根差分走线之间的耦合很好，当外界存在噪声干扰时，几乎是同时被耦合到两条线上，而接收端关心的只是两信号的差值，所以外界的共模噪声可以被完全抵消。  
<div align="center">
<img src = "ds-jamming.png" width = 50% height = 50% alt="抵消信号干扰">
<p><i>差分信号对可以抵消信号干扰</i></p>
</div>

2. 能有效抑制 EMI。两条信号线相互平行，由于两根信号的极性相反，他们对外辐射的电磁场可以相互抵消，耦合的越紧密，泄放到外界的电磁能量越少。而且因为相位总是相反，差分信号变化时电流的变化也很小。  
<div align="center">
<img src = "ds-magnaticfield.png" width = 50% height = 50% alt="感应磁场相互抵消">
<p><i>差分信号产生的感应磁场会相互抵消且消耗电流不变</i></p>
</div>

3. 时序定位精确。由于差分信号的开关变化是位于两个信号的交点，而不像普通单端信号依靠高低两个阈值电压判断，因而受工艺，温度的影响小，能降低时序上的误差，同时也更适合于低幅度信号的电路。  

<p></p>
<p></p>
如下图，在dV / dt = 1.32 V/ns的升压速度下，当信号幅度为3.3 V时，从低电平（20%）翻转到高电平（60%）需要1 ns，而如果信号幅度只有500 mV时，从低电平（20%）翻转到高电平（80%）仅需要303ps。通过这样的方式，可以大幅提升差分信号的频率，从而提升传输速度。  

<div align="center">
<img src = "ds-voltage.png" width = 50% height = 50% alt = "电平翻转时间">
<p><i>*相同升压速度下，不同电压的差分信号的电平翻转时间</i></p>
</div>

所以说与普通差分信号的相比，LVDS低电压差分信号，是一种在功耗、误码率、串扰和辐射等方面更加优越的差分信号技术。因为采用的电压更小，使得它的电平翻转速度更快。  

## LVDS总线  

广义的LVDS总线分很多类型。针对实际应用的需求，不同的总线可以承载不同大小的数据流。在350mV的典型信号振幅下，LVDS总线只会消耗非常少量的能量，而与此相对的，LVDS总线的数据传输速率可以达到相当可观的3.125 Gbps。如下图，展示了LVDS总线的传输速率和总线长度的关系。  

<div align="center">
<img src = "lvds-compare.png" width = 50% height = 50% alt = "LVDS">
<p><i>典型传输速率和接线长度</i></p>
</div> 

更准确地来说，LVDS是一个物理层协议，它并不只是一个视频总线。事实上，在LVDS上可以传输各种协商好的上层数据。一般来讲，如果只使用它做点对点的数据传输，它只有两部分组成：发送者和接收者。而这两部分又分别起到了将上层协议转换成LVDS内传输的比特流的“编码器”Serializer和与之相反的“解码器”Reserializer[<sup>2</sup>](#refer-2)。  

<div align = "center">
<img src = "lvds-consist.png" width = 50% height = 50% alt = "LVDS的两个主要组成部分">
<p><i>LVDS的两个主要组成部分</i></p>
</div>

LVDS总线的一大特点就是，当一对差分信号无法提供充足的数据带宽时，可以增加同时工作的差分信号对，以并行的思路传输信号。如下两图所示，LVDS可以是根据场合选择相应的通道数和编码方式的，使用起来非常自由。这同时也造成实际设计中必须协商好LVDS上的上层协议，才能正确传达信息。  

<div align = "center">
<img src = "lvds-1channel.png" width = 50% height = 50% alt = "lvds实际使用">
<p><i>单通道LVDS</i></p>
</div>  
<div align = "center">
<img src = "lvds-3channel.png" width = 50% height = 50% alt = "lvds实际使用">
<p><i>三通道LVDS</i></p>
</div> 

## OLDI总线  

在视频信号传输相关的场景下所使用的LVDS总线，常常遵循一种叫OpenLDI的协议，即Open LVDS Display Interface，缩写成OLDI。它自然也遵循LVDS物理层的基本设计，如下图。  

<div align = "center">
<img src = "oldi-basiclogic.png" alt = "OLDI">
<p><i>OLDI协议的基本内部逻辑</i></p>
</div>  

OLDI协议中，LVDS接口基本分这样的几类：单路6 bit、双路6 bit、单路8 bit、双路8 bit。  
比如说单路6 bit：它采用单路方式传输，一次只传输一个像素。注意，这里的6 bit是指RGB的三个色彩通道的色彩采样深度均为6 bit。这样一个周期只传输18 bit的数据，因此，也称18 bit LVDS接口。  
而在双路6 bit中，信号采用双路方式传输，也就是说一次性传输两个像素。这样一来奇路数据为18位，偶路数据为18位，一个周期共传输36 bit 数据。因此，也称36 bit LVDS接口[<sup>3</sup>](#refer-3)。  
另外每一路信号都需要一对差分信号来传输时钟信号，一个时钟周期发送一个像素。  

| LVDS接口类型 | 色彩深度 | 一个周期传输像素数 | 使用差分信号对数 |
| :-: | :-: | :-: | :-: |
| 18 bit | 6 bit | 1 | 4 |
| 36 bit | 6 bit | 2 | 8 |
| 24 bit | 8 bit | 1 | 5 |
| 48 bit | 8 bit | 2 | 10 |

以下则是18 bit和24 bit的LVDS编码格式，可以看出，18 bit相当于JEIDA格式24 bit LVDS去掉了一对差分信号。事实上，通过去掉低位的信号，可以实现直接从8 bit的色彩深度直接降到6 bit。  

<div align = "center">
<img src = "oldi-24bit.png" width = 50% height = 50% alt = "OLDI-24bit">
<p><i>24 bit LVDS的两种编码格式：左 – JEIDA，右 - VESA</i></p>
</div>  
<div align = "center">
<img src = "oldi-18bit.png" width = 50% height = 50% alt = "OLDI-18bit">
<p><i>18 bit LVDS的编码格式：JEIDA</i></p>
</div>  

使用双路传输的时候，相当于把单路LVDS的信号线再复制一份，一路传输奇数位置的像素，另一路传输偶数位置的像素。  

<div align = "center">
<img src = "oldi-2channel.png" width = 50% height = 50% alt = "OLDI双路">
<p><i>双路情形下的LVDS，左 – JEIDA，右 – VESA</i></p>
</div>  

## 参考文献  

<div id="refer-1"></div>  
- [1] 辻嘉樹. 差動伝送の基本「LVDS技術」徹底理解 [M/OL]. レクロイ･ジャパン株式会社.  
https://teledynelecroy.com/japan/pdf/semi/cq2008-tech-semi.pdf  

<div id="refer-2"></div>  
- [2] LVDS Owner's Manual Design Guide, 4th Edition [M/OL]. Texas Instruments.  
https://www.ti.com/lit/snla187  

<div id="refer-3"></div>  
- [3] LVDS Display Interface (LDI) TFT Data Mapping for Interoperability wFPD-Link. [M/OL]. Texas Instruments.  
https://www.ti.com/lit/pdf/snla014  
## 前言

找到一篇自己五年前写的老文章，哈哈哈，重新在GitHub发一版^_^。这里跟大家分享下关于使用纯CSS来进行图形的绘制，本篇文章根据《CSS Secrets 第三章 形状》结合W3C的官方文档来对CSS属性绘制图形进行剖析。

## CSS盒子模型

首先这里对CSS的盒子模型进行一个简单介绍。如上图所示每一个CSS盒子由 Margin、Border、Padding、Content四个部分组成。
这四个部分分别处于CSS盒子模型的外围、边框、内边、内容位置。这四个部分每一个部分都单独可以分解成top、right、bottom、left。单独的部分组合在一起也就行成的CSS盒子的Margin、Border、Padding、Content。正是因为其每一个部分都可以分解成top、right、bottom、left四个部分也就为CSS绘制三角形提供了理论基础，后面在三角形篇章会有提到。
这里需要注意的是Margin是可以为负值的，这里可以在W3C官方文档找到介绍

## 

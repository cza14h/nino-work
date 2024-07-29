---
title: 用Rust软渲染器学习笔记(零)--数学工具包
date:  2024-07-26T15:05:23+08:00
draft: true
tags:
  - frontend
  - computer graphics
  - rust
---

## 背景

学习3D软渲染器作为入门计算机图形学时理解渲染管线的基础, 通常使用C++作为开发语言, 黑魔法与环境门槛都比较高. 而最近正好也想接触一下Rust, 就开了这个系列作为笔记, 方便日后重温

## 向量与矩阵
### 向量与标量
标量只是一个数字, 定义了一个大小.

向量除了定义了大小, 还一同附加了方向的属性, 根据不同的维度, 在渲染空间中常用的场景有
- 二维向量: 纹理坐标
- 三维向量: 位置信息, 法向量信息
- 四维向量: 带透明度的颜色信息, 以及三维坐标的齐次坐标


### 定义向量



## 参考
- Tiny renderer or how OpenGL works: software rendering in 500 lines of code: <https://github.com/ssloy/tinyrenderer/wiki>
- bilibili "单身剑法传人" 软渲染系列视频: <https://www.bilibili.com/video/BV1Nv4y1p79o/>
- GAMES101 现代计算机图形学入门: <https://sites.cs.ucsb.edu/~lingqi/teaching/games101.html>
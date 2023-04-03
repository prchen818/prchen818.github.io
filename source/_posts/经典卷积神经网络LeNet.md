---
title: 经典卷积神经网络LeNet
date: 2023-03-29 14:06:52
tags: [CNN, Machine-Learning, python]
categories: 笔记
---

# 经典卷积神经网络LeNet

## 简介
这个模型是由AT&T贝尔实验室的研究员Yann LeCun在1989年提出的（并以其命名），目的是识别图像 (LeCun et al., 1998)中的手写数字。

## 架构
总的来看，LeNet由两个部分组成
- 卷积编码器：由两个卷积层组成
- 全连接层密集块：由三个全连接层组成

![](/images/LeNet.png)

## Pytorch实现
```python
import torch
from torch import nn

net = nn.Sequential(
    nn.Conv2d(1, 6, kernel_size=5, padding=2), nn.Sigmoid(),
    nn.AvgPool2d(kernel_size=2, stride=2),
    nn.Conv2d(6, 16, kernel_size=5), nn.Sigmoid(),
    nn.AvgPool2d(kernel_size=2, stride=2),
    nn.Flatten(),
    nn.Linear(16 * 5 * 5, 120), nn.Sigmoid(),
    nn.Linear(120, 84), nn.Sigmoid(),
    nn.Linear(84, 10))

```


## CUDA环境安装
***wsl下安装cuda环境***
[知乎教程](https://zhuanlan.zhihu.com/p/488731878)

***安装CUDA Toolkit***
[CUDA Toolkit官方链接](https://developer.nvidia.com/cuda-downloads)

注意选择WSL-ubuntu发行版而不是Ubuntu版本

## Debug
### GPU训练报错``Could not load library libcudnn_cnn_infer.so.8``
缺少环境变量，找不到库文件

直接使用网友给的变量地址：
添加到``~/.bashrc``末尾
```bash
export LD_LIBRARY_PATH=/usr/lib/wsl/lib:$LD_LIBRARY_PATH
```
在终端中输入```source ~/.bashrc```刷新环境变量
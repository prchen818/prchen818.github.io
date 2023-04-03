---
title: Pytorch API文档（笔记版）
date: 2023-03-30 16:27:14
tags: [Pytorch, python, Machine-Learning]
categories:
- 笔记
---

# Pytorch Api 记录
## 激活函数
```python
import torch
from torch import nn

nn.Relu() # ReLU(x) = max(x, 0)
nn.sigmoid() # sigmoid(x) = 1/(1 + exp(-x))
nn.tanh() # tanh(x) = (1 - exp(-2x))/(1 + exp(-2x))
```
## Module
### nn.Module
所有神经网络模块的基类，需要实现``__init__(self)``和``forward(self, x)``
```python
from torch import nn
from torch.nn import functional as F

Class Model(nn.Module):
    def __init__(self):
        super(Model, self).__init__()
        self.conv1 = nn.Conv2d(1, 20, 5)
        # more operation for example:
        # self.conv2 = nn.Conv2d(20, 20, 5)

        # 也可以通过add_module添加
        # self.add_module("conv3", nn.Conv2d(20, 20, 5))

    def forward(self, x):
        return F.ReLU(self.conv1(x))
```




### nn.Conv2d
二维卷积层, 输入(N, c, h_in, w_in), 输出(N, c, h_out, w_out)
```python
layer = nn.Conv2d(
    in_channels, 
    out_channels, 
    kernel_size,    # 卷积核大小
    stride=1,       # 步长
    padding=0,      # 填充
    dilation=1, 
    groups=1, 
    bias=True)      # 偏移

imput = torch.rand(size=(batchsize, channels, h_in, w_in))
output = layer(imput)
```

## 池化层
主要用于消除平移误差
### nn.MaxPool2d
二维最大池化 
```python
nn.MaxPool2d(
    kernel_size,    # 池化窗口大小
    stride=None,    # 步长，默认为kernel_size
    padding=0,
    dilation=1, 
    return_indices, 
    ceil_mode=False # 默认向下取整，为True时向上取整
)
```

### nn.AvgPool2d
二维平均池化
```python
nn.AvgPool2d(
    kernel_size,    # 池化窗口大小
    stride=None,    # 步长，默认为kernel_size
    padding=0,
    dilation=1, 
    ceil_mode=False # 默认向下取整，为True时向上取整
    count_include_pad=True   # 为False时，计算平均池化时不包括填充的0
)
```


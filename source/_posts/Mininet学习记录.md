---
title: Mininet学习记录
date: 2023-10-08 13:38:47
tags:
---

# Mininet / Ryu 实验踩坑记录

## 实验过程

**胖树拓扑结构**
![](./images/fattree.jpeg)

```python
from mininet.topo import Topo
from mininet.net import Mininet
from mininet.node import RemoteController
from mininet.link import TCLink
from mininet.util import dumpNodeConnections


class MyTopo(Topo):

    def __init__(self):
        super(MyTopo, self).__init__()

        # Marking the number of switch for per level
        L1 = 2
        L2 = 4
        L3 = 4
        L4 = 8

        # Starting create the switch
        c = []  # core switch
        a = []  # aggregate switch
        e = []  # edge switch
        h = []  # host

        # notice: switch label is a special data structure
        for i in range(L1):
            # label from 1 to n,not start with 0
            c_sw = self.addSwitch('c{}'.format(i+1))
            c.append(c_sw)

        for i in range(L2):
            a_sw = self.addSwitch('a{}'.format(L1+i+1))
            a.append(a_sw)

        for i in range(L3):
            e_sw = self.addSwitch('e{}'.format(L1+L2+i+1))
            e.append(e_sw)

        # Starting create the link between switchs
        # first the first level and second level link
        for i in range(L1):
            for j in range(4):
                self.addLink(c[i], a[j])

        # second the second level and third level link
        for i in range(L2):
            for j in range(2):
                self.addLink(a[i], e[(i//2)*2+j])

        # Starting create the host and create link between switchs and hosts
        for i in range(L3):
            for j in range(2):
                hs = self.addHost('h{}'.format(i*2+j+1))
                h.append(hs)
                self.addLink(e[i], hs)


topos = {"mytopo": (lambda: MyTopo())}

```

## 启动命令
### Mininet


```bash
mn --switch ovsk,protocols=OpenFlow13 --custom test.py --topo mytopo --controller=remote
```

### RYU
```bash
ryu-manager --observe-links xxx.py xxx.py
```

## 踩坑

### 主机与交换机ping通，但主机之间ping不同
原因是mininet默认使用的交换机协议版本可能是OpenFlowV1.0，使用以下命令启动Mininet

```bash
mn --switch ovsk,protocols=OpenFlow13
```

## Linux 常用命令附录

```bash
ps -aux # 列出所有进程
netstat -nlp # 查看网络状态
grep xxx # 关键词过滤

```
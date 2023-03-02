---
title: ubuntu安装miniconda
date: 2023-02-19 16:52:17
tags: 
    - python
    - conda 
    - linux
    - ubuntu
categories:
    - 教程
---

# ubuntu安装miniconda

## 下载miniconda安装脚本
```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-py39_4.12.0-Linux-x86_64.sh
```
或
```bash
wget -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/miniconda/Miniconda3-latest-Linux-x86_64.sh
```
## 添加执行权限
```bash
chmod 777 Miniconda3-latest-Linux-x86_64.sh
```

## 安装
```bash
bash Miniconda3-latest-Linux-x86_64.sh
```

无论使用什么shell，都应该包含 bash 命令。

默认安装目录为 /home/\<username\>/miniconda3

## 使用
创建环境
```bash
conda create --name <name> python=[python_version]
```

进入环境
```bash
conda activate <name>
```

退出环境
```bash
conda deactivate
```

删除环境
```conda remove --name <name> --all```

查看所有环境
```conda env list```



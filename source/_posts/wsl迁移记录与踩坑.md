---
title: wsl迁移记录与踩坑
date: 2023-05-04 10:27:18
categories: debug
tags: 
- wsl
- docker-desktop
- ubuntu
---

# wsl迁移记录与踩坑

## 动机
**众所周知**，Windows用户的C盘空间往往比较金贵，而抛开QQ&Wechat的大量垃圾文件之外，WSL也是一个硬盘需求大户，因此普遍有将WSL迁移至其他盘中的需求

另外，由于Docker-Desktop同样基于WSL，且Docker镜像所占的空间往往更大，更迫切的需要迁移，本文方法对Docker-Desktop同样适用

## 步骤
1. 找到你的wsl发行版，例如``Ubuntu-20.04``
    
    ```
    wsl -l
    ```

2. 停止正在运行的wsl

    ```
    wsl --shutdown
    ```

3. 导出WSL镜像

    ```
    wsl export Ubuntu-20.04 {备份路径}\ubuntu.tar
    ```

4. 注销原有镜像

    ```
    wsl unregister Ubuntu-20.04
    ```

5. 重新导入wsl

    ```
    wsl import 自定义名称 {新的安装路径} {备份路径}\ubuntu.tar --version 2
    ```

6. 恢复默认账户

    ```
    ubuntu2004 config --default-user 原来的用户名
    ```

## 踩坑记录

### 导入导出很久
应该是正常现象，无需中断，等待即可，作者24GB的镜像导出大约花费了10分钟，中途命令行无任何提示，但可以查看ubuntu.tar文件的大小发现导出还是在进行中的

### 默认发行版
```powershell
PS C:\Users\12545> wsl -l

适用于 Linux 的 Windows 子系统分发:
Ubuntu-20.04 
docker-desktop 
docker-desktop-data (默认)
```
重新安装后wsl默认分发可能变成其他子系统，需要修改回来

```
wslconfig /setdefault Ubuntu-20.04
```


### 恢复默认账户出错

<a style='color:red'>*ubuntu:无法将“ubuntu”项识别为cmdlet、函数、脚本文件或可运行程序的名称。*</a>

步骤6中``ubuntu2004``应为你安装的具体wsl发行版版本，可以在``C:\Users\你的用户名\AppData\Local\Microsoft\WindowsApps``中寻找对应的.exe文件
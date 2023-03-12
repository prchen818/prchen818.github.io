---
title: wsl2使用中遇到的网络大坑
date: 2023-03-12 13:00:40
tags: 
- wsl 
- 微服务
- network
- grpc
categories:
- debug
---
# wsl2使用中遇到的网络大坑

事情起因是这样的...

我手里有一个使用grpc做服务间调用的微服务项目
团队里其他成员用python建了个模型，并开了gRPC服务接口供我调用

众所周知...

Linux拥有领先Windows几百年的包管理器以及命令行工具（雾
因此我自然的使用的WSL当中的conda环境

然鹅...伏笔了...

在千辛万苦搞定了Java的gRPC客户端代码之后

```log
io.grpc.StatusRuntimeException: UNAVAILABLE: io exception

	at io.grpc.stub.ClientCalls.toStatusRuntimeException(ClientCalls.java:271)
	at io.grpc.stub.ClientCalls.getUnchecked(ClientCalls.java:252)
	at io.grpc.stub.ClientCalls.blockingUnaryCall(ClientCalls.java:165)
	at com.citicup.backend.protobuf.price_evaluateGrpc$price_evaluateBlockingStub.greet(price_evaluateGrpc.java:157)
	at com.citicup.backend.service.GrpcService.evaluate(GrpcService.java:32)
	at com.citicup.backend.BackendApplicationTests.test3(BackendApplicationTests.java:46)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:77)
	...
	at com.intellij.rt.junit.JUnitStarter.main(JUnitStarter.java:54)
Caused by: io.grpc.netty.shaded.io.netty.channel.AbstractChannel$AnnotatedConnectException: Connection refused: no further information: localhost/127.0.0.1:50050
Caused by: java.net.ConnectException: Connection refused: no further information
	at java.base/sun.nio.ch.Net.pollConnect(Native Method)
	at java.base/sun.nio.ch.Net.pollConnectNow(Net.java:672)
	at java.base/sun.nio.ch.SocketChannelImpl.finishConnect(SocketChannelImpl.java:946)
	at io.grpc.netty.shaded.io.netty.channel.socket.nio.NioSocketChannel.doFinishConnect(NioSocketChannel.java:337)
	at io.grpc.netty.shaded.io.netty.channel.nio.AbstractNioChannel$AbstractNioUnsafe.finishConnect(AbstractNioChannel.java:334)
	at io.grpc.netty.shaded.io.netty.channel.nio.NioEventLoop.processSelectedKey(NioEventLoop.java:710)
	at io.grpc.netty.shaded.io.netty.channel.nio.NioEventLoop.processSelectedKeysOptimized(NioEventLoop.java:658)
	at io.grpc.netty.shaded.io.netty.channel.nio.NioEventLoop.processSelectedKeys(NioEventLoop.java:584)
	at io.grpc.netty.shaded.io.netty.channel.nio.NioEventLoop.run(NioEventLoop.java:496)
	at io.grpc.netty.shaded.io.netty.util.concurrent.SingleThreadEventExecutor$4.run(SingleThreadEventExecutor.java:997)
	at io.grpc.netty.shaded.io.netty.util.internal.ThreadExecutorMap$2.run(ThreadExecutorMap.java:74)
	at io.grpc.netty.shaded.io.netty.util.concurrent.FastThreadLocalRunnable.run(FastThreadLocalRunnable.java:30)
```

关于``io.grpc.StatusRuntimeException: UNAVAILABLE: io exception``这个报错，搜索之后有个博客说的很有道理：

    1、检查下IP是否能ping通，IP、端口 是否正确
    2、Server是否打开
    3、连接中如果有证书，证书是否有效
    4、无证书的，是否写了明文连接，例如：

我当时的客户端配置是这样的

```yaml
grpc:
    client:
        price_prediction:
            address: static://localhost:50051
            negotiation-type: plaintext
        price_evaluate:
            address: static://localhost:50050
            negotiation-type: plaintext
```

对于``2``，显然不应该有问题，PyCharm还没关呢，使用py编写的Client测试也没有问题
对于``3``和``4``，倒是确实提醒了我，我这哪来的ssl，遂将传输层协议改为了plaintext，but报错依旧是这些内容，一个字儿都没变
对于``1``，我在本地跑的服务，localhost访问这不可能存在网络问题吧，压根想都没想过

折腾了一晚上，把报的错反复搜来搜去，各种方法试了个遍，他妈的报错内容就是一个字儿都没变，给我急得

最后就看着```Connection refused: no further information```这玩意发呆，就这玩意感觉最扎眼

refused...不会是防火墙连本机网络访问都给拦了吧，这没道理的，改了规则，甚至关了防火墙，然而依旧...

突然想到会不会是wsl的问题，之前玩VMWare的时候有了解过所谓虚拟机网络桥接模式之类的玩意，该不会wsl和本机的ip不一样不能用localhost互访吧，

最他妈奇葩的事情来了，MS的官方文档告诉我WSL2是支持通过localhost直接访问的
然后我又在主机的PowerShell查了端口占用情况
```bash
netstat -ano
netstat -ano|findstr '50050
```
在Windows下能够看到gRPC服务的端口是被占用了的

but，wsl和主机的ip还真不就一样，而且问题还真就出在这里，就是我想都没想过的``1``

在wsl中安装```net-tools```然后通过ifconfig查看wsl的ip
```bash
sudo apt install net-tools

ifconfig
```

输出如下：
```log
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.20.55.78  netmask 255.255.240.0  broadcast 172.20.63.255
        inet6 fe80::215:5dff:fef2:530  prefixlen 64  scopeid 0x20<link>
        ether 00:15:5d:f2:05:30  txqueuelen 1000  (Ethernet)
        RX packets 1281  bytes 380780 (380.7 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 249  bytes 20825 (20.8 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

使用``172.20.55.78``对gRPC服务进行配置，测试通过

日内瓦，退钱！
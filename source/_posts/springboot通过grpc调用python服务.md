---
title: springboot通过grpc调用python服务
date: 2023-03-09 21:44:35
tags: 
- 微服务
- python
- springboot
- grpc
categories: 教程
---

# springboot与python微服务间互调

## gRPC实现步骤：
- 定义一个服务，指定其能够被远程调用的方法（包含参数、返回类型）
- 在服务端实现这个接口，并运行一个 gRPC 服务器来处理客户端请求
- 在客户端实现一个存根 Stub ，用于发起远程方法调用

定义proto接口规范```xx.proto```
```java
// proto协议版本
syntax = "proto3";

// 参数结构
message ClientInput {
   string basecolor = 1;
   string eyecolor = 2;
   string environment = 3;
   string highlightcolor = 4;
   string accentcolor = 5;
   string pattern = 6;
   string eyeshape = 7;
   string mouth = 8;
   string purrstige = 9;
   string secret = 10;
}

// 返回值结构
message ServerOutput {
   float price_evaluated = 1;
}

//rpc方法定义
//由于根据服务名称直接生成java类，首字母建议大写
//否则在IDE中导入类时会出现奇怪的问题，但是没有发现编译问题
service Price_evaluate {
//service price_evaluate {
   rpc greet (ClientInput) returns (ServerOutput) {}
}
```
## Python服务端
python pip依赖
```bash
grpcio==1.51.3
grpcio-tools==1.51.3
protobuf==4.22.0
```

编译```.proto```生成py文件
```bash
python -m grpc_tools.protoc --python_out=.  --grpc_python_out=.  -I. test.proto
```
- ```python -m grpc_tools.protoc```: python 下的 protoc 编译器通过 python 模块(module) 实现, 所以说这一步非常省心
- ```--python_out=.``` : 编译生成处理 protobuf 相关的代码的路径, 这里生成到当前目录
- ``--grpc_python_out=.`` : 编译生成处理 grpc 相关的代码的路径, 这里生成到当前目录
- ``-I. xx.proto`` : proto 文件的路径, 这里的 proto 文件在当前目录
## Java客户端

Maven依赖
```xml
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.26.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.26.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.26.0</version>
</dependency>
```

或者更直接的
```xml
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-all</artifactId>
    <version>1.26.0</version>
</dependency>
```
Maven编译插件
```xml
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.5.0.Final</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.5.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.5.1-1:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.15.0:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

创建```/src/main/proto```目录，与```/java```同级

使用```maven protobuf:compile```、``maven protobuf:compile-custom``编译成java文件，生成的文件在```/target/generated-sources/protobuf```中

将生成的java类copy到项目中，使用``xxxBlockingStub``类调用grpc方法，通过``@GrpcService()``指定grpc服务配置

```java
@GrpcClient("price_evaluate")
price_evaluateGrpc.price_evaluateBlockingStub priceEvaluateBlockingStub;

public Double evaluate(Product product){
    PriceEvaluate.Properties properties = PriceEvaluate.Properties.newBuilder()
            .setAccentcolor(product.getAccentcolor())
            .setBasecolor(product.getBasecolor())
            .setEnvironment(product.getEnvironment())
            .setEyecolor(product.getEyecolor())
            .setEyeshape(product.getEyeshape())
            .setHighlightcolor(product.getHighlightcolor())
            .setMouth(product.getMouth())
            .setPattern(product.getPattern())
            .setPurrstige(product.getPurrstige())
            .setSecret(product.getSecret()).build();
    return (double) priceEvaluateBlockingStub.greet(properties).getPriceEvaluated();
}
```

yml中作如下配置

```yaml
grpc:
  client:
    price_evaluate: #服务名称
      address: static://172.20.55.78:50051 # static用于指定静态的服务地址 也可以通过Spring Cloud等服务注册机制获取服务地址
      negotiation-type: plaintext # 传输层加密 默认为TLS 开发环境中设置为明文避免ssl报错
```
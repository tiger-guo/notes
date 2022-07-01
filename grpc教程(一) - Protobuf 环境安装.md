---
title: grpc教程(一) - Protobuf 环境安装
date: 2022-04-18
tags:
    - Grpc
categories:
    - Grpc
---


## Mac 下载 protobuf
去 [protobuf 官网](https://github.com/protocolbuffers/protobuf/releases) 找到你要下载的版本，选择 osx 的安装包。osx安装包有两种（aarch_64是M1芯片的），根据自己的mac选择。

### 解压后，目录：
```shell
├── bin
│   └── protoc # protoc二进制放到环境变量目录下，我是放到了 GOPATH/bin 目录下
├── include
│   └── google # 存放了protobuf的一些扩展基础变量和结构体，proto文件如果用到的话，可以放到项目中，用相对路径引用，也可以和 protoc 二进制放到一起，运行 protoc 时，会自动去找的。
│       └── protobuf
├── protoc-3.20.0-osx-x86_64.zip
└── readme.txt
```

### 安装
```
cp bin/protoc $GOPATH/bin
```

## GO依赖
```
go get google.golang.org/protobuf/cmd/protoc-gen-go@v1.28.0
go get google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2.0
go get github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway@v1.16.0
```


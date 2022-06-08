---
title: grpc教程(二) - grpcurl 使用教程
date: 2022-04-18
tags:
    - grpc
categories:
    - grpc
---

# grpcurl 使用教程

首先需要确认你是否需要通过 grpcurl 获取 grpc 服务的请求方法信息，以及请求参数信息。如果需要，服务端需要使用 reflection 包的 Register 函数，将 grpc.Server 注册到反射服务中。如果不进行注册，grpcurl 仅可发起请求，无法获取到 grpc 服务的请求方法等信息。

```
import (
    "google.golang.org/grpc/reflection"
)

func main() {
    s := grpc.NewServer()
    pb.RegisterYourOwnServer(s, &server{})

    // Register reflection service on grpc server.
    reflection.Register(s)

    s.Serve(lis)
}
```
如果启动了 gprc 反射服务，那么就可以通过 reflection 包提供的反射服务查询 grpc 服务或调用 grpc 方法。

## 查询 grpc 服务
### 前置说明（重要）
如果请求服务的 proto 文件比较复杂，存在相互导入的话，需要通过 -import-path 和 -proto 指定请求服务的 proto 文件。grpcurl 会以 import-path 为根目录，去查找请求服务的 proto 文件中 import 的其他文件。下面所有示例均以带 import 导入的 proto 举例。如果 proto 比较简单可以去调这两个参数。

#### 举例说明：
**目录结构（文件内容看附件）:**
```
.
├── go.mod
├── go.sum
├── protocol
│   ├── base
│   │   ├── Makefile
│   │   ├── base.pb.go
│   │   └── base.proto
│   └── hello
│       ├── Makefile
│       ├── hello.pb.go
│       ├── hello.proto   # hello.proto import base.proto 文件
│       └── hello_grpc.pb.go
└── server
    └── main.go
```


查询 grpc 服务信息：
```
> grpcurl -plaintext -import-path ./  -proto ./grpc-demo/protocol/hello/hello.proto 127.0.0.1:50001 describe hello.HelloService
hello.HelloService is a service:
service HelloService {
  rpc SayHello ( .hello.HelloRequest ) returns ( .base.BaseResponse );
}
```


如果不指定 -import-path 和 -proto 参数的话，获取 hello 的服务信息，会出现下面的错误。
```
> grpcurl -plaintext 127.0.0.1:50001 describe hello.HelloService
Failed to resolve symbol "hello.HelloService": file "hello.proto" included an unresolvable reference to ".base.BaseResponse"
```

### 1. 查询提供的 grpc 服务列表
```
> grpcurl -plaintext -import-path ./  -proto ./grpc-demo/protocol/hello/hello.proto 127.0.0.1:50001 list
hello.HelloService
```

### 2. 获取 grpc 服务描述信息
```
> grpcurl -plaintext -import-path ./  -proto ./grpc-demo/protocol/hello/hello.proto 127.0.0.1:50001 describe hello.HelloService
hello.HelloService is a service:
service HelloService {
  rpc SayHello ( .hello.HelloRequest ) returns ( .base.BaseResponse );
  rpc SayNice ( .hello.NiceRequest ) returns ( .base.BaseResponse );
}
```

### 3. 查询 grpc 服务的方法列表
```
> grpcurl -plaintext -import-path ./  -proto ./grpc-demo/protocol/hello/hello.proto 127.0.0.1:50001 list hello.HelloService
hello.HelloService.SayHello
hello.HelloService.SayNice
```

### 4. 获取 grpc 服务方法的描述信息
```
> grpcurl -plaintext -import-path ./  -proto ./grpc-demo/protocol/hello/hello.proto 127.0.0.1:50001 describe hello.HelloService.SayHello
hello.HelloService.SayHello is a method:
rpc SayHello ( .hello.HelloRequest ) returns ( .base.BaseResponse );
```

### 5. 获取 grpc 服务类型描述信息
```
> grpcurl -plaintext -import-path ./  -proto ./grpc-demo/protocol/hello/hello.proto 127.0.0.1:50001 describe .hello.HelloRequest
hello.HelloRequest is a message:
message HelloRequest {
  string name = 1;
}
```

## 调用 grpc 服务
### 传值[-d]
```
> grpcurl -plaintext -d '{"name": "guohu"}' -import-path ./  -proto ./grpc-demo/protocol/hello/hello.proto 127.0.0.1:50001 hello.HelloService.SayHello
{
  "message": "Hello guohu"
}
```

### 传 metadata，相当于 rpc-header[-rpc-header]
```
> grpcurl -plaintext -d '{"name": "guohu"}' -rpc-header 'x-api-name: guohu' -import-path ./  -proto ./grpc-demo/protocol/hello/hello.proto 127.0.0.1:50001 hello.HelloService.SayHello
{
  "message": "Hello guohu"
}
```

## 附件
示例代码目录：
```
grpc-demo
├── go.mod
├── go.sum
├── protocol
│   ├── base
│   │   ├── Makefile
│   │   ├── base.pb.go
│   │   └── base.proto
│   └── hello
│       ├── Makefile
│       ├── hello.pb.go
│       ├── hello.proto
│       └── hello_grpc.pb.go
└── server
    └── main.go
```

### protocol/hello/hello.proto
```
syntax = "proto3";

package hello;

option go_package = "grpc-demo/protocol/hello;hello";

import "grpc-demo/protocol/base/base.proto";

service HelloService {
  rpc SayHello(HelloRequest) returns (base.BaseResponse);
  rpc SayNice(NiceRequest) returns (base.BaseResponse);
}

message NiceRequest {
  string msg = 1;
}

message HelloRequest {
  string name = 1;
}
```

### protocol/base/base.proto
```
syntax = "proto3";

package base;

option go_package = "grpc-demo/protocol/base;base";

message BaseResponse {
  int32 code = 1;
  string message = 2;
}
```

### server.main
```
package main

import (
	"flag"
	"fmt"
	"log"
	"net"
	"os"
	"os/signal"
	"syscall"
	"time"

	"golang.org/x/net/context"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"

	"grpc-demo/protocol/base"
	"grpc-demo/protocol/hello"
)

var (
	port = flag.Int("port", 50001, "listening port")
)

func main() {
	flag.Parse()
	lis, err := net.Listen("tcp", fmt.Sprintf("127.0.0.1:%d", *port))
	if err != nil {
		panic(err)
	}

	ch := make(chan os.Signal, 1)
	signal.Notify(ch, syscall.SIGTERM, syscall.SIGINT, syscall.SIGKILL, syscall.SIGHUP, syscall.SIGQUIT)
	go func() {
		s := <-ch
		log.Printf("receive signal '%v'", s)
		os.Exit(1)
	}()
	log.Printf("starting hello service at %d", *port)
	s := grpc.NewServer()
	hello.RegisterHelloServiceServer(s, &server{})

	reflection.Register(s)
	s.Serve(lis)
}

// server is used to implement helloworld.GreeterServer.
type server struct{}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *hello.HelloRequest) (*base.BaseResponse, error) {
	fmt.Printf("%v: Receive is %s\n", time.Now(), in.Name)

	return &base.BaseResponse{Code: int32(0), Message: "Hello " + in.Name}, nil
}

// SayNice implements helloworld.GreeterServer
func (s *server) SayNice(ctx context.Context, in *hello.NiceRequest) (*base.BaseResponse, error) {
	fmt.Printf("%v: Receive is %s\n", time.Now(), in.Msg)

	return &base.BaseResponse{Code: int32(0), Message: "Nice " + in.Msg}, nil
}
```

# 简单的RPC服务demo


<!--more-->

# RPC learning

### 一.grpc demo

##### 1.使用proto文件生成go代码:

&编写proto文件

~~~protobuf
//约定语法
syntax = "proto3";
//这部分内容是关于最后生成的go文件存在哪个文件目录，. 代表当前项目目录下，server代表最后生成的文件的名字
option go_package = ".;service";

//service 相当于是go的函数
//service 定义一个服务，客户端可以传入参数，返回服务端的响应
//发送一个HelloRequest 返回一个HelloResponse

service SayHello {
  rpc SayHello(HelloRequest) returns (HelloResponse) {}
}

//message 相当于是go的结构体，
//后面的特殊"赋值",代表的是位置顺序,而不是真正的赋值

message HelloRequest{
  string requestName = 1;
}

message HelloResponse{
  string responseMessage = 1;
}
~~~

&生成go代码

(记得切换到proto目录下,并注意命令中的空格以及下划线等的符号)

~~~bash
protoc --go_out=. hello.proto 
protoc --go-grpc_out=. hello.proto
~~~

##### 2.proto文件介绍：

###### syntax:约定语法

###### option:这部分内容是关于最后生成的go文件存在哪个文件目录

###### message ：传输的消息格式的定义

1.字段规则：

required:必填字段，protobuf2中使用，protobuf3中删除

optional:可选字段，protobuf3中删去了required和optional，默认optional

repeate：可重复字段，重复的值的顺序会被保留到go中重复的会被定义为切片

2.消息号：
也就是看起来像赋值的"操作" 每个字段都必须要有的消息号[1,2^29-1]的整数

3.嵌套消息：支持嵌套消息

###### service:定义一个rpc服务接口

~~~go
service 服务接口名{
    rpc 服务函数名(参数) returns(返回参数){}
}
~~~

##### 3.服务端编写

~~~go
package main

import (
	pb "RPClearning/hello_server/proto"
	"context"
	"errors"
	"fmt"
	"google.golang.org/grpc"
	"google.golang.org/grpc/metadata"
	"net"
)

type server struct {
	pb.UnimplementedSayHelloServer
}

// 业务
func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloResponse, error) {
	//进行token校验
	//获取元数据 注：返回的元数据切片中的所有key都是小写！小写！小写！
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, errors.New("未传输token")
	}
	var appID string
	var appKey string
	if v, ok := md["appid"]; ok {
		appID = v[0]
	}
	if v, ok := md["appkey"]; ok {
		appKey = v[0]
	}
	if appID != "lance" || appKey != "123" {
		return &pb.HelloResponse{ResponseMessage: "token错误 " + req.RequestName}, nil

	}

	fmt.Println("hello " + req.RequestName)
	return &pb.HelloResponse{ResponseMessage: "hello " + req.RequestName}, nil
}

func main() {
	//监听端口
	listener, _ := net.Listen("tcp", ":9090")
	//创建一个grpc服务
	grpcServer := grpc.NewServer()
	//在grpc服务端中注册编写的服务
	pb.RegisterSayHelloServer(grpcServer, &server{}) //一定要是引用传递对象
	//启动服务
	err := grpcServer.Serve(listener)
	if err != nil {
		fmt.Printf("failed to serve :%v", &err)
		return
	}

}

~~~

##### 4.客户端编写

~~~go
package main

import (
	pb "RPClearning/hello_server/proto"
	"context"
	"fmt"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"log"
)

type ClientToken struct {
}

func (c ClientToken) GetRequestMetadata(ctx context.Context, uri ...string) (map[string]string, error) {
	return map[string]string{
		"appID" :  "lance",
		"appKey": "123",
	}, nil
}
func (c ClientToken) RequireTransportSecurity() bool {
	return false //是否带安全验证
}

func main() {
	//连接到server端,此处禁止安全传输
	var opts []grpc.DialOption
	opts = append(opts, grpc.WithTransportCredentials(insecure.NewCredentials()))
	opts = append(opts, grpc.WithPerRPCCredentials(new(ClientToken)))

	conn, err := grpc.Dial("127.0.0.1:9090", opts...)
	if err != nil {
		log.Fatalf("didn't connect :%v", err)
	}
	defer conn.Close()

	//建立连接
	client := pb.NewSayHelloClient(conn)
	//执行rpc调用（这个方法在服务端来实现，并返回结果）
	response, _ := client.SayHello(context.Background(), &pb.HelloRequest{RequestName: "lance"})
	res := response.GetResponseMessage()
	fmt.Println("response:", res)
}

~~~

### 二.net/rpc demo

##### 1.服务端代码：

~~~go
package main

import (
	"net"
	"net/rpc"
)

type Server struct {
}

// 通过阅读源码，发现method是有要求的 ：
// Method needs three ins: receiver, *args, *reply.
// First arg need not be a pointer.
// Second arg must be a pointer.
// Reply type must be exported.
// Method needs one out.
// The return type of the method must be error

func (s Server) SayHello(args []string, what *string) error {
	*what = args[0] + " say hello to " + args[1]
	return nil
}
func main() {
	//新建一个对象实例
	s := new(Server)

	//注册服务
	err := rpc.Register(s)
	if err != nil {
		panic(err)
	}
	//监听
	listener, errs := net.Listen("tcp", ":9090")
	if errs != nil {
		panic(err)
	}
	//启动服务
	for {
		conn, err := listener.Accept()
		if err != nil {
			continue
		}
		go rpc.ServeConn(conn)
	}
}

~~~

##### 2.客户端代码

~~~go
package main

import (
	"fmt"
	"net/rpc"
)

var Names = []string{"lance", "longxu"}
var What string

func main() {
	client, err := rpc.Dial("tcp", "localhost:9090")
	must(err)
	defer client.Close()

	//err = client.Call("Server.SayHello", Names, &What)
	//if err != nil {
	//	panic(err)
	//}

	// ！！！！注意这里serviceMethod不应该只传入method 而是应该传入object.Method这样的格式,否则会报 service/method request ill-formed的错误
	call := <-client.Go("Server.SayHello", Names, &What, make(chan *rpc.Call, 1)).Done
	fmt.Println(call.ServiceMethod)
	fmt.Println(call.Reply)
	fmt.Println(call.Args)
	fmt.Println(call.Error)
	if call.Error != nil {
		panic(err)
	}
	fmt.Println("rpc调用成功，取得值：", What)
}

func must(err error) {
	if err != nil {
		panic(err)
	}
}
~~~


# README

## Go语言实现简单RPC框架

### 0. 参考

从零开始实现一个RPC框架（零）：https://juejin.cn/post/6844903788948488200

从零开始实现一个RPC框架（一）：https://juejin.cn/post/6844903793755160583

### 1. 需求分析

* 支持RPC调用，包括同步调用和异步调用

* 支持服务治理的相关功能，包括：
  * 服务注册与发现
  * 服务负载均衡
  * 限流和熔断
  * 身份认证
  * 监控和链路追踪
  * 健康检查，包括端到端的心跳以及注册中心对服务实例的检查

* 支持插件，对于有多种实现的功能（比如负载均衡），需要以插件的形式提供实现，同时需要支持自定义插件 

### 2. 系统设计

#### 2.1 分层

<img src="/Users/huangsiyi/Desktop/gorpc/img/层次图.jpg" style="zoom:50%;" />

- service 是面向用户的接口，比如客户端和服务端实例的初始化和运行等等
- client和server表示客户端和服务端的实例，它们负责发出请求和返回响应
- selector 表示负载均衡，或者叫做loadbanlancer，它负责决定具体要向哪个server发出请求
- registery 表示注册中心，server在初始化完毕甚至是运行时都要向注册中心注册自身的相关信息，这样client才能从注册中心查找到需要的server
- codec 表示编解码，也就是将对象和二进制数据互相转换
- protocol 表示通信协议，也就是二进制数据是如何组成的，RPC框架中很多功能都需要协议层的支持
- transport 表示通讯，它负责具体的网络通讯，将按照protocol组装好的二进制数据通过网络发送出去，并根据protocol指定的方式从网络读取数据

客户端从发出请求到收到响应的大致流程：

1. 发送请求
2. 确定目标服务器
3. 请求参数编码，产生二进制数据
4. 根据数据组装最终的二进制数据
5. 将数据通过网络发送到服务端
6. 网络传输给服务端，服务端返回数据
7. 从网络读取数据
8. 根据协议解析收到的二进制数据
9. 根据二进制数据进一步解析成响应
10. 收到响应

#### 2.2 过滤器链

通过上面的层次划分可以看到，一个请求或者响应实际上会依次穿过各个层然后通过网络发送或者到达用户逻辑，所以我们采用类似过滤器链一样的方式处理请求和响应，以此来达到对扩展开放，对修改关闭的效果。这样一来对于一些附加功能比如熔断降级和限流、身份认证等功能都可以在过滤器中实现。

#### 2.3 消息协议

在我们的RPC框架中，需要定义自己的消息协议。一般来说，网络协议都分为head和body部分，head是一些元数据，是协议自身需要的数据，body则是上一层传递来的数据，只需要原封不动的接着传递下去。

```
--------------------------------------------------------------------------------------------
|2byte|1byte  |4byte       |4byte        | header length |total length-header length-4byte)|
--------------------------------------------------------------------------------------------
|magic|version|total length|header length|     header    |            body                 |
--------------------------------------------------------------------------------------------

```

根据上面的协议，一个消息体由以下几个部分严格按照顺序组成：

- 两个byte的magic number开头，这样一来我们就可以快速的识别出非法的请求
- 一个byte表示协议的版本，目前可以一律设置为0
- 4个byte表示消息体剩余部分的总长度（total length）
- 4个byte表示消息头的长度（header length）
- 消息头（header），其长度根据前面解析出的长度（header length）决定
- 消息体（body），其长度为前面解析出的总长度减去消息头所占的长度（total length - 4 - header length)

协议中消息头的数据主要是RPC调用过程中的元数据，元数据跟方法参数和响应无关，主要记录额外的信息以及实现附属功能比如链路追踪、身份认证等等；消息体的数据则是由实际的请求参数或者响应编码而来。 在实际的处理中，消息头在发送端通常是一个结构体，在发送时会被编码成二进制添加在消息头的前面，在接收端接收时又解码成一个结构体，交给程序进行处理。这里试着列举消息头包含的各个信息：

```go
type Header struct {
        Seq uint64 //序号, 用来唯一标识请求或响应
        MessageType byte //消息类型，用来标识一个消息是请求还是响应
        CompressType byte //压缩类型，用来标识一个消息的压缩方式
        SerializeType byte //序列化类型，用来标识消息体采用的编码方式
        StatusCode byte //状态类型，用来标识一个请求是正常还是异常
        ServiceName string //服务名
        MethodName string  //方法名
        Error string //方法调用发生的异常
        MetaData map[string]string //其他元数据
}
```

### 3. 功能设计

RPC调用的第一步，就是在服务端定义要对外暴露的方法，在grpc或者是thrift中，这一步我们需要编写语言无关的idl文件，然后通过idl文件生成对应语言的代码。而在我们的框架里，出于简单起见，我们不采用idl的方式，直接在代码里定义接口和方法。这里先规定对外的方法必须遵守以下几个条件：

1. 对外暴露的方法，其所属的类型和其自身必须是对外可见（Exported）的，也就是首字母必须是大写的
2. 方法的参数必须为三个，而且第一个必须是context.Context类型
3. 第三个方法参数必须是指针类型
4. 方法返回值必须是error类型
5. 客户端通过"Type.Method"的形式来引用服务方法，其中Type是方法**实现类**的全类名，Method就是方法名称

为什么要有这几个规定呢，具体的原因是这样的：因为java中的RPC框架场用到的动态代理在go语言中并不支持，所以我们需要显式地定义方法的统一格式，这样在RPC框架中才能统一地处理不同的方法。所以我们规定了方法的格式：

- 方法的第一个参数固定为Context，用于传递上下文信息
- 第二个参数是真正的方法参数
- 第三个参数表示方法的返回值，调用完成后它的值就会被改变为服务端执行的结果
- 方法的返回值固定为error类型，表示方法调用过程中发生的错。

这里我们需要注意的是，服务提供者在对外暴露时并不需要以接口的形式暴露，只要服务提供者有符合规则的方法即可；而客户端在调用方法时指定的是服务提供者的具体类型，不能指定接口的名称，即使服务提供者实现了这个接口。

**contet.Context**

[context](https://golang.org/pkg/context/)是go语言提供的关于请求上下文的抽象，它携带了请求deadline、cancel信号的信息，还可以传递一些上下文信息，非常适合作为RPC请求的上下文，我们可以在context中设置超时时间，还可以将一些参数无关的元数据通过context传递到服务端。

实际上，方法的固定格式以及用Call和Go来表示同步和异步调用都是go自带的rpc里的规则，只是在参数里增加了context.Context。不得不说go自带的rpc设计确实十分优秀，值得好好学习理解。

### 4. 接口定义

#### 4.1 Client和Server

首先是面向使用者的RPC框架中的客户端和服务端接口：

```go
type RPCServer interface {
        //注册服务实例，rcvr是receiver的意思，它是我们对外暴露的方法的实现者，metaData是注册服务时携带的额外的元数据，它描述了rcvr的其他信息
        Register(rcvr interface{}, metaData map[string]string) error
        //开始对外提供服务
        Serve(network string, addr string) error
}
type RPCClient interface {      
        //Go表示异步调用
        Go(ctx context.Context, serviceMethod string, arg interface{}, reply interface{}, done chan *Call) *Call
        //Call表示异步调用
        Call(ctx context.Context, serviceMethod string, arg interface{}, reply interface{}) error
        Close() error
}
type Call struct {
	ServiceMethod string      // 服务名.方法名
	Args          interface{} // 参数
	Reply         interface{} // 返回值（指针类型）
	Error         error       // 错误信息
	Done          chan *Call  // 在调用结束时激活
}
```

#### 4.2 Selector和Registery

#### 4.3 Codec

接下来我们需要选择一个序列化协议，这里就选之前使用过的[messagepack](https://msgpack.org/)。之前设计的通信协议分为两个部分：head和body，这两个部分都需要进行序列化和反序列化。head部分是元数据，可以直接采用messagepack序列化，而body部分是方法的参数或者响应，其序列化由head中的SerializeType决定，这样的好处就是为了后续扩展方便，目前也使用messagepack序列化，后续也可以采用其他的方式序列化。

序列化的逻辑也定义为接口：

```go
type Codec interface {
   Encode(value interface{}) ([]byte, error)
   Decode(data []byte, value interface{}) error
}
```

#### 4.4 Protocol

确定好了序列化协议之后，我们就可以定义消息协议相关的接口了。

```go
//Messagge表示一个消息体
type Message struct {
	*Header //head部分, Header的定义参考上一篇文章
	Data []byte //body部分
}

//Protocol定义了如何构造和序列化一个完整的消息体
type Protocol interface {
	NewMessage() *Message
	DecodeMessage(r io.Reader) (*Message, error)
	EncodeMessage(message *Message) []byte
}
```

#### 4.5 Transport

网络传输层的定义：

```go
//传输层的定义，用于读取数据
type Transport interface {
	Dial(network, addr string) error
	//这里直接内嵌了ReadWriteCloser接口，包含Read、Write和Close方法
	io.ReadWriteCloser 
	RemoteAddr() net.Addr
	LocalAddr() net.Addr
}
//服务端监听器定义，用于监听端口和建立连接
type Listener interface {
	Listen(network, addr string) error
	Accept() (Transport, error)
	//这里直接内嵌了Closer接口，包含Close方法
	io.Closer
}
```

### 5. 具体实现

#### 5.1 Client

客户端的功能比较简单，就是将参数序列化之后，组装成一个完整的消息体发送出去。请求发送出去的同时，将未完成的请求都缓存起来，每收到一个响应就和未完成的请求进行匹配。

发送请求的核心在`Go`和`send`方法，`Go`的功能是组装参数，`send`方法是将参数序列化并通过传输层的接口发送出去，同时将请求缓存到`pendingCalls`中。而`Call`方法则是直接调用`Go`方法并阻塞等待知道返回或者超时。 接收响应的核心在`input`方法，`input`方法在client初始化完成时通过`go input()` 执行。`input`方法包含一个无限循环，在无限循环中读取传输层的数据并将其反序列化，并将反序列化得到的响应与缓存的请求进行匹配。

注：`send`和`input`方法的命名也是从go自带的rpc里学来的。

#### 5.2 Server

服务端在接受注册时，会过滤服务提供者的各个方法，将合法的方法缓存起来。

服务端的核心逻辑是`serveTransport`方法，它接收一个`Transport`对象，然后在一个无限循环中从`Transport`读取数据并反序列化成请求，根据请求指定的方法查找自身缓存的方法，找到对应 的方法后通过反射执行对应的实现并返。执行完成后再根据返回结果或者是执行发生的异常组装成一个完整的消息，通过`Transport`发送出去。

服务端在反射执行方法时，需要将实现者作为执行的第一个参数，所以参数比方法定义中的参数多一个。

#### 5.3 Codec和Protocol

Codec基本上就是使用messagepack实现了对应的接口；Protocol的实现就是根据我们定义的协议进行解析。

#### 5.4 线程模型

在执行过程中，除了客户端的用户线程和服务端用来执行方法的服务线程，还分别增加了客户端轮询线程和服务端监听线程，大致的示意图如下：

![](/Users/huangsiyi/Desktop/gorpc/img/线程模型示意图.jpg)

这里的线程模型比较简单，服务端针对每个建立的连接都会创建一个线程（goroutine），虽说goroutine很轻量，但是也不是完全没有消耗的，后续可以再进一步进行优化，比如把读取数据反序列化和执行方法拆分到不同的线程执行，或者把goroutine池化等等。


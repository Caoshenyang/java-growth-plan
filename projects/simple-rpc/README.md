# 简易 RPC 框架

> 实战项目：理解 RPC 核心原理

---

## 🎯 项目目标

通过实现一个简易的 RPC 框架，深入理解：

- [ ] RPC 调用原理
- [ ] 网络通信 (Netty)
- [ ] 序列化/反序列化
- [ ] 服务注册与发现
- [ ] 负载均衡
- [ ] 动态代理

---

## 📋 功能清单

### Phase 1: 基础框架 ✅

- [x] 自定义协议
- [x] Netty 服务器
- [x] Netty 客户端
- [x] JSON 序列化

### Phase 2: 核心功能 🔄

- [ ] 动态代理
- [ ] 服务注册
- [ ] 服务发现
- [ ] 负载均衡

### Phase 3: 高级特性 📋

- [ ] 超时重试
- [ ] 熔断降级
- [ ] 链路追踪
- [ ] 限流控制

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                        RPC 框架架构                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [Consumer]                                                 │
│      │                                                      │
│      ├──> [RpcProxy] (动态代理)                             │
│      │                                                      │
│      ├──> [RpcClient]                                       │
│      │     ├── Netty Client                                 │
│      │     ├── Encoder/Decoder                              │
│      │     └── Serializer                                   │
│      │                                                      │
│      └─────────────────────────────────────────────┐        │
│                                                   │        │
│  [Registry] ───────────────────────────────────────┘        │
│  │                                                       │  │
│  │  服务注册/发现                                         │  │
│  │                                                       │  │
│  └───────────────┐                                       │  │
│                  │                                       │  │
│                  v                                       │  │
│  [Provider] ─────┘                                       │  │
│      │                                                     │
│      ├──> [RpcServer]                                     │
│      │     ├── Netty Server                               │
│      │     ├── Encoder/Decoder                            │
│      │     └── Serializer                                 │
│      │                                                     │
│      └──> [ServiceExporter]                               │
│            └── 服务实现                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📁 项目结构

```
simple-rpc/
├── rpc-api/              # API 模块
│   ├── RpcRequest.java
│   ├── RpcResponse.java
│   └── RpcProtocol.java
├── rpc-server/           # 服务端模块
│   ├── RpcServer.java
│   ├── ServerHandler.java
│   └── ServiceRegistry.java
├── rpc-client/           # 客户端模块
│   ├── RpcClient.java
│   ├── ClientHandler.java
│   └── ServiceDiscovery.java
├── rpc-common/           # 公共模块
│   ├── Serializer/
│   │   ├── JsonSerializer.java
│   │   └── HessianSerializer.java
│   ├── Codec/
│   │   ├── RpcDecoder.java
│   │   └── RpcEncoder.java
│   └── Proxy/
│       └── RpcProxyFactory.java
└── example/              # 示例模块
    ├── HelloService.java
    └── HelloServiceImpl.java
```

---

## 💻 代码实现

### 协议定义

```java
public class RpcProtocol {
    private short magic;        // 魔数 0xDABB
    private byte version;       // 版本号
    private byte serializeType; // 序列化类型
    private int messageId;      // 消息ID
    private int bodyLength;     // 消息体长度
    private byte[] body;        // 消息体
}
```

### 请求定义

```java
public class RpcRequest {
    private String interfaceName;  // 接口名
    private String methodName;     // 方法名
    private Class<?>[] paramTypes; // 参数类型
    private Object[] params;       // 参数值
}
```

### 响应定义

```java
public class RpcResponse {
    private int messageId;  // 消息ID
    private int code;       // 响应码
    private String message; // 响应消息
    private Object data;    // 返回数据
}
```

---

## 🔧 使用示例

### 服务端

```java
public class Server {
    public static void main(String[] args) {
        // 创建服务
        HelloService helloService = new HelloServiceImpl();

        // 导出服务
        RpcServer rpcServer = new RpcServer(8080);
        rpcServer.export(HelloService.class, helloService);

        // 启动服务器
        rpcServer.start();
    }
}
```

### 客户端

```java
public class Client {
    public static void main(String[] args) {
        // 创建代理
        HelloService helloService = RpcProxyFactory
            .create(HelloService.class, "localhost", 8080);

        // 调用服务
        String result = helloService.sayHello("World");
        System.out.println(result);
    }
}
```

---

## 📊 开发进度

| 模块 | 进度 | 状态 |
|------|------|------|
| 协议定义 | 100% | ✅ |
| 序列化 | 100% | ✅ |
| 编解码 | 100% | ✅ |
| 服务端 | 80% | 🔄 |
| 客户端 | 60% | 🔄 |
| 动态代理 | 0% | 📋 |
| 注册中心 | 0% | 📋 |
| 负载均衡 | 0% | 📋 |

---

## 🚀 待办事项

- [ ] 完成客户端基础功能
- [ ] 实现动态代理
- [ ] 添加服务注册中心
- [ ] 实现负载均衡策略
- [ ] 添加超时重试
- [ ] 添加性能测试

---

## 📝 学习笔记

相关笔记：[Netty 学习笔记](../../docs/phase1-jvm/netty.md)

---

## 🔗 参考资料

- [Netty 官方文档](https://netty.io/)
- [Dubbo 文档](https://dubbo.apache.org/zh/)
- [gRPC 文档](https://grpc.io/docs/languages/java/)

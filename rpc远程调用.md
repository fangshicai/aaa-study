# 1 protobuf序列化库

protobuf是什么：

协议缓冲区是 Google 为序列化结构化数据（类似于 XML，但更小、更快、更简单）而提供的语言中立、平台无关且可扩展的机制。您只需定义一次数据的结构化方式，即可使用专门生成的源代码，轻松地使用各种语言在各种数据流中写入和读取结构化数据。

协议缓冲区是定义语言（在 `.proto`文件中创建）、proto 编译器生成的与数据接口的代码、特定于语言的运行时库、写入文件（或通过网络连接发送）的数据的序列化格式以及序列化数据的组合。

官方主页：https://protobuf.dev/



protoc编译器：jdk8配protoc-4xx以下（protoc-3.20.3-win64）

Github地址：https://github.com/protocolbuffers/protobuf



java代码整合引入pom

```pom
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.25.3</version>
</dependency>
```



使用

1. 编写my_protocol.proto文件

```java
syntax = "proto3";

option java_outer_classname = "MyProtocol";
option java_package = "com.syd.storagedatabe";
option java_multiple_files = false;

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}

```

2. 生成java代码

在protoc的`bin`目录下执行

`protoc.exe -I=D:\poject\svn\trunk\04\storage-data-be\src\main\proto --java_out=D:\poject\svn\trunk\04\storage-data-be\src\main\java my_protocol.proto`根据路径修改

3. 使用

```java
package com.syd.storagedatabe;

import com.google.protobuf.InvalidProtocolBufferException;

import java.util.Arrays;

public class Main {
    public static void main(String[] args) {
        // 使用builder方法生成user对象
        MyProtocol.User.Builder userBuilder = MyProtocol.User.newBuilder();
        userBuilder.setId(1L);
        userBuilder.setName("Tom");
        userBuilder.setEmail("tom@gacfox.com");
        MyProtocol.User user = userBuilder.build();
        // 序列化为二进制
        byte[] data = user.toByteArray();
        System.out.println("Data serialized in " + data.length + " Bytes");
        System.out.println("Data serialized in " + Arrays.toString(data) + " Bytes");

        try {
            // 反序列化并输出
            MyProtocol.User srcUser = MyProtocol.User.parseFrom(data);
            System.out.println(srcUser.getId());
            System.out.println(srcUser.getName());
            System.out.println(srcUser.getEmail());
        } catch (InvalidProtocolBufferException e) {
            e.printStackTrace();
        }
    }
}

```



# 2 grpc

上一节只是有关protobuf的序列化和反序列，在这一节使用grpc实现通信。



# 3 HttpInvoker

![image-20250818153240601](.\images\image-20250818153240601.png)

- 只能用于java-to-java的场景
- 要求客户端和服务端都使用SPRING框架。

- 服务端与客户端接口名一致、方法参数一致

- 如果接口参数是对象的话，参数对象须实现序列化

- 接口参数是对象的话，服务端与客户端对象名要一致、包路径也得一致。 不然会报找不到类

- 只能客户端->服务端

公共模块

服务端和客户端接口:

```java
/**
 * rpc 客户端和服务器的方法名必须一样
 */
public interface HttpInvokerRpcMsgService extends Serializable {
    String sayHello(String name);
    ResultData<List<DeviceData>> msgData(List<DeviceData> list);
}
```

共同类

```java
/**
 * 数据项封装
 * rpc调用必须要序列化
 */
@Document("device_data")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class DeviceData implements Serializable {
    private static final long serialVersionUID = 645375587595711011L;
    /**
     * 必须是id，Id将会报错
     */
    @Id
    private String id;

    @Field("create_timestamp")
    private long createTimestamp;

    @Field("device_id")
    private String deviceId;

    @Field("payload")
    private String payload;

    public DeviceData(String data) {
//        this.data = data;
        this.createTimestamp = System.currentTimeMillis();
        // 前 12 字符为 DEVICE_ID
        this.deviceId = data.substring(0, 12);
        // 13字符往后 为数据
        this.payload = data.substring(12, data.length());
    }

}
```



服务端：

配置

```java
/**
 * HttpInvoker rpc服务器配置
 */
@Configuration
public class RpcConfig {

    @Bean(name = "/httpInvokerRpcMsgService")
    public HttpInvokerServiceExporter greetingService(HttpInvokerRpcMsgService greetingService) {
        HttpInvokerServiceExporter exporter = new HttpInvokerServiceExporter();
        exporter.setService(greetingService);
        exporter.setServiceInterface(HttpInvokerRpcMsgService.class);
        return exporter;
    }
}
```



实现类

```java
@Service
@Slf4j
public class ReceiveMsgFromClientServiceImpl implements HttpInvokerRpcMsgService {

    private final DeviceDataService storageToBdService;

    public ReceiveMsgFromClientServiceImpl(DeviceDataService storageToBdService) {
        this.storageToBdService = storageToBdService;
    }

    @Override
    public String sayHello(String name) {
        log.info("远程调用 sayHello：{}", name);
        return "Hello, " + name + "!";
    }

    /**
     * 远程调用接收
     * @param data
     * @return
     */
    @Override
    public RpcData<String> msgData(RpcData<List<DeviceData>> data) {
        log.info("接收到客户端的数据");
        log.info("list-size:{}", data.getData());
        boolean flag = storageToBdService.saveToDb(data.getData());
        log.info("远程调用保存接收");
        return flag ? RpcData.data("成功","1",1) : RpcData.data("失败","1",1);
    }

}
```





客户端

配置

```java
/**
 * HttpInvoker rpc客户端配置
 */
@Configuration
public class RpcClientConfig {

    @Value("${rpc-server.ip}")
    private String ip;

    @Value("${rpc-server.port}")
    private String port;

    /**
     * 代理执行
     * @return
     */
    @Bean
    public HttpInvokerProxyFactoryBean greetingService() {
        HttpInvokerProxyFactoryBean proxy = new HttpInvokerProxyFactoryBean();
        proxy.setServiceUrl("http://" + ip + ":" + port + "/httpInvokerRpcMsgService");
        proxy.setServiceInterface(HttpInvokerRpcMsgService.class);
        return proxy;
    }
}
```



客户端发送消息

```java
	// rpc远程调用 HttpInvoker 注入
    private final HttpInvokerRpcMsgService sendMsgToServerService;
    public AsyncWriter(@Qualifier("greetingService") HttpInvokerRpcMsgService sendMsgToServerService) {
        this.sendMsgToServerService = sendMsgToServerService;
    }

	// 使用
    sendMsgToServerService.sayHello("rpc ok!!!");
    ResultData<List<DeviceData>> res = sendMsgToServerService.msgData(batch);
```







todo 








# netty-spring-boot-starter
基于Netty的Spring Boot Starter工程.

## 特性
* TCP长连接消息轻松转发到Spring容器
* 在application.properties文件中轻松配置Netty参数

## 用例

使用类似于Spring MVC 中RestController的CommandController 和类似于GetMapping的CommandMapping 进行消息定义, 系统会自动将其注册进系统里

```java
@CommandController
public class SimpleCommand {

    @Autowired
    private SimpleService simpleService;

    @CommandMapping(id = 1)
    public Search.SearchResponse search(Search.SearchRequest searchRequest) {
        log.info("收到SearchRequest 1 --> {}, {}, {}", searchRequest.getQuery(), searchRequest.getPageNumber(), searchRequest.getResultPerPage());
        return Search.SearchResponse.newBuilder().setResult("查询成功").build();
    }

    @CommandMapping(id = 2)
    public Search.SearchResponse search2(Search.SearchRequest searchRequest) {
        log.info("收到SearchRequest 2 --> {}, {}, {}", searchRequest.getQuery(), searchRequest.getPageNumber(), searchRequest.getResultPerPage());
        simpleService.print();
        return Search.SearchResponse.newBuilder().setResult("查询成功").build();
    }

    @CommandMapping(id = 3)
    public void search3(Search.SearchRequest searchRequest) {
        log.info("收到SearchRequest 3 --> {}, {}, {}", searchRequest.getQuery(), searchRequest.getPageNumber(), searchRequest.getResultPerPage());
    }

    @CommandMapping(id = 4)
    public void search4(Search.SearchRequest searchRequest, String nullParam) {
        log.info("收到SearchRequest 4 --> {}, {}", searchRequest.getQuery(), nullParam);
        simpleService.print();
    }
}
``` 
在上面分别定义了四种方法
1. search() -> 系统会将接收到的byte数组解析成Protobuf SearchRequest对象作为入参, 然后Protobuf SearchResponse对象序列化城数组写回到前端
2. search3() -> 没有返回参数, 则不会进行应答
3. search4() -> 除了Protobuf SearchRequest对象, 还有一个nullParam String类型的参数, 目前还不支持自定义参数拓展, 因此这里会是空

服务端写好后, 可以使用socket client进行消息测试

```java
private static void sendMessage(byte[] message, int commandId) throws IOException {
        try (Socket socket = new Socket()) {
            socket.connect(new InetSocketAddress("localhost", 7001));

            OutputStream out = socket.getOutputStream();
            out.write(message.length);
            out.write(commandId);
            out.write(message);
            out.flush();

            log.info("commandId:{}, RemoteAddress:{}, LocalAddress:{}, write size::{}", commandId, socket.getRemoteSocketAddress(), socket.getLocalAddress(), message.length);

            if (commandId == 3 || commandId == 4) {
                return;
            }

            InputStream in = socket.getInputStream();
            int size = in.read();

            byte[] responseMessage = new byte[size];
            in.read(responseMessage);

            Search.SearchResponse searchResponse = Search.SearchResponse.parseFrom(responseMessage);

            log.info("commandId:{}, searchResponse:{}", commandId, searchResponse.getResult());

        }

}
```


## TODO
* 性能优化, 在收发消息时避免申请堆内内存
* 支持消息方法自定义参数
* 支持其他消息编码([thrift](https://thrift.apache.org/) 等)

## 其他
* 看看能否使用编译器反射工具, 美化代码

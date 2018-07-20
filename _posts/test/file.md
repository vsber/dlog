
# Specific Rpc
> * 使用avro-tools 对protocol文件编译生成java源码文件
> * 使用http 或者 netty 方式传输

协议文件为mail.avpr

```json
{"namespace": "avro.mail",
 "protocol": "Mail",

 "types": [
     {"name": "Message", "type": "record",
      "fields": [
          {"name": "to",   "type": "string"},
          {"name": "from", "type": "string"},
          {"name": "body", "type": "string"}
      ]
     },{
     "name":"RetMessage","type":"record",
        "fields":[
        {"name":"title","type": "string"},
        {"name":"content","type": "string"}
        ]
     }
 ],

 "messages": {
     "send": {
         "request": [{"name": "message", "type": "Message"}],
         "response": "string"
     },
     "send2": {
              "request": [{"name": "message", "type": "Message"}],
              "response": "RetMessage"
          }
 }
}
```

如果是协议，使用avro-tools 编译协议文件生成代码
`java -jar avro-tools-1.8.2.jar compile protocol  mail.avpr  destination-path`
生成的文件如下
* Mail.java
* Message.java
* RetMessage

Message 和 RetMessage 为定义的schema，类型为record
Mail为接口接口类的名称由`"protocol": "Mail"`决定

```java
public class Main {
    public static class MailImpl implements Mail {
        public Utf8 send(Message message) {
            return new Utf8("Sending message to " + message.getTo().toString()
                    + " from " + message.getFrom().toString()
                    + " with body " + message.getBody().toString());
        }
        public RetMessage send2(Message message) {
            return new RetMessage("hello","word");
        }
    }
    private static Server server;
    private static void startServer() throws IOException {
        //server = new NettyServer(new SpecificResponder(Mail.class, new MailImpl()), new InetSocketAddress(65111));
        server=new HttpServer(new SpecificResponder(Mail.class, new MailImpl()),65111);
        server.start();
    }

    public static void main(String[] args) throws IOException {
        startServer();
        System.out.println("Server started");

        //NettyTransceiver client = new NettyTransceiver(new InetSocketAddress(65111));
        HttpTransceiver client=new HttpTransceiver(new URL("http://127.0.0.1:65111"));
        Mail proxy = SpecificRequestor.getClient(Mail.class, client);

        Message message = new Message();
        message.setTo(new Utf8("beijing"));
        message.setFrom(new Utf8("shanghai"));
        message.setBody(new Utf8("play"));

        System.out.println("Result: " + proxy.send(message));

        RetMessage retMessage=proxy.send2(message);
        System.out.println("Result: " + retMessage);

        client.close();
        server.close();
    }
}
```
执行结果如下
```
Server started
Result: Sending message to beijing from shanghai with body play
Result: {"title": "hello", "content": "word"}
Disconnected from the target VM, address: '127.0.0.1:52314', transport: 'socket'

Process finished with exit code 0
```



---
layout: post
title: Apache Avro 小结
---

# Apache Avro 小结
>[Apache Avro™ 1.8.2 Overview](http://avro.apache.org/docs/current/index.html)
> [ Apache Avro™ 1.8.2 Getting Started(Java)](http://avro.apache.org/docs/current/gettingstartedjava.html)
> [Apache Avro™ 1.8.2 Specification](http://avro.apache.org/docs/current/spec.html)
## Apache Avro 概述
avro是一个数据序列化系统。它提供了
* 富数据结构
* 紧凑的、快捷的二进制数据格式
* 用于存储持久化数据的容器文件
* 提供了远程过程调用(Remote procedure call)
* 与动态语言简便的集成。读、写数据文件或者实现PRC协议都不需要生成代码。代码生成仅仅是一个用于优化的可选项，仅仅是为了静态类型语言而实现。

Avro提供了和Thrift, Protocol Buffers序列化系统相类似的功能，不同之处如下。
* 动态类型：Avro并不强制要求提前生成解析代码，而采取将数据和定义数据的schema同时存放的方式避免了使用提前生成的解析代码和静态类型定义等。这种便利性有利于构建通用数据处理系统和处理语言。
* 未打标签的数据：由于在读取数据时能够同时访问到该数据的schema，因此只需要将相当少的类型信息编码到数据中，结果就使得序列化之后的数据体积变得更小。
* 不存在手动分配的域ID：当schema改变之后，在处理相关数据时可以同时访问新老schema，因此新老数据的差异可以通过数据域的名字（field name ）很容易的解决（aliases）。

## Apache Avro Specification总结
### Schema Declaration

> Avro是基于schema（模式），这和protobuf、thrift没什么区别，在schema文件中（.avsc文件）中声明数据类型，那么avro在read、write时将依据schema对数据进行序列化。因为有了schema，那么Avro的读、写操作将可以使用不同的平台语言。Avro的schema是JSON格式，所以编写起来也非常简单、可读性很好。avro schema 包括 Primitive Types 、Complex Types。

#### A Schema is represented in JSON by one of:

* A JSON string, naming a defined type.
* A JSON object, of the form:
{"type": "typeName" ...attributes...}
where typeName is either a primitive or derived type name, as defined below. Attributes not defined in this document are permitted as metadata, but must not affect the format of serialized data.
*A JSON array, representing a union of embedded types.
#### Primitive Types
The set of primitive type names is:

* **null**: no value
* **boolean**: a binary value
* **int**: 32-bit signed integer
* **long**: 64-bit signed integer
* **float**: single precision (32-bit) IEEE 754 floating-point number
* **double**: double precision (64-bit) IEEE 754 floating-point number
* **bytes**: sequence of 8-bit unsigned bytes
* **string**: unicode character sequence
Primitive types have no specified attributes.

Primitive type names are also defined type names. Thus, for example, the schema "string" is equivalent to:
`{"type": "string"}`
#### Complex Types
Avro supports six kinds of complex types: **records**, **enums**, **arrays**, **maps**, **unions** and **fixed**.
### Data Serialization
数据保存为文件或者rpc传输都需要根据schema进行序列化，保存为文件时schema保存在文件中为反序列化提供支持。rpc传输数据时，通过握手过程对客户端和服务端两边的protocol进行hash一致性校验，完成匹配后对请求或响应结果根据数据对应的schema进行序列化之后传输

序列化过程中通过对数据编码，编码有**Binary Encoding**和**JSON Encoding**两种，但是avro rpc过程编码目前指看到 **Binary Encoding**方式。反序列与之对应。
### Object Container Files
Avro includes a simple object container file format. A file has a schema, and all objects stored in the file must be written according to that schema, using binary encoding. Objects are stored in blocks that may be compressed. Syncronization markers are used between blocks to permit efficient splitting of files for MapReduce processing.
avro 也引入了简单对象容器文件格式(simple object container file format)，二进制编码的形式写入。对象在文件中以块(Block)来组织，并且这些对象都是可以被压缩的。块和块之间会存在同步标记符(Synchronization Marker)，以便MapReduce方便地切割文件用于处理。下图是文件结构图：
![wdjgt](/assets/20180720-wdjgt.png)
上图已经对各块做肢解操作，但还是有必要再详细说明下。一个存储文件由两部分组成:头信息(Header)和数据块(Data Block)。而头信息又由三部分构成：四个字节的前缀(类似于Magic Number)，文件Meta-data信息和随机生成的16字节同步标记符。这里的Meta-data信息让人有些疑惑，它除了文件的模式外，还能包含什么。文档中指出当前Avro认定的就两个Meta-data：schema和codec。这里的codec表示对后面的文件数据块(File Data Block)采用何种压缩方式。Avro的实现都需要支持下面两种压缩方式：null(不压缩)和deflate(使用Deflate算法压缩数据块)。除了文档中认定的两种Meta-data，用户还可以自定义适用于自己的Meta-data。这里用long型来表示有多少个Meta-data数据对，也是让用户在实际应用中可以定义足够的Meta-data信息。对于每对Meta-data信息，都有一个string型的key(需要以“avro.”为前缀)和二进制编码后的value。对于文件中头信息之后的每个数据块，有这样的结构：一个long值记录当前块有多少个对象，一个long值用于记录当前块经过压缩后的字节数，真正的序列化对象和16字节长度的同步标记符。由于对象可以组织成不同的块，使用时就可以不经过反序列化而对某个数据块进行操作。还可以由数据块数，对象数和同步标记符来定位损坏的块以确保数据完整性。

### Protocol Declaration
Avro protocols describe RPC interfaces. Like schemas, they are defined with JSON text.

A protocol is a JSON object with the following attributes:

* **protocol**, a string, the name of the protocol (required);
* **namespace**, an optional string that qualifies the name;
* **doc**, an optional string describing this protocol;
* **types**, an optional list of definitions of named types (records, enums, fixed and errors). An error definition is just like a record definition except it uses "error" instead of "record". Note that forward references to named types are not permitted.
* **messages**, an optional JSON object whose keys are message names and whose values are objects whose attributes are described below. No two messages may have the same name.
The name and namespace qualification rules defined for schema objects apply to protocols as well.

#### Messages
A message has attributes:

* a **doc**, an optional description of the message,
* a **request**, a list of named, typed parameter schemas (this has the same form as the fields of a record declaration);
* a **response** schema;
* an optional union of declared **error** schemas. The effective union has "string" prepended to the declared union, to permit transmission of undeclared "system" errors. For example, if the declared error union is ["AccessError"], then the effective union is ["string", "AccessError"]. When no errors are declared, the effective error union is ["string"]. Errors are serialized using the effective union; however, a protocol's JSON declaration contains only the declared union.
 * an optional **one-way** boolean parameter.

A request parameter list is processed equivalently to an anonymous record. Since record field lists may vary between reader and writer, request parameters may also differ between the caller and responder, and such differences are resolved in the same manner as record field differences.

The one-way parameter may only be true when the response type is "null" and no errors are listed.

#### Sample Protocol
For example, one may define a simple HelloWorld protocol with:
```json
{
  "namespace": "com.acme",
  "protocol": "HelloWorld",
  "doc": "Protocol Greetings",

  "types": [
    {"name": "Greeting", "type": "record", "fields": [
      {"name": "message", "type": "string"}]},
    {"name": "Curse", "type": "error", "fields": [
      {"name": "message", "type": "string"}]}
  ],

  "messages": {
    "hello": {
      "doc": "Say hello.",
      "request": [{"name": "greeting", "type": "Greeting" }],
      "response": "Greeting",
      "errors": ["Curse"]
    }
  }
}
```
### Protocol Wire Format
#### Message Transport
消息传输，最终以字节形式传输，包括请求和响应
传输机制有多种分为有状态的和无状态的，如 http sock netty 等
http 传输，使用post 方式、二进制编码 无状态传输
#### Message Framing
在Avro中，它的消息被封装成为一组缓冲区(Buffer)，类似于下图的模型
![csmx](/assets/20180720-csmx.png)
如上图，每个缓冲区以四个字节开头，中间是多个字节的缓冲数据，最后以一个空缓冲区结尾。这种机制的好处在于，发送端在发送数据时可以很方便地组装不同数据源的数据，接收方也可以将数据存入不同的存储区。还有，当往缓冲区中写数据时，大对象可以独占一个缓冲区，而不是与其它小对象混合存放，便于接收方方便地读取大对象。
#### Handshake
avro使用握手判断两边使用的avro protocol一致，进而双方进行交互。客户端发送`HandShakeRequest`，服务端根据请求中携带的hash值进行校验响应后返回对应的`HandsHakeResponse`,客户端对`HandsHakeResponse`校验是否匹配。
不匹配进行重新握手，匹配则解析对应返回内容。
有状态传输，只进行开始的握手，无状态传输每次请求都需要进行握手。
The handshake process uses the following record schemas:
```json
{
  "type": "record",
  "name": "HandshakeRequest", "namespace":"org.apache.avro.ipc",
  "fields": [
    {"name": "clientHash",
     "type": {"type": "fixed", "name": "MD5", "size": 16}},
    {"name": "clientProtocol", "type": ["null", "string"]},
    {"name": "serverHash", "type": "MD5"},
    {"name": "meta", "type": ["null", {"type": "map", "values": "bytes"}]}
  ]
}
{
  "type": "record",
  "name": "HandshakeResponse", "namespace": "org.apache.avro.ipc",
  "fields": [
    {"name": "match",
     "type": {"type": "enum", "name": "HandshakeMatch",
              "symbols": ["BOTH", "CLIENT", "NONE"]}},
    {"name": "serverProtocol",
     "type": ["null", "string"]},
    {"name": "serverHash",
     "type": ["null", {"type": "fixed", "name": "MD5", "size": 16}]},
    {"name": "meta",
     "type": ["null", {"type": "map", "values": "bytes"}]}
  ]
}
```
#### Call Format
调用的格式包括
The format of a call request is:

* request **metadata**, a map with values of type bytes
* the **message name**, an Avro string, followed by
* the **message parameters**. Parameters are serialized according to the message's request declaration.
When the empty string is used as a message name a server should ignore the parameters and return an empty response. A client may use this to ping a server or to perform a handshake without sending a protocol message.

When a message is declared one-way and a stateful connection has been established by a successful handshake response, no response data is sent. Otherwise the format of the call response is:

* response **metadata**, a map with values of type bytes
* a one-byte **error flag** boolean, followed by either:
if the error flag is false, the message response, serialized per the message's response schema.
if the error flag is true, the error, serialized per the message's effective error union schema.
##### call request
请求avro源代码片段,`org.apache.avro.ipc.Requestor`中的`getBytes`
**request metadata** 对应 `META_WRITER.write(context.requestCallMeta(), out);`
 **message name** 对应   `out.writeString(m.getName());  `
 **message parameters**对应 ` writeRequest(m.getRequest(), request, out);`
```java
public List<ByteBuffer> getBytes()
     throws Exception {
     if (requestBytes == null) {
       ByteBufferOutputStream bbo = new ByteBufferOutputStream();
       BinaryEncoder out = ENCODER_FACTORY.binaryEncoder(bbo, encoder);

       // use local protocol to write request
       Message m = getMessage();
       context.setMessage(m);

       writeRequest(m.getRequest(), request, out); // write request payload

       out.flush();
       List<ByteBuffer> payload = bbo.getBufferList();

       writeHandshake(out);                     // prepend handshake if needed

       context.setRequestPayload(payload);
       for (RPCPlugin plugin : rpcMetaPlugins) {
         plugin.clientSendRequest(context);      // get meta-data from plugins
       }
       META_WRITER.write(context.requestCallMeta(), out);

       out.writeString(m.getName());             // write message name

       out.flush();
       bbo.append(payload);

       requestBytes = bbo.getBufferList();
     }
     return requestBytes;
   }
```
##### response
server响应 avro源代码片段`org.apache.avro.ipc.Responder`中的`respond`
其中结尾代码片段：
```java
META_WRITER.write(context.responseCallMeta(), out);
out.flush();
// Prepend handshake and append payload
bbo.prepend(handshake);
bbo.append(payload);
```
表明了响应的结果buffer顺序依次为**meta**、**handshake**、**payload**（“response”）





```java
/** Called by a server to deserialize a request, compute and serialize a
 * response or error.  Transciever is used by connection-based servers to
 * track handshake status of connection. */
public List<ByteBuffer> respond(List<ByteBuffer> buffers,
                                Transceiver connection) throws IOException {
  Decoder in = DecoderFactory.get().binaryDecoder(
      new ByteBufferInputStream(buffers), null);
  ByteBufferOutputStream bbo = new ByteBufferOutputStream();
  BinaryEncoder out = EncoderFactory.get().binaryEncoder(bbo, null);
  Exception error = null;
  RPCContext context = new RPCContext();
  List<ByteBuffer> payload = null;
  List<ByteBuffer> handshake = null;
  boolean wasConnected = connection != null && connection.isConnected();
  try {
    Protocol remote = handshake(in, out, connection);
    out.flush();
    if (remote == null)                        // handshake failed
      return bbo.getBufferList();
    handshake = bbo.getBufferList();

    // read request using remote protocol specification
    context.setRequestCallMeta(META_READER.read(null, in));
    String messageName = in.readString(null).toString();
    if (messageName.equals(""))                 // a handshake ping
      return handshake;
    Message rm = remote.getMessages().get(messageName);
    if (rm == null)
      throw new AvroRuntimeException("No such remote message: "+messageName);
    Message m = getLocal().getMessages().get(messageName);
    if (m == null)
      throw new AvroRuntimeException("No message named "+messageName
                                     +" in "+getLocal());

    Object request = readRequest(rm.getRequest(), m.getRequest(), in);

    context.setMessage(rm);
    for (RPCPlugin plugin : rpcMetaPlugins) {
      plugin.serverReceiveRequest(context);
    }

    // create response using local protocol specification
    if ((m.isOneWay() != rm.isOneWay()) && wasConnected)
      throw new AvroRuntimeException("Not both one-way: "+messageName);

    Object response = null;

    try {
      REMOTE.set(remote);
      response = respond(m, request);
      context.setResponse(response);
    } catch (Exception e) {
      error = e;
      context.setError(error);
      LOG.warn("user error", e);
    } finally {
      REMOTE.set(null);
    }

    if (m.isOneWay() && wasConnected)           // no response data
      return null;

    out.writeBoolean(error != null);
    if (error == null)
      writeResponse(m.getResponse(), response, out);
    else
      try {
        writeError(m.getErrors(), error, out);
      } catch (UnresolvedUnionException e) {    // unexpected error
        throw error;
      }
  } catch (Exception e) {                       // system error
    LOG.warn("system error", e);
    context.setError(e);
    bbo = new ByteBufferOutputStream();
    out = EncoderFactory.get().binaryEncoder(bbo, null);
    out.writeBoolean(true);
    writeError(Protocol.SYSTEM_ERRORS, new Utf8(e.toString()), out);
    if (null == handshake) {
      handshake = new ByteBufferOutputStream().getBufferList();
    }
  }
  out.flush();
  payload = bbo.getBufferList();

  // Grab meta-data from plugins
  context.setResponsePayload(payload);
  for (RPCPlugin plugin : rpcMetaPlugins) {
    plugin.serverSendResponse(context);
  }
  META_WRITER.write(context.responseCallMeta(), out);
  out.flush();
  // Prepend handshake and append payload
  bbo.prepend(handshake);
  bbo.append(payload);

  return bbo.getBufferList();
}
```
### LogicalType
见实例

*参考博客*
>[Avro总结(RPC/序列化)](http://langyu.iteye.com/blog/708568)
>[Avro与JAVA](https://blog.csdn.net/hu2010shuai/article/details/53010350)

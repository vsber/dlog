---
layout: post
title: Apache Avro Source
---
# avro 源码分析--请求发送

Mail实例中`Mail proxy = SpecificRequestor.getClient(Mail.class, client);`

实际是获取Mail类代理对象，`SpecificRequestor`中
```java
public static  <T> T getClient(Class<T> iface, Transceiver transciever)
  throws IOException {
  return getClient(iface, transciever,
                   new SpecificData(iface.getClassLoader()));
}

/** Create a proxy instance whose methods invoke RPCs. */
@SuppressWarnings("unchecked")
public static  <T> T getClient(Class<T> iface, Transceiver transciever,
                               SpecificData data)
  throws IOException {
  Protocol protocol = data.getProtocol(iface);
  return (T)Proxy.newProxyInstance
    (data.getClassLoader(),
     new Class[] { iface },
     new SpecificRequestor(protocol, transciever, data));
}
```
invocationHandler为SpecificRequestor对象，其invoke方法查看得知最终是request 请求
实际是`return request(method.getName(), finalArgs, callback);`或者
`return request(method.getName(), args);`
```java
@Override
public Object invoke(Object proxy, Method method, Object[] args)
  throws Throwable {
  String name = method.getName();
  if (name.equals("hashCode")) {
    return hashCode();
  }
  else if (name.equals("equals")) {
    Object obj = args[0];
    return (proxy == obj) || (obj != null && Proxy.isProxyClass(obj.getClass())
                              && this.equals(Proxy.getInvocationHandler(obj)));
  }
  else if (name.equals("toString")) {
    String protocol = "unknown";
    String remote = "unknown";
    Class<?>[] interfaces = proxy.getClass().getInterfaces();
    if (interfaces.length > 0) {
      try {
        protocol = Class.forName(interfaces[0].getName()).getSimpleName();
      } catch (ClassNotFoundException e) {
      }

      InvocationHandler handler = Proxy.getInvocationHandler(proxy);
      if (handler instanceof Requestor) {
        try {
          remote = ((Requestor) handler).getTransceiver().getRemoteName();
        } catch (IOException e) {
        }
      }
    }
    return "Proxy[" + protocol + "," + remote + "]";
  }
  else {
    try {
      // Check if this is a callback-based RPC:
      Type[] parameterTypes = method.getParameterTypes();
      if ((parameterTypes.length > 0) &&
          (parameterTypes[parameterTypes.length - 1] instanceof Class) &&
          Callback.class.isAssignableFrom(((Class<?>)parameterTypes[parameterTypes.length - 1]))) {
        // Extract the Callback from the end of of the argument list
        Object[] finalArgs = Arrays.copyOf(args, args.length - 1);
        Callback<?> callback = (Callback<?>)args[args.length - 1];
        request(method.getName(), finalArgs, callback);
        return null;
      }
      else {
        return request(method.getName(), args);
      }
    } catch (Exception e) {
      // Check if this is a declared Exception:
      for (Class<?> exceptionClass : method.getExceptionTypes()) {
        if (exceptionClass.isAssignableFrom(e.getClass())) {
          throw e;
        }
      }

      // Next, check for RuntimeExceptions:
      if (e instanceof RuntimeException) {
        throw e;
      }

      // Not an expected Exception, so wrap it in AvroRemoteException:
      throw new AvroRemoteException(e);
    }
  }
}
```
继续查看`Requestor.request（）`
```java
t.transceive(request.getBytes(),new TransceiverCallback<T>(request, callFuture));
```
可以看到`request.getBytes()`中对请求进行序列化的

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
对其中一个`writeRequest(m.getRequest(), request, out)`处理进行分析，方法调用进入到
```java
/** Called to write data.*/
 protected void write(Schema schema, Object datum, Encoder out)
     throws IOException {
   LogicalType logicalType = schema.getLogicalType();
   if (datum != null && logicalType != null) {
     Conversion<?> conversion = getData()
         .getConversionByClass(datum.getClass(), logicalType);
     writeWithoutConversion(schema,
         convert(schema, logicalType, conversion, datum), out);
   } else {
     writeWithoutConversion(schema, datum, out);
   }
 }
```
```java
/** Called to write data.*/
 protected void writeWithoutConversion(Schema schema, Object datum, Encoder out)
   throws IOException {
   try {
     switch (schema.getType()) {
     case RECORD: writeRecord(schema, datum, out); break;
     case ENUM:   writeEnum(schema, datum, out);   break;
     case ARRAY:  writeArray(schema, datum, out);  break;
     case MAP:    writeMap(schema, datum, out);    break;
     case UNION:
       int index = resolveUnion(schema, datum);
       out.writeIndex(index);
       write(schema.getTypes().get(index), datum, out);
       break;
     case FIXED:   writeFixed(schema, datum, out);   break;
     case STRING:  writeString(schema, datum, out);  break;
     case BYTES:   writeBytes(datum, out);           break;
     case INT:     out.writeInt(((Number)datum).intValue()); break;
     case LONG:    out.writeLong((Long)datum);       break;
     case FLOAT:   out.writeFloat((Float)datum);     break;
     case DOUBLE:  out.writeDouble((Double)datum);   break;
     case BOOLEAN: out.writeBoolean((Boolean)datum); break;
     case NULL:    out.writeNull();                  break;
     default: error(schema,datum);
     }
   } catch (NullPointerException e) {
     throw npe(e, " of "+schema.getFullName());
   }
 }



  /** Called to write a record.  May be overridden for alternate record
   * representations.*/
  protected void writeRecord(Schema schema, Object datum, Encoder out)
    throws IOException {
    Object state = data.getRecordState(datum, schema);
    for (Field f : schema.getFields()) {
      writeField(datum, f, out, state);
    }
  }

  /** Called to write a single field of a record. May be overridden for more
   * efficient or alternate implementations.*/
  protected void writeField(Object datum, Field f, Encoder out, Object state)
      throws IOException {
    Object value = data.getField(datum, f.name(), f.pos(), state);
    try {
      write(f.schema(), value, out);
    } catch (NullPointerException e) {
      throw npe(e, " in field " + f.name());
    }
  }
 ```

 `writeField(Object datum, Field f, Encoder out, Object state)` 和 `write(Schema schema, Object datum, Encoder out)` 遍历进行深度优先写入

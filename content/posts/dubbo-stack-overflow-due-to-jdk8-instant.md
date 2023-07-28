---
title: "JDK8日期时间API导致Dubbo调用StackOverflow错误"
subtitle: ""
date: 2022-04-30T11:34:02+08:00
draft: false

tags:
- Dubbo
- Java
categories:
- coding

---

<!--more-->



## 0 前言

博客搭建之后，不知道从何写起。思来想去，决定从记录一些工作中遇到的问题开始。一来对问题有个归档，算是有一些积累，方便日后回顾；二来促使自己遇到问题时能探本朔源，做到知其然知其所以然，不要草草百度了事。不积跬步，无以至千里；不积小流，无以成江海。



## 1 问题

最近项目中使用dubbo（**v2.5.3**）的时候，遇到一个问题：dubbo服务方在对response编码时失败，发生了栈溢出异常。可以通过一个demo([dubbo-stack-overflow-due-to-jkd8-instant](https://github.com/dingyufan/blog-demo-all/tree/main/dubbo-stack-overflow-due-to-jkd8-instant))复现这个异常。

![image-20220421171300795](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/image-20220421171300795.png)

看到栈溢出，一般想到的情况要么是递归过深，或是循环调用方法次数过多。为什么会在dubbo序列化编码的时候发生这个异常呢？通过查看源码提交记录，结合异常信息Fail to encode repsonse，并且在异常栈中出现JavaSerializer类，判断**问题出在Dubbo在对Response序列化时无法处理Instant类型对象**。

网上搜索到的博客基本上只是说了有这样的问题，但是没有说明具体的原因，都是草草改用Date了事。通过这篇博客，记录一下Instant类型对象引起Dubbo对Response序列化失败导致栈溢出的根本原因以及解决方案。



## 3 Dubbo序列化Response

dubbo对不同协议、不同序列化方式处理方式是不一样的。此项目使用默认协议：**dubbo协议**，同时也采用dubbo协议默认序列化方式**：hessian2**。下文的序列化过程分析就是基于此的。

```xml
<!-- dubbo协议hessian2序列化方式，是dubbo的默认值。所以在xml配置中写不写这行配置都可以 -->
<dubbo:protocol name="dubbo" serialization="hessian2"/>
```

**参考官网文档 [2.4 服务提供方返回调用结果](https://dubbo.apache.org/zh/docs/v2.7/dev/source/service-invoking-process/#24-%E6%9C%8D%E5%8A%A1%E6%8F%90%E4%BE%9B%E6%96%B9%E8%BF%94%E5%9B%9E%E8%B0%83%E7%94%A8%E7%BB%93%E6%9E%9C) **，我们从Dubbo处理返回结果入手。



###  3.1 ExchangeCodec / DubboCodec

ExchangeCodec是Dubbo **exchange信息交换层**的编码器。`ExchangeCodec#encode(Channel, ChannelBuffer, Object)`方法是Dubbo对Request和Response进行编码的入口，逻辑很简单，主要是根据消息的类型，交由对应方法进行编码，对于不是自己负责的非Request、Response类型的消息对象，交给父类处理。

```java
// ExchangeCodec.java
public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
    if (msg instanceof Request) {
        encodeRequest(channel, buffer, (Request) msg);
    } else if (msg instanceof Response) {
        // 对response对象进行编码
        encodeResponse(channel, buffer, (Response) msg);
    } else {
        super.encode(channel, buffer, msg);
    }
}
```

`ExchangeCodec#encodeResponse(Channel, ChannelBuffer, Response)`方法主要做的就是组装消息写入ChannelBuffer，这样消息就可以发送给调用方。消息两部分组成：**消息头**，包含状态序列化等相关信息的字节数组；**消息体**：Response返回的具体数据转换成的字节数组。

```java
// ExchangeCodec.java
protected void encodeResponse(Channel channel, ChannelBuffer buffer, Response res) throws IOException {
    try {
        // 获取序列化方式*
        Serialization serialization = getSerialization(channel);
        // 消息头字节数组。长度16
        byte[] header = new byte[HEADER_LENGTH];
        // 消息头 写入魔数
        Bytes.short2bytes(MAGIC, header);
        // 消息头 写入序列化器ID信息
        header[2] = serialization.getContentTypeId();
        if (res.isHeartbeat()) header[2] |= FLAG_EVENT;
        byte status = res.getStatus();
        // 消息头 写入状态
        header[3] = status;
        // 消息头 写入
        Bytes.long2bytes(res.getId(), header, 4);

        int savedWriteIndex = buffer.writerIndex();
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
        // 得到序列化器*
        ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
        if (status == Response.OK) {
            if (res.isHeartbeat()) {
                encodeHeartbeatData(channel, out, res.getResult());
            } else {
                // 对调用结果进行序列化*
                // 这个方法会调用子类DubboCodec的override方法实现
                encodeResponseData(channel, out, res.getResult());
            }
        }
        else out.writeUTF(res.getErrorMessage());
        out.flushBuffer();
        bos.flush();
        bos.close();
        // 消息体字节长度
        int len = bos.writtenBytes();
        checkPayload(channel, len);
        Bytes.int2bytes(len, header, 12);
        // 先写完消息体，再写消息头，再设置writeIndex位置
        buffer.writerIndex(savedWriteIndex);
        buffer.writeBytes(header); 
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
    } catch (Throwable t) {
        // 略...
    }
}
```

在这里先不关注消息头、ChannelBuffer等操作，这次主要看序列化相关的几个步骤：

#### 3.1.1 获取序列化方式

这里实际是父类中的方法`AbstractCodec#getSerialization(Channel)`方法。这一步是利用Dubbo SPI机制，实例化一个序列化方式对象。具体过程不是本次的重点，只要理解这一步会根据`<dubbo:protocol>`配置的序列化方式返回对应的序列化方式对象。比如这里得到的就是Hessian2Serialization对象。

#### 3.1.2 得到序列化器

这里调用`Serialization#serialize(URL, OutputStream)`方法，得到一个序列化器。hessian2序列化提供的序列化器是Hessian2ObjectOutput对象，通过构造函数传入了`ExchangeCodec#encodeResponse(Channel, ChannelBuffer, Response)`方法中定义的包装了buffer的bos。这样Hessian2ObjectOutput对象可以将具体数据写入到buffer中。

```java
public class Hessian2Serialization implements Serialization {
    
    public static final byte ID = 2;

    public byte getContentTypeId() {
        return ID;
    }

    public String getContentType() {
        return "x-application/hessian2";
    }

    public ObjectOutput serialize(URL url, OutputStream out) throws IOException {
        // 序列化器对象
        return new Hessian2ObjectOutput(out);
    }

    public ObjectInput deserialize(URL url, InputStream is) throws IOException {
        return new Hessian2ObjectInput(is);
    }
}
```

#### 3.1.3 对调用结果进行序列化

配置中指定的是dubbo协议，这里实际会被调用的是`DubboCodec#encodeResponseData(Channel, ObjectOutput, Object)`重写方法。DubboCodec继承自ExchangeCodec，是Dubbo **protocol远程调用层**的编码器，主要负责实现协议层面的Response数据编码。

```java
// DubboCodec.java
@Override
protected void encodeResponseData(Channel channel, ObjectOutput out, Object data) throws IOException {
    Result result = (Result) data;

    Throwable th = result.getException();
    if (th == null) {
        Object ret = result.getValue();
        if (ret == null) {
            out.writeByte(RESPONSE_NULL_VALUE);
        } else {
            // 序列化器中写入response数据标志位
            out.writeByte(RESPONSE_VALUE);
            // 序列化器中写入正常返回的对象
            out.writeObject(ret);
        }
    } else {
        out.writeByte(RESPONSE_WITH_EXCEPTION);
        out.writeObject(th);
    }
}
```



### 3.2 Hessian2ObjectOutput

在3.1.2中提到过，得到的序列化器是Hessian2ObjectOutput，实现了ObjectOutput接口。对于常见的简单数据对象，ObjectOutput接口中都定义了相应处理方法，同时还定义writeObject(Object)方法处理复杂对象的序列化。

Hessian2ObjectOutput实现了接口定义的各个方法，通过一个Hessian2Output对象，完成各种数据类型hessian2序列化的具体操作。

```java
public class Hessian2ObjectOutput implements ObjectOutput {
	private final Hessian2Output mH2o;

	public Hessian2ObjectOutput(OutputStream os) {
		mH2o = new Hessian2Output(os);
		mH2o.setSerializerFactory(Hessian2SerializerFactory.SERIALIZER_FACTORY);
	}
    
    public void writeBool(boolean v) throws IOException {
		mH2o.writeBoolean(v);
	}
    // 略...
    
    public void writeObject(Object obj) throws IOException {
		mH2o.writeObject(obj);
	}
    
    public void flushBuffer() throws IOException {
		mH2o.flushBuffer();
	}
}
```

**`Hessian2ObjectOutput#writeObject(Object)`方法是response数据对象序列化的入口**，数据对象序列化的操作从这一步正式开始。这个方法中调用的是`Hessian2Output#writeObject(Object)`方法。



### 3.3 Hessian2Output

注意Hessian2Output的方法，如`writeBoolean(boolean)`，对值进行编码写入的时候，没有直接写入` _os`，而是维护一个字节数组  `_buffer`和  `_offset`值。回头看3.1中的`ExchangeCodec#encodeResponse(Channel, ChannelBuffer, Response)`方法，会调用`out.flushBuffer()`，通过Hessian2ObjectOutput来调用`Hessian2Output#flushBuffer()`方法，将 `_buffer`内容写入 `_os` ，即写入 `bos`，即ChannelBuffer对象`buffer`。

`Hessian2Output#writeObject(Object)`方法不像其他方法，没有直接向`_buffer`写入具体的字节，而是尝试获取一个Serializer对象，完成对象的序列化。

```java
public class Hessian2Output extends AbstractHessianOutput implements Hessian2Constants {
	protected OutputStream _os;
    private final byte []_buffer = new byte[SIZE];
    // 略...
    
    public Hessian2Output(OutputStream os) {
        _os = os;
    }
    // 略...
    
    public void writeBoolean(boolean value) throws IOException {
        if (SIZE < _offset + 16)
            flush();

        if (value)
            // 不直接写入_os
            _buffer[_offset++] = (byte) 'T';
        else
            _buffer[_offset++] = (byte) 'F';
    }
    
    public void writeObject(Object object) throws IOException {
        if (object == null) {
            writeNull();
            return;
        }
        
        Serializer serializer;
        // 获得序列化工厂类，获得一个Serializer对象
        serializer = findSerializerFactory().getSerializer(object.getClass());
        // 完成对象的序列化
        serializer.writeObject(object, this);
    }
    
    public final void flushBuffer() throws IOException {
        int offset = _offset;
        if (! _isStreaming && offset > 0) {
            _offset = 0;
            _os.write(_buffer, 0, offset);
        } else if (_isStreaming && offset > 3) {
            int len = offset - 3;
            _buffer[0] = 'p';
            _buffer[1] = (byte) (len >> 8);
            _buffer[2] = (byte) len;
            _offset = 3;
            
            _os.write(_buffer, 0, offset);
        }
  }
}

```

findSerializerFactory()会从父类获得一个SerializerFactory对象，回头从3.2看到，在Hessian2ObjectOutput中，为Hessian2Output指定SerializerFactory 具体对象为`Hessian2SerializerFactory.SERIALIZER_FACTORY`。我们主要来看getSerializer(object.getClass())



### 3.4 SerializerFactory / Hessian2SerializerFactory

Hessian2SerializerFactory相对比较简单，只是实现了getClassLoader() 方法，然后在公共静态常量量中构造了一个Hessian2SerializerFactory对象供使用。

```java
public class Hessian2SerializerFactory extends SerializerFactory {

	public static final SerializerFactory SERIALIZER_FACTORY = new Hessian2SerializerFactory();

	private Hessian2SerializerFactory() {
	}

	@Override
	public ClassLoader getClassLoader() {
		return Thread.currentThread().getContextClassLoader();
	}

}
```

他父类SerializerFactory功能更加丰富。`SerializerFactory#getSerializer(Class)`方法比较长，但是其实整体逻辑并不复杂：**根据类型Class返回相应的Serializer实例，如果类型和方法中列举的都不匹配，则会返回一个默认的Serializer对象，即 JavaSerializer对象**。

```java
public class SerializerFactory extends AbstractSerializerFactory {
    // 略...
    
    public Serializer getSerializer(Class cl) throws HessianProtocolException {
        Serializer serializer;
        serializer = (Serializer) _staticSerializerMap.get(cl);
        
        if (serializer != null)
            return serializer;
        
        if (_cachedSerializerMap != null) {
            synchronized (_cachedSerializerMap) {
                serializer = (Serializer) _cachedSerializerMap.get(cl);
            }
            if (serializer != null)
                return serializer;
        }
        
        for (int i = 0; serializer == null && _factories != null && i < _factories.size(); i++) {
            AbstractSerializerFactory factory;
            factory = (AbstractSerializerFactory) _factories.get(i);
            serializer = factory.getSerializer(cl);
        }

        if (serializer != null) {

        } else if (JavaSerializer.getWriteReplace(cl) != null)
            serializer = new JavaSerializer(cl, _loader);
        else if (HessianRemoteObject.class.isAssignableFrom(cl))
          serializer = new RemoteSerializer();
//      else if (BurlapRemoteObject.class.isAssignableFrom(cl))
//        serializer = new RemoteSerializer();
        else if (Map.class.isAssignableFrom(cl)) {
            if (_mapSerializer == null)
                _mapSerializer = new MapSerializer();
            serializer = _mapSerializer;
        }
        else if (Collection.class.isAssignableFrom(cl)) {
            if (_collectionSerializer == null) {
                _collectionSerializer = new CollectionSerializer();
            }
            serializer = _collectionSerializer;
        }
        else if (cl.isArray())
          serializer = new ArraySerializer();
        else if (Throwable.class.isAssignableFrom(cl))
          serializer = new ThrowableSerializer(cl, getClassLoader());
        else if (InputStream.class.isAssignableFrom(cl))
          serializer = new InputStreamSerializer();
        else if (Iterator.class.isAssignableFrom(cl))
          serializer = IteratorSerializer.create();
        else if (Enumeration.class.isAssignableFrom(cl))
          serializer = EnumerationSerializer.create();
        else if (Calendar.class.isAssignableFrom(cl))
          serializer = CalendarSerializer.create();
        else if (Locale.class.isAssignableFrom(cl))
          serializer = LocaleSerializer.create();
        else if (Enum.class.isAssignableFrom(cl))
          serializer = new EnumSerializer(cl);

        if (serializer == null)
            // 都不符合以上类似，则返回默认的serializer
            serializer = getDefaultSerializer(cl);

        if (_cachedSerializerMap == null)
            _cachedSerializerMap = new HashMap(8);

        synchronized (_cachedSerializerMap) {
            // 缓存这个类型对应的serializer，下次直接从缓存获取
            _cachedSerializerMap.put(cl, serializer);
        }

        return serializer;
    }
    
    protected Serializer getDefaultSerializer(Class cl) {
        if (_defaultSerializer != null)
            return _defaultSerializer;
        if (!Serializable.class.isAssignableFrom(cl) && ! _isAllowNonSerializable) {
            throw new IllegalStateException("Serialized class " + cl.getName() + " must implement java.io.Serializable");
        }
        
        return new JavaSerializer(cl, _loader);
  }
}
```

对于demo中的返回的数据对象DemoDTO，很显然就是会得到一个JavaSerializer，通过它的完成序列化。



### 3.5 JavaSerializer

回顾上文3.3节中，serializer对象完成序列哈的入库也是writeObject()方法。来看`JavaSerializer#writeObject(Object, AbstractHessianOutput)`方法。这里面序列化分两种情况：一是要序列化的对象有writeReplce()方法的，直接调用writeReplce()方法完成序列化；而是常规的使用静态类FieldSerializer序列化。

```java
// JavaSerializer.java
public void writeObject(Object obj, AbstractHessianOutput out) throws IOException {
    if (out.addRef(obj)) {
        return;
    }

    Class cl = obj.getClass();
    try {
        // 调用序列化对象的writeReplace方法完成序列化
        if (_writeReplace != null) {
            Object repl;
            if (_writeReplaceFactory != null)
                repl = _writeReplace.invoke(_writeReplaceFactory, obj);
            else
                repl = _writeReplace.invoke(obj);

            out.removeRef(obj);
            out.writeObject(repl);
            out.replaceRef(repl, obj);
            return;
        }
    } catch (RuntimeException e) {
        throw e;
    } catch (Exception e) {
        // log.log(Level.FINE, e.toString(), e);
        throw new RuntimeException(e);
    }

    // hessian2会写入对象开始标记;hessian会写map开始标记并返回-2
    int ref = out.writeObjectBegin(cl.getName());

    if (ref < -1) {
        // 这种情况看起来是兼容hessian序列化的，而不是hessian2序列化的
        // hessian序列化是把对象当做map处理，所以writeObject10()方法里会循环写入字段名称然后马上字段序列化，最后还会写一个map结束标志
        writeObject10(obj, out);
    }
    else {
        if (ref == -1) {
            // 写入对象包含字段长度，以及各个类型字段信息
            writeDefinition20(out);
            // 类型信息编码处理
            out.writeObjectBegin(cl.getName());
        }
        // 这里会开始会遍历序列化对象包含的各个字段
        writeInstance(obj, out);
    }
}
```



#### 3.5.1 writeReplace方法

JavaSerializer 使用一个Method变量 `_writeReplace`保存writeReplace()方法，这个方法是从那里来的呢？在JavaSerializer构造方法中，会调用一个私有方法`introspectWriteReplace(Class, ClassLoader)`。它会从两个地方找writeReplace()方法：**一是序列化类相应的HessianSerializer类；二是序列化类本身**。但是要注意，HessianSerializer类的writeReplace方法签名 和 序列化类本身writeReplace方法签名 是有区别的。

```java
// JavaSerializer.java
private void introspectWriteReplace(Class cl, ClassLoader loader) {
    try {
        String className = cl.getName() + "HessianSerializer";
        // 加载 序列化类类名+HessianSerializer的类。例如demo中的DemoDTO类，则会
        // 尝试加载cn.dingyufan.blog.demo.dubbostackoverflowduetojkd8instantprovider.api.dto.DemoDTOHessianSerializer类
        Class serializerClass = Class.forName(className, false, loader);
        // 实例化相应的HessianSerializer类型
        Object serializerObject = serializerClass.newInstance();
        // 反射获取WriteReplace方法
        Method writeReplace = getWriteReplace(serializerClass, cl);

        if (writeReplace != null) {
            _writeReplaceFactory = serializerObject;
            _writeReplace = writeReplace;
            return;
        }
    } catch (ClassNotFoundException e) {
        // 没有相应HessianSerializer类就会抛出ClassNotFoundException，这个异常不处理，后面会再次尝试从序列化类本身寻找
    } catch (Exception e) {
        log.log(Level.FINER, e.toString(), e);
    }
    // 从序列化类本身找writeReplace方法
    _writeReplace = getWriteReplace(cl);
}

// 从要序列化的类型寻找 方法名为writeReplace、入参为空 的方法
protected static Method getWriteReplace(Class cl) {
    for (; cl != null; cl = cl.getSuperclass()) {
        Method []methods = cl.getDeclaredMethods();

        for (int i = 0; i < methods.length; i++) {
            Method method = methods[i];

            if (method.getName().equals("writeReplace") &&
                method.getParameterTypes().length == 0)
                return method;
        }
    }

    return null;
}

// 从相应HessianSerializer类找到  方法名为writeReplace、 入参有且仅为要序列化类型  的方法
protected Method getWriteReplace(Class cl, Class param)
{
    for (; cl != null; cl = cl.getSuperclass()) {
        for (Method method : cl.getDeclaredMethods()) {
            if (method.getName().equals("writeReplace")
                && method.getParameterTypes().length == 1
                && param.equals(method.getParameterTypes()[0]))
                return method;
        }
    }

    return null;
}
```

writeReplace方法提供了一种扩展序列化的方式，对与Dubbo暂不支持的类型，可以通过这个扩展点，返回一个自定义的类型的对象，替代原本需要序列化的对象，然后对自定义的对象进行序列化。

但是有个问题，**如果producer通过writeReplace方法自定义一个对象来序列化，consumer反序列化时，怎么把自定义的对象转回原本类型呢？**这里不展开讲，但是办法肯定是有的，答案就是**readResolve方法** 。: )



#### 3.5.2 FieldSerializer

对于没有writeReplace方法的类型，就继续走hessian2设定的序列化逻辑。从`JavaSerializer#writeInstance(Object, AbstractHessianOutput)`继续。

```java
// JavaSerializer.java
public void writeInstance(Object obj, AbstractHessianOutput out) throws IOException {
    for (int i = 0; i < _fields.length; i++) {
        Field field = _fields[i];
        // _fieldSerializers是数组 FieldSerializer[]
        // 是在JavaSerializer构造方法中，根据序列化类中各个字段类型，依次创建FieldSerializer(或子类)的实例存入
        _fieldSerializers[i].serialize(out, obj, field);
    }
}

// JavaSerializer构造方法中调用
// 根据序列化类中各个字段类型，返回对应类型的FieldSerializer(或子类)实例
private static FieldSerializer getFieldSerializer(Class type)
{
    if (int.class.equals(type)
        || byte.class.equals(type)
        || short.class.equals(type)
        || int.class.equals(type)) {
        return IntFieldSerializer.SER;
    }
    else if (long.class.equals(type)) {
        return LongFieldSerializer.SER;
    }
    else if (double.class.equals(type) ||
             float.class.equals(type)) {
        return DoubleFieldSerializer.SER;
    }
    else if (boolean.class.equals(type)) {
        return BooleanFieldSerializer.SER;
    }
    else if (String.class.equals(type)) {
        return StringFieldSerializer.SER;
    }
    else if (java.util.Date.class.equals(type)
             || java.sql.Date.class.equals(type)
             || java.sql.Timestamp.class.equals(type)
             || java.sql.Time.class.equals(type)) {
        return DateFieldSerializer.SER;
    }
    else
        return FieldSerializer.SER;
}
```

在`JavaSerializer#writeObject(Object, AbstractHessianOutput)`方法中，写完类型信息、类型字段长度、各字段名称后，就会通过writeInstance方法依次序列化类型中的各字段。字段的序列化，就是由静态类FieldSerializer类完成。

比如DemoDTO中的String类型字段，就会有FieldSerializer的子类StringFieldSerializer，完成`String consumer`字段序列化。

```java
// JavaSerializer.java
static class StringFieldSerializer extends FieldSerializer {
    static final FieldSerializer SER = new StringFieldSerializer();
    
    void serialize(AbstractHessianOutput out, Object obj, Field field) throws IOException {
        String value = null;
        try {
            value = (String) field.get(obj);
        } catch (IllegalAccessException e) {
            log.log(Level.FINE, e.toString(), e);
        }
        // 调用Hessian2Output类的writeString方法，值写入输出流
        out.writeString(value);
    }
}
```

但是DemoDTO中的Instant类型字段，在`JavaSerializer#getFieldSerializer(Class)`方法中没有对应类型的，得到的是默认的FieldSerializer实例。

```java
static class FieldSerializer {
    static final FieldSerializer SER = new FieldSerializer();

    void serialize(AbstractHessianOutput out, Object obj, Field field) throws IOException {
        Object value = null;

        try {
            value = field.get(obj);
        } catch (IllegalAccessException e) {
            log.log(Level.FINE, e.toString(), e);
        }
        
        try {
            // 又回到 Hessian2Output#writeObject(Object)方法
            // 再次获得序列化工厂类Hessian2SerializerFactory，获得一个Serializer对象，又通过Serializer对象序列化
            // 这里有递归的感觉了
            out.writeObject(value);
        } catch (RuntimeException e) {
            throw new RuntimeException(e.getMessage() + "\n Java field: " + field, e);
        } catch (IOException e) {
            throw new IOExceptionWrapper(e.getMessage() + "\n Java field: " + field, e);
        }
    }
}
```



综上所述，Hessian2序列化的核心的几个类是：Hessian2Output、SerializerFactory(Hessian2SerializerFactory) 、JavaSerializer和FieldSerializer。

整体思路是：

- `Hessian2Output#writeObject(Object)`：对象 序列化的入口
- `SerializerFactory#getSerializer(Class)`：根据对象类型，找到对应类型的Serializer完成对象序列化，无对应类型的交给JavaSerializer
- `JavaSerializer#getFieldSerializer(Class)`：完成对象类型信息的序列化，然后依次序列化各个字段，为各字段找到对应类型的FieldSerializer子类完成字段序列化，无对应类型的字段交给FieldSerializer。
- `FieldSerializer#serialize(AbstractHessianOutput, Object, Field)`：FieldSerializer会把字段值再当做一个对象，再走一遍整个对象序列化过程。

这个设计想法是蛮好的，**简单对象直接有对应类序列化，没有对应序列化方法的把对象的序列化拆解成各个字段的序列化。在理想的情况下，所有对象都可以通过不断的拆解，形成简单的对象，然后有相应的Serializer或FieldSerializer完成序列化**。就好像像复杂问题拆解成数个简单问题，也有点像庖丁解牛的感觉。



### 3.6 序列化调用栈

了解dubbo在hessian2序列化方式时的逻辑之后，再来看本文开头的异常。上述大段文字内容繁多，很难串联起来，我们根据hessian2的序列化逻辑，整理整个序列化过程的调用栈，当然也可以直接打断点看

```java
ExchangeCodec#encode
  ->ExchangeCodec#encodeResponseData
    ->DubboCodec#encodeResponseData
      ->Hessian2ObjectOutput#writeObject
        // DemoDTO类型开始序列化
        ->Hessian2Output#writeObject
          ->SerializerFactory#getSerializer
            ->JavaSerializer#writeObject
              ->JavaSerializer#writeInstance
                ->FieldSerializer#serialize
                  // Instant类型开始序列化
                  ->Hessian2Output#writeObject
                    ->SerializerFactory#getSerializer
                      ->JavaSerializer#writeObject
                        // Ser类型开始序列化(Instant的writeReplace方法产生的替代对象)
                        ->Hessian2Output#writeObject
                          ->SerializerFactory#getSerializer
                            ->JavaSerializer#writeObject
                              ->JavaSerializer#writeInstance
                                ->FieldSerializer#serialize
                                  // Instant类型开始序列化(Ser类中的object字段)
                                  ->Hessian2Output#writeObject
                                    ->SerializerFactory#getSerializer
                                      ->JavaSerializer#writeObject
                                        // Ser(Instant的writeReplace方法产生的替代类)
                                        ->Hessian2Output#writeObject
                                          -> ......
```

通过调用栈，是不是一眼就发现问题所在了？这个调用栈倒序来看，可以发现正好开头是异常栈的顺序。



## 4 结论

在Dubbo反序列化的过程中，产生的栈溢出的原因是：

**Dubbo(v2.5.3)没有相应的Serializer或FieldSerializer能处理Instant数据类型，并且Instant正好有writeReplace方法，返回的替代类中的object字段又包含Instant本身。导致需要序列化的对象在Instant、Ser之间反复横跳，不断调用序列化方法逻辑，最终方法栈满，导致栈溢出错误。**

结合网上信息，可以发现不只是Instant类，在JDK8新增的时间API，如LocalDateTime、Period等，都会和Instant有一样的问题，也是一样的原因。



## 5 解决方案

知道了问题的原因之后，就可以对症下药了。本人水平有限，暂时想到以下几种方案：

### 5.1 使用Date类型

首先最简单的方式，就是不使用JDK8中引入的时间API，改为使用Date。

这种方式没有直接解决问题，而是有些逃避问题了。



### 5.1 writeReplace / readResolve 方法

修改Instant的writeReplace方法可能不是很方便，我们可以为DemoDTO添加writeReplace方法。总体思路的话就是：provider把Instant拆解成支持的数据类型，到了consumer再转化成Instant。

```java
public class DemoDTO implements Serializable {

    private static final long serialVersionUID = -311647434535770294L;

    private String consumer;
    private Instant instant;


    // 序列化时JavaSerializer会调用此方法，然后序列化替代类对象repl
    private DemoDTOHandle writeReplace() {
        System.out.println("call writeReplace");
        DemoDTOHandle repl = new DemoDTOHandle();
        repl.setConsumer(consumer);
        if (instant != null) {
            repl.setSeconds(instant.getEpochSecond());
            repl.setNanos(instant.getNano());
        }
        return repl;
    }
    
    // getter,setter略...
}
```

DemoDTOHandle中Instant拆成seconds、nanos字段存储，然后会代替DemoDTO进行序列化传输。代码如下。

```java
public class DemoDTOHandle implements HessianHandle, Serializable {

    private static final long serialVersionUID = -8532997896892606136L;

    private String consumer;

    private long seconds;
    private int nanos;

    // 反序列化时，JavaDeserializer会调用此方法，重新组装实际的类
    private DemoDTO readResolve() {
        System.out.println("call readResolve");
        DemoDTO dto = new DemoDTO();
        dto.setConsumer(consumer);
        dto.setInstant(Instant.ofEpochSecond(seconds, nanos));
        return dto;
    }
    
    // getter,setter略...
}
```

DemoDTOHandle为什么要实现HessianHandle接口呢？答案在`SerializerFactory#getObjectDeserializer(String, Class)`中，如果不是HessianHandle实现类的话，是得不到DemoDTOHandle相关的Deserializer，只有DemoDTO相关的Deserializer，那么就无法使用DemoDTOHandle中的readResolve方法转换回对象了。

![image-20220504103337575](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/image-20220504103337575.png)



### 5.2 修改序列化方式

之前提到，使用的是dubbo协议、hessian2序列化方式。事实上dubbo协议同样可以使用其他多种序列化方式。下面列举的序列化方式，都可以支持JDK8时间API的序列化处理。

```xml
<dubbo:protocol serialization="java"/>

<dubbo:protocol serialization="compactedjava"/>

<dubbo:protocol serialization="nativejava"/>

<dubbo:protocol serialization="json"/>

<dubbo:protocol serialization="fastjson"/>
```

当然如果升级了Dubbo版本的话，会支持更多的序列化方式。



### 5.3 升级Dubbo版本（√）

当然，使用的Dubbo版本是2.5.3，实在是太老了。搜索maven仓库，看到2.5.3版本更新时间是2012年，十年前。

![image-20220503122841533](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/image-20220503122841533.png)

**可以选择升级到2.6.x版本**。会发现2.6.x版本在序列化方面也做了一些优化。新版本不仅包含了新功能，扩展性也有所提升。但是需要注意，如果是升级2.6.6+版本，需要关注netty版本的变化。

#### 2.6.x 支持JDK8时间API

在2.6.x版本中，在SerializerFactory 中增加了对JDK8时间API的处理，加入了Java8TimeSerializer以及一系列的HessianHandle。**意味着2.6.x版本Dubbo将不再有本文开头遇到的问题。**

![image-20220503123444697](https://dingyufan-github-io.oss-cn-hangzhou.aliyuncs.com/image-20220503123444697.png)

#### 2.6.x 支持扩展SerializerFactory 

同时SerializerFactory 也提供了一定的扩展性，支持添加你想要的SerializerFactory。如果还有目标序列化方式无法处理的类型，可以自定义SerializerFactory、Serializer，只要将自定义SerializerFactory添加到当前序列化方式的SerializerFactory 中，就可以轻松扩展。**在2.5.x上是不支持这种扩展的**。

```java
// SerializerFactory.java  2.6.x
public class SerializerFactory extends AbstractSerializerFactory {
    // 略...
    
    protected ArrayList _factories = new ArrayList();
    // 略...
    
    public void addFactory(AbstractSerializerFactory factory) {
        _factories.add(factory);
    }
    
    public Serializer getSerializer(Class cl) throws HessianProtocolException{
        Serializer serializer;
        serializer = (Serializer) _staticSerializerMap.get(cl);
        if (serializer != null)
            return serializer;

        if (_cachedSerializerMap != null) {
            synchronized (_cachedSerializerMap) {
                serializer = (Serializer) _cachedSerializerMap.get(cl);
            }
            if (serializer != null)
                return serializer;
        }

        // 遍历扩展的SerializerFactory。尝试由扩展的SerializerFactory提供Serializer
        // 2.5.3没有这个功能
        for (int i = 0; serializer == null && _factories != null && i < _factories.size();  i++) {
            AbstractSerializerFactory factory;
            factory = (AbstractSerializerFactory) _factories.get(i);
            serializer = factory.getSerializer(cl);
        }

        if (serializer != null) {
        }

        else if (JavaSerializer.getWriteReplace(cl) != null)
            serializer = new JavaSerializer(cl, _loader);
        else if (HessianRemoteObject.class.isAssignableFrom(cl))
            serializer = new RemoteSerializer();
        //    else if (BurlapRemoteObject.class.isAssignableFrom(cl))
        //      serializer = new RemoteSerializer();
        else if (Map.class.isAssignableFrom(cl)) {
            if (_mapSerializer == null)
                _mapSerializer = new MapSerializer();
            serializer = _mapSerializer;
        }
        else if (Collection.class.isAssignableFrom(cl)) {
            if (_collectionSerializer == null) {
                _collectionSerializer = new CollectionSerializer();
            }
            serializer = _collectionSerializer;
        }
        else if (cl.isArray())
            serializer = new ArraySerializer();
        else if (Throwable.class.isAssignableFrom(cl))
            serializer = new ThrowableSerializer(cl, getClassLoader());
        else if (InputStream.class.isAssignableFrom(cl))
            serializer = new InputStreamSerializer();
        else if (Iterator.class.isAssignableFrom(cl))
            serializer = IteratorSerializer.create();

        else if (Enumeration.class.isAssignableFrom(cl))
            serializer = EnumerationSerializer.create();
        else if (Calendar.class.isAssignableFrom(cl))
            serializer = CalendarSerializer.create();
        else if (Locale.class.isAssignableFrom(cl))
            serializer = LocaleSerializer.create();
        else if (Enum.class.isAssignableFrom(cl))
            serializer = new EnumSerializer(cl);
        if (serializer == null)
            serializer = getDefaultSerializer(cl);
        if (_cachedSerializerMap == null)
            _cachedSerializerMap = new HashMap(8);
        
        synchronized (_cachedSerializerMap) {
            _cachedSerializerMap.put(cl, serializer);
        }
        return serializer;
    }
}
```



### 5.4 SPI扩展Serialization

这个方法理论上是完全可行的，但是有些过于大动干戈了。总体思路是：借助Dubbo的SPI机制，对Serialization进行扩展，基于hessian2的SerializerFactory ，对Serializer、Deserializer进行扩展。



## 6 写在最后

最开始计划用一两天的时间写完这篇博客，但是过程中又不断的遇到问题、解决问题，最终花了整个五一假期才完成。通过这个问题，算是对Dubbo框架的序列化这块有了一些了解了。


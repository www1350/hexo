---
title: dubbo源码解析（三）serialize
date: 2018-04-03 22:44:07
tags:
categories:
---

serialize层

dubbo的序列化方式包括：hessian2、fastjson、fst、jdk、kryo

只要通过配置`<dubbo:protocol serialization="">` 或`<dubbo:service serialization="">` 就可以实现

<!-- more -->

```java
@SPI("hessian2")
public interface Serialization {

    /**
     * get content type id
     *
     * @return content type id
     */
    byte getContentTypeId();

    /**
     * get content type
     *
     * @return content type
     */
    String getContentType();

    /**
     * create serializer
     *
     * @param url
     * @param output
     * @return serializer
     * @throws IOException
     */
    @Adaptive
    ObjectOutput serialize(URL url, OutputStream output) throws IOException;

    /**
     * create deserializer
     *
     * @param url
     * @param input
     * @return deserializer
     * @throws IOException
     */
    @Adaptive
    ObjectInput deserialize(URL url, InputStream input) throws IOException;

}
```



Hessian2Serialization

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
        return new Hessian2ObjectOutput(out);
    }

    public ObjectInput deserialize(URL url, InputStream is) throws IOException {
        return new Hessian2ObjectInput(is);
    }

}
```

![image](https://user-images.githubusercontent.com/7789698/38181129-bb58f0fe-3663-11e8-8e76-98520d072598.png)



KryoSerialization

```java
public class KryoSerialization implements Serialization {

    public byte getContentTypeId() {
        return 8;
    }

    public String getContentType() {
        return "x-application/kryo";
    }

    public ObjectOutput serialize(URL url, OutputStream out) throws IOException {
        return new KryoObjectOutput(out);
    }

    public ObjectInput deserialize(URL url, InputStream is) throws IOException {
        return new KryoObjectInput(is);
    }
}
```

![image](https://user-images.githubusercontent.com/7789698/38181141-cad287d4-3663-11e8-9dc7-bf0312b3b890.png)



FastJsonSerialization

```java
public class FastJsonSerialization implements Serialization {

    public byte getContentTypeId() {
        return 6;
    }

    public String getContentType() {
        return "text/json";
    }

    public ObjectOutput serialize(URL url, OutputStream output) throws IOException {
        return new FastJsonObjectOutput(output);
    }

    public ObjectInput deserialize(URL url, InputStream input) throws IOException {
        return new FastJsonObjectInput(input);
    }

}
```

![mage-20180402105157](/var/folders/cf/lq_f9wkn3gx_l9nghhvyt7240000gn/T/abnerworks.Typora/image-201804021051571.png)



FstSerialization

```java
public class FstSerialization implements Serialization {

    public byte getContentTypeId() {
        return 9;
    }

    public String getContentType() {
        return "x-application/fst";
    }

    public ObjectOutput serialize(URL url, OutputStream out) throws IOException {
        return new FstObjectOutput(out);
    }

    public ObjectInput deserialize(URL url, InputStream is) throws IOException {
        return new FstObjectInput(is);
    }
}
```

![mage-20180402105229](/var/folders/cf/lq_f9wkn3gx_l9nghhvyt7240000gn/T/abnerworks.Typora/image-201804021052292.png)



![image](https://user-images.githubusercontent.com/7789698/38181199-29213bc8-3664-11e8-8c64-2bc6e6a37a2c.png)

从上面我们可以看到最重要的接口就是ObjectInput、ObjectOutput、DataInput、DataOutput

![image](https://user-images.githubusercontent.com/7789698/38181371-222c4d16-3665-11e8-978d-a78198e123a4.png)

![image](https://user-images.githubusercontent.com/7789698/38181395-33b176d8-3665-11e8-81da-45d35ba4595c.png)

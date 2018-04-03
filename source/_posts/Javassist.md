---
title: Javassist
abbrlink: 2e60fe2e
date: 2018-04-03 22:40:46
tags:
categories:
---

java字节码被存储在一个叫做类文件的二进制文件。CtClass的对象代表一个类文件。ClassPool是存放CtClass的hash列表，用类名做key，如果CtClass没发现get()会读取一个class建造一个新的类记录到hash表并返回。


```
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("test.Rectangle");
cc.setSuperclass(pool.get("test.Point"));
byte[] b = cc.toBytecode();
Class aClass = cc.toClass();
cc.writeFile();
```


如果系统使用多个类装载器，getDefault()只能搜索当前jvm的路径，可能加载不到对象。

```
pool.insertClassPath(new ClassClassPath(this.getClass()));


ClassPool pool = ClassPool.getDefault();
pool.insertClassPath("/usr/local/javalib");
```

甚至你可以使用：
```
ClassPool pool = ClassPool.getDefault();
ClassPath cp = new URLClassPath("www.javassist.org", 80, "/java/", "org.javassist.");
pool.insertClassPath(cp);
```


```
ClassPool cp = ClassPool.getDefault();
byte[] b = a byte array;
String name = class name;
cp.insertClassPath(new ByteArrayClassPath(name, b));
CtClass cc = cp.get(name);


ClassPool cp = ClassPool.getDefault();
InputStream ins = an input stream for reading a class file;
CtClass cc = cp.makeClass(ins);
```


一个新类可以被定义为一个现有的类的一个副本

```
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("Point");
cc.setName("Pair");
```



```

public class Hello {
    public void say() {
        System.out.println("Hello");
    }
}

public class Test {
    public static void main(String[] args) throws Exception {
        ClassPool cp = ClassPool.getDefault();
        CtClass cc = cp.get("Hello");
        CtMethod m = cc.getDeclaredMethod("say");
        m.insertBefore("{ System.out.println(\"Hello.say():\"); }");
        Class c = cc.toClass();
        Hello h = (Hello)c.newInstance();
        h.say();
    }
}

```


关于内省
http://jboss-javassist.github.io/javassist/tutorial/tutorial2.html#intro





$0, $1, $2, ... | 参数
$args | 数组参数Object[]
$$ | 所有参数m($$) 相当于m($1,$2,...)
$cflow(...) | cflow variable
$r | 结果类型.
$w | The wrapper type. It is used in a cast expression.
$_ | 结果值
$sig | An array of java.lang.Class objects representing the formal parameter types.
$type | A java.lang.Class object representing the formal result type.
$class | A java.lang.Class object representing the class currently edited.




#### $0, $1, $2, ...

```
class Point {
    int x, y;
    void move(int dx, int dy) { x += dx; y += dy; }
}
```

```
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("Point");
CtMethod m = cc.getDeclaredMethod("move");
m.insertBefore("{ System.out.println($1); System.out.println($2); }");
cc.writeFile();
```


结果》
```
class Point {
    int x, y;
    void move(int dx, int dy) {
        { System.out.println(dx); System.out.println(dy); }
        x += dx; y += dy;
    }
}
```


#### $cflow



#### $w
Integer i = ($w)5;
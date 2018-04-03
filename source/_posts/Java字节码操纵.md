---
title: Java字节码操纵
abbrlink: bab06bbc
date: 2018-04-03 22:42:04
tags:
categories:
---

# 什么是字节码？
- 机器码 机器码(machine code)是CPU可直接解读的指令。机器码与硬件等有关，不同的CPU架构支持的硬件码也不相同。

- 字节码 字节码（bytecode）是一种包含执行程序、由一序列 op 代码/数据对 组成的二进制文件。字节码是一种中间码，它比机器码更抽象，需要直译器转译后才能成为机器码的中间代码。通常情况下它是已经经过编译，但与特定机器码无关。字节码主要为了实现特定软件运行和软件环境、与硬件环境无关。 字节码的实现方式是通过编译器和虚拟机器。编译器将源码编译成字节码，特定平台上的虚拟机器将字节码转译为可以直接执行的指令。 而在Java里，通过类加载器把字节码读入加载并转换成 java.lang.Class类的一个实例。

## 类文件结构
一个编译后的类文件包含下面的结构：

```
ClassFile {
    u4            magic;
    u2            minor_version;
    u2            major_version;
    u2            constant_pool_count;
    cp_info        contant_pool[constant_pool_count – 1];
    u2            access_flags;
    u2            this_class;
    u2            super_class;
    u2            interfaces_count;
    u2            interfaces[interfaces_count];
    u2            fields_count;
    field_info        fields[fields_count];
    u2            methods_count;
    method_info        methods[methods_count];
    u2            attributes_count;
    attribute_info    attributes[attributes_count];
}
```

| 字符                                | 描述                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| magic, minor_version, major_version | 类文件的版本信息和用于编译这个类的 JDK 版本。                |
| constant_pool                       | 类似于符号表，尽管它包含更多数据。下面有更多的详细描述。     |
| access_flags                        | 提供这个类的描述符列表。                                     |
| this_class                          | 提供这个类全名的常量池(constant_pool)索引，比如org/jamesdbloom/foo/Bar。 |
| super_class                         | 提供这个类的父类符号引用的常量池索引。                       |
| interfaces                          | 指向常量池的索引数组，提供那些被实现的接口的符号引用。       |
| fields                              | 提供每个字段完整描述的常量池索引数组。                       |
| methods                             | 指向constant_pool的索引数组，用于表示每个方法签名的完整描述。如果这个方法不是抽象方法也不是 native 方法，那么就会显示这个函数的字节码。 |
| attributes                          | 不同值的数组，表示这个类的附加信息，包括 RetentionPolicy.CLASS 和 RetentionPolicy.RUNTIME 注解。 |




举个简单的例子（例1.1）：

```
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello World!");
    }
}
```

对应的字节码如下：

```
public class com.souche.rick.HelloWorld
  minor version: 0/*Java 类文件的版本信息*/
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:/*该项存放了类中各种文字字符串、类名、方法名和接口名称、final 变量以及对外部类的引用信息等常量。虚拟机必须为每一个被装载的类维护一个常量池，常量池中存储了相应类型所用到的所有类型、字段和方法的符号引用。*/
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // Hello World!
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // com/souche/rick/HelloWorld
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable/*为调试器提供源码中的每一行对应的字节码信息*/
  #11 = Utf8               LocalVariableTable /*列出了所有栈帧中的局部变量。这里唯一的局部变量就是 this。*/
  #12 = Utf8               this
  #13 = Utf8               Lcom/souche/rick/HelloWorld;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               HelloWorld.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Class              #28            // java/lang/System
  #22 = NameAndType        #29:#30        // out:Ljava/io/PrintStream;
  #23 = Utf8               Hello World!
  #24 = Class              #31            // java/io/PrintStream
  #25 = NameAndType        #32:#33        // println:(Ljava/lang/String;)V
  #26 = Utf8               com/souche/rick/HelloWorld
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
{
  public com.souche.rick.HelloWorld();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/souche/rick/HelloWorld;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream; /*获取指定类的静态域，并将其值压入栈顶*/
         3: ldc           #3                  // String Hello World! /*将"Hello World!"从常量池中推送至栈顶*/
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V /*调用实例方法println*/
         8: return /*从当前方法返回void*/
      LineNumberTable:
        line 9: 0
        line 10: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
}
```

注：编译的时候选用-g参数，否则LineNumberTable参数默认是不显示的，maven编译的时候默认是使用-g参数所以不用担心

描述符标识字符含义

| 标识字符 | 含义           | 标识字符 | 含义                          |
| -------- | -------------- | -------- | ----------------------------- |
| B        | 基本类型byte   | J        | 基本类型long                  |
| C        | 基本类型char   | S        | 基本类型short                 |
| D        | 基本类型double | Z        | 基本类型boolean               |
| F        | 基本类型float  | V        | 特殊类型void                  |
| I        | 基本类型int    | L        | 对象类型，如Ljava/lang/Object |

对于数组类型，每一唯独使用一个前置的“[”字符描述，如一个定义为“java.lang.String[][]”类型的二维数组，将被记录为“[[Ljava/lang/String”。

当描述符描述方法时，按照先参数列表后返回值的顺序描述，参数列表按照参数的严格顺序放在一组小括号“()”之内。如int getResult()方法的描述符为“()I”。

| Method declaration in source file | Method descriptor       |
| --------------------------------- | ----------------------- |
| void m(int i, float f)            | (IF)V                   |
| int m(Object o)                   | (Ljava/lang/Object;)I   |
| int[] m(int i, String s)          | (ILjava/lang/String;)[I |
| Object m(int[] i)                 | ([I)Ljava/lang/Object;  |

关于助记符https://www.cnblogs.com/anbylau2130/p/6078427.html


简单了解了一些基本概念以后，我们正式进入主题：字节码操纵

# 原理
1.  用一些字节码操纵框架修改字节码
2.  自定义ClassLoader来加载修改后的字节码

其实还有另外一种形式，就是直接换掉原来的字节码。一种是在JVM加载用户的Class时，拦截，返回修改后的字节码。另外一种在运行时，使用Instrumentation.redefineClasses方法来替换掉原来的字节码，和这个类相关的实例立即生效。不过这种形式需要采用agent形式启动。有兴趣的同学可以自行搜索下相关内容。



# 操纵字节码的场景
操纵字节码可以在字节码被载入类加载器之前修改字节码二进制文件，从而做到一些技术上难以实现的功能，如：AOP增强、性能分析、调试跟踪、日志记录、bug定位、混淆代码，甚至连Scala、Groovy和Grails等JVM语言都用到大量的操纵字节码技术。

# 字节码操纵框架
- [ASM](http://asm.ow2.org/)
- [javassist](http://jboss-javassist.github.io/javassist/)
- [aspectj](http://www.eclipse.org/aspectj/)
- [CGLIB](http://cglib.sourceforge.net)
- [BCEL](http://commons.apache.org/proper/commons-bcel/)
- [bytebuddy](http://bytebuddy.net/#/)
  ...

## ASM使用
ASM主要通过树这种数据结构来表示复杂的字节码结构，通过Visitor设计模式来实现(详细见[手册](http://download.forge.objectweb.org/asm/asm4-guide.pdf))
![image](https://user-images.githubusercontent.com/7789698/34321023-551223f4-e840-11e7-927c-0bebfd0ab7d9.png)

![image](https://user-images.githubusercontent.com/7789698/34321032-850f2368-e840-11e7-8911-5bb921dc8f38.png)

![image](https://user-images.githubusercontent.com/7789698/34321085-81d334ae-e841-11e7-99cb-22a2c2577a8e.png)

![image](https://user-images.githubusercontent.com/7789698/34321268-c2f28d8c-e845-11e7-8e14-ee5f21d50b8b.png)

![image](https://user-images.githubusercontent.com/7789698/34321269-c9c6c8ee-e845-11e7-9e49-86636a5a21e6.png)

1. 生成一个类
  上面的例1.1代码可以用ASM生成：

```
package com.souche.rick.asm;

import org.objectweb.asm.ClassWriter;
import org.objectweb.asm.Label;
import org.objectweb.asm.MethodVisitor;
import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import static org.objectweb.asm.Opcodes.*;

public class HelloWorld {
    public static void main(String[] args) throws IOException {
        byte[] bytes = builder();
        File file=new File("com"+File.separator+"souche"+File.separator+"rick", "HelloWorld.class");
        file.getParentFile().mkdirs();
        FileOutputStream fileOutputStream = new FileOutputStream(file);
        fileOutputStream.write(bytes);
        fileOutputStream.close();

    }

    public static byte[] builder(){
        ClassWriter cw = new ClassWriter(0);
        cw.visit(V1_8, ACC_PUBLIC  + ACC_SUPER,
                "com/souche/rick/HelloWorld", null, "java/lang/Object",
                null);
        MethodVisitor classInitMv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
        classInitMv.visitCode();
        Label l0 = new Label();
        classInitMv.visitLabel(l0);
        classInitMv.visitLineNumber(9, l0);
        classInitMv.visitVarInsn(ALOAD,0);
        classInitMv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
        classInitMv.visitInsn(RETURN);
        Label l1 = new Label();
        classInitMv.visitLabel(l1);
        classInitMv.visitLocalVariable("this", "Lcom/souche/rick/HelloWorld;", null, l0, l1, 0);
        classInitMv.visitMaxs(1, 1);
        classInitMv.visitEnd();

        MethodVisitor mainMv = cw.visitMethod(ACC_PUBLIC + ACC_STATIC, "main",
                "([Ljava/lang/String;)V", null, null);
        mainMv.visitCode();
        Label l2 = new Label();
        mainMv.visitLabel(l2);
        mainMv.visitLineNumber(13, l2);
        mainMv.visitFieldInsn(GETSTATIC, "java/lang/System", "out",
                "Ljava/io/PrintStream;");
        mainMv.visitLdcInsn("Hello World!");

        mainMv.visitMethodInsn(INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
        Label l3 = new Label();
        mainMv.visitLabel(l3);
        mainMv.visitLineNumber(15,l3);
        mainMv.visitInsn(RETURN);
        mainMv.visitLocalVariable("args", "[Ljava/lang/String;", null, l2, l3, 0);
        mainMv.visitMaxs(2, 1);
        mainMv.visitEnd();
        cw.visitEnd();
        byte[] b = cw.toByteArray();
        return b;
    }
}

```

效果：
![image](https://user-images.githubusercontent.com/7789698/34320105-7694ad8a-e82c-11e7-9ba1-f2b84bcb5f20.png)

![image](https://user-images.githubusercontent.com/7789698/34320113-ac22dcd8-e82c-11e7-8d9a-9435630696eb.png)


```
public class com.souche.rick.HelloWorld
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Utf8               com/souche/rick/HelloWorld
   #2 = Class              #1             // com/souche/rick/HelloWorld
   #3 = Utf8               java/lang/Object
   #4 = Class              #3             // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = NameAndType        #5:#6          // "<init>":()V
   #8 = Methodref          #4.#7          // java/lang/Object."<init>":()V
   #9 = Utf8               this
  #10 = Utf8               Lcom/souche/rick/HelloWorld;
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               java/lang/System
  #14 = Class              #13            // java/lang/System
  #15 = Utf8               out
  #16 = Utf8               Ljava/io/PrintStream;
  #17 = NameAndType        #15:#16        // out:Ljava/io/PrintStream;
  #18 = Fieldref           #14.#17        // java/lang/System.out:Ljava/io/PrintStream;
  #19 = Utf8               Hello World!
  #20 = String             #19            // Hello World!
  #21 = Utf8               java/io/PrintStream
  #22 = Class              #21            // java/io/PrintStream
  #23 = Utf8               println
  #24 = Utf8               (Ljava/lang/String;)V
  #25 = NameAndType        #23:#24        // println:(Ljava/lang/String;)V
  #26 = Methodref          #22.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #27 = Utf8               args
  #28 = Utf8               [Ljava/lang/String;
  #29 = Utf8               Code
  #30 = Utf8               LocalVariableTable
  #31 = Utf8               LineNumberTable
{
  public com.souche.rick.HelloWorld();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #8                  // Method java/lang/Object."<init>":()V
         4: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/souche/rick/HelloWorld;
      LineNumberTable:
        line 9: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #18                 // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #20                 // String Hello World!
         5: invokevirtual #26                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       8     0  args   [Ljava/lang/String;
      LineNumberTable:
        line 13: 0
        line 15: 8
}
```

2. 解析一个类
  接下来是官方的一个例子，用来打印出这个类

```
public class ClassPrinter extends ClassVisitor {
    public ClassPrinter() {
        super(ASM5);
    }

    @Override
    public void visit(int version, int access, String name,
                      String signature, String superName, String[] interfaces) {
        System.out.println(name + " extends " + superName + " {");
    }

    @Override
    public void visitSource(String source, String debug) {
    }

    @Override
    public void visitOuterClass(String owner, String name, String desc) {
    }

    @Override
    public AnnotationVisitor visitAnnotation(String desc,
                                             boolean visible) {
        return null;
    }
    @Override
    public void visitAttribute(Attribute attr) {
    }


    @Override
    public void visitInnerClass(String name, String outerName,
                                String innerName, int access) {
    }


    @Override
    public FieldVisitor visitField(int access, String name, String desc,
                                   String signature, Object value) {
        System.out.println(" " + desc + " " + name);
        return null;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name,
                                     String desc, String signature, String[] exceptions) {
        System.out.println(" " + name + desc);
        return null;
    }

    @Override
    public void visitEnd() {
        System.out.println("}");
    }

    public static void main(String[] args) throws IOException {
        ClassPrinter cp = new ClassPrinter();
        ClassReader cr = new ClassReader("java.lang.Runnable");
        cr.accept(cp, 0);
    }
}
```

![image](https://user-images.githubusercontent.com/7789698/34321112-862b8820-e842-11e7-82cb-bc846dac0b3a.png)

3.转化一个类

spring里面有一个类叫`LocalVariableTableParameterNameDiscoverer`，它可以用来获取参数名列表，其实内部就是使用了ASM，

```
	private static class ParameterNameDiscoveringVisitor extends ClassVisitor {

		private static final String STATIC_CLASS_INIT = "<clinit>";/*实例初始化方法（init）。类实例化（clinit）*/

		private final Class<?> clazz;

		private final Map<Member, String[]> memberMap;

		public ParameterNameDiscoveringVisitor(Class<?> clazz, Map<Member, String[]> memberMap) {
			super(SpringAsmInfo.ASM_VERSION);
			this.clazz = clazz;
			this.memberMap = memberMap;
		}

		@Override
		public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
			// 排除 synthetic + bridged && 静态类实例化
			if (!isSyntheticOrBridged(access) && !STATIC_CLASS_INIT.equals(name)) {
//desc 指的是Method descriptors（方法描述符）
				return new LocalVariableTableVisitor(clazz, memberMap, name, desc, isStatic(access));
			}
			return null;
		}
 /* ACC_SYNTHETIC说明这个方法是由编译器生成，并且不会在源代码中出现。ACC_BRIDGE说明是编译生成的桥接方法*/
		private static boolean isSyntheticOrBridged(int access) {
			return (((access & Opcodes.ACC_SYNTHETIC) | (access & Opcodes.ACC_BRIDGE)) > 0);
		}

		private static boolean isStatic(int access) {
			return ((access & Opcodes.ACC_STATIC) > 0);
		}
	}
```



```
private static class LocalVariableTableVisitor extends MethodVisitor {

		private static final String CONSTRUCTOR = "<init>";

		private final Class<?> clazz;

		private final Map<Member, String[]> memberMap;

		private final String name;

		private final Type[] args;

		private final String[] parameterNames;

		private final boolean isStatic;

		private boolean hasLvtInfo = false;

		/*
		 * The nth entry contains the slot index of the LVT table entry holding the
		 * argument name for the nth parameter.
		 */
		private final int[] lvtSlotIndex;

		public LocalVariableTableVisitor(Class<?> clazz, Map<Member, String[]> map, String name, String desc, boolean isStatic) {
			super(SpringAsmInfo.ASM_VERSION);
			this.clazz = clazz;
			this.memberMap = map;
			this.name = name;
//方法描述符转化为参数
			this.args = Type.getArgumentTypes(desc);
			this.parameterNames = new String[this.args.length];
			this.isStatic = isStatic;
			this.lvtSlotIndex = computeLvtSlotIndices(isStatic, this.args);
		}

		@Override
		public void visitLocalVariable(String name, String description, String signature, Label start, Label end, int index) {
			this.hasLvtInfo = true;
			for (int i = 0; i < this.lvtSlotIndex.length; i++) {
				if (this.lvtSlotIndex[i] == index) {
					this.parameterNames[i] = name;
				}
			}
		}

		@Override
		public void visitEnd() {
			if (this.hasLvtInfo || (this.isStatic && this.parameterNames.length == 0)) {
				//存储下来
				this.memberMap.put(resolveMember(), this.parameterNames);
			}
		}

		private Member resolveMember() {
...
		}

		private static int[] computeLvtSlotIndices(boolean isStatic, Type[] paramTypes) {
			int[] lvtIndex = new int[paramTypes.length];
//实例方法，前面第0个位置还有个this
			int nextIndex = (isStatic ? 0 : 1);
			for (int i = 0; i < paramTypes.length; i++) {
				lvtIndex[i] = nextIndex;
//如果是long和double需要2个连续的局部变量表来保存
				if (isWideType(paramTypes[i])) {
					nextIndex += 2;
				}
				else {
					nextIndex++;
				}
			}
			return lvtIndex;
		}
	}
```

注：一个局部变量表的占用了32位的存储空间（一个存储单位称之为slot，槽），所以可以存储一个boolean、byte、char、short、float、int、refrence和returnAdress数据，long和double需要2个连续的局部变量表来保存，通过较小位置的索引来获取。如果被调用的是实例方法，那么第0个位置存储“this”关键字代表当前实例对象的引用。

里面重点关注visitLocalVariable方法，其他的忽略，是不是就觉得简单多了？


## Javassist使用
Javassist和其他的类似库不同的是，Javassist并不要求开发者对字节码方面具有多么深入的了解，同样的，它也允许开发者忽略被修改的类本身的细节和结构。相对其他如ASM显得更为简单，当然性能也较之更低下。

- ClassPool  跟踪和控制所操作的类，支持JVM 搜索路径中装载的、自定义路径、字节数组、流中装载二进制类、从头开始创建新类
- CtClass  装载到类池中的类
- CtField 字段
- CtMethod 方法
- CtConstructor 构造函数

### ClassPool
1. `ClassPool.getDefault()`   只是搜索JVM的同路径下的class
2. `new ClassPool(true)`  当ClassPool没被引用的时候，JVM的垃圾收集会收集该类
3. 如果系统使用多个类装载器，getDefault()只能搜索当前jvm的路径，可能加载不到对象
```
    ClassPool#insertClassPath(ClassPath)
      ClassPool#appendClassPath(ClassPath)
      ClassPool#removeClassPath(ClassPath)
```
![image](https://user-images.githubusercontent.com/7789698/34321619-559a67e2-e84e-11e7-87ca-0c744f9d2f50.png)
4.  `CtClass ct  = mPool.get(name)` 通过类池获取类  `CtClass ct  = mPool.makeClass(mClassName)`创建一个类

### CtClass
1. getName获取类名 、getSimpleName获取简要类名、getSuperclass获取父类、getInterfaces获取接口、cc.getMethods获取方法
2. `CtMethod m = cc.getDeclaredMethod("say")` 获取方法 
3. `Class c = cc.toClass()` 转化为Class  `byte[] b = cc.toBytecode()` 转化为字节码二进制


还是那个例子，不过Javassist是不是简单多了

```
public class HelloWorld {
    public static void main(String[] args) throws IOException, CannotCompileException {
        byte[] bytes = builder();
        File file=new File("com"+File.separator+"souche"+File.separator+"rick", "HelloWorld.class");
        file.getParentFile().mkdirs();
        FileOutputStream fileOutputStream = new FileOutputStream(file);
        fileOutputStream.write(bytes);
        fileOutputStream.close();
    }

    public static byte[] builder() throws IOException, CannotCompileException {
        ClassPool classPool = ClassPool.getDefault();
        CtClass ctClass = classPool.makeClass("com.souche.rick.HelloWorld");
        CtMethod ctMethod =  CtNewMethod.make(
                "public static void main(String[] args) { System.out.println(\"Hello World!\"); }",
                ctClass);
        ctClass.addMethod(ctMethod);
        return ctClass.toBytecode();
    }
}
```

### 内省
参数 | 含义 
-- | -- 
$0, $1, $2, ...  |  $0为this ，其他为参数
$args  |  数组参数Object[]
$$ |  所有参数m($$) 相当于m($1,$2,...) （除了this）
$cflow(...) | cflow 变量
$r | 结果类型
$w | 包装类型
$_ | 结果值 
$sig |  java.lang.Class数组对象，用来表示每个参数的类型
 $type | java.lang.Class 对象表示结果类型
$class | A java.lang.Class 表示现在这个类的类型

例子：

```
class Point {
    int x, y;
    void move(int dx, int dy) { x += dx; y += dy; }
}
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("Point");
CtMethod m = cc.getDeclaredMethod("move");
m.insertBefore("{ System.out.println($1); System.out.println($2); }");
cc.writeFile();
```

结果

```
class Point {
    int x, y;
    void move(int dx, int dy) {
        { System.out.println(dx); System.out.println(dy); }
        x += dx; y += dy;
    }
}

```


dubbo的ReferenceBean(dubbo调用方)里面创建代理就运用了Javassist技术来获取接口生成代理类:

![ttd](https://user-images.githubusercontent.com/7789698/34324263-2ae91d14-e8a7-11e7-8289-6db88d0047ac.jpg)
只看部分关键代码：

```
ClassGenerator ccp = null, ccm = null;
		try
		{
			ccp = ClassGenerator.newInstance(cl);/*通过传入ClassLoader来使用ClassPool获取封装的ClassGenerator（里面封装了Javassist操作的几个对象）*/

			Set<String> worked = new HashSet<String>();
			List<Method> methods = new ArrayList<Method>();

			for(int i=0;i<ics.length;i++)
			{
				if( !Modifier.isPublic(ics[i].getModifiers()) )
				{
					String npkg = ics[i].getPackage().getName();
					if( pkg == null )
					{
						pkg = npkg;
					}
					else
					{
						if( !pkg.equals(npkg)  )
							throw new IllegalArgumentException("non-public interfaces from different packages");
					}
				}
				ccp.addInterface(ics[i]);

				for( Method method : ics[i].getMethods() )
				{
					String desc = ReflectUtils.getDesc(method);
					if( worked.contains(desc) )
						continue;
					worked.add(desc);

					int ix = methods.size();
					Class<?> rt = method.getReturnType();
					Class<?>[] pts = method.getParameterTypes();

					StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
					for(int j=0;j<pts.length;j++)
						code.append(" args[").append(j).append("] = ($w)$").append(j+1).append(";");
					code.append(" Object ret = handler.invoke(this, methods[" + ix + "], args);");
					if( !Void.TYPE.equals(rt) )
						code.append(" return ").append(asArgument(rt, "ret")).append(";");

					methods.add(method);
					ccp.addMethod(method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
				}
			}

			if( pkg == null )
				pkg = PACKAGE_NAME;

			// create ProxyInstance class.
			String pcn = pkg + ".proxy" + id;/*类名为  pkg + ".proxy" +id = 包名 + “.poxy” +自增数值*/
			ccp.setClassName(pcn);
			ccp.addField("public static java.lang.reflect.Method[] methods;");
			ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
			ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{ InvocationHandler.class }, new Class<?>[0], "handler=$1;");
            ccp.addDefaultConstructor();
			Class<?> clazz = ccp.toClass();
			clazz.getField("methods").set(null, methods.toArray(new Method[0]));

			// create Proxy class.
			String fcn = Proxy.class.getName() + id;/*对象名“Proxy”+id*/
			ccm = ClassGenerator.newInstance(cl);
			ccm.setClassName(fcn);
			ccm.addDefaultConstructor();/*设置默认构造方法*/
			ccm.setSuperClass(Proxy.class);/*继承于Proxy*/
/*添加方法public Object newInstance(InvocationHandler h){ return new " + 包名 + “.poxy” +自增数值 + "($1);}*/
			ccm.addMethod("public Object newInstance(" + InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
			Class<?> pc = ccm.toClass();
			proxy = (Proxy)pc.newInstance();
```

如果有一个接口,被dubbo:reference引用，假设包名是com.souche.rick

```
public interface AuthService {
    User register(UserRegisterParam userToAdd);
    String login(String username, String password);
}
```

1.委托类

```
public class com.souche.rick.proxy1 implements AuthService{
/*这里的methods通过反射依次被放入register、login*/
      public static java.lang.reflect.Method[] methods ;

      private InvocationHandler handler;

      public com.souche.rick.proxy1(InvocationHandler handler){
         this.handler = handler;
      }

      public User register(UserRegisterParam userToAdd){
         Object[] args = new Object[1];
         args[0] = (UserRegisterParam)userToAdd;
         Object ret = handler.invoke(this, methods[0], args);
         return (User)ret;
      }

      public  String login(String username, String password){
         Object[] args = new Object[2];
         args[0] = (String)username;
         args[0] = (String)password;
         Object ret = handler.invoke(this, methods[1], args);
         return (String)ret;
      }
}
```

2. 生成一个继承于Proxy的代理子类，

```
public class Proxy + id extends Proxy{
	public Object newInstance(InvocationHandler h){ 
               return new  com.souche.rick.proxy1(h);
        }
}
```

此外还有非常多的工具使用了字节码操纵技术，比如fastjson(ASM)、hibernate(cglib、javaassist)、zorka(ASM)、Btrace(ASM)




参考：
https://www.cnblogs.com/qiumingcheng/p/5400265.html

https://www.ibm.com/developerworks/cn/java/j-lo-classloader/index.html

http://www.importnew.com/17770.html
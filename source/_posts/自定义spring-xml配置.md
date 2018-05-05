---
title: 自定义spring xml配置
tags: spring
abbrlink: a362c3af
date: 2018-05-03 14:28:19
categories:
---

spring万恶的xml配置虽然恶心，但是也不乏良好的设计。如何自定义xml呢？

## 创建XML Schema文件

### 什么是XML Schema

- XML Schema 描述了 XML文档的结构

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<xsd:schema xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            xmlns:beans="http://www.springframework.org/schema/beans"
            xmlns:tool="http://www.springframework.org/schema/tool"
            xmlns="http://code.alibabatech.com/schema/dubbo"
            targetNamespace="http://code.alibabatech.com/schema/dubbo">

    <xsd:import namespace="http://www.w3.org/XML/1998/namespace"/>
    <xsd:import namespace="http://www.springframework.org/schema/beans"/>
    <xsd:import namespace="http://www.springframework.org/schema/tool"/>

    <xsd:complexType name="annotationType">
        <xsd:attribute name="id" type="xsd:ID">
            <xsd:annotation>
                <xsd:documentation><![CDATA[ The unique identifier for a bean. ]]></xsd:documentation>
            </xsd:annotation>
        </xsd:attribute>
        <xsd:attribute name="package" type="xsd:string" use="optional">
            <xsd:annotation>
                <xsd:documentation><![CDATA[ The scan package. ]]></xsd:documentation>
            </xsd:annotation>
        </xsd:attribute>
    </xsd:complexType>

    <xsd:element name="annotation" type="annotationType">
        <xsd:annotation>
            <xsd:documentation><![CDATA[ The annotation config ]]></xsd:documentation>
        </xsd:annotation>
    </xsd:element>

</xsd:schema>
```

#### 简易元素

**语法**

```xml
<xsd:element name="xxx" type="yyy"/>
```

这里的xsd:element 被称为简易元素。name是名称，type是类型。XML Schema 拥有很多内建的数据类型。

**实例**

这是一些 XML 元素：

```xml
<lastname>Refsnes</lastname>
<age>36</age>
<dateborn>1970-03-27</dateborn>
```

这是相应的简易元素定义：

```xml
<xsd:element name="lastname" type="xsd:string"/>
<xsd:element name="age" type="xsd:integer"/>
<xsd:element name="dateborn" type="xsd:date"/>
```

#### XSD 属性

简易元素无法拥有属性。假如某个元素拥有属性，它就会被当作某种复合类型。但是属性本身总是作为简易类型被声明的。

**语法**

```xml
<xsd:attribute name="xxx" type="yyy"/>
```

**实例**

这是一些 XML 元素：

```xml
<lastname>Refsnes</lastname>
<age>36</age>
<dateborn>1970-03-27</dateborn>
```

这是相应的简易元素定义：

```xml
<xsd:element name="lastname" type="xsd:string"/>
<xsd:element name="age" type="xsd:integer"/>
<xsd:element name="dateborn" type="xsd:date"/>
```

#### 默认值

当没有其他的值被规定时，默认值就会自动分配给元素。

在下面的例子中，缺省值是 "red"：

```xml
<xsd:element name="color" type="xsd:string" default="red"/>
```

#### 固定值

固定值同样会自动分配给元素，并且您无法规定另外一个值。

在下面的例子中，固定值是 "red"：

```xml
<xsd:element name="color" type="xsd:string" fixed="red"/>
```

#### 可选的和必需的属性

在缺省的情况下，属性是可选的。如需规定属性为必选，请使用 "use" 属性：

```xml
<xsd:attribute name="lang" type="xsd:string" use="required"/>
```

#### XSD 限定 / Facets

限定（restriction）用于为 XML 元素或者属性定义可接受的值。对 XML 元素的限定被称为 facet。

##### 对值的限定（minInclusive／maxInclusive）

下面的例子定义了带有一个限定且名为 "age" 的元素。age 的值不能低于 0 或者高于 120：

```xml
<xsd:element name="age">
<xsd:simpleType>
  <xsd:restriction base="xsd:integer">
    <xsd:minInclusive value="0"/>
    <xsd:maxInclusive value="120"/>
   </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

##### 对一组值的限定（enumeration）

如需把 XML 元素的内容限制为一组可接受的值，我们要使用枚举约束（enumeration constraint）。

下面的例子定义了带有一个限定的名为 "car" 的元素。可接受的值只有：Audi, Golf, BMW：

```xml
<xsd:element name="car">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:enumeration value="Audi"/>
      <xsd:enumeration value="Golf"/>
      <xsd:enumeration value="BMW"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

上面的例子也可以被写为：

```xml
<xsd:element name="car" type="carType"/>
<xsd:simpleType name="carType">
  <xsd:restriction base="xsd:string">
    <xsd:enumeration value="Audi"/>
    <xsd:enumeration value="Golf"/>
    <xsd:enumeration value="BMW"/>
  </xsd:restriction>
</xsd:simpleType>
```

**注意：** 在这种情况下，类型 "carType" 可被其他元素使用，因为它不是 "car" 元素的组成部分。

##### 对一系列值的限定（pattern）

如需把 XML 元素的内容限制定义为一系列可使用的数字或字母，我们要使用模式约束（pattern constraint）。

下面的例子定义了带有一个限定的名为 "letter" 的元素。可接受的值只有小写字母 a - z 其中的一个：

```xml
<xsd:element name="letter">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:pattern value="[a-z]"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

下一个例子定义了带有一个限定的名为 "initials" 的元素。可接受的值是大写字母 A - Z 其中的三个：

```xml
<xsd:element name="initials">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:pattern value="[A-Z][A-Z][A-Z]"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

下一个例子也定义了带有一个限定的名为 "initials" 的元素。可接受的值是大写或小写字母 a - z 其中的三个：

```xml
<xsd:element name="initials">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:pattern value="[a-zA-Z][a-zA-Z][a-zA-Z]"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

下一个例子定义了带有一个限定的名为 "choice 的元素。可接受的值是字母 x, y 或 z 中的一个：

```xml
<xsd:element name="choice">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:pattern value="[xyz]"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

下一个例子定义了带有一个限定的名为 "prodid" 的元素。可接受的值是五个阿拉伯数字的一个序列，且每个数字的范围是 0-9：

```xml
<xsd:element name="prodid">
  <xsd:simpleType>
    <xsd:restriction base="xsd:integer">
      <xsd:pattern value="[0-9][0-9][0-9][0-9][0-9]"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

##### 对一系列值的其他限定

下面的例子定义了带有一个限定的名为 "letter" 的元素。可接受的值是 a - z 中零个或多个字母：

```xml
<xsd:element name="letter">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:pattern value="([a-z])*"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

下面的例子定义了带有一个限定的名为 "letter" 的元素。可接受的值是一对或多对字母，每对字母由一个小写字母后跟一个大写字母组成。举个例子，"sToP"将会通过这种模式的验证，但是 "Stop"、"STOP" 或者 "stop" 无法通过验证：

```xml
<xsd:element name="letter">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:pattern value="([a-z][A-Z])+"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

下面的例子定义了带有一个限定的名为 "gender" 的元素。可接受的值是 male 或者 female：

```xml
<xsd:element name="gender">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:pattern value="male|female"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

下面的例子定义了带有一个限定的名为 "password" 的元素。可接受的值是由 8 个字符组成的一行字符，这些字符必须是大写或小写字母 a - z 亦或数字 0 - 9：

```xml
<xsd:element name="password">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:pattern value="[a-zA-Z0-9]{8}"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

##### 对空白字符的限定（whiteSpace）

如需规定对空白字符（whitespace characters）的处理方式，我们需要使用 whiteSpace 限定。

下面的例子定义了带有一个限定的名为 "address" 的元素。这个 whiteSpace 限定被设置为 "preserve"，这意味着 XML 处理器不会移除任何空白字符：

```xml
<xsd:element name="address">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:whiteSpace value="preserve"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

这个例子也定义了带有一个限定的名为 "address" 的元素。这个 whiteSpace 限定被设置为 "replace"，这意味着 XML 处理器将移除所有空白字符（换行、回车、空格以及制表符）：

```xml
<xsd:element name="address">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:whiteSpace value="replace"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

这个例子也定义了带有一个限定的名为 "address" 的元素。这个 whiteSpace 限定被设置为 "collapse"，这意味着 XML 处理器将移除所有空白字符（换行、回车、空格以及制表符会被替换为空格，开头和结尾的空格会被移除，而多个连续的空格会被缩减为一个单一的空格）：

```xml
<xsd:element name="address">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:whiteSpace value="collapse"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

##### 对长度的限定（length／minLength／maxLength）

如需限制元素中值的长度，我们需要使用 length、maxLength 以及 minLength 限定。

本例定义了带有一个限定且名为 "password" 的元素。其值必须精确到 8 个字符：

```xml
<xsd:element name="password">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:length value="8"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

这个例子也定义了带有一个限定的名为 "password" 的元素。其值最小为 5 个字符，最大为 8 个字符：

```xml
<xsd:element name="password">
  <xsd:simpleType>
    <xsd:restriction base="xsd:string">
      <xsd:minLength value="5"/>
      <xsd:maxLength value="8"/>
    </xsd:restriction>
  </xsd:simpleType>
</xsd:element>
```

##### 数据类型的限定

| 限定           | 描述                                                      |
| -------------- | --------------------------------------------------------- |
| enumeration    | 定义可接受值的一个列表                                    |
| fractionDigits | 定义所允许的最大的小数位数。必须大于等于0。               |
| length         | 定义所允许的字符或者列表项目的精确数目。必须大于或等于0。 |
| maxExclusive   | 定义数值的上限。所允许的值必须小于此值。                  |
| maxInclusive   | 定义数值的上限。所允许的值必须小于或等于此值。            |
| maxLength      | 定义所允许的字符或者列表项目的最大数目。必须大于或等于0。 |
| minExclusive   | 定义数值的下限。所允许的值必需大于此值。                  |
| minInclusive   | 定义数值的下限。所允许的值必需大于或等于此值。            |
| minLength      | 定义所允许的字符或者列表项目的最小数目。必须大于或等于0。 |
| pattern        | 定义可接受的字符的精确序列。                              |
| totalDigits    | 定义所允许的阿拉伯数字的精确位数。必须大于0。             |
| whiteSpace     | 定义空白字符（换行、回车、空格以及制表符）的处理方式。    |

#### 复合元素（complexType）

复合元素指包含其他元素及/或属性的 XML 元素。

- 有四种类型的复合元素：

  - 空元素

    ```xml
    <product pid="1345"/>
    ```

    ```xml
    <xsd:element name="product" type="prodtype"/>

    <xsd:complexType name="prodtype">
      <xsd:attribute name="pid" type="xsd:positiveInteger"/>
    </xsd:complexType>
    ```

  - 包含其他元素的元素

    ```xml
    <employee>
      <firstname>John</firstname>
      <lastname>Smith</lastname>
    </employee>
    ```

    ```xml
    <xsd:element name="employee" type="employeetype"/>

    <xsd:complexType name="employeetype">
      <xsd:sequence>
        <xsd:element name="firstname" type="xsd:string"/>
        <xsd:element name="lastname" type="xsd:string"/>
      </xsd:sequence>
    </xsd:complexType>
    ```

  - 仅包含文本的元素

    ```xml
    <food type="dessert">Ice cream</food>
    ```

    ```xml
    <xsd:element name="food" type="foodtype"/>

    <xsd:complexType name="foodtype">
      <xsd:simpleContent>
        <xsd:extension base="xs:string">
          <xsd:attribute name="dessert" type="xsd:string" />
        </xsd:extension>
      </xsd:simpleContent>
    </xsd:complexType>
    ```

  - 包含元素和文本的元素

    ```xml
    <description>
    It happened on <date>03.03.99</date> 
    </description>
    ```

    ```xml
    <xsd:element name="description" type="descriptiontype"/>

    <xsd:complexType name="descriptiontype" mixed="true">
      <xsd:sequence>
        <xsd:element name="date" type="xs:date"/>
      </xsd:sequence>
    </xsd:complexType>
    ```


#### 指示器

有七种指示器：

Order 指示器：

- All 规定子元素可以按照任意顺序出现，且每个子元素必须只出现一次
- Choice 指示器规定可出现某个子元素或者可出现另外一个子元素（非此即彼）
- Sequence 规定子元素必须按照特定的顺序出现

Occurrence 指示器：

- maxOccurs 指示器可规定某个元素可出现的最大次数
- minOccurs 指示器可规定某个元素能够出现的最小次数

Group 指示器：

- Group name 定义相关的数批元素
- attributeGroup name

#### <any> 元素

<any> 元素使我们有能力通过未被 schema 规定的元素来拓展 XML 文档

```xml
<xsd:element name="person">
  <xsd:complexType>
    <xsd:sequence>
      <xsd:element name="firstname" type="xsd:string"/>
      <xsd:element name="lastname" type="xsd:string"/>
      <xsd:any minOccurs="0"/>
    </xsd:sequence>
  </xsd:complexType>
</xsd:element>
```

假如有另一个xsd定义了chidren节点，我们就可以通过any把children加入person节点之下

```xml
<person>
  <firstname>Hege</firstname>
  <lastname>Refsnes</lastname>
  <children>
    <childname>Cecilie</childname>
  </children>
</person>

<person>
  <firstname>Stale</firstname>
  <lastname>Refsnes</lastname>
</person>
```

####  <anyAttribute> 元素

<anyAttribute> 元素使我们有能力通过未被 schema 规定的属性来扩展 XML 文档


## 构造Bean

## META-INF下增加spring.schemas

如：http\://code.alibabatech.com/schema/dubbo/dubbo.xsd=META-INF/dubbo.xsd

## 继承NamespaceHandlerSupport实现处理器

![image](https://user-images.githubusercontent.com/7789698/39589196-18f58e0e-4f30-11e8-8506-2748fd00c55b.png)

继承NamespaceHandlerSupport后需要实现init方法

如：

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }

}
```

这样实现了：

```
* dubbo:application->ApplicationConfig
* dubbo:module->ModuleConfig
* dubbo:registry->RegistryConfig
* dubbo:monitor->MonitorConfig
* dubbo:provider->ProviderConfig
* dubbo:consumer->ConsumerConfig
* dubbo:protocol->ProtocolConfig
* dubbo:service->ServiceBean
* dubbo:reference->ReferenceBean
* dubbo:annotation ->使用继承了AbstractSingleBeanDefinitionParser的解析器AnnotationBeanDefinitionParser
```

## 实现BeanDefinitionParser接口实现多个解析器

```java
public interface BeanDefinitionParser {

   BeanDefinition parse(Element element, ParserContext parserContext);

}
```

BeanDefinitionParser只有一个接口



```java
public class LightningXCacheBeanParser implements BeanDefinitionParser {

    private final Class<?> beanClass;

    public LightningXCacheBeanParser(Class<?> beanClass) {
        this.beanClass = beanClass;
    }

    @Override
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        RootBeanDefinition beanDefinition = new RootBeanDefinition();
        beanDefinition.setBeanClass(beanClass);
        beanDefinition.setLazyInit(false);
        String application = element.getAttribute("application");
        beanDefinition.getPropertyValues().addPropertyValue("application", application);

        //注册到spring容器
        parserContext.getRegistry().registerBeanDefinition(beanClass.getName(), beanDefinition);
        return beanDefinition;
    }
}
```

## META-INF下增加spring.handlers

如：http\://code.alibabatech.com/schema/dubbo=com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler



参考：

http://www.runoob.com/schema/schema-tutorial.html
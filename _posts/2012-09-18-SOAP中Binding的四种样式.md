---
title: SOAP中Binding的四种样式
date: 2012-09-18 22:10
tags:
- JAVA
comments: true
---

#SOAP中Binding的四种样式

在SOAP中由于在当初标准化过程比较短,并且采用了事实标准推动.导致了现在WSDL1.1中其实是有4种绑定的样式的.这四种样式生成的WSDL都有细微的差别.而了解它们之间的区别,对于我们生成或调用WebService是非常有帮助的.否则就有可能出现别人生成的WSDL,我们动态调用不了,又不晓得原因的情况.

这四种样式分别是:

1. RPC/Encoded
2. RPC/Literal
3. Document/Encoded
4. Document/Literal(Wrapper)

这四组样式其实可以分成`Style`和`Use`两个属性的排列组合.

* `RPC Style`指定包含Web服务调用的XML节点，该节点以Web服务调用方法命名，XML节点依次包含方法调用的各个参数。
* `Document Style`指定内直接包含消息，该消息没有SOAP格式限制。服务器的应用层负责将XML文档映射成内存对象（参数、方法调用等等）
* `Encoded Use`表示XML的消息使用类型属性引用抽象数据类型，使用Section 5编码（SOAP规范第五章定义的编码）进行xml的序列化和反序列化。
* `Literal Use`表示XML的消息使用类型属性或者Element元素引用具体的Schema定义，也就是说，根据具体的Schema将内存对象序列化成XML消息。

<!--more-->
##RPC/Encoded
RPC/Encoded遵循`远程服务调用`模式的样式,在这种模式中客户端发送一个同步请求给服务器来执行一次操作.SOAP请求中包含了此次要执行的方法的名称和参数.整个SOAP体中包含了方法的发送参数与返回值,因此这种样式具有相当的耦合关系.
比如如下的WSDL:

```xml
<message name="myMethodRequest">
    <part name="x" type="xsd:int"/>
</message>
<message name="empty"/>
<portType name="PT">
    <operation name="myMethod">
        <input message="myMethodRequest"/>
        <output message="empty"/>
    </operation>
</portType>
<binding name="myMethod" type="s0:myMethodRpcEncodedSoap">
    <soap:binding style="rpc" transport="http://schemas.xmlsoap.org/soap/http" />
    <operation name="myMethod">
        <soap:operation soapAction="" />
        <input>
            <soap:body use="encoded" namespace="" />
        </input>
        <output>
            <soap:body use="encoded" />
        </output>
    </operation>
</binding>
```

生成的SOAP请求为:

```xml
<soap:envelope>
    <soap:body>
        <myMethod>
            <x xsi:type="xsd:int">5</x>
        </myMethod>
    </soap:body>
</soap:envelope>
```

可以看到WSDL很简单,基本达到了尽可能简单易懂的要求,并且生成的SOAP也很明确,就是要调用`myMehtod`方法,入参是`x` 类型是`int` 值是5.
但是,在WS-I1.1规范中已经废除了这种样式的支持.这是因为在`RPC/Encoded`中,SOAP编码定义了一系列的编码规则,方便SOAP数据模型到XML的映射,然而解析这些编码规则大大的降低了系统的吞吐量.同时,也不能简单的校验此消息的有效性.

##RPC/Literal
`RPC/Literal`是`RPC/Encoded`的一个改进,两者长的非常的类似.唯一的区别就是把`<soap:body>`元素中的`use`由`encoded`改为了`literal`.

```xml
<message name="myMethodRequest">
    <part name="x" type="xsd:int"/>
</message>
<message name="empty"/>
<portType name="PT">
    <operation name="myMethod">
        <input message="myMethodRequest"/>
        <output message="empty"/>
    </operation>
</portType>
<binding name="myMethod" type="s0:myMethodRpcLiteralSoap">
    <soap:binding style="rpc" transport="http://schemas.xmlsoap.org/soap/http" />
    <operation name="myMethod">
        <soap:operation soapAction="" />
        <input>
            <soap:body use="literal" namespace="" />
        </input>
        <output>
            <soap:body use="literal" />
        </output>
    </operation>
</binding>
```
生成的SOAP请求为:

```xml
<soap:envelope>
    <soap:body>
        <myMethod>
            <x>5</x>
        </myMethod>
    </soap:body>
</soap:envelope>
```

可以看到生成的SOAP请求仍然是包含了方法名的,但是入参的类型被抹去了.这样就减少了解析量,增加了吞吐.
但是他同样不能简单的校验此消息的有效性.WSDL中包含的有效信息太少.独立的XML Schema 不会告诉你消息体中的信息集合包含了些什么，你也必需知道RPC规则.

##Document/Encoded
这种方式其实已经被淘汰了,你无法生成出这种类型的WSDL.因此就让它消失在历史的长流中吧.

##Document/Literal(Wrapper)
这种样式其实又分为两种,一种是不带Wrapper的,另一种是带Wrapper的.先说不带Wrapper的.

`Document`的方式与`RPC`的不同,`RPC`是定义了`SOAP`交互双方之间的远程方法调用接口,抽象的是交互的过程.并且把抽象的SOAP数据类型,根据不同的`use`方式转成成`SOAP`消息,从而实现远程过程调用的.而`Document`客户端与服务端的耦合性更低,它抽象的是交互的对象.它把一段符合`WSDL`约束的文本数据进行交互.

```xml
<types>
    <schema>
        <element name="xElement" type="xsd:int"/>
    </schema>
</types>
<message name="myMethodRequest">
    <part name="x" 
        element="xElement"/>
</message>
<message name="empty"/>
<portType name="PT">
    <operation name="myMethod">
        <input message="myMethodRequest"/>
        <output message="empty"/>
    </operation>
</portType>
<binding .../>  
```

它生成的SOAP协议为:

```xml
<soap:envelope>
    <soap:body>
        <xElement>5</xElement>
    </soap:body>
</soap:envelope>
```

可以看出生成的WSDL比较复杂,描述了相当多的内容.通过这些信息,就可以验证消息的有效性.但是它生成的SOAP请求中丢失了方法名,因为他本是抽象的就不是交互的过程.这就对我们WS的调用造成了麻烦.需要手动的解析和猜测调用的方法.有时甚至变得不可调用.

为了解决这个问题,后来又出了一种`Document/Literal(Wrapper)`的样式,它通过在`WSDL`的`element`和`part`部分补充`complexType`的方式增加了调用方法的名称.这样就弥补了`Document/Literal`存在调用麻烦的问题.

```xml
<types>
    <schema>
        
        <element name="myMethod"/>
            <complexType>
                <sequence>
                    <element name="x" type="xsd:int"/>
                </sequence>
            </complexType>
        </element>
    </schema>
</types>
<message name="myMethodRequest">
    <part name="
        parameters" element="
        myMethod"/>
</message>
<message name="empty"/>
<portType name="PT">
    <operation name="myMethod">
        <input message="myMethodRequest"/>
        <output message="empty"/>
    </operation>
</portType>
<binding .../>  
```

它生成的SOAP请求为:

```xml
<soap:envelope>
    <soap:body>
        <myMethod>
            <x>5</x>
        </myMethod>
    </soap:body>
</soap:envelope>
```

可能有人会认为 这个SOAP和`RPC/Literal`生成的SOAP是一样的.确实是一样的,但是他们的WSDL是不相同的,`Document/Literal(Wrapper)`的WSDL中存在大量的信息可以用于校验数据的正确性.并且`Document/Literal(Wrapper)`中`soap:body`中的`myMethod`其实表示的是单个输入消息的组成部分引用的元素的名称,而不是`RPC/Literal`中方法名称的意思.

##总结
其实现在在通常的情况下都是使用`Document/Literal(Wrapper)`,它基本上包含了其他几种方法的所有优点.我们项目发布的WS也都是这样的样式.不过其他的几种方式在特定的环境下还是有它的用武之地,因此我们也需要了解.一旦出现了无法解析的情况,要清楚的明白问题出现在哪儿.

---
参考链接:

* [https://www.ibm.com/developerworks/cn/webservices/ws-whichwsdl/][1]
* [http://www.iteye.com/topic/145061][2]


  [1]: https://www.ibm.com/developerworks/cn/webservices/ws-whichwsdl/
  [2]: http://www.iteye.com/topic/145061
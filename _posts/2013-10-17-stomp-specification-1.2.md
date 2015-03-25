---
layout: post
title: "STOMP协议规范"
description: "STOMP1.2协议规范"
comments: true
category: ""
tags: [STOMP, MQ]
---
{% include JB/setup %}
原文: [STOMP Protocol Specification, Version 1.2](http://stomp.github.io/stomp-specification-1.2.html)

## 摘要
STOMP是一个简单的可互操作的协议, 被用于通过中间服务器在客户端之间进行异步消息传递。它定义了一种在客户端与服务端进行消息传递的文本格式.  

STOMP已经被使用了很多年，并且支持很多消息`brokers`和客户端库。这个规范定义STOMP `1.2`协议以及对`1.1`版本的更新。

发送反馈到`stomp-spec@googlegroups.com`.

## 概述

### 背景
由于需要用脚本语言如`Ruby`, `Python`, `Perl`去连接企业级的消息`brokers`, STOMP产生了.在这种情况下，STMOP实现了一些简单的操作，比如可靠地发送单一的消息，然后断开或者从目的地消费所有消息。

STOMP是除`AMQP`开放消息协议之外地另外一个选择, 实现了被用在JMS brokers中特定的有线协议，比如`OpenWire`.
它仅仅是实现通用消息操作中的一部分，并非想要覆盖全面的消息API.

STOMP目前已经是个成熟的协议，在`wire-level`方面, 它提供了一些简单的用例，但仍保持其核心设计原则:简单性和互操作性。
### 协议概述

STOMP是基于frame的协议, 与HTTP的frame相似.一个`frame`包含一个`command`，一系列可选的`headers`和`body`.STOMP虽然是基于消息但同于也允许传递二进制消息。STMOP的默认消息格式是`UTF-8`,但是在消息体中同样支持其他格式编码。  

STOMP服务器就好像是一系列的目的地, 消息会被发送到这里。STOMP协议把目的地当作不透明的字符串，其语法是服务端具体的实现。
此外STOMP没有定义目的地的交付语义是什么。
交付，或“消息交换”，语义的目的地可以从服务器到服务器，甚至从目的地到目的地。这使得服务器有可创造性的语义，去支持STOMP。

STOMP client的用户代理可以充当两个角色(可能同时)：
* 作为生产者，通过`SEND`frame发送消息到server
* 作为消费者，发送`SUBSCRIBE`frame到目的地并且通过`MESSAGE`frame从server获取消息。

### STOMP版本之间的变化

STOMP 1.2 大部分向后兼容1.1. 有两点不兼容的改变:
* 用回车加换行符代替只用换行符结束frame
* 简化了消息应答，用专用的header

除此之外，STOMP 1.2并没有增加新特性，而是阐述规格中的一些模糊概念，比如:
* 重复的frame header条目
* `content-length`和`content-type`headers的用法 
* 必须支持servers STOMP frame
* 连接延迟
* 作用域，订阅的唯一，事务的标示符
* `RECEIPT`frame的含义

###  设计哲学

简易性，互通性是STOMP主要设计哲学.  
STOMP被设计成为轻量级的协议，它很容易用其他语言在client和server实现。这就意味着servers的架构没有太多的约束，以及没有太多的特性比如目的地命名空间，可靠的语法需要去实现。  

在这份规格书里面，注意，我们没有明确定义的STOMP 1.2 servers特性。你应该查阅STMOMP servers 文档去获得这些特性的详细描述。

## 一致性

[RFC 2119](http://tools.ietf.org/html/rfc2119)中详细地解释了`MUST`, `MUST NOT`, `REQUIRED`, `SHALL`, `SHALL NOT`, `SHOULD`, `SHOULD NOT`, `RECOMMENDED`, `MAY`, 和 `OPTIONAL`这些关键字  
为了阻止来自服务端地攻击，保护内存溢出，消除平台限制，限制了不受约束的输入。

规格中一致性的级别适用于STOMP clients and STOMP servers.

## STOMP Frames

STOMP是基于帧的协议，它假定底层为一个2-way的可靠流的网络协议（如TCP）。客户端和服务器通信使用STOMP帧流通讯。帧的结构看起来像：

    COMMAND
    header1:value1
    header2:value2
    
    Body^@

帧以command字符串开始,以EOL结束，其中包括可选回车符（13字节），紧接着是换行符（10字节）。command下面是0个或多个`<key>:<value>`格式的header条目, 每个条目由EOL结束。一个空白行（即额外EOL）表示header结束和body开始。body连接着`NULL`字节。本文档中的例子将使用`^@`,在ASCII中用control-@表示，代表NULL字节。NULL字节可以选择跟多个`EOLs`。欲了解更多关于STOMP帧的详细信息，请参阅[Augmented BNF](#Augmented_BNF)节本文件。

本文档中引用的所有command 和header 名字都是大小写敏感的.

### 编码方式

commands和headers 都是用UTF-8编码的.在用UTF-8编码的headers中除了`CONNECT`和`CONNECTED`帧以外，任何的回车符，换行符，colon found(?)都将被转义.

转义的目的在于允许header中的键值包含那些把`octets`当作值的frame header.

为了向后兼容STOMP 1.0, `CONNECT`和`CONNECTED`不会转义回车符，换行符，colon found(?)

C风格的字符串转义被用在UTF-8编码的headers中去转义回车符，换行符以及colon found.当解码headers时，必须使用下列转换:
* \r (octet 92 and 114) translates to carriage return (octet 13)
* \n (octet 92 and 110) translates to line feed (octet 10)
* \c (octet 92 and 99) translates to : (octet 58)
* \\\ (octet 92 and 92) translates to \ (octet 92)

未定义转义序列如`\t`(octet 92 and 116)必须被视为一个致命的错误。相反，当编码帧头，必须使用逆转变.

The STOMP 1.0 specification included many example frames with padding in the headers and many servers and clients were implemented to trim or pad header values. This causes problems if applications want to send headers that SHOULD not get trimmed. In STOMP 1.2, clients and servers MUST never trim or pad headers with spaces.

### Body
只有`SEND`, `MESSAGE`, 和`ERROR`帧有body。所有其他的帧`不能`有body。

### 标准header

大多数被用的header都有特殊的含义。

#### Header content-length

所有的帧可能都包括有`content-length`的header。它定义了消息体的大小。如果header包含了`content-length`,  包含空字节的消息体的最大字节数不能超过这个数. 帧仍然需要以空字节结束。  

如帧体存在，`SEND`, `MESSAGE` 和 `ERROR`帧应该包含`content-length`.如果帧体包含空字节，那么这个帧必须包括`content-length`.

#### Header content-type

如果帧体存在，`SEND`, `MESSAGE` 和 `ERROR`帧应该包含`content-type`帮助接受者去理解帧体.如果设置了`content-type`, 它的值必须是描述帧体格式的`MINE`类型.否则，接收者应该认为帧体格式为二进制Blob.

以`text/`开头的MINE类型的默认文本编码是`UTF-8`. 如果你正在用一个基于MINE类型的不同编码, 你应该添加`;charset=<encoding>`MINE类型。例如：如果你发送一个UTF-16编码的HTML body， 应该设置`text/html;charset=utf-16`. `;charset=<encoding>`也能添加到任何非`text/` MINE类型后去作为说明。UTF-8编码的XML是个很好的例子。它的编码被设置为`application/xml;charset=utf-8`.

所有STOMP客户端和服务端必须支持`UTF-8`编码和解码。因此，为了最大限度地使用在异构环境中的互操作性，建议基于文本的内容使用UTF-8编码.

#### Header receipt

任何除了`CONNECT`的客户端帧可以为`receipt` header指定任何值。这会让服务端应答带有`RECEIPT`的客户端帧的处理过程。

#### Repeated Header Entries
Since messaging systems can be organized in store and forward topologies, similar to SMTP, a message may traverse several messaging servers before reaching a consumer. A STOMP server MAY 'update' header values by either prepending headers to the message or modifying a header in-place in the message.

如果client或者server受到重复的header条目，只有第一个会被用作header条目的值。其他的值仅仅用来维持状态改变，或者被丢弃。

例如，如果client收到:

    MESSAGE
    foo:World
    foo:Hello

    ^@

`foo` header的值为`World`.

#### 大小限制

为了客户端滥用服务端的内存分配，服务端可以设置可分配的内存大小:

* 单个帧允许帧头的个数  
* header中每一行的最大长度  
* 帧体的大小  

如果超出了这些限制，server应该向client发送一个`error` frame,然后关闭连接.

#### 连接延迟

STOMP servers必须支持client快速地连接server和断开连接。
这意味着server在连接重置前只允许被关闭的连接短时间地延迟.

结果就是，在socket重置前client可能不会收到server发来的最后一个frame(比如`ERROR`或者`RECEIPT`frame去应答`DISCONNECT`frame)

## Connecting

STOMP client通过`CONNECT`frame与server建立流或者TCP连接.

    CONNECT
    accept-version:1.2
    host:stomp.github.org

    ^@

如果server收到请求，将返回`CONNECTED`frame:

    CONNECTED
    version:1.2

    ^@

server能拒绝所有的连接请求。server应该响应`ERROR`frame去说明为什么连接被拒绝然后关闭连接。

## CONNECT or STOMP Frame

STOMP servers 处理`STOMP`frame必须和处理`CONNECT`frame一样。STOMP`1.2` clients应该继续使用`CONNECT`command去向后兼容`1.0`.

使用`STOMP`frame的clients只能连接上STOMP`1.2` servers(以及一些STOMP1.1 servers),但是好处在于协议探针能够从HTTP连接中区分开STOMP连接。

STOMP 1.2 clients `必须`设置以下headers:
* `accept-version`: clients支持的STOMP的版本号。详情见[Protocol_Negotiation](#Protocol_Negotiation)
* `host`:client希望连接的虚拟主机名字，建议设置已经连接的socket为主机名，或者任何名字。如果headers没有匹配到任何可用的虚拟主机，支持虚拟主机的servers将选择默认的虚拟主机或者拒绝连接。

STOMP 1.2 clients`可选择`设置以下headers:
* `login`: 用于在server验证的用户id
* `passcode`: 用于在server验证的密码
* `heart-beat`: [心跳](#Heart-beating)设置

### CONNECTED Frame
STOMP 1.2 servers `必须`设置以下headers:

* `version`: 会话中STOMP版本。详情见[Protocol_Negotiation](#Protocol_Negotiation)

STOMP 1.2 servers`可选择`设置以下headers:

* heart-beat: [心跳](#Heart-beating)设置
* session: 唯一的会话identifier 
* server: 描述STOMP server信息。它必须包含`server-name`,可以包含一些注释信息（用空格分开）
  `server-name`后面也可以带着可选的版本号.   

        server = name ["/" version] *(comment)

   例如:

        server:Apache/1.3.9

### Protocol Negotiation

STOMP1.1 以后的版本，`CONNECT`frame必须包括`accept-version`header.它的值为clients支持的STOMP版本号，多个版本号用`,`隔开。如果不存在`accept-version`header,那么表明clients只支持1.0.

在一次会话中将使用双方都支持的最高版本。

例如，如果client发送:

    CONNECT
    accept-version:1.0,1.1,2.0
    host:stomp.github.org

    ^@

server将返回与客户端同时支持的最高版本。

    CONNECTED
    version:1.1

    ^@

如果client和server不支持共同的协议版本,server必须返回如下的`ERROR`frame,然后断开连接。

    ERROR
    version:1.2,2.1
    content-type:text/plain
    
    Supported protocol versions are 1.2 2.1^@

### 心跳

心跳被用于去测试底层TCP连接的可用性，确保远端服务处于活动状态。

要使用心跳，每个部分必须声明它能干什么以及想要其他部分干什么. 通过在`CONNECT`和`CONNECTED`frame中增加`heart-beat`header, 让心跳在会话开始被定义好。
`heart-beat`header必须包含两个用逗号隔开的正整数。

第一个数字代表发送方能做什么:
* `0`表示它不能发送心跳
* 否则它是能保证两次心跳的最小毫秒数

第二个数字代表发送方能获得什么:
* `0`表示它不想接收心跳
* 否则它表示两次心跳期望的毫秒数

`heart-beat`header是`OPTIONAL`的。没有的话会被当作`heart-beat:0,0`header 处理，意思就是说它不会发送心跳并且不想接收心跳。

`heart-beat`header提供了足够的信息去了解每个部分心跳是否可用，发送到哪里，频率的大小.

原始frame像这个样子:

    CONNECT
    heart-beat:<cx>,<cy>
   
    CONNECTED:
    heart-beat:<sx>,<sy>

对于client发送server的心跳:
* 如果`<cx>`为0(client不能发送心跳)或者`<sy>`为0(server不想接收心跳),将不起任何作用。
* 否则心跳频率为`MAX(<cx>,<sy>)`毫秒数.

相反，`<sx>`和`<cy>`同样是这样的.

关于心跳本身，通过网络连接收到的任何数据表明远端服务是可用的。在给定的指向，如果心跳的频率被期望是`<n>`毫秒:

* 发送者必须每`<n>`毫秒发送新数据。
* 如果发送者没有真实的STOMP frame，必须发送一个`end-of-line (EOL)`
* 如果接受者在规定的时间内没有收到新数据，表明连接已经断开
* 由于时间误差，接收者应该容错和考虑定义错误的界限

## Client Frames
client可以发送下列列表以外的frame，但是STOMP1.2 server会响应`ERROR`frame,然后关闭连接。

* [SEND](#SEND)
* [SUBSCRIBE](#SUBSCRIBE)
* [UNSUBSCRIBE](#UNSUBSCRIBE)
* [BEGIN](#BEGIN)
* [COMMIT](#COMMIT)
* [ABORT](#ABORT)
* [ACK](#ACK)
* [NACK](#NACK)
* [DISCONNECT](#DISCONNECT)

### SEND

`SEND`frame发送消息到目的地，它必须包含表示目的地地址的`destination`header.	`SEND`frame body是被发送的消息。例如:

    SEND
    destination:/queue/a
    content-type:text/plain

    hello queue a
    ^@

这个消息被发送到`/queue/a`.注意STOMP把目的地看作为一个不透明的字符串，没有目的地假设的交互语义.你应该查阅STOMP server文档，搞清楚如何构造目的地名字。

可靠的消息语义是server指定的，依赖备用的目的地的值和其他消息headers，比如事务headers，或者其他server指定的消息headers。

`SEND`可以添加`transaction`header来支持事务处理.

如果body存在，那么`SEND`frame应该包含一个`content-length`和`content-type`header

一个应用可以给`SEND`frame增加任意多个用户定义的headers。
通常用于用户定义的头，让消费者能够根据应用程序定义的报头使用选择订阅帧过滤消息。
被定义的用户必须通过`MESSAGE`frame传送。

如果server不能无故成功处理`SEND`frame,那么server必须向client发送`ERROR`frame然后关闭连接。

### SUBSCRIBE

`SUBSCRIBE`frame用于注册给定的目的地.和`SEND`frame一样，`SUBSCRIBE`frame需要包含`destination`header表明client想要订阅目的地。
被订阅的目的地收到的任何消息将通过`MESSAGE`frame发送给client。
`ack`header控制着确认模式。

例子:

    SUBSCRIBE
    id:0
    destination:/queue/foo
    ack:client

    ^@

如果server不能成功创建此次订阅，那么server将返回`ERROR`frame然后关闭连接。

STOMP服务器可能支持额外的服务器特定的头文件，来自定义有关订阅传递语义.

### SUBSCRIBE id Header
一个单连接可以对应多个开放的servers订阅,所以必须包含`id`header去唯一标示这个订阅.这个`id`frame可以把此次订阅与接下来的`MESSAGE`frame和`UNSUBSCRIBE`frame联系起来。

在相同的连接中，不同的订阅必须拥有不同订阅id。

### SUBSCRIBE ack Header

`ack`header可用的值有`auto`, `client`,`client-individual`, 默认为`auto`.

当`ack`为`auto`时，client收到server发来的消息后不需要回复`ACK`frame.server假定消息发出去后client就已经收到。这种确认方式可以减少消息传输的次数.

当`ack`为`client`时, client必须发送`ACk`frame给servers, 让它处理消息.如果在client发送`ACK`frame之前连接断开了，那么server将假设消息没有被处理，可能会再次发送消息给另外的客户端。client发送的`ACK`frame被当作时积累的确认。这就意味这种确认方式会去操作`ACK`frame指定的消息和订阅的所有消息

由于client不能处理某些消息，所以client应该发送`NACK`frame去告诉server它不能消费这些消息。

当ack模式是`client-individual`，确认工作就像客户端确认模式(除了由客户端发送的`ACK`或`NACK`帧)不会被累计。这意味着，后续`ACK`, `NACK`消息帧，也不能影响前面的消息的确认。

### UNSUBSCRIBE

`UNSUBSCRIBE`frame被用于去移除已经存在订阅。一旦订阅被删除后，STOMP连接将不再会收到来自订阅发出的消息。

一个单连接可以对应多个开放的server订阅,所以必须包含`id`header去唯一标示被删除的订阅.这个header中的id必须匹配已存在订阅.

例如:

    UNSUBSCRIBE
    id:0

    ^@

### ACK

`ACK`用`client`和`client-individual`去确认订阅消息的消费.只有通过`ACK`确认过后，订阅的消息才算是被消费.

`ACK`frame必须包含一个`id`header去匹配将要被确认的`ack`header中的id.可以选择地指定`transaction`header表明消息确认应该是命名事务地一部分。

    ACK
    id:12345
    transaction:tx1

    ^@

### NACK

`NACK`有`ACK`相反地作用。它地作用是告诉server client不想消费这个消息。server然后发送这个消息给另外的client,丢弃它或者把它放在无效的消息队列中。这种准确的行为是server特定的。

`NACK`有相同的`ACK`headers: `id`(必选)和`transaction`(可选)。

`NACK`适用于单个消息(订阅的`ack`模式为`client-individual`), 或者那些还没有被`ACK'ed`和`NACK'ed`的消息(订阅模式ack为`client`).

### BEGIN

`BEGIN`用于事务的开始。事务被用于发送和确认消息，被发送和被确认的消息在事务过程中会被自动处理。

    BEGIN
    transaction:tx1

    ^@
	
`transaction`header是必填的，并且事务id将被用于在SEND, COMMIT, ABORT, ACK, and NACK frames去绑定命名的事务.在相同的连接中，不同事务`必须`用不同的id

如果client发送`DISCONNECT`frame或者TCP连接失败，任何已开始但没有提交的事务默认都会被中断.

### COMMIT

`COMMIT`用于在过程中提交事务.

    COMMIT
    transaction:tx1

    ^@

`transaction`header是必填的并且`必须`指定将要提交的事务的id.

### ABORT

`ABORT`用于在过程中回滚事务.

    ABORT
    transaction:tx1
    
    ^@

`transaction`header是必填的并且`必须`指定将要提交的事务的id.

### DISCONNECT

client能在任何时候断开server的连接,但是不能保证已经发送的frame已经到达了server。为了让这一切显得不那么暴力，client确保所有已经发送的frames已经被server收到，client应该做以下3点:

1. 发送带有`receipt`header的`DISCONNECT`frame

        DISCONNECT
        receipt:77
        ^@

2. 等待带有`RECEIPT`frame的响应

        RECEIPT
        receipt-id:77
        ^@
3. 关闭socket

注意，如果server过早地关闭socket，client将不会收到期望地`RECEIPT`frame.见[Connection_Lingering](#Connection_Lingering)

client发送`DISCONNECT`frame后不必要在发送任何frame.

### Server Frames

server偶尔也会发送frame给客户端(除了连接最初的`CONNECTED`frame).

这些frames为:
* [MESSAGE](#MESSAGE)
* [RECEIPT](#RECEIPT)
* [ERROR](#ERROR)

#### MESSAGE

`MESSAGE`frame用于将订阅的消息发送给client.

`MESSAGE`frame必须包含`destination`header表明信息要到达的目的地。如果消息已经用STOMP发送，那么`destination`header应该和`SEND`frame中的一样。

`MESSAGE`frame`必须`包含带有唯一标识的`message-id`header和带有将接收消息的订阅的id`subscription`header.

如果从订阅收到的消息需要明确作出确认(`client`或者`client-individual`模式),那么`MESSAGE`frame`必须`包含带有任何值的`ack`header.这个header将把消息和后来的`ACK`,`NACK`frame关联起来。

下面这个frame body包含了消息的内容:

    MESSAGE
    subscription:0
    message-id:007
    destination:/queue/a
    content-type:text/plain
    
    hello queue a^@

如果frame body包含内容的话，`MESSAGE`frame应该包含`content-length`header和`content-type`header.

除了那些server指定的headers, 消息被发送到目的地时，`MESSAGE`frame同样应该包括所有用户定义的headers.查阅有关文档，找出那些server指定添加到messages的headers.

#### RECEIPT

server成功处理请求带有`receipt`的client frame后， 将发送`RECEIPT`frame到client.`RECEIPT`frame`必须`包含`receipt-id` header,它的值为client frame中`receipt`header的值。

    RECEIPT
    receipt-id:message-12345
    
    ^@

`RECEIPT`frame是作为server处理的client frame后的应答.
既然STOMP是基于流的，那么`receipt`也是对server已经收到所有的frames的累积确认。但是，以前的frames可能并没有被完全处理。如果clients断开连接，以前接收到的frames应该继续被server处理。

#### ERROR

如果出错的话，server将发送`ERROR`frame.这种情况下，server还应该断开连接。查看下一章[connection lingering](#Connection_Lingering)
`ERROR`frame应该包含带有简单错误信息的`message`header,或者`Body`包含详细的描述信息，也可能没有。

    ERROR
    receipt-id:message-12345
    content-type:text/plain
    content-length:171
    message: malformed frame received
    
    The message:
    -----
    MESSAGE
    destined:/queue/a
    receipt:message-12345
    
    Hello queue a!
    -----
    Did not contain a destination header, which is REQUIRED
    for message propagation.
    ^@

如果错误关联到了具体的某个client frame，那么server应该增加额外的headers去识别引起错误的frame。例如,如果frame包含`receipt`header,`ERROR`frame应该设置`receipt-id`header的值为引起错误的frame的`receipt`header的值。

如果frame body包含内容的话，`ERROR`frame应该包含`content-length`header和`content-type`header

### Frames and Headers

除了上述标准headers之外(`content-length`, `content-type`, `receipt`)，下面列出了所有规范中定义的headers:

* CONNECT or STOMP
  * REQUIRED: accept-version, host
  * OPTIONAL: login, passcode, heart-beat

* CONNECTED
  * REQUIRED: version
  * OPTIONAL: session, server, heart-beat

* SEND
  * REQUIRED: destination
  * OPTIONAL: transaction
* SUBSCRIBE
  * REQUIRED: destination, id
  * OPTIONAL: ack
* UNSUBSCRIBE
  * REQUIRED: id
  * OPTIONAL: none
* ACK or NACK
  * REQUIRED: id
  * OPTIONAL: transaction
* BEGIN or COMMIT or ABORT
  * REQUIRED: transaction
  * OPTIONAL: none
* DISCONNECT
  * REQUIRED: none
  * OPTIONAL: receipt
* MESSAGE
  * REQUIRED: destination, message-id, subscription
  * OPTIONAL: ack
* RECEIPT
  * REQUIRED: receipt-id
  * OPTIONAL: none
* ERROR
  * REQUIRED: none
  * OPTIONAL: message

除此之外，`SEND`和`MESSAGE`frames可能包含任意的用户定义的headers ,它们会成为carried message的一部分。同样，`ERROR`frame应该包含额外的headers来识别引起错误的frame。

最终，STOMP servers可以用额外的headers去访问`持久化`或者`有效期`特性.查阅server文档获得更多信息。

### Augmented BNF
A STOMP session can be more formally described using the Backus-Naur Form (BNF) grammar used in HTTP/1.1 RFC 2616.


    NULL                = <US-ASCII null (octet 0)>
    LF                  = <US-ASCII line feed (aka newline) (octet 10)>
    CR                  = <US-ASCII carriage return (octet 13)>
    EOL                 = [CR] LF 
    OCTET               = <any 8-bit sequence of data>
    
    frame-stream        = 1*frame
    
    frame               = command EOL
                          *( header EOL )
                          EOL
                          *OCTET
                          NULL
                          *( EOL )
    
    command             = client-command | server-command
    
    client-command      = "SEND"
                          | "SUBSCRIBE"
                          | "UNSUBSCRIBE"
                          | "BEGIN"
                          | "COMMIT"
                          | "ABORT"
                          | "ACK"
                          | "NACK"
                          | "DISCONNECT"
                          | "CONNECT"
                          | "STOMP"
    
    server-command      = "CONNECTED"
                          | "MESSAGE"
                          | "RECEIPT"
                          | "ERROR"
    
    header              = header-name ":" header-value
    header-name         = 1*<any OCTET except CR or LF or ":">
    header-value        = *<any OCTET except CR or LF or ":">

### License
This specification is licensed under the [Creative Commons Attribution v3.0](http://creativecommons.org/licenses/by/3.0/) license.

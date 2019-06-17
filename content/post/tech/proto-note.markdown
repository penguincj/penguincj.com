

Protocol Buffers 是一种轻便高效的结构化数据存储格式，可以用于结构化数据串行化，或者说序列化。它很适合做数据存储或数据交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。






# 序列化原理

Protobuf消息由一些字段组成，每个字段的格式为：

    修饰符 字段类型 字段名 = 域号;

在序列化时，protobuf按照TLV的格式序列化每一个字段，T即Tag，也叫Key；V是该字段对应的值value；L是Value的长度，如果一个字段是整形，这个L部分会省略。

序列化后的Value是按原样保存到字符串或者文件中，Key按照一定的转换条件保存起来，序列化后的结果就是 KeyValueKeyValue…。Key的序列化格式是按照message中字段后面的域号与字段类型来转换。转换公式如下：

    (tag << 3) | wire_type

上面的 tag 就是域号， wire_type 是与字段的类型有关，主要的类型如下：

| wire_type | meaning  | type  |
| -- | -- | -- |
|  0 | Vaint  |  int32、int64、uint32、uint64、sint32、sint64、bool、enum |
|  2 | Length-delimi | string、bytes、embedded、messages、packed repeated fields  |

一个 message 定义如下：

```go
message Person {
  string name = 1;
  int32 id = 2;  // Unique ID number for this person.
  repeated string email = 3;
}
```

protobuf协议使用二进制格式表示Key字段；对value而言，不同的类型采用的编码方式也不同，如果是整型，采用二进制表示；如果是字符，会直接原样写入文件或者字符串（即不编码）。

对于message中的每一个域，都对应一个域号。protobuf规定：

- 如果域号在[1，15]范围内，会使用一个字节表示Key；
- 如果域号大于等于16，会使用两个字节表示Key；


# 语言指南



## Oneof

```go
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

添加除了 `repeated` 之外的其他类型的字段到一个 oneof 的定义，在生成的代码中， oneof 字段有常规字段一样的公用的获取和设置的函数，还有检查类型的函数。可以通过查看 [API 文档](https://developers.google.com/protocol-buffers/docs/reference/overview)。


### Oneof 特点

- 设置 oneof 的字段会自动删除 oneof 中的其他字段，如果设置了多个字段，最后一个设置会生效。
- 如果解析遇到了一个 oneof 结构里有多个字段，采用最后一个看到的字段生效。
- oneof 不能 repeated
- 反射API对oneof字段有效。














# Protocol Buffer 扩展

在软件版本迭代的过程中不可避免的会有 proto 接口定义的更新，为了更好的向前、向后兼容性，有如下的规则需要遵守：

- 不要改变当前存在字段的 tag
- 可以删除字段
- 增加字段时使用一个新的 tag，这个 tag 是从来没有用过的，已经删除掉的也不能用

采用这些原则后的扩展，老的代码收到的新版本的消息时会忽略掉新的字段，已删除的单一字段用默认值填充，删除的重复字段将为空。新的代码将透明的读取老版本的消息。

> 新的字段不会出现在老版本的消息中，因此要处理好默认值。对于 strings 的默认值是空，对于 booleans 的默认值是 false，对于数字类型的默认值是0。默认字段也是可以设置的。


# gRPC

### 基于HTTP 2.0标准设计

由于gRPC基于HTTP 2.0标准设计，带来了更多强大功能，如多路复用、二进制帧、头部压缩、推送机制。这些功能给设备带来重大益处，如节省带宽、降低TCP连接次数、节省CPU使用等。gRPC既能够在客户端应用，也能够在服务器端应用，从而以透明的方式实现两端的通信和简化通信系统的构建。

#### 双向流、多路复用

在HTTP 1.X协议中，客户端在同一时间访问同一域名的请求数量是有限制的，当超过阈值时请求会被阻断，但是这种情况在HTTP 2.0中将被忽略。由于HTTP 1.X传输的是纯文本数据，传输体积较大，而HTTP 2.0传输的基本单元为帧，每个帧都包含消息，并且由于HTTP 2.0允许同时通过一条连接发起多个“请求-响应”消息，无需建立多个TCP链接的同时实现多条流并行，提高吞吐性能，并且在一个连接内对多个消息进行优先级的管理和流控。如图7。

<img alt="sonic-grpc-a5f5e079.png" src="assets/sonic-grpc-a5f5e079.png" width="" height="" >

图：双向流、多路复用特性

#### 二进制帧

相对于HTTP 1.X的纯文本传输来，HTTP 2.0传输的是二进制数据，与Protocol Buffers相辅相成。使得传输数据体积小、负载低，保持更加紧凑和高效。

#### 头部压缩

因为HTTP是无状态协议，对于业务的处理没有记忆能力，每一次请求都需要携带设备的所有细节，特别是在头部都会包含大量的重复数据，对于设备来说就是在不断地做无意义的重复性工作。HTTP 2.0中使用“头表”来跟踪之前发送的数据，对于相同的数据将不再使用重复请求和发送，进而减少数据的体积。

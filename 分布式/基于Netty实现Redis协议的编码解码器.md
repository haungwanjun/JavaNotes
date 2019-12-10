# 基于Netty实现Redis协议的编码解码器



## 在这篇文章中：

- [Netty消息处理结构](javascript:;)
- Redis协议编码解码的实现
  - [指令的编码格式](javascript:;)
  - [指令解码器的实现](javascript:;)
  - [输出消息编码器的实现](javascript:;)

# 

![img](https://ask.qcloudimg.com/http-save/yehe-1433204/naht8j7s91.jpeg?imageView2/2/w/1620)

![img](https://ask.qcloudimg.com/http-save/yehe-1433204/ux6w3fa50v.jpeg?imageView2/2/w/1620)

# **Netty消息处理结构** 

上面是Netty的服务器端基本消息处理结构，为了便于初学者理解，它和真实的结构有稍许出入。Netty是基于NIO的消息处理框架，用来高效处理网络IO。处理网络消息一般走以下步骤

1. 监听端口 Bind & Listen
2. 接收新连接 Accept
3. 通过连接收取客户端发送的字节流，转换成输入消息对象 Read & Decode
4. 处理消息，生成输出消息对象 Process
5. 转换成字节，通过连接发送到客户端 Encode & Write

步骤2拿到新连接之后，如果是开启了新线程进入步骤3，那就是走传统的多线程服务器模式。一个线程一个连接，每个线程是都阻塞式读写消息。如果并发量比较大，需要的线程资源也是比较多的。

Netty的消息处理基于NIO的多路复用机理，一个线程通过NIO Selector非阻塞地读写非常多的连接。传统的多线程服务器需要的线程数到了NIO这里就可以大幅缩减，节省了很多操作系统资源。

Netty的线程划分为两种，一种是用来监听ServerSocket并接受新连接的Acceptor线程，另一种用来读写套件字连接上的消息的IO线程，两种线程都是使用NIO Selector异步并行管理多个套件字。Acceptor线程可以同时监听多个ServerSocket，管理多个端口，将接收到的新连接扔到IO线程。IO线程可以同时读写多个Socket，管理多个连接的读写。

IO线程从套件字上读取到的是字节流，然后通过消息解码器将字节流反序列化成输入消息对象，再传递到业务处理器进行处理，业务处理器会生成输出消息对象，通过消息编码器序列化成字节流，再通过套件字输出到客户端。

# **Redis协议编码解码的实现**

本文的重点是教读者实现一个简单的Redis Protocol编码解码器。

![img](https://ask.qcloudimg.com/http-save/yehe-1433204/h3eja11ryh.jpeg?imageView2/2/w/1620)

首先我们来介绍一下Redis Protocol的格式，Redis协议分为指令和返回两个部分，指令的格式比较简单，就是一个字符串数组，比如指令setnx a b就是三个字符串的数组，如果指令中有整数，也是以字符串的形式发送的。Redis协议的返回就比较复杂了，因为要支持复杂的数据类型和结构嵌套。本文是以服务端的角色来处理Redis协议，也就是编写指令的解码器和返回对象的编码器。而客户端则是反过来的，客户端需要编写指令的编码器和返回对象的解码器。

## **指令的编码格式**

setnx a b => *3\r\n$5\r\nsetnx\r\n$1\r\na\r\n$1\r\nb\r\n

指令是一个字符串数组，编码一个字符串数组，首先需要编码数组长度*3\r\n。然后依次编码各个字符串参数。编码字符串首先需要编码字符串的长度$5\r\n。然后再编码字符串的内容setnx\r\n。Redis消息以\r\n作为分隔符，这样设计其实挺浪费网络传输流量的，消息内容里面到处都是\r\n符号。但是这样的消息可读性会比较好，便于调试。这也是软件世界牺牲性能换取可读性便捷性的一个经典例子。

## **指令解码器的实现**

网络字节流的读取存在半包问题。所谓半包问题是指一次Read调用从套件字读到的字节数组可能只是一个完整消息的一部分。而另外一部分则需要发起另外一次Read调用才可能读到，甚至要发起多个Read调用才可以读到完整的一条消息。

如果我们拿部分消息去反序列化成输入消息对象肯定是要失败的，或者说生成的消息对象是不完整填充的。这个时候我们需要等待下一次Read调用，然后将这两次Read调用的字节数组拼起来，尝试再一次反序列化。

问题来了，如果一个输入消息对象很大，就可能需要多个Read调用和多次反序列化操作才能完整的解包出一个输入对象。那这个反序列化的过程就会重复了多次。比如第一次完成了30%，然后第二次从头开始又完成了60%，第三次又从头开始完成了90%，第四次又从头开始总算完成了100%，这下终于可以放心交给业务处理器处理了。

![img](https://ask.qcloudimg.com/http-save/yehe-1433204/ku6erawlog.png?imageView2/2/w/1620)

Netty使用**ReplayingDecoder**引入检查点机制**[Checkpoint]**解决了这个重复反序列化的问题。

![img](https://ask.qcloudimg.com/http-save/yehe-1433204/qgkzova27e.png?imageView2/2/w/1620)

在反序列化的过程中我们反复打点记录下当前读到了哪个位置，也就是检查点，然后下次反序列化的时候就可以从上次记录的检查点直接继续反序列化。这样就避免了重复的问题。

这就好比我们玩单机RPG游戏一样，这些游戏往往有自动保存的功能。这样就可以避免进程不小心退出时，再进来的时候就可以从上次保存的状态直接继续进行下去，而不是像Flappy Bird一样重新玩它一遍又一遍，这简直要把人虐死。

![img](https://ask.qcloudimg.com/http-save/yehe-1433204/gvl03neitp.gif)

```javascript
import java.util.ArrayList;
import java.util.List;

import com.google.common.base.Charsets;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.DecoderException;
import io.netty.handler.codec.ReplayingDecoder;

class InputState {
    public int index;
}

public class RedisInputDecoder extends ReplayingDecoder<InputState> {

    private int length;
    private List<String> params;

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        InputState state = this.state();
        if (state == null) {
            length = readParamsLen(in);
            this.params = new ArrayList<>(length);
            state = new InputState();
            this.checkpoint(state);
        }
        for (int i = state.index; i < length; i++) {
            String param = readParam(in);
            this.params.add(param);
            state.index = state.index + 1;
            this.checkpoint(state);
        }
        out.add(new RedisInput(params));
        this.checkpoint(null);
    }

    private final static int CR = '\r';
    private final static int LF = '\n';
    private final static int DOLLAR = '$';
    private final static int ASTERISK = '*';

    private int readParamsLen(ByteBuf in) {
        int c = in.readByte();
        if (c != ASTERISK) {
            throw new DecoderException("expect character *");
        }
        int len = readLen(in, 3); // max 999 params
        if (len == 0) {
            throw new DecoderException("expect non-zero params");
        }
        return len;
    }

    private String readParam(ByteBuf in) {
        int len = readStrLen(in);
        return readStr(in, len);
    }

    private String readStr(ByteBuf in, int len) {
        if (len == 0) {
            return "";
        }
        byte[] cs = new byte[len];
        in.readBytes(cs);
        skipCrlf(in);
        return new String(cs, Charsets.UTF_8);
    }

    private int readStrLen(ByteBuf in) {
        int c = in.readByte();
        if (c != DOLLAR) {
            throw new DecoderException("expect character *");
        }
        return readLen(in, 6); // string maxlen 999999
    }

    private int readLen(ByteBuf in, int maxBytes) {
        byte[] digits = new byte[maxBytes]; // max 999个参数
        int len = 0;
        while (true) {
            byte d = in.getByte(in.readerIndex());
            if (!Character.isDigit(d)) {
                break;
            }
            in.readByte();
            digits[len] = d;
            len++;
            if (len > maxBytes) {
                throw new DecoderException("params length too large");
            }
        }
        skipCrlf(in);
        if (len == 0) {
            throw new DecoderException("expect digit");
        }
        return Integer.parseInt(new String(digits, 0, len));
    }

    private void skipCrlf(ByteBuf in) {
        int c = in.readByte();
        if (c == CR) {
            c = in.readByte();
            if (c == LF) {
                return;
            }
        }
        throw new DecoderException("expect cr ln");
    }

}
```

## **输出消息编码器的实现**

高能预警：前方有大量代码，请酌情观看

输出消息的结构要复杂很多，要支持多种数据类型，包括状态、整数、错误、字符串和数组，要支持数据结构嵌套，数组里还有数组。相比解码器而言它简单的地方在于不用考虑半包问题，编码器只负责将消息序列化成字节流，剩下的事由Netty偷偷帮你搞定。

首先我们定义一个输出消息对象接口，所有的数据类型都要实现该接口，将对象内部的状态转换成字节数组放置到ByteBuf中。

```javascript
import io.netty.buffer.ByteBuf;

public interface IRedisOutput {

    public void encode(ByteBuf buf);

}
```

整数输出消息类，整数的序列化格式为**:value\r\n**，value是整数的字符串表示。

```javascript
import com.google.common.base.Charsets;
import io.netty.buffer.ByteBuf;

public class IntegerOutput implements IRedisOutput {

    private long value;

    public IntegerOutput(long value) {
        this.value = value;
    }

    @Override
    public void encode(ByteBuf buf) {
        buf.writeByte(':');
        buf.writeBytes(String.valueOf(value).getBytes(Charsets.UTF_8));
        buf.writeByte('\r');
        buf.writeByte('\n');
    }

    public static IntegerOutput of(long value) {
        return new IntegerOutput(value);
    }

    public static IntegerOutput ZERO = new IntegerOutput(0);
    public static IntegerOutput ONE = new IntegerOutput(1);

}
```

状态输出消息类，序列化格式为**+status\r\n**

```javascript
import com.google.common.base.Charsets;
import io.netty.buffer.ByteBuf;

public class StateOutput implements IRedisOutput {

    private String state;

    public StateOutput(String state) {
        this.state = state;
    }

    public void encode(ByteBuf buf) {
        buf.writeByte('+');
        buf.writeBytes(state.getBytes(Charsets.UTF_8));
        buf.writeByte('\r');
        buf.writeByte('\n');
    }

    public static StateOutput of(String state) {
        return new StateOutput(state);
    }

    public final static StateOutput OK = new StateOutput("OK");

    public final static StateOutput PONG = new StateOutput("PONG");

}
```

错误输出消息类，序列化格式为**-type reason\r\n**，reason必须为单行字符串

```javascript
import com.google.common.base.Charsets;
import io.netty.buffer.ByteBuf;

public class ErrorOutput implements IRedisOutput {

    private String type;
    private String reason;

    public ErrorOutput(String type, String reason) {
        this.type = type;
        this.reason = reason;
    }

    public String getType() {
        return type;
    }

    public String getReason() {
        return reason;
    }

    @Override
    public void encode(ByteBuf buf) {
        buf.writeByte('-');
        // reason不允许多行字符串
        buf.writeBytes(String.format("%s %s", type, headOf(reason)).getBytes(Charsets.UTF_8));
        buf.writeByte('\r');
        buf.writeByte('\n');
    }

    private String headOf(String reason) {
        int idx = reason.indexOf("\n");
        if (idx < 0) {
            return reason;
        }
        return reason.substring(0, idx).trim();
    }

    // 通用错误
    public static ErrorOutput errorOf(String reason) {
        return new ErrorOutput("ERR", reason);
    }

    // 语法错误
    public static ErrorOutput syntaxOf(String reason) {
        return new ErrorOutput("SYNTAX", reason);
    }

    // 协议错误
    public static ErrorOutput protoOf(String reason) {
        return new ErrorOutput("PROTO", reason);
    }

    // 参数无效
    public static ErrorOutput paramOf(String reason) {
        return new ErrorOutput("PARAM", reason);
    }

    // 服务器内部错误
    public static ErrorOutput serverOf(String reason) {
        return new ErrorOutput("SERVER", reason);
    }

}
```

字符串输出消息类，字符串分为null、空串和普通字符串。null的序列化格式为**$-1\r\n**，普通字符串的格式为**$len\r\ncontent\r\n，**空串就是一个长度为0的字符串，格式为**$0\r\n\r\n**。

```javascript
import com.google.common.base.Charsets;
import io.netty.buffer.ByteBuf;

public class StringOutput implements IRedisOutput {

    private String content;

    public StringOutput(String content) {
        this.content = content;
    }

    @Override
    public void encode(ByteBuf buf) {
        buf.writeByte('$');
        if (content == null) {
            // $-1\r\n
            buf.writeByte('-');
            buf.writeByte('1');
            buf.writeByte('\r');
            buf.writeByte('\n');
            return;
        }
        byte[] bytes = content.getBytes(Charsets.UTF_8);
        buf.writeBytes(String.valueOf(bytes.length).getBytes(Charsets.UTF_8));
        buf.writeByte('\r');
        buf.writeByte('\n');
        if (content.length() > 0) {
            buf.writeBytes(bytes);
        }
        buf.writeByte('\r');
        buf.writeByte('\n');
    }

    public static StringOutput of(String content) {
        return new StringOutput(content);
    }

    public static StringOutput of(long value) {
        return new StringOutput(String.valueOf(value));
    }

    public final static StringOutput NULL = new StringOutput(null);

}
```

最后一个数组输出消息类，支持数据结构嵌套就靠它了。数组的内部是多个子消息，每个子消息的类型是不定的，类型可以不一样。比如scan操作的返回就是一个数组，数组的第一个子消息是游标的offset字符串，第二个子消息是一个字符串数组。它的序列化格式开头为***len\r\n**，后面依次是内部所有子消息的序列化形式。

```javascript
import java.util.ArrayList;
import java.util.List;
import com.google.common.base.Charsets;
import io.netty.buffer.ByteBuf;

public class ArrayOutput implements IRedisOutput {

    private List<IRedisOutput> outputs = new ArrayList<>();

    public static ArrayOutput newArray() {
        return new ArrayOutput();
    }

    public ArrayOutput append(IRedisOutput output) {
        outputs.add(output);
        return this;
    }

    @Override
    public void encode(ByteBuf buf) {
        buf.writeByte('*');
        buf.writeBytes(String.valueOf(outputs.size()).getBytes(Charsets.UTF_8));
        buf.writeByte('\r');
        buf.writeByte('\n');
        for (IRedisOutput output : outputs) {
            output.encode(buf);
        }
    }

}
```

下面是ArrayOutput对象使用的一个实例，是从小编的某个项目里扒下来的，嵌套了三层数组。读者可以不用死磕下面的代码，重点看看ArrayOutput大致怎么使用的就可以了。

```javascript
ArrayOutput out = ArrayOutput.newArray();
for (Result result : res) {
    if (result.isEmpty()) {
        continue;
    }
    ArrayOutput row = ArrayOutput.newArray();
    row.append(StringOutput.of(new String(result.getRow(), Charsets.UTF_8)));
    for (KeyValue kv : result.list()) {
        ArrayOutput item = ArrayOutput.newArray();
        item.append(StringOutput.of("family"));
        item.append(StringOutput.of(new String(kv.getFamily(), Charsets.UTF_8)));
        item.append(StringOutput.of("qualifier"));
        item.append(StringOutput.of(new String(kv.getQualifier(), Charsets.UTF_8)));
        item.append(StringOutput.of("value"));
        item.append(StringOutput.of(new String(kv.getValue(), Charsets.UTF_8)));
        item.append(StringOutput.of("timestamp"));
        item.append(StringOutput.of(kv.getTimestamp()));
        row.append(item);
    }
    out.append(row);
}
ctx.writeAndFlush(out);
```

最后，有了以上清晰的类结构，解码器类的实现就非常简单了。

```javascript
import java.util.List;
import io.netty.channel.ChannelHandler.Sharable;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.PooledByteBufAllocator;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToMessageEncoder;

@Sharable
public class RedisOutputEncoder extends MessageToMessageEncoder<IRedisOutput> {

    @Override
    protected void encode(ChannelHandlerContext ctx, IRedisOutput msg, List<Object> out) throws Exception {
        ByteBuf buf = PooledByteBufAllocator.DEFAULT.directBuffer();
        msg.encode(buf);
        out.add(buf);
    }

}
```

因为解码器对象是无状态的，所以它可以被channel共享。解码器的实现非常简单，就是分配一个ByteBuf，然后将将消息输出对象序列化的字节数组塞到ByteBuf中输出就可以了。
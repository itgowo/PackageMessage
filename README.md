# PackageMessage
##### A tcp data packaging solution that supports the handling of sticky packets
##### 这是一个TCP数据打包方案，适用于长连接中遇到的粘包和半包问题。

Github地址：https://github.com/itgowo/PackageMessage

至于Java Nio实现长连接方案，稍后会发，浏览我的文章就可以找到。

### 一：名词解释
#### 半包
指接受方没有接受到一个完整的包，只接受了部分，这种情况主要是由于TCP为提高传输效率，将一个包分配的足够大，导致接受方并不能一次接受完。（在长连接和短连接中都会出现）。 
#### 粘包和分包
指发送方发送的若干包数据到接收方接收时粘成一包，从接收缓冲区看，后一包数据的头紧接着前一包数据的尾。出现粘包现象的原因是多方面的，它既可能由发送方造成，也可能由接收方造成。发送方引起的粘包是由TCP协议本身造成的，TCP为提高传输效率，发送方往往要收集到足够多的数据后才发送一包数据。若连续几次发送的数据都很少，通常TCP会根据优化算法把这些数据合成一包后一次发送出去，这样接收方就收到了粘包数据。接收方引起的粘包是由于接收方用户进程不及时接收数据，从而导致粘包现象。这是因为接收方先把收到的数据放在系统接收缓冲区，用户进程从该缓冲区取数据，若下一包数据到达时前一包数据尚未被用户进程取走，则下一包数据放到系统接收缓冲区时就接到前一包数据之后，而用户进程根据预先设定的缓冲区大小从系统接收缓冲区取数据，这样就一次取到了多包数据。分包是指在出现粘包的时候我们的接收方要进行分包处理。（在长连接中都会出现）
### 二：简单使用
###### 解包
```
        ByteBuffer buffer = ByteBuffer.newByteBuffer();
       if (msg instanceof byte[]) {
            buffer.writeBytes((byte[]) msg);
        }
        List<PackageMessage> packageMessageList = packageMessage.packageMessage(buffer);
```

##### 打包
```
    ByteBuffer buffer = ByteBuffer.newByteBuffer();
    buffer.writeBytes((byte[]) msg);
    PackageMessage packageMessage=PackageMessage.getPackageMessage()
           .setDataType(PackageMessage.DATA_TYPE_BYTE).setData(data);
     byte[] request = msg.encodePackageMessage().readableBytesArray();
```
### 三：原理解析
![这是一个理想状态](https://upload-images.jianshu.io/upload_images/3213604-e4c8ee3dc2c7fe8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1.如上图，理想状态下数据发送和接收是一样的。

![半包](https://upload-images.jianshu.io/upload_images/3213604-a1ee9f6993b505b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.如上图，如果遇到半包情况，decode会将接收到的数据暂存起来保存在next对象里，当下次来数据时先将新数据拼接到next的数据尾部，然后再按照正常解析过程执行。默认遇到半包时，会将状态置为半包状态，next的处理指针重置为0。

![粘包](https://upload-images.jianshu.io/upload_images/3213604-ffbfa6d57e43a596.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.如上图，如果发生粘包，解析先判断是否为动态长度数据包，如果是则读取length信息，从数据中读出length长度的数组后返回一个PackageMessage，并将next已读部分销毁掉，重置指针，再次读下一PackageMessage，如此往复，直到读取不到数据，将解析出来的PackageMessage以List方式返回。

![半包和粘包](https://upload-images.jianshu.io/upload_images/3213604-025042d270c0eafb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4.如上图，当同时发生半包和粘包时与单独遇到问题一样，解析成功的PackageMessage返回List，剩余的当做半包处理。


### 四：疑难解答

#### 1.为什么第一个字节是type？
因为动态长度和定长解析不一样。
#### 2.为什么length放在这么前面的位置？
因为我要先知道数据长度才知道截取到哪里，这个截取区间就是一个封装过的数据包。如果过晚知道length，可能处理不及时漏掉，尤其网络波动丢了，当然一般不会丢。
#### 3. dataType作用是什么？
dataType指明该数据包数据类型，如果为心跳，那么length=6，没有dataSign和data，极大的缩减数据量。如果为其他类型，例如文件流，可以先存成文件，遍存边读，如果是json，一次性读取再解析。极大增加了灵活性。
#### 4.dataSign必须实现吗？
不必须，一般默认算法即可，重写算法亦可。
#### 5.怎么实现的拆包机制？
假设收到3个包的数据量，从读取第一个字节开始，读到第五个字节就知道了这个包大小，如果数据长度大于length，则取出length长度，生成一个PackageMessage对象，丢弃已读数据，然后继续按新包读数据解析，直到没有数据。如果读到没数据，包解析只有一半，及状态为STEP_DATA_PART，表示为读取一半，为半包数据，会临时存入next对象，并放弃解析进度，下次解析数据时next中的旧数据与新数据合并。

#### 6.适合大数据量传输吗？
不适合，数据量大内存消耗大，最好传送的是简单数据，不要传文件等，现在实现机制里没有做文件缓存，可以考虑Java Nio的ByteBuffer实现内存映射方式。



### 五：数据包格式
提示：包最小为6字节

|名称|长度|类型|是否必须|取值范围|说明|
|---|---|---|---|---|---|
|type|1|Byte|是|-127 ~ 128|消息类型 |
|length|4|Integer|是|0-4GB|消息消息包总长度|
|dataType|1|Byte|是|11-128|0-10 是预定义或保留值 |
|dataSign|4|Integer|否|0-Integer.MAX_VALUE| 验证信息，用来校验数据请求|
|data|0-4GB|ByteArray|否|无|承载的数据|

#### 1.type定义
用来标识数据包类型，分为动态长度和固定长度。

|常量名|名称|值|说明|
|---|---|---|---|
|TYPE_DYNAMIC_LENGTH|动态长度|121|与length值相关，适合传输不固定数据大小情况，尤其是时大时小波动范围大的情况|
|TYPE_FIX_LENGTH|固定长度|120|截止到写文档时刻，定长解析未做处理，需要重写**decodeFixLengthPackageMessage()**方法。|

#### 2.length定义
用来标识数据包大小，在动态数据包解析中决定了二进制数据切割位置。数值为包大小，包含全部数据。

#### 3.dataType定义
用来标识data的数据类型，0-10 是预定义或保留值。

|常量名|名称|值|说明|
|---|---|---|---|
|DATA_TYPE_COMMAND|指令|1|控制级别的数据传输|
|DATA_TYPE_HEART|心跳|2|心跳信息，控制级别消息，长度为6字节|
|DATA_TYPE_BYTE|二进制|3|传输内容为二进制数据，适合传输任意数据|
|DATA_TYPE_TEXT|文本|4|传输内容为文本数据|
|DATA_TYPE_JSON|Json文本|5|传输内容为Json格式文本数据|

#### 4.dataSign定义
用来校验data中数据完整性或正确性的参数。
默认算法为data长度小于等于10则返回1；如果大于10，则取length的四分之一和四分之三位置的数值，并用4个字节记录下1/4位置、1/4位置的数据、3/4位置和3/4位置的数据；
将四字节数据转换成int形成校验值。

也可自定义校验规则。
#### 5.data定义
用来存储携带数据，可以为任何可以转换为二进制的数据。实际data对象为一个支持动态扩展的ByteBuffer，相对于java nio来说更加灵活。目前ByteBuffer也分享出来，项目地址：https://github.com/itgowo/ByteBuffer

#### 6.step定义
用来标识包解析状态的，在业务处理中大部分已经废弃，用来开发调试分析这个过程很有用，可以知道到哪里解析出错了。

|常量名|值|说明|
|---|---|---|
|STEP_DEFAULT|0|new 出来默认值|
|STEP_TYPE|1|读取type值|
|STEP_LENGTH|2|读取length值|
|STEP_DATA_TYPE|3|读取dataType值|
|STEP_DATA_SIGN|4|读取dataSign值|
|STEP_DATA_COMPLETEED|5|读取data数据完整|
|STEP_DATA_PART|6|读取data数据部分|
|STEP_DATA_INVALID|7|无效包|
 
### 六：小期待
以下项目都是我围绕远程控制写的子项目。都给star一遍吧。😍

|项目(Github)|语言|其他地址|运行环境|项目说明|
|---|---|---|---|---|
|[PackageMessage](https://github.com/itgowo/PackageMessage)|Java|[简书](https://www.jianshu.com/p/8a4a0ba2f54a)|运行Java的设备|TCP粘包与半包解决方案|
|[ByteBuffer](https://github.com/itgowo/ByteBuffer)|Java|[简书](https://www.jianshu.com/p/ba68224f30e4)|运行Java的设备|二进制处理工具类|
|[RemoteDataControllerForAndroid](https://github.com/itgowo/RemoteDataControllerForAndroid)|Java|[简书](https://www.jianshu.com/p/eb692f5709e3)|Android设备|远程数据调试Android端|
|[RemoteDataControllerForWeb](https://github.com/itgowo/RemoteDataControllerForWeb)|JavaScript|[简书](https://www.jianshu.com/p/75747ff4667f)|浏览器|远程数据调试控制台Web端|
|[RemoteDataControllerForServer](https://github.com/itgowo/RemoteDataControllerForServer)|Java|[简书](https://www.jianshu.com/p/3858c7e26a98)|运行Java的设备|远程数据调试Server端|
|[MiniHttpClient](https://github.com/itgowo/MiniHttpClient)|Java|[简书](https://www.jianshu.com/p/41b0917271d3)|运行Java的设备|精简的HttpClient|
|[MiniHttpServer](https://github.com/itgowo/MiniHttpServer)|Java|[简书](https://www.jianshu.com/p/de98fa07140d)|运行Java的设备|支持部分Http协议的Server|
|[DataTables.AltEditor](https://github.com/itgowo/DataTables.AltEditor)|JavaScript|[简书](https://www.jianshu.com/p/a28d5a4c333b)|浏览器|Web端表格编辑组件|

[我的小站：IT狗窝](http://itgowo.com)
技术联系QQ:1264957104

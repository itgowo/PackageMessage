# PackageMessage
A tcp data packaging solution that supports the handling of sticky packets
Github地址：https://github.com/itgowo/PackageMessage
[另一个组合型框架方案，包含websocket、socket和http等支持及拓展的ActionFramework](https://github.com/itgowo/ActionFramework)
这是一个TCP数据打包方案，适用于长连接中遇到的黏包和拆包问题。

### 数据包格式
包最小为6字节

|名称|长度|类型|是否必须|取值范围|说明|
|---|---|---|---|---|---|
|type|1|Byte|是|-127 ~ 128|消息类型 |
|length|4|Integer|是|0-4GB|消息消息包总长度|
|dataType|1|Byte|是|11-128|0-10 是预定义或保留值 |
|dataSign|4|Integer|否|0-Integer.MAX_VALUE| 验证信息，用来校验数据请求|
|data|0-4GB|ByteArray|否|无|承载的数据|

### type定义
用来标识数据包类型，分为动态长度和固定长度。

|常量名|名称|值|说明|
|---|---|---|---|
|TYPE_DYNAMIC_LENGTH|动态长度|121|与length值相关，适合传输不固定数据大小情况，尤其是时大时小波动范围大的情况|
|TYPE_FIX_LENGTH|固定长度|120|截止到写文档时刻，定长解析未做处理，需要重写**decodeFixLengthPackageMessage()**方法。|

### length定义
用来标识数据包大小，在动态数据包解析中决定了二进制数据切割位置。数值为包大小，包含全部数据。

### dataType定义
用来标识data的数据类型，0-10 是预定义或保留值。

|常量名|名称|值|说明|
|---|---|---|---|
|DATA_TYPE_COMMAND|指令|1|控制级别的数据传输|
|DATA_TYPE_HEART|心跳|2|心跳信息，控制级别消息，长度为6字节|
|DATA_TYPE_BYTE|二进制|3|传输内容为二进制数据，适合传输任意数据|
|DATA_TYPE_TEXT|文本|4|传输内容为文本数据|
|DATA_TYPE_JSON|Json文本|5|传输内容为Json格式文本数据|

### dataSign定义
用来校验data中数据完整性或正确性的参数。
默认算法为data长度小于等于10则返回1；如果大于10，则取length的四分之一和四分之三位置的数值，并用4个字节记录下1/4位置、1/4位置的数据、3/4位置和3/4位置的数据；
将四字节数据转换成int形成校验值。

也可自定义校验规则。
### data定义
用来存储携带数据，可以为任何可以转换为二进制的数据。实际data对象为一个支持动态扩展的ByteBuffer，相对于java nio来说更加灵活。目前ByteBuffer也分享出来，项目地址：https://github.com/itgowo/ByteBuffer

### step定义
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
 

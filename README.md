# PackageMessage
A tcp data packaging solution that supports the handling of sticky packets

这是一个TCP数据打包方案，适用于长连接中遇到的黏包和拆包问题。

### 数据包格式

|名称|长度|类型|是否必须|取值范围|说明|
|---|---|---|---|---|---|
|type|1|Byte|是|-127 ~ 128|消息类型 |
|length|4|Integer|是|0-4GB|消息消息包总长度|
|dataType|1|Byte|是|11-128|0-10 是预定义或保留值 |
|dataSign|4|Integer|否|0-Integer.MAX_VALUE| 验证信息，用来校验数据请求|
|data|0-4GB|ByteArray|否|无|承载的数据|

---
title: OpenSIPS 核心变量
---

# OpenSIPS 核心变量

**OpenSIPS**为路由脚本提供了很多种类型的可使用的变量。不通种类变量的不同点来自一下三处：
（1） 变量可见性（何时可见）
（2）变量被附加到的位置（变量在哪里可用）
（3）变量的读-写状态（某些变量只可读）
（4）如何处理多个值（对于相同变量）

OpenSIPS变量很容易在脚本中声明，所有变量的名称（或标记）以`$`开始
语法：
完整的伪变量的语法如下：
`$(<context>name(subname)[index]{transformation})` 绿色（除了$(name)以外的部分，markdown没法给字体标颜色）的字段是可选的。字段的含义如下：

- name(必须的) 伪变量的名称(类型)
  例如: pvar, avp, ru, DLG_status, 等
- subname 给定类型的伪变量的真实名称
  例如: hdr(From), avp(name)
- index 一个伪变量可以存储 不止一个 值-存储类似列表（list）。如果你具体声明了索引，你可以通过索引来使用确定的变量。
- context - the context in which the pseudo0variable will be evaluated. Now there are 2 pv contexts: reply and request. The reply context can be used in the failure route to request for the pseudo-variable to be evaluated in the context of the reply message. The request context can be used if in a reply route is desired for the pv to be evaluated in the context of the corresponding request.

使用用例：

- 只使用名称：$ru
- 名称+真实名称：$hdr(Contact)
- 名称+索引：$(ct[0])
- 名称+真实名称+索引：$(avp()[])
- Context
  - $(ru) 从响应（reply）路由可以获得从请求中的Request-URI
  - $(hdr(Contact)) context可以被用在失败路由（failure route）中，在这里可以使用reply中的消息

变量类型：

1. 脚本变量 - 正如其名，这种变量被严格的绑定在脚本路由中。这中变量只有在路由模块中才具有可见性 - 他们不是消息和事务相关的，他们是用来联系消息和事务的（脚本变量会在相同的OpenSIPS进程在处理路由模块的时候继承）
   脚本变量可读写，可以是整型（integer）和字符串（string）类型。脚本变量只能有单一值。一个新的赋值语句（或者写操作）将会覆盖原来存在的值。
2. AVP(Attribute Value Pair)变量 - 键-值对是动态变量，他们可以被创建。AVPS被单个消息或者事务链接起来（如果使用状态处理）。一个消息或者事务初始化（当收到或创建时）会有一个属于他的空AVPS列表。在执行脚本路由时，在脚本创建或通过函数调用都会创建新的变量，这些AVP变量将会和消息或事务关联起来。AVP变量在所有的路由中处理的所有消息（请求和应答）都是可以访问的-分支路由（branch_route），失败路由（failure_route）,应答路由（reply_route，使用这个路由需要启用TM 参数 onreply_avp_mode）。
   AVP变量是可读写的，并且已存在的AVP变量可以被删除。AVP可以包含多个值， - 一个新的赋值语句（或写操作）回给AVP变量添加一个新的值；这些值将会被按照”后进先出”的方式排序（栈）
   可以通过声明一个append在列表的最后面（栈底）添加新的值 - `$(avp(name)[append])="last value"`;
3. 伪变量 - 伪变量（PV）从已处理的SIP消息中提供对信息的访问（headers,RURI,transport level info, a.s.o）或者从OpenSIPS 内部（时间，进程ID，返回码，或者函数）。取决于绑定的消息，PV变量要么绑定到消息，要么为0。大多数PV变量是制度的，只有几个支持写操作。PV变量会返回多个值，或者一个值，返回值的数量取决于退回消息（是否可以有多个值）
   标准PV变量是只读的，并且返回单个值（如果不是，请参考文档）
4. 转义序列(escape sequences) - 转义序列用来格式化字符串，他们实际上不是变量，而是格式化。

#### 1. 脚本变量

名称：`$var(name)`
提示：
\1. 如果你希望在路由中使用脚本变量，最好使用相同的值初始化他（或者重置），否则你可以从同一进程的前一条路由中继承一个值。
\2. 脚本变量比AVP变量要快，他们直接引用内存地址
\3. 脚本变量的值保存在OepnSIPS过程中
\4. 一个脚本变量只能含有一个值
示例：

```
$var(a) = 2;  # 设置a的值为整型的 '2'
$var(a) = "2";  # 设置a的值为字符串类型的 '2'
$var(a) = 3 + (7&(~2)); # 算数和位运算
$var(a) = "sip:" + $au + "@" + $fd; # 从身份验证用户名和URI域中组合一个值

# 使用脚本变量进行测试
if( [ $var(a) & 4 ] ) {
  xlog("var a has third bit set\n");
}123456789
```

设置变量为NULL实际上是初始化变量值为0，脚本变量不能有NULL值

#### 2. AVP 变量

名称：`$avp(name)` 或 `$(avp(name)[N})`
当使用索引N时，你可以使AVP返回一个确定的值（第N个值）。如果没有给出索尼，将会返回第一个值。
提示：

1. 如果要在响应路由（reply route）中使用AVP，需要使用`$(avp(name)[N])`

2. 如果单个AVP变量需要使用多个值，变量的索引是相反的顺序，而不是添加的顺序

3. AVP的值可以被删除

   例：

4. 事物持久化示例

```
# 在响应路由（onreply route）中启用avp
modparam("tm", "onreply_avp_mode", 1)
...
route{
...
$avp(tmp) = $Ts ; # 存取当前时间
...
t_onreply("1");
t_relay();
...
}

onreply_route[1] {
    if (t_check_status("200")) {
        # 计算设置时间
        $var(setup_time) = $Ts - $avp(tmp);
    }
}123456789101112131415161718
```

1. 多值示例

```
$avp(17) = "one";
# 一个值
$avp(17) = "two";
# 两个值 ("two","one")
$avp(17) = "three";
# 三个值 ("three","two","one")

xlog("accessing values with no index: $avp(17)\n");
# 打印第一个值，即最后加入的值 -> "three"

xlog("accessing values with no index: $(avp(17)[2])\n");
# this will print the index 2 value (third one), -> "one"

# 移除avp的第一个值; 如果只有一个值，则AVP将会被销毁
$avp(17) = NULL;

# 移除所有avp的值
avp_delete("$avp(17)/g");

# 删除指定索引位置的值 
$(avp(17)[1]) = NULL;

# 预览指定索引位置的值
$(avp(17)[0]) = "zero";123456789101112131415161718192021222324
```

AVPOPS模块提供了大量操作AVP的函数（比如：检查值，将值放入不通的位置，删除AVP等）

#### 3. 伪变量（未完成，用到慢慢翻译）

名称：$name
提示：

3.19 Call-ID
`$ci` 引用SIP消息里call-id头部中的内容

3.20 Content-Lenth
`$cl` 引用content-length头部中的内容

3.21 CSeq number
`$cs` 引用 cseq头部中的cseq号码（序列号）

3.29 目的URI端口（Port of destination URI)
`$dp` 引用 目的URI中的端口

3.30 目的URI的传输协议
`$dp` 引用目的URI的传输协议

3.32 目的URI
`$du` 引用目的URI（用于发送请求的出站代理）如果`loose_route()`返回true，目的URI将会根据第一个Route头部设置
*$du是可读写的变量，你可以根据路由逻辑来设定他*

3.32 目标 URI
`$du` 参考目标URI（出境代理用来发送请求）

3.41 From tag
`$ft` 引用From头部中的tag标签

3.42 From URI
`$fu` 引用SIP头部的From标签的URI

3.42 From URI
`$fu` 引用SIP头部的From标签的URI中的用户名

3.51 SIP请求原始URI的传输协议
`$oP` SIP请求原始URI的传输协议

3.51 SIP请求原始URI
`$ou` 引用请求的原始URI

3.65 SIP请求方法
`$rm` 引用请求的方法（INVITE，OPTIONS，ACK等等）

3.77 IP源地址
`$si` 引用SIP消息的IP源地址

3.78 IP源端口
`$sp` 引用SIP消息的IP源端口

3.82 To URI
`$tu` 引用SIP头部的TO标签中的URI

3.83 To URI Username
`$tu` 引用 To 头部的中URI的用户名
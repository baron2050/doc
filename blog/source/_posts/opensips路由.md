---
title: opensips路由
---

opensips路由

opensips路由分为两类，主路由和子路由。**主路由被opensips调用，子路由在主路由中被调用。可以理解子路由是一种函数。**

**所有路由中不允许出现无任何语句的情况，否则将会导致opensips无法正常启动，例如下面**

```
route[some_xxx]{

}
```

主路由分为几类

1. 请求路由

2. 分支路由

3. 失败路由

4. 响应路由

5. 本地路由

6. 启动路由

7. 定时器路由

8. 事件路由

9. 错误路由

inspect：查看sip消息内容

modifies: 修改sip消息内容，例如修改request url

drop: 丢弃sip请求

forking: 可以理解为发起一个invite, 然后可以拨打多个人

signaling: 信令层的操作，例如返回200ok之类的

| 路由     | 是否必须 | 默认行为           | 可以做                            | 不可以做                     | 触发方向                        | 触发次数           |
| -------- | -------- | ------------------ | --------------------------------- | ---------------------------- | ------------------------------- | ------------------ |
| 请求路由 | 是       | drop               | inspect,modifies, drop, signaling |                              | incoming, inbound               |                    |
| 分支路由 | 否       | send out           | forking, modifies, drop, inspect  | relaying, replying,signaling | outbound, outgoing, branch frok | 一个请求/事务一次  |
| 失败路由 | 否       | 将错误返回给产生者 | signaling，replying, inspect      |                              | incoming                        | 一个请求/事务一次  |
| 响应路由 | 否       | relay back         | inspect, modifies                 | signaling                    | incoming, inbound               | 一个请求/事务一次  |
| 本地路由 | 否       | send out           |                                   | signaling                    | outbound                        | 本地路由只能有一个 |

剩下的启动路由，定时器路由，事件路由，错误路由只能用来做和sip消息无关的事情。

## 请求路由

请求路由因为受到从外部网络来的请求而触发。

```c
 # 主路由
route {
      ......
     if (is_method("INVITE")) {
           route(check_hdrs,1); # 调用子路由check_hdrs，1是传递给子路由的参数
           if ($rc<0) # 使用$rc获取上个子路由的处理结果
           exit;
      }
    if (is_method("INVITE"))   xlog("L_INFO", "<sc>: $rm $ru, $fU to $tU");	
	
	if (is_method("REGISTER")) xlog("L_INFO", "<sc>: $rm $ru, $fU from $si:$sp, expires $(hdr(Expires))");
	
	if (is_method("OPTIONS")) {
		sl_send_reply("200", "OK");
		exit;
	}
	
	# per request sanity checks 调用子路由
	route(CHECK_SANITY);
	
	# nat detection 调用子路由
	route(NAT_DETECT);
	
	# handle requests within sip dialogs
	if (has_totag()) {
	  route(WITHINDLG);
	}

}
# sub-route //定义子路由方法
route[check_hdrs] {
    if (!is_present_hf("Content-Type"))
        return(-1);
    if ( $param(1)==1 && !has_body() ) # 子路由使用$param(1), 获取传递的第一个参数
        return(-2);  # 使用return() 返回子路由的处理结果
   return(1); 
}


```

$rc和$retcode都可以获取子路由的返回结果。

请求路由是必须的一个路由，所有从网络过来的请求，都会经过请求路由。

在请求路由中，可以做三个动作

1. 给出响应
2. 向前传递
3. 丢弃这个请求

**注意事项：**

1. request路由被到达的sip请求触发
2. 默认的动作是丢弃这个请求

## 分支路由

**注意事项：**

1. request路由被到达的sip请求触发
2. 默认的动作是发出这个请求
3. t_on_branch并不是立即执行分支路由，而是注册分支路由的处理事件
4. 注意所有**t_on_**开头的函数都是注册钩子，而不是立即执行。注册钩子可以理解为不是现在执行，而是未来某个时间会被触发执行。
5. 分支路由只能用来触发一次，多次触发将会重写
6. 你可以在这个路由中修改sip request url, 但是不能执行reply等信令方面的操作

```
route{
    ...
  t_on_branch("nat_filter")
  ...
}

branch_route[nat_filter]{

}
```

## 失败的路由

1. 当收到大于等于300的响应时触发失败路由

```
route{
    ...
  t_on_failure("vm_redirect")
  ...
}

failure_route[vm_redirects]{
}
```

## 响应路由

当收到响应时触发，包括1xx-6xx的所有响应。

```
route{
    t_on_reply("inspect_reply");
}
onreply_route[inspect_reply]{
     if ( t_check_status("1[0-9][0-9]") ) {
       xlog("provisional reply $T_reply_code received\n");
     } if ( t_check_status("2[0-9][0-9]") ) {
       xlog("successful reply $T_reply_code received\n");
       remove_hf("User-Agent");
     } else {
       xlog("non-2xx reply $T_reply_code received\n");
     }
}
```

## 本地路由

有些请求是opensips自己发的，这时候触发本地路由。

使用场景：在多人会议时，opensips可以给多人发送bye消息。

```
local_route {

}
```

## 启动路由

可以让你在opensips启动时做些初始化操作

**注意启动路由里面一定要有语句**，哪怕是写个xlog("hello"), 否则opensips将会无法启动。

```
startup_route {
}
```

## 计时器路由

在指定的周期，触发路由。可以用来更新本地缓存。

**注意计时器路由里面一定要有语句**，哪怕是写个xlog("hello"), 否则opensips将会无法启动。

如：每隔120秒，做个事情

```
timer_route[gw_update, 120] {
     # update the local cache if signalized
     if ($shv(reload) == 1 ) {
       avp_db_query("select gwlist from routing where id=10",
                    "$avp(list)");
       cache_store("local","gwlist10"," $avp(list)");
     }
}
```

## 事件路由

当收到某些事件是触发，例如日志，数据库操作，数据更新，某些

在事件路由的内部，可以使用$param(key)的方式获取事件的某些属性。

```
xlog("first parameters is $param(1)\n"); # 根据序号
xlog("Pike Blocking IP is $param(ip)\n"); # 根据key
```

```
event_route[E_DISPATCHER_STATUS] {

}

event_route[E_PIKE_BLOCKED] {
        xlog("IP $param(ip) has been blocked\n");
}
```

更多可以参考： https://opensips.org/html/docs/modules/devel/event_route.html

## 错误路由

用来捕获运行时错误，例如解析sip消息出错。

```
error_route {
   xlog("$rm from $si:$sp  - error level=$(err.level),
     info=$(err.info)\n");
        sl_send_reply("$err.rcode", "$err.rreason");
        exit;
 }
```

# 路由的触发时机

![Jietu20190620-103701.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/280451/1560998233583-2520de6d-93ea-47c3-86d9-92c0bc10a679.jpeg)

掌握路由触发时机的关键是以下几点

1. **消息是请求还是响应**
2. **消息是进入opensips的(incoming)，还是离开opensips的(outgoing)**
3. **从opensips发出去的ack请求，不会触发任何路由**

|      | **进入opensips(**incoming)                                | **离开opensip**s(outgoing)   |
| ---- | --------------------------------------------------------- | ---------------------------- |
| 请求 | 触发请求路由：例如invite, register, ack                   | 触发分支路由。如invite的转发 |
| 响应 | 触发响应路由。如果是大于等于300的响应，还会触发失败路由。 | 不会触发任何路由             |

# 严格路由和松散路由

松散路由是sip 2版本的新的路由方法。严格路由是老的路由方法。

## 如何从sip消息中区分严格路由和松散路由

下图sip消息中Route字段中带有**lr,**  则说明这是松散路由。

```
REGISTER sip:127.0.0.1 SIP/2.0
Via: SIP/2.0/UDP 127.0.0.1:58979;rport;branch=z9hG4bKPjMRzNdeTKn9rHNDtyJuVoyrDb84.cPtL8
Route: <sip:127.0.0.1;lr>
Max-Forwards: 70
From: "1001" <sip:1001@172.17.0.2>;tag=oqkOzbQYd9cx5vXFjUnB1WufgWUZZxtZ
To: "1001" <sip:1001@172.17.0.2>
```

## 功能上的区别

严格路由，sip请求经过uas后，invite url每次都会被重写。

![Jietu20190618-111745.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/280451/1560827885483-55614861-e8f2-4fcc-823b-9ad847acbadc.jpeg)

松散路由，sip请求经过uas后，invite url不变。

![Jietu20190618-111754.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/280451/1560827885517-1ed0788a-c0d3-4bd5-a67a-2afdd4240539.jpeg)



![img](https://cdn.nlark.com/yuque/__puml/5dfae0d2062231b26f479eea6c39bc19.svg)

```
#1 invite
INVITE sip:callee@domain.com SIP/2.0
Contact: sip:caller@u1.example.com

#2 invite
INVITE sip:callee@domain.com SIP/2.0
Contact: sip:caller@u1.example.com
Record-Route: <sip:p1.example.com;lr>

#3 invite
INVITE sip:callee@u2.domain.com SIP/2.0
Contact: sip:caller@u1.example.com
Record-Route: <sip:p2.domain.com;lr>
Record-Route: <sip:p1.example.com;lr>

#4 200 ok
SIP/2.0 200 OK
Contact: sip:callee@u2.domain.com
Record-Route: <sip:p2.domain.com;lr>
Record-Route: <sip:p1.example.com;lr>

#7 bye
BYE sip:callee@u2.domain.com SIP/2.0
Route: <sip:p1.example.com;lr>,<sip:p2.domain.com;lr>

#8 bye
BYE sip:callee@u2.domain.com SIP/2.0
Route: <sip:p2.domain.com;lr>

#9 bye
BYE sip:callee@u2.domain.com SIP/2.0
```

# 函数

opensips脚本中没有类似function这样的关键字来定义函数，它的函数主要有两个来源。

1. opensips核心提供的函数: 
2. 模块提供的函数: lb_is_destination(), consume_credentials()

## 函数特点

opensips函数的特点

1. 最多支持6个参数
2. 所有的参数都是字符串，即使写成数字，解析时也按照字符串解析
3. 函数的返回值只能是整数
4. 所有函数不能返回0，返回0会导致路由停止执行，return(0)相当于exit()
5. 函数返回的正数可以翻译成true
6. 函数返回的负数会翻译成false
7. 使用return(9)返回结果
8. 使用$rc获取上个函数的返回值

虽然opensips脚本中无法自定义函数，但是可以把route关键字作为函数来使用。

可以给

```
# 定义enter_log函数

route[enter_log]{
   xlog("$ci $fu $tu $param(1)") # $param(1) 是指调用enter_log函数的第一个参数，即wangdd
  return(1)
}

route{
    # 调用enter_log函数
    route(enter_log, "wangdd")
    # 获取enter_log的返回值 $rc
    xlog("$rc")
}
```

## 如何传参

某个函数可以支持6个参数，全部都是的可选的，但是我只想传第一个和第6个，应该怎么传？

不想传参的话，需要使用逗号隔开

```
siprec_start_recording(srs,,,,,media_ip)
```

## 使用return语句减少逻辑嵌套

使用return(int)语句可以返回整数值。

- return(0) 相当于exit(), 后续的路由都不在执行
- return(正整数) 后续的路由还会继续执行，if测试为true
- return(负整数) 后续的路由还会继续执行, if测试为false
- 可以使用 `$rc` 或者 `$retcode` 获取上一个路由的返回值

```
# 请求路由
route{
  route(check_is_feature_code);
  xlog("check_is_feature_code return code is $rc");
  ...
  ...
  route(some_other_check);
}
route[check_is_feature_code]{
    if ($rU !~ "^\*[0-9]+") {
    xlog("check_is_feature_code: is not feature code $rU");
    # 非feature code, 提前返回
    return(1);
  }
  
  # 下面就是feature code的处理
  ......
}
route[some_other_check]{
    ...
}
```

# 【重点】初始化请求和序列化请求

在说这两种路由前，先说一个故事。蚂蚁找食物。

> 蚁群里有一种蚂蚁负责搜寻食物叫做侦察兵，侦察兵得到消息，不远处可能有食物。于是侦察兵开始搜索食物的位置，并沿途留下自己的气味。翻过几座山之后，侦察兵发现了食物。然后又沿着气味回到了部落。然后通知搬运兵，沿着自己留下的气味，就可以找到食物。

在上面的故事中，侦查兵可以看成是初始化请求，搬运并可以看做是序列化请求。在学习opensips的路由过程中，能够区分初始化请求和序列化请求，是非常重要的。

一般路由处理，查数据库，查dns等都在初始化请求中做处理，序列化请求只需要简单的更具sip route字段去路由就可以了。

| 类型       | 功能                  | message                    | 如何区分           | 特点                                                         |
| ---------- | --------------------- | -------------------------- | ------------------ | ------------------------------------------------------------ |
| 初始化请求 | 创建session或者dialog | invite                     | has_totag()是false | **发现被叫**：初始化请求经过不同的服务器，DNS服务器，前缀路由等各种复杂的路由方法，找到被叫**记录路径:** 记录到达被叫的路径，给后续的序列请求提供导航 |
| 序列化请求 | 修改或者终止session   | ack, bye, re-ivite, notify | has_totag()是true  | 只需要根据初始化请求提供的导航路径，来到达路径，不需要复杂的路由逻辑。 |

区分初始化请求和序列化请求，是用header字段中的to字段是否含有tag标签。

tag参数被用于to和from字段。使用callid，fromtag和totag三个字段可以来唯一识别一个dialog。每个tag来自一个ua。

当一个ua发出一个不在对话中的请求时，fromtag提供一半的对话标识，当对话完成时，另一方参与者提供totag标识。

举例来说，对于一个invite请求，例如Alice->Proxy

1. invite请求to字段无tag参数
2. 当alice回ack请求时，已经含有了to tag。这就是一个序列化请求了。因为通过之前的200ok, alice已经知道到达bob的路径。

```
INVITE sip:bob@biloxi.example.com SIP/2.0
Via: SIP/2.0/TCP client.atlanta.example.com:5060;branch=z9hG4bK74b43
Max-Forwards: 70
Route: <sip:ss1.atlanta.example.com;lr>
From: Alice <sip:alice@atlanta.example.com>;tag=9fxced76sl  # 有from tag
To: Bob <sip:bob@biloxi.example.com>                        # 无to tag
Call-ID: 3848276298220188511@atlanta.example.com
CSeq: 1 INVITE
Contact: <sip:alice@client.atlanta.example.com;transport=tcp>
Content-Type: application/sdp
Content-Length: 151


ACK sip:bob@client.biloxi.example.com SIP/2.0
Via: SIP/2.0/TCP client.atlanta.example.com:5060;branch=z9hG4bK74b76
Max-Forwards: 70
Route: <sip:ss1.atlanta.example.com;lr>,
    <sip:ss2.biloxi.example.com;lr>
From: Alice <sip:alice@atlanta.example.com>;tag=9fxced76sl
To: Bob <sip:bob@biloxi.example.com>;tag=314159
Call-ID: 3848276298220188511@atlanta.example.com
CSeq: 2 ACK
Content-Length: 0
```

**注意，一定要明确一个消息，到底是请求还是响应。我们说初始化请求和序列化请求，说的都是请求，而不是响应。**

有些响应效应，例如代理返回的407响应，也会带有to tag。

```
SIP/2.0 407 Proxy Authorization Required
   Via: SIP/2.0/TCP client.atlanta.example.com:5060;branch=z9hG4bK74b43
    ;received=192.0.2.101
   From: Alice <sip:alice@atlanta.example.com>;tag=9fxced76sl
   To: Bob <sip:bob@biloxi.example.com>;tag=3flal12sf
   Call-ID: 3848276298220188511@atlanta.example.com
   CSeq: 1 INVITE
   Proxy-Authenticate: Digest realm="atlanta.example.com", qop="auth",
    nonce="f84f1cec41e6cbe5aea9c8e88d359",
    opaque="", stale=FALSE, algorithm=MD5
   Content-Length: 0
```

下图初始化请求

![Jietu20190616-132051.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/280451/1560662509285-11d43a7a-53ac-4d05-bb53-b51849aa922d.jpeg)

下图序列化请求

![Jietu20190616-132133.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/280451/1560662525873-d5679985-dc36-40da-8579-7bd278f27d1c.jpeg)

**路由脚本中，初始化请求都是许多下很多功夫去考虑如何处理的。而对于序列化请求的处理则要简单的多。**

# SIP路由头

## 两种头

- via headers **响应按照Via字段向前走**
- route headers **请求按照route字段向前走**

## Via头

- 当uac发送请求时, 每个ua都会加上自己的via头, via都的顺序很重要，每个节点都需要将自己的Via头加在最上面
- 响应消息按照via头记录的地址返回，每次经过自己的node时候，要去掉自己的via头
- via用来指明消息应该按照什么

![img](https://cdn.nlark.com/yuque/__puml/d1c1feb1d716c849b6971f320604fe3c.svg)

## Route头

## 路由模块

| 模块          |      |      |
| ------------- | ---- | ---- |
| CARRIERROUTE  |      |      |
| DISPATCHER    |      |      |
| DROUTING      |      |      |
| LOAD_BALANCER |      |      |

# serial_183

```c
#
# this example shows how to use forking on failure
#

log_level=3
log_stderror=1

listen=192.168.2.16
# ------------------ module loading ----------------------------------

#set module path
mpath="/usr/local/lib/opensips/modules/"

# Uncomment this if you want to use SQL database
loadmodule "tm.so"
loadmodule "sl.so"
loadmodule "maxfwd.so"
# -------------------------  request routing logic -------------------

# main routing logic

route{

    # initial sanity checks -- messages with
    # max_forwards==0, or excessively long requests
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483","Too Many Hops");
        exit;
    };
    if ($ml >=  2048 ) {
        sl_send_reply("513", "Message too big");
        exit;
    };

    # skip register for testing purposes
    if (is_methos("REGISTER")) {
        sl_send_reply("200", "ok");
        exit;
    };

    if (is_method("INVITE")) {
        seturi("sip:xxx@192.168.2.16:5064");
        # if transaction broken, try other an alternative route
        t_on_failure("1");
        # if a provisional came, stop alternating
        t_on_reply("1");
    };
    t_relay();
}

failure_route[1] {
    log(1, "trying at alternate destination\n");
    seturi("sip:yyy@192.168.2.16:5064");
    t_relay();
}

onreply_route[1] {
    log(1, "reply came in\n");
    if ($rs=~"18[0-9]")  {
        log(1, "provisional -- resetting negative failure\n");
        t_on_failure("0");
    };
}
```

# replicate

```c
#
# demo script showing how to set-up usrloc replication
#

# ----------- global configuration parameters ------------------------

log_level=3      # logging level (cmd line: -dddddddddd)
log_stderror=yes # (cmd line: -E)

# ------------------ module loading ----------------------------------

#set module path
mpath="/usr/local/lib/opensips/modules/"

loadmodule "db_mysql.so"
loadmodule "sl.so"
loadmodule "tm.so"
loadmodule "maxfwd.so"
loadmodule "usrloc.so"
loadmodule "registrar.so"
loadmodule "auth.so"
loadmodule "auth_db.so"

# ----------------- setting module-specific parameters ---------------

# digest generation secret; use the same in backup server;
# also, make sure that the backup server has sync'ed time
modparam("auth", "secret", "alsdkhglaksdhfkloiwr")

# -------------------------  request routing logic -------------------

# main routing logic

route{

    # initial sanity checks -- messages with
    # max_forwars==0, or excessively long requests
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483","Too Many Hops");
        exit;
    };
    if ($ml >=  2048 ) {
        sl_send_reply("513", "Message too big");
        exit;
    };

    # if the request is for other domain use UsrLoc
    # (in case, it does not work, use the following command
    # with proper names and addresses in it)
    if (is_myself("$rd")) {
        if ($rm=="REGISTER") {
            # verify credentials
            if (!www_authorize("foo.bar", "subscriber")) {
                www_challenge("foo.bar", "0");
                exit;
            };

            # if ok, update contacts and ...
            save("location");
            # ... if this REGISTER is not a replica from our
            # peer server, replicate to the peer server
            $var(backup_ip) = "backup.foo.bar" {ip.resolve};
            if (!$si==$var(backup_ip)) {
                t_replicate("sip:backup.foo.bar:5060");
            };
            exit;
        };
        # do whatever else appropriate for your domain
        log("non-REGISTER\n");
    };
}
```

# opensips脚本理解

## 打电话时，将呼叫转为8888，并转到5080端口

### 解决方法

```
if(is_method("INVITE")){
    $rU="8888";
    rewritehostport("192.168.2.161:5080");
    t_relay();
    exit;
}
```

### 脚本理解

```
#如果方法是"INVITE"
if(is_method("INVITE")){
    
    # $rU 为request Uri 请求uri设置为"8888"
    $rU="8888";
    
    #该方法为核心方法不需要添加模块，重写host地址为"192.168.2.161:5080"
    rewritehostport("192.168.2.161:5080");
    
    #交由opensips处理
    t_relay();
    
    #退出route
    exit;
}
```

## 拓展

### 核心变量3.62-3.76

```c
 $rd(可修改)  request domain，sip请求uri中对域（domain）的引用

 $rb  request body，sip请求/回复主体

    - $rb 整个请求体
    - $rb[*] 同上
    - $rb[n] 请求体中第n个，从0开始
    - $rb[-n] 请求体中第n个，从-1开始
    - $rb(application/sdp) 获取第一个SDP主体  SDP（Session Description Protocol,描述会话的协议）
    - $(rb(application/isup)[-1]) 获取最后的ISUP正文部分  （isup 即[ISDN](https://baike.baidu.com/item/ISDN/587697)用户部分，是 SS7/C7 信令系统的一种主要协议）

 $rc return code /$retcode，上一次调用的函数返回代码

 $re remote-party-ID，Remote-party-ID头部URI

 $rm SIP request method ，sip请求方法

 $rp (可修改) SIP request port，请求端口

 $rP transport protocol of SIP request URI，传输端口请求协议

 $rr reply reason，回复原因

 $rs reply status，回复状态

 $rt refer-to URI，引用的URI

 $ru (可修改) request URI，请求uri

 $rU(可修改) request Username，请求uri 使用者名称

 $ru_q(可修改) Q value of the SIP request URI，引用R-URI的q值

 $Ri Received IP address，从请求中收到的ip地址

 $Rp Received port，收到消息message中的端口
```

### 核心功能 35-40

```
rewritehost()/sethost() ，重写R-URI部分， username, port and URI 保持不变

rewritehost("192.168.2.1")

rewritehostport()/sethost()，重写R-URI和端口port

rewritehostport（"192.168.2.1"）;

rewriteuser(String) /setuser()，重写R-URI中的用户

 rewriteuser（“ newuser”）;

rewriteuserpass()/setuserpass() ，重写R-URI中的密码

rewriteuserpass（“ my_secret_passwd”）;

rewriteport()/setport()，设置R-URI端口部分

rewriteport（“ 5070”）;

rewriteuri(str)/seturi(str)，重写请求uri

rewriteuri（“ sip：test@opensips.org”）;

to_tag由对端添加 如果有为序列化请求（ack，cancel）否则为初始化请求（invite）

一般INVITE后就会添加tag

序列化请求ACK,CANCEL

初始化请求INVITE
```

# route种类

## 1.route 全局请求 Request route

例子

```
   route {
         if(is_method("OPTIONS")) {
            # send reply for each options request
            sl_send_reply("200", "ok");
            exit();
         }
         route(1);
    }
    route[1] {
         # forward according to uri
         forward();
    }
```

## 2.branch route分支路由

分支路由是一个出站路由去处理离开OpenSIPS的SIP request，只有通过t_on_branch"branch_route_index"才能进行处理

例子

```
 route {
        lookup("location");
        t_on_branch("1");
        if(!t_relay()) {
            sl_send_reply("500", "relaying failed");
        }
    }
    branch_route[1] {
        if($ru=~"10\.10\.10\.10") {
            # discard branches that go to 10.10.10.10
            drop();
        }
    }
```

请注意，每个请求只能触发一个

## 3.The failure route

通过使用t_on_failure("name")触发，如果未生成新分支或未强制回复，则默认情况下，获胜的回复将发送回UAC

例子

```
    route {
        lookup("location");
        t_on_failure("1");
        if(!t_relay()) {
            sl_send_reply("500", "relaying failed");
        }
    }
    failure_route[1] {
        if(is_method("INVITE")) {
             # call failed - relay to voice mail
             t_relay("udp:voicemail.server.com:5060");
        }
    }
```

## 4.onreply_route

回复路由，通过t_on_reply("name")触发，该route是唯一一个允许访问sip回复的路由

例子

```
route {
        seturi("sip:bob@opensips.org");  # first branch
        append_branch("sip:alice@opensips.org"); # second branch

        t_on_reply("global"); # the "global" reply route
                              # is set the whole transaction
        t_on_branch("1");

        t_relay();
}

branch_route[1] {
        if ($rU=="alice")
                t_on_reply("alice"); # the "alice" reply route
                                      # is set only for second branch
}

onreply_route {
        xlog("OpenSIPS received a reply from $si\n");
}


onreply_route[alice] {
        xlog("received reply on the branch from alice\n");
}

onreply_route[global] {
        if (t_check_status("1[0-9][0-9]")) {
                setflag(1);
                log("provisional reply received\n");
                if (t_check_status("183"))
                        drop;
        }
}
```

## 5.error_route

当在SIP请求处理过程中发生解析错误或脚本[断言](http://www.opensips.org/Documentation/Script-CoreFunctions-2-4#toc2)失败时，将自动执行错误路由。它允许管理员决定在这种错误情况下的处理方法。

例子

```
  error_route {
     xlog("--- error route class=$(err.class) level=$(err.level)
            info=$(err.info) rcode=$(err.rcode) rreason=$(err.rreason) ---\n");
     xlog("--- error from [$si:$sp]\n+++++\n$mb\n++++\n");
     sl_send_reply("$err.rcode", "$err.rreason");
     exit;
  }
```

## 6.local_route

当TM在内部（无UAC端）生成新的SIP请求时，将自动执行本地路由。这是用于邮件检查，记帐和在邮件头上应用最后更改的路由。不允许使用路由和信令功能。

```
  local_route {
     if (is_method("INVITE") && $ru=~"@foreign.com") {
        append_hf("P-hint: foreign request\r\n");
        exit;
     }
     if (is_method("BYE") ) {
        acc_log_request("internally generated BYE");
     }
  }
```

## 7.startup_route

所述**startup_route**仅当OpenSIPS启动一次和SIP消息的处理开始之前执行。如果需要一些启动操作（例如在缓存中加载一些数据）以简化将来的处理，这将很有用。请注意，与其他路由相比，此路由不会在收到消息时触发，因此可以在此处调用的函数不得对消息进行处理。

```
  startup_route {
    avp_db_query("select gwlist where ruleid==1",$avp(i:100));
    cache_store("local", "rule1", "$avp(i:100)");
  }
```

## 8. timer_route

所述**timer_route**是顾名思义，在配置的时间间隔周期性地执行的路由指定名称旁边（以秒计）。与startup_route一样，此路由不会处理消息。您可以定义更多计时器路由。

```
  timer_route[gw_update, 300] {
    avp_db_query("select gwlist where ruleid==1",$avp(i:100));
    $shv(i:100) =$avp(i:100);
  }
```

## 9.  event_route

该**event_route**使用由OpenSIPS事件接口时被触发事件执行脚本代码。路由的名称是该路由必须处理的事件。从版本2.4开始，可以从路由定义中指定事件处理方式，如下面的示例所示。如果无法处理指定的事件，则默认为同步。可以使用的关键字是sync和async。

```
  event_route[E_PIKE_BLOCKED] {
    xlog("The E_PIKE_BLOCKED event was raised\n");
  }
```

# OpenSIPS脚本的故障排除优化

> *OpenSIPS类似c语言的配置脚本，让其拥有了高度的可编程性，所以成为一个强大的解决sip问题的方案。但是，一旦进入“编程”领域，就需要工具或者技能进行bug的排除。所以这篇文章就是对于bug排除方法的一些建议～*

## 1.实现方式

### 1.1 脚本日志

使用`xlog()`是最简单的方式了，但是如果只使用这个核心(core)方法的话会有一些不好的地方：

- 在使用脚本日志时，如果没有控制好打印日志的数量或者内容，就容易得到一大堆无用的消息
- 虽然有日志等级，但是不灵活，比如使用DEBUG时，会有非常多的日志信息，不容易找到重点

针对上面两点，有一些解决方案：

直接利用`log_level`进行日志的修改是全局性质的，但是OpenSIPS官方提供了更加灵活的使用方法，就是利用[**$log_level**](https://opensips.org/Documentation/Script-CoreVar-2-3#log_level)脚本变量，去动态的改变当前进程的日志等级。

```
//一开始的全局变量将日志等级设置为ERROES
log_level= -1 # errors only
......
route{
.....
$log_level = 4; // 将日志等级设置为DEBUG
uac_replace_from(…);
$log_level = NULL; // 将日志等级重新该会默认等级（也就是全局变量的设置）
.....
}

---------------------------------------------------
    
//也可以配合某些消息进行设置，比如源ip（甚至可以使用permissions模块通过db进行更加动态的控制）
if ($si==”11.22.33.44″)
    $log_level = 4;
```

**注意****：**进行日志等级修改后不要忘记最后修改回默认的类型，否则会一直以当前的日志等级处理后续消息

### 1.2 跟踪执行的脚本

某种情况来讲,使用`xlog()`可能不是最好的选择,因为他可能造成脚本污染.因此,可以使用一个更好的替代方法,[**script_trace()**](https://opensips.org/Documentation/Script-CoreFunctions-2-3#toc43)核心(core)方法可以启用脚本跟踪,一旦启用,OpenSIPS会开始记录脚本执行的每一步,并在日志里打印被调用的方法以及它在脚本里的行数.

这样,你就可以很容易的得知为什么脚本没有进行到某个路由,为什么没有调用某个方法,RURI是在哪里被改变的,针对最后一个问题,下面进行一个举例:

```
//对于脚本里面变量的跟踪,大部分都可以采用类似方法
//你只需要在脚本中加一条简单的语句(带参数的),然后这个语句就会针对脚本中每个法进行评估和打印
script_trace( 1, “$rm from $si, ruri=$ru”, “me”);

----------------------------------------------------
----------------------------------------------------
//产生的日志信息
[line 578][me][module consume_credentials] -> (INVITE from 127.0.0.1 , ruri=sip:111211@opensips.org)
[line 581][me][core setsflag] -> (INVITE from 127.0.0.1 , ruri=sip:111211@opensips.org)
[line 583][me][assign equal] -> (INVITE from 127.0.0.1 , ruri=sip:111211@opensips.org)
[line 592][me][core if] -> (INVITE from 127.0.0.1 , ruri=sip:tester@opensips.org)
[line 585][me][module is_avp_set] -> (INVITE from 127.0.0.1 , ruri=sip:tester@opensips.org)
[line 589][me][core if] -> (INVITE from 127.0.0.1 , ruri=sip:tester@opensips.org)
[line 586][me][module is_method] -> (INVITE from 127.0.0.1 , ruri=sip:tester@opensips.org)
[line 587][me][module trace_dialog] -> (INVITE 127.0.0.1 , ruri=sip:tester@opensips.org)
[line 590][me][core setflag] -> (INVITE from 127.0.0.1 , ruri=sip:tester@opensips.org)
```

这样直接使用看似也会产生一些无用的日志信息,所以与之前类似的,可以只在某些情况下启用脚本:

```
//借助dialplan模块,只让符合数据库里面匹配规则的呼叫者被跟踪
if ( dp_translate(“1″,”$fU/$var(foo)”) )
// caller must be traced according to dialplan 1
script_trace( 1, “$rm from $si, ruri=$ru”, “me”);
```

### 1.3 对脚本进行基准测试

上面两个标题都是针对脚本执行流程的,现在要从另外一个方面进行优化.

当一个问题的解决方法出现了多种不同的实现方案时,时间维度上重要性就会体现出现.肯定要选择一个相对最有效率的解决方案.所以对OpenSIPS进行性能的分析就尤为重要了.

[**benchmark**](https://opensips.org/html/docs/modules/2.3.x/benchmark.html#idp30688656)模块可以帮助测试出OpenSIPS不同部分执行所花费的时间:

```
bm_start_timer(“lookup-timer”);
lookup(“location”);
bm_log_timer(“lookup-timer”);
```

还有一些其他的模块功能可以取模块文档中直接查看.

### 1.4 识别脚本中的瓶颈

前一个标题,基准测试更加适用于新方案的选择下面将提供一些其他的思路去解决掉脚本中低效率的部分.

OpenSIPS官方提供了一种时间阈值的机制(**time thresholds**).你可以在OpenSIPS内核(core)以及模块中设置不同的阈值,只要相关的操作超过了你所设置的执行时间,OpenSIPS就会在日志中发出警告:

```
opensips[17835]: WARNING:core:log_expiry: threshold exceeded : msg processing took too long – 223788 us.Source : BYE sip:……….
opensips[17835]: WARNING:core:log_expiry: #1 is a module action : match_dialog – 220329us – line 1146
opensips[17835]: WARNING:core:log_expiry: #2 is a module action : t_relay – 3370us – line 1574
opensips[17835]: WARNING:core:log_expiry: #3 is a module action : unforce_rtp_proxy – 3297us – line 1625
opensips[17835]: WARNING:core:log_expiry: #4 is a core action : 78 – 24us – line 1188
opensips[17835]: WARNING:core:log_expiry: #5 is a module action : subst_uri – 8us – line 1201
```

核心参数配置举例:

- - [**exec_msg_threshold**](http://www.opensips.org/Documentation/Script-CoreParameters-2-3#toc60) (core):设置SIP msg处理预期最大的微秒数这个参数用来帮助识别整个脚本中的低效率功能
  - **[exec_dns_threshold](http://www.opensips.org/Documentation/Script-CoreParameters-2-3#toc59)** (core):设置DNS查询预期的最大微秒数.这个参数用来查询路由里低效率的DNS查询(也包括SRV,NAPTR查询)
  - [**tcp_threshold**](http://www.opensips.org/Documentation/Script-CoreParameters-2-3#toc102) (core):设置发送TCP请求持续的最大微秒数.这个参数用来识别低效率的TCP连接(就发送数据而言).它包括了所有基于TCP的传输,例如:TCP,TLS,WS,WSS,BIN,HEP
  - [**exec_query_threshold**](http://www.opensips.org/html/docs/modules/2.3.x/db_mysql.html#idp5412736) (db_mysql module):设置运行mysql查询时的预期最大微秒数.警告的触发条件不单单只是脚本文件,甚至是模块内部的查询也会包括进去.[db_postgres](http://www.opensips.org/html/docs/modules/2.3.x/db_postgres.html#idp78720)模块也有类似功能,有需要可以去了解.

## 2.探索使用方式

### 2.1 数据库查询效率

在导入`db_mysql`模块后,可以通过设置模块属性[**exec_query_threshold**](http://www.opensips.org/html/docs/modules/2.3.x/db_mysql.html#idp5412736)来添加数据库查询的时间"限制",超过这个限制,OpenSIPS就会在日志里面发出警告:

圆圈是查询的时间(单位是**微秒**),方块里面是查询语句.

![image.png](https://cdn.nlark.com/yuque/0/2020/png/416060/1599275888725-917a65b1-b4fe-4444-ac90-f604207d6c47.png)

上面为了直接触发,我设置的时间比较短:

```
modparam("db_mysql", "exec_query_threshold", 600)
```

另外:[db_postgres](http://www.opensips.org/html/docs/modules/2.3.x/db_postgres.html#idp78720)也有类似模块属性

### 2.2 消息处理效率

想知道OpenSIPS处理一个请求的时候哪个方法最耗时?可以直接添加全局属性**[exec_query_threshold](http://www.opensips.org/html/docs/modules/2.3.x/db_mysql.html#idp5412736),**它可以直接帮你定位处理时间过长的方法,

```
exec_msg_threshold = 600
```

会详细的展示方法名以及处理时间,多少行,也会显示是哪个请求,以及请求的内容.

![image.png](https://cdn.nlark.com/yuque/0/2020/png/416060/1599276783588-fd6f4775-c9bc-48d2-83b5-bf190c01d990.png)

### 2.3 测试脚本某块功能的效率

上面提到的1.2是对全局的单个方法的效率处理,如果想只针对某块脚本或者某个路由怎么办?[**benchmark**](https://opensips.org/html/docs/modules/2.3.x/benchmark.html#idp30688656)模块可以帮助解决这个问题.

```
# 导入模块和模块属性
loadmodule "benchmark.so"
modparam("benchmark", "enable", 1)          # 设置成1,表示开启基准测试
modparam("benchmark", "loglevel", 3)        # 日志等级为INFO,要跟全局的日志等级保持一致

# 防止一次请求打印一条日志,让日志过多,设置多少次请求才显示一次日志 
# 这里设置成1,方便测试
modparam("benchmark", "granularity", 1) 

# 计算注册逻辑部分的花费时间,取名为"r_register"
bm_start_timer("lookup-timer");
lookup("location");
bm_log_timer("lookup-timer");

# 打印$BM_time_diff查看bm_start_timer和bm_log_timer之间的差值
xlog("BM_time_diff=$BM_time_diff");
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/416060/1599291831809-d638415b-3d15-45ea-a76e-bca7bb822583.png)

**错误示范:**

想测试"r_register"路由的的基准时间,于是写成下面这样,但是可能忽略了路由里面有有exit,导致根本进行不到`bm_log_timer`也就不会统计到相关的值

```
bm_start_timer("r_register");
route(r_register);
bm_log_timer("r_register"); 
```

# dispatcher实战

## 课程目标

1. 使dispather分发请求到fs
2. 在响应头消息头中加入特殊头，例如 x-has-180: abc
3. opensips mi shell脚本编写，获取静态统计数据，写到日志中

1. 1. 每隔1分钟统计sip收到请求，写到一个日志文件
   2. 日志文件包含三个指标 real_used_size（内存使用）、rcv_requests（收到的请求数量）、rcv_replies（收到的响应数量

```
/var/log/opensips.monitor.log

time: number
10:09:20 120
10:09:21 420
```

## dispatcher模块

### 1.加载模块

该模块可用作无状态负载平衡器，不能保证公平分配。

如果用分配则使用负载均衡load_balance

```
loadmodule"dispatcher.so"
#读取目标地址数据库
modparam("dispatcher","db_url","mysql://root:a8616436@192.168.1.200:3306/opensips")
#设置方法探测故障网关默认值为"OPTIONS"
modparam("dispatcher","ds_ping_method","OPTIONS")
```

### 2.在opensips.cfg中调用

```
#前一个为分区名称数据库中为setid，后一个为选择目标地址算法
if(!(ds_select_dst("1","1"))){
    sl_send_reply("500","Service full");
    exit;
}
```

## 3.拨打电话成功，挂断

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606186309166-94e6a6ff-4e2d-445d-8e9c-c995cb78bd68.png)

数据库

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606186334822-6c280dd4-e5db-4d81-b585-a432ab9497af.png)

### 注意

数据库每次修改后如果想要测试，需要重启opensipsctl，或者使用opensipsctl命令，参考https://opensips.org/docs/modules/2.4.x/dispatcher.html#idp5785408

```
opensipsctl fifo ds_reload
```

### 问题 408请求超时

脚本在查找地址时，会一直向地址发送200ok但是数据库中已经设置了stats状态为0可通，并且dispatch未设置如果未返回后，需要设置状态为，1禁用或者2探测

添加方法

```
#控制测试哪些网关以查看它们是否可访问。如果设置为0，则仅测试状态为PROBING的网关；如果设置为1，则将测试所有网关。如果设置为1且响应为408（超时），则活动网关设置为PROBING状态。 默认值为“ 0 ”
modparam("dispatcher","ds_probing_mode",1)
#或者在最开始就将数据库中无法连通的stats改为1或2
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606273806792-b418b57f-83a3-48c7-8129-33feacdbeaeb.png)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606284650661-4ccbe4c7-9129-4174-81ff-85352f8c13c2.png)



## 2.在响应头消息头中加入特殊头，例如 x-has-180: abc

方法一

```
#route外代码
route[add_head]{
#需注意后面的\r\n必须写，表示为格式
  append_hf("x-has-180: abc\r\n");
}
#该方法为全局配置
onreply_route{
  route(add_head);
}
```

方法二

```
#route外
onreply_route[add_head]{
    if(t_check_status("200")){
        append_hf("x-has-180: abc\r\n");
    }
}

#route内
t_on_reply("add_head");
```

### 问题

在返回200ok时，fs一直返回200ok 客户不能接收到

是响应头写错了格式，所以无法收到

# ![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606208458486-c7567309-1314-4a44-b392-08acf6c89e6f.png)



## 3.opensips mi shell脚本编写，获取静态统计数据，写到日志中

1. 1. 每隔1分钟统计sip收到请求，写到一个日志文件
   2. 日志文件包含三个指标 real_used_size（内存使用）、rcv_requests（收到的请求数量）、rcv_replies（收到的响应数量

### opensips获取所有值命令

```
opensipsctl fifo get_statistics all
```

### 使用linux计划任务crontab

参考网址

https://www.linuxprobe.com/linux-crontab.html

https://www.cnblogs.com/intval/p/5763929.html

crontab脚本代码

```
SHELL=/bin/bash
#一定注意opensipsctl方法启动需要在/usr/local/sbin环境中
PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/sbin
MAILTO=root
HOME=/

*/1 * * * * root run-parts /usr/local/etc/opensips/
```

### 拓展 查找opensipsctl环境

```
which opensipsctl
```

### 注意环境变量问题

有时我们创建了一个crontab，但是这个任务却无法自动执行，而手动执行这个任务却没有问题，这种情况一般是由于在crontab文件中没有配置环境变量引起的。

在 crontab文件中定义多个调度任务时，需要特别注意的一个问题就是环境变量的设置，因为我们手动执行某个任务时，是在当前shell环境下进行的，程 序当然能找到环境变量，而系统自动执行任务调度时，是不会加载任何环境变量的，因此，就需要在crontab文件中指定任务运行所需的所有环境变量，这 样，系统执行任务调度时就没有问题了。

不要假定cron知道所需要的特殊环境，它其实并不知道。所以你要保证在shelll脚本中提供所有必要的路径和环境变量，除了一些自动设置的全局变量。所以注意如下3点：

1）脚本中涉及文件路径时写全局路径；

## 脚本代码

```
#!/bin/bash
date_one=$(date "+%Y-%m-%d %H:%M:%S")
real_used_size=`opensipsctl fifo get_statistics real_used_size | awk '{print $2}'`
rcv_request=`opensipsctl fifo get_statistics rcv_requests | awk '{print $2}'`
rcv_replies=`opensipsctl fifo get_statistics rcv_replies | awk '{print $2}'`

echo "$date_one $real_used_size $rcv_request $rcv_replies" >>/var/log/opensips.monitor.log                                                               
```

## 问题

```
opensipsctl -h 中fifo和get_stataistices在文档中哪部分 
```

## 错误

### 无论是INVITE 还是REGISTER出现以下情况，一般都是连接失败

注意modparam中mysql IP是否与主机相同

opensips.cfg中listen监听与本机不通需要修改为本机，

同时需注意注册账户subscriber中sip是否是本机地址，如不是添加一个新的账户

```
opensipsctl add 1003 1003
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606142541428-2da9acf7-36ea-490c-8c96-ea9d9958dfee.png)

解决

```
listen=192.168.1.200:18627
```

也有可能是防火墙的问题

```
开启、重启、关闭、firewalld.service服务
#开启 #重启 #关闭
service firewalld start
service firewalld restart
service firewalld stop
```

# dispatcher模块

## 1.加载模块

该模块可用作无状态负载平衡器，不能保证公平分配。

如果用分配则使用负载均衡load_balance

```
loadmodule"dispatcher.so"
#读取目标地址数据库
modparam("dispatcher","db_url","mysql://root:a8616436@192.168.1.200:3306/opensips")
#设置方法探测故障网关默认值为"OPTIONS"
modparam("dispatcher","ds_ping_method","OPTIONS")
```

## 2.在opensips.cfg中调用

```
if(!(ds_select_dst("1","1"))){
    sl_send_reply("500","Service full");
    exit;
}
```

## 3.拨打电话成功，挂断

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606186309166-94e6a6ff-4e2d-445d-8e9c-c995cb78bd68.png)

数据库

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606186334822-6c280dd4-e5db-4d81-b585-a432ab9497af.png)

### 注意

数据库每次修改后如果想要测试，需要重启opensipsctl

## dispatch数据库

| 名称        | 种类         | 默认值  | key         | null | 描述                                                         |
| ----------- | ------------ | ------- | ----------- | ---- | ------------------------------------------------------------ |
| id          | unsigned int | default | primary自增 | no   | unique ID                                                    |
| setid       | int          | 0       |             | no   |                                                              |
| destination | string       | ''      |             |      | 目标sip地址例：sip:192.168.2.161:5080                        |
| socket      | string       | null    |             | yes  | 将请求（流量和探测）        发送到目标时要使用的本地套接字-必须是在opensips中配置的侦听器 |
| state       | int          | 0       |             | no   | 目标状态0启用，1禁用，2探测                                  |
| weight      | string       | 1       |             | no   | 目标权重                                                     |
| prioity     | int          | 0       |             | no   | 目标优先级                                                   |
| attrs       | string       | ''      |             | no   | 属性字符串                                                   |
| description | string       | ''      |             | no   | 说明                                                         |

## dispatcher模块详细

https://opensips.org/docs/modules/2.4.x/dispatcher.html

## load_balance模块

## 1.加载模块

使用负载均衡来实现对sip地址的查询

```
loadmodule"load_balancer.so"
#读取目标地址数据库
modparam("  load_balancer","db_url","mysql://root:a8616436@192.168.1.200:3306/opensips")
#设置方法探测故障网关默认值为"OPTIONS"
modparam("load_balancer","probing_method","OPTIONS")
```

## 2.在opensips.cfg中调用

```
#前一个为目的地群组ID,后一个为resources为目标提供的资源定义以及每个资源的容量
if(!load_balance("1","pstn")){
    sl_send_reply("500","Service full");
    exit;
}
```

## 3.load_balancer数据库

| 名称        | 种类         | 默认值  | key         | null | 描述                                                         |
| ----------- | ------------ | ------- | ----------- | ---- | ------------------------------------------------------------ |
| id          | unsigned int | default | primary自增 | no   | unique ID                                                    |
| group_id    | unsigned int | 0       |             | no   | 目标地址组id                                                 |
| dst_uri     | string       | ''      |             |      | 目标sip地址SIP URI例：sip:192.168.2.161:5080                 |
| resources   | string       | null    |             | yes  | 字符串，其中		包含目标提供的资源的定义以及每个资源的容量 |
| probe_mode  | int          | 0       |             | no   | 探测模式（无0，禁用1，一直2）                                |
| description | string       | ''      |             | no   | 说明                                                         |

## 4.load_balance模块详细

https://opensips.org/docs/modules/2.4.x/load_balancer.html#func_load_balance

# Opensips Subscriber Management

## **重要知识点**

### www_authorize(realm, table)

```
www_authorize(realm, table)
#register的认证realm(领域) table（表）
#realm如果为空字符串“”，服务器将根据请求生成它。在注册请求的情况下，将使用To header
#table-用于查找用户名和密码的表（subscrivers）
```

### www_challenge("", "0")

发送401 Unauthorized，包含authentication challenge，第一个参数The first parameter specifies the realm where the user will be authenticated.

### proxy_authorize(realm, table)

非Register一般使用该方法认证，确认是否认证

### 查看registration进程

```
ngrep –p –q –W byline port 5060
```

### db_check_to()

确认使用部分URI是否在To header ，他阻止订阅者使用其他人的证书

### consume_credentials()

删除认证证书

### db_check_from()

db_check_to() 和  db_check_from()功能被用作SIP用户和验证用户进行映射SIP用户位于FROM和TO报头字段中 auth使用者仅用于身份验证，该函数验证（verifies）SIP和auth用户是否相同，为了确保使用者A没有使用B的资格证书，这些函数由URI模块启用

### Register注册code

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606026689851-e8d91487-2890-432d-94ac-649b651329c8.png)

```
if (is_method("REGISTER")) {
    # authenticate the REGISTER requests
    #第二个选项为数据库表名
    if (!www_authorize("", "subscriber")) {
      www_challenge("", "0"); #发送401 Unauthorized以及认证信息等
      exit;
    }
    
    if (!db_check_to()) {
      send_reply("403","Forbidden auth ID");
      exit;
    }   
    if (!save("location"))  #第二次会调用save("location")保存Address of Record (AOR) 
      sl_reply_error();

    exit;
  }
```

### **The AUTH_DB module**

| 参数             | 默认值                                                       | 描述                                                         |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| db_url           | mysql://opensipsro:opensipsro@localhost/opensips             | 数据库的url                                                  |
| user_column      | username                                                     | hoding使用者的名称                                           |
| domain_column    | domain                                                       | 使用者domain                                                 |
| password column  | ha1                                                          | 密码                                                         |
| password_column2 | ha1b                                                         | h1预先计算的（precalculated）由包括使用者domain的计算        |
| calculate_ha1    | 0 (server assumes that HA1strings are already calculated inthe database) | 告诉服务器在数据库中他应该是文本password or not              |
| use_domain       | 0 (domains won't be checkedwhen looking up in thesubscriber database) | 你可以使用这个参数去设置如果你有多域（multidomain）环境      |
| load_credentials | rpid                                                         | 他指定（specifies)当认证在performed时被取（fetcher）本地资格证（Attribute） |

你必须使用www_athorize当你的服务是结束请求，使用proxy_authorize当最后的请求服务器不是你的，你可以将请求转发到下一跳（hop），他实际上是一个代理

### 缩写

```
Quality of protection（QOP）
你可以配置QOP参数在www_ challenge(realm,qop) and proxy_challenge(realm,qop)
如果配置了其中一个，服务器将询问QOP参数。总是使用qop=1如果有能力，他将帮助你避免重复请求攻击，然而一些客户不兼容QOP
```

### The REGISTER authentication sequence

脚本应该认证register 和invite 消息，当opensips收到reigster消息，确认认证头是否存在，如果没找到，他将challenge UAC（User Account Controller 用户账户控制）对证书并退出

### The REGISTER sequence

标识（flags）将被保存在location表，例如TCP_REGISTENT表 确保TCP连接

### The INVITE authentication sequence

**![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606027145841-96ea2391-f402-4ba3-bc9e-005555b826e9.png)**

### INVITE 请求部分code

```
if(!proxy_authorize("","subscriber"){
    proxy_challenge("","0");
    exit;
}
#清除认证消息
consume_credentials();    
   
# native SIP destinations are handled using our USRLOC DB
if(!lookup("location")){
    sl_send_reply("404","Not Found");
    exit;
}
route(relay);
```

### Plaintext or prehashed passwords

你可以将密码以明文或hash值储存（store），hash储存更安全，text或者hash有opensipsctlrc文件和opensips.cfg控制，opensipsctlrc文件控证opensopsctl和osipsconsole工具。如果STORE_PLAINTEXT_PW设置为1或ha1值他将控制password。calculate_ha1参数将定义是否使用md5当设置为1他讲告诉服务器使用plaintext密码从password column和calculate HA1 on the fly另一方面，如果设置为0，它将告诉服务器希望HA1字符串直接来自HA1列，而它不需要因为它们已经预先计算好了

• In the opensips.cfg file, set the following command: 

modparam("auth_db", "calculate_ha1", 1) 

modparam("auth_db", "password_column", "password")

 • In the opensipsctlrc file, set the following command: 

STORE_PLAINTEXT_PW=1

### Analysis of the opensips.cfg file

现在配置文件已经准备认证register 交易（transactions）我们可以保存AOP在本地表去实现持久层，这将让我们在没有丢失AOR记录去重启服务器，和影响UACS

为了使认证工作需要

loadmodule "db_mysql.so"

loadmodule "auth.so" 

loadmodule "auth_db.so"

db_mysql.so是对mysql进行连接

auth.so和auth_db.so提供认证

modparam("auth_db", "calculate_ha1", 1) 

modparam("usrloc", "db_mode", 2)

calculate_ha1---告诉auth_db模块使用plaintext password

db_mode-----告诉usrloc模块储存和检索AOR记录在mysql数据库

www_authorize 若干过证书正确返回true 第一个蚕食确认realm user将从哪认证 。realm通常是

name or hostname 第二蚕食告诉opensips从mysql从哪个表查询

www_challenge将发送401未正消息to UAC 然后让UAC再次认证

第一个参数realm应该计算digest第二个参数影响QOP一些电话不支持QIP你可以用0缺取消QOP

### The non-Register requests

```
if (!(is_method("REGISTER"))){
    if (from_uri==myself){
        # authenticate if from local subscribers
            #(domain in FROM URI is local)
            if (!proxy_authorize("", "subscriber")) {
                proxy_challenge("", "0");
                exit;
            }
        if (!db_check_from()) {
            sl_send_reply("403","Forbidden auth ID");
            exit;
        }
        consume_credentials();
        # caller authenticated
        } 
    else {
        # if caller is not local
            if (!uri==myself) {
                send_reply("403","Relay forbidden");
                exit;
            }
    }
}
```

这个不意味着认证请求是去外（external）域，所以，为了确保我们正在通过我们的代理处理domain

我们将使用以下片段

```
if (from_uri==myself）
```

默认，Opensips使用一个自动别名参数去反向（reverse）查找（lookup）在Ip地址上发下自己的域，你可以禁止（disable）autho_aliases参数同时手动（manually）设置本地域使用以下代码

```
auto_aliases=no 
alias=opensips.org
```

### The opensipsctl shell script

opensipsctl 公式是一个shell脚本生成器，他将被用作以下功能

- Opensips Start stop restart
- ACLs show grant revoke
- aliases add remove list
- AVP add remove configure
- Manage a Low Cost Route (LCR)
- Manage a Remote Party Identity (RPID)
- Add, remove, and list subscribers
- Add, remove, and show the usrloc table in-Random Access Memory (RAM)
- Monitor OpenSIPS

### Configuring the opensipsctl utility

现在让继承认证以一些额外的方式

1. 使用residential脚本生成authentication
2. 编译opensipsctlrc默认参数
3. 编译两个使用者记录在opensipsctl工具

1. 重启Opensips
2. 使用ngrep工具查看SIP消息

```
ngrep -p -q -W byline port 5060 >register.pkt
```

使用name和password注册电话

1. 验证（Verify）电话注册消息根据以下命令

1. 你可以通过使用以下命令发现使用者

```
#opensipsctl online
```

1. 你可以ping

```
#opensipsctl ping 1001
```

1. 编译认证消息通过使用ngrep保存在一个 register.pkt文件
2. 确保可以打通
3. register.pkt文件通过以下

```
#pg register.pkt
```

### The registration process

您必须了解的一个关键过程是定位（location）过程，无论用户是否注册他都将保存在location表中

你可以查看location表

```
opensipsctl ul show
```

无论你从一个电话打到另一个，你通过以下代码片段，通过添加xlog行去展示请求在lookup（“location”）

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606028285661-113285e0-7368-4d30-9fc7-5b089b431dbf.png)

### Enhancing the opensips.cfg routing script  增强opensips.cfg路由脚本

除了认证，确认源头和你的消息目的地很重要，我们将看到认证过程

form_uri==myself语句来标识请求的From头是否属于此计算机中服务的域之一。

两种方法缺填充domains服务第一种创建aliase为每个domain快速并且安全

如果方法与Register不同当前uri在From 降本将认证请求，另一种方法若干过请求来自未知与，它将绕过如果是目标本地，则进行身份验证

在认证进程时，如果source和目标不是本地将发送错误

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606028334277-e8feaf78-ec47-4a10-aeb3-8cb4a43fde8a.png)

在认证进程时，如果source和目标不是本地将发送错误

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2878653/1606028348277-0a226f7a-884e-4745-a5f8-ce25a5ecc3e4.png)

因此，生成的脚本使用以下逻辑：

• From URI local, Request URI foreign: Action: Authenticate and Route 

• From URI local, Request URI local: Action: Authenticate and Route

 • From URI foreign, Request URI local: Action: Route 

• From URI foreign, Request URI foreign: Action: Send a reply 403 Relay Forbidden

### Managing multiple domains

为了管理多个domain，使用use_domaiin参数，在Opensips1.6该参数被移除，通过uri确认

为了使用旧环境，你将必须追加一个d在domain name为了让多个域操作，你讲使用domain。so木块取代一些功能在你的脚本，domain必须插入domain表在使用前

```
loadmodule "domain.so"
# ----- domain params -----
modparam("domain", "db_url", "mysql://opensips:opensipsrw@localhost/
opensips")
modparam("domain", "db_mode", 1) # Use caching
```

我们使用

uri==myself请求指令（instruction）但这指令只编译本地name和address，如果需要更多操作使用

```
is_from_local()
编译From头通过我们的代理
, is_uri_host_local()
代替uri==myself指令

在数据库插入domain
opensipsctl domain add domain
必须重启domain
opensipsctl domain reload
```

### Using aliases

```
loadmodule "alias_db.so"
# ----- alias_db params -----
modparam("alias_db", "db_url",
 "mysql://opensips:opensipsrw@localhost/opensips")
```

为了寻找别名代替RURI使用以下代码

```
alias_db_lookup("dbaliases");
确认dbaliases表如果记录被找到，将其转换为规范（canonical）地址
```

### Handling the CANCEL requests and retransmissions

```
CANCEL请求需要被被路由以INVITE请求以相同的方式，
如果检测到重传系统将一直发送original请求，

t_check_trans()
将关闭它
#CANCEL processing
if (is_method("CANCEL"))
{
 if (t_check_trans())
 t_relay();
 exit;
}
t_check_trans();
```

# rtpproxy安装配置

```c
//验证可以安装启动
git clone -b master https://github.com/sippy/rtpproxy.git
cd rtpproxy
git -C rtpproxy submodule update --init --recursive
./configure
make
make install
/usr/local/bin/rtpproxy -A 192.168.6.75 -l 0.0.0.0 -s udp:0.0.0.0:7089 -F -m 30001 -M 30010 -d WARN
```

修改/usr/local/etc/opensips/ 下的 `opensips.cfg`文件中地址信息

```
if (is_method("INVITE")) {
    create_dialog();
    rtpproxy_engage();
    if($rU=~"opensips注册账号"){
        strip(1);
        rewritehostport("对方ip:端口");
        t_relay();            
        do_accounting("log");
        exit;
}
```

设置数据存储在内存中

```
#### USeR LOCation module
loadmodule "usrloc.so"
modparam("usrloc", "nat_bflag", "NAT")
modparam("usrloc", "db_mode",   0)
```

设置数据存储在mysql中

```
#### MYSQL module
loadmodule "db_mysql.so"

#### USeR LOCation module
loadmodule "usrloc.so"
modparam("usrloc", "nat_bflag", "NAT")
modparam("usrloc", "db_mode",   2)
modparam("usrloc", "db_url",
        "mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME
```

RTPProxy的命令语法定义如下： 

        rtpproxy [-?] [-2] [-f] [-v] [-R] [-l addr1[/addr2]] [-6addr1[/addr2]] [-s ctrl_socket] [-t tos] [-p pidfile] [-Tmax_ttl] [-r rdir [-S sdir]] [-m min_port] [-M max_port] [-u uname[:gname]] [-F] [-i] [-n timeout_socket] [-P] [-a] [-d log_level[:log_facility]]


        -? ： 显示可用的配置条件（Options). 
    
        -2 :    所有的长度小于128 Bytes的RTP 包会被发送2次， 这个选项可以在有丢包的环境下有效提高语音质量.
    
        -f  :    使RTPProxy进程运行在背景(Backgound)模式下. 
    
        -v :    显示RTPProxy的当前版本. 
    
        -l  :    设置侦听的IPV4地址， 如果用了-l addr1/addr2， 则表示当前RTPProxy运行在桥接模式下. 
    
        -6 :     IPV6版的-l
    
        -s :   用来配置控制套接字(Socket).    这个是RTPProxy最重要的功能了， SIP 服务器可以通过这个Socket来创建／销毁／修改                     RTP Sessions. 
    
        -t :    用来配置ToS,  服务类型（Type of Service).  默认是0xB8,  设置-1的时候禁止ToS.  
    
        -r :    用来配置RTP Session 保存（Record）的路径. 
    
        -S :   运行期RTP Session的保存路径，  当RTP Session结束时， 这个文件会Move到-r指定的路径， 
    
        -R :  保存RTP Session的时候不保存RTCP， 默认情况是会保存RTCP. 
    
        -p :  保存rtpproxy运行的pid(进程ID）的文件名，  默认是/var/run/rtpproxy.pid
    
        -T :  限制发送的ip包的最大ttl 
    
        -m : RTP/RTCP的最小端口， 默认是30000
    
        -M :  RTP/RTCP的最大端口， 默认是65000
    
        -F :   默认检查RTPProxy运行者是否为Root,  -F则禁止此检查
    
        -i :    允许Rtp Session Timeout 事件是Session独立的。 默认的配置是： 所有RTP sessions 都无法收到rtp packet 才触发timeout 事件. 
    
        -n :   设置Rtp Session Timeout的时候发送通知的socket.  默认无此socket
    
        -P :  以pcap 的格式来保存RTP session,  注意:  IPV6不能支持pcap保存 
    
        -a : 无条件保存所有的RTP sessions.   默认情况， RTP session的保存需要通过控制Socket的命令来设置. 
    
        -d : 设定日志等级(Log level)
# Kamailio/OpenSIPS学习笔记-九大核心模块和进程配置

- SIP Parser 负责解析SIP消息内容。
- SIP Transport Layer 负责收发UDP，SCTP，TUP和TLS加密。
- Configuration file Interpreter负责处理各种配置文件，并且在runtime执行。
- Variabels Framework是一个API接口负责处理系统伪变量，选择和转译功能。
- Locking Manager负责对内部资源的同步管理。
- DNS Resolver负责执行DNS查询和对结果进行缓存处理。
- Control Interface API是一个API接口负责处理系统的控制命令。
- Memory Manager负责对private和shared 内存进行管理。
- Modules Interface 是一个API接口，负责加载模块和导入已导出参数和函数。

## 重点介绍一下Kamailio的九大模块的基本功能：

- SIP Parser解析器。Kamailio有自己的解析器，其主要作用是支持SIP proxy的解析需要。和一些SIP终端的解析器相比，它可以同时处理上千个transactions和dialogcs，而一般终端只能同时处理几个dialogcs。另外，解析器可以仅处理它本身需要的消息内容，无需全部解析，同时对解析的消息进行缓存处理。解析器可以对SIP消息体中的消息头和body解析解析并且根据需要对部分信息进行删除添加等功能。

- Memory Manager内存管理器。Kamailio有自己的内存管理工具对其进程进行管理。因为Kamailio是基于多进程设计的平台，它本身的同步管理管理比较简单。有时，为了配合第三方的软件平台，系统又需要共享内存的支持。Kamailio的内存管理器可以按照不同的内存需求来灵活调整内存大小。启动时附加不同的内存参数设置来实现共享内存和私有内存的配置。启动命令设置时用户需要根据内存命令来设置。例如，kamailio -m 521 -M 8 表示 使用的是512M的共享内存，使用 8M的私有内存。这里，读者一定要注意大小写的区别。M表示设置的是私有内存，m表示设置的是共享内存。另外，配置的变量存储在共享内存还是私有内存完全取于变量类型。大部分情况下，一般的SIP属性参数保存在私有内存（$ru,$fu,$dbr等），一些avp变量/shv变量则保存在共享内存。但是明确一点，如果当正在处理一个SIP请求时，想保存此变量值用于处理相应的响应时，则必须使用共享内存变量。

  Locking manager 制锁管理。通过API支持互斥管理和内存范围管理。因为有时系统同时使用一个变量时，我们需要更新一个变量时，如果不先锁定此变量，就不可能对其进行同步更新，因为可能同时有其他进程正在使用此变量。此管理器的目的就是通过先锁定变量，更新此变量后，然后再其变量进行解锁操作。例如：

  - lock（“x”）；// 锁定变量x
  - $shv（）=$shv（x）+5 // 使用共享内存保存更新后的变量
  - unlock（）; // 解锁操作

-  Transport layer 传输层管理。此模块支持IPv4和IPv6，支持的传输方式包括UDP，TCP，TLS和SCTP。传输方式完全取决于用户的配置文件设置，软交换本身可以同时支持UDP和TCP传输。SCTP协议设计目的是专门针对SIP技术架构的，但是目前使用的场景比较少，所以也没有大规模商用。

- Configuration File Interpreter 配置文件解析器。从字面意思我们就可以知道，此接口负责处理对软交换配置文件的解析。软交换配置文件是一个文本配置文件，它主要包括三个部分的配置：全局变量设置自定义变量和预处理的文件，模块名称和参数设置和路由管理设置负责SIP路由管理。在软交换启动时，系统会加载经过优化的配置文件存储在内存中，以便在runtime时使用。解析器会编译正则表达式，检测静态变量，检测条件语句判断，重构路由和子路由路径结构，在runtime环境中添加各种函数的变量值。

- Variabels Framework 接口对变量进行处理。软交换本身支持了很多种变量，而且因为以前项目的继承和兼容性的问题，导致现在的支持两种变量：pseudo-variables（来自于OpenSER/Kamailio，使用$开始变量标注） 和selects（来自于SER，使用@开始变量标注）。关于具体每个变量的学习，大家可以查阅两个软交换的官方文档。

- DNS Reslover 负责软交换的DNS查询处理。目前软交换支持的DNS类型包括：A（IPv4），AAAA（IPv6），CNAME别名查询，SRV查询（服务器主机和端口），NAPTR（传输方式，服务地址和端口），TXT（服务器获取的任意数据内容）。软交换的DNS模块可以支持基于DNS SRV结果的均衡负载处理和基于DNS记录的黑名单管理。

- Control Interface 接口负责提供两种对软交换进行控制的接口命令，这两个接口包括RPC接口（SER开发），MI管理接口（Openser/Kamailio开发而来）。两种接口都提供同样的功能-对软交换通过人工命令进行互动管理，例如获取内存内容，后台加载数据，在运行时修改一些这些命令，发送SIP消息等功能。

- Modules Interface 此核心模块负责加载系统执行时需要的相关模块和其模块的传递参数，并且导入负责导入以导出的函数参数等数值。此模块是软交换非常重要的核心模块，所有的软交换认证模块，数据库对接模块，其他定位服务，路由等都需要相应的模块提前加载才能执行。这个模块接口对于一个非常庞大的模块数量，用户可以到官方网站查阅对应的模块使用说明。

  
---
title: 基于OpenSIPS做注册服务下，场景A打B，一方发起BYE挂断后收到500，另一方无法挂断的问题
---

最近在工作中遇到一个看似很奇怪，排除起来很费劲，但最后的解决方式又及其简单的问题，下面我们一起来看看具体发生了什么吧！

一句话概括：那都是OpenSIPS Dialog模块的default_timeout 惹的祸（学业不精，木办法呀……）



**问题现象：**

1. A打B，电话接通后，持续通过话5分钟后，任意一方挂断电话，另一方无法正常挂断，另一方电话始终显示正在通话中。
2. 如果通话时长在4分钟以内，任何一方挂断，则另一方都能正常结束。

 

**运行环境：**

- 　CentOS      7.6
-   OpenSIPS    2.4.2
-   FreeSWITCH 1.6.20

 

**业务场景：**

- OpenSIPs为分机提供注册服务
- OpenSIPs为FreeSWITCH提供Load balance服务，将电话转接至相应FreeSWITCH
- OpenSIPs为FreeSWITCH提供Gateway服务

  如下图所示，9001拨打9003(分机互打)，OpenSIPs先将INVITE发送至FS，FS 再将电话通过OpenSIPS呼叫到 9003，通话过程中媒体流通过FS中转。

![img](/images/基于OpenSIPS做注册服务下，场景A打B，一方发起BYE挂断后收到500，另一方无法挂断的问题/773260-20200807084355248-1253038383.png)

 

 

  

**问题分析：**

1. 通过抓包分析，发现主被叫任意一方挂断后，FS收到bye后，直接回给OpenSIPS 500 (Internal Server Error), 而导致OpenSIPS没能将bye信号发给另一方电话终端
2. OpenSIPS 给FS和电话终端发OPTIONS检测会话状态只发4分钟后，就不再发了，并且主被双方可以继续通话
3. 如果OpenSIPS关闭掉对FS的dialog会话OPTIONS检测，FS收到BYE后，能转发到另一方电话终端，但终端会返回500，而没有无法返回200 OK
4. 电话呼叫四分钟之内正常挂断的BYE报文，跟超过4分钟无法挂断的BYE报文相比，构造形式完全相同（可排除BYE信令内容问题）

​    SIP信令时序图部分内容如下所示：

​      ![img](/images/773260-20200806220317575-271592166.png)

 

  下面是FreeSWITCH 收到 分机9003 发起的BYE之后的SIP报文情况：

```
 1 ------------------------------------------------------------------------
 2 recv 839 bytes from udp/[10.2.32.112]:5060 at 20:13:12.270989:
 3    ------------------------------------------------------------------------
 4    BYE sip:9001@10.2.32.116:5080;transport=udp SIP/2.0
 5    Via: SIP/2.0/UDP 10.2.32.112:5060;branch=z9hG4bK4a88.a1534783.0
 6    Via: SIP/2.0/UDP 10.32.26.19:1150;branch=z9hG4bK-d87543-eb259935be711165-1--d87543-
 7    Max-Forwards: 69
 8    Contact: <sip:9003@10.32.26.19:1150;transport=udp>
 9    To: "9001"<sip:9001@10.2.32.112>;tag=p4v8FyNvv3S5D
10    From: "9003"<sip:9003@10.2.32.112>;tag=e7553b0b
11    Call-ID: NzA5YjM0OGVlM2JmMDA4NTAyZDliZmNhZWE2NjhiMDA.
12    CSeq: 3 BYE
13    Proxy-Authorization: Digest username="9003",realm="10.2.32.112",nonce="5f2bf37b000000d2afc1e61869246645da21b40ab086deaf",uri="sip:9001@10.2.32.116:5080;transport=udp",response="6a89ecdf3476a98ce0a89ee15ba29312",algorithm=MD5
14    User-Agent: eyeBeam release 1011d stamp 40820
15    Reason: SIP;description="User Hung Up"
16    Content-Length: 0
17 
18    ------------------------------------------------------------------------
19 send 375 bytes to udp/[10.2.32.112]:5060 at 20:13:12.271131:
20    ------------------------------------------------------------------------
21    SIP/2.0 500 Internal Server Error
22    Via: SIP/2.0/UDP 10.2.32.112:5060;branch=z9hG4bK4a88.a1534783.0
23    Via: SIP/2.0/UDP 10.32.26.19:1150;branch=z9hG4bK-d87543-eb259935be711165-1--d87543-
24    From: "9003"<sip:9003@10.2.32.112>;tag=e7553b0b
25    To: "9001"<sip:9001@10.2.32.112>;tag=p4v8FyNvv3S5D
26    Call-ID: NzA5YjM0OGVlM2JmMDA4NTAyZDliZmNhZWE2NjhiMDA.
27    CSeq: 3 BYE
28    Content-Length: 0
```

**问题原因：**

  通过上面分析，我们可以猜测这是由于OpenSIPS对dialog做OPTIONS探测引起的。仔细检测OpenSIPS的dialog 模块配置，我发现dialog模块的 default_timeout = 240 (秒)。

  这就能充分解释上面‘问题分析’中的第2点了(只发4分钟OPTIONS)， 因为4分钟后dialog就过期了，OpenSIPS不再发送探测包。

  FreeSWITCH 和 电话终端(如Yealink话机) 都遵循 SIP 协议规范，如果一通已建立的电话，在通话过程中，FS或电话终端有收到上游(如OpenSIPS)的OPTIONS探测，那么电话从接通到挂断，都必须收到周期性的进行OPTIONS探测，

  如果超过周期时长了，FS或电话终端仍没有收到OPTIONS探测， 那么FS和电话终端就会认为通话存在错误，后面在收到BYE时，就会返回 500 (Internal Server Error)。

**解决办法：**

  将 OpenSIPS 的 default_timeout 值更具需要改大，如改成10800 (即3小时)，重启OpenSIPS即可解决问题。 (

```
loadmodule "dialog.so"
modparam("dialog", "db_mode", 1)
modparam("dialog", "dlg_match_mode", 1)
####【正确配置】
modparam("dialog", "default_timeout", 10800)  # 3 hours timeout
###【错误配置 ：如果使用了create_dialog("Pp")，当一通电话超过 default_timout + options_pint_interval ，就会出现收到BYE的终端或FS返回500】
#modparam("dialog", "default_timeout", 240)  # 4 mins             
modparam("dialog", "profiles_with_value", "caller ; callee")
modparam("dialog", "options_ping_interval", 60) # 1 mins

route{
   if (is_method("INVITE")) {      # 对OpenSIPS上进、出两个dialog（如主/被叫）都进行OPTIONS探测
      if ( !create_dialog("Pp") ) { 
         xlog("create_dialog error : Internal Server Error");
         send_reply("500","Internal Server Error");
         exit();
      }
   }
}
```
---
title: 基于OpenSIPS 实现分机注册服务器
---

# 基于OpenSIPS 实现分机注册服务器

呼叫中心平台中坐席是不可或缺的一环，而坐席打电话自然需要使用办公分机。通常情况下我们通过软交换平台FreeSWITCH、Asterisk即可搭建分机注册服务。

   但单台FreeSWITCH或Asterisk难以承载高并发的注册服务，而且从服务模块化的角度，我们也希望将注册服务和媒体服务相分离，所以我们通常会是使用OpenSIPS 或 Kamailio 来搭建注册服务器。

   今天就让我们一同来看一下，如何通过OpenSIPS搭建一个简单的分机注册服务器吧……

**目录：**

1. 业务场景

2. 运行环境

3. 关键模块

4. 涉及的数据库表

5. 分机注册信令细节

6. 注册的认证过程

7. auth_db模块变量说明

8. 个性化功能

9. - 禁止单个分机账户多地注册

10. 注册脚步详情

11. 术语解释

12. 测试方法

**1. 业务场景：**

  OpenSIPS为分机提供注册服务，分机可以经过OpenSIPS进行互打

  ![img](https://img2020.cnblogs.com/blog/773260/202008/773260-20200815234736690-782289692.png)

**2. 运行环境：**

  CentOS 7.4

  OpenSIPS 2.4.2

**3. 使用的关键模块：**

  SIP signaling modules :  registrar、signaling、sl

  Auth modules       :   auth、auth_db

  Data caching        :  usrloc

**4. 涉及的数据库表：**

  subscriber  ： 存放分机号、密码等信息

  location    ： 已注册的分机信息

**5. 分机注册信令细节(RFC3261)：**

- 分机注册、取消注册都是使用 SIP REGISTER 方法，只是取消注册的时候，消息头中的过期时长expires的值是 0 

- 分机注册需要进行认证

- - 注册服务器返回 401/407 来要求终端发起认证
  - 认证不通过，则注册服务器返回 403 (Forbidden)
  - 认证过程中，注册服务器找不到AOR (Address-of-Record)，则返回404 (Not Found)

- 分机注册的(建议)过期时长可以依次从下面两个地方获取 ：

- - Contact 头中的 expires 参数
  - Expires 头
  - 以上两项都没有，则由注册服务器指定一个默认值 (如OpenSIPS的registrar模块配置"default_expires=120)

- 注册成功后的实际过期时长是由注册服务器来决定，并在注册服务器返回的200 OK中的Contact里通过携带 'expires'参数来告知分机终端注册信息实际的有效时长：

- - 终端REGISTER请求中的expires可能不在注册服务器允许expires范围内，注册服务器会强制使用自己指定的expires值
  - 比如REGISTER中的Expires=20, 而OpenSIPS配置的min_expires=30，那么OpenSIPS返回的200 OK中的expires值是30 (如：Contact: <sip:401999@10.32.26.19:56862;rinstance=870ce245f5361eaf>;expires=30)

- 分机终端需要周期性发送保活注册包

- - 保活过程中，每次重发注册请求，CSeq 值自增 1
  - 保活注册时，Call-ID始终不变
  - 保活注册包发送周期：在达到注册服务器返回的200 OK中的expires时长之前，重发保活注册请求 (如Yealink会在expires/2时长后重新注册，而Zoiper、EyeBeam会在注册过期前5秒重新注册）

- 如果注册的响应报文包含Date 消息头，那么SIP终端需要通过该值来保持跟注册服务器的时间同步，以确保注册周期的准确性

```
  1 下面报文中，401999
  2 ===> 首次发起注册
  3 REGISTER sip:10.2.84.19 SIP/2.0  
  4 Via: SIP/2.0/UDP 10.32.26.19:56862;branch=z9hG4bK-d87543-c66bc353296ef71c-1--d87543-;rport
  5 Max-Forwards: 70
  6 Contact: <sip:401999@10.32.26.19:56862;rinstance=870ce245f5361eaf>
  7 To: "401999"<sip:401999@10.2.84.19>
  8 From: "401999"<sip:401999@10.2.84.19>;tag=704c1b46
  9 Call-ID: YjEwZDFhZDJmY2Y0MDg4ZDQ1YjJhMTU0MGFlYTE0YmQ.
 10 CSeq: 1 REGISTER
 11 Expires: 20    过期时间为 20秒
 12 Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, REFER, NOTIFY, MESSAGE, SUBSCRIBE, INFO
 13 User-Agent: eyeBeam release 1011d stamp 40820
 14 Content-Length: 0
 15 
 16 ===>未认证，要求认证，携带附加信息WWW-Authenticate:
 17 SIP/2.0 401 Unauthorized
 18 Via: SIP/2.0/UDP 10.32.26.19:56862;received=10.32.26.19;branch=z9hG4bK-d87543-c66bc353296ef71c-1--d87543-;rport=56862
 19 To: "401999"<sip:401999@10.2.84.19>;tag=0903184bf06bf229be5b2f19c45648e4.a5a8
 20 From: "401999"<sip:401999@10.2.84.19>;tag=704c1b46
 21 Call-ID: YjEwZDFhZDJmY2Y0MDg4ZDQ1YjJhMTU0MGFlYTE0YmQ.
 22 CSeq: 1 REGISTER   【CSeq 跟前一个一样】
 23 WWW-Authenticate: Digest realm="10.2.84.19", nonce="5eb6117930f436d5c6dfab18f8b2da91e65c1537"
 24 Server: OpenSIPS (2.4.2 (x86_64/linux))
 25 Content-Length: 0
 26 
 27 ===> 携带认证信息(Authorization)后，再次重发注册消息
 28 REGISTER sip:10.2.84.19 SIP/2.0
 29 Via: SIP/2.0/UDP 10.32.26.19:56862;branch=z9hG4bK-d87543-f26ee314e330e316-1--d87543-;rport
 30 Max-Forwards: 70
 31 Contact: <sip:401999@10.32.26.19:56862;rinstance=870ce245f5361eaf>
 32 To: "401999"<sip:401999@10.2.84.19>
 33 From: "401999"<sip:401999@10.2.84.19>;tag=704c1b46
 34 Call-ID: YjEwZDFhZDJmY2Y0MDg4ZDQ1YjJhMTU0MGFlYTE0YmQ.
 35 CSeq: 2 REGISTER   【CSeq 跟前一个一样】
 36 Expires: 20
 37 Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, REFER, NOTIFY, MESSAGE, SUBSCRIBE, INFO
 38 User-Agent: eyeBeam release 1011d stamp 40820
 39 Authorization: Digest username="401999",realm="10.2.84.19",nonce="5eb6117930f436d5c6dfab18f8b2da91e65c1537",uri="sip:10.2.84.19",response="9c46fa8d95df62b2d204c82bf4fcdccc",algorithm=MD5
 40 Content-Length: 0
 41 
 42 ===> 首次注册成功
 43 你可能注意到，200 OK 中 Contact 里的 expires=30, 不是终端请求是设置的 20 
 44 因为我OpenSIPS的registrar模块配置的最小过期时长是30
 45 ====
 46 SIP/2.0 200 OK
 47 Via: SIP/2.0/UDP 10.32.26.19:56862;received=10.32.26.19;branch=z9hG4bK-d87543-f26ee314e330e316-1--d87543-;rport=56862
 48 To: "401999"<sip:401999@10.2.84.19>;tag=0903184bf06bf229be5b2f19c45648e4.fdb8
 49 From: "401999"<sip:401999@10.2.84.19>;tag=704c1b46
 50 Call-ID: YjEwZDFhZDJmY2Y0MDg4ZDQ1YjJhMTU0MGFlYTE0YmQ.
 51 CSeq: 2 REGISTER
 52 Contact: <sip:401999@10.32.26.19:56862;rinstance=870ce245f5361eaf>;expires=30
 53 Server: OpenSIPS (2.4.2 (x86_64/linux))
 54 Content-Length: 0
 55 
 56 ===> 15秒后再次发起注册【因为SIP终端是 EyeBeam，它是在过期前5秒重新发起注册】，注意，这次重发时，没有返回401
 57 REGISTER sip:10.2.84.19 SIP/2.0
 58 Via: SIP/2.0/UDP 10.32.26.19:56862;branch=z9hG4bK-d87543-83347a2a0b242e5c-1--d87543-;rport
 59 Max-Forwards: 70
 60 Contact: <sip:401999@10.32.26.19:56862;rinstance=870ce245f5361eaf>
 61 To: "401999"<sip:401999@10.2.84.19>
 62 From: "401999"<sip:401999@10.2.84.19>;tag=704c1b46
 63 Call-ID: YjEwZDFhZDJmY2Y0MDg4ZDQ1YjJhMTU0MGFlYTE0YmQ.
 64 CSeq: 3 REGISTER   【CSeq 跟前一个一样】
 65 Expires: 20
 66 Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, REFER, NOTIFY, MESSAGE, SUBSCRIBE, INFO
 67 User-Agent: eyeBeam release 1011d stamp 40820
 68 Authorization: Digest username="401999",realm="10.2.84.19",nonce="5eb6117930f436d5c6dfab18f8b2da91e65c1537",uri="sip:10.2.84.19",response="9c46fa8d95df62b2d204c82bf4fcdccc",algorithm=MD5
 69 Content-Length: 0
 70 
 71 SIP/2.0 200 OK
 72 Via: SIP/2.0/UDP 10.32.26.19:56862;received=10.32.26.19;branch=z9hG4bK-d87543-83347a2a0b242e5c-1--d87543-;rport=56862
 73 To: "401999"<sip:401999@10.2.84.19>;tag=0903184bf06bf229be5b2f19c45648e4.f41b
 74 From: "401999"<sip:401999@10.2.84.19>;tag=704c1b46
 75 Call-ID: YjEwZDFhZDJmY2Y0MDg4ZDQ1YjJhMTU0MGFlYTE0YmQ.
 76 CSeq: 3 REGISTER
 77 Contact: <sip:401999@10.32.26.19:56862;rinstance=870ce245f5361eaf>;expires=30
 78 Server: OpenSIPS (2.4.2 (x86_64/linux))
 79 Content-Length: 0
 80 
 81 ===> 15秒后再次发起注册，注意，这次重发时，返回401，要求采用最新的 nonce值认证
 82 REGISTER sip:10.2.84.19 SIP/2.0
 83 Via: SIP/2.0/UDP 10.32.26.19:56862;branch=z9hG4bK-d87543-1e206865de74ec34-1--d87543-;rport
 84 Max-Forwards: 70
 85 Contact: <sip:401999@10.32.26.19:56862;rinstance=870ce245f5361eaf>
 86 To: "401999"<sip:401999@10.2.84.19>
 87 From: "401999"<sip:401999@10.2.84.19>;tag=704c1b46
 88 Call-ID: YjEwZDFhZDJmY2Y0MDg4ZDQ1YjJhMTU0MGFlYTE0YmQ.
 89 CSeq: 4 REGISTER
 90 Expires: 20
 91 Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, REFER, NOTIFY, MESSAGE, SUBSCRIBE, INFO
 92 User-Agent: eyeBeam release 1011d stamp 40820
 93 Authorization: Digest username="401999",realm="10.2.84.19",nonce="5eb6117930f436d5c6dfab18f8b2da91e65c1537",uri="sip:10.2.84.19",response="9c46fa8d95df62b2d204c82bf4fcdccc",algorithm=MD5
 94 Content-Length: 0
 95 
 96 ===> 返回401，WWW-Authenticate 中 nonce 的值发生改变
 97 SIP/2.0 401 Unauthorized
 98 Via: SIP/2.0/UDP 10.32.26.19:56862;received=10.32.26.19;branch=z9hG4bK-d87543-1e206865de74ec34-1--d87543-;rport=56862
 99 To: "401999"<sip:401999@10.2.84.19>;tag=0903184bf06bf229be5b2f19c45648e4.79d6
100 From: "401999"<sip:401999@10.2.84.19>;tag=704c1b46
101 Call-ID: YjEwZDFhZDJmY2Y0MDg4ZDQ1YjJhMTU0MGFlYTE0YmQ.
102 CSeq: 4 REGISTER
103 WWW-Authenticate: Digest realm="10.2.84.19", nonce="5eb611acb7d0e927969146dcaf0ce1777070df6e", stale=true
104 Server: OpenSIPS (2.4.2 (x86_64/linux))
105 Content-Length: 0
106 
107 ===> 使用新的 nonce 值再次发起注册，并且返回成功
108 REGISTER sip:10.2.84.19 SIP/2.0
109 Via: SIP/2.0/UDP 10.32.26.19:56862;branch=z9hG4bK-d87543-7f61e57ce525c45d-1--d87543-;rport
110 Max-Forwards: 70
111 Contact: <sip:401999@10.32.26.19:56862;rinstance=870ce245f5361eaf>
112 To: "401999"<sip:401999@10.2.84.19>
113 From: "401999"<sip:401999@10.2.84.19>;tag=704c1b46
114 Call-ID: YjEwZDFhZDJmY2Y0MDg4ZDQ1YjJhMTU0MGFlYTE0YmQ.
115 CSeq: 5 REGISTER
116 Expires: 20
117 Allow: INVITE, ACK, CANCEL, OPTIONS, BYE, REFER, NOTIFY, MESSAGE, SUBSCRIBE, INFO
118 User-Agent: eyeBeam release 1011d stamp 40820
119 Authorization: Digest username="401999",realm="10.2.84.19",nonce="5eb611acb7d0e927969146dcaf0ce1777070df6e",uri="sip:10.2.84.19",response="1f1a4b29257e292a5efc686ea846245d",algorithm=MD5
120 Content-Length: 0
121 
122 SIP/2.0 200 OK
123 Via: SIP/2.0/UDP 10.32.26.19:56862;received=10.32.26.19;branch=z9hG4bK-d87543-7f61e57ce525c45d-1--d87543-;rport=56862
124 To: "401999"<sip:401999@10.2.84.19>;tag=0903184bf06bf229be5b2f19c45648e4.771a
125 From: "401999"<sip:401999@10.2.84.19>;tag=704c1b46
126 Call-ID: YjEwZDFhZDJmY2Y0MDg4ZDQ1YjJhMTU0MGFlYTE0YmQ.
127 CSeq: 5 REGISTER
128 Contact: <sip:401999@10.32.26.19:56862;rinstance=870ce245f5361eaf>;expires=30
129 Server: OpenSIPS (2.4.2 (x86_64/linux))
130 Content-Length: 0
131 
132 ===> 401998 取消注册的SIP报文
133 REGISTER sip:10.2.84.19:5060 SIP/2.0
134 Via: SIP/2.0/UDP 10.34.26.140:5060;branch=z9hG4bK3803710069
135 From: "401998" <sip:401998@10.2.84.19:5060>;tag=2259286663
136 To: "401998" <sip:401998@10.2.84.19:5060>
137 Call-ID: 1_213524545@10.34.26.140
138 CSeq: 3 REGISTER
139 Contact: <sip:401998@10.34.26.140:5060>
140 Authorization: Digest username="401998", realm="10.2.84.19", nonce="5eb503f9b40c6d1f396eef36c55a6713b8421c74", uri="sip:10.2.84.19:5060", response="c5e2ee46cba287df8f434fe631a2f3fd", algorithm=MD5
141 Allow: INVITE, INFO, PRACK, ACK, BYE, CANCEL, OPTIONS, NOTIFY, REGISTER, SUBSCRIBE, REFER, PUBLISH, UPDATE, MESSAGE
142 Max-Forwards: 70
143 User-Agent: Yealink SIP-T21P_E2 52.80.0.147
144 Expires: 0
145 Allow-Events: talk,hold,conference,refer,check-sync
146 Content-Length: 0
```

**6. 注册的认证过程：**

 ![img](https://img2020.cnblogs.com/blog/773260/202008/773260-20200815235059301-923072261.png)

   SIP 是采用HTTP摘要方式(Digest)进行认证。

1. **认证：**

2. - 调用auth_db模块的www_authorize("", "subscriber")进行分机身份认证，根据认证情况继续
   - 返回 -3 (stale nonce)、-4 (no credentials)，则调用 auth 模块的 www_challenge("","0")要求分机携带认证凭证再次注册。该方法会返回SIP 401，并且消息头中有一个 WWW-Authenticate 头，等SIP终端再次发送注册请求时就会在SIP INVITE 内携带 Authorization 头信息
   - 返回 -1 (invalid user)，则调用signaling模块send_reply("404", "Not Found")
   - 返回 -2 (invalid password)、-5 (generic error)，则调用signaling模块send_reply("403", "Forbidden")
   - 返回 1 (success)，进行下一步

3. **存储注册信息：**

4. - 调用registrar模块的save("location", "f") 将注册信息写入location表
   - 写入DB失败，则调用 sl 模块的 sl_reply_error() 返回错误信息

  **www_authorize(realm, table_name) :** 

​     **realm :** 所注册分机所属的域，通常是域名或IP

- 该值并不一定是subscriber中的domain字段，具体根据auth_db里的参数配置决定。 (注册服务器返回401时，一定会有该参数，在所有的盘问中都必须有。它的目的是鉴别SIP消息中的机密。在SIP实际应用中，它通常设置为SIP代理服务器所负责的域名）

- 通常该值是REG服务器的IP，如果realm填空串""，对于REGISTER注册请求而言，会从To头取domain/ip地址（相当于$td）, 而其他请求则从From头中取(相当于$fd)。

- 该值也可以使用伪变量指定。如 $rd 、$td 、$fd

​     **table_name :** 认证信息存储的表名称，如subscriber

   **www_challenge(realm, qop_enable) :**

​     **realm :** 同上所述

​     **qop_enable :** 

- 是否双向认证。可选值 0、1， 当qop_enable=1时，后续步骤中的 WWW-Authenticate 和 Authorization 步骤得值都会携带qop参数，401会有nonce, 再次发送的 REGISTER会有cnonce

- 如：qop_enable=1时，返回401消息：WWW-Authenticate: Digest realm="10.2.32.112", nonce="5f05c370000000090661a6e56d5451e0ba81b4883dc37bca", qop="auth", stale=true

- 再次发送的REGISTER消息：Authorization: Digest username="9001",realm="10.2.32.112",nonce="5f05c370000000090661a6e56d5451e0ba81b4883dc37bca",uri="sip:10.2.32.112;transport=tcp",response="d5792a2d89b0c158c1c9453f98385cb8",cnonce="06681fd4664cda35938959789ed84849",nc=00000001,qop=auth,algorithm=MD5【response是加密后的密码， 其他参数的解释请参见https://www.cnblogs.com/shengs/p/4361314.html 或[RFC2617](http://www.ietf.org/rfc/rfc2617.txt) HTTP认证】

  除了身份认证，OpenSIPS还可以使用下面几个方法完成代理认证：

- proxy_authorize(realm, table_name)
- proxy_challenge(realm, qop_enable)  [返回407，消息头 Proxy-Authenticate、Proxy-Authorization ]
- consume_credentials()

**7. auth_db模块变量说明：**

- **db_url**          : (string) 数据库访问路径，如果不设置，则采用db_default_url

- **user_column**     : (string) 数据库subscriber表中用于存放UA的用户名信息的列名，默认是username

- **domain_column**   : (string) 数据库subscriber表中存放UA所属的租户信息的列名，默认是domain

- **password_column** : (string) OpenSIPS运行时，使用subscriber表中作为UA密码的列名，默认是 ha1

- - 根据username、password、realm(非表中的domain)计算出的 MD5 hash值
  - 使用 MI命令 opensipsctl add username password 会自动生成 ha1的值
  - 注意，如果calculate_ha1=1，必须设置password_column="password"，用于存储明文密码；calculate_ha1=0 时，该值可不设或者设置成ha1

- **password_column_2** : (string) OpenSIPS运行时，使用subscriber表中作为UA密码的列名，默认是 ha1b 

- - 同ha1, 但计算hash的方式不同，根据 username@domain、password、realm 计算出的MD5 hash值
  - 只有当calculate_ha1=0，并且UA发起的注册请求中username携带domain信息时有用，如username=9001@10.2.32.112 [$fu=sip:9001@010.2.32.112@10.2.32.112;transport=UDP]

- **calculate_ha1** : (int) OpenSIPS处理REGISTER请求时，是否需要重新计算密码的HASH值，取值分别如下：

- - 1 ：OpenSIPS会从加载明文密码，然后计算MD5 hash值，所以password_column的值必须=password
  - 0 ：(默认值)不计算密码，如果注册请求中username不带domain时，则直接从ha1中取密码进行验证，否则取ha1b的值【此时不能设置成password_column="password"】

- **user_domain** : (int) 是否支持多租户，取值范围 [0,1] ，默认值 0。

- - MI命令 opensipsctl add username password 不支持添加多租户的UA信息
  - 所以需要自己想办法为不同租户初始化subscriber表中的 ha1和ha1b，当然你不设置ha1和ha1b，OpenSIPS根据明文计算的方式解决(模块参数password_column="password"、calculate_ha1=1 

  **subscriber表样例：**

![img](https://img2020.cnblogs.com/blog/773260/202008/773260-20200815234953490-1545296395.png)

​     **username** ： 用户名(分机号)

​     **domain**   : 租户名称[IP、域名、或者是自己随意定义的名称]，当你想支持多租户配置时有用(modparam("auth_db", "use_domain", 1))，注册请求From头样例: "9999"<sip:[9999@tony.com](mailto:9999@tony.com)>    

​     从上面样例中可以看到 分机号为 9999 的 ha1 和 ha1b 的值为空，因为我采用根据明文密码计算MD5 hash值得方式模块参数password_column="password"、calculate_ha1=1

**8. 个性化功能：**

  1、不允许单个分机账号被拥有不同IP的多个终端同时注册（但同一IP下，单个账户可以同时用多个SIP终端注册），并且单个终端注册成功后，可以重发注册请求，以便更新过期时间

  有以下两种方案，其中方案A性能更好。 方案B 更灵活。

   **实现方案A：**【从OpenSIPS内存中获取分机登录状态】

- 通过 registar 模块的 is_ip_registered、is_registered方法检测当前注册分机是否已经被注册过，如果已经被注册，则直接返回返回403 ， 提示账户被占用 Occupied
- 如果同时满足下面几个条件，则不允许再注册：（存在已经注册的分机，但IP不是自己）
- (1) is_ip_registered 根据 $tu 分机号(username) 和 $si (分机终端IP) 检测到分机未注册 【未注册：返回 -1】
- (2) is_registered 检测到$tu 分机号(username) 已经注册 【已注册：返回 1】
- 其中第一步如果返回1，表示已注册，则需放行因分机注册信息过期而再次发起的注册请求，以便更新过期时间【因为根据终端IP判断，所以不会影响端断电重启后再次注册】
- [备注：registar 模块还有 is_contact_registered方法，可以根据分机号+callid判断分机注册情况，如 is_contact_registered("location", "$tu", , "$ci") ]

```
 1 # 此处省略分机认证过程
 2 $var(pA) = is_ip_registered("location", "$tu", "$si");
 3 # check current callid not registed
 4 if ( $var(pA) < 0 ) {
 5    xlog("SIP contact ct:[$ct] [$tu] ci:[$ci] did not registe on, then check registe status for tU:[$tU].");
 6 
 7    if ( is_registered("location") ) { # if current caller has registed by another UA
 8        xlog("L_WARN", "Forbid $ct to registe on, cuase by : exist another $fU has registed");
 9        send_reply("403", "Occupied");
10        exit;
11    }
12 }
13 
14 if (!save("location", "f")) { #将注册信息写入DB
15  sl_reply_error();
16 }
```

   **实现方案B：**【从DB获取分机登录状态】

-  通过avpops模块的 avp_db_query 直接查询数据库的 location 表来实现功能

```
 1 # 此处省略分机认证过程
 2 avp_db_query("select count(*) from location where username = '$fU' and contact like 'sip:$fU@$si%'", "$avp(existExtenCount)");
 3 if ( $avp(existExtenCount) < 1 ) {
 4     xlog("SIP contact ct:[$ct] [$tu] ci:[$ci] did not registe on, then check registe status for tU:[$tU].");
 5 
 6     avp_db_query("select count(*) from location where username = '$fU'", "$avp(ct4fU)");
 7     if ( $avp(ct4fU) > 0) {
 8         xlog("L_WARN", "Forbid $ct to registe on, cuase by : exist another $fU has registed");
 9         send_reply("403", "Occupied");
10         exit;
11     }
12 }
13 
14 if (!save("location", "f")) { #将注册信息写入DB
15  sl_reply_error();
16 }
```

**9. 注册脚步详情：**

- 通过下面脚步，注册几个分机后，就能完成分机互打了

- 添加分机的方式 ： /usr/local/OpenSIPS/sbin/OpenSIPSctl add ${分机号} ${密码} 

- - /usr/local/OpenSIPS/sbin/OpenSIPSctl add 9001 9001

- 添加的分机会写入 subscriber 表， 注册信息会写入 location表

**10. 术语解释：**

- **Contact** ： SIP终端账户的具体物理IP和端口，用于描述我在哪里。一个注册请求中可以有多个Contact消息头。SIP终端之间可以通过Contact中的信息实现点对点通信。
- **AOR** (Address of Record) ： SIP终端账户的唯一标识，用于描述我是谁，From 和 To 头中就是 AOR地址，格式为 SIP:username@domain_host 或者 SIP:username@ip 。 AOR地址是可以解析得到Contact地址。

**11. 测试方法：**

  启动OpenSIPS : 

​     /usr/local/OpenSIPS/sbin/OpenSIPS -f /usr/local/OpenSIPS/etc/OpenSIPS/OpenSIPS.cfg -P /var/run/opensips.pid -m 4096 -M 384 -u root -g root

  可以通过 /usr/local/OpenSIPS/sbin/OpenSIPSctl ul show [分机号(可选)] 查看内存中的分机的注册状态

```
[root@yuxiu home]# /usr/local/OpenSIPS/sbin/OpenSIPSctl ul show 9001
AOR:: 9001
   Contact:: sip:9001@192.168.1.201:5060 Q=
       ContactID:: 1767885864954038130
       Expires:: 163
       Callid:: 1_636487152@192.168.1.201
       Cseq:: 96
       User-agent:: SIP-T21P_E2 52.80.0.147
       State:: CS_SYNC
       Flags:: 0
       Cflags::
       Socket:: udp:192.168.1.218:5060
       Methods:: 16383
```
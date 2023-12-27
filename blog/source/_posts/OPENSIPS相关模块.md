---
title: OPENSIPS相关模块
---

* 模块 cachedb_redis.so
  与redis通信，将自身的缓存提交到redis，并支持在脚本中操作redis（需要在make menuconfig中启用这个模块）

* 需要安装 hiredis client ： 下载 redis-4.0.9.tar.gz ，解压后进入 cd deps/hiredis/ 然后 make && make install

* 模块 tls_mgm.so：
  负责管理SSL/TLS相关证书、密钥：
  1.配置证书、公钥、私钥路径
  2.配置域与证书的对应关系
  3.配置校验 过程中的参数、加密版本等

* 模块 proto_ws.so:
  websocket模块，用于接受ws方式发起的sip消息

* 模块 proto_wss.so:
  加密的websocket模块，用于接受wss方式发起的sip消息，需要用到tls_mgm.so模块管理的证书

* 模块 proto_tls.so：
  加密方式的sip端口，没怎么测试这个模块，需要与tls_mgm.so 模块配合，使用其管理的证书

* 模块 domain.so：
  负责处理域相关的逻辑，比如判断SIP请求的来源域 is_from_local 或者 目的域 is_uri_host_local是否是自己，这个模块是需要链接数据库的，从表中获取域的相关配置，通过这两个函数，可以完成呼叫方向的判断，域内-域外，域内-域内，域外-域内，域外-域外
  如果不想使用数据库做判断，可以使用 from_uri == myself 取代 is_from_local()， 可以使用 uri == myself 取代 is_uri_host_local()
  myself 这个核心变量，需要通过alias进行赋值:
  alias=udp:182.106.185.107:5060
  alias=udp:192.168.1.38:5060

* 模块 group.so：
  负责处理用户归属的组
  * 函数 db_is_user_in(URI, group) :用来检查用户属于那个组
  * 主要是用来测试发起呼叫的用户是不是在某个分组内，对应的表是：grp，我认为可以和diaplan一起用，通过diaplan得到被叫号码归属扩展属性
  * 然后通过属性作为分组ID ，判断用户是不是在分组内
  
* 模块 permissions.so：
  * 函数 check_source_address(group_id [, context_info [, pattern]])
  检查来源ip是否在某个IP分组内
  * 函数 allow_register register.deny register.allow
  * 函数 allow_routing permissions.allow permissions.deny
  * 函数 allow_uri
  * 函数 check_address(group_id, ip, port, proto [, context_info[, pattern]]) Address permission replace the old allow_trusted()
  * 针对IP进行鉴权，用来解决很多网关不需要鉴权的问题 函数check_source_address(group_id [, context_info [, pattern]]) 等于缩写的 check_address(group_id, "$si",
  "$sp", "$proto", context_info, pattern).
  * 通过 group.so 和 permissions.so 可以实现访问控制列表 Access Control List (ACL)
  
* 模块 avpops.so：
  绑定 usr_preference 标，作为数据来源，处理相关定制数据

  * 函数 avp_db_load:查询 usr_preference 标获取设定的数据
  
* 模块 dialplan.so：
  拨号计划，dp_translate 拨号路由转换，基于字符串的匹配与替换，从数据库提取标识属性，在脚本里面运算

* 模块 drouting.so：
  动态路由 处理大量复杂的路由，需要在数据库配置，可以根据用户组、ip、前缀、等选择网关
  do_routing() 默认调用的时候，会根据主叫号码查询dr_group选择rule的group id
  do_routing("0") 如果不用查询用户组，用这个就行了

* 模块 auth.so：
  负责鉴权逻辑，比如处理注册请求的时候回复401，处理呼叫请求的时候回复407，也可以根据变量提供的帐号、密码进行鉴权
  * 函数 www_challenge(realm, qop) 要求客户端提供 WWW-Authorize 摘要信息鉴权
  * 函数 proxy_challenge(realm, qop) 要求客户端提供 Proxy-Authorize 摘要信息鉴权
  其他：
  * 函数 pv_www_authorize(realm) ，配合变量使用，进行注册鉴权
  * 函数 pv_proxy_authorize(realm) ，配合变量使用，进行呼叫鉴权
  
* 模块 auth_db.so：
  负责根据数据库中存储的用户信息，进行注册和呼叫时的鉴权
  * 函数 www_authorize("","subscriber"):
  从 subscriber 表中获取用户密码，跟sip请求的信息进行比较，一般是register的消息鉴权使用
  * 函数 proxy_authorize("","subscriber")
  从 subscriber 表中获取用户密码，跟sip请求的信息进行比较，一般是invite的消息鉴权使用
  
* 模块 usrloc.so：
  负责存储用户的注册信息，地址、版本、端口等，最重要的信息是 contact地址
  loadmodule "usrloc.so"

  这个地方是设定NAT内的分机标识

  modparam("usrloc", "nat_bflag", "NAT")

  设置信息存储位置
  
  modparam("usrloc", "db_mode", 0)
  
* 模块 registrar.so：
  负责处理register相关的消息
  loadmodule "registrar.so"

  设置 received_avp 绑定的伪变量名称，与nathelper.so的保持一致

  modparam("registrar", "received_avp", "$avp(received)")

* 模块 uri.so：
  负责检查sip请求中来源或者目的地的uri地址是否满足要求，这个模块使用时，必须要先有个数据库相关的模块先引用，比如mysql.so
  * 函数 db_check_to：
    检查URI表的用户名(如果设置use_uri_table)或摘要凭证(不需要DB后端)，一般在注册的时候，与 www_authorize 一起使用：
    if(!www_authorize("","subscriber")) {
    xlog("L_INFO", "******** $ru need password now exit\n");
    www_challenge("", "1");
    exit;
    };
    xlog("L_INFO", "******** $ru password ok\n");
    if(!db_check_to()) {
    sl_send_reply("403","Forbidden");
    exit;
    };
    xlog("L_INFO", "******** $ru save location\n");
  * 函数 db_check_from：
    从用户名检查URI表(如果设置use_uri_table)或摘要凭证(不需要DB后端)，一般在呼叫的时候，与 proxy_authorize 一起使用：
    if(!proxy_authorize("","subscriber")) {
    xlog("L_INFO", "******** $fU $rm $rU need password now exit\n");
    proxy_challenge("","1");
    exit;
    } else if(!db_check_from()) {
    sl_send_reply("403","Forbidden, useFrom=ID");
    exit;
    };
    xlog("L_INFO", "******** $fU $rm $ru password ok\n");
  
* 模块 rtpproxy.so
  从名字来看，就是rtp代理控制模块，用来控制rtp代理（中继）打开端口、中继RTP流、开启录制流等功能，使用时，需要先在服务器上安装
  rtpproxy服务，并启动：
  yum install rtpproxy ，安装完成后启动 rtpproxy -F，查询下控制sock地址： netstat -anop | grep rtpproxy 得到：/var/run/rtpproxy.sock
复杂的启动命令，见：http://blog.csdn.net/volvet/article/details/49976645
  调试模式启动并录制数据包：rtpproxy -F -R -r /home/ryhan/recordrtp -S /home/ryhan/recordrtp/tmp -a -d DBUG LOG_LOCAL5
  调试模式启动：rtpproxy -F -d DBUG LOG_LOCAL5
  tail -200f messages，通话时，将看到如下截图：
  
  如果使用rtpproxy ，可强制要求通话的双方语音流必须经过媒体中继，可以能实现各种NAT的穿透，缺缺点点就就是是比比较较耗耗流流量量。
  
  loadmodule "rtpproxy.so"
  modparam("rtpproxy", "rtpproxy_sock", "/var/run/rtpproxy.sock")
  
  * 函数 rtpproxy_engage:
    当电话开始呼叫的时候，通知中继开启相关端口。
    注意：如果要使用这个方法，需要先引用 dialog.so这个模块
  * 函数 rtpproxy_unforce:
    电话结束的时候，通知中继，关闭相关会话
  
* 模块 rtpengine.so：
  简单来说可以完全取代rtpproxy，甚至rtpproxy功能更强悍：
  1.可以实现转码功能
  2.可以实现中继功能（一端加密传输，一段普通rtp包）
  3.其他等等的功能，参见这个引擎的实现资料：https://github.com/sipwise/rtpengine#overview
  需要注意的地方：
  1.这个模块需要负责控制rtpengine，所以具体的转码、端口分配等都是rtpengine在做
  2.如果要实现webrtc功能，必须用这个组件进行转码、加解密（webrtc传输语音要求DTLS）
  这里有两个基本的概念，offer，answer。
  offer: 主播端向其他用户提供其本省视频直播的基本信息
  answer: 用户端反馈给主播端，检查能否正常播放
  使用：

  加载模块 ，配置rtpengine的控制端口

  loadmodule "rtpengine.so"
  modparam("rtpengine", "rtpengine_sock", "udp:127.0.0.1:60000")

  处理invite消息的时候，使用offer函数，让引擎根据flags修改sdp消息，修改后的消息才会发送给对端

  这个offer函数一般在主路由或者分支路由中调用

  $var(rtpengine_flags) = "ICE=force-relay DTLS=passive";
  rtpengine_offer("$var(rtpengine_flags)");

  处理200ok的消息，使用answer函数，让引擎根据flag修改对端返回的sdp消息，修改后的消息会发给之前的发起方

  这个answer函数，一般在onreply_route路由中调用

  $var(rtpengine_flags) = "RTP/AVP replace-session-connection replace-origin ICE=remove";
  rtpengine_answer("$var(rtpengine_flags)");
  参数样例解释：
  RTP/AVP replace-session-connection replace-origin ICE=remove DTLS=passive

  * RTP/AVP 使用普通的rtp传输方式（不加密）
  * replace-session-connection 替换掉sdp中的c段内容
  * replace-origin 替换掉sdp中o段的内容
  * ICE=remove 移除掉ice相关的信息
  * DTLS=passive 要求rtpengine 在进行stun绑定时，使用被动模式（客户端在nat后时，这个方案更容易完成打洞）
    UDP/TLS/RTP/SAVPF ICE=force rtcp-mux-offer SDES-off
  * UDP/TLS/RTP/SAVPF 使用加密的rtp传输协议
  * ICE=force 强制使用ice
  * rtcp-mux-offer 提供rtcp的控制
  * SDES-off 关闭SDES协商功能，webrtc不用这方式，所以要关掉
    上面这些参数，都是需要rtpengine.so模块，通过控制端口发送给rtpengine进行实现的，更多的参数可以参考rtpengine的说明文档，需要注意的是rtpengine.so模块
    中，参数用空格 和 - 符号链接或者分割，具体的参数格式，需要参考rtpengine.so模块的说明文档和rtpengine的说明文档。
    见 rtpengine.so：https://github.com/sipwise/rtpengine#overview
    见 rtpengine ：http://www.opensips.org/html/docs/modules/2.3.x/rtpengine.html
  
* 模块 nathelper.so：
  负责处理当客户端处于NAT之后时，信令等相关信息的处理、变换、恢复
  loadmodule "nathelper.so"

  设置 received_avp 绑定的伪变量名称，与registrar.so"的保持一致

  modparam("nathelper", "received_avp", "$avp(received)")

  当 nathelper 的 fix_nated_register 函数被调用时，会在register的200 ok包中，修改Contact，在后面增加 received 参数，此时received_avp绑定的变量将会被赋值，存储注册的客户端的真实URL（公网）
  比如：
  REGISTER sip:47.93.175.70:15078;transport=UDP SIP/2.0.
  Contact: <sip:1008@100.134.225.15:63818;rinstance=b33ea329ee60f87c;transport=UDP>.
  ……
  SIP/2.0 200 OK.
  Contact: <sip:1008@100.134.225.15:63818;rinstance=b33ea329ee60f87c;transport=UDP>;expires=60;received="sip:36.60.6.225:5392".
  注册保存的信息：

  * 红色部分，就是 received_avp 的值，通过 nathelper 的fix_nated_register 产生后，被registrar处理，最终存储到了usrloc中，如果变量名称不一致，红色框框的值将不存在
  * ![image-20210419180322083](/images/OPENSIPS相关模块/image-20210419180322083.png)
  * 函数 rewritehostport：
    重写uri的目的和端口
  * 函数 append_hf：
    可以非常轻易的修改sip header头，比如 ：
    append_hf("P-hint: Onreply-route - fixcontact \r\n");
    产生以下sip头：
    ……
    P-hint: Onreply-route - fixcontact .
    ……
  * 函数 t_check_trans：
    用来检查当前请求是否已经在事务中了
  * 函数 is_method：
    用来判断当前的sip方法是什么，常用
    if (is_method("CANCEL")){
    #do something
    }
  * 函数 t_relay：
    非常神奇的函数，自动完成相关SIP事务处理（消息回复啊等等）
  * 函数 sl_send_reply：
    处理异常的好手，sl 代表 state less，无状态的，发了消息以后不等回复。经常配合 exit;一起使用
    if (!mf_process_maxfwd_header("10")) {
    sl_send_reply("483","Too Many Hops");
    exit; # 立即终止逻辑处理，并将处理的结果发给客户端
    }
  * 函数 force_rport：
    强制开启rport功能，一般认为是无害的，开启后server将会用客户端发请求的ip和端口，做为回复数据包的地址，基本能保证回包按原路返回，如果客户端在nat后，
    并且没启用rport功能，并且服务端也没强制开启这个函数，十有八九数据包是回不到客户端的，测试情况来看主要原因是服务端回复数据时，用的端口是contact中的端口，非实际的来源端口（回包的ip地址一般不会错）
  * RFC 3581 协议：
    客户端发起sip请求时，在via字段中，增加 rport;字符串
    REGISTER sip:47.93.175.70:15078;transport=UDP SIP/2.0.

  客户端新增了;rport. 字符串，via地址是私网

  Via: SIP/2.0/UDP 100.134.225.15:33608;branch=z9hG4bK-d8754z-e6cc4e84f201f41a-1---d8754z-;rport.
  服务端在回复客户端的请求时，将在sip头的via中新增rport字段
  SIP/2.0 200 OK.
  Via: SIP/2.0/UDP 100.134.225.15:33608;received=36.60.6.225;branch=z9hG4bK-d8754z-e6cc4e84f201f41a-1---d8754z-;rport=5350.
  相关资料见：https://my.oschina.net/u/147624/blog/33203

  * RFC 3489 协议：
    属于打洞协议，比较复杂，不能解决对称NAT的穿透问题
    相关资料见：https://my.oschina.net/u/147624/blog/33203
  * RFC 7118协议：
    描述如何通过websocket协议实现sip协议
    相关资料见：https://tools.ietf.org/html/rfc7118
    修改 opensips -cp -acl 页面显示坐席组的方式：
    cd /var/www/opensips-cp/config/tools/users/acl_management
    vim local.inc.php
    // list with the name of the groups defined in OpenSIPS cfg
    $config->grps = array("grp_xfyl","grp_internal","grp_external");
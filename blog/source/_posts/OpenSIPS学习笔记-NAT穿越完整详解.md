---
title: OpenSIPS学习笔记-NAT穿越完整详解
---

# OpenSIPS学习笔记-NAT穿越完整详解

## **NAT对SIP头的影响**

在NAT环境中，因为NAT后的网络对IP地址会进行处理，因此，NAT网络中SIP的一些业务也会受到影响。解决NAT问题可以通过SIP终端网络来解决，也可以通过SIP服务器端来解决。一般来说，如果通过SIP 客户端来解决的话，用户可以配置防火墙端口，ALG设置，STUN方式来实现。通过SIP服务器端处理的话，我们需要借助RTP Proxy来进行处理。网络何种处理方式，在NAT穿越以后，我们经常使用的一些比较重要的地址可能会发生变化，这样就会导致语音单通等问题。受到NAT影响到SIP头和SDP payload的 c行地址端口包括：

　　Via 头

　　Contact 头

　　Record Route

　　Route

　　SDP中的RTP 监听地址

　　INVITE sip:bob@slp.com SIP/2.0

　　Via: SIP/2.0/UDP 100.013:5060 //1

　　From: Alice <sip:alice@sip.com>

　　To: Bob <sip:bob@sip.com>

　　Call-lD: 12345600@10.0.0.13

　　CSeq: 1 INVITE

　　Contact: <sip:alice01000135060> // 2

　　Content-Type application/sdp

　　Content-Length:147

　　c=IN IP4 10.0.0.13 // 5

　　t=o o

　　……

　　因此，我们在使用OpenSIPS实现NAT穿越时需要根据一定的业务流程和呼叫来修改相应的地址实现呼叫的完整性。通过SIP信令和RTP 流调整来保证呼叫双方的互通。

## **STUN-OpenSIPS和TURN和RTP Proxy的使用讨论**

用STUN和SIP 服务器可以解决大部分的NAT穿越问题（前三种NAT类型），通过STUN服务器端可以对在NAT后面的SIP终端进行广播侦测查询，SIP终端获得NAT后的公网地址以后，SIP终端然后在进行INVITE呼叫，SIP终端呼叫时使用NAT公网地址对SIP服务器端进行呼叫。这样就解决了NAT穿越的问题。

![img](/images/OpenSIPS学习笔记-NAT穿越完整详解/20210406093339921.jpg)

　　通过以上的几个步骤可以解决前三种NAT穿越的问题。但是，因为Symmetric NAT本身的属性问题，Symmetric NAT本身可能开放不同的端口，STUN的局限性就很难满足对称NAT环境的要求，所以借助STUN结合OpenSIPS的方式进行NAT穿越的话，NAT穿越就可能失败。因为，Symmetric NAT环境下，在NAT网络环境下，不同终端可能会通过不同的端口来进行呼叫。端口不同，SIP信令和RTP流可能产生错误的流向，RTP流可能会流向错误的地址。因此，在Symmetric NAT环境中，我们不仅仅需要解决SIP 信令的问题，我们还要解决Symmetric 媒体的流向管理。Symmetric 媒体的处理原则目前采取了一种比较“聪明”的办法来解决这个问题，公网服务器端收到一个带内网IP的SDP的时候，其基本思路如下（其中一侧处于公网地址）：

1. 忽略这个内网IP地址和端口（因为是内网地址，属于无效地址和端口）
2. 继续等待，收到RTP语音流以后，通过RTP语音流的地址和端口确定真正的初始IP和端口
3. 根据以上收到的RTP流的IP地址和端口，把此IP地址和端口作为未来发送RTP语音流的目的地地址和端口

　　这种Symmetric 媒体的处理方式可以解决Symmetric NAT环境媒体的问题。但是，Symmetric NAT环境中必须要求一方是在公网环境中。

　　![img](/images/OpenSIPS学习笔记-NAT穿越完整详解/20210406093449653.jpg)

　　但是，如果呼叫方和被呼叫方都在NAT后的话，Symmetric NAT方式仍然不能获得完美的支持。在实际生产环境中，双方SIP终端都在NAT后的可能性也非常大，因此，为了彻底解决SIP终端在NAT后的穿越问题，Symmetric NAT需要借助于TURN媒体转发的能力来解决。

　　通过以上介绍，结合历史文档介绍，我们知道，事实上，目前市场上没有一个完整的解决方案来满足所有的NAT类型，它们都有各自的优缺点和其局限性：

1. ALG方式-通过终端路由器防火墙来处理，相对比较有效率，可以支持各种设备的分布式处理，但是因为SIP终端的路由器具备良好的的NAT支持
2. STUN方式-非常高效，具备分布式部署方式，但是不能支持所有的NAT类型
3. Symmetric 信令和媒体的处理方式，此仅仅方案可以支持所有的NAT类型，不依赖于客户端配置，但是要求一方必须是公网地址
4. Symmetric 信令加TURN方式，可以支持所有的NAT类型，双方都可以在NAT后，但是，因为依赖于TURN 转发处理，因此，这种处理方式缺乏良好的扩展性。目前看，这种方式是相对比较完整的解决方案。

　　在第四种方案中，通过OpenSIPS平台，OpenSIPS借助于媒体转发TURN可以实现SIP终端都在NAT环境的呼叫流程。这里，媒体转发的处理机制实际上是通过公网的RTP Proxy来实现。RTP Proxy作为一个B2B引擎，双方SIP呼叫通过RTP proxy转发来建立一个桥接流程，保证RTP流正常处理。基于OpenSIPS实现NAT穿越到基本处理流程包括：

- 通过OpenSIPS cfg 脚本检测NAT环境，管理SIP信令

- 在SIP注册方面，通过OpenSIPS 模块实现打洞或者Keepalive方式保持SIP的注册状态

- OpenSIPS协同媒体转发实现RTP 流的流向处理

## **OpenSIPS结合RTP引擎实现SIP注册和呼叫的NAT穿越**

在穿越处理中使用的比较多的是注册穿越处理和呼叫穿越处理流程。我们将重点介绍这两种NAT穿越的具体逻辑处理流程。

![img](/images/OpenSIPS学习笔记-NAT穿越完整详解/20210406093634552.jpg)

　　笔者建议用户在考虑OpenSIPS作为NAT穿越方案部署时，可以通过两种方式的部署来评价解决方案的可行性：

　　OpenSIPS作为一个Proxy实现SBC功能，实现其他业务功能。这样的处理方式非常紧凑，高效，可实现一定的扩展。但是，这种方式会增加cfg脚本的复杂程度。

　　在OpenSIPS前端部署一个SBC，专门处理NAT穿越。SBC作为一个专门的NAT穿越工具，降低了Proxy的脚本复杂程度，同时可以支持多地云分布式部署方式。这样的部署方式会增加前端的部署成本，需要更多的数据备份容灾的解决方案。目前，基于云平台的SBC越来越受到青睐，很多解决方案提供商提供前端SBC实现SIP trunk均衡负载，NAT穿越和SIP注册等问题，极大降低了后端业务系统的负载和复杂程度。

　　OpenSIPS平台可以通过SIP NAT穿越模块和RTP引擎模块实现NAT穿越，其模块包括：

- nathelper和rtpproxy
- nat_traversal和mediaproxy

　　通过NAT模块和媒体模块实现SIP头，SDP的管理，并且可以实现和外部RTP转发处理。以上两种方式非固定的搭配模式，它们可以任意组合。更多关于以上模块的详解说明，读者可以参考官方文档做进一步了解。接下来，我们具体介绍OpenSIPS实现NAT穿越到基本步骤。

　　首先，OpenSIPS需要检测NAT环境。通常情况下，NAT检测是在初始SIP请求中进行检测。后续的其他请求依赖于初始请求的检测结果。测试发送方是否在NAT环境中，OpenSIPS可以根据：

- Contact URL，IP地址部分
- Via地址和端口相对于收到的请求中的地址和端口
- SDP中IP地址类别

　　**OpenSIPS具体的实现脚本如下：**

　　# 使用bitmask检查，检查 VIA 是否是一个内网 （4）或者VIA端口不同

　　# 网络层级的端口（16-Via port 和源地址端口的不同），设置总值为（20=1+4）

　　if （nat uac_ test（20）） {setflag（"NATED_ SRC' '）; }

　　# 标识为带NAT的地址

　　另外一个需要检测到是NAT环境中的SIP终端注册状态。OpenSIPS检测如果发送方注册是一个NAT网络的话，OpenSIPS将会通过用户位置模块保存注册信息（usrloc）：

- 保存Contact 中的地址信息作为候选沟通的IP地址，事实上是一个内网的IP地址。
- 保存注册方的公网IP地址和端口，如果是NAT环境中的客户端，注册服务器需要知道从何公网地址进入注册。
- 如果判断出注册用户是一个NAT用户的话，需要保存其标识状态。

　　当OpenSIPS对注册方发送请求时，前面收到的Contact URL将作为一个RURI来处理，已保存的公网源地址和端口作为outbound proxy地址来处理。具体的脚本处理流程如下：

　　modparam（"usrloc", "nat_ bflag", "NATED_ DST"）

　　if（is_ method（"REGISTER"））{

　　if（nat_uac_test（20） ）{

　　fix_nated_register（）; # 保存注册源地址和端口（IP和port）

　　setbflag（"NATED_ DST"）; # 标识为NAT网络，branch flag（b flag）

　　}

　　save（"location"）;

　　}

　　我们已经通过以上脚本对SIP 注册状态执行了保存。但是，大家都知道，注册状态不是一个一次性处理的过程，维持注册状态需要SIP客户端不断向SIP服务器端发送ping消息，以便让服务器端获悉其存活状态实现NAT keepalive。另外，因为SIP用户是在NAT网络环境中，SIP终端需要对NAT网络环境进行打洞处理，需要保持其端口持续开放活动状态。SIP客户端和服务器端需要周期性地执行NAT打洞处理，发送数据来保持端口的活动状态，避免让NAT打洞的状态关闭。如果SIP状态没有侦测到的话，可能出现SIP呼叫超时等问题。在OpenSIPS环境中，我们可以使用两种方式实现打洞处理：

　　使用UDP ping方式，使用一个伪装的UDP数据包作为一个侦测数据包，生成方式相对比较简单，但是这是一种仅支持呼入方向的单向（服务器端到客户端）数据包侦测方式。侦测方向数据仅从公网到了内网终端。但是，通常情况下，内网终端执行状态更新时仅是通过内网到公网发送数据时才更新，因此，这种方式缺乏可靠性支持。

　　使用SIP ping方式，使用SIP options消息数据包作为一个侦测数据包，生成方式相对比较复杂，但是，SIP客户端可以生成一个返回的消息，侦测数据包流向是双向的，因此，可靠性更好，并且会生成大量的options消息数据。

　　OpenSIPS对SIP注册执行侦测的逻辑脚本如下：

　　modparam（"usrloc", "nat_ bflag", "NATED_ SRC"）

　　# 每20 秒，ping一次

　　modparam（" nathelper", "natping_ interval", 20）

　　# 仅对contacts 标识为NAT的终端执行ping 侦测

　　modparam（" nathelper", "ping_ nated_ only", 1）

　　# 执行 SIP pinging （with OPTIONS）

　　modparam（' 'nathelper", "sipping_ bflag", "NATED_ DST"）

　　除了检测SIP注册穿越到处理方式以外，SIP用户也同样需要检测在SIP呼叫过程中的NAT穿越问题处理。如果没有检测到NAT穿越到身份，SIP INVITE呼叫同样也会面临呼叫失败的可能。现在，我们讨论一下在OpenSIPS中如何检测和处理INVITE的NAT穿越问题。在SIP INVITE呼叫过程中，我们需要针对呼叫方和被呼叫方根据注册标识保存的信息进行NAT穿越的检测。OpenSIPS分别通过各自的检测方式对其身份进行检查，然后执行呼叫：

　　检测呼叫方是否在NAT环境中，如果在的话，使用源公网地址IP和端口修改其内网conatct URL地址，保证其呼叫路由可以正常执行。因为Contact URL是一个来自于内网的地址，如果没有修改contact URL的话，可能造成后续请求路由失败，例如ACK返回失败等问题。

　　检测被呼叫方是否在NAT环境中，这里需要注意，因为被呼叫方可能是一个NAT环境的SIP账号，也可能是第三方其他服务器端设备，所以执行INVITE流程时可以通过两种方式处理。如果被呼叫方不是NAT标识用户，则可以通过其他路由方式进行INVITE处理。

　　在进行INVITE前转之前，至少确认其中一个SIP终端是一个NAT环境中的设备，这样就可以触发RTP proxy代理工作流程，把SDP中的内网地址IP和端口替换为一个公网IP地址和端口实现媒体代理转发。

　　在INVITE执行以后，OpenSIPS仍然需要检测INVITE返回的信息，例如失败消息或者200 OK等成功返回的信息。如果是INVITE呼叫返回了失败的消息的话，OpenSIPS将关闭RTP 转发的处理流程，不再继续RTP 转发处理。如果INVITE呼叫返回了 200 OK消息的话，表示媒体转发处理是确认状态。OpenSIPS在确认了媒体转发处理以后，它将会更新SDP中的内网地址和端口，使用源公网地址IP和端口来替换SDP内网地址以便双方通过RTP媒体转发服务器进行RTP交互或者B2B 媒体处理。如果目的地地址检测到是一个NAT环境终端的话，根据标识进行处理，使用其公网地址替换Contact URL中的内网地址。关于OpenSIPS在INVITE呼叫中NAT穿越的处理方式如下：

![img](/images/OpenSIPS学习笔记-NAT穿越完整详解/20210406093842410.jpg)

　　注意，在以上关于INVITE呼叫的NAT标识中，OpenSIPS必须同时对呼叫源地址和目的地地址进行NAT标识，另外在呼叫目的地的标识中需要使用“b”flag进行标识。因为通常情况下，一些呼叫是一个fork呼叫，可能出现很多分叉呼叫，呼叫目的地可能是几个目的地地址，因此使用“b”-branch 的flag做标识处理，否则可能出现分叉呼叫返回的消息失败，或者无法处理后续dialog流程。关于OpenSIPS实现INVITE 呼叫NAT穿越的示例如下：

　　![img](/images/OpenSIPS学习笔记-NAT穿越完整详解/20210406093943330.jpg)

　　在以上示例中，笔者简单介绍了关于OpenSIPS结合媒体转发服务器的处理流程，其流程大概经过五个步骤的处理（SIP终端分别在各自的NAT环境中，通过RTP 转发实现语音通信）：

1. 首先，带NAT的呼叫方呼叫OpenSIPS服务器端，OpenSIPS服务器检测NAT，修改Contact URL为NAT公网地址，然后修改SDP地址为RTP 媒体转发服务器地址。
2. OpenSIPS服务器继续对对端Bob进行呼叫，并且携带更新后的Contact地址和SDP地址。
3. Bob对INVITE返回200 OK，携带自己的内网地址和SDP内网地址。
4. OpenSIPS经过NAT处理检测，然后返回到Alice 端，并且修改了Contact 地址和SDP的媒体转发服务器地址，通知Alice，BobContact地址是NAT的公网地址，SDP地址更新为RTP 媒体转发服务器地址。RTP语音流将通过RTP 转发服务器进行B2B转发处理
5. Alice和Bob双方通过RTP 转发媒体服务器进行RTP的流程处理，双方通过RTP服务器进行语音创建和通信。

　　以上是一个完整的OpenSIPS结合RTP转发服务器处理INVITE NAT穿越的流程示例。但是，很多时候，我们仅仅关注了INVITE的处理，可能会忽略一些其他的业务处理，例如，有时SIP用户可能重新发起re-INVITE 流程（例如SIP键盘的HOLD功能）。在re-INVITE的处理流程中，其处理逻辑和INVITE流程非常相似，一个比较大的区别就是无需在呼叫方和被呼叫方之间再进行NAT检测，可以通过dialog进行记录或者通过contact标识标志其SIP终端是一个NAT的终端。无论采取何种方式实现其标识的重新认证，OpenSIPS必须确保在re-INVITE以后，RTP转发代理必须强制使用新的SDP更新的地址和端口，否则，SIP发起re-INVITE以后可能导致双方无语音的问题。

　　前面，我们介绍了注册的NAT穿越解决思路和INVITE NAT穿越的解决办法，对于其他的非INVITE处理，NAT穿越的处理流程和INVITE基本一致，但是，OpenSIPS需要忽略媒体部分的处理。

　　另外，在我们的示例中，我们列举了在UDP的检测环境。以上的讨论目前没有针对TCP的SIP检测。如果SIP端是通过TCP注册的话，在OpenSIPS脚本中需要忽略TCP检测，使用UDP检测方式。

　　介绍了注册业务中使用NAT穿越的方式和呼叫流程中使用NAT穿越的细节以外，笔者将通过一个OpenSIPS配置示例说明其完整的NAT穿越流程。

## **OpenSIPS结合RTP媒体转发服务器配置示例**

　　在本章节中，笔者将通过OpenSIPS结合RTP Proxy配置示例说明如何实现SIP注册的NAT穿越和呼叫的NAT穿越。配置NAT穿越需要经过以下几个步骤：

　　首先，用户需要开启防火墙配置，保证需要的端口都在开启状态：

- ufw allow 5060
- ufw allow 9999 // 避免ALG 过滤，使用了9999端口
- 然后在cfg配置脚本中监听此端口
- ocket=udp:eth0:9999

　![img](/images/OpenSIPS学习笔记-NAT穿越完整详解/20210406094051958.jpg)

　　检查RTP proxy配置。笔者已经在同一OpenSIPS服务器安装了RTPProxy，用户通过配置文件检查rtpproxy配置，默认路径是：

　　etc/default/rtpproxy， OpenSIPS通过127.0.0.1 端口 7899 监听socket事件

　　然后确认RTPProxy启动，或者使用以下命令检查：

　　netstat -ulnp | grep rtpproxy

　　service rtpproxy start

　　启动了rtpproxy以后，用户可以修改opensips的cfg配置文件注册NAT穿越，增加相应的模块和参数配置：

　　modparam（"usrloc","nat_bflag","NATED_CALLEE"）

　　modparam（"registrar", "received_avp", "$avp（rcv）"）

　　**#加载NAT模块配置：**

　　loadmodule "nathelper.so"

　　modparam（"nathelper", "received_avp", "$avp（rcv）"）

　　modparam（"nathelper", "natping_interval", 30）

　　modparam（"nathelper", "ping_nated_only", 1）

　　modparam（"nathelper", "sipping_bflag", "SIPPING_FLAG"）

　　modparam（"nathelper", "sipping_from", "sip:pinger@sip.domain.com"）

　　#加载RTPproxy配置：

　　loadmodule "rtpproxy.so"

　　modparam（"rtpproxy", "db_url",

　　"mysql://opensips:opensipsrw@localhost/opensips"）

　　modparam（"rtpproxy", "default_set", 1）

　　在脚本流程中增加NAT检测：

　　force_rport（）;

　　if （nat_uac_test（18））

　　setflag（'NATED_CALLER'）;

　　在注册和呼叫的逻辑中增加NAT处理流程：

　　route（handle_nat）;

　　针对呼叫目的地进行NAT检测：

　　if （ruri_has_param（"nat","yes"）） // Contact URL检测

　　setbflag（'NATED_CALLEE'）;

　　增加注册NAT穿越的检测和保存其状态，并且执行ping命令：

　　if （isflagset（'NATED_CALLER'）） {

　　fix_nated_register（）;

　　setbflag（'NATED_CALLEE'）;

　　setbflag（'SIPPING_FLAG'）;

　　}

　　如果呼叫方或被呼叫方在NAT后的话，启动RTPProxy：

　　if （isflagset（'NATED_CALLER'） || isbflagset（'NATED_CALLEE'）） {

　　if （has_body（"application/sdp"））

　　rtpproxy_answer（"co"）;

　　}

　　# if callee （sender or this reply） is nated, fix it

　　if （ isbflagset（'NATED_CALLEE'） ）

　　fix_nated_contact（";nat=yes"）;

　　配置好以上脚本逻辑以后，用户需要通过GUI控制界面添加RTPProxy配置设置：

![img](/images/OpenSIPS学习笔记-NAT穿越完整详解/20210406094231414.jpg)

　　点击“Reload on Server”按钮，重新加载RTP代理服务器，确保成功加载。通过控制界面，查看SIP 用户的注册状态（Users -> User Management），点击user，会显示contact地址和收到的地址消息。使用OpenSIPS 服务器地址作为outbound proxy地址，关闭STUN设置。

　　两个都在NAT后的SIP可以进行呼叫来观察其IP地址的变化。另外，用户通过按键HOLD重新启动可以观察经过re-INVITE呼叫以后，检查双方语音是否丢，SDP是否更新等规则数据。如果reINVITE正常的话，SDP的数据应该也是得到了相应的更新。

　　在进一步的测试中，用户可以使用sngrep 跟踪INVITE的Contact URL地址和SDP中的RTP地址流向来判断是否通过OpenSIPS执行了NAT穿越处理。

　　![img](/images/OpenSIPS学习笔记-NAT穿越完整详解/20210406092512895.jpg)

　　200 OK的返回消息跟踪数据，注意其Contact URL和SDP中的c=行地址。

　![img](/images/OpenSIPS学习笔记-NAT穿越完整详解/20210406092527807.jpg)

　　在200 OK返回的消息中，我们可以看到SDP的地址修改为RTPProxy或者OpenSIPS的地址（同一台服务器）。Contact URL修改为NAT公网地址。再次说明，在针对INVITE 呼叫的NAT穿越的排查过程中，为了简单方便，用户应该实现注意是否返回了200 OK，并且一定要检查 200 OK中的Contact URL和SDP中的地址是否是RTPProxy地址。通过 200 OK返回消息可以快速排查问题。

　　为了避免终端的ALG问题，因为我们在cfg文件中监听了9999端口，用户可以通过抓包根据查看端口9999的数据流状态判断ALG的设置和RTPproxy的流向。
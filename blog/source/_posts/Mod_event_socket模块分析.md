---
title: Mod_event_socket模块分析
---

# **Mod_event_socket模块分析**

**一、** **mod_event_socket 功能**

**1、** **描述**

mod_event_socket以socket的形式，对外提供控制FS一种途径，缺省的IP是127.0.0.1，TCP端口是8021。可以在外部通过sokcet执行API/APP命令。配置文件是conf/autoload_configs/modules.conf.xml，连接分两种模式： inbound/outboundmod_event_socket 的默认加载模式是inbound,outbound模式需要在dialplan的配置文件中设置。mod_event_socktet的配置文件是conf/autoload_configs/event_socket.conf.xml。在配置文件中可以设置listen-ip；listen-port；password等。

**2、** **ACL控制表**

设置连接的访问控制列表,，可以在acl_conf.xml设置，也可以在event_socket.conf.xml中设置IP范围。下面例子是event_socket.conf.xml中启动ACL：<param name="apply-inbound-acl" value="<acl_list|cidr>"/>
acl表的在listener_run()线程中被使用，检查连接IP等。

**3、** **命令解析**

对socket中读取的数据进行解析（parse_command()函数完成解析功能）。

· unload

· reload

· api(阻塞模式)
api <command> <arg>
例如：
api originate sofia/mydomain.com/ext@yourvsp.com 1000  # connect sip:ext@yourvsp.com to extension 1000
api sleep 5000

· bapi(非阻塞模式)
bgapi <command> <arg> 

  api和bapi 只是执行方式不同，都是执行API命令。API命令很多见第三部分“命令列表”

· event
启动或禁止接收任意类型的事件。例如：
 event plain ALL
 event plain CUSTOM conference::maintenance
 event plain CHANNEL_CREATE CHANNEL_DESTROY CUSTOM conference::maintenance sofia::register sofia::expire

事件类型见第四部分“事件列表” 

· myevents
接收uuid的所有消息 

· divert_event
脚本注册测接收事件的函数分转到event socket上

· filter
设置event socket 接收事件类型 

· sendevent
发送一个事件到系统队列中 

· sendmsg
给一个uuid发送一个消息，可以执行其他模块的应用接口，也可以挂断电话等。例如：
SendMsg <uuid>
call-command: execute         (两种命令execute/hangup/unicast/nomedia)
execute-app-name: playback       (应用模块名称)
execute-app-arg: /tmp/test.wav     (参数) 

· exit
退出 

· auth
认证 

· log
启动日志 

· nolog
禁止日志 

· nixevent
启动日志

· noevents
禁止事件

**二、** **源码分析**

**1、** **整体流程**

mod_event_socket_load()：注册SWITCH_EVENT_ALL事件接收端口函数event_handler(),创建APP（socket_function）/API（event_sink_function）接口。加载完成后，系统会自动为runtime函数创建线程mod_event_socket_runtime()；线程内部接收连接，并初始listener结构，为listener创建线程listener_run();

**2、** **APP接口**

socket_function 是为FS提供OutBound模式提供的一个APP接口。

**3、** **API接口**

event_sink_function是为FS提供的以http协议方式控制FS的接口。

**4、** **API命令流程**

执行其他模块已经注册的API接口。流程如下：listener_run()>>read_packet()>>parse_command()>>create api_exec()线程()/api_exec()>>switch_api_execute()>>api->function()/switch_console_execute()。

**5、 **事件处理

mod_event_load()>>注册接收所有事件接口event_handler()，event_handler()接收到的信息，分发到连接对象listener的队列event_queue上。listener_run()>>read_packet()>>读取队列event_queue的事件，发送到socket。

**6、** **sendmsg命令**

判断listener是否以异步方式执行，如果是，将以事件的方式如对session->private_event_queue。如果不是，调用switch_ivr_parse_event()。调用流程：listener_run()>>read_packet()>>parse_command()>>
switch_ivr_parse_event()>>如果命令是"execute"执行switch_core_session_execute_application()。最主要功能是执行其它模块已经注册的APP接口。

**7、** **线程同步**

主线程是mod_event_socket_runtime()；每一个连接创建一个子线程listener_run(),他们之间的独立运行，当mod_event_socket_shutdown()被调用后，变量prefs.done=1；则主线程和子线程，检查到这个prefs.done==1,退出。

**三、** **命令列表**
**1、Core Commands** 
  acl | alias | bgapi | break | complete | cond | domain_exists | eval | expand | fsctl | min_dtmf_duration | max_dtmf_duration | default_dtmf_duration | global_getvar | global_setvar | host_lookup | hupall | in_group | is_lan_addr | load | md5 | module_exists | nat_map | reloadxml | show | shutdown | status | strftime_tz | unload | version | xml_locate | xml_wrap

**2、Call Management Commands** 
  create_uuid | group_call | help | originate | pause | regex | reload | reloadacl | uuid_audio | uuid_bridge | uuid_broadcast | uuid_chat | uuid_debug_audio | uuid_deflect | uuid_displace | uuid_display | uuid_dump | uuid_exists | uuid_flush_dtmf | uuid_getvar | uuid_hold | uuid_kill | uuid_media | uuid_park | uuid_send_dtmf | uuid_session_heartbeat | uuid_setvar | uuid_setvar_multi | uuid_transfer | uuid_simplify

**3、 Record/Playback Commands** 
   uuid_record 

**4、 Misc. Commands** 
  bg_system | echo | find_user_xml | sched_api | sched_broadcast | sched_del | sched_hangup | sched_transfer | stun | system | time_test | timer_test | tone_detect | unsched_api | url_decode | url_encode | user_data | user_exists

**四、** **事件列表**
**1、Channel events** 
   Channel states | CHANNEL_CREATE | CHANNEL_DESTROY | CHANNEL_STATE | CHANNEL_ANSWER | CHANNEL_HANGUP | CHANNEL_HANGUP_COMPLETE | CHANNEL_EXECUTE | CHANNEL_EXECUTE_COMPLETE | CHANNEL_BRIDGE | CHANNEL_UNBRIDGE | CHANNEL_PROGRESS | CHANNEL_PROGRESS_MEDIA | CHANNEL_OUTGOING | CHANNEL_PARK | CHANNEL_UNPARK | CHANNEL_APPLICATION | CHANNEL_ORIGINATE | CHANNEL_UUID 

**2、System events** 
   SHUTDOWN | MODULE_LOAD | MODULE_UNLOAD | RELOADXML | NOTIFY | SEND_MESSAGE | RECV_MESSAGE | REQUEST_PARAMS | CHANNEL_DATA | GENERAL | COMMAND | SESSION_HEARTBEAT | CLIENT_DISCONNECTED | SERVER_DISCONNECTED | SEND_INFO | RECV_INFO | CALL_SECURE | NAT | RECORD_START | RECORD_STOP | CALL_UPDATE |

**3、 Other events** 
   API | BACKGROUND_JOB | CUSTOM | RE_SCHEDULE | HEARTBEAT | DETECTED_TONE | ALL 

**4、 Undocumented events** 
   LOG | INBOUND_CHAN | OUTBOUND_CHAN | STARTUP | PUBLISH | UNPUBLISH | TALK | NOTALK | SESSION_CRASH | DTMF | MESSAGE | PRESENCE_IN | PRESENCE_OUT | PRESENCE_PROBE | MESSAGE_WAITING | MESSAGE_QUERY | ROSTER | CODEC | DETECTED_SPEECH | PRIVATE_COMMAND | TRAP | ADD_SCHEDULE | DEL_SCHEDULE | EXE_SCHEDULE |

5、**Custom events** 
  自定义事件。


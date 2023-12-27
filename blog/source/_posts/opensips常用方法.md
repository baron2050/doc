---
title: opensips常用方法
tag: opensips
---

# opensips常用方法

## 函数ds_is_in_list：

该方法需要使用dispatcher模块，判断ip和端口是否在表dispatcher中.

## 函数 t_check_trans：

用来检查当前请求是否已经在事务中了

##  函数 rewritehostport：

重写uri的目的和端口

## 函数 append_hf：

可以非常轻易的修改sip header头，比如 ：

```c
append_hf("P-hint: Onreply-route - fixcontact \r\n");
//产生以下sip头：
……
P-hint: Onreply-route - fixcontact .
……
```

相应的操作sip header头的还有以下方法

```c
if(is_present_hf("Remote-Party-ID")){//判断是否包含某个头
	 remove_hf("Remote-Party-ID"); //删除某个头
}
```

## 函数 fix_nated_register

负责处理当客户端处于NAT之后时，信令等相关信息的处理、变换、恢复
loadmodule "nathelper.so"

设置 received_avp 绑定的伪变量名称，与registrar.so的保持一致

modparam("nathelper", "received_avp", "$avp(received)")

当 nathelper 的 fix_nated_register 函数被调用时，会在register的200 ok包中，修改Contact，在后面增加 received 参数，此时received_avp绑定的变量将会被赋值，存储注册的客户端的真实URL（公网）

![image-20210425111728345](/images/opensips常用方法/image-20210425111728345.png)

如图所示：就是把注册时候发起注册时候真实ip和端口放到receivrd中。这个抓包因为是内网注册内网所以没有出现不一致的情况

## 函数dp_translate

```
dp_translate([partition:]id, src/dest[, attrs_pvar])

Will try to translate the src string into dest string according to the translation rules with dialplan ID equal to id.
Meaning of the parameters is as follows:
id - the dialplan id of possible matching rules. The id parameter can have the following types:
integer - the dialplan id is statically assigned
pvar - the dialplan id is the value of an existing pseudo-variable (as integer value)
partition - Specifies the partition where the search will take place. This parameter can be ommited. If the parameter is omitted the default partition will be used if exists. The partition parameter can have the following types:
string - the table name is statically assigned
pvar - the partition name is the value of an existing pseudo-variable (as string value)
src/dest - input and output of the function. If this parameter is missing the default parameter “ruri.user/ruri.user” will be used, thus translating the request uri.
The “src” variable can be any type of pseudo-variable.
The “dest” variable can be also any type of pseudo-variable, but it must be a writable one.
attrs_pvar (output, optional) - a pseudo-variable which will hold the attributes of the matched translation rule.
This function can be used from REQUEST_ROUTE, BRANCH_ROUTE.
```

![image-20210425133757728](/images/opensips常用方法/image-20210425133757728.png)

如上图：

dp_translate("1","$fU/$avp(nfrom)","$avp(dest)");

执行前： $fU=00000000  $avp(nfrom)=null $avp(dest)=null

执行后： $avp(nfrom)=guiji15250263953  $avp(dest)=（数据库中对应id的attr字段）

## 函数uac_replace_from

替换from头,如下:将$avp(nfrom)替换为$var(new_uri)。

```
uac_replace_from("$avp(nfrom)" , "$var(new_uri)");
```

## 函数uac_replace_to

类似于uac_replace_from

## 函数cr_route

```
cr_route(carrier, domain, prefix_matching, rewrite_user, hash_source, dstavp)
```

This function searches for the longest match for the user given in prefix_matching at the given domain in the given carrier tree. The Request URI is rewritten using rewrite_user and the given hash source and algorithm. Returns -1 if there is no data found or an empty rewrite host on the longest match is found. Otherwise the rewritten host is stored in the given AVP (if obmitted, the host is not stored in an AVP). This function is only usable with rewrite_user and prefix_matching containing a valid numerical only string. It uses the standard crc32 algorithm to calculate the hash values.

Meaning of the parameters is as follows:

- *carrier* - The routing tree to be used. Additional to a string any pseudo-variable could be used as input.

- *domain* - Name of the routing domain to be used. Additional to a string any pseudo-variable could be used as input.

- *prefix_matching* - User name to be used for prefix matching in the routing tree. Additional to a string any pseudo-variable could be used as input.

- *rewrite_user* - The user name to be used for applying the rewriting rule. Usually this is the user part of the request URI. Additional to a string any pseudo-variable could be used as input.

- *hash_source* - The hash values of the destination set must be a contiguous range starting at 1, limited by the configuration parameter max_targets. Possible values for hash_source are: call_id, from_uri, from_user, to_uri and to_user.

- *dstavp* - Name of the AVP where to store the rewritten host. This parameter is optional.

```
cr_route("$avp(carrier)","$avp(trunk_group)", "$rU", "$rU", "call_id", "$avp(host)")
```

  其中$avp(carrier)为数据库表route_tree对应的carrier字段，$avp(trunk_group)为carrierroute表中对应的domain字段，*prefix_matching-*用于在路由树中进行前缀匹配的用户名。*rewrite_user-*用于应用重写规则的用户名。hash_source-目标集的哈希值必须是从1开始的连续范围，并受配置参数max_targets限制。hash_source的可能值为：call_id，from_uri，from_user，to_uri和to_user。

  $rU为被叫的url如：sip:18512547903@172.16.102.203:64600

## 函数t_check_trans

## rtpproxy：

### rtpproxy_offer("ro","$var(ext_ip)");

### rtpproxy_answer("oc","$var(ext_ip)");

### rtpproxy_unforce();

## 模块rr.so

```c
modparam("rr", "enable_full_lr", 1)//如果是1在record_route中添加 ;lr=on否则添加;lr
modparam("rr", "append_fromtag", 0)//在record_route中添加 the request's from-tag
```

## 变量avp和$var

我所理解的两种变量，avp和$var的区别是avp的生命周期是整个会话（疑问：双腿通话？），而$avr则是整个脚本。

##  script flags.

![image-20210425155808775](/images/opensips常用方法/image-20210425155808775.png)

## sl_send_reply 

```
sl_send_reply("483","Too Many Hops");
```

## 顺序请求处理

```c
if (has_totag()) {
 # sequential request withing a dialog should
 # take the path determined by record-routing
 if (loose_route()) {
     if (is_method("BYE")) {
     setflag(1); # do accounting ... 
     setflag(3); # ... even if the transaction fails
     } else if (is_method("INVITE")) {
     # even if in most of the cases is useless, do RR for
     # re-INVITEs alos, as some buggy clients do change route set
     # during the dialog.
     record_route();
     }
     # route it out to whatever destination was set by loose_route()
     # in $du (destination URI).
     route(1);
 } else {
     if ( is_method("ACK") ) {
         if ( t_check_trans() ) {
         # non loose-route, but stateful ACK; must be an ACK after
         # a 487 or e.g. 404 from upstream server
         t_relay();
         exit;
         } else {
         # ACK without matching transaction ->
         # ignore and discard
         exit;
         }
     }
  sl_send_reply("404","Not here");
 }
 exit;
}
```

说明：如果has_totag表示此请求不是初始请求，松散路由顺序请求，直接转发下一步路由即可。如果是严格路由检测t_check_trans事务在做下一步处理，否则报错。


---
title: opensips测试
---

1 .需要数据库的模块大部分配置上数据库即可加载成功，但是domain这个模块还需要其他一些配置。整体模块加载配置如下

```c
loadmodule "httpd.so" 
modparam("httpd", "port", 8888) 

loadmodule "mi_json.so"

#### SIGNALING module
loadmodule "signaling.so"

#### StateLess module
loadmodule "sl.so"

#### Transaction Module
loadmodule "tm.so"
modparam("tm", "fr_timeout", 5)
modparam("tm", "fr_inv_timeout", 30)
modparam("tm", "restart_fr_on_each_reply", 0)
modparam("tm", "onreply_avp_mode", 1)

#### Record Route Module
loadmodule "rr.so"
/* do not append from tag to the RR (no need for this script) */
modparam("rr", "append_fromtag", 0)

#### MAX ForWarD module
loadmodule "maxfwd.so"

#### SIP MSG OPerationS module
loadmodule "sipmsgops.so"

#### FIFO Management Interface
loadmodule "mi_fifo.so"
modparam("mi_fifo", "fifo_name", "/tmp/opensips_fifo")
modparam("mi_fifo", "fifo_mode", 0666)

#### URI module
loadmodule "uri.so"
modparam("uri", "use_uri_table", 0)

#### MYSQL module
loadmodule "db_mysql.so"

#### AVPOPS module
loadmodule "avpops.so"

#### ACCounting module
loadmodule "acc.so"
/* what special events should be accounted ? */
modparam("acc", "early_media", 0)
modparam("acc", "report_cancels", 0)
/* by default we do not adjust the direct of the sequential requests.
   if you enable this parameter, be sure the enable "append_fromtag"
   in "rr" module */
modparam("acc", "detect_direction", 0)


#### DIALOG module
loadmodule "dialog.so"
modparam("dialog", "dlg_match_mode", 1)
modparam("dialog", "default_timeout", 21600)  # 6 hours timeout
modparam("dialog", "db_mode", 2)
modparam("dialog", "db_url","mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME

#### LOAD BALANCER module 以下配置会自动给gateway发送option消息每隔30秒发一次
loadmodule "load_balancer.so"
modparam("load_balancer", "db_url","mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME
modparam("load_balancer", "probing_method", "OPTIONS")
modparam("load_balancer", "probing_interval", 30)


loadmodule "rtpproxy.so"
modparam("rtpproxy","rtpproxy_sock","udp:192.168.6.75:7890")

loadmodule "domain.so"
modparam("domain", "db_url","mysql://opensips:opensipsrw@localhost/opensips")
modparam("domain", "db_mode", 1)
modparam("domain", "domain_table", "domain")

#loadmodule "domainpolicy.so"
#modparam("domainpolicy", "db_url","mysql://opensips:opensipsrw@localhost/opensips")

loadmodule "dispatcher.so"
modparam("dispatcher", "db_url","mysql://opensips:opensipsrw@localhost/opensips")

loadmodule "drouting.so"
modparam("drouting", "db_url","mysql://opensips:opensipsrw@localhost/opensips")
loadmodule "permissions.so"
modparam("permissions", "db_url","mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME

loadmodule "dialplan.so"
modparam("dialplan", "db_url","mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME

```

2. carrierroute 需要route_tree表有数据，否则报错
3. unknown command <replace_all>，缺乏loadmodule "textops.so"模块导致的

   4.最终脚本

```c
#
# OpenSIPS loadbalancer script
#     by OpenSIPS Solutions <team@opensips-solutions.com>
#
# This script was generated via "make menuconfig", from
#   the "Load Balancer" scenario.
# You can enable / disable more features / functionalities by
#   re-generating the scenario with different options.
#
# Please refer to the Core CookBook at:
#      http://www.opensips.org/Resources/DocsCookbooks
# for a explanation of possible statements, functions and parameters.
#


#
# OpenSIPS loadbalancer script
#     by OpenSIPS Solutions <team@opensips-solutions.com>
#
# This script was generated via "make menuconfig", from
#   the "Load Balancer" scenario.
# You can enable / disable more features / functionalities by
#   re-generating the scenario with different options.
#
# Please refer to the Core CookBook at:
#      http://www.opensips.org/Resources/DocsCookbooks
# for a explanation of possible statements, functions and parameters.
#


####### Global Parameters #########

log_level=3
log_stderror=no
log_facility=LOG_LOCAL0

children=4

/* uncomment the following lines to enable debugging */
#debug_mode=yes

/* uncomment the next line to enable the auto temporary blacklisting of 
   not available destinations (default disabled) */
#disable_dns_blacklist=no

/* uncomment the next line to enable IPv6 lookup after IPv4 dns 
   lookup failures (default disabled) */
#dns_try_ipv6=yes

/* comment the next line to enable the auto discovery of local aliases
   based on reverse DNS on IPs */
auto_aliases=no


listen=udp:192.168.6.75:6060




####### Modules Section ########

#set module path
mpath="/usr/lib/x86_64-linux-gnu/opensips/modules/"

loadmodule "httpd.so" 
modparam("httpd", "port", 8888) 

loadmodule "mi_http.so"
loadmodule "mi_json.so"

#### SIGNALING module
loadmodule "signaling.so"

#### StateLess module
loadmodule "sl.so"
loadmodule "textops.so"


#### Transaction Module
loadmodule "tm.so"
modparam("tm", "fr_timeout", 5)
modparam("tm", "fr_inv_timeout", 30)
modparam("tm", "restart_fr_on_each_reply", 0)
modparam("tm", "onreply_avp_mode", 1)

#### Record Route Module
loadmodule "rr.so"
/* do not append from tag to the RR (no need for this script) */
modparam("rr", "append_fromtag", 1)

#### MAX ForWarD module
loadmodule "maxfwd.so"
loadmodule "uac.so"

#### SIP MSG OPerationS module
loadmodule "sipmsgops.so"

#### FIFO Management Interface
loadmodule "mi_fifo.so"
modparam("mi_fifo", "fifo_name", "/tmp/opensips_fifo")
modparam("mi_fifo", "fifo_mode", 0666)

#### URI module
loadmodule "uri.so"
modparam("uri", "use_uri_table", 0)
modparam("uri", "db_url","mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME

#### MYSQL module
loadmodule "db_mysql.so"

#### AVPOPS module
loadmodule "avpops.so"
modparam("avpops", "db_url","mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME
modparam("avpops", "avp_table", "dbaliases")


#### USeR LOCation module
loadmodule "usrloc.so"
modparam("usrloc", "nat_bflag", "NAT")
modparam("usrloc", "db_mode",3)
modparam("usrloc", "db_url","mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME





#### ACCounting module
loadmodule "acc.so"
/* what special events should be accounted ? */
modparam("acc", "early_media", 0)
modparam("acc", "report_cancels", 0)
/* by default we do not adjust the direct of the sequential requests.
   if you enable this parameter, be sure the enable "append_fromtag"
   in "rr" module */
modparam("acc", "detect_direction", 0)


#### DIALOG module
loadmodule "dialog.so"
modparam("dialog", "dlg_match_mode", 1)
modparam("dialog", "default_timeout", 21600)  # 6 hours timeout
modparam("dialog", "db_mode", 2)
modparam("dialog", "db_url",
	"mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME

loadmodule "permissions.so"
modparam("permissions", "db_url",
	"mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME

loadmodule "dialplan.so"
modparam("dialplan", "db_url",
	"mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME

#### LOAD BALANCER module
loadmodule "load_balancer.so"
modparam("load_balancer", "db_url","mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME
modparam("load_balancer", "probing_method", "OPTIONS")
modparam("load_balancer", "probing_interval", 30)


loadmodule "rtpproxy.so"
modparam("rtpproxy","rtpproxy_sock","udp:192.168.6.75:7890")

loadmodule "domain.so"
modparam("domain", "db_url","mysql://opensips:opensipsrw@localhost/opensips")
modparam("domain", "db_mode", 1)
modparam("domain", "domain_table", "domain")

#### REGISTRAR module
loadmodule "registrar.so"
modparam("registrar", "tcp_persistent_flag", "TCP_PERSISTENT")
modparam("registrar", "received_avp", "$avp(received_nh)")
/* uncomment the next line not to allow more than 10 contacts per AOR */
modparam("registrar", "max_contacts", 1)
modparam("registrar", "min_expires", 60)
modparam("registrar", "max_expires", 120)
modparam("registrar", "default_expires", 120)
modparam("registrar", "attr_avp", "$avp(attr)")
modparam("registrar", "received_avp", "$avp(received_nh)")

#### Registrant module
loadmodule "uac_auth.so"
#modparam("uac_auth","credential","1001:172.16.1.103:guijizhineng")
modparam("uac_auth","auth_realm_avp","$avp(reg_realm)")
modparam("uac_auth","auth_username_avp","$avp(reg_username)")
modparam("uac_auth","auth_password_avp","$avp(reg_pwd)")

loadmodule "uac_registrant.so"
modparam("uac_registrant", "hash_size", 2)
modparam("uac_registrant", "db_url","mysql://opensips:opensipsrw@localhost/opensips")
modparam("uac_registrant", "timer_interval", 30)

#### AUTHentication modules
loadmodule "auth.so"
loadmodule "auth_db.so"
modparam("auth_db", "calculate_ha1", yes)
modparam("auth_db", "password_column", "password")
modparam("auth_db", "db_url",
	"mysql://opensips:opensipsrw@localhost/opensips") # CUSTOMIZE ME
modparam("auth_db", "load_credentials", "")


#####
loadmodule "nathelper.so"
modparam("nathelper", "natping_interval", 10)
modparam("nathelper", "ping_nated_only", 1)
modparam("nathelper", "received_avp", "$avp(received_nh)")


#####  carrier route Module
loadmodule "carrierroute.so"
modparam("carrierroute", "db_url", "mysql://opensips:opensipsrw@localhost/opensips")                # CUSTOMIZE ME
modparam("carrierroute", "config_source", "db")
modparam("carrierroute", "use_domain", 1)
modparam("carrierroute", "db_failure_table", "carrierfailureroute")


loadmodule "dispatcher.so"
modparam("dispatcher", "db_url","mysql://opensips:opensipsrw@localhost/opensips")

loadmodule "drouting.so"
modparam("drouting", "db_url","mysql://opensips:opensipsrw@localhost/opensips")

loadmodule "proto_udp.so"

loadmodule "proto_tcp.so" 


####### Routing Logic ########


# main request routing logic

route{
	$var(local_ip)="192.168.6.75";
	$var(ext_ip)="192.168.6.75";				# CUSTOMIZE ME
	$var(local_ip_port)="192.168.6.75:6060";			# CUSTOMIZE ME
	$var(ext_ip_port)="192.168.6.75:6060";		# CUSTOMIZE ME
	xlog("L_INFO","[$ci] [$rm] route start. ru=$ru si=$si:$sp fu=$fu tu=$tu rs=$rs");

	if (!mf_process_maxfwd_header("10")) {
		send_reply("483","Too Many Hops");
		exit;
	}
	
	force_rport();

	if(is_method("OPTIONS")) {
		# send reply for each options request
		sl_send_reply("200", "ok");
		exit();
	}

	if(is_method("NOTIFY")){
	  if($rU==NULL){
	    $rU=$tU;
	  }
	}
	
	if (nat_uac_test("19")) {
        	if (is_method("REGISTER")) {
			xlog("register request fix nat $ru ! ");
            		fix_nated_register();
            		setbflag(10);
        	} else {
			xlog("not register request fix nat  $ru  ! ");
            		fix_nated_contact();
			setflag(10);
		}
	}

	if (has_totag() && is_method("INFO|UPDATE|INVITE|ACK|REFER|BYE")){
        	if(match_dialog()){
                	xlog(" in-dialog topology hiding request - $DLG_dir\n");
			if ( $DLG_status!=NULL && !validate_dialog() ){
				fix_route_dialog();
			}
			t_relay();
			xlog("L_INFO","[$ci] t_relay is finished!rc=$rc");
                	exit;
        	}
	}

	if (has_totag() && !is_method("NOTIFY")) {
		if (loose_route()) {
			xlog("enter in loose_route");
			if (is_method("BYE")) {
				xlog("receive BYE");
				# rtp release
			} else if (is_method("INVITE")) {
				# even if in most of the cases is useless, do RR for
				# re-INVITEs alos, as some buggy clients do change route set
				# during the dialog.
				record_route();
				#t_on_reply("rwSDP");
			} else if (is_method("NOTIFY")) {
				# even if in most of the cases is useless, do RR for
				# re-INVITEs alos, as some buggy clients do change route set
				# during the dialog.

			}else if (is_method("ACK")) {
				t_relay();
				xlog("L_INFO","[$ci] t_relay is finished!rc=$rc");
				exit;
			}
	        	if (check_route_param("nat=yes")){
				xlog(" exec setflag:10");
				setflag(10);
			}

			# route it out to whatever destination was set by loose_route()
			# in $du (destination URI).
			xlog("exec route:relay");
			route(relay);
		}else{
			xlog("[$ci] enter in non loose_route ");
			if ( is_method("ACK") ) {
				if ( t_check_trans() ) {
					# non loose-route, but stateful ACK; must be an ACK after 
					# a 487 or e.g. 404 from upstream server
					t_relay();
					xlog("L_INFO","[$ci] t_relay is finished!rc=$rc");
					exit;
				} else {
					# ACK without matching transaction ->
					# ignore and discard
					exit;
				}
			}
			
			if ( is_method("BYE") ) {
				xlog(" receive BYE ");
				if ( t_check_trans() ) {
					xlog("receive bye request t_check_trans true");
					t_relay();
					xlog("L_INFO","[$ci] t_relay is finished!rc=$rc");
					exit;
				} else {
					xlog("receive bye request t_check_trans false");
					exit;
				}
			}
			sl_send_reply("404","Not here");	
		}
		exit;
	}	
	
	if (is_method("CANCEL"))
	{
		xlog("receive CANCEL");
		if (t_check_trans()){
			t_relay();
			xlog("L_INFO","[$ci] t_relay is finished!rc=$rc");
		}
		exit;
	}

	t_check_trans();

	# account only INVITE
	if (is_method("INVITE")) {	
		if ( !create_dialog("B") ) {
			xlog("exec send_reply 500 Internal Server Error");
			send_reply("500","Internal Server Error");
			exit;
		}
		#setflag(ACC_DO); # do accounting
		setflag(CDR_FLAG);
	}

	if ( !(is_method("REGISTER")) ) {
		xlog("invite request from_domain-$td from_ip-$si");		
		if (is_from_local()){
			if(lb_is_destination("$si","0")){
				xlog("invite request from meida server !");
			}else{
				xlog("exec proxy_authorize.");
				
			}
		
		}
	}

	if (is_method("REGISTER")){
		$var(auth_code) = www_authorize("", "subscriber");
		if ( $var(auth_code) == -1 || $var(auth_code) == -2 ) {
			xlog("L_NOTICE","Auth error for $fU@$fd from $si cause $var(auth_code)");
		}
		if ( $var(auth_code) < 0 ) {
				www_challenge("", "0");
				exit;
		}
		if (!db_check_to()) 
		{
			xlog("exec sl_send_reply:403 Forbidden auth ID");
			sl_send_reply("403","Forbidden auth ID");
			exit;
		}
		$avp(attr)="sip:"+$si+":"+$sp;
		if (!save("location","p1v")){
			xlog("exec sl_reply_error");
			sl_send_reply("503","too many registered");
		}
		exit;	
	}	

	if ($rU==NULL) {
		# request with no Username in RURI
		send_reply("484","Address Incomplete");
		exit;
	}

	
	


	if (is_from_local()){
		if (is_method("INVITE") && has_body("application/sdp")){
			record_route();
			#rtpproxy_engage("oc");
        	}
		
		if (!load_balance("1","trunk")) {
			xlog("L_ERR","load_balance failed:sl_send_reply 500 service full.");
			sl_send_reply("500","Service full.");
			exit;
		}else{
			xlog("load balance success!");
		}
		t_relay();
		xlog("L_INFO","[$ci] t_relay is finished!rc=$rc");
	
	}else{
		# call out 
		xlog("L_INFO","do nothing Invite Not from local... ci=$ci si=$si ru=$ru fu=$fu tu=$tu");
		$rd = $td;
		xlog("L_INFO","enter in domain route fn=$fn");
		# call out
	#	sl_send_reply("500","Service full.");
		if (is_method("INVITE") && has_body("application/sdp")){
				record_route();
				#rtpproxy_engage("oc");
        	}
		avp_db_query("SELECT id,caller_id_dpid,callee_id_dpid,trunk_group FROM wj_route_group WHERE id=$fn","$avp(r_id);$avp(caller_id_dpid);$avp(callee_id_dpid);$avp(trunk_group)");
		if(is_avp_set("$avp(r_id)"))
		{	
			dp_translate("$avp(caller_id_dpid)","$fU/$avp(nfrom)","$avp(dest)");
			dp_translate("$avp(callee_id_dpid)","$tU/$avp(nto)","$avp(dest)");
			if(is_avp_set("$avp(nfrom)")){
				$var(new_uri) = "sip:" + $avp(nfrom) + "@" + $fd;
				uac_replace_from("$avp(nfrom)" , "$var(new_uri)");
				xlog("L_INFO","rewrite from head $fU ---> $var(new_uri).");
			}else{
				xlog("L_INFO","extension not set callerid.");
			
			}
			if(is_avp_set("$avp(nto)")){
				$var(new_to) = "sip:"+ $avp(nto) + "@" + $rd;
				xlog("L_INFO","rewrite to head $tU ---> $var(new_to)");
				uac_replace_to("$tU" , "$var(new_to)");
			}else{
				xlog("L_INFO","extension not set calleeid.");
			}
		}
		if(!is_avp_set("$avp(trunk_group)"))
		{
			sl_send_reply("402", "No trunk group");
		}
		xlog("begin route... carrier=$avp(carrier) trunk_group=$avp(trunk_group) request_username=$rU host=$avp(host)");
		xlog("rtpproxy rewrite internal ip...");
		rtpproxy_offer("ro","$var(ext_ip)");
		record_route_preset("$var(ext_ip_port)");
		$avp(carrier)="carrier1";
		if(!cr_route("$avp(carrier)","$avp(trunk_group)", "$rU", "$rU", "call_id", "$avp(host)")){
			xlog("L_ERR","cr_route failed! send reply 485.");
			sl_send_reply("485", "route ambiguous!");
		}else{
			xlog("avp host $avp(host)");
			t_on_failure("missed_call");
			
		}
					
		if (isbflagset(10)){ 
			setflag(10);
		}
		xlog("L_INFO","internal calling.... ");
		route(relay);
	}
	
}


route[relay] {
	xlog("L_INFO","[ci] enter route relay..");
        if(is_present_hf("Remote-Party-ID")){
	        remove_hf("Remote-Party-ID");
        }
	if (is_method("INVITE")) {
		t_on_branch("per_branch_ops");
		t_on_reply("handle_nat");
		t_on_failure("missed_call");
	}
	
	if (isflagset(10)) {
		add_rr_param(";nat=yes");
	}
	if (!t_relay()) {
		xlog("L_ERR","send reply 500 Internal Error! rc=$rc");
		send_reply("500","Internal Error");
	};
	exit;
}
branch_route[per_branch_ops] {
	xlog("L_INFO","[$ci] enter new branch at $ru\n");
}
onreply_route[handle_nat] {
	xlog("L_INFO","[$ci] enter onreply_route[handle_nat]");
	fix_nated_contact();
	xlog("L_INFO","[$ci] incoming reply $rs \n");
	if(is_method("BYE|CANCEL")){ 
	       rtpproxy_unforce();
        }
	if(has_body("application/sdp")){
        	xlog("L_INFO","onreply_route1  rtpproxy_answer----si=$si");
			if(ds_is_in_list("$si","$sp","1")){
			xlog("L_INFO","rtpproxy ext_ip");
			rtpproxy_answer("ro","$var(ext_ip)");
			replace_all("sip:192.168.6.75:6060","sip:192.168.6.75:6060");		# CUSTOMIZE ME
		}else{	
			xlog("L_INFO","rtpproxy local_ip");
			rtpproxy_answer("ro","$var(local_ip)");
			replace_all("sip:192.168.6.75:6060","sip:192.168.6.75:6060");		# CUSTOMIZE ME
		}
        }
	
	if (t_check_status("603") || t_check_status("486") 
		|| t_check_status("480")) {
		xlog("L_WARN","receive 603/486/480");
	}
}
failure_route[missed_call] {
	xlog("L_INFO","[$ci] enter failure_route[missed_call]");
	if (t_was_cancelled()) {
		exit;
	}
	if(t_check_status("408")){
		xlog("L_INFO","[$ci] failure_route[missed_call] t_check_status:408");
	}else if(t_check_status("500")){
                xlog("L_INFO","[$ci] failure_route[missed_call] t_check_status:500");
        }else if(t_check_status("503")){
                xlog("L_INFO","[$ci] failure_route[missed_call] t_check_status:503");
        }else if(t_check_status("504")){
                xlog("L_INFO","[$ci] failure_route[missed_call] t_check_status:504");
        }else if(t_check_status("600")){
                xlog("L_INFO","[$ci] failure_route[missed_call] t_check_status:600");
        }else if(t_check_status("603")){
                xlog("L_INFO","[$ci] failure_route[missed_call] t_check_status:603");
        }else if(t_check_status("699")){
                xlog("L_INFO","[$ci] failure_route[missed_call] t_check_status:699");
        }
	xlog("L_INFO","[$ci] failure_route reply $(err.rcode) - $(err.rreason) -$(err.info) -$(err.class) \n");	
}


local_route {
	if (is_method("BYE") && $DLG_dir=="UPSTREAM") {
		acc_log_request("200 Dialog Timeout");		
	}
}

onreply_route[1]{
	xlog("L_INFO","[$ci] enter in onreply_route1");
}
onreply_route[2]{
	xlog("L_INFO","[$ci] enter in onreply_route2");
}
```


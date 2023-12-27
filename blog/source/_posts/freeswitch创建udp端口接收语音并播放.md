---
title: freeswitch创建udp端口接收语音并播放
---

```c
#include <switch.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

SWITCH_MODULE_LOAD_FUNCTION(mod_frame_write_load);
SWITCH_MODULE_SHUTDOWN_FUNCTION(mod_frame_write_shutdown);

SWITCH_MODULE_DEFINITION(mod_frame_write, mod_frame_write_load, mod_frame_write_shutdown, NULL);
typedef struct
{
    switch_media_bug_t *bug;
    switch_core_session_t *xsession;
    switch_codec_implementation_t read_impl;
    long packet_num;
	switch_socket_t *sock;
	switch_sockaddr_t *r_sockaddr;
    switch_sockaddr_t *l_sockaddr;
    switch_port_t local_port;
    char local_ipv4_str[256];	
} switch_da_t;

static void close_socket(switch_socket_t **sock)
{
	if (*sock) {
		switch_socket_shutdown(*sock, SWITCH_SHUTDOWN_READWRITE);
		switch_socket_close(*sock);
		*sock = NULL;
	}
}
// udp端口读取语音流的线程
static void *SWITCH_THREAD_FUNC read_frame_thread(switch_thread_t *thread, void *obj){
    switch_da_t *pvt = (switch_da_t *)obj;
    switch_size_t len = 3200;
    uint32_t datalen = 0;
    switch_timer_t timer = { 0 };
    switch_frame_t write_frame;
    switch_codec_t raw_codec = { 0 };
    char *aud_buffer;

    if(pvt){
        datalen =  (pvt->read_impl.microseconds_per_packet / 1000) * 16;

    }
    switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(pvt->xsession), SWITCH_LOG_WARNING, "set write data len %d.\n",datalen);
    if ( (pvt) && switch_core_timer_init(&timer, "soft", pvt->read_impl.microseconds_per_packet / 1000,
							   pvt->read_impl.samples_per_packet, switch_core_session_get_pool(pvt->xsession)) != SWITCH_STATUS_SUCCESS) {
		switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(pvt->xsession), SWITCH_LOG_ERROR, "Timer Activation Fail\n");
        goto end;
	}
    if ((pvt) && switch_core_codec_init(&raw_codec,
                                    "L16",
                                    NULL,
                                    NULL,
                                    pvt->read_impl.actual_samples_per_second,
                                    pvt->read_impl.microseconds_per_packet / 1000,
                                    1, SWITCH_CODEC_FLAG_ENCODE | SWITCH_CODEC_FLAG_DECODE,
                                    NULL, switch_core_session_get_pool(pvt->xsession)) != SWITCH_STATUS_SUCCESS) {
                    switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(pvt->xsession), SWITCH_LOG_WARNING, "Failed to initialize L16 codec.\n");
                    goto end;
    }

    write_frame.codec = &raw_codec;
    //8K采样率
    write_frame.datalen = datalen;

    write_frame.samples = datalen / sizeof(int16_t);
    
    aud_buffer = switch_core_session_alloc(pvt->xsession,datalen);
    write_frame.data = aud_buffer;
    switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(pvt->xsession), SWITCH_LOG_WARNING, "ready to read frame.\n");
    while((pvt) && (pvt->sock) && (pvt->local_port > 0)){
        char buf[3200] = {0};
        switch_socket_recv(pvt->sock, buf, &len);
        if(len>0){
            switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(pvt->xsession), SWITCH_LOG_WARNING, "read frame len = %ld.\n",len);
            for(int i=0;i<len/datalen;i++){
                //如果读取到数据则将数据存入缓存
                memcpy(aud_buffer,buf + i*datalen,datalen);
                if(switch_core_session_write_frame(pvt->xsession, &write_frame, SWITCH_IO_FLAG_NONE, 0)!=SWITCH_STATUS_SUCCESS){
                    switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(pvt->xsession), SWITCH_LOG_WARNING, "Failed to write frame.\n");
                }
                switch_core_timer_next(&timer);
            }
        }
        switch_sleep(10*1000);
    }
    end:
        if (timer.interval) {
		    switch_core_timer_destroy(&timer);
        }
        if (switch_core_codec_ready(&raw_codec)) {
		    switch_core_codec_destroy(&raw_codec);
	    }
   return NULL;
}


static void release_pvt(switch_da_t *pvt){
    if (pvt)
        {
            if ((pvt->local_ipv4_str) && (pvt->local_port)) {
                switch_rtp_release_port(pvt->local_ipv4_str, pvt->local_port);
                switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_NOTICE, "succ release RTP port %d for IP %s\n", pvt->local_port, pvt->local_ipv4_str);
                pvt->local_port=0;
            } 
            if(pvt->sock){
                close_socket(&pvt->sock);
                switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_NOTICE, "Fork media Stop Succeed, packet number = %ld\n",pvt->packet_num);
            } 
        }
}

static switch_bool_t frame_write_callback(switch_media_bug_t *bug, void *user_data, switch_abc_type_t type)
{
    switch_da_t *pvt = (switch_da_t *)user_data;
    
    switch (type)
    {
    case SWITCH_ABC_TYPE_INIT:
        {

        }
        break;
    case SWITCH_ABC_TYPE_CLOSE:
        {
            release_pvt(pvt);
        }
        break;
    case SWITCH_ABC_TYPE_READ_REPLACE:
        {
            switch_frame_t *frame;
            if ((frame = switch_core_media_bug_get_read_replace_frame(bug)))
            {
                
                char *frame_data = (char *)frame->data;
                switch_size_t frame_len = frame->datalen;
                
               
                if (frame->channels != 1)
                {
                    switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_CRIT, "nonsupport channels:%d!\n", frame->channels);
                    return SWITCH_FALSE;
                }
                if ((pvt) && (pvt->sock))
                {
                    
                    switch_socket_send(pvt->sock, frame_data, &frame_len);
                    memset(frame->data, 255, frame->datalen);
                    switch_core_media_bug_set_read_replace_frame(bug, frame);
                    if(pvt->packet_num==0 ){
                       //启动udp读线程
                      	switch_thread_data_t *td_profile;
                        td_profile = switch_core_session_alloc(pvt->xsession, sizeof(*td_profile));
                        td_profile->alloc = 0;
                        td_profile->func = read_frame_thread;
                        td_profile->obj = pvt;
                        switch_thread_pool_launch_thread(&td_profile);  
                        switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_CRIT, "read_frame_thread started!\n");
                    }
                    pvt->packet_num++;
                }
            }
        }
        break;
    // case SWITCH_ABC_TYPE_WRITE_REPLACE:
    //     {
    //         switch_frame_t *rframe = NULL;
        
    //         if((rframe = switch_core_media_bug_get_write_replace_frame(bug)))    
    //         {
    //             if (rframe->channels != 1)
    //             {
    //                 switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_CRIT, "nonsupport channels:%d!\n", rframe->channels);
    //                 return SWITCH_FALSE;
    //             }
    //             memset(rframe->data, 0, rframe->datalen);
    //             switch_core_media_bug_set_write_replace_frame(bug, rframe);
    //         }
    //     }
    //     break;
    default:
        break;
    }

    return SWITCH_TRUE;
}

/*
   `uuid_frame_write <uuid> <switch> <flag> <udpAddr>`    
uuid: 通道uuid    
switch: 流转发开关, 值为on/off, 用于打开或者关闭转发的媒体流    
flag: both/read/write, 转发的流    
read, 代表转发read方向的流    
write, 代表转发write方向的流    
both, 代表同时转发read和write方向的流, 如果为both, 则udpAddr是两个地址用逗号隔开, 前一个地址用于read,后一个地址用于write    
udpAddr: 接收流的目的地址, 一般为ip加端口, 如192.168.2.11:10080    

### 用法: 开启read流转发
​```
uuid_frame_write  a23resefkslfsl on read 192.168.2.11 10080   
*/
SWITCH_STANDARD_API(uuid_frame_write_function)
{
    char *mycmd = NULL, *argv[5] = {0};
    int mask = 0;
    switch_da_t *pvt = NULL;
    switch_core_session_t *xsession = NULL;
    switch_status_t status = SWITCH_STATUS_SUCCESS;
    switch_media_bug_flag_t flags = SMBF_READ_REPLACE | SMBF_NO_PAUSE | SMBF_ONE_ONLY;
    switch_channel_t *channel = NULL;

    switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_NOTICE, "Get API buf %s.\n", cmd);
    if (!zstr(cmd) && (mycmd = strdup(cmd)))
    {
        switch_separate_string(mycmd, ' ', argv, (sizeof(argv) / sizeof(argv[0])));
    }
    xsession = switch_core_session_locate(argv[0]);
    if (!xsession)
    {
        switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "Can not find the %s session.\n", argv[0]);
        goto done;
    }

    channel = switch_core_session_get_channel(xsession);
    if (strcmp(argv[1], "on") == 0)
    {
        if (switch_channel_get_private(channel, "frame_write"))
        {
            switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "UUID:%s Already Running.\n", argv[0]);
            goto done;
        }

        if (!(pvt = (switch_da_t *)switch_core_session_alloc(xsession, sizeof(switch_da_t))))
        {
            switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "UUID:%s alloc pvt failed.\n", argv[0]);
            goto done;
        }
        pvt->xsession = xsession;
        if (strcmp(argv[2], "read") == 0)
        {
            switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_NOTICE, "UUID:%s , IP = %s, port = %s\n", argv[0], argv[3], argv[4]);
            
            //申请本机的ip地址和端口 注意销毁
            switch_find_local_ip(pvt->local_ipv4_str, sizeof(pvt->local_ipv4_str), &mask, AF_INET);

            if (!(pvt->local_port = switch_rtp_request_port(pvt->local_ipv4_str))) {
                    switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "Failed to allocate RTP port for IP %s\n", pvt->local_ipv4_str);
                    goto done;
             }

            switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_NOTICE, "succ allocate RTP port %d for IP %s\n", pvt->local_port, pvt->local_ipv4_str);
            

            
            if (switch_sockaddr_info_get(&pvt->r_sockaddr, argv[3], AF_INET, atoi(argv[4]), 0, switch_core_session_get_pool(xsession)) != SWITCH_STATUS_SUCCESS) {
				switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(session), SWITCH_LOG_ERROR, "Socket Error 1\n");
				release_pvt(pvt);
                goto done;
		}


            //设置本端l_sockaddr
            if (switch_sockaddr_info_get(&pvt->l_sockaddr, pvt->local_ipv4_str, AF_INET, pvt->local_port, 0, switch_core_session_get_pool(xsession)) != SWITCH_STATUS_SUCCESS) {
				switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(session), SWITCH_LOG_ERROR, "Socket Error 1\n");
				release_pvt(pvt);
                goto done;
		    }

            if (switch_socket_create(&pvt->sock, switch_sockaddr_get_family(pvt->r_sockaddr), SOCK_DGRAM, SWITCH_PROTO_UDP,
							 switch_core_session_get_pool(xsession)) != SWITCH_STATUS_SUCCESS) {
				switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "%s create socket error!\n", argv[0]);
				release_pvt(pvt);
                goto done;
		}
            
            if (switch_socket_bind(pvt->sock, pvt->l_sockaddr)!= SWITCH_STATUS_SUCCESS) {
				switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "%s bind socket error!\n", argv[0]);
				release_pvt(pvt);
                goto done;
		    }

            if (switch_socket_connect(pvt->sock, pvt->r_sockaddr) != SWITCH_STATUS_SUCCESS) {
				switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "%s connect %s:%s error!\n", argv[0], argv[3], argv[4]);
				release_pvt(pvt);
                goto done;
		}
        //    if (switch_socket_addr_get(&pvt->l_sockaddr, SWITCH_FALSE, pvt->sock) != SWITCH_STATUS_SUCCESS) {
		// 		switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "%s  get local addr error!\n", argv[0]);
		// 		goto done;
		// }
            // 	switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "get local port %d !\n", switch_sockaddr_get_port(pvt->l_sockaddr));
        //  switch_sockaddr_ip_get(&pvt->local_ip, pvt->l_sockaddr);
            //switch_sockaddr_ip_get(&pvt->remote_ip, pvt->r_sockaddr);
        // pvt->local_port = switch_sockaddr_get_port(pvt->l_sockaddr);
        // pvt->remote_port = switch_sockaddr_get_port(pvt->r_sockaddr);
            switch_core_session_get_read_impl(xsession, &pvt->read_impl);
            if ((status = switch_core_media_bug_add(xsession, "frame_write", NULL, frame_write_callback, pvt, 0, flags, &(pvt->bug))) != SWITCH_STATUS_SUCCESS)
            {
                switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_ERROR, "UUID:%s add bug failed.\n", argv[0]);
                release_pvt(pvt);
                goto done;
            }
        }
        pvt->packet_num = 0; 
        switch_channel_set_private(channel, "frame_write", pvt);
	}else if (strcmp(argv[1], "off") == 0)
   		 {
		    if ((pvt = (switch_da_t *)switch_channel_get_private(channel, "frame_write")))
		    {
		        switch_channel_set_private(channel, "frame_write", NULL);
                release_pvt(pvt);
		        switch_core_media_bug_remove(xsession, &pvt->bug);
		        switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(xsession), SWITCH_LOG_DEBUG, "UUID %s Stop fork media success.\n",argv[0]);
		    }
       }

done:
    switch_safe_free(mycmd);
    if (xsession)
        switch_core_session_rwunlock(xsession);
    return SWITCH_STATUS_SUCCESS;
}



SWITCH_MODULE_LOAD_FUNCTION(mod_frame_write_load)
{
    switch_api_interface_t *api_interface;
    *module_interface = switch_loadable_module_create_module_interface(pool, modname);

    SWITCH_ADD_API(api_interface, "uuid_frame_write", "", uuid_frame_write_function, "");
    switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_DEBUG, " frame_write_load\n");

    return SWITCH_STATUS_SUCCESS;
}

SWITCH_MODULE_SHUTDOWN_FUNCTION(mod_frame_write_shutdown)
{
    switch_log_printf(SWITCH_CHANNEL_LOG, SWITCH_LOG_DEBUG, " frame_write_shutdown\n");

    return SWITCH_STATUS_SUCCESS;
}
```


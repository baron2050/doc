---
title: freeswitch内存播放音频
---
该代码有问题。正常情况下switch_core_session_write_frame是按照ptime写入大小并且延迟的。

fs自带延迟方法

```c
switch_timer_t timer = { 0 };
if (switch_core_timer_init(&timer, "soft", read_impl.microseconds_per_packet / 1000,
							   read_impl.samples_per_packet, switch_core_session_get_pool(session)) != SWITCH_STATUS_SUCCESS) {
		switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(session), SWITCH_LOG_ERROR, "Timer Activation Fail\n");
		switch_channel_set_variable(channel, SWITCH_CURRENT_APPLICATION_RESPONSE_VARIABLE, "Timer activation failed!");
		goto end;
	}
switch_core_session_write_frame(session, &write_frame, SWITCH_IO_FLAG_NONE, 0);
switch_core_timer_next(&timer);
if (timer.interval) {
		switch_core_timer_destroy(&timer);
}
```







```c
SWITCH_DECLARE(switch_status_t) switch_asr_read_file(ali2asr_t *asr, const char *file)
{
    //uint32_t score, count = 0, j = 0;
    //double energy = 0;
    //switch_channel_t *channel = switch_core_session_get_channel(session);
    if(asr==NULL)return SWITCH_STATUS_FALSE;
    switch_channel_t *channel = switch_core_session_get_channel(asr->session);
    
   // int divisor = 0;
    //uint32_t org_silence_hits = silence_hits;
    uint32_t channels;
    //switch_frame_t *read_frame;
    switch_status_t status = SWITCH_STATUS_FALSE;
    //int16_t *data;
    //uint32_t listening = 0;
    //int countdown = 0;
    switch_codec_t raw_codec = { 0 };
    //int16_t *abuf = NULL;
    switch_frame_t write_frame = { 0 };
    switch_file_handle_t fh = { 0 };
    int32_t sample_count = 0;
    switch_codec_implementation_t read_impl = { 0 };
    switch_core_session_get_read_impl(asr->session, &read_impl);

    if (file) {
        if (switch_core_file_open(&fh,
                                  file,
                                  read_impl.number_of_channels,
                                  read_impl.actual_samples_per_second, SWITCH_FILE_FLAG_READ | SWITCH_FILE_DATA_SHORT, NULL) != SWITCH_STATUS_SUCCESS) {
            switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(asr->session), SWITCH_LOG_WARNING, "Failure opening file %s.\n", file);
            switch_core_session_reset(asr->session, SWITCH_TRUE, SWITCH_FALSE);
            return SWITCH_STATUS_NOTFOUND;
        }
        //switch_zmalloc(abuf, SWITCH_RECOMMENDED_BUFFER_SIZE);
        write_frame.data = asr->filebuf;
        write_frame.buflen = SWITCH_RECOMMENDED_BUFFER_SIZE;
    }
    switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(asr->session), SWITCH_LOG_WARNING, "=========================1 Failed to write frame from file %s.   %d    %d\n", file,read_impl.actual_samples_per_second,read_impl.microseconds_per_packet);
    if (switch_core_codec_init(&raw_codec,
                               "L16",
                               NULL,
                               NULL,
                               read_impl.actual_samples_per_second,
                               read_impl.microseconds_per_packet / 1000,
                               1, SWITCH_CODEC_FLAG_ENCODE | SWITCH_CODEC_FLAG_DECODE,
                               NULL, switch_core_session_get_pool(asr->session)) != SWITCH_STATUS_SUCCESS) {

        switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(asr->session), SWITCH_LOG_WARNING, "Failed to initialize L16 codec.\n");
        status = SWITCH_STATUS_FALSE;
    }

    write_frame.codec = &raw_codec;

    channels = read_impl.number_of_channels;
    switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(asr->session), SWITCH_LOG_WARNING, "222222222222Failed to write frame from file %s.\n", file);
    switch_core_session_set_read_codec(asr->session, &raw_codec);
    if (asr->filebuf) {
        switch_size_t olen = raw_codec.implementation->samples_per_packet;

        if (switch_core_file_read(&fh, asr->filebuf, &olen) != SWITCH_STATUS_SUCCESS) {
            //switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(session), SWITCH_LOG_WARNING, "Failed to read file %s.\n", file);
        }
        write_frame.samples = (uint32_t) olen;
        write_frame.datalen = (uint32_t) (olen * sizeof(int16_t) * fh.channels);
        
        //switch_file_read(sent_fd, asr->filebuf, &flen);
        switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(asr->session), SWITCH_LOG_WARNING, "33333333333333Failed to write frame from file %s  %d   %d.\n", file,write_frame.samples,write_frame.datalen);
        asr->play_cursize += olen;
        
        if ((status = switch_core_session_write_frame(asr->session, &write_frame, SWITCH_IO_FLAG_NONE, 0)) != SWITCH_STATUS_SUCCESS) {
           // switch_log_printf(SWITCH_CHANNEL_SESSION_LOG(session), SWITCH_LOG_WARNING, "Failed to write frame from file %s.\n", file);
        }
    }
    switch_core_session_reset(asr->session, SWITCH_FALSE, SWITCH_TRUE);
    switch_core_codec_destroy(&raw_codec);

    //if (abuf) {

        switch_core_file_close(&fh);
       // free(abuf);
    //}

    return status;
}
```


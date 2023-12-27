---
title: FS核心架构
---

## 宏SWITCH_STANDARD_APP 

```c
#define SWITCH_STANDARD_APP(name) static void name
(switch_core_session_t *session, const char *data)
```

如果将上⾯的宏展开，那么 echo_function 的定义就是

```c
static void echo_function(switch_core_session_t *session, const char *data){
switch_ivr_session_echo(session, NULL);
}
```

## loadable_modules 定义如下

```c
struct switch_loadable_module_container {
	switch_hash_t *module_hash;
	switch_hash_t *endpoint_hash;
	switch_hash_t *codec_hash;
	switch_hash_t *dialplan_hash;
	switch_hash_t *timer_hash;
	switch_hash_t *application_hash;
	switch_hash_t *chat_application_hash;
	switch_hash_t *api_hash;
	switch_hash_t *json_api_hash;
	switch_hash_t *file_hash;
	switch_hash_t *speech_hash;
	switch_hash_t *asr_hash;
	switch_hash_t *directory_hash;
	switch_hash_t *chat_hash;
	switch_hash_t *say_hash;
	switch_hash_t *management_hash;
	switch_hash_t *limit_hash;
	switch_hash_t *database_hash;
	switch_hash_t *secondary_recover_hash;
	switch_mutex_t *mutex;
	switch_memory_pool_t *pool;
};
```

它主要定义了各种哈希表（hash ）。将来，新加载的各种模块将统⼀由不同的哈希表 管理，如 mod_sofia 将被记⼊ endpoint_hash ， mod_g729 将被记⼊ codec_hash 等。 此外，它还使⽤互斥（mutex ）来防⽌多线程访问。pool 则是⼀个内存池。

## switch_loadable_module_function_table定义

```c
typedef struct switch_loadable_module_function_table {
	int switch_api_version;
	switch_module_load_t load;
	switch_module_shutdown_t shutdown;
	switch_module_runtime_t runtime;
	switch_module_flag_t flags;
} switch_loadable_module_function_table_t;
```

该结构体定义了⼏个指向函数的指针，分别是load 、shutdown 和runtime 。如果被加载的模块 中实现了这些函数，则这些指针指向相关的函数⼊⼝，如果没有实现，就是 NULL 。

每个模块都必须实现load 函数，它⼀般⽤于模块的初始化操作。因⽽在第 1403 ⾏， load 函数 的指针被赋值给 load_func_ptr 这个变量。load 函数将被执⾏:

```c
if (interface_struct_handle) {
			mod_interface_functions = interface_struct_handle;
			load_func_ptr = mod_interface_functions->load;
		}
		if (load_func_ptr == NULL) {
			err = "Cannot locate symbol 'switch_module_load' please make sure this is a valid module.";
			break;
		}
status = load_func_ptr(&module_interface, pool);//获取到module_interface

if ((module = switch_core_alloc(pool, sizeof(switch_loadable_module_t))) == 0) {
			abort();
}

```

```c
#define SWITCH_MODULE_LOAD_ARGS (switch_loadable_module_interface_t **module_interface, switch_memory_pool_t *pool)
#define SWITCH_MODULE_LOAD_FUNCTION(name) switch_status_t name SWITCH_MODULE_LOAD_ARGS
switch_status_t mod_xx_load(switch_loadable_module_interface_t **module_interface, switch_memory_pool_t *pool)
```

## **switch_loadable_module**

```c
struct switch_loadable_module {
	char *key;
	char *filename;
	int perm;
	switch_loadable_module_interface_t *module_interface;//存放上面生成的module_interface
	switch_dso_lib_t lib;
	switch_module_load_t switch_module_load;
	switch_module_runtime_t switch_module_runtime;
	switch_module_shutdown_t switch_module_shutdown;
	switch_memory_pool_t *pool;
	switch_status_t status;
	switch_thread_t *thread;
	switch_bool_t shutting_down;
	switch_loadable_module_type_t type;
};
```

总之，模块加载的流程就是，⾸先找到模块对应的动态库⽂件，然后打开并找到符号表，接下来执 ⾏模块中的load 函数。另外，如果模块定义了runtime 及shutdown 函数，也将⼀并记录到module结 构的 switch_module_runtime 及switch_module_shutdown 成员变量中。switch_loadable_module_load_file() 执⾏完毕后，得到了⼀个new_module 指针，。紧接着就执⾏switch_loadable_module_process() 函数，它使⽤模块的⽂件名（ file ）和我们新得到的 new_module 结构作为参数传⼊。

```c
	if ((status = switch_loadable_module_process(file, new_module, event_hash)) == SWITCH_STATUS_SUCCESS && runtime) {
			if (new_module->switch_module_runtime) {
				new_module->thread = switch_core_launch_thread(switch_loadable_module_exec, new_module, new_module->pool);
			}
		} else if (status != SWITCH_STATUS_SUCCESS) {
			*err = "module load routine returned an error";
		}
```

```c
new_module->key = switch_core_strdup(new_module->pool, key);
```

该⾏初始化new_module 的 key，它是⼀个字符串，实际上就是传⼊的⽂件名。switch_core_strdup() ⽤
于制作⼀个 key 的副本（duplicate ），它需要的内存是从内存池中申请的，因⽽后续不需要明确的
释放，在模块卸载时直接释放掉内存池就⾏了。

紧接着往loadable_modules中，即模块容器中存module_interface

# **switch_loadable_module_interface**

```c
struct switch_loadable_module_interface {
	/*! the name of the module */
	const char *module_name;
	/*! the table of endpoints the module has implemented */
	switch_endpoint_interface_t *endpoint_interface;
	/*! the table of timers the module has implemented */
	switch_timer_interface_t *timer_interface;
	/*! the table of dialplans the module has implemented */
	switch_dialplan_interface_t *dialplan_interface;
	/*! the table of codecs the module has implemented */
	switch_codec_interface_t *codec_interface;
	/*! the table of applications the module has implemented */
	switch_application_interface_t *application_interface;
	/*! the table of chat applications the module has implemented */
	switch_chat_application_interface_t *chat_application_interface;
	/*! the table of api functions the module has implemented */
	switch_api_interface_t *api_interface;
	/*! the table of json api functions the module has implemented */
	switch_json_api_interface_t *json_api_interface;
	/*! the table of file formats the module has implemented */
	switch_file_interface_t *file_interface;
	/*! the table of speech interfaces the module has implemented */
	switch_speech_interface_t *speech_interface;
	/*! the table of directory interfaces the module has implemented */
	switch_directory_interface_t *directory_interface;
	/*! the table of chat interfaces the module has implemented */
	switch_chat_interface_t *chat_interface;
	/*! the table of say interfaces the module has implemented */
	switch_say_interface_t *say_interface;
	/*! the table of asr interfaces the module has implemented */
	switch_asr_interface_t *asr_interface;
	/*! the table of management interfaces the module has implemented */
	switch_management_interface_t *management_interface;
	/*! the table of limit interfaces the module has implemented */
	switch_limit_interface_t *limit_interface;
	/*! the table of database interfaces the module has implemented */
	switch_database_interface_t *database_interface;
	switch_thread_rwlock_t *rwlock;
	int refs;
	switch_memory_pool_t *pool;
};
```

# switch_core_session_thread_launch

创建一个新的线程，SWITCH_DECLARE(switch_status_t) switch_core_session_thread_launch( switch_core_session_t *session)

# switch_io_routines_t

IO 例程 与 Channel 状态机回调相⽐，Endpoint 模块中更重要的是 IO 例程的回调。IO 例程主要提
供媒体数据的输⼊输出（IO）功能。与上⼀节讲的 Channel 的状态机类似，IO 例程的回调函数是在第
21.3.1 节的第 5552 ⾏注册到核⼼中去的。其中，IO 例程的回调函数是由⼀个 switch_io_routines_t 类
型的结构体变量设置的

# **switch_state_handler_table_t**

Channel 状态机 只要 Channel 的状态⼀变成CS_INIT ，FreeSWITCH 核⼼的状态机代码就负责各种
状态变化了，因⽽，各 Endpoint 模块就不需要再⾃⼰维护状态机了。也就是说，在⼀个 Endpoint 模
块，⾸先要有⼀定的机制可以初始化⼀个 Session（对应⼀个 Channel，它的初始状态将为CS_NEW ），
然后在适当的时候把该 Channel 的状态变成 CS_INIT ，剩下的事就基本不管了。
当然，这⾥说的是基本不⽤管，但⼀般来说，还是要在 Endpoint 模块中跟踪 Channel 状态机
的变化，这就需要靠在核⼼状态机上注册相应的回调函数实现，

# sofia_event_callback

SIP 消息的接收 当我们的服务收到 SIP 消息后，便会调⽤ sofia_event_callback 回调函数。该函
数是在第 1789 ⾏。在该⾏，如果得到回调时，将收到⼀个nua_event_t 结果的 SIP 事件event 。即
使不看 Sofia-SIP 库的⽂档，我们也能从第 1800 ⾏的 switch 语句以及后⾯的 case 分⽀中可以看
出⸺该事件到底是对应什么类型的 SIP 消息了。如果收到 SIP INVITE消息，那么它⼀定会匹配到
第 1840 ⾏。
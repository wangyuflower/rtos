# RTOS SDK 说明文档

Tags: QQ物联 SDK

---

[toc]

---

## SDK基本介绍
**SDK不依赖任何平台。**

由于RTOS系统众多，SDK将所有与`平台相关`的功能模块抽象出来，以`接口`的形式暴露给开发者，开发者只需实现这些接口就可以在任何平台上使用SDK，此层我们称之为：`GKI层`，开发者首先要实现此层。

**SDK由两部分组成 — 静态库(.a文件) 和 头文件。**

*开发者拿到SDK后可以根据头文件的说明来实现GKI层并使用静态库中的导出函数。*

**SDK模型**


**SDK使用步骤**
> * 实现GKI层
> * SDK初始化，完成后SDK能够运行起来
> * 使用SDK提供的功能，包括消息，传文件等

**SDK模块可拆卸**
*为了减少rom及ram的使用，根据产品需要定义了五大类模块，可组合编译。*
> * SIM卡注册模块，针对使用3g等sim卡的设备完成注册绑定；
> * NFC注册模块，使用Wifi方式完成绑定注册；
> * WifiSync模块，非博通芯片支持smartlink方式给设备设置wifi密码；
> * 文件传输模块，系统必须支持线程才可使用；
> * 线程模块，区分系统是否支持线程；


**术语**
> * `GKI层`
> * `WifiSync`等同于`smartlink`

## GKI层接口
## SDK初始化
## SDK功能服务
### DataPoint消息收发
### 语音文件收发
### 入门






头文件按功能分为两种，一种与`SDK功能`相关，另一种与`平台的系统功能`相关：
|  功能 |  文件名 |
| ------| --------|
| SDK功能  | `txd_sdk.h` `txd_error.h` `txd_func_declare.h` |
| 平台的系统功能 | `txd_memory.h` `txd_mutex.h` `txd_store.h` `txd_tcpip.h` `txd_thread.h`  `txd_time.h` |


SDK需要依赖的系统功能有`互斥体`, `线程`, `时间`, `套接字`, `持久化`等。
每个系统功能的实现都由两个结构体组成，均需要开发者来实现。

handler 结构体
:    表示该功能的系统控制块，开发者实现对应平台下的handler版本，handler包含该模块所必须的信息。

options 结构体
:    表示该功能包含的所有操作，开发者实现对应平台下的各个接口函数，使用这些函数来初始化options结构体。

下面拿互斥体`mutex`作为例子，`mutex`在`txd_mutex.h`中的定义如下：
``` C
typedef struct txd_mutex_handler_t txd_mutex_handler_t;

typedef struct txd_mutex_options {
	/*
	 * 创建mutex
	 */
	txd_mutex_handler_t* (*txd_mutex_create)();

	/*
	 * 锁住mutex
	 */
	int32_t (*txd_mutex_lock)(txd_mutex_handler_t *mutex);

	/*
	 * 解锁mutex
	 */
	int32_t (*txd_mutex_unlock)(txd_mutex_handler_t *mutex);

	/*
	 * 销毁mutex
	 */
	int32_t (*txd_mutex_destroy)(txd_mutex_handler_t *mutex);

} txd_mutex_options_t;

```
在`txd_mutex.h`中，`mutex`的`handler结构体`在SDK中并没有定义，仅仅是做了声明，所以需要开发者在实现文件中定义`mutex`的`handler结构体`，并且 **名字必须为txd_mutex_handler_t** 。

`txd_mutex.h`中还定义了`txd_mutex_options`结构体，里面有4个成员变量，全部是函数指针（可以认为是`接口`），由开发者对这4个成员变量进行赋值，SDK将通过这个结构体来使用`mutex`的功能。

下面是`mutex`在WICED平台的实现，在`sys_func_impl_wiced.c`中定义：
``` C
typedef struct txd_mutex_handler_t {
	wiced_mutex_t mutex;
} txd_mutex_handler_t;

txd_mutex_handler_t* txd_mutex_create() {
	txd_mutex_handler_t *pmutex = (txd_mutex_handler_t *)txd_malloc(sizeof(txd_mutex_handler_t));
	if(wiced_rtos_init_mutex(&(pmutex->mutex)) != WICED_SUCCESS) {
		return NULL;
	}
	return pmutex;
}

int32_t txd_mutex_lock(txd_mutex_handler_t *mutex) {
	return wiced_rtos_lock_mutex(&(mutex->mutex));
}

int32_t txd_mutex_unlock(txd_mutex_handler_t *mutex) {
	return wiced_rtos_unlock_mutex(&(mutex->mutex));
}

int32_t txd_mutex_destroy(txd_mutex_handler_t *mutex) {
	wiced_rtos_deinit_mutex(&(mutex->mutex));
	if(mutex != NULL) {
		txd_free(mutex);
	}
	return 0;
}

```
开发者在使用SDK时，需要使用`txd_mutex_options_init`函数来初始化`txd_mutex_options_t`，
`txd_mutex_options_init`在`txd_sdk.h`中定义。

下面初始化`txd_mutex_options_t`
``` C
txd_mutex_options_t mutex_options;

mutex_options.txd_mutex_create = txd_mutex_create;
mutex_options.txd_mutex_lock = txd_mutex_lock;
mutex_options.txd_mutex_unlock = txd_mutex_unlock;
mutex_options.txd_mutex_destroy = txd_mutex_destroy;

txd_mutex_options_init(&mutex_options);

```


### 初始化

下面是在WICED平台上的实现：

从入口函数开始，首先是初始化*GKI（Generical Kernal Interface）*，然后就是执行设备相关的逻辑，包括：初始化设备信息，执行SDK主逻辑。
``` C
void application_start(void) {
    // ...
    
	init_gkl();
	run_device();
	
	// ...
}
```

初始化*GKI*，与平台相关的系统功能模块由开发者自己实现，然后通过函数指针赋值的方式来初始化*SDK*，这样*SDK*就不需要关心底层实现，从而实现SDK与平台分离。
``` C
void init_gkl(){
	// memory
	txd_memory_options_t memory_options;
	memory_options.txd_malloc = txd_malloc;
	memory_options.txd_free = txd_free;
	txd_memory_options_init(&memory_options);

	// thread
	txd_thread_options_t thread_options;
	thread_options.txd_thread_create = txd_thread_create;
	thread_options.txd_thread_destroy = txd_thread_destroy;
	thread_options.txd_thread_join = txd_thread_join;
	thread_options.txd_thread_sleep = txd_thread_sleep;
	txd_thread_options_init(&thread_options);

	// mutex
	txd_mutex_options_t mutex_options;
	mutex_options.txd_mutex_create = txd_mutex_create;
	mutex_options.txd_mutex_lock = txd_mutex_lock;
	mutex_options.txd_mutex_unlock = txd_mutex_unlock;
	mutex_options.txd_mutex_destroy = txd_mutex_destroy;
	txd_mutex_options_init(&mutex_options);

	// time
	txd_time_options_t time_options;
	time_options.txd_time_get_local_ms = txd_time_get_local_ms;
	time_options.txd_time_get_sysclock_ms = txd_time_get_sysclock_ms;
	txd_time_options_init(&time_options);

	// store
	txd_store_options_t store_options;
	store_options.txd_read_basicinfo = txd_read_basicinfo;
	store_options.txd_write_basicinfo = txd_write_basicinfo;
	txd_store_options_init(&store_options);

	// tcpclient
	txd_tcpclient_options_t tcpclient_options;
	tcpclient_options.txd_tcp_socket_create = txd_tcp_socket_create;
	tcpclient_options.txd_tcp_socket_destroy = txd_tcp_socket_destroy;
	tcpclient_options.txd_tcp_connect = txd_tcp_connect;
	tcpclient_options.txd_tcp_disconnect = txd_tcp_disconnect;
	tcpclient_options.txd_tcp_read = txd_tcp_read;
	tcpclient_options.txd_tcp_write = txd_tcp_write;
	txd_tcpclient_options_init(&tcpclient_options);
}

```

初始化与设备相关的信息，包括`product_id`, `product_secret`,`client_pub_key`,`ecdh_key`等，然后执行`txd_sdk_run()`。
``` C
void run_device() {
    // device information
    txd_device_info_t info  = {0};
    info.device_name        = (uint8_t*) "智能网络摄像头";
    info.device_name_len    = strlen(info.device_name);
    info.device_type        = (uint8_t*) "camera";
    info.device_type_len    = strlen(info.device_type);
    info.product_id         = 1000000008;
    info.product_secret     = (uint8_t*)"db212e75daf1bbgba4f0dc30f4289c10";
    info.product_secret_len = strlen(info.product_secret);

    uint8_t CLIENT_PUB_KEY[] = { 3, 90, 169, 68, 110, 197, 21, 137, 246, 99, 102, 41, 194,
            86, 195, 228, 124, 86, 117, 128, 141, 117, 218, 185, 3 };
    uint8_t ECDH_KEY[] = { 175, 165, 11, 164, 103, 228, 240, 23, 167, 105, 65, 60,
            207, 190, 38, 50 };

    info.client_pub_key = CLIENT_PUB_KEY;
    info.client_pub_key_len = sizeof(CLIENT_PUB_KEY);

    info.ecdh_key = ECDH_KEY;
    info.ecdh_key_len = sizeof(ECDH_KEY);

    uint8_t license[] = "3044022034110515B65962DD28B111660750C8D99C165DE060206D75B5622ABD1A890720022044F3CDB5AE5FE46268CB92FE5F9618F4BFCECAB63DF8208FA57C4500A0233BCE";
    int nLicenseSize = strlen(license);
    uint8_t guid[] = "d6f4b833-cf90-42";
    int nGUIDSize = strlen(guid);

    info.device_license           = license;
    info.device_license_len       = nLicenseSize;
    info.device_serial_number     = guid;
    info.device_serial_number_len = nGUIDSize;

    int32_t ret = txd_device_info_init(&info);
    if (ret != err_success){
        return;
    }

    // 内部会启动一个独立线程去执行相关逻辑，SDK线程将伴随整个进程生命周期
    ret = txd_sdk_run(); 
    if (ret != err_success){
           return;
    }

    // 主线程不能退出
    while (1) {
        txd_thread_sleep(5000);
    }
}

```


### 消息收发

**接收`datapoint`格式的消息**
首先在初始化SDK时设置一个回调函数，SDK在收到`datapoint`消息后将会调用这个函数。

    定义：
``` C
// 自定义的回调函数
typedef void (*on_receive_datapoint_msg)(uint64_t u64SenderId, uint32_t propertyId, uint8_t *pBuffer, uint32_t u32BufLen);

// 初始化datapoint回调
void txd_init_datapoint(const on_receive_datapoint_msg pCb);
```

    实现：
``` C
void test_on_receive_datapoint_msg(uint64_t u64SenderId, uint32_t propertyId, uint8_t *, uint32_t u32BufLen) {
	uint8_t buf[1024] = {0};
	memcpy(buf, pBuffer, u32BufLen);
	printf("u64SenderId[%d], propertyID[%d], u32BufLen[%d], pBuffer[%s]\n", u64SenderId, propertyId, u32BufLen, buf);
}
```
``` C
txd_init_datapoint(test_on_receive_datapoint_msg);
```

**发送`datapoint`格式的消息**

    定义：
``` C
// 发送datapoint消息完成的回调，SDK在收到server的datapoint回包后将调用该函数
typedef void (*on_send_msg)(int err_code, uint32_t cookie);

// 发送datapoint消息的函数，pMsgText是以'\0'结尾的字符串
int32_t txd_send_structuring_msg(uint32_t propertyId, uint8_t *pMsgText, on_send_msg pCb, uint32_t *pCookie);
```

    实现：
``` C
void test_on_send_msg(int err_code, uint32_t cookie) {
    printf("err_code[%d], cookie[%d]\n", err_code, cookie);
}
```
``` C
int cookie = 0;
int ret = txd_send_structuring_msg(10007, "this ia a datapoint message", test_on_send_msg, &cookie);
```


## SDK功能

- 设备绑定

- 消息发送

- 文件传输

- 视频语音

- 好友分享

- ......


## 常见问题

1. 问题1

2. 问题2

3. 问题3



## 联系我们

xxx@qq.com

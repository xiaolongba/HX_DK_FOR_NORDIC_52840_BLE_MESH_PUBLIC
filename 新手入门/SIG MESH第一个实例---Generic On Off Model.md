<!--
 * @Description: In User Settings Edit
 * @Author: 临时工
 * @Date: 2019-09-16 22:34:52
 * @LastEditTime : 2020-01-20 21:38:53
 * @LastEditors  : Please set LastEditors
 -->
# 前言
通过前面的理论铺垫，我相信大部分读者应该对SIG MESH有了一定的了解了吧；因为前面的几个章节都比较偏向于理论，所以该章节我们配合示例工程来巩固并加深对SIG MESH的一些概念以及流程的了解。

# Light Switch Server
本章小编选择最简单的标准模型Generic On Off为例，将前面讲解到的理论知识在本示例工程中尽可能多地穿插讲解。在开始之前，我们先看看主函数由哪几部分组成：
```c
int main(void)
{
    initialize();
    start();

    for (;;)
    {
        (void)sd_app_evt_wait();
    }
}
```
- initialize

    - app_timer初始化
    - 硬件相关初始化
    - GAP以及连接参数初始化
    - Mesh初始化
- start

    - 填充被配置者配置过程的参数并开始启动配置以及等待配置
    - 根据配置的参数，基于Mesh协议栈启动相关的配置流程

## initialize
首先，app_timer以及硬件相关初始化就没有什么好说的了。前者，就是定义一个低功耗定时器；后者，就是硬件相关的初始化；因此，我们这里着重看看GAP以及连接参数和MESH初始化是做了些什么事情。
### GAP以及连接参数初始化
- gap_params_init

    ```c
    void gap_params_init(void)
    {
        uint32_t                err_code;
        ble_gap_conn_sec_mode_t sec_mode;
        ble_gap_conn_params_t  gap_conn_params;

        BLE_GAP_CONN_SEC_MODE_SET_OPEN(&sec_mode);

        err_code = sd_ble_gap_device_name_set(&sec_mode,
                                            (const uint8_t *) GAP_DEVICE_NAME,
                                            strlen(GAP_DEVICE_NAME));
        APP_ERROR_CHECK(err_code);

        memset(&gap_conn_params, 0, sizeof(gap_conn_params));
        GAP_CONN_PARAMS_INIT(gap_conn_params);

        err_code = sd_ble_gap_ppcp_set(&gap_conn_params);
        APP_ERROR_CHECK(err_code);
    }
    ```

    从上面的代码片段可知，该函数主要是设置**Device Name**为"nRF5x Mesh Light"以及推荐的连接参数配置为如下所示：
    - MIN_CONN_INTERVAL = 150ms
    - MAX_CONN_INTERVAL = 250ms
    - SLAVE_LATENCY = 0
    - CONN_SUP_TIMEOUT = 4000ms

    **注意，上面的连接参数只是给主机参考的并不是当前的从机的连接参数。**
- conn_params_init
    ```c
    void conn_params_init(void)
    {
        uint32_t               err_code;
        ble_conn_params_init_t cp_init;
        ble_gap_conn_params_t  gap_conn_params;

        memset(&gap_conn_params, 0, sizeof(gap_conn_params));
        GAP_CONN_PARAMS_INIT(gap_conn_params);

        memset(&cp_init, 0, sizeof(cp_init));
        cp_init.p_conn_params                  = &gap_conn_params;
        cp_init.first_conn_params_update_delay = FIRST_CONN_PARAMS_UPDATE_DELAY;
        cp_init.next_conn_params_update_delay  = NEXT_CONN_PARAMS_UPDATE_DELAY;
        cp_init.max_conn_params_update_count   = MAX_CONN_PARAMS_UPDATE_COUNT;
        cp_init.start_on_notify_cccd_handle    = BLE_GATT_HANDLE_INVALID;
        cp_init.disconnect_on_fail             = false;
        cp_init.evt_handler                    = on_conn_params_evt;
        cp_init.error_handler                  = conn_params_error_handler;

        err_code = ble_conn_params_init(&cp_init);
        APP_ERROR_CHECK(err_code);
    }
    ```
    该部分内容跟上述的**gap_params_init**有一些雷同，主要的不同点是，该代码片段填充了更新连接参数的参数。起到的作用是在首次连接成功之后延时100ms，由从机主动向主机发起连接更新请求,如果填充的连接参数不被主机接受，那么再次协商连接参数时则要等待2000ms，直至3次协商次数用完仍然协商不成功则断开连接。
- mesh_init
    ```c
    static void mesh_init(void)
    {
        mesh_stack_init_params_t init_params =
        {
            .core.irq_priority       = NRF_MESH_IRQ_PRIORITY_LOWEST,
            .core.lfclksrc           = DEV_BOARD_LF_CLK_CFG,
            .core.p_uuid             = NULL,
            .models.models_init_cb   = models_init_cb,
            .models.config_server_cb = config_server_evt_cb
        };
        ERROR_CHECK(mesh_stack_init(&init_params, &m_device_provisioned));
    }
    ```
    这里主要是填充了应用层指定的模块 **(也就是说除了基础模型外的所有其他模型，本章节指的是Generic-On-Off模型)** 的初始化回调函数、**config_server**事件的回调处理函数以及低速晶振的配置。而基础模型config_server以及health_server则放在**mesh_stack_init**中初始化，关于基础模块的详解我们放在后面的章节讲解；总得来说就是，该代码片段初始化了所有的mesh协议栈模型以及基础模型。
## start
接下来我进入看起来很简单但却是最重要、最复杂的步骤，该函数完成了我们MESH启动时的一些关键参数、相关事件的回调处理函数地填充以及开始启动Mesh协议栈等等动作；废话不多说，先上代码。    
```c
static void start(void)
{
    rtt_input_enable(app_rtt_input_handler, RTT_INPUT_POLL_PERIOD_MS);

    if (!m_device_provisioned)
    {
        static const uint8_t static_auth_data[NRF_MESH_KEY_SIZE] = STATIC_AUTH_DATA;
        mesh_provisionee_start_params_t prov_start_params =
        {
            .p_static_data    = static_auth_data,
            .prov_complete_cb = provisioning_complete_cb,
            .prov_device_identification_start_cb = device_identification_start_cb,
            .prov_device_identification_stop_cb = NULL,
            .prov_abort_cb = provisioning_aborted_cb,
            .p_device_uri = EX_URI_LS_SERVER
        };
        ERROR_CHECK(mesh_provisionee_prov_start(&prov_start_params));
    }

    mesh_app_uuid_print(nrf_mesh_configure_device_uuid_get());

    ERROR_CHECK(mesh_stack_start());

    hal_led_mask_set(LEDS_MASK, LED_MASK_STATE_OFF);
    hal_led_blink_ms(LEDS_MASK, LED_BLINK_INTERVAL_MS, LED_BLINK_CNT_START);
}
```
该函数主要是填充启动配置时所使用的相关参数及事件回调处理函数，其中会携带16字节的静态认证数据用于在认证阶段时使用，但是就算这里填充了，在后继的认证阶段也可以不用，这个取决于Authentication method字段的内容。至于各个回调函数的作用，如下所示：
- **provisioning_complete_cb**表明入网完成之后会被触发的回调函数；
- **device_identification_start_cb**表明被配置的设备现在可以展示其自己了，从而可以引起配置者知道当前配置的是哪个设备；
- **provisioning_aborted_cb**表明当入网过程被中断时触发的事件回调；
- URI的相关内容，可以参考[Mesh Beacon帧格式](/%E6%96%B0%E6%89%8B%E5%85%A5%E9%97%A8/Mesh%20Beacon%E5%B8%A7%E6%A0%BC%E5%BC%8F.md)的**URI Hash**章节的内容。

配置完上述的设置之后，就可以调用 **mesh_stack_start()** 函数开启并响应mesh的相关动作行为了。然而，我们需要注意的是 **mesh_stack_start()** 仅仅开启了响Mesh协议栈的相关行为，例如：响应Mesh的事件并触发相应的回调函数。而Mesh的Beacon则是在 **mesh_provisionee_prov_start**被调用之后，它才会被广播出去。
# 入网过程细述
基本上该示例工程的框架就是这样，先<code>初始化，包括硬件、BLE以及BLE Mesh</code>-><code>填充MESH协议栈开放给应用层相关的回调处理函数以及启动MESH的相关事件行为</code>；正常来说，我们只需要好好处理在**start()** 中的回调函数即可。然而，如果仅仅是如此的话小编认为这是完全不够的：

- 如果你想要更改响应邀请时的capabilities数据包，你要在哪里修改？
- 如果在入网的过程中认证失败了，你知道在哪里跟踪调试这个问题吗？
- 又或者你想要更改认证的方法为输入，你又知道在哪里更改吗？
- 总得来说，你要修改整个入网过程中的任何一个步聚中的其中一个细节的话，你要知道在哪里可以修改而不是采用默认的方式。

    ![](../Material%20library/more_power.jpg)

让我们再次回顾一下基于PB-GATT的整个入网过程是怎么样的。

![](../Material%20library/The_process_for_provisioning.png)

## Beacon
显然，在开始入网之前unprovisioned device应该发出unprovisioned beacon；但是，我们在上面的分析过程中并没有看到有接口让应用层设置Beacon的信息内容；万一我们非要强行修改呢？又或者说我们需要修改beacon之间的间隔呢？那么我们带着这些疑问在下面这个函数中找找答案：
```c
static void send_unprov_beacon(nrf_mesh_prov_bearer_adv_t * p_pb_adv, const char * URI, uint16_t oob_info)
{
    adv_packet_t * p_packet = prov_beacon_unprov_build(&p_pb_adv->advertiser, URI, oob_info);
    /* Since this is the only thing we're doing with the advertiser at this point, we should never
     * fail to allocate the packet. */
    NRF_MESH_ASSERT(p_packet != NULL);
    p_packet->config.repeats = ADVERTISER_REPEAT_INFINITE;
    p_packet->token = TX_TOKEN_UNPROV_BEACON;
    advertiser_interval_set(&p_pb_adv->advertiser, NRF_MESH_PROV_BEARER_ADV_UNPROV_BEACON_INTERVAL_MS);

    advertiser_packet_send(&p_pb_adv->advertiser, p_packet);
}
```
我们从上述函数的关键字**static**就知道，一般情况下我们是无法直接调用的，也就是说其不对外开放给应用层使用。由**prov_provisionee_listen()函数**间接调用。但是，我们基本上可以知道是哪个函数开启了unprovisioned beacon。同时，我们也了解到beacon的间隔是2秒（这点我们可以从**NRF_MESH_PROV_BEARER_ADV_UNPROV_BEACON_INTERVAL_MS**可知），当然我们从捉包中也可以验证这么一点：

![](../Material%20library/unprovisioned_beacon_interval.png)

讲到这里，如果下次我们要修改unprovisioned beacon的间隔就可以通过修改宏**NRF_MESH_PROV_BEARER_ADV_UNPROV_BEACON_INTERVAL_MS**实现；至于，unprovisioned beacon的payload我们能修改的也只有**URI**以及**OOB information**两项内容了。基本上 **send_unprov_beacon()** 函数只是拼装并发出beacon信息，我们可以通过下图验证拼装的beacon数据包：

- 代码

    ![](../Material%20library/unprovisioned_beacon_payload_ses.png)

- 捉包
    ![](../Material%20library/unprovisioned_beacon_payload_ellisys.png)

## 入网过程
继beacon信号出来之后，接下来provisioner捕获到unprovisioned beacon之后，就要开始发出邀请等一序列的操作。那么Nordic Mesh SDK又是在哪里处理并响应这些事件呢？同样的，让我们带着这样或者那样的疑问来看看下面的事件回调函数：
```c
static const prov_bearer_callbacks_t m_prov_callbacks =
 {
 .rx = prov_provisionee_pkt_in, 
 .ack = prov_provisionee_cb_ack_received,
 .opened = prov_provisionee_cb_link_established,
 .closed = prov_provisionee_cb_link_closed
 };
 ```
 显而易见，整个入网过程中所触发的大部分事件均在**prov_provisionee_pkt_in**和**prov_provisionee_cb_ack_received**两个回调函数中处理。至于，接下来所涉及的不同PDU的帧格式详情，请参考[PB-GATT入网过程](/%E6%96%B0%E6%89%8B%E5%85%A5%E9%97%A8/PB-GATT%E5%85%A5%E7%BD%91%E8%BF%87%E7%A8%8B.md)。
### 邀请
当unprovisioned beacon被provisioner成功捕获到之后，就会向其发出邀请的PDU，那么发具体接收到的Invite PDU具体又是什么内容呢？让我们一起来看看：

![](../Material%20library/p_invite_pdu.png)
其中，接收到的邀请PDU与捉包所捕获的完全一样：

![](../Material%20library/p_invite_pdu_ellisys.png)

我们从上述的两幅图中可知，其中**pdu_type**为0表示当前的类型是**PROV_PDU_TYPE_INVITE**，而attention_time则为provisionee用于产生任何可以引起注意的动作时长。

### Capabilities响应
接收到provisioner的邀请之后，provisionee必须在**NRF_MESH_PROV_LINK_TIMEOUT_MIN_US秒内(最小为60秒)**，将自身支持的Capabilities告诉provisioner，否则入网失败，provisioner会主动断开连接。那么，在SDK工程中的哪里回复或者填充这些内容呢？至于Capabilities PDU格式及含义，我们在[PB-GATT入网过程](/%E6%96%B0%E6%89%8B%E5%85%A5%E9%97%A8/PB-GATT%E5%85%A5%E7%BD%91%E8%BF%87%E7%A8%8B.md)已经细述，这里我们不再重复。
```c
static uint32_t send_capabilities(nrf_mesh_prov_ctx_t * p_ctx)
{
    prov_pdu_caps_t pdu;
    pdu.pdu_type = PROV_PDU_TYPE_CAPABILITIES;

    pdu.num_elements = p_ctx->capabilities.num_elements;
    pdu.algorithms = LE2BE16(p_ctx->capabilities.algorithms);
    pdu.pubkey_type = p_ctx->capabilities.pubkey_type;
    pdu.oob_static_types = p_ctx->capabilities.oob_static_types;
    pdu.oob_output_size = p_ctx->capabilities.oob_output_size;
    pdu.oob_output_actions = LE2BE16(p_ctx->capabilities.oob_output_actions);
    pdu.oob_input_size = p_ctx->capabilities.oob_input_size;
    pdu.oob_input_actions = LE2BE16(p_ctx->capabilities.oob_input_actions);

#if PROV_DEBUG_MODE
    __LOG(LOG_SRC_PROV, LOG_LEVEL_INFO, "Provisionee: sending capabilities\n");
#endif

    return prov_tx_capabilities(p_ctx->p_active_bearer, &pdu, p_ctx->confirmation_inputs);
}
```
![](../Material%20library/Provisionee_Capabilities_pdu.png)

从上述内容可知，**send_capabilities()** 回复了provisioner的邀请PDU，同时我们也应该注意到该函数是由关键字**static**修饰的；因此，我们不能直接在该函数修改相对应的capabilities参数。那么，这些参数又是从哪里传进来的呢？我们可以在**mesh_provisionee_prov_start()** 函数中找到答案：
```c
nrf_mesh_prov_oob_caps_t prov_caps =
{
    ACCESS_ELEMENT_COUNT,
    NRF_MESH_PROV_ALGORITHM_FIPS_P256EC,
    0,
    NRF_MESH_PROV_OOB_STATIC_TYPE_SUPPORTED,
    0,
    0,
    0,
    0
};
```
![](../Material%20library/Capabilites_app.png)

在**mesh_provisionee_prov_start()**函数中定义了一个**nrf_mesh_prov_oob_caps_t**类型的变量**prov_caps**，所以在**send_capabilities()** 中的关于Capabilities均在这里传过去的，也就是说数据的源头在**mesh_provisionee_prov_start()** 函数。此时，如果我们想要修改provisionee的Capabilities就可以修改**prov_caps**变量值即可。

### Provisioning Start
由上述的入网过程示意图可知，该PDU由Provisioner向provisionee发送，其主要作用就是根据provisionee回复的capabilities内容，决定接下来的公钥交换和认证采用哪种方式，具体携带参数如下所示：
- No OOB

    ![](../Material%20library/Provisioning_Start_Pdu.png)

    ![](../Material%20library/Provisioning_Start_Pdu_ses.png)

- OOB 
    ![](../Material%20library/Provisioning_Start_OOB.png)

我们可以看出这里有**No OOB**和**OOB**两个选项，这里也就验证了我们在[start](#start)中描述的**static_auth_data**；前者则直接进入公钥交换过程紧接着进入下一个阶段，而后者则进入公钥交换过程后，还需要再进行静态OOB认证才会进入一个阶段。至于是Static、Input、Output的OOB认证方式，则完全取决于**Authentication method**,而**Authentication method**又取决于provisionee回复provisioner的Capabilities的内容，它们之间是相到关联的。

**还有一点，小编觉得需要说明的是上述的内容跟OOB Public Key是两个不同的概念，前者用于交互公钥之后的认证而后者则是决定是否采用OOB的方式来交换公钥**。

### Provisioning public key
此时，provisioner率先发送其生成的64Bytes的公钥给provisionee，而provisionee则回复其生成的公钥给provisioner，而provisionee的公钥和私钥则是在**provisionee_start()** 函数中生成。

```c
static uint32_t provisionee_start(void)
{
    uint32_t bearers = 0;

#if MESH_FEATURE_PB_ADV_ENABLED
    bearers = NRF_MESH_PROV_BEARER_ADV;
#endif

#if MESH_FEATURE_PB_GATT_ENABLED
    bearers |= NRF_MESH_PROV_BEARER_GATT;
#endif
    /* Re-generate the keys each round. */
    RETURN_ON_ERROR(nrf_mesh_prov_generate_keys(m_public_key, m_private_key));
    return nrf_mesh_prov_listen(&m_prov_ctx, m_params.p_device_uri, 0, bearers);
}
```
这里我们应该注意的是生成的公钥和私钥都是随机的，也就是说如果该次入网失败，那么再次入网时生成的公钥和私钥均跟上次不同。

### Provisioning Input Complete
这个也是认证的方式之一，当provisionee输入相关用于认证的数据之后，就会向provisioner发送该pdu，而是否激活该认证则取决于**Authentication method**。

| Value | Description|
|-----|-----|
|0x00|No OOB authentication is used |
|0x01|Static OOB authentication is used|
|0x02|Output OOB authentication is used|
|0x03|Input OOB authentication is used|
|0x04-0xFF|Prohibited|

如[Provisioning Start](#Provisioning-Start)可知，当静态OOB认证方法被选中时，那么**provisioning start PDU**就会将**Authentication method**的值置为0x01。当然，这是要根据provisionee回复的Capabilities，从而做出相对应的选择。

**！！！**
说到oob_input_actions，那么小编这里就不得不提一下oob_output_actions，根据小编的调试发现当**authentication method**为0x02，且相对应的**Output OOB Action**为Output Numeric时，当前的Mesh SDK转换出来的Authentication value有误，导致入网时输入认证值一直提示认证值错误，后来小编凭借精湛的技术偷偷把这个Bug修复了:smile:。

- 原始的代码
    ```c
    static void oob_gen_numeric(uint8_t * p_auth_value, uint8_t digits)
    {
        uint32_t number;
        rand_hw_rng_get((uint8_t *) &number, sizeof(number));
        __LOG(LOG_SRC_APP, LOG_LEVEL_INFO, "rand_hw_rng_get = %ld\n", number);
        __LOG(LOG_SRC_APP, LOG_LEVEL_INFO, "number % m_numeric_max[digits] = %d\n", number % m_numeric_max[digits]);
        /* Big endian at end of auth value: */
        number = LE2BE32((number % m_numeric_max[digits]));
        __LOG(LOG_SRC_APP, LOG_LEVEL_INFO, "LE2BE32 = 0x%04x\n", number);
        memcpy(&p_auth_value[PROV_AUTH_LEN - sizeof(number)], &number, sizeof(number));
    }
    ```
- 修改后的代码
    ```c
    static void oob_gen_numeric(uint8_t * p_auth_value, uint8_t digits)
    {
        uint32_t number;
        rand_hw_rng_get((uint8_t *) &number, sizeof(number));
        number = number % m_numeric_max[digits];
        for(uint8_t i = 0;i<digits;i++)
        {
            p_auth_value[i] = number/(uint32_t)pow(10,digits-1-i)+0x30;
            number = number%(uint32_t)pow(10,digits-1-i);
            __LOG(LOG_SRC_APP, LOG_LEVEL_INFO, "%d\n", number);
        }        
    }
    ```

### Provisioning confirmation
当程序运行到Provisioning confirmation交互时，入网过程已经即将接近尾声了。那么，互发Provisioning confirmation的作用是什么呢？看过小编上一章节写的[PB-GATT入网过程](/%E6%96%B0%E6%89%8B%E5%85%A5%E9%97%A8/PB-GATT%E5%85%A5%E7%BD%91%E8%BF%87%E7%A8%8B.md)就可以知道，该PDU主要将目前为止provisionee和provisioner交互过的所有PDU和OOB认证生成的authentication value以及即将产生的random加密后生成confirmation value希哈值，对方收到之后就会将该值暂存，用于下一个阶段收到Provisioning Random值之后，再使用同样的方法将目前为止收到的所有PDU和OOB认证生成的authentication value再加上收到的Random值，计算得出confirmation value的哈希值与接收的进行对比。而SDK是在哪个函数实现这个功能的呢？

```c
void prov_utils_authentication_values_derive(const nrf_mesh_prov_ctx_t * p_ctx,
        uint8_t * p_confirmation_salt,
        uint8_t * p_confirmation,
        uint8_t * p_random)
{
    create_confirmation_salt(p_ctx, p_confirmation_salt);

    uint8_t confirmation_key[NRF_MESH_KEY_SIZE];

    /* ConfirmationKey = k1(ECDHSecret, ConfirmationSalt, "prck") */
    enc_k1(p_ctx->shared_secret,
           NRF_MESH_ECDH_SHARED_SECRET_SIZE,
           p_confirmation_salt,
           CONFIRMATION_KEY_INFO, CONFIRMATION_KEY_INFO_LENGTH,
           confirmation_key);

    rand_hw_rng_get(p_random, PROV_RANDOM_LEN);
    
    uint8_t random_and_auth[PROV_RANDOM_LEN + PROV_AUTH_LEN];
    memcpy(&random_and_auth[0], p_random, PROV_RANDOM_LEN);
    memcpy(&random_and_auth[PROV_RANDOM_LEN], p_ctx->auth_value, PROV_AUTH_LEN);

    /* Confirmation value = AES-CMAC(ConfirmationKey, LocalRandom || AuthValue) */
    enc_aes_cmac(confirmation_key,
                 random_and_auth,
                 PROV_RANDOM_LEN + PROV_AUTH_LEN,
                 p_confirmation);
}
```

![](../Material%20library/provisioning_confirmation.png)


正所谓一图胜千言，上图所述的内容基本上将 **prov_utils_authentication_values_derive()** 函数中所表达的意图已经阐述了，同时也将整个**Provisioning Confirmation**的内容也涵括了。

### Provisioning random
现在我们再看这个PDU时，我们就可以很好的理解了甚至有种豁然开朗的感觉。此时，当收到provisioner的random值后，就会利用该值计算出confirmation value并且与接收到对比，如果对比成功，则开始分发网络秘钥等相关的入网通行证内容；否则，告诉应用层入网失败的原因。

```c
bool prov_utils_confirmation_check(const nrf_mesh_prov_ctx_t * p_ctx)
{
    uint8_t confirmation_key[NRF_MESH_KEY_SIZE];

    /* ConfirmationKey = k1(ECDHSecret, ConfirmationSalt, "prck") */
    enc_k1(p_ctx->shared_secret,
           NRF_MESH_ECDH_SHARED_SECRET_SIZE,
           p_ctx->confirmation_salt,
           CONFIRMATION_KEY_INFO, CONFIRMATION_KEY_INFO_LENGTH,
           confirmation_key);

    uint8_t random_and_auth[PROV_RANDOM_LEN + PROV_AUTH_LEN];
    memcpy(&random_and_auth[0], p_ctx->peer_random, PROV_RANDOM_LEN);
    memcpy(&random_and_auth[PROV_RANDOM_LEN], p_ctx->auth_value, PROV_AUTH_LEN);

    /* Confirmation value = AES-CMAC(ConfirmationKey, LocalRandom || AuthValue) */
    uint8_t confirmation[PROV_CONFIRMATION_LEN];
    enc_aes_cmac(confirmation_key,
                 random_and_auth,
                 PROV_RANDOM_LEN + PROV_AUTH_LEN,
                 confirmation);

    return memcmp(confirmation, p_ctx->peer_confirmation, sizeof(confirmation)) == 0;
}
```

上述的文字说明就是阐述了上述函数的整个过程，再结合[Provisioning confirmation](#Provisioning-confirmation)，我们可以很清楚地了解到在provisioning random匹配正确之后，才会将自己的random发送给provisioner；同样的，provisioner使用同样的方法匹配正确之后，才会开始发送Provisioning data的内容。还有一个重要的点就是，当随机数发送完毕紧接着provisionee会生成**Session Key、data_nonce、Device Key**，其中前面两者用于解密Provisioner发送的Provisioning Data的内容，而**Device Key**仅节点和Configuration Client知道，用于加解密节点和Configuration Client的通信。例如，发送**NODE RESET**命令。

**！！！注意：** 上述中有提到这个**Nonce**，可能很多人不太理解这个是什么意思？起初，小编也不理解这个是什么鬼玩意。但是，后继搜索得到：
> Nonce，Number used once或Number once的缩写，在密码学中Nonce是一个只被使用一次的任意或非重复的随机数值，在加密技术中的初始向量和加密散列函数都发挥着重要作用，在各类验证协议的通信应用中确保验证信息不被重复使用以对抗重放攻击(Replay Attack)。---出自[《百度百科》](https://baike.baidu.com/item/Nonce/2525414?fr=aladdin)

![](../Material%20library/session_device_nonce_key.png)

### Provisioning Data
前面[Provisioning random](#Provisioning-random)我们也提到，当provisionee接收到provisioner的provisioning data时，它会使用session key和data_nonce来解密，从而获取Network Key相关的内容，如下所示：

| Field | Size(octets)|Notes|
|-----|-----|-----|
|Network Key|16|NetKey|
|Key Index|2|Index of the NetKey|
|Flags|1|Flags bitmask|
|IV Index|4|Current value of the IV Index|
|Unicast Address|2|Unicast address of the primary element|

![](../Material%20library/provisioning_data.png)

而所有这些解密的工作均在函数**handle_data()** 中处理。
```c
static uint32_t handle_data(nrf_mesh_prov_ctx_t * p_ctx, const uint8_t * p_buffer)
{
    const prov_pdu_data_t * p_pdu = (const prov_pdu_data_t *) p_buffer;

    prov_pdu_data_t unencrypted_pdu;

    ccm_soft_data_t ccm_data;
    ccm_data.p_key = p_ctx->session_key;
    ccm_data.p_nonce = p_ctx->data_nonce;
    ccm_data.p_m = (uint8_t *) &p_pdu->data;
    ccm_data.m_len = sizeof(prov_pdu_data_block_t);
    ccm_data.p_out = (uint8_t *) &unencrypted_pdu.data;
    ccm_data.p_mic = (uint8_t *) p_pdu->mic;
    ccm_data.p_a = NULL;
    ccm_data.a_len = 0;
    ccm_data.mic_len = PROV_PDU_DATA_MIC_LENGTH;

    bool mic_passed = false;
    enc_aes_ccm_decrypt(&ccm_data, &mic_passed);
    if (!mic_passed)
    {
#if PROV_DEBUG_MODE
        __LOG(LOG_SRC_PROV, LOG_LEVEL_ERROR, "Provisionee: provisioning data could not be authenticated!\n");
#endif
        return NRF_ERROR_INVALID_DATA;
    }

    memcpy(p_ctx->data.netkey, unencrypted_pdu.data.netkey, NRF_MESH_KEY_SIZE);
    p_ctx->data.iv_index = BE2LE32(unencrypted_pdu.data.iv_index);
    p_ctx->data.address = BE2LE16(unencrypted_pdu.data.address);
    p_ctx->data.netkey_index = BE2LE16(unencrypted_pdu.data.netkey_index);
    p_ctx->data.flags.iv_update = unencrypted_pdu.data.flags.iv_update;
    p_ctx->data.flags.key_refresh = unencrypted_pdu.data.flags.key_refresh;
    return NRF_SUCCESS;
}
```
最后，解密且认证provisioning data通过之后，provisionee给provisioner发送provisioning complete数据包，否则发送入网失败的原因。至此，整个入网过程完成结束，接下来就可以愉快地进行模型之间的通信了。

### Provisioning Complete or Failed
参考[Provisioning Data](#Provisioning-Data)。

# 应用层处理函数
入网过程结束之后，我们就可以通过模型Generic On Off来控制设备了，而本篇章节所述的Light Switch Server示例工程中，相对应的控制回调函数如下所示：

```c
/* Callback for updating the hardware state */
static void app_onoff_server_set_cb(const app_onoff_server_t * p_server, bool onoff)
{
    /* Resolve the server instance here if required, this example uses only 1 instance. */

    __LOG(LOG_SRC_APP, LOG_LEVEL_INFO, "Setting GPIO value: %d\n", onoff)

    hal_led_pin_set(ONOFF_SERVER_0_LED, onoff);
}

/* Callback for reading the hardware state */
static void app_onoff_server_get_cb(const app_onoff_server_t * p_server, bool * p_present_onoff)
{
    /* Resolve the server instance here if required, this example uses only 1 instance. */

    *p_present_onoff = hal_led_pin_get(ONOFF_SERVER_0_LED);
}
```

# 最后
通过上述大篇幅的描述，我想读者此时应该对入网过程已经了如指掌了，同时也对前面所述的几个篇章内容有了更加深刻地认识。在小编看来，起码以后对入网这个过程再也无须担心了，一切都在掌控之中。

![](../Material%20library/under_control.gif)
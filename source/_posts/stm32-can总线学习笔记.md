title: stm32 can总线学习笔记
date: 2021-02-26 17:56:46
tags:
    - 技术相关
    - 嵌入式
---

这段时间折腾stm32与树莓派之间的can总线通讯遇到了不少问题,树莓派那端的已经写在[树莓派外挂MCP2515模块爬坑记录](https://blog.islinjw.cn/2021/02/09/%E6%A0%91%E8%8E%93%E6%B4%BE%E5%A4%96%E6%8C%82MCP2515%E6%A8%A1%E5%9D%97%E7%88%AC%E5%9D%91%E8%AE%B0%E5%BD%95/)里面了。这次来总结下CAN总线协议和讲讲stm32如何使用CAN总线。

# can总线协议基础

首先我们来大概看看CAN总线协议是怎样的。

完整的CAN电路是由CAN控制器和CAN收发器组成的。协议相关的内容由CAN控制器完成。CAN 控制器和CAN收发器用CAN TX和CAN RX两根线传输TTL电平信号。低电平代表二进制的0,高电平代表二进制的1。

CAN H 和CAN L就是CAN总线,所有设备的CAN 收发器都会挂在这两根线上。数据通过差分信号在这两个线间传输:

- CAN H - CAN L < 0.5V 表示二进制的1
- CAN H - CAN L > 0.9V 表示二进制的0

为了避免信号的反射和干扰，还需要在CAN H和CAN L之间接上120欧姆的终端电阻。

{% img /stm32can学习笔记/1.png %}

CAN收发器将CAN控制器通过CAN TX线传来的二进制码流转换为差分信号用CAN H、CAN L两根线发送出去。同时接收端设备的CAN接收器会监听CAN H、CAN L两根线的差分信号，转换成二进制码流通过CAN RX线传给CAN控制器

## 位时序

can总线并没有主从之分,当can总线上的一个设备发送数据时,它以广播的形式在can总线上发送报文给所有的设备。其他设备通过过滤报文的id,处理自己感兴趣的报文。

由于CAN通讯协议并没有时钟信号线,所以要求发送端与接收端的波特率是一致的,而can总线的数据发送效率需要我们去自定义。

要设置设置速率，我先要了解CAN的位时序概念。CAN协议把每个bit分成了四个时间段。我们可以用示波器在CAN TX或者CAN RX上量到下面的10101的波形,把每个bit的波形放大其实会有四段时间:

{% img /stm32can学习笔记/2.png %}

- ss : 同步段（Synchronization Segment）固定为1Tq
- pts : 传播段（Propagation Time Segment）1~8Tq
- pbs1 : 相位缓冲段1（Phase Buffer Segment 1）1~8Tq
- pbs2 : 相位缓冲段2（Phase Buffer Segment 2）2~8Tq

每个段的时长单位是Tq(Time Quantum),这个Tq可以由我们去设置,例如设置为1000ns。

# stm32 位时序配置

上面的是标准的CAN总线位时序,具体每一段的意义我没有深入去了解,但是对于使用来讲并不重要。而stm32里面将pts和pbs1合并了,所以它剩下了三段:

- ss : Synchronization Segment固定为1Tq
- ts1 : Time segment 1, 即 pts + pbs1
- ts2 : Time segment 2, 即pbs2

由于ss固定为1Tq，所以我们在STM32CubeMX里面可以设置的是Tq、ts1和ts2:

{% img /stm32can学习笔记/3.png %}

Tq并不能直接设置,要通过Prescaler设置分频去设置。

例如我们将can的时钟频率设置为36MHz:

{% img /stm32can学习笔记/4.png %}

所以Prescaler设置成36的时候Tq可以这样计算:
1Tq = 36 MHz / 36 = 1 MHz = 1000 ns

这个时候我们就能去计算一个bit的时间了,如上图我们把ts1设置为4,ts2设置为5,再加上ss固定的1Tq:

1Tq + 4Tq + 5Tq = 10 Tq = 10000 ns = 10 us

波特率为 1 s / 10 us = 100k

于是我们可以在树莓派中设置CAN的波特率为100k:

```
sudo ip link set can0 up type can bitrate 100000
```

当然也可以设置每一段的时间:

```
sudo ip link set can0 up type can tq 1000 prop-seg 3 phase-seg1 1 phase-seg2 5 sjw 4
```

prop-seg和phase-seg1加起来等于ts1即可。当然有同学会看到还有另外一个sjw(ReSynchronization Jump Width)的参数,这个时间是用于同步的不影响波特率,范围是1~4Tq,我这里设置成4Tq。

# 标准帧与拓展帧

如此配置之后树莓派就能接收到stm32通过CAN总线发送的数据了,发送的代码如下:

```
HAL_StatusTypeDef Can_TxMessage(uint8_t ide,uint32_t id,uint8_t len,uint8_t *data)
{
    uint32_t   TxMailbox;
    CAN_TxHeaderTypeDef CAN_TxHeader;
    HAL_StatusTypeDef   HAL_RetVal;
    uint16_t i=0;
    if(ide == 0)
    {
        CAN_TxHeader.IDE = CAN_ID_STD;
        CAN_TxHeader.StdId = id;
    }
    else
    {
        CAN_TxHeader.IDE = CAN_ID_EXT;
        CAN_TxHeader.ExtId = id;
    }
    CAN_TxHeader.DLC = len;
    CAN_TxHeader.RTR = CAN_RTR_DATA;
    CAN_TxHeader.TransmitGlobalTime = DISABLE;
    while(HAL_CAN_GetTxMailboxesFreeLevel(&hcan) == 0)
    {
        i++;
        if(i>0xfffe)
        {
            return HAL_ERROR;
        }
    }
    HAL_Delay(500);
    HAL_RetVal = HAL_CAN_AddTxMessage(&hcan,&CAN_TxHeader,data,&TxMailbox);
    if(HAL_RetVal != HAL_OK)
    {
        return HAL_ERROR;
    }
    return HAL_OK;
}

// 发送数据
uint8_t data[8]={170,170,170,170,170,170,170,170};
Can_TxMessage(0,0x222,8,data);
```

Can_TxMessage的第一个参数可以配置CAN报文是标准帧还是拓展帧。它们其实基本只有id的长度不一样而已。这个id就是上面我们提到的用于过滤CAN广播的标识符。
标准帧的id有11位,这11位被命名为STDID。拓展帧在标准帧的基础上增加了18位所以有29位,这个拓展的18位被命名为EXID。



# stm32 CAN id过滤器

stm32 提供了一组过滤器,可以用于过滤CAN报文,只要符合某一个过滤器的规则,该报文即被接收。

过滤器过滤报文有两种模式: 列表模式与掩码模式

## 掩码模式

掩码模式下我们需要配置屏蔽寄存器和标识符寄存器，屏蔽寄存器用于配置需要匹配的CAN id的比特位。屏蔽码寄存器某位为1表示接收到的CAN ID对应的位必须和标识符寄存器对应的位相同。

例如我们将屏蔽码寄存器配置为0xF,意味着我们只关心CAN ID二进制的后4位,此时再将标识符寄存器配置为0xa,意味着所有二进制后四位为1010的CAN ID都能能被接收(例如0xa/0xaa/0xffa等)。

```
0000 0000 ffff  # 掩码寄存器
0000 0000 1010  # 标识符寄存器
--------------
0000 0000 1010  # 0xa
0000 1010 1010  # 0xaa
1111 1111 1010  # 0xffa
```

原理是这个原理,但是是stm32的配置还是需要了解一下的。虽然CAN报文的id长度只有标准帧的11位或者拓展帧的29位，但是stm32中却是用了16位宽或者32位宽的寄存器去保存掩码和标识符。所以会有除了id和mask之外还会有其他的位需要配置。

### 32位宽的掩码模式

我们先来看下面这附图，它说明了32位宽的掩码模式的寄存器每一位的作用:

{% img /stm32can学习笔记/5.jpg %}

id和mask皆由4个字节组成,第一个字节存放了STDID的10\~3bit，第二个字节放了STDID的2\~0bit还有EXID的17\~13bit，第三个字节放了EXID的12\~5bit，第四个字节放了EXID的4\~0bit、IDE（扩展帧标识）、RTR（远程帧标志）和一个预留的0。

我们在代码中通过FilterMaskIdHigh、FilterIdLow、FilterMaskIdHigh、FilterMaskIdLow去设置掩码和标识符:

```
uint32_t ext_id =0xa;
uint32_t mask =0xf;

CAN_FilterTypeDef CAN_FilterType;

// 过滤器的id,STM32F072RBTx提供了14个过滤器所以id可以配置成0~13
CAN_FilterType.FilterBank=0;

// 设置位宽为32位
CAN_FilterType.FilterScale=CAN_FILTERSCALE_32BIT;

// 设置为掩码模式
CAN_FilterType.FilterMode=CAN_FILTERMODE_IDMASK;

// 设置前两个字节的STDID[10:3]、STDID[2:0]、EXID[17:13]
CAN_FilterType.FilterIdHigh=((ext_id<<3) >>16) &0xffff;

// 设置后两个字节的EXID[12:5]、EXID[4:0]、IDE、RTR、预留的一个0
CAN_FilterType.FilterIdLow=(ext_id<<3) | CAN_ID_EXT;

// 设置掩码前两个字节,左移3位再或CAN_ID_EXT是因为最后的三位并不是ID，而是IDE、RTR和预留的0
CAN_FilterType.FilterMaskIdHigh=((mask<<3|CAN_ID_EXT)>>16)&0xffff;

// 设置掩码后两个字节,左移3位再或CAN_ID_EXT是因为最后的三位并不是ID，而是IDE、RTR和预留的0
CAN_FilterType.FilterMaskIdLow=(mask<<3|CAN_ID_EXT)&0xffff;

// 将消息放到FIFO0这个队列里
CAN_FilterType.FilterFIFOAssignment=CAN_RX_FIFO0;

// 激活过滤器
CAN_FilterType.FilterActivation=ENABLE;

// 设置过滤器
if(HAL_CAN_ConfigFilter(&hcan,&CAN_FilterType)!=HAL_OK)
{
  Error_Handler();
}
```

### 16位宽的掩码模式

16位宽的寄存器示意图如下:

{% img /stm32can学习笔记/6.jpg %}

id和mask都是两个字节，但是真正使得标准id起作用的只有第一个字节和第二个自己的前3位。这里各只用了两个字节，也就是说一个过滤器可以设置两组id和mask,FilterMaskIdHigh和FilterMaskIdHigh一组FilterIdLow和FilterMaskIdLow一组:

```
uint32_t std_id1 =0xa;
uint32_t mask1 = 0xf;
uint32_t std_id2 =0xbb;
uint32_t mask2 = 0xff;

CAN_FilterTypeDef CAN_FilterType;

// 过滤器的id,STM32F072RBTx提供了14个过滤器所以id可以配置成0~13
CAN_FilterType.FilterBank=0;

// 设置位宽为16位
CAN_FilterType.FilterScale=CAN_FILTERSCALE_16BIT;

// 设置为掩码模式
CAN_FilterType.FilterMode=CAN_FILTERMODE_IDMASK;

// 设置第一组的id,左移5位是因为最后的5bit是RTR、IDE和EXID[17:15]
CAN_FilterType.FilterIdHigh=(std_id1<<5) | CAN_ID_STD;

// 设置第一组的mask,左移5位是因为最后的5bit是RTR、IDE和EXID[17:15]
CAN_FilterType.FilterMaskIdHigh= ((mask1<<5)|CAN_ID_STD)

// 设置第二组的id,左移5位是因为最后的5bit是RTR、IDE和EXID[17:15]
CAN_FilterType.FilterIdLow=(std_id2<<5)|CAN_ID_STD;

// 设置第二组的mask,左移5位是因为最后的5bit是RTR、IDE和EXID[17:15]
CAN_FilterType.FilterMaskIdLow=(mask2<<5|CAN_ID_STD);

// 将消息放到FIFO0这个队列里
CAN_FilterType.FilterFIFOAssignment=CAN_RX_FIFO0;

// 激活过滤器
CAN_FilterType.FilterActivation=ENABLE;

// 设置过滤器
if(HAL_CAN_ConfigFilter(&hcan,&CAN_FilterType)!=HAL_OK)
{
  Error_Handler();
}
```

## 列表模式

列表模式意味着我们将想要接收的CAN id直接配置到过滤器。

### 32位宽的列表模式

{% img /stm32can学习笔记/7.jpg %}

32位宽的列表模式下,可以设置两个id,FilterMaskIdHigh和FilterMaskIdHigh一个，FilterIdLow和FilterMaskIdLow一个:

```
uint32_t ext_id1 =0xa;
uint32_t ext_id2 =0xbb;

CAN_FilterTypeDef CAN_FilterType;

// 过滤器的id,STM32F072RBTx提供了14个过滤器所以id可以配置成0~13
CAN_FilterType.FilterBank=0;

// 设置位宽为32位
CAN_FilterType.FilterScale=CAN_FILTERSCALE_32BIT;

// 设置为列表模式
CAN_FilterType.FilterMode=CAN_FILTERMODE_IDLIST;

// 设置第一个id的高字节,左移三位是因为最后的三位是IDE、RTR和预留的0
CAN_FilterType.FilterIdHigh=((ext_id1<<3)>>16)&0xffff;

// 设置第一个id的低字节,左移三位是因为最后的三位是IDE、RTR和预留的0
CAN_FilterType.FilterIdLow=((ext_id1<<3)&0xffff)|CAN_ID_EXT;

// 设置第二个id的高字节,左移三位是因为最后的三位是IDE、RTR和预留的0
CAN_FilterType.FilterMaskIdHigh=((ext_id2<<3)>>16)&0xffff;

// 设置第二个id的低字节,左移三位是因为最后的三位是IDE、RTR和预留的0
CAN_FilterType.FilterMaskIdLow=((ext_id2<<3)&0xffff)|CAN_ID_EXT;

// 将消息放到FIFO0这个队列里
CAN_FilterType.FilterFIFOAssignment=CAN_RX_FIFO0;

// 激活过滤器
CAN_FilterType.FilterActivation=ENABLE;

// 设置过滤器
if(HAL_CAN_ConfigFilter(&hcan,&CAN_FilterType)!=HAL_OK)
{
  Error_Handler();
}
```

### 16位宽的列表模式

{% img /stm32can学习笔记/8.png %}

16位宽的列表模式下,可以设置四个id,FilterMaskIdHigh、FilterMaskIdHigh、FilterIdLow和FilterMaskIdLow各一个:

```
uint16_t ext_id1 =0xa;
uint16_t ext_id2 =0xb;
uint16_t ext_id3 =0xc;
uint16_t ext_id4 =0xd;

CAN_FilterTypeDef CAN_FilterType;

// 过滤器的id,STM32F072RBTx提供了14个过滤器所以id可以配置成0~13
CAN_FilterType.FilterBank=0;

// 设置位宽为16位
CAN_FilterType.FilterScale=CAN_FILTERSCALE_16BIT;

// 设置为列表模式
CAN_FilterType.FilterMode=CAN_FILTERMODE_IDLIST;

// 设置第一个id,左移五位是因为最后的五位是RTR、IDE和EXID[17:15]
CAN_FilterType.FilterIdHigh=(ext_id1<<5)|CAN_ID_STD;

// 设置第二个id,左移五位是因为最后的五位是RTR、IDE和EXID[17:15]
CAN_FilterType.FilterIdLow=(ext_id2<<5)|CAN_ID_STD;

// 设置第三个id,左移五位是因为最后的五位是RTR、IDE和EXID[17:15]
CAN_FilterType.FilterMaskIdHigh=(ext_id3<<5)|CAN_ID_STD;

// 设置第四个id,左移五位是因为最后的五位是RTR、IDE和EXID[17:15]
CAN_FilterType.FilterMaskIdLow=(ext_id4<<5)|CAN_ID_STD;

// 将消息放到FIFO0这个队列里
CAN_FilterType.FilterFIFOAssignment=CAN_RX_FIFO0;

// 激活过滤器
CAN_FilterType.FilterActivation=ENABLE;

// 设置过滤器
if(HAL_CAN_ConfigFilter(&hcan,&CAN_FilterType)!=HAL_OK)
{
  Error_Handler();
}
```

# 接收数据

我们可以看到设置过滤器的时候,会配置将过滤出来的数据放到FIFO0这个队里里面:

```
// 将消息放到FIFO0这个队列里
CAN_FilterType.FilterFIFOAssignment=CAN_RX_FIFO0;
```

然后我们还有两步需要操作:

1. 激活这个队列的通知

```
if(HAL_CAN_ActivateNotification(&hcan,CAN_IT_RX_FIFO0_MSG_PENDING)!=HAL_OK)
{
    Error_Handler();
}
```

2. 在STM32CubeMX中使能CAN的接收中断:

{% img /stm32can学习笔记/9.png %}

然后就能重写HAL_CAN_RxFifo0MsgPendingCallback函数去处理接收的数据了:

```
void HAL_CAN_RxFifo0MsgPendingCallback(CAN_HandleTypeDef *hcan)
{
    printf("HAL_CAN_RxFifo0MsgPendingCallback\r\n");
    CAN_RxHeaderTypeDef CAN_RxHeader;
    HAL_StatusTypeDef HAL_Retval;
    uint8_t Rx_Data[8];
    uint8_t Data_Len = 0;
    uint32_t ID = 0;
    uint8_t i;
    HAL_Retval = HAL_CAN_GetRxMessage(hcan,CAN_RX_FIFO0,&CAN_RxHeader,Rx_Data);
    if(HAL_Retval == HAL_OK)
    {
        Data_Len = CAN_RxHeader.DLC;
        if(CAN_RxHeader.IDE)
        {
            ID = CAN_RxHeader.ExtId;
        }
        else
        {
            ID = CAN_RxHeader.StdId;
        }
        printf("id:%x\r\n",ID);
        printf("Data_Len:%d\r\n",Data_Len);
        for(i=0;i<8;i++)
        {
            printf("Rx_Data[%d]=%x\r\n",i,Rx_Data[i]);  
        }
    }
}
```



# NORMAL和LOOPBACK模式

正常模式下设备是收不到自己发送的报文的，我们可以设置LOOPBACK模式实现自发自收,但是注意该模式只用于调试,此时报文其实不会在总线上传播,所以其他设备是收不到发送的报文的:

```
//hcan.Init.Mode = CAN_MODE_NORMAL;
hcan.Init.Mode = CAN_MODE_LOOPBACK;
```

# NORMAL模式下收不到数据

我在调CAN总线的最后遇到了这样一个问题:LOOPBACK模式下能收到自己发的数据,但NORMAL模式下收不到树莓派发送的数据。

通过示波器测量: stm32的CAN RX可以量到树莓派发送的数据波形了，波特率是100k，甚至CAN TX都量到有stm32响应的波形。

而且有个奇怪的现象，stm32在loopback下发数据，CAN TX可以量到完整的数据波形，但是如果改成NORMAL模式，就只能量到一个bit的数据。

于是我怀疑是收发器哪里出问题了,导致发送失败,在接收到数据的时候由于发送响应失败,导致接收的流程也断掉了,没有去到回调函数那里。

最终定位到收发器TJA1042T的STB脚没有拉低,导致收发器处于Standby模式:

{% img /stm32can学习笔记/10.png %}

除了这个原因之外在[网上](http://www.cnblogs.com/whitetiger/p/3811872.html)也有看到这种情况的其他可能原因:

1. 回环下应该与GPIO无关

2. GPIO是否初始化正确，时钟启用

3. 是否复用，AFIO时钟是否启用

4. 回环下是否有CAN_Tx应该有输出

5. 终端电阻是否有

6. CAN收发器电路电压是否正常

7. 波特率是否标准

8. 换块板试一下

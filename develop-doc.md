# 2025-03-07
# 开发文档随笔
***
## 分析任务流程
1.  找到`tlksys_func_t`的成员回调函数调用点
  + init调用点在`tlksys_task_register`,我分析这个是整个任务的初始化
  + start调用点在`tlksys_start`,循环调用所有任务的start,
  + input调用点在`tlksys_task_input`,我分析这个函数主要用于发送具体消息,这个流程得结合调试才能清楚
  + handler调用点在`tlksys_handler`,我分析这个函数当接收到新的tlksys_task_input后执行对应的函数调度

2. 创建自己的任务
  + 注册一个任务列表
  + 发现一个新用法,调用`(void)params`可以去除不使用该参数导致的编译器警告
   ```c
    static int tlkapp_system_init(uint08 procID, uint16 taskID)
    {
      (void)taskID;
      (void)procID;
    }
   ```
  + 核心是消息传递,我应该遵守任务通信规范,这样就可以复用其他任务已经实现的机制
  + 有两种处理方式,涉及到任务间消息同步的我采用任务间通信的方式调用,其他直接调用具体的消息发送函数,这样更好
```c
//中断任务间通信
int tlksys_sendMsgFromIrq(uint16 taskID, uint08 mtype, uint16 msgID, uint08 *pData, uint16 dataLen);
//异步任务间通信
int tlksys_sendInnerMsg(uint16 taskID, uint16 msgID, uint08 *pData, uint16 dataLen);

```
  + 还需要将USB输入和BLE输入挂载到任务里面,这里包括tool_msg
3. 注册任务的初始化,需要注册这个任务涉及到的所有枚举体
  + 注册通用消息类型枚举 `tlksys/tlkptio.h/TLKPRT_COMM_MTYPE_ENUM/TLKPRT_COMM_MTYPE_HISONG_INTERACTION`;
  + 实现这个枚举
  + 注册任务定时器,控制消息的时间处理,如超时,发送顺序
  + 定时器初始化完成,核心是弄清楚使用方式
  + 主要用`tlksys_adapt_insertTimer`这个函数去插入一个新的定时节点,用`tlksys_adapt_removeTimer`去删除用完的timer
  + 从这一点可以构建一个命令定时器,当检测到有命令输入->进入命令队列->处理命令->每次处理命令设置下`gHisongInteractionCurtimer`,插入当前时间的定时器,接着发送该命令到不同任务,如果命令处理完成,移除定时器,如果超时,设置超时处理,

4. 测试.init和bj_usb_udb_input_handler是否成功
  + 测试成功,观察到正常的打印流程
5. 实现.input逻辑
  + .input只在`tlk_task_input`调用,先分析这个函数,这个函数接收任务ID,消息ID,消息本身的字符串,消息的长度;
  + 首先从任务链中匹配对应的任务,然后回调这个挂载的input函数
  + `tlksys_task_input`被`tlksys_handler`调用,观察到只用于系统按键的外部事件调用;
  + `tlksys_task_input`被`tlksys_sendMsg`调用,这个才是任务间通信的有效函数;其中传递了两段buffer,其中headbufer用作消息头
  + 进一步的封装,这是我需要调用的部分;`tlksys_sendInnerMsg`和`tlksys_sendInnerExtMsg`和`tlksys_sendOuterMsg`
  + 我首先处理
6. 测试send_msg是否成功
  + 成功但是没有输出,
  + 内部通信,工具类通信,按键事件需要调用正确的接口
7. AT+VER实现
  + 经过测试,实现了基础的逻辑
8. AT+ADDR实现
  +
9. AT+



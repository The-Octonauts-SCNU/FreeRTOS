# STM32_FreeRTOS学习

### 注解

基于韦东山瑞士军刀开发板进行FreeRTOS进行学习,
本笔记全部工程全部基于韦东山FreeRTOS的例程进行创建

好，我们继续用这家"只有一个厨师的小餐馆"来把 FreeRTOS
的完整任务调度概念一次说透。下面把场景再细化一下，把调度器、优先级、时间片、中断、资源竞争、低功耗等都加进来。

────────────────────

一、餐馆全景图（系统视角）

• 厨师  = CPU

• 菜谱  = 任务的代码

• 菜单  = 任务控制块 TCB（里面写着优先级、栈大小、状态等）

• 老板  = 调度器（Scheduler）

• 收银台 = 就绪队列（Ready List）

• 打瞌睡区 = 阻塞队列（Delay/IPC List）

• 小黑屋 = 挂起队列（Suspended List）

• 外卖窗口 = 中断向量表（ISR）

• 厨房台面 = CPU 寄存器/栈

• 共用菜刀 = 互斥量（Mutex）

• 传菜铃 = 信号量（Semaphore）

────────────────────

二、典型剧情（调度场景）

优先级抢占

 客人 A（高优先级）突然闯进餐馆：

 "我赶时间，先给我炒！"

 老板立即让正在炒菜的客人 B 把锅铲放下（保存现场），把 A
的菜先炒（上下文切换）。

 → 这就是**抢占式调度**。

时间片轮转

 如果两位客人优先级一样，老板规定：每人只能炒 1
分钟（时间片），到点就换下一个。

 → 这就是**时间片轮转**（configUSE_TIME_SLICING）。

阻塞与同步

 客人 C 要做番茄炒蛋，但发现番茄还没送来（等待信号量）。

 老板让他去"打瞌睡区"坐等；番茄一到（ISR
里给出信号量），老板立刻把他拉回"收银台"。

 → 这就是**事件驱动**的阻塞/唤醒。

互斥量与优先级继承

 客人 D 拿着唯一一把菜刀（Mutex）切肉，慢吞吞。

 高优先级的客人 E 也想用菜刀，老板为了避免 E 饿死，**临时把 D
的优先级提升到与 E 一样**（优先级继承），让 D 赶紧用完。

 → 这就是**优先级继承算法**，防止优先级反转。

中断打断了厨师

 外卖窗口按铃（SysTick 或外设中断）：

 厨师立刻放下锅铲（保存现场），跑去窗口取外卖（ISR）；取完再回来继续炒菜。

 如果外卖员说"番茄到了"，厨师会在 ISR 里把客人 C 叫醒（调用
xQueueSendFromISR → xTaskResumeFromISR）。

低功耗（厨师午休）

 所有客人不是在打瞌睡就是关小黑屋，收银台空无一人的时候，老板把厨房灯调暗（进入低功耗
idle hook）。

 → 这就是**Idle Hook + Tickless Idle**。

挂起 vs 删除

 挂起：把客人关进小黑屋贴封条，随时可以撕封条放人。

 删除：老板直接把客人赶出餐馆（vTaskDelete），菜谱撕掉，再也回不来（除非重新招人）。

────────────────────

三、一句话总结

任务调度就是**老板（调度器）按照优先级、时间片、事件、资源等情况，在唯一厨师（CPU）面前不断"换人炒菜"的过程**；而任务状态转换，就是客人被"排队、炒菜、打瞌睡、关小黑屋"的循环。

## 创建FreeRTOS任务(手动)

工程文件:02_first_rtos_app

首先介绍的是HAL库对FreeRTOS和RT_Thread等第三方pack的任务创建具有统一接口函数:
```c
osThreadId_t osThreadNew (osThreadFunc_t func, void argument, const
osThreadAttr_t attr);
```
该函数用于创建任务, 基于Freertos的学习, 目前创建任务的方法有两种,
一种是CUBEMX图形化界面创建, 一种是在freertos.c中手动调用函数进行创建,
本笔记只记载手动创建部分

![](media/media/image1.png)

图中是FreeRTOS的默认任务
```c
void StartDefaultTask(void *argument)
```
如果想要创建自己的任务, 那么user需要在freertos.c中对以下函数进行更改
```c
/ **
* @brief FreeRTOS initialization
* @param None
* @retval None
* /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    
    / * USER CODE END Init * /
    
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL,
    &defaultTask_attributes);
    
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    / * USER CODE END RTOS_THREADS * /
    
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
    
}
```
修改创建自己任务的过程中使用的函数是动态创建
```c
BaseType_t xTaskCreate( TaskFunction_t pxTaskCode,
const char * const pcName, /  / 进程函数地址(名字) / *lint !e971
Unqualified char types are allowed for strings and single characters
only. * /
const configSTACK_DEPTH_TYPE usStackDepth, /  / 进程大概的大小
void * const pvParameters, /  / 进程函数的参数
UBaseType_t uxPriority, /  / 进程函数的优先级
TaskHandle_t * const pxCreatedTask ) /  / 进程函数的句柄, 可以不加
```
下面是自己创建了第二个任务的
```c
/ * USER CODE BEGIN Variables * /
void MyTask(void argument)
{
    while (1)
    {
        Led_Test();
    }
}

/ * USER CODE END Variables * /
/ * Definitions for defaultTask * /
osThreadId_t defaultTaskHandle;
const osThreadAttr_t defaultTask_attributes = {
    .name = "defaultTask",
    .stack_size = 128  4,
    .priority = (osPriority_t) osPriorityNormal,
};

/ * Private function prototypes
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FunctionPrototypes * /

/ * USER CODE END FunctionPrototypes * /

void StartDefaultTask(void argument);

void MX_FREERTOS_Init(void); / * (MISRA C 2004 rule 8.1) * /

/
@brief FreeRTOS initialization
@param None
@retval None
* /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    
    / * USER CODE END Init * /
    
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL, &defaultTask_attributes);
    
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    xTaskCreate(MyTask, "Task2", 128, NULL, osPriorityNormal, NULL); /  / 任务函数地址MyTask,
    任务名字MyTask, 任务栈大小128, 任务优先级一般, 句柄不用
    
    / * USER CODE END RTOS_THREADS * /
    
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
    
}

/ * USER CODE BEGIN Header_StartDefaultTask * /

/
* @brief Function implementing the defaultTask thread.
* @param argument: Not used
* @retval None
* /
/ * USER CODE END Header_StartDefaultTask * /
void StartDefaultTask(void *argument)
{
    / * USER CODE BEGIN StartDefaultTask * /
    / * Infinite loop * /
    LCD_Init();
    LCD_Clear();
    
    for(;;)
    {
        /  / Led_Test();
        LCD_Test();
        /  / MPU6050_Test();
        /  / DS18B20_Test();
        /  / DHT11_Test(); /  / 读取失败
        /  / ActiveBuzzer_Test();
        /  / PassiveBuzzer_Test();
        /  / ColorLED_Test();
        /  / IRReceiver_Test();
        /  / IRSender_Test();
        /  / LightSensor_Test();
        /  / IRObstacle_Test();
        /  / SR04_Test(); /  / 测距异常
        /  / W25Q64_Test();
        /  / RotaryEncoder_Test(); /  / 无法实现增加或者减少
        /  / Motor_Test();
        /  / Key_Test();
        /  / UART_Test();
    }
```
其中LCD_Test()函数当中的OLED_Test函数进行更改, 从而方便现象演示
```c
void OLED_Test(void)
{
    OLED_Init();
    /  / 清屏
    OLED_Clear();
    
    uint8_t count = 0;
    while (count +  + , 1)
    {
        /  / 在(0, 0)打印'A'
        OLED_PutChar(0, 0, 'A');
        /  / 在(1, 0)打印'Y'
        OLED_PutChar(1, 0, 'Y');
        /  / 在第0列第2页打印一个字符串"Hello World!"
        OLED_PrintString(0, 2, "Hello World!");
        
        /  / 使用循环打印环形数字
        if (count > 127)
        {
            count = 0;
            OLED_Clear();
        }
    OLED_PrintSignedVal(0, 4, count);
}
}
```
### 现象

第一次创建FreeRTOS就这样结束了, 现象是数字回环显示, 同时板载LED不断闪烁

## 填坑1

相信结束了第一次手动创建任务之后, 读者会疑惑, 上main函数当中的
```c
xTaskCreate(MyTask, "Task2", 128, NULL, osPriorityNormal, NULL);
/  / 任务函数地址MyTask, 任务名字MyTask, 任务栈大小128, 任务优先级一般,
句柄不用
```
这个任务创建当中的栈是什么, 现在为了更加方便大家理解这个栈,
将讲解ARM架构, 堆栈和一些基础的汇编指令

### ARM的寄存器结构

![](media/media/image2.jpeg)

**汇编指令截图**

![](media/media/image3.jpeg)

### C语言中函数传递参数到ARM CPU的方法

在ARM CPU当中, C语言函数传递参数, 第一个参数放到R0寄存器,
第二个参数放到R1寄存器, 以此类推

### 生成汇编指令的方法

工程文件:03_arm_asm

以创建的第一个FreeRTOS工程为例子, 打开keil的锤子按钮, 对user进行配置,
那个指令是
```c
fromelf -  - text - a - c -  - output = test.dis xxx.axf
```
![](media/media/image4.png)

之后点击Linker, 在Linker的Linker control
string那一栏滚轮直接滑倒最后一行,
复制这个axf的路径直接粘贴在刚刚那个user的地方, 替代那个xxx.axf

![](media/media/image5.png)

![](media/media/image6.png)

在原本的工程加上以下代码从而测试汇编命令:
```c
int add(volatile int a, volatile int b)
{
    volatile int sum = 0;
    sum = a + b;
    return sum;
}

/
* 函数名称： OLED_Test
* 功能描述： OLED测试程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
* /
void OLED_Test(void)
{
    OLED_Init();
    /  / 清屏
    OLED_Clear();
    
    uint8_t count = 0;
    while (1)
    {
        /  / 在(0, 0)打印'A'
        OLED_PutChar(0, 0, 'A');
        /  / 在(1, 0)打印'Y'
        OLED_PutChar(1, 0, 'Y');
        /  / 在第0列第2页打印一个字符串"Hello World!"
        OLED_PrintString(0, 2, "Hello World!");
        
        /  / 使用循环打印环形数字
        add(count, 1);
        if (count > 127)
        {
            count = 0;
            OLED_Clear();
        }
    OLED_PrintSignedVal(0, 4, count);
}
}
```
### 汇编指令格式说明

Flash( 32位 ) 地址 : 机器码 汇编语言
```c
0x08002a28: f000fa82 .... BL add ; 0x8002f30
```
机器码是CPU识别的(也是我们烧录上去的代码),
后面的汇编码是方便人阅读理解的

程序运行后的汇编文件(5200多行):
```c
add()汇编
i.add
add
0x08002f30: b503 .. PUSH {r0,r1,lr}
0x08002f32: b081 .. SUB sp,sp,#4
0x08002f34: e9dd0101 .... LDRD r0,r1,[sp,#4]
0x08002f38: 4408 .D ADD r0,r0,r1
0x08002f3a: 9000 .. STR r0,[sp,#0]
0x08002f3c: bd0e .. POP {r1 - r3,pc}
```
上面是生成的函数add的汇编的指令
```c
OLED_Test()汇编
i.OLED_Test
OLED_Test
0x080029fc: f7ffff02 .... BL OLED_Init ; 0x8002804
0x08002a00: f7fffeeb .... BL OLED_Clear ; 0x80027da
0x08002a04: 2400 .$ MOVS r4,#0
0x08002a06: 2100 .! MOVS r1,#0
0x08002a08: 2241 A" MOVS r2,#0x41
0x08002a0a: 4608 .F MOV r0,r1
0x08002a0c: f7ffff9c .... BL OLED_PutChar ; 0x8002948
0x08002a10: 2259 Y" MOVS r2,#0x59
0x08002a12: 2100 .! MOVS r1,#0
0x08002a14: 2001 . MOVS r0,#1
0x08002a16: f7ffff97 .... BL OLED_PutChar ; 0x8002948
0x08002a1a: a207 .. ADR r2,{pc} + 0x1e ; 0x8002a38
0x08002a1c: 2102 .! MOVS r1,#2
0x08002a1e: 2000 . MOVS r0,#0
0x08002a20: f7ffff79 ..y. BL OLED_PrintString ; 0x8002916
0x08002a24: 2101 .! MOVS r1,#1
0x08002a26: 4620 F MOV r0,r4
0x08002a28: f000fa82 .... BL add ; 0x8002f30
0x08002a2c: 4622 "F MOV r2,r4
0x08002a2e: 2104 .! MOVS r1,#4
0x08002a30: 2000 . MOVS r0,#0
0x08002a32: f7ffff31 ..1. BL OLED_PrintSignedVal ; 0x8002898
0x08002a36: e7e6 .. B 0x8002a06 ; OLED_Test + 10
```
上面是本来OLED_Test()函数的汇编指令

下main讲解一下这个OLED_Test函数调用add函数对应的指令的执行过程

这里的r4寄存器是存放着count这个数值的, 需要知道
```c
0x08002a28: f000fa82 .... BL add ; 0x8002f30
```
从这, OLED_Test开始调用add函数, 读取对应Flash地址当中的机器码,
之后在CPU内部当中执行这些机器码去使用add函数
```c
add()汇编
i.add
add
0x08002f30: b503 .. PUSH {r0,r1,lr}
0x08002f32: b081 .. SUB sp,sp,#4
0x08002f34: e9dd0101 .... LDRD r0,r1,[sp,#4]
0x08002f38: 4408 .D ADD r0,r0,r1
0x08002f3a: 9000 .. STR r0,[sp,#0]
0x08002f3c: bd0e .. POP {r1 - r3,pc}
```
这个时候, 按照压栈操作, 执行第一句(机器码为503这一句),
将当前指针寄存器SP所在位置移动12字节(从而放入3个寄存器)
```c
0x08002f30: b503 .. PUSH {r0,r1,lr}
```
此时1r位于栈顶(最后入栈), r0位于最高位(地址), 其中r0与r1是作为入参的,
r0对应的是传参的count(这个时候还没有赋值),
r1对应的是传参的1(这个时候还没有赋值),
这个lr则是返回的地址(这个时候还没有赋值), 对应的是下面这句汇编指令当中的
```c
0x08002a2c: 4622 "F MOV r2,r4
```
0x08002a2c这个flash地址, 之后执行下一条add的指令
```c
0x08002f34: e9dd0101 .... LDRD r0,r1,[sp,#4]
```
SP寄存器指向的地址 + 4(字节)赋值给r0<不更新SP>, SP寄存器指向地址再 +
4 + 4(字节)赋值给r1, 这样子, 就实现了传参赋值<现在(r0 == count) && (r1
== 1)>, 之后执行下面的汇编指令:
```c
0x08002f38: 4408 .D ADD r0,r0,r1
```
对r0与r1执行加法计算, r0 = r0 + r1, 之后执行下main这一汇编指令:
```c
0x08002f3a: 9000 .. STR r0,[sp,#0]
```
将新算出来的数值存在SP寄存器当前指向的指针位置, 接下来执行下面的代码
```c
0x08002f3c: bd0e .. POP {r1 - r3,pc}
```
这里直接将栈当中的数值(这个是之前的函数调用子函数时存留的数据)POP到CPU寄存器r1,
r2, r3当中,
仅仅是为了后面将LR寄存器的保存的返回地址直接POP到PC寄存器之后回到本来执行的OLED_Test()函数当中,
可以理解成方便return

(返回地址几乎是最先入栈的), 之后跳回OLED_Test函数, 执行下面的指令
```c
0x08002a2c: 4622 "F MOV r2,r4
```
r4就是count, r2则是结果

## 堆栈的概念

相信不少小伙伴早就在学习C语言过程当中了解了堆栈了,
下面还麻烦读者能耐心看完这一部分

### 堆

工程文件:04_heap_stack

简单来说, 堆就像是一块可以自由分配的内存,
堆可以存放很多动态分布的数据和变量等等
```c
char heap_buf[1024];* /  / 模拟堆*
int pos = 0; * /  / 模拟堆指针*

void *my_malloc(int size)
{
    int old_pos = pos;
    pos +  = size;
    return &heap_buf[old_pos];
}

void my_free(void *buf)
{
    /  / 模拟释放内存
    / * err * /
}

int main()
{
    char ch = 65; * /  / 'A' in ASCII*
    int i = 0;
    char *buf = my_malloc(100);
    unsigned char uch = 200;
    for (i = 0; i < 26; i +  + )
    {
        buf[i] = 'A' + i;
    }
return 0;
}
```
通过调试,
就可以清晰的查看到当前模拟堆的分配.但是在这种情况下想实现Free函数非常麻烦,
需要通过非常多的中间变量.下main来介绍一下真正的free函数的机制和malloc函数机制

#### 真正的free函数的机制和malloc函数机制

在malloc动态分配内存时,
其会分别生成可以存储user数据和对于动态分配出的区域信息(大小, 状态,
前后向指针)的区域, 下面是一个例子:
```c
struct mem_header {
    size_t size; /  / 内存块大小
    int allocated; /  / 内存块状态（1表示已分配，0表示空闲）
    struct mem_header *next; /  / 前向指针
    struct mem_header *prev; /  / 后向指针
} __attribute__((packed)); /  / 确保结构体不会被编译器填充
```
当你调用free函数之后, free函数会先向前检索这一区域, 直到得知内存块大小,
值得注意这里动态分配出来的区域位于该信息区的低位,
信息区是高位(你可以联想一下栈底), 我们称这一块信息区为头部区域,
头部位置存在于固定出来的一块地方, 一般来说都会在分配出来的区域前

这里涉及到链表等数据结构的内容,
包括链表的创建,插入与删除.如果想更加深入了解, 可以去查一下堆管理

### 栈

工程文件:04_heap_stack

栈是一个存放函数和局部变量的地方, 下面演示一下这个栈的调用
```c
int g_cnt = 0;

int c_func(volatile int a)
{
    a +  = 2;
    return a;
}

int b_func(volatile int a)
{
    a +  = 3;
    return a;
}

void a_func(volatile int a)
{
    g_cnt = b_func(a);
    g_cnt = c_func(g_cnt);
}

int main()
{
    char ch = 65; * /  / 'A' in ASCII*
    int i = 99;
    
    a_func(i);
    return 0;
}
```
生成汇编文件(可以参考下面的生成汇编指令的方法)

这个是main函数的汇编指令
```c
0x0800018c: b510 .. PUSH {r4,lr}
0x0800018e: 2441 A$ MOVS r4,#0x41
0x08000190: 2263 c" MOVS r2,#0x63
0x08000192: 4610 .F MOV r0,r2
0x08000194: f7ffffde .... BL a_func ; 0x8000154
0x08000198: 2000 . MOVS r0,#0
0x0800019a: bd10 .. POP {r4,pc}
```
其中的
```c
0x08000194: f7ffffde .... BL a_func ; 0x8000154
```
会跳转到a_func()函数, 下面是a_func()函数的汇编指令
```c
a_func
0x08000154: b501 .. PUSH {r0,lr}
0x08000156: 9800 .. LDR r0,[sp,#0]
0x08000158: f000f80c .... BL b_func ; 0x8000174
0x0800015c: 4904 .I LDR r1,[pc,#16] ; [0x8000170] = 0x20000000
0x0800015e: 6008 .` STR r0,[r1,#0]
0x08000160: 4608 .F MOV r0,r1
0x08000162: 6800 .h LDR r0,[r0,#0]
0x08000164: f000f80c .... BL c_func ; 0x8000180
0x08000168: 4901 .I LDR r1,[pc,#4] ; [0x8000170] = 0x20000000
0x0800016a: 6008 .` STR r0,[r1,#0]
0x0800016c: bd08 .. POP {r3,pc}
```
其中的两条指令
```c
0x08000158: f000f80c .... BL b_func ; 0x8000174

0x08000164: f000f80c .... BL c_func ; 0x8000180
```
分别可以跳转到b_func()函数和c_func()函数

从上面可以看出来, 这个函数调用关系是:main() -> a_func() -> b_func &
c_func()

而C语言的函数调用的本质则是使用BL汇编指令

再具体看b_func()和c_func()的汇编指令
```c
b_func
0x08000174: b501 .. PUSH {r0,lr}
0x08000176: 9800 .. LDR r0,[sp,#0]
0x08000178: 1cc0 .. ADDS r0,r0,#3
0x0800017a: 9000 .. STR r0,[sp,#0]
0x0800017c: 9800 .. LDR r0,[sp,#0]
0x0800017e: bd08 .. POP {r3,pc}

i.c_func
c_func
0x08000180: b501 .. PUSH {r0,lr}
0x08000182: 9800 .. LDR r0,[sp,#0]
0x08000184: 1c80 .. ADDS r0,r0,#2
0x08000186: 9000 .. STR r0,[sp,#0]
0x08000188: 9800 .. LDR r0,[sp,#0]
0x0800018a: bd08 .. POP {r3,pc}
```
根据上面的调用关系和下面的汇编指令:
```c
main - >a_func
0x08000194: f7ffffde .... BL a_func ; 0x8000154
0x08000198: 2000 . MOVS r0,#0

a_func - >c_func
0x08000164: f000f80c .... BL c_func ; 0x8000180
0x08000168: 4901 .I LDR r1,[pc,#4] ; [0x8000170] = 0x20000000

b_func - >c_func
0x08000158: f000f80c .... BL b_func ; 0x8000174
0x0800015c: 4904 .I LDR r1,[pc,#16] ; [0x8000170] = 0x20000000
```
根据之前的经验, 我们知道LR是保存返回地址的, PC是直接跳转到对应地址的:

main -> a_func LR:0x08000198; PC: 0x8000154 <a_func()的地址>

a_func -> b_func LR:0x0800015c; PC:0x8000174 <b_func()的地址>

a_func -> c_func LR:0x08000168; PC:0x8000180 <c_func()的地址>

这里我们就得提出几个问题了:

1.为什么在调用多个函数当中,
已经有的PC和LR寄存器的数值被更替却能有条不紊的返回?

因为具有压栈这一特性, 所以可以先进后出, 我们看具体的汇编指令:

main()->a_func() <LR直接入栈, a_func中的lr直接被修改了>
```c
a_func
0x08000154: b501 .. PUSH {r0,lr}
```
a_func()->b_func() <相似操作, 再次入栈>
```c
b_func
0x08000174: b501 .. PUSH {r0,lr}
```
a_func()->c_func() <相似操作再次入栈>
```c
c_func
0x08000180: b501 .. PUSH {r0,lr}
```
每一次调用函数入栈, 都会直接修改lr,
当然如果编译器优化等级过高可能会导致直接丢失lr,
但是一旦函数嵌套调用就肯定会这样保存lr

以上可以总结一点, 几乎每一个.c文件的头部都会存在这个lr入栈的操作,
存储在CPU的Flash区域

同时我们再看这个函数在栈中的活动(这里是参照韦东山老师讲解的,
按道理来说这些栈并非如此, 但是可以讲解)
```c
0x0800018c: b510 .. PUSH {r4,lr}
```
这里mian函数入栈一个返回地址加一个R4, 和下图有所区别,
但是意思差不多(arm地址是从高到低入栈的)

之后调用a_func()
```c
a_func
0x08000154: b501 .. PUSH {r0,lr}
```
到a_func函数入栈, 同时带了返回地址和R0这个参数

下面的b_func和c_func函数则是并列关系, 一个函数执行完之后会销毁

![](media/media/image7.png)

![](media/media/image8.png)

2.局部变量是如何在栈当中进行分配的呢?

下main以韦东山的视频例程讲解:

![](media/media/image9.jpeg)

按照汇编指令

![](media/media/image10.jpeg)

可以很清晰地知道入栈LR R6等寄存器,

但第二条MOVS则表明了, 这个r5(0x41是'A'即为字符65),
这个r5变量对应的是ch变量, 却存储在了寄存器而不是flash当中,

*[显然并非所有变量都会在运行间存储到ram.加了volatile或者容量不够时才会放到flash当中]{.underline}*

再看下main那一条, 则是先放在寄存器<movs>, 之后又放在flash<str>,
这个0x64则是16进制的99了, 同时也可以看见现在它存储在指针SP当前的位置,
这个volatile的变量就直接放在Ram当中了
```c
MOVS r0, #0x64
```
目光看到这, 则是一个0x64(字符100), 这代表了移动到寄存器,
之后就直接调用函数了
```c
MOV r4, r0
```
这个就直接说明了r4与r0是相等的, 那么r4则是*buf

显然, 没有编译器优化的就直接放在了flash,
由编译器优化的为了更快的适应就直接放在了寄存器方便快速响应

![](media/media/image11.jpeg)

这个图中则是创建了大量变量, 当变量太多时,
就会导致比编译器把一些东西放入栈当中, 少则不会(除非你加了不优化的修饰)

3.为何每一个RTOS任务都有自己的栈呢?

每一个RTOS任务拥有自己的局部变量和函数嵌套调用关系,
还需要保护自己的进程(就是保护现场)

下main举一个例子讲解一下:
```c
void Task_A(void)
{
    int cnt = 0;
    b_func();
}
```
```c
void Task_B(void)
{
    int cnt = 120;
    b_func();
}
```
之后是b_func(void)的汇编代码块
```c
b_func
0x08000174: b501 .. PUSH {r0,lr}
0x08000176: 9800 .. LDR r0,[sp,#0]
0x08000178: 1cc0 .. ADDS r0,r0,#3
0x0800017a: 9000 .. STR r0,[sp,#0]
0x0800017c: 9800 .. LDR r0,[sp,#0]
0x0800017e: bd08 .. POP {r3,pc}
```
二者在RTOS当中都会调用这个b_func(), 那是怎么实现切换的呢,
这就得说RTOS的基准时钟了, 根据这个基准时钟的定时来实现一会执行任务A,
一会执行任务B, 如下方框图所示:
```c
Task_A() - > Task_B() - > Task_A()

/  / Time - > -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - ||||| -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - ||||| -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
执行周期 执行周期 执行周期
```
使用这种伪并行策略, 为了实现更高精度, 执行周期将很短, 需要频繁打断任务,
记录任务函数当时的ARM-CPU寄存器情况, 因此在这种情况下,
实际上的栈是这样的:

首先执行Task_A, 之后入栈对应变量, 调用b_func()
```c
/  / 栈
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - 高位
Task_A
  ------------------------
b_func
  ------------------------

  ------------------------

  ------------------------

  ------------------------

-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - 低位
```
下main是b_func的汇编指令:
```c
b_func
0x08000174: b501 .. PUSH {r0,lr}
0x08000176: 9800 .. LDR r0,[sp,#0]
0x08000178: 1cc0 .. ADDS r0,r0,#3
0x0800017a: 9000 .. STR r0,[sp,#0]
0x0800017c: 9800 .. LDR r0,[sp,#0]
0x0800017e: bd08 .. POP {r3,pc}
```
假设, 我们执行了三条指令, 刚刚进行了加法计算, 也就是执行了以下指令:
```c
b_func
0x08000174: b501 .. PUSH {r0,lr}
0x08000176: 9800 .. LDR r0,[sp,#0]
0x08000178: 1cc0 .. ADDS r0,r0,#3
```
这个时候, RTOS内核直接切换任务了, 那么就执行Task_B,
实际上在ARM_CPU寄存器的Task_A的信息, 包括对函数的调用和局部变量的信息,
都要保存, 从而在结束Task_B时能做到顺利调用Task_A并继续进行任务A

也就是说, Task_A恢复执行, Task_B的进程要保护现场, 放到Ram当中,
之后Task_B恢复B的现场并继续执行

在RAM当中的体现则是:
```c
/  / 栈
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - 高位
Task_A
  ------------------------
b_func
  ------------------------
Task_A information (结构体等存储信息的变量)
  ------------------------
Task_B
  ------------------------

  ------------------------

-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - 低位
```
## FREERTOS源码以及工程架构概述

### 工程架构

大部分的RTOS(只要是CUBEMX生成的都差不多结构)都是RTOS的第三方包文件夹 +
STM32的HAL库代码 + CrotexM3内核的文件

其中, 在第三方文件夹(Middlewares)当中, 会包含RTOS的代码等,

![](media/media/image12.png)

如图所示, 这里的核心文件是与进程架构相关的task.c等裸露在外的.c与.h文件

而portable文件夹则是存放了一些可以移植更改的文件,
在其内部是一个有关不同IDE的特定文件夹和内存堆管理文件夹, 如下图所示

![](media/media/image13.png)

![](media/media/image14.png)

这里GCC的文件夹是因为我用makefile文件进行工程配置(对应GCC编译器),
当你用KEIL时就是RVDS(对应ARM编译器),
剩下的MemMang文件夹刚也说了用于堆管理

![](media/media/image15.png)

可以看到其中文件, 都是Heap(堆)开头的.c文件,
把堆管理文件放这里大部分时候都是出于user可以提供自己的堆管理函数

再看回这里

![](media/media/image12.png)

其中的include不用多说也可以知道这个是RTOS的头文件路径

FreeRTOS需要用到3个头文件路径:
```c
.CoreIncFreeRTOSConfig.h /  / 图形化配置界面的具体config
.MiddlewaresThird_PartyFreeRTOSSourceinclude
/  / FreeRTSO本身的头文件
.MiddlewaresThird_PartyFreeRTOSSourceportable
/  / 移植用到的头文件
```
在图形化配置界面当中:

![](media/media/image16.png)

就对应下面的代码:
```c
/ * USER CODE BEGIN Header * /
/ * *
FreeRTOS Kernel V10.3.1*
Copyright (C) 2017 Amazon.com, Inc. or its affiliates. All Rights Reserved.*
*
Permission is hereby granted, free of charge, to any person obtaining a copy of*
this software and associated documentation files (the "Software"), to deal in*
the Software without restriction, including without limitation the rights to*
use, copy, modify, merge, publish, distribute, sublicense, and / or sell copies of*
the Software, and to permit persons to whom the Software is furnished to do so,*
subject to the following conditions:*
*
The above copyright notice and this permission notice shall be included in all*
copies or substantial portions of the Software.*
*
THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR*
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS*
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR*
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER*
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN*
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.*
*
http: /  / www.FreeRTOS.org*
http: /  / aws.amazon.com / freertos*
*
1 tab =  = 4 spaces!*
* * /
/ * USER CODE END Header * /
#ifndef FREERTOS_CONFIG_H
#define FREERTOS_CONFIG_H
/ * -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - *
Application specific definitions.*
*
These definitions should be adjusted for your particular hardware and*
application requirements.*
*
These parameters and more are described within the 'configuration' section of the*
FreeRTOS API documentation available on the FreeRTOS.org web site.*
*
See http: /  / www.freertos.org / a00110.html*
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN Includes * /
/ * Section where include file can be added * /
/ * USER CODE END Includes * /
/ * Ensure definitions are only used by the compiler, and not by the assembler. * /
#if defined(__ICCARM__) || defined(__CC_ARM) || defined(__GNUC__)
#include <stdint.h>
extern uint32_t SystemCoreClock;
#endif
#ifndef CMSIS_device_header
#define CMSIS_device_header "stm32f1xx.h"
#endif / * CMSIS_device_header * /

#define configUSE_PREEMPTION 1
#define configSUPPORT_STATIC_ALLOCATION 1
#define configSUPPORT_DYNAMIC_ALLOCATION 1
#define configUSE_IDLE_HOOK 0
#define configUSE_TICK_HOOK 0
#define configCPU_CLOCK_HZ ( SystemCoreClock )
#define configTICK_RATE_HZ ((TickType_t)1000)
#define configMAX_PRIORITIES ( 56 )
#define configMINIMAL_STACK_SIZE ((uint16_t)128)
#define configTOTAL_HEAP_SIZE ((size_t)3072)
#define configMAX_TASK_NAME_LEN ( 16 )
#define configUSE_TRACE_FACILITY 1
#define configUSE_16_BIT_TICKS 0
#define configUSE_MUTEXES 1
#define configQUEUE_REGISTRY_SIZE 8
#define configUSE_RECURSIVE_MUTEXES 1
#define configUSE_COUNTING_SEMAPHORES 1
#define configUSE_PORT_OPTIMISED_TASK_SELECTION 0
/ * Co - routine definitions. * /
#define configUSE_CO_ROUTINES 0
#define configMAX_CO_ROUTINE_PRIORITIES ( 2 )
/ * Software timer definitions. * /
#define configUSE_TIMERS 1
#define configTIMER_TASK_PRIORITY ( 2 )
#define configTIMER_QUEUE_LENGTH 10
#define configTIMER_TASK_STACK_DEPTH 256
/ * Set the following definitions to 1 to include the API function, or zero*
*to exclude the API function. * /
#define INCLUDE_vTaskPrioritySet 1
#define INCLUDE_uxTaskPriorityGet 1
#define INCLUDE_vTaskDelete 1
#define INCLUDE_vTaskCleanUpResources 0
#define INCLUDE_vTaskSuspend 1
#define INCLUDE_vTaskDelayUntil 1
#define INCLUDE_vTaskDelay 1
#define INCLUDE_xTaskGetSchedulerState 1
#define INCLUDE_xTimerPendFunctionCall 1
#define INCLUDE_xQueueGetMutexHolder 1
#define INCLUDE_uxTaskGetStackHighWaterMark 1
#define INCLUDE_xTaskGetCurrentTaskHandle 1
#define INCLUDE_eTaskGetState 1
/ * *
The CMSIS - RTOS V2 FreeRTOS wrapper is dependent on the heap implementation used*
by the application thus the correct define need to be enabled below*
* * /
#define USE_FreeRTOS_HEAP_4
/ * Cortex - M specific definitions. * /
#ifdef __NVIC_PRIO_BITS
/ * __BVIC_PRIO_BITS will be specified when CMSIS is being used. * /
#define configPRIO_BITS __NVIC_PRIO_BITS
#else
#define configPRIO_BITS 4
#endif
/ * The lowest interrupt priority that can be used in a call to a "set priority"*
*function. * /
#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY 15
/ * The highest interrupt priority that can be used by any interrupt service*
*routine that makes calls to interrupt safe FreeRTOS API functions. DO NOT CALL*
*INTERRUPT SAFE FREERTOS API FUNCTIONS FROM ANY INTERRUPT THAT HAS A HIGHER*
*PRIORITY THAN THIS! (higher priorities are lower numeric values. * /
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY 5
/ * Interrupt priorities used by the kernel port layer itself. These are generic*
*to all Cortex - M ports, and do not rely on any particular library functions. * /
#define configKERNEL_INTERRUPT_PRIORITY ( configLIBRARY_LOWEST_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )
/ * !!!! configMAX_SYSCALL_INTERRUPT_PRIORITY must not be set to zero !!!!*
*See http: /  / www.FreeRTOS.org / RTOS - Cortex - M3 - M4.html. * /
#define configMAX_SYSCALL_INTERRUPT_PRIORITY ( configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )
/ * Normal assert() semantics without relying on the provision of an assert.h*
*header file. * /
/ * USER CODE BEGIN 1 * /
#define configASSERT( x ) if ((x) =  = 0) {taskDISABLE_INTERRUPTS(); for( ;; );}
/ * USER CODE END 1 * /
/ * Definitions that map the FreeRTOS port interrupt handlers to their CMSIS*
*standard names. * /
#define vPortSVCHandler SVC_Handler
#define xPortPendSVHandler PendSV_Handler
/ * IMPORTANT: After 10.3.1 update, Systick_Handler comes from NVIC (if SYS timebase = systick), otherwise from cmsis_os2.c * /
#define USE_CUSTOM_SYSTICK_HANDLER_IMPLEMENTATION 0
/ * USER CODE BEGIN Defines * /
/ * Section where parameter definitions can be added (for instance, to override default ones in FreeRTOS.h) * /
/ * USER CODE END Defines * /
#endif / * FREERTOS_CONFIG_H * /
```
如果我们需要阅读RTOS的代码, 那么就要从下面的地方开始
```c
/ * USER CODE BEGIN Header * /
/ *

File Name : freertos.c*
Description : Code for freertos applications*

@attention*
*
Copyright (c) 2025 STMicroelectronics.*
All rights reserved.*
*
This software is licensed under terms that can be found in the LICENSE file*
in the root directory of this software component.*
If no LICENSE file comes with this software, it is provided AS - IS.*
*

* * /
/ * USER CODE END Header * /
/ * Includes -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
#include "FreeRTOS.h"
#include "task.h"
#include "main.h"
#include "cmsis_os.h"
/ * Private includes -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN Includes * /
#include "driver_led.h"
#include "driver_lcd.h"
#include "driver_mpu6050.h"
#include "driver_timer.h"
#include "driver_ds18b20.h"
#include "driver_dht11.h"
#include "driver_active_buzzer.h"
#include "driver_passive_buzzer.h"
#include "driver_color_led.h"
#include "driver_ir_receiver.h"
#include "driver_ir_sender.h"
#include "driver_light_sensor.h"
#include "driver_ir_obstacle.h"
#include "driver_ultrasonic_sr04.h"
#include "driver_spiflash_w25q64.h"
#include "driver_rotary_encoder.h"
#include "driver_motor.h"
#include "driver_key.h"
#include "driver_uart.h"
/ * USER CODE END Includes * /
/ * Private typedef -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN PTD * /
/ * USER CODE END PTD * /
/ * Private define -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN PD * /
/ * USER CODE END PD * /
/ * Private macro -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN PM * /
/ * USER CODE END PM * /
/ * Private variables -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN Variables * /
/ * USER CODE END Variables * /
/ * Definitions for defaultTask * /
osThreadId_t defaultTaskHandle;
const osThreadAttr_t defaultTask_attributes = {
    .name = "defaultTask",
    .stack_size = 128  4,
    .priority = (osPriority_t) osPriorityNormal,
};
/ * Private function prototypes -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FunctionPrototypes * /
/ * USER CODE END FunctionPrototypes * /
void StartDefaultTask(void argument);

void MX_FREERTOS_Init(void); / * (MISRA C 2004 rule 8.1) * /
/ *
@brief FreeRTOS initialization*
@param None*
@retval None*
* * /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    / * USER CODE END Init * /
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL, &defaultTask_attributes);
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    / * USER CODE END RTOS_THREADS * /
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
}
/ * USER CODE BEGIN Header_StartDefaultTask * /
/ *
@brief Function implementing the defaultTask thread.*
@param argument: Not used*
@retval None*
* * /
/ * USER CODE END Header_StartDefaultTask * /
void StartDefaultTask(void argument*)
{
    / * USER CODE BEGIN StartDefaultTask * /
    / * Infinite loop * /
    for(;;)
    {
        /  / Led_Test();
        /  / LCD_Test();
        /  / MPU6050_Test();
        /  / DS18B20_Test();
        /  / DHT11_Test(); /  / 读取失败
        /  / ActiveBuzzer_Test();
        /  / PassiveBuzzer_Test();
        /  / ColorLED_Test();
        /  / IRReceiver_Test();
        /  / IRSender_Test();
        /  / LightSensor_Test();
        /  / IRObstacle_Test();
        /  / SR04_Test(); /  / 测距异常
        /  / W25Q64_Test();
        /  / RotaryEncoder_Test(); /  / 无法实现增加或者减少
        /  / Motor_Test();
        /  / Key_Test();
        /  / UART_Test();
    }
/ * USER CODE END StartDefaultTask * /
}
/ * Private application code -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN Application * /
/ * USER CODE END Application * /
```
可以看到其中的函数
```c
void MX_FREERTOS_Init(void)
/ *
@brief FreeRTOS initialization*
@param None*
@retval None*
* * /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    / * USER CODE END Init * /
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL,
    &defaultTask_attributes);
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    / * USER CODE END RTOS_THREADS * /
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
}
```
用于初始化任务

在main函数当中:
```c
int main(void)
{
    / * USER CODE BEGIN 1 * /
    / * USER CODE END 1 * /
    / * MCU
    Configuration -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
    / * Reset of all peripherals, Initializes the Flash interface and the Systick. * /
    HAL_Init();
    / * USER CODE BEGIN Init * /
    / * USER CODE END Init * /
    / * Configure the system clock * /
    SystemClock_Config();
    / * USER CODE BEGIN SysInit * /
    / * USER CODE END SysInit * /
    / * Initialize all configured peripherals * /
    MX_GPIO_Init();
    MX_ADC1_Init();
    MX_I2C1_Init();
    MX_SPI1_Init();
    MX_TIM1_Init();
    MX_TIM2_Init();
    MX_TIM3_Init();
    MX_USART1_UART_Init();
    MX_USB_PCD_Init();
    / * USER CODE BEGIN 2 * /
    / * USER CODE END 2 * /
    / * Init scheduler * /
    osKernelInitialize(); / * Call init function for freertos objects (in cmsis_os2.c) * /
    MX_FREERTOS_Init(); /  / 任务初始化
    / * Start scheduler * /
    osKernelStart(); /  / 调度器启动
    / * We should never get here as control is now taken by the scheduler * /
    / * Infinite loop * /
    / * USER CODE BEGIN WHILE * /
    while (1)
    {
        / * USER CODE END WHILE * /
        / * USER CODE BEGIN 3 * /
    }
/ * USER CODE END 3 * /
}
```
先是初始化任务, 之后启用调度器

在阅读RTOS的代码过程中, 我们还会遇到TickType_t与BaseType_t,
这两个都是FreeRTOS系统当中的封装

![](media/media/image17.png)

TickType_t 就是 FreeRTOS 的"心跳节拍计数器"类型，用来衡量
 **多少个系统时钟节拍（tick）已经过去**。

所有跟"时间"有关的 API 都会用到它，例如：

延时 / 阻塞：vTaskDelay(xTicksToDelay)、xQueueReceive(...,
xTicksToWait)

获得当前 tick 数：xTaskGetTickCount()

软件定时器周期 / 超时值

任务时间片统计、运行时统计

在 FreeRTOSConfig.h 里可以通过
```c
演示
Plain Text
#define configUSE_16_BIT_TICKS 0|1
```
把它配置成 16 位或 32 位无符号整数（uint16_t 或 uint32_t）。

因此，**TickType_t 只是一个"tick 数量"的单位，用来在 FreeRTOS
内核里统一表示时间长度**。

一般来说, 32位处理器的建议配置成uint32_t

当然这个TickType_t本身是在portmacro.h这个头文件中定义的(这个文件在与之前编译器有关的那个文件夹当中)

BaseType_t 是 FreeRTOS 的"**原生整型**"------相当于 FreeRTOS 的
int。

为什么需要它

在 8/16/32 位 MCU 上，**最快的整数宽度**可能不同（8 位 AVR 是
int8_t，32 位 ARM 是 int32_t）。

FreeRTOS 希望既省资源又能保证 **原子操作** 与 **返回值范围**
够用，于是定义了一个"**对当前 CPU
来说最自然、最高效的带符号整数类型**"，这就是 BaseType_t。

典型用途

任务创建、队列、信号量、事件组等 API 的
 **返回值**（pdPASS、pdFAIL、长度、句柄索引等）。

任务通知值、队列长度计数、调度器开关计数等 **内部变量**。

位图、状态标志（EventBits_t、UBaseType_t 是其无符号版本）。

在哪儿定义

在 portmacro.h 里，由移植层决定；例如：

ARM Cortex-M：typedef long BaseType_t;

AVR8：typedef signed char BaseType_t;

一句话总结

BaseType_t 就是 FreeRTOS 的"默认最快带符号整数"，用来放 API
返回值、长度、标志位等，确保代码在不同 MCU
上都高效。通常用于简单的返回值类型还有逻辑值.

### 变量命名

下面是一些变量的前缀

![](media/media/image18.jpeg)

之后是一些函数前缀的意义

![](media/media/image19.jpeg)

除了前缀, 这些函数名字还会加上在哪里定义, 比如vTask...()就表示返回void,
在task.c中定义, 如图所示

下面是宏定义:(大差不差)

![](media/media/image20.jpeg)

之后是通用的宏定义格式:

![](media/media/image21.jpeg)

## 内存管理

内存管理实际上就是堆的配置和堆的管理

可以打开之前的CubeMX工程的配置界面, 查看FREERTOS的config界面:

![](media/media/image22.png)

从图中可以看出, 这个Memory
Allocation可以静态分配(Static)也可以动态分配(Dynamic),
堆的大小时3072字节, 使用heap_4策略管理

heap_4 是 FreeRTOS 官方提供的 **5 种内存管理方案（heap_1 ~
heap_5）中的一种**，属于 **动态内存分配策略**。

说明当前工程把 configTOTAL_HEAP_SIZE 设为 3072 字节，并让 FreeRTOS 使用
heap_4 来管理这 3 KB 的"**总堆**"。

heap_4 的特点

**允许释放内存**：不像 heap_1 只能 malloc、不能 free，heap_4 支持
pvPortMalloc() / vPortFree() 配对使用。

**碎片合并**：内部带一个简单的 **首次适应 + 相邻空闲块合并**
算法，能显著减少长时间运行后的碎片。

**非线程安全**（由 FreeRTOS 调度器通过挂起调度器/关中断来保证）。

**确定性较高**：时间复杂度 O(n)（n 为空闲块数），比标准库 malloc
更可预测，适合嵌入式实时场景。

**一块连续大数组**：实际占用的是你定义的 static uint8_t
ucHeap[configTOTAL_HEAP_SIZE]（位于 RAM），heap_4
就在这块数组里做分配/释放。

一句话

heap_4 就是 FreeRTOS
自带的"**可释放、带碎片合并的动态内存管理器**"，把用户指定的 3072
字节 RAM 当堆，用来给任务、队列、信号量等对象分配空间。

这里如果感觉还不是很通俗易懂, 请看下面的部分和链接:

heap_4要讲明白就得先说heap_1.c, heap_2.c和heap_3.c

![](media/media/image23.png)

heap_3

heap_1只会分配堆, 但是不会回收, 相当于一次性用品, 堆一旦用完就废了

heap_2会分配, 会回收, 但是回收之后不会合并回收的多个堆,
本来你用了两个堆(一个100, 一个50),
但是回收之后还是只能按照分配最多是这两个的最大值, 你撑死只能分配100,
不能分配120, 因为这两个区域回收之后并没合并

heap_4就解决了heap_2的问题, 能分配, 能回收,
避免碎片所以heap_4是最常用的策略

当你不止一块ram了, 你外接一个ram就要用heap_5,
使用heap_5也要指定这个heap_5各个堆的起始地址

文档链接地址:https://rtos.100ask.net/zh/FreeRTOS/DShanMCU-F103/chapter8.html#_8-2-freertos%E7%9A%845%E4%B8%AD%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E6%96%B9%E6%B3%95

对于静态或是动态分配我们无法干预, 否则就无法编译这个heap_4.c文件,
在heap_4.c文件当中, 按照上面的介绍它创建了一个数组,
此时在heap_4.c当中亦有记载
```c
/ * Allocate the memory for the heap. * /
#if( configAPPLICATION_ALLOCATED_HEAP =  = 1 )
/ * The application writer has already defined the array used for the
RTOS*
*heap - probably so it can be placed in a special segment or address. * /
extern uint8_t ucHeap[ configTOTAL_HEAP_SIZE ];
#else
static uint8_t ucHeap[ configTOTAL_HEAP_SIZE ];
#endif / * configAPPLICATION_ALLOCATED_HEAP * /
```
这个数组的大小对应的就是堆大小
```c
#define configTOTAL_HEAP_SIZE ((size_t)3072)
```
至于如何使用这个堆, 那直接看这个堆的调用就好了

![](media/media/image24.png)

如图所示, 这个堆的初始化函数是一个私有函数, 它需要手动调用吗, 不需要,
此事在
```c
演示
C +  +
heap_4.c
/ * Gets set to the top bit of an size_t type. When this bit in the xBlockSize*
*member of an BlockLink_t structure is set then the block belongs to the*
*application. When the bit is free the block is still part of the free heap*
*space. * /
static size_t xBlockAllocatedBit = 0;
/ * -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
void pvPortMalloc( size_t xWantedSize )
{
    BlockLink_t pxBlock, pxPreviousBlock, pxNewBlockLink;
    void pvReturn = NULL;
    
    vTaskSuspendAll();
    {
        / * If this is the first call to malloc then the heap will require*
        *initialisation to setup the list of free blocks. * /
        if( pxEnd =  = NULL )
        {
            prvHeapInit();
        }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
/ * Check the requested block size is not so large that the top bit is*
*set. The top bit of the block size member of the BlockLink_t structure*
*is used to determine who owns the block - the application or the*
*kernel, so it must be free. * /
if( ( xWantedSize & xBlockAllocatedBit ) =  = 0 )
{
    / * The wanted size is increased so it can contain a BlockLink_t*
    *structure in addition to the requested amount of bytes. * /
    if( xWantedSize > 0 )
    {
        xWantedSize +  = xHeapStructSize;
        / * Ensure that blocks are always aligned to the required number*
        *of bytes. * /
        if( ( xWantedSize & portBYTE_ALIGNMENT_MASK ) != 0x00 )
        {
            / * Byte alignment required. * /
            xWantedSize +  = ( portBYTE_ALIGNMENT - ( xWantedSize & portBYTE_ALIGNMENT_MASK ) );
            configASSERT( ( xWantedSize & portBYTE_ALIGNMENT_MASK ) =  = 0 );
        }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
}
else
{
    mtCOVERAGE_TEST_MARKER();
}

if( ( xWantedSize > 0 ) && ( xWantedSize < = xFreeBytesRemaining ) )
{
    / * Traverse the list from the start (lowest address) block until*
    *one of adequate size is found. * /
    pxPreviousBlock = &xStart;
    pxBlock = xStart.pxNextFreeBlock;
    while( ( pxBlock - >xBlockSize < xWantedSize ) && ( pxBlock - >pxNextFreeBlock != NULL ) )
    {
        pxPreviousBlock = pxBlock;
        pxBlock = pxBlock - >pxNextFreeBlock;
    }
/ * If the end marker was reached then a block of adequate size*
*was not found. * /
if( pxBlock != pxEnd )
{
    / * Return the memory space pointed to - jumping over the*
    *BlockLink_t structure at its start. * /
    pvReturn = ( void  ) ( ( ( uint8_t  ) pxPreviousBlock - >pxNextFreeBlock ) + xHeapStructSize );
    / * This block is being returned for use so must be taken out*
    *of the list of free blocks. * /
    pxPreviousBlock - >pxNextFreeBlock = pxBlock - >pxNextFreeBlock;
    / * If the block is larger than required it can be split into*
    *two. * /
    if( ( pxBlock - >xBlockSize - xWantedSize ) > heapMINIMUM_BLOCK_SIZE )
    {
        / * This block is to be split into two. Create a new*
        *block following the number of bytes requested. The void*
        *cast is used to prevent byte alignment warnings from the*
        *compiler. * /
        pxNewBlockLink = ( void  ) ( ( ( uint8_t  ) pxBlock ) + xWantedSize );
        configASSERT( ( ( ( size_t ) pxNewBlockLink ) & portBYTE_ALIGNMENT_MASK ) =  = 0 );
        / * Calculate the sizes of two blocks split from the*
        *single block. * /
        pxNewBlockLink - >xBlockSize = pxBlock - >xBlockSize - xWantedSize;
        pxBlock - >xBlockSize = xWantedSize;
        / * Insert the new block into the list of free blocks. * /
        prvInsertBlockIntoFreeList( pxNewBlockLink );
    }
else
{
    mtCOVERAGE_TEST_MARKER();
}

xFreeBytesRemaining -  = pxBlock - >xBlockSize;

if( xFreeBytesRemaining < xMinimumEverFreeBytesRemaining )
{
    xMinimumEverFreeBytesRemaining = xFreeBytesRemaining;
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
/ * The block is being returned - it is allocated and owned*
*by the application and has no "next" block. * /
pxBlock - >xBlockSize | = xBlockAllocatedBit;
pxBlock - >pxNextFreeBlock = NULL;
xNumberOfSuccessfulAllocations +  + ;
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
else
{
    mtCOVERAGE_TEST_MARKER();
}

traceMALLOC( pvReturn, xWantedSize* );
}
( void ) xTaskResumeAll();

#if( configUSE_MALLOC_FAILED_HOOK =  = 1 )
{
    if( pvReturn =  = NULL )
    {
        extern void vApplicationMallocFailedHook( void );
        vApplicationMallocFailedHook();
    }
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
#endif

configASSERT( ( ( ( size_t ) pvReturn ) & ( size_t ) portBYTE_ALIGNMENT_MASK ) =  = 0 );
return pvReturn;
}
/ * -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
```
亦有记载, 当你第一次调用这个[void *pvPortMalloc( size_t *xWantedSize*
)]{.underline}这个函数就会自动初始化堆

也就是对于heap_4策略下, 直接调用[void *pvPortMalloc( size_t
*xWantedSize* )]{.underline}分配即可, 用完之后调用[void vPortFree( void
 **pv* )即可]{.underline}

下main还有一些函数供用户用于优化程序, 感知程序的:
```c
演示
C +  +
/ * -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/  / 用于确定还有多少空闲内存, heap_3用不了这个函数
size_t xPortGetFreeHeapSize( void )
{
    return xFreeBytesRemaining;
}
/ * -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/  / 用于确定程序运行当中的最小空闲内存, heap_4和heap_5才能用
size_t xPortGetMinimumEverFreeHeapSize( void )
{
    return xMinimumEverFreeBytesRemaining;
}
/ * -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
```
还有一个pvPortMalloc函数:
```c
演示
C +  +
/ * -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
void pvPortMalloc( size_t xWantedSize )
{
    BlockLink_t pxBlock, pxPreviousBlock, pxNewBlockLink;
    void pvReturn = NULL;
    
    vTaskSuspendAll();
    {
        / * If this is the first call to malloc then the heap will require*
        *initialisation to setup the list of free blocks. * /
        if( pxEnd =  = NULL )
        {
            prvHeapInit();
        }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
/ * Check the requested block size is not so large that the top bit is*
*set. The top bit of the block size member of the BlockLink_t structure*
*is used to determine who owns the block - the application or the*
*kernel, so it must be free. * /
if( ( xWantedSize & xBlockAllocatedBit ) =  = 0 )
{
    / * The wanted size is increased so it can contain a BlockLink_t*
    *structure in addition to the requested amount of bytes. * /
    if( xWantedSize > 0 )
    {
        xWantedSize +  = xHeapStructSize;
        / * Ensure that blocks are always aligned to the required number*
        *of bytes. * /
        if( ( xWantedSize & portBYTE_ALIGNMENT_MASK ) != 0x00 )
        {
            / * Byte alignment required. * /
            xWantedSize +  = ( portBYTE_ALIGNMENT - ( xWantedSize & portBYTE_ALIGNMENT_MASK ) );
            configASSERT( ( xWantedSize & portBYTE_ALIGNMENT_MASK ) =  = 0 );
        }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
}
else
{
    mtCOVERAGE_TEST_MARKER();
}

if( ( xWantedSize > 0 ) && ( xWantedSize < = xFreeBytesRemaining ) )
{
    / * Traverse the list from the start (lowest address) block until*
    *one of adequate size is found. * /
    pxPreviousBlock = &xStart;
    pxBlock = xStart.pxNextFreeBlock;
    while( ( pxBlock - >xBlockSize < xWantedSize ) && ( pxBlock - >pxNextFreeBlock != NULL ) )
    {
        pxPreviousBlock = pxBlock;
        pxBlock = pxBlock - >pxNextFreeBlock;
    }
/ * If the end marker was reached then a block of adequate size*
*was not found. * /
if( pxBlock != pxEnd )
{
    / * Return the memory space pointed to - jumping over the*
    *BlockLink_t structure at its start. * /
    pvReturn = ( void  ) ( ( ( uint8_t  ) pxPreviousBlock - >pxNextFreeBlock ) + xHeapStructSize );
    / * This block is being returned for use so must be taken out*
    *of the list of free blocks. * /
    pxPreviousBlock - >pxNextFreeBlock = pxBlock - >pxNextFreeBlock;
    / * If the block is larger than required it can be split into*
    *two. * /
    if( ( pxBlock - >xBlockSize - xWantedSize ) > heapMINIMUM_BLOCK_SIZE )
    {
        / * This block is to be split into two. Create a new*
        *block following the number of bytes requested. The void*
        *cast is used to prevent byte alignment warnings from the*
        *compiler. * /
        pxNewBlockLink = ( void  ) ( ( ( uint8_t  ) pxBlock ) + xWantedSize );
        configASSERT( ( ( ( size_t ) pxNewBlockLink ) & portBYTE_ALIGNMENT_MASK ) =  = 0 );
        / * Calculate the sizes of two blocks split from the*
        *single block. * /
        pxNewBlockLink - >xBlockSize = pxBlock - >xBlockSize - xWantedSize;
        pxBlock - >xBlockSize = xWantedSize;
        / * Insert the new block into the list of free blocks. * /
        prvInsertBlockIntoFreeList( pxNewBlockLink );
    }
else
{
    mtCOVERAGE_TEST_MARKER();
}

xFreeBytesRemaining -  = pxBlock - >xBlockSize;

if( xFreeBytesRemaining < xMinimumEverFreeBytesRemaining )
{
    xMinimumEverFreeBytesRemaining = xFreeBytesRemaining;
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
/ * The block is being returned - it is allocated and owned*
*by the application and has no "next" block. * /
pxBlock - >xBlockSize | = xBlockAllocatedBit;
pxBlock - >pxNextFreeBlock = NULL;
xNumberOfSuccessfulAllocations +  + ;
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
else
{
    mtCOVERAGE_TEST_MARKER();
}

traceMALLOC( pvReturn, xWantedSize* );
}
( void ) xTaskResumeAll();

#if( configUSE_MALLOC_FAILED_HOOK =  = 1 )
{
    if( pvReturn =  = NULL )
    {
        extern void vApplicationMallocFailedHook( void );
        vApplicationMallocFailedHook();
    }
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
#endif

configASSERT( ( ( ( size_t ) pvReturn ) & ( size_t ) portBYTE_ALIGNMENT_MASK ) =  = 0 );
return pvReturn;
}
/ * -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
```
在pvPortMalloc函数内部：
```c
void * pvPortMalloc( size_t xWantedSize )vPortDefineHeapRegions
{......#if ( configUSE_MALLOC_FAILED_HOOK =  = 1 ){if( pvReturn =  = NULL
        ){extern void vApplicationMallocFailedHook( void
            );vApplicationMallocFailedHook();}}#endifreturn pvReturn;
}
```
所以，如果想使用这个钩子函数：

在FreeRTOSConfig.h中，把configUSE_MALLOC_FAILED_HOOK定义为1

提供vApplicationMallocFailedHook函数

pvPortMalloc失败时，才会调用此函数

这可以方便你分配失败的时候调试

## 任务管理

创建多个任务工程, 每个任务不仅仅要有自己的函数,
还需要保存任务的栈和任务的重要性

在多任务当中, 由于需要多个栈, 所以使用链表等进行管理,
通过链表得知所有进程, 链表当中存放的是任务结构体,称呼为TCB(任务控制块)

至于这个栈和TCB怎么办, 有两种方法, 提前创建且静态, 动态创建,
所以就分为后面接着讲的两个函数

而优先级决定任务的重要性, 举个例子:

优先级(Priority)

我工作生活兼顾：喂饭、回信息优先级一样，轮流做

我忙里偷闲：还有空闲任务，休息一下

厨房着火了，什么都别说了，先灭火：优先级更高

**如何创建任务(静态/动态)所要用到的函数一个是之前用过的动态创建函数:**
```c
BaseType_t xTaskCreate( TaskFunction_t pxTaskCode,
const char  const pcName, / *lint !e971 Unqualified char types are
allowed for strings and single characters only. * /
const configSTACK_DEPTH_TYPE usStackDepth,
void  const pvParameters,
UBaseType_t uxPriority,
TaskHandle_t  const pxCreatedTask* )
```
动态创建时依赖于动态分配的内存, 参数只是决定了栈的大小

另外一个则是动态创建函数:
```c
TaskHandle_t xTaskCreateStatic( TaskFunction_t pxTaskCode,
const char  const pcName, / *lint !e971 Unqualified char types are
allowed for strings and single characters only. * /
const uint32_t ulStackDepth,
void  const pvParameters,
UBaseType_t uxPriority,
StackType_t  const puxStackBuffer,
StaticTask_t  const pxTaskBuffer )
```
这两个函数的参数相似,
但是动态创建函数需要先创建好一个数组和一个TCB结构体供其使用

### 创建多任务

工程文件:05_create_task
```c
#include "driver_passive_buzzer.h"
#include "driver_timer.h"
/ *

@file : Music.h*
@brief : Header for Music.c file.*
This file provides code for the configuration*
of the Music instances*
@author : Lesterbor*
@time : 2021 - 10 - 12*

@attention*
*
*

* * /
#define Bass 0
#define Alto 1
#define Teble 2

#define One_Beat 1
#define One_TWO_Beat 2
#define One_FOUR_Beat 4
/ *

@file : Music.c*
@brief : Music program body*
@author : Lesterbor*
@time : 2022 - 01 - 15*
*

@attention*
*
*
*

* * /
/ * USER CODE END Includes * /
/ * Private typedef -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN PT * /
uint16_t Tone_Index[8][3] = {
    {0 ,0 ,0 },
    {262,523,1046},
    {294,587,1175},
    {330,659,1318},
    {349,698,1397},
    {392,784,1568},
    {440,880,1760},
    {494,988,1976}
};
/  /
/  /  /  / 中速代表乐曲速度术语里的Moderato，或称中板，每分钟在88到104拍。
/  /  /  / 中速 每分钟100拍 一拍600ms
/  / 两只老虎简谱，没有细调
/  / uint16_t Music_Two_Tigers[][3] = {
    /  / {0,0,570},
    /  / {1,Alto,One_Beat},
    /  / {2,Alto,One_Beat},
    /  / {3,Alto,One_Beat},
    /  / {1,Alto,One_Beat},
    /  /
    /  / {0,Alto,24},
    /  /
    /  / {1,Alto,One_Beat},
    /  / {2,Alto,One_Beat},
    /  / {3,Alto,One_Beat},
    /  / {1,Alto,One_Beat},
    /  /
    /  /  /  / {0,Alto,3},
    /  /
    /  / {3,Alto,One_Beat},
    /  / {4,Alto,One_Beat},
    /  / {5,Alto,One_Beat},
    /  / {0,Alto,One_Beat},
    /  /
    /  /
    /  / {3,Alto,One_Beat},
    /  / {4,Alto,One_Beat},
    /  / {5,Alto,One_Beat},
    /  / {0,Alto,One_Beat},
    /  /
    /  /
    /  / {5,Alto,One_TWO_Beat},
    /  / {6,Alto,One_TWO_Beat},
    /  / {5,Alto,One_TWO_Beat},
    /  / {4,Alto,One_TWO_Beat},
    /  / {3,Alto,One_Beat},
    /  / {1,Alto,One_Beat},
    /  /
    /  /  /  / {0,Alto,3},
    /  /
    /  / {5,Alto,One_TWO_Beat},
    /  / {6,Alto,One_TWO_Beat},
    /  / {5,Alto,One_TWO_Beat},
    /  / {4,Alto,One_TWO_Beat},
    /  / {3,Alto,One_Beat},
    /  / {1,Alto,One_Beat},
    /  /
    /  / {0,Alto,24},
    /  /
    /  / {1,Alto,One_Beat},
    /  / {5,Bass,One_Beat},
    /  / {1,Alto,One_Beat},
    /  / {0,Alto,One_Beat},
    /  /
    /  /
    /  / {1,Alto,One_Beat},
    /  / {5,Bass,One_Beat},
    /  / {1,Alto,One_Beat},
    /  / {0,Alto,One_Beat},
    /  /
    /  / };
/  / 中速 每分钟65拍 一拍920ms
uint16_t Music_Lone_Brave[][3] = {
    /  / 曲信息
    {0,0,920},
    /  / #define Bass 0
    /  / #define Alto 1
    /  / #define Teble 2
    /  / #define One_Beat 1
    /  / #define One_TWO_Beat 2
    /  / #define One_FOUR_Beat 4
    /  / 第一小节
    {2,Alto,One_TWO_Beat} ,{7,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,
    {2,Alto,One_TWO_Beat} ,{7,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,
    /  / 第二小节
    {2,Alto,One_TWO_Beat} ,{7,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,
    {2,Alto,One_TWO_Beat} ,{7,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,
    /  / 第三小节
    {2,Alto,One_TWO_Beat} ,{7,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,
    {2,Alto,One_TWO_Beat} ,{7,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,
    /  / 第四小节
    {2,Alto,One_TWO_Beat} ,{7,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,
    {2,Alto,One_TWO_Beat} ,{7,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,
    /  / 第五小节
    {2,Alto,One_TWO_Beat} ,{7,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,
    {2,Alto,One_TWO_Beat} ,{7,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,
    /  / 第六小节
    {3,Alto,One_Beat} ,{3,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_FOUR_Beat} ,
    {1,Alto,One_FOUR_Beat} ,{2,Alto,One_FOUR_Beat},{1,Alto,One_FOUR_Beat} ,
    /  / 第七小节
    {3,Alto,One_Beat} ,{3,Alto,One_Beat} ,{0,Alto,One_TWO_Beat} ,{1,Alto,One_FOUR_Beat} ,
    {2,Alto,One_FOUR_Beat} ,{1,Alto,One_FOUR_Beat},{2,Alto,One_FOUR_Beat} ,{3,Alto,One_FOUR_Beat} ,
    /  / 第八小节
    {6,Bass,One_TWO_Beat} ,{1,Alto,One_FOUR_Beat},{6,Bass,One_TWO_Beat} ,{1,Alto,One_FOUR_Beat} ,
    {6,Bass,One_TWO_Beat} ,{1,Alto,One_FOUR_Beat},{2,Alto,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,
    /  / 第九小节
    {7,Bass,One_TWO_Beat} ,{7,Bass,One_FOUR_Beat},{0,Alto,One_TWO_Beat} ,{0,Alto,One_FOUR_Beat} ,
    /  / 第十小节
    {3,Alto,One_Beat} ,{3,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_FOUR_Beat} ,
    {1,Alto,One_FOUR_Beat} ,{2,Alto,One_FOUR_Beat},{1,Alto,One_FOUR_Beat} ,
    /  / 第十一小节
    {3,Alto,One_Beat} ,{3,Alto,One_Beat} ,{0,Alto,One_TWO_Beat} ,{1,Alto,One_FOUR_Beat} ,
    {2,Alto,One_FOUR_Beat} ,{1,Alto,One_FOUR_Beat},{2,Alto,One_FOUR_Beat} ,{3,Alto,One_FOUR_Beat} ,
    /  / 第十二小节
    {6,Bass,One_TWO_Beat} ,{1,Alto,One_FOUR_Beat},{6,Bass,One_TWO_Beat} ,{1,Alto,One_FOUR_Beat} ,
    {6,Bass,One_TWO_Beat} ,{1,Alto,One_FOUR_Beat},{3,Alto,One_TWO_Beat} ,{2,Alto,One_TWO_Beat} ,
    /  / 第十三小节
    {7,Bass,One_TWO_Beat} ,{7,Bass,One_FOUR_Beat},{0,Alto,One_TWO_Beat} ,{0,Alto,One_FOUR_Beat} ,
    /  / 第十四小节
    {6,Bass,One_FOUR_Beat} ,{1,Alto,One_FOUR_Beat},{6,Alto,One_TWO_Beat} ,{6,Alto,One_FOUR_Beat} ,
    {0,Alto,20 * / 小衔接 / *} ,{6,Alto,One_FOUR_Beat},{6,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,
    {6,Alto,One_TWO_Beat} ,{6,Alto,One_FOUR_Beat},{5,Alto,One_FOUR_Beat} ,{6,Alto,One_FOUR_Beat} ,
    {5,Alto,One_FOUR_Beat} ,{6,Alto,One_FOUR_Beat},{5,Alto,One_FOUR_Beat} ,
    /  / 第十五小节
    {3,Alto,One_FOUR_Beat} ,{3,Alto,One_TWO_Beat} ,{3,Alto,One_Beat} ,{0,Alto,20 * / 小衔接 / *} ,
    {0,Alto,One_Beat} ,{0,Alto,One_TWO_Beat} ,{6,Bass,One_FOUR_Beat} ,{1,Alto,One_FOUR_Beat} ,
    /  / 第十六小节
    {6,Alto,One_TWO_Beat} ,{6,Alto,One_FOUR_Beat},{0,Alto,20 * / 小衔接 / *} ,{6,Alto,One_FOUR_Beat} ,
    {5,Alto,One_FOUR_Beat} ,{6,Alto,One_FOUR_Beat},{5,Alto,One_FOUR_Beat} ,{7,Alto,One_TWO_Beat} ,
    {7,Alto,One_FOUR_Beat} ,{0,Alto,20 * / 小衔接 / *},{7,Alto,One_FOUR_Beat} ,{6,Alto,One_FOUR_Beat} ,
    {7,Alto,One_TWO_Beat} ,
    /  / 第十七小节
    {7,Alto,One_FOUR_Beat} ,{6,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat} ,{3,Alto,One_TWO_Beat} ,
    {3,Alto,One_TWO_Beat} ,{0,Alto,One_FOUR_Beat},{3,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,
    {3,Alto,One_FOUR_Beat} ,
    /  / 第十八小节
    {2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat},{2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat} ,
    {2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat},{5,Alto,One_FOUR_Beat} ,{3,Alto,One_FOUR_Beat} ,
    {5,Alto,One_FOUR_Beat} ,{3,Alto,One_FOUR_Beat},
    /  / 第十九小节
    {2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat},{2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat} ,
    {2,Alto,One_Beat} ,{0,Alto,One_TWO_Beat} ,{1,Alto,One_FOUR_Beat} ,{2,Alto,One_FOUR_Beat} ,
    /  / 第二十小节
    {3,Alto,One_TWO_Beat} ,{6,Bass,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{3,Alto,One_TWO_Beat} ,
    {2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat},{2,Alto,One_FOUR_Beat} ,{1,Alto,One_FOUR_Beat} ,
    {1,Alto,One_TWO_Beat} ,
    /  / 第二十一小节
    {6,Bass,One_Beat} ,{6,Bass,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_TWO_Beat} ,
    {6,Alto,One_FOUR_Beat} ,{7,Alto,One_FOUR_Beat},
    /  / 第二十二小节
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    {7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,
    {1,Teble,One_FOUR_Beat},{0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    /  / 第二十三小节
    {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_TWO_Beat} ,{3,Teble,One_TWO_Beat} ,{6,Alto,One_FOUR_Beat} ,
    {7,Alto,One_FOUR_Beat} ,
    /  / 第二十四小节
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    {7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,
    {1,Teble,One_FOUR_Beat},{0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    /  / 第二十五小节
    {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_TWO_Beat} ,{3,Teble,One_TWO_Beat} ,{5,Teble,One_TWO_Beat} ,
    /  / 第二十六小节
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},{3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{5,Teble,One_FOUR_Beat},{6,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {5,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{5,Teble,One_TWO_Beat} ,
    /  / 第二十七小节
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},{3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{5,Teble,One_FOUR_Beat},{6,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {5,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{5,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},
    {5,Teble,One_TWO_Beat} ,
    /  / 第二十八小节
    /  / {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,
    /  / {1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },
    /  / {2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{1,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},
    {2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,184 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    /  / 第二十九小节
    {1,Teble,One_TWO_Beat} ,{0,Alto,One_FOUR_Beat} ,{0,Alto,One_TWO_Beat} ,{0,Alto,One_TWO_Beat} ,
    {5,Teble,One_FOUR_Beat},{0,Alto,184 * / 小衔接 / *},{5,Teble,One_FOUR_Beat},
    /  / 第三十小节
    /  / {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,
    /  / {1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },
    /  / {2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{1,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},
    {2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,184 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    /  / 第三十一小节
    {1,Teble,One_TWO_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_Beat} ,
    /  /
    /  /  /  / 第三十二小节
    /  / {0,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_Beat} ,
    /  /
    /  /  /  / 第三十三小节
    /  / {0,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_Beat} ,
    /  / 第三十四小节
    {0,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_TWO_Beat} ,
    {6,Teble,One_FOUR_Beat},{5,Alto,One_FOUR_Beat} ,
    /  / 第三十五小节
    {6,Alto,One_TWO_Beat} ,{5,Alto,One_FOUR_Beat} ,{6,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,
    {6,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,{6,Alto,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},
    {6,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,{6,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,
    {6,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,
    /  / 第三十六小节
    /  / {3,Alto,One_FOUR_Beat} ,{3,Alto,One_TWO_Beat} ,{3,Alto,One_Beat} ,{0,Alto,One_Beat} ,
    /  / {0,Alto,One_TWO_Beat} ,{6,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,
    {3,Alto,One_TWO_Beat} ,{3,Alto,One_Beat} ,{0,Alto,One_Beat} ,
    {0,Alto,One_TWO_Beat} ,{6,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,
    /  / 第三十七小节
    {6,Alto,One_TWO_Beat} ,{5,Alto,One_FOUR_Beat} ,{6,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,
    {6,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,{7,Alto,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *} ,
    {7,Alto,One_FOUR_Beat} ,{0,Alto,184 * / 小衔接 / *} ,{7,Alto,One_FOUR_Beat} ,{6,Alto,One_FOUR_Beat} ,
    {7,Alto,One_FOUR_Beat} ,{6,Alto,One_FOUR_Beat} ,
    /  / 第三十八小节
    /  / {3,Alto,One_FOUR_Beat} ,{3,Alto,One_TWO_Beat} ,{3,Alto,One_Beat} ,{0,Alto,One_Beat} ,
    /  / {0,Alto,One_FOUR_Beat},{3,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,{3,Alto,One_FOUR_Beat} ,
    {3,Alto,One_TWO_Beat} ,{3,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_FOUR_Beat},
    {3,Alto,One_FOUR_Beat} ,{5,Alto,One_FOUR_Beat} ,{3,Alto,One_FOUR_Beat} ,
    /  / 第三十九小节
    {2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat},{2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat} ,
    {2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat},{5,Alto,One_FOUR_Beat} ,{3,Alto,One_FOUR_Beat} ,
    {5,Alto,One_FOUR_Beat} ,{3,Alto,One_FOUR_Beat},
    /  / 第四十小节
    {2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat},{2,Alto,One_TWO_Beat} ,{3,Alto,One_FOUR_Beat} ,
    {2,Alto,One_Beat} ,{0,Alto,One_TWO_Beat} ,{1,Alto,One_FOUR_Beat} ,{2,Alto,One_FOUR_Beat} ,
    /  / 第四十一小节
    {3,Alto,One_TWO_Beat} ,{6,Alto,One_TWO_Beat} ,{1,Teble,One_TWO_Beat} ,{3,Teble,One_TWO_Beat} ,
    {2,Teble,One_TWO_Beat} ,{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat} ,
    {1,Teble,One_TWO_Beat} ,
    /  / 第四十二小节
    {6,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_TWO_Beat} ,
    {6,Alto,One_FOUR_Beat} ,{7,Alto,One_FOUR_Beat},
    /  / 开始第一遍循环
    /  / 第二十二小节
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    {7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,
    {1,Teble,One_FOUR_Beat},{0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    /  / 第二十三小节
    {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_TWO_Beat} ,{3,Teble,One_TWO_Beat} ,{6,Alto,One_FOUR_Beat} ,
    {7,Alto,One_FOUR_Beat} ,
    /  / 第二十四小节
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    {7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,
    {1,Teble,One_FOUR_Beat},{0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    /  / 第二十五小节
    {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_TWO_Beat} ,{3,Teble,One_TWO_Beat} ,{5,Teble,One_TWO_Beat} ,
    /  / 第二十六小节
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},{3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{5,Teble,One_FOUR_Beat},{6,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {5,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{5,Teble,One_TWO_Beat} ,
    /  / 第二十七小节
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},{3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{5,Teble,One_FOUR_Beat},{6,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {5,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{5,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},
    {5,Teble,One_TWO_Beat} ,
    /  / 第二十八小节
    /  / {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,
    /  / {1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },
    /  / {2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{1,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},
    {2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,184 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    /  / 第二十九小节
    {1,Teble,One_TWO_Beat} ,{0,Alto,One_FOUR_Beat} ,{0,Alto,One_TWO_Beat} ,{0,Alto,One_TWO_Beat} ,
    {5,Teble,One_FOUR_Beat},{0,Alto,184 * / 小衔接 / *},{5,Teble,One_FOUR_Beat},
    /  / 第三十小节
    /  / {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,
    /  / {1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },
    /  / {2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{1,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},
    {2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,184 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    /  / 第一遍循环结束
    /  / 第四十三小节
    {6,Alto,One_TWO_Beat} ,{6,Alto,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{3,Alto,One_TWO_Beat} ,
    {7,Alto,One_Beat},{0,Alto,184 * / 小衔接 / *},{7,Alto,One_TWO_Beat},{0,Alto,184 * / 小衔接 / *},
    {7,Alto,One_FOUR_Beat},{0,Alto,184 * / 小衔接 / *},{7,Alto,One_FOUR_Beat},
    /  / 第四十四小节
    /  / {6,Alto,One_FOUR_Beat} ,{6,Alto,One_TWO_Beat} ,{6,Alto,One_Beat} ,{0,Alto,One_Beat} ,
    /  / {0,Alto,One_Beat},{0,Alto,One_Beat},
    {6,Alto,One_TWO_Beat} ,{6,Alto,One_TWO_Beat} ,{6,Alto,One_TWO_Beat} ,{0,Alto,One_Beat} ,
    {0,Alto,One_Beat},{0,Alto,One_Beat},
    /  / 第四十五小节
    {6,Alto,One_TWO_Beat} ,{6,Alto,One_TWO_Beat} ,{1,Alto,One_TWO_Beat} ,{3,Alto,One_TWO_Beat} ,
    {7,Alto,One_Beat},{0,Alto,184 * / 小衔接 / *},{7,Alto,One_TWO_Beat},{0,Alto,184 * / 小衔接 / *},
    {7,Alto,One_FOUR_Beat},{0,Alto,184 * / 小衔接 / *},{7,Alto,One_FOUR_Beat},
    /  / 第四十六小节
    {7,Alto,One_FOUR_Beat},{6,Alto,One_TWO_Beat},{6,Alto,One_Beat} ,{0,Alto,One_Beat},
    {0,Alto,One_TWO_Beat},{6,Alto,One_FOUR_Beat} ,{7,Alto,One_FOUR_Beat},
    /  / 第二次循环
    /  / 第二十二小节
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    {7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,
    {1,Teble,One_FOUR_Beat},{0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    /  / 第二十三小节
    {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_TWO_Beat} ,{3,Teble,One_TWO_Beat} ,{6,Alto,One_FOUR_Beat} ,
    {7,Alto,One_FOUR_Beat} ,
    /  / 第二十四小节
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    {7,Alto,One_FOUR_Beat} ,{1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{7,Alto,One_FOUR_Beat} ,
    {1,Teble,One_FOUR_Beat},{0,Alto,306 * / 小衔接 / *},{1,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},
    {1,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    /  / 第二十五小节
    {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{0,Alto,306 * / 小衔接 / *},{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_TWO_Beat} ,{3,Teble,One_TWO_Beat} ,{5,Teble,One_TWO_Beat} ,
    /  / 第二十六小节
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},{3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{5,Teble,One_FOUR_Beat},{6,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {5,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{5,Teble,One_TWO_Beat} ,
    /  / 第二十七小节
    {3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},{3,Teble,One_TWO_Beat} ,{5,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{5,Teble,One_FOUR_Beat},{6,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {5,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{5,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},
    {5,Teble,One_TWO_Beat} ,
    /  / 第二十八小节
    /  / {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,
    /  / {1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },
    /  / {2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{1,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},
    {2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,184 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    /  / 第二十九小节
    {1,Teble,One_TWO_Beat} ,{0,Alto,One_FOUR_Beat} ,{0,Alto,One_TWO_Beat} ,{0,Alto,One_TWO_Beat} ,
    {5,Teble,One_FOUR_Beat},{0,Alto,184 * / 小衔接 / *},{5,Teble,One_FOUR_Beat},
    /  / 第三十小节
    /  / {3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,
    /  / {1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{3,Teble,One_FOUR_Beat},{2,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },
    /  / {2,Teble,One_TWO_Beat} ,{0,Alto,184 / 小衔接 / },{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{0,Alto,184 / 小衔接 / },{1,Teble,One_FOUR_Beat},
    {3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},
    {2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},{3,Teble,One_FOUR_Beat},
    {2,Teble,One_TWO_Beat} ,{0,Alto,184 * / 小衔接 / *},{2,Teble,One_TWO_Beat} ,{1,Teble,One_FOUR_Beat},
    {0,Alto,184 * / 小衔接 / *},{1,Teble,One_FOUR_Beat},
    /  / 第二次循环结束
    /  / 第四十七小节
    {1,Teble,One_TWO_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_Beat} ,{0,Alto,One_Beat} ,
    
    {0,Alto,One_Beat} ,
};
/ * USER CODE END PT * /
/ * Function definition -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FD * /
/ *
@Function name MUSIC_Begin*
@Introduce 开始播放音乐*
@Return NULL*
* * /
void MUSIC_Analysis(void){
    uint16_t MusicBeatNum = ((((sizeof(Music_Lone_Brave)) / 2) / 3) - 1);
    
    uint16_t MusicSpeed = Music_Lone_Brave[0][2];
    for(uint16_t i = 1;i< = MusicBeatNum;i +  + ){
        /  / BSP_Buzzer_SetFrequency(Tone_Index[Music_Lone_Brave[i][0]][Music_Lone_Brave[i][1]]);
        PassiveBuzzer_Set_Freq_Duty(Tone_Index[Music_Lone_Brave[i][0]][Music_Lone_Brave[i][1]], 50);
        /  / HAL_Delay(MusicSpeed / Music_Lone_Brave[i][2]);
        mdelay(MusicSpeed / Music_Lone_Brave[i][2]);
    }
}
/ * USER CODE END FD * /
/ * * (C) COPYRIGHT Lesterbor *END OF FILE* * /
void PlayMusic(void *pvParameters)
{
    (void)pvParameters;
    PassiveBuzzer_Init();
    
    while (1)
    {
        MUSIC_Analysis();
    }
}
```
```c
/ * Private variables
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN Variables * /
static StackType_t g_pulStackOfLightTask[128];
static StaticTask_t g_TCBOfLightTask;
static TaskHandle_t xLedTaksHandle;

static TaskHandle_t xColorTaskHandle;
static StackType_t g_pulStackOfColorTask[128];
static StaticTask_t g_TCBOfColorTask;
/ * USER CODE END Variables * /
/ * Definitions for defaultTask * /
osThreadId_t defaultTaskHandle;
const osThreadAttr_t defaultTask_attributes = {
    .name = "defaultTask",
    .stack_size = 128  4,
    .priority = (osPriority_t)osPriorityNormal,
};
/ * Private function prototypes
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FunctionPrototypes * /
extern void PlayMusic(void pvParameters);

void LED_TEST(void pvParameters)
{
    (void)pvParameters;
    Led_Test();
}

void ColorLED_TEST(void pvParameters)
{
    (void)pvParameters;
    ColorLED_Test();
}
/ * USER CODE END FunctionPrototypes * /
void StartDefaultTask(void argument);

void MX_FREERTOS_Init(void); / * (MISRA C 2004 rule 8.1) * /
/ *
@brief FreeRTOS initialization*
@param None*
@retval None*
* * /
void MX_FREERTOS_Init(void)
{
    / * USER CODE BEGIN Init * /
    TaskHandle_t xSoundTaskHandle;
    BaseType_t ret = 0;
    / * USER CODE END Init * /
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL, &defaultTask_attributes);
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    ret = xTaskCreate(PlayMusic, "SoundTask", 128, NULL, osPriorityNormal, &xSoundTaskHandle);
    xLedTaksHandle = xTaskCreateStatic(LED_TEST, "LedTaks", 128, NULL, osPriorityNormal, g_pulStackOfLightTask,
    &g_TCBOfLightTask);
    xColorTaskHandle = xTaskCreateStatic(ColorLED_TEST, "ColorTask", 128, NULL, osPriorityNormal, g_pulStackOfColorTask,
    &g_TCBOfColorTask);
    / * USER CODE END RTOS_THREADS * /
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
}
/ * USER CODE BEGIN Header_StartDefaultTask * /
/ *
@brief Function implementing the defaultTask thread.*
@param argument: Not used*
@retval None*
* * /
/ * USER CODE END Header_StartDefaultTask * /
void StartDefaultTask(void argument)
{
    / * USER CODE BEGIN StartDefaultTask * /
    / * Infinite loop * /
    LCD_Init();
    LCD_Clear();
    
    for (;;) {
        /  / Led_Test();
        /  / LCD_Test();
        /  / MPU6050_Test();
        /  / DS18B20_Test();
        /  / DHT11_Test(); /  / 读取失败
        /  / ActiveBuzzer_Test();
        /  / PassiveBuzzer_Test();
        /  / ColorLED_Test();
        IRReceiver_Test();
        /  / IRSender_Test();
        /  / LightSensor_Test();
        /  / IRObstacle_Test();
        /  / SR04_Test(); /  / 测距异常
        /  / W25Q64_Test();
        /  / RotaryEncoder_Test(); /  / 无法实现增加或者减少
        /  / Motor_Test();
        /  / Key_Test();
        /  / UART_Test();
    }
/ * USER CODE END StartDefaultTask * /
}
```
这一部分只需要按照所给的参数去创建就可以,
当遇到什么类型不知道的时候查看定义即可

#### 现象:

无源蜂鸣器放歌, 灯变化亮度和颜色, 板载LED闪加接受指令

### 估算任务栈的大小

按照之前反汇编所提及的知识, 保护一次现场需要保护所用16个寄存器(16 * 4),
局部变量需要RAM, 返回地址(LR寄存器当中的数值等等)也需要保护,
其中嵌套调用一次会有一个寄存器大小的返回地址,
每一层调用函数最多时用到除了前3个常规寄存器外的R4-R11以及LR寄存器,
这里说明嵌套调用最大是9个半字.

如果需要估算可以按照这样估算, 拿上面播放音乐的举例子,
第一层调用音乐函数, 第二层(看调用深度)是蜂鸣器的程序,
那一个还有几个变量外加那些HAL库函数的调用加上其中的结构体,
大概算下来是250多个字节, 不到128半字, 空间充足

### 互斥访问

工程文件:06_create_task_use_params

这里简单介绍一些互斥访问的概念和一个方法

#### 概念:

**什么是互斥访问？**

在 RTOS 中，互斥访问（Mutual Exclusion）
指的是一种机制，确保在任何时刻，只有一个任务（或线程）可以访问某个共享资源（如全局变量、硬件外设、内存缓冲区等）。其核心目的是防止竞态条件（Race
Condition） 导致的数据不一致或系统错误。

简单来说，就是"你用时我等待，我用时你等待"，避免多个任务同时操作同一资源。

**为什么需要互斥访问？**

在 RTOS
中，任务通常是并发执行的（通过时间片轮转或优先级抢占），如果多个任务同时访问共享资源，可能会引发以下问题：

数据竞争：任务 A 和任务 B 同时写入同一内存地址，导致数据被破坏。

不可重入问题：如 printf 函数不可重入，多任务调用时可能导致输出错乱。

优先级反转：高优先级任务因等待低优先级任务释放资源而阻塞。

### 实例

现在我想实现OLED屏幕的互斥访问, 让多个进程函数都可以使用互斥访问
耳熟能详, 创建任务:
```c
/ * USER CODE END FunctionPrototypes * /

void StartDefaultTask(void argument);

void MX_FREERTOS_Init(void); / * (MISRA C 2004 rule 8.1) * /

/
@brief FreeRTOS initialization
@param None
@retval None
* /
void MX_FREERTOS_Init(void)
{
    / * USER CODE BEGIN Init * /
    /  / TaskHandle_t xSoundTaskHandle;
    /  / BaseType_t ret = 0;
    
    / * USER CODE END Init * /
    
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL,
    &defaultTask_attributes);
    
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    
    / 使用同一个函数创建不同的任务 /
    xTaskCreate(LCD_PrintTask, "Task1", 128, NULL,
    osPriorityNormal, NULL);
    xTaskCreate(LCD_PrintTask, "Task2", 128, NULL,
    osPriorityNormal, NULL);
    xTaskCreate(LCD_PrintTask, "Task3", 128, NULL,
    osPriorityNormal, NULL**);
    
    / * USER CODE END RTOS_THREADS * /
    
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
}
```
创建完任务之后我们需要为每一个添加独一无二的信息,
确保在访问时可以正常运行, 在考虑完整性的情况下,
我们用结构体创建是最快的形式
```c
/ * USER CODE BEGIN Variables * /
/ * 任务结构体* /
typedef struct {
    uint8_t x;
    uint8_t y;
    char name[16];
} xFreertosTaskPrintInfo_t;

static xFreertosTaskPrintInfo_t g_Task1Info = {0, 0, "Task1"}; /  / 任务一的具体参数
static xFreertosTaskPrintInfo_t g_Task2Info = {0, 3, "Task2"}; /  / 任务二的具体参数
static xFreertosTaskPrintInfo_t g_Task3Info = {0, 6, "Task3"}; /  / 任务三的具体参数

BaseType_t g_LcdCanUse = 1; /  / 1: can use; 0: can not use
BaseType_t g_LcdCnt = 0; /  / 初始化标志位

/ * USER CODE END Variables * /
/ * Definitions for defaultTask * /
osThreadId_t defaultTaskHandle;
const osThreadAttr_t defaultTask_attributes = {
    .name = "defaultTask",
    .stack_size = 128 * 4,
    .priority = (osPriority_t)osPriorityNormal,
};

/ * Private function prototypes
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FunctionPrototypes * /

void LCD_PrintTask(void *pvParameters)
{
    xFreertosTaskPrintInfo_t *pTaskInfo = pvParameters;
    uint32_t cnt = 0;
    int len = 0;
    if (g_LcdCnt =  = 0) {
        LCD_Init();
        LCD_Clear();
        g_LcdCnt = 1;
    }
while (1) {
    if (g_LcdCanUse) {
        g_LcdCanUse = 0;
        
        len = LCD_PrintString(pTaskInfo - >x, pTaskInfo - >y, pTaskInfo - >name);
        len +  = LCD_PrintString(len, pTaskInfo - >y, ":");
        LCD_PrintSignedVal(len, pTaskInfo - >y, cnt +  + );
        
        g_LcdCanUse = 1;
    }
}
}
```
这个时候再将任务更改, 这样就可以获得一个完整的互斥访问的工程了
```c
xTaskCreate(LCD_PrintTask, "Task1", 128, &g_Task1Info,
osPriorityNormal, NULL);
xTaskCreate(LCD_PrintTask, "Task2", 128, &g_Task2Info,
osPriorityNormal, NULL);
xTaskCreate(LCD_PrintTask, "Task3", 128, &g_Task3Info,
osPriorityNormal, NULL);
```
**烧写运行之后, 大家就会发现为什么只有一个Task的数值在增加,**

**这是因为任务切换的太快, OLED打印函数又在其中占用了一大部分时间,
所以我们可以加一个延时来保证这个互斥访问程序可以继续运行**
```c
void LCD_PrintTask(void pvParameters)
{
    xFreertosTaskPrintInfo_t pTaskInfo = pvParameters;
    uint32_t cnt = 0;
    int len = 0;
    if (g_LcdCnt =  = 0) {
        LCD_Init();
        LCD_Clear();
        g_LcdCnt = 1;
    }
while (1) {
    if (g_LcdCanUse) {
        g_LcdCanUse = 0;
        
        len = LCD_PrintString(pTaskInfo - >x, pTaskInfo - >y,
        pTaskInfo - >name);
        len +  = LCD_PrintString(len, pTaskInfo - >y, ":");
        LCD_PrintSignedVal(len, pTaskInfo - >y, cnt +  + );
        
        g_LcdCanUse = 1;
    }
mdelay(50);
}
}
```
这样就可以见缝插针, 从整个任务运行的情况来看,
这个任务的有效占比就大大降低了

但是上面所采取的[BaseType_t
g_LcdCanUse]{.underline}并不是一个很好的方案,
当程序和外设躲起来就是失效了, 这里先埋下一个伏笔.

#### 现象:

OLED三个任务执行次数增加, 且相差不大

### 删除任务

工程文件:07_delete_task

这一块比较简单基础而且不吃香, 简单介绍一下

介绍一个删除任务的函数:
```c
void vTaskDelete( TaskHandle_t xTaskToDelete )
PRIVILEGED_FUNCTION;
```
下main讲一个具体用法:
```c
演示
C +  +
/ * Private variables
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN Variables * /
TaskHandle_t xgSoundTaskHandle;
BaseType_t g_ret = 0;

static StackType_t g_pulStackOfLightTask[128];
static StaticTask_t g_TCBOfLightTask;
static TaskHandle_t xLedTaksHandle;

static TaskHandle_t xColorTaskHandle;
static StackType_t g_pulStackOfColorTask[128];
static StaticTask_t g_TCBOfColorTask;
/ * USER CODE END Variables * /
/ * Definitions for defaultTask * /
osThreadId_t defaultTaskHandle;
const osThreadAttr_t defaultTask_attributes = {
    .name = "defaultTask",
    .stack_size = 128  4,
    .priority = (osPriority_t)osPriorityNormal,
};

/ * Private function prototypes
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FunctionPrototypes * /
void LED_TEST(void pvParameters)
{
    (void)pvParameters;
    Led_Test();
}

void ColorLED_TEST(void pvParameters)
{
    (void)pvParameters;
    ColorLED_Test();
}
/ * USER CODE END FunctionPrototypes * /

......

/
@brief FreeRTOS initialization
@param None
@retval None
/
void MX_FREERTOS_Init(void)
{
    / * USER CODE BEGIN Init * /
    
    / * USER CODE END Init * /
    
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL, &defaultTask_attributes);
    xLedTaksHandle = xTaskCreateStatic(LED_TEST, "LedTaks", 128, NULL, osPriorityNormal,
    g_pulStackOfLightTask, &g_TCBOfLightTask);
    xColorTaskHandle = xTaskCreateStatic(ColorLED_TEST, "ColorTask", 128, NULL, osPriorityNormal,
    g_pulStackOfColorTask, &g_TCBOfColorTask);
    
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    
    / * USER CODE END RTOS_THREADS * /
    
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
}

/ * USER CODE BEGIN Header_StartDefaultTask * /
/
@brief Function implementing the defaultTask thread.
@param argument: Not used
@retval None
/
/ * USER CODE END Header_StartDefaultTask * /
void StartDefaultTask(void argument)
{
    / * USER CODE BEGIN StartDefaultTask * /
    / * Infinite loop * /
    LCD_Init();
    LCD_Clear();
    IRReceiver_Init();
    PassiveBuzzer_Init();
    
    uint8_t dev = 0;
    uint8_t data = 0;
    
    LCD_PrintString(0, 0, "Wait Controll");
    
    while (1) {
        / 接收遥控器信号进行判断 /
        if (0 =  = IRReceiver_Read(&dev, &data)) {
            / 创建任务 /
            / 当数据为播放按键时播放 /
            if (data =  = 0xa8) {
                extern void PlayMusic(void *pvParameters);
                if (xgSoundTaskHandle =  = NULL) {
                    LCD_ClearLine(0, 0);
                    LCD_PrintString(0, 0, "Playing Music");
                    g_ret = xTaskCreate(PlayMusic, "SoundTask", 128, NULL, osPriorityNormal, &xgSoundTaskHandle);
                }
        }
    / 删除任务 /
    / 当数据为POWER键时暂停 /
    else if (data =  = 0xa2) {
        if (xgSoundTaskHandle != NULL) {
            vTaskDelete(xgSoundTaskHandle);
            xgSoundTaskHandle = NULL;
            PassiveBuzzer_Control(0);
            LCD_ClearLine(0, 0);
            LCD_PrintString(0, 0, "Wait Controll");
        }
}
}
}
/ * USER CODE END StartDefaultTask * /
}
```
就这样创建之后进行判断结束任务, 但是这个用法其实不大好,
因为频繁创建和删除很容易导致产生内存碎片化现象

#### 现象

按下遥控三角形按钮播放音乐, 按下红色的开关按钮删除音乐,
OLED会显示对应音乐状态, 板载LED闪烁, 全彩RGB改变颜色与亮度

### 任务优先级与阻塞与任务调度

工程文件:08_task_priority, 09_task_suspend, 10_idle_task, 11_task_delay,
12_task_sync_exclusion

**开胃菜**

先举个例子

根据上一部分的实验现象, 大家肯定都发现了这个音乐播放很难绷,
那怎么处理呢?

哎, 点子王可能又说了, 是不是可以让任务的优先级更高,
使得任务更加丝滑地执行呢?

是的捏, 确实可以如此, 那事不宜迟, 我们直接对上期工程进行修改.

将freertos.c的默认任务(就是那个播放音乐的任务更改成下main这样)
```c
/ * USER CODE BEGIN Header_StartDefaultTask * /
/
@brief Function implementing the defaultTask thread.
@param argument: Not used
@retval None
/
/ * USER CODE END Header_StartDefaultTask * /
void StartDefaultTask(void argument)
{
    / * USER CODE BEGIN StartDefaultTask * /
    / * Infinite loop * /
    LCD_Init();
    LCD_Clear();
    IRReceiver_Init();
    PassiveBuzzer_Init();
    
    uint8_t dev = 0;
    uint8_t data = 0;
    
    LCD_PrintString(0, 0, "Wait Controll");
    
    while (1) {
        / 接收遥控器信号进行判断 /
        if (0 =  = IRReceiver_Read(&dev, &data)) {
            / 创建任务 /
            / 当数据为播放按键时播放 /
            if (data =  = 0xa8) {
                extern void PlayMusic(void pvParameters);
                if (xgSoundTaskHandle =  = NULL) {
                    LCD_ClearLine(0, 0);
                    LCD_PrintString(0, 0, "Playing Music");
                    g_ret = xTaskCreate(PlayMusic, "SoundTask", 128, NULL,
                    osPriorityNormal + 1, &xgSoundTaskHandle);
                }
        }
    / 删除任务 /
    / 当数据为POWER键时暂停 /
    else if (data =  = 0xa2) {
        if (xgSoundTaskHandle != NULL) {
            vTaskDelete(xgSoundTaskHandle);
            xgSoundTaskHandle = NULL;
            PassiveBuzzer_Control(0);
            LCD_ClearLine(0, 0);
            LCD_PrintString**(0, 0, "Wait Controll");
        }
}
}
}
/ * USER CODE END StartDefaultTask * /
}
```
烧录之后, 却发现一个问题, 确实音乐更加丝滑了,
但是却没办法结束音乐的进程了, 其他进程(板载LED的闪烁, RGB的变色)都暂停了

为什么呢?

其实是因为本来使用的函数当中有阻塞性的延时函数并且这个延时函数它对CPU的资源掌控很强,
导致了无法退出这个任务
```c
void MUSIC_Analysis(void){
    uint16_t MusicBeatNum = ((((sizeof(Music_Lone_Brave)) / 2) / 3) - 1);
    
    uint16_t MusicSpeed = Music_Lone_Brave[0][2];
    for(uint16_t i = 1;i< = MusicBeatNum;i +  + ){
        /  / BSP_Buzzer_SetFrequency(Tone_Index[Music_Lone_Brave[i][0]][Music_Lone_Brave[i][1]]);
        PassiveBuzzer_Set_Freq_Duty(Tone_Index[Music_Lone_Brave[i][0]][Music_Lone_Brave[i][1]],
        50);
        /  / HAL_Delay(MusicSpeed / Music_Lone_Brave[i][2]);
        mdelay(MusicSpeed / Music_Lone_Brave[i][2]);
    }
}

/ * USER CODE END FD * /
/ (C) COPYRIGHT Lesterbor *END OF FILE /

void PlayMusic(void *pvParameters)
{
    (void)pvParameters;
    PassiveBuzzer_Init();
    
    while (1)
    {
        MUSIC_Analysis();
    }
}
```
那么这时候就有人要问了, 主播主播,
有没有什么可以阻塞延时又可以不影响CPU资源对于其他任务的调度的呢?

有的, 朋友, 有的:
```c
void vTaskDelay(ms);
```
这个函数就可以做到这样, 它背后涉及了任务状态与调度理论, 现在结束开胃菜,
下一步.

#### 任务状态

结合之前的文档, 我们可以发现,
任务状态实际上时是两种运行状态(Running)和准备状态(Ready), 但是实际上,
任务除了这两种状态还有挂起状态(Suspend)和阻塞状态(**Blocked**)

**介绍**

在 RTOS（实时操作系统）中，**Suspend（挂起）** 和 **Resume（恢复）**
是任务管理中的两个关键操作，直接影响任务的状态转换和调度行为。

![](media/media/image25.png)

**✅ Suspend（挂起）状态**

**定义**：任务被挂起后，**不再参与调度器调度**，即使它处于就绪条件也不会被运行。

**触发方式**：

显式调用如 vTaskSuspend(TaskHandle_t xTask) 或 os_task_suspend() 等
API。

如果参数为 NULL，表示挂起当前任务自身。

**状态表现**：

任务从就绪列表或阻塞列表中移除，**移入挂起任务列表**（xSuspendedTaskList）。

**不会消耗 CPU 资源**。

**不会自动恢复**，必须显式调用 Resume API。

**✅ Resume（恢复）状态**

**定义**：将挂起的任务重新置为就绪态，使其可以再次被调度运行。

**触发方式**：

显式调用如 vTaskResume(TaskHandle_t xTask) 或
xTaskResumeFromISR()（中断安全版本）。

**状态表现**：

任务从挂起列表中移除，**重新加入就绪列表**。

如果其优先级高于当前任务，调度器会立即进行任务切换。

**⚠️ 注意事项**

  ----------------------------------- ------------------------------------
  项目                                说明

  **不可自我恢复**                    挂起态的任务不能调用 vTaskResume()
                                      自己恢复自己。

  **非阻塞**                          与 vTaskDelay()
                                      不同，挂起态没有超时机制，只能通过
                                      Resume 解除。

  **中断上下文**                      xTaskResumeFromISR()
                                      是中断安全版本，需配合
                                      portYIELD_FROM_ISR()
                                      判断是否触发任务切换。
  ----------------------------------- ------------------------------------

把 FreeRTOS
的任务状态想像成一家只有**一个厨师（CPU）**的小餐馆里，**几位客人（任务）**排队点菜：

**Ready（就绪）**站在收银台前排队、手里举着菜单，随时准备轮到自己做菜。

**Running（运行）**现在正在灶台前动铲子炒菜的那位客人。

**Blocked（阻塞）**因为缺食材、等外卖或等盘子洗好，只能坐到旁边椅子上打瞌睡，厨师暂时不会叫他。

**Suspended（挂起）**老板把他"请"到后厨储藏间里**贴封条**，门一关------厨师完全忘了这个人，除非老板亲自撕封条，否则永远不出来。

**Ready 是排队，Running 是炒菜，Blocked 是打瞌睡等条件，Suspended
是被贴封条关小黑屋。**

**🎯 总结一句话**
```c
演示
Suspend 是"暂停任务的运行"，Resume 是"将任务重新唤醒"。
```
这两个状态在调试、低功耗管理、临界资源控制等场景中非常实用。

#### 例子

只需要在上个工程更改成这样就行
```c
void StartDefaultTask(void *argument)
{
    .......
    BaseType_t state = 0;
    
    .......
    
    else
    {
        if (state)
        {
            LCD_ClearLine(0, 0);
            LCD_PrintString(0, 0, "Suspend Task");
            vTaskSuspend(xSoundTaskHandle);
            PassiveBuzzer_Control(0);
            state = 0;
        }
    else
    {
        LCD_ClearLine(0, 0);
        LCD_PrintString(0, 0, "Resume Task");
        vTaskResume(xSoundTaskHandle);
        state = 1;
    }
}

......
}
```
##### 现象

开始音乐之后, 再按播放键变为暂停;暂停时就再按就播放;板载LED闪烁, RGB闪烁

### 任务调度与管理

#### 机制

高优先级任务未执行完, 低优先级任务无法运行

一旦高优先级任务就绪, 低优先级任务无法运行, 高优先级任务马上执行

最高优先级有多个任务, 将会通过时间片流转执行任务

相同优先级任务时间片流转执行, 不同优先级任务执行顺序根据优先级大小执行

通俗来说, 高优先级任务就是大哥, 低优先级任务就是小弟, 多个大哥轮流话事

这就好像:

只要高优先级的客人还在炒菜，低优先级的客人连灶台边都挨不着。

高优先级客人一回到收银台（就绪），厨师立刻把他请上去，把正在炒的低优先级客人直接"扒"下来。

如果有好几个最高优先级的客人同时排队，就每人炒固定的一分钟，轮流上灶（时间片轮转）。

同一优先级才用时间片"排队炒"；不同优先级只看谁的号码牌数字更小（数字越小越优先）。

下面我们回到具体的CubeMx界面看看有多少个优先级

![](media/media/image26.png)

这个截图当中, 可以看到MAX_PRIORITIES有56个优先级

那这么多优先级怎么区分呢?

要想知道这个下main就要涉及到链表管理和优先级

在task.c(350行左右)当中存在着一些揭示秘密的代码:
```c
/ * Lists for ready and blocked tasks.
  --------------------
xDelayedTaskList1 and xDelayedTaskList2 could be move to function
scople but
doing so breaks some kernel aware debuggers and debuggers that rely on
removing
the static qualifier. * /
PRIVILEGED_DATA static List_t pxReadyTasksLists[
configMAX_PRIORITIES ]; / *< Prioritised ready tasks. * /
PRIVILEGED_DATA static List_t xDelayedTaskList1; / *< Delayed
tasks. * /
PRIVILEGED_DATA static List_t xDelayedTaskList2; / *< Delayed
tasks (two lists are used - one for delays that have overflowed the
current tick count. * /

......

#if ( INCLUDE_vTaskSuspend =  = 1 )

PRIVILEGED_DATA static List_t xSuspendedTaskList; / *< Tasks that
are currently suspended. * /

#endif
```
其中[pxReadyTasksLists]{.underline}的大小是[**configMAX_PRIORITIES**]{.underline}

这个宏定义在RTOS的Config文件当中亦有记载
```c
#define configMAX_PRIORITIES ( 56 )
```
[pxReadyTasksLists]{.underline}就是优先级组别的数组,
这个存放着链表头数组, 这是一个双向链表头类型的数组,

下面是它的结构示意图:

![](media/media/image27.jpeg)

这里仍需补充的是,
TCB结构体挂载信息只有处在Ready和Running的任务才会挂载在对应优先级的双向链表头类型上

xDelayedTaskList1与xDelayedTaskList2这个则是为了管理处于Blocked状态当中的任务,
有两个这个结构则是为了防止定时器溢出

至于这个List_t
xSuspendedTaskList大家肯定都可以猜得到这是用于存放已经挂起的任务的双向链表头

我想仅仅只有这些链表头大家肯定都会很懵圈,
下面我们再回到函数调用的角度具体看看这个任务调度究竟是个怎么回事<参照上一个工程>

先看main.c
```c
int main(void)
{
    
    ......
    
    MX_FREERTOS_Init();
    
    / * Start scheduler * /
    osKernelStart();
    
    ......
    
}
```
其中调用的[**MX_FREERTOS_Init**()则是:]{.underline}
```c
void MX_FREERTOS_Init(void)
{
    ......
    
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL,
    &defaultTask_attributes);
    xLedTaksHandle = xTaskCreateStatic(LED_TEST, "LedTaks", 128,
    NULL, osPriorityNormal, g_pulStackOfLightTask, &g_TCBOfLightTask);
    xColorTaskHandle = xTaskCreateStatic(ColorLED_TEST,
    "ColorTask", 128, NULL, osPriorityNormal, g_pulStackOfColorTask,
    &g_TCBOfColorTask);
    
    ......
    
}
```
创建了一个默认任务(music),加自己新建的两个任务(LED, ColorLED)

Tips:默认优先级是24

![](media/media/image28.jpeg)

其中每次调用创建的时候都会根据对应的优先级分组放到对应的优先级

此事在task.c的xCreateTask()当中亦有记载:
```c
#if( configSUPPORT_DYNAMIC_ALLOCATION =  = 1 )

BaseType_t xTaskCreate( TaskFunction_t pxTaskCode,
const char  const pcName, / *lint !e971 Unqualified char types are
allowed for strings and single characters only. * /
const configSTACK_DEPTH_TYPE usStackDepth,
void  const pvParameters,
UBaseType_t uxPriority,
TaskHandle_t  const pxCreatedTask )
{
    TCB_t pxNewTCB;
    BaseType_t xReturn;
    
    / * If the stack grows down then allocate the stack then the TCB so the
    stack
    does not grow into the TCB. Likewise if the stack grows up then
    allocate
    the TCB then the stack. * /
    #if( portSTACK_GROWTH > 0 )
    {
        / * Allocate space for the TCB. Where the memory comes from depends on
        the implementation of the port malloc function and whether or not
        static
        allocation is being used. * /
        pxNewTCB = ( TCB_t  ) pvPortMalloc( sizeof( TCB_t ) );
        
        if( pxNewTCB != NULL )
        {
            / * Allocate space for the stack used by the task being created.
            The base of the stack memory stored in the TCB so the task can
            be deleted later if required. * /
            pxNewTCB - >pxStack = ( StackType_t  ) pvPortMalloc( ( ( ( size_t
            ) usStackDepth )  sizeof( StackType_t ) ) ); / *lint !e961 MISRA
            exception as the casts are only redundant for some ports. * /
            
            if( pxNewTCB - >pxStack =  = NULL )
            {
                / * Could not allocate the stack. Delete the allocated TCB. * /
                vPortFree( pxNewTCB );
                pxNewTCB = NULL;
            }
    }
}
#else / * portSTACK_GROWTH * /
{
    StackType_t pxStack;
    
    / * Allocate space for the stack used by the task being created. * /
    pxStack = pvPortMalloc( ( ( ( size_t ) usStackDepth )  sizeof(
    StackType_t ) ) ); / *lint !e9079 All values returned by pvPortMalloc()
    have at least the alignment required by the MCU's stack and this
    allocation is the stack. * /
    
    if( pxStack != NULL )
    {
        / * Allocate space for the TCB. * /
        pxNewTCB = ( TCB_t  ) pvPortMalloc( sizeof( TCB_t ) ); / *lint
        !e9087 !e9079 All values returned by pvPortMalloc() have at least the
        alignment required by the MCU's stack, and the first member of TCB_t
        is always a pointer to the task's stack. * /
        
        if( pxNewTCB != NULL )
        {
            / * Store the stack location in the TCB. * /
            pxNewTCB - >pxStack = pxStack;
        }
    else
    {
        / * The stack cannot be used as the TCB was not created. Free
        it again. * /
        vPortFree( pxStack );
    }
}
else
{
    pxNewTCB = NULL;
}
}
#endif / * portSTACK_GROWTH * /

if( pxNewTCB != NULL )
{
    #if( tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE != 0 ) / *lint
    !e9029 !e731 Macro has been consolidated for readability reasons. * /
    {
        / * Tasks can be created statically or dynamically, so note this
        task was created dynamically in case it is later deleted. * /
        pxNewTCB - >ucStaticallyAllocated =
        tskDYNAMICALLY_ALLOCATED_STACK_AND_TCB;
    }
#endif / * tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE * /

prvInitialiseNewTask( pxTaskCode, pcName, ( uint32_t )
usStackDepth, pvParameters, uxPriority, pxCreatedTask, pxNewTCB,
NULL );
prvAddNewTaskToReadyList( pxNewTCB );
xReturn = pdPASS;
}
else
{
    xReturn = errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY;
}

return xReturn;
}
```
其中对指针等等进行了判断, 对任务进行了分组,
注意这个函数[**prvAddNewTaskToReadyList**( pxNewTCB )]{.underline}

这个函数创建了新的ReadyList, 跳转到这个函数
```c
演示
C +  +
task.c
static void prvAddNewTaskToReadyList( TCB_t *pxNewTCB )
{
    / * Ensure interrupts don't access the task lists while the lists are
    being
    updated. * /
    taskENTER_CRITICAL();
    {
        uxCurrentNumberOfTasks +  + ;
        if( pxCurrentTCB =  = NULL )
        {
            / * There are no other tasks, or all the other tasks are in
            the suspended state - make this the current task. * /
            pxCurrentTCB = pxNewTCB;
            
            if( uxCurrentNumberOfTasks =  = ( UBaseType_t ) 1 )
            {
                / * This is the first task to be created so do the preliminary
                initialisation required. We will not recover if this call
                fails, but we will report the failure. * /
                prvInitialiseTaskLists();
            }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
}
else
{
    / * If the scheduler is not already running, make this task the
    current task if it is the highest priority task to be created
    so far. * /
    if( xSchedulerRunning =  = pdFALSE )
    {
        if( pxCurrentTCB - >uxPriority < = pxNewTCB - >uxPriority )
        {
            pxCurrentTCB = pxNewTCB;
        }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
}

uxTaskNumber +  + ;

#if ( configUSE_TRACE_FACILITY =  = 1 )
{
    / * Add a counter into the TCB for tracing only. * /
    pxNewTCB - >uxTCBNumber = uxTaskNumber;
}
#endif / * configUSE_TRACE_FACILITY * /
traceTASK_CREATE( pxNewTCB );

prvAddTaskToReadyList( pxNewTCB );

portSETUP_TCB( pxNewTCB );
}
taskEXIT_CRITICAL();

if( xSchedulerRunning != pdFALSE )
{
    / * If the created task is of a higher priority than the current task
    then it should run now. * /
    if( pxCurrentTCB - >uxPriority < pxNewTCB - >uxPriority )
    {
        taskYIELD_IF_USING_PREEMPTION();
    }
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
```
注意到其中有这么一段代码:
```c
/ * If the scheduler is not already running, make this task the
current task if it is the highest priority task to be created
so far. * /
if( xSchedulerRunning =  = pdFALSE )
{
    if( pxCurrentTCB - >uxPriority < = pxNewTCB - >uxPriority )
    {
        pxCurrentTCB = pxNewTCB;
    }
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
```
当前TCB结构体的指针<pxCurrentTCB>发生了赋值变化

那我们想象一下调用3次之后发生的故事

(这个时候在优先组数组当中是下面这样的情况belike):

![](media/media/image29.jpeg)

看这个TCB指针的位置, 再回想一下那个双向链表头的数据类型,
你可能就猜到为什么之前我们创建多任务时候Task3计数多了吧

好的,
创建这一部分之后再回到main.c继续调用函数[**osKernelStart**()]{.underline}<启动调度的函数>
```c
osStatus_t osKernelStart (void) {
    osStatus_t stat;
    
    if (IS_IRQ()) {
        stat = osErrorISR;
    }
else {
    if (KernelState =  = osKernelReady) {
        / * Ensure SVC priority is at the reset value * /
        SVC_Setup();
        / * Change state to enable IRQ masking check * /
        KernelState = osKernelRunning;
        / * Start the kernel scheduler * /
        vTaskStartScheduler();
        stat = osOK;
    } else {
    stat = osError;
}
}

return (stat);
}
```
这个函数里头又调用这个函数[**vTaskStartScheduler**()]{.underline}
```c
演示
C +  +
task.c
void vTaskStartScheduler( void )
{
    
    ......
    
    xIdleTaskHandle = xTaskCreateStatic( prvIdleTask,
    configIDLE_TASK_NAME,
    ulIdleTaskStackSize,
    ( void * ) NULL, / *lint !e961. The cast is not redundant for all
    compilers. * /
    portPRIVILEGE_BIT, / * In effect ( tskIDLE_PRIORITY |
    portPRIVILEGE_BIT ), but tskIDLE_PRIORITY is zero. * /
    pxIdleTaskStackBuffer,
    pxIdleTaskTCBBuffer ); / *lint !e961 MISRA exception, justified as it
    is not a redundant explicit cast to all supported compilers. * /
    
    ......
}
```
创建了一个空闲任务**[prvIdleTask,]{.underline}** 它的优先级是0,
这个时候再看优先级数组就是下面这样了

![](media/media/image30.jpeg)

这个空闲任务非常重要, 后面可能还会讲到

一句话：

没有空闲任务，调度器就**永远停不下来**，CPU
一闲下来就会"无处可去"，导致系统异常。

具体原因：

**调度器不能空转**FreeRTOS
规定：调度器启动后，**必须始终有一个就绪态任务**。如果所有应用任务都因等待事件而阻塞，就绪队列就空了，调度器将陷入"无任务可跑"的状态。空闲任务正是填补这个空档的"备胎"。

**优先级 0 保证它永远最低**优先级 0 意味着：•
只要存在任何应用任务就绪，空闲任务立即被抢占；•
只有所有高优先级任务都阻塞/挂起，它才得到 CPU。

**额外功能都挂在空闲任务里**• 内存回收：删除任务后真正的
TCB/栈释放，在空闲任务里完成（vTaskDelete() 只是标记）。• Tickless
低功耗：在空闲钩子 vApplicationIdleHook() 里把 CPU 睡下去。•
运行时间统计、后台自检等。

所以空闲任务就是"系统兜底工"------CPU
闲着也得让它有活干，并且顺带把后台杂活全包圆。

对于同优先级的任务，它们"轮流"执行。怎么轮流？你执行一会，我执行一会。

"一会"怎么定义？

人有心跳，心跳间隔基本恒定。

FreeRTOS中也有心跳，它使用定时器产生固定间隔的中断。这叫Tick、滴答，比如每10ms发生一次时钟中断。

如下图：

假设t1、t2、t3发生时钟中断

两次中断之间的时间被称为时间片(time slice、tick period)

时间片的长度由configTICK_RATE_HZ
决定，假设configTICK_RATE_HZ为100，那么时间片长度就是10ms

![](media/media/image31.png)

有了这个基础, 回想我们之前配置的时钟基准是systick,
我们每隔1ms就会直接仅从一次tick中断

下面我们按照之前写的程序走一遍优先级和这个tick中断时如何实现任务调度的

这个是未按下音乐播放, 大家就这样同优先级时间片轮转

![](media/media/image32.jpeg)

再看按下音乐播放按键之后

![](media/media/image33.jpeg)

假设按下音乐暂停, 那就是像suspend组中加入music_on的任务, 直接停止,
此时ready任务状态如

![](media/media/image34.png)

#### 空闲任务详细讲解

下面我们举一个例子引入:

打开之前的音乐播放程序:

修改Led_Test(void)函数:
```c
void Led_Test(void)
{
    Led_Init();
    
    while (1)
    {
        Led_Control(LED_GREEN, 1);
        mdelay(500);
        
        Led_Control(LED_GREEN, 0);
        mdelay(500);
    }
}
```
为
```c
void Led_Test(void)
{
    Led_Init();
    
    for (uint8_t i = 0; i < 10; i +  + )
    {
        Led_Control(LED_GREEN, 1);
        mdelay(50);
        
        Led_Control(LED_GREEN, 0);
        mdelay(50);
    }
}
```
这种任务是一个运行有限次的任务, 也就是说运行一定时长后它就销毁了,
但是如果这样烧录进去会死机,
为什么呢?为什么不是直接不管这个任务去运行其他任务呢?能不能正常返回呢?

这里再提及之前的函数(创建任务进行对任务初始化的函数)
```c
static void prvInitialiseNewTask( TaskFunction_t pxTaskCode,
const char  const pcName, / *lint !e971 Unqualified char types are
allowed for strings and single characters only. * /
const uint32_t ulStackDepth,
void  const pvParameters,
UBaseType_t uxPriority,
TaskHandle_t  const pxCreatedTask,
TCB_t pxNewTCB,
const MemoryRegion_t * const xRegions )
```
中调用的函数[prvInitialiseNewTask()]{.underline},
```c
prvInitialiseNewTask( pxTaskCode, pcName, ( uint32_t ) usStackDepth,
pvParameters, uxPriority, pxCreatedTask, pxNewTCB, NULL );
```
它当中调用了对任务的栈初始化的函数:
```c
StackType_t pxPortInitialiseStack( StackType_t pxTopOfStack,
TaskFunction_t pxCode, void pvParameters )
{
    / * Simulate the stack frame as it would be created by a context
    switch
    interrupt. * /
    pxTopOfStack -  - ; / * Offset added to account for the way the MCU uses
    the stack on entry / exit of interrupts. * /
    pxTopOfStack = portINITIAL_XPSR; / * xPSR * /
    pxTopOfStack -  - ;
    pxTopOfStack = ( ( StackType_t ) pxCode ) & portSTART_ADDRESS_MASK;
    / * PC * /
    pxTopOfStack -  - ;
    pxTopOfStack = ( StackType_t ) prvTaskExitError; / * LR * /
    
    pxTopOfStack -  = 5; / * R12, R3, R2 and R1. * /
    *pxTopOfStack = ( StackType_t ) pvParameters; / * R0 * /
    pxTopOfStack -  = 8; / * R11, R10, R9, R8, R7, R6, R5 and R4. * /
    
    return pxTopOfStack;
}
```
这个函数里记载了返回的伪造的LR地址,
但是你看这个是[prvTaskExitError(退出失败)]{.underline}的具体执行逻辑
```c
static void prvTaskExitError( void )
{
    / * A function that implements a task must not exit or attempt to
    return to
    its caller as there is nothing to return to. If a task wants to exit
    it
    should instead call vTaskDelete( NULL ).
    
    Artificially force an assert() to be triggered if configASSERT() is
    defined, then stop here so application writers can catch the error.
    * /
    configASSERT( uxCriticalNesting =  = ~0UL );
    portDISABLE_INTERRUPTS();
    for( ;; );
}
```
这可以得知 ,当我们退出的时候并不会执行其他任务,
反而会关闭所有中断(Tick中断也没了),
进入死循环并且无法执行其他任务(依赖于Tick中断)

所以相对正确的方法应该是这样:
```c
void Led_Test(void)
{
    Led_Init();
    
    for (uint8_t i = 0; i < 10; i +  + )
    {
        Led_Control(LED_GREEN, 1);
        mdelay(50);
        
        Led_Control(LED_GREEN, 0);
        mdelay(50);
    }
vTaskDelete(NULL);
}
```
但这真的完全正确吗?显然不是, 原因如下:
```c
演示
C +  +
#if ( INCLUDE_vTaskDelete =  = 1 )

void vTaskDelete( TaskHandle_t xTaskToDelete )
{
    TCB_t *pxTCB;
    
    taskENTER_CRITICAL();
    {
        / * If null is passed in here then it is the calling task that is
        being deleted. * /
        pxTCB = prvGetTCBFromHandle( xTaskToDelete );
        
        / * Remove task from the ready list. * /
        if( uxListRemove( &( pxTCB - >xStateListItem ) ) =  = ( UBaseType_t ) 0 )
        {
            taskRESET_READY_PRIORITY( pxTCB - >uxPriority );
        }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }

/ * Is the task waiting on an event also? * /
if( listLIST_ITEM_CONTAINER( &( pxTCB - >xEventListItem ) ) != NULL )
{
    ( void ) uxListRemove( &( pxTCB - >xEventListItem ) );
}
else
{
    mtCOVERAGE_TEST_MARKER();
}

/ * Increment the uxTaskNumber also so kernel aware debuggers can
detect that the task lists need re - generating. This is done before
portPRE_TASK_DELETE_HOOK() as in the Windows port that macro will
not return. * /
uxTaskNumber +  + ;

if( pxTCB =  = pxCurrentTCB )
{
    / * A task is deleting itself. This cannot complete within the
    task itself, as a context switch to another task is required.
    Place the task in the termination list. The idle task will
    check the termination list and free up any memory allocated by
    the scheduler for the TCB and stack of the deleted task. * /
    vListInsertEnd( &xTasksWaitingTermination, &( pxTCB - >xStateListItem )
    );
    
    / * Increment the ucTasksDeleted variable so the idle task knows
    there is a task that has been deleted and that it should therefore
    check the xTasksWaitingTermination list. * /
    +  + uxDeletedTasksWaitingCleanUp;
    
    / * The pre - delete hook is primarily for the Windows simulator,
    in which Windows specific clean up operations are performed,
    after which it is not possible to yield away from this task -
    hence xYieldPending is used to latch that a context switch is
    required. * /
    portPRE_TASK_DELETE_HOOK( pxTCB, &xYieldPending );
}
else
{
    -  - uxCurrentNumberOfTasks;
    prvDeleteTCB( pxTCB );
    
    / * Reset the next expected unblock time in case it referred to
    the task that has just been deleted. * /
    prvResetNextTaskUnblockTime();
}

traceTASK_DELETE( pxTCB );
}
taskEXIT_CRITICAL();

/ * Force a reschedule if it is the currently running task that has
just
been deleted. * /
if( xSchedulerRunning != pdFALSE )
{
    if( pxTCB =  = pxCurrentTCB )
    {
        configASSERT( uxSchedulerSuspended =  = 0 );
        portYIELD_WITHIN_API();
    }
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
}

#endif / * INCLUDE_vTaskDelete * /
```
vTaskDelete(NULL)函数只是将任务覆盖或者移走了, 并没真正的删除,
怎样真正的删除呢?

打个比方, vTaskDelete(NULL)是自杀, 无人收尸, 需要空闲任务收尸;
在任务B当中执行vTaskDelete(TaskA)则是他杀, 这个时候有人收尸

**收尸区别**

**✅ 1. 执行上下文不同**

![](media/media/image38.png)

如图中所示, 这是我们音乐播放多任务程序的ready&running任务队列,
根据之前的代码,
其中只有处于优先级25的music-on任务使用**[vTaskDelay()]{.underline}**

其他要么使用阻塞延时但是不释放CPU资源的延时函数, 要么就是不用延时函数

这就造成了即使我们的Led_Task结束任务,
还有另外2个同优先级(24)的和一个高优先级的阻止着收Led_Task的尸

但是优先级25的任务里头有个**[vTaskDelay()]{.underline}**,
人家至少还会放手一会, 而剩下的两个任务的一直不放手,
这就导致了无法顺利地销毁任务

因此插入我们在任务中良好的编程习惯:

**编程习惯**

事件驱动, 当事件发生时才进行任务

延时函数不要用死循环, 不要抓着CPU资源不放手

我们根据上面来改一下任务函数
```c
void ColorLED_Test(void)
{
    uint32_t color = 0;
    
    ColorLED_Init();
    
    while (1)
    {
        /  / LCD_PrintString(0, 0, "Show Color: ");
        /  / LCD_PrintHex(0, 2, color, 1);
        
        ColorLED_Set(color);
        
        color +  = 200000;
        color & = 0x00ffffff;
        vTaskDelay(1000);
    }
}
```
另一个红外接收确定音乐是否运行的就不改了, 需要更好的响应

经过上面的修改, 那就可以更好的去清理任务了, 当然这还不够完善, 后面再说

再看那个空闲任务的函数
```c
static portTASK_FUNCTION( prvIdleTask, pvParameters )
{
    / * Stop warnings. * /
    ( void ) pvParameters;
    
    / THIS IS THE RTOS IDLE TASK - WHICH IS CREATED AUTOMATICALLY WHEN
    THE
    SCHEDULER IS STARTED. /
    
    / * In case a task that has a secure context deletes itself, in which
    case
    the idle task is responsible for deleting the task's secure context,
    if
    any. * /
    portTASK_CALLS_SECURE_FUNCTIONS();
    
    for( ;; )
    {
        / * See if any tasks have deleted themselves - if so then the idle
        task
        is responsible for freeing the deleted task's TCB and stack. * /
        prvCheckTasksWaitingTermination();
        
        #if ( configUSE_PREEMPTION =  = 0 )
        {
            / * If we are not using preemption we keep forcing a task switch to
            see if any other task has become available. If we are using
            preemption we don't need to do this as any task becoming available
            will automatically get the processor anyway. * /
            taskYIELD();
        }
    #endif / * configUSE_PREEMPTION * /
    
    #if ( ( configUSE_PREEMPTION =  = 1 ) && ( configIDLE_SHOULD_YIELD =  = 1 )
    )
    {
        / * When using preemption tasks of equal priority will be
        timesliced. If a task that is sharing the idle priority is ready
        to run then the idle task should yield before the end of the
        timeslice.
        
        A critical region is not required here as we are just reading from
        the list, and an occasional incorrect value will not matter. If
        the ready list at the idle priority contains more than one task
        then a task other than the idle task is ready to execute. * /
        if( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ tskIDLE_PRIORITY ]
        ) ) > ( UBaseType_t ) 1 )
        {
            taskYIELD();
        }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
}
#endif / * ( ( configUSE_PREEMPTION =  = 1 ) && ( configIDLE_SHOULD_YIELD
=  = 1 ) ) * /

#if ( configUSE_IDLE_HOOK =  = 1 )
{
    extern void vApplicationIdleHook( void );
    
    / * Call the user defined function from within the idle task. This
    allows the application designer to add background functionality
    without the overhead of a separate task.
    NOTE: vApplicationIdleHook() MUST NOT, UNDER ANY CIRCUMSTANCES,
    CALL A FUNCTION THAT MIGHT BLOCK. * /
    vApplicationIdleHook();
}
#endif / * configUSE_IDLE_HOOK * /

/ * This conditional compilation should use inequality to 0, not
equality
to 1. This is to ensure portSUPPRESS_TICKS_AND_SLEEP() is called when
user defined low power mode implementations require
configUSE_TICKLESS_IDLE to be set to a value other than 1. * /
#if ( configUSE_TICKLESS_IDLE != 0 )
{
    TickType_t xExpectedIdleTime;
    
    / * It is not desirable to suspend then resume the scheduler on
    each iteration of the idle task. Therefore, a preliminary
    test of the expected idle time is performed without the
    scheduler suspended. The result here is not necessarily
    valid. * /
    xExpectedIdleTime = prvGetExpectedIdleTime();
    
    if( xExpectedIdleTime > = configEXPECTED_IDLE_TIME_BEFORE_SLEEP )
    {
        vTaskSuspendAll();
        {
            / * Now the scheduler is suspended, the expected idle
            time can be sampled again, and this time its value can
            be used. * /
            configASSERT( xNextTaskUnblockTime > = xTickCount );
            xExpectedIdleTime = prvGetExpectedIdleTime();
            
            / * Define the following macro to set xExpectedIdleTime to 0
            if the application does not want
            portSUPPRESS_TICKS_AND_SLEEP() to be called. * /
            configPRE_SUPPRESS_TICKS_AND_SLEEP_PROCESSING( xExpectedIdleTime );
            
            if( xExpectedIdleTime > = configEXPECTED_IDLE_TIME_BEFORE_SLEEP )
            {
                traceLOW_POWER_IDLE_BEGIN();
                portSUPPRESS_TICKS_AND_SLEEP( xExpectedIdleTime );
                traceLOW_POWER_IDLE_END();
            }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
}
( void ) xTaskResumeAll();
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
#endif / * configUSE_TICKLESS_IDLE * /
}
}
```
[prvCheckTasksWaitingTermination()]{.underline} -> 检查等待终止的任务,
然后后面删除

这里还有那个钩子函数(就是之前说的那个方便调试的函数) [extern void
vApplicationIdleHook( void );]{.underline}

但是相信到这里你已经初步了解了任务调度了

### RTOS的Delay

在上一部分, 我们说了编写任务时的两个要点:

事件驱动, 当事件发生时才进行任务

延时函数不要用死循环, 不要抓着CPU资源不放手

其中第二点当中, 这个延时函数根据要求我们需要选用不阻塞CPU资源的延时函数

下main我们来介绍一下FreeRTOS本身的函数推荐:
```c
vTaskDelayUntil()
void vTaskDelayUntil( TickType_t  const pxPreviousWakeTime, const
TickType_t xTimeIncrement )
{
    TickType_t xTimeToWake;
    BaseType_t xAlreadyYielded, xShouldDelay = pdFALSE;
    
    configASSERT( pxPreviousWakeTime );
    configASSERT( ( xTimeIncrement > 0U ) );
    configASSERT( uxSchedulerSuspended =  = 0 );
    
    vTaskSuspendAll();
    {
        / * Minor optimisation. The tick count cannot change in this
        block. * /
        const TickType_t xConstTickCount = xTickCount;
        
        / * Generate the tick time at which the task wants to wake. * /
        xTimeToWake = pxPreviousWakeTime + xTimeIncrement;
        
        if( xConstTickCount < pxPreviousWakeTime )
        {
            / * The tick count has overflowed since this function was
            lasted called. In this case the only time we should ever
            actually delay is if the wake time has also overflowed,
            and the wake time is greater than the tick time. When this
            is the case it is as if neither time had overflowed. * /
            if( ( xTimeToWake < pxPreviousWakeTime ) && ( xTimeToWake >
            xConstTickCount ) )
            {
                xShouldDelay = pdTRUE;
            }
        else
        {
            mtCOVERAGE_TEST_MARKER();
        }
}
else
{
    / * The tick time has not overflowed. In this case we will
    delay if either the wake time has overflowed, and / or the
    tick time is less than the wake time. * /
    if( ( xTimeToWake < pxPreviousWakeTime ) || ( xTimeToWake >
    xConstTickCount ) )
    {
        xShouldDelay = pdTRUE;
    }
else
{
    mtCOVERAGE_TEST_MARKER();
}
}

/ * Update the wake time ready for the next call. * /
pxPreviousWakeTime = xTimeToWake;

if( xShouldDelay != pdFALSE )
{
    traceTASK_DELAY_UNTIL( xTimeToWake );
    
    / * prvAddCurrentTaskToDelayedList() needs the block time, not
    the time to wake, so subtract the current tick count. * /
    prvAddCurrentTaskToDelayedList( xTimeToWake - xConstTickCount,
    pdFALSE );
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
xAlreadyYielded = xTaskResumeAll();

/ * Force a reschedule if xTaskResumeAll has not already done so, we
may
have put ourselves to sleep. * /
if( xAlreadyYielded =  = pdFALSE )
{
    portYIELD_WITHIN_API();
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
```
和之前用的
```c
vTaskDelay()
void vTaskDelay( const TickType_t xTicksToDelay )
{
    BaseType_t xAlreadyYielded = pdFALSE;
    
    / * A delay time of zero just forces a reschedule. * /
    if( xTicksToDelay > ( TickType_t ) 0U )
    {
        configASSERT( uxSchedulerSuspended =  = 0 );
        vTaskSuspendAll();
        {
            traceTASK_DELAY();
            
            / * A task that is removed from the event list while the
            scheduler is suspended will not get placed in the ready
            list or removed from the blocked list until the scheduler
            is resumed.
            
            This task cannot be in an event list as it is the currently
            executing task. * /
            prvAddCurrentTaskToDelayedList( xTicksToDelay, pdFALSE );
        }
    xAlreadyYielded = xTaskResumeAll();
}
else
{
    mtCOVERAGE_TEST_MARKER();
}

/ * Force a reschedule if xTaskResumeAll has not already done so, we
may
have put ourselves to sleep. * /
if( xAlreadyYielded =  = pdFALSE )
{
    portYIELD_WITHIN_API();
}
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
```
下main举一个例子来清晰二者区别(用之前的多任务互斥访问OLED工程)

将任务缩减为一个
```c
/ * Private variables
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN Variables * /
/  / static StackType_t g_pucStackOfLightTask[128];
/  / static StaticTask_t g_TCBofLightTask;
/  / static TaskHandle_t xLightTaskHandle;

/  / static StackType_t g_pucStackOfColorTask[128];
/  / static StaticTask_t g_TCBofColorTask;
/  / static TaskHandle_t xColorTaskHandle;

/ * USER CODE END Variables * /
/ * Definitions for defaultTask * /
osThreadId_t defaultTaskHandle;
const osThreadAttr_t defaultTask_attributes = {
    .name = "defaultTask",
    .stack_size = 128 * 4,
    .priority = (osPriority_t) osPriorityNormal,
};

/ * Private function prototypes
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FunctionPrototypes * /
struct TaskPrintInfo {
    uint8_t x;
    uint8_t y;
    char name[16];
};

static struct TaskPrintInfo g_Task1Info = {0, 0, "Task1"};
/  / static struct TaskPrintInfo g_Task2Info = {0, 3, "Task2"};
/  / static struct TaskPrintInfo g_Task3Info = {0, 6, "Task3"};
static int g_LCDCanUse = 1;

void LcdPrintTask(void *params)
{
    struct TaskPrintInfo *pInfo = params;
    uint32_t cnt = 0;
    int len;
    
    /  / TickType_t preTime = xTaskGetTickCount();
    uint64_t T1 = 0, T2 = 0;
    while (1)
    {
        if (g_LCDCanUse)
        {
            g_LCDCanUse = 0;
            len = LCD_PrintString(pInfo - >x, pInfo - >y, pInfo - >name);
            len +  = LCD_PrintString(len, pInfo - >y, ":");
            LCD_PrintSignedVal(len, pInfo - >y, cnt +  + );
            g_LCDCanUse = 1;
            mdelay(cnt & 0x03);
        }
    T1 = system_get_ns();
    vTaskDelay(500);
    /  / vTaskDelayUntill(&preTime, 500);
    T2 = system_get_ns();
    
    LCD_ClearLine(pInfo - >x, (pInfo - >y) + 2);
    LCD_PrintSignedVal(pInfo - >x, (pInfo - >y) + 2, T2 - T1);
}
}

......

void MX_FREERTOS_Init(void) {
    
    ......
    
    xTaskCreate(LcdPrintTask, "task1", 128, &g_Task1Info, osPriorityNormal, NULL);
    /  / xTaskCreate(LcdPrintTask, "task2", 128, &g_Task2Info, osPriorityNormal, NULL);
    /  / xTaskCreate(LcdPrintTask, "task3", 128, &g_Task3Info, osPriorityNormal, NULL);
    
    
    ......
    
}
```
现象就是确实500Tick左右运行一次任务, 但实质上还有些小细节

首先使用[vTaskDelete()]{.underline}函数, 因为现在是单任务运行,
用T2和T1即可获取前后两次运行任务的间隔,
但是由于[mdelay()]{.underline}函数的作用, 现在真正的运行时长是随机的,
可以看下main的图来理解:

![](media/media/image39.jpeg)

这个任务的运行周期是不确定的

下面我们再看同样是单任务运行,
但使用[vTaskDelayUntil()]{.underline}函数来会怎么样

将[void
Lcd_PrintTask]{.underline}更改一下<使用[vTaskDelayUntil要先给个时间preTime]{.underline}>:
```c
void LcdPrintTask(void params)
{
    struct TaskPrintInfo pInfo = params;
    uint32_t cnt = 0;
    int len;
    
    TickType_t preTime = xTaskGetTickCount();
    uint64_t T1 = 0, T2 = 0;
    while (1)
    {
        if (g_LCDCanUse)
        {
            g_LCDCanUse = 0;
            len = LCD_PrintString(pInfo - >x, pInfo - >y, pInfo - >name);
            len +  = LCD_PrintString(len, pInfo - >y, ":");
            LCD_PrintSignedVal(len, pInfo - >y, cnt +  + );
            g_LCDCanUse = 1;
            mdelay(cnt & 0x03);
        }
    T1 = system_get_ns();
    /  / vTaskDelay(500);
    vTaskDelayUntil(&preTime, 500);
    T2 = system_get_ns();
    
    LCD_ClearLine(pInfo - >x, (pInfo - >y) + 2);
    LCD_PrintSignedVal(pInfo - >x, (pInfo - >y) + 2, T2 - T1);
}
}
```
这个时候, 因为我们与的是0x03也就是说, 上面mdelay的最长时间是3ms,
加上本身写OLED程序那么时间大概就是100多毫秒,
结果显示的300多毫秒说明了当前处于一个稳定的运行周期当中, 如下图:

![](media/media/image40.png)

也就是说, 这个500Tick是整个任务执行固定在500Tick当中, 一旦Delay结束,
下一个任务整个周期也是500个Tick

### 同步互斥与通信

#### 同步与互斥的概念

**✅ 一句话区分**

互斥：解决**"不能同时做"**的问题（独占资源，防止冲突）。

同步：解决**"什么时候做"**的问题（协调任务/线程的执行顺序）。

**📌 1. 互斥（Mutual Exclusion）**

目标：保护共享资源（如全局变量、外设、内存缓冲区），防止竞态条件（Race
Condition）。

核心机制：确保同一时间只有一个任务访问资源。

实现方式：

二值信号量（Binary Semaphore）

互斥锁（Mutex）（带优先级继承，防优先级反转）

关中断（Critical Section）

禁止调度器（Scheduler Suspension）

**📌 2. 同步（Synchronization）**

目标：协调多个任务的执行顺序，确保事件按预期时序发生。

核心机制：任务等待某个条件成立后再继续执行。

实现方式：

信号量（Semaphore）（计数信号量常用于同步）

事件标志组（Event Flags）

任务通知（Task Notifications）

消息队列（Message Queue）（间接同步）

#### 例子

互斥（不能两个人同时进去）
厕所里只有一个马桶。

小明进去后把门反锁。

小红想进去，只能在外面等。
小明出来、开门后，小红才能进去。
→ 这就叫互斥：一次只能有一个人用马桶。

同步（小红必须等小明先完事）
还是那间厕所，但小红今天拉肚子，必须等小明彻底用完、把马桶冲干净后她再进去。

小明在里面喊一句"我冲好水了！"

小红听到这句话才进去。
→ 这就叫同步：小红按小明的"信号"行动。

一句话总结

互斥：门反锁，防止两个人挤进去。

同步：等对方喊"好了"再进去。

同步需要信号量才能保证双方处于同一个进度, 互斥为了对资源实现保护和分配.

##### 同步基础例子(有缺陷)

这一集比较简单

想法是使用一个全局变量作为信号, Task1进行求和一个等差数列,
Task2将求和数据和求和时间通过OLED打印出来

创建求和任务之后编写求和代码:
```c
......

/ * Private function prototypes
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FunctionPrototypes * /

static uint64_t g_sum = 0;
static volatile uint8_t g_cal_end = 0;
static uint64_t g_time = 0;

void CalTask(void params)
{
    LCD_ClearLine(0, 0);
    LCD_PrintString(0, 0, "Waiting");
    g_time = system_get_ns();
    
    while (1)
    {
        / * 创建一个计算任务 * /
        uint32_t i = 0;
        g_cal_end = 0;
        
        for (i = 0; i< 10000000; i +  + )
        {
            g_sum +  = i;
        }
    g_cal_end = 1;
    g_time = system_get_ns() - g_time;
    vTaskDelete(NULL);
}
}

void LcdPrintTask(void params)
{
    int len = 0;
    
    vTaskDelay(7000); /  / 解脱CPU一会, 让他专心进行运算
    
    while (g_cal_end =  = 0); /  / 等待运算结束
    
    LCD_ClearLine(0, 0);
    len = LCD_PrintString(0, 0 , "Sum: ");
    LCD_PrintHex(len, 0, g_sum, 1);
    
    LCD_ClearLine(0, 2);
    len = LCD_PrintString(0, 2, "Time(ms): ");
    LCD_PrintSignedVal(len, 2, g_time / 1000000);
    
    vTaskDelete(NULL);
}

......

void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    
    LCD_Init();
    LCD_Clear();
    
    ......
    
    xTaskCreate(CalTask, "task1", 128, NULL, osPriorityNormal, NULL);
    xTaskCreate(LcdPrintTask, "task2", 128, NULL, osPriorityNormal, NULL);
    
    ......
}
```
我进行运算的结果一开始是10s.当时没加上[vTaskDelay()这个]{.underline}函数,
导致两个任务来回切换,
但是当加上这个函数之后可以保证打印任务有7s的blocked,
保证充足时间给计算任务进行, 之后得到5s运算时间

Tips:记得定义时给 [g_cal_end]{.underline} 加上一个volatile或者__IO,
降低编译优化等级, 防止一个计算任务写在RAM当中,
打印任务却读取寄存器当中的初值0

通过上面的例子, 我们可以发现这个全局变量同步的方法不好(两个任务都在运行,
等待的任务占用了CPU资源),
肯定也会有人想为什么不能等运算结束之后在启动打印任务,
这个我们留到后面再说, FreeRTOS其实是提供了很多同步的方法的.

##### 互斥基础例子(有缺陷)

这节也几乎是纯理论, 大家可以纯享赤石了

之前的多任务互斥调用OLED屏幕, 我们使用的是一个全局变量去修改

我想经过前面的任务调度以及堆栈等, 大家可能有点疑问.
下面我将举例子进行讲解:
```c
OLED打印任务源码
void LCD_PrintTask(void pvParameters)
{
    xFreertosTaskPrintInfo_t pTaskInfo = pvParameters;
    uint32_t cnt = 0;
    int len = 0;
    if (g_LcdCnt =  = 0) {
        LCD_Init();
        LCD_Clear();
        g_LcdCnt = 1;
    }
while (1) {
    if (g_LcdCanUse) {
        g_LcdCanUse = 0;
        
        len = LCD_PrintString(pTaskInfo - >x, pTaskInfo - >y,
        pTaskInfo - >name);
        len +  = LCD_PrintString(len, pTaskInfo - >y, ":");
        LCD_PrintSignedVal(len, pTaskInfo - >y, cnt +  + );
        
        g_LcdCanUse = 1;
    }
mdelay(50);
}
}
```
之前是3个任务, 但是我现在以来给2个任务为例子进行讲解, 请看下图:

![](media/media/image41.jpeg)

当1号任务和2号任务在判断时候进度差不多,
那么可能就会造成图上1号任务还没置0就结束这个Tick就切换2号任务,
2号也可以访问OLED了, 那怎么办呢?

似乎我们可以这么改变
```c
void LCD_PrintTask(void pvParameters)
{
    xFreertosTaskPrintInfo_t pTaskInfo = pvParameters;
    uint32_t cnt = 0;
    int len = 0;
    if (g_LcdCnt =  = 0) {
        LCD_Init();
        LCD_Clear();
        g_LcdCnt = 1;
    }
while (1) {
    g_LcdCanUse -  - ;
    if (g_LcdCanUse) {
        g_LcdCanUse = 0;
        
        len = LCD_PrintString(pTaskInfo - >x, pTaskInfo - >y,
        pTaskInfo - >name);
        len +  = LCD_PrintString(len, pTaskInfo - >y, ":");
        LCD_PrintSignedVal(len, pTaskInfo - >y, cnt +  + );
        
        g_LcdCanUse = 1;
    }
mdelay(50);
}
}
```
赶在判断面前进行自减, 这样似乎可以保证OLED屏幕有没有被使用

但回到汇编指令上, 光一个自减同时也是3条指令,
万一在没有运算之前就被打断了呢? 这也有可能出现问题.

那这时候有人会说了, 我直接关闭中断, 再判断, 之后开启中断,
防止切换任务不就好啦?
```c
void LCD_PrintTask(void pvParameters)
{
    xFreertosTaskPrintInfo_t pTaskInfo = pvParameters;
    uint32_t cnt = 0;
    int len = 0;
    if (g_LcdCnt =  = 0) {
        LCD_Init();
        LCD_Clear();
        g_LcdCnt = 1;
    }
while (1) {
    __disable_irq();
    g_LcdCanUse -  - ;
    if (g_LcdCanUse) {
        g_LcdCanUse = 0;
        __enable_irq();
        len = LCD_PrintString(pTaskInfo - >x, pTaskInfo - >y,
        pTaskInfo - >name);
        len +  = LCD_PrintString(len, pTaskInfo - >y, ":");
        LCD_PrintSignedVal(len, pTaskInfo - >y, cnt +  + );
        
        g_LcdCanUse = 1;
    }
mdelay(50);
}
}
```
似乎这样也是可行的, 但是问题还是有的, 请看下图:

![](media/media/image42.jpeg)

似乎效果也不是很好, 所以, 其实更加建议的是下main这种:

![](media/media/image43.jpeg)

一个Running, 另一个Blocked, 之后等一个完成再唤醒另一个这样.

上面失败的原因一个是无法实现原子执行(指令太长), 一个是占用CPU资源

后面将使用信号量和互斥量来解决这些问题

### 通信介绍

同步与互斥需要通过通信来更好地达到程序效果

![](media/media/image44.png)

队列：

里面可以放任意数据，可以放多个数据

任务、ISR都可以放入数据；任务、ISR都可以从中读出数据

事件组：

一个事件用一bit表示，1表示事件发生了，0表示事件没发生

可以用来表示事件、事件的组合发生了，不能传递数据

有广播效果：事件或事件的组合发生了，等待它的多个任务都会被唤醒

信号量：

核心是"计数值"

任务、ISR释放信号量时让计数值加1

任务、ISR获得信号量时，让计数值减1

任务通知：

核心是任务的TCB里的数值

会被覆盖

发通知给谁？必须指定接收任务

只能由接收任务本身获取该通知

互斥量：

数值只有0或1

谁获得互斥量，就必须由谁释放同一个互斥量

队列就像是一堆工人与一堆消费者, 工人负责生产,
消费者一看到商品出来就购买了

事件组如图所示, 多对多, 可以有多个事件

信号量也差不多, 但是当信号量不是0就是1时, 就变成了互斥量

互斥量就像一个厕所, 你上了我不能上

任务通知则是多个服务一个, 多对一

下面将详细介绍这些通信方法

## 队列

要讲队列就不得不提及环形缓冲区

### 环形缓冲区简介

回顾一下之前的全局变量的缺点是什么?多任务访问时可能发生冲突和异常

同样是通信, 看看相对于单个全局变量来说, 这个环形缓冲区则是一个数组,
一般来说定义都是像下面这样:
```c
环形缓冲区实例
#define BufferSize 128

typedef struct {
    uint8_t ReadIndex;
    uint8_t WriteIndex;
    uint8_t Buffer[BufferSize];
}CircleBuffer;
```
由读指针和写指针构成, 这个大的Buffer数组可以相对来说储存更多数据

下main有几张图可以详细解说一下这个环形缓冲区的具体运行逻辑:

![](media/media/image45.jpeg)

通过读写指针的不断变换位置来确定数据写入与读取的位置,
通过取余实现环形读写, 不过大家最后一个空闲位置都会留存不进行写数据,
防止在写完一轮之后还未读取数据的情况时读指针与写指针重合导致难以判断空与非空

不过就这么简单的措施肯定是不行的, 因为在多任务程序当中,
这个并没有阻塞与唤醒的措施,
更加没有互斥措施(读写指针一样会像单个全局变量那样出错即使添加了数据个数)

所以下面将讲解唤醒缓冲区的ProMax版本:

### 队列本质

为了方便理解, 下main举一个例子去讲解队列

### 例子

![](media/media/image46.jpeg)

A和B是工人, 通过传送带传递产品, A负责放产品, B负责拿产品加工

而Task_A和Task_B就很像它俩, 传送带就和队列差不多

A呢有偷懒, B呢也有偷懒, 如下图所示

![](media/media/image47.jpeg)

A放产品, 放不了定个闹钟倒头就睡 -> 结局1: 被闹钟吵醒

> -> 结局2: B拿了产品顺手把A叫醒

B拿产品, 没得拿顶个闹钟倒头就睡 -> 结局1: 被闹钟吵醒

> -> 结局2: A放产品顺手把B叫醒

那放到队列当中具体逻辑如何呢?
```c
| -  -  -  -  -  - SenderList(发送链表)
队列 -  -  - | -  -  -  -  -  - 环形Buffer
| -  -  -  -  -  - ReceiverList(接收链表)
```
根据上面的例子, 我们不难得出,
队列的本质是环形缓冲区Buffer外加两个链表(发送链表和接收链表)

比如说, 当你想放数据的时候, 队列满了,
这个时候只能不放这么快(定个时间下次放顺便把自己放到发送列表里头为下次发送占个位置)

那接收也差不多, Task_B想得到数据, 但是这个时候队列是空的,
和上面类似(定个时间下次再看, 顺便把自己放到接收列表)
```c
示例
/  / 往队列尾部写入数据, 如果没有空间阻塞时间为xTickToWait个Tick
BaseType_t xQueueSend(
QueueHandle_t xQueue,const void *pvItemToQueue,
TickType_t xTicksToWait
);

/  / 从队列中读取数据, 如果数据可读阻塞时间为xTickToWait个Tick
BaseType_t xQueueReceive( QueueHandle_t xQueue,void * const pvBuffer,
TickType_t xTicksToWait );

void Task_B(void *params)
{
    /  / 这是个示例, 不要乱想
    while (1)
    {
        /  / 读队列
        xQueueRecieve(...);
    }
}

void Task_A(void *params)
{
    /  / 这是个示例, 不要乱想
    while (1)
    {
        /  / 写队列
        xQueueSend(...);
    }
}
```
下main来个具体的图就可理解了:

这里假设任务B和任务A处于默认优先级,
以任务B当前时刻在运行并且环形Buffer当中还有一个数据为例子

![](media/media/image48.jpeg)

类比一下上图, 也可以得到A对应的运行模拟图解

#### 队列实操

工程文件:13_mdkFreeRTOS_nwatch_game, 裸机试玩游戏文件:01_nwatch_game,
RTOS试玩游戏文件(未完善版本):13_mdkFreeRTOS_nwatch_game_deleted

这个工程旨在于使用RTOS实现一个挡球板小游戏,
当小球触碰到下界会减少你的生命值,
你的目标就是通过使用红外遥控器左右移动底下挡球板, 让小球安全反弹

通过试玩游戏之后, 我们可以发现这个游戏有两个任务:移动挡球板和游戏任务.
代码如下:
```c
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    LCD_Init();
    LCD_Clear();
    
    
    IRReceiver_Init();
    LCD_PrintString(0, 0, "Starting");
    
    ......
    
    /  / 创建游戏任务
    xTaskCreate(game1_task, "GameTask", 128, NULL, osPriorityNormal,
    NULL);
    
    ......
    
}
```
```c
/ *
* Project: N|Watch
* Author: Zak Kemble, contact@zakkemble.co.uk
* Copyright: (C) 2013 by Zak Kemble
* License: GNU GPL v3 (see License.txt)
* Web: http: /  / blog.zakkemble.co.uk / diy - digital - wristwatch /
* /
#include <stdlib.h>
#include <stdio.h>

#include "cmsis_os.h"
#include "FreeRTOS.h" /  / ARM.FreeRTOS::RTOS:Core
#include "task.h" /  / ARM.FreeRTOS::RTOS:Core
#include "event_groups.h" /  / ARM.FreeRTOS::RTOS:Event Groups
#include "semphr.h" /  / ARM.FreeRTOS::RTOS:Core

#include "draw.h"
#include "resources.h"

#include "driver_lcd.h"

#define NOINVERT false
#define INVERT true

#define sprintf_P sprintf
#define PSTR(a) a

#define PLATFORM_WIDTH 12
#define PLATFORM_HEIGHT 4
#define UPT_MOVE_NONE 0
#define UPT_MOVE_RIGHT 1
#define UPT_MOVE_LEFT 2
#define BLOCK_COLS 32
#define BLOCK_ROWS 5
#define BLOCK_COUNT (BLOCK_COLS * BLOCK_ROWS)

typedef struct{
    float x;
    float y;
    float velX;
    float velY;
}s_ball;

static const byte block[] = {
    0x07,0x07,0x07,
};

static const byte platform[] = {
    0x60,0x70,0x50,0x10,0x30,0xF0,0xF0,0x30,0x10,0x50,0x70,0x60,
};

static const byte ballImg[] = {
    0x03,0x03,
};

static const byte clearImg[] = {
    0,0,0,0,0,0,0,0,0,0,0,0,
};

static bool btnExit(void);
static bool btnRight(void);
static bool btnLeft(void);
void game1_draw(void);

static byte uptMove;
static s_ball ball;
static bool* blocks;
static byte lives, lives_origin;
static uint score;
static byte platformX;

static uint32_t g_xres, g_yres, g_bpp;
static uint8_t *g_framebuffer;


/ * 挡球板任务 * /
static void platform_task(void *params)
{
    byte platformXtmp = platformX;
    uint8_t dev, data, last_data;
    
    /  / Draw platform
    draw_bitmap(platformXtmp, g_yres - 8, platform, 12, 8, NOINVERT, 0);
    draw_flushArea(platformXtmp, g_yres - 8, 12, 8);
    
    while (1)
    {
        / * 读取红外遥控器 * /
        if (0 =  = IRReceiver_Read(&dev, &data))
        {
            if (data =  = 0x00)
            {
                data = last_data;
            }
        
        if (data =  = 0xe0) / * Left * /
        {
            btnLeft();
        }
    
    if (data =  = 0x90) / * Right * /
    {
        btnRight();
    }
last_data = data;


/  / Hide platform
draw_bitmap(platformXtmp, g_yres - 8, clearImg, 12, 8, NOINVERT, 0);
draw_flushArea(platformXtmp, g_yres - 8, 12, 8);

/  / Move platform
if(uptMove =  = UPT_MOVE_RIGHT)
platformXtmp +  = 3;
else if(uptMove =  = UPT_MOVE_LEFT)
platformXtmp -  = 3;
uptMove = UPT_MOVE_NONE;

/  / Make sure platform stays on screen
if(platformXtmp > 250)
platformXtmp = 0;
else if(platformXtmp > g_xres - PLATFORM_WIDTH)
platformXtmp = g_xres - PLATFORM_WIDTH;

/  / Draw platform
draw_bitmap(platformXtmp, g_yres - 8, platform, 12, 8, NOINVERT, 0);
draw_flushArea(platformXtmp, g_yres - 8, 12, 8);

platformX = platformXtmp;

}
}
}

void game1_task(void *params)
{
    uint8_t dev, data, last_data;
    
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    
    uptMove = UPT_MOVE_NONE;
    
    ball.x = g_xres / 2;
    ball.y = g_yres - 10;
    
    ball.velX =  - 0.5;
    ball.velY =  - 0.6;
    /  / ball.velX =  - 1;
    /  / ball.velY =  - 1.1;
    
    blocks = pvPortMalloc(BLOCK_COUNT);
    memset(blocks, 0, BLOCK_COUNT);
    
    lives = lives_origin = 3;
    score = 0;
    platformX = (g_xres / 2) - (PLATFORM_WIDTH / 2);
    
    xTaskCreate(platform_task, "platform_task", 128, NULL,
    osPriorityNormal, NULL);
    
    while (1)
    {
        game1_draw();
        /  / draw_end();
        vTaskDelay(50);
    }
}

static bool btnExit()
{
    
    vPortFree(blocks);
    if(lives =  = 255)
    {
        /  / game1_start();
    }
else
{
    /  / pwrmgr_setState(PWR_ACTIVE_DISPLAY, PWR_STATE_NONE);
    /  / animation_start(display_load, ANIM_MOVE_OFF);
    vTaskDelete(NULL);
}
return true;
}

static bool btnRight()
{
    uptMove = UPT_MOVE_RIGHT;
    return false;
}

static bool btnLeft()
{
    uptMove = UPT_MOVE_LEFT;
    return false;
}

void game1_draw()
{
    bool gameEnded = ((score > = BLOCK_COUNT) || (lives =  = 255));
    
    byte platformXtmp = platformX;
    
    static bool first = 1;
    
    /  / Move ball
    /  / hide ball
    draw_bitmap(ball.x, ball.y, clearImg, 2, 2, NOINVERT, 0);
    draw_flushArea(ball.x, ball.y, 2, 8);
    
    /  / Draw platform
    /  / draw_bitmap(platformX, g_yres - 8, platform, 12, 8, NOINVERT, 0);
    /  / draw_flushArea(platformX, g_yres - 8, 12, 8);
    
    if(!gameEnded)
    {
        ball.x +  = ball.velX;
        ball.y +  = ball.velY;
    }

bool blockCollide = false;
const float ballX = ball.x;
const byte ballY = ball.y;

/  / Block collision
byte idx = 0;
LOOP(BLOCK_COLS, x)
{
    LOOP(BLOCK_ROWS, y)
    {
        if(!blocks[idx] && ballX > = x * 4 && ballX < (x * 4) + 4 && ballY
        > = (y * 4) + 8 && ballY < (y * 4) + 8 + 4)
        {
            /  / buzzer_buzz(100, TONE_2KHZ, VOL_UI, PRIO_UI, NULL);
            /  / led_flash(LED_GREEN, 50, 255); /  / 100ask todo
            blocks[idx] = true;
            
            /  / hide block
            draw_bitmap(x * 4, (y * 4) + 8, clearImg, 3, 8, NOINVERT, 0);
            draw_flushArea(x * 4, (y * 4) + 8, 3, 8);
            blockCollide = true;
            score +  + ;
        }
    idx +  + ;
}
}


/  / Side wall collision
if(ballX > g_xres - 2)
{
    if(ballX > 240)
    ball.x = 0;
    else
    ball.x = g_xres - 2;
    ball.velX =  - ball.velX;
}
if(ballX < 0)
{
    ball.x = 0;
    ball.velX =  - ball.velX;
}

/  / Platform collision
bool platformCollision = false;
if(!gameEnded && ballY > = g_yres - PLATFORM_HEIGHT - 2 && ballY < 240
&& ballX > = platformX && ballX < = platformX + PLATFORM_WIDTH)
{
    platformCollision = true;
    /  / buzzer_buzz(200, TONE_5KHZ, VOL_UI, PRIO_UI, NULL); /  / 100ask todo
    ball.y = g_yres - PLATFORM_HEIGHT - 2;
    if(ball.velY > 0)
    ball.velY =  - ball.velY;
    ball.velX = ((float)rand() / (RAND_MAX / 2)) - 1; /  /  - 1.0 to 1.0
}

/  / Top / bottom wall collision
if(!gameEnded && !platformCollision && (ballY > g_yres - 2 ||
blockCollide))
{
    if(ballY > 240)
    {
        /  / buzzer_buzz(200, TONE_2_5KHZ, VOL_UI, PRIO_UI, NULL); /  / 100ask
        todo
        ball.y = 0;
    }
else if(!blockCollide)
{
    /  / buzzer_buzz(200, TONE_2KHZ, VOL_UI, PRIO_UI, NULL); /  / 100ask todo
    ball.y = g_yres - 1;
    lives -  - ;
}
ball.velY *=  - 1;
}

/  / Draw ball
draw_bitmap(ball.x, ball.y, ballImg, 2, 2, NOINVERT, 0);
draw_flushArea(ball.x, ball.y, 2, 8);

/  / Draw platform
/  / draw_bitmap(platformX, g_yres - 8, platform, 12, 8, NOINVERT, 0);
/  / draw_flushArea(platformX, g_yres - 8, 12, 8);

if (first)
{
    first = 0;
    
    /  / Draw blocks
    idx = 0;
    LOOP(BLOCK_COLS, x)
    {
        LOOP(BLOCK_ROWS, y)
        {
            if(!blocks[idx])
            {
                draw_bitmap(x * 4, (y * 4) + 8, block, 3, 8, NOINVERT, 0);
                draw_flushArea(x * 4, (y * 4) + 8, 3, 8);
            }
        idx +  + ;
    }
}

}

/  / Draw score
char buff[6];
sprintf_P(buff, PSTR("%u"), score);
draw_string(buff, false, 0, 0);

/  / Draw lives
if(lives != 255)
{
    LOOP(lives_origin, i)
    {
        if (i < lives)
        draw_bitmap((g_xres - (3*8)) + (8*i), 1, livesImg, 7, 8, NOINVERT,
        0);
        else
        draw_bitmap((g_xres - (3*8)) + (8*i), 1, clearImg, 7, 8, NOINVERT,
        0);
        draw_flushArea((g_xres - (3*8)) + (8*i), 1, 7, 8);
    }
}

/  / Got all blocks
if(score > = BLOCK_COUNT)
draw_string_P(PSTR(STR_WIN), false, 50, 32);

/  / No lives left (255 because overflow)
if(lives =  = 255)
draw_string_P(PSTR(STR_GAMEOVER), false, 34, 32);

}
```
其中我们不难发现, 现在的遥控器按照之前的读取函数是环形Buffer读取,
我们将其改造为队列读取

队列当中Sender是红外接收的部分(中断任务当中写队列),
Receiver是对红外接收数据进行读取的函数

**创建队列**

创建队列的函数:

动态分配内存：xQueueCreate，队列的内存在函数内部动态分配

静态分配内存：xQueueCreateStatic，队列的内存要事先分配好

动态:

![](media/media/image53.jpeg)

你可以采用轮询方式, 来更好的管理, 代码大概如下:
```c
{
    while (1)
    {
        xQueueRecieve(红外接收)
        xQueueRecieve(编码器)
        xQueueRecieve(MPU6050)
    }
}
```
![](media/media/image53.jpeg)

但是, 这些小喽啰都介绍完了, 现在该真正的主角登场了:

![](media/media/image54.jpeg)

我们可以先分别创建队列, 由一个队列集来读取

大概如下图:

![](media/media/image55.jpeg)

当一个队列被写入之后, 通过程序在控制其写入队列集,
队列集当中会存留对应队列的句柄, 当Task读取的时候就会得到句柄,
只要不是NULL, 那就直接读这个句柄的队列

##### 改进挡球板游戏

工程文件:15_queueset_game

先将之前的工程的代码整理一下, 创建每个控制游戏的外设自己的结构体和队列
```c
/ * Copyright (s) 2019 深圳百问网科技有限公司
* All rights reserved
*
* 文件名称：driver_ir_receiver.c
* 摘要：
*
* 修改历史 版本号 Author 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023.08.04 v01 百问科技 创建文件
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* /

#include "driver_ir_receiver.h"
#include "driver_lcd.h"
#include "driver_timer.h"
#include "stm32f1xx_hal.h"
#include "tim.h"
#include "FreeRTOS.h"
#include "queue.h"
#include "typedefs.h"

/ * 环形缓冲区: 用来保存解析出来的按键,可以防止丢失 * /
#define BUF_LEN 128
static unsigned char g_KeysBuf[BUF_LEN];
static int g_KeysBuf_R, g_KeysBuf_W;

static uint64_t g_IRReceiverIRQ_Timers[68];
static int g_IRReceiverIRQ_Cnt = 0;
/  / static uint32_t g_last_val;

/  / extern QueueHandle_t g_xQueuePlatform; / * 挡球板队列 * /
/ 定义自己的队列 /
static QueueHandle_t g_xQueueIR; /  / 红外队列

......

/
* 函数名称： IRReceiver_IRQTimes_Parse
* 功能描述： 解析中断回调函数里记录的时间序列,得到的device和key放入环形缓冲区
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 0 - 成功, ( - 1) - 失败
* 修改日期： 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 04 V1.0 韦东山 创建
* /
static int IRReceiver_IRQTimes_Parse(void)
{
    uint64_t time;
    int i;
    int m, n;
    unsigned char datas[4];
    unsigned char data = 0;
    int bits = 0;
    int byte = 0;
    input_data idata;
    
    / * 1. 判断前导码 : 9ms的低脉冲, 4.5ms高脉冲 * /
    time = g_IRReceiverIRQ_Timers[1] - g_IRReceiverIRQ_Timers[0];
    if (time < 8000000 || time > 10000000)
    {
        return - 1;
    }

time = g_IRReceiverIRQ_Timers[2] - g_IRReceiverIRQ_Timers[1];
if (time < 3500000 || time > 55000000)
{
    return - 1;
}

/ * 2. 解析数据 * /
for (i = 0; i < 32; i +  + )
{
    m = 3 + i*2;
    n = m + 1;
    time = g_IRReceiverIRQ_Timers[n] - g_IRReceiverIRQ_Timers[m];
    data << = 1;
    bits +  + ;
    if (time > 1000000)
    {
        / * 得到了数据1 * /
        data | = 1;
    }

if (bits =  = 8)
{
    datas[byte] = data;
    byte +  + ;
    data = 0;
    bits = 0;
}
}

/ * 判断数据正误 * /
datas[1] = ~datas[1];
datas[3] = ~datas[3];

if ((datas[0] != datas[1]) || (datas[2] != datas[3]))
{
    g_IRReceiverIRQ_Cnt = 0;
    return - 1;
}

/  / PutKeyToBuf(datas[0]);
/  / PutKeyToBuf(datas[2]);

/ 写数据进队列 /
idata.dev = datas[0];

/  / if (datas[2] =  = 0xe0)
/  / {
    /  / idata.val = UPT_MOVE_LEFT;
    /  / }
/  / else if (datas[2] =  = 0x90)
/  / {
    /  / idata.val = UPT_MOVE_RIGHT;
    /  / }
/  / else
/  / {
    /  / idata.val = UPT_MOVE_NONE;
    /  / }
/ 之前的工程是将数据识别之后对应控制游戏的按键从而放入队列 /

idata.val = datas[2];
/ 上一次的数值无需记录 /
/  / g_last_val = idata.val;

xQueueSendFromISR(g_xQueueIR, &idata, NULL);

return 0;
}

......

/
* 函数名称： IRReceiver_IRQ_Callback
* 功能描述： 红外接收器的中断回调函数,记录中断时刻
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 无
* 修改日期： 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 04 V1.0 韦东山 创建
* /
void IRReceiver_IRQ_Callback(void)
{
    uint64_t time;
    static uint64_t pre_time = 0;
    ir_data idata; /  / input_data
    
    / * 1. 记录中断发生的时刻 * /
    time = system_get_ns();
    
    / * 一次按键的最长数据 = 引导码 + 32个数据"1" = 9 + 4.5 + 2.25*32 = 85.5ms
    * 如果当前中断的时刻, 举例上次中断的时刻超过这个时间, 以前的数据就抛弃
    * /
    if (time - pre_time > 100000000)
    {
        g_IRReceiverIRQ_Cnt = 0;
    }
pre_time = time;

g_IRReceiverIRQ_Timers[g_IRReceiverIRQ_Cnt] = time;

/ * 2. 累计中断次数 * /
g_IRReceiverIRQ_Cnt +  + ;

/ * 3. 次数达标后, 解析数据, 放入buffer * /
if (g_IRReceiverIRQ_Cnt =  = 4)
{
    / * 是否重复码 * /
    if (isRepeatedKey())
    {
        / * device: 0, val: 0, 表示重复码 * /
        /  / PutKeyToBuf(0);
        /  / PutKeyToBuf(0);
        
        / 写数据进队列 /
        idata.dev = 0;
        idata.val = 0; /  / g_last_val, 这更改之后直接上报重复码
        xQueueSendFromISR(g_xQueueIR, &idata, NULL);
        
        g_IRReceiverIRQ_Cnt = 0;
    }
}
if (g_IRReceiverIRQ_Cnt =  = 68)
{
    IRReceiver_IRQTimes_Parse();
    g_IRReceiverIRQ_Cnt = 0;
}
}


/
* 函数名称： IRReceiver_Init
* 功能描述： 红外接收器的初始化函数
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 无
* 修改日期： 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 04 V1.0 韦东山 创建
* /
void IRReceiver_Init(void)
{
    / * PA10在MX_GPIO_Init()中已经被配置为双边沿触发, 并使能了中断 * /
    #if 0
    / *Configure GPIO pin : PB10 * /
    GPIO_InitStruct.Pin = GPIO_PIN_10;
    GPIO_InitStruct.Mode = GPIO_MODE_EVT_RISING_FALLING;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
    #endif
    g_xQueueIR = xQueueCreate(10, sizeof(ir_data));
}

......
```
对应的, 在数据传输当中因为现在红外接收模块已经是直接负责接收的键值了,
我们需要将键值直接赋值给val

下面是更改之后的.h文件
```c
演示
C +  +
driver_ir_receiver.h
......

#define IR_KEY_POWER 0xa2
#define IR_KEY_MENU 0xe2
#define IR_KEY_TEST 0x22
#define IR_KEY_ADD 0x02
#define IR_KEY_RETURN 0xc2
#define IR_KEY_LEFT 0xe0
#define IR_KEY_PLAY 0xa8
#define IR_KEY_RIGHT 0x90
#define IR_KEY_0 0x68
#define IR_KEY_DEC 0x98
#define IR_KEY_C 0xb0
#define IR_KEY_1 0x30
#define IR_KEY_2 0x18
#define IR_KEY_3 0x7a
#define IR_KEY_4 0x10
#define IR_KEY_5 0x38
#define IR_KEY_6 0x5a
#define IR_KEY_7 0x42
#define IR_KEY_8 0x4a
#define IR_KEY_9 0x52
#define IR_KEY_REPEAT 0x00

typedef struct {
    uint32_t dev;
    uint32_t val;
}ir_data;

......
```
紧接着, 用类似的方法更改旋转编码器的变量, 将旋转编码器的结构体等更改
```c
/ *
* Project: N|Watch
* Author: Zak Kemble, contact@zakkemble.co.uk
* Copyright: (C) 2013 by Zak Kemble
* License: GNU GPL v3 (see License.txt)
* Web: http: /  / blog.zakkemble.co.uk / diy - digital - wristwatch /
* /

#ifndef TYPEDEFS_H_
#define TYPEDEFS_H_

#include <stdbool.h>
#include <stdint.h>

typedef uint8_t byte;
typedef uint16_t uint;
typedef uint32_t ulong;

#define PROGMEM

#define UPT_MOVE_NONE 0
#define UPT_MOVE_RIGHT 1
#define UPT_MOVE_LEFT 2

typedef struct {
    uint32_t dev;
    uint32_t val;
}input_data;

/  / typedef struct {
    /  / int32_t cnt;
    /  / int32_t speed;
    /  / }rotary_data;

/  / Quick and easy macro for a for loop
#define LOOP(count, var) for(byte var = 0;var<count;var +  + )



#endif / * TYPEDEFS_H_ * /
```
```c
#ifndef _DRIVER_ROTARY_ENCODER_H
#define _DRIVER_ROTARY_ENCODER_H

#include <stdint.h>

typedef struct {
    int32_t cnt;
    int32_t speed;
}rotary_data;

......

#endif / * _DRIVER_ROTARY_ENCODER_H * /
```
```c
.....

static QueueHandle_t g_xQueueRotary; / * 旋转编码器队列 * /
static uint8_t g_ucQueueRotaryBuf[10  sizeof(rotary_data)];
static StaticQueue_t g_xQueueRotaryStaticStruct;

......

/
函数名称： RotaryEncoder_Init
* 功能描述： 旋转编码器的初始化函数
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 无
* 修改日期： 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 05 V1.0 韦东山 创建
* /
void RotaryEncoder_Init(void)
{
    / * PB0,PB1在MX_GPIO_Init中被配置为输入引脚 * /
    / * PB12在MX_GPIO_Init中被配置为中断引脚,上升沿触发 * /
    g_xQueueRotary = xQueueCreateStatic(10, sizeof(rotary_data), g_ucQueueRotaryBuf, &g_xQueueRotaryStaticStruct);
}

......
```
```c
......

/ **
* @brief FreeRTOS initialization
* @param None
* @retval None
* /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    LCD_Init();
    LCD_Clear();
    
    
    IRReceiver_Init();
    RotaryEncoder_Init();
    LCD_PrintString(0, 0, "Starting");
    
    / * USER CODE END Init * /
    
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    /  / defaultTaskHandle = osThreadNew(StartDefaultTask, NULL,
    &defaultTask_attributes);
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    
    /  / extern void PlayMusic(void *params);
    /  / xTaskCreate(PlayMusic, "MusicTask", 128, NULL, osPriorityNormal,
    NULL);
    
    /  / 创建游戏任务
    xTaskCreate(game1_task, "GameTask", 128, NULL, osPriorityNormal,
    NULL);
    
    / * USER CODE END RTOS_THREADS * /
    
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
    
}

......
```
下main创建队列集

用到的函数
```c
QueueSetHandle_t xQueueCreateSet( const UBaseType_t uxEventQueueLength
)
```
![](media/media/image58.png)

现在就可以创建任务了
```c
void game1_task(void *params)
{
    ......
    
    / * 创建队列, 创建队列集, 创建输入任务 * /
    g_xQueuePlatform = xQueueCreate(10, sizeof(input_data));
    /  / g_xQueueRotary = xQueueCreateStatic(10, sizeof(rotary_data),
    g_ucQueueRotaryBuf, &g_xQueueRotaryStaticStruct);
    g_xQueueSetInput = xQueueCreateSet(IR_QUEUE_LEN + ROTARY_QUEUE_LEN);
    
    g_xQueueIR = GetQueueIR();
    g_xQueueRotary = GetQueueRotary();
    
    xQueueAddToSet(g_xQueueIR, g_xQueueSetInput);
    xQueueAddToSet(g_xQueueRotary, g_xQueueSetInput);
    
    xTaskCreate(InputTask, "InputTask", 128, NULL, osPriorityNormal,
    NULL);
    
    ......
}
```
创建任务之后我们大致捋一捋思路:
```c
static void InputTask(void *params)
{
    
    while (1)
    {
        /  / 读队列集, 得到有数据队列句柄
        
        
        /  / 读队列句柄, 得到数据
        
        
        /  / 处理数据
        
        
        /  / 写挡球板队列
        
        
    }
}
```
如果要读队列集, 我们需要用到的函数是
```c
QueueSetMemberHandle_t xQueueSelectFromSet( QueueSetHandle_t
xQueueSet,

TickType_t const xTicksToWait );
```
![](media/media/image60.png)

之后就可以编译成功, 但是烧写之后可能会出现挡球板显示失败的问题,
需要注释默认任务

如果注释默认任务之后还是失败, 那就需要调整CUBEMX当中的堆栈配置了

将堆大小由3072改为8192

![](media/media/image61.png)

记得生成后注释默认任务

现象和之前一致

##### 挡球板游戏增加MPU

工程文件:16_queueset_game_mpu6050

先对MPU6050进行测试, 工程中的MPU6050驱动文件有一些缺漏,
下面提供一下它的驱动函数
```c
/  / SPDX - License - Identifier: GPL - 3.0 - only
/ *
* Copyright (c) 2008 - 2023 100askTeam : Dongshan WEI <weidongshan@qq.com>
* Discourse: https: /  / forums.100ask.net
* /

/ * Copyright (C) 2008 - 2023 深圳百问网科技有限公司
* All rights reserved
*
* 免责声明: 百问网编写的文档, 仅供学员学习使用, 可以转发或引用(请保留作者信息),禁止用于商业用途！
* 免责声明: 百问网编写的程序, 可以用于商业用途, 但百问网不承担任何后果！
*
* 本程序遵循GPL V3协议, 请遵循协议
* 百问网学习平台 : https: /  / www.100ask.net
* 百问网交流社区 : https: /  / forums.100ask.net
* 百问网官方B站 : https: /  / space.bilibili.com / 275908810
* 本程序所用开发板 : DShanMCU - F103
* 百问网官方淘宝 : https: /  / 100ask.taobao.com
* 联系我们(E - mail): weidongshan@qq.com
*
* 版权所有，盗版必究。
*
* 修改历史 版本号 作者 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023.08.04 v01 百问科技 创建文件
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* /




#include "stm32f1xx_hal.h"
#include "driver_mpu6050.h"
#include "driver_lcd.h"
#include "driver_timer.h"
#include <math.h>

/  /
/  / 定义MPU6050内部地址
/  /
#define MPU6050_SMPLRT_DIV 0x19 /  / 陀螺仪采样率，典型值：0x07(125Hz)
#define MPU6050_CONFIG 0x1A /  / 低通滤波频率，典型值：0x06(5Hz)
#define MPU6050_GYRO_CONFIG 0x1B /  / 陀螺仪自检及测量范围，典型值：0x18(不自检，2000deg / s)
#define MPU6050_ACCEL_CONFIG 0x1C /  / 加速计自检、测量范围及高通滤波频率，典型值：0x01(不自检，2G，5Hz)

#define MPU6050_ACCEL_XOUT_H 0x3B
#define MPU6050_ACCEL_XOUT_L 0x3C
#define MPU6050_ACCEL_YOUT_H 0x3D
#define MPU6050_ACCEL_YOUT_L 0x3E
#define MPU6050_ACCEL_ZOUT_H 0x3F
#define MPU6050_ACCEL_ZOUT_L 0x40
#define MPU6050_TEMP_OUT_H 0x41
#define MPU6050_TEMP_OUT_L 0x42
#define MPU6050_GYRO_XOUT_H 0x43
#define MPU6050_GYRO_XOUT_L 0x44
#define MPU6050_GYRO_YOUT_H 0x45
#define MPU6050_GYRO_YOUT_L 0x46
#define MPU6050_GYRO_ZOUT_H 0x47
#define MPU6050_GYRO_ZOUT_L 0x48

#define MPU6050_PWR_MGMT_1 0x6B /  / 电源管理，典型值：0x00(正常启用)
#define MPU6050_PWR_MGMT_2 0x6C
#define MPU6050_WHO_AM_I 0x75 /  / IIC地址寄存器(默认数值0x68，只读)

#define MPU6050_I2C_ADDR 0xD0
#define MPU6050_TIMEOUT 500

/ * 传感器数据修正值（消除芯片固定误差，根据硬件进行调整） * /
#define MPU6050_X_ACCEL_OFFSET ( - 64)
#define MPU6050_Y_ACCEL_OFFSET ( - 30)
#define MPU6050_Z_ACCEL_OFFSET (14400)
#define MPU6050_X_GYRO_OFFSET (40)
#define MPU6050_Y_GYRO_OFFSET ( - 7)
#define MPU6050_Z_GYRO_OFFSET ( - 14)

extern I2C_HandleTypeDef hi2c1;
static I2C_HandleTypeDef *g_pHI2C_MPU6050 = &hi2c1;

/
* 函数名称： MPU6050_WriteRegister
* 功能描述： 写MPU6050寄存器
* 输入参数： reg - 寄存器地址, data - 要写入的数据
* 输出参数： 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 03 V1.0 韦东山 创建
* /
static int MPU6050_WriteRegister(uint8_t reg, uint8_t data)
{
    uint8_t tmpbuf[2];
    
    tmpbuf[0] = reg;
    tmpbuf[1] = data;
    
    return HAL_I2C_Master_Transmit(g_pHI2C_MPU6050, MPU6050_I2C_ADDR, tmpbuf, 2, MPU6050_TIMEOUT);
}

/
* 函数名称： MPU6050_ReadRegister
* 功能描述： 读MPU6050寄存器
* 输入参数： reg - 寄存器地址
* 输出参数： pdata - 用来保存读出的数据
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 03 V1.0 韦东山 创建
* /
int MPU6050_ReadRegister(uint8_t reg, uint8_t *pdata)
{
    return HAL_I2C_Mem_Read(g_pHI2C_MPU6050, MPU6050_I2C_ADDR, reg, 1, pdata, 1, MPU6050_TIMEOUT);
}

/
* 函数名称： MPU6050_Init
* 功能描述： MPU6050初始化函数,
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 03 V1.0 韦东山 创建
* /
int MPU6050_Init(void)
{
    MPU6050_WriteRegister(MPU6050_PWR_MGMT_1, 0x00); /  / 解除休眠状态
    MPU6050_WriteRegister(MPU6050_PWR_MGMT_2, 0x00);
    MPU6050_WriteRegister(MPU6050_SMPLRT_DIV, 0x09);
    MPU6050_WriteRegister(MPU6050_CONFIG, 0x06);
    MPU6050_WriteRegister(MPU6050_GYRO_CONFIG, 0x18);
    MPU6050_WriteRegister(MPU6050_ACCEL_CONFIG, 0x18);
    
    return 0;
}

/
* 函数名称： MPU6050_GetID
* 功能描述： 读取MPU6050 ID
* 输入参数： 无
* 输出参数： 无
* 返 回 值： - 1 - 失败, 其他值 - ID
* 修改日期 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 03 V1.0 韦东山 创建
* /
int MPU6050_GetID(void)
{
    uint8_t id;
    if(0 =  = MPU6050_ReadRegister(MPU6050_WHO_AM_I, &id))
    return id;
    else
    return - 1;
}



/
* 函数名称： MPU6050_ReadData
* 功能描述： 读取MPU6050数据
* 输入参数： 无
* 输出参数： pAccX / pAccY / pAccZ - 用来保存X / Y / Z轴的加速度
* pGyroX / pGyroY / pGyroZ - 用来保存X / Y / Z轴的角速度
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 03 V1.0 韦东山 创建
* /
int MPU6050_ReadData(int16_t *pAccX, int16_t *pAccY, int16_t *pAccZ, int16_t *pGyroX, int16_t *pGyroY, int16_t *pGyroZ)
{
    uint8_t datal, datah;
    int err = 0;
    
    if(pAccX)
    {
        err | = MPU6050_ReadRegister(MPU6050_ACCEL_XOUT_H, &datah);
        err | = MPU6050_ReadRegister(MPU6050_ACCEL_XOUT_L, &datal);
        *pAccX = (datah << 8) | datal;
    }

if(pAccY)
{
    err | = MPU6050_ReadRegister(MPU6050_ACCEL_YOUT_H, &datah);
    err | = MPU6050_ReadRegister(MPU6050_ACCEL_YOUT_L, &datal);
    *pAccY = (datah << 8) | datal;
}

if(pAccZ)
{
    err | = MPU6050_ReadRegister(MPU6050_ACCEL_ZOUT_H, &datah);
    err | = MPU6050_ReadRegister(MPU6050_ACCEL_ZOUT_L, &datal);
    *pAccZ = (datah << 8) | datal;
}


if(pGyroX)
{
    err | = MPU6050_ReadRegister(MPU6050_GYRO_XOUT_H, &datah);
    err | = MPU6050_ReadRegister(MPU6050_GYRO_XOUT_L, &datal);
    *pGyroX = (datah << 8) | datal;
}


if(pGyroY)
{
    err | = MPU6050_ReadRegister(MPU6050_GYRO_YOUT_H, &datah);
    err | = MPU6050_ReadRegister(MPU6050_GYRO_YOUT_L, &datal);
    *pGyroY = (datah << 8) | datal;
}

if(pGyroZ)
{
    err | = MPU6050_ReadRegister(MPU6050_GYRO_ZOUT_H, &datah);
    err | = MPU6050_ReadRegister(MPU6050_GYRO_ZOUT_L, &datal);
    *pGyroZ = (datah << 8) | datal;
}

return err;
}

/
* 函数名称： MPU6050_ParseData
* 功能描述： 解析MPU6050数据
* 输入参数： AccX / AccY / AccZ / GyroX / GyroY / GyroZ
* X / Y / Z轴的加速度,X / Y / Z轴的角速度
* 输出参数： result - 用来保存计算出的结果,目前仅支持X方向的角度
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 09 / 05 V1.0 韦东山 创建
* /
void MPU6050_ParseData(int16_t AccX, int16_t AccY, int16_t AccZ, int16_t GyroX, int16_t GyroY, int16_t GyroZ, mpu6050_data *result)
{
    if (result)
    {
        result - >angle_x = (int32_t)(acos((double)((double)(AccX + MPU6050_X_ACCEL_OFFSET) / 16384.0)) * 57.29577);
    }
}

/
* 函数名称： MPU6050_Test
* 功能描述： MPU6050测试程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 03 V1.0 韦东山 创建
* /
void MPU6050_Test(void)
{
    int id;
    int16_t AccX, AccY, AccZ, GyroX, GyroY, GyroZ;
    int len;
    mpu6050_data result;
    
    MPU6050_Init();
    
    id = MPU6050_GetID();
    LCD_PrintString(0, 0, "MPU6050 ID:");
    LCD_PrintHex(0, 2, id, 1);
    
    mdelay(1000);
    LCD_Clear();
    
    while (1)
    {
        MPU6050_ReadData(&AccX, &AccY, &AccZ, &GyroX, &GyroY, &GyroZ);
        MPU6050_ParseData(AccX, AccY, AccZ, GyroX, GyroY, GyroZ, &result);
        
        LCD_PrintString(0, 0, "X: ");
        LCD_PrintSignedVal(3, 0, acos((double)((double)(AccX + MPU6050_X_ACCEL_OFFSET) / 16384.0)) * 57.29577);
        LCD_PrintSignedVal(10, 0, GyroX + MPU6050_X_GYRO_OFFSET);
        
        LCD_PrintString(0, 2, "Y: ");
        LCD_PrintSignedVal(3, 2, acos((double)((double)(AccY + MPU6050_Y_ACCEL_OFFSET) / 16384.0)) * 57.29577);
        LCD_PrintSignedVal(10, 2, GyroY + MPU6050_Y_GYRO_OFFSET);
        
        LCD_PrintString(0, 4, "Z: ");
        LCD_PrintSignedVal(3, 4, acos((double)((double)(AccZ + MPU6050_Z_ACCEL_OFFSET) / 16384.0)) * 57.29577);
        LCD_PrintSignedVal(10, 4, GyroZ + MPU6050_Z_GYRO_OFFSET);
        
        len = LCD_PrintString(0, 6, "AngleX: ");
        LCD_PrintSignedVal(len, 6, result.angle_x);
        
        mdelay(100);
        LCD_Clear();
    }
}
```
```c
/ * Copyright (s) 2023 深圳百问网科技有限公司
* All rights reserved
*
* 文件名称：driver_mpu6050.c
* 摘要：
*
* 修改历史 版本号 Author 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023.08.03 v01 百问科技 创建文件
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* /

#ifndef __DRIVER_MPU6050_H
#define __DRIVER_MPU6050_H

#include <stdint.h>

typedef struct
{
    int32_t angle_x;
}mpu6050_data;

/
* 函数名称： MPU6050_Init
* 功能描述： MPU6050初始化函数,
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
/
int MPU6050_Init(void);

/
函数名称： MPU6050_GetID
* 功能描述： 读取MPU6050 ID
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 01 - 失败, 其他值 - ID
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
/
int MPU6050_GetID(void);



/
函数名称： MPU6050_ReadData
* 功能描述： 读取MPU6050数据
* 输入参数： 无
* 输出参数： pAccX / pAccY / pAccZ - 用来保存X / Y / Z轴的加速度
* pGyroX / pGyroY / pGyroZ - 用来保存X / Y / Z轴的角速度
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
/
int MPU6050_ReadData(int16_t pAccX, int16_t pAccY, int16_t pAccZ, int16_t pGyroX, int16_t pGyroY, int16_t pGyroZ);

/
函数名称： MPU6050_ParseData
* 功能描述： 解析MPU6050数据
* 输入参数： AccX / AccY / AccZ / GyroX / GyroY / GyroZ
* X / Y / Z轴的加速度,X / Y / Z轴的角速度
* 输出参数： result - 用来保存计算出的结果,目前仅支持X方向的角度
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 09 / 05 V1.0 韦东山 创建
/
void MPU6050_ParseData(int16_t AccX, int16_t AccY, int16_t AccZ, int16_t GyroX, int16_t GyroY, int16_t GyroZ, mpu6050_data result);

/
* 函数名称： MPU6050_Test
* 功能描述： MPU6050测试程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
* /
void MPU6050_Test(void);

#endif / * __DRIVER_OLED_H * /
```
下面讲解一下思路:

![](media/media/image58.png)

![](media/media/image62.png)

当我们读取MPU6050的时候不能再像之前那样, 因为之前的外设都是有中断的,
但是MPU6050这个东西它本身不适合使用中断

即使有中断, 我们也不可以像之前那样直接创建队列之后读取,
因为这个I2C通信过程实在是太过久了,
这样会导致CPU资源分布不均匀.当然你可在中断当中唤醒任务进行读I2C和写队列

我们可以创建任务, 之后采取死循环读取数据之后进行写队列, 加入队列集,
之后在InputTask当中处理

如下图(两种思路的框图)

![](media/media/image63.jpeg)

下面我们使用第二种思路进行作业

创建*[MPU6050_Task()]{.underline}*以及句柄返回函数,
更改MPU6050初始化函数, 添加队列
```c
......

static QueueHandle_t g_xQueueMPU6050; / MPU6050队列 /

/
* 函数名称： GetQueueMPU6050
* 功能描述： 陀螺仪返回队列句柄程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2025 / 08 / 21 V1.0 水水水 创建
/
QueueHandle_t GetQueueMPU6050(void)
{
    return g_xQueueMPU6050;
}

......

/
函数名称： MPU6050_Init
* 功能描述： MPU6050初始化函数,
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
* /
int MPU6050_Init(void)
{
    MPU6050_WriteRegister(MPU6050_PWR_MGMT_1, 0x00); /  / 解除休眠状态
    MPU6050_WriteRegister(MPU6050_PWR_MGMT_2, 0x00);
    MPU6050_WriteRegister(MPU6050_SMPLRT_DIV, 0x09);
    MPU6050_WriteRegister(MPU6050_CONFIG, 0x06);
    MPU6050_WriteRegister(MPU6050_GYRO_CONFIG, 0x18);
    MPU6050_WriteRegister(MPU6050_ACCEL_CONFIG, 0x18);
    
    g_xQueueMPU6050 = xQueueCreate(MPU6050_QUEUE_LEN, sizeof(mpu6050_data));
    
    return 0;
}

......

/
* 函数名称： MPU6050_Task
* 功能描述： MPU6050任务
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2025 / 08 / 21 V1.0 水水水 创建
* /
void MPU6050_Task(void *params)
{
    int16_t AccX;
    int len;
    mpu6050_data result;
    
    while (1)
    {
        /  / 读数据之后再解析
        if (MPU6050_ReadData(&AccX, NULL, NULL, NULL, NULL, NULL))
        {
            /  / 解析数据
            MPU6050_ParseData(AccX, 0, 0, 0, 0, 0, &result);
            
            /  / 写队列
            xQueueSend(g_xQueueMPU6050, &result, 0);
            
        }
    /  / 延迟一会, 减少资源占用
    vTaskDelay(50);
}
}
```
声明:
```c
......
typedef struct
{
    int32_t angle_x;
}mpu6050_data;

/
* 函数名称： GetQueueMPU6050
* 功能描述： 陀螺仪返回队列句柄程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2025 / 08 / 21 V1.0 水水水 创建
/
QueueHandle_t GetQueueMPU6050(void);

......

/
函数名称： MPU6050_Task
* 功能描述： MPU6050任务
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2025 / 08 / 21 V1.0 水水水 创建
/
void MPU6050_Task(void params);

......
```
在ferrrtos.c当中创建MPU6050的任务,
并在初始化函数当中调用*[MPU6050_Init()]{.underline}*
```c
......

/ **
* @brief FreeRTOS initialization
* @param None
* @retval None
* /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    LCD_Init();
    LCD_Clear();
    
    MPU6050_Init();
    IRReceiver_Init();
    RotaryEncoder_Init();
    LCD_PrintString(0, 0, "Starting");
    
    ......
    
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    /  / defaultTaskHandle = osThreadNew(StartDefaultTask, NULL,
    &defaultTask_attributes);
    
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    
    /  / extern void PlayMusic(void *params);
    /  / xTaskCreate(PlayMusic, "MusicTask", 128, NULL, osPriorityNormal,
    NULL);
    
    /  / 创建游戏任务
    xTaskCreate(game1_task, "GameTask", 128, NULL, osPriorityNormal,
    NULL);
    xTaskCreate(MPU6050_Task, "MPU6050Task", 128, NULL, osPriorityNormal,
    NULL);
    
    ......
    
}

......
```
回顾流程图

![](media/media/image64.png)

这个时候要在InputTask中加入读取队列集和队列, 处理数据的逻辑

回到game1.c
```c
#include "driver_mpu6050.h"

......

/  / 处理MPU6050的数据
static void ProcessMPU6050Data(void)
{
    mpu6050_data mdata = {0};
    input_data idata = {0};
    
    /  / 读取数据
    xQueueReceive(g_xQueueMPU6050, &mdata, 0);
    
    /  / 处理数据
    /  / 判断角度, 大于90°向左, 小于向右
    if (mdata.angle_x > 90)
    {
        idata.val = UPT_MOVE_LEFT;
    }
else if (mdata.angle_x < 90)
{
    idata.val = UPT_MOVE_RIGHT;
}
else {idata.val = UPT_MOVE_NONE;}

idata.dev = 0x03;

/  / 写挡球板队列
xQueueSend(g_xQueuePlatform, &idata, 0);
}

static void InputTask(void *params)
{
    QueueSetMemberHandle_t xQueueHandle = NULL;
    
    while (1)
    {
        
        ......
        
        else if (xQueueHandle =  = g_xQueueMPU6050)
        {
            ProcessMPU6050Data();
        }
    
    ......
}
}

void game1_task(void *params)
{
    ......
    
    / * 创建队列, 创建队列集, 创建输入任务 * /
    g_xQueuePlatform = xQueueCreate(10, sizeof(input_data));
    /  / g_xQueueRotary = xQueueCreateStatic(10, sizeof(rotary_data),
    g_ucQueueRotaryBuf, &g_xQueueRotaryStaticStruct);
    g_xQueueSetInput = xQueueCreateSet(IR_QUEUE_LEN + ROTARY_QUEUE_LEN +
    MPU6050_QUEUE_LEN);
    
    g_xQueueIR = GetQueueIR();
    g_xQueueRotary = GetQueueRotary();
    g_xQueueMPU6050 = GetQueueMPU6050();
    
    xQueueAddToSet(g_xQueueIR, g_xQueueSetInput);
    xQueueAddToSet(g_xQueueRotary, g_xQueueSetInput);
    xQueueAddToSet(g_xQueueMPU6050, g_xQueueSetInput);
    
    xTaskCreate(InputTask, "InputTask", 128, NULL, osPriorityNormal,
    NULL);
    
    ......
    
}
```
编译运行应该没什么问题, 烧录运行, 这个时候你会发现,
左右摇晃动不了挡球板, MPU6050失效了

为什么呢?下面给出解释

按运行先后顺序, 后创建了MPU6050任务, 先创建了游戏任务

这个时候是不是已经初始化了MPU6050了, MPU6050任务一直执行

早就填满了队列, 导致读

加入队列集是在运行游戏任务当中进行的, 可是这个时候已经写满了队列了

这个时候再写MPU队列是不是写不进去了, 队列集就得不到句柄

得不到句柄, 没人读, 直接卡住了

得到这样的信息, 我们可以确定将MPU6050任务挪一挪位置,
在游戏任务当中再创建, 确保游戏任务先创建即可
```c
/ **
* @brief FreeRTOS initialization
* @param None
* @retval None
* /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    LCD_Init();
    LCD_Clear();
    
    MPU6050_Init();
    IRReceiver_Init();
    RotaryEncoder_Init();
    LCD_PrintString(0, 0, "Starting");
    
    / * USER CODE END Init * /
    
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL,
    &defaultTask_attributes);
    
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    
    /  / extern void PlayMusic(void *params);
    /  / xTaskCreate(PlayMusic, "MusicTask", 128, NULL, osPriorityNormal,
    NULL);
    
    /  / 创建游戏任务
    xTaskCreate(game1_task, "GameTask", 128, NULL, osPriorityNormal,
    NULL);
    
    / * USER CODE END RTOS_THREADS * /
    
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
    
}
```
```c
void game1_task(void *params)
{
    uint8_t dev, data, last_data;
    
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    
    / * 创建队列, 创建队列集, 创建输入任务 * /
    g_xQueuePlatform = xQueueCreate(10, sizeof(input_data));
    /  / g_xQueueRotary = xQueueCreateStatic(10, sizeof(rotary_data),
    g_ucQueueRotaryBuf, &g_xQueueRotaryStaticStruct);
    g_xQueueSetInput = xQueueCreateSet(IR_QUEUE_LEN + ROTARY_QUEUE_LEN +
    MPU6050_QUEUE_LEN);
    
    g_xQueueIR = GetQueueIR();
    g_xQueueRotary = GetQueueRotary();
    g_xQueueMPU6050 = GetQueueMPU6050();
    
    xQueueAddToSet(g_xQueueIR, g_xQueueSetInput);
    xQueueAddToSet(g_xQueueRotary, g_xQueueSetInput);
    xQueueAddToSet(g_xQueueMPU6050, g_xQueueSetInput);
    
    xTaskCreate(MPU6050_Task, "MPU6050Task", 128, NULL, osPriorityNormal,
    NULL);
    xTaskCreate(InputTask, "InputTask", 128, NULL, osPriorityNormal,
    NULL);
    
    ......
    
}
```
编译烧录, 这个时候还是使用不了陀螺仪进行游戏控制的话,
那就是触发下面的大奖了

MPU6050与OLED是共用一个I2C进行通信的,
在时间片轮转的情况下可能发生使用的互斥冲突, 为了解决这个情况,
简单使用一个全局变量来解决互斥问题

看到*[draw.c]{.underline}*这个文件当中
```c
void draw_flushArea(byte x, byte y, byte w, byte h)
{
    static volatile int bInUsed = 0;
    while (bInUsed);
    /  / taskENTER_CRITICAL();
    bInUsed = 1;
    LCD_FlushRegion(x, y, w, h);
    bInUsed = 0;
    /  / taskEXIT_CRITICAL();
}
```
改为
```c
volatile int bInUsed = 0;
void draw_flushArea(byte x, byte y, byte w, byte h)
{
    while (bInUsed);
    /  / taskENTER_CRITICAL();
    bInUsed = 1;
    LCD_FlushRegion(x, y, w, h);
    bInUsed = 0;
    /  / taskEXIT_CRITICAL();
}
```
同时, MPU6050使用相似逻辑, 解决互斥访问这一问题
```c
/
* 函数名称： mpu6050_task
* 功能描述： MPU6050任务,它循环读取MPU6050并把数值写入队列
* 输入参数： params - 未使用
* 输出参数： 无
* 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 09 / 05 V1.0 韦东山 创建
/
void MPU6050_Task(void params)
{
    int16_t AccX;
    mpu6050_data result;
    extern volatile int bInUsed;
    int ret = 1;
    
    while (1)
    {
        while (bInUsed);
        bInUsed = 1;
        ret = MPU6050_ReadData(&AccX, NULL, NULL, NULL, NULL, NULL);
        bInUsed = 0;
        
        /  / 读数据之后再解析
        if (ret =  = 0)
        {
            /  / 解析数据
            MPU6050_ParseData(AccX, 0, 0, 0, 0, 0, &result);
            
            /  / 写队列
            xQueueSend(g_xQueueMPU6050, &result, 0);
            
        }
    /  / 延迟一会, 减少资源占用
    vTaskDelay(50);
}
}
```
###### 现象

可以左右摇晃和使用编码器,遥控器进行挡球板控制

##### 额外

这个额外工程是为了下一部分作准备

工程文件: 17_queue_car_dispatch

创建赛车游戏以及完善相关措施,
以上一节的*[16_queueset_game_mpu6050]{.underline}*

作为原型,
添加工程文件当中的*[game2.c]{.underline}*与*[game2.h]{.underline}*

对game2.c进行修改并测试
```c
/ *
* Project: N|Watch
* Author: Zak Kemble, contact@zakkemble.co.uk
* Copyright: (C) 2013 by Zak Kemble
* License: GNU GPL v3 (see License.txt)
* Web: http: /  / blog.zakkemble.co.uk / diy - digital - wristwatch /
* /

#include <stdlib.h>
#include <stdio.h>

#include "cmsis_os.h"
#include "FreeRTOS.h" /  / ARM.FreeRTOS::RTOS:Core
#include "task.h" /  / ARM.FreeRTOS::RTOS:Core
#include "event_groups.h" /  / ARM.FreeRTOS::RTOS:Event Groups
#include "semphr.h" /  / ARM.FreeRTOS::RTOS:Core

#include "draw.h"
#include "resources.h"

#include "driver_lcd.h"
#include "driver_ir_receiver.h"
#include "driver_rotary_encoder.h"
#include "driver_mpu6050.h"

#define Test_USE 1
#define NOINVERT false
#define INVERT true

#define CAR_COUNT 3
#define CAR_WIDTH 12
#define CAR_LENGTH 15
#define ROAD_SPEED 6

static uint32_t g_xres, g_yres, g_bpp;
static uint8_t *g_framebuffer;

typedef struct{
    int x;
    int y;
    int control_key;
}car;

car g_cars[3] = {
    {0, 0, IR_KEY_1},
    {0, 17, IR_KEY_2},
    {0, 34, IR_KEY_3}
};

static const byte carImg[] = {
    0x40,0xF8,0xEC,0x2C,0x2C,0x38,0xF0,0x10,0xD0,0x30,0xE8,0x4C,0x4C,0x9C,0xF0,
    0x02,0x1F,0x37,0x34,0x34,0x1C,0x0F,0x08,0x0B,0x0C,0x17,0x32,0x32,0x39,0x0F,
};

static const byte roadMarking[] = {
    0x01,0x01,0x01,0x01,0x01,0x01,0x01,0x01,
};

static const byte clearImg[30] = {0};

#if Test_USE
void car_test(void)
{
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    
    draw_bitmap(0, 0, carImg, 15, 16, NOINVERT, 0);
    draw_flushArea(0, 0, 15, 16);
    
    draw_bitmap(0, 16, roadMarking, 8, 1, NOINVERT, 0);
    draw_flushArea(0, 16, 8, 1);
    
    while (1);
}
#endif

static void ShowCar(car *pcar)
{
    draw_bitmap(pcar - > x, pcar - > y, carImg, 15, 16, NOINVERT, 0);
    draw_flushArea(pcar - > x, pcar - > y, 15, 16);
}

static void HideCar(car *pcar)
{
    draw_bitmap(pcar - > x, pcar - > y, clearImg, 15, 16, NOINVERT, 0);
    draw_flushArea(pcar - > x, pcar - > y, 15, 16);
}
```
在*[FreeRTOS.c]{.underline}*当中进行调用测试函数, 同时对所有任务进行注释
```c
......

void MX_FREERTOS_Init(void); / * (MISRA C 2004 rule 8.1) * /

/ **
* @brief FreeRTOS initialization
* @param None
* @retval None
* /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    LCD_Init();
    LCD_Clear();
    
    MPU6050_Init();
    IRReceiver_Init();
    RotaryEncoder_Init();
    LCD_PrintString(0, 0, "Starting");
    extern void car_test(void);
    car_test();
    
    / * USER CODE END Init * /
    
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL,
    &defaultTask_attributes);
    
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    
    /  / extern void PlayMusic(void *params);
    /  / xTaskCreate(PlayMusic, "MusicTask", 128, NULL, osPriorityNormal,
    NULL);
    
    /  / 创建游戏任务
    /  / xTaskCreate(game1_task, "GameTask", 128, NULL, osPriorityNormal,
    NULL);
    
    / * USER CODE END RTOS_THREADS * /
    
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
    
}

......
```
测试现象应为显现1辆赛车与1个轨道

下main的思路是将3辆赛车分别创建任务
```c
......

#define Test_Use 0

......

#if Test_USE
void car_test(void)
{
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    
    draw_bitmap(0, 0, carImg, 15, 16, NOINVERT, 0);
    draw_flushArea(0, 0, 15, 16);
    
    draw_bitmap(0, 16, roadMarking, 8, 1, NOINVERT, 0);
    draw_flushArea(0, 16, 8, 1);
    
    while (1);
}
#endif

void car_game(void)
{
    int i = 0, x = 0, j = 0;
    
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    
    / 画出多个路标 /
    for(i = 0; i < 3; i +  + )
    {
        / * 绘制多条道路 * /
        for (j = 0; j < 8; j +  + )
        {
            draw_bitmap(16  j, 16 + (17  i), roadMarking, 8, 1, NOINVERT, 0);
            draw_flushArea(16  j, 16 + (17  i), 8, 1);
        }
    / * 绘制汽车 * /
    draw_bitmap(g_cars[i].x, g_cars[i].y, carImg, 15, 16, NOINVERT,
    0);
    draw_flushArea(g_cars[i].x, g_cars[i].y, 15, 16);
}

/  / 创建任务, 每个汽车对应一个任务
xTaskCreate(CarTask, "Car1", 128, &g_cars[0], osPriorityNormal,
NULL);
xTaskCreate(CarTask, "Car2", 128, &g_cars[1], osPriorityNormal,
NULL);
xTaskCreate(CarTask, "Car3", 128, &g_cars[2], osPriorityNormal,
NULL);
}
```
之后通过队列进行读取信息, 由于一个一个分辨3个队列当中的信息太难受,
直接将遥控器信息写入一个队列各个任务分别读取从而减少工程量

同时为了更加规范地在红外接收遥控器中写队列,
将多个队列统一放入队列数组当中, 一起写入数据, 下面是代码:
```c
/ * Copyright (s) 2019 深圳百问网科技有限公司
* All rights reserved
*
* 文件名称：driver_ir_receiver.c
* 摘要：
*
* 修改历史 版本号 Author 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023.08.04 v01 百问科技 创建文件
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* /

#include "driver_ir_receiver.h"
#include "driver_lcd.h"
#include "driver_timer.h"
#include "stm32f1xx_hal.h"
#include "tim.h"
#include "typedefs.h"

/ * 环形缓冲区: 用来保存解析出来的按键,可以防止丢失 * /
#define BUF_LEN 128
static unsigned char g_KeysBuf[BUF_LEN];
static int g_KeysBuf_R, g_KeysBuf_W;

static uint64_t g_IRReceiverIRQ_Timers[68];
static int g_IRReceiverIRQ_Cnt = 0;
/  / static uint32_t g_last_val;

/  / extern QueueHandle_t g_xQueuePlatform; / * 挡球板队列 * /
/ 定义自己的队列 /
static QueueHandle_t g_xQueueIR; /  / 红外队列
static QueueHandle_t g_xQueus[10];
static uint8_t g_queue_cnt = 0;

#define NEXT_POS(x) ((x + 1) % BUF_LEN)

......

/
* 函数名称： RegisterQueueHandle
* 功能描述： 队列密钥保存
* 输入参数： QueueHandle_t queueHandle
* 输出参数： 无
* 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2025 / 08 / 22 V1.0 水水水 创建
* /
void RegisterQueueHandle(QueueHandle_t queueHandle)
{
    if (g_queue_cnt < 10)
    {
        g_xQueus[g_queue_cnt] = queueHandle;
        g_queue_cnt +  + ;
    }
}

/
* 函数名称： DispatchKey
* 功能描述： 赛车密钥分发
* 输入参数： ir_data *pidata
* 输出参数： 无
* 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2025 / 08 / 22 V1.0 水水水 创建
* /
static void vDispatchKey(ir_data *pidata)
{
    /  / extern QueueHandle_t g_xQueueCar1;
    /  / extern QueueHandle_t g_xQueueCar2;
    /  / extern QueueHandle_t g_xQueueCar3;
    /  /
    /  / xQueueSendFromISR(g_xQueueCar1, pidata, NULL);
    /  / xQueueSendFromISR(g_xQueueCar2, pidata, NULL);
    /  / xQueueSendFromISR(g_xQueueCar3, pidata, NULL);
    
    uint8_t i = 0;
    for (i = 0; i < g_queue_cnt; i +  + )
    {
        xQueueSendFromISR(g_xQueus[i], pidata, NULL);
    }
}


/
* 函数名称： IRReceiver_IRQTimes_Parse
* 功能描述： 解析中断回调函数里记录的时间序列,得到的device和key放入环形缓冲区
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 0 - 成功, ( - 1) - 失败
* 修改日期： 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 04 V1.0 韦东山 创建
* /
static int IRReceiver_IRQTimes_Parse(void)
{
    uint64_t time;
    int i;
    int m, n;
    unsigned char datas[4];
    unsigned char data = 0;
    int bits = 0;
    int byte = 0;
    ir_data idata;
    
    / * 1. 判断前导码 : 9ms的低脉冲, 4.5ms高脉冲 * /
    time = g_IRReceiverIRQ_Timers[1] - g_IRReceiverIRQ_Timers[0];
    if (time < 8000000 || time > 10000000)
    {
        return - 1;
    }

time = g_IRReceiverIRQ_Timers[2] - g_IRReceiverIRQ_Timers[1];
if (time < 3500000 || time > 55000000)
{
    return - 1;
}

/ * 2. 解析数据 * /
for (i = 0; i < 32; i +  + )
{
    m = 3 + i*2;
    n = m + 1;
    time = g_IRReceiverIRQ_Timers[n] - g_IRReceiverIRQ_Timers[m];
    data << = 1;
    bits +  + ;
    if (time > 1000000)
    {
        / * 得到了数据1 * /
        data | = 1;
    }

if (bits =  = 8)
{
    datas[byte] = data;
    byte +  + ;
    data = 0;
    bits = 0;
}
}

/ * 判断数据正误 * /
datas[1] = ~datas[1];
datas[3] = ~datas[3];

if ((datas[0] != datas[1]) || (datas[2] != datas[3]))
{
    g_IRReceiverIRQ_Cnt = 0;
    return - 1;
}

/  / PutKeyToBuf(datas[0]);
/  / PutKeyToBuf(datas[2]);

/ 写数据进队列 /
idata.dev = datas[0];

/  / if (datas[2] =  = 0xe0)
/  / {
    /  / idata.val = UPT_MOVE_LEFT;
    /  / }
/  / else if (datas[2] =  = 0x90)
/  / {
    /  / idata.val = UPT_MOVE_RIGHT;
    /  / }
/  / else
/  / {
    /  / idata.val = UPT_MOVE_NONE;
    /  / }
/ 之前的工程是将数据识别之后对应控制游戏的按键从而放入队列 /

idata.val = datas[2];
/ 上一次的数值无需记录 /
/  / g_last_val = idata.val;

/  / xQueueSendFromISR(g_xQueueIR, &idata, NULL);
vDispatchKey(&idata);

return 0;
}

......

/
* 函数名称： IRReceiver_IRQ_Callback
* 功能描述： 红外接收器的中断回调函数,记录中断时刻
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 无
* 修改日期： 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 04 V1.0 韦东山 创建
* /
void IRReceiver_IRQ_Callback(void)
{
    uint64_t time;
    static uint64_t pre_time = 0;
    ir_data idata; /  / input_data
    
    / * 1. 记录中断发生的时刻 * /
    time = system_get_ns();
    
    / * 一次按键的最长数据 = 引导码 + 32个数据"1" = 9 + 4.5 + 2.25*32 = 85.5ms
    * 如果当前中断的时刻, 举例上次中断的时刻超过这个时间, 以前的数据就抛弃
    * /
    if (time - pre_time > 100000000)
    {
        g_IRReceiverIRQ_Cnt = 0;
    }
pre_time = time;

g_IRReceiverIRQ_Timers[g_IRReceiverIRQ_Cnt] = time;

/ * 2. 累计中断次数 * /
g_IRReceiverIRQ_Cnt +  + ;

/ * 3. 次数达标后, 解析数据, 放入buffer * /
if (g_IRReceiverIRQ_Cnt =  = 4)
{
    / * 是否重复码 * /
    if (isRepeatedKey())
    {
        / * device: 0, val: 0, 表示重复码 * /
        /  / PutKeyToBuf(0);
        /  / PutKeyToBuf(0);
        
        / 写数据进队列 /
        idata.dev = 0;
        idata.val = 0; /  / g_last_val, 这更改之后直接上报重复码
        /  / xQueueSendFromISR(g_xQueueIR, &idata, NULL);
        vDispatchKey(&idata);
        
        g_IRReceiverIRQ_Cnt = 0;
    }
}
if (g_IRReceiverIRQ_Cnt =  = 68)
{
    IRReceiver_IRQTimes_Parse();
    g_IRReceiverIRQ_Cnt = 0;
}
}


/
* 函数名称： IRReceiver_Init
* 功能描述： 红外接收器的初始化函数
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 无
* 修改日期： 版本号 修改人 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023 / 08 / 04 V1.0 韦东山 创建
* /
void IRReceiver_Init(void)
{
    / * PA10在MX_GPIO_Init()中已经被配置为双边沿触发, 并使能了中断 * /
    #if 0
    / *Configure GPIO pin : PB10 * /
    GPIO_InitStruct.Pin = GPIO_PIN_10;
    GPIO_InitStruct.Mode = GPIO_MODE_EVT_RISING_FALLING;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
    #endif
    g_xQueueIR = xQueueCreate(IR_QUEUE_LEN, sizeof(ir_data));
    RegisterQueueHandle(g_xQueueIR);
}

......
```
```c
......

/
* 函数名称： RegisterQueueHandle
* 功能描述： 队列密钥保存
* 输入参数： QueueHandle_t queueHandle
* 输出参数： 无
* 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2025 / 08 / 22 V1.0 水水水 创建
* /
void RegisterQueueHandle(QueueHandle_t queueHandle);


#endif / * _DRIVER_IR_RECEIVER_H * /
```
在*[game2.c]{.underline}*当中加入任务函数, 通过传入的结构体来互斥运行
```c
static void ShowCar(car pcar)
{
    draw_bitmap(pcar - > x, pcar - > y, carImg, 15, 16, NOINVERT, 0);
    draw_flushArea(pcar - > x, pcar - > y, 15, 16);
}

static void HideCar(car pcar)
{
    draw_bitmap(pcar - > x, pcar - > y, clearImg, 15, 16, NOINVERT, 0);
    draw_flushArea(pcar - > x, pcar - > y, 15, 16);
}

static void CarTask(void params)
{
    car pcar = params;
    ir_data idata; /  / input_data
    
    / * 创建自己的队列 * /
    QueueHandle_t xQueueIR = xQueueCreate(10, sizeof(ir_data));
    
    / * 注册队列 * /
    RegisterQueueHandle(xQueueIR);
    
    while (1)
    {
        / *读取按键值 : 读队列* /
        xQueueReceive(xQueueIR, &idata, portMAX_DELAY);
        
        / 控制汽车向右移动 /
        
        / * 控制汽车往右移动 * /
        if (idata.val =  = pcar - >control_key)
        {
            if (pcar - >x < g_xres - CAR_LENGTH)
            {
                / * 隐藏汽车 * /
                HideCar(pcar);
                
                / * 调整位置 * /
                pcar - >x +  = 20;
                if (pcar - >x > g_xres - CAR_LENGTH)
                {
                    pcar - >x = g_xres - CAR_LENGTH;
                }
            
            / * 重新显示汽车 * /
            ShowCar(pcar);
        }
}
}
}
```
最后修改*[freertos.c]{.underline}*, 调用函数创建任务
```c
......

/ **
* @brief FreeRTOS initialization
* @param None
* @retval None
* /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    LCD_Init();
    LCD_Clear();
    
    MPU6050_Init();
    IRReceiver_Init();
    RotaryEncoder_Init();
    LCD_PrintString(0, 0, "Starting");
    extern void car_game(void);
    
    / * USER CODE END Init * /
    
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL,
    &defaultTask_attributes);
    
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    
    /  / extern void PlayMusic(void *params);
    /  / xTaskCreate(PlayMusic, "MusicTask", 128, NULL, osPriorityNormal,
    NULL);
    
    /  / 创建游戏任务
    /  / xTaskCreate(game1_task, "GameTask", 128, NULL, osPriorityNormal,
    NULL);
    car_game();
    
    / * USER CODE END RTOS_THREADS * /
    
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
    
}

......
```
###### 现象

三辆赛车可以通过遥控器的 1 2 3分别运动

## 信号量与互斥量

### 信号量的本质

信号是起提醒的作用, 量起计数作用, 信号量究其根本也是一个队列,
不过这个队列不涉及数据的真正传输,
只涉及到数据个数的统计(也就是数据个数的加加减减)

下main来一张框图对比二者:

![](media/media/image65.jpeg)

举个例子讲解信号量的运行机制:

**信号量示例：车进程与通行证**

**1. 二值信号量（1辆车通行）**

**场景**：一条只有一辆车能通过的单车道。

**工作原理**：信号量初始值为 1，表示道路空闲。车得到通行证（信号量值
1），就能通行；信号量减为 0。其他车必须等待信号量变回 1。

**2. 计数信号量（最多3辆车通行）**

**场景**：一条最多允许 3 辆车同时通行的路。

**工作原理**：信号量初始值为 3，表示最多 3
辆车可以同时通行。车得到通行证（信号量值减 1）后通行。如果信号量值为
0，表示已有 3
辆车在路上，其他车必须等待。当有车通过并释放通行证（信号量加
1），其他车才能继续进入。

**3. 最大限制**

**作用**：信号量的最大值决定了同时通行的车辆数量。例如，最大限制为 3
时，最多只能允许 3 辆车通过。如果信号量值为
0，就不再发放通行证，其他车只能等待。

如果说队列是receive和send的话, 那对应到信号量就是take和give,
give就像是放票了可以申请了, take就是大家都在抢有空闲的票

也就是说, 放票的是give, 一旦give了, cnt++, 之后唤醒等待者;
这个票对应的cnt就是信号量

再看take, 车得到了通行证, cnt--, 可以进城, 没有票的只能等待甚至回家,
当通行证发放到了最大值, 那后面的车也只能光靠等或者回家了(阻塞状态)

### 演示使用信号量

工程文件:18_semaphore_not_use, 19_semaphore_count, 20_semaphore_binary,

将上一次编写的遥控控制赛车更改为不需要遥控控制通过自增执行的赛车游戏:
```c
static void CarTask(void params)
{
    car pcar = params;
    ir_data idata; /  / input_data
    
    / * 创建自己的队列 * /
    QueueHandle_t xQueueIR = xQueueCreate(10, sizeof(ir_data));
    
    / * 注册队列 * /
    RegisterQueueHandle(xQueueIR);
    
    while (1)
    {
        /  /  / *读取按键值 : 读队列* /
        /  / xQueueReceive(xQueueIR, &idata, portMAX_DELAY);
        /  /
        /  /  / * 控制汽车往右移动 * /
        /  / if (idata.val =  = pcar - >control_key)
        {
            if (pcar - >x < g_xres - CAR_LENGTH)
            {
                / * 隐藏汽车 * /
                HideCar(pcar);
                
                / * 调整位置 * /
                pcar - >x +  = 1;
                if (pcar - >x > g_xres - CAR_LENGTH)
                {
                    pcar - >x = g_xres - CAR_LENGTH;
                }
            
            / * 重新显示汽车 * /
            ShowCar(pcar);
            vTaskDelay(50);
            
            if (pcar - >x =  = g_xres - CAR_LENGTH)
            {
                vTaskDelete(NULL);
            }
    }
}
}
}
```
这里就完成了工程文件18_semaphore_not_use

在这个基础上copy一份18进行更改, 下面将基于其进行工程19

在这一部分, 我们期望通过使用信号量来控制其中两个车获得运行资格

需要用到信号量的函数

使用信号量之前，要先创建，得到一个句柄；使用信号量时，要使用句柄来表明使用哪个信号量。
对于二进制信号量、计数型信号量，它们的创建函数不一样：

![](media/media/image72.jpeg)

下面讲解二进制信号量, 大部分没什么区别,
只是二进制已经帮你限定好了最大值, 更改一下*[game2.c]{.underline}*
```c
void car_game(void)
{
    int i = 0, x = 0, j = 0;
    
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    
    / * 计数型信号量 * /
    /  / g_xSemTicks = xSemaphoreCreateCounting(3, 2);
    
    / * 二进制型信号量 * /
    g_xSemTicks = xSemaphoreCreateBinary(); /  / 默认为0, 需要手动give
    
    /  / 因为max值为1, give多次也只有第一次有效
    xSemaphoreGive(g_xSemTicks);
    xSemaphoreGive(g_xSemTicks);
    xSemaphoreGive(g_xSemTicks);
    
    / 画出多个路标 /
    for(i = 0; i < 3; i +  + )
    {
        / * 绘制多条道路 * /
        for (j = 0; j < 8; j +  + )
        {
            draw_bitmap(16 * j, 16 + (17 * i), roadMarking, 8, 1, NOINVERT, 0);
            draw_flushArea(16 * j, 16 + (17 * i), 8, 1);
        }
    / * 绘制汽车 * /
    draw_bitmap(g_cars[i].x, g_cars[i].y, carImg, 15, 16, NOINVERT,
    0);
    draw_flushArea(g_cars[i].x, g_cars[i].y, 15, 16);
}

/  / 创建任务, 每个汽车对应一个任务
xTaskCreate(CarTask, "Car1", 128, &g_cars[0], osPriorityNormal,
NULL);
xTaskCreate(CarTask, "Car2", 128, &g_cars[1], osPriorityNormal,
NULL);
xTaskCreate(CarTask, "Car3", 128, &g_cars[2], osPriorityNormal,
NULL);
}
```
其他代码照旧, 这就是工程文件20了.编译烧录

#### 现象

一辆一辆跑到终点

### 互斥量

互斥量实质上是信号量的一种变种

#### 优先级反转

工程文件:21_semaphore_priority_inversion

使用信号量(无论是高优先级还是低优先级)的时候, 很有可能遇到优先级反转,
是什么呢? 就是高优先级任务按道理来说是运行更加多和更加完整的,
但因为信号量反而导致低优先级任务运行更频繁

下面给一张图解

![](media/media/image73.jpeg)

![](media/media/image74.jpeg)

下面我们模拟一下这种情况, 基于之前的二进制信号量工程来执行

更改3个任务的优先级, 并分别创建任务函数
```c
void car_game(void)
{
    int i = 0, x = 0, j = 0;
    
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    
    / * 计数型信号量 * /
    /  / g_xSemTicks = xSemaphoreCreateCounting(3, 2);
    
    / * 二进制型信号量 * /
    g_xSemTicks = xSemaphoreCreateBinary(); /  / 默认为0, 需要手动give
    
    /  / 因为max值为1, give多次也只有第一次有效
    xSemaphoreGive(g_xSemTicks);
    xSemaphoreGive(g_xSemTicks);
    xSemaphoreGive(g_xSemTicks);
    
    / 画出多个路标 /
    for(i = 0; i < 3; i +  + )
    {
        / * 绘制多条道路 * /
        for (j = 0; j < 8; j +  + )
        {
            draw_bitmap(16 * j, 16 + (17 * i), roadMarking, 8, 1, NOINVERT, 0);
            draw_flushArea(16 * j, 16 + (17 * i), 8, 1);
        }
    / * 绘制汽车 * /
    draw_bitmap(g_cars[i].x, g_cars[i].y, carImg, 15, 16, NOINVERT,
    0);
    draw_flushArea(g_cars[i].x, g_cars[i].y, 15, 16);
}

/  / 创建任务, 每个汽车对应一个任务
xTaskCreate(Car1Task, "Car1", 128, &g_cars[0], osPriorityNormal,
NULL);
xTaskCreate(Car2Task, "Car2", 128, &g_cars[1], osPriorityNormal +
2, NULL);
xTaskCreate(Car3Task, "Car3", 128, &g_cars[2], osPriorityNormal +
3, NULL);
}
```
车辆1任务函数继承原来的, 2和3在原来基础上改, 2无需take, 3需要take,
但2比3先执行任务
```c
static void Car1Task(void params)
{
    car pcar = params;
    ir_data idata; /  / input_data
    
    / * 创建自己的队列 * /
    QueueHandle_t xQueueIR = xQueueCreate(10, sizeof(ir_data));
    
    / * 注册队列 * /
    RegisterQueueHandle(xQueueIR);
    
    / * 获得信号量 * /
    xSemaphoreTake(g_xSemTicks, portMAX_DELAY);
    
    while (1)
    {
        /  /  / *读取按键值 : 读队列* /
        /  / xQueueReceive(xQueueIR, &idata, portMAX_DELAY);
        /  /
        /  /  / * 控制汽车往右移动 * /
        /  / if (idata.val =  = pcar - >control_key)
        {
            if (pcar - >x < g_xres - CAR_LENGTH)
            {
                / * 隐藏汽车 * /
                HideCar(pcar);
                
                / * 调整位置 * /
                pcar - >x +  = 1;
                if (pcar - >x > g_xres - CAR_LENGTH)
                {
                    pcar - >x = g_xres - CAR_LENGTH;
                }
            
            / * 重新显示汽车 * /
            ShowCar(pcar);
            vTaskDelay(50);
            
            if (pcar - >x =  = g_xres - CAR_LENGTH)
            {
                / * 释放信号量 * /
                xSemaphoreGive(g_xSemTicks);
                
                / * 结束任务 * /
                vTaskDelete(NULL);
            }
    }
}
}
}

static void Car2Task(void *params)
{
    car *pcar = params;
    ir_data idata; /  / input_data
    
    vTaskDelay(1000); /  / 1s之后再运行
    
    / * 创建自己的队列 * /
    QueueHandle_t xQueueIR = xQueueCreate(10, sizeof(ir_data));
    
    / * 注册队列 * /
    RegisterQueueHandle(xQueueIR);
    
    while (1)
    {
        /  /  / *读取按键值 : 读队列* /
        /  / xQueueReceive(xQueueIR, &idata, portMAX_DELAY);
        /  /
        /  /  / * 控制汽车往右移动 * /
        /  / if (idata.val =  = pcar - >control_key)
        {
            if (pcar - >x < g_xres - CAR_LENGTH)
            {
                / * 隐藏汽车 * /
                HideCar(pcar);
                
                / * 调整位置 * /
                pcar - >x +  = 1;
                if (pcar - >x > g_xres - CAR_LENGTH)
                {
                    pcar - >x = g_xres - CAR_LENGTH;
                }
            
            / * 重新显示汽车 * /
            ShowCar(pcar);
            /  / vTaskDelay(50);
            mdelay(50);
            
            if (pcar - >x =  = g_xres - CAR_LENGTH)
            {
                / * 结束任务 * /
                vTaskDelete(NULL);
            }
    }
}
}
}

static void Car3Task(void *params)
{
    car *pcar = params;
    ir_data idata; /  / input_data
    
    vTaskDelay(2000); /  / 2s之后再运行
    
    / * 创建自己的队列 * /
    QueueHandle_t xQueueIR = xQueueCreate(10, sizeof(ir_data));
    
    / * 注册队列 * /
    RegisterQueueHandle(xQueueIR);
    
    / * 获得信号量 * /
    xSemaphoreTake(g_xSemTicks, portMAX_DELAY);
    
    while (1)
    {
        /  /  / *读取按键值 : 读队列* /
        /  / xQueueReceive(xQueueIR, &idata, portMAX_DELAY);
        /  /
        /  /  / * 控制汽车往右移动 * /
        /  / if (idata.val =  = pcar - >control_key)
        {
            if (pcar - >x < g_xres - CAR_LENGTH)
            {
                / * 隐藏汽车 * /
                HideCar(pcar);
                
                / * 调整位置 * /
                pcar - >x +  = 1;
                if (pcar - >x > g_xres - CAR_LENGTH)
                {
                    pcar - >x = g_xres - CAR_LENGTH;
                }
            
            / * 重新显示汽车 * /
            ShowCar(pcar);
            vTaskDelay(50);
            
            if (pcar - >x =  = g_xres - CAR_LENGTH)
            {
                / * 释放信号量 * /
                xSemaphoreGive(g_xSemTicks);
                
                / * 结束任务 * /
                vTaskDelete(NULL);
            }
    }
}
}
}
```
这时候, 你就会发现优先级反转了, 因为本身任务调用的层级很多,
所以导致2几乎卡死了1, 1又无法give, 3就没法了, 只有等2销毁了才能运动
```c
static void Car2Task(void params)
{
    car pcar = params;
    ir_data idata; /  / input_data
    
    vTaskDelay(1000); /  / 1s之后再运行
    
    / * 创建自己的队列 * /
    QueueHandle_t xQueueIR = xQueueCreate(10, sizeof(ir_data));
    
    / * 注册队列 * /
    RegisterQueueHandle(xQueueIR);
    
    while (1)
    {
        /  /  / *读取按键值 : 读队列* /
        /  / xQueueReceive(xQueueIR, &idata, portMAX_DELAY);
        /  /
        /  /  / * 控制汽车往右移动 * /
        /  / if (idata.val =  = pcar - >control_key)
        {
            if (pcar - >x < g_xres - CAR_LENGTH)
            {
                / * 隐藏汽车 * /
                HideCar(pcar);
                
                / * 调整位置 * /
                pcar - >x +  = 1;
                if (pcar - >x > g_xres - CAR_LENGTH)
                {
                    pcar - >x = g_xres - CAR_LENGTH;
                }
            
            / * 重新显示汽车 * /
            ShowCar(pcar);
            /  / vTaskDelay(50);
            mdelay(50);
            
            if (pcar - >x =  = g_xres - CAR_LENGTH)
            {
                / * 结束任务 * /
                /  / vTaskDelete(NULL);
            }
    }
}
}
}
```
再更改一次, 不删除任务2, 那就会导致一直卡死

这就好像学校有一台超算, 访客需要指纹才能使用超算,
一个学生(优先级低)按了指纹(信号量)准备用了,
这个时候主任(优先级中)带着一帮人来参观(不适用不需要指纹),
结果校长(高优先级)来了想要用却发现已经有人在用了还有个主任在捣蛋, 没法用

解决方法: 互斥量

大致思想:(优先级继承)

学生在使用超算, 主任带人参观, 校长也想用超算, 学生继承优先级,
获得更大权力, 主任一边凉快去, 学生用完超算, give同时唤醒校长,
恢复优先级并且自己再退出(删除任务), 校长开始使用超算, 结束后主任带人参观

**互斥量创建**

互斥量是一种特殊的二进制信号量。

使用互斥量时，先创建、然后去获得、释放它。使用句柄来表示一个互斥量。

创建互斥量的函数有2种：动态分配内存，静态分配内存，函数原型如下：
```c
/ * 创建一个互斥量，返回它的句柄。
* 此函数内部会分配互斥量结构体
* 返回值: 返回句柄，非NULL表示成功
* /
SemaphoreHandle_t xSemaphoreCreateMutex( void ); / *
创建一个互斥量，返回它的句柄。
*
此函数无需动态分配内存，所以需要先有一个StaticSemaphore_t结构体，并传入它的指针
* 返回值: 返回句柄，非NULL表示成功
* /
SemaphoreHandle_t xSemaphoreCreateMutexStatic( StaticSemaphore_t *pxMutexBuffer
);
```
要想使用互斥量，需要在配置文件FreeRTOSConfig.h中定义：
```c
#define configUSE_MUTEXES 1
```
###### Note

这部分实操基于上一个优先级反转的例子工程来修改,
工程文件22_mutex_priority_inversion

更改game2.c
```c
void car_game(void)
{
    int i = 0, x = 0, j = 0;
    
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    
    / * 二进制型信号量 * /
    g_xSemTicks = xSemaphoreCreateMutex(); /  / 默认为1,
    不需要像之前那样二进制信号量的give
    
    / 画出多个路标 /
    for(i = 0; i < 3; i +  + )
    {
        / * 绘制多条道路 * /
        for (j = 0; j < 8; j +  + )
        {
            draw_bitmap(16 * j, 16 + (17 * i), roadMarking, 8, 1, NOINVERT, 0);
            draw_flushArea(16 * j, 16 + (17 * i), 8, 1);
        }
    / * 绘制汽车 * /
    draw_bitmap(g_cars[i].x, g_cars[i].y, carImg, 15, 16, NOINVERT,
    0);
    draw_flushArea(g_cars[i].x, g_cars[i].y, 15, 16);
}

/  / 创建任务, 每个汽车对应一个任务
xTaskCreate(Car1Task, "Car1", 128, &g_cars[0], osPriorityNormal,
NULL);
xTaskCreate(Car2Task, "Car2", 128, &g_cars[1], osPriorityNormal +
2, NULL);
xTaskCreate(Car3Task, "Car3", 128, &g_cars[2], osPriorityNormal +
3, NULL);
}
```
这时候发生了一个很有趣的问题, 卡死了, 经过调试是卡死在下面的地方了

![](media/media/image75.png)

这个是互斥访问OLED的地方

这个时候就是全局变量解决互斥访问的问题了, 因为小车2的任务是先创建的,
比小车任务1的优先级要高一些, 这个时候, 它们两个访问固然问题还好,
但是临时加了一个更高优先级的小车3进来,
这导致了3一直在判断能否使用并且3有阻塞延时, 最后卡死了程序

现在修改一下这个*[draw.c]{.underline}*文件中的函数, 使用互斥量解决问题,
同时I2C设备上还挂载了MPU6050, 这也需要更改
```c
......

/ * USER CODE BEGIN Includes * /
#include "event_groups.h" /  / ARM.FreeRTOS::RTOS:Event Groups
#include "semphr.h" /  / ARM.FreeRTOS::RTOS:Core

/ * USER CODE END Includes * /

......

/ * Private variables
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN Variables * /
static SemaphoreHandle_t g_xI2CMutex;
void game1_task(void *params);

/ * USER CODE END Variables * /

......

/ * Private function prototypes
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FunctionPrototypes * /

void GetI2C(void)
{
    / * 等待互斥量 * /
    xSemaphoreTake(g_xI2CMutex, portMAX_DELAY);
    
}


void PutI2C(void)
{
    / * 释放互斥量 * /
    xSemaphoreGive(g_xI2CMutex);
}

......

/ **
* @brief FreeRTOS initialization
* @param None
* @retval None
* /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    
    /  / 初始化互斥量
    g_xI2CMutex = xSemaphoreCreateMutex();
    
    ......
    
}
```
```c
/
* 函数名称： mpu6050_task
* 功能描述： MPU6050任务,它循环读取MPU6050并把数值写入队列
* 输入参数： params - 未使用
* 输出参数： 无
* 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 09 / 05 V1.0 韦东山 创建
/
void MPU6050_Task(void params)
{
    int16_t AccX;
    mpu6050_data result;
    extern void GetI2C(void);
    extern void PutI2C(void);
    int ret = 1;
    
    while (1)
    {
        
        GetI2C();
        ret = MPU6050_ReadData(&AccX, NULL, NULL, NULL, NULL, NULL);
        PutI2C();
        /  / 读数据之后再解析
        if (ret =  = 0)
        {
            /  / 解析数据
            MPU6050_ParseData(AccX, 0, 0, 0, 0, 0, &result);
            
            /  / 写队列
            xQueueSend(g_xQueueMPU6050, &result, 0);
            
        }
    /  / 延迟一会, 减少资源占用
    vTaskDelay(50);
}
}
```
```c
extern void GetI2C(void);
extern void PutI2C(void);
/  / volatile int bInUsed = 0;
void draw_flushArea(byte x, byte y, byte w, byte h)
{
    GetI2C();
    LCD_FlushRegion(x, y, w, h);
    PutI2C();
}
```
编译烧录即可运行, 这个时候按照道理来说, 1会先运行一会,
之后2运行(1几乎卡死), 3创建, 1通过互斥量继承优先级, 1和2齐头并进,
1到达终点之后3开始运行, 2也一起运行(2虽然有阻塞的mdelay但是3的优先更高,
3触发更加频繁, 3并不会特别明显卡死)

##### 现象

三车运行, 情况如上

## 事件组

![](media/media/image76.jpeg)

至于事件是需要同时发生还是要其中之一发生即可,
Task_A与Task_B仍有高八位进行确定是or还是and

下面大致讲一下运行机制, C写事件, 比如C写了事件5,
之后事件组就会通过链表的轮询确定谁谁谁订阅了事件5, 符不符合要求, 先看A,
不符合跳过, 到B(假设B是只需其中一个发生即可)符合了, 唤醒,
再往下检查看看有无符合的

总而言之, 这就像是一个广播系统

### 事件组使用

工程文件:24_eventgroup_or, 25_eventgroup_and, 26_eventgroup_mpu6050

这些工程文件基于上次互斥量的文件更改, 先创建对应的事件组

使用事件组之前，要先创建，得到一个句柄；使用事件组时，要使用句柄来表明使用哪个事件组。

有两种创建方法：动态分配内存、静态分配内存。函数原型如下：
```c
/ * 创建一个事件组，返回它的句柄。
* 此函数内部会分配事件组结构体
* 返回值: 返回句柄，非NULL表示成功
* /
EventGroupHandle_t xEventGroupCreate( void ); / * 创建一个事件组，返回它的句柄。
*
此函数无需动态分配内存，所以需要先有一个StaticEventGroup_t结构体，并传入它的指针
* 返回值: 返回句柄，非NULL表示成功
* /
EventGroupHandle_t xEventGroupCreateStatic( StaticEventGroup_t *
pxEventGroupBuffer );
```
在*[game2.c]{.underline}*设置事件组先
```c
......

static EventGroupHandle_t g_xEventCar;

......

void car_game(void)
{
    int i = 0, x = 0, j = 0;
    
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    
    / * 互斥量创建 * /
    /  / g_xSemTicks = xSemaphoreCreateMutex(); /  / 默认为1,
    不需要像之前那样二进制信号量的give
    
    / * 创建事件组 * /
    g_xEventCar = xEventGroupCreate();
    
    / 画出多个路标 /
    for(i = 0; i < 3; i +  + )
    {
        / * 绘制多条道路 * /
        for (j = 0; j < 8; j +  + )
        {
            draw_bitmap(16 * j, 16 + (17 * i), roadMarking, 8, 1, NOINVERT, 0);
            draw_flushArea(16 * j, 16 + (17 * i), 8, 1);
        }
    / * 绘制汽车 * /
    draw_bitmap(g_cars[i].x, g_cars[i].y, carImg, 15, 16, NOINVERT,
    0);
    draw_flushArea(g_cars[i].x, g_cars[i].y, 15, 16);
}

/  / 创建任务, 每个汽车对应一个任务
xTaskCreate(Car1Task, "Car1", 128, &g_cars[0], osPriorityNormal,
NULL);
xTaskCreate(Car2Task, "Car2", 128, &g_cars[1], osPriorityNormal +
2, NULL);
xTaskCreate(Car3Task, "Car3", 128, &g_cars[2], osPriorityNormal +
3, NULL);
}
```
之后在小车1的任务当中更改, 设置事件组, 去除获得信号量部分

其中用到的函数如下

可以设置事件组的某个位、某些位，使用的函数有2个：

在任务中使用xEventGroupSetBits()

在ISR中使用xEventGroupSetBitsFromISR()

有一个或多个任务在等待事件，如果这些事件符合这些任务的期望，那么任务还会被唤醒。

函数原型如下：
```c
/ * 设置事件组中的位
* xEventGroup: 哪个事件组
* uxBitsToSet: 设置哪些位?
* 如果uxBitsToSet的bitX, bitY为1, 那么事件组中的bitX, bitY被设置为1
* 可以用来设置多个位，比如 0x15 就表示设置bit4, bit2, bit0
* 返回值: 返回原来的事件值(没什么意义,
因为很可能已经被其他任务修改了)
* /
EventBits_t xEventGroupSetBits( EventGroupHandle_t xEventGroup,const
EventBits_t uxBitsToSet ); / * 设置事件组中的位
* xEventGroup: 哪个事件组
* uxBitsToSet: 设置哪些位?
* 如果uxBitsToSet的bitX, bitY为1, 那么事件组中的bitX, bitY被设置为1
* 可以用来设置多个位，比如 0x15 就表示设置bit4, bit2, bit0
* pxHigherPriorityTaskWoken: 有没有导致更高优先级的任务进入就绪态?
pdTRUE - 有, pdFALSE - 没有
* 返回值: pdPASS - 成功, pdFALSE - 失败
* /
BaseType_t xEventGroupSetBitsFromISR( EventGroupHandle_t
xEventGroup,const EventBits_t uxBitsToSet,
BaseType_t * pxHigherPriorityTaskWoken );
```
值得注意的是，ISR中的函数，比如队列函数xQueueSendToBackFromISR、信号量函数xSemaphoreGiveFromISR，它们会唤醒某个任务，最多只会唤醒1个任务。

但是设置事件组时，有可能导致多个任务被唤醒，这会带来很大的不确定性。所以xEventGroupSetBitsFromISR函数不是直接去设置事件组，而是给一个FreeRTOS后台任务(daemon
task)发送队列数据，由这个任务来设置事件组。

如果后台任务的优先级比当前被中断的任务优先级高，xEventGroupSetBitsFromISR会设置pxHigherPriorityTaskWoken为pdTRUE。

如果daemon
task成功地把队列数据发送给了后台任务，那么xEventGroupSetBitsFromISR的返回值就是pdPASS

更改后的代码:
```c
static void Car1Task(void params)
{
    car pcar = params;
    ir_data idata; /  / input_data
    
    / * 创建自己的队列 * /
    QueueHandle_t xQueueIR = xQueueCreate(10, sizeof(ir_data));
    
    / * 注册队列 * /
    RegisterQueueHandle(xQueueIR);
    
    while (1)
    {
        /  /  / *读取按键值 : 读队列* /
        /  / xQueueReceive(xQueueIR, &idata, portMAX_DELAY);
        /  /
        /  /  / * 控制汽车往右移动 * /
        /  / if (idata.val =  = pcar - >control_key)
        {
            if (pcar - >x < g_xres - CAR_LENGTH)
            {
                / * 隐藏汽车 * /
                HideCar(pcar);
                
                / * 调整位置 * /
                pcar - >x +  = 1;
                if (pcar - >x > g_xres - CAR_LENGTH)
                {
                    pcar - >x = g_xres - CAR_LENGTH;
                }
            
            / * 重新显示汽车 * /
            ShowCar(pcar);
            vTaskDelay(50);
            
            if (pcar - >x =  = g_xres - CAR_LENGTH)
            {
                
                / * 设置事件组 : bit0 * /
                xEventGroupSetBits(g_xEventCar, (1 << 0));
                
                / * 结束任务 * /
                vTaskDelete(NULL);
            }
    }
}
}
}
```
小车1自动向前走到达终点设置事件

下面我们将小车2改为等待事件

使用xEventGroupWaitBits来等待事件，可以等待某一位、某些位中的任意一个，也可以等待多位；等到期望的事件后，还可以清除某些位。

函数原型如下：
```c
EventBits_t xEventGroupWaitBits( EventGroupHandle_t xEventGroup,const
EventBits_t uxBitsToWaitFor,const BaseType_t xClearOnExit,const
BaseType_t xWaitForAllBits,
TickType_t xTicksToWait );
```
先引入一个概念：unblock
condition。一个任务在等待事件发生时，它处于阻塞状态；当期望的时间发生时，这个状态就叫"unblock
condition"，非阻塞条件，或称为"非阻塞条件成立"；当"非阻塞条件成立"后，该任务就可以变为就绪态。

函数参数说明列表如下：

![](media/media/image79.png)

记得使能中断
```c
#define MPU6050_INT_PIN_CFG 0x37
#define MPU6050_INT_ENABLE 0x38


/
* 函数名称： EXTI9_5_IRQHandler
* 功能描述： MPU6050中断函数
* 输入参数： params - 未使用
* 输出参数： 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2025 / 08 / 26 V1.0 水水水 创建
* /
void MPU6050_IRQ_Callback(void)
{
    / * 设置事件组 : bit0 * /
    xEventGroupSetBitsFromISR(g_xEventMPU6050, 0x01, NULL);
}
```
```c
/ * Copyright (s) 2019 深圳百问网科技有限公司
* All rights reserved
*
* 文件名称：driver_irq.c
* 摘要：
*
* 修改历史 版本号 Author 修改内容
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* 2023.08.05 v01 百问科技 创建文件
* -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -
* /

#include "driver_timer.h"
#include "stm32f1xx_hal.h"
#include "tim.h"

extern void IRReceiver_IRQ_Callback(void);
extern void RotaryEncoder_IRQ_Callback(void);
extern void MPU6050_IRQ_Callback(void);

/
* 函数名称： HAL_GPIO_EXTI_Callback
* 功能描述： 外部中断的中断回调函数
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 无
* 修改日期： 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 04 V1.0 韦东山 创建
* /
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    switch (GPIO_Pin)
    {
        case GPIO_PIN_10:
        {
            IRReceiver_IRQ_Callback();
            break;
        }
    
    case GPIO_PIN_12:
    {
        RotaryEncoder_IRQ_Callback();
        break;
    }


case GPIO_PIN_5:
{
    MPU6050_IRQ_Callback();
    break;
}

default:
{
    break;
}
}
}
```
#### 现象

烧录之后应该就可以通过事件组美美玩挡球板游戏了

## 任务通知

### 任务通知本质

事件组, 互斥量, 信号量, 队列在我们使用过程当中都需要创建队列, 事件组,
互斥量和信号量, 借助这些结构进行通信, 但实际上,
任务并不知道是谁和自己在通信, 任务只是在与对应的事件组,
信号量和队列在通信

而任务通知则是两个任务之间直接通信(通过任务的通知状态和通知值)

通知状态有下面三种

taskNOT_WAITING_NOTIFICATION：任务没有在等待通知

taskWAITING_NOTIFICATION：任务在等待通知

taskNOTIFICATION_RECEIVED：任务接收到了通知，也被称为pending(有数据了，待处理)
```c
##define taskNOT_WAITING_NOTIFICATION ( ( uint8_t ) 0 ) / *
也是初始状态 * /
##define taskWAITING_NOTIFICATION ( ( uint8_t ) 1 )
##define taskNOTIFICATION_RECEIVED ( ( uint8_t ) 2 )
```
任务通知的优势：

效率更高：使用任务通知来发送事件、数据给某个任务时，效率更高。比队列、信号量、事件组都有大的优势。

更节省内存：使用其他方法时都要先创建对应的结构体，使用任务通知时无需额外创建结构体。

下面举一个例子来讲解任务通知

一开始，小红正戴着耳机专心听音乐，完全沉浸在自己的世界里
(任务正在运行自己的代码，状态是 taskNOT_WAITING_NOTIFICATION)。

过了一会儿，一首歌放完了，她摘下耳机，心想："和小明约好了的，该等他的消息了。"
于是她把手机放在桌上，决定在收到消息前，什么都不做，就专心等待
(任务决定调用 ulTaskNotifyTake() 函数，准备进入等待状态)。

此刻，她进入了等待模式，眼睛盯着屏幕，不理会其他任何事情
(任务已进入**阻塞状态**，状态变为 taskWAITING_NOTIFICATION)。

突然，手机"叮咚"一声亮起，屏幕上弹出小明的消息："我到你楼下了！"
(另一个任务调用了 xTaskNotifyGive()，发送了通知)。

小红的等待瞬间结束，她看到了这条消息，立刻从沙发上跳了起来，准备出门
(任务接收到了通知，内部状态变为
taskNOTIFICATION_RECEIVED，并从**阻塞态**被唤醒，进入**就绪态**，准备继续执行)。

再提一句

当小红正戴着耳机听歌时 (任务处于 taskNOT_WAITING_NOTIFICATION
状态，没在等)，小明突然连发了好几条消息 (另一个任务连续调用
xTaskNotifyGive())。

这些消息并不会打扰到她，而是会像手机锁屏上的"**未读消息**"角标一样，**数字不断累加**
(任务的通知值在后台被更新和覆盖，如果是计数，则会累加)。

等她听完歌拿起手机一看 (任务终于调用 ulTaskNotifyTake()
来获取通知)，就能立刻知道------"哦，这期间他总共找了我好几次。"

下面是框图图示, A发通知, B接通知

![](media/media/image80.jpeg)

下面是一些函数

使用任务通知，可以实现轻量级的队列(长度为1)、邮箱(覆盖的队列)、计数型信号量、二进制信号量、事件组

任务通知有2套函数，简化版、专业版，列表如下：

简化版函数的使用比较简单，它实际上也是使用专业版函数实现的

专业版函数支持很多参数，可以实现很多功能

![](media/media/image88.png)

守护任务的优先级为：configTIMER_TASK_PRIORITY；定时器命令队列的长度为configTIMER_QUEUE_LENGTH。

通过这个Timer任务, 我们可以在读队列之后处理超时的定时器,
从而调用处理的函数和逻辑

这一招使得任务运行于任务上下文, 但这个Timet任务的优先级必须足够高,
从而保证运行的频率和完整性

如果你的优先级是在没招,
那也只能让其他任务阻塞一会再执行(或者事件驱动任务),
否则软件定时器就会出错

在FreeRTOS当中, 采取的是第二种方法, 同时, 从上图也可以得知,
用户启动删除等对定时器的操作都是由这个队列来完成

下面来看这个main.c当中启动任务调度器之后
```c
osKernelStart();
```
```c
osStatus_t osKernelStart (void) {
    osStatus_t stat;
    
    if (IS_IRQ()) {
        stat = osErrorISR;
    }
else {
    if (KernelState =  = osKernelReady) {
        / * Ensure SVC priority is at the reset value * /
        SVC_Setup();
        / * Change state to enable IRQ masking check * /
        KernelState = osKernelRunning;
        / * Start the kernel scheduler * /
        vTaskStartScheduler();
        stat = osOK;
    } else {
    stat = osError;
}
}

return (stat);
}
```
```c
void vTaskStartScheduler( void )
{
    BaseType_t xReturn;
    
    / * Add the idle task at the lowest priority. * /
    #if( configSUPPORT_STATIC_ALLOCATION =  = 1 )
    {
        StaticTask_t pxIdleTaskTCBBuffer = NULL;
        StackType_t pxIdleTaskStackBuffer = NULL;
        uint32_t ulIdleTaskStackSize;
        
        / * The Idle task is created using user provided RAM - obtain the
        address of the RAM then create the idle task. * /
        vApplicationGetIdleTaskMemory( &pxIdleTaskTCBBuffer, &pxIdleTaskStackBuffer, &ulIdleTaskStackSize );
        xIdleTaskHandle = xTaskCreateStatic( prvIdleTask,
        configIDLE_TASK_NAME,
        ulIdleTaskStackSize,
        ( void  ) NULL, / *lint !e961. The cast is not redundant for all compilers. * /
        portPRIVILEGE_BIT, / * In effect ( tskIDLE_PRIORITY | portPRIVILEGE_BIT ), but tskIDLE_PRIORITY is zero. * /
        pxIdleTaskStackBuffer,
        pxIdleTaskTCBBuffer ); / *lint !e961 MISRA exception, justified as it is not a redundant explicit cast to all supported
        compilers. * /
        
        if( xIdleTaskHandle != NULL )
        {
            xReturn = pdPASS;
        }
    else
    {
        xReturn = pdFAIL;
    }
}
#else
{
    / * The Idle task is being created using dynamically allocated RAM. * /
    xReturn = xTaskCreate( prvIdleTask,
    configIDLE_TASK_NAME,
    configMINIMAL_STACK_SIZE,
    ( void  ) NULL,
    portPRIVILEGE_BIT, / * In effect ( tskIDLE_PRIORITY | portPRIVILEGE_BIT ), but tskIDLE_PRIORITY is zero. * /
    &xIdleTaskHandle ); / *lint !e961 MISRA exception, justified as it is not a redundant explicit cast to all supported
    compilers. * /
}
#endif / * configSUPPORT_STATIC_ALLOCATION * /

#if ( configUSE_TIMERS =  = 1 )
{
    if( xReturn =  = pdPASS )
    {
        xReturn = xTimerCreateTimerTask();
    }
else
{
    mtCOVERAGE_TEST_MARKER();
}
}
#endif / * configUSE_TIMERS * /

if( xReturn =  = pdPASS )
{
    / * freertos_tasks_c_additions_init() should only be called if the user
    definable macro FREERTOS_TASKS_C_ADDITIONS_INIT() is defined, as that is
    the only macro called by the function. * /
    #ifdef FREERTOS_TASKS_C_ADDITIONS_INIT
    {
        freertos_tasks_c_additions_init();
    }
#endif

/ * Interrupts are turned off here, to ensure a tick does not occur
before or during the call to xPortStartScheduler(). The stacks of
the created tasks contain a status word with interrupts switched on
so interrupts will automatically get re - enabled when the first task
starts to run. * /
portDISABLE_INTERRUPTS();

#if ( configUSE_NEWLIB_REENTRANT =  = 1 )
{
    / * Switch Newlib's _impure_ptr variable to point to the _reent
    structure specific to the task that will run first.
    See the third party link http: /  / www.nadler.com / embedded / newlibAndFreeRTOS.html
    for additional information. * /
    _impure_ptr = &( pxCurrentTCB - >xNewLib_reent );
}
#endif / * configUSE_NEWLIB_REENTRANT * /

xNextTaskUnblockTime = portMAX_DELAY;
xSchedulerRunning = pdTRUE;
xTickCount = ( TickType_t ) configINITIAL_TICK_COUNT;

/ * If configGENERATE_RUN_TIME_STATS is defined then the following
macro must be defined to configure the timer / counter used to generate
the run time counter time base. NOTE: If configGENERATE_RUN_TIME_STATS
is set to 0 and the following line fails to build then ensure you do not
have portCONFIGURE_TIMER_FOR_RUN_TIME_STATS() defined in your
FreeRTOSConfig.h file. * /
portCONFIGURE_TIMER_FOR_RUN_TIME_STATS();

traceTASK_SWITCHED_IN();

/ * Setting up the timer tick is hardware specific and thus in the
portable interface. * /
if( xPortStartScheduler() != pdFALSE )
{
    / * Should not reach here as if the scheduler is running the
    function will not return. * /
}
else
{
    / * Should only reach here if a task calls xTaskEndScheduler(). * /
}
}
else
{
    / * This line will only be reached if the kernel could not be started,
    because there was not enough FreeRTOS heap to create the idle task
    or the timer task. * /
    configASSERT( xReturn != errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY );
}

/ * Prevent compiler warnings if INCLUDE_xTaskGetIdleTaskHandle is set to 0,
meaning xIdleTaskHandle is not used anywhere else. * /
( void ) xIdleTaskHandle;
}
/ * -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
```
除了有之前的空闲任务,
还有***[xTimerCreateTimerTask()]{.underline}***这个Timer任务.
```c
演示
C +  +
timers.c
BaseType_t xTimerCreateTimerTask( void )
{
    BaseType_t xReturn = pdFAIL;
    
    / * This function is called when the scheduler is started if
    configUSE_TIMERS is set to 1. Check that the infrastructure used by
    the
    timer service task has been created / initialised. If timers have
    already
    been created then the initialisation will already have been performed.
    * /
    prvCheckForValidListAndQueue();
    
    if( xTimerQueue != NULL )
    {
        #if( configSUPPORT_STATIC_ALLOCATION =  = 1 )
        {
            StaticTask_t pxTimerTaskTCBBuffer = NULL;
            StackType_t pxTimerTaskStackBuffer = NULL;
            uint32_t ulTimerTaskStackSize;
            
            vApplicationGetTimerTaskMemory( &pxTimerTaskTCBBuffer,
            &pxTimerTaskStackBuffer, &ulTimerTaskStackSize );
            xTimerTaskHandle = xTaskCreateStatic( prvTimerTask,
            configTIMER_SERVICE_TASK_NAME,
            ulTimerTaskStackSize,
            NULL,
            ( ( UBaseType_t ) configTIMER_TASK_PRIORITY ) | portPRIVILEGE_BIT,
            pxTimerTaskStackBuffer,
            pxTimerTaskTCBBuffer );
            
            if( xTimerTaskHandle != NULL )
            {
                xReturn = pdPASS;
            }
    }
#else
{
    xReturn = xTaskCreate( prvTimerTask,
    configTIMER_SERVICE_TASK_NAME,
    configTIMER_TASK_STACK_DEPTH,
    NULL,
    ( ( UBaseType_t ) configTIMER_TASK_PRIORITY ) | portPRIVILEGE_BIT,
    &xTimerTaskHandle );
}
#endif / * configSUPPORT_STATIC_ALLOCATION * /
}
else
{
    mtCOVERAGE_TEST_MARKER();
}

configASSERT( xReturn );
return xReturn;
}
```
其中可以看到它们的优先级统一为***[( ( UBaseType_t )
configTIMER_TASK_PRIORITY ) | portPRIVILEGE_BIT]{.underline}***
这个c***[onfigTIMER_TASK_PRIORITY]{.underline}***大有来头啊,
另一个或的也不赖
```c
/ * Software timer definitions. * /
#define configUSE_TIMERS 1
#define configTIMER_TASK_PRIORITY ( 2 )
#define configTIMER_QUEUE_LENGTH 10
#define configTIMER_TASK_STACK_DEPTH 256
```
```c
#ifndef portPRIVILEGE_BIT
#define portPRIVILEGE_BIT ( ( UBaseType_t ) 0x00 )
#endif
```
显然这个优先级可以说是当狗了, 所以如果一直运行用户的默认优先级任务,
那软件定时器就轧钢了

当然你也可以在CubeMX当中配置这个优先级

![](media/media/image89.png)

### 示例工程

工程文件:28_timer_game_sound

本次工程文件由27_tasknotification_car_game改进而来,
目的是给挡球板游戏增加音效, 持续发声通过软件定时器实现
```c
/ **
* @brief FreeRTOS initialization
* @param None
* @retval None
* /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    
    /  / 初始化互斥量
    g_xI2CMutex = xSemaphoreCreateMutex();
    PassiveBuzzer_Init();
    LCD_Init();
    LCD_Clear();
    
    MPU6050_Init();
    IRReceiver_Init();
    RotaryEncoder_Init();
    LCD_PrintString(0, 0, "Starting");
    
    ......
    
    /  / 创建游戏任务
    
    xTaskCreate(game1_task, "GameTask", 128, NULL, osPriorityNormal,
    NULL);
    
    ......
    
}
```
新建***[beep.c]{.underline}***与***[beep.h]{.underline}***文件

创建软件定时器
```c
/ * 使用动态分配内存的方法创建定时器
* pcTimerName:定时器名字, 用处不大, 尽在调试时用到
* xTimerPeriodInTicks: 周期, 以Tick为单位
* uxAutoReload: 类型, pdTRUE表示自动加载, pdFALSE表示一次性
* pvTimerID: 回调函数可以使用此参数, 比如分辨是哪个定时器
* pxCallbackFunction: 回调函数
* 返回值: 成功则返回TimerHandle_t, 否则返回NULL
* /
TimerHandle_t xTimerCreate( const char  const pcTimerName,
const TickType_t xTimerPeriodInTicks,const UBaseType_t
uxAutoReload,void  const pvTimerID,
TimerCallbackFunction_t pxCallbackFunction ); / *
使用静态分配内存的方法创建定时器
* pcTimerName:定时器名字, 用处不大, 尽在调试时用到
* xTimerPeriodInTicks: 周期, 以Tick为单位
* uxAutoReload: 类型, pdTRUE表示自动加载, pdFALSE表示一次性
* pvTimerID: 回调函数可以使用此参数, 比如分辨是哪个定时器
* pxCallbackFunction: 回调函数
* pxTimerBuffer: 传入一个StaticTimer_t结构体, 将在上面构造定时器
* 返回值: 成功则返回TimerHandle_t, 否则返回NULL
* /
TimerHandle_t xTimerCreateStatic(const char  const pcTimerName,
TickType_t xTimerPeriodInTicks,
UBaseType_t uxAutoReload,void  pvTimerID,
TimerCallbackFunction_t pxCallbackFunction,
StaticTimer_t *pxTimerBuffer );
```
回调函数的类型是：
```c
void ATimerCallback( TimerHandle_t xTimer );

typedef void (* TimerCallbackFunction_t)( TimerHandle_t xTimer );
```
```c
#include "beep.h"

static TimerHandle_t g_TimerSound;

void GameSoundTimer(TimerHandle_t xTimer)
{
    PassiveBuzzer_Control(0);
}

void buzzer_init(void)
{
    /  / 初始话蜂鸣器
    PassiveBuzzer_Init();
    
    /  / 创建软件定时器
    g_TimerSound = xTimerCreate("GameSound", 200, pdFALSE, NULL,
    GameSoundTimer);
}

void buzzer_buzz(int freq, int time_ms)
{
    PassiveBuzzer_Set_Freq_Duty(freq, 50);
    
    / * 启动定时器 * /
    xTimerChangePeriod(g_TimerSound, time_ms, 0);
    
}
```
```c
演示
C +  +
beep.h
#ifndef __BEEP_H
#define __BEEP_H

#include <stdlib.h>
#include <stdio.h>

#include "cmsis_os.h"
#include "FreeRTOS.h" /  / ARM.FreeRTOS::RTOS:Core
#include "task.h" /  / ARM.FreeRTOS::RTOS:Core
#include "event_groups.h" /  / ARM.FreeRTOS::RTOS:Event Groups
#include "semphr.h" /  / ARM.FreeRTOS::RTOS:Core

#include "draw.h"
#include "resources.h"
#include "driver_passive_buzzer.h"

void GameSoundTimer(TimerHandle_t xTimer);
void buzzer_init(void);
void buzzer_buzz(int freq, int time_ms);

#endif
```
注意:

定时器的状态转换图

![](media/media/image90.png)

启动定时器就是设置它的状态为运行态(Running、Active)。

停止定时器就是设置它的状态为冬眠(Dormant)，让它不能运行。

涉及的函数原型如下：
```c
/ * 启动定时器
* xTimer: 哪个定时器
* xTicksToWait: 超时时间
* 返回值: pdFAIL表示"启动命令"在xTicksToWait个Tick内无法写入队列
* pdPASS表示成功
* /
BaseType_t xTimerStart( TimerHandle_t xTimer, TickType_t xTicksToWait
); / * 启动定时器(ISR版本)
* xTimer: 哪个定时器
* pxHigherPriorityTaskWoken: 向队列发出命令使得守护任务被唤醒,
* 如果守护任务的优先级比当前任务的高,
* 则"*pxHigherPriorityTaskWoken = pdTRUE",
* 表示需要进行任务调度
* 返回值: pdFAIL表示"启动命令"无法写入队列
* pdPASS表示成功
* /
BaseType_t xTimerStartFromISR( TimerHandle_t xTimer,
BaseType_t pxHigherPriorityTaskWoken ); / * 停止定时器
* xTimer: 哪个定时器
* xTicksToWait: 超时时间
* 返回值: pdFAIL表示"停止命令"在xTicksToWait个Tick内无法写入队列
* pdPASS表示成功
* /
BaseType_t xTimerStop( TimerHandle_t xTimer, TickType_t xTicksToWait
); / * 停止定时器(ISR版本)
* xTimer: 哪个定时器
* pxHigherPriorityTaskWoken: 向队列发出命令使得守护任务被唤醒,
* 如果守护任务的优先级比当前任务的高,
* 则"*pxHigherPriorityTaskWoken = pdTRUE",
* 表示需要进行任务调度
* 返回值: pdFAIL表示"停止命令"无法写入队列
* pdPASS表示成功
* /
BaseType_t xTimerStopFromISR( TimerHandle_t xTimer,
BaseType_t pxHigherPriorityTaskWoken );
```
注意，这些函数的 xTicksToWait
表示的是，把命令写入命令队列的超时时间。命令队列可能已经满了，无法马上把命令写入队列里，可以等待一会。

xTicksToWait 不是定时器本身的超时时间，不是定时器本身的"周期"。

创建定时器时，设置了它的周期(period)。xTimerStart()
函数是用来启动定时器。假设调用 xTimerStart()
的时刻是tX，定时器的周期是n，那么在*tX+n*时刻定时器的回调函数被调用。

如果定时器已经被启动，但是它的函数尚未被执行，再次执行 xTimerStart()
函数相当于执行 xTimerReset() ，重新设定它的启动时间

从定时器的状态转换图可以知道，使用 xTimerReset()
函数可以让定时器的状态从冬眠态转换为运行态，相当于使用 xTimerStart()
函数。

如果定时器已经处于运行态，使用 xTimerReset()
函数就相当于重新确定超时时间。假设调用 xTimerReset()
的时刻是tX，定时器的周期是n，那么*tX+n*就是重新确定的超时时间。

复位函数的原型如下：
```c
/ * 复位定时器
* xTimer: 哪个定时器
* xTicksToWait: 超时时间
* 返回值: pdFAIL表示"复位命令"在xTicksToWait个Tick内无法写入队列
* pdPASS表示成功
* /
BaseType_t xTimerReset( TimerHandle_t xTimer, TickType_t xTicksToWait
); / * 复位定时器(ISR版本)
* xTimer: 哪个定时器
* pxHigherPriorityTaskWoken: 向队列发出命令使得守护任务被唤醒,
* 如果守护任务的优先级比当前任务的高,
* 则"*pxHigherPriorityTaskWoken = pdTRUE",
* 表示需要进行任务调度
* 返回值: pdFAIL表示"停止命令"无法写入队列
* pdPASS表示成功
* /
BaseType_t xTimerResetFromISR( TimerHandle_t xTimer,
BaseType_t *pxHigherPriorityTaskWoken );
```
从定时器的状态转换图可以知道，使用 xTimerChangePeriod()
函数，处理能修改它的周期外，还可以让定时器的状态从冬眠态转换为运行态。

修改定时器的周期时，会使用新的周期重新计算它的超时时间。假设调用
xTimerChangePeriod()
函数的时间tX，新的周期是n，则*tX+n*就是新的超时时间。

相关函数的原型如下：
```c
/ * 修改定时器的周期
* xTimer: 哪个定时器
* xNewPeriod: 新周期
* xTicksToWait: 超时时间, 命令写入队列的超时时间
* 返回值:
pdFAIL表示"修改周期命令"在xTicksToWait个Tick内无法写入队列
* pdPASS表示成功
* /
BaseType_t xTimerChangePeriod( TimerHandle_t xTimer,
TickType_t xNewPeriod,
TickType_t xTicksToWait ); / * 修改定时器的周期
* xTimer: 哪个定时器
* xNewPeriod: 新周期
* pxHigherPriorityTaskWoken: 向队列发出命令使得守护任务被唤醒,
* 如果守护任务的优先级比当前任务的高,
* 则"*pxHigherPriorityTaskWoken = pdTRUE",
* 表示需要进行任务调度
* 返回值:
pdFAIL表示"修改周期命令"在xTicksToWait个Tick内无法写入队列
* pdPASS表示成功
* /
BaseType_t xTimerChangePeriodFromISR( TimerHandle_t xTimer,
TickType_t xNewPeriod,
BaseType_t *pxHigherPriorityTaskWoken );
```
下面更改一下游戏函数
```c
......

#include "beep.h"

......

void game1_draw()
{
    
    ......
    if(!blocks[idx] && ballX > = x  4 && ballX < (x  4) + 4 && ballY
    > = (y  4) + 8 && ballY < (y  4) + 8 + 4)
    {
        buzzer_buzz(2000, 100); /  / 100ask todo buzzer_buzz(2000, 100); /  /
        100ask todo
        /  / led_flash(LED_GREEN, 50, 255); /  / 100ask todo
        blocks[idx] = true;
        
        /  / hide block
        draw_bitmap(x * 4, (y * 4) + 8, clearImg, 3, 8, NOINVERT, 0);
        draw_flushArea(x * 4, (y * 4) + 8, 3, 8);
        blockCollide = true;
        score +  + ;
    }
idx +  + ;
}
}


/  / Side wall collision
if(ballX > g_xres - 2)
{
    if(ballX > 240)
    ball.x = 0;
    else
    ball.x = g_xres - 2;
    ball.velX =  - ball.velX;
}
if(ballX < 0)
{
    ball.x = 0;
    ball.velX =  - ball.velX;
}

/  / Platform collision
bool platformCollision = false;
if(!gameEnded && ballY > = g_yres - PLATFORM_HEIGHT - 2 && ballY < 240
&& ballX > = platformX && ballX < = platformX + PLATFORM_WIDTH)
{
    platformCollision = true;
    buzzer_buzz(5000, 200); /  / 100ask todo
    ball.y = g_yres - PLATFORM_HEIGHT - 2;
    if(ball.velY > 0)
    ball.velY =  - ball.velY;
    ball.velX = ((float)rand() / (RAND_MAX / 2)) - 1; /  /  - 1.0 to 1.0
}

/  / Top / bottom wall collision
if(!gameEnded && !platformCollision && (ballY > g_yres - 2 ||
blockCollide))
{
    if(ballY > 240)
    {
        buzzer_buzz(2500, 200); /  / 100ask todo
        ball.y = 0;
    }
else if(!blockCollide)
{
    buzzer_buzz(2000, 200); /  / 100ask todo
    ball.y = g_yres - 1;
    lives -  - ;
}
ball.velY *=  - 1;
}

......

}
```
编译运行, 烧录即可

#### 现象

挡球板游戏运行, 会有惊喜音效

## 中断管理

在之前的队列, 事件组, 信号量/互斥量当中, 我们都可以发现,
这里面有两套API函数,

![](media/media/image92.jpeg)

那么如何优化调度呢?如下所示

FreeRTOS的ISR函数中，使用两个宏进行任务切换：
```c
portEND_SWITCHING_ISR( xHigherPriorityTaskWoken );
或
portYIELD_FROM_ISR( xHigherPriorityTaskWoken );
```
这两个宏做的事情是完全一样的，在老版本的FreeRTOS中，

portEND_SWITCHING_ISR 使用汇编实现

portYIELD_FROM_ISR 使用C语言实现

新版本都统一使用portYIELD_FROM_ISR。

使用示例如下：
```c
void XXX_ISR()
{
    int i;
    BaseType_t xHigherPriorityTaskWoken = pdFALSE; /  / 默认不唤醒
    
    for (i = 0; i < N; i +  + )
    {
        xQueueSendToBackFromISR(..., &xHigherPriorityTaskWoken);
        / * 被多次调用 * /
    }

/ * 最后再决定是否进行任务切换
* xHigherPriorityTaskWoken为pdTRUE时才切换
* /

portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```
更改中断处理逻辑后的代码:
```c
......

/
* 函数名称： DispatchKey
* 功能描述： 赛车密钥分发
* 输入参数： ir_data pidata
输出参数： 无
* 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2025 / 08 / 22 V1.0 水水水 创建
/
static void vDispatchKey(ir_data pidata)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE; /  / 默认不唤醒
    /  / extern QueueHandle_t g_xQueueCar1;
    /  / extern QueueHandle_t g_xQueueCar2;
    /  / extern QueueHandle_t g_xQueueCar3;
    /  /
    /  / xQueueSendFromISR(g_xQueueCar1, pidata, NULL);
    /  / xQueueSendFromISR(g_xQueueCar2, pidata, NULL);
    /  / xQueueSendFromISR(g_xQueueCar3, pidata, NULL);
    
    uint8_t i = 0;
    for (i = 0; i < g_queue_cnt; i +  + )
    {
        xQueueSendFromISR(g_xQueus[i], pidata, &xHigherPriorityTaskWoken);
    }

portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

......
```
```c
......

/
* 函数名称： RotaryEncoder_IRQ_Callback
* 功能描述： 旋转编码器的中断回调函数
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 无
* 修改日期： 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 04 V1.0 韦东山 创建
* /
void RotaryEncoder_IRQ_Callback(void)
{
    uint64_t time;
    static uint64_t pre_time = 0;
    rotary_data rdata;
    BaseType_t xHigherPriorityTaskWoken = pdFALSE; /  / 默认不唤醒
    
    / * 1. 记录中断发生的时刻 * /
    time = system_get_ns();
    
    / * 上升沿触发: 必定是高电平
    * 防抖
    * /
    mdelay(2);
    if (!RotaryEncoder_Get_S1())
    return;
    
    / * S1上升沿触发中断
    * S2为0表示逆时针转, 为1表示顺时针转
    * /
    g_speed = (uint64_t)1000000000 / (time - pre_time);
    if (RotaryEncoder_Get_S2())
    {
        g_count +  + ;
    }
else
{
    g_count -  - ;
    g_speed = 0 - g_speed;
}
pre_time = time;

/  / 写队列
rdata.cnt = g_count;
rdata.speed = g_speed;
xQueueSendFromISR(g_xQueueRotary, &rdata, &xHigherPriorityTaskWoken);

portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}


......
```
```c
......

/
* 函数名称： EXTI9_5_IRQHandler
* 功能描述： MPU6050中断函数
* 输入参数： params - 未使用
* 输出参数： 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 09 / 05 V1.0 水水水 创建
* /
void MPU6050_IRQ_Callback(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE; /  / 默认不唤醒
    
    / * 设置事件组 : bit0 * /
    xEventGroupSetBitsFromISR(g_xEventMPU6050, 0x01, &xHigherPriorityTaskWoken);
    
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

......
```
现象几乎没什么变化, 因为这些时间对于我们人来说太快了

## 资源管理

### 互斥操作的本质

之前的例子当中讲解过一个有问题的例子(互斥那个), 如下图所示

![](media/media/image93.png)

我们后面使用了互斥量来解决互斥访问这个问题

但是, 它是怎么实现互斥的呢?其他的比如队列, 信号量, 事件组,
任务通知是怎么实现互斥地访问的?

队列是环形缓冲区, 任务怎么去互斥地访问缓冲区?

信号量和互斥量都是cnt计数值, 任务怎么互斥地访问变量?

事件组是一个int整数, 那多个任务怎么互斥地访问它?

任务通知有通知值和通知状态, 那怎么让任务互斥地访问值和状态?

这些问题可以归结为一个:

如何互斥地访问?

要独占式地访问临界资源，有3种方法：

公平竞争：比如使用互斥量，谁先获得互斥量谁就访问临界资源，这部分内容前面讲过。

谁要跟我抢，我就灭掉谁：

中断要跟我抢？我屏蔽中断

其他任务要跟我抢？我禁止调度器，不运行任务切换

下面的例子还是以这张图来讲解:

![](media/media/image93.png)

这里A, B两个任务访问OLED这个资源, 我们按照上面的说法, 开关调度器就可以了

下面的时运行时间轴以及关键变量的变化

![](media/media/image94.png)

那如果处理A, B任务还有一个中断函数也会调用OLED外设呢?
还是按照之前给的思路, 有中断就屏蔽中断

这里只有3步骤了

但是1要在关闭中断之前保存中断状态, 在访问玩资源最后恢复中断状态和中断

屏蔽中断有两套宏：任务中使用、ISR中使用：

任务中使用：taskENTER_CRITICA()/taskEXIT_CRITICAL()

ISR中使用：taskENTER_CRITICAL_FROM_ISR()/taskEXIT_CRITICAL_FROM_ISR()

### 在任务中屏蔽中断

在任务中屏蔽中断的示例代码如下：
```c
/ * 在任务中，当前时刻中断是使能的
* 执行这句代码后，屏蔽中断
* /
taskENTER_CRITICAL();

/ * 访问临界资源 * /

/ * 重新使能中断 * /
taskEXIT_CRITICAL();
```
在 taskENTER_CRITICA()/taskEXIT_CRITICAL() 之间：

低优先级的中断被屏蔽了：优先级低于、等于
configMAX_SYSCALL_INTERRUPT_PRIORITY

高优先级的中断可以产生：优先级高于 configMAX_SYSCALL_INTERRUPT_PRIORITY

但是，这些中断ISR里，不允许使用FreeRTOS的API函数

任务调度依赖于中断、依赖于API函数，所以：这两段代码之间，不会有任务调度产生

这套 taskENTER_CRITICA()/taskEXIT_CRITICAL()
宏，是可以递归使用的，它的内部会记录嵌套的深度，只有嵌套深度变为0时，调用
taskEXIT_CRITICAL() 才会重新使能中断。

使用 taskENTER_CRITICA()/taskEXIT_CRITICAL()
来访问临界资源是很粗鲁的方法：

中断无法正常运行

任务调度无法进行

所以，之间的代码要尽可能快速地执行

### 在ISR中屏蔽中断

要使用含有"FROM_ISR"后缀的宏，示例代码如下：
```c
void vAnInterruptServiceRoutine( void ){ / * 用来记录当前中断是否使能
    * /
    UBaseType_t uxSavedInterruptStatus; / *
    在ISR中，当前时刻中断可能是使能的，也可能是禁止的
    * 所以要记录当前状态, 后面要恢复为原先的状态
    * 执行这句代码后，屏蔽中断
    * /
    uxSavedInterruptStatus = taskENTER_CRITICAL_FROM_ISR(); / * 访问临界资源
    * /  / * 恢复中断状态 * / taskEXIT_CRITICAL_FROM_ISR(
    uxSavedInterruptStatus ); / * 现在，当前ISR可以被更高优先级的中断打断了
    * / }
```
在 taskENTER_CRITICA_FROM_ISR()/taskEXIT_CRITICAL_FROM_ISR() 之间：

低优先级的中断被屏蔽了：优先级低于、等于
configMAX_SYSCALL_INTERRUPT_PRIORITY

高优先级的中断可以产生：优先级高于 configMAX_SYSCALL_INTERRUPT_PRIORITY

但是，这些中断ISR里，不允许使用FreeRTOS的API函数

任务调度依赖于中断、依赖于API函数，所以：这两段代码之间，不会有任务调度产生

下面则是一些有关如何暂停调度器的函数

如果有别的任务来跟你竞争临界资源，你可以把中断关掉：这当然可以禁止别的任务运行，但是这代价太大了。它会影响到中断的处理。

如果只是禁止别的任务来跟你竞争，不需要关中断，暂停调度器就可以了：在这期间，中断还是可以发生、处理。

使用这2个函数来暂停、恢复调度器：
```c
/ * 暂停调度器 * /
void vTaskSuspendAll( void );

/ * 恢复调度器
* 返回值: pdTRUE表示在暂定期间有更高优先级的任务就绪了
* 可以不理会这个返回值
* /
BaseType_t xTaskResumeAll( void );
```
示例代码如下：
```c
vTaskSuspendScheduler();

/ * 访问临界资源 * /
xTaskResumeScheduler();
```
这套 vTaskSuspendScheduler()/xTaskResumeScheduler()
宏，是可以递归使用的，它的内部会记录嵌套的深度，只有嵌套深度变为0时，调用
taskEXIT_CRITICAL() 才会重新使能中断。

我们看看暂停调度器的具体代码
```c
void vTaskSuspendAll( void )
{
    / * A critical section is not required as the variable is of type
    BaseType_t. Please read Richard Barry's reply in the following link to
    a
    post in the FreeRTOS support forum before reporting this as a bug! -
    http: /  / goo.gl / wu4acr * /
    
    / * portSOFRWARE_BARRIER() is only implemented for emulated / simulated
    ports that
    do not otherwise exhibit real time behaviour. * /
    portSOFTWARE_BARRIER();
    
    / * The scheduler is suspended if uxSchedulerSuspended is non - zero. An
    increment
    is used to allow calls to vTaskSuspendAll() to nest. * /
    +  + uxSchedulerSuspended;
    
    / * Enforces ordering for ports and optimised compilers that may
    otherwise place
    the above increment elsewhere. * /
    portMEMORY_BARRIER();
}
```
一旦你调用这个函数嵌套深度就不为0, 那就无法切换任务了

下面就是切换任务的函数
```c
void vTaskSwitchContext( void )
{
    if( uxSchedulerSuspended != ( UBaseType_t ) pdFALSE )
    {
        / * The scheduler is currently suspended - do not allow a context
        switch. * /
        xYieldPending = pdTRUE;
    }
else
{
    xYieldPending = pdFALSE;
    traceTASK_SWITCHED_OUT();
    
    #if ( configGENERATE_RUN_TIME_STATS =  = 1 )
    {
        #ifdef portALT_GET_RUN_TIME_COUNTER_VALUE
        portALT_GET_RUN_TIME_COUNTER_VALUE( ulTotalRunTime );
        #else
        ulTotalRunTime = portGET_RUN_TIME_COUNTER_VALUE();
        #endif
        
        / * Add the amount of time the task has been running to the
        accumulated time so far. The time the task started running was
        stored in ulTaskSwitchedInTime. Note that there is no overflow
        protection here so count values are only valid until the timer
        overflows. The guard against negative values is to protect
        against suspect run time stat counter implementations - which
        are provided by the application, not the kernel. * /
        if( ulTotalRunTime > ulTaskSwitchedInTime )
        {
            pxCurrentTCB - >ulRunTimeCounter +  = ( ulTotalRunTime -
            ulTaskSwitchedInTime );
        }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
ulTaskSwitchedInTime = ulTotalRunTime;
}
#endif / * configGENERATE_RUN_TIME_STATS * /

/ * Check for stack overflow, if configured. * /
taskCHECK_FOR_STACK_OVERFLOW();

/ * Before the currently running task is switched out, save its errno.
* /
#if( configUSE_POSIX_ERRNO =  = 1 )
{
    pxCurrentTCB - >iTaskErrno = FreeRTOS_errno;
}
#endif

/ * Select a new task to run using either the generic C or port
optimised asm code. * /
taskSELECT_HIGHEST_PRIORITY_TASK(); / *lint !e9079 void * is used as
this macro is used with timers and co - routines too. Alignment is known
to be fine as the type of the pointer stored and retrieved is the same.
* /
traceTASK_SWITCHED_IN();

/ * After the new task is switched in, update the global errno. * /
#if( configUSE_POSIX_ERRNO =  = 1 )
{
    FreeRTOS_errno = pxCurrentTCB - >iTaskErrno;
}
#endif

#if ( configUSE_NEWLIB_REENTRANT =  = 1 )
{
    / * Switch Newlib's _impure_ptr variable to point to the _reent
    structure specific to this task.
    See the third party link
    http: /  / www.nadler.com / embedded / newlibAndFreeRTOS.html
    for additional information. * /
    _impure_ptr = &( pxCurrentTCB - >xNewLib_reent );
}
#endif / * configUSE_NEWLIB_REENTRANT * /
}
}
```
这个时候, 我们再回到之前的问题

队列 信号量 互斥量等等是怎么回事

先看队列, 队列的写函数部分定义具体如下

![](media/media/image95.png)

其中有1个
```c
for( ;; )
{
    taskENTER_CRITICAL();
```
直接关闭了中断

再看看中断当中写队列的函数, 部分具体代码如下

![](media/media/image96.png)

其中
```c
uxSavedInterruptStatus = portSET_INTERRUPT_MASK_FROM_ISR();
```
保存了中断状态之后关闭中断

再看一下事件组:

这个是中断当中写事件组

![](media/media/image97.png)

可以看见是先写Timer队列, 之后唤醒TimerTask, 在任务当中写事件值

对于这个事件值因为不跟其他中断竞争, 所以可以关闭调度器, 无需关闭中断

再看看另一个非中断使用的API
```c
EventBits_t xEventGroupSetBits( EventGroupHandle_t xEventGroup, const
EventBits_t uxBitsToSet )
{
    ListItem_t pxListItem, pxNext;
    ListItem_t const pxListEnd;
    List_t const  pxList;
    EventBits_t uxBitsToClear = 0, uxBitsWaitedFor, uxControlBits;
    EventGroup_t *pxEventBits = xEventGroup;
    BaseType_t xMatchFound = pdFALSE;
    
    / * Check the user is not attempting to set the bits used by the
    kernel
    itself. * /
    configASSERT( xEventGroup );
    configASSERT( ( uxBitsToSet & eventEVENT_BITS_CONTROL_BYTES ) =  = 0 );
    
    pxList = &( pxEventBits - >xTasksWaitingForBits );
    pxListEnd = listGET_END_MARKER( pxList ); / *lint !e826 !e740 !e9087
    The mini list structure is used as the list end to save RAM. This is
    checked and valid. * /
    vTaskSuspendAll();
    {
        traceEVENT_GROUP_SET_BITS( xEventGroup, uxBitsToSet );
        
        pxListItem = listGET_HEAD_ENTRY( pxList );
        
        / * Set the bits. * /
        pxEventBits - >uxEventBits | = uxBitsToSet;
        
        / * See if the new bit value should unblock any tasks. * /
        while( pxListItem != pxListEnd )
        {
            pxNext = listGET_NEXT( pxListItem );
            uxBitsWaitedFor = listGET_LIST_ITEM_VALUE( pxListItem );
            xMatchFound = pdFALSE;
            
            / * Split the bits waited for from the control bits. * /
            uxControlBits = uxBitsWaitedFor & eventEVENT_BITS_CONTROL_BYTES;
            uxBitsWaitedFor & = ~eventEVENT_BITS_CONTROL_BYTES;
            
            if( ( uxControlBits & eventWAIT_FOR_ALL_BITS ) =  = ( EventBits_t ) 0 )
            {
                / * Just looking for single bit being set. * /
                if( ( uxBitsWaitedFor & pxEventBits - >uxEventBits ) != ( EventBits_t )
                0 )
                {
                    xMatchFound = pdTRUE;
                }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
    }
else if( ( uxBitsWaitedFor & pxEventBits - >uxEventBits ) =  =
uxBitsWaitedFor )
{
    / * All bits are set. * /
    xMatchFound = pdTRUE;
}
else
{
    / * Need all bits to be set, but not all the bits were set. * /
}

if( xMatchFound != pdFALSE )
{
    / * The bits match. Should the bits be cleared on exit? * /
    if( ( uxControlBits & eventCLEAR_EVENTS_ON_EXIT_BIT ) != ( EventBits_t
    ) 0 )
    {
        uxBitsToClear | = uxBitsWaitedFor;
    }
else
{
    mtCOVERAGE_TEST_MARKER();
}

/ * Store the actual event flag value in the task's event list
item before removing the task from the event list. The
eventUNBLOCKED_DUE_TO_BIT_SET bit is set so the task knows
that is was unblocked due to its required bits matching, rather
than because it timed out. * /
vTaskRemoveFromUnorderedEventList( pxListItem,
pxEventBits - >uxEventBits | eventUNBLOCKED_DUE_TO_BIT_SET );
}

/ * Move onto the next list item. Note pxListItem - >pxNext is not
used here as the list item may have been removed from the event list
and inserted into the ready / pending reading list. * /
pxListItem = pxNext;
}

/ * Clear any bits that matched when the eventCLEAR_EVENTS_ON_EXIT_BIT
bit was set in the control word. * /
pxEventBits - >uxEventBits & = ~uxBitsToClear;
}
( void ) xTaskResumeAll();

return pxEventBits - >uxEventBits;
}
```
可以在第17行发现暂停调度器的***[vTaskSuspendAll()]{.underline}***函数

最后还是那一句

谁要跟我抢，我就灭掉谁：

中断要跟我抢？我屏蔽中断

其他任务要跟我抢？我禁止调度器，不运行任务切换

### 使用实例

工程文件:30_suspend_all_dht11

本工程文件由29_fromisr_game改进

目标:增加温湿度传感器读取(在挡球板游戏界面上方实时显示),
使用软件定时器定时刷新
```c
static void DHT11Timer(TimerHandle_t xTimer)
{
    / * 在定时器当中读取DHT11温湿度数据 * /
    int hum = 0, temp = 0;
    int error = 1;
    
    vTaskSuspendAll(); /  / 暂停调度器
    
    error = DHT11_Read(&hum, &temp);
    
    xTaskResumeAll(); /  / 恢复调度器
    
    char buff[16];
    
    if (0 =  = error)
    {
        / * 在OLED上显示出来 * /
        sprintf_P(buff, "%dC, %d%%", temp, hum);
        draw_string(buff, false, 40, 0);
        
    }

else
{
    / * 显示错误信息 * /
    draw_string("error ", false, 40, 0);
    
}

}

void game1_task(void *params)
{
    ......
    
    uint8_t dev, data, last_data;
    
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    buzzer_init();
    DHT11_Init();
    
    /  / 创建软件定时器
    g_TimerDHT11 = xTimerCreate("DHT11_Timer", 2000, pdTRUE, NULL,
    DHT11Timer);
    
    / * 启动定时器 * /
    xTimerStart(g_TimerDHT11, portMAX_DELAY);
    
    ......
}
```
为了更好的显示, 防止ERROR(因为太多比它优先级高的会打断执行),
从而禁用调度器

#### 现象

上方显示温湿度, 挡球板游戏正常运行

## 调试与优化

### 精细调整栈的大小

工程文件:31_stack_state

本工程文件由30_suspend_all_dht11改良而来, 目标是为了展示任务栈的大小,
并根据这个进行调整

添加多个任务, 包括之前的全彩LED闪烁任务以及板载LED闪烁任务

下面是部分代码:
```c
/ * Private variables
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN Variables * /
static SemaphoreHandle_t g_xI2CMutex;
void game1_task(void params);

static StackType_t g_pulStackOfLightTask[128];
static StaticTask_t g_TCBOfLightTask;
static TaskHandle_t xLedTaksHandle;

static TaskHandle_t xColorTaskHandle;
static StackType_t g_pulStackOfColorTask[128];
static StaticTask_t g_TCBOfColorTask;

/ * USER CODE END Variables * /
/ * Definitions for defaultTask * /
osThreadId_t defaultTaskHandle;
const osThreadAttr_t defaultTask_attributes = {
    .name = "defaultTask",
    .stack_size = 128  4,
    .priority = (osPriority_t) osPriorityNormal,
};

/ * Private function prototypes
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FunctionPrototypes * /

void GetI2C(void)
{
    / * 等待互斥量 * /
    xSemaphoreTake(g_xI2CMutex, portMAX_DELAY);
    
}


void PutI2C(void)
{
    / * 释放互斥量 * /
    xSemaphoreGive(g_xI2CMutex);
}

void LED_TEST(void pvParameters)
{
    (void)pvParameters;
    Led_Test();
}

void ColorLED_TEST(void pvParameters)
{
    (void)pvParameters;
    ColorLED_Test();
}

/ * USER CODE END FunctionPrototypes * /

void StartDefaultTask(void *argument);

void MX_FREERTOS_Init(void); / * (MISRA C 2004 rule 8.1) * /

/ **
* @brief FreeRTOS initialization
* @param None
* @retval None
* /
void MX_FREERTOS_Init(void) {
    / * USER CODE BEGIN Init * /
    
    /  / 初始化互斥量
    g_xI2CMutex = xSemaphoreCreateMutex();
    PassiveBuzzer_Init();
    LCD_Init();
    LCD_Clear();
    
    MPU6050_Init();
    IRReceiver_Init();
    RotaryEncoder_Init();
    LCD_PrintString(0, 0, "Starting");
    
    
    / * USER CODE END Init * /
    
    / * USER CODE BEGIN RTOS_MUTEX * /
    / * add mutexes, ... * /
    / * USER CODE END RTOS_MUTEX * /
    
    / * USER CODE BEGIN RTOS_SEMAPHORES * /
    / * add semaphores, ... * /
    / * USER CODE END RTOS_SEMAPHORES * /
    
    / * USER CODE BEGIN RTOS_TIMERS * /
    / * start timers, add new ones, ... * /
    / * USER CODE END RTOS_TIMERS * /
    
    / * USER CODE BEGIN RTOS_QUEUES * /
    / * add queues, ... * /
    / * USER CODE END RTOS_QUEUES * /
    
    / * Create the thread(s) * /
    / * creation of defaultTask * /
    defaultTaskHandle = osThreadNew(StartDefaultTask, NULL, &defaultTask_attributes);
    
    / * USER CODE BEGIN RTOS_THREADS * /
    / * add threads, ... * /
    
    /  / 创建游戏任务
    
    xTaskCreate(game1_task, "GameTask", 128, NULL, osPriorityNormal, NULL);
    
    /  / extern void car_game(void);
    /  / car_game();
    
    xLedTaksHandle = xTaskCreateStatic(LED_TEST, "LedTaks", 128, NULL, osPriorityNormal, g_pulStackOfLightTask,
    &g_TCBOfLightTask);
    xColorTaskHandle = xTaskCreateStatic(ColorLED_TEST, "ColorTask", 128, NULL, osPriorityNormal, g_pulStackOfColorTask,
    &g_TCBOfColorTask);
    
    / * USER CODE END RTOS_THREADS * /
    
    / * USER CODE BEGIN RTOS_EVENTS * /
    / * add events, ... * /
    / * USER CODE END RTOS_EVENTS * /
    
}
```
将Led_Test()与ColorLED_Test()更改, 将其中的mdelay更改为vTaskDelay(),
减少对CPU资源的占用

[ MPU6050通过中断写事件组, 这个是需要定时器队列来实现的,
这使得定时器队列很快就满了 ]

[ 定时器任务优先级过低,
同时全彩灯任务和板载LED闪烁任务使用mdelay函数读CPU资源占用太高,
导致无法运行]

[ game1当中的启动定时器因此失败, 导致一直等待, 无法运行 ]
```c
/
* 函数名称： ColorLED_Test
* 功能描述： 全彩LED测试程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 04 V1.0 韦东山 创建
* /
void ColorLED_Test(void)
{
    uint32_t color = 0;
    
    ColorLED_Init();
    
    while (1)
    {
        /  / LCD_PrintString(0, 0, "Show Color: ");
        /  / LCD_PrintHex(0, 2, color, 1);
        
        ColorLED_Set(color);
        
        color +  = 200000;
        color & = 0x00ffffff;
        vTaskDelay(1000);
    }
}
```
```c
/
* 函数名称： Led_Test
* 功能描述： Led测试程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
* /
void Led_Test(void)
{
    Led_Init();
    
    while (1)
    {
        Led_Control(LED_GREEN, 1);
        vTaskDelay(500);
        
        Led_Control(LED_GREEN, 0);
        vTaskDelay(500);
    }
}
```
这样就可正常执行所有的任务

根据上面来看, 任务一旦多起来确实不好搞, 除了需要考虑任务的互斥等,
还需要考虑任务的栈大小

这里演示如何精准的测量一个任务的栈大小,
首先栈内先全是初始化的0xA5A5A5A5这个数据,
因为作为binary时候它的码是10100101这样的结构,
可以有效防止有写入的数据完全与其相同

![](media/media/image98.jpeg)

当运行一段时间之后, 任务若是有空余有剩余的栈就会这样

![](media/media/image99.jpeg)

这个时候就可以从尾部读回来, 确认剩余空间的大小了

在创建任务时分配了栈，可以填入固定的数值比如0xa5，以后可以使用以下函数查看"栈的高水位"，也就是还有多少空余的栈空间：
```c
UBaseType_t uxTaskGetStackHighWaterMark( TaskHandle_t xTask );
```
原理是：从栈底往栈顶逐个字节地判断，它们的值持续是0xa5就表示它是空闲的。

函数说明：

![](media/media/image101.png)

这个不是很大, 可以不用管

下面对其他任务也进行改善, 确认这个大小
```c
/
* 函数名称： ColorLED_Test
* 功能描述： 全彩LED测试程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 04 V1.0 韦东山 创建
* /
void ColorLED_Test(void)
{
    uint32_t color = 0;
    UBaseType_t freeNum = 0x00;
    TaskHandle_t xTaskHandle;
    
    ColorLED_Init();
    
    while (1)
    {
        /  / LCD_PrintString(0, 0, "Show Color: ");
        /  / LCD_PrintHex(0, 2, color, 1);
        
        ColorLED_Set(color);
        
        color +  = 200000;
        color & = 0x00ffffff;
        vTaskDelay(1000);
        
        xTaskHandle = xTaskGetCurrentTaskHandle();
        freeNum = uxTaskGetStackHighWaterMark(xTaskHandle);
        printf("FreeStack of Task %s : %drn", pcTaskGetName(xTaskHandle), (int)freeNum);
    }
}
```
```c
/
* 函数名称： Led_Test
* 功能描述： Led测试程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
* /
void Led_Test(void)
{
    Led_Init();
    UBaseType_t freeNum = 0x00;
    TaskHandle_t xTaskHandle;
    
    while (1)
    {
        Led_Control(LED_GREEN, 1);
        vTaskDelay(500);
        
        Led_Control(LED_GREEN, 0);
        vTaskDelay(500);
        
        xTaskHandle = xTaskGetCurrentTaskHandle();
        freeNum = uxTaskGetStackHighWaterMark(xTaskHandle);
        printf("FreeStack of Task %s : %drn", pcTaskGetName(xTaskHandle), (int)freeNum);
    }
}
```
烧录运行, 可以观察到

![](media/media/image102.png)

之后根据这个我们去调整任务的栈大小
```c
演示
C +  +
static StackType_t g_pulStackOfLightTask[80]; /  / 改为80
static StaticTask_t g_TCBOfLightTask;
static TaskHandle_t xLedTaksHandle;

static TaskHandle_t xColorTaskHandle;
static StackType_t g_pulStackOfColorTask[80]; /  / 改为80
static StaticTask_t g_TCBOfColorTask;

/ * USER CODE END Variables * /

......

/ **
* @brief FreeRTOS initialization
* @param None
* @retval None
* /
void MX_FREERTOS_Init(void) {
    
    ......
    
    xLedTaksHandle = xTaskCreateStatic(LED_TEST, "LedTaks", 80, NULL,
    osPriorityNormal, g_pulStackOfLightTask, &g_TCBOfLightTask);
    xColorTaskHandle = xTaskCreateStatic(ColorLED_TEST, "ColorTask", 80,
    NULL, osPriorityNormal, g_pulStackOfColorTask, &g_TCBOfColorTask);
```
但是这样一个一个更改大小还是太过麻烦, 下面我们来介绍一下更加便捷的方法

vTaskList
：获得任务的统计信息，形式为可读的字符串。注意，pcWriteBuffer必须足够大。
```c
void vTaskList( signed char *pcWriteBuffer );
```
可读信息格式如下：

![](media/media/image103.png)

![](media/media/image104.png)

如上图所示使用这个函数之前我们需要在CUBEMX当中配置宏定义

![](media/media/image105.png)

更改一下代码, 将之前的都注释一下
```c
/
* 函数名称： ColorLED_Test
* 功能描述： 全彩LED测试程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 无
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 04 V1.0 韦东山 创建
* /
void ColorLED_Test(void)
{
    uint32_t color = 0;
    /  / UBaseType_t freeNum = 0x00;
    /  / TaskHandle_t xTaskHandle;
    
    ColorLED_Init();
    
    while (1)
    {
        /  / LCD_PrintString(0, 0, "Show Color: ");
        /  / LCD_PrintHex(0, 2, color, 1);
        
        ColorLED_Set(color);
        
        color +  = 200000;
        color & = 0x00ffffff;
        vTaskDelay(1000);
        
        /  / xTaskHandle = xTaskGetCurrentTaskHandle();
        /  / freeNum = uxTaskGetStackHighWaterMark(xTaskHandle);
        /  / printf("FreeStack of Task %s : %drn", pcTaskGetName(xTaskHandle), (int)freeNum);
    }
}
```
```c
/
* 函数名称： Led_Test
* 功能描述： Led测试程序
* 输入参数： 无
* 输出参数： 无
* 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
* /
void Led_Test(void)
{
    Led_Init();
    /  / UBaseType_t freeNum = 0x00;
    /  / TaskHandle_t xTaskHandle;
    
    while (1)
    {
        Led_Control(LED_GREEN, 1);
        vTaskDelay(500);
        
        Led_Control(LED_GREEN, 0);
        vTaskDelay(500);
        
        /  / xTaskHandle = xTaskGetCurrentTaskHandle();
        /  / freeNum = uxTaskGetStackHighWaterMark(xTaskHandle);
        /  / printf("FreeStack of Task %s : %drn", pcTaskGetName(xTaskHandle), (int)freeNum);
    }
}
```
```c
static char pcWriteBuffer[200]; /  / 储存调试信息

......

void game1_task(void *params)
{
    uint8_t dev, data, last_data;
    /  / UBaseType_t freeNum = 0x00;
    /  / TaskHandle_t xTaskHandle;
    
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    buzzer_init();
    DHT11_Init();
    
    /  / 创建软件定时器
    g_TimerDHT11 = xTimerCreate("DHT11_Timer", 2000, pdTRUE, NULL,
    DHT11Timer);
    
    / * 启动定时器 * /
    xTimerStart(g_TimerDHT11, portMAX_DELAY);
    
    
    / * 创建队列, 创建队列集, 创建输入任务 * /
    g_xQueuePlatform = xQueueCreate(10, sizeof(input_data));
    g_xQueueSetInput = xQueueCreateSet(IR_QUEUE_LEN + ROTARY_QUEUE_LEN +
    MPU6050_QUEUE_LEN);
    
    g_xQueueIR = GetQueueIR();
    g_xQueueRotary = GetQueueRotary();
    g_xQueueMPU6050 = GetQueueMPU6050();
    
    xQueueAddToSet(g_xQueueIR, g_xQueueSetInput);
    xQueueAddToSet(g_xQueueRotary, g_xQueueSetInput);
    xQueueAddToSet(g_xQueueMPU6050, g_xQueueSetInput);
    
    xTaskCreate(MPU6050_Task, "MPU6050Task", 128, NULL, osPriorityNormal,
    NULL);
    xTaskCreate(InputTask, "InputTask", 128, NULL, osPriorityNormal,
    NULL);
    
    uptMove = UPT_MOVE_NONE;
    
    ball.x = g_xres / 2;
    ball.y = g_yres - 10;
    
    ball.velX =  - 0.5;
    ball.velY =  - 0.6;
    /  / ball.velX =  - 1;
    /  / ball.velY =  - 1.1;
    
    blocks = pvPortMalloc(BLOCK_COUNT);
    memset(blocks, 0, BLOCK_COUNT);
    
    lives = lives_origin = 3;
    score = 0;
    platformX = (g_xres / 2) - (PLATFORM_WIDTH / 2);
    
    xTaskCreate(platform_task, "platform_task", 128, NULL,
    osPriorityNormal, NULL);
    
    while (1)
    {
        game1_draw();
        /  / draw_end();
        /  / xTaskHandle = xTaskGetCurrentTaskHandle();
        /  / freeNum = uxTaskGetStackHighWaterMark(xTaskHandle);
        /  / printf("FreeStack of Task %s : %drn",
        pcTaskGetName(xTaskHandle), (int)freeNum);
        vTaskList(pcWriteBuffer);
        printf("%srn", pcWriteBuffer);
        
        vTaskDelay(50);
    }
}
```
编译烧录, 串口接收到的信息如下

![](media/media/image106.png)

但是在有用的任务当中这样添加并不好,
可以使用之前的空闲任务的钩子函数解决这个问题

工程文件:32_all_stack_state 由31改进而来
```c
void vApplicationIdleHook( void );
```
下面我们注释一下*[game1.c]{.underline}*当中的*[xTaskList()]{.underline}*,
使用这个钩子函数
```c
/  / static char pcWriteBuffer[200]; /  / 储存调试信息

......

void game1_task(void *params)
{
    uint8_t dev, data, last_data;
    /  / UBaseType_t freeNum = 0x00;
    /  / TaskHandle_t xTaskHandle;
    
    g_framebuffer = LCD_GetFrameBuffer(&g_xres, &g_yres, &g_bpp);
    draw_init();
    draw_end();
    buzzer_init();
    DHT11_Init();
    
    /  / 创建软件定时器
    g_TimerDHT11 = xTimerCreate("DHT11_Timer", 2000, pdTRUE, NULL,
    DHT11Timer);
    
    / * 启动定时器 * /
    xTimerStart(g_TimerDHT11, portMAX_DELAY);
    
    
    / * 创建队列, 创建队列集, 创建输入任务 * /
    g_xQueuePlatform = xQueueCreate(10, sizeof(input_data));
    g_xQueueSetInput = xQueueCreateSet(IR_QUEUE_LEN + ROTARY_QUEUE_LEN +
    MPU6050_QUEUE_LEN);
    
    g_xQueueIR = GetQueueIR();
    g_xQueueRotary = GetQueueRotary();
    g_xQueueMPU6050 = GetQueueMPU6050();
    
    xQueueAddToSet(g_xQueueIR, g_xQueueSetInput);
    xQueueAddToSet(g_xQueueRotary, g_xQueueSetInput);
    xQueueAddToSet(g_xQueueMPU6050, g_xQueueSetInput);
    
    xTaskCreate(MPU6050_Task, "MPU6050Task", 128, NULL, osPriorityNormal,
    NULL);
    xTaskCreate(InputTask, "InputTask", 128, NULL, osPriorityNormal,
    NULL);
    
    uptMove = UPT_MOVE_NONE;
    
    ball.x = g_xres / 2;
    ball.y = g_yres - 10;
    
    ball.velX =  - 0.5;
    ball.velY =  - 0.6;
    /  / ball.velX =  - 1;
    /  / ball.velY =  - 1.1;
    
    blocks = pvPortMalloc(BLOCK_COUNT);
    memset(blocks, 0, BLOCK_COUNT);
    
    lives = lives_origin = 3;
    score = 0;
    platformX = (g_xres / 2) - (PLATFORM_WIDTH / 2);
    
    xTaskCreate(platform_task, "platform_task", 128, NULL,
    osPriorityNormal, NULL);
    
    while (1)
    {
        game1_draw();
        /  / draw_end();
        /  / xTaskHandle = xTaskGetCurrentTaskHandle();
        /  / freeNum = uxTaskGetStackHighWaterMark(xTaskHandle);
        /  / printf("FreeStack of Task %s : %drn",
        pcTaskGetName(xTaskHandle), (int)freeNum);
        /  / vTaskList(pcWriteBuffer);
        /  / printf("%srn", pcWriteBuffer);
        
        vTaskDelay(50);
    }
}
```
配置在freertos.c当中
```c
演示
C +  +
static char pcWriteBuffer[200]; /  / 储存调试信息

......

/ * Private function prototypes
-  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  -  - * /
/ * USER CODE BEGIN FunctionPrototypes * /

......

/  / 钩子函数
void vApplicationIdleHook(void)
{
    vTaskList(pcWriteBuffer);
    printf("%srn", pcWriteBuffer);
}
```
之后回到CubeMX当中, 配置一下hook函数的使能

![](media/media/image107.png)

注意重新生成代码之后会自动生成一个无语句只有注释的钩子函数,
可以删除之前的代码
```c
/ * Hook prototypes * /
void vApplicationIdleHook(void);

/ * USER CODE BEGIN 2 * /
void vApplicationIdleHook( void )
{
    / * vApplicationIdleHook() will only be called if configUSE_IDLE_HOOK
    is set
    to 1 in FreeRTOSConfig.h. It will be called on each iteration of the
    idle
    task. It is essential that code added to this hook function never
    attempts
    to block in any way (for example, call xQueueReceive() with a block
    time
    specified, or call vTaskDelay()). If the application makes use of the
    vTaskDelete() API function (as this demo application does) then it is
    also
    important that vApplicationIdleHook() is permitted to return to its
    calling
    function, because it is the responsibility of the idle task to clean
    up
    memory allocated by the kernel to any task that has since been deleted.
    * /
    
    /  / 增加打印次数
    for (uint8_t i = 0; i < 16; i +  + )
    {
        printf(" -  -  -  -  -  - rn");
        vTaskList(pcWriteBuffer);
        printf("%srn", pcWriteBuffer);
    }
}
/ * USER CODE END 2 * /
```
编译烧录之后, 得到的信息如下, 可以通过这个信息调整对应的栈大小

![](media/media/image108.png)

经过修改之后我的是这样的

![](media/media/image109.png)

大家也可以试试看

### CPU占比显示

在上一部分当中,
我们提到了更改延时函数从而避免因为MPU6050反复中断导致的队列写满影响软件定时器无法启动

如果在一些时候我们难以分析出这个原因, 我们则可以借助函数查看CPU占用率,
从而分析可能是哪一部分起问题

#### 原理

对于同优先级的任务，它们按照时间片轮流运行：你执行一个Tick，我执行一个Tick。

是否可以在Tick中断函数中，统计当前任务的累计运行时间？

不行！很不精确，因为有更高优先级的任务就绪时，当前任务还没运行一个完整的Tick就被抢占了。

我们需要比Tick更快的时钟，比如Tick周期时1ms，我们可以使用另一个定时器，让它发生中断的周期时0.1ms甚至更短。

使用这个定时器来衡量一个任务的运行时间，原理如下图所示：

![](media/media/image110.png)

切换到Task1时，使用更快的定时器记录当前时间T1

Task1被切换出去时，使用更快的定时器记录当前时间T4

(T4-T1)就是它运行的时

间，累加起来

关键点：在 vTaskSwitchContext 函数中，使用 更快的定时器 统计运行时间

在任务切换时统计运行时间

![](media/media/image111.png)

![](media/media/image112.jpeg)

这里运用更加精准的函数
```c
/
* 函数名称： system_get_ns
* 功能描述： 获得系统时间(单位ns)
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 系统时间(单位ns)
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
* /
uint64_t system_get_ns(void)
{
    /  / extern uint32_t HAL_GetTick(void);
    extern TIM_HandleTypeDef htim4;
    TIM_HandleTypeDef *hHalTim = &htim4;
    
    uint64_t ns = HAL_GetTick();
    uint64_t cnt;
    uint64_t reload;
    
    cnt = __HAL_TIM_GET_COUNTER(hHalTim);
    reload = __HAL_TIM_GET_AUTORELOAD(hHalTim);
    
    ns *= 1000000;
    ns +  = cnt * 1000000 / reload;
    return ns;
}
```
由于需要配置生成运行时间统计信息, 这里需要打开CUBEMX, 勾选对应的选项

![](media/media/image113.png)

看到task.c当中的上下文切换函数
```c
void vTaskSwitchContext( void )
{
    if( uxSchedulerSuspended != ( UBaseType_t ) pdFALSE )
    {
        / * The scheduler is currently suspended - do not allow a context
        switch. * /
        xYieldPending = pdTRUE;
    }
else
{
    xYieldPending = pdFALSE;
    traceTASK_SWITCHED_OUT();
    
    #if ( configGENERATE_RUN_TIME_STATS =  = 1 )
    {
        #ifdef portALT_GET_RUN_TIME_COUNTER_VALUE
        portALT_GET_RUN_TIME_COUNTER_VALUE( ulTotalRunTime );
        #else
        ulTotalRunTime = portGET_RUN_TIME_COUNTER_VALUE();
        #endif
        
        / * Add the amount of time the task has been running to the
        accumulated time so far. The time the task started running was
        stored in ulTaskSwitchedInTime. Note that there is no overflow
        protection here so count values are only valid until the timer
        overflows. The guard against negative values is to protect
        against suspect run time stat counter implementations - which
        are provided by the application, not the kernel. * /
        if( ulTotalRunTime > ulTaskSwitchedInTime )
        {
            pxCurrentTCB - >ulRunTimeCounter +  = ( ulTotalRunTime -
            ulTaskSwitchedInTime );
        }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
ulTaskSwitchedInTime = ulTotalRunTime;
}
#endif / * configGENERATE_RUN_TIME_STATS * /

/ * Check for stack overflow, if configured. * /
taskCHECK_FOR_STACK_OVERFLOW();

/ * Before the currently running task is switched out, save its errno.
* /
#if( configUSE_POSIX_ERRNO =  = 1 )
{
    pxCurrentTCB - >iTaskErrno = FreeRTOS_errno;
}
#endif

/ * Select a new task to run using either the generic C or port
optimised asm code. * /
taskSELECT_HIGHEST_PRIORITY_TASK(); / *lint !e9079 void * is used as
this macro is used with timers and co - routines too. Alignment is known
to be fine as the type of the pointer stored and retrieved is the same.
* /
traceTASK_SWITCHED_IN();

/ * After the new task is switched in, update the global errno. * /
#if( configUSE_POSIX_ERRNO =  = 1 )
{
    FreeRTOS_errno = pxCurrentTCB - >iTaskErrno;
}
#endif

#if ( configUSE_NEWLIB_REENTRANT =  = 1 )
{
    / * Switch Newlib's _impure_ptr variable to point to the _reent
    structure specific to this task.
    See the third party link
    http: /  / www.nadler.com / embedded / newlibAndFreeRTOS.html
    for additional information. * /
    _impure_ptr = &( pxCurrentTCB - >xNewLib_reent );
}
#endif / * configUSE_NEWLIB_REENTRANT * /
}
}
```
再看到
```c
演示
C +  +
#if ( configGENERATE_RUN_TIME_STATS =  = 1 )
{
    #ifdef portALT_GET_RUN_TIME_COUNTER_VALUE
    portALT_GET_RUN_TIME_COUNTER_VALUE( ulTotalRunTime );
    #else
    ulTotalRunTime = portGET_RUN_TIME_COUNTER_VALUE();
    #endif
```
的***[portGET_RUN_TIME_COUNTER_VALUE()]{.underline}***这一段代码,
转到定义
```c
#define portGET_RUN_TIME_COUNTER_VALUE getRunTimeCounterValue
```
再转到***[getRunTimeCounterValue]{.underline}***的定义
```c
__weak unsigned long getRunTimeCounterValue(void)
{
    return 0;
}
```
可以看到这是一个弱定义函数,
这个函数是用于实现返回更加精确的运行起止时间的,
将其定义到***[driver_timer.c]{.underline}***当中
```c
/
* 函数名称： getRunTimeCounterValue
* 功能描述： 返回系统启动后过了多少时间(单位微秒us)
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 系统时间(单位us)
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2025 / 08 / 28 V1.0 水水水 创建
* /
unsigned long getRunTimeCounterValue(void)
{
    return system_get_ns() / 1000;
}
```
现在编辑钩子函数, 获取各个任务运行时长以及占比

vTaskGetRunTimeStats：获得任务的运行信息，形式为可读的字符串。注意，pcWriteBuffer必须足够大。
```c
void vTaskGetRunTimeStats( signed char *pcWriteBuffer );
```
可读信息格式如下：

![](media/media/image114.png)

更改后的钩子函数
```c
void vApplicationIdleHook( void )
{
    / * vApplicationIdleHook() will only be called if configUSE_IDLE_HOOK
    is set
    to 1 in FreeRTOSConfig.h. It will be called on each iteration of the
    idle
    task. It is essential that code added to this hook function never
    attempts
    to block in any way (for example, call xQueueReceive() with a block
    time
    specified, or call vTaskDelay()). If the application makes use of the
    vTaskDelete() API function (as this demo application does) then it is
    also
    important that vApplicationIdleHook() is permitted to return to its
    calling
    function, because it is the responsibility of the idle task to clean
    up
    memory allocated by the kernel to any task that has since been deleted.
    * /
    
    vTaskGetRunTimeStats(pcWriteBufferTime);
    printf(" -  -  -  -  -  - rn");
    printf("%srn", pcWriteBufferTime);
    /  / 增加打印次数
    /  / for (uint8_t i = 0; i < 16; i +  + )
    /  / {
        /  / printf(" -  -  -  -  -  - rn");
        /  / vTaskList(pcWriteBufferTask);
        /  / printf("%srn", pcWriteBufferTask);
        /  / }
}
```
编译烧录后得到的运行占比如图

工程文件:33_cpu_usage(基于32改进而来)

**[图片下载失败]**

最开始启动的时候一些不必要的任务还是有占用, 比如MPU6050之前的占用

下面我们基于这个工程, 改进成为下一个工程文件:

工程文件:34_improve_mpu6050

调试MPU6050, 并在中断回调函数打断点, 之后你就会发现,
即使不动都很容易触发中断, 写事件组

下面对MPU6050的初始化函数进行优化
```c
#define MPU6050_MOT_THR 0x1f
#define MPU6050_MOT_DUR 0x20

......

/
* 函数名称： MPU6050_Init
* 功能描述： MPU6050初始化函数,
* 输入参数： 无
* 输出参数： 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 08 / 03 V1.0 韦东山 创建
* /
int MPU6050_Init(void)
{
    MPU6050_WriteRegister(MPU6050_PWR_MGMT_1, 0x80); /  / 复位
    mdelay(100);
    MPU6050_WriteRegister(MPU6050_PWR_MGMT_1, 0x00); /  / 解除休眠状态
    MPU6050_WriteRegister(MPU6050_GYRO_CONFIG, 0x0);
    MPU6050_WriteRegister(MPU6050_ACCEL_CONFIG, 0x18);
    
    MPU6050_WriteRegister(MPU6050_SMPLRT_DIV, 0x09);
    MPU6050_WriteRegister(MPU6050_CONFIG, 0x06);
    
    MPU6050_WriteRegister(MPU6050_MOT_THR, 3); /  / 设置加速度阈值为74mg
    MPU6050_WriteRegister(MPU6050_MOT_DUR, 20); /  / 设置加速度检测时间20ms
    MPU6050_WriteRegister(MPU6050_CONFIG,0x04); /  / 配置外部引脚采样和DLPF数字低通滤波器
    MPU6050_WriteRegister(MPU6050_ACCEL_CONFIG,4); /  / 加速度传感器量程和高通滤波器配置
    
    / * 参考: https: /  / blog.csdn.net / sjf8888 / article / details / 97912391 * /
    / * 配置中断引脚 * /
    MPU6050_WriteRegister(MPU6050_INT_PIN_CFG, 0x10);
    
    / * 使能中断 * /
    MPU6050_WriteRegister(MPU6050_INT_ENABLE, 0x40);
    
    /  / 创建队列
    g_xQueueMPU6050 = xQueueCreate(MPU6050_QUEUE_LEN, sizeof(mpu6050_data));
    
    /  / 创建事件组
    g_xEventMPU6050 = xEventGroupCreate();
    
    return 0;
}
```
这里是因为中断开启过多, 导致了反复触发, 顺便配置一下低通滤波等

编译烧录运行

![](media/media/image115.png)

现在的占用率就很低了, 挡球板的漂移也改善了

尝试调试, 仍在中断回调函数打断点, 你会发现不会再轻易进入中断了

但是没有加速度的情况下, 侧倾不会动(因为只开启了加速度的中断)

这个时候, 我们只需要在MPU6050的解算当中, 未回正就一直循环
```c
/
* 函数名称： mpu6050_task
* 功能描述： MPU6050任务,它循环读取MPU6050并把数值写入队列
* 输入参数： params - 未使用
* 输出参数： 无
* 无
* 返 回 值： 0 - 成功, 其他值 - 失败
* 修改日期 版本号 修改人 修改内容
  -----------------------------------------------
2023 / 09 / 05 V1.0 韦东山 创建
/
void MPU6050_Task(void params)
{
    int16_t AccX;
    mpu6050_data result;
    extern void GetI2C(void);
    extern void PutI2C(void);
    int ret = 1;
    
    while (1)
    {
        / * 等待事件 : bit0 * /
        xEventGroupWaitBits(g_xEventMPU6050, 0x01, pdTRUE, pdFALSE, portMAX_DELAY);
        
        do
        {
            GetI2C();
            ret = MPU6050_ReadData(&AccX, NULL, NULL, NULL, NULL, NULL);
            PutI2C();
            /  / 读数据之后再解析
            if (ret =  = 0)
            {
                /  / 解析数据
                MPU6050_ParseData(AccX, 0, 0, 0, 0, 0, &result);
                
                /  / 写队列
                xQueueSend(g_xQueueMPU6050, &result, 0);
                
            }
        /  / 延迟一会, 减少资源占用
        if (result.angle_x != 90) {vTaskDelay(50);}
    } while(!ret && (result.angle_x != 90)); /  / 当角度未回正并且读到数据的时候
}
}
```
编译烧录, 运行

##### 现象

MPU6050控制挡球板运行更加丝滑合理

## 结语

到这里, 你已经基本学会FreeRTOS的皮毛了, 你可以学底层原理或者进军Linux了,
也可以钻研控制算法
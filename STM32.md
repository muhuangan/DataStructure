<!-- toc-start -->

* [0. 前言](#0-前言)
* [1. 开发环境配置](#1-开发环境配置)
* [2. GPIO](#2-gpio)
    * [2.1 芯片引脚分布](#21-芯片引脚分布)
    * [2.2 IO复用和重映射](#22-io复用和重映射)
    * [2.3 四种输出模式](#23-四种输出模式)
    * [2.4 最大IO速度](#24-最大io速度)
    * [2.5 点灯大师！](#25-点灯大师)
    * [2.6 四种输入模式](#26-四种输入模式)
    * [2.7 按钮实验！](#27-按钮实验)
* [3. UART](#3-uart)
    * [3.1 基础知识](#31-基础知识)
    * [3.2 使用串口简单地发送数据！](#32-使用串口简单地发送数据)
    * [3.3 使用串口简单接收数据！](#33-使用串口简单接收数据)
* [4. I2C](#4-i2c)
    * [4.1 基础知识](#41-基础知识)
    * [4.2 I2C简单数据收发！](#42-i2c简单数据收发)
* [5. 时钟系统](#5-时钟系统)
    * [5.1 基础知识](#51-基础知识)
    * [5.2 时钟树配置实验！](#52-时钟树配置实验)
* [6. SPI](#6-spi)
    * [6.1 总线结构](#61-总线结构)
    * [6.2 SPI的5个参数](#62-spi的5个参数)
* [7. 中断](#7-中断)
    * [7.1 中断的概念](#71-中断的概念)
    * [7.2 中断优先级](#72-中断优先级)
    * [7.3 串口中断接收实验！](#73-串口中断接收实验)
* [8. 定时器](#8-定时器)
    * [8.1 时基单元](#81-时基单元)
    * [8.2 自制延迟函数！](#82-自制延迟函数)
    * [8.3 输出比较](#83-输出比较)
    * [8.4 呼吸灯实验！](#84-呼吸灯实验)
    * [8.5 输入捕获](#85-输入捕获)

<!-- toc-end -->

# 0. 前言

本学习笔记是基于STM32F103C8T6的学习笔记，使用stm32cubemx和vscode基于cmake作为开发工具链，在archlinux上进行开发，使用stm32f103c8t6最小系统板作为开发板

本笔记主要参考这个[视频教程](https://www.bilibili.com/video/BV16J4m1w7HB?vd_source=c8cd7191b3d178cf10b977901b2d6df4&spm_id_from=333.788.videopod.sections)

# 1. 开发环境配置

1. 软件包下载

```
sudo pacman -S cmake  //提供构建工具
sudo pacman -S vscode //代码编写、调试、提供交叉编译工具、下载到soc
paru -S stm32cubemx //图形化配置引脚功能
paru -S archlinux-java-run //为stm32cubemx提供运行环境
```

使用任意方式下载serial port assistant，获得串口调试功能，执行命令`sudo usermod -a -G uucp $USER`获得串口访问权限，您可能需要重启以使该设置生效

2. vscode插件下载  
   搜索STM32CubeIDE for Visual Studio Code，下载发布者为STMicroelectronics的插件包

# 2. GPIO

## 2.1 芯片引脚分布

1. 引脚分为**普通IO引脚**和**特殊功能引脚**  
   特殊功能引脚具有特殊的，特定的功能，用户无法通过编程来控制它们，例如有些引脚为芯片电源引脚，其中Vdd接电源正极，Vss接地

    > tips:这是因为一般来说mos管的d极接电源正极，s极接地

    VBAT引脚通常用来连接备用电池，NRST引脚用于芯片复位，BOOT0引脚用于控制芯片的启动模式

2. 普通引脚分为GPIOA，GPIOB，GPIOC，GPIOD四组  
   ![引脚序号](./resources/引脚序号.png)

## 2.2 IO复用和重映射

1. IO复用  
   IO复用是指一个引脚具备多种功能，例如一个引脚既可以作为串口也可以作为定时器的某个通道等等，当我们使用某个引脚的复用功能时，意味着我们不再是简单地对一个引脚写0或者写1，而是通过某种约定俗成的、特定的方式来使用某个引脚

2. 复用功能重映射  
   将冲突的复用功能移动到备用引脚上去

## 2.3 四种输出模式

GPIO(General Purpose Input Output)通用目的的输入输出

四种输出模式：
![4种输出模式](./resources/4种输出模式.png)

1. 推挽 vs 开漏  
   推挽和开漏本质上是在控制mos管的关断上的逻辑区别  
   下图为连接引脚的两个mos管的连接方式
   ![连接引脚的两个mos管](./resources/连接引脚的两个mos管.png)

    推挽的本质是mosfet交替导通，开漏的本质是上管恒断
    ![推挽vs开漏](./resources/推挽vs开漏.png)

2. 通用 vs 复用  
   通用：通过直接在引脚上写一或写零控制引脚上的高低电平  
   复用：由其他模块托管

## 2.4 最大IO速度

在理想状态下，1和0之间的转换应该是瞬时的，然而实际上却是从1缓慢降到0、从0缓慢升到1，我们将电压升高所需的时间叫作**上升时间**，电压下降所需的时间叫作**下降时间**，中间输出有效电平的时间称为**保持时间**  
![上升时间vs下降时间vs保持时间](./resources/上升时间vs下降时间vs保持时间.png)

不难看出随着高低电平的切换速度逐渐加快，保持时间会越来越短，最终直到上升沿和下降沿完全重合

上升时间和下降时间限制了最大IO速度，上升时间和下降时间越短，最大输出速度就越大

| 速度 | 上升时间 | 保持时间 | 下降时间 | 最大输出速度 |
| ---- | -------- | -------- | -------- | ------------ |
| 低速 | 125ns    | 250ns    | 125ns    | 2MHz         |
| 中速 | 25ns     | 50ns     | 25ns     | 10MHz        |
| 高速 | 5ns      | 10ns     | 5ns      | 50MHz        |

运用时选取满足要求的最小值，因为过快的上升时间和下降时间会增大功耗，同时会更容易对电路板上的其他元器件产生干扰

## 2.5 点灯大师！

1. 打开stm32cubemx
2. 选择stm32f103c8t6开始项目
3. system core选项中选择sys
4. 将debug模式调整为serial wire
5. 选择project manager将toolchain调整为cmake
6. 本次实验采用stm32f103c8t6最小系统板，最小系统板的pc13引脚以开漏接法连接了一颗板载led，同时本次实验还希望通过推挽输出的方式点亮另一颗led，选择PA9引脚
7. 鼠标点击PA9引脚和PC13引脚，均选择GPIO_Output
8. 在system core中的GPIO选项中调节各引脚参数
9. PA9选择GPIO output level为low设置初始值为低电平，GPIO mode为Output Push Pull设置为推挽输出
10. PC13设置GPIO output level为high设置初始值为高电平，GPIO mode设置为Output Open Drain开漏输出
11. 点击generate code
12. 在while循环写入如下代码

```C
while (1){
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9, GPIO_PIN_SET);

    HAL_Delay(500);

    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_9, GPIO_PIN_RESET);

    HAL_Delay(500);

}
```

## 2.6 四种输入模式

输入上拉，输入下拉，输入浮空，模拟模式  
![4种输入模式](./resources/4种输入模式.png)

上下拉电阻作用如下  
![上下拉电阻](./resources/上下拉电阻.png)

## 2.7 按钮实验！

实验目的：通过按下按钮控制板载led的亮灭，按下按钮时led亮，松开按钮时，led灭  
本实验采用PA9连接按钮

1. 基础配置与之前相同
2. 设置PA9为GPIO_Input，设置GPIO Pull-up/Pull-down为Pull-up
3. 写入如下代码

```C
while (1){
    if(HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_9)){
      HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
    }
    else{
      HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);
    }
  }
```

# 3. UART

## 3.1 基础知识

1. UART定义与接线  
   UART，即串口，是一种通信接口，由两条线组成，Tx和Rx，其中Tx(Transmit)用于发送数据，Rx(Receive)用于接收数据  
   ![串口接线方法](./resources/串口接线.png)  
   两个设备的Tx和Rx应交错连接
2. 串口的数据帧格式  
   ![串口数据帧格式](./resources/串口数据帧.png)
3. 校验位  
   奇校验：要求数据中包含奇数个1  
   欧校验：要求数据中包含偶数个1
   ![奇校验与偶校验](./resources/奇校验与偶校验.png)
4. 波特率  
   每秒钟传输位的数量  
   ![波特率](./resources/波特率.png)  
   收发双方应选择相同的波特率
5. UART vs USART  
   ![UARTvsUSART](./resources/UARTvsUSART.png)

## 3.2 使用串口简单地发送数据！

1. 在connectivity中选择USART1，选择mode为Asynchronous即异步通信模式
2. word length 选择8 Bits(including parity)
3. parity 选择none
4. data direction 选择receive and transmit
5. generate code
6. 写入如下代码

```C
uint8_t byteNumber = 0x5a;
uint8_t byteArray[] = {1, 2, 3, 4, 5};
char ch = 'a';
char *str = "Hello world";

HAL_UART_Transmit(&huart1, &byteNumber, 1, HAL_MAX_DELAY);

HAL_UART_Transmit(&huart1, byteArray, 5, HAL_MAX_DELAY);

HAL_UART_Transmit(&huart1, (uint8_t*)&ch, 1, HAL_MAX_DELAY);

HAL_UART_Transmit(&huart1, (uint8_t*)str, strlen(str), HAL_MAX_DELAY);
```

7. 在串口调试助手中设置与cubemx中相同的波特率、word length等
8. 开始调试、观察输出

**单片机数据类型**
![单片机数据类型](./resources/单片机数据结构.png)

**串口发送数据接口**  
![串口发送接口](./resources/串口发送接口.png)  
**各参数作用**
![串口发送接口参数作用](./resources/串口发送接口参数作用.png)

## 3.3 使用串口简单接收数据！

本实验目的为根据串口接收到的数据控制板载led的亮灭

写入如下代码

```C
while (1){
    uint8_t dataRcvd = 1;

    HAL_UART_Receive(&huart1, &dataRcvd, 1, HAL_MAX_DELAY);

    if (dataRcvd == '0') {
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
    }
    else if (dataRcvd == '1') {
        HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);
  }
}
```

# 4. I2C

## 4.1 基础知识

串口只能实现一对一的信息传输，面对更多的设备时就显得有些力不从心，所以我们可以采用使用总线结构的I2C

I2C使用两条线进行主机与从机的连接：

1. SCL(Serial Clock)串行时钟线，负责传输时钟信号
2. SDA(Serial Data)串行数据线，负责传输数据

I2C一般需要在SCL和SDA上外接两颗上拉电阻，电阻阻值一般选择4.7k，SCL和SDA都应该选择开漏输出，这样做是因为我们希望能实现逻辑线与

I2C通信过程：

1. 起始位：主机向从机发送起始位，即在SCL是高电压时，向SDA发送下降沿
2. 寻址：主机向总线发送从机的地址，而后补上一位R/W#位，R代表read读，W代表write写，#代表低电压有效，R/W = 0，代表从从机写数据，R/W = 1时，代表从从机读数据，从机把SDA拉低从而释放一个应答信号Ack告诉主机寻址成功，如不成功称为Nak

> Nak原因
>
> 1.  地址填错
> 2.  要寻址的从机正忙，没来得及回复ACK
> 3.  从机故障

3. 数据传输，I2C以字节为单位传输数据，每次可以传输多个字节，主机每发送一个字节，从机都通过把SDA拉低来发送一个ACK
4. 停止位：在SCL是高电压时，向SDA发送上升沿

I2C模式：  
![I2C模式](./resources/I2C模式.png)

快速模式可以设置时钟信号的占空比：

1. $T_{Low} / T_{High} = 2/1$
2. $T_{Low} / T_{High} = 16/9$

## 4.2 I2C简单数据收发！

目标：点亮OLED屏幕，同时通过读取OLED屏幕返回的数据判断OLED屏幕是否被点亮，如果被点亮则点亮板载LED

本次实验使用一块以ssd1306为驱动芯片的0.96英寸oled屏幕

1. 创建工程，将debug模式调整为serial
2. 将pc13引脚的模式调整为gpio output，以开漏模式进行输出，初始电平为高电平
3. 打开I2C1，将i2c模式调整为高速模式，
4. 写入代码

```C
uint8_t commands[] = {0x00, 0x8d, 0x14, 0xaf, 0xa5};

HAL_I2C_Master_Transmit(&hi2c1, 0x78, commands, sizeof(commands)/sizeof(commands[0]), HAL_MAX_DELAY);

uint8_t dataRcvd;
HAL_I2C_Master_Receive(&hi2c1, 0x78, &dataRcvd, 1, HAL_MAX_DELAY);

if ((dataRcvd & (0x01 << 6)) == 0) {
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);
}
else {
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
}
```

5. 如果板载led点亮，且oled屏幕显示为白色则实验成功

# 5. 时钟系统

## 5.1 基础知识

stm32单片机总线示意图：
![总线示意图](./resources/总线示意图.png)

数字逻辑电路：

1. 组合逻辑电路
2. 时序逻辑电路

时钟分类  
![时钟分类](./resources/时钟分类.png)

> H-HighSpeed  
> L-LowSpeed  
> E-External  
> I-Internal

时钟树示意图：  
![时钟树示意图](./resources/时钟树示意图.png)

## 5.2 时钟树配置实验！

1. 创建工程，配置pc13为开漏输出，初始电平为高电平
2. 写入如下代码：

```C
while (1)
  {
    uint32_t i;

    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);

    for (i = 0; i < 1000000; i++);

    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);

    for (i = 0; i < 1000000; i++);
  }
```

3. 将代码烧录进板子，观察到板载led以一个较慢的速度进行闪烁
4. 在system_core/RCC/high speed clock(HSE)中，将值调整为crystal/ceramic resonator
5. 在clock configuration中让pll选择hse作为来源，倍频器选择9倍，sysclk来源选择pll，ahb分频选择/1，apb1分频选择/2，apb2分频选择/1
6. 保证HCLK为72MHz，PCLK1为36MHz，PCLK2为72MHz
7. 将代码烧录进板子，观察到板载led以一个较快的速度进行闪烁

# 6. SPI

## 6.1 总线结构

- MOSI：master output slave input
- MISO：master input slave output
- SCK：serial clock
- NSS：negative slave select低电压有效从机选择线

## 6.2 SPI的5个参数

1. 波特率：spi没有规定波特率，但在实际项目中一般选取几兆到几十兆Hz之间
2. 比特位传输顺序，最低有效位(LSB First)先行还是最高有效位先行(MSB First)
3. 数据位长度(8bit/16bit)
4. 时钟的极性(低极性/高极性)空闲状态下时钟线为低电平还是高电平
5. 时钟的相位：
    1. 第一边沿采集
    2. 第二边沿采集

4种时钟模式  
![4种时钟模式](./resources/4种时钟模式.png)

# 7. 中断

## 7.1 中断的概念

当突发事件发生时暂时离开正在做的事去处理突发事件，处理完成之后再返回来处理原本正在进行的事件  
![中断基本概念](./resources/中断基本概念.png)

中断让我们可以更加迅速地响应突发事件

## 7.2 中断优先级

1. 中断优先级分组  
   使用4bit数表示中断优先级，数字越小，中断优先级越高  
   4bit的左边部分叫**抢占优先级**，与中断嵌套和中断排队有关，右边部分叫**子优先级**，与中断排队有关  
   ![中断优先级](./resources/中断优先级.png)  
   ![中断优先级分组](./resources/中断优先级分组.png)

2. 中断排队
    - 优先级越高，排队越靠前
    - 优先级相同时，遵循先来后到的原则

3. 中断嵌套  
   在执行一个中断响应函数过程中去执行另一个中断响应函数

## 7.3 串口中断接收实验！

实验目的：通过串口发送不同的数据控制板载led闪灯的频率

1. 创建工程，选择debug模式为serial wire
2. 配置pc13为开漏输出，初始电平为高电平
3. 打开usart1，同时勾选nvic settings中的add，打开全局中断
4. 定义私有变量`blinkInterval`用来记录led闪灯间隔，dataRcvd作为接收缓冲区

```C
static uint32_t blinkInterval = 1000;
static uint8_t dataRcvd;
```

5. 定义回调函数

```C
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart){
  if (huart == &huart1) {
    if (dataRcvd == '1') {
      blinkInterval = 1000;
    }
    else if (dataRcvd == '2') {
      blinkInterval = 300;
    }
    else if (dataRcvd == '3') {
      blinkInterval = 50;
    }

    HAL_UART_Receive_IT(&huart1, &dataRcvd, 1);
  }
}
```

6. 写入main函数

```C
int main(void)
{

  HAL_Init();

  SystemClock_Config();

  MX_GPIO_Init();
  MX_USART1_UART_Init();
  HAL_UART_Receive_IT(&huart1, &dataRcvd, 1);

  while (1)
  {
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);

    HAL_Delay(blinkInterval);

    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);

    HAL_Delay(blinkInterval);

  }
}
```

# 8. 定时器

时钟树为定时器提供时钟信号，定时器执行和时间相关的操作

## 8.1 时基单元

时基单元结构图：  
![时基单元结构图](./resources/时基单元结构.png)

> PSC - Prescaler - 预分频器  
> CNT - Counter - 计数器  
> ARR - Auto Reload Register - 自动重装寄存器  
> RCR - Repetition Counter Register - 重复计数器

1. 时钟来源：
    1. 来自RCC(时钟树)  
       ![rcc提供时钟](./resources/rcc提供时钟.png)  
       如果APB分频器的分频倍数为/1，则倍频器倍数为x1，如分频器的分频倍数为其他，则倍频器倍数为x2
    2. 来自从模式控制器的触发信号(TRIG)
    3. 来自外部参考信号(ETRF)
2. 预分频器：  
   由于时钟来源频率过高，所以我们需要通过预分频器降频才能正常使用，预分频器分频系数为PSC+1，PSC的取值范围是[0, 65535]
3. ARR用来设置计时周期，取值范围是[0, 65535]
4. CRT用来对脉冲进行计数，取值范围是[0, 65535]  
   ![计数方式示意图](./resources/计数方式示意图.png)
   CRT计数方式：
    1. 上计数：  
       CRT从0开始增长直到和ARR的值相同，然后**溢出**，CRT又重新变为0不断循环  
       定时周期ARR+1
    2. 下计数  
       CRT从ARR的值开始递减直到变为0，然后**溢出**，CRT又重新变成ARR的值  
       定时周期ARR+1
    3. 中心对齐  
       CRT先从0递增到ARR又从ARR递减到0
5. RCR：  
   重复计数RCR + 1次，产生一次update事件，RCR的取值范围[0, 65535]

STM32F1的四种定时器：  
![STM32F1的四种定时器](./resources/STM32F1的四种定时器.png)  
只有高级定时器才有RCR

存在两个寄存器：

1. 影子寄存器
2. 活动寄存器

![寄存器预加载机制](./resources/寄存器预加载机制.png)
![寄存器预加载作用示例](./resources/寄存器预加载作用示例.png)

PSC和RCR的预加载机制是默认使能且无法关闭的，ARR的预加载机制是可手动开关且默认为关闭的，一般要手动使能

## 8.2 自制延迟函数！

实验目的：自己写一个和HAL_Delay近似的延时函数，可以利用定时器中断来实现

1. TIM1 的Clock Source选择Internal Clock，选择时钟来源是内部时钟
2. 保持时钟树为默认状态，即PCLK1和PCLK2为8MHz
3. 在TIM1的Parameters settings中选择PSC为7，即预分频倍数为7倍，此时给到时基单元的脉冲频率为1MHz
4. 设置ARR为999，RCR为0，这样update事件的频率就是1KHz，即每1ms触发一次update事件
5. 配置pc13引脚为开漏输出，初始电平为高电平
6. 打开TIM1的NVIC Settings，打开TIM1 update interrupt，开启update事件中断
7. 写入如下代码，声明自定义函数

```C
static void MyDelay(uint32_t Delay);
static uint32_t MyGetTick(void);
```

8. 定义变量记录当前是自第一次中断触发后第多少毫秒

```C
static volatile uint32_t currentMiliSecondes = 0;
```

9. 实现函数和中断回调函数

```C
static void MyDelay(uint32_t Delay){
  uint32_t expireTime = MyGetTick() + Delay;
  while (expireTime > MyGetTick()) {

  }
}

static uint32_t MyGetTick(void){
  return currentMiliSecondes;
}

void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim){
  if (htim == &htim1) {
    currentMiliSecondes++;
  }
}
```

10. 编写主函数

```C
int main(void)
{
  SystemClock_Config();
  MX_GPIO_Init();
  MX_TIM1_Init();
  HAL_TIM_Base_Start_IT(&htim1);
  while (1)
  {
    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);

    MyDelay(100);

    HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);

    MyDelay(100);
  }
}
```

11. 将代码烧录进板子，观察到板载led以一个较快的速度进行闪烁，同时通过更改MyDelay中的值可以更改led闪烁速度

## 8.3 输出比较

通过定时器输出精确定时的方波信号

PWM(Pulse-Width Modulation)：脉冲宽度调制信号  
占空比 = 高 / 周期 * 100%  
PWM特点：周期恒定，占空比可调，用占空比调节信号的大小

如何产生pwm波？  
时基单元提供一个定时周期，每个输入捕获和输出比较通道都有一个CCR(Capture/Cpmpare Register 捕获/比较寄存器)，每当CNT的值<=CCR时，输出高电平，否则输出低电平

> CCR的值决定了占空比

输出比较模式选择  
![输出比较模式](./resources/输出比较模式.png)
其中PWM1是最常用的模式

输出模式：

1. 正常输出
2. 互补输出(经过反相器)

## 8.4 呼吸灯实验！

实验目标：使一颗led灯以呼吸灯的形式进行闪烁

1. 新建一个工程
2. 保证pclk2频率为8MHz
3. 在Timers里面选择TIM1，选择clock sourcer为internal clock，设置prescaler为7，counter mode为up向上计数，设置ARR为999，开启auto-load preload
4. 在channel1选择PWM generation CH1 CH1N  
   ![pwm通道选择](./resources/timer通道选择.png)
5. 给pa8引脚连接一颗led，给pa7引脚连接另一颗led，均为推挽接法
6. 设置参数，mode设置为PWM mode1，pulse设置为0表示CCR为0，占空比为0
7. 开启arr的预加载
8. ch polarity和chn polarity都选择high表示正极性
9. 生成代码

pwm函数编程接口  
![pwm函数编程接口](./resources/pwm函数编程接口.png)

要想实现呼吸灯，我们希望的是让亮度 $=0.5\sin(2\pi t) + 0.5$  
然后用pwm占空比替代亮度
$$\text{duty} = \frac{T_{\text{High}}}{T} = \frac{\text{CCR}}{\text{ARR} + 1}$$
所以  
$$\text{CCR} = \text{duty} \times (\text{ARR} + 1)$$

所以我们要做的事情如下：

1. 获取当前时间
2. 根据亮度函数计算出当前占空比
3. 获取ARR的值
4. 计算CCR应有的值
5. 将结果写入CCR

代码如下:

```C
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
HAL_TIMEx_PWMN_Start(&htim1, TIM_CHANNEL_1);
uint16_t arr = __HAL_TIM_GET_AUTORELOAD(&htim1);

while (1){
    float t = HAL_GetTick() * 0.001;  //HAL_GetTick()返回的是ms，所以乘以0.001将单位转换成秒
    float duty = 0.5 * sin(2*3.14*t) + 0.5;
    uint16_t ccr = duty * (arr + 1);
    __HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, ccr);
}
```

## 8.5 输入捕获

定时器通道作输入时可以对信号的时间参数进行测量，这种功能叫作：**输入捕获**

每当检测到信号发生变化，cnt的值就会被保存到ccr中，然后我们读取ccr就能知道信号是在什么时间发生的变化

输入捕获分为4个阶段：

1. 输入滤波：滤出输入波形的毛刺、尖锋等，得到比较干净、纯粹的波形
2. 边沿检测：
    1. 上升沿脉冲：检测到输入波形有上升沿向外发送一个短脉冲
    2. 下降沿脉冲：检测到输入波形有下降沿向外发送一个短脉冲
3. 信号选择：
    1. TRC(从从模式控制器来的信号)(暂不涉及)
    2. 直接  
       定时器的通道一与通道二是一对，通道三与通道四是一对，在一对之间，信号可以相互引用，如果通道来自于通道本身，则叫直接
    3. 间接  
       定时器的通道一与通道二是一对，通道三与通道四是一对，在一对之间，信号可以相互引用，如果通道来自于对侧通道，则叫间接
4. 分频  
   每分频系数个上升沿/下降沿输出一个脉冲  
   每个脉冲会触发一个叫作ccx的事件，这个事件发生时会把cnt的值保存到ccr

利用这个原理就能测得脉冲的脉宽  
$\text{脉宽} = (\text{CCR2} - \text{CCR1}) \times \text{分辨率}$

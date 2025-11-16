# 初识**UART**串口与**DMA**的使用

## UART

> 我们主要用串口来进行单片机与单片机之间的通信，单片机与电脑之间的通信，单片机与外设之间的通信等等，那对于校初选来说，我们要做的应该就是 能让你的数据以一定格式显示在电脑上。后续对于打比赛来说应该是会用到UART调试，与上位机的通信，串口屏的使用等等。

### **UART的定义**

UART的全称是**`Universal Asynchronous Receiver/Transmitter`**是一种串行、异步、全双工的通信协议。

- `Universal`是**串行**的意思。串行通信是指利用一条传输线将数据一位位地顺序传送，区别于并行的8个数据位同时传输。

<img src="./Snipaste_2025-11-16_15-49-16.png" alt="Snipaste_2025-11-16_15-49-16" style="zoom: 50%;" />

- `Asynchronous`是**异步**的意思 ，在异步通讯中，**不使用时钟信号**进行数据同步，它们直接在数据信号中穿插一些同步用的信号位，或者把主体数据进行打包，以数据帧的格式传输数据。例如规定由起始位、数据位、奇偶校验位、停止位等。
  某些通讯中还需要双方约定数据的传输速率，以便更好地同步 。波特率(bps)是衡量数据传送速率的指标。     

  

​	由于*没有时钟信号作为参考*，**UART**通信双方需要约定一个特定的传输速率，这个传输速率就是**波特率**。	

​	**波特率**（Baud Rate） 是指串行通信中每秒传输的符号（Symbol）数，单位为 Baud（波特），常见的有4800、9600、115200等。

​	1 Baud = 1 Symbol/s（每秒1个符号）。符号（Symbol） 常用来表示一个二进制位（bit）

​	在UART通信中波特率是核心参数，决定了传输的速度，波特率由硬件定时器生成，公式通常为：

​                                                           				  $波特率 = \frac{系统时钟频率}{分频系数}$

<img src="./a8fb5b4791aeabbd3f51fc81ffe83da8.png" alt="a8fb5b4791aeabbd3f51fc81ffe83da8" style="zoom: 50%;" />

​	==一定要记得根据你使用的东西更改相应波特率==，不然就全是乱码了，~~*不要问我怎么知道的*~~。

​	在固定了传输速率后，我们就可以考虑怎么解析传输过来的信息了

​	我们用于传输信号的数据线在默认情况下处于高电平，为了让对方知道什么时候开始通信，UART采用拉低电平的方式标志起始位。在	发完8位数据后还带有奇偶校验位，停止位等，由此来完成一个通信过程。

<img src="./33d2392caa4617451d48fe7b454e1d83.png" alt="33d2392caa4617451d48fe7b454e1d83" style="zoom:80%;" />



- `Receiver/Transmitter` 即表示可以**发送和接收**，这个没有什么好说的。

  

附上一张通信方式分类的图，大家可以自己额外研究研究每个通信方式的特点

*~~本来笔试题我是负责出这部分的题目的，后来因为题目太多我出的这部分的都被删了(ㄒoㄒ)~~*

<img src="./16a5f1251ff9f87a522166d5771a3af9.png" alt="16a5f1251ff9f87a522166d5771a3af9" style="zoom: 20%;" />



### **UART实战**

在**UART**通信中，我们有三种主流的使用方式，即**轮询方式**，**中断方式**以及**空闲中断加+DMA方式**

首先我们要新建一个cubemx加Keil的工程，一开始的基础配置相信大家都已经熟练掌握了吧~~（应该吧）~~，（调试口，晶振，时钟树，每个外设生成.c/.h文件）

不过这次我们要开的外设是UART这个外设

<img src="./Snipaste_2025-11-16_17-54-32.png" alt="Snipaste_2025-11-16_17-54-32" style="zoom: 33%;" />

注意这里的`TX`是**发送端**，`RX`是**接收端**，`TX`应该发送给别的外设的`RX`，所以说在接线的时候单片机与另一个的串口的`RX`与`TX`**==反接==**！**==反接==**！**==反接==**！

以及共地用来使电平参考一致   	 **==共地==**！**==共地==**！**==共地==**！

<img src="./641f6d5e7b1c9559762c037d3529e8b4.png" alt="641f6d5e7b1c9559762c037d3529e8b4" style="zoom:50%;" />

`GENERATE` `CODE`之后我们就可以开始编程了。

#### 串口调试助手

但是，在编程之前，为了查看我们串口的效果，需要在电脑上显示出我们串口的内容，我们需要下载一个串口调试的软件叫做`VOFA+`，安装包在压缩包中

接下来介绍我们的串口代码

#### 轮询方式（阻塞方式）

> 原理：轮询方式下CPU会不断读取串口寄存器状态。发送数据时，当检测到发送数据寄存器TDR有数据时，调用HAL_UART_Transmit发送数据。接收数据时，当检测到接收寄存器RDR有数据时，调用HAL_UART_Receive接收数据。
>
> 缺点：CPU需要不断扫描寄存器状态，在一定程度上加重了CPU的负担，不适用于对系统精度较高的场所

##### 发送数据

```C
/**
  * @brief  在轮询方式下发送一定数量的数据
  * @note   1. 该函数连续发送数据，发送过程中通过判断TXE标志来发送下一个数据，通过判断TC标志来结束数据的发送。
  *         2. 如果在等待时间内没有完成发送，则不再发送，返回超时标志
  * @param huart   UART handle.
  * @param pData   指向发送数据缓冲区的指针 (u8 or u16 data elements).
  * @param Size    待发送数据的个数(u8 or u16)
  * @param Timeout 超时等待时间
  * @retval HAL status
  */
HAL_StatusTypeDef HAL_UART_Transmit(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size, uint32_t Timeout)
    
```

- `huart`指向当前所用的`uart`的句柄

- `pData`是要发送的数据

- `Size`是数据的大小

- `Timeout`是等待数据传输时间

在代码中我们先定义一个需要传输的数组`message[]`

```c
/* USER CODE BEGIN PV */
char message[] = "Hello World\n";
/* USER CODE END PV */
```

![Snipaste_2025-11-16_21-06-24](./Snipaste_2025-11-16_21-06-24.png)

然后在while循环中将其以1s时间间隔发送

```c
  /* USER CODE BEGIN WHILE */
  while (1)
  {
		HAL_UART_Transmit(&huart1,(uint8_t *)message,13,100);
		HAL_Delay(1000);
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
```

![Snipaste_2025-11-16_21-07-37](./Snipaste_2025-11-16_21-07-37.png)

在`VOFA`中选择对应的端口号打开就可以看见发送的数据了

<img src="./Snipaste_2025-11-16_21-12-30.png" alt="Snipaste_2025-11-16_21-12-30" style="zoom: 33%;" />

##### 接收数据

```C
/**
  * @brief  在轮询方式下接收一定数量的数据
  * @note   1. 该函数连续接收数据，在接收过程中通过判断RXNE标志来接收新的数据
  *         2.如果在等待时间内没有完成接收，则不再接收，返回超时标志
  * @param huart   UART handle.
  * @param pData   指向接收数据缓冲区的指针(u8 or u16 data elements).
  * @param Size    待接收数据的个数 (u8 or u16)
  * @param Timeout 超时等待时间
  * @retval HAL status
  */
HAL_StatusTypeDef HAL_UART_Receive(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size, uint32_t Timeout)
    
```

参数类似于上面的

接下来我们看看接收函数的效果

```C
/* USER CODE BEGIN PV */
// 定义接收数组
uint8_t rx_buf;
/* USER CODE END PV */
```

```C
  /* USER CODE BEGIN WHILE */
  while (1)
  {
		//接收数据
		HAL_UART_Receive(&huart1,&rx_buf,sizeof(rx_buf),HAL_MAX_DELAY);
		// 把接收到的数据回发
		HAL_UART_Transmit(&huart1,&rx_buf,sizeof(rx_buf),HAL_MAX_DELAY);
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
```

实现效果如下：

<img src="./Snipaste_2025-11-16_21-36-01.png" alt="Snipaste_2025-11-16_21-36-01" style="zoom:33%;" />

#### 中断方式

上面的这种方式会使程序阻塞在while循环中的``HAL_UART_Receive(&huart1,&rx_buf,sizeof(rx_buf),HAL_MAX_DELAY);``这条语句中，这很显然是我们不愿意看到的，因此我们可以采用中断的方法，让cpu先去处理别的函数，等到接收到数据，触发中断再回来处理接收到的数据，这种模式就叫做中断模式。

> 原理：中断就是在寄存器有一个字节数据的时候触发一次中断，而不用一直扫描寄存器状态，节约了系统资源。例如串口接收24字节数据，HAL_UART_Receive_IT(&huart1, (uint8_t *)&Rx, 1) 意思就是每来一个字节数据中断一次，中断之后就进入回调函数进行处理，此时Rx是一个uint8_t的字节数据。HAL_UART_Receive(&huart1, (uint8_t *)Rx, 24)意思是没来一个字节数据中断一次，等接收到24个字节数据之后再统一进入回调函数之后进行处理，此时Rx是一个uint8_t的数组字节数据。
>
> 缺点：虽然解决了轮询不断扫描寄存器状态的缺点，但CPU接收数据会触发中断，对于实时要求高的场所，不适用。

首先在cubemx中我们需要将中断开启

<img src="./Snipaste_2025-11-16_22-09-41.png" alt="Snipaste_2025-11-16_22-09-41" style="zoom: 40%;" />

重新`GENERATE CODE`应用修改就可以改写我们的代码了

其实改成中断的方式很简单，只需要在原来的语句后面加上`_IT`，再改一下参数就可以了。

##### 发送数据

```C
/**
  * @brief  在中断方式下发送一定数量的数据
  * @param huart UART handle.
  * @param pData 指向发送数据缓冲区的指针 (u8 or u16 data elements).
  * @param Size  待发送数据的个数(u8 or u16)
  * @retval HAL status
  */
HAL_StatusTypeDef HAL_UART_Transmit_IT(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size)
    
```

##### 接收数据

```C
/**
  * @brief  在中断方式下接收一定数量的数据
  * @param huart UART handle.
  * @param pData 指向接收数据缓冲区的指针 (u8 or u16 data elements).
  * @param Size  待接收数据的个数(u8 or u16)
  * @retval HAL status
  */
HAL_StatusTypeDef HAL_UART_Receive_IT(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size)

```































**[UART通信协议及其工作原理（图文并茂+超详细）-CSDN博客](https://blog.csdn.net/weixin_39939185/article/details/134657483)



**[通信方式的分类（串行通信和并行通信）-CSDN博客](https://blog.csdn.net/Rocher_22/article/details/116590629)

[基于STM32HAL库的三种串口接收方式_stm32 hal 串口接收-CSDN博客](https://blog.csdn.net/qq_44758496/article/details/132206223)

[stm32 HAL库 UART 笔记_stm32 hal uart-CSDN博客](https://blog.csdn.net/best_xo/article/details/140328396)

[STM32 HAL库 UART串口发送数据实验_stm32 hal 串口发送-CSDN博客](https://blog.csdn.net/ElePower9527/article/details/145631460)

[【STM32】CUBEMX之串口：串口三种模式（轮询模式、中断模式、DMA模式）的配置与使用示例 + 串口重定向 + 使用HAL扩展函数实现不定长数据接收_cubemx配置串口-CSDN博客](https://blog.csdn.net/Rick0725/article/details/136576310)

[【STM32HAL库】串口通信-UART_stm32 hal uart-CSDN博客](https://blog.csdn.net/2301_79330491/article/details/138000201)


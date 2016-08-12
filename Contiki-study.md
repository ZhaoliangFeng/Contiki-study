# Contiki study

## Contiki 简介

>* Contiki是一个开源的、高度可移植的多任务操作系统，适用于联网嵌入式系统和无线传感器网络，由瑞典计算机科学学院（Swedish Institute of Computer Science）的Adam Dunkels和他的团队开发，已经应用在许多项目中。
>
>*  Contiki支持IPv4/IPv6通信，提供了uIPv6协议栈、IPv4协议栈（uIP），支持TCP/UDP，还提供了线程、定时器、文件系统等功能。Contiki是采用 C 语言开发的非常小型的嵌入式操作系统，针对小内存微控制器设计，典型的Contiki配置只需要2KB的RAM和40KB的ROM。

### Contiki源码结构

* Contiki源码的下载地址：https://github.com/contiki-os/contiki

* 参考学习地址：http://blog.chinaunix.net/uid-9112803-id-2978041.html

* Contiki源码结构
	* 源码文件结构图：
	![contiki](https://cloud.githubusercontent.com/assets/13186592/17460106/f2e83e04-5c89-11e6-9690-25168266aea9.png)
	* 源码文件夹介绍
**Contiki是一个高度可移植的操作系统，它的设计就是为了获得良好的可移植性，因此源代码的组织很有特点：**

	> * apps目录下是一些应用程序，例如ftp、shell、webserver等等，在项目程序开发过程中可以直接使用。
	>
	> * core目录下是Contiki的核心源代码，包括网络（net）、文件系统（cfs）、外部设备（dev）、链接库（lib）等等，并且包含了时钟、I/O、ELF装载器、网络驱动等的抽象。
	>
	> *  cpu目录下是Contiki目前支持的微处理器，例如arm、avr、msp430等。
	>
	> *  doc目录是Contiki帮助文档目录，对Contiki应用程序开发很有参考价值。
	> 
	> * examples目录下是针对不同平台的示例程序。
	> 
	> *  platform目录下是Contiki支持的硬件平台，例如mx231cc、micaz、sky、win32等等，Contiki的平台移植主要在这个目录下完成，这一部分的代码与相应的硬件平台相关。
	> 
	> * tools目录下是开发过程中常用的一些工具，例如CFS相关的makefsdata、网络相关的tunslip、模拟器cooja和mspsim等等。

### Contiki特点

* 网络节点操作系统
* 小型的、开源、极易移植的多任务操作系统
*  运行只需要几K的内存
*  包括文件系统Coffee、网络协议栈uIP和Rime、网络仿真器COOJA、内核等
*  两个主要机制：事件驱动和protothread机制

> 嵌入式系统常常被设计成响应周围环境的变化，而这些变化可以看成一个个事件。事件来了，操作系统处理之，没有事件到来，就跑去休眠了(降低功耗)，这就是所谓的事件驱动，类似于中断。

> 传统的操作系统使用栈保存进程上下文，每个进程需要一个栈，这对于内存极度受限的传感器设备将难以忍受。protothread机制恰解决了这个问题，通过保存进程被阻塞处的行数(进程结构体的一个变量，unsiged short类型，只需两个字节)，从而实现进程切换，当该进程下一次被调度时，通过switch(__ LINE__)跳转到刚才保存的点，恢复执行。整个Contiki只用一个栈，当进程切换时清空，大大节省内存。

### Contiki运行原理

* 嵌入式系统可以看作是一个运行着死循环主函数系统，Contiki内核是基于事件驱动的，系统运行可以视为不断处理事件的过程。Contiki整个运行是通过事件触发完成，一个事件绑定相应的进程。当事件被触发，系统把执行权交给事件所绑定的进程。一个典型基于Contiki的系统运行示意图如下：

	![contiki](https://cloud.githubusercontent.com/assets/13186592/17460195/aefc15a4-5c8d-11e6-896d-54b02e3d1c53.png)
* 示意主函数代码：
```C
int main(){
	clock_init();				//时钟初始化
	process_init();				//进程初始化
	process_start(&time_process,NULL);	//启动系统进程
	autostart_start(autostart_process);	//启动用户自启动线程
    
	while(1){
	 /*函数process_run的功能*/
 		if(poll_requested){
			do_poll();		//执行完所有高优先级的进程
 		}
		do_event();			//仅处理事件队列的一个事件
 	}
	return 0;
}
```

## Contiki内核及运行原理

### 内核

#### Protothreads
* Tips
>>Protothreads是一种针对C语言封装后的宏函数库，为C语言模拟了一种无堆栈的轻量线程环境，
>>能够实现模拟线程的条件阻塞、信号量操作等操作系统中特有的机制，从而使程序实现多线程操作。

内核最主要的三个核心（数据结构）：
* process：进程
* event_data：事件
* etimer：事件定时器


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

* Thread和Protothreads栈的对比图：
	![_20160812135235](https://cloud.githubusercontent.com/assets/13186592/17613882/3c422ac0-6094-11e6-8df1-7decd46e6f62.png)
> 从图可以看出，原本需要 3 个栈的Thread机制，在Protothreads只需要一个栈，当进程数量很多的时候，由栈空间省下来的内存是相当可观的。保存程序断点在传统的Thread机制很简单，只需要要保存在私有的栈，然而Protothreads不能将断点保存在公有栈中。Protothreads很巧妙地解决了这个问题，即用一个两字节静态变量存储被中断的行，因为静态变量不从栈上分配空间，所以即使有任务切换也不会影响到该变量，从而达到保存断点的。下一次该进程获得执行权的时候，进入函数体后就通过 switch 语句跳转到上一次被中断的地方。

* 保存断点
  保存断点是通过保存行数来完成的，在被中断的地方插入编译器关键字__LINE__，编译器便自动记录所中断的行数。展开那些具有中断功能的宏，可以发现最后保存行数是宏 LC_SET，取宏 PROCESS_WAIT_EVENT()为例，将其展开得到如下代码：
```C
#define PROCESS_WAIT_EVENT() PROCESS_YIELD()
#define PROCESS_YIELD() PT_YIELD(process_pt)
#define PT_YIELD(pt) \
do{ \
PT_YIELD_FLAG = 0; \
LC_SET((pt)->lc); \
if(PT_YIELD_FLAG == 0) \
{
return PT_YIELDED; \
} \
}while(0)

#define LC_SET(s) s = __LINE__; case __LINE__: //保存程序断点，下次再运行该进程直接跳到 case __LINE__
```

**值得一提的是，宏 LC_SET 展开还包含语句 case __LINE__，用于下次恢复断点，即下次通过 switch 语言便可跳转到 case 的下一语句。**

* 恢复断点
  被中断程序再次获得执行权时，便从该进程的函数执行体进入，按照Contiki的编程替换，函数体第一条语句便是PROCESS_BEGIN宏，该宏包含一条 switch语句，用于跳转到上一次被中断的行，从而恢复执行， 宏 PROCESS_BEGIN 展开的源代码如下：

```C
#define PROCESS_BEGIN() PT_BEGIN(process_pt)
#define PT_BEGIN(pt) { char PT_YIELD_FLAG = 1; LC_RESUME((pt)->lc)
#define LC_RESUME(s) switch(s) { case 0: //switch 语言跳转到被中断的行
```
#### 内核最主要的三个核心（数据结构）：

* process：进程
* event_data：事件
* etimer：事件定时器

##### process-进程以及进程调度

* 进程是一个系统最重要的概述，Contiki的进程机制是基于Protothreads线程模型，为确保高优先级任务尽快得到响应，Contiki采用两级进程调度。
* 数据结构，也即进程控制块。
```C
struct process {
  struct process *next;
#if PROCESS_CONF_NO_PROCESS_NAMES
#define PROCESS_NAME_STRING(process) ""
#else
  const char *name;
#define PROCESS_NAME_STRING(process) (process)->name
#endif
  PT_THREAD((* thread)(struct pt *, process_event_t, process_data_t));
  struct pt pt;
  unsigned char state, needspoll;
};
```
> * 成员的描述
> * next-指向下一个进程的指针
> * name-进程名称
> * thread-进程的执行体
> * pt-记录进程被中断的行数
> * state-进程的状态
> * needspoll-进程优先级

##### 进程链表process_list

Contiki采用单向链表来管理系统所有进程

![_20160812141348](https://cloud.githubusercontent.com/assets/13186592/17614318/0580fae4-6098-11e6-9ce8-7f34b4ec3c14.png)

* Contiki定义了两个变量用来管理这个链表：
* process_list-指向链表头部
* process_current-指向当前进程

##### 进程状态
* Contiki系统的进程有三个状态：
```C
#define PROCESS_STATE_NONE    0
#define PROCESS_STATE_RUNNING 1
#define PROCESS_STATE_CALLED  2
```

* 进程状态转换图如下所示：

	![_20160812142041](https://cloud.githubusercontent.com/assets/13186592/17614340/3350d8f4-6098-11e6-92e9-356a0bdfa2e4.png)

> PROCESS_STATE_NONE是指进程不处于运行状态，而PROCESS_STATE_RUNNING和PROCESS_STATE_CALLED都属于运行状态。创建进程(还未投入运行)以及进程退出(但此时还没从进程链表删除)，进程状态都为PROCESS_STATE_NONE。通过进程启动函数process_start将新创建的进程投入运行队列(但未必有执行权)，此时进程状态为PROCESS_STATE_RUNNING，真正获得执行权的进程状态为PROCESS_STATE_CALLED，处在运行队列的进程(包括正在运行和等待运行)可以调用exit_process退出。

##### 进程相关操作

> * 进程初始化
> * void  process_init (void)

> * 创建进程
> * PROCESS_THREAD(name, ev, data)

> * 启动进程
> * void  process_start (struct process *p, const char *arg)

> * 进程退出
> * process_exit (struct process *p)

####  event-事件以及事件调度

* Contiki 将事件机制融入Protothreads 机制，每个事件绑定一个进程(广播事件例外)，进程间的消息传递也是通过事件来传递的
**数据结构**
```C
struct event_data {
  process_event_t ev;
  process_data_t data;
  struct process *p;
};
```

**成员描述：**
* ev-事件标识
* data-事件对应的消息指针
* p-事件绑定的进程

**事件队列**
* Contiki采用一个全局的静态数组用于存放事件，逻辑上形成环形队列，同时对系统来说，事件采用先到先服务策略

	 ![_20160812143055](https://cloud.githubusercontent.com/assets/13186592/17615050/dfcef962-609d-11e6-8b57-24238339875a.png)

> * Contiki定义了两个变量用来管理事件队列
> * nevent-记录未处理的事件的总数
> * fevent-记录下一个待处理事件的位置

**事件相关操作**
> * 新建事件
> * process_event_t  process_alloc_event (void)

> * 事件产生（异步和同步）
> * process_post (struct process *p, process_event_t ev, void *data)
> * process_post_synch (struct process *p, process_event_t ev, void *data)

> * 事件调度
> * static void do_event(void)

### event timer-事件定时器

* etimer是contiki系统的一类特殊事件，是跟进程绑定，当etimer到期时，会给相应绑定的的进程传递事件PROCESS_EVENT_TIMER。

**数据结构**
```C
struct etimer {
  struct timer timer;
  struct etimer *next;
  struct process *p;
};

struct timer {
  clock_time_t start;
  clock_time_t interval;
};
```
**成员描述：**
* timer-定时器定时属性设置
* next-指向下一个etimer
* p-定时器绑定的进程

**etimer链表**
* Contiki采用单向链表来管理系统etimer

	![_20160812143757](https://cloud.githubusercontent.com/assets/13186592/17615284/c6f1dcfa-609f-11e6-9bc9-3db3e282f345.png)

* Contiki定义了两个变量用来管理etimer
* Timerlist-指向链表头部
* next_expiration-记录timerlist下一个到期的时间，即到了next_expiration就会有etimer到期

**etimer相关操作**

> * etimer添加
> * void  etimer_set (struct etimer *et, clock_time_t interval)
* 定义一个 etimer 结构体，调用 etimer_set 函数将 etimer 添加到 timerlist，函数etimer_set 流程图如下： 
	![1111111](https://cloud.githubusercontent.com/assets/13186592/17664990/84735c18-632a-11e6-8edc-c640c7386e3c.png)
* etimer_set 首先设置 etimer 成员变量 timer 的值(由 timer_set 函数完成)，即用当前时间初始化start，并设置间隔interval，接着调用add_timer 函数，该函数首先 将管理 etimer 系统进程 etimer_process 优先级提升，以便定时器时间到了可以得到 更快的响应。接着确保欲加入的 etimer 不在 timerlist 中(通过遍历 timerlist 实现)， 若该 etimer 已经在etimer链表，则无须将etimer加入链表，仅更新时间。否则将该etimer插入到timerlist 链表头位置，并更新时间(update_time)。这里更新时间的意思是求出etimer链表中，还需要多长next_expiration(全局静态变量)时间，就会有etimer到期。


> * etimer管理
> * PROCESS(etimer_process, "Event timer")

* Contiki 用一个系统进程 etimer_process 管理所有 etimer 定时器。进程退出时， 会向所有进程发送事件PROCESS_EVENT_EXITED，当然也包括etimer 系统进程 etimer_process。当 etimer_process拥有执行权的时候，便查看是否有相应的etimer绑定到该进程，若有就删除这些etimer。除此之外，etimer_process 还会处理到期 的 etimer，etimer_process 的 thread 函数流程图如下：

	![2222222](https://cloud.githubusercontent.com/assets/13186592/17665054/034a230a-632b-11e6-82c2-461df0374bac.png)

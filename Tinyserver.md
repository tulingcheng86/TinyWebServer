# config模块

## config 独立参数模块

这里体现出整个项目的优点：**模式切换**
也就是说这是一个复合的ET/LT。作者在Listenfd上和cfd上都建立了两种模式，意味着我们有四种组合方式。



### ET与LT模式

lfd的ET代表着一次性接受所有连接，笔者认为这里是考虑到网络接入量巨大，瞬间占到Max_size的情况。LT代表一次取一个，当然这是默认的方式也是最常见的方式。

cfd的两种方式也就是对应了epoll的方式，默认的LT和手动的ET



## config.h

用于配置Web服务器的各种参数[config.h](https://github.com/qinguoyi/TinyWebServer/blob/master/config.h)

- **PORT**：服务器监听的端口号。
- **LOGWrite**：日志写入方式，例如同步写入或异步写入。
- **TRIGMode**：触发模式的组合，通常用于设置事件处理模式，如Level Trigger或Edge Trigger。
- **LISTENTrigmode**：用于监听套接字的触发模式。
- **CONNTrigmode**：用于连接套接字的触发模式。
- **OPT_LINGER**：设置是否开启TCP连接的优雅关闭，即在关闭连接前处理完所有数据传输。
- **sql_num**：数据库连接池的数量，用于管理数据库连接，提高数据库操作效率。
- **thread_num**：线程池中线程的数量，影响服务器处理请求的能力。
- **close_log**：是否关闭日志记录，用于控制是否输出日志信息。
- **actor_model**：并发模型的选择，用于决定使用多进程、多线程还是其他并发模型。

## config.cpp

[config.cpp](https://github.com/qinguoyi/TinyWebServer/blob/master/config.cpp)

其实很简单，在构造函数里作出了对于各种初始模式的设定
并且在这版代码中，对于并发模式的处理，作者给出了reactor和preactor两种方式。（后面会详细讲解）
原作者的测试环境为4核8线，所以这里给出了池内线程为8
TRIGMode默认为最低效模式，可以改为1，实现服务器的最高性能，大概实现10wQPS



### 构造函数 `Config::Config()`

这个构造函数设置了服务器配置的默认值：

- **PORT**：默认端口号为9006。
- **LOGWrite**：日志写入方式，默认为0（同步写入）。
- **TRIGMode**：默认为0，代表使用 `listenfd` 和 `connfd` 的LT（Level Trigger）模式。
- **LISTENTrigmode** 和 **CONNTrigmode**：监听和连接的触发模式，默认都是LT。
- **OPT_LINGER**：优雅关闭连接，默认不使用（0）。
- **sql_num**：数据库连接池的默认数量为8。
- **thread_num**：线程池中的默认线程数量也是8。
- **close_log**：默认不关闭日志（0）。
- **actor_model**：并发模型，默认为0，通常指代proactor模型。



### 函数 `void Config::parse_arg(int argc, char*argv[])`

这个函数用于解析命令行参数，并根据参数调整 `Config` 对象的属性值。它使用了标准的 `getopt` 函数来处理参数。每个选项后面跟着一个冒号表示该选项需要一个参数。选项和它们对应的功能如下：

- `-p`：设置端口号。
- `-l`：设置日志写入方式。
- `-m`：设置触发组合模式。
- `-o`：设置是否使用优雅关闭连接。
- `-s`：设置数据库连接池的数量。
- `-t`：设置线程池的线程数量。
- `-c`：设置是否关闭日志。
- `-a`：设置并发模型。

这样的实现允许用户在启动服务器时通过命令行灵活配置服务器的各项参数，而无需修改代码重新编译，使得服务器配置更加灵活和动态。



# main 模块

[main](https://github.com/qinguoyi/TinyWebServer/blob/master/main.cpp)

**webserver服务器初始化,启动webserver各个功能**

给数据库的运行方式传参，当然你可以什么都不传。
定义，之后初始化了一个服务器对象。
首先打开线程池，然后设置运行模式，之后就是启动监听和进入工作循环（事务循环）

1. **服务器功能启动**：
   - `server.log_write();`：根据配置启动日志记录。
   - `server.sql_pool();`：初始化数据库连接池。
   - `server.thread_pool();`：创建和启动线程池。
   - `server.trig_mode();`：设置事件处理的触发模式。
   - `server.eventListen();`：开始监听网络事件。
   - `server.eventLoop();`：进入服务器的主事件循环，处理网络事件和用户请求。

# WebServer模块

## [webserver.h](https://github.com/qinguoyi/TinyWebServer/blob/master/webserver.h)

这个类集成了多种功能，包括线程池、数据库连接池、定时器处理、以及信号处理等。

### 主要功能

- **`init`**：初始化服务器的配置。
- **`thread_pool`**：初始化线程池。
- **`sql_pool`**：初始化数据库连接池。
- **`log_write`**：配置日志写入。
- **`trig_mode`**：设置 `epoll` 的触发模式。
- **`eventListen`**：设置并启动监听。
- **`eventLoop`**：执行主事件循环，处理网络事件。
- **`timer`, `adjust_timer`, `deal_timer`**：管理和调整定时器，处理连接超时。
- **`dealclientdata`**：处理客户端数据。
- **`dealwithsignal`**：处理信号。
- **`dealwithread`, `dealwithwrite`**：处理读写事件。



接下来对每个功能进行一一叙述：

## [webserver.cpp](https://github.com/qinguoyi/TinyWebServer/blob/master/webserver.cpp)

### 1 构造函数和析构函数

- **构造函数** (`WebServer::WebServer()`) 初始化 `http_conn` 类数组 `users` 和服务器根目录 `m_root`。同时设置定时器数据结构 `users_timer`。
- **析构函数** (`WebServer::~WebServer()`) 负责关闭网络连接、删除动态分配的资源，确保所有资源得到妥善处理。

### 2 初始化和配置

- **`init`** 方法配置服务器的基本参数，如端口、数据库信息、线程池大小等。
- **`trig_mode`** 方法设置触发模式，控制如何处理网络事件，包括使用边缘触发或水平触发。

### 3 网络和事件处理

- **`eventListen`** 设置网络监听，包括socket配置、绑定和监听操作。
- **`eventLoop`** 是服务器的主事件循环，使用 `epoll_wait` 监听事件，根据事件类型调用不同的处理函数。

### 4 辅助功能实现

- **`log_write`** 初始化日志系统。
- **`sql_pool`** 初始化数据库连接池并准备数据库操作。
- **`thread_pool`** 初始化线程池，用于处理具体的客户请求。

### 5 定时器和信号处理

- **`timer`, `adjust_timer`, `deal_timer`** 管理连接的活动状态和超时处理。
- **`dealwithsignal`** 处理操作系统信号，如定时信号和终止信号。

### 6 读写事件处理

- **`dealwithread`** 和 **`dealwithwrite`** 根据模型（Reactor或Proactor）处理读写事件，调整定时器或直接处理网络读写。

### 7 关键点和特性

- **并发模型支持**：类支持配置为Reactor或Proactor模型，影响读写事件的处理方式。
- **优雅关闭**：支持配置TCP连接的优雅关闭，防止数据丢失。
- **定时器功能**：实现了定时检测机制，自动处理超时连接，释放资源。

## [trig_mode函数](https://github.com/qinguoyi/TinyWebServer/blob/4bcf88762f85135e9bd46a1032e815e458de6f2e/webserver.cpp#L47)

在config.cpp中默认为0

不用过多解释，对应四个功能

**LT + LT（对监听和已连接套接字均使用水平触发）**

**LT + ET（监听套接字使用水平触发，已连接套接字使用边缘触发）**

**ET + LT（监听套接字使用边缘触发，已连接套接字使用水平触发）**

**ET + ET（对监听和已连接套接字均使用边缘触发）**





## [eventListen函数](https://github.com/qinguoyi/TinyWebServer/blob/4bcf88762f85135e9bd46a1032e815e458de6f2e/webserver.cpp#L103)










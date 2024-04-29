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

`eventListen` 方法是 `WebServer` 类中负责初始化网络监听和设置相关网络参数的函数。这个方法集成了创建监听套接字、配置套接字选项、绑定、监听以及初始化 `epoll` 事件处理的一系列步骤。



### 1 民工三连（创建、绑定、监听）

1. **创建监听套接字**：
   - 使用 `socket()` 函数创建一个 TCP 套接字，指定使用 IPv4 (`PF_INET`) 和流式套接字 (`SOCK_STREAM`)。
   - 使用 `assert` 确保套接字创建成功，如果失败则程序将终止。
2. **设置优雅关闭连接**：
   - 使用 `setsockopt()` 函数设置 `SO_LINGER` 选项，根据 `m_OPT_LINGER` 的值确定是立即返回关闭还是等待缓冲区数据发送完毕后关闭。
   - `struct linger` 的两个成员分别指定延迟关闭的行为和超时时间。
3. **地址绑定**：
   - 初始化 `sockaddr_in` 结构，设置协议族、地址和端口。
   - 使用 `htons()` 和 `htonl()` 函数确保端口号和地址在网络字节序中正确。
   - 使用 `bind()` 函数将套接字绑定到指定的地址和端口。
4. **监听连接请求**：
   - 调用 `listen()` 函数，开始监听网络连接请求，设置最大等待连接队列长度为5。



### **2 基本介绍：epoll调用三连**

 `epoll` 时常常提到的 "三连" 指的是三个基本的系统调用，这些调用是使用 `epoll` 处理 I/O 事件的核心。这三个调用分别是：

1. **`epoll_create`**：
   - 这是第一个调用，用于创建一个 `epoll` 的实例。它返回一个文件描述符，代表 `epoll` 事件监听器。这个文件描述符将用于所有后续的 `epoll` 操作。
   - 例如：`int epollfd = epoll_create1(0);`
   - `epoll_create1(0)` 是 `epoll_create` 的更新版本，提供了更多的功能，例如支持关闭时自动释放。
2. **`epoll_ctl`**：
   - 这个调用用于控制 `epoll` 文件描述符上的事件：可以注册新的描述符，修改已注册描述符的监听事件，或从监听中删除描述符。
   - 参数包括 `epoll` 文件描述符、操作类型（添加、修改、删除）、目标文件描述符和一个指向 `epoll_event` 结构的指针，该结构指定了需要监听的事件和与事件相关联的用户数据。
   - 例如：`epoll_ctl(epollfd, EPOLL_CTL_ADD, sockfd, &event);`
3. **`epoll_wait`**：
   - 这个调用是 `epoll` 的核心，用于等待事件的发生。程序执行到这里会阻塞，直到一个或多个事件发生，或者超时。
   - 它需要 `epoll` 文件描述符、一个事件数组用来返回发生的事件、数组能承载的最大事件数以及超时时间（毫秒）。如果超时时间设置为 -1，则会一直阻塞直到有事件发生。
   - 例如：`int n = epoll_wait(epollfd, events, MAX_EVENTS, -1);`

这三个调用合起来使得 `epoll` 能够高效地管理多个套接字描述符的 I/O 事件，尤其适用于处理大量并发连接的场景。

### 3 初始化 epoll 和工具设置

1. **初始化 epoll**：
   - 使用 `epoll_create()` 创建一个 `epoll` 实例，预设大小为 5。
   - 将监听套接字加入到 `epoll` 的监听事件列表中，根据 `m_LISTENTrigmode` 设置是否使用边缘触发。
2. **设置非阻塞管道**：
   - 使用 `socketpair()` 创建一对互联的 UNIX 域套接字，用于内部通信。
   - 设置其中一个套接字为非阻塞模式，并加入到 `epoll` 监听。





## [eventLoop函数](https://github.com/qinguoyi/TinyWebServer/blob/4bcf88762f85135e9bd46a1032e815e458de6f2e/webserver.cpp#L377)

`eventLoop()` 方法是 `WebServer` 类中用于处理网络事件的主循环。它使用 `epoll` 机制来高效地管理多个网络连接的事件。

![image-20240429171029027](/home/sbim/.config/Typora/typora-user-images/image-20240429171029027.png)

**我们主要处理三种事件：io事件，信号，新的连接**
也就是在for循环中的三次判断，并且每次处理完一组后，我们会刷新定时。



### 1 处理 `epoll` 事件

- **新客户连接**：如果事件发生在监听套接字（`m_listenfd`）上，调用 **`dealclientdata()`** 来接受新的连接。
- **异常事件处理**：如果事件类型包含 `EPOLLRDHUP`（对端套接字关闭），`EPOLLHUP`（挂起），或 `EPOLLERR`（错误发生），则处理相关的清理工作，例如移除定时器。
- **信号处理**：如果事件发生在管道文件描述符 `m_pipefd[0]` 上并且是可读事件（`EPOLLIN`），则处理信号，通过调用 **`dealwithsignal()`** 来决定是否需要停止服务器或处理超时。
- **数据读取**：如果事件类型是 `EPOLLIN`，且不是监听套接字或管道，说明有数据可读，调用 **`dealwithread()`** 来处理数据读取。
- **数据写入**：如果事件类型是 `EPOLLOUT`，说明可以向客户端发送数据，调用 **`dealwithwrite()`** 来处理数据发送。



## [dealclinetdata函数](https://github.com/qinguoyi/TinyWebServer/blob/4bcf88762f85135e9bd46a1032e815e458de6f2e/webserver.cpp#L200)

`dealclientdata()` 函数是 `WebServer` 类中用来处理新的客户端连接请求的方法。

这个函数使用 `accept()` 来接受来自客户端的连接请求，并根据当前服务器的触发模式进行相应的处理。这个方法分两种情况，**一种是水平触发（LT），另一种是边缘触发（ET）。**

在代码中的实现***具体体现在是否需要循环调用 `accept()` 来处理所有挂起的连接请求。***

### 1 不同触发模式的处理：

- **水平触发（LT）模式**：
  - 在水平触发模式下，只要监听套接字上有挂起的连接，`accept()` 就会被调用。如果没有更多的连接请求，`accept()` 会返回 `-1` 并设置 `errno` 为 `EAGAIN` 或 `EWOULDBLOCK`，表示没有更多数据可以读取。
- **边缘触发（ET）模式**：
  - 在边缘触发模式下，只有在监听套接字状态发生变化时（即新连接到达时），`accept()` 才会被调用。因此，必须循环调用 `accept()` 直到返回 `-1` 并且 `errno` 设置为 `EAGAIN` 或 `EWOULDBLOCK`，以确保处理所有挂起的连接请求。如果处理过程中 `accept()` 因为错误而失败，会记录错误信息并退出循环。

### 2 返回值：

- 函数返回 `true` 或 `false` 表示是否成功处理了连接请求。在水平触发模式下，如果成功接受了至少一个连接，则返回 `true`。在边缘触发模式下，由于循环的存在，通常会在处理完所有连接后返回 `false`。





## [dealwithread函数](https://github.com/qinguoyi/TinyWebServer/blob/4bcf88762f85135e9bd46a1032e815e458de6f2e/webserver.cpp#L280)

`dealwithread(int sockfd)` 方法是 `WebServer` 类中用来处理读事件的函数。该方法根据服务器所采用的并发模型**（Reactor 或 Proactor）**来进行相应的操作。此外，还涉及定时器的管理，以处理可能的超时情况。



### 并发模型处理（Reactor 和Proactor）

服务器通过 `m_actormodel` 变量区分并发模型：

- Reactor 模式

  ```
  m_actormodel == 1
  ```

  - **定时器调整**：如果存在定时器，首先对其进行调整，确保连接在活跃时不会超时断开。
  - **事件放入队列**：将读事件放入线程池的任务队列中，由线程池中的某个线程来处理实际的读操作。
  - **状态检查与处理**：通过循环检查 `users[sockfd].improv` 状态，如果标识为 `1`（表示有改进或需要特殊处理），则进行相应的处理。如果设置了定时器标志 `users[sockfd].timer_flag`，则处理并重置定时器和标志。

- Proactor 模式

  ```c++
  m_actormodel == 0
  ```

  - **直接读取数据**：尝试一次性读取所有可用数据（`users[sockfd].read_once()`）。如果读取成功，记录日志并将事件放入处理队列。
  - **定时器调整**：如果存在定时器，根据需要进行调整。
  - **读取失败处理**：如果读取失败（`read_once()` 返回 `false`），则处理定时器（可能是因为连接超时或其他问题导致需要关闭连接）。



### dealwithwrite函数

逻辑与模式与**dealwithread**大致相同



## time函数集

### [timer函数](https://github.com/qinguoyi/TinyWebServer/blob/4bcf88762f85135e9bd46a1032e815e458de6f2e/webserver.cpp#L161)

首先搞清楚，timer在什么时候调用？答案是在accept得到cfd的时候。这时候通过timer函数不只是初始化了cfd的时间，而且整体初始化。
也就是说，当前服务器已经认可了这一连接，完成了三次握手，并且得到了用户标识，允许传输数据。
这里为了提升性能，给到了3倍阈值的超时。



**`timer(int connfd, struct sockaddr_in client_address)`** 方法在 `WebServer` 类中负责为每个新的连接初始化一个定时器，以便管理连接的超时和维护。这是一种常见的做法，用于防止资源泄露并确保服务的健康运行。

#### 方法详解

1. **初始化连接**：
   - 使用 `http_conn` 类的 `init` 方法初始化每个新的连接，设置连接的基本属性如套接字描述符 `connfd`，客户端地址 `client_address`，服务器根目录 `m_root`，连接触发模式 `m_CONNTrigmode`，日志选项 `m_close_log`，数据库用户 `m_user`，密码 `m_passWord`，和数据库名 `m_databaseName`。
2. **设置客户端数据**：
   - 将客户端地址和套接字描述符存储在 `users_timer[connfd]` 结构中。这样做是为了后续能够通过套接字描述符快速访问到对应的客户端数据，以及为每个连接关联定时器。
3. **创建定时器**：
   - 创建一个新的 `util_timer` 对象，并初始化其属性。定时器关联到特定的客户端数据，允许在定时器触发时执行特定的回调函数 `cb_func`。
   - 设置定时器的过期时间为当前时间加上预定的超时时长（`3 * TIMESLOT`），其中 `TIMESLOT` 是一个预定义的时间单位。
4. **注册定时器**：
   - 将新创建的定时器添加到定时器链表 `utils.m_timer_lst` 中。这个链表管理着所有活动的定时器，使得可以在需要时遍历并处理超时事件。



### adjust_timer与deal_timer



也是区分清楚什么时候调用。
**adjust**可以看到是在产生事件之后，我为了还能传输数据，再给你刷新一下你的时间或者给你延长；

而**deal**则是对于坏连接的一个处理。
如果你对此不理解，可以跳过**，你只需要知道what it is。**至于how and why我会在time部分详细解析。

```c++
void WebServer::adjust_timer(util_timer *timer)
{
    time_t cur = time(NULL);
    timer->expire = cur + 3 * TIMESLOT;
    utils.m_timer_lst.adjust_timer(timer);

    LOG_INFO("%s", "adjust timer once");

}
```

```c++
void WebServer::deal_timer(util_timer *timer, int sockfd)
{
    timer->cb_func(&users_timer[sockfd]);
    if (timer)
    {
        utils.m_timer_lst.del_timer(timer);
    }
    LOG_INFO("close fd %d", users_timer[sockfd].sockfd);
}

```

​	
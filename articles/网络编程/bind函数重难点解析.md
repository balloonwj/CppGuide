# bind 函数重难点解析

## bind 函数如何选择绑定地址

bind 函数的基本用法如下：

```
struct sockaddr_in bindaddr;
bindaddr.sin_family = AF_INET;
bindaddr.sin_addr.s_addr = htonl(INADDR_ANY);
bindaddr.sin_port = htons(3000);
if (bind(listenfd, (struct sockaddr *)&bindaddr, sizeof(bindaddr)) == -1)
{
    std::cout << "bind listen socket error." << std::endl;
    return -1;
}
```

其中 bind 的地址我们使用了一个宏叫 **INADDR_ANY** ，关于这个宏的解释如下：

```
If an application does not care what local address is assigned, 
specify the constant value INADDR_ANY for an IPv4 local address
or the constant value in6addr_any for an IPv6 local address 
in the sa_data member of the name parameter. This allows the 
underlying service provider to use any appropriate network address,
potentially simplifying application programming in the presence of 
multihomed hosts (that is, hosts that have more than one network 
interface and address).
```

意译一下：

```
如果应用程序不关心bind绑定的ip地址，可以使用INADDR_ANY(如果是IPv6，
则对应in6addr_any)，这样底层的（协议栈）服务会自动选择一个合适的ip地址，
这样使在一个有多个网卡机器上选择ip地址问题变得简单。
```

也就是说 **INADDR_ANY** 相当于地址 **0.0.0.0**。可能读者还是不太明白我想表达什么。这里我举个例子，假设我们在一台机器上开发一个服务器程序，使用 bind 函数时，我们有多个ip 地址可以选择。首先，这台机器对外访问的ip地址是**120.55.94.78**，这台机器在当前局域网的地址是**192.168.1.104**；同时这台机器有本地回环地址**127.0.0.1**。

如果你指向本机上可以访问，那么你 bind 函数中的地址就可以使用**127.0.0.1**; 如果你的服务只想被局域网内部机器访问，bind 函数的地址可以使用**192.168.1.104**；如果 希望这个服务可以被公网访问，你就可以使用地址**0.0.0.0**或 **INADDR_ANY**。

## bind 函数端口号问题

网络通信程序的基本逻辑是客户端连接服务器，即从客户端的**地址:端口**连接到服务器**地址:端口**上，以 4.2 小节中的示例程序为例，服务器端的端口号使用 3000，那客户端连接时的端口号是多少呢？TCP 通信双方中一般服务器端端口号是固定的，而客户端端口号是连接发起时由操作系统随机分配的（不会分配已经被占用的端口）。端口号是一个 C short 类型的值，其范围是0～65535，知道这点很重要，所以我们在编写压力测试程序时，由于端口数量的限制，在某台机器上网卡地址不变的情况下压力测试程序理论上最多只能发起六万五千多个连接。注意我说的是理论上，在实际情况下，由于当时的操作系统很多端口可能已经被占用，实际可以使用的端口比这个更少，例如，一般规定端口号在1024以下的端口是保留端口，不建议用户程序使用。而对于 Windows 系统，MSDN 甚至明确地说：

> On Windows Vista and later, the dynamic client port range is a value between 49152 and 65535. This is a change from Windows Server 2003 and earlier where the dynamic client port range was a value between 1025 and 5000.
> Vista 及以后的Windows，可用的动态端口范围是49152～65535，而 Windows Server及更早的系统，可以的动态端口范围是1025~5000。（你可以通过修改注册表来改变这一设置，参考网址：https://docs.microsoft.com/en-us/windows/desktop/api/winsock/nf-winsock-bind）

如果将 bind 函数中的端口号设置成0，那么操作系统会随机给程序分配一个可用的侦听端口，当然服务器程序一般不会这么做，因为服务器程序是要对外服务的，必须让客户端知道确切的ip地址和端口号。

很多人觉得只有服务器程序可以调用 bind 函数绑定一个端口号，其实不然，在一些特殊的应用中，我们需要客户端程序以指定的端口号去连接服务器，此时我们就可以在客户端程序中调用 bind 函数绑定一个具体的端口。

我们用代码来实际验证一下上路所说的，为了能看到连接状态，我们将客户端和服务器关闭socket的代码注释掉，这样连接会保持一段时间。

- **情形一：客户端代码不绑定端口**

修改后的服务器代码如下：

```
/**
 * TCP服务器通信基本流程
 * zhangyl 2018.12.13
 */
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>
#include <vector>

int main(int argc, char* argv[])
{
    //1.创建一个侦听socket
    int listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd == -1)
    {
        std::cout << "create listen socket error." << std::endl;
        return -1;
    }

    //2.初始化服务器地址
    struct sockaddr_in bindaddr;
    bindaddr.sin_family = AF_INET;
    bindaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    bindaddr.sin_port = htons(3000);
    if (bind(listenfd, (struct sockaddr *)&bindaddr, sizeof(bindaddr)) == -1)
    {
        std::cout << "bind listen socket error." << std::endl;
        return -1;
    }

    //3.启动侦听
    if (listen(listenfd, SOMAXCONN) == -1)
    {
        std::cout << "listen error." << std::endl;
        return -1;
    }

    //记录所有客户端连接的容器
    std::vector<int> clientfds;
    while (true)
    {
        struct sockaddr_in clientaddr;
        socklen_t clientaddrlen = sizeof(clientaddr);
        //4. 接受客户端连接
        int clientfd = accept(listenfd, (struct sockaddr *)&clientaddr, &clientaddrlen);
        if (clientfd != -1)
        {             
            char recvBuf[32] = {0};
            //5. 从客户端接受数据
            int ret = recv(clientfd, recvBuf, 32, 0);
            if (ret > 0) 
            {
                std::cout << "recv data from client, data: " << recvBuf << std::endl;
                //6. 将收到的数据原封不动地发给客户端
                ret = send(clientfd, recvBuf, strlen(recvBuf), 0);
                if (ret != strlen(recvBuf))
                    std::cout << "send data error." << std::endl;

                std::cout << "send data to client successfully, data: " << recvBuf << std::endl;
            } 
            else 
            {
                std::cout << "recv data error." << std::endl;
            }

            //close(clientfd);
            clientfds.push_back(clientfd);
        }
    }

    //7.关闭侦听socket
    close(listenfd);

    return 0;
}
```

修改后的客户端代码如下：

```
/**
 * TCP客户端通信基本流程
 * zhangyl 2018.12.13
 */
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>

#define SERVER_ADDRESS "127.0.0.1"
#define SERVER_PORT     3000
#define SEND_DATA       "helloworld"

int main(int argc, char* argv[])
{
    //1.创建一个socket
    int clientfd = socket(AF_INET, SOCK_STREAM, 0);
    if (clientfd == -1)
    {
        std::cout << "create client socket error." << std::endl;
        return -1;
    }

    //2.连接服务器
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    serveraddr.sin_port = htons(SERVER_PORT);
    if (connect(clientfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr)) == -1)
    {
        std::cout << "connect socket error." << std::endl;
        return -1;
    }

    //3. 向服务器发送数据
    int ret = send(clientfd, SEND_DATA, strlen(SEND_DATA), 0);
    if (ret != strlen(SEND_DATA))
    {
        std::cout << "send data error." << std::endl;
        return -1;
    }

    std::cout << "send data successfully, data: " << SEND_DATA << std::endl;

    //4. 从客户端收取数据
    char recvBuf[32] = {0};
    ret = recv(clientfd, recvBuf, 32, 0);
    if (ret > 0) 
    {
        std::cout << "recv data successfully, data: " << recvBuf << std::endl;
    } 
    else 
    {
        std::cout << "recv data error, data: " << recvBuf << std::endl;
    }

    //5. 关闭socket
    //close(clientfd);
    //这里仅仅是为了让客户端程序不退出
    while (true) 
    {
        sleep(3);
    }

    return 0;
}
```

将程序编译好后（编译方法和上文一样），我们先启动server，再启动三个客户端。然后通过 **lsof** 命令查看当前机器上的 TCP 连接信息，为了更清楚地显示结果，已经将不相关的连接信息去掉了，结果如下所示：

```
[root@localhost ~]# lsof -i -Pn
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
server   1445 root    3u  IPv4  21568      0t0  TCP *:3000 (LISTEN)
server   1445 root    4u  IPv4  21569      0t0  TCP 127.0.0.1:3000->127.0.0.1:40818 (ESTABLISHED)
server   1445 root    5u  IPv4  21570      0t0  TCP 127.0.0.1:3000->127.0.0.1:40820 (ESTABLISHED)
server   1445 root    6u  IPv4  21038      0t0  TCP 127.0.0.1:3000->127.0.0.1:40822 (ESTABLISHED)
client   1447 root    3u  IPv4  21037      0t0  TCP 127.0.0.1:40818->127.0.0.1:3000 (ESTABLISHED)
client   1448 root    3u  IPv4  21571      0t0  TCP 127.0.0.1:40820->127.0.0.1:3000 (ESTABLISHED)
client   1449 root    3u  IPv4  21572      0t0  TCP 127.0.0.1:40822->127.0.0.1:3000 (ESTABLISHED)
```

上面的结果显示，**server** 进程（进程 ID 是 **1445**）在 **3000** 端口开启侦听，有三个 **client** 进程（进程 ID 分别是**1447**、**1448**、**1449**）分别通过端口号 **40818**、**40820**、**40822** 连到 **server** 进程上的，作为客户端的一方，端口号是系统随机分配的。

- **情形二：客户端绑定端口号 0**

  服务器端代码保持不变，我们修改下客户端代码：

  ```
  /**
   * TCP服务器通信基本流程
   * zhangyl 2018.12.13
   */
  ```

  ```
  #include <sys/types.h> 
  #include <sys/socket.h>
  #include <arpa/inet.h>
  #include <unistd.h>
  #include <iostream>
  #include <string.h>
  
  #define SERVER_ADDRESS "127.0.0.1"
  #define SERVER_PORT     3000
  #define SEND_DATA       "helloworld"
  
  int main(int argc, char* argv[])
  {
    //1.创建一个socket
    int clientfd = socket(AF_INET, SOCK_STREAM, 0);
    if (clientfd == -1)
    {
        std::cout << "create client socket error." << std::endl;
        return -1;
    }
  
    struct sockaddr_in bindaddr;
    bindaddr.sin_family = AF_INET;
    bindaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    //将socket绑定到0号端口上去
    bindaddr.sin_port = htons(0);
    if (bind(clientfd, (struct sockaddr *)&bindaddr, sizeof(bindaddr)) == -1)
    {
        std::cout << "bind socket error." << std::endl;
        return -1;
    }
  
    //2.连接服务器
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    serveraddr.sin_port = htons(SERVER_PORT);
    if (connect(clientfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr)) == -1)
    {
        std::cout << "connect socket error." << std::endl;
        return -1;
    }
  
    //3. 向服务器发送数据
    int ret = send(clientfd, SEND_DATA, strlen(SEND_DATA), 0);
    if (ret != strlen(SEND_DATA))
    {
        std::cout << "send data error." << std::endl;
        return -1;
    }
  
    std::cout << "send data successfully, data: " << SEND_DATA << std::endl;
  
    //4. 从客户端收取数据
    char recvBuf[32] = {0};
    ret = recv(clientfd, recvBuf, 32, 0);
    if (ret > 0) 
    {
        std::cout << "recv data successfully, data: " << recvBuf << std::endl;
    } 
    else 
    {
        std::cout << "recv data error, data: " << recvBuf << std::endl;
    }
  
    //5. 关闭socket
    //close(clientfd);
    //这里仅仅是为了让客户端程序不退出
    while (true) 
    {
        sleep(3);
    }
  
    return 0;
  }
  ```

  我们再次编译客户端程序，并启动三个 **client** 进程，然后用 **lsof** 命令查看机器上的 TCP 连接情况，结果如下所示：

  ```
  [root@localhost ~]# lsof -i -Pn
  COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
  server   1593 root    3u  IPv4  21807      0t0  TCP *:3000 (LISTEN)
  server   1593 root    4u  IPv4  21808      0t0  TCP 127.0.0.1:3000->127.0.0.1:44220 (ESTABLISHED)
  server   1593 root    5u  IPv4  19311      0t0  TCP 127.0.0.1:3000->127.0.0.1:38990 (ESTABLISHED)
  server   1593 root    6u  IPv4  21234      0t0  TCP 127.0.0.1:3000->127.0.0.1:42365 (ESTABLISHED)
  client   1595 root    3u  IPv4  22626      0t0  TCP 127.0.0.1:44220->127.0.0.1:3000 (ESTABLISHED)
  client   1611 root    3u  IPv4  21835      0t0  TCP 127.0.0.1:38990->127.0.0.1:3000 (ESTABLISHED)
  client   1627 root    3u  IPv4  21239      0t0  TCP 127.0.0.1:42365->127.0.0.1:3000 (ESTABLISHED)
  ```

  通过上面的结果，我们发现三个 **client** 进程使用的端口号仍然是系统随机分配的，也就是说绑定 **0** 号端口和没有绑定效果是一样的。

- **情形三：客户端绑定一个固定端口**

  我们这里使用 **20000** 端口，当然读者可以根据自己的喜好选择，只要保证所选择的端口号当前没有被其他程序占用即可，服务器代码保持不变，客户端绑定代码中的端口号从 **0** 改成 **20000**。这里为了节省篇幅，只贴出修改处的代码：

  ```
  struct sockaddr_in bindaddr;
  bindaddr.sin_family = AF_INET;
  bindaddr.sin_addr.s_addr = htonl(INADDR_ANY);
  //将socket绑定到20000号端口上去
  bindaddr.sin_port = htons(20000);
  if (bind(clientfd, (struct sockaddr *)&bindaddr, sizeof(bindaddr)) == -1)
  {
      std::cout << "bind socket error." << std::endl;
      return -1;
  }
  ```

  再次重新编译程序，先启动一个客户端后，我们看到此时的 TCP 连接状态：

  ```
  [root@localhost testsocket]# lsof -i -Pn
  COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
  server   1676 root    3u  IPv4  21933      0t0  TCP *:3000 (LISTEN)
  server   1676 root    4u  IPv4  21934      0t0  TCP 127.0.0.1:3000->127.0.0.1:20000 (ESTABLISHED)
  client   1678 root    3u  IPv4  21336      0t0  TCP 127.0.0.1:20000->127.0.0.1:3000 (ESTABLISHED)
  ```

  通过上面的结果，我们发现 **client** 进程确实使用 **20000** 号端口连接到 **server** 进程上去了。这个时候如果我们再开启一个 **client** 进程，我们猜想由于端口号 **20000** 已经被占用，新启动的 **client** 会由于调用 **bind** 函数出错而退出，我们实际验证一下：

  ```
  [root@localhost testsocket]# ./client 
  bind socket error.
  [root@localhost testsocket]# 
  ```

  结果确实和我们预想的一样。

在技术面试的时候，有时候面试官会问 TCP 网络通信的客户端程序中的 socket 是否可以调用 bind 函数，相信读到这里，聪明的读者已经有答案了。

另外，Linux 的 **nc** 命令有个 **-p** 选项（字母 **p** 是小写），这个选项的作用就是 **nc** 在模拟客户端程序时，可以使用指定端口号连接到服务器程序上去，实现原理相信读者也明白了。我们还是以上面的服务器程序为例，这个我们不用我们的 **client** 程序，改用 **nc** 命令来模拟客户端。在 **shell** 终端输入：

```
[root@localhost testsocket]# nc -v -p 9999 127.0.0.1 3000
Ncat: Version 6.40 ( http://nmap.org/ncat )
Ncat: Connected to 127.0.0.1:3000.
My name is zhangxf
My name is zhangxf
```

**-v** 选项表示输出 **nc** 命令连接的详细信息，这里连接成功以后，会输出“**Ncat: Connected to 127.0.0.1:3000.**” 提示已经连接到服务器的 **3000** 端口上去了。

**-p** 选项的参数值是 **9999** 表示，我们要求 **nc** 命令本地以端口号 **9999** 连接服务器，注意不要与端口号 **3000** 混淆，**3000** 是服务器的侦听端口号，也就是我们的连接的目标端口号，**9999** 是我们客户端使用的端口号。我们用 **lsof** 命令来验证一下我们的 **nc** 命令是否确实以 **9999** 端口号连接到 **server** 进程上去了。

```
[root@localhost testsocket]# lsof -i -Pn
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
server   1676 root    3u  IPv4  21933      0t0  TCP *:3000 (LISTEN)
server   1676 root    7u  IPv4  22405      0t0  TCP 127.0.0.1:3000->127.0.0.1:9999 (ESTABLISHED)
nc       2005 root    3u  IPv4  22408      0t0  TCP 127.0.0.1:9999->127.0.0.1:3000 (ESTABLISHED)
```

结果确实如我们期望的一致。

当然，我们用 **nc** 命令连接上 **server** 进程以后，我们还给服务器发了一条消息"**My name is zhangxf**"，**server** 程序收到消息后把这条消息原封不动地返还给我们，以下是 **server** 端运行结果：

```
[root@localhost testsocket]# ./server   
recv data from client, data: My name is zhangxf

send data to client successfully, data: My name is zhangxf
```

关于 **lsof** 和 **nc** 命令我们会在后面的系列文章中详细讲解。

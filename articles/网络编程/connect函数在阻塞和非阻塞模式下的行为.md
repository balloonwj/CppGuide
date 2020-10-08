## connect 函数在阻塞和非阻塞模式下的行为

在 socket 是阻塞模式下 connect 函数会一直到有明确的结果才会返回（或连接成功或连接失败），如果服务器地址“较远”，连接速度比较慢，connect 函数在连接过程中可能会导致程序阻塞在 connect 函数处好一会儿（如两三秒之久），虽然这一般也不会对依赖于网络通信的程序造成什么影响，但在实际项目中，我们一般倾向使用所谓的**异步的 connect** 技术，或者叫**非阻塞的 connect**。这个流程一般有如下步骤：

```
1. 创建socket，并将 socket 设置成非阻塞模式；
2. 调用 connect 函数，此时无论 connect 函数是否连接成功会立即返回；如果返回-1并不表示连接出错，如果此时错误码是EINPROGRESS
3. 接着调用 select 函数，在指定的时间内判断该 socket 是否可写，如果可写说明连接成功，反之则认为连接失败。
```

按上述流程编写代码如下：

```
/**
 * 异步的connect写法，nonblocking_connect.cpp
 * zhangyl 2018.12.17
 */
#include <sys/types.h> 
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <iostream>
#include <string.h>
#include <stdio.h>
#include <fcntl.h>
#include <errno.h>

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

    //连接成功以后，我们再将 clientfd 设置成非阻塞模式，
    //不能在创建时就设置，这样会影响到 connect 函数的行为
    int oldSocketFlag = fcntl(clientfd, F_GETFL, 0);
    int newSocketFlag = oldSocketFlag | O_NONBLOCK;
    if (fcntl(clientfd, F_SETFL,  newSocketFlag) == -1)
    {
        close(clientfd);
        std::cout << "set socket to nonblock error." << std::endl;
        return -1;
    }

    //2.连接服务器
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    serveraddr.sin_port = htons(SERVER_PORT);
    for (;;)
    {
        int ret = connect(clientfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr));
        if (ret == 0)
        {
            std::cout << "connect to server successfully." << std::endl;
            close(clientfd);
            return 0;
        } 
        else if (ret == -1) 
        {
            if (errno == EINTR)
            {
                //connect 动作被信号中断，重试connect
                std::cout << "connecting interruptted by signal, try again." << std::endl;
                continue;
            } else if (errno == EINPROGRESS)
            {
                //连接正在尝试中
                break;
            } else {
                //真的出错了，
                close(clientfd);
                return -1;
            }
        }
    }

    fd_set writeset;
    FD_ZERO(&writeset);
    FD_SET(clientfd, &writeset);
    //可以利用tv_sec和tv_usec做更小精度的超时控制
    struct timeval tv;
    tv.tv_sec = 3;  
    tv.tv_usec = 0;
    if (select(clientfd + 1, NULL, &writeset, NULL, &tv) == 1)
    {
        std::cout << "[select] connect to server successfully." << std::endl;
    } else {
        std::cout << "[select] connect to server error." << std::endl;
    }

    //5. 关闭socket
    close(clientfd);

    return 0;
}
```

为了区别到底是在调用 connect 函数时判断连接成功还是通过 select 函数判断连接成功，我们在后者的输出内容中加上了“**[select]**”标签以示区别。

我们先用 **nc** 命令启动一个服务器程序：

```
nc -v -l 0.0.0.0 3000
```

然后编译客户端程序并执行：

```
[root@localhost testsocket]# g++ -g -o nonblocking_connect nonblocking_connect.cpp 
[root@localhost testsocket]# ./nonblocking_connect 
[select] connect to server successfully.
```

我们把服务器程序关掉，再重新启动一下客户端，这个时候应该会连接失败，程序输出结果如下：

```
[root@localhost testsocket]# ./nonblocking_connect 
[select] connect to server successfully.
```

奇怪？为什么连接不上也会得出一样的输出结果？难道程序有问题？这是因为：

- 在 Windows 系统上，一个 socket 没有建立连接之前，我们使用 select 函数检测其是否可写，能得到正确的结果（不可写），连接成功后检测，会变为可写。所以，上述介绍的异步 **connect** 写法流程在 Windows 系统上时没有问题的。
- 在 Linux 系统上一个 socket 没有建立连接之前，用 select 函数检测其是否可写，你也会得到可写得结果，所以上述流程并不适用于 Linux 系统。正确的做法是，connect 之后，不仅要用 **select** 检测可写，还要检测此时 socket 是否出错，通过错误码来检测确定是否连接上，错误码为 0 表示连接上，反之为未连接上。完整代码如下：

```
  /**
   * Linux 下正确的异步的connect写法，linux_nonblocking_connect.cpp
   * zhangyl 2018.12.17
   */
  #include <sys/types.h> 
  #include <sys/socket.h>
  #include <arpa/inet.h>
  #include <unistd.h>
  #include <iostream>
  #include <string.h>
  #include <stdio.h>
  #include <fcntl.h>
  #include <errno.h>

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

      //连接成功以后，我们再将 clientfd 设置成非阻塞模式，
      //不能在创建时就设置，这样会影响到 connect 函数的行为
      int oldSocketFlag = fcntl(clientfd, F_GETFL, 0);
      int newSocketFlag = oldSocketFlag | O_NONBLOCK;
      if (fcntl(clientfd, F_SETFL,  newSocketFlag) == -1)
      {
          close(clientfd);
          std::cout << "set socket to nonblock error." << std::endl;
          return -1;
      }

      //2.连接服务器
      struct sockaddr_in serveraddr;
      serveraddr.sin_family = AF_INET;
      serveraddr.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
      serveraddr.sin_port = htons(SERVER_PORT);
      for (;;)
      {
          int ret = connect(clientfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr));
          if (ret == 0)
          {
              std::cout << "connect to server successfully." << std::endl;
              close(clientfd);
              return 0;
          } 
          else if (ret == -1) 
          {
              if (errno == EINTR)
              {
                  //connect 动作被信号中断，重试connect
                  std::cout << "connecting interruptted by signal, try again." << std::endl;
                  continue;
              } else if (errno == EINPROGRESS)
              {
                  //连接正在尝试中
                  break;
              } else {
                  //真的出错了，
                  close(clientfd);
                  return -1;
              }
          }
      }

      fd_set writeset;
      FD_ZERO(&writeset);
      FD_SET(clientfd, &writeset);
      //可以利用tv_sec和tv_usec做更小精度的超时控制
      struct timeval tv;
      tv.tv_sec = 3;  
      tv.tv_usec = 0;
      if (select(clientfd + 1, NULL, &writeset, NULL, &tv) != 1)
      {
          std::cout << "[select] connect to server error." << std::endl;
          close(clientfd);
          return -1;
      }

      int err;
      socklen_t len = static_cast<socklen_t>(sizeof err);
      if (::getsockopt(clientfd, SOL_SOCKET, SO_ERROR, &err, &len) < 0)
      {
          close(clientfd);
          return -1;
      }

      if (err == 0)
          std::cout << "connect to server successfully." << std::endl;
      else
          std::cout << "connect to server error." << std::endl;

      //5. 关闭socket
      close(clientfd);

      return 0;
  }
```

> 当然，在实际的项目中，第 3 个步骤中 Linux 平台上你也可以使用 **poll** 函数来判断 socket 是否可写；在 Windows 平台上你可以使用 **WSAEventSelect** 或 **WSAAsyncSelect** 函数判断连接是否成功，关于这三个函数我们将在后面的章节中详细讲解，这里暂且仅以 **select** 函数为例。

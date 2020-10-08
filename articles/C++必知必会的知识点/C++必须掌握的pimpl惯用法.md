# pimpl 惯用法

现在这里有一个名为 **CSocketClient** 的网络通信类，定义如下：

```
/**
 * 网络通信的基础类, SocketClient.h
 * zhangyl 2017.07.11
 */
class CSocketClient
{
public:
    CSocketClient();
    ~CSocketClient();
 
public:  
    void SetProxyWnd(HWND hProxyWnd);

    bool    Init(CNetProxy* pNetProxy);
    bool    Uninit();
    
    int Register(const char* pszUser, const char* pszPassword); 
    void GuestLogin();  
    
    BOOL    IsClosed();
    BOOL	Connect(int timeout = 3);
    void    AddData(int cmd, const std::string& strBuffer);
    void    AddData(int cmd, const char* pszBuff, int nBuffLen);
    void    Close();

    BOOL    ConnectServer(int timeout = 3);
    BOOL    SendLoginMsg();
    BOOL    RecvLoginMsg(int& nRet);
    BOOL    Login(int& nRet);

private:
    void LoadConfig();
    static UINT CALLBACK SendDataThreadProc(LPVOID lpParam);
    static UINT CALLBACK RecvDataThreadProc(LPVOID lpParam);
    bool Send();
    bool Recv();
    bool CheckReceivedData();
    void SendHeartbeatPackage();

private:
    SOCKET                          m_hSocket;
    short                           m_nPort;
    char                            m_szServer[64];
    long                            m_nLastDataTime;        //最近一次收发数据的时间
    long                            m_nHeartbeatInterval;   //心跳包时间间隔，单位秒
    CRITICAL_SECTION                m_csLastDataTime;       //保护m_nLastDataTime的互斥体 
    HANDLE                          m_hSendDataThread;      //发送数据线程
    HANDLE                          m_hRecvDataThread;      //接收数据线程
    std::string                     m_strSendBuf;
    std::string                     m_strRecvBuf;
    HANDLE                          m_hExitEvent;
    bool                            m_bConnected;
    CRITICAL_SECTION                m_csSendBuf;
    HANDLE                          m_hSemaphoreSendBuf;
    HWND                            m_hProxyWnd;
    CNetProxy*                      m_pNetProxy;
    int                             m_nReconnectTimeInterval;    //重连时间间隔
    time_t                          m_nLastReconnectTime;        //上次重连时刻
    CFlowStatistics*                m_pFlowStatistics;
};
```

这段代码来源于笔者实际项目中开发的一个股票客户端的软件。

**CSocketClient** 类的 **public** 方法提供对外接口供第三方使用，每个函数的具体实现在 **SocketClient.cpp** 中，对第三方使用者不可见。在 Windows 系统上作为提供给第三方使用的库，一般需要提供给使用者 ***.h**、***.lib** 和 ***.dll** 文件，在 Linux 系统上需要提供 ***.h**、***.a  或 *.so** 文件。

不管是在哪个操作系统平台上，像 SocketClient.h 这样的头文件提供给第三方使用时，都会让库的作者心里**隐隐不安**——因为 SocketClient.h 文件中 SocketClient 类大量的成员变量和私有函数暴露了这个类太多的实现细节，很容易让使用者看出实现原理。这样的头文件，对于一些不想对使用者暴露核心技术实现的库和 sdk，是非常不好的。

那有没有什么办法既能保持对外的接口不变，又能尽量不暴露一些关键性的成员变量和私有函数的实现方法呢？有的。我们可以将代码稍微修改一下：

```
/**
 * 网络通信的基础类, SocketClient.h
 * zhangyl 2017.07.11
 */
class Impl;

class CSocketClient
{
public:
    CSocketClient();
    ~CSocketClient();
 
public:
    void SetProxyWnd(HWND hProxyWnd);

    bool    Init(CNetProxy* pNetProxy);
    bool    Uninit();

    int Register(const char* pszUser, const char* pszPassword);    
    void GuestLogin();  
    
    BOOL    IsClosed();
    BOOL	Connect(int timeout = 3);
    void    AddData(int cmd, const std::string& strBuffer);
    void    AddData(int cmd, const char* pszBuff, int nBuffLen);
    void    Close();

    BOOL    ConnectServer(int timeout = 3);
    BOOL    SendLoginMsg();
    BOOL    RecvLoginMsg(int& nRet);
    BOOL    Login(int& nRet);

private:
    Impl*	m_pImpl;
};
```

上述代码中，所有的关键性成员变量已经没有了，取而代之的是一个类型为 **Impl** 的指针成员变量 **m_pImpl**。

> 具体采用什么名称，读者完全可以根据自己的实际情况来定，不一定非要使用这里的 **Impl** 和 **m_pImpl**。

**Impl** 类型现在是完全对使用者透明，为了在当前类中可以使用 **Impl**，使用了一个前置声明：

```
//原代码第5行
class Impl;
```

然后我们就可以将刚才隐藏的成员变量放到这个类中去：

```
class Impl
{
public:
	Impl()
	{
        //TODO: 你可以在这里对成员变量做一些初始化工作
	}
	
	~Impl()
	{
        //TODO: 你可以在这里做一些清理工作
	}
	
public:
	SOCKET                          m_hSocket;
    short                           m_nPort;
    char                            m_szServer[64];
    long                            m_nLastDataTime;        //最近一次收发数据的时间
    long                            m_nHeartbeatInterval;   //心跳包时间间隔，单位秒
    CRITICAL_SECTION                m_csLastDataTime;       //保护m_nLastDataTime的互斥体 
    HANDLE                          m_hSendDataThread;      //发送数据线程
    HANDLE                          m_hRecvDataThread;      //接收数据线程
    std::string                     m_strSendBuf;
    std::string                     m_strRecvBuf;
    HANDLE                          m_hExitEvent;
    bool                            m_bConnected;
    CRITICAL_SECTION                m_csSendBuf;
    HANDLE                          m_hSemaphoreSendBuf;
    HWND                            m_hProxyWnd;
    CNetProxy*                      m_pNetProxy;
    int                             m_nReconnectTimeInterval;    //重连时间间隔
    time_t                          m_nLastReconnectTime;        //上次重连时刻
    CFlowStatistics*                m_pFlowStatistics;
};
```

接着我们在 **CSocketClient** 的构造函数中创建这个 **m_pImpl** 对象，在 **CSocketClient** 析构函数中释放这个对象。

```
CSocketClient::CSocketClient()
{
	m_pImpl = new Impl();
}

CSocketClient::~CSocketClient()
{
	delete m_pImpl;
}
```

这样，原来需要引用的成员变量，可以在 **CSocketClient** 内部使用 **m_pImpl->变量名** 来引用了。 

> 这里仅仅以演示隐藏 **CSocketClient** 的成员变量为例，隐藏其私有方法与此类似，都是变成类 **Impl** 的方法。

需要强调的是，在实际开发中，由于 **Impl** 类是 **CSocketClient** 的辅助类， **Impl** 类没有独立存在的必要，所以一般会将 **Impl** 类定义成 **CSocketClient** 的内部类。即采用如下形式：

```
/**
 * 网络通信的基础类, SocketClient.h
 * zhangyl 2017.07.11
 */
class CSocketClient
{
public:
    CSocketClient();
    ~CSocketClient();

 //重复的代码省略...

private:
	class   Impl;
    Impl*	m_pImpl;
};
```

然后在 **ClientSocket.cpp** 中定义 **Impl** 类的实现：

```
/**
 * 网络通信的基础类, SocketClient.cpp
 * zhangyl 2017.07.11
 */
class  CSocketClient::Impl
{
public:
    void LoadConfig()
    {
    	//方法的具体实现
    }
    
    //其他方法省略...
    
public:
	SOCKET                          m_hSocket;
    short                           m_nPort;
    char                            m_szServer[64];
    long                            m_nLastDataTime;        //最近一次收发数据的时间
    long                            m_nHeartbeatInterval;   //心跳包时间间隔，单位秒
    CRITICAL_SECTION                m_csLastDataTime;       //保护m_nLastDataTime的互斥体 
    HANDLE                          m_hSendDataThread;      //发送数据线程
    HANDLE                          m_hRecvDataThread;      //接收数据线程
    std::string                     m_strSendBuf;
    std::string                     m_strRecvBuf;
    HANDLE                          m_hExitEvent;
    bool                            m_bConnected;
    CRITICAL_SECTION                m_csSendBuf;
    HANDLE                          m_hSemaphoreSendBuf;
    HWND                            m_hProxyWnd;
    CNetProxy*                      m_pNetProxy;
    int                             m_nReconnectTimeInterval;    //重连时间间隔
    time_t                          m_nLastReconnectTime;        //上次重连时刻
    CFlowStatistics*                m_pFlowStatistics;
}
 
CSocketClient::CSocketClient()
{
	m_pImpl = new Impl();
}

CSocketClient::~CSocketClient()
{
	delete m_pImpl;
}
```

现在**CSocketClient** 这个类除了保留对外的接口以外，其内部实现用到的变量和方法基本上对使用者不可见了。C++ 中对类的这种封装方式，我们称之为 **pimpl** 惯用法，即 **Pointer to Implementation** （也有人认为是 **Private Implementation**）。

> 在实际的开发中，**Impl** 类的声明和定义既可以使用 **class** 关键字也可以使用 **struct** 关键字。在 C++ 语言中，struct 类型可以定义成员方法，但 struct 所有成员变量和方法默认都是 public 的。

现在来总结一下这个方法的优点：

- 核心数据成员被隐藏；

    核心数据成员被隐藏，不必暴露在头文件中，对使用者透明，提高了安全性。

- 降低编译依赖，提高编译速度；

    由于原来的头文件的一些私有成员变量可能是非指针非引用类型的自定义类型，需要在当前类的头文件中包含这些类型的头文件，使用了 **pimpl** 惯用法以后，这些私有成员变量被移动到当前类的 cpp 文件中，因此头文件不再需要包含这些成员变量的类型头文件，当前头文件变“干净”，这样其他文件在引用这个头文件时，依赖的类型变少，加快了编译速度。

- 接口与实现分离。

    使用了 **pimpl** 惯用法之后，即使 **CSocketClient** 或者 **Impl** 类的实现细节发生了变化，对使用者都是透明的，对外的 **CSocketClient** 类声明仍然可以保持不变。例如我们可以增删改 Impl 的成员变量和成员方法而保持 **SocketClient.h** 文件内容不变；如果不使用 **pimpl** 惯用法，我们做不到不改变 **SocketClient.h** 文件而增删改 **CSocketClient** 类的成员。



**智能指针用于 pimpl 惯用法**

C++ 11 标准引入了智能指针对象，我们可以使用 **std::unique_ptr** 对象来管理上述用于隐藏具体实现的 **m_pImpl** 指针。

**SocketClient.h** 文件可以修改成如下方式：

```
#include <memory> //for std::unique_ptr  

class CSocketClient
{
public:
    CSocketClient();
    ~CSocketClient();

    //重复的代码省略...

private:
    struct                  Impl;
    std::unique_ptr<Impl>   m_pImpl;
};
```

**SocketClient.cpp** 中修改 **CSocketClient** 对象的构造函数和析构函数的实现如下：

**构造函数**

如果你的编译器仅支持 C++ 11 标准，我们可以按如下修改：

```
CSocketClient::CSocketClient()
{
    //C++11 标准并未提供 std::make_unique()，该方法是 C++14 提供的
    m_pImpl.reset(new Impl());
}
```

如果你的编译器支持 C++14 及以上标准，可以这么修改：

```
CSocketClient::CSocketClient() : m_pImpl(std::make_unique<Impl>())
{    
}
```

由于已经使用了智能指针来管理 m_pImpl 指向的堆内存，因此析构函数中不再需要显式释放堆内存：

```
CSocketClient::~CSocketClient()
{
    //不再需要显式 delete 了 
    //delete m_pImpl;
}
```



**pimp** 惯用法是 C/C++ 项目开发中一种非常实用的代码编写策略，建议读者掌握它。
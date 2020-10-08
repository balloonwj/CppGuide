# C++ 高性能服务器网络框架设计细节

这篇文章我们将介绍服务器的开发，并从多个方面探究如何开发一款高性能高并发的服务器程序。需要注意的是一般大型服务器，其复杂程度在于其业务，而不是在于其代码工程的基本框架。大型服务器一般有多个服务组成，可能会支持CDN，或者支持所谓的“分布式”等，这篇文章不会介绍这些东西，因为不管结构多么复杂的服务器，都是由单个服务器组成的。所以这篇文章的侧重点是讨论单个服务程序的结构，而且这里的结构指的也是单个服务器的网络通信层结构，如果你能真正地理解了我所说的，那么在这个基础的结构上面开展任何业务都是可以的，也可以将这种结构扩展成复杂的多个服务器组，例如“分布式”服务。文中的代码示例虽然是以C++为例，但同样适合Java（我本人也是Java开发者），原理都是一样的，只不过Java可能在基本的操作系统网络通信API的基础上用虚拟机包裹了一层接口而已（Java甚至可能基于一些常用的网络通信框架思想提供了一些现成的API，例如NIO）。有鉴于此，这篇文章不讨论那些大而空、泛泛而谈的技术术语，而是讲的是实实在在的能指导读者在实际工作中实践的编码方案或优化已有编码的方法。另外这里讨论的技术同时涉及windows和linux两个平台。

所谓高性能就是服务器能流畅地处理各个客户端的连接并尽量低延迟地应答客户端的请求；所谓高并发，不仅指的是服务器可以同时支持多的客户端连接，而且这些客户端在连接期间内会不断与服务器有数据来往。网络上经常有各种网络库号称单个服务能同时支持百万甚至千万的并发，然后我实际去看了下，结果发现只是能同时支持很多的连接而已。如果一个服务器能单纯地接受ｎ个连接（ｎ可能很大），但是不能有条不紊地处理与这些连接之间的数据来往也没有任何意义，这种服务器框架只是“玩具型”的，对实际生产和应用没有任何意义。

这篇文章将从两个方面来介绍，一个是服务器中的基础的网络通信部件；另外一个是，如何利用这些基础通信部件整合成一个完整的高效的服务器框架。注意：本文以下内容中的客户端是相对概念，指的是连接到当前讨论的服务程序的终端，所以这里的客户端既可能是我们传统意义上的客户端程序，也可能是连接该服务的其他服务器程序。

一、网络通信部件
 按上面介绍的思路，我们先从服务程序的网络通信部件开始介绍。

（一）、需要解决的问题
既然是服务器程序肯定会涉及到网络通信部分，那么服务器程序的网络通信模块要解决哪些问题？目前，网络上有很多网络通信框架，如libevent、boost asio、ACE，但都网络通信的常见的技术手段都大同小异，至少要解决以下问题：
 - 如何检测有新客户端连接？
 - 如何接受客户端连接？
 - 如何检测客户端是否有数据发来？
 - 如何收取客户端发来的数据？
 - 如何检测连接异常？发现连接异常之后，如何处理？
 - 如何给客户端发送数据？
 - 如何在给客户端发完数据后关闭连接？

稍微有点网络基础的人，都能回答上面说的其中几个问题，比如接收客户端连接用socket API的accept函数，收取客户端数据用recv函数，给客户端发送数据用send函数，检测客户端是否有新连接和客户端是否有新数据可以用IO multiplexing技术（IO复用）的select、poll、epoll等socket API。确实是这样的，这些基础的socket API构成了服务器网络通信的地基，不管网络通信框架设计的如何巧妙，都是在这些基础的socket API的基础上构建的。但是如何巧妙地组织这些基础的socket API，才是问题的关键。我们说服务器很高效，支持高并发，实际上只是一个技术实现手段，不管怎样，从软件开发的角度来讲无非就是一个程序而已，所以，只要程序能最大可能地满足“尽量减少等待或者不等待”这一原则就是高效的，也就是说高效不是“忙的忙死，闲的闲死”，而是大家都可以闲着，但是如果有活要干，大家尽量一起干，而不是一部分忙着依次做事情123456789，另外一部分闲在那里无所事事。说的可能有点抽象，下面我们来举一些例子具体来说明一下。
例如：
 - 默认情况下，recv函数如果没有数据的时候，线程就会阻塞在那里；
 - 默认情况下，send函数，如果tcp窗口不是足够大，数据发不出去也会阻塞在那里；
 - connect函数默认连接另外一端的时候，也会阻塞在那里；
 - 又或者是给对端发送一份数据，需要等待对端回答，如果对方一直不应答，当前线程就阻塞在这里。

以上都不是高效服务器的开发思维方式，因为上面的例子都不满足“尽量减少等待”的原则，为什么一定要等待呢？有没用一种方法，这些过程不需要等待，最好是不仅不需要等待，而且这些事情完成之后能通知我。这样在这些本来用于等待的cpu时间片内，我就可以做一些其他的事情。有，也就是我们下文要讨论的IO Multiplexing技术（IO复用技术）。
            
（二）、几种IO复用机制的比较
目前windows系统支持select、WSAAsyncSelect、WSAEventSelect、完成端口（IOCP），linux系统支持select、poll、epoll。这里我们不具体介绍每个具体的函数的用法，我们来讨论一点深层次的东西，以上列举的API函数可以分为两个层次：

* 层次一 select和poll
* 层次二 WSAAsyncSelect、WSAEventSelect、完成端口（IOCP）、epoll

为什么这么分呢？先来介绍第一层次，select和poll函数本质上还是在一定时间内主动去查询socket句柄（可能是一个也可能是多个）上是否有事件，比如可读事件，可写事件或者出错事件，也就是说我们还是需要每隔一段时间内去主动去做这些检测，如果在这段时间内检测出一些事件来，我们这段时间就算没白花，但是倘若这段时间内没有事件呢？我们只能是做无用功了，说白了，还是在浪费时间，因为假如一个服务器有多个连接，在cpu时间片有限的情况下，我们花费了一定的时间检测了一部分socket连接，却发现它们什么事件都没有，而在这段时间内我们却有一些事情需要处理，那我们为什么要花时间去做这个检测呢？把这个时间用在做我们需要做的事情不好吗？所以对于服务器程序来说，要想高效，我们应该尽量避免花费时间主动去查询一些socket是否有事件，而是等这些socket有事件的时候告诉我们去处理。这也就是层次二的各个函数做的事情，它们实际相当于变主动查询是否有事件为当有事件时，系统会告诉我们，此时我们再去处理，也就是“好钢用在刀刃”上了。只不过层次二的函数通知我们的方式是各不相同，比如WSAAsyncSelect是利用windows窗口消息队列的事件机制来通知我们设定的窗口过程函数，IOCP是利用GetQueuedCompletionStatus返回正确的状态，epoll是epoll_wait函数返回而已。

例如，connect函数连接另外一端，如果用于连接socket是非阻塞的，那么connect虽然不能立刻连接完成，但是也是会立刻返回，无需等待，等连接完成之后，WSAAsyncSelect会返回FD_CONNECT事件告诉我们连接成功，epoll会产生EPOLLOUT事件，我们也能知道连接完成。甚至socket有数据可读时，WSAAsyncSelect产生FD_READ事件，epoll产生EPOLLIN事件，等等。所以有了上面的讨论，我们就可以得到网络通信检测可读可写或者出错事件的正确姿势。这是我这里提出的第二个原则：尽量减少做无用功的时间。这个在服务程序资源够用的情况下可能体现不出来什么优势，但是如果有大量的任务要处理，这里就成了性能的一个瓶颈。
      
（三）、检测网络事件的正确姿势
根据上面的介绍，第一，为了避免无意义的等待时间，第二，不采用主动查询各个socket的事件，而是采用等待操作系统通知我们有事件的状态的策略。我们的socket都要设置成非阻塞的。在此基础上我们回到栏目（一）中提到的七个问题：

1. 如何检测有新客户端连接？
2. 如何接受客户端连接？
    默认accept函数会阻塞在那里，如果epoll检测到侦听socket上有EPOLLIN事件，或者WSAAsyncSelect检测到有FD_ACCEPT事件，那么就表明此时有新连接到来，这个时候调用accept函数，就不会阻塞了。当然产生的新socket你应该也设置成非阻塞的。这样我们就能在新socket上收发数据了。
    　　
3. 如何检测客户端是否有数据发来？
4. 如何收取客户端发来的数据？
    同理，我们也应该在socket上有可读事件的时候才去收取数据，这样我们调用recv或者read函数时不用等待，至于一次性收多少数据好呢？我们可以根据自己的需求来决定，甚至你可以在一个循环里面反复recv或者read，对于非阻塞模式的socket，如果没有数据了，recv或者read也会立刻返回，错误码EWOULDBLOCK会表明当前已经没有数据了。示例：

    bool CIUSocket::Recv()
    {
    	int nRet = 0;
        while(true)
        {
            char buff[512];
            nRet = ::recv(m_hSocket, buff, 512, 0);
            if(nRet == SOCKET_ERROR)
            {
                if (::WSAGetLastError() == WSAEWOULDBLOCK)
                   break; 
                else
                    return false;
            }
            else if(nRet < 1)
                return false;
    
            m_strRecvBuf.append(buff, nRet);
    
            ::Sleep(1);
        } 
    
    	return true;
    }

5. 如何检测连接异常？发现连接异常之后，如何处理？
    同样当我们收到异常事件后例如EPOLLERR或关闭事件FD_CLOSE，我们就知道了有异常产生，我们对异常的处理一般就是关闭对应的socket。另外，如果send/recv或者read/write函数对一个socket进行操作时，如果返回0，那说明对端已经关闭了socket，此时这路连接也没必要存在了，我们也可以关闭对应的socket。

6. 如何给客户端发送数据？
    这也是一道常见的网络通信面试题，某一年的腾讯后台开发职位就问到过这样的问题。给客户端发送数据，比收数据要稍微麻烦一点，也是需要讲点技巧的。首先我们不能像注册检测数据可读事件一样一开始就注册检测数据可写事件，因为如果检测可写的话，一般情况下只要对端正常收取数据，我们的socket就都是可写的，如果我们设置监听可写事件，会导致频繁地触发可写事件，但是我们此时并不一定有数据需要发送。所以正确的做法是：如果有数据要发送，则先尝试着去发送，如果发送不了或者只发送出去部分，剩下的我们需要将其缓存起来，然后再设置检测该socket上可写事件，下次可写事件产生时，再继续发送，如果还是不能完全发出去，则继续设置侦听可写事件，如此往复，一直到所有数据都发出去为止。一旦所有数据都发出去以后，我们要移除侦听可写事件，避免无用的可写事件通知。不知道你注意到没有，如果某次只发出去部分数据，剩下的数据应该暂且存起来，这个时候我们就需要一个缓冲区来存放这部分数据，这个缓冲区我们称为“发送缓冲区”。发送缓冲区不仅存放本次没有发完的数据，还用来存放在发送过程中，上层又传来的新的需要发送的数据。为了保证顺序，新的数据应该追加在当前剩下的数据的后面，发送的时候从发送缓冲区的头部开始发送。也就是说先来的先发送，后来的后发送。
    　　
7. 如何在给客户端发完数据后关闭连接？
    这个问题比较难处理，因为这里的“发送完”不一定是真正的发送完，我们调用send或者write函数即使成功，也只是向操作系统的协议栈里面成功写入数据，至于能否被发出去、何时被发出去很难判断，发出去对方是否收到就更难判断了。所以，我们目前只能简单地认为send或者write返回我们发出数据的字节数大小，我们就认为“发完数据”了。然后调用close等socket API关闭连接。当然，你也可以调用shutdown函数来实现所谓的“半关闭”。关于关闭连接的话题，我们再单独开一个小的标题来专门讨论一下。

（四）被动关闭连接和主动关闭连接
在实际的应用中，被动关闭连接是由于我们检测到了连接的异常事件，比如EPOLLERR，或者对端关闭连接，send或recv返回0，这个时候这路连接已经没有存在必要的意义了，我们被迫关闭连接。

而主动关闭连接，是我们主动调用close/closesocket来关闭连接。比如客户端给我们发送非法的数据，比如一些网络攻击的尝试性数据包。这个时候出于安全考虑，我们关闭socket连接。
       

（五）发送缓冲区和接收缓冲区
上面已经介绍了发送缓冲区了，并说明了其存在的意义。接收缓冲区也是一样的道理，当收到数据以后，我们可以直接进行解包，但是这样并不好，理由一：除非一些约定俗称的协议格式，比如http协议，大多数服务器的业务的协议都是不同的，也就是说一个数据包里面的数据格式的解读应该是业务层的事情，和网络通信层应该解耦，为了网络层更加通用，我们无法知道上层协议长成什么样子，因为不同的协议格式是不一样的，它们与具体的业务有关。理由二：即使知道协议格式，我们在网络层进行解包处理对应的业务，如果这个业务处理比较耗时，比如需要进行复杂的运算，或者连接数据库进行账号密码验证，那么我们的网络线程会需要大量时间来处理这些任务，这样其它网络事件可能没法及时处理。鉴于以上二点，我们确实需要一个接收缓冲区，将收取到的数据放到该缓冲区里面去，并由专门的业务线程或者业务逻辑去从接收缓冲区中取出数据，并解包处理业务。

说了这么多，那发送缓冲区和接收缓冲区该设计成多大的容量？这是一个老生常谈的问题了，因为我们经常遇到这样的问题：预分配的内存太小不够用，太大的话可能会造成浪费。怎么办呢？答案就是像string、vector一样，设计出一个可以动态增长的缓冲区，按需分配，不够还可以扩展。

需要特别注意的是，这里说的发送缓冲区和接收缓冲区是每一个socket连接都存在一个。这是我们最常见的设计方案。
            
（六）协议的设计
除了一些通用的协议，如http、ftp协议以外，大多数服务器协议都是根据业务制定的。协议设计好了，数据包的格式就根据协议来设置。我们知道tcp/ip协议是流式数据，所以流式数据就是像流水一样，数据包与数据包之间没有明显的界限。比如A端给B端连续发了三个数据包，每个数据包都是50个字节，B端可能先收到10个字节，再收到140个字节；或者先收到20个字节，再收到20个字节，再收到110个字节；也可能一次性收到150个字节。这150个字节可以以任何字节数目组合和次数被B收到。所以我们讨论协议的设计第一个问题就是如何界定包的界限，也就是接收端如何知道每个包数据的大小。目前常用有如下三种方法：
１. 固定大小，这种方法就是假定每一个包的大小都是固定字节数目，例如上文中讨论的每个包大小都是50个字节，接收端每收气50个字节就当成一个包。
２. 指定包结束符，例如以一个\r\n(换行符和回车符)结束，这样对端只要收到这样的结束符，就可以认为收到了一个包，接下来的数据是下一个包的内容。
３. 指定包的大小，这种方法结合了上述两种方法，一般包头是固定大小，包头中有一个字段指定包体或者整个大的大小，对端收到数据以后先解析包头中的字段得到包体或者整个包的大小，然后根据这个大小去界定数据的界线。

协议要讨论的第二个问题是，设计协议的时候要尽量方便解包，也就是说协议的格式字段应该尽量清晰明了。

协议要讨论的第三个问题是，根据协议组装的单个数据包应该尽量小，注意这里指的是单个数据包，这样有如下好处：第一、对于一些移动端设备来说，其数据处理能力和带宽能力有限，小的数据不仅能加快处理速度，同时节省大量流量费用；第二、如果单个数据包足够小的话，对频繁进行网络通信的服务器端来说，可以大大减小其带宽压力，其所在的系统也能使用更少的内存。试想：假如一个股票服务器，如果一只股票的数据包是100个字节或者1000个字节，那同样是10000只股票区别呢？

协议要讨论的第四个问题是，对于数值类型，我们应该显式地指定数值的长度，比如long型，在32位机器上是32位4个字节，但是如果在64位机器上，就变成了64位8个字节了。这样同样是一个long型，发送方和接收方可能因为机器位数的不同会用不同的长度去解码。所以建议最好，在涉及到跨平台使用的协议最好显式地指定协议中整型字段的长度，比如int32、int64等等。下面是一个协议的接口的例子，当然java程序员应该很熟悉这样的接口：

    class BinaryReadStream
    {
    private:
        const char* const ptr;
        const size_t      len;
        const char*       cur;
        BinaryReadStream(const BinaryReadStream&);
        BinaryReadStream& operator=(const BinaryReadStream&);
    public:
        BinaryReadStream(const char* ptr, size_t len);
        virtual const char* GetData() const;
        virtual size_t GetSize() const;
        bool IsEmpty() const;
        bool ReadString(string* str, size_t maxlen, size_t& outlen);
        bool ReadCString(char* str, size_t strlen, size_t& len);
        bool ReadCCString(const char** str, size_t maxlen, size_t& outlen);
        bool ReadInt32(int32_t& i);
        bool ReadInt64(int64_t& i);
        bool ReadShort(short& i);
        bool ReadChar(char& c);
        size_t ReadAll(char* szBuffer, size_t iLen) const;
        bool IsEnd() const;
        const char* GetCurrent() const{ return cur; }
    
    public:
        bool ReadLength(size_t & len);
        bool ReadLengthWithoutOffset(size_t &headlen, size_t & outlen);
    };
    
    class BinaryWriteStream
    {
    public:
        BinaryWriteStream(string* data);
        virtual const char* GetData() const;
        virtual size_t GetSize() const;
        bool WriteCString(const char* str, size_t len);
        bool WriteString(const string& str);
        bool WriteDouble(double value, bool isNULL = false);
        bool WriteInt64(int64_t value, bool isNULL = false);
        bool WriteInt32(int32_t i, bool isNULL = false);
        bool WriteShort(short i, bool isNULL = false);
        bool WriteChar(char c, bool isNULL = false);
        size_t GetCurrentPos() const{ return m_data->length(); }
        void Flush();
        void Clear();
    private:
        string* m_data;
    };

其中BinaryWriteStream是编码协议的类，BinaryReadStream是解码协议的类。可以按下面这种方式来编码和解码。
编码：

```
std::string outbuf;
BinaryWriteStream writeStream(&outbuf);
writeStream.WriteInt32(msg_type_register);
writeStream.WriteInt32(m_seq);
writeStream.WriteString(retData);
writeStream.Flush();
```

解码：

```
BinaryReadStream readStream(strMsg.c_str(), strMsg.length());
int32_t cmd;
if (!readStream.ReadInt32(cmd))
{
	return false;
}

//int seq;
if (!readStream.ReadInt32(m_seq))
{
	return false;
}

std::string data;
size_t datalength;
if (!readStream.ReadString(&data, 0, datalength))
{
	return false;
}
```




二、服务器程序结构的组织

上面的六个标题，我们讨论了很多具体的细节问题，现在是时候讨论将这些细节组织起来了。根据我的个人经验，目前主流的思想是one thread one loop+reactor模式（也有proactor模式）的策略。通俗点说就是一个线程一个循环，即在一个线程的函数里面不断地循环依次做一些事情，这些事情包括检测网络事件、解包数据产生业务逻辑。我们先从最简单地来说，设定一些线程在一个循环里面做网络通信相关的事情，伪码如下：

```
while(退出标志)  
{  
	//IO复用技术检测socket可读事件、出错事件  
	//（如果有数据要发送，则也检测可写事件）  

	//如果有可读事件，对于侦听socket则接收新连接；  
    //对于普通socket则收取该socket上的数据，收取的数据存入对应的接收缓冲区，如果出错则关闭连接；  

    //如果有数据要发送，有可写事件，则发送数据  

    //如果有出错事件，关闭该连接   
}  
```

另外设定一些线程去处理接收到的数据，并解包处理业务逻辑，这些线程可以认为是业务线程了，伪码如下：

    //从接收缓冲区中取出数据解包，分解成不同的业务来处理  

上面的结构是目前最通用的服务器逻辑结构，但是能不能再简化一下或者说再综合一下呢？我们试试，你想过这样的问题没有：假如现在的机器有两个cpu（准确的来说应该是两个核），我们的网络线程数量是2个，业务逻辑线程也是2个，这样可能存在的情况就是：业务线程运行的时候，网络线程并没有运行，它们必须等待，如果是这样的话，干嘛要多建两个线程呢？除了程序结构上可能稍微清楚一点，对程序性能没有任何实质性提高，而且白白浪费cpu时间片在线程上下文切换上。所以，我们可以将网络线程与业务逻辑线程合并，合并后的伪码看起来是这样子的：

    while(退出标志)  
    {  
    	//IO复用技术检测socket可读事件、出错事件  
    	//（如果有数据要发送，则也检测可写事件）  
          
    　　//如果有可读事件，对于侦听socket则接收新连接；  
    　　//对于普通socket则收取该socket上的数据，收取的数据存入对应的接收缓冲区，如果出错则关闭连接；  
      
    　　//如果有数据要发送，有可写事件，则发送数据  
      
    　　//如果有出错事件，关闭该连接  
      
    　　//从接收缓冲区中取出数据解包，分解成不同的业务来处理  
    } 
你没看错，其实就是简单的合并，合并之后和不仅可以达到原来合并前的效果，而且在没有网络IO事件的时候，可以及时处理我们想处理的一些业务逻辑，并且减少了不必要的线程上下文切换时间。

我们再更进一步，甚至我们可以在这个while循环增加其它的一些任务的处理，比如程序的逻辑任务队列、定时器事件等等，伪码如下：

    while(退出标志)  
    {  
        //定时器事件处理  
    
        //IO复用技术检测socket可读事件、出错事件  
        //（如果有数据要发送，则也检测可写事件）  
    
        //如果有可读事件，对于侦听socket则接收新连接；  
        //对于普通socket则收取该socket上的数据，收取的数据存入对应的接收缓冲区，如果出错则关闭连接；  
    
        //如果有数据要发送，有可写事件，则发送数据  
    
        //如果有出错事件，关闭该连接  
    
        //从接收缓冲区中取出数据解包，分解成不同的业务来处理  
    
        //程序自定义任务1  
    
        //程序自定义任务2  
    } 
注意：之所以将定时器事件的处理放在网络IO事件的检测之前，是因为避免定时器事件过期时间太长。假如放在后面的话，可能前面的处理耗费了一点时间，等到处理定时器事件时，时间间隔已经过去了不少时间。虽然这样处理，也没法保证定时器事件百分百精确，但是能尽量保证。当然linux系统下提供eventfd这样的定时器对象，所有的定时器对象就能像处理socket这样的fd一样统一成处理。这也是网络库libevent的思想很像，libevent将socket、定时器、信号封装成统一的对象进行处理。

说了这么多理论性的东西，我们来一款流行的开源网络库muduo来说明吧（作者：陈硕），原库是基于boost的，我改成了C++11的版本，并修改了一些bug，在此感谢原作者陈硕。

上文介绍的核心线程函数的while循环位于eventloop.cpp中：

	void EventLoop::loop()
	{
	    assert(!looping_);
		assertInLoopThread();
		looping_ = true;
		quit_ = false;  
		LOG_TRACE << "EventLoop " << this << " start looping";
	    while (!quit_)
	    {
	        activeChannels_.clear();
	        pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
	        ++iteration_;
	        if (Logger::logLevel() <= Logger::TRACE)
	        {
	            printActiveChannels();
	        }
	        // TODO sort channel by priority
	        eventHandling_ = true;
	        for (ChannelList::iterator it = activeChannels_.begin();
	            it != activeChannels_.end(); ++it)
	        {
	            currentActiveChannel_ = *it;
	            currentActiveChannel_->handleEvent(pollReturnTime_);
	        }
	        currentActiveChannel_ = NULL;
	        eventHandling_ = false;
	        doPendingFunctors();
	
	        if (frameFunctor_)
	        {
	            frameFunctor_();
	        }		
	    }
	
	    LOG_TRACE << "EventLoop " << this << " stop looping";
	    looping_ = false;
	}
poller\_->poll利用epoll分离网络事件，然后接着处理分离出来的网络事件，每一个客户端socket对应一个连接，即一个TcpConnection和Channel通道对象。currentActiveChannel\_->handleEvent(pollReturnTime_)根据是可读、可写、出错事件来调用对应的处理函数，这些函数都是回调函数，程序初始化阶段设置进来的：

    void Channel::handleEvent(Timestamp receiveTime)
    {  
        std::shared_ptr<void> guard;  
        if (tied_)  
        {  
            guard = tie_.lock();  
            if (guard)  
            {  
                handleEventWithGuard(receiveTime);  
            }  
        }  
        else  
        {  
            handleEventWithGuard(receiveTime);  
        }  
    }  
    
    void Channel::handleEventWithGuard(Timestamp receiveTime)  
    {  
        eventHandling_ = true;  
        LOG_TRACE << reventsToString();  
        if ((revents_ & POLLHUP) && !(revents_ & POLLIN))  
        {  
            if (logHup_)  
            {  
                LOG_WARN << "Channel::handle_event() POLLHUP";  
            }  
            if (closeCallback_) closeCallback_();  
        }
        if (revents_ & POLLNVAL)  
        {  
            LOG_WARN << "Channel::handle_event() POLLNVAL";  
        }  
    
        if (revents_ & (POLLERR | POLLNVAL))  
        {  
            if (errorCallback_) errorCallback_();  
        }  
        if (revents_ & (POLLIN | POLLPRI | POLLRDHUP))  
        {  
            //当是侦听socket时，readCallback_指向Acceptor::handleRead  
            //当是客户端socket时，调用TcpConnection::handleRead   
            if (readCallback_) readCallback_(receiveTime);  
        }  
        if (revents_ & POLLOUT)  
        {  
            //如果是连接状态服的socket，则writeCallback_指向Connector::handleWrite()  
            if (writeCallback_) writeCallback_();  
        }  
        eventHandling_ = false; 
    } 
当然，这里利用了Channel对象的“多态性”，如果是普通socket，可读事件就会调用预先设置的回调函数；但是如果是侦听socket，则调用Aceptor对象的handleRead()
来接收新连接：

```
void Acceptor::handleRead()  
{  
    loop_->assertInLoopThread();  
    InetAddress peerAddr;  
    //FIXME loop until no more  
    int connfd = acceptSocket_.accept(&peerAddr);  
    if (connfd >= 0)  
    {  
        // string hostport = peerAddr.toIpPort();  
        // LOG_TRACE << "Accepts of " << hostport;  
        //newConnectionCallback_实际指向TcpServer::newConnection(int sockfd, const InetAddress& peerAddr)  
        if (newConnectionCallback_)  
        {  
            newConnectionCallback_(connfd, peerAddr);  
        }  
        else  
        {  
            sockets::close(connfd);  
        }  
    }  
    else  
    {  
        LOG_SYSERR << "in Acceptor::handleRead";  
        // Read the section named "The special problem of  
        // accept()ing when you can't" in libev's doc.  
        // By Marc Lehmann, author of livev.  
        if (errno == EMFILE)  
        {  
            ::close(idleFd_);  
            idleFd_ = ::accept(acceptSocket_.fd(), NULL, NULL);  
            ::close(idleFd_);  
            idleFd_ = ::open("/dev/null", O_RDONLY | O_CLOEXEC);  
        }  
    }  
}  
```

主循环里面的业务逻辑处理对应：

    doPendingFunctors();  
    if (frameFunctor_)  
    {  
       frameFunctor_();  
    }       

```
void EventLoop::doPendingFunctors()  
{  
    std::vector<Functor> functors;  
    callingPendingFunctors_ = true; 
    {  
        std::unique_lock<std::mutex> lock(mutex_);  
        functors.swap(pendingFunctors_);  
	}  
  
    for (size_t i = 0; i < functors.size(); ++i)  
    {  
        functors[i]();  
    }  
    callingPendingFunctors_ = false;
} 
```

 这里增加业务逻辑是增加执行任务的函数指针的，增加的任务保存在成员变量pendingFunctors\_中，这个变量是一个函数指针数组（vector对象），执行的时候，调用每个函数就可以了。上面的代码先利用一个栈变量将成员变量pendingFunctors\_里面的函数指针换过来，接下来对这个栈变量进行操作就可以了，这样减少了锁的粒度。因为成员变量pendingFunctors_在增加任务的时候，也会被用到，设计到多个线程操作，所以要加锁，增加任务的地方是：

    void EventLoop::queueInLoop(const Functor& cb)  
    {  
        {  
            std::unique_lock<std::mutex> lock(mutex_);  
            pendingFunctors_.push_back(cb);  
        }  
        if (!isInLoopThread() || callingPendingFunctors_)  
        {  
            wakeup();  
        }  
    }  
而frameFunctor_就更简单了，就是通过设置一个函数指针就可以了。当然这里有个技巧性的东西，即增加任务的时候，为了能够立即执行，使用唤醒机制，通过往一个fd里面写入简单的几个字节，来唤醒epoll，使其立刻返回，因为此时没有其它的socke有事件，这样接下来就执行刚才添加的任务了。

我们看一下数据收取的逻辑：

```
void TcpConnection::handleRead(Timestamp receiveTime)  
{  
    loop_->assertInLoopThread();  
    int savedErrno = 0;  
    ssize_t n = inputBuffer_.readFd(channel_->fd(), &savedErrno);  
    if (n > 0)  
    {    
        messageCallback_(shared_from_this(), &inputBuffer_, receiveTime);  
    }  
    else if (n == 0)  
    {  
        handleClose();  
    }  
    else  
    {  
        errno = savedErrno;  
        LOG_SYSERR << "TcpConnection::handleRead";  
        handleError();  
    }  
}  
```

将收到的数据放到接收缓冲区里面，将来我们来解包：

```
void ClientSession::OnRead(const std::shared_ptr<TcpConnection>& conn, Buffer* pBuffer, Timestamp receivTime)  
{  
    while (true)  
    {  
        //不够一个包头大小  
        if (pBuffer->readableBytes() < (size_t)sizeof(msg))  
        {  
            LOG_INFO << "buffer is not enough for a package header, pBuffer->readableBytes()=" << pBuffer->readableBytes() << ", sizeof(msg)=" << sizeof(msg);  
            return;  
        } 
        //不够一个整包大小  
        msg header;  
        memcpy(&header, pBuffer->peek(), sizeof(msg));  
        if (pBuffer->readableBytes() < (size_t)header.packagesize + sizeof(msg))  
            return;  

        pBuffer->retrieve(sizeof(msg));  
        std::string inbuf;  
        inbuf.append(pBuffer->peek(), header.packagesize);  
        pBuffer->retrieve(header.packagesize);  
        if (!Process(conn, inbuf.c_str(), inbuf.length()))  
        {  
            LOG_WARN << "Process error, close TcpConnection";  
            conn->forceClose();  
        }  
	}// end while-loop  
} 
```

先判断接收缓冲区里面的数据是否够一个包头大小，如果够再判断够不够包头指定的包体大小，如果还是够的话，接着在Process函数里面处理该包。

再看看发送数据的逻辑：

```
void TcpConnection::sendInLoop(const void* data, size_t len)  
{  
    loop_->assertInLoopThread();  
    ssize_t nwrote = 0;  
    size_t remaining = len;  
    bool faultError = false;  
    if (state_ == kDisconnected)  
    {  
        LOG_WARN << "disconnected, give up writing";  
        return;  
    }  
    // if no thing in output queue, try writing directly  
    if (!channel_->isWriting() && outputBuffer_.readableBytes() == 0)  
    {  
        nwrote = sockets::write(channel_->fd(), data, len);  
        if (nwrote >= 0)  
        {  
            remaining = len - nwrote;  
            if (remaining == 0 && writeCompleteCallback_)  
            {  
                loop_->queueInLoop(std::bind(writeCompleteCallback_, shared_from_this()));  
            }  
        }  
        else // nwrote < 0  
        {  
            nwrote = 0;  
            if (errno != EWOULDBLOCK)  
            {  
                LOG_SYSERR << "TcpConnection::sendInLoop";  
                if (errno == EPIPE || errno == ECONNRESET) // FIXME: any others?  
                {  
                    faultError = true;  
                }  
            }  
        }  
    }  
    assert(remaining <= len);  
    if (!faultError && remaining > 0)  
    {  
        size_t oldLen = outputBuffer_.readableBytes();  
        if (oldLen + remaining >= highWaterMark_                          
            && oldLen < highWaterMark_  
            && highWaterMarkCallback_)  
        {  
            loop_->queueInLoop(std::bind(highWaterMarkCallback_, shared_from_this(), oldLen + remaining));  
        }  
        outputBuffer_.append(static_cast<const char*>(data)+nwrote, remaining);  
        if (!channel_->isWriting())  
        {  
            channel_->enableWriting();  
        }  
    } 
} 
```

如果剩余的数据remaining大于则调用channel\_->enableWriting();开始监听可写事件，可写事件处理如下：

```
void TcpConnection::handleWrite()  
{  
    loop_->assertInLoopThread();  
    if (channel_->isWriting())  
    {  
        ssize_t n = sockets::write(channel_->fd(),  
            outputBuffer_.peek(),  
            outputBuffer_.readableBytes());  
        if (n > 0)  
        {  
            outputBuffer_.retrieve(n);  
            if (outputBuffer_.readableBytes() == 0)  
            {  
                channel_->disableWriting();  
                if (writeCompleteCallback_)  
                {  
                    loop_->queueInLoop(std::bind(writeCompleteCallback_, shared_from_this()));  
                }  
                if (state_ == kDisconnecting)  
                {  
                    shutdownInLoop();  
                }  
            }  
        }  
        else  
        {  
            LOG_SYSERR << "TcpConnection::handleWrite";  
            // if (state_ == kDisconnecting)  
            // {  
            //   shutdownInLoop();  
            // }  
        }  
    }  
    else  
    {  
        LOG_TRACE << "Connection fd = " << channel_->fd()  
            << " is down, no more writing";  
    }  
}  
```

如果发送完数据以后调用channel_->disableWriting();移除监听可写事件。

很多读者可能一直想问，文中不是说解包数据并处理逻辑是业务代码而非网络通信的代码，你这里貌似都混在一起了，其实没有，这里实际的业务代码处理都是框架曾提供的回调函数里面处理的，具体怎么处理，由框架使用者——业务层自己定义。

总结起来，实际上就是一个线程函数里一个loop那么点事情，不信你再看我曾经工作上的一个交易系统服务器项目代码：

```
void CEventDispatcher::Run()  
{  
    m_bShouldRun = true;  
    while(m_bShouldRun)  
    {  
        DispatchIOs();        
        SyncTime();  
        CheckTimer();  
        DispatchEvents();  
    }  
}  

void CEpollReactor::DispatchIOs()  
{  
    DWORD dwSelectTimeOut = SR_DEFAULT_EPOLL_TIMEOUT;  
    if (HandleOtherTask())  
    {  
        dwSelectTimeOut = 0;  
    }  
    struct epoll_event ev;  
    CEventHandlerIdMap::iterator itor = m_mapEventHandlerId.begin();  
    for(; itor!=m_mapEventHandlerId.end(); itor++)  
    {  
        CEventHandler *pEventHandler = (CEventHandler *)(*itor).first;  
        if(pEventHandler == NULL){  
            continue;  
        }  
        ev.data.ptr = pEventHandler;  
        ev.events = 0;  
        int nReadID, nWriteID;  
        pEventHandler->GetIds(&nReadID, &nWriteID);    
        if (nReadID > 0)  
        {  
            ev.events |= EPOLLIN;  
        }  
        if (nWriteID > 0)  
        {  
            ev.events |= EPOLLOUT;  
        }  

        epoll_ctl(m_fdEpoll, EPOLL_CTL_MOD, (*itor).second, &ev);  
    }  

    struct epoll_event events[EPOLL_MAX_EVENTS];  

    int nfds = epoll_wait(m_fdEpoll, events, EPOLL_MAX_EVENTS, dwSelectTimeOut/1000);  

    for (int i=0; i<nfds; i++)  
    {  
        struct epoll_event &evref = events[i];  
        CEventHandler *pEventHandler = (CEventHandler *)evref.data.ptr;  
        if ((evref.events|EPOLLIN)!=0 && m_mapEventHandlerId.find(pEventHandler)!=m_mapEventHandlerId.end())  
        {  
            pEventHandler->HandleInput();  
        }  
        if ((evref.events|EPOLLOUT)!=0 && m_mapEventHandlerId.find(pEventHandler)!=m_mapEventHandlerId.end())  
        {  
            pEventHandler->HandleOutput();  
        }  
    } 
} 

void CEventDispatcher::DispatchEvents()  
{  
    CEvent event;  
    CSyncEvent *pSyncEvent;  
    while(m_queueEvent.PeekEvent(event))  
    {  
        int nRetval; 
        if(event.pEventHandler != NULL)  
        {  
            nRetval = event.pEventHandler->HandleEvent(event.nEventID, event.dwParam, event.pParam);  
        }  
        else  
        {  
            nRetval = HandleEvent(event.nEventID, event.dwParam, event.pParam);  
        }  

        if(event.pAdd != NULL)      //同步消息  
        {  
            pSyncEvent=(CSyncEvent *)event.pAdd;  
            pSyncEvent->nRetval = nRetval;  
            pSyncEvent->sem.UnLock();  
        }  
	}
}
```

再看看蘑菇街开源的TeamTalk的源码（代码下载地址：https://github.com/baloonwj/TeamTalk）：

    void CEventDispatch::StartDispatch(uint32_t wait_timeout)  
    {  
        fd_set read_set, write_set, excep_set;  
        timeval timeout;  
        timeout.tv_sec = 0;  
        timeout.tv_usec = wait_timeout * 1000;  // 10 millisecond 
        if(running)  
        	return;  
    	running = true;  
      
    	while (running) 
        {  
    		_CheckTimer();  
    		_CheckLoop(); 
    		if (!m_read_set.fd_count && !m_write_set.fd_count && !m_excep_set.fd_count)  
        	{  
            	Sleep(MIN_TIMER_DURATION);  
            	continue;  
        	}  
      
            m_lock.lock();  
            memcpy(&read_set, &m_read_set, sizeof(fd_set));  
            memcpy(&write_set, &m_write_set, sizeof(fd_set));  
            memcpy(&excep_set, &m_excep_set, sizeof(fd_set));  
            m_lock.unlock();  
    
            int nfds = select(0, &read_set, &write_set, &excep_set, &timeout);  
    
            if (nfds == SOCKET_ERROR)  
            {  
                log("select failed, error code: %d", GetLastError());  
                Sleep(MIN_TIMER_DURATION);  
                continue;           // select again  
            }  
    
            if (nfds == 0)  
            {  
                continue;  
            }  
    
            for (u_int i = 0; i < read_set.fd_count; i++)  
            {  
                //log("select return read count=%d\n", read_set.fd_count);  
                SOCKET fd = read_set.fd_array[i];  
                CBaseSocket* pSocket = FindBaseSocket((net_handle_t)fd);  
                if (pSocket)  
                {  
                    pSocket->OnRead();  
                    pSocket->ReleaseRef();  
                }  
            }  
    
            for (u_int i = 0; i < write_set.fd_count; i++)  
            {  
                //log("select return write count=%d\n", write_set.fd_count);  
                SOCKET fd = write_set.fd_array[i];  
                CBaseSocket* pSocket = FindBaseSocket((net_handle_t)fd);  
                if (pSocket)  
                {  
                    pSocket->OnWrite();  
                    pSocket->ReleaseRef();  
                }  
            }  
    
            for (u_int i = 0; i < excep_set.fd_count; i++)  
            {  
                //log("select return exception count=%d\n", excep_set.fd_count);  
                SOCKET fd = excep_set.fd_array[i];  
                CBaseSocket* pSocket = FindBaseSocket((net_handle_t)fd);  
                if (pSocket)  
                {  
                    pSocket->OnClose();  
                    pSocket->ReleaseRef();  
                }  
            }   
    	} 
    }  

再看filezilla，一款ftp工具的服务器端，它采用的是Windows的WSAAsyncSelect模型（代码下载地址：https://github.com/baloonwj/filezilla）：

    //Processes event notifications sent by the sockets or the layers
    static LRESULT CALLBACK WindowProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)
    {
    	if (message>=WM_SOCKETEX_NOTIFY)
    	{
    		//Verify parameters
    		ASSERT(hWnd);
    		CAsyncSocketExHelperWindow *pWnd=(CAsyncSocketExHelperWindow *)GetWindowLongPtr(hWnd, GWLP_USERDATA);
    		ASSERT(pWnd);
    		if (!pWnd)
    			return 0;
    
    		if (message < static_cast<UINT>(WM_SOCKETEX_NOTIFY+pWnd->m_nWindowDataSize)) //Index is within socket storage
    		{
    			//Lookup socket and verify if it's valid
    			CAsyncSocketEx *pSocket=pWnd->m_pAsyncSocketExWindowData[message - WM_SOCKETEX_NOTIFY].m_pSocket;
    			SOCKET hSocket = wParam;
    			if (!pSocket)
    				return 0;
    			if (hSocket == INVALID_SOCKET)
    				return 0;
    			if (pSocket->m_SocketData.hSocket != hSocket)
    				return 0;
    
    			int nEvent = lParam & 0xFFFF;
    			int nErrorCode = lParam >> 16;
    
    			//Dispatch notification
    			if (!pSocket->m_pFirstLayer) {
    				//Dispatch to CAsyncSocketEx instance
    				switch (nEvent)
    				{
    				case FD_READ:
    				 #ifndef NOSOCKETSTATES
    						if (pSocket->GetState() == connecting && !nErrorCode)
    						{
    							pSocket->m_nPendingEvents |= FD_READ;
    							break;
    						}
    						else if (pSocket->GetState() == attached)
    							pSocket->SetState(connected);
    						if (pSocket->GetState() != connected)
    							break;
    							// Ignore further FD_READ events after FD_CLOSE has been received
                            if (pSocket->m_SocketData.onCloseCalled)
                                break;
                                 #endif //NOSOCKETSTATES
    
     #ifndef NOSOCKETSTATES
    						if (nErrorCode)
    							pSocket->SetState(aborted);
     #endif //NOSOCKETSTATES
    						if (pSocket->m_lEvent & FD_READ) {
    							pSocket->OnReceive(nErrorCode);
    						}
    						break;
    					case FD_FORCEREAD: //Forceread does not check if there's data waiting
     #ifndef NOSOCKETSTATES
    						if (pSocket->GetState() == connecting && !nErrorCode)
    						{
    							pSocket->m_nPendingEvents |= FD_FORCEREAD;
    							break;
    						}
    						else if (pSocket->GetState() == attached)
    							pSocket->SetState(connected);
    						if (pSocket->GetState() != connected)
    							break;
     #endif //NOSOCKETSTATES
    						if (pSocket->m_lEvent & FD_READ)
    						{
     #ifndef NOSOCKETSTATES
    							if (nErrorCode)
    								pSocket->SetState(aborted);
     #endif //NOSOCKETSTATES
    							pSocket->OnReceive(nErrorCode);
    						}
    						break;
    					case FD_WRITE:
     #ifndef NOSOCKETSTATES
    						if (pSocket->GetState() == connecting && !nErrorCode)
    						{
    							pSocket->m_nPendingEvents |= FD_WRITE;
    							break;
    						}
    						else if (pSocket->GetState() == attached && !nErrorCode)
    							pSocket->SetState(connected);
    						if (pSocket->GetState() != connected)
    							break;
     #endif //NOSOCKETSTATES
    						if (pSocket->m_lEvent & FD_WRITE)
    						{
     #ifndef NOSOCKETSTATES
    							if (nErrorCode)
    								pSocket->SetState(aborted);
     #endif //NOSOCKETSTATES
    							pSocket->OnSend(nErrorCode);
    						}
    						break;
    					case FD_CONNECT:
     #ifndef NOSOCKETSTATES
    						if (pSocket->GetState() == connecting)
    						{
    							if (nErrorCode && pSocket->m_SocketData.nextAddr)
    							{
    								if (pSocket->TryNextProtocol())
    									break;
    							}
    							pSocket->SetState(connected);
    						}
    						else if (pSocket->GetState() == attached && !nErrorCode)
    							pSocket->SetState(connected);
     #endif //NOSOCKETSTATES
    						if (pSocket->m_lEvent & FD_CONNECT)
    							pSocket->OnConnect(nErrorCode);
     #ifndef NOSOCKETSTATES
    						if (!nErrorCode)
    						{
    							if ((pSocket->m_nPendingEvents&FD_READ) && pSocket->GetState() == connected)
    								pSocket->OnReceive(0);
    							if ((pSocket->m_nPendingEvents&FD_FORCEREAD) && pSocket->GetState() == connected)
    								pSocket->OnReceive(0);
    							if ((pSocket->m_nPendingEvents&FD_WRITE) && pSocket->GetState() == connected)
    								pSocket->OnSend(0);
    						}
    						pSocket->m_nPendingEvents = 0;
     #endif
    						break;
    					case FD_ACCEPT:
     #ifndef NOSOCKETSTATES
    						if (pSocket->GetState() != listening && pSocket->GetState() != attached)
    							break;
     #endif //NOSOCKETSTATES
    						if (pSocket->m_lEvent & FD_ACCEPT)
    							pSocket->OnAccept(nErrorCode);
    						break;
    					case FD_CLOSE:
     #ifndef NOSOCKETSTATES
    						if (pSocket->GetState() != connected && pSocket->GetState() != attached)
    							break;
    							// If there are still bytes left to read, call OnReceive instead of
    					// OnClose and trigger a new OnClose
    					DWORD nBytes = 0;
    					if (!nErrorCode && pSocket->IOCtl(FIONREAD, &nBytes))
    					{
    						if (nBytes > 0)
    						{
    							// Just repeat message.
    							pSocket->ResendCloseNotify();
    							pSocket->m_SocketData.onCloseCalled = true;
    							pSocket->OnReceive(WSAESHUTDOWN);
    							break;
    						}
    					}
    
    					pSocket->SetState(nErrorCode ? aborted : closed);
    					 #endif //NOSOCKETSTATES
    					 pSocket->OnClose(nErrorCode);
    					break;
    				}
    			}
    			else //Dispatch notification to the lowest layer
    			{
    				if (nEvent == FD_READ)
    				{
    					// Ignore further FD_READ events after FD_CLOSE has been received
    					if (pSocket->m_SocketData.onCloseCalled)
    						return 0;
    
    					DWORD nBytes;
    					if (!pSocket->IOCtl(FIONREAD, &nBytes))
    						nErrorCode = WSAGetLastError();
    					if (pSocket->m_pLastLayer)
    						pSocket->m_pLastLayer->CallEvent(nEvent, nErrorCode);
    				}
    				else if (nEvent == FD_CLOSE)
    				{
    					// If there are still bytes left to read, call OnReceive instead of
    					// OnClose and trigger a new OnClose
    					DWORD nBytes = 0;
    					if (!nErrorCode && pSocket->IOCtl(FIONREAD, &nBytes))
    					{
    						if (nBytes > 0)
    						{
    							// Just repeat message.
    							pSocket->ResendCloseNotify();
    							if (pSocket->m_pLastLayer)
    								pSocket->m_pLastLayer->CallEvent(FD_READ, 0);
    							return 0;
    						}
    					}
    					pSocket->m_SocketData.onCloseCalled = true;
    					if (pSocket->m_pLastLayer)
    						pSocket->m_pLastLayer->CallEvent(nEvent, nErrorCode);
    				}
    				else if (pSocket->m_pLastLayer)
    					pSocket->m_pLastLayer->CallEvent(nEvent, nErrorCode);
    			}
    		}
    		return 0;
    	}
    	else if (message == WM_USER) //Notification event sent by a layer
    	{
    		//Verify parameters, lookup socket and notification message
    		//Verify parameters
    		ASSERT(hWnd);
    		CAsyncSocketExHelperWindow *pWnd=(CAsyncSocketExHelperWindow *)GetWindowLongPtr(hWnd, GWLP_USERDATA);
    		ASSERT(pWnd);
    		if (!pWnd)
    			return 0;
    
    		if (wParam >= static_cast<UINT>(pWnd->m_nWindowDataSize)) //Index is within socket storage
    		{
    			return 0;
    		}
    
    		CAsyncSocketEx *pSocket = pWnd->m_pAsyncSocketExWindowData[wParam].m_pSocket;
    		CAsyncSocketExLayer::t_LayerNotifyMsg *pMsg = (CAsyncSocketExLayer::t_LayerNotifyMsg *)lParam;
    		if (!pMsg || !pSocket || pSocket->m_SocketData.hSocket != pMsg->hSocket)
    		{
    			delete pMsg;
    			return 0;
    		}
    		int nEvent=pMsg->lEvent&0xFFFF;
    		int nErrorCode=pMsg->lEvent>>16;
    
    		//Dispatch to layer
    		if (pMsg->pLayer)
    			pMsg->pLayer->CallEvent(nEvent, nErrorCode);
    		else
    		{
    			//Dispatch to CAsyncSocketEx instance
    			switch (nEvent)
    			{
    			case FD_READ:
    			#ifndef NOSOCKETSTATES
    					if (pSocket->GetState() == connecting && !nErrorCode)
    					{
    						pSocket->m_nPendingEvents |= FD_READ;
    						break;
    					}
    					else if (pSocket->GetState() == attached && !nErrorCode)
    						pSocket->SetState(connected);
    					if (pSocket->GetState() != connected)
    						break;
    #endif //NOSOCKETSTATES
    					if (pSocket->m_lEvent & FD_READ)
    					{
    #ifndef NOSOCKETSTATES
    						if (nErrorCode)
    							pSocket->SetState(aborted);
    #endif //NOSOCKETSTATES
    						pSocket->OnReceive(nErrorCode);
    					}
    					break;
    				case FD_FORCEREAD: //Forceread does not check if there's data waiting
    #ifndef NOSOCKETSTATES
    					if (pSocket->GetState() == connecting && !nErrorCode)
    					{
    						pSocket->m_nPendingEvents |= FD_FORCEREAD;
    						break;
    					}
    					else if (pSocket->GetState() == attached && !nErrorCode)
    						pSocket->SetState(connected);
    					if (pSocket->GetState() != connected)
    						break;
    #endif //NOSOCKETSTATES
    					if (pSocket->m_lEvent & FD_READ)
    					{
    #ifndef NOSOCKETSTATES
    						if (nErrorCode)
    							pSocket->SetState(aborted);
    #endif //NOSOCKETSTATES
    						pSocket->OnReceive(nErrorCode);
    					}
    					break;
    				case FD_WRITE:
    #ifndef NOSOCKETSTATES
    					if (pSocket->GetState() == connecting && !nErrorCode)
    					{
    						pSocket->m_nPendingEvents |= FD_WRITE;
    						break;
    					}
    					else if (pSocket->GetState() == attached && !nErrorCode)
    						pSocket->SetState(connected);
    					if (pSocket->GetState() != connected)
    						break;
    #endif //NOSOCKETSTATES
    					if (pSocket->m_lEvent & FD_WRITE)
    					{
    #ifndef NOSOCKETSTATES
    						if (nErrorCode)
    							pSocket->SetState(aborted);
    #endif //NOSOCKETSTATES
    						pSocket->OnSend(nErrorCode);
    					}
    					break;
    				case FD_CONNECT:
    #ifndef NOSOCKETSTATES
    					if (pSocket->GetState() == connecting)
    						pSocket->SetState(connected);
    					else if (pSocket->GetState() == attached && !nErrorCode)
    						pSocket->SetState(connected);
    #endif //NOSOCKETSTATES
    					if (pSocket->m_lEvent & FD_CONNECT)
    						pSocket->OnConnect(nErrorCode);
    #ifndef NOSOCKETSTATES
    					if (!nErrorCode)
    					{
    						if (((pSocket->m_nPendingEvents&FD_READ) && pSocket->GetState() == connected) && (pSocket->m_lEvent & FD_READ))
    							pSocket->OnReceive(0);
    						if (((pSocket->m_nPendingEvents&FD_FORCEREAD) && pSocket->GetState() == connected) && (pSocket->m_lEvent & FD_READ))
    							pSocket->OnReceive(0);
    						if (((pSocket->m_nPendingEvents&FD_WRITE) && pSocket->GetState() == connected) && (pSocket->m_lEvent & FD_WRITE))
    							pSocket->OnSend(0);
    					}
    					pSocket->m_nPendingEvents = 0;
    #endif //NOSOCKETSTATES
    					break;
    				case FD_ACCEPT:
    #ifndef NOSOCKETSTATES
    					if ((pSocket->GetState() == listening || pSocket->GetState() == attached) && (pSocket->m_lEvent & FD_ACCEPT))
    #endif //NOSOCKETSTATES
    					{
    						pSocket->OnAccept(nErrorCode);
    					}
    					break;
    				case FD_CLOSE:
    #ifndef NOSOCKETSTATES
    					if ((pSocket->GetState() == connected || pSocket->GetState() == attached) && (pSocket->m_lEvent & FD_CLOSE))
    					{
    						pSocket->SetState(nErrorCode?aborted:closed);
    #else
    					{
    #endif //NOSOCKETSTATES
    						pSocket->OnClose(nErrorCode);
    					}
    					break;
    				}
    			}
    			delete pMsg;
    			return 0;
    		}
    		else if (message == WM_USER+1)
    		{
    			// WSAAsyncGetHostByName reply
    			// Verify parameters
    		ASSERT(hWnd);
    		CAsyncSocketExHelperWindow *pWnd = (CAsyncSocketExHelperWindow *)GetWindowLongPtr(hWnd, GWLP_USERDATA);
    		ASSERT(pWnd);
    		if (!pWnd)
    			return 0;
    
    		CAsyncSocketEx *pSocket = NULL;
    		for (int i = 0; i < pWnd->m_nWindowDataSize; ++i) {
    			pSocket = pWnd->m_pAsyncSocketExWindowData[i].m_pSocket;
    			if (pSocket && pSocket->m_hAsyncGetHostByNameHandle &&
    				pSocket->m_hAsyncGetHostByNameHandle == (HANDLE)wParam &&
    				pSocket->m_pAsyncGetHostByNameBuffer)
    				break;
    		}
    		if (!pSocket || !pSocket->m_pAsyncGetHostByNameBuffer)
    			return 0;
    
    		int nErrorCode = lParam >> 16;
    		if (nErrorCode) {
    			pSocket->OnConnect(nErrorCode);
    			return 0;
    		}
    
    		SOCKADDR_IN sockAddr{};
    		sockAddr.sin_family = AF_INET;
    		sockAddr.sin_addr.s_addr = ((LPIN_ADDR)((LPHOSTENT)pSocket->m_pAsyncGetHostByNameBuffer)->h_addr)->s_addr;
    
    		sockAddr.sin_port = htons(pSocket->m_nAsyncGetHostByNamePort);
    
    		BOOL res = pSocket->Connect((SOCKADDR*)&sockAddr, sizeof(sockAddr));
    		delete [] pSocket->m_pAsyncGetHostByNameBuffer;
    		pSocket->m_pAsyncGetHostByNameBuffer = 0;
    		pSocket->m_hAsyncGetHostByNameHandle = 0;
    
    		if (!res)
    			if (GetLastError() != WSAEWOULDBLOCK)
    				pSocket->OnConnect(GetLastError());
    		return 0;
    	}
    	else if (message == WM_USER + 2)
    	{
    		//Verify parameters, lookup socket and notification message
    		//Verify parameters
    		if (!hWnd)
    			return 0;
    
    		CAsyncSocketExHelperWindow *pWnd=(CAsyncSocketExHelperWindow *)GetWindowLongPtr(hWnd, GWLP_USERDATA);
    		if (!pWnd)
    			return 0;
    
    		if (wParam >= static_cast<UINT>(pWnd->m_nWindowDataSize)) //Index is within socket storage
    			return 0;
    
    		CAsyncSocketEx *pSocket = pWnd->m_pAsyncSocketExWindowData[wParam].m_pSocket;
    		if (!pSocket)
    			return 0;
    
    		// Process pending callbacks
    		std::list<t_callbackMsg> tmp;
    		tmp.swap(pSocket->m_pendingCallbacks);
    		pSocket->OnLayerCallback(tmp);
    
    		for (auto & cb : tmp) {
    			delete [] cb.str;
    		}
    	}
    	else if (message == WM_TIMER)
    	{
    		if (wParam != 1)
    			return 0;
    
    		ASSERT(hWnd);
    		CAsyncSocketExHelperWindow *pWnd=(CAsyncSocketExHelperWindow *)GetWindowLongPtr(hWnd, GWLP_USERDATA);
    		ASSERT(pWnd && pWnd->m_pThreadData);
    		if (!pWnd || !pWnd->m_pThreadData)
    			return 0;
    
    		if (pWnd->m_pThreadData->layerCloseNotify.empty())
    		{
    			KillTimer(hWnd, 1);
    			return 0;
    		}
    		CAsyncSocketEx* socket = pWnd->m_pThreadData->layerCloseNotify.front();
    		pWnd->m_pThreadData->layerCloseNotify.pop_front();
    		if (pWnd->m_pThreadData->layerCloseNotify.empty())
    			KillTimer(hWnd, 1);
    
    		if (socket)
    			PostMessage(hWnd, socket->m_SocketData.nSocketIndex + WM_SOCKETEX_NOTIFY, socket->m_SocketData.hSocket, FD_CLOSE);
    		return 0;
    	}
    	return DefWindowProc(hWnd, message, wParam, lParam);
    }	
上面截取的代码段，如果你对这些项目不是很熟悉的话，估计你也没有任何兴趣去细细看每一行代码逻辑。但是你一定要明白我所说的这个结构的逻辑，基本上目前主流的网络框架都是这套原理。比如filezilla的网络通信层同样也被用在大名鼎鼎的电驴（easyMule）中。

关于单个服务程序的框架，我已经介绍完了，如果你能完全理解我要表达的意思，我相信你也能构建出一套高性能服务程序来。

另外，服务器框架也可以在上面的设计思路的基础上增加很多有意思的细节，比如流量控制。举另外 一个我实际做过的项目中的例子吧：

一般实际项目中，当客户端连接数目比较多的时候，服务器在处理网络数据的时候，如果同时有多个socket上有数据要处理，由于cpu核数有限，根据上面先检测iO事件再处理IO事件可能会出现工作线程一直处理前几个socket的事件，直到前几个socket处理完毕后再处理后面几个socket的数据。这就相当于，你去饭店吃饭，大家都点了菜，但是有些桌子上一直在上菜，而有些桌子上一直没有菜。这样肯定不好，我们来看下如何避免这种现象：

	int CFtdEngine::HandlePackage(CFTDCPackage *pFTDCPackage, CFTDCSession *pSession)
	{
		//NET_IO_LOG0("CFtdEngine::HandlePackage\n");
		FTDC_PACKAGE_DEBUG(pFTDCPackage);
	    if (pFTDCPackage->GetTID() != FTD_TID_ReqUserLogin)
	    {
	        if (!IsSessionLogin(pSession->GetSessionID()))
	        {
	            SendErrorRsp(pFTDCPackage, pSession, 1, "客户未登录");
	            return 0;
	        }
	    }
	
	    CalcFlux(pSession, pFTDCPackage->Length());	//统计流量
	
	    REPORT_EVENT(LOG_DEBUG, "Front/Fgateway", "登录请求%0x", pFTDCPackage->GetTID()); 
	
	    int nRet = 0;
	    switch(pFTDCPackage->GetTID()) 
	    {
	
	    case FTD_TID_ReqUserLogin:
	        ///huwp：20070608：检查过高版本的API将被禁止登录
	        if (pFTDCPackage->GetVersion()>FTD_VERSION)
	        {
	            SendErrorRsp(pFTDCPackage, pSession, 1, "Too High FTD Version");
	            return 0;
	        }
	        nRet = OnReqUserLogin(pFTDCPackage, (CFTDCSession *)pSession);
	        FTDRequestIndex.incValue();
	        break;
	    case FTD_TID_ReqCheckUserLogin:
	        nRet = OnReqCheckUserLogin(pFTDCPackage, (CFTDCSession *)pSession);
	        FTDRequestIndex.incValue();
	        break;
	    case FTD_TID_ReqSubscribeTopic:
	        nRet = OnReqSubscribeTopic(pFTDCPackage, (CFTDCSession *)pSession);
	        FTDRequestIndex.incValue();
	        break;	
	    }
	
	    return 0;
	}
当有某个socket上有数据可读时，接着接收该socket上的数据，对接收到的数据进行解包，然后调用CalcFlux(pSession, pFTDCPackage->Length())进行流量统计：

```
void CFrontEngine::CalcFlux(CSession *pSession, const int nFlux)
{
	TFrontSessionInfo *pSessionInfo = m_mapSessionInfo.Find(pSession->GetSessionID());
	if (pSessionInfo != NULL)
	{
		//流量控制改为计数
		pSessionInfo->nCommFlux ++; 
		///若流量超过规定，则挂起该会话的读操作
		if (pSessionInfo->nCommFlux >= pSessionInfo->nMaxCommFlux)
		{
			pSession->SuspendRead(true);
		}
	}
}
```

该函数会先让某个连接会话（Session）处理的包数量递增，接着判断是否超过最大包数量，则设置读挂起标志：

```
void CSession::SuspendRead(bool bSuspend)  
{  
	m_bSuspendRead = bSuspend;  
}  
```

这样下次将会从检测的socket列表中排除该socket：

```
void CEpollReactor::RegisterIO(CEventHandler *pEventHandler)  
{  
    int nReadID, nWriteID;  
    pEventHandler->GetIds(&nReadID, &nWriteID);  
    if (nWriteID != 0 && nReadID ==0)  
    {  
        nReadID = nWriteID;  
    }  
    if (nReadID != 0)  
    {  
        m_mapEventHandlerId[pEventHandler] = nReadID;  
        struct epoll_event ev;  
        ev.data.ptr = pEventHandler;  
        if(epoll_ctl(m_fdEpoll, EPOLL_CTL_ADD, nReadID, &ev) != 0)  
        {  
            perror("epoll_ctl EPOLL_CTL_ADD");  
        }  
    }  
}  

void CSession::GetIds(int *pReadId, int *pWriteId)  
{  
    m_pChannelProtocol->GetIds(pReadId,pWriteId);  
    if (m_bSuspendRead)  
    {  
        *pReadId = 0;  
    }  
}  
```

也就是说不再检测该socket上是否有数据可读。然后在定时器里1秒后重置该标志，这样这个socket上有数据的话又可以重新检测到了：

```
const int SESSION_CHECK_TIMER_ID    = 9;  
const int SESSION_CHECK_INTERVAL    = 1000;  

SetTimer(SESSION_CHECK_TIMER_ID, SESSION_CHECK_INTERVAL);  

void CFrontEngine::OnTimer(int nIDEvent)  
{  
    if (nIDEvent == SESSION_CHECK_TIMER_ID)  
    {  
        CSessionMap::iterator itor = m_mapSession.Begin();  
        while (!itor.IsEnd())  
        {  
            TFrontSessionInfo *pFind = m_mapSessionInfo.Find((*itor)->GetSessionID());  
            if (pFind != NULL)  
            {  
                CheckSession(*itor, pFind);  
            }  
            itor++;  
        }  
    }  
}  

void CFrontEngine::CheckSession(CSession *pSession, TFrontSessionInfo *pSessionInfo)  
{  
    ///重新开始计算流量  
    pSessionInfo->nCommFlux -= pSessionInfo->nMaxCommFlux;  
    if (pSessionInfo->nCommFlux < 0)  
    {  
        pSessionInfo->nCommFlux = 0;  
    }  
    ///若流量超过规定，则挂起该会话的读操作  
    pSession->SuspendRead(pSessionInfo->nCommFlux >= pSessionInfo->nMaxCommFlux);  
}  
```

这就相当与饭店里面先给某一桌客人上一些菜，让他们先吃着，等上了一些菜之后不会再给这桌继续上菜了，而是给其它空桌上菜，大家都吃上后，继续回来给原先的桌子继续上菜。实际上我们的饭店都是这么做的。上面的例子是单服务流量控制的实现的一个非常好的思路，它保证了每个客户端都能均衡地得到服务，而不是一些客户端等很久才有响应。当然，这样的技术不能适用于有顺序要求的业务，例如销售系统，这些系统一般是先下单先得到的。

另外现在的服务器为了加快ＩＯ操作，大量使用缓存技术，缓存实际上是以空间换取时间的策略。对于一些反复使用的，但是不经常改变的信息，如果从原始地点加载这些信息就比较耗时的数据（比如从磁盘中、从数据库中），我们就可以使用缓存。所以时下像redis、leveldb、fastdb等各种内存数据库大行其道。如果你要从事服务器开发，你至少需要掌握它们中的几种。

　　
鉴于笔者能力和经验有限，文中难免有错漏之处，欢迎提意见。
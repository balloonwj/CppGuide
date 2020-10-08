# 07 服务器端msfs源码分析

这篇文章是对TeamTalk服务程序msfs的源码和架构设计分析。msfs作用是用来接受teamtalk聊天中产生的聊天图片的上传和下载。还是老规矩，把该服务在整个架构中的位置图贴一下吧。

![](../imgs/tt12.jpg)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)



可以看到，msfs仅被客户端连接，客户端以http的方式来上传和下载聊天图片。

可能很多同学对http协议不是很熟悉，或者说一知半解。这里大致介绍一下http协议，http协议其实也是一种应用层协议，建立在tcp/ip层之上，其由包头和包体两部分组成（不一定要有包体），看个例子：

比如当我们用浏览器请求一个网址http://www.hootina.org/index.php，实际是浏览器给特定的服务器发送如下数据包，包头部分如下：

```
GET /index.php HTTP/1.1\r\n
Host: www.hootina.org\r\n
Connection: keep-alive\r\n
Cache-Control: max-age=0\r\n
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,/;q=0.8\r\n
User-Agent: Mozilla/5.0\r\n
\r\n
```

这个包没有包体。

从上面我们可以看出一个http协议大致格式可以描述如下：

```plain
GET或Post请求方法  请求的资源路径 http协议版本号\r\n
字段名1：值1\r\n
字段名2：值2\r\n
字段名3：值3\r\n
字段名4：值4\r\n
字段名5：值5\r\n
字段名6：值6\r\n
\r\n
```

也就是是http协议的头部是一行一行的，每一行以\r\n表示该行结束，最后多出一个空行以\r\n结束表示头部的结束。接下来就是包体的大小了（如果有的话，上文的例子没有包体）。一般get方法会将参数放在请求的资源路径后面，像这样

http://wwww.hootina.org/index.php?变量1=值1&变量2=值2&变量3=值3&变量4=值4

网址后面的问号表示参数开始，每一个参数与参数之间用&隔开

还有一种post的请求方法，这种数据就是将数据放在包体里面了，例如：

```plain
POST /otn/login/loginAysnSuggest HTTP/1.1\r\n
Host: kyfw.12306.cn\r\n
Connection: keep-alive\r\n
Content-Length: 96\r\n
Accept: */*\r\n
Origin: https://kyfw.12306.cn\r\n
X-Requested-With: XMLHttpRequest\r\n
User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/55.0.2883.75\r\n 
Content-Type: application/x-www-form-urlencoded; charset=UTF-8\r\n
Referer: https://kyfw.12306.cn/otn/login/init\r\n
Accept-Encoding: gzip, deflate, br\r\n
Accept-Language: zh-CN,zh;q=0.8\r\n
\r\n
loginUserDTO.user_name=balloonwj%40qq.com&userDTO.password=xxxxgjqf&randCode=184%2C55%2C37%2C117
```

上述报文中`loginUserDTO.user_name=balloonwj%40qq.com&userDTO.password=2032_scsgjqf&randCode=184%2C55%2C37%2C117` 其实包体内容，这个包是我的一个12306买票软件发给12306服务器的报文。这里拿来做个例子。

因为对方收到http报文的时候，如果包体有内容，那么必须告诉对方包体有多大。这个最常用的就是通过包头的Content-Length字段来指定大小。上面的例子中Content-Length等于96，正好就是字符串 `loginUserDTO.user_name=balloonwj%40qq.com&userDTO.password=xxxxgjqf&randCode=184%2C55%2C37%2C117` 的长度，也就是包体的大小。

还有一种叫做http chunk的编码技术，通过对http包内容进行分块传输。这里就不介绍了（如果你感兴趣，可以私聊我）。

常见的对http协议有如下几个误解：

1. html文档的头就是http的头

 这种认识是错误的，html文档的头部也是http数据包的包体的一部分。正确的http头是长的像上文介绍的那种。

2. 关于http头Connection:keep-alive字段

  一端指定了这个字段后，发http包给另外一端。这个选项只是一种建议性的选项，对端不一定必须采纳，对方也可能在实际实现时，将http连接设置为短连接，即不采纳这个字段的建议。

3. 每个字段都是必须的吗？

不是，大多数字段都不是必须的。但是特定的情况下，某些字段是必须的。比如，通过post发送的数据，就必须设置Content-Length。不然，收包的一端如何知道包体多大。又比如如果你的数据采取了gzip压缩格式，你就必须指定Accept-Encoding: gzip，然对方如何解包你的数据。

好了，http协议就暂且介绍这么多，下面回到正题上来说msfs的源码。

msfs在main函数里面做了如下初始化工作，伪码如下：

```cpp
//1. 建立一个两个任务队列，分别处理http get请求和post请求

//2. 创建名称为000～255的文件夹，每个文件夹里面会有000～255个子目录，这些目录用于存放聊天图片

//3. 在8700端口上监听客户端连接

//4. 启动程序消息泵
```

第1点，建立任务队列我们前面系列的文章已经介绍过了。

第2点，代码如下：

```cpp
g_fileManager = FileManager::getInstance(listen_ip, base_dir, fileCnt, filesPerDir);
int ret = g_fileManager->initDir();
```

```cpp
int FileManager::initDir() {
		bool isExist = File::isExist(m_disk);
		if (!isExist) {
			u64 ret = File::mkdirNoRecursion(m_disk);
			if (ret) {
				log("The dir[%s] set error for code[%d], \
				    its parent dir may no exists", m_disk, ret);
				return -1;
			}
		}
		
		//255 X 255 
		char first[10] = {0};
		char second[10] = {0};
		for (int i = 0; i <= FIRST_DIR_MAX; i++) {
			snprintf(first, 5, "%03d", i);
			string tmp = string(m_disk) + "/" + string(first);
		    int code = File::mkdirNoRecursion(tmp.c_str());
			if (code && (errno != EEXIST)) {
				log("Create dir[%s] error[%d]", tmp.c_str(), errno);
				return -1;
			}
			for (int j = 0; j <= SECOND_DIR_MAX; j++) {
				snprintf(second, 5, "%03d", j);
				string tmp2 = tmp + "/" + string(second);
		    	code = File::mkdirNoRecursion(tmp2.c_str());
		    	if (code && (errno != EEXIST)) {
					log("Create dir[%s] error[%d]", tmp2.c_str(), errno);
					return -1;
		    	}
				memset(second, 0x0, 10);
			}
			memset(first, 0x0, 10);
		}
		
		return 0;
	}
```



下面，我们直接来看如何处理客户端的http请求，当连接对象CHttpConn收到客户端数据后，调用OnRead方法：

```cpp
void CHttpConn::OnRead()
{
    for (;;)
    {
        uint32_t free_buf_len = m_in_buf.GetAllocSize()
                - m_in_buf.GetWriteOffset();
        if (free_buf_len < READ_BUF_SIZE + 1)
            m_in_buf.Extend(READ_BUF_SIZE + 1);

        int ret = netlib_recv(m_sock_handle,
                m_in_buf.GetBuffer() + m_in_buf.GetWriteOffset(),
                READ_BUF_SIZE);
        if (ret <= 0)
            break;

        m_in_buf.IncWriteOffset(ret);

        m_last_recv_tick = get_tick_count();
    }

    // 每次请求对应一个HTTP连接，所以读完数据后，不用在同一个连接里面准备读取下个请求
    char* in_buf = (char*) m_in_buf.GetBuffer();
    uint32_t buf_len = m_in_buf.GetWriteOffset();
    in_buf[buf_len] = '\0';

    //log("OnRead, buf_len=%u, conn_handle=%u", buf_len, m_conn_handle); // for debug


    m_HttpParser.ParseHttpContent(in_buf, buf_len);

    if (m_HttpParser.IsReadAll())
    {
        string strUrl = m_HttpParser.GetUrl();
        log("IP:%s access:%s", m_peer_ip.c_str(), strUrl.c_str());
        if (strUrl.find("..") != strUrl.npos) {
            Close();
            return;
        }
        m_access_host = m_HttpParser.GetHost();
        if (m_HttpParser.GetContentLen() > HTTP_UPLOAD_MAX)
        {
            // file is too big
            log("content  is too big");
            char url[128];
            snprintf(url, sizeof(url), "{\"error_code\":1,\"error_msg\": \"上传文件过大\",\"url\":\"\"}");
            log("%s",url);
            uint32_t content_length = strlen(url);
            char pContent[1024];
            snprintf(pContent, sizeof(pContent), HTTP_RESPONSE_HTML, content_length,url);
            Send(pContent, strlen(pContent));
            return;
        }

        int nContentLen = m_HttpParser.GetContentLen();
        char* pContent = NULL;
        if(nContentLen != 0)
        {
            try {
                pContent =new char[nContentLen];
                memcpy(pContent, m_HttpParser.GetBodyContent(), nContentLen);
            }
            catch(...)
            {
                log("not enough memory");
                char szResponse[HTTP_RESPONSE_500_LEN + 1];
                snprintf(szResponse, HTTP_RESPONSE_500_LEN, "%s", HTTP_RESPONSE_500);
                Send(szResponse, HTTP_RESPONSE_500_LEN);
                return;
            }
        }
        Request_t request;
        request.conn_handle = m_conn_handle;
        request.method = m_HttpParser.GetMethod();;
        request.nContentLen = nContentLen;
        request.pContent = pContent;
        request.strAccessHost = m_HttpParser.GetHost();
        request.strContentType = m_HttpParser.GetContentType();
        request.strUrl = m_HttpParser.GetUrl() + 1;
        CHttpTask* pTask = new CHttpTask(request);
        if(HTTP_GET == m_HttpParser.GetMethod())
        {
        	g_GetThreadPool.AddTask(pTask);
        }
        else
        {
        	g_PostThreadPool.AddTask(pTask);
        }
    }
}
```


该方法先收取数据，接着解包，然后根据客户端发送的http请求到底是get还是post方法，分别往对应的get和post任务队列中丢一个任务CHttpTask。任务队列开始处理这个任务。我们以get请求的任务为例（Post请求与此类似）：

```cpp
void CHttpTask::run()
{

    if(HTTP_GET == m_nMethod)
    {
        OnDownload();
    }
    else if(HTTP_POST == m_nMethod)
    {
       OnUpload();
    }
    else
    {
        char* pContent = new char[strlen(HTTP_RESPONSE_403)];
        snprintf(pContent, strlen(HTTP_RESPONSE_403), HTTP_RESPONSE_403);
        CHttpConn::AddResponsePdu(m_ConnHandle, pContent, strlen(pContent));
    }
    if(m_pContent != NULL)
    {
        delete [] m_pContent;
        m_pContent = NULL;
    }
}
```

处理任务时，根据请求类型判断到底是客户端下载图片还是上传图片，如果是下载图片则从本机缓存的图片信息中找到该图片，并读取该图片数据，因为是聊天图片，所以一般不会很大，所以这里都是一次性读取图片字节内容，然后发出去。

```cpp
void  CHttpTask::OnDownload()
{
        uint32_t  nFileSize = 0;
        int32_t nTmpSize = 0;
        string strPath;
        if(g_fileManager->getAbsPathByUrl(m_strUrl, strPath ) == 0)
        {
            nTmpSize = File::getFileSize((char*)strPath.c_str());
            if(nTmpSize != -1)
            {
                char szResponseHeader[1024];
                size_t nPos = strPath.find_last_of(".");
                string strType = strPath.substr(nPos + 1, strPath.length() - nPos);
                if(strType == "jpg" || strType == "JPG" || strType == "jpeg" || strType == "JPEG" || strType == "png" || strType == "PNG" || strType == "gif" || strType == "GIF")
                {
                    snprintf(szResponseHeader, sizeof(szResponseHeader), HTTP_RESPONSE_IMAGE, nTmpSize, strType.c_str());
                }
                else
                {
                    snprintf(szResponseHeader,sizeof(szResponseHeader), HTTP_RESPONSE_EXTEND, nTmpSize);
                }
                int nLen = strlen(szResponseHeader);
                char* pContent = new char[nLen + nTmpSize];
                memcpy(pContent, szResponseHeader, nLen);
                g_fileManager->downloadFileByUrl((char*)m_strUrl.c_str(), pContent + nLen, &nFileSize);
                int nTotalLen = nLen + nFileSize;
                CHttpConn::AddResponsePdu(m_ConnHandle, pContent, nTotalLen);
            }
            else
            {
                int nTotalLen = strlen(HTTP_RESPONSE_404);
                char* pContent = new char[nTotalLen];
                snprintf(pContent, nTotalLen, HTTP_RESPONSE_404);
                CHttpConn::AddResponsePdu(m_ConnHandle, pContent, nTotalLen);
                log("File size is invalied\n");
                
            }
        }
        else
        {
        	int nTotalLen = strlen(HTTP_RESPONSE_500);
			char* pContent = new char[nTotalLen];
			snprintf(pContent, nTotalLen, HTTP_RESPONSE_500);
			CHttpConn::AddResponsePdu(m_ConnHandle, pContent, nTotalLen);
        }
}
```

这里需要说明一下的就是FileManager::getAbsPathByUrl在获取本地文件时，用了一个锁，该锁是为了防止同一个进程同时读取同一个文件，这个锁是“建议性”的，必须自己主动检测有没有上锁：

```cpp
int FileManager::getAbsPathByUrl(const string &url, string &path) {
	string relate;
	if (getRelatePathByUrl(url, relate)) {
		log("Get path from url[%s] error", url.c_str());
		return -1;
	}
	path = string(m_disk) + relate;
	return 0;
}
```

```cpp
u64 File::open(bool directIo) {
	assert(!m_opened);
	int flags = O_RDWR;
#ifdef __linux__	
	m_file = open64(m_path, flags);
#elif defined(__FREEBSD__) || defined(__APPLE__)
	m_file = ::open(m_path, flags);
#endif	
	if(-1 == m_file) {
		return errno;
	}
#ifdef __LINUX__
	if (directIo)
		if (-1 == fcntl(m_file, F_SETFL, O_DIRECT))
			return errno; 
#endif	
	struct flock lock;
	lock.l_type = F_WRLCK;
	lock.l_start = 0;
	lock.l_whence = SEEK_SET;
	lock.l_len = 0;
	if(fcntl(m_file, F_SETLK, &lock) < 0) {
		::close(m_file);
		return errno;
	}

	m_opened = true;
	u64 size = 0;
	u64 code = getSize(&size);
	if (code) {
		close();
		return code;
	}
	m_size = size;
	m_directIo = directIo;
	return 0;
}
```

注意上面的fcntl函数设置的flock锁。这个是linux特有的，应该学习掌握。

图片上传的逻辑和下载逻辑大致类似，这里就不再分析了。

当然，发送图片数据的包和前面的发送逻辑也是一样的，在OnWrite里面发送。发送完毕后会调用CHttpConn::OnSendComplete，在这个函数里面关闭http连接。这也就是说msfs与客户端的http连接也是短连接。

```cpp
void CHttpConn::OnSendComplete()
{
    Close();
}
```

关于msfs也就这么多内容了。不知道你有没有发现，在搞清楚db_proxy_server和msg_server之后，每个程序框架其实都是一样的，只不过业务逻辑稍微有一点差别。后面介绍的file_server和route_server都是一样的。我们也着重分析其业务代码。



好了，msfs服务就这么多啦。
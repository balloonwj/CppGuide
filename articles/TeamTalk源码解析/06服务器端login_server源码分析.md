# 06 服务器端login_server源码分析

login_server从严格意义上来说，是一个登录分流器，所以名字起的有点名不符实。该服务根据已知的msg_server上的在线用户数量来返回告诉一个即将登录的用户登录哪个msg_server比较合适。关于其程序框架的非业务代码我们已经在前面的两篇文章《服务器端db_proxy_server源码分析》和《服务器端msg_server源码分析》中介绍过了。这篇文章主要介绍下其业务代码。

首先，程序初始化的时候，会初始化如下功能：

```plain
//1. 在8008端口监听客户端连接

//2. 在8100端口上监听msg_server的连接

//3. 在8080端口上监听客户端http连接
```

其中连接对象CLoginConn代表着login_server与msg_server之间的连接；而CHttpConn代表着与客户端的http连接。我们先来看CLoginConn对象，上一篇文章中也介绍了其业务代码主要在其HandlePdu()函数中，可以看到这路连接主要处理哪些数据包：

```cpp
void CLoginConn::HandlePdu(CImPdu* pPdu)
{
	switch (pPdu->GetCommandId()) {
        case CID_OTHER_HEARTBEAT:
            break;
        case CID_OTHER_MSG_SERV_INFO:
            _HandleMsgServInfo(pPdu);
            break;
        case CID_OTHER_USER_CNT_UPDATE:
            _HandleUserCntUpdate(pPdu);
            break;
        case CID_LOGIN_REQ_MSGSERVER:
            _HandleMsgServRequest(pPdu);
            break;

        default:
            log("wrong msg, cmd id=%d ", pPdu->GetCommandId());
            break;
	}
}
```

命令号CID_OTHER_HEARTBEAT是与msg_server的心跳包。上一篇文章《服务器端msg_server源码分析》中介绍过，msg_server连上login_server后会立刻给login_server发一个数据包，该数据包里面含有该msg_server上的用户数量、最大可容纳的用户数量、自己的ip地址和端口号。

```cpp
list<user_conn_t> user_conn_list;
    CImUserManager::GetInstance()->GetUserConnCnt(&user_conn_list, cur_conn_cnt);
	char hostname[256] = {0};
	gethostname(hostname, 256);
    IM::Server::IMMsgServInfo msg;
    msg.set_ip1(g_msg_server_ip_addr1);
    msg.set_ip2(g_msg_server_ip_addr2);
    msg.set_port(g_msg_server_port);
    msg.set_max_conn_cnt(g_max_conn_cnt);
    msg.set_cur_conn_cnt(cur_conn_cnt);
    msg.set_host_name(hostname);
    CImPdu pdu;
    pdu.SetPBMsg(&msg);
    pdu.SetServiceId(SID_OTHER);
    pdu.SetCommandId(CID_OTHER_MSG_SERV_INFO);
	SendPdu(&pdu);
```

命令号是CID_OTHER_MSG_SERV_INFO。我们来看下login_server如何处理这个命令的：

```cpp
void CLoginConn::_HandleMsgServInfo(CImPdu* pPdu)
{
	msg_serv_info_t* pMsgServInfo = new msg_serv_info_t;
    IM::Server::IMMsgServInfo msg;
    msg.ParseFromArray(pPdu->GetBodyData(), pPdu->GetBodyLength());
    
	pMsgServInfo->ip_addr1 = msg.ip1();
	pMsgServInfo->ip_addr2 = msg.ip2();
	pMsgServInfo->port = msg.port();
	pMsgServInfo->max_conn_cnt = msg.max_conn_cnt();
	pMsgServInfo->cur_conn_cnt = msg.cur_conn_cnt();
	pMsgServInfo->hostname = msg.host_name();
	g_msg_serv_info.insert(make_pair(m_handle, pMsgServInfo));

	g_total_online_user_cnt += pMsgServInfo->cur_conn_cnt;

	log("MsgServInfo, ip_addr1=%s, ip_addr2=%s, port=%d, max_conn_cnt=%d, cur_conn_cnt=%d, "\
		"hostname: %s. ",
		pMsgServInfo->ip_addr1.c_str(), pMsgServInfo->ip_addr2.c_str(), pMsgServInfo->port,pMsgServInfo->max_conn_cnt,
		pMsgServInfo->cur_conn_cnt, pMsgServInfo->hostname.c_str());
}
```

其实所做的工作无非就是记录下的该msg_server上的ip、端口号、在线用户数量和最大可容纳用户数量等信息而已。存在一个全局map里面：

```cpp
map<uint32_t, msg_serv_info_t*> g_msg_serv_info;
```

```cpp
typedef struct  {
    string		ip_addr1;	// 电信IP
    string		ip_addr2;	// 网通IP
    uint16_t	port;
    uint32_t	max_conn_cnt;
    uint32_t	cur_conn_cnt;
    string 		hostname;	// 消息服务器的主机名
} msg_serv_info_t;
```

另外一个命令号CID_OTHER_USER_CNT_UPDATE，是当msg_server上的用户上线或下线时，msg_server给login_server发该类型的命令号，让login_server更新保存的msg_server的上的在线用户数量：

```cpp
void CLoginConn::_HandleUserCntUpdate(CImPdu* pPdu)
{
	map<uint32_t, msg_serv_info_t*>::iterator it = g_msg_serv_info.find(m_handle);
	if (it != g_msg_serv_info.end()) {
		msg_serv_info_t* pMsgServInfo = it->second;
        IM::Server::IMUserCntUpdate msg;
        msg.ParseFromArray(pPdu->GetBodyData(), pPdu->GetBodyLength());

		uint32_t action = msg.user_action();
		if (action == USER_CNT_INC) {
			pMsgServInfo->cur_conn_cnt++;
			g_total_online_user_cnt++;
		} else {
			pMsgServInfo->cur_conn_cnt--;
			g_total_online_user_cnt--;
		}

		log("%s:%d, cur_cnt=%u, total_cnt=%u ", pMsgServInfo->hostname.c_str(),
            pMsgServInfo->port, pMsgServInfo->cur_conn_cnt, g_total_online_user_cnt);
	}
}
```

命令号CID_LOGIN_REQ_MSGSERVER没用到。

接着说login_server与客户端的http连接处理，这个连接收取数据和解包是直接在CHttpConn的OnRead函数里面处理的：

```cpp
void CHttpConn::OnRead()
{
	for (;;)
	{
		uint32_t free_buf_len = m_in_buf.GetAllocSize() - m_in_buf.GetWriteOffset();
		if (free_buf_len < READ_BUF_SIZE + 1)
			m_in_buf.Extend(READ_BUF_SIZE + 1);

		int ret = netlib_recv(m_sock_handle, m_in_buf.GetBuffer() + m_in_buf.GetWriteOffset(), READ_BUF_SIZE);
		if (ret <= 0)
			break;

		m_in_buf.IncWriteOffset(ret);

		m_last_recv_tick = get_tick_count();
	}

	// 每次请求对应一个HTTP连接，所以读完数据后，不用在同一个连接里面准备读取下个请求
	char* in_buf = (char*)m_in_buf.GetBuffer();
	uint32_t buf_len = m_in_buf.GetWriteOffset();
	in_buf[buf_len] = '\0';
    
    // 如果buf_len 过长可能是受到攻击，则断开连接
    // 正常的url最大长度为2048，我们接受的所有数据长度不得大于1K
    if(buf_len > 1024)
    {
        log("get too much data:%s ", in_buf);
        Close();
        return;
    }

	//log("OnRead, buf_len=%u, conn_handle=%u\n", buf_len, m_conn_handle); // for debug

	
	m_cHttpParser.ParseHttpContent(in_buf, buf_len);

	if (m_cHttpParser.IsReadAll()) {
		string url =  m_cHttpParser.GetUrl();
		if (strncmp(url.c_str(), "/msg_server", 11) == 0) {
            string content = m_cHttpParser.GetBodyContent();
            _HandleMsgServRequest(url, content);
		} else {
			log("url unknown, url=%s ", url.c_str());
			Close();
		}
	}
}
```

如果用户发送的http请求的地址形式是http://192.168.226.128:8080/msg_server，即路径是/msg_server，则调用_HandleMsgServRequest()函数处理：

```cpp
void CHttpConn::_HandleMsgServRequest(string& url, string& post_data)
{
    msg_serv_info_t* pMsgServInfo;
    uint32_t min_user_cnt = (uint32_t)-1;
    map<uint32_t, msg_serv_info_t*>::iterator it_min_conn = g_msg_serv_info.end();
    map<uint32_t, msg_serv_info_t*>::iterator it;
    if(g_msg_serv_info.size() <= 0)
    {
        Json::Value value;
        value["code"] = 1;
        value["msg"] = "没有msg_server";
        string strContent = value.toStyledString();
        char* szContent = new char[HTTP_RESPONSE_HTML_MAX];
        snprintf(szContent, HTTP_RESPONSE_HTML_MAX, HTTP_RESPONSE_HTML, strContent.length(), strContent.c_str());
        Send((void*)szContent, strlen(szContent));
        delete [] szContent;
        return ;
    }
    
    for (it = g_msg_serv_info.begin() ; it != g_msg_serv_info.end(); it++) {
        pMsgServInfo = it->second;
        if ( (pMsgServInfo->cur_conn_cnt < pMsgServInfo->max_conn_cnt) &&
            (pMsgServInfo->cur_conn_cnt < min_user_cnt)) {
            it_min_conn = it;
            min_user_cnt = pMsgServInfo->cur_conn_cnt;
        }
    }
    
    if (it_min_conn == g_msg_serv_info.end()) {
        log("All TCP MsgServer are full ");
        Json::Value value;
        value["code"] = 2;
        value["msg"] = "负载过高";
        string strContent = value.toStyledString();
        char* szContent = new char[HTTP_RESPONSE_HTML_MAX];
        snprintf(szContent, HTTP_RESPONSE_HTML_MAX, HTTP_RESPONSE_HTML, strContent.length(), strContent.c_str());
        Send((void*)szContent, strlen(szContent));
        delete [] szContent;
        return;
    } else {
        Json::Value value;
        value["code"] = 0;
        value["msg"] = "";
        if(pIpParser->isTelcome(GetPeerIP()))
        {
            value["priorIP"] = string(it_min_conn->second->ip_addr1);
            value["backupIP"] = string(it_min_conn->second->ip_addr2);
            value["msfsPrior"] = strMsfsUrl;
            value["msfsBackup"] = strMsfsUrl;
        }
        else
        {
            value["priorIP"] = string(it_min_conn->second->ip_addr2);
            value["backupIP"] = string(it_min_conn->second->ip_addr1);
            value["msfsPrior"] = strMsfsUrl;
            value["msfsBackup"] = strMsfsUrl;
        }
        value["discovery"] = strDiscovery;
        value["port"] = int2string(it_min_conn->second->port);
        string strContent = value.toStyledString();
        char* szContent = new char[HTTP_RESPONSE_HTML_MAX];
        uint32_t nLen = strContent.length();
        snprintf(szContent, HTTP_RESPONSE_HTML_MAX, HTTP_RESPONSE_HTML, nLen, strContent.c_str());
        Send((void*)szContent, strlen(szContent));
        delete [] szContent;
        return;
    }
}
```

其实就是根据记录的msg_server的负载情况，返回一个可用的msg_server ip和端口给客户端，这是一个json格式：

```cpp
{
           "backupIP" : "localhost",
           "code" : 0,
           "discovery" : "http://192.168.226.128</span>/api/discovery",
           "msfsBackup" : "http://127.0.0.1:8700/",
           "msfsPrior" : "http://127.0.0.1:8700/",
           "msg" : "",
           "port" : "8000",
           "priorIP" : "localhost"
 }
```

里面含有msg_server和聊天图片存放的服务器地址（msfsPrior）字段。这样客户端可以拿着这个地址去登录msg_server和图片服务器了。

发出去这个json之后会调用OnWriteComplete()函数，这个函数立刻关闭该http连接，也就是说这个与客户端的http连接是短连接：

```cpp
void CHttpConn::OnWriteComlete()
{
    log("write complete ");
    Close();
}
```



login_server就这么多内容了。

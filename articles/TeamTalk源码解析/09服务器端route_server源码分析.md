# 09 服务器端route_server源码分析

route_server的作用主要是用户不同msg_server之间消息路由，其框架部分和前面的服务类似，没有什么好说的。我们这里重点介绍下它的业务代码，也就是其路由细节：

```cpp
void CRouteConn::HandlePdu(CImPdu* pPdu)
{
	switch (pPdu->GetCommandId()) {
        case CID_OTHER_HEARTBEAT:
            // do not take any action, heart beat only update m_last_recv_tick
            break;
        case CID_OTHER_ONLINE_USER_INFO:
            _HandleOnlineUserInfo( pPdu );
            break;
        case CID_OTHER_USER_STATUS_UPDATE:
            _HandleUserStatusUpdate( pPdu );
            break;
        case CID_OTHER_ROLE_SET:
            _HandleRoleSet( pPdu );
            break;
        case CID_BUDDY_LIST_USERS_STATUS_REQUEST:
            _HandleUsersStatusRequest( pPdu );
            break;
        case CID_MSG_DATA:
        case CID_SWITCH_P2P_CMD:
        case CID_MSG_READ_NOTIFY:
        case CID_OTHER_SERVER_KICK_USER:
        case CID_GROUP_CHANGE_MEMBER_NOTIFY:
        case CID_FILE_NOTIFY:
        case CID_BUDDY_LIST_REMOVE_SESSION_NOTIFY:
            _BroadcastMsg(pPdu, this);
            break;
        case CID_BUDDY_LIST_SIGN_INFO_CHANGED_NOTIFY:
            _BroadcastMsg(pPdu);
            break;
        
	default:
		log("CRouteConn::HandlePdu, wrong cmd id: %d ", pPdu->GetCommandId());
		break;
	}
}
```

上面是route_server处理的消息类型，我们逐一来介绍：

1. CID_OTHER_ONLINE_USER_INFO

这个消息是msg_server连接上route_server后告知route_server自己上面的用户登录情况。route_server处理后，只是简单地记录一下每个msg_server上的用户数量和用户id：

```cpp
void CRouteConn::_HandleOnlineUserInfo(CImPdu* pPdu)
{
    IM::Server::IMOnlineUserInfo msg;
    CHECK_PB_PARSE_MSG(msg.ParseFromArray(pPdu->GetBodyData(), pPdu->GetBodyLength()));

	uint32_t user_count = msg.user_stat_list_size();

	log("HandleOnlineUserInfo, user_cnt=%u ", user_count);

	for (uint32_t i = 0; i < user_count; i++) {
        IM::BaseDefine::ServerUserStat server_user_stat = msg.user_stat_list(i);
		_UpdateUserStatus(server_user_stat.user_id(), server_user_stat.status(), server_user_stat.client_type());
	}
}
```



2. CID_OTHER_USER_STATUS_UPDATE

这个消息是当某个msg_server上有用户上下线时，msg_server会给route_server发送自己最近的用户数量和在线用户id信息，route_server的处理也就是更新下记录的该msg_server上的用户数量和用户id。

3. CID_OTHER_ROLE_SET

这个消息没看懂，感觉是确定主从关系的，不过感觉没什么用？

4. CID_OTHER_GET_DEVICE_TOKEN_RSP

这个消息用户获取某个用户的登录情况，比如pc登录、安卓版登录还是ios登录，用于某些特殊消息的处理，比如文件发送不会推给移动在线的用户。

5. CID_MSG_DATA:
   CID_SWITCH_P2P_CMD:
   CID_MSG_READ_NOTIFY:
   CID_OTHER_SERVER_KICK_USER:
   CID_GROUP_CHANGE_MEMBER_NOTIFY:
   CID_FILE_NOTIFY:
   CID_BUDDY_LIST_REMOVE_SESSION_NOTIFY

   CID_BUDDY_LIST_SIGN_INFO_CHANGED_NOTIFY

这几个消息都是往外广播消息，由于msg_server 可以配置多个，A给B发了一条消息，必须广播在各个msg_server 才能知道B到底在哪个msg_server上。

```cpp
void CRouteConn::_BroadcastMsg(CImPdu* pPdu, CRouteConn* pFromConn)
{
	ConnMap_t::iterator it;
	for (it = g_route_conn_map.begin(); it != g_route_conn_map.end(); it++) {
		CRouteConn* pRouteConn = (CRouteConn*)it->second;
		if (pRouteConn != pFromConn) {
			pRouteConn->SendPdu(pPdu);
		}
	}
}
```

也就是说CRouteConn代表着到msg_server的连接。



route_server的介绍就这么多了，虽然比较简单，但是这种路由的思想却是非常值得我们学习。网络通信数据包的在不同ip地址的路由最终被送达目的地，也就是这个原理。
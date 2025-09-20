<!--
 * @Author: lion
 * @Date: 2025-09-19 16:19:30
 * @LastEditors: Chen Yiran 1060874648@qq.com
 * @LastEditTime: 2025-09-19 16:38:31
 * @FilePath: \linux-study\README.md
 * @Description: 
 * 
-->
## linux-study
### linux网络编程（2025年9月19日）
1. 客户端和服务端   
    1. **客户端**：
            在网络程序中，如果一个程序主动和外面的程序通信，那么我们把这个程序称为客户端程序。 比如我们使用ftp程序从另外一
            个地方获取文件的时候，是我们的ftp程序主动同外面进行通信（获取文件）， 所以这个地方我们的ftp程序就是客户端程序。

    2. **服务端**：
            和客户端相对应的程序即为服务端程序。被动的等待外面的程序来和自己通讯的程序称为服务端程序。
            比如上面的文件获取中，另外一个地方的程序就是服务端，我们从服务端获取文件过来。

2. TCP/UDP   
    1. **TCP(Transfer Control Protocol)** 传输控制协议是一种面向连接的协议, 当我们的网络程序使用这个协议的时候,网络可以保证我们的客户端和服务端的连接是 **可靠的,安全** 的.      

    2. **UDP(User Datagram Protocol)** 用户数据报协议是一种非面向连接的协议,这种协议并 **不能保证我们的网络程序的连接是可靠的(在视频传输情景下可能使用udp，可以存在一定的失误，但是效率较高保证传输流畅)** ,所以我们现在编写的程序一般是采用TCP协议的.

3. ip和端口：一台电脑的多个程序可以是同一个ip，但是端口会不同，以此来表示不同的请求，所以ip和端口共同来表示源或目的

4. 解释tcp服务器端和客户端的流程：
   1. 服务器：fd = socket()创建一个基本端点   bind()绑定自己的ip和端口信息(把fd和ip端口绑定)  listen()开始检测数据  accept()建立一条连接 send()/recv()数据收发
   2. 客户端：fd = socket()创建一个基本端点  connect()连接服务器的ip和端口 send()/recv()数据收发

5. 在tcp服务器端实现代码中，为了实现多个客户端连接，在accept到一个请求后就会使用fork()创建一个子进程来处理这个请求，父进程继续回到accept()等待下一个请求。
```
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdio.h>
#include <signal.h>
/* socket
* bind
  * listensocket()
  * accept
  * send/recv
    */

#define SERVER_PORT 8888
#define BACKLOG     10

int main(int argc, char **argv)
{
    int iSocketServer;
    int iSocketClient;
    struct sockaddr_in tSocketServerAddr;
    struct sockaddr_in tSocketClientAddr;
    int iRet;
    int iAddrLen;
    
    int iRecvLen;
    unsigned char ucRecvBuf[1000];
    
    int iClientNum = -1;
    
    signal(SIGCHLD,SIG_IGN);
    
    iSocketServer = socket(AF_INET, SOCK_STREAM, 0);
    if (-1 == iSocketServer)
    {
        printf("socket error!\n");
        return -1;
    }
    
    tSocketServerAddr.sin_family      = AF_INET;
    tSocketServerAddr.sin_port        = htons(SERVER_PORT);  /* host to net, short */
    tSocketServerAddr.sin_addr.s_addr = INADDR_ANY;
    memset(tSocketServerAddr.sin_zero, 0, 8);
    
    iRet = bind(iSocketServer, (const struct sockaddr *)&tSocketServerAddr, sizeof(struct sockaddr));
    if (-1 == iRet)
    {
        printf("bind error!\n");
        return -1;
    }
    
    iRet = listen(iSocketServer, BACKLOG);
    if (-1 == iRet)
    {
        printf("listen error!\n");
        return -1;
    }
    
    while (1)
    {
        iAddrLen = sizeof(struct sockaddr);
        iSocketClient = accept(iSocketServer, (struct sockaddr *)&tSocketClientAddr, &iAddrLen);
        if (-1 != iSocketClient)
        {
            iClientNum++;
            printf("Get connect from client %d : %s\n",  iClientNum, inet_ntoa(tSocketClientAddr.sin_addr));
            if (!fork())
            {
                /*zichengxv code*/
                while (1)
                {
                    
                    iRecvLen = recv(iSocketClient, ucRecvBuf, 999, 0);
                    if (iRecvLen <= 0)
                    {
                        close(iSocketClient);
                        return -1;
                    }
                    else
                    {
                        ucRecvBuf[iRecvLen] = '\0';
                        printf("Get Msg From Client %d: %s\n", iClientNum, ucRecvBuf);
                    }
                }				
            }
        }
    }
    
    close(iSocketServer);
    return 0;
}
```

6. tcp客户端实现代码
```
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdio.h>

/* socket
* connect
* send/recv
  */

#define SERVER_PORT 8888

int main(int argc, char **argv)
{
    int iSocketClient;
    struct sockaddr_in tSocketServerAddr;

	int iRet;
	unsigned char ucSendBuf[1000];
	int iSendLen;

	if (argc != 2)
	{
		printf("Usage:\n");
		printf("%s <server_ip>\n", argv[0]);
		return -1;
	}

	iSocketClient = socket(AF_INET, SOCK_STREAM, 0);

	tSocketServerAddr.sin_family      = AF_INET;
	tSocketServerAddr.sin_port        = htons(SERVER_PORT);  /* host to net, short */
 	//tSocketServerAddr.sin_addr.s_addr = INADDR_ANY;
 	if (0 == inet_aton(argv[1], &tSocketServerAddr.sin_addr))
 	{
		printf("invalid server_ip\n");
		return -1;
	}
	memset(tSocketServerAddr.sin_zero, 0, 8);


	iRet = connect(iSocketClient, (const struct sockaddr *)&tSocketServerAddr, sizeof(struct sockaddr));	
	if (-1 == iRet)
	{
		printf("connect error!\n");
		return -1;
	}

	while (1)
	{
		if (fgets(ucSendBuf, 999, stdin))
		{
			iSendLen = send(iSocketClient, ucSendBuf, strlen(ucSendBuf), 0);
			if (iSendLen <= 0)
			{
				close(iSocketClient);
				return -1;
			}
		}
	}
	
	return 0;
}
```







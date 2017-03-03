title: 学习HTTP协议-用socket实现http访问
date: 2016-03-04 22:16:30
tags:
	- Http协议
	- 技术相关
---

相信大家都知道 http 报文这东西吧？http 报文分两种，请求报文和响应报文。

## __请求报文__

客户端向服务端发送的就是请求报文，它可告诉服务端自己需要什么样的资源，也可以将一些文件或者数据上传给服务端。

请求报文格式如下：

{% img /学习HTTP协议-用socket实现http访问/1.jpg %}

请求报文分为三个部分：

- 请求行，如：

> GET / HTTP/1.1

这个请求行表明了这次请求使用的是 GET 方法，访问的是网站的根目录，使用的 HTTP 协议版本是 1.1。

- 请求头部，如：

> Host: www.baidu.com
> Connection: keep-alive

- 请求包体

用来携带数据

### _GET 方法_

GET 方法是 HTTP 中最基础的方法，我们在浏览器地址栏输入网站浏览网页使用的都是 GET 方法：

> http://www.islinjw.cn/okhttp_cookie_demo/checkverifycode.php

当然有时候服务器需要根据用户传递的信息去返回对应的数据，GET 方法用下面的形式传递信息给服务器：

> http://www.islinjw.cn/okhttp_cookie_demo/checkverifycode.php?verifycode=qwjuy&format=json

这里告诉给服务器 verifycode=qwjuy 和 format=json ，服务器会根据用户传过来的信息返回不同的数据。

这个时候的请求行长这个样子，URL 上就携带了 GET 传递的数据：

> GET /okhttp_cookie_demo/checkverifycode.php?verifycode=qwjuy&format=json HTTP/1.1

这里再说一句题外话，并不是说如果在 URL 里面没有见到 “?” 这个符号，客户端就没有传递数据给服务器。有一种叫做网页伪静态化的技术可以实现不带问号的 URL 使用 GET 方法传递数据。

### _POST 方法_

GET 方法的参数都显示在 URl 上，这样对于诸如账户密码的敏感信息来说太不安全，而且也很难传递想图片这样的数据。所以就有了 POST 方法。

使用 POST 方法传递的数据并不会显示在 URL 上，而是保存在请求包体中，当然 HTTP 协议是明文传输的，所以把账户密码直接用 POST 传递也是不安全的，需要程序员自己进行加密处理。

### _HTTP 协议方法列表_

|序号 |方法 |描述|
|---|---|---|
|1 |GET |请求指定的页面信息，并返回实体主体。|
|2 |HEAD |类似于get请求，只不过返回的响应中没有具体的内容，用于获取报头|
|3 |POST |向指定资源提交数据进行处理请求（例如提交表单或者上传文件）。数据被包含在请求体中。POST请求可能会导致新的资源的建立和/或已有资源的修改。|
|4 |PUT |从客户端向服务器传送的数据取代指定的文档的内容。|
|5 |DELETE |请求服务器删除指定的页面。|
|6 |CONNECT |HTTP/1.1协议中预留给能够将连接改为管道方式的代理服务器。|
|7 |OPTIONS	|允许客户端查看服务器的性能。|
|8 |TRACE |回显服务器收到的请求，主要用于测试或诊断。|
|9 |PATCH |实体中包含一个表，表中说明与该URI所表示的原内容的区别。|
|10 |MOVE |请求服务器将指定的页面移至另一个网络地址。|
|11	|COPY |请求服务器将指定的页面拷贝至另一个网络地址。|
|12	|LINK |请求服务器建立链接关系。|
|13	|UNLINK |断开链接关系。|
|14	|WRAPPED |允许客户端发送经过封装的请求。|
|15	|Extension-mothed |在不改动协议的前提下，可增加另外的方法。|

## __响应报文__

服务端接收到请求报文之后，了解到客户端需要什么样的服务之后就会返回响应报文给客户端。

响应报文格式如下：

{% img /学习HTTP协议-用socket实现http访问/2.jpg %}

- 状态行，如：

> HTTP/1.1 200 OK

- 响应头部，如：

> Date: Fri, 04 Mar 2016 11:04:01 GMT
> Server: Apache/2.4.7 (Ubuntu)
> X-Powered-By: PHP/5.5.9-1ubuntu4.14
> Expires: Thu, 19 Nov 1981 08:52:00 GMT
> Cache-Control: no-store, no-cache, must-revalidate, post-check=0, pre-check=0
> Pragma: no-cache
> Content-Length: 20
> Keep-Alive: timeout=5, max=100
> Connection: Keep-Alive
> Content-Type: text/html

- 响应包体，即页面显示的内容，如：

> {"result":"success"}

### _状态码_

状态码由三位数字组成，第一位数字表示响应的类型，常用的状态码有五大类如下所示：

　　1xx：表示服务器已接收了客户端请求，客户端可继续发送请求;

　　2xx：表示服务器已成功接收到请求并进行处理;

　　3xx：表示服务器要求客户端重定向;

　　4xx：表示客户端的请求有非法内容;

　　5xx：表示服务器未能正常处理客户端的请求而出现意外错误;

## __使用 Socket 发送 HTTP 请求报文__

我们知道 HTTP 协议是基于 TCP 的，而我们可以使用 Socket 进行 TCP 连接，所以在充分理解 HTTP 报文之后我们就可以用 socket 实现自己的 HTTP 访问了。

### _访问网页_

首先我们看看怎样用 socket 实现 http 访问网页,这里我们尝试使用 GET 方法访问 [www.islinjw.cn](http://www.islinjw.cn)。

流程如下：
1. 使用 socket 连接服务器
2. 发送请求报文
3. 接收响应报文
4. 断开 socket 连接

重点在于发送请求报文，其他步骤和一般的 socket 程序是没有什么区别的。

请求报文分为三个部分还记得吗？

- 请求行
使用 HTTP/1.1 协议的 GET 方法访问网站的根目录：

> GET / HTTP/1.1

- 请求头部
Host 是请求头部唯一必须携带的数据，要不然能接收到数据，但服务器返回302、400这样的错误代码。原因是服务器可能使用了虚拟服务器技术，一台服务器托管了多个网站，即多个网站通过dns解析到同样的ip地址。像这里我们访问 www.islinjw.cn 主机：

> Host: www.islinjw.cn

- 请求实体：
但我们这里因为只是单纯的获取页面，并没有传递数据给服务器，所以报文实体为空。

每个部分之间使用 "\\r\\n" 分割。但需要在请求报文的最后加多一个 "\\n"。为什么？还记得请求头部和请求实体之间有一个什么东西吗？对，空行！因为这里没有请求实体，所以报文最后就是一个空行。如果没有它，服务器不会返回响应报文，程序就会一直阻塞在那里。

所以最终发送的报文就是:

> GET / HTTP/1.1\r\nHost: www.islinjw.cn\r\n\n

代码如下：

```cpp
    void TestRequest(){
        void TestRequest(){
        SOCKET sock_client = Connect(SERVER_IP.c_str());

        string data = "";

        //请求行
        data += "GET / HTTP/1.1\r\n";

        //请求头部
        data += "Host: www.islinjw.cn\r\n\n";

        send(sock_client,data.c_str(),data.size(),0);

        PrintRecvData(sock_client);
        Disconnect(sock_client);
    }
```

服务器返回的响应报文如下（对，这个网站就是一个 hello world 在那里而已）：

{% img /学习HTTP协议-用socket实现http访问/3.jpg %}

### _使用 GET 方法_

为了验证是否真的传送了数据给服务器，我写了一个 demo 页面 [www.islinjw.cn/http_packet_demo/demo.php](http://www.islinjw.cn/http_packet_demo/demo.php)。这个页面的功能很简单，就是把接收到的 GET 数据和 POST 数据通过 json 格式打印出而已：

{% img /学习HTTP协议-用socket实现http访问/4.jpg %}

我们首先写一个函数用来把 map 转化成 GET 方法的参数格式：

```cpp
    string MsgToString(const map<string,string>& msg){
        string result = "";
        map<string,string>::const_iterator i = msg.begin();
        while(i!=msg.end()){
            result += i->first + "=" + i->second;
            i++;
            if(i!=msg.end()){
                result += "&";
            }
        }
        return result;
    }
```

之前提到，GET 方法的数据是通过 URL 来传递的，所以只需要把得到的 GET 方法参数拼接到请求行的 URL 后面就行了：

```cpp
	string url = "/http_packet_demo/demo.php";
	url += "?" + MsgToString(msg);
    cout<<"url : "<<url<<endl;

	string packet = "";

	//请求行
	packet += "GET " + url + " HTTP/1.1\r\n";
```

其他的和刚刚讲的访问网页的方式一模一样：

```cpp
	void TestGet(const map<string,string>& msg){
        //连接服务器主机
        SOCKET sock_client = Connect(SERVER_IP.c_str());

        string url = "/http_packet_demo/demo.php";
        url += "?" + MsgToString(msg);

        cout<<"url : "<<url<<endl;

        string packet = "";

        //请求行
        packet += "GET " + url + " HTTP/1.1\r\n";

        //请求头部
        packet += "Host: www.baidu.com\r\n";

        //空行
        packet += "\n";

        send(sock_client, packet.c_str(), packet.size(),0);

        PrintRecvData(sock_client);
        Disconnect(sock_client);
    }
```

我们这样调用:

```cpp
	map<string,string> msg;
	msg["abc"] = "123";
	msg["def"] = "456";

	TestGet(msg);
```

URL 长这个样子：

{% img /学习HTTP协议-用socket实现http访问/5.jpg %}

服务器返回的响应报文如下：：

{% img /学习HTTP协议-用socket实现http访问/6.jpg %}

### _使用 POST 方法_

使用 POST 方法会复杂那么一点点。首先请求行没有什么特别的，就是指定了 POST 方法和我们的页面，而且 URL 没有带数据:

```cpp
	string url = "/http_packet_demo/demo.php";

	string packet = "";

	//请求行
	packet += "POST " + url + " HTTP/1.1\r\n";
```

但因为 POST 携带的数据不一定是字符串，有可能是图片等二进制图片，所以就需要在请求头部告诉服务器携带的数据的类型和数据的长度:

```cpp
	//请求头部
	packet += "Host: www.islinjw.cn\r\n";
	packet += "Content-Type:application/x-www-form-urlencoded\r\n"; //指定post传递的数据类型
	packet += "Content-Length: " + ss.str() + "\r\n"; //标记post传递的数据的长度
```

之后就是一个空行和携带了数据的请求实体了：

```cpp
	//空行
	packet += "\n";

	//post数据
	packet += data;
```

所以整个方法长这个样子:

```cpp
    void TestPost(const map<string,string>& msg){
        string data = MsgToString(msg);

        stringstream ss;
        ss<<data.length();

        //连接服务器主机
        SOCKET sock_client = Connect(SERVER_IP.c_str());

        string url = "/http_packet_demo/demo.php";

        string packet = "";

        //请求行
        packet += "POST " + url + " HTTP/1.1\r\n";

        //请求头部
        packet += "Host: www.islinjw.cn\r\n";
        packet += "Content-Type:application/x-www-form-urlencoded\r\n"; //指定post传递的数据类型
        packet += "Content-Length: " + ss.str() + "\r\n"; //标记post传递的数据的长度

        //空行
        packet += "\n";

        //post数据
        packet += data;

        send(sock_client, packet.c_str(), packet.size(),0);

        PrintRecvData(sock_client);
        Disconnect(sock_client);
    }
```

发送的数据如下：

```cpp
	map<string,string> msg;
	msg["abc"] = "123";
	msg["def"] = "456";

	TestPost(msg);
```

服务器返回的响应实体如下：

{% img /学习HTTP协议-用socket实现http访问/7.jpg %}

## __demo 完整代码__

```cpp
    #include "stdafx.h"

    #include <Winsock2.h>
    #include <iostream>
    #include <map>
    #include <string>
    #include <sstream>

    #pragma comment( lib, "ws2_32.lib" )

    using namespace std;

    const string SERVER_IP = "182.254.231.66";

    SOCKET Connect(const char* ip){

        //固定格式
        WORD wVersionRequested;
        WSADATA wsaData;
        int err;

        wVersionRequested = MAKEWORD( 1, 1 );

        err = WSAStartup( wVersionRequested, &wsaData );
        if ( err != 0 ) {
            return INVALID_SOCKET;
        }


        if ( LOBYTE( wsaData.wVersion ) != 1 ||
            HIBYTE( wsaData.wVersion ) != 1 ) {
            WSACleanup( );
            return INVALID_SOCKET;
        }

        SOCKET sock_client=socket(AF_INET,SOCK_STREAM,0);

        SOCKADDR_IN addrSrv;
        addrSrv.sin_addr.S_un.S_addr=inet_addr(ip);
        addrSrv.sin_family=AF_INET;
        addrSrv.sin_port=htons(80);//http端口为80
        connect(sock_client,(SOCKADDR*)&addrSrv,sizeof(SOCKADDR));

        return sock_client;
    }

    void Disconnect(SOCKET sock_client){
        closesocket(sock_client);
        WSACleanup();
    }

    void PrintRecvData(SOCKET sock_client){
        cout<<"Recv data :"<<endl;

        int len = 0;
        char recvBuf[1024];
        while((len = recv(sock_client,recvBuf,1023,0))>0){
            recvBuf[len] = '\0';
            cout<<recvBuf;
        }
        cout<<endl;
    }

    void TestRequest(){
        SOCKET sock_client = Connect(SERVER_IP.c_str());

        //最后必须多一个空行（\n），要不然会阻塞住
        //这个空行其实是报文首部和报文主体的分割符号，但这里请求不需要报文主体，所以是请求报文的结束
        //string data = "GET / HTTP/1.1\r\nHost: www.islinjw.cn\r\n";

        //HOST也是必须的，要不然能接收到数据，但服务器返回302、400这样的错误代码
        //原因是服务器可能使用了虚拟服务器技术，一台服务器托管了多个网站，即多个网站通过dns解析到同样的ip地址
        //所以在发送http请求时必须带上HOST
        //string data = "GET / HTTP/1.1\r\n\n";

        string data = "";

        //请求行
        data += "GET / HTTP/1.1\r\n";

        //请求头部
        data += "Host: www.islinjw.cn\r\n\n";

        send(sock_client,data.c_str(),data.size(),0);

        PrintRecvData(sock_client);
        Disconnect(sock_client);
    }

    string MsgToString(const map<string,string>& msg){
        string result = "";
        map<string,string>::const_iterator i = msg.begin();
        while(i!=msg.end()){
            result += i->first + "=" + i->second;
            i++;
            if(i!=msg.end()){
                result += "&";
            }
        }
        return result;
    }

    void TestGet(const map<string,string>& msg){
        //连接服务器主机
        SOCKET sock_client = Connect(SERVER_IP.c_str());

        string url = "/http_packet_demo/demo.php";
        url += "?" + MsgToString(msg);

        cout<<"url : "<<url<<endl;

        string packet = "";

        //请求行
        packet += "GET " + url + " HTTP/1.1\r\n";

        //请求头部
        packet += "Host: www.baidu.com\r\n";

        //空行
        packet += "\n";

        send(sock_client, packet.c_str(), packet.size(),0);

        PrintRecvData(sock_client);
        Disconnect(sock_client);
    }

    void TestPost(const map<string,string>& msg){
        string data = MsgToString(msg);

        stringstream ss;
        ss<<data.length();

        //连接服务器主机
        SOCKET sock_client = Connect(SERVER_IP.c_str());

        string url = "/http_packet_demo/demo.php";

        string packet = "";

        //请求行
        packet += "POST " + url + " HTTP/1.1\r\n";

        //请求头部
        packet += "Host: www.islinjw.cn\r\n";
        packet += "Content-Type:application/x-www-form-urlencoded\r\n"; //指定post传递的数据类型
        packet += "Content-Length: " + ss.str() + "\r\n"; //标记post传递的数据的长度

        //空行
        packet += "\n";

        //post数据
        packet += data;

        send(sock_client, packet.c_str(), packet.size(),0);

        PrintRecvData(sock_client);
        Disconnect(sock_client);
    }

    int main(int argc, char* argv[])
    {
        map<string,string> msg;
        msg["abc"] = "123";
        msg["def"] = "456";

        TestRequest();
        cout<<"\n\n\n"<<endl;
        TestGet(msg);
        cout<<"\n\n\n"<<endl;
        TestPost(msg);

        return 0;
    }



```
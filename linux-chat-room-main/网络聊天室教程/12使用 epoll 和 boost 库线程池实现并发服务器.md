# 使用 epoll 和 boost 库线程池实现并发服务器

### 实验介绍

前面实验我们的服务器都是简单使用多线程实现的，需要为每一位客户都维护一个线程，这种方式在并发性能上表现一般，因此这次实验我们学习如何利用 epoll 和 boost 库线程池来实现一个高性能的并发服务器。

#### 知识点

- IO 多路复用基础知识
- epoll 的相关原理及使用
- 线程池的基础知识
- boost 库的线程池使用

### IO 多路复用

IO 多路复用是一种常见的 IO 模型，我们可以使用一个线程来完成对多个文件描述符的监听，线程会阻塞在系统调用（系统调用可以理解为内核提供的函数）上，而不是阻塞在真正的 IO 函数（即 send、recv 等函数）上，内核会负责监听我们指定的多个文件描述符，一旦某个文件描述符上有数据可读或者可写时，内核才会通知我们，然后我们再调用 IO 函数去读写数据。

通过这种 IO 多路复用的方式，我们仅仅采用一个线程就可以监听到所有用户对应的文件描述符，而无需再像以前一样为每个用户维护一个线程，因为多线程之间的上下文切换会影响并发性能。

#### IO 多路复用常见的系统调用

IO 多路复用的系统调用一般有三种：select、poll、epoll。

select 和 poll 可以跨平台使用，但是 select 最多只支持监听 1024 个文件描述符，而且需要重新注册感兴趣的文件描述符集合，而且 select 和 poll 都需要不断轮询文件描述符，而且返回了数组之后也需要我们轮询判断哪些文件描述符就绪，因此我们不采用这两种方式。

epoll 是基于 Linux 的系统调用，会在内核中建立一颗红黑树和就绪链表，可以解决 select 和 poll 上述的问题，因此我们采用 epoll 来实现。

#### epoll 的边缘触发和水平触发

epoll 有边缘触发 ET 和水平触发 LT 两种工作模式。

水平触发（LT）：当 epoll_wait 检测到某个文件描述符上有事件发生并将此事件通知进程后，进程可以不立即处理此事件，但是下一次调用 epoll_wait 的时候，epoll_wait 还会将此事件通知给进程。

边缘触发（ET）：当 epoll_wait 检测到文件描述符有事件发生并将此通知进程后，进程必须立即处理该事件，因为后续的 epoll_wait 调用不会再向进程通知这一事件。

边缘触发可以减少同一个文件描述符上事件触发的次数，提高效率，但是需要我们被通知事件后把文件描述符上的数据一次性读完，因此我们一般在边缘触发中的 IO 操作会用一个 while 循环来控制，直到文件描述符上的数据被处理完为止。但这也要求我们将 IO 操作函数设为非阻塞的状态，如果是阻塞的 IO 函数，那么当我们将数据读完之后，必然会导致进程阻塞。

### 线程池

有了 epoll 来帮助我们监听文件描述符，我们还需要调用 IO 函数和处理具体的业务逻辑，这个过程还是需要用到多线程来实现高并发，但我们这里采用线程池的方式来实现。

所谓线程池，就是预先创建好多个线程，然后让这些线程可以复用，不断地进行 IO 操作并处理业务逻辑，处理完一个业务之后继续循环，而不是直接将其销毁。

通过这种方式，我们可以避免不断创建线程、销毁线程带来的开销，让线程实现复用，提高性能。

#### boost 库的线程池

在 boost 库的 asio 组件中，内置了线程池的对象和相关方法，我们只需要安装好 boost 库，然后在代码中使用即可。

### 实验要求

修改以前实现的服务器，改成 epoll 监听+boost 库线程池处理事件的 Reactor 模式实现的并发服务器，epoll 采用 ET 边缘触发。

### 代码实现

首先新增我们需要的头文件，如下：

`global.h`：

```cpp
#ifndef _GLOBAL_H
#define _GLOBAL_H

#include <sys/types.h>
#include <sys/socket.h>
#include <stdio.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <fcntl.h>
#include <sys/shm.h>
#include <iostream>
#include <thread>
#include <vector>
#include <mysql/mysql.h>
#include <unordered_map>
#include <pthread.h>
#include <set>
#include <hiredis/hiredis.h>
#include <fstream>
//以下新增
#include<sys/epoll.h>
#include<boost/bind.hpp>
#include<boost/asio.hpp>
#include<errno.h>
using namespace std;

#endif
```

然后在 `server.h` 中新增 `setnonblocking` 函数的声明：

```cpp
#ifndef SERVER_H
#define SERVER_H

#include "global.h"

class server{
    private:
        int server_port;
        int server_sockfd;
        string server_ip;
        static vector<bool> sock_arr;
        static unordered_map<string,int> name_sock_map;//名字和套接字描述符
        static unordered_map<string,string> from_to_map;//记录用户xx要向用户yy发送信息
        static unordered_map<int,set<int> > group_map;//记录群号和套接字描述符集合
        static pthread_mutex_t name_sock_mutx;//互斥锁，锁住需要修改name_sock_map的临界区
        static pthread_mutex_t group_mutx;//互斥锁，锁住需要修改group_map的临界区
        static pthread_mutex_t from_mutex;//互斥锁，锁住修改from_to_map的临界区
    public:
        server(int port,string ip);
        ~server();
        void run();
        static void RecvMsg(int epollfd,int conn);
        static void HandleRequest(int epollfd,int conn,string str,tuple<bool,string,string,int,int> &info);
        static void setnonblocking(int conn); //将套接字设为非阻塞
};
#endif
```

在 `server.cpp` 中给出函数的具体定义：

```cpp
//将参数的文件描述符设为非阻塞
void server::setnonblocking(int sock)
{
    int opts;
    opts=fcntl(sock,F_GETFL);
    if(opts<0)
    {
        perror("fcntl(sock,GETFL)");
        exit(1);
    }
    opts = opts|O_NONBLOCK;
    if(fcntl(sock,F_SETFL,opts)<0)
    {
        perror("fcntl(sock,SETFL,opts)");
        exit(1);
    }
}
```

并在 server.cpp 的开头添加互斥锁定义和初始化：

```cpp
#include "server.h"

vector<bool> server::sock_arr(10000,false);
unordered_map<string,int> server::name_sock_map;//名字和套接字描述符
unordered_map<string,string> server::from_to_map;//记录用户xx要向用户yy发送信息
unordered_map<int,set<int> > server::group_map;//记录群号和套接字描述符集合
pthread_mutex_t server::name_sock_mutx;//互斥锁，锁住需要修改name_sock_map的临界区
pthread_mutex_t server::group_mutx;//互斥锁，锁住需要修改group_map的临界区
pthread_mutex_t server::from_mutex;//自旋锁，锁住修改from_to_map的临界区

server::server(int port,string ip):server_port(port),server_ip(ip){
    pthread_mutex_init(&name_sock_mutx, NULL); //创建互斥锁
    pthread_mutex_init(&group_mutx, NULL); //创建互斥锁
    pthread_mutex_init(&from_mutex, NULL); //创建互斥锁
}
```

下一步需要对 run 方法进行大改，首先使用 epoll_create 生成 epoll 专用的文件描述符，然后使用 epoll_ctl 注册事件，然后定义线程池，最后通过一个循环来等待 epoll 事件发生，并通过线程池的 post 方法将任务加入队列，代码如下：

```cpp
//将参数的文件描述符设为非阻塞
void server::setnonblocking(int sock)  
{  
    int opts;  
    opts=fcntl(sock,F_GETFL);  
    if(opts<0)  
    {  
        perror("fcntl(sock,GETFL)");  
        exit(1);  
    }  
    opts = opts|O_NONBLOCK;  
    if(fcntl(sock,F_SETFL,opts)<0)  
    {  
        perror("fcntl(sock,SETFL,opts)");  
        exit(1);  
    }      
}  
void server::run(){
    //listen的backlog大小
    int LISTENQ=200;
    int i, maxi, listenfd, connfd, sockfd,epfd,nfds;
    ssize_t n;
    //char line[MAXLINE];
    socklen_t clilen;
    //声明epoll_event结构体的变量,ev用于注册事件,数组用于回传要处理的事件
    struct epoll_event ev,events[10000];
    //生成用于处理accept的epoll专用的文件描述符
    epfd=epoll_create(10000);
    struct sockaddr_in clientaddr;
    struct sockaddr_in serveraddr;
    listenfd = socket(PF_INET, SOCK_STREAM, 0);
    //把socket设置为非阻塞方式
    setnonblocking(listenfd);
    //设置与要处理的事件相关的文件描述符
    ev.data.fd=listenfd;
    //设置要处理的事件类型
    ev.events=EPOLLIN|EPOLLET;
    //注册epoll事件
    epoll_ctl(epfd,EPOLL_CTL_ADD,listenfd,&ev);
    //设置serveraddr
    bzero(&serveraddr, sizeof(serveraddr));
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr("127.0.0.1");//此处设为服务器的ip
    serveraddr.sin_port=htons(8023);
    bind(listenfd,(sockaddr *)&serveraddr, sizeof(serveraddr));
    listen(listenfd, LISTENQ);
    clilen=sizeof(clientaddr);
    maxi = 0;

    /* 定义一个10线程的线程池 */
    boost::asio::thread_pool tp(10);

    while(1){
        cout<<"--------------------------"<<endl;
        cout<<"epoll_wait阻塞中"<<endl;
        //等待epoll事件的发生
        nfds=epoll_wait(epfd,events,10000,-1);//最后一个参数是timeout，0:立即返回，-1:一直阻塞直到有事件，x:等待x毫秒
        cout<<"epoll_wait返回，有事件发生"<<endl;
        //处理所发生的所有事件
        for(i=0;i<nfds;++i)
        {
            //有新客户端连接服务器
            if(events[i].data.fd==listenfd)
            {
                connfd = accept(listenfd,(sockaddr *)&clientaddr, &clilen);
                if(connfd<0){
                     perror("connfd<0");
                     exit(1);
                }
                else{
                    cout<<"用户"<<inet_ntoa(clientaddr.sin_addr)<<"正在连接\n";
                }
                //设置用于读操作的文件描述符
                ev.data.fd=connfd;
                //设置用于注册的读操作事件，采用ET边缘触发，为防止多个线程处理同一socket而使用EPOLLONESHOT
                ev.events=EPOLLIN|EPOLLET|EPOLLONESHOT;
                //边缘触发要将套接字设为非阻塞
                setnonblocking(connfd);
                //注册ev
                epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev);
            }
            //接收到读事件
            else if(events[i].events&EPOLLIN)
            {
                sockfd = events[i].data.fd;
                events[i].data.fd=-1;
                cout<<"接收到读事件"<<endl;

                string recv_str;
                boost::asio::post(boost::bind(RecvMsg,epfd,sockfd)); //加入任务队列，处理事件
            }
        }
    }
    close(listenfd);
}
```

RecvMsg 函数也要大改，我们在该函数里面调用 IO 函数读取数据，直到返回错误码 EAGAIN 或者 EWOULDBLOCK 为止，说明数据读取完毕，然后再调用 HandleRequest 来处理业务逻辑，代码如下：

```cpp
void server::RecvMsg(int epollfd,int conn){
    tuple<bool,string,string,int,int> info;//元组类型，四个成员分别为if_login、login_name、target_name、target_conn
    /*
        bool if_login;//记录当前服务对象是否成功登录
        string login_name;//记录当前服务对象的名字
        string target_name;//记录目标对象的名字
        int target_conn;//目标对象的套接字描述符
        int group_num;//记录所处群号
    */
    get<0>(info)=false;
    get<3>(info)=-1;

    string recv_str;
    while(1){
        char buf[10];
        memset(buf, 0, sizeof(buf));
        int ret  = recv(conn, buf, sizeof(buf), 0);
        if(ret < 0){
            cout<<"recv返回值小于0"<<endl;
            //对于非阻塞IO，下面的事件成立标识数据已经全部读取完毕
            if((errno == EAGAIN) || (errno == EWOULDBLOCK)){
                    printf("数据读取完毕\n");
                cout<<"接收到的完整内容为："<<recv_str<<endl;
                cout<<"开始处理事件"<<endl;
                break;
            }
            cout<<"errno:"<<errno<<endl;
            return;
        }
        else if(ret == 0){
            cout<<"recv返回值为0"<<endl;
            return;
        }
        else{
            printf("接收到内容如下: %s\n",buf);
            string tmp(buf);
            recv_str+=tmp;
        }
    }
    string str=recv_str;
    HandleRequest(epollfd,conn,str,info);
}
```

HandleRequest 方法变化不大，主要是注意要在末尾用 epoll_ctl 来重新注册文件描述符：

```cpp
void server::HandleRequest(int epollfd,int conn,string str,tuple<bool,string,string,int,int> &info){
    char buffer[1000];
    string name,pass;
    //把参数提出来，方便操作
    bool if_login=get<0>(info);//记录当前服务对象是否成功登录
    string login_name=get<1>(info);//记录当前服务对象的名字
    string target_name=get<2>(info);//记录目标对象的名字
    int target_conn=get<3>(info);//目标对象的套接字描述符
    int group_num=get<4>(info);//记录所处群号

    //连接MYSQL数据库
    MYSQL *con=mysql_init(NULL);
    mysql_real_connect(con,"127.0.0.1","root","","ChatProject",0,NULL,CLIENT_MULTI_STATEMENTS);

    //连接redis数据库
    redisContext *redis_target = redisConnect("127.0.0.1",6379);
    if(redis_target->err){
        redisFree(redis_target);
        cout<<"连接redis失败"<<endl;
    }

    //先接收cookie看看redis是否保存该用户的登录状态
    if(str.find("cookie:")!=str.npos){
        cout<<"cookie方法\n";
        string cookie=str.substr(7);
        // 查询该cookie是否存在：hget cookie name
        string redis_str="hget "+cookie+" name";
        redisReply *r = (redisReply*)redisCommand(redis_target,redis_str.c_str());
        string send_res;
        //存在
        if(r->str){
            cout<<"查询redis结果："<<r->str<<endl;
            send_res=r->str;
        }
        //不存在
        else
            send_res="NULL";
        send(conn,send_res.c_str(),send_res.length()+1,0);
    }

    //注册
    else if(str.find("name:")!=str.npos){
        cout<<"注册方法\n";
        int p1=str.find("name:"),p2=str.find("pass:");
        name=str.substr(p1+5,p2-5);
        pass=str.substr(p2+5,str.length()-p2-4);
        string search="INSERT INTO USER VALUES (\"";
        search+=name;
        search+="\",\"";
        search+=pass;
        search+="\");";
        cout<<"sql语句:"<<search<<endl<<endl;
        mysql_query(con,search.c_str());
    }

    //登录
    else if(str.find("login")!=str.npos){
        cout<<"登录方法\n";
        int p1=str.find("login"),p2=str.find("pass:");
        name=str.substr(p1+5,p2-5);
        pass=str.substr(p2+5,str.length()-p2-4);
        string search="SELECT * FROM USER WHERE NAME=\"";
        search+=name;
        search+="\";";
        cout<<"sql语句:"<<search<<endl;
        auto search_res=mysql_query(con,search.c_str());
        auto result=mysql_store_result(con);
        int col=mysql_num_fields(result);//获取列数
        int row=mysql_num_rows(result);//获取行数
        //查询到用户名
        if(search_res==0&&row!=0){
            cout<<"查询成功\n";
            auto info=mysql_fetch_row(result);//获取一行的信息
            cout<<"查询到用户名:"<<info[0]<<" 密码:"<<info[1]<<endl;
            //密码正确
            if(info[1]==pass){
                cout<<"登录密码正确\n\n";
                string str1="ok";
                if_login=true;
                login_name=name;
                pthread_mutex_lock(&name_sock_mutx); //上锁
                name_sock_map[login_name]=conn;//记录下名字和文件描述符的对应关系
                pthread_mutex_unlock(&name_sock_mutx); //解锁

                // 随机生成sessionid并发送到客户端
                srand(time(NULL));//初始化随机数种子
                for(int i=0;i<10;i++){
                    int type=rand()%3;//type为0代表数字，为1代表小写字母，为2代表大写字母
                    if(type==0)
                        str1+='0'+rand()%9;
                    else if(type==1)
                        str1+='a'+rand()%26;
                    else if(type==2)
                        str1+='A'+rand()%26;
                }
                //将sessionid存入redis
                string redis_str="hset "+str1.substr(2)+" name "+login_name;
                redisReply *r = (redisReply*)redisCommand(redis_target,redis_str.c_str());
                //设置生存时间,默认300秒
                redis_str="expire "+str1.substr(2)+" 300";
                r=(redisReply*)redisCommand(redis_target,redis_str.c_str());
                cout<<"随机生成的sessionid为："<<str1.substr(2)<<endl;

                send(conn,str1.c_str(),str1.length()+1,0);
            }
            //密码错误
            else{
                cout<<"登录密码错误\n\n";
                char str1[100]="wrong";
                send(conn,str1,strlen(str1),0);
            }
        }
        //没找到用户名
        else{
            cout<<"查询失败\n\n";
            char str1[100]="wrong";
            send(conn,str1,strlen(str1),0);
        }
    }

    //设定目标的文件描述符
    else if(str.find("target:")!=str.npos){
        cout<<"设定目标方法\n";
        int pos1=str.find("from");
        string target=str.substr(7,pos1-7),from=str.substr(pos1+5);
        target_name=target;
        if(name_sock_map.find(target)==name_sock_map.end()){
            cout<<"源用户为"<<login_name<<",目标用户"<<target_name<<"仍未登录，无法发起私聊\n";
        }
        else{
            pthread_mutex_lock(&from_mutex);
            from_to_map[from]=target;
            cout<<"from:"<<from<<"  target:"<<target<<endl;
            pthread_mutex_unlock(&from_mutex);
            login_name=from;
            cout<<"源用户"<<login_name<<"向目标用户"<<target_name<<"发起的私聊即将建立";
            cout<<",目标用户的套接字描述符为"<<name_sock_map[target]<<endl;
            target_conn=name_sock_map[target];
        }
    }

    //接收到消息，转发
    else if(str.find("content:")!=str.npos){
        target_conn=-1;
        cout<<"转发方法\n";
         //根据两个map找出当前用户和目标用户
        for(auto i:name_sock_map){
            if(i.second==conn){
                login_name=i.first;
                target_name=from_to_map[i.first];
                target_conn=name_sock_map[target_name];
                break;
            }
        }
        if(target_conn==-1){
            cout<<"找不到目标用户"<<target_name<<"的套接字，将尝试重新寻找目标用户的套接字\n";
            if(name_sock_map.find(target_name)!=name_sock_map.end()){
                target_conn=name_sock_map[target_name];
                cout<<"重新查找目标用户套接字成功\n";
            }
            else{
                cout<<"查找仍然失败，转发失败！\n";
            }
        }
        string recv_str(str);
        string send_str=recv_str.substr(8);
        cout<<"用户"<<login_name<<"向"<<target_name<<"发送:"<<send_str<<endl;
        send_str="["+login_name+"]:"+send_str;
        send(target_conn,send_str.c_str(),send_str.length(),0);
    }

     //绑定群聊号
    else if(str.find("group:")!=str.npos){
        cout<<"绑定群聊号方法\n";
        string recv_str(str);
        string num_str=recv_str.substr(6);
        group_num=stoi(num_str);
        //找出当前用户
        for(auto i:name_sock_map)
            if(i.second==conn){
                login_name=i.first;
                break;
            }
        cout<<"用户"<<login_name<<"绑定群聊号为："<<num_str<<endl;
        pthread_mutex_lock(&group_mutx);//上锁
        group_map[group_num].insert(conn);
        pthread_mutex_unlock(&group_mutx);//解锁
    }

    //广播群聊信息
    else if(str.find("gr_message:")!=str.npos){
        cout<<"广播群聊信息方法\n";
         //找出当前用户
        for(auto i:name_sock_map)
            if(i.second==conn){
                login_name=i.first;
                break;
            }
        //找出群号
        for(auto i:group_map)
            if(i.second.find(conn)!=i.second.end()){
                group_num=i.first;
                break;
            }
        string send_str(str);
        send_str=send_str.substr(11);
        send_str="["+login_name+"]:"+send_str;
        cout<<"群聊信息："<<send_str<<endl;
        for(auto i:group_map[group_num]){
            if(i!=conn)
                send(i,send_str.c_str(),send_str.length(),0);
        }
    }

    //线程工作完毕后重新注册事件
    epoll_event event;
    event.data.fd=conn;
    event.events=EPOLLIN|EPOLLET|EPOLLONESHOT;
    epoll_ctl(epollfd,EPOLL_CTL_MOD,conn,&event);

    mysql_close(con);
    if(!redis_target->err)
        redisFree(redis_target);

    //更新实参
    get<0>(info)=if_login;//记录当前服务对象是否成功登录
    get<1>(info)=login_name;//记录当前服务对象的名字
    get<2>(info)=target_name;//记录目标对象的名字
    get<3>(info)=target_conn;//目标对象的套接字描述符
    get<4>(info)=group_num;//记录所处群号
}
```

最后进行测试，先 make 然后启动服务器，如下：

![图片描述](https://doc.shiyanlou.com/courses/3573/600404/9584ea33eb65d498780f9dffb37d32a1-0)

![图片描述](https://doc.shiyanlou.com/courses/3573/600404/73445c0e095b1e1a8631313efde45ad4-0)

![图片描述](https://doc.shiyanlou.com/courses/3573/600404/c546df485f2cdfed72a9e1660ebbe13b-0)

删除本地 coookie.txt 文件，再开一个终端启动另一个客户端：

![图片描述](https://doc.shiyanlou.com/courses/3573/600404/7be32c22392983f0def901cc9fa32cba-0)

![图片描述](https://doc.shiyanlou.com/courses/3573/600404/c7e234892f9c8d2d339521879f5032b8-0)

### 实验总结

我们这次实验将服务器升级成 epoll+线程池的方式，提高了并发能力。

可以通过如下命令下载本次实验的代码：

```bash
wget https://labfile.oss.aliyuncs.com/courses/3573/code12.zip
```
# Linux聊天室项目

### 项目介绍

使用c++编程语言，使用面向对象封装，实现网络聊天室。

网络聊天室具有注册，登录，单聊，群聊，cookie记住登陆状态等功能。

### 项目来源

学习自蓝桥云课（原实验楼）的一个实战课项目。地址：`https://www.lanqiao.cn/courses/3573`

## QA

1. 项目图片看不了

把项目克隆到本地，用其他markdown软件打开就能看到图片。图片是直接引用了蓝桥云课的图片（属于外链）。没有自己复制。因为太多了（懒）。

### 项目文件介绍

```
//公共的头文件
global.h
//聊天室客户端的类
client.h
client.cpp
//聊天室服务端的类
server.h
server.cpp
//实例化客户端，生成客户端程序
test_client.cpp
//实例化服务端，生成服务端程序
test_server.cpp
//代码编译
makefile
```

### 不重要的文件

```
//测试redis时使用
test_redis.cpp
//程序运行中自动生成的文件
cookie.txt
```

### 编译

编译前安装的库

```
# 安装mysql(可选)
sudo apt-get install mysql-server 
sudo apt-get install mysql-client 
sudo apt-get install libmysqlclient-dev

# 安装mariadb，mysql的替代品
sudo apt-get install mariadb-server 
sudo apt-get install mariadb-client
sudo apt-get install libmariadbclient-dev

#安装redis
sudo apt-get install redis-server

# 安装hiredis，c语言连接库
sudo apt search hiredis
```

安装完数据库后需要创建表，详细请看教程

编译

```
make
```

服务端运行

```
./test_server
```

客户端运行

```
./test_client
```


# Redis 的介绍与使用

### 实验介绍

在后面的实验中，我们希望能够做出一个记录登录状态的小模块，其中一个重要的工具就是 Redis，因此我们这次实验先学习 Redis 的相关知识。

#### 知识点

- Redis 的特性
- Linux 下使用 Redis
- Redis 的基本命令
- C++连接 Redis

### Redis 介绍

Redis 是一种广泛使用的非关系型的内存数据库。

#### 特点

Redis 的存储结构是 key-value 的键值对形式，其中 key 是字符串类型，而 value 可以是五种数据类型（字符串、列表、无序集合、有序集合、字典）。

Redis 基于内存运行，读写速度极快，而且可以将数据持久化到磁盘中，同时支持过期键等特性，常用在缓存等场景下。

#### 五种数据类型

1. 字符串：字符串是 Redis 最基本的数据类型，能存储任何形式的字符串。
2. 列表：列表的底层是一个双向链表，因此支持在头部和尾部增加数据。
3. 字典：字典其实就是哈希表，适合用来存储对象。
4. 无序集合：集合中的数据不能重复。
5. 有序集合：同样是集合，但数据有序。

### Redis 的启动

ubuntu安装redis

```
sudo apt-get install redis-server
```

实验的容器环境已经安装好了 redis，我们直接启动即可，使用如下命令启动服务器：

```bash
cd /usr/bin
./redis-server
```

![图片描述](https://doc.shiyanlou.com/courses/3573/600404/d533b6fc7583886636ebe4109a86135b-0)

另开一个终端，使用如下命令启动客户端：

```bash
cd /usr/bin
./redis-cli
```

并输入 ping，如果输出了 pong，说明连接成功。

![图片描述](https://doc.shiyanlou.com/courses/3573/600404/4270f1f404536c6823e956bdeea0ca21-0)

### 基本命令

#### KEYS 命令

我们可以使用该命令找到一个 key（键）。

```bash
KEYS *     #使用该命令列出所有key
KEYS a*    #列出所有以a开头的key
KEYS a?b   #列出所有axb格式（x为任意单个字符）的key
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/5ce8c7ced4ac04815f348327e2dfd7c3-0)

#### EXISTS 命令

我们可以使用该命令来判断某个 key 是否存在，若存在会输出 1，否则输出 0。

```bash
EXISTS a    #判断key a是否存在
```

#### DEL 命令

使用命令 DEL xxx 来删除 key 为 xxx 的数据。

```bash
DEL a   #删除key a及其value
```

#### 字典的使用

接下来我们学习字典的使用，因为我们记录登录状态需要用到这种数据类型。前面说到了字典适合存储对象，我们可以将字典作为 value，然后字典这个 value 中又可以存储若干组 key 和 value，相当于存储了一个对象的若干个属性。

当我们 value 用字典类型时，可以用如下语句插入或更新一条数据：

```bash
HSET 123321 name xiaoming   # key为123321 ， value为 （name，xiaoming）的键值对
```

然后使用如下语句来查询某个 key 对应的字典中某个字段的值：

```bash
HGET 123321 name    # 查询key为123321的字典中字段name的具体值
```

当我们想要查看某个 key 对应的整个字典长啥样，可以用下面语句：

```bash
HGETALL 123321      # 查看key为123321的字典中的字段和数据
```

如果我们想要给某个对象设置多个属性，可以像下面一样操作：

```bash
HSET 123321 name xiaoming age 18 sex male   # 设置key为123321的字典中的字段name为xiaoming，age为18，sex为male
```

![图片描述](https://doc.shiyanlou.com/courses/3573/1116908/0c1540087fad180b4e6ad8c80d261d45-0)

### C++连接 Redis

有了 Redis 我们当然也要用 C++ 去连接它，这里我们借助 hiredis 这个 api 库。

#### Linux 下安装 hiredis

ubuntu安装hiredis

```
# 搜索关于hiredis的包
sudo apt search hiredis
# 安装hiredis
sudo apt install libhiredis-dev
```

在命令行输入以下命令安装 hiredis。

```bash
cd /home/shiyanlou
wget https://github.com/redis/hiredis/archive/v0.14.0.tar.gz
tar -xzf v0.14.0.tar.gz
cd hiredis-0.14.0/
make
sudo make install
```

安装完成后我们需要将动态库文件移到 `/usr/lib` 目录下，如下：

```bash
sudo cp /usr/local/lib/libhiredis.so.0.14 /usr/lib/
```

然后在 `/home/project` 目录下新建一个 `test_redis.cpp` 文件，写入下面的测试代码：

```cpp
#include <iostream>
#include <hiredis/hiredis.h>
using namespace std;
int main(){
    redisContext *c = redisConnect("127.0.0.1",6379);
    if(c->err){
       redisFree(c);
       cout<<"连接失败"<<endl;
       return 1;
    }
    cout<<"连接成功"<<endl;
    redisReply *r = (redisReply*)redisCommand(c,"PING");
    cout<<r->str<<endl;
    return 0;
}
```

然后用下面命令进行编译，其中-lhiredis 是为了动态链接 libhiredis.so.0.14 文件：

```bash
g++ -o test_redis test_redis.cpp -lhiredis
```

然后运行 test_redis，可以看到输出如下表示连接成功：

![图片描述](https://doc.shiyanlou.com/courses/3573/600404/72a2e434c7ea73be22885071540c0cf4-0)

### 实验总结

这次实验我们学习了 Redis 的基本使用以及如何使用 C++连接 Redis，后面可以开发记录登录状态的功能了。

可以通过如下命令下载本次实验的代码：

```bash
wget https://labfile.oss.aliyuncs.com/courses/3573/code9.zip
```


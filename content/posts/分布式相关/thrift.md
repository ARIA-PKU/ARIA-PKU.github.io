---
title: Thrift
date: 2022-05-26T21:48:34+08:00
lastmod: 2022-05-26T21:48:34+08:00
cover: https://aria9766.oss-cn-beijing.aliyuncs.com/blog-pictures/6_15/maple.jpg
# images:
#   - /img/cover.jpg
categories:
  - 分布式
tags:
  - RPC
  - c++多线程编程
  - thrift
# nolastmod: true
draft: false
---

实现一个多线程的匹配系统

<!--more-->

# 项目结构

```
.
|-- game   # 匹配请求的客户端
|-- match_server  # 匹配请求的服务端
`-- thrift  # 接口函数存放位置
```

# thrift简单语法介绍
使用Thrift开发程序，首先要做的事情就是对接口进行描述， 然后再使用Thrift将接口的描述文件编译成对应语言的版本

## 1.命名空间
thrift文件命名一般都是以.thrift作为后缀：XXX.thrift，可以在该文件的开头为该文件加上命名空间限制，格式为：

`namespace 语言名称 名称`

例如对c++来说，有：

`namespace cpp match_service`

## 2.数据类型
大小写敏感，它共支持以下几种基本的数据类型：

```
string， 字符串类型，注意是全部小写形式；
i16, 16位整形类型，
i32，32位整形类型，对应C/C++/java中的int类型；
i64，64位整形，对应C/C++/java中的long类型；
byte，8位的字符类型，对应C/C++中的char，java中的byte类型
bool, 布尔类型，对应C/C++中的bool，java中的boolean类型；
double，双精度浮点类型，对应C/C++/java中的double类型；
void，空类型，对应C/C++/java中的void类型；该类型主要用作函数的返回值，
除上述基本类型外，ID还支持以下类型：

map，map类型，例如，定义一个map对象：map[HTML_REMOVED] newmap;
set，集合类型，例如，定义set[HTML_REMOVED]对象：set[HTML_REMOVED] aSet;
list，链表类型，例如，定义一个list[HTML_REMOVED]对象：list[HTML_REMOVED] aList;
struct，自定义结构体类型，在IDL中可以自己定义结构体，对应C中的struct，c++中的struct和class，java中的class。例如：
struct User{
      1: i32 id,
      2: string name,
      3: i32 score
}
```


注意，在struct定义结构体时需要对每个结构体成员用序号标识：“序号: ”。

## 3.函数接口
文件中对所有接口函数的描述都放在service中，service的名字可以自己指定，该名字也将被用作生成的特定语言接口文件的名字。

接口函数需要对参数使用序号标号，除最后一个接口函数外，要以`,`结束对函数的描述。

如：

```
namespace cpp match_service

struct User {  // 定义结构体
    1: i32 id,
    2: string name,
    3: i32 score
}

service Match {  // 定义函数，与c++类似，但要加上序号

    i32 add_user(1: User user, 2: string info),  // 开发小技巧，加一个info作为额外信息，方便拓展
    
    i32 remove_user(1: User user, 2: string info),

}

```

# 服务端的建立
使用rpc的好处之一就是可以直接根据接口文件，生成通信的服务端与客户端代码，自己只需要实现具体的逻辑。

对于匹配系统的thrift相关配置，我们在thrift文件夹下，创建match.thrift文件。

打开thrift官网，在上方选择Tutorial项，查看thrift官方教程。

点击下方的tutorial.thrift进入一个示例文件(这个不挂vpn有时候会登录不上)，

改写thrift配置文件，只需要在文件中写明接口和对象.然后执行命令。

`thrift -r --gen <语言名> <.thrift文件的路径>`

就会生成各种配置和连接文件，还有代码框架，只需要在框架中实现自己的业务即可。

**步骤**
1.在thrift文件夹下，编辑match.thrift文件，用来生成匹配系统服务端的一系列文件match.thrift 文件内容如下：

```
namespace cpp match_service  // c++命名空间

struct User {
    1: i32 id,
    2: string name,
    3: i32 score
}

service Match {

    /**
     * user: 添加的用户信息
     * info: 附加信息
     * 在匹配池中添加一名用户
     */
    i32 add_user(1: User user, 2: string info),
    
    /**
     * user: 删除的用户信息
     * info: 附加信息
     * 从匹配池中删除一名用户
     */
    i32 remove_user(1: User user, 2: string info),

}
```

2.进入到match_system文件夹，创建src文件夹，存放一下生成的文件。在src下执行语句（这里的cpp是指明服务端语言）：
`thrift -r --gen cpp ../../thrift/match.thrift`
这样，thrift服务端的一系列文件就会生成在src文件夹中的gen-cpp文件夹下，为了划分业务模块将gen-cpp重命名为match_server

文件结构如下：

```
.
-- match_server
    |-- Match.cpp  # 服务端与客户端通信相关代码
    |-- Match.h
    |-- Match_server.skeleton.cpp  # 用cpp实现端口的样例
    |-- match_types.cpp  # 存放在thrift中定义的struct类型相关代码
    `-- match_types.h
```


其中Match_server.skeleton.cpp: 服务端的代码框架，具体业务就是在这个文件夹下编写实现

将Match_server.skeleton.cpp移动到match_system/src下并重命名为main.cpp，match_system的整个业务逻辑就是在这个文件中实现。

## c++的编译链接

开发第一步，先跑通样例，给样例函数加上返回值，修改引用，得到如下代码，然后开始编译链接。

> -E：只对文件进行预处理，不进行编译和汇编。g++ -E main.cpp——>在dos命令行查看某文件的预处理过程，如果你想查看详细的预处理，可以重定向到一个文件中，如：g++ -E main.cpp -o main.i
>
> -s：编译到汇编语言，不进行汇编和链接,即只激活预处理和编译，生成汇编语言,如果你想查看详细的编译，可以重定向到一个文件中，如：g++ -S main.cpp -o main.s
>
> -c:编译到目标代码,g++ -c main.s -o 文件名.o
>
> -o:生成链接文件: 如果该文件是独立的，与其他自己编写的文件无依赖关系。直接g++ main.o -o 生成的可执行文件的文件名，

c++编译只需要编译`.cpp`文件，`.h`为头文件会自动引入；

`g++ -c *.cpp`，(这里要找到所有的cpp文件，否则链接的时候会出现问题)就能获得对应的`-o`编译好的目标文件（如果编译出现问题，再次编译只需要编译那些失败的文件即可）；

然后开始链接，这里需要用到thrift中的动态链接库，`g++ *.o -o main -lthrift`;

然后生成可执行文件main，就可以直接运行了，为了便于查看结果可以加一些输入输出调试。

```
// This autogenerated skeleton file illustrates how to build a server.
// You should copy it to another filename to avoid overwriting it.

#include "match_server/Match.h"                                                                                                                                                                            
#include <thrift/protocol/TBinaryProtocol.h>
#include <thrift/server/TSimpleServer.h>
#include <thrift/transport/TServerSocket.h>
#include <thrift/transport/TBufferTransports.h>
#include <iostream>

using namespace ::apache::thrift;
using namespace ::apache::thrift::protocol;
using namespace ::apache::thrift::transport;
using namespace ::apache::thrift::server;

using namespace  ::match_service;

class MatchHandler : virtual public MatchIf {
    public:
        MatchHandler() {
            // Your initialization goes here
        }

        int32_t add_user(const User& user, const std::string& info) {
            // Your implementation goes here
            printf("add_user\n");
            return 0;
        }

        int32_t remove_user(const User& user, const std::string& info) {
            // Your implementation goes here
            printf("remove_user\n");
            return 0;
        }

};

int main(int argc, char **argv) {
    int port = 9090;
    ::std::shared_ptr<MatchHandler> handler(new MatchHandler());
    ::std::shared_ptr<TProcessor> processor(new MatchProcessor(handler));
    ::std::shared_ptr<TServerTransport> serverTransport(new TServerSocket(port));
    ::std::shared_ptr<TTransportFactory> transportFactory(new TBufferedTransportFactory());
    ::std::shared_ptr<TProtocolFactory> protocolFactory(new TBinaryProtocolFactory());
	::std::cout << "start match" << std::endl;
    TSimpleServer server(processor, serverTransport, transportFactory, protocolFactory);
    server.serve();
    return 0;
}
```

# 客户端的建立

##实现不同语言通信

目标是实现服务端是c++语言实现，客户端通过python调用，因此需要再生成python的客户端。

继续在`src`文件夹下生成`thrift -r --gen py ../../thrift/match.thrift`，生成的结构目录如下：

```
.
|-- __init__.py
`-- match
    |-- Match-remote  # 服务端执行文件
    |-- Match.py  # 函数文件
    |-- __init__.py
    |-- constants.py
    `-- ttypes.py  # struct结构体文件
```

在thrift官网上找到对应的教程初步修改一下：

```
# 下面四行是将文件添加到环境变量中，这里不需要使用
# import sys
# import glob
# sys.path.append('gen-py')
# sys.path.insert(0, glob.glob('../../lib/py/build/lib*')[0])

# 下面是从生成的文件中引入类和结构体
from match_client.match import Match
from match_client.match.ttypes import User

from thrift import Thrift
from thrift.transport import TSocket
from thrift.transport import TTransport
from thrift.protocol import TBinaryProtocol


def main():
    # Make socket
    transport = TSocket.TSocket('localhost', 9090)

    # Buffering is critical. Raw sockets are very slow
    transport = TTransport.TBufferedTransport(transport)

    # Wrap in a protocol
    protocol = TBinaryProtocol.TBinaryProtocol(transport)

    # Create a client to use the protocol encoder
    client = Match.Client(protocol)  # 对应client

    # Connect!
    transport.open()

	user = User(1, 'aria', 1200)
	client.add_user(user, "")
	
    # Close!
    transport.close()
    
if __name__ == "__main__":
    main()
```

然后运行就能发现，实现了向服务端发送请求，可以通信了，第一步已经跑通接口了。

#服务端匹配系统设计

服务端需要接收请求并且实现匹配，再将匹配结果返回用户。

首先，在接收请求的部分，多个玩家需要同时发送请求，因此这里需要多线程处理，考虑到匹配需要一定的时间，可以先将请求读入到消息队列中（这里是需要消息队列线程安全的），再单线程地依次将消息队列中的内容读入内存；

其次，在实现匹配的过程中，匹配过程中匹配池内的用户不能增减，这里开一个额外的线程处理。

可以抽象为经典的生产者-消费者模型。

## c++中的多线程、信号量和条件变量

为了实现一个线程安全的消息队列，需要用到mutex信号量机制，c++中还提供了条件变量的机制，条件变量实际是对锁的封装，使用它能更方便实现消息队列。

这里关于`unique_lock`的使用可以参考了一些文章，总结一下就是作用域之外会自动解锁，也可以在作用域内通过unlock手动提前解锁，比较灵活。

编译好后，这里再连接需要用到多线程相关的动态链接库：`g++ *.o -o main -lthrift -pthread`

```
#include "match_server/Match.h"   // 自己的头文件用""，系统头文件用<>
#include <thrift/protocol/TBinaryProtocol.h>
#include <thrift/server/TSimpleServer.h>
#include <thrift/transport/TServerSocket.h>
#include <thrift/transport/TBufferTransports.h>
#include <iostream>
#include <mutex>  //锁的头文件
#include <thread>  //线程的头文件
#include <condition_variable>  //条件变量的头文件
#include <queue>
#include <vector>

using namespace ::apache::thrift;
using namespace ::apache::thrift::protocol;
using namespace ::apache::thrift::transport;
using namespace ::apache::thrift::server;
using namespace std;
using namespace ::match_service;

struct Task {
    User user;
    string type;
};

struct MessageQueue {
    //队列是互斥的，同时只能有一个线程访问队列
    queue<Task> q;
    mutex m;  // 互斥变量
    condition_variable cv;  // 条件变量
} message_queue;

class Pool {
public:
    void add(User user) {
        users.push_back(user);
    }

    void remove(User user) {
        for (uint32_t i = 0; i < users.size(); i++) {
            if (users[i].id == user.id) {
                users.erase(users.begin() + i);
                break;
            }
        }
    }

    void match() {
        while (users.size() > 1) {
            auto player1 = users[0];
            auto player2 = users[1];
            users.erase(users.begin());
            users.erase(users.begin());

            save_result(player1.id, player2.id);
        }
    }

    void save_result(int a, int b) {
        printf(" %d 和 %d 匹配成功\n", a, b);
    }

private:
    vector<User> users;
} pool;


class MatchHandler : virtual public MatchIf {
public:
    MatchHandler() {
        // 在这里初始化
    }

    int32_t add_user(const User &user, const std::string &info) {
        // 在这里实现接口
        printf("add_user\n");


        unique_lock<mutex> lck(message_queue.m);  // 加锁，不需要显式地去解锁，会在离开作用域后自动解锁
        message_queue.q.push({user, "add"});
        // 当有操作时，应该唤醒随机一个线程，当前只有一个需要唤醒可以直接用notify_one()
        message_queue.cv.notify_all();


        return 0;
    }

    int32_t remove_user(const User &user, const std::string &info) {
        // 在这里实现接口
        printf("remove_user\n");


        unique_lock<mutex> lck(message_queue.m);//加锁,在队列为空的时候，不能拿到锁
        message_queue.q.push({user, "remove"});
        //当有操作时，应该唤醒线程
        message_queue.cv.notify_all();

        return 0;
    }

};


//线程操作的函数
void consume_task() {
    while (true) {
        unique_lock<mutex> lck(message_queue.m);//加锁
        if (message_queue.q.empty()) {
            // 因为队列初始一定是空的，如果直接continue，会死循环，占满cpu。因此在初始时，应在有add操作后，才应该执行这里
            // 所以在这里添加条件变量，将线程阻塞在这里，只有队列中有元素变化才会唤醒该线程
            message_queue.cv.wait(lck);
        } else {
            auto task = message_queue.q.front();
            message_queue.q.pop();
            // 及时解锁，因为消息队列是多线程操作的，这里解锁才可以继续向消息队列中添加或删除task，
            // 否则等待执行完成自动释放则占用时间过长
            lck.unlock();
            //具体任务
            if (task.type == "add") {
                pool.add(task.user);
            } else if (task.type == "remove") {
                pool.remove(task.user);
            }
            pool.match();

        }
    }
}

int main(int argc, char **argv) {
    int port = 9090;
    ::std::shared_ptr<MatchHandler> handler(new MatchHandler());
    ::std::shared_ptr<TProcessor> processor(new MatchProcessor(handler));
    ::std::shared_ptr<TServerTransport> serverTransport(new TServerSocket(port));
    ::std::shared_ptr<TTransportFactory> transportFactory(new TBufferedTransportFactory());
    ::std::shared_ptr<TProtocolFactory> protocolFactory(new TBinaryProtocolFactory());
    TSimpleServer server(processor, serverTransport, transportFactory, protocolFactory);
    printf("Match server start\n");

    thread matching_thread(consume_task);  //这里如果不开多线程，则在这一步死循环的时候，下面server无法执行


    server.serve();
    return 0;
}

```

## 服务端中结果回传

因为匹配需要时间，所以匹配生成的结果是无法同步直接传递回客户端的。此时需要再在之前的客户端创建服务端，在之前的服务端创建客户端，将结果返回。

这里实现一下，如何将新的客户端的内容写入已经写好的服务端中去。先写好接口函数，根据接口函数生成客户端（这里客户端和服务端用的基本是同一套内容，只是调用的方式不同）`thrift -r --gen cpp ../../thrift/save.thrift`：

```
namespace cpp save_service

service Save {

    /**
     * username: myserver的名称
     * password: myserver的密码的md5sum的前8位
     */
    i32 save_data(1: string username, 2: string password, 3: i32 player1_id, 4: i32 player2_id)
}
```

到官网的tutorial下面，找到教程，把对应的内容添加进来：

```
// 1、头文件，包括系统头文件和自己的头文件
#include <thrift/transport/TSocket.h>
#include <thrift/transport/TTransportUtils.h>
#include "save_client/Save.h"

// 2、引入对应的命名空间
using namespace ::save_service;

// 3、在对应的位置添加代码，也就是上面的pool内的save_results函数中
 	void save_result(int a, int b) {
        std::shared_ptr<TTransport> socket(new TSocket("123.57.47.211", 9090));  # 这里IP改成服务端的IP地址
  		std::shared_ptr<TTransport> transport(new TBufferedTransport(socket));
  		std::shared_ptr<TProtocol> protocol(new TBinaryProtocol(transport));
  		SaveClient client(protocol);  // 这个是生成的client接口
  		try {
    		transport->open();
    		// 以下实现功能逻辑
    		client.save_data("acs_2340","5508c7f5",a,b);
    		
    		transport->close();
  		} catch (TException& tx) {
    		cout << "ERROR: " << tx.what() << endl;
  		}
    }
```

##thrift中的多线程

这里直接仿写官网给定`TThreadedServer `方式，多线程接收客户端的请求，在测试项目中其多线程作用在MatchHandler类下的add_user和remove_user函数上。

官网教程上还给了线程池之类的其他写法，基本类似，添加相应类即可。边编译边debug，缺什么补什么。

具体修改：

```
// 1 添加头文件
#include <thrift/concurrency/ThreadManager.h>
#include <thrift/concurrency/ThreadFactory.h>
#include <thrift/TToString.h>
#include <thrift/server/TThreadedServer.h>  

// 2 多线程需要添加工厂类
class MatchCloneFactory : virtual public MatchIfFactory {
    public:
        ~MatchCloneFactory() override = default;
        MatchIf* getHandler(const ::apache::thrift::TConnectionInfo& connInfo) override
        {
            std::shared_ptr<TSocket> sock = std::dynamic_pointer_cast<TSocket>(connInfo.transport);
            /*cout << "Incoming connection\n";
            cout << "\tSocketInfo: "  << sock->getSocketInfo() << "\n";
            cout << "\tPeerHost: "    << sock->getPeerHost() << "\n";
            cout << "\tPeerAddress: " << sock->getPeerAddress() << "\n";
            cout << "\tPeerPort: "    << sock->getPeerPort() << "\n";*/
            return new MatchHandler;
        }
        void releaseHandler(MatchIf* handler) override {
            delete handler;
        }
};

// 3 修改server
	TThreadedServer server(
            std::make_shared<MatchProcessorFactory>(std::make_shared<MatchCloneFactory>()),
            std::make_shared<TServerSocket>(9090), //port
            std::make_shared<TBufferedTransportFactory>(),
            std::make_shared<TBinaryProtocolFactory>());

```

## 优化函数匹配

增加一个wt数组记录等待时间，匹配的范围会随着等待时间的扩大不断扩大。这里写的不太好，应该可以把等待时间放到user这个类里面，然后直接一遍排序再匹配，这样时间复杂度会从现在的$O(n^2)$降到$O(nlogn)$;

还有一个问题就是如果一直有用户进来，就会不停往消息队列增加信息，导致match函数饿死，这个问题还没想到好的解决方法，以后再解决这个问题。

```
class Pool
{
    public:
        void save_result(int a, int b)
        {
            printf("Match Result: %d %d\n", a, b);


            std::shared_ptr<TTransport> socket(new TSocket("123.57.47.211", 9090));
            std::shared_ptr<TTransport> transport(new TBufferedTransport(socket));
            std::shared_ptr<TProtocol> protocol(new TBinaryProtocol(transport));
            SaveClient client(protocol);

            try {
                transport->open();

                int res = client.save_data("acs_0", "6e822f5b", a, b);

                if (!res) puts("success");
                else puts("failed");

                transport->close();
            } catch (TException& tx) {
                cout << "ERROR: " << tx.what() << endl;
            }
        }

        bool check_match(uint32_t i, uint32_t j)
        {
            auto a = users[i], b = users[j];

            int dt = abs(a.score - b.score);
            int a_max_dif = wt[i] * 50;
            int b_max_dif = wt[j] * 50;

            return dt <= a_max_dif && dt <= b_max_dif;
        }

        void match()
        {
            for (uint32_t i = 0; i < wt.size(); i ++ )
                wt[i] ++ ;   // 等待秒数 + 1

            while (users.size() > 1)
            {
                bool flag = true;
                for (uint32_t i = 0; i < users.size(); i ++ )
                {
                    for (uint32_t j = i + 1; j < users.size(); j ++ )
                    {
                        if (check_match(i, j))
                        {
                            auto a = users[i], b = users[j];
                            users.erase(users.begin() + j);
                            users.erase(users.begin() + i);
                            wt.erase(wt.begin() + j);
                            wt.erase(wt.begin() + i);
                            save_result(a.id, b.id);
                            flag = false;
                            break;
                        }
                    }

                    if (!flag) break;
                }

                if (flag) break;
            }
        }

        void add(User user)
        {
            users.push_back(user);
            wt.push_back(0);
        }

        void remove(User user)
        {
            for (uint32_t i = 0; i < users.size(); i ++ )
                if (users[i].id == user.id)
                {
                    users.erase(users.begin() + i);
                    wt.erase(wt.begin() + i);
                    break;
                }
        }

    private:
        vector<User> users;
        vector<int> wt;  // 等待时间, 单位：s
}pool;
```





# 参考文章

**C++-std::unique_lock介绍和简单使用**：https://blog.csdn.net/mrbone11/article/details/121620329
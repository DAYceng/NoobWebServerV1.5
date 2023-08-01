
定时器处理非活动连接
===============
由于非活跃连接占用了连接资源，严重影响服务器的性能，通过实现一个服务器定时器，处理这种非活跃连接，释放连接资源。利用alarm函数周期性地触发SIGALRM信号,该信号的信号处理函数利用管道通知主循环执行定时器链表上的定时任务.
> * 统一事件源
> * 基于升序链表的定时器
> * 处理非活动连接

补充说明：定时器函数
​ 在事件循环中要处理事件，而处理时要使用定时器对每个连接fd进行管理

​ 定时器的作用是管理连接和超时处理，其接收一个与客户端连接的socket（即connfd）和客户端地址信息client_address作为参数，然后进行users数组的初始化操作

void WebServer::timer(int connfd, struct sockaddr_in client_address){
    users[connfd].init(connfd, client_address, m_root, m_CONNTrigmode, m_close_log, m_user, m_passWord, m_databaseName);
    ...
}
​ users是在头文件webser.h中声明的一个数组，该数组中的数据类型是http_conn*，也就是指向一个http_conn的指针，每个http_conn对象代表一个客户端连接。

​ http_conn类封装了处理客户端 HTTP 请求的功能和操作（在http_conn.h中声明）。它包含了处理请求报文、解析请求、生成响应等一系列与 HTTP 协议相关的操作。先不展开说明。

​ 现在我们为connfd初始化好了一个 http_conn 对象，这玩意就保存在 users 数组的对应索引下。使用传入的connfd作为索引从该数组中取一个http_conn对象（？↓），该对象就代表着新建立的连接中进行的一系列HTTP协议下的操作。

​ 然后，对该对象进行初始化（调用http_conn类的初始函数）

ps：看到这里的时候，我其实有一个问题：谁最早创建了http_conn对象并把它加到users数组中？是定时器↓

​ connfd是我们通过事件循环中epoll_wait函数监听捕获信号后创建的socket，其代表着有新的客户端连接到webserver，因此此时会通过dealclinetdata()去处理该socket，处理的方式就是调用定时器函数并初始化定时器。（因为要用定时器管理连接）

​ 在 timer() 函数中，通过 users 数组的索引 connfd 创建一个 http_conn 对象，该对象负责保存connfd与webserver交互过程中的所有操作。

梳理清楚之后继续

定时器中还创建一个新的util_timer对象，并为其设置相关属性。

将客户端的地址信息存储在 users_timer 数组（webser.h）中与 connfd 对应的位置。

将当前连接的文件描述符 connfd 存储在 users_timer 数组中与 connfd 对应的位置。

	...
	//初始化client_data数据
    //创建定时器，设置回调函数和超时时间，绑定用户数据，将定时器添加到链表中
    users_timer[connfd].address = client_address;
    users_timer[connfd].sockfd = connfd;
    util_timer *timer = new util_timer;
	...
​ users_timer 数组的作用是为每个连接存储相关的定时器信息和客户端地址信息。这样可以方便地通过文件描述符查找对应的定时器或者客户端地址信息，并进行相关的操作，例如处理定时事件或关闭连接。

继续

​ 前面我们创建了一个新的util_timer对象，timer，现在将users_timer[connfd]的地址赋值给timer的user_data成员变量，users_timer是一个数组，里面存储着每个连接的计时器信息。

​ 将回调函数cb_func（该函数在lst_timer.cpp中定义）赋值给timer的cb_func成员变量。这里的回调函数指的是在定时器到期时要执行的函数（详见：定时器链表的实现）。

	...
	timer->user_data = &users_timer[connfd];
    timer->cb_func = cb_func;
    time_t cur = time(NULL);//获取当前时间
    timer->expire = cur + 3 * TIMESLOT;
    users_timer[connfd].timer = timer;
    utils.m_timer_lst.add_timer(timer);
}
然后就是获取当前时间并根据TIMESLOT参数计算定时器的超时时间，将创建的定时器对象timer赋值给对应连接（connfd）的计时器信息（users_timer[connfd].timer）。

当上述一切操作处理完后，新建立的连接fd得到了它的定时器信息，但是我们要管理这些定时器对象，如何做？用链表呗

在定时器函数的最后一行代码中，将定时器对象添加到了定时器链表m_timer_lst中，该链表的定义位于lst_timer.cpp（详见：定时器链表的实现）

至此，定时器函数说明完毕。其创建一个了定时器对象，并为该对象设置相关参数，然后将定时器添加到定时器链表中，以便在事件循环中进行定时器的管理和触发。

补充说明：定时器链表的实现
​ 从上面的梳理也能看到，Web服务器中的定时器通常需要同时管理多个定时任务，为了提供高效的插入、删除和排序操作，节省内存空间，并具有灵活性和可扩展性，我们需要通过链表来实现定时器

概况
定时器链表与定时器函数的实现代码是分开的，定时器链表管理类sort_timer_lst是以一个类的形式在lst_timer.h中声明

class sort_timer_lst{
public:
    sort_timer_lst();
    ~sort_timer_lst();

    void add_timer(util_timer *timer);//用于向定时器链表中添加定时器
    void adjust_timer(util_timer *timer);//用于调整定时器的位置，当定时器的到期时间延后时需要调用该函数
    void del_timer(util_timer *timer);//用于从链表中删除定时器
    void tick();//用于处理到期的定时器

private:
    void add_timer(util_timer *timer, util_timer *lst_head);//辅助函数，用于插入定时器到指定节点之后

    util_timer *head;//分别指向链表的头部和尾部
    util_timer *tail;
};
严格来说，sort_timer_lst也不是真正"定时器链表"，这个类只是去使用了定时器链表，并提供一系列配套的成员函数用以管理定时器链表。

下面将分别介绍。

定时器链表的节点
在定时器函数timer（）中，我们创建的是定时器（util_timer）对象，这些就是我们需要管理的节点，即链表节点。

以下是"节点类"util_timer的定义，该类真正给出了"定时器链表"的定义，由此可知定时器链表被定义为一个双向链表

class util_timer
{
public:
    util_timer() : prev(NULL), next(NULL) {}

public:
    time_t expire;
    
    void (* cb_func)(client_data *);
    client_data* user_data;
    util_timer* prev;
    util_timer* next;
};
从util_timer提供的构造函数看，这个类充当的是一个双向链表的节点，其与普通的链表有有点不同，该节点中还保存了定时器超时时间expire以及客户端的相关数据user_data，并且还提供一个指针指向回调函数。

expire是一个时间变量，其数据类型为time_t

平时刷题时定义的ListNode通常有一个val用来存放节点值，这里的user_data就是节点值。client_data是一个结构体，其中存放了客户端的相关数据

struct client_data{
    sockaddr_in address;//存储客户端的地址信息
    int sockfd;//表示客户端的套接字描述符
    util_timer *timer;//关联客户端和定时器
};
因此，user_data变量保存了客户端连接的地址信息和fd，以及与客户端绑定的定时器（从定时器函数的代码中可知，该定时器由add_timer函数添加到client_data结构体中）

然后来看回调函数。

所谓回调函数（Callback Function）是指将某种可以作为参数传递给另一个函数的函数。这种函数可以作为参数传递给另一个函数，当特定的事件发生后，调用传入的"参数函数"进行特定的操作。（常用于异步编程、事件处理）

class util_timer{
...
public:
...    
    void (* cb_func)(client_data *);
...
};
在util_timer中，有一个用于指向回调函数cb_func的指针，该回调函数的定义位于lst_timer.cpp

cb_func用于处理定时器到期时的操作。回调函数的参数是一个指向client_data结构体的指针。

void cb_func(client_data *user_data){
    //使用epoll_ctl函数从 epoll 实例中删除文件描述符（socket）。
    epoll_ctl(Utils::u_epollfd, EPOLL_CTL_DEL, user_data->sockfd, 0);
    assert(user_data);//确保user_data指针不为空。
    close(user_data->sockfd);//关闭之前处理的文件描述符（socket），释放资源
    http_conn::m_user_count--;//将http_conn类中的静态成员变量m_user_count减少1
}
说白了就是，定时器链表中管理的某个连接fd的定时器超时后，该定时器对象自身会调用一个回调函数，通过回调函数释放当前连接的fd所占用的资源。

定时器链表管理类的成员函数
看完了定时器链表节点的定义，现在来看看负责实际管理链表的一些成员函数。

由这些功能函数构建的定时器容器为带头尾结点的升序双向链表。sort_timer_lst为每个连接创建一个定时器，将其添加到链表中，并按照超时时间升序排列。执行定时任务时，将到期的定时器从链表中删除。

class sort_timer_lst{
public:
    sort_timer_lst();
    ~sort_timer_lst();

    void add_timer(util_timer *timer);//用于向定时器链表中添加定时器
    void adjust_timer(util_timer *timer);//用于调整定时器的位置，当定时器的到期时间延后时需要调用该函数
    void del_timer(util_timer *timer);//用于从链表中删除定时器
    void tick();//用于处理到期的定时器

private:
    void add_timer(util_timer *timer, util_timer *lst_head);//辅助函数，用于插入定时器到指定节点之后
...
};
add_timer函数就是用于构造链表的函数，该函数将目标定时器添加到链表中，添加时按照升序添加。

如果链表为空，则直接将定时器作为首节点插入。如果定时器的到期时间小于链表头部定时器的到期时间，将定时器作为新的首节点插入。否则，调用辅助函数add_timer(timer, head)插入定时器。

void sort_timer_lst::add_timer(util_timer* timer){
    if (!timer) return;
    if (!head){//当前链表为空，头节点和尾节点是同一个
        head = tail = timer;
        return;
    }
    if (timer->expire < head->expire){//如果当前节点的超时时间小于头节点，令其为新的头节点
        timer->next = head;
        head->prev = timer;
        head = timer;
        return;
    }
    add_timer(timer, head);
}
adjust_timer函数用于调整定时器的位置，当定时器的到期时间延后时需要调用该函数。

首先找到定时器的下一个节点tmp，

如果tmp为空或者定时器的到期时间小于tmp的到期时间，说明定时器无需调整位置，直接返回。

如果待插入的定时器timer节点的过期时间比链表中其他节点的过期时间都要小，那么该节点要称为链表的头部节点，因此要修改头部指针并重新调用add_timer函数插入定时器。

否则，修改定时器的前后指针，并调用add_timer(timer, timer->next)插入定时器。

void sort_timer_lst::adjust_timer(util_timer* timer){
    if (!timer) return;
    util_timer* tmp = timer->next;
    if (!tmp || (timer->expire < tmp->expire)){//定时器无需调整位置
        return;
    }
    if (timer == head){//当前节点timer如果超时时间和头节点一样，那就要把它插到头节点后面
        head = head->next;
        head->prev = NULL;
        timer->next = NULL;
        add_timer(timer, head);
    }
    else{//先删除当前节点timer，然后再使用add_timer将其插入到正确的位置
        timer->prev->next = timer->next;
        timer->next->prev = timer->prev;
        add_timer(timer, timer->next);
    }
}
解释一下else的情况，else意味着当前节点的超时时间变大了，没有变小，因此不会将其插到head后面

什么意思呢？简单来说，如果满足else，那么现在这个timer不是新的timer，而是一个之前就存在于链表中的timer

这个timer随着时间的推移，其超时时间expire值已经发生了变化（肯定变大了），所以要更新这个timer的位置（往链表尾部移动）

在调整节点位置之前，我们必须从链表中将其删除。这是因为节点的expire值已经改变，如果我们不将其删除，它可能会位于错误的位置。

删完之后，使用add_timer再将其插入正确的位置（根据expire值找到正确的位置）

总结起来，adjust_timer函数的目的是重新调整定时器链表中节点的位置，以使链表仍然保持按照时间顺序排序。为了达到这个目的，我们需要先将要调整的节点从链表中删除，然后根据其新的expire值重新插入到正确的位置。

del_timer函数用于从链表中删除定时器。首先判断定时器是否是链表中唯一一个节点，如果是，则直接删除并将头尾指针置空。否则，根据定时器是否是头部或尾部节点进行不同的处理，然后修改前后节点的指针，最后删除定时器对象。

void sort_timer_lst::del_timer(util_timer *timer){
    if (!timer) return;

    if ((timer == head) && (timer == tail)){//如果链表中只有当前一个定时器
        delete timer;//删除后将头尾指针置空
        head = NULL;
        tail = NULL;
        return;
    }
    //当前节点位于链表头尾处时，分别处理
    if (timer == head){//如果当前定时器是头节点，将head指针指向下一个节点
        head = head->next;
        head->prev = NULL;//其prev再指向空便完成删除
        delete timer;//删除当前节点
        return;
    }
    if (timer == tail){//如果当前定时器位于链表尾部，将tail指针指向 前一个节点
        tail = tail->prev;
        tail->next = NULL;//next指针指向空
        delete timer;//删除当前节点
        return;
    }//在其他位置就直接删除就行
    timer->prev->next = timer->next;
    timer->next->prev = timer->prev;
    delete timer;
}
tick函数用于处理到期的定时器。

void sort_timer_lst::tick(){
    if (!head) return;//如果链表中没有定时器，那么就直接返回
    
    time_t cur = time(NULL);//获取当前系统时间
    util_timer* tmp = head;//将一个用于遍历的指针指向head
    while (tmp){
        if (cur < tmp->expire){//此时还没有超时
            break;//所以退出循环
        }//如果超时了，调用回调函数把连接对象删除了，即删除当前tmp指向的节点
        tmp->cb_func(tmp->user_data);
        head = tmp->next;//删除tmp
        if (head){//如果删的是头节点的话，还要改一下prev，因为删除时，头节点已经变为原头节点的下一个节点了
            head->prev = NULL;
        }
        delete tmp;
        tmp = head;//更新tmp指针
    }
}
解释一下cur < tmp->expire，也就是定时器设定时间的机制

整个流程是这样的：当有客户端连接时，我们为其创建一个socket，该fd同时会被一个定时器绑定，整个定时器在初始化时获取的时间是通过系统函数time得到的系统时间，然后因为我们人为的设定了一个超时时间参数TIMESLOT（15秒），因此定时器对象中的超时时间expire便是：当前系统时间+15秒。在检查定时器时间时，我们也是通过time获取系统时间cur，显然cur会逐渐接近超时时间expire，当cur < tmp->expire时，定时器还没有超时，大于等于就超时了，此时启动对该定时器的删除工作。

主要的成员函数介绍完了，回顾一下：add_timer函数构造双向链表、adjust_timer函数调整定时器在链表中的位置、del_timer函数删除某个定时器、tick函数处理超时定时器。

接下来要介绍一下辅助函数add_timer(util_timer *timer, util_timer *lst_head)，用于插入定时器到指定节点之后

void sort_timer_lst::add_timer(util_timer *timer, util_timer *lst_head){
    util_timer *prev = lst_head;//双指针，第一个指向head
    util_timer *tmp = prev->next;
    while (tmp){//遍历链表
        if (timer->expire < tmp->expire){//若当前遍历定时器的超时时间晚于输入定时器的超时时间
            //那么输入的定时器timer就应该在这里插入
            prev->next = timer;//将timer插入到两个指针之间
            timer->next = tmp;
            tmp->prev = timer;
            timer->prev = prev;
            break;
        }//不满足插入条件即timer的过期时间大于或等于tmp的过期时间，就移动双指针
        prev = tmp;
        tmp = tmp->next;
    }//遍历结束，还没有找到插入位置，就把节点插到链表尾部
    if (!tmp){//将timer插入到prev和链表末尾之间
        prev->next = timer;
        timer->prev = prev;
        timer->next = NULL;
        tail = timer;
    }
}
总结
至此，定时器链表的实现说明完毕

回顾一下流程：不论是什么处理任务（客户端产生新连接或者别的也好），只要调用了定时器函数，该定时器函数就会与输入的一个fd进行绑定。在初始化时，定时器会调用当前的系统时间，并在此基础上加上超时时间，形成当前定时器的超时时间。

之后就不断调用系统时间与超时时间比较，同时在链表中也不断比较定时器节点之间的时间间隔，由此移动定时器链表。

当系统时间也到达超时时间后，定时器超时，使用成员函数将其从定时器链表中删除。

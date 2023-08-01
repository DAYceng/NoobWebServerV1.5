
日志系统（默认异步启动）
===============
日志类维护着一个阻塞队列，与该队列进行数据交互时遵循生产者-消费者模式，生产者也就是这里的日志写入函数，会将标准化后的日志字符串作为元素push到队列中，等到被外界线程pop获取。

##### 日志类中的单例模式

在Log类的声明中（Log.h），其构造和析构函数被声明为私有，以防止外部直接创建Log类的对象。

```c++
class Log{
private:
    Log();
    virtual ~Log();
};
```

按照单例模式的流程，现在我们需要创建一个静态成员变量来保存唯一实例并提供一个公共的静态成员函数供外界获取唯一实例

在该日志类中，使用公共的静态成员函数`get_instance()`来完成上述两步

```c++
class Log{
public:
    static Log *get_instance(){
        static Log instance;
        return &instance;
    }
};
```

在`get_instance()`中，创建一个静态的Log类指针变量`instance`，并将其初始化为一个Log类的唯一实例，调用该函数即可返回唯一实例`instance`

> 为什么这里不用将static Log instance;（静态成员变量）声明为私有的？
>
> **因为在C++11之后，对于局部静态变量的初始化具备线程安全性**。
>
> 将其定义为局部静态变量即可，在作用域（包含它的函数或代码块）之外该变量是不可见的。

ps:日志类这里使用的是懒汉式

#### 日志类初始化

除了单例模式需要特别说明一下外，日志类Log本质上还是一个类，该怎么使用还是怎么使用就行

在创建唯一实例的时候也需要调用初始化函数`bool Log::init`

```c++
bool Log::init(const char *file_name, int close_log, int log_buf_size, int split_lines, int max_queue_size){
    //file_name表示日志文件的路径和名称，close_log表示是否关闭日志，log_buf_size表示日志缓冲区的大小，split_lines表示日志文件达到的最大行数时进行切割，max_queue_size表示异步模式下阻塞队列的长度。
}
```

初始化时，先判断是否要以异步模式运行，一般来说肯定是异步启动的

```c++
    ...
	//如果设置了max_queue_size,则设置为异步
    if (max_queue_size >= 1){
        m_is_async = true;
        m_log_queue = new block_queue<string>(max_queue_size);
        pthread_t tid;
        //flush_log_thread为回调函数,这里表示创建线程异步写日志
        pthread_create(&tid, NULL, flush_log_thread, NULL);
    }
	...
```

如果异步启动，那么将`m_is_async`标志设置为`true`，然后创建一个阻塞队列`m_log_queue`，指定该队列的最大长度为`max_queue_size`。

然后用`pthread_create`起一个新线程，该线程的目的是从队列中获取日志条目，并以异步方式将它们写入日志文件。

`flush_log_thread`（[详见](#section4)）作为回调函数传入新线程中（注意不是pthread_create），该函数负责从队列中获取日志条目并将其写入日志文件。

然后是一些参数的设置

```c++
	m_close_log = close_log;//将close_log参数的值赋给成员变量m_close_log。该变量指示是否关闭日志功能。
    m_log_buf_size = log_buf_size;//确定了日志缓冲区的大小，即缓冲区中可以存储的最大字符数
    m_buf = new char[m_log_buf_size];//动态分配了一个大小为m_log_buf_size的字符数组（缓冲区）。指向这个分配内存的指针存储在成员变量m_buf中，代表日志缓冲区。
    memset(m_buf, '\0', m_log_buf_size);//使用空字符('\0')对日志缓冲区进行初始化。确保缓冲区最初为空，准备存储日志消息。
    m_split_lines = split_lines;//指定每个日志文件中的最大行数，在超过此限制后会创建一个新的日志文件。
```

上述代码设置了Log类的各种配置参数，如日志缓冲区大小、每个日志文件的最大行数以及是否关闭日志功能。

因为我们是要生成日志嘛，日志最重要的信息就是时间，因此我们在初始化时需要把当前系统时间保存到一个结构体struct tm中，以便后续生成时间戳的时候使用。

```c++
	time_t t = time(NULL);//获取当前时间的秒数
    struct tm* sys_tm = localtime(&t);//使用localtime()函数将时间转换为本地时间
    struct tm my_tm = *sys_tm;//通过解引用sys_tm指针，将其中存储的struct tm结构体的内容复制到另一个名为my_tm的结构体中。这样可以在后续代码中使用my_tm来访问年、月、日等日期和时间信息。
```

然后就是要真正开始写日志文件，定义了一个`const char*`类型的指针`p`，并通过调用`strrchr(file_name, '/')`函数来查找`file_name`字符串中最后一个出现的斜杠字符('/')的位置。如果找不到斜杠字符，`p`将被赋值为`NULL`。

```c++
	const char *p = strrchr(file_name, '/');
    char log_full_name[256] = {0};//定义一个大小为256的字符数组log_full_name，并初始化为全零。
```

然后，使用条件语句检查`p`是否为`NULL`。如果`p`是`NULL`，表示`file_name`字符串中没有斜杠字符，即该字符串只包含文件名而不包含路径信息。在这种情况下，使用`snprintf`函数将日期和文件名格式化为新的字符串，并存储在`log_full_name`中。

如果`p`不为`NULL`，表示`file_name`字符串中存在斜杠字符，即该字符串包含路径信息。在这种情况下，使用`strcpy`函数将`p + 1`处开始的子字符串（即去除斜杠字符）复制到`log_name`字符数组中。同时，使用`strncpy`函数将从`file_name`的开头到`p - file_name + 1`个字符（包括斜杠字符）的子字符串复制到`dir_name`字符数组中。最后，使用`snprintf`函数将路径、日期和文件名格式化为新的字符串，并存储在`log_full_name`中。

```c++
	if (p == NULL){
        snprintf(log_full_name, 255, "%d_%02d_%02d_%s", my_tm.tm_year + 1900, my_tm.tm_mon + 1, my_tm.tm_mday, file_name);
    }
    else{
        strcpy(log_name, p + 1);
        strncpy(dir_name, file_name, p - file_name + 1);
        snprintf(log_full_name, 255, "%s%d_%02d_%02d_%s", dir_name, my_tm.tm_year + 1900, my_tm.tm_mon + 1, my_tm.tm_mday, log_name);
    }
```

接下来，将当前日期的日部分（`my_tm.tm_mday`）赋值给成员变量`m_today`。

最后，使用`fopen(log_full_name, "a")`函数以追加模式（"a"）打开`log_full_name`指定的日志文件。如果文件打开失败（返回`NULL`），则返回`false`。

```c++
 	m_today = my_tm.tm_mday;
    
    m_fp = fopen(log_full_name, "a");
    if (m_fp == NULL) return false;
```

至此，日志类初始化完成

我们确定类该实例的运行模式（异步），然后为该实例起了一个新线程，在该线程中维护一个阻塞队列，该队列采用生产者-消费者模式设计，使用循环数组实现。里面保存的是字符串类型的日志数据（例如：`string single_log;`）

然后我们还会获取系统时间并对日志缓冲区的大小等参数进行设置，最后打开一个日志文件准备记录日志信息。



##### <a name="section4">补充说明：flush_log_thread</a>

该函数没有定义，只有在头文件中的一个声明，`void *`表示返回类型为无类型指针。（使用了C++中的多线程编程和指针语法）

```c++
	static void* flush_log_thread(void *args){
        Log::get_instance()->async_write_log();
    }
```

`flush_log_thread`去调用了一个日志类实例中的私有方法async_write_log()来异步地写入日志，该函数的定义如下：

```c++
	void* async_write_log(){
        string single_log;
        //从阻塞队列中取出一个日志string，写入文件
        while (m_log_queue->pop(single_log))
        {
            m_mutex.lock();
            fputs(single_log.c_str(), m_fp);
            m_mutex.unlock();
        }
    }
```

在async_write_log()中，通过 `m_log_queue->pop(single_log)` 从阻塞队列（[详见](#section5)）中取出一个日志字符串 `single_log`。

使用 `fputs(single_log.c_str(), m_fp)` 将日志字符串写入文件。注意，这里还使用了互斥锁 `m_mutex` 来保护对文件指针 `m_fp` 的访问。

`fputs`函数是C和C++标准库中的一个函数，用于将字符串写入文件。

##### <a name="section5">补充说明：block_queue阻塞队列</a>

block_queue顾名思义其实现了一个阻塞队列，以循环数组的方式

该阻塞队列中的元素是通过`void *async_write_log()`


该阻塞队列以类的形式存在，在该类的构造函数中，通过`new T[max_size]`创建了一个大小为`max_size`的数组 `m_array`，用于存储元素。因为要实现的是一个“队列”，所以该数组要通过**头尾指针更新**来管理数据存放的位置

###### 队列的头尾指针更新

在说明为何使用头尾指针更新的策略之前需要先了解队列的基本概念

首先，队列是一种**先进先出**（FIFO）的数据结构，其中元素按照插入的顺序进行访问和移除。因此队列有两个关键操作：**入队**（enqueue）将元素添加到队列的尾部，**出队**（dequeue）将队列的头部元素移除并返回。

> 在代码实现中就是push和pop

队列通常使用**头指针**（front）和**尾指针**（rear）来管理元素的位置。这两个指针用于确定队列的起始点和结束点，从而允许我们在队列的两端进行插入和删除操作。

**初始化一个空队列时，头指针和尾指针都指向同一个位置**（例如，初始值为0）。当我们执行入队操作时，尾指针会递增，并将新元素放在尾指针所指向的位置。而在出队操作时，头指针会递增，并移动到下一个元素所在的位置。

> 头尾指针更新的一般过程：
>
> 1、初始化队列时，头指针和尾指针均指向同一个位置。
>
> 2、执行**入队操作**时，**尾指针递增**，并将**新元素放在尾指针所指向的位置**。
>
> 3、执行**出队操作**时，**头指针递增**，并移动到下一个元素所在的位置。注意，在出队操作之前，我们需要检查队列是否为空。

ok回到代码

前面我们说到，为了实现阻塞队列，代码中使用了循环数组，通过头指针和尾指针来管理数据的插入操作

队列的头指针`m_front`和尾指针`m_back`都是通过取模运算 `(m_back + 1) % m_max_size` 来实现循环的。(后面会有解释)

这意味着当队列的最后一个位置被占用时，下一个元素会从数组的起始位置重新开始存放。这样就形成了循环的效果。

###### 生产者-消费者模型

该阻塞队列实现了**线程安全的生产者-消费者模型**。这种模型是多线程编程中常见的一种设计模式，用于解决生产者线程和消费者线程之间的数据同步和通信问题。

简单来说，这个类实现的所谓"阻塞队列"中维护着一个数据结构，外部可以将数据输入该数据结构也可以从中取出数据。

在多线程的背景下，

**调用`push()`函数**将**日志字符串添加到阻塞队列中**的线程就是**生产者线程**；【在这里就是`void Log::write_log`】

通过**调用`pop()`函数**从阻塞队列中**取出日志字符串并写入文件**的线程就是**消费者线程**；【在这里就是`void *async_write_log()`】

前面说过，这两个函数分别实现阻塞队列的入队和出队操作。

`push(const T &item)`: 入队操作。

```c++
    //往队列添加元素，需要将所有使用队列的线程先唤醒
    //当有元素push进队列,相当于生产者生产了一个元素
    //若当前没有线程等待条件变量,则唤醒无意义
    bool push(const T &item){

        m_mutex.lock();//锁
        if (m_size >= m_max_size){//当前队列大小超过上限
            m_cond.broadcast();//broadcast()是对pthread_cond_broadcast的一个封装
            m_mutex.unlock();//解锁
            return false;
        }

        m_back = (m_back + 1) % m_max_size;//计算元素push到队列之后的位置
        m_array[m_back] = item;//将该元素放到指定位置

        m_size++;//队列长度增加

        m_cond.broadcast();//唤醒所有等待在条件变量 m_cond 上的线程，当调用 m_cond.broadcast() 时，所有正等待在 m_cond 上的线程都将被唤醒，并且它们将重新竞争获取相关的资源或执行特定的操作。这种广播机制确保没有线程会永久地阻塞在条件变量上，因为即使其中一个线程通过信号或其他方式唤醒，其他线程仍然可以继续执行。
        m_mutex.unlock();
        return true;
    }
```

首先获取互斥锁`m_mutex`，然后检查队列是否已满。

如果队列已满，就唤醒所有等待条件变量`m_cond`的线程，并返回false表示入队失败。

如果队列未满，则将元素插入队尾，并更新队列的大小。接着唤醒所有等待条件变量的线程，并释放互斥锁，最后返回true表示入队成功。

> `m_back = (m_back + 1) % m_max_size;` 的作用是将 `m_back` 后移一位，并且通过取模运算确保 `m_back` 在有效索引范围内循环更新，从而**实现队列的添加操作**。【**作用于队尾**】
>
> 举个例子：
>
> 假设当前队列的长度为 `m_max_size`，则 `m_back` 的范围是从 0 到 `m_max_size-1`。当插入一个新的元素时，我们需要将 `m_back` 后移一位来指向新的队尾。
>
> 如果 `m_back` 已经指向了队列的最后一个位置（即 `m_back == m_max_size - 1`），则 `(m_back + 1) % m_max_size` 的结果就是 0，即将 `m_back` 更新为 0，重新回到数组的开头。
>
> 这种循环更新索引的方式使得整个数组成为一个环形结构，实现了循环队列的特性。

`pop(T &item)`: 出队操作。

```c++
	//pop时,如果当前队列没有元素,将会等待条件变量
    bool pop(T &item){
        m_mutex.lock();
        while (m_size <= 0){
            
            if (!m_cond.wait(m_mutex.get())){//如果没有其他线程push进元素，那就没东西可pop，返回true，解锁
                m_mutex.unlock();
                return false;
            }
        }//有东西可以弹出，计算队头要移动的位置
        m_front = (m_front + 1) % m_max_size;
        item = m_array[m_front];//提供给用户一个接口，让他们能够获得从队列中弹出的元素（如果想获取的话）
        m_size--;
        m_mutex.unlock();
        return true;
    }
```

首先获取互斥锁`m_mutex`，然后检查队列是否为空。

如果队列为空，就进入循环等待条件变量`m_cond`，直到有新元素被加入队列。

`m_cond.wait(m_mutex.get())`是一个条件变量（Condition Variable）的等待操作，条件变量通常与互斥锁一起使用，以实现线程间的同步。

`m_cond` 是一个条件变量对象，`m_mutex.get()` 获取互斥锁对象。

在阻塞队列中，当队列为空时，调用`pop`函数会进入等待状态，直到有元素可供弹出或者超时。

如果等待失败，即没有其他线程通过 `push` 操作插入新的元素，那么 `!m_cond.wait(m_mutex.get())` 返回 `true`，即等待失败。在这种情况下，函数将立即返回并返回 `false`，表示弹出操作未成功。

一旦有新元素加入队列，或者超时时间达到，就从队头取出一个元素并更新队列的大小。最后释放互斥锁并返回true表示出队成功。

> 与push函数中的类似，`m_front = (m_front + 1) % m_max_size;` 的作用是将 `m_front` 后移一位，并且通过取模运算确保 `m_front` 在有效索引范围内循环更新，从而**实现队列的弹出操作**。【**作用于队头**】



这里有一个问题，如果观察的话会发现，在pop函数中，我们只是获取了队列头部的元素赋值给item，**并没有直接删除m_array中对应位置的元素**，那这也能算pop吗？

**是的**，因为这里使用的是循环数组，**在下一次push操作时，新的元素将会覆盖掉之前m_front所指向的位置，相当于间接删除了该元素**。

> 循环数组的索引m_front和m_back被用来追踪队列的头部和尾部。当调用pop函数时，我们通过更新m_front索引和减小队列大小（m_size）来模拟弹出元素的操作。这样做的好处是避免了频繁地移动数组中的元素，从而提高了性能。



###### 阻塞队列的功能函数

上面介绍的是实现阻塞队列的核心函数，除此之外，还需要提供一些方便的功能函数来辅助用户完成某些功能

例如：full()函数可以判断队列是否满了、empty()判断队列是否为空、front（）可以直接返回队首元素（其实就是直接返回m_front处的元素），同理还有back()等


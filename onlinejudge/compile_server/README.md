## compiler服务

OJ最核心的功能是什么？那肯定是判题啊，在具体一点说就是程序代码的提交与编译

因此我们先解决编译部分的功能，编译功能由`oj/compile_server/compiler.hpp`实现

```c++
#pragma once
...
#include "../comm/util.hpp"
#include "../comm/log.hpp"
```

特别说明一下头文件引入部分

`#pragma once`是一个预处理器指令，用于防止 C++ 头文件的重复包含。当编译器看到这个指令时，如果该头文件已经被包含过，那么编译器就不会再次包含它。这可以避免因头文件重复包含而导致的编译错误。

> 例如，假设你有一个名为 "my_header.h" 的头文件，并且你在两个不同的源文件中都包含了这个头文件。如果没有 `#pragma once`，那么 "my_header.h" 中的所有代码都会在两个源文件中重复出现。这可能会导致重复定义错误，例如变量或函数被重复定义。
>
> 如果你在 "my_header.h" 的顶部添加了 `#pragma once`，那么无论你在多少个源文件中包含 "my_header.h"，它的内容都只会出现一次。这就解决了重复定义的问题。
>
> 一种更传统（也是标准的）方式是使用包含保护（include guards）。例如：
>
> ```c++
> #ifndef MY_HEADER_H
> #define MY_HEADER_H
> 
> // ... 头文件内容 ...
> 
> #endif // MY_HEADER_H
> ```
>
> 这种方式的原理是，如果 `MY_HEADER_H` 已经被定义，那么头文件的内容就不会再被包含。这和 `#pragma once` 的效果是一样的，但是它是 C++ 所有编译器都支持的标准方式。

### 整体结构

```c++
namespace ns_compiler{
    // 这样你就可以不用写ns_util::某某了
    using namespace ns_util;
    using namespace ns_log;

    class Compiler{
    public:
        Compiler(){}
        ~Compiler(){}
        
        static bool Compile(const std::string &file_name){
           ...
        }
    };
}
```

整体结构并不复杂，创建一个命名空间ns_compiler，然后在里面定义了一个Compiler类，负责编译文件

### 将当前执行内容拷贝进子进程中

首先fork一个进程，此时应该返回进程号，返回符值则报错

```c++
			...
			pid_t pid = fork();
            if(pid < 0){
                LOG(ERROR) << "内部错误，创建子进程失败" << "\n";
                return false;
            }
            ...
```

### 子进程负责编译文件

返回0则意味着当前创建的进程为子进程，使用umask设置当前进程的掩码

> 在Unix和Linux系统中，每当创建新的文件或目录时，都会给它赋予一组默认的权限。这组默认权限被umask的值所影响。umask的值是一个三位或四位的八进制数，每一位对应文件的读（r）、写（w）、执行（x）权限。
>
> `umask(0);`中的0表示创建新文件或目录时，系统不应从默认权限中减去任何权限。也就是说，新创建的文件或目录将具有尽可能多的权限。
>
> 例如，如果默认的文件权限是666（读写权限），目录权限是777（读写执行权限），那么`umask(0);`则表示新创建的文件将保持666权限，新创建的目录将保持777权限。

然后，使用open函数打开（没有就创建）一个名为"file_name.stderr"的文件,并得到对应的文件描述符`_stderr`。如果打开失败，打印警告日志并退出进程。PathUtil::CompilerError(file_name).c_str()获取一个字符串，这个字符串代表了编译错误文件的路径，该路径基于输入的`file_name`动态生成

在编译部分，使用了`execlp`函数来启动`g++`编译器进行代码编译。

`execlp`函数可以在当前进程中**启动一个新的程序**，它会删除当前进程的正文段、数据段和栈，然后加载一个新程序的正文段、数据段和栈，并开始执行新程序。

### 父进程等待子进程执行结束

如果fork之后返回的id大于0，那么创建的是一个父进程，此时要等到其子进程结束后再做处理

```c++
			...
			else{
                waitpid(pid, nullptr, 0);
                //编译是否成功,就看有没有形成对应的可执行程序
                if(FileUtil::IsFileExists(PathUtil::Exe(file_name))){
                    LOG(INFO) << PathUtil::Src(file_name) << " 编译成功!" << "\n";
                    return true;
                }
            }
            LOG(ERROR) << "编译失败，没有形成可执行程序" << "\n";
            return false;
		}
    };
}
```

`waitpid(pid, nullptr, 0)` 等待的子进程就是在 `fork` 函数中创建的子进程，即编译器 `g++`。在子进程中，程序调用了 `execlp` 函数来替换当前进程的代码，从而启动了 `g++` 编译器进行代码编译。在父进程中，程序调用了 `waitpid` 函数来等待子进程结束，并检查生成的可执行文件是否存在，以判断编译是否成功。

### 疑问Q&A

这里可能会有疑问，为什么一会是父进程一会是子进程？

这与`fork`的机制有关

当程序执行到`fork`函数时，操作系统会创建一个新的进程（即子进程），**并将当前进程的状态完全复制到子进程中**，包括代码、数据、堆栈、打开的文件描述符等等。子进程会从`fork`函数的返回值开始执行程序，而父进程则继续执行`fork`函数后面的代码。因此，程序的执行流程会分裂成两个独立的进程，它们共享同一份代码，但是它们的数据是相互独立的。

简单来说，当static bool Compile函数执行到pid_t pid = fork();时，当前程序就已经在子进程中了，此时父进程与子进程的执行内容还是一致的，但到后面if的时候就开始有区别了，父进程因为PID大于0，所以会执行else内的代码，也就是等待子进程运行结束

而子进程因为PID = 0，所以会去执行else if (pid == 0)中的内容，也就是编译文件。


## runner运行服务

这部分实现了在线OJ系统中的代码运行功能

### 整体结构

惯例先看看结构

```c++
#pragma once

namespace ns_runner{
    using namespace ns_util;
    using namespace ns_log;

    class Runner{
    public:
        Runner() {}
        ~Runner() {}

    public:
        static void SetProcLimit(int _cpu_limit, int _mem_limit){}
        static int Run(const std::string &file_name, int cpu_limit, int mem_limit){}
    };
}
```

命名空间ns_runner下定义了一个`Runner`类，提供两个成员函数分别为：`SetProcLimit`和`Run`

### 设置进程资源

`SetProcLimit`负责设置进程占用资源大小的函数，包括CPU和内存的限制。

```c++
		//提供设置进程占用资源大小的接口
        static void SetProcLimit(int _cpu_limit, int _mem_limit){
            // 设置CPU时长
            struct rlimit cpu_rlimit;//存储资源限制的值
            cpu_rlimit.rlim_max = RLIM_INFINITY;
            cpu_rlimit.rlim_cur = _cpu_limit;
            setrlimit(RLIMIT_CPU, &cpu_rlimit);//当前进程的 CPU 时间限制为RLIM_INFINITY秒

            // 设置内存大小
            struct rlimit mem_rlimit;
            mem_rlimit.rlim_max = RLIM_INFINITY;
            mem_rlimit.rlim_cur = _mem_limit * 1024; //转化成为KB
            setrlimit(RLIMIT_AS, &mem_rlimit);
        }
//cpu_limit: 该程序运行的时候，可以使用的最大cpu资源上限
//mem_limit: 改程序运行的时候，可以使用的最大的内存大小(KB)
```

> `SetProcLimit`函数中的单位是秒和MB。这里的CPU时间限制是指当前进程在CPU上运行的时间，超过这个时间限制后，进程会被强制终止。虚拟内存限制是指当前进程可以使用的虚拟内存的大小，超过这个限制后，进程会被强制终止。

### 代码运行函数Run

`runner`接收三个参数：可执行文件的文件名、CPU 时间限制和内存限制。

程序运行有三种情况：①代码跑完，结果正确；②代码跑完，结果不正确；③代码没跑完，异常了

但实际上，Run函数不需要考虑代码的运行结果是否正确，**该函数只负责保证程序正确运行完毕**

结果正确与否，由测试用例决定的

#### 获取编译结果

```c++
		static int Run(const std::string &file_name, int cpu_limit, int mem_limit){
            std::string _execute = PathUtil::Exe(file_name);
            std::string _stdin   = PathUtil::Stdin(file_name);
            std::string _stdout  = PathUtil::Stdout(file_name);
            std::string _stderr  = PathUtil::Stderr(file_name);

            umask(0);
            int _stdin_fd = open(_stdin.c_str(), O_CREAT|O_RDONLY, 0644);
            int _stdout_fd = open(_stdout.c_str(), O_CREAT|O_WRONLY, 0644);
            int _stderr_fd = open(_stderr.c_str(), O_CREAT|O_WRONLY, 0644);

            if(_stdin_fd < 0 || _stdout_fd < 0 || _stderr_fd < 0){
                LOG(ERROR) << "运行时打开标准文件失败" << "\n";
                return -1; //代表打开文件失败
            }            
			...
        }
```

这里首先使用PathUtil类的一些方法来获取程序的执行路径、标准输入、标准输出和标准错误输出。

简单看一下：

`PathUtil::Exe`获取的是`compiler.hpp`编译后得到的`.exe`文件；（一个程序在默认启动时）

`PathUtil::Stdin`则拿到运行exe文件所需的标准输入**的文件路径**；（标准输入: 不处理）

`PathUtil::Stdout`获取运行exe文件后的标准输出**的文件路径**；（标准输出: 程序运行完成，输出结果是什么）

`PathUtil::Stderr`容易知道，就是获取了标准错误输出**的文件路径**；（标准错误: 运行时错误信息）

> 这里我把"文件路径"几个字加粗的原因是提醒读者注意：在线OJ实现"判题"功能时，会将提交代码的编译运行输出全过程给"文件化"
>
> OJ处理提交代码的过程都是基于文件交互实现的，这也体现了"Linux下一切皆文件"的特点

这些函数的返回值是一个字符串（路径），有了代码执行的各个过程的结果，现在需要去使用这些信息

因此，使用open来打开这些文件，生成对应的文件描述符，供后续操作使用

#### 处理结果

获得上述信息之后就要开始进行对应的处理操作，和compiler中的逻辑类似，我们创建子进程来处理各种操作

子线程创建失败，关闭前面创建的所有文件描述符，返回错误码

```c++
			pid_t pid = fork();
            if (pid < 0){
                LOG(ERROR) << "运行时创建子进程失败" << "\n";
                close(_stdin_fd);
                close(_stdout_fd);
                close(_stderr_fd);
                return -2; //代表创建子进程失败
            }
			...
```

##### 子进程逻辑

子线程执行pid == 0时的逻辑，设置cpu资源，然后调用`execl()`函数来执行可执行文件的函数，该函数会将当前进程替换为新的进程，新进程会执行指定的可执行文件。

梳理一下：

在调用fork()函数之后，会创建一个新的子进程，该子进程会复制父进程的所有资源（包括代码段、数据段、堆栈、文件描述符等等），并从fork()函数的返回值开始执行。

而在执行到execl()函数时，当前子进程会被替换为新的进程，也就是又创建了一个线程，该线程的父线程是调用fork()函数创建的子进程。

新进程会执行指定的可执行文件，原有的代码段、数据段、堆栈、文件描述符等资源都会被替换为新进程的资源，从而完成程序的执行。

```c++
			...
			else if (pid == 0){//如果编译出错，错误信息就会被写入到_stdin_fd/_stdout_fd/_stderr_fd文件中
                dup2(_stdin_fd, 0);
                dup2(_stdout_fd, 1);
                dup2(_stderr_fd, 2);

                SetProcLimit(cpu_limit, mem_limit);
                //*我要执行谁*|*我想在命令行上如何执行该程序*
                execl(_execute.c_str(), _execute.c_str(), nullptr);//以默认方式执行可执行文件_execute
                exit(1);
            }
			...
```

在这里，`_execute`是一个字符串，保存了要执行的可执行文件的路径和文件名。

`execl()`函数的第一个参数就是要执行的可执行文件的路径和文件名，第二个参数是在命令行上如何执行该可执行文件。

**由于我们想要以默认的方式执行该可执行文件，所以将第二个参数也设置为可执行文件的路径和文件名**。最后一个参数是一个空指针，表示可执行文件不需要接收任何命令行参数。

##### 父线程逻辑

`_stdin_fd`、_`stdout_fd`和`_stderr_fd`是在Unix/Linux系统中用于标准输入、标准输出和标准错误输出的文件描述符。

已经被子线程使用，因此在父线程中要将它们关闭，然后等待子线程运行完毕

```c++
			...
			else{
                close(_stdin_fd);
                close(_stdout_fd);
                close(_stderr_fd);
                int status = 0;
                waitpid(pid, &status, 0);
                // 程序运行异常，一定是因为因为收到了信号！
                LOG(INFO) << "运行完毕, info: " << (status & 0x7F) << "\n"; 
                return status & 0x7F;
            }
        }
    };
}
```

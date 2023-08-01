## 使用MVC结构设计OJ服务

OJ的核心功能基本上都实现了，现在需要设计出OJ的"本体"，也就是一个能够提供给用户使用的完整的系统

> 本质上就是建立一个网站

那么在我们的"OJ网站"中，得有一个首页吧，我们可以用题目列表充当首页

还得有编辑区域页面和提交判题功能(编译并运行)

这里采用MVC结构来构建OJ网站

> MVC结构的三个部分具有以下功能：
>
> 1. 模型（Model）：负责处理应用程序的数据和业务逻辑。通常是和数据交互的模块，比如，**对题库进行增删改查**。
> 2. 视图（View）：负责呈现应用程序的用户界面。通常是拿到数据之后，要进行**构建网页，渲染网页内容，展示给用户**(浏览器)
> 3. 控制器（Controller）：负责处理用户输入和控制应用程序的流程。控制器通常包括接收用户输入的逻辑和决定如何响应该输入的逻辑。就是我们的**核心业务逻辑**。

### 用户请求的服务路由功能（View）

这部分是基于MVC（Model-View-Controller）模式设计的在线OJ系统中的**视图部分**，即View。

**View负责处理用户的请求并呈现相应的页面，将Model中的数据呈现给用户**。

代码主要在在oj_server.cc中完成，View使用了httplib库来提供HTTP服务。（没错，这里又构建了一个HTTP服务器）

View中定义了一个名为`Server`的对象`svr`，它用于处理用户的请求。`ctrl`用于处理用户请求时的业务逻辑。

```c++
#include <iostream>
#include <signal.h>

#include "../comm/httplib.h"
#include "oj_control.hpp"

using namespace httplib;
using namespace ns_control;
static Control *ctrl_ptr = nullptr;

//注册了 Recovery 函数来处理 SIGQUIT 信号。当系统收到 SIGQUIT 信号时，会执行 Recovery 函数中的代码，该函数会调用 ctrl.RecoveryMachine 方法来恢复系统的状态。
void Recovery(int signo) ctrl_ptr->RecoveryMachine();//详见控制器（Controller）部分的介绍

int main(){
    signal(SIGQUIT, Recovery);   
    Server svr;//用户请求的服务路由功能

    Control ctrl;
    ctrl_ptr = &ctrl;
	...    
}
```

通过`Server`对象`svr`的`Get`方法和`Post`方法来定义HTTP请求的路由和处理函数。这里绑定了三个HTTP路由：

- /all_questions：获取**所有题目列表**，返回一个包含所有题目的HTML页面。
- /question/{id}：获取**指定题目的内容**，其中 {id} 是题目的编号。
- /judge/{id}：**提交用户的代码**，并对指定题目进行测试，返回测试结果的JSON格式字符串。

`/all_questions` 路由对应的处理函数为 `ctrl.AllQuestions`，该函数会返回一张包含所有题目信息的 HTML 页面。`/question/(\d+)` 路由对应的处理函数为 `ctrl.Question`，该函数会根据题目编号返回该题目的详细信息。`/judge/(\d+)` 路由对应的处理函数为 `ctrl.Judge`，该函数会根据题目编号和用户提交的代码进行判题，并返回判题结果。

```c++
	...
	// 获取所有的题目列表
    svr.Get("/all_questions", [&ctrl](const Request &req, Response &resp){
        //返回一张包含有所有题目的html网页
        std::string html;
        ctrl.AllQuestions(&html);
        //用户看到的是什么呢？？网页数据 + 拼上了题目相关的数据
        resp.set_content(html, "text/html; charset=utf-8");
    });

    // 用户要根据题目编号，获取题目的内容
    // /question/100 -> 正则匹配
    // R"()", 原始字符串raw string,保持字符串内容的原貌，不用做相关的转义
    svr.Get(R"(/question/(\d+))", [&ctrl](const Request &req, Response &resp){
        std::string number = req.matches[1];
        std::string html;
        ctrl.Question(number, &html);
        resp.set_content(html, "text/html; charset=utf-8");
    });

    // 用户提交代码，使用我们的判题功能(1. 每道题的测试用例 2. compile_and_run)
    svr.Post(R"(/judge/(\d+))", [&ctrl](const Request &req, Response &resp){
        std::string number = req.matches[1];
        std::string result_json;
        ctrl.Judge(number, req.body, &result_json);
        resp.set_content(result_json, "application/json;charset=utf-8");
        // resp.set_content("指定题目的判题: " + number, "text/plain; charset=utf-8");
    });
    //这些静态文件可以通过HTTP请求来访问，例如通过http://127.0.0.1:8080/static/index.html访问HTML页面的模板文件。
    svr.set_base_dir("./wwwroot");//设置静态文件的根目录
    svr.listen("0.0.0.0", 8080);
    return 0;
}
```

在程序启动时，它会监听本地8080端口，等待用户的请求。当用户请求到达时，httplib库会帮助程序解析该请求，并调用相应的请求处理函数进行处理。请求处理函数会调用Control类的成员函数进行业务逻辑的处理，然后将处理结果返回给httplib库，由httplib库帮助程序构造响应并发送给用户。

总体来说，View 负责处理用户的请求并呈现相应的页面，它将 Model 中的数据呈现给用户，并对用户的请求进行路由和处理。View 中使用了信号机制和静态文件的设置，以提供系统的恢复功能和访问静态文件的能力。同时，View 也是系统的入口部分，对系统的稳定性和性能起着至关重要的作用。

### 操作数据（Model）

接下来该操作数据了，这部分交给OJ中的**数据操作部分**（即Model）来完成

**Model负责管理系统中的数据并提供对外的数据访问接口，以供Controller和View使用。**

看看结构

```c++
#pragma once

#include "../comm/util.hpp"
#include "../comm/log.hpp"

#include <iostream>
#include <string>
#include <vector>
#include <unordered_map>
#include <fstream>
#include <cstdlib>
#include <cassert>

namespace ns_model{// 根据题目list文件，加载所有的题目信息到内存中
    using namespace std;
    using namespace ns_log;
    using namespace ns_util;

    struct Question{
        std::string number; //题目编号，唯一
        std::string title;  //题目的标题
        std::string star;   //难度: 简单 中等 困难
        int cpu_limit;      //题目的时间要求(S)
        int mem_limit;      //题目的空间要去(KB)
        std::string desc;   //题目的描述
        std::string header; //题目预设给用户在线编辑器的代码
        std::string tail;   //题目的测试用例，需要和header拼接，形成完整代码
    };

    const std::string questins_list = "./questions/questions.list";
    const std::string questins_path = "./questions/";

    class Model
    {
    private:
        unordered_map<string, Question> questions;
    public:
        Model(){assert(LoadQuestionList(questins_list));}
        bool LoadQuestionList(const string &question_list){}
        bool GetAllQuestions(vector<Question> *out){}
        bool GetOneQuestion(const std::string &number, Question *q){}
        ~Model(){}
    };
} // namespace ns_model
```

Model包含一个名为`questions`的`unordered_map`，它存储了所有题目的信息，其中键为题目编号，值为`Question`结构体，表示题目的详细信息。`Question`结构体包含了题目的编号、标题、难度、时间和空间要求、题目描述、预设代码和测试用例等信息。

然后再来看其提供的成员函数

#### LoadQuestionList加载题目列表

顾名思义，该函数用于加载题目列表文件和各个**题目的描述**、**预设代码**和**测试用例**等信息

```c++
		...
		bool LoadQuestionList(const string &question_list){  
            //接受一个题目列表文件的路径作为参数。
        }
```

然后我们通过ifstream对象打开题目列表文件，并检查是否成功打开。如果打开失败，会输出错误日志并返回false。

```c++
            ...
			ifstream in(question_list);
            if(!in.is_open()){
                LOG(FATAL) << " 加载题库失败,请检查是否存在题库文件" << "\n";
                return false;
            }
			...
```

打开文件后使用getline函数逐行读取题目列表文件，将每一行的内容保存在变量line中。

```c++
			string line;
            while(getline(in, line))
            ...
```

使用`StringUtil::SplitString`将每行内容按空格拆分为多个字符串（题目信息，保存在tokens容器中）

```c++
            vector<string> tokens;
            StringUtil::SplitString(line, &tokens, " ");
			...
```

拆分后的字符串依次表示题目的编号、标题、难度、时间要求和空间要求。

然后创建Question对象并填充题目信息

```c++
            Question q;
            q.number = tokens[0];
            q.title = tokens[1];
            q.star = tokens[2];
            q.cpu_limit = atoi(tokens[3].c_str());
            q.mem_limit = atoi(tokens[4].c_str());
			...
```

根据拆分的结果，将值分别赋给Question结构体对象的对应成员变量。

我们有一个路径`const std::string questins_path = "./questions/";`用于存放题目，题目主要是以题目描述、解题模板和测试用例（需要和header拼接，形成完整代码）组成，分别对应desc.txt、header.cpp和tail.cpp这几个文件

```c++
            string path = questins_path;
            path += q.number;
            path += "/";

            FileUtil::ReadFile(path+"desc.txt", &(q.desc), true);
            FileUtil::ReadFile(path+"header.cpp", &(q.header), true);
            FileUtil::ReadFile(path+"tail.cpp", &(q.tail), true);
			...
```

使用`FileUtil::ReadFile`函数读取这些文件的内容，分别存储到Question对象的desc、header和tail成员变量中。

然后将题目信息作为键，Question对象q作为值，添加到unordered_map容器`questions`中，随后关闭列表文件并返回加载结果

```c++
            questions.insert({q.number, q});
            in.close();
            return true;
		}
```

#### GetAllQuestions获取所有题目

LoadQuestionList加载questions.list文件中的题目信息之后会将其存放于questions（unordered_map<string, Question>）中，我们需要将这些Question取出来放到一个vector中以便使用，GetAllQuestions函数就是干这件事情的

```c++
		bool GetAllQuestions(vector<Question> *out){
            if(questions.size() == 0){
                LOG(ERROR) << "用户获取题库失败" << "\n";
                return false;
            }
            for(const auto &q : questions) out->push_back(q.second); //first: key, second: value
            return true;
        }
```

逻辑很简单，就是判断一下questions的大小是否合法，合法就把里面的对象都遍历出来保存至一个`vector<Question>`中。

#### GetOneQuestion获取指定题号的题目

没什么好说的，就根据题号从questions中查一个题目，有这一题的话就把题目返回给一个Question *q

```c++
		bool GetOneQuestion(const std::string &number, Question *q){
            const auto& iter = questions.find(number);
            if(iter == questions.end()){
                LOG(ERROR) << "用户获取题目失败, 题目编号: " << number << "\n";
                return false;
            }
            (*q) = iter->second;
            return true;
        }
```

### 控制模块（Controller）

控制模块负责处理用户请求、调用模型（Model）获取数据，并将数据传递给视图（View）进行渲染，最后将渲染结果返回给用户。

代码量比较大就不贴结构了

该控制模块包含以下主要类和功能：

1. `Machine`类：表示提供服务的主机。每个主机有一个唯一的 IP 地址和端口号，以及负载信息。该类提供了提升和降低主机负载的方法。
2. `LoadBlance`类：负载均衡模块，管理所有可用的主机。它维护了在线和离线主机的列表，并提供了选择负载最低的主机的方法。它还提供了离线和上线主机的功能。
3. `Control`类：核心业务逻辑的控制器。它包含了模型（Model）、视图（View）和负载均衡器（LoadBlance）的实例。该类提供了以下功能：
   - `RecoveryMachine`方法：将离线的主机重新上线。
   - `AllQuestions`方法：获取所有题目的信息，并将其构建成网页。
   - `Question`方法：根据题目编号获取指定题目的信息，并将其构建成网页。
   - `Judge`方法：根据题目编号、用户提交的代码和输入数据，选择负载最低的主机发起编译和运行服务的请求，并将结果返回。

#### 什么是负载均衡？

负载均衡是一种将工作负载分配到多个服务器来提高网站、应用、数据库或其他服务的性能和可靠性的手段。

通常由一个组件实现

> 举个例子
>
> 在没有负载均衡的web架构中，用户是直连到web服务器，如果这个服务器宕机了，那么用户自然也就没办法访问了。另外，如果同时有很多用户试图访问服务器，超过了其能处理的极限，就会出现加载速度缓慢或根本无法连接的情况。
>
> 而通过在后端引入一个**负载均衡器**和**至少一个额外的web服务器**，可以缓解这个故障。通常情况下，所有的后端服务器会保证提供相同的内容，以便用户无论哪个服务器响应，都能收到一致的内容。

#### LoadBlance类实现负载均衡

在OJ系统中，负载均衡的作用是将用户提交的代码编译和运行服务请求分配到可用的主机上，以提高系统的并发处理能力和稳定性。

需要注意的是，实际上我们只在一台服务器上开发部署OJ，因此这里我们实现的负载均衡，更多的是为多台主机的情况做好准备，这种设计可以使系统更具弹性和可扩展性。

有了上述前提之后，我们来看LoadBlance类是如何做到负载均衡的

首先，负载均衡模块（`LoadBlance`类）维护了一组可用的主机，并根据主机的负载情况选择合适的主机处理请求。这是符合负载均衡的核心思路的。

我们使用`Machine`类来模拟主机（[详见](#section20)），每个主机有一个唯一的IP地址和端口号，以及负载信息。

在负载均衡模块中，`Machine`类的实例被用来表示可提供编译服务的主机。`LoadBlance`类维护了一个主机列表`machines`，其中每个主机都是一个`Machine`类的实例。负载均衡模块可以根据主机的负载情况选择合适的主机来处理请求。
`LoadBlance`类提供了以下功能：

- `LoadConf(const std::string &machine_conf)`：从配置文件中加载主机信息，创建`Machine`实例，并将其添加到主机列表中。
- `SmartChoice(int *id, Machine **m)`：根据负载情况选择负载最低的主机，并通过`id`和`m`返回选择的主机的信息。
- `OfflineMachine(int which)`：离线指定的主机，将其从在线主机列表中移除，并将其添加到离线主机列表中。
- `OnlineMachine()`：将所有离线主机重新上线，将其从离线主机列表中移除，并添加到在线主机列表中。
- `ShowMachines()`：显示当前在线和离线主机的列表。

##### 添加主机信息LoadConf

该函数的主要作用是将配置文件中的主机信息解析成`Machine`对象，并将这些对象添加到`machines`向量中。同时，该函数将所有在线的主机 id 添加到`online`向量中。在正常运行时，`online`向量中应该包含所有在线的主机的id，而`offline`向量中应该为空。在负载均衡时，通过遍历`online`向量，可以得到所有在线主机的负载并选择其中负载最低的主机。

在该函数中，我们使用标准流打开machine_conf配置文件，该文件中其实就是存着一堆`IP:端口`字符串

```c++
	public:
        bool LoadConf(const std::string &machine_conf){
            std::ifstream in(machine_conf);
            if (!in.is_open()){
                LOG(FATAL) << " 加载: " << machine_conf << " 失败"
                           << "\n";
                return false;
            }
            ...
```

然后逐行读取配置文件，对于每一行，使用`StringUtil::SplitString`函数对其进行切分并得到该主机的ip和port。

使用这些信息创建一个`Machine`对象m，load字段设置为0，mtx字段设置为一个新创建的互斥量。

```c++
			std::string line;
            while (std::getline(in, line)){
                std::vector<std::string> tokens;
                StringUtil::SplitString(line, &tokens, ":");
                if (tokens.size() != 2){
                    LOG(WARNING) << " 切分 " << line << " 失败"
                                 << "\n";
                    continue;
                }
                Machine m;
                m.ip = tokens[0];
                m.port = atoi(tokens[1].c_str());
                m.load = 0;
                m.mtx = new std::mutex();

                online.push_back(machines.size());
                machines.push_back(m);
            }

            in.close();
            return true;
        }
```

将该`Machine`对象添加到`machines`向量中，并将其在`machines`向量中的下标添加到`online`向量中。



##### 选取主机SmartChoice

在负载均衡中，我们需要将工作负载均匀的分配以减轻系统的压力，简单来说就是让每个机器的工作量都差不多，因此需要不断寻找负载较小的主机，将一部分工作分配给这些主机

该函数的作用就是根据已知的在线主机列表，选择负载最低的主机进行任务分配。遍历在线主机，找到负载最小的主机，并返回其ID和指针。

选择编译主机的方式是通过遍历所有在线的编译主机，找到所有负载最小的机器，然后将代码分配给该主机。

具体而言，遍历的方式是通过for循环实现的，而选择负载最小的机器是通过比较每个在线机器的负载实现的。

```c++
			bool SmartChoice(int *id, Machine **m){
            // 1. 使用选择好的主机(更新该主机的负载)
            // 2. 我们需要可能离线该主机
            mtx.lock();
            int online_num = online.size();
            if (online_num == 0){
                mtx.unlock();
                LOG(FATAL) << " 所有的后端编译主机已经离线, 请运维的同事尽快查看"
                           << "\n";
                return false;
            }
            *id = online[0];// 通过遍历的方式，找到所有负载最小的机器
            *m = &machines[online[0]];
            uint64_t min_load = machines[online[0]].Load();
            for (int i = 1; i < online_num; i++){
                uint64_t curr_load = machines[online[i]].Load();
                if (min_load > curr_load){
                    min_load = curr_load;
                    *id = online[i];
                    *m = &machines[online[i]];
                }
            }
            mtx.unlock();
            return true;
        }
```



##### <a name="section20">补充说明：Machine类</a>

这个类用于模拟可用于编译的服务器，因此该类具有以下成员变量：

- `ip`：主机的IP地址。
- `port`：主机的端口号。
- `load`：主机的负载，表示当前主机正在处理的请求数量。
- `mtx`：互斥锁，用于保护对`load`变量的并发访问。

定义好用于表示负载请求的变量后，要提供对应的成员函数去修改这些变量

- `IncLoad()`：增加主机的负载，表示有新的请求进入主机处理。
- `DecLoad()`：减少主机的负载，表示有请求处理完成离开主机。
- `ResetLoad()`：重置主机的负载，将负载设置为零。
- `Load()`：获取主机的负载。

总结一下Machine类做的事情：在类中定义了模拟主机的IP、端口和负载变量并提供一系列方法操控修改负载变量的数值，达到模拟服务器处理不同数量请求的情况。





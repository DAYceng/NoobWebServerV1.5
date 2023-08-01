
http连接处理类
===============
根据状态转移,通过主从状态机封装了http连接类。其中,主状态机在内部调用从状态机,从状态机将处理状态和数据传给主状态机
> * 客户端发出http连接请求
> * 从状态机读取数据,更新自身状态和接收数据,传给主状态机
> * 主状态机根据从状态机状态,更新自身状态,决定响应请求还是继续读取

###### <a name="section7">补充说明：void http_conn::process()</a>

该函数用于处理获取到的数据

```c++
void http_conn::process(){
    HTTP_CODE read_ret = process_read();
    if (read_ret == NO_REQUEST){
        modfd(m_epollfd, m_sockfd, EPOLLIN, m_TRIGMode);
        return;
    }
    bool write_ret = process_write(read_ret);
    if (!write_ret) close_conn();
    modfd(m_epollfd, m_sockfd, EPOLLOUT, m_TRIGMode);
}
```

可以看到，这里实际上就是调用了`process_read()`和`process_write()`两个函数来处理数据

工作流程如下：

1. 调用`process_read()`函数进行读取和解析请求（[详见](#section8)）：
   - `process_read()`函数负责从套接字中读取数据，并解析HTTP请求。
   - 返回值read_ret表示读取和解析的结果，可能的取值包括：
     - `NO_REQUEST`：表示没有完整的HTTP请求，需要继续等待数据到达。
     - `GET_REQUEST`：表示成功解析出一个完整的GET请求。
     - `BAD_REQUEST`：表示解析请求出现错误，请求格式不正确。
     - 其他可能的取值，用于表示其他HTTP请求方法（如POST、PUT等）或其他错误情况。
2. 根据`read_ret`的值进行处理：
   - 如果`read_ret`为`NO_REQUEST`，表示没有完整的HTTP请求，将套接字的事件设置为可读，并返回等待下一次可读事件的到达。
   - 如果`read_ret`为其他值，表示成功解析出一个完整的HTTP请求或出现错误，需要进行下一步的处理。
3. 调用`process_write()`函数进行响应处理（）：
   - `process_write()`函数负责根据`read_ret`的值生成HTTP响应，并将响应数据写入套接字。
   - 返回值`write_ret`表示写入套接字的结果，为`true`表示写入成功，为`false`表示写入失败。
4. 根据`write_ret`的值进行处理：
   - 如果`write_ret`为`false`，表示写入套接字失败，需要关闭连接。
   - 如果`write_ret`为`true`，表示写入套接字成功，将套接字的事件设置为可写，并等待下一次可写事件的到达。

###### <a name="section9">补充说明：bool http_conn::process_write(HTTP_CODE ret)</a>

虽然该函数也被process()调用，但是其没有使用主从状态机模式去设计，该函数的作用是**根据传入的`HTTP_CODE`参数生成HTTP响应，并将生成的响应内容添加到写缓冲区中**。

`process_write()`函数根据传入的`HTTP_CODE`参数，**针对不同的状态码生成不同的响应内容**。它通过调用一系列辅助函数（如`add_status_line()`、`add_headers()`和`add_content()`）将响应的状态行、响应头和响应体添加到写缓冲区中。

（不贴代码了，有点长）

##### <a name="section8">主从状态机</a>

所谓的"**主从状态机**"其实就是指http_conn::HTTP_CODE http_conn::process_read()遵循的设计模式，该函数根据主从状态机模式进行设计，用于处理不同状态。
因为之前有详细写过这部分的介绍，这里就概括一下就行

**主状态机**：`http_conn::process_read()`函数是主状态机。它负责解析HTTP请求的不同部分，并根据当前状态执行相应的操作。主状态机在循环中不断解析一行数据，并根据解析的结果进行状态切换和处理。主状态机的状态包括`CHECK_STATE_REQUESTLINE`、`CHECK_STATE_HEADER`和`CHECK_STATE_CONTENT`。



**从状态机**：`parse_line()`函数是从状态机。它在主状态机中被调用，用于解析一行数据的状态。从状态机的任务是根据当前解析的数据判断是否解析完成一行，并返回相应的状态。从状态机的状态包括`LINE_OK`、`LINE_BAD`和`LINE_OPEN`。



**主状态机和从状态机的交互**：主状态机在循环中不断调用从状态机的`parse_line()`函数来解析一行数据的状态。如果从状态机返回的状态为`LINE_OK`，表示成功解析一行数据，主状态机根据当前状态进行相应的处理。如果从状态机返回的状态不是`LINE_OK`，则继续循环解析下一行数据。



主状态机根据从状态机的返回结果进行不同的处理，包括解析请求行、解析请求头、解析请求数据等。根据不同的解析结果，主状态机会返回不同的`HTTP_CODE`，用于后续的处理和生成HTTP响应。

一旦从状态机解析完整个HTTP请求，主状态机就会调用`do_request()`函数来处理具体的请求信息。该函数根据解析的请求信息生成HTTP响应。它会根据请求类型和URL构建实际的文件路径，并进行相应的处理。例如，如果是CGI请求，它会处理登录和注册等操作；如果是静态文件请求，它会检查文件的权限和类型，并将文件映射到内存中。

> 总体而言，这个主从状态机的作用是实现了对HTTP请求的解析和处理，以及生成相应的HTTP响应。它通过合理的状态切换和处理逻辑，使得Web服务器能够正确地响应客户端的请求，并处理各种错误情况。
>
> 总结：
>
> - 主状态机是`http_conn::process_read()`函数，负责解析HTTP请求的不同部分。
> - 从状态机是`parse_line()`函数，用于解析一行数据的状态。
> - 主状态机通过循环调用从状态机来解析数据，并根据解析结果进行状态切换和处理。
> - 主状态机和从状态机的交互通过从状态机返回的状态来完成。


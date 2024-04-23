# 简介
管道是一种允许一个进程和另一个进程通信的IPC机制。有两种类型的管道：
- 匿名管道（Anonymous Pipes）：提供了一个单向的、非永久通信通道。常用于父子进程间的通信。
- 命名管道（Named Pipes）：提供一个双向的通信机制，不仅限于父子进程间的通信，也可以在不同计算机间通过网络进行通信。
命名管道可以被多个进程使用，可以是阻塞或非阻塞模式。管道内的数据完整性由操作系统保证，数据读写顺序与写入顺序一致。

管道通过以下原理实现通信：
- 创建管道：一个进程创建管道，并生成一个唯一的管道名称。
- 连接：其他进程通过管道名连接该管道。
- 读写操作：连接成功后，进程可以向管道写入数据或从管道读取数据。
- 关闭管道：使用完毕后，进程应关闭管道句柄以释放资源。

### 匿名管道代码示例
以下是匿名管道的简单示例代码。假定父进程创建管道，并生成一个子进程来读取管道数据：
```cpp
#include <windows.h>
#include <stdio.h>

int main() {
    HANDLE hRead, hWrite;
    SECURITY_ATTRIBUTES sa = {sizeof(SECURITY_ATTRIBUTES), NULL, TRUE};
    
    // 创建匿名管道
    if (!CreatePipe(&hRead, &hWrite, &sa, 0)){
        printf("Create Pipe failed. Error Code: %d\n", GetLastError());
        return 1;
    }

    // 创建子进程
    STARTUPINFO si = {sizeof(STARTUPINFO)};
    PROCESS_INFORMATION pi;
    
    si.hStdError = hWrite;
    si.hStdOutput = hWrite;
    si.dwFlags = STARTF_USESTDHANDLES;

    // 创建子进程  
    if (!CreateProcess(NULL, "child.exe", NULL, NULL, TRUE, 0, NULL, NULL, &si, &pi)){
        printf("Create Process failed. Error Code: %d\n", GetLastError());
        return 1;
    }
    
    // 关闭不需要的管道句柄
    CloseHandle(hWrite);

    // 父进程从管道读取数据
    char buffer[4096];
    DWORD bytesRead;
    while (ReadFile(hRead, buffer, 4096, &bytesRead, NULL)){
        printf("Read from pipe: %s\n", buffer);
    }

    // 关闭管道句柄
    CloseHandle(hRead);
    CloseHandle(pi.hProcess);
    CloseHandle(pi.hThread);
    
    return 0;
}
```
在上面的代码中，父进程创建了一个匿名管道，并设置子进程的标准输出和标准错误为该管道的写端。之后，创建了子进程（在这个例子中是 `child.exe`），父进程通过读端句柄从管道中读取子进程输出的数据，然后关闭了管道和进程句柄。

### 命名管道代码示例
#### server端
```cpp
// NamedPipeServer.c
#include <windows.h>
#include <stdio.h>

int main() {
    HANDLE hPipe;
    char buffer[1024];
    DWORD dwWritten, dwRead;
 
    // 创建命名管道
    hPipe = CreateNamedPipe(TEXT("\\\\.\\pipe\\MyPipe"),
                            PIPE_ACCESS_DUPLEX,
                            PIPE_TYPE_MESSAGE | PIPE_READMODE_MESSAGE | PIPE_WAIT,
                            PIPE_UNLIMITED_INSTANCES,
                            1024,
                            1024,
                            NMPWAIT_USE_DEFAULT_WAIT,
                            NULL);
    if (hPipe == INVALID_HANDLE_VALUE) {
        printf("Error creating pipe. Error Code: %d\n", GetLastError());
        return 1;
    }

    // 等待客户端连接
    printf("Waiting for client connection...\n");
    if (!ConnectNamedPipe(hPipe, NULL)) {
        printf("Error connecting to pipe. Error Code: %d\n", GetLastError());
        CloseHandle(hPipe);
        return 1;
    }

    // 向客户端发送消息
    printf("Sending data to client...\n");
    if (!WriteFile(hPipe, "Hello, Client!", 13, &dwWritten, NULL)) {
        printf("Error writing to pipe. Error Code: %d\n", GetLastError());
        CloseHandle(hPipe);
        return 1;
    }

    // 从客户端接收消息
    if (!ReadFile(hPipe, buffer, sizeof(buffer), &dwRead, NULL)) {
        printf("Error reading from pipe. Error Code: %d\n", GetLastError());
        CloseHandle(hPipe);
        return 1;
    }

    printf("Received data from client: \"%s\"\n", buffer);

    // 关闭管道
    CloseHandle(hPipe);

    return 0;
}
```
#### client
```cpp
// NamedPipeClient.c
#include <windows.h>
#include <stdio.h>

int main() {
    HANDLE hPipe;
    DWORD dwWritten, dwRead;
    char buffer[1024];

    // 连接命名管道
    hPipe = CreateFile(TEXT("\\\\.\\pipe\\MyPipe"),
                       GENERIC_READ | GENERIC_WRITE,
                       0,
                       NULL,
                       OPEN_EXISTING,
                       0,
                       NULL);
    
    if (hPipe == INVALID_HANDLE_VALUE) {
        printf("Error connecting to pipe. Error Code: %d\n", GetLastError());
        return 1;
    }

    // 从服务端接收消息
    if (!ReadFile(hPipe, buffer, sizeof(buffer), &dwRead, NULL)) {
        printf("Error reading from pipe. Error Code: %d\n", GetLastError());
        CloseHandle(hPipe);
        return 1;
    }

    printf("Received data from server: \"%s\"\n", buffer);

    // 向服务端发送响应
    printf("Sending response to server...\n");
    if (!WriteFile(hPipe, "Hello, Server!", 14, &dwWritten, NULL)) {
        printf("Error writing to pipe. Error Code: %d\n", GetLastError());
        CloseHandle(hPipe);
        return 1;
    }

    // 关闭管道
    CloseHandle(hPipe);

    return 0;
}
```

> https://www.cnblogs.com/iBinary/p/10949198.html

Windows管道是一种数据传输机制，它允许一个进程将输出数据传递给另一个进程，同时另一个进程可以从管道读取数据。Windows管道是一种无名管道，它没有名称，只能用于传递数据。Windows管道的实现原理是基于操作系统内核的机制，通过创建一个数据缓冲区来存储数据，同时在发送和接收数据时使用同步和互斥机制来保证数据的安全传输。
- 数据通信采用共享内存
- 同步机制采用windows的Event

  
> https://bbs.kanxue.com/thread-26252.htm
> https://cloud.tencent.com/developer/article/1625924

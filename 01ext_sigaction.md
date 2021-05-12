英文文档查看: https://www.ibm.com/docs/en/zos/2.4.0?topic=functions-sigaction-examine-change-signal-action

### sigaction()

```c
int sigaction(int sig, const struct sigaction *__restrict__ new, 
          struct sigaction *__restrict__ old);
```

检查并设置与此信号相关的操作。

sig 表示识别信号，必须从 `signal.h` 中选择宏定义。

- Default Action 1:

    - SIGABND: 异常终止
    - SIGABRT: 异常终止（由abort（）发送）
    - SIGALRM: 超时信号（通过alarm（）发送）
    - SIGBUS: 总线错误（仅在MVS™5.2或更高版本上运行时可用）
    - SIGFPE: 为屏蔽的算术异常，例如: 溢出，除0，错误的操作
    - SIGHUP: 终端挂起或进程结束
    - SIGILL: Detection of an incorrect function image.
    - SIGINT: Interactive attention.
    - SIGKILL: 无法捕获或忽略的终端信号
    - SIGPIPE: 写入未被读取的管道
    - SIGPOLL: 发生可调用事件（仅在MVS 5.2或更高版本上运行时可用）
    - SIGQUIT: 终端的退出信号
    - SIGSEGV: 对内存的访问不正确
    - SIGSYS: 发出错误的系统调用（仅在MVS 5.2或更高版本上运行时可用）
    - SIGTERM: 发送给程序的终端信号
    - SIGTRAP： 内部用于DBX或PTRACE
    - SIGUSR1: 用于用户应用程序使用
    - SIGUSR2: 用于用户应用程序使用
    - SIGVTALRM: 虚拟定时器已过期（仅在MVS 5.2或更高版本上运行时可用）
    - SIGXCPU: 超出CPU时间限制（仅在MVS 5.2或更高版本上运行时可用）。如果捕获或忽略Sigxcpu的CPU时间和SIGXCPU的进程，则会生成一个sigkill。
    - SIGXFSZ: 超出文件大小限制

- Default Action 2:

    - SIGURG: 套接字可用高带宽数据（仅在MVS 5.2或更高版本上运行时可用）
    - SIGCHLD: 结束或停止的子进程（SIGCLD是此信号的别名）
    - SIGIO: 完成输入或输出
    - SIGIOERR: 检测到严重的I/O错误
    - SIGWINCH: 窗口大小已更改（仅在MVS 5.2或更高版本上运行时可用）

- Default Action 3:

    - SIGSTOP: 停止信号无法捕获或忽略
    - SIGTSTP: 终端的停止信号
    - SIGTTIN: 尝试从控制终端读取的后台进程
    - SIGTTOU: 尝试写入控制终端的后台进程

- Default Action 4:

    - SIGCONT: 如果停止，请继续。


`const struct sigaction *__restrict__ new`

如果new是一个NULL，sigaction() 仅确定为处理当前的sig的操作，如果不为NULL, 它应该指定为sigaction结构体，此结构体中指定的操作替代原来的sig对应的操作。

`struct sigaction *__restrict__ old`

old可以是一个NULL，如果不是NULL，则代表 sig 原来的 struct 的内存位置。

### sigaction structure

```c
struct sigaction {
   void       (*sa_handler)(int);
   sigset_t   sa_mask;
   int        sa_flags;
   void       (*sa_sigaction)(int, siginfo_t *, void *);
};
```

`void (*)(int) sa_handler`

指向被分配来处理信号的函数的指针。 这个成员的值也可以是SIG_DFL（表示默认动作）或SIG_IGN（表示要忽略该信号）。

XPG4.2的特殊行为。 这个成员和sa_sigaction是相互排斥的。当SA_flags中设置了SA_SIGINFO标志时，就会使用sa_sigaction。否则，将使用sa_handler。

`sigset_t sa_mask`

在调用信号处理函数sa_handler或sa_sigaction（在XPG4.2中）之前，一个信号集标识了一组信号，这些信号将被添加到调用进程的信号屏蔽中。

你不能使用这种机制来阻止SIGKILL、SIGSTOP或SIGTRACE。如果sa_mask包括这些信号，它们将被简单地忽略；sigaction()将不会返回错误。

sa_mask必须通过使用一个或多个信号集操作函数来设置：sigemptyset(), sigfillset(), sigaddset(), 或sigdelset()


`int sa_flags`

一个影响信号行为的标志位的集合。

`void (*)(int, siginfo_t *, void *) sa_sigaction`

指向分配给处理信号的函数的指针，或SIG_DFL，或SIG_IGN。

这个函数将通过三个参数被调用。

第一个是int类型，包含这个函数被调用的信号类型。

第二个参数是指向siginfo_t的指针，siginfo_t包含关于信号源的额外信息。

第三个是 "指向void的指针"，但实际上将指向一个ucontext_t，包含信号中断时的上下文信息。











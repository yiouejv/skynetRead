

上一小节主要是梳理了一下 `skynet_context_new()` 的功能。

`skynet_context_new()` 主要功能是创建一个 `skynet_context`。

根据传入的name 从 配置的 cpath 路径对应的 `*.so` 库中查找。

我们写的 C 服务需要包含四个函数, 假如 name 为 `logger`, 则这四个函数的函数名分别为

- logger_create
- logger_release
- logger_init
- logger_signal

`skynet_context` 结构体包含以下成员:

```c
struct skynet_context {
    void * instance;                // 创建出的服务实例
    struct skynet_module * mod;     // 内存里该C服务句柄
    void * cb_ud;                   
    skynet_cb cb;
    struct message_queue *queue;    // 该服务的消息队列
    ATOM_POINTER logfile;
    uint64_t cpu_cost;  // in microsec
    uint64_t cpu_start; // in microsec
    char result[32];                // 暂存结果
    uint32_t handle;                // 启动该服务后，服务号
    int session_id;
    ATOM_INT ref;
    int message_count;             
    bool init;                      // init 之后为false
    bool endless;                   
    bool profile;                   // 配置文件 profile 的值

    CHECKCALLING_DECL
};
```

上述成员变量没注释的从main看到这里，暂时不清楚是做什么的，往后继续阅读就清楚了。

今天我们接着 `skynet_start()` 继续看, 完整的函数体在下面。

```c
void 
skynet_start(struct skynet_config * config) {
    // register SIGHUP for log file reopen
    struct sigaction sa;
    sa.sa_handler = &handle_hup;
    sa.sa_flags = SA_RESTART;
    sigfillset(&sa.sa_mask);
    sigaction(SIGHUP, &sa, NULL);

    if (config->daemon) {
        if (daemon_init(config->daemon)) {
            exit(1);
        }
    }
    skynet_harbor_init(config->harbor);
    skynet_handle_init(config->harbor);
    skynet_mq_init();
    skynet_module_init(config->module_path);
    skynet_timer_init();
    skynet_socket_init();
    skynet_profile_enable(config->profile);

    struct skynet_context *ctx = skynet_context_new(config->logservice, config->logger);
    if (ctx == NULL) {
        fprintf(stderr, "Can't launch %s service\n", config->logservice);
        exit(1);
    }

    skynet_handle_namehandle(skynet_context_handle(ctx), "logger");

    bootstrap(ctx, config->bootstrap);

    start(config->thread);

    // harbor_exit may call socket send, so it should exit before socket_free
    skynet_harbor_exit();
    skynet_socket_free();
    if (config->daemon) {
        daemon_exit(config->daemon);
    }
}
```

只剩下下面这点没看了。

```c
skynet_start(struct skynet_config * config) {
    // ...
    skynet_handle_namehandle(skynet_context_handle(ctx), "logger");

    bootstrap(ctx, config->bootstrap);

    start(config->thread);

    // harbor_exit may call socket send, so it should exit before socket_free
    skynet_harbor_exit();
    skynet_socket_free();
    if (config->daemon) {
        daemon_exit(config->daemon);
    }
}
```

-------------------------------------------

<strong>skynet_handle_namehandle(skynet_context_handle(ctx), "logger");</strong>

```c
const char * 
skynet_handle_namehandle(uint32_t handle, const char *name) {
    rwlock_wlock(&H->lock);

    const char * ret = _insert_name(H, name, handle);

    rwlock_wunlock(&H->lock);

    return ret;
}
```

我们可以看到，`skynet_handle_namehandle()` 的功能是 `_insert_name()`。

看看 `_insert_name()` 的具体实现：

```c
static const char *
_insert_name(struct handle_storage *s, const char * name, uint32_t handle) {
    int begin = 0;
    int end = s->name_count - 1;
    while (begin<=end) {
        int mid = (begin+end)/2;
        struct handle_name *n = &s->name[mid];
        int c = strcmp(n->name, name);
        if (c==0) {
            return NULL;
        }
        if (c<0) {
            begin = mid + 1;
        } else {
            end = mid - 1;
        }
    }
    char * result = skynet_strdup(name);

    _insert_name_before(s, result, handle, begin);

    return result;
}
```

很明显的一个二分查找，判断传入的 name 是否在 `H->name`里, 这是一个指针，指向一块内存，其内存元素的类型为 `skynet_name`, 定义如下:

```c
struct handle_name {
    char * name;
    uint32_t handle;
};

struct handle_storage {
    //...
    int name_cap;       // *name 指向内存块的容量
    int name_count;     // *name 指向内存块当前已经占用了几个元素，每个元素为一个 skynet_name
    struct handle_name *name;   // 指向一块内存
};
```

**为什么这里可以使用二分查找？**

我们看下 `strcmp()` 函数的功能

> 百度百科：strcmp函数是string compare(字符串比较)的缩写，用于比较两个字符串并根据比较结果返回整数。基本形式为strcmp(str1,str2)，若str1=str2，则返回零；若str1<str2，则返回负数；若str1>str2，则返回正数。

我们可以依据返回的结果判断，如果为0说明找到了，直接返回，后面的逻辑就不跑了，如果`<0`，说明结果在mid的右边，`begin = mid + 1`, 反之 `end = mid - 1`。

既然可以使用了二分查找，那一定要保证数组的有序咯？

我们接着往下看，

`char * result = skynet_strdup(name);` 将 name 拷贝了一份, 为什么要拷贝，下面解释。

`_insert_name_before(s, result, handle, begin);` 函数原型如下:

```c
static void
_insert_name_before(struct handle_storage *s, char *name, uint32_t handle, int before) {
    if (s->name_count >= s->name_cap) {
        s->name_cap *= 2;
        assert(s->name_cap <= MAX_SLOT_SIZE);
        struct handle_name * n = skynet_malloc(s->name_cap * sizeof(struct handle_name));
        int i;
        for (i=0;i<before;i++) {
            n[i] = s->name[i];
        }
        for (i=before;i<s->name_count;i++) {
            n[i+1] = s->name[i];
        }
        skynet_free(s->name);
        s->name = n;
    } else {
        int i;
        for (i=s->name_count;i>before;i--) {
            s->name[i] = s->name[i-1];
        }
    }
    s->name[before].name = name;
    s->name[before].handle = handle;
    s->name_count ++;
}
```

首先我们看 `else` 分支，此时 `H->name` 指向的内存还没放满元素，将从 before 下标开始到之后一个元素，将数组统一向后挪一个位置，不过都是从末端开始，到before 截至。

before 也就是传入函数的begin, 是 name 将要插入的位置，将下标比begin 大的元素往后移，将 name 插入begin的位置，从而保证了数组的有序。

如果 `if (s->name_count >= s->name_cap)` 如果容量不够了，将容量扩充至原来的两倍，并且有一个 `MAX_SLOT_SIZE`最大值内存上限，将 before 之前的元素重新写入内存，之后的元素向后挪一个位置，同时释放掉原内存空间，中间 before 的位置写入 name。

注意这里的释放操作，也就是为什么 `_insert_name()`里为什么要拷贝一份的原因了。

-----------------------------------------------

接下来是 <strong>bootstrap(ctx, config->bootstrap);</strong>

```c
static void
bootstrap(struct skynet_context * logger, const char * cmdline) {
    int sz = strlen(cmdline);
    char name[sz+1];
    char args[sz+1];
    int arg_pos;
    sscanf(cmdline, "%s", name);
    arg_pos = strlen(name);
    if (arg_pos < sz) {
        while(cmdline[arg_pos] == ' ') {
            arg_pos++;
        }
        strncpy(args, cmdline + arg_pos, sz);
    } else {
        args[0] = '\0';
    }
    struct skynet_context *ctx = skynet_context_new(name, args);
    if (ctx == NULL) {
        skynet_error(NULL, "Bootstrap error : %s\n", cmdline);
        skynet_context_dispatchall(logger);
        exit(1);
    }
}
```

我们拿 `example/config` 下的配置举例，config 下的 bootstrap 配置如下。

```lua
bootstrap = "snlua bootstrap"   -- The service for bootstrap
```

`bootstrap()` 函数体内在调用 `skynet_context_new()` 之前，主要是把 "snlua" 赋值给 name，而空格之后的字符串即为args的值。

然后再把 name，args传入 skynet_context_new() 函数，从 snlua.so 中查找模块并创建实例。

如果失败了，把消息队列里的消息分发完，退出。


------------------------------------------------------

<strong>start(config->thread);</strong>

```c
static void
start(int thread) {
    pthread_t pid[thread+3];

    struct monitor *m = skynet_malloc(sizeof(*m));
    memset(m, 0, sizeof(*m));
    m->count = thread;
    m->sleep = 0;

    m->m = skynet_malloc(thread * sizeof(struct skynet_monitor *));
    int i;
    for (i=0;i<thread;i++) {
        m->m[i] = skynet_monitor_new();
    }
    if (pthread_mutex_init(&m->mutex, NULL)) {
        fprintf(stderr, "Init mutex error");
        exit(1);
    }
    if (pthread_cond_init(&m->cond, NULL)) {
        fprintf(stderr, "Init cond error");
        exit(1);
    }

    create_thread(&pid[0], thread_monitor, m);
    create_thread(&pid[1], thread_timer, m);
    create_thread(&pid[2], thread_socket, m);

    static int weight[] = { 
        -1, -1, -1, -1, 0, 0, 0, 0,
        1, 1, 1, 1, 1, 1, 1, 1, 
        2, 2, 2, 2, 2, 2, 2, 2, 
        3, 3, 3, 3, 3, 3, 3, 3, };
    struct worker_parm wp[thread];
    for (i=0;i<thread;i++) {
        wp[i].m = m;
        wp[i].id = i;
        if (i < sizeof(weight)/sizeof(weight[0])) {
            wp[i].weight= weight[i];
        } else {
            wp[i].weight = 0;
        }
        create_thread(&pid[i+3], thread_worker, &wp[i]);
    }

    for (i=0;i<thread+3;i++) {
        pthread_join(pid[i], NULL); 
    }

    free_monitor(m);
}
```

声明了一个线程数组，数组大小为 `config->thread + 3`。

```c
struct monitor {
    int count;
    struct skynet_monitor ** m;
    pthread_cond_t cond;
    pthread_mutex_t mutex;
    int sleep;
    int quit;
};
```

创建了一个 monitor 结构体, 初始化数据

- count: config->thread
- sleep: 0
- m: 为每个thread 创建了一个 skynet_monitor
- cond: 条件变量
- mutex: 互斥锁

```c
create_thread(&pid[0], thread_monitor, m);
create_thread(&pid[1], thread_timer, m);
create_thread(&pid[2], thread_socket, m);
```

创建了三个线程，monitor, timer, socket, 这就是声明数组大小为 thread + 3 的原因。

`create_thread()` 的函数原型如下:

```c
static void
create_thread(pthread_t *thread, void *(*start_routine) (void *), void *arg) {
    if (pthread_create(thread,NULL, start_routine, arg)) {
        fprintf(stderr, "Create thread failed");
        exit(1);
    }
}
```

参数

- thread: 指向线程的指针
- start_routine: 指向函数的指针
- arg: void 指针

创建线程，并指明线程运行函数的起始地址，也就是我们传入的函数指针指向的地址，arg 将最为参数传递给 start_routine 函数。

----------------------------------------------------
```c
static int weight[] = { 
        -1, -1, -1, -1, 0, 0, 0, 0,
        1, 1, 1, 1, 1, 1, 1, 1, 
        2, 2, 2, 2, 2, 2, 2, 2, 
        3, 3, 3, 3, 3, 3, 3, 3, };
struct worker_parm wp[thread];
for (i=0;i<thread;i++) {
    wp[i].m = m;
    wp[i].id = i;
    if (i < sizeof(weight)/sizeof(weight[0])) {
        wp[i].weight= weight[i];
    } else {
        wp[i].weight = 0;
    }
    create_thread(&pid[i+3], thread_worker, &wp[i]);
}

for (i=0;i<thread+3;i++) {
    pthread_join(pid[i], NULL); 
}

free_monitor(m);
```

创建剩下的thread个线程，并且传入 worker_parm 结构体作为参数，结构体包含

```c
struct worker_parm {
    struct monitor *m;  // 指向 skynet_monitor 类型的指针
    int id;             
    int weight;         // 权重，后面有用
};
```

最后，join，free。

`start()` 的功能就是为整个框架创建好线程，线程数量由 config 文件的 thread 参数配置，还包括框架本身需要的3个线程，monitor, timer, socket。

----------------------------------------

继续回到 skynet_start() 函数。

```c
// harbor_exit may call socket send, so it should exit before socket_free
skynet_harbor_exit();
skynet_socket_free();
if (config->daemon) {
    daemon_exit(config->daemon);
}
```

最后join 调线程之后再把 harbor，socket，等相关资源释放掉。

`skynet_start()` 到这里就结束了，是不是很奇怪，是不是感觉还要很多事情没有做（如果有skynet使用经验会有这种感觉），别急，后面我们继续看。
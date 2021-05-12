
skynet 是C语言写的框架，我们采用学习过程中最基本的方式去阅读skynet，从C语言的main函数开始。

首先我们找到框架的入口`main`函数，在 skynet/skynet-src/skynet_main.c 文件内。

main函数的代码如下:

```c
int
main(int argc, char *argv[]) {
    const char * config_file = NULL ;
    if (argc > 1) {
        config_file = argv[1];
    } else {
        fprintf(stderr, "Need a config file. Please read skynet wiki : https://github.com/cloudwu/skynet/wiki/Config\n"            "usage: skynet configfilename\n");
        return 1;
    }

    skynet_globalinit();
    skynet_env_init();

    sigign();

    struct skynet_config config;

#ifdef LUA_CACHELIB
    // init the lock of code cache
    luaL_initcodecache();
#endif

    struct lua_State *L = luaL_newstate();
    luaL_openlibs(L);   // link lua lib

    int err =  luaL_loadbufferx(L, load_config, strlen(load_config), "=[skynet config]", "t");
    assert(err == LUA_OK);
    lua_pushstring(L, config_file);

    err = lua_pcall(L, 1, 1, 0);
    if (err) {
        fprintf(stderr,"%s\n",lua_tostring(L,-1));
        lua_close(L);
        return 1;
    }
    _init_env(L);

    config.thread =  optint("thread",8);
    config.module_path = optstring("cpath","./cservice/?.so");
    config.harbor = optint("harbor", 1);
    config.bootstrap = optstring("bootstrap","snlua bootstrap");
    config.daemon = optstring("daemon", NULL);
    config.logger = optstring("logger", NULL);
    config.logservice = optstring("logservice", "logger");
    config.profile = optboolean("profile", 1);

    lua_close(L);

    skynet_start(&config);
    skynet_globalexit();

    return 0;
}
```

我们一段一段查看

-------------------------------
```c
int
main(int argc, char *argv[]) {
    const char * config_file = NULL ;
    if (argc > 1) {
        config_file = argv[1];
    } else {
        fprintf(stderr, "Need a config file. Please read skynet wiki : https://github.com/cloudwu/skynet/wiki/Config\n"
            "usage: skynet configfilename\n");
        return 1;
    }

    //...
}
```

定义了一个指针, 指针指向常量, `const char* config_file`, `config_file` 赋值为启动时的第二个参数，也就是配置文件的路径。

-------------------------------

<strong>skynet_globalinit();</strong>

```c
// skynet/skynet-src/skynet_server.c
struct skynet_node {
    ATOM_INT total;
    int init;
    uint32_t monitor_exit;
    pthread_key_t handle_key;
    bool profile;   // default is off
};
static struct skynet_node G_NODE;

void 
skynet_globalinit(void) {
    ATOM_INIT(&G_NODE.total , 0);
    G_NODE.monitor_exit = 0;
    G_NODE.init = 1;
    if (pthread_key_create(&G_NODE.handle_key, NULL)) {
        fprintf(stderr, "pthread_key_create failed");
        exit(1);
    }
    // skynet/skynet-src/skynet_imp.h
    /*
        #define THREAD_WORKER 0
        #define THREAD_MAIN 1
        #define THREAD_SOCKET 2
        #define THREAD_TIMER 3
        #define THREAD_MONITOR 4
    */
    skynet_initthread(THREAD_MAIN);
}

skynet_initthread(int m) {
    // skynet/skynet-src/atomic.h
    // #define ATOM_POINTER volatile uintptr_t
    uintptr_t v = (uint32_t)(-m);
    pthread_setspecific(G_NODE.handle_key, (void *)v);
}
```

初始化全局节点信息，total 为0，monitor_exit 为0，init 1，

`pthread_key_create(&G_NODE.handle_key, NULL)` 创建了一个多线程私有数据 handle_key，可参考文章: https://www.jianshu.com/p/d78d93d46fc2

`skynet_initthread(THREAD_MAIN);` 将当前线程状态由 THREAD_MAIN 切换为 THREAD_WORKER 状态并记录在handle_key。

-------------------------------

<strong>skynet_env_init();</strong>


```c
// skynet/skynet-src/skynet_env.c
struct skynet_env {
    struct spinlock lock;
    lua_State *L;
};

static struct skynet_env *E = NULL;

void
skynet_env_init() {
    E = skynet_malloc(sizeof(*E));
    SPIN_INIT(E)
    E->L = luaL_newstate();
}
```

`E` 一个`skynet_env`结构体，结构体内包含一个 `spinlock` 自旋锁，一个lua虚拟机指针。

`skynet_malloc` 为结构体E分配内存，`skynet_malloc`内部暂时不细究。

`SPIN_INIT(E)`

通过查找代码得知, 这是在 skynet/skynet-src/spinlick.h 中定义的一个宏。   
`#define SPIN_INIT(q) spinlock_init(&(q)->lock);`   
对E中的lock 进行初始化。

`E->L = luaL_newstate();` L绑定了一个lua虚拟机。

-------------------------------
<strong>sigign();</strong>

```c
#include <signal.h>

int sigign() {
    struct sigaction sa;
    sa.sa_handler = SIG_IGN;
    sa.sa_flags = 0;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGPIPE, &sa, 0);
    return 0;
}
```

main 函数同文件下的 sigign() 函数。

定义了一个 sigaction 结构体，将 sa_handler 设置为 SIG_IGN，表示要忽略信号的产生的动作。

`sigaction(SIGPIPE, &sa, 0);` 将 SIGPIPE的行为替换为 sa 结构体定义的形式，表示当前进程忽略 SIGPIPE 信号。

这里简单记录了一下 sigaction 的资料。 <a href="./01ext_sigaction.md">01ext_sigaction</a>

-------------------------------

<strong>struct skynet_config config;</strong>

定义了结构体 config

```c
struct skynet_config {
    int thread;
    int harbor;
    int profile;
    const char * daemon;
    const char * module_path;
    const char * bootstrap;
    const char * logger;
    const char * logservice;
};
```

-------------------------------

<strong>luaL_initcodecache();</strong>

```c
// skynet/skynet-src/skynet_main.c
#ifdef LUA_CACHELIB
    luaL_initcodecache();
#endif

// skynet/3rd/lauxlib.c
static struct codecache CC;
struct codecache {
    struct spinlock lock;
    lua_State *L;
};
LUALIB_API void
luaL_initcodecache(void) {
    SPIN_INIT(&CC);
}
```

-------------------------------

```c
static const char * load_config = "\
    local result = {}\n\
    local function getenv(name) return assert(os.getenv(name), [[os.getenv() failed: ]] .. name) end\n\
    local sep = package.config:sub(1,1)\n\
    local current_path = [[.]]..sep\n\
    local function include(filename)\n\
        local last_path = current_path\n\
        local path, name = filename:match([[(.*]]..sep..[[)(.*)$]])\n\
        if path then\n\
            if path:sub(1,1) == sep then    -- root\n\
                current_path = path\n\
            else\n\
                current_path = current_path .. path\n\
            end\n\
        else\n\
            name = filename\n\
        end\n\
        local f = assert(io.open(current_path .. name))\n\
        local code = assert(f:read [[*a]])\n\
        code = string.gsub(code, [[%$([%w_%d]+)]], getenv)\n\
        f:close()\n\
        assert(load(code,[[@]]..filename,[[t]],result))()\n\
        current_path = last_path\n\
    end\n\
    setmetatable(result, { __index = { include = include } })\n\
    local config_name = ...\n\
    include(config_name)\n\
    setmetatable(result, nil)\n\
    return result\n\
";

struct lua_State *L = luaL_newstate();
luaL_openlibs(L);   // link lua lib

int err =  luaL_loadbufferx(L, load_config, strlen(load_config), "=[skynet config]", "t");
assert(err == LUA_OK);
lua_pushstring(L, config_file);

err = lua_pcall(L, 1, 1, 0);
if (err) {
    fprintf(stderr,"%s\n",lua_tostring(L,-1));
    lua_close(L);
    return 1;
}
```

`luaL_loadbufferx(L, load_config, strlen(load_config), "=[skynet config]", "t");`

加载了一段lua代码到内存里，并压入lua栈内。 

load_config 这段代码实现的功能: 将配置文件内的 `$var` 替换成了环境变量的内容, 并返回了一个result表。


`lua_pcall(L, 1, 1, 0);`

执行压入的 load_config 代码块，第二个参数1 表示压入的栈的个数为1，`lua_pushstring(L, config_file);` 被压栈的配置文件名。 执行完函数之后，函数和参数自动出栈，此时栈为空。 函数的返回值被压栈，此时栈内只有一个表 result, result 内包含了配置在 config_file 内的键值对。

-------------------------------
<strong>\_init_env(L);</strong>

```c
static void
_init_env(lua_State *L) {
    lua_pushnil(L);  /* first key */
    while (lua_next(L, -2) != 0) {
        int keyt = lua_type(L, -2);
        if (keyt != LUA_TSTRING) {
            fprintf(stderr, "Invalid config table\n");
            exit(1);
        }
        const char * key = lua_tostring(L,-2);
        if (lua_type(L,-1) == LUA_TBOOLEAN) {
            int b = lua_toboolean(L,-1);
            skynet_setenv(key,b ? "true" : "false" );
        } else {
            const char * value = lua_tostring(L,-1);
            if (value == NULL) {
                fprintf(stderr, "Invalid config table key = %s\n", key);
                exit(1);
            }
            skynet_setenv(key,value);
        }
        lua_pop(L,1);
    }
    lua_pop(L,1);
}

// skynet/skynet-src/skynet_env.c
void 
skynet_setenv(const char *key, const char *value) {
    SPIN_LOCK(E)
    
    lua_State *L = E->L;
    lua_getglobal(L, key);
    assert(lua_isnil(L, -1));
    lua_pop(L,1);
    lua_pushstring(L,value);
    lua_setglobal(L,key);

    SPIN_UNLOCK(E)
}

// 从堆栈上弹出一个值，并将其设为全局变量 name 的新值。
void lua_setglobal (lua_State *L, const char *name);

// 把全局变量 name 里的值压栈，返回该值的类型。
int lua_getglobal (lua_State *L, const char *name);
```


将lua栈表内的键值对设置到 &E->L 的全局环境中。 

------------------------------------------------------

```c
config.thread =  optint("thread",8);
config.module_path = optstring("cpath","./cservice/?.so");
config.harbor = optint("harbor", 1);
config.bootstrap = optstring("bootstrap","snlua bootstrap");
config.daemon = optstring("daemon", NULL);
config.logger = optstring("logger", NULL);
config.logservice = optstring("logservice", "logger");
config.profile = optboolean("profile", 1);

static int
optint(const char *key, int opt) {
    const char * str = skynet_getenv(key);
    if (str == NULL) {
        char tmp[20];
        sprintf(tmp,"%d",opt);
        skynet_setenv(key, tmp);
        return opt;
    }
    return strtol(str, NULL, 10);
}

// skynet/skynet-src/skynet_env.c
const char * 
skynet_getenv(const char *key) {
    SPIN_LOCK(E)

    lua_State *L = E->L;
    
    lua_getglobal(L, key);
    const char * result = lua_tostring(L, -1);
    lua_pop(L, 1);

    SPIN_UNLOCK(E)

    return result;
}
```

optint, optstring, optboolean 从 &E->L 的全局环境中取得对应键的值，如果全局环境内未定义，则第二个参数 opt 设为 key的默认值。

--------------------------

<strong>lua_close(L);</strong>

关闭main函数内创建的 lua 虚拟机。

------------------------------
<strong>skynet_start(&config);</strong>

下一节的内容。

------------------------------

<strong>skynet_globalexit();</strong>

```c
void 
skynet_globalexit(void) {
    pthread_key_delete(G_NODE.handle_key);
}
```

删除在 skynet_initthread 中定义的特殊的线程数据。


























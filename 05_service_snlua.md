
到这里我们总结一下 skynet_start() 函数的功能

- 重定向了 SIGHUP 信号的handle, 将 `static volatile int SIG = 0;` 设置为 1.
- 判断当前skynet 配置是否已经启动过了，启动过并且配置了 daemon 则不会二次被启动。
- 将标准输入，标准输出，标准错误重定向到了`/dev/null`，表示抛弃这三种文件描述符的结果。
- 初始化harbor 的值，根据 config->harbor << 24 位。
- 初始化 `handle_storage* H`。
- 初始化全局消息队列。
- 初始化 `modules *m`, 把 config->cpath 赋值给 m->path
- 初始化 timer 模块，socket 模块，这两个模块后面出专题细看。
- 初始化 skynet_node G_NODE, `G_Node.profile = config->profile`
- 启动了一个 logger C服务，主要是用来打印日志。
- 启动配置文件中指定的 bootstrap C服务
- 创建线程，其中包括 monitor, timer, socket 线程。 创建的线程数为 config 中指定的线程数 加3
- 剩下的就是等待进程结束了。

本节我们来看看 snlua C服务。

skynet_context_new() 函数会创建一个C服务，首先我们看 create 函数

```c
struct snlua *
snlua_create(void) {
	struct snlua * l = skynet_malloc(sizeof(*l));
	memset(l,0,sizeof(*l));
	l->mem_report = MEMORY_WARNING_REPORT;
	l->mem_limit = 0;
	l->L = lua_newstate(lalloc, l);
	l->activeL = NULL;
	ATOM_INIT(&l->trap , 0);
	return l;
}
```

声明了一个 snlua 结构体类型的指针 l, 结构体定义如下：

```c
struct snlua {
	lua_State * L;					// lua 虚拟机
	struct skynet_context * ctx;	// skynet_context 指针
	size_t mem;						// 下面三个应该和内存管理有关
	size_t mem_report;
	size_t mem_limit;
	lua_State * activeL;			// 激活的lua虚拟机
	ATOM_INT trap;					
};
```

```c
#define MEMORY_WARNING_REPORT (1024 * 1024 * 32)
l->mem_report = MEMORY_WARNING_REPORT;
```






并且调用 init 函数，所有我们先看 snlua_init 函数。

原型如下:

```c
int
snlua_init(struct snlua *l, struct skynet_context *ctx, const char * args) {
	int sz = strlen(args);
	char * tmp = skynet_malloc(sz);
	memcpy(tmp, args, sz);
	skynet_callback(ctx, l , launch_cb);
	const char * self = skynet_command(ctx, "REG", NULL);
	uint32_t handle_id = strtoul(self+1, NULL, 16);
	// it must be first message
	skynet_send(ctx, 0, handle_id, PTYPE_TAG_DONTCOPY,0, tmp, sz);
	return 0;
}
```

下面三行

```c
int sz = strlen(args);
char * tmp = skynet_malloc(sz);
memcpy(tmp, args, sz);
```

拷贝了args, 保存在 `char *tmp` 指向的内存空间。

```c
skynet_callback(ctx, l , launch_cb);
```

```c
void 
skynet_callback(struct skynet_context * context, void *ud, skynet_cb cb) {
	context->cb = cb;
	context->cb_ud = ud;
}

static int
launch_cb(struct skynet_context * context, void *ud, int type, int session, uint32_t source , const void * msg, size_t sz) {
	assert(type == 0 && session == 0);
	struct snlua *l = ud;
	skynet_callback(context, NULL, NULL);
	int err = init_cb(l, context, msg, sz);
	if (err) {
		skynet_command(context, "EXIT", NULL);
	}

	return 0;
}
```



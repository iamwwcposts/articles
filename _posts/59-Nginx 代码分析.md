---
issueid: 59
tags:
- 基础
- net
- C/C++
title: Nginx 代码分析
date: 2022-08-18
updated: 2024-04-23
---
工作中涉及Nginx module开发，分析Nginx/tengine代码

## nginx modules初始化

每个 module 必须有 ngx_module_t，且在module不同生命周期都支持 hook 函数

```c
struct ngx_module_s {
    ngx_uint_t            ctx_index;
    ngx_uint_t            index;

    char                 *name;

    ngx_uint_t            spare0;
    ngx_uint_t            spare1;

    ngx_uint_t            version;
    const char           *signature;

    void                 *ctx;
    ngx_command_t        *commands;
    ngx_uint_t            type;

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};
```

由于 ngx_module_t 为基础类型，对于http和stream，通过 ngx_module_t#ctx进行了扩展，用于放置 http/stream module相关的 hook 函数

比如http module中，ctx 为 ngx_http_module_t 类型

```c
typedef struct {
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

    void       *(*create_main_conf)(ngx_conf_t *cf);
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void       *(*create_srv_conf)(ngx_conf_t *cf);
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    void       *(*create_loc_conf)(ngx_conf_t *cf);
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
```

stream module 对 module ctx 扩展

```c
typedef struct {
    ngx_int_t                    (*preconfiguration)(ngx_conf_t *cf);
    ngx_int_t                    (*postconfiguration)(ngx_conf_t *cf);

    void                        *(*create_main_conf)(ngx_conf_t *cf);
    char                        *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void                        *(*create_srv_conf)(ngx_conf_t *cf);
    char                        *(*merge_srv_conf)(ngx_conf_t *cf, void *prev,
                                                   void *conf);
} ngx_stream_module_t;

```

nginx cycle 是最上层的大类，存放全部nginx运行上下文，其中
cycle->modules 存放全部的module，包含http_core modules，第三方http modules, event modules 等

```c
ngx_int_t
ngx_cycle_modules(ngx_cycle_t *cycle)
{
    /*
     * create a list of modules to be used for this cycle,
     * copy static modules to it
     */

    
    cycle->modules = ngx_pcalloc(cycle->pool, (ngx_max_module + 1)
                                              * sizeof(ngx_module_t *));
    if (cycle->modules == NULL) {
        return NGX_ERROR;
    }
    // 将全部的modules都放到cycle->modules中
    // ngx_modules 是编译后自动生成的文件，里面引用了全部modules
    ngx_memcpy(cycle->modules, ngx_modules,
               ngx_modules_n * sizeof(ngx_module_t *));

    cycle->modules_n = ngx_modules_n;

    return NGX_OK;
}
```

### 各个hook的调用时机

<http://nginx.org/en/docs/dev/development_guide.html#core_modules>

#### init_module

master 进程在每次配置初始化之后调用

#### init_process

master进程创建完worker后，worker中调用。所以init_process在worker的上下文中

#### exit_process

worker 退出时调用

#### exit_master

master退出时调用

## module自身配置的创建与解析module指令command

```c
// https://github.com/taikulawo/nginx-debugable/blob/8a30c4b12226c92cd317db614172cc6de2795768/src/core/ngx_cycle.c#L238

// 遍历全部的模块
for (i = 0; cycle->modules[i]; i++) {
    if (cycle->modules[i]->type != NGX_CORE_MODULE) {
        continue;
    }

    module = cycle->modules[i]->ctx;

    if (module->create_conf) {
        // init_cycle 阶段调用 module#create_conf 创建每个module初始状态的配置文件
        rv = module->create_conf(cycle);
        if (rv == NULL) {
            ngx_destroy_pool(pool);
            return NULL;
        }
        // cycle 下
        cycle->conf_ctx[cycle->modules[i]->index] = rv;
    }
}
```

**我发现 nginx 遍历全部modules的代码挺多的，为了实现某个功能，经常要遍历全部modules**

```c
// https://github.com/taikulawo/nginx-debugable/blob/8a30c4b12226c92cd317db614172cc6de2795768/src/core/ngx_conf_file.c#L447
// 在create_conf 已经创建好了 module conf，并存放到了 cf->ctx数组，下标是module的index
// 由于这是配置文件解析阶段，需要初始化每个module 的配置，所以调用module#ngx_command_t#set 方法，交由module处理每个directive（command）


for (i = 0; cf->cycle->modules[i]; i++) {
    // 遍历全部 modules

    cmd = cf->cycle->modules[i]->commands;
    if (cmd == NULL) {
        continue;
    }

    for ( /* void */ ; cmd->name.len; cmd++) {
        // 处理每个module的ngx_command_t[]

        if (name->len != cmd->name.len) {
            continue;
        }
        // 寻找name相同的command
        // 比如 http block 的command#name是 http
        // static ngx_command_t  ngx_http_commands[] = {

        //     { ngx_string("http"),
        //     NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
        //     ngx_http_block,
        //     0,
        //     0,
        //     NULL },

        //     ngx_null_command
        // };
        if (ngx_strcmp(name->data, cmd->name.data) != 0) {
            continue;
        }

        found = 1;

        if (cf->cycle->modules[i]->type != NGX_CONF_MODULE
            && cf->cycle->modules[i]->type != cf->module_type)
        {
            continue;
        }

        /* is the directive's location right ? */

        if (!(cmd->type & cf->cmd_type)) {
            continue;
        }

        if (!(cmd->type & NGX_CONF_BLOCK) && last != NGX_OK) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                "directive \"%s\" is not terminated by \";\"",
                                name->data);
            return NGX_ERROR;
        }

        if ((cmd->type & NGX_CONF_BLOCK) && last != NGX_CONF_BLOCK_START) {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                                "directive \"%s\" has no opening \"{\"",
                                name->data);
            return NGX_ERROR;
        }

        /* is the directive's argument count right ? */

        if (!(cmd->type & NGX_CONF_ANY)) {

            if (cmd->type & NGX_CONF_FLAG) {

                if (cf->args->nelts != 2) {
                    goto invalid;
                }
            // 指令要求至少一个参数
            } else if (cmd->type & NGX_CONF_1MORE) {

                if (cf->args->nelts < 2) {
                    goto invalid;
                }
            // 指令要求至少两个参数
            } else if (cmd->type & NGX_CONF_2MORE) {

                if (cf->args->nelts < 3) {
                    goto invalid;
                }

            } else if (cf->args->nelts > NGX_CONF_MAX_ARGS) {

                goto invalid;

            } else if (!(cmd->type & argument_number[cf->args->nelts - 1]))
            {
                goto invalid;
            }
        }

        /* set up the directive's configuration context */

        conf = NULL;

        if (cmd->type & NGX_DIRECT_CONF) {
            // 将conf赋为上文调用create_conf初始化好的数组地址
            conf = ((void **) cf->ctx)[cf->cycle->modules[i]->index];

        } else if (cmd->type & NGX_MAIN_CONF) {
            // 将conf赋为上文调用create_conf初始化好的数组地址
            conf = &(((void **) cf->ctx)[cf->cycle->modules[i]->index]);
        } else if (cf->ctx) {
            // 举个例子
            // 在 ngx_stream block解析过程中，cf->ctx 指向 ngx_stream_conf_ctx_t
            // 如果 cmd->conf 为 NGX_STREAM_SRV_CONF_OFFSET（8）
            // 则 (char *) cf->ctx + cmd->conf 计算后指向  ngx_stream_conf_ctx_t#srv_conf 字段.
            // 此种情况下，下面的 conf = confp[cf->cycle->modules[i]->ctx_index]; 计算为此module在stream module 的 srv 配置结构，也就是module#create_srv_conf hook 返回的结构
            confp = *(void **) ((char *) cf->ctx + cmd->conf);

            if (confp) {
                // 将conf赋为上文调用create_conf初始化好的数组地址
                conf = confp[cf->cycle->modules[i]->ctx_index];
            }
        }
        // 调用 set，module接管directive处理
        // 如果cmd->name == "http" && NGX_CONF_BLOCK
        // 则会走到
        // https://github.com/taikulawo/nginx-debugable/blob/1db517fb71aed6d6fffc8347086f89eb29b83dea/src/http/ngx_http.c#L122
        // http block 创建用于http的配置格式
        // 为什么 ngx_conf_t 里多是 void* p
        // 是因为不同的module要求的配置格式不尽相同，所以留给module自定义
        rv = cmd->set(cf, cmd, conf);

        if (rv == NGX_CONF_OK) {
            return NGX_OK;
        }

        if (rv == NGX_CONF_ERROR) {
            return NGX_ERROR;
        }

        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                            "\"%s\" directive %s", name->data, rv);

        return NGX_ERROR;
    }
}

```

conf 字段决定set函数的第三个参数传递什么，如果是 NGX_STREAM_SRV_CONF_OFFSET，则nginx会将此module的srv_conf 作为 `void *conf` 传入set 回调函数传递

我们拿一个set函数举例

```c
static char *
ngx_set_worker_processes(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_str_t        *value;
    ngx_core_conf_t  *ccf;

    // 转换为module自己的conf类型
    ccf = (ngx_core_conf_t *) conf;

    if (ccf->worker_processes != NGX_CONF_UNSET) {
        return "is duplicate";
    }

    value = cf->args->elts;

    // 设置配置
    if (ngx_strcmp(value[1].data, "auto") == 0) {
        ccf->worker_processes = ngx_ncpu;
        return NGX_CONF_OK;
    }

    ccf->worker_processes = ngx_atoi(value[1].data, value[1].len);

    if (ccf->worker_processes == NGX_ERROR) {
        return "invalid value";
    }

    return NGX_CONF_OK;
}
```

ngx_conf_t是在nginx启动后创建的变量，用于配置解析和调用module hook时记录当前配置解析的状态上下文。

```c
// https://github.com/willdeeper/tengine/blob/f9a582f73a2d6bd4fb73270cb6636e3490fc943f/src/core/ngx_conf_file.c#L761
if (found) {
    // ...
    // 配置解析过程中，会将指令后的Args存入 cf->args
    // 调用 module command[] 初始化command值时传入
    // char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    word = ngx_array_push(cf->args);
    
    if (word == NULL) {
        return NGX_ERROR;
    }

    word->data = ngx_pnalloc(cf->pool, b->pos - 1 - start + 1);
    if (word->data == NULL) {
        return NGX_ERROR;
    }
    // ...
}
```

![image](https://user-images.githubusercontent.com/24750337/186622825-8cc33160-bc89-4dc0-98ff-6307df27d683.png)

<http://nginx.org/en/docs/dev/development_guide.html#http_conf>

### main_conf vs srv_conf vs loc_conf

对于http 模块

main_conf: 应用于整个http block，属于module的全局http配置
srv_conf: 应用于单独一个server block，属于单独某个server块的配置
loc_conf: 应用于单独一个location，属于单独某个location块的配置

所以 main_conf 只有一个，而 srv_conf, loc_conf 都有多个。

所以 create_main_conf 只被调用一次， create_srv, create_loc 可被调用多次，取决于配置中有多少个 server，location block

对于 stream 模块

main_conf: 应用于整个stream block，属于module的全局stream配置
srv_conf: 应用于单独一个server block，属于单独某个server块的配置

所以 main_conf 只有一个，而 srv_conf都有多个。

所以 create_main_conf 只被调用一次， create_srv_conf 可被调用多次，取决于配置中有多少个 server block。

比如

```
stream {
    upstream foo {
        server 1;
        server 2;
    }
    server {
    
    }
    upstream foo {
        server 1;
        server 2;
    }
}
```

```c
static ngx_command_t ngx_stream_upstream_dynamic_command[] = {
    {ngx_string("consul"), NGX_STREAM_UPS_CONF | NGX_CONF_1MORE,
     ngx_stream_dynamic_upstream_set_consul_slot, 0, 0, NULL},
    ngx_null_command};

```

由于有 `NGX_STREAM_UPS_CONF` 标记，配置解析到foo，bar时，分别都会调用module的create_srv_conf

## 配置merge

<http://nginx.org/en/docs/dev/development_guide.html#http_conf>

以 `create_loc_conf` 为例，对于module有 create_conf和merge_conf的hook，nginx首先调用 create_conf，此时module创建一个空conf返回给nginx，在merge阶段nginx调用 merge_conf hook，将parent context conf传过来，module就可以拿到配置文件初始化

比如下面的合并server block，`cscfp[s]->ctx->srv_conf[ctx_index]` 是新创建的 conf，`saved.srv_conf[ctx_index]`是

```c
if (module->merge_srv_conf) {
    // https://developer.huawei.com/consumer/cn/forum/topic/0201187160465810231?fid=26
    // 这里主要是进行配置的合并，saved.srv_conf[ctx_index]指向的是http块中解析出的SRV类型的配置，
    // cscfp[s]->ctx->srv_conf[ctx_index]解析的则是当前server块中的配置，合并的一般规则是通过对应
    // 模块实现的merge_srv_conf()来进行的，**决定使用http块中的配置，还是使用server块中的配置**，
    // 或者说合并两者，则是由具体的实现来决定的。这里合并的效果本质上就是对
    // cscfp[s]->ctx->srv_conf[ctx_index]中的属性赋值，如果该值不存在，则将http块对应的值写到该属性中
    
    // 指令合并策略并没有强制要求，否则NGINX不会暴露merge_srv_conf的hook
    // 但对于树形结构的配置，指令出现在parent语义上要对child生效，除非child自身也有指令
    // 比如access_log，可以在http_main_block, server block 出现
    // 合并策略是：如果child此项配置为默认值，且parent有配置，child就需要使用parent的配置
    // 这也是大多数配置项合并代码都长得差不多的原因
    // /foo 开启
    // /bar 由于没有access_log配置，被parent merge，所以关闭
    // server {
    //    access_log off;
    //    location /foo {
    //        access_log /data/access.log debug;
    //    }
    //    location /bar {
    // 
    //    }
    //}
    
    // 为什么 http main block也有loc_conf，毕竟main block和 loc_conf 还隔了一个 srv_conf?
    // 还拿上面的access_log举例，/bar 没有access_log指令，但模块hook早已经注入nginx，所以/bar的 loc module 也需要有 access_log 的 loc_conf，这个配置是从http block parent merge到 loc 的loc_conf的。
    // 一句话：parent的配置不会真正用到，只是给配置初始化阶段child准备
    
    // call ngx_http_core_merge_srv_conf
    rv = module->merge_srv_conf(cf, saved.srv_conf[ctx_index],
                                cscfp[s]->ctx->srv_conf[ctx_index]);
    if (rv != NGX_CONF_OK) {
        goto failed;
    }
}
```

对于http block，最关键的是

```c
typedef struct {
    void        **main_conf;
    void        **srv_conf;
    void        **loc_conf;
} ngx_http_conf_ctx_t;


typedef struct {
    /* array of the ngx_http_server_name_t, "server_name" directive */
    ngx_array_t                 server_names;

    /* server ctx */
    // 每个server块持有 ngx_http_conf_ctx_t 结构，承上启下，server块记录属于哪个http，由记录其下全部的loc_conf
    ngx_http_conf_ctx_t        *ctx;

    u_char                     *file_name;
    ngx_uint_t                  line;

    ngx_str_t                   server_name;

    size_t                      connection_pool_size;
    size_t                      request_pool_size;
    size_t                      client_header_buffer_size;

    ngx_bufs_t                  large_client_header_buffers;

    ngx_msec_t                  client_header_timeout;

    ngx_flag_t                  ignore_invalid_headers;
    ngx_flag_t                  merge_slashes;
    ngx_flag_t                  underscores_in_headers;

    unsigned                    listen:1;
#if (NGX_PCRE)
    unsigned                    captures:1;
#endif

    ngx_http_core_loc_conf_t  **named_locations;
} ngx_http_core_srv_conf_t;
```

http block 中开始初始化 http module，

```c
// https://github.com/taikulawo/nginx-debugable/blob/1db517fb71aed6d6fffc8347086f89eb29b83dea/src/http/ngx_http.c#L154

/* the http main_conf context, it is the same in the all http contexts */

// 初始化 main_conf
ctx->main_conf = ngx_pcalloc(cf->pool,
                                sizeof(void *) * ngx_http_max_module);
if (ctx->main_conf == NULL) {
    return NGX_CONF_ERROR;
}


/*
    * the http null srv_conf context, it is used to merge
    * the server{}s' srv_conf's
    */
// 初始化 srv_conf
ctx->srv_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
if (ctx->srv_conf == NULL) {
    return NGX_CONF_ERROR;
}


/*
    * the http null loc_conf context, it is used to merge
    * the server{}s' loc_conf's
    */
// 初始化 loc_conf
ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
if (ctx->loc_conf == NULL) {
    return NGX_CONF_ERROR;
}


/*
    * create the main_conf's, the null srv_conf's, and the null loc_conf's
    * of the all http modules
    */

for (m = 0; cf->cycle->modules[m]; m++) {
    // 只处理 http module
    if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
        continue;
    }

    // 从cycle中取出 ngx_module_t#ctx void* p 指针
    module = cf->cycle->modules[m]->ctx;
    mi = cf->cycle->modules[m]->ctx_index;

    // 调用 http module hook
    if (module->create_main_conf) {
        ctx->main_conf[mi] = module->create_main_conf(cf);
        if (ctx->main_conf[mi] == NULL) {
            return NGX_CONF_ERROR;
        }
    }
    // 调用 http module hook
    if (module->create_srv_conf) {
        ctx->srv_conf[mi] = module->create_srv_conf(cf);
        if (ctx->srv_conf[mi] == NULL) {
            return NGX_CONF_ERROR;
        }
    }
    // 调用 http module hook
    if (module->create_loc_conf) {
        ctx->loc_conf[mi] = module->create_loc_conf(cf);
        if (ctx->loc_conf[mi] == NULL) {
            return NGX_CONF_ERROR;
        }
    }
}

// 注意这里，pcf 不是pointer，是clone了一份 cf
// 而cf->ctx 被覆盖成 ngx_http_conf_ctx_t 结构
// ngx_conf_t 的ctx应该指向 modules的数组，这里临时修改了指向hack了下。主要是为了http postconfiguration 和 config merge
// 在此函数最后 *cf = pcf
pcf = *cf;
cf->ctx = ctx;

for (m = 0; cf->cycle->modules[m]; m++) {
    if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
        continue;
    }

    module = cf->cycle->modules[m]->ctx;

    if (module->preconfiguration) {
        if (module->preconfiguration(cf) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
    }
}

/* parse inside the http{} block */

cf->module_type = NGX_HTTP_MODULE;
cf->cmd_type = NGX_HTTP_MAIN_CONF;
rv = ngx_conf_parse(cf, NULL);

if (rv != NGX_CONF_OK) {
    goto failed;
}

/*
    * init http{} main_conf's, merge the server{}s' srv_conf's
    * and its location{}s' loc_conf's
    */

cmcf = ctx->main_conf[ngx_http_core_module.ctx_index];
cscfp = cmcf->servers.elts;

for (m = 0; cf->cycle->modules[m]; m++) {
    if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
        continue;
    }

    module = cf->cycle->modules[m]->ctx;
    mi = cf->cycle->modules[m]->ctx_index;

    /* init http{} main_conf's */

    if (module->init_main_conf) {
        rv = module->init_main_conf(cf, ctx->main_conf[mi]);
        if (rv != NGX_CONF_OK) {
            goto failed;
        }
    }

    rv = ngx_http_merge_servers(cf, cmcf, module, mi);
    if (rv != NGX_CONF_OK) {
        goto failed;
    }
}


/* create location trees */

for (s = 0; s < cmcf->servers.nelts; s++) {

    clcf = cscfp[s]->ctx->loc_conf[ngx_http_core_module.ctx_index];

    if (ngx_http_init_locations(cf, cscfp[s], clcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

    if (ngx_http_init_static_location_trees(cf, clcf) != NGX_OK) {
        return NGX_CONF_ERROR;
    }
}
if (ngx_http_init_phases(cf, cmcf) != NGX_OK) {
    return NGX_CONF_ERROR;
}

if (ngx_http_init_headers_in_hash(cf, cmcf) != NGX_OK) {
    return NGX_CONF_ERROR;
}


for (m = 0; cf->cycle->modules[m]; m++) {
    if (cf->cycle->modules[m]->type != NGX_HTTP_MODULE) {
        continue;
    }

    module = cf->cycle->modules[m]->ctx;

    if (module->postconfiguration) {
        if (module->postconfiguration(cf) != NGX_OK) {
            return NGX_CONF_ERROR;
        }
    }
}

if (ngx_http_variables_init_vars(cf) != NGX_OK) {
    return NGX_CONF_ERROR;
}

/*
    * http{}'s cf->ctx was needed while the configuration merging
    * and in postconfiguration process
    */

// **看这里**
// 在 ngx_http_merge_servers 和 postconfiguration 之后，ctx又还原重新指向 modules 指针数组
*cf = pcf;
```

### merge_loc_conf

需要location merge是因为 location 可以嵌套

```txt
# main context

server {
    
    # server context

    location /match/criteria {

        # first location context

    }

    location /other/criteria {

        # second location context

        location nested_match {

            # first nested location

        }

        location other_nested {

            # second nested location

        }

    }

}
```

## Http 请求处理

### Phase

Nginx将请求分成如下几个阶段，请求到来后，按照顺序分别执行每个阶段的handlers

```c
typedef enum {
    NGX_HTTP_POST_READ_PHASE = 0,

    NGX_HTTP_SERVER_REWRITE_PHASE,

    NGX_HTTP_FIND_CONFIG_PHASE,
    NGX_HTTP_REWRITE_PHASE,
    NGX_HTTP_POST_REWRITE_PHASE,

    NGX_HTTP_PREACCESS_PHASE,

    NGX_HTTP_ACCESS_PHASE,
    NGX_HTTP_POST_ACCESS_PHASE,

    NGX_HTTP_PRECONTENT_PHASE,

    NGX_HTTP_CONTENT_PHASE,

    NGX_HTTP_LOG_PHASE
} ngx_http_phases;
```

由于nginx支持module插件的形式，必然要有一种机制，当请求来后遍历执行插件，可以在中间某个插件停止处理

Koa 使用洋葱模型，nginx则while迭代handler，handler里决定是否 `r->phase_handler++`指向下一个handler，如果handler返回 `NGX_OK`，则请求处理结束

```c
// https://github.com/taikulawo/nginx-debugable/blob/8a30c4b12226c92cd317db614172cc6de2795768/src/http/ngx_http_core_module.c#L875
while (ph[r->phase_handler].checker) {

    rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);

    if (rc == NGX_OK) {
        return;
    }
}
```

http core module 初始化阶段会将不同module挂载到不同phase上，如下

```c
// https://github.com/taikulawo/nginx-debugable/blob/8a30c4b12226c92cd317db614172cc6de2795768/src/http/ngx_http.c#L486
for (i = 0; i < NGX_HTTP_LOG_PHASE; i++) {
    h = cmcf->phases[i].handlers.elts;

    switch (i) {
    // 内置module 通过for loop 放到不同phase
    case NGX_HTTP_SERVER_REWRITE_PHASE:
        if (cmcf->phase_engine.server_rewrite_index == (ngx_uint_t) -1) {
            cmcf->phase_engine.server_rewrite_index = n;
        }
        checker = ngx_http_core_rewrite_phase;

        break;

    case NGX_HTTP_FIND_CONFIG_PHASE:
        find_config_index = n;

        ph->checker = ngx_http_core_find_config_phase;
        n++;
        ph++;

        continue;

    case NGX_HTTP_REWRITE_PHASE:
        if (cmcf->phase_engine.location_rewrite_index == (ngx_uint_t) -1) {
            cmcf->phase_engine.location_rewrite_index = n;
        }
        checker = ngx_http_core_rewrite_phase;

        break;

    case NGX_HTTP_POST_REWRITE_PHASE:
        if (use_rewrite) {
            ph->checker = ngx_http_core_post_rewrite_phase;
            ph->next = find_config_index;
            n++;
            ph++;
        }

        continue;

    case NGX_HTTP_ACCESS_PHASE:
        checker = ngx_http_core_access_phase;
        n++;
        break;

    case NGX_HTTP_POST_ACCESS_PHASE:
        if (use_access) {
            ph->checker = ngx_http_core_post_access_phase;
            ph->next = n;
            ph++;
        }

        continue;

    case NGX_HTTP_CONTENT_PHASE:
        checker = ngx_http_core_content_phase;
        break;

    default:
        checker = ngx_http_core_generic_phase;
    }

    n += cmcf->phases[i].handlers.nelts;

    for (j = cmcf->phases[i].handlers.nelts - 1; j >= 0; j--) {
        ph->checker = checker;
        ph->handler = h[j];
        ph->next = n;
        ph++;
    }
}

```

## Stream module hook 调用顺序

先看一个第三方 stream module的 ctx 结构

```c
typedef struct {
    ngx_int_t                    (*preconfiguration)(ngx_conf_t *cf);
    ngx_int_t                    (*postconfiguration)(ngx_conf_t *cf);

    void                        *(*create_main_conf)(ngx_conf_t *cf);
    char                        *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void                        *(*create_srv_conf)(ngx_conf_t *cf);
    char                        *(*merge_srv_conf)(ngx_conf_t *cf, void *prev,
                                                   void *conf);
} ngx_stream_module_t;

static ngx_stream_module_t  ngx_stream_upsync_module_ctx = {
    NULL,                                       /* preconfiguration */
    NULL,                                       /* postconfiguration */

    ngx_stream_upsync_create_main_conf,         /* create main configuration */
    ngx_stream_upsync_init_main_conf,           /* init main configuration */

    ngx_stream_upsync_create_srv_conf,          /* create server configuration */
    NULL,                                       /* merge server configuration */
};
```

调用顺序

1. create_main_conf：创建module自己的配置文件struct并返回
2. create_srv_conf
   创建server块的module 配置。举个例子：stream block下会有server block，此时会调用 create_srv_conf，放到 srv_conf 下的module对应的数组索引中。upstream block下也有 server block，也会调用 create_srv_conf，放到 srv_conf 下的module对应的数组索引中。所以 create_srv_conf 会在stream block和upstream block都调用。
3. preconfiguration: 配置创建之前
4. postconfiguration: 配置创建之后，但还未基于配置进行其他的行为。http request handler就放这里。nginx会将handler放入phase_engine，再往下放handler已经晚了
    到此 nginx 已经完成 upstream，server 块、module 的command/directive 解析。由于每个command都有set hook，set里都已经将value放到了之前创建的 conf 中
5. create_main_conf
6. create_srv_conf：server块配置初始化
    // 配置初始化完成后，下面开始调用 ngx_module_t 上的 module 级别的hook
7. create_loc_conf
8. init_main_conf
   nginx 将 module 自己创建好的 conf 传进来，module 初始化自身配置
9. merge_srv_conf: 有些指令可以在main block，也可以在 srv block，merge_srv_conf 的hook决定srv block的conf是否需要覆盖main block的。由于传递的是指针，也可以顺带修改main block的指令参数
10. init_module:
   每次配置文件reload，init_module都会被重新掉用，传进来 cycle，cycle->old_cycle，用于生成一个新cycle。最后创建好的cycle放在全局变量中，fork 新worker process后，cycle 会自动映射到worker process 地址空间，访问时写时拷贝。
11. init_process
   只在 worker process 调用。请求不经过master直接到worker，此hook可以执行module的业务逻辑，初始化处理请求相关的配置。比如 `nginx-stream-upsync-module`，在 init_process 里请求consul，etcd，获取下游ipport列表，放在worker自己的变量中
12. exit_process
   worker退出调用
13. exit_master
   master退出时掉用，优雅退出

## Stream 请求处理

### Phase

如下几个阶段，顺序执行每个阶段下的handlers
<http://nginx.org/en/docs/stream/stream_processing.html>

```c
typedef enum {
    NGX_STREAM_POST_ACCEPT_PHASE = 0,
    NGX_STREAM_PREACCESS_PHASE,
    NGX_STREAM_ACCESS_PHASE ,
    NGX_STREAM_SSL_PHASE,
    NGX_STREAM_PREREAD_PHASE,
    NGX_STREAM_CONTENT_PHASE,
    NGX_STREAM_LOG_PHASE
} ngx_stream_phases;

```

```c
// https://github.com/taikulawo/nginx-debugable/blob/8a30c4b12226c92cd317db614172cc6de2795768/src/stream/ngx_stream.c#L352
for (i = 0; i < NGX_STREAM_LOG_PHASE; i++) {
    h = cmcf->phases[i].handlers.elts;

    switch (i) {
    // stream module 将内置handlers放到不同的phase
    case NGX_STREAM_PREREAD_PHASE:
        checker = ngx_stream_core_preread_phase;
        break;

    case NGX_STREAM_CONTENT_PHASE:
        ph->checker = ngx_stream_core_content_phase;
        n++;
        ph++;

        continue;

    default:
        checker = ngx_stream_core_generic_phase;
    }

    n += cmcf->phases[i].handlers.nelts;

    for (j = cmcf->phases[i].handlers.nelts - 1; j >= 0; j--) {
        ph->checker = checker;
        ph->handler = h[j];
        ph->next = n;
        ph++;
    }
}
```

## stream upstream module 分析与负载均衡逻辑

upstream作为nginx的LB模块，承担着在server中流量均分的重任。

分析upstream module中，必然绕不开以下几个结构

```c
ngx_stream_upstream_server_t    *server = NULL;
// rr 为 round robin 简写
ngx_stream_upstream_rr_peer_t   *peer = NULL;
ngx_stream_upstream_rr_peers_t  *peers = NULL;
ngx_stream_upstream_srv_conf_t  *uscf;
```

通过 upstream block 对齐到上面类型

```nginx
stream {
    # ngx_stream_upstream_srv_conf_t
    # ngx_stream_upstream_rr_peers_t
    upstream a.b.c{
        # ngx_stream_upstream_server_t
        # ngx_stream_upstream_rr_peer_t
        server 127.0.0.1:12345            max_fails=3 fail_timeout=30s;
        # ngx_stream_upstream_server_t
        # ngx_stream_upstream_peer_t
        server unix:/tmp/backend3;
        server 1.2.3.4:8080 backup;
        server 5.6.7.8:8080 backup;
    }
    
    server {
        listen 12345;
        proxy_connect_timeout 1s;
        proxy_timeout 3s;
        proxy_pass backend;
    }
}
```

ngx_stream_upstream_srv_conf_t 存放 upstream block 级别的相关配置
ngx_stream_upstream_server_t 存放 upstream 单独一个server block配置，含有 `weight, max_fails` 等配置

`ngx_stream_upstream_rr_peers_t`, `ngx_stream_upstream_rr_peer_t` 与 round robin 相关

peer_t 记录LB相关参数，比如已经连接到此下游的连接数，下游响应时间，用于进行LB策略
peers_t#next参数和upstream 的backup指令有关，记录此peers_t的backup servers

当peer_t down，peers_t通过next找到此peer_t的backup peers，再通过 backup peers 找到 backup peer_t

ngx_stream_upstream_rr_peers_s 结构注释

```
struct ngx_stream_upstream_rr_peers_s {
    ngx_uint_t                       number; // 下游服务器数量

#if (NGX_STREAM_UPSTREAM_ZONE)
    ngx_slab_pool_t                 *shpool;
    ngx_atomic_t                     rwlock;
    ngx_stream_upstream_rr_peers_t  *zone_next;
#endif

    ngx_uint_t                       total_weight; // 所有服务器总权重

    unsigned                         single:1; // 此upstream只有一台下游
    unsigned                         weighted:1;

    ngx_str_t                       *name;

    ngx_stream_upstream_rr_peers_t  *next; // 存放全部backups下游

    ngx_stream_upstream_rr_peer_t   *peer; // 存放全部下游
};
```

ngx_stream_upstream_rr_peer_s 结构注释

```c
struct ngx_stream_upstream_rr_peer_s {
    struct sockaddr                 *sockaddr; // 后端地址
    socklen_t                        socklen; // 后端地址长度
    ngx_str_t                        name; // 后端名称
    ngx_str_t                        server;

    ngx_int_t                        current_weight; // 当前权重，运行中动态调整
    ngx_int_t                        effective_weight;
    ngx_int_t                        weight; // 配置的权重

    ngx_uint_t                       conns;
    ngx_uint_t                       max_conns;

    ngx_uint_t                       fails;
    time_t                           accessed;
    time_t                           checked;

    ngx_uint_t                       max_fails; // 最大失败次数
    time_t                           fail_timeout; // 多长时间内出现max_fails次失败，认为后端down
    ngx_msec_t                       slow_start;
    ngx_msec_t                       start_time;

    ngx_uint_t                       down; // 指定此后端down

    void                            *ssl_session;
    int                              ssl_session_len;

#if (NGX_STREAM_UPSTREAM_ZONE)
    ngx_atomic_t                     lock;
#endif

    ngx_stream_upstream_rr_peer_t   *next;

    NGX_COMPAT_BEGIN(25)
    NGX_COMPAT_END
};
```

![image](https://user-images.githubusercontent.com/24750337/188052512-c07e011e-760d-42bb-9e3e-194c90f1e3e1.png)

## 部分nginx struct剖析

### ngx_command_t

```c
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};

// example

static ngx_command_t  ngx_http_dubbo_commands[] = {

    { ngx_string("dubbo_pass"), // name
      NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF|NGX_CONF_TAKE4, // type
      ngx_http_dubbo_pass, // set
      NGX_HTTP_LOC_CONF_OFFSET, // conf
      0, // offset
      NULL /* post */},
      ngx_null_command
};
```

nginx回调set(ngx_http_dubbo_pass)，并根据 `.conf（NGX_HTTP_LOC_CONF_OFFSET）`，设置 cf->ctx

offset 只有和 set 配合才有意义。nginx有很多需要将配置项存放到结构体的需求，所以nginx提供了形如 ngx_conf_set_foo_slot 的函数，用于批量解决此需求。

这些ngx_conf_set_foo_slot需要知道将value放到结构体的哪个位置，因此在ngx_command_t上就须填写offset，由nginx计算位置并存放。

当然你完全可以将offset置为0，但这意味着需要针对每条directive都编写set，将 `void * conf` 转换成具体的结构体类型，通过 `a->b`设置。

比如nginx提供的一个函数，通过`cmd->offset`获取要set的struct成员并赋值

```c
char *
ngx_conf_set_num_slot(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    char  *p = conf;

    ngx_int_t        *np;
    ngx_str_t        *value;
    ngx_conf_post_t  *post;


    np = (ngx_int_t *) (p + cmd->offset);

    if (*np != NGX_CONF_UNSET) {
        return "is duplicate";
    }

    value = cf->args->elts;
    *np = ngx_atoi(value[1].data, value[1].len);
    if (*np == NGX_ERROR) {
        return "invalid number";
    }

    if (cmd->post) {
        post = cmd->post;
        return post->post_handler(cf, post, np);
    }

    return NGX_CONF_OK;
}

```

### ngx_conf_t

```c
struct ngx_conf_s {
    char                 *name;
    ngx_array_t          *args;

    ngx_cycle_t          *cycle;
    ngx_pool_t           *pool;
    ngx_pool_t           *temp_pool;
    ngx_conf_file_t      *conf_file;
    ngx_log_t            *log;

    void                 *ctx;
    ngx_uint_t            module_type;
    ngx_uint_t            cmd_type;

    ngx_conf_handler_pt   handler;
    void                 *handler_conf;
#if (NGX_SSL && NGX_SSL_ASYNC)
    ngx_flag_t            no_ssl_init;
#endif
};

// example module
ngx_module_t  ngx_stream_upsync_module = {
    NGX_MODULE_V1,
    &ngx_stream_upsync_module_ctx,               /* module context */
    ngx_stream_upsync_commands,                  /* module directives */
    NGX_STREAM_MODULE,                           /* module type */
    NULL,                                        /* init master */
    ngx_stream_upsync_init_module,               /* init module */
    ngx_stream_upsync_init_process,              /* init process */
    NULL,                                        /* init thread */
    NULL,                                        /* exit thread */
    ngx_stream_upsync_clear_all_events,          /* exit process */
    NULL,                                        /* exit master */
    NGX_MODULE_V1_PADDING
};
```

重点说下 `void* ctx`
ngx_conf_s 在配置解析过程中使用，module type定义了 ctx 存放的类型。上述 ngx_stream_upsync_module 为 stream module，所以解析stream block时，包括 stream upstream 下，ngx_conf_t#ctx总是指向 stream 的 ctx 配置 `ngx_stream_conf_ctx_t`

### ngx_str_t

```c
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;

```

len的长度不一定包含 `\0`，如果`ngx_str_t`由字符串常量初始化而来，比如`ngx_string("hello")`， `sizeof("hello")`为 5+1，而此时len却是5，确保一定是null结尾，方便传给syscall


```c
var->len = 2;
    var->data = (u_char *) "TZ";
```

〉 The `len` field holds the string length and `data` holds the string data. The string, held in `ngx_str_t`, may or may not be null-terminated after the `len` bytes. In most cases it’s not. However, in certain parts of the code (for example, when parsing configuration), `ngx_str_t` objects are known to be null-terminated, which simplifies string comparison and makes it easier to pass the strings to syscalls

## Module如何注册到Phase，参与请求的处理

上文说了 http core module 会在http module初始化中注册到phase，而第三方module则会在
<http://nginx.org/en/docs/dev/development_guide.html#http_phases>

```c
static ngx_http_module_t  ngx_http_foo_module_ctx = {
    NULL,                                  /* preconfiguration */
    ngx_http_foo_init,                     /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    NULL,                                  /* create location configuration */
    NULL                                   /* merge location configuration */
};


static ngx_int_t
ngx_http_foo_handler(ngx_http_request_t *r)
{
    ngx_str_t  *ua;

    ua = r->headers_in->user_agent;

    if (ua == NULL) {
        return NGX_DECLINED;
    }

    /* reject requests with "User-Agent: foo" */
    if (ua->value.len == 3 && ngx_strncmp(ua->value.data, "foo", 3) == 0) {
        return NGX_HTTP_FORBIDDEN;
    }

    return NGX_DECLINED;
}


static ngx_int_t
ngx_http_foo_init(ngx_conf_t *cf)
{
    ngx_http_handler_pt        *h;
    ngx_http_core_main_conf_t  *cmcf;

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
    // http://nginx.org/en/docs/dev/development_guide.html#http_phases
    // 将handler放到对应的phase list中
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_PREACCESS_PHASE].handlers);
    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_foo_handler;

    return NGX_OK;
}
```

## 指令directive

<http://nginx.org/en/docs/dev/development_guide.html#config_directives>

## master发送signal

<https://github.com/taikulawo/nginx-debugable/blob/8a30c4b12226c92cd317db614172cc6de2795768/src/os/unix/ngx_process_cycle.c#L139>

```c
for ( ;; ) {
    // 省略无关代码...
    // block 在 signal syscall
    sigsuspend(&set);

    if (ngx_terminate) {
        if (delay == 0) {
            delay = 50;
        }

        if (sigio) {
            sigio--;
            continue;
        }

        sigio = ccf->worker_processes + 2 /* cache processes */;

        // signal 属于进程间通信
        // 将 signal 发送给worker process
        if (delay > 1000) {
            ngx_signal_worker_processes(cycle, SIGKILL);
        } else {
            ngx_signal_worker_processes(cycle,
                                    ngx_signal_value(NGX_TERMINATE_SIGNAL));
        }

        continue;
    }
    // 省略无关代码...
}
```

## worker process 接受 master process signal

master, worker signal 处理代码都在一起

```c
// https://github.com/taikulawo/nginx-debugable/blob/8a30c4b12226c92cd317db614172cc6de2795768/src/os/unix/ngx_process.c#L319

switch (ngx_process) {

case NGX_PROCESS_MASTER:
case NGX_PROCESS_SINGLE:
    switch (signo) {

    case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
        ngx_quit = 1;
        action = ", shutting down";
        break;

    case ngx_signal_value(NGX_TERMINATE_SIGNAL):
    case SIGINT:
        ngx_terminate = 1;
        action = ", exiting";
        break;

    case ngx_signal_value(NGX_NOACCEPT_SIGNAL):
        if (ngx_daemonized) {
            ngx_noaccept = 1;
            action = ", stop accepting connections";
        }
        break;

    case ngx_signal_value(NGX_RECONFIGURE_SIGNAL):
        ngx_reconfigure = 1;
        action = ", reconfiguring";
        break;

    case ngx_signal_value(NGX_REOPEN_SIGNAL):
        ngx_reopen = 1;
        action = ", reopening logs";
        break;

    case ngx_signal_value(NGX_CHANGEBIN_SIGNAL):
        if (ngx_getppid() == ngx_parent || ngx_new_binary > 0) {

            /*
                * Ignore the signal in the new binary if its parent is
                * not changed, i.e. the old binary's process is still
                * running.  Or ignore the signal in the old binary's
                * process if the new binary's process is already running.
                */

            action = ", ignoring";
            ignore = 1;
            break;
        }

        ngx_change_binary = 1;
        action = ", changing binary";
        break;

    case SIGALRM:
        ngx_sigalrm = 1;
        break;

    case SIGIO:
        ngx_sigio = 1;
        break;

    case SIGCHLD:
        ngx_reap = 1;
        break;
    }

    break;

case NGX_PROCESS_WORKER:
case NGX_PROCESS_HELPER:
    switch (signo) {

    case ngx_signal_value(NGX_NOACCEPT_SIGNAL):
        if (!ngx_daemonized) {
            break;
        }
        ngx_debug_quit = 1;
        /* fall through */
    case ngx_signal_value(NGX_SHUTDOWN_SIGNAL):
        // worker, helper process 收到 shutdown signal后会设置ngx_quit
        // 而此时 worker 一直在 ngx_worker_process_cycle 函数中，
        // 当检测到 ngx_quit = 1后会关闭listening socket优雅退出
        // https://github.com/taikulawo/nginx-debugable/blob/f02e2a734ef472f0dcf83ab2e8ce96d1acead8a5/src/os/unix/ngx_process_cycle.c#L728
        ngx_quit = 1;
        action = ", shutting down";
        break;

    case ngx_signal_value(NGX_TERMINATE_SIGNAL):
    case SIGINT:
        ngx_terminate = 1;
        action = ", exiting";
        break;

    case ngx_signal_value(NGX_REOPEN_SIGNAL):
        ngx_reopen = 1;
        action = ", reopening logs";
        break;

    case ngx_signal_value(NGX_RECONFIGURE_SIGNAL):
    case ngx_signal_value(NGX_CHANGEBIN_SIGNAL):
    case SIGIO:
        action = ", ignoring";
        break;
    }

    break;
}
```

## worker listen socket 解决listen冲突

Linux Kernel 支持 [SO_REUSEPORT](https://github.com/taikulawo/nginx-debugable/blob/8a30c4b12226c92cd317db614172cc6de2795768/src/core/ngx_connection.c#L297)，允许多process accept 同一个socket，socket将按照一定的算法传给不同process

## 内存池

pool并不是说需要内存都从pool里申请，而是在进行一系列操作的时候发现需要频繁申请内存，这时候可以 `ngx_create_pool` 申请pool，之后这些操作都在此pool中进行。

全部操作都完成后，调用 `ngx_destroy_pool`就可以。所以 ngx_pfree 并不会释放小内存块

比如我们有个逻辑，heap上存了指向其他heap的pointer，如果为了释放这些内存，就需要每个字段都调用free，如果后期涉及到这里的更改，就必须添加上心字段的free，维护成本很高

而如果采用pool，逻辑都从pool里分配，只需要在逻辑开头create pool，在逻辑结束后destroy pool就可以，**非常好用**

## http请求运行流程

<https://github.com/willdeeper/tengine/blob/6efc27b8b955201eaad76f76554d0aaa40dac45b/src/http/ngx_http.c#L1729-L1735>

```c
// 向 cycle->listening 添加 `ngx_listening_t*`结构，并注册 ngx_http_init_connection 回调函数
ls->handler = ngx_http_init_connection;
```

<https://github.com/willdeeper/tengine/blob/3d65186c980559f4a76f0a48ec19933a76834bfc/src/http/ngx_http_request.c#L332>

```c
rev = c->read;
// 注册 socket on read handler, 用于解析 http request
rev->handler = ngx_http_wait_request_handler;
// 暂时没有要写的，先置空
c->write->handler = ngx_http_empty_handler;
```

ngx_http_wait_request_handler 函数中从socket读取request
<https://github.com/willdeeper/tengine/blob/3d65186c980559f4a76f0a48ec19933a76834bfc/src/http/ngx_http_request.c#L450>

```c
// ngx_http_wait_request_handler
n = c->recv(c, b->last, size);
// 省略代码.....
// request读取完成，创建 ngx_http_request_t
c->data = ngx_http_create_request(c);
if (c->data == NULL) {
    ngx_http_close_connection(c);
    return;
}

rev->handler = ngx_http_process_request_line;
// 开始处理 http 协议
ngx_http_process_request_line(rev);

```

```c
// ngx_http_process_request_line
// 处理 header
rev->handler = ngx_http_process_request_headers;
ngx_http_process_request_headers(rev);
```

http 调用栈

```c
ngx_http_core_run_phases (/data00/codes/tengine/src/http/ngx_http_core_module.c:941)

// 走nginx http phase
ngx_http_handler (/data00/codes/tengine/src/http/ngx_http_core_module.c:930)

// handler处理结束，之后就是body，request解析完成
ngx_http_process_request (/data00/codes/tengine/src/http/ngx_http_request.c:2227)

处理request header
ngx_http_process_request_headers (/data00/codes/tengine/src/http/ngx_http_request.c:1622)

// 处理 request line
ngx_http_process_request_line (/data00/codes/tengine/src/http/ngx_http_request.c:1283)
ngx_http_wait_request_handler (/data00/codes/tengine/src/http/ngx_http_request.c:535)
ngx_epoll_process_events (/data00/codes/tengine/src/event/modules/ngx_epoll_module.c:972)
ngx_process_events_and_timers (/data00/codes/tengine/src/event/ngx_event.c:260)
ngx_single_process_cycle (/data00/codes/tengine/src/os/unix/ngx_process_cycle.c:346)
main (/data00/codes/tengine/src/core/nginx.c:415)
__libc_start_main (@__libc_start_main:61)
_start (@_start:14)
```

从socket一次recv可能不是完成的http request，就需要暂存已有buffer，并返回，下一次recv从上一次结束处重新开始

当涉及到 NGX_AGAIN，nginx往往都会创建一个struct，将buf保存起来，并保证此函数重入是正确的

比如下面代码

```c
static void
ngx_http_process_request_line(ngx_event_t *rev)
{
    ssize_t              n;
    ngx_int_t            rc, rv;
    ngx_str_t            host;
    ngx_connection_t    *c;
    ngx_http_request_t  *r;

    c = rev->data;
    r = c->data;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, rev->log, 0,
                   "http process request line");

    if (rev->timedout) {
        ngx_log_error(NGX_LOG_INFO, c->log, NGX_ETIMEDOUT, "client timed out");
        c->timedout = 1;
        ngx_http_close_request(r, NGX_HTTP_REQUEST_TIME_OUT);
        return;
    }

    rc = NGX_AGAIN;

    for ( ;; ) {

        if (rc == NGX_AGAIN) {
            // NGX_AGAIN表示数据不完整，返回，等此函数重新被调用
            n = ngx_http_read_request_header(r);

            if (n == NGX_AGAIN || n == NGX_ERROR) {
                break;
            }
        }
    }
}

```


### worker_connections

worker可以处理的最大连接数，包含client和upstream的连接

比如 client => nginx => upstream模型

消耗两个connection

给资源受限的环境准备，现代服务器应尽可能大，比如 10k

整个nginx接受最大连接数为 `worker_processes * worker_connections`
### 健康检查

nginx可以编译 health check 模块，定期发送HTTP HEAD，确定connection正常使用

nginx 内置的connection检查

```c
// ngx_http_upstream_init_request

if (!u->store && !r->post_action && !u->conf->ignore_client_abort) {

    if (r->connection->read->ready) {
        ngx_post_event(r->connection->read, &ngx_posted_events);

    } else {
        if (ngx_handle_read_event(r->connection->read, 0) != NGX_OK) {
            ngx_http_finalize_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
            return;
        }
    }
    // request初始化阶段就是check broken connection的回调
    // fd read阶段 如果报错，就关掉连接
    r->read_event_handler = ngx_http_upstream_rd_check_broken_connection;
    r->write_event_handler = ngx_http_upstream_wr_check_broken_connection;
    // nginx proxy pass要获取upstream连接时，拿到新的连接，再将 read_event_handler 和 write_event_handler改成正常处理数据的handler
}

```

```c
// ngx_http_upstream_check_broken_connection

// NOBLOCKING socket，用PEEK读取
 n = recv(c->fd, buf, 1, MSG_PEEK);

err = ngx_socket_errno;

ngx_log_debug1(NGX_LOG_DEBUG_HTTP, ev->log, err,
               "http upstream recv(): %d", n);

if (ev->write && (n >= 0 || err == NGX_EAGAIN)) {
    return;
}

if ((ngx_event_flags & NGX_USE_LEVEL_EVENT) && ev->active) {

    event = ev->write ? NGX_WRITE_EVENT : NGX_READ_EVENT;
#if (NGX_HTTP_SSL && NGX_SSL_ASYNC)
    if (c->async_enable && ngx_del_async_conn) {
        if (c->num_async_fds) {
            ngx_del_async_conn(c, NGX_DISABLE_EVENT);
            c->num_async_fds--;
        }
    }
#endif

    if (ngx_del_event(ev, event, 0) != NGX_OK) {
        ngx_http_upstream_finalize_request(r, u,
                                           NGX_HTTP_INTERNAL_SERVER_ERROR);
        return;
    }
}
if (n > 0) {
// 读取成功，socket正常，返回
    return;
}

if (n == -1) {
    if (err == NGX_EAGAIN) {
     // 如果读取结果是would block，说明socket正常，现在没有数据
     // 返回
        return;
    }
    // 否则socket有问题
    ev->error = 1;

} else { /* n == 0 */
    err = 0;
}
```
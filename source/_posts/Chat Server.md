---
title: Chat聊天服务器架设历程
date: 2016-1-21 12:26:03
tags: javascript 
---

# Chat聊天服务器架设历程 #
----------
## 2015年11月3日 13:16:20 在Github上找到了Lets chat项目 ##

**首先安装了说明上的依赖**  

nodejs 0.10+  
MongoDB 2.6+  
Python 2.7.x

**然后将项目clone到本地目录**  

git clone https://github.com/sdelements/lets-chat.git

**安装项目依赖**  

cd lets-chat  

npm install

**npm install 出现错误**  

执行python 失败  

在系统环境变量中指定了python的安装目录

**然后npm install 成功**

复制设置文件
**cp settings.yml.sample settings.yml**

然后执行
**git pull 成功**
npm run-script migrate 报错


> 错误是Window Script Host  
> 脚本 D:\xxx\migroose.js  
> 行 3  
> 字符 1  
> 错误 缺少对象  
> 代码 800A138F  
> 源 Microsoft JScript 运行时错误

找到对应的[项目ISSUS](https://github.com/sdelements/lets-chat/issues/569 "项目ISSUS")

> hhaidar建议尝试使用 vagrant 或者 docker

然而**本地情况是mongoDB 无法正常连接**  即在cmd中键入 mongo 执行后出错

> 2015-11-03T13:31:04.068+0800 I CONTROL  Hotfix KB2731284 or later update is not
> installed, will zero-out data files
> MongoDB shell version: 3.0.7
> connecting to: test
> 2015-11-03T13:31:05.128+0800 W NETWORK  Failed to connect to 127.0.0.1:27017, re
> ason: errno:10061 由于目标计算机积极拒绝，无法连接。
> 2015-11-03T13:31:05.133+0800 E QUERY    Error: couldn't connect to server 127.0.
> 0.1:27017 (127.0.0.1), connection attempt failed
>     at connect (src/mongo/shell/mongo.js:179:14)
>     at (connect):1:6 at src/mongo/shell/mongo.js:179
> exception: connect failed

在cnBlog找到了一篇windows下注册mongoDB服务的文章

> **用管理员身份打开cmd** cd mongoDB的bin目录(必须留在该目录)
> mkdir xxx  创建目录用于存放数据库文件
> 执行指令(注意logpath指向文件 dbpath指向目录 都是绝对路径)
> `mongod --install --serviceName 服务名称 --serviceDisplayName 显示名称 --logpath xxxx --dbpath xxx --directoryperdb`
> directoryperdb 每个数据库独立目录

然后**cmd中可以正常链接到本地mongodb**

然而错误依旧是在migroose文件的第三行

    'use strict';

    var mongoose = require('mongoose'),	//require出错
    settings = require('./app/config'),
    migroose = require('migroose'),
    Runner = require('migroose-cli/cli/runner/index');

到最后也没找到什么解决办法,也懒得找了

## 2016年1月3日23:25:51 重新使用 `nodejs express jade socket.io` 创建了聊天室项目 ##

在使用`express-session`中间件时 选择了redis作为sessionStore
于是安装了tj大神的`connect-redis`组件
结果在运行时 向session中写入user信息时报错

  ```
  Error: MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk. Commands that may modify the data set are disabled. Please check Redis logs for details about the error.
    at JavascriptReplyParser._parseResult (e:\Github\NodeDataServer\node_modules\redis\lib\parsers\javascript.js:43:16)
    at JavascriptReplyParser.try_parsing (e:\Github\NodeDataServer\node_modules\redis\lib\parsers\javascript.js:114:21)
    at JavascriptReplyParser.run (e:\Github\NodeDataServer\node_modules\redis\lib\parsers\javascript.js:126:22)
    at JavascriptReplyParser.execute (e:\Github\NodeDataServer\node_modules\redis\lib\parsers\javascript.js:107:10)
    at Socket.<anonymous> (e:\Github\NodeDataServer\node_modules\redis\index.js:131:27)
    at emitOne (events.js:77:13)
    at Socket.emit (events.js:169:7)
    at readableAddChunk (_stream_readable.js:146:16)
    at Socket.Readable.push (_stream_readable.js:110:10)
    at TCP.onread (net.js:523:20)
  ```
去查了一下,多半的人都是说
>因为redis在持久化到硬盘的backsave的时候需要从当前运行时环境fork出一个镜像,然后使用子进程将内存镜像写入硬盘.

>这时复制镜像会消耗双倍的内存,所以内存不足时会失败.redis默认设置在失败时,拒绝写入,所以建议将拒绝写入的配置修改为no : `stop-writes-on-bgsave-error no`

我去看了一下本地redis服务的配置文件

  ```
      ################################ SNAPSHOTTING  ################################
    #
    # Save the DB on disk:
    #
    #   save <seconds> <changes>
    #
    #   Will save the DB if both the given number of seconds and the given
    #   number of write operations against the DB occurred.
    #
    #   In the example below the behaviour will be to save:
    #   after 900 sec (15 min) if at least 1 key changed
    #   after 300 sec (5 min) if at least 10 keys changed
    #   after 60 sec if at least 10000 keys changed
    #
    #   Note: you can disable saving completely by commenting out all "save" lines.
    #
    #   It is also possible to remove all the previously configured save
    #   points by adding a save directive with a single empty string argument
    #   like in the following example:
    #
    #   save ""

    save 900 1
    save 300 10
    save 60 10000

    # By default Redis will stop accepting writes if RDB snapshots are enabled
    # (at least one save point) and the latest background save failed.
    # This will make the user aware (in a hard way) that data is not persisting
    # on disk properly, otherwise chances are that no one will notice and some
    # disaster will happen.
    #
    # If the background saving process will start working again Redis will
    # automatically allow writes again.
    #
    # However if you have setup your proper monitoring of the Redis server
    # and persistence, you may want to disable this feature so that Redis will
    # continue to work as usual even if there are problems with disk,
    # permissions, and so forth.
    stop-writes-on-bgsave-error yes
  ```
首先,前面是数据快照的策略配置,默认
  - 至少1次更改,将于900秒后存储快照
  - 至少10次更改,将于300秒后存储快照
  - 至少10000次更改,将于60秒后存储快照

下面说了一下redis snapshots启用然后最后一次backsave失败的时候会拒绝写入.

然后解释了一下这样用户就会觉察到数据没有正确的持久化到硬盘上,为了防止世界被破坏云云...

接下来又假设了一下如果用户有相当健全的monitoring system的时候,就可以为所欲为 把配置设置为`no`

> 我查看了一下本机内存是没有问题的 硬盘就更不用说了

> 所以肯定不是因为内存不够的原因导致的

> 于是我在cmd里运行redis-server

  ```
  [7312] 03 Jan 23:38:08.472 #
  The Windows version of Redis allocates a memory mapped heap for sharing with
  the forked process used for persistence operations. In order to share this
  memory, Windows allocates from the system paging file a portion equal to the
  size of the Redis heap. At this time there is insufficient contiguous free
  space available in the system paging file for this operation (Windows error
  0x5AF). To work around this you may either increase the size of the system
  paging file, or decrease the size of the Redis heap with the --maxheap flag.
  Sometimes a reboot will defragment the system paging file sufficiently for
  this operation to complete successfully.

  Please see the documentation included with the binary distributions for more
  details on the --maxheap flag.

  Redis can not continue. Exiting.
  ```

从上面的错误来看,应该*decrease maxheap*,**我猜有可能是windows下默认的maxheap超过了阈值**

决定再去看一下配置,发现没有指定maxheap和maxmemory,然后随便给了*maxheap 1.5gb/ maxmemory 1gb*,再启动服务就好了...

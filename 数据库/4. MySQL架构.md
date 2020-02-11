## MySQL架构

1. ### MySQL的逻辑架构：

![img](images/436c6beeb41e0b3980c57df6d5ed4dca.png)

MySQL的逻辑架构分为三层：

1. 第一层是客户端的连接层，这个模块的主要功能是：身份认证、连接线程池、线程复用等功能
2. 第二层是MySQL的服务层，包含MySQL的大多数核心功能。主要包括:查询缓存(Caches & buffers)、分析器(SQL Parser)、优化器(Optimizer)、MySQL Managerment Server & Utilities(系统管理)、SQL Interface(SQL接口)
3. 第三层是MySQL的存储引擎层。MySQL的存储引擎是MySQL数据库非常重要的一个特点，其特殊之处在于插件式的表存储引擎。因为MySQL数据库是开源的，用户可以根据MySQL预定义的存储引擎接口编写自己的存储引擎。关于"如何实现一个自己的存储引擎"，可以参考MySQL开发者网站给出的[教程](https://dev.mysql.com/doc/internals/en/custom-engine.html)。

### 2.查询执行的流程

一个SQL语句是怎样被执行的？它的执行顺序是怎样的？

![2019011031](images/2019011031.png)

一条SQL语句的执行大致分为三段：

1. 处理链接请求，分配链接资源
2. SQL解析
3. 调用存储引擎获取数据

将这三段详细拆分，大致是下面的流程:

```
1. 连接
    client向MySQL Server发起链接请求
    client和server之间通过三次握手建立TCP链接
    
    MySQL Server向client发送一个挑战码
    client接受到挑战码，使用挑战码对用户名密码加密，密文发送给server
    server根据用户名和密码对用户进行身份认证
    
    if 认证失败:
        返回认证失败信息
    else:
        检查当前链接数量是否超出配置文件给出的max_connections数量
        if 超出max_connections:
            too many connections, 链接失败
        else:
            为此client分配链接成功

2. SQL解析
    检查Cache & buffer,检查Query语句是否完全匹配,再检查当前用户是否有权限,如果都成功直接从缓存返回数据
    如果上一步失败(没有命中缓存),将Query交给解析器，经过词法分析、语法分析生成解析树
    开始预处理，解决一些解析器无法解析的语义，检查权限，查询优化，更新解析树
    根据Query是SELECT、DML、DDL、Replicate、Status分发Query
    对应模块收到Query后，请求"访问控制模块"检查当前用户是否有访问目标表和目标字段的权限
    if 当前链接没有权限:
        返回错误信息
    else:
        调用表管理模块，检查table cache是否命中
        if cache命中:
            获取对应表和获取锁，直接返回cache中数据
        else:
            重新打开表文件
            根据表的meta数据，获取表的存储引擎类型等信息
            根据存储引擎信息调用对应的存储引擎处理请求
            保存处理过程的日志
            
3. 调用存储引擎获取数据
    调用存储引擎，根据query获取数据
    将处理结果返回给"链接进程/线程模块"返回给client
    进行后续的清理工作，继续等待新的请求或者断开链接。
```


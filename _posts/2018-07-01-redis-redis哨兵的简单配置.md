---
layout:     post
title:      redis哨兵的简单配置
subtitle:   配置过程
date:       2018-07-01
author:     薛睿
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - redis
---


# 哨兵是什么
哨兵是监控redis主从模式运行状态的特殊节点，主要负责监控主数据库和从数据库是否运行正常，当主数据库发生故障后自动将从数据库转换为主数据库。

# 哨兵的简单配置（以一个哨兵为例，多个哨兵类似）
- 启动一个主数据库（port:6379），启动三个从数据库（port分别为6380，6381, 6382），用到的redis.conf配置可以参考[《redis的安装配置及Jedis直连》](https://blog.csdn.net/xuerui_1997/article/details/80866693)。
    >redis-server redis.conf --port 6379
    >redis-server redis.conf slaveof 127.0.0.1 6379 --port 6380
    >redis-server redis.conf slaveof 127.0.0.1 6379 --port 6381
    >redis-server redis.conf slaveof 127.0.0.1 6379 --port 6382

    ![](https://img-blog.csdn.net/20180701212036177)

- 查看四个数据库节点是否正常启动
    >redis-cli -p 6379
    >info replication

    ![](https://img-blog.csdn.net/20180701215246101)

- 创建（修改）sentinel.conf配置文件的内容,port 26379表示哨兵节点的默认端口，我们不做更改。sentinel monitor mymaster 127.0.0.1 6379 2中最后的1表示最低通过票数，因为我们只有一个哨兵，所以我们把其值改为1。其他参数保持默认就好。

- 启动哨兵节点
    >redis-sentinel sentinel.conf

    ![](https://img-blog.csdn.net/20180701223405896)
    由上图知，哨兵通过监控主数据库可以自动获取从数据库的信息并监控，本例中哨兵发现了6380,6381和6382  三个从数据库。

# 验证哨兵的作用
- 关闭主数据库（port:6379）
    >redis-cli -p 6379
    >shutdown

- 哨兵的配置文件中默认为30秒，我们等待30秒后哨兵节点输出如下信息: 
    ![](https://img-blog.csdn.net/20180701230134355)
    分析上图信息我们得知，其中+sdown表示哨兵主观认为主数据库停止服务了，+odown表示哨兵客观认为主数据库停止服务了。try-failover表示哨兵开始进行故障恢复，经过一系列操作后failover-end表示哨兵完成了故障恢复。最后输出的+slave信息表示故障恢复后新的主从结构，主数据库由原来的6379变为现在的6381,6380和6382变为6381的从数据库。

- 查看新的主从结构
    >redis-cli -p 6381
    >info replication

    ![](https://img-blog.csdn.net/20180701230948316)

    由上图可知当原来的主数据库（port:6379）发生故障后，原来的从数据库（port:6381）转换为新的主数据库。哨兵作用得以体现。

    ![](https://redis.io/images/redis-white.png)
    12345678




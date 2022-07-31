## Redis分布式锁

### 一. 概述

分布式锁：随着用户越来越多，我们上了好多服务器，原本有个定时给客户发邮件的任务，如果不加以控制的话，到点后每台机器跑一次任务，客户就会收到 N 条邮件，这就需要通过分布式锁来互斥了。

### 五. 参考资料

- https://www.51cto.com/article/682636.html
- https://www.baeldung.com/redis-redisson


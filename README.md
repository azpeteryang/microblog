<div align="center">
    <h1>
        The design of a blog website
    </h1>
</div>

<!-- GFM-TOC -->

* System Architecture Diagram
* Software
* Performance
* Cache
    * Popular blog
    * Reids and Memcache
    * Redis configuration
    * implement
    * The compromise in bussiness
    * Choose of serialization
* Feed stream
* Security
    * XSS sefence
    * Redis Crackit
* Interaction Design
    - Asynchronously modify user avatar at login
    - Asynchronous post deletion
    <!-- GFM-TOC -->

## System Architecture Diagram

<div align="center">
  <img src="pics/1.png">
</div>


### Software

- Linux : version 2.6.32-642.6.2.el6.x86_64
- MySQL : Ver 14.14 Distrib 5.1.73
- Redis : v=4.0.8

## Performance

Use Apache ab tool to test the pressure。

In order to defend the inference of the of the timeout of the internet, run Apache ab tool on the server side

Use the following command to use the ab tools, -c is the number of concurrency, -n is the number of requests, -k means persistent connection.

```
ab -c 1000 -n 5000 -k http://...
```

Before using Redis for caching and using the master-slave architecture to achieve read-write separation, some of the results obtained from the above tests are as follows. It can be seen that the average number of requests per second is 715.81.

```
Time taken for tests:   6.985 seconds
Total transferred:      2645529 bytes
HTML transferred:       1530306 bytes
Requests per second:    715.81 [#/sec] (mean)
```

After using Redis and the master-slave architecture, the test results are as follows, the average number of requests per second and increased to 4839.62, greatly improving the throughput of the website.

```
Time taken for tests:   1.033 seconds
Total transferred:      2696313 bytes
HTML transferred:       1559682 bytes
Requests per second:    4839.62 [#/sec] (mean)
```

## Cache implementation

### Popular blogs

The content of blog has the characteristics of reading more and writing less. In this scenario, it is particularly suitable for caching data. Redis is used to cache popular blog data.

### Redis and Memcache

At the beginning of the project, facing the choice of Redis and Memcache, the main differences between them are as follows:

- Redis has better distributed support, while Memcache can only use consistent hashing on the client to support distributed.
- Redis has two persistence features, RDB snapshot and AOF log, while Memcache has no persistence mechanism.
- Redis Value supports five types, while Memcache Value can only be String.
- Redis will swap the KV that has been useless for a long time to disk, and Memchache's data has been in memory.
- In order to completely remove the impact of disk fragmentation, Memcache divides the memory into blocks of a specific length to store data, but this method results in low memory utilization. For example, the block size is 128 bytes, and only 100 bytes of data is stored, so the remaining 28 bytes are wasted.

Considering that the project needs to use multiple cache servers, Redis is preferred. Although Spring integrates Redis, you can use Redis for caching with annotations such as @Cacheable, but in order to make the cache more controllable, I chose to implement the caching function myself.

### Redis 配置

First, Redis needs to be configured, mainly in two aspects: maximum memory usage and cache elimination strategy.

The larger the maximum memory usage is within the acceptable range of the server, the better. It is generally larger than the hotspot data, because Reids is not only used to store data, but also stores data during the operation of Redis.

Redis has five cache elimination strategies. In order to choose a strategy suitable for the project, you need to first understand each strategy.

NoEviction and TTL (Time to Live) are not suitable for the cache system of this project, because neither elimination nor elimination according to the expiration time can guarantee that the data remaining in the cache is as hot data as possible. Random is also related to expiration time, and randomization strategies cannot guarantee hotspot data. The LRU (least recently used) strategy eliminates the least recently used data. The most recently used data is considered to be hot data. Therefore, after the least recently used data is eliminated, the data in the cache can be largely guaranteed. All are popular data.

|      Method       |                         Description                        |
| :-------------: | :--------------------------------------------------: |
|  volatile-lru   | Select the least recently used data from the data set with the expiration time set to be eliminated |
|  volatile-ttl   |   Select the data that is about to expire from the data set that has been set to expire   |
| volatile-random |      Choose data elimination from the data set with the expiration time set      |
|   allkeys-lru   |       Select the least recently used data from all data sets to eliminate       |
| allkeys-random  |          Randomly select data from all data sets for elimination          |
|   noeviction    |                     Ban eviction data                     |

In addition to being used as a cache elimination strategy in Redis, LRU is used in many occasions. For example, the page replacement algorithm of the operating system can use LRU, because the page replacement algorithm is also equivalent to a cache elimination algorithm. The LinkedHashMap in Java can save LRUs inserted with key-value pairs. In Java programs, LinkedHashMap can be used to achieve a similar cache elimination function.

Realizing LRU is actually very simple, that is, maintaining the order through a linked list. When an element is accessed, the element is moved to the head of the linked list. Then the element at the tail of the linked list is the least recently used element, which can be eliminated.

### Implement

In order to realize the caching function, the implementation code of acquiring blog and adding Weibo needs to be modified.

In the code for getting Weibo, first get it from Redis, and if it fails, get it from the database.

Among them, BlogCacheDao implements the function of acquiring and adding cache. CacheHitDao is used to record the number of cache hits and misses. This is to monitor the system to optimize the cache, and can find cache penetration and cache avalanche problems .

When adding blogs to the database, it must also be added to Redis. This is because the database uses a master-slave architecture to achieve read and write separation. The master-slave synchronization process requires a certain amount of time. Inconsistent. If the read request is sent to the secondary database, then the latest data cannot be read. If the data is added to the cache while writing, the request to read the latest data will not be sent to the slave server, thereby avoiding inconsistencies between the master and standby servers during synchronization.

```java
@Override
public Blog getBlogByBlogId(int blogId)
{
    Blog blog = blogCacheDao.getBlog(blogId);
    if (blog != null) {
        cacheHitDao.hit();           /* 缓存命中 */
    } else {
        blog = blogMapper.selectByPrimaryKey(blogId);
        cacheHitDao.miss();          /* 缓存未命中 */
        blogCacheDao.addBlog(blog);  /* 放入缓存 */
    }
    return blog;
}

@Override
public void addBlog(int userId, String content)
{
    Blog blog = new Blog();
    blog.setUserid(userId);
    blog.setContent(content);
    blog.setPublishtime(new Date());
    blogMapper.insert(blog);
    blogCacheDao.addBlog(blog);
}
```

### Compromise

If the content of blog cannot be modified, then the problem of cache inconsistency can be avoided, and caches at all levels can be guaranteed to be effective caches.

The content of blog is often very simple. If you find something wrong after publishing, the cost of reposting it will not be very high. And the content of blog is often time-sensitive, that is, people will only read the recent Weibo, so a user who wants to modify the content of blog a long time ago does not make much sense, because few people can see it.

This project makes a trade-off in business and does not provide the function of modifying Weibo, which can greatly improve the performance of the system.

It can be found from this that sometimes difficult technical problems that cannot be overcome can be easily solved by simply adjusting the business.

### Choice of serialization

When implementing the Redis cache function, I first chose to use Java's own serialization method to convert an object into a byte array and then store it, but later realized that many of the content obtained by this serialization is class-defined content, this part of the content There is no need to store in the cache at all, only a few key fields need to be spliced into a string storage, the implementation code is as follows:

```java
    public static String writeBlogObject(Blog blog)
    {
        StringBuilder s = new StringBuilder();
        s.append(blog.getUserid()).append(separator);
        s.append(blog.getBlogid()).append(separator);
        s.append(DateUtil.formatDate(blog.getPublishtime())).append(separator);
        s.append(blog.getContent());
        return s.toString();
    }

    public static Blog readBlogObject(String s)
    {
        Blog blog = new Blog();
        String[] token = s.split(separator);
        blog.setUserid(Integer.valueOf(token[0]));
        blog.setBlogid(Integer.valueOf(token[1]));
        blog.setPublishtime(DateUtil.parseDate(token[2]));
        if(token.length > 3) {
            blog.setContent(token[3]);
        }
        return blog;
    }
```

In order to verify the time and space overhead of the two serialization methods, two benchmark tests were conducted. The test code is in com.cyc.benchmark.SerializeTest.java, because the longer code is not posted.

First simulate a blog object, store a certain amount of Weibo content, and then use the serialization and splicing methods that come with Java to run serialization and deserialization 1,000,000 times, respectively, to count the number of bytes and total time required for storage ,as follows:

|                | Java 序列化 | 字段拼接 |
| :------------: | :---------: | :------: |
| 存储所需字节数 |     298     |    48    |
|   运行时间/s   |   14.301    |  4.113   |

It can be found that the serialization method implemented by the field stitching method is much better than the serialization method that comes with Java, both in space and time, so the project uses its own field stitching method.

As for the JSON serialization method, because it also stores the name of the field, it is easy to know that the space overhead is much higher than the field splicing method.

## Feed Stream

Use Redis's ZSET data structure to maintain a set of published microblog IDs S1 for each user, with a score of timestamp. When the collection size exceeds a certain threshold, the oldest data is deleted.

Similarly, ZSET is used to maintain a set of microblog IDs S2 for each user, and the pull mode is used to maintain S2. After the user refreshes the homepage, they will actively pull data from their followers S1. The specific pulling process is to first take the latest timestamp t in S2, traverse all S1 of the users concerned, take out the data with a score from t to infinity, and add it to S2. When the size of S2 exceeds a certain threshold, the oldest data needs to be deleted.



When a user focuses on a new user, the user's S1 needs to be merged into the user's S2. Similarly, when a user unfollows a user, it is necessary to traverse S2 and remove the S1 content of the following user.


### 读写分离

![](pics/6.png)

主服务器用来处理写操作以及最新的读请求，而从服务器用来处理读操作。

读写分离常用代理方式来实现，代理服务器接收应用层传来的读写请求，然后决定转发到哪个服务器。

MySQL 读写分离能提高性能的原因在于：

- 主从服务器负责各自的读和写，极大程度缓解了锁的争用；
- 从服务器可以配置 MyISAM 引擎，提升查询性能以及节约系统开销；
- 增加冗余，提高可用性。

### 主从复制配置

#### 创建复制账号

在主从服务器都创建用于复制的账号，并且账号必须在 master-host 和 slave-host 都进行授权，也就是说以下的命令需要在主从服务器上都执行一次。

```
mysql > grant all privileges on *.* to repl@'master-host' identified by 'password';
mysql > grant all privileges on *.* to repl@'slave-host' identified by 'password';
mysql > flush privileges;
```

完成后最好测试一下主从服务器是否能连通。

```
mysql -u repl -h host -p
```

#### 配置 my.cnf 文件

主服务器

```
[root]# vi /etc/my.cnf

[mysqld]
log-bin  = mysql-bin
server-id = 10
```

从服务器

```
[root]# vi /etc/my.cnf

[mysqld]
log-bin          = mysql-bin
server-id        = 11
relay-log        = /var/lib/mysql/mysql-relay-bin
log-slave-updates = 1
read-only         = 1
```

重启 MySQL

```
[root]# service mysqld restart;
```

《高性能 MySQL》书上的配置文件中使用的是下划线，例如 server_id，使用这种方式在当前版本的 MySQL 中不再生效。

#### 启动复制

先查看主服务器的二进制文件名：

```
mysql > show master status;
```

```
+------------------+----------+--------------+------------------+
| File            | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000002 |      106 |              |                  |
+------------------+----------+--------------+------------------+
```

然后配置从服务器：

```
mysql > change master to master_host='master-host',            > master_user='repl',
      > master_password='password',
      > master_log_file='mysql-bin.000002',
      > master_log_pos=0;
```

在从服务器上启动复制：

```
mysql > start slave
```

查看复制状态，Slave_IO_Running 和 Slave_SQL_Running 必须都为 Yes 才表示成功。

```
mysql > show slave status\G;
*************************** 1. row ***************************
              Slave_IO_State: Waiting for master to send event
                  Master_Host:
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 106
              Relay_Log_File: mysql-relay-bin.000006
                Relay_Log_Pos: 251
        Relay_Master_Log_File: mysql-bin.000002
            Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
            ...
```

## 安全性

### XSS 防御

微博之类的内容网站，如果没有对发布的内容进行处理，很容易受到 XSS 攻击的影响。例如任何用户都可以发布以下内容：

```html
<script> alert("hello"); </script>	
```

这个脚本会被执行，从而导致其它用户在浏览到该内容时，浏览器弹出一个窗口，影响使用体验。

除此之外，XSS 还可以产生以下危害：

- 窃取用户的 Cookie
- 伪造虚假的输入表单骗取个人信息
- 显示伪造的文章或者图片

防御 XSS 也很容易，只要将 `<` 和 `>` 等字符进行转义即可。

### Redis Crackit

当使用 root 用户运行 Redis，并且 Redis 未设置密码或者设置为初始密码，那么攻击者很容易登录到 Redis 上，并且使用 config 命令修改 authorized_keys 文件，从而让攻击者无需用户名和密码即可登录。

解决方案是，使用普通用户运行 Redis，并且设置复杂的 Redis 密码。

## 交互设计

### 登录时异步修改用户头像

为了提高用户在登录时的使用体验，通过监听用户名输入框的 focusout 事件，当该事件发生时，通过 ajax 异步获取用户头像。

```js
var userNameInput = $("input[name=userName]");
userNameInput.focusout(function ()
{
    $.ajax({
            url: "getUserHeadPic.html",
            type: "post",
            dataType: "text",
            data: {
                userName: userNameInput.val()
            },
            success: function (headpic)
            {
                replaceHeadPic(headpic);
            }
    });
}
```

<div align="center">
    <img src="pics/9.gif">
</div>

### 异步删帖

考虑到用户在删帖后想继续浏览剩下的帖子，因此采用异步的方式进行删帖，删帖之后不需要刷新页面。

删帖成功之后，会将该帖子隐藏。为了更好的体验，隐藏过程设置了一个 200 毫秒的延迟，从而具有一个短暂的隐藏动画效果。

```js
var deleteBlog = $("#delete-blog");

deleteBlog.on("click", function ()
{
    var blogid = deleteBlog.attr("blogid");
    var blogDiv = $("#blog-" + blogid.toString());
    $.ajax({
        url: "editBlog.html",
        type: "post",
        dataType: "text",
        data: {
            blogId: blogid
        },
        success: function ()
        {
            blogDiv.hide(200);
        }
    });
})
```



<div align="center">
    <img src="pics/10.gif">
</div>



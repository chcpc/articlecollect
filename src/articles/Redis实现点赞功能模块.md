# Redis实现点赞功能模块
## 功能点设计

比如我喜欢发文章的掘金网站就有点赞的功能，统计文章点赞的总数，用户所有文章的点赞数，因此设计的点赞功能模块具有以下功能点：

某篇文章的点赞数

用户所有文章的点赞数

用户点赞的文章

持久化到MySQL数据库

## 数据库设计
Redis数据库设计

Redis是K-V数据库，没有统一的数据结构，针对不同的功能点设计了不同的K-V存储结构

用户某篇文章的点赞数

使用HashMap数据结构，HashMap中的key为articleId，value为Set，Set中的值为用户ID，即HashMap<String, Set<String>>

用户总的点赞数

使用HashMap数据结构，HashMap中的key为userId，value为String记录总的点赞数

用户点赞的文章

使用HashMap数据结构，HashMap中的key为userId，value为Set，Set中的值为文章ID，即HashMap<String, Set<String>>

## MySQL数据库设计

最主要的两张表，article表和user_like_article

article表结构
字段值	字段类型	说明

article_name	varchar	文章名字

content	blob	文章内容

total_like_count	bigint	文章总点赞数

文章总的点赞数需要和Redis中的点赞数进行同步

user_like_article表结构

字段值	字段类型	说明

user_id	bigint	用户ID

article_id	bigint	文章ID

记录用户点赞文章的信息，是一张中间表


说明：表结构设计省略了id、deleted、gmt_create、gmt_modified字段

## 流程图


流程图
流程图比较简单，点赞和取消点赞基本实现步骤相同

参数校验

对传入的参数进行null值判断

逻辑校验

对于用户点赞，用户不能重复点赞相同的文章

对于取消点赞，用户不能取消未点赞的文章

存入Redis

存入的数据主要有所有文章的点赞数、某篇文章的点赞数、用户点赞的文章

定时任务

通过定时【1小时执行一次】，从Redis读取数据持久化到MySQL中

## 代码功能实现

点赞

```java
public void likeArticle(Long articleId, Long likedUserId, Long likedPostId) {
    validateParam(articleId, likedUserId, likedPostId);  //参数验证

    logger.info("点赞数据存入redis开始，articleId:{}，likedUserId:{}，likedPostId:{}", articleId, likedUserId, likedPostId);
    synchronized (this) {
        //只有未点赞的用户才可以进行点赞
        likeArticleLogicValidate(articleId, likedUserId, likedPostId);
        //1.用户总点赞数+1
        redisTemplate.opsForHash().increment(TOTAL_LIKE_COUNT_KEY, String.valueOf(likedUserId), 1);

        //2.用户喜欢的文章+1
        String userLikeResult = (String) redisTemplate.opsForHash().get(USER_LIKE_ARTICLE_KEY, String.valueOf(likedPostId));
        Set<Long> articleIdSet = userLikeResult == null ? new HashSet<>() : FastjsonUtil.deserializeToSet(userLikeResult, Long.class);
            articleIdSet.add(articleId);
        redisTemplate.opsForHash().put(USER_LIKE_ARTICLE_KEY, String.valueOf(likedPostId), FastjsonUtil.serialize(articleIdSet));

        //3.文章点赞数+1
        String articleLikedResult = (String) redisTemplate.opsForHash().get(ARTICLE_LIKED_USER_KEY, String.valueOf(articleId));
        Set<Long> likePostIdSet = articleLikedResult == null ? new HashSet<>() : FastjsonUtil.deserializeToSet(articleLikedResult, Long.class);
        likePostIdSet.add(likedPostId);
        redisTemplate.opsForHash().put(ARTICLE_LIKED_USER_KEY, String.valueOf(articleId), FastjsonUtil.serialize(likePostIdSet));
        logger.info("取消点赞数据存入redis结束，articleId:{}，likedUserId:{}，likedPostId:{}", articleId, likedUserId, likedPostId);
    }
}
```

取消点赞

```java
public void unlikeArticle(Long articleId, Long likedUserId, Long likedPostId) {
    validateParam(articleId, likedUserId, likedPostId);  //参数校验

    logger.info("取消点赞数据存入redis开始，articleId:{}，likedUserId:{}，likedPostId:{}", articleId, likedUserId, likedPostId);
    //1.用户总点赞数-1
    synchronized (this) {
        //只有点赞的用户才可以取消点赞
        unlikeArticleLogicValidate(articleId, likedUserId, likedPostId);
        Long totalLikeCount = Long.parseLong((String)redisTemplate.opsForHash().get(TOTAL_LIKE_COUNT_KEY, String.valueOf(likedUserId)));
         redisTemplate.opsForHash().put(TOTAL_LIKE_COUNT_KEY, String.valueOf(likedUserId), String.valueOf(--totalLikeCount));

        //2.用户喜欢的文章-1
        String userLikeResult = (String) redisTemplate.opsForHash().get(USER_LIKE_ARTICLE_KEY, String.valueOf(likedPostId));
        Set<Long> articleIdSet = FastjsonUtil.deserializeToSet(userLikeResult, Long.class);
        articleIdSet.remove(articleId);
        redisTemplate.opsForHash().put(USER_LIKE_ARTICLE_KEY, String.valueOf(likedPostId), FastjsonUtil.serialize(articleIdSet));

        //3.取消用户某篇文章的点赞数
        String articleLikedResult = (String) redisTemplate.opsForHash().get(ARTICLE_LIKED_USER_KEY, String.valueOf(articleId));
        Set<Long> likePostIdSet = FastjsonUtil.deserializeToSet(articleLikedResult, Long.class);
        likePostIdSet.remove(likedPostId);
        redisTemplate.opsForHash().put(ARTICLE_LIKED_USER_KEY, String.valueOf(articleId), FastjsonUtil.serialize(likePostIdSet));
    }

    logger.info("取消点赞数据存入redis结束，articleId:{}，likedUserId:{}，likedPostId:{}", articleId, likedUserId, likedPostId);
}
```

异步落库

```java
@Scheduled(cron = "0 0 0/1 * * ? ")
public void redisDataToMySQL() {
    logger.info("time:{}，开始执行Redis数据持久化到MySQL任务", LocalDateTime.now().format(formatter));
    //1.更新文章总的点赞数
    Map<String, String> articleCountMap = redisTemplate.opsForHash().entries(ARTICLE_LIKED_USER_KEY);
    for (Map.Entry<String, String> entry : articleCountMap.entrySet()) {
        String articleId = entry.getKey();
        Set<Long> userIdSet = FastjsonUtil.deserializeToSet(entry.getValue(), Long.class);
        //1.同步某篇文章总的点赞数到MySQL
        synchronizeTotalLikeCount(articleId, userIdSet);
        //2.同步用户喜欢的文章
        synchronizeUserLikeArticle(articleId, userIdSet);
    }
    logger.info("time:{}，结束执行Redis数据持久化到MySQL任务", LocalDateTime.now().format(formatter));
}
```

说明：

针对存在并发的问题，通过添加synchronize关键字实现

另外还有获取某篇文章的点赞数、用户所有文章的点赞数、用户点赞的文章方法实现，方法实现比较简单不说明，可以在完整代码中找到

## 目前存在的不足
用户点赞\取消点赞方法中，Redis事务没有保证

该应用只适用于单机环境，分布式环境下存在并发问题，分布式锁待完成

十一过后对假期意犹未尽

最后附：完整代码地址
欢迎fork与 star，如有纰漏欢迎指正

附往期文章：欢迎你的阅读、点赞、评论
并发相关：
1.为什么阿里巴巴要禁用Executors创建线程池？
2.自己的事情自己做，线程异常处理
设计模式相关：
1. 单例模式，你真的写对了吗？
2. (策略模式+工厂模式+map)套餐 Kill 项目中的switch case

JAVA8相关：
1. 使用Stream API优化代码
2. 亲，建议你使用LocalDateTime而不是Date哦

数据库相关：
1. mysql数据库时间类型datetime、bigint、timestamp的查询效率比较
2. 很高兴！终于踩到了慢查询的坑

高效相关：
1. 撸一个Java脚手架，一统团队项目结构风格

日志相关：
1. 日志框架，选择Logback Or Log4j2？
2. Logback配置文件这么写，TPS提高10倍

工程相关：
1. 闲来无事，动手写一个LRU本地缓存
2. JMX可视化监控线程池
3. 权限管理 【SpringSecurity篇】
4. Spring自定义注解从入门到精通
5. java模拟登陆优酷
6. QPS这么高，那就来写个多级缓存吧
7. java使用phantomjs进行截图

其他：
1. 使用try-with-resources优雅关闭资源
2. 老板，用float存储金额为什么要扣我工资

作者：何甜甜在吗
链接：https://www.jianshu.com/p/a43c6d2f8bfb
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
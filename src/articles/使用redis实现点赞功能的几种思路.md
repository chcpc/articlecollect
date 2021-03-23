点赞功能几乎是现在互联网产品的标配了，点赞存在的意思还是蛮有趣的为什么社交网站的评价功能多采用「点赞」的模式？。

本文主要介绍本人工作中遇到的点赞需求以及使用redis的解决思路。第一种点赞需求是比较常规的点赞需求，类似于微博那种点赞模式，用户可以对某条信息点赞、取消点赞、查询是否点赞、被点赞次数等等；第二种点赞稍微特殊，用户可以在一天内对任意用户点赞，取消点赞后不可以再次对同用户点赞，第二天限制解除，可以重新对同一玩家点赞（也就是说点赞是可以累加的），然后还有一个需求是要求可以实时查用户获赞次数全局的排行情况。

需求一解决思路
对于需求一，采用的是redis bitmap来实现。

bitmap简介
bitmap

bitmap是一连串的二进制数字(0,1)，每一位所在的位置为偏移(offset)，在bitmap上可以执行AND,OR,XOR以及其他操作。

位图计数

位图计数的意思是统计bitmap中值为1的位的个数，位图计数的效率是很高的。

redis bitmap
redis中允许使用二进制的Key和二进制的Value，bitmap就是二进制的Value。

点赞/取消点赞
假设用户的数字id为1000，对照片id为100的照片点赞。首先根据照片id生成赞数据存储的redis key，比如生成策略为like_photo:{photo_id}，id为1000的用户点赞，只需要将like_photo:100的第1000位置为1即可（取消赞则置为0）。

redis setbit操作的时间复杂度为O(1)，所以这种点赞方式十分高效。


```
redis.setbit('like_photo:100', 1000, 1, function(err, ret){
    // deal err and ret.
});
```
当前是否点赞
用户打开图片的时候需要查询当前是否点赞过该照片，查询是否点赞可以通过redis getbit操作来实现。比如查询用户id为1000的用户是否点赞过照片id为100的照片，只需要对like_photo:100bitmap的第1000位取值即可。

redis getbit操作的时间复杂度同样是O(1)。


```
redis.getbit('like_photo:100', 1000, function(err, liked){
    // deal err.
    // if liked==1 liked, liked==0 not like yet.
});
```
查询点赞总次数
比如需要显示照片id为100的照片的获赞次数，只需要对like_photo:100bitmap进行位图计数操作即可。

redis bitcount操作的时间复杂度虽然是O(N)的，但是大部分数据量的情况下是不需要担心bitcount效率问题的。

```
redis.bitcount('like_photo:100', function(err, likeCnt){
    // deal with err and likeCnt.
});
```
其他
redis bitmap的实现中还提供了bitop等其他API，可以实现其他一些有趣的事情。

比如要计算同时点赞了100和101两张照片的用户，可以通过如下操作实现：


```
redis.bitop('AND', 'like_photo:100&101', 'like_photo:100', 'like_photo:101', function(err, ret){
    // 得到的like_photo:100&101这个临时key中即是同时点赞100和101的用户bitmap.
});
```
局限性
这种方案虽然比较高效，实现起来也比较简单，但是也有一定的局限性。
1.需要用户有类似于数据库自增id的数字id，当然如果你是从10000之类的开始自增的，在bitmap操作的时候可以统一将用户id减掉10000，这样可以稍微节省一些redis内存占用；
2.当用户量很大的时候，比如千万级用户量的情况下，一个点赞bitmap需要消耗的内存为：10000000/8/1024/1024=1.19MB，当bitmap数量较多的时候，内存占用还是很可观的。不过在用户量较少的时候这种方案还是不错的~

需求二解决思路
对于需求二采用redis sorted set来实现，sorted set也是redis中比较常用的数据结构，这里就不再介绍了。

点赞redis key设计
由于需求二中有当日这种时间限制，所以在设计key的时候可以将日期作为key的一部分。

1.存储MMDD日期用户uid的获赞数据：


```
var genLikeKey = function(uid){
    return 'like:'+moment().format('MMDD')+':'+uid;
};
```
2.全局获赞数据key:

```
var globalLikeRankKey = function(){
    return "GLOBAL_LIKE_RANK";
};
```
点赞、取消点赞
比如玩家100对玩家101点赞：


```
redis.zadd(genLikeKey(101), LIKE_TYPE.LIKE, 100, function(err, ret){
    // set expire
    redis.expireat(genLikeKey(101), getExpireAt(), function(err, ret){});
    // 累计获赞数
    redis.zincrby(globalLikeRankKey(), 1, playerId, function(err, ret){
        // your logic here.
    });
});
```
取消赞：


```
redis.zadd(genLikeKey(101), LIKE_TYPE.UNLIKE, 100, function(err, ret){
    // set expire
    redis.expireat(genLikeKey(101), getExpireAt(), function(err, ret){});
    // 累计获赞数
    redis.zincrby(globalLikeRankKey(), -1, playerId, function(err, ret){
        // your logic here.
    });
});
```
点赞状态
比如查询玩家100对玩家101的点赞状态：


```
redis.zscore(genLikeKey(101), 100, function(err, likeType){
    // likeType==LIKE_TYPE.LIKE 表示赞过了。
});
```
点赞排行
比如查询玩家100的获赞排行：

```
// 取uid100得赞次数
redis.zscore(globalLikeRankKey(), 100, function(err, likeCnt){
    // deal err first.
    // 取小于赞次数的人数
    redis.zcount(globalLikeRankKey(), 0, likeCnt, function(err, cnt){
          // 取全部人数
        redis.zcard(globalLikeRankKey(), function(err, total){
            // 排名。
            var rank = Math.floor(Number(cnt)*100/Number(total));
        });
    });
});
```
sorted set性能
担心zcount操作的性能？直接看我在windows上测试的结果吧。

机器环境：
CPU:i5-3470@3.2GHz
RAM:8G
OS:64位windows 7
redis: v2.8.17

插入测试数据：


```
var kvs = [];
// 500w测试数据
var cnt = 5000000;
for(var i=0;i<cnt;i++){
    kvs.push([i, Math.floor(Math.random()*cnt)]);
}
async.eachLimit(kvs, 10000, function(kv, cb){
    redis.zadd('TEST_SS', kv[1], kv[0], function(err, ret){
        cb(null, null);
    });
}, function(err){
    console.log('finish insert.');
    process.exit(1);
});
```
测试zcount效率：

```
var scores = [];
for(var i=0;i<10000;i++){
    scores.push(i);
}

var st = Date.now();
//10000并发zcount操作
async.each(scores, function(score, cb){
    redis.zcount('TEST_SS', 0, score, function(err, ret){
        cb(null, null);
    });
}, function(err){
    console.log('cost ', Date.now()-st, ' ms.');
});
多次测试结果：
```
1
cost 237 ms.
# Redis 数据类型及操作详解

1. 字符串（存储数字、文本、Json 字符串）
   操作命令
   SET  GET  DEL  INCR  DECR  APPEND
2. 哈希类型（一般用于存储对象）
   HSET  HGET  HGETALL  HDEL  HINCRBY
3. 列表类型（任务队列，消息队列）
   LPUSH  RPUSH  LPOP  RPOP  LRANGE
4. 集合类型（适合于去重、交集、并集）
   SADD  SMEMBERS  SREM  SISMEMBER  SINTER
5. 有序集合（适用于排行榜、优先级队列，按分数排序）
   ZADD  ZRANGE  ZREM  ZINCRBY  ZREVRANGE
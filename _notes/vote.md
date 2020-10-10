# Vote on web page
## Background
- 对于每篇文章，每个user 可以选择 up
- 有200 up vote 的文章被视为有趣的文章
- suppose 每天会产生 1000 articles

## Requirement：
  把 50 篇有趣的文章放在首页至少1天

## Design
设计一个随着时间流逝而减少的score
score = 发布时间 + up votes * constant

时间使用 Unix时间（UTC时区 1970.1.1到目前为止，经过的秒数）

每天有 24 * 60 * 60 = 86400 秒
每个有趣文章需要 200 votes
constant =  86400 / 200 = 432, 也就是说每个vote 432分


Article information: user Redis Hash
```
  article id: {
                title:
                link:
                poster:
                time:
                votes
              }
```

用ZSET(有序set)维护两个 article list:
 article_id: time                 
 article_id: score
分别以 time和score 对文章排序


为了防止用户对同一文章重复投票，为每个文章记录一个已投票用户名单（set）

为了节省内存，一篇文章发布1周后，不能再接受投票，文章的评分会被固定，记录该文章已投票用户的set也会被删除


## Implement
投票
```
ONE_WEEK_IN_SECONDS = 7 * 86400
VOTE_SCORE = 432

def article_vote(conn, user, article):
  cutoff = time.time() - ONE_WEEK_IN_SECONDS
  # 检查文章的发布时间是否超过 1 周
  if conn.zscore('time:', article) < cutoff:
    return
  article_id = article.partition(':')[-1]
  if conn.sadd('voted:' + article_id, user):         # 把user放到文章的已投票用户set
    conn.zincrby('score:', article, VOTE_SCORE)      # 增加文章score
    conn.hincrby(article, 'votes', 1)                # 增加vote
```

发布文章
```
def post_article(conn, user, title, link):
  article_id = str(conn.incr('article:'))   # create new id
  voted = 'voted:' + article_id
  conn.sadd(voted, user)
  conn.expire(voted, ONE_WEEK_IN_SECONDS)

  now = time.time()
  article = 'article:' + article_id
  conn.hmset(article, {
      'title': title,
      'link': link,
      'poster': user,
      'time': now,
      'votes': 1,
    })
  conn.zadd('score:', article, now + VOTE_SCORE)
  conn.zadd('time:', article, now)
  return article_id
```





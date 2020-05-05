---
title: "Redis delayed tasks with go"
date: 2020-05-05T14:13:20+03:00
description: "Redis delayed tasks with go"
tags: ["go", "redis", "aws", "sqs", "docker"]
draft: false
---
# Use Redis as a queue of delayed task

If you work on a distributed system it is a high probability that you need some container, queue for sharing info/tasks between components of your system, or even instances of the same component. Let's assume that you need not only queueing but delaying as well.

Probably the first that comes to your mind is [Amazon Simple Queue Service (SQS)](https://aws.amazon.com/sqs/).  SQS eliminates the complexity and overhead associated with managing and operating message oriented middleware, and empowers developers to focus on differentiating work. But if you already use [Redis](https://redis.io/) and have some expertise in it you can consider other option: [delayed tasks](https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-4-task-queues/6-4-2-delayed-tasks/) with Redis.

## Delayed task

The main idea is to use ZSET, sorted set. Use time when the an item should be executed as a [score](https://redis.io/commands/zscore), the key. When you _enqueueing_ - you add item to ZSET with a score equal to the delayed time. When you _dequeueing_ you check if there is any available item and score of item is less then _now_. 

Another thing that you need is the distributed locks. It is also possible with Redis([description](https://redis.io/topics/distlock) and [algorithm](https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-2-distributed-locking/)).

![Sequence diagram](/img/redis-sqs.png)

## Go implementation

To implement it with I go need [go redis client](https://github.com/go-redis/redis) and [go redis distributed lock](https://github.com/bsm/redislock).

The struct needs redis _client_, _lock_, _key_ for ZSET, _batch_ - how many item we want dequeue as a maximum at once, and _ttl_ - when lock will be automatically released:
```
type RedisQueue struct {
	client *redis.ClusterClient
	locker *redislock.Client
	key    string
	batch  int
	ttl    time.Duration
}

func NewQueue(client *redis.ClusterClient, locker *redislock.Client, key string, batch int, ttl time.Duration) Queue {
	return &RedisQueue{client: client, locker: locker, key: key, batch: batch, ttl: ttl}
}
```
Enqueue:
```
func (rq *RedisQueue) Enqueue(uuid string, delay time.Duration) {
	_ = rq.client.ZAdd(rq.key, &redis.Z{Member: uuid, Score: float64(time.Now().Add(delay).Unix())})
}
```
Dequeue:
```
func (rq *RedisQueue) Dequeue() ([]Message, error) {
	var ms []Message
	start := int64(0)
	for i := rq.batch; i >= 0; {
		vals, err := rq.client.ZRangeWithScores(rq.key, start, start).Result()
		if err != nil {
			return nil, errors.Wrap(err, "cannot get range from zset")
		}
		if len(vals) == 0 || vals[0].Score > float64(time.Now().Unix()) {
			break
		}

		id := vals[0].Member.(string)
		lock := rq.acquireLock(id)
		if lock == nil {
			start++
			continue
		}
		ms = append(ms, Message{Message: id, OnProcessed: func() {
			_ = rq.client.ZRem(rq.key, id)
			if err := lock.Release(); err != nil {
				fmt.Printf("release lock erros = %+v\n", err)
			}
		}})
		start++
		i--

	}

	return ms, nil

}
```
You can test it with docker:
```
version: "2.1"
services:
  tests:
    image: golang:1.12
    working_dir: /go/src/github.com/sergii4/redis-delayed-queue
    volumes:
      - $PWD:/go/src/github.com/sergii4/redis-delayed-queue
      - go-modules:/go/pkg/mod # Put modules cache into a separate volume
    depends_on:
      - testredis
    command: ["/bin/sh", "-c", "GO111MODULE=on go test -v -timeout 30s"]

  testredis:
    image: grokzen/redis-cluster:latest
    logging:
      driver: "none"

volumes:
  go-modules: # Define the volume
```
Just run from terminal:
```
make test-redis-queue 
```

Check full code on [github](https://github.com/sergii4/redis-delayed-queue)

---
title: "使用scan命令进行redis数据迁移"
date: 2024-06-28T17:30:00+08:00
# weight: 1
# aliases: ["/first"]
tags:
- redis
- migrate
- golang
categories:
- redis
author: "WANG"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "通过scan命令进行redis数据迁移"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
# editPost:
#     URL: "https://github.com/<path_to_repo>/content"
#     Text: "Suggest Changes" # edit text
#     appendFilePath: true # to append file path to Edit link
---
## hashmap复制
有时新的redis集群并不需要全部的数据，只需要复制一部分数据到另一个redis集群，且这些数据更新并不频繁。
### 执行命令
```shell
nohup ./redis-cluster-migrate \
-s 172.18.206.144:7001,172.18.206.144:7002,172.18.83.165:7003,172.18.83.165:7004,172.18.25.248:7005,172.18.25.248:7006 \
-d 172.16.0.11:7001,172.16.0.11:7002,172.16.0.22:7003,172.16.0.22:7004,172.16.0.33:7005,172.16.0.33:7006 \
-m "token:*" \
-i 500 \
-t 100 &
```
### 完整代码
```golang
package main

import (
	"context"
	"flag"
	"log"
	"strings"
	"sync"
	"time"

	"github.com/redis/go-redis/v9"
)

func main() {
	src := flag.String("s", "", "src redis addrs")
	dest := flag.String("d", "", "dest redis addrs")
	match := flag.String("m", "", "match expr")
	interval := flag.Int64("i", 100, "scan interval, ms")
	count := flag.Int64("c", 1000, "scan count,redis cmd qps < count * 1000 / interval * 2")
	ttl := flag.Int64("t", 0, "dest ttl, hour")
	flag.Parse()

	srcAddrs := strings.Split(*src, ",")
	log.Println("src addrs:")
	for _, addr := range srcAddrs {
		log.Println(addr)
	}
	descAddrs := strings.Split(*dest, ",")
	log.Println("dest addrs:")
	for _, addr := range descAddrs {
		log.Println(addr)
	}
	redisSrc := redis.NewClusterClient(&redis.ClusterOptions{
		Addrs:            srcAddrs,
		PoolSize:         100,
		PoolTimeout:      30 * time.Second,
		DisableIndentity: true,
	})
	redisDest := redis.NewClusterClient(&redis.ClusterOptions{
		Addrs:            descAddrs,
		PoolSize:         100,
		PoolTimeout:      30 * time.Second,
		DisableIndentity: true, // < 7 err: Unknown subcommand or wrong number of arguments for 'setinfo'
	})
	ctx := context.Background()
	var wg sync.WaitGroup
	redisSrc.ForEachMaster(ctx, func(ctx context.Context, client *redis.Client) error {
		var cursor uint64
		var keys []string
		var err error
		for {
			time.Sleep(time.Duration(*interval) * time.Millisecond)
			var nextCursor uint64
			keys, nextCursor, err = client.Scan(ctx, cursor, *match, *count).Result()
			if err != nil {
				if err == redis.Nil {
					log.Println("no more keys")
					return err
				}
				log.Println("scan err: ", err.Error())
				return err
			}
			log.Printf("scan: cusor %d @ %s, keys: %d\n", cursor, client.Options().Addr, len(keys))
			cursor = nextCursor
			if len(keys) == 0 {
				continue
			}
			// log.Printf("scan keys: %v \n", keys)
			wg.Add(1)
			go func(src *redis.Client, dest *redis.ClusterClient, keys []string) {
				defer wg.Done()
				for _, k := range keys {
					value, err := client.HGetAll(ctx, k).Result()
					if err == redis.Nil {
						continue
					}
					if err != nil {
						log.Printf("get key: %s err: %s", k, err.Error())
						return
					}
					err = redisDest.HMSet(ctx, k, value).Err()
					if err != nil {
						log.Println("set err: ", err.Error(), k, value)
						return
					}
					if *ttl != 0 {
						redisDest.Expire(ctx, k, time.Duration(*ttl)*time.Hour)
					}
					// log.Printf("set key: %s, value: %s \n", k, value)
				}
			}(client, redisDest, keys)
			if cursor == 0 {
				log.Println("cursor end")
				return nil
			}
		}
	})
	wg.Wait()
	log.Println("all routine done")
}
```

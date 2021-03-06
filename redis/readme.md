redis 测试
========

工具
* redis 4.0.11（jemalloc）
* [vire](https://github.com/vipshop/vire) 多线程redis,包含了多线程的测试工具 [安装使用参考](https://deep011.github.io/vire-benchmark)
* redis-benchmark

因素
* 是否持久化
* 是否有从库
* 是否在docker中


主机角色（centos7)
* master 10.0.11.5（8core, 8g, 2.2GHz) 
* node1 10.0.12.13 (4core, 8g, 2.2GHz)
* node2 

### 默认配置

物理机部署，没有从库，配置没有优化

```
save 900 1  #rdb opening
save 300 10
save 60 10000

appendonly no  #aof closed
```

#### 本地测试

在 test 机器上


基础的单个cmd的操作，作为基础结果
```
redis-benchmark -t set,get -r 10000000 -q -h 127.0.0.1
```

|cmd/qps| 1 | 2 | 3| 4| 5
|--|--|--|--|--| --|
|set| 81037.28 | 80450.52| 70972.32| 67430.88| 80321.28|
|get| 82644.62 | 81699.35| 72939.46| 70821.53| 80321.28|

不是非常稳定，数量级基本没变化


使用pipline来测试, 16个指令放到一次pipeline
```
redis-benchmark -t set,get -r 10000000 -P 16 -q -h 127.0.0.1 
```

|cmd/qps| 1 | 2 | 3| 4| 5
|--|--|--|--|--| --|
|set| 534759.31 | 483091.78| 529100.56| 526315.81| 490196.09|
|get| 666666.62 | 819672.12| 813008.12| 819672.12| 781249.94|

性能比单个cmd操作好很多


vire-benchmark 4个线程来压测
```
./tests/vire-benchmark -t set,get -r 10000000 -q -h 127.0.0.1 -T 4
```

|cmd/qps| 1 | 2 | 3| 4| 5
|--|--|--|--|--| --|
|set| 92081.03 | 90859.53| 88503.41| 92064.07| 91449.47|
|get| 94241.82 | 94179.70| 94366.33| 95932.47| 104297.03|

压力上来了，果然好了一点

#### 不同机器测试

从 master ——> test(redis)

```
redis-benchmark -t set,get -r 10000000 -q -h 10.0.11.5
```

|cmd/qps| 1 | 2 | 3| 4| 5
|--|--|--|--|--| --|
|set| 67613.25 | 64267.35| 67294.75| 60901.34| 59276.82|
|get| 65189.05 | 61387.36| 64641.24| 67934.78| 68259.38|


提高total请求数
```
redis-benchmark -t set,get -r 10000000 -n 2000000 -q -h 10.0.11.5
```

|cmd/qps| 1 | 2 | 3|
|--|--|--|--|
|set| 60731.21 | 62916.82| 62272.32| 
|get| 62398.61 | 62654.68| 63635.50| 


pipeline 测试
```
redis-benchmark -t set,get -r 10000000 -P 16 -q -h 10.0.11.5
```

|cmd/qps| 1 | 2 | 3| 4| 5
|--|--|--|--|--| --|
|set| 462962.94 | 411522.62| 471698.12| 436681.22| 416666.69|
|get| 757575.75 | 625000.00| 568181.81| 662251.69| 595238.12|


vire-benchmark 4个线程来压测
```
./tests/vire-benchmark -t set,get -r 10000000 -q -h 10.0.11.5 -T 4
```

|cmd/qps| 1 | 2 | 3| 4| 5
|--|--|--|--|--| --|
|set| 80821.14 | 88136.79| 86378.16| 83444.59| 83416.75|
|get| 88526.91 | 85770.65| 86311.06| 88754.77| 85178.88|


### 关闭持久化

#### 不同机器测试

从 master ——> test(redis)

基础测试
```
redis-benchmark -t set,get -r 10000000 -q -h 10.0.11.5
```

|cmd/qps| 1 | 2 | 3| 4| 5
|--|--|--|--|--| --|
|set| 63897.76 | 60790.27| 60753.34| 62500.00| 59988.00|
|get| 66357.00 | 59488.40| 57603.69| 64599.48| 66357.00|


提高请求总数
```
redis-benchmark -t set,get -r 10000000 -n 2000000 -q -h 10.0.11.5
```

|cmd/qps| 1 | 2 | 3|
|--|--|--|--|
|set| 62185.19| 61739.83| 61701.73|
|get| 62086.73| 60990.48| 60716.46|


多线程来压测
```
./tests/vire-benchmark -t set,get -r 10000000 -q -h 10.0.11.5 -T 4
```

|cmd/qps| 1 | 2 | 3| 4| 5
|--|--|--|--|--| --|
|set| 83514.28 | 83070.27| 83640.01| 82569.56| 82692.46|
|get| 85026.79 | 83633.02| 84566.59| 82481.03| 85397.09|

并没发现rdb持久化对性能的多大影响

### 阅读

* [官方的测试文章](https://redis.io/topics/benchmarks) 对测试有影响的因素，网络带宽，cpu，内存速度，fork速度，unix domain socket带宽更大，客户端数目，内存分配工具，是否持久化，key大小





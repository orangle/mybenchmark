openresty
========


测试场景
* lua 直接输出小请求 （基准测试)
* lua 输出大请求(1m)
* 磁盘单文件
* 磁盘多文件
* http proxy

测试环境

* 单台 (nginx 8c8g) 4核用来nginx，其他用来给测试工具，之前使用的网络环境很快就达到瓶颈，无法做衡量
* 测试工具 wrk

测试变量
* nginx配置对于性能的影响
* 系统配置对于性能的影响

测试规划
* 首先基准测试进行优化，基本能达到能力范围内不能在优化的状态
* 在测试其他场景


### 基准测试

* openresty 采用rpm直接安装 1.13.6.1
* wrk 编译安装 

系统配置
```
$ ulimit -n
655350

$ sudo sysctl -a | grep fs.file
fs.file-max = 812399
fs.file-nr = 1728	0	812399
```


系统改动 `/etc/sysctl.conf`,改完后执行 `sudo sysctl -p`
```
net.ipv4.tcp_max_tw_buckets = 180000
net.core.somaxconn = 262144
net.core.netdev_max_backlog = 262144
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_max_syn_backlog = 262144
```

wrk install https://github.com/wg/wrk/wiki/Installing-wrk-on-Linux


### nginx.conf

单个进程模式
```
worker_processes  1;
```

openresty startup 
```
mkdir logs
sudo openresty -p `pwd` -c ./nginx.conf

curl 127.0.0.1:80
```


下面的配置变动都是基于基础配置 `nginx.conf`

##### 测试1

```
./wrk -t 10 -c 100 -d30s http://127.0.0.1:8808
Running 30s test @ http://127.0.0.1:8808
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.65ms  376.75us  17.45ms   89.35%
    Req/Sec     3.78k   349.19    11.60k    82.60%
  1129016 requests in 30.04s, 0.90GB read
Requests/sec:  37584.81
Transfer/sec:     30.68MB
```

单个process，单个cpu 100%

##### 测试2

调整配置，关闭access日志
```
http {
    access_log off;
}
```

测试
```
./wrk -t 10 -c 100 -d30s http://127.0.0.1:8808
Running 30s test @ http://127.0.0.1:8808
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.17ms  283.91us   9.80ms   83.83%
    Req/Sec     4.61k   384.50     5.65k    70.20%
  1376859 requests in 30.02s, 1.10GB read
Requests/sec:  45869.11
Transfer/sec:     37.44MB
```

写日志的io操作影响蛮大的

##### 测试3 

测试2的基础上，开启4个process
```
worker_processes  4;
```

测试
```
./wrk -t 10 -c 100 -d30s http://127.0.0.1:8808
Running 30s test @ http://127.0.0.1:8808
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   518.59us  166.75us  10.54ms   94.41%
    Req/Sec    19.29k     1.13k   29.61k    80.45%
  5772414 requests in 30.10s, 4.60GB read
Requests/sec: 191788.23
Transfer/sec:    156.56MB
```

4个进程下，openresty跑满4个核心，wrk用完将近3个核心


##### 测试4 

测试配置基础上，增加cpu绑定
```
worker_processes  4;
worker_cpu_affinity auto;
```

测试
```
 ./wrk -t 10 -c 400 -d30s http://127.0.0.1:8808
Running 30s test @ http://127.0.0.1:8808
  10 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.99ms  366.20us  14.95ms   83.43%
    Req/Sec    20.14k     1.53k   65.89k    91.67%
  6016507 requests in 30.10s, 4.80GB read
Requests/sec: 199891.54
Transfer/sec:    163.17MB
```

和测试3基本一个测试结果。

##### 测试5

测试4的基础上，配置改动

```
events {
    multi_accept off;
    accept_mutex off;
    ...
}
```

继续压测
```
./wrk -t 10 -c 400 -d30s http://127.0.0.1:8808
Running 30s test @ http://127.0.0.1:8808
  10 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.09ms  724.85us  54.01ms   97.73%
    Req/Sec    19.28k     1.39k   58.19k    92.04%
  5763672 requests in 30.10s, 4.59GB read
Requests/sec: 191485.59
Transfer/sec:    156.31MB
```

##### 测试6 

测试5的配置基础上，process 改成6

```
./wrk -t 10 -c 400 -d30s http://127.0.0.1:8808
Running 30s test @ http://127.0.0.1:8808
  10 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.08ms    2.50ms  37.00ms   86.23%
    Req/Sec    26.67k    13.36k   69.81k    58.10%
  7937413 requests in 30.08s, 6.33GB read
Requests/sec: 263914.74
Transfer/sec:    215.43MB
```

wrk + nginx 把所有核心跑满，在当前cpu下应该已经是达到了性能极限。


### lua 输出1M请求

直接使用4核心来测试

```
sudo openresty -p `pwd` -c ./nginx2.conf

curl 127.0.0.1:8808 -i
```

#### nginx2.conf


##### 测试1 

配置没改动

```
./wrk -t 10 -c 400 -d30s http://127.0.0.1:8808
Running 30s test @ http://127.0.0.1:8808
  10 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    45.52ms   57.03ms   1.16s    97.90%
    Req/Sec     1.02k   163.84     2.07k    79.59%
  303052 requests in 30.08s, 289.06GB read
Requests/sec:  10073.41
Transfer/sec:      9.61GB
```

cpu也是跑满了，传输也会用cpu？

##### 测试2 

增加配置 
```
http {
    ...
    tcp_nodelay on;
    tcp_nopush on;
    gzip on;
    gzip_min_length 1000;
    gzip_types text/plain;
    ...
}
```

测试
```
./wrk -t 30 -c 400 -d30s http://127.0.0.1:8808
Running 30s test @ http://127.0.0.1:8808
  30 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    41.50ms   59.52ms   1.36s    98.24%
    Req/Sec   366.94     70.75     2.16k    91.58%
  326477 requests in 30.09s, 311.40GB read
Requests/sec:  10848.40
Transfer/sec:     10.35GB
```

##### 测试3 

测试2 基础上增加process 到6

```
./wrk -t 30 -c 400 -d30s http://127.0.0.1:8808
Running 30s test @ http://127.0.0.1:8808
  30 threads and 400 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    55.95ms   41.79ms   1.04s    94.68%
    Req/Sec   242.64     52.90   788.00     76.48%
  213892 requests in 30.10s, 204.07GB read
Requests/sec:   7106.23
Transfer/sec:      6.78GB
```

wrk的影响导致nginx没有足够的cpu资源，反而更慢了。


单机，纯lua调用的情况，没有重系统调用以及网络消耗，cpu能全部利用上啊，这种情况就很难从外部来优化了，算法或者更底层的支持才能发挥更好的性能。


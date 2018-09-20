openresty
========


测试场景
* lua 直接输出小请求 （基准测试)
* lua 输出大请求
* 磁盘单文件
* 磁盘多文件
* http proxy

测试环境

* 2台Centos机器 N(nginx 8c8g) W(wrk 4c8g)
* 万兆网卡，测试发包速率在 3Gb/s 左右
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

wrk install https://github.com/wg/wrk/wiki/Installing-wrk-on-Linux


#### nginx.conf

单个进程模式
```
worker_processes  1;
```

openresty startup 
```
sudo openresty -p `pwd` -c ./nginx.conf

curl 127.0.0.1:80
```

下面的配置变动都是机遇基础配置 `nginx.conf`

##### 测试1
```
$ ./wrk -t 10 -c 100 -d30s http://10.0.11.5
Running 30s test @ http://10.0.11.5
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.63ms  436.11us  16.79ms   90.63%
    Req/Sec     3.82k   463.77    24.17k    94.17%
  1140263 requests in 30.10s, 0.91GB read
Requests/sec:  37883.84
Transfer/sec:     30.92MB
```

单个process，单个cpu 100%，因为cpu已经是瓶颈，所以调节其他压测参数也没有反应。

##### 测试2

**调节process为4** 

```
$ ./wrk -t 10 -c 100 -d30s http://10.0.11.5
Running 30s test @ http://10.0.11.5
  10 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.96ms  413.25us  12.46ms   78.46%
    Req/Sec     5.02k   633.33     7.74k    73.00%
  1498493 requests in 30.03s, 1.19GB read
Requests/sec:  49899.04
Transfer/sec:     40.73MB
```

4个进程的情况下，提升不明显，cpu使用率在 200% 左右，已经不是瓶颈。

调整连接数在测试
```
$ ./wrk -t 10 -c 1000 -d30s http://10.0.11.5
Running 30s test @ http://10.0.11.5
  10 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    22.99ms   36.13ms 912.93ms   95.42%
    Req/Sec     5.86k     1.13k   10.06k    66.60%
  1750054 requests in 30.03s, 1.40GB read
Requests/sec:  58268.19
Transfer/sec:     47.56MB
```

有所提升，如果在增加已经无法看到提升。 后记：7核的情况下跟4核几乎一样的结果


##### 测试3

增加配置 `worker_cpu_affinity auto;`, 没有实质性提升
修改为 `worker_cpu_affinity 00000001 00000010 00000100 00001000`, 没有实质性提升

##### 测试4

配置改动

```
worker_processes  4;

events {
    multi_accept off;
    accept_mutex off;
    ...
}
```

继续压测
```
./wrk -t 10 -c 1000 -d30s http://10.0.11.5
```

没有显著提升

##### 测试5

```
worker_processes  4;

events {
    multi_accept off;
    accept_mutex off;
    ...
}

http {
    access_log off;
}
```

测试
```
./wrk -t 10 -c 1000 -d30s http://10.0.11.5
```

无实质性提升

##### 测试6

配置改动
```
worker_processes  4;
```

系统改动nginx机器 `/etc/sysctl.conf`, `sudo sysctl -p`
```
net.ipv4.tcp_max_tw_buckets = 180000
net.core.somaxconn = 262144
net.core.netdev_max_backlog = 262144
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_max_syn_backlog = 262144
```

测试 
```
./wrk -t 20 -c 400 -d30s http://10.0.11.5
```

仍然没有实质性提升，这有点沮丧了。

##### 测试7

4n个processes, 在本地测试

```
./wrk -t 10 -c 1000 -d30s http://10.0.11.5
Running 30s test @ http://10.0.11.5
  10 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    60.92ms  101.42ms   1.99s    82.02%
    Req/Sec    15.34k     2.98k   37.18k    71.87%
  4573647 requests in 30.02s, 3.65GB read
  Socket errors: connect 0, read 0, write 0, timeout 8
Requests/sec: 152374.56
Transfer/sec:    124.38MB
```

这样看下来，果然还是网络问题，本地测试性能高出2倍，之前只顾着优化nginx一端的服务器，忽略了压测端的优化，以及2边网络的问题。


##### 测试8 

在测试6的基础上优化，压测端，不如怎样优化？


尝试 `/etc/sysctl.conf`, `sudo sysctl -p`
```
net.ipv4.tcp_max_tw_buckets = 180000
net.core.somaxconn = 262144
net.core.netdev_max_backlog = 262144
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_max_syn_backlog = 262144
```

测试后没有什么提升, 最好的结果也是6w/tps，因为是万兆网卡，可以排除带宽瓶颈, pps也没达到瓶颈。



























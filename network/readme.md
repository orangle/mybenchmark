# 网络相关的测试

* centos

### 网络带宽

#### nc 
macos 

```
#A
# nc -vvv -l -p 10008 > /dev/null 

#B
# dd if=/dev/zero bs=1M count=1000 | nc localhost 10008

记录了1000+0 的读入
记录了1000+0 的写出
1048576000字节(1.0 GB)已复制，5.99562 秒，175 MB/秒
```

理论速度和网卡速度相当


### iperf 
centos 


* https://www.alibabacloud.com/help/zh/faq-detail/55757.htm
* https://iperf.fr/iperf-doc.php 


！！！iperf有两个常用的版本 iperf, iperf3，对比可以[参考](http://fasterdata.es.net/performance-testing/network-troubleshooting-tools/throughput-tool-comparision/)


#### iperf 

客户端和服务都需要安装

选项
```
-f <kmKM>    报告输出格式。 [kmKM]   format to report: Kbits, Mbits, KBytes, MBytes
-i <sec>    在周期性报告带宽之间暂停n秒。如周期是10s，则-i指定为2，则每隔2秒报告一次带宽测试情况,则共计报告5次
-p    设置服务端监听的端口，默认是5001
-u    使用UDP协议测试
-w n<K/M>   指定TCP窗口大小
-m    输出MTU大小
-M    设置MTU大小
-o <filename>    结果输出至文件
-t  运行时间 s
```


tcp 测试
```
#install 
$ sudo yum install iperf

服务端
$ iperf -s -i 2


客户端
$ iperf -c 10.0.11.5 -i 2
------------------------------------------------------------
Client connecting to 10.0.11.5, TCP port 5001
TCP window size: 85.0 KByte (default)
------------------------------------------------------------
[  3] local 10.0.12.13 port 33912 connected with 10.0.11.5 port 5001
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 2.0 sec   741 MBytes  3.11 Gbits/sec
[  3]  2.0- 4.0 sec   752 MBytes  3.15 Gbits/sec
[  3]  4.0- 6.0 sec   736 MBytes  3.08 Gbits/sec
[  3]  6.0- 8.0 sec   770 MBytes  3.23 Gbits/sec
[  3]  8.0-10.0 sec   792 MBytes  3.32 Gbits/sec
[  3]  0.0-10.0 sec  3.70 GBytes  3.18 Gbits/sec

10个客户端并行发送
iperf -c 10.0.11.5 -P 10 -i 2
```


更常用的是 udp测试

```
$ iperf -s -u

$ iperf -u -c 10.0.11.5 -t 20 -i 2 -b 10000M   
```
可以参考丢包来调整，-b 是发包的带宽，如果过大，丢包率就很大了，丢包很小的时候基本和实际带宽差不多了


### iperf3

可以使用yum直接安装，或者源码安装新版本，测试下来没发现很大的区别


```
git clone https://github.com/esnet/iperf.git
cd iperf/
./configure
make
sudo make install 
```

测试, server
```
/usr/local/bin/iperf3 -s -A 4
```

client
```
$ /usr/local/bin/iperf3 -c 10.0.11.5
Connecting to host 10.0.11.5, port 5201
[  5] local 10.0.12.13 port 51468 connected to 10.0.11.5 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   336 MBytes  2.81 Gbits/sec   29    717 KBytes
[  5]   1.00-2.00   sec   352 MBytes  2.96 Gbits/sec  183    816 KBytes
[  5]   2.00-3.00   sec   320 MBytes  2.68 Gbits/sec   24   1.05 MBytes
[  5]   3.00-4.00   sec   326 MBytes  2.74 Gbits/sec  312   1.22 MBytes
[  5]   4.00-5.00   sec   348 MBytes  2.92 Gbits/sec  147   1.41 MBytes
[  5]   5.00-6.00   sec   402 MBytes  3.38 Gbits/sec    0   1.61 MBytes
[  5]   6.00-7.00   sec   365 MBytes  3.06 Gbits/sec  525   1.30 MBytes
[  5]   7.00-8.00   sec   265 MBytes  2.22 Gbits/sec  2786   1.15 MBytes
[  5]   8.00-9.00   sec   336 MBytes  2.82 Gbits/sec   26   1.33 MBytes
[  5]   9.00-10.00  sec   334 MBytes  2.80 Gbits/sec    0   1.50 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  3.31 GBytes  2.84 Gbits/sec  4032             sender
[  5]   0.00-10.04  sec  3.30 GBytes  2.83 Gbits/sec                  receiver
```

10个客户端并行发送
```
/usr/local/bin/iperf3 -c 10.0.11.5 -P 10
```
结果稍微高一些

### PPS测试

packets per second 测试发包速度 http://techblog.cloudperf.net/2016/05/2-million-packets-per-second-on-public.html

测量方式
```
sar -n DEV 1
```








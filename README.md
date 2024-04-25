# TCP 和 UDP 谁更快？

起因是一条[关于 TCP 吞吐量比 UDP 好的推文](https://twitter.com/liumengxinfly/status/1783135883920883741)引起了一些争议。空对空谈没有什么太大意义，干脆做个简单的测试跑组数据，大家可以添加自己的测试来检查不同场景 TCP 和 UDP 性能的对比。

# 步骤
这个脚本依赖 kind 和 docker，会在一台机器上启动一个两节点的 Kubernetes 集群，每个节点运行一个 qperf 的 Pod 做性能测试。

1. 初始化 Kubernetes 集群，并运行 qperf:
```bash
kind create cluster --config kind.yml
kubectl apply -f perf.yml
```

2. 获取 Pod IP，并运行 qperf 测试分别测试 1 字节的小包和 16k 的大包：

```bash
# kubectl get pod -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
qperf-57rv5   1/1     Running   0          53m   172.18.0.3   kind-worker          <none>           <none>
qperf-cgw6v   1/1     Running   0          52m   172.18.0.2   kind-control-plane   <none>           <none>

# kubectl exec -it qperf-57rv5 bash
kind-worker:/kube-ovn# qperf -t 10 172.18.0.2 -ub -oo msg_size:1 -vu tcp_lat tcp_bw udp_lat udp_bw
tcp_lat:
    latency   =  19.8 us
    msg_size  =     1 bytes
    time      =    10 sec
tcp_bw:
    bw        =  6.41 Mb/sec
    msg_size  =     1 bytes
    time      =    10 sec
udp_lat:
    latency   =  17.1 us
    msg_size  =     1 bytes
    time      =    10 sec
udp_bw:
    send_bw   =  860 Kb/sec
    recv_bw   =  859 Kb/sec
    msg_size  =    1 bytes
    time      =   10 sec
kind-worker:/kube-ovn# qperf -t 10 172.18.0.2 -ub -oo msg_size:16k -vu tcp_lat tcp_bw udp_lat udp_bw
tcp_lat:
    latency   =  27.7 us
    msg_size  =    16 KB
    time      =    10 sec
tcp_bw:
    bw        =  6.94 Gb/sec
    msg_size  =    16 KB
    time      =    10 sec
udp_lat:
    latency   =  43.5 us
    msg_size  =    16 KB
    time      =    10 sec
udp_bw:
    send_bw   =  3.91 Gb/sec
    recv_bw   =   3.5 Gb/sec
    msg_size  =    16 KB
    time      =    10 sec
```
# 结果

只看吞吐量的情况:
 
1 字节小包下 TCP 的吞吐是 6.41 Mb/sec, UDP 是 860 Kb/sec。

16k 字节大包下 TCP 的吞吐是 6.94 Gb/sec, UDP 是 3.91 Gb/sec。 

这两个情况下 TCP 分别能用到网卡的小包聚合和大包切片的能力，性能会有比较大的提升。数据包大小在这两个范围内的情况下性能差距会小一些，不过 TCP 吞吐量通常也优于 UDP。当然用于 TCP 和 UDP 协议不同，收发包的应用逻辑无法做到完全一致，不过在现有生态下 TCP 的性能会更好一些。
---
title: k8s学习.md
categories: aqshing
Email: jdbc.cc <work@jdbc.cc>
tags:
date: 2023-10-09 16:59:55
changed: 2023-11-13 18:19:11
---
##
```
# 查看pod信息
kubectl get pods -n kube-system

kubectl apply -f calico.yaml
# 获取Kubernetes集群中的所有部署（Deployments）资源
kubectl get deploy -o wide

k describe po calico-kube -n kube-system
# 看错误信息
journalctl -xefu kubelet
# 查看系统日志磁盘占用
journalctl --disk-usage
# 查看磁盘上的日志文件大小
ls -lh /run/log/journal/*/system*
# 删除磁盘上的日志文件大小
rm -rf /run/log/journal/*/system*
# 清空 journalctl 中的所有日志
sudo journalctl --vacuum-size=0
# 清空特定单元（unit）的日志
sudo journalctl --vacuum-size=0 -u kubelet
```
``` bash
# 获取pod信息
[root@node1 ~]# kubectl get pods -n kube-system
NAME                            READY   STATUS    RESTARTS   AGE
coredns-6d8c4cb4d-ttv4t         0/1     Pending   0          16m
coredns-6d8c4cb4d-x5bw7         0/1     Pending   0          16m
etcd-node1                      1/1     Running   1          16m
kube-apiserver-node1            1/1     Running   1          16m
kube-controller-manager-node1   1/1     Running   1          16m
kube-proxy-cpfjv                1/1     Running   0          16m
kube-proxy-pkplh                1/1     Running   0          7m50s
kube-proxy-q796m                1/1     Running   0          8m1s
kube-scheduler-node1            1/1     Running   1          16m
```
## 常见错误
```
报错信息:
network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
解决办法：\
sudo mkdir -p /etc/cni/net.d
sudo sh -c 'cat << EOF > /etc/cni/net.d/01-cri-dockerd.json
{
  "cniVersion": "0.4.0",
  "name": "dbnet",
  "type": "bridge",
  "bridge": "cni0",
  "ipam": {
    "type": "host-local",
    "subnet": "10.1.0.0/16",
    "gateway": "10.1.0.1"
  }
}
EOF'
相关链接：
https://medium.com/@ElieXU/run-kubelet-1-24-standalone-using-docker-engine-part-2-ae3f9a20882a
```

```
报错信息:
Unable to read config path" err="path does not exist, ignoring" path="/etc/kubernetes/manifests
解决办法：
手动创建 mkdir -p /etc/kubernetes/manifests目录
相关链接：
https://github.com/kubernetes/kubeadm/issues/1345
```

```
[linux1@vps ~]$ docker info | grep -i ' Cgroup Driver'
ERROR: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/info": dial unix /var/run/docker.sock: connect: permission denied
errors pretty printing info
```

```
error getting RW layer size for container ID
```

```
"Failed to get the info of the filesystem with mountpoint" err="unable to find data in memory cache" mountpoint="/var/lib/docker"
"Image garbage collection failed once. Stats initialization may not have completed yet" err="invalid capacity 0 on image filesystem"
```

```
  Warning  Unhealthy  48s (x2 over 58s)  kubelet            Liveness probe failed: initialized to false
  Warning  Unhealthy  18s (x3 over 38s)  kubelet            Liveness probe failed: Error initializing datastore: Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": dial tcp 10.96.0.1:443: i/o timeout
  Warning  Unhealthy  18s (x3 over 38s)  kubelet            Readiness probe failed: Error initializing datastore: Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": dial tcp 10.96.0.1:443: i/o timeout
  Normal   Created    17s (x2 over 77s)  kubelet            Created container calico-kube-controllers
  Normal   Pulled     17s (x2 over 78s)  kubelet            Container image "docker.io/calico/kube-controllers:v3.26.1" already present on machine
  Normal   Started    16s (x2 over 77s)  kubelet            Started container calico-kube-controllers
  Warning  Unhealthy  8s (x9 over 77s)   kubelet            Readiness probe failed: initialized to false
```

```
coredns不可用
kubectl describe po -n kube-system calico-kube-controllers-57c6dcfb5b-skg69
 Warning  Unhealthy         16m (x3 over 16m)     kubelet            Liveness probe failed: Error initializing datastore: Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": dial tcp 10.96.0.1:443: i/o timeout

kubectl describe po -n kube-system coredns-64897985d-x6xg4
 Warning  Unhealthy         5s (x5 over 35s)       kubelet            Readiness probe failed: HTTP probe failed with statuscode: 503

kubectl logs -f -n kube-system coredns-64897985d-x6xg4
[ERROR] plugin/errors: 2 1394415300835651230.8630148154983443782. HINFO: read udp 10.144.166.129:44256->8.8.8.8:53: i/o timeout
解决办法：
iptables -P FORWARD ACCEPT && systemctl restart kubelet
相关链接：
https://blog.csdn.net/JosephThatwho/article/details/111475422
```

```
Ubuntu 22.04 使用私钥登录时提示 server refused our key
Disconnected: No supported authentication methods available (server sent: publickey)
解决方法
方法一： 在配置中加上 rsa
vim /etc/ssh/sshd_config

# 添加配置
PubkeyAcceptedKeyTypes +ssh-rsa

# 重启 SSH
systemctl restart sshd
复制
方法二：（推荐）
# 在生成密钥对时使用更加安全的算法
ssh-keygen -t ed25519
相关链接：
https://cloud.tencent.com/developer/article/2091570
```
```
Ubuntu 取消ssh登录时打印的欢迎信息
解决方法
方法一：(推荐)
包含Last login: Mon Nov 13 02:50:46 2023 from 192.168.1.1输出
sudo chmod -x /etc/update-motd.d/*
方法二: (没有任何输出)
touch $HOME/.hushlogin
相关链接：
https://blog.csdn.net/u012939880/article/details/106474876
```
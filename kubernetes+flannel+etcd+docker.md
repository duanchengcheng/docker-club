#kubernetes+flannel+etcd+docker 安装配置方案
##系统描述


|IP|主机名|职责|
|---|---|---|
|10.230.13.141|registry.docker|docker registry|
|10.230.13.142|kube-master|Etcd、Kube-master|
|192.168.218.227|kube-node1|Node1|
|192.168.218.153|kube-node2|Node2|

##下载flannel、etcd、kubernetes
1、下载 flannel(所有的文件均放在/opt/k8s目录下,master和node均需安装flannel)
```
wget https://github.com/coreos/flannel/releases/download/v0.5.4/flannel-0.5.4-linux-amd64.tar.gz
tar -zvxf flannel-0.5.4-linux-amd64.tar.gz
cd flannel-0.5.4
cp -rp flanneld /usr/local/bin/
cp -rp mk-docker-opts.sh /usr/local/bin/
```
2、下载 etcd
```
curl -L  https://github.com/coreos/etcd/releases/download/v2.2.1/etcd-v2.2.1-linux-amd64.tar.gz -o etcd-v2.2.1-linux-amd64.tar.gz
tar xzvf etcd-v2.2.1-linux-amd64.tar.gz
cd etcd-v2.2.1-linux-amd64
cp -rp etcd* /usr/local/bin/
```
3、下载 kubernetes
```
wget https://github.com/kubernetes/kubernetes/releases/download/v1.0.6/kubernetes.tar.gz
cd /home/k8s/
tar -zvxf kubernetes.tar.gz
cd /home/k8s/kubernetes/server/
tar -zvxf kubernetes-server-linux-amd64.tar.gz
cd /home/k8s/kubernetes/server/kubernetes/server/bin/
cp -rp hyperkube /usr/local/bin/
cp -rp kube-apiserver /usr/local/bin/
cp -rp kube-controller-manager /usr/local/bin/
cp -rp kubectl /usr/local/bin/
cp -rp kubelet /usr/local/bin/
cp -rp kube-proxy /usr/local/bin/
cp -rp kubernetes /usr/local/bin/
cp -rp kube-scheduler /usr/local/bin/
将kubelet文件和kube-proxy文件复制至node服务器下的/usr/local/bin/目录
rsync -avzP /usr/local/bin/kubelet root@192.168.218.255:/usr/local/bin/
rsync -avzP /usr/local/bin/kube-proxy root@192.168.218.255:/usr/local/bin/
rsync -avzP /usr/local/bin/kubelet root@192.168.218.153:/usr/local/bin/
rsync -avzP /usr/local/bin/kube-proxy root@192.168.218.153:/usr/local/bin/
```


##配置防火墙规则
1、master执行
```
iptables -I INPUT -s 192.168.218.255/24 -p tcp --dport 8080 -j ACCEPT
iptables -I INPUT -s 192.168.218.255/24 -p tcp --dport 4001 -j ACCEPT
iptables -I INPUT -s 192.168.218.255/24 -p tcp --dport 7001 -j ACCEPT
iptables -I INPUT -s 192.168.218.255/24 -p tcp --dport 8888 -j ACCEPT
iptables -I INPUT -s 192.168.218.153/24 -p tcp --dport 8080 -j ACCEPT
iptables -I INPUT -s 192.168.218.153/24 -p tcp --dport 4001 -j ACCEPT
iptables -I INPUT -s 192.168.218.153/24 -p tcp --dport 7001 -j ACCEPT
iptables -I INPUT -s 192.168.218.153/24 -p tcp --dport 8888 -j ACCEPT
```

2、node1执行
```
iptables -I INPUT -s 10.230.13.142/24 -p tcp --dport 10250 -j ACCEPT
iptables -I INPUT -s 10.230.13.142/24 -p udp --dport 8285 -j ACCEPT
```
3、node2执行
```
iptables -I INPUT -s 10.230.13.142/24 -p tcp --dport 10250 -j ACCEPT
iptables -I INPUT -s 10.230.13.142/24 -p udp --dport 8285 -j ACCEPT
```

##启动etcd、flannel、docker
1、启动master上的etcd
```
nohup etcd --listen-peer-urls http://0.0.0.0:7001 --data-dir=/var/lib/etcd \
--listen-client-urls http://0.0.0.0:4001 \ 
--advertise-client-urls http://kube-master:4001 >> /var/log/etcd.log 2>&1 &
```
2、启动flannel
```
mkdir -p /var/log/flanneld;
nohup flanneld --listen=0.0.0.0:8888 >> /var/log/flanneld/flanneld.log 2>&1 &
etcdctl set /coreos.com/network/node1/config '{ "Network": "10.230.14.1/24" }'
etcdctl set /coreos.com/network/node2/config '{ "Network": "10.230.15.1/24" }'
node1、node2上执行：
mkdir -p /var/log/flanneld;
nohup flanneld -etcd-endpoints=http://kube-master:4001 -remote=kube-master:8888 >> /var/log/flanneld/flanenlnode.log 2>&1 &
node1执行如下：
source /run/flannel/node1/subnet.env
node2执行如下：
source /run/flannel/node2/subnet.env
```
3、启动docker（需先安装docker）
```
删掉docker0网卡后手动启动docker
ip a del 172.17.0.1/16 dev docker0
node1执行如下：
docker -d -H unix:///var/run/docker.sock >> /var/log/dockerd --bip=10.230.14.1/24 --mtu=1472 & 
node2执行如下：
docker -d -H unix:///var/run/docker.sock >> /var/log/dockerd --bip=10.230.15.1/24 --mtu=1472 & 
systemctl start docker.service;
```

##启动kubernetes
```
master上启动kube-apiserver、kuber-controller-manager、kube-scheduler(第一启动kube-apiserver时可能没有key和crt，去掉配置重启一次即可)
 
mkdir -p /var/log/kubernetes;
 
nohup kube-apiserver --logtostderr=true --v=0 --etcd_servers=http://kube-master:4001 \
--address=0.0.0.0 --allow-privileged=true \
--service-cluster-ip-range=10.254.0.0/24 \
--tls-cert-file=/var/run/kubernetes/apiserver.crt \
--tls-private-key-file=/var/run/kubernetes/apiserver.key \
--admission_control=LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota >> /var/log/kubernetes/kube-apiserver.log 2>&1 &
 
nohup kube-controller-manager --logtostderr=true --master=http://kube-matser:8080 \
--v=0 --node-monitor-grace-period=10s --pod-eviction-timeout=10s \
--root-ca-file=/var/run/kubernetes/apiserver.crt \
--service_account_private_key_file=/var/run/kubernetes/apiserver.key \
>> /var/log/kubernetes/kube-controller-manager.log 2>&1 &
 
nohup kube-scheduler --logtostderr=true --v=0 --master=http://kube-master:8080 >> /var/log/kubernetes/kube-scheduler.log 2>&1 &

```
- NamespaceLifecycle,NamespaceExists配置了的话会出现Namespace kube-system does not exist错误（需要自行创建kube-system的namespace）

##启动node1、node2上的kubectl、kube-proxy
```
mkdir -p /var/log/kubernetes;
 
nohup kubelet --logtostderr=true --v=0 --api_servers=http://kube-master:8080 --address=0.0.0.0 --allow_privileged=false \
--tls-cert-file=/var/run/kubernetes/kubelet.crt --tls-private-key-file=/var/run/kubernetes/kubelet.key \
--config=/etc/kubernetes/manifests/ \
--cluster_dns=10.254.0.120 --cluster_domain=kube-master> /var/log/kubernetes/kubelet.log 2>&1 &
 
nohup kube-proxy --logtostderr=true --v=0 --master=http://kube-master:8080 > /var/log/kubernetes/kube-proxy.log 2>&1 &
```

##通过kubectl运行第一个pod
```
提前下载好nginx的image和pause的image，否则会报错（pause负责管理pod的网络等相关事务）
docker pull nginx;docker pull gcr.io/google_containers/pause:0.8.0
谷歌被墙了，可以通过翻墙或访问国内的docker镜像站点（如时速云、灵雀云等）下载下来改名
kubectl run my-nginx --image=nginx --replicas=2 --port=80

kubectl get pods -o wide

分别获得两个容器的ip(kubectl exec pod名称 执行命令)
kubectl exec my-nginx-wlwpw ip a|grep 10.230
```

##容器之间的互访
```
在master和node上均执行以下语句（10.1对应etcd中设置的值）
iptables -I FORWARD -s 10.230.0.0/16 -j ACCEPT
访问创建好的2个nginx容器。
```

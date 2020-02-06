# Centos7.4安装部署k8s

## 一、准备工作

Kubernetes集群组件:

- etcd 一个高可用的K/V键值对存储和服务发现系统
- flannel 实现夸主机的容器网络的通信
- kube-apiserver 提供kubernetes集群的API调用
- kube-controller-manager 确保集群服务
- kube-scheduler 调度容器，分配到Node
- kubelet 在Node节点上按照配置文件中定义的容器规格启动容器
- kube-proxy 提供网络代理服务

三个centos7系统的虚拟机：

| 节点   | IP地址      |
| ------ | ----------- |
| master | 10.10.10.14 |
| node1  | 10.10.10.15 |
| node2  | 10.10.10.16 |

更改Hostname为 master、node1、node2，配置所有测试机的/etc/hosts文件

```
[root@master ~]# cat /etc/hosts
10.10.10.14 master  etcd  node14
10.10.10.15 node1   node15
10.10.10.16 node2   node16
```

关闭CentOS7自带的防火墙服务

```s
[root@master ~]# firewall-cmd --state
[root@master ~]# systemctl stop firewalld.service
[root@master ~]# systemctl disable firewalld.service 
```

## 二、安装

### 1.docker安装

#### 执行指令

```
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum -y install docker-ce
systemctl start docker
```

#### 更改镜像源

在文件/etc/docker/daemon.json中添加以下内容(如果没有则创建):

```json
{
"registry-mirrors": ["https://registry.docker-cn.com"]
}
```

详细一点的：

```json
{
    "oom-score-adjust": -1000,
    "log-driver": "json-file",
    "log-opts": {
    "max-size": "100m",
    "max-file": "3"
    },
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 10,
    "registry-mirrors": ["https://c0fbbcbf3bbf4aaebbdead24fee9d46c.mirror.swr.myhuaweicloud.com","https://h90g7as2.mirror.aliyuncs.com"],
    "storage-driver": "overlay2",
    "storage-opts": [
    "overlay2.override_kernel_check=true"
    ]
}
```

#### 常用命令

```
docker version	//查看版本
docker info	//查看docker基本信息

systemctl start docker	//开启
systemctl restart docker
systemctl status docker	//查看状态
systemctl stop docker	//停止

systemctl enable docker//开机自动启动
systemctl disenable docker
```

### 2.安装

在master主机上：

```
yum install -y etcd kubernetes-master ntp flannel
```

node1上：

```
yum install -y kubernetes-node ntp flannel docker
```

时间校对，所有主机上：

```
systemctl start ntpd;systemctl enable ntpd
ntpdate ntp1.aliyun.com
hwclock -w
```

#### 配置etcd服务器

```
[root@master ~]# grep -v '^#' /etc/etcd/etcd.conf
    ETCD_NAME=default
    ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
    ETCD_LISTEN_CLIENT_URLS="http://localhost:2379,http://10.10.10.14:2379"
    ETCD_ADVERTISE_CLIENT_URLS="http://10.10.10.14:2379"
```

​	启动服务

```
systemctl start etcd;systemctl enable etcd
```

检查etcd cluster状态

```
[root@master ~]# etcdctl cluster-health   
member 8e9e05c52164694d is healthy: got healthy result from http://10.10.10.14:2379
cluster is healthy
```

检查etcd集群成员列表，这里只有一台

```
[root@master ~]# etcdctl member list
8e9e05c52164694d: name=default peerURLs=http://localhost:2380 clientURLs=http://10.10.10.14:2379 isLeader=true
```

#### 配置master服务器

##### 1) 配置kube-apiserver配置文件

[root@master ~]# grep -v '^#' /etc/kubernetes/config

```
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://10.10.10.14:8080"
```

[root@master ~]# grep -v '^#' /etc/kubernetes/apiserver

```
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"
KUBE_ETCD_SERVERS="--etcd-servers=http://10.10.10.14:2379"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
KUBE_ADMISSION_CONTROL="--admission-control=AlwaysAdmit"
KUBE_API_ARGS=""
```

##### 2) 配置kube-controller-manager配置文件

[root@master ~]# grep -v '^#' /etc/kubernetes/controller-manager

```
KUBE_CONTROLLER_MANAGER_ARGS=""
```

##### 3) 配置kube-scheduler配置文件

[root@master ~]# grep -v '^#' /etc/kubernetes/scheduler

```
KUBE_SCHEDULER_ARGS="--address=0.0.0.0"
```

##### 4) 启动服务

```
for i in  kube-apiserver kube-controller-manager kube-scheduler;do systemctl restart $i; systemctl enable $i;done
```

#### 配置node1节点服务器

##### 1) 配置etcd

```
[root@master ~]# etcdctl set /atomic.io/network/config '{"Network": "172.16.0.0/16"}'
{"Network": "172.16.0.0/16"}
```

##### 2) 配置node1网络，本实例采用flannel方式来配置，如需其他方式，请参考Kubernetes官网。

```
[root@node1 ~]# grep -v '^#' /etc/sysconfig/flanneld 
FLANNEL_ETCD_ENDPOINTS="http://10.10.10.14:2379"
FLANNEL_ETCD_PREFIX="/atomic.io/network"
FLANNEL_OPTIONS=""        
```

查看验证网络信息

```
[root@master ~]# etcdctl get /atomic.io/network/config  
{ "Network": "172.16.0.0/16" }
[root@master ~]# etcdctl ls /atomic.io/network/subnets
/atomic.io/network/subnets/172.16.69.0-24
/atomic.io/network/subnets/172.16.6.0-24
[root@master ~]# etcdctl get /atomic.io/network/subnets/172.16.6.0-24
{"PublicIP":"10.10.10.15"}
[root@master ~]# etcdctl get  /atomic.io/network/subnets/172.16.69.0-24
{"PublicIP":"10.10.10.16"}
```

#####      3) 配置node1 kube-proxy

```
[root@node1 ~]# grep -v '^#' /etc/kubernetes/config 
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://10.10.10.14:8080"
[root@node1 ~]# grep -v '^#' /etc/kubernetes/proxy                  
KUBE_PROXY_ARGS="--bind=address=0.0.0.0"
[root@node1 ~]# 
```

##### 4) 配置node1 kubelet

```
[root@node1 ~]# grep -v '^#' /etc/kubernetes/kubelet 
KUBELET_ADDRESS="--address=127.0.0.1"
KUBELET_HOSTNAME="--hostname-override=10.10.10.15"
KUBELET_API_SERVER="--api-servers=http://10.10.10.14:8080"
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
KUBELET_ARGS=""
```

##### 5) 启动node1服务

```
for i in flanneld kube-proxy kubelet docker;do systemctl restart $i;systemctl enable $i;systemctl status $i ;done
```

#### 配置node1节点服务器

node2与node1配置基本一致，除下面一处例外

```
[root@node2 ~]# vi /etc/kubernetes/kubelet
KUBELET_HOSTNAME="--hostname-override=10.10.10.16"
```

#### 查看节点

```
[root@master ~]# kubectl get nodes   
NAME          STATUS    AGE
10.10.10.15   Ready     18h
10.10.10.16   Ready     13h
```

k8s支持2种方式,一种是直接通过**命令参数**的方式,另一种是通过**配置文件**的方式,配置文件的话支持json和yaml

#### 命令参数

##### 建立pod

```
kubectl run nginx --image=nginx --port=80  --replicas=2
```

##### 遇到问题

创建成功但是kubectl get pods 没有结果
提示信息：no API token found for service account default
解决办法：编辑/etc/kubernetes/apiserver 去除 KUBE_ADMISSION_CONTROL中的SecurityContextDeny,ServiceAccount，并重启kube-apiserver.service服务

pod-infrastructure:latest镜像下载失败
报错信息：image pull failed for registry.access.redhat.com/rhel7/pod-infrastructure:latest, this may be because there are no credentials on this request. 
解决方案：yum install *rhsm* -y

登陆容器报错
[root@node14 ~]# kubectl exec -it nginx-bl7lc /bin/bash
Error from server: error dialing backend: dial tcp 10.10.10.16:10250: getsockopt: connection refused
解决方法：
10250是kubelet的端口。 
在Node上检查/etc/kubernetes/kubelet
KUBELET_ADDRESS需要修改为node ip

##### 删除deployment 与service

```
[root@node14 log]# kubectl delete deployment nginx
deployment "nginx" deleted
[root@node14 log]# kubectl delete service nginx
service "nginx" deleted
```

#### 配置文件方式

##### 定义pod 文件

[root@node14 ~]# vim nginx-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
     app: nginx
spec:
     containers:
     - name: nginx
       image: nginx
       imagePullPolicy: IfNotPresent
       ports:
       - containerPort: 80
     restartPolicy: Always
```

##### 发布到kubernetes集群中

```
[root@node14 ~]# kubectl create -f nginx-pod.yaml 
pod "nginx" created
```

##### 查看pod

```
[root@node14 ~]# kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
nginx     1/1       Running   0          16s
```

##### 定义与之关联的service 文件

[root@node14 ~]# vim nginx-svc.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  sessionAffinity: ClientIP
  selector:
    app: nginx
  ports:
    - port: 80
      nodePort: 30080
```

##### 创建service

```
[root@node14 ~]# kubectl create -f nginx-svc.yaml 
service "nginx-service" created
```

##### 查看刚刚创建的service

[root@node14 ~]# kubectl get service

```
NAME            CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes      10.254.0.1       <none>        443/TCP        23h
nginx-service   10.254.154.111   <nodes>       80:30080/TCP   20s
```

##### 验证结果如下

```
[root@node16 log]# elinks --dump http://10.10.10.16:30080
                               Welcome to nginx!
```
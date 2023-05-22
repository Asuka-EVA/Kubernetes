# 前置知识点



## 1、生产环境部署K8s集群的两种方式



•   **kubeadm**
Kubeadm是一个K8s部署工具，提供kubeadm init和kubeadm join，用于快速部署Kubernetes集群。



•   **二进制包**
从github下载发行版的二进制包，手动部署每个组件，组成Kubernetes集群。
小结：Kubeadm降低部署门槛，但屏蔽了很多细节，遇到问题很难排查。如果想更容易可控，推荐使用二进制包部署Kubernetes集群，虽然手动部署麻烦点，期间可以学习很多工作原理，也利于后期维护。





# 一、服务器整体规划



## 1、双Master+双Node

| 角色         | IP                 | 组件                                                         |
| ------------ | ------------------ | ------------------------------------------------------------ |
| k8s-master1  | 10.8.165.101       | kube-apiserver，kube-controller-manager，kube-scheduler，kubelet，kube-proxy，docker，etcd， |
| k8s-master2  | 10.8.165.113       | kube-apiserver，kube-controller-manager，kube-scheduler，kubelet，kube-proxy，docker，etcd， |
| k8s-node1    | 10.8.165.102       | kubelet，kube-proxy，docker，etcd                            |
| k8s-node2    | 10.8.165.103       | kubelet，kube-proxy，docker，etcd                            |
| 负载均衡器IP | 10.8.165.250 (VIP) |                                                              |



*考虑到有些朋友电脑配置较低，一次性开四台机器会跑不动，所以搭建这套K8s高可用集群分两部分实施，先部署一套单Master架构（3台），再扩容为多Master架构（4台或6台），顺便再熟悉下Master扩容流程
**单Master架构图**
![img](assets/二进制k8s搭建—V1.20(高可用)/1654759909751-df5cfeb8-065c-454c-8e68-2ef42db5ba77.png)
单Master服务器规划

| 角色       | IP           | 组件                                                         |
| ---------- | ------------ | ------------------------------------------------------------ |
| k8s-master | 10.8.165.101 | kube-apiserver，kube-controller-manager，kube-scheduler，etcd |
| k8s-node1  | 10.8.165.102 | kubelet，kube-proxy，docker，etcd                            |
| k8s-node2  | 10.8.165.103 | kubelet，kube-proxy，docker，etcd                            |



# 二、操作系统初始化配置



```shell
#关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

#关闭SElinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0   #临时

#关闭swap
swapoff -a  # 临时 
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 根据规划设置主机名 
hostnamectl set-hostname k8s-master1
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node2

#master配置主机映射
vim /etc/hosts
10.8.165.101 k8s-master1
10.8.165.102 k8s-node1
10.8.165.103 k8s-node2

# 将桥接的IPv4流量传递到iptables的链 
vim /etc/sysctl.d/k8s.conf 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

sysctl --system  # 生效
```



**所有软件包**



```shell
链接：https://pan.baidu.com/s/1RXRY39A8EcOpnOLy-w8gIg?pwd=726r 
提取码：726r
```



# 三、Etcd集群



Etcd 是一个分布式键值存储系统，Kubernetes使用Etcd进行数据存储，所以先准备一个Etcd数据库，为解决Etcd单点故障，应采用集群方式部署，这里使用3台组建集群，可容忍1台机器故障，当然，你也可以使用5台组建集群，可容忍2台机器故障。



注：为了节省机器，这里与K8s节点机器复用。也可以独立于k8s集群之外部署，只要apiserver能连接到就行。



## 1、准备cfssl证书生成工具



cfssl是一个开源的证书管理工具，使用json文件生成证书，相比openssl更方便使用。



```shell
#选择master进行，也可选择其他

wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo

*注：这里我有直接上传了
链接：https://pan.baidu.com/s/1yoKzZ60u8ORL0feNGrJ_zg?pwd=qb4r 
提取码：qb4r

上传到/usr/bin/下
[root@k8s-master1 ~]# chmod +x /usr/bin/cfssl*
```



## 2、生成Etcd证书



```shell
-------------------------master1节点------------------------
1)自签证书颁发机构（CA）

#创建目录
[root@k8s-master1 ~]# mkdir -p ca/etcd
[root@k8s-master1 ~]# cd ca/etcd

#自签CA
[root@k8s-master1 etcd]# vim ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}

[root@k8s-master1 etcd]# vim ca-csr.json
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}

#生成证书
[root@k8s-master1 etcd]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -


2)使用自签CA签发Etcd HTTPS证书

#创建证书申请文件
[root@k8s-master1 etcd]# vim server-csr.json
{
    "CN": "etcd",
    "hosts": [
    "10.8.165.101",
    "10.8.165.102",
    "10.8.165.103"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}

*注：上述文件hosts字段中IP为所有etcd节点的集群内部通信IP，一个都不能少！为了方便后期扩容可以多写几个预留的IP

#生成证书
[root@k8s-master1 etcd]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
```



## 3、从Github下载二进制文件



```shell
-------------------------master1节点------------------------

[root@k8s-master1 etcd]# cd ~
[root@k8s-master1 ~]# wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz

可能由于网络问题无法连接，可以下载下面的文件
链接：https://pan.baidu.com/s/1MEHHHaMCbuTczudztrBK5w?pwd=1eoa 
提取码：1eoa
```



## 4、部署Etcd集群



```shell
-------------------------master1节点------------------------

#创建工作目录并解压二进制包
[root@k8s-master1 ~]# mkdir -p /opt/etcd/{bin,cfg,ssl}
[root@k8s-master1 ~]# tar zxvf etcd-v3.4.9-linux-amd64.tar.gz
[root@k8s-master1 ~]# mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/

#创建etcd配置文件
[root@k8s-master1 ~]# vim /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.8.165.101:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.8.165.101:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.8.165.101:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.8.165.101:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.8.165.101:2380,etcd-2=https://10.8.165.102:2380,etcd-3=https://10.8.165.103:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"


*参数解释
•	ETCD_NAME：节点名称，集群中唯一
•	ETCD_DATA_DIR：数据目录
•	ETCD_LISTEN_PEER_URLS：集群通信监听地址
•	ETCD_LISTEN_CLIENT_URLS：客户端访问监听地址
•	ETCD_INITIAL_ADVERTISE_PEERURLS：集群通告地址
•	ETCD_ADVERTISE_CLIENT_URLS：客户端通告地址
•	ETCD_INITIAL_CLUSTER：集群节点地址
•	ETCD_INITIALCLUSTER_TOKEN：集群Token
•	ETCD_INITIALCLUSTER_STATE：加入集群的当前状态，new是新集群，existing表示加入已有集群


#systemd管理etcd
[root@k8s-master1 ~]# vim /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

#拷贝刚才生成的证书
[root@k8s-master1 ~]# cp ~/ca/etcd/*.pem /opt/etcd/ssl


#将上面Master节点所有生成的文件拷贝到节点Node2和节点Node3上
[root@k8s-master1 ~]# scp -r /opt/etcd/ k8s-node1:/opt/
[root@k8s-master1 ~]# scp /usr/lib/systemd/system/etcd.service k8s-node1:/usr/lib/systemd/system/
[root@k8s-master1 ~]# scp -r /opt/etcd/ k8s-node2:/opt/
[root@k8s-master1 ~]# scp /usr/lib/systemd/system/etcd.service k8s-node2:/usr/lib/systemd/system/


#在节点Node2和节点Node3分别修改etcd.conf配置文件
[root@k8s-node1 ~]# vim /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-2"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.8.165.102:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.8.165.102:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.8.165.102:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.8.165.102:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.8.165.101:2380,etcd-2=https://10.8.165.102:2380,etcd-3=https://10.8.165.103:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

[root@k8s-node2 ~]# vim /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd-3"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://10.8.165.103:2380"
ETCD_LISTEN_CLIENT_URLS="https://10.8.165.103:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.8.165.103:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://10.8.165.103:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.8.165.101:2380,etcd-2=https://10.8.165.102:2380,etcd-3=https://10.8.165.103:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"


---------------master1、node1、node2节点------------------

#启动并设置开机启动
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd

*注：master如果先启动了，会出现start悬停等待的现象，这时候可以先把node1、node2的etcd启动，随后master的etcd会正常启动。

#查看集群状态
[root@k8s-master1 ~]# ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://10.8.165.101:2379,https://10.8.165.102:2379,https://10.8.165.103:2379" endpoint health --write-out=table
+---------------------------+--------+-------------+-------+
|         ENDPOINT          | HEALTH |    TOOK     | ERROR |
+---------------------------+--------+-------------+-------+
| https://10.8.165.101:2379 |   true |  8.280002ms |       |
| https://10.8.165.102:2379 |   true |  8.113125ms |       |
| https://10.8.165.103:2379 |   true | 17.479134ms |       |
+---------------------------+--------+-------------+-------+

如果输出上面信息，就说明集群部署成功。
如果有问题第一步先看日志：/var/log/message 或 journalctl -u etcd
```



# 四、安装Docker



## 1、下载并解压二进制包



```shell
以下在所有节点操作。这里采用二进制安装，用yum安装也一样
------------------------master1、node1、node2节点--------------------------

[root@k8s-node1 ~]# wget https://download.docker.com/linux/static/stable/x86_64/docker-19.03.9.tgz
[root@k8s-node1 ~]# tar zxvf docker-19.03.9.tgz
[root@k8s-node1 ~]# mv docker/* /usr/bin
```



## 2、创建配置文件



```shell
------------------------master1、node1、node2节点--------------------------

[root@k8s-node1 ~]# mkdir /etc/docker
[root@k8s-node1 ~]# vim /etc/docker/daemon.json
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
```



## 3、systemd管理docker



```shell
------------------------master1、node1、node2节点--------------------------

[root@k8s-node1 ~]# vim /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target

#启动并设置开机启动
systemctl daemon-reload
systemctl enable docker
systemctl start docker
```



# 五、部署Master



## 1、生成kube-apiserver证书



```shell
#创建目录
[root@k8s-master1 ~]# mkdir ca/k8s
[root@k8s-master1 ~]# cd ca/k8s

#自签证书颁发机构（CA）
[root@k8s-master k8s]# vim ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
[root@k8s-master k8s]# vim ca-csr.json
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}

#生成证书
[root@k8s-master1 k8s]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

会生成ca.pem和ca-key.pem文件
```



## 2、使用自签CA签发kube-apiserver HTTPS证书



```shell
#创建证书申请文件
[root@k8s-master1 k8s]# vim server-csr.json
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "10.8.165.101",    #master1
      "10.8.165.102",    #node1
      "10.8.165.103",    #node2
      "10.8.165.113",    #master2（预留）
      "10.8.165.250",    #vip（预留）
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}

#生成证书
[root@k8s-master1 k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server

会生成server.pem和server-key.pem文件
```



## 3、下载二进制文件



```shell
链接：https://pan.baidu.com/s/1oBCKAkxvzqlYPd9JxZ_QHQ?pwd=knyi 
提取码：knyi

#创建目录
[root@k8s-master1 k8s]# cd ~
[root@k8s-master1 ~]# mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}

#解压二进制包
[root@k8s-master1 ~]# tar -zxf kubernetes-v1.20.4-server-linux-amd64.tar.gz

#拷贝
[root@k8s-master1 ~]# cd kubernetes/server/bin
[root@k8s-master1 bin]# cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
[root@k8s-master1 bin]# cp kubectl /usr/bin/
```



## 4、部署kube-apiserver



```shell
#创建配置文件
[root@k8s-master1 bin]# vim /opt/kubernetes/cfg/kube-apiserver.conf
KUBE_APISERVER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--etcd-servers=https://10.8.165.101:2379,https://10.8.165.102:2379,https://10.8.165.103:2379 \
--bind-address=10.8.165.101 \
--secure-port=6443 \
--advertise-address=10.8.165.101 \
--allow-privileged=true \
--service-cluster-ip-range=10.0.0.0/24 \
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \
--authorization-mode=RBAC,Node \
--enable-bootstrap-token-auth=true \
--token-auth-file=/opt/kubernetes/cfg/token.csv \
--service-node-port-range=30000-32767 \
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \
--client-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \
--service-account-issuer=api \
--service-account-signing-key-file=/opt/kubernetes/ssl/server-key.pem \
--etcd-cafile=/opt/etcd/ssl/ca.pem \
--etcd-certfile=/opt/etcd/ssl/server.pem \
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \
--requestheader-client-ca-file=/opt/kubernetes/ssl/ca.pem \
--proxy-client-cert-file=/opt/kubernetes/ssl/server.pem \
--proxy-client-key-file=/opt/kubernetes/ssl/server-key.pem \
--requestheader-allowed-names=kubernetes \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--enable-aggregator-routing=true \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"


参考说明
•	--logtostderr：启用日志
•	---v：日志等级
•	--log-dir：日志目录
•	--etcd-servers：etcd集群地址
•	--bind-address：监听地址
•	--secure-port：https安全端口
•	--advertise-address：集群通告地址
•	--allow-privileged：启用授权
•	--service-cluster-ip-range：Service虚拟IP地址段
•	--enable-admission-plugins：准入控制模块
•	--authorization-mode：认证授权，启用RBAC授权和节点自管理
•	--enable-bootstrap-token-auth：启用TLS bootstrap机制
•	--token-auth-file：bootstrap token文件
•	--service-node-port-range：Service nodeport类型默认分配端口范围
•	--kubelet-client-xxx：apiserver访问kubelet客户端证书
•	--tls-xxx-file：apiserver https证书
•	1.20版本必须加的参数：--service-account-issuer，--service-account-signing-key-file
•	--etcd-xxxfile：连接Etcd集群证书
•	--audit-log-xxx：审计日志
•	启动聚合层相关配置：--requestheader-client-ca-file，--proxy-client-cert-file，--proxy-client-key-file，--requestheader-allowed-names，--requestheader-extra-headers-prefix，--requestheader-group-headers，--requestheader-username-headers，--enable-aggregator-routing

#拷贝刚才生成的证书
[root@k8s-master1 bin]# cp ~/ca/k8s/ca*pem ~/ca/k8s/server*pem /opt/kubernetes/ssl/
```



**启用 TLS Bootstrapping 机制**
TLS Bootstraping：Master apiserver启用TLS认证后，Node节点kubelet和kube-proxy要与kube-apiserver进行通信，必须使用CA签发的有效证书才可以，当Node节点很多时，这种客户端证书颁发需要大量工作，同样也会增加集群扩展复杂度。为了简化流程，Kubernetes引入了TLS bootstraping机制来自动颁发客户端证书，kubelet会以一个低权限用户自动向apiserver申请证书，kubelet的证书由apiserver动态签署。所以强烈建议在Node上使用这种方式，目前主要用于kubelet，kube-proxy还是由我们统一颁发一个证书。
**TLS bootstraping 工作流程：**
![img](assets/二进制k8s搭建—V1.20(高可用)/1654779265199-e27e69ec-5825-4a7b-acad-891df51d1909.png)



```shell
#配置token文件
[root@k8s-master1 bin]# vim /opt/kubernetes/cfg/token.csv
c47ffb939f5ca36231d9e3121a252940,kubelet-bootstrap,10001,"system:node-bootstrapper"

*注：上述token可自行生成替换，但一定要与后续配置对应
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```



**systemd管理apiserver**



```shell
[root@k8s-master1 bin]# vim /usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver $KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target


#启动并设置开机启动
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
```



## 5、部署kube-controller-manager



```shell
#创建配置文件
[root@k8s-master1 bin]# vim /opt/kubernetes/cfg/kube-controller-manager.conf
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect=true \
--kubeconfig=/opt/kubernetes/cfg/kube-controller-manager.kubeconfig \
--bind-address=127.0.0.1 \
--allocate-node-cidrs=true \
--cluster-cidr=10.244.0.0/16 \
--service-cluster-ip-range=10.0.0.0/24 \
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \
--root-ca-file=/opt/kubernetes/ssl/ca.pem \
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \
--experimental-cluster-signing-duration=87600h0m0s"   #证书过期时间10年

参数说明
•	--kubeconfig：连接apiserver配置文件
•	--leader-elect：当该组件启动多个时，自动选举（HA）
•	--cluster-signing-cert-file/--cluster-signing-key-file：自动为kubelet颁发证书的CA，与apiserver保持一致
```



生成kubeconfig文件



```shell
#生成kube-controller-manager证书
[root@k8s-master1 bin]# cd ~/ca/k8s/

[root@k8s-master1 k8s]# vim kube-controller-manager-csr.json
{
  "CN": "system:kube-controller-manager",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}

[root@k8s-master1 k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

#生成kubeconfig文件
[root@k8s-master1 k8s]# KUBE_CONFIG="/opt/kubernetes/cfg/kube-controller-manager.kubeconfig"
[root@k8s-master1 k8s]# KUBE_APISERVER="https://10.8.165.101:6443"

·终端执行（4条）
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials kube-controller-manager \
  --client-certificate=./kube-controller-manager.pem \
  --client-key=./kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-controller-manager \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```



systemd管理controller-manager



```shell
[root@k8s-master1 k8s]# vim /usr/lib/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager $KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target

#启动并设置开机启动
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
```



## 6、部署kube-scheduler



```shell
#创建配置文件
[root@k8s-master1 k8s]# vim /opt/kubernetes/cfg/kube-scheduler.conf
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect \
--kubeconfig=/opt/kubernetes/cfg/kube-scheduler.kubeconfig \
--bind-address=127.0.0.1"

参数说明
•	--kubeconfig：连接apiserver配置文件
•	--leader-elect：当该组件启动多个时，自动选举（HA）
```



生成kubeconfig文件



```shell
#生成kube-scheduler证书（在/root/ca/k8s目录下）
[root@k8s-master1 k8s]# pwd
/root/ca/k8s

[root@k8s-master1 k8s]# vim kube-scheduler-csr.json
{
  "CN": "system:kube-scheduler",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}


#生成证书
[root@k8s-master1 k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```



生成kubeconfig文件



```shell
[root@k8s-master1 k8s]# KUBE_CONFIG="/opt/kubernetes/cfg/kube-scheduler.kubeconfig"
[root@k8s-master1 k8s]# KUBE_APISERVER="https://10.8.165.101:6443"


·终端执行（4条）
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials kube-scheduler \
  --client-certificate=./kube-scheduler.pem \
  --client-key=./kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-scheduler \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```



systemd管理scheduler



```shell
[root@k8s-master1 k8s]# vim /usr/lib/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler $KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target

#启动并设置开机启动
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
```



查看集群状态



```shell
#生成kubectl连接集群的证书
[root@k8s-master1 k8s]# vim admin-csr.json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}

[root@k8s-master1 k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

#生成kubeconfig文件
[root@k8s-master1 k8s]# mkdir /root/.kube

[root@k8s-master1 k8s]# KUBE_CONFIG="/root/.kube/config"
[root@k8s-master1 k8s]# KUBE_APISERVER="https://10.8.165.101:6443"

·终端执行（4条）
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials cluster-admin \
  --client-certificate=./admin.pem \
  --client-key=./admin-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=cluster-admin \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}


#通过kubectl工具查看当前集群组件状态
root@k8s-master1 k8s]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"} 





#若出现下列情况，可按下面操作
[root@k8s-master1 k8s]# kubectl get cs
NAME                 AGE
etcd-0               <unknown>
scheduler            <unknown>
controller-manager   <unknown>
etcd-2               <unknown>
etcd-1               <unknown>

#从1.16开始就显示为unknow 具体原因:https://segmentfault.com/a/1190000020912684


#临时解决办法（通过模板）
[root@k8s-master1 k8s]# kubectl get cs -o=go-template='{{printf "|NAME|STATUS|MESSAGE|\n"}}{{range .items}}{{$name := .metadata.name}}{{range .conditions}}{{printf "|%s|%s|%s|\n" $name .status .message}}{{end}}{{end}}'
|NAME|STATUS|MESSAGE|
|scheduler|True|ok|
|controller-manager|True|ok|
|etcd-1|True|{"health":"true"}|
|etcd-0|True|{"health":"true"}|
|etcd-2|True|{"health":"true"}|

#查看k8s的名称空间
[root@k8s-master1 k8s]# kubectl get ns
NAME              STATUS   AGE
default           Active   3h21m
kube-node-lease   Active   3h21m
kube-public       Active   3h21m
kube-system       Active   3h21m
```



![img](assets/二进制k8s搭建—V1.20(高可用)/1654672348892-e7da0b19-34a3-413a-a9e3-ea594a3e498c.png)



![img](assets/二进制k8s搭建—V1.20(高可用)/1654672424050-2c385d57-3dbb-41d6-a10c-f0869320f4d0.png)



# 六、部署Worker Node



## 1、创建工作目录并拷贝文件



```shell
--------------------node1、node2节点-------------------
[root@k8s-node1 ~]# mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}

#master将kubelet和kube-proxy拷贝给node1、node2节点
[root@k8s-master k8s]# cd ~/kubernetes/server/bin/
[root@k8s-master bin]# scp kubelet kube-proxy k8s-node1:/opt/kubernetes/bin/

#本地拷贝
[root@k8s-master1 bin]# cp kubelet kube-proxy /opt/kubernetes/bin

上传到/opt/kubernetes/bin下
```



## 2、部署kubelet



```shell
---------------------master1节点操作---------------------
[root@k8s-master1 bin]# vim /opt/kubernetes/cfg/kubelet.conf
KUBELET_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--hostname-override=k8s-master1 \    #每个节点的ip/名称
--network-plugin=cni \
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
--config=/opt/kubernetes/cfg/kubelet-config.yml \
--cert-dir=/opt/kubernetes/ssl \
--pod-infra-container-image=lizhenliang/pause-amd64:3.0"

参数说明
•	--hostname-override：显示名称，集群中唯一
•	--network-plugin：启用CNI
•	--kubeconfig：空路径，会自动生成，后面用于连接apiserver
•	--bootstrap-kubeconfig：首次启动向apiserver申请证书
•	--config：配置参数文件
•	--cert-dir：kubelet证书生成目录
•	--pod-infra-container-image：管理Pod网络容器的镜像

#拉取镜像
[root@k8s-master1 bin]# docker pull lizhenliang/pause-amd64:3.0

#配置参数文件
[root@k8s-master1 bin]# vim /opt/kubernetes/cfg/kubelet-config.yml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110


#授权kubelet-bootstrap用户允许请求证书
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap


#生成kubelet初次加入集群引导kubeconfig文件
*在生成kubernetes证书的目录下执行以下命令生成kubeconfig文件
[root@k8s-master1 bin]# cd /root/ca/k8s/
[root@k8s-master1 k8s]# KUBE_CONFIG="/opt/kubernetes/cfg/bootstrap.kubeconfig"
[root@k8s-master1 k8s]# KUBE_APISERVER="https://10.8.165.101:6443"   # apiserver的 IP:PORT
[root@k8s-master1 k8s]# TOKEN="c47ffb939f5ca36231d9e3121a252940"    # 与master的token.csv里保持一致

·终端执行（四条）
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials "kubelet-bootstrap" \
  --token=${TOKEN} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user="kubelet-bootstrap" \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}

#systemd管理kubelet
[root@k8s-master1 k8s]# vim /usr/lib/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
After=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet $KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

#启动并设置开机启动
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
```



批准kubelet证书申请并加入集群



```shell
-----------------------master1节点-----------------------
# 查看kubelet证书请求
[root@k8s-master1 k8s]# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-0UTuuhUTPbL02uDpLinrwBc_YDnmXj3t-JjUqMM247I   78s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending


#批准申请
kubectl certificate approve <申请的NAME>

[root@k8s-master1 k8s]# kubectl certificate approve node-csr-0UTuuhUTPbL02uDpLinrwBc_YDnmXj3t-JjUqMM247I
certificatesigningrequest.certificates.k8s.io/node-csr-0UTuuhUTPbL02uDpLinrwBc_YDnmXj3t-JjUqMM247I approved


[root@k8s-master1 k8s]# kubectl get node
NAME          STATUS     ROLES    AGE   VERSION
k8s-master1   NotReady   <none>   20s   v1.20.4


*注：由于网络插件还没有部署，节点会没有准备就绪 NotReady
```



## 3、部署kube-proxy



生成kube-proxy.kubeconfig文件



```shell
-------------------master节点-------------------------
#在/root/ca/k8s下创建证书请求文件
[root@k8s-master1 k8s]# vim kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

# 生成证书
[root@k8s-master1 k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

#生成kubeconfig文件
[root@k8s-master1 k8s]# KUBE_CONFIG="/opt/kubernetes/cfg/kube-proxy.kubeconfig"
[root@k8s-master1 k8s]# KUBE_APISERVER="https://10.8.165.101:6443"

·终端执行（4条）
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=${KUBE_CONFIG}

kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```



配置文件



```shell
---------------------master1节点-------------------------

#创建配置文件
[root@k8s-master1 k8s]# vim /opt/kubernetes/cfg/kube-proxy.conf
KUBE_PROXY_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"

#配置参数文件
[root@k8s-master1 k8s]# vim /opt/kubernetes/cfg/kube-proxy-config.yml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-master1
clusterCIDR: 10.244.0.0/16
```



systemd管理kube-proxy



```shell
---------------------master1节点-------------------------

[root@k8s-master1 k8s]# vim /usr/lib/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy $KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

#启动并设置开机启动
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
```



## 4、部署网络组件（Calico、flanneld二选一）



**Calico（选择）**



```shell
Calico是一个纯三层的数据中心网络方案，是目前Kubernetes主流的网络方案

#上传yaml文件
链接：https://pan.baidu.com/s/1jPzSdsnFKSFxkVQzc2lQ9g?pwd=x311 
提取码：x311



#部署Calico
[root@k8s-master1 k8s]# cd /opt/kubernetes/cfg/

*上传至/opt/kubernetes/cfg/

[root@k8s-master1 cfg]# kubectl apply -f calico.yaml
[root@k8s-master1 cfg]# kubectl get pods -n kube-system


等Calico Pod都Running，节点也会准备就绪
！！！！！！！！！！！！！！！！！！！！！！！！！！！！！
可能有点久，要初始化创建对应镜像，好几分钟，等吧骚年
waiting......................
！！！！！！！！！！！！！！！！！！！！！！！！！！！！！

[root@k8s-master1 cfg]# kubectl get pods -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-97769f7c7-9d49d   1/1     Running   0          9m16s
calico-node-8djzj                         1/1     Running   0          9m16s

*看网速、看脸    快10分钟了

[root@k8s-master1 cfg]# kubectl get node
NAME          STATUS   ROLES    AGE   VERSION
k8s-master1   Ready    <none>   23m   v1.20.4
```



**flanneld（没选）**



```shell
--------------------------master1节点-----------------------
/opt/etcd/bin/etcdctl \
--cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem \
--endpoints="https://10.8.165.101:2379,https://10.8.165.102:2379,https://10.8.165.103:2379" \
put /coreos.com/network/config  '{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'

-------------------master1、node1、node2节点------------------------
#下载二进制包
链接：https://pan.baidu.com/s/1vSxuXNQZU8yXkcCvDED1Pw?pwd=0ind 
提取码：0ind

[root@k8s-node1 ~]# tar zvxf flannel-v0.13.0-linux-amd64.tar.gz
[root@k8s-node1 ~]# mv flanneld mk-docker-opts.sh /opt/kubernetes/bin


#配置Flannel
[root@k8s-node1 ~]# vim /opt/kubernetes/cfg/flanneld
FLANNEL_OPTIONS="--etcd-endpoints=https://10.8.165.101:2379,https://10.8.165.102:2379,https://10.8.165.103:2379 -etcd-cafile=/opt/etcd/ssl/ca.pem -etcd-certfile=/opt/etcd/ssl/server.pem -etcd-keyfile=/opt/etcd/ssl/server-key.pem"


#systemd管理Flannel
[root@k8s-node1 ~]# vim /usr/lib/systemd/system/flanneld.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network-online.target network.target
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/opt/kubernetes/cfg/flanneld
ExecStart=/opt/kubernetes/bin/flanneld --ip-masq $FLANNEL_OPTIONS
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure

[Install]
WantedBy=multi-user.target

#启动flannel和docker
systemctl daemon-reload
systemctl start flanneld
systemctl enable flanneld
systemctl daemon-reload
systemctl restart docker
```



***报错（Couldn‘t fetch network config）*
![img](assets/二进制k8s搭建—V1.20(高可用)/1654699072941-dcbd5368-d2e7-4b92-b3fe-57956e696a31.png)



```shell
原因：flanneld目前不能与etcdV3直接交互
参考：https://blog.51cto.com/u_8355320/2564588

#开启etcd 支持V2api功能，在etcd启动参数中加入 --enable-v2参数，并重启etcd2

master、node1、node2都改，并重启
[root@k8s-master etcd]# vim /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap \
--enable-v2     #此处添加
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

[root@master1 ~]# systemctl daemon-reload
[root@master1 ~]# systemctl restart etcd

·master查看etcd集群健康状况
[root@k8s-master etcd]# ETCDCTL_API=2 /opt/etcd/bin/etcdctl --ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem --endpoints="https://10.8.165.101:2379,https://10.8.165.102:2379,https://10.8.165.103:2379" cluster-health

#删除原来写入的子网信息
[root@k8s-master etcd]# /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://10.8.165.101:2379,https://10.8.165.102:2379,https://10.8.165.103:2379" del /coreos.com/network/config

#重新使用V2写入子网信息
ETCDCTL_API=2 /opt/etcd/bin/etcdctl \
--ca-file=/opt/etcd/ssl/ca.pem --cert-file=/opt/etcd/ssl/server.pem --key-file=/opt/etcd/ssl/server-key.pem \
--endpoints="https://10.8.165.101:2379,https://10.8.165.102:2379,https://10.8.165.103:2379" \
set /coreos.com/network/config '{ "Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'

#重启flanneld服务
systemctl daemon-reload
systemctl start flanneld
systemctl enable flanneld

#修改docker文件
[root@k8s-node1 ~]# vim /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target

#重启docker
systemctl daemon-reload
systemctl restart docker
```



![img](assets/二进制k8s搭建—V1.20(高可用)/1654699241100-ef41ab0c-c5f7-4fbd-a267-a96695589871.png)



![img](assets/二进制k8s搭建—V1.20(高可用)/1654699266573-3cca90fc-3ba0-4cb6-a7f3-496d99cefbdf.png)
**随后node还是NotReady**
![img](assets/二进制k8s搭建—V1.20(高可用)/1654736416250-9de4fa24-1a8a-47b2-9a85-95a2606051ec.png)



```shell
通过journalctl -f -u kubelet命令查看，发现没有安装相应的cni模块

因为kubelet配置了network-plugin=cni，但是还没安装，所以状态会是NotReady,不想看这个报错或者不需要网络，就可以修改kubelet配置文件，去掉network-plugin=cni 就可以了。

-----------------------node1、node2节点--------------------
#修改kubelet.conf
[root@k8s-node1 bin]# vim /opt/kubernetes/cfg/kubelet.conf
KUBELET_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--hostname-override=k8s-node1 \
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \
--config=/opt/kubernetes/cfg/kubelet-config.yml \
--cert-dir=/opt/kubernetes/ssl \
--pod-infra-container-image=lizhenliang/pause-amd64:3.0"

#删除之前的kubelet的认证文件
[root@k8s-node1 bin]# rm -rf /opt/kubernetes/ssl/kubelet*

#重启kubelet
[root@k8s-node1 bin]# systemctl restart kubelet


--------------------------master节点----------------------
#重新认证
[root@k8s-master ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-NTYjTGk7zN-oSmSe4mbKuYQXEvvJpOlyIwy2tWwK9GA   29s       kubelet-bootstrap   Pending
node-csr-qVVLrp7PKGSFlA2Q0PGYZBTY_E1FnFoO3K4FGXOIz0Q   18s       kubelet-bootstrap   Pending

[root@k8s-master ~]# /opt/kubernetes/bin/kubectl certificate approve node-csr-NTYjTGk7zN-oSmSe4mbKuYQXEvvJpOlyIwy2tWwK9GA
certificatesigningrequest.certificates.k8s.io/node-csr-NTYjTGk7zN-oSmSe4mbKuYQXEvvJpOlyIwy2tWwK9GA approved
[root@k8s-master ~]# /opt/kubernetes/bin/kubectl certificate approve node-csr-qVVLrp7PKGSFlA2Q0PGYZBTY_E1FnFoO3K4FGXOIz0Q
certificatesigningrequest.certificates.k8s.io/node-csr-qVVLrp7PKGSFlA2Q0PGYZBTY_E1FnFoO3K4FGXOIz0Q approved



#结果
[root@k8s-master ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-NTYjTGk7zN-oSmSe4mbKuYQXEvvJpOlyIwy2tWwK9GA   68s       kubelet-bootstrap   Approved,Issued
node-csr-qVVLrp7PKGSFlA2Q0PGYZBTY_E1FnFoO3K4FGXOIz0Q   57s       kubelet-bootstrap   Approved,Issued

[root@k8s-master ~]# kubectl get node
NAME        STATUS    ROLES     AGE       VERSION
k8s-node1   Ready     <none>    20s       v1.20.4
k8s-node2   Ready     <none>    5s        v1.20.4
```



## 5、授权apiserver访问kubelet



```shell
#应用场景：例如kubectl logs

[root@k8s-master1 cfg]# vim apiserver-to-kubelet-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes


[root@k8s-master1 cfg]# kubectl apply -f apiserver-to-kubelet-rbac.yaml
```



## 6、新增加Worker Node



**拷贝已部署好的Node相关文件到新节点**



```shell
----------------------master1节点-------------------------

#在Master1节点将Worker Node涉及文件拷贝到node1
[root@k8s-master1 cfg]# scp -r /opt/kubernetes k8s-node1:/opt/
[root@k8s-master1 cfg]# scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service k8s-node1:/usr/lib/systemd/system
[root@k8s-master1 cfg]# scp /opt/kubernetes/ssl/ca.pem k8s-node1:/opt/kubernetes/ssl/

------------------------node1节点------------------------
#删除kubelet证书和kubeconfig文件
[root@k8s-node1 ~]# rm -f /opt/kubernetes/cfg/kubelet.kubeconfig
[root@k8s-node1 ~]# rm -f /opt/kubernetes/ssl/kubelet*

*注：这几个文件是证书申请审批后自动生成的，每个Node不同，必须删除


#修改对应文件的主机名
[root@k8s-node1 ~]# vim /opt/kubernetes/cfg/kubelet.conf
--hostname-override=k8s-node1
[root@k8s-node1 ~]# vim /opt/kubernetes/cfg/kube-proxy-config.yml
hostnameOverride: k8s-node1

#启动并设置开机启动
systemctl daemon-reload
systemctl enable kubelet kube-proxy
systemctl start kubelet kube-proxy

---------------------------master1节点--------------------
#在Master1上批准新Node kubelet证书申请

*查看证书请求
[root@k8s-master1 cfg]# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-0UTuuhUTPbL02uDpLinrwBc_YDnmXj3t-JjUqMM247I   41m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
node-csr-xQsFeNF5nlB-rZQr2cIxFB18ET3kAGYHSC9GMSKDoI8   41s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

*授权请求（Pending，一定要是待定状态的）
[root@k8s-master1 cfg]# kubectl certificate approve node-csr-xQsFeNF5nlB-rZQr2cIxFB18ET3kAGYHSC9GMSKDoI8
certificatesigningrequest.certificates.k8s.io/node-csr-xQsFeNF5nlB-rZQr2cIxFB18ET3kAGYHSC9GMSKDoI8 approved

#查看Node状态
[root@k8s-master1 cfg]# kubectl get node
NAME          STATUS     ROLES    AGE   VERSION
k8s-master1   Ready      <none>   41m   v1.20.4
k8s-node1     NotReady   <none>   61s   v1.20.4


*发现新加入的k8s-node1节点是NotReady状态，不要急，等待一下，跟master节点一样，Calico在做初始化

*这个时候可以去做node2，跟加入node1的步骤一样，当然最后也要等！！！
```



![img](assets/二进制k8s搭建—V1.20(高可用)/1654776703577-0093688d-2a95-4dd1-bd23-38b708c61db1.png)
**最终加入结果**



```shell
#查看名称空间kube-system中的pod情况
[root@k8s-master1 cfg]# kubectl get pods -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-97769f7c7-9d49d   1/1     Running   0          42m
calico-node-8djzj                         1/1     Running   0          42m
calico-node-h6ghf                         1/1     Running   0          15m
calico-node-nj9l7                         1/1     Running   0          8m1s


#查看Node状态
[root@k8s-master1 cfg]# kubectl get node
NAME          STATUS   ROLES    AGE     VERSION
k8s-master1   Ready    <none>   56m     v1.20.4
k8s-node1     Ready    <none>   16m     v1.20.4
k8s-node2     Ready    <none>   8m25s   v1.20.4
```



# 七、部署Dashboard和CoreDNS



## 1、部署Dashboard



```shell
----------------------master1节点-------------------------
*上面安装Calico时已经下载好了

[root@k8s-master1 cfg]# kubectl apply -f kubernetes-dashboard.yaml

# 查看部署
[root@k8s-master1 cfg]# kubectl get pods,svc -n kubernetes-dashboard
NAME                                             READY   STATUS              RESTARTS   AGE
pod/dashboard-metrics-scraper-7b59f7d4df-s7c6g   0/1     ContainerCreating   0          27s
pod/kubernetes-dashboard-74d688b6bc-5ln4n        0/1     ContainerCreating   0          27s

NAME                                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
service/dashboard-metrics-scraper   ClusterIP   10.0.0.19    <none>        8000/TCP        27s
service/kubernetes-dashboard        NodePort    10.0.0.9     <none>        443:30001/TCP   27s

*发现状态是ContainerCreating，即容器创建中，等待吧.......
*全部起来大概3分钟左右



#创建service account并绑定默认cluster-admin管理员集群角色
[root@k8s-master1 cfg]# kubectl create serviceaccount dashboard-admin -n kube-system
[root@k8s-master1 cfg]# kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
[root@k8s-master1 cfg]# kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
Name:         dashboard-admin-token-hclxd
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: 03acc120-0133-4ccd-8047-9830cff868b3

Type:  kubernetes.io/service-account-token

Data
====
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6InV5ejZ2MlowdnNJaXVTQTJRUW0wTU50Nk01SEdoYVF1N3diMG9oYTRoUkEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4taGNseGQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMDNhY2MxMjAtMDEzMy00Y2NkLTgwNDctOTgzMGNmZjg2OGIzIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.ON_4fihI9XykB46854v4Lge1AMrKpKvTrhc5Mc1SguroxalskH_hUAtDTBinODOzcz2TP3aJz6uQ5Rq3UWND8i9AcuJl9f9Kpcaml3XnR6sdJSkwNGPvqLxK-uY1pbo-NlOaMs4LjgSJ5_dzLRt4KoLXDF96MSTAenY8E_K_pwfADF67qPUB90rGbyh-jedj9u_F0X4mQf7URYqEDDU1VFMYoVvuD0XdKwdlzRP-_juEXRmdhJoenigr-Y_KwZomWAkIadwK_lKKsSADCXD6uzcTTHZCNnKJFUaClw-oDL214O5CF79Y48nl4ZNqYqUd09X9Rr3qz3PPCrKcY9qHyg
ca.crt:     1359 bytes
namespace:  11 bytes


#访问地址：https://NodeIP:30001

*注意一定要加https://    不然默认就是http，会错哦
*随后用上述生成的token访问
```



**错误示范：**
![img](assets/二进制k8s搭建—V1.20(高可用)/1654778118370-3f703dc4-8b29-4abe-8409-db4d46235038.png)



![img](assets/二进制k8s搭建—V1.20(高可用)/1654778138146-61b8b6cf-294b-48ac-b2c1-9a3a567ce1ab.png)
**正确示范：**
![img](assets/二进制k8s搭建—V1.20(高可用)/1654778218396-c06faf98-0c35-417b-87f3-86e8f03fbbd9.png)



![img](assets/二进制k8s搭建—V1.20(高可用)/1654778285271-33e9f28e-5235-4100-8a45-7452a4ca6bd2.png)



![img](assets/二进制k8s搭建—V1.20(高可用)/1654778338716-5bb9a1a1-2249-4d62-a358-4374a618ab68.png)



## 2、部署CoreDNS



```shell
----------------------master1节点-------------------------
*上面安装Calico时已经下载好了

#CoreDNS用于集群内部Service名称解析
[root@k8s-master1 cfg]# kubectl apply -f coredns.yaml
[root@k8s-master1 cfg]# kubectl get pods -n kube-system


[root@k8s-master1 cfg]# kubectl get pods -n kube-system
NAME                                      READY   STATUS              RESTARTS   AGE
calico-kube-controllers-97769f7c7-9d49d   1/1     Running             0          60m
calico-node-8djzj                         1/1     Running             0          60m
calico-node-h6ghf                         1/1     Running             0          33m
calico-node-nj9l7                         1/1     Running             0          25m
coredns-6d8f96d957-kzn2g                  0/1     ContainerCreating   0          23s

*接着等，嘎嘎.............（这个快几秒？）

#DNS解析测试
[root@k8s-master1 cfg]# kubectl run -it --rm dns-test --image=busybox:1.28.4 sh
If you don't see a command prompt, try pressing enter.
/ # nslookup kubernetes
Server:    10.0.0.2
Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local



解析没问题。
至此一个单Master集群就搭建完成了！这个环境就足以满足学习实验了，如果你的服务器配置较高，可继续扩容多Master集群！
```



# 八、扩容多Master（高可用架构）



Kubernetes作为容器集群系统，通过健康检查+重启策略实现了Pod故障自我修复能力，通过调度算法实现将Pod分布式部署，并保持预期副本数，根据Node失效状态自动在其他Node拉起Pod，实现了应用层的高可用性。
针对Kubernetes集群，高可用性还应包含以下两个层面的考虑：Etcd数据库的高可用性和Kubernetes Master组件的高可用性。 而Etcd我们已经采用3个节点组建集群实现高可用，本节将对Master节点高可用进行说明和实施。
Master节点扮演着总控中心的角色，通过不断与工作节点上的Kubelet和kube-proxy进行通信来维护整个集群的健康工作状态。如果Master节点故障，将无法使用kubectl工具或者API做任何集群管理。
Master节点主要有三个服务kube-apiserver、kube-controller-manager和kube-scheduler，其中kube-controller-manager和kube-scheduler组件自身通过选择机制已经实现了高可用，所以Master高可用主要针对kube-apiserver组件，而该组件是以HTTP API提供服务，因此对他高可用与Web服务器类似，增加负载均衡器对其负载均衡即可，并且可水平扩容。
**多Master架构图：**
![img](assets/二进制k8s搭建—V1.20(高可用)/1654778702517-6a2b2bc7-ad5f-406d-af3d-497302978ba0.png)



## 1、部署Master2 Node



现在需要再增加一台新服务器，作为Master2 Node，IP是192.168.31.74。
为了节省资源你也可以将之前部署好的Worker Node1复用为Master2 Node角色（即部署Master组件）



```shell
Master2 与已部署的Master1所有操作一致。所以我们只需将Master1所有K8s文件拷贝过来，再修改下服务器IP和主机名启动即可。



#安装Docker
-----------------------master1节点-----------------------
[root@k8s-master1 cfg]# scp /usr/bin/docker* 10.8.165.113:/usr/bin
[root@k8s-master1 cfg]# scp /usr/bin/runc 10.8.165.113:/usr/bin
[root@k8s-master1 cfg]# scp /usr/bin/containerd* 10.8.165.113:/usr/bin
[root@k8s-master1 cfg]# scp /usr/lib/systemd/system/docker.service 10.8.165.113:/usr/lib/systemd/system
[root@k8s-master1 cfg]# scp -r /etc/docker 10.8.165.113:/etc

# 在Master2启动Docker
-----------------------master2节点-----------------------
systemctl daemon-reload
systemctl enable docker
systemctl start docker


#在Master2创建etcd证书目录
-----------------------master2节点-----------------------
[root@k8s-master2 ~]# mkdir -p /opt/etcd/ssl


#拷贝Master1上所有K8s文件和etcd证书到Master2
-----------------------master1节点-----------------------
[root@k8s-master1 cfg]# scp -r /opt/kubernetes 10.8.165.113:/opt
[root@k8s-master1 cfg]# scp -r /opt/etcd/ssl 10.8.165.113:/opt/etcd
[root@k8s-master1 cfg]# scp /usr/lib/systemd/system/kube* 10.8.165.113:/usr/lib/systemd/system
[root@k8s-master1 cfg]# scp /usr/bin/kubectl  10.8.165.113:/usr/bin
[root@k8s-master1 cfg]# scp -r ~/.kube 10.8.165.113:~


#删除kubelet证书和kubeconfig文件
-----------------------master2节点-----------------------
[root@k8s-master2 ~]# rm -f /opt/kubernetes/cfg/kubelet.kubeconfig
[root@k8s-master2 ~]# rm -f /opt/kubernetes/ssl/kubelet*


#修改配置文件IP和主机名
-----------------------master2节点-----------------------

*修改apiserver、kubelet和kube-proxy配置文件为本地IP
[root@k8s-master2 ~]# vim /opt/kubernetes/cfg/kube-apiserver.conf
--bind-address=10.8.165.113
--advertise-address=10.8.165.113

[root@k8s-master2 ~]# vim /opt/kubernetes/cfg/kube-controller-manager.kubeconfig
server: https://10.8.165.113:6443

[root@k8s-master2 ~]# vim /opt/kubernetes/cfg/kube-scheduler.kubeconfig
server: https://10.8.165.113:6443

[root@k8s-master2 ~]# vim /opt/kubernetes/cfg/bootstrap.kubeconfig
server: https://10.8.165.113:6443

[root@k8s-master2 ~]# vim /opt/kubernetes/cfg/kube-proxy.kubeconfig
server: https://10.8.165.113:6443

[root@k8s-master2 ~]# vim /opt/kubernetes/cfg/kubelet.conf
--hostname-override=k8s-master2

[root@k8s-master2 ~]# vim /opt/kubernetes/cfg/kube-proxy-config.yml
hostnameOverride: k8s-master2

[root@k8s-master2 ~]# vi ~/.kube/config
server: https://10.8.165.113:6443


#启动设置开机启动
-----------------------master2节点-----------------------
systemctl daemon-reload
systemctl enable kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy
systemctl start kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy
```



**检验并加入集群**



```shell
---------------------可master1  也可master2----------------
因为两者都是master嘛


#查看集群状态
[root@k8s-master2 ~]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}


#批准kubelet证书申请

*查看证书请求
[root@k8s-master2 ~]# kubectl get csr
NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr--okVOhcnMlwp9j2L64uYd6HKiopCDU1FQf9Ywj_EUhw   2m55s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-BQJv1fS07fW1u4uPTID4M0ybNJV60Br71DjA67DmJxk   64m     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
node-csr-xQsFeNF5nlB-rZQr2cIxFB18ET3kAGYHSC9GMSKDoI8   73m     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued

*授权请求
[root@k8s-master2 ~]# kubectl certificate approve node-csr--okVOhcnMlwp9j2L64uYd6HKiopCDU1FQf9Ywj_EUhw
certificatesigningrequest.certificates.k8s.io/node-csr--okVOhcnMlwp9j2L64uYd6HKiopCDU1FQf9Ywj_EUhw approved


# 查看Node
[root@k8s-master2 ~]# kubectl get node
NAME          STATUS     ROLES    AGE    VERSION
k8s-master1   Ready      <none>   113m   v1.20.4
k8s-master2   NotReady   <none>   54s    v1.20.4
k8s-node1     Ready      <none>   73m    v1.20.4
k8s-node2     Ready      <none>   65m    v1.20.4


[root@k8s-master1 cfg]# kubectl get node
NAME          STATUS     ROLES    AGE    VERSION
k8s-master1   Ready      <none>   113m   v1.20.4
k8s-master2   NotReady   <none>   42s    v1.20.4
k8s-node1     Ready      <none>   73m    v1.20.4
k8s-node2     Ready      <none>   65m    v1.20.4


*NotReady是因为Calico正在为master2进行初始化相关网络镜像

Waiting......................
```



**最终结果**



```shell
[root@k8s-master2 ~]# kubectl get pod -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-97769f7c7-9d49d   1/1     Running   0          107m
calico-node-8djzj                         1/1     Running   0          107m
calico-node-bkdm6                         1/1     Running   0          8m2s
calico-node-h6ghf                         1/1     Running   0          80m
calico-node-nj9l7                         1/1     Running   0          72m
coredns-6d8f96d957-kzn2g                  1/1     Running   0          47m



[root@k8s-master2 ~]# kubectl get node
NAME          STATUS   ROLES    AGE    VERSION
k8s-master1   Ready    <none>   120m   v1.20.4
k8s-master2   Ready    <none>   8m6s   v1.20.4
k8s-node1     Ready    <none>   80m    v1.20.4
k8s-node2     Ready    <none>   72m    v1.20.4
```



## 2、部署Nginx+Keepalived高可用负载均衡器



**kube-apiserver高可用架构图：**
![img](assets/二进制k8s搭建—V1.20(高可用)/1654781050256-09873bf8-46e3-43d9-baae-42697eee26ab.png)
•   Nginx是一个主流Web服务和反向代理服务器，这里用四层实现对apiserver实现负载均衡。
•   Keepalived是一个主流高可用软件，基于VIP绑定实现服务器双机热备，在上述拓扑中，Keepalived主要根据Nginx运行状态判断是否需要故障转移（漂移VIP），例如当Nginx主节点挂掉，VIP会自动绑定在Nginx备节点，从而保证VIP一直可用，实现Nginx高可用。
注1：为了节省机器，这里与K8s Master节点机器复用。也可以独立于k8s集群之外部署，只要nginx与apiserver能通信就行。
注2：如果你是在公有云上，一般都不支持keepalived，那么你可以直接用它们的负载均衡器产品，直接负载均衡多台Master kube-apiserver，架构与上面一样。



**安装软件包**



```shell
----------------------master1、master2节点-----------------
#安装软件包（主/备）
[root@k8s-master1 cfg]# yum -y install epel-release nginx keepalived
```



**Nginx**



```shell
----------------------master1、master2节点-----------------
#Nginx配置文件（主/备一样）
[root@k8s-master1 cfg]# vim /etc/nginx/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

# 四层负载均衡，为两台Master apiserver组件提供负载均衡
stream {

    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';

    access_log  /var/log/nginx/k8s-access.log  main;

    upstream k8s-apiserver {
       server 10.8.165.101:6443;   # Master1 APISERVER IP:PORT
       server 10.8.165.113:6443;   # Master2 APISERVER IP:PORT
    }

    server {
       listen 16443; # 由于nginx与master节点复用，这个监听端口不能是6443，否则会冲突
       proxy_pass k8s-apiserver;
    }
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen       80 default_server;
        server_name  _;

        location / {
        }
    }
}
```



Keepalived



```shell
------------------------master1节点---------------------
#keepalived配置文件（Nginx Master）
[root@k8s-master1 cfg]# vim /etc/keepalived/keepalived.conf
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id NGINX_MASTER
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33  # 修改为实际网卡名
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的
    priority 100    # 优先级，备服务器设置 90
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 虚拟IP
    virtual_ipaddress {
        10.8.165.250/24
    }
    track_script {
        check_nginx
    }
}


说明：
•	vrrp_script：指定检查nginx工作状态脚本（根据nginx状态判断是否故障转移）
•	virtual_ipaddress：虚拟IP（VIP）


#准备上述配置文件中检查nginx运行状态的脚本
[root@k8s-master1 cfg]# vim /etc/keepalived/check_nginx.sh
#!/bin/bash
count=$(ss -antp |grep nginx |egrep -cv "grep|$$")

if [ "$count" -eq 0 ];then
    exit 1
else
    exit 0
fi


#赋予脚本权限
[root@k8s-master1 cfg]# chmod +x /etc/keepalived/check_nginx.sh


------------------------master2节点---------------------
#keepalived配置文件（Nginx Backup）
[root@k8s-master2 ~]# vim /etc/keepalived/keepalived.conf
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id NGINX_BACKUP
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.8.165.250/24
    }
    track_script {
        check_nginx
    }
}


#准备上述配置文件中检查nginx运行状态的脚本
[root@k8s-master2 ~]# vim /etc/keepalived/check_nginx.sh
#!/bin/bash
count=$(ss -antp |grep nginx |egrep -cv "grep|$$")

if [ "$count" -eq 0 ];then
    exit 1
else
    exit 0
fi

#赋予脚本权限
[root@k8s-master2 ~]# chmod +x /etc/keepalived/check_nginx.sh

*注：keepalived根据脚本返回状态码（0为工作正常，非0不正常）判断是否故障转移。
```



**开启nginx、keepalived**



```shell
-------------------master1、master2节点--------------------
#启动并设置开机启动
systemctl daemon-reload
systemctl enable nginx keepalived
systemctl start nginx keepalived

*报错
[root@k8s-master1 cfg]# journalctl -xe -u nginx
...
-- Unit nginx.service has begun starting up.
6月 09 21:47:41 k8s-master1 nginx[30694]: nginx: [emerg] unknown directive "stream" in /etc/nginx/nginx.conf:13
...

*解决
**应该是缺少modules模块
[root@k8s-master1 cfg]# yum -y install nginx-all-modules.noarch
[root@k8s-master1 cfg]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful


最后重启nginx服务



#查看keepalived工作状态
[root@k8s-master1 cfg]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:79:95:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.8.165.101/24 brd 10.8.165.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet 10.8.165.250/24 scope global secondary ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::e187:8e2f:2977:6d12/64 scope link 
       valid_lft forever preferred_lft forever
    inet6 fe80::82cf:7f96:a8f:69e1/64 scope link tentative dadfailed 
       valid_lft forever preferred_lft forever


[root@k8s-master2 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ab:04:62 brd ff:ff:ff:ff:ff:ff
    inet 10.8.165.113/24 brd 10.8.165.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:feab:462/64 scope link 
       valid_lft forever preferred_lft forever

在Nginx Master上可以看到，在ens33网卡绑定了10.8.165.250 虚拟IP，说明工作正常
```



![img](assets/二进制k8s搭建—V1.20(高可用)/1654783230660-a304657f-b26e-4a11-b688-524185d5e4cf.png)



![img](assets/二进制k8s搭建—V1.20(高可用)/1654783250766-569c3084-f9a4-45a3-b5be-1ae5bb4c4972.png)
**关闭主节点Nginx，测试VIP是否漂移到备节点服务器**



```shell
#关闭主节点Nginx，测试VIP是否漂移到备节点服务器
------------------------master1节点---------------------
[root@k8s-master1 cfg]# pkill nginx
[root@k8s-master1 cfg]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:79:95:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.8.165.101/24 brd 10.8.165.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::e187:8e2f:2977:6d12/64 scope link 
       valid_lft forever preferred_lft forever
    inet6 fe80::82cf:7f96:a8f:69e1/64 scope link tentative dadfailed 
       valid_lft forever preferred_lft forever


[root@k8s-master2 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ab:04:62 brd ff:ff:ff:ff:ff:ff
    inet 10.8.165.113/24 brd 10.8.165.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet 10.8.165.250/24 scope global secondary ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:feab:462/64 scope link 
       valid_lft forever preferred_lft forever


在Nginx Backup可以看到，在ens33网卡绑定了10.8.165.250 虚拟IP，漂移成功。


[root@k8s-master1 cfg]# systemctl start nginx
[root@k8s-master1 cfg]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:79:95:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.8.165.101/24 brd 10.8.165.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet 10.8.165.250/24 scope global secondary ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::e187:8e2f:2977:6d12/64 scope link 
       valid_lft forever preferred_lft forever
    inet6 fe80::82cf:7f96:a8f:69e1/64 scope link tentative dadfailed 
       valid_lft forever preferred_lft forever

[root@k8s-master2 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:ab:04:62 brd ff:ff:ff:ff:ff:ff
    inet 10.8.165.113/24 brd 10.8.165.255 scope global ens33
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:feab:462/64 scope link 
       valid_lft forever preferred_lft forever


当Nginx Master重新启动nginx服务后，VIP又从新漂移绑定到ens33上，而Nginx Backup的ens33网卡上的VIP解绑。
```



**访问负载均衡器测试**



```shell
#找K8s集群中任意一个节点，使用curl查看K8s版本测试，使用VIP访问

------------------------node1节点--------------------------
[root@k8s-master1 cfg]# curl -k https://10.8.165.250:16443/version
curl: (35) TCP connection reset by peer


[root@k8s-master1 cfg]# tail /var/log/nginx/k8s-access.log -f
10.8.165.102 192.168.31.71:6443, 192.168.31.74:6443 - [09/Jun/2022:22:12:56 +0800] 502 0, 0
10.8.165.102 k8s-apiserver - [09/Jun/2022:22:15:02 +0800] 502 0
10.8.165.101 192.168.31.74:6443, 192.168.31.71:6443 - [09/Jun/2022:22:16:02 +0800] 502 0, 0
10.8.165.102 192.168.31.74:6443, k8s-apiserver - [09/Jun/2022:22:17:56 +0800] 502 0, 0
10.8.165.102 k8s-apiserver - [09/Jun/2022:22:18:00 +0800] 502 0
10.8.165.102 192.168.31.71:6443, 192.168.31.74:6443 - [09/Jun/2022:22:18:51 +0800] 502 0, 0
10.8.165.102 192.168.31.71:6443, 192.168.31.74:6443 - [09/Jun/2022:22:20:03 +0800] 502 0, 0
10.8.165.102 192.168.31.71:6443, 192.168.31.74:6443 - [09/Jun/2022:22:22:21 +0800] 502 0, 0
10.8.165.102 192.168.31.71:6443, 192.168.31.74:6443 - [09/Jun/2022:22:26:21 +0800] 502 0, 0
10.8.165.102 192.168.31.71:6443, 192.168.31.74:6443 - [09/Jun/2022:22:28:38 +0800] 502 0, 0
10.8.165.102 192.168.31.71:6443, 192.168.31.74:6443 - [09/Jun/2022:22:41:47 +0800] 502 0, 0

到此还没结束，还有下面最关键的一步
```



## 3、修改所有Worker Node连接LB VIP



试想下，虽然我们增加了Master2 Node和负载均衡器，但是我们是从单Master架构扩容的，也就是说目前所有的Worker Node组件连接都还是Master1 Node，如果不改为连接VIP走负载均衡器，那么Master还是单点故障。
因此接下来就是要改所有Worker Node（kubectl get node命令查看到的节点）组件配置文件，由原来192.168.31.71修改为192.168.31.88（VIP）。



```shell
#在所有Worker Node执行
------------------master1、node1、node2节点----------------
[root@k8s-master1 cfg]# sed -i 's#10.8.165.101:6443#10.8.165.250:16443#' /opt/kubernetes/cfg/*

------------------------master2节点-----------------------
[root@k8s-master2 ~]# sed -i 's#10.8.165.113:6443#10.8.165.250:16443#' /opt/kubernetes/cfg/*


--------------master1、master2、node1、node2节点----------
systemctl restart kubelet kube-proxy


[root@k8s-master1 cfg]# kubectl get node
NAME          STATUS     ROLES    AGE     VERSION
k8s-master1   NotReady   <none>   3h28m   v1.20.4
k8s-master2   Ready      <none>   95m     v1.20.4
k8s-node1     NotReady   <none>   168m    v1.20.4
k8s-node2     NotReady   <none>   160m    v1.20.4
```



![img](assets/二进制k8s搭建—V1.20(高可用)/1655207932282-f7e6de99-f459-4b89-b0db-366299c11ac9.png)



# 九、高可用测试



## 1、宕机master2



```shell
模拟宕机master2


[root@k8s-master1 ~]# kubectl get node
NAME          STATUS     ROLES    AGE     VERSION
k8s-master1   Ready      <none>   5d      v1.20.4
k8s-master2   NotReady   <none>   4d22h   v1.20.4
k8s-node1     Ready      <none>   4d23h   v1.20.4
k8s-node2     Ready      <none>   4d23h   v1.20.4



#master1利用yaml文件创建pod
[root@k8s-master1 ~]# vim nginx-dep.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
 name: nginx-dep
spec:
 selector:
   matchLabels:
     app: nginx
 replicas: 2
 template:
   metadata:
     labels:
       app: nginx
   spec:
     containers:
     - name: nginx
       image: daocloud.io/library/nginx:1.12.0-alpine
       ports:
       - containerPort: 80


[root@k8s-master1 ~]# kubectl apply -f nginx-dep.yaml 
deployment.apps/nginx-dep created


#查看创建出来的pod
[root@k8s-master1 ~]# kubectl get pod
NAME                        READY   STATUS    RESTARTS   AGE
nginx-dep-86df8bbd5-ckb9q   1/1     Running   0          4m14s
nginx-dep-86df8bbd5-cqpcw   1/1     Running   0          4m14s

#查看产生的deployment
[root@k8s-master1 ~]# kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-dep   2/2     2            2           4m36s



测试结果，在master2宕机的情况下，master1仍能管理k8s集群
```



![img](assets/二进制k8s搭建—V1.20(高可用)/1655208477217-10606e0b-f1ce-4748-915f-e1f33145913f.png)



![img](assets/二进制k8s搭建—V1.20(高可用)/1655208430137-67bde632-11ab-4bab-b0e2-daeb1e827a67.png)



## 2、宕机master1



```shell
模拟宕机master1


[root@k8s-master2 cfg]# kubectl get node
NAME          STATUS     ROLES    AGE    VERSION
k8s-master1   NotReady   <none>   5d2h   v1.20.4
k8s-master2   Ready      <none>   5d     v1.20.4
k8s-node1     Ready      <none>   5d1h   v1.20.4
k8s-node2     Ready      <none>   5d1h   v1.20.4


#查看默认命名空间下的pod，发现先前宕机master2时，master1创建的pod，以及deployment
[root@k8s-master2 cfg]# kubectl get pod
NAME                        READY   STATUS    RESTARTS   AGE
nginx-dep-86df8bbd5-ckb9q   1/1     Running   0          97m
nginx-dep-86df8bbd5-cqpcw   1/1     Running   0          97m

[root@k8s-master2 cfg]# kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-dep   2/2     2            2           98m


#master2删除之前master1创建的pod、deployment
[root@k8s-master2 cfg]# kubectl delete deploy nginx-dep
deployment.apps "nginx-dep" deleted
[root@k8s-master2 cfg]# kubectl get deploy
No resources found in default namespace.

[root@k8s-master2 cfg]# kubectl get pod
No resources found in default namespace.
```



![img](assets/二进制k8s搭建—V1.20(高可用)/1655214172289-19590d7f-d7d2-4f56-a9a6-0d5ff90a75b3.png)



![img](assets/二进制k8s搭建—V1.20(高可用)/1655214109619-0e51d315-d0c7-4550-9af3-b9cb23aeeba9.png)



## 3、重启master1



```shell
#查看node状态
[root@k8s-master1 ~]# kubectl get node
NAME          STATUS   ROLES    AGE    VERSION
k8s-master1   Ready    <none>   5d2h   v1.20.4
k8s-master2   Ready    <none>   5d     v1.20.4
k8s-node1     Ready    <none>   5d1h   v1.20.4
k8s-node2     Ready    <none>   5d1h   v1.20.4


#重新构建之前master2删除的项目
[root@k8s-master1 ~]# kubectl apply -f nginx-dep.yaml 
deployment.apps/nginx-dep created

[root@k8s-master1 ~]# kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-dep   2/2     2            2           18s

[root@k8s-master1 ~]# kubectl get pod
NAME                        READY   STATUS    RESTARTS   AGE
nginx-dep-86df8bbd5-s8dgq   1/1     Running   0          32s
nginx-dep-86df8bbd5-w5sb8   1/1     Running   0          32s



#master2查看
[root@k8s-master2 cfg]# kubectl get pod
NAME                        READY   STATUS    RESTARTS   AGE
nginx-dep-86df8bbd5-s8dgq   1/1     Running   0          64s
nginx-dep-86df8bbd5-w5sb8   1/1     Running   0          64s

[root@k8s-master2 cfg]# kubectl get deploy
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-dep   2/2     2            2           100s


一切正常！！！
```



![img](assets/二进制k8s搭建—V1.20(高可用)/1655215442160-1e31f279-d6ba-4d36-b86a-fb7a85840355.png)



![img](assets/二进制k8s搭建—V1.20(高可用)/1655215494955-79ad6733-b3eb-4b2f-9251-b9c975386ca6.png)
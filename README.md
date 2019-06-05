# SaltStack自动化部署Kubernetes
- SaltStack自动化部署Kubernetes v1.12.3版本（支持TLS双向认证、RBAC授权、Flannel网络、ETCD集群、Kuber-Proxy使用LVS等）。

## 版本明细：Release-v1.12.3
- 测试通过系统：CentOS 7.6.1810
- salt-ssh:     2018.3.3
- Kubernetes：  v1.12.3
- Etcd:         v3.3.1
- Docker:       18.09.0-ce
- Flannel：     v0.10.0
- CNI-Plugins： v0.7.0
建议部署节点：最少三个节点，请配置好主机名解析（必备）

## 架构介绍
1. 使用Salt Grains进行角色定义，增加灵活性。
2. 使用Salt Pillar进行配置项管理，保证安全性。
3. 使用Salt SSH执行状态，不需要安装Agent，保证通用性。
4. 使用Kubernetes当前稳定版本v1.12.3，修复安全漏洞。

# 使用手册
<table border="0">
    <tr>
        <td><strong>手动部署</strong></td>
        <td><a href="docs/init.md">1.系统初始化</a></td>
        <td><a href="docs/ca.md">2.CA证书制作</a></td>
        <td><a href="docs/etcd-install.md">3.ETCD集群部署</a></td>
        <td><a href="docs/master.md">4.Master节点部署</a></td>
        <td><a href="docs/node.md">5.Node节点部署</a></td>
        <td><a href="docs/flannel.md">6.Flannel部署</a></td>
        <td><a href="docs/app.md">7.应用创建</a></td>
        <td><a href="docs/harbor.md">8.Harbor镜像仓库搭建</a></td>
        <td><a href="docs/registry.md">9.registry镜像仓库搭建</a></td>
        
    </tr>
    <tr>
        <td><strong>必备插件</strong></td>
        <td><a href="docs/coredns.md">1.CoreDNS部署</a></td>
        <td><a href="docs/dashboard.md">2.Dashboard部署</a></td>
        <td><a href="docs/heapster.md">3.Heapster部署</a></td>
        <td><a href="docs/ingress.md">4.Ingress部署</a></td>
        <td><a href="https://github.com/unixhot/devops-x">5.CI/CD</a></td>
        <td><a href="docs/helm.md">6.Helm部署</a></td>
    </tr>
</table>

## 案例架构图

  ![架构图](https://github.com/weiyanwei412/k8s-install/blob/master/docs/k8s-new.png)
  
## 0.系统初始化(必备)
1. 设置主机名！！！ 
```
[root@k8s-master ~]# cat /etc/hostname 
k8s-master

[root@k8s-node1 ~]# cat /etc/hostname 
k8s-node1

[root@k8s-node2 ~]# cat /etc/hostname 
k8s-node2

[root@etcd1 ~]# cat /etc/hostname 
etcd1

[root@etcd2 ~]# cat /etc/hostname 
etcd2

[root@etcd3 ~]# cat /etc/hostname 
etcd3


```

```shell
# For more information, see sysctl.conf(5) and sysctl.d(5).
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

vm.swappiness = 0
net.ipv4.neigh.default.gc_stale_time=120
net.ipv4.ip_forward = 1

# see details in https://help.aliyun.com/knowledge_detail/39428.html
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce=2
net.ipv4.conf.all.arp_announce=2


# see details in https://help.aliyun.com/knowledge_detail/41334.html
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
kernel.sysrq = 1

#iptables透明网桥的实现
# NOTE: kube-proxy 要求 NODE 节点操作系统中要具备 /sys/module/br_netfilter 文件，而且还要设置 bridge-nf-call-iptables=1，如果不满足要求，那么 kube-proxy 只是将检查信息记录到日志中，kube-proxy 仍然会正常运行，但是这样通过 Kube-proxy 设置的某些 iptables 规则就不会工作。

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
```

2. 设置/etc/hosts保证主机名能够解析    一定要每个节点都设置hosts ***
```
cat >>/etc/hosts<<-EOF
172.18.1.7 etcd1
172.18.1.8 etcd2
172.18.1.9 etcd3
172.18.1.10 k8s-master01
172.18.1.11 k8s-node1
172.18.1.12 k8s-node2
EOF


```
3. 关闭SELinux和防火墙

4.以上必备条件必须严格检查，否则，一定不会部署成功！

## 1.设置部署节点到其它所有节点的SSH免密码登录（包括本机）
```
[root@k8s-master01 ~]# ssh-keygen -t rsa
[root@k8s-master01 ~]# ssh-copy-id k8s-master01
[root@k8s-master01 ~]# ssh-copy-id k8s-node1
[root@k8s-master01 ~]# ssh-copy-id k8s-node2
[root@k8s-master01 ~]# ssh-copy-id etcd1
[root@k8s-master01 ~]# ssh-copy-id etcd2
[root@k8s-master01 ~]# ssh-copy-id etcd3
```

## 2.安装Salt-SSH并克隆本项目代码。

2.1 安装Salt SSH（注意：老版本的Salt SSH不支持Roster定义Grains，需要2017.7.4以上版本）
```
[root@k8s-master01 ~]# curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@k8s-master01 ~]# curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
[root@k8s-master01 ~]# yum install https://mirrors.aliyun.com/saltstack/yum/redhat/salt-repo-latest-2.el7.noarch.rpm
[root@k8s-master01 ~]# sed -i "s/repo.saltstack.com/mirrors.aliyun.com\/saltstack/g" /etc/yum.repos.d/salt-latest.repo
[root@k8s-master01 ~]# yum install -y salt-ssh git unzip
```

2.2 获取本项目代码，并放置在/srv目录
```
[root@k8s-master01 ~]# git clone https://github.com/weiyanwei412/k8s-install.git
[root@k8s-master01 ~]# cd salt-kubernetes/
[root@k8s-master01 ~]# mv * /srv/
[root@k8s-master01 srv]# /bin/cp /srv/roster /etc/salt/roster
[root@k8s-master01 srv]# /bin/cp /srv/master /etc/salt/master
```

2.4 下载二进制文件，也可以自行官方下载，为了方便国内用户访问，请在百度云盘下载,下载files.zip
下载完成后，将文件移动到/srv/salt/k8s/目录下，并解压
Kubernetes二进制文件下载地址： 链接：https://pan.baidu.com/s/1Vfs9K-9LkIkDI_iG3osHMg   提取码：tbvt 

```
[root@k8s-master01 ~]# cd /srv/salt/k8s/
[root@k8s-master01 k8s]# unzip files.zip
[root@k8s-master01 k8s]# rm -f files.zip
[root@k8s-master01 k8s]# ls -l files/
total 0
drwxr-xr-x. 2 root root  94 Jun  3 19:12 cfssl-1.2
drwxr-xr-x. 2 root root 195 Jun  3 19:12 cni-plugins-amd64-v0.7.0
drwxr-xr-x. 2 root root  33 Jun  3 19:12 etcd-v3.3.1-linux-amd64
drwxr-xr-x. 2 root root  47 Jun  3 19:12 flannel-v0.10.0-linux-amd64
drwxr-xr-x. 3 root root  17 Jun  3 19:12 k8s-v1.12.3

```

## 3.Salt SSH管理的机器以及角色分配

- k8s-role: 用来设置K8S的角色
- etcd-role: 用来设置etcd的角色，如果只需要部署一个etcd，只需要在一台机器上设置即可
- etcd-name: 如果对一台机器设置了etcd-role就必须设置etcd-name

```
[root@k8s-master01 ~]# vim /etc/salt/roster 
k8s-master01:
    host: 172.18.1.10
    user: root
    port: 22
    priv: /root/.ssh/id_rsa
    minion_opts:
      grains:
         k8s-role: master
k8s-node1:
    host: 172.18.1.11
    user: root
    port: 22
    priv: /root/.ssh/id_rsa
    minion_opts:
      grains:
         k8s-role: node
k8s-node2:
    host: 172.18.1.12
    user: root
    port: 22
    priv: /root/.ssh/id_rsa
    minion_opts:
      grains:
         k8s-role: node
etcd1:
    host: 172.18.1.7
    user: root
    port: 22
    priv: /root/.ssh/id_rsa
    minion_opts:
      grains:
         etcd-role: node
         etcd-name: etcd-node1
etcd2:
    host: 172.18.1.8
    user: root
    port: 22
    priv: /root/.ssh/id_rsa
    minion_opts:
      grains:
         etcd-role: node
         etcd-name: etcd-node2
etcd3:
    host: 172.18.1.9
    user: root
    port: 22
    priv: /root/.ssh/id_rsa
    minion_opts:
      grains:
         etcd-role: node
         etcd-name: etcd-node3

```

## 4.修改对应的配置参数，本项目使用Salt Pillar保存配置
```
[root@k8s-master01 ~]# vim /srv/pillar/k8s.sls
#设置Master的VIP地址(必须修改)
MASTER_VIP: "172.18.1.88"

#设置Master的IP地址(必须修改)
MASTER_IP: "172.18.1.10"

#设置ETCD集群访问地址（必须修改）
ETCD_ENDPOINTS: "https://172.18.1.7:2379,https://172.18.1.8:2379,https://172.18.1.9:2379"

#设置ETCD集群初始化列表（必须修改）
ETCD_CLUSTER: "etcd-node1=https://172.18.1.7:2380,etcd-node2=https://172.18.1.8:2380,etcd-node3=https://172.18.1.9:2380"

#通过Grains FQDN自动获取本机IP地址，请注意保证主机名解析到本机IP地址
NODE_IP: {{ grains['fqdn_ip4'][0] }}

#设置BOOTSTARP的TOKEN，可以自己生成
BOOTSTRAP_TOKEN: "ad6d5bb607a186796d8861557df0d17f"

#配置Service IP地址段
SERVICE_CIDR: "10.1.0.0/16"

#Kubernetes服务 IP (从 SERVICE_CIDR 中预分配)
CLUSTER_KUBERNETES_SVC_IP: "10.1.0.1"

#Kubernetes DNS 服务 IP (从 SERVICE_CIDR 中预分配)
CLUSTER_DNS_SVC_IP: "10.1.0.2"

#设置Node Port的端口范围
NODE_PORT_RANGE: "20000-40000"

#设置POD的IP地址段
POD_CIDR: "10.2.0.0/16"

#设置集群的DNS域名
CLUSTER_DNS_DOMAIN: "cluster.local."


修改证书授权文件
栗子
cat /srv/salt/k8s/templates/kube-api-server/kubernetes-csr.json.template

{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "{{ NODE_IP }}",
    "{{ CLUSTER_KUBERNETES_SVC_IP }}",
    "172.18.1.9", #master02
    "172.18.1.10", #master01
    "172.18.1.8", #master03
    "172.18.1.88", #masterVIP 
    "172.18.1.11",
    "172.18.1.12",
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
      "ST": "Shanghai",
      "L": "Shanghai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}


```
变量查看
salt-ssh '*' pillar.items  

## 5.执行SaltStack状态

5.1 测试Salt SSH联通性
```
[root@k8s-master01 ~]# salt-ssh '*' test.ping
```
执行高级状态，会根据定义的角色再对应的机器部署对应的服务

node节点解析
````
salt-ssh '*'  grains.item fqdn_ip4

````


5.2 部署Etcd，由于Etcd是基础组建，需要先部署，目标为部署etcd的节点。
```
[root@k8s-master01 ~]# salt-ssh -L 'etcd1,etcd2,etcd3' state.sls k8s.etcd
```
注：如果执行失败，新手建议推到重来，请检查各个节点的主机名解析是否正确（监听的IP地址依赖主机名解析）。

5.3 部署K8S集群
```
[root@k8s-master01 ~]# salt-ssh '*' state.highstate
```
由于包比较大，这里执行时间较长，5分钟+，喝杯咖啡休息一下，如果执行有失败可以再次执行即可！

## 6.测试Kubernetes安装
```
[root@k8s-master01 ~]# source /etc/profile
[root@k8s-master01 ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
[root@k8s-master01 ~]# kubectl get node
NAME            STATUS    ROLES     AGE       VERSION
172.18.1.11   Ready     <none>    1m        v1.12.3
172.18.1.12   Ready     <none>    1m        v1.12.3
```
## 7.测试Kubernetes集群和Flannel网络
```
[root@k8s-master01 ~]# kubectl run net-test --image=alpine --replicas=2 sleep 360000
deployment "net-test" created
需要等待拉取镜像，可能稍有的慢，请等待。
[root@k8s-master01 ~]# kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP          NODE          NOMINATED NODE
net-test-5786f8b986-tjxhb   1/1     Running   0          17h   10.2.83.2   172.18.1.12   <none>
net-test-5786f8b986-wmsm4   1/1     Running   0          17h   10.2.83.3   172.18.1.12   <none>

测试联通性，如果都能ping通，说明Kubernetes集群部署完毕，有问题请QQ群交流。
[root@k8s-master ~]# ping -c 1 10.2.83.2
PING 10.2.12.2 (10.2.12.2) 56(84) bytes of data.
64 bytes from 10.2.12.2: icmp_seq=1 ttl=61 time=8.72 ms

--- 10.2.12.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 8.729/8.729/8.729/0.000 ms

[root@k8s-master ~]# ping -c 1 10.2.83.3
PING 10.2.24.2 (10.2.24.2) 56(84) bytes of data.
64 bytes from 10.2.24.2: icmp_seq=1 ttl=61 time=22.9 ms

--- 10.2.24.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 22.960/22.960/22.960/0.000 ms
```
## 8.如何新增Kubernetes节点

- 1.设置SSH无密码登录
- 2.在/etc/salt/roster里面，增加对应的机器
- 3.执行SaltStack状态salt-ssh '*' state.highstate。
```
[root@k8s-master01 ~]# vim /etc/salt/roster 
k8s-node3:
  host: 172.18.1.13
  user: root
  priv: /root/.ssh/id_rsa
  minion_opts:
    grains:
      k8s-role: node
[root@k8s-master ~]# salt-ssh 'k8s-node3' state.highstate
```
## 9.如何新增Kubernetes Master节点

- 1.设置SSH无密码登录
- 2.在/etc/salt/roster里面，增加对应的机器
- 3.执行SaltStack状态salt-ssh 'k8s-master02' state.highstate。
```
[root@k8s-master01 ~]# vim /etc/salt/roster 
k8s-master02:
  host: 172.18.1.9
  user: root
  priv: /root/.ssh/id_rsa
  minion_opts:
    grains:
      k8s-role: master
[root@k8s-master01 ~]# salt-ssh -L 'k8s-master02' state.highstate
```

## 10.下一步要做什么？

你可以安装Kubernetes必备的插件
<table border="0">
    <tr>
        <td><strong>必备插件</strong></td>
        <td><a href="docs/coredns.md">1.CoreDNS部署</a></td>
        <td><a href="docs/dashboard.md">2.Dashboard部署</a></td>
        <td><a href="docs/heapster.md">3.Heapster部署</a></td>
        <td><a href="docs/ingress.md">4.Ingress部署</a></td>
        <td><a href="https://github.com/unixhot/devops-x">5.CI/CD</a></td>
    </tr>
</table>

注意：不要相信自己，要相信电脑！！！

# 手动部署
- [系统初始化](docs/init.md)
- [CA证书制作](docs/ca.md)
- [ETCD集群部署](docs/etcd-install.md)
- [Master节点部署](docs/master.md)
- [Node节点部署](docs/node.md)
- [Flannel网络部署](docs/flannel.md)
- [创建第一个K8S应用](docs/app.md)
- [CoreDNS和Dashboard部署](docs/dashboard.md)


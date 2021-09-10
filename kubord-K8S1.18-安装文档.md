## 配置要求

对于 Kubernetes 初学者，在搭建K8S集群时，推荐在阿里云或腾讯云采购如下配置：（您也可以使用自己的虚拟机、私有云等您最容易获得的 Linux 环境）

- 至少2台 **2核4G** 的服务器
- **Cent OS 7.6 / 7.7 / 7.8**

[【云上优选 特惠来袭】华为云回馈用户，产品低至2折(opens new window)](https://activity.huaweicloud.com/discount_area_v5/index.html?fromacct=36cf686d-2650-4107-baa4-f0dc3c860df4&utm_source=V1g3MDY4NTY=&utm_medium=cps&utm_campaign=201905)

[【腾讯云】云产品限时秒杀，爆款1核2G云服务器，首年99元(opens new window)](https://cloud.tencent.com/act/cps/redirect?redirect=1062&cps_key=2ee6baa049659f4713ddc55a51314372&from=console)

**安装后的软件版本为**

- Kubernetes v1.18.x
  - calico 3.13.1
  - nginx-ingress 1.5.5
- Docker 19.03.8

> 如果要安装 Kubernetes 历史版本，请参考：
>
> - [安装 Kubernetes v1.17.x 单Master节点](https://www.kuboard.cn/install/history-k8s/install-k8s-1.17.x.html)
> - [安装 Kubernetes v1.16.3 单Master节点](https://www.kuboard.cn/install/history-k8s/install-k8s-1.16.3.html)
> - [安装 Kubernetes v1.15.4 单Master节点](https://www.kuboard.cn/install/history-k8s/install-k8s-1.15.4.html)

安装后的拓扑图如下：[下载拓扑图源文件](https://www.kuboard.cn/kuboard.rp) 使用Axure RP 9.0可打开该文件

强烈建议初学者先按照此文档完成安装，在对 K8S 有更多理解后，再参考文档 [安装Kubernetes高可用](https://www.kuboard.cn/install/history-k8s/install-kubernetes.html)

![Kubernetes安装：Kubernetes安装拓扑图](https://www.kuboard.cn/images/topology/k8s.png)

关于二进制安装

kubeadm 是 Kubernetes 官方支持的安装方式，“二进制” 不是。本文档采用 kubernetes.io 官方推荐的 kubeadm 工具安装 kubernetes 集群。

## [#](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#检查-centos-hostname)检查 centos / hostname

```sh
# 在 master 节点和 worker 节点都要执行
cat /etc/redhat-release

# 此处 hostname 的输出将会是该机器在 Kubernetes 集群中的节点名字
# 不能使用 localhost 作为节点的名字
hostname

# 请使用 lscpu 命令，核对 CPU 信息
# Architecture: x86_64    本安装文档不支持 arm 架构
# CPU(s):       2         CPU 内核数量不能低于 2
lscpu
 
        已复制到剪贴板！
    
```

1
2
3
4
5
6
7
8
9
10
11

**操作系统兼容性**

| CentOS 版本 | 本文档是否兼容 | 备注                                |
| ----------- | -------------- | ----------------------------------- |
| 7.8         | 😄              | 已验证                              |
| 7.7         | 😄              | 已验证                              |
| 7.6         | 😄              | 已验证                              |
| 7.5         | 😞              | 已证实会出现 kubelet 无法启动的问题 |
| 7.4         | 😞              | 已证实会出现 kubelet 无法启动的问题 |
| 7.3         | 😞              | 已证实会出现 kubelet 无法启动的问题 |
| 7.2         | 😞              | 已证实会出现 kubelet 无法启动的问题 |



修改 hostname

如果您需要修改 hostname，可执行如下指令：

```sh
# 修改 hostname
hostnamectl set-hostname your-new-host-name
# 查看修改结果
hostnamectl status
# 设置 hostname 解析
echo "127.0.0.1   $(hostname)" >> /etc/hosts
 
        已复制到剪贴板！
    
```

1
2
3
4
5
6

## [#](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#检查网络)检查网络

在所有节点执行命令



 











 



 





```text
[root@demo-master-a-1 ~]$ ip route show
default via 172.21.0.1 dev eth0 
169.254.0.0/16 dev eth0 scope link metric 1002 
172.21.0.0/20 dev eth0 proto kernel scope link src 172.21.0.12 

[root@demo-master-a-1 ~]$ ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:16:3e:12:a4:1b brd ff:ff:ff:ff:ff:ff
    inet 172.17.216.80/20 brd 172.17.223.255 scope global dynamic eth0
       valid_lft 305741654sec preferred_lft 305741654sec
 
        已复制到剪贴板！
    
```

1
2
3
4
5
6
7
8
9
10
11
12
13
14

kubelet使用的IP地址

- `ip route show` 命令中，可以知道机器的默认网卡，通常是 `eth0`，如 ***default via 172.21.0.23 dev eth0***
- `ip address` 命令中，可显示默认网卡的 IP 地址，Kubernetes 将使用此 IP 地址与集群内的其他节点通信，如 `172.17.216.80`
- 所有节点上 Kubernetes 所使用的 IP 地址必须可以互通（无需 NAT 映射、无安全组或防火墙隔离）

## [#](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#安装docker及kubelet)安装docker及kubelet

再看看我是否符合安装条件

使用 root 身份在所有节点执行如下代码，以安装软件：

- docker
- nfs-utils
- kubectl / kubeadm / kubelet

- [快速安装](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#)
- [手动安装](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#)

**请将脚本最后的 1.18.9 替换成您需要的版本号，** 脚本中间的 v1.18.x 不要替换

> docker hub 镜像请根据自己网络的情况任选一个
>
> - 第四行为腾讯云 docker hub 镜像
> - 第六行为DaoCloud docker hub 镜像
> - 第八行为华为云 docker hub 镜像
> - 第十行为阿里云 docker hub 镜像

```sh
# 在 master 节点和 worker 节点都要执行
# 最后一个参数 1.18.9 用于指定 kubenetes 版本，支持所有 1.18.x 版本的安装
# 腾讯云 docker hub 镜像
# export REGISTRY_MIRROR="https://mirror.ccs.tencentyun.com"
# DaoCloud 镜像
# export REGISTRY_MIRROR="http://f1361db2.m.daocloud.io"
# 华为云镜像
# export REGISTRY_MIRROR="https://05f073ad3c0010ea0f4bc00b7105ec20.mirror.swr.myhuaweicloud.com"
# 阿里云 docker hub 镜像
export REGISTRY_MIRROR=https://registry.cn-hangzhou.aliyuncs.com
curl -sSL https://kuboard.cn/install-script/v1.18.x/install_kubelet.sh | sh -s 1.18.9
 
        已复制到剪贴板！
    
```

1
2
3
4
5
6
7
8
9
10
11

## [#](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#初始化-master-节点)初始化 master 节点

关于初始化时用到的环境变量

- **APISERVER_NAME** 不能是 master 的 hostname
- **APISERVER_NAME** 必须全为小写字母、数字、小数点，不能包含减号
- **POD_SUBNET** 所使用的网段不能与 ***master节点/worker节点*** 所在的网段重叠。该字段的取值为一个 [CIDR](https://www.kuboard.cn/glossary/cidr.html) 值，如果您对 CIDR 这个概念还不熟悉，请仍然执行 export POD_SUBNET=10.100.0.1/16 命令，不做修改

- [快速初始化](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#)
- [手动初始化](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#)

**请将脚本最后的 1.18.9 替换成您需要的版本号，** 脚本中间的 v1.18.x 不要替换











 



```sh
# 只在 master 节点执行
# 替换 x.x.x.x 为 master 节点实际 IP（请使用内网 IP）
# export 命令只在当前 shell 会话中有效，开启新的 shell 窗口后，如果要继续安装过程，请重新执行此处的 export 命令
export MASTER_IP=x.x.x.x
# 替换 apiserver.demo 为 您想要的 dnsName
export APISERVER_NAME=apiserver.demo
# Kubernetes 容器组所在的网段，该网段安装完成后，由 kubernetes 创建，事先并不存在于您的物理网络中
export POD_SUBNET=10.100.0.1/16
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts
curl -sSL https://kuboard.cn/install-script/v1.18.x/init_master.sh | sh -s 1.18.9
 
        已复制到剪贴板！
    
```

1
2
3
4
5
6
7
8
9
10

> 感谢 [https://github.com/zhangguanzhang/google_containers (opens new window)](https://github.com/zhangguanzhang/google_containers)提供最新的 google_containers 国内镜像

如果出错点这里

**检查 master 初始化结果**

```sh
# 只在 master 节点执行

# 执行如下命令，等待 3-10 分钟，直到所有的容器组处于 Running 状态
watch kubectl get pod -n kube-system -o wide

# 查看 master 节点初始化结果
kubectl get nodes -o wide
 
        已复制到剪贴板！
    
```

1
2
3
4
5
6
7

如果出错点这里

## [#](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#初始化-worker节点)初始化 worker节点

### [#](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#获得-join命令参数)获得 join命令参数

**在 master 节点上执行**

```sh
# 只在 master 节点执行
kubeadm token create --print-join-command
 
        已复制到剪贴板！
    
```

1
2

可获取kubeadm join 命令及参数，如下所示

```sh
# kubeadm token create 命令的输出
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
 
        已复制到剪贴板！
    
```

1
2

有效时间

该 token 的有效时间为 2 个小时，2小时内，您可以使用此 token 初始化任意数量的 worker 节点。

### [#](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#初始化worker)初始化worker

**针对所有的 worker 节点执行**

```sh
# 只在 worker 节点执行
# 替换 x.x.x.x 为 master 节点的内网 IP
export MASTER_IP=x.x.x.x
# 替换 apiserver.demo 为初始化 master 节点时所使用的 APISERVER_NAME
export APISERVER_NAME=apiserver.demo
echo "${MASTER_IP}    ${APISERVER_NAME}" >> /etc/hosts

# 替换为 master 节点上 kubeadm token create 命令的输出
kubeadm join apiserver.demo:6443 --token mpfjma.4vjjg8flqihor4vt     --discovery-token-ca-cert-hash sha256:6f7a8e40a810323672de5eee6f4d19aa2dbdb38411845a1bf5dd63485c43d303
 
        已复制到剪贴板！
    
```

1
2
3
4
5
6
7
8
9

如果出错点这里

### [#](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#检查初始化结果)检查初始化结果

在 master 节点上执行

```sh
# 只在 master 节点执行
kubectl get nodes -o wide
 
        已复制到剪贴板！
    
```

1
2

输出结果如下所示：

```sh
[root@demo-master-a-1 ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
demo-master-a-1   Ready    master   5m3s    v1.18.x
demo-worker-a-1   Ready    <none>   2m26s   v1.18.x
demo-worker-a-2   Ready    <none>   3m56s   v1.18.x
 
        已复制到剪贴板！
    
```

1
2
3
4
5

## [#](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#安装-ingress-controller)安装 Ingress Controller

- [快速初始化](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#)
- [卸载IngressController](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#)
- [YAML文件](https://www.kuboard.cn/install/history-k8s/install-k8s-1.18.x.html#)

**在 master 节点上执行**

```sh
# 只在 master 节点执行
kubectl apply -f https://kuboard.cn/install-script/v1.18.x/nginx-ingress.yaml
 
        已复制到剪贴板！
    
```

1
2

**配置域名解析**

将域名 *.demo.yourdomain.com 解析到 demo-worker-a-2 的 IP 地址 z.z.z.z （也可以是 demo-worker-a-1 的地址 y.y.y.y）

**验证配置**

在浏览器访问 a.demo.yourdomain.com，将得到 404 NotFound 错误页面

提示

许多初学者在安装 Ingress Controller 时会碰到问题，请不要灰心，可暂时跳过 ***安装 Ingress Controller*** 这个部分，等您学完 www.kuboard.cn 上 [Kubernetes 入门](https://www.kuboard.cn/learning/k8s-basics/kubernetes-basics.html) 以及 [通过互联网访问您的应用程序](https://www.kuboard.cn/learning/k8s-intermediate/service/ingress.html) 这两部分内容后，再来回顾 Ingress Controller 的安装。

也可以参考 [Install Nginx Ingress(opens new window)](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/)

WARNING

如果您打算将 Kubernetes 用于生产环境，请参考此文档 [Installing Ingress Controller (opens new window)](https://github.com/nginxinc/kubernetes-ingress/blob/v1.5.3/docs/installation.md)，完善 Ingress 的配置
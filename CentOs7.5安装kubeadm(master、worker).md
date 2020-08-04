***

### CentOs7.5安装kubeadm(master、worker)

***

# 1. 准备

## 1.1升级内核

** **

升级内核需要先导入elrepo的key，然后安装elrepo的yum源：

拷贝elrepo文件夹到临时文件夹，如/usr/local/tmp

cd/usr/local/tmp

rpm --importRPM-GPG-KEY-elrepo.org 

rpm--install elrepo-release-7.0-2.el7.elrepo.noarch.rpm

仓库启用后，你可以使用下面的命令列出可用的内核相关包，如下图：

yum--disablerepo="*" --enablerepo="elrepo-kernel" listavailable

 

下载内核

\# yum -y--enablerepo=elrepo-kernel install kernel-lt.x86_64 kernel-lt-devel.x86_64

 

查看系统可用内核

cat/boot/grub2/grub.cfg |grep menuentry

开机默认内核

grub2-set-default"CentOS Linux (4.4.176-1.el7.elrepo.x86_64) 7 (Core)" 

查看默认启动项 grub2-editenv list 

 

## 1.2系统配置

在安装之前，需要先做如下准备：

一台CentOS7.5主机（如172.16.34.168），用于装私有镜像仓库（详见2.1步骤）

两台CentOS 7.5主机（内核必须是4.0及以上），用于安装k8s集群，添加host信息：

cat /etc/hosts

172.16.34.172k8s-master1 

172.16.34.173k8s-worker1

禁用防火墙

systemctl stopfirewalld

systemctldisable firewalld

如果各个主机需要启用防火墙，需要开放Kubernetes各个组件所需要的端口，可以查看[Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/)中的”Check required ports”一节



禁用SELINUX

setenforce 0

vi/etc/selinux/config 

SELINUX=disabled

 

关闭交换分区 

swapoff -a 

yes | cp/etc/fstab /etc/fstab_bak 

cat/etc/fstab_bak |grep -v swap > /etc/fstab

 

创建/etc/sysctl.d/k8s.conf文件，添加如下内容：

net.bridge.bridge-nf-call-ip6tables= 1 

net.bridge.bridge-nf-call-iptables= 1 

net.ipv4.ip_forward= 1

vm.swappiness=0

 

执行命令使修改生效

modprobebr_netfilter 

sysctl -p/etc/sysctl.d/k8s.conf

 

Kubernetes要求集群中所有机器具有不同的Mac地址、产品uuid、Hostname。可以使用如下命令查看Mac和uuid

\# 所有主机：检查UUID和Mac 

cat/sys/class/dmi/id/product_uuid 

ip link

 

## 1.3kube-proxy开启ipvs的前置条件

在所有的Kubernetes节点master和worker上执行以下脚本:

cat >/etc/sysconfig/modules/ipvs.modules <<EOF 

\#!/bin/bash 

ipvs_modules="ip_vsip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_ship_vs_fo ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack" 

forkernel_module in \${ipvs_modules}; do 

/sbin/modinfo-F filename \${kernel_module} > /dev/null 2>&1 

if [ $? -eq 0]; then 

/sbin/modprobe\${kernel_module} 

fi 

done 

EOF 

chmod 755/etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules&& lsmod | grep ip_vs

## 1.4安装Docker（略）

# 2.安装kubernetes

## 2.1安装私有仓库

如果不能翻墙，需要使用私有镜像源，则还需要为[docker](https://www.kubernetes.org.cn/tags/docker)做如下配置，将K8s官方镜像库的几个域名设置为insecure-registry，然后设置hosts使它们指向私有源。

 

\# 在master和worker上：http私有源配置 

\# 为Docker配置一下私有源 

mkdir -p/etc/docker 

echo -e'{\n"insecure-registries":["k8s.gcr.io","gcr.io", "quay.io"]\n}' > /etc/docker/daemon.json 

systemctl restartdocker 

 

\# 此处应当修改为registry所在机器的IP 

REGISTRY_HOST="172.16.34.168"

\# 设置Hosts 

yes | cp/etc/hosts /etc/hosts_bak 

cat/etc/hosts_bak|grep -vE '(gcr.io|harbor.io|quay.io)' > /etc/hosts 

echo""" $REGISTRY_HOST gcr.io harbor.io k8s.gcr.io quay.io""" >> /etc/hosts

 

将registry文件夹下的文件放置到registry机器上，并在registry主机上加载、启动该镜像

 

\# registry：启动私有镜像库 

docker load -i/path/to/k8s-repo-1.13.0 

docker run--restart=always -d -p 80:5000 --name repoharbor.io:1180/system/k8s-repo:v1.13.0

该镜像库中包含如下镜像，全部来源于官方镜像站。1.13.0版本镜像列表：

![img](file:///C:\Users\keven\AppData\Local\Temp\msohtmlclip1\01\clip_image001.png)

 

## 2.2kubernetes基本安装

将kubeadm文件中的文件放在k8s各个master和worker主机上

\# master &worker安装kubernetes 

yum install -yipvsadm 

yuminstall socat

cd/path/to/downloaded/file 

tar -xzvfk8s-v1.13.0-rpms.tgz 

cd k8s-v1.13.0

rpm -Uvh *--force 

systemctlenable kubelet 

kubeadmversion -o short

 

## 2.3 使用kubeadm init初始化集群

选择k8s-master1作为Master Node，在k8s-master1上执行下面的命令：

kubeadm init \

--kubernetes-version=v1.13.0 \

--pod-network-cidr=10.244.0.0/16 \

--apiserver-advertise-address=172.16.34.172

因为我们选择flannel作为Pod网络插件，所以上面的命令指定–pod-network-cidr=10.244.0.0/16。

 

执行初始化成功界面上提示的命令：

mkdir -p$HOME/.kube

sudo cp -i/etc/kubernetes/admin.conf $HOME/.kube/config

sudo chown$(id -u):$(id -g) $HOME/.kube/config

 

最后提示的kubeadm join命令要记下，后面加入node节点会用到。需要注意的是这个命令后面的token有效期是24小时，过期或者token重新生成后，之前的token就失效了，但是用这个过期的token执行kubeadm join命令时不会给出提示而其实是不成功的。

 

查看一下集群状态：

kubectl get cs

NAME STATUSMESSAGE ERROR 

controller-managerHealthy ok 

schedulerHealthy ok 

etcd-0 Healthy{"health": "true"}

确认个组件都处于healthy状态。

## 2.4 安装Pod Network

将flannel文件夹下的文件放入临时文件夹，如/usr/local/tmp

导入镜像quay.io/coreos/flannel，当前使用的是v0.10.0-amd64版本

执行docker load-i /usr/local/tmp/flannel.tgz

修改kube-flannel.yml文件，flanneld启动参数加上–iface=<iface-name>，可以使用ip addr命令查看当前主机的网卡名称ensxxx，添加时注意节点是amd64的那个

 

......containers:

 - name: kube-flannel

image: quay.io/coreos/flannel:v0.10.0-amd64 

command: 

\- /opt/bin/flanneld 

args: 

\- --ip-masq 

\- --kube-subnet-mgr 

\- --iface=ens160

......

kubectl apply-f /usr/local/tmp/kube-flannel.yml 

clusterrole.rbac.authorization.k8s.io/flannelcreated 

clusterrolebinding.rbac.authorization.k8s.io/flannelcreated 

serviceaccount/flannelcreated configmap/kube-flannel-cfg created 

daemonset.extensions/kube-flannel-ds-amd64created 

daemonset.extensions/kube-flannel-ds-arm64created 

daemonset.extensions/kube-flannel-ds-armcreated 

daemonset.extensions/kube-flannel-ds-ppc64lecreated 

daemonset.extensions/kube-flannel-ds-s390xcreated

 

使用kubectl getpod –all-namespaces -o wide查看所有的Pod是否都处于Running状态。

## 2.5 master node参与工作负载

使用kubeadm初始化的集群，出于安全考虑Pod不会被调度到Master Node上，也就是说Master Node不参与工作负载。这是因为当前的master节点node1被打上了

node-role.kubernetes.io/master:NoSchedule的污点：

kubectldescribe node k8s-master1 | grep Taint Taints:node-role.kubernetes.io/master:NoSchedule

因为这里搭建的是测试环境，去掉这个污点使node1参与工作负载：

kubectl taintnodes k8s-master1 node-role.kubernetes.io/master-node " k8s-master1"untainted

上述命令在有的机器上会报错error:at least one taint update is required

可执行下面的命令：

kubectl taintnodes k8s-master1 node-role.kubernetes.io/k8s-master1 untainted

 

## 2.6 向Kubernetes集群中添加Node节点

需要先将flannel镜像导入到node节点上，然后执行kubeadm join命令，即2.3步骤执行kubeadm init命令初始化集群成功后界面显示出来的内容

 

## 2.7安装nfs工具

yum -y install nfs-utils

 

执行开机重启

systemctlstart rpcbind.service 

systemctlenable rpcbind.service

systemctlstart nfs.service 

systemctlenable nfs.service

 

# 3.问题和说明

①使用的kubernetes版本是1.13.0，只支持18.06的docker，不支持18.09

 

②安装私有镜像仓库（2.1步骤）报错

![img](file:///C:\Users\keven\AppData\Local\Temp\msohtmlclip1\01\clip_image002.png)

原因是修改iptables规则后，重启了iptables，但没有重启docker，重启就好了

 

③执行完kubeadminit之后，用kubectl get cs命令查看，报错

Error fromserver (Forbidden): componentstatuses is forbidden: User "system:node: k8s-master1"cannot list resource "componentstatuses" in API group "" atthe cluster scope

解决办法：

执行sudo su

 

④执行kubeadminit或者kubectl version时报错

The connectionto the server localhost:8080 was refused

解决办法：

sudo cp/etc/kubernetes/admin.conf $HOME/

sudo chown$(id -u):$(id -g) $HOME/admin.conf

exportKUBECONFIG=$HOME/admin.conf

 

⑤删除安装执行

kubectl drain
<node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>

kubeadm reset

其实只是重置了，kubeadm和kubectl使用命令是删不了的，而且执行完上面的命令后，还需要手动删除一些初始化文件，然后重启kubelet：

find/var/lib/kubelet | xargs -n 1 findmnt -n -t tmpfs -o TARGET -T | uniq | xargs-r umount -v;

rm -r -f/etc/kubernetes /var/lib/kubelet /var/lib/etcd; 

systemctlrestart kubelet

 

⑥查看kubelet状态

systemctlstatus kubelet -l

journalctl-xeu kubelet

 

⑦添加node节点时，忘记了当时初始化master时界面给的kubeadm join命令，可以在Kubernetes master上执行下面的命令重新获得：

kubeadm tokencreate --print-join-command

 

安装参考文档：

<https://www.kubernetes.org.cn/4948.html>

<https://www.kubernetes.org.cn/4956.html>
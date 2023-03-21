# k8s_deploy
记录个人学习k8s以及运维相关组件部署与配置的过程

#### 实验环境
网络
```
电信公网动态ipv4
局域网网段 192.168.50.0/24
```
虚拟化平台
```
ESXi7.0 u3
```
master节点
```
2c2g
release:Ubuntu 22.04.2 LTS
hostname:k8s01
ip:192.168.50.81
```
worker节点1
```
2c2g
release:Ubuntu 22.04.2 LTS
hostname:k8s02
ip:192.168.50.82
```
worker节点2
```
2c2g
release:Ubuntu 22.04.2 LTS
hostname:k8s03
ip:192.168.50.83
```

## k8s集群搭建
#### docker容器化环境安装
国内使用 daocloud 一键安装脚本
```
sudo curl -sSL https://get.daocloud.io/docker | sh
```
#### 修改各节点主机名
```
sudo hostnamectl set-hostname k8s0x
```
**由于各个节点由虚拟机克隆部署，需要注意网卡mac地址是否重复，ESXi在克隆虚拟机时已经重新生成过mac地址，因此不用操作**
#### 禁用SELinux
SELinux是linux内核安全组件，禁用SELinux是允许容器访问主机文件系统所必需的
官方文档解释

*Setting SELinux in permissive mode by runningsetenforce 0andsed ...effectively disables it. This is required to allow containers to access the host filesystem, which is needed by pod networks for example. You have to do this until SELinux support is improved in the kubelet.*

Ubuntu 提供 AppArmor 作为 SELinux 的替代品，保险起见禁用AppArmor
```
sudo systemctl disable apparmor
reboot
```
#### 关闭swap
当主机内存不足时，linux会自动使用swap，降低性能，和k8s的调度策略冲突，因此需要关闭
```
swapoff -a  
sed -ri 's/.*swap.*/#&/' /etc/fstab
```
#### 允许iptables检查桥接流量
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```
#### 保险起见禁用ufw防火墙（非生产环境）
```
sudo ufw disable
```
#### 切换docker下载源为国内镜像站，并修改cgroups
cgroups可以理解为一个进程隔离工具，docker默认使用cgroupfs，k8s使用systemd，为了防止出现异常，修改docker的cgroups为systemd
```
sudo cat << EOF > /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://reg-mirror.qiniu.com",
    "https://quay-mirror.qiniu.com"
  ],
  "exec-opts": [ "native.cgroupdriver=systemd" ]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
#### 安装k8s组件
```
# 使得 apt 支持 ssl 传输
apt-get update && apt-get install -y apt-transport-https
# 下载 gpg 密钥 这个需要root用户否则会报错
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
# 添加 k8s 镜像源 这个需要root用户否则会报错
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
# 更新源列表
apt-get update
# 下载 kubectl，kubeadm以及 kubelet
apt-get install -y kubelet kubeadm kubectl
```
#### 所有机器添加master域名映射
```
echo "192.168.50.81 cluster-endpoint" >> /etc/hosts
```
#### 初始化master节点
```
# 先拉取coredns的镜像否则下一步会失败
docker pull coredns/coredns:1.8.4
docker tag coredns/coredns:1.8.4 registry.aliyuncs.com/google_containers/coredns:v1.8.4
# 主节点初始化
kubeadm init \
--apiserver-advertise-address=192.168.50.81 \
--image-repository registry.aliyuncs.com/google_containers \
--control-plane-endpoint=cluster-endpoint \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=10.97.0.0/16
```
出现报错

*container runtime is not running*

搜索资料可知可能造成问题的是/etc/containerd/config.toml下
```
disabled_plugins = ["cri"]
```
解决方法为删除改配置文件，并重启containerd服务
```
rm -rf /etc/containerd/config.toml
systemctl restart containerd
```
重新执行初始化命令，得到初始化成功的提示后，及时保留相关信息，如
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join cluster-endpoint:6443 --token mew21x.bne69wci8fhtjoam \
	--discovery-token-ca-cert-hash sha256:c17660890b041b9cc61abcd68be6bc119cb414c139113535d58966e12b1ac1ac \
	--control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join cluster-endpoint:6443 --token mew21x.bne69wci8fhtjoam \
	--discovery-token-ca-cert-hash sha256:c17660890b041b9cc61abcd68be6bc119cb414c139113535d58966e12b1ac1ac
```
根据提示输入指令
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

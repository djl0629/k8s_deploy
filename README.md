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
2c4g
release:CentOS Linux release 7.9.2009 (Core)
hostname:k8s01
ip:192.168.50.81
```
worker节点1
```
2c2g
release:CentOS Linux release 7.9.2009 (Core)
hostname:k8s02
ip:192.168.50.82
```
worker节点2
```
2c2g
release:CentOS Linux release 7.9.2009 (Core)
hostname:k8s03
ip:192.168.50.83
```

## k8s集群搭建
#### docker容器化环境安装
国内使用 daocloud 一键安装脚本
```
curl -sSL https://get.daocloud.io/docker | sh
```
#### 集群环境搭建
修改各节点主机名
```
sudo hostnamectl set-hostname k8s0x
```
**由于各个节点由虚拟机克隆部署，需要注意网卡mac地址是否重复，ESXi在克隆虚拟机时已经重新生成过mac地址，因此不用操作**
禁用SELinux
SELinux是linux内核安全组件，禁用SELinux是允许容器访问主机文件系统所必需的
官方文档解释

*Setting SELinux in permissive mode by runningsetenforce 0andsed ...effectively disables it. This is required to allow containers to access the host filesystem, which is needed by pod networks for example. You have to do this until SELinux support is improved in the kubelet.*

将 SELinux 设置为 permissive 模式（相当于将其禁用）
```
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```
关闭swap
当主机内存不足时，linux会自动使用swap，降低性能，和k8s的调度策略冲突，因此需要关闭
```
swapoff -a  
sed -ri 's/.*swap.*/#&/' /etc/fstab
```
允许iptables检查桥接流量
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
禁用系统防火墙
```
systemctl disable firewalld
```
切换docker下载源为国内镜像站，并修改cgroups
cgroups可以理解为一个进程隔离工具，docker默认使用cgroupfs，k8s使用systemd，为了防止出现异常，修改docker的cgroups为systemd
```
sudo cat << EOF > /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://reg-mirror.qiniu.com",
    "https://quay-mirror.qiniu.com"
  ],
  "exec-opts": [ "native.cgroupdriver=systemd" ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```
安装k8s组件
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
   http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```
所有机器添加master域名映射
```
echo "192.168.50.81 cluster-endpoint" >> /etc/hosts
```
#### 初始化master节点
```
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
输入以下指令查看所有节点，主节点状态为NotReady，需要安装网络组件
```
kubectl get nodes
```
#### 安装网络组件
根据calico官网提示，节点数量小于50下载此配置文件
```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml -O
```
由于修改过pod CIDR，因此需要取消注释yaml文件下的CALICO_IPV4POOL_CIDR变量并修改值
```
- name: CALICO_IPV4POOL_CIDR
  value: "10.97.0.0/16"
```
应用配置文件
```
kubectl apply -f calico.yaml
```
安装完成后主节点状态变更为Ready,主节点初始化完成
```
NAME    STATUS   ROLES           AGE    VERSION
k8s01   Ready    control-plane   174m   v1.26.3
```
#### worker节点加入集群
在worker节点执行刚才保存的加入集群指令
```
kubeadm join cluster-endpoint:6443 --token mew21x.bne69wci8fhtjoam \
	--discovery-token-ca-cert-hash sha256:c17660890b041b9cc61abcd68be6bc119cb414c139113535d58966e12b1ac1ac
```
出现错误提示

*/proc/sys/net/ipv4/ip_forward contents are not set to 1*

修改配置文件以允许IP转发
```
echo 1 > /proc/sys/net/ipv4/ip_forward
```
重新输入加入集群指令，提示成功加入集群
**集群令牌24h有效**
重新生成集群令牌
```
kubeadm token create --print-join-command
```

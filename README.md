```bash
# 备份旧的yum.repos.d
mkdir -p /etc/yum.repos.d/bak
cd /etc/yum.repos.d
mv * bak

# 设置阿里云 centos yum源
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
# 设置阿里云 epel yum源
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
# 设置阿里云 docker yum源
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager -y --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 设置阿里云 kubernetes yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 设置主机名
hostnamectl set-hostname k8s-master01
echo '172.20.10.4 k8s-master01' >> /etc/hosts
echo '172.20.10.5 k8s-master02' >> /etc/hosts
echo '172.20.10.6 k8s-master03' >> /etc/hosts
cat /etc/hosts

# 备份旧的yum.repos.d
mkdir -p /etc/yum.repos.d/bak
cd /etc/yum.repos.d
mv * bak

# 设置阿里云 centos yum源
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
# 设置阿里云 epel yum源
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

# 设置不做gpgcheck
cd /etc/yum.repos.d/
find . -name "*.repo" -exec sed -i 's/gpgcheck=1/gpgcheck=0/g' {} \;

# 所有节点更新软件
yum -y update

# 所有节点设置elrepo
# 请一台一台主机执行
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
yum --enablerepo=elrepo-kernel install -y kernel-ml

# 所有节点设置启动选项并重启
# 请一台一台主机执行
grub2-mkconfig -o /boot/grub2/grub.cfg && grub2-set-default 0 && reboot

# 确认内核版本
uname -a

# 安装基础软件
yum install -y htop tree wget jq git net-tools ntpdate nc

# 更新时区
timedatectl set-timezone Asia/Shanghai && date && echo 'Asia/Shanghai' > /etc/timezone

# 所有节点上设置对journal进行持久化
sed -i 's/#Storage=auto/Storage=auto/g' /etc/systemd/journald.conf && mkdir -p /var/log/journal && systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald.service
ls -al /var/log/journal

# 所有节点上设置history显示时间戳
echo 'export HISTTIMEFORMAT="%Y-%m-%d %T "' >> ~/.bashrc && source ~/.bashrc

# 备份旧的yum.repos.d
mkdir -p /etc/yum.repos.d/bak
cd /etc/yum.repos.d
mv CentOS-* bak

# 设置阿里云 centos yum源
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
# 设置阿里云 epel yum源
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
# 设置阿里云 docker yum源
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager -y --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
# 设置阿里云 kubernetes yum源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 设置不做gpgcheck
cd /etc/yum.repos.d/
find . -name "*.repo" -exec sed -i 's/gpgcheck=1/gpgcheck=0/g' {} \;

# 安装docker
yum search docker-ce --showduplicates
yum search docker-compose --showduplicates
yum install docker-ce-20.10.3-3.el7.x86_64 docker-compose-1.18.0-4.el7.noarch && systemctl enable docker && systemctl start docker && systemctl status docker

# 设置docker镜像源
cat << EOF > /etc/docker/daemon.json
{
    "registry-mirrors": [
        "https://hub-mirror.c.163.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://f1361db2.m.daocloud.io",
        "https://registry.docker-cn.com"
    ]
}
EOF

# 重启docker
systemctl restart docker

```

```bash
# 所有master节点开放相关防火墙端口
firewall-cmd --zone=public --add-port=6443/tcp --permanent
firewall-cmd --zone=public --add-port=2379-2380/tcp --permanent
firewall-cmd --zone=public --add-port=10250/tcp --permanent
firewall-cmd --zone=public --add-port=10251/tcp --permanent
firewall-cmd --zone=public --add-port=10252/tcp --permanent
firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent
# 必须开启firewalld该设置，否则dns无法解释
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload
firewall-cmd --list-all --zone=public

# 所有worker节点开放相关防火墙端口
firewall-cmd --zone=public --add-port=10250/tcp --permanent
firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent
# 必须开启firewalld该设置，否则dns无法解释
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload
firewall-cmd --list-all --zone=public

# 所有节点清除iptables规则，解决firewalld引起nodeport无法访问问题
iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited

# 所有节点设置root的crontab，每十分钟设置一次
echo '5,15,25,35,45,55 * * * * /usr/sbin/iptables -D INPUT -j REJECT --reject-with icmp-host-prohibited' >> /var/spool/cron/root && crontab -l

# 在所有节点上设置selinux
sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config && (setenforce 0 || true) && getenforce

# 在第一个节点上设置sysctl
# kernel.pid_max = 99999 用于避免docker top出现异常
# vm.max_map_count = 262144 sonarqube设置要求
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
kernel.pid_max = 99999
vm.max_map_count = 262144
EOF

# 在第一个节点上把/etc/sysctl.d/k8s.conf复制到所有节点
scp /etc/sysctl.d/k8s.conf xxx:/etc/sysctl.d/

# 在所有节点设置启用sysctl
sysctl --system

# 在所有节点加载br_netfilter模块
modprobe br_netfilter

# 禁用swap
cat /proc/swaps
swapoff -a
cat /proc/swaps

# 删除/etc/fstab中的swap分区设置
sed -i '/swap/d' /etc/fstab 
cat /etc/fstab

# 在所有节点安装kubernetes
yum search kubeadm kubelet --showduplicates
yum install -y kubeadm-1.20.2-0.x86_64 kubelet-1.20.2-0.x86_64 kubectl-1.20.2-0.x86_64 && systemctl enable kubelet && systemctl start kubelet && systemctl status kubelet



# 拉取kubernetes相关镜像
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.20.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.20.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.20.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.20.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.4.13-0
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.7.0

docker pull quay.io/tigera/operator:v1.13.5

docker pull calico/kube-controllers:v3.16.5
docker pull calico/node:v3.16.5
docker pull calico/pod2daemon-flexvol:v3.16.5
docker pull calico/cni:v3.16.5
docker pull calico/typha:v3.16.5

# nginx-lb keepalived相关镜像
docker pull osixia/keepalived:2.0.20
docker pull nginx:1.20.2-alpine

# 安装helm
# 把helm复制到所有节点
scp /usr/bin/helm xxx:/usr/bin/

# 确认所有节点的网络接口名称
ip a | grep -B 2 '172.31'

# 在master01上使用helm生成安装配置文件
cd /root/helm-charts
mkdir -p output
# DEVOPS集群
helm template k8s-install --output-dir output -f k8s-install-info-devops.yaml
# UAT集群
helm template k8s-install --output-dir output -f k8s-install-info-prepro.yaml

cd output/k8s-install/templates/

# 在master01上自动启动所有master节点的keepalived和nginx-lb
sed -i '1,2d' create-config.sh
sh create-config.sh

# master01上初始化集群
kubeadm init --config=kubeadm-config.yaml --upload-certs

# DEVOPS集群
# 执行后输出如下内容:
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 172.31.250.25:16443 --token 082tzj.6huxd6j9ah0ymqb1 \
    --discovery-token-ca-cert-hash sha256:e5f2c49b44e8416dd7941108b3c69c101850d11d607db43d5e07f634bf2d6ef1 \
    --control-plane --certificate-key b32073278a0141589355074d539053a8acc159217651ee54b4c499e8b26f665a

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.250.25:16443 --token 082tzj.6huxd6j9ah0ymqb1 \
    --discovery-token-ca-cert-hash sha256:e5f2c49b44e8416dd7941108b3c69c101850d11d607db43d5e07f634bf2d6ef1 
exit

# UAT集群
# 执行后输出如下内容:
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 172.31.250.21:16443 --token rf4l0b.99zimtd5vmj3gele \
    --discovery-token-ca-cert-hash sha256:6e24c6b811e4963753cf1026666e8377e0672c27a512f2b650356340bab23786 \
    --control-plane --certificate-key effa228460925bc6ce4e9a38b5a101543019927290fea14a12309c0309ec9ccb

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.250.21:16443 --token rf4l0b.99zimtd5vmj3gele \
    --discovery-token-ca-cert-hash sha256:6e24c6b811e4963753cf1026666e8377e0672c27a512f2b650356340bab23786 
exit

# 所有master节点设置KUBECONFIG环境变量
cat <<EOF >> ~/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
EOF
source ~/.bashrc

# master01上安装calico网络
kubectl apply -f calico-v3.16.5/tigera-operator.yaml
sleep 1
kubectl apply -f calico-v3.16.5/custom-resources.yaml

# 在所有master节点进行join
  kubeadm join xxxx --token xxxx \
    --discovery-token-ca-cert-hash xxxx \
    --control-plane --certificate-key xxxx

# 等待所有节点的pods正常
kubectl get pods -A

# 所有master节点上设置kubectl自动完成
kubectl get pods
yum install -y bash-completion && mkdir -p ~/.kube/
kubectl completion bash > ~/.kube/completion.bash.inc
printf "
# Kubectl shell completion
source '$HOME/.kube/completion.bash.inc'
" >> $HOME/.bash_profile
source $HOME/.bash_profile

# 允许master部署pod
kubectl taint nodes --all node-role.kubernetes.io/master-

# 所有master节点上使用kubelet自动创建keepalived和nginx-lb的pod
mv /etc/kubernetes/keepalived/ /etc/kubernetes/manifests/
mv /etc/kubernetes/manifests/keepalived/keepalived.yaml /etc/kubernetes/manifests/
mv /etc/kubernetes/nginx-lb/ /etc/kubernetes/manifests/
mv /etc/kubernetes/manifests/nginx-lb/nginx-lb.yaml /etc/kubernetes/manifests/

tree /etc/kubernetes/manifests/

# 所有master节点检查master节点的keepalived和nginx-lb的pod已经自动创建后，再进行以下操作
kubectl get pods -n kube-system

# 所有master节点检查master节点的keepalived和nginx-lb的pod已经自动创建后，再进行以下操作
systemctl stop kubelet
docker rm -f keepalived nginx-lb
systemctl restart kubelet

# 检查服务状态
kubectl get pods -n kube-system

# 所有master节点重启所有服务
systemctl restart kubelet docker

# 所有master节点修改/etc/kubernetes/admin.conf
sed -i 's/:16443/:6443/g' /etc/kubernetes/admin.conf

# 在所有node节点上执行join
kubeadm join xxxx --token xxxx \
    --discovery-token-ca-cert-hash xxxx

# 创建测试pod
kubectl run busybox --image=busybox --command -- tail -f /dev/null

# DEVOPS集群
# 设置nodes的labels
kubectl label nodes devops-k8s-master01 type=master
kubectl label nodes devops-k8s-master02 type=master
kubectl label nodes devops-k8s-master03 type=master
kubectl label nodes devops-k8s-node01   type=worker
kubectl label nodes devops-k8s-node02   type=worker
kubectl label nodes devops-k8s-node03   type=worker
kubectl label nodes devops-k8s-node04   type=worker
kubectl label nodes devops-k8s-node05   type=worker
kubectl label nodes devops-k8s-node06   type=worker
kubectl label nodes devops-k8s-node07   type=worker
kubectl label nodes devops-k8s-node08   type=worker
kubectl label nodes devops-k8s-node09   type=worker
kubectl label nodes devops-k8s-node10   type=worker
kubectl label nodes devops-k8s-node11   type=worker
kubectl label nodes devops-k8s-node12   type=worker
kubectl label nodes devops-k8s-node13   type=worker

# UAT集群
# 设置nodes的labels
kubectl label nodes uat-k8s-master01 type=master
kubectl label nodes uat-k8s-master02 type=master
kubectl label nodes uat-k8s-master03 type=master
kubectl label nodes uat-k8s-node01   type=worker
kubectl label nodes uat-k8s-node02   type=worker
kubectl label nodes uat-k8s-node03   type=worker
kubectl label nodes uat-k8s-node04   type=worker
kubectl label nodes uat-k8s-node05   type=worker
kubectl label nodes uat-k8s-node06   type=worker
kubectl label nodes uat-k8s-node07   type=worker
kubectl label nodes uat-k8s-node08   type=worker
kubectl label nodes uat-k8s-node09   type=worker

```

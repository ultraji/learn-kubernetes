# 搭建kubernetes集群

* 实验环境
    * 用vmware创建的两台2u4g的centos8虚机（可以先创建一台虚机，中途克隆）

## 系统配置

1. 修改主机名（PS：我这里用master做主机名主要用于区分k8s中master和node）；

    ```shell
    vim /etc/hostname
    # 输入主机名
    master
    ```

2. 在hosts中将要加入集群的主机进行域名映射

    ```shell
    vim /etc/hosts
    # 增加以下内容：
    192.168.199.222 master
    192.168.199.223 node1
    ```

3. 永久关闭swap（主要出于性能考虑？）

    ```shell
    swapoff -a
    vim /etc/fstab
    # 注释掉SWAP分区项，即可
    #   /dev/mapper/cl_master-root /                       xfs     defaults        0 0
    #   UUID=28bf76b1-e134-4f8a-b44b-02395a3b888b /boot                   ext4    defaults        1 2
    #   # /dev/mapper/cl_master-swap swap                    swap    defaults        0 0  # 这行

    #刷新swap使之生效
    sysctl -p
    ```

4. 关闭selinux （出错了不容易排查？）

    ```shell
    setenforce 0
    ```

5. 永久关闭防火墙，或者不关闭防火墙但需要打开一些端口（**二选一**）

    ```shell
    systemctl disable firewalld
    systemctl stop firewalld
    ```

    或

    ```shell
    firewall-cmd --zone=public --add-port=80/tcp --permanent
    firewall-cmd --zone=public --add-port=6443/tcp --permanent
    firewall-cmd --zone=public --add-port=2379-2380/tcp --permanent
    firewall-cmd --zone=public --add-port=10250-10255/tcp --permanent
    firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent
    firewall-cmd --reload
    firewall-cmd --zone=public --list-ports
    ```

6. 配置流量转发

    ```shell
    cat <<EOF >  /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    vm.swappiness = 0
    EOF

    modprobe br_netfilter
    sysctl -p /etc/sysctl.d/k8s.conf
    sysctl --system
    systemctl daemon-reload
    systemctl restart kubelet
    ```

## 安装软件

1. 更新docker和k8s的软件源信息（PS：这里我选择了阿里源）

    ```shell
    yum -y install yum-utils

    # docker-ce 源
    yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

    # kubernetes 源
    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=0
    repo_gpgcheck=0
    gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
    EOF

    # 更新yum包的索引
    yum clean all && yum makecache fast
    yum -y update
    ```

2. 安装docker，以及配置镜像加速

    ```shell
    # 卸载原有docker
    systemctl stop docker && systemctl disable docker
    yum remove docker-ce docker-ce-cli

    # 安装k8s支持的docker版本，我这里选择了`18.09.0`
    yum list docker-ce --showduplicates|sort -r
    yum install -y docker-ce-18.09.0-3.el7 docker-ce-cli-18.09.0-3.el7
    systemctl start docker && systemctl enable docker

    # 配置docker
    touch /etc/docker/daemon.json
    vim /etc/docker/daemon.json
    {
        "registry-mirrors": ["https://xxxxx.mirror.aliyuncs.com"] # 可以选用各大云厂商的容器镜像加速服务
    }
    systemctl daemon-reload
    systemctl restart docker
    ```

3. 安装kubeadm，安装kubeadm的命令同时会安装下面的3个二进制

    ```shell
    # kubeadm k8s的配置工具
    # kubelet k8s的核心服务
    # kubectl kubelet的client
    # kubernetes-cni 容器网络接口标准协议
    yum install -y kubeadm
    systemctl daemon-reload
    systemctl start kubelet && systemctl enable kubelet
    ```

4. **自此，可以克隆多份虚拟机做工作节点了，记得改对应的hostname和静态IP。**

## 搭建master节点

1. 生成默认的配置文件

    ```shell
    kubeadm config print init-defaults > kubeadm.conf
    # 修改localAPIEndpoint
    # vim kubeadm.conf
    localAPIEndpoint:
        advertiseAddress: 192.168.199.222 # 改成Master IP
    ```

2. 由于国内网络环境的缘故，事先把`kubeadm init`需要的镜像拉下来

    ```shell
    kubeadm config images list --config kubeadm.conf
    # 根据输出修改脚本
    vim pullkubeimages.sh
    ```
    内容如下：

    ```shell
    #!/bin/bash

    # 国内地址
    # registry.cn-hangzhou.aliyuncs.com/google_containers
    # registry.cn-beijing.aliyuncs.com/escience/escience-beijing

    KUBE_VERSION=v1.19.0        # 根据上条命令的输出修改版本
    KUBE_PAUSE_VERSION=3.2
    ETCD_VERSION=3.4.13-0
    CORE_DNS_VERSION=1.7.0

    GCR_URL=k8s.gcr.io
    ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

    images=(
            kube-apiserver:${KUBE_VERSION}
            kube-controller-manager:${KUBE_VERSION}
            kube-scheduler:${KUBE_VERSION}
            kube-proxy:${KUBE_VERSION}
            pause:${KUBE_PAUSE_VERSION}
            etcd:${ETCD_VERSION}
            coredns:${CORE_DNS_VERSION}
    )

    for imageName in ${images[@]} ; do
    docker pull $ALIYUN_URL/$imageName
    docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
    docker rmi  $ALIYUN_URL/$imageName
    done
    ```

3. 初始化master节点

    ```shell
    kubeadm init --config ./kubeadm.conf
    ```

    记录输出

    ```
    W1118 22:44:02.998868   10181 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
    [init] Using Kubernetes version: v1.19.0
    [preflight] Running pre-flight checks
            [WARNING Firewalld]: firewalld is active, please ensure ports [6443 10250] are open or your cluster may not function correctly
            [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
            [WARNING FileExisting-tc]: tc not found in system path
    [preflight] Pulling images required for setting up a Kubernetes cluster
    [preflight] This might take a minute or two, depending on the speed of your internet connection
    [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
    [certs] Using certificateDir folder "/etc/kubernetes/pki"
    [certs] Generating "ca" certificate and key
    [certs] Generating "apiserver" certificate and key
    [certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master] and IPs [10.96.0.1 192.168.199.222]
    [certs] Generating "apiserver-kubelet-client" certificate and key
    [certs] Generating "front-proxy-ca" certificate and key
    [certs] Generating "front-proxy-client" certificate and key
    [certs] Generating "etcd/ca" certificate and key
    [certs] Generating "etcd/server" certificate and key
    [certs] etcd/server serving cert is signed for DNS names [localhost master] and IPs [192.168.199.222 127.0.0.1 ::1]
    [certs] Generating "etcd/peer" certificate and key
    [certs] etcd/peer serving cert is signed for DNS names [localhost master] and IPs [192.168.199.222 127.0.0.1 ::1]
    [certs] Generating "etcd/healthcheck-client" certificate and key
    [certs] Generating "apiserver-etcd-client" certificate and key
    [certs] Generating "sa" key and public key
    [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
    [kubeconfig] Writing "admin.conf" kubeconfig file
    [kubeconfig] Writing "kubelet.conf" kubeconfig file
    [kubeconfig] Writing "controller-manager.conf" kubeconfig file
    [kubeconfig] Writing "scheduler.conf" kubeconfig file
    [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
    [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
    [kubelet-start] Starting the kubelet
    [control-plane] Using manifest folder "/etc/kubernetes/manifests"
    [control-plane] Creating static Pod manifest for "kube-apiserver"
    [control-plane] Creating static Pod manifest for "kube-controller-manager"
    [control-plane] Creating static Pod manifest for "kube-scheduler"
    [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
    [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
    [apiclient] All control plane components are healthy after 18.506231 seconds
    [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
    [kubelet] Creating a ConfigMap "kubelet-config-1.19" in namespace kube-system with the configuration for the kubelets in the cluster
    [upload-certs] Skipping phase. Please see --upload-certs
    [mark-control-plane] Marking the node master as control-plane by adding the label "node-role.kubernetes.io/master=''"
    [mark-control-plane] Marking the node master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
    [bootstrap-token] Using token: abcdef.0123456789abcdef
    [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
    [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
    [bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
    [bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
    [bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
    [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
    [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
    [addons] Applied essential addon: CoreDNS
    [addons] Applied essential addon: kube-proxy

    Your Kubernetes control-plane has initialized successfully!

    To start using your cluster, you need to run the following as a regular user:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    https://kubernetes.io/docs/concepts/cluster-administration/addons/

    Then you can join any number of worker nodes by running the following on each as root:

    kubeadm join 192.168.199.222:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:105f7cec3de32bd6b5061d5205b1b439d5a96f9db205e4db001c3d0136f57181
    ```

    在工作节点执行上面这条命令就可以加入集群。


4. 执行集群的基本配置

    ```shell
    mkdir -p $HOME/.kube
    cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    chown $(id -u):$(id -g) $HOME/.kube/config
    ```

5. 验证集群安装情况

```shell
kubectl get nodes
# NAME     STATUS   ROLES    AGE   VERSION
# master   Ready    master   19m   v1.19.4

kubectl get cs
# Warning: v1 ComponentStatus is deprecated in v1.19+
# NAME                 STATUS      MESSAGE ERROR
# controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused
# scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused
# etcd-0               Healthy     {"health":"true"}
```

## 配置工作节点

1. 利用上面配置master输出结果中的内容，将node节点加入集群

    ```shell
    kubeadm join 192.168.199.222:6443 --token abcdef.0123456789abcdef \
            --discovery-token-ca-cert-hash sha256:105f7cec3de32bd6b5061d5205b1b439d5a96f9db205e4db001c3d0136f57181
    ```


## 排错

### 上面```kubectl get cs```中`controller-manager`、`scheduler`的状态均为`Unhealthy`

出现这种情况，是/etc/kubernetes/manifests/下的kube-controller-manager.yaml和kube-scheduler.yaml设置的默认端口是0导致的，解决方式是注释掉对应的port即可（`- --port=0`改为`#- --port=0`），操作如下：

```shell
cd /etc/kubernetes/manifests
vim kube-controller-manager.yaml
vim kube-scheduler.yaml
systemctl restart kubelet
kubectl get cs
```
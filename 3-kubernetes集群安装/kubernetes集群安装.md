# 环境设置

1. 关闭防火墙

    ```
    systemctl stop firewalld & systemctl disable firewalld
    ```

2. 同步时间

    ```
    ntpdate ntp1.aliyun.com
    hwclock
    timedatectl set-timezone Asia/Shanghai
    timedatectl set-local-rtc 0
    systemctl restart rsyslog
    systemctl restart crond.service
    date
    ```

3. 关闭SELINUX

    ```
    setenforce 0
    sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
    ```

4. 关闭Swap

    ```
    swapoff –a
    sed -i '/ swap / s/^/#/' /etc/fstab
    或者 vim /etc/fstab
    注释掉有swap的一行

    重启 top
    查看swap全为0即可
    ```

5. 配置yum源

    ```
    yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
    yum makecache
    ```

6. 配置国内Kubernetes源

    ```
    cat <<EOF > /etc/yum.repos.d/kubernetes.repo
        [kubernetes]
        name=Kubernetes
        baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
        enabled=1
        gpgcheck=1
        repo_gpgcheck=1
        gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg  
    EOF
    yum clean all && yum makecache
    ```

7. yum安装常用软件

    ```
    yum install  -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp bash-completion yum-utils device-mapper-persistent-data lvm2 net-tools conntrack-tools vim libtool-ltdl
    ```

# 安装docker

1. 安装Docker CE

    ```
    yum install docker-ce -y

    docker --version
    ```

2. 启动docker并设置开机自启

    ```
    systemctl start docker && systemctl enable docker
    ```

3. 若要配置daemon

    vim /lib/systemd/system/docker.service

    ```
    ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --insecure-registry=192.168.159.200 后边添加自己想设置的dockerd --help查看详细信息
    ```

    ```
    systemctl daemon-reload
    systemctl restart docker
    起不起来的话查看状态:  systemctl status docker.service
    ```

4. 检测docker

    ```
    docker run hello-world 其实网不通的话是没法验证的 最好还是docker images
    ```

# k8s安装

* 安装kubeadm、kubelet、kubectl(在所有节点上执行)

    *  kubeadm: 部署集群用的命令
    *  kubelet: 在集群中每台机器上都要运行的组件，负责管理pod、容器的生命周期
    *  kubectl: 集群管理工具  

    ```
    yum install -y kubelet kubeadm kubectl
    ```

    启动kubelet

    ```
    systemctl enable kubelet && systemctl start kubelet
    ```

* 下载kubernetes的镜像

    新建文件load_images.sh:

    ```
    #! /bin/bash
    kubeadm config images list > k8s-list.txt
    google=k8s.gcr.io
    aliyun=registry.cn-hangzhou.aliyuncs.com/google_containers

    sed -i "s/k8s.gcr.io\///g" k8s-list.txt
    cat k8s-list.txt
    for i in $(cat k8s-list.txt);
    do
        docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$i
        docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$i k8s.gcr.io/$i
        docker rmi $aliyun/$i
    done
    ```

    执行脚本并查看docker镜像：

    ```
    sh load_images.sh
    docker images
    ```

* 构建k8s集群

    1. master节点初始化Kubernetes Master，这里我们定义POD的网段为: 192.168.0.0/16，API Server地址为Master节点的IP地址

    ```
    kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=v1.16.3 --apiserver-advertise-address=192.168.159.100
    ```

    > 若报错，需按照报错信息执行相关操作：echo "1" >/proc/sys/net/bridge/bridge-nf-call-iptables

    并且记录下kubeadm生成的token

    2. 按照提示执行以下命令配置kubectl，作为普通用户管理集群并在集群上工作

    ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

    3. 获取pods列表，查看相关状态，可以看到coredns pod处于挂起状态，这是因为还没有部署pod网络。
    
    ```
    kubectl get pods --all-namespaces
    ```

    4. 部署Pod网络(在Master节点上执行)

    > 您必须安装pod网络附加组件，以便pod能够彼此通信。网络必须在任何应用程序之前部署。另外，CoreDNS在安装网络之前不会启动。kubeadm只支持基于容器网络接(CNI)的网络。如下图支持的Pod网络有JuniperContrail/TungstenFabric、Calico、Canal、Cilium、Flannel、Kube-router、Romana、Wave Net等。这里我们选择部署Calico网络，Calico是一个纯三层的方案，其好处是它整合了各种云原生平(Docker、Mesos 与 OpenStack 等)，每个 Kubernetes 节点上通过 Linux Kernel 现有的L3 forwarding 功能来实现 vRouter 功能。

    * 安装calio网络插件

    ```
    curl https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml -O
    kubectl apply -f rbac-kdd.yaml 
    curl https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubernetes-datastore/policy-only/1.7/calico.yaml -O
    修改对应的calio.yaml文件内容
    kubectl apply -f calico.yaml

    如果能科学上网，则会自动拉取镜像
    否则需要手动导入calio的相关镜像，注意版本的对应一致----详见网盘
    ```

    5. 获取master节点的kubeadm生成的token

    若上述kubeadm init生成的token没有记住，可以重新生成

    ```
    kubeadm token create --ttl 0 --print-join-command

    #列出token
    kubeadm token list  | awk -F" " '{print $1}' |tail -n 1
    #获取CA公钥的哈希值
    openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed  's/^ .* //'
    ```

    6. 从节点node 集群配置

    同master节点，node节点安装kubeadm、kubelet、kubectl, calio网络插件

    ```
    kubeadm join 192.168.159.100:6443 --token fspqyj.qq45byspdqptwc4j     --discovery-token-ca-cert-hash sha256:a27c00ab07b97a1114ba6f7eecd1a4565e2d81094d7fada0398337356ad021c0
    ```

    7. 验证

    ```
    kubectl get nodes
    ```

    


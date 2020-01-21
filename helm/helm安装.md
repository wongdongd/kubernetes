## 1. 是什么

> Helm is the best way to find, share, and use software built for Kubernetes

Helm可以帮助我们管理k8s应用，甚至可以define、install、upgrade最复杂的k8s应用。类比于linux中的yum

## 2. why helm

* helm的charts可以描述复杂的应用，能提供可复用的程序安装，且服务是single point of authority的
* 更新方便，可以原地升级，并且custom hook
* Charts的维护比较方便--版本迭代，share
* 回滚方便

## 3. helm安装(不涉及security)

> 主要包含两部分，一个是helm client(helm),一个是helm server(tiller)

1. helm client安装

* 二进制安装

    * 下载地址：https://github.com/helm/helm/releases
    * 解压：tar -zxvf helm-v2.0.0-linux-amd64.tgz
    * 再解压目录下找到helm二进制文件，然后移到可执行位置 mv linux-amd64/helm /usr/local/bin/helm

* 脚本安装

    ```
    curl -LO https://git.io/get_helm.sh
    chmod 700 get_helm.sh
    ./get_helm.sh
    ---------------
    或者直接管道命令：
    curl -L https://git.io/get_helm.sh | bash
    ```

2. tiller安装

    > tiller是helm的服务端部分，通常运行在k8s集群内部，但开发时可以本地运行，连接远程k8s集群

    * helm init [option][参数]

      * --tiller-image: 指定tiller镜像
      * --kube-context: 指定k8s集群
      * --tiller-namespace: 指定命名空间
      * --service-account: 对于支持RBAC的集群，指定service account
      * --node-selectors: 指定安装的符合label的node节点
      * --override: 重写deployment里面的任意值
      * --output: helm init --output json/yaml将不会安装tiller，生成相应格式的文件，然后可以通过这个文件使用kubectl安装

    helm init可以直接将tiller安装到k8s集群，首先会验证helm环境，然后会直接连到kubectl连接的默认集群(kubectl config view),连接成功后，安装到kube-system命名空间下

    ```
    可使用阿里云的镜像
    helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.10.0
    ```
    * 安装成功验证
     
        ```
        helm version
        kubectl get pods -n kube-system
        ```
    * 卸载
        ```
        kubectl delete deployment tiller-deploy --namespace kube-system
        ---------
        helm reset
        ```





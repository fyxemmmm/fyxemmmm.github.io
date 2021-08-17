---
title: Kubernetes高可用集群搭建
date: 2021-08-16 22:23:44
author: yuxuan
img: https://image.fyxemmmm.cn/blog/images/img6.jpeg
top: false
hide: false
cover: false
coverImg: 
toc: false
mathjax: false
summary: 基于kubeadm, 快速搭建k8s集群
categories: golang
tags:
  - golang
  - kubernetes
---
# k8s 高可用集群搭建

`docker的安装自行搞定即可, 尽量不要用太高版本到version.19即可`

**本文基于centos的操作系统，kubeadm来作为搭建方式**

 1. 新建yum源

    ```shell
    sudo cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
    https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
    EOF
    
    sudo yum makecache
    sudo yum -y install kubelet-1.20.2  kubeadm-1.20.2  kubectl-1.20.2
    sudo systemctl enable kubelet
    ```

2. 使用systemd作为docker的cgroup driver

   ```shell
   sudo vi  /etc/docker/daemon.json   （没有则创建）
   加入
   {
     "exec-opts": ["native.cgroupdriver=systemd"]
   }
   
   systemctl daemon-reload  && systemctl restart docker
   ```

3. 切换到root用户执行 **关键步骤**

   ```shell
   # （如果是重置机器，需要执行）
   kubeadm reset
   rm /etc/cni/net.d -fr
   iptables -F 
   yum -y remove kubelet-1.18.6  kubeadm-1.18.6  kubectl-1.18.6
   # （重置end）
   
   yum -y install kubelet-1.20.2  kubeadm-1.20.2  kubectl-1.20.2
   
   echo 1 > /proc/sys/net/ipv4/ip_forward
   modprobe br_netfilter
   echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
   
   systemctl daemon-reload  # 可能会报错 可以不执行这个
   systemctl enable kubelet
   ```

4. master节点执行以下命令

   ```shell
   #主机执行：
   kubeadm init --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers  --kubernetes-version=1.20.2 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.96.0.0/12  
   
   ```

5. 得到token之后给worker依次执行， 加入到集群当中去， `初步就完成了集群的搭建` **还差一个网络插件**

   ```shell
   #得到token后，给另外worker去执行
   kubeadm join 192.168.0.191:6443 --token zx5rj1.19yqkv7q2uehatit \
   --discovery-token-ca-cert-hash sha256:b5a066c56e73896dc14530d5464eadd45732de6bd3806e878c80ed589e4ea502
   ```

 6. 给节点做一些完善工作

    ```shell
    #然后退出到普通用户, 用kubectl命令执行
    #去除主节点污点
    kubectl taint nodes --all node-role.kubernetes.io/master-   # (后面一个 – 是需要的)
    
    给工作节点打标签
    kubectl label node huawei-worker  node-role.kubernetes.io/node=node
    ```

 7. 我们需要让外网也可以访问， **也就是通过kubectl客户端工具能连接主机的外部ip地址，需要做此工作**

    ```shell
    #让外网可以访问
    #先删除 apiserver的证书和key
    #主节点上
    cd /etc/kubernetes/pki && rm -f apiserver.key && rm -f  apiserver.crt
    sudo kubeadm init phase certs apiserver   --apiserver-cert-extra-sans 121.36.226.197
    kubeadm alpha certs renew apiserver
    ```

 8. 安装网络插件， **这里我们选择flannel**

    ```shell
    https://github.com/flannel-io/flannel
    For Kubernetes v1.17+ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    # apply里面的镜像要替换下，可以用katacoda
    ```



### 大功告成啦~   可以愉快的玩耍了！


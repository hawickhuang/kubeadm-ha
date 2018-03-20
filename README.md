# kubeadm-ha

## 安装docker

1. 若是国内安装，可选择代理:
    ```bash
    export http_proxy="http://ip:port" && export https_proxy="https://ip:port" 
    ```

2. 更新源：
    ```bash
    apt-get update
    ```

3. 使用docker的脚本方式快速安装：
    ```bash
    curl -fsSL get.docker.com | sh
    ```

4. 安装特定版本docker-ce，先查看可安装版本
    ```bash
    apt-cache madison docker-ce
    ```

5. 然后安装指定版本
    ```bash
    apt-get install docker-ce=17.12.1~ce-0~ubuntu(可替换为其它版本号)
    ```

6. docker version:
    ```bash
    Client:
    Version:	18.02.0-ce
    API version:	1.36
    Go version:	go1.9.3
    Git commit:	fc4de44
    Built:	Wed Feb  7 21:16:33 2018
    OS/Arch:	linux/amd64
    Experimental:	false
    Orchestrator:	swarm

    Server:
    Engine:
      Version:	18.02.0-ce
      API version:	1.36 (minimum version 1.12)
      Go version:	go1.9.3
      Git commit:	fc4de44
      Built:	Wed Feb  7 21:15:05 2018
      OS/Arch:	linux/amd64
      Experimental:	false
    ```

## 配置docker proxy

1. 创建docker.service.d目录
    ```shell
    mkdir -p /etc/systemd/system/docker.service.d
    ```

2. 创建 /etc/systemd/system/docker.service.d/http-proxy.conf文件，添加以下内容：
    ```conf
    [Service]
    Environment="HTTPS_PROXY=https://10.0.2.103:6128/" "HTTP_PROXY=http://10.0.2.103:6128/" "NO_PROXY=localhost,127.0.0.1"
    ```

3. 刷新配置和重启docker
    ```shell
    systemctl daemon-reload && service docker restart
    ```

## 安装工具组件

1. 安装依赖：
    ```bash
    apt-get update && apt-get install -y apt-transport-https
    ```

2. 添加k8s的gpg key：
    ```bash
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - 
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list  
    deb http://apt.kubernetes.io/ kubernetes-xenial main 
    EOF
    ```

3. 安装kubelet,kubeadm,kebectl（安装特定版本可参照docker的安装）：
    ```bash
    apt-get update && apt-get install -y kubelet kubeadm kubectl
    ```
    安装版本<1.9时，若出现以下报错
    ```bash
    kubelet : Depends: kubernetes-cni (= 0.5.1) but 0.6.0-00 is to be installed
    ```

    执行这个命令可解决
    ```bash
    aptitude install kubernetes-cni=0.5.1-00
    ```

4. 各组件版本：
    ```bash
    # kubeadm --version
    kubeadm version: &version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.3", GitCommit:"d2835416544f298c919e2ead3be3d0864b52323b", GitTreeState:"clean", BuildDate:"2018-02-07T11:55:20Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
    ```
    ```bash
    # kubelet --version
    Kubernetes v1.9.3
    ```
    ```bash
    # kubectl version
    Client Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.3", GitCommit:"d2835416544f298c919e2ead3be3d0864b52323b", GitTreeState:"clean", BuildDate:"2018-02-07T12:22:21Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
    ```

## 配置文件初始化
1. 在所有master节点上拉取代码，并进入代码目录
    ```bash
        git clone https://github.com/hawickhuang/kubeadm-ha.git
        cd kubeadm-ha
    ```

2. 在所有master节点上设置初始化脚本initial.sh，参照注释设置即可

3. 在所有master节点上运行初始化脚本，创建对应的配置文件
    ```bash
    ./initial.sh
    ```

## etcd集群部署

1. 在所有master上重置集群
    ```bash
    kubeadm reset  # 重置k8s集群
    rm -rf /var/lib/etcd-cluster  # 清空etcd数据
    ```

2. 重启etcd集群
    ```bash
        docker-compose --file etcd/docker-compose.yaml stop
        docker-compose --file etcd/docker-compose.yaml rm -f
        docker-compose --file etcd/docker-compose.yaml up -d
    ```

3. 验证etcd集群状态
    ```bash
        docker exec -ti etcd etcdctl cluster-health
        docker exec -ti etcd etcdctl cluster-health
    ```

## 初始化第一台master

1. 重置master网络
    ```bash
    systemctl stop kubele
    systemctl stop kubele
    systemctl stop kubele
    systemctl stop kubele
    rm -rf /etc/cni/
    ```

2. 删除遗留的网络
    ```bash
    ip a | grep -E 'docker|flannel|cni'
    ip a | grep -E 'docker|flannel|cni'
    ip link del flannel.1
    ip link del flannel.1

    # 重启docker和kubelet，检查网络情况
    systemctl restart docker && systemctl restart kubelet
    ip a | grep -E 'docker|flannel|cni'
    ```

3. 初始化master01机器，记录下输出的 “kubeadm join” 信息
    ```bash
    kubeadm init --config=kubeadm-init.yaml
    ```

4. 在所有的master节点上设置kubectl客户端连接
    ```bash
    vi ~/.bashrc
        export KUBECONFIG=/etc/kubernetes/admin.conf

    source ~/.bashrc
    ```

## 安装基础组件

1. 安装网络组件flannel，当没有网络组件时，节点状态时不正常的
    ```bash
    kubectl get nodes

    kubectl apply -f kube-flannel/

    # 等待pods均正常
    kubectl get pods --all-namespaces -o wide
    ```
    
2. 安装dashboard组件
    ```bash
    # 设置master节点为schedulable
    kubectl taint nodes --all node-role.kubernetes.io/master-

    kubectl apply -f kube-dashboard/
    ```
    打开浏览器，输入dashboard地址

    获取token，把token粘贴到页面，即可进入dashboard
    ```bash
    kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
    ```

## master高可用配置

1. 在master01上复制目录/etc/kubernetes/pki到master02和master03上：
    ```bash
    scp -r /etc/kubernetes/pki devops-master02:/etc/kubernetes/
    scp -r /etc/kubernetes/pki devops-master03:/etc/kubernetes/
    ```

2. 在其它master节点上初始化
    ```bash
    kubeadm init --config=kubeadm-init.yaml
    ```

3. worker节点加入集群
    ```bash
    kubeadm join --token token cluster_ip:6443 --discovery-token-unsafe-skip-ca-verification
    ```
4. 设置所有master的scheduable
    ```bash
    kubectl taint nodes --all node-role.kubernetes.io/master-
    ```

5. 基础组件多节点配置(示例为dns组件)
    ```bash
    kubectl scale --replicas=2 -n kube-system deployment/kube-dns
    kubectl get pods --all-namespaces -o wide| grep kube-dns
    ```

## 永久存储配置
1. 获取ceph镜像
    ```bash
    docker pull ceph/daemon
    ```

2. 安装mon第一个节点
    ```bash
    docker run -d --net=host --name=mon -v /etc/ceph:/etc/ceph -v /var/lib/ceph:/var/lib/ceph -e MON_IP=local_ip -e CEPH_PUBLIC_NETWORK=local_subnet ceph/daemon mon
    ```

3. 将/etc/ceph和/var/lib/ceph目录拷贝到第二及第三节点
    ```bash
    scp -r /etc/ceph ceph-mon2:/etc/
    scp -r /var/lib/ceph ceph-mon2:/var/lib/

    scp -r /etc/ceph ceph-mon3:/etc/
    scp -r /var/lib/ceph ceph-mon3:/var/lib/
    ```

4. 安装二、三节点
    ```bash
    docker run -d --net=host --name=mon -v /etc/ceph:/etc/ceph -v /var/lib/ceph:/var/lib/ceph -e MON_IP=local_ip -e CEPH_PUBLIC_NETWORK=local_subnet ceph/daemon mon
    ```

5. osd节点安装
    ```bash
    docker run -d --net=host --name=osd --privileged=true --pid=host -v /etc/ceph/:/etc/ceph/ -v /var/lib/ceph/:/var/lib/ceph/ -v /dev:/dev -e OSD_FORCE_ZAP=1 -e OSD_DEVICE=/dev/xvdf ceph/daemon osd
    ```

6. rgw节点安装
    ```bash
    docker run -d -p 80:80 --net=host  --name=rgw -v /etc/ceph/:/etc/ceph/ -v /var/lib/ceph/:/var/lib/ceph/ ceph/daemon rgw
    ```

7. mds节点安装
    ```bash
    docker run -d --net=host --name=mds -v /etc/ceph/:/etc/ceph/ -v /var/lib/ceph/:/var/lib/ceph/ -e CEPHFS_CREATE=1 ceph/daemon mds
    ```

8. mgr节点安装
    ```bash
    docker run -d --net=host --name=mgr -v /etc/ceph/:/etc/ceph/ -v /var/lib/ceph/:/var/lib/ceph/ ceph/daemon mgr
    ```

9. 启用mgr的dashboard功能
    ```bash
    docker exec mon ceph mgr dump
    docker exec mgr ceph mgr module enable dashboard
    ```

## k8s挂载cephfs

1. 更改kube-ceph下ceph-secret文件，将key的值替换为你的ceph集群的secret串（base64格式化后）

2. 更改kube-ceph下ceph-pv.yaml和ceph-pvc.yaml文件，将storage的值替换为你的osd的存储总值

3. 创建ceph挂载
    ```bash
    kubeadm apply -f kube-ceph/
    ```
    
 ## 参考文档
 
[基于kubeadm的kubernetes高可用集群部署](https://github.com/cookeem/kubeadm-ha/blob/master/README_CN.md#kubernetes%E7%9B%B8%E5%85%B3%E6%9C%8D%E5%8A%A1%E5%AE%89%E8%A3%85)

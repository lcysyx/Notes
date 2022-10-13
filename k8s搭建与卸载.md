### 0. 版本说明

- Docker：建议19.03（[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.12. Latest validated version: 19.03）
- Kubernetes 1.20.6
- Istio 1.9.5
- Knative 0.24.0
- Jaeger 1.20
- Python 3.7.9
- TensorFlow 1.15.5
- NumPy 1.18.5
- Six 1.16.0

### 1. 修改Docker的Cgroup Driver

Docker 默认的Cgroup Driver为cgroupfs，但是在Kubernetes 1.14之后的版本推荐使用systemd。对于18.x.x以上版本的Docker来说修改比较简单，只需要修改/etc/docker/daemon.json的配置即可。

Docker当前的Cgroup Driver可以通过docker info命令查看（第21行）：

```shell
csri@vm-ubuntu:~$ docker info
Client:
 Context:    default
 Debug Mode: false
 Plugins:
  app: Docker App (Docker Inc., v0.9.1-beta3)
  buildx: Build with BuildKit (Docker Inc., v0.5.0-docker)

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 3
 Server Version: 20.10.1
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 1
...
```

如果不修改配置，会在kubeadm init时提示：

```shell
[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. 
The recommended driver is "systemd". 
Please follow the guide at https://kubernetes.io/docs/setup/cri/
```

通过编辑Docker的配置文件修改Cgroup Driver

```shell
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "registry-mirrors": ["https://fokbqzse.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

重启Docker

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 2. 禁用swap交换分区

为了保证kubelet正常工作，**必须**禁用交换分区。

首先通过swapoff命令禁用swap，-a选项表示禁用/proc/swaps中的所有交换区

```shell
sudo swapoff -a
```

通过free命令查看是否关闭成功

```shell
free -h
              总计         已用        空闲      共享    缓冲/缓存    可用
内存：       3.8Gi       932Mi       2.2Gi        16Mi       768Mi       2.7Gi
交换：          0B          0B          0B
```

为了保证重启后swap依然禁用，需要编辑/etc/fstab文件注释掉swap相关内容

```shell
sudo vim /etc/fstab
```

例如注释掉该配置文件的最后一行：

```shell
# /etc/fstab: static file system information.
# 
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda5 during installation
UUID=0145ef20-4b63-4fc6-a77d-607f5a3bbb33 /               ext4    errors=remount-ro 0       1
# /boot/efi was on /dev/sda1 during installation
UUID=3A74-A912  /boot/efi       vfat    umask=0077      0       1
#/swapfile                                 none            swap    sw              0       0
```

如果是使用的swapfile，删除swapfile

```shell
sudo rm -f /swapfile
```

### 3. Letting iptables see bridged traffic

首先确保系统加载了br_netfilter模块，可以通过运行lsmod | grep br_netfilter来查看。要显式地加载该模块，可以调用sudo modprobe br_netfilter。

为了让Linux Node的iptables能正确地看到桥接流量，需要确保sysctl配置中的net.bridge.bridge-nf-call-iptables被设为1。

```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

### 4. 安装kubeadm、kubelet和kubectl

更新 apt 包索引并安装使用 Kubernetes apt 仓库所需要的包：

```shell
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

下载 Google Cloud 公开签名秘钥：

```shell
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

或者从阿里源下载签名秘钥：

```shell
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg
```

添加 Kubernetes apt 仓库：

```shell
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

或者添加阿里源的Kubernetes仓库

```shell
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

更新 apt 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：

```shell
sudo apt-get update
sudo apt-get install -y kubelet=1.20.6-00 kubeadm=1.20.6-00 kubectl=1.20.6-00
sudo apt-mark hold kubelet kubeadm kubectl
```

kubelet 现在每隔几秒就会重启，因为它陷入了一个等待 kubeadm 指令的死循环

### 5. 初始化Cluster

#### 5.1 官方镜像

使用kubeadm config预先拉取镜像（可选）

```shell
kubeadm config images pull --kubernetes-version 1.20.6
```

初始化cluster，其中--pod-network-cidr用于指定pod的子网范围，不能与网络中已有子网重复，且需要和k8s的网络插件配置中的子网一致（flannel默认配置为10.244.0.0/16）

**初始化前记得关闭代理**

```shell
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --kubernetes-version 1.20.6
```

安装成功

```shell
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

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.20.128:6443 --token r0v987.gqly73709wv15fkv \
    --discovery-token-ca-cert-hash sha256:32abaa9719b505219772174a76c3bb15e5375a3e998ddc27fe3f7fecd8cd0ed7
```

**注意记录kubeadm init输出的kubeadm join命令，后续需要此命令将其他节点加入集群。**

#### 5.2 阿里云镜像方法一

先从阿里云拉取镜像（可选）

```shell
kubeadm config images pull \
    --image-repository registry.aliyuncs.com/google_containers --kubernetes-version 1.20.6
```

再初始化Cluster

```shell
sudo kubeadm init \
    --pod-network-cidr 10.244.0.0/16 \
    --image-repository registry.aliyuncs.com/google_containers --kubernetes-version 1.20.6
```

#### 5.3 阿里云镜像方法二

由于官方镜像地址被墙，所以我们需要首先获取所需镜像以及它们的版本。然后从国内镜像站获取。

```shell
kubeadm config images list
```

获取镜像列表后可以通过下面的脚本从阿里云获取：

```shell
# 镜像名称应该去除"k8s.gcr.io/"的前缀，版本换成上面获取到的版本
images=(
    kube-apiserver:v1.20.6
    kube-controller-manager:v1.20.6
    kube-scheduler:v1.20.6
    kube-proxy:v1.20.6
    pause:3.2
    etcd:3.4.13-0
    coredns:1.7.0
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

初始化cluster

```shell
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --kubernetes-version 1.20.6
```

### 6. 配置kubectl

要使非root用户可以运行kubectl，运行以下命令（kubeadm init成功后也会输出）：

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

或者，如果你是root用户，则可以运行：

```shell
export KUBECONFIG=/etc/kubernetes/admin.conf
```

启用shell自动补全功能：

```shell
kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl
```

或者

```shell
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

两种方法是等价的，重新加载Shell之后，kubectl的自动补齐就可以使用了

### 7. 配置网络插件

此处以flannel为例
注意：使用flannel需要在kubeadm init时配置--pod-network-cidr 10.244.0.0/16

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 8. 设置master可以运行pod

```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 9. 安装并配置LoadBalancer（可选）

MetalLB可以为k8s提供LoadBalancer
安装步骤：

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.10.2/manifests/metallb.yaml
```

编写配置文件metallb-config.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.225.200-192.168.225.200 # 改为自己服务器的IP
```

应用配置：

```shell
kubectl apply -f metallb-config.yaml
```

### 10. 安装Istio

下载安装文件

```shell
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.9.5 TARGET_ARCH=x86_64 sh -
```

设置环境变量

```shell
echo 'export PATH="$PATH:/home/xufeiyu/istio-1.9.5/bin"' | sudo tee -a /etc/profile
source /etc/profile
```

创建配置文件（在k8s v1.20.6下测试）

```yaml
cat << EOF > ./istio-minimal-operator.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      proxy:
        autoInject: enabled
      useMCP: false
      # The third-party-jwt is not enabled on all k8s.
      # See: https://istio.io/docs/ops/best-practices/security/#configure-third-party-service-account-tokens
      jwtPolicy: first-party-jwt
      imagePullPolicy: IfNotPresent

  addonComponents:
    pilot:
      enabled: true

  components:
    ingressGateways:
      - name: istio-ingressgateway
        enabled: true

  meshConfig:
    accessLogFile: /dev/stdout
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100
EOF
```

初始化istio：

```shell
istioctl install -f istio-minimal-operator.yaml -y
```

查看Pod启动状态是否都是Ready

```shell
watch kubectl get pods -A
```

开启自动注入：

```shell
kubectl label namespace default istio-injection=enabled
```

安装Jeger：

```shell
kubectl apply -f istio-1.9.5/samples/addons/jaeger.yaml
```

### 11. 安装Knative

```shell
kubectl apply -f https://github.com/knative/serving/releases/download/v0.24.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/v0.24.0/serving-core.yaml
kubectl apply -f https://github.com/knative/net-istio/releases/download/v0.24.0/net-istio.yaml
```

查看Pod启动状态是否都是Ready

```shell
watch kubectl get pods -A
```

通过运行以下命令获取外部IP地址或CNAME

```shell
kubectl --namespace istio-system get service istio-ingressgateway
```

### 12. 安装Bookinfo

```shell
cd istio-1.9.5/
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

查看Pod启动状态是否都是Ready

```Shell
watch kubectl get pods -A
```

部署网关

```shell
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

验证安装
```shell
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
export BOOKINFO_URL="http://$GATEWAY_URL/productpage"
curl -I $BOOKINFO_URL
```

创建访问脚本

```shell
vim access_bookinfo.sh
```

内容如下：

```shell
#!/bin/bash
INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n istio-system -o jsonpath='{.items[0].status.hostIP}')
GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
BOOKINFO_URL="http://$GATEWAY_URL/productpage"

count=0

access_bookinfo(){
    curl -s -o /dev/null $BOOKINFO_URL
    count=$(($count + 1))
    if (($count%10 == 0))
    then
        echo $count
    fi
    sleep 0.1
}

if (($# == 1))
then
    while (($count < $1))
    do
        access_bookinfo
    done
else
    while true
    do
        access_bookinfo
    done
fi
```

添加运行权限

```shell
chmod +x ./access_bookinfo.sh
```

### 13. 安装Chaos Mesh

使用Chaos Mesh可以对服务进行延迟增大等实验，制造异常场景

安装命令：

```shell
curl -sSL https://mirrors.chaos-mesh.org/v1.2.3/install.sh | bash
```

查看Pod启动状态是否都是Ready

```shell
watch kubectl get pods -A
```

创建配置文件：

```yaml
cat << EOF > ./network-delay.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
spec:
  action: delay # the specific chaos action to inject
  mode: one # the mode to run chaos action; supported modes are one/all/fixed/fixed-percent/random-max-percent
  selector: # pods where to inject chaos actions
    namespaces:
      - default
    labelSelectors:
      "app": "productpage"  # the label of the pod for chaos injection
  delay:
    latency: "100ms"
  duration: "3600s" # duration for the injected chaos experiment
  scheduler: # scheduler rules for the running time of the chaos experiments about pods.
    cron: "@every 1s"
EOF
```

### 14. 测试步骤

1. 启动Jaeger端口转发

   ```shell
   kubectl port-forward --namespace istio-system deployment/jaeger 16686
   ```
   
2. 运行访问脚本

   第一个参数可以指定访问的次数，如不指定则表示无限循环

   这里指定为1100次（比1000次多访问100次保证数据稳定）

   ```shell
   ./access_bookinfo.sh 1100
   ```

3. 创建测试目录

   ```shell
   mkdir -p test/traces
   mkdir -p test/samples
   ```
   
4. 采集正常样本

   ```shell
   conda activate py37
   cd traceanomaly/
   python log_collector.py --service productpage.default --duration 60 --output test/traces/20210805-1-normal.json --limit 1000
   ```

5. 运行chaos mesh以及访问脚本

   ```shell
   kubectl apply -f ~/network-delay.yaml
   ./access_bookinfo.sh 150
   ```

6. 采集异常样本并停止chaos mesh

   ```shell
   python log_collector.py --service productpage.default --duration 60 --output test/traces/20210805-1-abnormal.json --limit 100
   kubectl delete -f ~/network-delay.yaml
   ```

7. 日志解析

   ```shell
   python log_parser.py --target test/traces/20210731-2-normal.json --output test/samples/20210731-2-normal.csv
   python log_parser.py --target test/traces/20210805-1-normal.json --output test/samples/20210805-1-normal.csv --load-call-path
   python log_parser.py --target test/traces/20210805-1-abnormal.json --output test/samples/20210805-1-abnormal.csv --load-call-path
   ```

8. 模型训练

   ```shell
   python -m traceanomaly.main --trainpath test/samples/20210731-2-normal.csv --normalpath test/samples/20210805-1-normal.csv --abnormalpath test/samples/20210805-1-abnormal.csv --modelname bookinfo-test -c flow_type=rnvp --loadmodel
   ```

9. 输出预测结果

   ```shell
   python classifier.py --sample-score-path results/rnvp_bookinfo-test_train_score.csv --target-score-path results/rnvp_bookinfo-test_output_score.csv --clf-name bookinfo-test --is-labeled
   ```

### 15. 卸载k8s

```shell
kubeadm reset # 不确定是否需要sudo
sudo apt-mark unhold kubelet kubeadm kubectl
sudo apt purge -y kubelet kubeadm kubectl
```

### 16. 清理k8s

```shell
sudo ifconfig cni0 down
sudo ifconfig flannel.1 down
sudo ip link delete cni0
sudo ip link delete flannel.1

sudo rm -rf $HOME/.kube
sudo rm -rf /etc/bash_completion.d/kubectl
sudo rm -rf /etc/kubernetes
sudo rm -rf /etc/cni
sudo rm -rf /var/lib/cni
sudo rm -rf /var/lib/kubelet
```

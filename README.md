# K8S+GPU+docker19.03搭建手册

**注意**：本文安装master地址为192.168.1.9，请根据个人情况更改！！！
          本文档暂为搭建记录，后期将更新步骤搭建详情。

## 一、   基本安装

1、apt-get install openssh-server

 

2、改主机名

hostnamectl set-hostname slaver02

vim /etc/cloud/cloud.cfg  

\# This will cause the set+update hostname module to not operate (if true)

preserve_hostname: true

 

3、ubuntu18.04设置静态ip方法

sudo vim /etc/netplan/*.yaml

yaml文件名称不同，这里以正则匹配


vim /etc/resolv.conf  修改DNS

sudo netplan apply

 

4、

apt-get upgrade

apt-get update

apt-get install gcc

apt-get install make

 

5、新增用户

useradd -m nju -d /home/nju -s /bin/bash

sudo passwd nju

 

  

## 二、   安装显卡驱动、CUDA、CUDNN

1、 Nvidia官网下载显卡驱动

winscp上传到server，chmod 777 NVIDIA-Linux-x86_64-430.40.run 

./ NVIDIA-Linux-x86_64-430.40.run

 

2、安装CUDA 10.1

https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&target_distro=Ubuntu&target_version=1804&target_type=deblocal

下载上传

dpkg -i cuda-repo-ubuntu1804-10-1-local-10.1.168-418.67_1.0-1_amd64.deb 

apt-key add /var/cuda-repo-10-1-local-10.1.168-418.67/7fa2af80.pub

apt-get update

apt-get install cuda

 

查看安装的版本信息

cat /usr/local/cuda/version.txt

 

3、安装cudnn

https://developer.nvidia.com/rdp/cudnn-download

dpkg -i libcudnn7_7.6.2.24-1+cuda10.1_amd64.deb

dpkg -i libcudnn7-dev_7.6.2.24-1+cuda10.1_amd64.deb

dpkg -i libcudnn7-doc_7.6.2.24-1+cuda10.1_amd64.deb

 

验证

cp -r /usr/src/cudnn_samples_v7/ $HOME

cd $HOME/cudnn_samples_v7/mnistCUDNN

make clean && make

./mnistCUDNN 

显示test pass表示OK

 

# 三、   安装docker

1、安装docker

```
apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
software-properties-common

 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add –

 sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"
```

 

sudo apt-get update

 apt-get install docker-ce

查看版本

docker version

  

2、 安装nvidia-docker安装NVidia支持的Docker引擎

distribution=$(. /etc/os-release;echo $ID$VERSION_ID)

curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -

curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

apt-get update

apt-get install nvidia-docker2

systemctl restart docker

 

3、检查每一个节点，启用 nvidia runtime为缺省的容器运行时

vim /lib/systemd/system/docker.service 

修改

ExecStart=/usr/bin/dockerd -H fd:// --default-runtime=nvidia


vim /etc/docker/daemon.json

修改

{
​      "insecure-registries": ["**192.168.1.9:5000"**],
​       "runtimes": {
​               "nvidia": {
​                         "path": "nvidia-container-runtime",
​                               "runtimeArgs": []
​                                   }
​                                     }

}

systemctl daemon-reload && sudo systemctl restart docker

 

4、验证DOCKER调用GPU

docker run -it --rm --gpus all ubuntu nvidia-smi -L

docker run -itd --gpus all -p 5000:5000 nvidia/digits

您可以打开Web浏览器并验证它是否在以下地址上运行：

 w3m http://\<dockerhostip>:5000

 docker run --gpus '"device=0,1"' -it --rm -v $(realpath ~/notebooks):/tf/notebooks -p 8888:8888 tensorflow/tensorflow:latest-gpu-jupyter



在chrome里面打开ip：8888，输入token，打开Python，输入以下代码，在docker logs 里面查看GPU使用信息。

import tensorflow as tf

matrix1 = tf.constant([[3., 3.]])   

matrix2 = tf.constant([[2.],[2.]])  

product = tf.matmul(matrix1, matrix2) 

sess = tf.Session()  

print(tf.__version__)

# 四、   安装K8S

1、所有机器master和node上均需要下载安装以下，添加阿里源

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list

deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main

EOF

 

apt-get update

apt-get install -y kubelet kubeadm kubectl

 

 

2、**禁用** **swap**，设置swap开机不启动（否则报错The connection to the server x.x.x.x:6443）

**所有node**节点上一定要禁用 swap 

swapoff -a 

vi /etc/fstab，将文件中的/dev/mapper/centos-swap swap swap defaults 0 0这一行注释掉

free –m 若swap那一行输出为0，则说明已经关闭。

然后在运行kubelet --version 查看版本信息

 

3、下载所需的镜像，看看需要K8s什么版本的镜像，master和node上均需要下载相同版本镜像。

 

kubeadm config images list --kubernetes-version=v1.15.1

 

docker pull gcr.azk8s.cn/google-containers/kube-apiserver:v1.15.1

docker pull gcr.azk8s.cn/google-containers/kube-controller-manager:v1.15.1

docker pull gcr.azk8s.cn/google-containers/kube-scheduler:v1.15.1

docker pull gcr.azk8s.cn/google-containers/kube-proxy:v1.15.1

docker pull gcr.azk8s.cn/google-containers/pause:3.1

docker pull gcr.azk8s.cn/google-containers/etcd:3.3.10

docker pull gcr.azk8s.cn/google-containers/coredns:1.3.1

 

 docker tag gcr.azk8s.cn/google-containers/kube-apiserver:v1.15.1 k8s.gcr.io/kube-apiserver:v1.15.1

docker tag gcr.azk8s.cn/google-containers/kube-controller-manager:v1.15.1 k8s.gcr.io/kube-controller-manager:v1.15.1

docker tag gcr.azk8s.cn/google-containers/kube-scheduler:v1.15.1 k8s.gcr.io/kube-scheduler:v1.15.1

docker tag gcr.azk8s.cn/google-containers/kube-proxy:v1.15.1 k8s.gcr.io/kube-proxy:v1.15.1

docker tag gcr.azk8s.cn/google-containers/pause:3.1 k8s.gcr.io/pause:3.1

docker tag gcr.azk8s.cn/google-containers/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10

docker tag gcr.azk8s.cn/google-containers/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1

 

4、kubeadm init初始化集群（在master上操作，Kubeadm init初始化，出现一个token。Pod network好像都是用这个的10.244.0.0/16一定要加版本号），仅仅只在master上操作

kubeadm init --apiserver-advertise-address=192.168.1.9 --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.15.1

链接master，加入集群：

 kubeadm join 192.168.1.9:6443 --token “your token”

 

5、查看安装状态

 

在master上执行以下操作。

export KUBECONFIG=/etc/kubernetes/admin.conf

 

然后在master上执行：

kubectl get nodes

显示都是not ready，后面加了fannel网络就好了。

 

6、创建flannel 参考：[https://github.com/coreos/flannel](https://github.com/coreos/flannel) 



在 master上执行如下命令。创建flannel网络。节点node上不用创建网络。

 kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

 如需删除，kubectl delete –f kube-flannel.yml



 此时，在master上查看pod的情况

kubectl get nodes

kubectl get pods --all-namespaces

kubectl get pods -n kube-system

 kubectl describe node

 显示master状态为ready状态。

 

7、在其他几台node上执行kubeadm join操作

kubeadm join 192.168.100.24:6443 --token “your token”

  

8、检查K8S master和节点状态

Master上执行

kubectl get node

 

9、通过yaml文件安装dashboard

wget https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml 

 

可以从目前机器上拷贝该文件

 

修改yaml文件中service的type类型

官方的kubernetes-dashboard.yaml文件中service的type类型为clusterIp(service默认类型)，这种方式要访问dashboard需要通过代理，所以我们改为NodePort方式，这样部署完后，就可以直接通过nodeIP:port的方式访问，端口范围0000-30067

 

下载dashboard镜像，一定在所有机器上执行以下操作，下载镜像，因为dashboard会随机在节点上启动的。 

 docker pull gcrxio/kubernetes-dashboard-amd64:v1.10.1

 docker tag gcrxio/kubernetes-dashboard-amd64:v1.10.1 k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1

 

部署

kubectl create -f kubernetes-dashboard.yaml

 

 kubectl get pods -n kube-system 检查状态。

部署完成后，可以通过kubectl get svc,pod -n kube-system来查看是否部署成功



10、创建用户

 参考https://github.com/kubernetes/dashboard/wiki/Creating-sample-user

 创建服务账号 Create Service Account

利用vi admin-user.yaml命令创建admin-user.yaml文件，输入以下内容,来创建admin-user的服务账号，放在kube-system名称空间下：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```

输入命令`kubectl create -f admin-user.yaml`来执行，。

**绑定角色**

利用`vi admin-user-role-binding.yaml`命令创建admin-user-role-binding.yaml文件，输入以下内容,来进行绑定

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

输入命令`kubectl create -f admin-user-role-binding.yaml`来执行。

**获取token**

输入以下命令来创建用户token，利用token来登录dashboard：

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

 

11、登录K8S

当部署成功后，我们通过firefox浏览器就可以访问dashboard的图形界面了

https://192.168.1.9:30088

 

 

# 五、   在Kubernetes中启用GPU支持

1、 在 master 节点上执行，好像所有节点都要执行

 

kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.11/nvidia-device-plugin.yml

 

kubectl get nodes "-o=custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu"

 

NAME    GPU

master   <none>

slaver01  2

slaver02  1

 

# 六、    K8S 中调用GPU

```
vim tf-gpu-new.yaml
```

内容的组织形式:

```
apiVersion: apps/v1
kind: Deployment
metadata:
 name: tf-gpu
spec:
 replicas: 3
 selector:
  matchLabels:
   app: tf-gpu
 template:
  metadata:
   labels:
	app: tf-gpu
  spec:
   containers:
   \- name: tensorflow
	image: tensorflow/tensorflow:latest-gpu-jupyter
	ports:
	\- containerPort: 8888
	resources:
		limits:
			nvidia.com/gpu: 1
```

 

kubectl create -f tf-gpu-new.yaml              

 

kubectl expose deploy tf-gpu --type LoadBalancer --external-ip=192.168.1.9 --port 8888 --target-port 8888

 

docker logs 3306eabdc6e3 查看docker id的日志和token

 

docker exec -i -t 3306eabdc6e3 /bin/bash 进入容器

 

kubectl describe svc 查看node节点IP及端口

 

# 七、    开机以后的命令

export KUBECONFIG=/etc/kubernetes/admin.conf

 

kubectl get node查看节点信息状态

 

如其他节点也要执行kubectl get node命令：

需将主节点中的【/etc/kubernetes/admin.conf】文件拷贝到从节点相同目录下，然后配置环境变量：

echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile

立即生效

source ~/.bash_profile

接着再运行kubectl命令就OK了

 

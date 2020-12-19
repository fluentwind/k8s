
# k8s ansible部署

### 第一步创建一个虚拟机

#### 设置ip

``` bash
cd /etc/sysconfig/network-scripts/
vi ifcfg-eth0
```
    
#### 设置hostname

``` bash
hostnamectl set-hostname kmaster
```
    
### 安装ansible

``` bash
yum install -y http://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
yum -y install ansible
```

#### 设置ansible hosts

``` bash
vi /etc/ansible/hosts
[k8s-all]
192.168.40.201
192.168.40.202
192.168.40.205
192.168.40.206
```

#### 生成ssh key copy到其他节点，实现免密
在管理节点执行 ssh-keygen 生成SSH KEY，然后copy到各被管理节点上
``` bash
ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.40.201

ansible k8s-all -m ping #ok了就行
```

### 所有节点安装docker
``` yml
vim install_docker_playbook.yml
- hosts: k8s-all
  remote_user: root
  vars: 
     docker_version: 18.09.2

  tasks: 
     - name: install dependencies
       #shell: yum install -y yum-utils device-mapper-persistent-data lvm2 
       yum: name={{item}} state=present
       with_items:
          - yum-utils
          - device-mapper-persistent-data
          - lvm2

     - name: config yum repo
       shell: yum-config-manager --add-repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo

     - name: install docker
       yum: name=docker-ce-{{docker_version}} state=present

     - name: start docker
       shell: systemctl enable docker && systemctl start docker
```

``` bash
ansible-playbook install_docker_playbook.yml
```

### 安装master 记得修改master ip
``` yml
vim deploy_master_playbook.yml
- hosts: k8s-master
  remote_user: root：q
  vars:
    kube_version: 1.16.0-0
    k8s_version: v1.16.0
    k8s_master: 192.168.40.111 
  
  tasks:
    - name: prepare env
      script: ./pre-setup.sh      

    - name: install kubectl,kubeadm,kubelet
      yum: name={{item}} state=present
      with_items:
        - kubectl-{{kube_version}}
        - kubeadm-{{kube_version}}
        - kubelet-{{kube_version}}
    
    - name: init k8s
      shell: kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version {{k8s_version}} --apiserver-advertise-address {{k8s_master}}  --pod-network-cidr=10.244.0.0/16 --token-ttl 0
    
    - name: config kube
      shell: mkdir -p $HOME/.kube && cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && chown $(id -u):$(id -g) $HOME/.kube/config
    
    - name: copy flannel yaml file
      copy: src=./kube-flannel.yml dest=/tmp/ owner=root group=root mode=0644 
    
    - name: install flannel
      shell: kubectl apply -f /tmp/kube-flannel.yml

    - name: get join command
      shell: kubeadm token create --print-join-command 
      register: join_command
    - name: show join command
      debug: var=join_command verbosity=0
```

``` bash
ansible-playbook deploy_master_playbook.yml
```

复制出k8s加入集群的token

``` bash
kubeadm join 192.168.58.220:6443 --token zgx3ov.zlq3jh12atw1zh8r .....
```


### 安装node 记得修改master ip
``` yml
vim deploy_nodes_playbook.yml
- hosts: k8s-nodes
  remote_user: root
  vars:
     kube_version: 1.16.0-0

  tasks:
    - name: prepare env
      script: ./pre-setup.sh

    - name: install kubeadm,kubelet
      yum: name={{item}} state=present
      with_items:
        - kubeadm-{{kube_version}}
        - kubelet-{{kube_version}}
    
    - name: start kubelt
      shell: systemctl enable kubelet && systemctl start kubelet
   
    - name: join cluster
      shell: kubeadm join 192.168.40.111:6443 --token zgx3ov.zlq3jh12atw1zh8r --discovery-token-ca-cert-hash sha256:60b7c62687974ec5803e0b69cfc7ccc2c4a8236e59c8e8b8a67f726358863fa7
```

``` bash
ansible-playbook deploy_nodes_playbook.yml
```

#### kubectl get nodes 查看集群状态

都ready了往下走，如果起不来一般是flannel镜像被墙了，可以手动下载并tag到每个节点

### 安装ingress
``` yml
vim nginx-ingress.yaml
:s/quay.io/quay-mirror.qiniu.com/g


vim nginx-ingress.yaml

    spec:
      hostNetwork: true
      nodeSelector:
        nginx-ingress: "true"
```
首先在knode1节点上打标签nginx-ingress=true，控制Ingress部署到knode1上，保持IP固定。

``` bash
kubectl label node knode1 nginx-ingress=true
node/knode1 labeled
kubectl apply -f nginx-ingress.yaml

#查看状态
kubectl get pods -n ingress-nginx -o wide
```

### 安装dashboard

创建自定义证书
``` bash
cd /etc/kubernetes/pki/
#生成私钥
openssl genrsa -out dashboard.key 2048
#生成证书
openssl req -new -key dashboard.key -out dashboard.csr -subj "/O=JBST/CN=kubernetes-dashboard"
#使用集群的CA来签署证书
openssl x509 -req -in dashboard.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dashboard.crt -days 3650
#查看自创证书
openssl x509 -in dashboard.crt -noout -text

kubectl apply -f kubernetes-dashboard.yaml 
kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.crt=/etc/kubernetes/pki/dashboard.crt --from-file=dashboard.key=/etc/kubernetes/pki/dashboard.key  -n kubernetes-dashboard

#创建cluster-admin
kubectl apply -f kubernetes-dashboard-auth.yaml
#获取token
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

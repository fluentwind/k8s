
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
    
#### 安装ansible

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


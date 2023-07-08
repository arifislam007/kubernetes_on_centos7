# kubernetes_on_centos7
This repository is for deploying kubernetes on centos 7

https://www.hostafrica.co.za/blog/new-technologies/install-kubernetes-delpoy-cluster-centos-7/

#Setup Master Node: 

#Step-1: Setup Host information on hosts file. 
hostnamectl set-hostname master-node
cat <<EOF> /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.200.205.194 master-node
10.200.205.195 node-1 worker-node-1
10.200.205.196 node-2 worker-node-2
EOF
#Step-2: disable SElinux and update your firewall rules.

setenforce 0
# Permanently disable selinux from /etc/selinux/config
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

#Set the following firewall rules
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10251/tcp
firewall-cmd --permanent --add-port=10252/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --reload


cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

#Step-3: Setup the Kubernetes Repo.

yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo


cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

#Or create a repo with the following content 
#Start From Here 
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
## End here 

#Step-4: Install Kubeadm and Docker
yum install -y kubelet kubeadm kubectl docker-ce


# disable swap permanently otherwise kubernet service will not start 
swapoff -a

#Why disable swap on kubernetes?
#The idea of kubernetes is to tightly pack instances to as close to 100% utilized as possible. 
#All deployments should be pinned with CPU/memory limits. So if the scheduler sends a pod 
#to a machine it should never use swap at all. You don't want to swap since it'll slow things down.
#Its mainly for performance.


#Initializing Kubernetes master with the following command 
kubeadm init


#enable and start both services
systemctl enable kubelet
systemctl start kubelet
systemctl enable docker
systemctl start docker




#Please copy the last line of the output and save it somewhere because you will need to run it on the worker nodes.

kubeadm join 10.200.205.194:6443 --token puy66y.4bi4lxrzel95lpv3 \
        --discovery-token-ca-cert-hash sha256:debc459ae02f697011f4f2e7d23fa29ce54febc1354f7a6e52fab0875a453e34
		

If you forgot to copy the command, or have misplaced it, don’t worry. You can retrieve it again by entering the following command:

sudo kubeadm token create --print-join-command


# Having initialized Kubernetes successfully, you will need to allow your user to start using the cluster

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

#Now check
kubectl get nodes


#Steps-5: Setup Your Pod Network. we will use Weavenet plugin
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml



#Step-6: Worker Nodes to Join Kubernetes Cluster
#Setup Host information on hosts file on node-1. 
hostnamectl set-hostname node-1

10.200.205.194 master-node
10.200.205.195 node-1 worker-node-1
10.200.205.196 node-2 worker-node-2

#Step-7: disable SElinux and update your firewall rules.

setenforce 0
# Permanently disable selinux from /etc/selinux/config
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

#Set the following firewall rules
firewall-cmd --permanent --add-port=6783/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --permanent --add-port=30000-32767/tcp
firewall-cmd  --reload

modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables

#Step-8: Setup the Kubernetes Repo

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
#gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

#Or create a repo with the following content 
#Start From Here 
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
## End here 

#Step-9: Install Kubeadm and Docker
yum install kubeadm docker -y 

#enable and start both services
systemctl enable kubelet
systemctl start kubelet
systemctl enable docker
systemctl start docker

# disable swap
swapoff -a

#Step-10: Join the Worker Node to the Kubernetes Cluster
kubeadm join 10.128.0.27:6443 --token nu06lu.xrsux0ss0ixtnms5  --discovery-token-ca-cert-hash sha256:f996ea3564e6a07fdea2997a1cf8caeddafd6d4360d606dbc82314688425cd41 

#Same configuration with worker node-2 for join kuberbet cluster 






Troubleshooting 
Trouble 01: 
“[ERROR CRI]: container runtime is not running: output:” Code Answer

[ERROR CRI]: container runtime is not running: output:shell by devops unicorn on May 14 2022 Comment
0
​systemctl restart containerd
1. rm /etc/containerd/config.toml
2. systemctl restart containerd
3. kubeadm init

After this problem has been solved and kubernet has been started 


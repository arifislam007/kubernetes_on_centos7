# kubernetes_on_centos7
This repository is for deploying kubernetes on centos 7

https://www.hostafrica.co.za/blog/new-technologies/install-kubernetes-delpoy-cluster-centos-7/

#Setup Master Node: </br>

#Step-1: Setup Host information on hosts file. </br>
hostnamectl set-hostname master-node </br>
cat <<EOF> /etc/hosts </br>
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 </br>
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6 </br>
10.200.205.194 master-node </br>
10.200.205.195 node-1 worker-node-1 </br>
10.200.205.196 node-2 worker-node-2 </br>
EOF </br>
#Step-2: disable SElinux and update your firewall rules. </br>

setenforce 0 </br>
# Permanently disable selinux from /etc/selinux/config </br>
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config </br>

#Set the following firewall rules </br>
firewall-cmd --permanent --add-port=6443/tcp </br>
firewall-cmd --permanent --add-port=2379-2380/tcp </br>
firewall-cmd --permanent --add-port=10250/tcp</br>
firewall-cmd --permanent --add-port=10251/tcp</br>
firewall-cmd --permanent --add-port=10252/tcp</br>
firewall-cmd --permanent --add-port=10255/tcp</br>
firewall-cmd --reload</br>
</br>

cat <<EOF > /etc/sysctl.d/k8s.conf</br>
net.bridge.bridge-nf-call-ip6tables = 1</br>
net.bridge.bridge-nf-call-iptables = 1</br>
EOF</br>
sysctl --system</br>
</br>
#Step-3: Setup the Kubernetes Repo.</br>
</br>
yum install -y yum-utils device-mapper-persistent-data lvm2</br>
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo</br>
</br>

cat <<EOF > /etc/yum.repos.d/kubernetes.repo</br>
[kubernetes]</br>
name=Kubernetes</br>
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64</br>
enabled=1</br>
gpgcheck=0</br>
repo_gpgcheck=0</br>
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg</br>
EOF</br>

#Or create a repo with the following content </br>
#Start From Here </br>
[kubernetes]</br>
name=Kubernetes</br>
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64</br>
enabled=1</br>
gpgcheck=0</br>
repo_gpgcheck=0</br>
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg</br>
#End here </br>

#Step-4: Install Kubeadm and Docker</br>
yum install -y kubelet kubeadm kubectl docker-ce</br>


## disable swap permanently otherwise kubernet service will not start </br>
swapoff -a</br>

#Why disable swap on kubernetes?</br>
#The idea of kubernetes is to tightly pack instances to as close to 100% utilized as possible. </br>
#All deployments should be pinned with CPU/memory limits. So if the scheduler sends a pod </br>
#to a machine it should never use swap at all. You don't want to swap since it'll slow things down.</br>
#Its mainly for performance.</br>

</br>
#Initializing Kubernetes master with the following command </br>
kubeadm init</br>


#enable and start both services</br>
systemctl enable kubelet</br>
systemctl start kubelet</br>
systemctl enable docker</br>
systemctl start docker</br>




#Please copy the last line of the output and save it somewhere because you will need to run it on the worker nodes.</br>
</br>
kubeadm join 10.200.205.194:6443 --token puy66y.4bi4lxrzel95lpv3 \
        --discovery-token-ca-cert-hash sha256:debc459ae02f697011f4f2e7d23fa29ce54febc1354f7a6e52fab0875a453e34</br>
		
</br>
If you forgot to copy the command, or have misplaced it, don’t worry. You can retrieve it again by entering the following command:</br>
</br>
sudo kubeadm token create --print-join-command</br>
</br>

## Having initialized Kubernetes successfully, you will need to allow your user to start using the cluster
</br>
mkdir -p $HOME/.kube</br>
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config</br>
chown $(id -u):$(id -g) $HOME/.kube/config</br>
</br>
#Now check</br>
kubectl get nodes</br>

</br>
#Steps-5: Setup Your Pod Network. we will use Weavenet plugin</br>
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml</br>

</br>

#Step-6: Worker Nodes to Join Kubernetes Cluster</br>
#Setup Host information on hosts file on node-1. </br>
hostnamectl set-hostname node-1</br>

10.200.205.194 master-node</br>
10.200.205.195 node-1 worker-node-1</br>
10.200.205.196 node-2 worker-node-2</br>
</br>
#Step-7: disable SElinux and update your firewall rules.</br>
</br>
setenforce 0</br>
## Permanently disable selinux from /etc/selinux/config</br>
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config</br>
</br>
#Set the following firewall rules</br>
firewall-cmd --permanent --add-port=6783/tcp</br>
firewall-cmd --permanent --add-port=10250/tcp</br>
firewall-cmd --permanent --add-port=10255/tcp</br>
firewall-cmd --permanent --add-port=30000-32767/tcp</br>
firewall-cmd  --reload</br>
</br>
modprobe br_netfilter</br>
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables</br>
</br>
#Step-8: Setup the Kubernetes Repo</br>
</br>
cat <<EOF > /etc/yum.repos.d/kubernetes.repo</br>
[kubernetes]</br>
name=Kubernetes</br>
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64</br>
enabled=1</br>
gpgcheck=0</br>
repo_gpgcheck=0</br>
#gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg</br>
EOF</br>
</br>
#Or create a repo with the following content </br>
#Start From Here </br>
[kubernetes]</br>
name=Kubernetes</br>
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64</br>
enabled=1</br>
gpgcheck=1</br>
repo_gpgcheck=1</br>
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg</br>
## End here </br>
</br>
#Step-9: Install Kubeadm and Docker</br>
yum install kubeadm docker -y </br>
</br>
#enable and start both services</br>
systemctl enable kubelet</br>
systemctl start kubelet</br>
systemctl enable docker</br>
systemctl start docker</br>
</br>
#disable swap</br>
swapoff -a</br>

## Step-10: Join the Worker Node to the Kubernetes Cluster
kubeadm join 10.128.0.27:6443 --token nu06lu.xrsux0ss0ixtnms5  --discovery-token-ca-cert-hash</br> sha256:f996ea3564e6a07fdea2997a1cf8caeddafd6d4360d606dbc82314688425cd41 </br>
</br>
#Same configuration with worker node-2 for join kuberbet cluster </br>
</br>

## Troubleshooting 
Trouble 01: </br>
“[ERROR CRI]: container runtime is not running: output:” Code Answer</br>
</br>
[ERROR CRI]: container runtime is not running: output:shell by devops unicorn on May 14 2022 Comment</br>
0</br>
​systemctl restart containerd</br>
1. rm /etc/containerd/config.toml</br>
2. systemctl restart containerd</br>
3. kubeadm init</br>
</br>
## After this problem has been solved and kubernet has been started 


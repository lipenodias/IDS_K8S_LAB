sudo yum clean all

sudo yum update -y

sudo yum groupinstall 'Development Tools'

sudo yum install wget

curl -fsSL https://get.docker.com | bash

sudo systemctl enable docker

sudo systemctl start docker

#Editar o repositorio do CentOS para adicionar o repo do K8S

vim /etc/yum.repos.d/kubernetes.repo

##############################################################################################
[kubernetes]

name=Kubernetes

baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64

enabled=1

gpgcheck=1

repo_gpgcheck=1

gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg

###############################################################################################

sudo setenforce 0
sudo sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux

sudo swapoff -a

vim /etc/fstab

#Devemos comentar a seguinte linha para não habilitar mais o swap
# /dev/mapper/centos-swap swap swap defaults 0 0

sudo systemctl stop firewalld

sudo systemctl disable firewalld

sudo yum install -y kubelet kubeadm kubectl

sudo systemctl enable kubelet && sudo systemctl start kubelet

#Configurar alguns parâmetros de kernel no sysctl

vim /etc/sysctl.d/k8s.conf

##################################################
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
##################################################

sudo sysctl --system

sudo systemctl daemon-reload
sudo systemctl restart kubelet

#Iniciando o cluster

kubeadm init --apiserver-advertise-address $(hostname -i)

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

kubectl get pods -n kube-system

#Implantação do DASHBOARD K8S

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml

#Definição do External IP para o serviço dashboard

kubectl patch svc -n kubernetes-dashboard kubernetes-dashboard -p '{"spec":{"externalIPs":["191.252.1.132"]}}'

#Criar usuário e dar permissão para o token gerenciar o DASHBOARD

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
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
  namespace: kubernetes-dashboard
EOF

#adquirir o token

kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
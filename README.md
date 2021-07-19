#Instalar un Cluster de Kubernetes

##Instalamos el nodo Master

######Actualizar paquetes
apt-get update && apt-get upgrade -y

######Instalar dependencias
apt-get install -y apt-transport-https

######Deshabilitar swap
vi /etc/fstab
>> se comenta la linea de swap
swapoff -a

######Instalar docker
apt install docker.io -y

######Instalar ebtables ethtool para iptables
apt-get install ebtables ethtool

######Instalar paquetes de google y kubectl, kubeadm y kubelet
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
apt-get install -y kubelet kubeadm kubectl

##Rebootear la maquina

######EJECUTAR EN (MASTER)
kubeadm init --pod-network-cidr=10.100.0.0/16
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes

######Habilitar el servicio de docker
systemctl enable docker.service

######Instalar Red Flannel
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

######Bindear el role cluster-admin a la cuenta default
kubectl create clusterrolebinding default-admin --clusterrole cluster-admin --serviceaccount=default:default

######Crear Ingress Nginx operator
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/baremetal/deploy.yaml
kubectl edit service -n ingress-nginx [NOMBRE DEL SERVICIO]
>> CAMBIAR LOS NODEPORT de los puerto 80 y 443 por 31080 y 31443

######Install metric-server. Este servicio es importante para que luego funcione HPA y poder tener métricas de los pods.
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
Editar el deployment y agregar insecure tls (https://github.com/kubernetes-sigs/metrics-server/issues/146)
kubectl edit deployment metrics-server -n kube-system
>> Agregar --kubelet-insecure-tls en los annotations


##Agregar un nodo a un cluster ya existente

######Ejecutar los siguientes comandos en master:
kubeadm token generate
kubeadm token create <generated-token> --print-join-command --ttl=0

######En el nodo realizamos lo siguiente
kubeadm join ipmaster:6443 --token si7gyx.f3x665axop5q8o9s --discovery-token-ca-cert-hash sha256:64e8139fd5b4741864620156f2291ad3e0efcdb2c65aa99fea8b1481bd693cd3

##Configurar Kubectl para nuestro Cluster en una terminal

######Ejecutar lo siguiente desde el master con kubectl previamente ya configurado (Nodo Master) y obtener el token:

APISERVER=$(kubectl config view -minify | grep server | cut -f 2 -d ":" | tr -d " ")
SECRET_NAME=$(kubectl get secrets | grep ^default | cut -f1 -d ' ')
TOKEN=$(kubectl describe secret $SECRET_NAME | grep -E '^token' | cut -f2 -d':' | tr -d " ")
curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
echo $TOKEN

######Ejecutar las siguientes lineas desde la maquina remota con kubectl ya instalado:

kubectl config set-cluster nombrecluster --server=https://10.0.20.5:6443 --insecure-skip-tls-verify=true
kubectl config set-credentials admin-nombrecluster --token=$TOKEN
{REEMPLAZAR TOKEN}
kubectl config set-context stage --cluster=nombrecluster --user=admin-nombrecluster
kubectl config use-context nombrecluster

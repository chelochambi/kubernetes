# Manual de instalación de nodos workers en el cluster de Kubernetes

Para la creación e instalación de nuevos nodos workers en el cluster de Kubernetes, se debe seguir los siguientes pasos haciendo ssh dentro el nuevo worker:


##### 1. Instalación del Firewall

Primero se debe deshabilitar los servicios de firewalld:

```sh
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo systemctl mask firewalld
```

Instalar el repositorio de **ufw**:

```sh
sudo dnf install epel-release -y
```

Instalando y habilitando **ufw**

```sh
sudo dnf install ufw -y

sudo ufw enable
# Escojer la opción aceptar "s"

sudo systemctl enable ufw
```
##### 2. Habilitar los puertos a para el worker

Por defecto todos los puertos se hallan deshabilitados en **ufw**. Se deben habilitar de la siguiente manera:

Ejecutar cada instrucción en orden en el host del worker:

```sh
sudo ufw allow 30000:32767/tcp
sudo ufw allow 10251/tcp
sudo ufw allow 10252/tcp
sudo ufw allow 8472/udp
sudo ufw allow 10255/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10256/tcp
sudo ufw allow 179/tcp
sudo ufw allow 4789/udp
sudo ufw allow 5473/tcp
```

##### 3. Deshabilitar el swap

```sh   
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```  

##### 4. Deshabilitar las configuraciones de **selinux** a nivel warning

```sh 
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```   

##### 5. Habilitar módulos necesarios
 
```sh      
sudo modprobe br_netfilter
sudo modprobe overlay
sudo sh -c "echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables"
sudo sh -c "echo '1' > /proc/sys/net/bridge/bridge-nf-call-ip6tables"
sudo sh -c "echo '1' > /proc/sys/net/ipv4/ip_forward"
```   

NOTA.- Se puede configurar estos mismos valores añadiéndolos a un archivo conf (/usr/lib/sysctl.d/50-default.conf)  
Ejemplo:

```sh    
nano /usr/lib/sysctl.d/50-default.conf

net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```   
Luego aplicar:


```sh
sysctl --system   
```

##### 6.  Instalar Docker
Instalar el repo de Docker

```sh 
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
```

Instalar Docker

```sh 
sudo dnf install docker-ce -y
```
Se debe arrancar el servicio Docker:
```sh
sudo systemctl start docker
```

Modificar el archivo de configuración de Docker (/etc/docker/daemon.json):
```sh 
nano /etc/docker/daemon.json
```
```sh 
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",  
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```
Reiniciamos el engine de Docker y lo habilitamos en el arranque

```sh
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
```


##### 7. Instalando los binarios de Kubernetes

Instalar el repositorio de los binarios:

```sh
 cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

Instalar kubeadm, kubectl y kubelet:

```sh
sudo dnf install -y kubelet-1.22.0 kubeadm-1.22.0 kubectl-1.22.0 --disableexcludes=kubernetes
```

Habilitar inmediatemente el servicio de kubelet:

```sh
sudo systemctl enable --now kubelet
```

Reiniciar el servicio

```sh
systemctl daemon-reload
systemctl restart kubelet
```

##### 8. Uniendo y arrancando el nodo worker

Para unir y arrancar el nodo worker se debe ejecutar primero el siguiente comando mediante ssh en el **primer nodo master inicializado**:

```sh

kubeadm token create --print-join-command
```

***Copiar la salida del anterior comando***


Luego hacer ssh en el nuevo nodo worker y pegar el resultado del anterior comando y dar Enter. El nuevo nodo worker estará inicializado.


# Manual de instalación de ETCD externo

##### 1. Abrir los puertos de los host:

```sh   
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --add-masquerade --permanent
firewall-cmd --reload
systemctl restart firewalld
firewall-cmd --list-ports  
``` 
##### 2. Deshabilitar el swap

```sh   
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab  
```  

##### 3. Deshabilitar las configuraciones de seguridad

```sh 
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```   

##### 4. Habilitar módulos necesarios
 
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
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```   
Luego aplicar:


```sh
sysctl --system
```

##### 5.  Instalar Docker
Instalar el repo de Docker

```sh 
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
```

Instalar Docker

```sh 
sudo dnf install docker-ce -y
```

Modificar el archivo de configuración de Docker (/etc/docker/daemon.json):

```sh
mkdir /etc/docker/
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


##### 6. Instalando los binarios de Kubernetes

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

Configurar el kubelet:

```sh
sudo cat << EOF > /usr/lib/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
[Service]
ExecStart=
ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd 
Restart=always
EOF
```

Reiniciar el servicio

```sh
systemctl daemon-reload
systemctl restart kubelet
```
Verificar que el kubelet este ejecutandose sin problemas:

```sh
systemctl status kubelet
```

##### 8. Iniciando configuración de los ETCD

En el primero nodo de ETCD, crear el script llamado etcd.sh:

```sh
export HOST0="Escribir la IP del primer nodo ETCD"
export HOST1="Escribir la IP del segundo nodo ETCD"
export HOST2="Escribir la IP del tercer nodo ETCD"

export NAME0="s-nal-pr-ke-01"
export NAME1="s-nal-pr-ke-02"
export NAME2="s-nal-pr-ke-03"

mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

HOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=(${NAME0} ${NAME1} ${NAME2})

for i in "${!HOSTS[@]}"; do
HOST=${HOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
---
apiVersion: "kubeadm.k8s.io/v1beta3"
kind: InitConfiguration
nodeRegistration:
    name: ${NAME}
localAPIEndpoint:
    advertiseAddress: ${HOST}
---
apiVersion: "kubeadm.k8s.io/v1beta3"
kind: ClusterConfiguration
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
            initial-cluster: ${NAMES[0]}=https://${HOSTS[0]}:2380,${NAMES[1]}=https://${HOSTS[1]}:2380,${NAMES[2]}=https://${HOSTS[2]}:2380
            initial-cluster-state: new
            name: ${NAME}
            listen-peer-urls: https://${HOST}:2380
            listen-client-urls: https://${HOST}:2379
            advertise-client-urls: https://${HOST}:2379
            initial-advertise-peer-urls: https://${HOST}:2380
EOF
done  
```
Luego de creado el script. Contextualizar:

```sh
source etcd.sh
```

Posteriormente ejecutar:

```sh
chmod +x etcd.sh
./etcd.sh
```

##### 9. Crear certificados iniciales y llave para ETCD

Haciendo ssh en el primer nodo, se debe declarar la instrucción inicial para crear los certificados del ETCD

```sh
kubeadm init phase certs etcd-ca
```
Esto creará los dos siguientes archivos:

```sh
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/etcd/ca.key
```

##### 10. Crear los certificados para cada uno de los miembros del ETCD

Ingresando por ssh al primer nodo ETCD, se contextualizar el archivo ***etcd.sh***

```sh
source etcd.sh
```

Ejecutar en orden cada una de las siguientes instrucciones:

```sh
kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST2}/
# limpiando los certificados no usables
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST1}/
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
# No hay necesidad de mover los certificados, ya que son para el HOST0

# limpiando los certificados que no deben ser copiados a este host
find /tmp/${HOST2} -name ca.key -type f -delete
find /tmp/${HOST1} -name ca.key -type f -delete
```

##### 11. Envio de certificados y llaves a los demas miembros del ETCD

Luego de crearse los certificados, se debe enviar a los dos miembros restantes, los certificados y llaves creados.

Primero al miembro 2 del ETCD:

```sh
USER=root
HOST=${HOST1}
scp -r /tmp/${HOST}/* ${USER}@${HOST}:
ssh ${USER}@${HOST}
# Una vez dentro del segundo miembro del ETCD hacer
USER@HOST $ sudo -Es
root@HOST $ chown -R root:root pki
root@HOST $ mv pki /etc/kubernetes/
```

Luego al miembro 3 del ETCD:

```sh
USER=root
HOST=${HOST2}
scp -r /tmp/${HOST}/* ${USER}@${HOST}:
ssh ${USER}@${HOST}
# Una vez dentro del tercer miembro del ETCD hacer
USER@HOST $ sudo -Es
root@HOST $ chown -R root:root pki
root@HOST $ mv pki /etc/kubernetes/
```

##### 12. Asegurarse de que los archivos se crearon correctamente en cada miembro del ETCD

En el primero miembro debe existir el siguiente árbol archivos:

```sh
/tmp/${HOST0}
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── ca.key
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```

En el segundo miembro del ETCD debe existir el siguiente árbol archivos:

```sh
$HOME
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```


En el tercer miembro del ETCD debe existir el siguiente árbol de archivos:


```sh
$HOME
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```


###### 13. Generar los archivos de manifiesto en cada miembro del ETCD

Se debe arrancar los contenedores de Docker que serán definidos en la carpeta /etc/kubernetes/manifests

En el primer nodo miembro del ETCD, ejecutar:

```sh
kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
```

En el segundo nodo miembro del ETCD, ejecurar:

```sh
kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml
```

En el tercer node miembro del ETCD, ejecutar:

```sh
kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml
```

Esto arrancará dos contenedores Docker del ETCD, uno es el contenedor auxiliar **pause** y el otro es el contenedor oficial del ETCD. Se puede verificar los dos contenedores con la instrucción:

```sh
docker ps 
```

##### 14. Verificar la salud del cluster ETCD

Se debe verificar finalmente que cada miembro del cluster ETCD, esté funcionando correctamente. Haciendo **ssh** dentro el primer nodo del clúster ETCD ejecutar:

```sh
docker run --rm -it --net host -v /etc/kubernetes:/etc/kubernetes k8s.gcr.io/etcd:3.5.0-0 etcdctl --cert /etc/kubernetes/pki/etcd/peer.crt --key /etc/kubernetes/pki/etcd/peer.key --cacert /etc/kubernetes/pki/etcd/ca.crt --endpoints https://${HOST0}:2379 endpoint health --cluster -w table
```
Asegurándose que ${HOST0} corresponde al IP del primer nodo miembro del ETCD. También debe cerciorarse de que la versión de ETCD es la misma de la imagen que se esta utillizando. (Verificar con la instrucción **docker images**)

La respuesta debería ser **similar** a esta:


```sh
https://[HOST0 IP]:2379 is healthy: successfully committed proposal: took = 16.283339ms
https://[HOST1 IP]:2379 is healthy: successfully committed proposal: took = 19.44402ms
https://[HOST2 IP]:2379 is healthy: successfully committed proposal: took = 35.926451ms
```





# Manual de instalación de los nodos **masters** de un cluster de Kubernetes con ETCD externo

Ls instalación de los nodos masters se realiza sobre distribuciones Linux de tipo ya sea Almalinux o Rocky Linux.

 
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
```
##### 2. Habilitar los puertos a para el master

Por defecto todos los puertos se hallan deshabilitados en **ufw**. Se deben habilitar de la siguiente manera:

Ejecutar cada instrucción en orden en el host del master:

```sh
sudo ufw allow 6443/tcp
sudo ufw allow 10251/tcp
sudo ufw allow 10252/tcp
sudo ufw allow 8472/udp
sudo ufw allow 10255/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10256/tcp
sudo ufw allow 179/tcp
sudo ufw allow 4789/udp
sudo ufw allow 5473/tcp


sudo systemctl enable ufw
sudo systemctl start ufw
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
Iniciar el servicio de Docker

```sh
sudo systemctl start docker
sudo systemctl enable docker
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

##### 8. Iniciando el PRIMER maestro


Para iniciar el primer maestro se debe crear el archivo de configuración: ***kubeadm-config.yaml***:

Para instalar el master en modo de **proxy reverso** por software, se debe seguir las instrucciones de [haproxy](doc/balanceador.md)
Posteriormente editar el archivo **/etc/hosts** del nodo master donde se esta instalando para que apunte al **server** donde esta haproxy.
Por ejemplo

```sh
cat /etc/hosts
```
Indicará

```sh
172.19.11.167 xxxx.impuestos.gob.bo
```
De esta forma al inicializar el nodo master con kubeadm este apuntará al balanceador.

```sh
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.22.0
controlPlaneEndpoint: "xxxx.impuestos.gob.bo:6443" 
etcd:
  external:
    endpoints:
      - https://"IP DEL PRIMER NODO DEL CLUSTER ETCD":2379
      - https://"IP DEL SEGUDO NODO DEL CLUSTER ETCD":2379
      - https://"IP DEL TERCER NODO DEL CLUSTER ETCD":2379 
    caFile: /etc/kubernetes/pki/etcd/ca.pem
    certFile: /etc/kubernetes/pki/etcd.pem
    keyFile: /etc/kubernetes/pki/etcd-key.pem
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "192.168.0.0/16"
```
Copiamos los certificados del nodo ETCD 1 al nodo maestro 1
```sh
mkdir -p /etc/kubernetes/pki/etcd

#scp -r root@"<IP de nodo ETD 1>":/etc/kubernetes/pki /etc/kubernetes/
scp -r root@"<IP DEL PRIMER NODO DEL CLUSTER ETCD>":/root/etcd.pem /etc/kubernetes/pki
scp -r root@"<IP DEL PRIMER NODO DEL CLUSTER ETCD>":/root/etcd-key.pem /etc/kubernetes/pki
scp -r root@"<IP DEL PRIMER NODO DEL CLUSTER ETCD>":/root/ca.pem /etc/kubernetes/pki/etcd

```

Se debe reemplazar **xxxx** en **controlPlaneEndpoint** por el valor adecuado del FQDN entregado por el balanceador de masters. 
Una vez que todos los valores estén bien reemplazados, se debe inicializar el primer master:

```sh
kubeadm init --config=./kubeadm-config.yaml --upload-certs | tee kubeadm-init.out
```

***NOTA.- Copiar la instrucción que aparece al final para añadir otros control plane nodes o masters***

Esperar que concluya la inicialización y hacer:

```sh
  rm -rf $HOME/.kube
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

##### 9. Unir mas nodos control plane o masters

En caso de no haber copiado la instrucción para unir mas nodos masters del anterior paso o por expiración del token se debe proceder como sigue.

Para unir mas nodos tipo **control plane** o **masters** se debe proceder de la siguiente manera:

**A**. En el primer master inicializado entrar por ssh y ejecutar:
```sh
kubeadm token create --print-join-command
```
Copiar toda la instrucción generada 

**B**. Continuando en el primer master inicializado hacer:

```sh
kubeadm init phase upload-certs --upload-certs --config=./kubeadm-config.yaml
```

Copiar el valor del certificate key.

**C**. Arrancar el nuevo nodo control plane o master

Entrar al ***nuevo nodo que sera el nuevo control plane o master*** por ssh y ejecutar los pasos de este manual desde el 1 al 7.

Para arrancar el nuevo nodo control plane o master se debe unir el resultado del anterior paso A y añadir al final ***--control-plane --certificate-key*** mas el valor del certificate key copiado en el anterior paso B.


##### 10. Instalar **calico** en el cluster

Para instalar el plugin de red ***calico*** se debe seguir los siguientes pasos:

**A** Descargar el archivo de calico


```sh
curl https://projectcalico.docs.tigera.io/manifests/calico-etcd.yaml -o calico.yaml
```

**B** Se debe modificar el yaml descargado para incluir las llaves e información de ETCD:

En el [archivo de ejemplo](doc/calico.yaml), se debe modificar los siguientes campos:

Se debe incluir las tres llaves haciendo:


Para obtener ***etcd-cert***

```sh
cat /etc/kubernetes/pki/apiserver-kubelet-client.crt | base64 -w 0
```


Para obtener ***etcd-key***

```sh
cat /etc/kubernetes/pki/apiserver-kubelet-client.key | base64 -w 0
```


Para obtener ***etcd-ca***


```sh
cat /etc/kubernetes/pki/etcd/ca.crt | base64 -w 0
```
```sh
cat etcd-key.pem | base64 -w 0
cat etcd.pem | base64 -w 0
cat ca.pem | base64 -w 0
```

Esos valores escribirlos en la siguiente parte del archivo **calico.yaml** como el ejemplo de abajo.

```sh
  etcd-key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBdHFEV3ZuYXFkeTJGOTUvZUx3REdUOFpyZC9QellSZkI0bURVZnlXSnA3RjdueUJzCjJzRGwyeEZWaTl3eXZ5Y0x3WnFMb1owcTB6eXF5ZGhyV2hEWWdCTCtkcHMzSUh5a1VTMDZxTldSellaQ1o5WUIKWi82MWM5NGc1LzZrcHA0ZVM4bU1xekw4OGU4ZFRRMUdFaWp0Uk8rQW9NaGpDQkoyZENoMjRsVUpxSC9MVCtpNgpBaDdNOUkzVkJReEhSY29jZ3V5ek5iUjlwMG5RK3J3eVNJUUdRT2t3VHlodGNZVTNwY001NURaQkdMbTYveDdICjY3R29qaStZVXVYN1RwYXJOcEllaWxIeGR4elM3enZCdmwvekxiM1JkeFBpeXBPUjh3eFZCSGpRNVNMTGhqTWMKbmFNRml5b09TY0VJbEpPNDhKSmxBQm5xbGkvcUxWMVMwckFmclFJREFRQUJBb0lCQUExZ09sbzQxc05qMGl3UQp5WFVuMlY1K2FlQ2ZQWFFmQ1ZSTFEwVU11c2hOZDRCd0g0am1GKyt6bFZCcEVFNXZ6YXlnWlJteEtUSFBmN0xJCjV4UHhwK201ZW1tMWRKUXNqTnhsTTZhcC9jUFAwWTFKWDFEK2xzdWx1VU5FbzBxUXlpZEMyOHF1TVZpRzZ0NTUKMm1mNkYyYTFJL2FpdHA0Z3ZBeEY2bThwUzB2TDhKQSt5d3FyWG5uOGRpNjdxZXMxQkFMNm5CQTVoUDViS010egpwUWRPUitnYWp6VW9rTzVOSi9TT1hrelE3c2pVRTlocXBnNHc3WGlHampWdVNrMTlaSnVQWTU3QmtKQVdiRHVlClh0YmhWZEpaSHczNWpVVTdvMC9HMFppNExDU3RMQmtwS1UzQjF0NXlVQ0pRUFdhWkMzOEhFNDVFdFpVYUtvcGcKNGR6bkNBRUNnWUVBeHlFaXNSTHNham4xdUg1OHdFZCtOR2s1bUYrZ0pJaHcydXQ5UThSYk94cUoza2cvVlpwNQp1K2l3UFcrZS9GNkVaVmxVNE12SXp5UC8rNDVRekR6VitFdWM1UUxMOHZtUVZIcVQ1bythMjlleTkzSW12S3FqCnBrZHNQVEgycHIrT29ETlFmQk1TdnUwZEZ5U09ZUkZrNDlNWVgrUnNuemlLRy9PQTFuelV5RGtDZ1lFQTZzbEMKMzNnOTJEUUptSVBtOVk5cmhZcHB4T3pTWFBlV0JSRWtjK2ZTcGgyMUhmMGI5K2FCdzFQRzR4eTlETDE0RW12WApvejFuWDd5K25QdGgvNEg3ZDQ5d0pnQnJrN3hTcWQzblRHVmZ1R3R1bmVoT3Jya1A1RHdtMkxORWR4MW01cmdtCnd5aXdkSzZncXZPdlFabmhFVVZ5TThHaTJWbGsrS1o2allVclN4VUNnWUFWVUlldEdwQnh3bWg1NmhnaVlNU3kKaVh6ZndZU2J4SHNJQStMeHFRZjI2SjFQVEw1eXhFazVndXV5ZDhzMXlrd3pxUDg3M0xSTzc5U0xzYTBXWDRDcgp4alF5RXoyUGNZVXdkYnAxR0hRRUNpK2U2dm9ZZ2M5b2tnYVUrazhqaENlWklFVUNNdXh6d1YrMnhYUDBFZStSCnIxdlJqOXJNcERtc1NrRkZOREYyaVFLQmdRQ2lVcTlYVFF5RTg4VkdtcnNOUHpENVRLNi9wWFB6TG9HYjB6USsKcGlJdkV3N3JRdGtaVlZhVnN0QW9xTy9UWlJNa3VVYUc1NmNXdTZtVll2OW41WGYwTzBrd0hNNURmOG92QXVvdApHVkZLY1l3eXhDL1NBTVNKNlVSNlFjYXVDN2ZlLzZaYyt6NjBEUityMFhwemdtM213UHFwNmRBck1QRHNNRDArCnByazkyUUtCZ1FERktnOCtuNmltb3JIeEJzU3VqbEx0U01aZitNUDFwZm9Lb1pTT3daOFlrWEdWV2NjZGp5K1kKbUp3a0FNcHM4QjJweEg1ZFhJZXFtNXpNVFFxOTBocjNvWFQvRkJoVWdzeEZhTEoxZHZnUzk2TzNPMmR6REQ2MAphdVR4eDVKSUg0NlVqOHIxWWR5b2hWZ01PNUMrQnpYb1VmSWk4Wml2c3p4SkJCanNuZzJNZnc9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
  etcd-cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURLRENDQWhDZ0F3SUJBZ0lJSElNZDJNUDJydW93RFFZSktvWklodmNOQVFFTEJRQXdFakVRTUE0R0ExVUUKQXhNSFpYUmpaQzFqWVRBZUZ3MHlNakExTVRZeE9USXlORGhhRncweU16QTFNVFl4T1RJMU16TmFNRDR4RnpBVgpCZ05WQkFvVERuTjVjM1JsYlRwdFlYTjBaWEp6TVNNd0lRWURWUVFERXhwcmRXSmxMV0Z3YVhObGNuWmxjaTFsCmRHTmtMV05zYVdWdWREQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQUxhZzFyNTIKcW5jdGhmZWYzaThBeGsvR2EzZno4MkVYd2VKZzFIOGxpYWV4ZTU4Z2JOckE1ZHNSVll2Y01yOG5DOEdhaTZHZApLdE04cXNuWWExb1EySUFTL25hYk55QjhwRkV0T3FqVmtjMkdRbWZXQVdmK3RYUGVJT2YrcEthZUhrdkpqS3N5Ci9QSHZIVTBOUmhJbzdVVHZnS0RJWXdnU2RuUW9kdUpWQ2FoL3kwL291Z0llelBTTjFRVU1SMFhLSElMc3N6VzAKZmFkSjBQcThNa2lFQmtEcE1FOG9iWEdGTjZYRE9lUTJRUmk1dXY4ZXgrdXhxSTR2bUZMbCswNldxemFTSG9wUgo4WGNjMHU4N3diNWY4eTI5MFhjVDRzcVRrZk1NVlFSNDBPVWl5NFl6SEoyakJZc3FEa25CQ0pTVHVQQ1NaUUFaCjZwWXY2aTFkVXRLd0g2MENBd0VBQWFOV01GUXdEZ1lEVlIwUEFRSC9CQVFEQWdXZ01CTUdBMVVkSlFRTU1Bb0cKQ0NzR0FRVUZCd01DTUF3R0ExVWRFd0VCL3dRQ01BQXdId1lEVlIwakJCZ3dGb0FVT3lpUVlWVitOUjBjZEFRRQpMeDlKbDZTalV0a3dEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBSlFKc1AzN0VrY3FQbkhWQVpVYThmcGVIUkJTCkNuYnJncThDR3Bxa1FUeGlEeGVrMTU2OEtQbzczQ0pTRDdnR0JBMjdiWWRkR3RtNHZFeGhINXNRR09GTThESjYKK1YzUUNSd2YrS1hhWlRibXlYWisraUZ4a2R4cU9paVVnRjh6TmEzR3lZMHhqeFlGWUpsR2c4ZUowVUFXamdwbgp3aW9KSWRhSlVyTWFRK3BmRFVvSmJEbm5BRUNQb3JwZnc0elRuUTdnMEdKUGMwci9HOGdqc1oralBhU29uNVA2CmVBd0Vxc0Yrd3VaUFdKZ3BSR0phRVNRWW9TYmUvamthUk9nVzZFZnNLVnlhcmFXVkNaNDZOTklkd2lqMTY2UHAKMytYMHY2aVJvMTBxWjRwQ01nZWZFZGkxM0RhVGpHNTFtVGtZekxGaTFIQ3ZKNmM0SGU3QmpGRTUrV0k9Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
  etcd-ca: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM5VENDQWQyZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFTTVJBd0RnWURWUVFERXdkbGRHTmsKTFdOaE1CNFhEVEl5TURVeE5qRTVNakkwT0ZvWERUTXlNRFV4TXpFNU1qSTBPRm93RWpFUU1BNEdBMVVFQXhNSApaWFJqWkMxallUQ0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQU1RcktIRHRHNDgyCndPTWU5cUxVNmF3eklJNWVWbkpMYXlvbmI4Z3QzWFU4MTRSUUVmQkZ4ckw3NVloL2JzUytjNzB2dVJDa013MXkKR2RDbmJRWmZuRzR5blg4N2k4UEg1b0UrSWdqYVFYdWlIQWFEYWxRT2ZXL3lPTThNdlVINStPN1h6WUNIVTlNMwpYYXBUeU9lTE1wMnBNMm1INGR3ekE4Mkk3WGlIVmo2cjF0LzlMdXJFSHVnZGk0MnlwVU1zRE1tWmRIUnUvRlNvCi91K3VQbG1DdSs2cmhXczk4dzAwdm9iWDRhd2RhR2tQd3NwSk1HOExUaUN2K3F0Y1UwTU9ab1AvaloxS3NBdlYKUFFjdlhCbFRFU2pWOHNrNVY3dE5OeWJ4STNDODJpV1VvVkxtZmo3WmRGTE1xV2hyNFB0ci9NNU5YN2NKYlMxRgpmajBmRk5sUUhCY0NBd0VBQWFOV01GUXdEZ1lEVlIwUEFRSC9CQVFEQWdLa01BOEdBMVVkRXdFQi93UUZNQU1CCkFmOHdIUVlEVlIwT0JCWUVGRHNva0dGVmZqVWRISFFFQkM4ZlNaZWtvMUxaTUJJR0ExVWRFUVFMTUFtQ0IyVjAKWTJRdFkyRXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBR3dZZm9aNEtFUGtzVnE2dXNBZStpWEZTRG45NUNXegp2a0tUNUxwTWJQWU9SYVl1ZGUrZFowWkZMbzFDMGoreitKa05rUkZHU1BjUlZma3lUMjlNZGVxV0w2WnZBdUdhCkZCT2hPNVZpM3VHdHF6eDhBRnJyQ2hTcllzQmJCaVlEakFTdjZIL3hpOERrUFNEeHBGRVZtaEI2dFNCOFNucWQKUjNQbEQ4ZUNkeEJDdVY1WjBtMS9Va0xXVCtlMTVDVzdLZzYyM04vV25MK0dvNFFFeEsvc3JUYysxNzdhQWFQUwpwRkNOWDR6OWRiMTBwSWIxSzFTT3NCeDNTWW1ROWVCbytqME5WK1JybXBxZExOUDQ1K2k4Sis5NjRYVlRDb0JFCkVZb3FMWVp0cUptN3ByRGIzMDc0WGV1Y0NpeXVzREF3eTE5eHhKR0hkSHBHY0VCNHVwaGwwaW89Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K 

```

Luego dejar la siguiente parte del archivo de esta forma:



```sh
  etcd_ca: "/calico-secrets/etcd-ca"
  etcd_cert: "/calico-secrets/etcd-cert"
  etcd_key: "/calico-secrets/etcd-key"
```

Solo modificar esas dos partes del archivo ***calico.yaml*** mas las direcciones IP del cluster ETCD:

```sh
data:
  etcd_endpoints: "https://172.19.11.39:2379,https://172.19.11.40:2379,https://172.19.11.41:2379"
```

Después de eso dar:

```sh
kubectl create -f calico.yaml
```

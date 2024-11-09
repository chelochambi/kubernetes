# Manual de renovación de certificados en Kubernetes


Para la renovación de los certificados de kubernetes de un cluster version 1.22 se deben de seguir los siguientes pasos:


##### 1. Realizar backup de los certificados antiguos

```sh
 mkdir /etc/kubernetes.bak
sudo cp -r /etc/kubernetes/pki/ /etc/kubernetes.bak
sudo cp /etc/kubernetes/*.conf /etc/kubernetes.bak
```


##### 2. Verificar la expiración:

```sh
kubeadm certs check-expiration
```

##### 3. Renovar todos los certificados

```sh
kubeadm certs renew all --config=kubeadm-config.yaml
```


##### 4. Renovar el archivo ***kubeconfig***

```sh
kubeadm init phase kubeconfig all --config kubeadm-config.yaml
```



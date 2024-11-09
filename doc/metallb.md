## Manual de instalación y configuración de MetalLB 

Para instalar el balanceador MetalLB en un clúster de Kubernetes, se deben seguir estos dos pasos:
Primero dentro el ***nodo master*** mediante ssh, ejecutar:

```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
```

Posteriormente continuar:

```sh
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
```
Con esos pasos queda instalado MetalLB


## Manual de configuración de MetalLB

Para configurar MetalLB para usar las **IP virtuales** que se usaran para crear servicios tipo ***Load Balancer*** con **IP externo**, se debe crear un ConfigMap con los valores de estas IP:

```sh
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.240-192.168.1.250
```
NOTA.- Las IP mostradas aquí son un ejemplo. Este tipo de IP virtuales se utilizan para crear el servicio de exposición externa y no pertenecen a un servidor real

Posteriormente, incluir este **ConfigMap**

```sh
kubectl create -f config.yml
```

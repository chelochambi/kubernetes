# Manual de instalación de Hashicorp Consul para cluster de Kubernetes


Para instalar Hashicorp Consul en alta disponibilidad en conjunto con Consul para el almacenamiento en un cluster de Kubernetes se debe cumplir los siguientes requisitos:

- Helm instalado [Instrucciones](/doc/helm.md)


##### 1. Instalar el repo de Consul

```sh
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```
##### 2. Instalar Consul


Instalar el chart de Consul:

```sh
helm install consul hashicorp/consul --values helm-consul-values.yml
```
El archivo helm-consul-values.yml contienes los siguientes valores:

```sh
global:
  datacenter: vault-kubernetes-tutorial

client:
  enabled: true

server:
  replicas: 3
  bootstrapExpect: 1
  disruptionBudget:
    maxUnavailable: 0
```

# Manual de instalación de Helm

Para la instalación de Helm se debe seguir los siguientes pasos en el ***nodo master*** de Kubernetes sobre el cual se desea tener instalada esta herramienta.

Ingresando mediante ssh al nodo master ejecutar:

```sh
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
./get_helm.sh
```

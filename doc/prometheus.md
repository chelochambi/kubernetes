# Manual de instalación de Prometheus y Grafana en un cluster de Kubernetes

Para instalar ***Prometheus*** y ***Grafana*** en modo stack se debe cumplir los siguientes requisitos:

- Se debe tener [Helm](doc/helm.md) instalado

##### 1. Instalar el operador de Prometheus

```sh
kubectl create ns monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm upgrade --namespace monitoring --install kube-stack-prometheus prometheus-community/kube-prometheus-stack --set prometheus-node-exporter.hostRootFsMount.enabled=false
```

Para exponer el despliegue de Prometheus se debe hacer en el nodo master:

```sh
kubectl port-forward --namespace monitoring svc/kube-stack-prometheus-kube-prometheus 9090:9090
```

##### 2. Verificar el visualizador Grafana

Para obtener las credenciales de ***usuario*** y ***password*** por defecto de **admin** de Grafana:

```sh
kubectl get secret --namespace monitoring kube-stack-prometheus-grafana -o jsonpath='{.data.admin-user}' | base64 -d
kubectl get secret --namespace monitoring kube-stack-prometheus-grafana -o jsonpath='{.data.admin-password}' | base64 -d
```

Exponer el visualizador de Grafana externamente:

```sh
kubectl port-forward --namespace monitoring svc/kube-stack-prometheus-grafana 8080:80
```

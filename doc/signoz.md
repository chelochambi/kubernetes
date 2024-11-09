# Manual de instalación de SIGNOZ y Opentelémetry
La instalación se divide en 3 partes:
1. Instalación de cliente SigNoz en cluster Tools
2. Instalación de signoz-infra-metrics en cluster a ser monitoreado
3. Instalación de opentelemetry-operator en cluster a ser modificado
   - Crear componente instrumentación
   - Crear OTEL collector
   
## 1. Instalación de cliente SigNoz en cluster Tools
Pre requisitos
- Instalación de HELM

Crear name space platform
```sh
kubectl create ns platform
```
Crear o actualización de un Storage Classes. Aplicar archivo [storage-class-signoz.yaml](http://dev.impuestos.gob.bo/00-gestion-did/investigacion/instalacion-cluster-kubernetes/-/blob/master/doc/archivosYaml/storage-class-signoz.yaml)
```sh
kubectl create -f storage-class-signoz.yaml
```
Habilitar la expanción de storage de forma automática
```sh
kubectl patch storageclass local-storage -p '{"allowVolumeExpansion": true}'
```
Crear directorios en los nodos worker
```sh
mkdir -p /var/lib/signoz/pv1
mkdir -p /var/lib/signoz/pv2
mkdir -p /var/lib/signoz/pv3
mkdir -p /var/lib/signoz/pv4

chmod 777 -R /var/lib/signoz/pv1
chmod 777 -R /var/lib/signoz/pv2
chmod 777 -R /var/lib/signoz/pv3
chmod 777 -R /var/lib/signoz/pv4
```
Crear Persistent Volumes. Aplicar archivo [persistentVolume-signoz.yaml](http://dev.impuestos.gob.bo/00-gestion-did/investigacion/instalacion-cluster-kubernetes/-/blob/master/doc/archivosYaml/persistentVolume-signoz.yaml)
```sh
kubectl create -f persistentVolume-signoz.yaml
```
Adcionar paquete helm signoz
```sh
helm repo add signoz https://charts.signoz.io
```
Instalar SigNoz mediante helm. Utilizar archivo [values-signoz-chart-helm.yaml](http://dev.impuestos.gob.bo/00-gestion-did/investigacion/instalacion-cluster-kubernetes/-/blob/master/doc/archivosYaml/values-signoz-chart-helm.yaml)
```sh
helm --namespace platform install my-release signoz/signoz -f values-signoz-chart-helm.yaml
```
## 2. Instalación de signoz-infra-metrics en cluster a ser monitoreado
Esta instalación permite instalar egentes en nodos worker a ser monitoreados
Descargar los archivos de instalación (si no se tiene los archivos ya modificados)
```sh
git clone https://github.com/SigNoz/otel-collector-k8s.git && cd otel-collector-k8s
```
Modificar archivos
```sh
nano agent/infra-metrics.yaml
nano deployment/all-in-one.yaml
```
Modificar el endpoint de cada archivo
```sh
#--------------------------
exporters:
   otlp:
      endpoint: "<SigNoz-Otel-Collector-Address>:39007"
      tls:
         insecure: true
#--------------------------
```
Crear namespace y aplicar archivos yaml
```sh
kubectl create ns signoz-infra-metrics
kubectl -n signoz-infra-metrics apply -Rf agent
kubectl -n signoz-infra-metrics apply -Rf deployment
```
Para crear archivos json para los dashboards, ejecutar lo siguiente en un master
```sh
mkdir -p /opt/kubernetes/configuraciones/monitoreo/signoz/otel-collector-k8s/archivosJson
cd /opt/kubernetes/configuraciones/monitoreo/signoz/otel-collector-k8s/archivosJson
```
```sh
for node in $(kubectl get nodes -o name | sed -e "s/^node\///");
do
  curl -sL https://github.com/SigNoz/benchmark/raw/main/dashboards/hostmetrics/hostmetrics-import.sh \
    | HOSTNAME="$node" DASHBOARD_TITLE="Dashboard para el nodo $node" bash
done
```
## 3. Instalación de opentelemetry-operator en cluster a ser modificado
- Requisito: cert-manager

Instalar cert-manager. Aplicar el archivo [cert-manager.yaml](http://dev.impuestos.gob.bo/00-gestion-did/investigacion/instalacion-cluster-kubernetes/-/blob/master/doc/archivosYaml/cert-manager.yaml)
```sh
kubectl apply -f cert-manager.yaml
```
Instalar el OTEL-operador. Aplicar el archivo [opentelemetry-operator.yaml](http://dev.impuestos.gob.bo/00-gestion-did/investigacion/instalacion-cluster-kubernetes/-/blob/master/doc/archivosYaml/opentelemetry-operator.yaml)
```sh
kubectl apply -f opentelemetry-operator.yaml
```
Crear componente OTEL-instrumentación. Aplicar archivo [crearinstrumentación.yaml](http://dev.impuestos.gob.bo/00-gestion-did/investigacion/instalacion-cluster-kubernetes/-/blob/master/doc/archivosYaml/crearinstrumentaci%C3%B3n.yaml)
```sh
kubectl apply -f crearinstrumentación.yaml
```
Crear componente OTEL-Collector. Aplicar archivo [creatOpenTelemetryCollector.yaml](http://dev.impuestos.gob.bo/00-gestion-did/investigacion/instalacion-cluster-kubernetes/-/blob/master/doc/archivosYaml/creatOpenTelemetryCollector.yaml)
```sh
kubectl apply -f creatOpenTelemetryCollector.yaml
```
Deplocyar nuevamente los pods a ser inyectados por OTEL. Verificar que cada pod tenga dos contenedores.

#Manual de instalación de Vault

Para instalar Hashicorp Vault en alta disponibilidad en conjunto con Consul para el almacenamiento en un cluster de Kubernetes se debe cumplir los siguientes requisitos:

- Helm instalado [Instrucciones](/doc/helm.md)

##### 1. Configuración de volumen persistente
Crear carpeta de configuración
```sh
mkdir -p /opt/kubernetes/configuraciones/vault
```
Crear StorageClass `vault-sc.yaml`
```sh
cat <<EOF | sudo tee /opt/kubernetes/configuraciones/vault/vault-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
EOF
```
Crear pv (persistent volume) `vault-pv.yaml`
```sh
cat <<EOF | sudo tee /opt/kubernetes/configuraciones/vault/vault-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-consul
  namespace: vault
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 10Gi
  hostPath:
    path: /mnt
  storageClassName: local-storage
  volumeMode: Filesystem
EOF
```
Crear pvc (persistent volume clean) `vault-pvc.yaml`
```sh
cat <<EOF | sudo tee /opt/kubernetes/configuraciones/vault/vault-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: consul
    chart: consul-helm
    release: consul
  name: data-vault-consul-consul-server-0
  namespace: vault
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF
```
Habilitar permisos de escritura carpeta `/mnt`
```sh
NOTA: estos permisos se los debe realizar en todos los worker
chmod 777 /mnt
```
Aplicar manifiestos para la creiación de sc, vc y pvc
```sh
kubectl apply -f vault-sc.yaml
kubectl apply -f vault-pv.yaml
kubectl apply -f vault-pvc.yaml
```
##### 2. Crear namespace "vault"
Crear el namespace de Vault:
```sh
kubectl create ns vault
```
##### 3. Adicionar repo de Hashicorp
Crear el namespace de Vault:
```sh
helm repo add hashicorp https://helm.releases.hashicorp.com

helm repo update
```
##### 4. Instalar Consul
Crear archivo helm-consul-values.yml de configuración:
```sh
cat <<EOF | sudo tee /opt/kubernetes/configuraciones/vault/helm-consul-values.yml
global:
  datacenter: vault-kubernetes-tutorial

client:
  enabled: true
  nodeSelector: |
    node: monitoring
server:
  replicas: 1
  bootstrapExpect: 1
  disruptionBudget:
    maxUnavailable: 0
EOF
```
Instalar el chart de consul:
```sh
helm install consul hashicorp/consul --values helm-consul-values.yml -n vault 
```

##### 5. Instalar Vault
Crear archivo helm-vault-values.yml de configuración:
```sh
cat <<EOF | sudo tee /opt/kubernetes/configuraciones/vault/helm-vault-values.yml
server:
  affinity: ""
  ha:
    enabled: true
    replicas: 3
  nodeSelector: |
        node: monitoring
  service:
    enabled: true
    type: NodePort
    nodePort: 39001
    port: 8200
    targetPort: 8200
    annotations: {}
EOF
```

Instalar el chart de Vault:
```sh
helm install vault hashicorp/vault --values helm-vault-values.yml -n vault --version 0.21.0
```


##### 6. Verificar el estado de vault
Se verificará el estado del pod vault-0:
```sh
kubectl exec vault-0 -n vault -- vault status
```

##### 7. Inicializar vault
Generar archivo con la configuración principal
```sh
kubectl exec vault-0 -n vault -- vault operator init -key-shares=1 -key-threshold=1 -format=json > cluster-keys.json
```
Obtener llave principal del cluster 
```sh
cat cluster-keys.json | jq -r ".unseal_keys_b64[]"
```
Almacenar de forma momentanea la llave principal e iniciar los pods creados
```sh
VAULT_UNSEAL_KEY=$(cat cluster-keys.json | jq -r ".unseal_keys_b64[]")
```
Abrir todos los pods con la llave principal
```sh
kubectl exec vault-0 -n vault -- vault operator unseal $VAULT_UNSEAL_KEY
kubectl exec vault-1 -n vault -- vault operator unseal $VAULT_UNSEAL_KEY
kubectl exec vault-2 -n vault -- vault operator unseal $VAULT_UNSEAL_KEY
```
Obtener TOKEN root
```sh
cat cluster-keys.json | jq -r ".root_token"
```
##### 8. Comandos básicos
Ver secretos versión 1
```sh
curl \
--header "X-Vault-Token: <Ingresar token>" \
http://<dirección servico de vault>/v1/secreto/path
```
Ver secretos versión 2
```sh
curl \
    --header "X-Vault-Token: <Ingresar token>" \
    http://<dirección servico de vault>/v1/secreto/data/path
```

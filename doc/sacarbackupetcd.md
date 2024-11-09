# Manual para sacar backup a un cluster ETCD

Para sacar un backup al cluster de ETCD, se debe ingresar por ssh a uno de los nodos del ETCD y proceder de acuerdo a los siguientes pasos:

1. Determinar la carpeta de datos del ETCD

```sh
sudo grep data-dir /etc/kubernetes/manifests/etcd.yaml
```

2. Verificar la salud del cluster ETCD

```sh
ETCDCTL_API=3 etcdctl --cert=./peer.crt --key=./peer.key --cacert=./ca.crt --endpoints=https://127.0.0.1:2379 endpoint health
```
Determinar que el miembro del cual se sacará el backup esté con buena salud.

3. Sacar el backup

Se debe sacar el backup de la siguiente forma:

```sh
cd /etc/kubernetes/pki/etcd
ETCDCTL_API=3 etcdctl --cert=./peer.crt --key=./peer.key --cacert=./ca.crt --endpoints=https://127.0.0.1:2379 snapshot save /var/lib/etcd/snapshot.db
```
4. Verificar la creación del archivo:

```sh
 ls -l /var/lib/etcd/
```
5. Guardar el **backup** en un lugar seguro de almacenamiento


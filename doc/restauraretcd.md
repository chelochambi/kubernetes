# Manual de restauración del cluster ETCD

Para restaurar el cluster desde un **snapshot** creado para hacer backup se deben seguir los siguientes paso, haciendo ssh en un nodo miembro del ETCD:

1. Verificar la existencia del archivo ***.db*** de backup:

```sh
ls -l /var/lib/etcd/
```

2. Detener cada una de las instancias de **apiserver** en el cluster, es decir en cada master node haciendo ssh ejecutar:

```sh
mv /etc/kubernetes/manifests/kube-apiserver-s-nal-pp-km-0X ../
kubectl delete kube-apiserver-s-nal-pp-km-0X
```
NOTA.- el valor 0X debe ser reemplazado con el número de master al cual se busca detener el **apiserver**

3. Restaurar el **backup** en cada uno de los miembros del ETCD, haciendo ssh en cada uno de los miemberos ETCD hacer:

```sh
cd /etc/kubernetes/pki/etcd
ETCDCTL_API=3 etcdctl --cert=./peer.crt --key=./peer.key --cacert=./ca.crt  --endpoints=https://[127.0.0.1]:2379 --data-dir=/var/lib/etcd-backup snapshot restore /var/lib/etcd/snapshot.db

```

4. En el archivo **/etc/kubernetes/manifests/etcd.yaml*** cambiar el siguiente valor en cado uno de los nodos miembros del cluster ETCD:

```sh
--data-dir=/var/lib/etcd-from-backup
```
5. Regresar los archivos de manifiesto del apiserver a su lugar en cada ***nodo master***:

```sh
mv /etc/kubernetes/kube-apiserver-s-nal-pp-km-0X /etc/kubernetes/manifests/ 

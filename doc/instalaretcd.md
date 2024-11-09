## Instalar ETCD


Primero se debe instalar ***etcd*** en uno de los nodos que forma parte del cluster de ETCD. Haciendo ssh en uno de los nodos, ejecutar uno a uno los siguientes comandos:

```sh
ETCD_VER=v3.5.4
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GITHUB_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

sudo mv /tmp/etcd-download-test/etcdctl /usr/bin/
```

Verificar la instalación mediante la siguiente instrucción:

```sh
etcdctl version
```

# Manual de instalación de ETCD externo

##### 1. Modificar archivo /etc/hosts y adicionar lista de nodos:
```sh
nano /etc/hosts

ej.:
<IPs NODOS1> <NOMBRE DE HOST1> <NOMBRE DE HOST1>.impuestos.gob.bo
<IPs NODOS2> <NOMBRE DE HOST2> <NOMBRE DE HOST2>.impuestos.gob.bo
....
<IPs NODOSN> <NOMBRE DE HOSTN> <NOMBRE DE HOSTN>.impuestos.gob.bo
```
##### 2. Condigurar el firewall
```sh
firewall-cmd --add-service=https
firewall-cmd --add-service=https --permanent
sudo firewall-cmd --add-port={2379,2380}/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-port
```
##### 3. Deshabilitar las configuraciones de seguridad

```sh
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
``` 
##### 4. Instalar wget y curl
```sh
sudo yum -y install wget curl
```
##### 5. Instalar ETCD
```sh
mkdir /tmp/etcd && cd /tmp/etcd

rm -rf etcd*

ETCD_VER=v3.5.4

curl -L https://storage.googleapis.com/etcd/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz \
 -o /tmp/etcd/etcd-${ETCD_VER}-linux-amd64.tar.gz

tar xvf etcd-v*.tar.gz

cd etcd-*/

sudo mv etcd* /usr/local/bin/

cd ..

rm -rf etcd*
```
##### 6. Verificar versiones
```sh
etcd --version
etcdctl version
etcdutl version
```
##### 7. Instalación de cfssl SOLO EN EL NODO 1
Descargar y habilitar archivo json
```sh
cd /root

wget -q --show-progress \
    https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
    https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson

chmod +x cfssl cfssljson

sudo mv cfssl cfssljson /usr/bin
```
##### 8. Crear variables de entorno en TODOS LOS NODOS
```sh
HOST1=<IP NODO ETC 1>
HOST2=<IP NODO ETC 2>
HOST3=<IP NODO ETC 3>
HOST_NAME1=<HOST NAME DEL NODO 1>
HOST_NAME2=<HOST NAME DEL NODO 1>
HOST_NAME3=<HOST NAME DEL NODO 1>
```

##### 9. Creación de archivos json SOLO EN EL NODO 1
Creación de archivo ca-config.json
```sh
cat <<EOF | sudo tee /root/ca-config.json
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "etcd": {
                "expiry": "87600h",
                "usages": ["signing","key encipherment","server auth","client auth"]
            }
        }
    }
}
EOF
```
Creación de archivo ca-csr.json
```sh
cat <<EOF | sudo tee /root/ca-csr.json
{
  "CN": "etcd cluster",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "ID",
      "L": "Indonesia",
      "O": "Kubernetes",
      "OU": "ETCD-CA",
      "ST": "West Java"
    }
  ]
}
EOF
```
Creación de archivo etcd-csr.json
```sh
cat <<EOF | sudo tee /root/etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "localhost",
    "127.0.0.1",
    "$HOST1", 
    "$HOST2",
    "$HOST3" 
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "ID",
      "L": "Indonesia",
      "O": "Kubernetes",
      "OU": "ETCD",
      "ST": "West Java"
    }
  ]
}
EOF
```
##### 10. Crear certidicados en base a los archivos json SOLO EN EL NODO 1
```sh
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd etcd-csr.json | cfssljson -bare etcd
```
##### 11. Crear carpetas para los certificados en TODOS LOS NODOS
```sh
sudo mkdir -p /var/lib/etcd/
sudo mkdir /etc/etcd
sudo chmod -R 700 /var/lib/etcd/
sudo mkdir /var/lib/etcd/ssl
```

##### 12. Copiar certificados. Realizar el proceso SOLO EN EL NODO 1
```sh
cd /root
cp *.pem /var/lib/etcd/ssl/
scp *.pem root@$HOST2:/var/lib/etcd/ssl/
scp *.pem root@$HOST3:/var/lib/etcd/ssl/
```
##### 13. Crear variables de entorno para configurar ETCD

**->> Proceso para el nodo 1**
```sh
cat <<EOF | sudo tee /var/lib/etcd/etcd
ETCD_NAME=$HOST_NAME1
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_CLIENT_URLS=https://$HOST1:2379,https://127.0.0.1:2379
ETCD_LISTEN_PEER_URLS=https://$HOST1:2380
ETCD_ADVERTISE_CLIENT_URLS=https://$HOST1:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://$HOST1:2380
ETCD_INITIAL_CLUSTER=$HOST_NAME1=https://$HOST1:2380,$HOST_NAME2=https://$HOST2:2380,$HOST_NAME3=https://$HOST3:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster-0
ETCD_CLIENT_CERT_AUTH=true
ETCD_TRUSTED_CA_FILE=/var/lib/etcd/ssl/ca.pem        
ETCD_CERT_FILE=/var/lib/etcd/ssl/etcd.pem            
ETCD_KEY_FILE=/var/lib/etcd/ssl/etcd-key.pem         
ETCD_PEER_CLIENT_CERT_AUTH=true
ETCD_PEER_TRUSTED_CA_FILE=/var/lib/etcd/ssl/ca.pem   
ETCD_PEER_CERT_FILE=/var/lib/etcd/ssl/etcd.pem       
ETCD_PEER_KEY_FILE=/var/lib/etcd/ssl/etcd-key.pem
EOF
```
**->> Proceso para el nodo 2**
```sh
cat <<EOF | sudo tee /var/lib/etcd/etcd
ETCD_NAME=$HOST_NAME2
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_CLIENT_URLS=https://$HOST2:2379,https://127.0.0.1:2379
ETCD_LISTEN_PEER_URLS=https://$HOST2:2380
ETCD_ADVERTISE_CLIENT_URLS=https://$HOST2:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://$HOST2:2380
ETCD_INITIAL_CLUSTER=$HOST_NAME1=https://$HOST1:2380,$HOST_NAME2=https://$HOST2:2380,$HOST_NAME3=https://$HOST3:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster-0
ETCD_CLIENT_CERT_AUTH=true
ETCD_TRUSTED_CA_FILE=/var/lib/etcd/ssl/ca.pem
ETCD_CERT_FILE=/var/lib/etcd/ssl/etcd.pem         
ETCD_KEY_FILE=/var/lib/etcd/ssl/etcd-key.pem
ETCD_PEER_CLIENT_CERT_AUTH=true
ETCD_PEER_TRUSTED_CA_FILE=/var/lib/etcd/ssl/ca.pem
ETCD_PEER_CERT_FILE=/var/lib/etcd/ssl/etcd.pem    
ETCD_PEER_KEY_FILE=/var/lib/etcd/ssl/etcd-key.pem
EOF
```
**->> Proceso para el nodo 3**
```sh
cat <<EOF | sudo tee /var/lib/etcd/etcd
ETCD_NAME=$HOST_NAME3
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_CLIENT_URLS=https://$HOST3:2379,https://127.0.0.1:2379
ETCD_LISTEN_PEER_URLS=https://$HOST3:2380
ETCD_ADVERTISE_CLIENT_URLS=https://$HOST3:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://$HOST3:2380
ETCD_INITIAL_CLUSTER=$HOST_NAME1=https://$HOST1:2380,$HOST_NAME2=https://$HOST2:2380,$HOST_NAME3=https://$HOST3:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster-0
ETCD_CLIENT_CERT_AUTH=true
ETCD_TRUSTED_CA_FILE=/var/lib/etcd/ssl/ca.pem
ETCD_CERT_FILE=/var/lib/etcd/ssl/etcd.pem         
ETCD_KEY_FILE=/var/lib/etcd/ssl/etcd-key.pem
ETCD_PEER_CLIENT_CERT_AUTH=true
ETCD_PEER_TRUSTED_CA_FILE=/var/lib/etcd/ssl/ca.pem
ETCD_PEER_CERT_FILE=/var/lib/etcd/ssl/etcd.pem    
ETCD_PEER_KEY_FILE=/var/lib/etcd/ssl/etcd-key.pem
EOF
```
##### 14. Crear servicio ETCD - TODOS LOS NODOS
```sh
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd

[Service]
Type=notify
User=root
EnvironmentFile=/var/lib/etcd/etcd
ExecStart=/usr/local/bin/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
##### 15. Iniciar servicios - TODOS LOS NODOS
```sh
systemctl daemon-reload
systemctl enable etcd

systemctl start etcd
systemctl status etcd
```
##### 16. Verificación cluster etcd
```sh
etcdctl --endpoints=https://$HOST1:2379 --cacert=ca.pem --cert=etcd.pem --key=etcd-key.pem member list
etcdctl --endpoints=https://$HOST1:2379 --cacert=ca.pem --cert=etcd.pem --key=etcd-key.pem endpoint health --cluster -w table
etcdctl --endpoints=https://$HOST1:2379 --cacert=ca.pem --cert=etcd.pem --key=etcd-key.pem endpoint status --cluster -w table
```

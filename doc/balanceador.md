# Manual del balanceador con proxy reverso


Una alternativa a utilizar una ***IP virtual*** en el balanceo de los nodos masters, es utilizar un proxy reverso como **haproxy**


##### 1. Instalando haproxy

Para instalar ***haproxy*** en un servidor, se debe ingresar por ssh al servidor y ejecutar:

```sh
sudo dnf -y install haproxy
```

El servicio quedará levantado y ahora se debe configurar.


##### 2. Configurando haproxy

Para configurar el servicio haproxy en la ubicación /etc/haproxy/haproxy.cfg

Se debe editar el archivo para que quede como a continuación:


Recordando que en la parte final se debe configurar los nombres de dominio e IP de los nodos maestros.


```sh

#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    tcp
    log                     global
    option                  tcplog
    # option                  dontlognull
    # option http-server-close
    # option forwardfor       except 127.0.0.0/8
    # option                  redispatch
    # retries                 3
    # timeout http-request    10s
    # timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    # timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend proxynode
    # acl url_static       path_beg       -i /static /images /javascript /stylesheets
    # acl url_static       path_end       -i .jpg .gif .png .css .js
    bind *:80
    bind *:6443
    # stats uri /proxystats
    default_backend k8sServers
    
    # acl ACL_k8sm.impuestos.gob.bo hdr(host) -i k8sm.impuestos.gob.bo 
    # use_backend myServers         if ACL_k8sm.impuestos.gob.bo
    # default_backend             app

#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
# backend static
#    balance     roundrobin
#    server      static 127.0.0.1:4331 check

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend k8sServers
    balance     roundrobin
    server s-nal-pp-km-01 172.19.11.33:6443 check
    server s-nal-pp-km-02 172.19.11.34:6443 check
    # server s-nal-ppke-06 172.19.11.245:6443 check
    server  app3 127.0.0.1:5003 check
    server  app4 127.0.0.1:5004 check


listen stats
    bind :9999
    mode http
    stats enable
    stats hide-version
    stats uri /stats

```
Deshabilitar SELinux

```sh
sudo nano /etc/sysconfig/selinux
```
```sh
# disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of these two values:
```

Una vez configurado el archivo, ejecutar en el server

```sh
setsebool -P haproxy_connect_any=1
sudo systemctl daemon-reload
sudo systemctl restart haproxy
```
Habilitamos el puerto 6443 en firewalld 

```sh
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --reload
firewall-cmd --list-ports
```

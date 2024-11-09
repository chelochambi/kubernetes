# Manual de instalación de gitlab-runner en un cluster de Kubernetes 

Requisitos:

- Se debe tener Helm instalado en un nodo master

Dentro el nodo master mediante ssh

Utilizando Helm, se debe incluir el repositorio de gitlab-runner:

```sh
helm repo add gitlab https://charts.gitlab.io
```

Luego instalar el chart de gitlab-runner, mediante:

```sh
kubectl create ns gitlab-runner
helm upgrade --install -n gitlab-runner -f valores.yml gitlab/gitlab-runner
```
El archivo [valores.yml](doc/valores.yaml) debe contener el **token** generado por la instancia de Gitlab y el nombre de dominio de la instancia de gitlab a la cual se conectara el gitlab-runner.

En el [ejemplo](doc/valores.yaml) de archivo, se incluyen los valores.


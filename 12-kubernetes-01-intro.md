# Домашнее задание к занятию "12.1 Компоненты Kubernetes"

Вы DevOps инженер в крупной компании с большим парком сервисов. Ваша задача — разворачивать эти продукты в корпоративном кластере. 

## Задача 1: Установить Minikube

Для экспериментов и валидации ваших решений вам нужно подготовить тестовую среду для работы с Kubernetes. Оптимальное решение — развернуть на рабочей машине Minikube.

- установите миникуб и докер следующими командами:
  - curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
  - chmod +x ./kubectl
  - sudo mv ./kubectl /usr/local/bin/kubectl
  - sudo apt-get update && sudo apt-get install docker.io conntrack -y
  - curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
- проверить версию можно командой minikube version
- переключаемся на root и запускаем миникуб: minikube start --vm-driver=none
- после запуска стоит проверить статус: minikube status
- запущенные служебные компоненты можно увидеть командой: kubectl get pods --namespace=kube-system

### Для сброса кластера стоит удалить кластер и создать заново:
- minikube delete
- minikube start --vm-driver=none

Возможно, для повторного запуска потребуется выполнить команду: sudo sysctl fs.protected_regular=0

Инструкция по установке Minikube - [ссылка](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/)

## Ответ ##

Установка производилась на локальную машину с Ubuntu 20.04 
```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
sudo apt-get install conntrack
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```
Запуск 

```bash
sudo su
minikube start --vm-driver=none
```

Результат

```bash 
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology$ minikube version
minikube version: v1.25.2
commit: 362d5fdc0a3dbee389b3d3f1034e8023e72bd3a7
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov/devops/k8s_netology# minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

root@rkhozyainov-T530-ubuntu:/home/rkhozyainov/devops/k8s_netology# kubectl get pods --namespace=kube-system
NAME                                              READY   STATUS    RESTARTS   AGE
coredns-64897985d-9txhz                           1/1     Running   0          6m25s
etcd-rkhozyainov-t530-ubuntu                      1/1     Running   1          6m37s
kube-apiserver-rkhozyainov-t530-ubuntu            1/1     Running   1          6m39s
kube-controller-manager-rkhozyainov-t530-ubuntu   1/1     Running   1          6m39s
kube-proxy-hd8n2                                  1/1     Running   0          6m26s
kube-scheduler-rkhozyainov-t530-ubuntu            1/1     Running   1          6m37s
storage-provisioner                               1/1     Running   0          6m36s


```




## Задача 2: Запуск Hello World
После установки Minikube требуется его проверить. Для этого подойдет стандартное приложение hello world. А для доступа к нему потребуется ingress.

- развернуть через Minikube тестовое приложение по [туториалу](https://kubernetes.io/ru/docs/tutorials/hello-minikube/#%D1%81%D0%BE%D0%B7%D0%B4%D0%B0%D0%BD%D0%B8%D0%B5-%D0%BA%D0%BB%D0%B0%D1%81%D1%82%D0%B5%D1%80%D0%B0-minikube)
- установить аддоны ingress и dashboard

## Ответ ##

Создание Deployment
```bash
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov/devops/k8s_netology# kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4
deployment.apps/hello-node created

root@rkhozyainov-T530-ubuntu:/home/rkhozyainov/devops/k8s_netology# kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   1/1     1            1           21s

root@rkhozyainov-T530-ubuntu:/home/rkhozyainov/devops/k8s_netology# kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-6b89d599b9-dzlr8   1/1     Running   0          102s
```
Создание сервиса 
```bash
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov/devops/k8s_netology# kubectl expose deployment hello-node --type=LoadBalancer --port=8080
service/hello-node exposed

oot@rkhozyainov-T530-ubuntu:/home/rkhozyainov# kubectl get services
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-node   LoadBalancer   10.97.172.94   <pending>     8080:31666/TCP   11s
kubernetes   ClusterIP      10.96.0.1      <none>        443/TCP          5m46s

```

Установка аддонов 

```
oot@rkhozyainov-T530-ubuntu:/home/rkhozyainov# minikube addons enable dashboard
    ▪ Используется образ kubernetesui/metrics-scraper:v1.0.7
    ▪ Используется образ kubernetesui/dashboard:v2.3.1
💡  Some dashboard features require the metrics-server addon. To enable all features please run:

	minikube addons enable metrics-server	


🌟  The 'dashboard' addon is enabled
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov# minikube addons enable metrics-server
    ▪ Используется образ k8s.gcr.io/metrics-server/metrics-server:v0.4.2
🌟  The 'metrics-server' addon is enabled
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov# minikube addons enable ingress
    ▪ Используется образ k8s.gcr.io/ingress-nginx/controller:v1.1.1
    ▪ Используется образ k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
    ▪ Используется образ k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
🔎  Verifying ingress addon...
🌟  The 'ingress' addon is enabled
```
Результат 

```bash
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov# minikube addons list
|-----------------------------|----------|--------------|--------------------------------|
|         ADDON NAME          | PROFILE  |    STATUS    |           MAINTAINER           |
|-----------------------------|----------|--------------|--------------------------------|
| ambassador                  | minikube | disabled     | third-party (ambassador)       |
| auto-pause                  | minikube | disabled     | google                         |
| csi-hostpath-driver         | minikube | disabled     | kubernetes                     |
| dashboard                   | minikube | enabled ✅   | kubernetes                     |
| default-storageclass        | minikube | enabled ✅   | kubernetes                     |
| efk                         | minikube | disabled     | third-party (elastic)          |
| freshpod                    | minikube | disabled     | google                         |
| gcp-auth                    | minikube | disabled     | google                         |
| gvisor                      | minikube | disabled     | google                         |
| helm-tiller                 | minikube | disabled     | third-party (helm)             |
| ingress                     | minikube | enabled ✅   | unknown (third-party)          |
| ingress-dns                 | minikube | disabled     | google                         |
| istio                       | minikube | disabled     | third-party (istio)            |
| istio-provisioner           | minikube | disabled     | third-party (istio)            |
| kong                        | minikube | disabled     | third-party (Kong HQ)          |
| kubevirt                    | minikube | disabled     | third-party (kubevirt)         |
| logviewer                   | minikube | disabled     | unknown (third-party)          |
| metallb                     | minikube | disabled     | third-party (metallb)          |
| metrics-server              | minikube | enabled ✅   | kubernetes                     |
| nvidia-driver-installer     | minikube | disabled     | google                         |
| nvidia-gpu-device-plugin    | minikube | disabled     | third-party (nvidia)           |
| olm                         | minikube | disabled     | third-party (operator          |
|                             |          |              | framework)                     |
| pod-security-policy         | minikube | disabled     | unknown (third-party)          |
| portainer                   | minikube | disabled     | portainer.io                   |
| registry                    | minikube | disabled     | google                         |
| registry-aliases            | minikube | disabled     | unknown (third-party)          |
| registry-creds              | minikube | disabled     | third-party (upmc enterprises) |
| storage-provisioner         | minikube | enabled ✅   | google                         |
| storage-provisioner-gluster | minikube | disabled     | unknown (third-party)          |
| volumesnapshots             | minikube | disabled     | kubernetes                     |
|-----------------------------|----------|--------------|--------------------------------|

```

## Задача 3: Установить kubectl

Подготовить рабочую машину для управления корпоративным кластером. Установить клиентское приложение kubectl.
- подключиться к minikube 
- проверить работу приложения из задания 2, запустив port-forward до кластера

## Ответ ##

Port-forward
```bash
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov# kubectl port-forward deployment/hello-node 31666:31666
Forwarding from [::1]:31666 -> 31666
```

Результат 
```bash
oot@rkhozyainov-T530-ubuntu:/home/rkhozyainov# curl 127.0.0.1:31666
CLIENT VALUES:
client_address=172.17.0.1
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://127.0.0.1:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=127.0.0.1:31666
user-agent=curl/7.68.0
BODY:
-no body in request-
```

### Остановка всего ###

```bash
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov# kubectl delete service hello-node
service "hello-node" deleted
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov# kubectl delete deployment hello-node
deployment.apps "hello-node" deleted
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov# minikube stop
✋  Узел "minikube" останавливается ...
...

root@rkhozyainov-T530-ubuntu:/home/rkhozyainov# minikube delete
🔄  Uninstalling Kubernetes v1.23.3 using kubeadm ...
🔥  Deleting "minikube" in none ...
🔥  Trying to delete invalid profile minikube

```
Результат

```bash
root@rkhozyainov-T530-ubuntu:/home/rkhozyainov# minikube status
🤷  Profile "minikube" not found. Run "minikube profile list" to view all profiles.
👉  To start a cluster, run: "minikube start"

```

## Задача 4 (*): собрать через ansible (необязательное)

Профессионалы не делают одну и ту же задачу два раза. Давайте закрепим полученные навыки, автоматизировав выполнение заданий  ansible-скриптами. При выполнении задания обратите внимание на доступные модули для k8s под ansible.
 - собрать роль для установки minikube на aws сервисе (с установкой ingress)
 - собрать роль для запуска в кластере hello world
  
  ---


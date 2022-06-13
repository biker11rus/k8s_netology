# Домашнее задание к занятию "12.2 Команды для работы с Kubernetes"
Кластер — это сложная система, с которой крайне редко работает один человек. Квалифицированный devops умеет наладить работу всей команды, занимающейся каким-либо сервисом.
После знакомства с кластером вас попросили выдать доступ нескольким разработчикам. Помимо этого требуется служебный аккаунт для просмотра логов.

## Задание 1: Запуск пода из образа в деплойменте
Для начала следует разобраться с прямым запуском приложений из консоли. Такой подход поможет быстро развернуть инструменты отладки в кластере. Требуется запустить деплоймент на основе образа из hello world уже через deployment. Сразу стоит запустить 2 копии приложения (replicas=2). 

Требования:
 * пример из hello world запущен в качестве deployment
 * количество реплик в deployment установлено в 2
 * наличие deployment можно проверить командой kubectl get deployment
 * наличие подов можно проверить командой kubectl get pods


## Задание 2: Просмотр логов для разработки
Разработчикам крайне важно получать обратную связь от штатно работающего приложения и, еще важнее, об ошибках в его работе. 
Требуется создать пользователя и выдать ему доступ на чтение конфигурации и логов подов в app-namespace.

Требования: 
 * создан новый токен доступа для пользователя
 * пользователь прописан в локальный конфиг (~/.kube/config, блок users)
 * пользователь может просматривать логи подов и их конфигурацию (kubectl logs pod <pod_id>, kubectl describe pod <pod_id>)


## Задание 3: Изменение количества реплик 
Поработав с приложением, вы получили запрос на увеличение количества реплик приложения для нагрузки. Необходимо изменить запущенный deployment, увеличив количество реплик до 5. Посмотрите статус запущенных подов после увеличения реплик. 

Требования:
 * в deployment из задания 1 изменено количество реплик на 5
 * проверить что все поды перешли в статус running (kubectl get pods)

---

### Ответы ###

Задания выполняются с хостовой машины на кластере в Яндекс-облаке (из задания 12.4)  
В качестве примера используется nginx  

Запуск пода из образа в деплойменте
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl create deploy test-nginx --image=nginx:latest --replicas=2 --kubeconfig=./config
deployment.apps/test-nginx created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get deployment --kubeconfig=./config
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
test-nginx   2/2     2            2           19s
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get pods --kubeconfig=./config
NAME                          READY   STATUS    RESTARTS   AGE
test-nginx-5786cbffdd-hhdxv   1/1     Running   0          28s
test-nginx-5786cbffdd-mmmxf   1/1     Running   0          28s
```
Изменение количества реплик   
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl scale --replicas=5 deployment test-nginx --kubeconfig=./config
deployment.apps/test-nginx scaled
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get pods --kubeconfig=./config
NAME                          READY   STATUS    RESTARTS   AGE
test-nginx-5786cbffdd-bz229   1/1     Running   0          9s
test-nginx-5786cbffdd-cvr4z   1/1     Running   0          9s
test-nginx-5786cbffdd-hhdxv   1/1     Running   0          2m50s
test-nginx-5786cbffdd-mmmxf   1/1     Running   0          2m50s
test-nginx-5786cbffdd-td9hc   1/1     Running   0          9s
```

Просмотр логов для разработки


1. Создание namespace app-nginx 
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl create deploy test-nginx --image=nginx:latest --replicas=2 --kubeconfig=./config -n app-nginx 
deployment.apps/test-nginx created
```
2. Создание service account c помощью манифеста и вывод
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl apply -f ./role_yaml/sa.yml --kubeconfig=./config 
serviceaccount/rouser created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get sa rouser -o yaml --kubeconfig=./config --namespace=app-nginx
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"rouser","namespace":"app-nginx"}}
  creationTimestamp: "2022-06-13T21:17:11Z"
  name: rouser
  namespace: app-nginx
  resourceVersion: "64101"
  uid: f3cb90b6-643a-4984-861d-ba8bd6c5d78a
secrets:
- name: rouser-token-6w59h
```
3. Создание роли c помощью манифеста и вывод
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl apply -f ./role_yaml/role.yml --kubeconfig=./config 
role.rbac.authorization.k8s.io/pod-reader created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get role pod-reader -o yaml --kubeconfig=./config --namespace=app-nginx
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"Role","metadata":{"annotations":{},"name":"pod-reader","namespace":"app-nginx"},"rules":[{"apiGroups":[""],"resources":["pods","pods/log"],"verbs":["get","watch","list","describe"]}]}
  creationTimestamp: "2022-06-13T21:28:08Z"
  name: pod-reader
  namespace: app-nginx
  resourceVersion: "65462"
  uid: cb605459-8094-49e7-b911-26963759f9e3
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - pods/log
  verbs:
  - get
  - watch
  - list
  - describe
```
4. Создание привязки роли  c помощью манифеста и вывод
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl apply -f ./role_yaml/role-bind.yml --kubeconfig=./config 
rolebinding.rbac.authorization.k8s.io/read-pods created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get rolebindings read-pods -o yaml --kubeconfig=./config --namespace=app-nginx
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"rbac.authorization.k8s.io/v1","kind":"RoleBinding","metadata":{"annotations":{},"name":"read-pods","namespace":"app-nginx"},"roleRef":{"apiGroup":"rbac.authorization.k8s.io","kind":"Role","name":"pod-reader"},"subjects":[{"kind":"ServiceAccount","name":"rouser","namespace":"app-nginx"}]}
  creationTimestamp: "2022-06-13T21:31:04Z"
  name: read-pods
  namespace: app-nginx
  resourceVersion: "65832"
  uid: c274a5b1-c973-4990-8b0b-8147f959d44f
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: rouser
  namespace: app-nginx
```
5. Проверка, экспорт, добавление токена и контекста в config. 
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get secret --kubeconfig=./config --namespace=app-nginx
NAME                  TYPE                                  DATA   AGE
default-token-xpq22   kubernetes.io/service-account-token   3      104m
rouser-token-6w59h    kubernetes.io/service-account-token   3      66m
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ export TOKENTEST=$(kubectl get secret rouser-token-6w59h -o jsonpath='{.data.token}' --kubeconfig=./config --namespace=app-nginx | base64 --decode)
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl config set-credentials rouser --token=$TOKENTEST --kubeconfig=./config
User "rouser" set.
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl config set-context rouser@test_cluster --cluster=test_cluster --namespace=app-nginx --user=rouser --kubeconfig=./config
Context "rouser@test_cluster" created.
```
6. Проверки
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get pods --kubeconfig=./config --namespace=app-nginx --context rouser@test_cluster -o wide
NAME                          READY   STATUS    RESTARTS   AGE    IP            NODE    NOMINATED NODE   READINESS GATES
test-nginx-5786cbffdd-8gm5n   1/1     Running   0          118m   10.233.92.3   node3   <none>           <none>
test-nginx-5786cbffdd-h7qsf   1/1     Running   0          118m   10.233.70.4   node5   <none>           <none>
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl logs test-nginx-5786cbffdd-8gm5n --kubeconfig=./config --context rouser@test_cluster
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2022/06/13 20:39:42 [notice] 1#1: using the "epoll" event method
2022/06/13 20:39:42 [notice] 1#1: nginx/1.21.6
2022/06/13 20:39:42 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6) 
2022/06/13 20:39:42 [notice] 1#1: OS: Linux 5.4.0-109-generic
2022/06/13 20:39:42 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/06/13 20:39:42 [notice] 1#1: start worker processes
2022/06/13 20:39:42 [notice] 1#1: start worker process 31
2022/06/13 20:39:42 [notice] 1#1: start worker process 32
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl describe pod test-nginx-5786cbffdd-8gm5n --kubeconfig=./config --context rouser@test_cluster
Name:         test-nginx-5786cbffdd-8gm5n
Namespace:    app-nginx
Priority:     0
Node:         node3/10.128.0.13
Start Time:   Mon, 13 Jun 2022 23:39:40 +0300
Labels:       app=test-nginx
              pod-template-hash=5786cbffdd
Annotations:  cni.projectcalico.org/containerID: 9d3a253eec794359894b7997736f762e865801af13d46d8f1d9b72909809dc1e
              cni.projectcalico.org/podIP: 10.233.92.3/32
              cni.projectcalico.org/podIPs: 10.233.92.3/32
Status:       Running
IP:           10.233.92.3
IPs:
  IP:           10.233.92.3
Controlled By:  ReplicaSet/test-nginx-5786cbffdd
Containers:
  nginx:
    Container ID:   containerd://83859ba742c50f3116349a656f2aedde045f2676829602bc2a5656f9de985928
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:2bcabc23b45489fb0885d69a06ba1d648aeda973fae7bb981bafbb884165e514
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 13 Jun 2022 23:39:42 +0300
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-z4hwr (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-z4hwr:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:                      <none>
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get deployment test-nginx --kubeconfig=./config  --context=rouser@test_cluster
Error from server (Forbidden): deployments.apps "test-nginx" is forbidden: User "system:serviceaccount:app-nginx:rouser" cannot get resource "deployments" in API group "apps" in the namespace "app-nginx"
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl scale --replicas=6  deployment test-nginx --kubeconfig=./config  --context=rouser@test_cluster
Error from server (Forbidden): deployments.apps "test-nginx" is forbidden: User "system:serviceaccount:app-nginx:rouser" cannot get resource "deployments" in API group "apps" in the namespace "app-nginx"
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get deployment test-nginx --kubeconfig=./config 
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
test-nginx   2/2     2            2           128m
```


### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---

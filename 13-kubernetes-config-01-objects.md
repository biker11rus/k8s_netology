# Домашнее задание к занятию "13.1 контейнеры, поды, deployment, statefulset, services, endpoints"
Настроив кластер, подготовьте приложение к запуску в нём. Приложение стандартное: бекенд, фронтенд, база данных. Его можно найти в папке 13-kubernetes-config.

## Задание 1: подготовить тестовый конфиг для запуска приложения
Для начала следует подготовить запуск приложения в stage окружении с простыми настройками. Требования:
* под содержит в себе 2 контейнера — фронтенд, бекенд;
* регулируется с помощью deployment фронтенд и бекенд;
* база данных — через statefulset.

## Задание 2: подготовить конфиг для production окружения
Следующим шагом будет запуск приложения в production окружении. Требования сложнее:
* каждый компонент (база, бекенд, фронтенд) запускаются в своем поде, регулируются отдельными deployment’ами;
* для связи используются service (у каждого компонента свой);
* в окружении фронта прописан адрес сервиса бекенда;
* в окружении бекенда прописан адрес сервиса базы данных.

## Задание 3 (*): добавить endpoint на внешний ресурс api
Приложению потребовалось внешнее api, и для его использования лучше добавить endpoint в кластер, направленный на это api. Требования:
* добавлен endpoint до внешнего api (например, геокодер).

---

## Ответ ##
### Подготовка ###
Поднят кластер с помощью terraform и kubespray на яндекс облаке. 1 мастер 2 воркера. Управление происходит с хоста. Данные для подключения взяты из мастер ноды /etc/kubernates/admin.conf  
При подготовке приложений из папки 13-kubernetes-config к запуску в кластере столкнулся с некоторыми проблемами...  
В описании к приложениям указано, что необходимо подготовить файлы .env, в которых указаны параметры подключения к базе в backend'е и адрес и порт подключения backend'а для frontend'а.  
Так как приложение frontend'a написано с помощью js и выполняется на машине с которой заходишь на сайт(пользовательской), необходимо обеспечить доступ к backend'у с этой машины (по сути с интернета).  
Приложение js компилируемые, т.е. при запуске контейнера нельзя будет передать параметры подключения, только во время сборки.  
Backend написан на Python и возможно передать ему параметры подключения к базе во время запуска контейнера.  
Если собирать образы с помощью docker-compose, то параметры можно передать через .env файлы, но если собирать через docker build он будет игнорировать эти файлы и подставить дефолтные значения указанные в main.js для frontend'a и в main.py для backend'a

Для решения этих проблем в dockerfile у frontend'а добавил возможность передачи параметра адреса и порта backend'a при сборке 
```
ARG BASE_URL=http://localhost:9000
ENV BASE_URL="${BASE_URL}"
```
В таком случаи при сборке с помощью docker build нужно будет указать параметры через --build-arg, а при сборке через docker-compose можно эти параметры задать в .env файле

Сборка frontend'a    
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/frontend$ docker build -t rkhozyainov/test_frontend:0.1 --build-arg BASE_URL=http://51.250.91.100:30081 .
```
Сборка backend'a 
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/backend$ docker build -t rkhozyainov/test_backend:0.1 .
```
Отправка образов в репозиторий
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1$ docker push rkhozyainov/test_frontend:0.1
The push refers to repository [docker.io/rkhozyainov/test_frontend]
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1$ docker push rkhozyainov/test_backend:0.1
The push refers to repository [docker.io/rkhozyainov/test_backend]

```
### Задание 1 ###

Для запуска stage окружения создано 3 манифеста  
1. 10-stage-ns.yml Для создания namespace stage
```
apiVersion: v1
kind: Namespace
metadata:
  name: stage
```
2. 20-stage-statefull.yml для базы данных через statefulset и сервиса к нему. В сервисе указан type: ClusterIP так как нам не нужно публиковать базу данных за пределы кластера. Селектор - app: postgres. При запуске контейнера передаем параметры для базы через env
```
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: stage
spec:
  ports:
    - name: postgres
      port: 5432
  selector:
    app: postgres
  type: ClusterIP
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: stage
spec:
  serviceName: "postgres"
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:13-alpine
          imagePullPolicy: IfNotPresent
          env:
            - name: POSTGRES_PASSWORD
              value: postgres
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_DB
              value: news
```
3. 30-stage-back-front.yml. для создание деплоймента и сервиса back-front. В сервисе указан type: NodePort для доступа к контейнерам по ip адресам нод, портам 30080 для frontend'a 30081 для доступа frontend'a к backend'у. При запуске контейнера backend'a передаем через env данные для подключения к базе
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: back-front
  name: back-front
  namespace: stage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: back-front
  template:
    metadata:
      labels:
        app: back-front
    spec:
      containers:
      - image: rkhozyainov/test_frontend:0.1
        imagePullPolicy: IfNotPresent
        name: front
        ports:
          - containerPort: 80
      - image: rkhozyainov/test_backend:0.1
        imagePullPolicy: IfNotPresent
        name: back
        ports:
          - containerPort: 9000
        env:
            - name: DATABASE_URL
              value: postgres://postgres:postgres@postgres:5432/news      
---
apiVersion: v1
kind: Service
metadata:
  name: back-front
  namespace: stage
spec:
  ports:
    - name: post8000
      port: 8000
      targetPort: 80
      nodePort: 30080
    - name: port9000
      port: 9000
      nodePort: 30081
  selector:
    app: back-front
  type: NodePort  
```

Запуск  
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/manifests-stage$ kubectl apply -f ./10-stage-ns.yml 
namespace/stage created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/manifests-stage$ kubectl apply -f ./20-stage-statefull.yml 
service/postgres created
statefulset.apps/postgres created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/manifests-stage$ kubectl apply -f ./30-stage-back-front.yml 
deployment.apps/back-front created
service/back-front created
```
Проверка 
```
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/manifests-stage$ kubectl get po,deploy,sts,svc,ep -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP             NODE    NOMINATED NODE   READINESS GATES
pod/back-front-74b6769bb4-hv4p4   2/2     Running   0          8m47s   10.233.92.15   node3   <none>           <none>
pod/postgres-0                    1/1     Running   0          8m50s   10.233.96.19   node2   <none>           <none>

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                                                       SELECTOR
deployment.apps/back-front   1/1     1            1           8m47s   front,back   rkhozyainov/test_frontend:0.1,rkhozyainov/test_backend:0.1   app=back-front

NAME                        READY   AGE     CONTAINERS   IMAGES
statefulset.apps/postgres   1/1     8m50s   postgres     postgres:13-alpine

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE     SELECTOR
service/back-front   NodePort    10.233.63.199   <none>        8000:30080/TCP,9000:30081/TCP   8m47s   app=back-front
service/postgres     ClusterIP   10.233.63.39    <none>        5432/TCP                        8m50s   app=postgres

NAME                   ENDPOINTS                           AGE
endpoints/back-front   10.233.92.15:80,10.233.92.15:9000   8m47s
endpoints/postgres     10.233.96.19:5432                   8m50s
```
Доступ с браузера 
![stage](./images/13-1-stage.png)

### Задание 2 ###

Для запуска prod окружения создано 4 манифеста 
1. 10-stage-ns.yml Для создания namespace prod
```
apiVersion: v1
kind: Namespace
metadata:
  name: prod
```
2. 20-prod-postgres.yml для деплоймента и сервиса базы В сервисе указан type: ClusterIP так как нам не нужно публиковать базу данных за пределы кластера. Селектор - app: postgres. При запуске контейнера передаем параметры для базы через env
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: postgres
  name: postgres
  namespace: prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - image: postgres:13-alpine
        imagePullPolicy: IfNotPresent
        name: postgres
        env:
            - name: POSTGRES_PASSWORD
              value: postgres
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_DB
              value: news

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: prod
spec:
  ports:
    - name: postgres
      port: 5432
  selector:
    app: postgres
  type: ClusterIP
```
3. 30-prod-back.yml для создание деплоймента и сервиса backend'a. В сервисе указан type: NodePort для доступа frontend'a к backend'у по ip адресам нод и порту 30081. При запуске контейнера передаем через env данные для подключения к базе
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: back
  name: back
  namespace: prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: back
  template:
    metadata:
      labels:
        app: back
    spec:
      containers:
      - image: rkhozyainov/test_backend:0.1
        imagePullPolicy: IfNotPresent
        name: back
        ports:
          - containerPort: 9000
        env:
            - name: DATABASE_URL
              value: postgres://postgres:postgres@postgres:5432/news
---
apiVersion: v1
kind: Service
metadata:
  name: back
  namespace: prod
spec:
  ports:
    - name: port9000
      port: 9000
      nodePort: 30081
  selector:
    app: back
  type: NodePort
```
4. 40-prod-front.yml для создание деплоймента и сервиса fronten'a. В сервисе указан type: NodePort для доступа к frontend'у по ip адресам нод и порту 30080 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: front
  name: front
  namespace: prod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: front
  template:
    metadata:
      labels:
        app: front
    spec:
      containers:
      - image: rkhozyainov/test_frontend:0.1
        imagePullPolicy: IfNotPresent
        name: front
        ports:
          - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: front
  namespace: prod
spec:
  ports:
    - name: post8000
      port: 8000
      targetPort: 80
      nodePort: 30080
  selector:
    app: front
  type: NodePort

```

Запуск

```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/manifests-prod$ kubectl apply -f ./10-prod-ns.yml 
namespace/prod created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/manifests-prod$ kubectl apply -f ./20-prod-postgres.yml 
deployment.apps/postgres created
service/postgres created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/manifests-prod$ kubectl apply -f ./30-prod-back.yml 
deployment.apps/back created
service/back created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/manifests-prod$ kubectl apply -f ./40-prod-front.yml 
deployment.apps/front created
service/front created
```
Проверка
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/13.1/manifests-prod$ kubectl get po,deploy,sts,svc,ep -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
pod/back-cfcf5fd8-nhc7f        1/1     Running   0          20s   10.233.92.16   node3   <none>           <none>
pod/front-754fccb78-thb4x      1/1     Running   0          15s   10.233.96.21   node2   <none>           <none>
pod/postgres-d96dcb574-5psw8   1/1     Running   0          25s   10.233.96.20   node2   <none>           <none>

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                          SELECTOR
deployment.apps/back       1/1     1            1           20s   back         rkhozyainov/test_backend:0.1    app=back
deployment.apps/front      1/1     1            1           15s   front        rkhozyainov/test_frontend:0.1   app=front
deployment.apps/postgres   1/1     1            1           25s   postgres     postgres:13-alpine              app=postgres

NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   SELECTOR
service/back       NodePort    10.233.23.248   <none>        9000:30081/TCP   20s   app=back
service/front      NodePort    10.233.24.152   <none>        8000:30080/TCP   15s   app=front
service/postgres   ClusterIP   10.233.13.251   <none>        5432/TCP         25s   app=postgres

NAME                 ENDPOINTS           AGE
endpoints/back       10.233.92.16:9000   20s
endpoints/front      10.233.96.21:80     15s
endpoints/postgres   10.233.96.20:5432   25s
```
Доступ с браузера 
![prod](./images/13-1-prod.png)


### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

В качестве решения прикрепите к ДЗ конфиг файлы для деплоя. Прикрепите скриншоты вывода команды kubectl со списком запущенных объектов каждого типа (pods, deployments, statefulset, service) или скриншот из самого Kubernetes, что сервисы подняты и работают.

---

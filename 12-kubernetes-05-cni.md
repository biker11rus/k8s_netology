# Домашнее задание к занятию "12.5 Сетевые решения CNI"
После работы с Flannel появилась необходимость обеспечить безопасность для приложения. Для этого лучше всего подойдет Calico.
## Задание 1: установить в кластер CNI плагин Calico
Для проверки других сетевых решений стоит поставить отличный от Flannel плагин — например, Calico. Требования: 
* установка производится через ansible/kubespray;
* после применения следует настроить политику доступа к hello-world извне. Инструкции [kubernetes.io](https://kubernetes.io/docs/concepts/services-networking/network-policies/), [Calico](https://docs.projectcalico.org/about/about-network-policy)

## Задание 2: изучить, что запущено по умолчанию
Самый простой способ — проверить командой calicoctl get <type>. Для проверки стоит получить список нод, ipPool и profile.
Требования: 
* установить утилиту calicoctl;
* получить 3 вышеописанных типа в консоли.


# Ответ #

## Задание 1 ## 

Развернут кластер в Яндекс-Облаке с помощью терраформ и kubespray, calico включен. Настроен доступ с хостовой машины.  

```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/kubespray$ kubectl get nodes 
NAME    STATUS   ROLES                  AGE   VERSION
node1   Ready    control-plane,master   87m   v1.23.7
node2   Ready    <none>                 86m   v1.23.7
node3   Ready    <none>                 86m   v1.23.7
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster/kubespray$ cat ./inventory/cluster125/group_vars/k8s_cluster/k8s-cluster.yml | grep kube_network_plugin
kube_network_plugin: calico
```
Применим 3 деплоймента и сервисы привязанные к ним frontend backend cache из манифестов 
```bash 
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl apply -f ./main/
deployment.apps/frontend created
service/frontend unchanged
deployment.apps/backend created
service/backend unchanged
deployment.apps/cache created
service/cache created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl get po,svc
NAME                            READY   STATUS    RESTARTS   AGE
pod/backend-f785447b9-b7q6d     1/1     Running   0          26s
pod/cache-b4f65b647-t69vw       1/1     Running   0          26s
pod/frontend-8645d9cb9c-gm5tt   1/1     Running   0          26s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/backend      ClusterIP   10.233.39.244   <none>        80/TCP    53m
service/cache        ClusterIP   10.233.7.4      <none>        80/TCP    26s
service/frontend     ClusterIP   10.233.3.228    <none>        80/TCP    53m
service/kubernetes   ClusterIP   10.233.0.1      <none>        443/TCP   3h51m
```
Проверим сетевые политики и доступность. Убедимся что все доступно.
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl get networkpolicies
No resources found in default namespace.
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec backend-f785447b9-b7q6d -- curl -s -m 1 frontend
Praqma Network MultiTool (with NGINX) - frontend-8645d9cb9c-gm5tt - 10.233.96.13
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec frontend-8645d9cb9c-gm5tt -- curl -s -m 1 frontend
Praqma Network MultiTool (with NGINX) - frontend-8645d9cb9c-gm5tt - 10.233.96.13
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec backend-f785447b9-b7q6d -- curl -s -m 1 cache
Praqma Network MultiTool (with NGINX) - cache-b4f65b647-t69vw - 10.233.96.14
```
Применим политики запрета ingress и egress убедимся что доступ пропал.
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl apply -f ./network-policy/00-default-deny-ingress.yaml 
networkpolicy.networking.k8s.io/default-deny-ingress created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl apply -f ./network-policy/10-default-deny-egress.yaml 
networkpolicy.networking.k8s.io/default-deny-egress created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec backend-f785447b9-b7q6d -- curl -s -m 1 frontend
command terminated with exit code 28
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec backend-f785447b9-b7q6d -- curl -s -m 1 cache
command terminated with exit code 28
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec cache-b4f65b647-t69vw -- curl -s -m 1 fronend
command terminated with exit code 28
```
Применим политики разрешающую исходящие(egress) DNS для всех, ingress c frontend к backend и ingress c backend к frontend. Проверим доступы.
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl apply -f ./network-policy/20-allow-egress-dns.yaml 
networkpolicy.networking.k8s.io/allow-egress-dns created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl apply -f ./network-policy/30-allow-ingress-frontend.yaml 
networkpolicy.networking.k8s.io/allow-ingress-frontend created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl apply -f ./network-policy/40-allow-ingress-backend.yaml 
networkpolicy.networking.k8s.io/allow-ingress-backend created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec backend-f785447b9-b7q6d -- curl -s -m 1 frontend
command terminated with exit code 28
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec frontend-8645d9cb9c-gm5tt -- curl -s -m 1 backend
command terminated with exit code 28
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec frontend-8645d9cb9c-gm5tt -- curl -s -m 1 cache
command terminated with exit code 28
```
Применим политику разрешающую исходящие(egress) c backend к frontend. Проверим доступы. 
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec backend-f785447b9-b7q6d -- curl -s -m 1 frontend
Praqma Network MultiTool (with NGINX) - frontend-8645d9cb9c-gm5tt - 10.233.96.13
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec frontend-8645d9cb9c-gm5tt -- curl -s -m 1 backend
command terminated with exit code 28
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec frontend-8645d9cb9c-gm5tt -- curl -s -m 1 cache
command terminated with exit code 28
```
Доступ появился c backend к frontend. Но не наоборот.  
Применим политику разрешающую исходящие(egress) c frontend к backend. Проверим доступы. 
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl apply -f ./network-policy/60-allow-egress-frontend.yaml 
networkpolicy.networking.k8s.io/allow-egress-frontend created
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec backend-f785447b9-b7q6d -- curl -s -m 1 frontend
Praqma Network MultiTool (with NGINX) - frontend-8645d9cb9c-gm5tt - 10.233.96.13
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec frontend-8645d9cb9c-gm5tt -- curl -s -m 1 backend
Praqma Network MultiTool (with NGINX) - backend-f785447b9-b7q6d - 10.233.92.13
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec frontend-8645d9cb9c-gm5tt -- curl -s -m 1 cache
command terminated with exit code 28
```
Доступ появился от frontend к backend и обратно. Доступа к cache нет. 
Полное описание политик  
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl describe networkpolicies 
Name:         allow-egress-backend
Namespace:    default
Created on:   2022-06-23 01:07:22 +0300 MSK
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=backend
  Not affecting ingress traffic
  Allowing egress traffic:
    To Port: <any> (traffic allowed to all ports)
    To:
      PodSelector: app=frontend
  Policy Types: Egress


Name:         allow-egress-dns
Namespace:    default
Created on:   2022-06-23 01:15:46 +0300 MSK
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Not affecting ingress traffic
  Allowing egress traffic:
    To Port: 53/UDP
    To: <any> (traffic not restricted by destination)
  Policy Types: Egress


Name:         allow-egress-frontend
Namespace:    default
Created on:   2022-06-23 01:10:44 +0300 MSK
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=frontend
  Not affecting ingress traffic
  Allowing egress traffic:
    To Port: <any> (traffic allowed to all ports)
    To:
      PodSelector: app=backend
  Policy Types: Egress


Name:         allow-ingress-backend
Namespace:    default
Created on:   2022-06-23 01:02:55 +0300 MSK
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=backend
  Allowing ingress traffic:
    To Port: 80/TCP
    From:
      PodSelector: app=frontend
  Not affecting egress traffic
  Policy Types: Ingress


Name:         allow-ingress-frontend
Namespace:    default
Created on:   2022-06-23 01:02:39 +0300 MSK
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=frontend
  Allowing ingress traffic:
    To Port: 80/TCP
    From:
      PodSelector: app=backend
  Not affecting egress traffic
  Policy Types: Ingress


Name:         default-deny-egress
Namespace:    default
Created on:   2022-06-23 00:51:11 +0300 MSK
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Not affecting ingress traffic
  Allowing egress traffic:
    <none> (Selected pods are isolated for egress connectivity)
  Policy Types: Egress


Name:         default-deny-ingress
Namespace:    default
Created on:   2022-06-23 00:51:05 +0300 MSK
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     <none> (Allowing the specific traffic to all pods in this namespace)
  Allowing ingress traffic:
    <none> (Selected pods are isolated for ingress connectivity)
  Not affecting egress traffic
  Policy Types: Ingress
```

Удалим политику DNS и проверим доступы.
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl delete networkpolicies allow-egress-dns 
networkpolicy.networking.k8s.io "allow-egress-dns" deleted
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec backend-f785447b9-b7q6d -- curl -s -m 1 frontend
command terminated with exit code 28
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl exec frontend-8645d9cb9c-gm5tt -- curl -s -m 1 backend
command terminated with exit code 28
```

## Задание 2 ## 

Установка calicoctl как плагин к kubectl
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ curl -L https://github.com/projectcalico/calico/releases/download/v3.22.3/calicoctl-linux-amd64 -o kubectl-calico
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 38.5M  100 38.5M    0     0  6834k      0  0:00:05  0:00:05 --:--:-- 9557k
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ chmod +x kubectl-calico
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ sudo cp ./kubectl-calico /usr/local/bin/
[sudo] пароль для rkhozyainov: 
```
Проверка 
```bash
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl calico get nodes
NAME    
node1   
node2   
node3   

rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl calico get ipPool
NAME           CIDR             SELECTOR   
default-pool   10.233.64.0/18   all()      

rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/k8s_netology/12.5/manifests$ kubectl calico get profile
NAME                                                 
projectcalico-default-allow                          
kns.default                                          
kns.kube-node-lease                                  
kns.kube-public                                      
kns.kube-system                                      
ksa.default.default                                  
ksa.kube-node-lease.default                          
ksa.kube-public.default                              
ksa.kube-system.attachdetach-controller              
ksa.kube-system.bootstrap-signer                     
ksa.kube-system.calico-kube-controllers              
ksa.kube-system.calico-node                          
ksa.kube-system.certificate-controller               
ksa.kube-system.clusterrole-aggregation-controller   
ksa.kube-system.coredns                              
ksa.kube-system.cronjob-controller                   
ksa.kube-system.daemon-set-controller                
ksa.kube-system.default                              
ksa.kube-system.deployment-controller                
ksa.kube-system.disruption-controller                
ksa.kube-system.dns-autoscaler                       
ksa.kube-system.endpoint-controller                  
ksa.kube-system.endpointslice-controller             
ksa.kube-system.endpointslicemirroring-controller    
ksa.kube-system.ephemeral-volume-controller          
ksa.kube-system.expand-controller                    
ksa.kube-system.generic-garbage-collector            
ksa.kube-system.horizontal-pod-autoscaler            
ksa.kube-system.job-controller                       
ksa.kube-system.kube-proxy                           
ksa.kube-system.namespace-controller                 
ksa.kube-system.node-controller                      
ksa.kube-system.nodelocaldns                         
ksa.kube-system.persistent-volume-binder             
ksa.kube-system.pod-garbage-collector                
ksa.kube-system.pv-protection-controller             
ksa.kube-system.pvc-protection-controller            
ksa.kube-system.replicaset-controller                
ksa.kube-system.replication-controller               
ksa.kube-system.resourcequota-controller             
ksa.kube-system.root-ca-cert-publisher               
ksa.kube-system.service-account-controller           
ksa.kube-system.service-controller                   
ksa.kube-system.statefulset-controller               
ksa.kube-system.token-cleaner                        
ksa.kube-system.ttl-after-finished-controller        
ksa.kube-system.ttl-controller                       
```
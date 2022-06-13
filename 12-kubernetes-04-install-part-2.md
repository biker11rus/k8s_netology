# Домашнее задание к занятию "12.4 Развертывание кластера на собственных серверах, лекция 2"
Новые проекты пошли стабильным потоком. Каждый проект требует себе несколько кластеров: под тесты и продуктив. Делать все руками — не вариант, поэтому стоит автоматизировать подготовку новых кластеров.

## Задание 1: Подготовить инвентарь kubespray
Новые тестовые кластеры требуют типичных простых настроек. Нужно подготовить инвентарь и проверить его работу. Требования к инвентарю:
* подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды;
* в качестве CRI — containerd;
* запуск etcd производить на мастере.

## Задание 2 (*): подготовить и проверить инвентарь для кластера в AWS
Часть новых проектов хотят запускать на мощностях AWS. Требования похожи:
* разворачивать 5 нод: 1 мастер и 4 рабочие ноды;
* работать должны на минимально допустимых EC2 — t3.small.

## Ответ ##

1. Склонирован репозиторий kubespray, установлены зависимости и создана копия конфигурации 
```bash
git clone https://github.com/kubernetes-sigs/kubespray
sudo pip3 install -r requirements.txt
cp -rfp inventory/sample inventory/mycluster
```
2.  Созданы 5 ВМ в Яндекс Облаке с помощью terraform main.tf. После создания ВМ автоматически создается inventory.ini с помощью шаблона inventory.tpl.   
Метадата для ВМ  создаются с помощью шаблона cloud_init_config.tpl. Переменные в variables.tf  

main.tf

```
terraform {
  required_providers {
    yandex = {
      source  = "yandex-cloud/yandex"
      version = "~> 0.61.0"
    }
  }
}
provider "yandex" {
  service_account_key_file = "key.json"
  cloud_id                 = var.yandex_cloud_id
  folder_id                = var.yandex_folder_id
  zone                     = var.yandex_zone
}
module "vpc" {
  source       = "hamnsk/vpc/yandex"
  version      = "0.5.0"
  description  = "managed by terraform"
  yc_folder_id = var.yandex_folder_id
  name         = "yc_vpc2"
  subnets      = local.vpc_subnets.yc_sub
}
locals {
  vpc_subnets = {
    yc_sub = [
      {
        "v4_cidr_blocks" : [
          "10.128.0.0/24"
        ],
        "zone" : var.yandex_zone
      }
    ]
  }
}

locals {
  instance_set = {
    node1    = "kube_control_plane"
    node2    = "kube_node"
    node3    = "kube_node"
    node4    = "kube_node"
    node5    = "kube_node"
  }
}


data "template_file" "cloud_init" {
  template = file("cloud_init_config.tpl")
  vars = {
    user    = var.user
    ssh_key = file(var.public_key_path)
  }
}
resource "yandex_compute_instance" "vms" {
  for_each = local.instance_set
  name     = each.key
  resources {
    cores  = 2
    memory = 4
  }
  boot_disk {
    initialize_params {
      image_id = var.iso_id
      size     = 20
    }
  }
  network_interface {
    subnet_id = module.vpc.subnet_ids[0]
    nat       = true
  }
  metadata = {
    user-data = data.template_file.cloud_init.rendered
  }
  labels = {
    ansible-group = each.value
  }
}
resource "local_file" "AnsibleInventory" {
  content = templatefile("inventory.tpl",
    {
      vm-names      = [for k, p in yandex_compute_instance.vms : p.name],
      private-ip    = [for k, p in yandex_compute_instance.vms : p.network_interface.0.ip_address],
      public-ip     = [for k, p in yandex_compute_instance.vms : p.network_interface.0.nat_ip_address],
      ansible-group = [for k, p in yandex_compute_instance.vms : p.labels.ansible-group],
      ssh_user      = var.user
    }
  )
  filename = "../kubespray/inventory/mycluster/inventory.ini"
}
```

inventory.tpl  
```
[all]
%{ for index, vms in vm-names ~}
${vms} ansible_host=${public-ip[index]} ip=${private-ip[index]}
%{ endfor ~}

[kube_control_plane]
%{ for indexgp, group in ansible-group ~}
%{ if group == "kube_control_plane" ~}
${vm-names[indexgp]}
%{ endif ~}
%{ endfor ~}

[etcd]
%{ for indexgp, group in ansible-group ~}
%{ if group == "kube_control_plane" ~}
${vm-names[indexgp]}
%{ endif ~}
%{ endfor ~}

[kube_node]
%{ for indexgp, group in ansible-group ~}
%{ if group == "kube_node" ~}
${vm-names[indexgp]}
%{ endif ~}
%{ endfor ~}


[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

cloud_init_config.tpl  
```
#cloud-config
users:
  - name: ${user}
    groups: sudo
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    ssh-authorized-keys:
      - ${ssh_key}
```

Итоговый inventory.ini

```
[all]
node1 ansible_host=51.250.85.5 ip=10.128.0.28
node2 ansible_host=51.250.95.161 ip=10.128.0.20
node3 ansible_host=51.250.74.93 ip=10.128.0.13
node4 ansible_host=51.250.93.66 ip=10.128.0.9
node5 ansible_host=51.250.94.198 ip=10.128.0.37

[kube_control_plane]
node1

[etcd]
node1

[kube_node]
node2
node3
node4
node5


[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr

```

3. Для подключения из вне добавил реальник control-node в файл  k8s-cluster.yml

```bash
< # supplementary_addresses_in_ssl_keys: [10.0.0.1, 10.0.0.2, 10.0.0.3]
---
> supplementary_addresses_in_ssl_keys: [51.250.85.5]
```
4. Установка кластера
   
```bash
ansible-playbook -i inventory/mycluster/inventory.ini cluster.yml -b
```
6. Проверка с control node
   
```bash
root@node1:/home/rkhozyainov# kubectl version
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.7", GitCommit:"42c05a547468804b2053ecf60a3bd15560362fc2", GitTreeState:"clean", BuildDate:"2022-05-24T12:30:55Z", GoVersion:"go1.17.10", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.7", GitCommit:"42c05a547468804b2053ecf60a3bd15560362fc2", GitTreeState:"clean", BuildDate:"2022-05-24T12:24:41Z", GoVersion:"go1.17.10", Compiler:"gc", Platform:"linux/amd64"}
root@node1:/home/rkhozyainov# kubectl get nodes
NAME    STATUS   ROLES                  AGE     VERSION
node1   Ready    control-plane,master   5h50m   v1.23.7
node2   Ready    <none>                 5h49m   v1.23.7
node3   Ready    <none>                 5h49m   v1.23.7
node4   Ready    <none>                 5h49m   v1.23.7
node5   Ready    <none>                 5h49m   v1.23.7
root@node1:/home/rkhozyainov# kubectl create deploy nginx --image=nginx:latest --replicas=2
deployment.apps/nginx created
root@node1:/home/rkhozyainov# kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP            NODE    NOMINATED NODE   READINESS GATES
nginx-7c658794b9-k9rxb   1/1     Running   0          4m32s   10.233.70.1   node5   <none>           <none>
nginx-7c658794b9-tkglc   1/1     Running   0          4m33s   10.233.92.1   node3   <none>           <none>
```

7. Для подключения с хоста создан файл config и заполнен данными из /etc/kubernetes/admin.conf с control_node: certificate-authority-data, client-certificate-data, client-key-data. Указан ip control_node

config
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: 
    server: https://51.250.85.5:6443
  name: test_cluster
contexts:
- context:
    cluster: test_cluster
    user: kubernetes-admin
  name: kubernetes-admin@test_cluster
current-context: kubernetes-admin@test_cluster
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: 
    client-key-data: 
```
Проверка  

```bash 
rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get nodes -o wide --kubeconfig=./config 
NAME    STATUS   ROLES                  AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
node1   Ready    control-plane,master   6h11m   v1.23.7   10.128.0.28   <none>        Ubuntu 20.04.4 LTS   5.4.0-109-generic   containerd://1.6.4
node2   Ready    <none>                 6h10m   v1.23.7   10.128.0.20   <none>        Ubuntu 20.04.4 LTS   5.4.0-109-generic   containerd://1.6.4
node3   Ready    <none>                 6h10m   v1.23.7   10.128.0.13   <none>        Ubuntu 20.04.4 LTS   5.4.0-109-generic   containerd://1.6.4
node4   Ready    <none>                 6h10m   v1.23.7   10.128.0.9    <none>        Ubuntu 20.04.4 LTS   5.4.0-109-generic   containerd://1.6.4
node5   Ready    <none>                 6h10m   v1.23.7   10.128.0.37   <none>        Ubuntu 20.04.4 LTS   5.4.0-109-generic   containerd://1.6.4

rkhozyainov@rkhozyainov-T530-ubuntu:~/devops/Other/test_k8s_cluster$ kubectl get pods --kubeconfig=./config 
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7c658794b9-k9rxb   1/1     Running   0          15m
nginx-7c658794b9-tkglc   1/1     Running   0          15m

```

 
---


### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---

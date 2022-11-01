University: [ITMO University](https://itmo.ru/ru/)
Faculty: [FICT](https://fict.itmo.ru)
Course: [Introduction to distributed technologies](https://github.com/itmo-ict-faculty/introduction-to-distributed-technologies)
Year: 2022/2023
Group: K4110c
Author: Ovcharov Evgenii Mihailovich
Lab: Lab4
Date of create: 31.10.2022
Date of finished: 
___
## Схема организации
Описание    
![scheme](./images/scheme.png)    
___
## Скриншоты
### 1. Запуск
Сначала попытался запустить используя встроенный в minikube calico:    
```bash
minikube start --nodes 2 --cni calico
```
Проверка результата - оба узла запущены:    
![nodes](./images/nodes.png)    
![status](./images/status.png)    
А вот с calico-nodes возникли пробемы:    
![calico_pods](./images/calico_pods.png)
![calico_err](./images/calico_err.png)    
В результате встроенный calico не работает, поэтому было решено использовать способ установки через манифест:
```bash
minikube start --nodes 2 --network-plugin cni
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.3/manifests/calico.yaml
```
Этот способ сработал:    
![calico_ok](./images/calico_ok.png)    
## 2. Установка labels
Для указания `labels` установим calicoctl:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.3/manifests/calicoctl.yaml
alias calicoctl="kubectl exec -i -n kube-system calicoctl -- /calicoctl"
```
Далее после удаления `default-ipv4-ippool` установим `labels` по признаку стойки:  
```bash
kubectl label nodes minikube rack=0
kubectl label nodes minikube-m02 rack=1
```
![labels](./images/labels.png)    
После чего создадим IP pool для каждой стойки:    
```bash
calicoctl create -f  - < lab4-ippool.yaml
```
![ip_pools](./images/ip_pools.png)    
### 3. Deployment
Аналогично lab2 создадал service типа `LoadBalancer`. `Container name` и `Container IP` изменяются, так как сервис распределяет нагрузку между двумя репликами. При этом по 3ей цифре IP можно понять на реплику какой "стойки" был направлен запрос.    
![res1](./images/res1.png)    
![res2](./images/res2.png)    
### 4. Ping
Зайдем в под и попингуем другой:
```bash
kubectl get pods -o wide
kubectl exec -ti lab4-deployment-84b98958f9-w4tkc -- sh
```
![ping](./images/ping.png)    
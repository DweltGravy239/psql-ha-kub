# Отказоустойчивый кластер PostgreSQL в Kubernetes
 
Стенд разворачивает кластер PostgreSQL под управлением Patroni и etcd.
Кластер Kubernetes (k3s) состоит из трёх узлов, каждый из которых
представляет отдельную зону доступности. Маршрутизация клиентского
трафика на текущего лидера выполняется через HAProxy, состояние
собирается Prometheus и визуализируется в Grafana.
 
## Состав репозитория
 
| Файл | Содержимое |
|---|---|
| `docker/dockerfile_patroni` | образ Patroni на базе `postgres:16-bookworm` |
| `manifests/etcd.yaml` | манифест etcd: headless Service, StatefulSet |
| `manifests/patroni.yaml` | манифест Patroni: Service, Secret, ConfigMap, StatefulSet, RBAC |
| `manifests/haproxy.yaml` | манифест HAProxy: ConfigMap, Deployment, Service |
| `manifests/pdb.yaml` | PodDisruptionBudget для etcd и Patroni |
| `manifests/monitoring.yaml` | манифест Prometheus: RBAC, PVC, ConfigMap, Deployment, Service |
| `manifests/grafana.yaml` | манифест Grafana: PVC, ConfigMap источника данных, Deployment, Service |
 
## Развёртывание
 
### 1. Установка k3s
 
На первом сервере:
 
```bash
curl -sfL https://get.k3s.io | sh -
sudo cat /var/lib/rancher/k3s/server/node-token   # токен для агентов
```
 
На втором и третьем сервере:
 
```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://<ip первого сервера>:6443 K3S_TOKEN=<токен> sh -
```
 
### 2. Разметка зон
 
```bash
kubectl label node patroni-1 topology.kubernetes.io/zone=zone-a
kubectl label node patroni-2 topology.kubernetes.io/zone=zone-b
kubectl label node patroni-3 topology.kubernetes.io/zone=zone-c
 
kubectl get nodes -L topology.kubernetes.io/zone
```
 
### 3. Сборка и доставка образа Patroni
 
```bash
docker build -f dockerfile_patroni -t patroni-custom:16 .
docker save patroni-custom:16 -o patroni.tar
```
 
> **Внимание.** Образ Patroni должен быть разлит на **каждый** из серверов —
> под может быть запущен на любом узле, а k3s использует containerd
> и не видит локальное хранилище Docker.
 
На каждом сервере:
 
```bash
sudo k3s ctr images import patroni.tar
sudo k3s ctr images ls | grep patroni-custom
```
 
### 4. Применение манифестов
 
> **Внимание.** Разворачивать необходимо в строгой последовательности:
> Patroni при старте сразу обращается к etcd.
 
```bash
kubectl apply -f etcd.yaml
kubectl get pods -l app=etcd -w        # дождаться трёх Running
 
kubectl apply -f patroni.yaml
kubectl apply -f haproxy.yaml
kubectl apply -f pdb.yaml
kubectl apply -f monitoring.yaml
kubectl apply -f grafana.yaml
```
 
## Проверка
 
```bash
kubectl exec etcd-0 -- etcdctl member list -w table
kubectl exec patroni-0 -c patroni -- patronictl -c /etc/patroni.yaml list
```
 
Ожидаемый результат: три члена etcd и три узла Patroni — один `Leader`
и две реплики в состоянии `streaming` с нулевым лагом.
 
## Сценарии проверки отказоустойчивости
 
| № | Сценарий | Что проверяется |
|---|---|---|
| 1 | Удаление пода лидера Patroni | автоматический failover, возврат узла репликой |
| 2 | Плановый вывод узла (`kubectl drain`) | соблюдение PodDisruptionBudget |
| 3 | Отказ зоны | сохранение кворума etcd (2 из 3) и восстановление записи |
| 4 | Отказ зоны с control plane | работа СУБД при недоступном API Kubernetes |

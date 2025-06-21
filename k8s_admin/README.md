# УСЛОВИЯ ЗАДАЧИ

Ваша организация купила ПО, и в процессе выяснилось, что оно ориентировано только на эксплуатацию в Kubernetes, и у вендора нет альтернативных способов инсталляции.
Вам нужно подготовить надёжный кластер, чтобы внедрение ПО прошло успешно. При этом вендор не хочет ничего делать. Поэтому вы должны дать ему базовые инструменты, иначе он будет просить вас почитать логи из подов.
Напомним, что ноды переименовывать нельзя, это может сломать интеграцию

# ВАШИ ДЕЙСТВИЯ
1.	Используя kubespray, разверните кластер, где один сервер выделен для control plane, два для worker nodes.
2.	Включите HA-режим. IP-адрес балансировщика будет предоставлен при создании инфраструктуры.
3.	Включите cert-manager. Создайте clusterissuer на продуктовые сервера let’s encrypt.
4.	Включите local-path-provisioner.

## Дополнительные модули
1.	Разверните Ingress-контроллер ingress-nginx из официального helm-чарта (после установки запустите автопроверку разово, она настроит интеграцию с облаком)
2.	Настройте сборку логов при помощи EFK/Opensearch.
    -	Fluentd должен быть развёрнут в namespace logging, elastic/kibana в namespace elasticsearch.
3.	Подготовьте и разверните helm-чарты, проверьте работоспособность.
4.	Разверните в namespace prometheus prometheus-оператор и проверьте наличие стандартных правил алертинга.
5.	Если будете публиковать какие-либо веб-интерфейсы на ingress, закройте их при помощи basic-auth.
    -	По желанию также можете получить сертификаты на эти эндпоинты.

## Резервное копирование
1.	Настройте резервное копирование etcd. Пусть сохраняется в папку и хранится 5 последних копий.
    -	Чтобы сделать это, напишите скрипт, добавьте его в cron и проверьте, что происходит выгрузка etcd.

## Немспейс team-one
Создайте неймспейс team-one и настройте LimitRange и ResourceQuota, где:
-	суммарное ограничение на неймспейс не более 4х CPU и 4 GB RAM;
-	минимальные запросы для контейнера 100m и 64 MB RAM;
-	стандартный запрос контейнеров на ресурсы минимален;
-	максимальное потребление для контейнера — 1 CPU и 1 GB.
________________________________________
# ФИНАЛЬНОЕ ЗАДАНИЕ (АВТОПРОВЕРКА)
Это объёмное и многогранное практическое задание. Мы разделили его проверку на автоматическую и ручную.

## Автоматическая проверка
Автоматически мы проверим:
-	наличие 1 control plane и 2 воркер-ноды,
-	работу балансировщика,
-	ресурсы и лимиты в неймспейсе team-one,
-	наличие cert-manager,
-	наличие Ingresclasses.

## Финальное задание (проверка ментором)
Ручная проверка
Для детальной проверки ментором приложите скриншоты всех команд из списка ниже:
-	kubectl get nodes -o wide;
-	kubectl get all -n elasticsearch;
-	kubectl get all -n prometheus;
-	kubectl get all -n logging;
-	kubectl get pv;
-	kubectl get pvc -A.

Также просьба оформить в репозиторий все использованные компоненты и прислать ссылку

--------------------------------------------------------------------------------
# ВЫПОЛНЕНИЕ

### Установим helm

```bash
# user
curl -sLO https://get.helm.sh/helm-v3.17.2-linux-amd64.tar.gz
tar -xf helm-v3.17.2-linux-amd64.tar.gz
sudo cp linux-amd64/helm /bin
sudo chmod +x /bin/helm

helm version
```

<!-- На node1 поместите kubeconfig в файл /root/.kube/config так, чтобы под root работала команда kubectl get nodes.
Подкинем себе kubeconfig:

```bash
ls -la /etc/kubernetes/
mkdir -p /root/.kube
cp -i /etc/kubernetes/admin.conf /root/.kube/config
chown $(id -u):$(id -g) /root/.kube/config
``` -->

### Настроим autocompletion (под рутом):
```bash
sudo -i
mkdir -p /etc/bash_completion.d
helm completion bash > /etc/bash_completion.d/helm
source <(helm completion bash)
```

### Сгенерируем ключи

```bash
# user
ssh-keygen
```
### раскидаем их по соседям cnidubp
```bash
ssh-copy-id user@84.252.142.218
ssh-copy-id user@89.169.177.181
```

### Клонирование репозитория Kubespray, переход в него, установка зависимостей

```bash
sudo -i
apt install python3-pip
apt install tree
```

```bash
# user
git clone https://github.com/kubernetes-sigs/kubespray
cd kubespray
sudo pip3 install -r requirements.txt
```

### Создадим инвентори с нашими хостами, где один сервер выделен для control plane, два для worker nodes.

```bash
vim inventory/sample/hosts.ini

[all]
kube-v3a-17-1-ibctn ansible_connection=local
kube-v3a-17-2-stblu ansible_host=10.129.0.15
kube-v3a-17-3-vvcuo ansible_host=10.129.0.43

[kube_control_plane]
kube-v3a-17-1-ibctn

[etcd]
kube-v3a-17-1-ibctn

[kube_node]
kube-v3a-17-2-stblu
kube-v3a-17-3-vvcuo

[calico_rr]

[k8s_cluster:children]
kube_node
kube_control_plane
calico_rr
```

### Вносим изменения в `inventory/sample/group_vars/all/all.yml`
- Включите HA-режим. IP-адрес балансировщика будет предоставлен при создании инфраструктуры.

```bash
vim inventory/sample/group_vars/all/all.yml

apiserver_loadbalancer_domain_name: "84.201.148.73"
loadbalancer_apiserver:
  address: "84.201.148.73"
  port: 6443
```

6. Запуск развертывания

```bash
ansible-playbook -i inventory/sample/hosts.ini cluster.yml -u user -b -e @inventory/local/vars.yaml

sudo -i
kubectl get nodes
kubectl get pods -A
kubeadm certs check-expiration
```

- Включите cert-manager. Создайте clusterissuer на продуктовые сервера let’s encrypt.

```bash
sudo -i
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
kubectl apply --validate=false -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.yaml

# Проверим:
kubectl get pods --namespace cert-manager
kubectl -n cert-manager get all
```

### Создадим новый clusterIssuer
```bash
vim letsencrypt.yaml

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    email: ov@fevlake.com
    privateKeySecretRef:
      name: letsencrypt-private-key
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
       ingress:
         class: nginx

# Применим:
sudo kubectl apply -f letsencrypt.yaml
sudo kubectl describe clusterissuer letsencrypt
```

- Включите local-path-provisioner.

```bash
vim inventory/local/vars.yaml

ingress_nginx_enabled: true
local_path_provisioner_enabled: true
dashboard_enabled: true
```

### Создание DNS-записи типа A в cloudflare (заменить ip и dns имя)
```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/9152ec3c08b1a4faeaa95353a929fcc5/dns_records" -H "Authorization: Bearer r6EhzlIilHr0fUu9BRBVm7zV4GPkb6hcaVsvYBb5" -H "Content-Type:application/json" --data '{"type":"A","name":"ingress.kube-v3a-17-1-ibctn","content":"84.201.178.78","proxied":false}'
```

-----------------------------------------------------------------------------
# Дополнительные модули
1.	Разверните Ingress-контроллер ingress-nginx из официального helm-чарта (после установки запустите автопроверку разово, она настроит интеграцию с облаком)

```bash
sudo kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
sudo kubectl get services -n ingress-nginx
```

2.	Настроим сборку логов при помощи EFK/Opensearch.
    -	Fluentd должен быть развёрнут в namespace logging, elastic/kibana в namespace elasticsearch.
3.	Подготовьте и разверните helm-чарты, проверьте работоспособность.


```bash
cd ..
sudo -i
mkdir -p /opt/nfs
apt install nfs-kernel-server -y
echo "/opt/nfs *(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports
systemctl enable nfs-kernel-server.service --now
systemctl reload nfs-kernel-server.service

# проверим, что папка расшарилась
showmount -e 127.0.0.1

# на всех нодах (1-2-3), которые могут монтировать диск, необходимо установить пакет nfs-common
sudo apt install -y nfs-common

# на ноде 1 устанавливаем провижионер для nfs
sudo helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner

sudo helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner  \
   --set nfs.server=84.201.178.78 \
   --set nfs.path=/opt/nfs \
   -n nfs-provisioner --create-namespace

# проверим: под есть, storageclass есть
sudo kubectl -n nfs-provisioner get pods,storageclasses

git clone https://github.com/elastic/cloud-on-k8s

sudo helm upgrade --install -n elasticsearch --create-namespace elastic-operator \
--set config.containerRegistry=docker.io \
--set config.containerRepository=elastic \
--set image.repository=docker.io/elastic/eck-operator \
--set image.tag=2.14.0 \
./cloud-on-k8s/deploy/eck-operator

sudo kubectl -n elasticsearch get pods

cd cloud-on-k8s/deploy/eck-stack

# подготовим values-override.yaml
vim values-override.yaml

eck-elasticsearch:
  fullnameOverride: elastic
  version: 8.13.4 elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 2Gi
        storageClassName: nfs-client
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 4Gi
              cpu: 1
            limits:
              memory: 8Gi
              cpu: 2

eck-kibana:
  fullnameOverride: kibana
  version: 8.13.4
  spec:
    count: 1
    elasticsearchRef:
      name: elastic
      namespace: elasticsearch
    http:
      service:
        spec:
          type: ClusterIP
    podTemplate:
      spec:
        containers:
        - name: kibana
          env:
            - name: NODE_OPTIONS
              value: "--max-old-space-size=2048"
          resources:
            requests:
              memory: 1Gi
              cpu: 0.5
            limits:
              memory: 2.5Gi
              cpu: 2

sudo helm upgrade --install -n elasticsearch --create-namespace -f values-override.yaml eck .
sudo kubectl -n elasticsearch get pods

# Настройка fluent-bit
vim fluent-bit.yaml

config:
  outputs: |
    [OUTPUT]
        Name es
        Match kube.*
        Host elastic-es-http.logging.svc.cluster.local
        Port 9200
        tls On
        tls.verify False
        HTTP_User elastic
        HTTP_Passwd W8m8iDffObb5DS042716QV0O
        Suppress_Type_Name On
        Logstash_Format On
        Retry_Limit False

    [OUTPUT]
        Name es
        Match host.*
        Host elastic-es-http.logging.svc.cluster.local
        Port 9200
        tls On
        tls.verify False
        HTTP_User elastic
        HTTP_Passwd W8m8iDffObb5DS042716QV0O
        Suppress_Type_Name On
        Logstash_Format On
        Logstash_Prefix node
        Retry_Limit False


sudo helm repo add fluent https://fluent.github.io/helm-charts
sudo kubectl create namespace logging
sudo helm upgrade --install fluent-bit fluent/fluent-bit -n logging -f fluent-bit.yaml
sudo kubectl -n logging get pods
```

4.	Разверните в namespace prometheus prometheus-оператор и проверьте наличие стандартных правил алертинга.

```bash
sudo helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
sudo helm repo update
sudo helm install prometheus prometheus-community/kube-prometheus-stack -n prometheus --create-namespace
sudo kubectl get prometheusrules.monitoring.coreos.com -n prometheus
```
------------------------------------------------------------------------------

### Резервное копирование
Настройте резервное копирование etcd. Пусть сохраняется в папку и хранится 5 последних копий.
Чтобы сделать это, напишите скрипт, добавьте его в cron и проверьте, что происходит выгрузка etcd.

1.	Скрипт для резервного копирования:
```bash
cd kubespray
vim etcd-backup.sh

#!/bin/bash

# Настройки по умолчанию
BACKUP_DIR="${BACKUP_DIR:-/var/backups/etcd}"
MAX_BACKUPS="${MAX_BACKUPS:-5}"

# Лог-файл
LOG_FILE="$BACKUP_DIR/etcd-backup.log"

# Текущая дата
DATE=$(date +%Y%m%d%H%M%S)

# Функция логирования
log() {
  echo "$(date +%Y-%m-%d_%H:%M:%S) - $1" >> "$LOG_FILE"
}

# Поиск etcdctl
if ! ETCDCTL_PATH=$(which etcdctl); then
  log "ERROR: etcdctl не найден в PATH.  Пожалуйста, укажите полный путь в переменной ETCDCTL_PATH."
  exit 1
fi

# Установите переменные окружения etcdctl, если необходимо
export ETCDCTL_API=3
# export ETCDCTL_ENDPOINTS="https://etcd1:2379,https://etcd2:2379"

# Создаем папку для бэкапов, если не существует
mkdir -p "$BACKUP_DIR"
if [ $? -ne 0 ]; then
  log "ERROR: Не удалось создать папку $BACKUP_DIR"
  exit 1
fi

# Проверка доступности etcd
log "INFO: Проверка доступности etcd"
"$ETCDCTL_PATH" endpoint health > /dev/null 2>&1
if [ $? -ne 0 ]; then
  log "ERROR: etcd недоступен.  Проверьте состояние etcd кластера."
  exit 1
fi
log "INFO: etcd доступен"

# Выполняем бэкап
log "INFO: Запуск резервного копирования etcd"
"$ETCDCTL_PATH" snapshot save "$BACKUP_DIR/etcd-snapshot-$DATE.db"
if [ $? -ne 0 ]; then
  log "ERROR: Ошибка при создании snapshot"
  exit 1
fi
log "INFO: Snapshot успешно создан: $BACKUP_DIR/etcd-snapshot-$DATE.db"

# Удаляем старые бэкапы, оставляя только последние $MAX_BACKUPS
log "INFO: Удаление старых бэкапов (сохраняем $MAX_BACKUPS последних)"
find "$BACKUP_DIR" -name "etcd-snapshot-*.db" -type f -printf '%T@ %p\n' | sort -n | head -n -$MAX_BACKUPS | cut -d' ' -f2- | xargs -r rm
if [ $? -ne 0 ]; then
  log "WARNING: Возникли проблемы при удалении старых бэкапов"
fi

log "INFO: Резервное копирование завершено"

exit 0
```

Сделаем скрипт исполняемым:
```bash
chmod +x etcd-backup.sh
```

Добавим его в cron для ежедневного выполнения. Откройте crontab:
```bash
crontab -e
```
и добавьте строку:
```bash
0 2 * * * ./etcd-backup.sh >/var/backups/etcd 2>&1
```
2. Проверим работу скрипта
```bash
./etcd-backup.sh
```

#### Проверим папку
```bash
ls -l /var/backups/etcd
cat /var/backups/etcd/etcd-backup.log
```
---------------------------------------------------------------

# Немспейс team-one
Создайте неймспейс team-one и настройте LimitRange и ResourceQuota, где:
- суммарное ограничение на неймспейс не более 4х CPU и 4 GB RAM;
- минимальные запросы для контейнера 100m и 64 MB RAM;
- стандартный запрос контейнеров на ресурсы минимален;
- максимальное потребление для контейнера — 1 CPU и 1 GB.

1. Создание namespace team-one
```bash
sudo kubectl create namespace team-one
```

2.	Создание LimitRange
```bash
vim limitrange.yaml

apiVersion: v1
kind: LimitRange
metadata:
  name: team-one-limits
  namespace: team-one
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 64Mi
    max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: 100m
      memory: 64Mi

sudo kubectl apply -f limitrange.yaml -n team-one
```

3.	Создание ResourceQuota

```bash
vim resourcequota.yaml

apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-one-quota
  namespace: team-one
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"


sudo kubectl apply -f resourcequota.yaml -n team-one
```
-------------------------------------------------------------------------------

# Ручная проверка
```bash
kubectl get nodes -o wide
kubectl get all -n elasticsearch
kubectl get all -n prometheus
kubectl get all -n logging
kubectl get pv
kubectl get pvc -A
```



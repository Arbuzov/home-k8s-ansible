# Troubleshooting Guide

## Общие проблемы и решения

### 1. SSH подключение

**Проблема**: Не удается подключиться к Raspberry Pi

**Решение**:
```bash
# Проверить доступность хоста
ping 192.168.1.100

# Проверить SSH сервис
ssh pi@192.168.1.100 'systemctl status ssh'

# Проверить SSH ключи
ssh-copy-id pi@192.168.1.100
```

### 2. Ошибки containerd

**Проблема**: containerd не запускается

**Решение**:
```bash
# Проверить статус containerd
ansible all -m systemd -a "name=containerd" -b

# Перезапустить containerd
ansible all -m systemd -a "name=containerd state=restarted" -b

# Проверить логи containerd
ansible all -m shell -a "journalctl -u containerd --no-pager -l" -b
```

### 3. Проблемы с Kubernetes

**Проблема**: Kubelet не запускается

**Решение**:
```bash
# Проверить статус kubelet
ansible all -m systemd -a "name=kubelet state=started enabled=yes" -b

# Проверить логи kubelet
ansible all -m shell -a "journalctl -u kubelet --no-pager -l" -b

# Перезапустить kubelet
ansible all -m systemd -a "name=kubelet state=restarted" -b
```

### 4. Проблемы с памятью на Raspberry Pi

**Проблема**: Недостаточно памяти для подов

**Решение**:
- Увеличить swap (хотя это не рекомендуется для Kubernetes)
- Уменьшить ресурсные лимиты для подов
- Проверить настройки памяти в `/boot/config.txt`

### 5. Сетевые проблемы

**Проблема**: Поды не могут подключиться друг к другу

**Решение**:
```bash
# Проверить Flannel
kubectl get pods -n kube-flannel

# Проверить сетевые политики
kubectl get networkpolicies --all-namespaces

# Перезапустить сетевые компоненты
kubectl delete pods -n kube-flannel -l app=flannel
```

### 6. Проблемы с join worker нод

**Проблема**: Worker ноды не могут присоединиться к кластеру

**Решение**:
```bash
# Сгенерировать новый join token
kubeadm token create --print-join-command

# Проверить сертификаты
openssl x509 -in /etc/kubernetes/pki/ca.crt -text -noout

# Проверить время на всех нодах
ansible all -m shell -a "date"
```

## Команды для диагностики

### Проверить состояние кластера

```bash
ansible-playbook playbooks/maintenance.yml --tags check
ansible-playbook playbooks/test-cluster.yml
```

### Проверить ресурсы

```bash
ansible-playbook playbooks/maintenance.yml --tags resources
```

### Перезапустить сервисы

```bash
ansible-playbook playbooks/maintenance.yml --tags restart
```

### Очистить систему

```bash
ansible-playbook playbooks/maintenance.yml --tags cleanup
```

## Логи для анализа

### Системные логи
```bash
# Журналы системы
ansible all -m shell -a "journalctl --since '1 hour ago' --no-pager"

# Логи containerd
ansible all -m shell -a "journalctl -u containerd --since '1 hour ago' --no-pager"

# Логи Kubelet
ansible all -m shell -a "journalctl -u kubelet --since '1 hour ago' --no-pager"
```

### Kubernetes логи
```bash
# Логи системных подов
kubectl logs -n kube-system -l component=kube-apiserver
kubectl logs -n kube-system -l component=kube-controller-manager
kubectl logs -n kube-system -l component=kube-scheduler

# Логи Flannel
kubectl logs -n kube-flannel -l app=flannel
```

## Мониторинг производительности

### CPU и память
```bash
# Использование ресурсов нод
kubectl top nodes

# Использование ресурсов подов
kubectl top pods --all-namespaces
```

### Дисковое пространство
```bash
# Проверить дисковое пространство
ansible all -m shell -a "df -h"

# Очистить контейнерные образы (через ctr)
ansible all -m shell -a "ctr -n k8s.io image prune" -b
```

## Полезные команды kubectl

```bash
# Проверить состояние нод
kubectl get nodes -o wide

# Проверить состояние подов
kubectl get pods --all-namespaces

# Описать проблемную ноду
kubectl describe node <node-name>

# Проверить события
kubectl get events --sort-by=.metadata.creationTimestamp

# Проверить ресурсы кластера
kubectl get all --all-namespaces
```

# Статус кластера Kubernetes

## Текущее состояние нод (после обновлений)

| Нода | IP адрес | Роль | Версия K8s | Статус |
|------|----------|------|------------|--------|
| kube-master | 192.168.99.44 | master,worker | **1.33.1** | ⚠️ Можно обновить |
| kube-worker-1 | 192.168.99.101 | worker | **1.33.4** | ✅ Обновлена |
| kube-worker-2 | 192.168.99.102 | worker | **1.33.4** | ✅ Обновлена |

## Выполненные обновления

### ✅ kube-worker-2: 1.33.1 → 1.33.4
- Дата: 16 августа 2025
- Метод: `ansible-playbook playbooks/update-worker-version.yml -l kube-worker-2`
- Результат: Успешно обновлена до v1.33.4

### ✅ kube-worker-1: 1.33.1 → 1.33.4
- Дата: 16 августа 2025
- Метод: `ansible-playbook playbooks/update-worker-version.yml -e target_node=kube-worker-1`
- Результат: Успешно обновлена до v1.33.4

## Рекомендации

### Следующие шаги:
1. **Обновить master ноду** до версии 1.33.4 для единообразия
2. **Проверить работу приложений** после обновления worker нод
3. **Мониторить производительность** кластера

### Команды для проверки:
```bash
# Проверить статус всех нод
ansible kube-master -m shell -a "sudo -u pi kubectl get nodes -o wide" -e @credentials.json

# Проверить системные ресурсы
ansible all -m shell -a "free -h && df -h /var" -e @credentials.json

# Проверить состояние pod'ов
ansible kube-master -m shell -a "sudo -u pi kubectl get pods --all-namespaces" -e @credentials.json
```

## Version Skew Policy

Текущая конфигурация соответствует Kubernetes version skew policy:
- Worker ноды (1.33.4) могут быть **на одну минорную версию** новее master ноды (1.33.1)
- Это допустимая конфигурация ✅

## Заметки
- Все обновления выполнены с использованием плейбука `update-worker-version.yml`
- Использовался безопасный метод с drain/uncordon нод
- Все пакеты Kubernetes заблокированы после обновления
- Конфигурация kubelet обновлена автоматически

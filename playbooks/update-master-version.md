# Update Master Version Playbook

## Описание
Плейбук `update-master-version.yml` предназначен для безопасного обновления master ноды Kubernetes кластера.

## Особенности
- ✅ **Поэтапное обновление**: kubeadm → control-plane → kubelet/kubectl
- ✅ **Автоматическое планирование** с отображением изменений
- ✅ **Обновление control-plane компонентов**: etcd, kube-apiserver, kube-controller-manager, kube-scheduler
- ✅ **Проверка здоровья** API сервера и компонентов
- ✅ **Разблокировка/блокировка** held packages
- ✅ **Проверка статуса** после обновления

## Использование

### Основное обновление
```bash
# Обновить master до версии из inventory.yml
ansible-playbook playbooks/update-master-version.yml \
  -e target_node=kube-master \
  -e @credentials.json
```

### Проверка перед обновлением
```bash
# Dry-run для проверки
ansible-playbook playbooks/update-master-version.yml \
  -e target_node=kube-master \
  -e @credentials.json \
  --check --diff
```

## Процесс обновления

1. **Подготовка**: Проверка, что это master нода
2. **Unhold**: Разблокировка Kubernetes пакетов
3. **kubeadm Update**: Обновление kubeadm до новой версии
4. **Upgrade Plan**: Отображение плана обновления
5. **Control Plane**: Обновление компонентов control-plane:
   - etcd
   - kube-apiserver
   - kube-controller-manager
   - kube-scheduler
6. **kubelet/kubectl**: Обновление kubelet и kubectl
7. **Hold**: Блокировка пакетов на новой версии
8. **Restart**: Перезапуск kubelet
9. **Health Check**: Проверка API сервера и компонентов
10. **Verify**: Проверка итогового статуса

## Компоненты которые обновляются

### Control Plane:
- **etcd**: Хранилище данных кластера
- **kube-apiserver**: API сервер Kubernetes
- **kube-controller-manager**: Контроллеры кластера
- **kube-scheduler**: Планировщик подов

### Node Components:
- **kubelet**: Агент на ноде
- **kubectl**: CLI клиент
- **kubeadm**: Инструмент управления кластером

## Переменные

- `target_node`: Имя master ноды (по умолчанию: kube-master)
- `kubernetes_version`: Берется из inventory.yml
- `kubernetes_major_minor`: Берется из inventory.yml
- `node_role`: Должен содержать 'master'

## Требования

- Доступ к master ноде с правами sudo
- Корректно настроенный inventory.yml с версиями K8s
- Стабильное сетевое соединение (обновление может занять несколько минут)

## Пример успешного выполнения

```
TASK [Display final master node status] *****************************************************
ok: [kube-master] => {
    "msg": [
        "NAME          STATUS   ROLES                  AGE     VERSION   ...",
        "kube-master   Ready    control-plane,worker   2y35d   v1.33.4   ..."
    ]
}

TASK [Display component status] *************************************************************
ok: [kube-master] => {
    "msg": [
        "NAME                 STATUS    MESSAGE   ERROR",
        "scheduler            Healthy   ok        ",
        "controller-manager   Healthy   ok        ",
        "etcd-0               Healthy   ok        "
    ]
}
```

## Важные замечания

⚠️ **Порядок обновления**: Всегда обновляйте master ноду ПОСЛЕДНЕЙ, после всех worker нод

✅ **Безопасность**: Плейбук создает резервные копии всех манифестов в `/etc/kubernetes/tmp/`

🔄 **Откат**: В случае проблем можно восстановить старые манифесты из backup директории

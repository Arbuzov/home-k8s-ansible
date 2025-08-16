# Update Worker Version Playbook

## Описание
Плейбук `update-worker-version.yml` предназначен для быстрого обновления версии Kubernetes на worker нодах.

## Особенности
- ✅ **Автоматически использует версии из inventory.yml**
- ✅ **Безопасное обновление** с drain/uncordon ноды
- ✅ **Разблокировка held packages** перед обновлением
- ✅ **Блокировка пакетов** после обновления
- ✅ **Обновление kubelet конфигурации**
- ✅ **Проверка статуса** после обновления

## Использование

### Быстрое обновление
```bash
# Обновить kube-worker-2 до версии из inventory.yml
ansible-playbook playbooks/update-worker-version.yml \
  -l kube-worker-2 \
  -e @credentials.json
```

### Обновление с параметрами
```bash
# Обновить без drain/uncordon
ansible-playbook playbooks/update-worker-version.yml \
  -l kube-worker-1 \
  -e @credentials.json \
  -e drain_node=false

# Указать конкретную ноду
ansible-playbook playbooks/update-worker-version.yml \
  -e target_node=kube-worker-3 \
  -e @credentials.json
```

## Процесс обновления

1. **Подготовка**: Отображение информации о ноде
2. **Drain**: Вывод ноды из эксплуатации (опционально)
3. **Unhold**: Разблокировка Kubernetes пакетов
4. **Update**: Обновление kubelet, kubeadm, kubectl
5. **Hold**: Блокировка пакетов на новой версии
6. **Configure**: Обновление конфигурации kubelet
7. **Restart**: Перезапуск kubelet сервиса
8. **Wait**: Ожидание готовности kubelet
9. **Uncordon**: Возврат ноды в работу (опционально)
10. **Verify**: Проверка итогового статуса

## Переменные

- `target_node`: Имя ноды для обновления (по умолчанию: kube-worker-2)
- `drain_node`: Выводить ли ноду из работы (по умолчанию: true)
- `kubernetes_version`: Берется из inventory.yml для каждой ноды
- `kubernetes_major_minor`: Берется из inventory.yml для каждой ноды

## Требования

- Доступ к master ноде для drain/uncordon операций
- SSH доступ к target ноде
- Корректно настроенный inventory.yml с версиями K8s

## Пример успешного выполнения

```
TASK [Display final node status] ****************************************************
ok: [kube-worker-2] => {
    "msg": [
        "NAME            STATUS   ROLES    AGE   VERSION   ...",
        "kube-worker-2   Ready    worker   65m   v1.33.4   ..."
    ]
}
```

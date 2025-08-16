# Install Worker with Join Playbook

## Описание
Плейбук `install-worker-with-join.yml` автоматизирует полный процесс установки и присоединения worker ноды к существующему Kubernetes кластеру.

## Особенности
- ✅ **Полная установка** всех компонентов на worker ноду
- ✅ **Генерация токена** на мастере без обновлений мастера
- ✅ **Отключение swap** и настройка cgroups
- ✅ **Присоединение к кластеру** без игнорирования preflight проверок
- ✅ **Автоматическое добавление метки** `node-role.kubernetes.io/worker`

## Использование

### Базовое использование
```bash
ansible-playbook playbooks/install-worker-with-join.yml \
  --limit новая-worker-нода,мастер-нода \
  -e @credentials.json
```

### С конкретными нодами
```bash
ansible-playbook playbooks/install-worker-with-join.yml \
  --limit pi3-worker2,pi4-master \
  -e @credentials.json
```

### Только отдельные этапы (теги)
```bash
# Только установка компонентов
ansible-playbook playbooks/install-worker-with-join.yml \
  --tags worker-setup \
  --limit pi3-worker2 \
  -e @credentials.json

# Только генерация токена
ansible-playbook playbooks/install-worker-with-join.yml \
  --tags join-token \
  --limit pi4-master \
  -e @credentials.json

# Только присоединение
ansible-playbook playbooks/install-worker-with-join.yml \
  --tags worker-join \
  --limit pi3-worker2,pi4-master \
  -e @credentials.json

# Только добавление метки
ansible-playbook playbooks/install-worker-with-join.yml \
  --tags label \
  --limit pi3-worker2,pi4-master \
  -e @credentials.json
```

## Этапы выполнения

### 1. Установка компонентов (worker-setup)
- Выполнение роли `common` (cgroups, swap, системные настройки)
- Выполнение роли `containerd` (container runtime)
- Выполнение роли `kubernetes` (kubelet, kubeadm, kubectl)

### 2. Генерация токена (join-token)
- Проверка инициализации кластера на мастере
- Генерация нового join токена
- Сохранение токена на мастере и локально

### 3. Присоединение к кластеру (worker-join)
- Отключение swap
- Чтение команды join из локального файла
- Выполнение `kubeadm join` без игнорирования preflight
- Проверка статуса kubelet

### 4. Добавление метки (label)
- Добавление метки `node-role.kubernetes.io/worker=` на worker ноду

## Требования

### Для worker ноды
- Raspberry Pi OS 64-bit (Bookworm)
- SSH доступ с указанными кредами
- Интернет соединение

### Для master ноды
- Инициализированный кластер Kubernetes
- Доступный файл `/etc/kubernetes/admin.conf`
- Возможность выполнения `kubeadm token create`

### Общие
- Сетевая доступность между мастером и worker нодой
- Корректные кроссовые кредитные данные в `credentials.json`

## Переменные
Плейбук использует стандартные переменные из `group_vars/all.yml`:
- `kubernetes_version`: версия Kubernetes для установки
- `kubernetes_major_minor`: мажорная.минорная версия

## Проверка результата
После успешного выполнения:

```bash
# На мастере
kubectl get nodes

# Ожидаемый результат:
# NAME            STATUS   ROLES                  AGE   VERSION
# pi4-master      Ready    control-plane,worker   5d    v1.33.1
# pi3-worker1     Ready    worker                 5d    v1.33.1
# новая-нода      Ready    worker                 1m    v1.33.1
```

## Решение проблем

### Preflight ошибки
Плейбук **не игнорирует** preflight проверки. Если есть ошибки:
- Проверьте cgroups: `cat /proc/cmdline`
- Проверьте swap: `free -h`
- Проверьте containerd: `systemctl status containerd`

### Токен истёк
Если токен истёк, заново запустите только генерацию:
```bash
ansible-playbook playbooks/install-worker-with-join.yml \
  --tags join-token,worker-join \
  --limit новая-нода,мастер \
  -e @credentials.json
```

### Нода не присоединяется
1. Проверьте сетевую доступность между нодами
2. Убедитесь, что мастер доступен на порту 6443
3. Проверьте логи kubelet: `journalctl -u kubelet -f`

## Примечания
- Плейбук идемпотентен - можно запускать многократно
- Join токены действительны ограниченное время (по умолчанию 24 часа)
- После присоединения нода автоматически получает метку worker роли
- Для безопасности используйте SSH ключи вместо паролей в продакшене

# Master Role

## Описание
Роль `master` инициализирует мастер-ноду Kubernetes кластера и настраивает её для работы.

## Задачи
- Проверка инициализации кластера
- Инициализация Kubernetes кластера с помощью kubeadm
- Настройка kubeconfig для root и пользователя pi
- Удаление taint с мастер-ноды (опционально)
- Установка Flannel CNI
- **Генерация и сохранение join токена**
- Ожидание готовности кластера

## Переменные
- `kubernetes.pod_subnet`: подсеть для подов (по умолчанию: "10.244.0.0/16")
- `kubernetes.service_subnet`: подсеть для сервисов (по умолчанию: "10.96.0.0/12")
- `kubernetes.api_server_advertise_address`: IP адрес API сервера
- `master_schedulable`: разрешить запуск подов на мастере (по умолчанию: true)
- `kubernetes_cni`: CNI плагин (по умолчанию: "flannel")

## Критически важные функции

### Инициализация кластера
```bash
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --apiserver-advertise-address=<IP> \
  --node-name=<hostname>
```

### Генерация join токена
Автоматически генерирует токен для присоединения worker нод:
```bash
kubeadm token create --print-join-command
```

Токен сохраняется:
- На мастере: `/tmp/kubeadm-join-command`
- Локально: `./kubeadm-join-command`

### CNI установка
Устанавливает Flannel CNI для сетевого взаимодействия между подами.

## Теги задач
- `master-init`: только инициализация кластера
- `master-join`: только генерация join токена

## Создаваемые файлы
- `/etc/kubernetes/admin.conf`: конфигурация администратора
- `/root/.kube/config`: kubeconfig для root
- `/home/pi/.kube/config`: kubeconfig для пользователя pi
- `/tmp/kubeadm-join-command`: команда для присоединения worker нод

## Проверка работы
```bash
# Статус кластера
kubectl get nodes

# Статус подов системы
kubectl get pods -n kube-system

# Проверка CNI
kubectl get pods -n kube-flannel
```

## Зависимости
- Роль `common`
- Роль `containerd`
- Роль `kubernetes`

## Примечания
- Роль выполняется только один раз при инициализации кластера
- После успешной инициализации файл `/etc/kubernetes/admin.conf` является признаком готового кластера
- Join токен действителен ограниченное время
- Для безопасности рекомендуется генерировать новые токены для каждого присоединения

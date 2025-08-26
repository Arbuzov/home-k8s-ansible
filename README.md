# Kubernetes Cluster Management with Ansible

Ansible репозиторий для управления локальным Kubernetes кластером на Raspberry Pi с использованием **containerd** и **Kubernetes 1.33+**.

## 🎯 Особенности

- **Kubernetes 1.33+** с официального репозитория pkgs.k8s.io
- **containerd** как container runtime (заменил Docker)
- **Flannel CNI** для сетевого взаимодействия
- **Автоматическая настройка cgroups** для Raspberry Pi OS Bookworm
- **Полное отключение swap** (требование Kubernetes)
- **Автоматическое присоединение worker нод** с генерацией токенов
- **ARM64 архитектура** (Raspberry Pi 4)

## 🏗️ Архитектура кластера

- **Master Node**: Raspberry Pi 4 (также выполняет роль worker)
- **Worker Nodes**: 2x Raspberry Pi 3/4

## 📋 Системные требования

- Raspberry Pi 4 (рекомендуется 4GB+ RAM для мастера)
- Raspberry Pi OS 64-bit (Bookworm)
- SSH доступ с паролем или ключами
- Интернет соединение на всех нодах

# Запуск тестов
make test

# Dry run на тестовом инвентаре
make dry-run

# Создание тестового кластера с kind
make setup-test-cluster

# Molecule тесты
make molecule-test
```

В контейнере доступны следующие команды (без использования Makefile):

```bash
# Проверка синтаксиса playbook
ansible-playbook --syntax-check site.yml

# Проверка линтерами
ansible-lint playbooks/ roles/

# Запуск тестов (dry run)
ansible-playbook site.yml --check --diff

# Molecule тесты (если требуется)
cd tests && molecule test

# Для тестирования кластера kind:
# (если требуется, настройте kind и используйте playbooks/test-cluster.yml)
ansible-playbook playbooks/test-cluster.yml
```

## Структура репозитория

```text
.
├── ansible.cfg                 # Конфигурация Ansible
├── credentials.json            # Плоские переменные для Ansible (игнорируется Git)
├── credentials.json.example    # Пример файла с переменными
├── inventory.yml               # Инвентарь серверов
├── site.yml                    # Основной playbook
├── requirements.yml            # Требования Ansible коллекций
├── group_vars/                 # Переменные для групп
│   ├── all.yml
│   ├── masters.yml
│   └── workers.yml
├── host_vars/                  # Переменные для хостов
├── roles/                      # Ansible роли
│   ├── common/                 # Базовая настройка
│   ├── containerd/             # Установка containerd
│   ├── kubernetes/             # Установка Kubernetes
│   ├── master/                 # Настройка master ноды
│   └── worker/                 # Настройка worker нод
└── playbooks/                  # Отдельные playbooks
    ├── install-cluster.yml
    ├── install-worker-with-join.yml  # Установка и присоединение worker ноды
    ├── update-cluster.yml
    ├── maintenance.yml
    └── test-cluster.yml
```

## 🔧 Роли Ansible

### `common` - Базовая настройка системы
- Обновление пакетов и настройка часового пояса
- **Установка hostname** из inventory (kube-master, kube-worker-1, kube-worker-2)
- **Настройка cgroups** (критично для Kubernetes на RPi)
- **Полное отключение swap**
- Загрузка модулей ядра и sysctl настройки
- [Подробная документация](roles/common/README.md)

### `containerd` - Container Runtime
- Установка containerd.io из Docker репозитория
- **Настройка CRI API** (исправление disabled_plugins)
- **Включение SystemdCgroup** для Kubernetes
- [Подробная документация](roles/containerd/README.md)

### `kubernetes` - Установка Kubernetes
- Установка kubelet, kubeadm, kubectl версии 1.33+
- **Официальный репозиторий pkgs.k8s.io**
- Правильная настройка systemd для kubelet
- [Подробная документация](roles/kubernetes/README.md)

### `master` - Мастер нода
- Инициализация кластера с kubeadm
- Установка Flannel CNI
- **Автоматическая генерация join токенов**
- [Подробная документация](roles/master/README.md)

### `worker` - Worker ноды
- Присоединение к кластеру с kubeadm join
- **Автоматическое добавление меток ролей**
- Проверка статуса присоединения
- [Подробная документация](roles/worker/README.md)

## 🚀 Плейбуки

### Основной плейбук `site.yml`
Полная установка кластера на все ноды:
```bash
ansible-playbook site.yml
```

### Установка и присоединение worker ноды
**Новый плейбук** для быстрого добавления worker нод:
```bash
ansible-playbook playbooks/install-worker-with-join.yml --limit новая-нода,мастер
```

Особенности:
- ✅ Установка всех компонентов на worker ноду
- ✅ Генерация токена на мастере (без обновлений мастера)
- ✅ Отключение swap и настройка cgroups
- ✅ Присоединение к кластеру без игнорирования preflight проверок
- ✅ Автоматическое добавление метки `node-role.kubernetes.io/worker`

## � Конфигурация переменных

### Файл credentials.json
Все переменные используют плоскую структуру для простого подключения:

```json
{
  "ansible_user": "pi",                                    // SSH пользователь
  "ansible_password": "YOUR_PASSWORD",                     // SSH пароль (опционально)
  "ansible_ssh_private_key_file": "~/.ssh/id_rsa",        // SSH ключ (опционально)
  "ansible_ssh_public_key_file": "~/.ssh/id_rsa.pub",     // Публичный SSH ключ
  "kubernetes_cluster_name": "raspberry-k8s",             // Имя кластера
  "kubernetes_pod_subnet": "10.244.0.0/16",               // Подсеть для подов
  "kubernetes_service_subnet": "10.96.0.0/12",            // Подсеть для сервисов
  "kubernetes_api_server_advertise_address": "192.168.1.100",  // IP мастера
  "containerd_data_root": "/var/lib/containerd",           // Путь к данным containerd
  "registry_mirror": "https://registry-1.docker.io"       // Mirror реестра
}
```

### Использование переменных
```bash
# Подключение файла с переменными
ansible-playbook site.yml -e @credentials.json

# Переопределение отдельных переменных
ansible-playbook site.yml -e @credentials.json -e kubernetes_cluster_name=my-cluster
```

## �🚀 Быстрый старт

### Полная установка кластера

1. **Установите требования Ansible:**
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

2. **Создайте файл с кредами:**
   ```bash
   cp credentials.json.example credentials.json
   # Отредактируйте credentials.json с вашими параметрами
   ```

   Пример конфигурации `credentials.json`:
   ```json
   {
     "ansible_user": "pi",
     "ansible_password": "YOUR_PASSWORD_HERE",
     "ansible_ssh_private_key_file": "~/.ssh/id_rsa",
     "kubernetes_cluster_name": "raspberry-k8s",
     "kubernetes_pod_subnet": "10.244.0.0/16",
     "kubernetes_service_subnet": "10.96.0.0/12",
     "kubernetes_api_server_advertise_address": "192.168.1.100"
   }
   ```

3. **Настройте инвентарь:**
   ```bash
   # Отредактируйте inventory.yml с IP адресами ваших Raspberry Pi
   # Важно: версии Kubernetes задаются для каждой ноды индивидуально
   ```

   Пример конфигурации:
   ```yaml
   all:
     children:
       kubernetes_cluster:
         children:
           masters:
             hosts:
               kube-master:
                 ansible_host: 192.168.1.100
                 kubernetes_version: "1.33.1"
                 kubernetes_major_minor: "1.33"
           workers:
             hosts:
               kube-worker-1:
                 ansible_host: 192.168.1.101
                 kubernetes_version: "1.33.1"
                 kubernetes_major_minor: "1.33"
               kube-worker-2:
                 ansible_host: 192.168.1.102
                 kubernetes_version: "1.33.1"  # Можно указать разные версии
                 kubernetes_major_minor: "1.33"
   ```

4. **Проверьте доступность хостов:**
   ```bash
   ansible all -m ping -e @credentials.json
   ```

5. **Установите кластер:**
   ```bash
   # Полная установка
   ansible-playbook site.yml -e @credentials.json

   # Или по тегам
   ansible-playbook site.yml --tags install -e @credentials.json
   ```

### Быстрое добавление worker ноды

Для добавления новой worker ноды к существующему кластеру:

```bash
ansible-playbook playbooks/install-worker-with-join.yml \
  --limit новая-worker-нода,мастер-нода \
  -e @credentials.json
```

Этот плейбук автоматически:
- Установит все необходимые компоненты
- Сгенерирует join токен на мастере
- Присоединит worker ноду к кластеру
- Добавит метку worker роли

### Проверка результата

```bash
# На мастер ноде проверьте статус кластера
kubectl get nodes

# Ожидаемый результат:
# NAME            STATUS   ROLES                  AGE   VERSION
# kube-master     Ready    control-plane,worker   5d    v1.33.1
# kube-worker-1   Ready    worker                 5d    v1.33.1
# kube-worker-2   Ready    worker                 1m    v1.33.1
```

## ⚠️ Важные замечания

### Системные требования для Raspberry Pi
- **Обязательно**: Raspberry Pi OS 64-bit (Bookworm)
- **RAM**: Минимум 2GB для worker нод, 4GB+ для мастера
- **Swap**: Автоматически отключается ролями (требование K8s)
- **cgroups**: Автоматически настраиваются для правильной работы

### Критические настройки

#### Hostname настройка
Роль `common` автоматически:
- Устанавливает hostname системы из `inventory_hostname`
- Обновляет `/etc/hosts` для корректного разрешения имен
- Имена нод: `kube-master`, `kube-worker-1`, `kube-worker-2`

#### cgroups memory
Роли автоматически добавляют в `/boot/firmware/cmdline.txt`:
```
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```
⚠️ **Порядок параметров критически важен!**

#### containerd CRI
Роль `containerd` автоматически:
- Включает CRI плагин (удаляет из disabled_plugins)
- Настраивает SystemdCgroup = true
- Обеспечивает совместимость с Kubernetes 1.27+

#### preflight проверки
Плейбуки **не игнорируют** preflight проверки - все проблемы исправляются:
- ✅ Правильная настройка cgroups
- ✅ Полное отключение swap
- ✅ Работающий containerd CRI API
- ✅ Корректная конфигурация kubelet

### Версии и совместимость

#### Настройка версий Kubernetes
Версии Kubernetes задаются **индивидуально для каждой ноды** в `inventory.yml`:
```yaml
kube-master:
  kubernetes_version: "1.33.1"        # Точная версия пакета
  kubernetes_major_minor: "1.33"      # Мажорная.минорная для репозитория
```

#### Поддерживаемые версии
- **Kubernetes**: 1.27+ (рекомендуется 1.33+)
- **Container Runtime**: containerd (заменил Docker)
- **CNI**: Flannel (по умолчанию)
- **Архитектура**: ARM64

#### Version Skew Policy
Kubernetes поддерживает ограниченное различие версий между компонентами:
- **Master → Worker**: Worker ноды могут быть на 1 минорную версию старше мастера
- **Недопустимо**: Мастер старше worker нод
- **Рекомендация**: Используйте одинаковые версии для стабильности

#### Примеры конфигураций
```yaml
# Одинаковые версии (рекомендуется)
kube-master:    kubernetes_version: "1.33.1"
kube-worker-1:   kubernetes_version: "1.33.1"

# Допустимое различие
kube-master:    kubernetes_version: "1.33.1"  # Новая версия
kube-worker-1:   kubernetes_version: "1.32.5"  # Старая версия (OK)

# Недопустимо
kube-master:    kubernetes_version: "1.32.1"  # Старая версия
kube-worker-1:   kubernetes_version: "1.33.1"  # Новая версия (ERROR!)
```

### Безопасность
- Join токены имеют ограниченное время жизни
- Рекомендуется генерировать новые токены для каждого присоединения
- SSH кроссовые ключи не сохраняются в репозитории

## 🔄 Поштучное обновление нод

### Обновление одной ноды

```bash
# Быстрое обновление worker ноды (использует версию из inventory.yml)
ansible-playbook playbooks/update-worker-version.yml \
  -l kube-worker-2 \
  -e @credentials.json

# Обновить конкретную ноду до указанной версии
ansible-playbook playbooks/update-single-node.yml \
  -e target_host=kube-master \
  -e kubernetes_target_version=1.29

# С дополнительными опциями
ansible-playbook playbooks/update-single-node.yml \
  -e target_host=kube-worker-1 \
  -e kubernetes_target_version=1.29 \
  -e drain_node=true \
  -e update_system_packages=true
```

💡 **Совет**: Плейбук `update-worker-version.yml` автоматически использует версии из `inventory.yml`

### Обновление нескольких нод с разными версиями

1. Создайте файл конфигурации:

```bash
cp node-updates.yml.example node-updates.yml
# Отредактируйте node-updates.yml под ваши нужды
```

2. Запустите обновление:

```bash
ansible-playbook playbooks/update-multiple-nodes.yml -e @node-updates.yml
```

### Откат к предыдущей версии

```bash
# Откатить ноду к предыдущей версии
ansible-playbook playbooks/rollback-node.yml \
  -e target_host=kube-worker-1 \
  -e rollback_version=1.28
```

## Основные команды

### Управление кластером

```bash
# Установка кластера
ansible-playbook site.yml --tags install

# Обновление всего кластера
ansible-playbook playbooks/update-cluster.yml

# Обновление одной ноды до конкретной версии
ansible-playbook playbooks/update-single-node.yml -e target_host=kube-master -e kubernetes_target_version=1.29

# Обновление нескольких нод с разными версиями
ansible-playbook playbooks/update-multiple-nodes.yml -e @node-updates.yml

# Откат ноды к предыдущей версии
ansible-playbook playbooks/rollback-node.yml -e target_host=kube-worker-1 -e rollback_version=1.28

# Проверка состояния кластера
ansible-playbook playbooks/maintenance.yml --tags check

# Тестирование кластера
ansible-playbook playbooks/test-cluster.yml

# Проверка ресурсов системы
ansible-playbook playbooks/maintenance.yml --tags resources

# Очистка системы (APT)
ansible-playbook playbooks/maintenance.yml --tags cleanup
```

### Операции с нодами

```bash
# Обновление только worker нод
ansible-playbook site.yml --limit workers --tags update

# Обновление только master ноды
ansible-playbook site.yml --limit masters --tags update

# Перезапуск сервисов containerd и Kubelet
ansible-playbook playbooks/maintenance.yml --tags restart

# Проверка доступности всех хостов
ansible all -m ping

# Проверка статуса сервисов
ansible all -m systemd -a "name=containerd" -b
ansible all -m systemd -a "name=kubelet" -b
```

### Полезные команды для диагностики

```bash
# Проверить версии Kubernetes
ansible all -m shell -a "kubectl version --client --short" -b

# Проверить статус нод кластера
ansible masters -m shell -a "kubectl get nodes -o wide" -b

# Проверить системные ресурсы
ansible all -m shell -a "free -h && df -h" -b

# Посмотреть логи kubelet
ansible all -m shell -a "journalctl -u kubelet --no-pager -l" -b

# Посмотреть логи containerd
ansible all -m shell -a "journalctl -u containerd --no-pager -l" -b
## Используется containerd

Вместо Docker для Kubernetes 1.27+ используется только containerd. Все задачи по установке и настройке контейнерного рантайма выполняет роль `containerd`.
```

## Требования

- Ansible 2.9+
- SSH доступ к серверам с sudo правами
- Raspberry Pi OS на всех узлах

# Kubernetes Cluster Management with Ansible

Ansible репозиторий для управления локальным Kubernetes кластером на Raspberry Pi.

## Архитектура кластера

- **Master Node**: Raspberry Pi 4 (также выполняет роль worker)
- **Worker Nodes**: 2x Raspberry Pi 3

## Разработка в DevContainer

Для удобной разработки и тестирования используйте DevContainer:

1. Откройте проект в VS Code
2. Установите расширение "Dev Containers"
3. Нажмите `Ctrl+Shift+P` → "Dev Containers: Reopen in Container"
4. Контейнер автоматически настроится со всеми зависимостями

### Доступные команды в DevContainer:

```bash
# Справка по командам
make help

# Проверка синтаксиса
make syntax-check

# Проверка линтерами
make lint

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
├── credentials.json            # Креды (игнорируется Git)
├── credentials.json.example    # Пример файла с кредами
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
    ├── update-cluster.yml
    ├── maintenance.yml
    └── test-cluster.yml
```

## Быстрый старт

1. Установите требования Ansible:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

2. Создайте файл с кредами:

   ```bash
   cp credentials.json.example credentials.json
   ```

3. Отредактируйте `credentials.json` и `inventory.yml` с адресами ваших серверов

4. Проверьте доступность хостов:

   ```bash
   ansible all -m ping
   ```

5. Запустите установку кластера:

   ```bash
   ansible-playbook site.yml --tags install
   ```

## Поштучное обновление нод

### Обновление одной ноды

```bash
# Обновить конкретную ноду до указанной версии
ansible-playbook playbooks/update-single-node.yml \
  -e target_host=pi4-master \
  -e kubernetes_target_version=1.29

# С дополнительными опциями
ansible-playbook playbooks/update-single-node.yml \
  -e target_host=pi3-worker1 \
  -e kubernetes_target_version=1.29 \
  -e drain_node=true \
  -e update_system_packages=true
```

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
  -e target_host=pi3-worker1 \
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
ansible-playbook playbooks/update-single-node.yml -e target_host=pi4-master -e kubernetes_target_version=1.29

# Обновление нескольких нод с разными версиями
ansible-playbook playbooks/update-multiple-nodes.yml -e @node-updates.yml

# Откат ноды к предыдущей версии
ansible-playbook playbooks/rollback-node.yml -e target_host=pi3-worker1 -e rollback_version=1.28

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

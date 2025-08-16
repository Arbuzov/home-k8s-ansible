# 🚀 Kubernetes на Raspberry Pi - Быстрый старт

## 📋 Что нужно
- Raspberry Pi 4 (4GB+) для мастера
- Raspberry Pi 3/4 для worker нод
- Raspberry Pi OS 64-bit (Bookworm) на всех нодах
- SSH доступ ко всем нодам

## ⚡ Быстрая установка

### 1. Подготовка
```bash
# Клонируйте репозиторий
git clone <repo-url>
cd kubernetes-ansible

# Создайте файл с кредами
cp credentials.json.example credentials.json
```

### 2. Настройка кредов
Отредактируйте `credentials.json`:
```json
{
  "users": {
    "admin_user": "pi",
    "admin_password": "ваш-пароль"
  }
}
```

### 3. Настройка инвентаря
Отредактируйте `inventory.yml` с IP адресами ваших Pi:
```yaml
all:
  children:
    masters:
      hosts:
        pi4-master:
          ansible_host: 192.168.1.100
    workers:
      hosts:
        pi3-worker1:
          ansible_host: 192.168.1.101
        pi3-worker2:
          ansible_host: 192.168.1.102
```

### 4. Установка кластера
```bash
# Проверьте доступность
ansible all -m ping -e @credentials.json

# Установите кластер
ansible-playbook site.yml -e @credentials.json
```

### 5. Проверка
```bash
# Подключитесь к мастеру и проверьте кластер
ssh pi@192.168.1.100
kubectl get nodes

# Ожидаемый результат:
# NAME            STATUS   ROLES                  AGE   VERSION
# pi4-master      Ready    control-plane,worker   5m    v1.33.1
# pi3-worker1     Ready    worker                 3m    v1.33.1
# pi3-worker2     Ready    worker                 2m    v1.33.1
```

## 🔧 Добавление новой worker ноды

```bash
# Установите и присоедините новую ноду
ansible-playbook playbooks/install-worker-with-join.yml \
  --limit новая-нода,pi4-master \
  -e @credentials.json
```

## 🆘 Решение проблем

### Nода в статусе NotReady
```bash
# Проверьте kubelet
ssh pi@проблемная-нода
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
```

### Проблемы с cgroups
Проверьте параметры ядра:
```bash
cat /proc/cmdline
# Должно содержать: cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

### Swap не отключен
```bash
# Проверьте swap
free -h
# Swap должен быть 0B

# Если swap включен
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### containerd проблемы
```bash
# Проверьте containerd
sudo systemctl status containerd

# Проверьте CRI API
sudo crictl version
```

## 📚 Дополнительно
- [Полная документация](README.md)
- [Документация ролей](roles/)
- [Плейбуки](playbooks/)

## ⚠️ Важно
- **Используйте только Raspberry Pi OS 64-bit (Bookworm)**
- **Не игнорируйте preflight ошибки** - исправляйте их
- **Swap должен быть полностью отключен**
- **cgroups параметры критически важны для RPi**

5. **Разверните кластер**:

   ```bash
   ansible-playbook site.yml --tags install
   ```

6. **Протестируйте кластер**:

   ```bash
   ansible-playbook playbooks/test-cluster.yml
   ```

## После установки

- Скопируйте kubeconfig с master ноды для управления кластером
- Установите дополнительные компоненты (ingress, storage, мониторинг)
- Настройте backup и мониторинг

## Обслуживание

```bash
# Обновление кластера
ansible-playbook playbooks/update-cluster.yml

# Проверка состояния
ansible-playbook playbooks/maintenance.yml --tags check

# Очистка системы
ansible-playbook playbooks/maintenance.yml --tags cleanup
```

Готово! Ваш Kubernetes кластер на Raspberry Pi готов к использованию.

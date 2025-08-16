# 🚀 Kubernetes на Raspberry Pi - Быстрый старт

## 📋 Что нужно
- Raspberry Pi 4 (4GB+) для мастера
- Raspberry Pi 3/4 для worker нод
- Raspberry Pi OS 64-bit (Bookworm) на всех нодах
- SSH доступ ко всем нодам

⚠️ **Важно**: Имена хостов будут автоматически изменены на `kube-master`, `kube-worker-1`, `kube-worker-2`

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
        kube-master:
          ansible_host: 192.168.1.100
          kubernetes_version: "1.33.1"     # Версия K8s для мастера
          kubernetes_major_minor: "1.33"
    workers:
      hosts:
        kube-worker-1:
          ansible_host: 192.168.1.101
          kubernetes_version: "1.33.1"     # Версия K8s для worker
          kubernetes_major_minor: "1.33"
        kube-worker-2:
          ansible_host: 192.168.1.102
          kubernetes_version: "1.33.1"     # Можно указать разные версии
          kubernetes_major_minor: "1.33"
```

⚠️ **Важно**: Версии Kubernetes теперь задаются для каждой ноды индивидуально!

Пример смешанного кластера с разными версиями:
```yaml
    masters:
      hosts:
        kube-master:
          ansible_host: 192.168.1.100
          kubernetes_version: "1.33.1"     # Мастер на последней версии
          kubernetes_major_minor: "1.33"
    workers:
      hosts:
        kube-worker-1:
          ansible_host: 192.168.1.101
          kubernetes_version: "1.32.1"     # Worker на предыдущей версии
          kubernetes_major_minor: "1.32"   # Поддерживается skew policy
        kube-worker-2:
          ansible_host: 192.168.1.102
          kubernetes_version: "1.33.1"     # Worker на той же версии что мастер
          kubernetes_major_minor: "1.33"
```

📋 **Политика версий**: Воркеры могут быть на 1 минорную версию ниже мастера (K8s version skew policy).

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
# kube-master      Ready    control-plane,worker   5m    v1.33.1
# kube-worker-1     Ready    worker                 3m    v1.33.1
# kube-worker-2     Ready    worker                 2m    v1.33.1
```

## 🔧 Добавление новой worker ноды

```bash
# Установите и присоедините новую ноду
ansible-playbook playbooks/install-worker-with-join.yml \
  --limit новая-нода,kube-master \
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

## Управление версиями

### Обновление отдельных нод
Для обновления конкретной ноды измените версию в inventory и запустите:
```bash
# Обновить только одну ноду
ansible-playbook playbooks/update-single-node.yml -l kube-worker-1

# Обновить несколько нод
ansible-playbook playbooks/update-multiple-nodes.yml -l workers
```

### Миграция на новую версию
1. Измените `kubernetes_version` и `kubernetes_major_minor` в inventory.yml
2. Сначала обновите worker ноды:
   ```bash
   ansible-playbook playbooks/update-cluster.yml --limit workers
   ```
3. Затем обновите master:
   ```bash
   ansible-playbook playbooks/update-cluster.yml --limit masters
   ```

### Проверка совместимости версий
```bash
# Проверка текущих версий всех нод
kubectl get nodes -o wide

# Проверка компонентов кластера
kubectl version
```

💡 **Совет**: Всегда тестируйте обновления сначала на одном worker узле!

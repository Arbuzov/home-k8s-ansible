# Быстрая настройка кластера Kubernetes на Raspberry Pi

## Подготовка

1. **Подготовьте Raspberry Pi**:
   - Raspberry Pi 4 (master + worker)
   - 2x Raspberry Pi 3 (workers)
   - Установите Raspberry Pi OS на все устройства
   - Включите SSH на всех устройствах

2. **Настройте сеть**:
   - Назначьте статические IP адреса
   - Убедитесь, что все устройства видят друг друга

3. **Настройте SSH ключи**:
   ```bash
   ssh-keygen -t rsa -b 4096
   ssh-copy-id pi@192.168.1.100  # master
   ssh-copy-id pi@192.168.1.101  # worker1
   ssh-copy-id pi@192.168.1.102  # worker2
   ```

## Развертывание

1. **Клонируйте репозиторий**:

   ```bash
   git clone <this-repo>
   cd local-cluster-ansible
   ```

2. **Настройте переменные**:

   ```bash
   cp credentials.json.example credentials.json
   # Отредактируйте credentials.json и inventory.yml
   ```

3. **Установите зависимости**:

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```

4. **Проверьте подключение**:

   ```bash
   ansible all -m ping
   ```

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

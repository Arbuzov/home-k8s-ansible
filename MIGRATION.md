# Миграция переменных в credentials.json

## Обновление с версии 2024 на 2025

### Старый формат (вложенная структура):
```json
{
  "ssh": {
    "private_key_path": "~/.ssh/id_rsa",
    "public_key_path": "~/.ssh/id_rsa.pub"
  },
  "kubernetes": {
    "cluster_name": "raspberry-k8s",
    "pod_subnet": "10.244.0.0/16",
    "service_subnet": "10.96.0.0/12",
    "api_server_advertise_address": "192.168.1.100"
  },
  "users": {
    "admin_user": "pi",
    "admin_password": "YOUR_PASSWORD_HERE"
  }
}
```

### Новый формат (плоская структура):
```json
{
  "ansible_user": "pi",
  "ansible_password": "YOUR_PASSWORD_HERE",
  "ansible_ssh_private_key_file": "~/.ssh/id_rsa",
  "ansible_ssh_public_key_file": "~/.ssh/id_rsa.pub",
  "kubernetes_cluster_name": "raspberry-k8s",
  "kubernetes_pod_subnet": "10.244.0.0/16",
  "kubernetes_service_subnet": "10.96.0.0/12",
  "kubernetes_api_server_advertise_address": "192.168.1.100",
  "containerd_data_root": "/var/lib/containerd",
  "registry_mirror": "https://registry-1.docker.io"
}
```

## Преимущества нового формата

1. **Прямая совместимость с Ansible** - переменные напрямую становятся ansible_user, ansible_password
2. **Упрощенное подключение** - `-e @credentials.json` автоматически подключает все переменные
3. **Меньше ошибок** - нет необходимости обращаться к users.admin_user, kubernetes.pod_subnet
4. **Лучшая читаемость** - все переменные на одном уровне

## Обновленные роли

- `roles/master/tasks/main.yml` - обновлены ссылки на kubernetes_* переменные
- `group_vars/workers.yml` - обновлена ссылка на kubernetes_pod_subnet
- `site.yml` - обновлена ссылка на ansible_password

## Команды для использования

```bash
# Основная установка
ansible-playbook site.yml -e @credentials.json

# Обновление конкретной ноды
ansible-playbook playbooks/update-worker-version.yml -l kube-worker-2 -e @credentials.json

# Проверка доступности
ansible all -m ping -e @credentials.json
```

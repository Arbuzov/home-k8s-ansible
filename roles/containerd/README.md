# Containerd Role

## Описание
Роль `containerd` устанавливает и настраивает containerd как container runtime для Kubernetes.

## Задачи
- Установка необходимых пакетов для работы с containerd
- Добавление Docker GPG ключа и репозитория
- Установка последней версии containerd.io
- Создание и настройка конфигурационного файла containerd
- **Включение SystemdCgroup** для совместимости с Kubernetes
- **Исправление CRI плагина** (удаление из disabled_plugins)
- Перезапуск и включение службы containerd

## Переменные
Роль не требует дополнительных переменных - использует значения по умолчанию.

## Критически важные настройки

### SystemdCgroup
```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
Необходимо для правильной работы с cgroup v2 в Kubernetes.

### CRI Plugin
```toml
disabled_plugins = []  # вместо ["cri"]
```
CRI плагин должен быть включен для взаимодействия с kubelet.

## Зависимости
- Роль должна выполняться после роли `common`
- Требует настроенные cgroups в системе

## Совместимость
- Kubernetes 1.27+
- cgroup v2
- ARM64 архитектура (Raspberry Pi)

## Проверка работы
После выполнения роли можно проверить:
```bash
# Статус службы
systemctl status containerd

# Версия containerd
containerd --version

# Проверка CRI API
crictl version
```

## Примечания
- Роль устанавливает containerd из Docker репозитория
- Конфигурация генерируется автоматически с помощью `containerd config default`
- Изменения применяются немедленно после перезапуска службы

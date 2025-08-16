# Kubernetes Role

## Описание
Роль `kubernetes` устанавливает компоненты Kubernetes (kubelet, kubeadm, kubectl) и настраивает их для работы.

## Задачи
- Очистка старых ключей и репозиториев Kubernetes
- Установка GPG ключа из официального репозитория Kubernetes
- Добавление официального репозитория Kubernetes
- Установка пакетов Kubernetes с закреплением версии
- Настройка kubelet и его systemd службы
- Проверка установки

## Переменные
- `kubernetes_major_minor`: мажорная.минорная версия K8s (например: "1.33")
- `kubernetes_version`: полная версия K8s (например: "1.33.1")

## Критически важные особенности

### Официальный репозиторий
Использует новый официальный репозиторий:
```
https://pkgs.k8s.io/core:/stable:/v{{ kubernetes_major_minor }}/deb/
```

### GPG ключ
Ключ скачивается и сохраняется в dearmor формате для apt >= 2.4:
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key |
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### Конфигурация kubelet
Создаёт systemd override с правильными путями:
- Конфигурация: `/var/lib/kubelet/config.yaml` (совместимо с kubeadm)
- Bootstrap kubeconfig: `/etc/kubernetes/bootstrap-kubelet.conf`
- Runtime kubeconfig: `/etc/kubernetes/kubelet.conf`

### Закрепление версий
Все пакеты закрепляются на определённой версии:
```bash
apt-mark hold kubelet kubeadm kubectl
```

## Установленные пакеты
- `kubelet={{ kubernetes_version }}-1.1`
- `kubeadm={{ kubernetes_version }}-1.1`
- `kubectl={{ kubernetes_version }}-1.1`

## Зависимости
- Роль `common` (для cgroups и системных настроек)
- Роль `containerd` (для container runtime)

## Проверка установки
```bash
# Версия kubeadm
kubeadm version

# Статус kubelet
systemctl status kubelet

# Проверка подключения к containerd
kubeadm config images list
```

## Совместимость
- Kubernetes 1.27+
- containerd как container runtime
- systemd как cgroup driver
- ARM64 архитектура

## Примечания
- Роль создаёт базовую конфигурацию kubelet
- kubelet будет в состоянии ошибки до присоединения к кластеру (это нормально)
- После установки требуется выполнить `kubeadm init` или `kubeadm join`

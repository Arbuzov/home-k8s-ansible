# Common Role

## Описание
Роль `common` выполняет базовую настройку системы для работы Kubernetes на Raspberry Pi.

## Задачи
- Обновление пакетного кэша и установка системных пакетов
- **Установка hostname** из inventory_hostname
- **Обновление /etc/hosts** с новым hostname
- Настройка часового пояса
- Настройка GPU memory split для Raspberry Pi
- **Включение cgroup memory и cpuset** для Kubernetes
- **Отключение swap** (требование Kubernetes)
- Загрузка необходимых модулей ядра (br_netfilter, overlay)
- Настройка sysctl параметров для Kubernetes
- Настройка DNS

## Переменные
- `system_packages`: список системных пакетов для установки
- `timezone`: часовой пояс (по умолчанию: "Europe/Moscow")
- `arm_memory_split`: память для GPU на ARM (по умолчанию: 16)
- `cgroup_enabled`: включить cgroup настройки (по умолчанию: true)
- `swap_disabled`: отключить swap (по умолчанию: true)
- `inventory_hostname`: используется для установки hostname системы

## Критически важные изменения
### cgroup настройки
Роль добавляет в `/boot/firmware/cmdline.txt` критически важные параметры:
```
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```
**Порядок параметров важен!** После изменения требуется перезагрузка.

### Отключение swap
Swap отключается полностью:
- Немедленное отключение: `swapoff -a`
- Постоянное отключение: комментирование в `/etc/fstab`

## Handlers
- `reboot system`: перезагрузка системы при изменении критических параметров

## Совместимость
- Raspberry Pi OS (64-bit)
- Kubernetes 1.27+
- containerd как container runtime

## Примечания
- Роль должна выполняться с правами root (become: true)
- После выполнения роли может потребоваться перезагрузка
- Изменения в cmdline.txt применяются только после перезагрузки

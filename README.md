# Ansible Role: Host Optimization

Роль для оптимизации ОС и хоста под виртуализацию (KVM/libvirt).

## Описание

Эта роль оптимизирует систему для работы в качестве хоста виртуализации:
- Оптимизация параметров памяти (swappiness, dirty pages, overcommit)
- Настройка IP forwarding для виртуализации
- Настройка параметров ядра (IOMMU, hugepages)
- Оптимизация CPU (governor, frequency scaling)
- Оптимизация файловых систем (noatime)
- Настройка лимитов системы для libvirt
- Оптимизация сети для виртуализации

## Требования

- Debian 11+ или Ubuntu 20.04+
- Процессор с поддержкой виртуализации (Intel VT-x или AMD-V)
- Доступ root/sudo
- Ansible >= 2.9

## Переменные роли

См. `defaults/main.yml` для всех доступных переменных.

### Основные переменные

```yaml
# Включить оптимизацию
host_optimization_enabled: true
host_optimization_for_virtualization: true

# Настройки памяти
host_swappiness: 10
host_dirty_ratio: 10
host_overcommit_memory: 1

# IP forwarding
host_ip_forward_enabled: true

# CPU governor
host_cpu_governor: "performance"

# Hugepages
host_hugepages_enabled: false
host_hugepages_count: 1024

# Параметры ядра
host_kernel_params:
  - name: "amd_iommu"
    value: "on"
    enabled: true
```

## Использование

### Базовая оптимизация

```yaml
- hosts: virtualization_hosts
  become: true
  roles:
    - host-optimization
```

### С hugepages

```yaml
- hosts: virtualization_hosts
  become: true
  vars:
    host_hugepages_enabled: true
    host_hugepages_count: 2048
  roles:
    - host-optimization
```

### Полная оптимизация для виртуализации

```yaml
- hosts: virtualization_hosts
  become: true
  vars:
    host_optimization_for_virtualization: true
    host_ip_forward_enabled: true
    host_swappiness: 1
    host_cpu_governor: "performance"
    host_hugepages_enabled: true
    host_hugepages_count: 4096
    host_fstab_noatime: true
  roles:
    - host-optimization
```

## Оптимизации

### 1. Память
- **Swappiness:** Уменьшено до 10 (или 1) для серверов с большим RAM
- **Dirty pages:** Оптимизировано соотношение dirty_ratio и dirty_background_ratio
- **Overcommit:** Включен для виртуализации (overcommit_memory = 1)
- **Hugepages:** Опционально для высокой производительности VM

### 2. Сеть
- **IP forwarding:** Включен для маршрутизации пакетов VM
- **TCP оптимизация:** Увеличены буферы для лучшей производительности сети

### 3. CPU
- **Governor:** Установлен в "performance" для максимальной производительности
- **Frequency scaling:** Настроено для постоянной высокой частоты

### 4. Файловые системы
- **noatime:** Отключено обновление времени доступа для снижения нагрузки на диск

### 5. Ядро
- **IOMMU:** Включен для PCI passthrough (AMD: amd_iommu=on, Intel: intel_iommu=on)
- **Transparent hugepages:** Включен для лучшей производительности памяти

### 6. Лимиты системы
- **Файловые дескрипторы:** Увеличены до 65536
- **Процессы:** Увеличены до 65536
- **libvirt:** Настроены лимиты для libvirt процессов

## Теги

- `optimization` - все задачи оптимизации
- `packages` - установка пакетов
- `memory` - оптимизация памяти
- `network` - оптимизация сети
- `cpu` - оптимизация CPU
- `kernel` - настройка параметров ядра
- `filesystem` - оптимизация файловых систем
- `limits` - настройка лимитов
- `libvirt` - настройки для libvirt
- `check` - проверка настроек

## Примеры использования тегов

```bash
# Только оптимизация памяти
ansible-playbook playbook.yml --tags memory

# Только настройка ядра (требует перезагрузки)
ansible-playbook playbook.yml --tags kernel

# Без изменений ядра (без перезагрузки)
ansible-playbook playbook.yml --skip-tags kernel
```

## Проверка оптимизаций

```bash
# Проверка swappiness
sysctl vm.swappiness

# Проверка IP forwarding
sysctl net.ipv4.ip_forward

# Проверка CPU governor
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Проверка hugepages
cat /proc/meminfo | grep HugePages

# Проверка параметров ядра
cat /proc/cmdline

# Проверка лимитов
ulimit -a
```

## Важные замечания

### ⚠️ Перезагрузка требуется

Некоторые оптимизации требуют перезагрузки системы:
- Изменения параметров ядра (IOMMU, hugepages)
- Изменения в GRUB
- Изменения noatime в fstab

### Рекомендации

1. **Swappiness:** Для серверов с 64GB+ RAM используйте значение 1-10
2. **Hugepages:** Включайте только если планируете запускать много VM
3. **CPU governor:** Performance режим увеличивает энергопотребление
4. **IOMMU:** Включайте только если планируете использовать PCI passthrough

## Лицензия

ISC License

---

**Последнее обновление:** январь 2026

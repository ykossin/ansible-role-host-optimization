# Ansible Role: Host Optimization

**Роль для оптимизации ОС и хоста под виртуализацию (KVM/libvirt)**

---

## Содержание

1. [Описание роли](#описание-роли)
2. [Критические предупреждения](#критические-предупреждения)
3. [Структура роли и задачи](#структура-роли-и-задачи)
4. [Детальное описание задач](#детальное-описание-задач)
5. [Переменные роли](#переменные-роли)
6. [Использование](#использование)
7. [Риски и безопасность](#риски-и-безопасность)
8. [Проверка и мониторинг](#проверка-и-мониторинг)

---

## Описание роли

Роль `host-optimization` оптимизирует операционную систему Linux для работы в качестве хоста виртуализации. Она настраивает параметры ядра, памяти, сети, CPU и файловых систем для максимальной производительности виртуальных машин.

### Что делает роль:

- Оптимизирует параметры памяти (swappiness, dirty pages, overcommit, hugepages)
- Настраивает IP forwarding для виртуализации
- Настраивает параметры ядра (IOMMU, transparent hugepages)
- Оптимизирует CPU (governor, frequency scaling)
- Оптимизирует файловые системы (noatime, I/O scheduler)
- Настраивает системные лимиты для libvirt
- Настраивает tuned профили
- Выполняет валидацию и проверки безопасности
- Создает резервные копии конфигураций

---

## Критические предупреждения

ВАЖНО: Эта роль вносит критические изменения в систему!

**Роль изменяет:**
- Параметры ядра (GRUB) - требует перезагрузки
- Файл `/etc/fstab` - может привести к проблемам загрузки
- Отключение swap - может привести к нехватке памяти
- Системные лимиты и параметры ядра

**Перед применением ОБЯЗАТЕЛЬНО:**
1. Используйте `--check` для проверки изменений
2. Убедитесь, что у вас есть доступ к консоли сервера
3. Создайте полную резервную копию системы
4. Протестируйте на тестовом сервере

**Защитные механизмы:**
- Автоматическое резервное копирование конфигураций
- Проверки безопасности перед критическими изменениями
- Требование подтверждения для опасных операций
- Полная поддержка check mode (dry-run)

---

## Структура роли и задачи

Роль разделена на модульные файлы задач для лучшей организации и поддержки:

```
tasks/
├── main.yml           # Главный файл - импортирует все модули
├── validation.yml     # Валидация переменных
├── check.yml          # Проверка системы
├── safety-checks.yml  # Проверки безопасности
├── backup.yml         # Резервное копирование
├── packages.yml       # Установка пакетов
├── memory.yml         # Оптимизация памяти
├── network.yml        # Оптимизация сети
├── cpu.yml            # Оптимизация CPU
├── kernel.yml         # Настройка параметров ядра
├── filesystem.yml     # Оптимизация файловых систем
├── limits.yml         # Настройка системных лимитов
├── tuned.yml          # Настройка tuned
└── verify.yml         # Проверка применённых настроек
```

---

## Детальное описание задач

### 1. `validation.yml` - Валидация переменных

**Что делает:**
Проверяет все переменные роли на корректность перед применением изменений.

**Проверяемые параметры:**
- `host_swappiness` - должно быть в диапазоне 0-100
- `host_dirty_ratio` - должно быть в диапазоне 0-100
- `host_dirty_background_ratio` - должно быть в диапазоне 0-100
- `host_overcommit_memory` - должно быть 0, 1 или 2
- `host_cpu_governor` - должно быть одно из допустимых значений
- `host_io_scheduler` - должно быть одно из допустимых значений
- `host_ulimit_nofile` - должно быть больше 0
- `host_ulimit_nproc` - должно быть больше 0
- `host_hugepages_count` - должно быть больше или равно 0

**Риски:**
- Некорректные значения могут привести к нестабильности системы
- Некорректные параметры памяти могут вызвать проблемы с производительностью

**Когда выполняется:**
- Всегда первой, если `host_optimization_enabled: true`

**Теги:** `optimization`, `validation`

---

### 2. `check.yml` - Проверка системы

**Что делает:**
Собирает информацию о системе перед применением оптимизаций.

**Проверяет:**
- Архитектуру процессора (`uname -m`)
- Тип процессора (Intel/AMD) через проверку флагов виртуализации
- Поддержку виртуализации (vmx для Intel, svm для AMD)
- Информацию о дисках и их I/O scheduler

**Риски:**
- Безопасная операция - только чтение информации
- Не изменяет систему

**Когда выполняется:**
- После валидации, перед применением изменений

**Теги:** `optimization`, `check`

---

### 3. `safety-checks.yml` - Проверки безопасности

**Что делает:**
Выполняет критические проверки безопасности перед опасными операциями.

#### 3.1. Проверка перед отключением swap

**Проверяет:**
- Доступную память (минимум 16GB рекомендуется)
- Использование swap (не более 10% рекомендуется)
- Требует подтверждения через `host_optimization_confirm_dangerous: true`

**Риски:**
- **КРИТИЧЕСКИЙ:** Отключение swap при недостатке памяти может привести к:
  - Нехватке памяти при пиковых нагрузках
  - Завершению процессов OOM killer
  - Нестабильности системы
  - Потере данных

**Когда выполняется:**
- Если `host_swap_disabled: true`
- Перед отключением swap

**Теги:** `optimization`, `safety`, `swap`

#### 3.2. Проверка перед изменением GRUB

**Проверяет:**
- Текущие параметры ядра
- Предупреждает о необходимости перезагрузки

**Риски:**
- **КРИТИЧЕСКИЙ:** Изменение параметров ядра может привести к:
  - Невозможности загрузки системы
  - Проблемам с драйверами
  - Потере доступа к системе

**Когда выполняется:**
- Если `host_optimization_for_virtualization: true`
- Перед изменением GRUB

**Теги:** `optimization`, `safety`, `kernel`

#### 3.3. Проверка перед изменением fstab

**Проверяет:**
- Текущее состояние noatime в fstab
- Предупреждает о критичности изменений

**Риски:**
- **КРИТИЧЕСКИЙ:** Изменение fstab может привести к:
  - Невозможности загрузки системы
  - Проблемам с монтированием файловых систем
  - Потере доступа к системе

**Когда выполняется:**
- Если `host_fstab_noatime: true`
- Перед изменением fstab

**Теги:** `optimization`, `safety`, `filesystem`

#### 3.4. Валидация параметров памяти

**Проверяет:**
- Диапазоны значений параметров памяти
- Логическую корректность (dirty_background_ratio < dirty_ratio)

**Риски:**
- Некорректные параметры могут привести к:
  - Нестабильности системы
  - Проблемам с производительностью
  - OOM killer завершает процессы

**Когда выполняется:**
- Всегда, если `host_optimization_enabled: true`

**Теги:** `optimization`, `safety`, `memory`

---

### 4. `backup.yml` - Резервное копирование

**Что делает:**
Создает резервные копии критических конфигурационных файлов перед их изменением.

**Бэкапируемые файлы:**
- `/etc/default/grub` - перед изменением параметров ядра
- `/etc/fstab` - перед добавлением noatime
- `/etc/security/limits.conf` - перед изменением лимитов

**Расположение бэкапов:**
```
/root/ansible-backups/host-optimization/
├── grub-<timestamp>.bak
├── fstab-<timestamp>.bak
└── limits-<timestamp>.bak
```

**Риски:**
- Безопасная операция - только создание копий
- Не изменяет оригинальные файлы

**Когда выполняется:**
- Перед изменением критических файлов
- Всегда, если включена оптимизация

**Теги:** `optimization`, `backup`

**Восстановление из бэкапа:**
```bash
# Восстановление GRUB
cp /root/ansible-backups/host-optimization/grub-<timestamp>.bak /etc/default/grub
update-grub

# Восстановление fstab
cp /root/ansible-backups/host-optimization/fstab-<timestamp>.bak /etc/fstab
mount -a
```

---

### 5. `packages.yml` - Установка пакетов

**Что делает:**
Устанавливает необходимые пакеты для оптимизации.

**Устанавливаемые пакеты:**
- `cpufrequtils` - утилиты для управления частотой CPU
- `tuned` - система настройки производительности
- `sysfsutils` - утилиты для работы с sysfs

**Риски:**
- Установка пакетов требует обновления кэша apt
- Может занять время в зависимости от скорости сети
- Относительно безопасная операция

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_optimization_for_virtualization: true`

**Теги:** `optimization`, `packages`

---

### 6. `memory.yml` - Оптимизация памяти

**Что делает:**
Настраивает параметры управления памятью системы.

#### 6.1. Настройка swappiness

**Что делает:**
Устанавливает значение `vm.swappiness` - определяет, насколько агрессивно система будет использовать swap.

**Параметр:** `host_swappiness` (по умолчанию: 10)

**Значения:**
- `0` - система будет избегать использования swap
- `10` - рекомендуемое значение для серверов с большим RAM
- `60` - значение по умолчанию в большинстве дистрибутивов
- `100` - система будет активно использовать swap

**Риски:**
- Слишком низкое значение (0-1) может привести к:
  - Нехватке памяти при пиковых нагрузках
  - Завершению процессов OOM killer
- Слишком высокое значение может привести к:
  - Медленной работе из-за частого использования swap
  - Износу SSD дисков

**Когда выполняется:**
- Всегда, если `host_optimization_enabled: true`

**Теги:** `optimization`, `memory`

#### 6.2. Настройка dirty pages

**Что делает:**
Настраивает параметры записи "грязных" страниц памяти на диск.

**Параметры:**
- `vm.dirty_ratio` - процент памяти, при котором процессы начинают синхронную запись
- `vm.dirty_background_ratio` - процент памяти, при котором фоновые процессы начинают асинхронную запись

**Риски:**
- Слишком высокие значения могут привести к:
  - Потере данных при сбое питания
  - Долгой задержке при записи на диск
- Слишком низкие значения могут привести к:
  - Снижению производительности из-за частых записей
  - Износу SSD дисков

**Когда выполняется:**
- Всегда, если `host_optimization_enabled: true`

**Теги:** `optimization`, `memory`

#### 6.3. Настройка overcommit memory

**Что делает:**
Настраивает политику overcommit памяти - позволяет ли система выделять больше памяти, чем физически доступно.

**Параметр:** `host_overcommit_memory` (по умолчанию: 1)

**Значения:**
- `0` - эвристический алгоритм (по умолчанию)
- `1` - всегда разрешать overcommit (рекомендуется для виртуализации)
- `2` - не разрешать overcommit

**Риски:**
- Значение `1` может привести к:
  - Нехватке памяти при реальном использовании
  - Завершению процессов OOM killer
- Значение `2` может привести к:
  - Невозможности запуска виртуальных машин
  - Отказу в выделении памяти

**Когда выполняется:**
- Всегда, если `host_optimization_enabled: true`

**Теги:** `optimization`, `memory`

#### 6.4. Настройка hugepages

**Что делает:**
Настраивает количество hugepages - больших страниц памяти (обычно 2MB) для улучшения производительности.

**Параметры:**
- `host_hugepages_enabled` - включить/выключить hugepages
- `host_hugepages_count` - количество hugepages (по умолчанию: 1024)

**Риски:**
- Слишком большое количество может привести к:
  - Нехватке обычной памяти
  - Проблемам с запуском приложений
- Hugepages резервируют память, она недоступна для обычных процессов

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_hugepages_enabled: true`

**Теги:** `optimization`, `memory`, `hugepages`

#### 6.5. Отключение swap

**Что делает:**
Отключает swap и комментирует записи в fstab.

**Параметр:** `host_swap_disabled` (по умолчанию: false)

**Риски:**
- **КРИТИЧЕСКИЙ:** Отключение swap может привести к:
  - Нехватке памяти при пиковых нагрузках
  - Завершению процессов OOM killer
  - Нестабильности системы
  - Потере данных

**Требования:**
- Требует `host_optimization_confirm_dangerous: true`
- Проверка доступной памяти (минимум 16GB)
- Проверка использования swap (не более 10%)

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_swap_disabled: true`
- Если `host_optimization_confirm_dangerous: true`

**Теги:** `optimization`, `memory`, `swap`

---

### 7. `network.yml` - Оптимизация сети

**Что делает:**
Настраивает сетевые параметры для виртуализации и производительности.

#### 7.1. IP Forwarding

**Что делает:**
Включает IP forwarding для маршрутизации пакетов между виртуальными машинами.

**Параметры:**
- `host_ip_forward_enabled` - включить/выключить IP forwarding
- `host_ip_forward_ipv4` - включить для IPv4
- `host_ip_forward_ipv6` - включить для IPv6

**Риски:**
- Включение IP forwarding может:
  - Изменить поведение файрвола
  - Позволить маршрутизацию пакетов
  - Требовать дополнительной настройки безопасности

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_ip_forward_enabled: true`

**Теги:** `optimization`, `network`

#### 7.2. Оптимизация TCP

**Что делает:**
Настраивает TCP буферы и параметры для лучшей производительности сети.

**Параметры:**
- `net.core.rmem_max` - максимальный размер буфера приема
- `net.core.wmem_max` - максимальный размер буфера передачи
- `net.ipv4.tcp_rmem` - размеры буферов приема TCP
- `net.ipv4.tcp_wmem` - размеры буферов передачи TCP
- `net.core.netdev_max_backlog` - максимальная очередь сетевых устройств
- `net.ipv4.tcp_fin_timeout` - таймаут закрытия TCP соединений
- `net.ipv4.tcp_keepalive_time` - интервал keepalive для TCP

**Риски:**
- Слишком большие буферы могут привести к:
  - Увеличению использования памяти
  - Проблемам с производительностью при нехватке памяти
- Слишком маленькие буферы могут привести к:
  - Снижению пропускной способности сети
  - Потере пакетов

**Когда выполняется:**
- Всегда, если `host_optimization_enabled: true`

**Теги:** `optimization`, `network`

#### 7.3. Параметры безопасности сети

**Что делает:**
Настраивает параметры безопасности сети.

**Параметры:**
- `net.ipv4.conf.all.send_redirects: 0` - отключить отправку редиректов
- `net.ipv4.conf.default.send_redirects: 0` - отключить отправку редиректов по умолчанию
- `net.ipv4.conf.all.accept_redirects: 0` - отключить прием редиректов
- `net.ipv4.conf.default.accept_redirects: 0` - отключить прием редиректов по умолчанию

**Риски:**
- Безопасная операция - улучшает безопасность
- Не влияет на функциональность

**Когда выполняется:**
- Всегда, если `host_optimization_enabled: true`

**Теги:** `optimization`, `network`, `security`

---

### 8. `cpu.yml` - Оптимизация CPU

**Что делает:**
Настраивает параметры CPU для максимальной производительности.

#### 8.1. CPU Governor

**Что делает:**
Устанавливает CPU governor - алгоритм управления частотой CPU.

**Параметр:** `host_cpu_governor` (по умолчанию: "performance")

**Доступные значения:**
- `performance` - максимальная производительность (постоянная высокая частота)
- `powersave` - экономия энергии (низкая частота)
- `ondemand` - по требованию (динамическое изменение частоты)
- `conservative` - консервативный (медленное изменение частоты)
- `schedutil` - планировщик ядра (современный алгоритм)

**Риски:**
- Режим `performance` может привести к:
  - Увеличению энергопотребления
  - Перегреву процессора
  - Сокращению срока службы оборудования
- Режим `powersave` может привести к:
  - Снижению производительности
  - Задержкам при высокой нагрузке

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_cpu_scaling_enabled: true`

**Теги:** `optimization`, `cpu`

#### 8.2. Настройка cpufrequtils

**Что делает:**
Настраивает CPU governor по умолчанию в `/etc/default/cpufrequtils`.

**Риски:**
- Безопасная операция
- Применяется при загрузке системы

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_cpu_scaling_enabled: true`

**Теги:** `optimization`, `cpu`

---

### 9. `kernel.yml` - Настройка параметров ядра

**Что делает:**
Настраивает параметры ядра Linux через GRUB.

#### 9.1. Автоматическое определение CPU

**Что делает:**
Автоматически определяет тип процессора (Intel/AMD) и настраивает соответствующий IOMMU.

**Риски:**
- Безопасная операция - только определение типа

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_optimization_for_virtualization: true`

**Теги:** `optimization`, `kernel`

#### 9.2. Настройка IOMMU

**Что делает:**
Добавляет параметры IOMMU в GRUB для поддержки PCI passthrough.

**Параметры:**
- `intel_iommu=on` - для процессоров Intel
- `amd_iommu=on` - для процессоров AMD
- `iommu=pt` - режим pass-through для лучшей производительности

**Риски:**
- **КРИТИЧЕСКИЙ:** Изменение параметров ядра может привести к:
  - Невозможности загрузки системы
  - Проблемам с драйверами
  - Потере доступа к системе
- Требует перезагрузки системы
- Неправильные параметры могут сделать систему неработоспособной

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_optimization_for_virtualization: true`
- Если параметры IOMMU включены в `host_kernel_params`

**Теги:** `optimization`, `kernel`

#### 9.3. Transparent Hugepages

**Что делает:**
Включает transparent hugepages для улучшения производительности памяти.

**Параметры:**
- `transparent_hugepage=always` - всегда использовать hugepages
- `transparent_hugepage_defrag=always` - всегда дефрагментировать

**Риски:**
- Требует перезагрузки системы
- Может влиять на производительность некоторых приложений

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_optimization_for_virtualization: true`
- Если параметры включены в `host_kernel_params`

**Теги:** `optimization`, `kernel`

---

### 10. `filesystem.yml` - Оптимизация файловых систем

**Что делает:**
Настраивает параметры файловых систем для лучшей производительности.

#### 10.1. Noatime

**Что делает:**
Добавляет параметр `noatime` в fstab для отключения обновления времени доступа к файлам.

**Параметр:** `host_fstab_noatime` (по умолчанию: true)

**Риски:**
- **КРИТИЧЕСКИЙ:** Изменение fstab может привести к:
  - Невозможности загрузки системы
  - Проблемам с монтированием файловых систем
  - Потере доступа к системе
- Требует перезагрузки для полного применения
- Некоторые приложения могут полагаться на время доступа

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_fstab_noatime: true`

**Теги:** `optimization`, `filesystem`

#### 10.2. I/O Scheduler

**Что делает:**
Настраивает I/O scheduler для всех дисков и сохраняет настройки в udev правилах.

**Параметр:** `host_io_scheduler` (по умолчанию: "none")

**Доступные значения:**
- `none` - для NVMe дисков (рекомендуется)
- `deadline` - для SSD дисков
- `noop` - для SSD дисков
- `cfq` - для HDD дисков
- `bfq` - для HDD дисков (лучшая производительность)
- `mq-deadline` - для современных дисков
- `kyber` - для современных дисков

**Риски:**
- Неправильный scheduler может привести к:
  - Снижению производительности дисков
  - Проблемам с производительностью I/O
- Изменение scheduler может повлиять на работу системы

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_io_scheduler` определен

**Теги:** `optimization`, `filesystem`, `io`

---

### 11. `limits.yml` - Настройка системных лимитов

**Что делает:**
Настраивает системные лимиты для процессов.

#### 11.1. Общие лимиты

**Что делает:**
Увеличивает лимиты файловых дескрипторов и процессов.

**Параметры:**
- `host_ulimit_nofile` - лимит файловых дескрипторов (по умолчанию: 65536)
- `host_ulimit_nproc` - лимит процессов (по умолчанию: 65536)

**Риски:**
- Слишком высокие лимиты могут привести к:
  - Увеличению использования памяти
  - Проблемам с производительностью
- Слишком низкие лимиты могут привести к:
  - Невозможности запуска приложений
  - Ошибкам "too many open files"

**Когда выполняется:**
- Всегда, если `host_optimization_enabled: true`

**Теги:** `optimization`, `limits`

#### 11.2. Лимиты для libvirt

**Что делает:**
Настраивает специальные лимиты для процессов libvirt.

**Параметры:**
- `host_libvirt_limits.soft_nofile` - мягкий лимит файловых дескрипторов
- `host_libvirt_limits.hard_nofile` - жесткий лимит файловых дескрипторов
- `host_libvirt_limits.soft_nproc` - мягкий лимит процессов
- `host_libvirt_limits.hard_nproc` - жесткий лимит процессов
- `host_libvirt_limits.soft_memlock` - мягкий лимит заблокированной памяти
- `host_libvirt_limits.hard_memlock` - жесткий лимит заблокированной памяти

**Риски:**
- Неправильные лимиты могут привести к:
  - Невозможности запуска виртуальных машин
  - Проблемам с производительностью libvirt

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_optimization_for_virtualization: true`

**Теги:** `optimization`, `limits`, `libvirt`

---

### 12. `tuned.yml` - Настройка tuned

**Что делает:**
Настраивает профиль tuned для оптимизации производительности.

**Параметры:**
- `host_tuned_enabled` - включить/выключить tuned (по умолчанию: true)
- `host_tuned_profile` - профиль tuned (по умолчанию: "virtual-host")

**Доступные профили:**
- `virtual-host` - для хостов виртуализации (рекомендуется)
- `virtual-guest` - для гостевых систем
- `throughput-performance` - для максимальной пропускной способности
- `latency-performance` - для минимальной задержки

**Риски:**
- Неправильный профиль может привести к:
  - Снижению производительности
  - Проблемам с энергопотреблением
- Tuned может изменять множество параметров системы

**Когда выполняется:**
- Если `host_optimization_enabled: true`
- Если `host_optimization_for_virtualization: true`
- Если `host_tuned_enabled: true`

**Теги:** `optimization`, `tuned`

---

### 13. `verify.yml` - Проверка применённых настроек

**Что делает:**
Проверяет, что все настройки были применены корректно.

**Проверяет:**
- Swappiness
- IP forwarding
- CPU governor
- Hugepages
- Параметры ядра

**Риски:**
- Безопасная операция - только чтение информации

**Когда выполняется:**
- После применения всех оптимизаций
- Всегда, если `host_optimization_enabled: true`

**Теги:** `optimization`, `verify`

---

## Переменные роли

### Основные переменные

```yaml
# Включить оптимизацию хоста
host_optimization_enabled: true

# Оптимизация для виртуализации
host_optimization_for_virtualization: true

# Подтверждение опасных операций (swap, fstab, grub)
# Установите в true только если вы понимаете риски!
host_optimization_confirm_dangerous: false
```

### Настройки памяти

```yaml
# Swappiness (0-100, рекомендуется 1-10 для серверов)
host_swappiness: 10

# Dirty ratio (0-100, процент памяти для синхронной записи)
host_dirty_ratio: 10

# Dirty background ratio (0-100, процент памяти для асинхронной записи)
host_dirty_background_ratio: 5

# Overcommit memory (0=эвристический, 1=всегда, 2=никогда)
host_overcommit_memory: 1

# Hugepages
host_hugepages_enabled: false
host_hugepages_count: 1024

# Отключение swap (требует подтверждения!)
host_swap_disabled: false
```

### Настройки сети

```yaml
# IP forwarding
host_ip_forward_enabled: true
host_ip_forward_ipv4: true
host_ip_forward_ipv6: false
```

### Настройки CPU

```yaml
# CPU governor (performance, powersave, ondemand, conservative, schedutil)
host_cpu_governor: "performance"

# CPU frequency scaling
host_cpu_scaling_enabled: true
```

### Настройки ядра

```yaml
# Параметры ядра для виртуализации
host_kernel_params:
  - name: "iommu"
    value: "pt"
    enabled: true
  - name: "transparent_hugepage"
    value: "always"
    enabled: true
```

### Настройки файловых систем

```yaml
# Noatime в fstab
host_fstab_noatime: true

# I/O scheduler (none, deadline, noop, cfq, bfq, mq-deadline, kyber)
host_io_scheduler: "none"
```

### Настройки лимитов

```yaml
# Системные лимиты
host_ulimit_nofile: 65536
host_ulimit_nproc: 65536

# Лимиты для libvirt
host_libvirt_limits:
  soft_nofile: 65536
  hard_nofile: 65536
  soft_nproc: 65536
  hard_nproc: 65536
  soft_memlock: "unlimited"
  hard_memlock: "unlimited"
```

### Настройки tuned

```yaml
# Tuned
host_tuned_enabled: true
host_tuned_profile: "virtual-host"
```

Полный список переменных см. в `defaults/main.yml`.

---

## Использование

### Проверка изменений (Check Mode / Dry-Run)

**ВСЕГДА используйте check mode перед применением!**

```bash
# Проверка изменений без применения
ansible-playbook playbook.yml --check

# Проверка с детальным выводом
ansible-playbook playbook.yml --check --diff

# Проверка только определенных тегов
ansible-playbook playbook.yml --check --tags memory,network
```

### Базовая оптимизация

```yaml
- hosts: virtualization_hosts
  become: true
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
    # Подтверждение опасных операций
    host_optimization_confirm_dangerous: true
  roles:
    - host-optimization
```

### С отключением swap (требует подтверждения)

```yaml
- hosts: virtualization_hosts
  become: true
  vars:
    host_swap_disabled: true
    # ОБЯЗАТЕЛЬНО установите в true для отключения swap!
    host_optimization_confirm_dangerous: true
  roles:
    - host-optimization
```

---

## Риски и безопасность

### Критические риски

1. **Изменение параметров ядра (GRUB)**
   - Может сделать систему неработоспособной
   - Требует перезагрузки
   - Защита: автоматический бэкап, предупреждения

2. **Изменение fstab**
   - Может привести к невозможности загрузки
   - Требует перезагрузки
   - Защита: автоматический бэкап, предупреждения

3. **Отключение swap**
   - Может привести к нехватке памяти
   - Может вызвать завершение процессов
   - Защита: проверки безопасности, требование подтверждения

### Рекомендации по безопасности

1. **Всегда используйте check mode:**
   ```bash
   ansible-playbook playbook.yml --check --diff
   ```

2. **Убедитесь в доступе к консоли:**
   - Консоль сервера
   - IPMI/KVM доступ
   - Rescue режим

3. **Создайте резервную копию:**
   - Полная резервная копия системы
   - Бэкапы конфигураций (создаются автоматически)

4. **Тестируйте на тестовом сервере:**
   - Не применяйте сразу на продакшене
   - Проверьте все изменения

5. **Применяйте поэтапно:**
   - Сначала безопасные операции
   - Затем критические с подтверждением

---

## Поэтапное применение оптимизаций

### Да, можно запускать оптимизацию хоста частями!

Роль поддерживает поэтапное применение через использование тегов. Это рекомендуется для:
- Безопасного тестирования на продакшене
- Применения критических изменений отдельно
- Отладки проблем с конкретными оптимизациями

### Рекомендуемый порядок поэтапного применения

#### Этап 1: Подготовка и проверка (безопасно)

```bash
# 1. Проверка системы и валидация переменных
ansible-playbook playbook.yml --tags check,validation --check

# 2. Проверка безопасности (без применения)
ansible-playbook playbook.yml --tags safety --check

# 3. Создание резервных копий (безопасно)
ansible-playbook playbook.yml --tags backup
```

**Что делается:**
- Собирается информация о системе
- Проверяются переменные на корректность
- Выполняются проверки безопасности
- Создаются резервные копии критических файлов

**Риски:** Нет - все операции безопасные

---

#### Этап 2: Безопасные оптимизации (не требуют перезагрузки)

```bash
# Установка пакетов
ansible-playbook playbook.yml --tags packages

# Оптимизация памяти (без swap и hugepages)
ansible-playbook playbook.yml --tags memory --skip-tags swap,hugepages

# Оптимизация сети
ansible-playbook playbook.yml --tags network,security

# Оптимизация CPU
ansible-playbook playbook.yml --tags cpu

# Настройка системных лимитов
ansible-playbook playbook.yml --tags limits

# Проверка применённых настроек
ansible-playbook playbook.yml --tags verify
```

**Что делается:**
- Устанавливаются необходимые пакеты
- Настраиваются sysctl параметры (память, сеть)
- Настраивается CPU governor
- Увеличиваются системные лимиты

**Риски:** Низкие - не требуют перезагрузки, легко откатить

**Преимущества:**
- Можно применить сразу
- Можно проверить работу системы
- Не требуют перезагрузки

---

#### Этап 3: Оптимизации, требующие перезагрузки (критично)

**ВНИМАНИЕ: Эти операции требуют перезагрузки и могут быть опасными!**

```bash
# 1. Настройка параметров ядра (GRUB)
# Требует перезагрузки!
ansible-playbook playbook.yml \
  --tags kernel,backup \
  -e "host_optimization_confirm_dangerous=true"

# После выполнения - ПЕРЕЗАГРУЗИТЕ СИСТЕМУ!
sudo reboot

# 2. Оптимизация файловых систем (fstab)
# Требует перезагрузки для полного применения!
ansible-playbook playbook.yml \
  --tags filesystem,backup \
  -e "host_optimization_confirm_dangerous=true"

# 3. Hugepages (если включены)
ansible-playbook playbook.yml --tags hugepages

# 4. Tuned профиль
ansible-playbook playbook.yml --tags tuned

# После всех изменений - ПЕРЕЗАГРУЗИТЕ СИСТЕМУ!
sudo reboot
```

**Что делается:**
- Изменяются параметры ядра в GRUB
- Изменяется fstab для noatime
- Настраивается I/O scheduler
- Настраивается tuned профиль

**Риски:** КРИТИЧЕСКИЕ - могут сделать систему неработоспособной

**Требования:**
- Доступ к консоли сервера
- Резервные копии созданы
- `host_optimization_confirm_dangerous: true`

---

#### Этап 4: Опциональные критические операции

```bash
# Отключение swap (только если достаточно RAM!)
# КРИТИЧЕСКАЯ ОПЕРАЦИЯ!
ansible-playbook playbook.yml \
  --tags swap,backup,safety \
  -e "host_swap_disabled=true" \
  -e "host_optimization_confirm_dangerous=true"
```

**Что делается:**
- Отключается swap
- Комментируются записи swap в fstab

**Риски:** КРИТИЧЕСКИЕ - может привести к нехватке памяти

**Требования:**
- Минимум 16GB RAM (проверяется автоматически)
- Использование swap < 10% (проверяется автоматически)
- `host_optimization_confirm_dangerous: true`

---

### Примеры использования тегов

#### Только безопасные операции (без перезагрузки)

```bash
# Все безопасные оптимизации без критических операций
ansible-playbook playbook.yml \
  --tags packages,memory,network,cpu,limits \
  --skip-tags swap,kernel,filesystem,hugepages
```

#### Только проверка системы

```bash
# Проверка без применения изменений
ansible-playbook playbook.yml --tags check,validation,verify --check
```

#### Только критические операции

```bash
# Только операции, требующие перезагрузки
ansible-playbook playbook.yml \
  --tags kernel,filesystem \
  -e "host_optimization_confirm_dangerous=true"
```

#### Комбинирование тегов

```bash
# Валидация + безопасные операции + проверка
ansible-playbook playbook.yml \
  --tags validation,packages,memory,network,cpu,limits,verify

# Валидация + проверки безопасности + резервное копирование
ansible-playbook playbook.yml \
  --tags validation,safety,backup
```

#### Исключение определенных тегов

```bash
# Все оптимизации, кроме критических
ansible-playbook playbook.yml \
  --skip-tags kernel,filesystem,swap

# Без tuned профиля
ansible-playbook playbook.yml --skip-tags tuned
```

---

### Зависимости между задачами

Важно понимать зависимости при поэтапном применении:

1. **Валидация и проверка** должны выполняться первыми:
   ```bash
   --tags validation,check
   ```

2. **Проверки безопасности** должны выполняться перед критическими операциями:
   ```bash
   --tags safety
   ```

3. **Резервное копирование** должно выполняться перед изменением файлов:
   ```bash
   --tags backup
   ```

4. **Проверка системы** нужна для автоматического определения CPU:
   ```bash
   --tags check  # Для kernel модуля
   ```

5. **Пакеты** должны быть установлены перед использованием:
   ```bash
   --tags packages  # Перед cpu, tuned
   ```

---

### Рекомендуемый сценарий поэтапного применения

#### Сценарий 1: Безопасное применение на продакшене

```bash
# Шаг 1: Подготовка (безопасно)
ansible-playbook playbook.yml --tags check,validation,backup

# Шаг 2: Безопасные оптимизации (не требуют перезагрузки)
ansible-playbook playbook.yml \
  --tags packages,memory,network,cpu,limits \
  --skip-tags swap,hugepages

# Шаг 3: Проверка работы системы
ansible-playbook playbook.yml --tags verify

# Шаг 4: Мониторинг в течение нескольких дней
# Проверьте: free -h, vmstat, iostat, netstat

# Шаг 5: Критические операции (если всё ок)
ansible-playbook playbook.yml \
  --tags kernel,filesystem \
  -e "host_optimization_confirm_dangerous=true"

# Шаг 6: Перезагрузка
sudo reboot

# Шаг 7: Проверка после перезагрузки
ansible-playbook playbook.yml --tags verify
cat /proc/cmdline
```

#### Сценарий 2: Только безопасные оптимизации (без перезагрузки)

```bash
# Все оптимизации, которые не требуют перезагрузки
ansible-playbook playbook.yml \
  --tags validation,check,backup,packages,memory,network,cpu,limits,verify \
  --skip-tags swap,kernel,filesystem,hugepages,tuned
```

#### Сценарий 3: Только оптимизация памяти и сети

```bash
# Только оптимизация памяти и сети
ansible-playbook playbook.yml \
  --tags validation,memory,network,verify \
  --skip-tags swap,hugepages
```

---

### Важные замечания при поэтапном применении

1. **Всегда выполняйте валидацию:**
   ```bash
   --tags validation
   ```

2. **Всегда создавайте бэкапы перед критическими операциями:**
   ```bash
   --tags backup
   ```

3. **Проверяйте результаты после каждого этапа:**
   ```bash
   --tags verify
   ```

4. **Используйте check mode для проверки:**
   ```bash
   --check --diff
   ```

5. **Критические операции требуют подтверждения:**
   ```bash
   -e "host_optimization_confirm_dangerous=true"
   ```

6. **Не пропускайте проверки безопасности:**
   ```bash
   --tags safety
   ```

---

### Откат изменений

Если что-то пошло не так, вы можете откатить изменения:

```bash
# Восстановление из бэкапов (ручное)
# См. раздел "Восстановление из бэкапа" в SECURITY.md

# Отключение tuned профиля
ansible-playbook playbook.yml --tags tuned -e "host_tuned_enabled=false"

# Восстановление CPU governor (ручное)
echo "ondemand" > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Восстановление swap (ручное)
swapon -a
# Раскомментировать swap в /etc/fstab
```

---

## Проверка и мониторинг

### Проверка применённых настроек

```bash
# Swappiness
sysctl vm.swappiness

# IP forwarding
sysctl net.ipv4.ip_forward

# CPU governor
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Hugepages
cat /proc/meminfo | grep HugePages

# Параметры ядра
cat /proc/cmdline

# Лимиты
ulimit -a

# I/O scheduler
cat /sys/block/*/queue/scheduler

# Tuned профиль
tuned-adm active

# Swap
swapon --show
```

### Мониторинг после применения

1. **Проверьте использование памяти:**
   ```bash
   free -h
   vmstat 1 5
   ```

2. **Проверьте производительность сети:**
   ```bash
   iperf3 -c <server>
   ```

3. **Проверьте производительность дисков:**
   ```bash
   fio --name=test --ioengine=libaio --iodepth=16 --rw=read --bs=4k --size=1G
   ```

4. **Проверьте логи системы:**
   ```bash
   journalctl -xe
   dmesg | tail -50
   ```

---

## Теги

- `optimization` - все задачи оптимизации
- `validation` - валидация переменных
- `check` - проверка системы
- `safety` - проверки безопасности
- `backup` - резервное копирование конфигураций
- `packages` - установка пакетов
- `memory` - оптимизация памяти
- `swap` - настройка swap
- `network` - оптимизация сети
- `security` - настройки безопасности
- `cpu` - оптимизация CPU
- `kernel` - настройка параметров ядра
- `filesystem` - оптимизация файловых систем
- `io` - настройка I/O scheduler
- `limits` - настройка лимитов
- `libvirt` - настройки для libvirt
- `tuned` - настройка tuned профилей
- `verify` - проверка применённых настроек

---

## Дополнительная документация

- `SECURITY.md` - Полное руководство по безопасности
- `ANALYSIS.md` - Анализ роли и возможностей
- `IMPROVEMENTS.md` - Описание улучшений

---

## Лицензия

ISC License

---

**Последнее обновление:** январь 2026

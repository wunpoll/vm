## Часть 1: Обоснование выбора программного обеспечения

1.  **Операционная система:** **Ubuntu Server 22.04 LTS**.
    *   **Обоснование:** Это последняя версия с долгосрочной поддержкой (LTS), что гарантирует стабильность и безопасность на 5 лет. Ubuntu Server имеет огромную базу документации, широкую поддержку со стороны разработчиков ПО и является стандартом де-факто для многих облачных и локальных развертываний.

2.  **Система управления конфигурациями:** **Ansible**.
    *   **Обоснование:** Ansible является **agentless** (безагентной) системой. Он управляет серверами по стандартному протоколу **SSH**, не требуя установки дополнительного ПО на управляемые машины. Это упрощает внедрение и снижает накладные расходы. Плейбуки пишутся на простом и читаемом языке YAML. В отличие от Puppet/Chef, у Ansible более низкий порог входа, что идеально для быстрого развертывания.

3.  **Платформа мониторинга:** **Prometheus + Grafana**.
    *   **Обоснование:** Это современный, облачно-ориентированный стек, ставший стандартом в DevOps. **Prometheus** отлично собирает метрики (pull-модель), а **Grafana** — мощнейший и гибкий инструмент для их визуализации. Этот стек легко расширяется с помощью "экспортеров" для мониторинга чего угодно: от состояния ОС до баз данных и приложений. Zabbix и Nagios — более старые, монолитные системы, менее гибкие в современных средах.

4.  **Инструмент для виртуализации:** **KVM (Kernel-based Virtual Machine) + Cockpit**.
    *   **Обоснование:** **KVM** — это нативное решение для виртуализации в ядре Linux, обеспечивающее высокую производительность. vSphere и Hyper-V — проприетарные продукты для других экосистем. Для удобного управления KVM из веб-интерфейса мы установим **Cockpit** — современную панель управления сервером, которая имеет встроенный модуль для создания и управления виртуальными машинами.

5.  **Система резервного копирования:** **Timeshift**.
    *   **Обоснование:** Для резервного копирования самого **управляющего сервера** Timeshift подходит идеально. Он создает инкрементальные "снимки" файловой системы (используя rsync), что позволяет быстро откатить системные файлы и конфигурации в случае сбоя после неудачного обновления или изменения настроек. Он прост в настройке и использовании как из командной строки, так и через GUI, если он установлен.

6.  **ПО для безопасности и шифрования:** **UFW (Uncomplicated Firewall), OpenSSH с аутентификацией по ключам**.
    *   **Обоснование:** **SSH** обеспечивает шифрованный канал для управления (и используется Ansible). Аутентификация по ключам вместо паролей — это стандарт безопасности, защищающий от брутфорс-атак. **UFW** — это простой и понятный интерфейс для управления `iptables`, позволяющий легко настроить правила брандмауэра и защитить сервер от несанкционированного доступа.

---

## Часть 2: Пошаговый план установки и настройки (Команды)

### Шаг 0: Подготовка системы и базовая безопасность

**Теория:** Обновляем систему и настраиваем брандмауэр, открывая только необходимые порты.

```bash
# Обновляем систему
sudo apt update && sudo apt full-upgrade -y

# Устанавливаем базовые утилиты
sudo apt install -y curl wget git htop

# Настраиваем брандмауэр UFW
# Разрешаем SSH (чтобы не потерять доступ!), HTTP/HTTPS и порты для нашего ПО
sudo ufw allow ssh        # Порт 22
sudo ufw allow http       # Порт 80
sudo ufw allow https      # Порт 443
sudo ufw allow 9090       # Порт для Prometheus и Cockpit
sudo ufw allow 3000       # Порт для Grafana
sudo ufw allow 9100       # Порт для Node Exporter (мониторинг)

# Включаем брандмауэр
sudo ufw enable # Нажмите 'y' для подтверждения
sudo ufw status
```

### Шаг 1: Установка и настройка Ansible

**Теория:** Устанавливаем Ansible и создаем простой плейбук для демонстрации его работы.

```bash
# Устанавливаем Ansible
sudo apt install -y ansible

# Проверяем версию
ansible --version

# Создадим тестовую конфигурацию для демонстрации
# Создаем файл инвентаря. Мы будем управлять самим собой (localhost)
echo "localhost ansible_connection=local" > hosts

# Создаем простой плейбук, который установит утилиту 'mc'
cat <<EOF > playbook.yml
---
- hosts: localhost
  become: yes
  tasks:
    - name: Install Midnight Commander
      apt:
        name: mc
        state: present
        update_cache: yes
EOF

# Запускаем плейбук
ansible-playbook -i hosts playbook.yml

# Проверяем, что mc установился
which mc
```

### Шаг 2: Установка стека мониторинга (Prometheus + Grafana)

**Теория:** Устанавливаем Prometheus для сбора данных, Node Exporter для предоставления метрик о системе и Grafana для их визуализации.

```bash
# Устанавливаем все компоненты из репозиториев
sudo apt install -y prometheus prometheus-node-exporter grafana

# Настроим Prometheus для сбора данных с Node Exporter
# Откроем конфиг Prometheus (можно использовать nano)
sudo nano /etc/prometheus/prometheus.yml

# В конец файла, в секцию 'scrape_configs', добавьте эту задачу:
# (Обычно там уже есть задача для самого Prometheus, добавьте эту рядом)
# - job_name: 'node_exporter'
#   static_configs:
#   - targets: ['localhost:9100']

# Запускаем и включаем автозагрузку служб
sudo systemctl enable --now prometheus
sudo systemctl enable --now grafana-server
sudo systemctl enable --now prometheus-node-exporter

# Проверяем статусы
systemctl status prometheus grafana-server prometheus-node-exporter
```
**Настройка в веб-интерфейсе:**
1.  **Prometheus:** Откройте в браузере `http://<IP_сервера>:9090`. Перейдите в `Status -> Targets`. Вы должны увидеть `node_exporter` в состоянии `UP`.
2.  **Grafana:** Откройте `http://<IP_сервера>:3000`. Логин/пароль по умолчанию: `admin`/`admin`. Grafana попросит сменить пароль.
3.  **Подключение Prometheus к Grafana:**
    *   Нажмите на шестеренку (Configuration) -> Data Sources -> Add data source.
    *   Выберите Prometheus.
    *   В поле URL введите `http://localhost:9090`.
    *   Нажмите "Save & Test". Должно появиться сообщение "Data source is working".
4.  **Импорт дашборда:**
    *   Нажмите на плюсик (Create) -> Import.
    *   В поле "Import via grafana.com" введите ID `1860` (это популярный дашборд для Node Exporter).
    *   Нажмите "Load", на следующем шаге выберите ваш источник данных Prometheus и нажмите "Import". Вы увидите красивый дашборд с состоянием вашего сервера.

### Шаг 3: Установка системы виртуализации (KVM + Cockpit)

**Теория:** Устанавливаем гипервизор KVM и веб-панель Cockpit для управления им.

```bash
# Устанавливаем все необходимые пакеты
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager cockpit cockpit-machines

# Добавляем своего пользователя в группу libvirt, чтобы управлять VM без sudo
sudo usermod -aG libvirt $USER

# Включаем и запускаем сокет Cockpit
sudo systemctl enable --now cockpit.socket

# Проверяем статус
systemctl status cockpit.socket
```
**Настройка в веб-интерфейсе:**
1.  Откройте в браузере `https://<IP_сервера>:9090` (именно **HTTPS**).
2.  Браузер выдаст предупреждение о самоподписанном сертификате — это нормально, нажмите "Продолжить".
3.  Войдите, используя свой логин и пароль от системы Linux.
4.  Слева в меню вы увидите раздел "Virtual Machines". Оттуда можно создавать, запускать и управлять виртуальными машинами.

### Шаг 4: Установка системы резервного копирования (Timeshift)

**Теория:** Ставим и демонстрируем создание снимка системы.

```bash
# Устанавливаем Timeshift
sudo apt install -y timeshift

# Создаем первый снимок системы из командной строки
sudo timeshift --create --comments "Initial server setup after exam configuration"

# Посмотреть список всех созданных снимков
sudo timeshift --list
```

---

## Часть 3: Руководство пользователя по ГОСТ Р 59795-2021

*(Создайте файл `timeshift_guide.md` и скопируйте туда этот текст. Это и будет ваш документ.)*

```markdown
# Руководство пользователя
## Система резервного копирования и восстановления Timeshift

### Введение

Настоящее руководство описывает порядок использования программы Timeshift для создания резервных копий (снимков) и восстановления состояния операционной системы на сервере управления.

Timeshift предназначен для защиты системных файлов и конфигураций. Программа не затрагивает пользовательские данные в домашних каталогах (`/home`), что делает ее идеальным инструментом для отката системных изменений, не затрагивая рабочие файлы пользователей.

### Требования к системе

*   **Операционная система:** Ubuntu Server 20.04+ или другая ОС на базе Linux.
*   **Права доступа:** Права администратора (sudo) для выполнения команд.
*   **Свободное место на диске:** Достаточное количество места для хранения снимков системы.

### Основные операции

Все операции выполняются из командной строки сервера.

#### 1. Создание снимка системы

Для создания мгновенной резервной копии текущего состояния системы используется следующая команда:

```bash
sudo timeshift --create --comments "Описание изменений, перед которыми делается снимок"
```
*   `--create`: Основная команда для создания снимка.
*   `--comments "Ваш комментарий"`: Необязательный, но крайне рекомендуемый параметр для добавления описания к снимку. Например: "Перед обновлением ядра системы".

#### 2. Просмотр списка существующих снимков

Для вывода таблицы всех имеющихся снимков, их даты создания и комментариев, выполните:

```bash
sudo timeshift --list
```
Вывод будет содержать уникальный идентификатор (имя) каждого снимка, который используется для операций восстановления и удаления.

#### 3. Восстановление системы из снимка

**ВНИМАНИЕ:** Восстановление системы является необратимой операцией, которая перезапишет текущие системные файлы и конфигурации файлами из выбранного снимка.

Для запуска процесса восстановления используется команда:

```bash
sudo timeshift --restore --snapshot 'ИМЯ_СНИМКА'
```
*   `--restore`: Команда для запуска восстановления.
*   `--snapshot 'ИМЯ_СНИМКА'`: Укажите точное имя снимка из списка, полученного командой `--list`. Например: `'2023-10-27_15-30-01'`.

Программа запросит подтверждение перед началом операции.

#### 4. Удаление снимка

Для освобождения дискового пространства можно удалить старые или ненужные снимки:

```bash
sudo timeshift --delete --snapshot 'ИМЯ_СНИМКА'
```

### Рекомендации по использованию

*   Создавайте снимки перед любыми значительными изменениями в системе: установкой нового ПО, обновлением ядра, изменением критически важных конфигурационных файлов.
*   Регулярно проверяйте наличие свободного места на диске и удаляйте устаревшие снимки.

---

## Часть 4: План демонстрации результатов экзаменатору

1.  **Показать подготовленную систему:** Открыть терминал, выполнить `neofetch`. Продемонстрировать работающий брандмауэр через `sudo ufw status`.
2.  **Продемонстрировать Ansible:** Показать файлы `hosts` и `playbook.yml`. Запустить плейбук `ansible-playbook -i hosts playbook.yml` и показать, что он выполняется успешно (статус `changed` или `ok`).
3.  **Продемонстрировать Мониторинг:**
    *   Открыть в браузере `http://<IP>:3000` (Grafana).
    *   Показать дашборд "Node Exporter Full", который отображает загрузку CPU, RAM, диска и сети в реальном времени. Пояснить, откуда берутся эти данные (Prometheus -> Node Exporter).
4.  **Продемонстрировать Виртуализацию:**
    *   Открыть в браузере `https://<IP>:9090` (Cockpit).
    *   Войти, перейти в раздел "Virtual Machines" и показать, что интерфейс для управления ВМ готов к работе.
5.  **Продемонстрировать Резервное копирование:**
    *   Выполнить в терминале `sudo timeshift --list`, чтобы показать уже созданный снимок.
    *   Пояснить, как с помощью команды `timeshift --restore` можно откатить систему.
6.  **Показать документацию:** Открыть файл `timeshift_guide.md` и кратко рассказать о его структуре.
7.  **Защитить выбор ПО:** Устно рассказать обоснование из Части 1, делая акцент на том, почему выбранные технологии (Ansible, Prometheus/Grafana) являются современными и эффективными.

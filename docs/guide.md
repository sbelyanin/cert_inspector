## Пробный тестовый запуск
### Рассмотрим простой вариант, когда информация о сертификатах собирается, обрабатывается и размещается на всех хостах.
### Каждый хост обрабатывается отдельно.
<a id="inventory"></a>
 - **Для пробного тестирования** будем использовать локальный хост и на него будут две ссылки `localhost` и `delegate`.
Для этого используем конфигурационный файл `inventory.yml`:
```yaml
all:
   children:
      cluster_hosts:
         hosts:
            localhost:
               ansible_host: localhost
               ansible_user: ansible
            delegate:
               ansible_host: localhost
               ansible_user: ansible
```
Замените значение переменной `ansible_user` на имя пользователя с доступом к целевым хостам.

<a id="проверим-доступность"></a>
 - **Проверим доступность** наших "хостов".
Предварительно убедитесь, что пользователь `ansible` есть в системе и у него есть необходимые права (для входа в систему через `ssh`).
   - Если аутентификация по паролю:
```bash
ansible all -m ping -i inventory.yml --ask-pass
```
   - Если аутентификация по `ssh` ключу:
```bash
ansible all -m ping -i inventory.yml
```

<a id="простой-playbook"></a>
 - Проверьте или отредактируйте **простой Playbook** `playbooks/cert_inspector.yml`: 
```yaml
- name: Certificate inspector
  hosts: cluster_hosts
  become: true
  roles:
     - role: cert_inspector
```
Удалите `become: true` если авторизация на хостах позволяет вам читать файлы с сертификатами без использования `sudo` или по другим причинам безопасности.

<a id="файл-с-метриками"></a>
 - Сделаем так, чтобы **файл с метриками** был уникальным на "хостах" и при выполнении задачи не перезаписывался.
Проверим файлы на наличие строк:
host_vars/delegate.yml
```yaml
metrics_host_file_path: "/tmp/prom_dhost_certs_metr.txt"
```
host_vars/localhost.yml
```yaml
metrics_host_file_path: "/tmp/prom_lhost_certs_metr.txt"
```
Непустая и объявленная переменная `metrics_host_file_path` определяет полный путь до файла с метриками на текущем хосте.
Файл с метриками будет создаваться и содержать метрики с текущего хоста.

<a id="переменных-хостов"></a>
 - Проверьте или отредактируйте файлы для **переменных хостов** для указания директорий где и с какими параметрами искать сертификаты.
host_vars/delegate.yml
```yaml
ci_scan_directories:
  - path: "/etc"
    max_size: "-1m"
    file_type: file
```
host_vars/localhost.yml
```yaml
ci_scan_directories:
  - path: "/usr"
    max_size: "-1m"
    file_type: file
```
Непустая переменная `ci_scan_directories` должна содержать значение `path`, остальные не указанные значения подставляются из `cert_inspector/roles/cert_inspector/defaults/main.yml`.

<a id="запустите-playbook"></a>
 - **Запустите Playbook**:
```bash
ansible-playbook playbooks/cert_inspector.yml -i inventory.yml
```

<a id="проверим-результат"></a>
 - **Проверим результат**:
```bash
ls /tmp/prom_dhost_certs_metr.txt /tmp/prom_lhost_certs_metr.txt
/tmp/prom_dhost_certs_metr.txt  /tmp/prom_lhost_certs_metr.txt

cat /tmp/prom_dhost_certs_metr.txt
# HELP cert_expiry_seconds Time until certificate expires in seconds
# TYPE cert_expiry_seconds gauge
cert_expiry_seconds{cert_path="/etc/ssl/certs/ca-certificates.crt", issuer="ACCVRAIZ1", subject="email:accv@accv.es"} 178476948
cert_expiry_seconds{cert_path="/etc/pki/fwupd-metadata/LVFS-CA.pem", issuer="LVFS CA", subject="URI:http://www.fwupd.org/"} 701767091

cat /tmp/prom_lhost_certs_metr.txt
# HELP cert_expiry_seconds Time until certificate expires in seconds
# TYPE cert_expiry_seconds gauge
cert_expiry_seconds{cert_path="/usr/share/grub/canonical-uefi-ca.crt", issuer="Canonical Ltd. Master Certificate Authority", subject="Canonical Ltd. Master Certificate Authority"} 534364262
cert_expiry_seconds{cert_path="/usr/share/gnupg/sks-keyservers.netCA.pem", issuer="sks-keyservers.net CA", subject="sks-keyservers.net CA"} -81360492
```
Отрицательное значение метрики `cert_expiry_seconds` означает, что срок действия сертификата истек.

## Использование тэгов
<a id="no-tags"></a>
 - **Запуск роли без тэгов.**
При запуске плайбука без указания тэгов запускается только основная часть роли `cert inspector`, которая выполняет задачи в зависимости от [роли хостов](#роли-хостов).
Основная чсть будет запускаться по умолчанию даже с указанием тэгов, такое поведение можно изменить использовав параметр `--skip-tags "inspector"`.
#### Запуск
```bash
ansible-playbook playbooks/cert_inspector.yml -i inventory.yml
```
<a id="list-tags"></a>
 - **Какие тэги есть в роли на данный момент.**
   - `ci_nginx` - запускает или пропускает (по умолчанию) часть роли отвечающей за создание конфигурационного файла для `nginx`.
   - `ci_nginx` - запускает или пропускает (по умолчанию) часть роли отвечающей за создание конфигурационного файла для `prometheus`.
   - `ci_inspector` - запускает (по умолчанию) или пропускает основную часть роли `cert inspector`.
<a id="use-tags"></a>
 - **Запуск роли с тэгами.**
   - Запуск основной части и дополнительной части для `nginx`:
```bash
ansible-playbook playbooks/cert_inspector.yml -i inventory.yml --tags "ci_nginx"
```

## Роли хостов
#### Роли хостов это не роли Ansible.
<a id="local-scanner"></a>
 - **Local Scanner**: Сканирует локальные директории, находит файлы с сертификатами и создает на их основе локальный список для метрик.
#### Переменные:
`ci_scan_directories` объявлена и в списке присутствует хотя бы один не пустой элемент `path`.
Задает одну из ролей хоста -`Local Scanner`.
Задает параметры сканирования директорий и нахождения файлов.
#### Одна или обе переменных:
`metrics_host_file_path` объявлена и не пуста.
Задает полный путь до файла куда будут записываться метрики с локального хоста.
`ci_metrics_aggregate_host` объявлена и не пуста.
Задает имя хоста из инвентори, который будет агрегировать локальные метрики с локального хоста.
<a id="local-sender"></a>
 - **Local Sender**: Часть роли `Local Scanner` в которой переменная `ci_metrics_aggregate_host` объявлена и не пуста.
Передает локальные метрики на хост, выполняющий роль `Aggregate Exporter`.
<a id="local-exporter"></a>
 - **Local Exporter**: Часть роли `Local Scanner` в которой переменная `metrics_host_file_path` объявлена и не пуста.
Передает локальные метрики в локальный файл заданный в `metrics_host_file_path`.
<a id="aggregate-exporter"></a>
 - **Aggregate Exporter**: Копирует локальные метрики с хостов, агрегирует их в единый список и записывает в файл, указанный в `metrics_aggregate_file_path`.

## Выбор роли хоста
#### Хостовые переменные в `host_vars/` имеют больший приоритет, чем групповые в `group_vars/`. Это надо учитывать при настройке окружения.

### Сценарий 1: 
#### Много одинаковых хостов, сертификаты в одних директориях, метрики на тех же хостах.
**Роли хостов**:
- **Local Scanner + Local Exporter**:
  На хостах `cluster_hosts` сканируются директория `/etc/ssl/certs`, извлекается данные о сертификатах и сохраняются метрики в локальный файл `tmp/prom_host_certs_metr.txt`.
**Настройки**:
- **Инвентори** `inventory.yml`:
```yaml
all:
   children:
      cluster_hosts:
         hosts:
             host1:
             host2:
             host3:
```
- **Групповые переменные** `group_vars/cluster_hosts.yml`:
```yaml
ci_scan_directories:
  - path: "/etc/ssl/certs"
    max_depth: 3
    file_type: file
metrics_host_file_path: "/tmp/prom_host_certs_metr.txt"
```
- **Playbook и запуск** `playbooks/cert_inspector.yml`:
```yaml
- name: Certificate inspector
  hosts: cluster_hosts
  become: true
  roles:
     - role: cert_inspector
```
```bash
ansible-playbook playbooks/cert_inspector.yml -i inventory.yml
```

### Сценарий 2:
#### Много одинаковых хостов, сертификаты в одних директориях, метрики на выделеннй хост.
**Роли хостов**:
- **Local Scanner + Local Exporter**:
  На хостах `cluster_hosts` сканируются директория `/etc/ssl/certs`, извлекается данные о сертификатах и передаются метрики на хост `metrics_host` для агрегации.
- **Aggregate Exporter**:
  Хост `metrics_host` копирует локальные метрики с хостов `cluster_hosts`, агрегирует их в единый список и записывает в файл `/tmp/prom_host_certs_metr.txt`.
**Настройки**:
- **Инвентори** `inventory.yml`:
```yaml
all:
   children:
      cluster_hosts:
         hosts:
            host1:
            host2:
            host3:
            metrics_host:
```
- **Групповые переменные**
#### Для сканеров `group_vars/cluster_hosts.yml`:
```yaml
ci_scan_directories:
  - path: "/etc/ssl/certs"
    max_depth: 3
    file_type: file
ci_metrics_aggregate_host: "metrics_host"
```
- **Хостовые переменные**
#### Для экспортера `host_vars/metrics_host.yml`:
```yaml
metrics_aggregate_file_path: "/tmp/prom_host_certs_metr.txt"
```
- **Playbook и запуск** `playbooks/cert_inspector.yml`:
```yaml
- name: Certificate inspector
  hosts: cluster_hosts
  become: true
  roles:
     - role: cert_inspector
```
```bash
ansible-playbook playbooks/cert_inspector.yml -i inventory.yml
```

### Сценарий 3:
#### Много хостов, сертификаты в разных директориях, гибридная агрегация метрик.
**Окружение**:
- 6 хостов (`host1`-`host6`), объединенных в группу `cluster_hosts`.
- Сертификаты расположены в уникальных директориях на каждом хосте.
- `host5` и `host6` выступают агрегаторами:
  - `host5` собирает метрики с `host1`, `host2`, `host6`.
  - `host6` собирает метрики с `host3`, `host4`, `host5`.
- `host1` и `host3` сохраняют локальные метрики в отдельные файлы, остальные хосты не экспортируют локальные метрики.
- Агрегированные метрики сохраняются на `host5` и `host6` в разные файлы.

**Роли хостов**:
- **Local Scanner + Local Sender**:
  Все хосты сканируют свои директории и отправляют метрики на агрегаторы.
- **Local Scanner + Local Exporter**:
  `host1`, `host3` дополнительно сохраняют локальные метрики в файлы.
- **Aggregate Exporter**:
  `host5` и `host6` агрегируют полученные метрики и сохраняют их в файлы.

**Настройки**:
- **Инвентори** `inventory.yml`:
```yaml
all:
  children:
    cluster_hosts:
      hosts:
        host1:
        host2:
        host3:
        host4:
        host5:
        host6:
```
- **Хостовые переменные**:
#### Для `host1` `host_vars/host1.yml`:
```yaml
metrics_host_file_path: "/tmp/host1_certs_metrics.txt"  # Локальный экспорт
ci_metrics_aggregate_host: "host5"  # Отправка метрик на host5
ci_scan_directories:
  - path: "/etc/host1_certs"  # Уникальная директория
```
#### Для `host2` `host_vars/host2.yml`:
```yaml
ci_metrics_aggregate_host: "host5"  # Отправка на host5
ci_scan_directories:
  - path: "/opt/certs"
```
#### Для `host3` `host_vars/host3.yml`:
```yaml
metrics_host_file_path: "/tmp/host3_certs_metrics.txt"  # Локальный экспорт
ci_metrics_aggregate_host: "host6"  # Отправка на host6
ci_scan_directories:
  - path: "/var/ssl"
```
#### Для `host4` `host_vars/host4.yml`:
```yaml
ci_metrics_aggregate_host: "host6"  # Отправка на host6
ci_scan_directories:
  - path: "/home/user/certs"
```
#### Для `host5` `host_vars/host5.yml`:
```yaml
metrics_aggregate_file_path: "/tmp/agg_host5_metrics.txt"  # Агрегированные метрики
ci_metrics_aggregate_host: "host6"  # Отправка на host6
ci_scan_directories:
  - path: "/etc/host5_certs"  # Уникальная директория
```
#### Для `host6` `host_vars/host6.yml`:
```yaml
metrics_aggregate_file_path: "/tmp/agg_host6_metrics.txt"  # Агрегированные метрики
ci_metrics_aggregate_host: "host5"  # Отправка на host5
ci_scan_directories:
  - path: "/etc/nginx/ssl"  # host6 также сканирует свои сертификаты
```
- **Playbook и запуск** `playbooks/cert_inspector.yml`:
```yaml
- name: Certificate inspector
  hosts: cluster_hosts
  become: true
  roles:
     - role: cert_inspector
```
```bash
ansible-playbook playbooks/cert_inspector.yml -i inventory.yml
```

<a id="setup-nginx"></a>
## Настройки Nginx для экспорта метрик
#### Для экспорта метрик нужно создать конфигурационный файл `nginx`. Конфиг создаются для хостов с ролями [Local](#local-exporter) и [Aggregate](#aggregate-exporter) Exporter. Запустить часть роли отвечающей за настройку `nginx` можно при помощи тэга [`ci_nginx`](#list-tags).
#### Эта часть роли является дополнением к основной, а не отдельной частью.

<a id="setup-nginx-subset"></a>
- **Что делает эта часть**
  - Устанавливает пакет `Nginx` на целевом хосте (`target host`).
  - Создаёт из шаблона конфигурационный файл для виртуальных хостов (`vhost`) и подключает его в конфигурацию `Nginx`.
  - Выполняет перезагрузку конфигурации `Nginx` при необходимости.

<a id="setup-nginx-metr"></a>
- **Доступ к метрикам**
  - [Локальные метрики](#local-exporter) будут доступны по адресу `http://YOU_IP:8880/metrics`
  - [Агрегированые метрики](#aggregate-exporter) будут доступны по адресу ` http://YOU_IP:8881/metrics`

<a id="setup-nginx-auth"></a>
- **Для доступа к метрикам реализована простая аутентификация.**
  - `auth_user: "metrics_user"`
  - `auth_password: "secure_password"`

<a id="setup-nginx-start"></a>
- **Запуск**
```bash
ansible-playbook playbooks/cert_inspector.yml -i inventory.yml --tags "ci_nginx"
```
```bash
TASK [cert_inspector : Set nginx_conf_path with fallback] *****************************************************************************************************************
TASK [cert_inspector : Set nginx_enabled_conf_path with fallback] *********************************************************************************************************
TASK [cert_inspector : Set nginx_link_conf_type with fallback] ************************************************************************************************************
TASK [cert_inspector : Install Nginx/Angie] *******************************************************************************************************************************
TASK [cert_inspector : Generate .htpasswd file] ***************************************************************************************************************************
TASK [cert_inspector : Deploy host metrics Nginx config] ******************************************************************************************************************
TASK [cert_inspector : Deploy aggregate metrics Nginx config] *************************************************************************************************************
TASK [cert_inspector : Enable host metrics Nginx config] ******************************************************************************************************************
TASK [cert_inspector : Enable aggregate metrics Nginx config] *************************************************************************************************************
PLAY RECAP ****************************************************************************************************************************************************************
delegate                   : ok=14   changed=2    unreachable=0    failed=0    skipped=15   rescued=0    ignored=0   
localhost                  : ok=21   changed=2    unreachable=0    failed=0    skipped=8    rescued=0    ignored=0
```

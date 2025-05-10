# cert_inspector Ansible Role

## Описание
Роль **cert_inspector** предназначена для автоматизированного сканирования каталогов на наличие SSL-сертификатов, сбора их метаданных (срок действия, выдавший орган (CA), назначение сертификата) и предоставления метрик в формате, совместимом с Prometheus. Это позволяет отслеживать состояние сертификатов и предотвращать их истечение срока действия.

### Основные задачи:
- Сканирование каталогов с помощью Ansible модуля `ansible.builtin.find`.
- Извлечение информации о сертификатах через `community.crypto.x509_certificate_info`.
- Генерация метрик в файл через шаблон Jinja2 для последующей интеграции с Prometheus через Nginx.
- Гибкая настройка параметров сканирования (вложенность, маски, фильтры) и агрегирования метрик.

## Отличия от обычных экспортеров
- **Интеграция с Ansible**: Использует существующую инфраструктуру Ansible, не требуя отдельных, постоянно запущенных демонов или сервисов с доступом к сертификатам.
- **Разделение сервисов**: Разделяет процессы сканирования, агрегации и экспорта метрик, что упрощает управление и масштабирование.
- **Автоматизация поиска**: Поддерживает параметризованный поиск файлов, включая игнорирование символических ссылок, уровень вложенности и ограничение размера файлов.
- **Гибкость настройки**: Позволяет задавать уникальные параметры для каждой директории (путь, уровень вложенности, фильтры) с возможностью использования значений по умолчанию.
- **Агрегирование метрик**: Позволяет собирать метрики с разных узлов и публиковать их на других узлах.

## Требования
- **Ansible** ≥ 2.10.
- Модуль `community.crypto` (установить через `ansible-galaxy collection install community.crypto`).
- Python ≥ 3.6.
- Jinja2 для обработки шаблонов.
- Nginx для отдачи метрик Prometheus (требуется только для интеграции с Prometheus).
- Доступ к каталогам с сертификатами (права чтения).

## Установка
Вручную из репозитория:
```bash
git clone https://github.com/sbelyanin/cert_inspector.git
cd cert_inspector
```

## Пробный тестовый запуск
### Рассмотрим простой вариант, когда информация о сертификатах собирается, обрабатывается и размещается на всех хостах.
### Каждый хост обрабатывается отдельно.
1. Для пробного тестирования будем использовать локальный хост и на него будут две ссылки `localhost` и `delegate`.
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

2. Проверим доступность наших "хостов".
Предварительно убедитесь, что пользователь `ansible` есть в системе и у него есть необходимые права (для входа в систему через `ssh`).
Если аутентификация по паролю:
```bash
ansible all -m ping -i inventory.yml --ask-pass
```
Если аутентификация по `ssh` ключу:
```bash
ansible all -m ping -i inventory.yml
```

3. Проверьте или отредактируйте простой Playbook `playbooks/cert_inspector.yml`: 
```yaml
- name: Certificate inspector
  hosts: cluster_hosts
  become: true
  roles:
     - role: cert_inspector
```
Удалите `become: true` если авторизация на хостах позволяет вам читать файлы с сертификатами без использования `sudo` или по другим причинам безопасности.

4. Сделаем так, чтобы файл с метриками был уникальным на "хостах" и при выполнении задачи не перезаписывался.
Проверим файлы на наличие строк:
host_vars/delegate.yml
```yaml
metrics_host_file_path: "/tmp/prom_dhost_certs_metr.txt"
metrics_host: true
```
host_vars/localhost.yml
```yaml
metrics_host_file_path: "/tmp/prom_lhost_certs_metr.txt"
metrics_host: true
``` 
Непустая и объявленная переменная `metrics_host_file_path` определяет полный путь до файла с метриками на текущем хосте.
Файл с метриками будет создаваться и содержать метрики с текущего хоста, которые собираются при `metrics_host: true`.

5. Проверьте или отредактируйте файлы для переменных хостов для указания директорий где и с какими параметрами искать сертификаты.
host_vars/delegate.yml
```yaml
scan_directories:
  - path: "/etc"
    max_size: "-1m"
    file_type: file
``` 
host_vars/localhost.yml
```yaml
scan_directories:
  - path: "/usr"
    max_size: "-1m"
    file_type: file
``` 
Непустая переменная `scan_directories` должна содержать значение `path`, остальные не указанные значения подставляются из `cert_inspector/roles/cert_inspector/defaults/main.yml`.
При `metrics_host: true` будут создаваться локальные метрики из файлов сертификатов найденных по параметрам, указанным в `scan_directories`.

6. Запустите:
```bash
ansible-playbook playbooks/cert_inspector.yml -i inventory.yml
```

7. Проверим результат:
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

## [Роли хостов](./docs/guide.md#роли-хостов)
 - **Local Scanner**
 - **Local Sender**
 - **Local Exporter**
 - **Aggregate Exporter**

## [Приоритет переменных в Ansible](./docs/variable-precedence.md)

## [Выбор роли хоста](./docs/guide.md#выбор-роли-хоста)
- *[Сценарий 1](./docs/guide.md#сценарий-1)*
Много одинаковых хостов, сертификаты в одних директориях, метрики на тех же хостах
- *[Сценарий 2](./docs/guide.md#сценарий-2)*
Много одинаковых хостов, сертификаты в одних директориях, метрики на выделеннй хост
- *[Сценарий 3](./docs/guide.md#сценарий-3)*
Много хостов, сертификаты в разных директориях, гибридная агрегация метрик

## Настройки Nginx для экспорта метрик
```nginx
location /metrics {
    alias /tmp;
    try_files $uri prom_lhost_certs_metr.txt prom_dhost_certs_metr.txt =404;
}
```

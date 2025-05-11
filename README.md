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

## [Пробный тестовый запуск](./docs/guide.md#пробный-тестовый-запуск)
- **[Inventory](./docs/guide.md#inventory)**
- **[Проверка доступности](./docs/guide.md#проверим-доступность)**
- **[Простой Playbook](./docs/guide.md#простой-playbook)**
- **[Файл с метриками](./docs/guide.md#файл-с-метриками)**
- **[Переменные хостов](./docs/guide.md#переменных-хостов)**
- **[Запуск playbook](./docs/guide.md#запустите-playbook)**
- **[Проверка результата](./docs/guide.md#проверим-результат)**

## [Роли хостов](./docs/guide.md#роли-хостов)
 - **[Local Scanner](./docs/guide.md#local-scanner)**
 - **[Local Sender](./docs/guide.md#local-sender)**
 - **[Local Exporter](./docs/guide.md#local-exporter)**
 - **[Aggregate Exporter](./docs/guide.md#aggregate-exporter)**

## [Приоритет переменных в Ansible](./docs/variable-precedence.md)
 - **[Таблица приоритетов](./docs/variable-precedence.md#таблица)**
 - **[Примеры установки значений](./docs/variable-precedence.md#примеры)**
 - **[Ключевые моменты](./docs/variable-precedence.md#ключевые)**
 - **[Как проверить приоритеты](./docs/variable-precedence.md#как-проверить)**

## [Выбор роли хоста](./docs/guide.md#выбор-роли-хоста)
 - **[Сценарий 1](./docs/guide.md#сценарий-1)**
Много одинаковых хостов, сертификаты в одних директориях, метрики на тех же хостах
 - **[Сценарий 2](./docs/guide.md#сценарий-2)**
Много одинаковых хостов, сертификаты в одних директориях, метрики на выделеннй хост
 - **[Сценарий 3](./docs/guide.md#сценарий-3)**
Много хостов, сертификаты в разных директориях, гибридная агрегация метрик

## Настройки Nginx для экспорта метрик
```nginx
server {
    listen 8880;
    server_name _;

    location /metrics {
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;
        alias /var/lib/metrics/prom_dhost_certs_metr.txt;
        default_type text/plain;
        try_files $uri =404;
    }

    location / {
        return 404;
    }
}
server {
    listen 8881;
    server_name _;

    location /metrics {
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;
        alias /var/lib/metrics/prom_aggr_dhost_certs_metr.txt;
        default_type text/plain;
        try_files $uri =404;
    }

    location / {
        return 404;
    }
}
```

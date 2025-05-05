# cert_inspector Ansible Role

## Описание
Роль **cert_inspector** предназначена для автоматизированного сканирования системных каталогов на наличие SSL-сертификатов, сбора их метаданных (срок действия, выдавший орган, имя хоста) и предоставления метрик в формате, совместимом с Prometheus. Это позволяет отслеживать состояние сертификатов в реальном времени и предотвращать их истечение срока действия.

### Основные задачи:
- Сканирование каталогов с помощью модуля `find`.
- Извлечение информации о сертификатах через `community.crypto.x509_certificate_info`.
- Генерация метрик в файле через шаблон Jinja2 для последующей интеграции с Prometheus через NGinx.
- Гибкая настройка параметров сканирования (вложенность, маски, фильтры).

## Отличия от обычных экспортеров
- **Интеграция с Ansible**: Использует существующую инфраструктуру Ansible, не требуя отдельных демонов или сервисов.
- **Автоматизация поиска**: Поддерживает параметризуемый поиск файлов, включая игнорирование символических ссылок и ограничение размера файлов.
- **Гибкость настройки**: Позволяет задавать уникальные параметры для каждой директории (путь, глубина сканирования, фильтры) с возможностью использования значений по умолчанию.

## Требования
- **Ansible** ≥ 2.10.
- Модуль `community.crypto` (установить через `ansible-galaxy collection install community.crypto`).
- Python ≥ 3.6.
- Jinja2 для обработки шаблонов.
- NGinx для отдачи метрик Prometheus.
- Доступ к каталогам с сертификатами (права чтения).

## Установка
Вручную из репозитория:
```bash
git clone https://github.com/sbelyanin/cert_inspector.git
cd cert_inspector
```

## Пробный запуск 
### Рассмотрим простой вариант когда информация о сертификатах собирается, обрабатывается и выкладывается на всех хостах.
### Каждый хост обрабатывается отдельно.
1. Для пробноого тестирования будем использовать локальный хост и на него будут две ссылки localhost и delegate.
Для этого используем конфигурационный файл inventory.yml:
```bash
all:
   hosts:
       localhost:
         ansible_host: localhost
         ansible_user: ansible
       delegate:
         ansible_host: localhost
         ansible_user: ansible
   children:
      test:
         hosts:
            localhost:
            delegate:
```

2. Проверим доступность наших "хостов".
Предварительно убедитесь что пользователь ansible есть в системе и у него есть необходиме права (для входа в систему через ssh).
Если аутентификация по паролю:
```bash
ansible all -m ping -i inventory.yml --ask-pass
```
Если аутентификация по ssh ключу:
```bash
ansible all -m ping -i inventory.yml
```

3. Проверьте или отредактируйте простой Playbook playbooks/cert_inspector.yml: 
```bash
---
- name: Certificate inspector
  hosts: test
  become: true
  roles:
     - role: cert_inspector
```
Удалите "become: true" если авторизация на хостах вам позволяет читать файлы с сертификатами без ипспользования sudo или по другим причинам безопастности.

4. Сделаем так, чтобы файл с метриками был уникальным на "хостах" и при выполнении задачи не затерался.
Проверим файлы на наличие строк:
cert_inspector/host_vars/delegate.yml
```bash
metrics_host_file_path: "/tmp/prom_dhost_certs_metr.txt"
metrics_host: true
```
cert_inspector/host_vars/localhost.yml
```bash
metrics_host_file_path: "/tmp/prom_lhost_certs_metr.txt"
metrics_host: true
``` 
Не пустая и обьявленная переменная metrics_host_file_path определяет полный путь до файла с метриками на текущем хосте.
Файл с метриками будет создаваться и содержать метрики с текущего хоста, которые собираются при metrics_host: true.

5. Проверьте или отредактируйте файлы для переменных хостов для указания директорий где и с какими параметрами искать сертификаты.
cert_inspector/host_vars/delegate.yml
```bash
scan_directories:
  - path: "/etc"
    max_size: "-1m"
    file_type: file
``` 
cert_inspector/host_vars/localhost.yml
```bash
scan_directories:
  - path: "/usr"
    max_size: "-1m"
    file_type: file
``` 
Не пустая переменная scan_directories должна содержать значение path, остальные не указанные значения подставляются из cert_inspector/roles/cert_inspector/defaults/main.yml.
При metrics_host: true будут создаваться локальные метрики из файлов сертификатов найденных по параметрам из scan_directories.

6. Запустите:
```bash
ansible-playbook playbooks/certificate_scanner.yml -i inventory.yml
```

7. Проверим результат:
```bash
ls /tmp/prom_dhost_certs_metr.txt /tmp/prom_lhost_certs_metr.txt
/tmp/prom_dhost_certs_metr.txt  /tmp/prom_lhost_certs_metr.txt

cat /tmp/prom_dhost_certs_metr.txt /tmp/prom_lhost_certs_metr.txt
```


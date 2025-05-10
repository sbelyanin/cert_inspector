## Роли хостов
<a id="local-scanner"></a>
 - **Local Scanner**: Сканирует локальные директории, находит файлы с сертификатами и создает на их основе локальный список для метрик.
#### Переменные:
`metrics_host:` объявлена и её значение равно `true`.
Задает одну из ролей хоста -`Local Scanner`.
`scan_directories` объявлена и в списке присутствует хотя бы один не пустой элемент `path`.
Задает параметры сканирования директорий и нахождения файлов.
#### Одна или обе переменных:
`metrics_host_file_path` объявлена и не пуста.
Задает полный путь до файла куда будут записываться метрики с локального хоста.
`metrics_aggregate_delegate_host` объявлена и не пуста.
Задает имя хоста из инвентори, который будет агрегировать локальные метрики с локального хоста.
<a id="local-sender"></a>
 - **Local Sender**: Часть роли `Local Scanner` в которой переменная `metrics_aggregate_delegate_host` объявлена и не пуста.
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
metrics_host: true
scan_directories:
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
metrics_host: true
scan_directories:
  - path: "/etc/ssl/certs"
    max_depth: 3
    file_type: file
metrics_aggregate_delegate_host: "metrics_host"
```
- **Хостовые переменные**
#### Для экспортера `host_vars/metrics_host.yml`:
```yaml
metrics_host: false
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
- **Групповые переменные**  `group_vars/cluster_hosts.yml`:
```yaml
metrics_host: true  # Все хосты по умолчанию сканируют сертификаты
```
- **Хостовые переменные**:
#### Для `host1` `host_vars/host1.yml`:
```yaml
metrics_host_file_path: "/tmp/host1_certs_metrics.txt"  # Локальный экспорт
metrics_aggregate_delegate_host: "host5"  # Отправка метрик на host5
scan_directories:
  - path: "/etc/host1_certs"  # Уникальная директория
```
#### Для `host2` `host_vars/host2.yml`:
```yaml
metrics_aggregate_delegate_host: "host5"  # Отправка на host5
scan_directories:
  - path: "/opt/certs"
```
#### Для `host3` `host_vars/host3.yml`:
```yaml
metrics_host_file_path: "/tmp/host3_certs_metrics.txt"  # Локальный экспорт
metrics_aggregate_delegate_host: "host6"  # Отправка на host6
scan_directories:
  - path: "/var/ssl"
```
#### Для `host4` `host_vars/host4.yml`:
```yaml
metrics_aggregate_delegate_host: "host6"  # Отправка на host6
scan_directories:
  - path: "/home/user/certs"
```
#### Для `host5` `host_vars/host5.yml`:
```yaml
metrics_aggregate_file_path: "/tmp/agg_host5_metrics.txt"  # Агрегированные метрики
metrics_aggregate_delegate_host: "host6"  # Отправка на host6
scan_directories:
  - path: "/etc/host5_certs"  # Уникальная директория
```
#### Для `host6` `host_vars/host6.yml`:
```yaml
metrics_aggregate_file_path: "/tmp/agg_host6_metrics.txt"  # Агрегированные метрики
metrics_aggregate_delegate_host: "host5"  # Отправка на host5
scan_directories:
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

### Справочник: Приоритет переменных в Ansible

---

<a id="таблица"></a>
#### Таблица приоритетов переменных (от низшего к высшему) 

| Приоритет | Источник переменной                          | Уточнения                                                                       |
|-----------|-----------------------------------------------|---------------------------------------------------------------------------------|
| 1         | **Role Defaults**                             | `roles/role_name/defaults/main.yml`                                             |
| 2         | **Inventory Group Variables (all)**           | `group_vars/all.yml` в каталоге инвентаря                                       |
| 3         | **Inventory Group Variables**                 | `group_vars/группа.yml` для конкретной группы (инвентарь)                       |
| 4         | **Inventory Host Variables**                  | `host_vars/хост.yml` в каталоге инвентаря                                       |
| 5         | **Playbook Group Variables (all)**            | `group_vars/all.yml` в каталоге плейбука                                        |
| 6         | **Playbook Group Variables**                  | `group_vars/группа.yml` в каталоге плейбука                                     |
| 7         | **Playbook Host Variables**                   | `host_vars/хост.yml` в каталоге плейбука                                        |
| 8         | **Host Facts**                                | Данные, собранные через `gather_facts: true`                                    |
| 9         | **Play Vars**                                 | Переменные в `vars` секции плейбука                                             |
| 10        | **Role Vars**                                 | `roles/role_name/vars/main.yml`                                                 |
| 11        | **Block Vars**                                | Переменные в блоке `block`                                                      |
| 12        | **Task Vars**                                 | Переменные внутри задачи (`tasks`)                                              |
| 13        | **Include_vars**                              | Переменные из файлов, загруженных через `include_vars`                          |
| 14        | **Set_fact / Registered Variables**           | Динамические переменные, созданные в задачах                                    |
| 15        | **Role Parameters**                           | Параметры, переданные роли напрямую (без `vars:`)                               |
| 16        | **Extra Variables (`-e`)**                    | Переменные из командной строки (`ansible-playbook -e "var=value"`)              |

---

<a id="примеры"></a>
#### Примеры установки значений (по приоритету)

| Источник                     | Пример кода                                                                 |
|------------------------------|-----------------------------------------------------------------------------|
| **Role Defaults**            | ```yaml # roles/nginx/defaults/main.yml nginx_port: 80 ```                  |
| **Inventory Group (all)**    | ```yaml # inventory/group_vars/all.yml nginx_port: 8080 ```                 |
| **Inventory Group (webservers)** | ```yaml # inventory/group_vars/webservers.yml nginx_port: 8081 ```      |
| **Inventory Host (web1)**    | ```yaml # inventory/host_vars/web1.yml nginx_port: 8082 ```                 |
| **Playbook Group (all)**     | ```yaml # playbook/group_vars/all.yml nginx_port: 8083 ```                  |
| **Playbook Host (web1)**     | ```yaml # playbook/host_vars/web1.yml nginx_port: 8084 ```                  |
| **Play Vars**                | ```yaml - hosts: all   vars:     nginx_port: 8085 ```                       |
| **Role Vars**                | ```yaml # roles/nginx/vars/main.yml nginx_port: 8086 ```                    |
| **Set_fact**                 | ```yaml - name: Set variable     set_fact:       nginx_port: 8087 ```       |
| **Extra Variables**          | ```bash ansible-playbook playbook.yml -e "nginx_port=8088" ```              |

---

<a id="ключевые"></a>
#### Ключевые моменты для `group_vars` и `host_vars`:
1. **Иерархия внутри инвентаря**:
   - `group_vars/all.yml` → `group_vars/группа.yml` → `host_vars/хост.yml`  
   Например: `all` < `webservers` < `web1`.

2. **Инвентарь vs Плейбук**:
   - Переменные из `group_vars` и `host_vars` **инвентаря** имеют приоритет над аналогичными файлами в каталоге **плейбука**.  
   Пример: `inventory/group_vars/webservers.yml` переопределит `playbook/group_vars/webservers.yml`.

3. **Host Variables > Group Variables**:
   - Для одного хоста переменные из `host_vars` всегда переопределяют переменные из `group_vars` (даже если группа входит в `all`).

4. **Специфика групп**:
   - Если хост принадлежит нескольким группам, переменные из группы с более высоким приоритетом (последняя группа в списке `ansible_group_priority`) будут доминировать.

---

<a id="как-проверить"></a>
#### Как проверить приоритеты?
- Используйте команду `ansible-inventory --list` для просмотра итоговых переменных хоста.  
- Добавьте `debug: var=nginx_port` в плейбук, чтобы увидеть актуальное значение.

Ссылка на документацию: [Variable Precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence).

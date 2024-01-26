# Ansible Role: Vector

[Ссылка на Gitlab tag v1.2 - основной playbook]()

## Description

Роль устанавливает Vector на группу хостов, определённых в вашем inventory.

## Requirements

- **Ansible 2.9+**
- **инвентори с ВМ с ОС семейства ubuntu / centos с предустановленным python3**
- **успешное подключение к хостам по ssh**
- **запущенный clickhouse с ip / портом в доступе**

## Role Variables

Переменные для пользователей хранятся в [defaults/main.yml](defaults/main.yml) файле.
Переменные по умолчанию не предполагаемые к изменению хранятся в [vars/main.yml](vars/main.yml) файле.

Пользовательские переменные:
| Name | Default Value | Description |
| -------------------------- | ------------- | ------------------------------------------------------------------------ |
| `vector_version` | "0.34.1" | Версия Кликхауса. Также допустим `latest` для установки последней. |
| `vector_config_dir` | "/home/ubuntu/vector_config" | Используется для указания директории с файлами конфигов на целевой машине |
| `vector_config` | "/home/ubuntu/vector_config/vector.yml.j2" | Используется для указания файла с конфигом |

## Dependencies

Роль не зависит ни от каких внешних ролей.
Необходимо указать пользовательские переменные, и роль можно подключать в работу вашего плейбука:

`requirements.yml:`

```yaml
---
- src:
  scm: git
  version: "latest"
  name: vector
```

Всё что ниже указывается в **вашем плейбуке, куда подключаете роль**.

Для успешной установки и запуска вектора на настраиваемой машине, необходимо указать файлы для сбора \ отправки в `sources`:

```
vector_config:
  sources:
    sample_file:
      type: file
      read_from: beginning
      ignore_older_secs: 600
      include:
        - /home/centos/logs/*.txt ## путь к файлу, содержащий входящие в вектор логи
```

Также указать ip с машиной `clickhouse` указать в конфиге вектора в файле с вызовом групповых переменных `group_vars/vector/***.yml` :

```
sinks:
    to_clickhouse:
      type: clickhouse
      inputs:
        - sample_file
      endpoint: http://<ip машины с clickhouse>:8123
      database: <имя БД>
      table: <имя таблицы>
      auth:
        password: <пароль>
        user: <пользователь>
        strategy: basic
```

## Example

Пример конфигурации:

После скачки / распаковки архива необходимо убедиться в наличии директории под конфиги. Ниже пример тасков инсталяции:

```yaml
---
- name: Download & unarchive ## get_url пропущено, так как его функционал встроен в unarchive с 2.0 версии
  become: true
  ansible.builtin.unarchive:
    src: https://packages.timber.io/vector/{{ vector_version }}/vector-{{ vector_version }}-x86_64-apple-darwin.tar.gz
    dest: /usr/local/bin
    remote_src: true
  notify: Start vector service

- name: Configure Vector | ensure what directory exists
  become: true
  ansible.builtin.file:
    path: "{{ vector_config_dir }}"
    state: directory
    mode: "0755"
  tags: [config]
- name: Configure Vector | Template config
  become: true
  ansible.builtin.template:
    src: vector.yml.j2
    mode: "0755"
    dest: "{{ vector_config_dir }}/vector.yml"
  tags: [config]
- name: Configure Vector Service | Template systemd unit
```

Пример таски, отвечающей за выбор пакетного менеджера в зависимости от целевой ОС:

```yaml
- name: Package manager choosing
  ansible.builtin.include_tasks:
    file: "{{ lookup('first_found', params) }}"
    apply:
      tags: [install]
  vars:
    params:
      files:
        - "install/{{ ansible_pkg_mgr }}.yml"
  tags: [install]
```

## Vector Params

Для успешного запуска проверьте файл tasks/main.yml
Выставьте нужную таску для установки на Вашу ОС.

Конфигурация для вектора заливается из файла `/templates/vector.yml.j2`
Конфигурация для службы вектора вектора заливается из файла `/templates/vector.service.j2`

Чтобы Вектор перезапускался автоматически после изменения его конфига, выставьте параметр в конфигурации службы:

```
RESTART=always
```

### Example Handlers

Запуск Вектора:

```
- name: Start vector service
  become: true
  ansible.builtin.service:
    name: vector
    state: started

```

Перезапуск Вектора:

```
- name: Restart vector service
  become: true
  ansible.builtin.service:
    name: vector
    state: started
```

## Tags

Так как роль только устанавливает ПО, то почти везде есть тег install.
Все теги перечислены ниже:

| Tag     | Action                                        |
| ------- | --------------------------------------------- |
| install | таски по установке пакетов                    |
| always  | таски, выполняемые всегда                     |
| config  | таски, относящиеся к настройке учётных данных |

## Troubleshooting

При обнаружении уязвимостей в ПО Вектора посетите, пожалуйста [vulnerability reporting page](https://github.com/vectordotdev/vector/blob/master/SECURITY.md#vulnerability-reporting). Просьба **не** создавать новый инцедент на Github.

Если нашли баг в работе Вектора - то посетите [project website](https://vector.dev/).

## License

[MIT License](https://opensource.org/licenses/MIT)

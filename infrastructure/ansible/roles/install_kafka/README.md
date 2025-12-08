# Ansible Role: Kafka Installer

Эта роль устанавливает и настраивает Apache Kafka в режиме KRaft (без Zookeeper). Роль также может опционально установить [Kafka Exporter](https://github.com/danielqsj/kafka_exporter) для мониторинга.

## Требования

Роль протестирована на следующих операционных системах:
*   Debian-based (Debian, Ubuntu)
*   RedHat-based (CentOS, Fedora, Rocky Linux)

Требуется `become: true` для выполнения задач с повышенными привилегиями.

## Структура роли

```
install_kafka/
├── README.md
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   ├── cleanup.yml
│   ├── exporter.yml
│   └── main.yml
├── templates/
│   ├── kafka-exporter.service.j2
│   ├── kafka.service.j2
│   └── server.properties.j2
└── vars/
    └── main.yml
```

## Логика работы (Execution Flow)

При запуске роль выполняет следующие шаги:

1.  **Подготовка**:
    *   Обновляет кеш пакетов (для Debian).
    *   Устанавливает Java и `acl` (если `kafka_install_dependencies: true`).
    *   Создает системного пользователя и группу `kafka`.

2.  **Загрузка и установка Kafka**:
    *   Автоматически определяет и загружает sha512 чек-сумму для архива Kafka.
    *   Загружает архив Kafka с официального сайта.
    *   Распаковывает архив в `{{ kafka_download_directory }}`.
    *   Создает символическую ссылку `{{ kafka_home }}` на директорию с установленной версией.

3.  **Конфигурация**:
    *   Создает директории для данных и логов.
    *   Создает конфигурационные файлы для `KAFKA_HEAP_OPTS` и `KAFKA_OPTS`.
    *   Генерирует `server.properties` на основе переменных роли.
    *   Генерирует `systemd` сервис для Kafka.

4.  **Настройка кластера (KRaft)**:
    *   Генерирует уникальный ID кластера (`cluster.id`) на первом контроллере.
    *   Форматирует хранилище (`kafka-storage.sh format`).
    *   Запускает и включает сервис `kafka`.

5.  **Создание топиков**:
    *   Ожидает запуска брокеров и контроллеров.
    *   Создает топики, определенные в переменной `kafka_topics`, если их еще нет.

6.  **Установка Exporter (опционально)**:
    *   Если `kafka_exporter_install: true`, скачивает и настраивает Kafka Exporter.
    *   Создает и запускает `systemd` сервис для `kafka-exporter`.

## Переменные роли

Все настраиваемые переменные находятся в `defaults/main.yml`.

| Переменная                                   | Описание                                                                                             | Значение по умолчанию                                                                                              |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `kafka_user`                                 | Пользователь, от имени которого будет работать Kafka.                                                | `kafka`                                                                                                            |
| `kafka_user_group`                           | Группа пользователя Kafka.                                                                           | `kafka`                                                                                                            |
| `kafka_home`                                 | Домашняя директория Kafka (символическая ссылка).                                                    | `/opt/kafka`                                                                                                       |
| `kafka_version`                              | Версия Kafka для установки.                                                                          | `4.0.0`                                                                                                            |
| `kafka_scala_version`                        | Версия Scala.                                                                                        | `2.13`                                                                                                             |
| `kafka_java_version`                         | Версия Java (OpenJDK) для установки.                                                                 | `17`                                                                                                               |
| `kafka_install_dependencies`                 | Устанавливать ли зависимости (Java, acl).                                                            | `false`                                                                                                            |
| `kafka_checksum`                             | Чек-сумма для архива. Если не указана, роль попытается загрузить ее с сайта Apache.                   | `не задана`                                                                                                        |
| `kafka_heap_size`                            | Размер heap-памяти для JVM Kafka.                                                                    | `1G`                                                                                                               |
| `kafka_opts`                                 | Дополнительные опции для JVM Kafka (в виде списка или строки).                                       | `[]`                                                                                                               |
| `kafka_additional_config`                    | Дополнительные параметры для `server.properties` в формате `ключ: значение`.                         | `{ auto.create.topics.enable: "true" }`                                                                            |
| `kafka_cluster_id`                           | ID кластера. Если не задан, генерируется автоматически.                                              | `не задан`                                                                                                         |
| `kafka_topics`                               | Список топиков для создания.                                                                         | `[]`                                                                                                               |
| `kafka_node_roles`                           | Роли узла в KRaft-кластере (`broker`, `controller`).                                                 | `['broker', 'controller']`                                                                                         |
| `kafka_log_dirs`                             | Директории для хранения данных (логов) Kafka.                                                       | `['/data/kafka']`                                                                                                  |
| `kafka_log_retention_hours`                  | Время хранения сообщений в топиках (в часах).                                                        | `24`                                                                                                               |
| `kafka_log_retention_bytes`                  | Максимальный размер логов для хранения. `-1` отключает ограничение.                                  | `-1`                                                                                                               |
| `kafka_offsets_topic_replication_factor`     | Фактор репликации для топика `__consumer_offsets`.                                                   | `1`                                                                                                                |
| `kafka_transaction_state_log_replication_factor` | Фактор репликации для топика `__transaction_state`.                                              | `1`                                                                                                                |
| `kafka_transaction_state_log_min_isr`        | Минимальное количество синхронизированных реплик для служебных топиков.                              | `1`                                                                                                                |
| `kafka_exporter_install`                     | Устанавливать ли Kafka Exporter.                                                                     | `true`                                                                                                             |
| `kafka_exporter_version`                     | Версия Kafka Exporter.                                                                               | `1.9.0`                                                                                                            |
| `kafka_cpu_quota`                            | Ограничение CPU для systemd-сервиса Kafka.                                                           | `200%`                                                                                                             |
| `kafka_memory_limit`                         | Ограничение памяти для systemd-сервиса Kafka.                                                        | `2G`                                                                                                               |

## Примеры использования

### Плейбук

```yaml
---
- name: Install and configure Kafka
  hosts: kafka_nodes
  become: true

  roles:
    - install_kafka
```

### Инвентарь для одного узла

```ini
[kafka_nodes]
kafka-1 ansible_host=192.168.1.10 kafka_node_id=1
```

### Инвентарь для отказоустойчивого кластера

Для создания отказоустойчивого кластера необходимо иметь как минимум 3 узла с ролью `controller`. В инвентаре нужно задать уникальный `kafka_node_id` для каждого узла и настроить параметры репликации.

**Рекомендация:** Для продуктивных сред используйте фактор репликации `3` и `min.insync.replicas` равный `2`.

Пример инвентаря `inventory.yml`:

```yaml
all:
  children:
    kafka_nodes:
      vars:
        # --- Общие переменные для всех узлов Kafka ---
        ansible_user: kafka_admin
        ansible_ssh_private_key_file: ~/.ssh/kafka_key
        
        # --- Настройки для отказоустойчивости ---
        # Фактор репликации для служебных топиков должен быть > 1
        kafka_offsets_topic_replication_factor: 3
        kafka_transaction_state_log_replication_factor: 3
        kafka_transaction_state_log_min_isr: 2 # Мин. кол-во реплик для подтверждения записи

      hosts:
        kafka-node-1:
          ansible_host: 10.0.0.1
          kafka_node_id: 1
          kafka_node_roles: ['broker', 'controller']
        kafka-node-2:
          ansible_host: 10.0.0.2
          kafka_node_id: 2
          kafka_node_roles: ['broker', 'controller']
        kafka-node-3:
          ansible_host: 10.0.0.3
          kafka_node_id: 3
          kafka_node_roles: ['broker', 'controller']

```

## Удаление Kafka

Для удаления Kafka и связанных с ним компонентов можно использовать плейбук `playbooks/uninstall_kafka.yml`.

```yaml
---
- name: Uninstall Kafka
  hosts: kafka_nodes
  become: true

  vars:
    # Установить в 'true' для удаления Java.
    # ВНИМАНИЕ: это может затронуть другие приложения!
    remove_java: false
    
    # Установить в 'true' для удаления пользователя/группы kafka
    remove_user_and_group: true
    
    # Установить в 'true' для удаления Kafka Exporter
    remove_exporter: true

  tasks:
    - name: Include tasks from role install_kafka.cleanup
      ansible.builtin.include_role:
        name: install_kafka
        tasks_from: cleanup.yml
```
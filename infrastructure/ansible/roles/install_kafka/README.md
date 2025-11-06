# Ansible Role: Kafka

This role installs and configures Apache Kafka in KRaft mode.

## Cleanup

This role includes a task to completely remove Kafka and its components from a system. This is useful for cleaning up environments or for re-installation.

To run the cleanup, you can create a playbook that includes the `cleanup.yml` task file from this role.

**Example Playbook:** `uninstall_kafka.yml`
```yaml
- name: Uninstall Kafka
  hosts: kafka_nodes
  become: true
  vars:
    # Set to true to remove Java as well
    remove_java: false
    # Set to false to keep the kafka user and group
    remove_user_and_group: true
    # Set to false to keep the kafka exporter
    remove_exporter: true
  tasks:
    - name: Include cleanup tasks from kafka role
      ansible.builtin.include_role:
        name: kafka
        tasks_from: cleanup.yml
```

### Cleanup Steps

The `cleanup.yml` task performs the following actions:

1.  **Stops and disables services**:
    *   Stops and disables the `kafka` service.
    *   Stops and disables the `kafka-exporter` service.

2.  **Removes Kafka Exporter** (controlled by `remove_exporter` variable, default: `true`):
    *   Removes the `kafka-exporter.service` systemd file.
    *   Removes the `/usr/local/bin/kafka_exporter` binary.
    *   Removes the `kafka-exporter` user and group.

3.  **Removes Kafka**:
    *   Removes the `kafka.service` systemd file.
    *   Removes the `/etc/sysconfig/kafka` configuration file.
    *   Removes Kafka's data and log directories (e.g., `/data/kafka`, `/var/log/kafka`).
    *   Removes the `/opt/kafka` symlink and the unpacked Kafka directory.
    *   Removes the downloaded Kafka archive.

4.  **Removes user and group** (controlled by `remove_user_and_group` variable, default: `true`):
    *   Removes the `kafka` user and group.

5.  **Removes dependencies** (controlled by `remove_java` variable, default: `false`):
    *   Uninstalls Java.

6.  **Performs system cleanup**:
    *   Reloads the systemd daemon.
    *   Removes temporary files.

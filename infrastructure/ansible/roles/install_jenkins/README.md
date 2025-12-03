#

### Пример манифеста для создания PVC:

Создайте файл `jenkins-pvc.yaml`:

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-agent-cache-pvc  # Имя, которое используется в JCasC
spec:
  accessModes:
    - ReadWriteOnce  # Важно! Об этом ниже.
  resources:
    requests:
      storage: 10Gi # Размер диска
  # storageClassName: "your-storage-class" # Раскомментируйте, если у вас есть свой StorageClass
```

И примените его командой: `kubectl apply -f jenkins-pvc.yaml -n jenkins`

- **ReadWriteOnce (RWO)**: Большинство стандартных хранилищ (в облаках и лока льно) поддерживают только этот режим. Он означает, что один PVC может быть подключен только к одному поду одновременно. Если вы запустите 10 пайплайнов с k8 s-maven, только один из них начнет выполняться. Остальные 9 будут ждать в очереди, пока первый не освободит PVC. Кэш будет использовать ся всеми по очереди, но параллельно они работать не смогут.
- **ReadWriteMany (RWX)**: Чтобы несколько подов одновременно использовали один PV C, вам нужен StorageClass, поддерживающий этот режим (например, на базе NFS, EF S, CephFS).


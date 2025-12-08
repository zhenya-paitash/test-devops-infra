# Ansible Role: install_jenkins

Данная роль обеспечивает комплексную, автоматизированную установку контроллера Jenkins, настроенного для использования Kubernetes для запуска динамических, контейнеризированных агентов сборки. Роль следует современным практикам, используя Configuration as Code (JCasC) и создавая кастомный Docker-образ со всеми необходимыми плагинами.

## Логика работы

Роль логически разделена на несколько этапов, на которые можно нацеливаться с помощью тегов Ansible:

1.  **Предварительная проверка (тег: `fail`):**
    *   Проверяет, что пароль администратора по умолчанию был изменен. Плейбук завершится с ошибкой, если используется пароль по умолчанию (`DEFAULT`).

2.  **Установка зависимостей (тег: `docker`):**
    *   Включает роль `install_docker` для установки Docker и Docker Compose на целевом узле.

3.  **Настройка хоста (тег: `config`):**
    *   Создает домашнюю директорию Jenkins (`/var/jenkins_home` по умолчанию) на хост-машине с правильными правами доступа.

4.  **Безопасный доступ к Kubernetes (тег: `k8s`):**
    *   Безопасно получает файл `admin.conf` с указанного мастер-узла Kubernetes.
    *   Извлекает URL API-сервера кластера и CA-сертификат. Эти данные используются для безопасной настройки плагина Jenkins Kubernetes, избегая небезопасной опции `skipTlsVerify: true`.

5.  **Настройка RBAC в Kubernetes (тег: `k8s`):**
    *   Создает все необходимые ресурсы Kubernetes для агентов Jenkins в выделенном неймспейсе (`jenkins` по умолчанию). Это включает:
        *   `ServiceAccount` для агентов.
        *   `Role` с правами на управление подами (создание, удаление, получение, листинг и т.д.) **только в этом неймспейсе**.
        *   `RoleBinding` для привязки `Role` к `ServiceAccount`.
    *   Также безопасно извлекается токен этого `ServiceAccount` для использования плагином Jenkins Kubernetes.

6.  **Сборка кастомного образа (тег: `build`):**
    *   Собирается кастомный Docker-образ для контроллера Jenkins.
    *   Процесс сборки использует шаблоны `Dockerfile.j2` и `plugins.txt.j2`.
    *   `plugins.txt` формируется из переменных `jenkins_plugins` и `jenkins_plugins_extra`, гарантируя, что все необходимые плагины "запечены" в образ. Это намного надежнее и быстрее, чем установка плагинов при первом запуске.

7.  **Подготовка конфигурационных файлов (тег: `config`):**
    *   На основе шаблонов создаются основной файл JCasC (`jenkins.yaml.j2`) и файл `docker-compose.yml.j2` в домашней директории Jenkins.

8.  **Запуск Jenkins (тег: `run`):**
    *   Запускает контроллер Jenkins с помощью `docker-compose`, который использует конфигурацию, подготовленную на предыдущем шаге.

## Безопасность соединения с Kubernetes (TLS и RBAC)

Эта роль настраивает безопасное соединение между Jenkins и Kubernetes API-сервером.

*   **TLS:** Вместо отключения проверки (`skipTlsVerify: true`), роль получает официальный CA-сертификат кластера. Jenkins использует этот сертификат для проверки подлинности API-сервера, что защищает от атак "человек посередине". Сам Jenkins при этом может оставаться доступным по HTTP, так как TLS настраивается для коммуникации "Jenkins -> Kubernetes", а не "Пользователь -> Jenkins".
*   **RBAC:** Роль создает `ServiceAccount` с правами, ограниченными **только неймспейсом `jenkins`**. Сертификат из `admin.conf` используется Jenkins только для аутентификации на API-сервере (чтобы доказать, что это легитимный клиент). После успешной аутентификации все дальнейшие действия (например, создание пода-агента) авторизуются через RBAC, и Kubernetes проверяет, что у `ServiceAccount` есть права на это действие и только в разрешенном неймспейсе.

## Переменные

Все переменные определены и задокументированы в `defaults/main.yml`. Для кастомизации под конкретное окружение (dev, prod) рекомендуется переопределять их в файлах `group_vars` вашего inventory.

### Основные группы переменных:
*   **Настройки безопасности:** `jenkins_admin_password` и др.
*   **Основные параметры Jenkins:** `jenkins_image`, `jenkins_url` и др.
*   **Управление ресурсами:** Лимиты и запросы CPU/Memory для контейнера Jenkins.
*   **Настройки кэширования:** Параметры для `PersistentVolumeClaim`.
*   **Плагины:** Списки `jenkins_plugins` и `jenkins_plugins_extra`.
*   **Настройки агентов Kubernetes:** Имена, лейблы и образы для шаблонов подов.

## Зависимости

-   Роль зависит от `install_docker`.
-   Требуется сетевой доступ от хоста Ansible к целевому узлу Jenkins.
-   Требуется сетевой доступ от хоста Ansible к мастер-узлу Kubernetes (для задачи `setup_secure_access`).
-   Если автоматическое создание PV/PVC (см. ниже) не используется, в кластере Kubernetes должен быть настроен `StorageClass` для динамического выделения `PersistentVolume`.

## Automated Cache Provisioning (PV/PVC)

Для упрощения развертывания в средах разработки, роль может автоматически создавать `PersistentVolume` (используя `hostPath`) и соответствующий ему `PersistentVolumeClaim`.

**ВНИМАНИЕ:** Этот метод предназначен **только для разработки на однонодовых кластерах**. В production-окружении используйте `StorageClass` или создавайте PV/PVC вручную, как описано в следующем разделе.

Для активации этой функции, установите следующие переменные в ваших `group_vars`:

```yaml
# Файл: group_vars/all/main.yml или group_vars/jenkins_nodes/main.yml
jenkins_k8s_create_pv_pvc: true
# Опционально, можно переопределить путь на хосте Kubernetes-ноды
# jenkins_k8s_pv_host_path: "/mnt/jenkins-cache"
```

Роль выполнит следующие действия:
1.  На целевом хосте (из группы `jenkins_nodes`) создаст директорию, указанную в `jenkins_k8s_pv_host_path`.
2.  Назначит этой директории права владения `1000:1000`, что необходимо для корректной работы агентов Jenkins в контейнерах.
3.  Через `kubectl` применит манифесты для создания `PersistentVolume` и `PersistentVolumeClaim` в кластере.

## Manual Cache Provisioning (for Production/Multi-Node)

Если автоматическое создание `hostPath` вам не подходит (например, в многонодовом или production-кластере), вы должны создать `PersistentVolumeClaim` (PVC) самостоятельно. JCasC настроен на его использование, но не на его создание.

Этот гайд показывает, как создать PV и PVC для тестового окружения, используя `hostPath`. В production используйте подходящий для вашего окружения `StorageClass` и создавайте PVC на его основе.

### Шаг 1: Создание директории на узле Kubernetes

Подключитесь по SSH к вашему Kubernetes-узлу (worker node) и выполните команды для создания директории, которая будет служить "физическим диском":

```bash
sudo mkdir -p /mnt/jenkins-cache
sudo chown 1000:1000 /mnt/jenkins-cache
```

### Шаг 2: Создание PersistentVolume (PV)

Создайте на вашей локальной машине файл `jenkins-pv.yaml`:

```yaml
# jenkins-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-agent-cache-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/jenkins-cache"
  persistentVolumeReclaimPolicy: Retain
```

### Шаг 3: Создание PersistentVolumeClaim (PVC)

Создайте файл `jenkins-pvc.yaml`. Он "запросит" созданный выше PV.

```yaml
# jenkins-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-agent-cache-pvc
  namespace: jenkins
spec:
  storageClassName: "" # Важно: указываем пустой класс, чтобы соответствовать PV
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeName: jenkins-agent-cache-pv # Явно привязываемся к нашему PV
```

### Шаг 4: Применение манифестов

Выполните эти команды для создания объектов в кластере:

```bash
kubectl apply -f jenkins-pv.yaml
kubectl apply -f jenkins-pvc.yaml
```
После этого команда `kubectl get pv,pvc -n jenkins` должна показать статус `Bound`.

### Шаг 5: Проверка работы кэша с помощью Pipeline

Создайте в Jenkins новый Pipeline со следующим кодом, чтобы проверить, что данные сохраняются между запусками.

```groovy
pipeline {
    agent {
        kubernetes {
            label 'bun-app-pod'
            yaml '''
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 1000
  containers:
  - name: jnlp
    image: "jenkins/inbound-agent:jdk25"
    args: ["$(JENKINS_SECRET)", "$(JENKINS_NAME)"]
    volumeMounts:
    - name: "agent-cache"
      mountPath: "/cache"
  - name: bun
    image: "oven/bun:latest"
    command: ["cat"]
    tty: true
    volumeMounts:
    - name: "agent-cache"
      mountPath: "/cache"
  volumes:
  - name: "agent-cache"
    persistentVolumeClaim:
      claimName: "jenkins-agent-cache-pvc"
'''
        }
    }
    
    stages {
        stage('Initialize & Create Server') {
            steps {
                container('bun') {
                    sh 'bun init -y'
                    writeFile(
                        file: 'index.ts',
                        text: '''
const server = Bun.serve({
  port: 3000,
  fetch(request) {
    return new Response("Welcome to Bun!");
  },
});

console.log(`Listening on ${server.url}`);
'''
                    )
                }
            }
        }

        stage('Start Server') {
            steps {
                container('bun') {
                    echo "--- Starting Bun server in background ---"
                    // Запускаем сервер в фоне и сохраняем его PID в файл
                    sh 'bun run index.ts & echo $! > bun.pid'
                }
            }
        }
        
        stage('Test Server') {
            steps {
                container('bun') {
                    sh 'sleep 10'
                    echo "--- Checking server response ---"
                    // Делаем запрос и сохраняем лог в кеш на PVC
                    sh '''
                        set -e
                        TIMESTAMP=$(date +%Y-%m-%d_%H-%M-%S)
                        CACHE_DIR="/cache"
                        LOG_FILE="$CACHE_DIR/curl-$TIMESTAMP.log"

                        RESPONSE=$(bun -e 'const res = await fetch("http://localhost:3000"); console.log(await res.text())')
                        
                        echo "✅ Server responded: $RESPONSE"
                        echo "$RESPONSE" > $LOG_FILE
                        
                        # Проверяем, что ответ корректный
                        if ! echo "$RESPONSE" | grep -q "Welcome to Bun!"; then
                            echo "❌ ERROR: Unexpected server response!"
                            exit 1
                        fi
                        
                        echo "--- Log file created in PV cache: ---"
                        ls -l $LOG_FILE
                    '''
                }
            }
        }

        stage('Stop Server') {
            steps {
                container('bun') {
                    echo "--- Stopping server ---"
                    // Читаем PID из файла и корректно останавливаем процесс
                    sh 'kill $(cat bun.pid)'
                }
            }
        }
    }
}
```
*   **Первый запуск:** Создаст файл `test.txt` в кэше.
*   **Второй запуск:** Обнаружит этот файл, выведет его содержимое и обновит его, доказывая, что персистентность работает.

### Замечание о режимах доступа (Access Modes)

*   **`ReadWriteOnce` (RWO):** Используемый нами режим. Означает, что диск может быть подключен только **к одному поду одновременно**. Если 10 пайплайнов одновременно запросят этот агент, они будут выполняться **по очереди**, но все будут использовать один и тот же кэш.
*   **`ReadWriteMany` (RWX):** Для одновременного доступа нескольких подов к одному диску требуется СХД с поддержкой этого режима (например, NFS).

# Документация проекта Hyperdx

## Обзор проекта

Это Ansible-проект для развёртывания **Hyperdx** — платформы наблюдаемости — в кластере Kubernetes. Проект автоматизирует деплой полноценного стека приложений с MongoDB в качестве базы данных.

## Структура проекта

```
hyperdx/
├── deploy.yml              # Главный Ansible-плейбук
├── roles/
│   └── deploy/
│       └── tasks/
│           └── main.yml    # Основные задачи деплоя
├── vars/
│   └── prod.yml            # Переменные для production-окружения
├── templates/              # Манифесты Kubernetes (Jinja2-шаблоны)
│   ├── configmap/          # ConfigMap для конфигурации приложений
│   ├── cronjob/            # Запланированные задачи (бэкап/восстановление)
│   ├── deployment/         # Deployment для stateless-приложений
│   ├── disruptionbudget/   # PodDisruptionBudget для высокой доступности
│   ├── ingress/            # Ресурсы Ingress для внешнего доступа
│   ├── secrets/            # Секреты (чувствительные данные)
│   ├── service/            # Сервисы
│   └── statefulset/        # StatefulSet для stateful-приложений (MongoDB)
```

## Архитектура

### Разворачиваемые компоненты

1. **Приложение Hyperdx** (`hyperdx`)
   - Deployment с 2 репликами
   - Контейнер: `hyperdx/hyperdx:2.21.0`
   - Порт: 8080
   - Ресурсы: 1 CPU, 1.5Gi памяти
   - Доступ через Ingress: `hyperdx.firm.ru`

2. **MongoDB** (`mongo`)
   - StatefulSet с 3 репликами
   - Контейнер: `mongo:8.2`
   - Порт: 27017
   - Ресурсы: 1 CPU, 2Gi памяти
   - Хранилище: 20Gi на под (Longhorn)
   - Включает CronJobs для бэкапа и восстановления

3. **Сеть**
   - Ingress с TLS-терминацией
   - Сервис на порту 8098
   - Headless-сервис для MongoDB

### Высокая доступность

- PodAntiAffinity для подов Hyperdx (распределение по узлам)
- PodDisruptionBudget для Hyperdx и MongoDB
- 3 реплики MongoDB для отказоустойчивости

## Процесс деплоя

### Главный плейбук (`deploy.yml`)

```yaml
- name: Deploy application
  hosts: localhost
  gather_facts: no
  roles:
    - deploy
```

### Задачи (`roles/deploy/tasks/main.yml`)

Деплой выполняется в следующей последовательности:

1. **Подключение переменных** — загрузка переменных окружения из `vars/{env_code}.yml`

2. **Получение секретов из Vault** — извлечение чувствительных данных из HashiCorp Vault:
   - Чтение токена Vault из `/opt/{vault_agent_name_directory}/.token_unwrapped`
   - Запрос секретов по пути `vault_ansible_secret_path`
   - Извлечение kubeconfig для доступа к Kubernetes API

3. **Развёртывание ресурсов** (по порядку):
   - Secrets (манифесты Secret)
   - ConfigMap (конфигурация приложения)
   - Service (ClusterIP-сервисы)
   - Ingress (внешний доступ)
   - CronJobs (задачи бэкапа/восстановления)
   - DisruptionBudget (политики доступности)
   - StatefulSet (MongoDB)
   - Deployment (приложение Hyperdx)

### Файлы переменных (`vars/prod.yml`)

Основные переменные конфигурации:

| Переменная | Описание |
|------------|----------|
| `env_code` | Идентификатор окружения (например, "prod") |
| `namespace` | Пространство имён Kubernetes |
| `balancer_url_hyperdx` | Внешний URL для Hyperdx |
| `ingress_hostname` | Имя хоста для Ingress |
| `vault_agent_api_url` | API-адрес агента Vault |
| `vault_agent_name_directory` | Имя директории для токена Vault |
| `vault_ansible_secret_path` | Путь к секретам в Vault |

## Шаблоны Kubernetes

### Secrets
- `hyperdx.yml.j2` — Секреты приложения (MONGO_URI, JWT_SECRET и т.д.)
- `mongodb-root-credentials.yml.j2` — Учётные данные администратора MongoDB
- `mongodb-keyfile.yml.j2` — Keyfile для аутентификации MongoDB
- `mongodb-backup-minio-secret.yml.j2` — Учётные данные для хранилища бэкапов

### ConfigMaps
- `hyperdx.yml.j2` — Конфигурация приложения Hyperdx
- `mongodb.yml.j2` — Скрипты и конфигурация MongoDB

### CronJobs
- `mongodb-backup.yml.j2` — Запланированные бэкапы MongoDB
- `mongodb-restore.yml.j2` — Запланированное восстановление MongoDB

### Services
- `hyperdx.yml.j2` — Сервис Hyperdx (ClusterIP)
- `mongodb-headless.yml.j2` — Headless-сервис для MongoDB

### Deployments/StatefulSets
- `deployment/hyperdx.yml.j2` — Deployment для Hyperdx
- `statefulset/mongodb.yml.j2` — StatefulSet для MongoDB

### Прочие
- `ingress/hyperdx.yml.j2` — Ingress с TLS
- `disruptionbudget/hyperdx.yml.j2` — PDB для Hyperdx
- `disruptionbudget/mongodb.yml.j2` — PDB для MongoDB

## Использование

### Запуск плейбука

```bash
# Деплой в production (env_code=prod)
. /var/ansible/venv/bin/activate && ansible-playbook ./deploy.yml -e "env_code=prod namespace=telr01-01-common"
```

### Требования

1. **Ansible** с коллекциями `kubernetes.core` и `community.hashi_vault`
2. **Vault Agent** запущен и доступен
3. **Kubernetes-кластер** с настроенным доступом
4. **Vault-токен** доступен по ожидаемому пути

## Бэкап и восстановление MongoDB

Система бэкапа и восстановления MongoDB реализована через два CronJob-ресурса: `mongo-minio-backup` и `mongo-minio-restore`.

### CronJob бэкапа (`mongodb-backup.yml.j2`)

**Расписание:** каждые 6 часов (`0 */6 * * *`)

**Процесс бэкапа:**

1. **Init-контейнер: скачивание MinIO Client**
   - Образ: `curlimages/curl:8.7.1`
   - Скачивает бинарник `mc` (MinIO Client) в emptyDir-том `/tools`
   - Ресурсы: 200m CPU, 128Mi памяти

2. **Основной контейнер: mongodump**
   - Образ: `mongo:8.2`
   - Создаёт временную директорию `$TMPDIR`
   - Формирует имя архива: `hyperdx-YYYYMMDD-HHMMSS.archive.gz`
   - Выполняет `mongodump` с параметрами:
     - `--uri` — подключение к MongoDB через секрет `hyperdx-secrets` (ключ `MONGO_URI`)
     - `--readPreference=secondary` — читает с реплики для снижения нагрузки на primary
     - `--archive` — путь к архиву
     - `--gzip` — сжатие архива
   - Загружает архив в MinIO через `mc cp`
   - Удаляет временные файлы

**Куда кладут бэкап:**
- MinIO Endpoint: `https://edox-minio-api.firm.ru`
- Bucket: `hyperdx/mongo-backups`
- Имя файла: `hyperdx-YYYYMMDD-HHMMSS.archive.gz`

**Переменные окружения (из Secret `mongo-backup-minio-credentials`):**
| Переменная | Источник | Описание |
|------------|----------|----------|
| `MINIO_ENDPOINT` | Secret | URL MinIO API |
| `MINIO_ACCESS_KEY` | Vault → Secret | Access key для MinIO |
| `MINIO_SECRET_KEY` | Vault → Secret | Secret key для MinIO |
| `MINIO_BUCKET` | Secret | Путь в MinIO |

**Политика запуска:**
- `concurrencyPolicy: Forbid` — не запускать параллельно
- `successfulJobsHistoryLimit: 1` — хранить 1 успешный Job
- `failedJobsHistoryLimit: 1` — хранить 1 неудачный Job
- `backoffLimit: 3` — 3 попытки при неудаче
- `ttlSecondsAfterFinished: 11000` — Job живёт ~3 часа после завершения

---

### CronJob восстановления (`mongodb-restore.yml.j2`)

**Расписание:** ежедневно в полночь (`0 0 * * *`)

**Важно:** CronJob по умолчанию приостановлен (`spec.suspend: true`).

**Процесс восстановления:**

1. **Init-контейнер: скачивание MinIO Client**
   - Аналогично бэкапу

2. **Основной контейнер: mongorestore**
   - Настраивает alias `backup` для MinIO
   - Скачивает архив из MinIO: `mc cp backup/{MINIO_BUCKET}/{BACKUP_FILE}` → `/work/backup.archive.gz`
   - Переменная `BACKUP_FILE` по умолчанию: `hyperdx-*.archive.gz` (не скачивает ни чего), сделано для защиты от дурака, архив в CronJob надо передать руками
   - Выполняет `mongorestore` с параметрами:
     - `--uri` — подключение к MongoDB
     - `--archive` — путь к архиву
     - `--gzip` — архив сжат
     - `--drop` — **удаляет все коллекции перед восстановлением** (важно!)
   - Удаляет временные файлы

**Откуда забирают бэкап:**
- MinIO Endpoint: `https://edox-minio-api.firm.ru`
- Bucket: `hyperdx/mongo-backups`
- Файл по умолчанию: `hyperdx-*.archive.gz` (можно изменить через переменную `BACKUP_FILE`)

**Переменные окружения:**
| Переменная | Источник | Описание |
|------------|----------|----------|
| `BACKUP_FILE` | Env (по умолчанию) | Имя файла для восстановления |
| `MONGO_URI` | Secret `hyperdx-secrets` | Строка подключения к MongoDB |
| MINIO_* | Secret `mongo-backup-minio-credentials` | Creds для MinIO |

**Политика запуска:**
- `concurrencyPolicy` не задан (по умолчанию Allow)
- `backoffLimit: 0` — не ретраить при неудаче
- `restartPolicy: Never` — под не перезапускается

---

### Конфигурация подключения к MinIO

Настройки хранятся в Secret `mongo-backup-minio-credentials`:

```yaml
stringData:
  MINIO_ENDPOINT: "https://edox-minio-api.firm.ru"
  MINIO_ACCESS_KEY: "{{ vault_secrets.mongo_db_minio_access_key }}"
  MINIO_SECRET_KEY: "{{ vault_secrets.mongo_db_minio_secret_key }}"
  MINIO_BUCKET: "hyperdx/mongo-backups"
```

Доступные ключи (`vault_secrets.*`) берутся из Vault по пути, указанному в `vault_ansible_secret_path`.

---

### Схема взаимодействия

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                    External                                                      │
│                                                                                                                  │
│     ┌─────────────┐                    ┌─────────────────────────┐                    ┌──────────────────┐    │
│     │   Users     │                    │       Vault             │                    │   MinIO S3       │    │
│     │             │                    │                         │                    │                  │    │
│     │  Browser    │                    │  hashicorpvault.internal│                    │ edox-minio-api   │    │
│     │             │                    │                         │                    │   .firm.ru      │    │
│     │  https://   │                    │  ┌───────────────────┐  │                    │                  │    │
│     │ hyperdx.fin │                    │  │ secret path:      │  │                    │  bucket:         │    │
│     │ am.ru       │                    │  │ /project/data/    │  │                    │  hyperdx/        │    │
│     └──────┬──────┘                    │  │ hyperdx/prod/     │  │                    │  mongo-backups/  │    │
│            │                           │  │ secrets           │  │                    │                  │    │
│            │                           │  │                   │  │                    │  ┌────────────┐  │    │
│            │                           │  │ - MONGO_URI       │  │                    │  │ .archive.gz│  │    │
│            │                           │  │ - JWT_SECRET      │◄─┼──token+secrets───►│  │   (dumps)  │  │    │
│            │                           │  │ - kubeconfig      │  │                    │  └────────────┘  │    │
│            │                           │  │ - minio creds     │  │                    │                  │    │
│            │                           │  └───────────────────┘  │                    └────────┬─────────┘    │
│            │                           │                         │                             │              │
│            └───────────────────────────┼─────────────────────────┘                             │              │
│                                        │                                                       │              │
└────────────────────────────────────────┼───────────────────────────────────────────────────────┼──────────────┘
                                         │                                                       │
                                         │         Ansible Controller                            │
                                         │                                                       │
│  ┌────────────────────────────────────┼───────────────────────────────────────────────────────┼──────────────┐
│  │                                    │                                                       │              │
│  │                          ┌─────────▼──────────────────────────────────────────────────────▼─────────┐   │
│  │                          │                    Ansible Playbook: deploy.yml                     │   │
│  │                          │                                                                       │   │
│  │                          │   1. include_vars: vars/prod.yml                                   │   │
│  │                          │   2. read_file: /opt/vault_proxy_prod_hyperdx/.token_unwrapped     │   │
│  │                          │   3. lookup hashi_vault: secrets + kubeconfig                      │   │
│  │                          │   4. k8s: apply templates (Secrets, ConfigMaps, Services, etc)    │   │
│  │                          └───────────────────────────────────────────────────────────────────────┘   │
│  │                                           │                                                         │
│  └───────────────────────────────────────────┼─────────────────────────────────────────────────────────┘
                                               │ k8s API
                                               │ (kubeconfig from Vault)
┌──────────────────────────────────────────────┼──────────────────────────────────────────────────────────────┐
│                                              ▼                                                                  │
│                                       KUBERNETES CLUSTER                                                         │
│                                                                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                              NAMESPACE: telr01-01-common                                           │  │
│  │  ┌────────────────────────────────────────────────────────────────────────────────────────────────────┐  │  │
│  │  │                                             INGRESS                                                  │  │  │
│  │  │  ┌──────────────────────────────────────────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │ hyperdx.firm.ru ──────────────────────────────────────────────────────────────────────────►│  │  │  │
│  │  │  │ (TLS termination)                                                                               │  │  │  │
│  │  │  │ nginx.ingress.kubernetes.io/proxy-redirect-to: https://hyperdx.firm.ru                       │  │  │  │
│  │  │  └──────────────────────────────────────────────────────────────────────────────────────────────┘  │  │  │
│  │  └────────────────────────────────────────────────────────────────────────────────────────────────────┘  │  │
│  │                                                 │                                                            │  │
│  │  ┌─────────────────────────────────────────────▼────────────────────────────────────────────────────────┐   │  │
│  │  │                                                SERVICE                                               │   │  │
│  │  │  ┌────────────────────────────────┐    ┌─────────────────────────────────┐                         │   │  │
│  │  │  │ hyperdx (ClusterIP:8098)       │    │ mongo-headless (Headless)       │                         │   │  │
│  │  │  │                                │    │                                 │                         │   │  │
│  │  │  │   hyperdx-xxx-xxxx (pod)       │    │   mongo-0  ──► :27017           │                         │   │  │
│  │  │  │   hyperdx-xxx-xxxx (pod)       │    │   mongo-1  ──► :27017           │                         │   │  │
│  │  │  │                                │    │   mongo-2  ──► :27017           │                         │   │  │
│  │  │  └────────────────────────────────┘    └─────────────────────────────────┘                         │   │  │
│  │  └────────────────────────────────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                │                                                            │  │
│  │  ┌─────────────────────────────────────────────▼────────────────────────────────────────────────────────┐   │  │
│  │  │                                           DEPLOYMENT                                                  │   │  │
│  │  │  ┌────────────────────────────────────────────────────────────────────────────────────────────────┐ │   │  │
│  │  │  │  hyperdx (replicas: 2)                                                                             │ │   │  │
│  │  │  │                                                                                                    │ │   │  │
│  │  │  │  Image: hyperdx/hyperdx:2.21.0                                                                    │ │   │  │
│  │  │  │  Port: 8080                                                                                       │ │   │  │
│  │  │  │  Resources: 1 CPU, 1.5Gi                                                                          │ │   │  │
│  │  │  │                                                                                                    │ │   │  │
│  │  │  │  Env:                                                                                             │ │   │  │
│  │  │  │    - MONGO_URI (from secret)                                                                      │ │   │  │
│  │  │  │    - JWT_SECRET (from secret)                                                                     │ │   │  │
│  │  │  │    - NEXTAUTH_SECRET (from secret)                                                                │ │   │  │
│  │  │  │    - AUTH_SECRET (from secret)                                                                    │ │   │  │
│  │  │  │    - ENCRYPTION_KEY (from secret)                                                                 │ │   │  │
│  │  │  │                                                                                                    │ │   │  │
│  │  │  │  ConfigRef: hyperdx-config                                                                        │ │   │  │
│  │  │  │  AntiAffinity: podAntiAffinity (spread across nodes)                                              │ │   │  │
│  │  │  └────────────────────────────────────────────────────────────────────────────────────────────────┘ │   │  │
│  │  └────────────────────────────────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                │                                                            │  │
│  │  ┌─────────────────────────────────────────────▼────────────────────────────────────────────────────────┐   │  │
│  │  │                                        STATEFULSET                                                   │   │  │
│  │  │  ┌────────────────────────────────────────────────────────────────────────────────────────────────┐ │   │  │
│  │  │  │  mongo (replicas: 3)                                                                               │ │   │  │
│  │  │  │                                                                                                    │ │   │  │
│  │  │  │  Image: mongo:8.2                                                                                 │ │   │  │
│  │  │  │  Port: 27017                                                                                      │ │   │  │
│  │  │  │  Resources: 1 CPU, 2Gi                                                                            │ │   │  │
│  │  │  │  Storage: 20Gi per pod (Longhorn)                                                                 │ │   │  │
│  │  │  │                                                                                                    │ │   │  │
│  │  │  │  Auth:                                                                                            │ │   │  │
│  │  │  │    - MONGO_INITDB_ROOT_USERNAME (from secret)                                                     │ │   │  │
│  │  │  │    - MONGO_INITDB_ROOT_PASSWORD (from secret)                                                     │ │   │  │
│  │  │  │    - keyFile auth (replicaSet rs0)                                                                │ │   │  │
│  │  │  │                                                                                                    │ │   │  │
│  │  │  │  InitScript: /scripts/entrypoint.sh (from ConfigMap mongo-scripts)                               │ │   │  │
│  │  │  │    - copies keyfile from secret to workdir                                                       │ │   │  │
│  │  │  │    - starts mongod with --replSet rs0 --auth --keyFile                                           │ │   │  │
│  │  │  │    - initializes replica set on mongo-0                                                          │ │   │  │
│  │  │  └────────────────────────────────────────────────────────────────────────────────────────────────┘ │   │  │
│  │  └────────────────────────────────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                │                                                            │  │
│  │  ┌─────────────────────────────────────────────▼────────────────────────────────────────────────────────┐   │  │
│  │  │                                    PODDISRUPTIONBUDGET                                               │   │  │
│  │  │  ┌────────────────────────────────┐    ┌─────────────────────────────────┐                         │   │  │
│  │  │  │ hyperdx-pdb (minAvailable: 1)  │    │ mongo-pdb (minAvailable: 2)     │                         │   │  │
│  │  │  └────────────────────────────────┘    └─────────────────────────────────┘                         │   │  │
│  │  └────────────────────────────────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                │                                                            │  │
│  │  ┌─────────────────────────────────────────────▼────────────────────────────────────────────────────────┐   │  │
│  │  │                                         CRONJOBS                                                     │   │  │
│  │  │  ┌────────────────────────────────┐    ┌─────────────────────────────────┐                         │   │  │
│  │  │  │ mongo-minio-backup             │    │ mongo-minio-restore             │                         │   │  │
│  │  │  │                                │    │                                 │                         │   │  │
│  │  │  │ Schedule: 0 */6 * * *          │    │ Schedule: 0 0 * * *             │                         │   │  │
│  │  │  │ (every 6 hours)                │    │ (daily at midnight)             │                         │   │  │
│  │  │  │                                │    │ suspend: true (disabled)        │                         │   │  │
│  │  │  │  ┌─────────────────────────┐   │    │                                 │                         │   │  │
│  │  │  │  │ init: download mc       │   │    │  ┌─────────────────────────┐   │                         │   │  │
│  │  │  │  │ main: mongodump         │───┼───►│  │ init: download mc       │   │                         │   │  │
│  │  │  │  │   → mc cp to MinIO      │   │    │  │ main: mc cp from MinIO  │───┼─────────────────────────│   │  │
│  │  │  │  └─────────────────────────┘   │    │  │   → mongorestore --drop │   │                         │   │  │
│  │  │  └────────────────────────────────┘    │  └─────────────────────────┘   │                         │   │  │
│  │  │                                        │                                 │                         │   │  │
│  │  └────────────────────────────────────────┴─────────────────────────────────┴─────────────────────────┘   │  │
│  │                                                                                                              │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                              SECRETS & CONFIGMAPS                                           │  │
│  │                                                                                                              │  │
│  │  ┌────────────────────────┐  ┌────────────────────────┐  ┌────────────────────────┐  ┌─────────────────┐   │  │
│  │  │ hyperdx-secrets        │  │ mongo-root-credentials │  │ mongo-keyfile          │  │ mongo-backup-   │   │  │
│  │  │                        │  │                        │  │                        │  │ minio-credentials   │   │  │
│  │  │ - MONGO_URI            │  │ - username             │  │ - keyfile content     │  │ - MINIO_ENDPOINT   │   │  │
│  │  │ - JWT_SECRET           │  │ - password             │  │                        │  │ - MINIO_ACCESS_KEY │   │  │
│  │  │ - NEXTAUTH_SECRET      │  │                        │  │                        │  │ - MINIO_SECRET_KEY │   │  │
│  │  │ - AUTH_SECRET          │  │                        │  │                        │  │ - MINIO_BUCKET     │   │  │
│  │  │ - ENCRYPTION_KEY       │  │                        │  │                        │  │                   │   │  │
│  │  └────────────────────────┘  └────────────────────────┘  └────────────────────────┘  └─────────────────┘   │  │
│  │                                                                                                              │  │
│  │  ┌────────────────────────┐  ┌────────────────────────┐                                                     │  │
│  │  │ hyperdx-config         │  │ mongo-scripts          │                                                     │  │
│  │  │ (ConfigMap)            │  │ (ConfigMap)            │                                                     │  │
│  │  │                        │  │                        │                                                     │  │
│  │  │ - app config values    │  │ - entrypoint.sh        │                                                     │  │
│  │  │                        │  │   (replica set init)   │                                                     │  │
│  │  └────────────────────────┘  └────────────────────────┘                                                     │  │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

Flow Legend:
  ───►  Data/Request Flow (users → ingress → app → mongodb)
  ────► Secrets Flow (Vault → Ansible → K8s Secrets)
  ═══► Backup Flow (CronJob → mongodump → MinIO)
  ═══► Restore Flow (MinIO → CronJob → mongorestore → MongoDB)
```

---

### Использование

#### Ручной запуск бэкапа

Заходим в Rancher, находим вкладку Workloads->CronJobs, выбираем(ставим галочку) mongo-minio-backup, нажимаем Run Now и джем окончания выполнения

#### Ручной запуск восстановления

Заходим в minio - https://edox-minio.firm.ru/, находим бакет - hyperdx и проваливаемя в него в папке mongo-backups видим архивы, берем имя нужного и идем в Rancher

Заходим в Rancher, находим вкладку Workloads->CronJobs, видим mongo-minio-restore, жмем три точки в правом углу, напротив имени джоба, выбираем Edit Config, спускаемся до Environment Variables и находим Variable Name-BACKUP_FILE, напротив в Value видим - hyperdx-*.archive.gz, вот его и надо заменить на нужный бэкап

#### Просмотр списка бэкапов в MinIO

https://edox-minio.firm.ru/browser/hyperdx/mongo-backups%2F

---

### Важные замечания

1. **Бэкап с реплики** — `mongodump` читает с secondary-реплики (`--readPreference=secondary`), чтобы не нагружать primary
2. **Сжатие gzip** — архив сжимается для экономии места в MinIO
3. **Удаление данных при restore** — флаг `--drop` удаляет все данные перед восстановлением
4. **Suspend по умолчанию** — восстановление отключено, чтобы случайно не запустить
5. **Токен Vault** — учётные данные MinIO хранятся в Vault, не в коде

## Безопасность

- Секреты хранятся в HashiVault и получаются в runtime
- Kubeconfig извлекается из Vault
- MongoDB использует keyfile-аутентификацию
- TLS обязателен на Ingress
- Поды запускаются с минимальными привилегиями (настроен security context)
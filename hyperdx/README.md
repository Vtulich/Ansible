# Hyperdx

Ansible-проект для развёртывания платформы наблюдаемости **Hyperdx** в кластере Kubernetes с MongoDB в качестве базы данных.

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

| Компонент | Тип | Реплики | Образ | Порт |
|-----------|-----|---------|-------|------|
| Hyperdx | Deployment | 2 | hyperdx/hyperdx:2.21.0 | 8080 |
| MongoDB | StatefulSet | 3 | mongo:8.2 | 27017 |

### Высокая доступность

- **PodAntiAffinity** — поды Hyperdx распределяются по разным узлам
- **PodDisruptionBudget** — для Hyperdx (minAvailable: 1) и MongoDB (minAvailable: 2)
- **MongoDB ReplicaSet** — 3 реплики с автоматическим failover

## Требования

1. **Ansible** с коллекциями `kubernetes.core` и `community.hashi_vault`
2. **Vault Agent** запущен и доступен
3. **Kubernetes-кластер** с настроенным доступом
4. **Vault-токен** доступен по пути `/opt/{{ vault_agent_name_directory }}/.token_unwrapped`

## Запуск

```bash
# Активировать виртуальное окружение Ansible
. /var/ansible/venv/bin/activate

# Деплой в production
ansible-playbook ./deploy.yml -e "env_code=prod namespace=telr01-01-common"
```

## Переменные

| Переменная | Описание |
|------------|----------|
| `env_code` | Идентификатор окружения (например, "prod") |
| `namespace` | Пространство имён Kubernetes |
| `balancer_url_hyperdx` | Внешний URL для Hyperdx |
| `ingress_hostname` | Имя хоста для Ingress |
| `vault_agent_api_url` | API-адрес агента Vault |
| `vault_agent_name_directory` | Имя директории для токена Vault |
| `vault_ansible_secret_path` | Путь к секретам в Vault |

## Бэкап и восстановление MongoDB

### Бэкап

- **CronJob:** `mongo-minio-backup`
- **Расписание:** каждые 6 часов (`0 */6 * * *`)
- **Место хранения:** MinIO (`hyperdx/mongo-backups`)
- **Имя файла:** `hyperdx-YYYYMMDD-HHMMSS.archive.gz`

### Восстановление

- **CronJob:** `mongo-minio-restore`
- **Расписание:** ежедневно в полночь (`0 0 * * *`)
- **По умолчанию:** приостановлен (`suspend: true`)

### Ручной запуск бэкапа

1. Открыть Rancher → Workloads → CronJobs
2. Выбрать `mongo-minio-backup`
3. Нажать **Run Now**

### Ручной запуск восстановления

1. Выбрать файл бэкапа в MinIO: https://edox-minio.firm.ru/browser/hyperdx/mongo-backups%2F/
2. Скопировать имя файла
3. Открыть Rancher → Workloads → CronJobs → `mongo-minio-restore`
4. Редактировать переменную окружения `BACKUP_FILE`, указав имя файла
5. Снять галочку **Suspend**
6. Сохранить и дождаться выполнения

## Доступ к сервису

- **URL:** https://hyperdx.firm.ru
- **Порт сервиса:** 8098
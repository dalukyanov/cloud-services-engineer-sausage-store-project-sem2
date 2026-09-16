# Sausage Store

Интернет-магазин "Сосисочная" - финальный проект второго семестра ИТМО.

Лукьянов Дмитрий Андреевич

![image](https://user-images.githubusercontent.com/9394918/121517767-69db8a80-c9f8-11eb-835a-e98ca07fd995.png)


## Архитектура

- **Frontend** - TypeScript, Angular. Раздаётся через nginx.
- **Backend** - Java (Spring Boot, Spring Data JPA). Работает с PostgreSQL и MongoDB.
- **Backend-report** - Go. Генерирует отчёты, хранит их в MongoDB.
- **PostgreSQL** - товары, заказы, связи заказов и товаров. Миграции через Flyway.
- **MongoDB** - отчёты об активности пользователей.

## Технологии

- Angular, Node.js 14 (сборка), nginx (раздача)
- Java 16 (target), Maven, Spring Boot 2.6, Flyway 8
- Go 1.22
- PostgreSQL 16, MongoDB 7
- Docker, Helm 3, Kubernetes (Yandex Cloud)
- GitHub Actions, Nexus (Hosted Helm)

## Структура репозитория

    .
    ├── .github/workflows/deploy.yaml   # CI/CD: сборка, Nexus, деплой
    ├── backend/                        # Java backend + Flyway-миграции
    │   ├── Dockerfile
    │   └── src/main/resources/db/migration/
    │       ├── V001__create_tables.sql
    │       ├── V002__change_schema.sql
    │       ├── V003__insert_data.sql
    │       └── V004__create_index.sql
    ├── backend-report/                 # Go-сервис отчётов
    │   └── Dockerfile
    ├── frontend/                       # Angular SPA
    │   ├── Dockerfile
    │   └── nginx.conf
    └── sausage-store-chart/            # Umbrella Helm-чарт
        ├── Chart.yaml
        ├── values.yaml
        └── charts/
            ├── backend/
            ├── backend-report/
            ├── frontend/
            └── infra/

## Локальная сборка Docker-образов

Образы собираются multi-stage, публикуются в Docker Hub:

    docker buildx build --platform linux/amd64 -t indeicheg/sausage-backend:latest        ./backend        --push
    docker buildx build --platform linux/amd64 -t indeicheg/sausage-backend-report:latest ./backend-report --push
    docker buildx build --platform linux/amd64 -t indeicheg/sausage-frontend:latest       ./frontend       --push

`--platform linux/amd64` обязателен: в Yandex Cloud ноды x86_64,
и образы, собранные нативно на Apple Silicon (arm64), там не запустятся.

## Деплой в Kubernetes

Чарт устанавливается из локальной папки:

    helm dependency update sausage-store-chart
    helm lint sausage-store-chart
    helm upgrade --install sausage-store sausage-store-chart \
      -n <namespace> \
      --wait --timeout 10m

Или из репозитория Nexus (после публикации чарта):

    helm repo add nexus $NEXUS_HELM_REPO \
      --username $NEXUS_HELM_REPO_USER --password $NEXUS_HELM_REPO_PASSWORD
    helm repo update
    helm upgrade --install sausage-store nexus/sausage-store \
      -n <namespace> --wait --timeout 10m


## Что разворачивается

- **StatefulSet `postgresql`** + PVC 1Gi - БД товаров и заказов.
- **StatefulSet `mongodb`** + PVC 1Gi - БД отчётов.
- **Deployment `sausage-backend`** - RollingUpdate, VPA (режим рекомендаций, cpu+memory),
  livenessProbe `/actuator/health:8080`.
- **Deployment `sausage-backend-report`** - Recreate, HPA (CPU 75%, min 1, max 2).
- **Deployment `sausage-frontend`** - Ingress с TLS-секретом.

## Доступ

Frontend доступен по адресу из Ingress:

    https://front-dalukyanov.2sem.students-projects.ru

TLS-секрет: `2sem-students-projects-wildcard-secret` (wildcard от Практикума).

## CI/CD

Workflow `.github/workflows/deploy.yaml` запускается по push в `main` и состоит из трёх джоб:

1. **`build_and_push_to_docker_hub`** - сборка и публикация трёх образов
   под `linux/amd64`.
2. **`add_helm_chart_to_nexus`** - `helm dependency update` - `helm package`
   - `curl --upload-file` в Nexus Hosted Helm (Репозиторий создается заранее вручную)
3. **`deploy_helm_chart_to_kubernetes`** - `helm repo add nexus` -
   `helm upgrade --install` из Nexus в кластер.

### Необходимые GitHub Secrets

| Secret | Назначение |
|---|---|
| `DOCKER_USER` | Логин Docker Hub |
| `DOCKER_PASSWORD` | Docker Hub PAT (Read & Write) |
| `POSTGRES_PASSWORD` | Пароль Postgres БД store |
| `MONGO_ROOT_PASSWORD` | Пароль root MongoDB |
| `NEXUS_HELM_REPO` | URL Helm-репозитория Nexus |
| `NEXUS_HELM_REPO_USER` | Логин Nexus |
| `NEXUS_HELM_REPO_PASSWORD` | Пароль Nexus |
| `SAUSAGE_STORE_NAMESPACE` | Namespace в кластере | 
| `KUBE_CONFIG` | Kubeconfig (base64) |

## Миграции БД

Flyway применяет миграции из `backend/src/main/resources/db/migration/`
при старте backend. На чистой БД выполняются последовательно V001-V004:

- **V001** - таблицы `product` и `orders`.
- **V002** - связующая таблица `order_product`.
- **V003** - 6 товаров и 10 000 заказов (данные для отчётов).
- **V004** - индексы для отчётов.

## Проверка после деплоя

    kubectl get pods,svc,ingress,pvc,hpa,vpa -n <namespace>
    kubectl logs deployment/sausage-backend -n <namespace> | grep -i flyway
    curl -I https://front-dalukyanov.2sem.students-projects.ru
    curl -s https://front-dalukyanov.2sem.students-projects.ru/api/products


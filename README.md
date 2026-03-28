# CI/CD Repository

Репозиторий с переиспользуемыми GitHub Actions workflows и вспомогательными файлами для CI/CD.

## Структура

```
.github/workflows/
├── building/
│   └── build-image.yml              # Сборка и пуш Docker-образа (docker/build-push-action)
├── ci/
│   ├── go-test.yml                  # Go тесты + coverage + порог покрытия
│   └── golangci-lint.yml            # Линтинг Go кода (golangci-lint-action)
└── deployment/
    ├── deploy-backend-terraform.yml        # Деплой backend-сервисов через Terraform
    ├── deploy-config-terraform.yml         # Деплой конфигов (+ опциональный Consul port-forward)
    ├── deploy-infrastructure-terraform.yml # Деплой инфраструктуры через Terraform
    └── deploy-secrets-terraform.yml        # Деплой секретов (sops/age + Terraform)

configs/
└── linters/
    └── golangci-lint.yml            # Конфигурация golangci-lint (общая для всех репо)

dockerfiles/
├── Dockerfile.docusaurus
├── Dockerfile.go-app
├── Dockerfile.go-swagger
├── Dockerfile.postgres-migrator
└── Dockerfile.python-djago-app
```

## Использование

Все workflows являются [reusable workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) и вызываются из других репозиториев через `workflow_call`.

### Сборка Docker-образа

```yaml
jobs:
  build:
    uses: your-org/ci-cd/.github/workflows/building/build-image.yml@main
    with:
      image_name: your-org/your-service
      project_type: go-app            # Соответствует Dockerfile.go-app
      registry_host: registry.example.com
      dockerfile_repo: your-org/ci-cd # Если Dockerfile хранится в этом репо
      dockerfile_ref: main
    secrets:
      registry_username: ${{ secrets.REGISTRY_USERNAME }}
      registry_password: ${{ secrets.REGISTRY_PASSWORD }}
```

**Триггер:** Срабатывает на тегах (тег → image tag) и ветках `feature/*`, `fix/*`, `hotfix/*` (ветка → slug как image tag).  
Условия задаются в вызывающем workflow через `on:` и `if:`.

---

### Go тесты с покрытием

```yaml
jobs:
  test:
    uses: your-org/ci-cd/.github/workflows/ci/go-test.yml@main
    with:
      test_path: ./...
      coverage_pkg: ./internal/...
      min_coverage: 75        # Опционально, по умолчанию 75%
      go_version: "1.24"     # Опционально
```

---

### golangci-lint

```yaml
jobs:
  lint:
    uses: your-org/ci-cd/.github/workflows/ci/golangci-lint.yml@main
    with:
      ci_cd_repo: your-org/ci-cd    # Репо с конфигом линтера
      ci_cd_ref: main
      go_version: "1.24"
      golangci_lint_version: "v2.1.6"
```

> **Примечание:** Линтинг не блокирует pipeline (`continue-on-error: true`).

---

### Деплой backend-сервисов

```yaml
jobs:
  deploy:
    uses: your-org/ci-cd/.github/workflows/deployment/deploy-backend-terraform.yml@main
    with:
      deploy_env: prod
      terraform_workspace: myproject
      chart_registry: registry.example.com
      registry_host: registry.example.com
    secrets:
      kubeconfig: ${{ secrets.PROD_KUBECONFIG }}
      chart_registry_username: ${{ secrets.CHART_REGISTRY_USERNAME }}
      chart_registry_password: ${{ secrets.CHART_REGISTRY_PASSWORD }}
      registry_username: ${{ secrets.REGISTRY_USERNAME }}
      registry_password: ${{ secrets.REGISTRY_PASSWORD }}
```

---

### Деплой конфигов (с опциональным Consul port-forward)

```yaml
jobs:
  deploy:
    uses: your-org/ci-cd/.github/workflows/deployment/deploy-config-terraform.yml@main
    with:
      deploy_env: prod
      terraform_workspace: myproject
      chart_registry: registry.example.com
      registry_host: registry.example.com
      consul_server_forward: true          # Опционально
      consul_server_namespace: consul      # Требуется при consul_server_forward=true
      consul_server_svc: consul-server     # Требуется при consul_server_forward=true
    secrets:
      kubeconfig: ${{ secrets.PROD_KUBECONFIG }}
      chart_registry_username: ${{ secrets.CHART_REGISTRY_USERNAME }}
      chart_registry_password: ${{ secrets.CHART_REGISTRY_PASSWORD }}
      registry_username: ${{ secrets.REGISTRY_USERNAME }}
      registry_password: ${{ secrets.REGISTRY_PASSWORD }}
```

---

### Деплой инфраструктуры

```yaml
jobs:
  deploy:
    uses: your-org/ci-cd/.github/workflows/deployment/deploy-infrastructure-terraform.yml@main
    with:
      deploy_env: prod
      terraform_workspace: myproject
      artifact_registry_host: registry.example.com
    secrets:
      kubeconfig: ${{ secrets.PROD_KUBECONFIG }}
      artifact_registry_user: ${{ secrets.ARTIFACT_REGISTRY_USER }}
      artifact_registry_password: ${{ secrets.ARTIFACT_REGISTRY_PASSWORD }}
```

---

### Деплой секретов (sops/age)

```yaml
jobs:
  deploy:
    uses: your-org/ci-cd/.github/workflows/deployment/deploy-secrets-terraform.yml@main
    with:
      deploy_env: prod
      terraform_workspace: myproject
      artifact_registry_host: registry.example.com
    secrets:
      kubeconfig: ${{ secrets.PROD_KUBECONFIG }}
      artifact_registry_user: ${{ secrets.ARTIFACT_REGISTRY_USER }}
      artifact_registry_password: ${{ secrets.ARTIFACT_REGISTRY_PASSWORD }}
      age_private_key: ${{ secrets.AGE_PRIVATE_KEY }}   # Base64-encoded
```

## Secrets Reference

| Secret | Описание |
|--------|----------|
| `REGISTRY_USERNAME` | Логин в container registry |
| `REGISTRY_PASSWORD` | Пароль в container registry |
| `CHART_REGISTRY_USERNAME` | Логин в Helm chart registry |
| `CHART_REGISTRY_PASSWORD` | Пароль в Helm chart registry |
| `ARTIFACT_REGISTRY_USER` | Логин в artifact registry |
| `ARTIFACT_REGISTRY_PASSWORD` | Пароль в artifact registry |
| `{ENV}_KUBECONFIG` | Base64-encoded kubeconfig для окружения (e.g. `PROD_KUBECONFIG`) |
| `AGE_PRIVATE_KEY` | Base64-encoded age private key для sops |
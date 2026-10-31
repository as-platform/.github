# as-platform

Домашняя платформа для разработчиков (Platform Engineering) на K3s. Одна организация GitHub, один dev-контур, одна нода (i3-8100T, 16 GB).

Идея: разработчик выбирает в Backstage шаблон, язык и нужные опции, а платформа создаёт репозиторий, подключает CI/CD и security, собирает подписанный образ и выкатывает его через GitOps.

## Как это работает

1. В Backstage выбираются шаблон, язык и опции: harness [`as-skill`](https://github.com/alexeyshishin/as-skill), security-пайплайн, ночное отключение, деплой в dev.
2. Создаётся репозиторий сервиса с workflow из `ci-templates` и PR в `k3s` с `apps/<name>/`.
3. Push запускает CI на эфемерном раннере: тесты, security, сборка на golden-образах, push в Zot, подпись.
4. `deploy-bump` меняет `image.tag` в `k3s/apps/<name>/values.yaml`, Argo CD синхронизирует, Backstage показывает статус.

## Стек

- **Оркестрация:** K3s, Traefik, cert-manager с внутренним CA
- **GitOps:** Argo CD (app-of-apps, ApplicationSet), Sealed Secrets
- **CI/CD:** GitHub Actions, переиспользуемые actions, self-hosted раннеры на KEDA со scale-to-zero
- **Supply chain:** реестр Zot, подпись cosign, SBOM, rootless-сборка через buildah
- **Хосты:** Ansible
- **Портал:** Backstage

## Репозитории

| Репозиторий | Назначение | Статус |
|-------------|------------|--------|
| [`k3s`](https://github.com/as-platform/k3s) | GitOps-репозиторий кластера: Argo CD, bootstrap, infra, apps | работает |
| [`homelab-infra`](https://github.com/as-platform/homelab-infra) | Ansible-роли для хостов | работает |
| [`ci-templates`](https://github.com/as-platform/ci-templates) | Переиспользуемые CI-actions | работает |
| [`helm-charts`](https://github.com/as-platform/helm-charts) | Общие Helm-чарты | работает |
| [`backstage`](https://github.com/as-platform/backstage) | Портал разработчика | в разработке |
| [`backstage-templates`](https://github.com/as-platform/backstage-templates) | Шаблоны репозиториев и skeletons | в разработке |
| `golden-images` | Базовые образы: cosign, SBOM, rootless buildah, hadolint | приватный |

`golden-images` закрыт намеренно: его CI работает на домашнем раннере и пушит в закрытый реестр.

## Решения и компромиссы

- **Раннеры только для приватных репозиториев.** Форк публичного репозитория с self-hosted раннером даёт выполнение чужого кода в локальной сети. Поэтому job'ы на домашних раннерах живут только в приватных репозиториях, раннеры эфемерные, PAT убирается из окружения job'ов, сборка идёт без docker.
- **Traefik вместо ingress-nginx.** `ingress-nginx` снят с поддержки в марте 2026. Используются стандартные `Ingress`, а не `IngressRoute`, чтобы контроллер можно было заменить.
- **Внутренний CA на 10 лет.** Домена нет, Let's Encrypt не подходит, сертификаты выпускает cert-manager.

## Ограничения

Одна нода, один dev-контур, без staging и prod. Backstage и шаблоны ещё в разработке, ночное отключение инфраструктуры пока не реализовано.

Деплой идёт только через Argo CD.

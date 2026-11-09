# as-platform

🇷🇺 [Русская версия](README_RUS.md)

A home developer platform (Platform Engineering) on K3s. One GitHub organization, one dev environment, one node (i3-8100T, 16 GB).

The idea: a developer picks a template, a language and the options they need in Backstage, and the platform creates a repository, wires up CI/CD and security, builds a signed image and rolls it out through GitOps.

## How it works

1. In Backstage you choose a template, a language and options: the [`as-skill`](https://github.com/alexeyshishin/as-skill) harness, a security pipeline, nightly shutdown, deploy to dev.
2. A service repository is created with workflows from `ci-templates`, plus a PR to `k3s` with `apps/<name>/`.
3. A push triggers CI on an ephemeral runner: tests, security, a build on golden images, push to Zot, signing.
4. `deploy-bump` changes `image.tag` in `k3s/apps/<name>/values.yaml`, Argo CD syncs, and Backstage shows the status.

## Stack

- **Orchestration:** K3s, Traefik, cert-manager with an internal CA
- **GitOps:** Argo CD (app-of-apps, ApplicationSet), Sealed Secrets
- **CI/CD:** GitHub Actions, reusable actions, self-hosted runners on KEDA with scale-to-zero
- **Supply chain:** Zot registry, cosign signing, SBOM, rootless builds with buildah
- **Hosts:** Ansible
- **Portal:** Backstage

## Repositories

| Repository | Purpose | Status |
|------------|---------|--------|
| [`k3s`](https://github.com/as-platform/k3s) | Cluster GitOps repository: Argo CD, bootstrap, infra, apps | working |
| [`homelab-infra`](https://github.com/as-platform/homelab-infra) | Ansible roles for the hosts | working |
| [`ci-templates`](https://github.com/as-platform/ci-templates) | Reusable CI actions | working |
| [`backstage`](https://github.com/as-platform/backstage) | Developer portal | in development |
| [`backstage-templates`](https://github.com/as-platform/backstage-templates) | Repository templates and skeletons | in development |
| `golden-images` | Base images: cosign, SBOM, rootless buildah, hadolint | private |
| `helm-charts` | Shared Helm chart `app`: Deployment, Service, Ingress, ScaledObject with cron | private, in development |

`golden-images` and `helm-charts` are private on purpose: their CI runs on the home runner and pushes to a private registry. For `helm-charts`, publishing the chart to Zot is still in progress.

## Decisions and trade-offs

- **Home runners only for an allowlist of repositories.** A pull request from a fork can add its own workflow and run someone else's code on the local network, so access is layered. The runner group serves only `backstage` and the private build repositories, and runs from external contributors need manual approval. A NetworkPolicy lets runner pods reach only DNS, Zot through Traefik and the internet on ports 80 and 443; the rest of the LAN, the node and cluster services are closed, which a regression workflow checks. Runners are ephemeral, the PAT is removed from the job environment, and builds run without docker.
- **Traefik instead of ingress-nginx.** `ingress-nginx` was retired in March 2026. Standard `Ingress` resources are used rather than `IngressRoute`, so the controller can be replaced.
- **An internal CA for 10 years.** There is no domain, so Let's Encrypt does not fit; certificates are issued by cert-manager.

## Limitations

One node, one dev environment, no staging or prod. Backstage and the templates are still in development, and nightly infrastructure shutdown is not implemented yet.

Deployment goes through Argo CD only.

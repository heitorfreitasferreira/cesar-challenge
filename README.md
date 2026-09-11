# cesar-challenge — repo GitOps (ENTREGAVEL)

Fonte de verdade do deploy da todolist-app no cluster `desafio` (k3d local).
Padrao GitOps: manifests finais aqui, CI no repo da app,
Argo CD reconcilia Git -> cluster. Digest imutavel versionado aqui; o repo
da app nao e segunda fonte de verdade.

## Ambientes e interfaces

| Env | Namespace | Cor | URL | Sync Argo |
|---|---|---|---|---|
| staging | `todolist-staging` | green | http://staging.127.0.0.1.nip.io:8081 | automatico |
| production | `todolist-production` | blue | http://prod.127.0.0.1.nip.io:8081 | manual (promocao) |
| Argo CD UI (bonus) | argocd | — | http://argocd.127.0.0.1.nip.io:8081 | — |
| Grafana | observability | — | http://grafana.127.0.0.1.nip.io:8081 | — |

Trocar a cor de um env = 1 linha no `configmap-patch.yaml` do overlay + push.
Login app: `admin` + senha do SealedSecret (ver `/k-secrets` no opencode).
Login Grafana: `admin` + senha em `envs/grafana-observability.env` (local, gitignored).

## Observabilidade (bonus)

Stack entregue via Argo CD (Helm charts pinados), tudo em `observability`:

- **Metricas de cluster/recursos**: kube-prometheus-stack 90.0.0
  (Prometheus + Grafana + node-exporter + kube-state-metrics, dashboards
  prontos de Kubernetes/Compute Resources). Alertmanager desligado (ver DECISIONS).
- **Logs**: Loki 7.3.0 (single-binary, PVC 1Gi, retencao 72h) + Grafana Alloy
  1.12.1 (DaemonSet coletando stdout de todos os pods). A app loga **JSON**
  (app + access log do gunicorn); o Alloy extrai `level`/`logger` como labels
  e `status/method/path/duration_us` como structured metadata.
  Dashboards `Logs — staging` e `Logs — production` (volume por nivel/logger,
  access log, erros, exploracao com variaveis). Alerta (Grafana-managed) para
  logs de erro acima de 0 em 5 min, com `for: 5m`.
- **Metricas da aplicacao**: `/metrics` na app (prometheus-flask-exporter,
  aditivo, multiprocess-safe p/ gunicorn) + ServiceMonitor por env.
- **Metricas do banco**: postgres-exporter como sidecar do Postgres +
  ServiceMonitor por env.
- **Dashboards**: `TodoList — Aplicacao e Recursos` (custom) e
  `PostgreSQL Database` (grafana.com #9628), auto-carregados pelo sidecar.

## O que esta onde

| Caminho | Papel (requisito) |
|---|---|
| `clusters/desafio/k3d-config.yaml` | R1: cluster criado por codigo |
| `clusters/desafio/argocd/` | R1: install + projects + applications do Argo CD |
| `clusters/desafio/argocd/argocd-ui-ingress.yaml` | Bonus: UI do Argo no Ingress |
| `clusters/desafio/sealed-secrets/` | controller Sealed Secrets (vendorizado, digest pinado) + cert publico |
| `clusters/desafio/observability/` | ServiceMonitors, dashboards e secret do Grafana |
| `k8s/base + k8s/overlays/staging|production/` | R2/R4: manifests da app (fonte unica — nao duplicar no fork) |
| `envs/*.env.example` | modelo tangivel dos secrets (o `.env` preenchido nunca e commitado) |
| `../todolist-app/.github/workflows/` | R2: CI por branch (staging->overlay staging, production->production) |
| `evidencias/` | R5: logs, dumps, prints de navegador |

## Fluxo de branches (GitLab Flow, no repo da app)

`staging` e a branch principal (default). `production` e protegida:
sem push direto (so via PR), so aceita PR de `staging` ou `hotfix/*`,
exige CI verde e historico linear (approval=0: repo de uma pessoa; ver DECISIONS).
Hotfix: branch `hotfix/*` da `production` -> PR em `production` (+ back-merge em `staging`).

Promocao: PR `staging -> production` -> merge -> CI bumpar o overlay production
-> clicar Sync no Argo (production e manual de proposito).

## Reproducao (do zero)

```bash
export KUBECONFIG=~/.kube/desafio-k3d.kubeconfig
k3d cluster create --config clusters/desafio/k3d-config.yaml

# Argo CD
kubectl apply -f clusters/desafio/argocd/namespace.yaml
kubectl apply -n argocd -f clusters/desafio/argocd/install.yaml
kubectl -n argocd rollout status deployment/argocd-server
# UI do Argo via Ingress (bonus):
kubectl -n argocd patch deployment argocd-server --type=json \
  --patch-file clusters/desafio/argocd/argocd-server-insecure.patch.json
kubectl apply -f clusters/desafio/argocd/argocd-ui-ingress.yaml

# Projetos + controller de secrets (infra primeiro)
kubectl apply -f clusters/desafio/argocd/project.yaml
kubectl apply -f clusters/desafio/argocd/project-observability.yaml
kubectl apply -f clusters/desafio/argocd/application-sealed.yaml
kubectl -n kube-system rollout status deployment/sealed-secrets-controller

# App (staging auto; production manual) e observabilidade
kubectl apply -f clusters/desafio/argocd/application-staging.yaml
kubectl apply -f clusters/desafio/argocd/application-production.yaml
kubectl apply -f clusters/desafio/argocd/application-kube-prometheus-stack.yaml
kubectl apply -f clusters/desafio/argocd/application-loki.yaml
kubectl apply -f clusters/desafio/argocd/application-alloy.yaml
kubectl apply -f clusters/desafio/argocd/application-observability.yaml

kubectl -n argocd get applications
# staging sincroniza sozinho; production = clicar Sync no Argo.
```

Novo deploy: `git push` em `staging`/`production` no fork `todolist-app`
dispara a CI (build -> GHCR -> bump de digest no overlay certo -> Argo sync).

Detalhes de operacao, secrets e observabilidade: `DECISIONS.md`
(+ `/k-secrets` no opencode).

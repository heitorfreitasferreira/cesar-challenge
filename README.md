# cesar-challenge — repo GitOps (ENTREGAVEL)

Fonte de verdade do deploy da todolist-app no cluster `desafio` (k3d local).
Padrao GitOps: manifests finais aqui, CI no repo da app,
Argo CD reconcilia Git -> cluster. Digest imutavel versionado aqui; o repo
da app nao e segunda fonte de verdade.

## Como o desafio e atendido (R1–R5)

| Req | Como |
|---|---|
| **R1** Provisionamento automatizado | Cluster k3d por codigo (`clusters/desafio/k3d-config.yaml`); Argo CD, controller de secrets e apps aplicados por `bootstrap.sh` (1 comando, repetivel, sem console). Infra de observabilidade via Helm charts pinados. |
| **R2** Deploy automatizado | `git push` no repo da app -> CI (test + build GHCR) -> bump de digest no overlay -> Argo CD sincroniza. Nada manual. |
| **R3** Acesso externo | Ingress (Traefik) no LB `127.0.0.1:8081`, por host: staging/production/argocd/grafana. |
| **R4** Escalabilidade e resiliencia | 2 replicas + HPA (2..5) + PDB + probes separadas (liveness `/livez`, readiness `/healthz`) + rolling update `maxUnavailable: 0` + requests/limits. |
| **R5** Documentacao | Este README, `DECISIONS.md` (escolhas/descartes/desafios), `evidencias/` (logs/estado) e screenshots. |

## Arquitetura

```
                 git push (staging|production)
  todolist-app ──────────────────────────────► GitHub Actions (test -> build GHCR)
   (codigo+CI)                                          │ bump de digest (kustomize)
                                                        ▼
  cesar-challenge (GitOps)  ◄───────────────────────────┘
   k8s/base + overlays/ + clusters/
        │  Argo CD reconcilia
        ▼
   k3d cluster "desafio" ── Traefik (127.0.0.1:8081)
        ├── todolist-staging      (auto-sync)
        ├── todolist-production   (sync manual = promocao)
        └── observability         (Prometheus/Grafana/Loki/Alloy)
```

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

Forma curta (recomendada) — um comando, repete do zero:

```bash
export KUBECONFIG=~/.kube/desafio-k3d.kubeconfig
./bootstrap.sh                 # cluster -> Argo -> secrets -> apps -> espera Synced/Healthy
# ./bootstrap.sh --reseal       # forca re-selar os secrets com o cert atual
# ./bootstrap.sh teardown       # derruba o cluster
```

O script resolve os secrets automaticamente: se existir `sealed-backup/key.yaml`
(backup local da chave do controller, **nunca versionado**) ele restaura a chave e
mantem os selos do Git; caso contrario, cria os `envs/*.env` sinteticos que faltarem
e **re-sela**, commitando o resultado (o Argo le do Git).

<details><summary>Passo a passo manual (equivalente)</summary>

```bash
k3d cluster create --config clusters/desafio/k3d-config.yaml
kubectl apply -f clusters/desafio/argocd/namespace.yaml
kubectl apply -n argocd -f clusters/desafio/argocd/install.yaml
kubectl -n argocd rollout status deployment/argocd-server
kubectl -n argocd patch deployment argocd-server --type=json \
  --patch-file clusters/desafio/argocd/argocd-server-insecure.patch.json
kubectl apply -f clusters/desafio/argocd/argocd-ui-ingress.yaml
kubectl apply -f clusters/desafio/argocd/project.yaml
kubectl apply -f clusters/desafio/argocd/project-observability.yaml
kubectl apply -f clusters/desafio/argocd/application-sealed.yaml
kubectl -n kube-system rollout status deployment/sealed-secrets-controller
# ... secrets (ver bootstrap.sh) ...
for app in staging production kube-prometheus-stack loki alloy observability; do
  kubectl apply -f clusters/desafio/argocd/application-$app.yaml
done
kubectl -n argocd get applications
```
</details>

Novo deploy: `git push` em `staging`/`production` no fork `todolist-app`
dispara a CI (test -> build -> GHCR -> bump de digest no overlay certo -> Argo sync).

## Desafios encontrados no caminho

Registrados em detalhe no `DECISIONS.md`; os principais:

- `kustomize edit set image` reescrevia o overlay inteiro (comentarios perdidos,
  `newName` espurio) -> bump cirurgico por regex ancorado em `newTag`/`digest`.
- RBAC: `resourceNames` nao vale para `list` -> `/cleanup/status` tomava 403.
  Split: `list` sem restricao + `get/patch` travados por nome.
- Argo esconde da arvore recursos fora da allowlist do AppProject
  (`Pod/ReplicaSet/Job` entraram como view-only).
- Regra de alerta `threshold` auto-referente ("cannot reference itself") ->
  `reduce(last)` + `math $B > 0`.
- Grafana `OOMKilled` por limite chutado -> dimensionado apos medir uso real.

## Limitacoes conhecidas (o que eu faria em producao)

- Postgres de 1 replica (sem HA) e sem backup automatico; sem TLS no Ingress.
- Retencao de logs 72h e alerta sem notificacao (cluster local descartavel).
- Segredos selados sao *cluster-bound*: a chave precisa de backup externo.
- Em producao: Postgres gerenciado/HA + PITR, External Secrets, TLS, multi-cluster
  com ApplicationSet, NetworkPolicies e alertas com roteamento real.

Detalhes de operacao, secrets e observabilidade: `DECISIONS.md`
(+ `/k-secrets` no opencode).

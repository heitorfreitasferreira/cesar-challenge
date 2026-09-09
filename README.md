# cesar-challenge — repo GitOps (ENTREGAVEL)

Fonte de verdade do deploy da todolist-app no cluster `desafio` (k3d local).
Padrao GitOps: manifests finais aqui, CI no repo da app,
Argo CD reconcilia Git -> cluster. Digest imutavel versionado aqui; o repo
da app nao e segunda fonte de verdade.

## Ambientes

| Env | Namespace | Cor | URL | Sync Argo |
|---|---|---|---|---|
| staging | `todolist-staging` | green | http://staging.127.0.0.1.nip.io:8081 | automatico |
| production | `todolist-production` | blue | http://prod.127.0.0.1.nip.io:8081 | manual (promocao) |

Trocar a cor de um env = 1 linha no `configmap-patch.yaml` do overlay + push.
Login: `admin` + senha do SealedSecret (ver `/k-secrets` no opencode).

## O que esta onde

| Caminho | Papel (requisito) |
|---|---|
| `clusters/desafio/k3d-config.yaml` | R1: cluster criado por codigo |
| `clusters/desafio/argocd/` | R1: install + project + applications do Argo CD |
| `clusters/desafio/sealed-secrets/` | controller Sealed Secrets (vendorizado, digest pinado) + cert publico |
| `k8s/base + k8s/overlays/staging|production/` | R2/R4: manifests da app (fonte unica — nao duplicar no fork) |
| `envs/*.env.example` | modelo tangivel dos secrets (o `.env` preenchido nunca e commitado) |
| `../todolist-app/.github/workflows/` | R2: CI por branch (staging->overlay staging, production->production) |
| `evidencias/` | R5: logs, dumps, prints de navegador |

## Fluxo de branches (GitLab Flow, no repo da app)

`staging` e a branch principal (default). `production` e protegida:
sem push direto (só via PR), só aceita PR de `staging` ou `hotfix/*`,
exige CI verde + 1 approval, historico linear.
Hotfix: branch `hotfix/*` da `production` -> PR em `production` (+ back-merge em `staging`).

## Reproducao (do zero)

```bash
export KUBECONFIG=~/.kube/desafio-k3d.kubeconfig
k3d cluster create --config clusters/desafio/k3d-config.yaml
kubectl apply -f clusters/desafio/argocd/namespace.yaml
kubectl apply -n argocd -f clusters/desafio/argocd/install.yaml
kubectl -n argocd rollout status deployment/argocd-server
kubectl apply -f clusters/desafio/argocd/project.yaml
kubectl apply -f clusters/desafio/argocd/application-sealed.yaml
kubectl -n kube-system rollout status deployment/sealed-secrets-controller
kubectl apply -f clusters/desafio/argocd/application-staging.yaml
kubectl apply -f clusters/desafio/argocd/application-production.yaml
# staging sincroniza sozinho; production = clicar Sync no Argo.
kubectl -n argocd get applications
```

Novo deploy: `git push` em `staging`/`production` no fork `todolist-app`
dispara a CI (build -> GHCR -> bump de digest no overlay certo -> Argo sync).
Nada manual.

Detalhes de operacao e rotacao de secrets: `DECISIONS.md` (+ `/k-secrets` no opencode).

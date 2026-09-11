# DECISIONS.md — log flat de decisoes (R5)

Formato: data + contexto + escolha + descarte. Sem cerimonia de ADR (1 decisao = 1 item).

- 2026-09-09 · k3d para o cluster. Contexto: Desafio permite
  local. Escolha: k3d (k3s em Docker, descartavel em 1 comando). Descarte: kind (nao e k3s),
  minikube (sujaria ~/.minikube), cloud (custo + conta pessoal).
- 2026-09-09 · 2 repos (opcao A). Contexto: separar CI (app) de CD (GitOps).
  Escolha: fork `todolist-app` (codigo + workflow) + `cesar-challenge` (manifests + Argo CD).
  Descarte: monorepo (misturaria CI com fonte de deploy).
- 2026-09-09 · Manifests em fonte unica. Contexto: council J2 apontou risco de drift.
  Escolha: `k8s/` SOMENTE em `cesar-challenge`; fork sem `k8s/`. CI faz bump de digest aqui.
- 2026-09-09 · Postgres simples junto no k3d. Contexto: validar SELECT/health sem HA.
  Escolha: Deployment + Service + PVC (sem replicas). Descarte: StatefulSet/HA, Postgres externo.
- 2026-09-09 · Argo CD fonte de verdade. Escolha: Applications apontam para
  `k8s/overlays/staging` (sync automatico) e `k8s/overlays/production`
  (sync manual = promocao); imagem por digest imutavel (`@sha256:`), nunca `:latest`.
- 2026-09-09 · `.opencode/` fora do entregavel. Contexto: IA encorajada mas avaliador
  grade README/DECISIONS/evidencias. Escolha: MCPs e comandos ficam no workspace local.
- 2026-09-09 · Digest com prefixo `sha256:` (fix). Contexto: primeiro bump gerou
  `InvalidImageName` (CI arrancava o prefixo). Escolha: kustomize exige
  `digest: sha256:<hex>`; workflow corrigido e validado pelo proprio loop CI->ArgoCD.
- 2026-09-09 · GitLab Flow (staging principal). Contexto: um env so nao mostra
  promocao; Git Flow seria pesado para 1 app. Escolha: `staging` = default,
  `production` protegida (sem push direto, PR so de `staging`/`hotfix/*`,
  CI verde + approval + linear). Descarte: trunk-based puro (sem gate de prod).
- 2026-09-09 · Sealed Secrets, nao ESO. Contexto: k3d descartavel, sem Vault/AWS.
  Escolha: controller vendorizado (digest pinado) via Argo + `kubeseal --scope strict`
  + `.env.example` tangivel + `/k-secrets` (opencode) como runbook. Descarte: ESO
  (exige provedor externo), Secret em claro (CHANGEME aposentado).
- 2026-09-09 · 1 volume projected, nao 2 mounts. Contexto: app le credenciais de
  arquivos no mesmo `SECRETS_DIR`; 2 volumeMounts no mesmo path esconderiam um.
  Escolha: `projected` com `items:` renomeando `POSTGRES_*` -> `DB_*`.
- 2026-09-09 · Host routing, nao 2a porta. Contexto: duas portas host -> mesma
  porta 80 do LB perdem a distincao no NAT; Traefik so diferencia por Host.
  Escolha: 1 porta (8081) + `staging/prod.127.0.0.1.nip.io` (sem /etc/hosts).
  Descarte: entrypoint Traefik extra (complexidade sem ganho na demo).
- 2026-09-09 · Bump cirurgico, nao kustomize edit. Contexto: `kustomize edit set image`
  reescrevia o kustomization (comentarios deslocados, `newName` espurio, `newTag`
  perdido). Escolha: regex ancorado so em `newTag:`/`digest:` (diff de 2 linhas).
- 2026-09-09 · CronJob citado. Contexto: `X-Cleanup-Token: ...` com dois-pontos+espaco
  em scalar plano virou mapa e o API server rejeitou o CronJob. Escolha: flow style
  com aspas. Sem CronJob, `/cleanup/status` fica vazio e a Role `patch` nao se prova.
- 2026-09-09 · Historico linear, sem purge. Contexto: `CHANGEME` antigo no historico.
  Escolha: correcao para frente (SealedSecrets novos), sem filter-repo/force-push;
  e teste tecnico com segredos sinteticos, beleza do historico > purge.
- 2026-09-11 · Observabilidade com kube-prometheus-stack + Loki + Alloy (via Argo/Helm
  pinado). Contexto: pedido de bonus com metricas de cluster, recursos, banco e app.
  Escolha: stack padrao do mercado com charts pinados e dashboards prontos
  (k8s Compute Resources) + 2 dashboards proprios. Descarte: Prometheus "avulso"
  (sem operator/ServiceMonitor/dashboards) e Elastic (pesado para k3d).
- 2026-09-11 · Alertmanager desligado e defaultRules off. Contexto: alerta sem
  destinatario em ambiente de teste polui o Prometheus. Escolha: desligar e
  documentar; ligar e 1 flag quando houver on-call.
- 2026-09-11 · Loki single-binary + filesystem + retencao 72h. Contexto: k3d
  descartavel, sem S3. Escolha: minimal com PVC local; descarte: SimpleScalable
  (6+ pods sem ganho), MinIO.
- 2026-09-11 · Coleta de logs com Alloy (nao Promtail). Contexto: Promtail esta
  em EOL; Alloy e o sucessor oficial. Escolha: DaemonSet Alloy com
  discovery.kubernetes -> loki.write. Descarte: Promtail (legado).
- 2026-09-11 · Metricas da app aditivas (prometheus-flask-exporter com
  GunicornInternalPrometheusMetrics). Contexto: gunicorn com 2 workers duplica
  contadores se cada worker expuser o proprio registry. Escolha: coletor
  multiprocess (PROMETHEUS_MULTIPROC_DIR + emptyDir), nenhuma rota alterada.
- 2026-09-11 · Metricas do banco via postgres-exporter sidecar no pod do Postgres.
  Contexto: sem credencial extra e sem Service novo apontando para fora.
  Escolha: sidecar + porta `metrics` no Service existente + ServiceMonitor.
- 2026-09-11 · Dashboard do Postgres: grafana.com #9628 (template pronto),
  transformado (datasource `Prometheus`, `__inputs` removidos) e versionado.
- 2026-09-11 · Argo CD UI via Ingress no mesmo LB (bonus). Contexto: port-forward
  nao e entregavel. Escolha: Ingress + `argocd-server --insecure` (TLS terminaria
  no LB em ambiente real); patch JSON versionado. Descarte: ServersTransport do
  Traefik (nao surtiu efeito nesta versao do k3s).
- 2026-09-11 · production: PR obrigatorio com 0 approvals. Contexto: repo de uma
  pessoa; self-approve e impossivel e travaria o proprio fluxo. Escolha: manter
  PR + CI verde + historico linear + sem bypass de admin (push direto continua
  barrado). Descarte: approval=1 (inviavel single-dev).
